# GF-7 L1A 紧凑 GCP 几何定位实验

这不是 Sentinel-2 正射影像自匹配。目标影像为带原始 RPC、`UsedGCPNo=0` 的 GF-7 BWDPAN L1A。

## 手机直接查看

- [存储量与跨传感器复用](plots/01_storage_vs_cross_sensor_reuse.png)
- [各方法匹配成功率](plots/02_method_match_success.png)
- [单点误差 CDF](plots/03_error_cdf.png)
- [RANSAC 几何验证](plots/04_ransac_gvr.png)
- [独立 ICP 校正前后](plots/05_icp_before_after.png)
- [存储量与最终 ICP 精度](plots/06_storage_vs_icp.png)
- [GCP 数量敏感性](plots/07_gcp_quantity_sensitivity.png)
- [GF-7 上 GCP/ICP 数字分布图](plots/08_scene_gcp_icp_distribution.png)
- [尺度与旋转归一化消融](plots/09_geometry_normalization_ablation.png)
- [RPC 高程敏感性](plots/10_elevation_sensitivity.png)
- [独立真值抽查图](plots/independent_gt_contact_sheet.jpg)

完整中文结论见 [GF7_GCP_SUMMARY_CN.md](GF7_GCP_SUMMARY_CN.md)。失败案例保留在 [failure_cases](failure_cases/)，汇总表在 [tables](tables/)。
