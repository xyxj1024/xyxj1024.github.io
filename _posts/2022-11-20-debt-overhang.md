---
layout:           post
title:            "A Note on Debt Overhang"
category:         "Money, Finance, Political Economy"
tags:             macroeconomics corporate-finance
permalink:        /debt-overhang/
last_modified_at: "2022-11-22"
---

Consider a simple, initial balance sheet of a firm:

|---
| **Assets** | **Liabilities**
|-:|:-
| Value of growth opportunity $V_{G}$ | $V_{D}$ Value of debt
|  | $V_{E}$ Value of equity
| Value of firm $V$ | $V$

Let us assume a no-arbitrage environment in which the total value of a typical firm depends on its total cach flows[^1]. If the firm chooses to make a one-period investment $I$ at $t = 0$, it obtains an asset worth $V(s)$ at $t = 1$, where $s$ is the state of nature. The equilibrium market value of $V_{D}$ can be written as:

$$V_{D} = \int_{s_{a}}^{\infty} q(s) [\min\{V(s) - I, P\}]\,\mathrm{d}s,$$

<!-- excerpt-end -->

where:
- $V(s)$ is an increasing function of $s$ and $s_{a}$ is the threshold at which the firm makes zero profit on investment ($V(s_{a}) = I$);
- $q(s)$ is the equilibrium price of a monetary unit delivered at period $t = 1$ if and only if $s$ occurs;
- $P$ is the promised payment on an unsecured debt issued by the firm.

If for all $s$, $P > V(s) - I$,

$$V_{D} = V = \int_{s_{a}}^{\infty} q(s) [V(s) - I]\,\mathrm{d}s.$$

The firm's balance sheet appears as follows after exercising the investment option with $I$ amount of debt raised (assume debt matures after the investment opportunity):

|---
| **Assets** | **Liabilities**
|-:|:-
| Value of newly acquired asset $V(s)$ | $$\min\{V(s),P\}$$ Value of debt
| | $$\max\{0,V(s)-P\}$$ Value of equity
| Value of firm $V(s)$ | $V(s)$

If for all $s$, $V(s) < I + P$, the firm will not exercise its growth option and the creditors will receive nothing. Otherwise, $$\min\{V(s), P\} = P$$,

$$V_{D} = \int_{s_{b}}^{\infty} P q(s)\,\mathrm{d}s < V,$$

where $s_{b}$ is the "breakeven" state such that $V(s_{b}) = I + P$. $V_{D}$ has an upper bound less than $V$, which is, in turn, less than $V = \int_{s_{a}}^{\infty} q(s) [V(s) - I]\,\mathrm{d}s$ without debt financing. Therefore, $V$ is a monotonically decreasing function of $P$. Optimal level of $V$ is reached when $P = V_{D} = 0$. In the case of $V(s) < P$, the gap between $V_{D}$ and $V(s)$ at $t = 1$ is often termed "debt overhang". This debt-overhang effect, first analyzed by Myers (1977)[^2], characterizes the situation when a firm's debt burden grows so large that default risk becomes high, distorting the shareholders' incentives to invest. Even profitable investment projects might be declined because much of the benefit from these projects accrues to creditors rather than shareholders. Therefore, a feasible investment must have a net present value (NPV) greater than the debt overhang.

Later on in this post, I would like to discuss different ways to extend this model.

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Some Preliminaries

The *sample space* $$\Omega$$ is a set of all possible outcomes $$\omega \in \Omega$$ of some random experiment. The *event space* $$\mathcal{F} \subset 2^{\Omega}$$ represents both the amount of information available as a result of the experiment conducted and the collection of all events of possible interest to us. Probabilities $$\mathbf{P}(A)$$ are assigned to $$A \in \mathcal{F}$$[^3].

**Definition 1.** We say that $$\mathcal{F}$$ is a $$\sigma$$-algebra if

- $$\Omega \in \mathcal{F}$$,
- If $$A \in \mathcal{F}$$ then $$\overline{A} \in \mathcal{F}$$, where $$\overline{A} = \Omega \backslash A$$,
- If $$A_{i} \in \mathcal{F}$$ for $$i = 1,2,3, \dots$$, then $$\bigcup_{i}A_{i} \in \mathcal{F}$$ and $$\overline{\big( \bigcup_{i} \overline{A_{i}} \big)} = \bigcap_{i}A_{i} \in \mathcal{F}$$.

**Definition 2.** A measure space is a triplet $$(\Omega, \mathcal{F}, \mu)$$, with $$\mu$$ a measure on the measurable space $$(\Omega, \mathcal{F})$$. A probability space is a measure space $$(\Omega, \mathcal{F}, \mathbf{P})$$ with $$\mathbf{P}$$ a probability measure.

**Definition 3.** We say that a measure space $$(\Omega, \mathcal{F}, \mu)$$ is complete if any subset $$N$$ of any $$B \in \mathcal{F}$$ with $$\mu(B) = 0$$ is also in $$\mathcal{F}$$. If $$\mu = \mathbf{P}$$ is a probability measure, we say that the probability space $$(\Omega, \mathcal{F}, \mathbf{P})$$ is a complete probability space.

**Definition 4.** A filtration $$(\mathcal{F}_{t})_{0 \leq t \leq \infty}$$ is a family of $$\sigma$$-algebras such that $$\mathcal{F}_{s} \subset \mathcal{F}_{t}$$ if $$s \leq t$$.

**Definition 5.** A stochastic process $$X$$ on $$(\Omega, \mathcal{F}, \mathbf{P})$$ is a collection of $$\mathbb{R}$$-valued or $$\mathbb{R}^{d}$$-valued random variables $$(X_{t})_{0 \leq t \leq \infty}$$. We say that $$X$$ is adapted if $$X_{t} \in \mathcal{F}_{t}$$ for each $$t$$.

**Definition 6.** The process $$N = (N_{t})_{0 \leq t \leq \infty}$$ defined by

$$N_{t} = \sum\limits_{n \geq 1} \mathbf{1}_{\{t \geq T_{n}\}}$$

with values in $$\mathbb{N} \cup \{\infty\}$$ is called the counting process (associated to the sequence $$(T_{n})_{n \geq 1}$$).

**Definition 7.** An adapted counting process $$N$$ is a Poisson process if
- for any $$s, t$$ satisfying $$0 \leq s < t < \infty$$, $$N_{t} - N_{s}$$ is independent of $$\mathcal{F}_{s}$$ (*increments independent of the past*);
- for any $$s, t, u, v$$ satisfying $$0 \leq s < t < \infty, 0 \leq u < v < \infty, t - s = v - u$$, then the distribution of $$N_{t} - N_{s}$$ is the same as that of $$N_{v} - N_{u}$$ (*stationary increments*).

We say that $$N_{t}$$ has the Poisson distribution with parameter $$\lambda t$$ if:

$$\mathbf{P}(N_{t} = n) = \frac{e^{-\lambda t}(\lambda t)^{n}}{n!}, \quad \lambda \geq 0, n = 0, 1, 2, \dots ,$$

where $$\lambda$$ is called the intensity, or arrival rate, of the process. A Poisson process $$N$$ with intensity $$\lambda$$ satisfies

$$\mathbf{E}[N_{t}] = \lambda t, \quad \mathbf{V}[N_{t}] = \lambda t.$$

**Definition 8.** An adapted process $$B = (B_{t})_{0 \leq t < \infty}$$ taking values in $$\mathbb{R}^{n}$$ is called an $$n$$-dimensional Brownian motion if
- for $$0 \leq s < t < \infty$$, $$B_{t} - B_{s}$$ is independent of $$\mathcal{F}_{s}$$ (*increments independent of the past*);
- for $$0 < s < t$$, $$B_{t} - B_{s}$$ is a Gaussian random variable with mean zero and variance matrix $$(t - s)C$$, given a non-random matrix $$C$$[^4].

## References

[^1]: Stephen A. Ross, "Comment on the Modigliani-Miller Propositions," *Journal of Economic Perspectives*, Volume 2, Number 4---Fall 1988---Pages 127-133.

[^2]: Stewart C. Myers, "Determinants of Corporate Borrowing," *Journal of Financial Economics* 5 (1977) 147-175.

[^3]: Amir Dembo, Lecture Notes of "Probability Theory: STAT310/MATH230" at Stanford University, [https://web.stanford.edu/~montanar/TEACHING/Stat310A/lnotes.pdf](https://web.stanford.edu/~montanar/TEACHING/Stat310A/lnotes.pdf).

[^4]: Philip E. Protter, "Stochastic Integration and Differential Equations," Second Edition, Springer-Verlag Berlin Heidelberg, 2004.