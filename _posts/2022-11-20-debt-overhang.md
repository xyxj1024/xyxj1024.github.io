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

The firm's balance sheet appears as follows after exercising the investment option with $I$ amount of funds raised:

|---
| **Assets** | **Liabilities**
|-:|:-
| Value of newly acquired asset $V(s)$ | $\min\{V(s), P\}$ Value of debt
|  | $\max\{0, V(s) - P\}$ Value of equity
| Value of firm $V(s)$ | $V(s)$

If for all $s$, $V(s) < I + P$, the firm will not exercise its growth option and the creditors will receive nothing. Otherwise, $\min\{V(s), P\} = P$,

$$V_{D} = \int_{s_{b}}^{\infty} P q(s)\,\mathrm{d}s < V,$$

where $s_{b}$ is the "breakeven" state such that $V(s_{b}) = I + P$. $V_{D}$ has an upper bound less than $V$, which is, in turn, less than $V = \int_{s_{a}}^{\infty} q(s) [V(s) - I]\,\mathrm{d}s$ without debt financing. Therefore, $V$ is a monotonically decreasing function of $P$. Optimal level of $V$ is reached when $P = V_{D} = 0$. In the case of $V(s) < P$, the gap between $V_{D}$ and $V(s)$ at $t = 1$ is often termed "debt overhang". This debt-overhang effect, first analyzed by Myers (1977)[^2], characterizes the situation when a firm seeking equity value maximization might decline a profitable investment opportunity. A feasible investment must have a net present value (NPV) greater than the debt overhang.

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## References

[^1]: Stephen A. Ross, "Comment on the Modigliani-Miller Propositions," *Journal of Economic Perspectives*, Volume 2, Number 4---Fall 1988---Pages 127-133.

[^2]: Stewart C. Myers, "Determinants of Corporate Borrowing," *Journal of Financial Economics* 5 (1977) 147-175.