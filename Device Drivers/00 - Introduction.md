# The Definitive Linux Kernel Module Programming Guide

*A comprehensive resource for understanding and developing Linux kernel modules and device drivers*

## 📑 Table of Contents

1. [Introduction: What Are Kernel Modules?](#1-introduction-what-are-kernel-modules)
2. [Kernel Space vs. User Space: Understanding the Stakes](#2-kernel-space-vs-user-space-understanding-the-stakes)
3. [Essential Tools and Commands](#3-essential-tools-and-commands)
4. [Prerequisites: Kernel Headers and Build Environment](#4-prerequisites-kernel-headers-and-build-environment)
5. [The Golden Rule: Development in a Virtual Machine](#5-the-golden-rule-development-in-a-virtual-machine)
6. [Module Dependencies: The Hidden Web](#6-module-dependencies-the-hidden-web)
7. [Module Versioning: The CRC Safety Mechanism](#7-module-versioning-the-crc-safety-mechanism)
8. [Inside a .ko File: ELF Sections Demystified](#8-inside-a-ko-file-elf-sections-demystified)
9. [Step-by-Step: Compilation to Execution](#9-step-by-step-compilation-to-execution)
10. [Security: SecureBoot and Signed Modules](#10-security-secureboot-and-signed-modules)
11. [Debugging: Where's My Output?](#11-debugging-wheres-my-output)
12. [Critical Warnings and Best Practices](#12-critical-warnings-and-best-practices)
13. [Troubleshooting Common Issues](#13-troubleshooting-common-issues)

---

## 1. Introduction: What Are Kernel Modules?

A kernel module is a code segment capable of dynamic loading and unloading within the running Linux kernel without requiring a system reboot. Think of it as a **plugin architecture for the kernel itself**.

**Why this matters:**
- **On-demand functionality**: Load a USB webcam driver only when you plug it in
- **Memory efficiency**: Keep the kernel lean by loading only necessary components
- **Rapid development**: Test new drivers without recompiling the entire kernel
- **Hardware support**: Add drivers for new devices without rebuilding your system

**The Alternative (Monolithic Kernel):** Without modules, every driver and feature must be statically compiled into the kernel image (`/boot/vmlinuz`). Adding a new driver requires:
1. Reconfiguring the entire kernel build
2. Recompiling 20-50 million lines of code (2-4 hours)
3. Installing the new kernel
4. Rebooting the system

This approach creates massive kernel binaries (100MB+) and makes testing impractical. Modern Linux uses a **hybrid architecture**: a relatively small core kernel with dynamically loadable modules for everything else.

---

## 2. Kernel Space vs. User Space: Understanding the Stakes

| Aspect | User-Space Program | Kernel Module |
|--------|-------------------|---------------|
| **Memory Access** | Limited to its own virtual address space | Unrestricted access to all system memory |
| **Crash Consequence** | Program terminates, system continues | System freeze, potential filesystem corruption, mandatory reboot |
| **Debugging** | GDB, core dumps, easy restart | Limited tools, often requires serial console, crashes the machine |
| **API** | Standard C library (`printf`, `malloc`) | Kernel API only (`printk`, `kmalloc`) |
| **Privileges** | Restricted by OS | Full hardware access, can bypass all security |

**The Critical Difference:** In user space, the kernel acts as a safety supervisor. In kernel space, **you are the kernel**. An unregulated pointer dereference can overwrite critical data structures, corrupt the filesystem cache, or trigger hardware malfunction. There is no memory protection, no segmentation fault safety net—just immediate and catastrophic failure.

---

## 3. Essential Tools and Commands

The `kmod` package provides the core utilities for module management:

### **Primary Commands**

-  **`insmod <module.ko>`**  : Loads a single module file **without checking dependencies**. If dependencies are missing, it fails with undefined symbol errors.
  
-  **`modprobe <module_name>`**  : The intelligent loader. It:
  1. Looks up the module in `/lib/modules/$(uname -r)/`
  2. Reads the dependency database (`modules.dep`)
  3. Loads all required dependencies in the correct order
  4. Then loads your module

-  **`rmmod <module_name>`**  : Unloads a module. Fails if other modules depend on it.

-  **`modinfo <module.ko>`**  : Displays metadata from the `.modinfo` section (author, license, dependencies, etc.).

### **Monitoring Commands**

-  **`lsmod`**  : Lists all loaded modules. Output format:
  ```
  Module                  Size  Used by
  ath9k                 159744  0
  ath                    32768  1 ath9k
  mac80211              892928  1 ath9k
  ```
  Shows module size and dependency relationships ("Used by").

-  **`cat /proc/modules`**  : The raw, kernel-space data that `lsmod` parses. Shows numeric values and load state flags.

-  **`dmesg | tail -f`**  : **Critical for debugging**. Shows kernel ring buffer messages in real-time.

-  **`journalctl -k -f`**  : Alternative to `dmesg` on systemd systems, showing kernel logs.

**Example workflow:**
```bash
# Install tools (Ubuntu/Debian)
sudo apt-get install build-essential kmod

# See what modules are loaded
lsmod | grep usb

# Load a module with dependencies
sudo modprobe usb-storage

# Check the logs
dmesg | tail
```

---

## 4. Prerequisites: Kernel Headers and Build Environment

**What are kernel headers?**  
Headers are the kernel's **API contract**—C header files (`*.h`) that define:
- Function prototypes (`kmalloc`, `printk`, `copy_from_user`)
- Data structures (`struct file`, `struct sk_buff`)
- Constants and macros (`GFP_KERNEL`, `MODULE_LICENSE`)

**Why not the full kernel source?**  
The full source is 1.5GB+. Headers are ~50MB and contain only the interface definitions needed for compilation, not implementation details.

### **Installation Commands**

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install build-essential kmod
sudo apt-get install linux-headers-$(uname -r)
```

**Arch Linux:**
```bash
sudo pacman -S gcc kmod linux-headers
```

**Fedora:**
```bash
sudo dnf install kernel-devel kernel-headers
```

### **Verification**
```bash
# Check if headers match your running kernel
ls /lib/modules/$(uname -r)/build
# Should exist and contain Makefile, include/, Module.symvers
```

**Critical:** The headers **must** match your *running* kernel version exactly. If you upgrade your kernel but don't reboot, you're still running the old kernel while headers point to the new version—this causes CRC mismatches.

---

## 5. The Golden Rule: Development in a Virtual Machine

 **⚠️ NEVER develop kernel modules on your primary machine**  .

### **Why VMs are Mandatory**

| Scenario | On Bare Metal | In a VM |
|----------|---------------|---------|
| Kernel crash | Full system freeze, potential data loss, forced hard reset | VM window freezes, close and restart VM |
| Filesystem corruption | Possible, requiring fsck and recovery | Isolated to virtual disk, snapshot rollback |
| Driver conflicts | Can render system unbootable | Snapshot before testing, instant rollback |
| SecureBoot testing | Risk of bricking hardware | Disable in VM firmware with no risk |

### **Recommended Setup**
1. **VirtualBox** or **QEMU/KVM** (free, snapshots supported)
2. Install minimal Ubuntu/Debian server (no GUI needed)
3. Create a snapshot after OS installation and header setup
4. Test modules → if crash → restore snapshot in 30 seconds
5. Use SSH from host for convenience: `ssh user@vm-ip`

### **Console vs. X11 Warning**

**Never compile or test modules from a terminal inside X11 (GNOME, KDE, etc.).** Here's why:

- The graphics driver (`nvidia`, `amdgpu`, `i915`) is a **kernel module**
- If your test module crashes the kernel, the graphics driver dies
- Result: Frozen GUI, can't see `dmesg`, can't switch to text console (Ctrl+Alt+F2 may not respond)
- You're blind and must hard-reset

**Solution:** Use a raw **VT console**:
```bash
# Switch to text console
Ctrl+Alt+F2  (F2 through F6 available)

# Switch back to X11 (if it still works)
Ctrl+Alt+F1 or F7
```

Or develop via **SSH** from another machine. If the kernel panics, the SSH session dies, but your terminal remains functional.

---

## 6. Module Dependencies: The Hidden Web

**What are dependencies?**  
Modules are not islands. A module can **export symbols** (functions, variables) that other modules use. This creates a dependency graph.

**Example dependency chain:**
```
[your_wifi_driver.ko] → depends on → [mac80211.ko]
  ↓
[mac80211.ko] → depends on → [cfg80211.ko]
  ↓
[cfg80211.ko] → depends on → [kernel core]
```

**The depmod Database**  
After installing any module, `depmod` runs automatically to scan all `.ko` files and build:

```
/lib/modules/$(uname -r)/modules.dep
```

This is a plain-text map:
```
kernel/drivers/net/wireless/ath/ath9k.ko:
  kernel/drivers/net/wireless/ath/ath.ko
  kernel/net/mac80211.ko
  kernel/net/wireless/cfg80211.ko
```

**How `modprobe` uses this:**
1. You run: `sudo modprobe ath9k`
2. `modprobe` reads `modules.dep` and sees ath9k needs ath, mac80211, cfg80211
3. It loads cfg80211 (no dependencies)
4. Loads mac80211 (depends on cfg80211, already loaded)
5. Loads ath (depends on mac80211, already loaded)
6. Finally loads ath9k

**What if dependencies are missing?**  
`insmod` fails with:  
```
insmod: ERROR: could not insert module hello.ko: Invalid module format
dmesg: hello: Unknown symbol kmalloc (err -2)
```

`modprobe` prevents this by loading the entire dependency tree automatically.

---

## 7. Module Versioning: The CRC Safety Mechanism

### **The Problem**

Assume kernel 6.8.0 defines:
```c
void *kmalloc(size_t size, gfp_t flags);
```

Kernel 6.8.1 changes it to:
```c
void *kmalloc(size_t size, gfp_t flags, int node);
```

If you load a module compiled for 6.8.0 into 6.8.1:
- Module passes 2 arguments
- Function expects 3 arguments
- Stack corruption → instant crash or silent data corruption

### **The Solution: CONFIG_MODVERSIONS**

This kernel option makes the build system generate a **CRC32 checksum** for every exported symbol's prototype. The checksum acts as a fingerprint of the function's signature.

**The Three-Stage Process:**

#### **Stage 1: Kernel Compilation**
When the kernel is built:
- For every `EXPORT_SYMBOL(function_name)`, the build system calculates a CRC from the function's prototype
- Creates `/lib/modules/$(uname -r)/build/Module.symvers`:
  ```
  0x35a4d4c8  kmalloc    vmlinux EXPORT_SYMBOL
  0x1b6a7d8e  printk     vmlinux EXPORT_SYMBOL
  ```

#### **Stage 2: Module Compilation**
When you compile `hello.c`:
1. The build system detects you call `kmalloc` and `printk`
2. It looks up their CRCs in `Module.symvers`
3. Creates a **list** in your `.ko` file (the `__versions` section):
   ```
   Required Symbol | Expected CRC
   ----------------------------
   kmalloc         | 0x35a4d4c8
   printk          | 0x1b6a7d8e
   ```

#### **Stage 3: Module Loading**
When you run `sudo insmod hello.ko`:
1. Kernel reads the `__versions` section from your `.ko` file
2. For each symbol, it looks up the **current CRC** in its in-memory symbol table
3. If **any** CRC differs: **Load rejected** with:
   ```
   insmod: ERROR: could not insert module hello.ko: Invalid module format
   dmesg: hello: disagrees about version of symbol kmalloc
   ```
4. If all CRCs match: Load proceeds, and the kernel patches actual function addresses into your code

### **Why Not a Single Kernel CRC?**
A single kernel-wide CRC would be too coarse. If *any* of the 50,000 kernel symbols change, all modules would break—even if they don't use that symbol. Per-symbol CRCs provide **granular compatibility** checking.

### **Common Mismatch Scenarios**
- **Kernel upgraded but not rebooted**: Headers are for the new kernel, but you're still running the old one
- **Different kernel flavor**: Compiled for `generic` but running `lowlatency`
- **Cross-machine copy**: Module from another PC with a different kernel patch level

**Solution:** Always compile on the **same machine** where you'll load the module, using `linux-headers-$(uname -r)`.

---

## 8. Inside a .ko File: ELF Sections Demystified

A `.ko` file is an **ELF (Executable and Linkable Format)** object. Think of it as a filing cabinet with labeled drawers called **sections**. Each section serves a specific purpose.

### **Key Sections in a Kernel Module**

| Section Name | Content | Purpose |
|--------------|---------|---------|
| **`.text`** | Your compiled machine code | The actual instructions for `hello_init()`, `hello_exit()`, etc. |
| **`.data`** | Initialized global variables | `int my_var = 42;` lives here |
| **`.bss`** | Uninitialized global variables | `int my_uninit_var;` (zero-initialized at load) |
| **`.modinfo`** | Metadata strings | License, author, description, dependencies, `vermagic` |
| **`__versions`** | Symbol→CRC mapping table | Only present with `CONFIG_MODVERSIONS` enabled |
| **`.symtab`** | Symbol table | For debugging (function names, addresses) |
| **`.rodata`** | Read-only data | String constants, `printk` format strings |
| **`.gnu.linkonce.this_module`** | Module descriptor | Tells kernel where `init` and `exit` functions are |

### **Viewing Sections**
```bash
# See all sections with sizes
objdump -h hello.ko

# View .modinfo content
modinfo hello.ko

# Dump raw __versions section (requires kernel debug tools)
hexdump -C hello.ko | grep -A 20 __versions
```

### **The Placeholder Mechanism**

When you call `kmalloc()` in your code, the compiler doesn't embed the actual `kmalloc` function. Instead, it creates **relocation entries**—placeholders that say:
- *"At offset 0x124 in .text, I need the address of symbol 'kmalloc'"*

The kernel loader **patches these placeholders** with real addresses when加载 the module. This is why modules are small: they only contain *your* code, not the entire kernel.

---

## 9. Step-by-Step: Compilation to Execution

### **Phase 1: Compilation (User Space)**

**Step 0: Source File `hello.c`**
```c
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void) {
    printk(KERN_INFO "Hello, kernel!\n");
    void *ptr = kmalloc(1024, GFP_KERNEL);
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye, kernel!\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

**Step 1: `make` Invokes Kernel Build System**
```bash
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
```
- `-C`: Uses kernel's Makefile (knows how to compile modules)
- `M=$(pwd)`: Tells it where your source is
- Reads `/lib/modules/$(uname -r)/build/Makefile` for compiler flags

**Step 2: Preprocessing**
- `#include <linux/module.h>`: Pulls in 50+ header files, defines `module_init` macro
- `#include <linux/kernel.h>`: Declares `kmalloc()`, `printk()`
- Resolves all macros and creates preprocessed `.i` file (temporary)

**Step 3: Compilation**
- `gcc` compiles `hello.c` → `hello.o` (object file)
- Object file contains **unresolved symbols**: `kmalloc`, `printk`, `module_init`
- Machine code is position-independent (can be loaded anywhere in memory)

**Step 4: Linking (The Magic)**
- Linker (`ld`) runs with kernel-specific scripts
- Reads `/lib/modules/$(uname -r)/build/Module.symvers`
- For each unresolved symbol, finds its CRC
- Creates **two special sections** in `hello.ko`:
  - `__versions`: Table of `symbol → expected_CRC`
  - `.modinfo`: String table with metadata
- Symbol addresses are **not resolved yet**—just placeholders

### **Phase 2: Loading (Kernel Space)**

**Step 5: `sudo insmod hello.ko`**
- `insmod` opens the file, reads entire `.ko` into user-space memory
- Makes `init_module()` system call, passing pointer to module data

**Step 6: Kernel Verification**
Kernel's `load_module()` function:
1. **ELF Validation**: Checks file is valid ELF format
2. **License Check**: Verifies MODULE_LICENSE is acceptable (some modules can't load into GPL-only kernels)
3. **CRC Check**: Iterates through `__versions` section
   - Looks up each symbol's **current CRC** in kernel's symbol table
   - **Any mismatch**: `return -ENOEXEC` (Invalid module format)
4. **Dependency Check**: Parses `.modinfo` → `depends=`
   - Loads required modules via `modules.dep` (if using `modprobe`)
   - If dependency fails, abort

**Step 7: Memory Allocation**
- Allocates contiguous kernel memory for `.text`, `.data`, `.bss` sections
- Sets appropriate page flags (executable for .text, writable for .data)
- Copies module content into kernel memory

**Step 8: Symbol Relocation**
For each placeholder in `.text`:
- Kernel looks up symbol address in its symbol table
- **Patches** the placeholder with the real address
- Example: `call 0x00000000` → `call 0xffffffff81234560` (real kmalloc address)

**Step 9: Initialization**
- Kernel calls your `hello_init()` function
- `printk()` writes to ring buffer
- `kmalloc()` allocates memory
- If `hello_init()` returns non-zero, module load is **rolled back** (unloaded)

**Step 10: Module is Live**
- Added to `/proc/modules` and `lsmod` list
- Your code runs with kernel privileges
- Can be unloaded with `rmmod` or `modprobe -r`

**Total time**: ~50-500ms from `insmod` to running code

---

## 10. Security: SecureBoot and Signed Modules

### **What is SecureBoot?**

UEFI SecureBoot is a security standard that ensures only **cryptographically signed code** can run during the boot process. Many modern PCs ship with it enabled, and some Linux distributions (Ubuntu, Fedora) enable kernel support by default.

### **Impact on Kernel Modules**

With SecureBoot enabled and kernel lockdown active:
- **Unsigned modules**: **Cannot be loaded** (even as root)
- **Error message**: 
  ```
  insmod: ERROR: could not insert module hello.ko: Operation not permitted
  dmesg: Lockdown: insmod: unsigned module loading is restricted; see man kernel_lockdown.7
  ```

### **Solutions for Learning**

**Option 1: Disable SecureBoot (Recommended for Beginners)**
1. Reboot, enter UEFI/BIOS setup (F2, Del, F12)
2. Find "SecureBoot" setting (usually under Security or Boot)
3. Set to "Disabled"
4. Save and reboot

*In VMs*: Disable in VM settings (VirtualBox: Settings → System → Enable EFI → uncheck "SecureBoot").

**Option 2: Sign Your Module (Advanced)**
1. Generate a local signing key: `openssl req -new -x509 -newkey rsa:2048 ...`
2. Enroll the key in your system's MOK (Machine Owner Key) database: `mokutil --import mykey.cer`
3. Sign the module: `/usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 mykey.priv mykey.x509 hello.ko`
4. Reboot and confirm key enrollment at the UEFI prompt

**For production/enterprise**: Always sign modules. For learning, disabling SecureBoot is acceptable and standard practice.

---

## 11. Debugging: Where's My Output?

### **The Kernel Ring Buffer**

Modules cannot write directly to your terminal or screen. Instead, they log to the **kernel ring buffer**—a circular memory buffer that stores kernel messages.

**Why?**
- Kernel may be running **headless** (no display)
- Terminal may not exist during early boot or panic
- Ring buffer is always available and fast

### **Viewing Output**

```bash
# Method 1: dmesg (traditional)
dmesg                    # View entire buffer
dmesg | tail -20         # Last 20 lines
dmesg -w                 # Follow in real-time (like tail -f)

# Method 2: journalctl (systemd)
journalctl -k            # Kernel messages only
journalctl -k -f         # Follow real-time
journalctl -k | grep hello  # Filter for your module

# Method 3: /proc/kmsg (low-level)
cat /proc/kmsg           # Blocks until messages arrive (requires root)
```

### **Logging Levels in `printk`**

```c
printk(KERN_EMERG   "System is dead\n");    // Priority 0
printk(KERN_ALERT   "Action required\n");   // Priority 1
printk(KERN_CRIT    "Critical error\n");    // Priority 2
printk(KERN_ERR     "Error\n");             // Priority 3
printk(KERN_WARNING "Warning\n");           // Priority 4
printk(KERN_NOTICE  "Notice\n");            // Priority 5
printk(KERN_INFO    "Info: %s\n", msg);     // Priority 6 (common)
printk(KERN_DEBUG   "Debug: %d\n", val);    // Priority 7
```

The default log level is usually `KERN_WARNING` (4), so lower priorities may not appear unless you adjust `/proc/sys/kernel/printk`.

---

## 12. Critical Warnings and Best Practices

### **🚨 Never Use X11 for Development**

**Why:** The graphics driver is a kernel module. If your test module crashes the kernel, the graphics driver dies → frozen GUI → no `dmesg` output → forced hard reset.

**Solutions:**
- Use **VT console**: `Ctrl+Alt+F2` (F2-F6 available)
- Use **SSH** from another machine (safest)
- Use **serial console** for embedded development
- Use **VM console** from host

### **🚨 Module Versioning is Strict**

**Symptom:**
```
insmod: ERROR: could not insert module hello.ko: Invalid module format
dmesg: hello: disagrees about version of symbol kmalloc
```

**Causes:**
```bash
# 1. Wrong headers
uname -r                    # Shows: 6.8.0-48-generic
ls /lib/modules/            # Shows: 6.8.0-49-generic ( mismatch! )

# 2. Kernel upgraded but not rebooted
#    Headers are for new kernel, but running old kernel

# 3. Different kernel flavor
#    Compiled for generic, running lowlatency
```

**Fix:** Ensure exact match:
```bash
sudo apt-get install linux-headers-$(uname -r)
make clean && make
```

### **🚨 No libc in Kernel Space**

You **cannot** use:
- `printf()`, `malloc()`, `free()`, `fopen()`
- Floating-point operations (without explicit kernel FPU save/restore)
- Standard C library functions

**Kernel equivalents:**
- `printk()` (not `printf`)
- `kmalloc()` / `kfree()` (not `malloc`/`free`)
- `vmalloc()` (for large allocations)
- `filp_open()` (for files, but generally discouraged)

### **🚨 Licensing Matters**

```c
MODULE_LICENSE("GPL");  // Allows linking with GPL-only symbols
MODULE_LICENSE("Proprietary");  // May not load in GPL-enforced kernels
```

Without a GPL-compatible license, your module cannot use GPL-only kernel functions, severely limiting functionality.

---

## 13. Troubleshooting Common Issues

### **Problem: Module fails to load**
```
insmod: ERROR: could not insert module hello.ko: Operation not permitted
```

**Checklist:**
1. **SecureBoot**: Disable in UEFI settings
2. **Already loaded**: `lsmod | grep hello` → `rmmod hello` first
3. **Wrong kernel**: `uname -r` vs. headers version
4. **CRC mismatch**: `dmesg` shows version disagreement

---

### **Problem: Unknown symbol error**
```
dmesg: Unknown symbol kmalloc (err -2)
```

**Causes:**
- Compiled without `CONFIG_MODVERSIONS` but running kernel has it enabled (or vice versa)
- Missing dependency module
- Forgetting MODULE_LICENSE() → kernel restricts GPL symbols

---

### **Problem: No output from `printk`**

**Debug:**
```bash
# Check current console log level
cat /proc/sys/kernel/printk
# Output: 4 4 1 7  (current, default, min, boot)

# Temporarily lower to see all messages
echo 8 | sudo tee /proc/sys/kernel/printk

# Or use higher priority
printk(KERN_ALERT "This will show\n");
```

---

### **Problem: Module loads but doesn't work**

**Enable dynamic debugging:**
```bash
# Enable pr_debug() messages for your module
echo 'module hello +p' | sudo tee /sys/kernel/debug/dynamic_debug/control

# Disable
echo 'module hello -p' | sudo tee /sys/kernel/debug/dynamic_debug/control
```

---

### **Problem: Can't unload module**
```
rmmod: ERROR: Module hello is in use
```

**Check:**
```bash
lsmod | grep hello
# Look at "Used by" column

# Find what is using it
sudo lsof /dev/hello  # If it's a device driver
sudo cat /proc/kallsyms | grep hello  # Shows who references it
```

**Force remove (dangerous):**
```bash
sudo rmmod -f hello  # Only if you're certain it's safe
```

---

## 📚 Additional Resources for Device Driver Development

- **Linux Kernel Source**: `apt-get source linux-image-$(uname -r)`
- **API Documentation**: `/lib/modules/$(uname -r)/build/Documentation/`
- **KernelNewbies**: https://kernelnewbies.org/
- **LWN Kernel Articles**: https://lwn.net/Kernel/Index/

---

## 🎯 Getting Started Checklist

Before writing your first module, verify:

- [ ] Installed `build-essential` and `kmod`
- [ ] Installed `linux-headers-$(uname -r)` (exact match)
- [ ] Running in a VM with snapshots enabled
- [ ] Disabled SecureBoot in VM firmware
- [ ] Developed from VT console or SSH, not X11 terminal
- [ ] Know how to check `dmesg` for output
- [ ] Understand that crashes require VM reboot, not host reboot

