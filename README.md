# T3 Code — ChatGPT Optimized

> [!IMPORTANT]
> **Design fork only.** This fork does not yet implement the behavior described below. The purpose of this README is to define the desired changes, constraints, and rationale before code is modified.

This repository is a fork of [pingdotgg/t3code](https://github.com/pingdotgg/t3code). Upstream T3 Code is an agent-harness control surface that can launch and manage Codex, Claude Code, Cursor, Grok Build, OpenCode, and other coding agents. Upstream also exposes an MCP server for outside agents such as ChatGPT.

The goal of this fork is intentionally narrow: turn T3 Code into a strong **ChatGPT-to-Codex bridge**.

This fork should be thought of primarily as a ChatGPT plugin backed by T3's execution machinery. Preserving the complete general-purpose T3 MCP experience for Claude, other outside agents, or arbitrary MCP clients is not a design requirement. If simplifying the ChatGPT-to-Codex path makes this fork less useful as a generic T3 distribution, that is an acceptable tradeoff.

The intended division of labor is:

```text
User
  |
  v
ordinary ChatGPT conversation
(requirements, discussion, repository inspection, implementation planning)
  |
  | explicit approved handoff
  v
T3 Code MCP
  |
  v
T3 top-level thread
  |
  v
Codex
(implementation)
  |
  v
project / worktree
```

ChatGPT should do the expensive conversational work: understanding the request, inspecting the relevant code, finding ambiguities, discussing implementation choices with the user, and producing a complete implementation specification. T3 should provide the durable execution environment. Codex should implement the approved specification.

This fork should reuse T3's durable thread, worktree, provider, pending-request, and Codex integration machinery rather than implement another Codex protocol layer. Compatibility work that exists only to preserve non-ChatGPT MCP workflows should not be allowed to complicate the primary Chat-to-Codex path.

## Why optimize for ordinary ChatGPT Chat?

This project deliberately targets ordinary ChatGPT conversations rather than assuming ChatGPT Work, Codex Cloud, or a custom Responses API application.

That is a usage and product decision, not a claim that the more agentic OpenAI surfaces are incapable of solving the same problems.

OpenAI currently describes [ChatGPT Work and Codex](https://help.openai.com/en/articles/20001275-chatgpt-work-and-codex) as long-running work surfaces and states that Work follows the same usage structure as Codex. For the intended user of this fork, Codex capacity is comparatively scarce and is consumed quickly. Moving requirements analysis, architecture discussion, repository reading, and handoff preparation into ordinary ChatGPT Chat is therefore advantageous. Codex usage should be spent primarily on implementation.

Similarly, OpenAI's API now supports [asynchronous tool calling](https://developers.openai.com/api/docs/guides/async-tool-calling), and Work can perform longer multi-step work. Those capabilities can support designs in which an agent waits on background work, receives later results, or resumes after a dependency completes. They do not remove the reason for this fork because this fork is intentionally designed around **ordinary ChatGPT plugin/MCP behavior and its usage economics**.

The project should not artificially reproduce Work inside Chat. It should give Chat a small, reliable control surface and let T3 remain the place where long-running coding work lives.

## Important ordinary-Chat constraint: do not depend on long-lived MCP calls

The intended workflow must not require an MCP tool call from ordinary ChatGPT to remain open for the duration of a coding task.

In prior experimentation with a separate Human Tool Call MCP project, the OpenAI MCP host was observed retrying a blocked request at roughly the one-minute mark and cancelling it around the ninety-second mark. Those timings are an empirical observation from that project, not a documented universal OpenAI guarantee, and they may change.

The architectural conclusion is more durable than the exact numbers: **a normal ChatGPT plugin call must be allowed to finish quickly**.

A call such as:

```text
ChatGPT -> launch Codex -> wait until Codex finishes or asks a question
```

is therefore outside the target design.

The target design is:

```text
ChatGPT -> launch Codex -> receive thread/run identifiers -> end the ChatGPT turn
```

T3 and Codex continue independently. The user can monitor the run in T3 Code. If Codex asks a question, the user can answer it directly in T3. If the user wants ChatGPT's help with the question or wants to steer the run, the user can return to ChatGPT and explicitly ask it to read or message the T3 thread.

There should be no requirement for ChatGPT to poll correctly, remember to issue another wait call, or keep adding filler wait results to the conversation context.

## No `t3_thread_wait` in this fork's MCP surface

This fork should **not advertise `t3_thread_wait` to ChatGPT at all**.

This is stronger than merely instructing ChatGPT not to use it. If a tool is inappropriate for the intended Chat-to-Codex workflow, it should not be in the model's tool surface.

The reason is reliability. A prompt-level instruction such as "do not call wait" still leaves a contradictory affordance in the model's context. The model can choose to call it anyway, can decide to poll repeatedly, or can make different choices after model updates. Removing the tool eliminates the decision.

There is no need to introduce a profile system merely to preserve the broader upstream MCP surface. This fork can directly curate the MCP tools it advertises. If that makes the fork less useful to other MCP clients, that is an accepted consequence of optimizing it specifically as a ChatGPT plugin.

The filtering should happen on the T3 server side so unwanted tools never appear in `tools/list`. ChatGPT's own plugin settings can still provide a second layer of filtering, but correctness should not depend on them.

## Tool descriptions must remain coherent after filtering

This requirement is important.

Upstream T3 tool descriptions sometimes reference other T3 tools. For example, the current `t3_thread_launch` description tells the model:

> Follow it with `t3_thread_read` or `t3_thread_wait`.

If `t3_thread_wait` is hidden but that description survives unchanged, the model is still being instructed to use a capability that the profile intentionally removed.

The fork therefore needs **descriptions rewritten for its actual MCP surface**, not merely a reduced tool list.

At minimum, every advertised tool description and every advertised parameter-schema description should be checked for references to tools that this fork does not advertise. The build should ideally contain an invariant test that fails when an advertised description names a T3 tool that is absent from the fork's MCP surface.

Examples of upstream descriptions that should be reconsidered for this profile include:

- `t3_thread_launch`, which currently mentions `t3_thread_wait`.
- `orchestrator_capabilities`, which currently describes its output primarily in terms of `delegate_task`.
- `t3_thread_interrupt`, which currently tells delegated-task callers to use `task_cancel`.

The intended `t3_thread_launch` description should communicate something closer to:

> Create a new top-level T3 thread and optionally start its first run. Return after the thread is launched. Do not poll or wait for completion. The user monitors long-running work in T3 Code. Read, steer, or interrupt the thread later only when the user explicitly asks.

The intended `orchestrator_capabilities` description should simply explain that it lists providers, models, model options, runtime modes, and interaction modes that can be selected for a T3 thread.

The desired rule is simple: **the descriptions presented to the model should describe the workflow we actually want the model to perform.**

## Initial MCP tool surface

The fork's MCP surface should be intentionally small and designed around ChatGPT planning and Codex execution.

The expected core tools are approximately:

```text
orchestrator_capabilities

t3_project_list
t3_project_read

t3_thread_launch
t3_thread_list
t3_thread_read
t3_thread_send
t3_thread_interrupt
t3_pending_request
t3_thread_configure
t3_thread_search
```

Attachment support is also useful and can remain available if its tool chain is kept coherent:

```text
t3_attachment_prepare_upload
t3_attachment_discard
t3_thread_send_attachments
```

The exact allowlist should be decided from actual use rather than by exposing everything preemptively.

Tools for delegation, task polling, scheduling, background waits, environment linking, PR watching, browser automation, HTML rendering, device control, and other unrelated orchestration should initially be absent unless a concrete ChatGPT planning workflow requires them.

In particular, the following families are not part of the initial target:

```text
t3_thread_wait
t3_background_wait

delegate_task
task_status
task_cancel

schedule_task
list_scheduled_tasks
delete_scheduled_task
run_scheduled_task_now
```

This is not a judgment that those tools are bad. They are useful in T3's broader orchestration environment. They increase the action space and create behavior we do not want from the ordinary-Chat planner.

## Add first-class read-only repository inspection

For ChatGPT to be the planning layer, it needs to understand the repository itself.

Project metadata is not enough. ChatGPT should be able to inspect the registered project's actual code without first launching Codex merely to act as its eyes.

The fork should therefore expose a small set of first-class, read-only repository tools to ChatGPT. Exact names can change, but the desired capabilities are:

```text
t3_project_tree
t3_project_file_read
t3_project_search
t3_project_git_status
t3_project_git_diff
```

These should be intentionally narrower than a generic shell tool.

Repository inspection should be restricted to a registered T3 project/workspace. Paths should be normalized and prevented from escaping the project root. File reads and search results should have sensible size limits and pagination. Git status and diff should be read-only. These tools should advertise appropriate MCP read-only/idempotent annotations.

The point is to let ChatGPT answer questions such as:

- Where is this behavior implemented?
- What interfaces will this change affect?
- Does the requested change conflict with existing architecture?
- Which files actually need to change?
- What is already modified in the worktree?
- What would Codex be starting from?

The planner should be grounded in repository reality before it sends a coding task.

## Planning and approval workflow

The ChatGPT-side workflow should favor discussion before execution.

For a substantive implementation request, ChatGPT should inspect the relevant code, identify material ambiguities, discuss implementation choices with the user, and produce a concrete handoff. It should not launch a coding thread merely because the user mentioned a desired feature.

The default should require explicit user approval before a new coding handoff is launched.

A handoff should carry enough context that Codex can execute without reconstructing product decisions. A useful structure is:

```text
TASK
One concrete objective.

CURRENT STATE
Relevant existing behavior and architecture.

REQUIRED CHANGES
Complete behavioral requirements.

IMPLEMENTATION DECISIONS
Choices already resolved with the user.

FILES / COMPONENTS IN SCOPE
Known scope. Necessary adjacent changes are allowed when required.

CONSTRAINTS
Compatibility requirements, prohibited changes, and project rules.

VALIDATION
How success should be checked.

TEST POLICY
What kinds of tests may or may not be added.

AMBIGUITY POLICY
What Codex should do if repository reality creates a material unresolved decision.
```

The specification sent to Codex should be authoritative for product and architecture decisions already resolved in ChatGPT.

Codex should not broaden scope or invent a material product/architecture decision that the handoff intentionally left unresolved. If repository reality creates a meaningful contradiction or ambiguity, Codex may stop the affected work and ask the user in T3.

Otherwise it should complete the specification.

## Prefer a fresh top-level T3 thread for substantive handoffs

The default handoff should create a new top-level T3 thread for a new substantive implementation task.

This keeps the implementation agent's working context aligned with the approved specification and reduces stale assumptions from unrelated previous work. It is especially useful in young projects where implementations and architecture change rapidly.

Follow-up corrections, steering, or answers to a question can continue in the same T3 thread.

This is a workflow default, not a hard server restriction. The user should be able to choose an existing thread when continuity is useful.

## Launches must be idempotent and return immediately

`t3_thread_launch` already supports a `clientRequestId`. The ChatGPT workflow should use a stable request identifier for a single approved handoff so that a lost MCP response can be retried without accidentally creating duplicate coding threads.

A successful launch should give ChatGPT enough information to report that execution has begun:

```text
threadId
runId
link
selected provider/model
initial status
```

That should be the end of the ChatGPT execution turn unless the user explicitly asked for another immediate action.

There should be no automatic `read -> wait -> read -> wait` loop after launch.

## Steering and mid-work questions

Codex can discover information during implementation that was not visible during planning.

The user should remain able to steer a running run through the T3 interface directly. ChatGPT should also retain `t3_thread_send` so the user can return to the planning conversation and say, for example:

> Tell the coding thread to preserve the old cache format.

When the target thread is running, T3's existing steering semantics should be used rather than interrupting and restarting unless the user specifically requests a restart.

For provider questions, T3's durable pending-request system should remain the source of truth. The user can answer a Codex question in T3 Code itself. The fork should expose `t3_pending_request` so that, when explicitly asked, ChatGPT can read the pending question, discuss it with the user, and submit the answer.

The fork should **not** attempt to wake an ordinary ChatGPT conversation automatically when Codex asks a question. That is deliberately outside the first version.

## No automatic completion wakeups or polling

This fork does not need to implement MCP Events, a webhook callback system, a polling daemon, or a replacement for ChatGPT Work.

When Codex completes, the user can see completion in T3. If the user wants a review in ChatGPT, they can return and ask ChatGPT to read the thread and relevant diff.

This intentionally trades automatic wakeups for predictable usage, simpler state, and fewer model-driven control loops.

If ordinary ChatGPT later gains a reliable asynchronous plugin execution model that fits this usage strategy, the design can be revisited. The T3 execution state should remain durable enough that adding a future notification mechanism does not require changing the coding-agent architecture.

## Default testing policy for coding handoffs

The intended personal workflow should support the following default policy in the ChatGPT-generated handoff:

> Use relevant existing tests when useful. Temporary diagnostic or component-specific tests may be created for ad hoc validation. Do not add new persistent regression, unit, or integration tests to the project's normal test suite unless the handoff explicitly requests them.

This policy exists because the target projects may be young and implementation approaches may be replaced frequently. Persistent regression tests can become maintenance obligations for implementation details that are intentionally short-lived.

The policy should live in the planning/handoff layer or project configuration rather than being hard-coded into Codex execution itself. This is a personal workflow choice for this fork.

## Preserve Codex model and reasoning selection

The ChatGPT planner should be able to inspect the currently available Codex models and supported model options and deliberately select the implementation model and reasoning effort when launching a thread.

The fork should not hard-code one Codex model identifier because OpenAI's Codex catalog changes over time. The relevant capability tool should remain the source of current Codex model IDs, reasoning-effort choices, and compatible runtime modes.

Upstream T3's broader provider architecture may remain internally because it is useful infrastructure, but preserving first-class MCP workflows for non-Codex providers is not a goal of this fork. The external product goal is ChatGPT planning work that is handed to Codex for execution.

## Desktop UX: keep ChatGPT and T3 visible together

T3 Code's desktop app already contains a multi-tab browser surface. This fork should make that useful for the planning/execution workflow.

The desired experience is to keep `chatgpt.com` open beside the active T3 coding thread so the user can discuss the implementation in ChatGPT while watching or steering Codex without constantly switching between T3 and a separate browser window.

If upstream behavior does not already provide it cleanly, useful refinements would include:

- A browser tab that can be pinned or scoped to the environment/workspace rather than disappearing with thread changes.
- Persistent browser profiles so the ChatGPT login survives normally.
- A clear human-owned state for a ChatGPT browser tab.
- Agent browser automation disabled for that tab by default.
- Easy side-by-side switching between the ChatGPT browser tab, files/diff, and the running coding thread.

The ChatGPT page should be treated as the user's interface, not as another page for Codex to automate.

This desktop convenience is independent from the MCP integration. Failure to embed ChatGPT successfully should not prevent the MCP workflow from working through the user's normal browser.

## Keep the fork delta small

The desired implementation should avoid creating a second orchestration system beside T3.

The preferred delta is:

1. Directly curate the MCP tools this fork advertises.
2. Rewrite their descriptions for the Chat-to-Codex workflow and validate that they do not reference removed tools.
3. Add a small read-only repository-inspection toolkit.
4. Add ChatGPT-specific launch/steering workflow guidance.
5. Make optional desktop browser UX improvements if necessary.

A general-purpose MCP profile system is unnecessary unless implementation work later reveals a concrete reason for one.

Everything else should continue to use upstream T3 services: authentication, provider discovery, thread management, worktrees, persistent thread state, Codex integration, pending requests, model selection, interruption, and steering.

This makes it easier to keep the fork rebased onto upstream T3.

If upstream later gains repository-inspection tools or other pieces that match this workflow, this fork should reuse them where practical. The fork does not need to preserve architectural symmetry with upstream when doing so would make the Chat-to-Codex path more complicated.

## Non-goals

This fork is not trying to:

- Reimplement Codex or Codex App Server.
- Replace T3's provider adapters.
- Build a general autonomous multi-agent hierarchy for ChatGPT.
- Keep ordinary ChatGPT MCP calls open for an entire coding run.
- Teach ChatGPT to poll `t3_thread_wait` on a timer.
- Recreate ChatGPT Work in a custom MCP plugin.
- Automatically wake ordinary ChatGPT on every T3 event.
- Preserve the full upstream MCP tool surface for non-ChatGPT clients.
- Preserve every upstream provider workflow at the ChatGPT plugin boundary.
- Make every T3 feature available to the planning conversation.

The project should optimize the boundary between a conversational planner and a durable coding harness.

## OpenAI product prerequisites and limitations

The exact ChatGPT custom-MCP product surface changes over time and should be rechecked during implementation.

As of October 2026, OpenAI's documentation says [full custom MCP, including write/modify actions, is available to Business and Enterprise/Edu](https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt), while Pro users can connect custom MCP servers with read/fetch permissions. A ChatGPT account/workspace that cannot invoke write-capable custom MCP tools will be able to inspect T3 but will not be able to launch, steer, or interrupt coding threads from Chat.

That is an OpenAI product/account limitation, not a T3 limitation.

OpenAI also documents per-app tool management for custom MCP apps. That is useful as a second safety layer, but this fork should still curate its MCP surface server-side so the model receives a coherent, minimal set of tools and descriptions that never refer to tools the fork has intentionally removed.

## Desired end state

The successful workflow should feel approximately like this:

```text
User:
"Change the cache invalidation behavior."

ChatGPT:
- reads the relevant project files through T3
- identifies two ambiguous behavioral choices
- discusses them with the user
- produces a final implementation specification
- asks for approval

User:
"Go ahead."

ChatGPT:
- resolves the current Codex provider/model/options
- launches one new T3 thread with the approved handoff
- returns the T3 thread link
- ends its turn

Codex in T3:
- implements the handoff
- runs appropriate validation
- asks in T3 only if a material unresolved issue appears

User:
- watches progress in T3
- answers Codex directly there when convenient

Later, optionally:

User to ChatGPT:
"Check what Codex did."

ChatGPT:
- reads the T3 thread
- reads the repository diff
- reviews the result
- discusses any corrections with the user
- steers the existing T3 thread only if requested
```

The key property is that **conversation and implementation can be separated without duplicating the implementation agent or relying on fragile background behavior in ordinary ChatGPT**.

## Upstream T3 documentation

The original T3 documentation remains relevant and should continue to be treated as the implementation baseline:

- [T3 outside-agent / MCP documentation](./docs/user/outside-agents.md)
- [T3 internals overview](./docs/internals/overview.md)
- [T3 permission modes](./docs/user/permission-modes.md)
- [T3 project settings](./docs/user/project-settings.md)
- [Upstream repository](https://github.com/pingdotgg/t3code)

Until code changes are made, installation and runtime behavior remain those of upstream T3 Code.
