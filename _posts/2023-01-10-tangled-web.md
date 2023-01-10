---
layout:             post
title:              Reading Michal Zalewski's "The Tangled Web"
category:           "Computing Systems"
tags:               network-security internet
permalink:          /tangled-web/
last_modified_at:   "2023-01-10"
---

<p class="message"><em>
<em>Security seems like an intuitive concept, but in the world of computing, it escapes all attempts to usefully define it. Sure, we can restate the problem in catchy yet largely unhelpful ways, but you know there's a problem when one of the definitions most frequently cited by practitioners is this:</em>
    <br />
    A system is secure if it behaves precisely in the manner intended&mdash;and does nothing more.
    <br />
<em>This definition is neat and vaguely outlines an abstract goal, but it tells very little about how to achieve it. It's computer science, but in terms of specificity, it bears a striking resemblance to a poem by Victor Hugo:</em>
    <br />
    Love is a portion of the soul itself, and it is of the same nature as the celestial breathing of the atmosphere of paradise.
</em></p>

Some words and thoughts taken from [Michal Zalewski, *The Tangled Web*, No Starch Press, 2011](https://lcamtuf.coredump.cx/tangled/). Footnotes are my own.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Security in the World of Web Applications

In the traditional model followed by virtually all personal computers over the last 15 years or so, there are very clear boundaries between high-level data objects (documents), user-level code (applications), and the operating system kernel that arbitrates all cross-application communications and hardware input/output (I/O) and enforces configurable security rules should an application go rogue.

## Footnotes