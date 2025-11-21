# Getting Started Guide

## 📑 Table of Contents

1. [What Are Kernel Modules?](#what-are-kernel-modules)
2. [Why Modules Matter for Device Drivers](#why-modules-matter-for-device-drivers)
3. [Prerequisites & Toolchain Setup](#prerequisites--toolchain-setup)
4. [Essential Module Management Commands](#essential-module-management-commands)
5. [Kernel Headers: The Only Thing You Need](#kernel-headers-the-only-thing-you-need)
6. [The Golden Rule: Development Environment](#the-golden-rule-development-environment)
7. [Critical Warnings Before You Begin](#critical-warnings-before-you-begin)
8. [Your First "Hello World" Module](#your-first-hello-world-module)
9. [Debugging: Where's My Output?](#debugging-wheres-my-output)
10. [Troubleshooting Common Issues](#troubleshooting-common-issues)
11. [Best Practices & Safety Checklist](#best-practices--safety-checklist)

---

## What Are Kernel Modules?

A **kernel module** is a hot-pluggable code component that extends the Linux kernel's functionality without requiring a reboot. Think of it as a plugin architecture for the operating system's core.

### The Monolithic Kernel Alternative

Without modules, Linux would be a **monolithic kernel** where every driver and feature is statically compiled into one massive kernel image (`/boot/vmlinuz`) file. The consequences:

- **Hardware support**: Adding a new device requires recompiling and reinstalling the entire kernel
- **Memory waste**: Every driver loads at boot, even for hardware you don't own
- **Testing overhead**: A single driver bug requires a full kernel rebuild to test a fix
- **Flexibility loss**: No way to load/unload features on-demand

### Real-World Examples

```bash
# These are all likely kernel modules
lsmod | grep -E '(nvidia|audio|usb|wifi)'
```

- **Device Drivers**: NVIDIA graphics (`nvidia.ko`), USB controllers (`usbhid.ko`), Wi-Fi chipsets (`ath9k.ko`)
- **Filesystems**: ext4, FAT, NTFS support loaded only when mounting those disks
- **System Features**: SELinux, advanced networking protocols, debugging tools

---

## Why Modules Matter for Device Drivers

Device drivers are the primary use case for kernel modules. When you're writing a driver for custom hardware:

1. **Rapid Prototyping**: Load/unload your driver 50 times an hour without rebooting
2. **Safe Testing**: A bug crashes the VM, not your development workstation
3. **Memory Efficiency**: Driver only occupies RAM when hardware is present
4. **Distribution**: Ship a `.ko` file instead of customizing the kernel for every user
5. **Version Independence**: Same driver can work across many kernel versions (with proper versioning)

---

## Prerequisites & Toolchain Setup

### Install Build Essentials

Every modern Linux distribution packages the required tools differently:

| Distribution | Install Command |
|--------------|-----------------|
| **Ubuntu/Debian** | `sudo apt-get install build-essential kmod` |
| **Arch Linux** | `sudo pacman -S gcc kmod` |
| **Fedora/RHEL** | `sudo dnf install gcc kernel-devel kmod` |
| **openSUSE** | `sudo zypper install -t pattern devel_kernel` |

This gives you:
- **`gcc`**: The C compiler configured for kernel builds
- **`make`**: Build automation (via `build-essential` meta-package)
- **`kmod`**: The modern replacement for module-init-tools (`insmod`, `modprobe`, etc.)

### Verify Installation

```bash
# Check compiler version (should be ≥ 9.0 for modern kernels)
gcc --version

# Verify kmod tools are present
which insmod modprobe lsmod
```

---

## Essential Module Management Commands

### `lsmod`: See What's Loaded

Displays currently loaded modules with their memory footprint and dependency tree:

```bash
$ lsmod
Module                  Size  Used by
ath9k                 163840  0
ath                    32768  1 ath9k
mac80211              901120  1 ath9k
cfg80211              770048  2 ath9k,mac80211
```

**Columns Explained**:
- **Module**: The `.ko` filename (without extension)
- **Size**: Memory usage in bytes (converted to human-readable)
- **Used by**: Dependency count and parent modules

### `insmod`: Load a Single Module

Loads one module file from an absolute path. **Does NOT resolve dependencies**.

```bash
# Load a module you compiled
sudo insmod /path/to/hello.ko

# Check if it loaded (will error if missing dependencies)
lsmod | grep hello
```

**Use Case**: Testing a standalone module with no dependencies during development.

### `modprobe`: The Smart Loader

Loads a module **and all dependencies** automatically by name (no path needed).

```bash
# Load with dependencies resolved
sudo modprobe ath9k

# Remove module and unused dependencies
sudo modprobe -r ath9k

# Show dependencies without loading
modinfo -F depends ath9k
```

**How it Works**: `modprobe` reads `/lib/modules/$(uname -r)/modules.dep` (built by `depmod`) to find the complete dependency graph.

### `depmod`: Build the Dependency Database

Scans all modules and creates the dependency map. Usually runs automatically after kernel updates.

```bash
# Rebuild dependency database for current kernel
sudo depmod -a $(uname -r)

# View the generated file (it's plain text)
cat /lib/modules/$(uname -r)/modules.dep | grep ath9k
```

---

## Kernel Headers: The Only Thing You Need

**You do NOT need the full kernel source code.** The full source is hundreds of megabytes and includes architecture-specific code, documentation, and build artifacts. For module development, you only need the **headers**.

### What Are Kernel Headers?

Headers are the "API contract" between your module and the kernel:
- **Function prototypes**: `extern int printk(const char *fmt, ...);`
- **Struct definitions**: `struct file_operations`, `struct device`
- **Macros**: `MODULE_LICENSE()`, `KERN_INFO`, `GFP_KERNEL`
- **Constants**: `#define PI 3.14159`

### Install Just the Headers

| Distribution | Command |
|--------------|---------|
| **Ubuntu/Debian** | `sudo apt-get install linux-headers-$(uname -r)` |
| **Arch Linux** | `sudo pacman -S linux-headers` |
| **Fedora** | `sudo dnf install kernel-devel kernel-headers` |
| **RHEL/CentOS** | `sudo yum install kernel-devel-$(uname -r)` |

### Size Comparison

| Package | Size (Ubuntu 22.04) | contents |
|---------|---------------------|----------|
| `linux-source` (full) | ~180 MB compressed | All kernel code, all architectures |
| `linux-headers` | ~25 MB | Only headers and build infrastructure |

### Verify Headers Are Installed

```bash
# Should return a directory path
ls -d /lib/modules/$(uname -r)/build

# Should show header files
ls /lib/modules/$(uname -r)/build/include/linux/
```

If this directory is missing, compilation will fail with: `"fatal error: linux/module.h: No such file or directory"`

---

## The Golden Rule: Development Environment

### Why You Must Use a Virtual Machine

In user-space, a segfault kills your process. In kernel-space, a bad pointer causes:

- **Complete system freeze**: No keyboard, no mouse, no SSH
- **Filesystem corruption**: If disk I/O is in progress when the kernel panics
- **Data loss**: Unsynced buffers are lost
- **Hardware state corruption**: GPU/Network card left in undefined state

### Recommended VM Setup

1. **VirtualBox** (free, easy snapshots)
2. **QEMU/KVM** (native performance, excellent debugging)
3. **VMware Workstation** (commercial, stable)

### VM Configuration Best Practices

```bash
# Inside the VM, install:
sudo apt-get install build-essential kmod linux-headers-$(uname -r) vim git

# Take a snapshot RIGHT NOW (clean development state)
# Name it: "Pre-Development - Kernel Headers Installed"

# Recommended VM specs:
# - RAM: 2-4 GB (more is better for kernel builds)
# - Disk: 20 GB dynamic allocation
# - CPUs: 2+ cores (speeds up compilation)
```

### Working with Snapshots

**Before each test session:**
```bash
# Save your code outside the VM (git push, shared folder)
git commit -am "Working on driver probe function"

# Revert to clean snapshot if things go wrong
# VirtualBox: Right-click VM → Snapshots → Restore
```

---

## Critical Warnings Before You Begin

### 1. SecureBoot: The Unsigned Module Blocker

**The Problem**: UEFI SecureBoot requires all kernel code to be cryptographically signed with a trusted key. Your manually compiled module is unsigned.

**Symptoms**:
```bash
$ sudo insmod hello.ko
insmod: ERROR: could not insert module hello.ko: Operation not permitted

# Check dmesg
$ dmesg | tail
Lockdown: insmod: unsigned module loading is restricted
```

**Solutions**:

| Approach | Security | Complexity | Recommendation |
|----------|----------|------------|----------------|
| **Disable SecureBoot** | Low | Very Easy | ✅ **Use for learning** |
| **Sign modules manually** | High | Complex | Use for production drivers |
| **Enroll custom keys** | Medium | Moderate | Advanced developers only |

**How to Disable SecureBoot (VM only):**
1. Reboot VM
2. Press `F2`/`Del`/`Esc` to enter UEFI settings
3. Navigate to "Boot" or "Security" tab
4. Find "SecureBoot" → Set to "Disabled"
5. Save and exit

**Verification:**
```bash
# Should show "SecureBoot disabled"
mokutil --sb-state
```

### 2. Module Versioning (CONFIG_MODVERSIONS)

Most distribution kernels enable this security feature. It means:

- A module compiled for **kernel 6.8.0-48** will NOT load on **6.8.0-49**
- Even minor patch-level differences create incompatible CRC checksums
- **This is intentional** to prevent subtle API mismatches that corrupt memory

**How to Check If Versioning Is Enabled:**
```bash
# If this returns "y", CONFIG_MODVERSIONS is on
grep CONFIG_MODVERSIONS /boot/config-$(uname -r)
```

**Solutions for Versioning Errors:**
```bash
# Error message example:
insmod: ERROR: could not insert module hello.ko: Invalid module format
dmesg: hello: disagrees about version of symbol kmalloc

# Fix 1: Install exact matching headers (recommended)
sudo apt-get install linux-headers-$(uname -r)

# Fix 2: Reboot into the kernel you compiled against
uname -r  # Verify running version matches headers

# Fix 3: Build a custom kernel without CONFIG_MODVERSIONS (advanced, not recommended for beginners)
```

### 3. Never Use X11 for Testing

**The Danger**: Your X11/Wayland desktop (GNOME, KDE, etc.) depends on kernel modules:
- `nvidia.ko` or `amdgpu.ko` for graphics
- `usbhid.ko` for keyboard/mouse
- `snd_hda_intel.ko` for audio

If you crash the kernel, these modules die, freezing your GUI. You won't see panic messages, and `Ctrl+Alt+F2` may not work.

**Safe Development Environments:**

```bash
# Option 1: Pure text console
# Switch to TTY2 (no GUI running) // F3 to F6
Ctrl+Alt+F3
# to getback use Ctrl+Alt+F1 or Ctrl+Alt+F2
Login: your_user
Password: ******
# Now compile and test safely

# Option 2: SSH from host
# On VM: sudo apt-get install openssh-server
# On host: ssh user@vm-ip-address
# If SSH dies during insmod, you know the kernel crashed

# Option 3: Serial console (advanced debugging)
# Configure VM with serial port, use minicom on host
```

**Rule of Thumb**: If you're testing code that could crash, use SSH. If you're testing graphics drivers, use a text console.

### 4. Kernel Logging vs. printf()

**The Misconception**: `printf()` doesn't exist in kernel space. Modules use `printk()`, but it doesn't print to your screen—it writes to the **kernel ring buffer**.

    The kernel ring buffer is a circular memory buffer inside the Linux kernel where all kernel messages are stored.

**How to See Your Output:**

```bash
# Method 1: Real-time monitoring
dmesg -w  # -w = follow (like tail -f)

# Method 2: View last 20 lines
dmesg | tail -20

# Method 3: Use journalctl (systemd systems)
journalctl -kf  # -k = kernel, -f = follow

# Method 4: From within your module code
printk(KERN_INFO "Hello from kernel space! Process: %s\n", current->comm);
# KERN_INFO is the log level (see below)
# current->comm = current process name
```

**Log Levels (Most to Least Severe):**
```c
printk(KERN_EMERG   "System is unusable\n");
printk(KERN_ALERT   "Action must be taken immediately\n");
printk(KERN_CRIT    "Critical conditions\n");
printk(KERN_ERR     "Error conditions\n");
printk(KERN_WARNING "Warning conditions\n");
printk(KERN_NOTICE  "Normal but significant condition\n");
printk(KERN_INFO    "Informational (most common for debugging)\n");
printk(KERN_DEBUG   "Debug-level messages\n");
```

**Important**: The kernel rate-limits messages. Flooding `dmesg` can cause messages to be dropped. Use `printk_ratelimit()` for repeated logs.

---

## Your First "Hello World" Module

### Directory Structure

```
~/kernel-dev/
├── hello/
│   ├── hello.c
│   └── Makefile
└── README.md
```

### `hello.c`

```c
#include <linux/module.h>    // Needed by all modules
#include <linux/kernel.h>    // Needed for KERN_INFO
#include <linux/init.h>      // Needed for __init and __exit macros

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A simple Hello World kernel module");

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello, kernel world! (pid: %d)\n", current->pid);
    return 0;  // Non-zero means module load failed
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "Goodbye, kernel world!\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

**Key Points**:
- `__init`: Marks function as initialization-only (freed after loading)
- `__exit`: Marks function as cleanup-only (omitted if module built-in)
- `module_init/module_exit`: Register your entry/exit points
- `MODULE_LICENSE`: Required to access GPL-only kernel APIs

### `Makefile`

```makefile
obj-m += hello.o

# Get the current kernel version's build directory
KDIR = /lib/modules/$(shell uname -r)/build

# Use the kernel's build system
PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

help:
	@echo "Targets: all (build), clean, load, unload, test"
	@echo "Module will be built for kernel: $(shell uname -r)"

load:
	sudo insmod hello.ko

unload:
	sudo rmmod hello

test: load
	@echo "Module loaded. Check dmesg:"
	@dmesg | tail -3
	@echo "Unloading..."
	@sudo rmmod hello
	@dmesg | tail -2
```

### Build and Test

```bash
# In VM text console or SSH
cd ~/kernel-dev/hello

# Build
make

# Should create: hello.ko, hello.o, modules.order, Module.symvers

# Load the module
sudo insmod hello.ko

# Check it's loaded
lsmod | grep hello

# See output
dmesg | tail
# Output: Hello, kernel world! (pid: 1234)

# Unload
sudo rmmod hello

# See goodbye message
dmesg | tail
```

---

## Troubleshooting Common Issues

### Problem: "Can't find linux/module.h"

**Cause**: Kernel headers not installed or build directory missing
```bash
# Fix
sudo apt-get install linux-headers-$(uname -r)
# Verify
ls -d /lib/modules/$(uname -r)/build
```

### Problem: "Invalid module format" or "disagrees about version"

**Cause**: Version mismatch between compile-time and run-time kernel
```bash
# Check versions
uname -r                            # Running kernel
ls /lib/modules/                    # Installed headers

# If different, you either:
# 1. Need to reboot into the matching kernel
# 2. Install headers for the running kernel
```

### Problem: "Operation not permitted"

**Cause**: SecureBoot is enabled, or you're not root
```bash
# Check if root
whoami  # Should be root or using sudo

# Check SecureBoot
mokutil --sb-state  # If enabled, disable in VM UEFI
```

### Problem: No output in dmesg

**Cause**: Looking at wrong terminal or not running dmesg
```bash
# Make sure module loaded
lsmod | grep hello

# Check dmesg without filtering
dmesg | grep Hello

# Real-time monitor
dmesg -w &
sudo insmod hello.ko
```

### Problem: Module is "in use" can't unload

**Cause**: Another module depends on yours, or your code is stuck
```bash
# Check dependencies
lsmod | grep hello

# Force remove (dangerous, can crash if module is stuck)
sudo rmmod -f hello  # Use with extreme caution
```

---

## Best Practices & Safety Checklist

### Before You Write Code
- [ ] VM snapshot taken and named clearly
- [ ] Kernel headers installed and verified: `ls -d /lib/modules/$(uname -r)/build`
- [ ] SecureBoot disabled in VM (for learning)
- [ ] Working in text console (Ctrl+Alt+F2) or SSH
- [ ] Code in version control (git) outside the VM

### During Development
- [ ] Compile with `-Werror` to catch warnings: Add `ccflags-y := -Wall -Werror` to Makefile
- [ ] Use `printk(KERN_INFO ...)` for normal debug, `printk(KERN_ERR ...)` for errors
- [ ] Always check return values from kernel APIs
- [ ] Test load/unload cycle 10 times before adding complexity
- [ ] Monitor dmesg continuously: `dmesg -w`

### Before Loading Untested Code
- [ ] Code review: read every line before compiling
- [ ] Static analysis: `sparse` tool can catch kernel API misuse
  ```bash
  sudo apt-get install sparse
  make C=2  # Check every .c file with sparse
  ```
- [ ] Have VM snapshot ready for instant restore
- [ ] No important processes running in VM

### When Things Go Wrong
- [ ] If kernel hangs: Force VM power-off, restore snapshot
- [ ] If module won't load: Check `dmesg`, verify versions
- [ ] If filesystem corrupts: Restore VM from backup snapshot
- [ ] Document the crash scenario, fix code, try again

---

## Next Steps

Now that you understand the basics, explore:

- **README2: Linux-Kernel-Modules-Internals.md** - Deep dive into dependencies and versioning
- **Linux Device Drivers, 3rd Edition** (LDD3) - The classic book (free online)
- **kernelnewbies.org** - Community for new kernel developers
- **Documentation/ directory** in kernel headers: `/lib/modules/$(uname -r)/build/Documentation/`

Remember: Every kernel developer crashes their first VM. The goal is to learn from it—safely.

