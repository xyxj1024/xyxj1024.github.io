---
layout:             post
title:              "Linux Control Groups: Observing CPU Contention"
category:           "Linux System Programming"
tags:               linux-kernel process-thread cgroup
permalink:          /posts/linux-plumbing/cgroups/cpu-contention
---

Some writeup for Washington University CSE 522S: Studio 9 "CPU Control and Timing Events".

The `parallel_dense_mm.c` program shown below, using the OpenMP library to run in parallel,
1. takes a single command-line argument,
2. creates two dense matrices of size specified by the command-line argument,
3. fills them with randomly-generated values, and then
4. multiplies them.

<!-- excerpt-end -->

Full credit to [Dr. David Ferry](https://cs.slu.edu/people/ferry):

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

Later, we will use this program to generate heavy CPU usage on all available cores of the Raspberry Pi. To compile it against the OpenMP library, issue the following command:

```bash
gcc -Wall -o parallel_dense_mm parallel_dense_mm.c -fopenmp
```

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## The CPU Resource Controller

CPU utilization is another area where implementing `cgroup2` architecture can make major resource control improvements. When enabled, the CPU controller regulates distribution of CPU cycles and enforces CPU limits for its child `cgroups`. It implements both weight (through `cpu.max` API) and absolute bandwidth limit (through `cpu.weight` API) models for normal scheduling policy, and an absolute bandwidth allocation model for real-time scheduling policy.

`cgroup v1` allowed threads to be in any `cgroups` which created an interesting problem where threads belonging to a parent `cgroup` and its children `cgroups` competed for resources. This was nasty as two different types of entities competed and there was no obvious way to settle it. Different controllers did different things. The CPU controller considered threads and `cgroups` as equivalents and mapped nice levels to `cgroup` weights. This worked for some cases but fell flat when children wanted to be allocated specific ratios of CPU cycles and the number of internal threads fluctuated &mdash; the ratios constantly changed as the number of competing entities fluctuated. There also were other issues. The mapping from nice level to weight was not obvious or universal, and there were various other knobs which simply were not available for threads. Tejun Heo summarized current discussions around the CPU controller on `cgroups v2` in [this document](https://git.kernel.org/cgit/linux/kernel/git/tj/cgroup.git/tree/Documentation/cgroup-v2-cpu.txt?h=cgroup-v2-cpu).

```c
/* exec_time.c */
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

The above `exec_time.c` program
1. prints its own PID,
2. blocks on input from `stdin`,
3. once it receives any input, proceeds to execute the command: `time ./parallel_dense_mm 500`.

Compile and run this program. After it prints its PID, but before pressing a key to proceed, write its PID into the `cgroup.procs` file in the CPU `cgroup`'s directory. Then, allow the program to proceed:

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

## Observing CPU Contention with Concurrent Tasks

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

## Assigning Larger Weight

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

Below is my console outputs with `cpu.weight` modified into `500`:

```console
(First terminal window)
$ ./exec_time 5000
PID: 1988
Wait for user input to proceed...

Generating matrices...
^CCommand terminated by signal 2
3.17user 1.20system 0:05.09elapsed 85%CPU (0avgtext+0avgdata 392996maxresident)k
0inputs+0outputs (0major+97986minor)pagefaults 0swaps

(Second terminal window)
$ ./exec_time 500
PID: 1989
Wait for user input to proceed...

Generating matrices...
Multiplying matrices...
Multiplication done!
8.84user 0.00system 0:02.88elapsed 306%CPU (0avgtext+0avgdata 7324maxresident)k
0inputs+0outputs (0major+1563minor)pagefaults 0swaps
```

For the instance that processed matrix size $$n = 500$$, both the total number of CPU-seconds spent in user mode and the elapsed real time are greater than that in the first section but smaller than that in the second section. The total number of CPU-seconds spent in kernel mode is the smallest so far. By assigning higher weights, the CPU `cgroups` can access more CPU resources and thereby ensure performance.

## Applying Bandwidth Constraint

First, reset the value of the `cpu.weight` controller file to its original, default value. Next, apply a bandwidth limit by writing into the `cpu.max` interface file. This file takes the format:

```text
MAX PERIOD
```

Where `MAX` indicate the maximum total time (in microseconds) that processes in the `cgroup` can execute on contended CPUs for every `PERIOD` of elapsed time. This restricts the bandwidth of processes in that `cgroup` to `MAX/PERIOD`. The initial content of the `cpu.max` file is as follows:

```console
# cat cpu.max
max 100000
```

Use values that are sufficiently small so that we will be able to see throttling behavior. For example, if our `exec_time` program measured an elapsed time of $$t$$ seconds to run `parallel_dense_mm` with matrices of size $$500 \times 500$$, then use a `MAX` of $$t/5$$ seconds (converted to microseconds) and a `PERIOD` at least twice the `MAX` value. Note that `PERIOD` cannot be set to a value exceeding $$1,000,000$$. Here, I would like to use this new value pair:

```console
# echo "400000 1000000" > cpu.max
# cat cpu.max
400000 1000000
```

i.e., an expected CPU utilization of $$40\%$$.

Now, proceed to measure the execution time the same way we did in the previous section, running an instance of our program with large matrices, and a second instance with $$500 \times 500$$ matrices, which is added to the `cgroup` to constrain its bandwidth.

Below is my console outputs:

```console
(First terminal window)
$ ./exec_time 5000
PID: 2142
Wait for user input to proceed...

Generating matrices...
^CCommand terminated by signal 2
71.98user 1.81system 0:25.55elapsed 288%CPU (0avgtext+0avgdata 587380maxresident)k
0inputs+0outputs (0major+158484minor)pagefaults 0swaps

(Second terminal window)
$ ./exec_time 500
PID: 2144
Wait for user input to proceed...

Generating matrices...
Multiplying matrices...
Multiplication done!
9.06user 0.04system 0:21.80elapsed 41%CPU (0avgtext+0avgdata 7304maxresident)k
0inputs+0outputs (0major+2021minor)pagefaults 0swaps
```

We can see that the total number of CPU-seconds spent in user mode is slightly greater than that in the second section; the CPU-seconds spent in kernel mode is smaller than that in the second section; the elapsed real time is significantly greater than all of the experiments we have run. The bandwidth constraint takes effect on the elapsed real time value rather than the total number of CPU-seconds spent in user mode.