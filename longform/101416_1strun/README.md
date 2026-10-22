# Recovered longform run 101416_1strun

These files are copied byte-for-byte from the sibling implementation checkout's `benchmarks/longform_example/101416_1strun/`. [provenance.json](provenance.json) records source location and SHA-256 hashes.

- [search.json](search.json): 22 candidate configurations; 9 full Validation scores; selected `c1912a62`.
- [report.json](report.json): recovered completed result, Promotion and Test outcomes. Its 1,256.2-second elapsed duration covers the final cache-replay session, not all sessions.
- [promotion/decision.json](promotion/decision.json): gate accepted at mean gain +0.03206; champion store unchanged due to base mismatch.
- [base.yaml](base.yaml) / [best.yaml](best.yaml): frozen baseline and final Script/Storyboard rewrite; memory empty in both.
- [calibration.json](calibration.json): frozen search/report weights and disabled loop penalty.
- [budget.json](budget.json): 615 Search + 80 Promotion + 21 Test = 716 H3 requests.
- [videos.json](videos.json): original output locations and per-project component metrics. Original `outputs/...` paths are not local media links.

The [main README](../../README.md#2-longform-rsi-five-clips-searched-as-one-project) summarizes this run. The [town-fair worked example](../../slides_md/subdocs_examples/11-town-fair-longform-production.md) preserves the first two displayed productions' recovered traces and media hashes, and [§8.2](../../slides_md/subdocs_examples/11-town-fair-longform-production.md#82-storyboard-and-final-winner-identities) resolves the other two. Net active-session duration is **12 h 32 min 52 s of recorded activity** across four processes, a lower bound because the third process died without an exit record; the [timing evidence](../../slides_md/subdocs_examples/assets/run-context/run-timings.json) records every session boundary and the excluded downtime.
