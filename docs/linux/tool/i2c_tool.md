# I2C Tools 常用命令

## 安装

```bash
# Ubuntu/Debian
sudo apt install i2c-tools

# Buildroot / Yocto
# 在 menuconfig 中启用 i2c-tools 包
```

## i2cdetect — 检测 I2C 总线和设备

```bash
# 列出系统所有 I2C 总线
i2cdetect -l

# 扫描指定总线上的设备（0x03 - 0x77）
i2cdetect -y 0              # 总线 0
i2cdetect -y 1              # 总线 1

# 扫描时区分读写
i2cdetect -y -a 0           # 扫描所有地址（0x00 - 0x7F）
```

> 输出中 `--` 表示探测无响应，`UU` 表示该地址已被内核驱动占用，其他十六进制值为检测到的设备地址。

## i2cget — 读取寄存器

```bash
# 读取单个字节
i2cget -y <总线> <设备地址> <寄存器地址>

# 示例：从总线 0 的 0x50 设备读取 0x00 寄存器
i2cget -y 0 0x50 0x00

# 读取 16 位寄存器地址（适用于寄存器地址为 2 字节的芯片）
i2cget -y 0 0x50 0x0000 w

# 读取后不发送 STOP（连续读取时使用）
i2cget -y 0 0x50 0x00
```

## i2cset — 写入寄存器

```bash
# 写入单个字节
i2cset -y <总线> <设备地址> <寄存器地址> <值>

# 示例：向总线 0 的 0x50 设备的 0x10 寄存器写入 0xFF
i2cset -y 0 0x50 0x10 0xFF

# 写入 16 位值
i2cset -y 0 0x50 0x10 0xFFFF w

# 写入后读取回验（模式：先写后读验证）
i2cset -y 0 0x50 0x10 0xFF c
```

## i2cdump — 批量导出寄存器

```bash
# 导出设备所有寄存器值（字节模式）
i2cdump -y <总线> <设备地址>

# 示例：导出总线 0 上 0x50 设备的寄存器
i2cdump -y 0 0x50

# 指定模式导出
i2cdump -y 0 0x50 b       # 字节模式（默认）
i2cdump -y 0 0x50 w       # 字模式（16 位）
i2cdump -y 0 0x50 s       # SMBus 模块
```

## i2ctransfer — 灵活的读写组合（推荐）

```bash
# 基本格式
i2ctransfer -y <总线> <消息1> [消息2] ...

# 写入 1 字节到寄存器 0x10
i2ctransfer -y 0 w3@0x10 0x10 0xAB 0xCD
i2ctransfer -f -y 0 w3@0x10 0x01 0x00 0x00 
# w3 = 写入 3 字节（寄存器地址 + 数据）

# 从寄存器 0x00 读取 8 字节
i2ctransfer -y 0 w1@0x50 0x00 r8
# w1 = 先写 1 字节寄存器地址，r8 = 再读 8 字节

# 从 16 位寄存器地址 0x0100 读取 2 字节
i2ctransfer -y 0 w2@0x50 0x01 0x00 r2
```

## 常见场景示例

### EEPROM 读写（0x50）

```bash
# 读取前 16 字节
i2ctransfer -y 0 w1@0x50 0x00 r16

# 向地址 0x00 写入 4 字节数据
i2ctransfer -y 0 w5@0x50 0x00 0x11 0x22 0x33 0x44

# 导出全部内容
i2cdump -y 0 0x50
```

### 传感器芯片调试

```bash
# 1. 先扫描找到设备地址
i2cdetect -y 0

# 2. 读取芯片 ID 寄存器（假设为 0x00-0x01）
i2cget -y 0 0x20 0x00
i2cget -y 0 0x20 0x01

# 3. 导出全部寄存器，对比 datasheet
i2cdump -y 0 0x20

# 4. 修改某个配置寄存器
i2cset -y 0 0x20 0x10 0x01
```

### 开启/关闭 I2C 外设

```bash
# 通过 GPIO 扩展器控制（假设 PCA9535 地址 0x20）
# 读取当前输出状态
i2cget -y 0 0x20 0x02    # 输出寄存器

# 设置 bit0 为高（开启）
i2cset -y 0 0x20 0x02 0x01
```

## 注意事项

- 所有地址均为 **7 位地址**（不含读写位），如设备手册给出的是 8 位地址需右移 1 位
- `-y` 参数取消交互确认，脚本中必须加
- `-f` 参数强制访问已被内核驱动占用的设备（`i2cdetect` 显示 `UU` 的地址），调试时常用：

  ```bash
  i2cset -f -y 0 0x50 0x10 0xFF
  i2cget -f -y 0 0x50 0x00
  ```
- 嵌入式平台上可能需要先加载内核模块：`modprobe i2c-dev`
- 操作前确认总线编号：`i2cdetect -l` 查看可用总线
