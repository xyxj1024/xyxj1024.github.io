---
layout: 		      post
title: 			      "Pattern Matching in Standard ML"
category:		      "Programming Languages"
tags:			        functional-programming
permalink:		    /pattern-matching-sml/
last_modified_at: "2022-10-09"
---

Homework 3 of the [programming language course](https://www.coursera.org/learn/programming-languages/) taught by Professor Dan Grossman from University of Washington.

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## String Functions

### <code>only_capitals()</code>

Write a function that takes a <code>string list</code> and returns a <code>string list</code> that has only the strings in the argument that start with an uppercase letter:

```sml
val only_capitals =
  List.filter (fn str => Char.isUpper(String.sub(str, 0)))
```

### <code>longest_string1()</code>

Write a function that takes a <code>string list</code> and returns the longest <code>string</code> in the list. If the list is empty, return <code>""</code>. In the case of a tie, return the string closest to the beginning of the list:

```sml
val longest_string1 =
  foldl
  ( fn (str, max) => 
      case (String.size str > String.size max) of 
        true => str | false => max )
  ""
```

### <code>longest_string2()</code>

Write a function that is exactly like <code>longest_string1()</code> except in the case of ties it returns the string closest to the end of the list:

```sml
val longest_string2 =
  foldl
  ( fn (str, max) =>
      (* >= rather than > *) 
      case (String.size str >= String.size max) of 
        true => str | false => max )
  ""
```

### <code>longest_string_helper()</code>, <code>longest_string3()</code>, and <code>longest_string4()</code>

* <code>longest_string3()</code> has the same behavior as <code>longest_string1()</code> and <code>longest_string4()</code> has the same behavior as <code>longest_string2()</code>;
* <code>longest_string_helper()</code> has type <code>(int * int -> bool) -> string list -> string</code>. This function is more general that takes a function as an argument;
* If <code>longest_string_helper()</code> is passed a function that behaves like <code>></code>, then the function returned has the same behavior as <code>longest_string1()</code>;
* <code>longest_string3()</code> and <code>longest_string4()</code> are defined with value bindings and partial applications of <code>longest_string_helper()</code>.

```sml
var longest_string_helper f =
  foldl
  ( fn (x, y) =>
      (* A more general function*)
      case f (String.size x, String.size y) of 
        true => x | false => y )
  ""

val longest_string3 =
  longest_string_helper(fn (x, y) => x > y)

val longest_string4 =
  longest_string_helper(fn (x, y) => x >= y)
```

### <code>longest_capitalized()</code>

Write a function that takes a <code>string list</code> and returns the longest <code>string</code> in the list that begins with an uppercase letter, or <code>""</code> if there are no such strings. Assume all strings have at least one character. Resolve ties like in <code>longest_string1()</code>.

```sml
val longest_capitalized = longest_string1 o only_capitals
```

### <code>rev_string()</code>

Write a function that takes a <code>string</code> and returns the <code>string</code> that is the same characters in reverse order.

```sml
val rev_string = implode o rev o explode
```

## Utilities for Pattern Matching

### Type Declarations

Given <code>v</code> of <code>valu</code> and <code>p</code> of <code>pattern</code>, either <code>p</code> matches <code>v</code> or not. If it does, the match produces a list of <code>string * valu</code> pairs; order in the list does not matter.

```sml
(* Inspired by the type definitions an ML implementation
 * would use to implement pattern matching: *)
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

The rules for matching are:

* <code>Wildcard</code> matches everything and produces the empty list of bindings;
* <code>Variable s</code> matches any value <code>v</code>, and produces the one-element list <code>(s, v)</code>;
* <code>UnitP</code> matches only <code>Unit</code> and produces the empty list of bindings;
* <code>ConstP 17</code> matches only <code>Const 17</code> and produces the empty list of bindings (similarly for other integers).
* <code>TupleP ps</code> matches a value of the form <code>Tuple vs</code> if <code>ps</code> and <code>vs</code> have the same length and for all $i$, the $i$th element of <code>ps</code> matches the $i$th element of <code>vs</code>. The list of bindings produced is all the lists from the nested pattern matches appended together.
* <code>ConstructorP(s1, p)</code> matches <code>Constructor(s2, v)</code> if <code>s1</code> and <code>s2</code> are the same string (we can compare them with <code>=</code>) and <code>p</code> matches <code>v</code>. The list of bindings produced is the list from the nested pattern matche. We call the strings <code>s1</code> and <code>s2</code> the *constructor name*;
* Nothing else matches.

### Challenge Problem: <code>typecheck_patterns()</code>

Consider a case expression with different patterns:

```sml
case x of p1 | p2 | ... | pn
```

> The objective of this challenge exercise is to create an algorithm that, like the SML compiler, is capable of inferring the type <code>t<code> of <code>x</code> based on the patterns <code>p1</code>, <code>p2</code>, ..., <code>pn</code>.