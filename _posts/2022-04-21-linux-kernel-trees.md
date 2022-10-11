---
layout:           post
title:            "Tree Data Structures in the Linux Kernel"
category:         "Computing Systems"
tags:		      unix operating-system linux-kernel
permalink:        /linux-trees/
last_modified_at: "2022-10-10"
---

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Radix Tree

Linux's radix tree implementation lives in the file [<code>lib/radix-tree.c</code>](https://elixir.bootlin.com/linux/latest/source/lib/radix-tree.c). To use it, do:

```c
#include <linux/radix-tree.h>
```

### Integer ID Management

idr, ida, Andrew Morton

### XArray

Matthew Wilcox[^2]

## Red-Black Tree

**Definition 1.** A **read-black tree**{: style="color: red"} is a binary search tree which has the following *red-black properties*:
1. Every node is either red or black.
2. Every leaf (<code>NULL</code>) is black.
3. If a node is red, then both of its children are black.
4. Every simple path from a node to a descendant leaf contains the same number of black nodes.

Each node in the tree contains a value and up to two children; the node's value will be greater than that of all children in the "left" child branch, and less than that of all children in the "right" branch. (Thus, it is possible to serialize a red-black tree by performing a depth-first. left-to-right traversal.) Implementations of the red-black tree algorithms will usually include the **sentinel**{: style="color: red"} nodes as a convenient means of flagging that you have reached a leaf node. They are <code>NULL</code> black nodes of **Property 2**. Formally, a red-black tree with $n$ internal nodes has height at least $\log_{2}(n+1)$ but at most $2\log_{2}(n+1)$.

Linux kernel's <code>rbtree</code> representation lives in the file [<code>include/linux/rbtree_types.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree_types.h). Basic utilities can be viewed in: [<code>include/linux/rbtree_augmented.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree_augmented.h) and [<code>include/linux/rbtree.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree.h). To use the red-black tree facility, do:

```c
#include <linux/rbtree.h>
```

A <code>rb_node</code> structure stores two pointers to its children along with information about its parent:
```c
struct rb_node {
    unsigned long  __rb_parent_color;
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
```

The root node structure is useful for tree access (or initialization):
```c
struct rb_root {
    struct rb_node *rb_node;
};
```

The color of a node is indicated by the least significant bit of its <code>__rb_parent_color</code> field:
```c
#define	RB_RED      0
#define	RB_BLACK    1
```
which can be retrieved using:
```c
#define __rb_color(pc)  ((pc) & 1)
#define rb_color(rb)    __rb_color((rb)->__rb_parent_color)
```
The address of the parent is obtained with the help of this macro:
```c
#define rb_parent(r)    ((struct rb_node *)((r)->__rb_parent_color & ~3))
```

A node's <code>__rb_parent_color</code> field can be set given the address of the parent:
```c
static inline void rb_set_parent(struct rb_node *rb, struct rb_node *p)
{
    rb->__rb_parent_color = rb_color(rb) | (unsigned long)p;
}
```
or at the same time assigned with <code>RB_BlACK</code> so that <code>rb</code> is changed into a black node:
```c
static inline void rb_set_parent_color(struct rb_node *rb,
				                       struct rb_node *p, int color)
{
    rb->__rb_parent_color = (unsigned long)p | color;
}
```

## References

[^1]: *Trees I: Radix Trees* by Jonathan Corbet, March 13, 2006, [lwn.net/Articles/175432/](https://lwn.net/Articles/175432/).

[^2]: *XArray* by Matthew Wilcox, [kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt](https://www.kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt).

[^3]: *Examining Linux 2.6 Page-Cache Performance* by Sonny Rao, Dominique Heger, and Steven Pratt, [landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf](https://landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf)