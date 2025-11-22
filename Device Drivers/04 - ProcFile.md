# Linux Character Device Drivers & procfs

## 📑 Table of Contents

### **procfs Deep Dive**
1. [procfs vs Character Devices - Complete Guide](#1-procfs-vs-character-devices---complete-guide)
    - 1.1 [Context: procfs vs Character Devices](#11-context-procfs-vs-character-devices)
    - 1.2 [The proc_ops Structure (Linux v5.6+)](#12-the-proc_ops-structure-linux-v56)
      - 1.2.1 [Why the Change from file_operations?](#121-why-the-change-from-file_operations)
      - 1.2.2 [Structure Comparison - Old vs New](#122-structure-comparison---old-vs-new)
      - 1.2.3 [Function Signature Changes](#123-function-signature-changes)
      - 1.2.4 [Key Differences Summary](#124-key-differences-summary)
    - 1.3 [Writing Version-Compatible procfs Code](#13-writing-version-compatible-procfs-code)
      - 1.3.1 [Conditional Compilation Pattern](#131-conditional-compilation-pattern)
      - 1.3.2 [proc_create() Usage](#132-proc_create-usage)
    - 1.4 [Complete procfs Example](#14-complete-procfs-example)
      - 1.4.1 [Module Code](#141-module-code)
      - 1.4.2 [Makefile](#142-makefile)
      - 1.4.3 [Testing Commands](#143-testing-commands)
    - 1.5 [When to Use proc_create_single()](#15-when-to-use-proc_create_single)
    - 1.6 [Summary Table - Character Devices vs procfs](#16-summary-table---character-devices-vs-procfs)

### **procfs Practical Implementation**
2. [The /proc Filesystem - Practical Implementation](#2-the-proc-filesystem---practical-implementation)
    - 2.1 [Basic Concepts and Rationale](#21-basic-concepts-and-rationale)
      - 2.1.1 [Why You Need procfile](#2111-why-you-need-procfile)
      - 2.1.2 [Why to Use It](#2112-why-to-use-it)
      - 2.1.3 [What Happens Without procfile](#2113-what-happens-without-procfile)
      - 2.1.4 [Write Operations on procfile - Are They Needed?](#2114-write-operations-on-procfile---are-they-needed)
      - 2.1.5 [When to Use procfile (Not Just for Debugging)](#2115-when-to-use-procfile-not-just-for-debugging)
    - 2.2 [The proc_ops Structure in Depth](#22-the-proc_ops-structure-in-depth)
      - 2.2.1 [Definition and Purpose](#221-definition-and-purpose)
      - 2.2.2 [Performance Optimizations with proc_flags](#222-performance-optimizations-with-proc_flags)
      - 2.2.3 [Backward Compatibility with file_operations](#223-backward-compatibility-with-file_operations)
    - 2.3 [Read and Write a /proc File - Complete Walkthrough](#23-read-and-write-a-proc-file---complete-walkthrough)
      - 2.3.1 [procfile_read() Implementation](#231-procfile_read-implementation)
      - 2.3.2 [procfile_write() Implementation](#232-procfile_write-implementation)
      - 2.3.3 [The Offset Parameter - Critical Detail](#233-the-offset-parameter---critical-detail)
      - 2.3.4 [Full Example: procfs1.c](#234-full-example-procfs1c)
      - 2.3.5 [Full Example: procfs2.c](#235-full-example-procfs2c)
    - 2.4 [Manage /proc file with Standard Filesystem](#24-manage-proc-file-with-standard-filesystem)
      - 2.4.1 [struct inode_operations and Permissions](#241-struct-inode_operations-and-permissions)
      - 2.4.2 [module_permission() Function](#242-modulepermission-function)
      - 2.4.3 [Full Example: procfs3.c](#243-full-example-procfs3c)
    - 2.5 [Manage /proc file with seq_file](#25-manage-proc-file-with-seq_file)
      - 2.5.1 [seq_file API Overview](#251-seq_file-api-overview)
      - 2.5.2 [Sequence Functions: start(), next(), stop(), show()](#252-sequence-functions-start-next-stop-show)
      - 2.5.3 [How seq_file Works - The Loop](#253-how-seq_file-works---the-loop)
      - 2.5.4 [Diagram: seq_file Operation](#254-diagram-seq_file-operation)
      - 2.5.5 [Full Example: procfs4.c](#255-full-example-procfs4c)
      - 2.5.6 [Benefits of seq_file](#256-benefits-of-seq_file)
    - 2.6 [procfs Best Practices](#26-procfs-best-practices)
      - 2.6.1 [When to Use procfs vs sysfs](#261-when-to-use-procfs-vs-sysfs)
      - 2.6.2 [Security Considerations](#262-security-considerations)
      - 2.6.3 [Performance Tips](#263-performance-tips)

---

## **procfs Deep Dive**

### 1. [procfs vs Character Devices - Complete Guide](#1-procfs-vs-character-devices---complete-guide)

#### 1.1 [Context: procfs vs Character Devices](#11-context-procfs-vs-character-devices)

The `/proc` filesystem and character devices (`/dev`) serve fundamentally different purposes in the Linux kernel ecosystem, despite both providing user-space access to kernel information.

**Critical Distinction:**
- **Character Devices (`/dev`)**: Represent hardware or virtual devices that exchange data streams. They use `file_operations` and are registered with a major/minor number system. Examples: serial ports, custom hardware, virtual devices.
- **procfs (`/proc`)**: A virtual filesystem for exposing kernel state, process information, and system statistics. Originally designed for process information (hence "proc"), now used for general kernel introspection and configuration.

**Why They Need Different Structures:**
Before Linux v5.6, procfs **misused** `file_operations` from character devices. This created security risks and code bloat because:
1. **Security Risk**: Some `file_operations` callbacks (like `mmap`, `splice`, `flock`) were never meant for procfs and had undefined behavior
2. **Code Bloat**: Every time VFS expanded `file_operations`, procfs inherited unused fields
3. **Confusion**: Made it unclear which callbacks were safe/valid for procfs

The solution was `proc_ops` - a **procfs-specific** structure that only includes callbacks that make sense for virtual files.

#### 1.2 [The proc_ops Structure (Linux v5.6+)](#12-the-proc_ops-structure-linux-v56)

##### 1.2.1 [Why the Change from file_operations?](#121-why-the-change-from-file_operations)

The old design had **critical problems**:

**Problem 1: Security Vulnerabilities**
```c
// Before v5.6: procfs could implement mmap
static const struct file_operations old_proc_fops = {
    .mmap = my_mmap,  // ❌ DANGER! procfs has no physical memory to map!
};

// This could leak kernel memory or cause undefined behavior
```

**Problem 2: Performance Overhead**
```c
// Every procfs open had to initialize 30+ function pointers
struct file_operations has 30+ fields
→ Only 4-5 are used by procfs
→ Wasted memory and CPU cycles
```

**Problem 3: Maintenance Nightmare**
When VFS added new fields to `file_operations`, procfs code had to be updated even though it never used those fields.

##### 1.2.2 [Structure Comparison - Old vs New](#122-structure-comparison---old-vs-new)

**Old Way (≤ v5.5):**
```c
static const struct file_operations old_proc_fops = {
    .owner = THIS_MODULE,      // Required
    .open = my_open,
    .read = my_read,
    .write = my_write,
    .release = my_release,
    // .mmap, .splice, etc. would be silently ignored but waster space
};
```

**New Way (v5.6+):**
```c
static const struct proc_ops new_proc_fops = {
    .proc_open = my_open,
    .proc_read = my_read,
    .proc_write = my_write,
    .proc_release = my_release,
    // NO owner field! Kernel manages it automatically
    // NO unused fields! Clean and minimal
};
```

##### 1.2.3 [Function Signature Changes](#123-function-signature-changes)

**Important**: Function signatures are **identical**, only the structure field names changed:

| Operation | Character Device (`file_operations`) | procfs (`proc_ops`) |
|-----------|--------------------------------------|---------------------|
| **open** | `int (*open)(struct inode *, struct file *)` | `int (*proc_open)(struct inode *, struct file *)` |
| **read** | `ssize_t (*read)(struct file *, char __user *, size_t, loff_t *)` | `ssize_t (*proc_read)(struct file *, char __user *, size_t, loff_t *)` |
| **write** | `ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *)` | `ssize_t (*proc_write)(struct file *, const char __user *, size_t, loff_t *)` |
| **ioctl** | `long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long)` | **NOT AVAILABLE** |

**Critical Limitation**: `ioctl` is **not supported** in procfs. Use `write` with custom parsing instead.

##### 1.2.4 [Key Differences Summary](#124-key-differences-summary)

| Feature | Character Devices (`/dev`) | procfs (`/proc`) |
|---------|---------------------------|------------------|
| **Structure** | `struct file_operations` | `struct proc_ops` (v5.6+) |
| **Owner field** | Required (`THIS_MODULE`) | Not needed |
| **Registration** | `register_chrdev()` / `cdev_add()` | `proc_create()` |
| **Cleanup** | `unregister_chrdev()` / `cdev_del()` | `remove_proc_entry()` |
| **ioctl support** | Yes (`unlocked_ioctl`) | No (use `write`) |
| **Auto-create dev file** | Yes (`device_create`) | No (procfs is virtual) |
| **Thread safety** | f_pos lock (v3.14+) | Same as character devices |
| **Typical use** | Hardware interaction | Kernel data/debugging |

#### 1.3 [Writing Version-Compatible procfs Code](#13-writing-version-compatible-procfs-code)

##### 1.3.1 [Conditional Compilation Pattern](#131-conditional-compilation-pattern)

```c
#include <linux/version.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
#define HAVE_PROC_OPS 1
#endif

// Function implementations (same signatures work for both)
static int my_proc_open(struct inode *inode, struct file *file)
{
    printk(KERN_INFO "proc file opened\n");
    return 0;
}

static ssize_t my_proc_read(struct file *file, char __user *buf, 
                            size_t count, loff_t *ppos)
{
    return simple_read_from_buffer(buf, count, ppos, my_data, data_len);
}

#ifdef HAVE_PROC_OPS
static const struct proc_ops my_proc_ops = {
    .proc_open = my_proc_open,
    .proc_read = my_proc_read,
};
#else
static const struct file_operations my_proc_ops = {
    .owner = THIS_MODULE,  // Required for old kernels
    .open = my_proc_open,
    .read = my_proc_read,
};
#endif
```

##### 1.3.2 [proc_create() Usage](#132-proc_create-usage)

The `proc_create()` function signature is **identical** for both versions:

```c
// In init function
static int __init my_module_init(void)
{
    struct proc_dir_entry *entry;
    
    // The 4th argument type changes based on version
#ifdef HAVE_PROC_OPS
    entry = proc_create("my_proc", 0666, NULL, &my_proc_ops);
#else
    entry = proc_create("my_proc", 0666, NULL, &my_proc_ops);
#endif
    
    if (!entry)
        return -ENOMEM;
        
    return 0;
}
```

#### 1.4 [Complete procfs Example](#14-complete-procfs-example)

##### 1.4.1 [Module Code](#141-module-code)

```c
// proc_lkm_example.c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/proc_fs.h>
#include <linux/version.h>
#include <linux/uaccess.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
#define HAVE_PROC_OPS
#endif

#define PROC_NAME "lkm_example"
#define BUFFER_SIZE 128

static char proc_buffer[BUFFER_SIZE];
static size_t proc_buff_len = 0;

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 10, 0)
#include <linux/minmax.h>
#endif

static ssize_t proc_read(struct file *file, char __user *ubuf, 
                         size_t count, loff_t *ppos)
{
    if (*ppos > 0 || count < proc_buff_len)
        return 0;

    if (copy_to_user(ubuf, proc_buffer, proc_buff_len))
        return -EFAULT;

    *ppos = proc_buff_len;
    return proc_buff_len;
}

static ssize_t proc_write(struct file *file, const char __user *ubuf,
                          size_t count, loff_t *ppos)
{
    proc_buff_len = min(count, BUFFER_SIZE - 1);
    
    if (copy_from_user(proc_buffer, ubuf, proc_buff_len))
        return -EFAULT;

    proc_buffer[proc_buff_len] = '\0';
    pr_info("procfs: wrote %s\n", proc_buffer);
    
    *ppos += proc_buff_len;
    return proc_buff_len;
}

#ifdef HAVE_PROC_OPS
static const struct proc_ops proc_fops = {
    .proc_open = simple_open,
    .proc_read = proc_read,
    .proc_write = proc_write,
};
#else
static const struct file_operations proc_fops = {
    .owner = THIS_MODULE,
    .open = simple_open,
    .read = proc_read,
    .write = proc_write,
};
#endif

static int __init proc_init(void)
{
    struct proc_dir_entry *entry;
    
    strcpy(proc_buffer, "Hello from kernel!\n");
    proc_buff_len = strlen(proc_buffer);

    entry = proc_create(PROC_NAME, 0666, NULL, &proc_fops);
    if (!entry) {
        pr_err("proc_create failed\n");
        return -ENOMEM;
    }

    pr_info("/proc/%s created\n", PROC_NAME);
    return 0;
}

static void __exit proc_exit(void)
{
    remove_proc_entry(PROC_NAME, NULL);
    pr_info("/proc/%s removed\n", PROC_NAME);
}

module_init(proc_init);
module_exit(proc_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Cross-version procfs example");
```

##### 1.4.2 [Makefile](#142-makefile)

```makefile
obj-m += proc_lkm_example.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

##### 1.4.3 [Testing Commands](#143-testing-commands)

```bash
# Build and load
$ make
$ sudo insmod proc_lkm_example.ko
$ dmesg | tail
[ 1234.567890] /proc/lkm_example created

# Read from procfs
$ cat /proc/lkm_example
Hello from kernel!

# Write to procfs
$ echo "New data" | sudo tee /proc/lkm_example
New data
$ dmesg | tail
[ 1234.567890] procfs: wrote New data

# Check it persisted
$ cat /proc/lkm_example
New data

# Cleanup
$ sudo rmmod proc_lkm_example
$ dmesg | tail
[ 1234.567890] /proc/lkm_example removed
```

#### 1.5 [When to Use proc_create_single()](#15-when-to-use-proc_create_single)

For **simple, read-only** proc files, Linux 4.18+ provides a simpler API that eliminates boilerplate:

```c
// Instead of defining proc_ops, just provide a show function
static int my_show(struct seq_file *m, void *v)
{
    seq_printf(m, "System uptime: %ld\n", jiffies);
    seq_printf(m, "CPU count: %d\n", num_online_cpus());
    return 0;
}

static int __init my_init(void)
{
    // Creates read-only proc entry with single_show function
    proc_create_single("system_info", 0444, NULL, my_show);
    return 0;
}
```

**Benefits**:
- No need for `proc_ops` or `file_operations`
- No `open`, `read`, `write` functions to implement
- Works across versions (with compatibility checks)
- More secure and less error-prone
- Handles offset management automatically

**When to use**:
- Read-only informational files
- Debugging output
- System status reports
- Any simple data display

#### 1.6 [Summary Table - Character Devices vs procfs](#16-summary-table---character-devices-vs-procfs)

| Aspect | Character Devices (`/dev`) | procfs (`/proc`) |
|--------|---------------------------|------------------|
| **Purpose** | Hardware/software data streams | Kernel state/configuration |
| **Structure** | `struct file_operations` | `struct proc_ops` (v5.6+) |
| **Owner Field** | Required (`THIS_MODULE`) | **Not needed** |
| **Registration** | `register_chrdev()` / `cdev_add()` | `proc_create()` / `proc_create_single()` |
| **Cleanup** | `unregister_chrdev()` / `cdev_del()` | `remove_proc_entry()` |
| **Access Path** | `/dev/mydevice` | `/proc/myfile` |
| **ioctl Support** | Yes (`unlocked_ioctl`) | **No** (use `write`) |
| **Auto-creation** | Yes (with udev + class) | No (always manual) |
| **Major/Minor** | Yes, required | No, procfs is virtual |
| **Thread-Safe f_pos** | Yes (v3.14+) | Yes |
| **Typical Use** | Hardware drivers, virtual devices | Kernel debugging, statistics |
| **Power Management** | Part of device model | Not applicable |
| **Hotplug Support** | Full udev integration | Manual only |

---

## **procfs Practical Implementation**

### 2. [The /proc Filesystem - Practical Implementation](#2-the-proc-filesystem---practical-implementation)

#### 2.1 [Basic Concepts and Rationale](#21-basic-concepts-and-rationale)

##### 2.1.1 [Why You Need procfile](#2111-why-you-need-procfile)

A `/proc` file (also called "procfile") is the **kernel's window to user-space**. Without it:

- **No runtime visibility**: You can't see internal kernel state while the module is running
- **No configuration**: You can't change module behavior without reloading it
- **No debugging**: When something goes wrong, you have nothing but dmesg
- **No statistics**: Can't monitor performance, counters, or health

**Procfile solves this** by providing a simple, text-based interface that works with **standard Unix tools** (`cat`, `echo`, `grep`, `awk`).

##### 2.1.2 [Why to Use It](#2112-why-to-use-it)

**Advantages over character devices**:
1. **Simplicity**: No major/minor numbers, no device files
2. **Visibility**: Automatically appears in `/proc` without udev
3. **Accessibility**: Any user can read `/proc` (with proper permissions)
4. **Text-based**: Easy to parse, human-readable

**Advantages over sysfs**:
- **No hierarchy required**: Can create standalone files
- **Simpler API**: Don't need kobject hierarchies
- **More flexible**: Can implement any read/write logic

##### 2.1.3 [What Happens Without procfile](#2113-what-happens-without-procfile)

```c
// Without procfile, debugging is painful:
static int counter = 0;

static ssize_t device_read(...) {
    counter++;
    // How do you know the current counter value?
    // printk() every time? That's slow and floods dmesg!
}
```

**With procfile**:
```bash
$ cat /proc/mydevice_counter
42  # Instant visibility, no dmesg flooding
```

##### 2.1.4 [Write Operations on procfile - Are They Needed?](#2114-write-operations-on-procfile---are-they-needed)

**Yes!** Write operations allow:
- **Runtime configuration**: Change module parameters without reloading
- **Command interface**: Send commands to your driver
- **Reset counters**: Clear statistics on demand
- **Debugging triggers**: Enable debug mode, dump state

**Example**:
```bash
# Enable debug mode
$ echo "debug=1" > /proc/mydevice_config

# Reset statistics
$ echo "reset" > /proc/mydevice_stats

# Dump internal state
$ echo "dump" > /proc/mydevice_debug
$ cat /proc/mydevice_debug
Internal buffer: [data...]
```

##### 2.1.5 [When to Use procfile (Not Just for Debugging)](#2115-when-to-use-procfile-not-just-for-debugging)

**Production use cases**:
- **Performance monitoring**: Expose request rates, latency percentiles
- **Health checks**: Systemd can monitor `/proc/health` to restart services
- **Feature flags**: Enable/disable features without restart
- **Statistics**: Export prometheus-style metrics
- **Configuration**: Runtime tuning parameters

**Example**: `iptables` uses `/proc/net/ip_tables_matches` to show available match modules.

#### 2.2 [The proc_ops Structure in Depth](#22-the-proc_ops-structure-in-depth)

##### 2.2.1 [Definition and Purpose](#221-definition-and-purpose)

```c
// include/linux/proc_fs.h (v5.6+)
struct proc_ops {
    unsigned int proc_flags;  // NEW: Performance flags
    int (*proc_open)(struct inode *, struct file *);  // Called on open()
    ssize_t (*proc_read)(struct file *, char __user *, size_t, loff_t *);  // Called on read()
    ssize_t (*proc_write)(struct file *, const char __user *, size_t, loff_t *);  // Called on write()
    loff_t (*proc_lseek)(struct file *, loff_t, int);  // Called on seek()
    int (*proc_release)(struct inode *, struct file *);  // Called on close()
    __poll_t (*proc_poll)(struct file *, struct poll_table_struct *);  // For poll/select
    long (*proc_ioctl)(struct file *, unsigned int, unsigned long);  // For ioctl (rare)
    // ... more fields for advanced use
};
```

**Purpose**: Provide **procfs-specific** callbacks that are **guaranteed safe** for virtual files. The kernel validates that only these callbacks are used.

##### 2.2.2 [Performance Optimizations with proc_flags](#222-performance-optimizations-with-proc_flags)

```c
struct proc_ops my_proc_ops = {
    .proc_flags = PROC_ENTRY_PERMANENT,  // File never disappears
    // This saves:
    // - 2 atomic operations per open/read/close
    // - 1 memory allocation per open
    // - 1 memory free per close
    .proc_read = my_read,
};
```

**Available flags**:
- `PROC_ENTRY_PERMANENT`: File exists for lifetime of module (most common)
- `PROC_ENTRY_SAFE_NO_PPROC`: No parent process checks (faster)
- `PROC_ENTRY_WILL_WORK`: Optimistic locking (advanced)

**Real-world impact**: On a busy system with 1000s of proc reads/second, these flags can save **thousands of atomic operations** per second.

##### 2.2.3 [Backward Compatibility with file_operations](#223-backward-compatibility-with-file_operations)

```c
// For maximum compatibility, use this pattern:
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
static struct proc_ops my_ops = {
    .proc_flags = PROC_ENTRY_PERMANENT,
    .proc_read = my_read,
    .proc_write = my_write,
};
#else
static const struct file_operations my_ops = {
    .owner = THIS_MODULE,
    .read = my_read,
    .write = my_write,
};
#endif

// In init:
proc_create("myfile", 0644, NULL, &my_ops);
```

**Key insight**: The **function signatures are identical**, only the **container structure** changes. This makes conditional compilation easy.

#### 2.3 [Read and Write a /proc File - Complete Walkthrough](#23-read-and-write-a-proc-file---complete-walkthrough)

##### 2.3.1 [procfile_read() Implementation](#231-procfile_read-implementation)

The `read` callback has a **critical requirement**: handle the `offset` parameter correctly.

```c
static ssize_t procfile_read(struct file *file, char __user *buffer,
                             size_t length, loff_t *offset)
{
    static const char *msg = "Hello from kernel!\n";
    static const int msg_len = 18;  // Including \n
    
    // CRITICAL: Check offset - if we've already read everything, return 0
    if (*offset >= msg_len) {
        return 0;  // EOF - tells kernel to stop calling us
    }
    
    // Calculate how much we can send
    size_t bytes_to_send = msg_len - *offset;
    if (bytes_to_send > length)
        bytes_to_send = length;
    
    // Copy data to user space
    if (copy_to_user(buffer, msg + *offset, bytes_to_send))
        return -EFAULT;
    
    // Update offset for next call
    *offset += bytes_to_send;
    
    return bytes_to_send;
}
```

**Why this works**:
1. **First call**: `offset = 0`, copy full message, set `offset = 18`
2. **Second call**: `offset = 18`, `>= msg_len`, return 0 → stop

**Common mistake**:
```c
// WRONG - Never returns 0, causes infinite loop!
static ssize_t bad_read(...) {
    copy_to_user(buffer, msg, msg_len);
    return msg_len;  // Always returns same value, never EOF
}
```

##### 2.3.2 [procfile_write() Implementation](#232-procfile_write-implementation)

Write is simpler - just copy from user and process:

```c
static char proc_buffer[1024];
static size_t proc_buffer_len = 0;

static ssize_t procfile_write(struct file *file, const char __user *buffer,
                              size_t len, loff_t *offset)
{
    // Limit size
    proc_buffer_len = min(len, sizeof(proc_buffer) - 1);
    
    // Copy from user - THIS IS MANDATORY
    if (copy_from_user(proc_buffer, buffer, proc_buffer_len))
        return -EFAULT;
    
    // Null-terminate
    proc_buffer[proc_buffer_len] = '\0';
    
    // Process the command
    if (strcmp(proc_buffer, "reset\n") == 0) {
        reset_counters();
    } else if (strstr(proc_buffer, "debug=")) {
        int debug_val = simple_strtol(proc_buffer + 6, NULL, 10);
        debug_mode = debug_val;
    }
    
    *offset += proc_buffer_len;
    return proc_buffer_len;  // Must return bytes written
}
```

**Key points**:
- **Always use `copy_from_user()`**: Never dereference `buffer` directly
- **Validate input**: Check bounds, parse carefully
- **Return value**: Must be bytes written or negative error
- **Offset**: Update it (though less critical than in read)

##### 2.3.3 [The Offset Parameter - Critical Detail](#233-the-offset-parameter---critical-detail)

The `loff_t *offset` parameter is **pointer to file position**. It **persists across calls** within the same `read()` system call.

**Visualization**:
```
User calls: read(fd, buf, 100)

Kernel calls procfile_read():
  offset = 0
  → returns 50 (copied 50 bytes)
  → *offset = 50

Kernel calls procfile_read() AGAIN:
  offset = 50
  → returns 30 (copied 30 more bytes)
  → *offset = 80

Kernel calls procfile_read() AGAIN:
  offset = 80
  → returns 0 (EOF)
  → read() returns 80 total to user

User gets back: 80 bytes in buf
```

**Golden rule**: `*offset` must **advance** by the number of bytes you return. When `*offset >= total_data_size`, return 0 to end.

##### 2.3.4 [Full Example: procfs1.c](#234-full-example-procfs1c)

```c
/* procfs1.c - Simple read-only /proc file */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/uaccess.h>
#include <linux/version.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
#define HAVE_PROC_OPS
#endif

#define procfs_name "helloworld"

static struct proc_dir_entry *our_proc_file;

static ssize_t procfile_read(struct file *file_pointer, char __user *buffer,
                             size_t buffer_length, loff_t *offset)
{
    char s[13] = "HelloWorld!\n";
    int len = sizeof(s);
    ssize_t ret = len;

    if (*offset >= len || copy_to_user(buffer, s, len)) {
        pr_info("copy_to_user failed\n");
        ret = 0;
    } else {
        pr_info("procfile read %s\n", file_pointer->f_path.dentry->d_name.name);
        *offset += len;
    }

    return ret;
}

#ifdef HAVE_PROC_OPS
static const struct proc_ops proc_file_fops = {
    .proc_read = procfile_read,
};
#else
static const struct file_operations proc_file_fops = {
    .read = procfile_read,
};
#endif

static int __init procfs1_init(void)
{
    our_proc_file = proc_create(procfs_name, 0644, NULL, &proc_file_fops);
    if (NULL == our_proc_file) {
        pr_alert("Error:Could not initialize /proc/%s\n", procfs_name);
        return -ENOMEM;
    }

    pr_info("/proc/%s created\n", procfs_name);
    return 0;
}

static void __exit procfs1_exit(void)
{
    proc_remove(our_proc_file);
    pr_info("/proc/%s removed\n", procfs_name);
}

module_init(procfs1_init);
module_exit(procfs1_exit);
MODULE_LICENSE("GPL");
```

**Testing**:
```bash
$ make && sudo insmod procfs1.ko
$ cat /proc/helloworld
HelloWorld!
$ sudo rmmod procfs1
```

##### 2.3.5 [Full Example: procfs2.c](#235-full-example-procfs2c)

```c
/* procfs2.c - Read and write /proc file */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/uaccess.h>
#include <linux/version.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
#define HAVE_PROC_OPS
#endif

#define PROCFS_MAX_SIZE 1024
#define PROCFS_NAME "buffer1k"

static struct proc_dir_entry *our_proc_file;
static char procfs_buffer[PROCFS_MAX_SIZE];
static unsigned long procfs_buffer_size = 0;

static ssize_t procfile_read(struct file *file_pointer, char __user *buffer,
                             size_t buffer_length, loff_t *offset)
{
    char s[13] = "HelloWorld!\n";
    int len = sizeof(s);
    ssize_t ret = len;

    if (*offset >= len || copy_to_user(buffer, s, len)) {
        pr_info("copy_to_user failed\n");
        ret = 0;
    } else {
        pr_info("procfile read %s\n", file_pointer->f_path.dentry->d_name.name);
        *offset += len;
    }

    return ret;
}

static ssize_t procfile_write(struct file *file, const char __user *buff,
                              size_t len, loff_t *off)
{
    procfs_buffer_size = len;
    if (procfs_buffer_size >= PROCFS_MAX_SIZE)
        procfs_buffer_size = PROCFS_MAX_SIZE - 1;

    if (copy_from_user(procfs_buffer, buff, procfs_buffer_size))
        return -EFAULT;

    procfs_buffer[procfs_buffer_size] = '\0';
    *off += procfs_buffer_size;
    pr_info("procfile write %s\n", procfs_buffer);
    return procfs_buffer_size;
}

#ifdef HAVE_PROC_OPS
static const struct proc_ops proc_file_fops = {
    .proc_read = procfile_read,
    .proc_write = procfile_write,
};
#else
static const struct file_operations proc_file_fops = {
    .read = procfile_read,
    .write = procfile_write,
};
#endif

static int __init procfs2_init(void)
{
    our_proc_file = proc_create(PROCFS_NAME, 0644, NULL, &proc_file_fops);
    if (NULL == our_proc_file) {
        pr_alert("Error:Could not initialize /proc/%s\n", PROCFS_NAME);
        return -ENOMEM;
    }

    pr_info("/proc/%s created\n", PROCFS_NAME);
    return 0;
}

static void __exit procfs2_exit(void)
{
    proc_remove(our_proc_file);
    pr_info("/proc/%s removed\n", PROCFS_NAME);
}

module_init(procfs2_init);
module_exit(procfs2_exit);
MODULE_LICENSE("GPL");
```

**Testing**:
```bash
$ echo "test data" | sudo tee /proc/buffer1k
test data
$ cat /proc/buffer1k
HelloWorld!
# Note: read doesn't echo back what you wrote - this example just shows the mechanism
```

#### 2.4 [Manage /proc file with Standard Filesystem](#24-manage-proc-file-with-standard-filesystem)

##### 2.4.1 [struct inode_operations and Permissions](#241-struct-inode_operations-and-permissions)

For advanced control, you can customize **inode operations** to handle permissions, links, and other file metadata.

```c
static int module_permission(struct inode *inode, int op)
{
    // op = MAY_READ, MAY_WRITE, MAY_EXEC, etc.
    if (op == 4 || (op == 2 && current_euid().val == 0))
        return 0;  // Allow
    
    return -EACCES;  // Deny
}

static struct inode_operations my_inode_ops = {
    .permission = module_permission,  // Custom permission check
};
```

**When to use**:
- Custom permission logic beyond simple mode bits
- Dynamic permissions based on system state
- Advanced features like extended attributes

##### 2.4.2 [module_permission() Function](#242-modulepermission-function)

```c
static int module_permission(struct inode *inode, int op)
{
    /* 
     * op values:
     * 4 (MAY_READ) - Check if reading is allowed
     * 2 (MAY_WRITE) - Check if writing is allowed
     * 1 (MAY_EXEC) - Check if executing is allowed
     * 
     * current_euid().val - Current effective user ID
     */
    
    if (op == 4) { // READ
        // Allow read for everyone
        return 0;
    }
    
    if (op == 2) { // WRITE
        // Only root can write
        if (current_euid().val == 0)
            return 0;
        else
            return -EACCES;
    }
    
    return -EACCES;  // Default deny
}
```

**Integration**:
```c
static const struct proc_ops fops = {
    .proc_read = proc_read,
    .proc_write = proc_write,
};

static struct proc_dir_entry *entry;

static int __init init(void)
{
    entry = proc_create("myfile", 0, NULL, &fops);
    if (entry)
        entry->proc_iops = &my_inode_ops;  // Set custom inode ops
    return 0;
}
```

##### 2.4.3 [Full Example: procfs3.c](#243-full-example-procfs3c)

```c
/* procfs3.c - Advanced with permissions */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/uaccess.h>
#include <linux/sched.h>
#include <linux/version.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
#define HAVE_PROC_OPS
#endif

#define PROCFS_MAX_SIZE 2048UL
#define PROCFS_ENTRY_FILENAME "buffer2k"

static struct proc_dir_entry *our_proc_file;
static char procfs_buffer[PROCFS_MAX_SIZE];
static unsigned long procfs_buffer_size = 0;

static int module_permission(struct inode *inode, int op)
{
    if (op == 4 || (op == 2 && current_euid().val == 0))
        return 0;
    return -EACCES;
}

static ssize_t procfs_read(struct file *filp, char __user *buffer,
                           size_t length, loff_t *offset)
{
    if (*offset || procfs_buffer_size == 0) {
        *offset = 0;
        return 0;
    }
    procfs_buffer_size = min(procfs_buffer_size, length);
    
    if (copy_to_user(buffer, procfs_buffer, procfs_buffer_size))
        return -EFAULT;
    *offset += procfs_buffer_size;
    
    pr_debug("procfs_read: read %lu bytes\n", procfs_buffer_size);
    return procfs_buffer_size;
}

static ssize_t procfs_write(struct file *file, const char __user *buffer,
                            size_t len, loff_t *off)
{
    procfs_buffer_size = min(PROCFS_MAX_SIZE, len);
    if (copy_from_user(procfs_buffer, buffer, procfs_buffer_size))
        return -EFAULT;
    
    *off += procfs_buffer_size;
    pr_debug("procfs_write: write %lu bytes\n", procfs_buffer_size);
    return procfs_buffer_size;
}

static int procfs_open(struct inode *inode, struct file *file)
{
    try_module_get(THIS_MODULE);
    return 0;
}

static int procfs_close(struct inode *inode, struct file *file)
{
    module_put(THIS_MODULE);
    return 0;
}

#ifdef HAVE_PROC_OPS
static struct proc_ops file_ops_4_our_proc_file = {
    .proc_read = procfs_read,
    .proc_write = procfs_write,
    .proc_open = procfs_open,
    .proc_release = procfs_close,
};
#else
static const struct file_operations file_ops_4_our_proc_file = {
    .read = procfs_read,
    .write = procfs_write,
    .open = procfs_open,
    .release = procfs_close,
};
#endif

static int __init procfs3_init(void)
{
    our_proc_file = proc_create(PROCFS_ENTRY_FILENAME, 0644, NULL,
                                &file_ops_4_our_proc_file);
    if (our_proc_file == NULL) {
        pr_debug("Error: Could not initialize /proc/%s\n",
                 PROCFS_ENTRY_FILENAME);
        return -ENOMEM;
    }
    
    proc_set_size(our_proc_file, 80);
    proc_set_user(our_proc_file, GLOBAL_ROOT_UID, GLOBAL_ROOT_GID);
    
    pr_debug("/proc/%s created\n", PROCFS_ENTRY_FILENAME);
    return 0;
}

static void __exit procfs3_exit(void)
{
    remove_proc_entry(PROCFS_ENTRY_FILENAME, NULL);
    pr_debug("/proc/%s removed\n", PROCFS_ENTRY_FILENAME);
}

module_init(procfs3_init);
module_exit(procfs3_exit);
MODULE_LICENSE("GPL");
```

#### 2.5 [Manage /proc file with seq_file](#25-manage-proc-file-with-seq_file)

##### 2.5.1 [seq_file API Overview](#251-seq_file-api-overview)

`seq_file` is a **helper API** for producing formatted output in `/proc` files. It handles:
- **Offset management**: Automatically tracks position across reads
- **Large outputs**: Breaks big data into chunks automatically
- **Multiple items**: Iterates through lists, arrays, trees
- **Formatting**: Provides `seq_printf()`, `seq_puts()`, etc.

**When to use**: Any read-only proc file that produces multiple lines or items.

##### 2.5.2 [Sequence Functions: start(), next(), stop(), show()](#252-sequence-functions-start-next-stop-show)

The seq_file API uses **four functions** to create an iterator:

```c
struct seq_operations {
    void * (*start)(struct seq_file *m, loff_t *pos);  // Initialize sequence
    void * (*next)(struct seq_file *m, void *v, loff_t *pos);  // Get next item
    void (*stop)(struct seq_file *m, void *v);  // Cleanup
    int (*show)(struct seq_file *m, void *v);  // Display current item
};
```

**Flow**:
```
start(pos=0) → show() → next() → show() → next() → ... → next() returns NULL → stop()
```

##### 2.5.3 [How seq_file Works - The Loop](#253-how-seq_file-works---the-loop)

```c
static void *my_seq_start(struct seq_file *m, loff_t *pos)
{
    static unsigned long counter = 0;
    
    // pos is the item number (0, 1, 2, ...)
    if (*pos == 0) {
        return &counter;  // Return pointer to first item
    }
    
    *pos = 0;
    return NULL;  // End of sequence
}

static void *my_seq_next(struct seq_file *m, void *v, loff_t *pos)
{
    unsigned long *tmp_v = (unsigned long *)v;
    (*tmp_v)++;  // Increment value
    (*pos)++;    // Increment position
    
    if (*pos >= 10)  // Print 10 items
        return NULL;   // End
    
    return v;  // Continue
}

static void my_seq_stop(struct seq_file *m, void *v)
{
    // Cleanup if needed (e.g., release locks)
    // In this simple example, nothing to do
}

static int my_seq_show(struct seq_file *m, void *v)
{
    loff_t *spos = (loff_t *)v;
    seq_printf(m, "Position: %Ld\n", *spos);
    return 0;  // 0 = success
}

// Wire it up:
static const struct seq_operations my_seq_ops = {
    .start = my_seq_start,
    .next = my_seq_next,
    .stop = my_seq_stop,
    .show = my_seq_show,
};

static int my_open(struct inode *inode, struct file *file)
{
    return seq_open(file, &my_seq_ops);  // Kernel handles the rest
}

#ifdef HAVE_PROC_OPS
static const struct proc_ops my_file_ops = {
    .proc_open = my_open,
    .proc_read = seq_read,      // Use kernel's seq_read
    .proc_lseek = seq_lseek,    // Use kernel's seq_lseek
    .proc_release = seq_release, // Use kernel's seq_release
};
#else
static const struct file_operations my_file_ops = {
    .open = my_open,
    .read = seq_read,
    .llseek = seq_lseek,
    .release = seq_release,
};
#endif
```

##### 2.5.4 [Diagram: seq_file Operation](#254-diagram-seq_file-operation)

```
User: cat /proc/mydata

Kernel: open()
  → my_open()
    → seq_open() [kernel]
      → calls my_seq_start(pos=0)
        → returns ptr to first item

Kernel: read()
  → seq_read() [kernel]
    → calls my_seq_show()
      → prints "Position: 0\n"
    → calls my_seq_next()
      → pos=1, returns ptr
    → calls my_seq_show()
      → prints "Position: 1\n"
    → ... repeats until ...
    → my_seq_next() returns NULL

Kernel: close()
  → seq_release() [kernel]
    → calls my_seq_stop()
      → cleanup

User sees:
Position: 0
Position: 1
Position: 2
...
Position: 9
```

##### 2.5.5 [Full Example: procfs4.c](#255-full-example-procfs4c)

```c
/* procfs4.c - Using seq_file for multiple entries */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/version.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0)
#define HAVE_PROC_OPS
#endif

#define PROC_NAME "iter"

static void *my_seq_start(struct seq_file *s, loff_t *pos)
{
    static unsigned long counter = 0;
    if (*pos == 0) {
        return &counter;
    }
    *pos = 0;
    return NULL;
}

static void *my_seq_next(struct seq_file *s, void *v, loff_t *pos)
{
    unsigned long *tmp_v = (unsigned long *)v;
    (*tmp_v)++;
    (*pos)++;
    return (*pos < 10) ? v : NULL;  // 10 items max
}

static void my_seq_stop(struct seq_file *s, void *v)
{
    // Nothing to do for this simple example
}

static int my_seq_show(struct seq_file *m, void *v)
{
    loff_t *spos = (loff_t *)v;
    seq_printf(m, "%Ld\n", *spos);
    return 0;
}

static struct seq_operations my_seq_ops = {
    .start = my_seq_start,
    .next = my_seq_next,
    .stop = my_seq_stop,
    .show = my_seq_show,
};

static int my_open(struct inode *inode, struct file *file)
{
    return seq_open(file, &my_seq_ops);
}

#ifdef HAVE_PROC_OPS
static const struct proc_ops my_file_ops = {
    .proc_open = my_open,
    .proc_read = seq_read,
    .proc_lseek = seq_lseek,
    .proc_release = seq_release,
};
#else
static const struct file_operations my_file_ops = {
    .open = my_open,
    .read = seq_read,
    .llseek = seq_lseek,
    .release = seq_release,
};
#endif

static int __init procfs4_init(void)
{
    struct proc_dir_entry *entry;
    
    entry = proc_create(PROC_NAME, 0, NULL, &my_file_ops);
    if (entry == NULL) {
        pr_debug("Error: Could not initialize /proc/%s\n", PROC_NAME);
        return -ENOMEM;
    }
    
    return 0;
}

static void __exit procfs4_exit(void)
{
    remove_proc_entry(PROC_NAME, NULL);
    pr_debug("/proc/%s removed\n", PROC_NAME);
}

module_init(procfs4_init);
module_exit(procfs4_exit);
MODULE_LICENSE("GPL");
```

**Testing**:
```bash
$ cat /proc/iter
0
1
2
3
4
5
6
7
8
9
```

##### 2.5.6 [Benefits of seq_file](#256-benefits-of-seq_file)

1. **No manual offset handling**: Kernel manages `*offset` across calls
2. **Automatic chunking**: Large outputs are split across multiple reads automatically
3. **Iterator pattern**: Easy to traverse lists, arrays, trees
4. **Formatting helpers**: `seq_printf()`, `seq_puts()`, `seq_escape()` etc.
5. **Powerful and safe**: Less error-prone than manual read implementation

**When NOT to use seq_file**:
- Simple single-value files (use `proc_create_single()`)
- Write support (seq_file is read-only)
- Files that need non-linear access (e.g., seek to arbitrary positions)

#### 2.6 [procfs Best Practices](#26-procfs-best-practices)

##### 2.6.1 [When to Use procfs vs sysfs](#261-when-to-use-procfs-vs-sysfs)

**Use procfs for**:
- Process-specific information (the original purpose)
- Debug and trace data
- Statistics and counters
- Simple kernel-to-user notifications
- Text-heavy outputs

**Use sysfs for**:
- Device-specific configuration
- Hardware parameters
- Power management settings
- One-value-per-file semantics
- Integration with device model

**Modern guidance**: sysfs is preferred for **hardware-related** data, procfs for **process/kernel** data.

##### 2.6.2 [Security Considerations](#262-security-considerations)

1. **Never trust user input**:
```c
static ssize_t proc_write(..., const char __user *buf, size_t len, ...)
{
    char kbuf[100];
    
    // BAD: No size check
    if (copy_from_user(kbuf, buf, len))  // len could be 1MB!
        return -EFAULT;
    
    // GOOD: Limit size
    size_t to_copy = min(len, sizeof(kbuf) - 1);
    if (copy_from_user(kbuf, buf, to_copy))
        return -EFAULT;
}
```

2. **Validate commands**:
```c
if (strcmp(kbuf, "reset") == 0) {
    reset();
} else if (strcmp(kbuf, "enable") == 0) {
    enable();
} else {
    return -EINVAL;  // Unknown command
}
```

3. **Use proper permissions**:
```bash
# In module:
proc_create("config", 0600, NULL, &ops);  # Only root can access

# Or dynamic:
if (current_euid().val == 0)  // Check in write callback
```

4. **Avoid information leaks**:
```c
// BAD: Dumping kernel pointers
seq_printf(m, "Internal pointer: %p\n", internal_ptr);  // Security leak!

// GOOD: Only show what's necessary
seq_printf(m, "Status: active\n");
```

##### 2.6.3 [Performance Tips](#263-performance-tips)

1. **Use `proc_create_single()`** for read-only files:
```c
// Instead of full proc_ops, just:
proc_create_single("stats", 0444, NULL, stats_show);
// Saves memory and improves cache locality
```

2. **Use `seq_file` for multi-line outputs**:
```c
// Manual read with offset handling is error-prone
// seq_file handles it automatically and efficiently
```

3. **Cache expensive computations**:
```c
static char cached_output[1024];
static time_t cache_time;

static int show_cache(struct seq_file *m, void *v)
{
    if (time_before(jiffies, cache_time + HZ)) {  // Cache for 1 second
        seq_puts(m, cached_output);
    } else {
        regenerate_output(cached_output);
        cache_time = jiffies;
        seq_puts(m, cached_output);
    }
}
```

4. **Avoid printk in hot paths**:
```c
// BAD: printk on every read
static ssize_t proc_read(...) {
    printk(KERN_INFO "read called\n");  // Floods dmesg
    // ...
}

// GOOD: Use tracepoints or rate limiting
static ssize_t proc_read(...) {
    if (printk_ratelimit())  // Only prints occasionally
        printk(KERN_INFO "read called\n");
    // ...
}
```

5. **Use `preempt_disable()` for atomic reads**:
```c
static ssize_t proc_read_stats(...) {
    unsigned long flags;
    u64 value;
    
    // If stats are updated from interrupt context:
    raw_spin_lock_irqsave(&stats_lock, flags);
    value = stats.counter;
    raw_spin_unlock_irqrestore(&stats_lock, flags);
    
    seq_printf(m, "%llu\n", value);
}
```