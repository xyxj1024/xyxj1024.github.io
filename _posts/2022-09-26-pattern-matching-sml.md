---
layout: 		  post
title: 			  "Pattern Matching in Standard ML"
category:		  "Programming Languages"
tags:			  functional-programming
permalink:		  /pattern-matching-sml/
last_modified_at: "2022-10-09"
---

Homework 3 of the [programming language course](https://www.coursera.org/learn/programming-languages/) taught by Dan Grossman from University of Washington.

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Type Declarations

```sml
datatype pattern = Wildcard
                 | Variable of string
                 | UnitP
                 | ConstP of int
                 | TupleP of pattern list
                 | ConstructorP of string * pattern

datatype valu = Const of int
              | Unit
              | Tuple of valu list
              | Constructor of string * valu

fun g f1 f2 p =
    let 
        val r = g f1 f2 
    in
        case p of
            Wildcard          => f1 ()
          | Variable x        => f2 x
          | TupleP ps         => List.foldl (fn (p,i) => (r p) + i) 0 ps
          | ConstructorP(_,p) => r p
          | _                 => 0
    end

(**** for the challenge problem only ****)

datatype typ = Anything           (* any type of value is okay *)
	         | UnitT              (* type of Unit *)
	         | IntT               (* type for integers *)
	         | TupleT of typ list (* tuple types *)
	         | Datatype of string (* some named datatype *)
```

## Challenge Problem: <code>typecheck_patterns()</code>

Consider a case expression with different patterns:

```sml
case x of p1 | p2 | ... | pn
```

> [The objective of this challenge exercise is to create an algorithm that, like the SML compiler, is capable of inferring the type <code>t<code> of <code>x</code> based on the patterns <code>p1</code>, <code>p2</code>, ..., <code>pn</code>.](https://www.coursera.org/learn/programming-languages/supplement/FEymH/hints-and-gotchas-for-section-3)