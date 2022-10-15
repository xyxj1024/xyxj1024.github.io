---
layout:           post
title:            "Tree Data Structures in the Linux Kernel"
category:         "Computing Systems"
tags:		      unix operating-system data-structures
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

The Linux kernel documentation for <code>rbtree</code> can be found [here](https://www.kernel.org/doc/html/latest/core-api/rbtree.html). Linux kernel's <code>rbtree</code> representation lives in the file [<code>include/linux/rbtree_types.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree_types.h). Basic utilities can be viewed in: [<code>include/linux/rbtree_augmented.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree_augmented.h) and [<code>include/linux/rbtree.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree.h). To import the red-black tree facility, do:
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

### Virtual Memory Areas in the Kernel Space

Red-black tree suppots virtual memory area tracking in the Linux **kernel space**. Here is a [quote](https://lwn.net/Articles/304188/) from Jonathan Corbet:

> Kernel memory is normally allocated in relatively small chunks - usually just a single page at a time. As the size of an allocation grows, satisfying that allocation with physically-contiguous pages gets progressively harder. So most of the kernel has been written with an eye toward avoiding the use of large, contiguous allocations. There are times, though, when a large memory array needs to be virtually contiguous, but not necessarily physically contiguous. One example is the allocation of space for loadable modules; any given module should live in a single, contiguous address range, but nobody cares how it's laid out in physical RAM.[^4]

Noncontiguous physical memory can be mapped into virtually contiguous memory between <code>VMALLOC_START</code> and <code>VMALLOC_END</code> with the <code>vmalloc()</code> function. This function wraps <code>__vmalloc_node_range()</code> in order to perform memory allocation and return the address of the allocated kernel virtual area (KVA). Note that <code>VMALLOC_START</code> and <code>VMALLOC_END</code> are architecture-specific.

KVAs are represented by two different <code>struct</code>'s at the same time, defined in [<code>include/linux/vmalloc.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/vmalloc.h):
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
Connections between them can be setup using:
```c
static inline void setup_vmalloc_vm_locked(struct vm_struct *vm,
    struct vmap_area *va, unsigned long flags, const void *caller)
{
    vm->flags = flags;
    vm->addr = (void *)va->va_start;
    vm->size = va->va_end - va->va_start;
    vm->caller = caller;
    va->vm = vm;
}
```

Every <code>vmap_area</code> contains a pointer to a corresponding <code>vm_struct</code> and allows operations from two global data structures: red-black tree and doubly linked list. When performing KVA allocation given certain size and alignment, <code>vmalloc()</code> calls <code>insert_vmap_area()</code>:
```c
static void
insert_vmap_area(struct vmap_area *va, 
                 struct rb_root *root, 
                 struct list_head *head)
{
    struct rb_node **link;
    struct rb_node *parent;

    link = find_va_links(va, root, NULL, &parent);
    if (link)
        link_va(va, root, parent, link, head);
}
```
to insert a free block onto the global red-black tree and the allocation is done over free area lookups rather than finding a hole between two busy blocks, [which allows to have lower number of objects representing the free space and thus to have less external fragmentation](https://patchwork.kernel.org/project/linux-mm/cover/20190406183508.25273-1-urezki@gmail.com/). <code>vmap_struct</code>'s are sorted by their <code>va_start</code> fields. Given a virtual memory address <code>addr</code>, a tree lookup probably resembles this:
```c
while (node) {
    struct vmap_area *va;

    va = rb_entry(node, struct vmap_area, rb_node);
    if (addr < va->va_start) {
        node = node->rb_left;
    } else if (addr >= va->va_end) {
        node = node->rb_right;
    } else {
        /* do something */
    }
}
```

### <code>epoll</code> Subsystem

The eventpoll API was first introduced in version 2.5.44 of the Linux kernel to manage I/O events on high loads with $O(1)$ performance. It uses red-black tree to sort all the file descriptors that are added to the eventpoll interface based on their <code>struct file</code> pointers. Inside [the <code>eventpoll</code> structure](https://elixir.bootlin.com/linux/latest/source/fs/eventpoll.c), there is a field called "<code>rbr</code>":
```c
struct rb_root_cached rbr;
```
which is the tree root used to store monitored file descriptors. The <code>rb_root_cached</code> structure represents a augmented tree root that caches the leftmost node of the tree:
```c
struct rb_root_cached {
    struct rb_root rb_root;
    struct rb_node *rb_leftmost;
};
```

Each <code>struct file</code> is linked to a <code>struct epitem</code>, which is treated as a tree node by the eventpoll red-black tree:
```c
struct epitem {
    union {
        struct rn_node rbn;      /* Tree node that links this structure to the eventpoll red-black tree */
        struct rcu_head rcu;     /* Used to free the struct epitem */
    };
    struct list_head     rdllink;
    struct epitem        *next;
    struct epoll_filefd  ffd;
    struct eppoll_entry  *pwqlist;
    struct eventpoll     *ep;    /* The container of this item */
    struct hlist_node    fllink; /* List head used to link this item to the struct file items list */
    struct wakeup_source __rcu *ws;
    struct epoll_event   event;
}
```

To find an <code>epitem</code> given an <code>epoll_filefd</code>, the following function is used:
```c
static struct epitem *ep_find(struct eventpoll *ep, struct file *file, int fd)
{
    int kcmp;
    struct rb_node *rbp;
    struct epitem *epi, *epir = NULL;
    struct epoll_filefd ffd;

    ep_set_ffd(&ffd, file, fd);
    for (rbp = ep->rbr.rb_root.rb_node; rbp; ) {
        epi = rb_entry(rbp, struct epitem, rbn);
        kcmp = ep_cmp_ffd(&ffd, &epi->ffd);
        if (kcmp > 0)
            rbp = rbp->rb_right;
        else if (kcmp < 0)
            rbp = rbp->rb_left;
        else {
            epir = epi;
            break;
        }
	}

    return epir;
}
```

To insert an <code>epitem</code> onto the tree, the following function is used:
```c
static void ep_rbtree_insert(struct eventpoll *ep, struct epitem *epi)
{
    int kcmp;
    struct rb_node **p = &ep->rbr.rb_root.rb_node, *parent = NULL;
    struct epitem *epic;
    bool leftmost = true;

    while (*p) {
        parent = *p;
        epic = rb_entry(parent, struct epitem, rbn);
        kcmp = ep_cmp_ffd(&epi->ffd, &epic->ffd);
        if (kcmp > 0) {
            p = &parent->rb_right;
            leftmost = false;
        } else
            p = &parent->rb_left;
	}
    rb_link_node(&epi->rbn, parent, p);
    rb_insert_color_cached(&epi->rbn, &ep->rbr, leftmost);
}
```

It is more complicated to remove an <code>epitem</code> from the tree because all the associated resources and links should be freed as well. We need to first remove it from the wait queue:
```c
ep_unregister_pollwait(ep, epi);
```
Remove it from the <code>hlist</code> maintained by the file structure pointer <code>file = epi->ffd.file</code>:
```c
spin_lock(&file->f_lock);
to_free = NULL;
head = file->f_ep;
if (head->first == &epi->fllink && !epi->fllink.next) {
    file->f_ep = NULL;
	if (!is_file_epoll(file)) {
	    struct epitems_head *v;
		v = container_of(head, struct epitems_head, epitems);
		if (!smp_load_acquire(&v->next))
		    to_free = v;
	}
}
hlist_del_rcu(&epi->fllink);
spin_unlock(&file->f_lock);
free_ephead(to_free);
```
Then, remove it from the eventpoll tree:
```c
rb_erase_cached(&epi->rbn, &ep->rbr);
```
Finally, remove it from the read list:
```c
write_lock_irq(&ep->lock);
if (ep_is_linked(epi))
    list_del_init(&epi->rdllink);
write_unlock_irq(&ep->lock);
```

## References

[^1]: [Linux Weekly News](lwn.net): "Trees I: Radix Trees" by Jonathan Corbet, March 13, 2006.

[^2]: *XArray* by Matthew Wilcox, [kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt](https://www.kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt).

[^3]: *Examining Linux 2.6 Page-Cache Performance* by Sonny Rao, Dominique Heger, and Steven Pratt, [landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf](https://landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf)

[^4]: [Linux Weekly News](lwn.net): "Reworking <code>vmap()</code>" By Jonathan Corbet, October 21, 2008.