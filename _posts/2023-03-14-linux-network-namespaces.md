---
layout:             post
title:              "Linux Network Namespaces"
category:           "Linux System Programming"
tags:               namespace container networking
permalink:          /blog/linux-plumbing/linux-network-namespaces
---

Network namespaces entered the Linux kernel in version 2.6.24. They partition the use of the system resources associated with networking&mdash;network devices, addresses, ports, routes, firewall rules, etc.&mdash;into separate boxes, essentially virtualizing the network within a single running kernel instance[^1].

<!-- excerpt-end -->

Once upon a time, there was a patch set called "[process containers](https://lwn.net/Articles/236038/)". The "container subsystem" allows an administrator to group processes into hierarchies of "containers"; each hierarchy is managed by one or more "subsystems." As of 2.6.23, virtualization is quite well supported on Linux, at least for the x86 architecture. Containers lag a little behind, instead. In 2.6.24, "containers" were renamed "control groups." It makes sense to pair "control groups" with the management of the various namespaces and resource management in general to create a framework for a container implementation[^2].

Current container implementations use network namespaces to give each container its own view of the network, untrammeled by processes outside of the container. The network namespace patches merged for 2.6.24 added a line to `<linux/sched.h>`:

```c
#define CLONE_NEWNET      0x40000000     /* New network namespace */
```

A central structure defined in `<include/net/net_namespace.h>` was used to keep track of all available network namespaces:

```c
struct net {
    atomic_t              count;         /* To decided when the network
                                          * namespace should be freed
                                          */
    atomic_t              use_count;     /* To track references we
                                          * destroy on demand
                                          */
    struct list_head      list;          /* list of network namespaces */
    struct work_struct    work;          /* work struct for freeing */

    struct proc_dir_entry *proc_net;
    struct proc_dir_entry *proc_net_stat;
    struct proc_dir_entry *proc_net_root;

    struct net_device     *loopback_dev; /* The loopback */

    struct list_head      dev_base_head;
    struct hlist_head     *dev_name_head;
    struct hlist_head     *dev_index_head;
};
```

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Establish Communication Between Network Namespaces

Let's first `ssh` into an Amazon EC2 instance and execute `ip link show` or `ip link list` to display the state of all network interfaces on the system:

```console
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 12:57:06:38:46:eb brd ff:ff:ff:ff:ff:ff
```

where `lo` is the loopback interface and `eth0` is the first Ethernet network interface. Now, let's create two network namespaces:

```console
$ sudo ip netns add ns-1
$ sudo ip netns add ns-2
$ ip netns list
ns-2
ns-1
```

Compare the ARP table of the global namespace and our manually created namespaces:

```console
$ arp
Address                  HWtype  HWaddress           Flags Mask            Iface
ip-172-31-80-1.ec2.inte  ether   12:19:04:c5:6c:6d   C                     eth0
$ sudo ip netns exec ns-1 arp # nothing
$ sudo ip netns exec ns-2 arp # nothing
```

To enable communication between these two network namespaces, we can make use of a virtual Ethernet (`veth`) device, which is a local Ethernet tunnel created in interconnected pairs. Let's create a `veth` pair using the following command:

```bash
sudo ip link add veth-1 type veth peer name veth-2
```

Issue the `ip link show` command again to verify that the `veth` pair has been successfully created:

```console
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 12:57:06:38:46:eb brd ff:ff:ff:ff:ff:ff
3: veth-2@veth-1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 6e:ee:6b:30:5f:bf brd ff:ff:ff:ff:ff:ff
4: veth-1@veth-2: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 92:0a:ed:2d:30:a7 brd ff:ff:ff:ff:ff:ff
```

The `veth` pair, at this moment, exists within the global network namespace. We need to place one end of the `veth` pair in `ns-1` and the other end in `ns-2`:

```bash
sudo ip link set veth-1 netns ns-1

sudo ip link set veth-2 netns ns-2
```

Subsequently, the `ip link show` command should only display the initial two network devices. To view the `veth` pair we have just created, execute the `ip link show` command inside the corresponding network namespace, for example:

```console
$ sudo ip netns exec ns-1 ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth-1@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 92:0a:ed:2d:30:a7 brd ff:ff:ff:ff:ff:ff link-netns ns-2
```

Let's assign each `veth` interface a unique IP address and bring it up:

```bash
sudo ip netns exec ns-1 ip addr add 10.0.0.10/16 dev veth-1 && sudo ip netns exec ns-1 ip link set dev veth-1 up

sudo ip netns exec ns-2 ip addr add 10.0.0.20/16 dev veth-2 && sudo ip netns exec ns-2 ip link set dev veth-2 up
```

Now, we can open two separate terminals, one for each network namespace:

```bash
# Terminal 1
sudo ip netns exec ns-1 bash

# Terminal 2
sudo ip netns exec ns-2 bash
```

Execute the following command in Terminal 2 to capture the ICMP traffic using `tcpdump`:

```console
$ sudo ip netns exec ns-2 tcpdump -i veth-2 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth-2, link-type EN10MB (Ethernet), capture size 262144 bytes
```

Then, in Terminal 1, type:

```bash
ping 10.0.0.20
```

We should see in Terminal 2 the following output:

```console
listening on veth-2, link-type EN10MB (Ethernet), capture size 262144 bytes
20:30:26.942779 IP 10.0.0.10 > ip-172-31-95-167.ec2.internal: ICMP echo request, id 3892, seq 1, length 64
20:30:26.942790 IP ip-172-31-95-167.ec2.internal > 10.0.0.10: ICMP echo reply, id 3892, seq 1, length 64
20:30:27.944142 IP 10.0.0.10 > ip-172-31-95-167.ec2.internal: ICMP echo request, id 3892, seq 2, length 64
20:30:27.944162 IP ip-172-31-95-167.ec2.internal > 10.0.0.10: ICMP echo reply, id 3892, seq 2, length 64
20:30:28.956516 IP 10.0.0.10 > ip-172-31-95-167.ec2.internal: ICMP echo request, id 3892, seq 3, length 64
20:30:28.956535 IP ip-172-31-95-167.ec2.internal > 10.0.0.10: ICMP echo reply, id 3892, seq 3, length 64
20:30:29.980557 IP 10.0.0.10 > ip-172-31-95-167.ec2.internal: ICMP echo request, id 3892, seq 4, length 64
20:30:29.980577 IP ip-172-31-95-167.ec2.internal > 10.0.0.10: ICMP echo reply, id 3892, seq 4, length 64
20:30:31.004562 IP 10.0.0.10 > ip-172-31-95-167.ec2.internal: ICMP echo request, id 3892, seq 5, length 64
20:30:31.004582 IP ip-172-31-95-167.ec2.internal > 10.0.0.10: ICMP echo reply, id 3892, seq 5, length 64
```

Inside either of the network namespaces, let's take a look at the current state of the ARP table:

```console
# ip --all netns exec arp
netns: ns-2
Address                  HWtype  HWaddress           Flags Mask            Iface
10.0.0.10                ether   a6:ae:c7:74:56:b8   C                     veth-2

netns: ns-1
Address                  HWtype  HWaddress           Flags Mask            Iface
10.0.0.20                ether   06:22:d8:20:50:0b   C                     veth-1
```

Also, inspect the routing table and packet filtering rules within either of the network namespaces:

```console
$ sudo ip netns exec ns-1 route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.0.0        0.0.0.0         255.255.0.0     U     0      0        0 veth-1
$ sudo ip netns exec ns-1 iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

The `iptables -L` command by default displays the `filter` table, which is, at this moment, empty. An `ACCEPT` target is specified by each chain.

Delete our manually created network namespaces:

```bash
sudo ip netns delete ns-1

sudo ip netns delete ns-2
```

## Footnotes

[^1]: See Jake Edge, "[Namespaces in operation, part 7: Network namespaces](https://lwn.net/Articles/580893/)," January 22, 2014. [This Linux manual page](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) can also serve as a good starting point.

[^2]: See Jonathan Corbet, "[Notes from a container](https://lwn.net/Articles/256389/)," October 29, 2007.