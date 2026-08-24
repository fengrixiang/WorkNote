# VISCA 云台测试脚本

## 通用云台控制脚本

```python
import argparse
import socket
import signal
import sys
import time

# VISCA 指令定义（Pan-Tilt Drive）
CMD = {
    "stop":       "81 01 06 01 03 03 03 03 FF",
    "left":       "81 01 06 01 {pspeed:02X} 01 01 03 FF",
    "right":      "81 01 06 01 {pspeed:02X} 01 02 03 FF",
    "up":         "81 01 06 01 {pspeed:02X} 03 01 01 FF",
    "down":       "81 01 06 01 {pspeed:02X} 03 02 01 FF",
    "left_up":    "81 01 06 01 {pspeed:02X} 01 01 01 FF",
    "left_down":  "81 01 06 01 {pspeed:02X} 01 02 01 FF",
    "right_up":   "81 01 06 01 {pspeed:02X} 02 01 01 FF",
    "right_down": "81 01 06 01 {pspeed:02X} 02 02 01 FF",
    # AI Tracking Pan-Tilt
    "ai_left":       "81 01 06 01 00 60 50 01 03 FF",
    "ai_right":      "81 01 06 01 00 60 50 02 03 FF",
    "ai_up":         "81 01 06 01 00 60 50 03 01 FF",
    "ai_down":       "81 01 06 01 00 60 50 03 02 FF",
    "ai_left_up":    "81 01 06 01 00 60 50 01 01 FF",
    "ai_left_down":  "81 01 06 01 00 60 50 01 02 FF",
    "ai_right_up":   "81 01 06 01 00 60 50 02 01 FF",
    "ai_right_down": "81 01 06 01 00 60 50 02 02 FF",
    "ai_stop":       "81 01 06 01 00 03 03 03 03 FF",
    # 速度切换: 01=96级, 03=24级
    "speed_96":  "81 0a 11 71 01 01 FF",
    "speed_24":  "81 0a 11 71 01 03 FF",
}

# 预定义测试序列: (指令名, 延迟秒)
SEQUENCES = {
    "left_sweep": [
        ("left", 0.066), ("left", 0.066), ("left", 0.066),
        ("left", 0.066), ("left", 0.066), ("left", 0.066),
        ("left", 0.066), ("left", 0.066), ("left", 0.066),
        ("left", 0.066), ("left", 0.066), ("stop", 0.066),
    ],
    "left_right": [
        ("left", 1), ("stop", 1), ("right", 1),
    ],
    "ai_sweep": [
        ("ai_left", 0.066), ("ai_left", 0.066), ("ai_up", 0.066),
        ("ai_left", 0.066), ("ai_left_up", 0.066), ("ai_stop", 0.066),
        ("ai_right", 0.066), ("ai_down", 0.066),
        ("ai_right_down", 0.066), ("ai_right", 0.066),
    ],
}

running = True

def signal_handler(sig, frame):
    global running
    print("\n正在停止...")
    running = False

def build_command(name, pspeed=0x17):
    """根据指令名和水平速度构建字节指令"""
    template = CMD[name]
    hex_str = template.format(pspeed=pspeed)
    return bytes.fromhex(hex_str)

def run_ptz_test(ip, port, sequence, pspeed=0x17, init_cmd=None, loop=True):
    global running
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(2)
    signal.signal(signal.SIGINT, signal_handler)

    try:
        sock.connect((ip, port))
        print(f"已连接到 {ip}:{port}")

        # 发送初始化指令（如速度切换）
        if init_cmd:
            sock.send(bytes.fromhex(CMD[init_cmd]))
            print(f"初始化: {init_cmd}")

        # 预转换指令序列，避免循环中重复转换
        commands = []
        for name, delay in sequence:
            commands.append((build_command(name, pspeed), name, delay))

        while running:
            for cmd_bytes, name, delay in commands:
                if not running:
                    break
                sock.send(cmd_bytes)
                print(f"发送: {name}")
                time.sleep(delay)

    except socket.timeout:
        print("连接超时")
    except ConnectionRefusedError:
        print(f"连接被拒绝: {ip}:{port}")
    except OSError as e:
        print(f"通信错误: {e}")
    finally:
        sock.close()
        print("已关闭连接")

def main():
    parser = argparse.ArgumentParser(description="VISCA 云台测试工具")
    parser.add_argument("--ip", default="192.168.13.10", help="设备 IP 地址")
    parser.add_argument("--port", type=int, default=5678, help="端口号")
    parser.add_argument("--seq", default="left_sweep",
                        choices=SEQUENCES.keys(), help="测试序列")
    parser.add_argument("--speed", type=lambda x: int(x, 0), default=0x17,
                        help="水平速度 (十六进制如 0x17 或十进制如 23)")
    parser.add_argument("--init", default="speed_96", choices=["speed_96", "speed_24"],
                        help="初始化速度切换指令")
    parser.add_argument("--no-loop", action="store_true", help="只执行一轮")
    args = parser.parse_args()

    run_ptz_test(
        ip=args.ip,
        port=args.port,
        sequence=SEQUENCES[args.seq],
        pspeed=args.speed,
        init_cmd=args.init,
        loop=not args.no_loop,
    )

if __name__ == "__main__":
    main()
```

### 使用示例

```bash
# 默认: 192.168.13.10, 左转扫描, 96级速度
python ptz_test.py

# 指定另一台设备, 左右摆动测试
python ptz_test.py --ip 192.168.13.7 --seq left_right

# AI 追踪测试, 24级速度
python ptz_test.py --seq ai_sweep --init speed_24

# 自定义速度, 只跑一轮
python ptz_test.py --speed 0x09 --no-loop

# 帮助
python ptz_test.py --help
```