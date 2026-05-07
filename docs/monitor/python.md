# 测试脚本

## 测试云台

```python
import socket
import time
ip = "192.168.13.10"
# ip = "192.168.13.9"
port = 5678
# 01: 96 level, 03: 24 level
speed_switch = "81 0a 11 71 01 01 FF"
def netvisca():
    visca_pt_left = "81 01 06 01 17 01 01 03 FF"
    visca_pt_left_1 = "81 01 06 01 11 01 01 03 FF"
    visca_pt_left_2 = "81 01 06 01 15 01 01 03 FF"
    visca_pt_left_3 = "81 01 06 01 14 01 01 03 FF"
    visca_pt_left_4 = "81 01 06 01 10 01 01 03 FF"
    visca_pt_right = "81 01 06 01 17 01 02 03 FF"
    visca_pt_stop = "81 01 06 01 03 03 03 03 FF"
    ai_visca_pt_left = "81 01 06 01 00 60 50 01 03 FF"
    ai_visca_pt_right = "81 01 06 01 00 60 50 02 03 FF"
    ai_visca_pt_stop = "81 01 06 01 00 03 03 03 03 FF"
    ai_visca_pt_up = "81 01 06 01 00 60 50 03 01 FF"
    ai_visca_pt_down = "81 01 06 01 00 60 50 03 02 FF"
    ai_visca_pt_left_up = "81 01 06 01 00 60 50 01 01 FF"
    ai_visca_pt_left_down = "81 01 06 01 00 60 50 01 02 FF"
    ai_visca_pt_right_up = "81 01 06 01 00 60 50 02 01 FF"
    ai_visca_pt_right_down = "81 01 06 01 00 60 50 02 02 FF"
    socket.setdefaulttimeout(2)
    tcp_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_addr = (ip, port)
    
    try:
        tcp_socket.connect(server_addr)
        print(f"已连接到 {ip}:{port}")
        cmd = bytes.fromhex(speed_switch)
        tcp_socket.send(cmd)
        commands = [
            (visca_pt_left, "向左转动", 0.066), 
            (visca_pt_left_1, "向左转动1", 0.066),
            (visca_pt_left_2, "向左转动2", 0.066),
            (visca_pt_left_3, "向左转动3", 0.066),
            (visca_pt_left_4, "向左转动4", 0.066),
            (visca_pt_left, "向左转动", 0.066),
            (visca_pt_left_1, "向左转动1", 0.066),
            (visca_pt_left_2, "向左转动2", 0.066),
            (visca_pt_left_3, "向左转动3", 0.066),
            (visca_pt_left_4, "向左转动4", 0.066),
            (visca_pt_left, "向左转动", 0.066),
            (visca_pt_stop, "停止", 0.066),
            #(ai_visca_pt_left, "AI向左转动", 0.066),
            #(ai_visca_pt_left, "AI向左转动", 0.066),
            #(ai_visca_pt_up, "AI向上转动", 0.066),
            #(ai_visca_pt_left, "AI向左转动", 0.066),
            #(ai_visca_pt_left, "AI向左转动", 0.066),
            #(ai_visca_pt_left_up, "AI向左上转动", 0.066),
            #(ai_visca_pt_stop, "停止", 0.066),
            #(ai_visca_pt_right, "AI向右转动", 0.066),
            #(ai_visca_pt_right, "AI向右转动", 0.066),
            #(ai_visca_pt_down, "AI向下转动", 0.066),
            #(ai_visca_pt_right_down, "AI向右下转动", 0.066),
            #(ai_visca_pt_right, "AI向右转动", 0.066),
            #(ai_visca_pt_right, "AI向右转动", 0.066),
        ]
        while True:
            for cmd_str, cmd_name, slp in commands:
                b = cmd_str.replace(' ', '')
                cmd = bytes.fromhex(b)
                tcp_socket.send(cmd)
                print(f"发送命令: {cmd_name}")
                time.sleep(slp)
                
    except Exception as e:
        print(f"通信错误: {e}")
        
    finally:
        tcp_socket.close()
        print("已关闭连接")
if __name__ == "__main__":
    netvisca()


```

## 测试云台

```python
import socket
import time

ip = "192.168.13.7"

def netvisca():
    visca_pt_left = "81 01 06 01 09 01 01 03 FF"
    visca_pt_right = "81 01 06 01 09 01 02 03 FF"
    visca_pt_stop = "81 01 06 01 03 03 03 03 FF"

    
    socket.setdefaulttimeout(2)
    tcp_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_addr = (ip, 5678)
    
    try:
        tcp_socket.connect(server_addr)
        print(f"已连接到 {ip}:5678")

        commands = [
            (visca_pt_left, "向左转动"),
            (visca_pt_stop, "停止"),
            (visca_pt_right, "向右转动")
        ]

        while True:
            for cmd_str, cmd_name in commands:
                b = cmd_str.replace(' ', '')
                cmd = bytes.fromhex(b)
                tcp_socket.send(cmd)
                print(f"发送命令: {cmd_name}")
                time.sleep(1)
                
    except Exception as e:
        print(f"通信错误: {e}")
        
    finally:
        tcp_socket.close()
        print("已关闭连接")

if __name__ == "__main__":
    netvisca()

```