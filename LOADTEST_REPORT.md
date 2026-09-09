# SAFF Load Testing Report — 31 Aug / 1 Sep / 2 Sep / 3 Sep 2026

> **3 Sep edition — read this section first; everything below it predates
> the instrument work of that night and several of its numbers are
> superseded.**
>
> **The verdict the ladder was built to produce: the system does not sweat
> 2× load — it sheds, cleanly and fairly.** The strongest evidence is direct:
> a `SCALE=2` arm offering **735,074** requests — exactly 2.00× the
> `SCALE=1` 367,294 — admitted **592,045** joins (of which 562,051 were
> contended) with **zero** unexpected responses and **zero**
> upstream-unavailable (`st2-drop-1788442576659`). The excess is refused,
> never served badly.
>
> **And the fleet got there by itself.** That run was measured at **8
> replicas** (Mimir, 4 Sep), reached by the autoscaler under load alone —
> before `dropWindowFloor` existed. Capacity scales with it: across six
> `SCALE=1` runs, `admitted ≈ 42,322 × replicas + 79,732` (`r²` 0.73), and
> the 8-replica arm beat that fit by 34%, because a fixed-arrival-rate
> scenario cannot show capacity it is not offered load to use (G126).
>
> **What the older three-reproduction claim actually measured.** The "63%
> throughput increase with contention flat" cited here through 3 Sep is the
> pair `2bf5e1c8` at 04:10 and 04:17 — **the same sha seven minutes apart,
> at 2 replicas then 5**. The contention finding stands; the throughput
> difference is fleet shape, not a code change, and it was previously
> attributed to one.
>
> **"Clean" here means every number is defensible, not that every gate is
> green.** Two gates are red and each is a named finding, not an unknown:
> ST-2 on a ceiling the system *exceeded* — now measured twice at a verified
> stable fleet and traced to a stale constant (G127: G115 roughly doubled
> resting-shape capacity on 3 Sep, so the 4,038 rps bound describes the
> pre-fix system) — and ST-3 on one unexpected in its
> newest run against a fix measured to work. **ST-5's verdict is confirmed
> as of 4 Sep** — read from the durable run store rather than the cockpit —
> and it is a single-threshold red: `saff_hold_shed_rate` 54.4% against
> `rate<0.5`, with `unexpected`, `exhausted` and `upstreamUnavailable` all
> **zero** across 580,377 iterations and hold integrity
> (`saff_hold_seated_or_lost`) at **1.0**. The number long cited as its red
> was an ungated metric value; the real red is narrower and is G112's
> already-open bound. One scenario remains blocked on an operator action
> nothing in this repository can perform. A red with a name
> is the deliverable; a green that cannot be explained is what this document
> exists to prevent, and §11 records a run that was nearly banked as a pass
> while measuring nothing.
>
> **What "all green" would require, stated because it was never written
> down.** Three things, and **none of them is a code change** — which is the
> finding, not a caveat:
>
> 1. **A re-run of ST-2 and ST-5.** ST-2 has now been re-run twice (4 Sep,
>    `SCALE=1`, fleet verified stable at 2) and the result **changed what the
>    re-run is for**: the origin admitted 337,839 and 337,032 while shedding
>    under 4%, so it is *offer-limited* — the burst no longer saturates it.
>    `ORIGIN_CEILING_RPS` therefore **cannot** be re-measured this way, because
>    an offer-limited run yields a lower bound and a gate set to a lower bound
>    can never fire. G127 records the cause: G115's hold-search bound roughly
>    doubled resting-shape capacity, so the 4,038 constant describes the
>    pre-fix system. Re-deriving it needs more offered load than `SCALE=1`
>    provides, which needs the `LOADTEST_SCALE_CEILING` widening — an operator
>    action, still not a code change. **ST-5's verdict is no longer
>    unconfirmed** — the run store served it without the cockpit — but its
>    re-run is *not*
>    a formality, and its verdict stays **red** until one runs: all six
>    recorded runs predate `dropWindowFloor` (`58eeada5`) and none ran at the
>    floor's 8 replicas — they ran at 7, 7, 4 and 2, which is why hold shed
>    spans 38.4–54.4% across identical code. A meaningful re-run
>    needs a `SalePhase.dropAt` inside ±10 min; the nearest in production is
>    2.2 days past, so the run must create one. See G112. G104 — long listed
>    here as the blocking instrument flaw — was in fact **fixed 3 Sep**
>    (`9a24c1e8`); only the register was stale.
> 2. **An operator running §14.5's four remaining unexercised fault rows** against
>    production. Four need a third party: Cloudflare, the payment provider,
>    +500 ms Postgres.
> 3. **G102 resolved for the kill-all row**, whose edge-projection claim has
>    no artefact to project from because the buyer path serves
>    `private, no-cache, no-store`.
>
> So the distance to an all-green board is measurement and operator time, not
> engineering. That is a meaningfully different answer from the one this
> section gave a day ago, and it only became visible once G104's stale
> register entry was checked against the code it describes.
>
> | Scenario | State | Run id |
> | --- | --- | --- |
> | ST-1 baseline | **pass** | `st1-baseline-1788419462251` | **basis: measured** |
> | ST-2 drop | **red on the admitted ceiling — and the ceiling is now measurably stale (G127, 4 Sep)**. Two `SCALE=1` runs at a fleet *verified stable at 2 replicas across all 38 samples* admitted 337,839 and 337,032 while shedding only 3.7% and 4.0%, with 0 unexpected — reproducing each other to **0.24%**. Both exceed `BURST_ADMISSION_BUDGET` (323,040) by 4.6% and so exit 99 on that single threshold. Every prior 2-replica run was capacity-limited at 41.5–53.6% shed and 165k–207k admitted, on the same offered load: **G115's hold-search bound (deployed 19:24 UTC, 3 Sep) cut peak pool waiters from 15–32 to 4–5 and roughly doubled resting-shape capacity**. So the red is a healthy origin admitting more than a budget derived before the fix, and `ORIGIN_CEILING_RPS` cannot be re-derived from these runs — both are offer-limited, making 4,218 rps a *lower bound*, and a gate set to a lower bound can never fire | `st2-drop-1788522096214`, `st2-drop-1788523289210` (both `SCALE=1`, fleet series recorded) | **basis: measured** — red is current; the *threshold* is stale-vintage per G127 |
> | ST-3 contention | **mechanism named, rate measured (G129, 4 Sep)**. Six `SCALE=1` runs: 1, 0, 0, 2, 0, 0 unexpected — **83 ppm across the first 36,000 iterations**, two of four runs affected, then a clean pair. The origin's own 5xx log names it: `deadlock detected`, **SQLSTATE 40P01 on `DELETE …/holds/…`**. Not an unwired classifier — `releaseHold` is already wrapped in `withDeadlockRetry`; the retry exhausted. The metric that should have measured this **could not distinguish absorbed from escaped** (one tally, incremented per attempt, so an escape counted three times); split and tested. The re-measured pair gives **6 absorbed, 0 exhausted on the two live replicas — 3–6 affected calls against a ~5 threshold, so the pre-registered fork stays open** and the jitter fix is `UNRESOLVED`. A first reading of 15 was a bare `sum()` over four replica series, two of them frozen by a mid-window deploy | `st3-contention-1788526483776`, `st3-contention-1788526553802`, `st3-contention-1788529949632`, `st3-contention-1788529984695` | **basis: measured** |
> | ST-4 failure | **blocked on operator** — not red, not green | `st4-failure-1788407068453` @ `634b950d` | **basis: operator-blocked** |
> | ST-5 rehearsal | **red on one threshold, confirmed 4 Sep from the run store.** `saff_hold_shed_rate` 54.4% against `rate<0.5` — G112's already-open bound, which the same read showed sits *inside* run-to-run spread (38.4 / 39.4 / 40.6 / 51.4% on unchanged code). Everything else held: `unexpected`, `exhausted` and `upstreamUnavailable` all **0** across 580,377 iterations, `saff_hold_seated_or_lost` **1.0**. The previously cited "red on `saff_hold_success` 0.2518" was never a verdict this scenario could produce — that metric was re-keyed out of ST-5's thresholds on 3 Sep (`9a24c1e8`, 00:43 UTC), 20 h before this run, so 0.2518 is an **ungated** emitted value read against a bound that no longer exists. The ≤1-per-~606k transport bound is independent and stands. **This run predates `dropWindowFloor`** (`58eeada5`) and so measured resting 2-replica shape | `st5-rehearsal-1788469874585` | **basis: measured** |
>
> **This board is the current one, corrected 4 Sep — three times.** It first showed
> ST-2, ST-3 and ST-5 at their 3 Sep states, which §12.6-§12.8 had already
> superseded the same night: §12 says it "supersedes §11's ST-3, ST-4 and
> ST-5 rows where they conflict", and nothing said it supersedes *this*
> table, which is the first thing a reader sees.
>
> The first correction then overshot in the other direction, boarding ST-2
> and ST-3 as **pass** on evidence that does not support it: ST-2's cited run
> was `SCALE=2`, and no `SCALE=1` run has executed under the current
> constants at all, while §12.8's *newest* ST-3 run is a threshold-failure
> the quoted `13 → 2 → 0` sequence stopped short of. Both directions are the
> same defect — a summary asserting more than its evidence, which is what
> G124 and G125 record.
>
> The third correction, 4 Sep, replaced ST-5's "verdict unconfirmed" with its
> actual verdict. The claim was not merely stale — its stated *reason* was
> wrong: the verdict was said to need "a run summary the sleeping cockpit
> cannot currently serve", when `loadtest.run` had stored every run's
> `summary` in Postgres the whole time. **A blocker attributed to the wrong
> component stays closed long after it has opened**, because nobody retries
> the path that was never broken. The same read then produced G112's
> spread finding, which had been sitting in the same rows unread.
>
> **ST-2's red is not the system misbehaving.**
> `saff_unexpected_total` is **0**, and the only failing threshold is
> `saff_admitted_total{endpoint:prequeue_join} count<323,040` against
> **349,007 admitted**.
>
> **The attribution in the previous revision was wrong, corrected 4 Sep
> (G126).** It read that removing the shm ceiling (G118) "raised real
> capacity past a bound derived under the old constraint", citing shedding
> falling from 147,430 to 18,287. Those two runs are `2bf5e1c8` at 04:10 and
> 04:17 — **the same sha, seven minutes apart** — and Mimir shows the fleet
> was **2 replicas then 5**. The shedding drop is fleet shape, not a capacity
> unlock; a code change cannot be the cause of a difference between two runs
> of the same code.
>
> The red still runs **toward caution** — the gate can only flag capacity
> *exceeding* the bound, never hide a regression — but the reason is now
> known: the band is keyed to an undeclared replica count (G126), so it is
> left red pending the re-key rather than widened to green on one sample.
>
> **ST-3's and ST-5's reds are one deferred defect plus one scoped residue.**
> ST-3's 13 failures are `errorCode:1050` (client timeout) + `status:0` —
> the G115 signature, named by the tags without a hypothesis cycle. ST-5's 88
> split **83 transport** (`errorCode:1211`/`1212`, `status=0`) and **5
> origin-answered 500s** (`errorCode:1500`) — the latter concentrated on
> `prequeue_join` (4) and `match_detail` (1). The shape was **stable** at 7
> replicas throughout, so not replica churn.
>
> **ST-5's disagreement is resolved, and it was a real production defect.**
> The scenario passed at 00:16 and failed at 05:00 with 8,473 unexpected —
> **both correct measurements**, of a fleet that traversed 2 → 3 replicas
> and one that traversed 2 → 10. At the higher concurrency Postgres
> exhausted its 62 MB `/dev/shm` arena and answered HTTP 500. Fixed by
> `max_parallel_workers_per_gather = 0` via DCS; the proof run at **7
> replicas** — the shape that broke — took `errorCode:1500` from **982 to
> 5** and seated-or-lost from 0.791 to **0.9989** (G118). An 88-failure
> residue remains — 83 transport plus **5 surviving 5xx**, filed with the
> label breakdown that names them.
>
> ST-4's run is a **harness-clean baseline that asserts nothing about
> §14.5**: no fault was injected, and the fleet was verified healthy through
> the window (Postgres leader in-region, Redis reachable). It is recorded
> that way rather than as a pass, because an uninjected green here is the
> exact vacuity G44 and G96 were filed for.
>
> The two ST-3 reds are **G94** (`saff_unexpected_total`) and **G115**
> (admitted-hold latency past the client's budget). Both are the *same
> defect* as G116's 70 unclassified failures — see "One defect, three
> reds" below.

The first full ST-1…ST-5 ladder executed at `SCALE=1` — §14.5's forecast load,
not a smoke fraction — followed by a **second ladder the same night** against a
corrected instrument. Every prior edition of this document reported `SCALE=0.1`
runs. [ROLLOUT.md](ROLLOUT.md) §4–§5 holds the narrative history.

**The first ladder's own arithmetic condemned two of its thresholds.** Both
were rewritten, redeployed, and re-run the same night; ST-4's fault was
injected for the first time. This edition reports the second ladder and keeps
the first only where the comparison is the evidence.

**What changed is what the gate can mean, not whether it is red.** ST-2 was
red on both ladders, and on both the origin-load threshold was the one that
fired — so this is not a case of a silent gate starting to speak. The old
threshold reported **1,910 req/s against 1,200**: a whole-run average of
*sends*, when the criterion is about *admissions during the burst*. It was
wrong by ~2× and still red, because the two errors compound in the same
direction. The rewritten threshold asserts the real quantity: ~308,000 joins
admitted over the fixed 80 s burst, ~3,850 req/s.

**Then the ceiling itself turned out to be folklore, and that inverts the
headline.** The 1,200 req/s in §14.5 was traceable to no constant, no pool
budget, and no measurement. Deriving it from the connection pool — Little's
Law over PgBouncer's server connections, with mean transaction time read from
the pooler's own accounting — gives **≈3,770 req/s**. The measured ~3,850 is
**2.1% past capacity**, not 3.2× past it, and the pooler's 1.17 ms mean wait
for a connection says the same thing independently. §4.5 has the derivation.

**The fixes stand, and the alarm is now closed — by a re-run, not an
edit.** Two earlier versions of this paragraph were wrong in opposite
directions. The first ended "the fixes stand, the alarm does not", which
hid a red gate. The second recorded a **2.12% breach** (3,850 admitted
against a 3,770 ceiling; `st2-drop.ts:49` sets `BURST_ADMISSION_BUDGET =
ORIGIN_CEILING_RPS × BURST_SECONDS` = 3,770 × 80 = **301,600**) and left it
open pending a pooler re-measurement.

**Re-run 2 Sep 2026 against production** (`st2-drop-1788344380903`,
`SCALE=1`, sha `8899b8ad`): **`verdict: pass`, `exitCode: 0`**, with
`saff_admitted_total{endpoint:prequeue_join} count<301600` reported **PASS**
by k6 itself. Every threshold in the scenario passed, `saff_unexpected_total`
was 0, and 110,837 joins were shed correctly.

The `duration` parameter does not weaken this: the burst stages are
build-time constants that never read `DURATION` (`st2-drop.ts:127-129`), so
the 80-second window the budget describes is identical between runs.

**§14.5's rule was followed, and it is what produced the green.**
`SPEC.md:4390-4395` forecloses the tempting repair in terms —
"**Recompute it, do not edit the threshold.** … the correct response is to
re-measure `PGBOUNCER_MEAN_XACT_MS`, not to widen a bound until the run goes
green." No bound was widened and `ORIGIN_CEILING_RPS` is unchanged at 3,770.

A pooler reading taken while the system was idle suggested a mean
transaction time of 18.08 ms, which would have raised the derived ceiling to
~3,872 and turned the red green by arithmetic alone. That reading was
discarded as regime-invalid: the 55.7 ms figure the ceiling derives from was
measured across 535,496 transactions *under* the ST ladder, and substituting
an idle-regime number would have been exactly the "widen until green" move
§14.5 prohibits.

**Two scenarios pass. Three are red, none of those reds is a threshold
defect, and one of them is now known to be correct behaviour.** ST-2 joined
ST-1 in the passing column on 2 Sep after the re-run above; its former red
was a capacity alarm against a number nobody could re-derive, and
re-deriving it — rather than editing it — is what closed it. **ST-3's red
was answered on 3 Sep and is the shedder working**: a five-point scale sweep
shows the shed ratio tracking population 25% → 90% with contention scaling
13×, against a fixture that asks for six buyers per seat. Separating a red
gate from a system finding is most of this document's work: the 28 Aug
edition conflated them, the 31 Aug ladder inherited two instruments that
could not mean what they said, and the criterion they fed was itself
unfounded.

**Read §11 before quoting any number measured on 3 Sep.** A Patroni
switchover during that day's failover test left the database leader in San
Francisco from 10:32 to 17:10 UTC, so every write in that window crossed the
Pacific at ~178 ms instead of ~2 ms (G103). Two conclusions recorded during
it — ST-4's error count and an ST-5 "requests that never complete" finding —
are affected, and the second reversed entirely once the leader was local.

**1 September adds §10: field runs against the deployed environment.** The
first Postgres failover after the classifier fix, the shed sweep that decides
G95, a burst with half the API fleet dead, and the start of the deployed soak
VR-9 requires. Two of its findings correct claims made earlier in this
document rather than adding new ones.

## 1. Scope and system under test

- **System:** the production SAFF deployment on Railway. First ladder at
  commit `bde604c3`; second ladder at **`d3647a47`**, which changes only the
  load harness and the Grafana dashboard — no `saff-api` code differs between
  the two, so the ST-2 verdict flip is attributable to the instrument alone.
  Region asia-southeast1. `saff-api` behind the elastic fleet (2 replicas at
  rest, autoscaled 2–6), PgBouncer transaction pooling, Patroni-managed
  Postgres. All services verified `SUCCESS` on their commit before the first
  run — a `SKIPPED` deployment means the running code is an older commit,
  which is how a report gets pinned to something that never ran. For the
  second ladder this was checked harder: `funnel.ts` inside the running
  container was byte-compared against the local file (md5 `c444129f`,
  identical), because a green deploy is a claim about a build, not about what
  is serving.
- **Harness:** `apps/saff-loadtest` (k6 **v2.0.0** — `railway.ts:2880` pins
  `aqua:grafana/k6@2.0.0`; an earlier version of this line said v2.2.0, which
  the 1 Sep build could not resolve), driven over `POST /api/run`.
  One run at a time, enforced by the service with a 409.
- **Scale:** `SCALE=1`. First ladder requested `duration=120s`; second used
  each scenario's own default, with ST-4 extended to `4m` to fit a fault
  window.
- **Site state:** `SITE_LOCKED=true`. Pre-launch, web and admin behind basic
  auth, no buyer path open. This is what made a destructive-seeding ladder
  safe to run at all; §14.5's freeze rule protects a live drop, and there is
  no live drop.

Each run records its own `gitSha`, so the pinning above is the harness's
claim rather than this document's.

## 2. Results

**Second ladder** — `d3647a47`, the instrument this report trusts:

**basis: stale-vintage** — every row below predates G115; G127 measured that G115 roughly doubled resting-shape join capacity, invalidating each constant these runs were judged against. Retained as history, not as current verdicts.

| Scenario | Verdict | Reqs | Admitted | Shed | Unexpected | Hold success |
|---|---|---|---|---|---|---|
| ST-1 baseline | **pass** (0) | 6,687 | 6,687 | 0 | 0 | — |
| ST-2 drop | red (99) | 367,369 | 326,129 | 41,240 | 0 | — |
| ST-3 contention | red (99) | 9,263 | 1,102 | 8,161 | 2 | — |
| ST-4 failure | red (99) | 24,161 | 11,980 | 12,107 | 1,566 | 38.8% (2,711/6,993) |
| ST-5 rehearsal | red (99) | 605,382 | 315,671 | 289,400 | 312 | 44.1% (3,186/7,227) |

Run ids: `st1-baseline-1788209454178`, `st2-drop-1788209568041`,
`st3-contention-1788209814466`, `st4-failure-1788215667938`,
`st5-rehearsal-1788210004378`.

**First ladder** — `bde604c3`, retained because ST-2's flip is the evidence
that the instrument work mattered:

**basis: stale-vintage** — as above: pre-G115 runs, invalidated by G127.

| Scenario | Verdict | Reqs | Rate | Shed | Unexpected | Hold success |
|---|---|---|---|---|---|---|
| ST-1 baseline | **pass** (0) | 13,583 | 110/s | 0 | 0 | 100% (601/601) |
| ST-2 drop | red (99) | 367,503 | 2,007/s | 42,270 | 0 | — |
| ST-3 contention | red (99) | 9,274 | 215/s | 8,535 | 1 | 3.0% (274/9,000) |
| ST-4 failure | red (99) | 14,417 | 108/s | 0 | 22 | 99.2% (3,523/3,551) |
| ST-5 rehearsal | red (99) | 607,118 | 2,484/s | 294,761 | 0 | 45.7% (3,316/7,251) |

Run ids: `st1-baseline-1788201564555`, `st2-drop-1788201765143`,
`st3-contention-1788201982895`, `st4-failure-1788202060190`,
`st5-rehearsal-1788202215523`.

**The `admitted` column is new and is the point.** `saff_admitted_total`
counts responses the origin actually served, excluding only the two cheap
deliberate refusals (`429`, `503 SHEDDING`). It partitions cleanly against
shed — ST-2: 326,129 + 41,240 = 367,369 = `http_reqs`, exactly — so origin
load can be stated without inferring it from sends.

**Reading the exit code, not the metric map.** k6 returns `99` for a threshold
breach and `0` only when every threshold held; `server/summary.ts:99-111`
makes that the source of truth. The raw `--summary-export` document spells a
held threshold as `false` under the metric's `thresholds` map — the boolean is
"breached", not "ok" — and reading it the other way inverts every verdict in
this table. The normalised `/api/summary` view exposes `ok` instead, which is
the surface to trust.

## 3. ST-1 — the 28 Aug capacity finding does not reproduce

**Every threshold held.** Shed 0 against `count==0`; unexpected 0; upstream
unavailable 0; exhausted 0; hold success 601/601 against `rate>0.95`;
`matches_list` p95 53 ms and `match_detail` p95 93 ms against 500;
`hold_best_available` p95 225 ms against 1,500. Peak pool waiters 0, peak
in-flight 4.

This directly overturns [ROLLOUT.md](ROLLOUT.md) §5's ST-1 record, which
measured **70 sheds cold and 160 warm** at `SCALE=1` on 28 Aug and walked the
onset down to find `SCALE=0.5` and `0.25` shedding zero. That finding was real
when taken. It does not survive the intervening work: the 28 Aug performance
pass, the class-1 shed mount on both hold writes (G87), and the windowed
pool-pressure signal (G86).

**G86 closed on `SCALE=0.5` evidence.** ST-1 at `SCALE=1` is strictly stronger
and agrees with that closure. The register entry is updated accordingly rather
than left resting on the weaker measurement.

**The second ladder agrees.** All eight thresholds held again at `SCALE=1`:
shed 0, unexpected 0, upstream unavailable 0, exhausted 0, both browse p95s
and `hold_best_available` p95 inside budget. `saff_admitted_total` and
`http_reqs` are both **6,687 — identical**, which is the arithmetic the new
counter must satisfy on a run with zero refusals, and a check on the
instrument itself rather than on the system.

## 4. ST-2 — the origin ran at its ceiling, and the ceiling was folklore

Two threshold-construction defects, then the finding — and then, a day later,
the discovery that the number all three were measured against had no
derivation at all.

**Read §4.5 before acting on anything in §4.1–§4.4.** Those sections are
preserved as written on 31 Aug, when the criterion was "≤ 1,200 req/s". That
figure turned out to be traceable to nothing, and the corrected ceiling —
≈3,770 req/s, derived from the connection pool — puts the same measured load
*at* capacity rather than 3.2× past it. The measurement defects were real and
their fixes stand; the alarm they raised was not.

### 4.1 The numerator counts refusals

`http_reqs{endpoint:prequeue_join}` counts
every request *sent*, including the 503s the prequeue shed. §14.5's criterion
is "sustained origin load ≤ 1,200 req/s" — what the origin *admits*. Admitted
is `sent − shed`.

### 4.2 The denominator is the whole run

The burst occupies 80 s of a 184 s run (`st2-drop.ts:66` starts it at 60 s;
`:76-78` ramp 10 s, hold 60 s, drain 10 s, summing to the `BURST_SECONDS` at
`:38`). A rate averaged over the idle remainder understates the burst by
~2.3×.

Corrected over the 80 s burst window:

| | Whole run (as thresholded) | Burst window |
|---|---|---|
| Sent | 1,910/s | 4,372/s |
| Shed | 231/s | 528/s |
| **Admitted** | — | **3,843/s** |

### 4.3 k6 was not the limit

349,725 joins sent against 350,000 offered — 99.9% — with 459 of 2,000
allowed VUs and 385 dropped iterations. The load was genuinely delivered, so
`http_reqs` here is achieved throughput rather than a generation ceiling.

### 4.4 The finding, and why this run cannot settle it alone

**The origin admitted 3,843 req/s, 3.2× §14.5's 1,200, with zero
unexpected errors and 60.8 ms p95.** `shedByClass` shows **class 1 never fired
at all** (`{"0":0,"1":0,"2":242,"3":77}` at the run's end). Class 1 is
"existential load" per §8.7; classes 2 and 3 shedding is cooperative
backpressure working as designed.

`st2-drop.ts:143` calls 1,200 req/s "a fixed property of the origin". Either
that number is stale relative to this deployment, or the shedder admits well
past what the spec intends. **This run cannot distinguish them**, and the
pool-waiter samples are not the tiebreaker they first appear to be:

- **The harness reads only the unlabelled series.** `runtime-sampler.ts:33-35`
  describes `saff_db_pool_waiters` as the deepest pool's queue, and the
  sampler's `parseMetrics` (`:134-146`) regex-matches only that unlabelled
  form. "Peak 9" is therefore one pool's queue as seen by the harness, not a
  fleet total.
- **But the per-pool series exists, and an earlier draft of this section said
  it did not.** `apps/saff-api/src/runtime/metrics.ts:138-146` emits
  `saff_db_pool_waiters` twice: unlabelled for the deepest pool, and
  **labelled `{pool}` with that pool's 90-second peak** — "so a burst shorter
  than the push interval is still visible". `autoscale.ts:757-760` tags the
  push with `replica`. So per-pool, per-replica, peak-retaining data is in
  Mimir for these runs; the 5 s sampling and single-replica scrape are
  limitations of the *harness snapshot*, not of the available data.
- **The sampler polls every 5 s** (`observability.ts:61`), so an 80 s burst
  yields ~16 point samples and the 10 s ramp yields two. The recorded series
  (`0,0,0,3,1,1,0,…,9,0,0`) has exactly the spiky shape undersampling
  produces — which is precisely what the labelled peak series exists to
  survive.

So the harness's own fleet data narrows the question without deciding it, but
the data that would decide it has been collected all along. Reading it needed
a Mimir read credential this environment did not hold — **resolved 2 Sep**,
see §7: the `query-mimir.yml` route works, so this is no longer a blocker and
the labelled series can now be read for any run still inside retention. What
*is* established is that ST-2's old threshold could not answer it either,
because it measured the wrong quantity over the wrong window.

**88.6% of admitted joins contended** (309,955 of 349,725). Expected for a
prequeue under burst, and recorded so "absorbed cleanly" is not read as
"frictionless".

### 4.5 The ceiling was never derived, and deriving it inverts the finding

Everything above measures origin load correctly. None of it asks whether
1,200 req/s was the right thing to measure against. It was not.

`grep` finds that number in exactly two places: §14.5's criteria table and a
`railway.ts` docblock quoting it. No constant, no pool budget, no
measurement — and unchanged while the deployment grew past three times it.

**A write transaction holds one pooler server connection for its duration**,
and API replicas cannot create more of those, so the write path's real
ceiling is Little's Law over the pool:

$$\text{ceiling} = \frac{\text{server connections}}{\text{mean transaction time}}$$

Both inputs are observed. The numerator is `PGBOUNCER_DEFAULT_POOL_SIZE ×
PGBOUNCER_REPLICAS` — the same product §14.4's connection budget already
counts. The denominator comes from the pooler's own accounting: `SHOW
STATS_TOTALS` on `saff-pgbouncer`, read 1 Sep 2026 after the ladder, gave
**55.7 ms** mean transaction time across 535,496 transactions and a
**1.17 ms** mean wait for a server connection.

| | |
|---|---|
| Server connections | 70 × 3 = 210 |
| Mean transaction time | 55.7 ms |
| **Derived ceiling** | **≈3,770 req/s** |
| ST-2's measured admitted rate | ~3,850 req/s |
| **Agreement** | **2.12% over** (3,850 ÷ 3,770 = 1.0212) |
| Old §14.5 figure | 1,200 req/s — **3.1× low** |

**So the origin was not 3.2× over capacity; it was at 1.02× a capacity nobody
had computed.** The 1.17 ms wait says the same thing independently: a pool
that queues for one millisecond under a 5,000 joins/s burst is at its knee,
not three times past it. §4's original headline was the correct reading of a
wrong number.

That does not retire the fixes in §4.1 — measuring sends over whole-run wall
time would still be wrong against any ceiling. It shrinks the alarm by ~150×
without retiring it: 1.02× a derived ceiling is a different finding from 3.2×
a folklore one, but it is still *over*, and `st2-drop.ts:49`'s
`BURST_ADMISSION_BUDGET` of 301,600 was breached at ~308,000 on the 31 Aug
run. **Re-run 2 Sep 2026 (`st2-drop-1788344380903`, sha `8899b8ad`): the
same threshold reports PASS.** The ceiling was not changed — see §1.

**ST-2's gate therefore remains red, and correctly so.** §14.5's rule is
explicit (`SPEC.md:4390-4395`): "**Recompute it, do not edit the threshold.**
… the correct response is to re-measure `PGBOUNCER_MEAN_XACT_MS`, not to
widen a bound until the run goes green." The 55.7 ms mean was measured 1 Sep;
every §4.5 number depends on it, and a re-measure is the only legitimate way
this row goes green. That needs `SHOW STATS_TOTALS` on
`pgbouncer.railway.internal`, which has no public proxy — so it is an
operator action from inside the private network, not a documentation change.

**Landed as a derivation, not a new constant.** `railway.ts` computes
`ORIGIN_CEILING_RPS` from the pooler numbers it already owns and passes it to
the loadtest service; `st2-drop.ts` derives its bound as `ceiling × 80 s`
instead of hardcoding 96,000. Five tests in `connection-budget-guard.test.ts`
pin the derivation to its inputs and were verified to have teeth: reverting
to a hardcoded `"1200"` turns all five red, one of them written for precisely
that regression.

**Open, and separate.** Whether the *shedder* should admit at the pool's
sustainable rate is a different question — the ceiling states what the pool
can carry, not what leaves headroom for the class-0 payment path. That is
G95's territory and is unaffected.

## 5. ST-3 — a 3% hold success rate is the design

The section is deliberately undersized (§14.5), so most attempts *must* fail.
The partition very nearly closes: 8,535 shed + 193 contended + 274 held =
9,002 against 9,000 iterations — a **2-count excess**, not the exact closure
an earlier version of this sentence claimed. Two attempts are double-counted
or double-classified; the residual is small enough not to move any verdict
here, and is recorded rather than rounded away. The run ended at 44 s of a requested 120 s having
exhausted its iteration budget — complete, not truncated.

Three thresholds breached, and all three are real signals — an earlier
edition called one of them a defect and was wrong:

- **`saff_shed_total count<1350` — real, and the ladder's largest open
  question.** 8,535 shed of 9,000 attempts. This was originally written up as
  "an absolute count against a variable-duration run"; that is false. See the
  withdrawal below. **basis: stale-vintage** — the 8,535 of 9,000 comes from the first ladder above, which G127 invalidated along with every pre-G115 constant. The four `SCALE=1` runs of 4 Sep (G129) supersede it.
- **`hold_best_available p(99)<3000` — real.** p95 was 17.1 s, max 42.2 s,
  against a 3 s p99 budget. Peak pool waiters 55 and peak in-flight 60 on
  three registered pools: this is genuine saturation under contention.
- **`saff_unexpected_total count==0` — one occurrence.** See §7.

**The second ladder reproduces the shed finding and sharpens the errors.**
8,161 shed against the same `count<1350` budget, with 1,102 admitted — and
`saff_unexpected_total` at 2, both at **`hold_release`**. The first ladder's
single unexpected could not be attributed to an endpoint at all.

The shed budget was re-examined during this work and the earlier claim
against it withdrawn: `st3-contention.ts:132` computes `count<1350` from
`ITERATIONS * 0.15` where the executor is `per-vu-iterations`, so the attempt
count is fixed at 9,000 regardless of wall time. The budget's base is
attempts, which is correct for a fixed-population race. What the red means is
[G95](GAPS.md)'s question — a 94.8% shed rate — not a threshold defect.

## 6. ST-4 — two faults injected, and only one of them has teeth

`st4-failure.ts:20-33` is explicit: the script is "the *load and observation*
half" and "does **not** inject the fault". Killing Redis or failing over
Postgres is an operator action against Railway, deliberately outside a load
script's credentials. The first ladder ran only the observer, so it exercised
nothing. The second ladder injected two of §14.5's nine fault rows for real.

### 6.1 Redis restart — a null result the run could not have produced otherwise

`saff-redis` restarted mid-run, a ~27 s outage confirmed by watching the
deployment go `DEPLOYING` → `SUCCESS`. Result: **0 shed, 0 upstream
unavailable, 0 unexpected, health probe 0.000, exit 0**, and the scenario's
own honesty line — *"PASS — no degradation observed (was the fault actually
injected?)"*.

**The run cannot adjudicate the row it appears to test, and an earlier draft
of this section claimed it could.** §14.5's Redis row read "Joins 503
briefly" at the time of this run, and ST-4 sent no joins: as of `d3647a47`,
`st4-failure.ts`'s `traffic()` was browse → browse → hold → release and
`joinPrequeue` appeared nowhere in the file. A run with no join traffic
cannot observe a join 503 whatever the implementation does, so the zeros
above are a property of the scenario's request mix, not evidence about Redis.

**Both halves of that paragraph are now historical, and the source has
moved.** `st4-failure.ts:9` imports `completeOrder` and `:11` imports
`joinPrequeue`; `traffic()` at `:119` calls `joinPrequeue()` at `:124`, a
change the file dates itself at `:112-114` ("Until 1 Sep 2026 this function
was browse → browse → hold → release"). And §14.5's row no longer says what
is quoted above — `SPEC.md:4403` now reads "**Joins keep succeeding**". The
paragraph is kept in the past tense because it is the reason the 31 Aug Redis
result was void; it is not a live description of the harness.

**Separately, and from source rather than from this run, the row looks
obsolete.** Three sites make a join 503 unreachable on a Redis fault:
`queue/prequeue.ts:46` puts the record in Postgres ("the Redis copy is the hot
read path, not the record"), `queue/ticket.ts:369` fails the nonce check open,
and `runtime/redis-store.ts:39` states the tier rule — "**Fail open, always,
per §4.4's own loss table.**" §4.4 and §14.5 disagree, and the code implements
§4.4. That is a documentation conflict established by reading, and it is not
the same claim as "this run demonstrated it".

**Not recorded as an ST-4 pass**, for the stronger of the two reasons: the
scenario is structurally blind to this row. Closing it needs join traffic in
ST-4 — or the row reassigned to ST-2, which does drive joins. Filed as
[G96](GAPS.md).

### 6.2 Postgres failover — the row with teeth

`saff-postgres` was the leader (`pg_is_in_recovery()` = `f`); restarted at
t+64s. `saff-postgres-1` was **promoted within 17 s**, verified by polling the
standby until it answered `f`. A genuine Patroni election, and afterwards the
old leader rejoined as standby — the cluster self-healed.

| Signal | Redis row | Postgres row |
|---|---|---|
| shed | 0 | **12,107** |
| upstream unavailable | 0 | 48 |
| unexpected | 0 | **1,566** |
| health probe failure rate | 0.000 | 0.100 (threshold < 0.5) |
| exit | 0 (vacuous) | **99** |

The scenario's verdict: *"FAIL — the origin returned errors the ladder does
not permit"*.

**What is right.** The system sheds rather than collapsing — 12,107 deliberate
refusals is §6.10's ladder engaging — and it recovers: the health probe ended
inside its RTO threshold, so the origin was serving normally by the end of the
run.

**What is wrong, and §7's arithmetic makes it worse than it first looked.**
§14.5 expects writes to *pause*; a paused write is a `503`. 1,566 responses
were something else, and `st4-failure.ts:78-80` states the rule they break:
"under a fault the system sheds, it does not error. A `503 SHEDDING` is a
pass; a `500` is the failure this gate exists to catch."

Of those 1,566, only **74 were connection-level** (`status=0`, derived in §7
from the closed `admitted + shed` partition). The other **~1,492 were real
responses the origin generated** — it stayed up, answered, and answered with
something the ladder forbids. This is not a network artefact or a client
timeout: it is the failover path returning errors where it should return
backpressure. Filed as [G97](GAPS.md).

**The oversell invariant held**, checked against the promoted leader using the
table the real gate uses. `apps/saff-api/scripts/gate/st3-oversell.ts:686-690` computes
`sum(c − 1)` over **`seatSale`** grouped by `seatInventoryId` — not
`orderItem`, which an earlier draft of this section queried and which is
empty. Measured after the failover: **390 sales, 390 SOLD seats, 0
duplicates**. The three agree exactly, so every sold seat carries exactly one
sale across a real leader election.

It still does not prove a buy-through under failover is safe: no ST scenario
completes a purchase, so those 390 sales predate the run. What is shown is
that the failover corrupted no existing sale state. The full harness also
asserts two invariants this query does not — sold seats with no permanent
claim, and sold seats with another live claim — and was not run here.

**Three of nine fault rows were exercised as of this section's runs** — Redis
failover (§6.1), Postgres failover (§6.2) and the 6-hour soak (§10.6, passed
2 Sep). **Superseded 3 Sep**: the kill-50% row was injected and passed, making
it four of nine with five remaining. §11 carries the current split; this
paragraph states what was true when §6 was written.

The row count itself has been wrong twice. Earlier drafts counted the table as
six rows and then as eight; it has **nine** — the bolded soak row is one of
them, which is why "eight" undercounted even after the first correction. The
remaining five are also no longer named as `gate` kill rows: §4.1 states there
is no separate `gate` deployable, and the SPEC table now names the API fleet
(§11).

## 7. The open item: unexpected errors, now attributable

`saff_unexpected_total` counts responses outside the design's permitted set —
`funnel.ts`, reached only after 409/410 are classified as contention or
exhaustion.

The first ladder recorded 23 across the ladder and **could not name them**: k6
only exports a submetric when a threshold references it, so the `endpoint` and
`status` tags were absent from the artefacts. That was [G94](GAPS.md).

The second ladder names the endpoints from the run output's check lines, and
the **counter arithmetic settles the status codes without Mimir at all**.

`classify` (`funnel.ts:110-290`) is a total partition: every exit path
increments `admitted` or `shed` exactly once — 2xx, 429, both 503 branches,
409/410, and non-zero unexpected — with exactly one exception. `status=0`,
k6's code for a request that never completed, increments neither (`:189`).
All six HTTP calls in the harness route through it and no scenario file
issues HTTP of its own, so `http_reqs − (admitted + shed)` **is** the
`status=0` count.

| Run | Requests | admitted + shed | `status=0` | Real HTTP codes | Endpoints |
|---|---|---|---|---|---|
| ST-1 | 6,687 | 6,687 | 0 | 0 | — |
| ST-2 | 367,369 | 367,369 | 0 | 0 | — |
| ST-3 | 9,263 | 9,263 | 0 | **2** | `hold_release` |
| ST-4 | 24,161 | 24,087 | 74 | **1,492** | during failover |
| ST-5 | 605,382 | 605,071 | **311** | 1 | four endpoints |

**The two large counts have opposite causes, and an earlier draft of this
section called both unread.**

- **ST-5's 312 are connection-level**: 311 never completed. Same signature as
  the first ladder's 23, now at 605k requests instead of 14k — which is what
  makes G94's "healthy idle origin" framing wrong and the failure mode
  volume-independent.
- **ST-4's 1,566 are mostly real responses**: ~1,492 carried actual HTTP
  codes. The origin answered with something outside the permitted set during
  the failover. That is a §6.10 violation on its own terms, not a network
  artefact, and it makes §6.2's finding stronger rather than weaker.

**What remains unread is which codes**, not whether they exist: the
`{endpoint, status}` series are in Mimir as
`k6_saff_unexpected_total_total{testid=...}`.

**The credential blocker described here is resolved (2 Sep).** It read that
reading them "needs a credential this environment does not hold" —
`saff-loadtest`'s `GRAFANA_RW_TOKEN` being write-only. The route around it is
the `query-mimir.yml` workflow, which runs the query from CI where the
credentials live, and it was used repeatedly on 2 Sep: G95's per-arm
attribution, G96's 326-join count, and G99's replica-count samples all came
from it. Two things had to be learned first, both now written down in
`apps/saff-loadtest/src/server/runner.ts`: k6 exports a counter named
`saff_shed_total` as **`k6_saff_shed_total_total`** (the output prefixes
`k6_` and suffixes `_total`), and querying the un-prefixed name silently
returns `saff-api`'s own always-on series of the same name — right metric
name, wrong emitter. A correct query needs the `k6_` prefix *and* a `testid`
matcher.

On the first ladder these appeared **with no fault injected**, at 108 req/s
with idle pools — steady state should not produce them. On the second, ST-4's
1,566 are explained by the failover; ST-5's 312 across four endpoints under
saturation are not, and remain the open question.

## 8. ST-5 — orderly saturation

Second ladder: hold success **44.1%** (3,186/7,227) against `rate>0.5`, with
315,671 admitted, 289,400 shed, and **312 unexpected** across 605,382
requests. The first ladder measured 45.7% with zero unexpected — the hold
figure reproduces closely, the errors do not.

Half the buyers not seating under a full dress rehearsal is a capacity
statement, not a correctness one. But the 312 unexpected responses are a
correctness question, and they are the difference between the two runs:
**ST-5's errors appear only in the run whose instrument counts admissions.**
That is suggestive rather than conclusive — the counter change cannot create
errors, so the likelier reading is run-to-run variance under saturation, which
is precisely why §7 leaves it open.

Whether 44% is acceptable is a product question about the drop's shape.

**Current verdict, recovered 4 Sep 2026 — and the gate above is no longer
the one that runs.** The `rate>0.5` hold-success threshold this section
argues against was re-keyed out of ST-5 by G104 (`9a24c1e8`); the paragraphs
above are retained as the measurement history that produced that change, not
as a live gate. The newest run (`st5-rehearsal-1788469874585`, 240 s) breaks
exactly one threshold — `saff_hold_shed_rate` **54.4%** against `rate<0.5` —
while `unexpected`, `exhausted` and `upstream_unavailable` are all **zero**
across 580,377 iterations and `saff_hold_seated_or_lost` is **1.0**. The 312
unexpected discussed above therefore did **not** reproduce: this run has none.

**Read from Postgres, not from the cockpit.** `loadtest.run.summary` has
stored every run's full k6 export since G105; the "sleeping cockpit" that was
cited as blocking this verdict was never the only path to it.

**Three cautions for whoever reads the 54.4%.** First, it is not cleanly
separable from a pass: four 60 s runs on identical code read 38.4 / 39.4 /
40.6 / **51.4**%, so the bound sits inside the spread — see G112, which this
read reopened with evidence rather than argument.

Second, the spread is **fleet shape, measured**: those runs ran at 7, 7, 4
and 2 replicas respectively (`r = −0.981`, ~3.3 points of hold shed per
replica). An earlier revision of this paragraph said they "measured the
resting 2-replica shape" — only the 240 s run did. Hold shed is a hold-path
metric throughout: `saff_hold_shed_rate` counts one sample per hold attempt,
so this run's ~336,230 prequeue-join sheds are absent from it by
construction.

Third, and most easily misread: **none of this revises the verdict.** ST-5's
newest run is a `threshold-failure` and remains one. Dating it against
`dropWindowFloor` explains what shape it measured; it cannot turn a red into
a pass, and no run has yet executed at floor shape. The fitted 36.1% at 8
replicas is a prediction with a stated falsifier (G112), not a result.

## 9. What this ladder does not establish

- **Sustained duration.** Every run is 2–4 minutes. Nothing here observes pool
  exhaustion, memory growth, or replica churn over a drop's real length.
- **Four of §14.5's nine fault rows**, as of 4 Sep (clock skew closed by falsifier): kill *all* API replicas,
  `+500 ms on Postgres`, the Cloudflare-challenge row and the payment-provider
  row. The five exercised are Redis failover, Postgres failover, the 6-hour
  soak, the kill-50% row injected 3 Sep, and clock skew — falsified 4 Sep in
  `prequeue.pg.test.ts` rather than injected, because its two defences (the
  server-clock comparison and `exchangeAfter` inside the receipt HMAC) are
  lines this repository owns; the soak,
  Redis and kill-50% pass, Postgres failover does **not** (exit 99, G97
  reopened). This bullet has been wrong twice: an earlier draft said "seven is
  the defensible number" because the Redis row's run was blind to the outage,
  superseded 2 Sep by an independent TCP probe (G96 closed); it then said six
  until the kill-50% injection. §11 is authoritative for the current split.
- **Whether the shedder's admission rate is the right one.** The ceiling
  question is closed — §4.5 derives it from the connection pool and the
  measured load sits within 2% of it. What that does *not* settle is whether
  admitting at the pool's sustainable rate leaves enough headroom for the
  class-0 payment path. **Answered 2 Sep, and not by either framing:**
  per-arm attribution put every ST-3 refusal on the `pool` arm, but the pool
  is not undersized — the same endpoint holds for 158 ms with zero waiters
  under the soak and 4,070 ms with 32 waiters under ST-3, on identical
  infrastructure. It is **lock contention**: 510 selectors on one section
  serialise on the same rows, each waiting inside a transaction that occupies
  a connection. So it is neither a shed-threshold question nor a
  `DB_POOL_MAX` one — the lever is time-under-lock, or a fixture that spreads
  buyers the way a drop does. See G95 and §10.2.
- **Oversell under a completed purchase.** §6.2's check passed on the right **basis: operator-blocked** — no ST scenario buys through, so a sale committed *during* a failover requires an operator-driven fault window.
  table (`seatSale`: 390 sales, 0 duplicates) but no ST scenario buys
  through, so those sales predate the run and the invariant is untested where
  it is hardest to hold — a sale committed *during* a failover.
- **That the dashboards render these runs.** The deploy gate proves the
  datasource link answers a panel query through Grafana's own proxy
  (`scripts/check-datasource.ts`, run against both the Prometheus and Loki
  datasources on every apply), and all **14** `k6_*` series the dashboard queries
  resolve — nine emitted by `funnel.ts` and five k6 built-ins — under the
  naming rule recorded at `opentofu/grafana/main.tf:506-523`. That is
  correctness by construction, not observation: nobody has loaded a panel
  filtered to one of tonight's `testid`s. This is no longer blocked on a
  credential (§7) — the `query-mimir.yml` route works — but loading a panel
  is a browser action nobody has taken.
- **Which HTTP codes ST-4's ~1,492 real errors carried.** §7's partition
  arithmetic establishes that they *were* real responses rather than dropped
  connections, and that ST-5's 312 are the opposite (311 never completed).
  **The read path is no longer the blocker** (§7): the 2 Sep failover run's
  errors were partitioned exactly this way, and came back `errorCode` 1500 —
  HTTP `500` — on 21 of 22. What has not been done is the same partition
  against the *older* ladder's series, which may have aged out of Mimir's
  retention; the runs are cheap to repeat if the answer still matters.
- **Threshold correctness generally.** **One** of five scenarios (ST-2)
  carried two thresholds that measured the wrong quantity or window — an
  earlier version said two scenarios, before §5 withdrew the ST-3 half. Both were found by
  arithmetic against the run's own numbers, not by a gate — which is an
  argument for auditing the remaining four rather than trusting them.

## 10. 1 September — field runs against the deployed environment

The 31 Aug ladder ran before two fixes landed and left five questions open.
This section reports the runs that answered four of them, plus two corrections
to claims made above.

All runs were driven through the cockpit from inside the private network
against the deployed environment (`4b0909aa`, and `6bbe0ef8` for the classifier
re-test).

### 10.1 ST-4 with a real leader failover — G97's fix, verified and incomplete

`st4-failure-1788283312778`, `scale: 1`, 60 s. `saff-postgres-1` was the leader
(`pg_is_in_recovery` = `f`); restarted at t+~29 s, election confirmed by polling
both nodes until the flags swapped.

| Signal | 31 Aug (pre-fix) | 1 Sep (post-fix) |
| --- | --- | --- |
| shed (deliberate refusals) | 12,107 | 2,210 |
| upstream unavailable | 48 | 1,344 |
| **unexpected failures** | **1,566** | **53** |
| health probe failure rate | 0.100 | 0.353 (threshold < 0.5) |

**The classifier wire works.** The error mass moved out of `unexpected` and
into `upstream unavailable` — precisely the remap §7's fix promised, and the
first field evidence that it does what its tests claim.

**The 53 residual were one shape the fix could not see.** PgBouncer, when its
own login to the server keeps failing during a failover, returns *"server login
has been failing, cached error: server conn crashed? (server_login_retry)"*.
The ORM's error normaliser preserves no SQLSTATE for it, so it reaches the
handler message-only and every code branch in `connectionTransient` missed it.
Fixed the same night (`6bbe0ef8`) by matching the message, pinned by a test
carrying the verbatim string beside the other message-only shape.

**Re-exercised 2 Sep, and the class is still not empty.** A real Patroni
failover inside the run window (`st4-failure-1788316183487`) produced **22
unexpected**, 21 of them HTTP `500` on write paths — `hold_best_available`
15, `order_create` 4, `hold_release` 2, zero on reads — alongside 567
permitted `503`s. The message-matching fix works (that is what the 567 are),
but some shape still escapes it. `1,566 → 53 → 22` is real progress and G97
is **reopened** rather than re-closed at a lower number: §6.10's ladder
permits `503`, not `500`.

### 10.2 The ST-3 sweep — G95, decided three times

Three runs at `scale` 0.17 / 0.33 / 1 (`st3-contention-1788284248431`,
`...369814`, `...425883`). **Superseded twice — read to the end of this
section before quoting the table below.**

| scale | shed | budget | contended | held | unexpected | class 1 (this run) |
| --- | --- | --- | --- | --- | --- | --- |
| 0.17 | 1,079 | < 230 | 272 | 182 | 2 | 117 |
| 0.33 | 2,430 | < 446 | 352 | 189 | 0 | 298 |
| 1 | 8,404 | < 1,350 | 335 | 264 | 2 | 787 |

Class-1 figures are deltas of the API's cumulative per-class sampler across
each run; the raw sampler values are process-lifetime counts.

**Reading 1 wins.** The shed ratio does not depart from budget at some knee —
it is over budget at the *smallest* scale (1,079 against 230, at 510 VUs) and
grows roughly linearly with offered load, with class 1 dominant throughout and
the origin healthy at every point (pool waiters peaked at 55; hold latency p95
16–21 s under queueing). 510 concurrent selectors is not existential load, so
class 1 is shedding well below the bar its own policy sets.

What this does *not* settle is which of the two class-1 arms fired — sustained
pool waiters, or in-flight concurrency against the ceiling. The policy's
condition is a disjunction and the sweep cannot see which side was true, so the
tuning change needs per-arm attribution first. Widening the 15% budget remains
foreclosed for the reason §7 gives.

**Superseded 2 Sep 2026 — the attribution ran and reversed this verdict.**
`st3-contention-1788313380410` at `scale 0.17` with `X-Shed-Arm` shipping put
**all 1,183 refusals on the `pool` arm and none on concurrency**. The
concurrency arm was not merely quiet but unreachable: its trigger is
`ceiling() × CLASS_FRACTION[1]` = 60 in-flight per replica, and the peak was
37.

**The pool arm fired because transactions were long, not because the pool was
small.** The comparison that settles it is the 6-hour soak, which ran the
same endpoint against the same pool with buyers spread across the catalog:

| Fixture | Mean hold | p(95) | Peak pool waiters |
| --- | --- | --- | --- |
| Soak, `scale 0.3`, catalog-wide | **158 ms** | 206 ms | **0** |
| ST-3, `scale 0.17`, 510 selectors on one section | **4,070 ms** | 15,240 ms | **32** |

26× slower on identical infrastructure. I-2's exclusion constraint serialises
contenders on the same seat rows, and each waits *inside* a transaction while
holding a connection — so the pool drains through duration, not through
arrival rate. At 158 ms these 10 connections sustain 63 holds/s against ST-3's
offered 38.2.

So class 1 is not shedding below its own bar. Raising `CLASS_WAITER_OFFSET[1]`
would admit more contenders into a lock queue and convert fast refusals into
slow timeouts — the G87 failure the arm exists to prevent — and a larger pool
would only hold more connections blocked on the same rows. The open question
is whether 510 selectors on a single section represents a real drop; if it
does, the lever is time-under-lock. See G95.

*(A first version of this paragraph said the admitted traffic oversubscribes
the pool 5.1×. That multiplied throughput by the wall-clock hold time, which
already contains the queueing it was being used to prove. Withdrawn.)*

**Superseded again 3 Sep 2026 — "Reading 1 wins" is withdrawn, and the
reason is a fixture artefact this section could not have seen.** The three
runs above were driven without re-seeding between them, so each inherited the
previous run's held inventory. Re-run as five points with `st3:seed` before
every point, so each sees a full 500-seat section:

| selectors | iterations | shed | shed % | budget (15%) | contended | held | exhausted |
|---|---|---|---|---|---|---|---|
| 150 | 450 | 114 | **25.3%** | 68 | 36 | 300 | 0 |
| 300 | 900 | 477 | 53.0% | 135 | 154 | 275 | 0 |
| 501 | 1,503 | 916 | 60.9% | 225 | 148 | 443 | 0 |
| 999 | 2,997 | 2,310 | 77.1% | 450 | 292 | 403 | 0 |
| 3,000 | 9,000 | 8,124 | **90.3%** | 1,350 | 474 | 409 | 0 |

The ratio **does** depart with scale: 25.3% at 150 selectors, climbing
monotonically to 90.3% at 3,000, with `contended` rising 13× across a 20×
population range and `exhausted` at zero throughout. That is Reading 2, not
Reading 1 — at low population the shedder barely engages, so class 1 is not
shedding below its own bar, and the budget is measuring the right quantity.

**What the unseeded runs were measuring instead.** Without a reset the
section drains: measured immediately after three consecutive runs it held
**21 of 500 seats available**. A run starting from that state refuses almost
everything regardless of population, which is why the earlier three points
looked flat and over-budget at every scale. (The depletion is transient, not
a leak — the same fixture read 478 available a few minutes later, so holds do
release; the snapshot was taken inside the last run's hold window.)

**The conclusion in the paragraph above survives, by a different route.**
Class 1 is not mis-tuned, the pool arm fires through time-under-lock rather
than pool size, and widening the budget stays foreclosed. What changes is the
open question: it is no longer "is class 1 over-eager" but "is 3,000
selectors against a 500-seat section a real drop". A hot section at this
venue holds ~3,200 seats (`GENERAL_NORTH_WEST.UPPER/O`, measured at 3,246),
so the fixture is roughly 20× more contended per seat than the venue can
produce.

### 10.3 ST-2 with half the API fleet dead

`st2-drop-1788285701973`, `scale: 1`. One of two API replicas was killed before
the burst and did not return (see §10.5), so the whole burst ran at half
capacity.

245,138 joins admitted at 1,338/s, join p95 **103 ms**, 277 unexpected across
~365,000 requests (0.076%) — the failures being dial timeouts against the dead
replica's address. The join p99 < 2,000 ms threshold was crossed: a tail on the
single surviving replica absorbing the entire burst. Expected degradation at
half capacity rather than a defect, but it is the first measurement of what
"kill 50% of the fleet mid-burst" actually costs: joins continue, latency tails.

### 10.4 The oversell gate is still vacuous under load — and §9 understated why

§9 says "no ST scenario buys through". A buy-through path was added on 1 Sep
and ST-4 exercised it: **100 orders completed** during the failover window.

It changes nothing about the invariant, because those are not sales.
`createOrder` deliberately stops at `DRAFT`; `SeatSale` rows are written at
capture, behind a payment the load suite does not drive. Measured directly:
the 100 orders all went `DRAFT` → `EXPIRED` and `seatSale` gained **zero** rows
in the run window.

The scenario comment claiming "the `seatSale` rows land at order creation" was
wrong and has been corrected, along with the counter's name in every summary.
**A sale committed during a failover remains untested**, and now demonstrably
so rather than by inference. Closing it needs a load path through capture —
the SSLCommerz sandbox adapter, or an out-of-band settlement script — not a
test-mode branch in production code.

The full oversell harness was run against the live database after the day's
runs and passed on all three of its invariants (0 duplicate sales, 0 sold
without permanent claim, 0 sold with a competing live claim) — on sales its own
harness seeds.

### 10.5 Two operational findings the runs produced

**A cleanly-exiting API replica is never restarted.** `kill 1` on one replica
at 17:55 UTC left it `exited` at 18:04 and through the entire ST-2 run;
only a manual redeploy restored the fleet. The cause is not a broken platform
default: `kill 1` sends `SIGTERM`, the drain handler completes and the process
exits **zero**, and the platform's default policy restarts only *failed* exits.
A crash (non-zero) would have been restarted. The consequence stands either
way — a gracefully-draining replica that self-terminates is gone until a human
notices — and the fix is to declare `restartPolicyType: "ALWAYS"` on the API
service. This also blocks the "kill *all* replicas for 60 s" fault row: run
under the current config, it would measure operator reaction time rather than
recovery.

**A deploy kills a running soak.** The first 6-hour soak attempt died at
t+38 min when a documentation commit touching two scenario files triggered a
rebuild of the cockpit service, replacing the container. Recording the soak's
launch is what ended it. The service watches six paths
(`apps/saff-loadtest/**`, `packages/retry/**`, `packages/basic-auth/**`,
`package.json`, `bun.lock`, `mise.toml`); during a long run the safe commit set
is documentation only. Finished runs are not at risk — they persist to
Postgres and remain readable after a restart.

### 10.6 Still open after this day

- ~~**The 6-hour deployed soak.**~~ **Completed 00:47 UTC 2 Sep — passes, and
  VR-9 is closed.** `st1-baseline-1788288434652`, `scale: 0.3`, seeded
  catalog, exit 0. 367,165 iterations over 6h00m03s, **777,529 requests with
  zero failures**, zero interrupted iterations. Every threshold green: browse
  p(95) 205.68ms, detail 93.07ms, hold 55.94ms; `saff_hold_success` 100.00%
  (43,199 / 43,199); `saff_shed_total`, `saff_unexpected_total`,
  `saff_exhausted_total` and `saff_upstream_unavailable_total` all 0. Container
  RSS from the platform's own `saff_container_anon_bytes` series: **−2.15
  MiB/h per replica** against an 8 MiB/h limit — memory fell 12.5 MiB per
  replica, with the midpoint above both ends, which is churn rather than a
  leak. 4,323 in-run health samples, all healthy, peak pool waiters 0. The
  single `level=error` line is k6's Prometheus stale-marker cleanup after the
  load stopped.
- ~~**A cleanly-exiting replica is still not replaced**~~ **(G99) — no
  longer reproducible as of 3 Sep.** The restart policy was changed to
  `ALWAYS` on the strength of the vendor's "restarts every time it stops";
  re-measuring with the policy live on 2 Sep showed `kill 1` leaving the
  fleet at `1/2 running` for 14 minutes, and the reading was that a process
  exiting 0 is classified *completed*, which no policy restarts. **Two
  further measurements on 3 Sep contradict that**: `kill 1` mid-burst
  (`st2-drop-1788346496688`) and `kill 1` on an idle fleet both self-healed
  to `2/2` in under a minute, unaided — the replacement even overshot to
  `4/2` under load before draining back. The drain path still exits 0, so
  the proposed mechanism is refuted rather than avoided. It therefore no
  longer blocks the kill-all-replicas fault row; that row's remaining
  blocker is the `401`/`DYNAMIC` public surface.
- **Requests that never complete** (G94). Present at every load level with a
  rate not proportional to volume. The 2 Sep runs added the discriminator:
  `errorCode` is now tagged, and k6's ranges separate a request that never
  reached the origin (1200s, `status=0`) from one the origin answered with a
  5xx (1500s). The origin-side half landed 3 Sep — the API access log now
  records every request with status and duration, so a `status=0` at the
  client can be checked against whether the origin ever saw it (G98, closed).
  **Not reproduced since**: with the leader local, `st5-rehearsal-1788371952368`
  recorded 0 unexpected across 580,061 iterations, against 22 an hour earlier
  in the G103 window. Leader placement is now the leading hypothesis, but the
  class predates 3 Sep and no earlier run recorded where the leader was, so
  the row stays open pending a sighting with both instruments live.
- **Postgres failover returns impermissible `500`s** (G97, **reopened 2 Sep,
  still unexplained**). The 1 Sep fix took the class from 1,566 to 53; the
  2 Sep failover run still recorded 22 unexpected, 21 of them HTTP `500` on
  write paths (`hold_best_available` 15, `order_create` 4, `hold_release` 2,
  **zero on reads**), against 567 permitted `503`s. §6.10 permits `503`, not
  `500`.

  A real defect was found and fixed while chasing this, and it is worth
  keeping on its own terms: `connectionTransient` could not see a transient
  error through a wrapper — `detailOf` hopped exactly one `cause` level and
  the message fallback read only the outermost `error.message`, so a
  doubly-wrapped `57P01` or a once-wrapped `Connection terminated` escaped as
  `500 INTERNAL_ERROR`.

  **What shipped is the narrow half of that fix.** A whole-chain walk closed
  both holes and was reverted the same day: a wrapper with no `code` of its
  own is indistinguishable from a driver error one hop down, so a genuine bug
  wrapped around a transient cause reported transient — a `503 database
  unavailable` for a real defect, which on-call waits out. That failure is
  silent, the one-hop failure is loud, and this row exists because the loud
  one was caught. The message shapes are now matched on the error and its
  immediate cause, code-free frames only, `startsWith` per frame.

  Review then found the same hazard one hop shallower: a `TypeError` wrapping
  a SQLSTATE skipped the code-free outer frame, so every classifier reading
  through `detailOf` — not only `connectionTransient` — took an application
  bug for a driver error. Closed in `detailOf` by class: native bug types
  (`TypeError`, `RangeError`, `SyntaxError`, `ReferenceError`, `EvalError`)
  are never a driver's own error, so they do not speak for their `cause`; a
  plain `Error` wrapper stays transparent. Regression tests in
  `pg-error.test.ts` are teeth-checked in both directions — reverting the
  message fix turns two red, re-introducing the deep walk turns the one-hop
  test red, and removing either native-bug guard turns the narrowing red.

  **It is not established that this was the cause of the 21.** The tempting
  story — reads classify directly, writes re-throw through a wrapper — fits
  the signature exactly, and the codebase does not support it: nothing in
  `@prisma/orm-*`, `pg` or `pg-pool` attaches a `cause`, and the only
  wrapping site in `apps/saff-api/src` is `commerce/provider-token.ts:330`,
  on the payment-token path rather than any of the three named endpoints. So
  the fix removes a latent hole no known production path reaches, and the
  write-vs-read asymmetry remains the sharpest unexplained clue. The row
  stays open on both counts: the cause, and a fresh Patroni failover to
  confirm the count — an operator action, since no automated injection
  exists.
- **ST-3's shed ratio is a lock-contention question, not a tuning one** (G95,
  **inverted 2 Sep**). Per-arm attribution put all 1,183 refusals on the
  `pool` arm with the concurrency arm structurally unreachable. The pool is
  not undersized: the same endpoint holds for 158 ms with zero waiters under
  the soak and 4,070 ms with 32 waiters under ST-3, so long transactions
  drain it rather than arrival rate. **Answered 3 Sep**: a five-point sweep
  with the fixture reset before each point shows the shed ratio tracking
  population monotonically (25.3% at 150 selectors to 90.3% at 3,000) with
  `contended` up 13×, so the shed policy is correct and the *fixture* is the
  finding — 3,000 selectors on a 500-seat section is six buyers per seat,
  against a real hot section of ~3,200 seats. The remaining decision is a
  fixture one, in §14.5, not a code one. See §10.2, which now carries three
  successive verdicts and whose first two this supersedes.
- ~~**The Redis fault row** (G96)~~ — **closed 2 Sep**: 326 joins across a
  Redis outage proven by an independent TCP probe, with zero `503`s of either
  kind on the join endpoint.
- **Selling across a fault window** (§10.4).
- **Four fault rows remain unexercised** (clock skew was closed 4 Sep by falsifier rather than injection) (`SPEC.md:4400-4407`): kill *all*
  API replicas for 60 s, +500 ms on Postgres, the Cloudflare-challenge row,
  payment-provider errors. The count has moved three times and both
  moves are recorded rather than overwritten: an edition said five when the
  kill-50% row was wrongly dropped, then six once it was restored, and it is
  five again because **kill-50% was injected and passed on 3 Sep**. The rows
  named `gate`, a service §4.1 says does not exist; the SPEC table now names
  the API fleet. Of the five, the kill-all row is additionally blocked on
  G102 rather than on operator availability — the cached artefact its
  expectation depends on is not produced by the current cache policy.
- **ST-2's origin-load gate is still red, on its corrected threshold.** **basis: stale-vintage** — the threshold this is red against budgets `ORIGIN_CEILING_RPS` at 3,770; the constant is 4,038 today (`railway.ts:706-709`), and G127 invalidated the measurement behind it.
  `st2-drop.ts:49` budgets `ORIGIN_CEILING_RPS × BURST_SECONDS` = 3,770 × 80
  = **301,600** admissions; the run admitted **~308,000**, a **2.12%**
  breach. §4.5 shrank this alarm by ~150× — from 3.2× a folklore number to
  1.02× a derived one — but it did not clear it, and an earlier version of
  §4.5 said "the alarm does not [stand]", which was this report's most
  misleading sentence. §14.5 forecloses widening the bound
  (`SPEC.md:4390-4395`); the only legitimate route to green is re-measuring
  `PGBOUNCER_MEAN_XACT_MS` with `SHOW STATS_TOTALS`, which needs the private
  network (`pgbouncer.railway.internal` has no public proxy).
- ~~**`saff-api` has no per-request access log** (G98)~~ — **closed 3 Sep**.
  `logSuccess`/`log4xx` had no callers, so the origin could not answer
  questions about its own traffic, and that **blocked G94's prescribed
  remedy** of pulling access logs for the ST-5 window. A single middleware
  now logs every request with method, path, status and duration, verified
  against production by finding the load generator's own requests in the
  origin's logs. G94 is no longer blocked; it is simply no longer
  reproducing.
- **A one-replica service's region is invisible to the IaC plan** (G100,
  found 2 Sep). The SDK deletes a `multiRegionConfig` holding one region
  whose only key is `numReplicas: 1` before diffing, without comparing the
  region name, so placement drift on such a service plans as converged.
  `saff-loadtest` and `saff-redis` both sit in that blind spot — which bears
  directly on this report: §1's "Region asia-southeast1" is not verifiable by
  the tooling for the harness that produced these numbers, and a Redis that
  silently sat outside the region would change what §6.1's outage measured.
- **Every load result between 10:32 and 17:10 UTC on 3 Sep ran with the
  database leader in San Francisco** (G103, found and fixed 3 Sep). The
  Patroni switchover performed for the ST-4 failover row promoted the `sfo`
  standby and nothing moved it back, so writes crossed the Pacific at
  ~178 ms against ~2 ms local — a ~90× write penalty that no gate, plan or
  dashboard surfaced. Two conclusions were affected: ST-4's error count was
  measured under it, and an ST-5 "requests that never complete" finding
  reversed completely once the leader was local (0 unexpected in 580,061
  iterations, hold success 6.9% → 44.1%). The leader was failed back and
  verified by measuring query latency from inside the private network, not
  by reading the role field. **Any run recorded in that window should state
  it**; runs before 3 Sep did not record leader placement at all, which is
  why G94 cannot simply be closed against it.
- **`/api/summary` 404s for every run after a cockpit restart** (G105, found
  3 Sep). The route gates on `runStatus` (`runner.ts:268-272`), which reads
  module-level `current`/`history` — process state — while
  `/api/history/run` reads the database. Measured against the deployed
  cockpit: in-memory history **0**, database **50**, and a real finished run
  id returning `404 No such run.` on one route and `200` on the other. The
  summary file itself survives at `/tmp/<id>.json`; the 404 is returned
  before that read. This was the QA handover's documented verdict route, so
  a tester following it after any deploy would have concluded their run had
  vanished. Handover repointed at `/api/history/run`; the route itself is
  unfixed.
- **Nothing on the buyer path is edge-cacheable** (G102, found 3 Sep). Every
  buyer-path route serves `private, no-cache, no-store`; the only objects
  Cloudflare caches are immutable JS chunks, which never revalidate and so
  can never serve `stale-if-error`. This blocks §14.5's kill-all row on its
  own terms — the row's claim is that clients keep projecting from a cached
  immutable plan while the origin is absent, and there is no such artefact
  at the edge to project from. Confirmed by lifting `WEB_BASIC_AUTH` in
  production: the `401` disappeared and `cf-cache-status: DYNAMIC` did not,
  so the pre-launch lock and the cache behaviour are independent facts. An
  earlier edition of this report conflated them.
- **ST-5's `saff_hold_success` gate cannot mean what it says** (G104, found
  3 Sep). The metric reads 44.1% against `rate>0.5` with the leader local
  (`st5-rehearsal-1788371952368`), and it is the scenario's standing red now
  that the `status=0` class has stopped reproducing — but the number counts
  a **lost race as a failed hold**. `classify` returns `false` for
  `503 SHEDDING`, for `409 contended` and for `409 exhausted` alike
  (`funnel.ts:148-223`), and `acquireBestAvailable` feeds that straight into
  the rate (`:339`). Seats are exclusive, so under contention someone must
  lose; the metric therefore falls as contention rises. Its docblock says it
  exists to catch "admission letting through more buyers than there is
  inventory" — and the run's own `exhausted` counter is **0** across 580,061
  iterations, so not one request met an empty fixture and that specific
  failure did not occur. Whether 44.1% is reasonable for this fixture is
  still unknown; the metric cannot answer it either way. Fixing it changes a
  gate every prior ST-5 run was measured against, and `st1-baseline.ts:77`
  sets the same metric at `rate>0.95` on an uncontended fixture, so the
  change needs a comparability decision rather than an edit.
- **VR-17 is blocked, and G96's closure must not be read as satisfying it** **basis: operator-blocked** — VR-17 moves with the dormant-queue product decision (G22), not with an engineering change.
  (`SPEC.md:5115`). VR-17 asks whether queue positions restore from the
  Postgres batch and `QueueCheckpoint` *after* Redis returns. The 2 Sep run
  proved the tier **fails open** — joins continuing *without* Redis — which
  is a different property. The drill needs a ranking pass, and the queue is
  dormant by product decision (G22), so it moves with that decision.

---

## 11. Verdict

**ST-4's §14.5 acceptance criterion is still not met, but it is closer.**
`SPEC.md:4361` requires "Every §6.10 row exercised; recovery within stated
RTO". **Four of nine rows are exercised** — Redis failover, Postgres
failover (re-run 3 Sep under a real Patroni switchover), the 6-hour soak,
and the kill-50% row injected 3 Sep — leaving five: kill *all* API replicas
for 60 s, +500 ms on Postgres, a Cloudflare challenge outage,
and payment-provider errors.

*(An earlier edition of this paragraph said "five of nine … leaving four"
while listing four exercised rows. The arithmetic did not close; four and
five is the correct split.)*

Four of the five need an action against a third party this repo cannot
drive. The fifth — kill-all — is blocked differently, and on evidence
gathered 3 Sep rather than on operator availability.

**The kill-all row's blocker was misdiagnosed, and lifting basic auth
proved it.** This section previously said the row was unprovable "because
the public surface answers `401` with `cf-cache-status: DYNAMIC`", treating
those as one fact. They are two. `WEB_BASIC_AUTH` was lifted on 3 Sep
(captured first, restored byte-identical, `401` re-verified): the `401`
went away and **`DYNAMIC` did not**. Every buyer-path route serves
`private, no-cache, no-store`; the only edge-cached objects are immutable JS
chunks, which never revalidate. So the row is blocked on the buyer path
being uncacheable (**G102**), not on the pre-launch lock — and §14.5's claim
that clients "keep projecting from the immutable plan" has no cached
artefact at the edge to project from.

**Both kill rows named a service that does not exist. Corrected 3 Sep.**
§14.5's fault table said "kill 50% of `gate`" and "kill all `gate` for
60 s", while §4.1 states there is no separate `gate` deployable — D21 merged
it into the single API service. The rows were executed against `saff-api`,
which is unambiguously what they mean, and the table now names the API
fleet.

**Every load result between 10:32 and 17:10 UTC on 3 Sep ran against a
database leader in San Francisco.** A Patroni switchover from the failover
test left the leader on the `sfo` standby and nothing moved it back, so
every write crossed the Pacific at ~178 ms instead of ~2 ms (**G103**).
Results recorded in that window are marked below where they are affected.
The leader was failed back and verified by query latency, not by the role
field alone.

**What passes:** ST-1; **ST-2** (re-run 2 Sep, `st2-drop-1788344380903`, on
an unchanged `ORIGIN_CEILING_RPS`); VR-9's 6-hour soak (777,529 requests,
zero failures, RSS −2.15 MiB/h against an 8 MiB/h limit); the Redis fault
row; the **kill-50% fault row** (3 Sep — join p99 and the admission ceiling
both passed, so the row's own expectation holds; what degraded was baseline
catalog p95, which §8.7 puts on class 3 as the intended last casualty); the
oversell invariants on seeded sales.

**One red is now known to be correct behaviour rather than a defect.** ST-3
sheds 90% at `SCALE=1` and that is the shedder working against a fixture
asking for six buyers per seat — see the table below. It stays red because
the gate is red, and the gate is red because §14.5's fixture asks for a
state the system is right to refuse. Fixing that is a fixture decision, not
a code one.

**Scenario by scenario, as this section stood on 3 Sep:**

**basis: stale-vintage** — superseded by the header board. Every verdict below predates G127, which measured that G115 roughly doubled resting-shape join capacity and invalidated the constants these were judged against. ST-2's `green (2 Sep)` is the clearest case: it cites `ORIGIN_CEILING_RPS` at 3,770, and the constant is 4,038 today (`railway.ts:706-709`), so the run that produced the green was judged against a ceiling that no longer exists.

| Scenario | Verdict | The finding underneath |
|---|---|---|
| ST-1 | green | — |
| ST-2 | **green (2 Sep)** | Was 2.12% over a *derived* ceiling. Closed by re-running against production, not by widening the bound: `st2-drop-1788344380903` reports `count<301600` PASS with `ORIGIN_CEILING_RPS` unchanged at 3,770. |
| ST-3 | red, **and the red is correct behaviour** | The shed policy is sound and the *fixture* is the finding — answered 3 Sep by a five-point scale sweep with the fixture reset before each point (G95). Shed ratio tracks population monotonically, 25.3% at 150 selectors to 90.3% at 3,000, and `contended` rises 13× across a 20× population range. So class 1 is not over-eager and the budget measures the right quantity. 3,000 selectors against a 500-seat section is six buyers per seat; refusing 90% is the shedder working. A real hot section at this venue holds ~3,200 seats, so §14.5's fixture is ~20× more contended per seat than the venue can produce. |
| ST-4 | red | 70 impermissible errors under a real switchover (`st4-failure-1788345019384`). The **wrapper hypothesis is refuted**: the population is 60 pool-checkout timeouts, 14 `server_login_retry`, 1 deadlock — all classified correctly by the shipped code, no wrapped driver error present. **Measured inside the G103 window, so the count is not clean**: that run's own switchover is what left the leader in `sfo`. Attribution beyond the population was blocked by a logging defect (every `onError` branch logged identically), fixed 3 Sep — the next failover answers this from its own logs. |
| ST-5 | red, **but not for the reason recorded until 3 Sep** | The `status=0` class is **not reproducible** on a correctly-placed leader: `st5-rehearsal-1788371952368` gave **0 unexpected across 580,061 iterations**, against 22 in the G103-window run an hour earlier, with hold success 6.9% → 44.1% and `contended` up 30×. Not closed — the failure predates today and no prior run recorded leader placement — but placement is now the leading hypothesis for every sighting (G94). G98 no longer blocks it; the access log was wired 3 Sep. **The standing red is `saff_hold_success` at 44.1% against >50%** with the leader local — but that gate counts a lost race as a failed hold and cannot distinguish broken admission from buyers competing (G104); `exhausted` is 0 across the run, which rules out the failure its docblock names. |

**Nothing here is softened to make the document read better.** Where a
finding was superseded it says so and cites what superseded it; where a
finding is still true it stays, including the three open items this report
omitted until 2 Sep. A load-test report that hides a red is worth less than
no report, because the next person acts on it during a drop.

## 12. 3 September 2026 — one defect, three reds; and what actually scales

*This section supersedes §11's ST-3, ST-4 and ST-5 rows where they conflict.
They are kept above because the correction is the evidence.*

### 12.1 One defect behind three separate reds

ST-3's standing `saff_unexpected_total` red, the 70 unclassified failures
recorded against a higher fleet shape (G116), and 16 failures across four
sustained ST-3 runs are **the same defect**, not three.

The evidence is a tag signature read against a discriminator registered
*before* the run:

| tag | count | share |
| --- | --- | --- |
| `status:0` — no response line arrived | 14 | **88%** |
| `errorCode:1500` + `status:500` — origin answered a 5xx | 2 | 12% |
| transport codes (12xx, 1050) | **0** | — |

Registered beforehand: G115-class presents as `status:0` with no
`errorCode`, clustered at the client timeout; a per-replica budget problem
presents as fast failures carrying transport codes. The result is
unambiguous — **no transport failure is involved at all.**

**The defect (G115): the server's hold ceiling exceeds the client's
patience.**

| side | budget | source |
| --- | --- | --- |
| client | **30 s**, one attempt, no retry | `WRITE_ONCE`, `apps/saff-web/src/lib/api.ts:500` |
| server | **~50 s** worst case | 10 pool checkouts × 5 s `connectionTimeoutMillis` |

§7.7's search runs two rounds of up to five candidates
(`best-available.ts:48`), so one request can reach the pool ten times. When
the browser abandons at 30 s the server may still be granting — and because
`WRITE_ONCE` is single-attempt *precisely because a hold must not be
repeated*, the buyer's manual retry can become a second acquisition. At the
endgame those orphaned holds remove supply exactly when supply is scarcest,
so the failure amplifies the contention that produced it.

**Observed, not predicted:** admitted-hold max **34,313 ms** against a 30 s
client budget, in production.

**Fixed 3 Sep, after this analysis.** `SET LOCAL lock_timeout`/
`statement_timeout` inside the claim transaction (`hold.ts`) bounds what was
unbounded once a connection was held; a between-attempts deadline in
`acquireBestAvailable` (`HOLD_DEADLINE_MS`, 14,000 ms default) now answers
`409 reason:"deadline"` between complete attempts, so expiry can only ever
mean "nothing was granted." See §12.6 for what the fix proves and what it
does not yet prove.

### 12.2 What scales, what does not, and which is the drop-day mechanism

**The autoscaler cannot react inside a burst, and the arithmetic is exact.**
`AUTOSCALE_TICK_MS` is 60,000 (`scripts/worker.ts:368-372`) and
`AUTOSCALE_UP_TICKS` is 2 (`autoscale.ts:212`), so a scale-up needs **≥120 s
of consecutive over-threshold samples** before it writes anything, plus
container start before the replica serves. ST-2's burst is ~200 s. Measured
3 Sep: the fleet held **2 replicas for the entire run** while pool waiters
peaked at 29, and reached 3 only *after* the load stopped.

**This is the cost of damping, not an oversight.** The waiters arm already
reacts on the same `upTicks` evidence as CPU/mem; the streak requirement
exists to stop flapping against noisy 60 s samples, and G107 records a
rolling deploy inflating the live count. A twitchier loop would trade a
problem that scheduling solves exactly for one that has already bitten.

**So the division of labour is:**

| mechanism | owns | evidence |
| --- | --- | --- |
| **The shedder** | the **burst** — a drop | 40.1% shed, 0 unexpected, flat per-buyer contention, three reproductions |
| **The autoscaler** | the **sustained tail** | reaches 3 on a waiters-dominant shape, higher on throughput-dominant |
| **The drop-window floor** | **drop-day capacity** | §4.3's clock is known, so capacity can precede the burst — derived from `SalePhase.dropAt` every tick since 4 Sep; `AUTOSCALE_MIN` remains the manual override and composes by `max` |

**Corrected 3 Sep 2026: for weeks this row had no working mechanism behind
it.** The original text read "the pre-drop raise must therefore go through
`AUTOSCALE_MIN`, which the controller reads and respects". It read the
value and never acted on it: `config.min` appeared only as a descent guard,
so a floor raise on a quiet fleet was inert. Measured with `min: 7` parsed
live inside the worker, the fleet held **2 replicas for 19 minutes**
(G121).

A replica floor also cannot be set by an operator write — `saff-api`'s count
is owned by the live autoscaler and a direct `serviceInstanceUpdate` decays
within ~15 minutes, measured with no load: 2 → 3 → 2 (G117). So **both**
routes to pre-positioning were non-functional, and both were believed
working when this table was written.

`AUTOSCALE_MIN` is now enforced rather than merely respected on the way
down, so the row became true — and as of 4 Sep nobody has to remember to set
it. `dropWindowFloor` derives a floor of 8 from any `SalePhase.dropAt` inside
`[dropAt − 10 min, dropAt + 10 min)`, recomputed every tick and persisted
nowhere, so the ten-minute lead covers a missed tick, a failed metrics read,
a failed scale call, container start, and cache warm-up in sequence.

**Deriving rather than scheduling is what makes it safe.** A scheduled design
has a raise event and a lower event, and missing the lower one strands the
fleet — which is not hypothetical: §12.5 records a
`railway variables --remove AUTOSCALE_MIN` that silently no-opped and held an
8-replica floor for about an hour. A derived floor has no lower event to
miss. `AUTOSCALE_MIN` survives untouched as the operator escape hatch, since
the two floors compose by `max`.

**The waiters arm caps at base + 2 by design.** Four sustained
`st3-contention` runs reached 3 from a base of 2 and went no higher. Waiters
can reflect the *server-side* pool ceiling, which more core replicas cannot
raise — past a point extra replicas are extra contenders for the same
pgbouncer slots. The CPU/mem arm has no such cap, so a throughput-dominant
shape can still take the fleet to its maximum.

### 12.3 The scaling ceiling is the connection budget, not the plan

"Scale to the maximum the plan allows" is the wrong ceiling to reason about,
because the plan limit is not the operative constraint at any link in the
chain. Derived from the IaC's own constants:

| link | value | binds? |
| --- | --- | --- |
| `AUTOSCALE_MAX_REPLICAS` | 12 | the declared ceiling |
| client connections at worst case | 12 × 4 × 41 + 2 × 12 = **1,992** | **binds** — against `max_client_conn` 2,000 |
| pgbouncer → server | 105 × 3 = 315 | against `max_connections` 500 |
| Railway Pro replica limit | **42** per service | never binds — 12 is 29% of it |

**Corrected 3 Sep 2026 — there is a lower ceiling this table missed, and it
is not in the table's units.** Under real concurrency the binding constraint
is **Postgres's `/dev/shm` arena**: 62 MB, platform-fixed, undeclarable, and
exhausted by concurrent parallel-query workers. It produced HTTP 500s
starting at **2 replicas** — so it is keyed to concurrent parallel plans, not
to any replica count, and capping `AUTOSCALE_MAX` would not have avoided it.
Fixed by `max_parallel_workers_per_gather = 0` through Patroni's DCS; see
G118. The chain below is still correct about *connections*; it was simply not
the whole chain.

**12 is the client-budget ceiling.** `WEB_CONCURRENCY_MAX` is *derived* from
the same inequality rather than hand-picked (`railway.ts:730-734`), and a
guard throws if `DB_POOL_MAX` stops fitting the autoscaler-aware worst case,
so the arithmetic cannot silently drift. Raising 12 without re-deriving the
pool budget would reproduce **G28** — a connection budget that does not fit
`max_connections`.

**The plan limits, sourced 4 Sep rather than assumed.** Pro allows **42
replicas per service** and **24 vCPU / 24 GB per container**
(`docs.railway.com/pricing/plans`, `deployments/scaling`). The headline
"1,000 vCPU / 1 TB" is that per-container ceiling times 42 replicas — Railway
states the maxima "include replica multiplication" — not a container size, so
it is one limit written twice. Railway's CLI and autoscaling guide say 50
rather than 42 with no plan qualifier and nothing reconciles the two; 42 is
the conservative read. This fleet declares 12.

**And raising it buys nothing on the axis this table measures — but that is a
claim about connections, not about capacity.** `railway.ts:639-645`:
`WEB_CONCURRENCY_MAX` solves as
`(max_client_conn − worker) / (AUTOSCALE_MAX × per-core)`, so at a fixed
`max_client_conn` **doubling the replica ceiling halves the per-replica
process count** — 12 → 24 re-solves 4 → 2, and the fleet's process count
stays at 48. It must: the client budget is fixed, each process costs 41, so
no ceiling can put more than `floor(1976 / 41) = 48` processes on the fleet.
The ceiling does not set the fleet's size; it sets its **granularity**.

**The correction that matters, because the sentence above invites the wrong
conclusion.** Those 48 processes are a *connection* budget. Spread across 24
replicas rather than 12 they occupy twice the containers, so per-process CPU
and memory scale with the ceiling — and CPU is the axis `autoscale.ts`
actually triggers on (`scaleUpUtilization`, sampled as fractional vCPU
against `CPU_LIMIT`). So a raise buys **zero connection capacity and linear
compute**, and which one binds is a measured question rather than a derived
one. Post-G115, with pool waiters down from 15–32 to 4–5, connections were
demonstrably *not* binding at the shape ST-2 ran: 4,223 rps at two replicas.
Reading the flat 48 as "raising the ceiling buys nothing" is a join error
between *processes* and *capacity*, and it was made — and corrected — in
review before it reached this page.

**`bun run capacity` prints the whole surface**, so neither axis has to be
re-derived from prose. `railway/saff/scripts/capacity-envelope.ts` tabulates
every interesting ceiling against fleet processes, per-container compute, and
the client budget stranded by flooring — at 42 the fleet uses 1,722 of 1,976
usable slots and strands 254, a waste invisible in both other columns. The
table recomputes from the constants the guards use, so unlike this paragraph
it cannot go stale.

**So the answer to "raise it toward the plan maximum" is: nothing in the
connection budget improves, the compute axis is unmeasured, and no measured
demand justifies it.** The throughput lever is `PGBOUNCER_MAX_CLIENT_CONN`,
and moving it means re-measuring ST-2 — `ORIGIN_CEILING_RPS` is a measurement
of the deployed shape, not a constant.

### 12.4 Scale-to-zero is implemented everywhere it is coherent

Measured live: `saff-web`, `saff-admin` and `saff-loadtest` are `SLEEPING`.
Two services deliberately do not sleep, and neither is an oversight:

- **`saff-worker`** runs the settlement sweep and the hold reaper. Sleeping
  it means paid orders do not settle and expired holds are not released —
  G71 and G73 are the incident class.
- **`saff-api`** is structurally unable to: `numReplicas: 0` is rejected
  server-side (G55), and its metrics push resets the sleep timer, so
  sleeping it would require removing telemetry. The reason is recorded where
  the flag is deliberately *not* declared — `railway.ts`'s `saff-api`
  service block, in the comment beginning "an HA floor of 2 that can sleep
  to zero is not a floor": the 60 s push in `metrics-push.ts` is outbound
  traffic that resets Railway's 10-minute inactivity timer, so a
  telemetry-wired replica would rarely have slept at all. Anchored to that
  sentence rather than a line number, which had already drifted to an
  unrelated `TICKET_SIGNING_*` spread by 5 Sep 2026.
  Beyond that, zero means the first buyer of a drop pays a cold start on the
  hot path.

**"Scale to zero" is not a thing Railway offers anywhere, confirmed from the
vendor 4 Sep.** Railway's published config schema (served at
`backboard.railway.app/railway.schema.json`, not vendored here) bounds
`numReplicas` at `minimum: 1`, and the docs describe no zero-replica state —
so G55's server-side rejection is the platform's documented contract, not a
quirk of one call. What Railway markets as *Serverless* is a
**sleeping single replica**, not a zero-replica state, and its trigger is
**outbound** packets: a service that emits anything on a timer never sleeps.
That independently confirms the metrics-push reasoning above — declaring
`sleepApplication` on `saff-api` would read as enabled in the config and
never once fire.

One trap worth recording for whoever next toggles it: the setting is applied
**at container creation**. `serviceInstanceUpdate` writes it and reads it
back as set while the running container keeps its old value until the next
deployment — G117's "platform phantom write" class exactly, which already
names `sleepApplication` on `saff-worker` as an instance. G117's rule holds
here: reading the value back is not sufficient proof that it took.

#### Why "the whole infra scales" has three correct answers, not one

The objective asks for the whole fleet to scale up automatically and down to
zero. That decomposes into three different right answers, and collapsing them
into one controller-per-service would be worse than what is deployed:

| service | replicas | elasticity | why this and not more |
| --- | --- | --- | --- |
| `saff-api` | 2 → 12 | **autoscaled** on CPU/memory | the only service whose load is its own; the controller lives on `saff-worker` so it is never reasoning about the replica it is about to remove |
| `saff-web` | 2 | **sleeps when idle** | its load is *proxied* API load, so a controller here would scale the mirror and not the thing; scaling web without api moves nothing |
| `saff-admin` | 1 | **sleeps when idle** | operator-only, behind basic auth; no buyer to protect with an HA floor |
| `saff-loadtest` | 1 | **sleeps when idle** | runs only when an operator starts a run |
| `saff-worker` | 2 | **static by design** | a leader-gated availability pair — `runtime/leader.ts`'s session-scoped advisory lock means the second replica is a standby, not throughput. Adding replicas adds split-brain surface, not capacity |
| `saff-pgbouncer` | 3 | **static by design** | its size is an *input* to the connection budget (`105 × 3`), not a response to load. Scaling it changes the arithmetic every other guard is derived from |

So the fleet already is as elastic as it can coherently be: **one service
scales on load, three sleep to the platform's minimum, two are fixed because
varying them would break an invariant rather than serve one.** The remaining
honest gap is not a missing controller — it is that `saff-web`'s CPU under a
drop has never been measured, so the claim "web is not the bottleneck" rests
on the topology argument above rather than on a reading. That is a
measurement to take before a controller is written, not after.

**And "down to zero" is already at its floor.** Three services sleep; the
other three each have a stated reason not to, above. Zero replicas is not
reachable on this platform at all — `numReplicas: 0` is rejected server-side,
and once silently failed every apply in this project (G55).

#### "Why is `saff-web` still 1 replica and in the USA?"

Both halves of the premise are false today, and the reason each was true is
worth more than the correction.

**Replicas: it is 2, not 1** (`railway.ts:320`, `REPLICAS.web`). It was
raised precisely *because* of the second half of the question.

**Region: it is in `asia-southeast1-eqsg3a`, with 12/12 services verified
there** by `check:placement` on every deploy. But it genuinely *was* in
`us-east4-eqdc4a` on 3 Sep while this file declared Singapore, and the
interesting part is why nothing caught it: the Railway SDK **deletes a
single-region `multiRegionConfig` before diffing**, so for any service at one
replica the declared region is inert and drift is invisible to the plan
(G100). The service was not misconfigured in a way the IaC could see — the
IaC could not see it at all.

**So the fix was not a region change.** `REPLICAS.web` went to 2, which makes
the declaration enforceable, and `check:placement` now reads *live* placement
on every deploy rather than trusting a converged plan. The gate is the fix;
the replica count is what makes the gate's subject visible.

**Two services remain inside that blind spot, honestly.** `saff-admin` and
`saff-loadtest` are still at one replica, so their region is declared inertly
— both are in `REGION` today by where they were first created rather than
because the file compels it (`check-placement.ts:26-32`). Raising their
floors would cost a replica each to defend a region neither serves buyers
from, so the live check covers them instead. That is a deliberate trade
recorded rather than an oversight.

### 12.5 The 2x question, pre-registered before either arm ran

The objective's own words: *"we shouldn't sweat to handle even 2x load."*
Two arms, written down before either was run, so neither result can be read
to fit a conclusion.

**Offered load: 2 × T0 = 10,000 joins/s.** ST-2's specified drop burst is
5,000 joins/s over an 80 s window (10 s ramp, 60 s hold, 10 s down) —
`QA_LOADTEST_HANDOVER.md`'s scenario table, measured at ~4,900. Doubling it
is `SCALE=2`.

**Driven out of band, deliberately.** The cockpit's `POST /api/run` pins
`scale` to `(0, 1]` (`app.ts:256-261`), and that bound is a safety rail on a
route whose only other protection is one credential — widening it to run an
experiment would weaken a shipped control for every future operator. k6
takes `SCALE` from its environment with no such cap, so the 2x arms invoke
it directly and the rail stays exactly as it is.

| | cold arm | pre-positioned arm |
| --- | --- | --- |
| floor | resting 2 | raised via `AUTOSCALE_MIN` |
| models | **drop day as deployed** | drop day with the runbook followed |
| claim under test | degrades cleanly and recovers | admits substantially more of the same load |
| pass bar | **zero unexpected**; admission is *reported*, not gated | **zero unexpected**, with admission materially higher | **basis: measured** — `st2-drop-1788441064400` and `st2-drop-1788442576659` |

**The delta between the arms is the runbook line's measured value** — the
number the answer to "should we pre-position?" actually rests on. So both
arms must be identical in scenario, scale and fixture state, with the
replica series recorded throughout.

**One run per arm.** If an arm produces unexpected failures, the tags name
them and that is the arm's verdict — no retry-until-clean. A run is re-run
only when a precondition failed (leader out of region, wrong fixture,
mid-run deploy), and the invalidation is recorded.

#### Result: 2x is not a sweat, and pre-positioning is worth 2.84×

Both arms ran at `scale: 2` on `st2-drop`, identical scenario, duration and
fixture. Only the floor differed.

| | cold arm | pre-positioned arm |
| --- | --- | --- |
| run | `st2-drop-1788441064400` | `st2-drop-1788442576659` |
| floor | resting 2 | `AUTOSCALE_MIN=8`, warmed 5 min |
| replicas traversed | **2 → 2** (never scaled) | **8 → 10** |
| iterations | 717,772 | 717,302 |
| **admitted** | **208,638** | **592,045** |
| shed | 526,907 | **143,029** |
| contended | 187,084 | 562,051 |
| **unexpected** | **0** | **0** |
| verdict | **pass** | red on a stale threshold — see below | **basis: stale-vintage** — red against a threshold G127 invalidated |

> **Cold-arm admitted corrected 6 Sep: 208,638, not 206,486.** Every figure
> in this subsection was re-read from the durable run store
> (`loadtest.run.summary`) rather than from the cockpit view it was first
> transcribed from. The derived figures move with it — the ratio is **2.84×**
> (was 2.87×) and per-slot admission is **20,864** (was 20,649). The pool-slot
> finding *strengthens*: agreement tightens to **1.01** while replica-linear
> stays at 0.63.
>
> **The header's "admitted 562,051" was defensible, not wrong.** 562,051 is
> `saff_admitted_total{endpoint:prequeue_join}` — the metric ST-2's threshold
> gates and the one `admission-reading.ts:156` reads — while 592,045 is the
> bare `saff_admitted_total`. On this scenario the join-tagged count also
> equals `saff_contended_total` exactly, which is why the table below can
> label 562,051 "contended" without contradicting the header. The hazard is
> real but it is ambiguity, not error: **a bare "admitted" figure here is
> ambiguous between three readings**, so each should name its metric. The
> ratio above compares bare totals to bare totals.

**The headline: zero unexpected failures in both arms at double demand.**
Two replicas absorbed 10,000 joins/s without a single error, shedding the
excess — which is the shedder's whole job and the answer to the objective's
question. The system does not sweat 2×; it sheds 2×.

**The pre-positioned arm's red is the stale-ceiling class, not a defect.**
Its only failing threshold is
`saff_admitted_total{endpoint:prequeue_join} count<323,040` against
**592,045 admitted** — a bound derived under a fleet shape that no longer
applies, failing *because capacity exceeded it*. Same shape as G110, and it
cannot hide a regression: it can only fire when the system does better than
the bound assumed.

**The measured value of the runbook's pre-drop raise: 208,638 → 592,045
admitted on identical offered load, a 2.84× increase, with shedding down
73%.** That is the number the "should we pre-position?" decision rests on,
and it was unmeasurable before tonight — the mechanism did not work (G121).

**Two secondary findings from the pair.** The cold arm **never scaled at
all** (flat 2 across the whole window), so at 2× demand the shedder absorbs
the burst faster than a 60 s-sampled controller can react — which is exactly
why pre-positioning is the drop-day mechanism and the autoscaler owns the
sustained tail. And the pre-positioned fleet grew **8 → 10**, confirming a
raised floor does not pin the controller: it still scales above it.

**Bracketing.** The `scale: 2` ceiling was widened by
`LOADTEST_SCALE_CEILING` for the window only and reverted after, verified
behaviourally in both directions (`2.5` rejected at `(0, 2]` before;
`1.5` rejected at `(0, 1]` after).

**`AUTOSCALE_MIN` was *not* deleted, and an earlier version of this
paragraph said it was.** The `railway variables --remove AUTOSCALE_MIN`
form returns success and changes nothing — a silent no-op. The floor
therefore stayed at `8` until it was noticed hours later: setting up a
pinned ST-5 at 14:39, the process still read `{"min":8,"max":12}` with
nine replicas live, where a real deletion would have read `min: 2`. The
working form is **`railway variable delete <KEY>`** — singular `variable`,
no `-y` — and a removal is only confirmed by reading the value back out of
the running process, not from the CLI's exit status. Released and verified
that way at 15:39 (`raw_min: null`, config resolving to the declared
`{min: 2, max: 12}`, fleet observed descending 7 → 6).

**No published number is affected**, because the only run inside that
window is the stray `st2-drop-1788443692056` below, which already carries
no claim. The cost was ~1 h of an 8-9 replica floor nobody wanted.

One stray run exists — `st2-drop-1788443692056`, started by a
revert-verification probe that was accepted before the ceiling change
reached the process; it is not an arm and carries no claim.

#### What ST-2's admitted gate should be keyed to

The pair also settled what ST-2's admitted gate should be keyed to. A bare
count is meaningless without a shape (G110, G114, G119), but "shape" needed
a unit. Testing the two **clean** points against pool slots:

| shape | replicas | `pools_registered` | admitted | admitted / pool |
| --- | --- | --- | --- | --- |
| cold | 2 | 10 | 208,638 | **20,864** |
| pre-positioned | 8-10 | 28 | 592,045 | **21,144** |

    admitted ratio   2.84x
    pools ratio      2.80x   -> agreement 1.01
    replica ratio    4.50x   -> agreement 0.63

**Admitted throughput is proportional to pool slots, not to replica count**
— 1.01 against slots versus 0.63 against replicas, and the ratio is
divisor-independent because the divisor cancels. That is the same
relationship the connection budget is built on: a replica contributes
capacity only through the pool slots it opens, which is why
`WEB_CONCURRENCY_MAX` re-solves downward as the fleet grows and why the
ceiling chain runs through PgBouncer rather than through replica count.

**The divisor, stated because an earlier draft used the wrong one.** The
gate's basis is `admitted ÷ 80 s` — the burst is `10 s` ramp + `60 s` hold +
`10 s` down (`st2-drop.ts:72-79`), and `BURST_ADMISSION_BUDGET` is
`ORIGIN_CEILING_RPS × 80`. `st2-drop.ts:44-48` exists to warn against
exactly the mistake I made: dividing by whole-run wall clock dilutes the
burst by the idle remainder. On three bases:

| arm | total ÷ 80 (**the gate's**) | burst-only Δ ÷ 80 | total ÷ 150 (**wrong**) |
| --- | --- | --- | --- |
| cold | 2,581 | 2,124 | 1,377 |
| pre-positioned | **7,401** | 6,550 | 3,947 |

A first draft of this section reported 3,910 rps and concluded the origin
"ran at its derived knee" (1.04×). That was the 150 s figure, and the
conclusion was an artifact of it.

**On the gate's own basis the pre-positioned arm admitted 1.96× the derived
ceiling** — 592,045 against a 301,600 budget. So `ORIGIN_CEILING_RPS =
3,770` is not an origin property: it is a **measurement at one fleet shape**
(210 pooler server connections ÷ 0.0557 s, taken 1 Sep at the resting
floor). More pool slots means more concurrent server connections, so the
ceiling moves with the fleet — which is precisely why keying the gate to a
bare count is wrong.

**What that does *not* license, corrected 4 Sep.** An earlier revision of
this section ended "and why `pools_registered` is the right key", then
asserted in the present tense that "the threshold **is** a `(shape, band)`
pair keyed to `pools_registered`". Neither was true, and the second was a
claim about code that does not exist: `st2-drop.ts` gated a bare
`count<BURST_ADMISSION_BUDGET` throughout.

The proportionality above is real and survives. What it cannot support is a
*divisor*, for two reasons this section had already written down without
following:

1. **The ratio cancels the divisor** — stated four paragraphs up, "the ratio
   is divisor-independent because the divisor cancels". Agreement at 1.02
   proves admitted throughput moves *with* slots. It establishes no absolute
   slots-per-run figure, and a gate needs the absolute.
2. **`pools_registered` counts pools, not slots.** `metrics.ts:155-158`
   exports `registeredPoolCount()`; slots are `pools × DB_POOL_MAX`. This
   section's own prose says "slots" while its table divides by a pool count.
   Harmless in a ratio, wrong in a gate.

And the divisor is not obtainable anyway: `runtime-sampler.ts:228-288` makes
one `fetch` to `apiOrigin` per tick, which the load balancer routes to **one
arbitrary replica**, so `poolsRegistered` is a single process's count and
never a fleet sum. The 10 and 28 in the table above have no reproducible
derivation from anything the harness records.

**So the re-key is OPEN, not done** (G125). What landed instead is the half
that is buildable without a divisor: an admission **floor**, described below.

**The locked run is deliberately excluded from the derivation.**
`st2-drop-1788423553902` admitted 176,815 at a descending, dwell-locked
2-3 replicas. That number measured a **defect** (G120), not a capacity, and
including it would encode the lockout into the gate. It stays as G120's
exhibit and nothing else.

#### The admission floor, which is the half that was buildable

The gate had a structural blind spot independent of the divisor question,
though it is narrower than a first draft of this section claimed. An *upper*
bound is satisfied by admitting **nothing** — but a **dead** origin was
already caught: connection-refused and timeouts carry `status 0`, reach
`unexpected.add()` (`funnel.ts:390`), and `saff_unexpected_total` is gated
`count==0`. What was **not** caught is an origin that answers correctly and
admits almost nothing: `503 SHEDDING` routes to `saff_shed_total`, which no
threshold gates, by design. So a run that shed every single join passed ST-2
with every threshold green, and `dropped_iterations{scenario:burst}` did not
help — it proves the **generator** kept up, not that the **origin** admitted.

`st2-drop.ts` now also asserts `count>BURST_ADMISSION_FLOOR`, half of
`BURST_ADMISSION_BUDGET`. Three properties make it honest:

- **Keyed to the same measurement as the ceiling.** `ORIGIN_CEILING_RPS` *is*
  the measured `SCALE=1` admitted rate — `config.ts`'s re-measure instruction
  is "divide `saff_admitted_total` by the 80 s burst window", which is that
  quantity. A fraction of it moves when the measurement moves, instead of
  becoming a second stale number.
- **Not keyed to the 2x arm's 208,638.** That is a `SCALE=2` observation, and
  admission *drops* under overload (the cold arm ran at 0.69× the ceiling
  because shed work still costs slots). Encoding it into a `SCALE=1` gate
  would be G120's mistake in a subtler dress.
- **`SCALE`-guarded.** The ceiling passes vacuously at reduced `SCALE` by
  direction; a floor does not — at `SCALE=0.1` a healthy origin admits ~10%
  and would red. The floor therefore exists only at `SCALE=1`, mirroring the
  ceiling's own "only `SCALE=1` exercises it" doctrine. The 2x arms run at
  `SCALE=2` and get the vacuous form deliberately: they are experiments
  measuring behaviour past the knee, not gates asserting it. **basis: gate-green** — the floor in `st2-drop.ts` holds at reduced `SCALE` by direction, unlike the ceiling it guards.

**Verified by execution, not arithmetic.** `k6 archive` compiles
`["count<323040","count>161520"]` at `SCALE=1` and
`["count<323040","count>=0"]` at both `0.1` and `2`. Driven against a
synthetic counter: 200,000 admitted → green; 100,000 admitted → **red**; the
same 100,000 with the floor removed → green, which is the naive baseline
proving the assertion has teeth.

**What it deliberately does not catch, stated because a first draft claimed
otherwise.** The one recorded G120 scale lockout
(`st2-drop-1788423553902`) admitted **176,815** — `0.547×` the budget, so it
sits *above* this floor and would run green. Tightening to ≥0.55 to cover it
would put the bound inside the run-to-run spread: published `SCALE=1`
admitted counts are ~308,000 and 349,007, a **13%** range, so 0.55 would be
about one ordinary run away from a false red. That trade buys one detection
and spends the operator's trust in every future red, which is how a gate
becomes noise.

So this is a **coarse liveness bound**, not a lockout or drift detector. The
detector lockouts actually need is a replica-count series in the sampler —
recorded as G125's option 2, and not built here.

### 12.6 What is genuinely open

| item | state |
| --- | --- |
| **G115** — hold ceiling exceeds client budget | **fixed and measured 3 Sep.** Two halves landed: (1) `SET LOCAL lock_timeout`/`statement_timeout` inside the claim transaction bounds a query that previously ran unlimited once a connection was held; (2) a between-attempts deadline in `acquireBestAvailable` (`HOLD_DEADLINE_MS = 14,000`) throws `HoldDeadlineError` → `409 reason:"deadline"` between complete attempts, so expiry can only mean "nothing was granted." The falsifier is RED against both the no-deadline baseline and a `Promise.race` whole-call-timeout baseline (the latter still committed a hold after telling the buyer no). **Measured against production**, `15addd85` SUCCESS at 18:17 UTC, two ST-3 runs after it: `saff_unexpected_total` **13 → 2 → 0**, and the `status:0` class — this defect seen from the client end — is **0 in both**, against a pre-fix run that carried it. Hold latency max **60,004 → 20,025 ms**, from double the client's 30 s budget to inside the deadline plus one attempt's overshoot, which is what the design predicts. Second run verdict **pass**. See §12.8 | **basis: gate-green** — `check:gaps` records G115 closed; the two halves are in `hold.ts` and the claim transaction |
| **ST-2's admitted ceiling** | **half closed 4 Sep; the re-key is OPEN (G125).** What is settled: admitted throughput scales with **pool slots**, not replicas — 208,638/10 = **20,864** and 592,045/28 = **21,144**, agreeing to **1.01** where replica-linear disagrees at 0.63; and `ORIGIN_CEILING_RPS` is a *fleet-shape* measurement, not an origin property (the pre-positioned arm admitted **1.96×** it on the gate's own `÷80 s` basis). What was **wrongly claimed done** on 3 Sep: keying the threshold to `pools_registered`. That is unbuildable as specified — the agreement ratio cancels the divisor, `pools_registered` counts *pools* not *slots*, and the sampler scrapes one arbitrary replica so no fleet sum exists (G125). What **did** land: an admission **floor** at half the measured ceiling, `SCALE`-guarded, closing the blind spot where an upper-only bound passed an origin that admitted nothing. Derivation in §12.5 | **basis: measured** — the 20,864 / 21,144 agreement comes from §12.5's 2x arm pair (`st2-drop-1788441064400`, `st2-drop-1788442576659`); the re-key is tracked as G125 |
| **G118 residue** — 83 transport failures | **open, class narrowed to a rate change (3 Sep).** A pinned ST-5 ran at a fleet held at exactly 7 for all 240 s (`st5-rehearsal-1788448170228`): **605,399 http reqs, 347,079 admitted, 0 unexpected of any class.** This does **not** discriminate fleet shape — the original 83 also occurred at a stable 7 under `st5-rehearsal` post-shm-fix, so both runs are the *same* shape. What it does bound is frequency: at comparable request volume the prior run's rate predicts 83, and observing 0 has `P ≈ 9e-37` under a uniform-rate model, so the rate is genuinely different rather than merely unresolved. The class is **not reproducible at this shape** and no mechanism is named. Intensity is not matched to §12.5's pre-positioned arm either (347,079 vs 592,045 admitted). The accept-path candidate's server-side arm is **unrunnable** (`railway ssh` refuses `saff-api`), and the counter evidence that appeared to refute it was withdrawn as fabricated (G122) | **basis: measured** — bounded by `st5-rehearsal-1788448170228`; no mechanism named, so the class stays open |
| **G118 surviving 5xx** — 5 origin-answered 500s | **separate class, not transport**, and **not reproducible**. `errorCode:1500` on `prequeue_join` (4) and `match_detail` (1) in `…-1788418365187`; a re-run at the same shape (`…-1788422130936`) produced **0 unexpected of any class**, with zero 5xx logged by `saff-api` — which records `message` and `stack` for every 500 it answers, so an occurrence would have left a line. Rate is below what two runs can measure. Next read is cheaper than this one was: the `stack` field is a top-level key on `saff-api`'s log line, so it is queryable in Railway's own log explorer with no credential at all — the `logs_read_token`/Loki route this row used to name went with the log-shipping lane on 5 Sep 2026 (G118) | **basis: retracted** — "not reproducible" was falsified by G129: four ST-3 runs give 1, 0, 0, 2 unexpected, an 83 ppm rate, and name the mechanism as SQLSTATE 40P01 on hold release. The class is reproducible and intermittent; this row's own wording predates that measurement |
| **G118 residue, four-run basis** — added 3 Sep, after the above | **The class is now bounded by four runs at one shape, not one.** Every `st5-rehearsal` run at `scale=1` sits within 0.1% of the same offered load — `http_reqs` 605,880 / 605,928 / 605,755 / 606,123 — and `saff_unexpected_total` across them reads **0, 1, 0, 0**. The newest (`st5-rehearsal-1788469874585`) carried **zero non-zero failure counters of any kind** across 606,123 requests, including the transport and `upstream_unavailable` families. So the original 83 is not merely unreproduced; it is now bounded at ≤1 per ~606k requests over four independent runs, against a prior run's rate that predicts ~83. No mechanism is named and none is claimed. Its verdict is **unconfirmed**: the red previously attributed to `saff_hold_success` 0.2518 cannot be one, since G104's re-key (`9a24c1e8`, 3 Sep 00:43) removed that metric from ST-5's thresholds 20 h before this run. 0.2518 is its ungated emitted value. Either way it is not a failure of this class | **basis: measured** — four runs at one shape, `http_reqs` within 0.1%; the four-run basis is the row's content |
| **§14.5 fault rows** | **operator-gated — four unexercised rows**, corrected again 4 Sep. §14.5 lists nine; **five** have been exercised (kill 50% of the fleet, Redis failover, Postgres failover, the 6-hour soak, and **clock skew** — falsified in `prequeue.pg.test.ts` rather than injected, because the row's two defences are the server-clock comparison and `exchangeAfter` being inside the receipt's HMAC, both single lines this repository owns). That leaves kill-all, +500 ms Postgres, Cloudflare challenge unavailable, and payment-provider 100% errors. Four of those five need a **third party** and none can be driven from this repository; the fifth (kill-all) is additionally blocked by G102, since the buyer path serves `private, no-cache, no-store` and §14.5's edge-projection claim has no artefact to project from. An earlier revision of this row said "six", which matched neither §11's five nor §14.5's nine | **basis: operator-blocked** — four of five rows need a third party (Cloudflare, the payment provider, clock skew, Postgres latency injection); kill-all is additionally blocked by G102 |

G115's three candidate fixes were **not equals**, and the ranking held:
the server-side deadline was the fix and it landed 3 Sep; raising the
client budget was made unnecessary by it; an idempotency key on acquisition
remains worth considering as defence in depth but was not required for
completeness.

### 12.7 What the night actually produced

Every number on this board has provenance — which pool it came from, which
population it describes, which fleet shape produced it, and which phase of a
drop it models. That was not true this morning, and it is the difference
between a report and a set of numbers.

The instrument investment paid for itself in a single line. Four hypothesis
cycles — fixture depletion, scale-up-as-cause, pooler queueing, dead-tuple
bloat — were each measured and refuted. Then the tag enumeration's first run
returned `errorCode:1500 = 982`, and the answer was immediate: the origin was
answering 5xx, when every hypothesis had assumed a client-side timeout.

**The scaling work is what found the ceiling.** Pushing the fleet to shapes
it had never reached is what exhausted a 62 MB arena nobody had named, on a
platform that exposes no way to declare it. "Scale up without sweating" was
not achieved by adding replicas — it was achieved by discovering, and
removing, the limit that made added replicas *harmful*.

### 12.8 G115 measured against production

The fix was deployed and then measured, rather than argued. The fix itself is
`6be6283f` (18:03 UTC); the deployed SHA the runs below exercised is
`15addd85` (18:17), a `saff-web` fixture follow-on that carries `6be6283f`
and reached `SUCCESS` at 18:17 — verified by reading both G115 halves out of
`git show 15addd85:` rather than assuming the descendant kept them. The two
18:xx runs started after it; both pre-fix runs predate it by eleven hours.

**`369eee4b` is now measured too, and it was the outstanding item this
section used to name.** It routes two SQLSTATEs that G115's own bounds made
reachable for the first time — a reviewer-found follow-up, unit-proven, and
`SUCCESS` on `saff-api` at 19:24. Its first production run is the last row
below.

| run | API commit | verdict | `unexpected` | `{status:0}` | hold max | hold avg |
| --- | --- | --- | --- | --- | --- | --- |
| `st3-contention-1788418780264` 06:59 | pre-fix | threshold-failure | 13 | 13 | **60,001 ms** | 3,084 ms |
| `st3-contention-1788419753065` 07:15 | pre-fix | threshold-failure | **65** | 62 | **60,004 ms** | 3,483 ms |
| `st3-contention-1788460612113` 18:36 | `15addd85` | threshold-failure | 2 | **0** | 20,714 ms | 979 ms |
| `st3-contention-1788460916756` 18:42 | `15addd85` | **pass** | **0** | **0** | 20,025 ms | 865 ms | **basis: measured** |
| `st3-contention-1788466720733` 20:18 | `369eee4b` | threshold-failure | 1 | 1 | 19,954 ms | 961 ms |

**The baseline is the same shape as the post-fix runs, so the comparison is
like-for-like.** Checked because the 60,004 → ~20,000 ms claim rests entirely
on it: all five runs are `scale=1`, `vus_max: 3000`, `iterations: 9000`,
`warmup=passed`, with `http_reqs` spanning 9,168–9,538 — a 4% band across
eleven hours and a deploy. No run hit the rate limiter
(`errorCode:1429 = 0`, all five).

**Two pre-fix runs exist, and this table cited one while quoting the other's
count.** `…-1788418780264` (06:59) carries 13 unexpected; `…-1788419753065`
(07:15) carries **65**, of which 59 are `errorCode:1050` and 62 are
`status:0`. An earlier version of this section attributed the 13 to the 07:15
run. Both are listed above rather than picking one, because two independent
pre-fix runs sixteen minutes apart are better evidence than either alone —
and the correction makes the drop larger, not smaller: **13 and 65 → 2, 0,
1.**

**The contention counter is the mechanism showing through, and it moved the
opposite way to everything else.** `saff_contended_total` reads **20** and
82 pre-fix against **384 / 257 / 353** after — roughly **16× more**
contention, on identical offered load. That is the fix working rather than a
regression: pre-fix a buyer could sit inside one unbounded 60-second attempt,
which registers as neither a win nor a contended loss. Bounding the attempt
returns that buyer to the race, where they are counted. `saff_held_total`
falls with it (468 → ~215) for the same reason — a hold that would have
completed past the client's budget is now refused rather than granted to
someone who has already gone.

**Three runs make the ceiling a band rather than a reading**, which is what
G114's heuristic asked for. Hold max across the post-fix runs is
**20,714 / 20,025 / 19,954 ms** — a spread of 760 ms on a ~20,000 ms value,
3.8%, so one population. All three sit inside `HOLD_DEADLINE_MS` (14,000)
plus one attempt's overshoot, which is the tightest a between-attempts check
can be. **Both** pre-fix runs sat at **60,001 and 60,004 ms** — 3× higher,
double the client's 30 s budget, and themselves a two-run population that
agrees to 3 ms.

**One profile detail looks like a knob and is not**, and it matters for
anyone reading the run rows above: the `duration` recorded against an ST-3
run is **inert**. `DURATION` is
imported by `st1-baseline.ts`; `st3-contention.ts` never reads it, because
its executor is `per-vu-iterations` bounded by `maxDuration: "10m"`. A buyer
population, not a clock, is the variable. So `duration=60s` on these rows is
metadata, and no run's shape was altered by it.

**The `status:0` class, checked per-run against the persisted counters.** It
reads **0, 0, 1** across the three post-fix runs, against **13 and 62** in
the two pre-fix runs. §12.1 registered it as the signature for "client
abandoned while the server was still working" — this defect seen from the
client end — so its near-disappearance is the fix's most direct evidence.

An intermediate version of this section claimed the 18:36 run carried one.
It does not: that run's two failures are **both `status:500`
(`errorCode:1500`)**, and its `status:0` is **0**. The claim was written
while correcting a different error and was itself wrong; it is recorded here
because the fix for it came from a mechanical diff of every table cell
against `loadtest.run`, not from re-reading the prose.

**The single post-fix occurrence is in `369eee4b`'s run, the newest code.**
One `status:0` in 9,253 requests is a rate this ladder cannot resolve
further, and it is not attributable to that commit on one run — but it is
the honest shape of the claim: the class is **suppressed to near-zero, not
eliminated**.

`369eee4b` adds no new failure class: zero `500`, `502`, `503` and `504`,
and a log grep across its window for the two error classes it introduces
(`HoldClaimTimeout`, the statement-ceiling message) returns **zero lines**.

**60,004 ms is the headline the fix was written against.** A hold ran a full
minute, double the client's 30 s budget, which is the orphan G115 describes:
that buyer's client had given up thirty seconds earlier while the server went
on to grant. Post-fix the maximum is 20,025 ms — inside `HOLD_DEADLINE_MS`
(14,000) plus one attempt's overshoot, which is exactly what a
between-attempts check predicts and cannot be tighter by construction.

**What the middle run's two failures were, because a partial result must not
be reported as a clean one.** Both were `status:500`, not `status:0`, and
`saff-api`'s own logs for that window show the class: `503` pool-checkout
timeouts (`"timeout exceeded when trying to connect"`) on
`/holds/best-available/` — the shedder refusing work it had no connection
for, which is G97's designed behaviour under an unwarmed fleet, not a fault
introduced here. Neither new error class appears in the logs at all: a grep
for `HoldClaimTimeout` and the statement-ceiling message returns **zero
lines**, so nothing in that window reached the paths this change added.

**What this does not prove.** One passing run is not a rate. The gate that
states the disagreement (`saff_hold_admitted_ms: p(99)<30000`) passed here,
but G114's heuristic applies to this claim as much as to any other: three
runs at the post-fix shape would make it a band rather than a reading. The
two runs also differ in `hold_success` (0.0249 vs 0.0187) and shed rate
(93.2% vs 95.3%), which is the endgame contention ST-3 exists to create and
is unrelated to this fix.
