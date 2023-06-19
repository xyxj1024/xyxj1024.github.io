---
layout: 		      post
title: 			      "A Made Up Programming Language in Racket"
category:		      "Programming Languages"
tags:			        interpreter functional-programming cse425-assignment
permalink:		    /posts/mupl-racket
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

Our main tasks in this assignment have to do with a Racket function <code>eval-exp</code>, which is an interpreter of our Made Up Programming Language (hereinafter "MUPL") that takes a MUPL expression <code>e</code> and either returns the MUPL value that <code>e</code> evaluates to under the empty environment or calls Racket's <code>error</code> if evaluation encounters a run-time MUPL type error or unbound MUPL variable. A MUPL *value* is a MUPL integer constant, a MUPL closure, a MUPL aunit, or a MUPL pair of MUPL values. An *environment* is represented by a Racket list of [Racket pairs](https://docs.racket-lang.org/reference/pairs.html) <code>'(name . value)</code> where each <code>name</code> is a Racket string and each <code>value</code> is a MUPL value.

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Problem 1: Warm-Up

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

## Problem 2: Implementing the MUPL Language

```racket
;; lookup a variable in an environment
;; Do NOT change this function
(define (envlookup env str)
  (cond [(null? env) (error "unbound variable during evaluation" str)]
        [(equal? (car (car env)) str) (cdr (car env))]
        [#t (envlookup (cdr env) str)]))
```

```racket
;; Author: Xingjian Xuanyuan
;; CSE 425 utility
;; Evaluates to #t if value is any of the MUPL value types
(define (mupl-value? value)
  (let ([reg (and (not (int? value))
                  (not (closure? value))
                  (not (aunit? value))
                  (not (apair? value)))])
    (not reg)))

;; CSE 425 utility
;; Evaluates to an expanded version of the provided list env
;; by cons-ing a racket (name, value) pair onto env
(define (expand-environment name value env)
  (cond
    [(not (string? name)) (error (string-append "illegal name: " (~a name)))]
    [(not (mupl-value? value)) (error (string-append "illegal value: " (~a value)))]
    [#t (cons (cons name value) env)]))
```

Here is a description of the semantics of MUPL expressions:
- All values (including closures) evaluates to themselves.
- A variable evaluates to the value associated with it in the environment.
- An addition evaluates its subexpressions, assuming they both produce integers, and produces the integer that is their sum.
- Functions are lexically scoped: A function evaluates to a closure holding the function and the current environment.
- An <code>ifgreater</code> evaluates its first two subexpressions to values <code>v1</code> and <code>v2</code> respectively. If both values are integers, it evaluates its third subexpression if <code>v1</code> is a strictly greater integer than <code>v2</code>, else it evaluates its fourth subexpression.
- An <code>mlet</code> expression evaluates its first expression to a value <code>v</code>. Then it evaluates the second expression to a value, in an environment extended to map the name in the <code>mlet</code> expression to <code>v</code>.
- A <code>call</code> evaluates its first and second subexpressions to values. If the first is not a closure, it is an error. Else, it evaluates the closure's function's body in the closure's environment extended to map the function's name to the closure (unless the name field is <code>#f</code>) and the function's argument-name (i.e., the parameter name) to the result of the second subexpression.
- An <code>apair</code> expression evaluates its two subexpressions and produces a (new) pair holding the results.
- A <code>fst</code> expression evaluates its subexpression. If the result for the subexpression is a pair, then the result for the <code>fst</code> expression is the <code>e1</code> field in the pair.
- A <code>snd</code> expression evaluates its subexpression. If the result for the subexpression is a pair, then the result for the <code>snd</code> expression is the <code>e2</code> field in the pair.
- An <code>isaunit</code> expression evaluates its subexpression. If the result is an <code>aunit</code> expression, then the result for the <code>isaunit</code> expression is the MUPL value <code>(int 1)</code>, else the result is the MUPL value <code>(int 0)</code>.

<br>

```racket
(define (eval-under-env e env)
  (cond [(var? e) 
         (envlookup env (var-string e))]
        [(int? e) e] ; this is trivial
        [(add? e) 
         (let ([v1 (eval-under-env (add-e1 e) env)]
               [v2 (eval-under-env (add-e2 e) env)])
           (if (and (int? v1)
                    (int? v2))
               (int (+ (int-num v1) 
                       (int-num v2)))
               (error "MUPL addition applied to non-number")))]
        [(ifgreater? e)
         (let ([v1 (eval-under-env (ifgreater-e1 e) env)]
               [v2 (eval-under-env (ifgreater-e2 e) env)])
           (if (and (int? v1) (int? v2))
               (if (> (int-num v1) (int-num v2))
                   (eval-under-env (ifgreater-e3 e) env)
                   (eval-under-env (ifgreater-e4 e) env))
               (error "MUPL ifgreater comparing non-numbers")))]
        [(fun? e) (closure env e)] ; this is trivial
        [(call? e)
         (let ([v1 (eval-under-env (call-funexp e) env)]
               [v2 (eval-under-env (call-actual e) env)])
           (if (not (closure? v1))
               (error "MUPL call-funexp not a closure")
               (let ([function-name (fun-nameopt (closure-fun v1))])
                 (if (not function-name) ; the first argument is #f
                     (eval-under-env (fun-body (closure-fun v1))
                                     (expand-environment (fun-formal (closure-fun v1))
                                                         v2
                                                         (closure-env v1)))
                     (eval-under-env (fun-body (closure-fun v1))
                                     (expand-environment (fun-formal (closure-fun v1))
                                                         v2
                                                         (expand-environment function-name
                                                                             v1
                                                                             (closure-env v1))))))))]
        [(mlet? e)
         (let ([v (eval-under-env (mlet-e e) env)])
           (eval-under-env (mlet-body e) (expand-environment (mlet-var e) v env)))]
        [(apair? e)
         (let ([v1 (eval-under-env (apair-e1 e) env)]
               [v2 (eval-under-env (apair-e2 e) env)])
           (apair v1 v2))]
        [(fst? e)
         (let ([v (eval-under-env (fst-e e) env)])
           (if (apair? v)
               (apair-e1 v)
               (error "MUPL fst applied to non-pair")))]
        [(snd? e)
         (let ([v (eval-under-env (snd-e e) env)])
           (if (apair? v)
               (apair-e2 v)
               (error "MUPL snd applied to non-pair")))]
        [(aunit? e) e] ; this is trivial
        [(isaunit? e)
         (let ([v (eval-under-env (isaunit-e e) env)])
           (if (aunit? v) (int 1) (int 0)))]
        [(closure? e) e] ; this is trivial
        [#t (error (format "bad MUPL expression: ~v" e))]))

;; Function eval-exp takes a MUPL expression e and either
;; returns the MUPL value that e evaluates to under the empty
;; environment or calls Racket's error if evaluation encounters a
;; run-time MUPL type error or unbound UPL variable.
;; Do NOT change
(define (eval-exp e)
  (eval-under-env e null))
```

## Problem 3: Expanding the Language

MUPL is a small language, but we can write Racket functions that act like MUPL macros so that users of these functions feel like MUPL is larger. The following are Racket functions that produce MUPL expressions that could then be put inside larger MUPL expressions or passed to <code>eval-exp</code>. In implementing these Racket functions, do not use <code>closure</code> and <code>eval-exp</code>.

```racket
;; Function ifaunit evaluates e1, if the result is MUPL aunit then it
;; evaluates e2, else it evaluates e3.
(define (ifaunit e1 e2 e3)
  (ifgreater (isaunit e1) (int 0) e2 e3)) 

;; Function mlet* takes a Racket list of Racket pairs, where
;; the first component is a Racket string and the second component
;; is a MUPL expression, and a final MUPL expression; returns a
;; MUPL expression whose value is evaluated in an environment defined
;; by the list of '(name . value) pairs.
(define (mlet* lstlst e2)
  (cond
    [(null? lstlst) e2]
    [else
     (let ([v (first lstlst)])
       (mlet (car v) (cdr v) (mlet* (list-tail lstlst 1) e2)))]))

;; Function ifeq acts like ifgreater except e3 is evaluated if and only if
;; e1 and e2 are equal integers.
;; (define (ifeq e1 e2 e3 e4)
;;   (if (and (int? e1)
;;            (int? e2)
;;            (equal? (int-num e1) (int-num e2)))
;;       e3
;;       e4))
;; Better:
(define (ifeq e1 e2 e3 e4)
  (mlet* (list (cons "_x" e1) (cons "_y" e2))
         (ifgreater (var "_x")
                    (var "_y")
                    e4
                    (ifgreater (var "_y") (var "_x") e4 e3))))
```

## Problem 4: Using the Language

We can write MUPL expressions directly in Racket using the constructors for the <code>struct</code>'s and (for convenience) the functions we wrote in the previous problem.

```racket
;; Racket version for reference
(define (racket-double n) (+ n n))
;; CSE 425 MUPL Program
;; mupl-double return a mupl-function (closure) which doubles its
;; mupl-int argument
(define mupl-double
  (closure '() (fun #f "x" (add (var "x") (var "x")))))

;; Racket version for reference
(define (racket-sum-curry a) (lambda (b) (+ a b)))
;; CSE 425 MUPL Program
;; mupl-sum-curry return a mupl-function which returns a mupl-function
;; which adds the two mupl-function's mupl-int arguments together
(define mupl-sum-curry
  (closure '() (fun #f "b" (fun #f "a" (add (var "a") (var "b"))))))

;; Racket version for reference
(define (racket-map-one proc) (proc 1))
;; CSE 425 MUPL Program
;; mupl-map-one: return a mupl-function that invokes the mupl-function
;; passed in with the mupl-int argument 1
(define mupl-map-one
  (closure '() (fun #f "proc" (call (var "proc") (int 1)))))
```

```racket
;; UW HW4 Continues Below

;; Function mupl-map takes a MUPL function and returns a MUPL function
;; that takes a MUPL list and applies the function to every element of
;; the list returning a new MUPL list.
(define mupl-map
  (closure '()
           (fun "map" "f"
                (fun "rmap" "l"
                     (ifaunit (var "l")
                              (aunit)
                              (apair (call (var "f") (fst (var "l")))
                                     (call (var "rmap") (snd (var "l")))))))))

;; Function mupl-mapAddN takes an MUPL integer i and returns a MUPL
;; function that takes a MUPL list of MUPL integers and returns a new
;; MUPL list of MUPL integers that adds i to every element of the list.
(define mupl-mapAddN 
  (mlet "map" mupl-map
        (fun "f-int" "i"
             (fun "f-list" "is"
                  (call (call (var "map") (fun "add-i" "x" (add (var "i") (var "x"))))
                        (var "is")))))) 
```