# Sensor 开发

图像传感器驱动调试相关的接口协议、框架知识和测试记录。

## 基础知识

| 文档 | 说明 |
| ---- | ---- |
| [图像格式入门](/docs/sensor/common/image_format) | RAW/RGB/YUV/NV12/MJPEG 全链路解析、数据量对比、格式选型与常见踩坑 |
| [MIPI CSI-2 接口](/docs/sensor/mipi) | D-PHY / C-PHY / D/C-PHY 物理层、CSI-2 协议层、数据包结构 |
| [V4L2 框架](/docs/sensor/v4l2) | ioctl 速查、VB2 缓冲管理、子设备与 Media Controller、像素格式 |
| [IAV 框架](/docs/sensor/iav) | Ambarella 视频编码框架：DSP 通道、码率控制、状态机、编程流程、调试方法 |

## 测试记录

| 文档 | 说明 |
| ---- | ---- |
| [SP50E40 帧同步/频率/相位测试](/docs/sensor/sp50e40) | VX720A + SP50E40 上 VTS Auto/Reset 模式的帧率跟随、Δt 线性分析、vsync 抖动、QSC 场景相位评估 |
