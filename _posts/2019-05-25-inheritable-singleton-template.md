---
title: 用于继承树的可继承单例模板类
layout: post
categories: c++
tags:
  - c++
  - singleton
last_modified_at: 2020-01-26T22:00:00+08:00
---
### 用于继承树的可继承单例模板类
最近需要在一个继承树中大量用到单例模式，最经典的单例实现是不够用了。
> 经典单例实现
> ```c++
> class Singleton {
>  public:
>   ~Singleton() = default;
>   Singleton(const Singleton &) = delete;
>   Singleton &operator=(const Singleton &) = delete;
>   Singleton(Singleton &&) = delete;
>   Singleton &operator=(Singleton &&) = delete;
> 
>   /// Get singleton
>   static Singleton &Instance() {
>     static Singleton instance_;
>     return instance_;
>   }
> 
>  private:
>   Singleton() = default;
> };
> ```

就算把构造函数改成protected的允许继承，但static的Instance()函数在每个子类中都得重写。这还只是饿汉模式的情况，如果是懒汉模式，还要定义私有变量储存单例实例，并且要在构造函数中加锁避免多线程问题，重写的代码太多了，实在是bug藏身的好地方。

---

在这种情况下当然首先会想到模板继承的解决方法：
```c++
template <typename T>
class Singleton {
 public:
  virtual ~Singleton() = default;
  Singleton(const Singleton &) = delete;
  Singleton &operator=(const Singleton &) = delete;
  Singleton(Singleton &&) = delete;
  Singleton &operator=(Singleton &&) = delete;

  /// Get singleton
  static T &Instance() {
    static T instance_;
    return instance_;
  }

 protected:
  Singleton() = default;
};
```
模板就是自然的根据不同的类型来自动生成代码的好工具，这个模板类就可以简单地被继承来获得单例模式了。关键有一点，就是要**把这个模板类声明为友元类**，因为它要调用你隐藏起来的构造函数。
```c++
class Object : public Singleton<Object> {
 public:
  ~Object() override = default;
  Object(const Object &) = delete;
  Object &operator=(const Object &) = delete;
  Object(Object &&) = delete;
  Object &operator=(Object &&) = delete;

 private:
  Object() = default;
  friend Singleton<Object>;
};
```
但这个模板在我的继承树里面依然有问题，因为
```c++
class BaseClass : public Singleton<BaseClass>{...};
class DerivedClass : public BaseClass, public Singleton<DerivedClass>{...};
```
里面虽然父类和子类都实现了单例模式，但由于子类是多继承的关系，会存在static的Instance()函数**二义性问题**，把模板的继承改成private的也无法解决。

---

不会又要全部重写一遍Instance()函数的实现来覆盖掉继承下来的Instance()吧？！<br>
**当然不！**C++的**using**语句最终帮我解决了这个问题。只要在每个类中声明一下
```c++
using Singleton<T>::Instance;
```
就可以解决这个二义性问题。

---

然后我的实际需要是Instance()返回基类的引用，因此我最后实现的模板是这样的：
```c++
/**
 * @brief Singleton template for easily deriving from an inheritance tree.
 * @tparam T Class derived from U Class.
 *
 * Guidelines for derived classes:
 *    ** privately derived from Singleton<T, U>
 *    ** hide constructors of T
 *    ** make class Singleton<T, U> friendly
 *    ** declare using Singleton<T, U>::Instance()
 */
template <typename T, typename U = T>
class Singleton {
 public:
  virtual ~Singleton() = default;
  Singleton(const Singleton &) = delete;
  Singleton &operator=(const Singleton &) = delete;
  Singleton(Singleton &&) = delete;
  Singleton &operator=(Singleton &&) = delete;

 protected:
  Singleton() = default;

  /// Get singleton
  static std::shared_ptr<U> Instance() { return instance_; }

 private:
  /// Singleton instance
  static std::shared_ptr<U> instance_;
};

template <typename T, typename U>
std::shared_ptr<U> Singleton<T, U>::instance_ = std::shared_ptr<U>(new T);
```
使用方法在注释中说明：
* 继承private Singleton<T, U>
* 把构造函数隐藏起来
* 声明friend Singleton<T, U>
* 在public中声明using Singleton<T, U>::Instance;

