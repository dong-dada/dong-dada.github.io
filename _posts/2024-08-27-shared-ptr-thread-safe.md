---
layout: post
title:  "std::shared_ptr 的线程安全性"
date:   2024-08-27 12:00:00 +0800
---

* TOC
{:toc}

## 线程安全性

关于 `std::shared_ptr` 的线程安全性, [cppreference](https://en.cppreference.com/w/cpp/memory/shared_ptr) 中提到:
> All member functions (including copy constructor and copy assignment) can be called by multiple threads on different shared_ptr objects without additional synchronization even if these objects are copies and share ownership of the same object. If multiple threads of execution access the same shared_ptr object without synchronization and any of those accesses uses a non-const member function of shared_ptr then a data race will occur; the std::atomic<shared_ptr> can be used to prevent the data race.

前面一句话的意思是“即使不同线程中的不同 shared_ptr 指向相同对象，也可以安全地调用所有成员函数”。后面一句话的意思是“如果在多个线程中访问同一个 shared_ptr 指针，其中有线程使用了非 const 成员，那么会出现发生数据争用，此时可以使用 `std::atomic<shared_ptr>`”。


也就是说下面的代码是线程安全的，尽管不能确定对象是在哪个线程中被释放，但是可以保证对象只被释放一次:

```cpp
int main() {
  std::shared_ptr<int> p1(new int(42));

  std::thread t1([p1]() {
    // p1 和 p2 指向同一个对象
    std::shared_ptr<int> p2 = p1;

    // 释放 p1 和 p2
    p1.reset();
    p2.reset();
  });

  // 释放 p1
  p1.reset();
  t1.join();
  return 0;
}
```

但下面代码不是线程安全的，因为在两个线程中访问了同一个 `std::shared_ptr` 指针:

```cpp
int main() {
  std::shared_ptr<int> p1(new int(42));

  std::thread t1([&p1]() {
    // 使用 p1
    cout << p1.use_count() << endl;
  });

  // 释放 p1
  p1.reset();
  t1.join();
  return 0;
}
```


## 实现原理

实现上通常是通过原子操作来保障线程安全性的，如以下伪代码所示，通过原子操作可以保证只有一个线程进入析构步骤：

```cpp
template<class T>
class shared_ptr {
public:
  ~shared_ptr() {
    // 将引用计数原子地减一，返回原先的值
    if (shared_count_->fetch_sub(1) == 1) {
      // 只会有一个线程进入这个步骤
      delete data_;
      delete shared_count_;     // 这一步会有线程安全问题吗？
    }
  }

private:
  T* data_;
  std::atomic_int* shared_count_;
};
```

上述代码中 `delete shared_count_` 可能引起疑问，如果在线程 A 中执行了这一句，在线程 B 中又试图访问 `shared_count_` 应该会有线程安全问题。

但这种情况只会出现在 “多个线程访问同一个 `shared_ptr` 的场景下”，如果我们遵循正确的使用方式，让线程 A 和 B 持有各自的 `shared_ptr` 拷贝，那么线程 A 和 B 同时进入析构函数的情况下，`shared_count_` 至少是 2，只有在线程 B 已经将其减为 1 的情况下，线程 A 才有机会执行到 `delete shared_count_` 这一句。所以上述代码不存在线程安全问题。

总的来说，向其它线程传递 `std::shared_ptr` 时应该以拷贝的方式传递，这样就不会有线程安全问题。


## 应用

以上内容表明在多个线程中通过不同 `std::shared_ptr` 指向同一个对象是线程安全的。当然这是针对 `std::shared_ptr` 本身而言，对于它所指向的对象还是需要加锁的。

一种有趣的应用是利用 `std::shared_ptr` 的引用计数及自定义析构函数来判断并行的一批任务是否全部完成:

```cpp
void runOnAllThreads(std::function<void()> all_threads_complete_cb) {
  std::shared_ptr<int> cb_guard(new int(42), [all_threads_complete_cb](int* obj) {
    // 自定义析构函数
    // 对象被析构意味着所有线程执行完毕
    all_threads_complete_cb();
    delete obj;
  });

  for (auto& thread : threads) {
    thread.post([cb_guard]() {
      // 每个线程执行后都触发这个回调
      // 全部线程执行完毕后, cb_guard 的引用计数减为 0, 触发析构函数
    });
  }
}
```