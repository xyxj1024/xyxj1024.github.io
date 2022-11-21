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
|:-:|:-:|
| **Assets** | **Liabilities**
|-:|:-
| Value of growth opportunity $V_{G}$ | $V_{D}$ Value of debt
|  | $V_{E}$ Value of equity
| Value of firm $V$ | $V$

Let us assume a no-arbitrage environment in which the total value of a typical firm depends on its total cach flows[^1]. If the firm chooses to make a one-period investment $I$ at $t = 0$, it obtains an asset worth $V(s)$ at $t = 1$, where $s$ is the state of nature. The equilibrium market value of $V_{D}$ can be written as:

$$V_{D} = \int_{s_{a}}^{\infty} q(s) [\min\{V(s) - I, P\}]\,\mathrm{d}s,$$

where:
- $V(s)$ is an increasing function of $s$ and $s_{a}$ is the threshold at which the firm makes zero profit on investment;
- $q(s)$ is the equilibrium price of a monetary unit delivered at period $t = 1$ if and only if $s$ occurs;
- $P$ is the promised payment on an unsecured debt issued by the firm.

If for all $s$, $P > V(s) - I$,

$$V_{D} = V = \int_{s_{a}}^{\infty} q(s) [V(s) - I]\,\mathrm{d}s.$$

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## References

[^1]: Stephen A. Ross, "Comment on the Modigliani-Miller Propositions," *Journal of Economic Perspectives*, Volume 2, Number 4---Fall 1988---Pages 127-133.