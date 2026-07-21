---
paths:
  - "test/**"
  - "browse/test/**"
  - "package.json"
---

# Testing internals (moved from root CLAUDE.md)

## Env keys in Conductor workspaces

The `GSTACK_*` env-shim (v1.39.2.0+, `lib/conductor-env-shim.ts`) promotes
`GSTACK_ANTHROPIC_API_KEY` / `GSTACK_OPENAI_API_KEY` to their canonical names
inside gstack's TS binaries. Tests run through gstack entrypoints inherit this
promotion automatically. Don't echo the key value to stdout, logs, or shell
history. Gotcha history: the SDK's `Options.env` REPLACES the child's entire
environment — the runner now always passes a COMPLETE hermetic env with per-test
`env:` merged last, so per-test overrides are safe; ambient
`process.env.ANTHROPIC_API_KEY` mutation also works (env builder reads
process.env at call time).

## Hermetic local E2E (default)

Every E2E runner (claude -p, PTY, Agent SDK, codex, gemini) spawns children
through `test/helpers/hermetic-env.ts`: allowlist-scrubbed env (operator
`CONDUCTOR_*`, `CLAUDE_*`, `GSTACK_*`, `MCP_*`, `GBRAIN_*`, and credentials like
`GH_TOKEN` never reach children), a fresh seeded `CLAUDE_CONFIG_DIR` (no operator
`~/.claude` CLAUDE.md / MCP servers / skills), a temp `GSTACK_HOME`, and
`--strict-mcp-config`. Local eval signal matches CI. Debug against real operator
state with `EVALS_HERMETIC=0` (restores the legacy env AND drops the strict-MCP
flag). Per-test `env:` overrides merge last, so deliberate contamination
(`CONDUCTOR_WORKSPACE_PATH`, per-test `GSTACK_HOME`) keeps working. Wiring is
pinned by `test/hermetic-wiring.test.ts` (static tripwire) and two gate-tier
canaries in `test/skill-e2e-hermetic-canary.test.ts`.

E2E tests stream progress in real-time (tool-by-tool via `--output-format
stream-json --verbose`). Results persist to `~/.gstack-dev/evals/` with
auto-comparison against the previous run.

## Diff-based test selection

`test:evals` and `test:e2e` auto-select tests based on `git diff` against the
base branch. Each test declares its file dependencies in
`test/helpers/touchfiles.ts`. Changes to global touchfiles (session-runner,
eval-store, touchfiles.ts itself) trigger all tests. Use `EVALS_ALL=1` or the
`:all` script variants to force all tests. Run `eval:select` to preview.

## Two-tier system

Tests are classified `gate` or `periodic` in `E2E_TIERS`
(`test/helpers/touchfiles.ts`). CI runs only gate tests (`EVALS_TIER=gate`);
periodic tests run weekly via cron or manually. When adding E2E tests, classify:
1. Safety guardrail or deterministic functional test? -> `gate`
2. Quality benchmark, Opus model test, or non-deterministic? -> `periodic`
3. Requires external service (Codex, Gemini)? -> `periodic`

## E2E test fixtures: extract, don't copy

NEVER copy a full SKILL.md file into an E2E test fixture — 1500-2000 lines of
context bloat causes timeouts and 5-10x slower tests. Extract only the section
the test needs:

```typescript
// BAD — agent reads 1900 lines, burns tokens on irrelevant sections
fs.copyFileSync(path.join(ROOT, 'ship', 'SKILL.md'), path.join(dir, 'ship-SKILL.md'));

// GOOD — agent reads ~60 lines, finishes in 38s instead of timing out
const full = fs.readFileSync(path.join(ROOT, 'ship', 'SKILL.md'), 'utf-8');
const start = full.indexOf('## Review Readiness Dashboard');
const end = full.indexOf('\n---\n', start);
fs.writeFileSync(path.join(dir, 'ship-SKILL.md'), full.slice(start, end > start ? end : undefined));
```

When debugging failures with targeted E2E runs: run in **foreground**
(`bun test ...`), never `pkill` running eval processes and restart — you lose
results and waste money. One clean run beats three killed-and-restarted runs.
