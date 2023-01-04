---
layout:     post
title:      "Concurrent Stacks"
category:   "Data Structures, Algorithms, Programming Languages"
tags:       object-oriented-programming concurrency tree
permalink:  /concurrent-stacks/
---



<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Some Preliminaries

Please refer to Herlihy et al. (2021)[^1] (or Herlihy's 1991 paper[^2]) for more details.

### Amdahl's Law

Define the *speedup* $$S$$ of a job to be the ratio between the time it takes one processor to complete the job (as measured by a wall clock) versus the time it takes $$n$$ concurrent processors to complete the same job. Assume, for simplicity, that it takes (normalized) time $$1$$ for a single processor to complete the job. With $$n$$ concurrent processors and $$p$$ the fraction of the job that can be executed in parallel, the parallel part takes time $$p/n$$ and the sequential part takes time $$1 - p$$. Overall, the parallelized computation takes time

$$1 - p + \frac{p}{n}.$$

The maximum speedup, that is, the ratio between the sequential (single-processor) time and the parallel time, is

$$S = \frac{1}{1 - p + \frac{p}{n}}.$$

It follows from Amdahl's law that concurrent objects whose methods hold exclusive locks, and therefore effectively execute one after the other, are less desirable than ones with finer-grained locking or no locks at all.

### Synchronization

A *concurrent object* is a data structure shared by concurrent threads or processes. The traditional approach to implementing such objects centers around the use of *critical sections*:

**Definition 1.** (Mutual Exclusion) A block of code (i.e., critical section) can be executed by only one thread at a time. The standard way to achieve mutual exclusion is through a `Lock` object satisfying:

```java
public interface Lock {
    public void lock();
    public void unlock();
}
```

A lock is needed for critical sections that take more than one instruction. Atomic instructions are used to arbitrate between simultaneous attempts to acquire the lock, but if the lock is busy, waiting is done in software. When a lock is busy, the waiting process can either block, relinquishing the processor to do other work, or spin until the lock is released. The principal synchronization instruction on many early multiprocessor architectures was the **test-and-set** instruction. It operates on a single memory word (or byte) that may be either `true` or `false`. The following `TASLock` class implements this lock in Java using the `AtomicBoolean` class in the `java.util.concurrent` package:

```java
public class TASLock implements Lock {
    AtomicBoolean state = new AtomicBoolean(false);
    public void lock() {
        while (state.getAndSet(true)) {}
    }
    public void unlock() {
        state.set(false);
    }
}
```

*Contention* on a lock occurs when multiple threads try to acquire the lock at the same time. Not surprisingly, the performance of spinning on test-and-set degrades badly as the number of spinning processors increases. Two factors cause this degradation:
1. In order to release the lock, the lock holder must contend with spinning processors for exclusive access to the lock location.
2. On architectures where test-and-set requests share the same bus or network as normal memory references, the requests of spinning processors can slow accesses to other locations by the lock holder or by other busy processors.

It is more effective for the thread to *back off* for some duration, giving the competing threads a chance to finish, for example:

```java
public class Backoff {
    final int minDelay, maxDelay;
    int limit;
    public Backoff(int min, int max) {
        minDelay = min;
        maxDelay = max;
        limit = minDelay;
    }
    public void backoff() throws InterruptedException {
        int delay = ThreadLocalRandom.current().nextInt(limit);
        limit = Math.min(maxDelay, 2 * limit);
        Thread.sleep(delay);
    }
}
```

The **compare-and-swap** (CAS) instruction is similar to, but more complicated than, the test-and-set instruction. It reads a memory location, compares the read value with an expected value, and stores a new value in the memory location when the read value matches the expected value. Otherwise, nothing is done:

```java
public class SimulatedCAS {
    private int value;
    public synchronized int getValue() {
        return value;
    }
    public synchronized int compareAndSwap(int expectedValue, int newValue) {
        int oldValue = value;
        if (value == expectedValue) {
            value = newValue;
        }
        return oldValue;
    }
}
```

Concurrent algorithms based on CAS are called *lock-free*, because threads do not ever have to wait for a lock. Either the CAS operation succeeds or it doesn't, but in either case, it completes in a predictable amount of time. If the CAS fails, the caller can retry the CAS operation or take another action as it sees fit.

### Basic Concepts of Shared-Memory Computation

**Definition 2.** (Compositionality) A correctness property $$\mathcal{P}$$ is *compositional* if, whenever each object in the system satisfies $$\mathcal{P}$$, the system as a whole satisfies $$\mathcal{P}$$.

**Definition 3.** (Sequential Consistency) Method calls should appear to happen in a one-at-a-time, sequential order (in which method calls do not overlap). Method calls should appear to take effect in program order (the order in which a single thread issues method calls). Sequential consistency is a *nonblocking* correctness condition such that for any pending method call in a sequentially consistent concurrent execution, there is some sequentially consistent response, that is, a response to the invocation that could be given immediately without violating sequential consistency[^2]. Sequential consistency is not compositional; that is, the result of composing sequentially consistent components is not itself necessarily sequentially consistent.

**Definition 4.** (Linearizability) Each method call should appear to take effect instantaneously at some moment between its invocation and response. Every linearizable execution is sequentially consistent, but not vice versa. Linearizability is compositional.

**Definition 5.** (Wait-freedom) A method of an object implementation is *wait-free* if every call finishes its execution in a finite number of steps; that is, if a thread with a pending invocation to a wait-free method keeps taking steps, it completes in a finite number of steps. Wait-freedom is a nonblocking progress condition because a wait-free implementation cannot be blocking: An arbitrary delay by one thread cannot prevent other threads from making progress. The wait-free condition rules out any kind of mutual exclusion, and guarantees independent progress, that is, without making assumptions about the operating system scheduler.

**Definition 6.** (Lock-freedom) A method of an object implementation is *lock-free* if executing the method guarantees that *some* method call finishes in a finite number of steps; that is, if a thread with a pending invocation to a lock-free method keeps taking steps, then within a finite number of its steps, some pending call to a method of that object (not necessarily the lock-free method) completes.

Concurrent threads reading and writing shared memory locations, which are called *registers* for historical reasons, is the simplest form of shared-memory computation. A *read-write register* is an object that encapsulates a value that can be observed by a `read()` method ("load") and modified by a `write()` method ("store"):

```java
public interface Register<T> {
    T read();
    void write(T v);
}
```

If method calls do not overlap, a register implementation should behave as shown below:

```java
public class SequentialRegister<T> implements Register<T> {
    private T value;
    public T read() {
        return value;
    }
    public void write(T v) {
        value = v;
    }
}
```

A single-reader, single-writer (SRSW) or multi-reader, single-writer (MRSW) register implementation is *safe* if:
- A `read()` call that does not overlap a `write()` call returns the value written by the most recent `write()` call.
- A `read()` call that overlaps a `write()` call may return any value within the register's allowed range of values.

An *atomic* register is a linearizable implementation of the sequential register. A *regular* register is an SRSW or MRSW register where writes do not happen atomically. Both safe and regular registers permit only a single writer.

A regular Boolean MRSW register:

```java
public class RegularBooleanMRSWRegister implements Register<Boolean> {
    ThreadLocal<Boolean> last;
    boolean s_value;    // safe MRSW register
    RegularBooleanMRSWRegister(int capacity) {
        last = new ThreadLocal<Boolean>() {
            protected Boolean initialValue() { return false; };
        };
    }
    public void write(Boolean x) {
        if (x != last.get()) {
            last.set(x);
            s_value = x;
        }
    }
    public Boolean read() {
        return s_value;
    }
}
```

When the newly written value `x` is the same as the old, the regular register can only return `x`, while a safe register may return either Boolean value.

### Shared Counting

In its purest form, a *counter* is an object that holds an integer value and provides a *fetch-and-increment* operation, incrementing the counter and returning its previous value. The code snippet below shows the counter class written to use CAS:

```java
public class CASCounter {
    private SimulatedCAS value = new SimulatedCAS();
    public int getValue() {
        return value.getValue();
    }
    public int increment() {
        int oldValue = value.getValue();
        while (value.compareAndSwap(oldValue, oldValue + 1) != oldValue) {
            oldValue = value.getValue();
        }
        return oldValue + 1;
    }
}
```

Scalable counting can only achieved by methods that are distributed and therefore have low contention on memory and interconnect, and are parallel, and thus allow many requests to be dealt with concurrently[^4].

## Concurrent Stack

## Notes

[^1]: Maurice Herlihy, Nir Shavit, Victor Luchangco and Michael Spear, *The Art of Multiprocessor Programming (Second Edition)*, Morgan Kaufmann, 2021.

[^2]: Maurice Herlihy, "Wait-Free Synchronization," *ACM Transactions on Programming Languages and Systems*, Vol. 11, No. 1, January 1991, Pages 124-149.

[^3]: In the systems literature, a nonblocking operation returns immediately without waiting for the operation to take effect, whereas a blocking operation does not return until the operation is complete.

[^4]: Nir Shavit and Asaph Zemach, "Diffracting Trees," *ACM Transactions on Computer Systems*, Vol. 14, No. 4, November 1996, Pages 385-428.