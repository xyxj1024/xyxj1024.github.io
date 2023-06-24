---
layout:             post
title:              "Linux Network Namespaces"
category:           "Linux System Programming"
tags:               namespace container networking
permalink:          /posts/linux-plumbing/linux-network-namespaces
---

Network namespaces entered the Linux kernel in version 2.6.24. They partition the use of the system resources associated with networking&mdash;network devices, addresses, ports, routes, firewall rules, etc.&mdash;into separate boxes, essentially virtualizing the network within a single running kernel instance. Container implementations use network namespaces to give each container its own view of the network, untrammeled by processes outside of the container[^1].

<!-- excerpt-end -->

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

Delete the network namespaces:

```bash
sudo netns delete ns-1

sudo netns delete ns-2
```

```console
$ sudo ip netns exec ns-1 route

$ sudo ip netns exec ns-1 iptables -L
```

Network devices called *bridges* (or synonymously, *switches*[^2]) connect two or more physical Ethernet segments together to form one bigger (logical) Ethernet segment[^3]. The job of bridges is to examine the destination of the data packets one at a time and decide whether or not to pass the packets to the other side of the Ethernet. The Linux bridging module decides whether to bridge data or to drop it not by looking at the protocol type, but by looking at the MAC address unique to each NIC[^4]. Unlike routers that understand Layer-3 network protocols, bridges understand Layer-2 protocols and therefore copy data frame by frame, instead of bit by bit.

## Footnotes

[^1]: See Jake Edge, "[Namespaces in operation, part 7: Network namespaces](https://lwn.net/Articles/580893/)," January 22, 2014. [This Linux manual page](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) can also serve as a good starting point.

[^2]: People might refer to a bridge device using the term *bridge* when the device is equipped with only two ports, although the term *switch* is more commonly used. See Christian Benvenuti, "Chapter 14: Bridging" in *Understanding Linux Network Internals*, p.320, O'Reilly Media, 2005. Or, to put it alternatively, [a linux bridge implements a Layer-2 switch in software](https://paulgorman.org/technical/linux-bridges-and-virtual-networking.txt.html).

[^3]: See the [official documentation](https://wiki.linuxfoundation.org/networking/bridge) for Linux bridging.

[^4]: See [Linux BRIDGE-STP-HOWTO](https://tldp.org/HOWTO/BRIDGE-STP-HOWTO/what-is-a-bridge.html).