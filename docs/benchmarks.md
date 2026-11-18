# Benchmarks

Four real features from one production codebase (a serverless Node.js backend plus React web apps) were replayed from before they shipped, comparing plain Claude Code, Claude Code's built-in subagents, and these custom agents. Three are shown here.

> One codebase, one developer, single runs; models and ticket wording changed between some runs. Treat these as strong signals, not benchmarks.

## Time

| Feature | Plain Claude Code | Built-in subagents | Custom agents |
|---|---|---|---|
| Small change following an existing flow | ~15m | ~20m | ~20m |
| Device export (background job + loading state) | 1h | ~45m | ~30m\* |
| Duplicate-session feature (backend + web app) | not run | 1h 30m | **50m** (optimized agents) |

\*Excludes live testing and deployment. The first two custom-agent results use the first version of the agents; the duplicate-session result uses the optimized version.

## Tokens

Counted two ways — compare within a row, not down the table.

| Feature | What was counted | Plain Claude Code | Built-in subagents | Custom agents |
|---|---|---|---|---|
| Small change | Main session only | +76k | +82k, plus ~210k inside Explore agents | +75k |
| Device export | Main session only | +160k | +160k | +63k |
| Duplicate-session feature | All agents combined | not run | 40.3M | **14.1M** (optimized) |

- **Subagents aren't free.** On the small change, two Explore agents spent ~210k tokens of their own.
- **"Main session only" flatters subagent setups**, since it leaves out what subagents spend — treat +63k as a lower bound.
- **Tokens aren't cost.** Cache reads were 92–99% of every total, and on the duplicate-session feature the built-in subagents actually used the fewest fresh (non-cached) tokens.

## Quality

- The critic caught **17 defects before any code ran** — for example an off-by-one that would have failed exports of exactly 3,000 records.
- Two confident claims from built-in Explore agents about the codebase turned out to be false.
- One planner default ("newest sign-in wins, no prompt") hid a product decision until a developer noticed — which is why the planner now raises every decision that changes the build.

## From a failed first version to a 50-minute build

The first version of the custom agents, on the duplicate-session feature, ran **3h 52m and 51.2M tokens**, hit the 5-hour plan limit once, and was ended as unsuccessful: UI bugs and core backend logic issues the agents couldn't resolve. The critic caught real defects slice by slice but missed whole-feature gaps — e.g. a session check that covered only 5 of the app's 19 data-fetching files.

The agents were rewritten against Anthropic's agent and Claude Code guidance and OWASP's AI Agent Security Cheat Sheet. The changes that mattered most:

1. **Limits** — turn caps on every agent; at most two rounds for plan fixes and slice fixes.
2. **Project knowledge moved into `CLAUDE.md`** — out of the prompts, so the same agents work anywhere.
3. **Every crucial decision raised** — no quiet defaults.
4. **A critic that proves its findings** — every finding needs a check that tries to disprove it.
5. **Implementers that check themselves** — they lint what they changed and list exactly which files they touched.

Result: the same feature in **50 minutes and 14.1M tokens**, with problems surfacing in review instead of live testing.

## References

- [Claude Code docs: Subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code docs: Best practices](https://code.claude.com/docs/en/best-practices)
- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Anthropic: How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- [OWASP: AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html)
- [Cognition: Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) — a useful counterpoint
