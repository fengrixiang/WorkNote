# LIS2DW12 G-Sensor 技术文档

## 1. 概述

VX730 使用 **LIS2DW12**（ST 超低功耗三轴加速度计）实现**自动翻转检测**功能。通过检测 Z 轴加速度方向判断云台是否被倒置安装，并自动通知 SOC 切换图像方向。

### 芯片参数

| 参数 | 值 |
|------|------|
| 芯片型号 | LIS2DW12 |
| 制造商 | STMicroelectronics |
| 测量轴 | 3 轴（X/Y/Z） |
| 量程可选 | ±2g / ±4g / ±8g / ±16g |
| 分辨率 | 12/14 bit |
| 工作电压 | 1.62V ~ 3.6V |
| 接口 | I2C / SPI |
| Device ID | 0x44（WHO_AM_I 寄存器 0x0F） |

### 当前项目配置

| 配置项 | 值 | 说明 |
|--------|------|------|
| 量程 | ±2g | `LIS2DW12_2g` |
| 电源模式 | High Performance | `LIS2DW12_HIGH_PERFORMANCE` |
| 输出数据率 | 25 Hz | `LIS2DW12_XL_ODR_25Hz` |
| 滤波器 | 低通滤波 LPF_ON_OUT | `LIS2DW12_LPF_ON_OUT` |
| 滤波带宽 | ODR/4 | `LIS2DW12_ODR_DIV_4` |
| BDU | 使能 | 块数据更新，保证高低字节一致性 |
| 灵敏度 | 0.061 mg/LSB | ±2g 量程下 |

---

## 2. 硬件接口

### 2.1 GPIO 连接

使用 GPIO 模拟 I2C（bit-bang），不使用硬件 I2C 外设：

```text
AT32F413 MCU
  ├── PC0 (GPIO) ──→ LIS2DW12 SDA（I2C 数据）
  └── PC5 (GPIO) ──→ LIS2DW12 SCL（I2C 时钟）
```

| 引脚 | 宏定义 | GPIO 端口 |
|------|--------|-----------|
| SCL | `I2C_SCL_PIN` / `I2C_SCL_GPIO_PORT` | PC5 |
| SDA | `I2C_SDA_PIN` / `I2C_SDA_GPIO_PORT` | PC0 |

### 2.2 I2C 地址

```text
I2C 7位地址：0x18（SA0=0）或 0x19（SA0=1）
本项目使用 8位写地址：0x30（LTS_ADDR）
读地址：0x31（= 0x30 + 1）
```

---

## 3. 软件架构

### 3.1 文件结构

```text
sourse/g_sensor.c        — 应用层：初始化、翻转检测、主循环任务
sourse/g_sensor.h        — 应用头文件：I2C 宏、寄存器结构体、枚举定义
sourse/lis2dw12_reg.h    — ST 官方驱动头文件：完整寄存器定义和 API 声明
sourse/lis2dw12_reg.c    — ST 官方驱动源文件（平台无关层）
sourse/common.h          — USER_GENSOR 宏开关
```

### 3.2 条件编译

```c
// common.h
#define USER_GENSOR     // 启用 G-Sensor 功能

// user_main.c 中的使用
#ifdef USER_GENSOR
    Init_LIS2DW12();    // 上电初始化
#endif

#ifdef USER_GENSOR
    Gsensor_Task();     // 主循环任务
#endif
```

### 3.3 主循环位置

`Gsensor_Task()` 在 `user_main.c` 主循环中**最先执行**（在 SOC 命令处理和电机任务之前），确保翻转状态及时更新：

```text
while(1) {
    Gsensor_Task()          ← 第1个执行
    Uart1_Command_Process() ← SOC 命令
    Motor_Drv_Task()        ← 电机驱动
    Tmc5272_Pt_Task()       ← 云台状态机
    ...
}
```

---

## 4. 初始化流程

`Init_LIS2DW12()` 执行步骤：

```text
1. WHO_AM_I 验证
   └── 循环读取 0x0F 寄存器，期望值 0x44
   └── 超时 1000 次后报错退出

2. 软复位（CTRL2 寄存器 0x21）
   └── 置 bit6=1 触发 soft_reset
   └── 轮询等待 bit6 自动清零（复位完成）

3. 功能配置
   ├── CTRL2 (0x21): BDU=1（块数据更新）
   ├── CTRL6 (0x25): FS=00（±2g）+ LPF 使能
   ├── CTRL7 (0x3F): BW_FILT=01（ODR/4）
   └── CTRL1 (0x20): 模式=High Performance + ODR=25Hz

4. 读回验证
   └── 读取并打印 CTRL1/CTRL6/CTRL7 的实际值
```

初始化日志示例：

```text
[OK] LIS2DW12 ID verified (0x44)
[OK] Device reset complete
[CONFIG] Block Data Update enabled
[CONFIG] Full scale: +-2g | Filter: LPF_ON_OUT
[CONFIG] Filter bandwidth: ODR/4
[CONFIG] Power mode: High Performance | ODR: 25Hz
CTRL1: 0x34
CTRL6: 0x00
CTRL7: 0x40
```

---

## 5. 关键寄存器

### 5.1 控制寄存器

| 地址 | 名称 | 关键位 | 当前配置 |
|------|------|--------|---------|
| 0x20 | CTRL1 | odr[7:4] + mode[3:2] + lp_mode[1:0] | 0x34 = 25Hz + High Performance |
| 0x21 | CTRL2 | bdu[3] + soft_reset[6] | BDU=1 |
| 0x22 | CTRL3 | slp_mode[1:0] + h_lactive[3] | 默认 |
| 0x25 | CTRL6 | fs[5:4] + fds[3] + bw_filt[7:6] | ±2g, LPF, ODR/4 |
| 0x3F | CTRL_REG7 | usr_off_on_out[4] + bw_filt via CTRL6 | ODR/4 带宽 |

### 5.2 数据寄存器

| 地址 | 名称 | 说明 |
|------|------|------|
| 0x27 | STATUS | bit0=drdy（数据就绪标志） |
| 0x28 | OUT_X_L | X 轴加速度低字节 |
| 0x29 | OUT_X_H | X 轴加速度高字节 |
| 0x2A | OUT_Y_L | Y 轴加速度低字节 |
| 0x2B | OUT_Y_H | Y 轴加速度高字节 |
| 0x2C | OUT_Z_L | Z 轴加速度低字节 |
| 0x2D | OUT_Z_H | Z 轴加速度高字节 |
| 0x0F | WHO_AM_I | 设备 ID = 0x44 |

---

## 6. 数据读取

### 6.1 读取流程

```text
Read_LIS()
 ├── 读取 STATUS 寄存器 (0x27)
 ├── 检查 bit0 (drdy) == 1？
 │   ├── 否 → 返回 0（数据未就绪）
 │   └── 是 ↓
 ├── Multiple_Read_LIS2DW12(0x28, 6)  // 连续读 X/Y/Z 共 6 字节
 ├── 拼合 X/Y/Z 原始值（有符号 16 位）
 ├── 独立读取 Z 轴寄存器 (0x2C, 0x2D)
 ├── 拼合 Z 轴原始值
 ├── 换算为 mg：acc = raw × 0.061
 └── 返回 1（读取成功）
```

### 6.2 单位换算

```text
原始值（16位有符号）→ 加速度（mg）

±2g 量程下：mg = raw × 0.061

示例：
  raw = 16384 → acc = 16384 × 0.061 = 999.4 mg ≈ 1g（静止朝上）
  raw = -16384 → acc = -999.4 mg ≈ -1g（静止朝下）
  raw = 0 → acc = 0 mg（水平）
```

---

## 7. 翻转检测算法

### 7.1 检测原理

通过 Z 轴加速度方向判断云台朝向：

```text
         正常安装                   倒置安装
         Z 轴朝上                   Z 轴朝下
           ↑                          ↓
      ┌────┤                     ┌────┤
      │ 云台│                     │ 云台│
      │    │                     │    │
      └────┘                     └────┘
    acc ≈ +1000mg              acc ≈ -1000mg
    (重力方向与Z轴同向)         (重力方向与Z轴反向)
```

### 7.2 阈值定义

```c
#define Z_UP_THRESHOLD_MIN    100.0f    // 恢复阈值：Z > 100mg → 正常
#define Z_DOWN_THRESHOLD_MAX  -100.0f   // 翻转阈值：Z < -100mg → 倒置
#define FLIP_FILTER_THRESHOLD  10       // 连续 10 次确认才切换状态
```

| 条件 | 判定 | 说明 |
|------|------|------|
| acc < -100 mg | 翻转 | Z 轴朝下，云台倒置 |
| acc > +100 mg | 正常 | Z 轴朝上，云台正常安装 |
| -100 ≤ acc ≤ 100 | 不变 | 处于过渡区域，保持当前状态 |

### 7.3 状态机

```text
                ┌──────────────────────────┐
                │                          │
                ▼                          │
         ┌─────────────┐   acc < -100mg    │   连续10次
  ──→    │ 正常(flip=0) │──────────────→ 翻转(flip=1)
         └─────────────┘                  └─────────────┐
                ▲                                         │
                │       acc > +100mg     连续10次         │
                └─────────────────────────────────────────┘

  状态切换需要连续 10 次采样确认（滤波消抖）
  任一次不满足条件 → 计数器清零
```

### 7.4 滤波参数

| 参数 | 值 | 含义 |
|------|------|------|
| `FLIP_FILTER_THRESHOLD` | 10 | 连续确认次数 |
| `FLIP_CHECK_INTERVAL` | 100 ms | 检测周期（10 Hz） |

有效消抖时间 = 10 次 × 100ms = **1 秒**。状态切换至少需要 Z 轴加速度持续超过阈值 1 秒。

---

## 8. 任务系统

### 8.1 Gsensor_Task()

```text
Gsensor_Task()  — 每 100ms 执行一次
 ├── 时间检查：Is_Time_Out_Ms(last_time, 100)
 ├── Get_Stable_Flip()
 │   ├── Read_LIS() → 读取 Z 轴加速度
 │   └── 翻转状态机 → 返回 flip_state
 ├── 状态变化检测
 │   └── last_flip_state != current_flip？
 │       ├── 是 → 发送 VISCA 翻转通知
 │       └── 否 → 不发送
 └── [MOTOR_STATUS_MONITOR] 电机状态监控（可选，1秒周期）
```

### 8.2 VISCA 翻转通知

状态变化时，主动向 SOC 发送翻转命令：

```text
帧格式：81 01 04 66 XX ff

XX 取值：
  0x05 (GSENSOR_ON_COMMAND)  → 云台倒置，SOC 应翻转图像
  0x04 (GSENSOR_OFF_COMMAND) → 云台正常，SOC 应恢复图像
```

### 8.3 SOC 初始化查询

SOC 启动时（`soc_init_func`）也会查询当前翻转状态，ARM 主动上报一次：

```c
uint8_t current_flip = Get_Stable_Flip();
visca_flip_v_buf[4] = current_flip ? GSENSOR_ON_COMMAND : GSENSOR_OFF_COMMAND;
Usart1_SendBuffer(visca_flip_v_buf, 0, 6);
```

### 8.4 G-Sensor 查询命令

SOC 可主动查询 G-Sensor 数据：

```text
查询帧：81 0B 01 30 ff
回复帧：81 0A 01 30 01 00 00 XX ff

  [4] = 1 (sensor type)
  [5] = 0
  [6] = 0
  [7] = XX → Z 轴加速度映射值（map_float_to_uint8_fixed(acc)）
```

---

## 9. I2C 通信实现

### 9.1 GPIO 模拟 I2C 时序

本项目使用软件模拟 I2C，关键时序参数：

| 操作 | 时序 |
|------|------|
| 延时单位 | 5 μs（`MMA_Delay5us`） |
| 写操作后延时 | 5 ms（`MMA_Delay5ms`） |
| START 条件 | SDA 高→低，SCL 为高 |
| STOP 条件 | SDA 低→高，SCL 为高 |
| 数据有效 | SCL 高电平期间 SDA 稳定 |

### 9.2 读写函数

| 函数 | 用途 |
|------|------|
| `Single_Write_LIS2DW12(addr, reg, data)` | 单字节写寄存器 |
| `Single_Read_LIS2DW12(addr, reg)` | 单字节读寄存器 |
| `Multiple_Read_LIS2DW12(addr, start, count)` | 多字节连续读 |

### 9.3 代码适配说明

`g_sensor.c` 中包含两套寄存器操作 API：

- **ST 官方 API**（带 `stmdev_ctx_t` 参数）：在 `lis2dw12_reg.h` 中声明，但本项目**未使用**（因为需要平台适配层）
- **自定义直接操作 API**（`Single_Read/Write_LIS2DW12`）：直接调用 GPIO 模拟 I2C，本项目实际使用

`g_sensor.c` 中还内联了部分 ST API 的简化版本（如 `lis2dw12_reset_set`、`lis2dw12_full_scale_set`），内部调用自定义的 `lis2dw12_read_reg` / `lis2dw12_write_reg`，但实际初始化流程使用的是更直接的 `Single_Read/Write` 方式。
