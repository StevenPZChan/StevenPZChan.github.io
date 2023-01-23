---
title: 行为树基础实现
layout: post
categories: c++
tags:
  - c++
  - behavior tree
last_modified_at: 2023-01-24T23:00:00+08:00
---

### 行为树基础实现
最近的项目需要移植行为树，所以我设计了一个较为简单的行为树框架，在这分享出来。

#### 行为树概述

一般跟行为树相提并论的是状态机。我们在实现一个物体的某种行为的时候，一般最先想到的是使用状态机，因为它很容易实现，在行为复杂度不太高的情况下，实现出来很快，也不容易出错，是我们对新功能做演示时的首选。但是一旦行为开始变得复杂，状态数量不断增加的情况下，状态机的维护难度急剧升高，代码逻辑出错的可能性就很高了。这个时候我们就应该考虑使用行为树了。行为树的优势在于其树状结构，对于不同尺度的逻辑可以进行对应的修改，不会牵一发而动全身。同时也由于其树状结构，行为树是可以分块进行测试的。

但是行为树也有明显的缺点，就是需要有一个完备的运行引擎，以保证程序能正确执行设计的行为。这个引擎必须从树的根部开始运行（因此简单的行为使用行为树实际上会比状态机要花更多的时间，但是这个并没有那么重要，因为我们选择行为树自然是行为复杂度已经比较高了），按照树的设计和物体当前的状态进行节点的遍历，必须保证在同一时刻有且仅有某一个行为在执行（并行节点除外）。此外，为了使物体对变化的状态有快速的响应，节点的遍历必须要足够快，行为的执行必须可以被随时中断。

因此我在这里把我设计的这个简单实用的完备的行为树运行引擎分享出来。

#### 类型定义

行为树必要的类型定义只有一个，是节点当前运行的状态。

```c++
enum class NodeStatus {
  kIdle,     // Do nothing
  kRunning,  // Ongoing
  kSuccess,
  kFailure,
};
```

其中`kIdle`状态指的是节点初始状态，或者没有被遍历到的情况，`kRunning`是指行为节点正在被执行，或者是被执行节点的所有祖先节点，`kSuccess`和`kFailure`指的是条件节点和行为节点的运行结果，或者控制节点和装饰节点的子节点状态的汇总。

#### 黑板

行为树框架中另外一个很重要的元素就是黑板，它是行为树各个节点之间信息交流的场所。通常我们使用一个`string: string`的哈希表来做存储，需要我们手动实现类型转换部分。

```c++
template <typename Target, typename Source>
Target lexical_cast(const Source &arg);

class Blackboard {
 public:
  Blackboard();
  virtual ~Blackboard();

  template <typename T>
  typename std::remove_reference<T>::type Get(
      const std::string &key, T &&default_value = T()) const {
    std::lock_guard<Spinlock> lock(lock_);
    const auto iter = storage_.find(key);
    if (iter == storage_.end()) {
      return std::forward<T>(default_value);
    }
    typedef typename std::remove_cv<
        typename std::remove_reference<T>::type>::type Target;
    return lexical_cast<Target>(iter->second);
  }

  template <typename T>
  void Set(const std::string &key, T &&value) {
    const auto &str_value =
        lexical_cast<std::string>(std::forward<T>(value));
    std::lock_guard<Spinlock> lock(lock_);
    auto ret = storage_.emplace(key, str_value);
    if (!ret.second) {
      ret.first->second = str_value;
    }
  }

 private:
  mutable Spinlock lock_;
  std::unordered_map<std::string, std::string> storage_;
};
```

其中有几个细节我想说明一下：

1. 锁必须要使用自旋锁，因为在行为树运行引擎的遍历过程中，`Get`函数会被调用得很频繁，只有自旋锁才能满足遍历的速度要求。而`Set`函数一般只在状态数据有变化的时候才会被调用，自旋锁被争抢的机会其实很少。
2. `T &&`和`std::forward<T>`的正确使用，可以极大程度地利用到移动语义来减少拷贝。
3. `Blackboard`类可以被继承，扩展出一些更实用的状态获取和保存的方法，所以析构函数是虚函数。

#### 行为树节点

然后就是一个表达行为树每个节点的抽象基类。行为树的每个节点共同必备三部分内容，一是节点状态的记录和获取，二是节点的控制方法，三是节点从黑板获取和保存数据的途径。

```c++
class NodeBase {
 public:
  NodeBase()
      : status_(NodeStatus::kIdle),
        blackboard_(std::make_shared<Blackboard>()) {}
  virtual ~NodeBase() = default;
  NodeBase(const NodeBase &other) = delete;
  NodeBase &operator=(const NodeBase &other) = delete;

  NodeStatus GetStatus() const {
    std::lock_guard<Spinlock> lock(status_lock_);
    return status_;
  }

  virtual NodeStatus Tick() = 0;
  virtual void Halt() { SetStatus(NodeStatus::kIdle); }
  virtual void SetGlobalBlackboard(std::shared_ptr<Blackboard> blackboard) {
    global_blackboard_ = std::move(blackboard);
  }

 protected:
  const Blackboard *GetBlackboard() const { return blackboard_.get(); }
  Blackboard *GetBlackboard() { return blackboard_.get(); }
  const Blackboard *GetParentBlackboard() const {
    return parent_blackboard_.get();
  }
  Blackboard *GetParentBlackboard() { return parent_blackboard_.get(); }
  const Blackboard *GetGlobalBlackboard() const {
    return global_blackboard_.get();
  }
  Blackboard *GetGlobalBlackboard() { return global_blackboard_.get(); }

  void SetStatus(NodeStatus node_status) {
    std::lock_guard<Spinlock> lock(status_lock_);
    status_ = node_status;
  }

  void SetParentBlackboard(NodeBase *child_node) {
    if (child_node) {
      child_node->parent_blackboard_ = blackboard_;
    }
  }

 private:
  mutable Spinlock status_lock_;
  NodeStatus status_;

  std::shared_ptr<Blackboard> blackboard_;
  std::shared_ptr<Blackboard> parent_blackboard_;
  std::shared_ptr<Blackboard> global_blackboard_;
};
```

同样是细节讲解：

1. 节点的状态获取和保存显然是一个高频的操作，所以锁必须要用自旋锁。实际上行为树的运行引擎是运行在同一个线程的，自旋锁几乎没有消耗。发生锁争抢的唯一场景是有更高层级的调用使得运行引擎强制中断。
2. 节点的控制方法只有两个，`Tick()`和`Halt()`，前者是在节点遍历的时候做的操作，后者是节点需要重置时做的操作。两者都是虚函数，需要根据所继承的节点的特点来进行重写。
3. 行为树中进行节点间信息交流的场所只有黑板，但是如果一棵很大的行为树只有一块黑板的话数据量会很大，而且很容易造成名称污染。所以这里我设计每个节点都拥有一块黑板，该黑板可以被这个节点的所有子节点访问，同时也有一块全局的黑板，以方便层级差别很大的两个节点进行信息交流。

#### 行为树运行引擎

这样一来，行为树运行引擎可以大致表达为如下：

```c++
class ObjectData : public Blackboard {
  /// Some implementation
};
class SampleTree : public NodeBase {
 public:
  NodeStatus Tick() override { /* Some implementation */ }
  void Halt() override { /* Some implementation */ }
};

data_ = std::make_shared<ObjectData>();
bt_ = SampleTree();  /// Some implementation to setup the behavior tree
bt_.SetGlobalBlackboard(data_);

void ExecuteThread() {
  while (running_) {
    const auto status = bt_.Tick();
    std::this_thread::sleep_for(/* some time */);
  }
}

void Exit() {
  running_ = false;
  bt_.Halt();
}
```

### 简单吧？

#### 行为树各类型节点

最后再分享下行为树几个常见的类型的节点实现。

* 条件节点

```c++
class ConditionNode : public NodeBase {
 public:
  NodeStatus Tick() final {
    const bool ret = CheckCondition();
    const auto status = ret ? NodeStatus::kSuccess : NodeStatus::kFailure;
    const auto prev_status = GetStatus();
    if (status != prev_status) {
      SetStatus(status);
    }
    return status;
  }

 protected:
  virtual bool CheckCondition() = 0;
};
```

* 行为节点

```c++
class ActionNode : public NodeBase {
 public:
  NodeStatus Tick() final {
    const auto prev_status = GetStatus();
    if (NodeStatus::kSuccess == prev_status ||
      NodeStatus::kFailure == prev_status) {
      return prev_status;
    }

    const auto status = ExecuteTick();
    SetStatus(status);
    return status;
  }

  void Halt() final {
    ExecuteHalt();
    NodeBase::Halt();
  }

 protected:
  virtual NodeStatus ExecuteTick() = 0;
  virtual void ExecuteHalt() = 0;
};
```

* 控制节点

```c++
class ControlNode : public NodeBase {
 public:
  NodeStatus Tick() final {
    const auto status = ExecuteTick();
    SetStatus(status);
    return status;
  }

  void Halt() final {
    HaltChildren();
    ExecuteHalt();
    NodeBase::Halt();
  }

  void SetGlobalBlackboard(std::shared_ptr<Blackboard> blackboard) final {
    for (auto &child : children_) {
      child->SetGlobalBlackboard(blackboard);
    }
    NodeBase::SetGlobalBlackboard(std::move(blackboard));
  }

  size_t GetChildrenNumber() const { return children_.size(); }
  std::vector<const NodeBase *> GetChildren() const {
    std::vector<const NodeBase *> ret;
    for (auto &child : children_) {
      ret.emplace_back(child.get());
    }
    return ret;
  }

 protected:
  const NodeBase *GetChild(size_t i) const { return children_.at(i).get(); }
  NodeBase *GetChild(size_t i) { return children_.at(i).get(); }

  virtual bool AddChild(std::unique_ptr<NodeBase> node) {
    SetParentBlackboard(node.get());
    children_.emplace_back(std::move(node));
    return true;
  }

  void HaltChildren() {
    for (auto &child : children_) {
      child->Halt();
    }
  }
  void HaltChild(size_t i) { GetChild(i)->Halt(); }

  virtual NodeStatus ExecuteTick() = 0;
  virtual void ExecuteHalt() = 0;

 private:
  std::vector<std::unique_ptr<NodeBase>> children_;
};
```

* 装饰节点

```c++
class DecoratorNode : public NodeBase {
 public:
  NodeStatus Tick() final {
    const auto status = ExecuteTick();
    SetStatus(status);
    return status;
  }

  void Halt() final {
    HaltChild();
    ExecuteHalt();
    NodeBase::Halt();
  }

  void SetGlobalBlackboard(std::shared_ptr<Blackboard> blackboard) final {
    if (child_) {
      child_->SetGlobalBlackboard(blackboard);
    }
    NodeBase::SetGlobalBlackboard(std::move(blackboard));
  }

  const NodeBase *GetChild() const { return child_.get(); }

 protected:
  NodeBase *GetChild() { return child_.get(); }

  virtual inline bool SetChild(std::unique_ptr<NodeBase> node) {
    return SetChildImpl(std::move(node));
  }

  void HaltChild() { child_->Halt(); }

  virtual NodeStatus ExecuteTick() = 0;
  virtual void ExecuteHalt() = 0;

 private:
  bool SetChildImpl(std::unique_ptr<NodeBase> node) {
    SetParentBlackboard(node.get());
    child_ = std::move(node);
    return static_cast<bool>(child_);
  }

  std::unique_ptr<NodeBase> child_;
};
```

