---
title: flare协程框架（四）- flare同步和锁语义
categories: 协程
tags: 
date: 2024-07-28 16:24:32
---

对应c++的各种同步和锁语义，flare实现了协程的同步和锁语义。这也是个人比较关心的，比如mutex可能造成thread睡眠，那么flare是如何实现mutex，不让thread睡眠，只让协程睡眠。

<!--more-->

## SpinLock

```cpp
// The class' name explains itself well.
//
// FIXME: Do we need TSan annotations here?
class Spinlock {
 public:
  void lock() noexcept {
    // Here we try to grab the lock first before failing back to TTAS.
    //
    // If the lock is not contend, this fast-path should be quicker.
    // If the lock is contended and we have to fail back to slow TTAS, this
    // single try shouldn't add too much overhead.
    //
    // What's more, by keeping this method small, chances are higher that this
    // method get inlined by the compiler.
    if (FLARE_LIKELY(try_lock())) {
      return;
    }

    // Slow path otherwise.
    LockSlow();
  }

  bool try_lock() noexcept {
    return !locked_.exchange(true, std::memory_order_acquire);
  }

  void unlock() noexcept { locked_.store(false, std::memory_order_release); }

 private:
  void LockSlow() noexcept;

 private:
  std::atomic<bool> locked_{false};
};

void Spinlock::LockSlow() noexcept {
  do {
    // Test ...
    while (locked_.load(std::memory_order_relaxed)) {
      FLARE_CPU_RELAX();
    }

    // ... and set.
  } while (locked_.exchange(true, std::memory_order_acquire));
}
```

这个实现是比较简单的(Spinlock好像都这样实现)，要注意的有两点：

1. lock时，先采用`try_lock`语义作为fast path, 在竞争不大时，这个能节省CPU
2. locked_原子变量在做exchage时，需要使用 memory_order_acquire 内存序。因为lock不仅要保证互斥性，还承担着同步的功能。

该lock可以用在协程也可以用在线程。

## RWSpinLock

```cpp
// copy from folly, but replace thread::yield with FLARE_CPU_RELAX
// https://github.com/facebook/folly/blob/main/folly/synchronization/RWSpinLock.h
/*
 * A simple, small (4-bytes), but unfair rwlock.  Use it when you want
 * a nice writer and don't expect a lot of write/read contention, or
 * when you need small rwlocks since you are creating a large number
 * of them.
 *
 * Note that the unfairness here is extreme: if the lock is
 * continually accessed for read, writers will never get a chance.  If
 * the lock can be that highly contended this class is probably not an
 * ideal choice anyway.
 *
 * It currently implements most of the Lockable, SharedLockable and
 * UpgradeLockable concepts except the TimedLockable related locking/unlocking
 * interfaces.
 */
class RWSpinLock {
  enum : int32_t { READER = 4, UPGRADED = 2, WRITER = 1 };

 public:
  constexpr RWSpinLock() : bits_(0) {}

  RWSpinLock(RWSpinLock const&) = delete;
  RWSpinLock& operator=(RWSpinLock const&) = delete;
  // 实际上 move 相关的函数也被删除了

  // Lockable Concept
  void lock() {
    while (!FLARE_LIKELY(try_lock())) {
      FLARE_CPU_RELAX();
    }
  }

  // Writer is responsible for clearing up both the UPGRADED and WRITER bits.
  void unlock() {
    static_assert(READER > WRITER + UPGRADED, "wrong bits!");
    bits_.fetch_and(~(WRITER | UPGRADED), std::memory_order_release);
  }

  // SharedLockable Concept
  void lock_shared() {
    while (!FLARE_LIKELY(try_lock_shared())) {
      FLARE_CPU_RELAX();
    }
  }

  void unlock_shared() { bits_.fetch_add(-READER, std::memory_order_release); }

  // Downgrade the lock from writer status to reader status.
  void unlock_and_lock_shared() {
    bits_.fetch_add(READER, std::memory_order_acquire);
    unlock();
  }

  // UpgradeLockable Concept
  void lock_upgrade() {
    while (!try_lock_upgrade()) {
      FLARE_CPU_RELAX();
    }
  }

  void unlock_upgrade() {
    bits_.fetch_add(-UPGRADED, std::memory_order_acq_rel);
  }

  // unlock upgrade and try to acquire write lock
  void unlock_upgrade_and_lock() {
    while (!try_unlock_upgrade_and_lock()) {
      FLARE_CPU_RELAX();
    }
  }

  // unlock upgrade and read lock atomically
  void unlock_upgrade_and_lock_shared() {
    bits_.fetch_add(READER - UPGRADED, std::memory_order_acq_rel);
  }

  // write unlock and upgrade lock atomically
  void unlock_and_lock_upgrade() {
    // need to do it in two steps here -- as the UPGRADED bit might be OR-ed at
    // the same time when other threads are trying do try_lock_upgrade().
    bits_.fetch_or(UPGRADED, std::memory_order_acquire);
    bits_.fetch_add(-WRITER, std::memory_order_release);
  }

  // Attempt to acquire writer permission. Return false if we didn't get it.
  bool try_lock() {
    int32_t expect = 0;
    return bits_.compare_exchange_strong(expect, WRITER,
                                         std::memory_order_acq_rel);
  }

  // Try to get reader permission on the lock. This can fail if we
  // find out someone is a writer or upgrader.
  // Setting the UPGRADED bit would allow a writer-to-be to indicate
  // its intention to write and block any new readers while waiting
  // for existing readers to finish and release their read locks. This
  // helps avoid starving writers (promoted from upgraders).
  bool try_lock_shared() {
    // fetch_add is considerably (100%) faster than compare_exchange,
    // so here we are optimizing for the common (lock success) case.
    int32_t value = bits_.fetch_add(READER, std::memory_order_acquire);
    if (FLARE_UNLIKELY(value & (WRITER | UPGRADED))) {
      bits_.fetch_add(-READER, std::memory_order_release);
      return false;
    }
    return true;
  }

  // try to unlock upgrade and write lock atomically
  bool try_unlock_upgrade_and_lock() {
    int32_t expect = UPGRADED;
    return bits_.compare_exchange_strong(expect, WRITER,
                                         std::memory_order_acq_rel);
  }

  // try to acquire an upgradable lock.
  bool try_lock_upgrade() {
    int32_t value = bits_.fetch_or(UPGRADED, std::memory_order_acquire);

    // Note: when failed, we cannot flip the UPGRADED bit back,
    // as in this case there is either another upgrade lock or a write lock.
    // If it's a write lock, the bit will get cleared up when that lock's done
    // with unlock().
    return ((value & (UPGRADED | WRITER)) == 0);
  }

  // mainly for debugging purposes.
  int32_t bits() const { return bits_.load(std::memory_order_acquire); }

 private:
  std::atomic<int32_t> bits_;
};

```

读写自旋锁，实现也比较简单。支持upgrade语义，但可能存在writer被饿死的场景。同样的，该lock可以用在协程也可以用在线程。

## Mutex

```cpp
class Mutex {
 public:
  bool try_lock() {
    FLARE_DCHECK(IsFiberContextPresent());

    std::uint32_t expected = 0;
    return count_.compare_exchange_strong(expected, 1,
                                          std::memory_order_acquire);
  }

  void lock() {
    FLARE_DCHECK(IsFiberContextPresent());

    if (FLARE_LIKELY(try_lock())) {
      return;
    }
    LockSlow();
  }

  void unlock();

 private:
  void LockSlow();

 private:
  Waitable impl_;

  // Synchronizes between slow path of `lock()` and `unlock()`.
  Spinlock slow_path_lock_;

  // Number of waiters (plus the owner). Hopefully `std::uint32_t` is large
  // enough.
  std::atomic<std::uint32_t> count_{0};
};
```

这里只分析 LockSlow和unlock:

### LockSlow

```cpp
void Mutex::LockSlow() {
  FLARE_DCHECK(IsFiberContextPresent());

  if (try_lock()) {
    return;  // Your lucky day.
  }

  // It's locked, take the slow path.
  std::unique_lock splk(slow_path_lock_);

  // Tell the owner that we're waiting for the lock.
  if (count_.fetch_add(1, std::memory_order_acquire) == 0) {
    // The owner released the lock before we incremented `count_`.
    //
    // We're still kind of lucky.
    return;
  }

  // Bad luck then. First we add us to the wait chain.
  auto current = GetCurrentFiberEntity();
  std::unique_lock slk(current->scheduler_lock);
  WaitBlock wb = {.waiter = current};
  FLARE_CHECK(impl_.AddWaiter(&wb));  // This can't fail as we never call
                                      // `SetPersistentAwakened()`.

  // Now the slow path lock can be unlocked.
  //
  // Indeed it's possible that we're awakened even before we call `Halt()`,
  // but this issue is already addressed by `scheduler_lock` (which we're
  // holding).
  splk.unlock();

  // Wait until we're woken by `unlock()`.
  //
  // Given that `scheduler_lock` is held by us, anyone else who concurrently
  // tries to wake us up is blocking on it until `Halt()` has completed.
  // Hence no race here.
  current->scheduling_group->Halt(current, std::move(slk));

  // Lock's owner has awakened us up, the lock is in our hand then.
  FLARE_DCHECK(!impl_.TryRemoveWaiter(&wb));
  return;
}
```

1. 尝试try lock
2. try lock失败之后，尝试另一种”加锁“（fetch_add)，如果刚好碰到在try_lock到fetch_add之间，上一个lock owner释放了lock，则加锁成功。
3. 如果都失败，就**需要切换协程**。

获取当前FiberEntity，加fiber的调度lock，加`slow_path_lock`, 加入到`impl_`的等待队列（注意这里的WaitBlock是在stack上分配的），释放 `slow_path_lock`。注意此时我们已经加了fiber的调度lock，所以即使加入到了等到队列，释放了slow_path_lock后，lock owner释放了lock，并尝试唤醒本fiber也没关系，因为调度lock被加上了，不存在丢失调度问题。

接着进入Halt函数:

```cpp
void SchedulingGroup::Halt(
    FiberEntity* self, std::unique_lock<Spinlock>&& scheduler_lock) noexcept {
  if (!self->need_halt) {  // 这个分支可以忽略，只有在整个sg都stop了，才会进入这个分支。
    self->need_halt = true;
    if (scheduler_lock) {
      scheduler_lock.unlock();
    }
    return;
  }

  FLARE_CHECK_EQ(self, GetCurrentFiberEntity(),
                 "`self` must be pointer to caller's `FiberEntity`.");
  FLARE_CHECK(
      scheduler_lock.owns_lock(),
      "Scheduler lock must be held by caller prior to call to this method.");
  FLARE_CHECK(
      self->state == FiberState::Running,
      "`Halt()` is only for running fiber's use. If you want to `ReadyFiber()` "
      "yourself and `Halt()`, what you really need is `Yield()`.");
  auto master = GetMasterFiberEntity();
  self->state = FiberState::Waiting;  // 从running转到wait阶段

  // We simply yield to master fiber for now.
  //
  // TODO(luobogao): We can directly yield to next ready fiber. This way we can
  // eliminate a context switch.
  //
  // Note that we need to hold `scheduler_lock` until we finished context swap.
  // Otherwise if we're in ready queue, we can be resume again even before we
  // stopped running. This will be disastrous.
  //
  // Do NOT pass `scheduler_lock` ('s pointer)` to cb. Instead, play with raw
  // lock.
  //
  // The reason is that, `std::unique_lock<...>::unlock` itself is not an atomic
  // operation (although `Spinlock` is).
  //
  // What this means is that, after unlocking the scheduler lock, and the fiber
  // starts to run again, `std::unique_lock<...>::owns_lock` does not
  // necessarily be updated in time (before the fiber checks it), which can lead
  // to subtle bugs.
  master->ResumeOn(
      [self_lock = scheduler_lock.release()]() { self_lock->unlock(); });  // 切到master fiber，在切之前，完成本fiber的调度锁解锁

  // When we're back, we should be in the same fiber.
  FLARE_CHECK_EQ(self, GetCurrentFiberEntity());
}
```

这里很明显了，fiber的mutex和thread mutex的不同之处，fiber mutex的lock，如果没拿到lock，切换到master fiber，本fiber被加入都一个调度组中。同时在切换到master fiber时（master fiber运行前），把scheduler lock释放。

### unlock

```cpp
// Implementation of `Mutex` goes below.

void Mutex::unlock() {
  FLARE_DCHECK(IsFiberContextPresent());

  if (auto was = count_.fetch_sub(1, std::memory_order_release); was == 1) {
    // Lucky day, no one is waiting on the mutex.
    //
    // Nothing to do.
  } else {
    FLARE_CHECK_GT(was, 1);

    // We need this lock so as to see a consistent state between `count_` and
    // `impl_` ('s internal wait queue).
    std::unique_lock splk(slow_path_lock_);
    auto fiber = impl_.WakeOne();
    FLARE_CHECK(fiber);  // Otherwise `was` must be 1 (as there's no waiter).
    splk.unlock();
    fiber->scheduling_group->ReadyFiber(fiber, std::unique_lock(fiber->scheduler_lock), true);
  }
}
```

如果当前count_等于1（即为当前要释放lock的fiber），直接返回。

否则，一定有fiber在wait，从wait chain中，取出头部fiber，将其放入到sg调度队列中，等待调度（即唤醒成功）。

## ConditionVariable

条件变量定义如下：

```cpp
// Condition variable for fiber.
class ConditionVariable {
 public:
  template <class LockType>
  void wait(std::unique_lock<LockType> &lock);

  template <class LockType, class F>
  void wait(std::unique_lock<LockType> &lock, F &&pred) {
    FLARE_DCHECK(IsFiberContextPresent());

    while (!std::forward<F>(pred)()) {
      wait(lock);
    }
    FLARE_DCHECK(lock.owns_lock());
  }

  // You can always assume this method returns as a result of `notify_xxx` even
  // if it can actually results from timing out. This is conformant behavior --
  // it's just a spurious wake up in latter case.
  //
  // Returns `false` on timeout.
  template <class LockType>
  bool wait_until(std::unique_lock<LockType> &lock, std::chrono::steady_clock::time_point expires_at);

  template <class LockType, class F>
  bool wait_until(std::unique_lock<LockType> &lk, std::chrono::steady_clock::time_point timeout, F &&pred) {
    FLARE_DCHECK(IsFiberContextPresent());

    while (!std::forward<F>(pred)()) {
      wait_until(lk, timeout);
      if (ReadSteadyClock() >= timeout) {
        return std::forward<F>(pred)();
      }
    }
    FLARE_DCHECK(lk.owns_lock());
    return true;
  }

  void notify_one() noexcept;
  void notify_all() noexcept;

 private:
  Waitable impl_;
};

```

这里我们只关注 `wait` 和 `notify_one`的实现。

### wait

```cpp
template <class LockType>
void ConditionVariable::wait(std::unique_lock<LockType> &lock) {
  FLARE_DCHECK(IsFiberContextPresent());
  FLARE_DCHECK(lock.owns_lock());

  wait_until(lock, std::chrono::steady_clock::time_point::max());
}

```

wait 是通过 `wait_util` 实现，只不过等待时间为无穷大：

```cpp
template <class LockType>
bool ConditionVariable::wait_until(std::unique_lock<LockType> &lock, std::chrono::steady_clock::time_point expires_at) {
  FLARE_DCHECK(IsFiberContextPresent());

  auto current = GetCurrentFiberEntity();
  auto sg = current->scheduling_group;
  bool use_timeout = expires_at != std::chrono::steady_clock::time_point::max();
  DelayedInit<AsyncWaker> awaker;

  // Add us to the wait queue.
  std::unique_lock slk(current->scheduler_lock);
  WaitBlock wb = {.waiter = current};
  FLARE_CHECK(impl_.AddWaiter(&wb));
  if (use_timeout) {  // Set a timeout if needed.
   ...
  }

  // Release user's lock.
  lock.unlock();

  // Block until being waken up by either `notify_xxx` or the timer.
  sg->Halt(current, std::move(slk));  // `slk` is released by `Halt()`.

  ...
  // Grab the lock again and return.
  lock.lock();
  return !timeout;
}
```

将本fiber加入到等待队列，然后切到主fiber上（通过halt），挂起本fiber。

再看wait 带上predict的实现：

```cpp
  template <class LockType, class F>
  void wait(std::unique_lock<LockType> &lock, F &&pred) {
    FLARE_DCHECK(IsFiberContextPresent());

    while (!std::forward<F>(pred)()) {
      wait(lock);
    }
    FLARE_DCHECK(lock.owns_lock());
  }

```

wait外带上一层循环，check predict即可。



### notify_one

```cpp
void ConditionVariable::notify_one() noexcept {
  // FLARE_DCHECK(IsFiberContextPresent());

  auto fiber = impl_.WakeOne();
  if (!fiber) {
    return;
  }
  fiber->scheduling_group->ReadyFiber(fiber, std::unique_lock(fiber->scheduler_lock), true);
}

```

从等到队列中取出一个，加入到调度队列中。

## Semaphore

信号量的视实现如下:

```cpp
// @sa: https://en.cppreference.com/w/cpp/thread/counting_semaphore

// BasicCountingSemaphore is mostly used internally, to avoid code duplication
// between pthread / fiber version semaphore.
template <class Mutex, class ConditionVariable, std::ptrdiff_t kLeastMaxValue>
class BasicCountingSemaphore {
 public:
  static_assert(kLeastMaxValue >= 0);
  explicit BasicCountingSemaphore(std::ptrdiff_t desired) : current_(desired) {}

  static constexpr ptrdiff_t max() noexcept { return kLeastMaxValue; }

  // Acquire / release semaphore, blocking.
  void acquire();
  void release(std::ptrdiff_t count = 1);

  // Non-blocking counterpart of `acquire`. This one fails immediately if the
  // semaphore can't be acquired.
  bool try_acquire() noexcept;

  // `acquire` with timeout.
  template <class Rep, class Period>
  bool try_acquire_for(std::chrono::duration<Rep, Period> expires_in);
  template <class Clock, class Duration>
  bool try_acquire_until(std::chrono::time_point<Clock, Duration> expires_at);

 private:
  Mutex lock_;
  ConditionVariable cv_;
  std::uint32_t current_;
};

```

这里我们只看acquire和release的实现。

### acquire

```cpp
template <class Mutex, class ConditionVariable, std::ptrdiff_t kLeastMaxValue>
void BasicCountingSemaphore<Mutex, ConditionVariable,
                            kLeastMaxValue>::acquire() {
  std::unique_lock lk(lock_);
  cv_.wait(lk, [&] { return current_ != 0; });
  --current_;
  return;
}

```

尝试获取一个资源，如果有，则把资源数(current_) 减一，如果没有资源则wait。

### release

```cpp
template <class Mutex, class ConditionVariable, std::ptrdiff_t kLeastMaxValue>
void BasicCountingSemaphore<Mutex, ConditionVariable, kLeastMaxValue>::release(
    std::ptrdiff_t count) {
  // Count should be less than LeastMaxValue and greater than 0
  FLARE_CHECK_LE(count, kLeastMaxValue);
  FLARE_CHECK_GT(count, 0);
  std::scoped_lock lk(lock_);
  // Internal counter should be less than LeastMaxValue
  FLARE_CHECK_LE(current_, kLeastMaxValue - count);
  current_ += count;
  if (count == 1) {
    cv_.notify_one();
  } else {
    cv_.notify_all();
  }
}
```

资源归还count个，如果归还数为1，则唤醒一个fiber，否则唤醒所有。

> 为什么这里需要唤醒所有，有一部分再被唤醒后有会陷入等待，而不是循环一个个notify?
>
> 在ConditionVar的notify_all中有一段注释:
>
> ```cpp
>   // We cannot keep calling `notify_one` here. If a waiter immediately goes to
>   // sleep again after we wake up it, it's possible that we wake it again when
>   // we try to drain the wait chain.
>   //
>   // So we remove all waiters first, and schedule them then.
> ```
>
> 难道是一个个notify，最先notify的人又睡眠加入到wait队列，然后又被唤醒？但是调度时是fifo的，对于semaphore的release函数，这种case应该不存在。 暂时不知道为什么。
>
> TODO(zhangxingrui)

## 总结

至此，分析完常用的锁和同步实现。代码相对简单，除了spinlock外，其余lock和thread的核心不同在于，fiber lock是将fiber挂起到等到队列，然后切换到其余fiber，而不像thread那样直接将整改thread挂住。对于io密集型场景，性能会更好。
