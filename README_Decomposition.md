# Major Difficult Step (2): Counterfactual Decomposition (GE + Flow vs Valuation)

**Goal (for the code session):** explain (i) code structure, (ii) analysis steps, (iii) the major difficult step (GE fixed-point / pseudo-Newton solver).  
We focus on **structure + algorithms**, not R syntax.

**This note covers three tasks:**
1. Construct counterfactual “worlds” (`compute_cf()`).
2. Re-solve general equilibrium inside each world (`RunCF()`).
3. Decompose NFA changes into **flow vs valuation** and aggregate by blocks (`getDecomp()`, `analyze_cf()`).

# 1. What the decomposition is doing

The core object of the paper is the U.S. Net Foreign Assets (NFA) position. Changes in the NFA originate from two main sources:

### 1.1 Accounting identity: NFA change = Flows + Valuation

Any international balance sheet has an intuitive decomposition:

* **Flows:** The "quantity changes" resulting from net purchases or net issuances during the current period.
* **Valuation:** Even without trading a single share or bond, existing holdings will be revalued due to changes in asset prices, exchange rates, and returns.


#### Paper Definition: Flow Effect vs. Valuation Effect (Log Decomposition)

Beyond the intuition that "NFA change = flows + valuation," the framework formally defines the **Flow Effect** and **Valuation Effect** in log form using a **Constant-Price NFA** benchmark.

##### 1. Conceptual Framework
Let the U.S. Net Foreign Assets at step $j$ of the decomposition at time $t$ be represented by:

$$\widehat{NFA}^{j}_{US,t}$$

We define the constant-price NFA, which holds asset prices and exchange rates fixed at the levels from step $j-1$, as:

$$\widehat{NFA}^{ConstPrice,j}_{US,t}$$

---

##### 2. Log-Decomposition Formulas

The decomposition separates the total change into two distinct components:

##### **A. Flow Effect ($\Delta^{Flow}_{j,t}$)**
This captures the change driven by **quantities and flows** while holding prices and FX constant:

$$\Delta^{Flow}_{j,t} = \log\left(1 + \frac{\widehat{NFA}^{ConstPrice,j}_{US,t} - \widehat{NFA}^{j-1}_{US,t}}{\widehat{NFA}^{j-1}_{US,t}}\right)$$

##### **B. Valuation Effect ($\Delta^{Val}_{j,t}$)**
This captures the change driven by **revaluation** via asset price and FX updates:

$$\Delta^{Val}_{j,t} = \log\left(1 + \frac{\widehat{NFA}^{j}_{US,t} - \widehat{NFA}^{ConstPrice,j}_{US,t}}{\widehat{NFA}^{j-1}_{US,t}}\right)$$

---

##### 3. Summary of Interpretation

| Component | Driver | Variable Fixed At |
| :--- | :--- | :--- |
| **Flow Effect** | Changes in Quantity / Holdings | Prices & FX (Step $j-1$) |
| **Valuation Effect** | Changes in Prices & FX | Quantities (Step $j$) |

##### 4. Economic Significance: Why This Structural Decomposition Matters

While traditional Balance of Payments (BOP) accounting also separates Current Account flows from valuation changes, the JRZ (2024) **model-implied decomposition** offers a critical advantage:

**Counterfactual Isolation:** Traditional accounting can only observe *ex-post* aggregate changes. Our framework allows us to feed a specific, isolated shock (e.g., a US monetary policy tightening) into the solver.
**Mechanism Unpacking:** We can then cleanly unpack exactly how much of the resulting global wealth transfer is driven by **capital flight (Flow Effect)** versus the **repricing of existing global portfolios (Valuation Effect)**. This structural identification is the core power of the portfolio approach to global imbalances.


> **Note:** The paper's emphasis on the "post-2010 valuation reversal" highlights that NFA dynamics are driven not merely by flows, but significantly by valuation.

### 1.2 Why we need GE counterfactuals, not partial equilibrium

When conducting the attribution analysis of NFA changes, the paper discards the traditional partial equilibrium method of taking static differences and instead strictly adopts a **General Equilibrium (GE) counterfactual framework**. 

This is because international financial markets are highly endogenously interconnected: 
1. A shock to any primitive driver, such as central bank reserves, will trigger the substitution and reallocation of investors' portfolio weights.
2. This disrupts the market supply and demand balance, forcing asset prices and exchange rates to adjust. 
3. Concurrently, changes in prices and exchange rates will alter asset returns, dynamically revaluing global assets under management (Wealth/AUM).
4. This wealth revaluation in turn feeds back into holding demands.

Based on this dynamic transmission mechanism, at every step of the decomposition path—that is, whenever a primitive driver block is released—a general equilibrium must be **re-solved**. This allows prices, exchange rates, and portfolios to iteratively adjust together until comprehensive **"market clearing"** is achieved. 

This deduction path of repeatedly solving for fixed-point general equilibrium, rather than simply calculating static differences, constitutes the core and most challenging technical procedure in the counterfactual decomposition of this study.

---


# 2. Algorithm

The computational workflow of the entire decomposition can be broken down into three distinct layers.

## Code map (where to look)

| Component | What it does | Main file / function |
|---|---|---|
| Build counterfactual worlds | freeze selected primitives, label worlds by `cftype` |  `compute_cf()` |
| GE solver (major difficult step) | fixed-point / pseudo-Newton iterations to clear markets |  `RunCF()` |
| Primitive reset helpers | implement “freeze X at baseline” for chars/supply/flows/reserves | `CFHelperFunctions.R`|
| Flow vs valuation accounting | constant-price repricing and log decomposition | `DecompHelperFunctions.R` → `getDecomp()` |
| Block aggregation (paper tables) | sequential + cumulative block contributions | `DecompHelperFunctions.R` → `analyze_cf()` |

### Step A. Build Counterfactual "Worlds" by Freezing Primitive Drivers (`compute_cf()`)

The core objective of Step A is to construct a series of counterfactual "worlds" by freezing specific primitive drivers using the `compute_cf()` function. 

Specifically, the program follows this procedural logic:
1. **Define the Baseline**: It first treats the real-world dataset (`datin`) as the baseline **"full model / across"** world.
2. **Initialize Deduction Panel**: It copies the baseline data to create a new `cf_panel` for counterfactual deductions. 
3. **Freeze Drivers**: To construct each counterfactual world, the code uses `Reset` functions to forcibly **"freeze"** selected primitive drivers back to a historical baseline state (e.g., locking variables at their 2002 levels). 
4. **Synchronize Holdings**: It then calls `UpdateHoldings()` to ensure these artificially frozen settings translate into logically consistent initial asset holdings and portfolio weights. 
5. **Solve General Equilibrium**: Once the initial setup is complete, the program calls `RunCF()` on this frozen world to re-solve for the General Equilibrium (GE). 
6. **Aggregate and Label**: The resulting equilibrium data is saved as an independent object (e.g., `cf_vals_k`). Finally, all the deduced counterfactual worlds are aggregated together, with each assigned its respective `cftype` label to denote which drivers were manipulated.

### Code Reference (Excerpt from `RunCF.R`)

Finally, `compute_cf()` assigns a label to each counterfactual world and aggregates them into a single panel using `rbindlist`:

```r

cfout <- rbindlist(list(
  cf_vals_2[, cftype:='bench'],
  cf_vals_3[, cftype:='us_savings'],
  cf_vals_4[, cftype:='dm_savings_asia'],
  cf_vals_5[, cftype:='dm_savings_europe'],
  cf_vals_6[, cftype:='dmem_savings'],
  cf_vals_7[, cftype:='QE_US'],
  cf_vals_8[, cftype:='res_dm'],
  cf_vals_9[, cftype:='res_em'],
  cf_vals_10[, cftype:='rates_US'],
  cf_vals_11[, cftype:='rates_EMDM'],
  cf_vals_12[, cftype:='chars'],
  cf_vals_13[, cftype:='within'],
  cf_vals_1[, cftype:='across']
))
```

Therefore, the attribution method is: **Every time a set of economic primitive driver blocks is changed, the general equilibrium must be solved again to allow prices, exchange rates, and asset portfolios to adjust together to a "market clearing" state.**

---

### Step B. Solve GE Equilibrium Inside Each World (`RunCF()`)

This is one of the major difficult steps in the methodology: a General Equilibrium (GE) must be solved within each counterfactual world. 

[Image of macroeconomic general equilibrium iterative numerical solver]

#### math: Fixed-Point Problem

We need to solve for the following high-dimensional fixed points in each counterfactual world.：

$$
\mathbf{P}^* = \mathcal{F}(\mathbf{P}^*;\ \mathbf{X}_{\text{frozen}})
$$

with $\mathbf{P} = (\text{ner}_d,\ \ln P_{d,k},\ \ln S_{d,\text{st}})$ is the endogenous price vector，$\mathcal{F}(\cdot)$ is a mapping function jointly defined by the demand system, wealth dynamics, and market clearing.

The solution method is **pseudo-Newton**（pseudo-Newton），until the market clearing error is satisfied：

$$
\max_{d,k} | \text{gap}_{d,k} | < \text{tol} \quad (\text{tol}=1\mathrm{e}{-6})
$$

Within the equilibrium, the following endogenous objects are solved and updated iteratively (note that this is a fixed-point iteration, not a regression):
* `ner_d_loop`: Destination country exchange rate (Nominal Exchange Rate to USD).
* `ln_price_lc_d_loop`: Asset price (Long-term debt/equity priced in local currency).
* `ln_supply_loop`: Short-term debt supply (adjusted via quantity for pegged FX countries).
* `aum_loop`: Wealth / Assets Under Management (dynamically updated by returns when `endog.aum=TRUE`).
* `holding_loop`: Portfolio holdings (implied by weights $\times$ wealth).
* `gap`: Market clearing error (Supply market capitalization $-$ Demand).

---

#### (B1) Calculate Expected Excess Returns (`erx_loop`) Based on Current Prices/FX

This step synthesizes prices, exchange rates, and return predictors into a single expected return metric. Later, the portfolio weights will be updated based on $\exp(\beta_{erx} \times erx\_loop + preference)$.

```r
# File: Replication/Programs/RunCF.R (inside RunCF iteration)

cfin[, erx_loop :=
  b_ln_price_lc*ln_price_lc_d_loop + b_rer*(ner_d_loop - rel_cpi_d)
  - b_ln_price_lc_st*ln_price_lc_st_o_loop - b_rer_st*(ner_o_loop - rel_cpi_o)
  + rxfe_d
]
```

#### (B2) Update Wealth Dynamics via Returns (Revaluation $\rightarrow$ AUM)

This is a crucial mechanism in the GE framework: wealth is not static. The transmission channel works as follows:
Prices/FX adjust $\rightarrow$ Returns change $\rightarrow$ Existing asset stocks are revalued $\rightarrow$ Multiplied by flows $\rightarrow$ Yields new AUM. The updated AUM subsequently alters holding demands, which in turn impacts market clearing.


```r
# Calculate price returns (return_price_loop) and total returns (return_tot_loop)
cfin[year > 2002 & type != 'Equity',
     return_price_loop := ln_price_lc_d_loop - ln_price_lc_d_lag + (ner_d_loop - ner_d_lag)]
cfin[year > 2002 & type == 'Equity',
     return_price_loop := ln_price_lc_d_loop - ln_price_lc_d_lag]
cfin[year > 2002, return_tot_loop := return_price_loop + div]

# If endogenous AUM is enabled: update aum_loop via revaluation and flows
if(endog.aum==TRUE){
  cfin[year > 2002,
       revaluation_loop := sum(holding_lag * exp(return_tot_loop), na.rm=T),
       by=.(type, iso3_o, year)]
  # ...
}
cfin[year > 2002, aum_loop := revaluation_loop * flow]
```


#### (B3) Update Portfolio Weights: Within + Across (Nested Logit)

**Within**  

The attractiveness of country $d$ to investors within asset class $k$ is  

$$
\delta_{o,d,k} = \beta_{\text{erx}}\cdot\text{erx}_{d} + \text{tmppref}_{o,d,k}
$$

After normalization, we get：

$$
w_{o,d|k}^{\text{within}} = \frac{\exp(\delta_{o,d,k})}{1 + \sum_{d'}\exp(\delta_{o,d',k})} = \frac{\text{weightN}}{1+\text{weightD}}
$$

**Across**  

Calculate the included value for each asset class:

$$
\zeta_k = \frac{1}{\text{weight0}_k}
$$

Recalculate cross-class attractiveness：

$$
V_k = \zeta_k^\lambda \cdot \exp(\alpha_k + \xi_k)
$$

Final cross-class weights：

$$
w_k^{\text{across}} = \frac{V_k}{\sum_{k'}V_{k'}}
$$

**Final holding weight**（Two-level multiplication）：

$$
w_{o,d,k} = w_{o,d|k}^{\text{within}} \times w_k^{\text{across}}
$$

**Code**：

```r
# Within
cfin[ , weightN := 0]
cfin[w != 0 & w_out != 0,
     weightN := exp(b_erx * erx_loop + tmppref)]
cfin[w_out != 0, weightD := sum(weightN, na.rm=TRUE), by=.(type, iso3_o, year)]

cfin[ , weight := 0]
cfin[w != 0 & w_out != 0, weight := weightN / (1 + weightD)]
cfin[w_out != 0, weight0 := 1 / (1 + weightD)]
```

```r

# Across

```r
cfin[ , zeta_asset := 0]
cfin[w_out != 0, zeta_asset := 1 / weight0]
cfin[w_out != 0, weightN_asset := zeta_asset^lambda * exp(alpha + xi)]
...
tmp[ , weight_asset := weightN_asset / sum(weightN_asset), by=.(iso3_o, year)]

```


**Explanation:**

Within: Determines which specific country to invest in within the same asset class.

Across: Determines how to allocate capital among the three major categories: short-term debt, long-term debt, and equity.


These two nested layers jointly dictate the final aggregate holdings demand.


#### (B4) Obtain holding demand from weights $\times$ wealth, then calculate market clearing gap

```r
# holdings implied by weights
cfin[w_out != 0, holding_loop := total_aum_loop * weight_asset * weight]

# aggregate demand
cfin[ , demand := sum(holding_loop, na.rm=TRUE), by=.(type, iso3_d, year)]
cfin[type=='DebtShort', demand := sum(holding_loop, na.rm=TRUE), by=.(type, ccy_d, year)]

# market cap in USD
cfin[ , ln_mktcap_usd_loop := ln_supply_loop + ner_d_loop + ln_price_lc_d_loop]

# market clearing gap
cfin[ , gap := ln_mktcap_usd_loop - log(demand)]
```


**Explanation:** The market clearing gap is calculated as follows:

$$
gap = \log(\text{supply value}) - \log(\text{demand})
$$

A $gap > 0$ indicates that the "supply market cap is larger / demand is insufficient." This requires price/FX adjustments to either raise demand or lower the supply market cap.







##### Market-Clearing Condition

The solver iterates until the system reaches equilibrium, defined by the following market-clearing condition:

$$\text{Demand}^{USD}_{d,\ell,t} = \text{MktCap}^{USD}_{d,\ell,t}$$

The model achieves convergence when the implied gap between aggregate demand and total market capitalization approaches zero.



#### (B5) Adjust FX / prices using a pseudo-Newton update rule until convergence

Update exchange rates (using the short debt market to pin down FX):


##### Appendix Algorithm: GE Counterfactual Equilibrium

The counterfactual equilibrium is solved as a root-finding problem using an **approximate Newton Method**. The model seeks to zero out the market-clearing function $H(P)$ across all asset markets.

###### 1. Mathematical Objective

The solver targets the equilibrium price vector $P$ (including prices, FX, and short-debt quantities) where:

$$H(P) = 0$$

To find the root, we update $P$ using the Jacobian $J_H$:

$$P' = P - J_H^{-1} H(P)$$

> **Note:** Due to high dimensionality, the implementation approximates $J_H$ using its **diagonal elements** to ensure computational efficiency.

---

###### 2. Implementation Logic (`RunCF.R`)

In our `RunCF()` function, we implement this iteration by calculating the log-difference between supply and demand:

$$gap = \log(\text{MktCap}^{USD}) - \log(\text{Demand}^{USD})$$

We then update the Nominal Exchange Rate ($ner$) and prices ($P$) using diagonal-style scalers ($D$) and a learning rate ($scale$):

$$ner \leftarrow ner - scale \cdot D_{ner} \cdot gap$$
$$\ln P \leftarrow \ln P - scale \cdot D_{price} \cdot gap$$

The solver iterates until convergence is achieved: $\max(|gap|) < \text{tol}$.




```r
# update NER based on DebtShort market
cfin[type=='DebtShort',
     D_ner := sum(b_erx * b_rer * holding_loop * (1 - weight), na.rm=TRUE),
     by=.(ccy_d, year)]
cfin[, D_ner := (D_ner / demand)]
cfin[, D_ner := 1 / (-D_ner)]
cfin[abs(D_ner) > 1, D_ner := sign(D_ner)]

# floating FX: adjust ner
cfin[type=='DebtShort' & !(ccy_d %in% peggedfx),
     ner_d_loop := ner_d_loop - scale * D_ner * gap]

# pegged FX: adjust short debt quantity (supply) instead
cfin[type=='DebtShort' & ccy_d %in% peggedfx,
     ln_supply_loop := ln_supply_loop - scale * 1 * gap]
```


Update prices (LT debt & equity):


```r
cfin[ , D_price := sum(b_erx * b_ln_price_lc * holding_loop * (1 - weight), na.rm=TRUE),
     by=.(type, iso3_d, year)]
cfin[ , D_price := D_price / demand]
cfin[ , D_price := 1 / (1 - pmin(0, D_price))]
cfin[abs(D_price) > 1 , D_price := sign(D_price)]

cfin[type != 'DebtShort',
     ln_price_lc_d_loop := ln_price_lc_d_loop - scale * D_price * gap]
```

**Explanation:**

This represents the core of the "pseudo-Newton / fixed-point" algorithm:

It uses approximate derivatives (D_ner, D_price) to determine the update direction and step size.

scale controls the step size (too large may cause oscillations, while too small results in slow convergence).

Derivative clipping is applied (abs(D_ner) > 1 takes sign) to guarantee numerical stability.

Convergence criterion:
```r
tmpgap <- max(abs(cfin$gap), na.rm=TRUE)
if(tmpgap < tol) break
```


# 3. Flow vs valuation: the exact decomposition formula in code (getDecomp())

#### Paper Construction: Constant-Price NFA

To compute the Constant-Price Net Foreign Assets ($\widehat{NFA}^{ConstPrice,j}_{US,t}$), the framework "reprices" step-$j$ holdings using the prices and exchange rates from step $j-1$.

##### 1. Price Conversion Ratio (Theoretical)
The model defines a conversion ratio $\rho^j_t(n,\ell)$ to revert the dollar price-per-share of asset $n$ in class $\ell$ from the current step ($j$) back to the previous step ($j-1$):

$$\rho^j_t(n,\ell) = \frac{\widehat{P}^{j}_t(n)\widehat{E}^{j}_t(n)\widehat{S}^{j}_t(n,\ell)}{\widehat{P}^{j-1}_t(n)\widehat{E}^{j-1}_t(n)\widehat{S}^{j-1}_t(n,\ell)}$$

---

##### 2. Implementation Logic (`getDecomp.R`)

In `DecompHelperFunctions.R::getDecomp()`, we implement this constant-price logic via a return-based repricing method. This approach is mathematically consistent with the paper's "constant price/FX" concept but is optimized for return-series data:

$$Holding^{const} = Holding^{end} \cdot \exp(r^{start} - r^{end})$$

**Key Equivalence:**
* **$r^{start}$**: Returns at the baseline (step $j-1$).
* **$r^{end}$**: Returns at the counterfactual (step $j$).
* The exponential term effectively strips out the price impact of the current step to isolate the quantity effect.



### 3.1 Extract U.S.-Related Asset and Liability Holdings `getQ`

```r
# File: Replication/Programs/DecompHelperFunctions.R

getQ <- function(datin) {
  tmp <- datin[iso3_o=='USA' | iso3_d=='USA',
               .(iso3_o, iso3_d, type, year, return_tot, holding, aum,
                 ln_price_lc_d, ner_d, div)]
  tmp[, .(iso3_o, iso3_d, type, year, return_tot, holding)]
}、
```
**Explanation:**

`iso3_o == 'USA'` represents U.S. external assets where U.S. investors are holding foreign assets.

`iso3_d == 'USA'` represents U.S. external liabilities where foreign investors are holding U.S. assets.

### 3.2 Core: Construct Constant-Price Holdings to Isolate Flows `holding_constprice`

```r
# File: Replication/Programs/DecompHelperFunctions.R

both[, holding_constprice := holding.y * exp(return_tot.x - return_tot.y)]
both[iso3_d=='OUT', holding_constprice := holding.y]
```

Here `.x` denotes the start world and `.y` denotes the end world.

Key Intuition:

`holding.y `is the nominal holding in the end world.

If we want to strip out the price and return effects, we must reprice the end-world holdings according to the start-world returns.

`exp(return_tot.x - return_tot.y)` performs this repricing via logarithmic transformation.

This can be understood as:
**constant-price holdings = end-world quantities × start-world prices**
Implemented in the code using the exponential form of the return differential.



### 3.3 Calculate NFA and Extract the Three Components: Total, Quantity, and Revaluation

```r
# aggregate assets and liabilities
tmpa <- both[iso3_o=='USA',
             .(assets.x=sum(holding.x), assets.y=sum(holding.y),
               assets_constprice=sum(holding_constprice)),
             keyby=.(year)]

tmpl <- both[iso3_d=='USA',
             .(liab.x=sum(holding.x), liab.y=sum(holding.y),
               liab_constprice=sum(holding_constprice)),
             keyby=.(year)]

both <- merge(tmpa, tmpl, by='year')

both[, NFA.x := assets.x - liab.x]
both[, NFA.y := assets.y - liab.y]
both[, NFA_constprice := assets_constprice - liab_constprice]

# three deltas
both[, delta := log(NFA.y / NFA.x)]                         # total log change
both[, delta_Q := log(NFA_constprice / NFA.x)]              # "quantity/flow"
both[, delta_reval := log(1 + (NFA.y - NFA_constprice)/NFA.x)]  # "valuation"
```

**Explanation:**

`delta`: Total NFA change between the two worlds in log form.

`delta_Q`: The change under constant-price conditions removing price and return effects $\rightarrow$ closer to the contribution of flows and quantity.

`delta_reval`: The residual portion $\rightarrow$ the contribution of valuation and revaluation.


# 4. From Many Worlds to Paper-Style Block Decomposition (`analyze_cf()`)

With the `getDecomp()` function, we can chain the counterfactual worlds together to calculate the cumulative contribution by block. Inside `analyze_cf()`, the code performs two types of comparisons:

### 4.1 Sequential Comparison (Step-by-Step Transition)


```r
alltypes <- c('bench', 'us_savings', ..., 'within', 'across')

allout <- foreach(idx=2:length(alltypes)) %do% {
  getDecomp(dat, alltypes[idx-1], alltypes[idx])
}
```
**Explanation:**

This represents the sequential decomposition path: releasing the primitive blocks step-by-step from one world to the next.

### 4.2 Key Cumulative Blocks (Corresponding to Tables II, III, and IV in the Paper)

```r
cum1 <- getDecomp(dat,'bench','across')           # total
cum2 <- getDecomp(dat,'bench','dmem_savings')     # total_savings
cum3 <- getDecomp(dat,'dmem_savings','res_em')    # total_reserves
cum4 <- getDecomp(dat,'res_em','rates_EMDM')      # total_rates
cum5 <- getDecomp(dat,'rates_EMDM','across')      # total_demand
```


**Explanation of the Decomposition Sequence:**

`bench -> dmem_savings`: Restoring the savings and issuances block from the frozen baseline state $\rightarrow$ yields the contribution of savings/issuances.

`dmem_savings -> res_em`: Restoring reserves on top of the already released savings $\rightarrow$ yields the contribution of central bank reserves.

`res_em -> rates_EMDM:` Restoring interest rates next $\rightarrow$ yields the contribution of interest rates and monetary policy.

`rates_EMDM -> across:` Finally restoring investor demand $\rightarrow$ yields the contribution of investor demand and asset characteristics.

# 5. Why decomposition is one of the “major difficult step”

**Difficulty 1: High-Dimensional GE Fixed Point**

This approach is not a simple regression, but a complex, high-dimensional fixed-point iterative equilibrium solving process. Endogenous variables—including asset prices (LT/Equity), exchange rates (NER), pegged short-term debt supply, nested portfolio weights, and endogenous wealth revaluation (AUM)—continuously feed back into each other. The entire system must be solved iteratively until all global markets clear simultaneously ($gap \to 0$).

**Difficulty 2: Path Dependence**
The structural decomposition exhibits strong path dependence, meaning the sequence in which the primitive blocks are accumulated (e.g., `bench` $\to$ `savings` $\to$ `reserves` $\to$ `rates` $\to$ `demand` $\to$ `across`) directly impacts their calculated marginal contributions. The paper deliberately selects an economically interpretable sequence as the baseline, though robustness can be verified by altering the order or averaging the effects across different paths.

**Difficulty 3: Numerical Convergence**

The update mechanism in `RunCF()` acts as a pseudo-Newton algorithm, which introduces significant numerical challenges. The step size (`scale`) must be carefully tuned, as too large a step causes severe oscillations, while too small a step leads to extremely slow convergence. Furthermore, derivative clipping on `D_ner` and `D_price` is necessary to guarantee numerical stability, and a strict tolerance threshold (`tol`) must be defined to establish the acceptable error margin for market clearing.
