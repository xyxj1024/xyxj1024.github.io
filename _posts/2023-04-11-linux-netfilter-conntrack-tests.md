---
layout:     post
title:      "Linux Netfilter Connection Tracking: Two Simple Tests"
category:   "Linux System Programming"
tags:       linux-kernel networking namespace packet-filtering syn-flood
permalink:  /posts/linux-plumbing/netfilter-conntrack-tests
---

The Linux connection tracking subsystem (in many scenarios called "the `conntrack` table"), built on top of [the Netfilter framework](https://www.netfilter.org/), stores information about the state of a network connection in a memory structure that contains the source and destination IP addresses, port number pairs, protocol types, state, and timeout.

![netfilter-packet-flow](/assets/images/schematic/netfilter-packet-flow.png){:class="img-responsive"}
<p style="text-align:center;color:gray;font-size:80%;">
<a href="https://commons.wikimedia.org/wiki/File:Netfilter-packet-flow.svg">Schematic for the packet flow paths through Linux networking and Xtables</a> (Author: Jan Engelhardt)
</p>

[This post](https://arthurchiao.art/blog/conntrack-design-and-implementation/) by Arthur Chiao (simplified Chinese: 赵亚楠) provides a solid knowledge base for understanding connection tracking in Linux. I document here my attempt to reproduce two tests described in [Cloudflare's 04/06/2020 blog post](https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-flows/) on an AWS EC2 instance.

<!-- excerpt-end -->

The max number of entries in the `conntrack` table can be retrieved by:

```console
$ cat /proc/sys/net/netfilter/nf_conntrack_max
32768
```
On my Raspberry Pi, this value is $$61440$$.

The `struct nf_conntrack_tuple` variable defined in [`nf_conntrack_tuple.h`](https://elixir.bootlin.com/linux/latest/source/include/net/netfilter/nf_conntrack_tuple.h) is at the heart of connection tracking:

```c
/* This contains the information to distinguish a connection. */
struct nf_conntrack_tuple {
    struct nf_conntrack_man src;

    /* These are the parts of the tuple which are fixed. */
    struct {
        union nf_inet_addr u3;
        union {
            /* Add other protocols here. */
            __be16 all;

            struct {
                __be16 port;
            } tcp;
            struct {
                __be16 port;
            } udp;
            struct {
                u_int8_t type, code;
            } icmp;
            struct {
                __be16 port;
            } dccp;
            struct {
                __be16 port;
            } sctp;
            struct {
                __be16 key;
            } gre;
        } u;

        /* The protocol. */
        u_int8_t protonum;

        /* The direction (for tuplehash) */
        u_int8_t dir;
    } dst;
};
```

## Test One

First of all, I have this shell script named `conntrack-test-1.bash`:

```bash
ip link set lo up
ip tuntap add name tun0 mode tun
ip link set tun0 up
ip addr add 192.0.2.1 peer 192.0.2.2 dev tun0
ip route add 0.0.0.0/0 via 192.0.2.2 dev tun0

# Refer to conntrack at least once to ensure it's enabled
iptables -t raw -A PREROUTING -j CT
# Create a counter in mangle table
iptables -t mangle -A PREROUTING
# Make sure reverse traffic doesn't affect conntrack state
iptables -t raw -A OUTPUT -p tcp --sport 80 -j DROP

echo "[*] tcpdump"
# Capture packets from all interfaces:
tcpdump -ni any -B 16384 -ttt &
TCPDUMP_PID=$!
# Cleanup on exit:
function finish_tcpdump {
    kill ${TCPDUMP_PID}
    wait
}
trap finish_tcpdump EXIT

sleep 0.3
/usr/bin/python3 send_syn.py

echo "[*] conntrack"
conntrack -L

echo "[*] iptables -t raw"
iptables -nvx -t raw -L PREROUTING
echo "[*] iptables -t mangle"
iptables -nvx -t mangle -L PREROUTING
```

- The first line "`ip link set lo up`" brings the loopback interface up.
- The second line create a [TUN interface](https://docs.kernel.org/networking/tuntap.html) named `tun0` and then the third line brings it up. Because TUN interface is a virtual network device working with IP frames, it acts as a point-to-point link. `192.0.2.1` is the local address for our own machine and `192.0.2.2` is the remote address for Scapy.
- `iptables` is a command-line tool implemented within the Netfilter project that configures IP packet filter rules. It is widely used in service meshes to intercept TCP traffic. Check Michael Kerrisk's Linux manual page [here](https://man7.org/linux/man-pages/man8/iptables.8.html).

Run this shell script with the command:

```bash
unshare --user --net -r bash conntrack-test-1.bash
```

The `unshare` utility creates an isolated environment (user and network namespace with "fake root") where we can precisely control the packets going through and test `iptables` and `conntrack` without tainting the global namespace. Note that rules referencing `conntrack` must exist in the namespace's `iptables` for `conntrack` to be active.

It is interesting that when I reproduced the test on my Raspberry Pi, the `tcpdump` command did not work:

```console
tcpdump: data link type LINUX_SLL2
tcpdump: Couldn't change to 'tcpdump' uid=115 gid=125: Operation not permitted
```

For troubleshooting, my first thought is to take a look at the source code. The error stems from the following part of the `droproot()` function inside [`tcpdump.c`](https://github.com/the-tcpdump-group/tcpdump/blob/master/tcpdump.c):

```c
#ifdef HAVE_LIBCAP_NG
{
    int ret = capng_change_id(pw->pw_uid, pw->pw_gid, CAPNG_NO_FLAG);
    if (ret < 0)
        fprintf(stderr, "error : ret %d\n", ret);
    else
        fprintf(stderr, "dropped privs to %s\n", username);
}
#else
if (initgroups(pw->pw_name, pw->pw_gid) != 0 ||
    setgid(pw->pw_gid) != 0 || setuid(pw->pw_uid) != 0)
    error("Couldn't change to '%.32s' uid=%lu gid=%lu: %s",
        username,
        (unsigned long)pw->pw_uid,
        (unsigned long)pw->pw_gid,
        pcap_strerror(errno));
else {
    fprintf(stderr, "dropped privs to %s\n", username);
}
```

It seems that I do not have the [libcap-ng](https://github.com/stevegrubb/libcap-ng) library installed on my Raspberry Pi. The definitions for `setuid()` and `setgid()` can be found [here](https://www.unix.com/man-page/minix/2/setuid/). The `pw` variable is given by:

```c
// In: https://github.com/the-tcpdump-group/tcpdump/blob/master/tcpdump.c
struct passwd *pw = NULL;

pw = getpwnam(username);

// In: https://elixir.bootlin.com/glibc/glibc-2.26/source/pwd/pwd.h
/* The passwd structure. */
struct passwd
{
    char *pw_name;      /* Username. */
    char *pw_passwd;    /* Password. */
    __uid_t pw_uid;     /* User ID. */
    __gid_t pw_gid;     /* Group ID. */
    char *pw_gecos;     /* Real name. */
    char *pw_dir;       /* Home directory. */
    char *pw_shell;     /* Shell program. */
};
```

According to the Linux manual page:

> The `getpwnam()` function returns a pointer to a structure containing the broken-out fields of the record in the password database (e.g., the local password file /etc/passwd, NIS, and LDAP) that matches the username.

In my case, the username is "`tcpdump`". On the AWS EC2 instance, I can find the following entry for `tcpdump`:

```text
tcpdump:x:72:72::/:/sbin/nologin
```

The following `send_syn.py` program is used to send 10 SYN packets over the `tun0` interface:

```python
import logging
logging.getLogger("scapy.runtime").setLevel(logging.ERROR)

from scapy.all import *

# Remote side of the TUN interface
tun = TunTapInterface("tun0", mode_tun=True)
# tun.open()

for i in range(10000, 10000+10):
    ip = IP(src="198.18.0.2", dst="192.0.2.1")
    tcp = TCP(sport=i, dport=80, flags="S")
    send(ip/tcp, verbose=False, inter=0.01, socket=tun)
```

The original program in Cloudflare's blog post contains the line `tun.open()`. I comment it out since [the returned socket object](https://scapy.readthedocs.io/en/latest/layers/tuntap.html) does not have an `open()` method. Before we run the shell script, we need to create a file:

```bash
sudo touch /run/xtables.lock
```

The output is shown below:

```console
$ unshare --user --net -r bash conntrack-test-1.bash
[*] tcpdump
error : ret -4
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
 00:00:00.000000 IP 198.18.0.2.ndmp > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.003394 IP6 fe80::1990:1fe9:8620:5a6b > ff02::2: ICMP6, router solicitation, length 8
 00:00:00.011175 IP 198.18.0.2.scp-config > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012011 IP 198.18.0.2.documentum > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012000 IP 198.18.0.2.documentum_s > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012007 IP 198.18.0.2.emcrmirccd > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.011977 IP 198.18.0.2.emcrmird > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.011935 IP 198.18.0.2.10006 > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012207 IP 198.18.0.2.mvs-capacity > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.011979 IP 198.18.0.2.octopus > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.011960 IP 198.18.0.2.swdtp-sv > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
[*] conntrack
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10005 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10005 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10004 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10004 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10000 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10000 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10002 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10002 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10001 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10001 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10009 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10009 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10003 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10003 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10008 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10008 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10006 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10006 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10007 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10007 use=1
conntrack v1.4.4 (conntrack-tools): 10 flow entries have been shown.
[*] iptables -t raw
Chain PREROUTING (policy ACCEPT 10 packets, 400 bytes)
    pkts      bytes target     prot opt in     out     source               destination         
      10      400 CT         all  --  *      *       0.0.0.0/0            0.0.0.0/0            CT
[*] iptables -t mangle
Chain PREROUTING (policy ACCEPT 10 packets, 400 bytes)
    pkts      bytes target     prot opt in     out     source               destination         
      10      400            all  --  *      *       0.0.0.0/0            0.0.0.0/0           

11 packets captured
11 packets received by filter
0 packets dropped by kernel
```

I obtained an error: `ret -4`, again. It tells me that the return value of the `capng_change_id()` function call is `-4`. Take a look at the following code:

```c
// In: https://github.com/stevegrubb/libcap-ng/blob/master/src/cap-ng.c
/* Change gid */
if (gid != -1) {
    rc = setresgid(gid, gid, gid);
    if (rc) {
        ret = -4;
        goto err_out;
    }
}
```

So, the problem this time is that the `setresgid()` function failed because the current process was not privileged. According to an email from Dr. Marion Sudvarg:

> The basic idea is that some of the syscalls employed by the `tcpdump` utility require the `CAP_NET_ADMIN` and `CAP_NET_RAW` capabilities. While running `unshare --user -r` does confer root privileges (i.e., all capabilities) to the running user, these are only in the context of the new namespace that has been created. As far as I can tell, the `tcpdump` functionality you're requesting would operate at the global namespace level, so it's the global namespace's user/group ID that are checked for the capability. If this check weren't happening, it would represent a security flaw, as it would allow arbitrary unprivileged users access to network packet information just by creating a new namespace.
> 
> I'm not sure what your use case is for not running in the root user context, but you may be able to get around these restrictions by adding file capabilities to the `tcpdump` executable.
> 
> Run `which tcpdump` to get the path. Then run:
> 
> `sudo setcap cap_net_raw,cap_net_admin=eip /path/to/tcpdump`

```c
// In: include/uapi/linux/capability.h
/* Allow interface configuration */
/* Allow administration of IP firewall, masquerading and accounting */
/* Allow setting debug option on sockets */
/* Allow modification of routing tables */
/* Allow setting arbitrary process / process group ownership on
   sockets */
/* Allow binding to any address for transparent proxying (also via NET_RAW) */
/* Allow setting TOS (type of service) */
/* Allow setting promiscuous mode */
/* Allow clearing driver statistics */
/* Allow multicasting */
/* Allow read/write of device-specific registers */
/* Allow activation of ATM control sockets */

#define CAP_NET_ADMIN        12

/* Allow use of RAW sockets */
/* Allow use of PACKET sockets */
/* Allow binding to any address for transparent proxying (also via NET_ADMIN) */

#define CAP_NET_RAW          13
```

According to [Michael Kerrisk](https://lwn.net/Articles/540087/):

> When a new IPC, mount, network, PID, or UTS namespace is created via `clone()` or `unshare()`, the kernel records the user namespace of the creating process against the new namespace. Whenever a process operates on global resources governed by a namespace, permission checks are performed according to the process's capabilities in the user namespace that the kernel associated with the that namespace.

Alright, let's switch back to the `conntrack` table test. Our question is: Given that the `conntrack` table is size constrained, what exactly happens when it fills up? First, we need to shrink the `conntrack` table size:

```console
$ echo 7 | sudo tee /proc/sys/net/netfilter/nf_conntrack_max
7
```

Then run it again:

```console
$ unshare --user --net -r bash conntrack-test-1.bash
[*] tcpdump
error : ret -4
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
 00:00:00.000000 IP 198.18.0.2.ndmp > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.000250 IP6 fe80::9693:f272:d960:e33f > ff02::2: ICMP6, router solicitation, length 8
 00:00:00.011956 IP 198.18.0.2.scp-config > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012090 IP 198.18.0.2.documentum > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.011931 IP 198.18.0.2.documentum_s > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012031 IP 198.18.0.2.emcrmirccd > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012025 IP 198.18.0.2.emcrmird > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012024 IP 198.18.0.2.10006 > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.012331 IP 198.18.0.2.mvs-capacity > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.017577 IP 198.18.0.2.octopus > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
 00:00:00.018532 IP 198.18.0.2.swdtp-sv > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
[*] conntrack
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10002 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10002 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10004 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10004 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10003 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10003 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10005 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10005 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10000 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10000 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10001 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10001 use=1
tcp      6 119 SYN_SENT src=198.18.0.2 dst=192.0.2.1 sport=10006 dport=80 [UNREPLIED] src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10006 use=1
conntrack v1.4.4 (conntrack-tools): 7 flow entries have been shown.
[*] iptables -t raw
Chain PREROUTING (policy ACCEPT 10 packets, 400 bytes)
    pkts      bytes target     prot opt in     out     source               destination         
      10      400 CT         all  --  *      *       0.0.0.0/0            0.0.0.0/0            CT
[*] iptables -t mangle
Chain PREROUTING (policy ACCEPT 7 packets, 280 bytes)
    pkts      bytes target     prot opt in     out     source               destination         
       7      280            all  --  *      *       0.0.0.0/0            0.0.0.0/0           

11 packets captured
11 packets received by filter
0 packets dropped by kernel
```

The `dmesg` command gives us:
```console
[2682876.561434] tun: Universal TUN/TAP device driver, 1.6
[2682876.599635] bpfilter: Loaded bpfilter_umh pid 31684
[2682876.604757] Started bpfilter
[2682951.736435] IPv6: ADDRCONF(NETDEV_CHANGE): tun0: link becomes ready
[2683336.112744] IPv6: ADDRCONF(NETDEV_CHANGE): tun0: link becomes ready
[2683414.082289] IPv6: ADDRCONF(NETDEV_CHANGE): tun0: link becomes ready
[2683483.867954] IPv6: ADDRCONF(NETDEV_CHANGE): tun0: link becomes ready
[2683509.313225] IPv6: ADDRCONF(NETDEV_CHANGE): tun0: link becomes ready
[2683560.494809] IPv6: ADDRCONF(NETDEV_CHANGE): tun0: link becomes ready
[2683794.216678] IPv6: ADDRCONF(NETDEV_CHANGE): tun0: link becomes ready
[2683794.308623] nf_conntrack: nf_conntrack: table full, dropping packet
[2683794.326200] nf_conntrack: nf_conntrack: table full, dropping packet
[2683794.344731] nf_conntrack: nf_conntrack: table full, dropping packet
```

## Test Two

Our second test script looks like this (make sure the `netcat` utility is installed):

```bash
ip link set lo up
ip tuntap add name tun0 mode tun
ip link set tun0 up
ip addr add 192.0.2.1 peer 192.0.2.2 dev tun0
ip route add 0.0.0.0/0 via 192.0.2.2 dev tun0

iptables -t raw -A PREROUTING -j CT

echo "[*] tcpdump"
nc -k -l 80 &
NC_PID=$!

tcpdump -ni any -B 16384 ip -t &
TCPDUMP_PID=$!
function finish_tcpdump {
    kill ${NC_PID}
    wait ${NC_PID} 2>/dev/null
    kill ${TCPDUMP_PID}
    wait
}
trap finish_tcpdump EXIT

sleep 0.3
/usr/bin/python3 send_syn.py

echo "[*] conntrack"
conntrack -L
```

Run it with the default `conntrack` table size:

```console
$ unshare --user --net -r bash conntrack-test-2.bash
[*] tcpdump
error : ret -4
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
IP 198.18.0.2.ndmp > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.ndmp: Flags [S.], seq 3936458797, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.scp-config > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.scp-config: Flags [S.], seq 4214098727, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.documentum > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.documentum: Flags [S.], seq 395921776, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.documentum_s > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.documentum_s: Flags [S.], seq 1099917875, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.emcrmirccd > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.emcrmirccd: Flags [S.], seq 1723656050, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.emcrmird > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.emcrmird: Flags [S.], seq 94323351, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.10006 > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.10006: Flags [S.], seq 1845890353, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.mvs-capacity > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.mvs-capacity: Flags [S.], seq 2672138941, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.octopus > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.octopus: Flags [S.], seq 135480193, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.swdtp-sv > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.swdtp-sv: Flags [S.], seq 217074585, ack 1, win 64240, options [mss 1460], length 0
[*] conntrack
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10000 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10000 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10009 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10009 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10004 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10004 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10006 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10006 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10001 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10001 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10005 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10005 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10002 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10002 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10003 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10003 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10008 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10008 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10007 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10007 use=1
conntrack v1.4.4 (conntrack-tools): 10 flow entries have been shown.

20 packets captured
20 packets received by filter
0 packets dropped by kernel
```

Again, reduce the `conntrack` table size to seven, and rerun the script:

```console
$ unshare --user --net -r bash conntrack-test-2.bash
[*] tcpdump
error : ret -4
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
IP 198.18.0.2.ndmp > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.ndmp: Flags [S.], seq 233441982, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.scp-config > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.scp-config: Flags [S.], seq 511079863, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.documentum > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.documentum: Flags [S.], seq 987867838, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.documentum_s > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.documentum_s: Flags [S.], seq 1691861748, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.emcrmirccd > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.emcrmirccd: Flags [S.], seq 2315598131, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.emcrmird > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.emcrmird: Flags [S.], seq 686264674, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.10006 > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.10006: Flags [S.], seq 2437830852, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.mvs-capacity > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 192.0.2.1.http > 198.18.0.2.mvs-capacity: Flags [S.], seq 3264080891, ack 1, win 64240, options [mss 1460], length 0
IP 198.18.0.2.octopus > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
IP 198.18.0.2.swdtp-sv > 192.0.2.1.http: Flags [S], seq 0, win 8192, length 0
[*] conntrack
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10004 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10004 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10006 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10006 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10001 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10001 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10007 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10007 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10005 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10005 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10003 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10003 use=1
tcp      6 59 SYN_RECV src=198.18.0.2 dst=192.0.2.1 sport=10002 dport=80 src=192.0.2.1 dst=198.18.0.2 sport=80 dport=10002 use=1
conntrack v1.4.4 (conntrack-tools): 7 flow entries have been shown.

18 packets captured
18 packets received by filter
0 packets dropped by kernel
```

We can see 7 SYN+ACK's flying out of the port 80 listening socket. The final three SYN's go nowhere as they are dropped by the `conntrack` table.

The results here have security implications. Just as Marek Majkowski wrote:

> If you use `conntrack` on publicly accessible ports, during [SYN flood]({{ site.baseurl }}/posts/seedlabs/attacks-on-tcp) mitigation technologies like SYN Cookies won't help. You are still at risk of running out of `conntrack` space and therefore affecting legitimate connections.

## Further Reading

[Jakub Sitnicki, "Conntrack turns a blind eye to dropped SYNs," 03/04/2021](https://blog.cloudflare.com/conntrack-turns-a-blind-eye-to-dropped-syns/):

> Linux connection tracking subsystem processes every received packet (unless we explicitly exclude it from being tracked with `iptables -j NOTRACK`) and creates a flow for it. A flow entry is always created even if the packet is dropped shortly after. The flow might never be promoted to the main `conntrack` table and can be short lived.