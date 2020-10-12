---
title: leveldb源码阅读记录-Env类
categories: leveldb
abbrlink: c451920b
date: 2020-10-12 09:30:40
tags:
---

本系列的前几篇文章并不准备深入到leveldb中的”核心“，而是分析一些leveldb中用到的”杂项“内容，如本文要说的Env类，后面要提到的编码、文件命名方式等。

<!--more-->

## Env

Env代表Environment，环境类，在其内部封装了leveldb所有的文件操作和线程调度操作。Env是一个虚基类，具体操作由子类实现，看一下它的继承关系：

<img src="https://pic.downk.cc/item/5f83a9d71cd1bbb86b971527.png" alt="image-20201012085159807" style="zoom: 50%;" />

EnvWrapper类主要是供用户自定义一些操作，一般不管，主要看Windows平台和PosixEnv（linux属于这个范围），因为我都是在linux平台上看的，所以这里只用关注PosixEnv。

### 构造env

默认情况下，Env由`Env::Default()`函数调用创建：

```c++
leveldb/util/env_posix.cc

using PosixDefaultEnv = SingletonEnv<PosixEnv>;
Env* Env::Default() {
  static PosixDefaultEnv env_container;
  return env_container.env();
}

Env* env() { return reinterpret_cast<Env*>(&env_storage_); }
```

Evn.Default调用`env_container.env();`,  env_container是PosixDefaultEnv类型（PosixDefaultEnv是SingletonEnv<PosixEnv>的别名), env返回`env_storage`.

关于env_storage，这个变量在SingletonEnv类中：

```c++
// 用于字节对齐，这里表示env_storage_按照EnvType的大小对齐，同时自身的大小也是env_storage_
typename std::aligned_storage<sizeof(EnvType), alignof(EnvType)>::type
      env_storage_;

// env_storage_在PosixDefaultEnv的构造函数中被初始化：
new (&env_storage_) EnvType();		// 这句话的意思是，按照EvnType的大小申请一个对象，然后将这个对象填充到env_storage_的地址空间中去。
```

最后的默认环境为 **PosixEnv类**, 看一下它的类图：

<img src="https://pic.downk.cc/item/5f633eec160a154a67e8687e.png" alt="image-20200916170716873" style="zoom: 50%;" />

可以看到都是一些辅助函数，包括文件创建，监测，线程管理（同步）等。文件的相关操作比较简单，我们说一下线程池的实现：

### 线程池、队列

线程池的实现中有一个工作队列，队列中放的都是需要被线程执行的任务。工作队列由`background_work_queue_`表示，任务则由`BackgroundWorkItem`表示：

```c++
// 队列
std::queue<BackgroundWorkItem> background_work_queue_
      GUARDED_BY(background_work_mutex_);
      
// work item
// Stores the work item data in a Schedule() call.
//
// Instances are constructed on the thread calling Schedule() and used on the
// background thread.
//
// This structure is thread-safe beacuse it is immutable.
struct BackgroundWorkItem {
    explicit BackgroundWorkItem(void (*function)(void* arg), void* arg)
        : function(function), arg(arg) {}

    void (*const function)(void*);
    void* const arg;
};
```

如何调度一个线程？看Schedule函数，*目前看来线程池中只有一个线程*：

```c++
void PosixEnv::Schedule(
    void (*background_work_function)(void* background_work_arg),
    void* background_work_arg) {
  background_work_mutex_.Lock();

  // Start the background thread, if we haven't done so already.
  if (!started_background_thread_) {	// 首次执行，开启线程
    started_background_thread_ = true;
    std::thread background_thread(PosixEnv::BackgroundThreadEntryPoint, this);
    background_thread.detach();			// detach模式，由内核去回收
  }

  // If the queue is empty, the background thread may be waiting for work.
  if (background_work_queue_.empty()) {
    background_work_cv_.Signal();
  }

  background_work_queue_.emplace(background_work_function, background_work_arg);
  background_work_mutex_.Unlock();
}
```

这里重点看下唤醒消费者的代码，注意这里Signal后，消费者的线程虽然被唤醒，但是依然处于阻塞状态，只有当前线程，调用 

```c++
background_work_mutex_.Unlock();
```

后，消费者线程才能正常执行。

```c++
// If the queue is empty, the background thread may be waiting for work.
if (background_work_queue_.empty()) {
	background_work_cv_.Signal();
}
background_work_queue_.emplace(background_work_function, background_work_arg);	// 插入一个任务
background_work_mutex_.Unlock();
```

所以只有当真正插入一个任务后，线程才正式执行。

如果第一次调度，则首先启动线程, 线程执行的函数为` BackgroundThreadEntryPoint:`

```c++
 static void BackgroundThreadEntryPoint(PosixEnv* env) {
    env->BackgroundThreadMain();
  }
  
  void PosixEnv::BackgroundThreadMain() {
  while (true) {
    background_work_mutex_.Lock();

    // Wait until there is work to be done.
    while (background_work_queue_.empty()) {
      background_work_cv_.Wait();		
    }

      // 取出任务
    assert(!background_work_queue_.empty());
    auto background_work_function = background_work_queue_.front().function;
    void* background_work_arg = background_work_queue_.front().arg;
    background_work_queue_.pop();

      // 执行
    background_work_mutex_.Unlock();
    background_work_function(background_work_arg);
  }
}
```

### 互斥量Mutex和条件变量CondVar

```c++
class LOCKABLE Mutex {
 public:
  Mutex() = default;
  ~Mutex() = default;

  Mutex(const Mutex&) = delete;
  Mutex& operator=(const Mutex&) = delete;

  void Lock() EXCLUSIVE_LOCK_FUNCTION() { mu_.lock(); }
  void Unlock() UNLOCK_FUNCTION() { mu_.unlock(); }
  void AssertHeld() ASSERT_EXCLUSIVE_LOCK() {}

 private:
  friend class CondVar;
  std::mutex mu_;
};
```

Mutex就是对标准mutex的封装，同时用了些宏来修饰，这些宏是clang用于 语法分析（猜想） 的宏，如这里的 ASSERT_EXCLUSIVE_LOCK， 表示调用AssertHeld函数时，表示必须是在获取独占lock的情况下，才能执行之后的代码。比如：

```c++
Status DBImpl::Recover(VersionEdit* edit, bool* save_manifest) {
  mutex_.AssertHeld();
```

DBImpl::Recover函数的第一行代码就是表示该函数必须在本线程获取独占锁的情况下执行。

下面再看看条件变量，CondVar：

条件变量只是对std::condition_variable的浅封装：

```c++
// Thinly wraps std::condition_variable.
class CondVar {
 public:
  explicit CondVar(Mutex* mu) : mu_(mu) { assert(mu != nullptr); }
  ~CondVar() = default;

  CondVar(const CondVar&) = delete;
  CondVar& operator=(const CondVar&) = delete;

  void Wait() {
    std::unique_lock<std::mutex> lock(mu_->mu_, std::adopt_lock);
    cv_.wait(lock);
    lock.release();		// 断开lock和mutex的关联，不释放mutex
  }
  void Signal() { cv_.notify_one(); }
  void SignalAll() { cv_.notify_all(); }

 private:
  std::condition_variable cv_;
  Mutex* const mu_;
};
```

## 总结

本文解释了leveldb中Env类的作用，并分析了内部线程池的实现，下文我们将说一下leveldb内的编码。