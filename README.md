# TaskFlow – Week 5: Testing, Debugging & Optimization

A small full-stack task manager used as the *system under test* for this assignment.

| Layer | Tech | Location |
|---|---|---|
| Back-end | Node.js + Express REST API, in-memory store | `backend/src/` |
| Front-end | Vanilla JS modules (`ui.js`, `api.js`, `app.js`) + `index.html` | `frontend/` |
| Tests | **Jest 29** (two projects: `backend` = node env, `frontend` = jsdom env) + **Supertest** | `tests/` |
| Perf | V8 `--prof` profiler + custom micro-benchmark | `benchmarks/` |

## 1. Quick start

```bash
npm install            # Node 18+ (developed on Node 22)
npm test               # all 86 tests, both projects
npm run coverage       # + coverage report (text + HTML in logs/coverage/)
npm run test:backend   # only back-end tests
npm run test:frontend  # only front-end tests
npm run test:unit      # unit tests only
npm run test:integration
npm run bench          # baseline vs optimised benchmark (bench:save also rewrites logs/05_*)
npm start              # run the app at http://localhost:3000
```

## 2. Project layout

```
backend/src/   app.js (routes) · taskService.js (logic, optimised) · validators.js · cache.js (LRU) · server.js
frontend/      public/index.html · src/ui.js (render/escape/debounce) · src/api.js (fetch client) · src/app.js (controller)
tests/
  backend/unit/         validators · taskService · cache · taskService.optimisation (incl. differential test)
  backend/integration/  api.test.js  (HTTP routes + validation + service through Supertest)
  frontend/unit/        ui.test.js · api.test.js (fetch mocked)
  frontend/integration/ app.test.js  (jsdom DOM + real API client + REAL Express app, no mocks)
  setup/                jsdom-polyfills.js
benchmarks/    bench.js · profile_baseline.js · baseline/taskServiceBaseline.js (bug-fixed, unoptimised reference)
logs/          error logs, profiler output, benchmark results, diffs, coverage
docs/          DEBUGGING_REPORT.md (process + lessons learned)
```

## 3. Testing strategy

**Test pyramid.** Many fast unit tests, fewer integration tests, no browser-level E2E (out of scope for 30–35 h).

| Level | What it proves | Files | Tests |
|---|---|---|---|
| Unit – back-end | validation rules, CRUD/list/stats logic, LRU cache | `validators`, `taskService`, `cache` | 33 |
| Unit – optimisation | cache invalidation, incremental stats, **randomised differential test** (optimised ≡ baseline over 4 seeds × 400 ops) | `taskService.optimisation` | 12 |
| Unit – front-end | HTML escaping (XSS), date formatting, debounce (fake timers), DOM rendering, fetch client (mocked) | `ui`, `api` | 19 |
| Integration – back-end | every route, status code, error path, query-string handling, route ordering, full lifecycle | `api.test.js` | 14 |
| Integration – front-end | user actions in a jsdom DOM → real `fetch` bridge → real Express app → DOM update | `app.test.js` | 8 |
| **Total** | | | **86** |

**Techniques used:** boundary-value analysis (100/101-char titles), equivalence partitioning (valid/invalid priorities), negative testing
(malformed JSON, unknown ids, network failure), security testing (XSS payloads, information-leak check on 500s, limit cap),
state-based testing (create→complete→stats→delete), regression tests (one per bug found), fake timers for time-dependent code,
mocks/spies only at true boundaries (`fetch`, `console.error`), and differential testing to guard the optimisation.

**Coverage (final):** 98.7 % statements · 92.6 % branches · 100 % functions · 98.9 % lines. Back-end `taskService.js` and `validators.js` are 100 % on every metric.

## 4. Debugging summary (details in `docs/DEBUGGING_REPORT.md`)

The first test run failed **16 tests + 1 suite that could not start** (`logs/01_initial_run_FAILURES.log`). Root-cause analysis found
**8 distinct defects** (plus 1 hardening item found in review). After fixing, all tests passed
(`logs/02_after_debugging_PASS.log`); the exact code changes are in `logs/06_bugfix_*.diff`.

| # | Defect | Where | Effect |
|---|---|---|---|
| 1 | Title length checked before `trim()` | `validators.js` | whitespace-only titles accepted |
| 2 | `changes.completed \|\| task.completed` | `taskService.update` | tasks could never be un-completed |
| 3 | `start = page * limit` (off-by-one) | `taskService.list` | first page skipped; page 1 was empty |
| 4 | `parseInt` result used unchecked | `app.js` | `?page=abc` → NaN → empty list; huge `limit` allowed |
| 5 | `&` escaped last, quotes not escaped | `ui.escapeHtml` | double-escaped text; attribute-injection risk |
| 6 | `debounce` was actually a leading-edge throttle | `ui.debounce` | search used stale keystrokes |
| 7 | jsdom lacks `TextEncoder` | test config | front-end integration suite could not start |
| 8 | Fixed number of event-loop ticks used to wait for HTTP | `app.test.js` | flaky test (passed/failed randomly) |
| 9 | (hardening) 500 handler echoed `err.message` | `app.js` | possible information leak |

Techniques: reading Jest diffs & stack traces, minimal reproduction per bug, hypothesis → fix → re-run loop, `git diff`-style change review,
running the suite repeatedly to expose flakiness, V8 CPU profiling for performance bugs.

## 5. Optimisation summary

**Method:** measure → profile → hypothesise → change → re-measure → prove behaviour unchanged.
`node --prof` on the baseline (`logs/03_profile_baseline_top.txt`) showed the time in the sort comparator inside `list()` and in
GC/`memcpy` from cloning every task on every request (~74 % of ticks inside V8 internals, not application code).

| Optimisation | Complexity change |
|---|---|
| `Map<id, task>` instead of array scans | `getById/update/remove` O(n) → O(1) |
| Priority buckets instead of `sort()` on each request | O(n log n) → O(n), and only when data changed |
| Pre-computed lowercase titles | removes n × `toLowerCase()` per search |
| Copy only the requested page (≤ 100 rows) | O(n) allocations → O(limit) |
| Incremental counters | `stats()` O(n) → O(1) |
| LRU cache of `list()` results, invalidated by any write | repeated queries O(1) |
| Server-side `limit` cap (100) | bounds worst-case response size |

**Results** (`logs/05_benchmark_results.txt`, Node 22, 50 000 tasks; *cold* = cache deliberately missed so caching is not flattering the numbers):

| Scenario | Baseline | Optimised | Speed-up |
|---|---|---|---|
| list page 1, cold cache | 17.7 ms | 1.6 ms | 11× |
| search + paginate, cold cache | 14.7 ms | 5.6 ms | 2.6× |
| create then list (write path) | 18.9 ms | 6.1 ms | 3.1× |
| list, warm cache (polling) | 19.3 ms | ≈0.003 ms | >5 000× |
| stats() | 1.7 ms | <0.001 ms | >5 000× |
| getById | 1.9 ms | <0.001 ms | >5 000× |

Trade-offs: the cache and indexes cost extra memory and add invalidation logic; correctness is protected by the invalidation tests and the
differential test. Search is still a linear scan (a substring index would be the next step at >100 k tasks).
Absolute timings vary by machine; the *ratios* are what matter. Run `npm run bench` to reproduce.

## 6. Error logs & other artefacts

| File | Content |
|---|---|
| `logs/01_initial_run_FAILURES.log` | raw Jest output of the first run (16 failed tests, 1 suite error) |
| `logs/02_after_debugging_PASS.log` | run after bug fixes (65/65) |
| `logs/03_profile_baseline_top.txt` | V8 profiler summary of the baseline |
| `logs/04_final_run_with_coverage.log` | final run (86/86) with coverage table |
| `logs/05_benchmark_results.{txt,json}` | benchmark results |
| `logs/06_bugfix_{backend,frontend}.diff` | exact source changes made while debugging |
| `docs/DEBUGGING_REPORT.md` | debugging process and lessons learned (optional deliverable) |

## 7. Known limitations
* Data is in-memory (resets on restart); a real DB would add its own integration tests.
* No browser-level E2E (Playwright/Cypress) – the jsdom integration suite covers the DOM/HTTP chain instead.
* Benchmarks are micro-benchmarks of the service layer, not full HTTP load tests.
