# 16. Experimental Boundaries and Implementation Roadmap

Source: PPTX page 16 of 17.

### Demonstrated in existing runs

- Real H3 outputs drive Harness search.
- Validation, Promotion, and Test have independent roles.
- A winning version was promoted to `gepa-base.1`.

These results include the recovered `101416` run, which completed Promotion and Test after interruption and resume. Its final winner passed the gate, but the champion store was not updated because its current champion did not match the expected base. The `gepa-base.1` promotion above refers to the earlier single-clip run. See the [run summary](subdocs_examples/README.md#how-search-and-promotion-relate).

### Next stage

- Establish benefits from verl-omni model-weight training.
- Develop long-form credit assignment and a joint gate.
- Test sustained improvement across runs and fresh storylines.

The implementation sequence is to validate an H3 training recipe with the Harness fixed, establish checkpoint versions, and then connect the four-combination evaluation described on [page 15](15-joint-optimization.md).

---

[Contents](README.md) · [Previous](15-joint-optimization.md) · [Next](17-findings-and-pilot-prompts.md)
