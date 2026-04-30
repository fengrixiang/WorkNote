# tcpdump 抓包工具

## 基本语法

```bash
tcpdump [选项] [过滤表达式]
```

## 常用选项

```bash
-i <网卡>        # 指定监听网卡，-i any 监听所有网卡
-c <数量>        # 抓取指定数量的包后停止
-w <文件名>      # 将抓包结果保存到文件（.pcap 格式）
-r <文件名>      # 从文件读取抓包数据
-n               # 不解析主机名（显示 IP）
-nn              # 不解析主机名和端口号（显示数字）
-v / -vv / -vvv  # 显示详细程度递增
-S               # 显示绝对序号（而非相对序号）
-s <字节数>      # 每个包的抓取长度，默认 262144，-s 0 抓取完整包
```

## 按接口抓包

```bash
# 查看可用网卡
tcpdump -D

# 监听指定网卡
tcpdump -i eth0

# 监听所有网卡
tcpdump -i any
```

## 按 IP 过滤

```bash
# 抓取指定 IP 的包（源或目的）
tcpdump -i eth0 host 192.168.1.100

# 抓取源 IP
tcpdump -i eth0 src host 192.168.1.100

# 抓取目的 IP
tcpdump -i eth0 dst host 192.168.1.100

# 抓取两个 IP 之间的通信
tcpdump -i eth0 host 192.168.1.100 and host 192.168.1.200
```

## 按端口过滤

```bash
# 抓取指定端口
tcpdump -i eth0 port 8080

# 抓取源端口
tcpdump -i eth0 src port 8080

# 抓取目的端口
tcpdump -i eth0 dst port 80

# 抓取端口范围
tcpdump -i eth0 portrange 8000-9000
```

## 按协议过滤

```bash
tcpdump -i eth0 tcp       # TCP 协议
tcpdump -i eth0 udp       # UDP 协议
tcpdump -i eth0 icmp      # ICMP 协议（ping）
tcpdump -i eth0 arp       # ARP 协议
```

## 按协议端口组合

```bash
# 抓取 HTTP 请求
tcpdump -i eth0 tcp port 80

# 抓取 SSH 通信
tcpdump -i eth0 tcp port 22

# 抓取 DNS 查询
tcpdump -i eth0 udp port 53
```

## 组合过滤（与、或、非）

```bash
# AND — 抓取 192.168.1.100 的 80 端口通信
tcpdump -i eth0 host 192.168.1.100 and port 80

# OR — 抓取 80 或 443 端口的包
tcpdump -i eth0 port 80 or port 443

# NOT — 排除 SSH 流量，减少干扰
tcpdump -i eth0 not port 22
```

## 保存与读取

```bash
# 保存抓包到文件（用 Wireshark 分析）
tcpdump -i eth0 -w capture.pcap

# 保存指定数量的包
tcpdump -i eth0 -c 100 -w capture.pcap

# 读取抓包文件
tcpdump -r capture.pcap

# 读取文件并按条件过滤
tcpdump -r capture.pcap port 80
```

## 按 TCP 标志位过滤

```bash
# 抓取 SYN 包（连接请求）
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# 抓取 SYN+ACK 包（连接响应）
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack) = (tcp-syn|tcp-ack)'

# 抓取 FIN 包（连接关闭）
tcpdump -i eth0 'tcp[tcpflags] & tcp-fin != 0'

# 抓取 RST 包（连接重置）
tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0'

# 抓取所有标志位包
tcpdump -i eth0 'tcp[tcpflags]'
```

## 按 HTTP 内容过滤

```bash
# 抓取 HTTP GET 请求
tcpdump -i eth0 -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'

# 抓取 HTTP POST 请求
tcpdump -i eth0 -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'

# 抓取包含指定字符串的包
tcpdump -i eth0 -A -s 0 | grep 'login'
```

## 实用示例

```bash
# 抓取完整 TCP 三次握手和四次挥手
tcpdump -i eth0 -S host 192.168.1.100 and tcp

# 抓包并以 ASCII 显示内容（调试 HTTP 等）
tcpdump -i eth0 -A port 80

# 抓包并以十六进制和 ASCII 显示
tcpdump -i eth0 -XX port 80

# 抓取大于指定大小的包
tcpdump -i eth0 greater 1000

# 抓取小于指定大小的包
tcpdump -i eth0 less 200

# 实时抓包保存，同时限制文件大小（滚动保存）
tcpdump -i eth0 -w capture.pcap -C 10 -W 5
# -C 10 每个文件最大 10MB，-W 5 最多保留 5 个文件
```
