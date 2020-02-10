---
title: 共享内存一写多读无锁实现
layout: post
categories: c++
tags:
  - c++
  - pubsub
  - shared memory
last_modified_at: 2020-02-10T15:00:00+08:00
---
### 共享内存一写多读无锁实现
最近的项目开发需要用到多进程，使用`PubSub`发布订阅模式进行进程间的通信。通信方式可以是`TCP/UDP`，`管道Pipe/消息队列`，`共享内存shared memory`等等。其中`TCP/UDP`的方式是可以用作局域网以及跨平台的通信，`Pipe/消息队列`是进程间基于系统实现比较基础的通信，这两者有大量优秀的第三方库支持，如`ZeroMQ`，只要加入我们自定义数据的转换方式即可方便实现；而`共享内存`是实现进程间通信最快的方式，但因为`共享内存`的设计并不是用来做类似`PubSub`这种模式的实现的，并且`共享内存`实质上就是一段进程间共享的内存空间，使用自由度是极高的，所以也很少有第三方库来实现`共享内存`方式的进程间通信。
#### 因此本文的重点是如何使用`共享内存shared memory`来实现高效的`PubSub`发布订阅模式。
---

首先我们整理一下需求：
1. 消息通过事先分配好的共享内存空间来传递
2. 需要有一定的机制来管理消息的发送（写）和接收（读）
3. 需要实现发布订阅模式，也就是一个发布者（一写）多个订阅者（多读）

首先第**1**点不同的系统有不同的实现，比如`Linux`的`shmget`一系列的函数可以很容易地管理分配的共享内存，但是`Android`的`shmget`就被阉割了。通过对比不同的实现，我们最后采用了文件映射内存的这种方式，在各种系统中都有比较通用的实现。例如`Linux`中：
```c++
// open file descriptor
int fd = open(file_name, O_RDWR | O_CREAT | O_EXCL, 0600);
if (fd < 0) {
  fd = open(file_name, O_RDWR, 0600);
}
// set file size
struct stat fs;
fstat(fd, &fs);
if (fs.st_size < 1) {
  ftruncate(fd, file_size);
}
// mmap
void *shm = mmap(NULL, file_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);

// unmap
munmap(shm, file_size);
// remove file
remove(file_name);
```

显然，只要创建了一个文件并且设置好需要的大小，即可以使用`mmap`映射到进程的内存空间，并且在退出时可以用`munmap`将映射释放掉。但是空间真正的释放是要把文件删掉的，因此我们需要一个计数器来记录使用这块共享内存的进程数，类似共享指针`shared_ptr`的实现，在计数为零时把文件删掉。在修改这个计数的时候还需要一把进程间读写锁：
```c++
struct ShmSlice {
  int attached_;
  pthread_rwlock_t rwlock_;
  char data_[1];

  ShmSlice(const bool init = false) {
    if (init) {
      pthread_rwlockattr_t rwattr;
      pthread_rwlockattr_init(&rwattr);
      pthread_rwlockattr_setpshared(&rwattr, PTHREAD_PROCESS_SHARED);
      pthread_rwlock_init(&rwlock_, &rwattr);
    }
    LockWrite();
    if (init) {
      attached_ = 1;
    } else {
      ++attached_;
    }
    UnlockWrite();
  }
  ~ShmSlice() {
    LockWrite();
    const int count = --attached_;
    UnlockWrite();
    if (0 == count) {
      pthread_rwlock_destroy(&rwlock_);
    }
  }
  int count() {
    LockRead();
    const int count = attached_;
    UnlockRead();
    return count;
  }

  void LockWrite() { pthread_rwlock_wrlock(&rwlock_); }
  void UnlockWrite() { pthread_rwlock_unlock(&rwlock_); }
  void LockRead() { pthread_rwlock_rdlock(&rwlock_); }
  void UnlockRead() { pthread_rwlock_unlock(&rwlock_); }
};

class ShmManager {
 public:
  ShmManager(std::string file_name, const int file_size)
      : name_(std::move(file_name)), size_(file_size) {
    bool init = false;
    // open file descriptor
    int fd = open(name_, O_RDWR | O_CREAT | O_EXCL, 0600);
    if (fd < 0) {
      fd = open(name_, O_RDWR, 0600);
    } else {
      // set file size
      struct stat fs;
      fstat(fd, &fs);
      if (fs.st_size < 1) {
        ftruncate(fd, size_);
      }
      init = true;
    }
    // mmap
    void *shmaddr = mmap(NULL, size_, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    new(shmaddr) ShmSlice(init);
    auto deleter = [](ShmSlice *ptr) { ptr->~ShmSlice(); }
    slice_ = std::shared_ptr<ShmSlice>(reinterpret_cast<ShmSlice *>(shmaddr), deleter);
    close(fd);
  }
  ~ShmManager() {
    const int count = slice_->count();
    auto ptr = slice_.get();
    slice_.reset();
    if (count > 1) {
      // unmap
      munmap(ptr, size_);
    } else {
      // remove file
      remove(name_);
    }

 private:
  std::string name_;
  int size_;
  std::shared_ptr<ShmSlice> slice_;
};
```

这样共享内存空间的分配和释放就做好了。

---

接下来是**2**，实现消息发送（写）和接收（读）的管理。因为我们已经有了一把读写锁，很自然地想到可以用它来管理读写啊。事实上并不是这样，因为发布者**写**完数据之后可能会有一段时间不会占有写锁，这时候就要一种机制来限制订阅者不会重复来**读**这个数据。对于这个实现，已有的方案有：
1. 对于只有单个订阅者，数据之后包含一个标志位，发布者**写**完后置为`true`，订阅者**读**完之后置为`false`，可能再加上一个**信号灯**的控制，来避免频繁读写；
2. 对于多个订阅者，数据中的这个标志位变成一个计数，发布者**写**完之后将计数器置为**订阅者的数量**，订阅者**读**完之后将计数器**减1**，再加上一个**进程条件变量**的控制，来避免频繁读写。

这两种方案都有一定的弊端，最大的问题在于，订阅者还需要修改共享内存的内容，这样就发挥不出读写锁支持多读的优势了。我们需要一个更好的机制。

一个简单的实现是数据中带有一个单调递增的标签，订阅者读到数据后本地保存一下这个标签的值，如果下次读到的这个值不比保存的值大，就认为读到了旧数据，忽略之。这个标签比较好的实现是用当前的系统时间而不是计数，因为发布者可能会重启清零，就算重启后可以从已经写入的数据中读取，但后面为了实现无锁队列会让这个事情变得麻烦。这样还有一个问题是，依然会频繁地去读取这个标签。因此需要加入**进程条件变量**的控制来减少这种频繁。
```c++
struct ShmData {
  long timestamp_;
  size_t size_;
  char data_[1];

  void Write(const char *data, const size_t len) {
    memcpy(data_, data, len);
    size_ = len;
    timestamp_ = GetTimestamp();
  }
  size_t Read(char *data, long *time = nullptr) {
    if (time) {
      *time = timestamp_;
    }
    memcpy(data, data_, size_);
    return size_;
  }

  static long GetTimestamp() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    return ts.tv_sec * 1000000 + ts.tv_nsec / 1000;
  }
};

struct ShmSlice {
  int attached_;
  pthread_rwlock_t rwlock_;
  pthread_mutex_t mutex_;
  pthread_cond_t cond_;
  char data_[1];

  ShmSlice(const bool init = false) {
    if (init) {
      // init rwlock
      pthread_rwlockattr_t rwattr;
      pthread_rwlockattr_init(&rwattr);
      pthread_rwlockattr_setpshared(&rwattr, PTHREAD_PROCESS_SHARED);
      pthread_rwlock_init(&rwlock_, &rwattr);
      // init mutex
      pthread_mutexattr_t mattr;
      pthread_mutexattr_init(&mattr);
      pthread_mutexattr_setpshared(&mattr, PTHREAD_PROCESS_SHARED);
      pthread_mutex_init(&mutex_, &mattr);
      // init condition variable
      pthread_condattr_t cattr;
      pthread_condattr_init(&cattr);
      pthread_condattr_setpshared(&cattr, PTHREAD_PROCESS_SHARED);
      pthread_cond_init(&cond_, &cattr);
    }
    LockWrite();
    if (init) {
      attached_ = 1;
    } else {
      ++attached_;
    }
    UnlockWrite();
  }
  ~ShmSlice() {
    LockWrite();
    const int count = --attached_;
    UnlockWrite();
    if (0 == count) {
      pthread_cond_destroy(&cond_);
      pthread_mutex_destroy(&mutex_);
      pthread_rwlock_destroy(&rwlock_);
    }
  }
  int count() {
    LockRead();
    const int count = attached_;
    UnlockRead();
    return count;
  }

  void Write(const char *data, const size_t len) {
    LockWrite();
    (reinterpret_cast<ShmData *>(data_))->Write(data, len);
    UnlockWrite();
  }
  size_t Read(char *data, long *time) {
    LockRead();
    const size_t size = (reinterpret_cast<ShmData *>(data_))->Read(data, time);
    UnlockRead();
    return size;
  }

  void LockWrite() { pthread_rwlock_wrlock(&rwlock_); }
  void UnlockWrite() { pthread_rwlock_unlock(&rwlock_); }
  void LockRead() { pthread_rwlock_rdlock(&rwlock_); }
  void UnlockRead() { pthread_rwlock_unlock(&rwlock_); }

  void LockMutex() {
    while(EOWNERDEAD == pthread_mutex_lock(&mutex_)) {
      UnlockMutex();
    }
  }
  void UnlockMutex() { pthread_mutex_unlock(&mutex_); }
  void NotifyOne() { pthread_cond_signal(&cond_); }
  void NotifyAll() { pthread_cond_broadcast(&cond_); }
  void Wait() {
    LockMutex();
    pthread_cond_wait(&cond_, &mutex_);
    UnlockMutex();
  }
  bool WaitFor(struct timespec *ts, const std::function<bool()> &cond) {
    if (cond && cond()) {
      return true;
    }
    LockMutex();
    pthread_cond_timedwait(&cond_, &mutex_, ts);
    UnlockMutex();
    bool ret;
    if (cond) {
      ret = cond();
    } else {
      struct timespec now;
      clock_gettime(CLOCK_REALTIME, &now);
      ret = now.tv_sec < ts->tv_sec ||
            (now.tv_sec == ts->tv_sec && now.tv_nsec <= ts->tv_nsec);
    }
    return ret;
  }
};

class ShmManger {
 public:
  void Write(const char *data, const size_t len) {
    slice_->Write(data, len);
    slice_->NotifyAll();
  }
  void ReadThread() {
    long read_time = 0;
    while (running_) {
      data = new char[size_];
      size_t len;
      long time;
      struct timespec ts;
      clock_gettime(CLOCK_REALTIME, &ts);
      ts.tv_sec += 5;
      if (!slice_->WaitFor(&ts, [&]{
            len = slice_->Read(data, &time);
            return time > read_time;
          })) {
        delete[] data;
        continue;
      }
      read_time = time;
      // deal with data
      delete[] data;
    }
  }

 private:
  std::string name_;
  int size_;
  std::shared_ptr<ShmSlice> slice_;
  std::atomic_bool running_;
};
```

这样就实现了发布者每次**写**数据时打上当前的时间戳，**写**完数据之后通过**进程间条件变量**通知到所有的订阅者，而订阅者开启**读**线程来等待**进程间条件变量**的通知并读取数据，根据读取数据的时间戳来判断是否为新数据，处理新数据而忽略旧数据。

---

这样一来，基本的基于共享内存的发布订阅模式就实现了。还剩最后一个需求，真正地实现**一写多读**。刚才我们使用读写锁来做一写多读存在这样一个问题，订阅者一旦多起来的时候，很可能在读数据的时候占用了读锁，导致发布者拿不到写锁，写数据受到了约束，极大地影响了通信的效率。解决这个问题可以有如下的方案：
1. 对于每一个订阅者都开辟一块共享内存，可以按一对一的方式同时复制多份数据；
2. 使用生产消费模式，使用循环队列来实现读写分离。

第**1**种方案是解决了读写锁争抢的问题，但是增加了内存复制的开销，反而没有第**2**种方案好。但是我们要稍微修改一下传统的生产消费模式的实现，只用一个指针来指向最新的数据。之所以这样做是因为内存是事先分配好的，我们把它改造成环形的内存缓冲区，很难保证数据读取的序列性；再者就是循环的尾指针应该由订阅者自己来维护，因为每个订阅者处理的速度是不一样的。

如此一来，所有数据的修改完全是由发布者来做的，也就是说对于订阅者来说，这是个无锁队列：
```c++
struct ShmData {
  bool written_;
  long timestamp_;
  size_t size_;
  char data_[1];

  void Write(const char *data, const size_t len) {
    written_ = false;
    memcpy(data_, data, len);
    size_ = len;
    timestamp_ = GetTimestamp();
    written_ = true;
  }
  bool Read(std::vector<char> *data, long *time = nullptr) {
    if (!written_) { return false; }
    if (time) { *time = timestamp_; }
    data->resize(size_);
    memcpy(data->data(), data_, size_);
    return true;
  }

  static long GetTimestamp() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    return ts.tv_sec * 1000000 + ts.tv_nsec / 1000;
  }
};

struct ShmQueue {
  size_t size_;
  int count_;
  int head_;
  char data_[1];

  ShmQueue(const size_t size, const int count)
      : size_(size), count_(count), head_(0) {}

  void Write(const char *data, const size_t len) {
    const int next = (head_ + 1) % count_;
    auto shmdata = reinterpret_cast<ShmData *>(data_ + next * size_);
    shmdata->Write(data, len);
    head_ = next;
  }
  bool Read(std::vector<char> *data, long *time) {
    auto shmdata = reinterpret_cast<ShmData *>(data_ + head_ * size_);
    return shmdata->Read(data, time);
  }
};
```

---
#### 至此就实现了所有的需求。最后整理一下代码。
```c++
struct ShmData {
  bool written_;
  long timestamp_;
  size_t size_;
  char data_[1];

  ShmData() : written_(false) {}

  void Write(const char *data, const size_t len) {
    written_ = false;
    memcpy(data_, data, len);
    size_ = len;
    timestamp_ = GetTimestamp();
    written_ = true;
  }
  bool Read(std::vector<char> *data, long *time = nullptr) {
    if (!written_) { return false; }
    if (time) { *time = timestamp_; }
    data->resize(size_);
    memcpy(data->data(), data_, size_);
    return true;
  }

  static long GetTimestamp() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    return ts.tv_sec * 1000000 + ts.tv_nsec / 1000;
  }
};

struct ShmQueue {
  size_t size_;
  int count_;
  int head_;
  char data_[1];

  ShmQueue(const size_t size, const int count)
      : size_(sizeof(ShmData) + size), count_(count), head_(0) {
    new(data_) ShmData();
  }

  void Write(const char *data, const size_t len) {
    const int next = (head_ + 1) % count_;
    (reinterpret_cast<ShmData *>(data_ + next * size_))->Write(data, len);
    head_ = next;
  }
  bool Read(std::vector<char> *data, long *time) {
    return (reinterpret_cast<ShmData *>(data_ + head_ * size_))->Read(data, time);
  }
};

struct ShmSlice {
  int attached_;
  pthread_rwlock_t rwlock_;
  pthread_mutex_t mutex_;
  pthread_cond_t cond_;
  char data_[1];

  ShmSlice(const size_t size, const int count, const bool init = false) {
    if (init) {
      // init rwlock
      pthread_rwlockattr_t rwattr;
      pthread_rwlockattr_init(&rwattr);
      pthread_rwlockattr_setpshared(&rwattr, PTHREAD_PROCESS_SHARED);
      pthread_rwlock_init(&rwlock_, &rwattr);
      // init mutex
      pthread_mutexattr_t mattr;
      pthread_mutexattr_init(&mattr);
      pthread_mutexattr_setpshared(&mattr, PTHREAD_PROCESS_SHARED);
      pthread_mutex_init(&mutex_, &mattr);
      // init condition variable
      pthread_condattr_t cattr;
      pthread_condattr_init(&cattr);
      pthread_condattr_setpshared(&cattr, PTHREAD_PROCESS_SHARED);
      pthread_cond_init(&cond_, &cattr);
      // init shm queue
      new(data_) ShmQueue(size, count);
    }
    LockWrite();
    if (init) {
      attached_ = 1;
    } else {
      ++attached_;
    }
    UnlockWrite();
  }
  ~ShmSlice() {
    LockWrite();
    const int count = --attached_;
    UnlockWrite();
    if (0 == count) {
      pthread_cond_destroy(&cond_);
      pthread_mutex_destroy(&mutex_);
      pthread_rwlock_destroy(&rwlock_);
    }
  }
  int count() {
    LockRead();
    const int count = attached_;
    UnlockRead();
    return count;
  }

  void Write(const char *data, const size_t len) {
    LockWrite();
    (reinterpret_cast<ShmQueue *>(data_))->Write(data, len);
    UnlockWrite();
  }
  bool Read(std::vector<char> *data, long *time) {
    return (reinterpret_cast<ShmQueue *>(data_))->Read(data, time);
  }

  void LockWrite() { pthread_rwlock_wrlock(&rwlock_); }
  void UnlockWrite() { pthread_rwlock_unlock(&rwlock_); }
  void LockRead() { pthread_rwlock_rdlock(&rwlock_); }
  void UnlockRead() { pthread_rwlock_unlock(&rwlock_); }

  void LockMutex() {
    while(EOWNERDEAD == pthread_mutex_lock(&mutex_)) {
      UnlockMutex();
    }
  }
  void UnlockMutex() { pthread_mutex_unlock(&mutex_); }
  void NotifyOne() { pthread_cond_signal(&cond_); }
  void NotifyAll() { pthread_cond_broadcast(&cond_); }
  void Wait() {
    LockMutex();
    pthread_cond_wait(&cond_, &mutex_);
    UnlockMutex();
  }
  bool WaitFor(struct timespec *ts, const std::function<bool()> &cond) {
    if (cond && cond()) {
      return true;
    }
    LockMutex();
    pthread_cond_timedwait(&cond_, &mutex_, ts);
    UnlockMutex();
    bool ret;
    if (cond) {
      ret = cond();
    } else {
      struct timespec now;
      clock_gettime(CLOCK_REALTIME, &now);
      ret = now.tv_sec < ts->tv_sec ||
            (now.tv_sec == ts->tv_sec && now.tv_nsec <= ts->tv_nsec);
    }
    return ret;
  }
};

class ShmManger {
 public:
  ShmManager(std::string file_name, const int size)
      : name_(std::move(file_name)),
        size_(sizeof(ShmSlice) + sizeof(ShmQueue) + 3 * (sizeof(ShmData) + size)) {
    bool init = false;
    // open file descriptor
    int fd = open(name_, O_RDWR | O_CREAT | O_EXCL, 0600);
    if (fd < 0) {
      fd = open(name_, O_RDWR, 0600);
    } else {
      // set file size
      struct stat fs;
      fstat(fd, &fs);
      if (fs.st_size < 1) {
        ftruncate(fd, size_);
      }
      init = true;
    }
    // mmap
    void *shmaddr = mmap(NULL, size_, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    new(shmaddr) ShmSlice(size, 3, init);
    auto deleter = [](ShmSlice *ptr) { ptr->~ShmSlice(); }
    slice_ = std::shared_ptr<ShmSlice>(reinterpret_cast<ShmSlice *>(shmaddr), deleter);
    close(fd);
  }
  ~ShmManager() {
    running_ = false;
    slice_->NotifyAll();
    if (read_thread_.joinable()) { read_thread_.join(); }
    const int count = slice_->count();
    auto ptr = slice_.get();
    slice_.reset();
    if (count > 1) {
      // unmap
      munmap(ptr, size_);
    } else {
      // remove file
      remove(name_);
    }

  void Publish(const std::vector<char> &data) {
    slice_->Write(data.data(), data.size());
    slice_->NotifyAll();
  }
  void Subscribe(std::function<void(const std::vector<char> &)> callback) {
    callback_ = std::move(callback);
    running_ = true;
    read_thread_ = std::thread(&ShmManger::ReadThread, this);
  }

 private:
  void ReadThread() {
    long read_time = 0;
    while (running_) {
      std::vector<char> data;
      long time;
      struct timespec ts;
      clock_gettime(CLOCK_REALTIME, &ts);
      ts.tv_sec += 5;
      if (!slice_->WaitFor(&ts, [&]{
            return slice_->Read(&data, &time) && time > read_time;
          })) {
        continue;
      }
      read_time = time;
      // deal with data
      callback_(data);
    }
  }

  std::string name_;
  int size_;
  std::shared_ptr<ShmSlice> slice_;
  std::function<void(const std::vector<char> &)> callback_;
  std::atomic_bool running_;
  std::thread read_thread_;
};
```

享受共享内存的超高效率吧！

