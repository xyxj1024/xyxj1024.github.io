---
layout:             post
title:              "Concurrent Stacks in Java"
category:           "Data Structures and Algorithms"
tags:               object-oriented-programming concurrency tree
permalink:          /posts/concurrent-stacks
---

A **concurrent stack**{: style="color: red"} is a data structure linearizable to a sequential stack that provides `push` and `pop` operations with the usual LIFO semantics. Linearizability is the *de facto* standard correctness condition for concurrent algorithms. Intuitively, an algorithm is linearizable with respect to a sequential specification if each execution of the algorithm is equivalent to some sequential execution of the specification, where the order between the non-overlapping methods is preserved. In this post, I would like to review a Java implementation of a concurrent stack data structure called the elimination back-off stack. Those who are interested in C++ implementations of concurrent data structures may find [this repo](https://github.com/cksystemsgroup/scal) useful. I also found [this article](http://people.csail.mit.edu/shanir/publications/concurrent-data-structures.pdf) by Mark Moir and Nir Shavit a good source for concurrent programming.

<!-- excerpt-end -->

[R. Kent Treiber](https://dominoweb.draco.res.ibm.com/58319a2ed2b1078985257003004617ef.html) proposed the first nonblocking implementation of concurrent list-based stack. The Treiber stack maintains a `top` pointer which points to the candidate. Every time a new element is pushed, the new element becomes the candidate element and therefore the `top` pointer is redirected to the new element. Similarly, every time the candidate element is popped, the `top` pointer is redirected to the next candidate element. This means that every operation which accesses the Treiber stack modifies the `top` pointer. When an increasing number of threads accesses the Treiber stack concurrently, the contention on the `top` pointer increases, and eventually the Treiber stack does not scale anymore.

```java
public class TreiberStack<T> {
    AtomicReference<Node<T>> top = new AtomicReference<Node<T>>();
    public void push(T item) {
        Node<T> newHead = new Node<T>(item);
        Node<T> oldHead;
        do {
            oldHead = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead, newHead));
    }
    public T pop() {
        Node<T> oldHead;
        Node<T> newHead;
        do {
            oldHead = top.get();
            if (oldHead == null)
                return null;
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));
        return oldHead.item;
    }
    private static class Node <T> {
        public final T item;
        public Node<T> next;
        public Node(T item) {
            this.item = item;
        }
    }
}
```

where

```java
public final boolean compareAndSet(V expect, V update) {
    return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
}
```

atomically sets the value to the given updated value if the current value equals the expected value[^1].

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Some Preliminaries

Please refer to Herlihy et al. (2021)[^2] (or Herlihy's 1991 paper[^3]) for more details.

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

The Java syntax

```java
synchronized (expression) {
    statements
}
```

works as follows:
1. The `expression` is evaluated. It must produce (a reference to) an object which is treated as a lock.
2. The synchronized statement acquires the lock. Locks are reentrant in Java, so the statement will not block if the executing thread already holds it.
3. After the lock is successfully acquired, the `statements` are executed.
4. When control leaves the `statements`, the lock is released[^4].

Many concurrent data structure algorithms that have been developed use locks in order to enforce the correct semantics of the operations to be performed. However, locking risks blocking. Concurrent algorithms based on CAS are called *lock-free*, because threads do not ever have to wait for a lock. Either the CAS operation succeeds or it doesn't, but in either case, it completes in a predictable amount of time. If the CAS fails, the caller can retry the CAS operation or take another action as it sees fit[^5].

Often, programmers perform CAS in a loop to repeatedly attempt an action. For example, incrementing an atomic integer in a CAS loop:

```java
AtomicInteger atomicVar = new AtomicInteger(0);
public void funcLockFree() {
    int localVar = atomicVar.get();
    while (!atomicVar.compareAndSet(localVar, localVar + 1)) {
        localVar = atomicVar.get();
    }
}
```

The `AtomicInteger` class has a [`compareAndSet()`](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/atomic/AtomicInteger.html#compareAndSet(int,int)) method which will compare the value of the `AtomicInteger` instance to an expected value: if `atomicVar` has the expected value of `localVar`, the method sets the value of `atomicVar` to `localVar + 1`. The method returns `true` if the value was set, and `false` if not.

### Shared-Memory Computation

*Instructions* are the lowest level of operations. An instruction accesses a single unit of memory called a *primitive element*. A primitive element is either a basic data type such as an integer, float, pointer, or lock. A data structure is composed of *cells*. Cells are the unit of locking. An *execution* is a sequence of instructions and the return values of those instructions. A *proper execution* of a concurrent program, $$P$$, is an execution, $$E$$, where:
- The first instruction is the first statement of the program.
- The last instruction is an exit from the program.
- Instruction $$k + 1$$ in $$E$$ is the one that the program would emit based on the return values of the first $$k$$ instructions.

**Definition 2.** (Compositionality) A correctness property $$\mathcal{P}$$ is *compositional* if, whenever each object in the system satisfies $$\mathcal{P}$$, the system as a whole satisfies $$\mathcal{P}$$.

**Definition 3.** (Sequential Consistency) Method calls should appear to happen in a one-at-a-time, sequential order (in which method calls do not overlap). Method calls should appear to take effect in program order (the order in which a single thread issues method calls). Sequential consistency is a *nonblocking* correctness condition such that for any pending method call in a sequentially consistent concurrent execution, there is some sequentially consistent response, that is, a response to the invocation that could be given immediately without violating sequential consistency[^6]. Sequential consistency is not compositional; that is, the result of composing sequentially consistent components is not itself necessarily sequentially consistent.

**Definition 4.** (Linearizability) Each method call should appear to take effect instantaneously at some moment between its invocation and response. Every linearizable execution is sequentially consistent, but not vice versa. Linearizability is compositional.

**Definition 5.** (Wait-freedom) A method of an object implementation is *wait-free* if every call finishes its execution in a finite number of steps; that is, if a thread with a pending invocation to a wait-free method keeps taking steps, it completes in a finite number of steps. Wait-freedom is a nonblocking progress condition because a wait-free implementation cannot be blocking: An arbitrary delay by one thread cannot prevent other threads from making progress. The wait-free condition rules out any kind of mutual exclusion, and guarantees independent progress, that is, without making assumptions about the operating system scheduler.

**Definition 6.** (Lock-freedom) A method of an object implementation is *lock-free* if executing the method guarantees that *some* method call finishes in a finite number of steps; that is, if a thread with a pending invocation to a lock-free method keeps taking steps, then within a finite number of its steps, some pending call to a method of that object (not necessarily the lock-free method) completes.

Note that a *wait-free* data structure is a *lock-free* data structure with the additional property that every thread accessing the data structure can complete its operation within a bounded number of steps, regardless of the behavior of other threads.

[This post](https://preshing.com/20120612/an-introduction-to-lock-free-programming/) by [Jeff Preshing](https://github.com/preshing) could serve as a good starting point on lock-free or lockless programming. There is also a short list of links in [this answer on Stack Overflow](https://stackoverflow.com/questions/14167767/what-is-the-difference-between-using-explicit-fences-and-stdatomic/14185300#14185300).

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

## Shared Counting

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

Scalable counting can only achieved by methods that are distributed[^7] and therefore have low contention on memory and interconnect, and are parallel, and thus allow many requests to be dealt with concurrently[^8].

A *combining tree* is a distributed binary-tree-based data structure with a shared counter at its root. Processors combine their increment requests going up the tree from the leaves to the root and propagate the answers down the tree, thus eliminating the need for all processors to actually reach the root in order to increment the counter. In the classic combining tree scheme, scalability as the number of processors $$P$$ increases is achieved by making the tree deeper, adding more levels to make sure that the number of leaves is $$\lceil P / 2 \rceil$$. Under maximal load, the throughput of such a tree will be $$P / (2 \log P)$$ operations per time unit, offering a significant speedup.

A counting tree *balancer* is a computing element with one input wire and two output wires. Tokens arrive on the balancer's input wire at arbitrary times and are output on its output wires. A *balancing tree* of width $$w$$ is a binary tree of balancers, where output wires of one balancer are connected to input wires of another, having one designated root input wire and $$w$$ designated output wires. On a shared-memory, multiprocessor one can implement a balancing tree as a shared data structure, where balancers are records, and wires are pointers from one record to another. Threads arrive at a balancer and it repeatedly sends them up and down, so its top wire always has the same or at most one more than the bottom one. One could implement the balancers in a straightforward way using a bit that threads toggle: they fetch the bit and then complement it, exiting on the output wire they fetched (zero or more):

```java
public class ToggleBit {
    AtomicBoolean toggle = new AtomicBoolean(true);
    public boolean toggle() {
        boolean result;
        do {
            result = toggle.get();
        } while (!toggle.compareAndSet(result, !result));
        return result;
    }
}
```

Elimination is a state when two threads carrying a *token* and *anti-token* meet, and "eliminate" each other. When such two threads meet in a data structure, they can exchange the information they carry. A common way to use elimination is to build an *elimination array*, which is a set of cells where each thread randomly chooses a location and spins waiting another thread to "collide" with. When collision occurred, in case the two collided threads carry a token and anti-token, they can exchange information and leave the data structure:

```java
public class EliminationArray<T> {
    Exchanger<T>[] exchanger;
    final long TIMEOUT;
    final TimeUnit UNIT;
    Random random;
    public EliminationArray(int capacity, long timeout, TimeUnit unit) {
        exchanger = new Exchanger[capacity];
        for (int i = 0; i < capacity; i++)
            exchanger[i] = new Exchanger<>();
        random = new Random();
        TIMEOUT = timeout;
        UNIT = unit;
    }
    public T visit(T value, int range) throws TimeoutException {
        int slot = random.nextInt(exchanger.length);
        return exchanger[slot].exchange(value, TIMEOUT, UNIT);
    }
}

public class Exchanger {
    AtomicReference<ExchangerPackage> slot;
}

public class ExchangerPackage {
    Object value;
    State state;
    Type type;
}
```

Each exchanger contains a single `AtomicReference` which is used as an atomic placeholder for exchanging `ExchangerPackage`, where the `ExchangerPackage` is an object used to wrap the actual data and to mark its state and type.

The `Balancer` data structure can be implemented as:

```java
public class Balancer {
    ToggleBit producerToggle, consumerToggle;
    Exchanger[] eliminationArray;
    Balancer topChild, bottomChild;
    ThreadLocal<Integer> lastSlotRange; // saved as the initial range at the beginning of the next operation
}
```

If many tokens attempt to pass through the same balancer concurrently, the toggle bit quickly becomes a hot spot. Contention would be greatest at the root balancer through which all tokens must pass. A diffracting-balancer data structure adds a special "prism" array in front of the toggle bit in every balancer:

```java
public class Prism {
    private static final int duration = 100;
    Exchanger<Integer>[] exchanger;
    public Prism(int capacity) {
        exchanger = (Exchanger<Integer>[]) new Exchanger[capacity];
        for (int i = 0; i < capacity; i++) {
            exchanger[i] = new Exchanger<Integer>();
        }
    }
    public boolean visit() throws TimeoutException, InterruptedException {
        int me = ThreadID.get();
        int slot = ThreadLocalRandom.current().nextInt(exchanger.length);
        int other = exchanger[slot].exchange(me, duration, TimeUnit.MILLISECONDS);
        return (me < other);
    }
}
```

When a token $$T$$ enters the balancer, it first selects a location $$l$$ in prism uniformly at random. $$T$$ tries to "collide" with the previous token to select $$l$$ or, by waiting for a fixed time, with the next token to do so. If a collision occurs, both tokens leave the balancer on separate wires; otherwise the undiffracted token $$T$$ toggles the bit and leaves accordingly.

```java
public class DiffractingBalancer {
    Prism prism;
    Balancer toggle;
    public DiffractingBalancer(int capacity) {
        prism = new Prism(capacity);
        toggle = new Balancer();
    }
    public int traverse() {
        boolean direction = false;
        try {
            if (prism.visit())
                return 0;
            else
                return 1;
        } catch(TimeoutException ex) {
            return toggle.traverse();
        }
    }
}
```

The `DiffractingTree` structure is based on the `DiffractingBalancer` structure:

```java
public class DiffractingTree {
    DiffractingBalancer root;
    DiffractingTree[] child;
    int size;
    public DiffractingTree(int mySize) {
        size = mySize;
        root = new DiffractingBalancer(size);
        if (size > 2) {
            child = new DiffractingTree[] {
                new DiffractingTree(size / 2);
                new DiffractingTree(size / 2);
            }
        }
    }
    public int traverse() {
        int half = root.traverse();
        if (size > 2) {
            return (2 * (child[half].traverse()) + half);
        } else {
            return half;
        }
    }
}
```

## Elimination Back-off Stack

The idea proposed by Hendler et al. (2004)[^9] is to use a single elimination array as a back-off scheme on a shared lock-free stack. If the threads fail on the stack, they attempt to eliminate on the array; if they fail in eliminating, they attempt to access the stack again and so on. Any operation on the shared stack can be linearized at the access point, and any pair of eliminated operations can be linearized when they met. It delivers the same performance as the simple stack at low loads since it is a back-off scheme. However, unlike the simple stack, it scales well as load increases because:
- the number of successful eliminations grows, allowing many operations to complete in parallel, and
- contention on the head of the shared stack is reduced beyond levels achievable by the best exponential back-off schemes since scores of backed off operations are eliminated in the array and *never* re-attempt to access the shared structure.

```java
public class EliminationBackoffStack<T> {
    AtomicReference<Node<T>> top;
    EliminationArray<T> eliminationArray;
    static final int CAPACITY = 100;
    static final long TIMEOUT = 10;
    static final TimeUnit UNIT = TimeUnit.MILLISECONDS;
    public EliminationBackoffStack() {
        top = new AtomicReference<>(null);
        eliminationArray = new EliminationArray<>(
            CAPACITY, TIMEOUT, UNIT
        );
    }
    public void push(T x) {
        Node<T> n = new Node<>(x);
        while (true) {
            if (tryPush(n)) return;
            try {
                T y = eliminationArray.visit(x);
                if (y == null) return;
            } catch (TimeoutException e) {}   
        }
    }
    public T pop() throws EmptyStackException {
        while (true) {
            Node<T> n = tryPop();
            if (n != null) return n.value;
            try {
                T y = eliminationArray.visit(null);
                if (y != null) return y;
            } catch (TimeoutException e) {}
        }
    }
    protected boolean tryPush(Node<T> n) {
        Node<T> m = top.get();
        n.next = m;
        return top.compareAndSet(m, n);
    }
    protected Node<T> tryPop() throws EmptyStackException {
        Node<T> m = top.get();
        if (m == null) throw new EmptyStackException();
        Node<T> n = m.next;
        return top.compareAndSet(m, n)? m : null;
    }
}

public class Node<T> {
    public T value;
    public Node<T> next;

    public Node(T x) {
        value = x;
    }
}
```

<!--

Lock-free data structures:
- [A single-producer, single-consumer lock-free queue for C++](https://github.com/cameron314/readerwriterqueue), [the article](https://moodycamel.com/blog/2013/a-fast-lock-free-queue-for-c++)
- [`boost/lockfree/spsc_queue.hpp`](https://www.boost.org/doc/libs/1_82_0/boost/lockfree/spsc_queue.hpp), [Class template `spsc_queue`](https://www.boost.org/doc/libs/1_82_0/doc/html/boost/lockfree/spsc_queue.html)
- [A lock-free,concurrent, generic queue in 32 bits](https://nullprogram.com/blog/2022/05/14/), on [Hacker News](https://news.ycombinator.com/item?id=31384602)
- [C11 lock-free stack](https://nullprogram.com/blog/2014/09/02/)
- [Optimizing a ring buffer for throughput](https://rigtorp.se/ringbuffer/)
- [Dmitry Vyukov on Producer-Consumer Queues](https://www.1024cores.net/home/lock-free-algorithms/queues)

-->

## Notes

[^1]: Please refer to [this post](https://blog.csdn.net/qqqqq1993qqqqq/article/details/75211993) for more details.

[^2]: Maurice Herlihy, Nir Shavit, Victor Luchangco and Michael Spear, *The Art of Multiprocessor Programming (Second Edition)*, Morgan Kaufmann, 2021.

[^3]: Maurice Herlihy, "Wait-Free Synchronization," *ACM Transactions on Programming Languages and Systems*, Vol. 11, No. 1, January 1991, Pages 124-149.

[^4]: Please refer to [Professor Dan Grossman's teaching materials](https://homes.cs.washington.edu/~djg/).

[^5]: Herlihy proved the universality of CAS by showing that it can be used to convert any sequential data object into a concurrent, wait-free data object. See Maurice Herlihy, "Impossibility and Universality Results for Wait-Free Synchronization," In *Proceedings of the Seventh Annual ACM Symposium on Principles of Distributed Computing*, pages 276-290, Toronto, Ontario, Canada, August 1988.

[^6]: In the systems literature, a *nonblocking* operation returns immediately without waiting for the operation to take effect, whereas a *blocking* operation does not return until the operation is complete. Algorithms for concurrent access to shared data structures are called *nonblocking* if the failure of any subset of the processes will not indefinitely delay the progress of the live processes.

[^7]: A *distributed counter* is a concurrent object which provides a test-and-increment operation on a shared value. On the basis of a distributed counter, one can implement various fundamental data structures, such as queues or stacks.

[^8]: Nir Shavit and Asaph Zemach, "Diffracting Trees," *ACM Transactions on Computer Systems*, Vol. 14, No. 4, November 1996, Pages 385-428.

[^9]: Danny Hendler, Nir Shavit and Lena Yerushalmi, "A Scalable Lock-free Stack Algorithm," *SPAA'04*, June 27-30, 2004, Barcelona, Spain.