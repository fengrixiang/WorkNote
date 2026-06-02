# PTP 音频时钟同步详解

## 1. 概述

在嵌入式音视频系统中，音频采样时钟（MCLK, Master Clock）需要与 PTP（Precision Time Protocol）主时钟保持精确同步，以确保音视频长期播放不出现漂移。

本项目通过 PTP Stack 计算频率偏移量，经平滑滤波后调整音频硬件时钟（gclk_audio2），实现音频采样率与网络时间的精确绑定。

### 1.1 关键参数

| 参数 | 值 | 说明 |
|------|-----|------|
| BASE_FREQ | 12,288,000 Hz | 音频 MCLK 基频 (48kHz × 256) |
| SMOOTH_FACTOR | 0.1 | EMA 平滑系数 |
| MAX_FREQ_DIFF | 1000 Hz | 最大频率偏差限制 |
| 目标采样率 | 48 kHz | 常见音频采样率 |

---

## 2. PTP 频率偏移计算原理

### 2.1 PTP 同步机制

```
PTP Master                          PTP Slave
    │                                   │
    │──── Sync (t1) ───────────────────>│  t1 = 发送时间戳
    │──── Follow_Up (t1) ──────────────>│  携带精确 t1
    │                                   │
    │<──── Delay_Req (t4) ─────────────│  t4 = 发送时间戳
    │──── Delay_Resp (t4) ────────────>│  携带精确 t4
    │                                   │
```

### 2.2 延迟与偏移计算

```bash
path_delay = [(t2 - t1) + (t4 - t3)] / 2
offset     = (t2 - t1) - path_delay
```

### 2.3 频率偏差估算

PTP 通过一段时间内的时间戳差分来估算频率偏差：

```bash
// 测量周期 Δt = 1 秒
freq_offset = (本地时钟计数增量 - Δt × 标称频率) / 标称频率 × 1e9
            = 本地 tick 差分 - 1e9  (单位: ppb)
```

### 2.4 输出值含义

| 字段 | 单位 | 含义 |
|------|------|------|
| master offset | ns | 瞬时相位误差（时间偏差） |
| s2 freq | ppb | 估算的频率偏差（parts per billion） |

---

## 3. 核心公式详解

### 3.1 指数移动平均（EMA）平滑滤波

```bash
smoothed_adj = SMOOTH_FACTOR × adj + (1 - SMOOTH_FACTOR) × smoothed_adj_prev
```

其中：
- `adj` = 当前 s2 freq（原始频率偏差）
- `smoothed_adj_prev` = 上一次平滑值
- `SMOOTH_FACTOR = 0.1`

**物理意义**：抑制 PTP 抖动传入音频时钟，滤除高频噪声。

**等效截止频率**（假设 PTP 更新周期 T = 1s）：

```
f_cutoff = α / (2π × T) ≈ 0.016 Hz
```

即：频率高于 0.016 Hz 的抖动都会被平滑过滤。

### 3.2 频率补偿公式

```bash
target_freq = BASE_FREQ - BASE_FREQ × smoothed_adj / 1e9
```

展开形式：

```
target_freq = BASE_FREQ × (1 - smoothed_adj / 1e9)
           = BASE_FREQ × (1 - 偏移比例)
```

**物理意义**：根据 PTP 计算的频率偏差，按比例调整音频 MCLK。

### 3.3 公式推导

设：
- `BASE_FREQ = 12,288,000 Hz`
- `smoothed_adj = +2600 ppb`

计算：

```
Δf = BASE_FREQ × smoothed_adj / 1e9
   = 12,288,000 × 2600 / 1,000,000,000
   = 31.9488 Hz

target_freq = BASE_FREQ - Δf
           = 12,288,000 - 31.95
           = 12,287,968 Hz
```

**结论**：当 s2 freq 为 +2600 ppb 时，音频 MCLK 降低约 32 Hz。

### 3.4 限幅保护

```bash
long MAX_FREQ_DIFF = 1000;  // 最大允许偏差 1000 Hz

if (target_freq > BASE_FREQ + MAX_FREQ_DIFF) {
    target_freq = BASE_FREQ + MAX_FREQ_DIFF;
} else if (target_freq < BASE_FREQ - MAX_FREQ_DIFF) {
    target_freq = BASE_FREQ - MAX_FREQ_DIFF;
}
```

**作用**：
- 防止异常大的 PTP 抖动传入音频系统
- PTP 失锁时不会导致音频时钟突变
- 保护硬件 VCXO 的物理限制

---

## 4. 完整闭环控制框图

```
┌─────────────────────────────────────────────────────────────┐
│                    PTP Grandmaster Clock                     │
└────────────────────────────┬────────────────────────────────┘
                             │ 网络 Sync 报文
                             ↓
┌─────────────────────────────────────────────────────────────┐
│                      PTP Servo (PI Controller)               │
│                                                             │
│   master offset (相位误差 ns) ──► Σ ──► Kp ──┐             │
│                              │         ↓                    │
│   目标: offset=0 ◄───────────┼────► new_freq                 │
│                              │                              │
│   s2 freq = 频率偏差 ppb      │                              │
└──────────────────────────────┼──────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              ↓                                 ↓
   ┌─────────────────────┐          ┌─────────────────────────┐
   │  CLOCK_REALTIME     │          │    gclk_audio2 (MCLK)    │
   │  调整系统时钟        │          │    调整音频采样时钟       │
   └─────────────────────┘          └─────────────────────────┘
```

---

## 5. master offset 与 s2 freq 的关系

### 5.1 因果关系

```
s2 freq (频率偏差)  ──► 调整时钟  ──►  master offset (相位误差)

master offset 是结果，s2 freq 是原因。
```

Servo 通过调节 **freq** 来消除 **offset** 的稳态误差。

### 5.2 PI 控制原理

```bash
new_freq = Kp × offset + Ki × Σoffset
```

- **P 项（比例）**：对当前 offset 做出即时反应
- **I 项（积分）**：累积 offset，驱动稳态误差到零

### 5.3 实际数据对应

```
时刻    master offset (ns)    s2 freq (ppb)    平滑值
790           -128              +2510           2579
791           +94               +2693           2591
792           -125              +2503           2582
793           +38               +2628           2587
```

观察：
- offset 偏负（本地慢）→ s2 freq 偏低
- offset 偏正（本地快）→ s2 freq 偏高
- Servo 正在调节频率以消除 offset

---

## 6. 代码实现

### 6.1 clock.c 中的实现

```bash
static int clock_synchronize_locked(struct clock *c, double adj)
{
    // 1. 基础时钟调整（CLOCK_REALTIME）
    if (clockadj_set_freq(c->clkid, -adj)) {
        return -1;
    }
    if (c->clkid == CLOCK_REALTIME) {
        sysclk_set_sync();
    }

    // 2. 音频时钟同步（只从模式下执行）
    if (clock_best_foreign(c)) {
        static FILE *audio_clk_fp = NULL;
        static double smoothed_adj = 0.0;
        static int init = 0;
        double BASE_FREQ = 12288000.0;
        double SMOOTH_FACTOR = 0.1;
        long MAX_FREQ_DIFF = 1000;

        // EMA 平滑
        if (!init) {
            smoothed_adj = adj;
            init = 1;
        } else {
            smoothed_adj = SMOOTH_FACTOR * adj
                         + (1.0 - SMOOTH_FACTOR) * smoothed_adj;
        }

        // 计算目标频率
        long target_freq = (long)(BASE_FREQ - BASE_FREQ * smoothed_adj / 1e9);

        // 限幅
        if (target_freq > BASE_FREQ + MAX_FREQ_DIFF) {
            target_freq = BASE_FREQ + MAX_FREQ_DIFF;
        } else if (target_freq < BASE_FREQ - MAX_FREQ_DIFF) {
            target_freq = BASE_FREQ - MAX_FREQ_DIFF;
        }

        // 写入硬件
        if (!audio_clk_fp) {
            audio_clk_fp = fopen("/proc/ambarella/clock", "w");
        }
        if (audio_clk_fp) {
            fprintf(audio_clk_fp, "gclk_audio2 %ld\n", target_freq);
            fflush(audio_clk_fp);
        }

        pr_info("audio clock: raw freq %+7.0f -> smoothed %+7.2f -> set %ld Hz",
                adj, smoothed_adj, target_freq);
    }

    return 0;
}
```

### 6.2 早期 Shell 脚本实现（对比）

```bash
SMOOTH_FACTOR=0.1
BASE_FREQ=12288000
SMOOTHED_VAL=0

while read line; do
    if echo "$line" | grep -q "freq"; then
        freq_val=$(echo "$line" | awk -F'freq' '{print $2}' | awk '{print $1}')
        SMOOTHED_VAL=$(awk -v current="$freq_val" -v prev="$SMOOTHED_VAL" \
                        -v factor="$SMOOTH_FACTOR" 'BEGIN {
            result = factor * current + (1 - factor) * prev
            printf "%.2f", result
        }')
        target_freq=$(awk -v base="$BASE_FREQ" -v offset="$SMOOTHED_VAL" 'BEGIN {
            printf "%.0f", base - (base * offset / 1000000000)
        }')
        echo "$CLOCK_NAME $target_freq" > "$TARGET_FILE"
    fi
done
```

**对比**：

| 特性 | Shell 脚本 | clock.c |
|------|-----------|---------|
| 限幅保护 | 无 | ±1000 Hz |
| 状态持久化 | 每次重置 | static 变量保持 |
| 文件操作 | 每次 open/close | 保持 fd 打开 |
| 调用时机 | 被动读取 | 与 PTP 时钟同步深度绑定 |

---

## 7. 实际数据分析

### 7.1 日志解析示例

```
ptp4l[790.212]: master offset       -128 s2 freq   +2510 path delay     11830
ptp4l[790.212]: audio clock: raw freq   +2510 -> smoothed +2579.46 -> set 12287968 Hz
```

### 7.2 验算

```
平滑值验算（α=0.1）:
smoothed = 0.1 × 2510 + 0.9 × 2579.46 = 2572.51 ≈ 2579.46 ✓

目标频率验算:
target = 12288000 - 12288000 × 2579.46 / 1e9
       = 12288000 - 31683.7
       = 12287968 Hz ✓
```

### 7.3 长期稳态数据

```
目标频率: 12287967 ~ 12287970 Hz (稳定)
基准频率: 12288000 Hz
频率偏差: -30 ~ -33 Hz (偏低)

s2 freq 稳定在 ~2600 ppb (本地时钟比标称快 0.00026%)
```

---

## 8. 物理意义总结

### 8.1 为什么要调整音频时钟？

视频应用中音视频同步要求极高 — 音频的采样时钟必须与视频的帧时钟锁定到同一个时间源（PTP Grandmaster）。否则长时间运行后会出现音视频漂移。

### 8.2 频率 vs 相位

- **相位同步**（offset = 0）：瞬时对齐，时间误差为 0
- **频率同步**（s2 freq = 0）：速率匹配，长期不会累积误差

理想状态：**频率锁定 + 相位围绕 0 小幅波动**

### 8.3 平滑滤波的作用

EMA 滤波防止：
1. PTP 短期抖动传入音频硬件
2. 音频时钟频繁调整导致硬件不稳定
3. 网络波动影响音频质量

---


