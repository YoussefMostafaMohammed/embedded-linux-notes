# Linux Character Device Drivers

## 📑 Table of Contents

### **Part 1: Foundation and Basics**
1. [Introduction to Character Devices](#1-introduction-to-character-devices)
   - 1.1 [What Are Character Devices?](#11-what-are-character-devices)
   - 1.2 [The File-Like Interface](#12-the-file-like-interface)
   - 1.3 [Common Examples](#13-common-examples)

2. [Core Data Structures](#2-core-data-structures)
   - 2.1 [struct file_operations - The Heart of Your Driver](#21-struct-file_operations---the-heart-of-your-driver)
     - 2.1.1 [Essential Fields and Their Purposes](#211-essential-fields-and-their-purposes)
     - 2.1.2 [Modern Designated Initializers](#212-modern-designated-initializers)
     - 2.1.3 [The Owner Field - Critical Importance](#213-the-owner-field---critical-importance)
   - 2.2 [struct file (filp) - Per-Open State](#22-struct-file-filp---per-open-state)
     - 2.2.1 [The filp Pointer - What It Represents](#221-the-filp-pointer---what-it-represents)
     - 2.2.2 [Key Fields: private_data and f_pos](#222-key-fields-private_data-and-f_pos)
     - 2.2.3 [Complete User-Kernel Interaction Flow](#223-complete-user-kernel-interaction-flow)
     - 2.2.4 [Why struct file is Essential - Without It Would Fail](#224-why-struct-file-is-essential---without-it-would-fail)

3. [Device Number Management](#3-device-number-management)
   - 3.1 [Major and Minor Numbers Explained](#31-major-and-minor-numbers-explained)
   - 3.2 [Static vs Dynamic Allocation - Trade-offs](#32-static-vs-dynamic-allocation---trade-offs)
   - 3.3 [dev_t Type and Manipulation Macros](#33-dev_t-type-and-manipulation-macros)
   - 3.4 [Checking Allocated Numbers](#34-checking-allocated-numbers)

4. [Legacy Device Registration](#4-legacy-device-registration)
   - 4.1 [register_chrdev() - The Old Way](#41-register_chrdev---the-old-way)
     - 4.1.1 [Characteristics and Usage](#411-characteristics-and-usage)
     - 4.1.2 [The Fatal Flaws](#412-the-fatal-flaws)

### **Part 2: Modern Device Registration**
5. [Modern Device Registration (cdev Interface)](#5-modern-device-registration-cdev-interface)
   - 5.1 [Two-Step Process Overview](#51-two-step-process-overview)
   - 5.2 [Step 1: Allocating Device Numbers](#52-step-1-allocating-device-numbers)
     - 5.2.1 [Static Allocation - register_chrdev_region()](#521-static-allocation---register_chrdev_region)
     - 5.2.2 [Dynamic Allocation - alloc_chrdev_region()](#522-dynamic-allocation---alloc_chrdev_region)
   - 5.3 [Step 2: cdev Initialization](#53-step-2-cdev-initialization)
     - 5.3.1 [Understanding cdev_alloc()](#531-understanding-cdev_alloc)
     - 5.3.2 [Understanding cdev_init()](#532-understanding-cdev_init)
     - 5.3.3 [cdev_alloc() vs cdev_init() - The ONLY Difference](#533-cdev_alloc-vs-cdev_init---the-only-difference)
   - 5.4 [Step 3: Registering the Device](#54-step-3-registering-the-device)
     - 5.4.1 [cdev_add() Parameters and Validation](#541-cdev_add-parameters-and-validation)
     - 5.4.2 [What Happens Internally During cdev_add()](#542-what-happens-internally-during-cdev_add)
   - 5.5 [The Global Device Hash Table](#55-the-global-device-hash-table)
   - 5.6 [Complete Modern Registration Example](#56-complete-modern-registration-example)

### **Part 3: Memory Allocation Patterns**
6. [Memory Allocation Patterns - The Definitive Guide](#6-memory-allocation-patterns---the-definitive-guide)
   - 6.1 [Three Ways to Allocate Memory](#61-three-ways-to-allocate-memory)
   - 6.2 [Static Allocation - When and How](#62-static-allocation---when-and-how)
     - 6.2.1 [Simple Static Character Device - Full Example](#621-simple-static-character-device---full-example)
   - 6.3 [Dynamic Allocation with kmalloc](#63-dynamic-allocation-with-kmalloc)
   - 6.4 [The Embedding Pattern - Professional Method](#64-the-embedding-pattern---professional-method)
     - 6.4.1 [Why Embed cdev in Your Struct?](#641-why-embed-cdev-in-your-struct)
     - 6.4.2 [Complete Multi-Device Driver Example](#642-complete-multi-device-driver-example)
     - 6.4.3 [Error Handling in Multi-Device Drivers](#643-error-handling-in-multi-device-drivers)
   - 6.5 [Managed Allocations (devm_) - Modern Approach](#65-managed-allocations-devm---modern-approach)
     - 6.5.1 [When to Use devm_ Functions](#651-when-to-use-devm_-functions)
     - 6.5.2 [Complete USB Probe Example](#652-complete-usb-probe-example)
   - 6.6 [cdev_alloc() When and Why - Rare Cases](#66-cdev_alloc-when-and-why---rare-cases)
     - 6.6.1 [USB Gadget Example - Unknown Device Count](#661-usb-gadget-example---unknown-device-count)
   - 6.7 [The Complete Decision Tree](#67-the-complete-decision-tree)

### **Part 4: Concurrency and Synchronization**
7. [Concurrency and Thread Safety](#7-concurrency-and-thread-safety)
   - 7.1 [The Race Condition Problem](#71-the-race-condition-problem)
     - 7.1.1 [Real-World Scenarios](#711-real-world-scenarios)
   - 7.2 [Atomic Operations (atomic_t) - Lock-Free](#72-atomic-operations-atomic_t---lock-free)
     - 7.2.1 [What is atomic_t and Why a Struct?](#721-what-is-atomic_t-and-why-a-struct)
     - 7.2.2 [ATOMIC_INIT() - Compile-Time Initialization](#722-atomic_init---compile-time-initialization)
     - 7.2.3 [atomic_cmpxchg() - Line-by-Line Breakdown](#723-atomic_cmpxchg---line-by-line-breakdown)
     - 7.2.4 [Assembly-Level Operation](#724-assembly-level-operation)
     - 7.2.5 [Advantages Over Mutexes](#725-advantages-over-mutexes)
   - 7.3 [Other Synchronization Primitives](#73-other-synchronization-primitives)
     - 7.3.1 [Mutexes - For Complex Critical Sections](#731-mutexes---for-complex-critical-sections)
     - 7.3.2 [Spinlocks - For Short Atomic Contexts](#732-spinlocks---for-short-atomic-contexts)
     - 7.3.3 [Read-Write Semaphores - For Read-Heavy Workloads](#733-read-write-semaphores---for-read-heavy-workloads)
   - 7.4 [The Golden Rule of Synchronization](#74-the-golden-rule-of-synchronization)

### **Part 5: User-Kernel Data Transfer**
8. [User-Kernel Data Transfer](#8-user-kernel-data-transfer)
   - 8.1 [The User Pointer Problem - Why You Can't Dereference Directly](#81-the-user-pointer-problem---why-you-cant-dereference-directly)
   - 8.2 [put_user() and get_user() - Single Value Transfer](#82-put_user-and-get_user---single-value-transfer)
     - 8.2.1 [What They Do Internally](#821-what-they-do-internally)
   - 8.3 [copy_to_user() and copy_from_user() - Buffer Transfer](#83-copy_to_user-and-copy_from_user---buffer-transfer)
   - 8.4 [Page Faults - What They Are and Why They Matter](#84-page-faults---what-they-are-and-why-they-matter)
     - 8.4.1 [User-Space Pages vs Kernel-Space Pages](#841-user-space-pages-vs-kernel-space-pages)
   - 8.5 [Correct vs Incorrect Data Transfer - Real Examples](#85-correct-vs-incorrect-data-transfer---real-examples)

### **Part 6: Kernel Version Compatibility**
9. [Kernel Version Compatibility](#9-kernel-version-compatibility)
   - 9.1 [Why Kernel APIs Change - Internal vs System Call Stability](#91-why-kernel-apis-change---internal-vs-system-call-stability)
   - 9.2 [Checking Version at Compile Time](#92-checking-version-at-compile-time)
     - 9.2.1 [The KERNEL_VERSION Macro](#921-the-kernel_version-macro)
   - 9.3 [Common Compatibility Issues and Solutions](#93-common-compatibility-issues-and-solutions)
     - 9.3.1 [v5.6+ - proc_ops for procfs](#931-v56---proc_ops-for-procfs)
     - 9.3.2 [v6.4+ - class_create() Signature](#932-v64---class_create-signature)
     - 9.3.3 [v3.14+ - Thread-Safe f_pos](#933-v314---thread-safe-f_pos)
     - 9.3.4 [v5.4+ - New file_operations Fields](#934-v54---new-file_operations-fields)
   - 9.4 [Best Practice for Version-Aware Code](#94-best-practice-for-version-aware-code)
   
---

# Part 1: Foundation and Basics

## 1. Introduction to Character Devices

### 1.1 What Are Character Devices?

Character device drivers are a fundamental part of the Linux kernel that handle I/O operations as a stream of bytes. Unlike block devices (which transfer data in fixed-size blocks), character devices transfer data character-by-character. They expose a file-like interface in user space via device files (typically in `/dev`), allowing applications to use standard system calls (`open()`, `read()`, `write()`, `ioctl()`, etc.) to interact with hardware.

The key concept is abstraction: your application doesn't need to know it's talking to hardware. It opens a file and reads/writes bytes—the driver translates these operations into hardware commands.

### 1.2 The File-Like Interface

Character devices appear as special files in the `/dev` directory. When you call `open("/dev/chardev", O_RDWR)`, the kernel:
1. Resolves the path to a device number (major:minor)
2. Looks up the registered driver for that major number
3. Calls the driver's `.open()` method
4. Returns a file descriptor

From that point on, `read()`, `write()`, `close()` all map to your driver's functions. This makes hardware accessible using familiar file I/O semantics.

### 1.3 Common Examples

- **Serial ports** (`/dev/ttyS0`): Stream of characters from UART hardware
- **Keyboards and mice**: Input events as character streams
- **Custom hardware interfaces**: FPGAs, sensors, motor controllers
- **Virtual devices**: `/dev/null`, `/dev/zero`, `/dev/random`
- **Debug interfaces**: Many drivers create `/dev` entries for runtime control

---

## 2. Core Data Structures

### 2.1 struct file_operations - The Heart of Your Driver

`file_operations` is a function pointer table that maps system calls to your driver implementation. It's how you tell the kernel: "when a user reads from this device, call my_read()".

#### 2.1.1 Essential Fields and Their Purposes

The modern preferred syntax uses designated initializers:

```c
// Modern syntax (recommended)
struct file_operations fops = {
    .read = device_read,
    .write = device_write,
    .open = device_open,
    .release = device_release,
    // Unspecified fields automatically become NULL
};
```

| Field | Purpose | When Called | Critical? |
|-------|---------|-------------|-----------|
| `owner` | Module ownership (usually `THIS_MODULE`) | Prevents module unloading while in use | **YES** |
| `read` | Read data from device | `read()` system call | Yes |
| `write` | Write data to device | `write()` system call | Yes |
| `open` | Initialize device access | `open()` system call | Yes |
| `release` | Cleanup after device close | `close()` system call | Yes |
| `unlocked_ioctl` | Device-specific commands | `ioctl()` system call | Optional |
| `poll` | Check for I/O readiness | `poll()`/`select()` | Optional |
| `mmap` | Memory map device memory | `mmap()` system call | Optional |

**Note**: Since Linux v5.6, `proc_ops` should be used for procfs files instead of `file_operations`.

#### 2.1.2 Modern Designated Initializers

Designated initializers (`.field = value`) are superior to positional initialization because:
- Self-documenting: you see which function maps to which operation
- Resilient to structure changes: if kernel adds new fields, your code still compiles
- Unspecified fields are zeroed automatically

```c
// BAD: Old positional initialization
static struct file_operations fops = {
    NULL,  // llseek
    device_read,  // read
    device_write, // write
    NULL,  // readdir
    NULL,  // poll
    device_ioctl, // ioctl
    NULL,  // mmap
    device_open,  // open
    NULL,  // flush
    device_release, // release
    NULL,  // fsync
    // ... and so on for 20+ fields!
    // If you miscount, you assign wrong function!
};

// GOOD: Modern designated initializers
static struct file_operations fops = {
    .owner = THIS_MODULE,
    .read = device_read,
    .write = device_write,
    .open = device_open,
    .release = device_release,
};
```

#### 2.1.3 The Owner Field - Critical Importance

The `.owner` field is **mandatory** and should always be `THIS_MODULE`. This tells the kernel: "this file_operations belongs to my module". The kernel uses this to increment the module's reference count when the device is in use, preventing `rmmod` from unloading the module while a user has the device open.

```c
// WRONG! Forgetting owner field
static struct file_operations fops = {
    .read = my_read,
};

// CORRECT - Always set owner
static struct file_operations fops = {
    .owner = THIS_MODULE,
    .read = my_read,
};
```

Without `.owner`, if a user opens your device and you try to `rmmod`, the module will be unloaded while still in use, causing an instant kernel panic when the user tries to read/write.

---

### 2.2 struct file (filp) - Per-Open State

`struct file` represents **one open file descriptor** in the kernel. Every successful `open()` creates a new `struct file`. This is different from `struct cdev` which represents the device itself.

#### 2.2.1 The filp Pointer - What It Represents

`filp` (file pointer) is the conventional name for `struct file *` passed to your driver methods. It contains:
- **f_op**: Pointer to your file_operations (for routing system calls)
- **f_pos**: Current read/write position (offset) in the file
- **private_data**: For storing per-open driver-specific data
- **f_flags**: File open flags (O_RDONLY, O_NONBLOCK, etc.)
- **f_mode**: File access mode
- **f_count**: Reference count (how many processes share this fd)

#### 2.2.2 Key Fields: private_data and f_pos

**`private_data`** is a `void *` that you can use to store any per-open state. This is crucial for multi-device drivers (see Section 6.4).

**`f_pos`** tracks the current offset. Since Linux v3.14, updates to `f_pos` are thread-safe (protected by an internal lock), so you don't need manual locking for position updates.

#### 2.2.3 Complete User-Kernel Interaction Flow

```c
// User space program:
int fd = open("/dev/chardev", O_RDWR);  // Step 1
read(fd, buffer, 100);                   // Step 2
close(fd);                               // Step 3

// Kernel space - what happens:

// Step 1: open()
// Kernel: struct file *filp = alloc_file();
//         filp->f_op = &my_cdev.ops;
//         filp->private_data = NULL;
//         my_cdev.ops->open(inode, filp);
//         return file descriptor to user

// Step 2: read()
// Kernel: struct file *filp = fdtable[fd];
//         filp->f_op->read(filp, user_buffer, 100, &filp->f_pos);
//         Copy data to user_buffer
//         Update filp->f_pos
//         Return bytes read

// Step 3: close()
// Kernel: filp->f_op->release(inode, filp);
//         Free struct file
//         Return 0
```

#### 2.2.4 Why struct file is Essential - Without It Would Fail

**Scenario**: Multiple processes open the same device. Without `struct file`, there would be **no per-open state**:

```c
// BAD: Global offset shared by ALL processes
static loff_t global_offset;

static ssize_t device_read(struct file *filp, char __user *buf, size_t len, loff_t *off)
{
    // Process A reads 100 bytes, global_offset = 100
    // Process B reads 50 bytes, but global_offset is still 100!
    // Process B reads from wrong position!
    *off = global_offset;  // WRONG - shared state
}

// GOOD: Each open gets its own f_pos
static ssize_t device_read(struct file *filp, char __user *buf, size_t len, loff_t *off)
{
    // *off is unique to this struct file
    // Process A: *off = 100 (only for Process A's filp)
    // Process B: *off = 0 (only for Process B's filp)
    // No interference!
}
```

**Without `struct file`**: A read by one process would corrupt the position for all other processes. The device would be unusable with multiple applications.

---

## 3. Device Number Management

### 3.1 Major and Minor Numbers Explained

Linux uses a two-level numbering scheme:
- **Major Number**: Identifies the **driver** (e.g., 4 for tty devices)
- **Minor Number**: Identifies a **specific device instance** managed by that driver

Example: `/dev/ttyS0` might be major 4, minor 64, while `/dev/ttyS1` is major 4, minor 65.

**How it works**:
```c
// When you open /dev/chardev:
// 1. Kernel reads the device number from the inode
dev_t dev = inode->i_rdev;  // Returns MKDEV(major, minor)

// 2. Looks up the driver for this major
struct cdev *cdev = cdev_map[major];

// 3. Uses cdev->ops to route system calls
```

### 3.2 Static vs Dynamic Allocation - Trade-offs

**Static Allocation**: You choose a fixed major number
```c
// Pros: Predictable device file names
// Cons: Risk of conflicts, requires checking Documentation/admin-guide/devices.txt
int register_chrdev_region(MKDEV(240, 0), 1, "mydev");
```

**Dynamic Allocation**: Let the kernel choose
```c
// Pros: No conflicts, recommended for most drivers
// Cons: Must create device file after loading module
int alloc_chrdev_region(&dev, 0, 1, "mydev");
major = MAJOR(dev);
```

**Always use dynamic allocation** for new drivers to avoid conflicts with other drivers.

### 3.3 dev_t Type and Manipulation Macros

`dev_t` is a 32-bit integer that encodes both major and minor numbers:

```c
typedef u32 __kernel_dev_t;
typedef __kernel_dev_t dev_t;

// Macros to manipulate dev_t:
MAJOR(dev_t dev);      // Extracts major number (bits 20-31)
MINOR(dev_t dev);      // Extracts minor number (bits 0-19)
MKDEV(int major, int minor);  // Combines into dev_t

// Example:
dev_t dev = MKDEV(240, 5);
printk("Major: %d, Minor: %d\n", MAJOR(dev), MINOR(dev));
// Output: Major: 240, Minor: 5
```

**Bit Layout** (since Linux 2.6):
```
bits [31:20] = major number (12 bits, range 0-4095)
bits [19:0]  = minor number (20 bits, range 0-1048575)
```

### 3.4 Checking Allocated Numbers

To see which major numbers are already in use:

```bash
# List all allocated character devices
$ cat /proc/devices
Character devices:
  1 mem
  4 tty
  5 /dev/tty
  5 /dev/console
  5 /dev/ptmx
  7 vcs
 10 misc
 13 input
 21 sg
 29 fb
...
 240 mydev  # Your driver appears here after insmod

# Create device file manually (if not using class_create)
$ sudo mknod /dev/mydevice c 240 0
```

---

## 4. Legacy Device Registration

### 4.1 register_chrdev() - The Old Way

```c
int register_chrdev(unsigned int major, const char *name, 
                    struct file_operations *fops);

// Usage:
ret = register_chrdev(0, "mydev", &fops);  // 0 = dynamic major
if (ret > 0) major = ret;  // Save assigned major
```

#### 4.1.1 Characteristics and Usage

**Characteristics**:
- Registers a **single** major number
- Occupies **all 256 minor numbers** (wasteful!)
- Simple but deprecated for new drivers
- Returns negative on failure, or allocated major if `major=0`

#### 4.1.2 The Fatal Flaws

**Three major problems**:
1. **Wastes minors**: You get all 256 minors, even if you only need 1
2. **No class integration**: No automatic `/dev` creation, no sysfs
3. **No scalability**: Can't register multiple devices with different majors

**Example of waste**:
```c
// You only need one device:
register_chrdev(0, "mydev", &fops);  // But you get minors 0-255!

// Kernel internally:
chrdevs[major].range = 256;  // All minors reserved for you!
// No other driver can use any minor under this major
```

**Modern kernels discourage this**: While still supported, `register_chrdev()` is considered legacy and should not be used in new code.

---

# Part 2: Modern Device Registration

## 5. Modern Device Registration (cdev Interface)

The `cdev` (character device) interface provides fine-grained control over device registration. It's the standard for all modern kernel development.

### 5.1 Two-Step Process Overview

**Step 1**: Allocate device numbers (choose major/minor)
**Step 2**: Initialize and register a `struct cdev`

```c
// Modern registration pattern:
dev_t dev_num;
struct cdev my_cdev;

// Step 1: Get device numbers
alloc_chrdev_region(&dev_num, 0, 1, "mydev");

// Step 2: Initialize cdev
cdev_init(&my_cdev, &fops);
my_cdev.owner = THIS_MODULE;

// Step 3: Register with kernel
cdev_add(&my_cdev, dev_num, 1);
```

### 5.2 Step 1: Allocating Device Numbers

#### 5.2.1 Static Allocation - register_chrdev_region()

```c
int register_chrdev_region(dev_t from, unsigned count, const char *name);

// Example: Request major 240, minor 0-2
dev_t dev = MKDEV(240, 0);
ret = register_chrdev_region(dev, 3, "mydev");
// Now you own minors 0, 1, 2 under major 240
```

**When to use**: When you need a **predictable major number** for compatibility with hardcoded applications.

**Danger**: You must check `Documentation/admin-guide/devices.txt` to avoid conflicts. If another driver already uses major 240, your `insmod` will fail.

#### 5.2.2 Dynamic Allocation - alloc_chrdev_region() (RECOMMENDED)

```c
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, 
                        unsigned count, const char *name);

// Example:
dev_t dev_num;
ret = alloc_chrdev_region(&dev_num, 0, 1, "mydev");
if (ret == 0) {
    major = MAJOR(dev_num);  // Kernel-assigned major
    minor = MINOR(dev_num);  // Your requested baseminor
}
```

**Advantages**:
- No conflicts guaranteed
- No need to check device.txt
- Recommended for all new drivers

**Minor inconvenience**: You must create `/dev` file after loading (or use `class_create()` to automate it).

### 5.3 Step 2: cdev Initialization

#### 5.3.1 Understanding cdev_alloc()

`cdev_alloc()` is a convenience function that allocates and initializes a cdev:

```c
struct cdev *cdev_alloc(void);

// What it does internally:
struct cdev *p = kzalloc(sizeof(*p), GFP_KERNEL);
if (p) {
    p->kobj.ktype = &ktype_cdev_dynamic;
    INIT_LIST_HEAD(&p->list);
    // NOTE: Does NOT set p->ops!
}
return p;
```

**Use when**: You need **dynamic allocation** and don't want to embed cdev in your own struct (rare).

**Example**:
```c
struct cdev *my_cdev = cdev_alloc();
if (!my_cdev) return -ENOMEM;
my_cdev->ops = &fops;
my_cdev->owner = THIS_MODULE;
```

**Memory management**: `cdev_alloc()` allocates memory. You must free it with `kobject_put(&cdev->kobj)` or let `cdev_del()` do it.

#### 5.3.2 Understanding cdev_init()

`cdev_init()` initializes a **pre-allocated** cdev:

```c
void cdev_init(struct cdev *cdev, const struct file_operations *fops);

// What it does:
memset(cdev, 0, sizeof *cdev);      // Zero memory
INIT_LIST_HEAD(&cdev->list);        // Initialize list
cdev->kobj.ktype = &ktype_cdev_default;
cdev->ops = fops;                   // Attach fops
```

**Use when**: You have **statically allocated** or **kmalloc'd** a cdev and need to initialize it.

**Example**:
```c
static struct cdev my_cdev;  // Static allocation
cdev_init(&my_cdev, &fops);
my_cdev.owner = THIS_MODULE;
```

**Or with kmalloc**:
```c
struct cdev *my_cdev = kmalloc(sizeof(*my_cdev), GFP_KERNEL);
cdev_init(my_cdev, &fops);
```

#### 5.3.3 cdev_alloc() vs cdev_init() - The ONLY Difference

| Feature | cdev_alloc() | cdev_init() |
|---------|--------------|-------------|
| **Allocates memory** | Yes (calls kzalloc) | No, you allocate |
| **Sets fops** | No, you must set manually | Yes, parameter sets it |
| **Use case** | Dynamic, unknown count | Static or kmalloc'd |
| **Cleanup** | cdev_del() or kobject_put() | kfree() if kmalloc'd + cdev_del() |

**Key point**: Use **one or the other**, never both. For most drivers, **embed cdev in your struct + cdev_init()** is the professional pattern.

### 5.4 Step 3: Registering the Device

#### 5.4.1 cdev_add() Parameters and Validation

```c
int cdev_add(struct cdev *p, dev_t dev, unsigned count);

// Parameters:
//   p: Pointer to initialized cdev
//   dev: First device number (MKDEV(major, baseminor))
//   count: Number of sequential minor numbers to register
```

**What kernel validates**:
1. Is this `dev_t` range already registered? (returns `-EBUSY` if yes)
2. Is count reasonable? (must be ≤ 256 minors per major)
3. Does cdev->ops have minimum required functions? (warns if open/release missing)

**Returns**: 0 on success, negative errno on failure.

#### 5.4.2 What Happens Internally During cdev_add()

```c
// Simplified cdev_add() implementation:
int cdev_add(struct cdev *p, dev_t dev, unsigned count)
{
    // 1. Store device numbers in cdev
    p->dev = dev;
    p->count = count;
    
    // 2. Add to global hash table
    //    cdev_map[MAJOR(dev)] = p
    kobj_map(cdev_map, dev, count, NULL, exact_match, exact_lock, p);
    
    // 3. Initialize and activate kobject
    kobject_init(&p->kobj, &ktype_cdev_default);
    kobject_add(&p->kobj, NULL, "cdev-%d:%d", MAJOR(dev), MINOR(dev));
    
    // 4. Create sysfs entry
    //    /sys/dev/char/MAJOR:MINOR appears
    
    // 5. Send uevent (if parent kobject exists)
    //    kobject_uevent(&p->kobj, KOBJ_ADD);
    
    return 0;
}
```

**Critical point**: After `cdev_add()`, your device is **live**. Any `open()` call on your major/minor will invoke your `fops` immediately.

### 5.5 The Global Device Hash Table

The kernel maintains a hash table `cdev_map` that maps major numbers to cdev structures:

```c
// fs/char_dev.c
static struct kobj_map *cdev_map;

// cdev_add() inserts:
// cdev_map[major] → your_cdev

// open() path looks up:
struct kobject *kobj = kobj_lookup(cdev_map, inode->i_rdev, &idx);
struct cdev *cdev = container_of(kobj, struct cdev, kobj);
inode->i_cdev = cdev;  // Cache for next open
```

**Why a hash table? ** O(1) lookup. Kernel doesn't search a list—it directly indexes by major.

### 5.6 Complete Modern Registration Example

```c
// drivers/char/modern_cdev.c
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/module.h>
#include <linux/slab.h>

static int major;
static struct class *my_class;
static struct cdev my_cdev;

static int device_open(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Device opened\n");
    return 0;
}

static int device_release(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Device closed\n");
    return 0;
}

static ssize_t device_read(struct file *filp, char __user *buf, 
                           size_t len, loff_t *off)
{
    char *msg = "Hello from modern driver!\n";
    return simple_read_from_buffer(buf, len, off, msg, strlen(msg));
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = device_open,
    .release = device_release,
    .read = device_read,
};

static int __init modern_init(void)
{
    dev_t dev_num;
    int ret;
    
    // Step 1: Allocate device numbers
    ret = alloc_chrdev_region(&dev_num, 0, 1, "modern_cdev");
    if (ret < 0) return ret;
    major = MAJOR(dev_num);
    
    // Step 2: Initialize cdev
    cdev_init(&my_cdev, &fops);
    
    // Step 3: Register device
    ret = cdev_add(&my_cdev, dev_num, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, 1);
        return ret;
    }
    
    // Step 4: Create class (for udev)
    my_class = class_create(THIS_MODULE, "modern_cdev");
    if (IS_ERR(my_class)) {
        ret = PTR_ERR(my_class);
        cdev_del(&my_cdev);
        unregister_chrdev_region(dev_num, 1);
        return ret;
    }
    
    // Step 5: Create device (auto /dev creation)
    device_create(my_class, NULL, dev_num, NULL, "modern_cdev");
    
    printk(KERN_INFO "modern_cdev: loaded with major %d\n", major);
    return 0;
}

static void __exit modern_exit(void)
{
    device_destroy(my_class, MKDEV(major, 0));
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(MKDEV(major, 0), 1);
    printk(KERN_INFO "modern_cdev: unloaded\n");
}

module_init(modern_init);
module_exit(modern_exit);
MODULE_LICENSE("GPL");

// ============================================================
// TESTING:
// ============================================================
$ make && sudo insmod modern_cdev.ko
$ dmesg | tail
modern_cdev: loaded with major 240

$ ls -l /dev/modern_cdev  # Automatically created by udev!
crw-rw---- 1 root root 240, 0 Nov 22 10:00 /dev/modern_cdev

$ cat /dev/modern_cdev
Hello from modern driver!

$ sudo rmmod modern_cdev
$ ls /dev/modern_cdev  # Automatically removed!
ls: cannot access '/dev/modern_cdev': No such file or directory
```

---

# Part 3: Memory Allocation Patterns

## 6. Memory Allocation Patterns - The Definitive Guide

### 6.1 Three Ways to Allocate Memory

| Method | When to Use | Lifetime | Scope | Error Handling |
|--------|-------------|----------|-------|----------------|
| **Static** (`static struct X x`) | Single device, simple drivers | Module lifetime | Global | None (always succeeds) |
| **Stack** (`struct X x;` in function) | Temporary objects | Function lifetime | Local | None (always succeeds) |
| **Dynamic** (`kmalloc()` / `cdev_alloc()`) | Multiple devices, complex state | Until `kfree()` | Heap | Must check for NULL |

**Critical rule**: Never allocate cdev on stack. Stack variables disappear when function returns, but cdev must persist for module lifetime.

### 6.2 Static Allocation - When and How

Static allocation is the simplest pattern for **single-device drivers** with **no hardware state**.

#### 6.2.1 Simple Static Character Device - Full Example

```c
// drivers/char/simple_cdev.c
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/module.h>

static struct cdev my_cdev;          // Statically allocated
static dev_t dev_num;                // Statically allocated
static int major;                    // Statically allocated

static int simple_open(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Simple device opened\n");
    return 0;
}

static ssize_t simple_read(struct file *filp, char __user *buf, 
                           size_t len, loff_t *off)
{
    char *msg = "Hello from simple device!\n";
    return simple_read_from_buffer(buf, len, off, msg, strlen(msg));
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = simple_open,
    .read = simple_read,
};

static int __init simple_init(void)
{
    int ret;
    
    // Allocate major/minor
    ret = alloc_chrdev_region(&dev_num, 0, 1, "simple_cdev");
    if (ret < 0) return ret;
    major = MAJOR(dev_num);
    
    // Initialize static cdev
    cdev_init(&my_cdev, &fops);
    
    // Register
    ret = cdev_add(&my_cdev, dev_num, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, 1);
        return ret;
    }
    
    printk(KERN_INFO "simple_cdev: mknod /dev/simple_cdev c %d 0\n", major);
    return 0;
}

static void __exit simple_exit(void)
{
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
}

module_init(simple_init);
module_exit(simple_exit);
MODULE_LICENSE("GPL");
```

**Key characteristics**:
- No `kmalloc` anywhere
- cdev exists for entire module lifetime
- No per-device state (no IRQs, no buffers, no hardware regs)
- Manual `/dev` file creation required (no class_create)
- **Use when**: Learning, debugging, virtual devices with no state

### 6.3 Dynamic Allocation with kmalloc

Use `kmalloc` when you need to allocate **at runtime** based on configuration or when you have **multiple devices**.

```c
struct my_device *dev = kmalloc(sizeof(*dev), GFP_KERNEL);
if (!dev) return -ENOMEM;

// Initialize embedded cdev
cdev_init(&dev->cdev, &fops);
```

**Lifetime control**: You decide when to free with `kfree()`.

### 6.4 The Embedding Pattern - Professional Method

This is the **recommended pattern** for real drivers. You embed `struct cdev` inside your own device structure.

#### 6.4.1 Why Embed cdev in Your Struct?

**Problem with separate cdev**: No easy way to get from cdev to your device state.

```c
// BAD: Separate structures
struct cdev my_cdev;
struct my_device {
    void __iomem *regs;
    int irq;
} my_dev;

// In open():
static int device_open(struct inode *inode, struct file *filp)
{
    // inode->i_cdev points to my_cdev
    // How do you get to my_dev??? No link!
    // Must use global variable: extern struct my_device my_dev;
}
```

**Solution: Embedding **:
```c
// GOOD: Embedded cdev
struct my_device {
    struct cdev cdev;              // Must be first (or use container_of)
    void __iomem *regs;            // Hardware registers
    int irq;                       // IRQ number
    spinlock_t lock;               // Device lock
    char *buffer;                  // Device buffer
};

// In open():
static int device_open(struct inode *inode, struct file *filp)
{
    // Get cdev from inode
    struct cdev *cdev = inode->i_cdev;
    
    // Get my_device from cdev
    struct my_device *dev = container_of(cdev, struct my_device, cdev);
    
    // Store in private_data for other ops
    filp->private_data = dev;
    
    // Now access all fields
    printk("Device IRQ: %d\n", dev->irq);
    return 0;
}
```

#### 6.4.2 Complete Multi-Device Driver Example

```c
// drivers/char/professional.c
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/spinlock.h>

// YOUR device structure
struct prof_device {
    struct cdev cdev;              // Embedd cdev HERE
    int device_id;                 // Custom: device number
    char *buffer;                  // Custom: kmalloc'd buffer
    size_t buffer_size;            // Custom: buffer size
    atomic_t is_open;              // Custom: open flag
    spinlock_t lock;               // Custom: device lock
};

static dev_t first_dev;            // First device number
static struct class *prof_class;
#define DEVICE_COUNT 3             // Support 3 devices

static int prof_open(struct inode *inode, struct file *filp)
{
    struct prof_device *dev = container_of(inode->i_cdev, 
                                           struct prof_device, 
                                           cdev);
    
    if (atomic_cmpxchg(&dev->is_open, 0, 1)) {
        return -EBUSY;  // Already open
    }
    
    filp->private_data = dev;
    printk(KERN_INFO "prof_dev%d: opened\n", dev->device_id);
    return 0;
}

static int prof_release(struct inode *inode, struct file *filp)
{
    struct prof_device *dev = filp->private_data;
    atomic_set(&dev->is_open, 0);
    printk(KERN_INFO "prof_dev%d: closed\n", dev->device_id);
    return 0;
}

static ssize_t prof_read(struct file *filp, char __user *buf, 
                         size_t len, loff_t *off)
{
    struct prof_device *dev = filp->private_data;
    ssize_t ret;
    
    spin_lock(&dev->lock);
    ret = simple_read_from_buffer(buf, len, off, 
                                  dev->buffer, dev->buffer_size);
    spin_unlock(&dev->lock);
    
    return ret;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = prof_open,
    .release = prof_release,
    .read = prof_read,
};

static struct prof_device *devices[DEVICE_COUNT];  // Array of pointers

static int __init prof_init(void)
{
    int ret, i;
    
    // Allocate a range of device numbers
    ret = alloc_chrdev_region(&first_dev, 0, DEVICE_COUNT, "prof_dev");
    if (ret < 0) return ret;
    
    // Create class
    prof_class = class_create(THIS_MODULE, "prof_dev");
    if (IS_ERR(prof_class)) {
        ret = PTR_ERR(prof_class);
        goto fail_class;
    }
    
    // Allocate and initialize EACH device
    for (i = 0; i < DEVICE_COUNT; i++) {
        // Allocate memory for prof_device + embedded cdev
        devices[i] = kzalloc(sizeof(*devices[i]), GFP_KERNEL);
        if (!devices[i]) {
            ret = -ENOMEM;
            goto fail_alloc;
        }
        
        // Initialize YOUR fields
        devices[i]->device_id = i;
        devices[i]->buffer_size = 1024 * (i + 1);
        devices[i]->buffer = kmalloc(devices[i]->buffer_size, GFP_KERNEL);
        if (!devices[i]->buffer) {
            ret = -ENOMEM;
            goto fail_buf;
        }
        snprintf(devices[i]->buffer, devices[i]->buffer_size, 
                 "Data from device %d\n", i);
        atomic_set(&devices[i]->is_open, 0);
        spin_lock_init(&devices[i]->lock);
        
        // Initialize cdev (inside prof_device)
        cdev_init(&devices[i]->cdev, &fops);
        devices[i]->cdev.owner = THIS_MODULE;
        
        // Compute device number for this minor
        dev_t this_dev = MKDEV(MAJOR(first_dev), MINOR(first_dev) + i);
        
        // Register this device's cdev
        ret = cdev_add(&devices[i]->cdev, this_dev, 1);
        if (ret < 0) {
            kfree(devices[i]->buffer);
            kfree(devices[i]);
            goto fail_cdev;
        }
        
        // Create device file
        device_create(prof_class, NULL, this_dev, NULL, "prof_dev%d", i);
        
        printk(KERN_INFO "prof_dev%d: created with minor %d\n", 
               i, MINOR(this_dev));
    }
    
    return 0;
    
// Error handling (critical for real drivers!)
fail_cdev:
    for (i--; i >= 0; i--) {
        dev_t this_dev = MKDEV(MAJOR(first_dev), MINOR(first_dev) + i);
        device_destroy(prof_class, this_dev);
        cdev_del(&devices[i]->cdev);
        kfree(devices[i]->buffer);
        kfree(devices[i]);
    }
fail_buf:
    for (; i >= 0; i--) {
        if (devices[i]) {
            kfree(devices[i]->buffer);
            kfree(devices[i]);
        }
    }
fail_alloc:
    class_destroy(prof_class);
fail_class:
    unregister_chrdev_region(first_dev, DEVICE_COUNT);
    return ret;
}

static void __exit prof_exit(void)
{
    int i;
    
    for (i = 0; i < DEVICE_COUNT; i++) {
        dev_t this_dev = MKDEV(MAJOR(first_dev), MINOR(first_dev) + i);
        
        device_destroy(prof_class, this_dev);
        cdev_del(&devices[i]->cdev);
        kfree(devices[i]->buffer);
        kfree(devices[i]);
    }
    
    class_destroy(prof_class);
    unregister_chrdev_region(first_dev, DEVICE_COUNT);
}

module_init(prof_init);
module_exit(prof_exit);
MODULE_LICENSE("GPL");

// ============================================================
// TESTING:
// ============================================================
$ sudo insmod professional.ko
prof_dev0: created with minor 0
prof_dev1: created with minor 1
prof_dev2: created with minor 2

$ ls -l /dev/prof_dev*
crw-rw---- 1 root root 240, 0 Nov 22 10:30 /dev/prof_dev0
crw-rw---- 1 root root 240, 1 Nov 22 10:30 /dev/prof_dev1
crw-rw---- 1 root root 240, 2 Nov 22 10:30 /dev/prof_dev2

$ cat /dev/prof_dev0
Data from device 0

$ cat /dev/prof_dev1
Data from device 1

$ cat /dev/prof_dev2
Data from device 2

$ sudo rmmod professional.ko
# All /dev files automatically removed
```

#### 6.4.3 Error Handling in Multi-Device Drivers

**Critical**: In multi-device drivers, you must clean up **partial allocations** on error. Use `goto` chains to unwind in reverse order.

```c
// Pattern:
for (i = 0; i < count; i++) {
    ret = allocate_device(i);
    if (ret) goto error;
}

return 0;

error:
    // Clean up devices 0..i-1
    for (i--; i >= 0; i--) {
        cleanup_device(i);
    }
    return ret;
```

**Why this matters**: If device 2 fails to register, you must free devices 0 and 1 or they leak memory.

### 6.5 Managed Allocations (devm_) - Modern Approach

`devm_` functions are **device-managed resources**. They automatically free when the device is removed.

#### 6.5.1 When to Use devm_ Functions

Use `devm_` when writing drivers for **hot-pluggable devices** (USB, PCI, platform devices):

```c
// drivers/usb/example.c
static int usb_probe(struct usb_interface *intf, 
                     const struct usb_device_id *id)
{
    struct device *dev = &intf->dev;  // Managed device
    
    // All allocations tied to dev lifecycle
    struct my_dev *my = devm_kzalloc(dev, sizeof(*my), GFP_KERNEL);
    devm_alloc_chrdev_region(dev, &dev_num, 0, 1, "usbdev");
    devm_cdev_add(dev, &my->cdev, dev_num, 1);
    
    // No __exit cleanup needed!
    return 0;
}

static void usb_disconnect(struct usb_interface *intf)
{
    // Nothing to free - devm_ handles it all
}
```

#### 6.5.2 Complete USB Probe Example

```c
struct usb_gadget {
    struct cdev cdev;
    struct usb_device *udev;
    char *buffer;
};

static int usb_probe(struct usb_interface *intf, 
                     const struct usb_device_id *id)
{
    struct usb_gadget *gadget;
    dev_t dev_num;
    
    // 1. Managed allocation (freed on disconnect)
    gadget = devm_kzalloc(&intf->dev, sizeof(*gadget), GFP_KERNEL);
    if (!gadget) return -ENOMEM;
    
    // 2. Managed device numbers
    devm_alloc_chrdev_region(&intf->dev, &dev_num, 0, 1, "usb_gadget");
    
    // 3. Initialize cdev
    cdev_init(&gadget->cdev, &gadget_fops);
    gadget->cdev.owner = THIS_MODULE;
    
    // 4. Managed cdev_add (auto unregistered on disconnect)
    devm_cdev_add(&intf->dev, &gadget->cdev, dev_num, 1);
    
    // 5. Create device (also auto destroyed)
    device_create(gadget_class, NULL, dev_num, NULL, "usb_gadget");
    
    return 0;
    // NO __exit NEEDED! Everything auto-freed when USB disconnects
}
```

**Advantage**: No manual cleanup. Resources are freed in reverse order when device is unbound.

### 6.6 cdev_alloc() When and Why - Rare Cases

`cdev_alloc()` is for **variable-count dynamic scenarios** where you don't know how many devices at compile time.

#### 6.6.1 USB Gadget Example - Unknown Device Count

```c
/**
 * USB Gadget Example: You don't know how many devices will be plugged in
 * Each USB device needs its own cdev, but you can't allocate static cdevs 
 * for "maybe 10 devices"
 */

struct usb_gadget {
    struct cdev *cdev;  // MUST be pointer, allocated per device
    struct usb_device *udev;
    char name[20];
};

// Called when user plugs in USB device
static int usb_gadget_probe(struct usb_device *udev)
{
    struct usb_gadget *gadget;
    dev_t dev_num;
    int ret;
    
    // Allocate gadget struct
    gadget = kzalloc(sizeof(*gadget), GFP_KERNEL);
    if (!gadget) return -ENOMEM;
    
    // Allocate cdev dynamically (don't know how many at compile time)
    gadget->cdev = cdev_alloc();
    if (!gadget->cdev) {
        kfree(gadget);
        return -ENOMEM;
    }
    
    // Set up cdev
    gadget->cdev->ops = &gadget_fops;
    gadget->cdev->owner = THIS_MODULE;
    
    // Get device number (could be different for each device)
    ret = alloc_chrdev_region(&dev_num, 0, 1, gadget->name);
    if (ret < 0) {
        kobject_put(&gadget->cdev->kobj);  // Free cdev
        kfree(gadget);
        return ret;
    }
    
    // Register
    ret = cdev_add(gadget->cdev, dev_num, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, 1);
        kobject_put(&gadget->cdev->kobj);
        kfree(gadget);
        return ret;
    }
    
    // Store for disconnect
    usb_set_gadget_data(udev, gadget);
    
    printk(KERN_INFO "USB gadget %s registered\n", gadget->name);
    return 0;
}

// Called when user unplugs USB device
static void usb_gadget_disconnect(struct usb_device *udev)
{
    struct usb_gadget *gadget = usb_get_gadget_data(udev);
    
    // Cleanup
    cdev_del(gadget->cdev);  // Also does kobject_put()
    unregister_chrdev_region(gadget->cdev->dev, 1);
    
    // Free our struct
    kfree(gadget);
    
    printk(KERN_INFO "USB gadget disconnected\n");
}
```

**Key insight**: `cdev_alloc()` is for **dynamic, variable-count scenarios**. If you know you have exactly 1 device, static is better.

### 6.7 The Complete Decision Tree


```mermaid
graph TD
    A[Start: Writing a driver] --> B{Do you need per-device state?};
    B -->|"No (simple virtual dev)"| C[Use static cdev];
    C --> D[static struct cdev my_cdev; cdev_init(); cdev_add();];
    
    B -->|"Yes (IRQs, buffers, etc.)"| E{How many devices?};
    E -->|"Exactly 1 device"| F[Static struct + embed];
    F --> G[static struct my_device my_dev; cdev_init(&my_dev.cdev);];
    
    E -->|"Multiple/unknown count"| H[Dynamic allocation];
    H --> I{Can use devm_?};
    I -->|"Yes (PCI/USB)"| J[Use devm_kzalloc + devm_cdev_add];
    I -->|"No (legacy)"| K[Use kzalloc + manual cleanup];
    
    K --> L[struct my_device *dev = kzalloc(...); cdev_init(&dev->cdev);];
    
    M[Rare: Only need cdev, no state] --> N[cdev_alloc];
    N --> O[struct cdev *cdev = cdev_alloc(); cdev->ops = &fops;];
```

### Quick Reference Table

| Pattern | `kmalloc`? | `cdev_alloc`? | Custom Struct? | Use Case |
|---------|------------|---------------|----------------|----------|
| **Single virtual device, no state** | ❌ No | ❌ No | ❌ No | `/dev/null` style |
| **Single device, with state** | ❌ No | ❌ No | ✅ Yes | Simple hardware |
| **Multiple devices** | ✅ Yes | ❌ No | ✅ Yes | Multi-port serial |
| **Unknown device count** | ✅ Yes | ✅ Yes | ✅ Yes | USB gadgets |
| **Hot-plug devices** | ✅ Yes (devm_) | ❌ No | ✅ Yes | PCI, USB |

**Bottom Line**: For **real drivers**, use **Embed + `kmalloc/devm_kzalloc`**. It's the pattern in 99% of kernel drivers.

---

# Part 4: Concurrency and Synchronization

## 7. Concurrency and Thread Safety

### 7.1 The Race Condition Problem

#### 7.1.1 Real-World Scenarios

**Scenario 1**: Two processes open device simultaneously

```c
static int already_open = 0;

static int device_open(struct inode *inode, struct file *filp)
{
    if (already_open) return -EBUSY;  // Check
    
    // RACE HERE: Process A checks, Process B checks
    // Both see already_open == 0
    // Both proceed!
    
    already_open = 1;  // Both set it, corrupting state
    return 0;  // Device opened twice - disaster!
}
```

**Scenario 2**: Counter increment lost

```c
static int counter = 0;

static ssize_t device_read(..., loff_t *off)
{
    // Process A: reads counter (0)
    // Process B: reads counter (0)
    // Process A: counter++ (now 1)
    // Process B: counter++ (now 1, but should be 2!)
    // One increment lost due to race
    sprintf(msg, "Count: %d\n", counter++);
    // Result: counter is 1, not 2
}
```

**Scenario 3**: Buffer corruption

```c
static char global_buffer[100];

static ssize_t device_write(..., char __user *buf, size_t len)
{
    // Process A copies 50 bytes
    copy_from_user(global_buffer, buf, len);
    // Process B copies 50 bytes (overwrites A's data)
    // Process A thinks its data is still there - corruption!
}
```

**Result of races**: Data corruption, system crashes, undefined behavior.

### 7.2 Atomic Operations (atomic_t) - Lock-Free

`atomic_t` provides **lock-free** operations on integers. Perfect for simple flags and counters.

#### 7.2.1 What is atomic_t and Why a Struct?

```c
typedef struct {
    int counter;
} atomic_t;

// Why a struct? Type safety!
atomic_t already_open;
// already_open.counter = 1;  // ❌ Compile error
atomic_set(&already_open, 1);  // ✅ Must use API
```

**Why not just `int`? ** The kernel ** forces** you to use atomic operations by making the raw `int` inaccessible. This prevents accidental non-atomic access.

#### 7.2.2 ATOMIC_INIT() - Compile-Time Initialization

```c
static atomic_t already_open = ATOMIC_INIT(0);  // Sets counter = 0

// What it expands to:
static atomic_t already_open = { .counter = 0 };

// Advantage: Zero overhead at module load
```

#### 7.2.3 atomic_cmpxchg() - Line-by-Line Breakdown

```c
static atomic_t already_open = ATOMIC_INIT(0);  // 0 = free, 1 = busy

static int device_open(struct inode *inode, struct file *filp)
{
    // Try to change 0 → 1, but only if currently 0
    int old = atomic_cmpxchg(&already_open, 0, 1);
    
    // If old wasn't 0, someone beat us to it
    if (old != 0) {
        return -EBUSY;  // "Device or resource busy"
    }
    
    // Success! We changed it from 0 → 1
    return 0;
}
```

** Logic **:
```c
// atomic_cmpxchg(&already_open, 0, 1) does:
if (*already_open == 0) {
    *already_open = 1;
    return 0;  // Old value
} else {
    return *already_open;  // Non-zero old value
}
```

**Real-world analogy**: Bathroom stall lock (0=unlocked, 1=locked). `cmpxchg` tries to lock it, but only if it's currently unlocked. If someone already locked it, you wait (or return -EBUSY).

#### 7.2.4 Assembly-Level Operation

```c
// atomic_cmpxchg(&already_open, 0, 1) on x86-64:
mov    eax, 1          // Desired value (1 = locked)
mov    edx, 0          // Expected value (0 = unlocked)
lock cmpxchg [already_open], eax  // Atomic operation
// If [already_open] == edx (0), set it to eax (1)
// Else, fail and return old value
```

**The `lock` prefix**: Locks the **cache line**, not the entire system bus. Very efficient on modern CPUs.

#### 7.2.5 Advantages Over Mutexes

| Feature | atomic_t | mutex_t |
|---------|----------|---------|
| **Can sleep** | ❌ No (interrupt context ok) | ✅ Yes (process only) |
| **Overhead** | Very low (single instruction) | Higher (lock/unlock) |
| **Use case** | Flags, counters, simple state | Complex data structures |
| **Contention** | Lock-free, no waiting | May sleep if contended |

**Rule**: Use `atomic_t` for simple operations. Use mutexes for protecting complex data structures or critical sections that take time.

### 7.3 Other Synchronization Primitives

#### 7.3.1 Mutexes - For Complex Critical Sections

```c
static DEFINE_MUTEX(my_mutex);

void complex_operation(void)
{
    mutex_lock(&my_mutex);
    // Critical section:
    // - Modify linked lists
    // - Update multiple fields
    // - Allocate memory
    // CAN sleep here (kmalloc, etc.)
    mutex_unlock(&my_mutex);
}
```

#### 7.3.2 Spinlocks - For Short Atomic Contexts

```c
static DEFINE_SPINLOCK(my_lock);
unsigned long flags;

void update_hardware(void)
{
    // MUST be short! Cannot sleep!
    spin_lock_irqsave(&my_lock, flags);  // Disables interrupts
    // Critical section: few instructions only
    // NO kmalloc, NO copy_to_user, etc.
    writel(reg_value, dev->regs + REG_OFFSET);
    spin_unlock_irqrestore(&my_lock, flags);
}
```

**Important**: Never sleep with a spinlock held! If you do, the system deadlocks.

#### 7.3.3 Read-Write Semaphores - For Read-Heavy Workloads

```c
static DEFINE_RWLOCK(my_rwlock);

void read_data(void)
{
    read_lock(&my_rwlock);  // Multiple readers allowed
    // Read-only access
    read_unlock(&my_rwlock);
}

void modify_data(void)
{
    write_lock(&my_rwlock);  // Exclusive write access
    // Read and modify
    write_unlock(&my_rwlock);
}
```

**Use when**: You have many more reads than writes, and reads don't block each other.

### 7.4 The Golden Rule of Synchronization

**Always prefer the simplest primitive that works**:

1. **Atomic operations** (atomic_t) - for flags, counters
2. **Mutexes** - for most everything else
3. **Spinlocks** - only for very short, atomic contexts where you cannot sleep
4. **RW locks** - for read-heavy workloads with occasional writes

The example driver uses `atomic_t` because the operation (exclusive open) is very simple—just a flag check. For protecting a buffer, a mutex would be more appropriate.

---

# Part 5: User-Kernel Data Transfer

## 8. User-Kernel Data Transfer

### 8.1 The User Pointer Problem - Why You Can't Dereference Directly

User-space memory is:
- In a **different address space** (not directly accessible from kernel)
- May be **swapped out** (page fault if accessed)
- May be **invalid** (malicious user can crash kernel)
- Must respect **permissions** (read-only pages)

**Never do this**:
```c
// CRASHES KERNEL!
char *user_buf = (char *)buf;
*user_buf = 'A';  // If buf is invalid → kernel oops
```

### 8.2 put_user() and get_user() - Single Value Transfer

For transferring single values (int, char, long):

```c
int value = 42;
int ret;

// Copy TO user space
ret = put_user(value, (int __user *)user_ptr);
// Returns: 0 on success, -EFAULT on failure

// Copy FROM user space
ret = get_user(value, (int __user *)user_ptr);
```

#### 8.2.1 What They Do Internally

```c
// put_user(value, user_ptr) performs:
1. Verify user_ptr is in user address range
2. Check write permissions (for put_user)
3. Disable page fault handling temporarily
4. Perform single-instruction copy
5. Re-enable page faults
6. Return error code if failed

// Why disable page faults?
// If the user pointer is invalid, kernel would panic trying to handle page fault.
// Instead, put_user() catches the fault and returns -EFAULT.
```

### 8.3 copy_to_user() and copy_from_user() - Buffer Transfer

For transferring buffers/arrays:

```c
// Copy TO user space
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);

// Copy FROM user space
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n);

// Returns: Number of bytes NOT copied (0 on success)
```

**Example from chardev.c**:
```c
// BAD (no bounds checking)
put_user(*(msg_ptr++), buffer++);

// GOOD (with bounds checking)
while (length && *msg_ptr) {
    if (put_user(*(msg_ptr++), buffer++))
        return -EFAULT;  // User buffer fault
    length--;
    bytes_read++;
}
```

### 8.4 Page Faults - What They Are and Why They Matter

A **page fault** occurs when the CPU tries to access memory that's not currently mapped:

```c
// User buffer: 0x7fff12345000
// That page might be:
// - Swapped to disk (major fault: slow, loads from disk)
// - Not allocated yet (minor fault: fast, allocates page)
// - Read-only (fault: segfault if writing)

// In kernel:
put_user(data, buffer);  // If buffer points to unmapped page → safe error
```

#### 8.4.1 User-Space Pages vs Kernel-Space Pages

**User-space memory**:
- Can be swapped out
- Has different permissions per page
- Must be accessed through copy functions
- Addresses above 0x00007fffffffffff (on x86-64)

**Kernel-space memory**:
- Never swapped (always resident)
- Accessible directly
- Addresses above 0xffff800000000000

### 8.5 Correct vs Incorrect Data Transfer - Real Examples

```c
// WRONG - kernel oops on invalid user pointer
static ssize_t bad_read(struct file *filp, char __user *buf, ...)
{
    char *kbuf = "test";
    memcpy(buf, kbuf, 4);  // ❌ Direct dereference of user pointer
    return 4;
}

// CORRECT - safe user access
static ssize_t good_read(struct file *filp, char __user *buf, ...)
{
    char *kbuf = "test";
    if (copy_to_user(buf, kbuf, 4))  // ✅ Safe transfer
        return -EFAULT;
    return 4;
}
```

**Another wrong pattern**:
```c
// WRONG - no length validation
static ssize_t bad_write(struct file *filp, char __user *buf, size_t len)
{
    char kbuf[100];
    if (len > 100) len = 100;  // Potential overflow
    copy_from_user(kbuf, buf, len);  // Still risky
}

// CORRECT - strict validation
static ssize_t good_write(struct file *filp, char __user *buf, size_t len)
{
    char kbuf[100];
    if (len > sizeof(kbuf)) return -EINVAL;  // Reject oversized
    if (copy_from_user(kbuf, buf, len)) return -EFAULT;
    return len;
}
```

---

# Part 6: Kernel Version Compatibility

## 9. Kernel Version Compatibility

### 9.1 Why Kernel APIs Change - Internal vs System Call Stability

**System call ABI** (userspace interface) is **stable** across kernel versions. An application compiled for Linux 3.0 still runs on Linux 6.0.

**Internal kernel APIs** (what drivers use) change frequently. The kernel developers reserve the right to modify internal structures and functions between releases.

**Key document**: `Documentation/process/stable-api-nonsense.rst` explains why kernel APIs are not stable—it's to enable refactoring and improvement.

### 9.2 Checking Version at Compile Time

#### 9.2.1 The KERNEL_VERSION Macro

```c
#include <linux/version.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
    // Code for kernel 5.6+
    cls = class_create(DEVICE_NAME);
#else
    // Code for older kernels
    cls = class_create(THIS_MODULE, DEVICE_NAME);
#endif
```

**KERNEL_VERSION** macro: `(a << 16) + (b << 8) + c`
- `KERNEL_VERSION(5, 6, 0)` = `(5 << 16) | (6 << 8) | 0` = `0x050600`

### 9.3 Common Compatibility Issues and Solutions

#### 9.3.1 v5.6+ - proc_ops for procfs

**Change**: procfs separated from file_operations to improve security and clarity.

```c
// v5.5 and older:
static struct file_operations proc_fops = {
    .owner = THIS_MODULE,
    .read = proc_read,
    .write = proc_write,
};

// v5.6+:
static struct proc_ops proc_fops = {
    .proc_read = proc_read,
    .proc_write = proc_write,
    // No .owner needed!
};
```

**Solution**: Conditional compilation:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
static const struct proc_ops fops = { ... };
#else
static const struct file_operations fops = { ... };
#endif
```

#### 9.3.2 v6.4+ - class_create() Signature

**Change**: `class_create()` no longer takes `THIS_MODULE` as first parameter.

```c
// v6.3 and older:
cls = class_create(THIS_MODULE, "myclass");

// v6.4+:
cls = class_create("myclass");
```

**Solution**:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 4, 0)
    cls = class_create(DEVICE_NAME);
#else
    cls = class_create(THIS_MODULE, DEVICE_NAME);
#endif
```

#### 9.3.3 v3.14+ - Thread-Safe f_pos

**Change**: `f_pos` updates are now protected by an internal lock.

```c
// v3.13 and older:
static ssize_t read(..., loff_t *off)
{
    // Had to manually lock f_pos updates
    static DEFINE_MUTEX(pos_lock);
    mutex_lock(&pos_lock);
    *off += bytes_read;
    mutex_unlock(&pos_lock);
}

// v3.14+:
static ssize_t read(..., loff_t *off)
{
    // Kernel handles locking automatically
    *off += bytes_read;  // Thread-safe
}
```

#### 9.3.4 v5.4+ - New file_operations Fields

Kernels add new fields to `file_operations` frequently. Leave unused fields as `NULL` and use designated initializers to avoid breakage.

### 9.4 Best Practice for Version-Aware Code

**Always check version at compile time**, not runtime. This avoids adding runtime overhead and makes code clearer.

**Example pattern**:
```c
static int my_compat_function(void)
{
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
    // Use new API
    return new_api_call();
#else
    // Use old API
    return old_api_call();
#endif
}
```

**Document compatibility**: In module description, note which kernel versions are supported.
