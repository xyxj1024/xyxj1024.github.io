---
layout:           post
title:            "A Note on Debt Overhang"
category:         "Money, Finance, Political Economy"
tags:             financial-economics investment asset-pricing
permalink:        /debt-overhang/
last_modified_at: "2022-11-22"
---

<p class="epigraph">
    [Critias]: As a matter of fact, I am almost ready to assert that this very thing, to know oneself, is temperance, and I am of the same mind as the person who put up an inscription to that effect at Delphi. (<em>Charmides 164d4</em>)<br />
</p>

Consider a typical firm's initial balance sheet:

|---
| **Assets** | **Liabilities**
|-:|:-
| Value of growth opportunity $V_{G}$ | $V_{D}$ Value of debt
|  | $V_{E}$ Value of equity
| Value of firm $V$ | $V$

Let us assume a no-arbitrage environment in which the total value of the firm depends on its total cach flows (this is based on Modigliani-Miller irrelevance theorem)[^1]. If the firm chooses to make a one-period investment $I$ at $t = 0$, it obtains an asset worth $V(s)$ at $t = 1$, where $s$ is the state of nature. The equilibrium market value of $V_{D}$ can be written as:

$$V_{D} = \int_{s_{a}}^{\infty} q(s) [\min\{V(s) - I, P\}]\,\mathrm{d}s,$$

<!-- excerpt-end -->

where:
- $V(s)$ is an increasing function of $s$ and $s_{a}$ is the threshold at which the firm makes zero profit on investment ($V(s_{a}) = I$);
- $q(s)$ is the equilibrium price of a monetary unit delivered at period $t = 1$ if and only if $s$ occurs;
- $P$ is the promised payment on an unsecured debt issued by the firm.

If for all $s$, $P > V(s) - I$,

$$V_{D} = V = \int_{s_{a}}^{\infty} q(s) [V(s) - I]\,\mathrm{d}s.$$

The firm's balance sheet appears as follows after exercising the investment option with $I$ amount of debt raised (in some literature, this may be seen as senior long-term debt since debt matures after the investment decision[^2]):

|---
| **Assets** | **Liabilities**
|-:|:-
| Value of newly acquired asset $V(s)$ | $$\min\{V(s),P\}$$ Value of debt
| | $$\max\{0,V(s)-P\}$$ Value of equity
| Value of firm $V(s)$ | $V(s)$

If for all $s$, $V(s) < I + P$, the firm will not exercise its growth option and the creditors will receive nothing. Otherwise, $$\min\{V(s), P\} = P$$,

$$V_{D} = \int_{s_{b}}^{\infty} P q(s)\,\mathrm{d}s < V,$$

where $s_{b}$ is the "breakeven" state such that $V(s_{b}) = I + P$. $V_{D}$ has an upper bound less than $V$, which is, in turn, less than $V = \int_{s_{a}}^{\infty} q(s) [V(s) - I]\,\mathrm{d}s$ without debt financing. Therefore, $V$ is a monotonically decreasing function of $P$. Optimal level of $V$ is reached when $P = V_{D} = 0$. In the case of $V(s) < P$, the gap between $V_{D}$ and $V(s)$ at $t = 1$ is often termed "debt overhang". This debt-overhang effect, first analyzed by Myers (1977)[^3], characterizes the situation when a firm's debt burden grows so large that default risk becomes high, distorting the shareholders' incentives to invest. Even profitable investment projects might be declined because much of the benefit from these projects accrues to creditors rather than shareholders. Therefore, a feasible investment must have a net present value (NPV) greater than the debt overhang.

In subsequent sections of this post, I would like to discuss different ways to extend this model.

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Some Preliminaries

### Probability Theory

The *sample space* $$\Omega$$ is a set of all possible outcomes $$\omega \in \Omega$$ of some random experiment. The *event space* $$\mathcal{F} \subset 2^{\Omega}$$ represents both the amount of information available as a result of the experiment conducted and the collection of all events of possible interest to us. *Probabilities* $$\mathbf{P}(A)$$ are assigned to $$A \in \mathcal{F}$$[^4].

**Definition 1.** We say that $$\mathcal{F}$$ is a $$\sigma$$-*algebra* if

- $$\Omega \in \mathcal{F}$$,
- If $$A \in \mathcal{F}$$ then $$\overline{A} \in \mathcal{F}$$, where $$\overline{A} = \Omega \backslash A$$,
- If $$A_{i} \in \mathcal{F}$$ for $$i = 1,2,3, \dots$$, then $$\bigcup_{i}A_{i} \in \mathcal{F}$$ and $$\overline{\big( \bigcup_{i} \overline{A_{i}} \big)} = \bigcap_{i}A_{i} \in \mathcal{F}$$.

**Definition 2.** A *measure space* is a triplet $$(\Omega, \mathcal{F}, \mu)$$, with $$\mu$$ a measure on the measurable space $$(\Omega, \mathcal{F})$$. A *probability space* is a measure space $$(\Omega, \mathcal{F}, \mathbf{P})$$ with $$\mathbf{P}$$ a probability measure.

**Definition 3.** We say that a measure space $$(\Omega, \mathcal{F}, \mu)$$ is *complete* if any subset $$N$$ of any $$B \in \mathcal{F}$$ with $$\mu(B) = 0$$ is also in $$\mathcal{F}$$. If $$\mu = \mathbf{P}$$ is a probability measure, we say that the probability space $$(\Omega, \mathcal{F}, \mathbf{P})$$ is a *complete probability space*.

**Definition 4.** A *filtration* $$(\mathcal{F}_{t})_{0 \leq t \leq \infty}$$ is a family of $$\sigma$$-algebras such that $$\mathcal{F}_{s} \subset \mathcal{F}_{t}$$ if $$s \leq t$$.

**Definition 5.** A *stochastic process* $$X$$ on $$(\Omega, \mathcal{F}, \mathbf{P})$$ is a collection of $$\mathbb{R}$$-valued or $$\mathbb{R}^{d}$$-valued random variables $$(X_{t})_{0 \leq t \leq \infty}$$. We say that $$X$$ is *adapted* if $$X_{t} \in \mathcal{F}_{t}$$ for each $$t$$.

**Definition 6.** The process $$N = (N_{t})_{0 \leq t \leq \infty}$$ defined by

$$N_{t} = \sum\limits_{n \geq 1} \mathbf{1}_{\{t \geq T_{n}\}}$$

with values in $$\mathbb{N} \cup \{\infty\}$$ is called the *counting process* (associated to the sequence $$(T_{n})_{n \geq 1}$$).

**Definition 7.** An adapted counting process $$N$$ is a *Poisson process* if
- for any $$s, t$$ satisfying $$0 \leq s < t < \infty$$, $$N_{t} - N_{s}$$ is independent of $$\mathcal{F}_{s}$$ (*increments independent of the past*);
- for any $$s, t, u, v$$ satisfying $$0 \leq s < t < \infty, 0 \leq u < v < \infty, t - s = v - u$$, then the distribution of $$N_{t} - N_{s}$$ is the same as that of $$N_{v} - N_{u}$$ (*stationary increments*).

We say that $$N_{t}$$ has the *Poisson distribution* with parameter $$\lambda t$$ if:

$$\mathbf{P}(N_{t} = n) = \frac{e^{-\lambda t}(\lambda t)^{n}}{n!}, \quad \lambda \geq 0, n = 0, 1, 2, \dots ,$$

where $$\lambda$$ is called the *intensity*, or arrival rate, of the process. A Poisson process $$N$$ with intensity $$\lambda$$ satisfies

$$\mathbf{E}[N_{t}] = \lambda t, \quad \mathbf{V}[N_{t}] = \lambda t.$$

The ratio, $$\frac{\mathbf{V}[N_{t}]}{\mathbf{E}[N_{t}]} = 1$$, is sometimes called the *dispersion index* of the Poisson process. By stationarity of the Poisson process, we may find, for all $$t > 0$$:

$$
\begin{align*}
    \mathbf{P}(N_{t + h} - N_{t} = 0) &= e^{-\lambda h} = 1 - \lambda h + o(h), \quad h \rightarrow 0,\\
    \mathbf{P}(N_{t + h} - N_{t} = 1) &= \lambda h e^{-\lambda h} \simeq \lambda h, \quad h \rightarrow 0,\\
    \mathbf{P}(N_{t + h} - N_{t} = 2) &\simeq h^{2}\frac{\lambda^{2}}{2} = o(h), \quad h \rightarrow 0.
\end{align*}
$$

```r
# Some R code that simulates a standard Poisson process
lambda = 0.6; T = 10; N = 1000 * lambda; dt = T * 1.0 / N
t = 0; s = c();
for (k in 1 : N) {
    if (runif(1) < lambda *dt) {
        s = c(s, t);
        t = t + dt;
    }
}
dev.new(width = T, height = 5)
plot(stepfun(s, cumsum(c(0, rep(1, length(s))))), 
             xlim = c(0, T), xlab = "t", ylab = expression('N'[t]),
             pch = 1, cex = 0.8, col = 'blue', lwd = 2, main = "",
             las = 1, cex.axis = 1.2, cex.lab = 1.4)
```

**Definition 8.** Let $$(Z_{k})_{k \geq 1}$$ denote a sequence of i.i.d. square-integrable random variables and be represented as $$Z$$ with probability distribution $$f(\mathrm{d}y)$$ on $$\mathbb{R}$$, independent of the Poisson process $$(N_{t})_{0 \leq t \leq \infty}$$. The process $$(Y_{t})_{0 \leq t \leq \infty}$$ given by the random sum:

$$Y_{t} \equiv Z_{1} + Z_{2} + \cdots + Z_{N_{t}} = \sum\limits_{k = 1}^{N_{t}} Z_{k}$$

is called a *compound Poisson process*. The *jump size* of $$(Y_{t})_{0 \leq t \leq \infty}$$ is written as:

$$\Delta Y_{t} \equiv Y_{t} - Y_{t^{-}} = Z_{N_{t}} \Delta N_{t},$$

where $$Y_{t^{-}}$$ is the left limit and $$\Delta N_{t} \equiv N_{t} - N_{t^{-}} \in \{0, 1\}$$ is the jump size of the standard Poisson process $$(N_{t})_{0 \leq t \leq \infty}$$[^5].

```r
# R code that simulates a sample path of a compound Poisson process
N <- 50; Tk <- cumsum(rexp(N, rate = 0.5)); Zk <- cumsum(c(0, rexp(N, rate = 0.5)))
plot(stepfun(Tk, Zk), xlim = c(0, 10), do.points = F, main = "L = 0.5", col = "blue")
Zk <- cumsum(c(0, rnorm(N, mean = 0, sd = 1)))
plot(stepfun(Tk, Zk), xlim = c(0, 10), do.points = F, main = "L = 0.5", col = "blue")
```

**Definition 9.** An adapted process $$B = (B_{t})_{0 \leq t < \infty}$$ taking values in $$\mathbb{R}^{n}$$ is called an $$n$$-*dimensional Brownian motion* if
- for $$0 \leq s < t < \infty$$, $$B_{t} - B_{s}$$ is independent of $$\mathcal{F}_{s}$$ (*increments independent of the past*);
- for $$0 < s < t$$, $$B_{t} - B_{s}$$ is a Gaussian random variable with mean zero and variance matrix $$(t - s)C$$, given a non-random matrix $$C$$[^6].

**Definition 10.** An adapted process $$X$$ with values in $$\mathbb{R}^{n}$$ is a *diffusion* if it has continuous sample paths and if it satisfies the strong Markov property.

> An intuitive notion of a diffusion is to imagine a pollen grain floating downstream in a river. The grain is subject to two forces: the current of the river (drift), and the aggregate bombardment of the grain by the surrounding water molecules (diffusion).

### Game Theory

Please refer to Maskin (2001)[^7] for more details.

Let $$G$$ be a game with $$n$$ players and $$T$$ periods. In every period $$t \in \{1, \dots , T\}$$, each player $$i \in \{1, \dots , n\}$$ chooses an action $$a_{t}^{i}$$ in his/her finite action space. Let $$\mathbf{a}_{t} \equiv (a_{t}^{1} , \dots , a_{t}^{n})$$ and $$\mathbf{a} \equiv (\mathbf{a}_{1}, \dots , \mathbf{a}_{T})$$. The *history* in period $$t$$ is the sequence of actions chosen before period $$t$$:

$$h_{t} \equiv (\mathbf{a}_{1}, \dots , \mathbf{a}_{t - 1}).$$

(*future* $$f_{t}$$ can be similarly defined.) Player $$i$$ has von Neumann-Morgenstern preferences represented by the utility function:

$$u^{i}(\mathbf{a}) = u^{i}(h_{t}, f_{t}).$$

Denote the set of all possible period $$t$$ histories by $$H_{t}$$. A (behavior) *strategy* $$s^{i}$$ for player $$i$$ is a function that, for all $$t$$ and each history $$h_{t} \in H_{t}$$, assigns a probability distribution to the action space $$A_{t}^{i}(h_{t})$$. Also, denote a *collection of partitions* $$\{H_{t}(\cdot)\}_{t = 1}^{T}$$ by $$H.(\cdot)$$ where $$H_{t}(h_{t}) \subset H_{t}$$. We shall call collection $$H.'(\cdot)$$ *weakly coarser* (*weakly finer*) than collection $$H.(\cdot)$$, if, for all $$t$$, either $$H_{t}'(\cdot)$$ is coarser (finer) than $$H_{t}(\cdot)$$ or $$H_{t}(\cdot) = H_{t}'(\cdot)$$.

**Definition 11.** $$\overline{H}.(\cdot)$$ is called the collection of players' *action-space-invariant partitions* if for all $$i, t, h_{t}$$,

$$h_{t}' \in H_{t}, h_{t}' \in \overline{H}_{t}(h_{t}) \Leftrightarrow S_{t}^{i}(h_{t}) = S_{t}^{i}(h_{t}').$$

If the collection $$H.^{i}(\cdot)$$ is weakly finer than $$\overline{H}.(\cdot)$$, then strategy $$s^{i}$$ is *measurable* with respect to $$H.^{i}(\cdot)$$ if, for all $$t, h_{t}, h_{t}' \in H_{t}^{i}(h_{t})$$:

$$s^{i}(h_{t}') = s^{i}(h_{t}).$$

**Definition 12.** A *subgame-perfect equilibrium (SPE)* is a strategy vector $$\mathbf{s}$$ that forms a Nash equilibrium after any history; i.e., for all $$t, h_{t} \in H_{t}, i$$, and alternative strategy $$\hat{s}^{i}$$:

$$v^{i}(s^{i}, \mathbf{s}^{-i} \mid h_{t}) \geq v^{i}(\hat{s}^{i}, \mathbf{s}^{-i} \mid h_{t}),$$

where $$\mathbf{s}^{-i}$$ denotes the vector of strategies by players other than player $$i$$ and $$v^{i}(\cdot)$$ is player $$i$$'s expected payoff.

We shall call the vector of collections $$\big( H.^{1}(\cdot) , \dots , H.^{n}(\cdot) \big)$$ *consistent* if, for all $$i$$:
- $$H.^{i}(\cdot)$$ is weakly finer than $$\overline{H}.(\cdot)$$,
- for all $$s^{-i} \in \prod\limits_{k \not = i} S^{k} (H.^{k}(\cdot))$$, for all $$t$$, for all $$h_{t}, h_{t}' \in H_{t}$$ such that $$h_{t}' \in H_{t}^{i}(h_{t})$$:

$$v^{i}(\cdot, \mathbf{s}^{-i} \mid h_{t}) \sim v^{i}(\cdot, \mathbf{s}^{-i} \mid h_{t}').$$

**Definition 13.** The game $$G$$ is *simultaneous-nondegenerate* if, holding some future sequence of random actions fixed, in any period and given any two histories $$h_{t}$$ and $$h_{t}'$$, and any active player $$i$$, any other active player $$j$$ moving simultaneously can ensure that $$i$$'s decision problem after $$h_{t}$$ differs from that after $$h_{t}'$$.

If a game is simultaneous-nondegenerate, then there exists a unique maximally coarse consistent collection $$H.^{*}(\cdot)$$ such that $$H.^{*}(h_{t})$$ constitutes the state of the system or the payoff-relevant history. If we do not impose simultaneous-nondegeneracy, there is a unique maximally coarse consistent vector of collections

$$H.^{i*}(\cdot), \quad i = 1, \dots , n.$$

**Definition 14.** A strategy is *Markovian* if it is measurable with respect to $$H.^{*}(\cdot)$$.

**Definition 15.** A *Markov Perfect Equilibrium (MPE)* is a SPE in which all players use Markov strategies.

## The Effect of Maturity

### Optimal Level of Short-Term vs. Long-Term Debt

Let us assume that:
- investors are risk-neutral,
- the risk-free interest rate is zero,
- the financial structure of a typical firm is chosen at $$t = 0$$ so as to maximize the *aggregate* return of the shareholders,
- the only way to prevent the manager of the firm from investing in a bad project is to make necessary funds unavailable (management has *empire-buidling* motive),
- the probability distributions of liquidation, investment costs, returns on assets and investments are all common knowledge, and values are revealed at $$t = 1$$,
- $$y_{1} < i$$ and $$y_{2} \geq L$$ with probability $$1$$.

<br>

|---
| $$t = 0$$ | $$t = 1$$ | $$t = 2$$
|:-:|:-:
| Old assets-in-place | Old assets yield return $$y_{1}$$; <br /> Decision whether to make new investment of size $$i$$; <br /> Decision whether to liquidate and realize $$L$$ | Old assets yield return $$y_{2}$$; <br /> New investment, if taken, yields return $$r$$

Firm's management will invest at $$t = 1$$ if and only if

$$y_{1} + y_{2} + r - d_{1} - d_{2} \geq i,$$

where $$d_{1}$$ is the amount owed at $$t = 1$$ and $$d_{2}$$ the amount owed at $$t = 2$$, i.e., the face values of short-term and long-term debt, respectively. Without available funds, the firm can still operate as long as
- Debt at $$t = 1$$ can be paid out of current earnings: $$y_{1} \geq d_{1}$$, or
- Debt at $$t = 1$$ can be paid out of future earnings: $$y_{1} + y_{2} \geq d_{1} + d_{2}$$.

In either case, the total return to creditors at $$t = 0$$ is $$R = y_{1} + y_{2}$$; Otherwise, the firm is forced to liquidation at $$t = 1$$ with $$R = y_{1} + L$$. If firm's management chooses to maximize the market value at $$t = 0$$, then the optimal level of $$d_{1}$$ is zero.

Given $$d_{1} = 0$$ and thus $$L$$ is irrelevant,
1. If $$r > i$$ with probability $$1$$, then the first-best outcome can be achieved by letting $$d_{2} = 0$$ (this is all-equity financing);
2. If $$r < i$$ with probability $$1$$, then the first-best outcome can be achieved by setting $$d_{2}$$ large enough;
3. If the sum of $$y_{1}$$ and $$y_{2}$$ is constant with probability $$1$$, then the first-best outcome can be achieved by letting $$d_{2} = y_{1} + y_{2}$$;
4. Finally, if $$i$$ and $$y_{1}$$ are deterministic, and $$r \equiv g(y_{2})$$ where $$g(\cdot)$$ is a strictly increasing function, then the first-best outcome can be achieved by letting $$d_{2} = y_{1} + g^{-1}(i)$$[^2].

What if the risk-free rate is non-zero? We may assume that the firm issues perpetual callable coupon debt and the face value of the debt $$F$$ is equal to the value of perpetual debt with periodic payment (coupon) $$d$$ discounted at the risk-free rate $$r$$:

$$F = \frac{d}{r}.$$

The firm has to rollover its maturing debt at the current market price:

$$w \cdot D(p, A, d),$$

where
- $$A$$ is the value of the firm's fixed (tangible) assets,
- $$p$$ is the unit market price of the product produced and sold by the firm,
- $$w$$ is the debt repurchase (or retirement) rate.

The net refinancing expenses is given by:

$$w \cdot F - w \cdot D(p, A, d) = w \cdot [F - D(p, A, d)] > 0.$$

The net instantaneous cash payment to the creditors is $$d$$ plus net refinancing expenses[^8].

### Black-Scholes-Merton Setting

At any future time $$t > 0$$, the value of a typical firm's assets-in-place follows a log-normal diffusion (and thus follows a martingale):

$$V_{t} = V_{0} \cdot e^{-\frac{\sigma^{2}}{2}t + \sigma Z_{t}},$$

where $$V_{0}$$ is the current market value, volatility $$\sigma$$ is a constant, and $$Z_{t} \sim N(0, t)$$. Consider a three-date model. The standard Black-Scholes calculation gives the corresponding debt value at $$t = 0$$ as:

$$D(V_{0}; F_{i}, m_{i}) = V_{0}(1 - N(d_{i})) + F_{i}N(d_{i} - \sigma \sqrt{m_{i}}),$$

where $$F_{t}$$ is the face value of a zero-coupon debt issue that matures at time $$t$$, $$m_{t}$$ is the maturity, $$d_{i} \equiv \frac{\ln (V_{0} / F_{i}) + 0.5 \sigma^{2}m_{i}}{\sigma \sqrt{m_{i}}}$$, $$i = 1, 2$$. Debt overhang is given by:

$$D_{V} \equiv \frac{\partial D(V_{0}; F, m)}{\partial V_{0}}.$$

It can be proved that for a given initial debt market value, long-term debt imposes stronger overhang than short-term debt; that is, $$D_{V}(V_{0}; F_{2}, m_{2}) > D_{V}(V_{0}; F_{1}, m_{1})$$ whenever $$D(V_{0}; F_{2}, m_{2}) = D(V_{0}; F_{1}, m_{1})$$[^9].

## Dynamic Model of Investment

### The Leverage Ratchet Effect

> Intuitively, by reducing leverage, shareholders transfer wealth to existing creditors. Conversely, if shareholders can raise new debt of equal seniority to fund a payout for themselves, wealth is transferred in the other direction and shareholders benefit at the expense of existing creditors[^10].

The leverage ratchet effect states that:
1. No matter how large the potential gain from leverage reduction to the total value of the firm, shareholders could resist it;
2. Shareholders could gain from a one-time debt issuance even when new debt must bear junior priority, unless the tax benefit of debt has been fully exhausted.

DeMarzo and He (2021)[^11] considered a typical firm with (*exogenous*) EBIT rate of $$Y_{t}$$ generated from its assets-in-place, which evolves according to:

$$\mathrm{d}Y_{t} = \mu(Y_{t})\,\mathrm{d}t + \sigma(Y_{t})\,\mathrm{d}Z_{t} + \zeta(Y_{t^{-}})\,\mathrm{d}N_{t},$$

where the drift $$\mu(Y_{t})$$ and the volatility $$\sigma(Y_{t})$$ satisfy standard regularity conditions, $$\mathrm{d}Z_{t}$$ is the increment of a standard Brownian motion, $$\mathrm{d}N_{t}$$ is an independent Poisson increment with intensity $$\lambda(Y_{t}) > 0$$, and $$\xi(Y_{t^{-}})$$ is the jump size.

Let the cumulative debt issuance process $$\Gamma_{t}$$ be right-continuous, left-limit, and measurable with respect to the filtration generated by the operating cash flow $$Y_{t}$$: $$(Y_{s})_{0 \leq s \leq t}$$. Denote the aggregate face value of outstanding debt by $$F_{t}$$, which has a constant coupon rate of $$c > 0$$. Assume debt takes the form of exponentially maturing coupon bonds with a constant amortization rate $$\xi > 0$$. Combining interest and principal, equity holders are required to pay creditors a total flow payment of $$(c + \xi)F_{t}\,\mathrm{d}t$$ to avoid default. Assuming zero transaction costs in issuing (or repurchasing) debt, the evolution of the outstanding face value of $$F_{t}$$ is given by:

$$\mathrm{d}F_{t} = \mathrm{d}\Gamma_{t} - \xi F_{t}\,\mathrm{d}t.$$

Over the time interval $$\mathrm{d}t$$, without tax payment, the net cash flows to equity holders are

$$\big( Y_{t} - (c + \xi)F_{t} \big)\,\mathrm{d}t + p_{t}\,\mathrm{d}\Gamma_{t},$$

where $$p_{t}$$ is the (*endogenous*) debt price per unit of the promised face value.

Adding a corporate tax regime that strictly increases with the firm's profit net of interest such that the firm pays

$$\pi(Y_{t} - cF_{t})\,\mathrm{d}t$$

over the period $$[t, t + \mathrm{d}t]$$, Proposition 1 of the paper can be concluded as: In any MPE given the state variable pair $$(Y, F)$$, $$\Gamma_{t}$$ is a monotonically increasing process, i.e., the firm will never repurchase debt. The equity value function $$V(Y, F)$$ is convex and decreasing in $$F$$, with the debt price as a subgradient also decreasing in $$F$$:

$$p(Y, F) \in -\partial_{F}V(Y, F).$$

## References

[^1]: Stephen A. Ross, Comment on the Modigliani-Miller Propositions, *Journal of Economic Perspectives* **2**(4), 127-133 (1988).

[^2]: Oliver Hart and John Moore, Debt and Seniority: An Analysis of the Role of Hard Claims in Constraining Management, *The American Economic Review* **85**(3), 567-585 (1995).

[^3]: Stewart C. Myers, Determinants of Corporate Borrowing, *Journal of Financial Economics* **5**, 147-175 (1977).

[^4]: Amir Dembo, *Lecture Notes of "Probability Theory: STAT310/MATH230"* at Stanford University, [https://web.stanford.edu/~montanar/TEACHING/Stat310A/lnotes.pdf](https://web.stanford.edu/~montanar/TEACHING/Stat310A/lnotes.pdf).

[^5]: Nicolas Privault, *Lecture Notes on Stochastic Finance* at Nanyang Technological University, [https://personal.ntu.edu.sg/nprivault/indext.html](https://personal.ntu.edu.sg/nprivault/indext.html).

[^6]: Philip E. Protter, *Stochastic Integration and Differential Equations*, Second Edition, Springer-Verlag Berlin Heidelberg, 2004.

[^7]: Eric Maskin, Markov Perfect Equilibrium, *Journal of Economic Theory* **100**, 191-219 (2001).

[^8]: Sheridan Titman and Sergey Tsyplakov, A Dynamic Model of Optimal Capital Structure, *Review of Finance* **11**(3), 401-451 (2007).

[^9]: Douglas W. Diamond and Zhiguo He, A Theory of Debt Maturity: The Long and Short of Debt Overhang, *The Journal of Finance* **69**(2), 719-762 (2014).

[^10]: Anat R. Admati, Peter M. DeMarzo, Martin F. Hellwig, and Paul Pfleiderer, The Leverage Ratchet Effect, *The Journal of Finance* **73**(1), 145-198 (2018).

[^11]: Peter M. DeMarzo and Zhiguo He, Leverage Dynamics without Commitment, *The Journal of Finance* **76**(3), 1195-1250 (2021).