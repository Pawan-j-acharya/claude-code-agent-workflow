---
description: Draft the four CLAUDE.md sections the agent workflow reads (Commands, Finishing a change, Pitfalls that have shipped, Related repos) from this project's code and history
argument-hint: [backend-path] [frontend-path ...]
allowed-tools: Read, Grep, Glob, Edit, Bash(ls:*), Bash(git ls-files:*), Bash(git log:*)
---

The agents (planner, critic, backend, frontend) carry the process. Everything about this project comes from each repo's CLAUDE.md, under four headings the agents look for by name. Your job is to draft those sections from evidence, for the developer to approve. Never edit the agent files.

Arguments: $ARGUMENTS
(first = backend repo path, the rest = frontend repo paths)

## 1. Get the repository locations

If no arguments were given, ask the developer before doing anything else:

- Where is the backend repository? (default: the current directory)
- Are there frontend repositories? If yes, list every path, one per app.
- Are there repos the agents must never touch (e.g. mobile apps)?

Do not guess locations from folder names nearby.

## 2. Check access and CLAUDE.md

For each path, confirm it exists and you can read it. If a path outside the current directory cannot be read, tell the developer to add it with /add-dir (or relaunch with --add-dir), then continue. If a repo has no CLAUDE.md, ask the developer to run /init there first.

## 3. Draft each section from evidence

Read the code and config in each repo, not just file names.

- ## Commands — build and deploy commands, plus per-file check commands with no auto-fix (e.g. npx eslint <file>). Say what to use if a linter is not installed.
- ## Finishing a change — what "done" means here: where new handlers are wired, which schema file new fields go in, API docs to update, which changes need a redeploy or image rebuild.
- ## Pitfalls that have shipped — scan recent commit messages (git log, about the last 200) for fix, bug and hotfix commits and group them into patterns. One line each, with what to check. These are suggestions for the developer to confirm, not facts.
- ### Related repos — backend CLAUDE.md only, for multi-repo products: relative paths to the other repos, which are out of scope, and a reminder to read their CLAUDE.md.

## 4. Propose before editing

For each repo, show the drafted sections with the evidence for every line (path or commit). List anything you could not confirm as an open question. Wait for approval before editing any CLAUDE.md.

## 5. Write and report

After approval, add or update only those four sections, keeping the headings exactly as written. Keep each CLAUDE.md under about 200 lines. Then report what you added, the evidence, anything left open, and any /add-dir paths the team will need.

Rules: never invent a path, name or command. Never edit anything except the four sections in each CLAUDE.md. Never edit files in .claude/agents/ or .claude/commands/.
