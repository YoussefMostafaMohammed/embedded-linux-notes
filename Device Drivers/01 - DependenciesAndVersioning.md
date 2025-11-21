# Dependencies & Versioning

## 📑 Table of Contents

1. [Module Dependencies: The Foundation](#module-dependencies-the-foundation)
2. [The depmod System: Building the Dependency Graph](#the-depmod-system-building-the-dependency-graph)
3. [Module Versioning (CONFIG_MODVERSIONS) Explained](#module-versioning-configmodversions-explained)
4. [CRC32 Symbol Checksum Mechanism](#crc32-symbol-checksum-mechanism)
5. [ELF Sections in .ko Files](#elf-sections-in-ko-files)
6. [Compilation Phase: From Source to .ko](#compilation-phase-from-source-to-ko)
7. [Insertion Phase: Kernel Loading Process](#insertion-phase-kernel-loading-process)
8. [Common Version Mismatch Scenarios](#common-version-mismatch-scenarios)
9. [Advanced Topics & Edge Cases](#advanced-topics--edge-cases)
10. [Reference Tables & File Locations](#reference-tables--file-locations)

---

## Module Dependencies: The Foundation

### What Are Module Dependencies?

A module rarely functions in isolation. Device drivers especially are built in layers:

```
USB Wi-Fi Driver (ath9k.ko)
    ↓ depends on ↓
Wireless Core (mac80211.ko)
    ↓ depends on ↓
Configuration API (cfg80211.ko)
    ↓ depends on ↓
USB Subsystem (usbcore.ko) ← part of kernel
```

If you load `ath9k.ko` without `mac80211.ko` first, the kernel will reject it with:
```
insmod: ERROR: could not insert module ath9k.ko: Unknown symbol in module
dmesg: ath9k: Unknown symbol mac80211_register_hw
```

### Why Not Static Linking?

**Static linking** (bundling all dependencies into one giant `.ko`) seems simpler but has fatal flaws:

1. **Memory Bloat**: 50 Wi-Fi drivers × 5MB each = 250MB wasted if each embeds its own copy of `mac80211.ko`
2. **Update Nightmare**: Fix a bug in `mac80211` → rebuild 50 drivers
3. **License Conflicts**: Can't mix GPL-only code with proprietary drivers if statically linked
4. **Symbol Conflicts**: Two modules defining the same static function name

Dynamic dependencies solve these by sharing code at runtime.

### Dependency Types

**Hard Dependencies**: Module won't load without these symbols
```bash
# Check hard dependencies
modinfo -F depends ath9h
# Output: mac80211,cfg80211,usbcore
```

**Soft Dependencies**: Optional features loaded if available
```c
// In code: request_module("feature") can load on-demand
```

---

## The depmod System: Building the Dependency Graph

### What Is depmod?

`depmod` is a user-space utility that scans every `.ko` file in `/lib/modules/$(uname -r)/` and builds a dependency database. It runs automatically after:
- Kernel package installation
- Manual module installation (`make modules_install`)

### The modules.dep Database

Location: `/lib/modules/$(uname -r)/modules.dep`

**Format**: Plain text, one line per module:
```
kernel/drivers/net/wireless/ath/ath9k.ko: kernel/drivers/net/wireless/ath/ath.ko kernel/net/mac80211.ko kernel/net/wireless/cfg80211.ko
```

**Meaning**: `ath9k.ko` requires `ath.ko`, `mac80211.ko`, and `cfg80211.ko` to be loaded first.

### How modprobe Uses the Database

When you run `sudo modprobe ath9k`:
1. Reads `modules.dep` to find ath9k's dependencies
2. Checks if each dependency is loaded (`/proc/modules`)
3. Loads missing dependencies in correct order (depth-first)
4. Finally loads ath9k.ko itself

**Without modprobe**, you'd have to manually:
```bash
sudo insmod /lib/modules/.../cfg80211.ko
sudo insmod /lib/modules/.../mac80211.ko
sudo insmod /lib/modules/.../ath.ko
sudo insmod /lib/modules/.../ath9k.ko
```

### Regenerating the Database

```bash
# Force rebuild for current kernel
sudo depmod -a $(uname -r)

# Verbose output (shows what's being scanned)
sudo depmod -av

# Check a specific module's deps
depmod -n | grep ath9k
```

---

## Module Versioning (CONFIG_MODVERSIONS) Explained

### The Problem: Silent API Breakage

Kernel APIs change frequently. Consider this subtle breakage:

```c
// Kernel 6.8.0: net/device.h
struct net_device {
    char name[16];
    int mtu;
};

// Kernel 6.8.1: net/device.h
struct net_device {
    char name[32];  // Array size changed!
    int mtu;
    int new_field;  // Added field
};
```

A module compiled against 6.8.0 would corrupt memory when run on 6.8.1 because:
- It allocates based on old struct size (20 bytes)
- Kernel writes based on new struct size (40 bytes)
- Stack/heap corruption → unpredictable crashes

### The Solution: CRC32 Symbol Versioning

**CONFIG_MODVERSIONS** embeds a **fingerprint** (CRC32 checksum) for every public kernel symbol. The mechanism:

1. **At kernel compile time**: For each exported symbol (function, struct, variable), compute CRC from its prototype:
   ```c
   // Symbol: kmalloc
   // Prototype: void *kmalloc(size_t size, gfp_t flags);
   // CRC computed from this exact string: 0x4a7b3c9d
   ```

2. **At module compile time**: For every kernel symbol your module uses, embed the **expected CRC** into the `.ko` file

3. **At module load time**: Verify each symbol's CRC matches the running kernel's CRC

### Checking If Versioning Is Enabled

```bash
# Method 1: Check kernel config
grep CONFIG_MODVERSIONS /boot/config-$(uname -r)
# Expected: CONFIG_MODVERSIONS=y

# Method 2: Check vermagic string
modinfo hello.ko | grep vermagic
# Output: vermagic: 6.8.0-48-generic SMP preempt mod_unload modversions
#                                                     ↑ indicates versioning on
```

---

## CRC32 Symbol Checksum Mechanism

### Where CRCs Come From

The kernel's build process generates a **`Module.symvers`** file during compilation:

```bash
# Location in installed headers
cat /lib/modules/$(uname -r)/build/Module.symvers
```

**Format**: `CRC Symbol Name Source Export_Type`
```
0x4a7b3c9d  kmalloc   vmlinux EXPORT_SYMBOL
0x8f2e1a4b  printk    vmlinux EXPORT_SYMBOL
0x2c9f5b8a  module_init vmlinux EXPORT_SYMBOL
0x1a3c5e7f  my_function mac80211 EXPORT_SYMBOL_GPL
```

**Fields Explained**:
- **CRC**: 32-bit checksum computed from symbol's interface
- **Symbol Name**: Function, variable, or struct name
- **Source**: `vmlinux` (core kernel) or module name
- **Export Type**: Who can use this symbol

### Export Type Control

**EXPORT_SYMBOL()**: Available to any module (GPL or proprietary)
```c
// In kernel source
void *kmalloc(size_t size, gfp_t flags) { ... }
EXPORT_SYMBOL(kmalloc);
```

**EXPORT_SYMBOL_GPL()**: GPL-only modules only
```c
// Proprietary modules cannot use this
void *internal_kmalloc(size_t size, gfp_t flags) { ... }
EXPORT_SYMBOL_GPL(internal_kmalloc);
```

**EXPORT_SYMBOL_GPL_FUTURE()**: Will become GPL-only in future kernel versions

### The Granular Nature of CRCs

**Critical Point**: A module doesn't have *one* kernel version CRC—it has **dozens**, one per symbol used:

```
Symbol Name       | Expected CRC
------------------|-------------
kmalloc           | 0x4a7b3c9d
printk            | 0x8f2e1a4b
module_init       | 0x2c9f5b8a
__this_module     | 0x3f6c2d1e
register_netdev   | 0x7a9c3e5f
...
```

This precision allows a module to load if it uses only unchanged symbols, even if other unrelated kernel APIs changed.

---

## ELF Sections in .ko Files

### What Is ELF?

**ELF** (Executable and Linkable Format) is the file format for `.ko` modules, executables, and shared libraries. It's a structured binary format with named sections.

### Inspecting Sections

```bash
# Install binutils if missing
sudo apt-get install binutils

# List all sections in a module
readelf -S hello.ko

# Typical output (abbreviated):
Section Headers:
  [Nr] Name              Type            Addr     Off    Size
  [ 0]                   NULL            00000000 000000 000000
  [ 1] .text             PROGBITS        00000000 001000 0000a4
  [ 2] .init.text        PROGBITS        00000000 0010a4 000030
  [ 3] .exit.text        PROGBITS        00000000 0010d4 000020
  [ 4] .data             PROGBITS        00000000 001100 000010
  [ 5] .bss              NOBITS          00000000 001110 000008
  [ 6] .modinfo          PROGBITS        00000000 001110 0000f0
  [ 7] __versions        PROGBITS        00000000 001200 0000c0
  [ 8] .symtab           SYMTAB          00000000 002000 000180
  [ 9] .strtab           STRTAB          00000000 002180 0000a0
```

### Key Sections Explained

| Section Name | Description | Contents |
|--------------|-------------|----------|
| **`.text`** | Main code | Your `hello_init()`, `hello_exit()` machine code |
| **`.init.text`** | Initialization code | `__init` marked functions (freed after load) |
| **`.exit.text`** | Cleanup code | `__exit` marked functions (omitted in built-in modules) |
| **`.data`** | Initialized globals | `static int my_var = 42;` |
| **`.bss`** | Uninitialized globals | `static int my_var;` (zero-initialized) |
| **`.modinfo`** | Module metadata | License, author, description, dependencies |
| **`__versions`** | Versioning data | Symbol→CRC mapping (only with CONFIG_MODVERSIONS) |
| **`.symtab`** | Symbol table | Debug symbols for stack traces |
| **`.strtab`** | String table | Names of symbols and sections |

### Viewing Section Contents

```bash
# See .modinfo metadata
modinfo hello.ko

# See raw __versions data (requires some parsing)
readelf -x __versions hello.ko

# Hex dump of .modinfo
objdump -s -j .modinfo hello.ko
```

---

## Compilation Phase: From Source to .ko

### Step-by-Step Breakdown

Let's trace what happens when you type `make`:

#### **Step 0: Source Code**
```c
// hello.c
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello, kernel!\n");
    kmalloc(1024, GFP_KERNEL);  // Call kernel function
    return 0;
}
```

#### **Step 1: Makefile Invocation**
```makefile
# Makefile sets up kernel build system
KDIR = /lib/modules/$(shell uname -r)/build
make -C $(KDIR) M=$(PWD) modules
```
- `-C $(KDIR)`: Change to kernel build directory (contains master Makefile)
- `M=$(PWD)`: Tell kernel build system where your module source is
- `modules`: Build target for loadable modules

#### **Step 2: Preprocessing**
```bash
# Generated command (simplified)
gcc -E -I/lib/modules/$(uname -r)/build/include \
    -DMODULE -D__KERNEL__ \
    hello.c -o hello.i
```
- `-I.../include`: Find kernel headers
- `-DMODULE`: Compile as loadable module (not built-in)
- `-D__KERNEL__`: Kernel-space compilation (not user-space)

**Result**: `hello.i` with all `#include` statements expanded (10,000+ lines).

#### **Step 3: Compilation to Assembly**
```bash
gcc -S hello.i -o hello.s
```
Compiles preprocessed C to assembly code. You can view it:
```bash
cat hello.s | grep kmalloc
# Shows: call kmalloc
```

#### **Step 4: Assembly to Object Code**
```bash
gcc -c hello.s -o hello.o
```
Creates `hello.o` (ELF relocatable object file). At this stage:
- Machine code is generated
- **Unresolved symbols** (`printk`, `kmalloc`) are marked as "external references"
- Symbol table is created (read with `nm hello.o`)

```bash
# View unresolved symbols
nm hello.o | grep "U "
# Output: U kmalloc
#         U printk
```

#### **Step 5: Linking to Create .ko**
```bash
# Linking command (simplified)
ld -r hello.o -o hello.ko \
   -T /lib/modules/$(uname -r)/build/scripts/module-common.lds \
   --build-id
```
- `-r`: Relocatable link (not final executable)
- `-T ...lds`: Use kernel's linker script for sections
- **Magic happens here**: Build system reads `Module.symvers`, extracts CRCs for `printk` and `kmalloc`, and embeds them into `__versions` section

#### **Step 6: Generate Module Metadata**
```bash
# Create .mod.c file automatically
echo 'MODULE_INFO(vermagic, "6.8.0 SMP mod_unload modversions");' > hello.mod.c
gcc -c hello.mod.c -o hello.mod.o
ld -r hello.ko hello.mod.o -o hello.ko
```
Adds `.modinfo` section with version magic string.

### Final Build Artifacts

```
hello/
├── hello.c          # Your source code
├── hello.o          # Temporary object file
├── hello.mod.c      # Auto-generated module info source
├── hello.mod.o      # Compiled module info
├── hello.ko         # ✅ Final loadable module
├── Module.symvers   # Symbol versions (if you export symbols)
├── modules.order    # Build order for multiple modules
└── .hello.ko.cmd    # Dependency tracking for make
```

---

## Insertion Phase: Kernel Loading Process

### Step-by-Step: What Happens When You Run insmod

#### **Step 6: User-Space (insmod Program)**
```bash
# You execute
sudo insmod hello.ko
```
1. `insmod` opens `hello.ko` file, reads entire file into memory buffer
2. Makes `init_module()` system call: `syscall(__NR_init_module, buf, len, args)`
3. Transfers control to kernel

#### **Step 7: Kernel Verification (load_module())**
The kernel's `kernel/module.c:load_module()` function:

**Phase A: ELF Header Validation**
- Verify file is valid ELF format
- Check architecture matches (x86_64 vs ARM)
- Ensure required sections exist (.text, .modinfo, etc.)

**Phase B: CRC Version Check (if CONFIG_MODVERSIONS)**
```c
// Pseudo-code of kernel's check
for each symbol in module's __versions section:
    crc_from_module = symbol.crc
    crc_from_kernel = lookup_symbol_in_kernel_table(symbol.name).crc
    
    if crc_from_module != crc_from_kernel:
        return -ENOEXEC  // Invalid module format
```
- **Mismatch**: `insmod` returns error, `dmesg` shows "disagrees about version of symbol kmalloc"
- **Match**: Proceed to Step 8

**Phase C: License Check**
- Verify `MODULE_LICENSE` is present
- GPL modules can use GPL-only symbols; proprietary modules cannot

**Phase D: Dependency Resolution**
- Read `.modinfo` section's "depends" field
- For each dependency:
  - Check if already loaded (`/proc/modules`)
  - If not loaded, search filesystem for dependency `.ko`
  - Recursively load dependencies
- If dependency missing: fail with "Unknown symbol" error

#### **Step 8: Memory Allocation & Loading**
```c
// Kernel allocates memory for module
vmalloc(mod->core_size);  // For code and data
vmalloc(mod->init_size);  // For __init functions (freed after init)
```

**Address Space Placement**:
- Modules load into kernel's virtual address space (not user-space)
- Typically in high memory region (above physical RAM mappings)
- Each module gets its own `struct module` object in kernel memory

#### **Step 9: Symbol Resolution & Relocation**
The kernel **patches** your module's code with real addresses:

```c
// In your module's .text section (before loading):
0xffffffffc0001000:  call 0x0  // placeholder for kmalloc

// After relocation:
0xffffffffc0001000:  call 0xffffffff81234560  // real kmalloc address
```

**Relocation Process**:
1. Parse relocation entries in `.rela.text` section
2. For each entry: "symbol Y used at offset X"
3. Look up symbol Y's address in kernel symbol table
4. Write that address into module's code at offset X

**Viewing Kernel Symbol Table**:
```bash
# All symbols kernel knows about
cat /proc/kallsyms

# Filter for kmalloc
grep kmalloc /proc/kallsyms
# Output: ffffffff81234560 T kmalloc
#         ^address       ^type ^name
```

#### **Step 10: Execute Module Init Function**
```c
// Kernel calls your function
ret = module->init();  // Your hello_init()

// If init returns non-zero, unload everything
if (ret != 0) {
    free_module(module);
    return ret;
}
```

**What Your Init Function Does**:
- Register device drivers (`register_chrdev()`, `platform_driver_register()`)
- Allocate resources (IRQs, I/O ports, memory)
- Initialize hardware
- Return 0 on success, negative error code on failure

#### **Step 11: Module Is Live**
- Add `struct module` to global linked list (`/proc/modules` reflects this)
- Mark module state as "LIVE"
- Unlock module mutex
- Return success to user-space

```bash
# Verify loaded
lsmod | grep hello
# Output: hello                  16384  0

# View module details in /sys
ls /sys/module/hello/
# sections/  parameters/  initstate  holders/  notes/
```

---

## Common Version Mismatch Scenarios

### Scenario 1: Forgot to Reboot After Kernel Update

```bash
# Yesterday: Installed headers for 6.8.0-48
sudo apt-get install linux-headers-$(uname -r)  # Got 6.8.0-48

# Today: Ran apt upgrade, kernel upgraded to 6.8.0-49
# BUT: Haven't rebooted, still running 6.8.0-48

# You compile against 6.8.0-48 headers
make

# Reboot...
sudo reboot

# Now running 6.8.0-49
sudo insmod hello.ko
# ERROR: disagrees about version of symbol printk

# Fix: Install headers for new kernel
sudo apt-get install linux-headers-$(uname -r)  # Now gets 6.8.0-49
make clean && make
sudo insmod hello.ko  # Works!
```

**Prevention**: Always verify after reboot:
```bash
uname -r                              # Running kernel
ls /lib/modules/                      # Available headers
# Should match!
```

### Scenario 2: Generic vs. Lowlatency Kernels

```bash
# Installed generic headers
sudo apt-get install linux-headers-generic

# But running lowlatency kernel
uname -r
# Output: 6.8.0-49-lowlatency

# Headers are for -generic, not -lowlatency
make
sudo insmod hello.ko
# ERROR: version magic '6.8.0-49-generic SMP' should be '6.8.0-49-lowlatency SMP'

# Fix: Install exact flavor
sudo apt-get install linux-headers-$(uname -r)  # -lowlatency specific
```

### Scenario 3: Cross-Machine Module Copy

```bash
# On Machine A (kernel 6.8.0-48)
make
scp hello.ko user@machineb:/tmp/

# On Machine B (kernel 6.8.0-49)
sudo insmod /tmp/hello.ko
# ERROR: version magic mismatch

# Solution: Rebuild on target machine
# NEVER copy .ko files between machines with different kernels
```

### Scenario 4: Custom Kernel Build

```bash
# Built custom kernel 6.9.0-rc1
# Installed headers for it

# But module build uses wrong vermagic
make
# vermagic shows: 6.8.0-48-generic (wrong!)

# Fix: Ensure KDIR points to custom kernel build dir
make KDIR=/path/to/my/kernel/build
```

---

## Advanced Topics & Edge Cases

### What Happens When You Disable CONFIG_MODVERSIONS?

If you build a custom kernel with `CONFIG_MODVERSIONS=n`:
- `Module.symvers` is empty
- `__versions` section is not created in modules
- Load-time CRC check is skipped
- **Risk**: Silent memory corruption if APIs changed

**When to Disable**: Only for kernel development debugging, never for production or learning.

### Forcing Module Load Despite Version Mismatch

**DANGEROUS**: Can corrupt your kernel instantly. Only use in VMs for testing.

```bash
# Use insmod --force (not recommended)
sudo insmod --force hello.ko

# Kernel will warn but load
dmesg: hello: module verification failed: signature and/or required key missing - tainting kernel
dmesg: hello: version magic '6.8.0-48-generic SMP' should be '6.8.0-49-lowlatency SMP'
dmesg: hello: loading out-of-tree module taints kernel
```

**What "tainting" means**:
- Kernel marks itself as "tainted" (untrusted code loaded)
- Bug reports from tainted kernels are rejected by kernel developers
- Indicates system is in unknown state

### Version Magic String Detail

The `vermagic` string contains build configuration:
```
6.8.0-48-generic SMP preempt mod_unload modversions
└─ version ──┘ └─ flavor ┘ └─ SMP? ┘ └─ preempt? ┘ └─ features ─┘
```

**Fields**:
- **Kernel version**: `6.8.0-48-generic`
- **SMP**: Symmetric Multi-Processing (multi-core support)
- **preempt**: Preemptible kernel (low-latency)
- **mod_unload**: Module unloading supported
- **modversions**: CRC versioning enabled

Any mismatch → module rejected (unless forced).

### Stacked Module Dependencies

Complex dependency chains:
```
sound_driver.ko
    ↓ depends on
sound_core.ko
    ↓ depends on
acpi.ko
    ↓ depends on
bus_core.ko ← part of kernel
```

**Circular dependencies** are not allowed and will be detected by `depmod`.

---

## Reference Tables & File Locations

### Kernel Build & Module Files

| File Path | Purpose | Who Creates/Updates |
|-----------|---------|---------------------|
| `/lib/modules/$(uname -r)/build` | Symlink to kernel source/headers | Package manager |
| `/lib/modules/$(uname -r)/build/Module.symvers` | Symbol CRC database | Kernel package |
| `/boot/vmlinuz-$(uname -r)` | Compressed kernel image | Package manager |
| `/boot/config-$(uname -r)` | Kernel configuration | Package manager |
| `/lib/modules/$(uname -r)/modules.dep` | Dependency database | `depmod` |
| `/lib/modules/$(uname -r)/modules.order` | Module load order | `depmod` |
| `/proc/modules` | Currently loaded modules (runtime) | Kernel (in-memory) |
| `/proc/kallsyms` | Kernel symbol table with addresses | Kernel (in-memory) |
| `/sys/module/<name>/` | Module parameters and info | Kernel (when module loaded) |
| `hello.ko` | Your compiled module | You (via make) |

### Common Error Messages & Meanings

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `Unknown symbol in module` | Missing dependency or typo in symbol name | Run `modprobe` instead of `insmod`, or load dependencies first |
| `Invalid module format` | CRC/version mismatch or wrong architecture | Verify headers match `uname -r`, rebuild module |
| `Module has wrong vermagic` | Kernel configuration mismatch | Use exact matching kernel/headers |
| `Operation not permitted` | SecureBoot enabled or not root | Disable SecureBoot in VM, use `sudo` |
| `Device or resource busy` | Module in use (has dependencies) | `modprobe -r` to remove dependencies, or `rmmod -f` (dangerous) |
| `disagrees about version` | CONFIG_MODVERSIONS CRC mismatch | Install exact kernel headers, recompile |
| `module verification failed` | Unsigned module with SecureBoot | Disable SecureBoot or sign module |

### Symbol Types in /proc/kallsyms

```
Address          Type  Symbol Name
ffffffff81234560 T kmalloc
                ^ type
```

| Type | Meaning |
|------|---------|
| **T** | Global function (text/code) |
| **t** | Local function (static) |
| **D** | Global data variable |
| **d** | Local data variable (static) |
| **B** | Global BSS (uninitialized) |
| **R** | Read-only data |
| **A** | Absolute symbol (address fixed) |
| **U** | Undefined (external reference) |

---

## Additional Resources

- **Kernel Source**: `Documentation/admin-guide/modules.rst`
- **Tool Documentation**: `man depmod`, `man modprobe`, `man insmod`
- **Module API**: `/lib/modules/$(uname -r)/build/include/linux/module.h`
- **Linux Weekly News (LWN.net)**: Excellent kernel development articles

---

## Summary

Kernel modules are not just "plugins"—they are sophisticated components with:
- **Dependency management** via `depmod` and `modules.dep`
- **Binary compatibility enforcement** via CRC32 versioning
- **Complex loading process** involving relocation, symbol resolution, and memory management
- **Strict safety requirements** due to kernel-space privileges

