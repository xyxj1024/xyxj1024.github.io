---
layout:     post
title:      "Merkle Trees"
category:   "Data Structures and Algorithms"
tags:       tree cryptography
permalink:  /blog/crypto/merkle-trees
---

The [Merkle tree](http://www.ralphmerkle.com/papers/Certified1979.pdf)[^1] data structure was proposed by Ralph C. Merkle in 1979 to mitigate the huge storage requirements of traditional one-time signature methods. The research problem was originally stated as shown below:

> Given a vector of data items $$Y = Y_{1}, Y_{2}, \dots , Y_{n}$$, design an algorithm which can quickly authenticate a randomly chosen $$Y_{i}, i = 1, \dots , n$$ but which has modest memory requirements, i.e., does not have a table of $$Y_{1}, Y_{2}, \dots , Y_{n}$$.

Each node of the authentication tree stores the hash value $$H(i, j, \underline{Y})$$, which is a function of $$Y_{i}, Y_{i + 1}, \dots , Y_{j}$$ and can be calculated using its children: $$H(i, j, \underline{Y}) = F(H(i, (i + j - 1) / 2, \underline{Y}) \Vert H((i + j + 1) / 2, j, \underline{Y}))$$. The computation of the root $$H(1, n, \underline{Y})$$ forms a binary tree of recursive calls.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Notes

[^1]: Ralph C. Merkle, "A Certified Digital Signature," In: G. Brassard (eds): *CRYPTO 1989*, LNCS 435, pp. 218-238, 1989.

Michael Szydlo, "Merkle Tree Traversal in Log Space and Time," In: C. Cachin and J. Camenisch (eds.): *EUROCRYPT 2004*, LNCS 3027, pp. 541-554, 2004.

Dan Williams and Emin G&uuml;n Sirer, "Optimal Parameter Selection for Efficient Memory Integrity Verification Using Merkle Hash Trees," In: *Proceedings of the Third IEEE International Symposium on Network Computing and Applications*, 2004.

Blaise Gassend, G. Edward Suh, Dwaine Clarke, Marten van Dijk and Srinivas Devadas, "Caches and Hash Trees for Efficient Memory Integrity Verification," *HPCA '03: Proceedings of the 9th International Symposium on High-Performance Computer Architecture*, February 2003.

Roberto J. Bayardo and Jeffrey Sorensen, "Merkle Tree Authentication of HTTP Responses," *WWW 2005*, May 10-14, 2005, Chiba, Japan.