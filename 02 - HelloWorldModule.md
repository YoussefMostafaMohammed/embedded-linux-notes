# The Complete Guide to Writing Your First Linux Kernel Modules

## 📑 Table of Contents

1. [Your First Module: The Minimal Hello World](#1-your-first-module-the-minimal-hello-world)
2. [The Makefile: Your Build Foundation](#2-the-makefile-your-build-foundation)
3. [Compilation to Execution: Full Workflow](#3-compilation-to-execution-full-workflow)
4. [Module Metadata and Information](#4-module-metadata-and-information)
5. [Modern Module Initialization](#5-modern-module-initialization)
6. [Memory Optimization with `__init` and `__exit`](#6-memory-optimization-with-__init-and-__exit)
7. [Documentation and Licensing Macros](#7-documentation-and-licensing-macros)
8. [Command-Line Parameter Passing](#8-command-line-parameter-passing)
9. [Multi-File Module Architecture](#9-multi-file-module-architecture)
10. [Building for Precompiled Kernels](#10-building-for-precompiled-kernels)
11. [Troubleshooting Common Errors](#11-troubleshooting-common-errors)
12. [Coding Standards and Best Practices](#12-coding-standards-and-best-practices)
13. [From Hello World to Device Drivers](#13-from-hello-world-to-device-drivers)

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

## 8. Command-Line Parameter Passing

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

## 9. Multi-File Module Architecture

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

## 10. Building for Precompiled Kernels

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

## 11. Troubleshooting Common Errors

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

## 12. Coding Standards and Best Practices

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

## 13. From Hello World to Device Drivers

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

## 📚 Exercise: Experiment with Return Values

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

## 🔧 Quick Reference: Module Commands

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
