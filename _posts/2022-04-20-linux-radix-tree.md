---
layout:           post
title:            "Linux Radix Tree Data Structure"
category:         "Linux System Programming"
tags:		      linux-kernel tree
permalink:        /blog/linux-plumbing/radix-tree
last_modified_at: "2023-02-18"
---

<p lang="en" class="message"><em>
On SOSP 2015 History Day, Peter J. Denning in his presentation slides listed six patterns he learned from operating systems research &mdash; there is never certainty; occasionally an insight charts a new direction; technology inflection points may trigger avalanches; searching for what works: building, experimenting, tinkering; always in a social network; theory follows practice.
</em></p>

This post digs into Linux's implementation of the **radix tree**{: style="color: red"} API. [This GitBook](https://0xax.gitbook.io/linux-insides/) is absolutely great but covers little about the radix tree. So I decided to set out on my own journey. The most complex and important user of Linux's radix tree API is the **page cache**{: style="color: red"}: every time we look up a page in a file, we consult the corresponding radix tree to see if the page is already in the cache. Besides, both the `proc` pseudo-filesystem and SCTP rely upon a minimalistic version of radix tree called generic radix tree (or `genradix` for short) that allocates one page of entries at a time. `genradix` is not discussed in this post.

<!-- excerpt-end -->

As the first step, our inquiry of radix tree should start with **trie**{: style="color: red"}[^1]. A trie, also known as *prefix tree*, is a binary tree (or generally, a $$k$$-ary tree where $$k$$ is the *radix* or *branching factor*) where the root represents the empty bit sequence and the two children of a node representing sequence $$x$$ represent the extended sequences $$x_{0}$$ and $$x_{1}$$ (or generally, $$x_{0}, x_{1}, \dots , x_{k-1}$$). In this way, a key is not stored at a particular node but is instead represented *bit-by-bit* (or *digit-by-digit*) along some path. Children of a trie node have a common "prefix" of the key associated with that parent node. A trie node may be simply defined as follows in C:

```c
#define TRIE_BASE   (2)

struct trie_node {
    char *              key;
    struct trie_node *  children[TRIE_BASE];
};
```

By storing common prefixes of keys (e.g., in the form of text strings) just once, tries provide a way to reduce redundancy. Consider a tree data structure (e.g., a binary search tree) that is constructed based on the assumption that the keys to be stored are either integers or can be checked in constant time and require a constant space. This time, each node has to store a generic string with unbounded length. The total memory $$S(n)$$ needed to store an $$n$$-node tree will become the sum of all the keys' lengths:

$$\mathbb{E}[S(n)] = \mathbb{E} \biggl[ \sum\limits_{i = 0}^{n - 1}|k_{i}| \biggr] = \sum\limits_{i = 0}^{n - 1} \mathbb{E}[k_{i}] \approx \sum\limits_{i = 0}^{n - 1} L = n \cdot L,$$

where $$k_{i}, i = 0, 1, \dots , n - 1$$ are the keys, $$L$$ is the average length of the strings held by the tree. Quite a lot of memory consumption, isn't it?

Now we know that, in tries, most of the nodes do not store keys and are just hops on a path between a key and the ones that extend it. Although most of these hops are necessary, when we store long words using tries, they tend to produce long chains of internal nodes, each with just one child, which requires further space optimization. This is where radix trees (a.k.a. *radix tries*, a.k.a. *PATRICIA tries*[^2]) come into the picture. According to Wikipedia, in a radix tree, each node that is the only child is merged with its parent, resulting in the fact that the number of children of every internal node is at most the radix $$r$$ of the radix tree, where $$r$$ is a positive integer and a power $$x$$ of $$2$$, having $$x \geq 1$$.

The *depth* of a leaf in a trie, also known as *depth of insertion* or *successful search time*, is the number of internal nodes on the path from the root of the trie to the leaf. It is of particular interest since it provides useful information in many applications. For example, when keys are stored in the leaves of the trie, the depth of a key gives an estimate of the search time for that key in searching and sorting algorithms[^3]. The expected value of the insertion cost for a trie respectively a radix trie built from $$N$$ records with keys from random bit streams is:

$$\log_{2}N + \frac{\gamma}{\log 2} + \frac{1}{2} + \delta^{[T]}(\log_{2} N) + O(N^{-1})$$

respectively

$$\log_{2}N + \frac{\gamma}{\log 2} + \frac{1}{2} + \delta^{[P]}(\log_{2} N) + O(N^{-1})$$

where $$\gamma = .57721 \cdots$$ is Euler's constant; $$\delta^{[T]}(x) = \delta^{[P]}(x)$$ are periodic functions with period $$1$$ and very small amplitude:

$$\delta^{[T]}(x) = \frac{1}{\log 2} \sum\limits_{k \in \mathbb{Z}, k \not = 0} \omega_{k} \cdot \Gamma(-\omega_{k}) e^{2k\pi ix},$$

with $$\omega_{k} = 1 + 2k\pi i / \log 2$$. So the averages are of order $$\log N$$[^4].

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Kernel Source Code

Linux's radix tree implementation lives in the file [<code>lib/radix-tree.c</code>](https://elixir.bootlin.com/linux/latest/source/lib/radix-tree.c). To use it, do:
```c
#include <linux/radix-tree.h>
```

There are two modes for declaring and initializing radix trees. The first form utilizes the following macro:

```c
#define RADIX_TREE(name, mask)  struct radix_tree_root name = RADIX_TREE_INIT(name, mask)
```

to declare and initialize a radix tree with the given `name`. The second form:

```c
struct radix_tree_root my_tree;
INIT_RADIX_TREE(my_tree, gfp_mask);
```

performs the initialization at run time, where:

```c
// In: include/linux/xarray.h
struct xarray {
    spinlock_t      xa_lock;
/* private: The rest of the data structure is not to be used directly. */
    gfp_t           xa_flags;
    void __rcu *    xa_head;
};

static inline void xa_init_flags(struct xarray *xa, gfp_t flags)
{
    spin_lock_init(&xa->xa_lock);
    xa->xa_flags = flags;
    xa->xa_head = NULL;
}

// In: include/linux/radix-tree.h
#define radix_tree_root             xarray
#define radix_tree_node             xa_node

#define INIT_RADIX_TREE(root, mask) xa_init_flags(root, mask)
```

In either case, a `gfp_mask` must be provided to tell the code how memory allocations are to be performed[^5]. Please do note that the macro, variable, and function definitions listed in this post may vary across different Linux kernel versions. A pointer with the `__rcu` prefix is RCU-protected, which means that any dereference of it must be covered by `rcu_read_lock()`, `rcu_read_lock_bh()`, `rcu_read_lock_sched()`, or by the appropriate update-side lock.

It surprised me that something called "[XArray](https://www.kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt)" is actually behind the radix tree data structure right now. In [this email](https://lkml.iu.edu/hypermail/linux/kernel/1810.2/06430.html), kernel developer Matthew Wilcox introduced XArray ("eXtensible Array") for Linux kernel version 4.20 and explained its advantages over radix tree:

> The XArray provides an improved interface to the radix tree data structure, providing locking as part of the API, specifying GFP flags at allocation time, eliminating preloading, less re-walking the tree, more efficient iterations and not exposing RCU-protected pointers to its users.

### Outdated Radix Tree API

What does the original radix tree look like? Out of curiosity, I checked the `radix-tree.h` file in Linux kernel version 4.10.17 and found the following definitions:

```c
struct radix_tree_node {
    unsigned char               shift;          /* Bits remaining in each slot */
    unsigned char               offset;         /* Slot offset in parent */
    unsigned char               count;          /* Total entry count */
    unsigned char               exceptional;    /* Exceptional entry count */
    struct radix_tree_node *    parent;         /* Used when ascending tree */
    void *                      private_data;   /* For tree user */
    union {
        struct list_head        private_list;   /* For tree user */
        struct rcu_head         rcu_head;       /* Used when freeing node */
    };
    void __rcu *                slots[RADIX_TREE_MAP_SIZE];
    unsigned long               tags[RADIX_TREE_MAX_TAGS][RADIX_TREE_TAG_LONGS];
};

struct radix_tree_root {
    gfp_t                           gfp_mask;
    struct radix_tree_node __rcu *  rnode;
};
```

where:

```c
#ifndef RADIX_TREE_MAP_SHIFT
#define RADIX_TREE_MAP_SHIFT	(CONFIG_BASE_SMALL ? 4 : 6)
#endif

#define RADIX_TREE_MAP_SIZE     (1UL << RADIX_TREE_MAP_SHIFT)
```

The longest path of the radix tree is defined as:

```c
#define RADIX_TREE_INDEX_BITS   (8 /* CHAR_BIT */ * sizeof(unsigned long))
#define RADIX_TREE_MAX_PATH     (DIV_ROUND_UP(RADIX_TREE_INDEX_BITS, \
                                              RADIX_TREE_MAP_SHIFT))
```

The height of a fully populated tree is thus `RADIX_TREE_MAX_PATH + 1`. If the root node (`rnode` field of `radix_tree_root`) is a `NULL` pointer, then the radix tree is considered as empty. Typically, each layer (a node and its siblings) of a radix tree contains `1UL << 6` pointers, that is, the length of the `slots` array is $$64$$. Each slot is indexed by a portion of an integer "key" of type `unsigned long` as seen in:

```c
struct radix_tree_iter {
    unsigned long               index;
    unsigned long               next_index;
    unsigned long               tags;
    struct radix_tree_node *    node;
#ifdef CONFIG_RADIX_TREE_MULTIORDER
    unsigned int                shift;
#endif
};
```

Hereinafter, this "key" is referred to as "index." Given the available bits in each slot of a node, the maximum index can be calculated by:

```c
static inline unsigned long shift_maxindex(unsigned int shift)
{
    return (RADIX_TREE_MAP_SIZE << shift) - 1;
}

static inline unsigned long node_maxindex(struct radix_tree_node *node)
{
    return shift_maxindex(node->shift);
}
```

A tree of height $$N$$ may contain any index between $$0$$ and $$64^{N} - 1$$. The `count` field is the count of every non-`NULL` element in the `slots` array whether that is an exceptional entry, a retry entry, a user pointer, a sibling entry or a pointer to the next level of the tree[^6]. For debugging, we can print out relevant informtion about a radix tree node:

```c
#ifndef __KERNEL__
static void dump_node(struct radix_tree_node *node, unsigned long index)
{
    unsigned long i;

    pr_debug("radix node: %p offset %d indices %lu-%lu parent %p tags %lx %lx %lx shift %d count %d exceptional %d\n",
        node, node->offset, index, index | node_maxindex(node),
        node->parent,
        node->tags[0][0], node->tags[1][0], node->tags[2][0],
        node->shift, node->count, node->exceptional);

    for (i = 0; i < RADIX_TREE_MAP_SIZE; i++) {
        unsigned long first = index | (i << node->shift);
        unsigned long last = first | ((1UL << node->shift) - 1);
        void *entry = node->slots[i];
        if (!entry)
            continue;
        if (entry == RADIX_TREE_RETRY) {
            pr_debug("radix retry offset %ld indices %lu-%lu parent %p\n",
                            i, first, last, node);
		} else if (!radix_tree_is_internal_node(entry)) {
            pr_debug("radix entry %p offset %ld indices %lu-%lu parent %p\n",
                            entry, i, first, last, node);
		} else if (is_sibling_entry(node, entry)) {
            pr_debug("radix sblng %p offset %ld indices %lu-%lu parent %p val %p\n",
                            entry, i, first, last, node,
                            *(void **)entry_to_node(entry));
		} else {
            dump_node(entry_to_node(entry), first);
		}
	}
}

static void radix_tree_dump(struct radix_tree_root *root)
{
    pr_debug("radix root: %p rnode %p tags %x\n",
                    root, root->rnode,
                    root->gfp_mask >> __GFP_BITS_SHIFT);
    if (!radix_tree_is_internal_node(root->rnode))
        return;
    dump_node(entry_to_node(root->rnode), 0);
}
#endif
```

Counting the number of items held by a given node is straightforward:

```c
static inline unsigned long
get_slot_offset(struct radix_tree_node *parent, void **slot)
{
    return slot - parent->slots;
}

static inline int
slot_count(struct radix_tree_node *node, void **slot)
{
    int n = 1;
#ifdef CONFIG_RADIX_TREE_MULTIORDER
    void *ptr = node_to_entry(slot);
    unsigned offset = get_slot_offset(node, slot);
    int i;

    for (i = 1; offset + i < RADIX_TREE_MAP_SIZE; i++) {
        if (node->slots[offset + i] != ptr)
            break;
        n++;
    }
#endif
    return n;
}
```

A radix tree iterator (the `radix_tree_iter` structure shown above) works in terms of "chunks" of slots. A chunk is a sub-interval of slots contained within one radix tree leaf node. It is described by a pointer to its first slot and a `struct radix_tree_iter` which holds the chunk's position in the tree and its size. The chunk size is calculated by:

```c
static __always_inline long
radix_tree_chunk_size(struct radix_tree_iter *iter)
{
    return (iter->next_index - iter->index) >> iter_shift(iter);
}
```

If we are aware of the address of a parent node, the following function returns the offset to its descendant and at the same time updates the node pointer that points to the address of that descendant:

```c
static unsigned int
radix_tree_descend(struct radix_tree_node *parent,
                   struct radix_tree_node **nodep,
                   unsigned long index)
{
    unsigned int offset = (index >> parent->shift) & RADIX_TREE_MAP_MASK;
    void **entry = rcu_dereference_raw(parent->slots[offset]);

#ifdef CONFIG_RADIX_TREE_MULTIORDER
    if (radix_tree_is_internal_node(entry)) {
        if (is_sibling_entry(parent, entry)) {
            void **sibentry = (void **) entry_to_node(entry);
            offset = get_slot_offset(parent, sibentry);
            entry = rcu_dereference_raw(*sibentry);
        }
    }
#endif

    *nodep = (void *)entry;
    return offset;
}
```

This `radix_tree_descend` function is then used to perform tree lookup:

```c
void *__radix_tree_lookup(struct radix_tree_root *root, unsigned long index,
                          struct radix_tree_node **nodep, void ***slotp)
{
    struct radix_tree_node *node, *parent;
    unsigned long maxindex;
    void **slot;

restart:
    parent = NULL;
    slot = (void **)&root->rnode;
    radix_tree_load_root(root, &node, &maxindex);
    if (index > maxindex)
        return NULL;

    while (radix_tree_is_internal_node(node)) {
        unsigned offset;

        if (node == RADIX_TREE_RETRY)
            goto restart;
        parent = entry_to_node(node);
        offset = radix_tree_descend(parent, &node, index);
        slot = parent->slots + offset;
    }

    if (nodep)
        *nodep = parent;
    if (slotp)
        *slotp = slot;
    return node;
}
```

Both the node pointer that points to the address of the parent and the slot pointer that points to the address of the `slots` array are updated before returning the node pointer that points to the node associated with the given index.

The third slide of Matthew Wilcox's [presentation](https://lca-kernel.ozlabs.org/2018-Wilcox-Replacing-the-Radix-Tree.pdf) during the 2018 linux.conf.au Kernel miniconf summarized main characteristics of the Linux kernel's radix tree implementation[^7]:
- Implicit keys (like a trie), but not a bitwise trie
- Grow/shrink, but never rebalanced
- RCU-safe

He described the radix tree as a "great data structure but really hard to use."

### XArray API

Similar to the original radix tree, an XArray interprets each entry (or slot) based on its bottom two bits: `00` (pointer entry); `10` (internal entry); `x1` (value entry or tagged pointer). Most internal entries are pointers to the next node in the XArray, except the following: `0`-`62` (sibling entries); `256` (retry entry); `257` (zero entry). To determine if an entry is a value:

```c
static inline bool xa_is_value(const void *entry)
{
    return (unsigned long)entry & 1;
}
```

To determine if an entry is internal:

```c
static inline bool xa_is_internal(const void *entry)
{
    return ((unsigned long)entry & 3) == 2;
}
```

## Integer ID Management

idr, ida, Andrew Morton

assoc_array

[^8]

IPv6 route lookup

## References

[^1]: The name *trie* comes from the phrase "information re*trie*val." Despite the etymology, trie is now almost always pronounced like *try* instead of *tree* to avoid confusion with other tree data structures. See [course notes](http://www.cs.yale.edu/homes/aspnes/classes/223/notes.html) written by Dr. James Aspnes from Yale University. Another good source for trie-based data structures is Peter Brass, *Advanced Data Structures*, Cambridge University Press, 2008.

[^2]: The name *patricia* comes from the phrase "*p*ractical *a*lgorithm to re*tr*ieve *i*nformation *c*oded *i*n *a*lphanumeric."

[^3]: See Donald E. Knuth, *The Art of Computer Programming. Sorting and Searching*, Vol. III, Addison-Wesley, Reading, MA, 1973.

[^4]: Peter Kirschenhofer and Helmut Prodinger, "Further Results on Digital Search Trees," *Theoretical Computer Science*, Volume 58, Issues 1-3, June 1988, Pages 143-154.

[^5]: See [Linux Weekly News](https://www.lwn.net): "Trees I: Radix Trees" by Jonathan Corbet, March 13, 2006.

[^6]: Most users of the radix tree store pointers but `shmem`/`tmpfs` stores swap entries in the same tree. They are marked as exceptional entries to distinguish them from pointers to `struct page`. The internal entry may be a pointer to the next level in the tree, a sibling entry, or an indicator that the entry in this slot has been moved to another location in the tree and the lookup should be restarted. Sibling slots point directly to another `slot` in the same node. The bottom two bits of the slot determine how the remaining bits in the slot are interpreted: `00` (data pointer); `01` (internal entry); `10` (exceptional entry); `11` (unused/reserved).

[^7]: The video recording is [here](https://archive.org/details/lca2018-The_design_and_implementation_of_the_XArray). In an LWN article, Jonathan Corbet added that addition of an item to a tree has been called "insertion" for decades (since at least 1968), but an "insert" operation does not really describe what happens with a radix tree, especially if an item with the given key is already present there. See [Linux Weekly News](lwn.net): "The XArray Data Structure" by Jonathan Corbet, January 24, 2018.

[^8]: *Examining Linux 2.6 Page-Cache Performance* by Sonny Rao, Dominique Heger, and Steven Pratt, [landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf](https://landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf)