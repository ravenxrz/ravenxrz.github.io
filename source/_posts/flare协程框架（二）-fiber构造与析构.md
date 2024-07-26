---
title: flare协程框架（二）- fiber构造与析构
categories: 协程
tags: gdb
date: 2024-07-26 16:24:32
---

## 背景

本文分析 flare中，fiber是如何构造析构以及如何它的两种launch策略。

一些名词解释：

- Fiber： flare对外的有栈协程调度实体
- FiberDesc： flare内部，用于描述一个fiber任务的元数据结构，该结构不包含fiber运行起来时的stack
- FiberEntity：flare内部，用于表示一个真正运行的Fiber的实体，可以理解为linux中的task struct。包含运行时的stack。

<!--more-->

## Fiber构造函数

选择一个ctor来分析，常用如下ctor:

```cpp
// Create a fiber with attributes. It will run from `start`.
Fiber(const Attributes& attr, Function<void()>&& start);
```

源码：

```cpp
Fiber::Fiber(const Attributes& attr, Function<void()>&& start) {
  // Choose a scheduling group for running this fiber.
  auto sg =
      GetSchedulingGroup(attr.scheduling_group_attr, attr.scheduling_group);
  FLARE_CHECK(sg, "No scheduling group is available?");
  if (attr.execution_context) {
    // Caller specified an execution context, so we should wrap `start` to run
    // in the execution context.
    //
    // `ec` holds a reference to `attr.execution_context`, it's released when
    // `start` returns.
    start = [start = std::move(start),
             ec = RefPtr(ref_ptr, attr.execution_context)] {
      ec->Execute(start);
    };
  }
  // `desc` will cease to exist once `start` returns. We don't own it.
  auto desc = fiber::detail::NewFiberDesc();
  desc->worker =
      fiber::detail::GetFiberWorker(attr.scheduling_group, attr.fiber_worker);
  desc->start_proc = std::move(start);
  desc->scheduling_group_local = attr.scheduling_group_local;
  desc->system_fiber = attr.system_fiber;

  // If `join()` is called, we'll sleep on this.
  desc->exit_barrier = object_pool::GetRefCounted<fiber::detail::ExitBarrier>();
  join_impl_ = desc->exit_barrier;

  // Schedule the fiber.
  if (attr.launch_policy == fiber::Launch::Post) {
    sg->StartFiber(desc);
  } else {
    sg->SwitchTo(fiber::detail::GetCurrentFiberEntity(),
                 fiber::detail::InstantiateFiberEntity(sg, desc));
  }
}
```

首先获取一个调度组：

## 调度组选择

```cpp
fiber::detail::SchedulingGroup* GetSchedulingGroup(
    Fiber::SchedulingGroupAttr attr, std::size_t id) {
  switch (attr) {
    case Fiber::SchedulingGroupAttr::ATTR_FOREGROUND: {
      if (FLARE_LIKELY(id == Fiber::kNearestSchedulingGroup)) {
        // specify nearest
        return fiber::detail::NearestSchedulingGroup();
      }

      if (id == Fiber::kUnspecifiedSchedulingGroup) {
        // unspecified, get a random one
        return fiber::detail::GetSchedulingGroup(
            Random<std::size_t>(0, fiber::GetSchedulingGroupCount() - 1));
      }

      // specify other id, less likely to happen
      return fiber::detail::GetSchedulingGroup(id);
    }

    case Fiber::SchedulingGroupAttr::ATTR_BACKGROUND: {
      if (FLARE_LIKELY(id == Fiber::kNearestSchedulingGroup)) {
        // specify nearest
        return fiber::detail::NearestSchedulingGroup(true);
      }

      if (id == Fiber::kUnspecifiedSchedulingGroup) {
        // unspecified, get a random one
        return fiber::detail::GetBgSchedulingGroup(
            Random<std::size_t>(0, fiber::GetBgSchedulingGroupCount() - 1));
      }

      // specify other id, less likely to happen
      return fiber::detail::GetBgSchedulingGroup(id);
    }

    case Fiber::SchedulingGroupAttr::ATTR_UNSPECIFIED: {
      // unspecified, check if the id is set to a special value
      if (id == Fiber::kBackgroundSchedulingGroup) {
        // id is set to be background
        return fiber::detail::GetBgSchedulingGroup(
            Random<std::size_t>(0, fiber::GetBgSchedulingGroupCount() - 1));
      }

      // go with a random one in foreground
      return fiber::detail::GetSchedulingGroup(
          Random<std::size_t>(0, fiber::GetSchedulingGroupCount() - 1));
    }

    // by default, return the nearest one in foreground groups
    default:
      return fiber::detail::NearestSchedulingGroup();
  }
}
```

1. 如何是UNSPECIFIED, 则random one
2. 如果是前台/后台的，看是否使用了 nearest 调度组，如果是则获取 nearest 调度。
3. 使用了 nearest 调度组，则获取 nearest 调度组。

### neartest 调度组

这里只分析nearest调度组：

```cpp
// Get scheduling group "nearest" to the calling thread.
//
// - If calling thread is a fiber worker, it's current scheduling group is
//   returned.
//
// - Otherwise if NUMA aware is enabled, a randomly chosen scheduling group in
//   the same node is returned.
//
// - If no scheduling group is initialized in current node, or NUMA aware is not
//   enabled, a randomly chosen one is returned.
//
// - If no scheduling group is initialize at all, `nullptr` is returned instead.
inline SchedulingGroup* NearestSchedulingGroup(const bool background = false) {
  FLARE_INTERNAL_TLS_MODEL thread_local SchedulingGroup* nearest{};
  if (FLARE_LIKELY(nearest)) {
    return nearest;
  }
  return NearestSchedulingGroupSlow(&nearest, background);
}
```

使用了一个 `thread_local`变量，只会初始化一次，一旦初始化，则直接返回。也就是说，**一个thread归属于一个sg**。否则，找一个neartest， 实际上也是随机:

```cpp
SchedulingGroup* NearestSchedulingGroupSlow(SchedulingGroup** cache,
                                            const bool background) {
  if (auto rc = SchedulingGroup::Current()) {
    // Only if we indeed belong to the scheduling group (in which case the
    // "nearest" scheduling group never changes) we fill the cache.
    *cache = rc;
    return rc;
  }

  // We don't pay for overhead of initialize `next` unless we're not in running
  // fiber worker.
  FLARE_INTERNAL_TLS_MODEL thread_local std::size_t next = Random();

  std::vector<std::unique_ptr<FullyFledgedSchedulingGroup>>* target_groups =
      nullptr;
  if (background) {
    target_groups = &bg_scheduling_groups;
  } else {
    target_groups = &scheduling_groups;
  }

  if (!target_groups->empty()) {
    return target_groups->at(next++ % target_groups->size())
        ->scheduling_group.get();
  }

  return nullptr;
}
```

## FiberDesc

在fiber的ctor中，会构造一个FiberDesc, 用于后续实例化一个 `FiberEntity`,  避免提前分配 fiber stack.

```cpp
// This structure stores information describing how to instantiate a
// `FiberEntity`. The instantiation is deferred to first run of the fiber.
//
// This approach should help performance since:
//
// - Reduced memory footprint: We don't need to allocate a stack until actual
//   run.
//
// - Alleviated producer-consumer effect: The fiber stack is allocated in fiber
//   worker, where most (exited) fiber' stack is freed. This promotes more
//   thread-local-level reuse. If we keep allocating stack from thread X and
//   consume it in thread Y, we'd have a hard time in transfering fiber stack
//   between them (mostly because we can't afford a big transfer-size to avoid
//   excessive memory footprint.).
struct alignas(hardware_destructive_interference_size) FiberDesc
    : RunnableEntity {
  Function<void()> start_proc;
  RefPtr<ExitBarrier> exit_barrier;
  std::uint64_t last_ready_tsc;
  // std::uint32_t fiber_worker;
  bool scheduling_group_local;
  bool system_fiber;

  FiberDesc();
};

```

fiber的ctor中：

```cpp
  // `desc` will cease to exist once `start` returns. We don't own it.
  auto desc = fiber::detail::NewFiberDesc();
  desc->worker =
      fiber::detail::GetFiberWorker(attr.scheduling_group, attr.fiber_worker);   // 是否指定fiber worker来run, 如果没有绑定,则丢到公共队列，由后续的sg中的其他worker来run
  desc->start_proc = std::move(start);
  desc->scheduling_group_local = attr.scheduling_group_local; // 是否支持在不同的sg中运行
  desc->system_fiber = attr.system_fiber;    // system fiber 是fiber框架内部用的fiber
```

## Launch Policy

构造好， FiberDesc 后， 根据不同的Launch policy, 启动fiber:

```cpp
  // Schedule the fiber.
  if (attr.launch_policy == fiber::Launch::Post) {
    sg->StartFiber(desc);
  } else {
    sg->SwitchTo(fiber::detail::GetCurrentFiberEntity(),
                 fiber::detail::InstantiateFiberEntity(sg, desc));
  }
```

先看post的launch policy:

### Post方式

```cpp
void SchedulingGroup::StartFiber(FiberDesc* desc) noexcept {
  desc->last_ready_tsc = ReadTsc();
  // if (desc->fiber_worker != -1U) {
  //   QueueRunnableEntity(desc, desc->scheduling_group_local);
  // }
  if (FLAGS_flare_fiber_schedule_policy_mode == SCHEDULE_BALANCE_POLICY ||
      desc->worker == nullptr) {
    QueueRunnableEntity(desc, desc->scheduling_group_local);
  } else {
    SG_WORKER_PRIVATE_QUEUE_COUNTER_INCR;
    desc->worker->QueueRunnableEntity(desc, true);
  }
}
```

这里两个分支， 第一个分支，是放入sg的公共队列. 只有当 flare_fiber_schedule_policy_mode = SCHEDULE_AFFINITY_POLICY && attr中指定了 fiber worker, 才放入 fiber worker自己的队列。

### Dispatch

从当前fiber切换到一个新实例化的fiber上。

```cpp
    sg->SwitchTo(fiber::detail::GetCurrentFiberEntity(),
                 fiber::detail::InstantiateFiberEntity(sg, desc));
```

先看下如何实例化一个fiber:

```cpp
FiberEntity* InstantiateFiberEntity(SchedulingGroup* scheduling_group,
                                    FiberDesc* desc) noexcept {
  ScopedDeferred _{[&] { DestroyFiberDesc(desc); }};  // Don't leak.
  auto stack = desc->system_fiber ? CreateSystemStack() : CreateUserStack();  // 分配 fiber stack. 这里通常是user stack, TODO(zhangxingrui): 内存分配如何申请待看
  auto stack_size =
      desc->system_fiber ? kSystemStackSize : FLAGS_flare_fiber_stack_size;
  auto bottom = reinterpret_cast<char*>(stack) + stack_size;
  // `FiberEntity` (and magic) is stored at the stack bottom.
  auto ptr = bottom - kFiberStackReservedSize;
  FLARE_DCHECK(reinterpret_cast<std::uintptr_t>(ptr) % alignof(FiberEntity) ==
               0);
  // NOT value-initialized intentionally, to save precious CPU cycles.
  auto fiber = new (ptr) FiberEntity;  // A new life has born.
  fiber->worker = desc->worker;
  fiber->debugging_fiber_id = id_alloc::Next<FiberIdTraits>();
  fiber->stack_size = stack_size - kFiberStackReservedSize;
  fiber->state_save_area =
      make_context(fiber->GetStackTop(), fiber->GetStackLimit(), FiberProc);
  fiber->scheduling_group = scheduling_group;
  fiber->state = FiberState::Ready;

  // Now move fields from `desc` into `fiber`.
  fiber->start_proc = std::move(desc->start_proc);
  fiber->exit_barrier = std::move(desc->exit_barrier);
  fiber->last_ready_tsc = desc->last_ready_tsc;
  fiber->scheduling_group_local = desc->scheduling_group_local;
  fiber->system_fiber = desc->system_fiber;
  return fiber;
}
```

新生成一个fiber entity，使用desc来构造。

```cpp
void SchedulingGroup::SwitchTo(FiberEntity* self, FiberEntity* to) noexcept {
  FLARE_CHECK_EQ(self, GetCurrentFiberEntity());

  // We need scheduler lock here actually (at least to comfort TSan). But so
  // long as this check does not fiber, we're safe without the lock I think.
  FLARE_CHECK(to->state == FiberState::Ready,
              "Fiber `to` is not in ready state.");
  FLARE_CHECK_NE(self, to, "Switch to yourself results in U.B.");

  // TODO(luobogao): Ensure neither `self->scheduler_lock` nor
  // `to->scheduler_lock` is currrently held (by someone else).

  // We delay queuing `self` to run queue until `to` starts to run.
  //
  // It's possible that we first add `self` to run queue with its scheduler lock
  // locked, and unlock the lock when `to` runs. However, if `self` is grabbed
  // by some worker prior `to` starts to run, the worker will spin to wait for
  // `to`. This can be quite costly.
  to->ResumeOn([this, self]() {
    ReadyFiber(self, std::unique_lock(self->scheduler_lock), false);  // 重点关注这行, 切换到目标fiber后，将原来的fiber重新入队
  });

  // When we're back, we should be in the same fiber.
  FLARE_CHECK_EQ(self, GetCurrentFiberEntity());
}


void SchedulingGroup::ReadyFiber(  // 入队
    FiberEntity* fiber, std::unique_lock<Spinlock>&& scheduler_lock, bool normal_prio) noexcept {
  FLARE_DCHECK_NE(fiber, GetMasterFiberEntity(),
                  "Master fiber should not be added to run queue.");
  fiber->state = FiberState::Ready;
  fiber->scheduling_group = this;
  fiber->last_ready_tsc = ReadTsc();
  if (scheduler_lock) {  // TODO(zhangxingrui)： 解锁
    scheduler_lock.unlock();
  }
  if (FLAGS_flare_fiber_schedule_policy_mode == SCHEDULE_BALANCE_POLICY ||
      fiber->worker == nullptr) {
    QueueRunnableEntity(fiber, fiber->scheduling_group_local);   // 重新入队
  } else {
    SG_WORKER_PRIVATE_QUEUE_COUNTER_INCR;
    fiber->worker->QueueRunnableEntity(fiber, normal_prio);
  }
}
```

## ResumeOn （切fiber实际的地方）

```cpp
void FiberEntity::ResumeOn(Function<void()>&& cb) noexcept {
  auto caller = GetCurrentFiberEntity();
  FLARE_CHECK(!resume_proc,
              "You may not call `ResumeOn` on a fiber twice (before the first "
              "one has executed).");
  FLARE_CHECK_NE(caller, this, "Calling `Resume()` on self is undefined.");


  
  // This pending call will be performed and cleared immediately when we
  // switched to `*this` fiber (before calling user's continuation).
  resume_proc = std::move(cb);  
  Resume();
}

inline void FiberEntity::Resume() noexcept {
  // Note that there are some inconsistencies. The stack we're running on is not
  // our stack. This should be easy to see, since we're actually running in
  // caller's context (including its stack).
  auto caller = GetCurrentFiberEntity();
  FLARE_DCHECK_NE(caller, this, "Calling `Resume()` on self is undefined.");

    
  // Argument `context` (i.e., `this`) is only used the first time the context
  // is jumped to (in `FiberProc`).
  jump_context(&caller->state_save_area, state_save_area, this);  // 换fiber
  
	// ...
  SetCurrentFiberEntity(caller);  // The caller has back.

  // Check for pending `ResumeOn`.
  DestructiveRunCallbackOpt(&caller->resume_proc); 
}


```

## jump context

已经有一篇文章专门分析jump context: [这里](https://ravenblog.vercel.app/archives/7dbfb0e8.html)

## FiberProc

jump context 来到的入口函数是: FiberProc

```cpp
// Entry point for newly-started fibers.
//
// NOT put into anonymous namespace to simplify its displayed name in GDB.
//
// `extern "C"` doesn't help, unfortunately. (It does simplify mangled name,
// though.)
//
// Do NOT mark this function as `noexcept`. We don't want to force stack being
// unwound on exception.
static void FiberProc(void* context) {
  auto self = reinterpret_cast<FiberEntity*>(context);
  // We're running in `self`'s stack now.


  SetCurrentFiberEntity(self);  // We're alive.
  self->state = FiberState::Running;

  // Hmmm, there is a pending resumption callback, even if we haven't completely
  // started..
  //
  // We'll run it anyway. This, for now, is mostly used for `Dispatch` fiber
  // launch policy.
  DestructiveRunCallbackOpt(&self->resume_proc);   // 入队原来的fiber
  DestructiveRunCallback(&self->start_proc);  // 开始跑本start

  // We're leaving now.
  FLARE_CHECK_EQ(self, GetCurrentFiberEntity());

  // This fiber should not be waiting on anything (mutex / condition_variable
  // / ...), i.e., no one else should be referring this fiber (referring to its
  // `exit_barrier` is, since it's ref-counted, no problem), otherwise it's a
  // programming mistake.

  // Let's see if there will be someone who will be waiting on us.
  if (!self->exit_barrier) {
    // Mark the fiber as dead. This prevent our GDB plugin from listing this
    // fiber out.
    self->state = FiberState::Dead;

    // No one is waiting for us, this is easy.
    GetMasterFiberEntity()->ResumeOn([self] { FreeFiberEntity(self); });
  } else {
    ...
  }
  FLARE_CHECK(0);  // Can't be here.
}

```

重新入队原fiber， 并开始跑自己的proc:

```cpp
  // Hmmm, there is a pending resumption callback, even if we haven't completely
  // started..
  //
  // We'll run it anyway. This, for now, is mostly used for `Dispatch` fiber
  // launch policy.
  DestructiveRunCallbackOpt(&self->resume_proc);   // 入队原来的fiber
  DestructiveRunCallback(&self->start_proc);  // 开始跑本fiber start
```

完成后，释放本fiber memory:

```cpp
// No one is waiting for us, this is easy.
  GetMasterFiberEntity()->ResumeOn([self] { FreeFiberEntity(self); });  // 切换到MasterFiberEntity 后，释放本fiber
```

TODO(zhangxingrui): MasterFiberEntity 的机制

## 析构FiberEntity，FreeFiberEntity

由于Fiber Entity是放在stack上的，不能直接delete，走它的dtoc，然后主动释放stack。

```cpp
void FreeFiberEntity(FiberEntity* fiber) noexcept {
  bool system_fiber = fiber->system_fiber;

#ifdef FLARE_INTERNAL_USE_TSAN
  flare::internal::tsan::DestroyFiber(fiber->tsan_fiber);
#endif

  fiber->~FiberEntity();

  auto p = reinterpret_cast<char*>(fiber) + kFiberStackReservedSize -
           (system_fiber ? kSystemStackSize : FLAGS_flare_fiber_stack_size);
  if (system_fiber) {
    FreeSystemStack(p);
  } else {
    FreeUserStack(p);
  }
}
```
