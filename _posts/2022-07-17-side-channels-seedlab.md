---
layout:             post
title:              "SEED Labs 2.0: Meltdown and Spectre Attack Labs Writeup"
category:           "Computing Systems"
tags:               hardware-security microarchitecture cache
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

"Higher-level" caches, which are closer to the processor core, are smaller but faster than "lower-level" caches, which are closer to main memory. The last-level cache (LLC) is a *unified* cache (storing both data and instruction), typically shared among all cores of a multicore chip. An important feature of the LLC in modern Intel processors is its *inclusivity*, i.e. the LLC contains copies of all of the data stored in the higher cache levels. The $\log_{2}B$ lowest-order bits of the memory address (the *line offset*) are used to locate a datum in the cache line. The $\log_{2}S$ consecutive bits starting from bit $\log_{2}B$ of the memory address (the *set index*) is used to locate a cache set when the cache is accessed. The remaining high-order bits are used as a *tag* for each cache line. After locating the cache set, the tag field of the address is matched against the tag of the $W$ lines in the set to identify if one of the cache lines is a cache hit.

[The 2005 paper by Colin Percival](https://www.daemonology.net/papers/htt.pdf) investigated the cryptographic side-channel created by cache memory sharing (on the 2.8 GHz Intel Pentium 4 processor)[^1].

```assembly
    mov         ecx, start_of_buffer
    sub         length_of_buffer, 0x2000    ; Spy's 8192-byte buffer
    rdtsc
    mov         esi, eax
    xor         edi, edi                    ; %edi <- 0
loop:
    prefetcht2   [ecx + edi + 0x2800]       ; buffer[64i + 10240]
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

AES software relies heavily upon **S-box lookups**, which are table lookups using input-dependent indices; i.e., loads from input-dependent addresses in memory. As an example, Barreto's 2003 implementation scrambles a $16$-byte input $n$ using a $16$-byte key $k$, a constant $256$-byte table $S = (99, 124, 119, 123, 242, ...)$, and another constant $256$-byte table $S' = (198, 248, 238, 246, 255, ...)$. These two $256$-byte tables are expanded into four $1024$-byte tables $T_{0}, T_{1}, T_{2}, T_{3}$:

$$
\begin{align}
T_{0}[b] &= (S'[b], S[b], S[b], S[b] \oplus S'[b]) \\
T_{1}[b] &= (S[b] \oplus S'[b], S'[b], S[b], S[b]) \\
T_{2}[b] &= (S[b], S[b] \oplus S'[b], S'[b], S[b]) \\
T_{3}[b] &= (S[b], S[b], S[b] \oplus S'[b], S'[b]).    
\end{align}
$$

where $\oplus$ means exclusive-or. What is the consequence? Consider calculating $T_{0}[k[0] \oplus n[0]]$ near the beginning of the AES computation.

## References

[^1]: Colin Percival, "Cache Missing for Fun and Profit," In *BSDCan 2005*, Ottawa, CA, 2005.