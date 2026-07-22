# EWN-8822CS开发心得

EWN-8822CS 是一款基于 Realtek **RTL8822CS** 芯片的 **WiFi + Bluetooth 5.0** 二合一模组，2T2R 双频 802.11a/b/g/n/ac，常用于 IP Camera、AI 盒子、IoT 网关等嵌入式产品。本文档记录其驱动移植、WiFi/BT 联调、吞吐优化及常见问题排查。

## 1. 模组特性

### 1.1 关键参数

| 项目 | 规格 |
|------|------|
| 芯片方案 | Realtek RTL8822CS |
| Wi-Fi 标准 | 802.11a/b/g/n/ac，2.4G + 5G 双频并发 |
| 天线 | 2T2R（双天线） |
| 物理速率 | 5G: 867 Mbps；2.4G: 400 Mbps |
| 蓝牙 | BT 5.0（BR/EDR + BLE） |
| 接口 | SDIO 3.0（WiFi）+ UART/USB（BT，可选） |
| 工作电压 | VBAT 3.3V；I/O 1.8V 或 3.3V（依型号） |
| 尺寸 | 常见 12mm × 12mm / 15mm × 13mm |

### 1.2 接口说明

- **SDIO**：WiFi 数据通信，走 SDIO 3.0（最高 208MHz），建议四线 SDIO
- **WL_REG_ON**：WiFi 电源使能，HOST GPIO 控制
- **BT_REG_ON**：BT 电源使能，部分方案与 WL_REG_ON 共用
- **WL_HOST_WAKE / BT_HOST_WAKE**：模组唤醒 HOST 的 GPIO（低电平有效）
- **HOST_WAKE_WL / HOST_WAKE_BT**：HOST 唤醒模组的 GPIO（可选）
- **ANT0 / ANT1**：双天线接口，IPEX-4 代换座或 PCB 走线天线

## 2. 驱动移植（Linux）

### 2.1 内核配置

```makefile
# SDIO 控制器与 MMC 子系统（多数 SoC 已自带）
CONFIG_MMC=y
CONFIG_MMC_SDHCI=y
CONFIG_MMC_SDHCI_PLTFM=y
CONFIG_MMC_SDHCI_OF_ARASAN=y          # Xilinx / Ambarella / RK 等
CONFIG_MMC_BLOCK=y
CONFIG_MMC_BLOCK_MINORS=32

# Realtek 88XX 系列驱动
CONFIG_RTL8XXXU=m                     # USB 接口
CONFIG_RTW88=m                        # 新一代（8822C/8821C）
CONFIG_RTW88_8822CS=m                 # SDIO 接口
CONFIG_RTW88_SDIO=y
CONFIG_RTW88_DEBUG=y
CONFIG_CFG80211=m
CONFIG_MAC80211=m
CONFIG_WLAN=y
```

> **驱动选择**：RTL8822CS **优先使用 `rtw88`**（社区维护），老旧的 `rtl8821cs` 厂商树驱动已被逐步淘汰，吞吐与稳定性均不如 `rtw88`。

### 2.2 设备树配置

```dts
/* SDIO 节点 */
&sdhci0 {
    #address-cells = <1>;
    #size-cells = <0>;
    pinctrl-names = "default";
    pinctrl-0 = <&sdhci0_pins>;
    bus-width = <4>;
    cap-sd-highspeed;
    sd-uhs-sdr50;             /* 100MHz SDIO，RTL8822CS 上限 208MHz */
    sd-uhs-sdr104;            /* 208MHz，需 IO 1.8V 配置 */
    max-frequency = <208000000>;
    keep-power-in-suspend;
    non-removable;
    status = "okay";

    #address-cells = <1>;
    #size-cells = <0>;

    /* RTL8822CS SDIO 子节点（func 1 = WiFi） */
    rtl8822cs: wifi@1 {
        reg = <1>;
        compatible = "realtek,rtl8822cs";
        interrupt-parent = <&gpio>;
        interrupts = <RK_PB2 GPIO_ACTIVE_HIGH>;  /* WL_HOST_WAKE */
        /* rtw88 驱动需配合 firmware 路径 */
    };
};

/* 电源使能 GPIO */
&wifi_pwrseq {
    compatible = "mmc-pwrseq-simple";
    clocks = <&rtc 1>;
    clock-names = "ext_clock";
    pinctrl-names = "default";
    pinctrl-0 = <&wifi_enable_h>;
    reset-gpios = <&gpio0 RK_PB0 GPIO_ACTIVE_LOW>;   /* WL_REG_ON */
};

/* BT 部分（独立 UART + BT_REG_ON） */
&uart1 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart1_xfer &uart1_rtscts>;
    status = "okay";
    bluetooth {
        compatible = "realtek,rtl8822cs-bt";
        reset-gpios = <&gpio0 RK_PB1 GPIO_ACTIVE_LOW>;
        device-wake-gpios = <&gpio0 RK_PB2 GPIO_ACTIVE_HIGH>;
        host-wake-gpios = <&gpio0 RK_PB3 GPIO_ACTIVE_HIGH>;
    };
};
```

### 2.3 固件与配置文件

`rtw88` 驱动运行时需加载下列固件（路径 `/lib/firmware/rtw88/`）：

```text
rtw8822c_fw.bin
rtw8822c_wow_fw.bin              # WOW（Wake-on-WLAN）
```

如果内核版本较老（< 5.10），需要同时放置：

```text
rtl8822cs_fw.bin
rtl8822cs_wow_fw.bin
```

在 Buildroot / Yocto 中将固件打包到 rootfs：

```bash
# Buildroot
BR2_PACKAGE_RTW88_FW=y
# 或拷贝
target/Install/lib/firmware/rtw88/rtw8822c_fw.bin
```

### 2.4 关键内核参数

```bash
# SDIO 速率与稳定性
sdhci debug_quirks2=0x80000000     # 视平台而异
clk_ignore_unused

# 关闭省电
i915.enable_psr=0                   # 仅在带屏设备相关
```

## 3. WiFi 调试

### 3.1 基础命令

```bash
# 加载模块
modprobe rtw88
modprobe rtw88_8822cs

# 查看网卡
ifconfig wlan0
ip link show wlan0

# 启用 wpa_supplicant
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant.conf -D nl80211

# 分配 IP
udhcpc -i wlan0                     # busybox udhcpc
# 或
dhclient wlan0
```

### 3.2 wpa_supplicant 配置

```conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=root
update_config=1
country=CN

network={
    ssid="TestAP"
    psk="12345678"
    key_mgmt=WPA-PSK
    proto=RSN
    pairwise=CCMP
    group=CCMP
    scan_ssid=1
    priority=5
}

network={
    ssid="EnterpriseAP"
    key_mgmt=WPA-EAP
    eap=TTLS
    identity="user@example.com"
    password="xxx"
    phase2="auth=MSCHAPV2"
}
```

### 3.3 iw 调试 5G/2.4G

```bash
# 扫描
iw dev wlan0 scan | grep SSID

# 查看信道与频段
iw dev wlan0 scan | grep -E "freq|signal"
# freq=5180 → 5G 信道 36
# freq=2412 → 2.4G 信道 1

# 设置工作频段
iw reg set CN
iw dev wlan0 set channel 36          # 5G
iw dev wlan0 set channel 6 HT20      # 2.4G
```

### 3.4 吞吐量测试

```bash
# 服务端
iperf3 -s -p 5201

# 客户端（TCP）
iperf3 -c 192.168.1.100 -p 5201 -t 30 -i 1

# 客户端（UDP，查看丢包）
iperf3 -c 192.168.1.100 -p 5201 -u -b 200M -t 30

# 期望值参考（短距 5G AC 867）
# TCP: ~300~400 Mbps；UDP: 0.2% 丢包以内
```

## 4. 蓝牙调试

### 4.1 用户态组件

```bash
# 必要组件
bluez5-utils
bluez-firmware          # 部分平台
rtk-bt-firmware          # Realtek BT firmware loader（rtw88 系列内置）

# 启动 bluetoothd
/usr/libexec/bluetooth/bluetoothd -n -d &
```

### 4.2 加载 BT 固件（RTK HCI 模式）

RTL8822CS 的 BT 走 UART HCI，需在 bluetoothd 启动前将固件通过 HCI 指令注入：

```bash
rtk_hciattach -n -s 115200 ttyS1 rtk_h5   # RTS/CTS 流控
# 或
/usr/bin/rtk_bt_loader ttyS1 3000000 rtk8822cs
```

加载日志中应出现：

```text
RTK BT init: chip ID 0x0001
RTK BT download fw offset 0x1000
```

### 4.3 蓝牙常用工具

```bash
# 扫描
bluetoothctl scan on

# 配对
bluetoothctl pair AA:BB:CC:DD:EE:FF
bluetoothctl trust AA:BB:CC:DD:EE:FF
bluetoothctl connect AA:BB:CC:DD:EE:FF

# BLE 外设 / GATT 测试
gatttool -b AA:BB:CC:DD:EE:FF -I
> connect
> primary
> char-read-hnd 0x000a

# 列出已注册服务
hciconfig hci0
hciconfig hci0 up
hcitool dev
```

### 4.4 蓝牙 Wi-Fi 共存

RTL8822CS 内部已有 PTA（Packet Traffic Arbitration）共存逻辑，默认通过 SDIO 上行通道协商。需要检查：

```bash
# 查看 coex 状态（rtw88 debugfs）
cat /sys/kernel/debug/rtw88/rtw8822cs/coex
# 输出示例：
# TDMA: 0x3
# PTA: ON
# Current WL/BT state
```

共存关键点：

- BT SCO 通话时 WiFi 吞吐会下降 10%~30%，属正常
- BT 持续大流量传输（如 A2DP）时建议关闭 5G DFS 信道
- 2.4G 与 BT 共存最敏感，距离路由器/蓝牙耳机过近要评估

## 5. 性能优化

### 5.1 5G 吞吐调优

```bash
# 强制 80MHz + VHT
iw dev wlan0 set channel 149 HT80

# 关闭低速率，避免远端弱信号拖累吞吐
iw dev wlan0 set bitrates legacy-2.4 mcs-2.4 mcs-5 vht-mcs-5

# 调整 TX power（区域法规限制）
iw dev wlan0 set txpower fixed 1700          # 17 dBm
```

### 5.2 5G DFS 信道避坑

- 5G 频段中 **52~144** 为 DFS 信道，路由器扫描到雷达会强制切换 1~30 分钟
- 量产产品**建议默认绑定非 DFS 信道**（36, 40, 44, 48, 149, 153, 157, 161, 165）
- 国家码 `iw reg set CN` 时 149~165 全可用

### 5.3 SDIO 吞吐调优

```bash
# 确认 SDIO 实际速率
cat /sys/kernel/debug/mmc0/ios
# 期望：clock 200000000Hz, bus width 4, timing MMC_TIMING_UHS_SDR104

# 调整 SDIO 驱动队列深度（kernel bootargs）
mmc_core.max_segs=256
mmc_core.max_seg_size=65536
```

### 5.4 吞吐不达预期排查清单

1. `iw dev wlan0 link` 是否协商到期望的 MCS 速率
2. 信号强度是否 ≥ -55dBm（弱信号吞吐按指数下降）
3. 路由器是否开启 MU-MIMO / Beamforming
4. 模组天线 VSWR 是否达标（≤ 2）
5. SDIO 速率是否被 SoC 限到 50MHz

## 6. 常见问题

### 6.1 模组无法识别

```text
mmc0: error -110 whilst initialising SDIO card
```

排查：

- 测量 VBAT/WL_REG_ON 时序，上电波形需先 VBAT 稳定 100ms，再拉高 WL_REG_ON
- 检查 SDIO CMD/CLK/D0~D3 上拉电阻（建议 10K~100K）
- 示波器看 SDIO 波形，CLK 上升沿单调性差往往是线缆/连接器接触不良
- 试降低 SDIO 时钟到 50MHz：`sdhci debug_quirks=0x8`

### 6.2 扫描不到任何 AP

```bash
# 1. 检查 RF
dmesg | grep -i rtw
# 看到 "rtw88: failed to read efuse" 说明校准表丢失

# 2. 重写 efuse（需厂商工具）
# PC 端 Realtek BT/WiFi Tool → EFUSE 工具 → 写入 EFUSE_MAP

# 3. 检查天线
# ANT0/ANT1 任一虚焊，2T2R 降为 1T1R，吞吐腰斩
```

### 6.3 频繁断连

- 路由器兼容性：RTL8822CS 对部分 TP-LINK/小米老固件 roaming 不友好，开启 `bss_expire_age=180`
- 电源：动态负载下 VBAT 跌到 3.0V 以下会导致重传/掉线
- SDIO CRC error 频发：`dmesg | grep sdhci`，检查 SI/PI 引脚信号完整性

### 6.4 蓝牙扫描不到设备

- 确认 BT_REG_ON 拉高时序（与 WL_REG_ON 错开 50ms）
- 确认 firmware 加载成功（`dmesg | grep -i rtk_bt`）
- 检查 UART 流控 RTS/CTS 是否接反或悬空
- 模组与 BT 设备距离 ≥ 1m 测试，避开 USB 3.0 干扰

### 6.5 WOW（Wake-on-WLAN）失效

```bash
# 验证 suspend/resume 流程
echo enabled > /sys/class/net/wlan0/power/wakeup
rtw_wowlan enable      # 厂商工具命令

# 主机端需要挂的 wake GPIO 中断
echo "gpio_keys" > /sys/bus/platform/drivers/gpio-keys/bind
```

## 7. 量产与认证

### 7.1 量产要点

- 固件烧录（efuse + firmware）建议统一走工厂产测脚本
- 模组贴片后做 SDIO 读写测试 + 5G 吞吐抽检（≥ 200 Mbps 视为合格）
- BT 出厂扫描：能扫到 ≥ 3 个 AP 视为射频合格

### 7.2 认证注意事项

- **SRRC**（中国）：5G 必须支持 DFS
- **FCC / CE**：天线增益之和不得超出限值，PIFA/PCB 天线需报告最大增益
- **蓝牙 SIG**：RTL8822CS 已含 QDID，引用即可
- 量产固件需固化 `regulatory domain`，避免不同地区法规冲突

## 8. 参考命令速查

```bash
# 模组状态
dmesg | grep -E "rtw|rtl88|8822|mmc0"
lsmod | grep rtw

# 实时信号
watch -n 1 "iw dev wlan0 link; iw dev wlan0 station dump"

# 抓包
tcpdump -i wlan0 -w /tmp/wifi.pcap

# 复位 WiFi
ifconfig wlan0 down
modprobe -r rtw88_8822cs
modprobe rtw88_8822cs
ifconfig wlan0 up
```

## 9. 经验小结

- **驱动选择**：能用 `rtw88` 就别用 `rtl8821cs`，新驱动吞吐、稳定性和维护度都更好
- **天线**：双频 5G 吞吐对 ANT1 的位置非常敏感，layout 时 ANT0/ANT1 间距 ≥ λ/4（5G 约 8mm）
- **电源**：模组 TX 峰值电流可达 800mA，3.3V LDO 必须独立且靠近模组，避免与 SoC 共源引起跌落
- **调试**：先确认 dmesg 状态 → 再看 SDIO 时序 → 再看 RF 链路，90% 的不通都出在前面两步
- **认证**：量产前在 eeprom 中固化正确的 country code 与 tx power table，能省掉大量现场投诉
