---
layout:     post
title:      "Observing Containerized Envoy with bpftrace Programs: Part 1"
category:   "Cloud Native Computing"
tags:       envoy-proxy bpf container docker
permalink:  /blog/observing-containerized-envoy-part-1
---

Recently, I have been learning the basics of [Envoy](https://github.com/envoyproxy/envoy), which is a high-performance, open source, application-level service proxy written in C++. Envoy was developed at Lyft, where large distributed systems problem needed to be overcome. Envoy was not originally intended as an edge proxy, but was designed to be deployed as a [sidecar](https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar). [The union of performance, extensibility, and dynamic configurability](https://blog.envoyproxy.io/the-universal-data-plane-api-d15cec7a) has made Envoy the universal data plane in cloud native architectures for application/L7 networking solutions[^1].

<!-- excerpt-end -->

Since I'm using [my MacBook Air](https://support.apple.com/kb/sp714?locale=en_US) instead of a Linux machine, [Docker Compose](https://docs.docker.com/compose/) appears to be a natural choice to run those demos. When debugging containerized Envoy proxies, it might be helpful for someone like me who is not familiar with Envoy source code to observe system-level details at runtime. The [eBPF](https://ebpf.io/) approach is championed by many to improve network observability. However, the environment configuration is not a trivial task.

In this post, I would like to share my attempt to deploy bpftrace programs that monitors an Envoy proxy inside a *single* Docker container. [bpftrace](https://github.com/iovisor/bpftrace) is a high-level tracing language and runtime for Linux based on eBPF. It supports static and dynamic tracing for both the kernel- and user-space.

I have [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) for Mac with Intel chip installed. If I issue the `docker info` command in the terminal, I can find the following lines:

```console
...
Kernel Version: 5.15.49-linuxkit
Operating System: Docker Desktop
OSType: linux
Architecture: x86_64
CPUs: 2
Total Memory: 3.842GiB
Name: docker-desktop
...
```

Dominic White in [his May 3, 2021 tweet](https://twitter.com/singe/status/1389224943435620360?s=20) provided a setup guide for playing with eBPF on Docker Desktop for Mac. The GitHub repo is [here](https://github.com/singe/ebpf-docker-for-mac). The provided example loads a simple python program [`hello_world.py`](https://gist.github.com/lizrice/47ad44a15cce912502f8667a403f5649) into the container, which, as being executed, prints out a trace line "Hello World" every time the `clone()` system call is called. The contents of the Dockerfile resemble what we need to build external kernel modules, based on a LinuxKit kernel image that matches the kernel version of Docker Desktop. Specifically, the requirements include:
- Kernel source;
- kernel development headers available in the `linuxkit/kernel` image as `kernel-dev.tar`;
- OS with sources and compiler.

In my case, the first few lines of Dockerfile should look like:

```console
# Please go to: https://hub.docker.com/r/docker/for-desktop-kernel/tags
FROM docker/for-desktop-kernel:5.15.49-13422a825f833d125942948cf8a8688cef721ead AS ksrc

FROM ubuntu:latest

# Please go to: https://github.com/linuxkit/linuxkit/blob/master/docs/kernels.md
COPY --from=ksrc /kernel-dev.tar /
RUN tar xf kernel-dev.tar && rm kernel-dev.tar
```

Moreover, before we copy the Python program, we have to install Ubuntu's `python3-bpfcc` package and the `kmod` package[^2]:

```console
RUN apt-get update
RUN apt install -y kmod python3-bpfcc
```

It shoule be noted that only privileged users could load eBPF programs right now. Re-enable this ability for unprivileged users with the following command:

```bash
sudo sysctl kernel.unprivileged_bpf_disabled=0
```

Finally, we need to mount the [debug filesystem](https://www.kernel.org/doc/Documentation/filesystems/debugfs.txt), since it is required by BCC but not mounted by default on Docker Desktop for Mac:

```bash
mount -t debugfs debugfs /sys/kernel/debug
```

If our eBPF program is attached to tracepoints, the `bpf_<ID>` string[^3] is written to the appropriate `filter` file in the tracing debug filesystem. In this "Hello World" example, the `attach_kprobe()` method is used, which has the following prototype:

```python
BPF.attach_kprobe(event="event", fn_name="name")
```

The method instruments the kernel function `event()` using kernel dynamic tracing of the function entry, and attaches the `name()` function (defined in C inside the Python program) to be called when the kernel function is called.

Nevertheless, the above procedure does not suffice if we want to harness the full power of eBPF. [This Alibaba Cloud blog post](https://developer.aliyun.com/article/798714) gave a detailed account of how to configure a Dockerfile that builds BCC and bpftrace with all dependencies properly installed. Although the setup does not work for my case, I can make modifications on top of it.

The Dockerfile I used is shown below:

```console
# Stage: kernel
FROM docker/for-desktop-kernel:5.15.49-13422a825f833d125942948cf8a8688cef721ead AS ksrc

# Stage: OS
FROM ubuntu:20.04 AS bpftrace

# Stage: Envoy
FROM envoyproxy/envoy:v1.25-latest

## Kernel headers
COPY --from=ksrc /kernel-dev.tar /
RUN tar xf kernel-dev.tar && rm kernel-dev.tar

## Use Alibaba Cloud mirror for ubuntu
RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/' /etc/apt/sources.list

## Install LLVM
RUN apt-get update && \
    apt-get upgrade && \
    apt-get dist-upgrade && \
    apt-get autoremove && \
    apt-get clean && \
    apt-get install -y zip wget lsb-release software-properties-common && \
    wget https://apt.llvm.org/llvm.sh && \
    chmod +x llvm.sh && \
    ./llvm.sh 12
ENV PATH "$PATH:/usr/lib/llvm-12/bin"

## Install bpftrace
RUN apt-get install -y bpftrace

## Install bcc
WORKDIR /root
RUN apt-get install -y bison build-essential cmake flex git curl vim kmod libedit-dev \
    libllvm12 llvm-12-dev libclang-12-dev zlib1g-dev libelf-dev libfl-dev python3-distutils python3-bpfcc
RUN git clone https://github.com/iovisor/bcc.git && \
    mkdir bcc/build && \
    cd bcc/build && \
    cmake -DENABLE_LLVM_SHARED=1 .. && \
    make && \
    make install && \
    cmake -DPYTHON_CMD=python3 .. && \
    cd src/python/ && \
    make && \
    make install && \
    sed -i "s/self._syscall_prefixes\[0\]/self._syscall_prefixes\[1\]/g" /usr/lib/python3/dist-packages/bcc/__init__.py

COPY trace-envoy-socket.bt /root
COPY envoy-demo.yaml /etc/envoy/envoy-config.yaml
RUN chmod go+r /etc/envoy/envoy-config.yaml
```

I also created the following shell script to automate build and run:

```bash
#!/bin/sh

docker volume create --driver local --opt type=debugfs --opt device=debugfs debugfs
docker build -t ebpf-envoy:v1 -f ./Dockerfile.tools --build-arg OS_TAG=20.04 .
docker run --rm -it \
    --privileged \
    --cap-add=ALL \
    --pid=host \
    --env BPFTRACE_STRLEN=200 \
    --name ebpf-envoy \
    -v /lib/modules:/lib/modules:ro \
    -v debugfs:/sys/kernel/debug:rw \
    ebpf-envoy:v1
```

The Docker image might take 10-20 minutes to build.

[^1]: Organizations that run a large, distributed architecture (service-oriented architecture) are beginning to deploy Envoy on a large scale. [This Dropbox tech blog](https://dropbox.tech/infrastructure/how-we-migrated-dropbox-from-nginx-to-envoy), for example, reflected in great details upon the considerations involved when upgrading Dropbox's Nginx-based traffic architecture to Envoy.

[^2]: The former is [the Python3 wrappers for BPF Compiler Collection (BCC)](https://packages.ubuntu.com/bionic/python3-bpfcc). [BCC](https://github.com/iovisor/bcc) is a toolkit that makes use of eBPF VM and provides tools for BPF-based Linux I/O analysis, networking, monitoring, and more. The latter is a [toolkit for managing Linux kernel modukles](https://packages.ubuntu.com/bionic/kmod).

[^3]: `ID` is the ID number of a loaded eBPF program.