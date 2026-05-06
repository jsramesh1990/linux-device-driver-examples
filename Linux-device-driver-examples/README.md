
# Linux Device Driver Examples: The Complete Guide

## 1. Introduction & Definitions
A **Linux Device Driver** is a software component that allows the operating system (kernel) to interact with hardware devices. It provides an abstraction layer so software can control hardware without knowing exact electrical or register-level details .

### Core Terminology
*   **Kernel Space**: The privileged area where the kernel and drivers run. Has complete access to hardware.
*   **User Space**: Where applications (like your browser or terminal) run. Restricted for security.
*   **Device File (Character Device)**: A special file in `/dev/` (e.g., `/dev/ami0`) that allows user-space apps to communicate with the driver using standard read/write calls .
*   **Sysfs**: A virtual filesystem (usually mounted at `/sys/`) that exports kernel objects (devices, drivers) to user space as files. Preferred over `ioctl` for simple interactions .
*   **Device Tree (DTB/DTS)**: A data structure used by ARM, PowerPC, and RISC-V architectures to describe hardware. It solves the problem of hard-coding pin addresses. Drivers scan the Device Tree to know which hardware to initialize .

## 2. Why Do We Use Drivers?
Without drivers, the operating system sees the hardware as a black box.
1.  **Hardware Abstraction**: Allows an application to write to a USB drive without knowing if the USB controller is made by Intel or ASMedia .
2.  **Separation of Concerns**: Keeps stable application code separated from volatile hardware logic.
3.  **Standardized Interfaces**: User space uses `open()`, `read()`, `write()` regardless of the physical hardware.
4.  **Security & Management**: The kernel manages resource sharing, preventing one app from crashing the whole system by messing with hardware registers.

## 3. How It Works: The Architecture
The Linux Driver model is heavily object-oriented (implemented in C) and revolves around three main components: **Bus**, **Device**, and **Driver** .

```text
[User Space Application] -> open/read/write -> [Device File (/dev/xxx)]
                                                       |
                                                       v
[Kernel Space]  +---------------------------------------------+
                |           Core Kernel / VFS                  |
                +---------------------------------------------+
                                | (Match)
+-------------------------------+-------------------------------+
|   Device (Hardware)           |    Driver (Software Logic)    |
| - Physical Address            |    - probe()                  |
| - IRQ Number                  |    - open()                   |
| - Vendor ID                   |    - read()/write()           |
+-------------------------------+-------------------------------+
                |
                v
[ Physical Hardware ]
```

### The Matching Process
When the kernel boots or a new device is plugged in, the bus (e.g., PCI, USB, I2C) signals the presence of a device.
1.  **Bus scans** for devices.
2.  **Comparison**: The kernel compares the device's ID (e.g., `of_match_table` for Device Tree, or `id_table` for PCI) with the list of registered drivers.
3.  **Probe**: If a match is found, the kernel calls the driver's `probe` function. This is the "birth" of the driver instance .

## 4. Syntax & Core Structures
In Linux, a driver is defined by filling out specific structures. Here is the anatomy of a typical driver.

### The `device_driver` Structure
This is the base skeleton. Most specific bus drivers (like `pci_driver` or `i2c_driver`) wrap this .

```c
// Kernel view of a driver (simplified)
struct device_driver {
    const char *name;                // Name of the driver
    struct bus_type *bus;            // Which bus it belongs to (platform, i2c, spi...)
    struct module *owner;            // Usually THIS_MODULE
    const struct of_device_id *of_match_table; // Device Tree matching table
    int (*probe) (struct device *dev);  // Called when device is found
    void (*remove) (struct device *dev); // Called when device is removed
};
```

### 1. Includes
```c
#include <linux/module.h>      // Core module macros (module_init, module_exit)
#include <linux/kernel.h>      // printk() (kernel's printf)
#include <linux/fs.h>          // File operations structure
#include <linux/device.h>      // Device classes
#include <linux/of.h>          // Device Tree helpers
#include <linux/gpio/consumer.h> // GPIO subsystem
```

### 2. The "Hello World" Boilerplate
Every driver needs an initialization and exit function.

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init my_driver_init(void) {
    printk(KERN_INFO "Hello: Driver loaded!\n");
    return 0; // 0 means success
}

static void __exit my_driver_exit(void) {
    printk(KERN_INFO "Goodbye: Driver unloaded!\n");
}

module_init(my_driver_init);
module_exit(my_driver_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A simple example driver");
```

### 3. File Operations (User Space Interaction)
To let user space interact, you define a `file_operations` struct .

```c
static int dev_open(struct inode *inodep, struct file *filep) {
    printk(KERN_INFO "Device opened\n");
    return 0;
}

static ssize_t dev_read(struct file *filep, char __user *buffer, 
                        size_t len, loff_t *offset) {
    // Copy data from kernel space to user space
    // copy_to_user(buffer, kernel_data, len);
    return len;
}

// This is the lookup table for syscalls
static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = dev_open,
    .read = dev_read,
};

// In init function:
// register_chrdev(MAJOR_NUM, "device_name", &fops);
```

## 5. How to Make Changes & Debug

### Workflow: Modify -> Compile -> Test 

1.  **Locate the Code**:
    - In-kernel drivers: `/usr/src/linux/drivers/`
    - Out-of-tree drivers: Your working directory.

2.  **The Compilation (Makefile)**:
    You cannot compile a driver with `gcc`. You must use the kernel build system.
    ```makefile
    obj-m += mydriver.o          # Turns mydriver.c into mydriver.ko
    mydriver-y := main.o utils.o # If driver has multiple files

    KDIR := /lib/modules/$(shell uname -r)/build
    PWD := $(shell pwd)

    all:
        $(MAKE) -C $(KDIR) M=$(PWD) modules
    clean:
        $(MAKE) -C $(KDIR) M=$(PWD) clean
    ```

3.  **Build & Insert**:
    ```bash
    make                    # Compiles to .ko (kernel object)
    sudo insmod mydriver.ko # Insert the driver
    lsmod | grep mydriver   # Check if loaded
    sudo rmmod mydriver     # Remove the driver
    ```

### Debugging Techniques 
- **printk**: The simplest. Use log levels: `printk(KERN_ERR "Value is %d\n", val);`. View with `dmesg`.
- **ftrace**: Built-in tracer to see function calls.
    ```bash
    echo function > /sys/kernel/debug/tracing/current_tracer
    echo 1 > /sys/kernel/debug/tracing/tracing_on
    # Run your test
    cat /sys/kernel/debug/tracing/trace
    ```
- **Devicetree Overlays**: If you are modifying hardware definitions (like GPIO pins), you might recompile the device tree:
    ```bash
    dtc -I dts -O dtb -o myboard.dtb myboard.dts
    ```

## 6. Supported Platforms
Linux drivers run on almost any CPU architecture. However, the **Device Tree** is primarily used in **ARM**, **PowerPC**, and **RISC-V**. x86 (Intel/AMD PCs) prefers ACPI (BIOS/UEFI) .

| **Platform** | **Hardware Description** | **Typical Use** |
| :--- | :--- | :--- |
| **x86 / x86_64** | ACPI / PCI Enumeration | Servers, Desktops, Laptops |
| **ARM / ARM64** | Device Tree (DTB) | Embedded Systems, Raspberry Pi, Android Phones |
| **RISC-V** | Device Tree (DTB) | Emerging embedded & server platforms |

### Driver Types
1.  **Character Drivers** (Keyboard, UART, Sensors) - Byte stream access.
2.  **Block Drivers** (SSD, HDD) - Block (512 byte) access.
3.  **Network Drivers** (Ethernet, WiFi) - Socket interface.

## 7. How to Use Subsystems (GPIO, I2C)

Don't write directly to registers. Use the Kernel APIs .

### GPIO Example
```c
#include <linux/gpio/consumer.h>
#include <linux/of.h>

// Inside probe() function:
struct device *dev = &pdev->dev;
struct gpio_desc *reset_gpio;

// Get GPIO from Device Tree (e.g., reset-gpios = <&gpio 20 0>)
reset_gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW);
if (IS_ERR(reset_gpio)) {
    dev_err(dev, "Failed to get reset GPIO\n");
    return PTR_ERR(reset_gpio);
}

// Set high
gpiod_set_value(reset_gpio, 1);
msleep(20);
// Set low
gpiod_set_value(reset_gpio, 0);
```

### I2C Example
```c
#include <linux/i2c.h>

static int my_i2c_probe(struct i2c_client *client, const struct i2c_device_id *id) {
    // Read 1 byte from register 0x10
    char reg = 0x10;
    int ret;
    
    ret = i2c_master_send(client, &reg, 1);
    if (ret < 0) 
        dev_err(&client->dev, "I2C error: %d\n", ret);
    return 0;
}

// Match table
static const struct of_device_id my_i2c_ids[] = {
    { .compatible = "myvendor,mydevice", },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, my_i2c_ids);
```

## 8. Interview Preparation (Questions & Answers)

Here are specific questions asked in technical interviews for Embedded Linux roles .

### Q1: What is the difference between `insmod` and `modprobe`?
**Answer:** `insmod` inserts a single `.ko` file but doesn't handle dependencies. `modprobe` loads the module plus *all* its dependencies by parsing `modules.dep`. `modprobe` is preferred for production.

### Q2: Can I do `malloc` or `printf` inside a driver?
**Answer:** No.
- `printf` -> use `printk` (kernel version).
- `malloc` -> use `kmalloc` (kernel memory allocator). `kmalloc` uses flags like `GFP_KERNEL` (can sleep) or `GFP_ATOMIC` (cannot sleep, e.g., in interrupt context).

### Q3: Walk me through the `probe()` function. Why is it needed?
**Answer:** The `probe()` function is the driver's starting point. It is called when the kernel matches a physical device to our driver. Inside probe, we:
1. Allocate device structures (`devm_kzalloc`).
2. Get hardware resources (IRQ numbers, memory regions).
3. Initialize hardware (enable clocks, configure registers).
4. Create device files (`class_create`, `device_create`).
5. Set up interrupt handlers (`request_irq`).

### Q4: What happens when a user calls `read()` on a device file?
**Answer:**
1.  Syscall handler saves user process context and switches to kernel mode.
2.  Kernel VFS layer routes the call to the driver's `file_operations.read` function.
3.  Driver prepares data (reads from hardware buffer).
4.  Driver uses `copy_to_user()` to transfer data from Kernel space to User space.
5.  Control returns to user process.

### Q5: Why can't interrupts sleep? (Tricky Question)
**Answer:** Interrupt handlers run in an atomic context. They do not have a process structure (`task_struct`) backing them. Since they have no process context, there is nothing to "sleep" on or schedule back in. Sleeping in an interrupt would freeze the kernel. Bottom halves (Tasklets/Workqueues) should be used for heavy lifting .

### Q6: How does Device Tree work?
**Answer:** It replaces hard-coded `#define` macros for board-specific addresses. The bootloader loads a binary DTB (Device Tree Blob) into memory. The kernel parses it. In the driver, we use `of_property_read_u32()` or `devm_gpiod_get()` to fetch parameters dynamically. This allows one kernel binary to support multiple different SoCs and boards .

## 9. Practical Checklist for Testing

Before submitting code for a **linux-device-driver-examples** project:

1.  **Check Memory Leaks**: Unloading the driver (`rmmod`) should free all memory. Use `kmemleak`.
2.  **Concurrency**: Can two apps open the device simultaneously? Use Mutexes or Spinlocks to protect global data.
3.  **Error Handling**: If `probe()` fails, all allocated resources must be freed (rolled back).
4.  **Coding Style**: Run `scripts/checkpatch.pl` on your patch.

## 10. References & Further Reading
- **Linux Kernel Module Programming Guide** (LDD3 - though old, concepts are gold).
- **Documentation/driver-api/** in the kernel source tree.
- **Device Tree Usage** (devicetree.org).

---
*Generated for the linux-device-driver-examples repository.*
