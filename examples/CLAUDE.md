# Example CLAUDE.md sections

The agents look for these four sections by name in each repo's `CLAUDE.md`. Add them below whatever `/init` generates, or run `/setup-agents` to draft them from your code and git history. Everything here is a sample — replace it with your project's real commands and history.

## Commands
- Install: `npm ci`
- Lint one file (no auto-fix): `npx eslint <file>`
- Type-check: `npx tsc --noEmit`
- Deploy (developer only, never agents): `npx serverless deploy --stage dev`

## Finishing a change
- New endpoint: wired in the service config and added to the API docs.
- New field: added to the schema file, or it is silently dropped.
- New env var or AWS permission: added to `serverless.yml` for every stage; needs a redeploy.

## Pitfalls that have shipped
- Login/logout across browser tabs: reason about the second tab.
- Retries: a job that sends a notification must not send twice on re-run.
- Timestamps: store UTC, convert to the user's timezone only when displaying.
- Module-level clients survive between Lambda invocations: test the second call, not just the first.

### Related repos
- `../web-admin` — admin web app (in scope; read its `CLAUDE.md` before planning UI work)
- `../web-portal` — customer web app (in scope)
- `../mobile-app` — out of scope for the agents
