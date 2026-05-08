# Work Note

嵌入式开发技术文档，基于 Docsify 构建。

## 文档目录

### SoC 平台

| 平台 | 说明 |
| ---- | ---- |
| [安霸 CV72](/docs/cv72/cv72) | 视频编码、PTP 时间同步、Lua 管线配置 |
| [海思 Hi3559](/docs/hi3559/ch1.1) | U-Boot 源码分析、平台调试 |
| [瑞芯微 RK3588](/docs/rk3588/rk3588) | HDMI CEC、蓝牙、GPIO、IR 遥控、云台 |
| [瑞芯微 RK3576](/docs/rk3576/rk3576) | SDK 搭建、系统调试命令 |
| [晨星 SSC268G](/docs/ssc268g/ch1.1) | SSC268G 调试指南 |
| [ARM 平台](/docs/ARM/arm) | STM32 编译分析、编译优化、printf 重定向 |

### Camera / Sensor

| 文档 | 说明 |
| ---- | ---- |
| [云台开发](/docs/monitor/monitor) | 步进电机控制、驱动芯片 |
| [TMC5272 八点斜坡](/docs/monitor/tmc5272) | 8-Point Ramp 原理与使用 |
| [Sensor 开发](/docs/sensor/sensor) | 传感器驱动调试 |
| [MIPI CSI-2](/docs/sensor/mipi) | D-PHY / C-PHY / D/C-PHY 基础 |
| [V4L2 框架](/docs/sensor/v4l2) | ioctl、VB2 缓冲管理、子设备 |
| [帧同步测试](/docs/sensor/test) | 频率同步与相位同步测试结论 |

### Linux

| 文档 | 说明 |
| ---- | ---- |
| [Linux 常用命令](/docs/linux/linux_cmd) | 系统调试命令集 |
| [时钟频偏计算](/docs/linux/clock) | 时钟精度与频偏分析 |
| [Coredump 调试](/docs/linux/coredump) | 核心转储分析 |
| [Linux 信号](/docs/linux/signal) | SIGILL/SIGSEGV/SIGBUS 等信号 |

### 开发工具

| 文档 | 说明 |
| ---- | ---- |
| [Git 常用操作](/docs/linux/tool/git) | 分支管理、patch、.gitignore |
| [Repo 多仓库管理](/docs/linux/tool/repo) | Android 多仓库工具 |
| [SVN 操作](/docs/linux/tool/svn) | Subversion 使用 |
| [Docker 常用操作](/docs/linux/tool/docker) | 容器管理 |
| [I2C 工具](/docs/linux/tool/i2c_tool) | i2ctools 使用 |
| [tcpdump 抓包](/docs/linux/tool/tcpdump) | 网络抓包分析 |
| [VS Code 技巧](/docs/linux/tool/vscode) | Remote SSH、快捷键 |

### 协议

| 文档 | 说明 |
| ---- | ---- |
| [ONVIF](/docs/protocol/onvif) | 网络摄像头标准协议 |
| [RTSP](/docs/protocol/rtsp) | 实时流传输协议 |
| [RTMP](/docs/protocol/rtmp) | 实时消息传输协议 |
| [NDI](/docs/protocol/ndi) | 网络设备接口协议 |
| [USB](/docs/protocol/usb) | USB 协议基础 |
| [Pelco-D / Pelco-P](/docs/protocol/pelco_d) | 云台控制协议 |

### 系统底层

| 文档 | 说明 |
| ---- | ---- |
| [Kernel 开发](/docs/kernel/kernel) | Linux 内核开发笔记 |
| [U-Boot 开发](/docs/uboot/u-boot) | Bootloader 开发 |

### 调试工具

| 文档 | 说明 |
| ---- | ---- |
| [Wireshark](/docs/Debugger/wireshark) | 网络协议分析 |
