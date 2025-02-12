---
title: 记录特殊的环形缓存开发过程
layout: post
categories: c++
tags:
  - c++
  - ring buffer
last_modified_at: 2025-02-10T21:00:00+08:00
---

### 原始需求

最近在嵌入式开发中需要重构一下传感器数据读取和分发，特别是需要支持在中断中跟传感器进行通信，然后对读取到的数据进行处理和分发。
这里有以下几个重点要求：

1. 需要适配很多的传感器数据类型
1. 允许在中断中操作数据，并且不能影响正在使用的数据

### 需求分析

第一点要求，适配很多传感器数据类型，通常的方法就是使用模板。

第二点要求，要在中断中操作数据，一般来说就不能新申请一块内存，而且又不能影响到正在使用的数据，最容易想到的是用同一块内存加锁保护来实现，但是中断中不能等待锁，所以只能用某种队列的方式实现，而用链表实现的队列的方式需要新申请内存，也就可以排除了。
这里其实很容易想到环形缓存，用数组的排列方式来提前申请好数据使用的内存，而在中断中写数据和后面读数据的时候操作不同块内存就可以了。

这样就确定了技术方向。

### 最初的环形缓存实现

首先定义一下数据结构，

```c++
template <size_t buffer_size = 3>
class RingBuffer {
 private:
  const size_t size_;
  char *data_;
  size_t front_;
  size_t rear_;
};
```

这里用一个模板参数来定义缓存的数量，是为了使用上方便。数据结构上很简单地定义了数据首指针和读写两个索引量。
基础逻辑实现：

```c++
RingBuffer::RingBuffer(size_t data_size)
    : size_(data_size),
      front_(0),
      rear_(0) {
  data_ = new char[buffer_size * size_];
}

RingBuffer::~RingBuffer() {
  delete[] data_;
}

const void *RingBuffer::GetCurrentBuffer() const {
  return data_ + rear_ * size_;
}

void *RingBuffer::GetNextBuffer() {
  size_t next = (front_ + 1) % buffer_size;
  front_ = next;
  return data_ + front_ * size_;
}

void RingBuffer::SetNextBufferAvailable() {
  rear_ = front_;
}
```

这里有一个取巧，就是我们在实际使用中可以只考虑当前最新的传感器数据，所以只需要在中断的时候拿出一块新的数据块，写完数据之后把读索引更新成这个数据块，之后读数据就是读到了这个最新的数据：

```c++
/** In ISR **/
void *ptr = buffer.GetNextBuffer();
// Note: Read sensor data
buffer.SetNextBufferAvailable();

/** In handler **/
const void *ptr = buffer.GetCurrentBuffer();
// Note: Handle sensor data
```

### 多类型适配

我们使用模板的方式，直接给环形缓存增加一个带类型模板参数的构造函数：

```c++
template <typename T>
struct tag<>;
template <typename T>
RingBuffer::RingBuffer(tag<T>) : RingBuffer(sizeof(T)) {
  for (size_t i = 0; i < buffer_size; ++i) {
    new (data_ + i * size_) T;
  }
}

// Usage
RingBuffer<> buffer(tag<SensorData>);
```

这里还存在一个析构的问题，所以在数据结构中还需要增加析构需要的函数指针：

```c++
using Deleter = void (*)(const void *);
class RingBuffer {
  // ...
  Deleter deleter_;
};

RingBuffer::RingBuffer(size_t data_size, Deleter deleter = nullptr)
    : size_(data_size),
      front_(0),
      rear_(0),
      deleter_(deleter) {
  data_ = new char[buffer_size * size_];
}

RingBuffer::~RingBuffer() {
  if (nullptr != deleter_) {
    for (size_t i = 0; i < buffer_size; ++i) {
      deleter_(data_ + i * size_);
    }
  }
  delete[] data_;
}

RingBuffer::RingBuffer(tag<T>)
    : RingBuffer(
          sizeof(T),
          [](const void * data) { static_cast<const T *>(data)->~T(); }
      ) {
  for (size_t i = 0; i < buffer_size; ++i) {
    new (data_ + i * size_) T;
  }
}
```

同理，我们还可以让构造函数传参，支持不同类型数据的初始化方式，

```c++
template <typename T, typename... Args>
RingBuffer::RingBuffer(tag<T>, Args &&...args)
    : RingBuffer(
          sizeof(T),
          [](const void * data) { static_cast<const T *>(data)->~T(); }
      ) {
  for (size_t i = 0; i < buffer_size; ++i) {
    new (data_ + i * size_) T(std::forward<Args>(args)...);
  }
}
```

### 优化

至此，我们的环形缓存已经可以满足原始需求了。但是在移植完所有的数据类型之后发现编译大小大了好多。仔细检查发现应该是那个模板构造函数导致的。
那我们有没有办法做这个优化呢？既然可以实现 deleter，那一样可以实现 initializer！

```c++
using Initializer = void (*)(void *);
RingBuffer::RingBuffer(size_t data_size, const Initializer &initializer,
                       Deleter deleter = nullptr)
    : RingBuffer(data_size, deleter) {
  if (nullptr != initializer) {
    for (size_t i = 0; i < buffer_size; ++i) {
      initializer(data_ + i * size_);
    }
  }
}

// Usage
template <typename T>
void SensorDataInitializer(void *data) {
  new (data) T;
}
// Note: can specialize if need to pass arguments
// template <>
// void SensorDataInitialize<SpecialData>(void *data) {
//   new (data) SpecialData(foo, bar, ...);
// }

template <typename T>
void SensorDataDeleter(const void *data) {
  static_cast<const T *>(data)->~T();
}

RingBuffer<> buffer(sizeof(SensorData),
                    SensorDataInitializer<SensorData>,
                    SensorDataDeleter<SensorData>);
```
这样实现的关键是initializer和deleter都是模板，并且代码量很少，编译的时候也只有它们会按数据类型展开成很多份代码，也就最小化了编译大小。

### 回顾

最终的代码综合起来就是

```c++
// Ring buffer class
template <size_t buffer_size = 3>
class RingBuffer {
 public:
  using Initializer = void (*)(void *);
  using Deleter = void (*)(const void *);

  RingBuffer(size_t data_size, Deleter deleter = nullptr)
      : size_(data_size),
        front_(0),
        rear_(0),
        deleter_(deleter) {
    data_ = new char[buffer_size * size_];
  }

  ~RingBuffer() {
    if (nullptr != deleter_) {
      for (size_t i = 0; i < buffer_size; ++i) {
        deleter_(data_ + i * size_);
      }
    }
    delete[] data_;
  }

  RingBuffer::RingBuffer(size_t data_size, const Initializer &initializer,
                         Deleter deleter = nullptr)
      : RingBuffer(data_size, deleter) {
    if (nullptr != initializer) {
      for (size_t i = 0; i < buffer_size; ++i) {
        initializer(data_ + i * size_);
      }
    }
  }

  const void *GetCurrentBuffer() const {
    return data_ + rear_ * size_;
  }

  void *GetNextBuffer() {
    size_t next = (front_ + 1) % buffer_size;
    front_ = next;
    return data_ + front_ * size_;
  }

  void SetNextBufferAvailable() {
    rear_ = front_;
  }

 private:
  const size_t size_;
  char *data_;
  size_t front_;
  size_t rear_;
  Deleter deleter_;
};

// Usage
template <typename T>
void SensorDataInitializer(void *data) {
  new (data) T;
}
// Note: can specialize if need to pass arguments
// template <>
// void SensorDataInitialize<SpecialData>(void *data) {
//   new (data) SpecialData(foo, bar, ...);
// }

template <typename T>
void SensorDataDeleter(const void *data) {
  static_cast<const T *>(data)->~T();
}

RingBuffer<> buffer(sizeof(SensorData),
                    SensorDataInitializer<SensorData>,
                    SensorDataDeleter<SensorData>);
/** In ISR **/
void *ptr = buffer.GetNextBuffer();
// Note: Read sensor data
buffer.SetNextBufferAvailable();

/** In handler **/
const void *ptr = buffer.GetCurrentBuffer();
// Note: Handle sensor data
```
