---
layout:             post
title:              "Configure and Cross-Compile a Linux Kernel for a Raspberry Pi"
category:           "Computing Systems, Systems Security"
tags:               operating-system raspberry-pi
permalink:          /posts/configure-compile-linux-kernel
---

Studio 1 and 2 of [CSE 522S: "Advanced Operating Systems" at Washington University in St. Louis](https://classes.engineering.wustl.edu/cse522s). I am using a Raspberry Pi 3 Model B Plus for the course, which has a quad-core ARM Cortex-A53 (ARMv8) CPU.

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

```console
[x.xingjian@linuxlab002 linux]$ make kernelversion
5.10.17
```

The first several lines of Makefile look like:

```cmake
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

```console
pi@xingjian: ̃ $ uname -a
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