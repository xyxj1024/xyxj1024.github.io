---
layout:             post
title:              "Microarchitectural Attacks on a Raspberry Pi"
category:           "Computing Systems"
tags:               hardware-security microarchitecture cache raspberry-pi
permalink:          /march-attacks-raspberry-pi/
last_modified_at:   "2022-12-16"
---

I have attempted to shed light on whys and hows regarding the [Spectre](https://xingjianxuanyuan.github.io/side-channels-seedlab/) and [Meltdown](https://xingjianxuanyuan.github.io/meltdown-seedlab/) attacks with materials drawn from [SEED Labs 2.0](https://seedsecuritylabs.org). A MacBook Air equipped with Intel Core i5-5250U was used to run the C programs. Here in this post, I would like to address the problem of launching microarchitectural attacks on a Raspberry Pi 3 Model B+[^1], which is an ARM machine.

<!-- excerpt-end -->

The officially released CPU information of Raspberry Pi 3 Model B+:

```text
Broadcom BCM2837B0, Cortex-A53 (ARMv8) 64-bit quad-core System-on-Chip (MPSoC) @ 1.4GHz
```

The kernel version can be retrieved by:

```console
$ uname -a
Linux xingjian 5.10.103-v7+ #1529 SMP Tue Mar 8 12:21:37 GMT 2022 armv7l GNU/Linux
```

Executing the `lscpu` command gives me the following information:

```console
Architecture:           armv7l
Byte Order:             Little Endian
CPU(s):                 4
On-line CPU(s) list:    0-3
Thread(s) per core:     1
Core(s) per socket:     4
Socket(s):              1
Vendor ID:              ARM
Model:                  4
Model name:             Cortex-A53
Stepping:               r0p4
CPU max MHz:            1200.0000
CPU min MHz:            600.0000
BogoMIPS:               38.40
Flags:                  half thumb fastmult vfp edsp neon vfpv3 tls vfpv4 idiva idivt vfpd32 lpae evtstrm crc32
```

If I add the line `arm_control=0x200` to the end of the file `/boot/config.txt` and reboot the system, executing the `lscpu` command gives me the following instead:

```console
Architecture:           aarch64
Byte Order:             Little Endian
CPU(s):                 4
On-line CPU(s) list:    0-3
Thread(s) per core:     1
Core(s) per socket:     4
Socket(s):              1
Vendor ID:              ARM
Model:                  4
Model name:             Cortex-A53
Stepping:               r0p4
CPU max MHz:            1200.0000
CPU min MHz:            600.0000
BogoMIPS:               38.40
Flags:                  fp asimd evtstrm crc32 cpuid
```

For this post, I will use the `armv7l` Linux kernel.

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Some Attempts to Adapt x86 Intrinsics Functions for ARMv7

### Replacing `__rdtscp()`

The following C code snippet was used to time memory accesses in our previous Intel-based attacks:

```c
addr = &array[i];
time1 = __rdtscp(&junk);
junk = *addr;
time2 = __rdtscp(&junk) - time1;
```

I conducted an experiment using the C program shown below on my Raspberry Pi:

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdint.h>
#include <time.h>

#define N 10

uint8_t array[N * 4096];

uint64_t arm_v7_get_timing(void)
{
    uint32_t result = 0;
    asm volatile ("MRC p15, 0, %0, c9, c13, 0" : "=r" (result));
    return result;
}

uint64_t get_timing(void)
{
    uint64_t result = 0;
    result = arm_v7_get_timing();
    return result;
}

int main()
{
    uint64_t time1, time2;
    volatile uint8_t *addr;
    unsigned int junk = 0;
    int i;

    for (i = 0; i < N; i += 2) {
        array[i * 4096] = 1;
    }

    for (i = 0; i < N; i++) {
        addr = &array[i * 4096];
        time1 = get_timing();
        junk = *addr;
        time2 = get_timing() - time1;
        printf("Access time for array[%d * 4096]: %5u\n", i, (unsigned int)time2);
    }

    return 0;
}
```

Although the code can be compiled successfully, I received the `Illegal instruction` error after running the program.

### Replacing `_mm_mfence()`

A memory barrier in ARM (or "memory fence" in other architectures) is an instruction that requires the core to apply an ordering constraint between memory operations that occur before and after the memory barrier instruction in the program.

I added the following macro and function definitions to my timing program:

```c
#define DSB     asm volatile ("dsb sy" : : : "memory") /* Data Synchronization Barrier */
#define ISB     asm volatile ("isb sy" : : : "memory") /* Instruction Synchronization Barrier */
#define dsb()   DSB
#define isb()   ISB

void arm_v7_memory_barrier(void)
{
    dsb();
    isb();
}
```

and I tried to call my `get_timing()` function with this memory barrier function. Unfortunately, I obtained the error message stating that selected processor does not support `dsb` and `isb` in ARM mode.

### Replacing `_mm_clflush()`

A cache flush instruction, which serves the legitimate purpose of manually maintaining cache coherency (e.g., for memory-mapped input/output or self-modifying code) is used by attacks such as Flush+Reload and Flush+Flush. On any x86 processor implementing the SSE2 instruction set extension, this flush instruction is available from all privilege levels as `clflush`. For example:

```c
#include <x86intrin.h>

uint8_t array[256 * 4096];

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
```

A similar instruction was introduced for ARM processors only in the most recent architecture version, ARMv8. In contrast to `clflush`, it must be specifically enabled to be accessible from user-space. For all processors with a disabled flush instruction or an earlier architecture version, e.g., ARMv7, only eviction-based cache attacks can be deployed (e.g., Evict+Time, Prime+Probe, Evict+Reload attacks). I conducted an experiment with the following C program that makes use of the built-in cache clearing system call:

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdint.h>
#include <time.h>

#define N 10

uint8_t array[N * 4096];

/* 
 * Source:
 * https://minghuasweblog.wordpress.com/2013/03/29/
 * https://issuetracker.google.com/issues/36906645
 */
void clear_cache(void* start, void* end)
{
    const int syscall = 0xf002; /* __ARM_NR_cacheflush */
    asm volatile (
        "mov    r0, %0\n"
        "mov    r1, %1\n"
        "mov    r7, %2\n"
        "mov    r2, #0x0\n"
        "svc    0x00000000\n"
        :
        :   "r" (start), "r" (end), "r" (syscall)
        :   "r0", "r1", "r7"
    );
}

uint64_t get_monotonic_time(void)
{
    struct timespec t1;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    return t1.tv_sec * 1000000000ULL + t1.tv_nsec;
}

uint64_t get_timing(void)
{
    uint64_t result = 0;
    result = get_monotonic_time();
    return result;
}

int main()
{
    uint64_t time1, time2;
    volatile uint8_t *addr;
    unsigned int junk = 0;
    int i;

    for (i = 0; i < N; i++) {
        array[i * 4096] = 1;
    }

    for (i = 0; i < N; i++) {
        clear_cache(&array[i * 4096], &array[i * 4096] + 1);
    }

    /* Wait for cache flushing to complete */
    for (i = 0; i < 100; i++) {}

    array[3 * 4096] = 100;
    array[7 * 4096] = 200;

    for (i = 0; i < N; i++) {
        addr = &array[i * 4096];
        time1 = get_timing();
        junk = *addr;
        time2 = get_timing() - time1;
        printf("Access time for array[%d * 4096]: %5u\n", i, (unsigned int)time2);
    }

    return 0;
}
```

The syscall number of `cacheflush` is defined in [this header file](https://elixir.bootlin.com/linux/v3.1/source/arch/arm/include/asm/unistd.h) by:

```c
#define __NR_OABI_SYSCALL_BASE	0x900000

#if defined(__thumb__) || defined(__ARM_EABI__)
#define __NR_SYSCALL_BASE	0
#else
#define __NR_SYSCALL_BASE	__NR_OABI_SYSCALL_BASE
#endif

#define __ARM_NR_BASE			(__NR_SYSCALL_BASE+0x0f0000)
#define __ARM_NR_breakpoint		(__ARM_NR_BASE+1)
#define __ARM_NR_cacheflush		(__ARM_NR_BASE+2)
```

Below is the results from running the program:

```console
Access time for array[0 * 4096]:  1146
Access time for array[1 * 4096]:   677
Access time for array[2 * 4096]:   365
Access time for array[3 * 4096]:   364
Access time for array[4 * 4096]:   365
Access time for array[5 * 4096]:   572
Access time for array[6 * 4096]:   364
Access time for array[7 * 4096]:   365
Access time for array[8 * 4096]:   365
Access time for array[9 * 4096]:   364
```

There is no difference between the memory access time of two array elements we accessed after flushing and the other selected elements.

## Denial-of-Service Attack on Shared Cache

## Notes

[^1]: Pi 3 B+ has a $32$ KiB 2-way set-associative **L1 instruction cache** and a $32$ KiB 4-way set-associative **L1 data cache** ($128$ cache sets) per core, and a $512$ KiB 16-way set-associative **shared L2 cache** ($512$ cache sets). The cache line size of these components is $64$ bytes. One can also refer to [the latest Arm Cortex-A53 MPCore Processor Technical Reference Manual](https://developer.arm.com/documentation/ddi0500/latest/) or this site: [https://patchwork.kernel.org/project/linux-arm-kernel/patch/20211218200009.16856-1-rs@noreya.tech/](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20211218200009.16856-1-rs@noreya.tech/).