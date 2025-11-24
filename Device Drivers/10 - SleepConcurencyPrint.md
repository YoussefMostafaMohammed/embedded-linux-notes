# **Linux Kernel Module Development Guide: Concurrency, Synchronization & Debugging**

## **Table of Contents**

1. [Introduction](#introduction)
2. [Process & Thread Management](#process-thread-management)
   - 2.1 [Blocking with Wait Queues](#blocking-wait-queues)
   - 2.2 [Completions: Ordered Thread Synchronization](#completions)
3. [Synchronization Mechanisms](#synchronization)
   - 3.1 [Mutexes: Sleepable Locking](#mutexes)
   - 3.2 [Spinlocks: Atomic Context Locking](#spinlocks)
   - 3.3 [Read-Write Locks: Specialized Spinlocks](#rwlocks)
   - 3.4 [Atomic Operations: Lockless Primitives](#atomic-ops)
4. [Debugging & Communication Techniques](#debugging)
   - 4.1 [Direct TTY Output](#tty-output)
   - 4.2 [Physical LED Signaling](#led-signaling)
5. [Best Practices & Critical Pitfalls](#best-practices)
6. [Kernel Version Compatibility Matrix](#version-matrix)
7. [Performance Considerations](#performance)
8. [Troubleshooting Guide](#troubleshooting)
9. [Additional Resources & Further Reading](#resources)

---

## **1. Introduction** <a name="introduction"></a>

This document provides a comprehensive guide to managing concurrency, synchronization, and debugging in Linux kernel modules. Unlike user-space programming, kernel code runs in a privileged environment where errors can crash the entire system. Understanding these mechanisms is crucial for:

- **Preventing race conditions** when multiple processes/threads access shared resources
- **Managing blocking operations** without hanging the system
- **Coordinating kernel threads** with precise ordering guarantees
- **Implementing non-intrusive debugging** when traditional logging fails

The examples and concepts herein are derived from real-world kernel module patterns and reflect best practices across kernel versions 4.x through 6.x.

---

## **2. Process & Thread Management** <a name="process-thread-management"></a>

### **2.1 Blocking with Wait Queues** <a name="blocking-wait-queues"></a>

#### **Core Concept: Process States in the Kernel**

When a process cannot proceed (e.g., waiting for hardware or exclusive resource access), it must be put to sleep rather than polling. The kernel maintains a `task_struct` for each process, which includes a `state` field:

- **TASK_RUNNING**: Process is ready to run or running
- **TASK_INTERRUPTIBLE**: Process is sleeping but can be woken by signals or events
- **TASK_UNINTERRUPTIBLE**: Process is sleeping and ignores signals (use sparingly)
- **TASK_KILLABLE**: A compromise between interruptible and uninterruptible

#### **wait_event_interruptible() Deep Dive**

```c
wait_event_interruptible(waitq, condition);
```

This macro performs several critical operations atomically:

1. **Adds current task to wait queue**: Links the task into `waitq` linked list
2. **Sets task state to TASK_INTERRUPTIBLE**: Marks the process as sleepable
3. **Calls schedule()**: Yields CPU, triggering context switch
4. **Checks condition**: After wakeup, verifies if the condition is truly met
5. **Removes from queue**: Cleans up the wait queue entry

**Why INTERRUPTIBLE?** Using `wait_event()` (uninterruptible) makes processes immune to signals, causing unkillable processes that frustrate users. Always prefer interruptible variants unless absolutely necessary.

#### **Wakeup Mechanisms**

- **wake_up(&waitq)**: Wakes ALL processes waiting on the queue (broadcast)
- **wake_up_interruptible(&waitq)**: Wakes only interruptible sleepers
- **wake_up_process(task)**: Wakes a specific task

**Key Insight**: Multiple wakeups are harmless—processes recheck conditions after waking.

#### **O_NONBLOCK: Non-Blocking Semantics**

User-space can open files with `O_NONBLOCK` to demand immediate success/failure:

```c
if (file->f_flags & O_NONBLOCK)
    return -EAGAIN; /* Try again later */
```

This is crucial for event-driven applications (e.g., web servers, GUI apps) that cannot afford to block on file I/O.

#### **Resource Management Pattern**

The `/proc/sleep` example implements **exclusive access** with automatic cleanup:

```c
static atomic_t already_open = ATOMIC_INIT(0);

/* Open */
if (atomic_cmpxchg(&already_open, 0, 1)) {
    /* Already open - block or return -EAGAIN */
}

/* Close */
atomic_set(&already_open, 0);
wake_up(&waitq);
```

**Atomic Operations**: `atomic_cmpxchg()` provides lockless test-and-set, critical for avoiding races between multiple CPUs.

---

### **2.2 Completions: Ordered Thread Synchronization** <a name="completions"></a>

#### **Philosophy Behind Completions**

Completions provide a **higher-level abstraction** than raw wait queues for **one-time events**. Think of them as kernel-space "thread join" operations or "barriers" that ensure happen-before relationships.

#### **Three-Phase Pattern**

1. **Initialization**: `init_completion(&comp)` sets done count to 0
2. **Waiting**: `wait_for_completion(&comp)` blocks until count > 0
3. **Signaling**: `complete(&comp)` increments count and wakes one waiter

```c
struct completion crank_comp;
init_completion(&crank_comp);

/* Waiter thread */
wait_for_completion(&crank_comp); /* Blocks */

/* Signaler thread */
complete_all(&crank_comp); /* Wakes all waiters */
```

#### **complete() vs complete_all()**

- **complete()**: Wakes ONE waiter, increments count by 1 (useful for resource counting)
- **complete_all()**: Sets count to UINT_MAX, waking ALL waiters (useful for shutdown signals)

#### **Thread Termination: kthread_complete_and_exit()**

Starting from kernel 5.17, `kthread_complete_and_exit()` ensures proper cleanup:
- Signals completion BEFORE exiting
- Prevents race where waiter might see "dead" thread ID
- Replaces deprecated `complete_and_exit()`

#### **Practical Use Cases**

- **Module shutdown**: Ensure all worker threads finish before unloading
- **Probe sequencing**: Hardware initialization must complete before functionality is exposed
- **Error propagation**: Signal failure conditions across threads

---

## **3. Synchronization Mechanisms** <a name="synchronization"></a>

### **3.1 Mutexes: Sleepable Locking** <a name="mutexes"></a>

#### **Ownership Semantics**

Linux kernel mutexes enforce **strict ownership**:
- Only the **lock holder** can unlock
- Unlocking an unheld mutex = BUG()
- Recursive locking (same task locks twice) = deadlock or error

```c
DEFINE_MUTEX(mymutex);

mutex_lock(&mymutex);  /* Sleep if needed */
/* Critical section */
mutex_unlock(&mymutex);
```

#### **Variants & Their Semantics**

| Function | Behavior | Use Case |
|----------|----------|----------|
| `mutex_lock()` | Uninterruptible sleep | Rare, for critical sections that must not be interrupted |
| `mutex_lock_interruptible()` | Sleep with signal handling | **Default choice** for user-triggered operations |
| `mutex_lock_killable()` | Sleep, but only fatal signals | Balance between safety and responsiveness |
| `mutex_trylock()` | Non-blocking attempt | Interrupt handlers, O_NONBLOCK paths |

#### **Interrupt Context Prohibition**

**NEVER** use mutexes in interrupt context (IRQ handlers, softirqs, tasklets) because:
- `mutex_lock()` can sleep
- Sleeping with interrupts disabled deadlocks the system
- IRQ handlers have no process context to schedule out

#### **Nested Locking & Lockdep**

`mutex_lock_nested()` helps lockdep (kernel lock validator) understand intentional nesting hierarchies:

```c
/* Subclass 0 is default, 1+ indicate nesting levels */
mutex_lock_nested(&lock1, 0);
mutex_lock_nested(&lock2, 1); /* Tell lockdep this is nested */
```

Lockdep tracks lock dependencies at runtime and reports potential deadlocks during development.

---

### **3.2 Spinlocks: Atomic Context Locking** <a name="spinlocks"></a>

#### **Fundamental Principle**

Spinlocks **busy-wait** on a locked CPU, preventing any other work (including scheduler) on that core. Suitable only for **critical sections < few milliseconds**.

#### **The "Sleep-in-Atomic-Context" Bug**

**Atomic context** = code that cannot schedule:
- Holding a spinlock
- In interrupt handler (IRQ/softirq)
- RCU read-side critical sections
- Preemption disabled (`preempt_disable()`)

**Consequences of sleeping in atomic context**:
- System hang (CPU permanently spinning)
- RCU stalls
- Kernel panic: "scheduling while atomic"
- Data corruption from re-entrant access

**Common sleepers to avoid**:
- `kmalloc(GFP_KERNEL)` → Use `GFP_ATOMIC`
- `mutex_lock()` → Use `spin_lock()`
- `msleep()` → Use `udelay()` (but only VERY briefly)
- `copy_to_user()` → May sleep if page fault occurs

#### **IRQ-Safe Variants Explained**

```c
unsigned long flags;
spin_lock_irqsave(&lock, flags);  /* Save and disable IRQs */
/* Critical section - safe from all interrupts */
spin_unlock_irqrestore(&lock, flags); /* Restore IRQ state */
```

**Why save flags?** On same CPU, disabling IRQs twice is safe only if you restore the original state. `spin_lock_irq()` is dangerous because it discards previous IRQ state.

#### **Spinlock Type Matrix**

| Function Type | Disables SoftIRQs | Disables HardIRQs | Saves State | Use Case |
|---------------|-------------------|-------------------|-------------|----------|
| `spin_lock()` | No | No | No | Process context only, no IRQ conflict |
| `spin_lock_irq()` | Yes | Yes | No | Known IRQ state, short sections |
| `spin_lock_irqsave()` | Yes | Yes | Yes | **Default choice**, safe everywhere |
| `spin_lock_bh()` | Yes | No | No | Protect from softirqs, keep hardware IRQs |

#### **Realtime Linux Considerations**

PREEMPT_RT converts spinlocks to sleeping mutexes, breaking atomicity assumptions. Use `raw_spinlock_t` for true spinning in RT kernels.

---

### **3.3 Read-Write Locks: Specialized Spinlocks** <a name="rwlocks"></a>

#### **Concurrency Model**

- **Multiple readers**: Can hold lock simultaneously (shared access)
- **Single writer**: Exclusive access, no readers or other writers
- **Writer starvation**: Readers can block writers indefinitely (avoid long read sections)

#### **IRQ-Safe Usage**

```c
DEFINE_RWLOCK(myrwlock);

/* Read side - non-destructive */
read_lock_irqsave(&myrwlock, flags);
/* Read data structures */
read_unlock_irqrestore(&myrwlock, flags);

/* Write side - destructive */
write_lock_irqsave(&myrwlock, flags);
/* Modify data structures */
write_unlock_irqrestore(&myrwlock, flags);
```

#### **When to Use RWLocks vs Regular Spinlocks**

Use RWLocks when:
- Read operations **far outnumber** writes (>10:1 ratio)
- Read critical sections are **read-only** (no modifications)
- Data structure is **large** and frequently accessed

Otherwise, prefer regular spinlocks—simpler logic and avoids writer starvation.

---

### **3.4 Atomic Operations: Lockless Primitives** <a name="atomic-ops"></a>

#### **Memory Ordering & CPU Caches**

Atomic operations use **LOCK prefixes** on x86 or **LL/SC** (Load-Linked/Store-Conditional) on ARM to ensure:
- **Cache coherency**: All CPUs see consistent view
- **Memory barriers**: Prevent compiler/CPU reordering
- **Atomicity**: Operation appears indivisible

#### **Operation Categories**

**1. Arithmetic (on `atomic_t`)**:
```c
atomic_t counter = ATOMIC_INIT(0);
atomic_inc(&counter);         /* ++counter */
atomic_add(5, &counter);      /* counter += 5 */
atomic_sub_and_test(-1, &counter); /* Used for reference counting */
```

**2. Bitwise (on `unsigned long`)**:
```c
unsigned long flags = 0;
set_bit(3, &flags);           /* flags |= (1 << 3) */
clear_bit(5, &flags);         /* flags &= ~(1 << 5) */
change_bit(7, &flags);        /* flags ^= (1 << 7) */
test_and_set_bit(0, &flags);  /* Atomic test-and-set */
```

#### **vs C11 Atomics**

Kernel atomics predate C11 `_Atomic` and have:
- **Explicit memory ordering**: `atomic_read()`/`atomic_set()` are relaxed
- **Architecture-specific tuning**: e.g., `atomic_add_unless()`
- **Integration with kernel debug features**: KASAN, lockdep

**Migration status**: Some subsystems are adopting C11 atomics, but kernel-wide migration is slow due to different memory models.

---

## **4. Debugging & Communication Techniques** <a name="debugging"></a>

### **4.1 Direct TTY Output** <a name="tty-output"></a>

#### **Accessing the Current TTY**

The `current` macro points to the `task_struct` of the running process. Not all tasks have TTYs (daemons, kthreads).

```c
struct tty_struct *my_tty = get_current_tty();
if (my_tty) {
    const struct tty_operations *ops = my_tty->driver->ops;
    ops->write(my_tty, str, strlen(str));
}
```

#### **TTY Operations & Drivers**

TTY drivers implement `struct tty_operations` with `.write()`, `.ioctl()`, etc. The console driver (`/dev/console`) is special—it always exists.

#### **ASCII vs. Terminal Behavior**

Legacy terminals require CR-LF (`\015\012`) for newline. Modern terminals accept `\n`, but raw hardware drivers may not.

---

### **4.2 Physical LED Signaling** <a name="led-signaling"></a>

#### **KDSETLED IOCTL**

```c
#include <linux/kd.h>
ioctl(tty, KDSETLED, value);
```

- **LED_SHOW_IOCTL** (value < 8): LEDs show programmed pattern
- **LED_SHOW_FLAGS** (value > 7): LEDs reflect actual keyboard state (Caps/Num/Scroll)

#### **Timer API Evolution (v4.14 → v4.15+)** <a name="timer-api"></a>

**Pre-4.14**:
```c
setup_timer(&timer, callback, data); /* data passed as unsigned long */
void callback(unsigned long data);
```

**Post-4.15**:
```c
timer_setup(&timer, callback, 0); /* callback receives timer_list* */
void callback(struct timer_list *t);

/* Extract custom data using container_of */
struct my_struct *obj = container_of(t, struct my_struct, timer);
```

**Motivation**: Security (prevents ROP attacks), type safety, and CFI (Control Flow Integrity) compliance.

#### **Jiffies & HZ**

- `HZ` = timer interrupts per second (typically 1000)
- `jiffies` = global tick counter since boot
- `add_timer()` schedules execution at absolute `jiffies` value

---

## **5. Best Practices & Critical Pitfalls** <a name="best-practices"></a>

### **Universal Rules**

1. **Never sleep in atomic context**: This is the #1 cause of kernel hangs
2. **Prefer interruptible sleeps**: Always use `_interruptible` variants unless impossible
3. **Keep critical sections short**: < 1ms for spinlocks, < 100ms for mutexes
4. **Document lock ordering**: Prevent deadlocks with consistent acquisition order
5. **Use lockdep during development**: Enable `CONFIG_LOCKDEP` to catch lock violations
6. **Avoid global variables**: Use per-CPU data or thread-local storage where possible

### **Common Anti-Patterns**

| Anti-Pattern | Consequence | Fix |
|--------------|-------------|-----|
| Double unlock | Kernel panic, undefined behavior | Use RAII pattern or check holding status |
| Locking in IRQ handler with non-IRQ-safe lock | Deadlock with process context | Use `spin_lock_irqsave()` |
| Calling `kmalloc(GFP_KERNEL)` with spinlock held | Sleep-in-atomic-context bug | Use `GFP_ATOMIC` or allocate before locking |
| Forgetting `wake_up()` | Process hangs forever | Use completions to auto-wake |
| Unlocking mutex from wrong thread | Logic error, potential data corruption | Enforce ownership in API design |

### **Debugging Sleep-in-Atomic-Context**

Enable these configs:
```bash
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_STACKTRACE=y
```

Kernel will dump stack trace like:
```
BUG: sleeping function called from invalid context
in_atomic(): 1, irqs_disabled(): 1
Call Trace:
 [<ffffffff>] my_function+0x123/0x456
```

---

## **6. Kernel Version Compatibility Matrix** <a name="version-matrix"></a>

| Feature | < 4.14 | 4.14 - 4.15 | >= 4.15 | >= 5.17 |
|---------|--------|-------------|---------|---------|
| **Completions Exit** | `complete_and_exit()` | Transition | Transition | `kthread_complete_and_exit()` |
| **Timer API** | `setup_timer()` | `timer_setup()` wrapper | `timer_setup()` native | Same |
| **ProcFS Ops** | `file_operations` | Transition | `proc_ops` | Same |
| **kthread_create** | Available | Available | Available | Same |
| **Spinlock RT** | Always spins | PartialPREEMPT_RT | PREEMPT_RT spins become mutex | Same |

### **Version Detection Macros**

```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 17, 0)
    kthread_complete_and_exit(&comp, 0);
#else
    complete_and_exit(&comp, 0);
#endif
```

---

## **7. Performance Considerations** <a name="performance"></a>

### **Latency Comparison**

| Mechanism | Approx. Latency | CPU Usage | Can Sleep | Use Case |
|-----------|-----------------|-----------|-----------|----------|
| **Atomic ops** | ~10-50ns | 0% | No | Counters, flags |
| **Spinlock** | ~100ns-1μs (contended) | 100% on 1 CPU | No | Very short sections |
| **Mutex** | ~1-100μs (contended) | 0% while waiting | Yes | Long operations |
| **Wait Queue** | ~5-50μs wakeup | 0% while waiting | Yes | Event waiting |
| **Completion** | ~5-50μs wakeup | 0% while waiting | Yes | Thread signaling |

### **Scalability Tips**

- **Per-CPU data**: Eliminates locking for CPU-local stats
- **Read-Copy-Update (RCU)**: For read-mostly data structures
- **Seqlocks**: For write-rarely, read-frequently data (e.g., jiffies)
- **Lockless ring buffers**: Single-producer, single-consumer patterns

---

## **8. Troubleshooting Guide** <a name="troubleshooting"></a>

### **System Hangs: Is it a deadlock?**

1. **Enable Magic SysRq**: `echo 1 > /proc/sys/kernel/sysrq`
2. **Trigger backtrace**: Press `Alt+SysRq+t` (shows all tasks)
3. **Look for**: `D` state (uninterruptible sleep) tasks, lock chains
4. **Check lockdep**: `dmesg | grep -i deadlock`

### **Intermittent Data Corruption**

- **Likely cause**: Missing memory barrier or non-atomic access
- **Check**: Are all shared variables accessed with proper locks?
- **Test**: Enable `CONFIG_KASAN` (Kernel Address Sanitizer) to catch races

### **Module Won't Unload: Device Busy**

- **Check**: `cat /proc/modules` shows reference count
- **Cause**: `wait_for_completion()` in `module_exit()` waiting for non-exiting thread
- **Fix**: Use `kthread_stop()` or `complete_and_exit()` pattern

### **LEDs Don't Blink**

- **Verify**: `fg_console` is correct (may be 0 on headless systems)
- **Check**: `my_driver` is not NULL; some virtual terminals lack LED control
- **Permission**: Module must run with CAP_SYS_TTY_CONFIG capability

---

## **9. Additional Resources & Further Reading** <a name="resources"></a>

### **Kernel Documentation**
- `Documentation/locking/locktypes.rst` - Official lock type rules
- `Documentation/timers/timers-howto.rst` - Timer API usage
- `Documentation/RCU/whatisRCU.rst` - Read-Copy-Update for scalable reads
- `Documentation/core-api/atomic_ops.rst` - Atomic operations reference

### **External Resources**
- **"Linux Device Drivers, 3rd Edition"**: Classic LDD3 (free online)
- **Kernel Newbies**: https://kernelnewbies.org/ - Great for beginners
- **Brendan Gregg's Blog**: Performance analysis tools
- **Linux Kernel Mailing List (LKML)**: Latest discussions on API changes

### **Debugging Tools**
- **`ftrace`**: Trace function calls and latency
- **`perf`**: Performance counters and profiling
- **`crash`**: Analyze kernel core dumps
- **`trace-cmd`**: User-friendly ftrace frontend

### **Security Considerations**
- **Spectre/Meltdown**: Spinlocks can be side-channel attack vectors
- **Use `raw_spinlock_t`** in security-critical paths after careful analysis
- **Avoid timers in modules**: Prefer workqueues for flexibility

---

## **10. Example Makefile Compilation**

```makefile
obj-m += sleep.o completions.o example_mutex.o
obj-m += example_spinlock.o example_rwlock.o example_atomic.o
obj-m += print_string.o kbleds.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

install:
	sudo insmod sleep.ko
	tail -f /proc/sleep &
	cat_nonblock /proc/sleep
	kill %1
	sudo rmmod sleep
```

---

## **11. Summary Decision Tree**

```
Need to protect data?
├── Short (< 1ms) and no sleep possible?
│   ├── Interrupts involved? → spin_lock_irqsave()
│   └── Only process context? → spin_lock()
│
├── Read-mostly data?
│   ├── Many readers, rare writers → read_lock_irqsave()
│   └── Mixed reads/writes → mutex_lock_interruptible()
│
└── Long operation or may sleep?
    ├── Exclusive access needed → mutex_lock_interruptible()
    └── Ordering between threads → wait_for_completion()
```

---

This README consolidates essential kernel module development knowledge for concurrent programming, synchronization, and debugging. Master these patterns before writing production kernel code.