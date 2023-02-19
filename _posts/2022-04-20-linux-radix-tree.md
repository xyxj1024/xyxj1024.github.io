---
layout:           post
title:            "Linux Radix Tree Data Structure"
category:         "Computing Systems, Systems Security"
tags:		      linux-kernel tree
permalink:        /posts/linux-plumbing/radix-tree
last_modified_at: "2023-02-18"
---

<p lang="en" class="message"><em>
On SOSP 2015 History Day, Peter J. Denning's presentation slides page 39 lists six patterns he learned from OS research &mdash; there is never certainty; occasionally an insight charts a new direction; technology inflection points may trigger avalanches; searching for what works: building, experimenting, tinkering; always in a social network; theory follows practice.
</em></p>

The **radix tree**{: style="color: red"} API is used considerably in Linux. As the first step, our inquiry of radix trees should start with **trie**{: style="color: red"}[^1]. A trie is a binary tree (or generally, a $$k$$-ary tree where $$k$$ is the radix) where the root represents the empty bit sequence and the two children of a node representing sequence $$x$$ represent the extended sequences $$x_{0}$$ and $$x_{1}$$ (or generally, $$x_{0}, x_{1}, \dots , x_{k-1}$$). In this way, a key is not stored at a particular node but is instead represented **bit-by-bit** (or digit-by-digit) along some path. Children of a trie node have a common "prefix" of the key associated with that parent node. A trie node may be defined as follows in C:

```c
#define TRIE_BASE   (2)

struct trie_node {
    char *key;
    struct trie_node *children[TRIE_BASE];
};
```

<!-- excerpt-end -->

Consider a tree data structure (e.g., a binary search tree) that is constructed based on the assumption that the keys to be stored are either integers or can be checked in constant time and require a constant space. This time, each node has to store a generic string with unbounded length. The total memory $$S(n)$$ needed to store an $$n$$-node tree will become the sum of all the keys' lengths:

$$\mathbb{E}[S(n)] = \mathbb{E} \big[ \sum\limits_{i = 0}^{n - 1}|k_{i}| \big] = \sum\limits_{i = 0}^{n - 1} \mathbb{E}[k_{i}] \approx \sum\limits_{i = 0}^{n - 1} L = n \cdot L,$$

where $$k_{i}, i = 0, 1, \dots , n - 1$$ are the keys, $$L$$ is the average length of the strings held by the tree. Quite a lot of memory consumption, isn't it?

Although tries provide a way of compression, when we store long words using tries, they tend to produce long chains of internal nodes, each with just one child, which requires further space optimization. This is where radix trees come into the picture. According to Wikipedia, in a radix tree, each node that is the only child is merged with its parent, resulting in the fact that the number of children of every internal node is at most the radix $$r$$ of the radix tree, where $$r$$ is a positive integer and a power $$x$$ of $$2$$, having $$x \geq 1$$.

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
// In: include/linux/radix-tree.h
#define RADIX_TREE(name, mask)  struct radix_tree_root name = RADIX_TREE_INIT(name, mask)
```

to declare and initialize a radix tree with the given `name`. The second form:

```c
struct radix_tree_root my_tree;
INIT_RADIX_TREE(my_tree, gfp_mask);
```

performs the initialization at run time. The object definitions are shown below:

```c
// In: include/linux/xarray.h
struct xarray {
	spinlock_t      xa_lock;
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
#define radix_tree_root		xarray

#define INIT_RADIX_TREE(root, mask) xa_init_flags(root, mask)
```

In either case, a `gfp_mask` must be provided to tell the code how memory allocations are to be performed[^2].

What is notable here is actually that the radix tree data structure in Linux has been converted to something called "[XArray](https://www.kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt)." Matthew Wilcox's third slide of his [presentation](https://lca-kernel.ozlabs.org/2018-Wilcox-Replacing-the-Radix-Tree.pdf) during the 2018 linux.conf.au Kernel miniconf summarized main characteristics of the Linux kernel's radix tree implementation[^3]:
- Implicit keys (like a trie), but not a bitwise trie
- Grow/shrink, but never rebalanced
- RCU-safe

## Genradix

## Integer ID Management

idr, ida, Andrew Morton

[^4]

## References

[^1]: The name *trie* comes from the phrase "information re*trie*val." Despite the etymology, trie is now almost always pronounced like *try* instead of *tree* to avoid confusion with other tree data structures. See [course notes](http://www.cs.yale.edu/homes/aspnes/classes/223/notes.html) written by Dr. James Aspnes from Yale University. Another good source for trie-based data structures is Peter Brass, *Advanced Data Structures*, Cambridge University Press, 2008.

[^2]: See [Linux Weekly News](lwn.net): "Trees I: Radix Trees" by Jonathan Corbet, March 13, 2006.

[^3]: Also, an example from Jonathan Corbet states that addition of an item to a tree has been called "insertion" for decades (since at least 1968), but an "insert" operation does not really describe what happens with a radix tree, especially if an item with the given key is already present there. See [Linux Weekly News](lwn.net): "The XArray Data Structure" by Jonathan Corbet, January 24, 2018.

[^4]: *Examining Linux 2.6 Page-Cache Performance* by Sonny Rao, Dominique Heger, and Steven Pratt, [landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf](https://landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf)