---
layout: 		      post
title: 			      "Pattern Matching in Standard ML"
category:		      "Programming Languages"
tags:			        functional-programming cse425-assignment
permalink:		    /blog/pattern-matching-sml
last_modified_at: "2022-10-09"
---

Homework 3 of the [programming language course](https://www.coursera.org/learn/programming-languages/) taught by Professor Dan Grossman from University of Washington.

**Standard ML**{: style="color: red"} extensively revised earlier dialects of the functional programming language ML, including the module facility that supports large-scale program development. The use of **pattern matching**{: style="color: red"} to access components of aggregate data structures is one of the most powerful, and distinctive, features of ML family[^1].

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## String Functions

### <code>only_capitals()</code>

Write a function that takes a <code>string list</code> and returns a <code>string list</code> that has only the strings in the argument that start with an uppercase letter:

```sml
val only_capitals =
  List.filter (fn str => Char.isUpper(String.sub(str, 0)))
```

### <code>longest_string1()</code>

Write a function that takes a <code>string list</code> and returns the longest <code>string</code> in the list. If the list is empty, return empty string <code>""</code>. In the case of a tie, return the string closest to the beginning of the list:

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
* If <code>longest_string_helper()</code> is passed a function that behaves like <code>></code> operator, then the function returned has the same behavior as <code>longest_string1()</code>;
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

## Pattern Matching Functions

### Utilities for Pattern Matching

Write a function <code>first_answer()</code> of type:
```sml
('a -> 'b option) -> 'a list -> 'b
```
The first argument should be applied to elements of the second argument in order until the first time it returns <code>SOME v</code> for some <code>v</code> and then <code>v</code> is the result of the call to <code>first_answer()</code>. If the first argument returns <code>NONE</code> for all list elements, then <code>first_answer()</code> should raise the exception <code>NoAnswer</code>.

```sml
fun first_answer f lst =
  case lst of 
    [] => raise NoAnswer
  | x::xs => 
      case f x of
        SOME v => v
      | NONE => first_answer f xs
```

Next, write another function <code>all_answers()</code> of type:
```sml
('a -> 'b list option) -> 'a list -> 'b list option
```
The first argument should be applied to elements of the second argument. If it returns <code>NONE</code> for any element, then the result for <code>all_answers()</code> is <code>NONE</code>. Else the calls to the first argument will have produced <code>SOME lst1</code>, <code>SOME lst2</code>, ... <code>SOME lstn</code> and the result of <code>all_answers()</code> is <code>SOME lst</code> where <code>lst</code> is <code>lst1</code>, <code>lst2</code>, ... <code>lstn</code> appended together (order doesn't matter).

```sml
fun all_answers f lst =
  let fun aux(f, lst, acc) =
    case lst of
      [] => SOME acc
    | x::xs =>
        case f x of
          SOME v => aux(f, xs, v@acc)
        | NONE => NONE
  in
    aux(f, lst, [])
  end
```

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
```

The rules for matching are:

* <code>Wildcard</code> matches everything and produces the empty list of bindings;
* <code>Variable s</code> matches any value <code>v</code>, and produces the one-element list <code>(s, v)</code>;
* <code>UnitP</code> matches only <code>Unit</code> and produces the empty list of bindings;
* <code>ConstP 17</code> matches only <code>Const 17</code> and produces the empty list of bindings (similarly for other integers).
* <code>TupleP ps</code> matches a value of the form <code>Tuple vs</code> if <code>ps</code> and <code>vs</code> have the same length and for all $i$, the $i$th element of <code>ps</code> matches the $i$th element of <code>vs</code>. The list of bindings produced is all the lists from the nested pattern matches appended together.
* <code>ConstructorP(s1, p)</code> matches <code>Constructor(s2, v)</code> if <code>s1</code> and <code>s2</code> are the same string (we can compare them with <code>=</code> operator) and <code>p</code> matches <code>v</code>. The list of bindings produced is the list from the nested pattern matche. We call the strings <code>s1</code> and <code>s2</code> the *constructor name*;
* Nothing else matches.

### <code>count_wildcards()</code>

Use the provided <code>g</code> function to define a function that takes a pattern and returns how many <code>Wildcard</code> patterns it contains:

```sml
val count_wildcards = g (fn _ => 1) (fn _ => 0)
```

### <code>count_wild_and_variable_lengths()</code>

Use the provided <code>g</code> function to define a function that takes a pattern and returns the number of <code>Wildcard</code> patterns it contains plus the sum of the string lengths of all the variables in the variable patterns it contains:

```sml
val count_wild_and_variable_lengths = g (fn _ => 1) String.size
```

### <code>count_some_var()</code>

Use the provided <code>g</code> function to define a function that takes a string-pattern pair and returns the number of times the string appears as a variable in the pattern:

```sml
val count_some_var = fn (str, pat) =>
  g (fn _ => 0) (fn x => case x = str of true => 1 | false => 0) pat
```

### <code>check_pat()</code>

Write a function that takes a pattern and returns true if and only if all the variables appearing in the pattern are distinct from each other (i.e., use different strings):

```sml
val check_pat =
  let
    (* Takes a pattern and returns a list of all strings the pattern
     * uses for variables. *)
    fun get_strlst pat =
      case pat of
        Variable x => [x]
      | TupleP ps => foldl (fn (p, acc) => (get_strlst p)@acc) [] ps
      | ConstructorP(_, p) => get_strlst p
      | _ => []

    (* Takes a list of strings and returns true iff no repeated strings. *)
    fun diff_str lst =
      case lst of
        [] => true 
      | x::xs =>
          case List.exists (fn y => x = y) xs of
            false => diff_str xs
          | true => false
  in
    diff_str o get_strlst
  end
```

### <code>match()</code>

Write a function that takes a <code>valu * pattern</code> pair and returns a <code>(string * valu) list option</code>, namely <code>NONE</code> if the pattern does not match and <code>SOME lst</code> where <code>lst</code> is the list of bindings if it does. Note that if the value matches but the pattern has no patterns of the form <code>Variable s</code>, then the result is <code>SOME []</code>.

```sml
fun match x = (* To recursively call match, avoid the anonymous function binding *)
  case x of
    (* Wildcard matches everything *)
    (_, Wildcard) => SOME []
  | (v, Variable s) => SOME [(s, v)]
    (* UnitP matches only Unit *)
  | (Unit, UnitP) => SOME []
    (* int value matches *)
  | (Const y, ConstP z) => 
      case y = z of 
      true => SOME [] 
    | false => NONE
  | (Tuple vs, TupleP ps) =>
      case List.length vs = List.length ps of
        true => all_answers match (ListPair.zip (vs, ps))
      | false => NONE
  | (Constructor(s, v), ConstructorP(t, p)) =>
      case s = t of 
        true => match(v, p)
      | false => NONE )
  | _ => NONE
```

### <code>first_match()</code>

Write a function that takes a <code>valu</code> and a list of patterns and returns a <code>(string * valu) list option</code>, namely <code>NONE</code> if no pattern in the list matches or <code>SOME lst</code> where <code>lst</code> is the list of bindings for the first pattern in the list that matches.

```sml
fun first_match v lst =
  SOME (first_answer (fn p => match (v, p)) lst)
  handle NoAnswer => NONE
```

### Challenge Problem: <code>typecheck_patterns()</code>

Write a function <code>typecheck_patterns()</code> of type
```sml
((string * string * typ) list) * (pattern list) -> typ option
```
that "type-checks" a pattern list. Types for our made-up pattern language are values of type <code>typ</code>, which is defined as:
```sml
datatype typ = Anything           (* any type of value is okay *)
             | UnitT              (* type of Unit *)
             | IntT               (* type for integers *)
             | TupleT of typ list (* tuple types *)
             | Datatype of string (* some named datatype *)
```

The first argument of type <code>(string * string * typ) list</code> contains elements that take the form of:
```sml
(* constructor name, datatype name, typ *)
("foo", "bar", IntT)
```
which means constructor <code>foo</code> makes a value of type <code>datatype bar</code> given a value of type <code>IntT</code>. Assume list elements all have different first fields (the constructor name), but there are probably elements with the same second field (the datatype name). <code>typecheck_patterns()</code> "type-check" the pattern list to see if there exists some <code>typ t</code> that all the patterns in the list can have. If so, return <code>SOME t</code>, else return <code>NONE</code>.

Consider a case expression with different patterns:

```sml
case x of p1 | p2 | ... | pn
```

> The objective of this challenge exercise is to create an algorithm that, like the SML compiler[^2], is capable of inferring the type <code>t</code> of <code>x</code> based on the patterns <code>p1</code>, <code>p2</code>, ..., <code>pn</code>.

My implementation here includes four helper functions. The first one, <code>pattern_to_typ()</code>, converts a <code>pattern</code> to a <code>typ</code>:

```sml
fun pattern_to_typ (pat, cons) =
  case pat of
    Wildcard => Anything
  | Variable _ => Anything
  | UnitP => UnitT
  | ConstP _ => IntT
  | TupleP ps =>
      (* Converts every pattern to typ in ps,
       * where foldl returns a typ list *)
      TupleT (foldl (fn (p, acc) => acc@[pattern_to_typ (p, cons)]) [] ps)
  | ConstructorP (s, p) => 
    (* Matches a ConstructorP in a list of string * string * typ pairs
     * (the constructor list) and converts it to a named Datatype *)
      let
        fun cons_to_nd cs =
          case cs of
            [] => Datatype ""
          | (x, y, z)::rest =>
              if (x = s) andalso
                 (pattern_to_typ (p, cons) = z orelse
                  pattern_to_typ (p, cons) = Anything)
              then Datatype y
              else cons_to_nd rest
      in
        cons_to_nd cons
      end
```

The second helper function, <code>patlst_to_typlst()</code>, takes a list of patterns and returns a <code>typ list</code> making use of the above function:

```sml
fun patlst_to_typlst (cons, pats, acc) =
  case (pats, acc) of
    ([], []) => [Datatype ""]
  | ([], _) => acc
  | (p::ps, _) => patlst_to_typlst (cons, ps, acc@[pattern_to_typ (p, cons)])
```

The third helper function, <code>typlst_to_typ()</code>, takes a <code>typ list</code> and returns the "most lenient" <code>typ</code>:

```sml
val typlst_to_typ =
  foldl
  (* Given two typs, returns an appropriate typ *)
  ( fn (t1, t2) => case (t1, t2) of
      (Anything, t2) => t2
    | (t1, Anything) => t1
    | (TupleT(x), TupleT(y)) =>
        let
          fun aux(ps, acc) =
            case ps of
              [] => TupleT(acc)
            | (p1, p2)::rest => aux(rest, acc@[typlst_to_typ([p1, p2])])
        in
          (* raises UnequalLengths with zipEq *)
          aux(ListPair.zipEq(x, y), [])
          handle UnequalLengths => TupleT([Datatype ""])
        end
    | (d1, d2) => case d1 = d2 of true => d1 | false => Datatype ""
  Anything
```

The last helper function, <code>typop()</code>, converts a <code>typ</code> to a <code>typ option</code>:

```sml
fun typop t =
  case t of
    Datatype "" => NONE
  | TupleT(typlst) =>
      let 
        fun find([]) = SOME t
          | find(head::rest) =
              case typop head of
                NONE => NONE
              | _ => find(rest)
      in
        find(typlst)
      end
  | _ => SOME t
```

Finally, our <code>typecheck_patterns()</code> function is just a wrapper:

```sml
fun typecheck_patterns (conslst, patlst) =
  (typop o typlst_to_typ o patlst_to_typlst) (conslst, patlst, [])
```

## Notes

[^1]: See [Robert Harper, *Programming in Standard ML*, 2011.](http://www.cs.cmu.edu/~rwh/isml/book.pdf)

[^2]: For the SML/NJ compiler's type-checking code, please go to the GitHub directory: [https://github.com/smlnj/smlnj/compiler/](https://github.com/smlnj/smlnj/blob/main/compiler/).