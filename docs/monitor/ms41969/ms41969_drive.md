# MS41969 驱动控制指南

> 本指南基于本仓库 (`motor_pt_drv.c` / `motor.c` / `motor.h`) 的**实际代码**整理,
> 描述如何在 ssd268g 平台上通过 SPI + VD 同步方式驱动 MS41969 双通道步进电机驱动 IC。
> 所有寄存器值、时序均以代码为准,行号可点击跳转。

---

## 1. 概述

MS41969 是一颗双通道步进电机驱动芯片,内置微步 PWM、步进生成与 VD 同步逻辑。本平台用它驱动云台的两台电机:

| 通道 | 芯片内部 | 本项目用途 | 关键寄存器组 |
|---|---|---|---|
| α 电机 | AB 通道 | **水平 (PAN)** | `PSUMAB` / `INTCTAB` / `PPWA/B` |
| β 电机 | CD 通道 | **垂直 (TILT)** | `PSUMCD` / `INTCTCD` / `PPWC/D` |

**驱动模式:VD 同步激磁(VDFZ)**,不是传统 S/D(脉冲+方向)模式。

- 方向、步数、每步周期、电流都通过 **SPI 写寄存器**下达;
- SoC 的 hrtimer 在 `VF1`(水平)/`VF2`(垂直)引脚上产生 **VD 同步信号**,每个 VD 周期触发芯片走 `PSUMXX` 步;
- 真正驱动线圈的微步 PWM 由芯片**内部产生**,SoC 不直接发步进脉冲。

---

## 2. 硬件连接

引脚因产品而异,定义在 `motor.h` 的 `#if defined <产品>` 块里。以 `V6xSS_V` 为例([motor.h:200-211](motor.h#L200-L211)):

| 信号 | 宏 | 方向 | 作用 |
|---|---|---|---|
| SPI 总线 | `SPI_NUM` (`"spi1.0"`) | — | 挂载的 SPI 控制器 |
| CS | `CS_MS41969` (6) | SoC→芯片 | 片选,即 SPI 的 CS |
| RSTB | `RSTB_MS41969` (145) | SoC→芯片 | 复位,高=正常工作 ([motor.c:5290](motor.c#L5290)) |
| VF1 | `VF1` (41) | SoC→芯片 | α(水平)VD 同步信号输入 |
| VF2 | `VF2` (144) | SoC→芯片 | β(垂直)VD 同步信号输入 |
| PLS1 | `PLS1` (143) | 芯片→SoC | α 电机步进监控脉冲输出 |
| PLS2 | `PLS2` (73) | 芯片→SoC | β 电机步进监控脉冲输出 |
| HOR_OC | `HOR_OC_IO_NUMBER` | 芯片→SoC | 水平光耦(限位/零位) |
| VER_OC | `VER_OC_IO_NUMBER` | 芯片→SoC | 垂直光耦 |

> `PLS1/PLS2` 是**输出**(芯片告知走了多少步),`VF1/VF2` 是**输入**(SoC 给同步节拍)。勿混淆。

---

## 3. SPI 通信协议

SPI 物理参数([motor_pt_drv.c:1562-1565](motor_pt_drv.c#L1562-L1565)):

```
max_speed_hz = 1 000 000   (1 MHz)
mode         = SPI_MODE_0 | SPI_CS_HIGH
bits_per_word = 8
cs_gpio      = CS_MS41969
```

### 3.1 位序翻转(关键)

MS41969 在线上是 **LSB 先发**,而 SPI 硬件默认 MSB 先发。代码用软件预翻转来适配:

- `convert_char(byte)`:字节内 bit 顺序反转(b0↔b7)([motor_pt_drv.c:1327](motor_pt_drv.c#L1327))
- `convert_short(short)`:高低字节各自 `convert_char`,再交换高低字节([motor_pt_drv.c:1345](motor_pt_drv.c#L1345))

即**每个地址字节、数据字节都要先做位翻转再上线**。读回时同样要把收到的字节翻转回来(`bits_revest`)。

### 3.2 写寄存器(`My_Spi_Write`,[motor_pt_drv.c:1449](motor_pt_drv.c#L1449))

3 字节帧:`[addr, data_hi, data_lo]`

```
addr     = convert_char(addr)          // 位翻转,bit6 保持 0 = 写
data     = convert_short(data)         // 位翻转+换字节
tx[0]=addr, tx[1]=data_hi, tx[2]=data_lo
```

### 3.3 读寄存器(`My_Spi_Read`,[motor_pt_drv.c:1392](motor_pt_drv.c#L1392))

两段传输:
1. 发 `addr | 0x40`(**bit6=1 表示读**)再 `bits_revest`,1 字节;
2. 发 2 字节 dummy,读回 2 字节;
3. 组装:`data = (bits_revest(rx[1]) << 8) | bits_revest(rx[0])`。

> 命令字节的 **bit6 是读/写标志**:0=写,1=读。地址位于 bit5~0。

所有 SPI 访问在自旋锁 `spi_lock` 下同步进行(`spi_sync_transfer`),禁止中断抢占。

---

## 4. 寄存器地图

定义见 [motor.h:57-98](motor.h#L57-L98)。寄存器为 16 位,字段按位分布。

### 公共/配置寄存器

| 地址 | 宏 | 字段 | 说明 |
|---|---|---|---|
| 0x0B | `TESTEN1_REG_ADDR` / `MODESEL_FZ_REG_ADDR` | D7 TESTEN1;D9 VFx 极性 | 测试使能 / VD 同步信号极性选择 |
| 0x20 | `DT1_REG_ADDR` | D[7:0] DT1 | 起始点等待时间 |
| 0x20 | `PWMMODE_REG_ADDR` | D[12:8] PWMMODE | 微步 PWM 频率分频 |
| 0x20 | `PWMRES_REG_ADDR` | D[14:13] PWMRES | PWM 分辨率 |
| 0x21 | `FZTEST_REG_ADDR` | D[4:0] | PLS1/2 输出信号选择 |
| 0x21 | `TESTEN2_REG_ADDR` | D7 | TEST 使能 2 |
| 0x2C | `INSWICH_REG_ADDR` | D2 | 直流电机控制输入模式 |
| 0x2C | `IN1/IN2_REG_ADDR` | D0/D1 | 直流电机输入控制 |

### α 电机(水平 PAN)

| 地址 | 宏 | 字段 | 说明 |
|---|---|---|---|
| 0x22 | `DT2A_REG_ADDR` | D[7:0] DT2A | α 起始点激励等待时间 |
| 0x22 | `PHMODAB_REG_ADDR` | D[13:8] PHMODAB | α 相位矫正(A/B 相差 90°) |
| 0x23 | `PPWA_REG_ADDR` | D[7:0] PPWA | A 通道峰值脉冲宽度(**电流**) |
| 0x23 | `PPWB_REG_ADDR` | D[15:8] PPWB | B 通道峰值脉冲宽度(**电流**) |
| 0x24 | `PSUMAB_REG_ADDR` | D[7:0] PSUMAB | **每 VD 周期步数** |
| 0x24 | `CCWCWAB_REG_ADDR` | D8 | **转动方向** |
| 0x24 | `BRAKEAB_REG_ADDR` | D9 | 刹车状态 |
| 0x24 | `ENDISAB_REG_ADDR` | D10 | Enable/Disable |
| 0x24 | `MICROAB_REG_ADDR` | D[13:12] | 正弦波微步细分数 |
| 0x25 | `INTCTAB_REG_ADDR` | D[15:0] | **每一步的周期(速度控制量)** |

### β 电机(垂直 TILT)

地址 `0x27`(DT2B/PHMODCD)、`0x28`(PPWC/PPWD)、`0x29`(PSUMCD/CCWCWCD/BRAKECD/ENDISCD/MICROCD)、`0x2A`(INTCTCD),字段定义与 α 完全对应,只是通道换成 CD。

---

## 5. 初始化序列(`Init_Ms41969`,[motor.c:5381](motor.c#L5381))

```
1. 拉高 RSTB_MS41969 = 1,退出复位                        [motor.c:5387-5391]
2. 写 0x20 (DT1/PWMMODE/PWMRES):
       非 TV = 0x4F01,TV = 0x5501
       → PWMMODE 设 PWM 频率 ≈ 27MHz/(0x1E×2^3)=112.5kHz   [motor.c:5393-5401]
3. 写 0x22 (DT2A/PHMODAB) = 0x0001
       → A/B 相差 90°,DT2A ≈ 1×303us                      [motor.c:5411]
4. 写 0x23 (PPWA/PPWB) = PAN_MIN_DUTY_CURRENT
       → 水平负载电流(占空比 = PPWX/(PWMMODE×8))          [motor.c:5418]
5. 写 0x27 (DT2B/PHMODCD) = 0x0001                        [motor.c:5427]
6. 写 0x28 (PPWC/PPWD) = TILT_MIN_DUTY_CURRENT            [motor.c:5433]
7. 写 0x21 (FZTEST) = 0x87  → 使能 PLS1/PLS2 输出         [motor.c:5441]
8. 预置水平:VD↓ → 写 PSUMAB=0x408, INTCTAB=0x666 → VD↑   [motor.c:5450-5453]
9. 预置垂直:VD↓ → 写 PSUMCD=0x411, INTCTCD=0x666 → VD↑   [motor.c:5455-5458]
```

> 第 8/9 步的「VD↓ → 写寄存器 → VD↑」是 VDFZ 模式下达令的标准时序(见第 7 节)。
> `0x666`(INTCT)是上电默认的低速空跑参数,真正运动参数由控制流程动态写入。

---

## 6. 运动控制流程

核心函数:`Create_Motor_Control_Cmd`([motor.c:2245](motor.c#L2245))组装寄存器值,`Update_MS419xx_Reg`([motor.c:1467](motor.c#L1467))下发。

### 6.1 速度等级 → 寄存器值(查表)

速度表存为 `{级别, INTCT(速度控制量), PSUM(步数)}` 三列([motor.h:45-46](motor.h#L45-L46)):

- `INTCT` = 写入 `INTCTXX` 的每步周期(越大越慢);
- `PSUM`  = 写入 `PSUMXX` 低 8 位的每 VD 步数。

查表函数:`Get_List_Speed(motor, level)` 取 INTCT,`Get_List_Step(motor, level)` 取 PSUM。表分普通/跟踪两套(`NORMAL`/`TRACK`),由 `speed_level_mode` 选择。

### 6.2 方向 + 步数合成(PSUMXX 寄存器 0x24/0x29)

方向编在 `PSUMXX` 的 **bit8 (CCWCW)**,与步数合成后一次写入([motor.c:2287-2302](motor.c#L2287-L2302)):

```c
// 非 TV
方向 OP_DIR  → base = 0x500   // bit10(ENDIS=1) + bit8(方向=1)
方向 ZERO_DIR→ base = 0x400   // bit10(ENDIS=1)        (方向=0)
// TV 机型方向极性相反:OP_DIR=0x400, ZERO_DIR=0x500   [motor.c:2277-2297]

cmd = base | (0xFF & step);   // 低 8 位 = 步数
Set_MS419xx_Dir_Step_Cmd(motor, cmd);
```

> `0x400/0x500` 已包含 `ENDIS`(bit10)=1(使能)。默认初值 `0x1500`([motor.c:2250](motor.c#L2250))还带 `MICRO[13:12]` 微步档位。

### 6.3 下发顺序(`Update_MS419xx_Reg`,[motor.c:1481-1500](motor.c#L1481-L1500))

每个 VD 周期,先写 `PSUMXX`(方向+步数),再写 `INTCTXX`(每步周期=速度):

```c
My_Spi_Write(PSUMAB_REG_ADDR, dir_step_cmd);   // 水平
My_Spi_Write(INTCTAB_REG_ADDR, speed_cmd);
```

电流(`PPWX`)在加减速阶段动态调整(见第 9 节)。

### 6.4 一个 VD 周期的完整动作(hrtimer,[motor.c:4877](motor.c#L4877))

```
1. Motor_Process(motor)         // 状态机算出本轮 speed_level / step / dir
2. Pan_VD_Low()                 // VD 拉低(必须)
3. Update_MS419xx_Reg(motor)    // SPI 写 PSUMXX + INTCTXX
4. Pan_VD_High()                // VD 拉高,触发芯片执行本轮 PSUM 步
5. hrtimer_forward(pan_timeout) // 排下一个 VD 周期
```

---

## 7. VD 同步时序

VD 信号由 hrtimer 在 `VF1/VF2` 上产生。两个关键时序常数([motor.c:50](motor.c#L50)、[motor.h:113](motor.h#L113)):

| 宏 | 值 | 含义 |
|---|---|---|
| `NEW_COMMAND_EFFECTIVE_TIME_NS` | 2 000 000 ns (2ms) | VD 拉高后,寄存器命令生效所需时间 |
| `MS41XX_DT_NS` | 303 000 ns (0.303ms) | DT 等待时间补偿 |

**VD 周期计算**([motor.c:4965](motor.c#L4965)):

```
非 TV:  T_VD = INTCT × PSUM × 888.8 ns − 2 000 000 + 303 000   (ns)
                       └ 888.8 = 4444/5 ┘
TV:    T_VD = INTCT × PSUM × (V6xSS_TV_MS41969_VD_CAL/20) − 2 000 000 + 303 000
              └ V6xSS_TV_MS41969_VD_CAL = 39600 → 1980 ┘
```

**步频**(电机轴):

```
f_step = PSUM / T_VD ≈ 1 / (INTCT × 888.8 ns)
```

> **PSUM 被约掉** → 稳态角速度只由 INTCT 决定。PSUM 只影响「每个 VD 包多少步」(影响 VD 频率与加减速粒度),不影响匀速转动快慢。
> 所以调速度 = 改速度等级 = 改查表得到的 INTCT。

---

## 8. 速度与角速度换算

```
角速度(°/s) = f_step × 每 value 云台角度
           = [1e9 / (INTCT × 888.8)] × 每 value 角度
```

每 value 角度由步距角 0.9°、微步细分 256、减速比、`MS41969_PSUMXX_VAL_CONVERSION_SETP_ANGLE=8` 决定([motor.h:401-405](motor.h#L401-L405))。以行程标定反推:每 value ≈ 0.112° 电机轴,折算到云台:

- 水平(减速比 23/5=4.6):≈ 0.0244°/value
- 垂直(减速比 17/5=3.4):≈ 0.0318°/value

(减速比为整数除法近似,理论值,以实测为准。)

---

## 9. 电流控制

电流由 `PPWA/B`(水平)、`PPWC/D`(垂直)设定,占空比 = `PPWX / (PWMMODE × 8)`([motor.c:5417](motor.c#L5417))。电流档位宏见 [motor.h:101-108](motor.h#L101-L108):

| 宏 | 典型值 | 含义 |
|---|---|---|
| `PAN_MAX/MIN_DUTY_CURRENT` | 0x2929 / 0x3838 | 水平最大/最小电流 |
| `TILT_MAX/MIN_DUTY_CURRENT` | 0x3838 / 0x3838 | 垂直最大/最小电流 |
| `PAN/TILT_STOP_DUTY_CURRENT` | 产品相关 | 停止保持电流 |
| `*_CURRENT_MA_550/640/...` | 0x2525/0x3030/... | 毫安档位标定值 |

控制策略([motor.c:2254-2255](motor.c#L2254-L2255)):加减速阶段在 MIN~MAX 之间动态调 `PPWX`,稳速用 MIN,停止切 STOP 档抱紧。

---

## 10. 常见操作速查

> 以下为内核侧逻辑,用户态通过 `/dev/VHD_MOTOR_PT` 的 ioctl 间接驱动,不直接操作寄存器。

| 操作 | 内核入口 | 说明 |
|---|---|---|
| 读寄存器 | `My_Spi_Read(addr)` | addr bit6 置 1 |
| 写寄存器 | `My_Spi_Write(addr, data)` | 同步、加锁 |
| 设置目标位置 | `Set_Tar_Position(motor, pos)` | 触发运动 |
| 设置方向/步数 | `Set_MS419xx_Dir_Step_Cmd` | bit8 方向 + 低 8 步数 |
| 设置速度等级 | `Set_Cur_Speed(motor, level)` | 查表得 INTCT/PSUM |
| 急停 | `Instancy_Stop(motor)` ([motor.c:1892](motor.c#L1892)) | |
| 查光耦 | `Get_Pan/Tilt_Optocoupler_Status()` | 限位/零位判断 |
| 读步数监控 | `My_Spi_Read(PLS 计数)` / PLS1/2 中断 | 实际走了多少步 |

用户态 ioctl 命令(magic `'Z'`,见 [motor_pt_drv.c:59-74](motor_pt_drv.c#L59-L74)):`SET/QUE_PT_POSTION`、`SET_PT_LIMIT`、`PT_STOP`、`SSP_READ/WRITE_REG`、`TEST_PT_MODE`、`STANDBY` 等。

---

## 11. 调试手段

- **proc 节点**:`cat /proc/VHD_MOTOR_PT` 打印 VF1/VF2、PLS1/PLS2、光耦等 GPIO 实时电平([motor_pt_drv.c:602](motor_pt_drv.c#L602))。
- **debug 开关:模块参数 `motor_trace_param`**(默认 0),置非 0 打开 `motor_debug(...)` 详细 printk([motor_pt_drv.h:55-59](motor_pt_drv.h#L55-L59)、[motor_pt_drv.c:43](motor_pt_drv.c#L43))。
- **标定开关:模块参数 `pt_cal_en`**(默认 1),置 0 跳过上电自检标定([motor_pt_drv.c:159-161](motor_pt_drv.c#L159-L161))。
- **寄存器回读**:用 `My_Spi_Read` 读 `PSUMAB/INTCTAB` 等确认下发值是否生效([motor.c:4507-4508](motor.c#L4507-L4508) 有示例)。

---

## 附:关键文件索引

| 文件 | 内容 |
|---|---|
| [motor_pt_drv.c](motor_pt_drv.c) | SPI 读写、misc 设备、ioctl、初始化入口 |
| [motor.c](motor.c) | 电机控制算法、VD hrtimer、`Init_Ms41969`、速度表 |
| [motor.h](motor.h) | 寄存器地址、产品引脚/行程配置、速度/电流常数 |
