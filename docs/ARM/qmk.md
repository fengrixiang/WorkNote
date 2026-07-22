# QMK使用和AT32F423移植案例

⽂件密级：□绝密	□秘密	■内部资料	□公开

------

**免责声明**

​		本⽂档按“现状”提供，深圳市维海德技术股份有限公司（“本公司”，下同）不对本⽂档的任何陈述、信息和内容的准确性、可靠性、完整性、适销性、特定⽬的性和⾮侵权性提供任何明⽰或暗⽰的声明或保证。本⽂档仅作为使⽤指导的参考。

​		由于产品版本升级或其他原因，本⽂档将可能在未经任何通知的情况下，不定期进⾏更新或修改。

**版权所有 © 2023 深圳市维海德技术股份有限公司**

​		超越合理使⽤范畴，⾮经本公司书⾯许可，任何单位和个⼈不得擅⾃摘抄、复制本⽂档内容的部分或全部，并不得以任何形式传播。

​		深圳市维海德技术股份有限公司

​		Shenzhen ValueHD Technologies Co., Ltd.

​		⽹址：https://www.valuehd.com.cn/

------

**读者对象**

本⽂档主要适⽤于以下⼯程师：

- 软件开发工程师

**修订记录**

| 日期       | 版本   | 作者   | 修改说明 |
| ---------- | ------ | ------ | -------- |
| 2024/09/18 | V1.0.0 | 冯日祥 | 初始版本 |

------

## chibiOS简介
ChibiOS 是一个开源的实时操作系统（RTOS），主要用于嵌入式系统开发，尤其适合资源受限的微控制器（MCU）平台。它以高效、轻量和模块化著称，提供了内核、设备驱动和 HAL（硬件抽象层）等丰富的功能支持，常用于小型、低功耗嵌入式设备。

### ChibiOS 主要特点：

- 实时内核（ChibiOS/RT）：

ChibiOS 的内核是一个高度优化的实时操作系统内核，支持任务管理、线程调度、优先级抢占和时间管理。
支持多任务：允许多个线程并发运行，且可以通过抢占式调度确保高优先级任务获得执行。
任务间通信机制：包括信号量、消息队列、互斥锁等，提供了简单而有效的进程间通信和同步方式。

- 硬件抽象层（HAL）：

ChibiOS 的 HAL 层为不同硬件平台提供了统一的接口，开发者可以使用 HAL API 轻松访问底层硬件设备，如 GPIO、I2C、SPI、UART、ADC、PWM 等。
由于 HAL 层的抽象，开发者可以在不同的硬件平台上复用代码，极大减少了硬件依赖。

- 设备驱动：

ChibiOS 提供了丰富的设备驱动支持，可以直接与硬件外设通信，简化了开发过程。包括常见的传感器、存储设备、显示屏等驱动。
- 模块化设计：

ChibiOS 采用模块化设计，开发者可以根据实际需求选择启用哪些模块，从而减少系统开销，提升运行效率。
例如，可以仅启用 RTOS 内核，而不使用 HAL 或驱动部分，以最小化内存占用和提升运行性能。

- 可移植性：

ChibiOS 支持多种 MCU 平台，包括 STM32、AVR、Cortex-M 等主流微控制器架构，具有很好的可移植性。
通过简单的配置修改，ChibiOS 可以在不同的硬件平台上运行。
- 代码简洁、易于维护：

ChibiOS 的代码风格简洁，注重性能优化且具有较高的代码质量。其 RTOS 内核部分的代码非常精炼，使得它在资源受限的嵌入式环境中依然能够高效运行。

- 多种协议栈支持：

ChibiOS 支持多种常用的协议栈，包括 TCP/IP（如 lwIP）、USB 堆栈等，方便开发者在网络通信、数据传输等场景下使用。


### ChibiOS 主要组件：
- ChibiOS/RT：

这是核心的实时操作系统，负责任务调度、时间管理和任务间通信等。

- ChibiOS/HAL：

提供了硬件抽象层，实现了对 MCU 外设的统一接口封装，便于移植和硬件访问。

- ChibiOS/EX：

扩展库，包括文件系统、TCP/IP 网络协议栈、USB 协议栈等，可用于复杂的嵌入式应用开发。

- ChibiStudio：

官方提供的集成开发环境（IDE），基于 Eclipse，方便用户开发和调试 ChibiOS 应用。

## VIA键盘与default键盘的区别

	1. 键映射方式不同：
		VIA 键盘：支持在运行时通过 VIA 软件动态地改变键盘的按键映射，而无需重新编译和刷入固件。用户可以使用图形化界面随时调整按键布局。
		default 键盘：采用的是 QMK 固定的键映射。如果你想改变按键功能或布局，需要修改 keymap.c 文件并重新编译固件，然后刷入键盘。
	2. 修改键位的便捷性：
		VIA 键盘：用户可以非常轻松地通过 VIA 界面拖拽和选择按键功能，无需编程基础。对于想要频繁调整键位的人来说非常方便。
		default 键盘：修改键位需要掌握一定的 QMK 编程知识，并手动修改源代码，编译并烧录。
	3. 固件大小和复杂性：
		VIA 键盘：由于需要支持 VIA 动态配置功能，固件可能比默认 QMK 固件稍大，并且固件中包含了对 VIA 协议的支持代码。
		default 键盘：固件较为简单，只需要包含按键映射和功能定义，因此可能体积较小。
	4. 功能支持：
		VIA 键盘：除了键映射，VIA 还允许用户动态调整 RGB、背光、键盘层的功能，甚至创建复杂的宏，而这些操作都可以在不重新编译固件的情况下完成。
		default 键盘：所有功能需要在代码中提前定义并编译，因此功能修改相对复杂。
	5. 适用人群：
		VIA 键盘：更适合那些对键盘功能和布局有频繁调整需求的用户，尤其是非技术用户，因为不需要编程经验就可以修改键位。
      default 键盘：更适合那些对布局有明确需求、并愿意编写和修改代码的开发者或高级用户。

### 代码获取

``` C
git clone --single-branch --branch vhd-master ssh://git@192.168.1.204:33/vcaster/keyboard/qmk_firmware.git
```

### 环境搭建linux

```c
sudo apt install -y git python3-pip
python3 -m pip install --user qmk
```
*提示：尽量在docker容器里面构建*

## QMK简介

QMK（Quantum Mechanical Keyboard）是一个强大的开源键盘固件框架，允许用户为机械键盘编写自定义功能。其架构设计灵活，允许通过修改键盘布局、功能层、RGB灯光、宏和其他功能来控制键盘行为。以下是 QMK 的主要代码架构和关键组件的概述：


### 核心目录结构
核心目录结构如图所示：
![](@attachment/Clipboard_2024-09-18-14-40-08.png)

QMK 的代码库由多个核心目录组成，每个目录都包含不同功能的实现：

- quantum/：包含了 QMK 核心代码，如键盘矩阵扫描、键位处理、层处理、灯效控制等关键逻辑。
- tmk_core/：是 QMK 早期基于 TMK（另一款键盘固件）开发的部分代码，保留了很多低级别的硬件抽象和兼容性代码。
- drivers/：包含了硬件驱动，例如 USB 通信、LED 驱动器、OLED 显示驱动等硬件相关代码。
- keyboards/：每个支持的键盘都有一个独立的文件夹，定义了该键盘的硬件布局、键位映射等。
- layouts/：提供了通用的键盘布局定义，支持为不同类型的键盘快速应用相同的布局。
- users/：允许不同用户编写自己的配置文件，并为其键盘设置个性化的功能和布局。
- quantum/keymap_extras/：包含了键盘符号的定义，提供了 ISO、ANSI 和其他布局的符号映射支持。

### 代码逻辑结构
#### config.h 文件
每个键盘项目都包含一个 config.h 文件，用来定义硬件相关的配置，包括：

矩阵的行列数（MATRIX_ROWS 和 MATRIX_COLS）。
USB 设备描述符信息（厂商 ID、产品 ID）。
默认按键扫描频率等硬件参数。

#### keymap.c 文件
keymap.c 是用户自定义键位布局和功能的核心文件。主要结构包括：

按键矩阵布局：定义了每个按键的功能，通常用二维数组表示不同层的键位。

```c
const uint16_t PROGMEM keymaps[][MATRIX_ROWS][MATRIX_COLS] = {
    [0] = LAYOUT(
        KC_ESC,   KC_Q,    KC_W,   KC_E,    KC_R, ...
    ),
    [1] = LAYOUT(
        KC_TRNS,  KC_F1,   KC_F2,  KC_F3,   KC_F4, ...
    )
};
```

层支持：QMK 支持多个功能层，用户可以通过按下特定组合键切换不同层的按键功能。
宏和自定义功能：用户可以在这里定义宏按键，或自定义复杂的按键功能。

#### rules.mk 文件
rules.mk 是用来定义编译选项的文件。你可以在这个文件中指定启用的模块和功能，例如是否启用 RGB 灯光控制、是否支持 VIA 配置工具等。例如：

```makefile
RGBLIGHT_ENABLE = yes
VIA_ENABLE = yes
```

*提示：rules.mk文件的优先级会比config.h高*

#### matrix.c 和 matrix.h
这些文件处理物理键盘矩阵的扫描过程。键盘上的每个按键都与矩阵中的某一行和某一列相关联。QMK 通过扫描行和列来检测按键按下或松开的状态。

matrix_init()：初始化键盘矩阵。
matrix_scan()：每次循环调用以检测按键状态。

## 功能模块


## 编译和烧录流程
QMK 使用 make 系统来编译固件。编译一个键盘的固件的基本流程如下：

配置键盘： 在 keyboards/your_keyboard/keymaps/ 下创建自定义 keymap.c 文件。

### 编译固件
使用 make 命令或者qmk，编译指定键盘的固件。例如：

```bash
make your_keyboard:your_keymap

采用qmk命令直接编译：

qmk compile -kb vhd/d1hd/at32f423 -km default_f423
```


### 烧录固件
编译完成后，将生成的 .hex 或 .bin 文件刷入键盘。例如，如果使用的是 dfu 刷入工具：

```bash
make your_keyboard:your_keymap:dfu
```
*提示：这里没使用这个进行烧录，可能是QMK对ATM32的支持不是很好，采用的是使用亚特力的工具ArteryISPProgrammer.exe，直接将hex文件刷进芯片*

## QMK移植AT32F423VCT7
qmk官方的master分支是不支持AT32系列的，要想在qmk上使用AT32系列芯片需要进行一系列的移植，目前主要是qmk主分支
```c
https://github.com/qmk/qmk_firmware
```
基础上，参考qmk的分支
```c
https://github.com/zhaqian12/qmk_firmware.git
```
进行移植，并根据我们的环境和芯片对芯片板级文件和系统时钟数、驱动进行进一步修改完成。

*QMK的移植，事实上是ChibiOs操作系统支持AT32芯片的移植，QMK到ChibiOS这一层并不需要移植和修改。*



### 链接器脚本 AT32F423xC_uf2.ld
![](@attachment/Clipboard_2024-09-18-15-49-01.png)
```c
/*
 * AT32F423 memory setup.
 */
MEMORY
{
    flash0 (rx) : org = 0x08000000 + 16k, len = 256k - 16k
    flash1 (rx) : org = 0x00000000, len = 0
    flash2 (rx) : org = 0x00000000, len = 0
    flash3 (rx) : org = 0x00000000, len = 0
    flash4 (rx) : org = 0x00000000, len = 0
    flash5 (rx) : org = 0x00000000, len = 0
    flash6 (rx) : org = 0x00000000, len = 0
    flash7 (rx) : org = 0x00000000, len = 0
    ram0   (wx) : org = 0x20000000, len = 48k
    ram1   (wx) : org = 0x00000000, len = 0
    ram2   (wx) : org = 0x00000000, len = 0
    ram3   (wx) : org = 0x00000000, len = 0
    ram4   (wx) : org = 0x00000000, len = 0
    ram5   (wx) : org = 0x00000000, len = 0
    ram6   (wx) : org = 0x00000000, len = 0
    ram7   (wx) : org = 0x00000000, len = 0
}

/* For each data/text section two region are defined, a virtual region
   and a load region (_LMA suffix).*/

/* Flash region to be used for exception vectors.*/
REGION_ALIAS("VECTORS_FLASH", flash0);
REGION_ALIAS("VECTORS_FLASH_LMA", flash0);

/* Flash region to be used for constructors and destructors.*/
REGION_ALIAS("XTORS_FLASH", flash0);
REGION_ALIAS("XTORS_FLASH_LMA", flash0);

/* Flash region to be used for code text.*/
REGION_ALIAS("TEXT_FLASH", flash0);
REGION_ALIAS("TEXT_FLASH_LMA", flash0);

/* Flash region to be used for read only data.*/
REGION_ALIAS("RODATA_FLASH", flash0);
REGION_ALIAS("RODATA_FLASH_LMA", flash0);

/* Flash region to be used for various.*/
REGION_ALIAS("VARIOUS_FLASH", flash0);
REGION_ALIAS("VARIOUS_FLASH_LMA", flash0);

/* Flash region to be used for RAM(n) initialization data.*/
REGION_ALIAS("RAM_INIT_FLASH_LMA", flash0);

/* RAM region to be used for Main stack. This stack accommodates the processing
   of all exceptions and interrupts.*/
REGION_ALIAS("MAIN_STACK_RAM", ram0);

/* RAM region to be used for the process stack. This is the stack used by
   the main() function.*/
REGION_ALIAS("PROCESS_STACK_RAM", ram0);

/* RAM region to be used for data segment.*/
REGION_ALIAS("DATA_RAM", ram0);
REGION_ALIAS("DATA_RAM_LMA", flash0);

/* RAM region to be used for BSS segment.*/
REGION_ALIAS("BSS_RAM", ram0);

/* RAM region to be used for the default heap.*/
REGION_ALIAS("HEAP_RAM", ram0);

/* Generic rules inclusion.*/
INCLUDE rules.ld

/* TinyUF2 bootloader reset support */
_board_dfu_dbl_tap = ORIGIN(ram0) + 48k - 4; /* this is based off the linker file for tinyuf2 */

```

## 驱动移植
### QMK驱动移植

![](@attachment/Clipboard_2024-09-18-17-22-59.png)
这块是QMK提供给APP使用的驱动接口，例如编码器，旋钮，遥感，USB等模块驱动，基本不用改。

### ChibiOS驱动移植

![](@attachment/Clipboard_2024-09-18-17-14-18.png)

根据不同芯片平台进行添加补充，主要还是调用chibiOS hal层驱动。

### ChibiOS hal层驱动移植
![](@attachment/Clipboard_2024-09-18-16-14-05.png)

ChibiOS hal层驱动移植说白了就是将AT32F423的底层驱动模块，全部分装成ChibiOS的的驱动调用的接口，
具体见ChibiOS的底层启动接口：lib/chibios/os/hal/src/hal.c

```c
void halInit(void) {

  /* Initializes the OS Abstraction Layer.*/
  osalInit();

  /* Platform low level initializations.*/
  hal_lld_init();

#if (HAL_USE_PAL == TRUE) || defined(__DOXYGEN__)
#if defined(PAL_NEW_INIT)
  palInit();
#else
  palInit(&pal_default_config);
#endif
#endif
#if (HAL_USE_ADC == TRUE) || defined(__DOXYGEN__)
  adcInit();
#endif
#if (HAL_USE_CAN == TRUE) || defined(__DOXYGEN__)
  canInit();
#endif
#if (HAL_USE_CRY == TRUE) || defined(__DOXYGEN__)
  cryInit();
#endif
#if (HAL_USE_DAC == TRUE) || defined(__DOXYGEN__)
  dacInit();
#endif
#if (HAL_USE_EFL == TRUE) || defined(__DOXYGEN__)
  eflInit();
#endif
#if (HAL_USE_GPT == TRUE) || defined(__DOXYGEN__)
  gptInit();
#endif
#if (HAL_USE_I2C == TRUE) || defined(__DOXYGEN__)
  i2cInit();
#endif
#if (HAL_USE_I2S == TRUE) || defined(__DOXYGEN__)
  i2sInit();
#endif
#if (HAL_USE_ICU == TRUE) || defined(__DOXYGEN__)
  icuInit();
#endif
#if (HAL_USE_MAC == TRUE) || defined(__DOXYGEN__)
  macInit();
#endif
#if (HAL_USE_PWM == TRUE) || defined(__DOXYGEN__)
  pwmInit();
#endif
#if (HAL_USE_SERIAL == TRUE) || defined(__DOXYGEN__)
  sdInit();
#endif
#if (HAL_USE_SDC == TRUE) || defined(__DOXYGEN__)
  sdcInit();
#endif
#if (HAL_USE_SIO == TRUE) || defined(__DOXYGEN__)
  sioInit();
#endif
#if (HAL_USE_SPI == TRUE) || defined(__DOXYGEN__)
  spiInit();
#endif
#if (HAL_USE_TRNG == TRUE) || defined(__DOXYGEN__)
  trngInit();
#endif
#if (HAL_USE_UART == TRUE) || defined(__DOXYGEN__)
  uartInit();
#endif
#if (HAL_USE_USB == TRUE) || defined(__DOXYGEN__)
  usbInit();
#endif
#if (HAL_USE_MMC_SPI == TRUE) || defined(__DOXYGEN__)
  mmcInit();
#endif
#if (HAL_USE_SERIAL_USB == TRUE) || defined(__DOXYGEN__)
  sduInit();
#endif
#if (HAL_USE_RTC == TRUE) || defined(__DOXYGEN__)
  rtcInit();
#endif
#if (HAL_USE_WDG == TRUE) || defined(__DOXYGEN__)
  wdgInit();
#endif
#if (HAL_USE_WSPI == TRUE) || defined(__DOXYGEN__)
  wspiInit();
#endif

  /* Community driver overlay initialization.*/
#if defined(HAL_USE_COMMUNITY) || defined(__DOXYGEN__)
#if (HAL_USE_COMMUNITY == TRUE) || defined(__DOXYGEN__)
  halCommunityInit();
#endif
#endif

  /* Board specific initialization.*/
  boardInit();

/*
 *  The ST driver is a special case, it is only initialized if the OSAL is
 *  configured to require it.
 */
#if OSAL_ST_MODE != OSAL_ST_MODE_NONE
  stInit();
#endif
}
```

*boardInit()主要用于特殊板子的需求，可以为空*

### 芯片级底层驱动移植

![](@attachment/Clipboard_2024-09-18-17-25-32.png)
重点移植部分，芯片级驱动的接口，这层跟ChibiOS没有关系，主要是datasheet里面的描述编写的驱动功能接口，包括I2C，GPIO,DMA,SPI等。

其中驱动版本的选择，可以再platform.mk中进行选择，

```bash
include ${CHIBIOS_CONTRIB}/os/hal/ports/AT32/LLD/GPIOv2/driver.mk
include $(CHIBIOS_CONTRIB)/os/hal/ports/AT32/LLD/DMAv2/driver.mk
include $(CHIBIOS_CONTRIB)/os/hal/ports/AT32/LLD/TMRv1/driver.mk
include $(CHIBIOS_CONTRIB)/os/hal/ports/AT32/LLD/SYSTICKv1/driver.mk
include $(CHIBIOS_CONTRIB)/os/hal/ports/AT32/LLD/OTGv1/driver.mk
include $(CHIBIOS_CONTRIB)/os/hal/ports/AT32/LLD/ADCv2/driver.mk
include $(CHIBIOS_CONTRIB)/os/hal/ports/AT32/LLD/I2Cv2/driver.mk
include $(CHIBIOS_CONTRIB)/os/hal/ports/AT32/LLD/USARTv1/driver.mk
include $(CHIBIOS_CONTRIB)/os/hal/ports/AT32/LLD/SPIv1/driver.mk
```

*这块可以参考亚特力的hal库，接口可能不一样，但是寄存器是一样的*

## starup文件
启动文件（startup file）通常用汇编语言（.S 文件）编写，负责初始化硬件，并准备程序运行的基本环境。对于 AT32F423 这种基于 ARM Cortex-M 的微控制器，startup 文件的主要作用是配置中断向量表、初始化堆栈指针、设置复位处理程序，并调用主程序的入口点。

![](@attachment/Clipboard_2024-09-18-16-22-04.png)

AT32F423是M4架构，所以是starup文件是**ARMCMx/compilers/GCC/crt0_v7m.S**，.S启动这块没有做修改，因为都是ARM架构，应该大同小异，

![](@attachment/Clipboard_2024-09-18-16-28-48.png)
ARMv7-M (Cortex-M3/M4/M7) 这几个系列应该都是这个。

### 启动文件的基本结构
AT32F423 启动文件 .S 包含以下主要部分：

1. 中断向量表
```
.section .isr_vector, "a", %progbits
.type g_pfnVectors, %object
.size g_pfnVectors, .-g_pfnVectors

g_pfnVectors:
    .word _estack                    /* 堆栈顶地址 */
    .word Reset_Handler              /* 复位处理函数 */
    .word NMI_Handler                /* NMI 处理函数 */
    .word HardFault_Handler          /* 硬故障处理函数 */
    .word MemManage_Handler          /* 内存管理故障 */
    .word BusFault_Handler           /* 总线故障 */
    .word UsageFault_Handler         /* 用法故障 */
    ...

```
每个 .word 表示一个异常或中断的处理函数。处理器在出现某个异常或中断时，会跳转到对应的地址执行相应的处理程序。启动时，处理器会首先执行 Reset_Handler。

2. 堆栈指针初始化
处理器在启动时会从中断向量表中取出堆栈顶地址，并将其赋给堆栈指针。这通常由 _estack 来定义，它是堆栈区域的最高地址。
```
assembly
复制代码
.word _estack  /* 堆栈顶指针 */
_estack 通常在链接器脚本中定义。
```

3. 复位处理程序 Reset_Handler
Reset_Handler 是启动文件中的关键部分，它负责系统复位后的初始化工作，包括：

- 初始化 .data 段，将初始数据从闪存复制到 SRAM 中；
- 清零 .bss 段，确保未初始化的全局变量为 0；
- 调用系统初始化代码和主程序 main 函数。

```
Reset_Handler:  
    ldr   r0, =_sidata          /* 加载数据段初始化源地址 */
    ldr   r1, =_sdata           /* 加载数据段起始地址 */
    ldr   r2, =_edata           /* 加载数据段结束地址 */
    bl    CopyDataInit          /* 复制数据段 */
    
    ldr   r1, =_sbss            /* 加载 bss 段起始地址 */
    ldr   r2, =_ebss            /* 加载 bss 段结束地址 */
    bl    ZeroBss               /* 清零 bss 段 */
    
    bl    SystemInit            /* 初始化系统时钟和外设 */
    bl    main                  /* 跳转到主函数 */
```

4. 中断和异常处理程序
启动文件还包含一些默认的中断处理函数（弱符号定义），如果用户没有定义这些函数，则处理器在发生中断时会执行这些默认的空处理程序。
```
Weak_Default_Handler:
    b .

NMI_Handler:  
    b Weak_Default_Handler
HardFault_Handler:  
    b Weak_Default_Handler
...
```
用户可以在自己的代码中定义这些处理程序来处理不同的中断和异常。

5. 栈和堆
启动文件中通常会为堆栈和堆区域保留内存，具体大小可以通过链接器脚本定义。

QMK框架的GPIO和默认时钟初始化是放到汇编里面启动的。相当与上电后先跑了这部分，才会去跑main函数。

![](@attachment/Clipboard_2024-09-18-17-03-39.png)

```c
void __early_init(void) {

  at32_gpio_init();
  at32_clock_init();
}
```

### 时钟树修改
![](@attachment/Clipboard_2024-09-18-15-11-11.png)

默认的时钟树是按照8M 外部高速时钟进行配置的。
需要进行对应修改，我是先使用AT32_New_Clock_Configuration配好时钟后再编写代码进行改动的。

时钟配置相关的文件路径：
![](@attachment/Clipboard_2024-09-18-17-43-27.png)
*提示：其中USB时钟必须是48Mhz*

## 调试
qmk提供了qmk_toolbox和qmk_hid两个工具用于调试，均提交导了上述tools仓库中。
建议使用qmk_hid命令行工具来调试：
在插入键盘后，在命令行窗口中运行：

```c
qmk_hid.exe qmk -c
```

即可查看键盘通过console上报的调试消息。
通过修改代码目录中的keyboards/vhd/d1hd/rules.mk，注释掉下面一行即可关闭console输出：
```c
CONSOLE_ENABLE = yes
```

## TODO
目前只能通过DFU下载固件，需要将BOOT0拉低才能进入DFU模式。后续需要增加IAP方式下载固件，这样在正常工作状态就可以通过HID下载固件。
直接通过IAP进行线刷的功能还没弄
