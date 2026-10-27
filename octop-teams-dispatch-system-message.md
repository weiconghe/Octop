# 上游 PR 材料：团队派工简报 SystemMessage 触发中段 system 拒单

> 用途：给上游 TencentCloud/Octop 提 PR 的现状与改动思路文档。
> 核实基线：upstream/main `890111b7`（v1.0.3b1，2026-10-27 核对）。上一版基线 `dd9dce47`（v1.0.2）
> 到 `890111b7` 之间共 35 个提交，**无一触碰 `src/octop/infra/agents/teams/`**，问题代码自引入起未变；
> `origin/develop` 同一行同样未修。
> 问题引入：上游 PR #868 `feat(teams): add expert team hosts without changing 1:1 chat`
> （commit `12e5b706`，2026-09-21）。
> **上游已有 open PR #2022** `fix(teams): fold team-dispatch briefings into the member's system prompt`
> （2026-10-14，base=`develop`，当前 `CONFLICTING`，Fixes #1945），采用 middleware 折叠方案，
> 与本文方案不同 —— 提交前必须处理这层关系，详见第三节末。
> 本 fork 已同步修复（REBRAND #71），实测团队面 **194 passed / 1 skipped**（v1.0.3b1 基线）。

---

## 一、上游现状（Root Cause）

**位置**：`src/octop/infra/agents/teams/team_manager.py` → `wire_host_dispatch()` 内的 `enrich()`
（`890111b7` 位于 2351–2374 行，问题行为 2362）：

```python
def _enrich_request(agent_id: str, req: Any) -> Any:   # 实为 async enrich()
    ...
    from langchain_core.messages import HumanMessage, SystemMessage

    question, dispatch = pair
    messages: list[Any] = []
    if dispatch and dispatch != question:
        messages.append(SystemMessage(content=dispatch))   # ← 问题行 (2362)
    if question:
        messages.append(
            HumanMessage(
                content=question,
                id=f"team-peer:{getattr(req, 'thread_id', None)}:{new_ulid()}:human",
            )
        )
    if messages:
        req.messages = messages
    return req
```

**入口是两条、实现是一条**：`AgentManager._install_team_host_dispatch()`（`manager.py:4278`）优先走
`TeamManager.install_host_dispatch()`（`team_manager.py:736`），拿不到时才直接调 `wire_host_dispatch()`
（`manager.py:4293`）。两个入口最终都进同一个 `enrich()`，所以修一处即覆盖全部。

**机制链**：

1. `dispatch` 是派工简报，由 `_dispatch_message()`（`team_manager.py:862`）产出：有群聊转录时用
   i18n `teams.member_briefing_with_history`（`zh.json:946` / `en.json:946`，内嵌整段转录，体量很大），
   否则用 `teams.member_briefing`（`zh.json:945` / `en.json:945`）。两者都自带标记前缀
   `[团队派工，不是用户在直接问你]`（en: `[Team assignment, not a direct user question]`）。
   简报在 `prepare_peer_session()`（`:163`）里由 `stash_peer_prompt()` 暂存，`enrich()` 时由
   `take_peer_prompt()` 取出配对。
2. `enrich()` 把它作为 **SystemMessage** 放进成员（member）调用的输入。成员在共享群聊线程上
   执行时，LangGraph 把输入 append 到既有 checkpoint 历史。
3. 模型请求最终形态：

   ```
   [system(agent prompt), ...房间历史..., SystemMessage(dispatch), HumanMessage(question)]
   ```

   ——system 出现在**中段**（非队首、也不与前一条 system 相邻）。

4. **失败分两条互不相干的路，错误串不同，此前把它们混为一谈了**：

   | 路径 | 触发方 | 实际报错 |
   |---|---|---|
   | OpenAI 兼容推理服务（vLLM / DashScope 兼容模式、Qwen 模板） | **服务端**请求校验 | HTTP 400 `System message must be at the beginning.`（issue #1945 实测，Qwen 3.6） |
   | Anthropic 适配层 | **客户端** `langchain_anthropic` 本地抛 | `ValueError: Received multiple non-consecutive system messages.` —— 出自 `langchain_anthropic/chat_models.py:549`（`_format_messages()` 内 `if system is not None: raise ValueError`），langchain-anthropic 1.7.2 实测存在 |

   上一版文档断言该 `ValueError` 是"上游推理服务透传"，并称"本地全栈逐层 grep 过……均无此字符串"——
   **这个结论是错的**：当时的 grep 清单（octop / octop-harness 1.0.4 / langchain 1.4.2 /
   deepagents 0.7.16 / langchain-openai 1.3.3）恰好漏掉了唯一含此串的 `langchain-anthropic`。
   octop 源码与 octop-harness **1.0.5** wheel（当前 `pyproject.toml` 要求 `>=1.0.5`）确实都没有该串。
5. **首次派工到全新成员不报错，但简报静默丢失**：harness 的 bootstrap middleware 会在 agent 首轮
   丢弃对话内的 system message，所以第一次派工"看起来正常"，实际成员根本没收到派工简报。
   这条分支上一版文档完全没提（PR #2022 正文已记录）。
6. **更糟的是持久化污染**：SystemMessage 条目会留在线程 checkpoint 里，此后该线程每一轮模型
   调用都带着中段 system，直到历史被修剪/压缩才自愈。

**生产表现**（自部署实例实测）：主持人派工轮次连续 9 次同一 `stream_error`；非派工轮次完全
正常；两次故障窗口间隔约 9 小时。是否复现取决于当次调用命中的模型/上游引擎——宽容的引擎
（直接合并或忽略多余 system）完全无感，所以看起来「时好时坏」，极难排查。

## 二、影响面

- 团队/专家团的每次派工（host → member 的 `ask_agent`/回叫）在严格上游上**必然失败**，且
  污染线程后续轮次（连普通聊天轮次也一起失败，直到历史修剪）。
- provider 相关性强：同一产品在不同模型/中转组合下表现迥异。Anthropic 与 vLLM 系是两条独立
  失败路径（见上表），不要指望"换个模型就好了"能定位问题。
- Anthropic 路径同样不允许中段 system，同样受益。

## 三、改动思路（Proposed Fix）

把派工简报从 `SystemMessage` 改为**带原标记前缀的 `HumanMessage`**（独立一条，不并入 question），
并**同步给 fallback 转录路径补标记过滤**：

```diff
-        from langchain_core.messages import HumanMessage, SystemMessage
+        from langchain_core.messages import HumanMessage

         question, dispatch = pair
         messages: list[Any] = []
         if dispatch and dispatch != question:
-            messages.append(SystemMessage(content=dispatch))
+            messages.append(
+                HumanMessage(
+                    content=dispatch,
+                    id=f"team-peer:{getattr(req, 'thread_id', None)}:{new_ulid()}:dispatch",
+                )
+            )
         if question:
             messages.append(
                 HumanMessage(
```

```diff
     def _transcript_line(self, msg: Any, host_id: str, locale: str) -> str | None:
         role = _message_role(msg)
         if role in {"tool", "system"}:
             return None
         if _message_tool_calls(msg):
             return None
         text = _assistant_text(msg)
-        if not text:
+        if not text or _is_team_system_prompt(text):
             return None
```

**为什么必须补第二处**（这是上一版文档的遗漏，也是方案正确性的关键）：

`_is_team_system_prompt()`（`team_manager.py:1575`）按文本前缀匹配 `[团队派工` / `[Team assignment`
等标记。直接调用它的地方实际是 **3 处**，不是上一版写的"`_last_room_user_text` / `_visible_user_prompt` /
`_projected_transcript_line`"：

| 消费点 | 行号 | 是否过滤 |
|---|---|---|
| `_last_projected_user_text()` | 898 | ✅ 有 |
| `_last_room_user_text()` | 910 | ✅ 有 |
| `_projected_transcript_line()` | 956 | ✅ 有 |
| **`_transcript_line()`** | **986** | **❌ 没有——只按角色跳过 `tool`/`system`（988）** |

`_visible_user_prompt()`（877）只是聚合调用方（先 `_last_room_user_text`、后 `_last_projected_user_text`），
本身不直接过滤。而 `_room_transcript()`（914）**投影优先、fallback 兜底**：
`_projected_transcript()` 返回空（repo 未注入 / `projection_status != "ready"` / 无行）时，退回
`_history()` + `_transcript_line()`。

实测反证（v1.0.3b1，直接调用 `TeamManager._transcript_line`）：

```
_is_team_system_prompt(dispatch) = True
今天（SystemMessage）   → None
改后（HumanMessage）    → '用户: [团队派工，不是用户在直接问你]\n你是这个专家团队的成员……'
```

即：**只改 enrich、不补 `_transcript_line`，投影尚未 ready 的房间里，派工简报会被当成用户发言
整段写进群聊转录**，再通过 `member_briefing_with_history` 的 `{history}` 喂给下一轮成员——
派工简报自我嵌套。上一版"标记前缀保证房间转录与主持人上下文行为完全一致"的判断，只在投影路径
成立，fallback 路径不成立。

**为什么是这个形态**（三个备选里最稳）：

| 备选 | 结论 | 原因 |
|---|---|---|
| ① 并进 question（一条 HumanMessage） | ❌ | 合并文本以 `[团队派工]` 标记开头，会被 `_is_team_system_prompt()` 整条吞掉——question 从主持人 wrap-up 上下文和转录里消失（回归） |
| ② 提升到队首 system | ❌ | `ChatRequest` 没有 `system` 字段（已核对 `octop_harness/request.py`，字段为 `messages` / `thread_id` / `user` / `source` / `model` / `mcp_servers` / `mcp_use_default` / `skills` / `conversation_mode` / `agent_id` / `configurable` / `metadata` / `recursion_limit`）；agent 提示词在模型请求期才由 middleware 注入，`enrich()` 阶段拿不到队首 system 可合并 |
| ③ **HumanMessage + 标记前缀 + 补全 `_transcript_line`（推荐）** | ✅ | 三个既有过滤点按文本标记挡掉派工文本；补上 `_transcript_line` 后 fallback 与投影路径行为一致。投影响应层 `projection.py:70`（`if role in ("", "system"): return None`）今天靠角色跳过 System 行，改后由文本过滤接管，可见性相同 |

**顺手对齐**：`team_manager.py:1570` 已有 `_peer_user_message_id(thread_id, turn_id)`（257 / 837 行在用），
`enrich()` 里两处 id 却手写 f-string。建议统一为
`_peer_user_message_id(thread_id, new_ulid())` 派生前缀，避免三处 id 格式漂移。

**附带收益**：模型请求不再出现中段 system → 所有 OpenAI 兼容引擎与 Anthropic 适配都可接受；
checkpoint 不再被 system 条目污染。

### 与上游 PR #2022 的关系（提交前必读）

#2022（`alison-xx`，2026-10-14，base=`develop`，`Fixes #1945`，+153/−0，当前 `CONFLICTING`）用另一种
思路解同一个 bug。其核心是新增 `src/octop/infra/agents/middleware/system_messages.py`（55 行）：

```python
def fold_system_messages(request: ModelRequest[Any]) -> ModelRequest[Any]:
    folded = [m for m in request.messages if isinstance(m, SystemMessage)]
    if not folded:
        return request
    blocks = list(request.system_message.content_blocks) if request.system_message else []
    for message in folded:
        text = message.text
        if not text:
            continue
        blocks.append({"type": "text", "text": f"\n\n{text}" if blocks else text})
    return request.override(
        system_message=SystemMessage(content_blocks=blocks) if blocks else None,
        messages=[m for m in request.messages if not isinstance(m, SystemMessage)],
    )
```

外层 `SystemMessageFoldMiddleware` 在 `wrap_model_call` / `awrap_model_call` 里调用它，装配点在
`manager.py:3617`（`WorkspaceImageMaterialize` 之后、`OctopUiOffload` 之前）。
一句话：**只在模型调用那一刻把中段 system 抽出来拼进队首 prompt，存储侧一行不动。**

| 维度 | #2022（middleware fold） | 本文（源头改 HumanMessage） |
|---|---|---|
| 改动位置 | model-call middleware + `manager.py` 一行装配 | `team_manager.py` 生成侧 + `_transcript_line` |
| 改动量 | +153/−0（新文件 55 + 测试 93 + 装配 1） | 约 3 行 + 1 行，加测试 |
| checkpoint | 仍留 SystemMessage 条目（脏数据在，只是请求期被中和） | 不再产生 system 条目 |
| 转录/投影回归风险 | **零** —— 角色没变，投影层 `projection.py:70` 照旧按 role 跳过 | 有 —— 必须补 `_transcript_line`，否则 fallback 转录把简报当用户发言（本文已给实证） |
| 语义 | 简报以 **system** 身份进模型（高优先级，模型更愿意服从） | 简报以 **human** 身份进模型（降级为普通输入） |
| 覆盖面 | 任何来源的中段 system 都被兜住 | 只解团队派工这一处（当前确实只有这一处） |
| 首轮静默丢失 | **未修** —— body 承认"首轮 bootstrap 丢弃、简报静默丢失"，但 55 行实现里没有任何针对它的逻辑，也没相关测试 | 修了 —— HumanMessage 不在 bootstrap 丢弃范围内 |
| 潜在副作用 | 无条件折叠所有 SystemMessage；注释称"团队派工是唯一来源"只是**当前事实**而非契约，将来别处再塞 system 会被静默提到队首 | 依赖文本标记过滤，新增派工文案若不以 `[团队派工` 开头会漏网 |
| 现状 | `CONFLICTING`，未合并 | 未提交 |

**该选哪个**：

- 站维护者角度，#2022 更容易被接受：改动集中、不动存储语义、零转录风险、所有 provider 一次性兜住。
- 站正确性角度，本文更彻底：源头不产生中段 system，且覆盖 #2022 明确没修的首轮静默丢失分支。
  但代价是必须同时改 `_transcript_line`，否则引入比原 bug 更隐蔽的回归（简报自我嵌套）。
- 结论：**两条不是重复，是不同层的解**。若维护者已倾向 #2022，本文可退为"补首轮 + 源头收敛"；
  若另开 PR，**必须在描述里点名 #2022 并说明差异**，否则会按重复处理（`check-duplicates` 精确匹配标题）。

## 四、测试

- 改写 `test_install_team_host_enrich_uses_host_assignment`（`tests/unit/agents/test_agent_manager.py:477`，
  原断言 `assert isinstance(out.messages[0], SystemMessage)`（:498）恰好锁死了 bug 行为）：
  - enrich 输出**不含任何 SystemMessage**；
  - 两条均为 HumanMessage，id 后缀 `:dispatch` / `:human`；
  - 派工标记文本与 question 内容原样保留。
- **新增** `_transcript_line` 回归测试：喂入 `HumanMessage(content=<marker-prefixed dispatch>)`，
  断言返回 `None`；喂入普通 `HumanMessage` 断言仍产出「用户: …」行。没有这条，第三节的第二处
  改动就是无主改动。
- 回归范围：`tests/unit/agents/test_agent_manager.py` + `tests/unit/agents/test_shared_team_peers.py`
  + `tests/unit/gateway/test_global_processor_team.py` → **194 passed, 1 skipped**（`890111b7` 实测，
  耗时约 120s；上一版文档写的 192 passed 是 `dd9dce47` 的旧数字）。

## 五、可直接粘贴的英文 PR 草稿

**Title**: `fix(teams): deliver member dispatch briefing as a human turn to avoid mid-conversation system messages`

**Body**:

> ### Summary
> `wire_host_dispatch()`'s `enrich()` puts the member dispatch briefing (i18n `teams.member_briefing[_with_history]`) into the peer invocation as a `SystemMessage`. When the member runs on a shared room thread, LangGraph appends that input after the existing history, so the final model request looks like:
>
> `[system(agent prompt), ...room history..., SystemMessage(dispatch), HumanMessage(question)]`
>
> That mid-conversation system message breaks in two independent ways:
> - OpenAI-compatible servers (vLLM / DashScope compatible mode with Qwen templates) reject it server-side with `400 System message must be at the beginning.` (#1945).
> - `langchain_anthropic` raises `ValueError: Received multiple non-consecutive system messages.` locally from `_format_messages()` (`chat_models.py:549`) as soon as a second system message shows up.
>
> Because the system entry persists in the thread checkpoint, **every later turn in that thread keeps failing** until the history gets trimmed. Lenient providers merge/ignore extra system messages, which is why the failure looks intermittent and provider-dependent. On a brand-new member the first dispatch silently loses the briefing instead (bootstrap drops in-conversation system messages on turn one).
>
> ### Fix
> Deliver the briefing as a marker-prefixed `HumanMessage` (id suffix `:dispatch`) instead, and let `_transcript_line()` skip marker-prefixed texts like the other projector helpers already do.
>
> `_is_team_system_prompt()` consumers (`_last_projected_user_text`, `_last_room_user_text`, `_projected_transcript_line`) filter team-system texts by text marker. `_transcript_line()` — the fallback used by `_room_transcript()` when the thread projection is not ready — only skipped by role, so it would start echoing the briefing back as a user line once the briefing is a human turn. Adding the marker check there keeps fallback and projected transcripts identical.
>
> Alternatives considered: merging the briefing into the question message breaks transcript filtering (the merged text starts with the marker and gets dropped whole); hoisting it into the leading system message is impossible from `enrich()` (`ChatRequest` has no system field; the agent prompt is injected at model-request time by middleware).
>
> ### Relation to #2022
> #2022 solves the same bug at the middleware layer: `SystemMessageFoldMiddleware` lifts mid-conversation system messages into the leading prompt at model-call time (`request.override(...)`), leaving the checkpoint untouched. It is a smaller, lower-risk change and covers every provider in one shot.
>
> This PR removes the mid-conversation system message at the source instead. Two differences worth calling out:
> 1. The checkpoint stays clean, so no later reader of the thread ever sees a system turn in the middle.
> 2. It also fixes the first-dispatch case. #2022's own description notes that on a brand-new member the bootstrap middleware drops the in-conversation system message, so the briefing is silently lost — its implementation contains no handling for that path.
>
> What this PR costs: `_transcript_line()` (the fallback used by `_room_transcript()` when the projection is not ready) only skipped by role, so it must learn the marker check too — otherwise the briefing starts echoing back as a user line. That is the one extra line this approach needs, and the reason it is not a drop-in replacement for #2022.
>
> Both can coexist; if #2022 is preferred, this one can be reduced to the `enrich()` + `_transcript_line()` pair.
>
> ### Testing
> - Updated `test_install_team_host_enrich_uses_host_assignment`: enriched input must contain no `SystemMessage`, both entries are `HumanMessage` with `:dispatch`/`:human` id suffixes.
> - Added `_transcript_line` coverage: marker-prefixed human turns are skipped, plain user turns still render.
> - Regression: `test_agent_manager.py` + `test_shared_team_peers.py` + `test_global_processor_team.py` → 194 passed, 1 skipped.

---

## 附：本 fork 侧已同步落地的内容（自有化修复）

- `src/octop/infra/agents/teams/team_manager.py`：同 diff（REBRAND #71 登记）。
- `tests/unit/agents/test_agent_manager.py`：断言改写。
- 回归：agents + gateway 团队面 194 passed / 1 skipped（`890111b7` 基线）。
- 注意：本修复只阻止**新增**污染；已经带中段 system 的存量房间线程，需等历史修剪/压缩自然
  自愈，或手动重开线程。
- 若后续合入 #2022 的 middleware 方案，本节与第三节第一处 diff 需重新评估是否保留。
