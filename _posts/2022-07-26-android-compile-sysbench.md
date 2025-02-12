---
title: Android编译sysbench
layout: post
categories: misc
tags:
  - android
  - sysbench
last_modified_at: 2025-02-10T23:00:00+08:00
---

### Android 编译 sysbench

最近使用了 sysbench 做各个 SoC 的性能评估，其中有的开发板默认是 Android 系统，这里记录下编译 sysbench 的方法。

1. [源代码](https://github.com/akopytov/sysbench/tree/0.5)
1. 设置编译器环境变量`export CC=/opt/android-ndk-r20b-standalone/bin/aarch64-linux-android-clang CXX=/opt/android-ndk-r20b-standalone/bin/aarch64-linux-android-clang++`
1. 执行`./autogen.sh`
1. 编辑`configure`文件，注释掉`#define malloc rpl_malloc`
1. 编辑`sysbench/sysbench.c`文件，增加`pthread_cancel`的支持：

   ```c++
   #ifdef __ANDROID__
   int pthread_cancel(pthread_t thread) { return pthread_kill(thread, SIGUSR1); }
   #endif

   void thread_exit_handler(int sig) { pthread_exit(0); }

   struct sigaction actions;
   memset(&actions, 0, sizeof(actions));
   sigemptyset(&actions.sa_mask);
   actions.sa_flags = 0;
   actions.sa_handler = thread_exit_handler;
   sigaction(SIGUSR1, &actions, NULL);
   ```

1. 执行`./configure --host=aarch64-linux-android --without-mysql`
1. 执行`make -j`编译，编译结果在`sysbench/sysbench`
