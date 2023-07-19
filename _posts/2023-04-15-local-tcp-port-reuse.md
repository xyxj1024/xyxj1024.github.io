---
layout:     post
title:      "The Secret Tri-State Life of a Local TCP Port"
category:   "Linux System Programming"
tags:       linux-kernel networking tcp
permalink:  /blog/linux-plumbing/local-tcp-port-reuse
---

I document in this post my attempt to reproduce eight experiments described in [Cloudflare's 03/20/2023 blog post](https://blog.cloudflare.com/the-quantum-state-of-a-tcp-port/) on an AWS EC2 instance. Python programs are provided [here](https://github.com/cloudflare/cloudflare-blog/blob/master/2023-03-quantum-state-of-tcp-port/).

A TCP/IP connection is identified by a 4-tuple: {source IP, source port, destination IP, destination port}. To establish a TCP/IP connection, only a destination IP and port number are needed. The operating system automatically selects source IP and port. On Linux we can see which source IP will be chosen with `ip route get`.

<!-- excerpt-end -->

The `/proc/sys/net/ipv4/ip_local_port_range` interface file defines the **ephemeral port range** that is used by TCP and UDP traffic to choose the source port. On the AWS EC2 instance I use, this range contains $$60999 + 1 - 32768 = 28232$$ ports:

```console
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768	60999
```

As we run each of the Python `doctests`, we first modify the value `net.ipv4.ip_local_port_range` within the newly created user and network namespace[^1] such that the local port range contains only one port (port number `60000`).

It is possible to ask the kernel to select a specific source IP and port by calling `bind()` before `connect()`. By specifying port number `0`, we ask the kernel to do a port allocation for us. This is a trick called "[bind-before-connect](https://idea.popcount.org/2014-04-03-bind-before-connect/)."

It is always helpful to get a grasp on the Linux kernel data structures before we jump into the experiments. Internally, the kernel stores listening sockets in a hash table ("listener table")[^2] indexed by port numbers using exactly $$32$$ buckets[^3]:

```c
// In: include/linux/list_nulls.h
struct hlist_nulls_head {
    struct hlist_nulls_node *first;
};

struct hlist_nulls_node {
    struct hlist_nulls_node *next, **pprev;
};
#define NULLS_MARKER(value) (1UL | (((long)value) << 1))

// In: include/net/inet_hashtables.h
struct inet_listen_hashbucket {
    spinlock_t              lock;
    struct hlist_nulls_head nulls_head;
};

#define INET_LHTABLE_SIZE   32
```

and binding sockets in a hash table ("bind table") indexed by local port:

```c
struct inet_bind_hashbucket {
    spinlock_t          lock;
    struct hlist_head   chain;
};
```

The bind table can be accessed through `struct inet_hashinfo` as `bhash`. Linux relies on `bhash` to track all TCP ports that are currently in use. The `ehash` field in `struct inet_hashinfo`:

```c
struct inet_ehash_bucket {
    struct hlist_nulls_head chain;
};
```

is used to track sockets with both local and remote address already assigned. For example, to perform established sockets lookup:

```c
// In: net/ipv4/inet_hashtables.c - __inet_lookup_established()
/* skipped */
    unsigned int hash = inet_ehashfn(net, daddr, hnum, saddr, sport);
    unsigned int slot = hash & hashinfo->ehash_mask;
    struct inet_ehash_bucket *head = &hashinfo->ehash[slot];

begin:
    sk_nulls_for_each_rcu(sk, node, &head->chain) {
        if (sk->sk_hash != hash)
            continue;
        if (likely(inet_match(net, sk, acookie, ports, dif, sdif))) {
            if (unlikely(!refcount_inc_not_zero(&sk->sk_refcnt)))
                goto out;
            if (unlikely(!inet_match(net, sk, acookie,
                                    ports, dif, sdif))) {
                sock_gen_put(sk);
                goto begin;
            }
            goto found;
        }
    }
/* skipped */
```

Whenever the Linux kernel needs to bind a socket to a local port, it has to look up the `inet_bind_bucket` structure for that port first:

```c
// In: net/ipv4/inet_connection_sock.c - inet_csk_find_get_port()
/* skipped */
int ret = -EADDRINUSE, port = snum, l3mdev;
struct inet_bind_hashbucket *head, *head2;
struct inet_bind2_bucket *tb2 = NULL;
struct inet_bind_bucket *tb = NULL;
bool head2_lock_acquired = false;
struct net *net = sock_net(sk);

l3mdev = inet_sk_bound_l3mdev(sk);
if (!port) {
    head = inet_csk_find_open_port(sk, &tb, &tb2, &head2, &port);
    if (!head)
        return ret;

    head2_lock_acquired = true;

    if (tb && tb2)
        goto success;
    found_port = true;
} else {
    head = &hinfo->bhash[inet_bhashfn(net, port,
                                      hinfo->bhash_size)];
    spin_lock_bh(&head->lock);
    inet_bind_bucket_for_each(tb, &head->chain)
        if (inet_bind_bucket_match(tb, net, port, l3mdev))
            break;
}
/* skipped */

// In: include/net/inet_hashtables.h
#define inet_bind_bucket_for_each(tb, head) \
    hlist_for_each_entry(tb, head, node)

// In: net/ipv4/inet_hashtables.c
bool inet_bind_bucket_match(const struct inet_bind_bucket *tb, const struct net *net,
                            unsigned short port, int l3mdev)
{
    return net_eq(ib_net(tb), net) && tb->port == port &&
                                      tb->l3mdev == l3mdev;
}
```

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Quiz #1

For Quiz #1, we bind two sockets to distinct, explicitly specified IP addresses, and connect the sockets to the same remote address &mdash; `127.9.9.9:1234`. Our `quiz_1.py` program is listed below:

```python
#!/usr/bin/env -S unshare --user --map-root-user --net -- strace -e %net -- python3

"""
Quiz #1
-------
>>> s1 = socket(AF_INET, SOCK_STREAM)
>>> s1.bind(('127.1.1.1', 0))
>>> s1.connect(('127.9.9.9', 1234))
>>> s1.getsockname(), s1.getpeername()
(('127.1.1.1', 60000), ('127.9.9.9', 1234))
>>>
>>> s2 = socket(AF_INET, SOCK_STREAM)
>>> s2.bind(('127.2.2.2', 0))
>>> s2.connect(('127.9.9.9', 1234))
>>> s2.getsockname(), s2.getpeername()
(('127.2.2.2', 60000), ('127.9.9.9', 1234))
>>>
>>> s1.close()
>>> s2.close()
"""

from os import system
from socket import *

def setup(test):
    system("ip link set dev lo up")
    system("sysctl --write --quiet net.ipv4.ip_local_port_range='60000 60000'")

    ln = socket(AF_INET, SOCK_STREAM)
    ln.bind(("", 1234))
    ln.listen(SOMAXCONN)

    test.globs["ln"] = ln

def teardown(test):
    ln = test.globs["ln"]
    ln.close()

def run_doctest(module_name):
    import doctest
    import unittest

    testsuite = doctest.DocTestSuite(module=module_name,
                                     setUp=setup,
                                     tearDown=teardown)
    unittest.TextTestRunner(verbosity=2).run(testsuite)

if __name__ == "__main__":
    run_doctest(__name__)
```

The function call `socket(AF_INET, SOCK_STREAM)` creates a stream socket in the Internet domain. The output results show that our manually created sockets share the only available port:

```console
$ unshare --user --map-root-user --net \
> -- strace -e %net -- \
> python3 quiz_1.py
__main__ ()
Doctest: __main__ ... --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=27669, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=27670, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 128)                          = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 4
bind(4, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("127.1.1.1")}, 16) = 0
connect(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.1.1.1")}, [16]) = 0
getpeername(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, [16]) = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 5
bind(5, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("127.2.2.2")}, 16) = 0
connect(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = 0
getsockname(5, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.2.2.2")}, [16]) = 0
getpeername(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, [16]) = 0
ok

----------------------------------------------------------------------
Ran 1 test in 0.019s

OK
+++ exited with 0 +++
```

## Quiz #2

For Quiz #2, by not calling `bind()`, we ask the operating system to select both the local IP address and the local port for the first socket:

```python
"""
Quiz #2
-------
>>> s1 = socket(AF_INET, SOCK_STREAM)
>>> s1.connect(('127.9.9.9', 1234))
>>> s1.getsockname(), s1.getpeername()
(('127.0.0.1', 60000), ('127.9.9.9', 1234))
>>>
>>> s2 = socket(AF_INET, SOCK_STREAM)
>>> s2.bind(('127.2.2.2', 0))
>>> s2.connect(('127.9.9.9', 1234))
>>> s2.getsockname(), s2.getpeername()
(('127.2.2.2', 60000), ('127.9.9.9', 1234))
>>>
>>> s1.close()
>>> s2.close()
"""
```

The port is again shared and the selected IP address for the first socket is `127.0.0.1`:

```console
$ unshare --user --map-root-user --net \
> -- strace -e %net -- \
> python3 quiz_2.py
__main__ ()
Doctest: __main__ ... --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=27879, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=27880, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 128)                          = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 4
connect(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
getpeername(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, [16]) = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 5
bind(5, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("127.2.2.2")}, 16) = 0
connect(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = 0
getsockname(5, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.2.2.2")}, [16]) = 0
getpeername(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, [16]) = 0
ok

----------------------------------------------------------------------
Ran 1 test in 0.019s

OK
+++ exited with 0 +++
```

## Quiz #3

For Quiz #3, we explicitly specify the local IP address and the local port for the first socket while let the operating system manage the second socket. Interestingly, just with a different ordering, we receive "`OSError: [Errno 99] Cannot assign requested address`".

```console
$ unshare --user --map-root-user --net \
> -- strace -e %net -- \
> python3 quiz_3.py
__main__ ()
Doctest: __main__ ... --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=28050, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=28051, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 128)                          = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 4
bind(4, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("127.1.1.1")}, 16) = 0
connect(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.1.1.1")}, [16]) = 0
getpeername(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, [16]) = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 5
connect(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = -1 EADDRNOTAVAIL (Cannot assign requested address)
getsockname(5, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("0.0.0.0")}, [16]) = 0
getpeername(5, 0x7ffcaca80ca0, [16])    = -1 ENOTCONN (Transport endpoint is not connected)
FAIL

======================================================================
FAIL: __main__ ()
Doctest: __main__
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/lib64/python3.7/doctest.py", line 2204, in runTest
    raise self.failureException(self.format_failure(new.getvalue()))
AssertionError: Failed doctest test for __main__
  File "quiz_3.py", line 2, in __main__

----------------------------------------------------------------------
File "quiz_3.py", line 13, in __main__
Failed example:
    s2.connect(('127.9.9.9', 1234))
Exception raised:
    Traceback (most recent call last):
      File "/usr/lib64/python3.7/doctest.py", line 1337, in __run
        compileflags, 1), test.globs)
      File "<doctest __main__[5]>", line 1, in <module>
        s2.connect(('127.9.9.9', 1234))
    OSError: [Errno 99] Cannot assign requested address
----------------------------------------------------------------------
File "quiz_3.py", line 14, in __main__
Failed example:
    s2.getsockname(), s2.getpeername()
Exception raised:
    Traceback (most recent call last):
      File "/usr/lib64/python3.7/doctest.py", line 1337, in __run
        compileflags, 1), test.globs)
      File "<doctest __main__[6]>", line 1, in <module>
        s2.getsockname(), s2.getpeername()
    OSError: [Errno 107] Transport endpoint is not connected


----------------------------------------------------------------------
Ran 1 test in 0.022s

FAILED (failures=1)
+++ exited with 0 +++
```

`connect()` returns `EADDRNOTAVAIL`:

```c
// In: net/ipv4/af_inet.c - __inet_bind()
/* Not specified by any standard per-se, however it breaks too
 * many applications when removed.  It is unfortunate since
 * allowing applications to make a non-local bind solves
 * several problems with systems using dynamic addressing.
 * (ie. your servers still start up even if your ISDN link
 *  is temporarily down)
 */
err = -EADDRNOTAVAIL;
if (!inet_addr_valid_or_nonlocal(net, inet, addr->sin_addr.s_addr, chk_addr_ret))
    goto out;
```

What's the issue here? According to Cloudflare's original post:

> The bind bucket lookup can happen early &mdash; at `bind()` time &mdash; or late &mdash; at `connect()` time. Which one gets called depends on how the connected socket has been set up . . . However, whether we land in `inet_csk_get_port` or `__inet_hash_connect`, we always end up walking the bucket chain in the `bhash` looking for the bucket with a matching port number. The bucket might already exist or we might have to create it first. But once it exists, its `fastreuse` field is in one of three possible states: `-1`, `0`, or `+1`.

`connect()` returns `-EADDRNOTAVAIL` without doing a 4-tuple check when the hash buckets were previously allocated by `bind()` and all local ports are used. `bind()` creates the local port hash buckets in `inet_csk_get_port()`. Depending on the socket options it sets `tb->fastreuse` and `tb->fastreuseport` to `0` or `1` in the bucket. `connect()`, or late local address allocation, always fails when TCP bind bucket for a local port is in any state other than `fastreuse = -1`:

```c
// In: net/ipv4/inet_hashtables.c - __inet_hash_connect()
/* skipped */
	offset &= ~1U;
other_parity_scan:
	port = low + offset;
	for (i = 0; i < remaining; i += 2, port += 2) {
		if (unlikely(port >= high))
			port -= remaining;
		if (inet_is_local_reserved_port(net, port))
			continue;
		head = &hinfo->bhash[inet_bhashfn(net, port,
						  hinfo->bhash_size)];
		spin_lock_bh(&head->lock);

        inet_bind_bucket_for_each(tb, &head->chain) {
            if (inet_bind_bucket_match(tb, net, port, l3mdev)) {
                if (tb->fastreuse >= 0 ||
                    tb->fastreuseport >= 0)
                    goto next_port;
                WARN_ON(hlist_empty(&tb->owners));
                if (!check_established(death_row, sk, port, &tw))
                    goto ok;
                goto next_port;
            }
        }

        tb = inet_bind_bucket_create(hinfo->bind_bucket_cachep,
                                     net, head, port, l3mdev);
        if (!tb) {
            spin_unlock_bh(&head->lock);
            return -ENOMEM;
        }
        tb_created = true;
        tb->fastreuse = -1;
        tb->fastreuseport = -1;
        goto ok;
next_port:
        spin_unlock_bh(&head->lock);
        cond_resched();
    }
/* skipped */
```

To get around this, we can add the function call `setsockopt(SOL_IP, IP_BIND_ADDRESS_NO_PORT, 1)` for the first socket:

```python
"""
Quiz #3
-------
>>> s1 = socket(AF_INET, SOCK_STREAM)
>>> s1.setsockopt(SOL_IP, IP_BIND_ADDRESS_NO_PORT, 1)
>>> s1.bind(('127.1.1.1', 0))
>>> s1.connect(('127.9.9.9', 1234))
>>> s1.getsockname(), s1.getpeername()
(('127.1.1.1', 60000), ('127.9.9.9', 1234))
>>>
>>> s2 = socket(AF_INET, SOCK_STREAM)
>>> s2.connect(('127.9.9.9', 1234))
>>> s2.getsockname(), s2.getpeername()
(('127.0.0.1', 60000), ('127.9.9.9', 1234))
>>>
>>> s1.close()
>>> s2.close()
"""
```

The solution is also discussed [here](https://blog.cloudflare.com/how-to-stop-running-out-of-ephemeral-ports-and-start-to-love-long-lived-connections/). According to the [Linux manual page](https://man7.org/linux/man-pages/man7/ip.7.html), the `IP_BIND_ADDRESS_NO_PORT` option "inform the kernel to not reserve an ephemeral port when using `bind()` with a port number of `0`. The port will later be automatically chosen at `connect()` time, in a way that allows sharing a source port as long as the 4-tuple is unique."

```c
// In: include/uapi/linux/in.h
#define IP_BIND_ADDRESS_NO_PORT 24

// In: include/linux/socket.h
#define SOL_IP                  0

// In: net/ipv4/ip_sockglue.c
int do_ip_setsockopt(struct sock *sk, int level, int optname,
                     sockptr_t optval, unsigned int optlen)
{
    struct inet_sock *inet = inet_sk(sk);
    /* skipped */
    switch (optname) {
    case IP_BIND_ADDRESS_NO_PORT:
    /* skipped */
        inet->bind_address_no_port = val ? 1 : 0;
        break;
    /* skipped */
    }
    /* skipped */
}

// In: net/ipv4/af_inet.c
int __inet_bind(struct sock *sk, struct sockaddr *uaddr, int addr_len,
                u32 flags)
{
    /* skipped */
	/* Make sure we are allowed to bind here. */
	if (snum || !(inet->bind_address_no_port ||
                  (flags & BIND_FORCE_ADDRESS_NO_PORT))) {
        err = sk->sk_prot->get_port(sk, snum);
        if (err) {
            inet->inet_saddr = inet->inet_rcv_saddr = 0;
            goto out_release_sock;
        }
        /* skipped */
	}
    /* skipped */
}
```

## Quiz #4

For Quiz #4, we try to connect to two distinct remote IP addresses from `127.0.0.1:60000`.

```python
"""
Quiz #4
-------
>>> s1 = socket(AF_INET, SOCK_STREAM)
>>> s1.connect(('127.8.8.8', 1234))
>>> s1.getsockname(), s1.getpeername()
(('127.0.0.1', 60000), ('127.8.8.8', 1234))
>>>
>>> s2 = socket(AF_INET, SOCK_STREAM)
>>> s2.connect(('127.9.9.9', 1234))
>>> s2.getsockname(), s2.getpeername()
(('127.0.0.1', 60000), ('127.9.9.9', 1234))
>>>
>>> s1.close()
>>> s2.close()
"""
```

The local port is shared by our manually created sockets:

```console
__main__ ()
Doctest: __main__ ... --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=28826, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=28827, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 128)                          = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 4
connect(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.8.8.8")}, 16) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
getpeername(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.8.8.8")}, [16]) = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 5
connect(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = 0
getsockname(5, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
getpeername(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, [16]) = 0
ok

----------------------------------------------------------------------
Ran 1 test in 0.016s

OK
+++ exited with 0 +++
```

## Quiz #5

How about explicitly specifying the source IP and port for the first socket using `s1.bind(('127.0.0.1', 0))`? We obtain another "`OSError: [Errno 99] Cannot assign requested address`":

```console
__main__ ()
Doctest: __main__ ... --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=28934, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=28935, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 128)                          = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 4
bind(4, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
connect(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.8.8.8")}, 16) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
getpeername(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.8.8.8")}, [16]) = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 5
connect(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = -1 EADDRNOTAVAIL (Cannot assign requested address)
getsockname(5, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("0.0.0.0")}, [16]) = 0
getpeername(5, 0x7fff1b0c6880, [16])    = -1 ENOTCONN (Transport endpoint is not connected)
FAIL

======================================================================
FAIL: __main__ ()
Doctest: __main__
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/lib64/python3.7/doctest.py", line 2204, in runTest
    raise self.failureException(self.format_failure(new.getvalue()))
AssertionError: Failed doctest test for __main__
  File "quiz_5.py", line 2, in __main__

----------------------------------------------------------------------
File "quiz_5.py", line 13, in __main__
Failed example:
    s2.connect(('127.9.9.9', 1234))
Exception raised:
    Traceback (most recent call last):
      File "/usr/lib64/python3.7/doctest.py", line 1337, in __run
        compileflags, 1), test.globs)
      File "<doctest __main__[5]>", line 1, in <module>
        s2.connect(('127.9.9.9', 1234))
    OSError: [Errno 99] Cannot assign requested address
----------------------------------------------------------------------
File "quiz_5.py", line 14, in __main__
Failed example:
    s2.getsockname(), s2.getpeername()
Exception raised:
    Traceback (most recent call last):
      File "/usr/lib64/python3.7/doctest.py", line 1337, in __run
        compileflags, 1), test.globs)
      File "<doctest __main__[6]>", line 1, in <module>
        s2.getsockname(), s2.getpeername()
    OSError: [Errno 107] Transport endpoint is not connected


----------------------------------------------------------------------
Ran 1 test in 0.018s

FAILED (failures=1)
+++ exited with 0 +++
```

The solution is to bind source IP and port, then execute `setsockopt(SOL_IP, IP_BIND_ADDRESS_NO_PORT, 1)` for each of the sockets:

```python
"""
Quiz #5
-------
>>> s1 = socket(AF_INET, SOCK_STREAM)
>>> s1.setsockopt(SOL_IP, IP_BIND_ADDRESS_NO_PORT, 1)
>>> s1.bind(('127.0.0.1', 0))
>>> s1.connect(('127.8.8.8', 1234))
>>> s1.getsockname(), s1.getpeername()
(('127.0.0.1', 60000), ('127.8.8.8', 1234))
>>>
>>> s2 = socket(AF_INET, SOCK_STREAM)
>>> s2.setsockopt(SOL_IP, IP_BIND_ADDRESS_NO_PORT, 1)
>>> s2.bind(('127.0.0.1', 0))
>>> s2.connect(('127.9.9.9', 1234))
>>> s2.getsockname(), s2.getpeername()
(('127.0.0.1', 60000), ('127.9.9.9', 1234))
>>>
>>> s1.close()
>>> s2.close()
"""
```

## Quiz #6

For Quiz #6, let's use `bind(('127.0.0.1', 60_000))` for both sockets:

```console
__main__ ()
Doctest: __main__ ... --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=1211, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=1212, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 128)                          = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 4
bind(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
connect(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.8.8.8")}, 16) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
getpeername(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.8.8.8")}, [16]) = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 5
bind(5, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EADDRINUSE (Address already in use)
connect(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = -1 EADDRNOTAVAIL (Cannot assign requested address)
getsockname(5, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("0.0.0.0")}, [16]) = 0
getpeername(5, 0x7ffd3cca98a0, [16])    = -1 ENOTCONN (Transport endpoint is not connected)
getsockname(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
getpeername(4, 0x7ffd3cca9a00, [16])    = -1 ENOTCONN (Transport endpoint is not connected)
getsockname(5, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("0.0.0.0")}, [16]) = 0
getpeername(5, 0x7ffd3cca9a00, [16])    = -1 ENOTCONN (Transport endpoint is not connected)
FAIL

======================================================================
FAIL: __main__ ()
Doctest: __main__
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/lib64/python3.7/doctest.py", line 2204, in runTest
    raise self.failureException(self.format_failure(new.getvalue()))
AssertionError: Failed doctest test for __main__
  File "quiz_6.py", line 2, in __main__

----------------------------------------------------------------------
File "quiz_6.py", line 13, in __main__
Failed example:
    s2.bind(('127.0.0.1', 60_000))
Exception raised:
    Traceback (most recent call last):
      File "/usr/lib64/python3.7/doctest.py", line 1337, in __run
        compileflags, 1), test.globs)
      File "<doctest __main__[5]>", line 1, in <module>
        s2.bind(('127.0.0.1', 60_000))
    OSError: [Errno 98] Address already in use
----------------------------------------------------------------------
File "quiz_6.py", line 14, in __main__
Failed example:
    s2.connect(('127.9.9.9', 1234))
Exception raised:
    Traceback (most recent call last):
      File "/usr/lib64/python3.7/doctest.py", line 1337, in __run
        compileflags, 1), test.globs)
      File "<doctest __main__[6]>", line 1, in <module>
        s2.connect(('127.9.9.9', 1234))
    OSError: [Errno 99] Cannot assign requested address
----------------------------------------------------------------------
File "quiz_6.py", line 15, in __main__
Failed example:
    s2.getsockname(), s2.getpeername()
Exception raised:
    Traceback (most recent call last):
      File "/usr/lib64/python3.7/doctest.py", line 1337, in __run
        compileflags, 1), test.globs)
      File "<doctest __main__[7]>", line 1, in <module>
        s2.getsockname(), s2.getpeername()
    OSError: [Errno 107] Transport endpoint is not connected


----------------------------------------------------------------------
Ran 1 test in 0.020s

FAILED (failures=1)
+++ exited with 0 +++
```

`bind()` returns `EADDRINUSE`. We need to use `setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)` on both sockets:

```python
"""
Quiz #6
-------
>>> s1 = socket(AF_INET, SOCK_STREAM)
>>> s1.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
>>> s1.bind(('127.0.0.1', 60_000))
>>> s1.connect(('127.8.8.8', 1234))
>>> s1.getsockname(), s1.getpeername()
(('127.0.0.1', 60000), ('127.8.8.8', 1234))
>>>
>>> s2 = socket(AF_INET, SOCK_STREAM)
>>> s2.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
>>> s2.bind(('127.0.0.1', 60_000))
>>> s2.connect(('127.9.9.9', 1234))
>>> s2.getsockname(), s2.getpeername()
(('127.0.0.1', 60000), ('127.9.9.9', 1234))
>>>
>>> s1.close()
>>> s2.close()
"""
```

## Quiz #7

What if we remove the `bind()` function call for the first socket in the success version of `quiz_6.py`? The program succeeds as well.

## Quiz #8

This is Quiz #7 but in reverse.

```console
__main__ ()
Doctest: __main__ ... --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=1326, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=1327, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 128)                          = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 4
setsockopt(4, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
connect(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, 16) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(60000), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
getpeername(4, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.9.9.9")}, [16]) = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_IP) = 5
setsockopt(5, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
connect(5, {sa_family=AF_INET, sin_port=htons(1234), sin_addr=inet_addr("127.8.8.8")}, 16) = -1 EADDRNOTAVAIL (Cannot assign requested address)
FAIL

======================================================================
FAIL: __main__ ()
Doctest: __main__
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/lib64/python3.7/doctest.py", line 2204, in runTest
    raise self.failureException(self.format_failure(new.getvalue()))
AssertionError: Failed doctest test for __main__
  File "quiz_8.py", line 2, in __main__

----------------------------------------------------------------------
File "quiz_8.py", line 15, in __main__
Failed example:
    s2.connect(('127.8.8.8', 1234))
Exception raised:
    Traceback (most recent call last):
      File "/usr/lib64/python3.7/doctest.py", line 1337, in __run
        compileflags, 1), test.globs)
      File "<doctest __main__[7]>", line 1, in <module>
        s2.connect(('127.8.8.8', 1234))
    OSError: [Errno 99] Cannot assign requested address


----------------------------------------------------------------------
Ran 1 test in 0.021s

FAILED (failures=1)
+++ exited with 0 +++
```

`connect()` returns `EADDRNOTAVAIL` for the second socket. Currently, I don't know how to fix this situation.

## Footnotes

[^1]: A network namespace, represented by `struct net` defined in `include/net/net_namespace.h`, is logically another copy of the network stack, with its routes, firewall rules, and network devices. The initial network namespace, `struct net init_net`, includes the loopback device and all physical devices, the networking tables, etc. Each newly created network namespace includes only the loopback device (to verify this, run `unshare --net bash` then run `ifconfig -a`). Check out [this post]({{ site.baseurl }}/blog/linux-plumbing/linux-network-namespaces) for a general overview of Linux network namespaces.

[^2]: [Jonathan Corbet](https://lwn.net/Articles/612021/): "Hash tables are heavily used within the kernel to speed access to objects of interest. Using a hash table will be faster than, say, a linear search through a single list, but there is always value in making accesses faster yet. Quite a bit of work has been done toward this goal over the years; for example, the use of read-copy-update (RCU) can allow the addition and removal of items from a hash bucket list without locking out readers. Some operations have proved harder to make concurrent in that manner, though, with table resizing being near the top of the list."

[^3]: [Eric Dumazet](https://lore.kernel.org/netdev/20200102212844.0D734E0095@unicorn.suse.cz/): "They must use [the nulls protocol](https://www.kernel.org/doc/Documentation/RCU/rculist_nulls.txt), so that a lookup can detect a socket in a hash list was moved in another one."