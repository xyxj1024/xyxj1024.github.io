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

Compile and run this `exec_time` program:

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

After it prints its PID, but before pressing a key to proceed, write its PID into the `cgroup.procs` file in the CPU `cgroup`'s directory. Then, allow the program to proceed:

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
(First terminal window)
$ ./exec_time 5000
PID: 1944
Wait for user input to proceed...

Generating matrices...
^CCommand terminated by signal 2
3.72user 1.45system 0:05.61elapsed 92%CPU (0avgtext+0avgdata 465544maxresident)k
0inputs+0outputs (0major+116134minor)pagefaults 0swaps

(Second terminal window)
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

The management of large computer systems, with many processors (CPUs)[^1], complex memory cache hierarchies and multiple memory nodes[^2] having non-uniform access times presents additional challenges for the efficient scheduling and memory placement of processes. Large computer systems can benefit from explicitly placing jobs on properly sized subsets of the system. These subsets, or "soft partitions" must be able to be dynamically adjusted, as the job mix changes, without impacting other concurrently executing jobs. The location of the running jobs pages may also be moved when the memory locations are changed. `cpuset` provides a Linux kernel mechanism to constrain which CPUs and memory nodes are used by a process or set of processes.

## Footnotes

[^1]: The CPUs of a system include all the logical processing units on which a process can execute, including, if present, multiple processor cores within a package and Hyper-Threads within a processor core.

[^2]: Memory nodes include all distinct banks of main memory; small and SMP systems typically have just one memory node that contains all the system's main memory, while NUMA (non-uniform memory access) systems have multiple memory nodes.