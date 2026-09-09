# SAFF load testing — QA handover

Where the suite stands, how to run it, and how to read a result.
Current as of 3 September 2026.

**This document is self-contained** — no source access required. Full
measurements and their history are in `LOADTEST_REPORT.md`; engineering gap
IDs (`G94`, `G103`…) are in `GAPS.md`. Quote those IDs when filing.

---

## 0. Where it stands

**Two scenarios pass. Three are red. None of the three is a threshold set too
low, and one of them is the system behaving correctly.**

| | Verdict | What it means for you |
| --- | --- | --- |
| **ST-1** Baseline | **pass** | — |
| **ST-2** Drop burst | **pass** | — |
| **ST-3** Contention | **red — expected** | The shed gate fails because the fixture asks for six buyers per seat. Refusing 90% is arithmetic, not a defect. Report against **G95**; do not file |
| **ST-4** Failure injection | **red — real** | A Postgres failover still returns errors the spec does not permit. The count is not yet clean. Report against **G97** |
| **ST-5** Full rehearsal | **red — instrument** | `saff_hold_success` 44.1% vs >50%, but the metric counts a buyer *losing a race for a seat* as a failure. Report against **G104** |

**The release gate is not met.** `SPEC.md` §14.5 requires every fault row
exercised; **four of nine** are. The other five need an operator or a third
party — see §6.

**What a green run does and does not prove.** ST-1 and ST-2 passing means the
system is healthy at baseline and holds the drop-open burst. It does not cover
sustained contention, failover, or the full rehearsal shape, all of which are
red for the reasons above.

**Three known traps before you start**, each of which silently invalidates a
run or a conclusion:

1. **Confirm the database leader is in-region** (§5). A stranded leader makes
   every write ~90× slower and voids the run. **G103**
2. **Read verdicts from `/api/history/run`**, never `/api/summary` (§2).
   **G105**
3. **Ask for a re-seed before *every* ST-3 run** (§2). Runs inherit the
   previous run's holds. **G95**

---

## 1. What the suite proves

`saff-api` is the origin behind a ticketing drop. The suite answers one
question: **when ~400,000 people want ~25,000 sellable seats, does the system
refuse work in a controlled way, or does it break?**

**Refusing is a pass.** A `503` that says "shedding" is the degradation ladder
working. What fails a run is an error the ladder does not permit — a `500`, a
duplicate sale, or an origin that never recovers once a fault clears. Most
misread results come from treating a refusal as a failure.

Five scenarios, run in order. A failure at ST-1 makes the rest meaningless.

| ID | Name | Shape | Answers |
| --- | --- | --- | --- |
| ST-1 | Baseline | 50 browse/s + 5 purchase/s | Is the system healthy when nothing is wrong? |
| ST-2 | Drop burst | 5,000 joins/s, 80 s window (10 s ramp + 60 s hold + 10 s down) | Does the waiting room hold at drop-open? |
| ST-3 | Contention | 3,000 concurrent selectors, one section | Can two buyers get the same seat? |
| ST-4 | Failure injection | 30 req/s across a fault window | Does a dependency failure degrade or break? |
| ST-5 | Full rehearsal | 500 → 5,000 → 200 ramp | Does the whole shape hold end to end? |

---

## 2. Running a test

Runs start from the operator cockpit (the `saff-loadtest` service). It holds
the load generator, the scenarios and the run history.

**Getting access.** The cockpit is not reachable from the public internet: it
answers `400` to any request whose `Host` is not on its allowlist, and every
route except `/health` additionally requires HTTP Basic auth. Ask engineering
for two things: a shell session on the cockpit host (so requests originate
inside the private network) and the cockpit credential. A public URL will not
work, by design, and neither will the credential on its own.

From that session, requests go to `http://localhost:8080`:

```http
POST /api/run          { "scenario": "st1-baseline", "scale": 1, "duration": "60m" }
GET  /api/status                    # live progress of the current run
GET  /api/summary?id=<run>          # verdict — CURRENT PROCESS ONLY, see below
GET  /api/report?id=<run>           # full generator output
GET  /api/history                   # every finished run — survives restarts
GET  /api/history/run?id=<run>      # one finished run's stored result — use this one
```

All of them need the Basic credential; without it you get `401`.

**Read verdicts from `/api/history/run`, not `/api/summary`.** `/api/summary`
only answers for runs the current cockpit process started, so after any
restart — and §5's deploy warning is a restart — it returns `404 No such run`
for runs that finished normally. `/api/history/run` reads the database and
keeps serving them. Treat that `404` as "the cockpit restarted", never as
"the run does not exist" (G105).

**The three request fields.**

| Field | Values |
| --- | --- |
| `scenario` | `st1-baseline`, `st2-drop`, `st3-contention`, `st4-failure`, `st5-rehearsal` |
| `scale` | A number in `(0, 1]`. Anything above 1 is rejected with `400` |
| `duration` | `<digits><s\|m\|h>` — `60s`, `30m`, `6h`. Composite forms like `1h30m` are rejected. **Defaults to `60s`** |

**`scale: 1` alone is not a sign-off run.** `duration` defaults to 60
seconds, and the release gate is **`scale: 1` for 60 minutes** — the SLO
window is an hour, so a green 60-second run proves the harness works, not that
the system holds. ST-2's burst stages are fixed at 80 s and ignore `duration`.

**Every run needs a seeded fixture, and ST-3 needs its own.** Seeding is an
engineering task; request it before the run and name which one:

| Fixture | Command | Needed by |
| --- | --- | --- |
| The deployed match (`ban-vs-sri-group-a`) | `bun run demo:seed` **from `apps/saff-api`, with `DATABASE_URL` set** | ST-1, ST-2, ST-4, ST-5 |
| Contention fixture — one match, **one hot section** | `task saff-api:st3:seed` | ST-3 |
| Large catalogue for the soak | `task saff-api:soak:seed` | 6-hour soak |

The scenarios read one fixed match slug, so a seed that builds a *different*
match leaves every browse and hold hitting nothing. ST-3 additionally contends
3,000 selectors over a single section the general seed does not build; without
its own seed, `saff_exhausted_total` fires and the run is void.

**The exit code is the verdict, and `/api/history/run` carries it.** Read the
`verdict` and `exitCode` fields:

| `exitCode` | `verdict` | Means |
| --- | --- | --- |
| `0` | `pass` | Every threshold held |
| `99` | `threshold-failure` | A real result: some threshold broke |
| anything else | `error` | The run failed to execute — its numbers are evidence of nothing; re-run it |

Summaries are for interpretation, not adjudication: if a scenario summary
prints `PASS` while the exit code is non-zero, trust the exit code. This has
happened — a summary read `PASS` while the recovery threshold was failing.

---

## 3. Gates

Every threshold, quoted in full. A run passes only if all of its gates hold.

### ST-1 — Baseline

| Threshold | Meaning |
| --- | --- |
| `matches_list` p95 < 500 ms | Catalogue read stays interactive |
| `match_detail` p95 < 500 ms | Same, for one match |
| `hold_best_available` p95 < 1500 ms | A hold is a write; slower budget |
| `saff_shed_total == 0` | **Nothing may be refused at baseline load** |
| `saff_unexpected_total == 0` | No impermissible errors |
| `saff_upstream_unavailable_total == 0` | No dependency was down |
| `saff_exhausted_total == 0` | The fixture had inventory — guards a vacuous run |
| `saff_hold_success > 0.95` | Holds mostly succeed when uncontended |

Shedding at baseline means the fleet is undersized before the drop starts.

### ST-2 — Drop burst

| Threshold | Meaning |
| --- | --- |
| `saff_unexpected_total == 0` | No impermissible errors under burst |
| `saff_upstream_unavailable_total == 0` | No dependency failed |
| `matches_list` p95 < 1000 ms | Browsing survives the burst |
| `prequeue_join` p99 < 2000 ms | The queue door stays responsive |
| `saff_admitted_total{endpoint:prequeue_join}` < **301,600** | **The origin-load gate** — see below |
| `dropped_iterations{scenario:burst} < 3500` | The *generator* kept up — 1% of the burst's 350,000 offered joins |

**The origin-load gate only bites at `scale: 1`.** Its budget is the origin's
measured ceiling (≈3,770 req/s, derived from pooler capacity ÷ transaction
time) × the fixed 80-second window. Below full scale it passes without meaning
anything, so **a green ST-2 at `scale: 0.3` says nothing about origin load**.

The `dropped_iterations` row measures the *generator*, and it stops the gate
passing vacuously — an upper bound on admissions is satisfied by admitting
nothing. A healthy `scale: 1` burst dropped 385 of 349,725.

Shedding is **not** thresholded here. A burst that sheds is the design working.

### ST-3 — Contention

| Threshold | Meaning |
| --- | --- |
| `saff_unexpected_total == 0` | No impermissible errors |
| `saff_upstream_unavailable_total == 0` | No dependency failed |
| `hold_best_available` p99 < 3000 ms | Contention resolves promptly |
| `saff_exhausted_total == 0` | **The fixture had inventory** — guards a vacuous run |
| `saff_shed_total` < 15% of attempts | Refusals stayed incidental — **currently expected to fail, see below** |

Attempts = selectors × 3. At `scale: 1` that is 9,000 attempts and a shed
budget of 1,350; at 0.33, 446; at 0.17, 230.

**This gate is known-red and the red is correct behaviour.** A sweep at five
populations, re-seeded before each, showed the shed ratio tracking load
monotonically — 25% at 150 selectors up to 90% at 3,000 — with `exhausted` at
zero throughout. Nothing is undersized; 3,000 selectors on a 500-seat section
is six buyers per seat, and refusing 90% is arithmetic. Whether §14.5 should
ask for that ratio is a spec decision. **Report the shed count against
attempts and reference G95; do not file it as a fresh finding.**

**Ask for a re-seed before *every* ST-3 run, not just the first.** Runs
inherit the previous run's holds: after three back-to-back runs the section
held **21 of 500** seats available, and a run starting there refuses almost
everything whatever the load — so two ST-3 runs without a reset between them
are not comparable. The seed prints what it built; expect **500 seats across
25 rows of 20**. A different seat count is a different fixture.

The other four ST-3 thresholds are live gates and a breach of any of them is
real.

**The oversell invariant is checked after the run, not by a threshold** — an
engineering-run script inspects the sales table for duplicate seats. Ask for it
to be run and for its output; a passing ST-3 does not by itself prove no
oversell.

### ST-4 — Failure injection

| Threshold | Meaning |
| --- | --- |
| `saff_unexpected_total == 0` | Under a fault the system sheds; it does not error |
| `http_req_failed{endpoint:health}` rate < 0.5 | The origin was answering again by the end |

Shedding and upstream-unavailable are **reported, not thresholded**: this
scenario exists to take a dependency down. The scenario does not inject the
fault itself; an operator does, by hand, during the run.

**Whether zero shed means "the fault missed" depends on the fault**, so agree
the expectation before the run:

| Fault injected | Expected |
| --- | --- |
| Postgres leader failover | Shedding and upstream-unavailable rise, then recover. **Zero shed means the fault did not land — re-run it.** |
| Redis restart | **Nothing changes.** The ephemeral tier fails open by design and queue positions live in Postgres, so no shed, no error and an unchanged join rate *is* the pass. Confirm the restart actually happened inside the run window, or the run proves nothing. |
| API replica kill | Joins continue on the surviving replicas; a latency tail is expected, errors are not. |

### ST-5 — Full rehearsal

| Threshold | Meaning |
| --- | --- |
| `saff_unexpected_total == 0` | No impermissible errors across the whole shape |
| `saff_upstream_unavailable_total == 0` | No dependency failed |
| `matches_list` p95 < 1000 ms | Browsing holds through the ramp |
| `saff_hold_success > 0.5` | Half of holds succeed at peak contention. **A red here is not by itself a defect (G104):** the rate counts a buyer *losing a race for a seat* as a failed hold, and under contention someone must lose. Check `saff_exhausted_total` — if it is 0, no request met an empty fixture and the gate is measuring contention, not a seating failure |

---

## 4. Reading a result

| Counter | Means |
| --- | --- |
| `saff_admitted_total` | The origin did the work the request asked for |
| `saff_shed_total` | A deliberate refusal — **healthy**, tagged by `class` |
| `saff_upstream_unavailable_total` | A dependency was unreachable — **not healthy** |
| `saff_unexpected_total` | An error the ladder does not permit — **a defect** |
| `saff_contended_total` | Someone else took the seats first — healthy under load |
| `saff_exhausted_total` | No inventory left — a fixture problem |
| `saff_held_total` | Holds acquired |
| `saff_ordered_total` | Checkouts created (`DRAFT` orders) — **not sales** |
| `saff_hold_success` | Share of hold attempts that produced a hold |
| `saff_hold_duration_ms` | Hold latency distribution |

**Two `503`s that are not the same thing.** `SHEDDING` is the system choosing
to refuse, credited as a free refusal. Every other `503` — chiefly
`UPSTREAM_UNAVAILABLE` — counts as work the origin absorbed *and* as a
dependency being down. A run where the second is non-zero has found something.

**`saff_shed_total` carries a `class` tag** naming the funnel depth that
refused: `0` a commit path (never shed), `1` a hold, `2` a queue join, `3`
catalogue browsing. Higher classes shedding first is the design; class 1
shedding heavily means existential load. `rate-limit` tags a `429`, which is
not funnel-depth shedding.

**No load scenario records a sale.** A load run creates `DRAFT` orders and
stops; sale rows are only written when a payment is captured, which the suite
deliberately does not drive. So `saff_ordered_total` measures the checkout path
under load, and the post-run oversell check only ever inspects sales seeded by
its own harness — not sales the load test made.

**Never-completed requests are counted, not hidden.** A request that got no
response at all is recorded with status `0`. It lands in
`saff_unexpected_total` but in neither `admitted` nor `shed`, so:

```
never-completed  =  http_reqs − (saff_admitted_total + saff_shed_total)
```

Compute that before calling an over-budget `unexpected` a regression: **the
remainder after subtracting is the number to file.** Report both figures.
Under an injected fault most `unexpected` are real `5xx` responses instead,
which is why the subtraction matters.

**Check the leader's region first** (§5). A remote leader manufactures this
class wholesale — 22 of them on one run, **zero** an hour later on the same
scenario with the leader local (G103, G94).

---

## 5. Standing constraints

**Nothing may deploy the cockpit while a run is in flight.** A deploy replaces
the container and kills the running load process — its live progress and its
on-disk result files go with it. This has already cost one 38-minute soak.
*Finished* runs are safe: they are written to the database at exit and stay
readable through `/api/history`. A commit touching any
of these paths triggers that deploy:

```
apps/saff-loadtest/**   packages/retry/**   packages/basic-auth/**
package.json            bun.lock            mise.toml
```

The API service watches its own paths (`apps/saff-api/**`, several shared
packages, and the same three root files). A deploy there does not kill the load
process but restarts every replica, resetting the memory baseline a soak exists
to measure. **Before a long run, tell engineering to hold merges; during one,
only documentation changes are safe.**

**The cockpit's credential must never be removed.** `POST /api/run` fires 5,000
joins/s at production. The credential and the `Host` allowlist are both
required; neither alone contains it.

**The public hostname is recreated by the platform after deletion** — deleting
it is not a containment fix.

**Soak memory is judged from container metrics, which live in the platform
dashboard, not in the run output.** The measured quantity is container RSS; the
known leak class is invisible to in-process heap figures, so the load
generator's own numbers cannot answer it. Ask engineering for the RSS graph
over the run window.

**Confirm the database leader is in-region before every run, and record it
with the result** (G103). A failover can leave it on the SFO standby, where
writes cross the Pacific at ~178 ms instead of ~2 ms; nothing fails back and
nothing alerts. On 3 Sep that went unnoticed for over six hours. The same
scenario an hour apart, remote leader then local: 22 never-completed requests
then **0**, hold success 6.9% then 44.1%. **A run on a remote leader is not
evidence of anything.**

**Ask for the median write latency, not the leader's role.** The role field
says which node is leader, not how far away it is — on 3 Sep it read `leader`
throughout. Engineering can measure it in a few seconds; **single digits in
milliseconds is in-region, anything near 178 ms means the run is invalid**
(§7, item 4). Record the figure with the result.

**One run at a time.** The cockpit serialises runs, so a soak occupies it for
its full duration; scenario runs and a soak cannot overlap.

---

## 6. Status

**Holds** — measured against a running system. Quote these when a run's result
looks anomalous; they are the reference points.

| Property | Evidence | When |
| --- | --- | --- |
| Origin ceiling is derived, not declared | ≈3,770 req/s, from pooler capacity ÷ measured transaction time | 1 Sep |
| Redis loss does not stop joins | 326 joins across a Redis outage proven by an independent TCP probe: zero `503`s of either kind on the join endpoint | 2 Sep |
| The cockpit is not reachable publicly | Public hostname answers `400` on every route, including health | 1 Sep |
| Shed refusals carry their funnel class | Every refusal in the ST-3 sweep was tagged; server-side and generator counts agree | 1 Sep |
| Half the api fleet dead does not stop joins | ST-2 `scale: 1`, 1 of 2 replicas dead for the whole burst: 245,138 joins at 1,338/s, p95 103 ms, 0.076% failures | 1 Sep |
| Postgres failover sheds, but still returns some errors | Real Patroni failover in-window: 567 upstream-unavailable (permitted) against **22 unexpected**, 21 of them `500` on write paths, zero on reads. Down from 1,566 → 53 → 22 but not zero; §6.10 permits `503`, not `500` (G97) | 2 Sep |
| Failover leaves no duplicate sales | Oversell script against the live database: 0 duplicates, 0 sold-without-claim, 0 sold-with-live-claim | 1 Sep |

**Open** — treat as unverified. Quote the `Ref` when filing so a known issue
is not raised twice; the full entry for each lives in `GAPS.md`.

| Item | Why it matters to a run | Ref |
| --- | --- | --- |
| Postgres failover returns impermissible `500`s | Expect a small non-zero `unexpected` count on any failover run. Do not file it as new | G97 |
| Requests that never complete (status `0`) | Adds noise to a fault-run verdict. **Not reproducing since 3 Sep** with the leader local — check leader placement (§5) before filing one | G94 |
| ST-3 sheds far past its budget | Expected; the fixture asks for six buyers per seat. Report the count, do not file | G95 |
| ST-5's `saff_hold_success` reads low | The metric counts a lost race as a failed hold. Check `saff_exhausted_total` is 0 before reading it as a defect | G104 |
| `/api/summary` 404s after a cockpit restart | Use `/api/history/run`. The `404` means the cockpit restarted, not that the run is missing | G105 |
| A cross-region database leader voids a run | Nothing alerts. Confirm placement before every run and record it (§5) | G103 |
| Five of nine fault rows never injected | +500 ms on Postgres, kill-all API replicas, an edge challenge, payment-provider errors, clock skew. Each needs an operator or a third party; the kill-all row additionally has no cacheable buyer path to project from | G102 |
| Selling across a fault window | No scenario drives payment capture, so oversell *during* a failover is untested | — |
| Queue position restore after Redis returns | The 2 Sep run proved joins continue **without** Redis, which is a different property | VR-17 |

---

## 7. Filing a failure

A load-test failure is one of four things, and saying which is most of the
work:

1. **A system defect** — `saff_unexpected_total > 0` *after* subtracting
   never-completed requests (§4), a duplicate sale, or an origin that never
   recovered. File against the API.
2. **A known open issue** — the ST-3 shed budget (G95), never-completed
   requests (G94), or an ST-5 `saff_hold_success` red (G104). Report the
   numbers against the existing ref; do not file a new item.
3. **A fixture problem** — `saff_exhausted_total > 0`, or a run against an
   unseeded or wrongly-seeded match. Re-seed and re-run; do not file.
4. **An invalid run** — `verdict: "error"` (any exit code other than `0` or
   `99`), `dropped_iterations` over budget, **or a run whose database leader
   was out of region** (§5, G103). The run did not execute, under-delivered,
   or measured a ~90× write penalty instead of the system; its numbers are
   evidence of nothing either way. Re-run it.

Include: run id, scenario, `scale`, `duration`, the `/api/history/run` output
(which carries `verdict` and `exitCode`), and — if a fault was injected — what
was done and when, in wall-clock time. A fault window that cannot be aligned to
the run timeline cannot be interpreted.
