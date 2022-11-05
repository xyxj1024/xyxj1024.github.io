---
layout: 		      post
title: 			      "A Made Up Programming Language in Racket"
category:		      "Programming Languages"
tags:			        data-structures functional-programming
permalink:		    /mupl-racket/
last_modified_at: "2022-11-04"
---

Homework 5 of the [programming language course](https://www.coursera.org/learn/programming-languages/) taught by Professor Dan Grossman from University of Washington.

Racket programming language is the grandchild of LISP. Compared to ML, Racket does not use a static type system and has a very minimalist and uniform syntax. Racket's <code>struct</code>, which will be used extensively in this assignment, is like an ML constructor. A <code>struct</code> definition may look like:
```racket
(struct foo (bar baz quux) #:transparent)
```

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />