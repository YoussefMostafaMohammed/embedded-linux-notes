# Hello World Module 


## 📑 Table of Contents

1. [Your First Module: The Minimal Hello World](#1-your-first-module-the-minimal-hello-world)
2. [The Makefile: Your Build Foundation](#2-the-makefile-your-build-foundation)
3. [Compilation to Execution: Full Workflow](#3-compilation-to-execution-full-workflow)
4. [Module Metadata and Information](#4-module-metadata-and-information)
5. [Modern Module Initialization](#5-modern-module-initialization)
6. [Memory Optimization with `__init` and `__exit`](#6-memory-optimization-with-__init-and-__exit)
7. [Documentation and Licensing Macros](#7-documentation-and-licensing-macros)
8. [How Modules Begin and End: The Entry/Exit Contract](#8-how-modules-begin-and-end-the-entryexit-contract)
9. [Functions Available to Modules: The Kernel Symbol Universe](#9-functions-available-to-modules-the-kernel-symbol-universe)
10. [User Space vs Kernel Space: The Privilege Divide](#10-user-space-vs-kernel-space-the-privilege-divide)
11. [Namespace Management: Avoiding Symbol Collisions](#11-namespace-management-avoiding-symbol-collisions)
12. [Code Space: Where Your Module Lives](#12-code-space-where-your-module-lives)
13. [Introduction to Device Drivers: From Modules to Hardware](#13-introduction-to-device-drivers-from-modules-to-hardware)
14. [Command-Line Parameter Passing](#14-command-line-parameter-passing)
15. [Multi-File Module Architecture](#15-multi-file-module-architecture)
16. [Building for Precompiled Kernels](#16-building-for-precompiled-kernels)
17. [Troubleshooting Common Errors](#17-troubleshooting-common-errors)
18. [Coding Standards and Best Practices](#18-coding-standards-and-best-practices)
19. [From Hello World to Device Drivers](#19-from-hello-world-to-device-drivers)
20. [Exercise: Experiment with Return Values](#20-exercise-experiment-with-return-values)

---

## 1. Your First Module: The Minimal Hello World

Let's create the simplest possible kernel module to understand the fundamental structure.

### **Directory Setup**
```bash
mkdir -p ~/develop/kernel/hello-1
cd ~/develop/kernel/hello-1
```

### **hello-1.c: The Bare Minimum**
```c
/*   
 * hello-1.c - The simplest kernel module.  
 */

#include <linux/module.h> /* Needed by all modules */
#include <linux/printk.h> /* Needed for pr_info() */

int init_module(void)
{
    pr_info("Hello world 1.\n");

    /* A nonzero return means init_module failed; module can't be loaded. */
    return 0;
}

void cleanup_module(void)
{
    pr_info("Goodbye world 1.\n");
}

MODULE_LICENSE("GPL");
```

### **Key Components Explained**

**`init_module()`** - The module entry point, automatically called when you `insmod` the module. This is where you register functionality with the kernel. **Returning a negative value signals failure**, preventing module load. Returning 0 means success.

**`cleanup_module()`** - The module exit point, automatically called when you `rmmod`. This must undo everything `init_module()` did to prevent resource leaks and system instability.

**`MODULE_LICENSE("GPL")`** - **Critically important**. Without this, your module "taints" the kernel, marking it as potentially non-open-source. Many kernel symbols are **GPL-only** and will be inaccessible without this declaration. Valid licenses: `"GPL"`, `"GPL v2"`, `"Dual BSD/GPL"`, `"Dual MIT/GPL"`, `"Proprietary"`.

**`pr_info()` vs `printk()`** - `pr_info()` is a modern wrapper around `printk(KERN_INFO, ...)`. It reduces typing and improves readability. Priority levels range from `KERN_EMERG` (system crash imminent) to `KERN_DEBUG` (verbose debugging).

---

## 2. The Makefile: Your Build Foundation

### **Basic Makefile Structure**
```makefile
obj-m += hello-1.o

PWD := $(CURDIR)

all:
	$(MAKE) -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	$(MAKE) -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

### **Component Breakdown**

**`obj-m += hello-1.o`** - Tells the kernel build system to create `hello-1.ko` (`m` = module, `y` = built-in, `n` = exclude).

**`-C /lib/modules/$(shell uname -r)/build`** - Invokes the **kernel's own Makefile** (kbuild system). This ensures your module uses the exact same compiler flags, headers, and configuration as the running kernel.

**`M=$(PWD)`** - Points the kernel build system to *your* source directory. The kernel's Makefile will descend into your directory to build the module.

**`PWD := $(CURDIR)`** - Captures the current directory. **This is crucial** because `sudo make` drops environment variables, including PWD.

### **The Sudo Environment Variable Problem**

When you run `sudo make`, the sudo security policy (by default) **clears most environment variables**, including `PWD`. This causes `M=$(PWD)` to expand to nothing, making the kernel build system fail.

**Three Solutions:**

**Solution 1: Use `-E` flag (preserve environment)**
```bash
sudo -E make
```

**Solution 2: Edit `/etc/sudoers` (persistent)**
```bash
sudo visudo
# Add this line:
Defaults env_keep += "PWD"
```

**Solution 3: Use explicit path in Makefile**
```makefile
all:
	$(MAKE) -C /lib/modules/$(shell uname -r)/build M=$(CURDIR) modules
```
**Note**: `$(CURDIR)` is a built-in GNU Make variable that always works, even with sudo.

---

## 3. Compilation to Execution: Full Workflow

### **Step-by-Step Process**

```bash
# 1. Compile the module
make

# 2. Inspect module metadata
modinfo hello-1.ko
# Output should show:
# license: GPL
# vermagic: 6.8.0-48-generic SMP preempt mod_unload modversions

# 3. Verify module is NOT loaded yet
lsmod | grep hello  # Should return nothing

# 4. Load the module
sudo insmod hello-1.ko

# 5. Verify it's loaded (note: dash becomes underscore)
lsmod | grep hello  # Shows: hello_1 16384 0

# 6. View kernel messages
dmesg | tail -10
# Should see: "Hello world 1."

# 7. Unload the module
sudo rmmod hello_1

# 8. Verify removal
lsmod | grep hello  # Should return nothing

# 9. View exit message
dmesg | tail -1
# Should see: "Goodbye world 1."
```

### **Understanding the Dash-to-Underscore Conversion**

The kernel's module naming subsystem converts dashes (`-`) to underscores (`_`) in module names. This is because:
- Dashes are valid in filenames but problematic in symbol names
- The module's internal structures use underscores
- When loading `hello-1.ko`, it's registered as `hello_1` in the kernel's module list

**Always use underscores** when referencing the module after loading (`rmmod hello_1`, not `rmmod hello-1`).

---

## 4. Module Metadata and Information

### **The `modinfo` Command**

`modinfo` extracts data from the `.modinfo` section of your `.ko` file:

```bash
$ modinfo hello-1.ko
filename:       /home/user/hello-1/hello-1.ko
license:        GPL
srcversion:     A1B2C3D4E5F678901234567
depends:        
retpoline:      Y
name:           hello_1
vermagic:       6.8.0-48-generic SMP preempt mod_unload modversions
```

**Metadata Fields:**
- `license`: Determines taint status and symbol visibility
- `depends`: Space-separated list of required modules
- `vermagic`: Version magic string (kernel version, SMP, preempt, etc.)
- `srcversion`: Git commit hash or build-time hash
- `name`: Module name as seen by kernel

### **Adding Rich Metadata**

Use these macros in your module:

```c
MODULE_AUTHOR("Your Name <your.email@example.com>");
MODULE_DESCRIPTION("A simple Hello World driver for learning");
MODULE_VERSION("1.0");
MODULE_ALIAS("alternative-name");  // Allows modprobe to find it by another name
MODULE_DEVICE_TABLE("pci", my_pci_tbl);  // For automatic driver binding
```

**Example:**
```c
MODULE_AUTHOR("LKMPG");
MODULE_DESCRIPTION("A sample driver demonstrating basic module structure");
MODULE_VERSION("1.0");
```
`MODULE_DEVICE_TABLE()` is a kernel macro that **publishes a table of hardware IDs** your driver supports, enabling **automatic driver binding** when matching hardware is detected.

### How it works:

1. **You define a device ID table**: A static array listing hardware Vendor IDs, Device IDs, etc. your driver can handle
2. **The macro exports it**: Makes the table visible to the kernel and userspace (`/lib/modules/*/modules.alias`)
3. **Kernel performs automatic matching**: When a device is detected (boot or hot-plug), the bus subsystem (PCI, USB, etc.) scans this table
4. **Match = automatic probe**: If IDs match, kernel loads your driver and calls its `probe()` function

### Example for PCI:
```c
static const struct pci_device_id my_pci_tbl[] = {
    { PCI_DEVICE(0x8086, 0x1234) },  // Intel specific device
    { PCI_DEVICE(PCI_VENDOR_ID_REALTEK, PCI_DEVICE_ID_REALTEK_8139) },
    { }  // Terminating entry
};
MODULE_DEVICE_TABLE(pci, my_pci_tbl);
```

**Without this**: You'd need manual `insmod` and `bind` operations every time.  
**With this**: Plug-and-play just works - the kernel matchmaker handles everything automatically.

A **device ID table** is a static array in your driver that lists **specific hardware identifiers** (Vendor ID, Device ID, etc.) your driver supports. It acts as a **match catalog** for the kernel's automatic driver binding system.

### What it looks like (PCI example):

```c
static const struct pci_device_id my_device_id_table[] = {
    { PCI_DEVICE(0x8086, 0x1234) },        // Vendor: 0x8086 (Intel), Device: 0x1234
    { PCI_DEVICE(0x1234, 0x5678) },        // Another hardware combination
    { PCI_DEVICE_CLASS(PCI_CLASS_NETWORK_ETHERNET << 8, 0xffff00) }, // All Ethernet devices
    { } // EMPTY ENTRY - marks end of table (REQUIRED)
};
```

### What each entry contains:
- **Vendor ID**: Manufacturer (e.g., `0x8086` = Intel)
- **Device ID**: Specific product/chip
- **Subvendor/Subdevice**: For OEM variants
- **Class**: Broad category (network, storage, video, etc.)
- **Mask**: Which bits to match on

### How it's used:
1. Kernel detects a new PCI device
2. Reads the hardware IDs from the device's configuration space
3. Scans all drivers' device ID tables for a matching pattern
4. **Match found** → Loads your driver and calls its `probe()` function

**Without this table**: No automatic binding - you'd have to manually load and bind the driver every time.

---

## 5. Modern Module Initialization

### **The Old Way vs. The New Way**

**Legacy (pre-2.3.13):**
```c
int init_module(void) { /* ... */ }
void cleanup_module(void) { /* ... */ }
```

**Modern (preferred):**
```c
static int __init hello_2_init(void) { /* ... */ }
static void __exit hello_2_exit(void) { /* ... */ }

module_init(hello_2_init);
module_exit(hello_2_exit);
```

### **Why Use `module_init()` Macros?**

1. **Clarity**: Function names describe what they do, not just their role
2. **Flexibility**: Can have multiple initialization functions in the same file
3. **Initdata**: Works seamlessly with `__init` and `__exit` macros
4. **Consistency**: Matches modern kernel coding style

### **hello-2.c: Modern Template**
```c
#include <linux/init.h>     /* Needed for __init/__exit */
#include <linux/module.h>   /* Needed by all modules */
#include <linux/printk.h>   /* Needed for pr_info() */

static int __init hello_2_init(void)
{
    pr_info("Hello, world 2\n");
    return 0;
}

static void __exit hello_2_exit(void)
{
    pr_info("Goodbye, world 2\n");
}

module_init(hello_2_init);
module_exit(hello_2_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("LKMPG");
MODULE_DESCRIPTION("Modern init/exit style");
```

**Best Practice**: Always use `static` for init/exit functions to prevent polluting the global namespace.

---

## 6. Memory Optimization with `__init` and `__exit`

### **The Memory Waste Problem**

When a module loads, its `init` function executes once, then sits in memory forever, never to be called again. On embedded systems with limited RAM, this waste matters.

### **The `__init` Macro Solution**

```c
static int __init hello_3_init(void)
{
    pr_info("Hello, world 3\n");
    return 0;
}
```

**What `__init` does:**
- Places the function in a special `.init.text` section
- After init completes, kernel frees that memory
- Saves ~2-10KB per module on a typical system

**Kernel message when freeing:**
```
Freeing unused kernel memory: 236k freed
```

### **`__initdata` for Variables**

```c
static int hello3_data __initdata = 3;

static int __init hello_3_init(void)
{
    pr_info("Hello, world %d\n", hello3_data);
    return 0;
}
```

`hello3_data` is only used during initialization, so it's placed in `.init.data` and freed after `init` returns.

### **`__exit` for Cleanup Functions**

```c
static void __exit hello_3_exit(void)
{
    pr_info("Goodbye, world 3\n");
}
```

**What `__exit` does:**
- On **loadable modules**: No effect (function is kept for `rmmod`)
- On **built-in modules** (built into vmlinuz): The function is **omitted entirely** because built-in modules never unload
- Saves space in the core kernel image

### **hello-3.c: Complete Example**
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/printk.h>

static int hello3_data __initdata = 3;

static int __init hello_3_init(void)
{
    pr_info("Hello, world %d\n", hello3_data);
    return 0;
}

static void __exit hello_3_exit(void)
{
    pr_info("Goodbye, world 3\n");
}

module_init(hello_3_init);
module_exit(hello_3_exit);

MODULE_LICENSE("GPL");
```

**Result**: After the module loads, `hello3_data` and the `hello_3_init` code are freed. Only `hello_3_exit` remains for when you unload.

---

## 7. Documentation and Licensing Macros

### **The Taint Warning**

If you omit `MODULE_LICENSE` or use "Proprietary":
```bash
$ sudo insmod xxxxxx.ko
loading out-of-tree module taints kernel.
module license 'unspecified' taints kernel.
```

A **tainted kernel**:
- Makes bug reports worthless to kernel developers
- Can disable GPL-only symbols
- Indicates "non-free code running" for debugging

### **Complete Documentation Example (hello-4.c)**
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/printk.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("LKMPG");
MODULE_DESCRIPTION("A sample driver");
MODULE_VERSION("1.0");

static int __init init_hello_4(void)
{
    pr_info("Hello, world 4\n");
    return 0;
}

static void __exit cleanup_hello_4(void)
{
    pr_info("Goodbye, world 4\n");
}

module_init(init_hello_4);
module_exit(cleanup_hello_4);
```

### **Macro Reference**

| Macro | Purpose | Example |
|-------|---------|---------|
| `MODULE_LICENSE()` | **Required** - Prevents taint | `"GPL"` |
| `MODULE_AUTHOR()` | Attribution | `"Your Name <email>"` |
| `MODULE_DESCRIPTION()` | Human-readable purpose | `"USB webcam driver"` |
| `MODULE_VERSION()` | Version string | `"2.1.4"` |
| `MODULE_ALIAS()` | Alternative name for modprobe | `"pci:v00008086d000015FBsv*sd*bc*sc*i*"` |
| `MODULE_DEVICE_TABLE()` | Auto-binding for device classes | `MODULE_DEVICE_TABLE(pci, my_pci_tbl)` |

---

## 8. How Modules Begin and End: The Entry/Exit Contract

### **The Module Lifecycle Pattern**

Unlike user-space programs that start with `main()` and run linearly, kernel modules follow a **callback-based lifecycle**:

**Entry Point**: `init_module()` or `module_init()`
- Called **once** when module is loaded
- Registers functionality with kernel (drivers, filesystems, etc.)
- Returns 0 on success, negative errno on failure
- After returning, module remains **dormant** until kernel invokes its functions

**Exit Point**: `cleanup_module()` or `module_exit()`
- Called **once** when module is removed
- Must **exactly** undo everything the entry function did
- No return value (cannot fail)
- After returning, module code is unloaded from memory

### **The Mandatory Two-Function Rule**

**Every module MUST have both entry and exit functions.** This is non-negotiable because:
- The kernel needs to know how to initialize your code
- The kernel needs to know how to safely remove your code without leaking resources
- Missing exit function = module cannot be unloaded (rmmod will fail)
- Missing entry function = module has no purpose

### **Alternative Naming Convention**

While `module_init()` and `module_exit()` are preferred, you may still see:
```c
int init_module(void) { /* ... */ }
void cleanup_module(void) { /* ... */ }
```

These are **legacy names** but functionally identical. The terms **"entry function"** and **"exit function"** are generic descriptors that apply regardless of the specific names you choose.

### **Real-World Entry Function Responsibilities**

A typical entry function does:
```c
static int __init mydriver_init(void)
{
    // 1. Allocate resources
    data_buffer = kmalloc(BUF_SIZE, GFP_KERNEL);
    
    // 2. Register with subsystem
    register_chrdev(MAJOR_NUM, "mydriver", &fops);
    
    // 3. Probe hardware
    if (!detect_hardware()) {
        kfree(data_buffer);
        return -ENODEV;
    }
    
    // 4. Register interrupt handler
    request_irq(IRQ_NUM, my_isr, 0, "mydriver", NULL);
    
    return 0;  // Success
}
```

### **Real-World Exit Function Responsibilities**

The matching exit function:
```c
static void __exit mydriver_exit(void)
{
    // 1. Unregister in reverse order
    free_irq(IRQ_NUM, NULL);
    
    // 2. Unregister from subsystem
    unregister_chrdev(MAJOR_NUM, "mydriver");
    
    // 3. Free resources
    kfree(data_buffer);
    
    // 4. Log removal
    pr_info("Mydriver unloaded\n");
}
```

**Critical Principle**: Exit must handle **partial initialization**. If `init` fails halfway, `exit` must still clean up what was done.

---

## 9. Functions Available to Modules: The Kernel Symbol Universe

### **The Symbol Resolution Mechanism**

In user-space, you link against `libc` which provides `printf()`, `malloc()`, etc. In kernel-space, **there is no libc**. Your module links directly against the **running kernel's symbol table**.

### **Viewing Available Symbols**

```bash
# See all symbols exported by your kernel
cat /proc/kallsyms | grep kmalloc
# Output: ffffffff81234560 T kmalloc

# Symbols marked with 'T' are in kernel's text (code) section
# 'D' = initialized data, 'B' = uninitialized data

# Filter for exported symbols (available to modules)
cat /proc/kallsyms | grep __ksymtab
# These are the symbols actively exported via EXPORT_SYMBOL()
```


**Key Files:**
- `/proc/kallsyms`: Runtime symbol table (addresses + names)
- `/lib/modules/$(uname -r)/build/Module.symvers`: Build-time symbol table (names + CRCs)


**Exported symbols** are kernel functions/variables intentionally made **visible and usable** by loadable modules. **Unexported symbols** are private to the kernel and **cannot be accessed** by modules.

### How it works:

**Exporting** (in kernel source):
```c
// In kernel source, drivers/base/core.c
void *kmalloc(size_t size, gfp_t flags) { ... }
EXPORT_SYMBOL(kmalloc);  // Makes it available to modules
```

**Module usage**:
```c
// In your module - you CAN use this
void *data = kmalloc(1024, GFP_KERNEL);
```

**Unexported symbol**:
```c
// In kernel source, some internal function
static void internal_function(void) { ... }
// No EXPORT_SYMBOL() - modules CANNOT use this
```

### What the commands show:

```bash
# All symbols (exported + unexported)
cat /proc/kallsyms | grep kmalloc
# Shows: ffffffff81234560 T kmalloc

# ONLY exported symbols
cat /proc/kallsyms | grep __ksymtab
# Shows entries in kernel's export symbol table
```

-  **`T`**  : Code symbol (in text section)
-  **`D`**  : Initialized data
-  **`B`**  : Uninitialized data

### Why this matters:

If your module tries to use an **unexported symbol**, you'll get:
```
ERROR: "some_function" [/path/to/your.ko] undefined!
```

The build will fail because the linker can't find that symbol in the kernel's **exported symbol table** (`__ksymtab`). Only symbols explicitly marked with `EXPORT_SYMBOL()` or `EXPORT_SYMBOL_GPL()` are available for modules to use.

### **How Symbol Resolution Works**

When you compile `hello.c`:
```c
pr_info("Hello\n");  // Calls kernel function
```

1. Compiler sees `pr_info` is undefined in your code
2. Linker looks it up in `Module.symvers`
3. Creates **relocation entry**: *"I need symbol 'pr_info' at offset 0x48"*
4. When `insmod` runs, kernel patches that offset with **actual address** from `/proc/kallsyms`

### **The Critical Difference: Library Functions vs. System Calls**

**User-Space Flow:**
```c
printf("Hello") → libc: format string → write() syscall → kernel
```

**Kernel-Space Flow:**
```c
pr_info("Hello") → direct kernel function call (no syscall)
```

**System calls** are the **interface between user-space and kernel-space**. They are **not** the kernel's internal API. From within the kernel, you call functions directly.

### **Exploring with `strace`**

```bash
# Create test program
$ cat > test.c <<EOF
#include <stdio.h>
int main(void) { printf("hello\n"); return 0; }
EOF

# Compile
$ gcc -Wall -o test test.c

# Trace system calls
$ strace ./test
...
write(1, "hello\n", 6hello) = 6
exit_group(0) = ?
...

# The printf() library function ultimately used the write() SYSTEM CALL
```

**Moral**: Kernel modules **cannot** use `printf()` because it's a **library function** in user-space libc. They must use `pr_info()` which is a **kernel function**.

### **Replacing System Calls (Advanced)**

Kernel modules can **override** system calls by modifying the syscall table (on kernels that support it):
```c
// Save original
orig_write = sys_call_table[__NR_write];

// Replace with your function
sys_call_table[__NR_write] = my_custom_write;
```

**Security Implication**: This technique is used by rootkits for backdoors. Legitimate uses include auditing, debugging, and security monitoring.

---

## 10. User Space vs Kernel Space: The Privilege Divide

### **The Ring Architecture**

x86 CPUs have **4 privilege levels (rings)**: Ring 0 (most privileged) to Ring 3 (least). Linux uses a simplified model:

- **Ring 0**: **Kernel space** - Full hardware control, can execute any instruction
- **Ring 3**: **User space** - Restricted, cannot directly access hardware or kernel memory

### **What Triggers the Transition**

**User → Kernel (Ring 3 → Ring 0):**
- **System calls**: `int 0x80` or `syscall` instruction
- **Hardware interrupts**: Timer, keyboard, network card
- **Exceptions**: Division by zero, page fault, segmentation fault

**Kernel → User (Ring 0 → Ring 3):**
- **Returning from system call**: Restores user registers and stack
- **Scheduling**: When kernel decides to run a different process
- **Signal delivery**: Kernel delivers signal to user process

### **Resource Access Control**

The kernel acts as a **gatekeeper** for all resources:

| Resource | User-Space Access | Kernel-Space Access |
|----------|-------------------|---------------------|
| Physical Memory | Via `malloc()` (virtual, limited) | Direct, unlimited via `kmalloc()` |
| Hardware Devices | Via device files (`/dev/sda`) | Direct I/O port/memory access |
| CPU Privileged Instructions | **Forbidden** (triggers exception) | Permitted ( `cli`, `sti`, `hlt` ) |
| Other Processes' Memory | **Forbidden** (can read own only) | Can read/write any process memory |
| System Time | Via `gettimeofday()` syscall | Direct hardware timer access |

### **The Role of System Calls**

System calls are the **controlled gateway** between user and kernel:
```c
// User code
fd = open("/etc/passwd", O_RDONLY);  // Library call → triggers the open/openat syscall

// Kernel syscall implementation
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
    // Kernel validates filename, checks permissions,
    // opens file, returns file descriptor
}
```

### **Why This Matters for Module Development**

Your module runs in **Ring 0** with the kernel. This means:
- **No protection**: Your bug can crash the entire system
- **Direct hardware access**: You can read/write any I/O port
- **No system calls**: You call kernel functions directly
- **Shared address space**: Your code is part of the kernel's code space

**Analogy**: User-space is a sandboxed playground. Kernel-space is the nuclear reactor control room. One wrong move has catastrophic consequences.

---

## 11. Namespace Management: Avoiding Symbol Collisions

### **The Pollution Problem**

In a kernel module, **all global symbols** are visible to the entire kernel. If you declare:
```c
int buffer_size = 4096;  // Global variable
```

This `buffer_size` could collide with **another module** or even the kernel itself that also declares `buffer_size`. The result is a **link-time error** when loading the second module.

### **Solution 1: Static Everything**

```c
// Declare all variables and functions as static
static int buffer_size = 4096;  // Only visible in this file

static int my_driver_init(void) { /* ... */ }  // Not exported
```

**Rule**: **Every** function and variable in your module should be `static` unless you explicitly want to export it.

### **Solution 2: Prefixed Naming Conventions**

Use a **unique prefix** for all non-static symbols:
```c
// Good
static int mydriver_buffer_size = 4096;
static int mydriver_irq_handler(int irq, void *dev_id);

// Bad (pollutes namespace)
static int buffer_size = 4096;  // Too generic
static int irq_handler(...);     // Conflicts likely
```

**Kernel convention**: Prefixes are **lowercase**, matching the driver name.

### **Solution 3: Symbol Tables (Advanced)**

If you must export symbols for other modules to use:
```c
// In mydriver.c
int mydriver_shared_func(void) { /* ... */ }
EXPORT_SYMBOL(mydriver_shared_func);

// In another module
extern int mydriver_shared_func(void);
```

**Viewing exported symbols**:
```bash
# Symbols exported by a loaded module
cat /sys/module/mydriver/sections/__ksymtab
```

### **What About `/proc/kallsyms`?**

```bash
$ cat /proc/kallsyms | grep mydriver
ffffffffc0123000 t mydriver_init    [mydriver]  # 't' = local symbol
ffffffffc0123050 T mydriver_func    [mydriver]  # 'T' = global/exported
```

**Lowercase 't'**: Symbol is **static** (local to module)
**Uppercase 'T'**: Symbol is **global** (exported via EXPORT_SYMBOL)

**Best Practice**: Your module should show **only lowercase symbols** in `/proc/kallsyms` unless you have a specific reason to export.

---

## 12. Code Space: Where Your Module Lives

### **The Shared Code Space Reality**

Unlike user-space processes with **isolated virtual address spaces**, kernel modules share **one** code space with the entire kernel. When you `insmod`, your code is **physically linked** into the running kernel image.

### **Consequences of Shared Space**

1. **No memory protection**: Your module can:
   - **Write over kernel code**: `memset(0xffffffff81000000, 0, 0x1000)` crashes system
   - **Corrupt kernel data**: Bad pointer can overwrite scheduler queues
   - **Access any kernel symbol**: Including internal (non-exported) ones (if you're clever/careless)

2. **No separate address space**: The kernel does **not** set aside a protected region for your module. Your `.text`, `.data`, and `.bss` sections are just more kernel memory.

3. **Segmentation faults = system crash**: There's no SIGSEGV handler in kernel space. A bad pointer dereference triggers an **Oops** or **Panic**, freezing the entire machine.

### **Memory Layout of a Loaded Module**

```bash
# View where your module lives in kernel memory
$ sudo cat /sys/module/hello_1/sections/.text
0xffffffffc0123000

$ sudo cat /sys/module/hello_1/sections/.data
0xffffffffc0124000

$ sudo cat /sys/module/hello_1/sections/.bss
0xffffffffc0125000
```

**These are real kernel addresses**, not virtual sandboxed addresses like in user-space.

### **The "Segmentation Fault" Translation**

**User-space segfault:**
```
Process: *writes to 0x00000000* → CPU trap → Kernel sends SIGSEGV → Process dies
System: Still running
```

**Kernel-space "segfault":**
```
Module: *writes to 0x00000000* → CPU trap → **No handler** → **Kernel Oops** → System freeze
```

**Oops Example:**
```
[   12.345] BUG: unable to handle kernel NULL pointer dereference at 0000000000000000
[   12.346] IP: hello_bad+0x5/0x20 [hello_bad]
[   12.347] Oops: 0002 [#1] SMP
[   12.348] RIP: 0010:hello_bad+0x5/0x20
[   12.349] Code: Bad RIP value.
```

### **Microkernels vs. Monolithic Kernels**

**Linux (Monolithic):**
- All code (kernel + modules) shares one address space
- **Advantage**: Speed (no context switches for kernel calls)
- **Disadvantage**: One bug kills entire system

**Microkernel (e.g., Zircon, GNU Hurd):**
- Each driver has **private code space**
- **Advantage**: Fault isolation (driver crash doesn't kill kernel)
- **Disadvantage**: Slower (IPC messages between spaces)

Linux chose performance over isolation, making **correctness critical**.

---

## 13. Introduction to Device Drivers: From Modules to Hardware

### **What is a Device Driver?**

A **device driver** is a specialized kernel module that creates a **software interface** for hardware devices. It translates between:
- **User-space programs**: Read/write device files (`/dev/sda`, `/dev/ttyS0`)
- **Hardware**: Memory-mapped registers, I/O ports, DMA, interrupts

### **Device Files: The User Interface**

All hardware is represented by files in `/dev`:

```bash
# Block devices (storage)
$ ls -l /dev/sda[1-3]
brw-rw---- 1 root disk 8, 1 Apr  9 2025 /dev/sda1
brw-rw---- 1 root disk 8, 2 Apr  9 2025 /dev/sda2
brw-rw---- 1 root disk 8, 3 Apr  9 2025 /dev/sda3
```

**Device numbers**: `<major>, <minor>`
- `8, 1` = Major 8, Minor 1
- `8, 2` = Major 8, Minor 2

### **Major and Minor Numbers**

**Major Number**: Identifies the **driver** that handles the device.
- All devices with **major 8** are controlled by the **SCSI disk driver**
- Major numbers are allocated centrally: see `Documentation/admin-guide/devices.txt`

**Minor Number**: Identifies **which specific device** the driver should access.
- Minor 0 = First SCSI disk (`/dev/sda`)
- Minor 16 = Second SCSI disk (`/dev/sdb`)
- Minor 1 = First partition of first disk (`/dev/sda1`)

### **Character vs. Block Devices**

| Feature | Character Device | Block Device |
|---------|------------------|--------------|
| **Access** | Byte streams (any size) | Fixed-size blocks (512B, 4KB) |
| **Buffering** | No kernel buffer | Has request queue and buffer cache |
| **Performance** | Simpler, lower overhead | Optimized for throughput |
| **Seeking** | Not needed | Random access, seeks optimized |
| **Examples** | Serial ports (`/dev/ttyS0`), keyboards | Hard drives (`/dev/sda`), SSDs |

**Identification**: First character in `ls -l` output:
- `c` = character device
- `b` = block device

### **Creating Device Files Manually**

```bash
# Create a char device with major 12, minor 2
sudo mknod /dev/mydevice c 12 2

# Create a block device with major 8, minor 32
sudo mknod /dev/mydisk b 8 32

# Permissions
sudo chmod 666 /dev/mydevice  # Read/write for all
```

**Convention**: Put device files in `/dev`, but for testing you can put them in your build directory.

### **The Driver's Role**

When a user program does:
```c
fd = open("/dev/sda1", O_RDONLY);
read(fd, buffer, 4096);
```

The kernel:
1. Sees major 8 on `/dev/sda1`
2. Looks up driver registered for major 8 (`sd` driver)
3. Calls driver's `open()` method with minor=1
4. Driver uses minor to select first partition of first disk
5. Calls `read()` method → translates to SCSI commands → hardware
6. Returns data to user

### **Abstract "Hardware"**

**Not just physical cards**: "Hardware" can be:
- **Physical**: PCI card, USB device
- **Virtual**: Loop devices (`/dev/loop0`), RAM disks (`/brd/brd0`)
- **Pseudo**: `/dev/null`, `/dev/random` (generate data algorithmically)

Two device files with same major but different minors can represent:
- Different partitions on same disk (`/dev/sda1`, `/dev/sda2`)
- Different functions of same card (`/dev/video0`, `/dev/vbi0`)

### **Device Numbers in Code**

```c
// In driver registration
static int my_major = 240;  // Request specific major
static int my_minor = 0;    // Starting minor

register_chrdev(my_major, "mydriver", &fops);
// Creates /dev/mydriver with major 240, minor 0

// Or use dynamic allocation
my_major = register_chrdev(0, "mydriver", &fops);
// Kernel assigns unused major number
```

### **Looking Up Assigned Major Numbers**

```bash
# Current major number assignments
cat /usr/src/linux/Documentation/admin-guide/devices.txt | grep -A 5 "Block devices"

# Or online:
# https://www.kernel.org/doc/Documentation/admin-guide/devices.txt
```

**Key ranges**:
- `0-199`: Reserved for traditional devices
- `200-234`: Reserved for local/experimental use
- `235-254`: Dynamic assignment
- ` major` : Reserved for miscellaneous

---

## 14. Command-Line Parameter Passing

### **The `module_param()` Framework**

User-space programs use `argc/argv`. Kernel modules use the `module_param()` macro.

### **Parameter Types**

```c
static int myint = 420;
static long mylong = 9999;
static char *mystring = "default";
static short myshort = 1;
static int myarray[2] = { 1, 2 };

module_param(myint, int, 0644);
module_param(mylong, long, 0444);
module_param(mystring, charp, 0000);  // charp = char pointer
module_param(myshort, short, 0644);
module_param_array(myarray, int, NULL, 0644);  // NULL = don't track count
```

### **Permission Bits (The Third Parameter)**

Permissions control **sysfs** visibility (we'll cover sysfs in later chapters):
- `0000`: No sysfs file (parameter only at load time)
- `0644` (`S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH`): Readable by all, writable by owner
- `0444`: Read-only
- `0200`: Write-only (rare)

**Security Note**: Writable parameters after module load can be dangerous. Use `0000` for sensitive values.

### **`module_param_array()` Details**

```c
static int myintarray[4];
static int count;

module_param_array(myintarray, int, &count, 0644);
MODULE_PARM_DESC(myintarray, "An array of up to 4 integers");
```

- `&count`: Kernel stores how many values user provided
- **No bounds checking**: Kernel trusts you declared array large enough
- **Overflow** = buffer overrun = security hole

### **`MODULE_PARM_DESC()` Documentation**

```c
static int debug = 0;
module_param(debug, int, 0644);
MODULE_PARM_DESC(debug, "Enable verbose debugging (0=off, 1=on)");
```

Shows up in `modinfo`:
```
$ modinfo hello-5.ko
parm:           debug:int:Enable verbose debugging (0=off, 1=on)
```

### **hello-5.c: Complete Parameter Example**
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/printk.h>
#include <linux/kernel.h> /* for ARRAY_SIZE */

MODULE_LICENSE("GPL");

static short int myshort = 1;
static int myint = 420;
static long int mylong = 9999;
static char *mystring = "blah";
static int myintarray[2] = { 420, 420 };
static int arr_argc = 0;

module_param(myshort, short, S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP);
MODULE_PARM_DESC(myshort, "A short integer");

module_param(myint, int, S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH);
MODULE_PARM_DESC(myint, "An integer");

module_param(mylong, long, S_IRUSR);
MODULE_PARM_DESC(mylong, "A long integer");

module_param(mystring, charp, 0000);
MODULE_PARM_DESC(mystring, "A character string");

module_param_array(myintarray, int, &arr_argc, 0000);
MODULE_PARM_DESC(myintarray, "An array of integers");

static int __init hello_5_init(void)
{
    int i;
    pr_info("Hello, world 5\n=============\n");
    pr_info("myshort: %hd\n", myshort);
    pr_info("myint: %d\n", myint);
    pr_info("mylong: %ld\n", mylong);
    pr_info("mystring: %s\n", mystring);
    
    for (i = 0; i < ARRAY_SIZE(myintarray); i++)
        pr_info("myintarray[%d] = %d\n", i, myintarray[i]);
    
    pr_info("got %d arguments for myintarray\n", arr_argc);
    return 0;
}

static void __exit hello_5_exit(void)
{
    pr_info("Goodbye, world 5\n");
}

module_init(hello_5_init);
module_exit(hello_5_exit);
```

### **Loading with Parameters**

```bash
# Override parameters at load time
sudo insmod hello-5.ko mystring="custom" myint=123 myintarray=10,20,30

# View results
dmesg | tail

# Example output:
# mystring: custom
# myint: 123
# myintarray[0] = 10
# myintarray[1] = 20
# got 2 arguments for myintarray  # Array capped at declared size!

# Type mismatch fails immediately
sudo insmod hello-5.ko mylong=hello
# insmod: ERROR: Invalid parameters
```

---

## 15. Multi-File Module Architecture

### **When to Split Modules**

Split a module into multiple files when:
- Code exceeds 500-1000 lines (maintainability)
- Multiple logical components (core driver + helper functions)
- Shared code between related modules
- Platform-specific code separated from common code

### **Example: start.c + stop.c**

**start.c** (initialization):
```c
#include <linux/kernel.h>
#include <linux/module.h>

int init_module(void)
{
    pr_info("Hello, world - this is the kernel speaking\n");
    return 0;
}

MODULE_LICENSE("GPL");
```

**stop.c** (cleanup):
```c
#include <linux/kernel.h>
#include <linux/module.h>

void cleanup_module(void)
{
    pr_info("Short is the life of a kernel module\n");
}

MODULE_LICENSE("GPL");
```

### **The Combined Makefile**

```makefile
obj-m += startstop.o  # Single module name
startstop-objs := start.o stop.o  # List of constituent objects

PWD := $(CURDIR)

all:
	$(MAKE) -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	$(MAKE) -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

### **How It Works**

- `obj-m += startstop.o` tells kbuild to create **one module** called `startstop.ko`
- `startstop-objs` lists the `.o` files to **link together** into that module
- The kernel build system automatically links `start.o` and `stop.o` into a single `startstop.ko`
- `insmod startstop.ko` loads the combined module

### **Advanced Multi-File Patterns**

For a driver with core + hardware-specific parts:
```makefile
obj-m += mydriver.o
mydriver-objs := driver-core.o driver-pci.o driver-usb.o

# Optionally build only one variant:
obj-$(CONFIG_PCI) += driver-pci.o
obj-$(CONFIG_USB) += driver-usb.o
```

---

## 16. Building for Precompiled Kernels

### **The Version Magic Problem**

When you try to load a module compiled against different kernel source:

```bash
$ sudo insmod poet.ko
insmod: ERROR: could not insert module poet.ko: Invalid module format

$ dmesg
poet: disagrees about version of symbol module_layout
```

**Root cause**: The `vermagic` string in your module doesn't match the running kernel's `vermagic`.

**Vermagic components**:
- Kernel version (`5.4.0-70-generic`)
- Build options (`SMP`, `preempt`, `mod_unload`, `modversions`)
- **EXTRAVERSION** from Makefile
- GCC version used to compile the kernel

### **Solution: Build in Identical Environment**

**Step 1: Get exact kernel config**
```bash
# Copy the config from your running kernel
cp /boot/config-$(uname -r) /usr/src/linux-headers-$(uname -r)/.config
```

**Step 2: Align Makefile version**
```bash
# Check your kernel's EXTRAVERSION
cat /lib/modules/$(uname -r)/build/Makefile | grep EXTRAVERSION

# If your kernel source Makefile differs, edit it to match
vim /usr/src/linux-headers-$(uname -r)/Makefile
# Change EXTRAVERSION to match exactly
```

**Step 3: Prepare build environment**
```bash
cd /usr/src/linux-headers-$(uname -r)

# Update generated headers without full compile
make modules_prepare

# OR let it start compiling and interrupt after SPLIT
make
# Wait for "SPLIT" message, then Ctrl-C
```

**Step 4: Build your module**
```bash
cd ~/my-module
make -C /lib/modules/$(uname -r)/build M=$(PWD) modules
```

### **Alternative: Use Distribution Packages**

**Ubuntu/Debian:**
```bash
sudo apt-get install linux-headers-$(uname -r)
# This already includes matching config and prepared headers
```

**Why this works**: Distribution headers are **pre-configured** with the exact `vermagic` and `Module.symvers` from the installed kernel. You don't need the full kernel source.

### **The Danger of `--force-vermagic`**

```bash
sudo insmod --force-vermagic poet.ko
```

**NEVER do this in production**. It bypasses CRC and version checks, risking memory corruption if kernel APIs changed. Only use for emergency testing on disposable VMs.

---

## 17. Troubleshooting Common Errors

### **Error: "Invalid module format"**

**Symptoms:**
- `insmod` fails immediately
- `dmesg` shows version mismatch

**Causes & Fixes:**
1. **Wrong kernel headers**: `uname -r` mismatch. Install correct headers.
2. **Kernel upgraded, not rebooted**: Reboot to match running kernel.
3. **EXTRAVERSION mismatch**: Check and align Makefile EXTRAVERSION.
4. **SecureBoot enabled**: Disable in BIOS/UEFI.

### **Error: "Unknown symbol"**

**Symptoms:**
```
dmesg: mymodule: Unknown symbol kmalloc (err -2)
```

**Causes & Fixes:**
1. **Missing MODULE_LICENSE**: Add `MODULE_LICENSE("GPL")`.
2. **Missing dependency**: Load dependency first: `sudo modprobe dependee`.
3. **CRC mismatch**: Rebuild module on target machine.
4. **Built-in vs. module**: Symbol compiled out of kernel or as module.

### **Error: "Module is in use"**

**Symptoms:**
```
rmmod: ERROR: Module hello is in use
```

**Diagnosis:**
```bash
# Find what's using it
lsmod | grep hello
# "Used by" column shows count and dependencies

# Check open files
sudo lsof /dev/hello_device

# Check kernel references
sudo cat /sys/module/hello/refcnt
```

**Fix:**
- Wait for processes to release the device
- Close applications using the driver
- **Last resort**: `sudo rmmod -f hello` (**DANGEROUS**)

### **Error: No output from `printk`**

**Diagnosis:**
```bash
# Check console log level
cat /proc/sys/kernel/printk
# Default: 4 4 1 7 (only priorities >= 4 shown)

# Lower threshold to see all messages
echo 7 4 1 7 | sudo tee /proc/sys/kernel/printk
```

**Fix**: Use higher priority or adjust log level:
```c
printk(KERN_ALERT "This will always show\n");
```

---

## 18. Coding Standards and Best Practices

### **Indentation: Tabs, Not Spaces**

The Linux kernel **mandates tabs** (8-character width). Violations will be rejected if you submit upstream.

**Correct:**
```c
static int __init my_init(void)
{
	pr_info("Hello\n");  // Tab before pr_info
	return 0;
}
```

**Incorrect:**
```c
static int __init my_init(void)
{
    pr_info("Hello\n");  // Spaces - WRONG!
    return 0;
}
```

Configure your editor:
- **Vim**: `set ts=8 sw=8 noexpandtab`
- **Emacs**: `(setq indent-tabs-mode t)`
- **VS Code**: `"editor.insertSpaces": false, "editor.tabSize": 8`

### **Naming Conventions**

**Functions:**
```c
drivername_component_action()
// Examples:
usb_storage_probe()
e1000e_open()
```

**Variables:**
```c
descriptive_names_like_this
// NOT: camelCase or PascalCase
```

**Macros/Constants:**
```c
UPPER_CASE_WITH_UNDERSCORES
// Examples:
MAX_DEVICE_COUNT
IRQ_HANDLED
```

### **Print Macro Usage**

**Use specific macros, not raw printk:**
- `pr_info()` - Informational
- `pr_err()` - Errors
- `pr_warn()` - Warnings
- `pr_debug()` - Debug (can be compiled out)
- `dev_info()` - For device-specific messages (covered later)

**Include context:**
```c
// Bad:
pr_info("Failed\n");

// Good:
pr_info("hello-5: Failed to allocate buffer, size=%zu\n", size);
```

### **Error Handling Pattern**

**Always check allocations:**
```c
struct my_struct *data = kmalloc(sizeof(*data), GFP_KERNEL);
if (!data) {
	pr_err("hello: Out of memory\n");
	return -ENOMEM;
}
```

**Return negative errno codes** from init:
```c
if (hardware_not_found) {
	pr_err("Device not detected\n");
	return -ENODEV;  // No such device
}
```

---

## 19. From Hello World to Device Drivers

### **Conceptual Bridge**

Hello World modules teach you:
- ✅ Module lifecycle (load/unload)
- ✅ Kernel logging
- ✅ Build system
- ✅ Parameter passing
- ✅ Multi-file organization

Device drivers add:
- **Device registration**: `register_chrdev()`, `pci_register_driver()`
- **Hardware interaction**: I/O ports, memory-mapped registers, DMA
- **Interrupt handling**: `request_irq()`, interrupt handlers
- **User-space interface**: `read()`, `write()`, `ioctl()` system calls
- **Concurrency**: Locks, semaphores, RCU
- **Power management**: Suspend/resume callbacks
- **Device trees / ACPI**: Platform description

### **Next Steps on Your Journey**

1. **Character Devices**: Implement `open()`, `read()`, `write()`, `release()` callbacks
2. **Platform Drivers**: Learn device tree bindings for embedded systems
3. **PCI/USB Drivers**: Understand bus enumeration and automatic binding
4. **Interrupts**: Write ISR (Interrupt Service Routines) with proper locking
5. **Memory Management**: DMA buffers, coherent vs. streaming mappings
6. **Sysfs Integration**: Expose runtime controls to user space
7. **Testing**: Use `kunit` for unit tests, `kmemleak` for leak detection

### **Recommended Reading Order**

1. (This guide) Chapters 1-7: Master hello world and basic module mechanics
2. Chapters 8-12: Character devices and file operations
3. Chapters 13-15: Concurrency, locking, and advanced topics
4. PCI/USB specific chapters: For hardware driver writers
5. [Linux Device Drivers, 3rd Edition](https://lwn.net/Kernel/LDD3/) (free online) - Classic reference

---

## 20. Exercise: Experiment with Return Values

**Task**: In `hello-1.c`, change the return value in `init_module()` to a negative number:
```c
int init_module(void)
{
    pr_info("Hello world 1.\n");
    return -EINVAL;  // Invalid argument error
}
```

**Expected behavior:**
```bash
$ sudo insmod hello-1.ko
insmod: ERROR: could not insert module hello-1.ko: Invalid parameters

$ dmesg | tail
Hello world 1.
init_module returned -22 (Invalid argument)
module initialization failed, module not loaded
```

**Why**: Negative return codes from init are treated as **errno** values. The kernel:
1. Calls your `init_module()`
2. Sees negative return
3. Logs the error
4. **Unloads the module** without calling cleanup
5. Returns error to user space

**Lesson**: Always return 0 on success. Use standard errno codes like `-ENOMEM`, `-ENODEV`, `-EINVAL` for specific failures.

---

## 📚 Quick Reference: Module Commands

| Command | Purpose | Example |
|---------|---------|---------|
| `make` | Compile module | `make` |
| `make clean` | Remove build artifacts | `make clean` |
| `modinfo` | Show module metadata | `modinfo hello.ko` |
| `sudo insmod` | Load module (no deps) | `sudo insmod hello.ko` |
| `sudo modprobe` | Load module (with deps) | `sudo modprobe hello` |
| `sudo rmmod` | Remove module | `sudo rmmod hello_1` |
| `sudo modprobe -r` | Remove module + deps | `sudo modprobe -r hello` |
| `lsmod` | List loaded modules | `lsmod \| grep hello` |
| `dmesg` | View kernel messages | `dmesg \| tail -f` |
| `journalctl -k` | View kernel logs (systemd) | `journalctl -k -f` |

