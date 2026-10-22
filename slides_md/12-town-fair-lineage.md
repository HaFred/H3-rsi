# 12. Long-Form Example: The town-fair Rewrite Lineage

Source: PPTX page 12 of 17.

The same recursive lineage progresses from Base to script rewrite, storyboard rewrite, and a further script rewrite.

| Candidate | Change | Mean Validation S_project | Mean Validation R_project |
|---|---|---:|---:|
| `d16c2e95` | Base | 0.7293 | 0.7645 |
| `c863aae2` | Round 1: script | 0.7436 | 0.7698 |
| `3252791a` | Round 7: storyboard | 0.7477 | 0.7834 |
| `c1912a62` | Round 57: script again | 0.7516 | 0.7672 |

S_project/R_project are means across **four Validation projects**, rather than town-fair scores. The final edge improves S_project while reducing R_project. Both are measured on the same Validation projects; S_project determines selection, while R_project is also available there as a diagnostic. Promotion would use newly measured R_project on separate held-out cases. The displayed `101416` run was interrupted before promotion/Test.

The five intended segment scenes are red-umbrella arrival, park kite, fountain cyclist, bakery preparation, and pottery-stall opening. Each row's images correspond to the videos on [page 13](13-town-fair-videos.md), with one final frame per segment. Each project contains 5 × 124 frames, about 25.83 seconds.

Final frames show endings, rather than complete actions or proof that each intended scene was fulfilled. Train seeds differ across these rows: base, storyboard and script-again are seed 1, and the first script rewrite is seed 0. The worked example resolves all four by byte identity — base seed 1 and script rewrite seed 0 in [§8.1](subdocs_examples/11-town-fair-longform-production.md#81-resolve-the-two-original-production-identities), storyboard and final winner in [§8.2](subdocs_examples/11-town-fair-longform-production.md#82-storyboard-and-final-winner-identities).

The [detailed five-clip production comparison](subdocs_examples/11-town-fair-longform-production.md) expands the Base and first Script rewrite into segment briefs, agent calls, H3 requests, state transitions and stitching. Its runtime records were recovered from the 10x run; the comparison does not cover the later two rewrites.

### Source images

![Page 12, image 1](assets/image7.png)

![Page 12, image 2](assets/image8.png)

![Page 12, image 3](assets/image9.png)

![Page 12, image 4](assets/image10.png)

![Page 12, image 5](assets/image11.png)

![Page 12, image 6](assets/image12.png)

![Page 12, image 7](assets/image13.png)

![Page 12, image 8](assets/image14.png)

![Page 12, image 9](assets/image15.png)

![Page 12, image 10](assets/image16.png)

![Page 12, image 11](assets/image17.png)

![Page 12, image 12](assets/image18.png)

![Page 12, image 13](assets/image19.png)

![Page 12, image 14](assets/image20.png)

![Page 12, image 15](assets/image21.png)

![Page 12, image 16](assets/image22.png)

![Page 12, image 17](assets/image23.png)

![Page 12, image 18](assets/image24.png)

![Page 12, image 19](assets/image25.png)

![Page 12, image 20](assets/image26.png)

---

[Contents](README.md) · [Previous](11-long-form-production.md) · [Next](13-town-fair-videos.md)
