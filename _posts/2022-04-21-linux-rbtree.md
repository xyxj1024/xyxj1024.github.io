---
layout:           post
title:            "Linux Red-Black Tree Data Structure"
category:         "Linux System Programming"
tags:		      linux-kernel tree
permalink:        /blog/linux-plumbing/rbtree
last_modified_at: "2022-12-30"
---

The [previous post]({{ site.baseurl }}/blog/linux-plumbing/radix-tree) gives a broad-brush survey of radix tree data structure in Linux. Another useful tree-based API is red-black tree, which has been particularly crucial for Linux kernel memory management.

A **red-black tree**{: style="color: red"} is a binary search tree which has the following *red-black properties*:
1. Every node is either red or black.
2. Every leaf (<code>NULL</code>) is black.
3. If a node is red, then both of its children are black.
4. Every simple path from a node to a descendant leaf contains the same number of black nodes.

<!-- excerpt-end -->

Each node in the tree contains a value and up to two children; the node's value will be greater than that of all children in the "left" child branch, and less than that of all children in the "right" branch. (Thus, it is possible to serialize a red-black tree by performing a depth-first. left-to-right traversal.) Implementations of the red-black tree algorithms will usually include the **sentinel**{: style="color: red"} nodes as a convenient means of flagging that you have reached a leaf node. They are <code>NULL</code> black nodes of **Property 2**. Formally, a red-black tree with $n$ internal nodes has height at least $\log_{2}(n+1)$ but at most $2\log_{2}(n+1)$.

[Here](http://www.mew.org/~kazu/proj/red-black-tree/) is a pretty decent article on red-black trees written by Kazu Yamamoto with [source code](https://github.com/kazu-yamamoto/llrbtree) in Haskell.

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Kernel Source Code

The Linux kernel documentation for <code>rbtree</code> can be found [here](https://www.kernel.org/doc/html/latest/core-api/rbtree.html). Linux kernel's <code>rbtree</code> representation lives in the file [<code>include/linux/rbtree_types.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree_types.h). Basic utilities can be viewed in: [<code>include/linux/rbtree_augmented.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree_augmented.h) and [<code>include/linux/rbtree.h</code>](https://elixir.bootlin.com/linux/latest/source/include/linux/rbtree.h). To import the red-black tree facility, do:
```c
#include <linux/rbtree.h>
```

Linux kernel's <code>rb_node</code> structure stores two pointers to its children along with information about its parent:
```c
struct rb_node {
    unsigned long  __rb_parent_color;
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
```
The <code>rb_root</code> structure describes a red-black tree by its root node and allows users to include other variable fields:
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

### Insert

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
which initializes the new node and has it pointed by the node already on the tree, and then call:
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
The function <code>__rb_insert()</code> checks and maintains the invariant properties of red-black tree. Whenever a red node becomes a child of another red node, recoloring and/or rotation operations are required. By default, any node being inserted is initially colored red. Mere recoloring is performed when both parent and uncle of the inserted node are red:
```text
<Case 1>:
    gparent(black)
     /          \
 parent(red) uncle(red)
    /
node(red)
```
shoule be turned into
```text
     gparent(red)
     /          \
 parent(black) uncle(black)
    /
node(red)
```
and
```text
<Case 2>:
    gparent(black)
     /         \
uncle(red)   parent(red)
                 \
               node(red)
```
should be turned into
```text
    gparent(red)
    /          \
uncle(black) parent(black)
                 \
               node(red)
```

If uncle node is colored black, rotation is then performed. Consider the case when we have inserted a node that becomes the leftmost child of <code>gparent</code>:
```text
<Case 3>:
   gparent(black)
    /           \
parent(red)  uncle(black)
   /
node(red)
```
We first right rotate to get:
```text
    parent(red)
    /         \
node(red)  gparent(black)
                \
             uncle(black)
```
and immediately perform recoloring:
```text
    parent(black)
    /          \
node(red)   gparent(red)
                 \
              uncle(black)
```
Similarly for the case when we have inserted a node that becomes the rightmost child of <code>gparent</code>:
```text
<Case 4>:
    gparent(black)
     /          \
uncle(black)  parent(red)
                  \
                node(red)
```
we first left rotate to get:
```text
     parent(red)
    /           \
 gparent(black) node(red)
    /
uncle(black)
```
and immediately perform recoloring:
```text
     parent(black)
      /          \
 gparent(red)  node(red)
    /
uncle(black)
```

How about the cases when the node we have inserted, whose uncle node is colored black, is neither the leftmost child nor the rightmost child of <code>gparent</code>? Consider the case:
```text
<Case 5>:
    gparent(black)
    /            \
parent(red)   uncle(black)
    \
  node(red)
     /
(child(black))
```
We have to first left rotate to get:
```text
      gparent(black)
      /            \
  node(red)     uncle(black)
    /
parent(red)
    \
(child(black))
```
and then right rotate and recolor as in Case 3. Similarly for the case:
```text
<Case 6>:
      gparent(black)
      /            \
 uncle(black)    parent(red)
                   /
               node(red)
                   \
               (child(black))
```
We have to first right rotate to get:
```text
      gparent(black)
      /            \
 uncle(black)    node(red)
                     \
                   parent(red)
                     /
               (child(black))
```
and then left rotate and recolor as in Case 4.

For Rust in Linux kernel, the [<code>insert()</code>](https://github.com/Rust-for-Linux/linux/blob/rust/rust/kernel/rbtree.rs) function calls these two C functions with the [<code>bindings</code>](https://lore.kernel.org/lkml/20220802015052.10452-16-ojeda@kernel.org/) crate (C header files are automatically generated by <code>bindgen</code>):
```rust
unsafe { bindings::rb_link_node(node_links, parent, new_link) };
unsafe { bindings::rb_insert_color(node_links, &mut self.root) };
```
Memory safety issues will be isolated within these code blocks marked with <code>unsafe</code>.

### Remove

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

## Virtual Memory Areas in the Kernel Space

Red-black tree suppots virtual memory area tracking in the Linux **kernel space**. Here is a [quote](https://lwn.net/Articles/304188/) from Jonathan Corbet[^1]:

> Kernel memory is normally allocated in relatively small chunks - usually just a single page at a time. As the size of an allocation grows, satisfying that allocation with physically-contiguous pages gets progressively harder. So most of the kernel has been written with an eye toward avoiding the use of large, contiguous allocations. There are times, though, when a large memory array needs to be virtually contiguous, but not necessarily physically contiguous. One example is the allocation of space for loadable modules; any given module should live in a single, contiguous address range, but nobody cares how it's laid out in physical RAM.

Apart from physically contiguous memory blocks that are allocated by the buddy allocator as well as slab allocators (a slab is multiple pages of contiguous physical memory), noncontiguous physical memory can be allocated and mapped into virtually contiguous memory between <code>VMALLOC_START</code> and <code>VMALLOC_END</code> with the <code>vmalloc()</code> function. This function wraps <code>__vmalloc_node_range()</code> in order to perform (large) memory allocation and return the address of the allocated kernel virtual area (KVA). Note that <code>VMALLOC_START</code> and <code>VMALLOC_END</code> are architecture-specific.

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
Connection between <code>vm_struct</code> and <code>vmap_area</code> can be created using:
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
to insert a free block onto the global red-black tree and the allocation is done over free area lookups rather than finding a hole between two busy blocks, [which allows to have lower number of objects representing the free space and thus to have less external fragmentation](https://patchwork.kernel.org/project/linux-mm/cover/20190406183508.25273-1-urezki@gmail.com/). <code>vmap_struct</code>'s are sorted by their <code>va_start</code> fields. Basically, given a virtual memory address <code>addr</code>, a tree lookup resembles this:
```c
while (node) {
    struct vmap_area *va;

    va = rb_entry(node, struct vmap_area, rb_node);
    if (addr < va->va_start) {
        node = node->rb_left;
    } else if (addr >= va->va_end) {
        node = node->rb_right;
    } else {
        /* Do something */
    }
}
```

Memory objects allocated by <code>vmalloc()</code> may be traced by the [kmemleak API (kernel memory leak detector)](https://elixir.bootlin.com/linux/latest/source/mm/kmemleak.c) with
```c
static int kmemleak_enabled = 1;
```
If <code>CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF</code> is set, the function <code>kmemleak_disable()</code> will be called at boot-time and the variable <code>kmemleak_enabled</code> will be set to zero. A <code>kmemleak_object</code> can be accessed through red-black tree operations since the structure contains a <code>rb_node</code> field. Two types of search trees are supported by the kmemleak API depending on whether a memory object is allocated with physical address:
```c
#define OBJECT_PHYS     (1 << 4)

static struct rb_root object_tree_root = RB_ROOT;
static struct rb_root object_phys_tree_root = RB_ROOT;
```
The lookup function looks like this:
```c
static struct kmemleak_object *__lookup_object(unsigned long ptr, int alias, bool is_phys)
{
    struct rb_node *rb = is_phys ? object_phys_tree_root.rb_node : object_tree_root.rb_node;
    unsigned long untagged_ptr = (unsigned long)kasan_reset_tag((void *)ptr);

    while (rb) {
        struct kmemleak_object *object;
        unsigned long untagged_objp;

        object = rb_entry(rb, struct kmemleak_object, rb_node);
        untagged_objp = (unsigned long)kasan_reset_tag((void *)object->pointer);

        if (untagged_ptr < untagged_objp)
            rb = object->rb_node.rb_left;
        else if (untagged_objp + object->size <= untagged_ptr)
            rb = object->rb_node.rb_right;
        else if (untagged_objp == untagged_ptr || alias)
            return object;
        else {
            kmemleak_warn("Found object by alias at 0x%08lx\n", ptr);
            dump_object_info(object);
            break;
        }
    }
    return NULL;
}
```

In his [*Linux Device Drivers, Third Edition*](https://lwn.net/Kernel/LDD3/), Jonathan Corbet said that:

> We should note, however, that use of `vmalloc` is discouraged in most situations. Memory obtained from `vmalloc` is slightly less efficient to work with, and, on some architectures, the amount of address space set aside for `vmalloc` is relatively small. Code that uses `vmalloc` is likely to get a chilly reception if submitted for inclusion in the kernel. If possible, you should work directly with individual pages rather than trying to smooth things over with `vmalloc`.

## Kernel Samepage Merging (KSM)

Systems running virtualized guests can have large number of pages holding identical data, but the kernel has no way to let guests share those pages. By merging identical memory contents, memory de-duplication allows more virtual machines to run concurrently on top of a hypervisor.

KSM is a memory-saving de-duplication feature, enabled by <code>CONFIG_KSM=y</code>, added to the Linux kernel in version 2.6.32. KSM was originally developed as a device driver for use with kernel-based virtual machine (KVM), where it was known as "Kernel Shared Memory". A process which wants to take part in the page sharing regime can open that device and register with an <code>ioctl()</code> function call a portion of its address space with the KSM driver. Once the page sharing mechanism is turned on via another <code>ioctl()</code> call, the kernel will start looking for pages to share. Inside a kernel thread, the KSM driver picks one of the memory regions registered with it and start scanning over it. For each page which is resident in memory, KSM will generate an SHA1 hash of the page's contents. That hash will then be used to look up other pages with the same hash value. If a subsequent <code>memcmp()</code> function call shows that the contents of the pages are truly identical, all processes with a reference to the scanned page will be pointed (in Copy-on-Write mode) to the other one, and the redundant page will be returned to the system. As long as nobody modifies the page, the sharing can continue; once a write operation happens, the page will be copied and the sharing will end[^2].

[The current KSM implementation](https://elixir.bootlin.com/linux/latest/source/mm/ksm.c) has replaced the hash table with two separate red-black trees. Pages tracked by KSM are initially stored in the "unstable tree":

```c
static struct rb_root one_unstable_tree[1] = { RB_ROOT };
static struct rb_root *root_unstable_tree = one_unstable_tree;
```

The term "unstable" means that KSM considers their contents to be volatile. Placement in the tree is determined by a simple <code>memcmp()</code> of the page's contents; essentially, the page is treated as containing a huge number and sorted accordingly. The unstable tree is suitable for finding pages with duplicate contents; a relatively quick traversal of the tree will turn up the only candidates:

```c
static struct ksm_rmap_item *unstable_tree_search_insert(struct ksm_rmap_item *rmap_item,
                                                         struct page *page,
                                                         struct page **tree_pagep)
{
    struct rb_node **new;
    struct rb_root *root;
    struct rb_node *parent = NULL;
    int nid;

    nid = get_kpfn_nid(page_to_pfn(page));
    root = root_unstable_tree + nid;
    new = &root->rb_node;

    while (*new) {
        struct ksm_rmap_item *tree_rmap_item;
        struct page *tree_page;
        int ret;

        cond_resched();
        tree_rmap_item = rb_entry(*new, struct ksm_rmap_item, node);
        tree_page = get_mergeable_page(tree_rmap_item);
        if (!tree_page)
            return NULL;

        /*
         * Don't substitute a ksm page for a forked page.
         */
        if (page == tree_page) {
            put_page(tree_page);
            return NULL;
        }

        ret = memcmp_pages(page, tree_page);

        parent = *new;
        if (ret < 0) {
            put_page(tree_page);
            new = &parent->rb_left;
        } else if (ret > 0) {
            put_page(tree_page);
            new = &parent->rb_right;
        } else if (!ksm_merge_across_nodes &&
            page_to_nid(tree_page) != nid) {
            /*
             * If tree_page has been migrated to another NUMA node,
             * it will be flushed out and put in the right unstable
             * tree next time: only merge with it when across_nodes.
             */
            put_page(tree_page);
            return NULL;
        } else {
            *tree_pagep = tree_page;
            return tree_rmap_item;
        }
    }

    rmap_item->address |= UNSTABLE_FLAG;
    rmap_item->address |= (ksm_scan.seqnr & SEQNR_MASK);
    DO_NUMA(rmap_item->nid = nid);
    rb_link_node(&rmap_item->node, parent, new);
    rb_insert_color(&rmap_item->node, root);

    ksm_pages_unshared++;
    return NULL;
}
```

where the <code>ksm_rmap_item</code> structure is defined as:

```c
struct ksm_rmap_item {
    struct ksm_rmap_item *rmap_list;
    union {
        struct anon_vma *anon_vma;  /* when stable */
#ifdef CONFIG_NUMA
        int nid;                    /* when node of unstable tree */
#endif
    };
    struct mm_struct *mm;
    unsigned long address;          /* + low bits used for flags below */
    unsigned int oldchecksum;       /* when unstable */
    union {
        struct rb_node node;        /* when node of unstable tree */
        struct {                    /* when listed from stable tree */
            struct ksm_stable_node *head;
            struct hlist_node hlist;
        };
    };
};
```

The nature of red-black trees means that search and insertion operations are almost the same thing, so there is little real cost to rebuilding the unstable tree from the beginning every time. Since shared pages are marked read-only, KSM knows that their contents cannot change. Those pages are put into a separate "stable tree." The stable tree is also a red-black tree, but, since pages cannot become misplaced there, it need not be rebuilt regularly. Once a page goes into the stable tree, it stays there until all users have either modified or unmapped it.

```c
struct ksm_stable_node {
    union {
        struct rb_node node;                    /* when node of stable tree */
        struct {                                /* when listed for migration */
            struct list_head *head;
            struct {
                struct hlist_node hlist_dup;    /* linked into the stable_node->hlist with a stable_node chain */
                struct list_head list;          /* linked into migrate_nodes, pending placement in the proper node tree */
            };
        };
    };
    struct hlist_head hlist;                    /* hlist head of rmap_items using this ksm page */
    union {
        unsigned long kpfn;                     /* page frame number of this ksm page */
        unsigned long chain_prune_time;         /* time of the last full garbage collection */
    };
    /*
     * STABLE_NODE_CHAIN can be any negative number in
     * rmap_hlist_len negative range, but better not -1 to be able
     * to reliably detect underflows.
     */
#define STABLE_NODE_CHAIN -1024
    int rmap_hlist_len;
#ifdef CONFIG_NUMA
    int nid;
#endif
};

static struct rb_root one_stable_tree[1] = { RB_ROOT };
static struct rb_root *root_stable_tree = one_stable_tree;

/* Recently migrated nodes of stable tree, pending proper placement */
static LIST_HEAD(migrate_nodes);
```

Due to the extra copy operation, a write to a de-duplicated page and a normal page will incur different access times. In a virtualized environment where both an attacker's virtual machine and a victim's virtual machine might co-exist on the same host machine, the attacker can detect whether a given page is located in the memory of a neighboring virtual machine by loading the same page into its own memory, waiting for some time until the memory de-duplication takes effect, and then writing to that page, i.e., the one that has been loaded into its own memoy: if the page is de-duplicated, writing to it would take longer than writing to a normal page.

## <code>epoll</code> Subsystem

The eventpoll API was first introduced in version 2.5.44 of the Linux kernel to manage I/O events on high loads with $$O(1)$$ performance. It uses red-black tree to sort all the file descriptors that are added to the eventpoll interface based on their <code>struct file</code> pointers. Inside [the <code>eventpoll</code> structure](https://elixir.bootlin.com/linux/latest/source/fs/eventpoll.c), there is a field called "<code>rbr</code>":
```c
struct rb_root_cached rbr;
```
which is the tree root used to store monitored file descriptors. The <code>rb_root_cached</code> structure represents an augmented tree root that caches the leftmost node of the tree:
```c
struct rb_root_cached {
    struct rb_root rb_root;
    struct rb_node *rb_leftmost;
};
```
The use of the <code>rb_root_cached</code> structure makes it convenient to loop through a eventpoll red-black tree as in:
```c
for (rbp = rb_first_cached(&ep->rbr); rbp; rbp = rb_next(rbp)) {
    /* Do something */
}
```
where the function <code>rb_next()</code> returns the in-order successor of the current node:
```c
struct rb_node *rb_next(const struct rb_node *node)
{
    struct rb_node *parent;
    
    if (RB_EMPTY_NODE(node))
        return NULL;

    if (node->rb_right) {
        node = node->rb_right;
        while (node->rb_left)
            node = node->rb_left;
        return (struct rb_node *)node;
    }

    while ((parent = rb_parent(node)) && node == parent->rb_right)
        node = parent;
    
    return parent;
}
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
Remove it from the <code>hlist</code> maintained by the corresponding file structure pointer <code>file = epi->ffd.file</code>:
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

[^1]: [Linux Weekly News](lwn.net): "Reworking <code>vmap()</code>" By Jonathan Corbet, October 21, 2008.

[^2]: [Linux Weekly News](lwn.net): "<code>/dev/ksm</code>: dynamic memory sharing" By Jonathan Corbet, November 12, 2008.