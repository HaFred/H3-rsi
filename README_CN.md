# 搜索评分与独立验收

项目面向长程视频生成的持续改进；当前已实现冻结模型下的 Harness 搜索与独立晋升，模型权重训练及 Harness／权重联合晋升属于后续目标。

当前 Harness RSI 在冻结模型与评测口径下，用 GEPA 搜索五个 Agent 的系统提示词及共享经验 memory。评分名称明确区分单位：`R_clip` 是单个短片 / 段落的基础奖励，`S_clip` 是其搜索目标；`R_project` 与 `S_project` 是完整多段项目的奖励 / 报告分与搜索分。项目评分共享 `C = mean(R_clip)`，但项目分不等于片段分，也不以 `mean(S_clip)` 替代 C。

任务按来源或完整项目隔离为四组：Train 提供反思证据，并用对应层级的搜索分（`S_clip` 或 `S_project`）比较父子候选；Validation 默认按平均搜索分选择 best；Promotion 在独立任务上配对复评 base 与 best，结合对应层级的奖励（`R_clip` 或 `R_project`）、有效性及回退约束决定是否晋升；Test 仅报告留出任务上的质量差异，不参与搜索或晋升。基础奖励在搜索 case 上也会计算；“先搜索、再晋升”是决策顺序，不表示搜索依赖未来的 Promotion/Test 结果。搜索分提升不保证奖励分提升或候选能够晋升。完整公式见[评分与切分说明](slides_md/07-data-splits-and-scores.md)。

