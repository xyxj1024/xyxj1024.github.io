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