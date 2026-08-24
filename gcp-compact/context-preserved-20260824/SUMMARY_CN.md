# GCP Compact Representation：Context-Preserved 真实实验总结

## 结论摘要

本轮重新制作了 **849** 个 256x256 GCP，正式 held-out 高置信评测使用 **755** 对。所有候选中心经过上海行政边界与软区域排除，但所有 patch 均保留完整影像上下文；`patch_mask_applied=0`，逐像素保存一致性为 100.0%。
其中 **640** 个 patch 的 soft-context ratio >=30%，**397** 个 >=50%。这批数据不是把水体或植被擦掉后的净化切片。

## 数据与制作方法

- 数据来源：上海多年 Sentinel-2 L2A B04 10 m 正射影像与 SCL；Overture 上海全量水体、land cover、land use 仅用于候选中心语义约束。
- 时间隔离：reference 年份为 2024；2021, 2022, 2023 仅用于候选稳定性和可选模型训练；每瓦片采用 reference 之后最新可用的 2025, 2026 时相作为完全 held-out target。
- 候选检测：SIFT、ORB、AKAZE、BRISK、KAZE、FAST、AGAST、Shi-Tomasi、Harris、MSER、Blob、Laplacian、Hough intersection 共 13 类；20 px 单方法 NMS 后再做跨方法融合。
- 本轮从 13,709,047 个原始响应得到 2,309,513 个单方法 NMS 候选和 287,105 个融合中心；行政边界过滤后候选 169,435 个，正式扫描 11,448 个，形成 958 个稳定池，最终均衡抽取 849 个。候选总库没有 10,000 点上限。
- 空间范围：仅保留上海市 OSM relation 913067 行政边界内中心；正式入选瓦片为 51RUQ, 51RVQ, 51SUR, 51SVR。边界只判断中心，不修改 patch。
- 中心过滤：若中心落在 SCL water/vegetation/cloud 等类别或 Overture soft geometry 中则拒绝。
- 切片制作：通过经纬度转换到影像 CRS 与像素坐标，从原始正射 B04 直接裁 256x256；没有 64x64 放大，没有对 patch 应用 soft mask。另存 uint16 带地理参考 GeoTIFF 与完整 8-bit PNG。
- target：按同一地理位置从 held-out 时相裁 384x384 搜索窗，对应最大 +/-64 px 搜索。
- GT：由 held-out target 上的灰度、Scharr 与 edge 相关面共识得到，并做 3x3 亚像素精化；它是实验定位标注而非外业测量真值。本文的 px 指 Sentinel-2 10 m 原始像素，不能据此直接声称亚米级绝对地理精度。

## 表示、存储与匹配

- 表示方法：Sobel/Scharr 幅值、Sobel 方向、幅值+方向联合码、edge occupancy、median threshold、5x5 rank transform、LBP、3x3/5x5/sparse/multiscale Census、DCT sign、Haar sign，以及中心 Gaussian/ring 权重。
- 真低比特存储：每个量化码按其 1/2/3/4 bit 位宽连续写入 bitstream，不用 `uint8` 冒充低比特。固定数据库 schema 不产生每点 header；若未来采用自适应布局，其 mask/index/header 必须计入 Bytes/GCP。
- Hamming：对等长 bitstream/量化码逐位 XOR 后 popcount；它是距离函数，不是特征提取器。多 bit 表示同时比较 packed Hamming、L1、L2、cosine。
- 搜索：target 内 1 px 步长密集滑窗；每个候选窗直接生成同构 compressed spatial code 后比较，不恢复图像，也不使用 GT 缩小候选范围。
- 亚像素：先保存整数最优结果，再在其 3x3 cost surface 上做分离抛物线插值；Hamming 平台导致分母接近零时返回整数位置，不用 GT 破除并列。
- 场景几何：按 target scene 汇总 GCP，比较 translation、affine、homography RANSAC，阈值 0.5/1/2/3 px。

## 关键结果

- Raw NCC：`raw_ncc`：65536 B/GCP，Recall<=2 px=1.000，RMSE=0.079 px，P95=0.109 px。
- 约 32 B Pareto 候选：`rank_transform_g16_b1_median_uniform_hamming_r64`：32 B/GCP，Recall<=2 px=0.919，RMSE=11.327 px，P95=12.214 px。
- 约 64 B Pareto 候选：`sobel_orientation_g16_b2_uniform_uniform_l1_r64`：64 B/GCP，Recall<=2 px=0.922，RMSE=1.295 px，P95=2.251 px。
- 约 128 B Pareto 候选：`dct_sign_g32_b1_median_uniform_hamming_r64`：128 B/GCP，Recall<=2 px=0.999，RMSE=0.471 px，P95=0.883 px。
- 约 256 B Pareto 候选：`grid_mag_orientation_g32_m1_o1_e0_percentile_hamming_r64`：256 B/GCP，Recall<=2 px=1.000，RMSE=0.336 px，P95=0.651 px。
- 最佳 Census：`census3_g16_b1_median_uniform_hamming_r64`：256 B/GCP，Recall<=2 px=0.964，RMSE=8.077 px，P95=1.394 px。
- 最佳 Sobel/Scharr：`sobel_mag_g32_b2_percentile_uniform_hamming_r64`：256 B/GCP，Recall<=2 px=0.991，RMSE=0.554 px，P95=1.172 px。
- 最佳中心权重配置相对同预算 uniform 的 Recall<=2 px 变化：+0.0000。
- 最佳搜索半径行：`dct_sign_g32_b1_median_uniform_hamming_r32`，Recall<=2 px=0.999。
- 64 B 场景几何：translation / 2.0 px，RANSAC inlier ratio=0.927，校正后 RMSE=0.102 px；仅统计每景至少 10 个 GCP 的场景。
- 深度空间码：最佳深度空间码为 `tiny_spatial_encoder_16x16x8_256B_hamming_r64`：256 B/GCP，Recall<=2 px=0.770，RMSE=16.246 px，P95=7.235 px，共享模型 0.01 MiB；同一空间隔离测试子集、相近字节级的最佳非学习方法为 `grid_mag_orientation_g32_m1_o1_e0_percentile_hamming_r64`：256 B/GCP，Recall<=2 px=1.000，RMSE=0.372 px，P95=0.779 px，深度方法 Recall<=2 px 差值 -0.2302。模型参数是数据库共享开销，不计入单个 GCP payload，但已单独报告。

## 十二个核心问题回答

1. 新重新制作了多少 GCP？849 个；其中 755 个进入高置信 held-out 正式评测。
2. 有多少 patch 包含明显水体/绿化？组合 soft ratio >=30% 的有 640 个；>=50% 的有 397 个。
3. soft-context ratio 分布是什么？中位数 0.473；中心到 soft 区域边界分组为 Group A (>100m)=89, Group B (30-100m)=398, Group C (<30m)=362。详见 `soft_context_statistics.csv` 与 `soft_boundary_statistics.csv`。
4. 64 B 表现如何？`sobel_orientation_g16_b2_uniform_uniform_l1_r64`：64 B/GCP，Recall<=2 px=0.922，RMSE=1.295 px，P95=2.251 px。
5. 与旧 clean 数据下降多少？`rank_transform_g16_b1_median_uniform_hamming_r64` 的 Recall<=2 px：A=0.910，B=0.919，B-A=+0.009；`sobel_orientation_g16_b2_uniform_uniform_l1_r64` 的 Recall<=2 px：A=0.873，B=0.922，B-A=+0.049；`dct_sign_g32_b1_median_uniform_hamming_r64` 的 Recall<=2 px：A=1.000，B=0.999，B-A=-0.001；`sobel_mag_g32_b2_percentile_uniform_hamming_r64` 的 Recall<=2 px：A=0.993，B=0.991，B-A=-0.003；`raw_ncc` 的 Recall<=2 px：A=1.000，B=1.000，B-A=+0.000。Dataset A 存在多时相强筛选泄漏，只能作为 easy/legacy 对照。
6. 哪种 representation 对复杂背景最稳？以全体结果为准，目前最佳 64 B 家族是 `sobel_orientation`；分组见 `soft_context_matching_results.csv`。
7. Census 是否比 Sobel 更适合复杂上下文？最佳 Census 的 Recall<=2 px 为 0.964，最佳 Sobel/Scharr 为 0.991；结论应结合 soft ratio 分组而不是只看总体。
8. 中心加权是否有效？同预算最佳变化为 +0.0000 Recall；详见 `weighting_results.csv`。
9. RANSAC 能否消除复杂背景错误？64 B 最佳配置拒绝 catastrophic matches=3，场景成功率=1.000；小于 10 点的场景不参与此结论。
10. 是否存在明确 64/128/256 B Pareto 点？已输出候选及 `pareto.csv`；是否称为稳定 Pareto 点应以不同场景复测后决定。
11. 深度模型是否超过非学习方法？最佳深度空间码为 `tiny_spatial_encoder_16x16x8_256B_hamming_r64`：256 B/GCP，Recall<=2 px=0.770，RMSE=16.246 px，P95=7.235 px，共享模型 0.01 MiB；同一空间隔离测试子集、相近字节级的最佳非学习方法为 `grid_mag_orientation_g32_m1_o1_e0_percentile_hamming_r64`：256 B/GCP，Recall<=2 px=1.000，RMSE=0.372 px，P95=0.779 px，深度方法 Recall<=2 px 差值 -0.2302。模型参数是数据库共享开销，不计入单个 GCP payload，但已单独报告。
12. 是否具备形成论文的证据？具备数据构造、复杂背景分层、真实 bitstream、直接匹配、场景几何和失败案例的完整实验链；在旧 clean 同协议复测和独立高分辨率传感器外部验证完成前，证据属于有力但尚非最终定论。

## 输出索引

- `new_gcp_manifest.csv`：849 个 GCP 完整元数据。
- `dataset_integrity_audit.json`：独立审计，全部强制检查通过=True。
- `patch_quality_report.csv` / `soft_context_statistics.csv` / `soft_boundary_statistics.csv` / `soft_boundary_matching_results.csv`：数据质量、软背景比例、边界距离分组及其匹配敏感性。
- `baseline_results.csv` / `spatial_results.csv` / `grid_ablation_results.csv` / `census_results.csv` / `distance_results.csv`：主实验与 72 组全网格消融。
- `search_radius_results.csv` / `weighting_results.csv` / `allocation_results.csv`：搜索、中心权重与固定预算空间分配消融。
- `deep_results.csv` / `deep_test_subset_nonlearning_results.csv`：空间隔离深度实验及同子集公平对照。
- `legacy_comparison.csv`：Dataset A（旧 strong-selection）与 Dataset B（context-preserved）的受限对照。
- `ransac_results.csv` / `ransac_scene_summary.csv`：逐场景及跨有效场景汇总的几何评测。
- `pareto.csv` / `plots/`：论文图表与 Pareto。
- `failure_cases/`：最差 20 个完整失败案例及 score surface。
