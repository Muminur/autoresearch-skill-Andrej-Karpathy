# Benchmark Harness Protocol

## When to Activate

Invoked by SKILL.md Setup Phase whenever `/autoresearch <goal>` is called with a goal that requires empirical measurement. This protocol is MANDATORY for the Default Path — the loop must not begin until every Phase A–E gate has passed.

## The Single-File Rule

ONE editable harness at the repo root:

- Python → `benchmark.py`
- Node → `benchmark.mjs`
- Go → `benchmark.go`
- Shell → `benchmark.sh`

The harness loads the corpus, runs each case through the **real** code path, and prints a single line `metric: <float>` on stdout. During iteration the harness is **read-only**; iteration modifies SOURCE files only. Any harness edit requires a separate commit prefixed `harness:`.

---

## Phase A — Corpus Ingestion (Real Data Only)

1. Identify the corpus source from the goal (logs, exports, fixtures, API, scraped pages).
2. If a corpus directory already exists under `autoresearch/data/`, use it.
3. If not, SCRAPE it (Playwright / curl / `tail` on prod logs / database export) into `autoresearch/data/<name>.jsonl` — one real case per line.
4. **NEVER synthesise cases.** If scraping is blocked (auth, rate limit, access), STOP and ask the user for a data source. Do not fabricate examples to unblock yourself.
5. Print `corpus: N cases from <source>`. If `N == 0`, refuse to proceed.

Corpus is waived ONLY when the metric is purely structural and deterministic — `LOC`, `build time`, `bundle size`, `cold-start time`. For those, `corpus_required=false` and Phase A is skipped.

---

## Phase B — Harness Construction

The harness MUST:

- load the corpus from `autoresearch/data/<name>.jsonl`
- run each case through the **REAL** code path — no mocks of the target module under optimisation
- print exactly `metric: <float>` on stdout (optionally plus `avg_latency_ms: <float>`, `reliability: <float>`)
- exit 0 on success; exit non-zero if any case crashes
- use env vars (`TESTNET_URL`, `STAGING_URL`) — never hit production
- time cases with `time.perf_counter()` / `performance.now()` / `time.Now()`
- run warm-up calls before the measurement loop where connection pools / caches matter

The harness is the contract. If the harness crashes during baseline, FIX THE HARNESS (not source code). Commit separately: `harness: initial benchmark.py for <metric>`.

---

## Phase C — Baseline Capture (GATE)

1. Run the harness fresh: `<verify_cmd>`.
2. Record as iteration #0 in `autoresearch/results.tsv` with status `baseline`.
3. TSV columns: `commit\tmetric\tavg_latency_ms\tstatus\tdescription`.
4. If the harness crashes, loop back to Phase B — fix the harness, not the source.
5. Commit separately: `harness: baseline <metric>=<value>`.

**Gate:** no baseline row in `results.tsv` → loop cannot start.

---

## Phase D — Regression Gate (GATE)

Detect the existing test suite (first match wins):

| Check | Suite |
|-------|-------|
| `pyproject.toml` or `pytest.ini` exists | `pytest -q` |
| `package.json` has `"test"` script | `npm test` |
| `Cargo.toml` exists | `cargo test` |
| `go.mod` exists | `go test ./...` |

1. Run the detected suite once **before any change**.
2. Store the pre-change pass count as `N_pre` in `autoresearch/.regression-baseline` (plain text, single integer).
3. Every iteration re-runs the suite **after** modification.
4. If `passed < N_pre` → auto-discard the iteration. Use `git reset --hard HEAD~1` (the change was committed before verify per the Loop protocol). Log status `discard-regression`.
5. **No exceptions.** A tempting perf win that drops pre-existing tests is not a win.

**Gate:** no `.regression-baseline` file → loop cannot start.

---

## Phase E — Hot-Path Reading

Before the Ideate step of iteration #1:

1. Trace the code path the metric flows through: entry point → handler → I/O layer.
2. Read each file on that path in full (no skimming).
3. List 3–5 candidate bottlenecks as `status=idea` rows in `results.tsv` — one per line, no code yet.
4. Apply **at most one** idea per iteration. Never bundle.

Candidates should cite file:line anchors so future iterations can cross-reference.

---

## Enforcement Refusals

Refuse to enter the loop and print the refusal reason if ANY of:

- corpus is synthetic or `count == 0` (and `corpus_required=true`)
- baseline row missing in `results.tsv`
- `regression_cmd` not set, or the suite has not been run
- harness prints something other than `metric: <float>` on stdout

Print the refusal reason, then halt. Do not paper over missing gates.

---

## Harness Templates

### Python (`benchmark.py`)

```python
import asyncio, json, time, sys
from pathlib import Path
from src.your_module import your_entry_point  # REAL code path

CORPUS = Path("autoresearch/data/cases.jsonl")

async def run_case(case):
    t0 = time.perf_counter()
    ok = await your_entry_point(case)
    return ok, (time.perf_counter() - t0) * 1000

async def main():
    cases = [json.loads(l) for l in CORPUS.read_text().splitlines() if l.strip()]
    results = await asyncio.gather(*(run_case(c) for c in cases))
    ok_count = sum(1 for ok, _ in results if ok)
    avg_ms = sum(ms for _, ms in results) / len(results)
    print(f"reliability: {ok_count / len(results):.4f}")
    print(f"avg_latency_ms: {avg_ms:.1f}")
    print(f"metric: {avg_ms:.1f}")
    sys.exit(0 if ok_count == len(results) else 1)

if __name__ == "__main__":
    asyncio.run(main())
```

### Node (`benchmark.mjs`)

```js
import fs from "node:fs";
import { performance } from "node:perf_hooks";
import { yourEntryPoint } from "./src/your_module.js";

const cases = fs.readFileSync("autoresearch/data/cases.jsonl", "utf8")
  .split("\n").filter(Boolean).map(JSON.parse);

let ok = 0;
const latencies = [];
for (const c of cases) {
  const t0 = performance.now();
  try { await yourEntryPoint(c); ok++; } catch {}
  latencies.push(performance.now() - t0);
}
const avg = latencies.reduce((a,b) => a+b, 0) / latencies.length;
console.log(`reliability: ${(ok / cases.length).toFixed(4)}`);
console.log(`avg_latency_ms: ${avg.toFixed(1)}`);
console.log(`metric: ${avg.toFixed(1)}`);
process.exit(ok === cases.length ? 0 : 1);
```

### Go (`benchmark.go`)

```go
package main

import (
    "bufio"
    "encoding/json"
    "fmt"
    "os"
    "time"
    "your/module/pkg"
)

func main() {
    f, _ := os.Open("autoresearch/data/cases.jsonl")
    defer f.Close()
    s := bufio.NewScanner(f)
    var total time.Duration
    ok, n := 0, 0
    for s.Scan() {
        var c map[string]any
        json.Unmarshal(s.Bytes(), &c)
        t0 := time.Now()
        if err := pkg.YourEntryPoint(c); err == nil { ok++ }
        total += time.Since(t0)
        n++
    }
    avg := float64(total.Milliseconds()) / float64(n)
    fmt.Printf("reliability: %.4f\n", float64(ok)/float64(n))
    fmt.Printf("avg_latency_ms: %.1f\n", avg)
    fmt.Printf("metric: %.1f\n", avg)
    if ok != n { os.Exit(1) }
}
```

### Bash (`benchmark.sh`)

```bash
#!/usr/bin/env bash
set -eu
CASES=$(wc -l < autoresearch/data/cases.jsonl)
ok=0; total_ms=0
while IFS= read -r line; do
  t0=$(date +%s%N)
  if ./your_command "$line" >/dev/null 2>&1; then ok=$((ok+1)); fi
  t1=$(date +%s%N)
  total_ms=$(( total_ms + (t1 - t0) / 1000000 ))
done < autoresearch/data/cases.jsonl
avg=$(awk -v t=$total_ms -v n=$CASES 'BEGIN{printf "%.1f", t/n}')
echo "reliability: $(awk -v o=$ok -v n=$CASES 'BEGIN{printf "%.4f", o/n}')"
echo "avg_latency_ms: $avg"
echo "metric: $avg"
[ "$ok" -eq "$CASES" ] || exit 1
```

Each template is a minimal skeleton — adapt field names, entry points, and metric choice to the project. Keep the harness **one file**; resist the urge to split into modules.
