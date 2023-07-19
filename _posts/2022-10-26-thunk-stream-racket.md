---
layout: 		      post
title: 			      "Thunks and Streams in Racket"
category:		      "Programming Languages"
tags:			        functional-programming lazy-evaluation cse425-assignment
permalink:		    /blog/thunk-stream-racket
last_modified_at: "2022-11-06"
---

Some code collected from the Homework 4 of ["Programming Languages, Part B"](https://www.coursera.org/learn/programming-languages/) taught by Professor Dan Grossman from University of Washington and the Warm-Up exercise of [CSE 425S: "Programming Systems and Languages"](https://classes.engineering.wustl.edu/cse425s/) taught by Professor Dennis Cosgrove from Washington University in St. Louis (also see [Professor Chris Gill's Home Page](https://www.cse.wustl.edu/~cdgill/) for past offerings).

Professor Dennis Cosgrove hosts a [YouTube channel](https://www.youtube.com/@denniscosgrove8978) which might be of interest to those computer science newcomers.

<!-- excerpt-end -->

Functional programming is, indeed, all about **expression evaluation**{: style="color: red"}. Expression evaluation is the computation of the value of an expression. An expression can contain variables. Here is some concise interpretation of the **evaluation model** of functional programming as well as the concept of "**lazy evaluation**" from [UCSD CSE130's course website](https://cseweb.ucsd.edu/classes/wi00/cse130/) (another pretty informative note on lazy evaluation can be found at [Cornell CS312's course website](https://www.cs.cornell.edu/courses/cs312/2004fa/lectures/rec08.htm)):
> The values for the variables are taken from an *environment* which holds name-value pairs. Ideally, the evaluation of an expression should not change the environment. This property is called *referential transparency*.
> ...
> Functional languages have the facility for function creation. Such functions are associated with two environments: *definition environment* and *activation environment*. So a function should always carry its definition environment. The pair <code>(function, definition-environment)</code> is called a *closure*.
> ...
> There are two ways of evaluating expressions. In the *innermost evaluation* rule, a function application <code>function-name (actual-parameter)</code> is calculated by: (1) evaluating the expression represented by <code>actual-parameter</code>; (2) substituting the result for the formal in the function body; (3) evaluating the body; (4) finally returning the result. Under the *outermost evaluation* rule, a function application is calculated by: (1) substituting the actual for the formal in the function body; (2) evaluating the body; (3) finally returning its value as the answer. Both type of evaluation produce the same result under "normal" circumstances. ... Outermost evaluation is also called *lazy evaluation* since an expression is not evaluated unless it is required. Compiler can prevent duplicate evaluations. So the programmer is free from such concerns.

Technically, in a functional language like Haskell, lazy evaluation means ["call-by-name" plus "sharing"](https://wiki.haskell.org/Lazy_evaluation). Unlike Haskell, Racket (and most other languages) *eagerly evaluates* function arguments, i.e. [a function call is performed as soon as it is encountered in a procedure](https://en.wikipedia.org/wiki/Evaluation_strategy#Eager_evaluation).

A **thunk**{: style="color: red"} <code>fn () => e</code> is a value (or zero-argument function) that is yet to be evaluated. A lazy run-time system does not evaluate a thunk unless it has to. Below is a thunk ADT implemented in Standard ML:

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

A **promise**{: style="color: red"} is a <code>struct</code> containing a mutable field (a pointer). A promise can hold a thunk:

```sml
signature PROMISE =
  sig
    (* Type of promises for 'a. *)
    type 'a t

    (* Takes a thunk for an 'a and
     * makes a promise to produce an 'a. *)
    val delay : (unit -> 'a) -> 'a t

    (* Calls thunk and saves if promise not yet forced;
     * Returns saved thunk result. *)
    val force : 'a t -> 'a
  end

structure Promise :> PROMISE =
  struct
    datatype 'a promise = Thunk of unit -> 'a
                        | Value of 'a
    type 'a t = 'a promise ref
    fun delay thunk = ref (Thunk thunk)
    fun force p =
      case !p of
        Value v => v
      | Thunk th =>
          let
            val = th ()
            val _ = p := Value v
          in
            v
          end
  end
```

Thunks represent explicit emulation of lexically-scoped call-by-name semantics as shown by the following ML code:

```sml
fun if_by_name x y z =
  if x () then y () else z ()

fun fac n =
  if_by_name (fn () => n = 0)
             (fn () => 1)
             (fn () => n * (fac (n - 1)))
```

A lazy version of function application implemented in Scheme by [Barzilay and Clements (2005)](https://www2.ccs.neu.edu/racket/pubs/fdpe05-bc.pdf):

```scheme
(module lazy mzscheme
  (define-syntax (~app stx)
    (syntax-case stx (!)
      [(_ ! x) (syntax/loc stx (! x))]
      [(_ f x ...)
       (with-syntax ([(y ...) (generate-temporaries #'(x ...))])
         (~ (let ([p (! f)] [y x] ...)
              (if (lazy? p) (p y ...) (p (! y) ...)))))]))
  (define (~apply f . xs)
    (let ([f (! f)] [xs (!list (apply list* xs))])
      (apply f (if (lazy? f) xs (map ! xs)))))
  (provide (all-from-except mzscheme #%app apply)
           (rename ~app #%app)
           (rename ~apply apply)))
```

**Streams**{: style="color: red"}, or lazy lists, are a sequential data structure containing elements computed only on demand and cached afterwards in case of being needed again. A stream is either <code>null</code> or a pair with a stream in its <code>cdr</code>. Streams can be of infinite length since their elements are computed only when accessed. [Abelson and Sussman (1996)](https://web.mit.edu/6.001/6.037/sicp.pdf) implemented stream as a <code>cons</code> pair with a delayed object in its <code>cdr</code>:

```scheme
(cons-stream a b)
```

is equivalent to

```scheme
(cons a (delay b))
```

Accessor functions:

```scheme
(define (stream-car stream) (car stream))
(define (stream-cdr stream) (force (cdr stream)))
```

[Philip L. Bewig](https://sites.google.com/site/schemephil/) implemented *even* stream where the parity of the number of constructors in the stream is even:

```scheme
(define-syntax stream-cons
  (syntax-rules ()
    ((stream-cons obj strm)
      (stream-eager (make-stream-pair (stream-delay obj) (stream-lazy strm))))))
```

And also, an Racket implementation of basic stream operations that matches list operations by [David Liu](https://www.cs.toronto.edu/~lczhang/324/files/csc324_notes.pdf):

```racket
; Empty stream value, and check for empty stream.
(define s-null 's-null)
(define (s-null? stream) (equal? stream s-null))

#|
(s-cons <first> <rest>)
  <first>: A Racket expression
  <rest>: A stream (e.g., s-null or another s-cons expression)
  Returns a new stream whose first item is <first>, and whose other items are <rest>.
  Unlike a regular list, both <first> and <rest> are wrapped in a thunk, thereby not
  evaluated until called.
|#
(define-syntax s-cons
  (syntax-rules ()
    [(s-cons <first> <rest>)
     (cons (thunk <first>) (thunk <rest>))]))

; The stream equivalents of first and rest.
(define (s-first stream) ((car stream)))
(define (s-rest stream) ((cdr stream)))

; Make a stream.
(define-syntax make-stream
  (syntax-rules ()
    [(make-stream) s-null]
    [(make-stream <first> <rest> ...)
     (s-cons <first> (make-stream <rest> ...))]))

; 1. When the stream s is non-empty, we would like to:
; 1.1. Update the stream to s-rest of the stream;
; 1.2. Return the s-first of the stream.
; 2. When the stream s is empty:
; 2.1. Simply return the symbol 'DONE.
(define-syntax next!
  (syntax-rules ()
    [(next! <g>)
     (if (s-null? <g>)
        'DONE
        (let* ([tmp <g>])
          (begin
            (set! <g> (s-rest <g>))
            (s-first tmp))))]))
```

Below is a stream ADT implemented in Standard ML:

```sml
signature STREAM =
  sig
    type 'a stream
    val make : ('a * ('a -> 'a)) -> 'a stream
    val next : 'a stream -> ('a * 'a stream)
    val make2 : ('b * ('b -> 'a * 'b)) -> 'a stream
  end

structure Stream :> STREAM =
  struct
    datatype 'a stream = Cons of ('a * 'a stream) Thunk.thunk
    fun make (init : 'a, f : 'a -> 'a) : 'a stream =
      Cons (Thunk.make(fn () => (init, make (f init, f))))
    fun make2 (init : 'b, f: 'b -> 'a * 'b): 'a stream =
      Cons (Thunk.make(fn () =>
        let
          val (next_elem, next_state) = f init
        in
          (next_elem, make2 (next_state, f))
        end))
  end
```

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

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

## Thunk Utilities

Define a function <code>thunk?</code> which returns whether the specified parameter is a thunk or not:
```racket
(define (thunk? th)
  (if (and (procedure? th) (zero? (procedure-arity th))) #t #f))
```

Define a macro <code>thunk-that</code> which takes a parameter <code>e</code> and creates a thunk:
```racket
(define-syntax-rule (thunk-that e)
  (lambda () e))
```

Define a function <code>dethunk</code> which takes a thunk parameter <code>e</code> and returns the result of invoking <code>e</code>:
```racket
(define (dethunk-that thunk)
  (if (thunk? thunk) (thunk) (raise-argument-error 'thunk "thunk?" thunk)))
```

(It may seem unnecessary to use <code>dethunk-that</code> when implementing UW Homework 4 when we could simply <code>(thunk)</code>. However, a bit of verbosity can sometimes help in debugging a sea already full of parentheses.)

## Stream Utilities

Define a function <code>destream</code> which takes a <code>stream</code> parameter and evaluates to the resulting dethunked pair:
```racket
(define (destream stream)
  (let ([res (dethunk-that stream)])
    (if (pair? res)
        (if (thunk? (cdr res)) res
            (error "Wrong next-stream"))
        (error "Failed dethunking stream"))))
```

Define a function <code>cons-with-thunk-check-on-next-stream</code> which takes two parameters <code>value</code> and <code>next-stream</code>:
```racket
(define (cons-with-thunk-check-on-next-stream element next-stream)
  (cond
    [(thunk? next-stream) (cons element next-stream)]
    [else
     (raise-argument-error 'next-stream "thunk?" next-stream)]))
```

## UW Warm-Up Exercises for Racket

Write a function <code>sequence</code> that takes three arguments <code>low</code>, <code>high</code>, and <code>stride</code>, all assumed to be numbers, and produces a list of numbers from <code>low</code> to <code>high</code> (including <code>low</code> and possibly <code>high</code>) separated by <code>stride</code> and in sorted order. Further assume <code>stride</code> is positive.
```racket
(define (sequence low high stride)
  (if (< high low) '()
      (cons low (sequence (+ low stride) high stride))))
```

Write a function <code>string-append-map</code> that takes a list of strings and a string suffix and returns a list of strings. Each element of the output should be the corresponding element of the input appended with suffix (with no extra space between the element and suffix).
```racket
(define (string-append-map string-list suffix)
  (map (lambda (e) (string-append e suffix)) string-list))
```

Write a function <code>list-nth-mod</code> that takes a list and a number. If the number is negative, terminate the computation with <code>(error "list-nth-mod: negative number")</code>. Else if the list is empty, terminate the computation with <code>(error "list-nth-mod: empty list")</code>. Else return the <code>i</code>th element of the list where we count from zero and <code>i</code> is the remainder of the number divided by the length of the list.
```racket
(define (list-nth-mod xs n)
  (cond
    [(< n 0) (error "list-nth-mod: negative number")]
    [(empty? xs) (error "list-nth-mod: empty list")]
    [#t
     (let ([rem (remainder n (length xs))])
       (list-ref xs rem))]))
```

## Stream Processing

Write a function <code>stream-for-n-steps</code> that takes a stream <code>s</code> and a number <code>n</code>, and returns a list holding the first <code>n</code> values produced by <code>s</code> in order. Assume <code>n</code> is non-negative.
```racket
(define (stream-for-n-steps s n)
  (cond
    [(zero? n) '()]
    [else (let ([p (destream s)])
            (cons (car p) (stream-for-n-steps (cdr p) (- n 1))))]))
```

Write a stream <code>funny-number-stream</code> that is like the stream of natural numbers except numbers divisible by five are negated.
```racket
(define funny-number-stream
  (letrec ([f (lambda (x)
                (cons x (lambda ()
                          (if (= (remainder (+ x 1) 5) 0)
                              (f (- 0 (+ x 1)))
                              (f (+ (abs x) 1))))))])
    (lambda () (f 1))))
;
; Sample solution is better:
; (define funny-number-stream
;   (letrec ([f (lambda (n) (cons (if (= (remainder n 5) 0) (- n) n)
;                                 (lambda () (f (+ n 1)))))])
;     (lambda () (f 1))))
```

Write a stream <code>dan-then-dog</code>, where the elements of the stream alternate between the strings "<code>dan.jpg</code>" and "<code>dog.jpg</code>", starting with "<code>dan.jpg</code>".
```racket
(define dan-then-dog
  (letrec
      ([f (lambda (str ctr)
            (cons str
                  (lambda () (let ([new-ctr (+ ctr 1)])
                               (if (= (remainder new-ctr 2) 0)
                                   (f "dog.jpg" new-ctr)
                                   (f "dan.jpg" new-ctr))))))])
    (lambda () (f "dan.jpg" 1))))
;
; Sample solution is better:
; (define dan-then-dog
;   (letrec ([dan-st (lambda () (cons "dan.jpg" dog-st))]
;            [dog-st (lambda () (cons "dog.jpg" dan-st))])
;     dan-st))
;
; or even shorter:
;
; (define (dan-then-dog)
;   (cons "dan.jpg"
;         (lambda () (cons "dog.jpg" dan-then-dog))))
```

Write a function <code>stream-add-zero</code> that takes a stream <code>s</code> and returns another stream. If <code>s</code> would produce <code>v</code> for its <code>i</code>th element, then <code>stream-add-zero s</code> would produce the pair <code>'(0 . v)</code> for its <code>i</code>th element.
```racket
(define (stream-add-zero s)
  (letrec ([f (lambda (s)
                (let ([p (destream s)])
                  (cons-with-thunk-check-on-next-stream (cons 0 (car p))
                                                        (stream-add-zero (cdr p)))))])
    (lambda () (f s))))
;
; Sample solution:
; (define (stream-add-zero s)
;   (lambda ()
;     (let ([next (s)])
;       (cons (cons 0 (car next)) (stream-add-zero (cdr next))))))
```

Write a function <code>cycle-lists</code> that takes two lists and returns a stream. The elements produced by the stream are pairs where the first part is from the first input list and the second part is from the second input list. The stream cycles forever through the lists.
```racket
(define (cycle-lists xs ys)
  (letrec
      ([f (lambda (n)
            (cons (cons (list-nth-mod xs n)
                        (list-nth-mod ys n))
                  (lambda () (f (+ n 1)))))])
    (lambda () (f 0))))
```

## Memoization

> [(Frost, 1994)](https://dl.acm.org/doi/pdf/10.1145/181761.181764): The conventional notion of memoization involves a process by which a function is made to automatically memoize and subsequently recall all results computed.
> ...
> Memoization can result in an improvement in efficiency and in some cases an improvement in complexity owing to the fact that a memoized function never recomputes any result.

> [(Brown and Cook, 2009)](https://www.cs.utexas.edu/users/wcook/Drafts/2009/sblp09-memo-mixins.pdf): Memoization can be implemented in many ways. A function can be memoized by rewriting it to explicitly maintain its own memo table, but rewriting many functions this way is tedious and fails localize the common memoization behavior, which could otherwise allow modular memoization strategies.
> ...
> Lazy functional languages indirectly support a form of memoization as a side effect of the lazy evaluation strategy.

As an example (borrowed from [Cornell CS3110](https://www.cs.cornell.edu/courses/cs3110/)), consider the problem of computing the <code>n</code>th Fibonacci number <code>f(n) = f(n-1) + f(n-2)</code>:
```ocaml
let f(n) =
  if n < 2
  then 1
  else f(n-1) + f(n-2)
```
The key observation is that the recursive implementation is inefficient because it recomputes the same Fibonacci numbers over and over again. If we store the value of <code>f(n)</code> whenever we compute it in a table indexed by <code>n</code>, we can avoid this redundant work:
```ocaml
let fibm(n) =
  let memo: int option array = Array.create (n+1) None in
  let rec f_mem(n) =
    match memo.(n) with
    Some result -> result
       | None ->
          let result = if n < 2 then 1 else f_mem(n-1) + f_mem(n-2) in
            memo.(n) <- (Some result)
            result
  in
    f_mem(n)
```

Write a function <code>vector-assoc</code> that takes a value and a vector. It should behave like Racket’s <code>assoc</code> library function except:

1. it processes a vector (Racket’s name for an array) instead of a list,
2. it allows vector elements not to be pairs in which case it skips them, and
3. it always takes exactly two arguments.<br>

The function returns false if no vector element is a pair with a <code>car</code> field equal to the value, else returns the first pair with an equal <code>car</code> field. Process the vector elements in order starting from zero.

```racket
(define (vector-assoc v vec)
  (letrec
      ([f (lambda (n)
            (cond
              [(= n (vector-length vec)) #f]
              [else
               (let ([p (vector-ref vec n)])
                 (cond
                   [(pair? p) (if (equal? v (car p)) p (f (+ n 1)))]
                   [else (f (+ n 1))]))]))])
    (f 0)))
;
; Sample solution
; (define (vector-assoc v vec)
;   (letrec ([loop (lambda (i)
;                    (if (= i (vector-length vec))
;                        #f
;                        (let ([x (vector-ref vec i)])
;                          (if (and (cons? x) (equal? (car x) v))
;                              x
;                              (loop (+ i 1))))))])
;     (loop 0)))
```

Write a function <code>cached-assoc</code> that takes a list <code>xs</code> and a number <code>n</code> and returns a function that takes one argument <code>v</code> and returns the same thing that <code>(assoc v xs)</code> would return. However, an <code>n</code>-element cache of recent results should be used to possibly make this function faster than just calling <code>assoc</code> (if <code>xs</code> is long and a few elements are returned often). The cache must be a Racket vector of length <code>n</code> that is created by the call to <code>cached-assoc</code> (use Racket library function <code>vector</code> or <code>make-vector</code>) and used-and-possibly-mutated each time the function returned by <code>cached-assoc</code> is called. Assume <code>n</code> is positive.
```racket
(define (cached-assoc-mine xs n) ; cache slot 1 should be used for the 2 cache miss in a cache of size 3
  (letrec ([cache (make-vector n #f)]
           [ctr 0])
    (lambda (v)
      (cond
        [(vector-assoc v cache) (vector-assoc v cache)]
        [#t (let ([e (assoc v xs)])
              (if e ((lambda ()
                       (vector-set! cache ctr e)
                       (if (= ctr (- (vector-length cache) 1))
                           (set! ctr 0)
                           (+ ctr 1))
                       e))
                  #f))]))))
;
; Sample solution
(define (cached-assoc lst n)
  (let ([cache (make-vector n #f)]
        [next-to-replace 0])
    (lambda (v)
      (or (vector-assoc v cache)
          (let ([ans (assoc v lst)])
            (and ans
                 (begin (vector-set! cache next-to-replace ans)
                        (set! next-to-replace 
                              (if (= (+ next-to-replace 1) n)
                                  0
                                  (+ next-to-replace 1)))
                        ans)))))))
```

## Recursive Thunk

Define a macro that is used like "<code>while-less e1 do e2</code>" where <code>e1</code> and <code>e2</code> are expressions and <code>while-less</code> and <code>do</code> are syntax (keywords). The macro should do the following:

1. It evaluates <code>e1</code> exactly once.
2. It evaluates <code>e2</code> at least once.
3. It keeps evaluating <code>e2</code> until and only until the result is not a number less than the result of the evaluation of <code>e1</code>.
4. Assuming evaluation terminates, the result is true.
5. Assume <code>e1</code> and <code>e2</code> produce numbers.<br>

```racket
(define-syntax while-less
  (syntax-rules (do)
    [(while-less e1 do e2)
     (let ([condition e1])
       (letrec ([loop (lambda ()
                        (let ([body e2])
                          (if (or (not (number? body)) (>= body condition))
                              #t
                              (loop))))])
         (loop)))]))
```