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