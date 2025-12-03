# Linux Kernel Task Scheduling and Interrupt Handling

## 📑 Table of Contents

1. [Chapter Overview](#chapter-overview)
2. [The Task Scheduling Landscape](#scheduling-landscape)
3. [Tasklets - Legacy Softirq Mechanism](#tasklets)
   - [What Are Tasklets and Why They Exist](#tasklets-purpose)
   - [Tasklet Code Deep Dive](#tasklet-code)
   - [Drawbacks and Deprecation Status](#tasklets-drawbacks)
4. [Workqueues - Modern Deferred Work](#workqueues)
   - [Workqueue Architecture](#workqueue-architecture)
   - [Workqueue Code Analysis](#workqueue-code)
   - [Comparison with Tasklets](#workqueue-vs-tasklet)
5. [Interrupt Handler Fundamentals](#interrupt-fundamentals)
   - [IRQ Types and Categories](#irq-types)
   - [Top Half vs Bottom Half Concept](#top-bottom-half)
   - [Linux IRQ Subsystem Architecture](#irq-subsystem)
6. [GPIO Interrupt Example - Button Handling](#gpio-interrupts)
   - [Hardware Setup](#button-hardware)
   - [Interrupt Registration Process](#irq-registration)
   - [Complete Driver Walkthrough](#intrpt-code)
   - [Kernel Version Compatibility](#compatibility-issues)
7. [Bottom Halves with Workqueues](#bottomhalf-workqueue)
   - [Design Pattern](#bh-pattern)
   - [Implementation Details](#bh-code)
8. [Threaded IRQs - The Modern Standard](#threaded-irqs)
   - [Threaded IRQ Architecture](#threaded-architecture)
   - [Implementation Example](#threaded-code)
   - [Advantages Over Traditional Approaches](#threaded-advantages)
9. [API Function Reference](#api-reference)
   - [Tasklet API](#api-tasklet)
   - [Workqueue API](#api-workqueue)
   - [IRQ Management API](#api-irq)
   - [GPIO to IRQ Conversion](#api-gpio-to-irq)
10. [Critical Concepts and Gotchas](#critical-concepts)
11. [Troubleshooting Guide](#troubleshooting)
12. [Best Practices and Recommendations](#best-practices)
13. [Further Resources](#resources)

---

## Chapter Overview

This chapter covers **task scheduling** and **interrupt handling** in the Linux kernel, two fundamental mechanisms for managing asynchronous work. While GPIO drivers execute synchronously, real-world hardware requires handling events that occur at unpredictable times (button presses, network packets, timer expiry). This requires splitting work between **immediate, urgent processing** and **deferred, time-consuming tasks**.

**Key Learning Objectives:**
- Understand the difference between tasklets, workqueues, and threaded IRQs
- Master the interrupt top-half/bottom-half design pattern
- Implement GPIO-based interrupt handlers for physical buttons
- Manage deferred work in a thread-safe manner
- Handle kernel version compatibility (especially Linux 6.10+)
- Debug concurrency issues and race conditions

---

## The Task Scheduling Landscape

The Linux kernel provides multiple mechanisms for scheduling deferred work, each with different trade-offs:

```
Urgency Level    Mechanism              Context          Can Sleep?
─────────────    ─────────────────────  ───────────────  ──────────
Critical         Interrupt Top Half     Hard IRQ         NO
High             SoftIRQ/Tasklet        Atomic           NO
Medium           Workqueue              Process          YES
Low              Kernel Thread          Process          YES
Immediate        Threaded IRQ           Both             YES (bottom)
```

**Why Not Just Do Everything in the Interrupt Handler?**

Interrupt handlers execute in **atomic context** with these restrictions:
- **Cannot sleep**: No `msleep()`, mutexes, or blocking calls
- **Cannot access user space**: No `copy_from_user()`
- **Must be fast**: Holding off interrupts for too long causes system latency
- **Limited stack**: Typically 2-4KB, far less than process context

The **deferred execution model** solves this by splitting work:
1. **Top Half**: Acknowledge hardware, save minimal state, schedule deferred work
2. **Bottom Half**: Perform the real processing in a more permissive context

---

## Tasklets - Legacy Softirq Mechanism

### What Are Tasklets and Why They Exist

**Tasklets** are a legacy mechanism built on top of **softirqs** (software interrupts). They were introduced in Linux 2.3 to replace the older "bottom half" (BH) API. Tasklets provide:
- **Atomic execution**: Runs with interrupts enabled but cannot be preempted by other tasklets on the same CPU
- **Serialization**: Only one instance of a tasklet can run at a time (same tasklet won't run in parallel)
- **Easy API**: Simple function pointer-based interface

**Historical Context**: In early Linux, BH handlers were global and non-preemptible. Tasklets improved this by allowing multiple tasklet types and per-CPU processing. However, they still run with softirq scheduling, which has latency issues.

### Tasklet Code Deep Dive

```c
#include <linux/delay.h>
#include <linux/interrupt.h>
#include <linux/module.h>
#include <linux/printk.h>

/* Compatibility macro for kernel 5.16+ where DECLARE_TASKLET changed */
#ifndef DECLARE_TASKLET_OLD
#define DECLARE_TASKLET_OLD(arg1, arg2) DECLARE_TASKLET(arg1, arg2, 0L)
#endif

static void tasklet_fn(unsigned long data)
{
    pr_info("Example tasklet starts\n");
    mdelay(5000);  // Busy-wait for 5 seconds!
    pr_info("Example tasklet ends\n");
}
```

**Tasklet Function Constraints**:
- Runs in **softirq context** (atomic)
- **Cannot sleep**: Using `msleep()` would cause a kernel panic
- The code incorrectly uses `mdelay()` (busy-wait) which **wastes CPU** for 5 seconds
- Real tasklets should complete in **microseconds to milliseconds** maximum

```c
static DECLARE_TASKLET_OLD(mytask, tasklet_fn);

static int __init example_tasklet_init(void)
{
    pr_info("tasklet example init\n");
    tasklet_schedule(&mytask);
    
    mdelay(200);  // Main thread continues after 200ms
    pr_info("Example tasklet init continues...\n");
    return 0;
}

static void __exit example_tasklet_exit(void)
{
    pr_info("tasklet example exit\n");
    tasklet_kill(&mytask);  // Wait for tasklet to complete
}
```

**Execution Flow**:
1. `tasklet_schedule(&mytask)` adds the tasklet to a per-CPU queue
2. The scheduler runs it **after** the current IRQ/top-half completes
3. `mdelay(200)` allows the init function to continue **before** the tasklet finishes
4. `tasklet_kill()` ensures clean shutdown by waiting for completion

**Output Analysis**:
```
tasklet example init          ← Init function runs
Example tasklet starts        ← Tasklet scheduled and starts
Example tasklet init continues... ← Init continues after 200ms
Example tasklet ends          ← Tasklet completes after 5s
```

The init function **doesn't wait** for the tasklet. The tasklet runs asynchronously in softirq context.

### Drawbacks and Deprecation Status

**Why Tasklets Are Discouraged:**

1. **Cannot Sleep**: All operations must be atomic, limiting functionality
2. **Unpredictable Latency**: Run at softirq priority, can be delayed by other softirqs
3. **No Concurrency**: Same tasklet cannot run on multiple CPUs simultaneously (serialization can become a bottleneck)
4. **Memory Management Issues**: Difficult to coordinate with memory reclaim
5. **Debugging Difficulty**: Tracepoints and debugging are limited in atomic context

**Current Status** (Linux 6.x):
- **Not yet removed** but marked as **deprecated**
- **130+ uses remain** in mainline kernel (slow migration)
- **Threaded IRQs** are the recommended replacement
- **LWN article** (https://lwn.net/Articles/830964/) discusses removal plans since **2007**

**Migration Path**:
- Simple tasklets → **Workqueues** (process context)
- IRQ-related tasklets → **Threaded IRQs** (`request_threaded_irq()`)

---

## Workqueues - Modern Deferred Work

### Workqueue Architecture

**Workqueues** execute work in **process context** (kernel threads), solving tasklet limitations:
- **Can sleep**: Full access to blocking APIs, mutexes, memory allocation
- **Better scheduling**: Uses CFS (Completely Fair Scheduler) like normal processes
- **Parallel execution**: Can run multiple work items simultaneously on different CPUs
- **Flexible**: Can be bound to specific CPUs or unbound (runs anywhere)

**Workqueue Types**:
1. **System-wide workqueues**: `system_wq` (shared, use `schedule_work()`)
2. **Dedicated workqueues**: Custom queue for your driver (better isolation)
3. **BH workqueues**: `WQ_BH` flag runs in softirq context (atomic, but flexible)

### Workqueue Code Analysis

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/workqueue.h>

static struct workqueue_struct *queue = NULL;
static struct work_struct work;

static void work_handler(struct work_struct *data)
{
    pr_info("work handler function.\n");
    /* Can safely sleep here! */
    msleep(1000);  // OK in workqueue context
}
```

**Key Structures**:
-  **`workqueue_struct` **: Represents the queue and its properties
-  **`work_struct` **: Encapsulates a single work item and its handler

```c
static int __init sched_init(void)
{
    /* Create a dedicated workqueue */
    queue = alloc_workqueue("HELLOWORLD", WQ_UNBOUND, 1);
    if (!queue) {
        pr_err("Failed to allocate workqueue\n");
        return -ENOMEM;
    }
    
    /* Initialize work item */
    INIT_WORK(&work, work_handler);
    
    /* Submit work to queue */
    queue_work(queue, &work);
    
    return 0;
}

static void __exit sched_exit(void)
{
    flush_workqueue(queue);  // Wait for all work to complete
    destroy_workqueue(queue);
}
```

**Workqueue Creation Flags**:
-  **`WQ_UNBOUND` **: Work can run on any CPU (better for load balancing)
- ** `WQ_HIGHPRI` **: High priority queue (uses higher priority kernel thread)
- ** `WQ_BH` **: Runs in softirq context (atomic, like tasklets but more flexible)
-  **`WQ_MEM_RECLAIM` **: Special flag for memory reclaim paths

**Comparison**:
- ** `queue_work()` **: Submit to custom queue
- ** `schedule_work()` **: Submit to system-wide `system_wq`
- ** `schedule_delayed_work()` **: Submit with timeout delay

---

## Interrupt Handler Fundamentals

### IRQ Types and Categories

**IRQ (Interrupt ReQuest)** is the hardware signal from devices to CPU. Linux classifies interrupts:

**By Duration**:
- **Short IRQ**: Microseconds to 100s of microseconds. Blocks all other interrupts on same CPU.
- **Long IRQ**: Can be milliseconds. Allows other interrupts during processing.

**By Type**:
- **Edge-triggered**: IRQ fires on signal transition (rising/falling edge). Common for GPIO buttons.
- **Level-triggered**: IRQ fires while signal is active. Requires acknowledgment.

**By Sharing**:
- **Shared IRQ**: Multiple devices on same IRQ line (requires `IRQF_SHARED` flag)
- **Exclusive IRQ**: Single device per IRQ (preferred for GPIO)

### Top Half vs Bottom Half Concept

**Design Pattern**:
```
Hardware Event
     ↓
Interrupt Occurs (CPU stops current work)
     ↓
┌─────────────────────────────────────────┐
│   TOP HALF (Interrupt Context)          │
│   - Acknowledge hardware                │
│   - Save minimal state                  │
│   - Schedule bottom half                │
│   - Must be FAST (< 100μs)             │
│   - Cannot sleep                        │
└─────────────────────────────────────────┘
     ↓ (schedules)
┌─────────────────────────────────────────┐
│   BOTTOM HALF (Deferred Work)           │
│   - Process data                        │
│   - Access user space                   │
│   - Can sleep                           │
│   - Runs in process context             │
└─────────────────────────────────────────┘
```

**Historical Evolution**:
1. **Linux 1.x**: "Bottom Halves" (global, non-preemptible)
2. **Linux 2.2**: Softirqs introduced
3. **Linux 2.3**: Tasklets replace BH
4. **Linux 2.6**: Workqueues introduced
5. **Linux 3.x**: Threaded IRQs introduced
6. **Linux 6.x**: Tasklets deprecated

### Linux IRQ Subsystem Architecture

```
Hardware Device
    ↓ (physical IRQ line)
GPIO Controller (e.g., bcm2835_gpio)
    ↓ (generates IRQ)
IRQ Controller (GIC, APIC)
    ↓ (IRQ number)
Kernel IRQ Core (kernel/irq/)
    ↓
Interrupt Descriptor Table (IDT)
    ↓
Your driver (request_irq())
```

**Key APIs**:
-  **`request_irq()` **: Register interrupt handler
-  **`free_irq()` **: Unregister handler
-  **`enable_irq()` / `disable_irq()` **: Control IRQ line
-  **`irq_set_affinity()` **: Bind IRQ to specific CPU(s)

---

## GPIO Interrupt Example - Button Handling

### Button Hardware Setup

**Components**:
- Raspberry Pi 5
- 2x Push buttons (momentary switches)
- 2x 10kΩ pull-up resistors
- 1x LED with 220Ω resistor (same as before)

**Circuit Connections**:
```
Button 1: GPIO17 (Pin 11) → Switch → GND
Button 2: GPIO18 (Pin 12) → Switch → GND
LED: GPIO4 (Pin 7) → 220Ω → LED → GND

Both buttons need internal pull-up enabled (or external resistors):
- GPIO pin reads HIGH when button is OPEN
- GPIO pin reads LOW when button is PRESSED (connected to GND)
```

**GPIO Configuration**:
- Input mode with pull-up resistor
- Interrupt on **both edges** (rising and falling) to detect press and release

### Interrupt Registration Process

```c
/* Convert GPIO number to IRQ number */
int irq_num = gpio_to_irq(gpio_button_pin);

/* Request IRQ with handler */
ret = request_irq(irq_num, button_isr, IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING, 
                  "button_driver", NULL);
```

**IRQ Flags**:
- ** `IRQF_TRIGGER_RISING` **: Trigger on 0→1 transition
- ** `IRQF_TRIGGER_FALLING` **: Trigger on 1→0 transition
- ** `IRQF_TRIGGER_HIGH` **: Trigger while level is high
- ** `IRQF_TRIGGER_LOW` **: Trigger while level is low
- ** `IRQF_SHARED` **: Allow sharing IRQ with other drivers
- ** `IRQF_ONESHOT` **: Keep IRQ disabled until thread finishes (threaded IRQs only)

### Complete Driver Walkthrough

```c
#include <linux/gpio.h>
#include <linux/interrupt.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/printk.h>

static int button_irqs[] = { -1, -1 };

static struct gpio leds[] = { { 4, GPIOF_OUT_INIT_LOW, "LED 1" } };
static struct gpio buttons[] = {
    { 17, GPIOF_IN, "LED 1 ON BUTTON" },
    { 18, GPIOF_IN, "LED 1 OFF BUTTON" }
};
```

**Key Design**:
-  **`button_irqs` ** array stores IRQ numbers for both buttons
-  **`leds` ** and  **`buttons` ** arrays define GPIO configurations
-  **`GPIOF_IN` ** configures pins as inputs (no direction call needed)

```c
static irqreturn_t button_isr(int irq, void *data)
{
    /* Check which button caused the IRQ */
    if (irq == button_irqs[0] && !gpio_get_value(leds[0].gpio))
        gpio_set_value(leds[0].gpio, 1);  // Turn LED ON if currently OFF
    else if (irq == button_irqs[1] && gpio_get_value(leds[0].gpio))
        gpio_set_value(leds[0].gpio, 0);  // Turn LED OFF if currently ON
    
    return IRQ_HANDLED;  // Must return IRQ status
}
```

**ISR Implementation Details**:
- ** `irq` ** parameter identifies which button was pressed
- ** `gpio_get_value()` ** checks current LED state to prevent redundant operations
- ** `IRQ_HANDLED` ** tells kernel the IRQ was processed correctly
- ** No debouncing **: Mechanical buttons bounce (multiple IRQs per press). Production code needs debouncing logic.

```c
static int __init intrpt_init(void)
{
    int ret = 0;
    pr_info("%s\n", __func__);
    
    /* Register LED GPIOs */
    ret = gpio_request_array(leds, ARRAY_SIZE(leds));
    if (ret) {
        pr_err("Unable to request GPIOs for LEDs: %d\n", ret);
        return ret;
    }
    
    /* Register Button GPIOs */
    ret = gpio_request_array(buttons, ARRAY_SIZE(buttons));
    if (ret) {
        pr_err("Unable to request GPIOs for BUTTONs: %d\n", ret);
        goto fail1;
    }
```

**Initialization Flow**:
1. Request LED GPIOs first (outputs)
2. Request button GPIOs (inputs)
3. Note: `gpio_request_array()` is deprecated in Linux 6.10+

```c
    /* Convert GPIO17 to IRQ number */
    ret = gpio_to_irq(buttons[0].gpio);
    if (ret < 0) {
        pr_err("Unable to get IRQ: %d\n", ret);
        goto fail2;
    }
    button_irqs[0] = ret;
    pr_info("Successfully requested BUTTON1 IRQ # %d\n", button_irqs[0]);
    
    /* Register ISR for Button 1 */
    ret = request_irq(button_irqs[0], button_isr,
                      IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                      "gpiomod#button1", NULL);
    if (ret) {
        pr_err("Unable to request IRQ: %d\n", ret);
        goto fail2;
    }
```

**IRQ Registration Process**:
1.  **`gpio_to_irq()` ** translates GPIO pin to kernel IRQ number
2.  **`request_irq()` ** registers handler with specified trigger conditions
3. ** `"gpiomod#button1"` ** appears in `/proc/interrupts` for debugging
4. ** `NULL` ** is the driver data pointer (could pass struct pointer)

```c
    /* Repeat for Button 2 */
    ret = gpio_to_irq(buttons[1].gpio);
    if (ret < 0) {
        pr_err("Unable to get IRQ: %d\n", ret);
        goto fail3;  // Need to free first IRQ on error
    }
    button_irqs[1] = ret;
    
    ret = request_irq(button_irqs[1], button_isr,
                      IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                      "gpiomod#button2", NULL);
    if (ret) {
        pr_err("Unable to request IRQ: %d\n", ret);
        goto fail3;
    }
    
    return 0;
```

**Error Handling Pattern**:
- Each `goto` label represents cleanup of previously allocated resources
- **Reverse order**: Free in opposite order of allocation
- IRQs must be freed with `free_irq()` before GPIOs

```c
fail3:
    free_irq(button_irqs[0], NULL);
fail2:
    gpio_free_array(buttons, ARRAY_SIZE(buttons));
fail1:
    gpio_free_array(leds, ARRAY_SIZE(leds));
    return ret;
}
```

### Kernel Version Compatibility

**Linux 6.10+ Breaking Change**:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 10, 0)
#define NO_GPIO_REQUEST_ARRAY
#endif

#ifdef NO_GPIO_REQUEST_ARRAY
    ret = gpio_request(leds[0].gpio, leds[0].label);  // Individual requests
    ret = gpio_request(buttons[0].gpio, buttons[0].label);
    ret = gpio_request(buttons[1].gpio, buttons[1].label);
#else
    ret = gpio_request_array(leds, ARRAY_SIZE(leds));  // Deprecated/removed
    ret = gpio_request_array(buttons, ARRAY_SIZE(buttons));
#endif
```

**Why `gpio_request_array()` was removed**:
-  **`ARRAY_SIZE()` ** is error-prone with zero-length arrays
- ** Individual requests ** provide better error reporting
- ** Consistency ** with modern resource management patterns

---

## Bottom Halves with Workqueues

### Design Pattern

The ** bottom half pattern ** separates urgent interrupt work from deferred processing:

```
IRQ Triggered
    ↓
Top Half (ISR)
├─ Critical action (e.g., toggle LED)
└─ schedule_work(&work)  // Defer rest
    ↓
Bottom Half (Workqueue)
├─ Heavy processing
├─ Can sleep (msleep, mutex)
└─ Non-time-critical cleanup
```

**Advantages**:
- Keeps ISR latency low
- Full kernel API access in bottom half
- Better system responsiveness

### Implementation Details

```c
#include <linux/workqueue.h>

static struct work_struct bottomhalf_work;

static void bottomhalf_work_fn(struct work_struct *work)
{
    pr_info("Bottom half workqueue starts\n");
    
    /* Safe to sleep here! */
    msleep(500);  // Simulate heavy processing
    
    pr_info("Bottom half workqueue ends\n");
}

static DECLARE_WORK(bottomhalf_work, bottomhalf_work_fn);

static irqreturn_t button_isr(int irq, void *data)
{
    /* Top half: quick LED toggle */
    if (irq == button_irqs[0] && !gpio_get_value(leds[0].gpio))
        gpio_set_value(leds[0].gpio, 1);
    else if (irq == button_irqs[1] && gpio_get_value(leds[0].gpio))
        gpio_set_value(leds[0].gpio, 0);
    
    /* Bottom half: defer heavy work */
    schedule_work(&bottomhalf_work);
    
    return IRQ_HANDLED;
}
```

**Key Differences from Simple ISR**:
- ** `schedule_work()` ** adds work to system-wide `system_wq`
- ** Non-blocking **: ISR returns immediately; work runs later
- ** Process context **: Can access files, network, user memory (via workarounds)
- ** SMP-safe **: Work can run on any CPU (unless `WQ_UNBOUND` is specified)

---

## Threaded IRQs - The Modern Standard

### Threaded IRQ Architecture

**Threaded IRQs** unify top-half and bottom-half into a single API:

```
request_threaded_irq(irq, top_half, bottom_half, flags, name, dev_id)
                                      ↓
Hardware IRQ Triggered
    ↓
Top Half Handler (runs in IRQ context)
├─ Quick processing
├─ Return IRQ_WAKE_THREAD or IRQ_HANDLED
    ↓
If IRQ_WAKE_THREAD:
    Kernel wakes dedicated kernel thread
    ↓
Bottom Half Handler (runs in process context)
├─ Full kernel API access
├─ Can sleep
└─ Preemptible like normal process
```

**Advantages**:
- **Automatic thread management**: Kernel creates/destroys thread
- **Serialization**: One thread per IRQ (no concurrent execution)
- **Simplified code**: No manual workqueue management
- **Better latency**: Thread priority can be adjusted

### Implementation Example

```c
static irqreturn_t button_top_half(int irq, void *ident)
{
    /* Minimal top half: just wake thread */
    pr_info("Top half: IRQ %d triggered\n", irq);
    return IRQ_WAKE_THREAD;  // Must return this to wake thread
}

static irqreturn_t button_bottom_half(int irq, void *ident)
{
    /* Full processing in thread context */
    pr_info("Bottom half task starts\n");
    
    /* Safe to sleep, access data structures, etc. */
    msleep(500);
    
    /* Handle the button press */
    if (irq == button_irqs[0] && !gpio_get_value(leds[0].gpio))
        gpio_set_value(leds[0].gpio, 1);
    else if (irq == button_irqs[1] && gpio_get_value(leds[0].gpio))
        gpio_set_value(leds[0].gpio, 0);
    
    pr_info("Bottom half task ends\n");
    return IRQ_HANDLED;
}

static int __init bottomhalf_init(void)
{
    /* ... GPIO setup ... */
    
    /* Register threaded IRQ */
    ret = request_threaded_irq(button_irqs[0], 
                               button_top_half,      // Top half (can be NULL)
                               button_bottom_half,   // Bottom half (thread)
                               IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                               "gpiomod#button1", 
                               &buttons[0]);         // Pass data
    
    /* ... */
}
```

**Handler Flexibility**:
-  **`NULL` top half **: Just wakes thread (simpler)
-  **`NULL` bottom half **: Behaves like `request_irq()` (legacy compatibility)
- ** Both non-NULL **: Separate urgent and deferred work

---

## API Function Reference

### Tasklet API

| Function | Purpose | Context | Can Sleep? |
|----------|---------|---------|------------|
| `DECLARE_TASKLET(name, func, data)` | Statically declare tasklet | Any | N/A |
| `tasklet_schedule(&name)` | Queue tasklet for execution | Any | No |
| `tasklet_kill(&name)` | Wait for completion and disable | Process | Yes (blocks) |
| `tasklet_disable(&name)` | Temporarily disable tasklet | Any | No |
| `tasklet_enable(&name)` | Re-enable disabled tasklet | Any | No |

** Status**: Deprecated but not removed. Use for legacy code only.

### Workqueue API

| Function | Purpose | Context |
|----------|---------|---------|
| `alloc_workqueue(name, flags, max_active)` | Create dedicated workqueue | Process |
| `create_singlethread_workqueue(name)` | Create single-thread queue | Process |
| `INIT_WORK(work, func)` | Initialize work structure | Any |
| `queue_work(queue, work)` | Submit work to specific queue | Any |
| `schedule_work(work)` | Submit to system queue | Any |
| `flush_workqueue(queue)` | Wait for all work to complete | Process |
| `destroy_workqueue(queue)` | Cleanup workqueue | Process |

** Common Flags **:
- `WQ_UNBOUND`: No CPU affinity
- `WQ_HIGHPRI`: High priority
- `WQ_BH`: Atomic context (like tasklets)

### IRQ Management API

| Function | Purpose | Parameters |
|----------|---------|------------|
| `int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev)` | Register IRQ handler | irq: IRQ number<br>handler: ISR function<br>flags: trigger/behavior<br>name: /proc name<br>dev: driver data |
| `int request_threaded_irq(unsigned int irq, irq_handler_t top, irq_handler_t bottom, unsigned long flags, const char *name, void *dev)` | Register threaded IRQ | top: top half (can be NULL)<br>bottom: thread handler |
| `void free_irq(unsigned int irq, void *dev)` | Unregister IRQ handler | irq: IRQ number<br>dev: must match request_irq() |
| `int gpio_to_irq(unsigned int gpio)` | Convert GPIO to IRQ | gpio: GPIO number<br>Returns IRQ number or negative error |

**IRQ Return Values**:
-  `IRQ_HANDLED` : IRQ was processed, no further action
- `IRQ_WAKE_THREAD`: Wake threaded bottom half (threaded IRQ only)
- `IRQ_NONE`: IRQ was not for this device (shared IRQs)

### GPIO to IRQ Conversion

**How it Works**:
1. **GPIO pin** must be configured as input first
2. **IRQ controller** maps each GPIO pin to a virtual IRQ number
3. **Mapping is platform-specific**:
   - Raspberry Pi: Usually `irq = gpio + 64` (but use `gpio_to_irq()`!)
   - Other SoCs: May have different offset schemes

**Important**: Always use `gpio_to_irq()` **never** hardcode the offset. The mapping can change between kernel versions and hardware revisions.

---

## Critical Concepts and Gotchas

### 1. **Atomic Context vs Process Context**

**Atomic Context** (Interrupts, SoftIRQs, Tasklets):
```c
/* CANNOT DO THIS */
void bad_tasklet_fn(unsigned long data)
{
    msleep(1000);        // ↓ Kernel panic
    mutex_lock(&mutex);  // ↓ Deadlock
    kmalloc(GFP_KERNEL); // ↓ May sleep
}
```

**Process Context** (Workqueues, Threads):
```c
/* SAFE TO DO THIS */
void good_work_fn(struct work_struct *work)
{
    msleep(1000);        // OK
    mutex_lock(&mutex);  // OK
    kmalloc(GFP_KERNEL); // OK
}
```

**How to Check Context**:
```c
#include <linux/preempt.h>

if (in_interrupt()) {
    pr_err("Running in atomic context!\n");
}
if (in_softirq()) {
    pr_err("Running in softirq/tasklet!\n");
}
```

### 2. **IRQ Debouncing**

**Problem**: Mechanical buttons bounce, causing multiple IRQs per press.
**Solutions**:
```c
/* Hardware debouncing: Use RC filter */
GPIO → 10nF cap → GND
     → 10kΩ resistor → Button → GND

/* Software debouncing in top half */
static irqreturn_t button_isr(int irq, void *data)
{
    static unsigned long last_time = 0;
    unsigned long now = jiffies;
    
    if (now - last_time < msecs_to_jiffies(50))  // Ignore < 50ms
        return IRQ_HANDLED;  // Spurious, ignore
    
    last_time = now;
    // Process real press
}
```

### 3. **IRQ Sharing**

**When to Share**: Limited IRQ lines (e.g., on x86 PC)
**How to Share**:
```c
/* Both drivers must use IRQF_SHARED */
ret = request_irq(irq, my_isr, IRQF_SHARED, "driver1", dev1);
ret = request_irq(irq, other_isr, IRQF_SHARED, "driver2", dev2);

/* ISR must check if IRQ is for this device */
static irqreturn_t my_isr(int irq, void *dev)
{
    if (!device_has_interrupt())  // Check hardware status
        return IRQ_NONE;  // Not for us
    
    // Process interrupt
    return IRQ_HANDLED;
}
```

**Raspberry Pi**: GPIO IRQs are **not shared** (each GPIO has its own IRQ), so `IRQF_SHARED` is unnecessary.

### 4. **SMP and CPU Affinity**

**Problem**: IRQs may arrive on any CPU, causing cache bouncing.
**Solution**: Set IRQ affinity:
```bash
# Check current CPU affinity
cat /proc/irq/IRQ_NUMBER/smp_affinity_list

# Pin IRQ to CPU0 only
echo 0 | sudo tee /proc/irq/IRQ_NUMBER/smp_affinity_list
```

**In Code**:
```c
#include <linux/cpumask.h>

cpumask_t mask;
cpumask_clear(&mask);
cpumask_set_cpu(0, &mask);  // CPU 0 only
irq_set_affinity(irq, &mask);
```

### 5. **IRQf_ONESHOT for Threaded IRQs**

**Use Case**: Shared IRQs with threaded handlers
```c
ret = request_threaded_irq(irq, top_half, bottom_half, 
                           IRQF_SHARED | IRQF_ONESHOT, name, dev);
```

**What it does**:
- Keeps IRQ **disabled** after top half
- Re-enables after bottom half completes
- Prevents re-entrance on shared IRQs
- Mandatory for `IRQF_SHARED` with threaded IRQs

---

## Troubleshooting Guide

### Problem: `request_irq()` returns -EBUSY (Device or resource busy)

**Diagnosis**:
```bash
# Check if IRQ is already in use
cat /proc/interrupts | grep <IRQ_NUMBER>

# Find which driver claimed it
ls -l /sys/kernel/irq/<IRQ_NUMBER>/  # Shows driver name
```

**Solutions**:
- Use different GPIO pin
- Check for duplicate module loading: `lsmod`
- Unload conflicting driver: `sudo rmmod conflicting_driver`
- If sharing is necessary, use `IRQF_SHARED` flag

### Problem: ISR not called when button pressed

**Checklist**:
```bash
# 1. Verify GPIO is configured as input
cat /sys/kernel/debug/gpio | grep GPIO17

# 2. Check IRQ is registered
cat /proc/interrupts | grep gpiomod

# 3. Verify trigger mode
cat /sys/kernel/debug/gpio | grep gpio-17  # Should show IRQ edge configuration

# 4. Test with sysfs
echo 17 > /sys/class/gpio/export
echo in > /sys/class/gpio/gpio17/direction
cat /sys/class/gpio/gpio17/value  # Manually read button
```

**Code Fixes**:
```c
// Enable pull-up resistor if not already done
gpio_set_pullup(gpio, true);

// Check gpio_to_irq() return value
int irq = gpio_to_irq(gpio);
if (irq < 0) {
    pr_err("Invalid IRQ: %d\n", irq);
    return irq;  // Pass error up
}
```

### Problem: Multiple IRQs per button press (bouncing)

**Symptom**: LED flickers or toggles multiple times
**Solution**:
```c
/* Add 50ms debounce in top half */
static irqreturn_t button_isr(int irq, void *data)
{
    static unsigned long last_irq_time[2] = {0, 0};
    int button_idx = (irq == button_irqs[0]) ? 0 : 1;
    
    if (time_before(jiffies, last_irq_time[button_idx] + msecs_to_jiffies(50)))
        return IRQ_HANDLED;  // Ignore spurious IRQ
    
    last_irq_time[button_idx] = jiffies;
    
    // ... process real press ...
}
```

### Problem: System hangs or latency spikes

**Cause**: ISR taking too long
**Debug**:
```bash
# Enable IRQ latency tracing
echo 1 > /proc/sys/kernel/irq_time_tracking

# Check /proc/interrupts for excessive counts
watch -n 1 'cat /proc/interrupts | grep gpiomod'
```

**Fix**: Move heavy processing to workqueue or threaded IRQ bottom half.

### Problem: Kernel panic on `msleep()` in ISR

**Root Cause**: ISR is atomic context; `msleep()` tries to schedule.
**Solution**: Use workqueue or threaded IRQ instead.

```c
/* WRONG */
static irqreturn_t bad_isr(int irq, void *data)
{
    msleep(100);  // ↓ Kernel panic
    return IRQ_HANDLED;
}

/* CORRECT */
static irqreturn_t good_isr(int irq, void *data)
{
    schedule_work(&my_work);  // Defer to workqueue
    return IRQ_HANDLED;
}
```

---

## Best Practices and Recommendations

### 1. **Use Threaded IRQs for New Code**

**Modern approach** (Linux 3.0+):
```c
/* Single handler - simple case */
request_threaded_irq(irq, NULL, button_thread_fn, flags, name, dev);

/* Split handlers - complex case */
request_threaded_irq(irq, button_top, button_bottom, flags, name, dev);
```

**Advantages over workqueues**:
- Automatic thread management
- No need to create/destroy workqueues
- Better integration with IRQ core
- Priority inheritance

### 2. **Prefer Descriptor-Based GPIO API**

**Modern GPIO API** (`include/linux/gpio/consumer.h`):
```c
struct gpio_desc *gpiod;

// Request GPIO
gpiod = gpiod_get(dev, "button", GPIOD_IN);
if (IS_ERR(gpiod))
    return PTR_ERR(gpiod);

// Convert to IRQ
int irq = gpiod_to_irq(gpiod);

// Request threaded IRQ
request_threaded_irq(irq, NULL, handler, IRQF_TRIGGER_BOTH, "button", dev);
```

**Benefits**:
- **No magic numbers**: Uses device tree or board files
- **Automatic cleanup**: `gpiod_put()` releases all resources
- **Better error messages**: Includes device and property names

### 3. **Implement Proper Debouncing**

**Hybrid approach** (hardware + software):
```c
static irqreturn_t button_isr(int irq, void *data)
{
    /* Immediately disable IRQ */
    disable_irq_nosync(irq);
    
    /* Schedule delayed work to re-enable after debounce */
    schedule_delayed_work(&debounce_work, msecs_to_jiffies(50));
    
    /* Process the press */
    // ...
    
    return IRQ_HANDLED;
}

static void debounce_work_fn(struct work_struct *work)
{
    /* Re-enable IRQ after debounce period */
    enable_irq(button_irq);
}
```

### 4. **Use devm_* for Automatic Resource Cleanup**

**Device-managed resources** (never leak on error paths):
```c
static int probe(struct platform_device *pdev)
{
    /* Request GPIO - auto-freed on driver removal */
    int gpio = devm_gpio_request(dev, gpio_num, "button");
    
    /* Get IRQ - auto-freed */
    int irq = gpio_to_irq(gpio);
    devm_request_threaded_irq(dev, irq, NULL, handler, flags, "button", NULL);
    
    return 0;  // No cleanup needed on error!
}
```

**Benefits**: Eliminates error-prone `goto` cleanup chains.

### 5. **Document IRQ Timing Requirements**

**In driver header**:
```c
/**
 * Button IRQ Handler
 * 
 * Top half must complete within 50μs to avoid system latency.
 * Bottom half processes press events and may take up to 1ms.
 * 
 * Debouncing: 50ms software debounce in top half.
 * 
 * @irq: IRQ number from gpio_to_irq()
 * @data: Pointer to button_device struct
 * 
 * Returns IRQ_HANDLED if button press was processed.
 */
static irqreturn_t button_isr(int irq, void *data)
```

### 6. **Test Under Load**

**Stress testing**:
```bash
# Generate heavy IRQ load
sudo apt install stress-ng
stress-ng --io 4 --cpu 4 --timeout 60s &

# In another terminal, hammer the button
while true; do
    # Simulate rapid button presses
    gpioset gpiochip0 17=0; sleep 0.01; gpioset gpiochip0 17=1; sleep 0.01
done &

# Monitor for missed IRQs or latency
watch -d 'cat /proc/interrupts | grep button'
```

---

## Further Resources

### Official Documentation
- **Linux IRQ Subsystem**: `Documentation/core-api/irq.rst`
- **GPIO IRQ Binding**: `Documentation/devicetree/bindings/gpio/gpio.txt`
- **Threaded IRQs**: `Documentation/core-api/threaded-irqs.rst`

### Tools and Debugging
```bash
# IRQ latency measurement
sudo apt install irqbalance
sudo irqbalance --debug

# Trace IRQs in real-time
sudo trace-cmd record -e irq_handler_entry -e irq_handler_exit

# Visualize IRQ distribution
sudo apt install kernelshark
```

### Kernel Source Files to Study
- **`kernel/irq/manage.c`**: Core IRQ management (`request_irq()` implementation)
- **`kernel/workqueue.c`**: Workqueue subsystem
- **`kernel/softirq.c`**: Softirq and tasklet implementation
- **`drivers/gpio/gpio-bcm2835.c`**: Raspberry Pi GPIO IRQ handling

### Recommended Reading
- *Linux Kernel Development, 3rd Ed.* - Robert Love (Chapter 7: Interrupts)
- *Understanding the Linux Kernel, 3rd Ed.* - Bovet & Cesati (Chapter 4: Interrupts and Exceptions)
- *Professional Linux Kernel Architecture* - Wolfgang Mauerer

### Migration Guides
- **Tasklet to Workqueue**: https://lwn.net/Articles/830964/
- **Legacy IRQ to Threaded IRQ**: `Documentation/core-api/threaded-irqs.rst`

---

## Summary

This chapter covers the evolution of Linux deferred work mechanisms:

| Mechanism | Context | Can Sleep | Use Case | Status |
|-----------|---------|-----------|----------|--------|
| ** Tasklet ** | Softirq | No | Simple, fast work (< 1ms) | Deprecated |
| ** Workqueue ** | Process | Yes | Heavy processing, flexible | Current standard |
| ** Threaded IRQ ** | Both | Yes (bottom) | Hardware IRQ handling | ** Recommended ** |

**Decision Tree for New Code**:

```
Need to handle hardware IRQ?
├─ YES → Use threaded IRQ (request_threaded_irq())
│         ├─ Simple case: NULL top half
│         └─ Complex: Split top/bottom handlers
│
└─ NO  → Need deferred work?
          ├─ YES → Use workqueue (alloc_workqueue())
          │        ├─ Unbound for flexibility
          │        └─ High priority if latency-sensitive
          │
          └─ NO  → Process synchronously
```

** Key Takeaways **:
1. ** Never block in ISR **: Keep top halves under 100μs
2. ** Use threaded IRQs **: Modern, flexible, kernel-managed
3. ** Debounce hardware **: Mechanical switches need debouncing
4. ** Check kernel version **: APIs change (tasklets deprecated, gpio_request_array removed)
5. ** Test under load **: IRQ latency issues appear only under stress

The ** GPIO interrupt example ** provides a practical foundation for all interrupt-driven drivers. The ** threaded IRQ approach ** represents the current best practice, combining the urgency of immediate IRQ handling with the flexibility of process context execution.