---
layout:             post
title:              "SEED Labs 2.0: Meltdown and Spectre Attack Labs Writeup"
category:           "Computing Systems"
tags:               system-security microarchitecture cache
permalink:          /side-channels-seedlab/
last_modified_at:   "2022-11-07"
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org). The lab assignment was conducted using SEED virtual machine configured on a AWS EC2 instance.

In 2017, it was discovered that many modern processors, including those from Intel and ARM, are vulnerable to attacks called **Meltdown**{: style="color: red"} and **Spectre**{: style="color: red"}. Meltdown allows a user-level program to read data stored inside the kernel memory, leading to data leakage. Spectre exploits a race condition vulnerability in the design of the speculative execution implemented in most CPUs, which allows a malicious program to read the data from the area that is not accessible to it. Unlike the Meltdown attack, the restricted area does not need to be inside the kernel; it can be in the same process space as the malicious program, making defending the Spectre attack much more difficult.

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## An Introduction to Cache Attacks

Assume that a **set-associative memory cache** on modern processors is organized into $S$ *cache sets*, each of which contains $W$ *cache lines*. In common terminology, $W$ is called the *associativity* and the corresponding cache is called $W$-*way associative*. A cache line is a storage cell consisting of, let's say, $B$ bytes. Thus, a cache has $S \cdot W \cdot B$ bytes in size.

### Comparing Data Access Time of CPU Cache with Main Memory

In the following code (<code>CacheTime.c</code>), we have an array of size <code>10 * 4096</code> initialized to ones. Since a typical cache block size is 64 bytes, no two elemnets used in the program fall into the same cache block.

```c
/* CacheTime.c */
#include <x86intrin.h>
#include <stdio.h>

#define N 10

uint8_t array[N * 4096];

int main(int argc, const char **argv)
{
    unsigned int junk = 0;
    register uint64_t time1, time2;
    volatile uint8_t *addr;
    int i;

    for (i = 0; i < N; i++) {
        array[i * 4096] = 1;
    }

    for (i = 0; i < N; i++) {
        _mm_clflush(&array[i * 4096]);
    }

    array[3 * 4096] = 100;
    array[7 * 4096] = 200;

    for (i = 0; i < N; i++) {
        addr = &array[i * 4096];
        time1 = __rdtscp(&junk);
        junk = *addr;
        time2 = __rdtscp(&junk) - time1;
        printf("Access time for array[%d * 4096]: %5u CPU cycles.\n", i, (unsigned int)time2);
    }

    return 0;
}
```

[<code>x86intrin.h</code>](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.7-4.6/+/refs/heads/jb-dev/lib/gcc/x86_64-linux/4.6.x-google/include/x86intrin.h) is the main header file for all x86 intrinsics functions.

The prototypes for [Intel Streaming SIMD Extensions 2 (Intel SSE2) intrinsics](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/) for cacheability support are in the <code>emmintrin.h</code> header file, where the function <code>_mm_clflush()</code> is defined as:
```c
#include <emmintrin.h>

/* Cache line containing p is flushed and invalidated
 * from all caches in the coherency domain. */
void _mm_clflush (void const* p);
```

The function <code>__rdtscp()</code> is defined as:
```c
#include <immintrin.h>

/* dst[63:0] := TimeStampCounter
 * MEM[mem_addr+31:mem_addr] := IA32_TSC_AUX[32:0] */
unsigned __int64 __rdtscp (unsigned int * mem_addr);
```

Compile <code>CacheTime.c</code> using the following command:
```bash
gcc -march=native CacheTime.c -o CacheTime
```

On my Mac PC (1.6 GHz Dual-Core Intel Core i5 with macOS Monterey 12.6.1 installed):
```console
$ sysctl -a | grep machdep
...
machdep.cpu.cache.linesize: 64
machdep.cpu.cache.L2_associativity: 8
machdep.cpu.cache.size: 256
...
machdep.cpu.brand_string: Intel(R) Core(TM) i5-5250U CPU @ 1.60GHz
$ sysctl -a | grep cache
...
hw.l1icachesize: 32768
hw.l1dcachesize: 32768
hw.l2cachesize: 262144
hw.l3cachesize: 3145728
...
```
running the program produces the following results:
```console
$ ./CacheTime
Access time for array[0 * 4096]:   134 CPU cycles.
Access time for array[1 * 4096]:   134 CPU cycles.
Access time for array[2 * 4096]:   182 CPU cycles.
Access time for array[3 * 4096]:    50 CPU cycles.
Access time for array[4 * 4096]:   138 CPU cycles.
Access time for array[5 * 4096]:   150 CPU cycles.
Access time for array[6 * 4096]:   140 CPU cycles.
Access time for array[7 * 4096]:    68 CPU cycles.
Access time for array[8 * 4096]:   142 CPU cycles.
Access time for array[9 * 4096]:   142 CPU cycles.
```

We can see that the accesses of <code>array[3 * 4096]</code> and <code>array[7 * 4096]</code> are consistenty faster than the accesses of other elements because these two elements are already in the cache while the others are not.