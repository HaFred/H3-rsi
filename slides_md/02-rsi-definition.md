# 02. RSI and AI Agents: The Project Definition

Source: PPTX page 2 of 17.

In this project, RSI means using execution evidence to modify reusable generation strategies, comparing old and new versions, validating on independent tasks, and persistently inheriting effective changes for further iteration.

| Criterion | Basic agent capability | RSI requirement in this project |
|---|---|---|
| Version comparison | Record experience, update memory, adapt execution | Compare the modification with a baseline on identical tasks and comparable budgets |
| Independent validation | Complete the current task using past experience | Decide promotion on tasks excluded from search; use Test only for reporting |
| Stable inheritance and recursion | Read saved experience on later calls | Make an effective version the starting point for later production and improvement, allowing changes to accumulate across generations |

Remembering experience is a capability; verifying and inheriting a more effective strategy closes the improvement loop. This is an operational definition for this project, rather than a universal definition or a claim that OpenClaw cannot support such extensions.

Earlier short-clip and long-form runs completed independent Promotion and Test. The recovered `101416` records also establish completion after interruption and resume: the final winner passed the gate, but the champion store was not updated because its current champion did not match the expected base. See the [run summary](subdocs_examples/README.md#how-search-and-promotion-relate). Repeated gains on fresh works after inheriting a previous champion remain unverified; missing evidence does not establish that improvement is ineffective.

---

[Contents](README.md) · [Previous](01-overview.md) · [Next](03-harness-workflow.md)
