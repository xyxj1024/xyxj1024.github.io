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

The `cgroups` feature consists of several subsystems (or *resource controllers*), each of which is responsible for a particular resource type, such as CPUs, memory, I/O, or networks. `cgroups` provide an API (via a pseudo-filesystem) through which users can get and set parameters and limits associated with its subsystems. All controller behaviors are **hierarchical**---if a controller is enabled on a `cgroup`, it affects all processes which belong to the `cgroups` consisting the inclusive sub-hierarchy of the `cgroup`.

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

A request for comments for the memory controller was posted by Balbir Singh[^1]. In that RFC, two types of memory were discussed:

- Non-Reclaimable memory: this memory is not reclaimable until it is explicitly released by the allocator. Examples of such memory include slab allocated memory and memory allocated by the kernel components in process context.
- Reclaimable memory:
  - Anonymous pages - pages allocated by the user space, they are mapped into the user page tables, but not backed by a file
  - File-mapped pages - pages that map a portion of a file
  - Page cache pages - consist of pages used during IPC using `shmfs`; pages of a user mode process that are swapped out; pages from block read/write operations; pages from file read/write operations

The first RSS ("resident set size", the portion of memory occupied by a process that is held in main memory) controller was posted by Balbir Singh in February 2007[^2].

To enable memory `cgroups`, add the following commands to the `/boot/cmdline.txt` file:

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

### Fork Bomb Revisited

[here]({{ site.baseurl }}/posts/linux-plumbing/fork-execve#fork-bomb) I rediscovered some history episode about fork bomb attack. In this section, I would like to use a fork bomb together with `malloc()` function calls that generate significant memory usage for a control group.

## Footnotes

[^1]: Balbir Singh, "RFC: Memory Controller", 30 Oct 2006, [https://lwn.net/Articles/206697/](https://lwn.net/Articles/206697/).

[^2]: Balbir Singh, "Memory Controller (RSS Control)", 19 Feb 2007, [https://lwn.net/Articles/222762/](https://lwn.net/Articles/222762/).