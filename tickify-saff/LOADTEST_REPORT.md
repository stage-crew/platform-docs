# SAFF Load Testing Report — 28 Aug 2026

Full ST-1…ST-5 scenario ladder executed against the production SAFF
deployment, with durable per-commit run history. This document is the
standalone report of the current ladder; [ROLLOUT.md](ROLLOUT.md) §4–§5 holds
the narrative history of earlier runs.

## 1. Scope and system under test

- **System:** the production SAFF deployment on Railway at commit `43cffd0f`,
  region asia-southeast1: `saff-api` behind the elastic fleet (2 replicas at
  rest, autoscaled 2–6 on live CPU/memory, up to `WEB_CONCURRENCY_MAX=4`
  processes per replica derived per-container), PgBouncer transaction pooling
  (3 replicas × `default_pool_size` 70, `max_client_conn` 1000),
  Patroni-managed Postgres.
- **Harness:** `apps/saff-loadtest` (k6 v2.2.0), driven over its
  authenticated `POST /api/run` API from inside the Railway private network.
  One run at a time by design. All runs `SCALE=0.1`, 120 s.
- The commit under test carries the 28 Aug performance pass (`01a90599`,
  `dd319279`): parallelised match-detail and seatmap-snapshot reads,
  seat-label read moved outside `acquireHold`'s transaction, independent
  order-transaction reads paired, batched scan-sync inserts, boot-frozen
  OpenAPI document, projected match-list columns, narrowed availability-poll
  lookup, hoisted payment methods, content-derived `ETag`/304 on the
  availability poll, in-process cache for immutable geometry artefacts.

## 2. Methodology and evidence integrity

Each scenario measures a distinct production concern:

| Scenario | Exercises |
| --- | --- |
| ST-1 baseline | steady browse + purchase at rest |
| ST-2 drop | prequeue burst at drop time |
| ST-3 contention | hot-zone hold race on a deliberately undersized section (SPEC §14.5) |
| ST-4 failure | failure injection; holds must release cleanly |
| ST-5 rehearsal | full dress rehearsal, browse through purchase |

Evidence controls, both in force for every run in this report:

- **Durable history.** Every finished run is upserted into a
  `loadtest.run` table in Postgres: id, scenario, scale, duration,
  timestamps, exit code, verdict, git SHA, warm-up status, full parsed k6
  summary and observability samples as `jsonb`. Results are diffable across
  commits and survive service sleeps, redeploys and rebuilds.
- **Warm-up gating.** The runner polls the API for 10 consecutive 200s
  before spawning k6 and records `warmup: passed | timeout` on the run row,
  so a cold start cannot masquerade as a serving-path regression. Every run
  below: warm-up `passed`.

## 3. Results

| Run | Verdict | Iterations | Unexpected | Holds (success rate) | http med / p95 |
| --- | --- | --- | --- | --- | --- |
| `st1-baseline-1787871501677` | **pass** | 709 | 0 | 119 (1.0) | 60 ms / 114 ms |
| `st2-drop-1787871750378` | **pass** | 35,888 | 0 | — (prequeue burst) | 12 ms / 28 ms |
| ~~`st3-contention-1787871945386`~~ | **void — invalid fixture, § 4.1** | 900 | 1 | 852 (0.947) | 515 ms / 5.7 s |
| `st3-contention-1787900126524` | **fail — finding F-LT-1, § 4.2** | 900 | **231** | 587 (0.652), 87 contended, 0 exhausted | 1.2 s / 18.2 s |
| `st4-failure-1787872096682` | **pass** | 597 | 0 | 357 (1.0) | 62 ms / 160 ms |
| `st5-rehearsal-1787872231655` | **pass** | 58,029 | 0 | 717 (1.0), 55,649 contended by design | 13 ms / 60 ms |

All runs at `gitSha 43cffd0f` except the ST-3 re-run (`fa18a883`, the
fixture correction described in § 4.1).

Four of five scenarios pass. ST-3 fails with a real, newly visible finding,
documented below with its risk assessment and disposition.

## 4. Findings

### 4.1 Voided run: ST-3's first execution measured the wrong fixture

The first ST-3 run is excluded from results, and the exclusion is itself
evidence-relevant, so it is recorded rather than deleted.

ST-3 is the only scenario with a dedicated fixture: a seed step builds match
`st3-contention` with hot zone `st3-hot`, sized so that concurrent buyers
genuinely race for the same seats. The deployed service configuration pinned
the harness's shared `MATCH_SLUG` to the demo match, overriding the
scenario's fixture — so the first run contended an ordinary, never-seeded
demo section. Its numbers (0.947 hold success, 5.7 s p95) measured neither
regression nor health and cannot be used as evidence in either direction.

**Correction:** the harness now always targets ST-3's own seeded match and
zone, and a regression test pins that behaviour. The ST-3 re-run in § 3
executes against the corrected fixture.

### 4.2 Finding F-LT-1: the hold path has no load-shed admission control

**Severity: high (availability), none (correctness). Status: open,
registered as G87 in [GAPS.md](GAPS.md); class decision pending.**

Against the correct fixture, 300 virtual users produced 231 unexpected
errors in 900 iterations, hold success 0.652, p95 18.2 s. Server logs over
the run window reduce to exactly two messages: 228 × pool
`connection timeout` and 2 × `deadlock detected`.

**Root cause.** The API sheds load per route class under pressure: orders,
payments, webhooks and scan-sync at class 0, the prequeue join at class 2,
catalog reads at class 3 (the class map is pinned by a route-coverage test).
The hold-acquisition routes — `POST /holds/`,
`POST /holds/best-available/` — carry a per-IP rate limit but **no shed
mount**. Under hot-zone saturation the hottest write on the buyer path
admits every request regardless of pool pressure, and the connection pool
answers with its 5 s timeout instead. The timeout bound itself is working as
designed; before it existed, the same saturation surfaced as an indefinite
hang rather than a fast error.

**Correctness invariants held.** SPEC §14.5's contention criteria are
"exactly `sellable` sold, no duplicate seat, conflict rate within budget":

- **0 oversell** — no seat sold twice, exact sellable count sold (verified
  by the dedicated in-process oversell harness, which owns this half of
  §14.5).
- **87 clean contended refusals, 0 exhausted** — losers were refused
  correctly, not double-booked.

The failure is an availability/UX defect — buyers at peak receive slow
errors instead of fast shed responses — not a data-integrity defect.

**Why it is not fixed in this change.** Assigning the hold path a shed class
determines what a buyer sees at peak and interacts with open capacity
decisions (G86). That is a deliberate design decision, registered as G87
with the evidence above, not an oversight.

## 5. Evidence access

Run history and full k6 summaries are retrievable from the harness service,
basic-auth gated, on the Railway private network
(`saff-loadtest.railway.internal:8080`):

```
GET /api/history                  # last 50 runs: id, verdict, exitCode, gitSha, warmup
GET /api/history?scenario=st1-baseline
GET /api/history/run?id=<run id>  # full row incl. k6 summary + observability jsonb
GET /api/report?id=<run id>       # k6 web-dashboard HTML for one run
```

The same data is browsable interactively from the operator admin app
(`saff-admin.tickify.live`, Basic-Auth) at **`/loadtest`**: start/stop runs,
watch a run in flight, list run history, and open each run's HTML report at
`/loadtest/report?id=…` — the admin server proxies to the private-network
harness, so no direct network access is required.

The history table survives sleeps, redeploys and rebuilds; it is dropped
only if the `loadtest` schema is dropped.

## Appendix: internal history

Context for repository maintainers; not required for audit review.

- **Prior-ladder comparison (18 Aug):** ST-1 p95 105 → 114 ms (within
  noise; median improved to 60 ms). ST-2 stayed clean across 35,888
  iterations (G57 remains closed). ST-5, previously red on
  `saff_unexpected_total`, now passes with 0 unexpected across 58,029
  iterations — the last browse/prequeue-shaped failure is gone; ST-3's G87
  is a different and newly visible finding. ST-4 unchanged.
- **Harness durability changes** (motivation and mechanics in
  [ROLLOUT.md](ROLLOUT.md) §5.1): earlier runs' evidence lived in container
  memory and `/tmp` and was lost on service sleep — the two 27 Aug ST-1 run
  artefacts were lost exactly this way. The Postgres history table and
  runner-level warm-up gating in § 2 are the structural fixes.
- **ST-3 fixture defect details:** the shared `MATCH_SLUG` default and the
  service-level override that produced § 4.1, the scenario docblock that
  predicted it, and the precedence test that pins the fix are in
  `apps/saff-loadtest` (`scenarios/config.ts`, `st3-contention.ts`,
  `server/runner.test.ts`); fix landed in `fa18a883`.
- The G-numbered entries referenced above (G32 pool timeout bound, G57
  shuffle secret, G86 capacity options, G87 hold-path shed class) are in
  [GAPS.md](GAPS.md).
