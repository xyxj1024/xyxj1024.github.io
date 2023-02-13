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

### The Memory Resource Controller

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

### The CPU and CPU Set Controllers

CPU utilization is another area where implementing `cgroup2` architecture can make major resource control improvements. When enabled, the CPU controller regulates distribution of CPU cycles and enforces CPU limits for its child `cgroups`. It implements both weight and absolute bandwidth limit models for normal scheduling policy, and an absolute bandwidth allocation model for realtime scheduling policy.

The following C program, using the OpenMP library to run in parallel, takes a single command-line argument, creates two dense matrices of size specified by the command-line argument, fills them with randomly-generated values, and then multiplies them:

```c
/******************************************************************************
* 
* parallel_dense_mm.c
* 
* This program implements a dense matrix multiply and can be used as a
* hypothetical workload. 
*
* Usage: This program takes a single input describing the size of the matrices
*        to multiply. For an input of size N, it computes A*B = C where each
*        of A, B, and C are matrices of size N*N. Matrices A and B are filled
*        with random values. 
*
* Written Sept 6, 2015 by David Ferry
******************************************************************************/

// Compile using: gcc -Wall -o parallel_dense_mm parallel_dense_mm.c -fopenmp
#include <stdio.h>  //For printf()
#include <stdlib.h> //For exit() and atoi()
#include <assert.h> //For assert()

const int num_expected_args = 2;
const unsigned sqrt_of_UINT32_MAX = 65536;

// The following line can be used to verify that the parallel computation
// gives identical results to the serial computation. If the verficiation is
// successful then the program executes normally. If the verification fails
// the program will terminate with an assertion error.

int main( int argc, char* argv[] ){

	unsigned index, row, col; //loop indicies
	unsigned matrix_size, squared_size;
	double *A, *B, *C;
	#ifdef VERIFY_PARALLEL
	double *D;
	#endif

	if( argc != num_expected_args ){
		printf("Usage: ./parallel_dense_mm <size of matrices>\n");
		exit(-1);
	}

	matrix_size = atoi(argv[1]);
	
	if( matrix_size > sqrt_of_UINT32_MAX ){
		printf("ERROR: Matrix size must be between zero and 65536!\n");
		exit(-1);
	}

	squared_size = matrix_size * matrix_size;

	printf("Generating matrices...\n");

	A = (double*) malloc( sizeof(double) * squared_size );
	B = (double*) malloc( sizeof(double) * squared_size );
	C = (double*) malloc( sizeof(double) * squared_size );
	#ifdef VERIFY_PARALLEL
	D = (double*) malloc( sizeof(double) * squared_size );
	#endif

	for( index = 0; index < squared_size; index++ ){
		A[index] = (double) rand();
		B[index] = (double) rand();
		C[index] = 0.0;
		#ifdef VERIFY_PARALLEL
		D[index] = 0.0;
		#endif
	}

	printf("Multiplying matrices...\n");

	#pragma omp parallel for private(col, row, index)
	for( col = 0; col < matrix_size; col++ ){
		for( row = 0; row < matrix_size; row++ ){
			for( index = 0; index < matrix_size; index++){
			C[row*matrix_size + col] += A[row*matrix_size + index] *B[index*matrix_size + col];
			}	
		}
	}

	
	#ifdef VERIFY_PARALLEL
	printf("Verifying parallel matrix multiplication...\n");
	for( col = 0; col < matrix_size; col++ ){
		for( row = 0; row < matrix_size; row++ ){
			for( index = 0; index < matrix_size; index++){
			D[row*matrix_size + col] += A[row*matrix_size + index] *B[index*matrix_size + col];
			}	
		}
	}

	for( index = 0; index < squared_size; index++ ) 
		assert( C[index] == D[index] );
	#endif //ifdef VERIFY_PARALLEL

	printf("Multiplication done!\n");

	return 0;
}
```

Compile it against the OpenMP library:

```bash
gcc -Wall -o parallel_dense_mm parallel_dense_mm.c -fopenmp
```

The program, while running, generates heavy CPU usage on all available cores.

Let's write another program (call it "`exec_time`") that:
1. prints its own PID,
2. blocks on input from `stdin`,
3. once it receives any input, proceeds to execute the command: `time ./parallel_dense_mm 500`.

```c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); } while (0)

int main()
{
    char c;
    const char * ptime = "/bin/time";
    char * const arg[] = {"-p", "./parallel_dense_mm", "500", (char *)0};

    printf("PID: %d\n", (int)getpid());

    printf("Wait for user input to proceed...\n");

    while ((c = getc(stdin)) != '\n' && c != EOF) continue;

    if (execve(ptime, arg, NULL) == -1) {
        errExit("execve");
    }

    return 0;
}
```

Compile and run this `exec_time` program. After it prints its PID, but before pressing a key to proceed, write its PID into the `cgroup.procs` file in the CPU `cgroup`'s directory. Then, allow the program to proceed:

```console
$ ./exec_time
PID: 1719
Wait for user input to proceed...

Generating matrices...
Multiplying matrices...
Multiplication done!
8.47user 0.06system 0:02.41elapsed 353%CPU (0avgtext+0avgdata 7248maxresident)k
0inputs+0outputs (0major+1560minor)pagefaults 0swaps
```

Open the `cpu.stat` file under our `/sys/fs/cgroup/child` directory:

```console
# cat cpu.stat
usage_usec 8586965
user_usec 8476490
system_usec 110475
nr_periods 0
nr_throttled 0
throttled_usec 0
```

Although the "`-p`" option was given, the `exec_time` program still output default format string:

```console
%Uuser %Ssystem %Eelapsed %PCPU (%Xtext+%Ddata %Mmax)k
%Iinputs+%Ooutputs (%Fmajor+%Rminor)pagefaults %Wswaps
```

where:

```text
%U  total number of CPU-seconds that the process spent in user mode
%S  total number of CPU-seconds that the process spent in kernel mode
%E  elapsed real time (in [hours:]minutes:seconds)
%P  percentage of the CPU that this job got, computed as (%U + %S) / %E
```

Here we shall notice that, in the `cpu.stat` file, `system_usec` plus `user_usec` equals `usage_usec`. The total number of CPU-seconds that the process spent in user mode reported by the `time` utility matches `user_usec`.

#### Observing CPU Contention with Concurrent Tasks

This time, use this modified version of `exec_time`:

```c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); } while (0)

int main(int argc, char *argv[])
{
    char c;

    if (argc != 2) {
        printf("Usage: %s <matrix size>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    printf("PID: %d\n", (int)getpid());

    printf("Wait for user input to proceed...\n");

    while ((c = getc(stdin)) != '\n' && c != EOF) continue;

    const char * bin = "/bin/time";
    char * const arg[] = {"-p", "./parallel_dense_mm", argv[1], (char *)0};
    if (execve(bin, arg, NULL) == -1) {
        errExit("execve");
    }

    return 0;
}
```

And then, proceed as follows:
1. In one terminal window, run (but do not yet provide it with input to make it proceed) one instance of `exec_time` with a large matrix size (e.g. $$5000$$). This will run sufficiently long such that can be guaranteed to start before, and end after, a second instance of the program.
2. In a second terminal window, run (but do not yet provide it with input to make it proceed) one instance of `exec_time` with the same matrix size as in the previous exercise (i.e. $$500$$). We will time the execution time of this instance.
3. Press `<enter>` in the first terminal window to kick off the larger matrix multiply, then immediately press `<enter>` in the second terminal window to kick off the smaller instance that we will time.
4. Once the second, smaller instance completes, use `<CTRL+C>` to terminate the first instance of the matrix multiplication.

Below is my console outputs:

```console
$ ./exec_time 5000
PID: 1944
Wait for user input to proceed...

Generating matrices...
^CCommand terminated by signal 2
3.72user 1.45system 0:05.61elapsed 92%CPU (0avgtext+0avgdata 465544maxresident)k
0inputs+0outputs (0major+116134minor)pagefaults 0swaps

$ ./exec_time 500
PID: 1946
Wait for user input to proceed...

Generating matrices...
Multiplying matrices...
Multiplication done!
9.04user 0.05system 0:03.17elapsed 286%CPU (0avgtext+0avgdata 7312maxresident)k
0inputs+0outputs (0major+1562minor)pagefaults 0swaps
```

For the instance that processed matrix size $$n = 500$$, with the presence of CPU contention, both the total number of CPU-seconds spent in user mode and the elapsed real time are greater than those of the previous exercise; CPU utilization decreases $$20\%$$.

#### Assigning Larger `cpu.weight`

The initial contents of the `cpu,weight` file:

```console
# cat cpu.weight
100
```

This time, proceed as follows:

1. In one terminal window, run (but do not yet provide it with input to make it proceed) one instance of `exec_time` with a large matrix size (e.g. $$5000$$). This will run sufficiently long such that can be guaranteed to start before, and end after, a second instance of the program.
2. In a second terminal window, run (but do not yet provide it with input to make it proceed) one instance of `exec_time` with the same matrix size as in the previous exercise (i.e. $$500$$). We will time the execution time of this instance.
3. In a third terminal window, write the PID of the second instance of the program (i.e., the smaller instance that we will be timing) into the `cgroup.procs` file in our CPU `cgroup`.
4. Then, we will apply a higher weight to the execution of this process. Write a larger value into the `cpu.weight` file than the current configured value; recall that this value can range from $$1$$ to $$10000$$.
5. Press `<enter>` in the first terminal window to kick off the larger matrix multiply, then immediately press `<enter>` in the second terminal window to kick off the smaller instance that we will time.
6. Once the second, smaller instance completes, use `<CTRL+C>` to terminate the first instance of the matrix multiplication.

The management of large computer systems, with many processors (CPUs)[^5], complex memory cache hierarchies and multiple memory nodes[^6] having non-uniform access times presents additional challenges for the efficient scheduling and memory placement of processes. Large computer systems can benefit from explicitly placing jobs on properly sized subsets of the system. These subsets, or "soft partitions" must be able to be dynamically adjusted, as the job mix changes, without impacting other concurrently executing jobs. The location of the running jobs pages may also be moved when the memory locations are changed. `cpuset` provides a Linux kernel mechanism to constrain which CPUs and memory nodes are used by a process or set of processes.


## Fork Bomb Revisited

[Here]({{ site.baseurl }}/posts/linux-plumbing/fork-execve#fork-bomb) I rediscovered some history episode about fork bomb attack. In this section, I would like to test a fork bomb program which additionally makes `malloc()` function calls that generate significant memory usage for a control group:

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

According to the Linux kernel documentation, each non-root `cgroup` has a "`cgroup.events`" file containing "populated" field indicating whether the `cgroup`&prime;s sub-hierarchy has live processes in it. If there is no live process in the `cgroup` and its descendants, its value is zero; otherwise, `1`. `poll` and `inotify` events are triggered when the value changes.

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

Rewrite and recompile our fork bomb program so that it again delays before each call to `fork()`. Next, let's write a monitor program that does the following:
1. Take exactly two command-line arguments (and print a helpful usage message if more or fewer are given) which are the paths to the files named `cgroup.events` and `memory.events` within the `child` control group, respectively.
2. Attempt to open both files, read-only, using the `open()` system call. If either file cannot be opened, it should print a helpful error message and exit.
3. Create an `inotify` instance, then subsequently add watches for both files, watching for `IN_MODIFY` events.
4. In a loop, watch for events on these files by reading from the file descriptor returned by the `inotify_init()` function. Your program should associate the watch descriptors with the file descriptors returned by the opened files, so that, if either file has been changed, your program knows which is the corresponding opened file descriptor.
5. If the `cgroup.events` file has been changed, the monitor program should print a message indicating whether the `cgroup` is populated or not.
6. If the `memory.events` file has been changed, the monitor program should print the current value of its "`high`" field.

Compile the C program shown below:

```c
#include <sys/inotify.h>
#include <limits.h>
#include <errno.h>
#include <fcntl.h>
#include <poll.h>
#include <unistd.h>
#include <string.h>
#include <signal.h>
#include <stdlib.h>
#include <stdio.h>

#define MAX_EVENTS      1024
#define EVENT_SIZE      (sizeof(struct inotify_event))
#define BUF_LEN         (MAX_EVENTS * (EVENT_SIZE + NAME_MAX))

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); } while (0)

int fd_inotify;
int *fd;
int *wd;
char *p;

void signal_handler(int signum)
{
    inotify_rm_watch(fd_inotify, wd[1]);
    inotify_rm_watch(fd_inotify, wd[2]);
    free(wd);

    close(fd_inotify);

    close(fd[1]);
    close(fd[2]);
    free(fd);

    exit(EXIT_SUCCESS);
}

static void handle_events()
{
    char *pos1;
    char *pos2;
    char *tmp;
    char buffer[BUF_LEN] __attribute__ ((aligned(__alignof__(struct inotify_event))));
    char fileln[BUF_LEN];
    ssize_t l1;
    ssize_t l2;
    const struct inotify_event *event;

    while (1) {
        l1 = read(fd_inotify, buffer, BUF_LEN);
        if (l1 == -1 && errno != EAGAIN) {
            errExit("Read inotify fd");
        }
        if (l1 <= 0) break;
        printf("Read %ld bytes from inotify fd...\n", (long)l1);
            
        for (p = buffer; p < buffer + l1; p += sizeof(struct inotify_event) + event->len) {
            event = (const struct inotify_event *)p;
            if (event->mask & IN_MODIFY) {
                if (wd[1] == event->wd) {
                    l2 = read(fd[1], fileln, BUF_LEN);
                    if (l2 == -1 && errno != EAGAIN) {
                        errExit("Read inotify fd");
                    }
                    // printf("%s\n", fileln);
                    pos1 = strstr(fileln, "populated");
                    pos2 = strstr(fileln, "frozen");
                    for (tmp = pos1; tmp != pos2; ) {
                        putchar(*tmp++);
                    }
                    printf("\n");
                    lseek(fd[1], 0, SEEK_SET); // places the current position at the first byte
                }

                if (wd[2] == event->wd) {
                    l2 = read(fd[2], fileln, BUF_LEN);
                    if (l2 == -1 && errno != EAGAIN) {
                        errExit("Read inotify fd");
                    }
                    // printf("%s\n", fileln);
                    pos1 = strstr(fileln, "high");
                    pos2 = strstr(fileln, "max");
                    for (tmp = pos1; tmp != pos2; ) {
                        putchar(*tmp++);
                    }
                    printf("\n");
                    lseek(fd[2], 0, SEEK_SET);
                }
            }
        }
    }
}

int main(int argc, char *argv[])
{
    int poll_num;
    nfds_t nfds = 1;
    struct pollfd fds[1];

    if (argc != 3) {
        printf("Usage: %s <path/to/cgroup.events> <path/to/memory.events>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    signal(SIGINT, signal_handler);

    /* Open files */
    fd = calloc(argc, sizeof(int));
    if (fd == NULL) {
        errExit("Calloc file descriptors");
    }
    fd[1] = open(argv[1], O_RDONLY);
    if (fd[1] == -1) {
        errExit("Open cgroup.events");
    }
    fd[2] = open(argv[2], O_RDONLY);
    if (fd[2] == -1) {
        errExit("Open memory.events");
    }

    /* Initialize inotify */
    fd_inotify = inotify_init();
    if (fd_inotify == -1) {
        errExit("Inotify init");
    }

    /* Add watches */
    wd = calloc(argc, sizeof(int));
    if (wd == NULL) {
        errExit("Calloc watch descriptors");
    }
    wd[1] = inotify_add_watch(fd_inotify, argv[1], IN_MODIFY);
    if (wd[1] == -1) {
        errExit("Inotify add watch cgroup.events");
    }
    wd[2] = inotify_add_watch(fd_inotify, argv[2], IN_MODIFY);
    if (wd[2] == -1) {
        errExit("Inotify add watch memory.events");
    }

    fds[0].fd = fd_inotify;
    fds[0].events = POLLIN;

    while (1) {
        poll_num = poll(fds, nfds, -1);
        if (poll_num == -1) {
            if (errno == EINTR) continue;
            errExit("poll");
        }
        if (poll_num > 0) {
            if (fds[0].revents & POLLIN) {
                handle_events();
            }
        }
    }

    exit(EXIT_SUCCESS);
}
```

Open three terminal windows, the first running as root, the second running the monitor program, and the third to be added to the `cgroup`. In the third terminal window, again, print its PID using `echo $$`. In the second terminal window, launch our monitor program. In the first terminal window, write the PID of the third terminal window into `cgroup.procs`. Then, also in the first terminal window, print the content of `memory.current`. Write a value into `memory.high` that is sufficiently larger than the current value of `memory.current` such that the fork bomb will cause this limit to be exceeded relatively quickly. Finally, in the third terminal window, launch the fork bomb, allowing it to run until the monitor program begins to give outputs. Let the monitor program print a few messages, then kill the fork bomb, and close the third terminal window.

This is what I have done in the root shell before launching the fork bomb:

```console
root@xingjian:/sys/fs/cgroup# cd child
root@xingjian:/sys/fs/cgrou/child# cat memory.current
0
root@xingjian:/sys/fs/cgrou/child# cat memory.high
max
root@xingjian:/sys/fs/cgrou/child# echo "911" > cgroup.procs
root@xingjian:/sys/fs/cgrou/child# cat cgroup.procs
911
root@xingjian:/sys/fs/cgrou/child# echo "81920" > memory.high
root@xingjian:/sys/fs/cgrou/child# cat memory.high
81920
root@xingjian:/sys/fs/cgrou/child# cat cgroup.events
populated 1
frozen 0
root@xingjian:/sys/fs/cgrou/child# cat memory.events
low 0
high 0
max 0
oom 0
oom_kill 0
```

After launching the fork bomb, the monitor program's window begins to print out the changing values of the `high` field of `memory.events`. After I close the fork bomb's window, the monitor program's window looks like this:

```console
...
Read 16 bytes from inotify fd...
high 122

Read 16 bytes from inotify fd...
high 123

Read 16 bytes from inotify fd...
high 125

Read 16 bytes from inotify fd...
populated 0
frozen 0
oom 0
oom_kill 0

populated 0

^C
```

Check the two files in the root shell:

```console
root@xingjian:/sys/fs/cgroup/child# cat cgroup.events
populated 0
frozen 0
root@xingjian:/sys/fs/cgroup/child# cat memory.events
low 0
high 125
max 0
oom 0
oom_kill 0
```

The results are just as expected. First of all, after the shell process that ran the fork bomb became a member of our `child` control group, the `populated` field of the `cgroup.events` file obtained value `1`. As our fork bomb was running, the monitor program kept printing out the `high` field of the `memory.events` file, reporting its changes. The final value matched the `memory.events` file shown in the root shell. After I closed the third terminal window (thereby killed the shell process that ran the fork bomb), a change to the `populated` field was reported by the monitor program.

## Footnotes

[^1]: Balbir Singh, "RFC: Memory Controller," 30 Oct 2006, [https://lwn.net/Articles/206697/](https://lwn.net/Articles/206697/).

[^2]: Balbir Singh, "Memory Controller (RSS Control)," 19 Feb 2007, [https://lwn.net/Articles/222762/](https://lwn.net/Articles/222762/).

[^3]: Jonathan Corbet, "Controlling Memory Use in Containers," 31 Jul 2007, [https://lwn.net/Articles/243795/](https://lwn.net/Articles/243795/).

[^4]: Note that all memory amounts are in bytes. If a value which is not aligned to `PAGE_SIZE` is written, the value may be rounded up to the closest `PAGE_SIZE` multiple when read back.

[^5]: The CPUs of a system include all the logical processing units on which a process can execute, including, if present, multiple processor cores within a package and Hyper-Threads within a processor core.

[^6]: Memory nodes include all distinct banks of main memory; small and SMP systems typically have just one memory node that contains all the system's main memory, while NUMA (non-uniform memory access) systems have multiple memory nodes.