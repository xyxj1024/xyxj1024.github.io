---
layout:     post
title:      "Configure and Cross-Compile a Linux Kernel for a Raspberry Pi"
category:   "Linux System Programming"
tags:       linux-kernel raspberry-pi cse522-assignment
permalink:  /blog/configure-compile-linux-kernel
---

Studio 1 and 2 of [CSE 522S: "Advanced Operating Systems" at Washington University in St. Louis](https://classes.engineering.wustl.edu/cse522s). I am using a Raspberry Pi 3 Model B Plus for the course, which has a quad-core Arm&reg; Cortex-A53 (ARMv8) CPU.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Download the Linux Kernel Source Code

On a Linux lab machine besides my Raspberry Pi, I created a folder called `linux_source` in which to keep all of my source code and build files organized. Inside this new folder, issue the following commands:

```bash
wget https://github.com/raspberrypi/linux/archive/raspberrypi-kernel_1.20210527-1.tar.gz

tar -xzf raspberrypi-kernel_1.20210527-1.tar.gz
```

This has the effect of downloading a specific version of the Raspberry Pi Linux distribution, which is the version that the course was developed with. After the files finish unpacking, I renamed the new directory as `linux` with the `mv` command and deleted the `.tar.gz` file to save space.

We can check the Linux kernel version:

```sh
[x.xingjian@linuxlab002 linux]$ make kernelversion
5.10.17
```

The first several lines of Makefile look like:

```makefile
# SPDX-License-Identifier: GPL-2.0
VERSION = 5
PATCHLEVEL = 10
SUBLEVEL = 17
EXTRAVERSION =
NAME = Kleptomaniac Octopus
```

## Configure the Linux Kernel

Add the cross-compiler to `PATH` variable:

```bash
module add arm-rpi
```

Add an updated C compiler to `PATH` variable

```bash
module add gcc-8.3.0
```

One can add the two commands above as individual lines at the end of the file `~/.bashrc`, and then in order to ensure the command is executed when logged in via SSH, add code like the following to the file `~/.bash_profile`:

```bash
if [ -f ~/.bashrc ]; then
. ~/.bashrc;
fi
```

Issue the following command:

```bash
make -j8 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig
```

If Raspberry Pi 4 or 4B is used, use the following command instead:

```bash
make -j8 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
```

Then, issue the command:

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```

to get a kernel configuration menu.

### Add your own unique identifier to the kernel

Navigate to `General Setup`, select `Local Version`. The local version string we specify here will be appended to the output of the `uname` command. The default for Raspberry Pi 3B+ is `-v7`. I renamed it to `-v7-x.xingjian`.

### Change the kernel's preemption model

I could not select the `Preemption Model` option, so I manually modified the `.config` file by commenting out `CONFIG_PREEMPT_VOLUNTARY=y` and adding `CONFIG_PREEMPT=y`:

```bash
# CONFIG_PREEMPT_NONE is not set
# CONFIG_PREEMPT_VOLUNTARY is not set
CONFIG_PREEMPT=y
CONFIG_PREEMPT_COUNT=y
CONFIG_PREEMPTION=y
```

### Enable the ARM Performance Monitor Unit driver.

This will enable us to use the hardware counters provided by the Raspberry Pi. While still in `General setup`, select `Kernel Performance Events and Counters`. Make sure `Kernel performance events and counters` is enabled. Then, return to `General setup` and ensure that `Profiling support` is enabled. Exit `General setup` to return to the main configuration menu. Exit the configurator and be sure to answer `Yes` when asked to save our changes.

## Cross-Compile the Linux Kernel

To track how long the compilation step takes, issue the following command, which compiles the kernel, while writing start and end times to a file:

```bash
date>>time.txt; make -j8 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs; date>>time.txt
```

Once it is done, issue the following command:

```bash
mkdir ../modules
```

This will create a directory that the cross-compiler will use to store the kernel modules that it creates. Then issue the command:

```bash
make -j8 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=../modules modules_install
```

Congratulations! We have just compiled an operating system from source! Now, we can transfer our compiled kernel and built modules to the Raspberry Pi in order to install them. Compress the files to make the transfer process faster:

```bash
tar -C modules/lib -czf modules.tgz modules

tar -C linux/arch/arm -czf boot.tgz boot
```

In a local terminal on the Raspberry Pi, I created a directory called `linux_source` that serves as a place to organize my code. I used `sftp` to transfer the above two compressed files into this directory.

Back up our directories:

```bash
sudo cp -r /usr/lib/modules ~/Desktop/modules_backup

sudo cp -r /boot ~/Desktop/boot_backup
```

## Install the Linux Kernel

Run the following commands to install the kernel we just built:

```bash
tar -xzf modules.tgz

tar -xzf boot.tgz

cd modules

sudo cp -rd * /usr/lib/modules

cd ..

sudo cp boot/dts/*.dtb /boot/

sudo cp boot/dts/overlays/*.dtb* /boot/overlays

sudo cp boot/dts/overlays/README /boot/overlays

sudo cp boot/zImage /boot/kernel7.img
```

If a Raspberry Pi 4 or 4B is used, the last command should be:

```bash
sudo cp boot/zImage /boot/kernel7l.img
```

Finally, our new kernel has installed. When reboot, we will be running our very own, custom kernel.

```sh
pi@xingjian:~ $ uname -a
Linux xingjian 5.10.17-v7-x.xingjian #1 SMP PREEMPT Wed Jan 16 11:52:31 CST 2023 armv7l GNU/Linux
```

The directory tree on the Linux lab machine:

```text
/project/scratch01/compile/x.xingjian
    |- linux_source
        |- linux
        |- modules/lib/modules/5.10.17-v7-x.xingjian (in-tree)
    |- modules (out-of-tree)
```

In-tree modules will match the "local version" on my Raspberry Pi:

```text
linux_source/modules/lib/modules/5.10.17-v7-x.xingjian
```

## Build & Install Kernel Modules

Now, assume that we are using our Linux lab machine. Let's `cd` into the directory that holds our out-of-tree kernel modules (`/project/scratch01/compile/x.xingjian/modules`) and create a simple C program:

```c
/* simple_module.c - a simple template for a loadable kernel module in Linux,
   based on the hello world kernel module example on pages 338-339 of Robert
   Love's "Linux Kernel Development, Third Edition."
 */

#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

/* init function - logs that initialization happened, returns success */
static int 
simple_init(void)
{
    printk(KERN_ALERT "simple module initialized\n");
    return 0;
}

/* exit function - logs that the module is being removed */
static void 
simple_exit(void)
{
    printk(KERN_ALERT "simple module is being unloaded\n");
}

module_init(simple_init);
module_exit(simple_exit);

MODULE_LICENSE ("GPL");
MODULE_AUTHOR ("LKD Chapter 17");
MODULE_DESCRIPTION ("Simple Module Template");
```

Create a `Makefile` in the same directory that contains the following line:

```makefile
obj-m := simple_module.o
```

We can then build the module by issuing the commands:

```bash
module add arm-rpi

LINUX_SOURCE=/project/scratch01/compile/x.xingjian/linux_source/linux
```

and finally compile via:

```bash
make -C $LINUX_SOURCE ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- M=$PWD modules
```

which, if successful, should produce a kernel module file named `simple_module.ko`.

Boot up our Raspberry Pi, open up a terminal window, create a directory to hold our kernel modules. Use `sftp` to get the `simple_module.ko` file. To load the kernel module into the kernel, issue the command:

```bash
sudo insmod simple_module.ko
```

If no error messages pop out, then our module has been successfully loaded. To confirm this, we can check the system log by issuing the command:

```bash
dmesg
```

To confirm our module was loaded, we can also issue the command:

```bash
lsmod
```

to see a listing of all currently loaded kernel modules. To remove the module, issue the command:

```bash
sudo rmmod simple_module.ko
```

The `rmmod` utility calls the underlying `delete_module()` system call. Looking at the source code for `delete_module()` system call, pay attention to these lines:

```c
if (!capable(CAP_SYS_MODULE) || modules_disabled)
    return -EPERM;
```

Back in 1999, with the release of the Linux kernel version 2.2, kernel developers started breaking up the privileges of the root user into distinct **capabilities**, allowing processes to inherit subsets of root's privilege, without giving away too much. Combining capabilities with user namespaces will allow administrators to apply those fine-grained privileges to containers. The `capable()` function (which actually wraps around `ns_capable_common()` determines whether a task has a particular capability or not. If loading or unloading kernel modules is not permitted, `EPERM` error is returned:

```c
// In: /kernel/capability.c
static bool ns_capable_common(struct user_namespace *ns,
                              int cap,
                              unsigned int opts)
{
    int capable;

    if (unlikely(!cap_valid(cap))) {
        pr_crit("capable() called with invalid cap=%u\n", cap);
        BUG();
    }

    capable = security_capable(current_cred(), ns, cap, opts);
    if (capable == 0) {
        current->flags |= PF_SUPERPRIV;
        return true;
    }
    return false;
}

// In: /include/uapi/linux/capability.h
/* Insert and remove kernel modules - modify kernel without limit */
#define CAP_SYS_MODULE       16
```

Userspace programs must use system calls to access operating system resources (e.g., memory, I/O ports, I/O memory, interrupt lines, etc.), and even then, most of the kernel remains opaque to user processes. With kernel modules, we can access all of the kernel's resources directly.