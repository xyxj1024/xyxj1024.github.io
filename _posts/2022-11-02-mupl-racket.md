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

This defines a new <code>struct</code> called <code>foo</code>, which adds to the environment functions for constructing a <code>foo</code>, testing if something is a <code>foo</code>, and extracting the fields <code>bar</code>, <code>baz</code>, and <code>quux</code> from a <code>foo</code>. The <code>#:transparent</code> attribute makes the fields and accessor functions visible even outside the module that defines the <code>struct</code>. Including <code>#:transparent</code> in our definition allows the REPL to print <code>struct</code> values with their contents rather than just as an abstract value. Racket's <code>struct</code> actually creates a new type of data.

Below is our starter code:
```racket
#lang racket
(provide (all-defined-out)) ;; so we can put tests in a second file

;; definition of structures for MUPL programs - Do NOT change
(struct var  (string) #:transparent)  ;; a variable, e.g., (var "foo")
(struct int  (num)    #:transparent)  ;; a constant number, e.g., (int 17)
(struct add  (e1 e2)  #:transparent)  ;; add two expressions
(struct ifgreater (e1 e2 e3 e4)    #:transparent) ;; if e1 > e2 then e3 else e4
(struct fun  (nameopt formal body) #:transparent) ;; a recursive(?) 1-argument function
(struct call (funexp actual)       #:transparent) ;; function call
(struct mlet (var e body) #:transparent) ;; a local binding (let var = e in body) 
(struct apair (e1 e2)     #:transparent) ;; make a new pair
(struct fst  (e)    #:transparent) ;; get first part of a pair
(struct snd  (e)    #:transparent) ;; get second part of a pair
(struct aunit ()    #:transparent) ;; unit value -- good for ending a list
(struct isaunit (e) #:transparent) ;; evaluate to 1 if e is unit else 0

;; a closure is not in "source" programs but /is/ a MUPL value; it is what functions evaluate to
(struct closure (env fun) #:transparent)
```

Our main tasks in this assignment has to do with a Racket function <code>eval-exp</code>, which is an interpreter of our Made Up Programming Language (hereinafter "MUPL") that takes a MUPL expression <code>e</code> and either returns the MUPL value that <code>e</code> evaluates to under the empty environment or calls Racket's <code>error</code> if evaluation encounters a run-time MUPL type error or unbound MUPL variable. A MUPL *value* is a MUPL integer constant, a MUPL closure, a MUPL aunit, or a MUPL pair of MUPL values. A environment is represented by a Racket list of [Racket pairs](https://docs.racket-lang.org/reference/pairs.html) <code>'(name . value)</code> where <code>name</code> is a Racket string and <code>value</code> is a MUPL value.

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Problem 1

First, write a Racket function <code>racketlist->mupllist</code> that takes a Racket list (presumably of MUPL values but that will not affect the solution) and produces an analogous MUPL list with the same elements in the same order.

```racket
(define (racketlist->mupllist rl)
  (cond
    [(null? rl) (aunit)] ; (aunit) is a MUPL expression but aunit is not
    [#t (apair (first rl) (racketlist->mupllist (list-tail rl 1)))]))
```

Then write a Racket function <code>mupllist->racketlist</code> that takes a MUPL list (presumably of MUPL values but that will not affect the solution) and produces an analogous Racket list (of MUPL values) with the same elements in the same order.

```racket
(define (mupllist->racketlist ml)
  (cond
    [(aunit? ml) '()]
    [#t (cons (apair-e1 ml) (mupllist->racketlist (apair-e2 ml)))]))
```

## Problem 2

