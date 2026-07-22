# DDR 压测方案

> 基于 `memtester` 与 `stressapptest` 的内存稳定性测试指南，适用于 ARM SoC 平台（CV72 / hi3559 / RK3588 / RK3576 / 268g 等）量产前的 DDR 验证。

---

## 1. 方案概述

### 1.1 测试目的

DDR 是嵌入式系统中故障率最高的器件之一，常见的失效模式包括：

- **硬件缺陷**：颗粒虚焊、PCB 走线信号完整性差、阻抗不匹配
- **时序裕量不足**：频率升高后读写眼图收缩
- **温升失效**：高温下数据翻转错误率上升
- **软件压测覆盖不足**：业务负载路径无法触达所有 Bank / Row

本方案通过对 DDR 进行**长时间、高带宽、多 pattern**的压测，尽早暴露上述问题。

### 1.2 测试工具对比

| 维度 | memtester | stressapptest |
|------|-----------|---------------|
| 作者 | Martin Cracauer（开源） | Google（开源） |
| 主要用途 | 单元级内存 bit 翻转测试 | 系统级高带宽压力测试 |
| 测试模式 | Stuck Address、Random Value、Solid Bits、Bit Flip、SUB、Sequential、Block Sequential 等 10+ 种算法 | 内存拷贝、校验和校验、磁盘 IO 联动 |
| 并行能力 | 单进程、可多实例 | 多线程，自动利用多通道 |
| 典型场景 | 验证颗粒本身质量、工厂产线 | 验证 SoC 控制器 + DDR 通路整体稳定性 |
| 占用内存 | 申请多少测多少（可控） | 占用物理内存 80%+ |
| 适合平台 | 用户态工具，无需驱动 | 用户态工具，无需驱动 |

> **建议**：量产前必须两个工具都跑。先 memtester 跑颗粒质量，再用 stressapptest 跑业务场景。

---

## 2. memtester 测试方案

### 2.1 命令格式

```
memtester [-p PHYSADDR] MEMSIZE[M/G] [ITERATIONS]
```

常用参数：

| 参数 | 说明 |
|------|------|
| `-p <addr>` | 从指定物理地址开始测试（用于特定地址段验证） |
| `MEMSIZE` | 测试内存大小，建议单位 `M` 或 `G` |
| `ITERATIONS` | 循环次数（默认无限） |

### 2.2 测试算法（按顺序执行）

memtester 内部按以下顺序循环执行 13 种 pattern：

1. **Stuck Address**：检测地址线粘连、颗粒虚焊
2. **Random Value**：写入随机值再读回比对
3. **Compare XOR**：异或比较，校验每一位
4. **Compare SUB**：减法比较
5. **Compare MUL**：乘法比较
6. **Compare DIV**：除法比较
7. **Compare OR**：或运算比较
8. **Compare AND**：与运算比较
9. **Sequential Increment**：地址顺序递增扫描
10. **Solid Bits**：全 0 / 全 1 写入
11. **Block Sequential**：分块顺序扫描
12. **Checkerboard**：棋盘格 pattern
13. **Bit Spread**：位分散 pattern
14. **Bit Flip**：位翻转
15. **Walking Ones / Walking Zeros**：走位 1 / 走位 0

### 2.3 推荐测试用例

#### 用例 1：基础全内存扫描（产线必跑）

```bash
# 单次扫描整片 DDR
memtester 2G 1
```

- **场景**：产线下线检测，覆盖整片 DDR
- **耗时**：2G 大约 3~8 分钟（视 DDR 频率）
- **判定**：所有 pattern 无 FAIL、无 ECC error

#### 用例 2：颗粒质量验证（多轮）

```bash
# 5 轮循环，更严格
memtester 2G 5
```

- **场景**：来料检验、样品试产
- **耗时**：2G × 5 ≈ 30 分钟
- **判定**：任何一轮出现错误即不合格

#### 用例 3：高温运行测试

```bash
# 高温箱 85℃ 环境下连续运行 12 小时
memtester 4G &
PID=$!
while true; do
    TEMP=$(cat /sys/class/thermal/thermal_zone0/temp)
    echo "$(date) TEMP=${TEMP}" >> /tmp/memtest.log
    kill -USR1 $PID 2>/dev/null  # memtester 收到 USR1 会打印进度
    sleep 60
done
```

#### 用例 4：指定地址段测试

```bash
# 验证 kernel reserved 区域（假设保留区在 0x100000000）
memtester -p 0x100000000 256M 1
```

#### 用例 5：CPU 亲和性绑定（多核平台）

```bash
# 在 RK3588（4×A76 + 4×A55）上每个核跑一个实例，并行覆盖
for cpu in /sys/devices/system/cpu/cpu[0-9]; do
    cpuid=$(basename $cpu | sed 's/cpu//')
    taskset -c $cpuid memtester 512M 10 &
done
wait
```

### 2.4 memtester 输出解读

```
Loop 1/1 (1):
  Stuck Address: ok
  Random Value: ok
  Compare XOR: ok
  ...
  Bit Flip: ok
```

- **`ok`**：本轮该 pattern 通过
- **`FAILURE`**：发现错误，会打印错误地址、期望值、实际值
- **`ECC error`**：触发硬件 ECC 修正（记录到日志，分析频率）

### 2.5 常见问题

| 现象 | 可能原因 | 排查方向 |
|------|----------|----------|
| 随机值测试 FAIL | 颗粒虚焊 / 频率过高 | 检查焊接、降低 DDR 频率 |
| 高温下才出错 | 时序裕量不足 | 调松 tRFC/tRP/tRCD |
| 某 Bank 区域 FAIL | PCB 走线问题 | 检查 SI、检查该 Bank 物料 |
| ECC error 频发 | 颗粒品质 | 更换颗粒批次 |

---

## 3. stressapptest 测试方案

### 3.1 命令格式

```
stressapptest [-h] [-s N] [-M N] [-m N] [-W N] [-n threads] [-D pages] [-f file]
```

| 参数 | 说明 | 默认 |
|------|------|------|
| `-s N` | 测试时长（秒） | 20 |
| `-M N` | 测试内存大小（M） | 物理内存可用量 |
| `-m N` | 单次 memcpy 大小（字节） | 262144 |
| `-W N` | memcpy 拷贝次数后做校验和 | 524288 |
| `-n N` | 并发线程数 | 核数 × 4 |
| `-D N` | 磁盘线程写入的页数 | 0（不测磁盘） |
| `-f N` | 磁盘 IO 的文件路径 | /dev/shm/tmpfile |
| `-l logfile` | 输出日志 | stderr |
| `-v N` | 详细级别 1~5 | 1 |
| `--max_errors N` | 错误上限，达到即停 | 默认 128 |

### 3.2 推荐测试用例

#### 用例 1：标准 4 小时烤机

```bash
stressapptest -s 14400 -M 2048 -m 8 -W 8 -n 16 -l /tmp/stress.log -v 3
```

- **场景**：正式压测、版本发布前
- **耗时**：4 小时
- **覆盖**：8 核 × 2 线程，并发 memcpy + 校验
- **判定**：无 FAIL、无严重 log

#### 用例 2：高并发极限测试

```bash
# 占满全部物理内存的 85%
TOTAL_MEM=$(free -m | awk '/Mem:/{print $2}')
stressapptest -s 7200 -M $((TOTAL_MEM * 85 / 100)) -n 32 -l /tmp/stress_hi.log -v 4
```

- **场景**：压力极限验证、多核性能基线
- **注意**：必须保留足够内存给 kernel，否则触发 OOM

#### 用例 3：CPU + DDR + 磁盘联动

```bash
stressapptest -s 3600 \
              -M 1024 \
              -D 4096 \
              -f /mnt/sdcard/tmpfile \
              -n 8 \
              -l /tmp/stress_io.log \
              -v 4
```

- **场景**：验证 DDR + eMMC/SD 卡联动下的稳定性
- **注意**：磁盘 IO 会降低 DDR 测试压力，根据场景选择

#### 用例 4：长时间生产烤机（72h）

```bash
nohup stressapptest -s 259200 -M 1024 -n 16 -l /var/log/stress_72h.log -v 2 &
echo $! > /tmp/stress.pid

# 配套的温度/日志监控脚本
while kill -0 $(cat /tmp/stress.pid) 2>/dev/null; do
    DATE=$(date +%Y-%m-%d_%H:%M:%S)
    TEMP=$(cat /sys/class/thermal/thermal_zone0/temp 2>/dev/null)
    echo "$DATE TEMP=${TEMP}" >> /tmp/stress_monitor.log
    sleep 300
done
```

#### 用例 5：CPU 亲和性分配

```bash
# 让线程绑定到指定 CPU，模拟特定业务核的使用情况
stressapptest -s 3600 -M 2048 -n 8 -l /tmp/stress_aff.log -v 3
# 通过 taskset 包装
taskset -c 0-3 stressapptest -s 3600 -M 1024 -n 4 -l /tmp/stress_a76.log -v 3 &
taskset -c 4-7 stressapptest -s 3600 -M 1024 -n 4 -l /tmp/stress_a55.log -v 3 &
wait
```

### 3.3 stressapptest 输出解读

```
Logfile: stress.log
# 阶段1: Warmup
# 阶段2: Main test
Memory Input: 2048MB
Threads: 16
...
Report:
Errors: 0
```

关键字段：

- **`Hardware bit-flips in memory`**：硬件发现的位翻转（即 DDR 错误）
- **`Software bit-flips`**：软件逻辑对比错误（说明数据被破坏）
- **`Errors`**：错误总数
- **`ECC Recovered`**：ECC 修正次数（少量可接受，频繁则不合格）
- **`Page faults`**：缺页异常次数（与 mmap 行为有关）

### 3.4 异常告警与处理

| 告警 | 含义 | 处理 |
|------|------|------|
| `Hardware bit-flips` | 真实 DDR 错误 | 立即停止测试，标记 FAIL |
| `Software bit-flips` | 内存被破坏 | 检查 kernel oops / 看门狗 |
| `ECC Recovered` 高频 | 颗粒接近失效率 | 建议更换物料 |
| `Page fault rate 高` | 测试过程中触发换页 | 减小 `-M` 或关闭 swap |
| `Segfault` | 测试进程崩溃 | 检查 kernel log / 内存不足 |

---

## 4. 联合测试方案

### 4.1 标准流程（推荐）

```bash
#!/bin/bash
# run_ddr_stress.sh - 完整 DDR 压测脚本

set -e
LOG_DIR=/var/log/ddr_stress_$(date +%Y%m%d_%H%M%S)
mkdir -p $LOG_DIR

echo "========== Stage 1: memtester 单次扫描 =========="
memtester 2G 1 2>&1 | tee $LOG_DIR/memtester_quick.log

echo "========== Stage 2: memtester 多轮深度测试 =========="
memtester 2G 5 2>&1 | tee $LOG_DIR/memtester_deep.log

echo "========== Stage 3: stressapptest 4 小时烤机 =========="
stressapptest -s 14400 -M 2048 -m 8 -W 8 -n 16 \
              -l $LOG_DIR/stressapptest_4h.log -v 3 2>&1

echo "========== ALL PASSED =========="
ls -lh $LOG_DIR
```

### 4.2 自动化冒烟测试（CI 用）

```bash
#!/bin/bash
# smoke_ddr.sh - 5 分钟快速冒烟
set -e

# memtester 跑 1G × 1
timeout 180 memtester 1G 1 || {
    echo "FAIL: memtester"
    exit 1
}

# stressapptest 跑 60s
timeout 90 stressapptest -s 60 -M 1024 -n 8 -v 1 || {
    echo "FAIL: stressapptest"
    exit 1
}

echo "DDR SMOKE PASSED"
exit 0
```

### 4.3 量产烤机模板（72h）

```
┌────────────────────────────────────────────────┐
│  Day 1 (0-24h)   Day 2 (24-48h)   Day 3 (48-72h)│
│  ─────────────   ─────────────    ──────────── │
│  memtester 4G×10    stressapptest 72h         │
│  期间每 1h 记录温度                              │
│  期间每 1h 检查 kernel log (dmesg)              │
│  Day3 结束检查 ECC 修正次数 < 10 次              │
└────────────────────────────────────────────────┘
```

---

## 5. Pass / Fail 判定标准

### 5.1 判定矩阵

| 项目 | PASS 条件 | FAIL 阈值 |
|------|-----------|-----------|
| memtester 全 pattern 通过 | 13 种 pattern 全部 ok | 任何一种 FAIL |
| memtester 循环 N 轮 | N 轮无错误 | 任意一轮错误 |
| stressapptest Errors | Errors = 0 | Errors ≥ 1 |
| Hardware bit-flips | 0 | ≥ 1 即 FAIL |
| ECC 修正次数（4h 内） | < 5 | ≥ 10 |
| Kernel panic / oops | 0 | ≥ 1 即 FAIL |
| 温度稳定性 | < 阈值温度 | 持续高温触发 throttle |

### 5.2 边界情况判定

- **单次 1bit ECC 错误**：可接受，但要记录到 `eec_audit.log`
- **同地址多次 ECC 错误**：标记为可疑 Bank，FAIL
- **memtester 偶发 1 个错误**：复测 3 次，仍出现则 FAIL
- **stressapptest 偶发 1 个错误**：复测 3 次，仍出现则 FAIL


### 6 测试脚本
```c
#!/bin/sh
# run_ddr_stress.sh - 完整 DDR 压测脚本

set -e
LOG_DIR=/tmp/log/ddr_stress_$(date +%Y%m%d_%H%M%S)
mkdir -p $LOG_DIR

echo "========== Stage 1: memtester 单次扫描 =========="
/home/memtester 400M 1 2>&1 | tee $LOG_DIR/memtester_quick.log

echo "========== Stage 2: memtester 多轮深度测试 =========="
/home/memtester 400M 5 2>&1 | tee $LOG_DIR/memtester_deep.log

echo "========== Stage 3: stressapptest 4 小时烤机 =========="
/home/stressapptest -s 14400 -M 400 --max_errors 10  \
              -l $LOG_DIR/stressapptest_4h.log -v 3 2>&1

echo "========== ALL PASSED =========="
ls -lh $LOG_DIR



```

