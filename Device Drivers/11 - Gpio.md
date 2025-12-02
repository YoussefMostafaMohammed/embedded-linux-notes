# Linux GPIO Device Drivers - Complete Technical Guide

## Table of Contents
1. [Chapter Overview](#chapter-overview)
2. [Why Not Direct Register Access?](#why-not-direct-register-access)
3. [The Linux GPIO Subsystem Architecture](#the-linux-gpio-subsystem-architecture)
4. [GPIO API Functions - Origins and Purpose](#gpio-api-functions-origins-and-purpose)
5. [Example 1: GPIO LED Control Driver](#example-1-gpio-led-control-driver)
   - [Hardware Setup](#led-hardware-setup)
   - [Complete Code Walkthrough](#led-code-walkthrough)
   - [Build and Testing Procedures](#led-build-and-test)
6. [Example 2: DHT11 Temperature/Humidity Sensor](#example-2-dht11-sensor-driver)
   - [Hardware Setup](#dht11-hardware-setup)
   - [Protocol Implementation Details](#dht11-protocol-details)
   - [Complete Code Walkthrough](#dht11-code-walkthrough)
   - [Build and Testing Procedures](#dht11-build-and-test)
7. [API Deep Dive: Function-by-Function Reference](#api-deep-dive)
8. [Critical Concepts and Best Practices](#critical-concepts)
9. [Troubleshooting and Debugging](#troubleshooting)
10. [Further Resources and Next Steps](#further-resources)

---

## Chapter Overview

This chapter explores Linux kernel GPIO (General Purpose Input/Output) device driver development through two practical examples: an LED controller and a DHT11 sensor reader. GPIO pins serve as programmable digital interfaces that can be configured as either inputs or outputs, enabling communication between the Raspberry Pi and external electronic components.

**Key Learning Objectives:**
- Understand the Linux GPIO subsystem and its abstraction layers
- Master device file creation and character driver implementation
- Implement GPIO request, configuration, and data transfer operations
- Handle kernel-space vs user-space memory access
- Manage device cleanup and error recovery
- Work with timing-sensitive protocols in kernel space

---

## Why Not Direct Register Access?

A fundamental question arises: **"Why can't I just write directly to the Raspberry Pi's GPIO registers instead of using these API functions?"** The answer involves multiple critical aspects of modern kernel development:

### 1. **Hardware Abstraction and Portability**
- **Direct register access** would lock your code to the BCM2835/BCM2711/BCM2712 chipset used in Raspberry Pi models
- The Linux GPIO API provides **architecture independence** - the same code works on x86, ARM, MIPS, and other platforms with minimal changes
- Register addresses, bit layouts, and functionality vary significantly between SoC generations (BCM2835 vs BCM2712)

### 2. **Concurrency and Race Condition Protection**
The GPIO subsystem handles complex synchronization:
- Multiple processes/threads might attempt GPIO access simultaneously
- The kernel maintains **usage counters** and **locking mechanisms** to prevent conflicts
- Direct register writes bypass these safety measures, causing **unpredictable behavior**

### 3. **System Stability and Security**
- **Kernel oops/panic prevention**: The GPIO API validates pin numbers and configurations
- **Memory protection**: GPIO registers are memory-mapped in protected space; improper access triggers segmentation faults
- **Privilege isolation**: Only the kernel should have direct hardware access; user processes go through controlled interfaces
- **Power management**: The subsystem coordinates with PM frameworks to maintain GPIO states during sleep/wake cycles

### 4. **Multiplexing and Resource Management**
Modern SoCs have complex pin multiplexing:
- A single physical pin can serve multiple functions (GPIO, UART, SPI, I2C, etc.)
- The GPIO API coordinates with the **pinctrl subsystem** to manage function selection
- Direct register access ignores these dependencies, potentially breaking other drivers

### 5. **Debug and Monitoring Infrastructure**
The kernel GPIO subsystem provides:
- **debugfs interface** (`/sys/kernel/debug/gpio`) for real-time monitoring
- **SysFS integration** (`/sys/class/gpio`) for user-space access
- **Tracepoints** for debugging state changes

**Direct register access is only acceptable in:**
- Bootloaders (before kernel starts)
- Bare-metal firmware
- Architecture-specific platform code within the kernel itself

---

## The Linux GPIO Subsystem Architecture

The GPIO API originates from **gpiolib**, a core kernel subsystem introduced in Linux 2.6.30+ and significantly enhanced over subsequent versions.

### Layered Architecture:

```
User Space
    ↓ (system calls)
Character Device Interface (/dev/gpio_led)
    ↓ (file_operations)
Your Device Driver
    ↓ gpio_request(), gpio_set_value()
Linux GPIO Subsystem (gpiolib)
    ↓ (abstract interface)
Platform-Specific Driver (bcm2835-gpio)
    ↓ (memory-mapped registers)
Hardware (BCM2712 SoC → GPIO Pins)
```

### Key Components:

1. **gpiolib core** (`drivers/gpio/gpiolib.c`): Provides the unified API
2. **Platform drivers** (`drivers/gpio/gpio-bcm2835.c`): Implements hardware-specific operations
3. **Device Tree**: Defines GPIO controllers and pin configurations
4. **debugfs**: Runtime debugging interface
5. **Character device interface**: Your driver connects here

### API Evolution:
- **Linux 6.4.0+**: Removed the `.owner` field from `class_create()` and changed function signatures
- **Linux 6.10+**: Removed `gpio_request_array()` - use iterative requests instead
- **Modern kernels**: Prefer descriptor-based GPIO (`gpiod_` functions) over legacy integer-based API

---

## GPIO API Functions - Origins and Purpose

### Core API Functions Explained:

#### `gpio_request(unsigned gpio, const char *label)`
- **Origin**: `drivers/gpio/gpiolib.c` - gpiolib core
- **Purpose**: Claims exclusive ownership of a GPIO pin
- **Parameters**: 
  - `gpio`: Pin number (platform-specific, e.g., `4` for GPIO4)
  - `label`: Descriptive name appears in `/sys/kernel/debug/gpio`
- **Failure**: Returns negative error code if already claimed or invalid
- **Why use it?**: Prevents conflicts, registers ownership, enables debug visibility

#### `gpio_direction_output(unsigned gpio, int value)`
- **Origin**: Platform driver (e.g., `gpio-bcm2835.c`)
- **Purpose**: Configures pin as output and sets initial state
- **Atomic operation**: Direction + value in one step to prevent glitches
- **Why use it?**: Ensures clean transitions, coordinates with direction changes

#### `gpio_direction_input(unsigned gpio)`
- **Purpose**: Configures pin as input, tri-stating the output driver
- **Why use it?**: Properly configures pull-up/down resistors and input buffers

#### `gpio_set_value(unsigned gpio, int value)`
- **Origin**: Platform driver
- **Purpose**: Sets output pin high (1) or low (0)
- **Constraint**: Only valid after `gpio_direction_output()`
- **Why use it?**: Provides value caching, validation, and traceability

#### `gpio_get_value(unsigned gpio)`
- **Purpose**: Reads current pin state (0 or 1)
- **Why use it?**: Handles input synchronization, debouncing logic, and voltage level translation

---

## Example 1: GPIO LED Control Driver

### LED Hardware Setup

**Components:**
- Raspberry Pi 5 (or Pi 4/3)
- Red LED (forward voltage ~2V)
- 220Ω current-limiting resistor
- 2x Female-to-male jumper wires

**Circuit Connections:**
```
Raspberry Pi GPIO4 (Pin 7) → LED Anode (long leg)
LED Cathode (short leg) → 220Ω Resistor → GND (Pin 6)
```

**Electrical Considerations:**
- GPIO4 outputs 3.3V logic high
- Current calculation: (3.3V - 2V) / 220Ω ≈ 5.9mA (safe for GPIO)
- Resistor prevents overcurrent and GPIO damage
- Pull-down resistor is inherent in the GPIO configuration

### LED Code Walkthrough

```c
#include <linux/cdev.h>
#include <linux/delay.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/gpio.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/printk.h>
#include <linux/types.h>
#include <linux/uaccess.h>
#include <linux/version.h>
#include <asm/errno.h>

#define DEVICE_NAME "gpio_led"
#define DEVICE_CNT 1
#define BUF_LEN 2

static char control_signal[BUF_LEN];
static unsigned long device_buffer_size = 0;

struct LED_dev {
    dev_t dev_num;
    int major_num, minor_num;
    struct cdev cdev;
    struct class *cls;
    struct device *dev;
};

static struct LED_dev led_device;

/* GPIO Definition Table */
static struct gpio leds[] = { 
    { 4, GPIOF_OUT_INIT_LOW, "LED 1" } 
};
```

**Key Structures:**
- **`struct LED_dev`**: Encapsulates all device state
- **`struct gpio leds[]`**: GPIO descriptor array containing:
  - Pin number (`4`)
  - Initial flags (`GPIOF_OUT_INIT_LOW` = output, start low)
  - Debug label (`"LED 1"`)

```c
/* File Operations Implementation */

static int device_open(struct inode *inode, struct file *file)
{
    return 0;  // Simple implementation, no per-open state
}
```

**Open Handler**: Called when `open("/dev/gpio_led", ...)` is invoked. In production drivers, you'd handle private data allocation here.

```c
static ssize_t device_write(struct file *file, const char __user *buffer,
                            size_t length, loff_t *offset)
{
    device_buffer_size = min(BUF_LEN, length);
    
    /* Critical: Copy data from user space to kernel space */
    if (copy_from_user(control_signal, buffer, device_buffer_size)) {
        return -EFAULT;  // Bad address - user gave invalid pointer
    }
    
    /* Command parsing - minimal but effective */
    switch (control_signal[0]) {
    case '0':
        gpio_set_value(leds[0].gpio, 0);
        pr_info("LED OFF");
        break;
    case '1':
        gpio_set_value(leds[0].gpio, 1);
        pr_info("LED ON");
        break;
    default:
        pr_warn("Invalid value!\n");
        break;
    }
    
    *offset += device_buffer_size;
    return device_buffer_size;  // Return bytes consumed
}
```

**Write Handler Deep Dive**:
-  **`copy_from_user()`**  : **MANDATORY** for user data - prevents kernel memory corruption
-  **`min()`**  : Bounds checking prevents buffer overflow
-  **`control_signal[0]`**  : Only checks first character ('0' or '1')
- Return value: Must be positive bytes consumed or negative error code

```c
static int __init led_init(void)
{
    int ret = 0;
    
    /* Device Number Allocation Strategy */
    if (led_device.major_num) {
        /* Static allocation - for fixed major numbers */
        led_device.dev_num = MKDEV(led_device.major_num, led_device.minor_num);
        ret = register_chrdev_region(led_device.dev_num, DEVICE_CNT, DEVICE_NAME);
    } else {
        /* Dynamic allocation - recommended for most drivers */
        ret = alloc_chrdev_region(&led_device.dev_num, 0, DEVICE_CNT, DEVICE_NAME);
    }
    
    if (ret < 0) {
        pr_alert("Failed to register character device, error: %d\n", ret);
        return ret;
    }
```

**Device Registration Flow**:
1. **Dynamic allocation** (`alloc_chrdev_region`): Kernel assigns an unused major number - **preferred**
2. **Static allocation** (`register_chrdev_region`): Driver requests a specific major number - for legacy compatibility
3. `MKDEV()`: Creates combined device number from major/minor

```c
    /* Character Device Registration */
    cdev_init(&led_device.cdev, &fops);
    ret = cdev_add(&led_device.cdev, led_device.dev_num, 1);
    if (ret) {
        pr_err("Failed to add the device to the system\n");
        goto fail1;
    }
```

**cdev structure**: Links file operations to device numbers. The kernel maintains a global table of these.

```c
    /* Device Class Creation (for udev/auto-devices) */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 4, 0)
    led_device.cls = class_create(DEVICE_NAME);
#else
    led_device.cls = class_create(THIS_MODULE, DEVICE_NAME);
#endif

    if (IS_ERR(led_device.cls)) {
        pr_err("Failed to create class for device\n");
        ret = PTR_ERR(led_device.cls);
        goto fail2;
    }
```

**Kernel 6.4.0+ Change**: `class_create()` no longer requires `THIS_MODULE` as first parameter.

```c
    /* Device File Creation */
    led_device.dev = device_create(led_device.cls, NULL, led_device.dev_num,
                                   NULL, DEVICE_NAME);
    if (IS_ERR(led_device.dev)) {
        pr_err("Failed to create the device file\n");
        ret = PTR_ERR(led_device.dev);
        goto fail3;
    }
```

**Device Creation**: This automatically creates `/dev/gpio_led` with proper permissions via udev.

```c
    /* GPIO Request and Configuration */
    ret = gpio_request(leds[0].gpio, leds[0].label);
    if (ret) {
        pr_err("Unable to request GPIOs for LEDs: %d\n", ret);
        goto fail4;
    }
    
    ret = gpio_direction_output(leds[0].gpio, leds[0].flags);
    if (ret) {
        pr_err("Failed to set GPIO %d direction\n", leds[0].gpio);
        goto fail5;
    }
    return 0;
```

**GPIO Initialization**: Order matters! Request → Configure. The `goto failX` pattern provides clean rollback.

```c
    /* Error Handling Cleanup */
fail5:
    gpio_free(leds[0].gpio);
fail4:
    device_destroy(led_device.cls, led_device.dev_num);
fail3:
    class_destroy(led_device.cls);
fail2:
    cdev_del(&led_device.cdev);
fail1:
    unregister_chrdev_region(led_device.dev_num, DEVICE_CNT);
    return ret;
}
```

**Cleanup Order**: Reverse of initialization - free resources in opposite order.

### LED Build and Test

**Makefile** (create this):
```makefile
obj-m := led.o
KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

**Build and Install**:
```bash
make
sudo insmod led.ko
dmesg | tail  # Should show "Major = X, Minor = 0" and device creation
```

**Testing**:
```bash
# Turn LED on
echo "1" | sudo tee /dev/gpio_led
# Check kernel log
dmesg | grep LED

# Turn LED off
echo "0" | sudo tee /dev/gpio_led

# Cleanup
sudo rmmod led
```

---

## Example 2: DHT11 Sensor Driver

### DHT11 Hardware Setup

**Components:**
- Raspberry Pi 5
- DHT11 sensor module (3-pin or 4-pin)
- Jumper wires

**Pin Connections:**
```
DHT11 VCC → Raspberry Pi 3.3V (Pin 1)
DHT11 GND → Raspberry Pi GND (Pin 6)
DHT11 DATA → Raspberry Pi GPIO4 (Pin 7) + 10kΩ pull-up resistor
```

**Protocol Specifications:**
- Single-wire bidirectional protocol
- Timing-critical: microsecond-level precision required
- Data format: 40 bits (8b humidity + 8b temp + 8b temp decimal + 8b checksum)
- Bit encoding: High pulse width determines 0/1 (26-28μs vs 70μs)

### DHT11 Protocol Details

**Communication Sequence:**
1. **Host Start Signal**: Pull data line LOW for 18-20ms
2. **Host Release**: Pull HIGH for 20-40μs
3. **Sensor Response**: Pulls LOW for 80μs (ack)
4. **Sensor Ready**: Releases HIGH for 80μs
5. **Data Transmission**: 40 bits of data (LSB first)
6. **Line Return**: Sensor releases line to idle HIGH

**Bit Encoding:**
- **'0'**: 50μs total (26-28μs LOW + 23-25μs HIGH)
- **'1'**: 70μs total (26-28μs LOW + 44-46μs HIGH)

### DHT11 Code Walkthrough

```c
#define GPIO_PIN_4 575
/* NOTE: On Raspberry Pi, GPIO numbers vary:
 * - GPIO4 = 4 on most models
 * - GPIO4 = 575 on some kernel versions with pinctrl abstraction
 * Use: gpioinfo | grep GPIO4 to find correct number
 */
```

**Critical GPIO Number Issue**: The code uses `575` which is specific to certain Raspberry Pi kernel configurations. **Always verify** with:
```bash
gpioinfo | grep GPIO4
cat /sys/kernel/debug/gpio | grep GPIO4
```

```c
static struct gpio dht11[] = { 
    { GPIO_PIN_4, GPIOF_OUT_INIT_HIGH, "Signal" } 
};
```

**DHT11 Dev Structure**: Similar to LED but optimized for reading:

```c
static int dht11_read_data(void)
{
    int timeout;
    uint8_t sensor_data[5] = { 0 };
    uint8_t i, j;
    
    /* Step 1: Send start signal - pull low for 20ms */
    gpio_set_value(dht11[0].gpio, 0);
    mdelay(20);  // Millisecond delay
    
    /* Step 2: Release line and wait */
    gpio_set_value(dht11[0].gpio, 1);
    udelay(30);  // Microsecond delay
    
    /* Step 3: Switch to input mode */
    gpio_direction_input(dht11[0].gpio);
    udelay(2);
```

**Timing Critical Section**: The `mdelay()`/`udelay()` functions are **busy-wait loops** that block the CPU but provide precise timing. Sleep functions would be too coarse.

```c
    /* Step 4: Wait for sensor response (low pulse) */
    timeout = 300;
    while (gpio_get_value(dht11[0].gpio) && timeout--)
        udelay(1);
    if (timeout == -1)
        return -ETIMEDOUT;
```

**Timeout Mechanism**: Each unit of `timeout` represents 1μs. `timeout = 300` = 300μs maximum wait. This prevents infinite hangs if sensor is disconnected.

```c
    /* Step 5: Read 40 bits (5 bytes) of data */
    for (j = 0; j < 5; j++) {
        uint8_t byte = 0;
        for (i = 0; i < 8; i++) {
            /* Wait for low pulse (start of bit) */
            timeout = 300;
            while (gpio_get_value(dht11[0].gpio) && timeout--)
                udelay(1);
            
            /* Wait for high pulse (data) */
            timeout = 300;
            while (!gpio_get_value(dht11[0].gpio) && timeout--)
                udelay(1);
            
            /* Critical: Wait 50μs and sample */
            udelay(50);
            byte <<= 1;
            if (gpio_get_value(dht11[0].gpio))
                byte |= 0x01;  // It's a '1' bit
        }
        sensor_data[j] = byte;
    }
```

**Bit Sampling Strategy**: After the rising edge, wait 50μs:
- For a '0' bit: HIGH pulse ends at ~27μs, so at 50μs line is LOW → reads 0
- For a '1' bit: HIGH pulse lasts ~46μs, so at 50μs line is still HIGH → reads 1

```c
    /* Step 6: Verify checksum */
    if (sensor_data[4] != (uint8_t)(sensor_data[0] + sensor_data[1] +
                                    sensor_data[2] + sensor_data[3]))
        return -EIO;  // I/O error - data corrupted
    
    /* Step 7: Format output message */
    gpio_direction_output(dht11[0].gpio, 1);
    sprintf(msg, "Humidity: %d%%\nTemperature: %d deg C\n", 
            sensor_data[0], sensor_data[2]);
    return 0;
}
```

**Checksum Validation**: DHT11 sends 5 bytes. The 5th byte should equal the sum of the first 4 (mod 256). This detects communication errors.

```c
static int device_open(struct inode *inode, struct file *file)
{
    int ret, retry;
    
    /* Retry mechanism - sensor may be busy */
    for (retry = 0; retry < 5; ++retry) {
        ret = dht11_read_data();
        if (ret == 0)
            return 0;
        msleep(10);  // Wait 10ms before retry
    }
    
    gpio_direction_output(dht11[0].gpio, 1);
    return ret;  // Return error after 5 failures
}
```

**Open Handler Insights**: Unlike LED, DHT11 doesn't need write operations. The sensor reading occurs **during open()**, which is unusual but effective for simple character devices.

```c
static ssize_t device_read(struct file *filp, char __user *buffer,
                           size_t length, loff_t *offset)
{
    int msg_len = strlen(msg);
    
    if (*offset >= msg_len)
        return 0;  // EOF - all data read
    
    size_t remain = msg_len - *offset;
    size_t bytes_read = min(length, remain);
    
    if (copy_to_user(buffer, msg + *offset, bytes_read))
        return -EFAULT;
    
    *offset += bytes_read;
    return bytes_read;
}
```

**Read Handler Pattern**: Implements **sequential reading**:
- First call reads entire message, increments offset
- Second call sees `*offset >= msg_len` and returns 0 (EOF)
- This allows standard shell tools like `cat` to work correctly

### DHT11 Build and Test

**Makefile**:
```makefile
obj-m := dht11.o
KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

**Build and Test**:
```bash
make
sudo insmod dht11.ko
dmesg | tail  # Check for successful loading
cat /sys/kernel/debug/gpio  # Verify GPIO4 is claimed

# Read sensor (takes ~1 second due to protocol)
sudo cat /dev/dht11

# Expected Output:
# Humidity: 61%
# Temperature: 30°C

# Cleanup
sudo rmmod dht11
```

**Troubleshooting**:
```bash
# If timeout errors occur:
sudo cat /sys/kernel/debug/gpio | grep GPIO4  # Verify pin is free
gpioinfo  # Check pin configuration
# Try increasing timeout values in code
```

---

## API Deep Dive

### Legacy vs Modern GPIO API

**Current Code Uses (Legacy):**
```c
gpio_request(pin, label);
gpio_direction_output(pin, value);
gpio_set_value(pin, value);
```

**Modern Alternative (Descriptor API):**
```c
struct gpio_desc *desc;
desc = gpiod_get(dev, "led", GPIOD_OUT_LOW);
gpiod_set_value(desc, 1);
gpiod_put(desc);
```

**Why the book uses legacy API:**
- Simpler for learning
- Works on older kernels
- Direct integer pin numbers
- Less setup required

### Kernel Version Compatibility

**Linux 6.4.0+ Changes:**
```c
/* Old (pre-6.4) */
led_device.cls = class_create(THIS_MODULE, DEVICE_NAME);

/* New (6.4+) */
led_device.cls = class_create(DEVICE_NAME);
```

**Rationale**: The module pointer became redundant with improved reference counting. The `class_create()` macro now extracts the module automatically.

### Timing Functions Reference

| Function | Precision | Behavior | Use Case |
|----------|-----------|----------|----------|
| `msleep()` | ~1ms | Sleeps (scheduler) | Non-critical delays > 10ms |
| `mdelay()` | ~1ms | Busy-wait (CPU blocked) | Precise millisecond delays |
| `usleep_range()` | ~10μs | Sleeps + busy-wait | Flexible microsecond delays |
| `udelay()` | ~1μs | Busy-wait (CPU blocked) | Critical microsecond timing |
| `ndelay()` | ~10ns | Busy-wait | Nanosecond precision (rare) |

**Critical Warning**: Busy-wait functions (`mdelay`, `udelay`) **block the CPU** and should only be used for short durations (< 1ms) in time-critical sections.

---

## Critical Concepts

### 1. **copy_from_user() and copy_to_user()**

**Mandatory for user-space buffers:**
- User-space memory might be swapped out, causing page faults
- The kernel can't directly dereference user pointers
- These functions handle page fault exceptions, returning error if failed
- **Never** use `memcpy()` with user pointers → kernel oops/crash

### 2. **Error Handling with Goto**

The `goto fail` pattern is **kernel-standard**:
- Provides centralized cleanup
- Ensures resources are freed in reverse allocation order
- Prevents resource leaks
- More readable than nested if/else

### 3. **Device File Permissions**

Your driver creates `/dev/gpio_led` with default permissions (root-only). To make it accessible:

```bash
sudo chmod 666 /dev/gpio_led
# Or better, add udev rule:
echo 'SUBSYSTEM=="gpio_led", MODE="0666"' | sudo tee /etc/udev/rules.d/99-gpio.rules
```

### 4. **GPIO Numbering Pitfalls**

Raspberry Pi has **three numbering schemes**:
- **BCM (Broadcom)**: Internal SoC numbers (used in code)
- **Physical**: Pin numbers on the header (e.g., GPIO4 = Pin 7)
- **WiringPi**: Legacy numbering system (deprecated)

**Always verify**:
```bash
# Find your GPIO number
gpioinfo | grep -i gpio4
# OR
cat /sys/kernel/debug/gpio | grep GPIO4
```

### 5. **Module Parameters**

Make the GPIO pin configurable:
```c
static int gpio_pin = 4;
module_param(gpio_pin, int, 0644);
MODULE_PARM_DESC(gpio_pin, "GPIO pin number for LED");
```

Load with custom pin:
```bash
sudo insmod led.ko gpio_pin=17
```

---

## Troubleshooting

### Problem: `gpio_request()` fails with error -16 (EBUSY)
**Cause**: GPIO already claimed by another driver or sysfs
**Solutions**:
```bash
# Find who claimed it
cat /sys/kernel/debug/gpio | grep GPIO4
# Release from sysfs
echo 4 > /sys/class/gpio/unexport
# Check device tree overlays
sudo vcdbg log msg
```

### Problem: LED doesn't light
**Debugging steps**:
```bash
# Check module loaded
lsmod | grep led
# Check device file exists
ls -l /dev/gpio_led
# Check kernel messages
dmesg | tail -20
# Verify GPIO state
cat /sys/kernel/debug/gpio | grep GPIO4
# Test with sysfs (temporary)
echo 4 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio4/direction
echo 1 > /sys/class/gpio/gpio4/value
```

### Problem: DHT11 read timeouts
**Possible causes**:
- Wrong GPIO number (especially with 575 vs 4)
- Faulty sensor or wiring
- Timing too strict for your Pi model
- Interrupts interfering with busy-wait

**Fixes**:
```c
// Increase timeout values
timeout = 1000;  // Was 300
// Add slight delays
udelay(60);  // Was 50
```

### Problem: Device file not created
**Check kernel version compatibility**:
```bash
uname -r  # Ensure your KDIR matches
# Fix class_create version issues
# Add #define LINUX_VERSION_CODE checks
```

---

## Further Resources

### Official Documentation
- **Linux GPIO Subsystem**: `Documentation/gpio/` in kernel source
- **BCM2835 Datasheet**: Broadcom peripherals specification
- **Device Tree Bindings**: `Documentation/devicetree/bindings/gpio/`

### Tools and Utilities
```bash
# For GPIO debugging
sudo apt install gpiod
gpioinfo  # Show all GPIO lines
gpioset gpiochip0 4=1  # Set GPIO4 high
gpioget gpiochip0 4  # Read GPIO4

# Kernel debugging
sudo apt install linux-tools-generic
trace-cmd record -e gpio_*  # Trace GPIO events
```

### Advanced Topics
1. **Interrupt-Driven GPIO**: Rewrite DHT11 using `request_irq()` for edge detection
2. **Poll Interface**: Add `poll()` support for asynchronous sensor reads
3. **sysfs Attributes**: Expose configuration via `/sys/class/gpio_led/`
4. **Platform Device**: Convert to platform driver with device tree support
5. **GPIO Descriptors**: Migrate to modern `gpiod_*` API for better resource management

### Recommended Reading
- *Linux Device Drivers, 3rd Ed.* - Corbet, Rubini, Kroah-Hartman
- *Mastering Embedded Linux Programming* - Chris Simmonds
- Linux kernel source: `drivers/gpio/gpio-bcm2835.c` for hardware implementation details

---

## Summary

This chapter teaches **three essential skills**:

1. **GPIO Abstraction**: Why and how to use kernel APIs instead of direct register access
2. **Character Device Drivers**: Complete lifecycle from registration to cleanup
3. **Hardware Protocols**: Implementing timing-sensitive communication in kernel space

The **LED driver** provides a foundation for output control, while the **DHT11 driver** demonstrates input handling with complex protocols. Together, they equip you to interface virtually any digital sensor or actuator with Linux.

**Remember**: Always respect the kernel's abstraction layers. Direct register access might seem simpler initially, but proper API usage ensures your code is portable, stable, and maintainable across kernel versions and hardware platforms.