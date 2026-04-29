# hi3559 调试

## 系统信息查看

```bash
# 查看芯片信息
cat /proc/umap/sys
cat /proc/umap/media

# 查看 CPU 占用率（每秒刷新）
cat /proc/cpuinfo
top

# 查看内存使用
cat /proc/meminfo
free -m

# 查看内核日志
dmesg | tail -100
dmesg | grep -i error
```

## MMZ 调试

MMZ（Media Memory Zone）是海思平台为多媒体处理预留的专用内存区域，是 MPP 各模块运行的基础。

```bash
# 查看 MMZ 总体使用情况
cat /proc/media-mem

# 输出示例：
# ---MMZ Use Info---
# MMZ Size:       512MB
# MMZ Free:       256MB
# Block Count:     32
# ...

# 查看 MMZ 详细分配信息
cat /proc/umap/mmz

# 查看内核 MMZ 参数
cat /proc/cmdline | grep mmz

# 典型 MMZ 内核启动参数格式
# mmz=ddr2:0x88000000,512M,aw=256M:ddr3:0x80000000,512M
```

### MMZ 常见问题

- **MMZ 内存不足**：调整启动参数中 MMZ 大小，或减少 VB 缓存池数量/大小
- **MMZ 分配失败**：检查是否有内存泄漏，确认 VB 池配置合理
- **双核 MMZ**：Hi3559 支持双 DDR，需确保 MMZ 参数正确映射到对应 DDR 区域

## VB（视频缓存池）调试

VB 是 MPP 系统中各模块间数据交换的核心缓冲区。

```bash
# 查看 VB 使用状态
cat /proc/umap/vb

# 查看 VB 缓存池配置和占用
cat /proc/umap/vbq

# 输出包含：
# - 池ID、大小、块数
# - 已分配/空闲块数
# - 各模块的占用情况

# 查看 VB 用户态映射信息
cat /proc/umap/vb_sup

# 通过 MPI 接口获取 VB 信息
# VB_CONFIG_S 结构体中记录了所有缓存池配置
```

### VB 常见问题排查

| 现象 | 可能原因 | 排查方法 |
| ------- | ------------ | -------------------------------------- |
| VB 获取失败 | 缓存池太小或数量不够 | 检查 VB 配置是否匹配实际分辨率 |
| VB 释放不了 | 模块未正确释放绑定 | 检查 bind 关系，确认解绑流程 |
| VB 内存泄漏 | 持续申请未释放 | 多次执行 `cat /proc/umap/vb` 对比变化 |

## MPP 各模块调试

### VI（视频输入）

```bash
# 查看 VI 模块状态
cat /proc/umap/vi

# 查看 VI 通道信息
cat /proc/umap/vi_chn

# 输出包含：
# - VI 设备号、通道号
# - 输入分辨率、像素格式
# - 帧率、绑定关系
# - 帧计数（可判断是否断流）

# 常见排查
# 1. 无图像 → 检查 sensor 是否出帧: cat /proc/umap/vi 看帧计数
# 2. 图像异常 → 检查 VI PIPE 配置和 WDR 模式
```

### VENC（视频编码）

```bash
# 查看 VENC 通道状态
cat /proc/umap/venc

# 查看 VENC 通道详细信息
cat /proc/umap/venc_chn

# 输出包含：
# - 通道号、编码类型（H264/H265/JPEG）
# - 编码分辨率、帧率、码率
# - 目标码率 / 实际码率
# - 编码帧数 / 跳帧数
# - QP 值分布

# 查看码率控制信息
cat /proc/umap/rc

# 强制产生一个 I 帧（调试用）
# HI_MPI_VENC_RequestIDR(VencChn)
```

### VPSS（视频处理子系统）

```bash
# 查看 VPSS 组信息
cat /proc/umap/vpss

# 查看 VPSS 通道信息
cat /proc/umap/vpss_chn

# 输出包含：
# - 组号、通道号
# - 输入源分辨率
# - 输出分辨率、裁剪信息
# - 帧计数（判断通道是否在工作）
# - 绑定关系
```

### VDEC（视频解码）

```bash
# 查看 VDEC 通道状态
cat /proc/umap/vdec

# 查看 VDEC 通道详情
cat /proc/umap/vdec_chn

# 输出包含：
# - 通道号、解码类型
# - 解码分辨率
# - 已解码帧数 / 解码错误数
# - 码流缓冲区使用情况
```

### VO（视频输出）

```bash
# 查看 VO 模块状态
cat /proc/umap/vo

# 查看 VO 通道/图层信息
cat /proc/umap/vo_chn
cat /proc/umap/vo_layer

# 输出包含：
# - VO 设备、图层、通道号
# - 输出分辨率、显示区域
# - 帧计数
# - 同步信息
```

### AUDIO（音频）

```bash
# 查看 AI（音频输入）状态
cat /proc/umap/ai

# 查看 AO（音频输出）状态
cat /proc/umap/ao

# 查看 AENC（音频编码）/ ADEC（音频解码）
cat /proc/umap/aenc
cat /proc/umap/adec

# 输出包含：
# - 通道号、采样率、位宽
# - 编码格式（G711/G726/AAC）
# - 帧计数
# - 缓冲区状态
```

## BIND（绑定关系）查看

```bash
# 查看所有模块绑定关系
cat /proc/umap/bind

# 输出格式：
# SrcMod[VI].SrcDev[0].SrcChn[0] ---> DstMod[VPSS].DstDev[0].DstChn[0]
# 用于梳理数据流向，排查数据通路是否正确
```

## ISP 调试

```bash
# 查看 ISP 运行状态
cat /proc/umap/isp

# 查看 ISP 帧信息
cat /proc/umap/isp_frame

# AE（自动曝光）信息
cat /proc/umap/ae

# AWB（自动白平衡）信息
cat /proc/umap/awb

# ISP 调试命令行工具
isp_ctrl --help
```

## 系统级调试手段

### 中断与寄存器

```bash
# 查看中断统计
cat /proc/interrupts

# 查看寄存器（需对应的 debug 节点）
cat /proc/umap/vi_reg
cat /proc/umap/venc_reg
```

### 性能分析

```bash
# 查看各模块性能统计
cat /proc/umap/perf

# 查看系统负载
cat /proc/loadavg
uptime
```

### 帧率统计

```bash
# 快速统计 VI 帧率（每秒读帧计数差值）
watch -n 1 "cat /proc/umap/vi | grep 'frame'"
```

## 常见调试流程

### 1. 无图像输出

```text
检查 sensor 出帧 → cat /proc/umap/vi 看帧计数
         ↓
检查 VI 配置 → cat /proc/umap/vi 确认分辨率/格式
         ↓
检查 BIND → cat /proc/umap/bind 确认绑定关系
         ↓
检查 VPSS → cat /proc/umap/vpss 看通道帧计数
         ↓
检查 VENC → cat /proc/umap/venc 看编码帧计数
         ↓
检查 VB → cat /proc/umap/vb 确认缓存池充足
```

### 2. 编码异常（花屏/卡顿）

```text
检查码率 → cat /proc/umap/venc 看实际码率 vs 目标码率
      ↓
检查 VB → cat /proc/umap/vb 看是否有缓存池耗尽
      ↓
检查 MMZ → cat /proc/media-mem 看 MMZ 剩余
      ↓
检查带宽 → 确认 VI+VPSS+VENC 总带宽是否超限
```

### 3. 内存泄漏排查

```text
记录初始状态 → cat /proc/umap/vb > /tmp/vb_before.txt
        ↓
运行一段时间
        ↓
记录结束状态 → cat /proc/umap/vb > /tmp/vb_after.txt
        ↓
对比差异 → diff /tmp/vb_before.txt /tmp/vb_after.txt
        ↓
定位泄漏模块 → 根据增长的缓存池找到对应模块
```