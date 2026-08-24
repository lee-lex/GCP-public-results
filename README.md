# GCP 公开成果图

这个仓库只保存可公开查看的实验图片和简要结果，不包含原始遥感数据、模型或内部代码。

## 纪松等（2022）轻量化影像控制点方法复现

上海全量实验使用 12,804 个候选点，其中 12,227 个点形成 31,827 条有效多时相测量。

### 压缩与匹配质量

[点击查看原始分辨率图片](ji-song-2022/compression_and_quality_cn.png)

![压缩倍数、解码匹配质量和候选检索召回率](ji-song-2022/compression_and_quality_cn.png)

### 可解码切片对比

[点击查看原始分辨率图片](ji-song-2022/chip_examples_cn.png)

![64x64 基准、50x50 原始切片及 WebP q80/q30 解码结果](ji-song-2022/chip_examples_cn.png)

### 主要结果

| 方法 | 全量估算 | 模型摊销后压缩倍数 | 全时相保留率 | 中位位置变化 |
|---|---:|---:|---:|---:|
| RAW50 + 128 bit | 29.63 MiB | 393.53x | 94.70% | 0 m |
| PNG50 + 128 bit | 26.45 MiB | 440.85x | 94.70% | 0 m |
| WebP q80 + 128 bit | 13.40 MiB | 869.91x | 94.41% | 0.034 m |
| WebP q30 + 128 bit | 7.73 MiB | 1509.01x | 92.84% | 0.100 m |

质量优先推荐 WebP q80。128 bit 哈希只作为候选检索索引，最终精匹配仍使用可解码图像切片。

## 空间结构保持型轻量表示（非学习版）

实现了不恢复原图的低比特二维结构编码和压缩域滑窗匹配，并完成 72 组消融与 10 个传统/编解码基线。

- [查看完整中文实验页和全部图表](gcp-compact/README.md)
- [查看 Pareto 前沿原图](gcp-compact/pareto_frontier.png)
- [下载消融汇总 CSV](gcp-compact/ablation_summary.csv)

## 历史 GCP 整景可复用性评估

500 个固定 2024 GCP 在 2025/2026 完整 Sentinel-2 目标场景中进行无 GT 候选注入的重识别。首轮结果把候选覆盖、身份识别、条件定位和 RANSAC 几何验证分开统计。

- [手机查看完整中文结果、9 张图和 140 个案例](gcp-compact/reusability-20260824/README.md)
- [直接打开候选池规模与 Reuse@1 图](gcp-compact/reusability-20260824/plots/04_candidate_pool_size_vs_reuse_at_1.png)
- [浏览成功与失败案例图集](gcp-compact/reusability-20260824/FAILURE_GALLERY.md)
