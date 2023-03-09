---
layout:       post
title:        "SEED Labs 2.0: TCP/IP Attack Lab Writeup"
category:     "Computing Systems, Systems Security"
tags:         networking cybersecurity
permalink:    /posts/seedlabs/tcp-ip
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org/Labs_20.04/Networking/TCP_Attacks/). The lab assignment was conducted using Docker Compose. Download the [`Labsetup.zip`](https://seedsecuritylabs.org/Labs_20.04/Files/TCP_Attacks/Labsetup.zip) file, unzip it, enter the `Labsetup` folder, and use the `docker-compose.yml` file to set up the lab environment.

<!-- excerpt-end -->

Any operating system that supports networking has some type of network stack.

![linux-network-stack](/assets/images/lns.png){:class="img-responsive"}
<p style="text-align:center;color:gray;font-size:80%;">
Source: <span><a href="https://www.usenix.org/conference/srecon22apac/presentation/jiang">Jizhong Jiang and Shane Xie, Alibaba Cloud</a></span>
</p>

![tcp-socket-listening](/assets/images/synq-acceptq.png){:class="img-responsive"}
<p style="text-align:center;color:gray;font-size:80%;">
Source: <span><a href="http://arthurchiao.art/blog/tcp-listen-a-tale-of-two-queues/">Arthur Chiao's Blog</a></span>
</p>

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Task 1: SYN Flooding Attack

## Notes

[Linux kernel map](https://makelinux.github.io/kernel/map/)

[Why we use the Linux kernel's TCP stack](https://blog.cloudflare.com/why-we-use-the-linux-kernels-tcp-stack/)