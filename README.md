# HJM model with stochastic volatility (CIR)

**This notebook implements a single-factor HJM forward-rate model with CEV scaling and a single mean-reverting stochastic volatility factor, calibrated to caplet and swaption markets.**  

---

## Model

**Forward-rate SDE (risk-neutral measure $(\mathbb{Q})$):**
$$
{\;df(t,T) \;=\; \mu(t,T)\,dt \;+\; \sigma(t,T)\,dW_t\;}
$$

**_where:_**

Instantaneous forward volatility (CEV × amplitude):
$$
{\sigma(t,T) = v_t\,\phi(t,T)\,\bigl[f(t,T)+\delta\bigr]^{\beta}}
$$
where: 
- $f(t,T)$ = instantaneous forward rate maturing at $T$ observed at time $t$.  
- $v_t$ = scalar stochastic volatility (amplitude).  
- $\phi(t,T)$ = deterministic shape (term-structure), e.g. $\phi(t,T) = a^{-b(T-t)}$.  
- $\beta$ = CEV exponent.  
- $\delta \geq 0$ = small shift (default $1\times 10^{-4}$) to handle near-zero/negative rates.  

where:

**Stochastic volatility SDE (CIR / Heston-like):**
$$
{\;dv_t \;=\; \kappa(\theta - v_t)\,dt \;+\; \xi\sqrt{v_t}\,dZ_t\;}
$$
where:
- $\kappa$ = speed of mean reversion, $\theta$ = long-run level, $\xi$ = vol-of-vol.  
- Use Andersen QE exact/approx scheme where possible; fallback to full-truncation Euler for robustness.

**Correlation between the driving noises:**
The two Brownian motions $W_t$ (for forwards) and $Z_t$ (for volatility) satisfy
$$
{\;d\langle W,Z\rangle_t \;=\; \rho\,dt\;}
$$
with $\rho \in [-1,1]$ (typical bounds used in calibration code: $[-0.99,0.99]$).

**HJM no-arbitrage drift:**
$$
{\;\mu(t,T) \;=\; \sigma(t,T)\,\int_{t}^{T}\sigma(t,s)\,ds\;.}
$$
- In practice the integral $\int_t^T \sigma(t,s)\,ds$ is approximated numerically on the model $T$-grid (trapezoid / Simpson).

---

Deterministic shape (example parameterisation):
A simple, calibratable choice used in the code:
$$
\phi(t,T) = a \, e^{-b (T - t)} 
$$
with $a > 0$, $b \ge 0$ exposed as calibration parameters.

---

## Parameters exposed (defaults / bounds)
- $\beta \in [-0.5, 1.5]$ — default: $0.0$  
- $\delta \ge 0$ — default: $1\times 10^{-4}$  
- $a \in (0, 1.0]$ — default: $0.02$  
- $b \in [0, 2.0]$ — default: $0.5$  
- $\kappa \in [10^{-3}, 10]$ — default: $1.0$  
- $\theta \in [10^{-4}, 2.0]$ — default: $0.04$  
- $\xi \in [10^{-4}, 2.0]$ — default: $0.3$  
- $\rho \in [-0.99, 0.99]$ — default: $-0.5$  
- $v_0 \in [10^{-6}, 2.0]$ — default: $0.04$

---

## Process

**1. Configuration & environment**  
   - CONFIG dictionary: random number generator (RNG) seed, MC path counts, grid resolution, weights for swaption vs caplet, file paths.

**2. Synthetic market data generation** (`/data/*.csv`)  
   - Create deterministic OIS/RFR curve CSV (`curve.csv`) with times, zero rates and discount factors.  
   - Create synthetic caplet vol matrix (`caplet_quotes.csv`) and synthetic swaption surface (`swaption_quotes.csv`).  
   - Files are deterministic (seeded) so runs are reproducible.

**3. Bootstrapping / interpolation**  
   - Read `curve.csv`, build interpolant \(P(0,t)\) (log-linear interpolation of discount factors).  
   - Compute initial instantaneous forward curve $f(0,T) = -\partial_T \ln P(0,T)$ on a fixed $T$-grid.

**4. Model initialisation**  
   - Instantiate `HJMModel` with: T-grid, initial forwards $f(0,T)$, parameter struct ($a, b, \beta, \delta, \kappa, \theta, \xi, \rho, v_0$)
, and $\phi(t,T)$ function.

**5. Monte-Carlo simulation engine**  
   - Simulate forward-curve paths and the volatility factor $(v_t)$ using correlated normals (common random numbers, optional antithetic variates).  
   - Use full-truncation Euler for $v_t$.  
   - Compute HJM drift $\mu(t,T)$ via numerical quadrature on the $T$-grid at each step.  

**6. Instrument pricers (MC-based)**  
   - **Caplet pricer:** uses simulated $f(\text{expiry}, \text{expiry}+\tau)$ to compute payoff $\mathrm{DF}\,\tau\,\max(F-K,0)$.
   - **Swaption pricer:** approximate via forward swap rate and annuity (MC used to evolve state to expiry where needed).  
   - Provide Black implied-vol inversion utilities to compare model prices to quoted vols.

**7. Calibration — two stages**  
   - **Stage A (ATM term structure):** calibrate $(a,b,\beta)$ to match ATM caplet & ATM swaption term structures (fast MC or approximations).  
  Swaption vs caplet weights are configurable; default places higher weight on swaption surface for mid/long tenors.  

   - **Stage B (smiles):** fix Stage A parameters and calibrate $(\kappa,\theta,\xi,\rho,v_0)$ to capture smile/skew across strikes/deltas using Monte Carlo pricing.  
  Use global $\rightarrow$ local optimisation strategy in practice (the notebook uses local methods for speed).  


**8. Diagnostics & outputs**  
   - Save: `outputs/calibration_results.json`, `outputs/calibrated_parameters.json`, `outputs/model_vs_market_caplets.csv`, `outputs/model_vs_market_swaptions.csv`.  
   - Plots: ATM caplet fit, ATM swaption surface fit, selected smile comparisons (PNG files under `outputs/plots/`).

-------------
  ABOUT ME
-------------
- I began my python journey 15 weeks ago on 9 June 2025. Today is 21 Sept 2025. 
- I am seeking to learn python skills to secure the most rewarding finance career opportunities possible. 
- 20 years total experience in finance with Morgan Stanley, Credit Agricole CIB, Barclays Capital.
- Former Head of Cross Asset Derivatives Structuring - Rates, FX, Credit, Equities, Commodities.
- Top year desk PnL $26.5m (I was desk head). Top year individual PnL $15m. 
