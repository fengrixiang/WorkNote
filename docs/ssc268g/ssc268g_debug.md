# SSC268g 调试指南

## 系统信息查看

```bash
# 查看 CPU 信息
cat /proc/cpuinfo

# 查看 CPU 占用率
top

# 查看内存使用
cat /proc/meminfo
free -m

# 查看内核版本
uname -a
cat /proc/version

# 查看内核启动参数
cat /proc/cmdline

# 查看内核日志
dmesg | tail -100
dmesg | grep -i error

# 查看系统运行时间与负载
uptime
cat /proc/loadavg
```

## MI 系统调试（Media Interface）

SSC268g 的多媒体子系统称为 MI（Media Interface），类似于海思的 MPP，各模块通过 BIND 串联形成数据通路。

### MI 系统状态

```bash
# 查看 MI 系统总体信息
cat /proc/mi_modules/mi_sys

# 查看内存使用情况
cat /proc/mi_modules/mi_sys/mem_info

# 查看各模块绑定关系
cat /proc/mi_modules/mi_sys/bind
```

### SYS 常见问题

| 现象 | 可能原因 | 排查方法 |
| --- | --- | --- |
| MI 初始化失败 | 驱动未加载或内核配置错误 | `dmesg \| grep mi` 查看加载日志 |
| 内存分配失败 | MMZ 大小不足 | 检查 bootargs 中 mmz 参数配置 |
| BIND 失败 | 源/目标端口参数不匹配 | 确认分辨率、像素格式一致 |

## VI（视频输入）调试

```bash
# 查看 VI 设备信息
cat /proc/mi_modules/mi_vpe

# 查看 VI 通道帧率与分辨率
cat /proc/mi_modules/mi_vpe/pe0

# 查看前端 Sensor 连接状态
cat /proc/mi_modules/mi_sensor

# 输出包含：
# - 设备号、通道号
# - 输入分辨率、像素格式
# - 帧率、帧计数
# - WDR 模式
```

### VI 常见问题

- **无图像**：检查 Sensor 是否出帧 → 查看帧计数是否增长
- **图像偏色/异常**：检查 Sensor 输出格式与 VI 配置是否匹配（YUV422/YUYV/RAW等）
- **帧率不稳定**：检查 Sensor 曝光时间与 VI 帧率配置是否冲突
- **画面撕裂**：检查 MIPI/LVDS 接口时序配置

## VENC（视频编码）调试

```bash
# 查看 VENC 通道状态
cat /proc/mi_modules/mi_venc

# 查看指定通道编码信息
cat /proc/mi_modules/mi_venc/chn0

# 输出包含：
# - 通道号、编码类型（H264/H265/JPEG）
# - 编码分辨率、帧率
# - 目标码率 / 实际码率
# - QP 值
# - 编码帧数 / 跳帧数
```

### VENC 码率控制

```bash
# 查看码率控制参数
# CBR（固定码率）：码率稳定，画质波动
# VBR（可变码率）：画质稳定，码率波动
# AVBR（自适应码率）：根据场景复杂度自动调整

# 调整码率（示例，具体接口参考 MI API）
# MI_VENC_SetRcParam()
```

### VENC 常见问题

| 现象 | 可能原因 | 排查方法 |
| --- | --- | --- |
| 编码无输出 | VB 缓存池不足或 BIND 断开 | 检查 VB 池配置和绑定关系 |
| 画面模糊 | 码率设置过低 | 提高目标码率或降低分辨率 |
| 画面卡顿 | 编码性能不足或码率波动大 | 降低帧率/分辨率，或切换为 CBR 模式 |
| 花屏 | 参考帧丢失或码流损坏 | 检查 I 帧间隔和 DPB 配置 |

## VPSS（视频处理子系统）调试

```bash
# 查看 VPSS 组和通道信息
cat /proc/mi_modules/mi_divp

# 查看指定通道详情
cat /proc/mi_modules/mi_divp/chn0

# 输出包含：
# - 组号、通道号
# - 输入源分辨率
# - 输出分辨率、裁剪区域
# - 帧计数
```

### VPSS 常见问题

- **通道无输出**：检查 BIND 关系是否正确建立
- **缩放异常**：确认输入输出分辨率在硬件支持的缩放范围内
- **性能瓶颈**：减少同时启用的 VPSS 通道数量

## VB（视频缓存池）调试

VB 是 MI 系统中各模块间数据交换的核心缓冲区。

```bash
# 查看 VB 使用状态
cat /proc/mi_modules/mi_sys/vb

# 查看缓存池详细分配
cat /proc/mi_modules/mi_sys/vb_poolinfo

# 输出包含：
# - 池 ID、块大小、块数
# - 已使用 / 空闲块数
# - 各模块占用情况
```

### VB 常见问题

| 现象 | 可能原因 | 排查方法 |
| --- | --- | --- |
| VB 获取失败 | 缓存池太小或数量不够 | 增大 VB 池大小或数量 |
| VB 释放不了 | 模块未正确释放绑定 | 检查 bind 关系和解绑流程 |
| VB 内存泄漏 | 持续申请未释放 | 多次读取 VB 信息对比变化 |

## Sensor 调试

```bash
# 查看 Sensor 设备信息
cat /proc/mi_modules/mi_sensor

# I2C 读写 Sensor 寄存器（通过 i2ctools）
i2ctransfer -y -a <bus> w2@<addr> <reg_h> <reg_l>
i2ctransfer -y -a <bus> w2@<addr> <reg_h> <reg_l> r2

# 示例：读取 Sensor chip ID（地址 0x300A）
i2ctransfer -y -a 0 w2@0x3c 0x30 0x0A r2
```

### Sensor 常见问题

- **Sensor 不出帧**：检查 I2C 通信、MCLK 供电、复位时序
- **曝光异常**：检查 AE 配置，确认 Sensor 曝光寄存器是否可写
- **帧率不对**：检查 Sensor PLL 配置和 VTS/HTS 寄存器

## 音频调试

```bash
# 查看 AI（音频输入）状态
cat /proc/mi_modules/mi_ai

# 查看 AO（音频输出）状态
cat /proc/mi_modules/mi_ao

# 查看 AENC（音频编码）状态
cat /proc/mi_modules/mi_aenc

# 输出包含：
# - 通道号、采样率、位宽
# - 编码格式（G711/G726/AAC）
# - 帧计数、缓冲区状态
```

### 音频常见问题

- **无声**：检查 AI/AO 绑定关系，确认采样率和位宽一致
- **噪音**：检查 I2S/PCM 时序配置，确认主从模式匹配
- **回声**：检查 AEC（回声消除）是否启用及参数配置

## ISP 调试

```bash
# 查看 ISP 运行状态
cat /proc/mi_modules/mi_isp

# AE（自动曝光）信息
cat /proc/mi_modules/mi_ae

# AWB（自动白平衡）信息
cat /proc/mi_modules/mi_awb

# 使用 IQ 调试工具（需连接 PC 端 IQ Tool）
# 通过串口或网络连接，可实时调节 AE/AWB/AF 等参数
```

## 接口调试

### MIPI / LVDS 接口

```bash
# 查看 MIPI 接收状态
cat /proc/mi_modules/mi_vif

# 检查 MIPI 时序错误计数
cat /proc/mi_modules/mi_vif/mipi0
```

### 常见 MIPI 问题

- **MIPI 无数据**：检查 Lane 数、速率配置是否与 Sensor 一致
- **MIPI 校验错误**：检查 LP/HS 信号电平、走线长度匹配
- **MIPI 帧同步丢失**：检查 Sensor 输出时序，确认帧起始/结束信号正确

## 中断与寄存器

```bash
# 查看中断统计
cat /proc/interrupts

# 查看寄存器（需对应 debug 节点）
cat /proc/mi_modules/mi_venc/reg
cat /proc/mi_modules/mi_vpe/reg
```

## 性能分析

```bash
# 查看系统负载
uptime
cat /proc/loadavg

# 查看各模块帧率（通过帧计数差值计算）
# VI 帧率
watch -n 1 "cat /proc/mi_modules/mi_vpe | grep 'frame'"

# VENC 帧率
watch -n 1 "cat /proc/mi_modules/mi_venc | grep 'frame'"
```

## 常见调试流程

### 1. 无图像输出

```text
检查 Sensor 出帧 → cat /proc/mi_modules/mi_sensor 看帧计数
         ↓
检查 VIF 接口 → cat /proc/mi_modules/mi_vif 看接收状态
         ↓
检查 VPE/VI → cat /proc/mi_modules/mi_vpe 确认分辨率/格式
         ↓
检查 BIND → cat /proc/mi_modules/mi_sys/bind 确认绑定关系
         ↓
检查 DIVP → cat /proc/mi_modules/mi_divp 看通道帧计数
         ↓
检查 VENC → cat /proc/mi_modules/mi_venc 看编码帧计数
         ↓
检查 VB → cat /proc/mi_modules/mi_sys/vb 确认缓存池充足
```

### 2. 编码异常（花屏/卡顿）

```text
检查码率 → cat /proc/mi_modules/mi_venc 看实际码率 vs 目标码率
      ↓
检查 VB → cat /proc/mi_modules/mi_sys/vb 看是否有缓存池耗尽
      ↓
检查内存 → cat /proc/mi_modules/mi_sys/mem_info 看 MMZ 剩余
      ↓
检查带宽 → 确认 VIF+VPE+VENC 总带宽是否超限
```

### 3. 内存泄漏排查

```text
记录初始状态 → cat /proc/mi_modules/mi_sys/vb > /tmp/vb_before.txt
        ↓
运行一段时间
        ↓
记录结束状态 → cat /proc/mi_modules/mi_sys/vb > /tmp/vb_after.txt
        ↓
对比差异 → diff /tmp/vb_before.txt /tmp/vb_after.txt
        ↓
定位泄漏模块 → 根据增长的缓存池找到对应模块
```

### 4. Sensor 不出帧排查

```text
检查硬件 → 确认供电、MCLK、复位信号正常
       ↓
检查 I2C → i2ctransfer 读写 Sensor 寄存器，确认通信正常
       ↓
检查 Sensor 配置 → 确认初始化序列已正确写入
       ↓
检查 MIPI 配置 → 确认 Lane 数/速率与 Sensor 输出匹配
       ↓
检查 VIF → cat /proc/mi_modules/mi_vif 看是否有接收错误
```
