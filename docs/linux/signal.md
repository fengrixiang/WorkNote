# Linux 信号 (Signals)

## 常见信号一览

| 信号 | 数值 | 默认行为 | 触发方式 | 可捕获/忽略 |
| ---- | ---- | -------- | -------- | ----------- |
| SIGILL | 4 | 终止 + core | 非法指令 | 可捕获 |
| SIGABRT | 6 | 终止 + core | `abort()` 调用 | 可捕获 |
| SIGBUS | 7 | 终止 + core | 总线错误 | 可捕获 |
| SIGFPE | 8 | 终止 + core | 算术异常 | 可捕获 |
| SIGSEGV | 11 | 终止 + core | 段错误 | 可捕获 |
| SIGPIPE | 13 | 终止 | 写入已关闭的管道/Socket | 可捕获 |
| SIGKILL | 9 | 终止 | `kill -9` | **不可捕获** |
| SIGTERM | 15 | 终止 | `kill` 默认信号 | 可捕获 |
| SIGXCPU | 24 | 终止 + core | 超出 CPU 时间限制 | 可捕获 |
| SIGXFSZ | 25 | 终止 + core | 文件大小超限 | 可捕获 |

## 详细说明

### SIGILL（4）— 非法指令

进程尝试执行非法指令。常见原因：CPU 架构不匹配（如 ARM 二进制在 x86 上运行）、二进制文件损坏、代码段被破坏。

错误码细分：

| 错误码 | 含义 |
| ------ | ------ |
| ILL_ILLOPC | 非法操作码 |
| ILL_ILLOPN | 非法操作数 |
| ILL_PRVOPC | 特权操作码（用户态执行内核指令） |
| ILL_ILLTRP | 非法陷阱 |

### SIGABRT（6）— 进程主动终止

由 `abort()` 函数触发，用于异常终止并生成 core dump。开发者常在断言失败或检测到不可恢复错误时主动调用。

```c
// 触发 SIGABRT
assert(ptr != NULL);  // 断言失败时调用 abort()
abort();              // 直接触发
```

### SIGBUS（7）— 总线错误

硬件总线错误，通常由地址未对齐或访问无效物理地址触发。

| 错误码 | 含义 | 典型场景 |
| ------ | ------ | ------ |
| BUS_ADRALN | 未对齐访问 | 32 位系统要求 4 字节对齐，直接用指针访问未对齐地址 |
| BUS_ADRERR | 无效物理地址 | `mmap` 的文件被截断后访问 |
| BUS_OBJERR | 对象特定硬件错误 | 硬件故障 |

### SIGSEGV（11）— 段错误

进程访问了无权限或无效的内存地址，是最常见的崩溃信号。

典型场景：野指针、数组越界、写入只读内存（代码段）、栈溢出、使用已释放的内存。

**SIGSEGV vs SIGBUS**：SIGSEGV 是合法地址的非法操作（权限/映射问题），SIGBUS 是物理地址或对齐问题。

### SIGFPE（8）— 算术异常

致命算术错误，不仅限于浮点运算。

```c
int a = 1 / 0;         // 整数除零 → SIGFPE
int b = INT_MIN / -1;   // 整数溢出 → SIGFPE（部分平台）
```

### SIGKILL（9）— 强制终止

立即终止进程，**不可被捕获、阻塞或忽略**。由内核直接处理，常用于管理员强制结束无响应进程。

```bash
kill -9 <pid>    # 发送 SIGKILL
```

> 注意：SIGKILL 无法生成 core dump，进程来不及做任何清理。

### SIGPIPE（13）— 管道/Socket 断开

写入无读取端的管道或已关闭的 Socket 连接时触发。默认终止进程，需要在网络编程中显式处理。

```c
// 忽略 SIGPIPE（网络编程常见做法）
signal(SIGPIPE, SIG_IGN);

// 或通过 socket 选项忽略
int set = 1;
setsockopt(sockfd, SOL_SOCKET, SO_NOSIGPIPE, &set, sizeof(set));
```

### SIGXCPU（24）/ SIGXFSZ（25）— 资源限制

| 信号 | 触发条件 | 说明 |
| ------ | ------ | ------ |
| SIGXCPU | 超出 CPU 时间限制 | `setrlimit(RLIMIT_CPU, ...)` 设置 |
| SIGXFSZ | 文件大小超限 | `setrlimit(RLIMIT_FSIZE, ...)` 设置 |

```bash
# 查看当前资源限制
ulimit -a

# 设置 CPU 时间限制为 10 秒
ulimit -t 10
```
