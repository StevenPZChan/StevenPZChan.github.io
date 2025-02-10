---
title: CMake中使用ExternalProject引入其他编译系统的代码库
layout: post
categories: c++
tags:
  - c++
  - cmake
last_modified_at: 2025-02-01T21:00:00+08:00
---

### 起因

我们经常要引入一些三方库，而会有一些比较远古的不太维护的，或者作者没有选用 CMake 作为编译系统的，这样的代码库我们往往是要使用作者指定的编译方式编译好库文件，然后再在 CMake 中导入使用。

我们当然是希望我们的代码能够开箱即用，也就是说了解 CMake 系统的使用者按 CMake 的通常流程操作一遍就能完成编译，并且相关代码修改之后重新编译也能正确编译。对于这一类不是使用 CMake 作为编译系统的，CMake 其实也提供了以下方式：

- 自己编写`CMakeLists.txt`，指定源文件和编译参数等
- `execute_process`，显式执行某个命令行语句
- `ExternalProject_Add`，指定某个源文件目录为外部项目

对于小型的代码库，自己重新写一下`CMakeLists.txt`还算可行，但稍微复杂一点的代码库，要指定编译的源文件和复杂的编译参数会显得很费力；而`execute_process`只有在 CMake 配置的时候才会去执行，这种方式会对使用者有一定的要求；最终选择了`ExternalProject_Add`这个自由度比较高的系统。

### ExternalProject_Add 参数说明

官方文档说明见：[ExternalProject](https://cmake.org/cmake/help/latest/module/ExternalProject.html)

该函数的参数格式为`ExternalProject_Add(<name> [<option>...])`

常用的参数如下：

| 参数名            | 内容                   | 备注                                                                                                     |
| ----------------- | ---------------------- | -------------------------------------------------------------------------------------------------------- |
| SOURCE_DIR        | 源代码目录             | 一般需要显式指定，CMake 自己的查找规则比较乱                                                             |
| BINARY_DIR        | 编译文件目录           | 一般需要显式指定，CMake 自己的查找规则比较乱。如果`BUILD_IN_SOURCE=ON`，则无需设置，会跟`SOURCE_DIR`一样 |
| INSTALL_DIR       | 安装目录               | 一般设置成`${CMAKE_INSTALL_PREFIX}`                                                                      |
| CONFIGURE_COMMAND | 配置命令               | 设置成希望 CMake 运行配置时执行的命令，一般也会是对应编译系统的配置命令                                  |
| BUILD_COMMAND     | 编译命令               | 设置成对应编译系统的编译命令                                                                             |
| INSTALL_COMMAND   | 安装命令               | 设置成对应编译系统的安装命令，或者设置成`""`，我们自己写 install                                         |
| BUILD_IN_SOURCE   | 是否在源代码目录下编译 | 有些源代码必须要在当前目录下编译，比如写得不好的`Makefile`，就需要设置这个为`ON`                         |

#### 常见例子

普通 Makefile 的代码库：

```cmake
ExternalProject_Add(makefile_proj
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/makefile_proj
  INSTALL_DIR ${CMAKE_INSTALL_PREFIX}
  CONFIGURE_COMMAND ""
  BUILD_COMMAND $(MAKE)
  INSTALL_COMMAND $(MAKE) install
  BUILD_IN_SOURCE ON
)
```

使用 autotool 的代码库：

```cmake
ExternalProject_Add(autotool_proj
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/autotool_proj
  BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/autotool_proj
  INSTALL_DIR ${CMAKE_INSTALL_PREFIX}
  CONFIGURE_COMMAND <SOURCE_DIR>/configure
  BUILD_COMMAND $(MAKE)
  INSTALL_COMMAND ""
)
```

### CMake 使用代码库

上述步骤配置完成后只能实现 CMake 配置和编译的时候能相应触发运行指定的命令，但是要让我们的代码实际使用到编译出来的代码库，还需要增加一些配置。

假设外部项目的目标名为`foo_src`，先使用`ExternalProject_Get_Property(foo_src SOURCE_DIR BINARY_DIR)`获取到外部项目的源文件和编译文件目录

然后根据项目内容得到对应的头文件目录和库文件路径，如`FOO_INCLUDE_DIRS`和`FOO_LIBRARY_PATH`

使用如下方式创建一个导入的库

```cmake
add_library(foo [SHARED|STATIC] IMPORTED)
set_target_properties(foo PROPERTIES
  INTERFACE_INCLUDE_DIRECTORIES ${FOO_INCLUDE_DIRS}
  INTERFACE_LINK_LIBRARIES ${FOO_LINK_LIBRARIES}
  IMPORTED_LOCATION ${FOO_LIBRARY_PATH}
)
```

这样的话我们的代码里就可以直接链接这个新定义的库`foo`来使用导入的代码库了。

但是这里还有一个坑，就是 CMake 并不知道`ExternalProject_Add`和`add_library`这两个目标的关系，会默认把他们做并行处理，这样在实际编译的时候就有可能出现编译顺序错误的问题。因此还需要一步`add_dependencies(foo foo_src)`来指定编译的依赖关系，这样就完成了整个的配置。

### 可能遇到的坑

#### CONFIGURE_COMMAND 的执行时机

根据官网的说明，CONFIGURE_COMMAND 实际上并不会在每次 CMake 配置时都会执行，所以需要在外部项目的源文件可能发生改变时强制让 CMake 执行到 CONFIGURE_COMMAND。有一个比较简单的方式是通过`ExternalProject_Add_Step`，让每次编译都依赖于 CONFIGURE_COMMAND 的执行：

```cmake
ExternalProject_Add_Step(<name> <step>
  COMMAND <reconfigure command>
  WORKING_DIRECTORY <SOURCE_DIR>
  DEPENDERS configure
  ALWAYS TRUE
)
```

以上的配置目的是在`configure`这个步骤前增加一个自定义步骤，该步骤会执行一些重配置的命令，并且这个步骤是每次执行的`ALWAYS=TRUE`。比如使用 autotool 的代码库就可以是

```cmake
ExternalProject_Add_Step(autotool_proj autoreconf
  COMMAND autoreconf --force --install
  WORKING_DIRECTORY <SOURCE_DIR>
  COMMENT "Autoreconf autotool_proj"
  DEPENDERS configure
  ALWAYS TRUE
)
```

#### 编译器相关参数的传递

为了保持编译结果的可用性，一般还需要配置好编译器和编译参数，需要注意的变量一般有

```cmake
CC=${CMAKE_C_COMPILER}
CXX=${CMAKE_CXX_COMPILER}
AR=${CMAKE_AR}
RANLIB=${CMAKE_RANLIB}
CFLAGS=${CMAKE_C_FLAGS}
CXXFLAGS=${CMAKE_CXX_FLAGS}
LDFLAGS=${CMAKE_SHARED_LINKER_FLAGS}
```

所有这些变量一般是通过环境变量的方式传递的，比如使用 autotool 的代码库可以是

```cmake
set(autotool_proj_env)
list(APPEND autotool_proj_env CC=${CMAKE_C_COMPILER})
list(APPEND autotool_proj_env CXX=${CMAKE_CXX_COMPILER})
list(APPEND autotool_proj_env AR=${CMAKE_AR})
list(APPEND autotool_proj_env RANLIB=${CMAKE_RANLIB})
list(APPEND autotool_proj_env CFLAGS=${CMAKE_C_FLAGS})
list(APPEND autotool_proj_env CXXFLAGS=${CMAKE_CXX_FLAGS})
list(APPEND autotool_proj_env LDFLAGS=${CMAKE_SHARED_LINKER_FLAGS})

ExternalProject_Add(autotool_proj
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/autotool_proj
  BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/autotool_proj
  INSTALL_DIR ${CMAKE_INSTALL_PREFIX}
  CONFIGURE_COMMAND ${CMAKE_COMMAND} -E env ${autotool_proj_env} <SOURCE_DIR>/configure
  BUILD_COMMAND $(MAKE)
  INSTALL_COMMAND ""
)
```

#### 并行编译

官方文件的说明里有提到，如果外部项目也是使用`make`来编译的，需要在 BUILD_COMMAND 中将`make`换成`$(MAKE)`，这样外部项目的编译并行数就可以归并到总的项目的编译并行数中统一管理。不然的话，实测只能进行单线程编译。遗憾的是除了`make`以外还暂不支持。
