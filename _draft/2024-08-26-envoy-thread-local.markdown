---
layout: post
title:  "Envoy ThreadLocal 机制"
date:   2024-08-26 18:00:00 +0800
---

* TOC
{:toc}


[这篇文章](https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310) 提到了 TLS 机制，利用这种方式可以将主线程中的配置分发给若干工作线程。通常对于共享资源的访问需要加锁，利用这种 TLS 机制可以避免每次访问都加锁。

## 原理

它的原理很简单：
- 主线程和工作线程都在 TLS 上维护要共享的数据。主线程发起数据的更新，工作线程只会读取数据。
- 主线程发起数据更新的方式是向工作线程的 event loop 投递任务，在工作线程中完成数据的更新。

以下是一段示例代码：

```cpp
class ThreadLocalObject {
public:
  virtual ~ThreadLocalObject() = default;
};

class ThreadLocalInstance {
public:
  void update(std::shared_ptr<ThreadLocalObject> new_data) {
    // 更新数据的时候给每个 worker 线程 post 一个任务，在这个任务中进行 TLS 的更新
    for (auto dispatcher : registered_threads_) {
      dispatcher->post([new_data]() {
        // Worker 线程中跟新 TLS
        set(new_data);
      });
    }

    set(new_data);
  }

  std::shared_ptr<ThreadLocalObject> get() {
    // worker 线程从自己的 TLS 中获取数据，不需要加锁
    return shared_data_;
  }

private:
  static void set(std::shared_ptr<ThreadLocalObject> new_data) {
    // 这一步是线程安全的，由 shared_ptr 的原子操作进行保证
    shared_data_ = new_data;
  }

private:
  // 在每个线程的 TLS 上维护一个数据的指针
  static thread_local std::shared_ptr<ThreadLocalObject> shared_data_;

  // 所有 worker 线程
  std::vector<Dispatcher*> registered_threads_;
};
```

可以看出这种方式本质是在利用 event loop 避免数据的并发访问，使用 `thread_local` 只是为了方便把数据绑定到线程上。

注意这里使用了 `std::shared_ptr` 来引用数据，主线程更新数据之后，旧数据不会立刻销毁，因为工作线程的 TLS 中仍然通过 `std::shared_ptr` 引用着旧数据。


## 实现

具体的实现稍微复杂一些，上面的例子中只能维护一份数据，实现中则是通过 slots 的分配使得 TLS 可以维护多份数据:

```plantuml
@startuml

interface Slot {
    ThreadLocalObjectSharedPtr get();
}

interface SlotAllocator{
    SlotPtr allocateSlot();
}

interface Instance{
    void registerThread(Dispatcher& dispatcher);
}

interface ThreadLocalObject{}

SlotAllocator <|-- Instance

class SlotImpl{
    InstanceImpl& parent_;
    const uint32_t index_;
}

class InstanceImpl{
    static thread_local std::vector<ThreadLocalObjectSharedPtr> data_;

    std::vector<Slot*> slots_;
    std::list<Dispatcher*> registered_threads_;
}

class CustomThreadLocalObject {
    string data1;
    string data2;
}

Slot <|-- SlotImpl
ThreadLocalObject <|-- CustomThreadLocalObject
Instance <|-- InstanceImpl
InstanceImpl *-- Slot: 1...n
InstanceImpl *-- ThreadLocalObject: 1...n

Slot .> ThreadLocalObject

@enduml
```

如类图所示，主要数据存储在 `InstanceImpl` 当中: 
- `data_` 成员，它是一个 TLS 上的数组，保存了若干 `ThreadLocalObject` 对象。
- `slots_` 成员，`Slot` 封装了针对某个槽位的 get set 操作:
  - 调用 `Slot::get()` 方法可以通过 index 找到当前线程 TLS 上的 `ThreadLocalObject` 对象
  - 调用 `Slot::set()` 方法可将配置分发到所有工作线程。
- `registered_threads_` 成员，所有工作线程。