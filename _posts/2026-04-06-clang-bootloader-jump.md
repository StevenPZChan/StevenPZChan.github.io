---
title: Clang编译bootloader跳转失败问题解决
layout: post
categories: c++
tags:
  - c++
  - mcu
  - bootloader
last_modified_at: 2026-11-02T15:00:00+08:00
---

### 缘由

最近的一个 MCU 项目开发 OTA 功能的时候出现了一个很奇特的问题，花了好些时间才定位出来的问题。这个问题的根本原因在 MCU 开发中需要非常重视，很值得拿出来说一下。

### 背景

负责 MCU 开发的小伙伴用的是 IAR，但是 IAR 在做 CI（持续集成）的时候会比较麻烦，所以我们项目中做编译检查和发布版本就还是选择了 Makefile + arm-none-eabi 工具链。但有一个很奇特的情况，就是使用 `arm-none-eabi-gcc` 编译的话，程序会卡在某个地方（与本文的问题无关，我们也没有深入研究），而尝试使用 `clang` 进行交叉编译之后程序就能正常运行。因此，实际的 CI 任务使用的是 `clang` 编译。

#### Clang 交叉编译 MCU 工程

虽然使用 `clang` 编译 MCU 代码属于曲线救国的方案，但是也值得简单提一下，毕竟 `clang` 在编译效率和汇编优化方面确实是比 `gcc` 强上一些的。

配置起来也非常简单，就是把 Makefile 里面的 `CC` 变量从 `arm-none-eabi-gcc` 换成 `clang --target=arm-none-eabi --sysroot=$(SYSROOT) -fshort-enums`，但是链接器 `LD` 还是得用 `arm-none-eabi-gcc`：

```make
SYSROOT ?= /usr/lib/arm-none-eabi
CC = clang --target=arm-none-eabi --sysroot=$(SYSROOT) -fshort-enums
LD = arm-none-eabi-gcc

$(BUILD_DIR)/%.o: %.c
	$(CC) -c $(CFLAGS) $< -o $@

$(BUILD_DIR)/$(TARGET_NAME).elf: $(OBJECTS)
	$(LD) $(OBJECTS) $(LDFLAGS) -o $@
```

这里 `--target=<target> --sysroot=<sysroot>` 参数是 `clang` 标准的交叉编译参数，其他的平台也是类似的配置方式，而 `-fshort-enums` 是因为链接时还得用 `arm-none-eabi-gcc`，为了保证枚举类型的一致性而必须使用的。

### 问题

回到我要分享的问题。小伙伴在实现 OTA 的功能，这里最基础的就是开发 bootloader，然后根据 app 固件烧录地址来找到 app 的 `ResetHandler`，调用它来启动 app。

我们遇到的奇特问题就是，用 IAR 和 clang 分别编译了 bootloader 和 app，而 IAR 编译的 bootloader 可以成功跳转到两个 app，而 clang 编译的 bootloader 可以成功跳转到 IAR 编译的 app，但是跳转到 clang 自己编译的 app 会卡死！

#### Cortex-M 向量表

我们先回顾一下 Cortex-M 的通用向量表。向量表放在 flash 或 RAM 的起始地址，本质上是一个存放函数指针的数组，前 16 项是所有 Cortex-M 芯片通用的：

| 偏移 | 内容 | 作用 |
| ---- | ---- | ---- |
| 0x00 | 初始 MSP 值 | 复位后硬件取它作为初始主栈指针 |
| 0x04 | ResetHandler | 复位入口，CPU 复位后跳到这里启动程序 |
| 0x08 | NMI | 不可屏蔽中断，不受优先级屏蔽 |
| 0x0C | HardFault | 兜底异常，未处理的错误都会汇聚到这里 |
| 0x10 | MemManage | 存储器访问违例（如 MPU 触发） |
| 0x14 | BusFault | 取指或数据访问非法地址时的总线错误 |
| 0x18 | UsageFault | 未定义指令、非对齐访问等用法错误 |
| 0x1C ~ 0x28 | 保留 | — |
| 0x2C | SVCall | 执行 SVC 指令进入，常用来做系统调用 |
| 0x30 | DebugMon | 调试监视异常，调试器专用 |
| 0x34 | 保留 | — |
| 0x38 | PendSV | 可挂起的系统服务，RTOS 常用它做任务切换 |
| 0x3C | SysTick | 系统滴答定时器，RTOS 常用它做时基 |

因此 bootloader 跳转 app 的方式就是，从 app 的固件起始地址取出初始主栈地址值，从偏移 0x04 的位置取出 ResetHandler 地址值，然后设置 MSP 值为初始主栈地址值，然后跳转到 ResetHandler 地址执行。代码如下：

```c
typedef void (*const pFunction)(void);

void jump_to_app(void)
{
    uint32_t app_msp = *(volatile uint32_t *)APP_FLASH_BASE; /* 初始 MSP */
    pFunction app_reset = (pFunction)*(volatile uint32_t *)(APP_FLASH_BASE + 4); /* ResetHandler */

    __disable_irq();

    __set_MSP(app_msp);
    app_reset();
}
```

### 分析

这个代码仔细看了一遍又一遍我们都没看出来有什么问题，因为 clang 编译的 bootloader 是可以成功跳转 IAR 编译的 app 的，所以理论上代码也是没有问题的。我们仔细对比了两个编译的 app，发现它们的 MSP 值还是有一些差别的，clang 编译的 app 的 MSP 值为 RAM 的最大地址，而 IAR 编译的 app 的 MSP 值则比较小，但是也没看出什么所以然来。

经过若干断点、日志和加辅助代码的尝试，仍然没有找到根本原因，我们大概只能确定问题出在 `__set_MSP` `app_reset` 这两句里，但是这两句又是必须的基本操作，确实让人摸不着头脑。

### 破局

最终是接了硬件调试器，在 `__set_MSP` 那里开始断点，一条一条指令来执行，看看是哪里的问题。我们发现，代码在执行到跳转指令之前，突然就进了 HardFault。我们仔细查看汇编指令，大概长这样：

```armasm
push    {r4, r5, lr}
ldrd    r4, r5, [r0]
msr     MSP, r4
mov     r0, r5
ldmia.w sp!, {r4, r5, lr}
bx      r0
pop     {r4, r5, pc}
```

执行到了 `ldmia.w sp!, {r4, r5, lr}` 这一步的时候进入了 HardFault，把前后的寄存器内容打出来一看，发现栈顶爆了，超出了内存范围！

问题清晰了：是 clang 编译的时候自己在 `__set_MSP` 和 `app_reset` 之间插入了一行出栈指令，导致栈顶超出内存范围。为什么会出现这个情况呢？是因为我们编写 app 跳转的代码时包装了一个函数，而 clang 在编译函数的时候需要处理函数返回，指令就有可能有一些重排，因此就在关键的 `__set_MSP` 和 `app_reset` 之间插入了致命的 sp 操作。

### 解决

解决方案其实也很简单，这个地方的指令是不能有重排的，所以直接用汇编写就好了：

```c
__asm volatile (
    "msr   msp, %0       \n"
    "bx    %1            \n"
    : : "r"(app_msp), "r"(app_reset) : "memory"
);
```

这样一来，什么编译器都不能够在这两句之间插入任何致命的语句了。

回过头看这个问题，其实引起了我们高度的重视，就是在 MCU 开发中，在重要的有时序要求的地方，不但要使用各种技巧来处理（比如 `volatile` `__disable_irq` 等等），还要确保编译结果是正确的，必要时还是自己写汇编最保险。
