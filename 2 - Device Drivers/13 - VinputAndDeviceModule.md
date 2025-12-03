# Linux Virtual Input Device Drivers & Device Model - Complete Technical Guide

## 📑 Table of Contents

1. [Chapter Overview: Virtual Input & Device Model](#chapter-overview)
2. [The Virtual Input Framework (vinput)](#vinput-framework)
   - [Why Virtual Input Devices?](#why-virtual-input)
   - [Architecture Overview](#vinput-architecture)
   - [Core Data Structures](#core-structures)
3. [Deep Dive: vinput.h - The User API](#vinput-h-deep-dive)
4. [Deep Dive: vinput.c - Framework Implementation](#vinput-c-deep-dive)
   - [Device Registration and Management](#device-management)
   - [sysfs Interface Creation](#sysfs-interface)
   - [Character Device Operations](#char-ops)
   - [Cleanup and Resource Management](#cleanup)
5. [Virtual Keyboard Example (vkbd.c)](#virtual-keyboard)
   - [Key Mapping and Event Codes](#key-mapping)
   - [Event Injection Protocol](#event-protocol)
   - [Implementation Walkthrough](#vkbd-impl)
6. [Linux Device Model: Platform Drivers](#device-model)
   - [Platform Device Concepts](#platform-concepts)
   - [Probe and Remove Lifecycle](#probe-remove)
   - [Power Management Integration](#pm-integration)
7. [API Reference](#api-reference)
   - [vinput API Functions](#vinput-api)
   - [Input Subsystem API](#input-api)
   - [Device Model API](#device-model-api)
8. [Building and Testing](#building-testing)
9. [Critical Concepts and Troubleshooting](#concepts-troubleshooting)
10. [Best Practices and Recommendations](#best-practices)
11. [Further Resources](#resources)

---

## Chapter Overview: Virtual Input & Device Model

This chapter introduces two advanced Linux driver concepts:

1. **Virtual Input Device Framework (vinput)**: A mechanism to create software-generated input events (keyboard, mouse, touchscreen) that appear as real hardware devices to user-space applications. This is useful for testing, automation, remote desktop, accessibility tools, and screen sharing.

2. **Linux Device Model**: A standardized object-oriented framework for managing devices, drivers, and their lifecycle including power management (suspend/resume), hotplug, and sysfs integration.

**Key Learning Objectives:**
- Understand how to create virtual input devices that integrate seamlessly with Linux input subsystem
- Master the vinput framework architecture and plugin model
- Implement sysfs-based dynamic device creation/destruction
- Use the Linux Device Model for standardized driver interfaces
- Implement power management hooks for real hardware drivers

---

## The Virtual Input Framework (vinput)

### Why Virtual Input Devices?

Virtual input devices solve critical problems:

- **Automated Testing**: Simulate user input without physical hardware
- **Remote Desktop**: Inject mouse/keyboard events from network
- **Accessibility**: Convert eye-tracking or voice commands into keyboard events
- **Screen Sharing**: Mirror input events across multiple machines
- **Multi-seat Setup**: Share one keyboard/mouse across multiple sessions
- **Device Emulation**: Test input drivers without physical hardware

Without vinput, you'd have to manually implement the entire input subsystem protocol, handle device registration, manage sysfs nodes, and write char device handlers. The vinput framework provides a **plugin architecture** that abstracts all this complexity.

### Architecture Overview

```
User Space Applications
    ↓ (read /dev/input/eventX)
Linux Input Subsystem
    ↓
Virtual Input Device (struct input_dev)
    ↓
vinput Framework (vinput.c)
    ↓ (calls)
Virtual Device Plugin (vkbd.c, vmouse.c, etc.)
    ↓
User-Space String Protocol (/dev/vinput0)
```

The vinput framework implements the **common infrastructure**:
- Character device driver (`/dev/vinputX`)
- sysfs device creation (`/sys/class/vinput/`)
- Device lifecycle management
- Thread safety (spinlocks)
- Input device registration

Plugins implement **device-specific logic**:
- Keyboard (vkbd.c)
- Mouse
- Touchscreen
- Joystick
- Custom devices

### Core Data Structures

```c
struct vinput {
    long id;                      // Unique device ID
    long devno;                   // Device number
    long last_entry;              // Last injected event (for readback)
    spinlock_t lock;              // Thread-safe access
    void *priv_data;              // Plugin private data
    struct device dev;            // Linux device model object
    struct list_head list;        // Linked list node
    struct input_dev *input;      // Input subsystem device
    struct vinput_device *type;   // Plugin operations
};

struct vinput_ops {
    int (*init)(struct vinput *);    // Initialize input capabilities
    int (*kill)(struct vinput *);    // Cleanup (optional)
    int (*send)(struct vinput *, char *, int);  // Parse user string
    int (*read)(struct vinput *, char *, int);  // Debug readback
};

struct vinput_device {
    char name[16];                // Plugin name (e.g., "vkbd")
    struct list_head list;        // Global plugin list
    struct vinput_ops *ops;       // Operation callbacks
};
```

**Key Design Decisions**:

-  **`priv_data` **: Allows plugins to store per-device state (e.g., key repeat timers)
-  **`spinlock_t lock`** : Protects concurrent access from multiple processes reading/writing `/dev/vinputX`
- **Plugin model**: Clean separation of framework vs. device logic
- **sysfs export/unexport**: Dynamic device creation without module reload

---

## Deep Dive: vinput.h - The User API

The header file defines the contract between framework and plugins:

### 1. **vinput Structure** - Per-Device Instance

```c
struct vinput {
    long id;                      // Minor number (0-31)
    long devno;                   // Combined major/minor
    long last_entry;              // Stores last event for debugging
    spinlock_t lock;              // Critical section protection
    void *priv_data;              // Opaque pointer for plugins
    struct device dev;            // For sysfs and device model
    struct list_head list;        // Links all active vdevices
    struct input_dev *input;      // The actual input device
    struct vinput_device *type;   // Points to plugin ops
};
```

**Why each field exists**:

-  **`id` **: Maps to `/dev/vinput0`, `/dev/vinput1`, etc. Limited to `VINPUT_MINORS` (32) devices
-  **`last_entry` **: Allows `cat /dev/vinput0` to show last injected event (development/debug)
-  **`spinlock_t lock` **: Essential because `input_report_key()` and friends can be called from interrupt context or multiple user processes
-  **`priv_data` **: Plugin might need to store key repeat state, LED status, etc.
-  **`struct device dev` **: Required for `device_unregister()`, sysfs integration, reference counting
-  **`struct list_head list` **: Links into `vinput_vdevices` global list for lookup by ID
-  **`struct input_dev *input` **: The actual device that generates events to user space

### 2. ** vinput_ops Structure ** - Plugin Callbacks

```c
struct vinput_ops {
    int (*init)(struct vinput *);    // REQUIRED: Set up input capabilities
    int (*kill)(struct vinput *);    // OPTIONAL: Cleanup before unregister
    int (*send)(struct vinput *, char *, int);  // REQUIRED: Parse user commands
    int (*read)(struct vinput *, char *, int);  // OPTIONAL: Debug output
};
```

**Callback Semantics**:

** `init()` **: Called during `echo "vkbd" > /sys/class/vinput/export`
- Must call `input_set_capability(vinput->input, EV_KEY, KEY_A)` for each supported key
- Must call `input_register_device(vinput->input)` at the end
- Return 0 on success, negative error code on failure

** `send()` **: Called during `echo "+34" > /dev/vinput0`
- `buff`: Contains string from user (e.g., "+34\n")
- `len`: Length of string (including newline)
- Must parse and validate the string format
- Must call `input_report_key()` or `input_event()` to inject events
- Return number of bytes consumed or negative error

** `read()` **: Called during `cat /dev/vinput0`
- Should format `vinput->last_entry` into human-readable string
- Return string length
- Used for debugging - can be omitted for production devices

** `kill()` **: Called during `echo "0" > /sys/class/vinput/unexport`
- Cleanup `priv_data` (e.g., cancel timers, free memory)
- Optional because many plugins don't need cleanup

### 3. **vinput_device Structure** - Plugin Registration

```c
struct vinput_device {
    char name[16];                // Must be unique (e.g., "vkbd", "vmouse")
    struct list_head list;        // Links into global vinput_devices list
    struct vinput_ops *ops;       // Pointer to plugin's ops structure
};
```

**Registration Process**:
```c
// In plugin's module_init()
static struct vinput_device vkbd_dev = {
    .name = "vkbd",
    .ops = &vkbd_ops,
};

vinput_register(&vkbd_dev);  // Adds to global list
```

The framework uses this to find the plugin when user runs `echo "vkbd" > export`.

---

## Deep Dive: vinput.c - Framework Implementation

### 1. **Global State Management**

```c
static DECLARE_BITMAP(vinput_ids, VINPUT_MINORS);  // Bitmap for minor numbers
static LIST_HEAD(vinput_devices);                  // Registered plugins
static LIST_HEAD(vinput_vdevices);                 // Active instances
static int vinput_dev;                            // Major number from register_chrdev()
static spinlock_t vinput_lock;                    // Protects both lists
static struct class vinput_class;                 // For /sys/class/vinput/
```

**Critical Design**: Two separate lists:
-  **`vinput_devices` **: Plugins (registered once, lives forever)
-  **`vinput_vdevices` **: Device instances (created/destroyed dynamically)

### 2. **Device Allocation: `vinput_alloc_vdevice()` **

```c
static struct vinput *vinput_alloc_vdevice(void)
{
    struct vinput *vinput = kzalloc(sizeof(struct vinput), GFP_KERNEL);
    // GFP_KERNEL: Can sleep, normal allocation
    
    vinput->id = find_first_zero_bit(vinput_ids, VINPUT_MINORS);
    // Finds first unused ID (0-31)
    
    set_bit(vinput->id, vinput_ids);  // Mark as used
    
    list_add(&vinput->list, &vinput_vdevices);  // Add to active list
    
    vinput->input = input_allocate_device();  // Allocate input_dev
    // This is the actual device that will send events
    
    dev_set_name(&vinput->dev, DRIVER_NAME "%lu", vinput->id);
    // Sets name to "vinput0", "vinput1", etc.
}
```

**Allocation Strategy**:
- Uses **bitmap** for O(1) allocation of minor numbers
- **Zero-initializes** all fields (important for security)
- Immediately adds to list to prevent race conditions
- **Never fails after list addition** - must have matching cleanup path

### 3. **sysfs Export Interface: export_store()**

```c
static ssize_t export_store(const struct class *class,
                            const struct class_attribute *attr,
                            const char *buf, size_t len)
{
    // Example: buf = "vkbd\n", len = 5
    
    device = vinput_get_device_by_type(buf);  // Find plugin
    
    vinput = vinput_alloc_vdevice();  // Allocate new instance
    
    vinput->type = device;  // Link plugin ops
    
    device_register(&vinput->dev);  // Creates /sys/class/vinput/vinput0/
    
    vinput_register_vdevice(vinput);  // Calls plugin's init()
}
```

**Flow**:
1. Parse device name from user string
2. Look up plugin in `vinput_devices` list
3. Allocate new vinput instance
4. Register with Linux device model (creates sysfs)
5. Call plugin's `init()` → plugin registers `input_dev`
6. **Result**: `/dev/vinput0` and `/dev/input/eventX` both exist

### 4. **sysfs Unexport Interface: unexport_store()**

```c
static ssize_t unexport_store(const struct class *class,
                              const struct class_attribute *attr,
                              const char *buf, size_t len)
{
    // Example: buf = "0\n", len = 2
    
    kstrtol(buf, 10, &id);  // Parse device ID
    
    vinput = vinput_get_vdevice_by_id(id);  // Find instance
    
    vinput_unregister_vdevice(vinput);  // Calls input_unregister_device()
    
    device_unregister(&vinput->dev);  // Triggers vinput_release_dev()
}
```

**Cleanup Flow**:
1. User provides device ID (not name)
2. Look up in `vinput_vdevices` list
3. Call `input_unregister_device()` → removes `/dev/input/eventX`
4. Call `device_unregister()` → triggers release callback
5.  **`vinput_release_dev()` ** frees memory

### 5. ** Character Device Operations **

```c
static const struct file_operations vinput_fops = {
    .owner = THIS_MODULE,
    .open = vinput_open,
    .release = vinput_release,
    .read = vinput_read,
    .write = vinput_write,
};

static int vinput_open(struct inode *inode, struct file *file)
{
    vinput = vinput_get_vdevice_by_id(iminor(inode));  // Get from minor number
    file->private_data = vinput;  // Store for other ops
}

static ssize_t vinput_write(struct file *file, const char __user *buffer,
                            size_t count, loff_t *offset)
{
    struct vinput *vinput = file->private_data;
    
    raw_copy_from_user(buff, buffer, count);  // Copy user string
    
    return vinput->type->ops->send(vinput, buff, count);  // Call plugin
}
```

**Key Points**:
- Uses **minor number** as device ID (clean mapping)
-  **`private_data` ** is standard pattern for storing per-open state
-  **`raw_copy_from_user()`** : Variant that doesn't verify access permissions (already done by VFS)
- **No locking in write()**: Plugin's `send()` must handle concurrency with spinlocks

---

## Virtual Keyboard Example (vkbd.c)

### Key Mapping and Event Codes

```c
#define VINPUT_KBD "vkbd"
#define VINPUT_RELEASE 0
#define VINPUT_PRESS 1

static unsigned short vkeymap[KEY_MAX];  // Array of Linux key codes

// In init():
for (i = 0; i < KEY_MAX; i++)
    vkeymap[i] = i;  // Identity mapping: code 34 = KEY_G
```

**Linux Input Event Codes** (from `include/linux/input.h`):
- `EV_KEY`: Key press/release events
- `KEY_G`: 34 (lowercase 'g')
- `KEY_ENTER`: 28
- `KEY_LEFTSHIFT`: 42
- `KEY_A`: 30

**Why `vkeymap` exists**: Allows plugins to remap codes (e.g., QWERTY → AZERTY). Keyboard uses identity mapping; mouse could map relative motion values.

### Event Injection Protocol

User-space string format:
```
+34\n   → Press key 34 (KEY_G)
-34\n   → Release key 34
```

**Protocol Design Decisions**:
-  **`+` prefix **: Unambiguously identifies press
-  **`-` prefix **: Could be part of negative number or prefix
- ** No separator needed**: Simple parse with `buff[0]` check
- ** `\n` termination **: Makes `echo` command work naturally

### Implementation Walkthrough

```c
static int vinput_vkbd_send(struct vinput *vinput, char *buff, int len)
{
    long key = 0;
    short type = VINPUT_PRESS;
    
    // Parse press/release
    if (buff[0] == '+')
        ret = kstrtol(buff + 1, 10, &key);  // Parse after '+'
    else
        ret = kstrtol(buff, 10, &key);       // Parse whole string (could be -34)
    
    // Extract sign
    if (key < 0) {
        type = VINPUT_RELEASE;
        key = -key;  // Make positive for input_report_key()
    }
    
    // Inject event
    input_report_key(vinput->input, key, type);  // Example: input_report_key(input, 34, 1)
    input_sync(vinput->input);  // Flush event to user-space
}
```

**Critical Call: `input_report_key()`**
- First parameter: `vinput->input` (the registered input device)
- Second parameter: Linux key code (34 = KEY_G)
- Third parameter: 1=press, 0=release
- This function **queues the event** in the input subsystem

**Why `input_sync()` is mandatory **:
- Without it, events are buffered but not delivered
- Sync flushes the buffer to all listening applications
- Some drivers auto-sync on every event; explicit sync is safer

### Usage Examples

```bash
# Load the framework module first
sudo insmod vinput.ko

# Load the keyboard plugin
sudo insmod vkbd.ko

# Create a virtual keyboard (echo plugin name to export)
echo "vkbd" | sudo tee /sys/class/vinput/export
# dmesg should show: vinput: Registered virtual input vkbd 0

# Verify device created
ls -l /dev/vinput0
ls -l /dev/input/event*  # Should show new input device

# Simulate pressing 'g' (KEY_G = 34)
echo "+34" | sudo tee /dev/vinput0
# dmesg: Event VINPUT_PRESS code 34

# Check what X11 sees (install evtest first)
sudo evtest /dev/input/eventX
# Should show: Event: time 123456.789, -------------- EV_SYN ------------
# Event: time 123456.789, type 1 (EV_KEY), code 34 (KEY_G), value 1

# Simulate releasing 'g'
echo "-34" | sudo tee /dev/vinput0

# Read last event (debugging)
cat /dev/vinput0
# Output: +34

# Remove device (echo ID to unexport)
echo "0" | sudo tee /sys/class/vinput/unexport
```

**Testing with X11/Wayland **:
The virtual keyboard will appear as a real keyboard. Any application will receive the events. You can type into a terminal by injecting the right key codes in sequence.

** Key Sequence Example **: Type "hi"
```bash
echo "+35" > /dev/vinput0  # KEY_H = 35 (press)
echo "-35" > /dev/vinput0  # KEY_H (release)
echo "+23" > /dev/vinput0  # KEY_I = 23 (press)
echo "-23" > /dev/vinput0  # KEY_I (release)
```

---

## Linux Device Model: Platform Drivers

### Platform Device Concepts

The **platform device model** is for devices that are **not discoverable by bus** (I2C, PCI, USB). These are typically:
- On-SoC devices (GPIO controllers, timers, DMA engines)
- Board-specific devices (custom FPGA logic)
- Virtual devices (like our vinput framework)

**Key Components **:
1.  ** Platform Device **: Describes the hardware (device tree or platform code)
2. ** Platform Driver **: Handles the device (probe, remove, suspend, resume)
3. ** Platform Data **: Configuration passed from board code to driver

### Probe and Remove Lifecycle

```c
static int devicemodel_probe(struct platform_device *pdev)
{
    // pdev->dev.platform_data contains board-specific configuration
    struct devicemodel_data *pd = dev->dev.platform_data;
    
    // This runs when kernel matches driver to device
    // - Allocate resources
    // - Initialize hardware
    // - Register with subsystems
    
    return 0;  // Success
}

static void devicemodel_remove(struct platform_device *dev)
{
    // This runs when device is removed or module unloaded
    // - Unregister from subsystems
    // - Free resources
    
    // In Linux 6.11+, remove returns void (can't fail)
    // Before 6.11, returned int (could fail, but rarely did)
}
```

**How Matching Works **:
- Kernel compares `platform_driver.driver.name` with device name
- For device tree: `compatible = "devicemodel_example"` string
- When match found, calls `.probe()`
- On removal/hot-unplug, calls `.remove()`

### Power Management Integration

```c
static int devicemodel_suspend(struct device *dev)
{
    // Called when system suspends (sleep, hibernate)
    // - Save device state to RAM
    // - Put hardware in low-power mode
    // - Disable clocks/interrupts
    return 0;
}

static int devicemodel_resume(struct device *dev)
{
    // Called when system resumes
    // - Restore device state
    // - Re-enable clocks/interrupts
    // - Re-initialize hardware
    return 0;
}

static const struct dev_pm_ops devicemodel_pm_ops = {
    .suspend = devicemodel_suspend,
    .resume = devicemodel_resume,
    .poweroff = devicemodel_suspend,  // Same as suspend for most devices
    .freeze = devicemodel_suspend,    // For hibernation
    .thaw = devicemodel_resume,       // For hibernation resume
    .restore = devicemodel_resume,    // For restore from disk
};
```

**PM Callback Context **:
- Runs in process context (can sleep)
- Must be fast (< 100ms) to not delay suspend/resume
- For devices that can't wake system, use `pm_runtime` instead

### Modern vs Legacy Remove Handler

** Linux 6.11 Breaking Change **:
```c
// Old style (pre-6.11)
static int devicemodel_remove(struct platform_device *dev) 
{
    return 0;  // Could return error
}

// New style (6.11+)
static void devicemodel_remove(struct platform_device *dev)
{
    // Cannot fail - must always succeed
}
```

**Rationale**: In practice, remove almost never failed. Changing to `void` simplifies error handling and makes it clear that failure is not an option.

---

## API Reference

### vinput API Functions

#### `int vinput_register(struct vinput_device *dev)`
- **Purpose**: Register a new virtual input device type (plugin)
- **Parameters**: `dev` - Filled `vinput_device` structure
- **Returns**: 0 on success, negative error code
- **Call from**: Plugin's `module_init()`
- **Example**:
```c
static struct vinput_device vkbd_dev = {
    .name = "vkbd",
    .ops = &vkbd_ops,
};

vinput_register(&vkbd_dev);
```

#### `void vinput_unregister(struct vinput_device *dev)`
- **Purpose**: Unregister a virtual input device type
- **Parameters**: `dev` - Same pointer passed to `vinput_register()`
- **Returns**: Nothing
- **Call from**: Plugin's `module_exit()`
- **Effect**: Removes from global list and **destroys all active instances**

### Input Subsystem API

#### `struct input_dev *input_allocate_device(void)`
- **Purpose**: Allocate a new input device structure
- **Returns**: Pointer to zero-initialized `input_dev` or `NULL`
- **Must call**: Before registering any input device

#### `void input_set_capability(struct input_dev *dev, unsigned int type, unsigned int code)`
- **Purpose**: Declare that device can generate specific event type/code
- **Parameters**:
  - `dev`: Input device structure
  - `type`: Event type (`EV_KEY`, `EV_REL`, `EV_ABS`, etc.)
  - `code`: Event code (`KEY_A`, `REL_X`, `ABS_X`, etc.)
- **Example**:
```c
// Declare keyboard can press keys A-Z
for (i = KEY_A; i <= KEY_Z; i++)
    input_set_capability(input, EV_KEY, i);
```

#### `void input_report_key(struct input_dev *dev, unsigned int code, int value)`
- **Purpose**: Inject a key press/release event
- **Parameters**:
  - `dev`: Input device
  - `code`: Key code (e.g., `KEY_G`)
  - `value`: 1=press, 0=release, 2=autorepeat
- **Context**: Can be called from interrupt or process context
- **Note**: Events are queued until `input_sync()`

#### `void input_sync(struct input_dev *dev)`
- **Purpose**: Flush all pending events to user-space listeners
- **Must call**: After reporting one or more events
- **Effect**: Generates `EV_SYN` event with `SYN_REPORT` code

### Device Model API

#### `int platform_driver_register(struct platform_driver *driver)`
- **Purpose**: Register a platform driver with the kernel
- **Parameters**: `driver` - Filled `platform_driver` structure
- **Returns**: 0 on success, negative error code
- **Matching**: Driver binds to devices with matching `driver.name`

#### `void platform_driver_unregister(struct platform_driver *driver)`
- **Purpose**: Unregister platform driver
- **Parameters**: `driver` - Same pointer passed to register
- **Effect**: Calls `.remove()` on all bound devices

#### `int device_register(struct device *dev)`
- **Purpose**: Register a device with device model (creates sysfs entries)
- **Parameters**: `dev` - Filled device structure (parent, class, release callback)
- **Returns**: 0 on success
- **vinput usage**: Creates `/sys/class/vinput/vinput0/` when exporting

---

## Building and Testing

### Makefile for vinput Framework

```makefile
# Makefile
obj-m := vinput.o vkbd.o

# Or build as separate modules
# obj-m += vinput.o
# obj-m += vkbd.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

**Bonus - Building Both Modules**:
```bash
# vinput.ko (framework)
make -C /lib/modules/$(uname -r)/build M=$PWD vinput.ko

# vkbd.ko (plugin)
make -C /lib/modules/$(uname -r)/build M=$PWD vkbd.ko
```

### Complete Test Script

```bash
#!/bin/bash
# test_vinput.sh

set -e

echo "Loading vinput framework..."
sudo insmod vinput.ko

echo "Loading virtual keyboard..."
sudo insmod vkbd.ko

echo "Available virtual device types:"
ls /sys/class/vinput/

echo "Creating virtual keyboard..."
echo "vkbd" | sudo tee /sys/class/vinput/export

echo "Listing created devices:"
ls /sys/class/vinput/vinput0/

echo "Finding input event device..."
EVENT_DEV=$(grep -l "vkbd" /sys/class/input/event*/device/name | head -1 | sed 's|/device/name||')
echo "Virtual keyboard is at: $EVENT_DEV"

echo "Testing with evtest (press Ctrl+C to stop)..."
sudo evtest $EVENT_DEV &
EVTEST_PID=$!

sleep 1
echo "Injecting 'g' press..."
echo "+34" | sudo tee /dev/vinput0

sleep 0.1
echo "Injecting 'g' release..."
echo "-34" | sudo tee /dev/vinput0

sleep 2
kill $EVTEST_PID

echo "Reading last event:"
cat /dev/vinput0

echo "Destroying virtual keyboard..."
echo "0" | sudo tee /sys/class/vinput/unexport

echo "Unloading modules..."
sudo rmmod vkbd
sudo rmmod vinput

echo "Test complete!"
```

---

## Critical Concepts and Troubleshooting

### 1. **Understanding Input Event Types and Codes**

**Event Types** (`EV_*`):
- `EV_KEY`: Key press/release (keyboard, buttons)
- `EV_REL`: Relative movement (mouse)
- `EV_ABS`: Absolute position (touchscreen, joystick)
- `EV_SYN`: Synchronization (always use with `input_sync()`)
- `EV_LED`: LED state (caps lock, num lock)
- `EV_MSC`: Miscellaneous

**Key Codes** (`KEY_*`):
- Range from 0 to `KEY_MAX` (currently ~0x2FF)
- Defined in `include/linux/input-event-codes.h`
- **Important**: Codes are **not ASCII values**. `KEY_A` = 30, not 97.

### 2. **Spinlock Usage and Deadlock Prevention**

```c
// CORRECT: Lock around shared data access
spin_lock(&vinput->lock);
vinput->last_entry = key;
spin_unlock(&vinput->lock);

// WRONG: Calling input_report_key() with lock held
spin_lock(&vinput->lock);
input_report_key(vinput->input, key, type);  // May sleep! DEADLOCK!
spin_unlock(&vinput->lock);
```

**Rule**: Spinlocks protect **your data structures**, not the input subsystem. `input_report_key()` can sleep internally, so **never call it with spinlock held**.

### 3. **Error: "No such virtual input device" on export**

**Cause**: Plugin not registered or name mismatch
**Debug**:
```bash
# Check loaded modules
lsmod | grep vkbd

# Check if plugin registered
dmesg | grep "vinput: registered new virtual input device"

# List available plugins (not directly exposed, but can hack)
# Add debug printk in vinput_get_device_by_type()
```

**Solution**:
- Ensure `vkbd.ko` is loaded **after** `vinput.ko`
- Verify plugin name matches exactly: `"vkbd"` not `"vkbd\n"`

### 4. **Error: "Unable to register vinput class"**

**Cause**: Class already exists or sysfs permission issue
**Debug**:
```bash
# Check if class already exists
ls /sys/class/vinput/
```

**Solution**:
- Unload old version: `sudo rmmod vinput`
- Check for kernel module in use: `lsmod | grep vinput`

### 5. **Events Not Appearing in X11/Wayland**

**Diagnosis**:
```bash
# Check kernel receives events
sudo evtest /dev/input/eventX

# If evtest works but X11 doesn't, check Xorg logs
grep -i input /var/log/Xorg.0.log

# Modern systems use libinput
sudo libinput debug-events --device /dev/input/eventX
```

**Solution**:
- X11/Wayland may need `ENV{ID_INPUT_KEYBOARD}="1"` udev property
- Input device needs `EV_KEY` capabilities set correctly
- May need to set `input->id.bustype = BUS_USB` to trick X11

### 6. **Device Not Created After Export**

**Check dmesg**:
```bash
dmesg | tail
# Look for: "vinput: Cannot allocate char dev region" → Major number conflict
# Look for: "vinput: Cannot allocate vinput input device" → Memory issue
# Look for: "vkbd: Event VINPUT_PRESS code 34" → Success!
```

**Fix**:
```bash
# Check allocated major number
cat /proc/devices | grep vinput
# Should show: 250 vinput (or similar)

# If missing, register_chrdev failed
```

---

## Best Practices and Recommendations

### 1. **Always Use `input_sync()`**

```c
// WRONG: Events may be delayed or dropped
input_report_key(input, KEY_A, 1);
input_report_key(input, KEY_A, 0);

// CORRECT: Sync after logical event group
input_report_key(input, KEY_A, 1);
input_sync(input);  // Flush press
input_report_key(input, KEY_A, 0);
input_sync(input);  // Flush release
```

**Best**: Sync after each logical action (key press, mouse movement, touch).

### 2. **Validate User Input Rigorously**

```c
static int vinput_vkbd_send(struct vinput *vinput, char *buff, int len)
{
    long key;
    
    // Check length
    if (len < 2 || len > 10)  // "+123\n" is 6 chars
        return -EINVAL;
    
    // Check format
    if (buff[0] != '+' && buff[0] != '-')
        return -EINVAL;
    
    // Parse with error checking
    if (kstrtol(buff + 1, 10, &key) < 0)
        return -EINVAL;
    
    // Check bounds
    if (abs(key) >= KEY_MAX)
        return -EINVAL;
    
    // Check if key is actually supported
    if (!test_bit(key, vinput->input->keybit))
        return -EINVAL;
    
    // Now safe to inject
    input_report_key(vinput->input, abs(key), key > 0 ? 1 : 0);
    input_sync(vinput->input);
    
    return len;
}
```

**Security**: Malicious users could crash kernel by injecting invalid key codes.

### 3. **Use Device Tree for Real Hardware**

For **real** input devices (not virtual), avoid `platform_data`:

```c
// In device tree (board.dts)
virtual_keyboard: vkbd {
    compatible = "vinput,kbd";
    keymap = <30 31 32>;  // Supported keys
    repeat-delay = <250>;
    repeat-rate = <33>;
};

// In driver
static int vinput_vkbd_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    
    // Read from device tree
    of_property_read_u32_array(np, "keymap", keymap, size);
    of_property_read_u32(np, "repeat-delay", &delay);
    
    // Register with framework
    return vinput_register(&vkbd_dev);
}
```

**Benefits**:
- No hardcoded platform data
- Board-specific configuration in device tree
- Kernel can probe/initialize automatically

### 4. **Handle Concurrency in Plugins**

If plugin needs timers or workqueues:

```c
struct vkbd_priv {
    struct timer_list repeat_timer;
    struct vinput *vinput;
    int current_key;
};

static int vinput_vkbd_send(struct vinput *vinput, char *buff, int len)
{
    struct vkbd_priv *priv = vinput->priv_data;
    
    spin_lock(&priv->lock);  // Plugin's own lock
    
    // Cancel any existing timer
    del_timer_sync(&priv->repeat_timer);
    
    // Start new timer for key repeat
    priv->current_key = key;
    mod_timer(&priv->repeat_timer, jiffies + msecs_to_jiffies(500));
    
    spin_unlock(&priv->lock);
    
    input_report_key(vinput->input, abs(key), 1);
    input_sync(vinput->input);
}
```

### 5. **Add Module Parameters for Flexibility**

```c
static int debug_level = 1;
module_param(debug_level, int, 0644);
MODULE_PARM_DESC(debug_level, "Enable verbose logging (0=off, 1=on, 2=trace)");

// In code:
if (debug_level > 1)
    dev_dbg(&vinput->dev, "Injecting key %ld\n", key);
```

### 6. **Document the String Protocol**

```c
// In vkbd.c header comment
/**
 * vkbd Protocol:
 * Format: [+-]<keycode>\n
 *   +: Key press
 *   -: Key release (or negative number)
 *   keycode: Linux input event code (0-KEY_MAX)
 * Example:
 *   "+30\n" - Press 'A'
 *   "-30\n" - Release 'A'
 *   "30\n"  - Press 'A' (legacy)
 *   "-30\n" - Release 'A' (negative number)
 */
```

**User-facing documentation**: Helps users write scripts to control the device.

---

## Further Resources

### Official Documentation
- **Linux Input Subsystem**: `Documentation/input/` in kernel source
- **Input Event Codes**: `Documentation/input/event-codes.rst`
- **Device Model**: `Documentation/driver-api/driver-model/`
- **Platform Devices**: `Documentation/driver-api/driver-model/platform.rst`

### Kernel Source Files to Study
- **`drivers/input/input.c`**: Core input subsystem
- **`drivers/input/evdev.c`**: Evdev handler (creates `/dev/input/eventX`)
- **`drivers/base/platform.c`**: Platform device/driver core
- **`drivers/base/class.c`**: Class registration (creates `/sys/class/`)

### Tools for Testing
```bash
# evtest: Monitor input events
sudo apt install evtest
sudo evtest /dev/input/eventX

# evemu-record/evemu-play: Record and replay events
sudo apt install evemu-tools
sudo evemu-record > events.txt
sudo evemu-play /dev/input/eventX < events.txt

# udevadm: Debug device properties
sudo udevadm info -a /dev/input/eventX
```

### Example Virtual Devices to Implement

As exercises:
1. **Virtual Mouse**: `vmouse.c` with relative movement (`EV_REL`, `REL_X`, `REL_Y`)
2. **Virtual Touchscreen**: with absolute coordinates (`EV_ABS`, `ABS_X`, `ABS_Y`)
3. **Virtual Joystick**: with axes and buttons
4. **Virtual Multitouch**: with `EV_ABS` and `ABS_MT_*` codes

**Mouse Protocol Example**:
```
echo "X10" > /dev/vinput0  # Move right 10 units
echo "Y-5" > /dev/vinput0  # Move up 5 units
echo "B1D" > /dev/vinput0  # Button 1 down
```

---

## Summary

**vinput framework** provides a **plugin-based architecture** for creating virtual input devices:
- **Framework** (`vinput.c`): Manages device lifecycle, sysfs, char dev
- **Plugins** (`vkbd.c`): Implement specific device types (keyboard, mouse, etc.)
- **Protocol**: Simple string-based injection via `/dev/vinputX`
- **Integration**: Appears as real device in `/dev/input/eventX`

**Linux Device Model** standardizes driver interfaces:
- **Platform drivers**: For non-discoverable devices
- **Probe/Remove**: Lifecycle management
- **Power Management**: Suspend/resume callbacks
- **Standardization**: Consistent interface across all drivers

**Key Takeaways**:
1. Use vinput for **testing and emulation** - don't abuse for production unless necessary
2. Always **validate user input** - kernel crashes are unacceptable
3. Use **device tree** for configuration on real hardware
4. **Threaded IRQs** are better for real input devices (not virtual)
5. **Document your protocol** - future you will thank you