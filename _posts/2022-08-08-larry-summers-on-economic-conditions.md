---
layout:           post
title:            "Larry Summers on 2022 U.S. Economic Conditions"
category:         "Macroeconomics and Finance"
tags:             central-bank macroeconomics inflation unemployment
permalink:        /blog/larry-summers-on-economic-conditions
last_modified_at: "2022-12-03"
---

<p class="message"><em>There are two kinds of forecasters: those who don't know, and those who don't know they don't know.</em></p>

Summers' recent research papers may be quite informative if we look forward to some technical judgments towards the U.S. macro-economic conditions as well as the Fed's goals during an abnormal time.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Declines in Worker (Bargaining) Power over Recent Decades

In a [2018 working paper with Stansbury](https://www.piie.com/system/files/documents/wp18-5.pdf)[^1], Summers investigated the stark divergence between labor productivity, measured as net total economy output per hour, and the typical or average worker's compensation inclusive of non-wage benefits, measured as median compensation or mean total economy compensation. The compensation measures are constructed using wage series from the Current Population Survey (CPS) Outgoing Rotation Groups (ORG) as well as Bureau of Labor Statistics (BLS) Current Employment Statistics (CES), adjusted by the average real compensation/wage ratio to include non-wage compensation. Major findings are:

- Over the period of 1975-2015, a 1 percentage point increase in productivity growth was associated with a 0.73 percentage point increase in the growth rate of median compensation,
- Over the period of 1950-2015 and 1975-2015, a 1 percentage point increase in productivity growth was associated with a 0.74 and 0.77 percentage point increase in the growth rate of average compensation respectively,
- Over the period since 2000, the labor share has declined.

<center><b>Total labor rents = union rents + industry rents + firm size rents</b></center>

- "Labor rents": measured as a share of net value added in the nonfinancial corporate business sector,
- "Union rents": rents arising from union wage premia for unionized workers,
- "Industry rents": industry wage premia,
- "Firm size rents": rents arising from large-firm wage premia.

In the [2020 Brookings working paper](https://www.brookings.edu/wp-content/uploads/2020/12/StansburySummers-Final-web.pdf)[^2], Stansbury and Summers argued that changes in institutions, economic conditions, and corporate management combined contribute to the decline in worker power or rent-sharing with labor in the US economy over recent decades. Labor rents declined from around 12 percent in the early 1980s to around 6 percent in the 2010s. According to data from the CPS Annual Social and Economic Supplement, observably equivalent private sector workers at large firms (firms with 500 or more employees) enjoy attenuating wage premia relative to those at small firms during 1990-2019. However, the share of workers in large firms has actually grown over the period from 32 percent to 42 percent. More importantly, since the 1980s, the declining labor share of income has been matched with a rise in the capital share of income.

## Tightened Labor Markets

> [Blanchflower and Levin (2015)](https://www.nber.org/system/files/working_papers/w21094/w21094.pdf)[^3] - It is readily apparent that the conventional unemployment rate has *not* served as an accurate synopsis of the evolution of labor market slack. For example, the declining unemployment rate over the course of 2010 and most of 2011 was not induced by a pickup in job growth but instead reflected the extent to which many Americans gave up searching for work and departed from the labor force.

- Labor market slack measures on the supply-side: **(1) unemployment rate** (labeled "U3" by BLS) and **(2) prime-age (25-54) nonemployment rate** (the share of the civilian noninstitutional population aged 25-54 that is not working measured by one minus the prime-age employment-to-population ratio),
- Labor market slack measures on the demand-side: **(3) job vacancy rate** (the level of job vacancies divided by the size of the civilian labor force) and **(4) quits rate** (the level of quits divided by the size of the civilian labor force).

According to OECD data, there are 11,254,000 unfilled job vacancies in May 2022 for the United States, indicating a job vacancy rate of 6.8 percent (see [FRED database](https://fred.stlouisfed.org/series/LMJVTTUVUSM647S)), similar to the near-record high level in December 2021 reported by the BLS. For June 2022, the BLS Job Openings and Labor Turnover Survey reported a job openings level of 10,698,000, down 605,000 from a month earlier, the lowest in nine months and below market expectations of 11 million. Since the unemployed population is 5,670,000, the number of job openings per unemployed is 1.88, a little drop from the all-time high of 1.99 in March. Meanwhile, some 4.2 million Americans quit their jobs, little changed from the prior month, with the quits rate unchanged at 2.8 percent. In July, there are 3,041,000 unemployed persons aged 25-54 and the prime-age nonemployment rate is 20 percent.

> [BLS Employment Situation Summary](https://www.bls.gov/news.release/empsit.nr0.htm) - In July, the unemplyment rate edged down to 3.5 percent, and the number of unemployed persons edged down to 5.7 million. These measures have returned to their levels in February 2020, prior to the coronavirus (COVID-19) pandemic . . . The labor force particpation rate, at 62.1 percent, and the employment-population ratio, at 60.0 percent, were little changed over the month. Both measures remain below their February 2020 values (63.4 percent and 61.2 percent, respectively).

[Domash and Summers (2022a)](https://www.nber.org/system/files/working_papers/w29739/w29739.pdf)[^4] compared the predictive power of the aforementioned four different slack indicators on wage inflation. The baseline regression takes the following form:

$$\log \textsf{wage}_{t} - \log \textsf{wage}_{t-1} = \alpha + \beta\,\textsf{slack}_{t} + \gamma\,\textsf{lagged inflation}_{t} + \epsilon_{t},$$

where the lagged inflation variable is a 3-year weighted average of CPI that assigns a weight of three to inflation in period $t-1$, a weight of two to inflation in period $t-2$, and a weight of one to inflation in period $t-3$, to control for expected inflation. The results show that **the unemployment rate is much more significant than the labor force participation rate in explaining wage inflation** using the CPS-ORG composition-adjusted mean and median wages (given that the prime-age employment-to-population ratio is mathematically equivalent to the labor force participation rate multiplied by one minus unemployment rate); the vacancy rate and the quits rate are broadly comparable to the unemployment rate in their ability to predict wage inflation. To assess the tightness of the current labor markets, another linear-log model is estimated with data from January 2001 to December 2019, avoiding any changes induced by the Covid-19 pandemic:

$$
\begin{align}
\textsf{Predicted unemployment rate}_{t} = \alpha \\
+ \beta \sum\limits_{0}^{12} \log \textsf{vacancy rate}_{t - i} \\
+ \delta \sum\limits_{0}^{12} \log \textsf{quits rate}_{t - i} \\
+ \gamma\,\textsf{time trend}_{t} \\
+ \theta\,\textsf{structural break}_{t} \\
+ \epsilon_{t},
\end{align}
$$

where the selection for the structural break is July 2009, learning from [Michaillat and Saez (2019)](https://eml.berkeley.edu/~saez/michaillat-saezRESTUD19.pdf)[^5], who found a break in the relationship between the job vacancy rate and the unemployment rate at the time. The underlying reasoning is to examine what unemployment rate is consistent with the current levels of job vacancies and quits. 

> For December 2021, this model specification predicts a firm-side equivalent unemployment rate (at the national level) between 1.2 and 1.7 percent, compared to the actual unemployment rate of 3.9 percent, signaling a very tight labor market from the perspective of employers. Using a linear model instead of a linear-log model would lower the estimated firm-side unemployment even further.

How can we explain the phenomenon that unemployment is low while vacancies remain high? Denote vacancies as $V$ and unemployed populaton $U$, then we have a simple index of activity $x$:

$$x \equiv V / U.$$

Dividing vacancies and unemployed population by the labor force $N$, we can write:

$$v / u = x,$$

where $v$ is the vacancy rate and $u$ is the unemployment rate. Next, think of gross hires $H$ depending on the number of vacancies and the numbers of unemployed:

$$H = \alpha m(U, V),$$

where the RHS is a matching function: the parameter $\alpha$ reflects the efficiency of matching and $m(\cdot)$ is an increasing function of $U$ and $V$. The equation above can also be written as

$$H = \alpha U^{r} V^{1-r}$$

or

$$H/N = \alpha (U/N)^{r} (V/N)^{1-r}$$

if the function $m(\cdot)$ can be assumed to have constant returns to scale. Using this framework developed by [Blanchard, Domash and Summers (2022)](https://www.piie.com/sites/default/files/documents/pb22-7.pdf)[^6], we can decompose movements in the vacancy and unemployment rates between movements due to aggregate activity, matching efficiency, and reallocation. The authors' analysis indicated increased natural unemployment rate, worse matching, and higher reallocation for the U.S. labor markets.

## Run-up in Inflation

> [A Twitter thread posted on September 21](https://twitter.com/LHSummers/status/1572696918815277059) - The Fed is projecting $4.6$ federal funds rate at the end of 2023. If this happens, I expect that rate will have been higher during year. So, the Fed's implied expectation for the terminal rate must be very much in the range of $5$ percent. Encouraging but not sure fully grasped by the market. Gas prices are lower than they were before Putin invaded. In addition to a paragraph about the Ukraine war, I would have liked to see some recognition, even subtle, of the need to counteract the effects of past excessive fiscal and monetary stimulus. Would help credibility. This is especially the case when so many economists are saying things that echo the things said in support of the errors of the 1960s and 1970s. "Transitory factors, it's all supply shocks, expectations are securely anchored, we absolutely need a high-pressure economy." The Fed's dot-plot economics are more plausible than they have been but still implausibly optimistic. I cannot see how they expect as a central case, core inflation to fall to $2$ percent with unemployment peaking at $4.5$ (below reasonable NAIRU estimates), two vacancies for every job and wage growth running near $6$. Their forecast would for me be the optimistic tail of the distribution. Happy to bet anyone that we see six months of unemployment above $5$ before we see six months of inflation below $2.5$. Also do not see a basis for the argument that policy is already at beginning of restrictive given government debt expansion and ongoing core inflation predicted to run above $3$ for next 18 months. Chairman Powell is very thoughtful at press conferences but I wonder whether the Fed's credibility is well served by frequent hour long dialogues on hypotheticals and the unforecastable, with the backdrop of gyrating markets. Between press conferences and dot plots and minutes, the Fed should consider the idea of TMI ("too much information").

> [Larry Summers on Powell's August 26 Jackson Hole Speech](https://www.youtube.com/watch?v=w1z6Dk-segQ) - He prioritized inflation, making clear that he recognized that that prioritization would have short-term adverse consequences and that wouldn't be easy. But, by bringing down inflation ultimately there're going to be more jobs with higher real wages for more people. And that's, after all, what economic policy is all about.

Understanding inflation patterns is important to professional economists and those tracking it in real time. By looking at the past five economic expansions (1975Q2-1979Q2, 1982Q4-1989Q1, 1992Q3-2000Q4, 2003Q2-2006Q4, 2009Q4-2019Q3), [Heise et al. (2022)](https://onlinelibrary.wiley.com/doi/full/10.1111/jmcb.12896)[^7] discovered cumulative consumer price inflation lowering over time, from $25\%$ in the first 20 quarters of the recovery following the 1981-92 recession to less than $10\%$ in the last expansion.

Federal Reserve Bank of Atlanta's [Wage Growth Tracker](https://www.atlantafed.org/chcs/wage-growth-tracker) shows the historically unprecedented acceleration of wage growth in recent months with median wage inflaton (using the weighted 3-month moving average of median wages) in July 2022 reaching 7.0 percent.

"The idea that inflation can fall dramatically without a corresponding rise in labor market slack runs counter to standard economic theory, and is inconsistent with the historical evidence," says [Domash and Summers (2022b)](https://www.nber.org/system/files/working_papers/w29910/w29910.pdf)[^8]. The results from Domash and Summers (2022a) suggest that a given level of unemployment today is likely associated with a significantly more inflationary rate of wage growth than in the past.

[The Bureau of Labor Statistics (BLS) Consumer Price Index for All Urban Consumers (CPI-U)](https://www.bls.gov/news.release/cpi.t01.htm) reported a **8.5 percent** year-over-year growth for July 2022 (almost unchanged from June), well above any other period since 1981. Among all items, gasoline expenditure rose 44.0 percent from a year before. [The Personal Consumption Expenditures (PCE) Price Index](https://www.bea.gov/data/personal-consumption-expenditures-price-index) released monthly by the Bureau of Economic Analysis (BEA) gave 6.8 percent to the year-over-year growth rate for June 2022.

Consumer price indices are constructed as an aggregate of inflation rates by sector, such that

$$\pi_{t} = \sum\limits_{t} \omega_{i} \pi_{i, t},$$

where $\omega_{i}$ is the expenditure weight of sector $i$ in the household consumption basket. The most simple and well-known example of a division of inflation is core PCE inflation, where overall inflation is divided into a core component and non-core component depending on whether the sector $i$ belongs to the food or energy sectors:

$$\pi_{t} = \underbrace{\sum\limits_{i}(1 - \mathbb{I}_{i \in f, e}) \omega_{i} \pi_{i, t}}_\textsf{core} + \underbrace{\sum\limits_{i}\mathbb{I}_{i \in f, e} \omega_{i} \pi_{i, t}}_\textsf{non-core},$$

where $\mathbb{I}_{i \in f, e}$ is the indicator function:

$$
\mathbb{I}_{i \in f, e} =
\begin{cases}
     1, & \textsf{if equals energy (e) or food (f)},\\
     0, & \textsf{otherwise}.
\end{cases}
$$

|---
| **Component** / **Series** | **CPI** | **Core CPI** | **PCE** | **Core PCE**
|:-|:-|:-|:-|:-
| Owners' Equivalent Rent | 23.6 | 29.9 | 11.2 | 12.8
| Rent | 7.4 | 9.6 | 3.6 | 4.1

*Table 1: Differences in weight shares in housing PCE vs. CPI, percent, from [BCS (2022a)](https://www.nber.org/system/files/working_papers/w29795/w29795.pdf)*[^9]
{:.table-caption}

Numerous articles have thoroughly discussed the methodologies adopted by the BLS and BEA. As documented by [BCS (2022b)](https://www.nber.org/system/files/working_papers/w30116/w30116.pdf)[^10], the BLS exchanged homeownership costs for owners' equivalent rent in 1983, stripping away the investment aspect of housing to isolate owner-occupiers' consumption of residential services, which had large effects on both headline and core measures of inflation.

> In December 1982, "homeownership" received 26.1 percent of the weight of the overall CPI index. For the core CPI, it was a full 36.1 percent...In January 1983, owners' equivalent rent of residence accounted for only 13.5 percent of the weight for the overall CPI in its first month of existence.

BCS argued that the peak of the Volcker-era inflation (March 1980), currently understood to have been at 14.8 percent, is only 11.4 percent when adjusted for the switch from homeownership costs to owners' equivalent rent. The growth in core CPI at its peak in June 1980 falls from 13.6 percent to 9.1 percent when measured using the new method. The Fed's preferred measure of inflation since 2000 is the PCE price index, which has measured rent inflation consistently.

> In order to return to 2 percent core CPI today, we need nearly the same 5 percentage points of disinflation that Volcker achieved.

## References

[^1]: Anna Stansbury and Lawrence H. Summers, Productivity and Pay: Is the Link Broken? *PIIE Working Paper* 18-5, June 2018.

[^2]: Anna Stansbury and Lawrence H. Summers, The Declining Worker Power Hypothesis: An Explanation for the Recent Evolution of the American Economy, *Brookings Papers on Economic Activity*, Spring 2020.

[^3]: David G. Blanchflower and Andrew T. Levin, Labor Market Slack and Monetary Policy, *NBER Working Paper* 21094, April 2015.

[^4]: Alex Domash and Lawrence H. Summers, How Tight Are U.S. Labor Markets? *NBER Working Paper* 29739, February 2022.

[^5]: Pascal Michaillat and Emmanuel Saez, Optimal Public Expenditure with Inefficient Unemployment, *Review of Economics Studies* (2019) **86**, 1301-1331.

[^6]: Olivier Blanchard, Alex Domash, and Lawrence Summers, Bad News for the Fed from the Beveridge Space, *PIIE Working Paper* 22-7, July 2022.

[^7]: Sebastian Heise, Fatih Karahan, and Aysegul Sahin, The Missing Inflation Puzzle: The Role of the Wage-Price Pass-Through, *Journal of Money, Credit and Banking*, Volume 54, Issue S1, February 2022.

[^8]: Alex Domash and Lawrence H. Summers, A Labor Market View on the Risks of a U.S. Hard Landing, *NBER Working Paper* 29910, April 2022.

[^9]: Marijn A. Bolhuis, Judd N. L. Cramer, and Lawrence H. Summers, The Coming Rise in Residential Inflation, *NBER Working Paper* 29795, February 2022.

[^10]: Marijn A. Bolhuis, Judd N. L. Cramer, and Lawrence H. Summers, Comparing Past and Present Inflation, *NBER Working Paper* 30116, June 2022.