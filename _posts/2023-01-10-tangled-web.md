---
layout:             post
title:              Reading Michal Zalewski's "The Tangled Web"
category:           "Computing Systems"
tags:               network-security internet
permalink:          /tangled-web/
last_modified_at:   "2023-01-10"
---

<p class="message">
<em>Security seems like an intuitive concept, but in the world of computing, it escapes all attempts to usefully define it. Sure, we can restate the problem in catchy yet largely unhelpful ways, but you know there's a problem when one of the definitions most frequently cited by practitioners is this:</em>
    <br />
    A system is secure if it behaves precisely in the manner intended&mdash;and does nothing more.
    <br />
<em>This definition is neat and vaguely outlines an abstract goal, but it tells very little about how to achieve it. It's computer science, but in terms of specificity, it bears a striking resemblance to a poem by Victor Hugo:</em>
    <br />
    Love is a portion of the soul itself, and it is of the same nature as the celestial breathing of the atmosphere of paradise.
</p>

Some words and thoughts taken from [Michal Zalewski, *The Tangled Web*, No Starch Press, 2011](https://lcamtuf.coredump.cx/tangled/). Footnotes are my own.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Part I. Anatomy of the Web

In the traditional model followed by virtually all personal computers over the last 15 years or so, there are very clear boundaries between high-level data objects (documents), user-level code (applications), and the operating system kernel that arbitrates all cross-application communications and hardware input/output (I/O) and enforces configurable security rules should an application go rogue.

In the browser world, this separation is virtually nonexistent: Documents and code live as parts of the same intermingled blobs of HTML, isolation between completely unrelated applications is partial at best (with all sites nominally sharing a global JavaScript environment), and many types of interaction between sites are implicitly permitted with few, if any, flexible, browser-level security arbitration frameworks. In a sense, the model is reminiscent of CP/M, DOS, and other principally nonmultitasking operating systems with no rebost memory protection, CPU preemption, or multiuser features. The obvious difference is that few users depended on these early operating systems to simultaneously run multiple untrusted, attacker-supplied applications, so there was no particular reason for alarm.

In a traditional client-server model with well-specified APIs, one can easily evaluate a server's behavior without looking at the client, and vice versa. Moreover, within each of these components, it is possible to easily isolate smaller functional blocks and make assumptions about their intended operation. With the new model, coupled with the opaque, one-off application APIs common on the Web, these analytical tools, and the resulting ease of reasoning about the security of a system, have been brutally taken away.

## Footnotes