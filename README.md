# Compute Macro Index (CMI) v0.1 — Technical Methodology & Data Contract

## Executive Overview
The **Compute Macro Index (CMI)** measures the physical and financial viability of hyperscale AI infrastructure. Unlike spot-market price indexes that track cloud API rental rates, CMI evaluates two distinct spreads: the raw physical generation margin ($V_{physical}$) and the fully leveraged, macro-adjusted cost of compute ($V_c$).

* **Physical Coverage ($V_{physical} > 0$):** Indicates spot market revenues successfully cover real-world energy, hardware depreciation, and cooling drag at the server level.
* **Levered Shortfall ($V_c < 0$):** Indicates that despite physical profitability, the systemic burden of low utilization, corporate CapEx, and off-balance-sheet debt creates a negative levered spread. Current spot rates are under-earning their true replacement and capital costs.
---

## Monitored Targets & Hardware Register
To maintain an accurate macro view, the CMI engine scrapes a specific basket of hyperscale entities and GPU hardware classes.

**Hyperscaler SEC Targets (CapEx & Debt):**
* Microsoft (MSFT)
* Amazon (AMZN)
* Alphabet (GOOGL)
* Meta (META)
* Oracle (ORCL)
* SpaceX (SPCX)

**Hardware Tracking Basket (Spot Rates & Power Draw):**
* **Nvidia:** A100 (80GB), H100 (80GB), H200 (141GB), B100, B200, B300, GH200, L40S
* **AMD:** MI300X, MI325X, MI350X

---

## Variable Definitions & Oracle Mapping
The CMI relies on a blend of public API feeds (SEC filings, EIA power grids, National Weather Service) and active scraping of private market data (bespoke PPAs, unlisted secondary market hardware liquidations).

| Variable | Metric Name | Source / Pipeline | Ref Value / Range |
| :--- | :--- | :--- | :--- |
| $E_{capex}$ | Infrastructure CapEx | `oracle_sec.py` (SEC EDGAR API: PP&E Net) | 437.03B |
| $D_{spv}$ | Off-Balance-Sheet Debt | `oracle_sec.py` (SEC EDGAR API: Minimum Payments) | 250.00B |
| $R_{ext}$ | Verified External Revenue | `oracle_revenue.py` (SEC EDGAR API: Segment Cloud) | 28.22B |
| $O_a$ | Circular Offtake Agreements | `oracle_revenue.py` (Round-tripped capital proxy) | 7.06B |
| $P_{futures}$ | Spot Market Compute Rate | `oracle_gpu.py` (Vast.ai, RunPod, Lambda blended) | 2.86 – 3.35 / hr |
| $U_r$ | Fleet Utilization Rate | `oracle_utilization.py` (Blended cluster APIs) | 34.4% – 59.4% |
| $P_{kwh}$ | Commercial Power Rate | `oracle_energy.py` (EIA API + Private PPAs) | 0.1234 / kWh |
| $R_{interest}$ | Macro Cost of Capital | `oracle_macro.py` (FRED API: SOFR / DGS10) | 3.62% – 5.30% |
| $V_{resale}$ | Hardware Collateral Floor | `oracle_hardware.py` (Secondary resale floor) | 25,000 (Fixed v0.1) |
| $E_{pue}$ | Dynamic HVAC Cooling Drag | `oracle_pue.py` (Weather-adjusted PUE) | 1.151 – 1.233 |
| $C_{gpu}$ | Hardware Acquisition Cost | `config_tags.py` (Blended H100/H200 basket baseline) | 32,727.27 |
| $T_{life}$ | Depreciation Lifespan | `config_tags.py` (5-Year / 43,800-Hour baseline) | 43,800 hrs |
| $W_c$ | Active Power Draw | `config_tags.py` (KW draw per GPU node) | 0.7955 kW |

---

## Baseline Configuration Tags (`config_tags.py`)
The CMI engine utilizes standardized baseline assumptions for physical hardware constraints:
* **Hardware Depreciation Horizon ($T_{life}$):** 43,800 hours (Standardized 5-year operational lifespan).
* **Model Utilization Rate ($U_r$):** Benchmark baseline set to 40% (0.40).
* **Baseline PUE:** 1.15 facility efficiency constant (adjusts dynamically based on weather oracles).
* **Public Grid Hubs:** Telemetry currently tracks US interconnects including Northern Virginia (Ashburn), Texas (ERCOT), Ohio, Georgia, and California.

---

## Core Mathematical Engine (`engine.py`)

### 1. Organic Revenue Isolation ($R_{organic}$)
Strips circular vendor-financing and VC round-tripping from top-line reported cloud revenue:

$$R_{organic} = R_{ext} - O_a$$

### 2. Unleveraged Physical Cost Basis ($C_p$)
Calculates the direct hourly physical cost to operate one unit of compute, combining silicon depreciation, interest on hardware capital, active power draw, and real-time PUE thermal penalties:

$$C_p = \left( \frac{C_{gpu} - V_{resale}}{T_{life}} \right) \cdot (1 + R_{interest}) + (W_c \cdot P_{kwh} \cdot E_{pue})$$

### 3. Circular Premium Multiplier ($M_c$)
Quantifies systemic debt leverage and capital friction per active compute unit by scaling off-balance-sheet debt against true organic cash flow and utilization:

$$M_c = \left( \frac{D_{spv} + E_{capex}}{R_{organic}} \right) \cdot U_r$$

### 4. Coverage Spread ($V_{physical}$ & $V_c$)
Evaluates spot market rental rates against both the base thermodynamic cost and the fully leveraged macro cost:

**Unlevered Physical Spread (Hardware Margin):**
$$V_{physical} = P_{futures} - C_p$$

**Compute-to-Fiat Cross (Macro Index):**
$$V_c = P_{futures} - (C_p \cdot M_c)$$

**Index Status Definitions:**
* **COVERED ($V_{physical} > 0$, $V_c > 0$):** Spot revenues safely clear both physical generation costs and the macro leverage overlay.
* **PLANT COVERED, LEVERED SHORT ($V_{physical} > 0$, $V_c < 0$):** Hardware generates positive unit economics, but fails to cover the systemic capital-adjusted deficit.
* **PHYSICAL SHORT ($V_{physical} < 0$, $V_c < 0$):** Spot revenues fail to cover even the base thermodynamic and depreciation costs.
---

## Worked Telemetry Sample (Timestamp 1789365863)

Below is an audited single-hour telemetry row extracted from the baseline dataset:

### 1. Input Telemetry
* **$P_{futures}$**: 3.0895 / hr
* **$E_{pue}$**: 1.160
* **$P_{kwh}$**: 0.1234 / kWh
* **$U_r$**: 49.17%
* **$R_{interest}$**: 3.62%
* **$W_c$**: 0.7955 kW

### 2. Intermediate Engine Outputs

**Organic Revenue ($R_{organic}$):**

$$R_{organic} = 28.2238 - 7.0560 = 21.1678$$

*(Result: 21.17B)*

**Physical Cost Basis ($C_p$):**

$$C_{power} = 0.7955 \cdot 0.1234 \cdot 1.160 = 0.1139$$

$$C_{depreciation} = \left( \frac{32,727 - 25,000}{43,800} \right) \cdot (1 + 0.0362) = 0.1828$$

$$C_p = 0.1828 + 0.1139 = 0.2967$$

*(Result: 0.2967 / hr)*

**Circular Premium Multiplier ($M_c$):**

$$M_c = \left( \frac{250.00 + 437.03}{21.1678} \right) \cdot 0.4917$$

$$M_c = 32.45 \cdot 0.4917 = 15.96$$

*(Result: 15.96)*

### 3. Final Master Print

**Unlevered Physical Spread (Hardware Operations):**
$$V_{physical} = P_{futures} - C_p$$
$$V_{physical} = 3.0895 - 0.2967 = +2.7928$$
* **STATUS: PLANT COVERED ( +$2.793 / hr )**

**Leveraged Macro Spread ($V_c$ - The CMI Index):**
$$V_c = P_{futures} - (C_p \cdot M_c)$$
$$V_c = 3.0895 - (0.2967 \cdot 15.96)$$
$$V_c = 3.0895 - 4.7353 = -1.6458$$
* **STATUS: LEVERED SHORT ( -$1.646 / hr )**
---

## Methodological Disclosures & Version Notes
1. **Friction Baseline:** Silicon degradation, physical maintenance overhead, and localized water consumption penalties are currently modeled within $T_{life}$ decay constraints.
2. **Fixed Corporate Baseline:** SEC variables ($E_{capex}$, $D_{spv}$) update quarterly on official 10-Q/10-K filing releases.
3. **Hardware Fleet Blending:** $C_{gpu}$ and $W_c$ are calculated as a blended basket across the monitored hardware register to reflect macro fleet averages rather than isolated single-node performance.
4. **v0.1 Status & Private Markets:** CMI is in active, developing R&D. While current baseline variables ($P_{kwh}$, $V_{resale\_blended}$) heavily weight public and semi-public commercial pricing models, the index is aggressively building ingestion pipelines to map bespoke private PPAs (e.g., dedicated nuclear) and unlisted secondary-market hardware liquidations.

---
*© 2026 LCS. All Rights Reserved. This methodology and index telemetry are provided for informational purposes and structural research.*
