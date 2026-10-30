---
title: 编译C++编写的Linux内核模块
layout: post
categories: c++
tags:
  - c++
  - linux
  - kernel
last_modified_at: 2025-05-30T15:39:00+08:00
---

### 背景

Linux 内核模块通常使用 C 编写，内核的 Kbuild 也主要面向 C 代码。正好最近在做预研，想要把目前 MCU 中的 C++ 驱动代码移植到 Linux 系统里做验证，就尝试摸索了一下方法。要在模块中使用 C++，一种比较直接的做法是把 C++ 逻辑编译成静态库，再由 C 入口调用，并通过 Kbuild 把它们链接成 `.ko` 文件。简单梳理一下编译过程。

### 编译流程

整体可以分成三层：

1. **C++ 核心逻辑**：由 CMake 编译成 `.a` 静态库，相关的 C++ 源文件、头文件和编译选项在 `CMakeLists.txt` 中维护。
2. **C 语言模块入口**：负责调用内核 API，例如注册 platform driver、实现 `module_init` 和 `module_exit`。入口文件可以包含 Linux 内核头文件，再通过 C 接口调用 C++ 逻辑。
3. **Kbuild 链接模块**：Kbuild 编译 C 入口，并把入口对象和前面生成的静态库组合成最终的 `.ko` 文件。

举一个简单的 GPIO 驱动的例子， `gpio_sensor_main.c` 负责驱动注册，C++ 的 `gpio_sensor.cc` 实现传感器逻辑。两边通过 `CreateGpioSensor`、`DestroyGpioSensor` 这样的 C 接口衔接。对应的 Kbuild 规则大致如下：

```make
obj-m += gpio_sensor.o
gpio_sensor-y := gpio_sensor_main.o $(kernel-libs)
```

这里的 `gpio_sensor.o` 是组合目标；`$(kernel-libs)` 则由构建配置填入需要链接的 C++ 库及内核 C 库。
而后按照常规的外部模块的编译方式配置好内核源代码目录和相应的编译选项之后就可以编译生成 `.ko` 文件。

以下再展开说一下一些处理细节。

### C 和 C++ 的接口边界

Linux 内核头文件只能在 C 源文件中包含，而且所有的 C 源文件都是要通过 Kbuild 框架去编译；C++ 源文件通过窄的 C 接口使用内核功能。接口头文件需要同时兼容两种语言，例如：

```c++
#ifdef __cplusplus
extern "C" {
#endif

void *CreateGpioSensor(struct device *dev, const struct GpioSensorConfig *config);
void DestroyGpioSensor(const void *gpio_sensor);

#ifdef __cplusplus
}
#endif
```

`extern "C"` 用来避免 C++ 名称修饰，让 C 入口能够链接到对应实现。相应的，C++ 逻辑也可能需要调用内核 API，这些也需要由 C 文件实现一层封装，再提供 C/C++ 都能包含的接口。

### C++ 使用限制

内核不是用户态 C++ 运行环境，可以认为 libc 和常规 C++ 运行库都不存在，避免使用任何 C++ STL 特性。首先要关闭 RTTI 和异常（`-fno-rtti -fno-exceptions`），然后提供有限的 `new`/`delete` （内存操作需要使用内核专用的 API）：

```c++
// memory.h
#pragma once

#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

void *CxxMalloc(size_t size);
void kfree(const void *);

#ifdef __cplusplus
}
#endif

// memory.c
#include <memory.h>

#include <linux/slab.h>

void *CxxMalloc(const size_t size) { return kmalloc(size, GFP_KERNEL); }

// cpp_support.cc
#include <cstddef>

#include <memory.h>

void *operator new(const std::size_t count) { return CxxMalloc(count); }

void *operator new[](const std::size_t count) { return CxxMalloc(count); }

void operator delete(void *const ptr) noexcept { kfree(ptr); }

void operator delete[](void *const ptr) noexcept { kfree(ptr); }

void operator delete(void *const ptr, std::size_t) noexcept { kfree(ptr); }

void operator delete[](void *const ptr, std::size_t) noexcept { kfree(ptr); }

extern "C" void __cxa_pure_virtual() {}
```

另外内核代码还需要注意浮点寄存器使用，Linux 内核里几乎没有浮点计算，线程切换通常不进行浮点寄存器的上下文保存和恢复处理。我们可以默认跟内核源码一样，对相关源文件使用 `-mgeneral-regs-only` 编译选项，确保不会使用浮点寄存器；而确实需要浮点运算的代码，不使用 `-mgeneral-regs-only` 编译选项，在浮点计算的上下文中调用内核相关的浮点寄存器管理 API：

```c++
// float.h
#pragma once

#ifdef __cplusplus
extern "C" {
#endif

#define NO_FLOATING_POINT_OPERATION __attribute__((target("general-regs-only")))
#define FLOATING_POINT_OPERATION_BEGIN FPUBegin()
#define FLOATING_POINT_OPERATION_END FPUEnd()
#define FLOATING_POINT_OPERATION_SCOPE(x)                                      \
  do {                                                                         \
    FLOATING_POINT_OPERATION_BEGIN;                                            \
    x;                                                                         \
    FLOATING_POINT_OPERATION_END;                                              \
  } while (0)

void FPUBegin(void);
void FPUEnd(void);

#ifdef __cplusplus
}
#endif

// float.c
#include <float.h>

#ifdef __x86_64__
// clang-format off
#include <linux/types.h>
#include <asm/fpu/api.h>
// clang-format on

void FPUBegin(void) { kernel_fpu_begin(); }

void FPUEnd(void) { kernel_fpu_end(); }
#endif

#ifdef __aarch64__
#include <asm/neon.h>

void FPUBegin(void) { kernel_neon_begin(); }

void FPUEnd(void) { kernel_neon_end(); }
#endif
```

### 其他问题

共用的内核接口实现源文件可以编译成静态库来供多个内核模块来链接，避免重复编译，实现方式是在 Kbuild 中使用 `lib-y` 这个变量，而头文件的引用目录可以使用 `ccflags-y` 这个变量，如：

```make
lib-y := float.o \
         memory.o
ccflags-y += -I$(src)/../include
```

这样的话 Kbuild 编译完会在相应的目录下生成 `lib.a` 文件（就叫这个名字，没有任何额外的字符），内核模块的 Kbuild 文件再引用链接这个 `lib.a` 就可以了。

### 总结

以上就是用 C++ 编写内核模块（事实上是 C/C++ 混合编写的）的编译方法和注意事项。目前 MCU 的驱动代码大部分都移植完了，功能上表现得也差得不多，也算是一种新的架构思路吧。
