# Agent Teams — Master Reference Guide

> Source: [code.claude.com/docs/en/agent-teams](https://code.claude.com/docs/en/agent-teams)
> Purpose: reference material for designing and running effective Claude Code agent teams in this project.

**Status:** Agent teams are experimental and disabled by default. This repo enables them via `.claude/settings.local.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Without that variable, no team is set up at session start, no team directories are written, and Claude does not spawn or propose teammates.

Doc snapshot: as of Claude Code v2.1.178+. With the env var set, spawning a teammate needs no setup step and cleanup is automatic on exit. Before v2.1.178, teams required explicit `TeamCreate`/`TeamDelete` calls — those tools no longer exist.

---

## Table of contents

1. [What agent teams are](#what-agent-teams-are)
2. [When to use them (and when not to)](#when-to-use-them-and-when-not-to)
3. [Agent teams vs. subagents](#agent-teams-vs-subagents)
4. [Enabling agent teams](#enabling-agent-teams)
5. [Starting a team](#starting-a-team)
6. [Controlling a team](#controlling-a-team)
7. [How it works under the hood (architecture)](#how-it-works-under-the-hood-architecture)
8. [Permissions](#permissions)
9. [Context and communication](#context-and-communication)
10. [Token usage](#token-usage)
11. [Use case examples](#use-case-examples)
12. [Best practices](#best-practices)
13. [Troubleshooting](#troubleshooting)
14. [Known limitations](#known-limitations)
15. [Quick-reference checklist for building effective teams](#quick-reference-checklist-for-building-effective-teams)

---

## What agent teams are

Agent teams coordinate **multiple independent Claude Code sessions** working together on one task:

- One session is the **team lead** — it spawns teammates, assigns/coordinates work, and synthesizes results.
- **Teammates** work independently, each in its own full context window, and can message each other directly (not just report back to the lead).
- You (the human) can talk to the lead *or* to any individual teammate directly — you don't have to go through the lead.

This is distinct from [subagents](https://code.claude.com/docs/en/sub-agents), which run inside a single session and report a result back to the caller.

---

## When to use them (and when not to)

Agent teams add real coordination overhead and cost significantly more tokens than a single session. They pay off when teammates can work **independently** and benefit from cross-talk. They are a poor fit for sequential work, same-file edits, or heavily-dependent task chains.

### Strong use cases

| Scenario | Why it works |
|---|---|
| **Research & review** | Multiple teammates investigate different aspects in parallel, then share/challenge findings |
| **New modules or features** | Each teammate owns a separate piece — no stepping on each other |
| **Debugging with competing hypotheses** | Teammates test different theories in parallel and converge faster |
| **Cross-layer coordination** | Frontend / backend / tests each owned by a different teammate |

### Weak use cases (avoid teams)

- Sequential tasks with hard dependencies
- Edits concentrated in the same file(s)
- Small, quick tasks where coordination overhead dwarfs the work itself
- Anything where a single session or a subagent would resolve it just as well

---

## Agent teams vs. subagents

| | Subagents | Agent teams |
|---|---|---|
| **Context** | Own context window; result returns to the caller | Own context window; fully independent |
| **Communication** | Returns a result to the caller. Named subagents can also message each other | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Self-coordination via messages + shared task list (for agents with Task tools) |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion and collaboration |
| **Token cost** | Lower — results summarized back to main context | Higher — each teammate is a separate Claude instance |

**Rule of thumb:** use subagents for quick, focused workers that report back; use agent teams when teammates need to share findings, challenge each other, and coordinate on their own.

> Note: for separate sessions that pass messages to each other *without* forming a team, see [cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging).

---

## Enabling agent teams

Set the environment variable, either in the shell or in `settings.json` (any level: user, project, or local):

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

**Side effect to know about:** enabling this changes ordinary delegation too. Claude may name a subagent on its own during normal delegation, and while agent teams are enabled, *any named subagent launches as a teammate* — so a team can form even when you never asked for one. See [Claude spawns teammates instead of subagents](#claude-spawns-teammates-instead-of-subagents) if this is unwanted.

**Requires an interactive session.** In non-interactive mode (`-p` flag, or Agent SDK sessions), Claude does not spawn teammates — a named subagent runs as an ordinary subagent even with the flag set.

---

## Starting a team

Describe the task and desired teammates in natural language:

```text
I'm designing a CLI tool that helps developers track TODO comments across
their codebase. Spawn three teammates to explore this from different angles:
one on UX, one on technical architecture, one playing devil's advocate.
```

This works well because the three roles are independent and don't block on each other.

Claude will populate a shared task list, spawn teammates per perspective, have them explore, and synthesize findings when done.

> **Caveat:** Claude may use subagents instead of forming a team, and both appear in the same agent panel — so the panel alone doesn't confirm a team formed. If you wanted a team and got subagents, ask again and explicitly request an **agent team**.

### The agent panel (lead's terminal)

Below the prompt input:

- **↑ / ↓** — select a teammate
- **Enter** — open the selected teammate's transcript and message it directly
- **Escape** — interrupt the selected teammate's current turn

Behavior notes by version:
- **v2.1.199+**: an idle teammate's row stays visible while *any* teammate/subagent is still working. Once everything is idle, idle rows hide after 30s and reappear on the teammate's next turn (it keeps running/addressable while hidden).
- **v2.1.181–v2.1.198**: a row hid 30s after *its own* turn ended, even if others were still working.
- **< v2.1.181**: idle rows were never hidden.
- More than 3 idle teammates at once collapse into one row (e.g. `2 idle agents`) — select + Enter to expand, Esc to collapse. Working/failed teammates and the one you're viewing always keep their own row.

For a dedicated pane per teammate, see [Choose a display mode](#choose-a-display-mode).

---

## Controlling a team

Talk to the lead in natural language — it handles coordination, task assignment, and delegation.

### Choose a display mode

| Mode | Description | Requirements |
|---|---|---|
| `in-process` (default) | All teammates run in your main terminal; select via agent panel | Works anywhere |
| Split panes | Each teammate gets its own pane; click in to interact directly | tmux, or iTerm2 |

Set globally in `~/.claude/settings.json`:

```json
{
  "teammateMode": "auto"
}
```

Or per-session:

```bash
claude --teammate-mode auto
```
(`--teammate-mode` is experimental and not listed in `claude --help`.)

Mode values:
- **`in-process`** — default since v2.1.179. (Before that, default was `auto`, so upgraded sessions that used to open split panes now stay in one terminal unless set explicitly.)
- **`auto`** — split panes if already inside tmux, or iTerm2 with `it2` CLI installed; otherwise falls back to in-process.
- **`tmux`** — forces split-pane mode; auto-detects tmux vs. iTerm2.
- **`iterm2`** (v2.1.186+) — forces iTerm2 native split panes explicitly; errors with an install command if `it2` is missing.

Install requirements for split panes:
- **tmux**: install via your system package manager ([tmux wiki](https://github.com/tmux/tmux/wiki/Installing)). Works best on macOS; `tmux -CC` in iTerm2 is the suggested entrypoint.
- **iTerm2**: install the [`it2` CLI](https://github.com/mkusaka/it2), then enable **iTerm2 → Settings → General → Magic → Enable Python API**.

### Specify teammates and models

```text
Spawn 4 teammates to refactor these modules in parallel. Use Sonnet for
each teammate.
```

- If your prompt doesn't name a model, teammates run on the **lead's current model**, unless `CLAUDE_CODE_SUBAGENT_MODEL` is set.
- `teammateDefaultModel` setting was **removed in v2.1.234** — name the model in the prompt or set `CLAUDE_CODE_SUBAGENT_MODEL` instead.
- Requested models are checked against your org's `availableModels` allowlist:
  - A blocked **family alias** (e.g. `opus`) on the Anthropic API / Claude Platform on AWS → substitutes the newest allowed version of that family.
  - Any other blocked value (or a family alias where substitution doesn't apply, or with no permitted version) → teammate runs on the **lead's model**.
- Teammates inherit the lead's **effort level**. In split-pane mode this has applied since v2.1.186 (earlier versions didn't pass it through).

### Require plan approval for teammates

For complex/risky work, require a plan-then-approve flow:

```text
Spawn an architect teammate to refactor the authentication module.
Require plan approval before they make any changes.
```

Flow: teammate plans (read-only) → sends plan approval request to lead → lead approves or rejects with feedback → if rejected, teammate revises and resubmits → once approved, teammate exits plan mode and implements.

The **lead** decides approval autonomously — steer it with explicit criteria in your prompt, e.g. *"only approve plans that include test coverage"* or *"reject plans that modify the database schema."*

### Talk to teammates directly

Each teammate is a full, independent Claude Code session.

- **In-process**: ↑/↓ to select, Enter to view/message. `x` on a selected teammate stops it. Ctrl+T toggles the task list.
- **Split-pane**: click into the teammate's pane directly.

While viewing an in-process teammate, plain text and skills go to *that* teammate — but built-in commands still run in the **lead's** session.

- A teammate's model and fast mode are **fixed at spawn time** — `/model` and `/fast` while viewing a teammate only change the **lead's** settings (v2.1.199+ shows a notice; earlier versions silently applied to the lead).
- `/effort` **does** apply to the viewed teammate's later turns, since teammates follow the lead's effort level.

### Assign and claim tasks

The shared task list coordinates work. States: **pending → in progress → completed**. Tasks may depend on other tasks — a pending task with unresolved dependencies can't be claimed until those dependencies complete. (Dependent-task unblocking is automatic.)

Agents **without** the Task tools coordinate via messages instead of the shared list.

- **Lead assigns**: tell the lead which task goes to which teammate.
- **Self-claim**: after finishing a task, a teammate can pick up the next unassigned, unblocked task itself.

Task claiming uses file locking to avoid race conditions when multiple teammates try to claim the same task at once.

### Shut down teammates

Refer to the teammate by name:

```text
Ask the researcher teammate to shut down
```

The lead sends a shutdown request; the teammate can approve (exit gracefully) or reject with an explanation. Shared team directories are cleaned up automatically when the session ends — no separate cleanup step needed (see [Architecture](#how-it-works-under-the-hood-architecture) for what's removed vs. persisted).

### Enforce quality gates with hooks

- **`TeammateIdle`** — runs when a teammate is about to go idle. Exit code 2 → send feedback and keep the teammate working.
- **`TaskCreated`** — runs when a task is being created. Exit code 2 → prevent creation and send feedback.
- **`TaskCompleted`** — runs when a task is being marked complete. Exit code 2 → prevent completion and send feedback.

---

## How it works under the hood (architecture)

### How Claude starts agent teams

A teammate launches when Claude calls the Agent tool **with a `name`** while agent teams are enabled — no confirmation prompt. Claude also names ordinary subagents on its own (so it can message them later), and while agent teams are enabled, **any named subagent launches as a teammate**. If you want subagents instead, [turn agent teams off](#claude-spawns-teammates-instead-of-subagents).

### Components

| Component | Role |
|---|---|
| **Team lead** | The main session; spawns teammates and coordinates work |
| **Teammates** | Separate Claude Code instances, each working assigned tasks |
| **Task list** | Shared work-item list that teammates claim and complete |
| **Mailbox** | Messaging system between agents |

### Storage layout

- **Mailbox**: `~/.claude/teams/{team-name}/inboxes/{agent-name}.json` (JSON file per agent). Claude Code validates every entry on read; malformed entries are reported as errors and dropped — valid messages still deliver. *(Before v2.1.207: one malformed entry caused a repeated per-second error and blocked delivery until the file was deleted manually.)*
- **Team config**: `~/.claude/teams/{team-name}/config.json` — generated automatically at session startup, updated as teammates join/idle/leave, **removed when the session ends**. Contains a `members` array (name + agent ID). The lead's entry always has agent type `team-lead`; a teammate's entry carries whatever type the lead named (built-in or subagent definition), or omits the field if none was named. **Don't hand-edit or pre-author this file** — it's overwritten on the next state update, and a project-level `.claude/teams/teams.json` is *not* recognized as config (Claude treats it as an ordinary file).
- **Task list**: `~/.claude/tasks/{team-name}/` — persists locally (never uploaded), so resumed sessions keep their tasks. Retention follows the same `cleanupPeriodDays` setting used for session transcripts.
- **Team name**: session-derived — `session-` + first 8 chars of the session ID.

### Message delivery guarantee

A message (plain text or a structured protocol message like plan-approval / shutdown) is reported as **sent** only when the write to the recipient's mailbox file succeeds. If the write fails (disk full, mailbox dir not writable), the **sender** gets an error and nothing is sent.

### Reusable roles: subagent definitions as teammates

Reference any [subagent](https://code.claude.com/docs/en/sub-agents) type (project, user, plugin, or CLI-defined) when spawning a teammate — define a role once (e.g. `security-reviewer`) and reuse it as both a delegated subagent and a team teammate:

```text
Spawn a teammate using the security-reviewer agent type to audit the auth module.
```

- The teammate honors that definition's `tools` allowlist and `model`.
- The definition's body is **appended** to the teammate's system prompt (additional instructions, not a replacement).
- Claude Code auto-adds `SendMessage` to the allowlist for in-process teammates, and (in sessions with Task tools) also adds `TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate`.
- **Not applied**: a subagent definition's `skills` and `mcpServers` frontmatter fields are ignored when it runs as a teammate — teammates load skills/MCP servers from your project & user settings like a normal session.

---

## Permissions

- Teammates **start with the lead's permission settings** (e.g. `--dangerously-skip-permissions` propagates to all teammates).
- You can change an individual teammate's mode *after* spawning, but **not** set per-teammate modes at spawn time.
- Teammate permission prompts surface **in the lead session** — approve them there.
- **Plan approval is the designed exception**: the lead session grants teammate plan approvals itself, without prompting you separately.

### Messages between agents

- When Agent A messages Agent B via `SendMessage`, Claude Code tells B the message came from **another Claude session**, not from the human. B cannot use it to approve a permission prompt or supply consent on your behalf, and a teammate denied an action can't relay through another teammate to bypass the check. Same rule applies to messages arriving from other Claude Code sessions outside the team entirely.
- In **auto mode**, the classifier applies two checks to inter-agent messages:
  1. Treats a relayed "approval" claim from another agent as **untrusted input**, not confirmation from the human.
  2. Reviews every message before delivery (plain or structured/protocol) — a blocked message never reaches the recipient.

---

## Context and communication

Each teammate has its own context window. On spawn, a teammate loads the same project context as a normal session — **CLAUDE.md, MCP servers, skills** — plus the spawn prompt from the lead. **The lead's conversation history does NOT carry over.**

How information flows:

- **Automatic message delivery** — messages between teammates deliver automatically; the lead doesn't poll.
- **Idle notifications** — when a teammate finishes and stops, it auto-notifies the lead. The notification carries **no output** — the teammate must message the lead or update the task list to share results. (v2.1.198+: a teammate whose turn ends on an API error notifies the lead of the failure + error text, instead of appearing to finish normally.)
- **Shared task list** — visible to agents with Task tools; they can see status and claim work.
- **Teammate messaging** — message one teammate by name; to reach everyone, send one message per recipient (no broadcast).

The lead names every teammate at spawn time; any teammate can message any other by name. **Tell the lead what to name each teammate** in your spawn instruction if you want predictable names to reference later.

---

## Token usage

Agent teams use **significantly more tokens** than a single session — each teammate is a separate context window, and usage scales with the number of active teammates.

- Worth it for: research, review, new feature work.
- Not worth it for: routine/small tasks — use a single session instead.

**Caching note:** an in-process teammate's requests fall **outside** the main conversation's cache TTL bucket, so its cache defaults to a 5-minute hold (even on a Claude subscription). To extend to 1 hour, set `subagentPromptCacheTtl: "1h"` — note the API bills 1-hour cache writes at a higher rate.

---

## Use case examples

### Parallel code review

```text
Spawn three teammates to review PR #142:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```

Why it works: a single reviewer tends to fixate on one issue type at a time. Splitting by domain (security / performance / tests) means all three get simultaneous, thorough attention. The lead synthesizes findings once all three finish.

### Competing-hypothesis debugging

```text
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific
debate. Update the findings doc with whatever consensus emerges.
```

Why it works: sequential investigation anchors on the first plausible theory found. Making teammates **explicitly adversarial** — each trying to disprove the others — counters that bias. The theory that survives active attempts to kill it is much more likely to be the real root cause.

---

## Best practices

### 1. Give teammates enough context
Teammates auto-load project context (CLAUDE.md, MCP servers, skills) but **not** the lead's conversation history. Put task-specific details directly in the spawn prompt:

```text
Spawn a security reviewer teammate with the prompt: "Review the authentication module
at src/auth/ for security vulnerabilities. Focus on token handling, session
management, and input validation. The app uses JWT tokens stored in
httpOnly cookies. Report any issues with severity ratings."
```

### 2. Choose an appropriate team size
No hard limit, but real constraints apply:
- **Token cost scales linearly** with teammate count.
- **Coordination overhead increases** with more teammates (more comms, more task juggling, more conflict potential).
- **Diminishing returns** past a certain point — more teammates ≠ proportionally faster.

**Start with 3–5 teammates** for most workflows. E.g., 15 independent tasks → 3 teammates is a good starting point. Scale up only when work genuinely benefits from simultaneous execution. Three focused teammates usually beat five scattered ones.

### 3. Size tasks appropriately
- **Too small** → coordination overhead exceeds the benefit.
- **Too large** → teammates go too long without check-ins, risking wasted effort.
- **Just right** → self-contained units with a clear deliverable (a function, a test file, a review).

Aim for **5–6 tasks per teammate** to keep everyone productive and let the lead reassign if someone gets stuck. If the lead isn't generating enough tasks, explicitly ask it to split work into smaller pieces.

### 4. Wait for teammates to finish
If the lead starts implementing itself instead of waiting on delegated work:

```text
Wait for your teammates to complete their tasks before proceeding
```

### 5. Start with research and review
If new to agent teams, start with tasks that have clear boundaries and don't require writing code — reviewing a PR, researching a library, investigating a bug. This demonstrates the value of parallel exploration without the coordination challenges of parallel *implementation*.

### 6. Avoid file conflicts
Two teammates editing the same file → overwrites. **Partition work so each teammate owns a distinct set of files.**

### 7. Monitor and steer
Check in on progress, redirect approaches that aren't working, synthesize findings as they arrive. An unattended team running too long risks wasted effort.

---

## Troubleshooting

### Teammates not appearing
- In-process mode: check the agent panel below the prompt (↑/↓ to select, Enter to view).
- A row that vanished after sitting idle is **hidden, not stopped**. Idle rows hide 30s after the whole panel goes idle and reappear on the teammate's next turn. >3 idle teammates collapse into an `N idle agents` row (Enter to expand). **Message the teammate by name** to bring a hidden row back.
- Confirm the task was actually complex enough for Claude to decide a team was warranted.
- If you requested split panes explicitly, verify tmux is on PATH: `which tmux`.
- For iTerm2: verify `it2` CLI is installed and Python API is enabled in preferences.

### Claude spawns teammates instead of subagents
While agent teams are enabled, any subagent Claude names in the lead session launches as a **teammate** — this can happen during ordinary delegation you never framed as team work. This matters because subagents and teammates report back differently:
- **Subagents** → Claude receives the result directly when it completes.
- **Teammates** → idle notification reports only that the teammate *stopped*, no output.

An orchestration flow expecting a subagent's return value can **stall** if it silently became a teammate.

**Fix — disable agent teams:**
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "0"
  }
}
```
No new session needed — Claude Code reapplies settings-file `env` changes to the running session on save, and re-reads the variable each time it spawns a subagent.

**Settings precedence caveat:** setting `0` in **user** settings can be overridden by higher-precedence sources:
- Project settings, local settings, or a `--settings` payload that sets it to `1` — these apply *after* user settings and win.
- **Managed settings** apply after everything else — if your org enables agent teams there, you need an admin to change it.

After disabling, Claude may still name subagents — the name still works as a `SendMessage` address, and Claude receives the result when the subagent completes.

### Too many permission prompts
Teammate permission requests bubble up to the lead, causing friction. **Pre-approve common operations** in permission settings before spawning teammates.

### Agents stopping early
Teammates may stop on errors instead of recovering. Select the teammate (Enter in in-process, click pane in split mode) to inspect, then either:
- Give additional instructions directly, or
- Spawn a replacement teammate to continue the work.

A message from the lead or another teammate **wakes** an in-process teammate that's waiting to retry a failed API request — it retries immediately instead of waiting out the full delay.

The lead itself can also decide the team is "finished" prematurely — tell it to keep going if so.

### Orphaned tmux sessions
If a tmux session persists after the Claude Code session ends:
```bash
tmux ls
tmux kill-session -t <session-name>
```

---

## Known limitations

Agent teams are **experimental**. Be aware of:

- **No session resumption for in-process teammates** — `/resume` and `/rewind` do not restore them. After resuming, the lead may try to message teammates that no longer exist; tell it to spawn new ones.
- **Task status can lag** — teammates sometimes fail to mark tasks complete, blocking dependents. If a task looks stuck, verify the work is actually done and update status manually, or nudge the lead.
- **Shutdown can be slow** — teammates finish their current request/tool call before shutting down.
- **One team per session** — exactly one team, scoped to that session; no additional named teams, no sharing a team across sessions.
- **No nested teams** — teammates cannot spawn their own teammates; only the lead manages the team.
- **No background subagents from in-process teammates** — a teammate's own subagents always run in the foreground (a teammate's background work can't outlive the lead's process). A subagent definition with `background: true` errors when spawned by a teammate; a teammate's `run_in_background: true` request also fails or silently runs in the foreground. Subagents launched from the *main* conversation still follow the normal foreground/background default.
- **Lead is fixed** — the main session is lead for its lifetime; no promoting a teammate to lead or transferring leadership.
- **Permissions set at spawn** — all teammates start with the lead's permission mode; individual modes can be changed after spawning but not set per-teammate at spawn time.
- **Split panes require tmux or iTerm2** — default in-process mode works anywhere, but split-pane mode is **not supported** in VS Code's integrated terminal, Windows Terminal, or Ghostty.

---

## Quick-reference checklist for building effective teams

Use this before spawning a team:

- [ ] **Is this actually parallelizable?** If the work is sequential or touches shared files, use a single session or subagents instead.
- [ ] **Team size 3–5** to start. Scale only if the work genuinely benefits from more simultaneous workers.
- [ ] **Partition ownership** — assign each teammate a distinct file/module/domain to avoid overwrite conflicts.
- [ ] **Front-load context in the spawn prompt** — teammates don't inherit the lead's conversation history. Be specific: what to look at, what constraints exist, what "done" looks like.
- [ ] **Name teammates explicitly** in the spawn instruction if you'll want to address them directly later.
- [ ] **Size tasks to 5–6 per teammate**, each a self-contained deliverable (a function, a test file, a review).
- [ ] **Use adversarial framing for debugging** — "try to disprove each other's theories" surfaces the real root cause faster than sequential investigation.
- [ ] **Use plan approval for risky/complex changes** — give the lead explicit approval criteria in the prompt.
- [ ] **Pre-approve common permissions** before spawning to cut down on prompt friction.
- [ ] **Reuse subagent definitions** for recurring roles (e.g. `security-reviewer`) instead of re-describing them each time.
- [ ] **Monitor actively** — don't let a team run unattended for long stretches; check in, redirect, and synthesize as results land.
- [ ] **Explicitly ask the lead to wait** for teammates rather than implementing solo, if you notice it skipping delegation.
- [ ] **Confirm a team actually formed** — check the agent panel; if Claude used subagents instead, ask again and say "agent team" explicitly.
- [ ] **Budget for token cost** — each teammate is a full separate context window; don't spin up a team for routine/small tasks.

---

## Related / next steps

- [Subagents](https://code.claude.com/docs/en/sub-agents) — lightweight delegation within a single session, no inter-agent coordination needed.
- [Git worktrees](https://code.claude.com/docs/en/worktrees) — manual parallel sessions without automated team coordination.
- [Cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging) — message-passing between separate sessions without forming a team.
- [Settings reference](https://code.claude.com/docs/en/settings-reference) — `teammateMode`, `subagentPromptCacheTtl`, `cleanupPeriodDays`, etc.
- [Hooks](https://code.claude.com/docs/en/hooks) — `TeammateIdle`, `TaskCreated`, `TaskCompleted` payloads and quality-gate patterns.
