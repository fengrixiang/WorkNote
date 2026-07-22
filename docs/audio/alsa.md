# ALSA 开发指南

## 目录

- [一、ALSA 概述](#一alsa-概述)
- [二、ALSA 架构](#二alsa-架构)
- [三、ALSA-lib 基础 API](#三alsa-lib-基础-api)
- [四、PCM 设备编程](#四pcm-设备编程)
- [五、硬件参数详解](#五硬件参数详解)
- [六、软件参数](#六软件参数)
- [七、播放示例](#七播放示例)
- [八、录制示例](#八录制示例)
- [九、异步模式](#九异步模式)
- [十、Mixer 控制](#十mixer-控制)
- [十一、命令行工具](#十一命令行工具)
- [十二、调试与常见问题](#十二调试与常见问题)
- [十三、进阶话题](#十三进阶话题)

---

## 一、ALSA 概述

### 1.1 什么是 ALSA

**ALSA**（Advanced Linux Sound Architecture，高级 Linux 声音架构）是 Linux 内核中提供音频功能的标准接口，自 Linux 2.6 起取代了旧的 OSS（Open Sound System）。它既包括内核驱动框架（`sound/` 目录），也包括用户态库 `libasound`（即 alsa-lib）。

### 1.2 ALSA 的组成

| 组件 | 说明 |
|------|------|
| **ALSA 内核驱动** | `sound/core`、`sound/pci`、`sound/soc` 等，负责硬件访问 |
| **alsa-lib** | 用户态 API 库，应用程序通过它访问驱动 |
| **alsa-utils** | 一组命令行工具：`aplay`、`arecord`、`amixer`、`alsamixer` 等 |
| **alsa-plugins** | 插件层，支持 PulseAudio、JACK、OSS 模拟等 |
| **alsa-firmware** | 部分声卡需要的固件（HDSP、SB Live 等） |

### 1.3 为什么选 ALSA

- Linux 原生支持，内核自带驱动
- 支持 PCM、Mixer、MIDI、Sequencer、Timer 等多种子接口
- 灵活支持中断、DMA、softmmap、硬件指针同步
- 嵌入式平台（RK、HiSilicon、Ambarella 等）几乎都使用 ALSA 作为后端

---

## 二、ALSA 架构

### 2.1 整体层次

```
┌──────────────────────────────────────┐
│         应用 / 用户态工具             │
│  (aplay, arecord, your_app, ffmpeg)  │
└──────────────┬───────────────────────┘
               │  ioctl / read / write / mmap
               ▼
┌──────────────────────────────────────┐
│        alsa-lib（用户态封装）         │
│   snd_pcm_*, snd_mixer_*, snd_ctl_*  │
└──────────────┬───────────────────────┘
               │  ioctl
               ▼
┌──────────────────────────────────────┐
│         Linux 内核 ALSA 子系统        │
│  (sound/core, sound/soc/ASoC 架构)   │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│            声卡硬件 / Codec            │
│  (I2S/PCM/TDM 控制器 + Audio Codec)  │
└──────────────────────────────────────┘
```

### 2.2 ASoC 架构（嵌入式重点）

嵌入式平台（SoC）多采用 **ASoC**（ALSA System on Chip）架构，把音频系统拆成三部分：

- **Platform（CPU DAI）**：SoC 集成的 I2S/PCM 控制器（如 `rockchip_i2s`、`snd-soc-hisi-i2s`）
- **Codec（编解码器）**：外置音频 DAC/ADC（如 ES7210、ES8311、ALC5616）
- **Machine（板级绑定）**：通过 `sound/soc/<soc>/<board>.c` 将 Platform 与 Codec 绑定，注册声卡

> 移植新板卡时，重点是 machine 驱动和 codec 驱动的对接，详见 [ES7210 开发指南](es7210)。

### 2.3 设备节点

ALSA 在 `/dev/snd/` 下暴露设备文件：

```bash
$ ls /dev/snd/
controlC0   # Mixer / 控制接口（C0 表示 Card 0）
pcmC0D0c    # PCM Capture（录音）
pcmC0D0p    # PCM Playback（播放）
pcmC0D1c    # 第二个 PCM 设备
timer       # 高精度定时器
seq         # 音序器
```

- `C` 后缀为 Card 编号
- `D` 后缀为 Device 编号
- `c` / `p` 标识方向：Capture / Playback

---

## 三、ALSA-lib 基础 API

### 3.1 头文件与链接

```c
#include <alsa/asoundlib.h>
```

编译时链接 `-lasound`：

```bash
gcc demo.c -o demo -lasound
```

### 3.2 错误处理宏

```c
#define CHECK_RC(rc) \
    do { if (rc < 0) { \
        fprintf(stderr, "%s:%d: %s\n", __func__, __LINE__, snd_strerror(rc)); \
        return -1; } } while (0)
```

### 3.3 常用 API 一览

| 模块 | 函数 | 用途 |
|------|------|------|
| PCM  | `snd_pcm_open` | 打开 PCM 设备 |
| PCM  | `snd_pcm_close` | 关闭 |
| PCM  | `snd_pcm_hw_params_*` | 设置硬件参数 |
| PCM  | `snd_pcm_sw_params_*` | 设置软件参数 |
| PCM  | `snd_pcm_readi/writei` | 同步读写 |
| PCM  | `snd_pcm_mmap_*` | mmap 模式 |
| PCM  | `snd_pcm_async` | 异步通知 |
| Mixer| `snd_mixer_open` | 打开混音器 |
| Mixer| `snd_mixer_selem_*` | 操作简单元素（音量/开关） |
| CTL  | `snd_ctl_open` | 通用控制接口 |

---

## 四、PCM 设备编程

PCM 数据流是 ALSA 最核心的部分。所有播放 / 录制都通过 PCM 设备完成。

### 4.1 PCM 名称与打开

```c
int snd_pcm_open(snd_pcm_t **pcm,
                 const char *name,
                 snd_pcm_stream_t stream,
                 int mode);
```

- `name` 形式：`hw:0,0`（Card 0 Device 0）、`plughw:0,0`（带插件转换）、`default`（默认设备）
- `stream`：`SND_PCM_STREAM_PLAYBACK` 或 `SND_PCM_STREAM_CAPTURE`
- `mode`：常用 `0` 或 `SND_PCM_NONBLOCK`（非阻塞）

> `hw` 直接对接硬件，参数必须严格匹配；`plughw` 会自动重采样/格式转换，推荐开发期使用。

### 4.2 编程流程总览

```
打开设备
  ↓
分配 hw_params 结构
  ↓
设置访问类型 (interleaved/noninterleaved)
  ↓
设置采样格式 (S16_LE/S24_LE/S32_LE/FLOAT_LE)
  ↓
设置声道数
  ↓
设置采样率
  ↓
设置缓冲区/周期大小
  ↓
写入 hw_params
  ↓
分配 sw_params 并设置
  ↓
准备 snd_pcm_prepare
  ↓
循环 readi / writei
  ↓
drop + close
```

### 4.3 一个最简示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <alsa/asoundlib.h>

int main(void)
{
    snd_pcm_t *pcm;
    int rc;

    rc = snd_pcm_open(&pcm, "default", SND_PCM_STREAM_PLAYBACK, 0);
    CHECK_RC(rc);

    rc = snd_pcm_set_params(pcm,
                            SND_PCM_FORMAT_S16_LE,
                            SND_PCM_ACCESS_RW_INTERLEAVED,
                            2,                  // 通道数
                            48000,              // 采样率
                            1,                  // 软重采样
                            500000);            // 缓冲区 500ms
    CHECK_RC(rc);

    short buf[1024 * 2] = {0};
    for (int i = 0; i < 100; i++) {
        rc = snd_pcm_writei(pcm, buf, 1024);
        if (rc < 0) {
            snd_pcm_recover(pcm, rc, 0);
        }
    }

    snd_pcm_drain(pcm);
    snd_pcm_close(pcm);
    return 0;
}
```

---

## 五、硬件参数详解

### 5.1 hw_params 设置的完整步骤

```c
snd_pcm_hw_params_t *hw;
snd_pcm_hw_params_alloca(&hw);

snd_pcm_hw_params_any(pcm, hw);                              // 初始化
snd_pcm_hw_params_set_access(pcm, hw, SND_PCM_ACCESS_RW_INTERLEAVED);
snd_pcm_hw_params_set_format(pcm, hw, SND_PCM_FORMAT_S16_LE);
snd_pcm_hw_params_set_channels(pcm, hw, 2);

unsigned int rate = 48000;
snd_pcm_hw_params_set_rate_near(pcm, hw, &rate, 0);

snd_pcm_uframes_t period_size = 1024;
snd_pcm_hw_params_set_period_size_near(pcm, hw, &period_size, 0);

unsigned int periods = 4;
snd_pcm_hw_params_set_periods_near(pcm, hw, &periods, 0);

snd_pcm_hw_params(pcm, hw);                                 // 真正生效
```

### 5.2 访问模式（Access）

| 宏 | 说明 |
|----|------|
| `SND_PCM_ACCESS_RW_INTERLEAVED` | 交错帧，**最常用**（LRLRLR…） |
| `SND_PCM_ACCESS_RW_NONINTERLEAVED` | 平面，所有 L 通道再所有 R 通道 |
| `SND_PCM_ACCESS_MMAP_INTERLEAVED` | mmap 模式 |
| `SND_PCM_ACCESS_MMAP_NONINTERLEAVED` | mmap + 平面 |
| `SND_PCM_ACCESS_MMAP_COMPLEX` | 复杂布局 |

### 5.3 采样格式（Format）

| Format | 位宽 | 字节序 | 用途 |
|--------|------|--------|------|
| `SND_PCM_FORMAT_S16_LE` | 16 | LE | 通用 CD 音质 |
| `SND_PCM_FORMAT_S24_LE` | 24 | LE | 录音常用 |
| `SND_PCM_FORMAT_S24_3LE` | 24 (3 字节) | LE | 紧凑存储 |
| `SND_PCM_FORMAT_S32_LE` | 32 | LE | 高端处理 |
| `SND_PCM_FORMAT_FLOAT_LE` | 32 | LE | 内部混音 |
| `SND_PCM_FORMAT_U8` | 8 | — | 兼容性 |

注意：
- 24 bit 在 32 bit 容器中左对齐（低 8 位为 0）
- Codec 给的 24 bit 真实数据在低 24 bit（**视具体平台而定**，需读 codec 手册）

### 5.4 声道数（Channels）

`channels`：单声道 = 1，立体声 = 2，TDM 可达 8/16 通道。

### 5.5 采样率（Rate）

- 常用 8k、16k、44.1k、48k、96k、192k
- `set_rate_near` 会返回驱动实际设置的最近值
- 强匹配可用 `set_rate`

### 5.6 缓冲与周期（Buffer / Period）

PCM 内部使用 **buffer = periods × period_size** 的环形缓冲。

```
|<-        buffer_size          ->|
|---|---|---|---|---|---|---|---|
  p0  p1  p2  p3  ...           pN
```

- **period_size**：每次硬件中断/DMA 搬运的帧数
- **periods**：一个完整 buffer 含多少个 period
- **latency ≈ periods × period_size / rate**

> 嵌入式场景典型：`period_size=1024`, `periods=4`, 48kHz → 单向延迟约 85ms。

设置建议：

```c
snd_pcm_uframes_t buffer_size;
snd_pcm_hw_params_get_buffer_size(hw, &buffer_size);
printf("buffer = %lu frames\n", buffer_size);
```

---

## 六、软件参数

软件参数控制传输行为、启动阈值、停止阈值等。

```c
snd_pcm_sw_params_t *sw;
snd_pcm_sw_params_alloca(&sw);
snd_pcm_sw_params_current(pcm, sw);

// 启动阈值：buffer 达到 N 帧后才真正开始播放
snd_pcm_uframes_t start_threshold = buffer_size / 2;
snd_pcm_sw_params_set_start_threshold(pcm, sw, start_threshold);

// 停止阈值：剩余 < N 帧时自动停止（underrun）
snd_pcm_uframes_t stop_threshold = buffer_size;
snd_pcm_sw_params_set_stop_threshold(pcm, sw, stop_threshold);

// avail_min：唤醒至少 N 帧
snd_pcm_uframes_t avail_min = period_size;
snd_pcm_sw_params_set_avail_min(pcm, sw, avail_min);

snd_pcm_sw_params(pcm, sw);
```

---

## 七、播放示例

### 7.1 同步写（writei）

```c
static int pcm_write(snd_pcm_t *pcm, const void *buf, snd_pcm_uframes_t frames)
{
    int rc;
    while (frames > 0) {
        rc = snd_pcm_writei(pcm, buf, frames);
        if (rc == -EPIPE) {           // underrun
            snd_pcm_recover(pcm, rc, 0);
            continue;
        } else if (rc == -EAGAIN) {
            snd_pcm_wait(pcm, 100);
            continue;
        } else if (rc < 0) {
            return rc;
        }
        buf   += rc * snd_pcm_frames_to_bytes(pcm, 1);
        frames -= rc;
    }
    return 0;
}
```

### 7.2 mmap 模式

```c
const snd_pcm_channel_area_t *areas;
snd_pcm_uframes_t offset, frames = period_size;
snd_pcm_mmap_begin(pcm, &areas, &offset, &frames);

// 写入数据（指针换算较复杂）
char *p = (char *)areas[0].addr + offset * areas[0].step / 8;
// ... 填充数据 ...

snd_pcm_mmap_commit(pcm, offset, frames);
```

mmap 优点：零拷贝，性能高；缺点：复杂度高，调试困难。

### 7.3 WAV 文件播放

```c
// 1. 解析 WAV 头：采样率、通道、位宽
// 2. snd_pcm_set_params 与之一致
// 3. 循环 read() 数据并 writei()
```

可参考 `alsa-utils/aplay/lists.c` 的实现。

---

## 八、录制示例

```c
snd_pcm_t *cap;
snd_pcm_open(&cap, "default", SND_PCM_STREAM_CAPTURE, 0);
snd_pcm_set_params(cap, SND_PCM_FORMAT_S16_LE,
                   SND_PCM_ACCESS_RW_INTERLEAVED,
                   2, 48000, 1, 500000);

short buf[1024 * 2];
FILE *fp = fopen("record.pcm", "wb");

for (;;) {
    int rc = snd_pcm_readi(cap, buf, 1024);
    if (rc == -EPIPE) {
        snd_pcm_recover(cap, rc, 0);  // overrun
        continue;
    }
    if (rc > 0) fwrite(buf, sizeof(short) * 2, rc, fp);
    // ... 退出条件 ...
}
```

录制对应 **overrun**（应用读太慢导致硬件缓冲覆盖未读数据），对应 `snd_pcm_recover`。

---

## 九、异步模式

ALSA 支持基于 `SIGIO` 的异步通知：

```c
snd_pcm_async(pcm, 1);   // 启用
// 收到 SIGIO 时，可用 snd_pcm_avail_update 知道有多少帧
```

也可以使用 `snd_pcm_wait(pcm, timeout_ms)` 配合 poll/select/epoll：

```c
struct pollfd pfd = { .fd = snd_pcm_poll_descriptors(pcm)[0].fd,
                      .events = POLLIN };
int rc = poll(&pfd, 1, 1000);
if (rc > 0) snd_pcm_readi(pcm, buf, frames);
```

适合需要与其他 I/O 事件一起复用的事件循环。

---

## 十、Mixer 控制

### 10.1 简单元素（Simple Element）

```c
snd_mixer_t *mixer;
snd_mixer_open(&mixer, 0);
snd_mixer_attach(mixer, "default");
snd_mixer_selem_register(mixer, NULL, NULL);
snd_mixer_load(mixer);

snd_mixer_selem_id_t *sid;
snd_mixer_selem_id_alloca(&sid);
snd_mixer_selem_id_set_name(sid, "Master");

snd_mixer_elem_t *elem = snd_mixer_find_selem(mixer, sid);

long vol = 80;
snd_mixer_selem_set_playback_volume_all(elem, vol);
snd_mixer_selem_set_playback_switch_all(elem, 1);
```

### 10.2 常用控件名

| 名称 | 含义 |
|------|------|
| `Master` | 总音量 |
| `Headphone` | 耳机 |
| `Speaker` | 喇叭 |
| `Playback Switch` | 静音 |
| `Capture Source` | 录制输入源 |
| `Mic` / `Mic Boost` | 麦克风增益 |

实际名称可用 `amixer scontrols` 或 `alsamixer` 查看。

### 10.3 CTL 通用接口

对于非标准控件（如自定义寄存器、ASoC DAPM 控件），需使用 `snd_ctl_*`：

```c
snd_ctl_t *ctl;
snd_ctl_open(&ctl, "hw:0", 0);
snd_ctl_elem_info_t *info;
snd_ctl_elem_info_alloca(&info);
snd_ctl_elem_info_set_id(info, id, 0);
snd_ctl_elem_info(ctl, info);
// ... read / write value ...
snd_ctl_close(ctl);
```

---

## 十一、命令行工具

### 11.1 设备与能力查询

```bash
aplay -l                    # 列出播放设备
arecord -l                  # 列出录音设备
cat /proc/asound/cards      # 声卡列表
cat /proc/asound/pcm        # PCM 设备能力
ls /proc/asound/card0/      # 单卡详细信息
```

### 11.2 播放 / 录制

```bash
# 播放 wav
aplay -D hw:0,0 sample.wav

# 录音
arecord -D hw:0,0 -f S16_LE -r 48000 -c 2 -d 10 test.wav

# 指定格式
aplay -D plughw:0,0 -t raw -f S16_LE -r 16000 -c 1 input.pcm
```

### 11.3 混音器

```bash
amixer scontrols              # 简单控件
amixer scontents              # 详细控件 + 值
amixer set Master 80%         # 设置音量
amixer set Master mute        /  unmute
amixer cset numid=3 1         # 直接设置 control id
alsamixer                     # 交互式 TUI
```

### 11.4 文件格式说明

| 后缀 | 格式 |
|------|------|
| `.wav` | WAV（带 44 字节头） |
| `.pcm` / `.raw` | 裸 PCM（aplay 通过 `-t raw -f` 解析） |
| `.au` | AU/SND |

---

## 十二、调试与常见问题

### 12.1 打开设备失败

```
snd_strerror: No such file or directory
```

排查：
1. 驱动是否加载：`lsmod | grep snd`
2. 设备节点是否存在：`ls /dev/snd`
3. 设备名拼写是否正确（`plughw:0,0` vs `hw:0,0`）

### 12.2 参数设置失败

`set_format` / `set_channels` 失败：硬件不支持。改用 `plughw:` 让 alsa-lib 自动转换。

### 12.3 播放噪声 / 杂音

- 采样率不匹配（44.1k vs 48k）
- I2S 时钟配置错误
- MCLK / BCLK 比例不在 codec 支持范围
- 检查 `period_size`，太小容易 underrun

### 12.4 xrun 调试

```c
snd_pcm_status_t *st;
snd_pcm_status_alloca(&st);
snd_pcm_status(pcm, st);
printf("state=%d avail=%ld delay=%ld\n",
       snd_pcm_status_get_state(st),
       snd_pcm_status_get_avail(st),
       snd_pcm_status_get_delay(st));
```

- 频繁 **underrun**（播放）：加大 buffer 或减少 CPU 占用
- 频繁 **overrun**（录音）：应用读得太慢

### 12.5 启用 trace

```bash
echo 1 > /proc/asound/card0/pcm0p/xrun_debug
# 或环境变量
export LIBASOUND_DEBUG=1
```

### 12.6 常见错误码

| 错误码 | 含义 | 处理 |
|--------|------|------|
| `-EBUSY` | 设备被占用 | 关闭占用进程 |
| `-EPIPE` | underrun/overrun | recover |
| `-ESTRPIPE` | 流挂起（电源管理） | prepare 后继续 |
| `-ENOTTY` | 不支持 ioctl | 多半设备名错 |

---

## 十三、进阶话题

### 13.1 多声道 / TDM

```c
snd_pcm_set_params(pcm, SND_PCM_FORMAT_S32_LE,
                   SND_PCM_ACCESS_RW_INTERLEAVED,
                   8,        // 8 通道
                   48000, 1, 500000);
```

需在 machine 驱动中正确配置 TDM slot 宽度、偏移。

### 13.2 硬件指针与时间戳

```c
snd_pcm_uframes_t hptr;
snd_pcm_hwsync(pcm);
snd_pcm_get_hw_ptr(pcm, &hptr);

snd_pcm_status_t *st;
snd_pcm_status_get_htstamp(st, &ts);  // 最近一次硬件更新时刻
```

对音视频同步（如 PTP 音频时钟同步）非常关键。

### 13.3 自定义插件

通过 `alsa-plugins`，可挂接：
- `ladspa` / `lv2` 效果
- `pulse` / `jack` 转发
- 自定义文件 I/O / 网络

### 13.4 与 PulseAudio / PipeWire 关系

```
App → PulseAudio / PipeWire → ALSA
```

- PulseAudio 默认打开后独占 `hw:`，应用需改用 `pulse` 或 `default`
- PipeWire 提供 `pipewire` PCM 插件

### 13.5 在嵌入式平台常见问题

1. **DMA 缓冲区对齐**：部分 SoC 要求 buffer 4K 对齐
2. **I2S 抖动**：长距离走线加屏蔽，注意 BCLK/MCLK 走线等长
3. **Codec 复位时序**：上电顺序、寄存器恢复（需用 `regcache`）
4. **Suspend / Resume**：保存 DAPM 状态、关闭时钟

### 13.6 推荐阅读

- ALSA Project 官网：<https://www.alsa-project.org/>
- alsa-lib API 文档：`/usr/share/doc/libasound2-doc/`（apt 安装 `libasound2-doc`）
- 《Linux Device Drivers》LDD3 第 7 章
- ASoC 文档：`Documentation/sound/soc/`（内核源码）
- 源码：`alsa-lib/aplay`、`alsa-lib/arecord` 是最佳参考示例

---

## 附录 A：常用函数速查

```c
/* PCM */
int  snd_pcm_open(snd_pcm_t **, const char *, snd_pcm_stream_t, int);
int  snd_pcm_close(snd_pcm_t *);
int  snd_pcm_prepare(snd_pcm_t *);
int  snd_pcm_drop(snd_pcm_t *);
int  snd_pcm_drain(snd_pcm_t *);
int  snd_pcm_recover(snd_pcm_t *, int, int);
int  snd_pcm_writei(snd_pcm_t *, const void *, snd_pcm_uframes_t);
int  snd_pcm_readi (snd_pcm_t *, void *,       snd_pcm_uframes_t);
int  snd_pcm_set_params(snd_pcm_t *, snd_pcm_format_t,
                        snd_pcm_access_t, unsigned int,
                        unsigned int, int, unsigned int);
snd_pcm_sframes_t snd_pcm_avail(snd_pcm_t *);
snd_pcm_sframes_t snd_pcm_avail_update(snd_pcm_t *);

/* hw_params 家族 */
int  snd_pcm_hw_params_any(snd_pcm_t *, snd_pcm_hw_params_t *);
int  snd_pcm_hw_params_set_access(snd_pcm_t *, snd_pcm_hw_params_t *, snd_pcm_access_t);
int  snd_pcm_hw_params_set_format(snd_pcm_t *, snd_pcm_hw_params_t *, snd_pcm_format_t);
int  snd_pcm_hw_params_set_channels(snd_pcm_t *, snd_pcm_hw_params_t *, unsigned int);
int  snd_pcm_hw_params_set_rate(snd_pcm_t *, snd_pcm_hw_params_t *, unsigned int, int);
int  snd_pcm_hw_params_set_period_size(snd_pcm_t *, snd_pcm_hw_params_t *, snd_pcm_uframes_t, int);
int  snd_pcm_hw_params_set_periods(snd_pcm_t *, snd_pcm_hw_params_t *, unsigned int, int);
int  snd_pcm_hw_params_set_buffer_size(snd_pcm_t *, snd_pcm_hw_params_t *, snd_pcm_uframes_t);
int  snd_pcm_hw_params(snd_pcm_t *, snd_pcm_hw_params_t *);
```

## 附录 B：采样格式字节序速查

| Format | LE 字节序（一个 sample） |
|--------|--------------------------|
| S16_LE | L, H |
| S24_LE | L, M, H, **0**（4 字节容器）|
| S24_3LE| L, M, H（3 字节） |
| S32_LE | L, M, H, 0（按需） |
| FLOAT_LE | 4 字节 IEEE-754 |

> 写入时务必确认 codec 给的数据在 L 还是 H 起始 — 不同厂商不同，参考 codec 数据手册。
