---
title: "Adjusted t-stat for overlapping samples"
date: 2023-12-13T19:56:41Z
draft: false
toc: false
images:
math: true
tags:
  - t-stat
  - sampling
---

When computing a regression coefficient, we aim to reject the null hypothesis that there is no relationship between the dependent and independent variable ($ \beta_0 = 0 $). Similarly, when evaluating a trading strategies *returns*, we aim to reject the null hypothesis that they are unprofitable random noise around 0. 

If we assume the distribution underlying the inputs are normal ($X_{0}$ for regression and $r_{t}$ for returns) we can use the *t-statistic* to decide to reject these hypothesis' or not. The t-stat measures the number of standard errors (SE) the mean value is from a reference point ($ \beta_0 $). As discussed in our examples above our reference point is 0, so we can drop this term going forward. The SE is a estimator which uses the sample standard deviation ($\hat{\sigma}$) to estimate the population standard deviation ($\sigma$). Note that as the number of samples (N) increases the SE gets smaller, reflecting our increased confidence in our approximation of $\hat{\sigma}$ due to more datapoints - we'll revisit this point later.

$$ t_{\hat{\beta}} = \frac{\hat{\beta} - \beta_0}{SE(\hat{\beta})} \ \ \ \ \ \ \ SE = \frac{\hat{\sigma}}{\sqrt{N}} $$

A well known fact of the normal distribution is that around 95% of the values are within $2\sigma$ of the mean. If our t-stat is greater than 2 (read: the mean value is more than $2\sigma$ from 0 - our null hypothesis) then there is a 5% probability this is by random chance. This probability threshold (*p-value*) of 5% is a common cutoff to reject the null hypothesis in literature. *Note:* this cutoff is for a single test, if we make multiple attempts the p-value should be reduced accordingly.

Reading *Long Horizon Predictability: A Cautionary Tale* [^1], It warns that when sample overlap is present (common in time series modelling) the standard error can be underestimated. Due to serial correlation in the data we have datapoints which hold less value than expected in an Independent and Identically distributed (IID) world. 

For example if we forecast 10 ticks into the future, a target datapoint at t and t+1 will share 9/10 ticks of lookahead, this shared dependency path is likely to lead to correlated or identical target values. Similarly, lookback feature measurements based on rolling windows are not likely to change much tick to tick. To trivially highlight this issue, we can duplicate all of our sample data points and watch the SE drop even though we are adding no new information.

To correct for this the paper proposes measuring and adjusting the non overlapping variance ($\sigma^2$) before we incorporate it into the SE and t-stat, the formula is split out below. *J* is the number of lookahead ticks, *ol* signifies a series with overlaps, *nol* is one with no overlaps.

$$ 
var(\hat{\beta}_J^{ol}) = var(\hat{\beta}_J^{nol})[\frac{1}{J} + \theta(J, \rho_J)] 
$$ 

$$ \theta(J, \rho_J) = 2 \sum_{j=1}^{J-1} (\frac{J-j}{J^2}) \rho_j $$

where $\rho_j$ is the $j$th order autocorrelation of $X_{t}$.

The first part applies a multiplier to the variance of the non-overlapping variance. The multiplier is the summation of $\frac{1}{J}$ and a function $\theta$ of the autocorrelation of the series.

The multiplier has offsetting adjustments:
* $\frac{1}{J}$ adjusts the multiplier (variance) of *nol* downwards as *ol* has J times more observations.
* $\theta$ adjusts the multiplier (variance) upwards as a function of the autocorrelation.  If autocorrelation is high then we are not adding additional information over the *nol* version (similar to our trivial example where we duplicate all values).

A Python implementation is shown below:

{{< highlight Python >}}
from statsmodels.graphics.tsaplots import acf

def adj_t_stat(signal, lookahead):
    J = lookahead
    
    lags = acf(signal, nlags=J)
    theta = 2 * sum([((J-j)/(J**2))*lags[j] for j in range(1, J)])
    adj = (1/J) + theta
    
    signal_nol = signal[::J]
    var_nol = signal_nol.var()
    
    var_adj = var_nol*adj
    se = np.sqrt(var_adj) / np.sqrt(len(signal_nol))
    
    t = signal.mean() / se
    
    return t
{{< /highlight >}}

[^1]: Jacob Boudoukh, Ronen Israel & Matthew Richardson (2019) Long-Horizon Predictability: A Cautionary Tale, Financial Analysts Journal, 75:1, 17-30, DOI: 10.1080/0015198X.2018.1547056
