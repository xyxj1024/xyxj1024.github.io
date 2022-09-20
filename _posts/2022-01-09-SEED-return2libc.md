---
layout:       post
title:        "SEED Labs 2.0: Return-to-libc Attack Lab (32-bit) Writeup"
category:     "Computing Systems"
tags:         system-security
permalink:    /seed-return2libc/
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org). The lab assignment was conducted using SEED virtual machine configured on a AWS EC2 instance.

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

```console
     Non-Executable Stack
-----------------------------
|                        _  |
| ...                    |  |
-------------------------|---          Executable Code Region
| Return Address         |  |--------> -----------------------
-------------------------|---          |  system() function  |
| Previous Frame Pointer |  |          -----------------------
-------------------------|---
| ...           Overflow |  |
| ...           Buffer   |  |
-----------------------------
|                           |
```

## Environment Setup

First, Ubuntu and several other Linux-based systems use **address space randomization** to randomize the starting address of heap and stack, making guessing the exact addresses difficult. Guessing addresses is one of the critical steps of buffer-overflow attacks. In this lab, we disable this
feature using the following command:

```console
$ sudo sysctl -w kernel.randomize_va_space=0
```

When the memory address randomization is turned off, for the *same* program, the <code>libc</code> library is always loaded in the *same* memory address.

The <code>gcc</code> compiler implements a security mechanism called *StackGuard* to prevent buffer overflows. In the presence of this protection, buffer overflow attacks do not work. We can compile a program with StackGuard disabled using the <code>-fno-stack-protector</code> option:

```console
# Use the -m32 flag to compile the program into 32-bit binary
$ gcc -m32 -fno-stack-protector <program_name>.c
```

Always compile a program using the "<code>-z noexecstack</code>" option in this lab:

```console
$ gcc -m32 -z noexecstack   -o test test.c
```

The non-executable stack countermeasure is exactly what we are trying to defeat!

Link <code>/bin/sh</code> to <code>zsh</code> to disable the protection by <code>/bin/dash</code> so that our commands can be executed in a <code>Set-UID</code> process:

```console
$ sudo ln -sf /bin/zsh /bin/sh
```

Finally, we can compile our target vulnerable program <code>retlib.c</code>:

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#ifndef BUF_SIZE
#define BUF_SIZE 12
#endif

int bof(char *str)
{
    char buffer[BUF_SIZE];
    unsigned int *framep;

    // Copy ebp into framep
    asm("movl %%ebp, %0" : "=r" (framep));

    // Print out information for experiment purpose
    printf("Address of buffer[] inside bof():   0x%.8x\n", (unsigned)buffer);
    printf("Frame Pointer value inside bof():   0x%.8x\n", (unsigned)framep);

    // Buffer Overflow!
    strcpy(buffer, str);

    return 1;
}

int main(int argc, char **argv)
{
    char input[1000];
    FILE *badfile;

    badfile = fopen("badfile", "r");
    int length = fread(input, sizeof(char), 1000, badfile);
    printf("Address of input[] inside main():   0x%x\n", (unsigned int) input);
    printf("Input size: %d\n", length);

    bof(input);

    printf("(^_^)(^_^) Returned Properly (^_^)(^_^)\n");
    return 1;
}

// This function will be used in the optional task
void foo()
{
    static int i = 1;
    printf("Function foo() is invoked %d times\n", i++);
    return;
}
```

```console
seed@xingjian:~$ make
gcc -m32 -DBUF_SIZE=12 -fno-stack-protector -z noexecstack -o retlib retlib.c
sudo chown root retlib && sudo chmod 4755 retlib
seed@xingjian:~$ ls
Makefile    exploit.py  retlib  retlib.c
```

## Task 1: Finding out the Addresses of <code>libc</code> Functions

```console
$ gdb -q retlib
Reading symbols from retlib...
(No debugging symbols found in retlib)
gdb-peda$ break main
Breakpoint 1 at 0x130f
gdb-peda$ run
Starting program: /home/seed/Documents/return2libc/retlib 
[----------------------------------registers-----------------------------------]
EAX: 0xf7fac808 --> 0xffffd6ac --> 0xffffd7f8 ("SHELL=/bin/bash")
EBX: 0x0 
ECX: 0x51a74f1d 
EDX: 0xffffd634 --> 0x0 
ESI: 0xf7faa000 --> 0x1e6d6c 
EDI: 0xf7faa000 --> 0x1e6d6c 
EBP: 0x0 
ESP: 0xffffd60c --> 0xf7de1ee5 (<__libc_start_main+245>:        add    esp,0x10)
EIP: 0x5655630f (<main>:        endbr32)
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x5655630a <foo+58>: mov    ebx,DWORD PTR [ebp-0x4]
   0x5655630d <foo+61>: leave  
   0x5655630e <foo+62>: ret    
=> 0x5655630f <main>:   endbr32 
   0x56556313 <main+4>: lea    ecx,[esp+0x4]
   0x56556317 <main+8>: and    esp,0xfffffff0
   0x5655631a <main+11>:        push   DWORD PTR [ecx-0x4]
   0x5655631d <main+14>:        push   ebp
[------------------------------------stack-------------------------------------]
0000| 0xffffd60c --> 0xf7de1ee5 (<__libc_start_main+245>:       add    esp,0x10)
0004| 0xffffd610 --> 0x1 
0008| 0xffffd614 --> 0xffffd6a4 --> 0xffffd7d0 ("/home/seed/Documents/return2libc/retlib")
0012| 0xffffd618 --> 0xffffd6ac --> 0xffffd7f8 ("SHELL=/bin/bash")
0016| 0xffffd61c --> 0xffffd634 --> 0x0 
0020| 0xffffd620 --> 0xf7faa000 --> 0x1e6d6c 
0024| 0xffffd624 --> 0xf7ffd000 --> 0x2bf24 
0028| 0xffffd628 --> 0xffffd688 --> 0xffffd6a4 --> 0xffffd7d0 ("/home/seed/Documents/return2libc/retlib")
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x5655630f in main ()
gdb-peda$ p system
$1 = {<text variable, no debug info>} 0xf7e08420 <system>
gdb-peda$ p exit
$2 = {<text variable, no debug info>} 0xf7dfaf80 <exit>
```

Or we can also put these <code>gdb</code> commands into a file and then ask <code>gdb</code> to execute the commands from this file:

```console
$ cat gdb_command.txt
break main
run
p system
p exit
quit
$ gdb -q -batch -x gdb_command.txt ./retlib
```

## Task 2: Putting the shell string in the memory

Let us define a new shell variable <code>MYSHELL</code>, and let it contain the string "<code>/bin/sh</code>":

```console
$ export MYSHELL=/bin/sh
$ env | grep MYSHELL
MYSHELL=/bin/sh
```

The location of this variable in the memory can be found out easily using the following program:

```c
#include <stdio.h>

void main()
{
    char* shell = getenv("MYSHELL");
    if (shell)
        printf("%x\n", (unsigned int) shell);
}
```

The program name <code>prtenv</code> is chosen to match the length of <code>retlib</code>.

```console
$ gcc -m32 -z noexecstack -o prtenv prtenv.c
$ ./prtenv
ffffd838
```

## Task 3: Launching the Attack

