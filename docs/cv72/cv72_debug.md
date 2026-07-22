## CV72调试方法


### 查看帧率

```bash
    #方法1
    echo 0x43f3 3 > /proc/ambarella/iav_audit;cat /proc/ambarella/iav_audit 

    #方法2
    test_vin -t 0
```

### 查看中断

```bash
    #方法1
    cat /proc/interrupts
    #方法2
    dsp
```

### 查看时钟信息

```bash
    cat /proc/ambarella/clock
```

```
$ cat /sys/kernel/debug/clk/clk_summary
                                 enable  prepare  protect                                duty  hardware
   clock                          count    count    count        rate   accuracy phase  cycle    enable
-------------------------------------------------------------------------------------------------------
 ffd0000000.cdns-usb-phy_refclk-driver       0        0        0           0          0     0  50000         N
 ff20000000.cdns-pcie-phy_refclk-driver       0        0        0           0          0     0  50000         N
 gclk-wdt                             0        0        0    12000000          0     0  50000         Y
 cdns-phy-refclk                      2        2        0   100000000          0     0  50000         Y
 osc                                  2        2        0    24000000          0     0  50000         Y
    gclk_ir                           0        0        0     3000000          0     0  50000         Y
    gclk_adc                          0        0        0     3000000          0     0  50000         Y
    gclk_uart4                        0        0        0    24000000          0     0  50000         Y
    gclk_uart3                        0        0        0    24000000          0     0  50000         Y
    gclk_uart2                        0        0        0    24000000          0     0  50000         Y
    gclk_uart1                        0        0        0    24000000          0     0  50000         Y
    gclk_uart0                        0        0        0    24000000          0     0  50000         Y
    pll_out_enet                      1        1        0   600000000          0     0  50000         Y
       gclk_can                       0        0        0   600000000          0     0  50000         Y
       gclk_ssi3                      0        0        0   150000000          0     0  50000         Y
       gclk_pwm                       0        0        0     2000000          0     0  50000         Y
    gclk_nand                         0        0        0   264000000          0     0  50000         Y
    pll_out_sd                        0        0        0   792000000          0     0  50000         Y
       gclk_sd2                       0        0        0    19800000          0     0  50000         Y
       gclk_sd1                       0        0        0    39600000          0     0  50000         Y
       gclk_sd0                       0        0        0   198000000          0     0  50000         Y
    gclk_audio2                       0        0        0    12287555          0     0  50000         Y
    gclk_audio                        0        0        0    12287555          0     0  50000         Y
    pll_out_vo2                       0        0        0   297000000          0     0  50000         Y
    pll_out_hdmi                      0        0        0   742500000          0     0  50000         Y
    gclk_so1                          0        0        0    24000000          0     0  50000         Y
    gclk_so                           0        0        0    24000000          0     0  50000         Y
    gclk_nvp                          0        0        0   900000000          0     0  50000         Y
    gclk_idspv                        0        0        0   816000000          0     0  50000         Y
    gclk_idsp                         0        0        0  1272000000          0     0  50000         Y
    gclk_vdsp                         0        0        0   960000000          0     0  50000         Y
    gclk_dsu                          0        0        0  1200000000          0     0  50000         Y
       gclk_axi                       0        0        0   600000000          0     0  50000         Y
    gclk_cortex                       0        0        0  1608000000          0     0  50000         Y
    gclk_ddr0                         0        0        0  3192000000          0     0  50000         Y
    gclk_core                         0        0        0   624000000          0     0  50000         Y
       gclk_ssi2                      0        0        0   312000000          0     0  50000         Y
       gclk_ssi                       0        0        0    48000000          0     0  50000         Y
       gclk_apb                       0        0        0   156000000          0     0  50000         Y
       gclk_ahb                       0        0        0   312000000          0     0  50000         Y
 gclk_dummy                           0        0        0           0          0     0  50000         Y

```


### 打开telnet 和 ssh
```
81 11 12 13 2 ff
81 ab 1 1 1 3 2 ff
```


### 查看板子温度
```
/home/temperature_read.sh
```