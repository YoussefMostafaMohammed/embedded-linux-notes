# Kernel Module Communication: sysfs, ioctl & System Calls

## 📑 Table of Contents

- [1. Introduction: Three Pillars of Kernel-Userspace Communication](#1-introduction)
- [2. sysfs: The /sys Filesystem Interface](#2-sysfs)
  - [2.1 What is sysfs and Why It Exists](#21-what-is-sysfs)
  - [2.2 Core Concepts: kobjects and Attributes](#22-core-concepts)
  - [2.3 Code Deep Dive: hello-sysfs.c](#23-code-hello-sysfs)
  - [2.4 Building, Testing, and Debugging](#24-testing-sysfs)
- [3. ioctl: Device-Specific Control Interface](#3-ioctl)
  - [3.1 The Problem ioctl Solves](#31-ioctl-purpose)
  - [3.2 ioctl Command Encoding Magic](#32-ioctl-encoding)
  - [3.3 Kernel Implementation: ioctl.c](#33-kernel-ioctl)
  - [3.4 Character Device Implementation: chardev2.c](#34-chardev-impl)
  - [3.5 Userspace Program: userspace_ioctl.c](#35-userspace-program)
  - [3.6 Race Conditions & Synchronization](#36-race-conditions)
- [4. System Call Interception: The Nuclear Option](#4-syscall-interception)
  - [4.1 What Are System Calls?](#41-what-are-syscalls)
  - [4.2 Why This Is Dangerous (And Heavily Restricted)](#42-dangers)
  - [4.3 Kernel Version Compatibility Matrix](#43-version-matrix)
  - [4.4 Code Deep Dive: syscall-steal.c](#44-code-syscall-steal)
  - [4.5 Finding sys_call_table: Four Methods](#45-finding-table)
  - [4.6 Bypassing Write Protection](#46-write-protection)
  - [4.7 The Kprobes Alternative](#47-kprobes-alternative)
- [5. Comparative Analysis: When to Use What](#5-comparative-analysis)
- [6. Security Considerations & Best Practices](#6-security)
- [7. Troubleshooting Guide](#7-troubleshooting)
- [8. Additional Resources](#8-resources)

---

## 1. Introduction

This document explains three fundamental mechanisms for communicating with and extending the Linux kernel. Each represents a different level of abstraction and power:

- **sysfs** (Section 2): The *declarative* approach - expose module variables as filesystem entries
- **ioctl** (Section 3): The *procedural* approach - send commands to device files
- **System Calls** (Section 4): The *invasive* approach - intercept kernel entry points

**Understanding the progression is critical**: sysfs is for configuration, ioctl is for device control, and syscall interception is for modifying global kernel behavior. Jumping to syscalls without understanding the others is like performing surgery before learning anatomy.

---

## 2. sysfs: The `/sys` Filesystem Interface

### 2.1 What is sysfs and Why It Exists

**sysfs** is a pseudo-filesystem that exports kernel data structures to userspace. Think of it as the kernel's "control panel" - every file in `/sys` represents a kernel variable, statistic, or configuration option.

**Key Principles:**
- **Everything is a file**: Even kernel internals appear as regular files you can `cat` and `echo`
- **Structure reflects hierarchy**: `/sys/kernel/`, `/sys/devices/`, `/sys/module/` mirror kernel object relationships
- **No custom protocol needed**: Uses standard file I/O (`open`, `read`, `write`)
- **Security via permissions**: Standard Unix file permissions (`0660`, etc.) control access

**Why not just use `/proc`?**  
Historically, `/proc` was for process information. sysfs was created to handle *system* information and device models more cleanly. Modern kernels prefer sysfs for driver parameters.

### 2.2 Core Concepts: kobjects and Attributes

#### The `kobject` Glue
A `kobject` is the fundamental building block of the kernel's device model. It's not just a data structure—it's a **reference-counted object** that provides:

- **Hierarchical organization**: kobjects can be parent/child
- **Sysfs representation**: Each kobject appears as a directory
- **Reference counting**: Automatic cleanup when no longer needed
- **Event handling**: Hotplug events via udev

**Mission Creep Alert**: kobjects started as simple reference counters but grew into the backbone of the entire Linux device model.

#### The `attribute` Structure
```c
struct attribute {
    char *name;          // Filename in sysfs
    struct module *owner; // Owning module (for ref counting)
    umode_t mode;        // File permissions (0660, etc.)
};
```
This is just metadata. The real work is done by **container attributes** like `device_attribute` that add `show()` and `store()` function pointers.

### 2.3 Code Deep Dive: hello-sysfs.c

Let's dissect the example line-by-line to understand the magic:

```c
static struct kobject *mymodule;  // Global pointer to our sysfs directory
static int myvariable = 0;        // The actual variable we want to expose
```

#### Attribute Operations
```c
static ssize_t myvariable_show(struct kobject *kobj,
                               struct kobj_attribute *attr, char *buf)
{
    return sprintf(buf, "%d\n", myvariable);  // Kernel → Userspace
}

static ssize_t myvariable_store(struct kobject *kobj,
                                struct kobj_attribute *attr, const char *buf,
                                size_t count)
{
    sscanf(buf, "%d", &myvariable);  // Userspace → Kernel
    return count;  // Must return bytes consumed
}
```
**Critical Detail**: The `count` parameter is the size of the buffer from userspace. Returning `count` tells sysfs "I processed everything." Returning less would cause partial writes.

#### Attribute Declaration
```c
static struct kobj_attribute myvariable_attribute =
    __ATTR(myvariable, 0660, myvariable_show, myvariable_store);
```
The `__ATTR` macro expands to:
```c
struct kobj_attribute myvariable_attribute = {
    .attr = {.name = "myvariable", .mode = 0660},
    .show = myvariable_show,
    .store = myvariable_store
};
```

#### Module Initialization
```c
static int __init mymodule_init(void)
{
    // Create directory: /sys/kernel/mymodule
    mymodule = kobject_create_and_add("mymodule", kernel_kobj);
    if (!mymodule)
        return -ENOMEM;  // Out of memory is only failure mode

    // Create file: /sys/kernel/module/myvariable
    error = sysfs_create_file(mymodule, &myvariable_attribute.attr);
    if (error) {
        kobject_put(mymodule);  // Cleanup on failure
        // ...
    }
}
```
**Why `kernel_kobj`?** This makes our directory appear under `/sys/kernel/`. Alternatives:
- `&THIS_MODULE->mkobj.kobj`: Under `/sys/module/<module_name>/`
- `NULL`: Creates at `/sys/` root (bad practice)

#### Cleanup
```c
static void __exit mymodule_exit(void)
{
    kobject_put(mymodule);  // Decrement refcount, removes when 0
}
```
**Memory Safety**: The kobject will only be removed when its reference count hits zero. This prevents use-after-free if userspace still has files open.

### 2.4 Building, Testing, and Debugging

**Makefile** (you'll need this):
```makefile
obj-m += hello-sysfs.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

**Testing Sequence Explained**:
```bash
# 1. Insert module (creates sysfs entries)
sudo insmod hello-sysfs.ko

# 2. Verify module loaded
lsmod | grep hello_sysfs
# If you see it, the module is in memory

# 3. Read current value (invokes myvariable_show)
cat /sys/kernel/mymodule/myvariable
# Returns "0\n" because myvariable was initialized to 0

# 4. Write new value (invokes myvariable_store)
echo "32" | sudo tee /sys/kernel/mymodule/myvariable
# tee is used because > redirection happens in current shell (no sudo)

# 5. Verify change
cat /sys/kernel/mymodule/myvariable
# Returns "32\n"

# 6. Remove module (removes sysfs entries)
sudo rmmod hello_sysfs
# Directory /sys/kernel/mymodule disappears automatically
```

**Debugging Tips**:
- Use `dmesg -w` to watch kernel logs in real-time
- Check permissions: `ls -l /sys/kernel/mymodule/myvariable`
- If `echo` fails with "Permission denied", check the `.mode` in `__ATTR`
- The file appears as soon as `sysfs_create_file()` succeeds

---

## 3. ioctl: Device-Specific Control Interface

### 3.1 The Problem ioctl Solves

**The Scenario**: You have a serial port connected to a modem. You need to:
1. **Send data**: Write to `/dev/ttyS0` → goes to phone line
2. **Configure the port**: Set baud rate, parity, stop bits

**Problem**: How do you send configuration commands to the *driver* without mixing them with data sent to the *device*?

**Solutions Considered**:
- Separate device files? (`/dev/ttyS0` and `/dev/ttyS0-config`) → Clutter, non-standard
- Magic strings in data stream? → Fragile, error-prone
- **ioctl**: A dedicated control channel

**ioctl Principle**: Device files support a third operation beyond read/write:
```c
int ioctl(int fd, unsigned long request, ...);
```
- `fd`: Open device file
- `request`: Command number (tells driver what to do)
- `...`: Optional argument (pointer to data structure)

### 3.2 ioctl Command Encoding Magic

The ioctl number is not random—it's a carefully encoded integer that contains four fields:

| Bits | Field | Purpose |
|------|-------|---------|
| 31-30 | Direction | `_IOC_NONE`, `_IOC_READ`, `_IOC_WRITE`, `_IOC_READ|_IOC_WRITE` |
| 29-16 | Size | Size of argument structure (14 bits = max 16KB) |
| 15-8 | Type (Magic) | Unique driver identifier (prevents collisions) |
| 7-0 | Number | Command ID within this driver (0-255) |

**The Macros** (from `<linux/ioctl.h>`):
```c
_IO(type, nr)      // No argument
_IOR(type, nr, size)  // Read from kernel (kernel → userspace)
_IOW(type, nr, size)  // Write to kernel (userspace → kernel)
_IOWR(type, nr, size) // Read/Write both directions
```

**Example from chardev.h**:
```c
#define IOC_MAGIC '\x66'  // Magic number (arbitrary but unique)

#define IOCTL_VALSET _IOW(IOC_MAGIC, 0, struct ioctl_arg)
// Write operation, magic 'f', cmd 0, sends struct ioctl_arg

#define IOCTL_VALGET _IOR(IOC_MAGIC, 1, struct ioctl_arg)
// Read operation, magic 'f', cmd 1, receives struct ioctl_arg

#define IOCTL_VALGET_NUM _IOR(IOC_MAGIC, 2, int)
// Read operation, magic 'f', cmd 2, receives an int

#define IOCTL_VALSET_NUM _IOW(IOC_MAGIC, 3, int)
// Write operation, magic 'f', cmd 3, sends an int
```

 **⚠️ CRITICAL**  : The **magic number** must be unique across all drivers. Collisions cause undefined behavior. Official assignments are documented in `Documentation/userspace-api/ioctl/ioctl-number.rst`.

### 3.3 Kernel Implementation: ioctl.c

This module demonstrates a standalone character device with ioctl support.

#### Key Data Structure
```c
struct ioctl_arg {
    unsigned int val;
};
```
This is the **contract** between userspace and kernelspace. Both must agree on its layout.

#### The ioctl Handler
```c
static long test_ioctl_ioctl(struct file *filp, unsigned int cmd,
                             unsigned long arg)
{
    // arg is userspace pointer passed from ioctl() call
    // cmd is the decoded ioctl number
}
```

**Switch Statement Pattern**:
```c
switch (cmd) {
case IOCTL_VALSET:
    // Validate: is this driver expecting this command?
    // The kernel does some validation, but you should too
    
    if (copy_from_user(&data, (int __user *)arg, sizeof(data))) {
        retval = -EFAULT;  // Bad address = userspace bug
        goto done;
    }
    // Process data...
    break;
    
case IOCTL_VALGET:
    // Prepare data...
    if (copy_to_user((int __user *)arg, &data, sizeof(data))) {
        retval = -EFAULT;
        goto done;
    }
    break;
}
```
**Memory Safety**: Always use `copy_from_user()` and `copy_to_user()`. Direct pointer dereference (`*arg`) is a **security vulnerability** and will crash the kernel on some architectures.

#### Read/Write Operations
```c
static ssize_t test_ioctl_read(struct file *filp, char __user *buf,
                               size_t count, loff_t *f_pos)
{
    struct test_ioctl_data *ioctl_data = filp->private_data;
    // private_data is set in open(), unique per file handle
}
```
**File Handle State**: `private_data` is your per-open-file storage. It's where you keep instance-specific data.

#### Open/Close Lifecycle
```c
static int test_ioctl_open(struct inode *inode, struct file *filp)
{
    ioctl_data = kmalloc(sizeof(*ioctl_data), GFP_KERNEL);
    // Allocate per-handle data
    rwlock_init(&ioctl_data->lock);  // Initialize synchronization
    ioctl_data->val = 0xFF;          // Set default
    filp->private_data = ioctl_data; // Attach to handle
    return 0;
}

static int test_ioctl_close(struct inode *inode, struct file *filp)
{
    kfree(filp->private_data);  // Clean up
    filp->private_data = NULL;  // Paranoia
    return 0;
}
```

### 3.4 Character Device Implementation: chardev2.c

This is a more complete example with **concurrency protection**.

#### The Atomic Counter Pattern
```c
static atomic_t already_open = ATOMIC_INIT(CDEV_NOT_USED);
// CDEV_NOT_USED = 0, CDEV_EXCLUSIVE_OPEN = 1
```
This prevents multiple processes from using ioctl simultaneously. The check in `device_ioctl`:
```c
if (atomic_cmpxchg(&already_open, CDEV_NOT_USED, CDEV_EXCLUSIVE_OPEN))
    return -EBUSY;
```
`atomic_cmpxchg()` is "compare-and-swap": only succeed if current value is `CDEV_NOT_USED`.

#### The Device Message Buffer
```c
static char message[BUF_LEN + 1];  // Global shared buffer
```
**⚠️ DESIGN FLAW**: This is **not** per-handle data. All processes share the same buffer, causing race conditions. The ioctl example is better because it uses `private_data`.

#### Ioctl Command Implementation
```c
case IOCTL_SET_MSG:
    // DANGEROUS: No length checking!
    char __user *tmp = (char __user *)ioctl_param;
    get_user(ch, tmp);
    for (i = 0; ch && i < BUF_LEN; i++, tmp++)
        get_user(ch, tmp);
    device_write(..., i, NULL);  // Writes to shared buffer
    break;
```
**Security Issue**: This is a classic buffer overflow. A malicious process could write past `BUF_LEN`. Always validate lengths.

```c
case IOCTL_GET_MSG:
    loff_t offset = 0;
    i = device_read(..., 99, &offset);  // Max 99 bytes
    put_user('\0', ... + i);            // Null-terminate
    break;
```
**Better**: This limits output to 99 bytes, preventing buffer overflow.

```c
case IOCTL_GET_NTH_BYTE:
    ret = (long)message[ioctl_param];  // Direct array access
```
**Race Condition**: `message[]` could be modified by another process/thread between checks.

### 3.5 Userspace Program: userspace_ioctl.c

This shows how userspace interacts with the driver.

#### Device Opening
```c
file_desc = open(DEVICE_PATH, O_RDWR);
// Must match the mode used in the driver
```
**Common Error**: If driver doesn't implement `.write`, opening with `O_RDWR` may still succeed, but writes will return `-EBADF`.

#### Ioctl Wrapper Functions
```c
int ioctl_set_msg(int file_desc, char *message)
{
    ret_val = ioctl(file_desc, IOCTL_SET_MSG, message);
    // message is a pointer to userspace buffer
}
```
**Type Safety**: The macro ensures the argument type matches, but C's weak typing won't catch mismatches at compile time.

#### Dangerous Pattern
```c
/* Warning - this is dangerous because we don't tell
 * the kernel how far it's allowed to write */
ret_val = ioctl(file_desc, IOCTL_GET_MSG, message);
```
The comment is honest: the kernel could write past the 100-byte buffer. **Best Practice**: Use two ioctls:
1. `IOCTL_GET_MSG_SIZE` - returns required buffer size
2. `IOCTL_GET_MSG` - fills pre-sized buffer

#### Byte-by-Byte Reading
```c
do {
    c = ioctl(file_desc, IOCTL_GET_NTH_BYTE, i++);
    putchar(c);
} while (c != 0);
```
**Performance**: This is **extremely inefficient** - one syscall per byte. For 1KB message, that's 1024 context switches.

### 3.6 Race Conditions & Synchronization

The code mentions atomic CAS (Compare-And-Swap) in Section 6.5. Here are the patterns:

#### **Pattern 1: Atomic Flags** (used in chardev2.c)
```c
static atomic_t already_open = ATOMIC_INIT(0);

if (atomic_cmpxchg(&already_open, 0, 1))  // Try to acquire
    return -EBUSY;
// ... critical section ...
atomic_set(&already_open, 0);  // Release
```
**Pros**: Simple, lock-free  
**Cons**: Only allows one accessor (no concurrency at all)

#### **Pattern 2: Read-Write Locks** (used in ioctl.c)
```c
rwlock_t lock;  // Allows multiple readers OR one writer

read_lock(&lock);    // Shared lock for reading
val = data->val;
read_unlock(&lock);

write_lock(&lock);   // Exclusive lock for writing
data->val = new_val;
write_unlock(&lock);
```
**Pros**: Allows concurrent reads  
**Cons**: Writers can starve readers in some implementations

#### **Pattern 3: Mutexes** (not shown but recommended for ioctl)
```c
static DEFINE_MUTEX(ioctl_mutex);

mutex_lock(&ioctl_mutex);
// ... critical section ...
mutex_lock(&ioctl_mutex);
```
**Pros**: Sleeping lock (safer), better for long operations  
**Cons**: Can't be used in interrupt context

**Key Insight**: The ioctl.c example is **more correct** than chardev2.c because it protects *data* with rwlocks, while chardev2.c only protects the *device open state* with an atomic, leaving the shared buffer unprotected.

---

## 4. Advanced: System Call Interception

### 4.1 What Are System Calls?

**The Big Picture**: 
- **Userspace**: Process runs with restricted CPU mode (Ring 3 on x86)
- **Kernelspace**: Kernel runs with full privileges (Ring 0)
- **System Call**: The **only** legal way to cross this boundary

**The Execution Flow**:
1. Process loads syscall number into register (e.g., `rax` on x86_64)
2. Process loads arguments into registers
3. Process executes `syscall` instruction (or `int 0x80` on older x86)
4. CPU switches to kernel mode, jumps to `entry_SYSCALL_64` (arch/x86/entry/entry_64.S)
5. Kernel looks up function in `sys_call_table[__NR_openat]`
6. Kernel executes function
7. Kernel returns to userspace

**Why This Matters**: Every process uses this path. Hijack it, and you control the entire system.

### 4.2 Why This Is Dangerous (And Heavily Restricted)

#### **The Horror Story**
> "The open() system call was inadvertently disrupted. This resulted in an inability to open any files, run programs, or shut down the system, necessitating a restart of the virtual machine."

**What Happened**: The module likely:
1. Saved original `sys_openat` pointer
2. Overwrote it with a broken function
3. Crashed before restoring the original
4. System left in unrecoverable state

**Why It's Catastrophic**:
- `open()` is used by **every** program including `ls`, `cat`, `sudo`, `reboot`
- If `open()` returns error, shells can't start, logs can't be written, recovery is impossible
- Even `rmmod` needs to `open()` the module file!

#### **The Security Nightmare**
- **Rootkits**: Malware intercepts syscalls to hide files/processes
- **Data theft**: Intercept `read()` to copy sensitive data
- **Privilege escalation**: Modify `setuid()` behavior

**Kernel's Response**:
1. **Unexport `sys_call_table`** (since v2.6.x): Can't find it easily
2. **Unexport `kallsyms_lookup_name`** (v5.7+): Can't find other symbols
3. **Make table read-only**: Protect with CR0.WP bit
4. **Change to switch statement** (v6.9+): Can't hijack table at all

### 4.3 Kernel Version Compatibility Matrix

| Kernel Version | `sys_call_table` | `kallsyms_lookup_name` | CR0.WP | Recommended Method |
|----------------|------------------|------------------------|--------|-------------------|
| ≤ 5.4 | Manual lookup | Exported | `write_cr0()` | Symbol scanning |
| 5.5 - 5.6 | Manual lookup | Exported | `write_cr0()` | `kallsyms_lookup_name()` |
| 5.7 - 5.10 | Manual lookup | **Unexported** | `write_cr0()` | Kprobes or parameter |
| 5.11 - 5.15 | Manual lookup | **Unexported** | **Assembly** | Kprobes or parameter |
| 5.16 - 6.8 | Manual lookup | **Unexported** | **Assembly** | Kprobes or parameter |
| ≥ 6.9 | **Switch statement** | **Unexported** | **Assembly** | **Kprobes only** |

### 4.4 Code Deep Dive: syscall-steal.c

This is the most complex example. Let's break it down by kernel version.

#### **Architecture-Dependent Wrapper Detection**
```c
#ifdef CONFIG_ARCH_HAS_SYSCALL_WRAPPER
static asmlinkage long (*original_call)(const struct pt_regs *);
#else
static asmlinkage long (*original_call)(int, const char __user *, int, umode_t);
#endif
```
**Why**: Newer kernels (v5.1+) pass a single `pt_regs` struct containing all arguments, rather than individual parameters. This makes syscall wrappers more generic.

#### **The Spy Function**
```c
#ifdef CONFIG_ARCH_HAS_SYSCALL_WRAPPER
static asmlinkage long our_sys_openat(const struct pt_regs *regs)
#else
static asmlinkage long our_sys_openat(int dfd, const char __user *filename,
                                      int flags, umode_t mode)
#endif
{
    if (__kuid_val(current_uid()) != uid)  // Check if we're spying on this user
        goto orig_call;

    // Print filename character by character
    pr_info("Opened file by %d: ", uid);
    do {
#ifdef CONFIG_ARCH_HAS_SYSCALL_WRAPPER
        get_user(ch, (char __user *)regs->si + i);  // si register = 2nd arg
#else
        get_user(ch, (char __user *)filename + i);
#endif
        pr_info("%c", ch);
    } while (ch != 0);
    pr_info("\n");

orig_call:
    // Call original - CRITICAL: maintains system functionality
    return original_call(...);
}
```
**Key Insight**: We **must** call the original syscall. Failing to do so breaks the entire system.

#### **Module Parameters**
```c
static uid_t uid = -1;
module_param(uid, int, 0644);  // Can be set at insmod time
```
Usage: `sudo insmod syscall-steal.ko uid=1000`

### 4.5 Finding sys_call_table: Four Methods

#### **Method 1: Brute Force Scan (≤ 5.4)**
```c
unsigned long int offset = PAGE_OFFSET;
while (offset < ULLONG_MAX) {
    sct = (unsigned long **)offset;
    if (sct[__NR_close] == (unsigned long *)ksys_close)
        return sct;
    offset += sizeof(void *);
}
```
**How It Works**: Scans entire kernel memory looking for a pointer to `ksys_close()` at the index of `__NR_close`.
**Problems**: 
- Extremely slow (minutes on large memory)
- `ksys_close()` was removed in v5.11
- KASLR makes this unreliable

#### **Method 2: kallsyms_lookup_name() (5.5-5.6)**
```c
unsigned long **sct = (unsigned long **)kallsyms_lookup_name("sys_call_table");
```
**Simplest method**, but symbol was unexported in v5.7 for security.

#### **Method 3: Kprobes (≥ 5.7)**
```c
struct kprobe kp = { .symbol_name = "kallsyms_lookup_name" };
register_kprobe(&kp);
kallsyms_lookup_name = (void *)kp.addr;
unregister_kprobe(&kp);
```
**How Kprobes Work**:
1. Insert breakpoint at function entry
2. When hit, save registers and call your handler
3. Your handler gets function address from `kp.addr`
4. Remove breakpoint

**Pros**: Works even for unexported symbols  
**Cons**: Requires `CONFIG_KPROBES=y` in kernel config

#### **Method 4: Module Parameter (Fallback)**
```c
static unsigned long sym = 0;
module_param(sym, ulong, 0644);
```
**Usage**:
```bash
# Get address from /proc/kallsyms
$ sudo grep sys_call_table /proc/kallsyms
ffffffff820013a0 R sys_call_table

# Pass as parameter
$ sudo insmod syscall-steal.ko sym=0xffffffff820013a0 uid=1000
```
**Caveats**:
- `/boot/System.map` is static and may be wrong due to KASLR
- `/proc/kallsyms` is correct but changes every boot with KASLR
- **Disable KASLR** for development: Add `nokaslr` to kernel cmdline

### 4.6 Bypassing Write Protection

#### **The CR0 Register**
CR0 is a control register with bit flags:
- **Bit 0 (PE)**: Protection Enable
- **Bit 16 (WP)**: Write Protect - when set, kernel can't write to read-only pages

#### **Historical: write_cr0()**
```c
// In kernels < 5.3
write_cr0(read_cr0() & ~0x10000);  // Clear WP bit
```
**Why Removed**: Attackers could use this to disable all memory protections.

#### **Modern: Custom Assembly**
```c
static inline void __write_cr0(unsigned long cr0)
{
    asm volatile("mov %0,%%cr0" : "+r"(cr0) : : "memory");
}
```
**Bypass**: Directly write to CR0 using inline assembly. The kernel can't prevent this from *inside* the kernel.

**The Dance**:
```c
disable_write_protection();  // Clear WP bit
sys_call_table[__NR_openat] = (unsigned long *)our_sys_openat;  // Modify
enable_write_protection();   // Set WP bit again
```
**⚠️ CRITICAL**: Must re-enable protection immediately. Leaving it off is like leaving your front door open.

### 4.7 The Kprobes Alternative (Modern Method)

Since v6.9 (and backported to stable kernels), `sys_call_table` is a **switch statement**, not a function pointer table. This means **you cannot modify it**.

**The Solution**: Use Kprobes to intercept at syscall *entry*.

```c
static int sys_call_kprobe_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    if (__kuid_val(current_uid()) != uid)
        return 0;  // Not our target, continue normally
    
    pr_info("%s called by %d\n", syscall_sym, uid);
    return 0;  // Continue to real syscall
}

static struct kprobe syscall_kprobe = {
    .symbol_name = "__x64_sys_openat",
    .pre_handler = sys_call_kprobe_pre_handler,
};

// In init:
register_kprobe(&syscall_kprobe);

// In exit:
unregister_kprobe(&syscall_kprobe);
```
**How It Works Now**:
1. Kprobe places breakpoint at `__x64_sys_openat` entry
2. Before real syscall executes, your handler runs
3. Handler can inspect registers (arguments) but **cannot easily modify them**
4. Real syscall runs after handler returns

**Pros**: 
- Works on modern kernels
- Safer (no table corruption)
- Reversible

**Cons**: 
- Cannot prevent syscall execution
- Cannot modify return value easily
- Higher overhead

---

## 5. Comparative Analysis

| Feature | sysfs | ioctl | System Call Interception |
|---------|-------|-------|--------------------------|
| **Purpose** | Configuration | Device Control | Global Behavior Modification |
| **Scope** | Per-module | Per-device file | System-wide |
| **Interface** | Files in `/sys` | `ioctl(fd, cmd, arg)` | `syscall()` (transparent) |
| **Security** | File permissions | File descriptor access | Full kernel access |
| **Difficulty** | Easy | Medium | Expert/Dangerous |
| **Compatibility** | Excellent (stable ABI) | Good (magic numbers) | Broken by kernel changes |
| **Performance** | Medium (file I/O) | Fast (direct call) | Very fast (when not intercepted) |
| **Use Case** | Module parameters | Device commands | Debugging, security research |

**Rule of Thumb**:
- Need to adjust a setting? → **sysfs**
- Need to send complex commands to hardware? → **ioctl**
- Need to monitor/modify all processes? → **Kprobes** (never table modification)

---

## 6. Security Considerations & Best Practices

### ✅ DO:
1. **Use sysfs for simple parameters** - It's the safest, most idiomatic way
2. **Validate all userspace input** - Check bounds, permissions, sanity
3. **Use private_data for per-handle state** - Avoid global variables
4. **Prefer mutexes over atomics for complex ops** - Mutexes are easier to reason about
5. **Always restore state in cleanup** - Assume your module will crash
6. **Test in a VM** - Use QEMU/KVM, never your main system
7. **Run `sync` before insmod/rmmod** - Flushes filesystem caches to disk
8. **Use Kprobes for syscall observation** - It's the modern, supported method

### ❌ DON'T:
1. **Never modify sys_call_table** on production or kernels ≥ 6.9 (or ≥ 5.15.154)
2. **Never trust userspace pointers** without `copy_from/to_user()`
3. **Never leave CR0.WP disabled** longer than necessary
4. **Never use global buffers** without synchronization
5. **Don't hardcode syscall numbers** - Use `__NR_openat` from `<asm/unistd.h>`
6. **Don't ignore error returns** from `copy_from_user()` - It means userspace is gone
7. **Don't forget module_param permissions** - `0644` allows users to read parameter values

### 🛡️ VM Setup for Safe Development

```bash
# Create disk image
qemu-img create -f qcow2 kernel-dev.img 20G

# Install minimal Ubuntu (or use prebuilt image)
# Boot with KASLR disabled and Kprobes enabled
qemu-system-x86_64 -kernel /boot/vmlinuz-$(uname -r) \
    -append "root=/dev/sda1 nokaslr" \
    -enable-kvm -m 4G -smp 2 \
    -drive file=kernel-dev.img,format=qcow2
```

**Snapshot Before Tests**:
```bash
# In QEMU monitor (Ctrl+Alt+2)
savevm before_module_test

# If system crashes
loadvm before_module_test
```

---

## 7. Troubleshooting Guide

### sysfs Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| `cat: /sys/.../myvariable: No such file` | Module not loaded | `lsmod`, `dmesg` for errors |
| `echo: Permission denied` | Wrong file mode | Check `__ATTR(mode,...)` |
| `dmesg` shows "failed to create file" | Out of memory | Check `kobject_create_and_add` return |
| Value doesn't update after write | show() not refreshing | Ensure show() reads actual variable, not cached |

### ioctl Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| `ioctl: Inappropriate ioctl for device` | Wrong magic number or command | Check header file matches driver |
| `ioctl: Bad address` | `copy_from_user` failed | Validate userspace pointer before use |
| System hangs on ioctl | Deadlock in kernel | Use `mutex_lock_interruptible()` instead |
| Data corruption | Race condition | Add locks, use `private_data` |

### System Call Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| `insmod: Unknown symbol` | Using unexported symbol | Use Kprobes or manual lookup |
| System freezes on insmod | CR0.WP not restored | Double-check enable_write_protection() call |
| Can't find sys_call_table | KASLR enabled | Use `/proc/kallsyms`, not `/boot/System.map` |
| Oops on module removal | Another module modified same syscall | Check before/after values, use Kprobes |
| syscall doesn't intercept | v6.9+ switch statement | Must use Kprobes, not table modification |

### Debug Commands
```bash
# View kernel logs live
dmesg -wH

# Check module info
modinfo hello-sysfs.ko

# View exported symbols
cat /proc/kallsyms | grep mymodule

# Trace syscalls (userspace)
strace -e trace=openat ./your_program

# Check kernel config
grep CONFIG_KPROBES /boot/config-$(uname -r)

# Disable module signing check (for testing)
sudo mokutil --disable-validation  # UEFI systems only
```

---

## 8. Additional Resources

### Official Documentation
- **sysfs**: `Documentation/admin-guide/sysfs-rules.rst`
- **ioctl**: `Documentation/userspace-api/ioctl/ioctl-number.rst`
- **System Calls**: `Documentation/user/hardware/entry.rst`
- **Kprobes**: `Documentation/trace/kprobes.rst`
- **Device Model**: `Documentation/driver-api/driver-model/driver.rst`

### Recommended Reading
- **"Linux Device Drivers, 3rd Edition"** (Corbet, Rubini, Kroah-Hartman)
  - Chapter 4: Debugging Techniques
  - Chapter 14: The Linux Device Model
  - Chapter 6: Advanced Char Driver Operations (ioctl)
  
- **"Understanding the Linux Kernel"** (Bovet & Cesati)
  - Chapter 10: System Calls
  
- **LWN.net Articles**:
  - [sysfs: The Filesystem for Exporting Kernel Objects](https://lwn.net/Articles/51437/)
  - [Unexporting the system call table](https://lwn.net/Articles/813350/)
  - [Control-flow integrity for the kernel](https://lwn.net/Articles/810071/)

### Tools
- **Kernelshark**: Visualize kernel traces
- **ftrace**: Function tracing built into kernel
- **bpftrace**: Modern eBPF-based tracing (safer than module hijinks)
- **QEMU + GDB**: Source-level kernel debugging

### Example: Safe Modern Alternative (eBPF)
Instead of syscall stealing, use eBPF:
```c
// Much safer, works on modern kernels
SEC("kprobe/__x64_sys_openat")
int trace_openat(struct pt_regs *ctx)
{
    uid_t uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
    if (uid == target_uid) {
        bpf_printk("User %d opened file\n", uid);
    }
    return 0;
}
```

---

## Final Thoughts

**The Evolution of Kernel Security**:
The progression from easy table modification → Kprobes → eBPF shows the kernel community's response to security threats. Each step makes it harder to accidentally (or maliciously) break the system.

**Your Learning Path**:
1. **Master sysfs** - It's simple, safe, and used everywhere
2. **Understand ioctl** - Essential for real device drivers
3. **Study Kprobes** - The modern way to observe kernel behavior
4. **Avoid syscall table modification** - It's obsolete and dangerous

The code examples provided are educational, but **syscall-steal.c should never be used in production**. For monitoring syscalls, **eBPF is the future**—it provides the same capabilities with sandboxed safety and works on all modern kernels.

**Remember**: The kernel is not user space. A bug doesn't cause a segmentation fault—it causes a system crash, data loss, or security vulnerability. Always test in a VM, keep backups, and use the safest API available for your use case.