# Linux Kernel Device Tree, Optimizations & Pitfalls - Complete Technical Guide

## 📑 Table of Contents

1.  [Chapter Overview](#chapter-overview)
2.  [Device Tree Fundamentals](#device-tree-fundamentals)
    -   [What is Device Tree?](#what-is-device-tree)
    -   [Why Device Tree Matters for Kernel Modules](#why-device-tree-matters)
    -   [.dts vs .dtb Files](#dts-vs-dtb)
3.  [Device Tree and Kernel Modules](#device-tree-and-modules)
    -   [Platform Device Framework Integration](#platform-device-framework)
    -   [Compatible Strings](#compatible-strings)
    -   [Property Reading Mechanisms](#property-reading)
4.  [Deep Dive: Device Tree Module Example](#deep-dive-device-tree)
    -   [Module Structure and Data](#module-structure)
    -   [Probe Function Walkthrough](#probe-walkthrough)
    -   [Property Extraction in Detail](#property-extraction)
    -   [Remove Function and Kernel 6.11+ Changes](#remove-function)
5.  [Device Tree Source Syntax](#device-tree-source)
    -   [Node Structure](#node-structure)
    -   [Property Types](#property-types)
    -   [Phandle References](#phandles)
6.  [Testing Device Tree Modules](#testing-device-tree)
    -   [Platform Device Detection](#platform-detection)
    -   [Device Tree Overlays](#device-tree-overlays)
    -   [QEMU Virtualization](#qemu-testing)
    -   [Debugging with sysfs](#debugging-sysfs)
7.  [Common Device Tree Functions Reference](#dt-functions-reference)
    -   [String Properties](#string-properties)
    -   [Numeric Properties](#numeric-properties)
    -   [Boolean Properties](#boolean-properties)
    -   [Advanced Parsing Functions](#advanced-parsing)
8.  [Kernel Optimizations](#kernel-optimizations)
    -   [likely() and unlikely() Macros](#likely-unlikely)
    -   [Static Keys and Jump Labels](#static-keys)
    -   [Performance Impact Analysis](#performance-impact)
9.  [Common Pitfalls](#common-pitfalls)
    -   [Standard Libraries](#standard-libraries)
    -   [Interrupt Handling](#interrupt-handling)
    -   [Memory Allocation Context](#memory-context)
    -   [Configuration Dependencies](#config-dependencies)
10. [Best Practices](#best-practices)
11. [Further Resources](#further-resources)

---

## Chapter Overview

This chapter covers three critical topics for advanced kernel module development:

1. **Device Tree Integration**: How kernel modules can discover and configure hardware dynamically from Device Tree descriptions instead of hard-coded values. This is essential for ARM-based embedded systems where hardware configurations vary significantly between board revisions.

2. **Kernel Optimizations**: Performance techniques using `likely()`/`unlikely()` branch prediction and static keys (jump labels) to minimize overhead in hot paths like interrupts and context switches.

3. **Common Pitfalls**: Dangerous mistakes that crash systems, including using user-space libraries in kernel code and improper interrupt management.

**Key Learning Objectives**:
- Parse Device Tree properties from kernel modules
- Use the platform driver framework for DT binding
- Apply compiler hints for branch optimization
- Implement runtime code patching with static keys
- Avoid catastrophic errors in kernel development

---

## Device Tree Fundamentals

### What is Device Tree?

**Device Tree (DT)** is a data structure that describes hardware topology in a ** human-readable text format ** (.dts files). It's a tree of nodes representing hardware components with properties describing their characteristics.

** Historical Context **: Traditional x86 PCs use BIOS/ACPI for hardware discovery. ARM SoCs lacked this standardization, leading to ** board-specific kernel hard-coding **. Device Tree solves this by externalizing hardware description from kernel source.

** Key Benefits **:
- ** Single kernel binary ** supports multiple boards
- Hardware configuration changes ** don't require kernel recompilation **
- ** Board-specific details ** live in separate files
- ** Vendor customization ** without touching kernel code

### Why Device Tree Matters for Kernel Modules

For a kernel module, Device Tree provides ** runtime configuration **:

```c
// WITHOUT Device Tree (hard-coded)
#define GPIO_LED_PIN 17
#define GPIO_BUTTON_PIN 18

// WITH Device Tree (dynamic)
// Module reads from DT node:
// gpios = <&gpio 17 GPIO_ACTIVE_HIGH>, <&gpio 18 GPIO_ACTIVE_HIGH>;
gpio_led = of_get_named_gpio(np, "gpios", 0);  // Gets 17 from DT
gpio_button = of_get_named_gpio(np, "gpios", 1);  // Gets 18 from DT
```

**Use Cases for Kernel Modules**:
- Extract pin numbers, interrupt numbers, memory addresses
- Read custom configuration parameters (timeouts, thresholds)
- Detect hardware variants (rev1 vs rev2 board)
- Enable/disable features based on board capabilities

### .dts vs .dtb Files

- ** `.dts` **: ** Device Tree Source ** - Human-readable text file
- ** `.dtb` **: ** Device Tree Binary ** - Machine-readable compiled file

**Compilation Flow**:
```
hardware.dts (text)
    ↓ dtc (Device Tree Compiler)
hardware.dtb (binary)
    ↓ Bootloader loads to kernel
Kernel parses at boot
```

**Kernel Usage**:
- At boot, kernel parses `.dtb` into in-memory tree
- Creates `struct device_node` for each DT node
- Platform devices are registered for nodes with `compatible` strings
- Kernel modules can query the tree at runtime

---

## Device Tree and Kernel Modules

### Platform Device Framework Integration

Device Tree nodes automatically become **platform devices** when they have a `compatible` property:

```dts
my_device@40000000 {
    compatible = "vendor,my-device";
    reg = <0x40000000 0x1000>;
};
```

**Kernel Processing**:
1. Bootloader loads `.dtb` into memory
2. Kernel parses DT → builds `struct device_node` tree
3. For each node with `compatible`, calls `of_platform_device_create()`
4. Creates `struct platform_device` and registers it
5. Platform driver with matching `compatible` string gets probed

### Compatible Strings

The **link** between DT nodes and kernel drivers:

```c
// In your kernel module
static const struct of_device_id dt_match_table[] = {
    { .compatible = "vendor,my-device-v1" },
    { .compatible = "vendor,my-device-v2" },
    {}
};
MODULE_DEVICE_TABLE(of, dt_match_table);

static struct platform_driver my_driver = {
    .driver = {
        .name = "my_driver",
        .of_match_table = dt_match_table,  // ← This links DT to driver
    },
    .probe = my_probe,
};

module_platform_driver(my_driver);
```

**Matching Logic**:
- Kernel iterates through all DT nodes
- For each node, iterates through driver's `of_match_table`
- First match wins → calls `driver.probe()`

**Naming Convention**: Use `vendor,device-variant` format (e.g., `raspberrypi,bcm2835-gpio`).

### Property Reading Mechanisms

Device Tree properties are **key-value pairs**:

```dts
my_device {
    string-prop = "hello world";      // String
    int-prop = <42>;                  // 32-bit integer
    bool-prop;                        // Boolean (no value)
    array-prop = <0x10 0x20 0x30>;    // Array of integers
};
```

**Kernel API for Reading**:
```c
struct device_node *np = pdev->dev.of_node;

// Read string
const char *str;
of_property_read_string(np, "string-prop", &str);  // str = "hello world"

// Read integer
u32 val;
of_property_read_u32(np, "int-prop", &val);  // val = 42

// Check boolean
bool exists = of_property_read_bool(np, "bool-prop");  // true if present

// Read array
u32 array[3];
of_property_read_u32_array(np, "array-prop", array, 3);  // array = {0x10, 0x20, 0x30}
```

**Error Handling**: Functions return 0 on success, negative error code on failure.

---

## Deep Dive: Device Tree Module Example

### Module Structure and Data

```c
struct dt_device_data {
    const char *label;
    u32 reg_value;
    u32 custom_value;
    bool has_clock;
};
```

**Purpose**: Store parsed DT properties for use during driver operation.

-  **`label` **: Human-readable name from DT
-  **`reg_value` **: Base address of device register space
-  **`custom_value` **: Driver-specific configuration parameter
-  **`has_clock` **: Boolean flag indicating if device has clock input

**Memory Management**: Uses `devm_kzalloc()` which automatically frees memory when device is removed (no explicit `kfree()` needed).

### Probe Function Walkthrough

```c
static int dt_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;
    struct dt_device_data *data;
    const char *string_prop;
    u32 value;
    int ret;

    // Allocate device data with automatic cleanup
    data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    // Read string property with error handling
    ret = of_property_read_string(np, "label", &string_prop);
    if (ret == 0) {
        data->label = string_prop;
        pr_info("%s: Found label: %s\n", DRIVER_NAME, data->label);
    } else {
        data->label = "unnamed";
        pr_info("%s: No label, using default\n", DRIVER_NAME);
    }

    // Read register base address
    ret = of_property_read_u32(np, "reg", &value);
    if (ret == 0) {
        data->reg_value = value;
        pr_info("%s: Found reg: 0x%x\n", DRIVER_NAME, data->reg_value);
    }

    // Read custom value with default fallback
    ret = of_property_read_u32(np, "lkmpg,custom-value", &value);
    if (ret == 0) {
        data->custom_value = value;
    } else {
        data->custom_value = 42; // Default value
    }

    // Check for boolean property
    data->has_clock = of_property_read_bool(np, "lkmpg,has-clock");

    // Store data for later use in other functions
    platform_set_drvdata(pdev, data);

    return 0;
}
```

** Key Points **:
- ** `devm_kzalloc()` **: Allocates memory tied to device lifecycle; auto-freed on remove
- **of_property_read_string()**: Returns pointer to DT string (no copy needed)
- **Error handling**: Functions return 0 on success, negative on failure
- **platform_set_drvdata()**: Stores private data for retrieval in other functions

### Property Extraction in Detail

**String Properties**:
```c
const char *str;
of_property_read_string(np, "label", &str);
```
- DT string is stored in read-only memory
- Kernel returns pointer directly (no allocation)
- String persists as long as DT is loaded (entire kernel lifetime)

**Integer Properties**:
```c
u32 val;
of_property_read_u32(np, "reg", &val);
```
- Converts big-endian DT value to CPU native format
- Returns error if property doesn't exist or has wrong size

**Boolean Properties**:
```c
bool exists = of_property_read_bool(np, "prop");
```
- Checks for **existence**, not value
- Property with no value = `true`
- Property absent = `false`

### Remove Function and Kernel 6.11+ Changes

```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 11, 0)
static void dt_remove(struct platform_device *pdev)
#else
static int dt_remove(struct platform_device *pdev)
#endif
{
    struct dt_device_data *data = platform_get_drvdata(pdev);
    pr_info("%s: Removing device %s\n", DRIVER_NAME, data->label);
    // Cleanup handled automatically by devm_* functions
}
```

**The Change**: Linux 6.11+ changed `.remove` prototype from `int` to `void`.

**Rationale**: In practice, remove rarely failed. Changing to `void` simplifies error handling and makes it explicit that cleanup must succeed.

**Devres Management**: `devm_kzalloc()` allocated memory is automatically freed by the driver core. No explicit cleanup needed.

---

## Device Tree Source Syntax

### Node Structure

```
node-name@unit-address {
    property = value;
    child-node {
        property = value;
    };
};
```

**Example**:
```dts
ethernet@1c50000 {
    compatible = "allwinner,sun8i-h3-emac";
    reg = <0x01c50000 0x10000>;
    interrupts = <GIC_SPI 82 IRQ_TYPE_LEVEL_HIGH>;
    phy-mode = "mii";
    phy-handle = <&phy1>;
};
```

**Components**:
- **Node name**: `ethernet` (human-readable)
- **Unit address**: `@1c50000` (register base address)
- **Properties**: `compatible`, `reg`, `interrupts`, etc.
- **Child nodes**: `phy1` could be a sub-node

### Property Types

**1. String**:
```dts
property = "hello world";
```
**Kernel read**: `of_property_read_string(np, "property", &str);`

**2. Integer (32-bit)**:
```dts
property = <42>;  // Hex: <0x2a>
```
**Kernel read**: `of_property_read_u32(np, "property", &val);`

**3. Boolean**:
```dts
property;  // No value = true
// Absent = false
```
**Kernel read**: `bool val = of_property_read_bool(np, "property");`

**4. Array**:
```dts
property = <0x10 0x20 0x30>;
```
**Kernel read**: `of_property_read_u32_array(np, "property", array, 3);`

**5. Byte array**:
```dts
property = [01 02 03 04];
```
**Kernel read**: `of_property_read_u8_array();`

**6. Phandle (node reference)**:
```dts
property = <&another_node>;
```
**Kernel read**: `of_parse_phandle(np, "property", 0);`

### Phandle References

**Connecting nodes**:
```dts
i2c@1c2ac00 {
    compatible = "allwinner,sun8i-h3-i2c";
    #address-cells = <1>;
    #size-cells = <0>;
    
    temperature-sensor@48 {
        compatible = "national,lm75";
        reg = <0x48>;  // I2C address
    };
};
```

** `#address-cells = <1>` **: One cell for address (I2C device address)
** `#size-cells = <0>` **: Zero cells for size (I2C devices have no size)

---

## Testing Device Tree Modules

### Platform Device Detection

After loading module, check if device was probed:

```bash
# List all platform devices
ls /sys/bus/platform/devices/

# Should show:
platform-lkmpg_device.0 -> ../../../devices/platform/lkmpg_device.0/

# Check if driver is bound
ls /sys/bus/platform/devices/lkmpg_device.0/driver/
# Should show: ../../../bus/platform/drivers/lkmpg_devicetree

# View device properties
cat /sys/bus/platform/devices/lkmpg_device.0/of_node/label
# Output: LKMPG Test Device

# View driver binding
cat /sys/bus/platform/drivers/lkmpg_devicetree/bind
# Shows which devices driver can bind to
```

### Device Tree Overlays (Raspberry Pi)

**Overlays** allow runtime DT modification without reboot:

```bash
# Example: Enable SPI overlay on Raspberry Pi
sudo dtoverlay spi1-3cs

# Check loaded overlays
sudo vcdbg log msg | grep dts

# Your custom overlay:
# Create my-overlay.dtbo and place in /boot/overlays/
sudo dtoverlay my-overlay
```

**Overlay Syntax**:
```dts
/dts-v1/;
/plugin/;

&i2c1 {
    status = "okay";
    clock-frequency = <400000>;
    
    my-device@48 {
        compatible = "vendor,my-device";
        reg = <0x48>;
    };
};
```

### QEMU Virtualization

**Testing without hardware**:
```bash
# Create custom DTB
qemu-system-arm -M versatilepb -dtb myboard.dtb -kernel mymodule.ko

# Or use QEMU's built-in DT
qemu-system-aarch64 -M virt -kernel mymodule.ko

# In QEMU monitor, inspect DT
(qemu) info qtree  # Show device tree
```

### Debugging with sysfs

**Inspect DT at runtime**:
```bash
# View raw device tree
ls /proc/device-tree/

# Find your device
find /proc/device-tree/ -name "*lkmpg*"

# Read properties
hexdump -C /proc/device-tree/lkmpg_device@0/reg
# Output: 00000000  00 00 00 00  40 00 00 00  00 00 10 00

# Check compatible string
cat /proc/device-tree/lkmpg_device@0/compatible
# Output: lkmpg,example-device
```

---

## Common Device Tree Functions Reference

### String Properties

```c
int of_property_read_string(const struct device_node *np,
                            const char *propname,
                            const char **out_string);
```
- **Returns**: 0 on success, -EINVAL if property missing, -EILSEQ if not a string
- **Example**:
```c
const char *label;
if (of_property_read_string(np, "label", &label) == 0)
    pr_info("Label: %s\n", label);
```

### Numeric Properties

```c
int of_property_read_u32(const struct device_node *np,
                         const char *propname,
                         u32 *out_value);
```
- **Returns**: 0 on success
- **Default fallback pattern**:
```c
u32 val;
int ret = of_property_read_u32(np, "timeout", &val);
if (ret)
    val = DEFAULT_TIMEOUT;  // Use default if not in DT
```

**Array variant**:
```c
int of_property_read_u32_array(const struct device_node *np,
                               const char *propname,
                               u32 *out_values,
                               size_t sz);
```
- **sz**: Number of elements expected
- **Example**: `of_property_read_u32_array(np, "coordinates", coords, 3);`

### Boolean Properties

```c
bool of_property_read_bool(const struct device_node *np,
                           const char *propname);
```
- **Returns**: `true` if property exists (any value), `false` if absent
- **Use case**: Feature flags, optional hardware
```c
bool has_dma = of_property_read_bool(np, "dma-supported");
if (has_dma)
    enable_dma();
```

### Advanced Parsing Functions

**1. Get property length**:
```c
int of_property_count_elems_of_size(const struct device_node *np,
                                    const char *propname,
                                    int elem_size);
```
- Returns element count (e.g., for arrays)

**2. Parse phandle**:
```c
struct device_node *of_parse_phandle(const struct device_node *np,
                                     const char *phandle_name,
                                     int index);
```
- **Use case**: Reference to another DT node
```dts
gpio = <&gpio_controller 17>;
```
```c
struct device_node *gpio_np = of_parse_phandle(np, "gpio", 0);
int gpio_pin = of_get_named_gpio(np, "gpio", 0);
```

**3. Match device**:
```c
const struct of_device_id *of_match_device(
    const struct of_device_id *matches,
    const struct device *dev);
```
- Returns matching entry from `of_device_id` table

---

## Kernel Optimizations

### likely() and unlikely() Macros

**Purpose**: Provide **branch prediction hints** to the compiler.

**Mechanism**:
```c
if (likely(success)) {
    // Branch predicted taken → code placed sequentially
    // CPU pre-fetches this path
} else {
    // Branch predicted not taken → jump target
    // CPU may not pre-fetch
}
```

**Use Case**: Error handling (rare) vs. success path (common):
```c
buffer = kmalloc(size, GFP_KERNEL);
if (unlikely(!buffer)) {  // Allocation failure is rare
    pr_err("Out of memory\n");
    return -ENOMEM;
}
// Success path: CPU optimizes for this case
```

**Assembly Impact**:
```asm
# Without unlikely:
    cmp    rax, 0
    jne    .success      # 50/50 chance of misprediction
    jmp    .error

# With unlikely:
    cmp    rax, 0
    je     .error        # Predict not taken
    # Success code placed here (no jump)
```

** ** When to Use **:
- ** `likely()` **: Condition is ** true > 95% ** of the time (e.g., success path)
- ** `unlikely()` **: Condition is ** false > 95% ** of the time (e.g., error path)

** ** Real-world Example **:
```c
// In networking: Packet reception is common, errors are rare
if (unlikely(skb->len < ETH_HLEN)) {
    dev_kfree_skb(skb);
    return -EINVAL;
}
```

### Static Keys and Jump Labels

**Concept**: **Runtime code patching** to eliminate branch overhead entirely.

**Traditional boolean flag**:
```c
static bool enable_feature = false;

void hot_function(void)
{
    if (enable_feature) {  // Branch overhead every call!
        // Feature code
    }
}
```

** Static key **:
```c
DEFINE_STATIC_KEY_FALSE(feature_key);

void hot_function(void)
{
    if (static_branch_unlikely(&feature_key)) {  // No branch when disabled!
        // Feature code
    }
}
```

** How it Works **:
1. ** Compilation **: Compiler inserts a ** nop ** instruction (no-op) in the disabled path
2. ** Runtime **: When key is ** enabled **, kernel ** rewrites the nop to a jump ** to the feature code
3. ** Performance**: **Zero overhead** when disabled (no branch prediction, no pipeline flush)
4. **Trade-off**: Initial enable/disable is expensive (synchronizes all CPUs)

**Use Cases**:
- **Tracepoints**: Low-overhead debugging disabled by default
- **Feature flags**: Optional functionality that is usually off
- **Performance counters**: Rarely-enabled monitoring

### Performance Impact Analysis

**Benchmark Comparison**:
- **Without optimization**: 100M iterations, condition always false
  - Traditional if: ~3.2 seconds (branch overhead)
  - Static key: ~3.0 seconds (nop overhead negligible)

- **With optimization**: 100M iterations, condition always true
  - Traditional if: ~3.2 seconds (branch correctly predicted)
  - Static key: ~3.1 seconds (slightly better due to no misprediction)

**Key Insight**: Static keys shine when the condition is **rarely true** (<1% of time) and called in **hot paths** (millions of times per second).

**Enable/Disable Overhead**:
- `static_branch_enable()`: ~50 microseconds (synchronizes all CPUs)
- `static_branch_disable()`: ~50 microseconds
- **Only call at initialization or infrequently! **

### Static Key Example Module Walkthrough

```c
static DEFINE_STATIC_KEY_FALSE(fkey);

// Fastpath code
pr_info("fastpath 1\n");
if (static_branch_unlikely(&fkey))  // Initially nop
    pr_alert("do unlikely thing\n");
pr_info("fastpath 2\n");
```

** Initial state **:
```
fastpath 1
[nop instruction]  ← no-op, falls through
fastpath 2
```

** After `echo enable > /dev/key_state`**:
```
fastpath 1
[jmp to unlikely_code]  ← patched to jump
fastpath 2

unlikely_code:
    pr_alert("do unlikely thing");
    jmp back
```

### Testing Static Keys

```bash
# Compile and load
make
sudo insmod static_key.ko

# Default state (disabled)
cat /dev/key_state
# Output: disabled

# Observe fastpath (no "do unlikely thing" message)
dmesg | tail

# Enable the key
echo enable | sudo tee /dev/key_state

# Now observe slowpath (with "do unlikely thing")
cat /dev/key_state
# Output: enabled

# Disable again
echo disable | sudo tee /dev/key_state

# Monitor overhead
# Use trace-cmd to measure function latency
sudo trace-cmd record -e function_graph -F cat /dev/key_state
sudo trace-cmd report
```

---

## Common Pitfalls

### Standard Libraries

**The Mistake **:
```c
// In kernel module
#include <stdio.h>      // WRONG!
#include <stdlib.h>     // WRONG!

printf("Hello\n");      // Won't compile
malloc(100);            // Won't compile
```

**Why**: Kernel modules run in **kernel space**, not user space. Standard libraries (libc) only exist in user space.

**Kernel Alternatives **:
```c
#include <linux/printk.h>    // For pr_info()
#include <linux/slab.h>      // For kmalloc()

pr_info("Hello\n");           // Correct
kmalloc(100, GFP_KERNEL);     // Correct
```

**Rule**: **Only use functions in `/proc/kallsyms`** (kernel symbols).

**Checking Available Functions**:
```bash
# See all kernel functions you can call
cat /proc/kallsyms | grep ' T '

# Find memory allocation functions
cat /proc/kallsyms | grep kmalloc

# Result: ffffffffa0b4d2b0 T kmalloc
#         ^^^^^^^^^^^^^^^^  Address (changes per boot)
```

### Disabling Interrupts

**The Danger**:
```c
local_irq_disable();  // Disables all interrupts on this CPU

// Do something that takes a long time...
mdelay(1000);  // System frozen for 1 second!

local_irq_enable();  // Re-enable interrupts
```

**Consequences**:
- **System freeze**: Timer interrupts blocked → scheduler stops
- **Data loss**: Network packets dropped, missed hardware events
- **Watchdog reset**: If interrupts disabled too long, hardware watchdog reboots system

**Correct Pattern**:
```c
unsigned long flags;

// Save interrupt state and disable
local_irq_save(flags);
// Critical section: MUST be < 100 microseconds
// Do atomic operation here...
local_irq_restore(flags);  // Restore previous state
```

**When It's OK**:
- **Atomic operations**: Updating a shared 64-bit variable on 32-bit system
- **Critical sections**: Very short (< 10μs) hardware register access
- **Spinlock acquisition**: `spin_lock_irqsave()` disables interrupts internally

**When It's NOT OK**:
- **Memory allocation**: `kmalloc()` can sleep
- **Sleeping**: `msleep()`, `usleep_range()`
- **User copy**: `copy_from_user()` may page fault
- **Mutex locks**: Can sleep waiting for lock

**Alternative: Threaded IRQs**:
```c
// Instead of disabling interrupts, use threaded IRQs
static irqreturn_t my_isr(int irq, void *dev)
{
    // Top half: quick acknowledgment
    return IRQ_WAKE_THREAD;
}

static irqreturn_t my_thread_fn(int irq, void *dev)
{
    // Bottom half: can sleep safely
    msleep(10);  // Safe here!
    return IRQ_HANDLED;
}
```

### Memory Allocation Context

**Pitfall**: Using wrong GFP flags:
```c
// In interrupt context
void *ptr = kmalloc(100, GFP_KERNEL);  // WRONG! Can sleep!

// Correct
void *ptr = kmalloc(100, GFP_ATOMIC);  // Won't sleep
```

**GFP Flags**:
-  **`GFP_KERNEL` **: Normal allocation, ** can sleep ** (use in process context)
-  **`GFP_ATOMIC` **: Emergency allocation, ** never sleeps ** (use in interrupt, atomic, spinlock context)
-  **`GFP_USER` **: For user-space pages
-  **`GFP_HIGHUSER` **: For user-space high memory

** Rule of Thumb **: If you hold a spinlock or are in interrupt context, use `GFP_ATOMIC`.

### Configuration Dependencies

**Pitfall**: Module depends on kernel config not enabled:
```c
// In code
DEFINE_STATIC_KEY_FALSE(my_key);  // Requires CONFIG_JUMP_LABEL=y

// If kernel built without CONFIG_JUMP_LABEL:
// Compilation error: "undefined reference to `jump_label_set'"
```

**Solution**: Use `#ifdef` guards:
```c
#ifdef CONFIG_JUMP_LABEL
DEFINE_STATIC_KEY_FALSE(my_key);
#endif

// In usage
#ifdef CONFIG_JUMP_LABEL
if (static_branch_unlikely(&my_key))
    // ...
#endif
```

**Checking Configuration**:
```bash
# See kernel config
cat /boot/config-$(uname -r) | grep JUMP_LABEL
# Should show: CONFIG_JUMP_LABEL=y

# Or check /proc
zcat /proc/config.gz | grep JUMP_LABEL
```

---

## Best Practices

### Device Tree Best Practices

**1. Use meaningful compatible strings**:
```dts
// Good
compatible = "raspberrypi,pi3-gpio-shifter", "microchip,mcp23017";

// Bad (too generic)
compatible = "gpio-expander";
```

**2. Document custom properties**:
```dts
// Add documentation node
/ {
    ...
    __symbols__ {
        my_device = "/soc/i2c@1c2ac00/my-device@48";
    };
};

// And in binding document:
// Documentation/devicetree/bindings/vendor,my-device.txt
```

**3. Validate all properties**:
```c
// Always check return values
if (of_property_read_u32(np, "critical-param", &val)) {
    dev_err(dev, "Missing required 'critical-param'\n");
    return -EINVAL;
}
```

### Optimization Best Practices

**1. Profile first, optimize second**:
```bash
# Use perf to find hot paths
sudo perf record -g ./my-benchmark
sudo perf report
# Only optimize functions in top 10%
```

**2. Use static keys for rarely-enabled features**:
```c
DEFINE_STATIC_KEY_FALSE(trace_enabled);

void hot_function(void)
{
    if (static_branch_unlikely(&trace_enabled)) {
        trace_event();  // Only when debugging
    }
    // Normal code
}

// Enable only during debug session
echo 1 > /sys/kernel/debug/tracing/events/my_event/enable
```

**3. Don't overuse likely/unlikely**:
- Only when probability is **extreme** (>95% or <5%)
- Modern branch predictors are very good
- Wrong hint can **hurt** performance
- Profile to confirm prediction accuracy

### Pitfall Avoidance Checklist

**Before compiling**:
- [ ] No `<stdio.h>`, `<stdlib.h>`, `<string.h>`
- [ ] All functions from `/proc/kallsyms` or kernel headers
- [ ] Correct GFP flags for context
- [ ] Interrupt handlers don't sleep
- [ ] Spinlock-protected sections are short (<10μs)

**Before insmod**:
- [ ] Kernel config matches module requirements
- [ ] All dependencies loaded
- [ ] Symbol versions match (`modinfo mymodule.ko`)

---

## Further Resources

### Device Tree
- **Official Specification**: https://www.devicetree.org/specifications/
- **Kernel Documentation**: `Documentation/devicetree/`
- **Bindings**: `Documentation/devicetree/bindings/`
- **DT Compiler**: `scripts/dtc/dtc` in kernel source

### Kernel Optimizations
- **likely/unlikely**: `include/linux/compiler.h`
- **Static keys**: `Documentation/static-keys.txt`
- **Jump Labels**: `include/linux/jump_label.h`

### Common Pitfalls
- **Coding style**: `Documentation/process/coding-style.rst`
- **Sparse checker**: `make C=2` (finds mixing of user/kernel pointers)
- **Coccinelle**: `scripts/coccinelle/` (automated patch generation)

### Tools
- **dtc**: Device Tree Compiler
  ```bash
  sudo apt install device-tree-compiler
  dtc -I dts -O dtb -o output.dtb input.dts
  ```
- **dtc -O dts -I dtb` (decompile dtb to dts for inspection)
- **evtest**: Input device testing
- **trace-cmd**: Kernel tracing for optimization

---

## Summary

**Device Tree** enables hardware description without kernel hard-coding. Use `of_property_read_*()` functions to extract configuration. The **platform driver framework** automatically binds drivers to DT nodes via `compatible` strings.

**Optimizations**: Use `likely()`/`unlikely()` for extreme probability cases. **Static keys** provide near-zero overhead for rarely-enabled code paths but have expensive enable/disable.

**Pitfalls **: ** Never ** use user-space libraries in kernel modules. ** Never** sleep in interrupt context. **Never** disable interrupts for long periods. Always validate DT properties.

Mastering these concepts transforms kernel module development from "make it work" to "make it robust, maintainable, and production-ready."