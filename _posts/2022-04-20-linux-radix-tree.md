---
layout:           post
title:            "Linux Radix Tree Data Structure"
category:         "Computing Systems, Systems Security"
tags:		      operating-system tree
permalink:        /linux-radix-tree/
last_modified_at: "2022-12-28"
---

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

Linux's radix tree implementation lives in the file [<code>lib/radix-tree.c</code>](https://elixir.bootlin.com/linux/latest/source/lib/radix-tree.c). To use it, do:
```c
#include <linux/radix-tree.h>
```

[^1]

## Genradix

## Integer ID Management

idr, ida, Andrew Morton

## XArray

Matthew Wilcox[^2]

[^3]

## References

[^1]: [Linux Weekly News](lwn.net): "Trees I: Radix Trees" by Jonathan Corbet, March 13, 2006.

[^2]: *XArray* by Matthew Wilcox, [kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt](https://www.kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt).

[^3]: *Examining Linux 2.6 Page-Cache Performance* by Sonny Rao, Dominique Heger, and Steven Pratt, [landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf](https://landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf)