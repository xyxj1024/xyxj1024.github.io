---
layout: 		      post
title: 			      "Compile Standard ML to Lambda Language"
category:		      "Programming Languages"
tags:			        functional-programming
permalink:		    /def-sml/
last_modified_at: "2022-10-17"
---

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

$$B \textsf{ or } B_{\textsc{STAT}, B_{DYN}} \in \textsf{Basis} = \textsf{StaticBasis} \times \textsf{DynamicBasis}$$

Simple objects:
- $\alpha \textsf{ or } tyvar \in \textsf{TyVar}$ (type variables)
- $t \in \textsf{TyName}$ (type constructor names with arity $k \geq 0$)
- $is \in \textsf{IdStatus} = \{c, e, v\}$ (value variable, value constructor, or exception constructor&mdash;&mdash;identifier status descriptors)

An environment $E \textsf{ or } (SE, TE, VE) \in \textsf{Env} = \textsf{StrEnv} \times \textsf{TyEnv} \times \textsf{ValEnv}$:
- $SE \in \textsf{StrEnv} = \textsf{StrId} \overset{\textsf{fin}}{\rightarrow} \textsf{Env}$ (structure environment is the set of finite maps from structure identifiers to environments)
- $TE \in \textsf{TyEnv} = \textsf{TyCon} \overset{\textsf{fin}}{\rightarrow} \textsf{TyStr}$ (type environment is the set of finite maps from type constructors to type structures)
- $VE \in \textsf{ValEnv} = \textsf{VId} \overset{\textsf{fin}}{\rightarrow} \textsf{TypeScheme} \times \textsf{IdStatus}$ (variable environment is the set of finite maps from value identifiers to (type scheme, identifier status descriptor) pairs)

initial static basis $$B_{0} = T_{0}, F_{0}, G_{0}, E_{0},$$ where $F_{0} = \{\}$, $G_{0} = \{\}$, and $$T_{0} = \{ \texttt{bool}, \texttt{int}, \texttt{real}, \texttt{string}, \texttt{char}, \texttt{word}, \texttt{list}, \texttt{ref}, \texttt{exn} \}$$

initial static type environment $TE_{0}$.

$$E_{0} = (SE_{0}, TE_{0}, VE_{0}),$$ where $SE_{0} = \{\}$.

The compiler contains a number of modules associated with analyzing, compiling, and executing declarations. These are brought together in a single linkage module which evaluates top-level declarations. This top-level module of the compiler translates (core-level) declarations to lambda code which is fed through a simple optimizer and then executed. If successful, the execution results in a new set of dynamic bindings to add to the top-level environment.

When a declaration is compiled, each variable bound by the declaration is associated with a new, unique, lambda identifier (<code>lvar</code>).
The compiler environment maps SML variables to lambda identifiers.
The dynamic environment maps lambda identifiers to runtime objects.

Finally, the compiler environment and dynamic environment are incorporated into the top-level environment.

Lambda identifiers are bound in function expressions and recursion declarations and referenced by variable leaf nodes.

The declaration compiler has two purposes:
- to translate declarations in the SML core language into expressions in the lambda language;
- to generate a compiler environment (<code>CEnv</code>) for the variables bound in each declaration.

The compiler is a function <code>compileDec</code> with type:
```sml
CEnv -> dec -> (CEnv * (LambdaExp -> LambdaExp))
```
Every SML declaration is treated as if it had a *scope*. For top-level declarations, the scope does not actually exist, although each such declaration <code>dec</code> is treated as if it occured as part of an expression of the form:
```sml
let
  dec
in
  scope
end
```
The result of compiling a declaration is a <code>CEnv</code> for the declaration, plus a function which, when applied to a compiled *scope* for the declaration, returns a lambda expression corresponding to the declaration compiled locally for that scope.

A decision tree is an intermediate data structure which encompasses the decompositions and selections necessary to determine, for any data value, the topmost pattern which matches that value. For any data value, a decision tree performs an analysis which either delivers the first matching rule, or indicates a failure (if the value matches no rule). If a rule is matched, the decision tree also delivers the value environment for the pattern.