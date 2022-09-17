---
layout: post
title: "Cloud Operating Systems"
category: "Computing Systems"
---

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

David Wentzlaff et al., "An Operating System for Multicore and Clouds: Mechanisms and Implementation," *SoCC'10*, June 10-11, 2010, Indianapolis, Indiana, USA.

Cloud computing infrastructure and multicore processors present many common challenges with respect to the operating system.

Ricardo Koller and Dan Williams, "Will Serverless End the Dominance of Linux in the Cloud?" *HotOS'17*, May 08-10, 2017, Whistler, BC, Canada.

The basic idea behind severless is that users upload code that is associated with an event in the cloud (e.g., from a data store or other service). The code, called a lambda, or an *action*, is only run when the event happens and billed on a 100 ms granularity; the user does not need to pay for a server to wait idly for events (hence "serverless").

Eli Cortez et al., "Resource Central: Understanding and Predicting Workloads for Improved Resource Management in Large Cloud Platforms," *SOSP'17*, October 28, 2017, Shanghai, China.

Azure hosts both first- and third-party workloads, split into IaaS (52%) and PaaS (48%) VMs. In Azure's offering, PaaS defines functional roles for VMs, e.g. Web server VM, worker VM. The first-party workloads comprise internal VMs (e.g., research/development, infrastructure management) and first-party services (e.g., communication, gaming, data management) offered to third-party customers. We characterize Azure's entire VM workload over three months, from November 16, 2016 to February 16, 2017, including third-party VMs. VM workloads fundamentally differ from bare-metal container workloads. Mainly for security reasons, public cloud providers must encapsulate their customers' workloads using VMs. Unfortunately, VMs impose higher creation/termination overheads than containers running on bare metal, so they are likely to live longer, produce lower resource utilization, and be deployed in smaller numbers.