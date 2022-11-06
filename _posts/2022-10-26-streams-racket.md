---
layout: 		      post
title: 			      "Thunks and Streams in Racket"
category:		      "Programming Languages"
tags:			        functional-programming lazy-evaluation
permalink:		    /streams-racket/
last_modified_at: "2022-11-06"
---

Some code collected from the Homework 4 of [Programming Languages, Part B](https://www.coursera.org/learn/programming-languages/) taught by Professor Dan Grossman from University of Washington and the Warm-Up exercise of [CSE 425S: Programming Systems and Languages](https://classes.engineering.wustl.edu/cse425s/) taught by Professor Dennis Cosgrove from Washington University in St. Louis (also see [Professor Chris Gill's Home Page](https://www.cse.wustl.edu/~cdgill/) for past offerings).

<!-- excerpt-end -->

Functional programming is, indeed, all about **expression evaluation**{: style="color: red"}. Expression evaluation is the computation of the value of an expression. An expression can contain variables. Here is some concise interpretation of the **evaluation model** of functional programming as well as the concept of "**lazy evaluation**" from [UCSD CSE130: Programming Languages: Principles & Paradigms](https://cseweb.ucsd.edu/classes/wi00/cse130/):
> The values for the variables are taken from an *environment* which holds name-value pairs. Ideally, the evaluation of an expression should not change the environment. This property is called *referential transparency*.
> ...
> Functional languages have the facility for function creation. Such functions are associated with two environments: *definition environment* and *activation environment*. So a function should always carry its definition environment. The pair <code>(function, definition-environment)</code> is called a *closure*.
> ...
> There are two ways of evaluating expressions. In the *innermost evaluation* rule, a function application <code>function-name (actual-parameter)</code> is calculated by: (1) evaluating the expression represented by <code>actual-parameter</code>; (2) substituting the result for the formal in the function body; (3) evaluating the body; (4) finally returning the result. Under the *outermost evaluation* rule, a function application is calculated by: (1) substituting the actual for the formal in the function body; (2) evaluating the body; (3) finally returning its value as the answer. Both type of evaluation produce the same result under "normal" circumstances. ... Outermost evaluation is also called *lazy evaluation* since an expression is not evaluated unless it is required. Compiler can prevent duplicate evaluations. So the programmer is free from such concerns.

```sml
signature THUNK =
  sig
    (* A 'a thunk is a lazily evaluated
     * expression e of type 'a. *)
    type 'a thunk

    (* Creates a thunk for e. *)
    val make : (unit->'a) -> 'a thunk

    (* Returns the value of t's
     * expression, which is only
     * evaluated once. *)
    val apply : 'a thunk -> 'a
  end

structure Thunk :> THUNK =
  struct
    datatype 'a thunkPart =
      Done of 'a
    | Undone of unit -> 'a

    type 'a thunk = ('a thunkPart) ref

    fun make (f : unit -> 'a) : 'a thunk =
      ref (Undone f)
    
    fun apply (th : 'a thunk) : 'a =
      case !th of
        Done x => x
      | Undone f =>
          let
            val ans = f()
          in
            (th := Done ans; ans)
          end
  end
```

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Racket <code>cons</code> vs. <code>values</code>

The behavior of Racket's <code>values</code> shows similarity to pattern matching as in programming languages like ML. Below are some functions used by Professor Dennis Cosgrove in [this video](https://www.youtube.com/watch?v=iWCcL049dOg):

```racket
#lang racket

(define (div-mod-pair n d)
  (cons (quotient n d) (modulo n d)))

(define (printf-div-mod-pair n d)
  (local [(define pr (div-mod-pair n d))
          (define qt (car pr))
          (define rm (cdr pr))]
    (printf "~a/~a => ~a rem ~a\n" n d q r)))

(define (div-mod-values n d)
  (values (quotient n d) (modulo n d)))

(define (printf-div-mod-values n d)
  (local [(define-values (qt rm) (div-mod-values n d))]
    (printf "~a/~a => ~a rem ~a\n" n d q r)))

; Try them with REPL
(printf-div-mod-pair 425 231)
(printf-div-mod-values 425 231)
```