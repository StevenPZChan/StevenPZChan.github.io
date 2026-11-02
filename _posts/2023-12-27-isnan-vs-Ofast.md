---
title: 浮点数isnan判断失效问题
layout: post
categories: c++
tags:
  - c++
last_modified_at: 2026-11-02T00:00:00+08:00
---

最近朋友做项目遇到一个很奇特的问题拿出来分享一下。

他说项目里算法老是跑飞，加了数据记录发现跑一会就变成 NaN 了，然后加了防御机制在关键步骤用 `std::isnan` 判断是否为 NaN，结果还是跑飞，最终他发现在数据为 NaN 的时候直接用 `std::isnan` 判断居然是 false ！

我起初是不相信的，C++ 或者编译器怎么会有这么明显的 bug。我简单写了一个测试来验证：

```c++
#include <cmath>
#include <cstdlib>
#include <iostream>

int main() {
    const float nan = std::strtof("nan", nullptr);

    std::cout << std::boolalpha
              << "std::isnan(NaN): " << std::isnan(nan) << std::endl;
}
```

毫无疑问结果是 true。我让朋友也去验证，确实也是 true。但他觉得事情一定有蹊跷，把这个简单的测试代码直接加到项目代码里编译运行，哎，结果居然就是 false ！

至此，我敏锐地发现一定是项目配置的编译器或者是编译选项有问题。编译器也是用的 `g++`，没什么特别的，我让他把完整的编译参数导出来仔细研究。我按他们项目的编译参数来进行编译，果然复现了这个奇特的现象！

随后我开始了逐个排查，最终锁定在了一个编译优化参数 `-Ofast` 上。我赶紧去查了下资料，发现 `-Ofast` 是一个非常激进的优化参数，我们通常都是开 `-O2` 或者 `-O3`，而 `-Ofast` 相当于在 `-O3` 的基础上加上了 `-ffast-math`，目的是允许编译器做激进的简化来提升浮点数运算速度。这里相关的最主要的一点就是会忽略特殊值如 NaN、Inf 等，也就是说 `std::isnan` `std::isinf` 这一类的函数会直接失效！从 STL 的库函数的实现来看，`-ffast-math` 实际上是自动定义了 `__FINITE_MATH_ONLY__` 这个宏，而在 `std::isnan` `std::isinf` 的实现里，如果定义了这个宏，会直接返回 false。

问题原因找到了，那怎么解决呢？朋友的这个项目之所以开了这个编译参数，肯定还是出于项目有极致的性能追求，那编译选项不能动，就只能自己根据标准来实现 NaN 的检测了。大致的思路如下：

```c++
template <typename T>
struct IEEE754Helper {};
template <>
struct IEEE754Helper<float> {
    // reference: https://en.wikipedia.org/wiki/Single-precision_floating-point_format
    using MaskType = uint32_t;
    using UnionType = union {
        MaskType u;
        float f;
    };
    static constexpr MaskType kExponentMask = 0xFFU << 23;  // bit 23-30
    static constexpr MaskType kFactionMask = 0x7FFFFFU;     // bit 0-22
};
template <>
struct IEEE754Helper<double> {
    // reference: https://en.wikipedia.org/wiki/Double-precision_floating-point_format
    using MaskType = uint64_t;
    using UnionType = union {
        MaskType u;
        double f;
    };
    static constexpr MaskType kExponentMask = 0x7FFUL << 52;     // bit 52-62
    static constexpr MaskType kFactionMask = 0xFFFFFFFFFFFFFUL;  // bit 0-51
};

template <typename T, typename = typename std::enable_if<std::is_floating_point<T>::value>::type>
inline constexpr bool IsNaN(T num) {
#if !__FINITE_MATH_ONLY__
    return std::isnan(num);
#else
    typename IEEE754Helper<T>::UnionType ieee754{.f = num};
    return (ieee754.u & IEEE754Helper<T>::kExponentMask) == IEEE754Helper<T>::kExponentMask &&
           (ieee754.u & IEEE754Helper<T>::kFactionMask) != 0;
#endif
}

template <typename T, typename = typename std::enable_if<std::is_floating_point<T>::value>::type>
inline constexpr bool IsInf(T num) {
#if !__FINITE_MATH_ONLY__
    return std::isinf(num);
#else
    typename IEEE754Helper<T>::UnionType ieee754{.f = num};
    return (ieee754.u & IEEE754Helper<T>::kExponentMask) == IEEE754Helper<T>::kExponentMask &&
           (ieee754.u & IEEE754Helper<T>::kFactionMask) == 0;
#endif
}

template <typename T, typename = typename std::enable_if<std::is_floating_point<T>::value>::type>
inline constexpr bool IsFinite(T num) {
#if !__FINITE_MATH_ONLY__
    return std::isfinite(num);
#else
    typename IEEE754Helper<T>::UnionType ieee754{.f = num};
    return (ieee754.u & IEEE754Helper<T>::kExponentMask) != IEEE754Helper<T>::kExponentMask;
#endif
}

template <typename T, typename = typename std::enable_if<std::is_floating_point<T>::value>::type>
inline constexpr bool IsNormal(T num) {
#if !__FINITE_MATH_ONLY__
    return std::isnormal(num);
#else
    typename IEEE754Helper<T>::UnionType ieee754{.f = num};
    const auto exponent = ieee754.u & IEEE754Helper<T>::kExponentMask;
    return exponent != IEEE754Helper<T>::kExponentMask && exponent != 0;
#endif
}
```

这样一来，如果项目中开了 `-Ofast` 优化，也能用这个包装函数来做 NaN 等防御，并且如果后续项目不打算用这么激进的优化了，也能无缝回归到 `std::isnan` 等函数的调用，问题就解决了。

最后补充一下各个等级的优化选项的说明，大家以后做项目可以心里有个数：

| 优化等级 | 说明 |
| ------- | ---- |
| `-O0` | 不指定 `-O` 时的默认等级，不做优化，编译最快，变量、断点齐全，最适合调试 |
| `-O1` | 基础优化，只做不显著增加编译时间、不牺牲调试体验的优化 |
| `-O2` | 最常用的发布等级，包含绝大多数不涉及空间与速度权衡的优化，性能与编译时间的平衡点 |
| `-O3` | 在 `-O2` 基础上追加更激进的优化（如更积极的内联、循环展开、向量化），性能可能更高，但代码体积和编译时间也会增加 |
| `-Os` | 以 `-O2` 为基础，只保留不增加代码体积的优化，适合嵌入式等存储、缓存受限的场景 |
| `-Oz` | 比 `-Os` 更激进地压缩体积，速度让位于体积 |
| `-Og` | 以 `-O1` 为基础、针对调试场景筛选的优化，比 `-O0` 快一些又保留了大部分调试体验 |
| `-Ofast` | 在 `-O3` 的基础上追加 `-ffast-math` 等违反标准的优化 |
