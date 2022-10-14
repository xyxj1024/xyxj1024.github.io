---
layout:       post
title:        "Defeating NX: The Return-to-libc Method"
category:     "Computing Systems"
tags:         system-security aslr nx
permalink:    /ret2libc/
---

In "[Defeating ASLR: The Return-to-pop Method](xingjianxuanyuan.github.io/ret2pop)", I constructed a payload that used the <code>ret</code> and <code>pop-ret</code> instructions to inject shellcode into our vulnerable program with **ASLR**{: style="color: red"} enabled but non-executable stack (hereinafter "**NX**{: style="color: red"}") disabled. The payload worked just as expected because we knew that the memory space between <code>ans_buf</code> and the address we were returning to is larger than our shellcode. What if the memory space is not large enough? More importantly, what if we do not have a shellcode at our disposal? In this post, I would like to describe an exploitation technique called "**return-to-libc**{: style="color: red"}", which makes use of existing code (the C standard library) to spawn a shell. The documentation is organized into two parts: for the first part, the ASLR protection is turned off; and for the second part, both ASLR and NX are enabled.

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />