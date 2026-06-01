# TMC5272 调试记录

## 1、电流配置

### 1.1 相关寄存器

| 寄存器 | 地址 | 说明 |
|--------|------|------|
| IHOLD_IRUN | 0x30 | 保持/运行/启动电流配置 |
| IHOLD | [4:0] | 保持电流 (0-31) |
| IRUN | [9:5] | 运行电流 (0-31) |
| IHOLDDELAY | [15:12] | 保持电流延迟切换时间 |
| TMC5272_IRUN_SCALE | 0xED | 运行电流缩放 |
| TMC5272_IHOLD_SCALE | 0xEE | 保持电流缩放 |

### 1.2 电流计算

```
实际电流 = 寄存器值 / 32 × VREF / sense电阻

典型配置：
- sense电阻 = 0.1Ω
- VREF = 2.5V
- IRUN = 20 (20/32 × 2.5 / 0.1 = 1.56A RMS)
- IHOLD = 8 (8/32 × 2.5 / 0.1 = 0.625A RMS)
```

### 1.3 调试步骤

```c
// 1. 设置 sense 电阻值对应的 VREF
// VREF = 目标电流 × sense电阻 × 32
// 例如目标 1.5A：VREF = 1.5 × 0.1 × 32 = 4.8V（注意不要超过芯片极限）

// 2. 配置 IHOLD_IRUN
writeRegister(0x30, (IHOLD << 0) | (IRUN << 5) | (IHOLDDELAY << 12));

// 3. 运行电流缩放（可选）
writeRegister(0xED, 31);  // 31 = 100% 缩放

// 4. 验证电流
// 示波器测量 sense 电阻两端电压，计算 I = V / R
```

### 1.4 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 电机发热严重 | IHOLD 过高或未进入保持模式 | 检查 IHOLDDELAY，确保进入省电模式 |
| 启动无力 | IRUN 不足 | 增加 IRUN 值或提高 VREF |
| 启动抖动 | 电流突变太大 | 增加 IHOLDDELAY 延迟切换 |

## 2、抖动配置

### 2.1 八点斜坡参数

TMC5272 的 8 点斜坡允许精确控制加减速过程，减小抖动：

| 参数 | 寄存器 | 说明 |
|------|--------|------|
| VSTART | 0x1B | 起始速度 |
| A1 | 0x1C | 第一段加速度 |
| V1 | 0x1D | 第一速度阈值 |
| A2 | 0x1E | 第二段加速度 |
| V2 | 0x1F | 第二速度阈值 |
| AMAX | 0x20 | 最大加速度 |
| VMAX | 0x21 | 最大速度 |
| DMAX | 0x22 | 最大减速度 |
| D2 | 0x23 | 第二段减速度 |
| D1 | 0x24 | 第一段减速度 |
| VSTOP | 0x25 | 停止速度 |
| TVMAX | 0x26 | 匀速段最短时间 |
| TZEROWAIT | 0x27 | 过零等待时间 |

### 2.2 伪 S 形曲线配置

```c
// 典型低抖动配置
writeRegister(0x1B, 0);        // VSTART = 0
writeRegister(0x1C, 200);      // A1 = 200 (低速起步缓)
writeRegister(0x1D, 20000);    // V1 = 20000
writeRegister(0x1E, 500);      // A2 = 500 (中速)
writeRegister(0x1F, 50000);    // V2 = 50000
writeRegister(0x20, 200);      // AMAX = 200 (高速小加速)
writeRegister(0x21, 100000);   // VMAX = 100000
writeRegister(0x22, 400);      // DMAX = 400
writeRegister(0x23, 600);      // D2 = 600
writeRegister(0x24, 800);      // D1 = 800 (低速减速可稍急)
writeRegister(0x25, 1000);     // VSTOP = 1000 (不要为0)
writeRegister(0x26, 500);      // TVMAX = 500 (启用jerk抑制)
writeRegister(0x27, 100);      // TZEROWAIT = 100
```

### 2.3 速度换算

```
速度[Hz] = 寄存器值 × fCLK / 2^24
加速度[Hz/s] = 寄存器值 × fCLK² / 2^41

fCLK = 12MHz (典型值)
VMAX = 100000 → v = 100000 × 12M / 2^24 ≈ 71,582 steps/s
```

### 2.4 抖动优化技巧

| 场景 | 优化方法 |
|------|---------|
| 启动抖动 | 增大 VSTART（跳过低速共振区）|
| 停止抖动 | 减小 VSTOP，确保不为 0 |
| 高速抖动 | 降低 AMAX，减小加速度 |
| 匀速抖动 | 增大 TVMAX，增加匀速段过渡 |
| 换向抖动 | 增大 TZEROWAIT |

## 3、限位设置

### 3.1 相关寄存器

| 寄存器 | 地址 | 说明 |
|--------|------|------|
| SW_MODE | 0x36 | 限位开关模式配置 |
| RGATE_TMC5272 | 0x38 | 限位gate配置 |
| ALM_CONFIG | 0x39 | 报警配置 |
| ALM_REFERENCES | 0x3A | 报警参考 |

### 3.2 限位模式配置

```c
// SW_MODE 寄存器
// [0] - left_stop_enable：左限位停止使能
// [1] - right_stop_enable：右限位停止使能
// [2] - stop_latch_enable：停止锁存使能
// [3] - ladder_active： ladders 激活
// [4] - latch_l_active：左锁存激活
// [5] - latch_r_active：右锁存激活
// [6] - invert_left：左限位反相
// [7] - invert_right：右限位反相

// 典型配置：左限位停止，右限位停止，启用锁存
writeRegister(0x36, 0x07);  // 0000 0111
```

### 3.3 限位读取

```c
// 读取限位状态
uint32_t status = readRegister(0x34);  // NODE_STEP_STATUS

// status[0] - left_latch：左限位已锁存
// status[1] - right_latch：右限位已锁存
// status[2] - left_active：左限位激活
// status[3] - right_active：右限位激活
// status[4] - left_stop：左限位已停止
// status[5] - right_stop：右限位已停止

if (status & 0x04) {
    // 左限位触发
}
if (status & 0x08) {
    // 右限位触发
}
```

### 3.4 软限位（软件实现）

```c
// 软件限位检查
void move_to_position(uint32_t target) {
    uint32_t xactual = readRegister(0x21);  // XACTUAL

    // 检查限位
    if (target > xactual && xactual >= max_position) {
        target = max_position;  // 右限位
    }
    if (target < xactual && xactual <= min_position) {
        target = min_position;  // 左限位
    }

    writeRegister(0x28, target);  // XTARGET
}

// 归零操作
void find_home() {
    // 1. 高速向一个方向运动
    writeRegister(0x21, 0x01);  // RAMPMODE = velocity positive
    writeRegister(0x21, 50000); // VMAX

    // 2. 等待限位触发
    while (!(readRegister(0x34) & 0x04));

    // 3. 停止
    writeRegister(0x28, readRegister(0x21));  // XTARGET = XACTUAL

    // 4. 反向一小段离开限位
    // ... 略
}
```

## 4、运动轨迹

### 4.1 速度模式

适用于连续旋转场景（如传送带）：

```c
// 速度模式配置
writeRegister(0x00, 0x00);   // GCONF = 0
writeRegister(0x21, 0x01);   // RAMPMODE = velocity positive

// 设置速度和加速度
writeRegister(0x20, 1000);   // AMAX
writeRegister(0x21, 100000); // VMAX

// 停止：设置 VMAX = 0
writeRegister(0x21, 0);

// 改变方向
writeRegister(0x21, 0x02);   // RAMPMODE = velocity negative
writeRegister(0x21, 100000); // VMAX
```

### 4.2 位置模式

适用于精确定位场景（如云台）：

```c
// 位置模式配置
writeRegister(0x21, 0x00);   // RAMPMODE = positioning

// 配置八点斜坡
writeRegister(0x1B, 0);       // VSTART
writeRegister(0x1C, 500);     // A1
writeRegister(0x1D, 20000);   // V1
writeRegister(0x1E, 300);     // A2
writeRegister(0x1F, 50000);   // V2
writeRegister(0x20, 200);     // AMAX
writeRegister(0x21, 100000);  // VMAX
writeRegister(0x22, 400);     // DMAX
writeRegister(0x23, 600);     // D2
writeRegister(0x24, 800);     // D1
writeRegister(0x25, 1000);    // VSTOP
writeRegister(0x26, 0);       // TVMAX
writeRegister(0x27, 100);     // TZEROWAIT

// 设置目标位置
writeRegister(0x28, 500000);  // XTARGET

// 读取当前位置
uint32_t pos = readRegister(0x21);  // XACTUAL
```

### 4.3 运动状态监控

```c
// 读取运动状态
uint32_t vactual = readRegister(0x22);  // VACTUAL - 实际速度
uint32_t xactual = readRegister(0x21);  // XACTUAL - 实际位置
uint32_t rampstat = readRegister(0x35); // RAMP_STAT - 斜坡状态

// rampstat 位定义
// [0] - position_reached：位置到达
// [1] - velocity_reached：速度到达
// [2] - event_pos_reached：位置事件
// [3] - event_sg_reached：stallGuard 事件
// [4] - event_tz_reached：TZEROWAIT 事件
// [5] - second_move：第二次运动
// [6] - t_zerowait_active：TZEROWAIT 激活

// 等待运动完成
while (!(readRegister(0x35) & 0x01));  // 等待位置到达
```

### 4.4 斜坡状态寄存器详解

| 位 | 名称 | 说明 |
|----|------|------|
| 0 | position_reached | XTARGET 已到达 |
| 1 | velocity_reached | VMAX 已到达 |
| 2 | event_pos_reached | 位置事件标志 |
| 3 | event_sg_reached | stallGuard 事件 |
| 4 | event_tz_reached | TZEROWAIT 结束 |
| 5 | second_move | 第二次运动开始 |
| 6 | t_zerowait_active | 等待反转延迟中 |
| 7 | status_latch_u | 状态锁存（上）|
| 8 | status_latch_d | 状态锁存（下）|

### 4.5 运动轨迹优化

```c
// 1. 平滑换向
void reverse_direction() {
    // 先减速到 VSTOP
    writeRegister(0x21, 0x00);  // 位置模式
    writeRegister(0x21, 1000);  // VSTOP

    // 等待停止
    while (readRegister(0x22) > 100);

    // 等待 TZEROWAIT
    uint32_t rampstat;
    do {
        rampstat = readRegister(0x35);
    } while (rampstat & 0x40);  // t_zerowait_active

    // 反向运动
    writeRegister(0x28, -500000);  // 新目标位置
}

// 2. 运动中改变目标
// TMC5272 支持运动中修改 XTARGET
writeRegister(0x28, new_target);  // 直接写入新目标

// 3. 运动中改变速度
writeRegister(0x21, new_vmax);    // 直接写入新速度
```

## 5、调试工具

### 5.1 寄存器读写

```c
// SPI 读写函数示例
uint32_t tmc5272_read(uint8_t reg) {
    uint8_t tx[5] = {0x00, 0x00, 0x00, 0x00, reg & 0x7F};
    uint8_t rx[5];

    GPIO_LOW(CS);
    HAL_SPI_TransmitReceive(tx, rx, 5);
    GPIO_HIGH(CS);

    return (rx[0] << 24) | (rx[1] << 16) | (rx[2] << 8) | rx[3];
}

void tmc5272_write(uint8_t reg, uint32_t value) {
    uint8_t tx[5] = {0x80, (value >> 24) & 0xFF, (value >> 16) & 0xFF,
                     (value >> 8) & 0xFF, value & 0xFF};

    GPIO_LOW(CS);
    HAL_SPI_Transmit(tx, 5);
    GPIO_HIGH(CS);
}
```

### 5.2 调试检查清单

| 检查项 | 方法 |
|--------|------|
| 供电电压 | 万用表测量 VM 和 VIO |
| SPI 通信 | 读写芯片 ID (0x00)，应为 0x5272 |
| 电机连接 | 测量四路输出波形 |
| 限位开关 | 手动触发，读取状态寄存器 |
| 电流设置 | 示波器测量 sense 电阻 |
| 运动响应 | 发送少量脉冲，检查响应 |
