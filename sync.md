# sync

## 线性一致性模型

从原理上来讲，atomic 操作和非 atomic 操作之间不满足线性一致性模型。这和现代计算机的 CPU 乱序执行，以及 compiler 为优化而进行的指令重排有关。在 C++ 中针对各种场景和性能需求提供了各种 memory order 选项：

1. memory_order_relaxed    Relaxed operation: 只保证当前操作的原子性，不保证其它读写的顺序，也不进行任何多余的同步。就是说 CPU 和编译器可以任意重排其它指令。
2. memory_order_consume    A load operation with this memory order performs a consume operation on the affected memory location: no reads or writes in the current thread dependent on the value currently loaded can be reordered before this load. Writes to data-dependent variables in other threads that release the same atomic variable are visible in the current thread. On most platforms, this affects compiler optimizations only (see Release-Consume ordering below)
3. memory_order_acquire    A load operation with this memory order performs the acquire operation on the affected memory location: no reads or writes in the current thread can be reordered before this load. All writes in other threads that release the same atomic variable are visible in the current thread (see Release-Acquire ordering below)
4. memory_order_release    A store operation with this memory order performs the release operation: no reads or writes in the current thread can be reordered after this store. All writes in the current thread are visible in other threads that acquire the same atomic variable (see Release-Acquire ordering below) and writes that carry a dependency into the atomic variable become visible in other threads that consume the same atomic (see Release-Consume ordering below).
5. memory_order_acq_rel    A read-modify-write operation with this memory order is both an acquire operation and a release operation. No memory reads or writes in the current thread can be reordered before or after this store. All writes in other threads that release the same atomic variable are visible before the modification and the modification is visible in other threads that acquire the same atomic variable.
6. memory_order_seq_cst    A load operation with this memory order performs an acquire operation, a store performs a release operation, and read-modify-write performs both an acquire operation and a release operation, plus a single total order exists in which all threads observe all modifications in the same order (see Sequentially-consistent ordering below)

这里面时序最为严格的是 memory_order_seq_cst。Go 语言的 atomic 类似这个最严格的时序。简单说明一下：

TODO

## atomic

```go
TEXT ·AddUint32(SB),NOSPLIT,$0-20
    MOVQ    addr+0(FP), BP
    MOVL    delta+8(FP), AX
    MOVL    AX, CX
    LOCK
    XADDL   AX, 0(BP)
    ADDL    AX, CX
    MOVL    CX, new+16(FP)
    RET
```

```go
0x0036 00054 (atomic.go:10)    MOVL    $10, CX
0x003b 00059 (atomic.go:10)    LOCK
0x003c 00060 (atomic.go:10)    XADDL    CX, (AX)
```

在 intel 平台上被翻译为:

```shell
mov ecx, 0xa
lock xadd DWORD PTR [rax], ecx
```

lock 指令前缀可以使许多指令操作（ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, and XCHG）变成原子操作。CMPXCHG 指令用来实现 CAS 操作。

atomic.CompareAndSwap 即是使用 lock cmpxchg 来实现的。

在使用 lock 指令时，会导致 CPU 锁总线。

## waitgroup

```go
// 在主 goroutine 中 Add 和 Wait，在其它 goroutine 中 Done
// 在第一次使用之后，不能对 WaitGroup 再进行拷贝
type WaitGroup struct {
    noCopy noCopy

    // state1 的高 32 位是计数器，低 32 位是 waiter 计数
    // 64 位的 atomic 操作需要按 64 位对齐，但是 32 位编译器没法保证这种对齐
    // 所以分配 12 个字节(多分配了 4 个字节)
    // 当 state 没有按 8 对齐时，我们可以偏 4 个字节来使用
    // 按 8 对齐时：
    // 0000...0000      0000...0000       0000...0000
    // |- 4 bytes-|    |- 4 bytes -|     |- 4 bytes -|
    //     使用              使用             不使用
    // 没有按 8 对齐时：
    // |- 4 bytes-|    |- 4 bytes -|     |- 4 bytes -|
    //    不使用              使用             使用
    // |-low->  ---------> ------> -----------> high-|
    state1 [12]byte
    sema   uint32
}

func (wg *WaitGroup) state() *uint64 {
    // 判断 state 是否按照 8 字节对齐
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        // 已对齐时，使用低 8 字节即可
        return (*uint64)(unsafe.Pointer(&wg.state1))
    } else {
        // 未对齐时，使用高 8 字节
        return (*uint64)(unsafe.Pointer(&wg.state1))
        return (*uint64)(unsafe.Pointer(&wg.state1[4]))
    }
}

// Add 一个 delta，delta 可能是负值，在 WaitGroup 的 counter 上增加该值
// 如果 counter 变成 0，所有阻塞在 Wait 函数上的 goroutine 都会被释放
// 如果 counter 变成了负数，Add 会直接 panic
// 当 counter 是 0 且 Add 的 delta 为正的操作必须发生在 Wait 调用之前。
// 而当 counter > 0 且 Add 的 delta 为负的操作则可以发生在任意时刻。
// 一般来讲，Add 操作应该在创建 goroutine 或者其它需要等待的事件发生之前调用
// 如果 wg 被用来等待几组独立的事件集合
// 新的 Add 调用应该在所有 Wait 调用返回之后再调用
// 参见 wg 的 example
func (wg *WaitGroup) Add(delta int) {
    statep := wg.state()

    state := atomic.AddUint64(statep, uint64(delta)<<32)
    v := int32(state >> 32) // counter 高位 4 字节
    w := uint32(state) // waiter counter，截断，取低位 4 个字节

    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    if v > 0 || w == 0 {
        return
    }

    // 当前 goroutine 已经把 counter 设为 0，且 waiter 数 > 0
    // 这时候不能有状态的跳变
    // - Add 不能和 Wait 进行并发调用
    // - Wait 如果发现 counter 已经等于 0，则不应该对 waiter 数加一了
    // 这里是对 wg 误用的简单检测
    if *statep != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }

    // 重置 waiter 计数为 0
    *statep = 0
    for ; w != 0; w-- {
        runtime_Semrelease(&wg.sema, false)
    }
}

// Done 其实就是 wg 的 counter - 1
// 进入 Add 函数后
// 如果 counter 变为 0 会触发 runtime_Semrelease 通知所有阻塞在 Wait 上的 g
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}

// Wait 会阻塞直到 wg 的 counter 变为 0
func (wg *WaitGroup) Wait() {
    statep := wg.state()

    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32) // counter
        w := uint32(state) // waiter count
        if v == 0 { // counter
            return
        }

        // 如果没成功，可能有并发，循环再来一次相同流程
        // 成功直接返回
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            runtime_Semacquire(&wg.sema) // 和上面的 Add 里的 runtime_Semrelease 是对应的
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            return
        }
    }
}
```

## once

```go
// 内含一个锁和用来做原子操作的变量
type Once struct {
    m    Mutex
    done uint32
}

// Do 被用来执行那些只能执行一次的初始化操作
//     config.once.Do(func() { config.init(filename) })
//
// 因为对 Do 的调用直到其中的一个 f 执行之后才会返回，所以
// f 中不能调用同一个 once 实例的 Do 函数，否则会死锁
// 如果 f 内 panic 了，Do 也认为已经返回了，未来对 Do 的调用不会再执行 f

// once.Do(f) 被调用多次时，只有第一次调用会真正的执行 f
// 对于每一个要执行的 f，都需要一个对应的 once 实例
// 在 done 已经被改成 1 之后
// 所有进入函数调用的行为会用 atomic 读取值之后直接返回
func (o *Once) Do(f func()) {
    // 轻量级的原子变量 load
    if atomic.LoadUint32(&o.done) == 1 {
        // 如果原子 load 后发现已经是 1 了，直接返回
        return
    }

    // Slow-path.
    o.m.Lock()
    defer o.m.Unlock()
    // 在 atomic load 的时候为 0，不代表进入 lock 之后也是 0
    // 所以还需要再判断一次
    // 临界区内的判断和修改是比较稳妥的
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

once.Do 实际上是一种优化，只要过程被执行过了，那么之后所有判断都走 atomic，不用进入临界区。

## futex

```go
// This implementation depends on OS-specific implementations of
//
//    futexsleep(addr *uint32, val uint32, ns int64)
//        Atomically,
//            if *addr == val { sleep }
//        Might be woken up spuriously; that's allowed.
//        Don't sleep longer than ns; ns < 0 means forever.
//
//    futexwakeup(addr *uint32, cnt uint32)
//        If any procs are sleeping on addr, wake up at most cnt.

const (
    mutex_unlocked = 0
    mutex_locked   = 1
    mutex_sleeping = 2

    active_spin     = 4
    active_spin_cnt = 30
    passive_spin    = 1
)

// Possible lock states are mutex_unlocked, mutex_locked and mutex_sleeping.
// mutex_sleeping means that there is presumably at least one sleeping thread.
// Note that there can be spinning threads during all states - they do not
// affect mutex's state.

// We use the uintptr mutex.key and note.key as a uint32.
//go:nosplit
func key32(p *uintptr) *uint32 {
    return (*uint32)(unsafe.Pointer(p))
}

func lock(l *mutex) {
    gp := getg()

    if gp.m.locks < 0 {
        throw("runtime·lock: lock count")
    }
    gp.m.locks++

    // Speculative grab for lock.
    v := atomic.Xchg(key32(&l.key), mutex_locked)
    if v == mutex_unlocked {
        return
    }

    // wait is either MUTEX_LOCKED or MUTEX_SLEEPING
    // depending on whether there is a thread sleeping
    // on this mutex. If we ever change l->key from
    // MUTEX_SLEEPING to some other value, we must be
    // careful to change it back to MUTEX_SLEEPING before
    // returning, to ensure that the sleeping thread gets
    // its wakeup call.
    wait := v

    // On uniprocessors, no point spinning.
    // On multiprocessors, spin for ACTIVE_SPIN attempts.
    spin := 0
    if ncpu > 1 {
        spin = active_spin
    }
    for {
        // Try for lock, spinning.
        for i := 0; i < spin; i++ {
            for l.key == mutex_unlocked {
                if atomic.Cas(key32(&l.key), mutex_unlocked, wait) {
                    return
                }
            }
            procyield(active_spin_cnt)
        }

        // Try for lock, rescheduling.
        for i := 0; i < passive_spin; i++ {
            for l.key == mutex_unlocked {
                if atomic.Cas(key32(&l.key), mutex_unlocked, wait) {
                    return
                }
            }
            osyield()
        }

        // Sleep.
        v = atomic.Xchg(key32(&l.key), mutex_sleeping)
        if v == mutex_unlocked {
            return
        }
        wait = mutex_sleeping
        futexsleep(key32(&l.key), mutex_sleeping, -1)
    }
}

func unlock(l *mutex) {
    v := atomic.Xchg(key32(&l.key), mutex_unlocked)
    if v == mutex_unlocked {
        throw("unlock of unlocked lock")
    }
    if v == mutex_sleeping {
        futexwakeup(key32(&l.key), 1)
    }

    gp := getg()
    gp.m.locks--
    if gp.m.locks < 0 {
        throw("runtime·unlock: lock count")
    }
    if gp.m.locks == 0 && gp.preempt { // restore the preemption request in case we've cleared it in newstack
        gp.stackguard0 = stackPreempt
    }
}

// One-time notifications.
func noteclear(n *note) {
    n.key = 0
}

func notewakeup(n *note) {
    old := atomic.Xchg(key32(&n.key), 1)
    if old != 0 {
        print("notewakeup - double wakeup (", old, ")\n")
        throw("notewakeup - double wakeup")
    }
    futexwakeup(key32(&n.key), 1)
}

func notesleep(n *note) {
    gp := getg()
    if gp != gp.m.g0 {
        throw("notesleep not on g0")
    }
    ns := int64(-1)
    if *cgo_yield != nil {
        // Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
        ns = 10e6
    }
    for atomic.Load(key32(&n.key)) == 0 {
        gp.m.blocked = true
        futexsleep(key32(&n.key), 0, ns)
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }
        gp.m.blocked = false
    }
}

// May run with m.p==nil if called from notetsleep, so write barriers
// are not allowed.
//
//go:nosplit
//go:nowritebarrier
func notetsleep_internal(n *note, ns int64) bool {
    gp := getg()

    if ns < 0 {
        if *cgo_yield != nil {
            // Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
            ns = 10e6
        }
        for atomic.Load(key32(&n.key)) == 0 {
            gp.m.blocked = true
            futexsleep(key32(&n.key), 0, ns)
            if *cgo_yield != nil {
                asmcgocall(*cgo_yield, nil)
            }
            gp.m.blocked = false
        }
        return true
    }

    if atomic.Load(key32(&n.key)) != 0 {
        return true
    }

    deadline := nanotime() + ns
    for {
        if *cgo_yield != nil && ns > 10e6 {
            ns = 10e6
        }
        gp.m.blocked = true
        futexsleep(key32(&n.key), 0, ns)
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }
        gp.m.blocked = false
        if atomic.Load(key32(&n.key)) != 0 {
            break
        }
        now := nanotime()
        if now >= deadline {
            break
        }
        ns = deadline - now
    }
    return atomic.Load(key32(&n.key)) != 0
}

func notetsleep(n *note, ns int64) bool {
    gp := getg()
    if gp != gp.m.g0 && gp.m.preemptoff != "" {
        throw("notetsleep not on g0")
    }

    return notetsleep_internal(n, ns)
}

// same as runtime·notetsleep, but called on user g (not g0)
// calls only nosplit functions between entersyscallblock/exitsyscall
func notetsleepg(n *note, ns int64) bool {
    gp := getg()
    if gp == gp.m.g0 {
        throw("notetsleepg on g0")
    }

    entersyscallblock()
    ok := notetsleep_internal(n, ns)
    exitsyscall()
    return ok
}

func pauseSchedulerUntilCallback() bool {
    return false
}

func checkTimeouts() {}
```

## sync.RWMutex


## sync.Map

# 参考资料

http://www.weixianmanbu.com/article/736.html

https://www.cnblogs.com/gaochundong/p/lock_free_programming.html
