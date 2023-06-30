---
layout:             post
title:              "Linux Kernel Memory Management"
category:           "Linux System Programming"
tags:               linux-kernel raspberry-pi cse422-assignment
permalink:          /posts/linux-plumbing/kernel-memory-management
last_modified_at:   "2023-03-22"
---

Washington University E81 CSE 422S: "Operating Systems Organization" Studio 11: "Kernel Memory Management", Spring 2022. I'm using a Raspberry Pi 3 Model B+ running Linux kernel version 5.4.42. The output produced by running `uname -a` is:

```console
Linux xingjian 5.4.42-v7xingjian #1 SMP PREEMPT Thu Feb 10 14:37:24 CST 2022 armv7l GNU/Linux
```

The Linux kernel offers several approaches to memory management, including a variety of memory allocation and deallocation facilities for use in kernel space, which offer different trade-offs between memory utilization and performance, as well as different guarantees on physical contiguity. In this studio, we will:
1. Allocate and deallocate memory for different numbers of objects, using the kernel-level page allocator.
2. Use address translation structures to manipulate the memory allocated via the kernel-level page allocator.

<!-- excerpt-end -->

The operating system lifespan can be split up into two phases: normal execution and bootstrapping. The bootstrapping phase makes temporary use of memory. The normal execution phase splits the memory between a portion that is permanently assigned to the kernel code and data, and a second portion that is assigned for dynamic memory requests. Dynamic memory requests come about from process creation and growth[^1].

For the sake of efficiency, linear addresses (also known as virtual addresses) are grouped in fixed-length intervals called *pages*; contiguous linear addresses within a page are mapped into contiguous physical addresses. In this way, the kernel can specify the physical address and the access rights of a page instead of those of all the linear addresses included in it.

The kernel represents every physical page on the system with a `struct page`. This structure is defined in [`<linux/mm_types.h>`](https://elixir.bootlin.com/linux/v5.4.42/source/include/linux/mm_types.h) (155 lines in total for Linux-5.4.42 kernel). The act of moving a page from memory to disk and back is called *paging*, which includes the translation of the virtual address onto the physical address.

The *memory manager* is a part of the operating system that keeps track of associations between virtual addresses and physical addresses and handles paging. Starting with the 80386, the paging unit of Intel processors handles 4KB pages. The paging unit thinks of all RAM as partitioned into fixed-length *page frames* (sometimes referred to as *physical pages*). Each page frame is a constituent of main memory which contains a page[^2].

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## A Simple Kernel Module Framework

The `kernel_memory.c` file:

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/miscdevice.h>
#include <linux/sched.h>
#include <linux/gfp.h>
#include <linux/slab.h>
#include <linux/list.h>
#include <linux/time.h>
#include <linux/kthread.h>
#include <linux/mm.h>

#include <asm/uaccess.h>

static uint nr_structs = 2000;
module_param(nr_structs, uint, 0644); 

static struct task_struct * kthread = NULL;

static unsigned int
my_get_order(unsigned int value)
{
    unsigned int shifts = 0;

    if (!value)
        return 0;

    if (!(value & (value - 1)))
        value--;

    while (value > 0) {
        value >>= 1;
        shifts++;
    }

    return shifts;
}

static int
thread_fn(void * data)
{
    printk("Hello from thread %s. nr_structs=%u\n", current->comm, nr_structs);

    while (!kthread_should_stop()) {
        schedule();
    }

    return 0;
}

static int
kernel_memory_init(void)
{
    printk(KERN_INFO "Loaded kernel_memory module\n");

    kthread = kthread_create(thread_fn, NULL, "k_memory");
    if (IS_ERR(kthread)) {
        printk(KERN_ERR "Failed to create kernel thread\n");
        return PTR_ERR(kthread);
    }
    
    wake_up_process(kthread);

    return 0;
}

static void 
kernel_memory_exit(void)
{
    kthread_stop(kthread);
    printk(KERN_INFO "Unloaded kernel_memory module\n");
}

module_init(kernel_memory_init);
module_exit(kernel_memory_exit);

MODULE_LICENSE ("GPL");
```

The `Makefile`:

```makefile
# TODO: Change this to the location of your kernel source code
KERNEL_SOURCE=/project/scratch01/compile/x.xingjian/linux_source/linux

EXTRA_CFLAGS += -DMODULE=1 -D__KERNEL__=1

kernel_memory-objs := $(kernel_memory-y)
obj-m := kernel_memory.o

PHONY: all

all:
	$(MAKE) -C $(KERNEL_SOURCE) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- M=$(PWD) modules

clean:
	$(MAKE) -C $(KERNEL_SOURCE) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- M=$(PWD) clean
```

This kernel module takes a single unsigned integer parameter, which can be specified as an argument to `insmod`:

```bash
sudo insmod kernel_memory.ko nr_structs=some value
```

The value defaults to $$2,000$$ if no value is provided for it. The module's initialization function also creates a single kernel thread that simply outputs a message to the system log with the value of the module parameter. The `module_param` macro does not declare the variable for us. We must do that before using the macro. Therefore, typical usage might resemble:

```c
static int allow_live_bait = 1;
module_param(allow_live_bait, bool, 0644);
```

This would be in the outermost scope of our module's source file. In other words, `allow_live_bait` is global to the file[^3].

## Studio Writeup

<p>Here is the <a href="https://drive.google.com/uc?id=1j6oN_g0FguHvQX5GpZZim6cstMjz4VUJ">PDF document</a> for studio writeup.</p>

## Footnotes

[^1]: See Claudia Salzberg Rodriguez, Gordon Fischer and Steven Smolski, *The Linux Kernel Primer: A Top-Down Approach for x86 and PowerPC Architectures*, Pearson Education, 2006, pp. 180.

[^2]: See Daniel P. Bovet and Marco Cesati, *Understanding the Linux Kernel: From I/O Ports to Process Management, 3rd Edition*, O'Reilly Media, 2006, pp. 64.

[^3]: See Robert Love, *Linux Kernel Development, 3rd Edition*, Addison-Wesley, 2010, pp. 346.