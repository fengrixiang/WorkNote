## 安霸CV72开发心得

媒体的Pipline主要通过.lua文件+ test_encode工具

### 1、test_encode核心参数分类与解析

1. 流与通道管理

- -S/--stream [0~19]: 指定要操作的编码流（Stream）ID。这是多路编码的基础，主码流通常是0，子码流是1、2等。

- -c/--chan [0~4]: 指定视频输入通道（Channel）ID。一个通道（来自某个Sensor或处理分支）可以供给多个编码流。

2. 编码格式与分辨率设定

- -h/--h264 [分辨率]: 将当前流设置为H.264编码，并指定分辨率（如 1080p）。

- -H/--h265 [分辨率]: 将当前流设置为H.265编码。

- -m/--mjpeg [分辨率]: 将当前流设置为MJPEG编码。

- -n/--none: 将编码类型设为NONE，通常用于仅预览不编码。

3. 码率控制与图像质量

- --bc [cbr|vbr|...]: 指定码率控制模式。

- --bitrate [value]: 设置CBR的平均码率（单位：bps）。

- --vbr-bitrate [min~max]: 设置VBR的码率范围。

- -q/--quality [质量]: 设置JPEG/MJPEG的编码质量（通常1-100）。

4. GOP结构与高级编码控制

- -M, -N, --idr: 设置GOP结构（M为IP帧间隔，N为GOP长度，IDR帧间隔）。

- --gop [0~8]: 选择复杂的GOP模型（如SVCT分层编码、快速定位GOP等）。

- --force-idr: 强制插入一个IDR帧。

- --hflip/--vflip: 对当前流进行水平/垂直翻转。

5. 系统配置与资源管理

- <font color='red'>--resource-cfg <文件>: 最关键参数之一。指定定义整个视频输入管线（VIN/ISP/通道/画布） 的Lua配置文件</font>。

- <font color='red'>--vout-cfg <文件>: 指定视频输出（显示）的Lua配置文件</font>。

- --max-stream-num [2~20]: 指定当前模式下支持的最大编码流数量。

6. 信息查询与调试

```C
    --show-system-state          Show system state
    --show-encode-config         show stream(H.264/H.265/MJPEG) encode config
    --show-stream-info           Show stream format , size, info & state
    --show-resource-info         Show codec resource limit info
    --show-enc-mode-cap          Show encode mode capability
    --show-driver-info           Show IAV driver info
    --show-chip-info             Show chip info
    --show-dram-layout           Show DRAM layout info
    --show-feature-set           Show feature set info
    --show-resource-cfg          Show resource config
    --show-canvas-state          Show canvas state
    --show-vsrc-info             Show vsrc information including sensor name, topology between vinc and vsrc.
    --show-vout-info             Show vout information.
    --show-all-info              Show all info
```
7. 操作控制

- -e/--encode: 开始编码当前配置的流。

- -s/--stop: 停止编码当前流。

- --start-multi [m~n]: 启动多个流（如 --start-multi 0~2启动流0,1,2）。

- --nopreview: 不进入预览模式，直接编码（常用于后台服务）。

### 2、test_encode典型工作流与命令示例


#### 场景1：基础单流编码测试

```c
# 配置流0为H.265 1080p，使用指定的资源配置文件，并开始编码
test_encode -S 0 -H 1080p --resource-cfg /etc/amba/lua/imx327_4m.lua -e
```

#### 场景2：配置多路编码并查询信息
```c
# 步骤1：配置主码流（流0）为4M H.265
test_encode -S 0 -H 4k
# 步骤2：配置子码流（流1）为1080p H.264
test_encode -S 1 -h 1080p
# 步骤3：加载资源配置（这会使步骤1、2的配置生效）
test_encode --resource-cfg /etc/amba/lua/dual_stream_cfg.lua
# 步骤4：启动这两个流
test_encode --start-multi 0~1
# 步骤5：查看流状态
test_encode --show-stream-info
```

#### 场景3：性能分析与调试
```c
# 在编码过程中，查看系统负载和内存情况
test_encode --show-system-state
# 查看详细的DSP内存布局，分析是否有溢出
test_encode --show-dram-layout
```

#### 场景4：动态调整参数（无需重启编码）
```c
# 1. 首先启动编码
test_encode -S 0 -H 1080p --resource-cfg sensor_cfg.lua -e
# 2. 动态修改流0的码率（例如更改为2Mbps）
test_encode -S 0 --bitrate 2000000
# 3. 强制插入一个IDR帧
test_encode -S 0 --force-idr
```

#### 场景5
```c
test_encode --resource-cfg <输入配置文件.lua> --vout-cfg <输出配置文件.lua> 

#这是最常见的调试命令，可以实时在显示器上看到摄像头画面。
test_encode --resource-cfg imx327_1080p.lua --vout-cfg cv5_hdmi_1080p.lua

#运行此命令后，视频信号会经过处理并实时显示在连接的HDMI显示器上。按 q键退出预览。

#在预览基础上启动编码录制
#如果您需要在预览的同时进行录像，可以添加编码参数。

test_encode --resource-cfg imx327_1080p.lua \
            --vout-cfg cv5_hdmi_1080p.lua \
            -e h264 -o /tmp/record.h264

#不预览，直接后台编码（无显示）

对于无屏幕的设备（如网络摄像机），可以禁用预览，仅编码。

test_encode --resource-cfg imx327_1080p.lua \
            --vout-cfg cv5_hdmi_1080p.lua \
            --nopreview \
            -e h264 -o /tmp/record.h264

```

#### 场景6：动态调整参数（无需重启编码）
```c
#通过 --cfg-params参数，可以在命令行中直接覆盖Lua文件内的VOUT配置项，便于快速调试。
# 动态设置显示分辨率和翻转模式
test_encode --resource-cfg imx327_1080p.lua \
            --vout-cfg cv5_hdmi_1080p.lua \
            --cfg-params "vout0: mode='1920x1080p60', video_flip_mode='v'"

```

### 3.核心要点总结

1. 配置与执行分离：先使用 -S, -h, -H等参数“声明”流的配置，然后通过 --resource-cfg加载Lua文件来使能整个硬件管道，最后用 -e或 --start-multi启动编码。

2. <font color='red'>Lua文件是核心：绝大多数硬件相关的复杂配置（Sensor、ISP参数、管道连接、画布分配）都在 --resource-cfg指定的Lua文件中。命令行参数常用来覆盖或调整编码层面的参数（如码率、分辨率）。</font>

3. 多路编码逻辑：通过 -S指定不同的流ID来配置多路。确保Lua配置文件中已为该流分配了相应的源（Canvas）和编码资源。

4. 强大的调试能力：善用 --show-*系列命令来了解系统内部状态，这是解决资源分配错误、性能问题的最直接手段。

建议：初次使用时，从一个最简单的单流命令开始，逐步增加复杂度。每次修改参数后，使用 --show-stream-info和 --show-system-state来验证配置是否生效。

###