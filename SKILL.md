---
name: autoresearch
description: Autonomous Goal-directed Iteration. Apply Karpathy's autoresearch principles to ANY task. Loops autonomously — modify, verify, keep/discard, repeat. Supports optional loop count via Claude Code's /loop command. Invoking /autoresearch <free-form goal> builds a real-data benchmark harness, captures a baseline, and iterates with a regression gate until the goal is hit.
version: 1.0.4
---

# Claude Autoresearch — Autonomous Goal-directed Iteration

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch). Applies constraint-driven autonomous iteration to ANY work — not just ML research.

**Core idea:** You are an autonomous agent. Modify → Verify → Keep/Discard → Repeat.

## Subcommands

| Subcommand | Purpose |
|------------|---------|
| `/autoresearch <goal>` | **Default path** — parse free-form goal, build harness, capture baseline, loop until goal met |
| `/autoresearch` | Run the autonomous loop (default) |
| `/autoresearch:plan` | Interactive wizard to build Scope, Metric, Direction & Verify from a Goal |
| `/autoresearch:security` | Autonomous security audit: STRIDE threat model + OWASP Top 10 + red-team (4 adversarial personas) |

### Default Path: /autoresearch <free-form goal>

When the user invokes `/autoresearch <goal>` with any free-form string after the command, parse the goal into seven slots, print the parsed-slot dump back for user visibility, then run the harness protocol in `references/benchmark-harness.md` before entering the loop.

**Goal-parsing rubric:**

| Slot | Extraction rule | Fallback |
|------|-----------------|----------|
| metric | First measurable noun (`latency`, `reliability`, `coverage`, `flakiness`, `bundle size`, `p95`, `accuracy`, `error-rate`, `LOC`, `build time`) | Ask user (1 sentence) |
| direction | `reduce/lower/below/under/minimise/to 0%` + cost-word → minimise; `increase/raise/above/over/maximise/to 100%` + quality-word → maximise | minimise for cost/time/size/error, maximise for coverage/score/throughput |
| target | Number + unit in goal (`500ms`, `95%`, `0%`, `<200KB`) | `"best achievable"` — unbounded loop |
| scope | Grep repo for goal's domain terms (`API`, `test`, `build`); propose globs | Whole repo minus `node_modules`, `.venv`, `dist`, `target` |
| corpus_source | If goal names inputs (signals, queries, PRs, logs) → find source; if absent → ASK, never fabricate | `corpus_required=false` only when metric is purely structural (LOC, build time, bundle size) |
| verify_cmd | Single shell command that prints `metric: <float>` on stdout — typically `python benchmark.py` or equivalent single-file rig | Constructed during harness build |
| regression_cmd | Auto-detect: first of `pytest -q`, `npm test`, `cargo test`, `go test ./...` whose config exists | Ask user |

**Worked examples:**

```
/autoresearch reduce API p95 latency to 200ms
→ metric=p95_latency_ms, direction=minimise, target=200, scope=src/api/**,
  corpus_source=prod log tail or fixtures, verify_cmd=python benchmark.py,
  regression_cmd=pytest -q

/autoresearch reduce test flakiness to 0%
→ metric=flaky_test_rate, direction=minimise, target=0, scope=tests/**,
  corpus_source=CI run history, verify_cmd=python benchmark.py (N reruns),
  regression_cmd=pytest -q

/autoresearch increase signal-parser reliability to 99%
→ metric=reliability, direction=maximise, target=0.99, scope=src/parser/**,
  corpus_source=autoresearch/data/signals.jsonl, verify_cmd=python benchmark.py,
  regression_cmd=pytest -q
```

Print the parsed slot dump to the user before any action — this is the single confirmation checkpoint before the harness protocol begins.

### /autoresearch:security — Autonomous Security Audit (v1.0.3)

Runs a comprehensive security audit using the autoresearch loop pattern. Generates a full STRIDE threat model, maps attack surfaces, then iteratively tests each vulnerability vector — logging findings with severity, OWASP category, and code evidence.

Load: `references/security-workflow.md` for full protocol.

**What it does:**

1. **Codebase Reconnaissance** — scans tech stack, dependencies, configs, API routes
2. **Asset Identification** — catalogs data stores, auth systems, external services, user inputs
3. **Trust Boundary Mapping** — browser↔server, public↔authenticated, user↔admin, CI/CD↔prod
4. **STRIDE Threat Model** — Spoofing, Tampering, Repudiation, Info Disclosure, DoS, Elevation of Privilege
5. **Attack Surface Map** — entry points, data flows, abuse paths
6. **Autonomous Loop** — iteratively tests each vector, validates with code evidence, logs findings
7. **Final Report** — severity-ranked findings with mitigations, coverage matrix, iteration log

**Key behaviors:**
- Follows red-team adversarial mindset (Security Adversary, Supply Chain, Insider Threat, Infra Attacker)
- Every finding requires **code evidence** (file:line + attack scenario) — no theoretical fluff
- Tracks OWASP Top 10 + STRIDE coverage, prints coverage summary every 5 iterations
- Composite metric: `(owasp_tested/10)*50 + (stride_tested/6)*30 + min(findings, 20)` — higher is better
- Creates `security/{YYMMDD}-{HHMM}-{audit-slug}/` folder with structured reports:
  `overview.md`, `threat-model.md`, `attack-surface-map.md`, `findings.md`, `owasp-coverage.md`, `dependency-audit.md`, `recommendations.md`, `security-audit-results.tsv`

**Flags:**

| Flag | Purpose |
|------|---------|
| `--diff` | Delta mode — only audit files changed since last audit |
| `--fix` | After audit, auto-fix confirmed Critical/High findings using autoresearch loop |
| `--fail-on {severity}` | Exit non-zero if findings meet threshold (for CI/CD gating) |

**Usage:**
```
# Unlimited — keep finding vulnerabilities until interrupted
/autoresearch:security

# Bounded — exactly 10 security sweep iterations
/loop 10 /autoresearch:security

# With focused scope
/autoresearch:security
Scope: src/api/**/*.ts, src/middleware/**/*.ts
Focus: authentication and authorization flows

# Delta mode — only audit changed files since last audit
/autoresearch:security --diff

# Auto-fix confirmed Critical/High findings after audit
/loop 15 /autoresearch:security --fix

# CI/CD gate — fail pipeline if any Critical findings
/loop 10 /autoresearch:security --fail-on critical

# Combined — delta audit + fix + gate
/loop 15 /autoresearch:security --diff --fix --fail-on critical
```

**Inspired by:**
- [Strix](https://github.com/usestrix/strix) — AI-powered security testing with proof-of-concept validation
- `/plan red-team` — adversarial review with hostile reviewer personas
- OWASP Top 10 (2021) — industry-standard vulnerability taxonomy
- STRIDE — Microsoft's threat modeling framework

### /autoresearch:plan — Goal → Configuration Wizard

Converts a plain-language goal into a validated, ready-to-execute autoresearch configuration.

Load: `references/plan-workflow.md` for full protocol.

**Quick summary:**

1. **Capture Goal** — ask what the user wants to improve (or accept inline text)
2. **Analyze Context** — scan codebase for tooling, test runners, build scripts
3. **Define Scope** — suggest file globs, validate they resolve to real files
4. **Define Metric** — suggest mechanical metrics, validate they output a number
5. **Define Direction** — higher or lower is better
6. **Define Verify** — construct the shell command, **dry-run it**, confirm it works
7. **Confirm & Launch** — present the complete config, offer to launch immediately

**Critical gates:**
- Metric MUST be mechanical (outputs a parseable number, not subjective)
- Verify command MUST pass a dry run on the current codebase before accepting
- Scope MUST resolve to ≥1 file

**Usage:**
```
/autoresearch:plan
Goal: Make the API respond faster

/autoresearch:plan Increase test coverage to 95%

/autoresearch:plan Reduce bundle size below 200KB
```

After the wizard completes, the user gets a ready-to-paste `/autoresearch` invocation — or can launch it directly.

## When to Activate

- User invokes `/autoresearch <goal-string>` (anything after the command) → parse with Default Path rubric, then build harness per `references/benchmark-harness.md`
- User types `/autoresearch` with no argument → ask for a one-sentence goal OR suggest `/autoresearch:plan`
- User invokes `/autoresearch` or `/ug:autoresearch` → run the loop
- User invokes `/autoresearch:plan` → run the planning wizard
- User invokes `/autoresearch:security` → run the security audit
- User says "help me set up autoresearch", "plan an autoresearch run" → run the planning wizard
- User says "security audit", "threat model", "OWASP", "STRIDE", "find vulnerabilities", "red-team" → run the security audit
- User says "work autonomously", "iterate until done", "keep improving", "run overnight" → run the loop
- Any task requiring repeated iteration cycles with measurable outcomes → run the loop

## Optional: Controlled Loop Count

By default, autoresearch loops **forever** until manually interrupted. However, users can optionally specify a **loop count** to limit iterations using Claude Code's built-in `/loop` command.

> **Requires:** Claude Code v1.0.32+ (the `/loop` command was introduced in this version)

### Usage

**Unlimited (default):**
```
/autoresearch
Goal: Increase test coverage to 90%
```

**Bounded (N iterations):**
```
/loop 25 /autoresearch
Goal: Increase test coverage to 90%
```

This chains `/autoresearch` with `/loop 25`, running exactly 25 iteration cycles. After 25 iterations, Claude stops and prints a final summary.

### When to Use Bounded Loops

| Scenario | Recommendation |
|----------|---------------|
| Run overnight, review in morning | Unlimited (default) |
| Quick 30-min improvement session | `/loop 10 /autoresearch` |
| Targeted fix with known scope | `/loop 5 /autoresearch` |
| Exploratory — see if approach works | `/loop 15 /autoresearch` |
| CI/CD pipeline integration | `/loop N /autoresearch` (set N based on time budget) |

### Behavior with Loop Count

When a loop count is specified:
- Claude runs exactly N iterations through the autoresearch loop
- After iteration N, Claude prints a **final summary** with baseline → current best, keeps/discards/crashes
- If the goal is achieved before N iterations, Claude prints early completion and stops
- All other rules (atomic changes, mechanical verification, auto-rollback) still apply

## Setup Phase (Do Once)

1. **Parse the goal** using the Default Path rubric → print slot dump
2. **Read all in-scope files** for full context before any modification
3. **Ingest corpus** per `references/benchmark-harness.md` Phase A — refuse if synthetic or empty
4. **Build single-file harness** per Phase B — must print `metric: <float>` on stdout
5. **Capture baseline** per Phase C — record as iteration #0 in `autoresearch/results.tsv`
6. **Establish regression gate** per Phase D — detect existing test suite, record pass count in `autoresearch/.regression-baseline`
7. **Read hot path** per Phase E — list candidate ideas (never apply more than one per iteration)
8. **Confirm and go** — show setup summary, BEGIN THE LOOP

## The Loop

Read `references/autonomous-loop-protocol.md` for full protocol details.

```
LOOP (FOREVER or N times):
  1. Review: Read current state + git history + results log
  2. Ideate: Pick next change based on goal, past results, what hasn't been tried
  3. Modify: Make ONE focused change to in-scope files
  4. Commit: Git commit the change (before verification)
  5. Verify: Run the mechanical metric (tests, build, benchmark, etc.)
  5a. Regress: Run regression_cmd. If passed < N_pre → STATUS=discard (skip Decide).
  6. Decide:
     - IMPROVED → Keep commit, log "keep", advance
     - SAME/WORSE → Git revert, log "discard"
     - CRASHED → Try to fix (max 3 attempts), else log "crash" and move on
  7. Log: Record result in results log
  8. Repeat: Go to step 1.
     - If unbounded: NEVER STOP. NEVER ASK "should I continue?"
     - If bounded (N): Stop after N iterations, print final summary
```

## Critical Rules

1. **Loop until done** — Unbounded: loop until interrupted. Bounded: loop N times then summarize.
2. **Read before write** — Always understand full context before modifying
3. **One change per iteration** — Atomic changes. If it breaks, you know exactly why
4. **Mechanical verification only** — No subjective "looks good". Use metrics
5. **Automatic rollback** — Failed changes revert instantly. No debates
6. **Simplicity wins** — Equal results + less code = KEEP. Tiny improvement + ugly complexity = DISCARD
7. **Git is memory** — Every kept change committed. Agent reads history to learn patterns
8. **When stuck, think harder** — Re-read files, re-read goal, combine near-misses, try radical changes. Don't ask for help unless truly blocked by missing access/permissions
9. **Real data only** — corpus must be scraped/exported from reality; synthetic cases forbidden
10. **Regression gate is absolute** — any drop in pre-existing passing tests triggers auto-discard
11. **Harness is read-only during iteration** — only source files change; harness edits need a separate commit prefixed `harness:`

## Principles Reference

See `references/core-principles.md` for the 7 generalizable principles from autoresearch.

## Adapting to Different Domains

| Domain | Metric | Scope | Verify Command | Corpus Source |
|--------|--------|-------|----------------|---------------|
| Backend code | Tests pass + coverage % | `src/**/*.ts` | `npm test` | test fixtures |
| Frontend UI | Lighthouse score | `src/components/**` | `npx lighthouse` | staging URLs |
| ML training | val_bpb / loss | `train.py` | `uv run train.py` | training dataset |
| Blog/content | Word count + readability | `content/*.md` | Custom script | source manuscripts |
| Performance | Benchmark time (ms) | Target files | `npm run bench` | benchmark inputs |
| Refactoring | Tests pass + LOC reduced | Target module | `npm test && wc -l` | existing test suite |
| Security | OWASP + STRIDE coverage + findings | API/auth/middleware | `/autoresearch:security` | codebase |
| Real-traffic performance | p95 latency (ms) | hot-path files | `python benchmark.py` | prod log tail / export |

Adapt the loop to your domain. The PRINCIPLES are universal; the METRICS are domain-specific.
