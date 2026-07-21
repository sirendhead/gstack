@AGENTS.md
# gstack development

## Commands (cost-annotated — build/install basics live in AGENTS.md)

```bash
bun test               # free, <2s — run before EVERY commit
bun run test:evals     # PAID ~$4/run max — LLM judge + E2E, diff-based; run before shipping
bun run test:gate      # gate-tier only (CI default, blocks merge)
bun run test:periodic  # periodic-tier (weekly cron / manual)
bun run eval:select    # preview which tests the current diff selects
bun run dev <cmd>      # CLI dev mode, e.g. bun run dev goto https://example.com
bun run slop:diff      # slop-scan findings in files changed on this branch
```

`test:evals` requires `ANTHROPIC_API_KEY`; Codex E2E uses `~/.codex/` auth (no
`OPENAI_API_KEY` env var). Testing internals (hermetic env, Conductor env-shim,
diff-based selection, gate/periodic tiers, fixture rules) load from
`.claude/rules/testing.md` when you touch `test/**`.

## SKILL.md workflow

- SKILL.md files are **generated** from `.tmpl` templates: edit the `.tmpl`, run
  `bun run gen:skill-docs`, commit BOTH the template and the generated `.md`.
- New browse command → `browse/src/commands.ts` and rebuild; new snapshot flag →
  `SNAPSHOT_FLAGS` in `browse/src/snapshot.ts` and rebuild.
- **NEVER resolve merge conflicts on generated SKILL.md by accepting either
  side.** Resolve on the `.tmpl` + `scripts/gen-skill-docs.ts`, run
  `bun run gen:skill-docs`, stage the regenerated files.
- Token ceiling: generated SKILL.md warns above 160KB (~40K tokens) — a
  feature-bloat tripwire, not a hard gate. Look at WHAT grew before compressing
  carefully-tuned prose (cuts to coverage audit / review army have real cost).
- Template authoring rules (prose-not-bash-state, no hardcoded branch names,
  self-contained blocks) load from `.claude/rules/skill-templates.md` when you
  touch `*.tmpl`.

## Platform-agnostic design

Skills NEVER hardcode framework-specific commands, file patterns, or directory
structures. Read the target project's CLAUDE.md for its config; if missing,
AskUserQuestion — then persist the answer to that project's CLAUDE.md so we
never ask again. The project owns its config; gstack reads it.

## Writing style (V1)

Default output from every tier-≥2 skill follows the Writing Style section in
`scripts/resolvers/preamble.ts` (jargon glossed via `scripts/jargon-list.json`,
questions framed in outcome terms, decisions close with user impact). Power
users: `gstack-config set explain_level terse`. Rationale:
`docs/designs/PLAN_TUNING_V1.md`.

## Browser + server subsystem

- Browser interaction (QA, dogfooding, cookies): use the `/browse` skill or
  `$B <command>`. NEVER use `mcp__claude-in-chrome__*` tools.
- Before editing `browse/src/server.ts`, sidebar files, security classifiers,
  SSE/CDP wiring, or the tunnel surface: read `browse/CLAUDE.md` (loads when you
  touch `browse/`) — the module boundaries there are load-bearing and several
  are CI-tripwired.

## Compiled binaries — NEVER commit browse/dist/ or design/dist/

Tracked-by-historical-mistake ~58MB Mach-O arm64 binaries; they show as modified
in `git status` — ignore them. Stage specific filenames only
(`git add file1 file2`); never `git add .` or `git add -A`. (Eventual real fix:
`git rm --cached`.)

## Redaction guard (PII / secrets / legal)

Engine: `lib/redact-patterns.ts` (3 tiers: HIGH blocks, MEDIUM confirms via
AskUserQuestion, LOW FYI) + `lib/redact-engine.ts`. CLI: `bin/gstack-redact`
(exit 0 clean / 2 MEDIUM / 3 HIGH); `bin/gstack-redact-prepush` is the opt-in
hook. **Scan-at-sink:** scan the EXACT bytes being sent — temp file → scan →
pass the SAME file to `gh`/`git`; never scan a string then re-render. Public
repos get sterner per-finding confirmation; MEDIUM never auto-promotes to HIGH;
there is intentionally NO key to disable HIGH blocking. It is a guardrail, not
airtight enforcement — don't claim it stops a determined leaker.

## Commit style

**Always bisect commits** — one logical change each: rename/move separate from
behavior, test infra separate from tests, template changes separate from
regeneration, mechanical refactors separate from features. "Bisect and push" =
split staged/unstaged into logical commits and push.

## Slop-scan: AI code quality, not AI code hiding

`bun run slop` / `slop:diff` (config: `slop-scan.config.json`). Fix genuine
quality issues: empty catches around file ops → `safeUnlink()`, around process
kills → `safeKill()` (helpers in `browse/src/error-handling.ts`), redundant
`return await`, untyped exception catches. Do NOT game the linter:
string-matching error messages is brittle, exemption comments are noise,
extension catch-and-log is correct (extensions crash on uncaught), best-effort
cleanup should stay `safeUnlinkQuiet()`. Don't chase the score — accept findings
where the "sloppy" pattern is the correct engineering choice.

## Community PR guardrails

**Always AskUserQuestion** before accepting any commit that (1) touches ETHOS.md
(Garry's personal builder philosophy — no edits from external contributors or AI
agents, period), (2) removes or softens promotional material (YC references,
founder perspective, product voice are intentional), or (3) rewrites Garry's
voice to be more "neutral"/"professional". No exceptions, no auto-merging.
Fork PRs from non-collaborators don't receive CI secrets — see CONTRIBUTING.md
"Fork PRs and secrets" for the re-target workflow before debugging eval CI.

## CHANGELOG + VERSION invariants

Branch-scoped: every shipping branch bumps VERSION and adds its OWN topmost
CHANGELOG entry describing the diff vs main — never branch-internal history
(consolidate mid-branch bumps into one entry; gaps in version numbers are fine).
Scale-aware bumps: big diff = MINOR, not PATCH; breaking public surface = MAJOR.
CHANGELOG is for USERS — lead with what they can now do. Full style guide
(release-summary format, voice rules, merge-main checklist):
`docs/CHANGELOG-STYLE.md` — `/ship` consumes it at CHANGELOG time.

## Dev symlink awareness

`.claude/skills/gstack` may be a symlink back to this working directory
(gitignored) → template changes + `gen:skill-docs` are LIVE immediately for all
concurrent sessions. Check `ls -la .claude/skills/gstack` once per session;
during big refactors remove the symlink so the global install is used. Setup
creates real top-level dirs with a SKILL.md symlink inside (prefix via
`skill_prefix` in `~/.gstack/config.yaml`). Windows: `setup` copies instead of
symlinking (re-run `./setup` after every `git pull`); every link site in `setup`
must go through `_link_or_copy` (CI-enforced by
`test/setup-windows-fallback.test.ts`). Vendoring gstack into a project repo is
deprecated — global install + `./setup --team`. Changes that break on-disk state
need a migration in `gstack-upgrade/migrations/` (see CONTRIBUTING.md).

## Deploying to the active skill

`cd ~/.claude/skills/gstack && git fetch origin && git reset --hard origin/main
&& bun run build` (or copy the built binaries directly). gbrain users: the reset
reverts the rendered GBRAIN blocks — re-run `gstack-config gbrain-refresh` after
deploying (idempotent).

## E2E discipline (for agents)

- **Blame protocol:** never claim an eval failure is "pre-existing" without
  running the same eval on main and showing it fails there too. Passes on main
  but fails on branch = it IS your change. No receipts → say "unverified" and
  flag it as a risk in the PR body.
- **Don't give up on long runs:** poll to completion (`sleep 180` + TaskOutput
  loop; the full suite is 30-45 min = 10-15 cycles, do all of them, report
  progress each check). Never "I'll be notified when it completes" and stop.
- **Agent-launched evals detach:** run through `bin/gstack-detach` via the
  `eval:bg*` scripts — fresh session (survives turn-boundary SIGTERM) +
  `caffeinate` + machine-wide `gstack-evals` lock (concurrent worktrees
  serialize) + run-scoped log under `~/.gstack-dev/eval-runs/`. Poll the printed
  logfile for the `### gstack-detach EXIT=<code> ###` sentinel (success AND
  failure are marked). Export `ANTHROPIC_API_KEY` first — never pass keys in
  argv. Humans running evals foreground in their own terminal don't need this.

## Skill routing

When the user's request matches an available skill, invoke it via the Skill
tool. When in doubt, invoke the skill.

- Product ideas/brainstorming → /office-hours · Strategy/scope → /plan-ceo-review
- Architecture → /plan-eng-review · Design review → /design-consultation or /plan-design-review
- Full review pipeline → /autoplan · Bugs/errors → /investigate
- QA/testing site behavior → /qa or /qa-only · Code review → /review
- Visual polish → /design-review · Ship/deploy/PR → /ship or /land-and-deploy
- Save progress → /context-save · Resume → /context-restore

## Cross-session decision memory

Durable decisions live in an append-only store at
`~/.gstack/projects/<slug>/decisions.jsonl` (works with gbrain OFF; semantic
recall is an optional layer). **Resurface** before re-deciding:
`bin/gstack-decision-search` (`--recent N`, `--scope`, `--query`, `--semantic`).
A listed decision is settled — reversing it must be said explicitly. **Capture**
durable calls with `bin/gstack-decision-log` (`--supersede <id>` to reverse,
`--redact <id>` for accidents). Durable = architecture choice, scope cut,
tool/vendor choice, reversal — NOT turn-level edits; curate or it becomes noise.

## Local plans

`~/.gstack-dev/plans/` holds local-only (not checked in) vision/design docs.
When reviewing TODOS.md, check `plans/` for promotion candidates.

## Pointers

- `ARCHITECTURE.md` — server/tunnel/sanitization module boundaries
- `ETHOS.md` — builder philosophy: Boil the Ocean (completeness is cheap — don't
  recommend shortcuts), Search Before Building ("{runtime} {thing} built-in"
  before designing infra), effort estimates show human-team vs CC+gstack time
- `CONTRIBUTING.md` — fork-PR secrets workflow, upgrade migrations
- `docs/CHANGELOG-STYLE.md` · `docs/howto-clawhub-publishing.md`
- `browse/CLAUDE.md` · `.claude/rules/testing.md` · `.claude/rules/skill-templates.md`

## GBrain Search Guidance (configured by /sync-gbrain)
<!-- gstack-gbrain-search-guidance:start -->

GBrain is set up and synced on this machine. The agent should prefer gbrain
over Grep when the question is semantic or when you don't know the exact
identifier yet.

**This worktree is pinned to a worktree-scoped code source** via the
`.gbrain-source` file in the repo root (kubectl-style context). Any
`gbrain code-def`, `code-refs`, `code-callers`, `code-callees`, or `query`
call from anywhere under this worktree routes to that source by default —
no `--source` flag needed. Conductor sibling worktrees of the same repo
each have their own pin and their own indexed pages, so semantic results
match the actual code on disk in this worktree.

Two indexed corpora available via the `gbrain` CLI:
- This worktree's code (auto-pinned via `.gbrain-source`).
- `~/.gstack/` curated memory (registered as `gstack-brain-<user>` source via
  the existing federation pipeline).

Prefer gbrain when:
- "Where is X handled?" / semantic intent, no exact string yet:
    `gbrain search "<terms>"` or `gbrain query "<question>"`
- "Where is symbol Y defined?" / symbol-based code questions:
    `gbrain code-def <symbol>` or `gbrain code-refs <symbol>`
- "What calls Y?" / "What does Y depend on?":
    `gbrain code-callers <symbol>` / `gbrain code-callees <symbol>`
- "What did we decide last time?" / past plans, retros, learnings:
    `gbrain search "<terms>" --source gstack-brain-<user>`

Grep is still right for known exact strings, regex, multiline patterns, and
file globs. Run `/sync-gbrain` after meaningful code changes; for ongoing
auto-sync across all worktrees, run `gbrain autopilot --install` once per
machine — gbrain's daemon handles incremental refresh on a schedule.

Safety: don't run `/sync-gbrain` while `gbrain autopilot` is active — the
orchestrator refuses destructive source ops when it detects a running autopilot
to avoid racing it (#1734). Prefer registering user repos with `gbrain sources
add --path <dir>` (no `--url`): URL-managed sources can auto-reclone, and the
sync code walk for them requires an explicit `--allow-reclone` opt-in.

<!-- gstack-gbrain-search-guidance:end -->
