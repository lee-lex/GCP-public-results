# 数据、控制点、切片与匹配方法说明

## 1. 先区分三个概念

这套工程包含三个不同层次，不能把它们都称为“特征匹配”：

1. **候选控制点提取**：决定控制点中心坐标在哪里。
2. **控制点切片制作**：以确定的中心坐标从参考影像和目标影像读取局部窗口。
3. **切片表示与匹配**：把参考切片压成空间结构码，并在目标搜索窗口中寻找同一点。

`gcp_compact` 当前实现的是第 3 层。它不会在整景影像里重新检测控制点，也不会修改控制点经纬度；第 1 层产生的中心坐标是它的输入先验。

## 2. 当前实际数据来源

| 数据 | 当前路径 | 当前用途 | 是否进入本次 64 B 实验 |
| --- | --- | --- | --- |
| Sentinel-2 L2A B04 10 m 与 SCL | `D:\projects\260701\cache\sentinel2_b04_scl` | 上海候选点提取、多时相稳定性筛选、历史 64×64 切片 | 否，本次只跑了合成烟雾数据 |
| 上海多年 Sentinel-2 | `N:\Sentinel-2\OIMGShangHai` | 后续真实多时相验证的完整影像库 | 否 |
| 低精度已有 GCP | `N:\Sentinel-2\GCPs` | 需要时提供粗定位，不作为亚米级真值 | 否 |
| Overture 上海全量 Parquet | `N:\260701_resources\Overture\overture_shanghai_full_20260805` | 建筑、道路连接点/线段等硬质候选先验；剔除水域和软地表 | 否 |
| GF-7 L1A | `H:\gf` | 独立的平差、正射和高分辨率控制点工作流 | 否 |

因此，已经公开的 `64 B / Recall≤2px=1.00` 是 4 组确定性合成平移实验，用来验证代码链路，不是上海 Sentinel-2、GF-7 或 Overture 的真实精度结论。

## 3. 现有上海候选控制点如何提取

现有的 12,804 个上海稳定点来自 Sentinel-2 多方法融合表：

```text
D:\260701\sentinel2_control_point_method_benchmark\final_method_comparison\tables\
sentinel2_final_multimethod_stable_control_points.csv
```

### 3.1 参考影像选择

- 数据使用 Sentinel-2 L2A 的 B04 10 m 红波段。
- 每个 MGRS 瓦片优先选择最近 16 景中 SCL 清晰比例不低于 0.65 的最新影像。
- 如果没有达到 0.65，则选择候选景中清晰比例最高、时间尽可能新的影像。
- SCL 类别 `2/4/5/6/7` 作为可用像素；云、云影等位置不产生候选点。

### 3.2 影像预处理

- B04 在局部块内按 2% 和 98% 分位拉伸到 `uint8`。
- 候选提取脚本再应用 CLAHE，增强局部硬边和角点。
- 全域按 `1024×1024` 块处理，相邻块重叠 64 像素。

### 3.3 候选点检测器

当前不是只用 SIFT，而是同时比较并融合 13 种方法：

```text
SIFT, ORB, AKAZE, BRISK, KAZE, FAST, AGAST,
Shi-Tomasi, Harris, MSER, Blob, Laplacian extrema,
Hough line intersection
```

每个块的 `max_block_keypoints=4000` 只是单块计算保护，不是全上海候选点总数上限。每种方法在每个 MGRS 瓦片内按响应值排序，并做 20 像素空间非极大值抑制；不同检测器在 20 像素内命中的点再融合为一个候选点，同时保留 `source_methods`。

### 3.4 多时相稳定筛选

候选点在多个 Sentinel-2 时相中做局部模板匹配，统计有效年份数、NCC、峰值边际和位置漂移。最终表只保留：

- `stability_status = trial_stable`；
- `passes_1m_error = True`；
- 再按 120 m 最小距离做空间去重。

候选点综合分数为：

```text
0.34 × stability_score
+ 0.24 × median_ncc
+ 0.17 × normalized_peak_margin
+ 0.15 × normalized_valid_year_count
+ 0.10 × normalized_inverse_p90_shift
```

最终得到 12,804 个稳定点。这里的“1 m 内”来自 Sentinel-2 多时相匹配位置的内部一致性筛选，不等同于吉林一号或测量真值给出的绝对平面误差。

## 4. Overture 候选点是什么

Overture 是另一条候选来源，目前不应与上述 12,804 个 `s2_fused_*` 点混为一批。

当前 Overture 脚本：

- 读取 `building / connector / segment`；
- 以要素包围盒中心生成硬质候选中心；
- 使用真实 WKB 几何剔除 `water`；
- 剔除 forest、shrub、grass、wetland、crop 等软地表；
- 剔除 park、garden、meadow、farmland 等不宜长期作为硬质控制点的 land-use；
- 线状软几何默认缓冲 16 m；
- 输出 EPSG:32651 和 WGS84 坐标，不设置任意总点数上限。

这些点后续还需要在正射光学影像上吸附到明确的角点或硬边，并通过多时相稳定性验证，才能成为最终 GCP。`gcp_compact` 本次实验尚未读取这批 Overture 点。

## 5. 控制点切片如何制作

### 5.1 历史生产切片

现有压缩工程使用 `64×64` Sentinel-2 B04 灰度切片：

1. 从控制点表读取 `lon/lat` 和 `reference_b04`。
2. 将 WGS84 坐标转换到参考影像 CRS。
3. 用影像仿射变换换算到 `row/col`。
4. 以该像素为中心，读取 `64×64` 窗口。
5. 越界位置以 0 填充。
6. 按 2% 和 98% 分位拉伸成 `uint8`。

历史 WebP/JPEG2000 报告和 12,227 点压缩验证使用的是这套 64×64 切片。

### 5.2 新空间结构实验切片

本项目的新输入契约是 `256×256`：

- 参考切片：以稳定控制点为中心，从参考正射影像读取 `256×256`。
- 目标搜索窗口：在新时相同一粗略地理位置读取比 256 更大的窗口，例如搜索半径为 `r` 时读取 `(256+2r)×(256+2r)`。
- `pairs.csv` 中的 `gt_x/gt_y` 是控制点中心在目标搜索窗口内的像素坐标。
- RGB 会先转灰度；PNG、JPG、TIF 均支持。

当前 `gcp_compact` 不会自动从 12,804 点表批量生成这组 256×256 数据。真实实验必须从原始正射影像重新切 256×256，不能把 64×64 图像直接放大成 256×256。

## 6. 切片内部提取的不是稀疏特征点

参考中心确定后，`gcp_compact` 对整张切片提取密集结构图：

```text
Gx, Gy = Sobel(gray)
magnitude = sqrt(Gx² + Gy²)
orientation = atan2(Gy, Gx) mod π
edge = Canny 或 magnitude 阈值
```

随后池化成 `8×8 / 16×16 / 32×32` 二维网格。每个 cell 统计：

- 平均梯度幅值；
- 按梯度幅值加权的主方向；
- 边缘像素占有率。

这里保留的是固定二维位置上的结构统计，而不是一组无序 SIFT/ORB 描述子。因此匹配时第 `(i,j)` 个参考 cell 始终和候选窗口第 `(i,j)` 个 cell 比较。

## 7. 当前汉明匹配的完整方法名

当前默认方法应准确写成：

> **基于低比特二维空间结构码的局部密集滑窗模板匹配，使用精确汉明距离作为代价函数。**

它不是“只用了汉明距离”，也不是 ORB 的关键点描述子匹配。

### 7.1 搜索方式

- 搜索范围是输入的目标局部窗口或 `SearchBounds`。
- 默认步长为 1 像素。
- 每个合法左上角位置都生成与参考切片配置完全相同的空间结构码。
- 目标梯度图只计算一次；每个 cell 的统计通过积分图读取。
- 当前是精确穷举搜索，不是 FLANN/LSH 近似检索，也没有随机抽样候选位置。

### 7.2 汉明代价

参考码流为 `b_ref`，候选码流为 `b_xy`：

```text
D_H(x,y) = Σ popcount(b_ref XOR b_xy)
```

实现使用逐字节 XOR 和 256 项 popcount 查找表，直接比较真实 bitstream。距离越小越相似，最小值位置为 Top-1，按代价排序得到 Top-5。

这在检索意义上等价于对每个候选码使用 `BFMatcher + NORM_HAMMING` 做精确最近邻，但这里直接实现是为了保留规则滑窗的二维代价面，并用于后续亚像素插值。

### 7.3 其他距离

- `L1`：展开的只是量化 cell 码，不是原始图像；适合幅值等有序量化值。
- `cosine`：比较归一化量化向量的方向一致性。
- `Hamming`：速度快且可直接在打包码流上计算，但所有 bit 权重相同。

因此实验不能只报“汉明最好”。应在同一数据和相同字节数下比较 Hamming、L1、cosine，再由真实多时相定位误差选择。

### 7.4 亚像素位置

先得到整数最小代价位置，然后取其周围 `3×3` 代价面，在 x、y 方向分别做三点抛物线插值：

```text
delta = 0.5 × (left - right) / (left - 2×center + right)
```

最终输出候选左上角和控制点中心的亚像素坐标。若最优点位于搜索边界，则不进行 3×3 插值。

## 8. 对照基线使用什么匹配

| 基线 | 参考表示 | 匹配方式 |
| --- | --- | --- |
| raw grayscale | 原始 256×256 灰度图 | `cv2.matchTemplate`，NCC |
| WebP/JPEG2000 | 压缩码流 | 解码后 NCC，仅作为重建型对照 |
| ORB | ORB 关键点与二进制描述子 | BFMatcher Hamming、ratio test、RANSAC 单应/仿射 |
| SIFT | SIFT 关键点与浮点描述子 | BFMatcher L2、ratio test、RANSAC 单应/仿射 |
| median binary | 1-bit 中位数幅值网格 | 精确汉明滑窗匹配 |

ORB/SIFT 基线与主方法的区别是：它们匹配无序局部关键点集合；主方法比较固定二维网格中的整体空间结构。

## 9. 真实遥感实验还缺哪一步

代码功能已验证，但尚未完成下面这次真实数据实验：

```text
12,804 个稳定中心点
-> 从参考 Sentinel-2/GF-7 正射影像重新切 256×256
-> 从独立时相切局部目标搜索窗口
-> 建立带独立真值的 pairs.csv
-> 跑 72 组位宽消融和 Hamming/L1/cosine 对比
-> 再用吉林一号或其他独立正射影像验证绝对精度
```

吉林一号若自身地理定位不可靠，只能先作为独立影像做相对匹配验证；要声称绝对亚米级精度，仍需要可靠测量点、已知高精度正射产品或其他独立真值。

## 10. 对应代码

- 候选点多方法提取：`sentinel2_control_point_method_benchmark/extract_sentinel2_control_point_methods.py`
- 多时相评分与 120 m 去重：`sentinel2_control_point_method_benchmark/build_sentinel2_method_report.py`
- 历史 64×64 切片：`sentinel2_control_point_method_benchmark/export_recommended_control_point_chip_packages.py`
- Overture 硬质候选：`build_overture_geometry_soft_excluded_candidates.py`
- 空间结构编码：`gcp_compact/encode.py`
- 真 bit packing：`gcp_compact/bitpack.py`
- 滑窗与汉明匹配：`gcp_compact/match.py`
- 评测与消融：`gcp_compact/evaluate.py`、`gcp_compact/experiments.py`
