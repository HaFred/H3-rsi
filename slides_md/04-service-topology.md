# 04. Inference and Evaluation Services

Source: PPTX page 4 of 17.

| Role | Model / responsibility | Endpoint | GPU |
|---|---|---|---|
| Video generation | MiniMax-H3 fl2va, SGLang TP2 | `:30010` | a/b |
| Production text | neohorse-1-4B | `:8000` | c |
| Reflection | Qwen3.5-9B; propose one-component Harness rewrites | `:8001` | c |
| Decisions | NeoHorse-Jev-4B; story / gates | `:8085` | c |
| Visual judge | UnifiedReward-2.0-qwen3vl-2B; video axis | `:8090` | c |

One production text model serves five roles: scriptwriter, storyboard planner, director, text critic, and editing adviser. The first three produce video inputs. The latter two provide diagnostics after generation.

The short-clip run used **65 H3 requests**: 45 for search, 12 for promotion, and 8 for Test, against a total limit of 80 (60 + 12 + 8).

- Search includes iteration 0 Base evaluation and iterations 1–13. Each attempted rewrite compares parent/child on the same two Train tasks; passing candidates receive full evaluation on two Validation tasks. Cache hits do not add physical H3 requests.
- Four new candidates reached full validation. The winning lineage changed script at iteration 1 and director at iteration 10.
- Promotion: 6 tasks × 2 versions × 1 seed = 12 requests.
- Test: 4 tasks × 2 versions × 1 seed = 8 requests.

The two versions are Base and winner. Promotion/Test do not rewrite the Harness; eight Test requests do not mean eight search rounds. Each H3 request produces one clip: 124 frames, about 5.17 seconds, 1344 × 768, 24 fps, and 10 inference steps.

---

[Contents](README.md) · [Previous](03-harness-workflow.md) · [Next](05-search-boundary.md)
