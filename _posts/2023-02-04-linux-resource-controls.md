---
layout:             post
title:              "Linux Resource Controls"
category:           "Computing Systems, Systems Security"
tags:               linux-kernel process-thread namespace cgroup container
permalink:          /posts/linux-plumbing/resource-control
last_modified_at:   "2023-02-06"
---

Some writeup for Washington University CSE 522S studios, which closely follow the [LWN](https://lwn.net/) article series "Namespaces in operation".

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Isolation with Namespaces

There are three system calls that can be used to put tasks into specific namespaces. These are `clone`, `unshare`, and `setns`. The `clone` and `setns` syscalls result in creating a `nsproxy` object and then adding the specific namespaces needed for the task.

## Within-Namespace Resource Controls

The Linux kernel makes use of a resource management feature called "control groups" (`cgroups`) to apply limits on the amount of system resources a process (or group of processes) can acquire. `cgroups` form a tree structure and every process in the system belongs to one and only one `cgroup`. All threads of a process belong to the same `cgroup`.

The `cgroups` feature consists of several subsystems (or *resource controllers*), each of which is responsible for a particular resource type, such as CPUs, memory, I/O, or networks. `cgroups` provide an API (via a pseudo-filesystem) through which users can get and set parameters and limits associated with its subsystems. All controller behaviors are **hierarchical** &mdash; if a controller is enabled on a `cgroup`, it affects all processes which belong to the `cgroups` consisting the inclusive sub-hierarchy of the `cgroup`.

There are two versions of `cgroups` in Linux: [`cgroup v1`](https://docs.kernel.org/admin-guide/cgroup-v1/index.html#cgroup-v1) and [`cgroup v2`](https://docs.kernel.org/admin-guide/cgroup-v2.html). `cgroup v2` is the new generation of the `cgroup` API. Some Linux distributions still use `cgroup v1` by default. Since we cannot simultaneously use a resource controller in both version one and two, we need to reboot the system if the following command shows a value greater than $$1$$:

```bash
# Count cgroup mounts
grep -c cgroup /proc/mounts
```

Unlike version one, `cgroup v2` has only single hierarchy. Initially, only the root `cgroup` exists to which all processes belong. A child `cgroup` can be created by creating a subdirectory in `/sys/fs/cgroup`:

```bash
mkdir child
```

If this `child` control group no longer has any children or live processes, it can be destroyed by removing the directory:

```bash
rmdir child
```

The Raspberry Pi OS launches the `systemd` daemon during system startup. This utility is responsible for configuring much of the kernel and userspace functionality of the Raspberry Pi, including mounting the appropriate `cgroup` pseudo-filesystem(s). Certain commands can be issued to `systemd` to change its boot-time behavior via the `/boot/cmdline.txt` file. Note that commands in that file are separated by spaces. To disable the `cgroups v1` subsystem, add the following command to the end of the file:

```text
cgroup_no_v1=all
```

After rebooting the Raspberry Pi, we can check that only the `cgroup v2` subsystem is mounted by issuing the following command:

```console
pi@xingjian:~ $ mount | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,relatime,nsdelegate)
```

We can see that the filesystem type is `cgroup2`. In control groups version one, the filesystem type is `cgroup`.

### Memory Resource Controller

Memory is a unique resource in the sense that it is present in a limited amount. If a task requires a lot of CPU processing, the task can spread its processing over a period of hours, days, months or years, but with memory, the same physical memory needs to be reused to accomplish the task.

The memory controller *isolates* the memory behavior of a group of tasks from the rest of the system. A request for comments for the memory controller was posted by Balbir Singh[^1]. In that RFC, two types of memory were discussed:

- Non-Reclaimable memory: this memory is not reclaimable until it is explicitly released by the allocator. Examples of such memory include slab allocated memory and memory allocated by the kernel components in process context.
- Reclaimable memory:
  - Anonymous pages - pages allocated by the user space, they are mapped into the user page tables, but not backed by a file
  - File-mapped pages - pages that map a portion of a file
  - Page cache pages - consist of pages used during IPC using `shmfs`; pages of a user mode process that are swapped out; pages from block read/write operations; pages from file read/write operations

The first RSS ("resident set size", the portion of memory occupied by a process that is held in main memory) controller was posted by Balbir Singh in February 2007[^2]. According to Jonathan Corbet[^3], the memory controller can be used to:

1. Isolate an application or a group of applications; Memory hungry applications can be isolated and limited to a smaller amount of memory,
2. Create a `cgroup` with limited amount of memory, this can be used as a good alternative to booting with `mem=XXXX`,
3. Virtualization solutions can control the amount of memory they want to assign to a virtual machine instance,
4. A CD/DVD burner could control the amount of memory used by the rest of the system to ensure that burning does not fail due to lack of available memory . . .

Currently, the following types of memory usages are tracked:

- Userland memory: page cache and anonymous memory;
- Kernel data structures such as dentries and inodes;
- TCP socket buffers.

To enable memory `cgroups` (actually, `cgroups` with memory controller), add the following commands to the `/boot/cmdline.txt` file:

```text
cgroup_memory=1 cgroup_enable=memory
```

After rebooting the Raspberry Pi, we can verify that the memory controller is enabled by issuing the following command:

```console
pi@xingjian:~ $ cat /proc/cgroups
#subsys_name    hierarchy    num_cgroups    enabled
cpuset  0   87  1
cpu     0   87  1
cpuacct 0   87  1
blkio   0   87  1
memory  0   87  1
devices 0   87  1
freezer 0   87  1
net_cls 0   87  1
perf_event  0   87  1
net_prio    0   87  1
pids    0   87  1
```

## Fork Bomb Revisited

[Here]({{ site.baseurl }}/posts/linux-plumbing/fork-execve#fork-bomb) I rediscovered some history episode about fork bomb attack. In this section, I would like to use a fork bomb together with `malloc()` function calls that generate significant memory usage for a control group:

```c
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>

#define PAGE_SIZE   4096

int main()
{
    unsigned int counter = 0;

    while (1) {
        printf("%d. Allocating an entire page...\n", ++counter);
        void *p = malloc(PAGE_SIZE);
        memset(p, 0, PAGE_SIZE);
        sleep(1);

        printf("%d. Forking a child...\n", counter);
        fork();
        sleep(1);
    }

    return 0;
}
```

First, before running the fork bomb, open a terminal window and obtain root privileges with `sudo su` or `sudo bash`. Go to the child control group directory we just created: `/sys/fs/cgroup/child`. Open another terminal window, which we will use to run the fork bomb program, and check its PID:

```bash
echo $$
```

Write this value into our child `cgroup`&prime;s `cgroup.procs` file. This file should be empty before we modify it. Remember that when writing a PID into the `cgroup.procs` file of certain control group, all threads in the corresponding process are moved into that control group at once. Let's take a look at some files:

```console
root@xingjian:/sys/fs/cgroup/child# cat cgroup.procs
root@xingjian:/sys/fs/cgroup/child# echo "912" > cgroup.procs
root@xingjian:/sys/fs/cgroup/child# cat cgroup.procs
912
root@xingjian:/sys/fs/cgroup/child# cat memory.current
0
root@xingjian:/sys/fs/cgroup/child# cat memory.stat
anon 0
file 0
kernel_stack 0
percpu 0
sock 0
shmem 0
file_mapped 0
file_dirty 0
file_writeback 0
inactive_anon 0
active_anon 0
inactive_file 0
active_file 0
unevictable 0
slab_reclaimable 0
slab_unreclaimable 0
slab 0
...
```

Note that at the current stage all the fields inside the `memory.stat` file have value zero. So does `memory.current`. Then, in the second terminal window, we can verify control group membership:
```console
$ cat /proc/self/cgroup
0::/child
```

Compile and run the fork bomb inside the second terminal window with `PID=912`. When it is still running, in the first terminal window, open the `cgroup.procs`, `memory.current`, and `memory.stat` files:

```console
root@xingjian:/sys/fs/cgroup/child# cat cgroup.procs
912
926
927
929
928
930
931
932
933
root@xingjian:/sys/fs/cgroup/child# cat memory.current
4374528
root@xingjian:/sys/fs/cgroup/child# cat memory.stat
anon 11489280
file 0
kernel_stack 2048000
...
inactive_anon 11354112
...
slab_unreclaimable 2904812
slab 2904812
...
pgfault 10395
```

After terminating the fork bomb, check those three files again:

```console
root@xingjian:/sys/fs/cgroup/child# cat cgroup.procs
912
root@xingjian:/sys/fs/cgroup/child# cat memory.current
122880
root@xingjian:/sys/fs/cgroup/child# cat memory.stat
anon 270336
...
inactive_anon 270336
...
slab_unreclaimable 143656
slab 143656
...
pgfault 20427
```

Our `malloc()` function calls do consume a large amount of anonymous memory. However, the sum of `inactive_anon` and `active_anon` is not always equal to `anon`, since the `anon` counter is type-based, not list-based. It is also easy to observe that when the fork bomb program is running, the amount of memory allocated to kernel stacks (`kernel_stack`) is nonzero.

### Setting Hard Limit on Memory Usage

The `cgroups` memory controller provides a way to enforce a hard limit on memory usage through the `memory.max` interface. According to the Linux kernel documentation, this is a read-write single value file which exists on non-root control groups. The default is "`max`":

```console
root@xingjian:/sys/fs/cgroup/child# cat memory.max
max
```

In the first terminal window (running as root), write a value that is sufficiently larger than the current value of `memory.current` such that the fork bomb will cause this limit to be exceeded relatively quickly[^4]:

```console
root@xingjian:/sys/fs/cgroup/child# echo "409600" > memory.max
root@xingjian:/sys/fs/cgroup/child# cat memory.max
409600
```

In the second terminal window, which is a member of the child `cgroup`, modify the fork bomb program so that it no longer delays before each call to `fork()`. Compile and run the fork bomb. The second terminal window closes automatically soon after I launch the fork bomb. Then, open the `memory.events` file in the first terminal window:

```console
root@xingjian:/sys/fs/cgroup/child# cat memory.events
low 0
high 0
max 24
oom 1
oom_kill 1
```

It is clear that our fork bomb was killed by the out-of-memory killer as its memory usage exceeded the hard limit of $$409600$$ bytes.

### Monitoring Memory Usage Using Inotify API

The `cgroups` memory controller also supplies the `memory.high` controller. This allows an administrator to define a memory usage threshold beyond which (1) a "high" event, in the `memory.events` file, is triggered, which subsequently (2) signals to the Linux kernel that it should begin aggressively reclaiming memory from the processes in the `cgroup` (though those processes will not be killed).

```c
// In: include/uapi/linux/inotify.h
struct inotify_event {
    __s32   wd;         /* watch descriptor */
    __u32   mask;       /* watch mask */
    __u32   cookie;     /* cookie to synchronize two events */
    __u32   len;        /* length (including nulls) of name */
    char    name[];     /* stub for possible name */
};
```

## Footnotes

[^1]: Balbir Singh, "RFC: Memory Controller," 30 Oct 2006, [https://lwn.net/Articles/206697/](https://lwn.net/Articles/206697/).

[^2]: Balbir Singh, "Memory Controller (RSS Control)," 19 Feb 2007, [https://lwn.net/Articles/222762/](https://lwn.net/Articles/222762/).

[^3]: Jonathan Corbet, "Controlling Memory Use in Containers," 31 Jul 2007, [https://lwn.net/Articles/243795/](https://lwn.net/Articles/243795/).

[^4]: Note that all memory amounts are in bytes. If a value which is not aligned to `PAGE_SIZE` is written, the value may be rounded up to the closest `PAGE_SIZE` multiple when read back.