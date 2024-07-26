---
title: flare协程框架（三）- fiber调度算法
categories: 协程
tags: 
date: 2024-07-26 16:24:32
---

# 前言

除了协程的上下文切换外，个人对flare协程库最感兴趣的地方就是协程调度算法了。本文简单分析fiber的调度算法。

最好先阅读下官方文档：https://github.com/Tencent/flare/blob/master/flare/doc/fiber-scheduling.md

<!--more-->

# Fiber入队

上一篇文章介绍了 [fiber的构造和析构](https://ravenblog.vercel.app/archives/644e596f.html)， 一个fiber构造后，如果采用Post 方式入队，就会进入到一个运行队列。

```cpp
void SchedulingGroup::StartFiber(FiberDesc* desc) noexcept {
  desc->last_ready_tsc = ReadTsc();
  // if (desc->fiber_worker != -1U) {
  //   QueueRunnableEntity(desc, desc->scheduling_group_local);
  // }
  if (FLAGS_flare_fiber_schedule_policy_mode == SCHEDULE_BALANCE_POLICY ||
      desc->worker == nullptr) {
    QueueRunnableEntity(desc, desc->scheduling_group_local); // here
  } else {
    SG_WORKER_PRIVATE_QUEUE_COUNTER_INCR;
    desc->worker->QueueRunnableEntity(desc, true);
  }
}
```

我们从这里入手，看一个fiber是如何被调度的。

首先要知道一些背景知识，flare中调度是基于scheduling group（调度组）的，后文简称sg。一个sg内含有多个fiber worker（实际上就是线程），一个sg有公共的调度队列，每个fiber worker有自己的私有调度队列。大体逻辑如下图：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240726162051123.png" alt="image-20240726162051123" style="zoom:50%;" />

另外，一个sg最多64个worker，这和sg内调度算法有关。

继续看源码：

```cpp
void SchedulingGroup::QueueRunnableEntity(RunnableEntity* entity,
                                          bool sg_local) noexcept {
  FLARE_DCHECK(!stopped_.load(std::memory_order_relaxed),
               "The scheduling group has been stopped.");

  if (FLARE_UNLIKELY(!run_queue_.Push(entity, sg_local))) {
    auto since = ReadSteadyClock();

    while (!run_queue_.Push(entity, sg_local)) {
      FLARE_LOG_WARNING_EVERY_SECOND(
          "Run queue overflow. Too many ready fibers to run. If you're still "
          "not overloaded, consider increasing `flare_fiber_run_queue_size`.");
      FLARE_LOG_FATAL_IF(ReadSteadyClock() - since > 5s,
                         "Failed to push fiber into ready queue after retrying "
                         "for 5s. Gave up.");
      std::this_thread::sleep_for(100us);
    }
  }

  // run_queue_ is the `global` queue of a scheduling group, we call it `public_q` and we need stats of it
  SG_PUBLIC_QUEUE_COUNTER_INCR;
  if (FLARE_UNLIKELY(!WakeUpOneWorker())) {
    no_worker_available->Increment();
  }
}
```

将entity放入到 run_queue_中， 如果第一次尝试失败，则循环放入，如果超过5s都不行（这个时候系统太忙了，处于不可用），则LOG_FATAL。不过基本不可。所以这里假设入队成功，调用 `WakeUpOneWorker`

```cpp
bool SchedulingGroup::WakeUpOneWorker() noexcept {
  return WakeUpOneSpinningWorker() || WakeUpOneDeepSleepingWorker();
}
```

**上述两个函数这flare的调度核心。**

## 唤醒worker

### WakeUpOneSpinningWorker

先看 `WakeUpOneSpinningWorker`

在flare的系统中，每个sg最多存在2个一直在`spinning`的worker，**不断polling公共队列， 来减少wakeup开销，降低时延。**

> 最多2个是一个均衡，如果全在spining，时延可以做得更低，但是CPU太浪费了。如果都不spining，全都通过wakeup机制，那么wakeup的开销有太大。flare的调度核心就在于只最多保持两个spinning worker，既保证了时延，又保证了没有太多CPU浪费。

```cpp
bool SchedulingGroup::WakeUpOneSpinningWorker() noexcept {
  // FIXME: Is "relaxed" order sufficient here?
  while (auto spinning_mask =
             spinning_workers_.load(std::memory_order_relaxed)) {
    // Last fiber worker (LSB in `spinning_mask`) that is spinning.
    auto last_spinning = __builtin_ffsll(spinning_mask) - 1;
    auto claiming_mask = 1ULL << last_spinning;
    if (FLARE_LIKELY(spinning_workers_.fetch_and(~claiming_mask,
                                                 std::memory_order_relaxed) &
                     claiming_mask)) {
      // We cleared the `last_spinning` bit, no one else will try to dispatch
      // work to it.
      spinning_worker_wakeups->Add(1);
      return true;  // Fast path then.
    }
    Pause();
  }  // Keep trying until no one else is spinning.
  return false;
}
```

`spinning_workers` 代表当前sg中正在`spinning`的worker的情况，bit位上对应为1的，代表该fiber正在运行。所以这里的循环是找到有正在spinning的worker，然后找到LSB为1的worker（通过 `__builtin_ffsll` 函数），构造 `claimming_mask` (该mask用于将`spining_mask`对应worker置0，宣告现在有worker可以处理fiber entity)。

如下函数将 `spinning_workers_` 对应worker置0。

```cpp
FLARE_LIKELY(spinning_workers_.fetch_and(~claiming_mask,
                                                 std::memory_order_relaxed) &
                     claiming_mask)
```

如果置0成功，则`return true` ，否则Pause一下(避免竞争过大），再看是否有正在spinning的worker。

```cpp
// Emit (a series of) pause(s) to relax CPU.
//
// This can be used to delay execution for some time, or backoff from contention
// in case you're doing some "lock-free" algorithm.
template <std::size_t N = 1>
[[gnu::always_inline]] inline void Pause() {
  if constexpr (N != 0) {
    Pause<N - 1>();
#if defined(__x86_64__)
    asm volatile("pause" ::: "memory");  // x86-64 only.
#elif defined(__aarch64__)
    asm volatile("yield" ::: "memory");
#elif defined(__powerpc__)
    // FIXME: **RATHER** slow.
    asm volatile("or 31,31,31   # very low priority" ::: "memory");
#else
#error Unsupported architecture.
#endif
  }
}
```

> 做基础库的人的基本功还是比较好的，我在公司项目中基本就没用过内敛汇编

这里对`spinning_workers_`置0，实际上是”唤醒“了对应那个spinning worker。我们会在后文详细分析。

### WakeUpOneDeepSleepingWorker

如果没有正在spinning的worker，则调用 `WakeUpOneDeepSleepingWorker`

```cpp
bool SchedulingGroup::WakeUpOneDeepSleepingWorker() noexcept {
  // We indeed have to wake someone that is in deep sleep then.
  while (auto sleeping_mask =
             sleeping_workers_.load(std::memory_order_relaxed)) {
    // We always give a preference to workers with a lower index (LSB in
    // `sleeping_mask`).
    //
    // If we're under light load, this way we hopefully can avoid wake workers
    // with higher index at all.
    auto last_sleeping = __builtin_ffsll(sleeping_mask) - 1;
    auto claiming_mask = 1ULL << last_sleeping;
    if (FLARE_LIKELY(sleeping_workers_.fetch_and(~claiming_mask,
                                                 std::memory_order_relaxed) &
                     claiming_mask)) {
      // We claimed the worker. Let's wake it up then.
      //
      // `WaitSlot` class it self guaranteed no wake-up loss. So don't worry
      // about that.
      FLARE_CHECK_LT(last_sleeping, group_size_);
      wait_slots_[last_sleeping].Wake();
      sleeping_worker_wakeups->Add(1);
      return true;
    }
    Pause();
  }
  return false;
}
```

函数框架和`WakeUpOneSpinningWorker`完全一致。 `sleeping_mask` 代表当前正在sleep的worker的情况。找到LSB=1的`sleeping worker`, 把它对应的bit置为0， 然后通过 `wait_slots_` 来唤醒对应的worker。否则 `Pause`。 如果找不到 sleeping 的worker，则反false。

这里的 wait_slots 实现还不错，保证了不会wakeup loss。分析下， 不感兴趣的读者可以直接跳过。

### Wait slots

源码如下：

```cpp
// This class guarantees no wake-up loss by keeping a "wake-up count". If a wake
// operation is made before a wait, the subsequent wait is immediately
// satisfied without actual going to sleep.
class alignas(hardware_destructive_interference_size)
    SchedulingGroup::WaitSlot {
 public:
  void Wake() noexcept {
    ScopedDeferred _([start = ReadTsc()] {
      wakeup_sleeping_worker_latency->Report(TscElapsed(start, ReadTsc()));
    });

    if (wakeup_count_.fetch_add(1, std::memory_order_relaxed) == 0) {
      FLARE_PCHECK(syscall(SYS_futex, &wakeup_count_, FUTEX_WAKE_PRIVATE, 1, 0,
                           0, 0) >= 0);
    }
    // If `Wait()` is called before this check fires, `wakeup_count_` can be 0.
    FLARE_CHECK_GE(wakeup_count_.load(std::memory_order_relaxed), 0);
  }

  void Wait() noexcept {
    if (wakeup_count_.fetch_sub(1, std::memory_order_relaxed) == 1) {
      do {
        // TODO(luobogao): I saw spurious wake up. But how can it happen? If
        // `wakeup_count_` is not zero by the time `futex` checks it, the only
        // values it can become is a positive one, which in this case is a
        // "real" wake up.
        //
        // We need further investigation here.
        auto rc =
            syscall(SYS_futex, &wakeup_count_, FUTEX_WAIT_PRIVATE, 0, 0, 0, 0);
        FLARE_PCHECK(rc == 0 || errno == EAGAIN);
      } while (wakeup_count_.load(std::memory_order_relaxed) == 0);
    }
    FLARE_CHECK_GT(wakeup_count_.load(std::memory_order_relaxed), 0);
  }

  void PersistentWake() noexcept {
    // Hopefully this is large enough.
    wakeup_count_.store(0x4000'0000, std::memory_order_relaxed);
    FLARE_PCHECK(syscall(SYS_futex, &wakeup_count_, FUTEX_WAKE_PRIVATE, INT_MAX,
                         0, 0, 0) >= 0);
  }

 private:
  // `futex` requires this.
  static_assert(sizeof(std::atomic<int>) == sizeof(int));

  std::atomic<int> wakeup_count_{1};
};

```

整个实现依赖 futex（Fast userspace mutex) 系统调用。

futex manual参考：https://man7.org/linux/man-pages/man2/futex.2.html

> futex的wait和wakeup语义，个人感觉是先在用户空间自旋，如果自旋时间内依然没有wakeup，才陷入内核，避免上下文切换的开销。但是在竞争比较大的时候，自旋时间过长，还是会陷入内核。

glibc 不直接暴露futex函数，linux上使用syscall来调用futex：

```c
long syscall(SYS_futex, uint32_t *uaddr, int futex_op, uint32_t val,
                    const struct timespec *timeout,   /* or: uint32_t val2 */
                    uint32_t *uaddr2, uint32_t val3);
```

简单说， futex 在竞争不大时候，比mutex性能更好（glibc的mutex实现底层就是futex）。这里用到了futex的两个操作：

- FUTEX_WAIT:  如果当前 uaddr 的值等于val，就陷入等待。否则唤醒返回。也可能在没wait之前，uaddr地址的值就不等于val， 这时候直接返回EAGAIN。
- FUTEX_WAKE: 最多唤醒`va`个在uaddr地址处等到的waiter。通常这个值是1（唤醒一个），或者INT_MAX(唤醒所有)

futex可以在多个进程之间同步，所以也和共享内存一起使用。在flare里面，只在同一个进程中使用，所以op参数为: `FUTEX_WAKE_PRIVATE` 和 `FUTEX_WAIT_PRIVATE`, 代表uaddr地址参数为一个私有地址。 实际上两个op为一个或操作的结合体：

```c
#define FUTEX_WAIT_PRIVATE	(FUTEX_WAIT | FUTEX_PRIVATE_FLAG)
#define FUTEX_WAKE_PRIVATE	(FUTEX_WAKE | FUTEX_PRIVATE_FLAG)
```

现在再来分析源码：

#### 对齐size

WaitSlot按照 `hardware_destructive_interference_size` 对齐，同时，WaitSlot仅有一个成员变量， `std::atomic<int> wakeup_count_`。

> `hardware_destructive_interference_size` 在c++标准中定义，用于避免 false sharing。 但glibc 目前并没有实现。flare仓库里面将其定义为128B。 
>
> 另一个是 `hardware_constructive_interference_size`，用于促进true sharing。 glibc同样没有实现。flare仓库里面将其定义为 64B.
>
> ```
> // On Sandy Bridge, accessing adjacent cache lines also see destructive
> // interference.
> //
> // @sa: https://github.com/facebook/folly/blob/master/folly/lang/Align.h
> //
> // Update at 20201124: Well, AMD's Zen 3 does the same.
> constexpr std::size_t hardware_destructive_interference_size = 128;
> constexpr std::size_t hardware_constructive_interference_size = 64;
> 
> ```
>
> 源码注释里面也说 destructive_interference 在相邻的cacheline 也有影响，这也是定义为128B的原因。
>
> 再进一步看注释里面各处的链接，folly库里面是这样说的：
>
> ```
> //  Memory locations within the same cache line are subject to destructive
> //  interference, also known as false sharing, which is when concurrent
> //  accesses to these different memory locations from different cores, where at
> //  least one of the concurrent accesses is or involves a store operation,
> //  induce contention and harm performance.
> //
> //  Microbenchmarks indicate that pairs of cache lines also see destructive
> //  interference under heavy use of atomic operations, as observed for atomic
> //  increment on Sandy Bridge.
> //
> //  We assume a cache line size of 64, so we use a cache line pair size of 128
> //  to avoid destructive interference.
> //
> //  mimic: std::hardware_destructive_interference_size, C++17
> ```
>
> 所以这里可以学习到，**对于并发特别高的操作：考虑用128B对齐（两个cacheline）**

再分别来看WaitSlot的三个成员函数:

#### Wait函数

```cpp
  void Wait() noexcept {
    if (wakeup_count_.fetch_sub(1, std::memory_order_relaxed) == 1) {
      do {
        // TODO(luobogao): I saw spurious wake up. But how can it happen? If
        // `wakeup_count_` is not zero by the time `futex` checks it, the only
        // values it can become is a positive one, which in this case is a
        // "real" wake up.
        //
        // We need further investigation here.
        auto rc =
            syscall(SYS_futex, &wakeup_count_, FUTEX_WAIT_PRIVATE, 0, 0, 0, 0);
        FLARE_PCHECK(rc == 0 || errno == EAGAIN);
      } while (wakeup_count_.load(std::memory_order_relaxed) == 0);
    }
    FLARE_CHECK_GT(wakeup_count_.load(std::memory_order_relaxed), 0);
  }

```

`wakeup_count_` 的初始值为1. Wait操作，首先将其原子减一。

> 思考：如果两个线程并发调用同一个wait slot的Wait，wakeup_count_就被减为了一个负数，但这在flare中是不允许的，出现这种情况一定是调用不正常，不应该有两个线程调用同一个wait slot。 因为在Wait函数的最后一行用了 `FLARE_CHECK_GT(wakeup_count_.load(std::memory_order_relaxed), 0);` 强制约束不能为负数。
>
> 另一种情况是如果在未调用wait前，就有人调用了wakeup，可以跳过等待。

主体函数为一个do while. 内部调用 futex 系统调用。

```c
        auto rc =
            syscall(SYS_futex, &wakeup_count_, FUTEX_WAIT_PRIVATE, 0, 0, 0, 0);
```

如果wakeup_count_的值为0， 则陷入等待，否则函数直接返回或唤醒后返回。然后又加了一层 while check `while (wakeup_count_.load(std::memory_order_relaxed) == 0)`。 

>  个人最开始不太理解为什么需要加这一次check，因为理论上说，futex唤醒后，`wakup_count_`就是一个正数了。 从作者的注释中说，他观察到 spurious wakup, 可能加这一个while check是为了避免这种假唤醒吧。

> 同时，我想到在OSTEP书籍中也提到OS可能会虚假唤醒一个睡眠的线程，在使用条件变量时，往往需要加while check，避免假唤醒，基本框架如下：
>
> ```
> while(some condition) {
> 	cv.wait();
> }
> ```

#### Wake函数

```cpp
  void Wake() noexcept {
    ScopedDeferred _([start = ReadTsc()] {
      wakeup_sleeping_worker_latency->Report(TscElapsed(start, ReadTsc()));
    });

    if (wakeup_count_.fetch_add(1, std::memory_order_relaxed) == 0) {
      FLARE_PCHECK(syscall(SYS_futex, &wakeup_count_, FUTEX_WAKE_PRIVATE, 1, 0,
                           0, 0) >= 0);
    }
    // If `Wait()` is called before this check fires, `wakeup_count_` can be 0.
    FLARE_CHECK_GE(wakeup_count_.load(std::memory_order_relaxed), 0);
  }

```

使用一个scope defer来做时延report，这个可以忽略。接着 `wakupe_count_`做了原子加操作，如果加之前是0（说明有一个人在等待），则通过futex唤醒，如果加之前已经是一个正数，说明有人调用了多次Wake，那么之后的wait就不用等待，这里也不会唤醒。

#### PersistentWake 函数

```cpp
  void PersistentWake() noexcept {
    // Hopefully this is large enough.
    wakeup_count_.store(0x4000'0000, std::memory_order_relaxed);
    FLARE_PCHECK(syscall(SYS_futex, &wakeup_count_, FUTEX_WAKE_PRIVATE, INT_MAX,
                         0, 0, 0) >= 0);
  }

```

> 为啥取这个名字，不太理解。

该函数用于做大量唤醒。查了下flare的源码，只有wait slot 所属的schedule group stop时候才会调用，用于唤醒这个sg中的所有的worker。

# 把Fiber Run起来

上文介绍了一个新生成的Fiber是如何入队并”唤醒“一个worker的，那真正干活的worker在哪里”睡眠“着，等待做任务呢？下面我们介绍flare中的worker--FiberWorker。

FiberWorker底层就是一个pthread：

```cpp
void FiberWorker::Start(bool no_cpu_migration) {
  FLARE_CHECK(!no_cpu_migration || !sg_->Affinity().empty());
  worker_ = std::thread([this, no_cpu_migration] {
    if (!sg_->Affinity().empty()) {
      if (no_cpu_migration) {
        FLARE_CHECK_LT(worker_index_, sg_->Affinity().size());
        auto cpu = sg_->Affinity()[worker_index_];

        SetCurrentThreadAffinity({cpu});
        FLARE_VLOG(10,
                   "Fiber worker #{} is started on dedicated processsor #{}.",
                   worker_index_, cpu);
      } else {
        SetCurrentThreadAffinity(sg_->Affinity());
      }
    }
    WorkerProc();
  });
  auto thread_name = "fbth_" + std::to_string(group_idx_) + "_" +
                     std::to_string(worker_index_);
  pthread_setname_np(worker_.native_handle(), thread_name.c_str());
}
```

先不管这里的绑核操作，整个worker的主体函数是 `WorkerProc`

```cpp
void FiberWorker::WorkerProc() {
  sg_->EnterGroup(worker_index_, this); // 绑定sg

  while (true) {
    auto fiber = sg_->AcquireFiber(this); // 从公共队列或者worker的本地队列里面拿一个fiber

    if (!fiber) {  // 没有fiber需要run
      fiber = sg_->SpinningAcquireFiber(this);  // 可以的话，将本thread加入到spinning worker中, 并拿fiber
      if (!fiber) { // 还是没有
        fiber = StealFiber();  // 从其它sg中偷
        FLARE_CHECK_NE(fiber, SchedulingGroup::kSchedulingGroupShuttingDown);
        if (!fiber) { // 还是偷不到
          fiber =
              sg_->WaitForFiber(this);  // This one either sleeps, or succeeds.  // 再尝试一次拿，拿不到就睡眠
          // `FLARE_CHECK_NE` does not handle `nullptr` well.
          FLARE_CHECK_NE(fiber, static_cast<FiberEntity*>(nullptr));
        }
      }
    }

    if (FLARE_UNLIKELY(fiber ==
                       SchedulingGroup::kSchedulingGroupShuttingDown)) {
      break;
    }
		// 走到这里，一定是拿到了fiber来运行
    fiber->iouring = io_uring_engine;
    fiber->Resume();

    ++nr_fiber_done_;

    // Notify the framework that any pending operations can be performed.
    NotifyThreadOutOfDutyCallbacks();
  }
  FLARE_CHECK_EQ(GetCurrentFiberEntity(), GetMasterFiberEntity());
  sg_->LeaveGroup();
}
```

先把本fiber worker thread 和当前的sg绑定。

然后进入一个死循环:这个循环中会多次尝试拿fiber:

1. 从sg公共队列或者fiber worker自己本身的queue中拿，拿不到进入下一步
2. spinning一会，看这个spinning过程中能拿到fiber entity与否（如果当前在spinning的worker没有达到两个，才会把当前的worker做spinning，避免浪费cpu），如果连spinning都拿不到，则下一步
3. 从其他sg中偷，如果偷不到（或这不允许偷），则下一步
4. 再次尝试拿fiber，拿不到就睡眠。 （这里的睡眠，对应在【唤醒Worker】一节中介绍的唤醒方式）
5. 拿到了Fiber，通过Resume切换执行。

我们一步步分析上述四个步骤：

## AcquireFiber

```cpp
FiberEntity* SchedulingGroup::AcquireFiber(FiberWorker* worker) noexcept {
  const uint32_t round_quota = FLAGS_flare_fiber_schedule_policy_round;  // 默认值为8
  auto round = worker->GetAquireRound() % round_quota;
  RunnableEntity* entity = nullptr;
  if ((FLAGS_flare_fiber_schedule_policy_mode == SCHEDULE_BALANCE_POLICY &&
       round < round_quota - 1) ||
      (FLAGS_flare_fiber_schedule_policy_mode == SCHEDULE_AFFINITY_POLICY &&
       round == round_quota - 1)) { // 如果为BALANCE_POLICY，则7/8的概率做sg 公共的queue，如果是AFFINITY_POLICY, 则1/8的功率做sg公共的queue
    entity = run_queue_.Pop();  // 从公共queue获取fiber entity
    if (entity == nullptr) {
      entity = worker->AcquireLocalRunnableEntity(); // 没有就从本地队列
    }
  } else {
    entity = worker->AcquireLocalRunnableEntity();  // 从本地队列获取
    if (entity == nullptr) {
      entity = run_queue_.Pop();
    }
  }
  if (entity) { // 获取到了
    // Acquiring the lock here guarantees us anyone who is working on this fiber
    // (with the lock held) has done its job before we returning it to the
    // caller (worker).
    auto rc = GetOrInstantiateFiber(entity);
    std::scoped_lock _(rc->scheduler_lock);

    FLARE_CHECK(rc->state == FiberState::Ready);

    rc->state = FiberState::Running;  // 标记当前fiber entity 为running

    auto now = ReadTsc();
    ready_to_run_latency->Report(TscElapsed(rc->last_ready_tsc, now));
    ready_to_run_latency_monitoring.Report(
        DurationFromTsc(rc->last_ready_tsc, now));
    return rc;   // 返回
  }
  return stopped_.load(std::memory_order_relaxed) ? kSchedulingGroupShuttingDown
                                                  : nullptr;  // 拿不到就返回nullptr
}
```

根据不同的策略，要么从公共的队列拿fiber，要么从worker自己私有的队列拿fiber。 

> 这里算概率(严格来说这里不是算概率，而是一个定值）的方式也值得借鉴，我们通常用 rand() % 8 == 0 与否来模拟1/8的概率，这里直接用加法操作判定，在指令层面比取余操作更好

## SpinningAcquireFiber

```cpp
FiberEntity* SchedulingGroup::SpinningAcquireFiber(
    FiberWorker* worker) noexcept {
  // We don't want too many workers spinning, it wastes CPU cycles.
  static constexpr auto kMaximumSpinners = 2;

  FLARE_CHECK_NE(worker_index_, kUninitializedWorkerIndex);
  FLARE_CHECK_LT(worker_index_, group_size_);

  FiberEntity* fiber = nullptr;
  auto spinning = spinning_workers_.load(std::memory_order_relaxed);
  auto mask = 1ULL << worker_index_;
  bool need_spin = false;

  // Simply test `spinning` and try to spin may result in too many workers to
  // spin, as it there's a time windows between we test `spinning` and set our
  // bit in it.
  while (CountNonZeros(spinning) < kMaximumSpinners) { // cas操作，看能否把当前的worker index上到这个spinning中
    FLARE_DCHECK_EQ(spinning & mask, 0);
    if (spinning_workers_.compare_exchange_weak(spinning, spinning | mask,
                                                std::memory_order_relaxed)) {
      need_spin = true;
      break;
    }
  }
	
  if (need_spin) { // 如果mark成功，说明当前worker可以进入spinning状态
    static constexpr auto kMaximumCyclesToSpin = 10'000;
    // Wait for some time between touching `run_queue_` to reduce contention.
    static constexpr auto kCyclesBetweenRetry = 1000;
    auto start = ReadTsc(), end = start + kMaximumCyclesToSpin;
    // ReadTsc的实现为：__rdtsc, 返回cpu自启动以来的时钟周期数

    ScopedDeferred _([&] {
      // Note that we can actually clear nothing, the same bit can be cleared by
      // `WakeOneSpinningWorker` simultaneously. This is okay though, as we'll
      // try `AcquireFiber()` when we leave anyway.
      spinning_workers_.fetch_and(~mask, std::memory_order_relaxed);
    });

    do {  // spinning 开始
      if (auto rc = AcquireFiber(worker)) {  // 尝试拿fiber entity
        fiber = rc; 
        break; // 拿到就break
      }
      auto next = start + kCyclesBetweenRetry;
      while (start < next) { // 这个循环最多耗费 kCyclesBetweenRetry 个时钟周期，注意到工作是wakeup在睡眠的worker（如果需要的话），要么就pause，减少锁竞争
        if (pending_spinner_wakeup_.load(std::memory_order_relaxed) &&
            pending_spinner_wakeup_.exchange(false,
                                             std::memory_order_relaxed)) {
          // There's a pending wakeup, and it's us who is chosen to finish this
          // job.
          WakeUpOneDeepSleepingWorker();
        } else {
          Pause<16>();
        }
        start = ReadTsc();
      }
      // 最多过了 kCyclesBetweenRetry 个时钟周期
    } while (start < end && // spinning的最长周期没到，且没有其他人标记本fiber worker可以去拿fiber entity(这个动作在 WakeUpOneSpiningWorker函数中，上文已经分析)
             (spinning_workers_.load(std::memory_order_relaxed) & mask));
  } else {
    // Otherwise there are already at least 2 workers spinning, don't waste CPU
    // cycles then.
    return nullptr;
  }
	
  // 要么已经拿到fiber，要么其他人通知本fiber worker可以去拿fiber entity了
  if (fiber || ((fiber = AcquireFiber(worker)))) {
    // 本fiber worker有事可以做，但系统内少了一个在spining的worker，所以这里标记一下 `pending_spinner_wakeup_`, 让其他在spinning的worker去唤醒另一个worker来做spinning，既可以保证本fiber可以快速去做fiber entity，又可以减少唤醒一个worker的开销。个人感觉是flare fiber调度算法里面值得借鉴的设计
    // Given that we successfully grabbed a fiber to run, we're likely under
    // load. So wake another worker to spin (if there are not enough spinners).
    //
    // We don't want to wake it here, though, as we've had something useful to
    // do (run `fiber)`, so we leave it for other spinners to wake as they have
    // nothing useful to do anyway.
    if (CountNonZeros(spinning_workers_.load(std::memory_order_relaxed)) <
        kMaximumSpinners) {
      pending_spinner_wakeup_.store(true, std::memory_order_relaxed);
    }
  }
  return fiber;
}
```

这里的核心逻辑是：

1. 死转一段时间，如果在这个时间内，有fiber entity可做，则可以直接去做。
2. 同时保证系统内最多有两个在spinning的worker，减少cpu开销的同时，可以保证更低的时延。
3. **如果有fiber可做，本worker就去做了，此时系统内会少一个在spinning的worker，所以需要唤醒另一个worker来spinning。但唤醒工作不在本worker中做，本worker仅打下一个标记，具体唤醒工作交给其他worker去做。（值得借鉴）**

> 值得借鉴的是计算高精度时钟的方式，采用了 `__rdtsc`

## StealFiber

暂不分析，我司项目暂未开启跨sg偷fiber entity。

可以想到，开启steal fiber，系统的尾时延应该会更好。

```cpp
FiberEntity* FiberWorker::StealFiber() {
  if (victims_.empty()) {
    return nullptr;
  }

  ++steal_vec_clock_;
  while (victims_.top().next_steal <= steal_vec_clock_) {
    auto&& top = victims_.top();
    if (auto rc = top.sg->RemoteAcquireFiber()) {
      // We don't pop the top in this case, since it's not empty, maybe the next
      // time we try to steal, there are still something for us.
      return rc;
    }
    victims_.push({.sg = top.sg,
                   .steal_every_n = top.steal_every_n,
                   .next_steal = top.next_steal + top.steal_every_n});
    victims_.pop();
    // Try next victim then.
  }
  return nullptr;
}
```

## WaitForFiber

```cpp
FiberEntity* SchedulingGroup::WaitForFiber(FiberWorker* worker) noexcept {
  FLARE_CHECK_NE(worker_index_, kUninitializedWorkerIndex);
  FLARE_CHECK_LT(worker_index_, group_size_);
  auto mask = 1ULL << worker_index_;

  while (true) {
    ScopedDeferred _([&] {
      // If we're woken up before we even sleep (i.e., we're "woken up" after we
      // set the bit in `sleeping_workers_` but before we actually called
      // `WaitSlot::Wait()`), this effectively clears nothing.
      sleeping_workers_.fetch_and(~mask, std::memory_order_relaxed); // wakeup时，保证sleeping_workers对应的bit是0
    });
    FLARE_CHECK_EQ( // check本worker还没有sleep,并且把当前worker标记为sleep
        sleeping_workers_.fetch_or(mask, std::memory_order_relaxed) & mask, 0);

    // We should test if the queue is indeed empty, otherwise if a new fiber is
    // put into the ready queue concurrently, and whoever makes the fiber ready
    // checked the sleeping mask before we updated it, we'll lose the fiber.
    if (auto f = AcquireFiber(worker)) {  // 这里必须再次获取，不然可能会丢fiber
      // A new fiber is put into ready queue concurrently then.
      //
      // If our sleeping mask has already been cleared (by someone else), we
      // need to wake up yet another sleeping worker (otherwise it's a wakeup
      // miss.).
      //
      // Note that in this case the "deferred" clean up is not needed actually.
      // This is a rare case, though. TODO(luobogao): Optimize it away.
      if ((sleeping_workers_.fetch_and(~mask, std::memory_order_relaxed) &
           mask) == 0) {  // 清空sleep_workers对应的mask bit
        // Someone woken us up before we cleared the flag, wake up a new worker
        // for him.
        WakeUpOneWorker();  // TODO(zhangxingrui): 不理解这里为什么需要唤醒其他人，或许是为了保证系统正在做的worker数量更多，加大并发？
      }
      return f;
    }

    wait_slots_[worker_index_].Wait();  // wait

    // We only return non-`nullptr` here. If we return `nullptr` to the caller,
    // it'll go spinning immediately. Doing that will likely waste CPU cycles.
    if (auto f = AcquireFiber(worker)) { // 有人把我唤醒，尝试看有没有fiber需要做
      return f;
    }  // Otherwise try again (and possibly sleep) until a fiber is ready.
  }
}
```

这里唯一需要注意的点是，为什么在标记sleeping_workers_ 后， 还需要去AcquireFiber一把。

考虑这个时序：

在本worker标记 sleeping_workers_ 对应bit为1之前，有人将FiberEntity入队，并调用了WakeUpOneWorker，此时sleeping_workers_为0，他不会调用任何Wakeup函数，因为他认为一定会有人来做这个entity。 但是回到本fiber worker，如果不造AcuqireFiber检查，直接进入wait流程，此时就被卡住了，那么刚才的fiber entity就被丢了。

## 总结

至此，本文分析了fiber如何入队与调度，最值得借鉴的是系统保证了最多两个spinning worker，既保证了时延，又保证了没有过多CPU开销。

