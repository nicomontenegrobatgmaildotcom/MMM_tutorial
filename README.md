# Marketing Mix Modeling from scratch: when does it find the truth?

A step-by-step Bayesian Marketing Mix Model (MMM) built in PyMC, tested on simulated data where the true channel effects are known.

There is no replacement to doing this for yourself I you want to learn MMMs. You can read papers, books on the topic but you will never get the nuances of MMMs by not simulating this yourself. 

## Approach

Each step adds a single problem to the data, then fits:
1. a model that ignores the problem, to see how it fails,
2. a model that handles it, to see whether it recovers the true values.

| Step | Added to the data | Status |
|---|---|---|
| 1 | Two independent channels, linear effects | Done |
| 2 | Carryover (adstock) on TV | Done |
| 3 | Saturation (diminishing returns) on TV | Done |
| 4 | Seasonality | Planned |
| 5 | Correlated channels (shared budget) | Planned |
| 6 | Confounder (drives both spend and sales) | Planned |
| 7 | Mediator (TV drives search) | Planned |

## The model

    sales = intercept + Σ beta_c × saturation(adstock(spend_c)) + controls + noise

- **Adstock:** geometric decay with normalized weights over an L-week window, decay `alpha ~ Beta(2, 2)`.
- **Saturation:** logistic, `(1 − e^(−λx)) / (1 + e^(−λx))`, applied to scaled spend.
- **Scaling:** spend divided by its max, sales by their max. Results converted back to real units.
- **Checks:** prior predictive checks, r_hat / ESS / divergences, prior vs posterior comparison, and comparison with the true values.

## Results so far

### Step 1: linear, independent channels
104 weeks, true values: intercept 1000, TV 3, social 5, noise 50.

| Parameter | Estimate (94% HDI) | True |
|---|---|---|
| Intercept | 991 (965–1019) | 1000 |
| TV (sales per €) | 2.99 (2.64–3.35) | 3 |
| Social (sales per €) | 5.11 (4.37–5.84) | 5 |
| Sigma | 58.9 (51.7–66.9) | 57.6* |

\*Set to 50 in the code, but the noise actually drawn in this sample has a standard deviation of 57.6, and the model estimates that.

**Takeaway:** with independent channels and no complications, the model recovers everything. This is the benchmark for the next steps.

### Step 2: carryover (adstock)
TV now has carryover (alpha = 0.6) and runs in bursts (on in about half the weeks).

| | Without adstock | With adstock | True |
|---|---|---|---|
| TV | 1.32 | 3.11 | 3 |
| Alpha (TV) | – | 0.60 | 0.6 |
| Intercept | 1065 | 999 | 1000 |
| Sigma | 61.5 | 47.2 | ~50 |

**Takeaway:** ignoring carryover underestimates TV by more than half: the model only sees the same-week effect. The leftover carryover is absorbed by the intercept and the noise.

**Side experiment: TV always on (no bursts), plain OLS.** TV = 1.20 and intercept = 1136. This matches theory: the same-week share of TV's effect is 1/Σweights ≈ 41% (≈ 1.22), and when TV is always on, the carryover becomes an almost constant layer of about 130 sales that looks like base sales. The OLS output gave no clear warning: high t-values and a reasonable R², but TV was 60% too low.

### Step 3: saturation
TV gets adstock and saturation. With saturation, `beta` becomes the ceiling (the most sales a channel can ever add), not sales per €.

| True ceiling | alpha | lam | Ceiling estimate | Comment |
|
