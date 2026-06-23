# OpenCode Agent Instructions

These rules apply ONLY to OpenCode agents. They are loaded from `~/.config/opencode/AGENTS.md` (global user-level, not shared with other agent tools).

---

## 1. Background Tasks via Subagents

### Current limitation (honest status)

**Both `bash` AND the Task tool block the main conversation.** When a subagent is running, the user CANNOT continue chatting — the whole agent turn is synchronous. This is a known OpenCode limitation tracked in:
- [#5887](https://github.com/anomalyco/opencode/issues/5887) — True Async/Background Sub-Agent Delegation
- [#28034](https://github.com/anomalyco/opencode/issues/28034) — Non-blocking background task dispatch (like Claude Code's ^+B)
- [#29638](https://github.com/anomalyco/opencode/issues/29638) — Subagents dispatched sequentially instead of in parallel

Subagents also run **sequentially**, not in parallel — even when multiple Task calls are sent in one message.

### Why delegate anyway (even though it blocks)

Despite being blocking, subagents are still useful for:
- **Logical separation** — long builds/deploys run in their own session with their own context
- **Reduced main-session context bloat** — subagent output goes to a child session, not the main conversation
- **Navigable sessions** — user can use `session_child_first` (Leader+Down) to view subagent work, then return with `session_parent` (Up)

### When to delegate

Delegate to a subagent for long-running work, especially when the output is large (build logs, deployment output):

- Builds, deploys, compilations (`npm run build`, `docker build`, Coolify deploys, etc.)
- Long-running CLI commands (compilation, Docker ops, npm install on large projects)
- Batch operations (bulk file processing, multi-step image generation, bulk API calls)
- Any Coolify MCP deploy/restart/stop operations
- Cerebrium deploys — use `general` subagent with `scripts/cloud-run-log.ps1 <label> -- cerebrium deploy ...`
- Batch image generation — use `general` subagent, set timeout 900000-1200000ms

### Subagent types

- **`general`** — full tool access (bash, file writes, MCP). Use for builds, deploys, any write operations.
- **`explore`** — read-only, fast. Use for codebase research, finding files, answering questions.

### Rules

1. **Long work goes to Task, never to direct bash.** Delegate anything >15s to a `general` subagent. Even though it blocks, it keeps the main session context cleaner.
2. **Include explicit timeout instructions in the subagent prompt.** The subagent runs its own bash calls — tell it what timeout to use (e.g. "Set bash tool timeout to 600000ms").
3. **Tell the user the work is running and will take time.** Be upfront that the conversation is blocked until it finishes.
4. **Report results when they come back.** Summarize the outcome to the user.
5. **Always send multiple independent Task calls in a single message.** When tasks don't depend on each other (e.g. "build project A AND search codebase for X"), launch both as separate Task calls in the same message. Currently they run sequentially (bug #29638), but this is the correct pattern that will run in parallel once the bug is fixed. Sending them together is always better than waiting to launch the second one after the first finishes.

### If the user needs to work in parallel

OpenCode supports **multi-session**. The user can open a second terminal and run `opencode` again in the same project. That's the only real workaround for true parallel work today.

---

## 2. Todo List Preservation (CRITICAL)

The `todowrite` tool **replaces the entire todo list** each call — there is no "append" or "update single item" operation. This means you MUST explicitly preserve existing tasks or they are lost.

### Rules

1. **NEVER call `todowrite` without including ALL existing items** — both pending and in_progress. Read the current list first if unsure.
2. **Only mark `completed` when genuinely done and verified.** Do NOT mark tasks completed based on intent or assumption — only after the actual work is finished (e.g., code is written AND tests pass AND lint is clean).
3. **When the user changes direction or adds a new task:** keep all existing `pending` and `in_progress` tasks in the list. Add the new task(s) as additional items. Do NOT delete or overwrite existing tasks unless they are genuinely `completed`.
4. **When the user interrupts:** preserve all existing tasks. Add a new task for the interruption/request if needed. Never wipe the list.
5. **Priorities matter.** Keep existing priority levels. New urgent tasks can be `high`, but don't downgrade existing tasks without reason.
6. **Do NOT deduplicate or merge** items into vaguer descriptions. Keep specific task descriptions.
7. **Do NOT delete items** unless the user explicitly says to remove them or they are genuinely completed.
8. **Sub-items are OK.** If the user says "also do X and Y and Z", add three separate items, not one vague item.

### Correct pattern

```
Current todo list:
  1. [in_progress, high] Fix authentication bug
  2. [pending, medium] Update documentation

User says: "Also check the deployment logs"

CORRECT todowrite call:
  1. [in_progress, high] Fix authentication bug        ← preserved
  2. [pending, medium] Update documentation             ← preserved
  3. [in_progress, high] Check deployment logs          ← new task added

WRONG (DO NOT DO THIS):
  1. [in_progress, high] Check deployment logs          ← existing tasks lost!
```

### Verification checklist (before EVERY `todowrite` call)

- [ ] Do I have the current full list? (If not, look at the previous todowrite call in conversation.)
- [ ] Am I including ALL existing `pending` and `in_progress` items?
- [ ] Am I only marking `completed` if actually verified done?
- [ ] Am I adding new items, not replacing the whole list?

---

## 3. Bash Timeout Management (CRITICAL)

The `bash` tool has a **default timeout of 600 seconds (10 minutes)**. When the tool times out, it **kills the running process** — which can waste work that was already in progress (e.g., a server generating an image, a deployment mid-stream, a build almost done).

### Rules

1. **Always set `timeout` explicitly when the task may take longer than 10 minutes.** Use the `timeout` parameter on the `bash` tool (in milliseconds). Examples:
   - App builds: `timeout: 600000` (10 min) to `timeout: 1200000` (20 min)
   - Deployments (Coolify, Modal, Cerebrium, Docker): `timeout: 900000` (15 min) to `timeout: 1800000` (30 min)
   - Image generation scripts (especially batch/multiple images): `timeout: 1800000` (30 min) or more
   - `npm install` on large projects: `timeout: 600000` (10 min)
   - When in doubt, set a higher timeout. It's better to wait than to kill a process mid-execution.

2. **A timed-out process = wasted work.** If a server was generating an image and the bash tool kills the connection, the image may have been generated but never downloaded. If a deployment was 90% done, all progress is lost. Always prevent this.

3. **For very long tasks (>10 min), prefer subagents.** Delegate to a `general` subagent via the Task tool and include explicit timeout instructions in the prompt. Example:
   ```
   Task(subagent_type="general", prompt="Run the image generation script with timeout 1800000ms. The script generates 10 images and may take up to 25 minutes. Report results when done.")
   ```

4. **Parallel tasks still need proper timeouts.** When launching multiple bash commands in parallel, set appropriate timeouts on each one individually.

### Quick reference

| Task type | Suggested timeout |
|---|---|
| Quick commands (git, ls, grep) | default (120000ms) |
| `npm install` | 600000ms (10 min) |
| App build | 600000-1200000ms (10-20 min) |
| Deployment | 900000-1800000ms (15-30 min) |
| Image generation (single) | 600000-900000ms (10-15 min) |
| Image generation (batch) | 1800000-3600000ms (30-60 min) |
| Docker build | 1200000-1800000ms (20-30 min) |

---

## 4. General Behavior

- Be concise. The user is on a CLI. Short answers preferred.
- Use parallel tool calls when operations are independent.
- Delete temporary scripts after they execute successfully.
- Don't add comments to code unless explicitly asked.


