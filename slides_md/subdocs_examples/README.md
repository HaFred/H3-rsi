# Worked examples: runs, harnesses and promotion

These examples show the same production pipeline at different stages of development. **`c863aae2` and `f22f8b8a` do not belong to the same search run.** A harness hash identifies its prompt/memory content, not a run or a video; the same frozen harness can be reused across runs.

| Worked example | Run | What is compared |
| --- | --- | --- |
| [Single-clip prompt lifecycle](08-script-rewrite-prompt-lifecycle.md) | `101214_1strun` | Base versus a script rewrite on `train-001` |
| [Town-fair production](11-town-fair-longform-production.md) | `101416_1strun` | Base `d16c2e95`, Train seed 1, versus script rewrite `c863aae2`, Train seed 0 |
| [Tea-farm production](11-tea-farm-longform-production.md) | `101704_1strun` | Frozen base `531468a1` versus selected memory rewrite `1d079e4c`, both Promotion seed 0 |

## How search and promotion relate

GEPA proposes harness text changes using Train feedback. Validation ranks the candidates under the search objective and selects a winner. Independent Promotion compares that winner with the run's frozen base on held-out cases. A search winner is therefore not automatically a promoted champion, and one intermediate candidate's video does not establish the final winner's promotion result. Test is report-only.

For these longform runs, `S_project = 0.70 C + 0.30 H` drives search, while `R_project = 0.40 C + 0.20 F + 0.20 P + 0.20 H` is the report/gate score when all metrics are present. Here C is mean clip reward, F confirmed-state rate, P eligible payoff pass rate, and H boundary-frame similarity. See the [score reference](../07-data-splits-and-scores.md).

| Run | Recorded wall-clock duration | Search result | Independent promotion / champion outcome |
| --- | --- | --- | --- |
| `101214_1strun` | About **2 h 34 min**; Oct 12 22:20 to Oct 13 00:54 | Selected `4ddd73aa`; the illustrated `d58fde46` is an intermediate script rewrite | Accepted and promoted as `gepa-base.1`. See the [single-clip results](../../rfc748/readme.md). |
| `101416_1strun` | **[DGX_PLACEHOLDER]** Net active-session duration; exclude downtime between interruptions/resumes | Town-fair presentation lineage: `d16c2e95` → `c863aae2` → `3252791a` → selected `c1912a62` | Recovered gate record accepts the final winner: mean R_project gain **+0.03206** over eight Promotion cases. **The champion store was not updated**: its champion was `gepa-base.1`, whereas this run expected `gepa-base`. This verdict concerns `c1912a62`, not intermediate `c863aae2`. |
| `101704_1strun` | **18 h 58 min 20 s**; Oct 17 12:57 to Oct 18 07:55 | Started from the new frozen `gepa-base.2` (`531468a1`); selected memory rewrite `1d079e4c` | **Rejected**: mean gain +0.004785 is below 0.02, and tea-farm seed 0 regresses by −0.07474, beyond the allowed 0.05. No new champion. See the [decision](../../longform/101704_1strun/promotion/decision.json). |

Times above are **2026, Asia/Shanghai (UTC+8)** and rounded to the minute; durations describe elapsed wall time, not continuous GPU computation. The [timing evidence](assets/run-context/run-timings.json) records the exact source timestamps. `101214` spans manifest creation through the last Test evaluation. `101704` uses its full run report and matches the [main README](../../README.md#2-longform-rsi-five-clips-searched-as-one-project). For `101416`, net active-session duration is not yet established locally. **[DGX_PLACEHOLDER]** Recover process/session start and exit timestamps for the original run and every resume/recovery session; sum non-overlapping active intervals, including initial calibration, search, reflection, Promotion and Test, and exclude downtime between sessions. The local log starts with a completed project (initial start missing), and the Oct 16 03:05 UTC recovery has no explicit session-end marker, so subtracting the obvious gaps cannot establish a complete total. Preserve the source timestamps and calculation in the timing evidence. Do not substitute summed H3 generation time, which omits other work, or the report's **1,256.2 s**, which covers only the final cache-replay session. `101704`'s 18 h 58 min is full reported wall time, not GPU-only time; use active-session wall time for a comparable total after removing `101416`'s inter-session downtime.

**The bridge between the two longform examples:** the `101704` [base snapshot's adoption notes](../../longform/101704_1strun/base.yaml) say it combines `101416` winner `c1912a62`'s script/storyboard with 12 memory heuristics carried from `gepa-base.1` after `101318_1strun`. That combination becomes `gepa-base.2`, hash `531468a1`. It is a new frozen configuration, not the earlier `c863aae2` candidate renamed. The `101704` memory search then produces `1d079e4c`, `04f60f24` and `f22f8b8a`, among other candidates.

The town-fair presentation describes an earlier interrupted snapshot; the recovered `101416` report and gate record establish the later completed outcome above. Gate acceptance and a successful champion-store update are separate events. The relevant recovered gate record is [copied here](assets/run-context/101416-promotion-decision.json); the base snapshot independently records the adoption and store mismatch.

## Harnesses referenced by these examples

SHA prefixes below are **candidate hashes**, not media SHA-256 values or project-directory hashes. Changes are relative to the base of the named run; they are not necessarily the single edit made on the latest parent.

| Harness SHA prefix | Run / use | Changes relative to that run's base | Role in the examples / selection |
| --- | --- | --- | --- |
| `d16c2e95` | Base reused in `101214` and `101416` | None | Single-clip baseline and displayed town-fair baseline |
| `d58fde46` | `101214_1strun` | Script | Illustrated intermediate single-clip rewrite; not the final winner |
| `4ddd73aa` | `101214_1strun` | Script + director | Final selected and promoted single-clip harness |
| `c863aae2` | `101416_1strun` | Script | Round 1; displayed town-fair rewrite; not this run's final winner |
| `3252791a` | `101416_1strun` | Script + storyboard | Round 7; storyboard edit in the town-fair lineage |
| `c1912a62` | `101416_1strun` | Script + storyboard | Round 57; further script edit, final search winner; source of the later base's script/storyboard |
| `531468a1` | `101704_1strun` | None | New frozen `gepa-base.2`; displayed tea-farm baseline |
| `1d079e4c` | `101704_1strun` | Memory | Validation S_project 0.755050; selected winner, rejected by Promotion; displayed tea-farm rewrite |
| `04f60f24` | `101704_1strun` | Memory + storyboard | Fully validated child; Validation S_project 0.750795; not selected |
| `f22f8b8a` | `101704_1strun` | Memory + script | Fully validated child; Validation S_project 0.753540; not selected and not the displayed tea-farm rewrite |

Sources: [single-clip search report](../../search_report.json), [town-fair lineage](../12-town-fair-lineage.md), [town-fair recovered identities](11-town-fair-longform-production.md#81-resolve-the-two-original-production-identities), [101704 search records](../../longform/101704_1strun/search.json), and [tea-farm harness identities](11-tea-farm-longform-production.md#12-what-actually-changed-between-the-two-harnesses).

The town-fair videos use different seeds and are illustrative. Tea-farm compares the same held-out case and seed, but its lower quality is a rejection example. Neither pair's production scores should be confused with mean Validation scores across other projects.
