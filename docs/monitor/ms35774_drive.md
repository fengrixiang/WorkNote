# MS35774 步进电机驱动分析文档

## 1. 概述

MS35774是一款两相步进电机驱动芯片，本驱动基于该芯片实现云台（PTZ）电机控制，支持水平和垂直两个轴向的精确运动控制。

### 基本信息

| 项目 | 值 |
|------|-----|
| 模块名称 | ms35774.ko |
| 设备节点 | /dev/motor_ctrl |
| 字符设备主设备号 | 243 |
| 设备树兼容 | vhd, motor-control |

### 系统架构

```
┌─────────────────────────────────────────────────────────┐
│                    应用层 (motor.c)                     │
│         motor_attr_init / motor_preset / motor_stop    │
└─────────────────────────┬───────────────────────────────┘
                          │ ioctl
┌─────────────────────────▼───────────────────────────────┐
│                  驱动层 (ms35774.ko)                    │
│    motor_ioctl / hrtimer / GPIO控制                    │
└─────────────────────────┬───────────────────────────────┘
                          │ GPIO
┌─────────────────────────▼───────────────────────────────┐
│                   硬件层 (MS35774)                       │
│    step / dir / enable / dete_gpio                      │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 硬件接口

### GPIO定义

| GPIO | 功能 | 方向 | 说明 |
|------|------|------|------|
| step_gpio | 步进脉冲 | 输出 | 每脉冲一步 |
| dir_gpio | 方向控制 | 输出 | 0=正向, 1=反向 |
| enble_gpio | 使能 | 输出 | 低电平使能 |
| dete_gpio | 光耦检测 | 输入 | 用于归零校准 |

### motor.json GPIO配置

| 参数 | CHANNL0 (水平) | CHANNL1 (垂直) |
|------|---------------|---------------|
| enble_gpio | 133 | 134 |
| step_gpio | 130 | 131 |
| dir_gpio | 135 | 139 |
| dete_gpio | 137 | 138 |

---

## 3. 电机规格

### 基本参数

| 参数 | CHANNL0 (水平) | CHANNL1 (垂直) |
|------|---------------|---------------|
| 电机步距角 | 0.9° | 0.9° |
| 减速比 | 4.5 | 8 |
| 驱动芯片细分 | 16 | 16 |
| 等效步距角 | 0.0125° | 0.00703° |
| 角度范围 | 90° | 30° |
| 最大步数 | 7200 | 4266 |
| 档位0转速 | 0.69°/s | 0.89°/s |

### 核心公式

```
等效步距角 = 电机步距角 ÷ 细分 ÷ 减速比
最大步数 = 角度范围 ÷ 等效步距角
步频(steps/s) = 1,000,000 / 定时周期预设值
角速度 = 步频 × 等效步距角
```

### VISCA坐标转换

```c
// VISCA角度 → 电机步数
steps = visca角度 × (max_position - min_position) / (max_angle - min_angle)

// CHANNL0: 90° / 7200步 = 80步/度
// CHANNL1: 30° / 4266步 = 142.2步/度
```

---

## 4. 定时器与脉冲

### 定时器时钟

- **定时器时钟 = 1MHz**（每计数周期 = 1μs）
- 脉冲周期 = speed × 1000ns = speed × 1μs
- 高电平时间 = speed × 500ns

### 脉冲时序（双脉冲机制）

```
         ┌─────────────────────────────────────┐
step     │ 高电平(speed×500ns) │ 低电平(speed×500ns) │
─────────┘                     └─────────────────────┘
         |<──── 高电平 ────→|    |<──── 低电平 ────→|
                               ↑
                      执行速度调节

每步时间 = speed × 1000ns = speed × 1μs
2次定时器周期 = 1个完整步进脉冲
```

### 速度档位对照

| 档位 | CHANNL0 定时周期(μs) | CHANNL1 定时周期(μs) |
|------|---------------------|---------------------|
| 0 | 18017 | 7883 |
| 5 | 1126 | 493 |
| 10 | 581 | 254 |
| 15 | 392 | 171 |
| 20 | 296 | 129 |
| 23 | 257 | 113 |

### 三种速度表

| 特性 | 普通速度 | 跟踪速度 | 会议跟踪 |
|------|---------|---------|----------|
| max_speed | 24 | 48 | 96 |
| 适用场景 | 常规操作 | 教育跟踪 | 会议跟踪 |

---

## 5. 数据结构

### struct motor_ctrl (电机控制器)

```c
struct motor_ctrl {
    struct gpio_desc *enble_desc;  // 使能GPIO
    struct gpio_desc *step_desc;   // 步进脉冲GPIO
    struct gpio_desc *dir_desc;    // 方向GPIO
    struct gpio_desc *dete_desc;   // 光耦检测GPIO
    struct hrtimer hrt;             // 高精度定时器
    struct motor_attr motor;         // 电机属性
    struct motor_target target;      // 当前目标
    struct motor_target target_set; // 设置的目标
    struct mutex lock;              // 互斥锁
    u8 step_status;                 // GPIO输出状态
    u8 init_flag;                   // 初始化标志
    u8 dete_flag;                   // 光耦触发标志
    u8 status;                      // 电机状态
    wait_queue_head_t wait;         // 等待队列
    bool done;                      // 完成标志
};
```

### struct motor_attr (电机属性)

```c
struct motor_attr {
    u8  isMove;               // 运动状态
    u8  dir;                  // 当前方向
    u16 speed;                // 当前速度(定时器预设值)
    u32 position;             // 当前位置(步数)
    u32 min_position;         // 最小位置
    u32 max_position;         // 最大位置
    u32 min_position_limit;    // 软件限位
    u32 max_position_limit;   // 软件限位

    u16 *speed_list;          // 速度表
    u8  *acc_list;            // 加速表
    u8  *slow_acc_list;       // 慢速加速表
    u32 *acc_distance;        // 加速距离表
    u32 *slow_distance;       // 减速距离表

    u8  max_speed;            // 最大速度等级
    u8  track_max_speed;      // 跟踪模式最大速度
    u8  person_track_max_speed; // 会议跟踪最大速度
    u8  stop_stage;           // 可立即停止的速度等级
    u8  speed_stage;          // 当前速度等级
    u8  count;                // 加减速计数器
};
```

### struct motor_target (运动目标)

```c
struct motor_target {
    u8  motor_num;         // 电机编号: 0=水平, 1=垂直
    u8  moveMode;          // 运动模式: 0=停止, 1=方向, 2=预置位
    u8  dirTarget;         // 目标方向
    u8  speedTarget;       // 目标速度等级
    u32 positionTarget;    // 目标位置
    u8  accel_rate_switch; // 加减速曲线: 0=普通, 1=慢速
    u32 cntTarget;         // 校准步数目标
};
```

---

## 6. S曲线加减速

### 核心概念

```
加速: count从0开始，每次+1，达到acc_list[speed_stage]时升档
减速: count从acc_list[speed_stage]开始，每次-1，减到0时降档
```

### acc_list 加速表

**acc_list[i] = 从等级i跳到等级i+1需要的脉冲数**

| 电机 | acc_list特点 |
|------|-------------|
| CHANNL0 (水平) | 全部为1，全速切换 |
| CHANNL1 (垂直) | acc_list[0]=4，起始加速较缓 |

**例**: acc_list[0] = 4 表示在等级0需要4个脉冲才能跳到等级1

### acc_list配置示例

**CHANNL0 (acc全为1)**
```
acc: [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]
每1个脉冲跳一级，加速非常快
```

**CHANNL1 (acc第0级为4)**
```
acc: [4,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3]
第0级需要4个脉冲（加速较平缓），其他级别需要3个脉冲
```

### 加减速示意图

```
速度
  ^
  |        /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
  |       /
  |      /
  |     /
  |    /
  |   /
  |  /___________________
  |
  +-------------------------> 时间
    加速    匀速    减速
```

### 线性插值公式

```
// 加速时（从等级N到N+1）
当前速度 = speed_list[N] - (speed_list[N] - speed_list[N+1]) × count / acc_list[N]
```

**例：CHANNL0 档位0升档过程**
```
speed_list[0] = 18017, speed_list[1] = 4504, acc_list[0] = 4

count=0: speed = 18017 - (18017-4504)×0/4 = 18017
count=1: speed = 18017 - (18017-4504)×1/4 = 15888.25
count=2: speed = 18017 - (18017-4504)×2/4 = 13760.5
count=3: speed = 18017 - (18017-4504)×3/4 = 11632.75
count=4: count >= acc_list[0]，升入档位1，speed = 4504
```

### 加速等级切换流程

```
初始: speed_stage=0, count=0, speed=speed_list[0]

第1次: count=1 < acc_list[0] → 插值过渡
第2次: count=2 < acc_list[0] → 插值过渡
第3次: count=3 < acc_list[0] → 插值过渡
第4次: count=4 >= acc_list[0] → speed_stage=1, count=0 → 跳入等级1
第5次: count=1 < acc_list[1] → 插值过渡
...
```

### slow_acc_list (慢加速)

当`accel_rate_switch=1`时使用slow_acc_list替代acc_list，实现更平缓的加减速曲线，通常用于精细位置控制场景。

### 减速距离预计算

**calc_slow_down_distance()** 提前计算从各速度等级减速到stop_stage需要的总距离，存入slow_distance[]，减少中断处理时间。

### 加速 vs 减速

| 特性 | 加速(motor_acc) | 减速(motor_stop) |
|------|-----------------|------------------|
| count初始值 | 0 | acc_list[speed_stage] |
| count变化 | 每次+1 | 每次-1 |
| 档位变化 | 每次+1 | 每次-1 |
| 结束条件 | 达到目标档位 | count减到0 |

### accel_rate_switch

| 值 | 使用加速表 | 适用场景 |
|----|-----------|---------|
| 0 | acc_list | 普通运动 |
| 1 | slow_acc_list | 精细控制 |

### stop_stage

motor.json配置：
- CHANNL0: stop_stage=0 (任何速度都需要减速停止)
- CHANNL1: stop_stage=1 (档位0可直接停止)

---

## 7. 核心函数

### motor_timer_func (定时器中断)

**代码**: [motor_ctrl.c:435-502](module/motor-ctrl/motor_ctrl.c#L435-L502)

```c
static enum hrtimer_restart motor_timer_func(struct hrtimer *timer)
{
    struct motor_ctrl *motor_ptr = container_of(timer, struct motor_ctrl, hrt);
    int ret = 0;

    if (motor_ptr->init_flag == 0)
        return HRTIMER_NORESTART;

    // 翻转step GPIO (双脉冲机制)
    if (motor_ptr->step_status)
        gpiod_set_value(motor_ptr->step_desc, 0);
    else
        gpiod_set_value(motor_ptr->step_desc, 1);

    // 仅在step_status=0时执行速度调节
    if (!motor_ptr->step_status)
    {
        switch (motor_ptr->status)
        {
        case INIT_STATUS:     ret = motor_init_fun(motor_ptr); break;
        case CALIBRATE_STATUS: ret = motor_cali_fun(motor_ptr); break;
        case PRESET_STATUS:
        case MOVE_STATUS:     ret = motor_move_fun(motor_ptr); break;
        }
    }

    if (!ret)
    {
        hrtimer_forward_now(timer, ns_to_ktime(motor_ptr->motor.speed * 500));
        return HRTIMER_RESTART;
    }
    else
    {
        // 运动完成
        motor_ptr->motor.isMove = 0;
        motor_ptr->motor.speed_stage = 0;
        motor_ptr->motor.count = 0;
        motor_ptr->done = 1;
        wake_up(&motor_ptr->wait);
    }
    return HRTIMER_NORESTART;
}
```

### motor_acc (加速函数)

**代码**: [motor_ctrl.c:220-259](module/motor-ctrl/motor_ctrl.c#L220-L259)

```c
static void motor_acc(struct motor_attr *motor, u8 speedTarget,
                      u8 accel_rate_switch, struct motor_ctrl *ctrl)
{
    u32 temp;
    u8 *acc_list = accel_rate_switch ? motor->slow_acc_list : motor->acc_list;

    if (motor->speed_stage >= speedTarget)
    {
        motor->speed = motor->speed_list[motor->speed_stage];
        return;
    }

    if (motor->count >= acc_list[motor->speed_stage])
    {
        motor->count = 0;
        if (motor->speed_stage < speedTarget)
            motor->speed_stage++;
        motor->speed = motor->speed_list[motor->speed_stage];
    }
    else
    {
        temp = motor->speed_list[motor->speed_stage] - motor->speed_list[motor->speed_stage + 1];
        temp = temp * motor->count / acc_list[motor->speed_stage];
        motor->speed = motor->speed_list[motor->speed_stage] - temp;
    }
    motor->count++;
}
```

### motor_stop (减速函数)

**代码**: [motor_ctrl.c:90-155](module/motor-ctrl/motor_ctrl.c#L90-L155)

```c
static int motor_stop(struct motor_attr *motor, struct motor_target *target,
                      u8 mode, u8 accel_rate_switch, struct motor_ctrl *ctrl)
{
    u8 stop_stage = (mode == 1 && target->speedTarget < motor->stop_stage)
                    ? target->speedTarget : motor->stop_stage;

    if (motor->speed >= motor->speed_list[stop_stage])
        return 1;  // 已达最低速

    u8 *acc_list = accel_rate_switch ? motor->slow_acc_list : motor->acc_list;

    if (motor->count <= 0)
    {
        motor->count = acc_list[motor->speed_stage];
        if (motor->speed_stage)
            motor->speed_stage--;
        else
            return 1;
    }
    else
    {
        motor->count--;
    }
    return 0;
}
```

### motor_move_fun (运动处理)

**代码**: [motor_ctrl.c:296-400](module/motor-ctrl/motor_ctrl.c#L296-L400)

```c
static int motor_move_fun(struct motor_ctrl *motor_ptr)
{
    u32 distance;
    Step_Counter(&motor_ptr->motor);

    // 边界检测
    if (motor_ptr->target.moveMode == 0)  // 停止
        return motor_stop(...);
    else if (motor_ptr->target.moveMode == 1)  // 方向运动
    {
        distance = motor_ptr->motor.acc_distance[motor_ptr->motor.speed_stage];
        if (motor_ptr->motor.dir == 1)
        {
            if (distance + motor_ptr->motor.position >= motor_ptr->motor.max_position_limit
                || motor_ptr->target.speedTarget < motor_ptr->motor.speed_stage)
                return motor_stop(...);
        }
        motor_acc(...);
    }
    else  // 预置位运动
    {
        if ((motor_ptr->motor.dir == 1 && motor_ptr->motor.position >= motor_ptr->target.positionTarget)
            || (motor_ptr->motor.dir == 0 && motor_ptr->motor.position <= motor_ptr->target.positionTarget))
            return motor_stop(...);
        motor_acc(...);
    }
    return 0;
}
```

---

## 8. 运动模式

| moveMode | 模式 | 说明 |
|----------|------|------|
| 0 | 停止 | S曲线减速停止 |
| 1 | 方向运动 | 沿设定方向运行至限位 |
| 2 | 预置位 | 运行至指定位置 |

### 状态机

```
INIT_STATUS (0)
      ↓
CALIBRATE_STATUS (4) ←→ 光耦触发检测
      ↓
PRESET_STATUS (1) ←→ 预置位运动
      ↓
MOVE_STATUS (2) ←→ 正常运行
      ↓
READY_STATUS (3) ←→ 就绪
```

---

## 9. ioctl命令

| 命令 | 说明 | 参数类型 |
|------|------|----------|
| MOTOR_WR_MOTOR_ATTR | 初始化电机属性 | motor_init_attr |
| MOTOR_MOTOR_INIT | 复位电机 | motor_target |
| MOTOR_MOTOR_CALI | 光耦校准 | motor_target |
| MOTOR_MOTOR_CTRL | 运动控制 | motor_target |
| MOTOR_MOTOR_STATUS | 查询状态 | motor_status |
| MOTOR_MOTOR_WAIT | 等待完成 | motor_status |
| MOTOR_MOTOR_EXIT | 退出 | - |
| MOTOR_MOTOR_LIMIT | 设置限位 | motor_limit |
| MOTOR_SW_SPEED | 切换速度表 | num高4位选择速度表 |

---

## 10. 电机状态机详解

### 初始化流程 (INIT_STATUS)

1. 电机往光耦反方向运动cnt_target步
2. 使用motor_init_fun()，调用motor_acc()加速
3. 到达步数后进入CALIBRATE_STATUS

### 校准流程 (CALIBRATE_STATUS)

1. 电机往光耦方向运动
2. 检测dete_gpio状态
3. 光耦触发后调用motor_stop()减速停止
4. 进入PRESET_STATUS

### 预置位/运动流程 (PRESET_STATUS / MOVE_STATUS)

1. 调用motor_move_fun()
2. 根据moveMode执行方向运动或预置位运动
3. 接近目标位置时调用motor_stop()减速

---

## 11. 距离预计算

### calc_slow_down_distance

**代码**: [motor_ctrl.c:157-176](module/motor-ctrl/motor_ctrl.c#L157-L176)

```c
static void calc_slow_down_distance(struct motor_attr *motor)
{
    motor->slow_distance[0] = motor->stop_stage > 0 ? 0 : motor->slow_acc_list[0];
    motor->acc_distance[0] = motor->acc_list[0];

    for (int i = 1; i < motor->max_speed; i++)
    {
        if (i < motor->stop_stage)
            motor->slow_distance[i] = 0;
        else
        {
            motor->slow_distance[i] = motor->slow_acc_list[i] + motor->slow_distance[i - 1];
            motor->acc_distance[i] = motor->acc_list[i] + motor->acc_distance[i - 1];
        }
    }
}
```

**acc_distance[i]**: 从档位i加速到max_speed需要的总步数
**slow_distance[i]**: 从档位i减速到stop_stage需要的总步数

**用途**: 运动中提前判断是否需要减速，避免过冲

---

## 12. 应用层架构

### 主要函数

| 函数 | 说明 |
|------|------|
| motor_35774_init() | MS35774初始化入口 |
| motor_attr_init() | 解析motor.json |
| motor_init_all() | 初始化两个电机 |
| motor_preset() | 双电机预置位 |
| vhd_motor_preset() | 单电机预置位 |

### 文件列表

| 路径 | 说明 |
|------|------|
| module/motor-ctrl/motor_ctrl.c | 驱动源文件 |
| module/motor-ctrl/motor_ctrl.h | 驱动头文件 |
| include/motor_ctrl.h | 应用层接口 |
| lib_source/libvhdcomm/motor_ctrl.c | 应用层库 |
| app/vx600_main/src/motor/motor.c | 应用层实现 |
| config/json/motor.json | 配置文件 |

---

## 13. 使用示例

### 初始化电机
```c
struct motor_init_attr init = {
    .motor_num = 0,
    .enble_gpio = 133,
    .step_gpio = 130,
    .dir_gpio = 135,
    .dete_gpio = 137,
    .min_position = 0,
    .max_position = 7200,
    .max_speed = 24,
    // ... 速度表配置
};
ioctl(fd, MOTOR_WR_MOTOR_ATTR, &init);
```

### 预置位运动
```c
struct motor_target target = {
    .motor_num = 0,
    .moveMode = 2,
    .speedTarget = 20,
    .positionTarget = 3600,
};
ioctl(fd, MOTOR_MOTOR_CTRL, &target);
```

### 停止电机
```c
struct motor_target stop = {
    .motor_num = 0,
    .moveMode = 0,
};
ioctl(fd, MOTOR_MOTOR_CTRL, &stop);
```

### 查询状态
```c
struct motor_status status = { .motor_num = 0 };
ioctl(fd, MOTOR_MOTOR_STATUS, &status);
printf("Moving: %d, Position: %d\n", status.isMove, status.position);
```

---

## 14. 附录：速度计算示例

### 档位0定时周期计算

```
步频 = 55.50 steps/s
定时周期 = 1,000,000 ÷ 55.50 = 18,017 μs/步
```

### 角速度验算

```
档位0 水平角速度 = 55.50 steps/s × 0.0125°/step = 0.69°/s
档位23 水平角速度 = 3886.01 steps/s × 0.0125°/step = 48.58°/s
```

### 水平与垂直电机差异

| 参数 | 水平 (CHANNL0) | 垂直 (CHANNL1) |
|------|---------------|---------------|
| 减速比 | 4.5 | 8 |
| 等效步距角 | 0.0125° | 0.00703° |
| 档位0转速 | 0.69°/s | 0.89°/s |

**原因**: 垂直电机减速比更大，等效步距角更小，同样步频下角速度更大。


## 15. 加减速时间计算

### 基本原理

1. **双脉冲机制**: 每个步进脉冲由2次定时器中断组成（高电平+低电平）
2. **motor_acc调用**: 仅在step_status=0时执行，即每步调用1次
3. **速度插值**: 每步speed在线性插值变化，从speed_list[i]渐变到speed_list[i+1]
4. **每步耗时**: speed × 500ns × 2 = speed × 1μs

### 速度插值过程

加速阶段，motor_acc在档位i的插值过程：
```
count=0: speed = speed_list[i]
count=1: speed = speed_list[i] - (speed_list[i] - speed_list[i+1]) × 1 / acc_list[i]
count=2: speed = speed_list[i] - (speed_list[i] - speed_list[i+1]) × 2 / acc_list[i]
...
count=acc_list[i]-1: speed ≈ speed_list[i+1]（最后一次插值）
count=acc_list[i]: 升档，speed = speed_list[i+1]
```

由于speed在每步之间变化，每步耗时不同，因此需要用等差数列求和。

### 核心公式

**单档位过渡时间**（精确，基于等差数列求和）：
```
T_i = acc_list[i] × (speed_list[i] + speed_list[i+1])    (μs)
```

推导：
```
speed_k = speed_list[i] - (speed_list[i] - speed_list[i+1]) × k / acc_list[i]
每步耗时 = speed_k × 1 μs
T_i = Σ speed_k  (k = 0 ~ acc_list[i]-1)
    = acc_list[i] × speed_list[i] - (speed_list[i]-speed_list[i+1]) × (0+1+...+(acc_list[i]-1)) / acc_list[i]
    = acc_list[i] × (speed_list[i] + speed_list[i+1]) / 2 × 2
    = acc_list[i] × (speed_list[i] + speed_list[i+1])
```

**从0档加速到N档的总时间**：
```
T_total = Σ acc_list[i] × (speed_list[i] + speed_list[i+1])    (i = 0 到 N-1, 单位μs)
```

### CHANNL0 计算示例

**已知条件**:
- acc_list全为1
- speed_list: [18017, 4504, 2574, 1802, 1386, 1126, 948, 819, 721, 644, 581, 530, 487, 450, 419, 392, 368, 347, 328, 311, 296, 281, 269]

**逐阶段计算**:

| 阶段 | speed_list[i] | speed_list[i+1] | acc | T_i (μs) |
|------|-------------|-----------------|-----|----------|
| 0→1 | 18017 | 4504 | 1 | 22521 |
| 1→2 | 4504 | 2574 | 1 | 7078 |
| 2→3 | 2574 | 1802 | 1 | 4376 |
| 3→4 | 1802 | 1386 | 1 | 3188 |
| 4→5 | 1386 | 1126 | 1 | 2512 |
| 5→6 | 1126 | 948 | 1 | 2074 |
| 6→7 | 948 | 819 | 1 | 1767 |
| 7→8 | 819 | 721 | 1 | 1540 |
| 8→9 | 721 | 644 | 1 | 1365 |
| 9→10 | 644 | 581 | 1 | 1225 |
| 10→11 | 581 | 530 | 1 | 1111 |
| 11→12 | 530 | 487 | 1 | 1017 |
| 12→13 | 487 | 450 | 1 | 937 |
| 13→14 | 450 | 419 | 1 | 869 |
| 14→15 | 419 | 392 | 1 | 811 |
| 15→16 | 392 | 368 | 1 | 760 |
| 16→17 | 368 | 347 | 1 | 715 |
| 17→18 | 347 | 328 | 1 | 675 |
| 18→19 | 328 | 311 | 1 | 639 |
| 19→20 | 311 | 296 | 1 | 607 |
| 20→21 | 296 | 281 | 1 | 577 |
| 21→22 | 281 | 269 | 1 | 550 |
| 22→23 | 269 | 257 | 1 | 526 |

**加速到档位5**:
```
T = 22521 + 7078 + 4376 + 3188 + 2512 = 39675 μs ≈ 40 ms
```

**加速到档位23 (全程)**:
```
T = 22521+7078+4376+3188+2512+2074+1767+1540+1365+1225
    +1111+1017+937+869+811+760+715+675+639+607+577+550+526
  = 56259 μs ≈ 56 ms
```

### CHANNL1 计算示例

**已知条件**:
- acc_list[0]=4, acc_list[1~22]=3
- speed_list: [15766, 3942, 2252, 1576, 1212, 989, 842, 733, 648, 579, 523, 478, 440, 409, 383, 360, 341, 324, 309, 296, 284, 274, 265]

**逐阶段计算**:

| 阶段 | speed_list[i] | speed_list[i+1] | acc | T_i (μs) |
|------|-------------|-----------------|-----|----------|
| 0→1 | 15766 | 3942 | 4 | 78832 |
| 1→2 | 3942 | 2252 | 3 | 18582 |
| 2→3 | 2252 | 1576 | 3 | 11484 |
| 3→4 | 1576 | 1212 | 3 | 8364 |
| 4→5 | 1212 | 989 | 3 | 6603 |
| 5~22 | ... | ... | 3 | 各约500~1700 |

**加速到档位5**:
```
T = 78832 + 18582 + 11484 + 8364 + 6603 = 123865 μs ≈ 124 ms
```

### 减速时间计算

减速与加速对称，motor_stop中count从acc_list[speed_stage]递减到0，速度线性插值递增（即减速）。

**从档位N减速到档位M的总时间**：
```
T_decel = Σ acc_list[i] × (speed_list[i] + speed_list[i+1])    (i = M 到 N-1, 单位μs)
```

减速与加速经过相同的档位时，时间完全相同。

### 各档位加速时间一览

**CHANNL0** (acc_list全为1):

| 档位 | 累积时间 | 说明 |
|------|--------|------|
| 0→1 | 22.5 ms | 起始档 |
| 0→5 | 39.7 ms | 常规速度 |
| 0→10 | 50.8 ms | 中速 |
| 0→20 | 55.8 ms | 高速 |
| 0→23 | 56.3 ms | 全速 |

**CHANNL1** (acc_list[0]=4, 其他=3):

| 档位 | 累积时间 | 说明 |
|------|--------|------|
| 0→1 | 78.8 ms | 起始档(较慢) |
| 0→5 | 123.9 ms | 常规速度 |
| 0→10 | 148.2 ms | 中速 |
| 0→20 | 168.5 ms | 高速 |
| 0→23 | 176.0 ms | 全速 |


### 影响加减速时间的因素

| 因素 | 影响 | 调整方式 |
|------|------|---------|
| acc_list[i] | 值越大，该档位停留越久，加速越慢 | 增大=加速更平缓 |
| speed_list[i] | 值越大，定时周期越长，每步越慢 | 调整速度表 |
| 目标档位 | 越高，累积阶段越多 | 由应用层设定 |
| slow_acc_list | 慢速加速表，值更大 | accel_rate_switch=1时使用 |
| 减速比 | 影响角速度，不影响加减速时间 | 硬件固定 |

### 实际应用

1. **预估运动时间**: 总时间 = 加速时间 + 匀速时间 + 减速时间
2. **调速参考**: 调整acc_list可改变加减速平缓程度
3. **位置控制**: 了解减速距离(slow_distance)可避免过冲
4. **速度表切换**: 通过MOTOR_SW_SPEED命令在普通/跟踪/会议跟踪三套速度表间切换