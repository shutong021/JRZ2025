# Major Difficult Step (2): Counterfactual Decomposition (GE + Flow vs Valuation)

Goal:
We explain (i) code structure, (ii) steps of analysis, and (iii) the major difficult step.
We focus on analysis structure + algorithms, not programming syntax.

## 0. Where this part sits in the whole code pipeline

### Master pipeline: `Replication/Programs/Run.R`
`Run.R` is a “table of contents” that sources all scripts in order:

1. **Import** raw datasets  
2. **Build** merged panels (holdings, returns, prices, FX, characteristics, reserves, etc.)  
3. **Run analysis** (this includes instruments + counterfactuals + decomposition)  
4. **Output** paper tables/figures

For decomposition specifically, it is executed inside:

- `Replication/Programs/SensitivityAnalysis.R`  (main analysis driver)
  - `source('IVHelperFunctions.R')`
  - `source('CFHelperFunctions.R')`
  - `source('DecompHelperFunctions.R')`
  - `source('RunCF.R')`

So the decomposition “engine” is:  
**`compute_cf()` (scenario construction) → `RunCF()` (GE solver) → `analyze_cf()` (flow vs valuation + tables).**

---

## 1. What decomposition tries to answer (paper-side logic)

The paper studies why U.S. Net Foreign Assets (NFA) evolves the way it does, and why **valuation effects** reverse after 2010.

A key accounting identity for any external balance sheet is:

> **Change in NFA = Flow component + Valuation component**

- **Flows**: net purchases / net issuance (quantity changes)
- **Valuation**: changes in prices + exchange rates + returns (revaluation of the existing stock)

The paper then goes further: it attributes these changes to **blocks of “primitive drivers”** such as:

1) **Savings & Issuances** (capital flows, issuance, dividends)  
2) **Monetary policy** (reserves, QE, interest rate environment)  
3) **Demand shifts** (characteristics + latent demand within/across asset classes)

Because prices, FX and portfolio weights jointly adjust in equilibrium, attribution must be done under **general equilibrium (GE) counterfactuals**, not partial equilibrium.

---

## 2. The counterfactual design in code: `compute_cf()` (in `RunCF.R`)

### 2.1 Scenario naming (“cftype”)
At the end of `compute_cf()`, the code stacks a list of scenarios and tags them with `cftype`:

- `'bench'` benchmark with key primitives reset
- `'us_savings'`, `'dm_savings_asia'`, `'dm_savings_europe'`, `'dmem_savings'` (savings/issuance blocks)
- `'QE_US'`, `'res_dm'`, `'res_em'` (monetary policy via central bank holdings)
- `'rates_US'`, `'rates_EMDM'` (rate environment)
- `'chars'`, `'within'` (characteristics and within-asset latent demand)
- `'across'` (full model / across-asset latent demand also active)

This ordered list is crucial: it defines the sequential decomposition path.

### 2.2 “Reset” functions = how primitives are held fixed
`compute_cf()` builds each scenario by starting from the observed panel `datin` and applying reset rules:

- `ResetChars(cf_panel, datin, ...)`  
  Freeze (or partially freeze) **country characteristics and/or latent demand components** back to earlier values.
- `ResetSupply(cf_panel, datin, countries=...)`  
  Freeze **issuance / supply** (possibly for selected countries).
- `ResetFlows(cf_panel, datin, countries=...)`  
  Freeze **dividends and capital flows (savings)**.
- `ResetCBSupply(cf_panel, datin, cbs=...)`  
  Freeze **central bank reserve holdings** for selected CB groups.
- `UpdateHoldings(cf_panel)`  
  Recompute holdings and weights consistent with the imposed resets.

**Interpretation**: each scenario is a “world” where some primitives are held at baseline while others evolve.

---

## 3. The GE solver (major difficult step): `RunCF()` (in `RunCF.R`)

`RunCF(cfin, ...)` takes a scenario panel and **solves for equilibrium** prices/FX/holdings by fixed-point iterations.

### 3.1 The equilibrium objects (what is solved endogenously)
In the code, the “loop variables” are (suffix `_loop`):

- `ner_d_loop`, `ner_o_loop` : exchange rates (destination/origin)
- `ln_price_lc_d_loop` : local-currency asset price (LT debt & equity)
- `ln_supply_loop` : (for pegged FX countries, ST supply adjusts instead of FX)
- `aum_loop` : wealth / assets under management
- `holding_loop` : holdings implied by current weights and wealth

### 3.2 Step-by-step algorithm inside each iteration

Each iteration does:

**(1) Update expected excess return `erx_loop`**
\[
erx = b_{p}\ln P_d + b_{rer}(\text{NER}_d - \text{RER proxy}) 
      - b_{p,st}\ln P_{st,o} - b_{rer,st}(\text{NER}_o - \text{RER proxy}) + \text{fixed effects}
\]
In code: `cfin[, erx_loop := ...]`

**(2) Update total return and revaluation (wealth dynamics)**
- compute `return_price_loop`, then `return_tot_loop = return_price_loop + div`
- if `endog.aum = TRUE`, revalue last period holdings and update `aum_loop`

**(3) Update within-asset portfolio weights**
Within an asset class, weights follow a (logit-like) structure:
\[
w_{i\to d} \propto \exp(b_{erx}\cdot erx_{d} + \text{preference shifters})
\]
In code: `weightN := exp(b_erx * erx_loop + tmppref)`, then normalize to `weight`.

**(4) Update across-asset weights (nested-logit layer)**
The code constructs an “inclusive value / zeta” type term and updates cross-asset allocation:
- compute `weightN_asset := zeta_asset^lambda * exp(alpha + xi)`
- normalize to `weight_asset`

**(5) Update holdings and compute market clearing gaps**
Holdings implied by wealth and weights:
\[
\text{holding}_{i\to d} = \text{total\_aum}_{i}\cdot \text{weight\_asset}\cdot \text{weight}
\]
Then aggregate demand by destination:
- `demand := sum(holding_loop)` by market
Market capitalization in USD:
\[
\ln \text{mktcap}_{d} = \ln \text{supply}_{d} + \text{NER}_{d} + \ln P_{d}
\]
Gap:
\[
gap = \ln \text{mktcap}_{d} - \ln \text{demand}_{d}
\]
Convergence criterion: `max(abs(gap)) < tol`.

**(6) Update FX and prices to reduce gaps (approx Newton step)**
- For **short-term debt** markets, adjust `ner_d_loop` for floating FX currencies.
- For pegged FX set, adjust `ln_supply_loop` instead.
- For **LT debt & equity**, update `ln_price_lc_d_loop`.

This is the numerical core: update rules are proportional to `gap` scaled by `scale`, with safeguards (clipping derivatives).

### 3.3 Why this is “major difficult”
- It is a **high-dimensional fixed point**: prices, FX, wealth, and portfolio weights interact.
- It is **not a simple regression**: you must compute equilibrium each time primitives change.
- It is **numerically delicate**: tolerance, step size (`scale`), derivative clipping all matter.

---

## 4. Turning counterfactual paths into decomposition: `analyze_cf()` (in `DecompHelperFunctions.R`)

The output of `compute_cf()` is a big table with multiple `cftype` worlds.
Decomposition compares them sequentially.

### 4.1 Compute U.S. NFA in each world: `getQ()`
`getQ(datin)` extracts all positions where U.S. is origin or destination and returns holdings and returns.

### 4.2 Flow vs valuation split via a “constant-price” counterfactual: `getDecomp()`

Given two worlds (starttype → endtype):
- `val1 = getQ(world=starttype)`  
- `val2 = getQ(world=endtype)`  
merge into `both`.

**Constant-price holdings**
The code constructs:
\[
\text{holding}^{const} = \text{holding}^{(end)}\cdot \exp\left(r^{(start)} - r^{(end)}\right)
\]
This is “what the end-world holdings would be worth if priced using start-world returns.”

Then it aggregates U.S. assets and liabilities:

- U.S. assets = sum holdings where `iso3_o == 'USA'`
- U.S. liabilities = sum holdings where `iso3_d == 'USA'`

Compute:
- \(NFA^{(start)}\), \(NFA^{(end)}\), and \(NFA^{const}\)

**Three log contributions**
- Total change:
\[
\delta = \ln\left(\frac{NFA^{end}}{NFA^{start}}\right)
\]
- Quantity/flow component (constant prices):
\[
\delta_Q = \ln\left(\frac{NFA^{const}}{NFA^{start}}\right)
\]
- Valuation component:
\[
\delta_{reval} = \ln\left(1 + \frac{NFA^{end}-NFA^{const}}{NFA^{start}}\right)
\]
(so that roughly \(\delta \approx \delta_Q + \delta_{reval}\) for small changes)

In code, these are `delta`, `delta_Q`, `delta_reval`.

### 4.3 Aggregation into paper-style tables
`analyze_cf(dat)` loops across all `cftype` transitions and also constructs cumulative blocks:
- `bench → across` = total
- `bench → dmem_savings` = total savings/issuances
- `dmem_savings → res_em` = total reserves
- `res_em → rates_EMDM` = total rates
- `rates_EMDM → across` = total demand

Then it computes averages (post-2002) and reports:
- log changes * 100 (percent)
- level changes (exp(delta)-1)
- pre vs post split for valuation (exorbitant privilege reversal)

---

## 5. A clean 10-minute presentation plan

# holdings implied by weights
cfin[w_out != 0, holding_loop := total_aum_loop * weight_asset * weight]

# aggregate demand
cfin[ , demand := sum(holding_loop, na.rm=TRUE), by=.(type, iso3_d, year)]
cfin[type=='DebtShort', demand := sum(holding_loop, na.rm=TRUE), by=.(type, ccy_d, year)]

# market cap in USD
cfin[ , ln_mktcap_usd_loop := ln_supply_loop + ner_d_loop + ln_price_lc_d_loop]

# market clearing gap
cfin[ , gap := ln_mktcap_usd_loop - log(demand)]
- GE solver: `RunCF()` in `Replication/Programs/RunCF.R`
- Reset primitives: `ResetChars/ResetSupply/ResetFlows/ResetCBSupply/UpdateHoldings` in `Replication/Programs/CFHelperFunctions.R`
- Flow vs valuation and tables: `getDecomp()` + `analyze_cf()` in `Replication/Programs/DecompHelperFunctions.R`
- Driver script: `Replication/Programs/SensitivityAnalysis.R`

