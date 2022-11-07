---
layout:       post
title:        "SEED Labs 2.0: Meltdown and Spectre Attack Labs Writeup"
category:     "Computing Systems"
tags:         system-security microarchitecture cache
permalink:    /side-channels-seedlab/
---

## Comparing Data Access Time of CPU Cache with Main Memory

```c
/* CacheTime.c */
#include <emmintrin.h>
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

<code>x86intrin.h</code> is the main header file for all x86 intrinsics functions.

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

On my Mac PC (1.6 GHz Dual-Core Intel Core i5 with macOS Monterey 12.6.1 installed), running the program produces the following results:
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