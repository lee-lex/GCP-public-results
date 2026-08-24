# GCP Compact Representation 可复用性评估首轮结果

## 结论口径

本轮主要指标是整景候选池中的 `GCP Reuse@K`，不是 GT 附近局部搜索的定位成功率。候选池在读取 GCP/GT 之前冻结；GT 只用于标记真实候选并在完成排序后评价。局部精化仅对 Top-1 身份正确的匹配计算。

## 数据与审计

- 固定历史 GCP：500 个，全部从 2024 reference 只编码一次。
- 跨时相查询：666 条。
- 主候选池：M=1000；另有多级嵌套候选池对比。
- 候选来源：Harris、Shi-Tomasi、FAST、AGAST、SIFT keypoint、ORB、AKAZE、KAZE、BRISK、MSER、Blob、Laplacian、Hough intersection 与 Overture connector。
- SCL/Overture 只过滤候选中心；256×256 reference 与 candidate patch 均保留完整水体、植被、阴影和建筑上下文。
- RANSAC 不参与候选排名，只在全部历史 GCP 独立完成匹配后运行。

## 核心结果

- 32 B Rank/Hamming: 32.0 B/GCP，Reuse@1=0.093，Reuse@5=0.101，MRR=0.097，5 px 候选覆盖率=0.111，候选存在时 Reuse@1=0.851。
- 32 B 中值二值基线: 32.0 B/GCP，Reuse@1=0.092，Reuse@5=0.098，MRR=0.095，5 px 候选覆盖率=0.111，候选存在时 Reuse@1=0.825。
- 64 B 方向/L1: 64.0 B/GCP，Reuse@1=0.111，Reuse@5=0.111，MRR=0.111，5 px 候选覆盖率=0.111，候选存在时 Reuse@1=1.000。
- 128 B DCT 符号/Hamming: 128.0 B/GCP，Reuse@1=0.111，Reuse@5=0.111，MRR=0.111，5 px 候选覆盖率=0.111，候选存在时 Reuse@1=1.000。
- 256 B 幅值+方向/Hamming: 256.0 B/GCP，Reuse@1=0.111，Reuse@5=0.111，MRR=0.111，5 px 候选覆盖率=0.111，候选存在时 Reuse@1=1.000。
- JPEG2000 100 + NCC: 6447.8 B/GCP，Reuse@1=0.110，Reuse@5=0.111，MRR=0.110，5 px 候选覆盖率=0.111，候选存在时 Reuse@1=0.982。
- WebP Q30 + NCC: 11674.3 B/GCP，Reuse@1=0.110，Reuse@5=0.111，MRR=0.110，5 px 候选覆盖率=0.111，候选存在时 Reuse@1=0.982。
- 原始灰度 NCC: 65536.0 B/GCP，Reuse@1=0.110，Reuse@5=0.111，MRR=0.110，5 px 候选覆盖率=0.111，候选存在时 Reuse@1=0.982。

## 必答问题

1. **128 B 能否重新找到自己？** Reuse@1=0.111，Reuse@5=0.111，MRR=0.111，候选存在时 Reuse@1=1.000。这是整景候选排名结果，不是局部 ±64 px 指标。
2. **64 B 能否？** Reuse@1=0.111，Reuse@5=0.111，MRR=0.111，候选存在时 Reuse@1=1.000。
3. **候选池扩大后的变化：** compact_64B_orientation_l1: M=100 时 0.044，M=5000 时 0.267；compact_128B_dct_sign_hamming: M=100 时 0.044，M=5000 时 0.270。端到端 Reuse@1 随 M 上升，是因为无 GT 注入的全景池覆盖率从小池到大池持续增加；方法辨识力应同时查看 `conditional_reuse_at_1`。
4. **2024→2025 与 2024→2026：** compact_64B_orientation_l1: 2024→2025=0.160，2024→2026=0.170；compact_128B_dct_sign_hamming: 2024→2025=0.160，2024→2026=0.170。这里只统计两个目标年份都具备高质量 GT 的同一批历史 GCP。
5. **Rank 最稳定：** 128 B DCT 符号/Hamming，MRR=0.111。
6. **Margin 最大：** JPEG2000 100 + NCC，median normalized margin=1.36711；不同距离只比较归一化间隔。
7. **Soft context 影响：** 详见 `soft_context_reuse_summary.csv` 与图 06；统计使用完整 patch，没有把软区域像素清零。
8. **匹配成功后的定位精度：** 128 B ±32 px 时 RMSE=0.500 px，median=0.338 px，P95=0.968 px。
9. **RANSAC 系统效果：** 以全部尝试复用的 GCP 为分母，最佳全局 GVR=0.081（64 B 方向/L1, affine, 阈值 3 px），保留错误匹配 3 个。
10. **推荐存储预算：** 首轮 Pareto 需同时看候选覆盖率；当前精度优先点为 64 B 方向/L1，最小存储点为 32 B Rank/Hamming。
11. **Storage-Reuse Pareto：** 见图 01-03、09 和 `storage_projection.csv`；不会只用视觉质量或压缩倍数选方法。
12. **新体系是否更符合工程价值？** 是。它把候选生成失败、身份识别失败、局部定位失败和场景几何验证拆开，旧 `Recall <= 2 px` 只保留为 `Localization Success @ 2 px`。

## 文件索引

- `reuse_pair_results.csv`：逐 GCP、逐方法、逐候选池的完整排名与 margin。
- `reuse_summary.csv`：Reuse@K、MRR、rank、coverage 汇总。
- `conditional_localization_results.csv`：仅身份识别正确后的局部精化。
- `ransac_results.csv`：独立匹配完成后的 translation/affine/homography 几何验证。
- `outcome_abcd_summary.csv`：A/B/C/D 结果分类。
- `failure_cases/`：Top-5、错误、复杂背景与跨年份案例。

## 解释限制

候选池规模是全景固定排序的前缀。若真实对应在池中不存在，记为 Type D，而不是把 GT 候选补入池中。`similar_building_like` 和 `repeated_road_like` 是自动上下文启发式标签，不冒充人工语义真值。
