# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A documentation-only repository. It holds reference material about Claude Code
agent teams — there is no application code, build system, or test suite. Work
here means editing Markdown.

## Commands

There is no build, lint, or test tooling. Common tasks:

- Preview a doc: open the `.md` file in any Markdown viewer, or `glow DOCS/agent-teams.md`.
- Publish a change: commit on a branch, `git push`, `gh pr create`, then merge
  the PR (the established workflow — see the merge history, PRs #1 and #2, which
  used branch names like `claude/<topic>-<hash>`).

## Structure

- `DOCS/agent-teams.md` — the master reference guide for designing and running
  Claude Code agent teams. Sourced from
  code.claude.com/docs/en/agent-teams, kept as a curated, version-annotated
  snapshot (notes which Claude Code version each behavior changed in).
- `.claude/settings.local.json` — sets `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`,
  enabling the experimental agent-teams feature for sessions in this repo.
- `README.md` — currently just the repo name.

## Working notes

- The agent-teams doc tracks a moving experimental feature. When updating it,
  preserve the per-version annotations ("as of v2.1.x", "removed in v2.1.234",
  etc.) — they are the point of keeping a local copy rather than linking out.
- Enabling `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` has a side effect: any named
  subagent launches as a teammate, so a team can form during ordinary
  delegation. See the "Claude spawns teammates instead of subagents" section of
  the doc.
