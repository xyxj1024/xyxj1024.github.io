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

### Genradix

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

### Kernel Source Code

Linux kernel's <code>rbtree</code> representation lives in the file [<code>include/linux/rbtree_types.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree_types.h). Basic utilities can be viewed in: [<code>include/linux/rbtree_augmented.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree_augmented.h) and [<code>include/linux/rbtree.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree.h). To import the red-black tree facility, do:
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
To create an empty red-black tree:
```c
#define RB_ROOT (struct rb_root) { NULL, }
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
This latter task can also be done with
```c
static inline void rb_set_black(struct rb_node *rb)
{
    rb->__rb_parent_color |= RB_BLACK;
}
```

#### Insert

Whenver we want to insert a new node onto a <code>rbtree</code>, we should first make a call to function:
```c
static inline void rb_link_node(struct rb_node *node,
                                struct rb_node *parent,
                                struct rb_node **rb_link)
{
    node->__rb_parent_color = (unsigned long)parent;
    node->rb_left = node->rb_right = NULL;

    *rb_link = node;
}
```
which finds the place to insert the node, and then call:
```c
void rb_insert_color(struct rb_node *node, struct rb_root *root)
{
    __rb_insert(node, root, dummy_rotate);
}
```
where <code>dummy_rotate</code> actually does nothing:
```c
static inline void dummy_rotate(struct rb_node *old, struct rb_node *new) {}
```
which inserts the node onto the tree and "recolors" the tree. For Rust in Linux kernel, the [<code>insert()</code> function](https://github.com/Rust-for-Linux/linux/blob/rust/rust/kernel/rbtree.rs) calls these two C functions with the [<code>bindings</code> crate](https://lore.kernel.org/lkml/20220802015052.10452-16-ojeda@kernel.org/) (C header files are automatically generated by <code>bindgen</code>):
```rust
unsafe { bindings::rb_link_node(node_links, parent, new_link) };
unsafe { bindings::rb_insert_color(node_links, &mut self.root) };
```

#### Remove

Whenever we want to remove an existing node, make a call to function:
```c
void rb_erase(struct rb_node *node, struct rb_root *root)
{
    struct rb_node *rebalance;
    rebalance = __rb_erase_augmented(node, root, &dummy_callbacks);
    if (rebalance)
        ____rb_erase_color(rebalance, root, dummy_rotate);
}
```
The [Rust wrappers](https://github.com/Rust-for-Linux/linux/blob/rust/rust/kernel/rbtree.rs) looks like this:
```rust
pub fn remove_node(&mut self, key: &K) -> Option<RBTreeNode<K, V>>
where
    K: Ord,
{
    let mut node = self.find(key)?;

    unsafe { bindings::rb_erase(&mut node.as_mut().links, &mut self.root) };

    Some(RBTreeNode {
        node: unsafe { Box::from_raw(node.as_ptr()) },
    })
}

pub fn remove(&mut self, key: &K) -> Option<V>
where
    K: Ord,
{
    let node = self.remove_node(key)?;
    let RBTreeNode { node } = node;
    let Node {
        links: _,
        key: _,
        value,
    } = *node;
    Some(value)
}
```

### Virtual Memory Areas

Red-black tree suppots virtual memory area tracking in the Linux kernel. Noncontiguous physical memory can be mapped into virtually contiguous memory between <code>VMALLOC_START</code> and <code>VMALLOC_END</code> with the <code>vmalloc()</code> function. This function actually wraps <code>__vmalloc_node_range()</code> in order to perform memory allocation and return the address of the allocated kernel virtual area. Note that <code>VMALLOC_START</code> and <code>VMALLOC_END</code> are architecture-specific.

Virtual memory areas are represented by two different structures at the same time, defined in [<code>include/linux/vmalloc.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/vmalloc.h):
```c
struct vm_struct {
    struct vm_struct    *next;
    void                *addr;
    unsigned long       size;
    unsigned long       flags;
    struct page         **pages;
#ifdef CONFIG_HAVE_ARCH_HUGE_VMALLOC
    unsigned int        page_order;
#endif
    unsigned int        nr_pages;
    phys_addr_t         phys_addr;
    const void          *caller;
};

struct vmap_area {
    unsigned long va_start;
    unsigned long va_end;

    struct rb_node rb_node;
    struct list_head list;

    union {
        unsigned long subtree_max_size; /* in "free" tree */
        struct vm_struct *vm;           /* in "busy" tree */
    }
}
```

Every <code>vmap_area</code> contains a pointer to a corresponding <code>vm_struct</code> and can be accessed by both the red-black tree and doubly linked list data structures.

## References

[^1]: *Trees I: Radix Trees* by Jonathan Corbet, March 13, 2006, [lwn.net/Articles/175432/](https://lwn.net/Articles/175432/).

[^2]: *XArray* by Matthew Wilcox, [kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt](https://www.kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt).

[^3]: *Examining Linux 2.6 Page-Cache Performance* by Sonny Rao, Dominique Heger, and Steven Pratt, [landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf](https://landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf)