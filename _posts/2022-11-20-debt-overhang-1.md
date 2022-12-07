---
layout:           post
title:            "A Note on Debt Overhang - Part I"
category:         "Money, Finance, Political Economy"
tags:             stochastic-mathematics financial-economics asset-pricing
permalink:        /debt-overhang-1/
last_modified_at: "2022-11-25"
---

<p class="message"><em>
    Critias: "As a matter of fact, I am almost ready to assert that this very thing, to know oneself, is temperance, and I am of the same mind as the person who put up an inscription to that effect at Delphi."
</em>(Charmides 164d4)</p>

Consider a typical firm's initial balance sheet:

|---
| **Assets** | **Liabilities**
|-:|:-
| Value of growth opportunity<br /> $V_{G}$ | Value of debt<br /> $V_{D}$
|  | Value of equity<br /> $V_{E}$
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

The firm's balance sheet appears as follows after exercising the investment option with $I$ amount of debt raised (in some literature, this may be seen as senior long-term debt since debt matures after the investment decision):

|---
| **Assets** | **Liabilities**
|-:|:-
| Value of newly acquired asset<br /> $V(s)$ | Value of debt<br /> $$\min\{V(s),P\}$$
| | Value of equity<br /> $$\max\{0,V(s)-P\}$$
| Value of firm $V(s)$ | $V(s)$

If for all $s$, $V(s) < I + P$, the firm will not exercise its growth option and the creditors will receive nothing. Otherwise, $$\min\{V(s), P\} = P$$,

$$V_{D} = \int_{s_{b}}^{\infty} P q(s)\,\mathrm{d}s < V,$$

where $s_{b}$ is the "breakeven" state such that $V(s_{b}) = I + P$. $V_{D}$ has an upper bound less than $V$, which is, in turn, less than $V = \int_{s_{a}}^{\infty} q(s) [V(s) - I]\,\mathrm{d}s$ without debt financing. Therefore, $V$ is a monotonically decreasing function of $P$. Optimal level of $V$ is reached when $P = V_{D} = 0$. In the case of $V(s) < P$, the gap between $V_{D}$ and $V(s)$ at $t = 1$ is often termed "**debt overhang**{: style="color: red"}". This debt-overhang effect, first analyzed by Myers (1977)[^2], characterizes the situation when a firm's debt burden grows so large that default risk becomes high, distorting the shareholders' incentives to invest. Even profitable investment projects might be declined because much of the benefit from these projects accrues to creditors rather than shareholders. A feasible investment must have a net present value (NPV) greater than the debt overhang.

This is the first post of a series about the debt-overhang effect, which, I believe, poses fundamental threat to our modern economy. Classic academic literature from intellectual fields such as corporate finance, financial economics, and macroeconomics, etc. will be revisited and discussed. In subsequent sections of this post, I would like to summarize some advanced mathematics required to understand the literature and take a brief look at the principal-agent problem associated with debt overhang.

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Some Preliminaries

### Probability Theory

The *sample space* $$\Omega$$ is a set of all possible outcomes $$\omega \in \Omega$$ of some random experiment. The *event space* $$\mathcal{F} \subset 2^{\Omega}$$ represents both the amount of information available as a result of the experiment conducted and the collection of all events of possible interest to us. *Probabilities* $$\mathbf{P}(A)$$ are assigned to $$A \in \mathcal{F}$$[^3].

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

where $$\lambda$$ is called the *intensity*, or arrival rate, of the process. A Poisson process $$N$$ with intensity $$\lambda$$ satisfies:

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

where $$Y_{t^{-}}$$ is the left limit and $$\Delta N_{t} \equiv N_{t} - N_{t^{-}} \in \{0, 1\}$$ is the jump size of the standard Poisson process $$(N_{t})_{0 \leq t \leq \infty}$$[^4].

```r
# R code that simulates a sample path of a compound Poisson process
N <- 50; Tk <- cumsum(rexp(N, rate = 0.5)); Zk <- cumsum(c(0, rexp(N, rate = 0.5)))
plot(stepfun(Tk, Zk), xlim = c(0, 10), do.points = F, main = "L = 0.5", col = "blue")
Zk <- cumsum(c(0, rnorm(N, mean = 0, sd = 1)))
plot(stepfun(Tk, Zk), xlim = c(0, 10), do.points = F, main = "L = 0.5", col = "blue")
```

**Definition 9.** An adapted process $$B = (B_{t})_{0 \leq t < \infty}$$ taking values in $$\mathbb{R}^{n}$$ is called an $$n$$-*dimensional Brownian motion* if
- for $$0 \leq s < t < \infty$$, $$B_{t} - B_{s}$$ is independent of $$\mathcal{F}_{s}$$ (*increments independent of the past*);
- for $$0 < s < t$$, $$B_{t} - B_{s}$$ is a Gaussian random variable with mean zero and variance matrix $$(t - s)C$$, given a non-random matrix $$C$$[^5].

**Definition 10.** An adapted process $$X$$ with values in $$\mathbb{R}^{n}$$ is a *diffusion* if it has continuous sample paths and if it satisfies the strong Markov property.

> An intuitive notion of a diffusion is to imagine a pollen grain floating downstream in a river. The grain is subject to two forces: the current of the river (drift), and the aggregate bombardment of the grain by the surrounding water molecules (diffusion).

```r
# Simulate a discrete version of the diffusion process using random walks
# (http://www.ub.edu/viscagroup/joan/rcode.html)

random_walk <- function(m = 0.2, sig = 0.75, th = c(5, -5), y0 = 0)
{
    #m: consant input to the process
    #sig: amplitude of Gaussian noise
    #th: response threshold values for correct responses and errors
    #y0: initial state
    y <- y0 + cumsum(m + sig * rnorm(10000))
    id <- which (y > th[1] | y < th[2])[1]
    y[1 : id]
}

diffusion <- function(th = 5, m = 0.2, sig = 0.75, y = 0, N = 1000)
{
    require(dplyr)
    # Simulate N random walks for one of the responses (positive threshold)
    df <- data.frame(trial = 1 : N)
    df <- group_by(df, trial) %>% do(data.frame(y = random_walk(m = m, sig = sig, th = c(th, -th), y0 = y)))
    df <- mutate(df, step = 1 : n())
    df
}

require(ggplot2)
rws <- diffusion(N = 5000)
ggplot(rws, aes(step, y, color = trial, group = trial)) +
    geom_point(alpha = 0.02) +
    geom_line(alpha = 0.2) +
    geom_hline(yintercept = c(-5, 5)) +
    theme_classic() +
    theme(legend.position = "none") +
    scale_x_log10()
```

### Game Theory

Please refer to Maskin (2001)[^6] for more details.

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

### Black-Scholes-Merton (BSM) Model

A summary of the Black-Scholes-Merton (BSM) model is presented here for convenience.

The assumptions of the BSM model include:

* The BSM model is a model with a single interest rate;
* This interest rate is the risk-free rate $$r$$;
* Equity prices follow geometric Brownian motion;
* Short selling is permitted;
* There are no taxes and transaction costs;
* No dividends on the underlying securities;
* No arbitrage opportunities exist;
* Continuous trading of securities.

**Definition 16.** *Geometric Brownian motion*, a non-negative variant of Brownian motion, is defined by

$$S(t) = S_{0} e^{X(t)},$$

where $$X(t) = \sigma B(t) + \mu t$$ is Brownian motion with drift and $$S(0) = S_{0} > 0$$ is the initial value. For each $$t$$, $$S(t)$$ has a lognormal distribution because if we take logarithms on both sides of the above equation, we have $$X(t) = \ln \big( S(t) \big) - \ln \big( S_{0} \big)$$. Hence, $$\ln \big( S(t) \big) = X(t) + \ln \big( S_{0} \big)$$ is normal with mean $$\mu t + \ln \big( S_{0} \big)$$ and variance $$\sigma^{2} t$$. From

$$
\begin{align}
S(t + h) &= S_{0} e^{X(t + h)} \\
         &= S_{0} e^{X(t) + X(t + h) - X(t)} \\
         &= S_{0} e^{X(t)} e^{X(t + h) - X(t)} \\
         &= S(t) e^{X(t + h) - X(t)}
\end{align}
$$

we know that given $$S(t)$$, the future $$S(t + h)$$ only depends on the future increment of the Brownian motion $$X(t + h) - X(t)$$. Since Brownian motion has independent increments, this future is independent of the past. Consequently, geometric Brownian motion is a Markov process.

Now, let $$T$$ denote the fixed time of maturity of a derivative contract and $$\sigma$$ as the volatility of the underlying security (in this case an equity price). The BSM partial differential equation (PDE) is given by:

$$\frac{\partial f}{\partial t} + rS\frac{\partial f}{\partial S} + \frac{1}{2}\sigma^{2}S^{2}\frac{\partial^{2} f}{\partial^{2} S} = rf,$$

where $$f$$ is the price of a derivative which is contingent on the equity price $$S_{t}$$ and time $$t \in [0, T]$$. For a European call and put option with strike $$K$$, the BSM PDE has solution:

$$V_{t} = \alpha \Big( S_{0}N(\alpha d_{1}) - Ke^{-r(T - t)}N(\alpha d_{2}) \Big),$$

where 

$$d_{1} = \frac{\ln\big(\frac{S_{0}}{K}\big) + (r + \frac{1}{2}\sigma^{2})(T - t)}{\sigma \sqrt{T - t}},$$

and

$$d_{2} = d_{1} - \sigma \sqrt{T - t},$$

$$\alpha = 1$$ for a call option and $$\alpha = -1$$ for a put option, $$N(\cdot)$$ is the cumulative distribution function of the standard normal distribution.

## Managerial Agency Problem

**Definition 17.** *Free cash flow* is cash flow in excess of that required to fund all projects that have positive net present values when discounted at the relevant cost of capital.

Things will be more complicated if we take into account the principal-agent problem (stated in some cases as "the separation of ownership and control"). Corporate managers are the agents of shareholders. The payout of cash to shareholders reduces the resources under managers' control, and thus reducing managers' power. Managers with substantial free cash flow can increase dividends or repurchase stock and thereby pay out current cash that would otherwise be invested in low-return projects or wasted. By issuing debt in exchange for stock, corporate managers are bonding their promise to pay out future cash flows in a way such that they give shareholder recipients of the debt the right to take the firm into bankruptcy court if they do not maintain their promise to make the interest and principle payments[^7]. The agency problem can be formulated as in DeMarzo and Fishman (2007)[^8], where a risk-neutral agent operating a firm observes the firm's cash flow privately and can divert some or all of the cash flow for private benefit. In Lambrecht and Myers (2008)[^9], a self-interested coalition of managers that makes investment, financing, and payout decisions maximizes the present value of the expected cash flows taken from the firm's operations, which is the managerial rents, yet subject to the threat of collective action by the shareholders who can either close the firm or manage it privately if they decide to take over. An expected utility function of long-lived managers at each time $$t$$ may be written as:

$$M_{t} \equiv \max \mathbf{E}_{t} \Big( \sum\limits_{j = 0}^{\infty} \omega^{j} u(r_{t + jh}^{v})h \Big),$$

where $$u(\cdot)$$ is a concave utility function, $$r_{t}^{v}$$ is the monetary value of managerial rents extracted in period $$t$$ with $$v \leq 1$$, the time interval $$h$$ is the time between payouts to shareholders, $$\omega \equiv e^{-\delta h}$$ and $$\delta$$ is managers' subjective discount rate[^10].

Let's consider a firm that owns a mine with a commodity inventory $$Q$$. When the mine is open, the commodity is extracted at a constant annual rate $$q$$, and at a constant real average annual cost $$a$$. When the mine is closed, a constant real annual maintenance cost $$m$$ is incurred. At any point in time, the mine can be closed at a real cost $$k_{1}$$ and reopened at a real cost $$k_{2}$$. The real spot price of the commodity $$s$$ is determined in a competitive market and follows the exogenous process:

$$\mathrm{d}s = \mu s\,\mathrm{d}t + \sigma s \,\mathrm{d}z,$$

where $$\mathrm{d}z$$ is the increment to a standard Gauss-Wiener process; $$\sigma$$, the instantaneous standard deviation of the spot price, is assumed to be known and constant; and $$\mu$$ is the instantaneous drift in the real price. The market value of the mine is a function of the current commodity price $$s$$, of the inventory $$Q$$, of whether the mine is currently open or closed $$j = 1, 2$$, and of the optimal operating strategy $$\phi$$:

$$v \equiv v(s, Q; j, \phi).$$

Assume that corporate taxes are paid at rate $$\tau$$ on net income and full offsets are allowed. The cash flow from the mine is:

$$[q(s - a)(j - 1) - m(2 - j)](1 - \tau).$$

Applying Ito's lemma, the instantaneous change in the value of the mine is given by:

$$\mathrm{d}v = \frac{\partial v}{\partial s}\,\mathrm{d}s + \frac{\partial v}{\partial Q}\,\mathrm{d}Q + \frac{1}{2} \frac{\partial^{2} v}{\partial^{2} s}\,(\mathrm{d}s)^{2}.$$

We can solve for the first-best value of the mine $$v^{FB}$$ and the first-best operating policy $$\phi^{FB}$$ under the BSM setting[^11].

**Definition 18.** The difference in maximal firm values between the results from ex ante and ex post risk choices (that is, before and after debt is in place) serves as a measure of *agency costs*, because it reflects the loss in value that follows from the risk strategy maximizing equity value rather than firm value[^12].

Without any agency costs of debt, the value of the levered firm would be the first-best value of the firm plus the interest tax shield of debt. Each added unit of debt increases the value of the firm by the value of its associated interest tax shield. With agency costs, as the size of debt increases, the total agency costs may far outweigh the total tax shields, making the value of the levered firm less than the first-best.

How about a corporate manager with *empire-building* motive?

|---
| $$t = 0$$ | $$t = 1$$ | $$t = 2$$
|:-:|:-:
| Old assets-in-place | Old assets yield return $$y_{1}$$; <br /> Decision whether to make new investment of size $$i$$; <br /> Decision whether to liquidate and realize $$L$$ | Old assets yield return $$y_{2}$$; <br /> New investment, if taken, yields return $$r$$

Let us assume that:
- investors are risk-neutral,
- the risk-free interest rate is zero,
- the financial structure of a typical firm is chosen at $$t = 0$$ so as to maximize the *aggregate* return of the shareholders,
- the only way to prevent the manager of the firm from investing in a bad project is to make necessary funds unavailable,
- the probability distributions of liquidation, investment costs, returns on assets and investments are all common knowledge, and values are revealed at $$t = 1$$,
- $$y_{1} < i$$ and $$y_{2} \geq L$$ with probability $$1$$.

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
4. Finally, if $$i$$ and $$y_{1}$$ are deterministic, and $$r \equiv g(y_{2})$$ where $$g(\cdot)$$ is a strictly increasing function, then the first-best outcome can be achieved by letting $$d_{2} = y_{1} + g^{-1}(i)$$[^13].

## References

[^1]: Stephen A. Ross, Comment on the Modigliani-Miller Propositions, *Journal of Economic Perspectives* **2**(4), 127-133 (1988).

[^2]: Stewart C. Myers, Determinants of Corporate Borrowing, *Journal of Financial Economics* **5**, 147-175 (1977).

[^3]: Amir Dembo, *Lecture Notes of "Probability Theory: STAT310/MATH230"* at Stanford University, [https://web.stanford.edu/~montanar/TEACHING/Stat310A/lnotes.pdf](https://web.stanford.edu/~montanar/TEACHING/Stat310A/lnotes.pdf).

[^4]: Nicolas Privault, *Lecture Notes on Stochastic Finance* at Nanyang Technological University, [https://personal.ntu.edu.sg/nprivault/indext.html](https://personal.ntu.edu.sg/nprivault/indext.html).

[^5]: Philip E. Protter, *Stochastic Integration and Differential Equations*, Second Edition, Springer-Verlag Berlin Heidelberg, 2004.

[^6]: Eric Maskin, Markov Perfect Equilibrium, *Journal of Economic Theory* **100**, 191-219 (2001).

[^7]: Michael C. Jenson, Agency Costs of Free Cash Flow, *The American Economic Review* **76**(2), 323-329 (1986).

[^8]: Peter M. DeMarzo and Michael J. Fishman, Agency and Optimal Investment Dynamics, *The Review of Financial Studies* **20**(1), 151-188 (2007).

[^9]: Bart M. Lambrecht and Stewart C. Myers, Debt and Managerial Rents in a Real-Options Model of the Firm, *Journal of Financial Economics* **89**, 209-231 (2008).

[^10]: Bart M. Lambrecht and Stewart C. Myers, The Dynamics of Investment, Payout, and Debt, *The Review of Financial Studies* **30**(11), 3759-3800 (2017).

[^11]: Antonio S. Mello and John E. Parsons, Measuring the Agency Costs of Debt, *The Journal of Finance* **47**(5), 1887-1904 (1992).

[^12]: Hayne E. Leland, Agency Costs, Risk Management, and Capital Structure, *The Journal of Finance* **53**(4), 1213-1243 (1998).

[^13]: Oliver Hart and John Moore, Debt and Seniority: An Analysis of the Role of Hard Claims in Constraining Management, *The American Economic Review* **85**(3), 567-585 (1995).