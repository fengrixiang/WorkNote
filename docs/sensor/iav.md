# Ambarella IAV 框架

> 适用 SoC：CV2, CV22, CV25, CV72, S3L, S5L

IAV（Image Audio Video）是 Ambarella SoC 上控制视频编码、解码、ISP 和音频处理的核心用户空间 API 层，封装了与底层 DSP 硬件和内核驱动的交互细节。

## 软件架构

```text
┌─────────────────────────────────────────────────┐
│             应用层 (Application)                  │
│        录像管理 / 流媒体 / AI 推理接入             │
├─────────────────────────────────────────────────┤
│          中间件层 (Middleware)                     │
│   RTSP / ONVIF / 文件封装 (MP4/MKV)              │
├─────────────────────────────────────────────────┤
│          ★ IAV API 层 (libiavutils.so)           │
│   编码控制 / 解码控制 / ISP 参数 / 音频           │
├─────────────────────────────────────────────────┤
│          内核驱动层 (Kernel Driver)                │
│   ambarella_iav.ko / ambarella_isp.ko            │
│   /dev/iav  /dev/isp0                            │
├─────────────────────────────────────────────────┤
│          硬件抽象层 (HAL)                          │
│   DSP 固件 / 硬件编码器 / DMA 引擎                │
├─────────────────────────────────────────────────┤
│          硬件层 (Hardware)                         │
│   DSP / ARM Cortex / ISP Pipeline / MIPI CSI     │
└─────────────────────────────────────────────────┘
```

核心库文件：

| 库文件 | 功能 |
|--------|------|
| `libiavutils.so` | IAV 核心工具库，封装 ioctl 调用为友好 API |
| `libiav.so` | IAV 基础接口库 |
| `libambaaudio.so` | 音频编解码库 |
| `libambadsp.so` | DSP 控制辅助库 |

## IAV / DSP / 硬件编码器的关系

```text
              ARM Cortex (Linux)                DSP Core
              ┌──────────────────┐              ┌──────────────────┐
              │  用户空间应用      │              │  DSP 固件(ucode)  │
              │    ↓ (ioctl)      │              └──────┬───────────┘
              │  /dev/iav         │                     │ 调度
              │    ↓              │   Mailbox/          ↓
              │  ambarella_iav.ko │──AmbaLink──→ 硬件编码器 IP
              └──────────────────┘              H.264/H.265/MJPEG
```

- ARM 侧运行 Linux，执行 IAV 驱动和用户空间应用
- DSP 侧运行专有固件，直接控制硬件编码器 IP
- 两者通过 **Mailbox**（老平台）或 **AmbaLink**（新平台如 CV2 系列）通信

## 数据流路径

```text
图像传感器 (Sensor)
    │ MIPI CSI-2 / VIP
    ↓
VIN (Video Input) ─── 捕获原始帧 (Raw Bayer)
    │
    ↓
ISP Pipeline ─── 去噪 / 3A (AE/AWB/AF) / WDR / 色彩校正
    │
    ├──→ YUV Buffer ──→ DSP Encode ──→ BSB Buffer ──→ ARM 读取码流
    │
    └──→ DISP ──→ VOUT (HDMI/LCD/MIPI DSI)
```

## 核心概念

### Context（上下文）

| Context | 说明 | 典型用途 |
|---------|------|---------|
| **ENC** (Encode) | 编码上下文 | 管理编码会话、码流参数 |
| **DISP** (Display) | 显示上下文 | 合成画面、叠加 OSD |
| **VIN** (Video Input) | 视频输入上下文 | 配置传感器接口、帧率、分辨率 |
| **VOUT** (Video Output) | 视频输出上下文 | 配置 HDMI/LCD 输出 |

初始化顺序：`VIN 初始化 → DISP 配置 → ENC 配置 → 开始编码`

### DSP Channel（DSP 通道）

DSP 内部以通道为单位管理编码任务，每个编码流对应一个 DSP 通道：

```text
                ISP 输出 (YUV)
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
    Channel 0   Channel 1   Channel 2
    (主码流)     (子码流1)    (子码流2)
    4K@30fps    1080P@30fps  720P@15fps
    H.265       H.265       MJPEG
```

- 通道数量取决于 SoC 型号和 DSP 固件配置（CV2 系列通常 4~8 个）
- 每个通道可独立配置：分辨率、帧率、编码格式、码率、GOP 结构

### 编码格式

| 格式 | 说明 | 适用场景 |
|------|------|---------|
| **H.264/AVC** | 最广泛兼容 | 兼容性优先、低复杂度 |
| **H.265/HEVC** | 比 H.264 节省约 40% 码率 | 高分辨率、带宽受限 |
| **MJPEG** | 每帧独立 JPEG | 逐帧访问、高品质抓拍 |

### 码率控制

| 模式 | 特点 | 适用场景 |
|------|------|---------|
| **CBR** | 严格维持目标码率 | 存储带宽固定、网络传输带宽确定 |
| **VBR** | 根据内容复杂度动态调整 | 存储、画质优先 |
| **AVBR** | Ambarella 自研，根据场景自适应 | 带宽受限的网络摄像机 |

```c
// CBR 配置
enc_param.bitrate_control = IAV_BRC_CBR;
enc_param.target_bitrate  = 4096;    // 4 Mbps

// VBR 配置
enc_param.bitrate_control = IAV_BRC_VBR;
enc_param.target_bitrate  = 4096;    // 4 Mbps 目标
enc_param.max_bitrate     = 8192;    // 8 Mbps 上限

// AVBR 配置
enc_param.bitrate_control = IAV_BRC_AVBR;
enc_param.target_bitrate  = 4096;
enc_param.avbr_sensitivity = 50;
```

### GOP 结构

```text
GOP: IDR_interval = 30, B_frames = 2, Reference = 1

帧类型:  I  B  B  P  B  B  P  B  B  P ... B  B  P  I
帧序号:  0  1  2  3  4  5  6  7  8  9 ... 27 28 29 30
         └─── GOP ──────────────────────────────┘
```

| 参数 | 说明 | 典型值 |
|------|------|--------|
| **IDR Interval** | IDR 帧间隔，决定 GOP 长度 | 15~60 帧（安防常用 30~60） |
| **B Frame Count** | B 帧数量，越多压缩率越高但延迟越大 | 0~3（实时场景用 0） |
| **Reference Count** | 参考帧数，影响编码质量和内存占用 | 1~4 |

### BSB（Bitstream Buffer）

BSB 是编码码流的输出缓冲区，是 DSP 到 ARM 的关键数据桥梁：

```text
┌─────────────────────────────────────────────────┐
│                   BSB Memory Pool                │
│  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐       │
│  │ BSB 0 │ │ BSB 1 │ │ BSB 2 │ │ BSB 3 │  ...  │
│  │ Ch.0  │ │ Ch.0  │ │ Ch.1  │ │ Ch.1  │       │
│  │  NAL  │ │  NAL  │ │  NAL  │ │  NAL  │       │
│  └───────┘ └───────┘ └───────┘ └───────┘       │
│          DMA 可访问（物理连续内存）                 │
└─────────────────────────────────────────────────┘
```

工作流程：

1. **编码前**：ARM 通过 IAV 为每个编码通道分配 BSB 内存
2. **编码中**：DSP 将编码后的 NAL 单元写入 BSB
3. **读取码流**：ARM 通过 `IAV_IOC_READ_BITSTREAM_EX` 读取
4. **释放缓冲**：ARM 处理完后通过 `IAV_IOC_RELEASE_BITSTREAM_BUF` 释放

> BSB 必须是物理连续内存（DMA 要求），大小需根据码率和帧率合理配置，不足会导致丢帧或阻塞。

### ROI（Region of Interest）

ROI 允许在画面中指定区域分配更多码率，保证关键区域画质：

```text
┌─────────────────────────────────┐
│         普通区域：正常码率        │
│     ┌─────────────┐             │
│     │   ROI 区域   │             │
│     │ (2x 码率权重)│             │
│     └─────────────┘             │
│         普通区域：正常码率        │
└─────────────────────────────────┘
```

典型应用：安防监控中对画面中心的人脸/车牌区域设置 ROI。

## IAV 状态机

```text
            IAV_STATE_INIT
                 │
                 │ open("/dev/iav")
                 ↓
          IAV_STATE_IDLE ─────────────────────┐
                 │                            │
                 │ IAV_IOC_IDLE_TO_PREVIEW    │
                 ↓                            │
         IAV_STATE_PREVIEW                    │
                 │                            │
                 │ IAV_IOC_START_ENCODE       │
                 ↓                            │
        IAV_STATE_ENCODING ───────────────────┘
                          IAV_IOC_STOP_ENCODE
```

| 状态 | 说明 | 允许的操作 |
|------|------|-----------|
| **IDLE** | DSP 未运行 | 分配缓冲区、加载 DSP 固件 |
| **PREVIEW** | ISP 运行，可看预览画面 | 修改编码参数、配置通道 |
| **ENCODING** | DSP 执行编码 | 读取码流、动态调参（码率/帧率） |

> 状态切换必须按顺序，不能跳过。编码参数大部分需在 PREVIEW 状态配置，部分参数（码率）可在 ENCODING 下动态修改。

## 编程流程

### 1. 打开设备

```c
int fd_iav = open("/dev/iav", O_RDWR);
if (fd_iav < 0) {
    perror("open /dev/iav");
    return -1;
}
```

### 2. 进入 IDLE 状态

```c
int state;
ioctl(fd_iav, IAV_IOC_GET_STATE, &state);

if (state == IAV_STATE_ENCODING)
    ioctl(fd_iav, IAV_IOC_STOP_ENCODE, 0);
if (state == IAV_STATE_PREVIEW)
    ioctl(fd_iav, IAV_IOC_PREVIEW_TO_IDLE, 0);
```

### 3. 加载 DSP 固件

```c
struct iav_dsp_fw fw;
fw.fw_addr = dsp_fw_buffer;
fw.fw_size = dsp_fw_size;
ioctl(fd_iav, IAV_IOC_LOAD_DSP_FW, &fw);
```

### 4. 分配缓冲区

```c
// 分配 BSB
struct iav_buffer_alloc buf_req = {0};
buf_req.type      = IAV_BUF_BSB;
buf_req.stream_id = 0;
buf_req.size      = 4 * 1024 * 1024;  // 4MB
ioctl(fd_iav, IAV_IOC_ALLOC_BUFFER, &buf_req);

// 分配 Y/UV 帧缓冲
buf_req = (struct iav_buffer_alloc){0};
buf_req.type    = IAV_BUF_YUV;
buf_req.width   = 3840;
buf_req.height  = 2160;
buf_req.num_buf = 8;
ioctl(fd_iav, IAV_IOC_ALLOC_BUFFER, &buf_req);
```

### 5. 进入 PREVIEW 并配置编码流

```c
ioctl(fd_iav, IAV_IOC_IDLE_TO_PREVIEW, 0);

// 主码流 H.265
struct iav_stream_cfg stream_cfg = {0};
stream_cfg.stream_id   = 0;
stream_cfg.encode_type = IAV_ENCODE_H265;
stream_cfg.width       = 3840;
stream_cfg.height      = 2160;
stream_cfg.frame_rate  = IAV_FPS_30;
ioctl(fd_iav, IAV_IOC_SET_STREAM_CFG, &stream_cfg);

// 编码参数
struct iav_enc_param enc_param = {0};
enc_param.stream_id        = 0;
enc_param.bitrate_control  = IAV_BRC_AVBR;
enc_param.target_bitrate   = 8192;
enc_param.gop.idr_interval = 30;
enc_param.gop.b_frame_count = 0;
enc_param.gop.ref_count    = 1;
ioctl(fd_iav, IAV_IOC_SET_ENC_PARAM, &enc_param);
```

### 6. 开始编码

```c
struct iav_encode_start start = {0};
start.stream_mask = 0x03;  // Bit0=Stream0, Bit1=Stream1
ioctl(fd_iav, IAV_IOC_START_ENCODE, &start);
```

### 7. 码流读取循环

```c
while (encoding) {
    struct iav_bitstream_info bs_info;

    // 查询码流就绪状态
    if (ioctl(fd_iav, IAV_IOC_GET_BITSTREAM, &bs_info) < 0)
        continue;

    for (int ch = 0; ch < MAX_STREAMS; ch++) {
        if (!(bs_info.stream_mask & (1 << ch)))
            continue;

        // 读取码流
        struct iav_bitstream_read bs_read = {0};
        bs_read.stream_id   = ch;
        bs_read.buffer      = bitstream_buffer[ch];
        bs_read.buffer_size = BS_BUF_SIZE;

        int bytes = ioctl(fd_iav, IAV_IOC_READ_BITSTREAM_EX, &bs_read);
        if (bytes > 0)
            process_bitstream(ch, bs_read.buffer, bytes);

        // 释放缓冲
        ioctl(fd_iav, IAV_IOC_RELEASE_BITSTREAM_BUF, &ch);
    }
}
```

### 8. 停止编码

```c
ioctl(fd_iav, IAV_IOC_STOP_ENCODE, 0);     // → PREVIEW
ioctl(fd_iav, IAV_IOC_PREVIEW_TO_IDLE, 0); // → IDLE
close(fd_iav);
```

## ioctl 参考

### 状态管理

| ioctl 命令 | 方向 | 参数 | 说明 |
|-----------|------|------|------|
| `IAV_IOC_GET_STATE` | _IOR | `int *` | 获取当前 IAV 状态 |
| `IAV_IOC_IDLE_TO_PREVIEW` | _IOW | `void` | IDLE → PREVIEW |
| `IAV_IOC_PREVIEW_TO_IDLE` | _IOW | `void` | PREVIEW → IDLE |
| `IAV_IOC_START_ENCODE` | _IOW | `struct iav_encode_start *` | PREVIEW → ENCODING |
| `IAV_IOC_STOP_ENCODE` | _IOW | `void` | ENCODING → PREVIEW |

### 编码控制

| ioctl 命令 | 方向 | 参数 | 说明 |
|-----------|------|------|------|
| `IAV_IOC_SET_STREAM_CFG` | _IOW | `struct iav_stream_cfg *` | 配置编码流 |
| `IAV_IOC_GET_STREAM_CFG` | _IOR | `struct iav_stream_cfg *` | 获取编码流配置 |
| `IAV_IOC_SET_ENC_PARAM` | _IOW | `struct iav_enc_param *` | 设置编码参数 |
| `IAV_IOC_SET_BITRATE` | _IOW | `struct iav_bitrate_cfg *` | 动态修改码率 |
| `IAV_IOC_SET_FRAMERATE` | _IOW | `struct iav_framerate_cfg *` | 动态修改帧率 |
| `IAV_IOC_FORCE_IDR` | _IOW | `int *` | 强制插入 IDR 帧 |

### 码流读取

| ioctl 命令 | 方向 | 参数 | 说明 |
|-----------|------|------|------|
| `IAV_IOC_GET_BITSTREAM` | _IOR | `struct iav_bitstream_info *` | 查询码流就绪状态 |
| `IAV_IOC_READ_BITSTREAM_EX` | _IOWR | `struct iav_bitstream_read *` | 读取码流数据 |
| `IAV_IOC_RELEASE_BITSTREAM_BUF` | _IOW | `int *` | 释放码流缓冲区 |

### 缓冲区管理

| ioctl 命令 | 方向 | 参数 | 说明 |
|-----------|------|------|------|
| `IAV_IOC_GET_BUFFER_INFO` | _IOR | `struct iav_buffer_info *` | 获取缓冲区信息 |
| `IAV_IOC_ALLOC_BUFFER` | _IOW | `struct iav_buffer_alloc *` | 分配缓冲区 |
| `IAV_IOC_FREE_BUFFER` | _IOW | `struct iav_buffer_alloc *` | 释放缓冲区 |

## 调试方法

### /proc 调试接口

```bash
# DSP 负载
cat /proc/ambarella/dsp

# 编码状态（帧率、码率、丢帧统计）
cat /proc/ambarella/encoder

# 缓冲区使用情况（BSB/YUV/ME）
cat /proc/ambarella/buffer

# VIN 输入状态（传感器、MIPI）
cat /proc/ambarella/vin

# 设置 IAV 调试等级
echo 7 > /proc/ambarella/iav/debug_level
# 0=无日志 1=ERROR 3=WARNING 5=INFO 7=DEBUG 8=VERBOSE
```

### 常见问题排查

| 现象 | 可能原因 | 排查方法 |
|------|---------|---------|
| 编码启动失败 | DSP 固件未加载/缓冲区未分配/状态不对 | `cat /proc/ambarella/dsp`、`cat /proc/ambarella/iav` |
| 编码丢帧 | BSB 太小 / DSP 过载 / 帧缓冲不足 | 检查 encoder 的 Dropped 计数、DSP Load、BSB 使用率 |
| 码率控制不准 | QP 范围太窄 / AVBR 未收敛 | 对比 target bitrate 和实际 bitrate |
| 画面质量差 | 码率过低 / BRC 参数不当 / ISP 问题 | 提高码率、调整 QP 范围、检查 3A |

### 监控脚本

```bash
#!/bin/sh
# iav_monitor.sh - IAV 编码监控
INTERVAL=${1:-2}

while true; do
    clear
    echo "===== $(date '+%Y-%m-%d %H:%M:%S') ====="
    echo "--- DSP Load ---"
    cat /proc/ambarella/dsp
    echo "--- Encoder ---"
    cat /proc/ambarella/encoder
    echo "--- BSB ---"
    cat /proc/ambarella/buffer | grep -A5 "BSB"
    sleep $INTERVAL
done
```

## 编码参数推荐

| 场景 | 格式 | 分辨率 | 帧率 | BRC | GOP |
|------|------|--------|------|-----|-----|
| 4K 安防录像 | H.265 | 3840x2160 | 30 | AVBR 6Mbps | IDR=30, B=0 |
| 1080P 网传 | H.264 | 1920x1080 | 15 | CBR 2Mbps | IDR=15, B=0 |
| 720P 子码流 | H.264 | 1280x720 | 15 | CBR 1Mbps | IDR=15, B=0 |
| 抓拍流 | MJPEG | 1920x1080 | 1 | - | - |
| 车载 ADAS | H.264 | 1920x1080 | 30 | CBR 4Mbps | IDR=15, B=0 |
