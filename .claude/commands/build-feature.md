---
name: build-feature
description: Run the full plan → gate → implement → audit cycle (planner, critic, backend, frontend agents) on a ticket, slice by slice, instead of relying on the agents being invoked by name in conversation
argument-hint: <ticket text or ticket key> — paste the ticket's job-to-be-done/outcomes/DoD, or a ticket key plus the scope note (which app/listing, if partial)
allowed-tools: Agent, AskUserQuestion, SendMessage, Read, Write, Edit, Bash, ExitPlanMode
---

# Build Feature — planner → critic → implement → audit

Runs the cycle: a ticket goes through a dedicated `planner` agent, gets gated by `critic` before
any code is written, then each delivery slice is implemented by `backend` or `frontend` and
independently diff-audited by `critic` again before you're asked to test it. See
`.claude/agents/planner.md`, `critic.md`, `backend.md`, `frontend.md` for what each one actually
does — this command just sequences them; it doesn't duplicate their instructions here, so read
those files if their behavior seems off rather than assuming this file's summary is authoritative.

If `$ARGUMENTS` is empty, ask for the ticket text before doing anything else.

## Before planning: every repo in scope needs a CLAUDE.md

Before dispatching to `planner`, work out which repos the ticket touches (the backend, and any of
the web apps — the backend's `CLAUDE.md` lists the sibling repos). For each one, check that a
`CLAUDE.md` or `.claude/CLAUDE.md` exists. If one is missing, stop and ask the developer via
`AskUserQuestion` to run `/init` in that repo, then continue once they confirm. `/init` is a
command only the developer can run in an interactive session — do not try to run it yourself, and
do not plan against that repo's conventions without it, since the agents would be guessing them.

## Sequence

1. **Enter Plan mode** (this command should normally be invoked while already in Plan mode —
   if not, treat everything through step 4 as planning: no file edits, no non-readonly tools).

2. **Dispatch to `planner`** with the full ticket text (job-to-be-done, outcomes, Definition of
   Done) and explicit scope — which app/listing/repo this covers if the ticket is broader than
   what's being built now. Don't summarize or trim the ticket before handing it over; the
   planner needs the actual acceptance criteria to build its COS MAPPING against.

3. **If the planner returns any OPEN QUESTIONS marked BLOCKING**, put them to the developer via
   `AskUserQuestion` — do not guess an answer on their behalf, and do not let the planner guess
   either. `AskUserQuestion` takes at most 4 questions per call, so ask in batches of 4, in the
   planner's order, until every one is answered — never drop or default a decision to fit. Fold
   the answers back to the planner via `SendMessage` and get a revised plan before moving on.
   List every NON-BLOCKING question together with the default the planner used in one message to
   the developer, and proceed on those defaults unless they object.

4. **Gate the plan with `critic` (mode 1)** before writing anything to the plan file, passing the
   ticket text (the COS) and the full plan. A `pass` or `pass-with-findings` verdict clears the
   gate — show the developer any findings and move on. If the verdict is `block`, relay the
   specific findings back to the planner via `SendMessage` — don't silently patch the plan
   yourself, since the critic's value depends on the planner (not you) producing the fix and the
   critic re-verifying it. Repeat gate → correct → re-gate until the verdict is no longer
   `block`, **at most 2 correction rounds**. If the critic still blocks after the second
   revision, stop the loop and put the unresolved findings to the developer via `AskUserQuestion`
   (accept as-is, give the planner a direction, or abandon). Tell the user plainly when a round
   blocks and why — this is the workflow working as intended, not a failure to hide.

5. **Write the final, corrected plan** to the Plan mode plan file, including a short note on
   what the critic caught and corrected, then call `ExitPlanMode`.

6. **Per slice**, in the order the plan specifies:
   - Dispatch the slice to `backend` or `frontend` (whichever the slice's files belong to),
     passing the slice's text from the plan verbatim (Pattern, Files, Change, Deploy, Verify) and
     the plan file path for context, and reminding it not to implement other slices. For
     `frontend`, also pass the absolute path of the target web app, of that app's `CLAUDE.md`, and
     of the backend repo: it starts without the launching repo's CLAUDE.md and cannot look them up.
   - Audit the resulting diff with `critic` (mode 2), passing the slice, the plan file path, and
     the implementer's latest full report — including its `FILES TOUCHED` list, which the critic
     diffs instead of the whole working tree.
   - If the audit finds a blocker, send it back to the same implementer agent to fix, then
     re-audit — don't fix it yourself in the main thread. **At most 2 fix rounds per slice**;
     if a blocker survives the second fix, stop and put it to the developer via
     `AskUserQuestion` rather than dispatching a third.
   - Report to the developer: the audit's HOW TO VERIFY steps as concrete on-screen/API steps;
     the implementer's `CHECKS RUN` results and anything it listed under `NOT DONE / DIFFERENTLY`;
     and the critic's SUSPECTED items, which do not affect the verdict, so they can decide whether
     to check them. Then wait for them before moving to the next slice, unless they've said to
     proceed through all slices first and test at the end.

7. After the last slice passes its audit, summarize: what shipped; what's still manual (e.g. a
   Docker rebuild, a redeploy), taken from the plan's OUT OF SCOPE section and each implementer's
   `NOT DONE` and `Deploy` notes; and what the developer should test. Say that nothing is
   committed or deployed — the agents do neither — and that each repo's commit-message format is
   enforced by its own hook.

## When an implementer cannot complete its slice

`backend` and `frontend` stop and report instead of widening scope. When one does, do not
re-plan or improvise a workaround in the main thread. Put the blocker to the developer via
`AskUserQuestion` (adjust the slice, re-plan the remaining slices, or stop), and only continue on
their answer.

## Re-audits: fresh first, then compare

On any critic re-run (plan re-gate or slice re-audit), launch a **new** `critic` with the revised
plan or the new diff and do **not** include the previous report — it anchors the critic on its
old findings instead of looking fresh. Once the fresh report is back, `SendMessage` that same
critic the previous round's FINDINGS and ask it to mark each `resolved`, `still open`, or
`no longer applicable`. Show the developer both the fresh report and the markings.

## When an agent returns partial output

Every agent has a `maxTurns` cap in its frontmatter. An agent that hits it returns output marked
partial. Do not resume it automatically and do not treat partial output as a finished plan,
slice or audit. Tell the developer which agent stopped, at what point, and — for an implementer
— which files it may have left half-edited, then ask via `AskUserQuestion` whether to resume it,
narrow the task, or stop. Count a resumed run toward the round caps above.

## What this command deliberately does not do

- It does not skip the critic gate to save time, even for a ticket that looks small — that
  judgment call is what this command exists to remove.
- It does not let slice N+1 start before slice N's audit comes back without a `block` verdict.
- It does not invent scope beyond what the ticket/developer stated — if the ticket is broader
  than one app or one listing, ask which part is in scope for this run rather than assuming.
