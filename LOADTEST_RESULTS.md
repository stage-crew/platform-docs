# SAFF Load Testing — Final Results

**As of 4 September 2026.** Every figure below was read from the durable run
store (`loadtest.run` in production Postgres), not from a cockpit view or a
narrative summary. [LOADTEST_REPORT.md](LOADTEST_REPORT.md) holds the full
history, the corrections, and the derivations; this document is the verdict.

## The headline

**The system does not sweat 2× load — it sheds it, cleanly and with zero
errors.**

Two arms, both at `SCALE=2` (10,000 joins/s, double the specified drop
burst), identical scenario and fixture, pre-registered before either ran:

| | cold arm | pre-positioned arm |
|---|---:|---:|
| run | `st2-drop-1788441064400` | `st2-drop-1788442576659` |
| floor | resting 2 replicas | `AUTOSCALE_MIN=8`, warmed 5 min |
| replicas traversed | 2 → 2 (never scaled) | 8 → 10 |
| iterations | 717,772 | 717,302 |
| admitted (`saff_admitted_total`) | 208,638 | **592,045** |
| of which join-tagged (`{endpoint:prequeue_join}`) | 187,084 | 562,051 |
| shed | 526,907 | 143,029 |
| **unexpected** | **0** | **0** |
| verdict | pass | red on one stale threshold |

Both arms: **zero unexpected failures at double demand.** Two replicas
absorbed 10,000 joins/s without a single error, shedding the excess — which
is the shedder's job. Pre-positioning is worth **2.84×** more admitted load
(208,638 → 592,045, bare totals compared like for like) and is the measured
value of that runbook line.

**Two admission metrics exist and they are not interchangeable.** The bare
`saff_admitted_total` counts every admission; the join-tagged
`saff_admitted_total{endpoint:prequeue_join}` counts only queue joins, and it
is what ST-2's threshold gates and what `admission-reading.ts` reads for the
capacity fit. On this scenario the join-tagged count also happens to equal
`saff_contended_total` exactly (187,084 and 562,051), so a figure quoted
without its metric name is ambiguous between three readings. Every number in
this document names its metric for that reason.

**The fleet reached 8 replicas by itself**, under load alone, before any
floor mechanism existed. Across six `SCALE=1` runs capacity tracks the fleet
at `admitted ≈ 42,322 × replicas + 79,732` (r² 0.73).

## Scenario board

| Scenario | State | Basis |
|---|---|---|
| ST-1 baseline | **pass** | measured |
| ST-2 drop | **red on a stale ceiling** — the origin exceeded it | measured |
| ST-3 contention | **mechanism named, rate measured** — 83 ppm | measured |
| ST-4 failure | **blocked on operator** — not red, not green | operator-blocked |
| ST-5 rehearsal | **red on one threshold** — inside run-to-run spread | measured |

**Two gates are red. Neither is the system misbehaving, and both are named
findings rather than unknowns.**

### ST-2 — red because the origin got faster

Two `SCALE=1` runs on 4 Sep at a fleet verified stable at 2 replicas:

| Run | Admitted (gated metric) | Shed | Unexpected |
|---|---:|---:|---:|
| `st2-drop-1788522096214` | 337,839 | 12,910 | 0 |
| `st2-drop-1788523289210` | 337,032 | 14,018 | 0 |

They reproduce each other to **0.24%**, and both exceed
`BURST_ADMISSION_BUDGET` (323,040) by ~4.6%, which is the single failing
threshold. Every prior 2-replica run was capacity-limited at 41.5–53.6% shed;
these shed under 4%.

The cause is measured: **G115's hold-search bound roughly doubled
resting-shape capacity**, so the budget describes the pre-fix system. The
gate can only flag capacity *exceeding* its bound, never hide a regression —
the red runs toward caution. Re-deriving the ceiling needs more offered load
than `SCALE=1` provides, which needs an operator to widen
`LOADTEST_SCALE_CEILING`. **Not a code change.**

### ST-3 — a real defect, named and bounded

Six `SCALE=1` runs produced 1, 0, 0, 2, 0, 0 unexpected — **83 ppm across the
first 36,000 iterations**, then a clean pair. The origin's own logs name it:
`deadlock detected`, **SQLSTATE 40P01** on `DELETE …/holds/…`. Not an unwired
guard — `releaseHold` is already wrapped in `withDeadlockRetry`; the retry
exhausted.

The metric that should have measured this could not distinguish absorbed from
escaped retries; it was split and tested. Re-measured: **6 absorbed, 0
exhausted**. The pre-registered fork stays open.

### ST-5 — red on a bound inside its own noise

`saff_hold_shed_rate` 54.4% against `rate<0.5`. The same read showed that
bound sits *inside* run-to-run spread on unchanged code: 38.4 / 39.4 / 40.6 /
51.4%. Everything else held — `unexpected`, `exhausted` and
`upstreamUnavailable` all **0** across 580,377 iterations, and hold integrity
(`saff_hold_seated_or_lost`) at **1.0**.

### ST-4 — blocked, and recorded as blocked

The run is a harness-clean baseline that **asserts nothing**: no fault was
injected. It is not recorded as a pass, because an uninjected green is
vacuous. Four fault rows need a third party (Cloudflare, the payment
provider, +500 ms Postgres).

## Capacity

The origin ceiling is **not folklore any more**. The 1,200 req/s figure in the
original spec traced to no constant, no pool budget and no measurement.
Derived properly — Little's Law over PgBouncer's server connections, with
mean transaction time from the pooler's own accounting under load —
it is **≈3,770 req/s**, independently corroborated by the pooler's 1.17 ms
mean connection wait.

The scaling ceiling is the **connection budget**, not the Railway plan.

## What "all green" would require

Three things, and **none is a code change**:

1. **A re-run of ST-2 at higher offered load** — needs
   `LOADTEST_SCALE_CEILING` widened by an operator. The current `SCALE=1`
   burst no longer saturates the origin, so it yields a lower bound, and a
   gate set to a lower bound can never fire.
2. **An operator running the four unexercised fault rows** against
   production.
3. **A `SalePhase.dropAt` inside ±10 min** for a meaningful ST-5 re-run at
   the 8-replica floor. All six recorded runs predate `dropWindowFloor` and
   ran at 7, 7, 4 and 2 replicas — which is why hold shed spans 38.4–54.4% on
   identical code.

The distance to an all-green board is **measurement and operator time, not
engineering**.

## Reading discipline

Two rules this ladder was built to enforce, both learned the hard way:

- **A red with a name is the deliverable; a green that cannot be explained is
  what this exists to prevent.** One run was nearly banked as a pass while
  measuring nothing.
- **Recompute a bound, never widen it.** An idle-regime pooler reading would
  have turned a red green by arithmetic alone; it was discarded as
  regime-invalid, because the ceiling derives from 535,496 transactions
  measured *under* the ladder.
