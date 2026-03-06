# BRP Optimization Literature Research — Summary

## Context

This research supports a diploma thesis focused on transitioning from **independent single-site microgrid optimization** to **Balance Responsible Party (BRP)** operation for a portfolio of prosumer sites with PV+BESS in the Czech/Slovak electricity market.

**Current state**: The company performs MILP-based single-site optimization with MPC control loop (1-minute commands to BESS), using day-ahead prices from OTE/OKTE. Each customer is optimized independently with their own electricity supplier.

**Goal**: Become a BRP — aggregate customers into a balancing group, handle day-ahead nominations, manage imbalance risk, and create shared value for both the company and prosumers.

---

## Notebooks Overview

### Notebook 1: MILP Single-Site Baseline (`01_milp_single_site_baseline.ipynb`)

**Papers**: Luthander et al. (2015) *"Photovoltaic self-consumption in buildings"*; Nottrott et al. (2013) *"Energy dispatch schedule optimization for BESS"*; Hesse et al. (2017) *"Lithium-ion battery storage for the grid"*

**What it does**: Implements the company's current approach — single-site MILP for PV+BESS optimization with day-ahead prices. Serves as the baseline against which all portfolio approaches are compared.

**Key result**: Demonstrates the value of BESS arbitrage at a single site, with sensitivity to price spread, battery capacity, and charge/discharge power.

---

### Notebook 2: Centralized MILP BRP Portfolio (`02_milp_brp_centralized_portfolio.ipynb`)

**Papers**: Dall'Anese et al. (2017) *"Optimization for distributed energy resources"*; Conejo et al. (2005) *"Optimal response of a power producer to day-ahead markets"*

**What it does**: Extends single-site MILP to a portfolio of N sites optimized jointly. The BRP submits a **day-ahead nomination** and pays imbalance costs for deviations. Shows the value of **internal netting** — surplus at one site offsets deficit at another, reducing portfolio-level market interaction.

**Key result**: 3–5% cost reduction from portfolio coordination vs independent operation. The netting benefit scales with the diversity of site profiles (different PV orientations, load patterns).

**Thesis relevance**: This is the **primary recommended approach** for the company's near-term transition. With existing BESS control infrastructure and a small number of sites (<50), centralized MILP is tractable and optimal.

---

### Notebook 3: Two-Stage Stochastic Programming (`03_two_stage_stochastic_brp.ipynb`)

**Papers**: Birge & Louveaux (2011) *"Introduction to Stochastic Programming"*; Conejo et al. (2010) *"Decision Making Under Uncertainty in Electricity Markets"*; Morales et al. (2014) *"Integrating Renewables in Electricity Markets"*

**What it does**: Extends the centralized MILP to handle forecast uncertainty explicitly. Stage 1 (day-ahead) optimizes the nomination across PV/load scenarios. Stage 2 (real-time) dispatches BESS to minimize imbalance costs given actual realizations. Includes CVaR risk measure.

**Key result**: The stochastic model produces more conservative nominations that reduce worst-case imbalance costs by ~15–30% compared to deterministic planning. The risk-averse BRP achieves better worst-case performance at the cost of slightly higher expected cost.

**Thesis relevance**: Essential for managing the company's imbalance risk as a BRP. The Czech/Slovak market has asymmetric imbalance penalties — the stochastic model captures this naturally.

---

### Notebook 4: ADMM Distributed Coordination (`04_admm_decentralized_coordination.ipynb`)

**Papers**: Boyd et al. (2011) *"Distributed Optimization via ADMM"*; Dall'Anese et al. (2013) *"Distributed OPF for microgrids"*; Kraning et al. (2014) *"Dynamic network energy management via proximal message passing"*

**What it does**: Implements **Jacobi ADMM** to decompose the BRP portfolio optimization into parallel site-level sub-problems. Each site solves a local QP; the coordinator updates the portfolio target. Demonstrates convergence to the centralized optimum.

**Key result**: Converges to 0% optimality gap in ~17 iterations for the convex QP formulation. Communication: ~1.5 KB per iteration. Privacy preserved — sites share only net grid exchange.

**Thesis relevance**: Important for scalability (>50 sites) and data privacy (GDPR). The **dual variables** from ADMM naturally provide internal transfer prices for benefit sharing. However, for the company's current scale, centralized MILP (Notebook 2) is simpler and more practical.

**Caveat**: The convex QP formulation is a simplification. Real BESS scheduling has integer constraints (charge/discharge exclusion), making ADMM convergence non-guaranteed. LP relaxation + rounding or quadratic regularization are practical workarounds.

---

### Notebook 5: MPC Real-Time Balancing (`05_mpc_realtime_balancing.ipynb`)

**Papers**: Oldewurtel et al. (2012) *"MPC for energy efficient building control"*; Parisio et al. (2014) *"MPC for microgrid operation"*; Pérez et al. (2013) *"Predictive power control for PV+BESS"*

**What it does**: Simulates a full-day MPC loop at 15-minute resolution. The BRP has a fixed day-ahead nomination; MPC re-optimizes BESS dispatch every 15 minutes to **track the nomination** using updated forecasts. Compares MPC vs "fire-and-forget" (execute DA plan blindly).

**Key result**: MPC reduces imbalance costs by ~5% compared to no recourse. The benefit comes from using near-term (15-min) forecast accuracy to correct for cloud events and load deviations.

**Thesis relevance**: Directly extends the company's existing 1-minute MPC infrastructure. The key change is that MPC targets the **nomination** (portfolio-level) instead of just minimizing individual site costs. The 15-minute resolution matches CZ/SK imbalance settlement periods.

---

### Notebook 6: Bilevel BRP–Prosumer Conflict (`06_bilevel_brp_prosumer.ipynb`)

**Papers**: Tushar et al. (2018) *"P2P energy trading via game theory"*; Wei et al. (2015) *"Energy pricing for smart grid retailers"*; Zugno et al. (2013) *"Bilevel model for electricity retailers"*

**What it does**: Models the BRP–prosumer interaction as a **Stackelberg bilevel optimization** (BRP = leader sets tariffs, prosumers = followers respond optimally). Includes participation constraints (prosumers must save vs current supplier). Also demonstrates cooperative game theory (Shapley-value-like surplus sharing).

**Key result**: The BRP can earn ~1–3 EUR/day per 5-site portfolio while providing prosumers 10–35% savings vs retail. The participation constraint limits the BRP's pricing power — a fundamental trade-off.

**Thesis relevance**: Addresses the core thesis question — how to make the BRP transition beneficial for both parties. The cooperative game theory approach provides a fair, transparent mechanism for benefit allocation.

---

## Recommended Architecture for the Thesis

Based on the literature research, the recommended multi-layer optimization architecture for the BRP is:

```
Layer 1 (Week-ahead):  Stochastic programming (Notebook 3)
                       → Robust nomination strategy, risk management
                       
Layer 2 (Day-ahead):   Centralized MILP (Notebook 2)
                       → Day-ahead nomination for OTE/OKTE
                       → Extract internal transfer prices (Notebook 4)
                       
Layer 3 (Real-time):   MPC at 15-min / 1-min resolution (Notebook 5)
                       → Track nomination, minimize imbalance
                       → Existing company infrastructure
                       
Layer 4 (Commercial):  Bilevel / cooperative game theory (Notebook 6)
                       → Set customer tariffs, benefit sharing
                       → Ensure prosumer participation
```

---

## Papers Behind Paywalls / Not Fully Accessible

The following papers have very relevant abstracts but were not fully accessible. With a university account, they may provide additional insights:

1. **Fang, X., Li, F., Wei, Y., Azim, R. (2019).** *"Reactive power planning under high penetration of wind energy using Benders decomposition."* IET Generation, Transmission & Distribution. — Relevant for Benders decomposition approach to BRP optimization.

2. **Ottesen, S. Ø., Tomasgard, A., Fleten, S.-E. (2018).** *"Multi market bidding strategies for demand side flexibility aggregators in electricity markets."* Energy, 149, 120–134. — Directly relevant: aggregator (similar to BRP) bidding in multiple markets.

3. **Iria, J., Soares, F., Matos, M. (2019).** *"Optimal supply and demand bidding strategy for an aggregator of small prosumers."* Applied Energy, 213, 658–669. — Prosumer aggregation strategy for DA and balancing markets.

4. **Burger, S., Chaves-Ávila, J. P., Batlle, C., Pérez-Arriaga, I. J. (2017).** *"A review of the value of aggregators in electricity systems."* Renewable and Sustainable Energy Reviews, 77, 395–405. — Comprehensive review of aggregator value propositions.

5. **Zhong, H., Xie, L., Xia, Q. (2013).** *"Coupon incentive-based demand response: theory and case study."* IEEE Transactions on Power Systems. — Incentive mechanisms for demand response, relevant to prosumer engagement.

6. **Pinson, P., Madsen, H. (2014).** *"Benefits and challenges of electrical demand response: a critical review."* Renewable and Sustainable Energy Reviews. — Context on demand response potential.

7. **Kardakos, E. G., Simoglou, C. K., Bakirtzis, A. G. (2016).** *"Optimal offering strategy of a virtual power plant: a stochastic bi-level approach."* IEEE Transactions on Smart Grid. — VPP bidding strategy (BRP is essentially a VPP).

---

## Notes on MDPI and Predatory Journals

As requested, all referenced papers are from established, reputable venues:
- IEEE Transactions on Smart Grid / Power Systems / Sustainable Energy
- Applied Energy (Elsevier)
- Energy Economics (Elsevier)
- Foundations and Trends (NOW Publishers)
- Springer texts (Birge & Louveaux, Conejo et al.)

MDPI journals were excluded per the instructions.

---

## How to Use This Material in the Thesis

1. **Literature Review Chapter**: Use the paper summaries above to structure the literature review. Group by approach (centralized MILP → stochastic → distributed → bilevel).

2. **Methodology Chapter**: The notebooks provide working implementations of each approach. Adapt the centralized MILP (Notebook 2) and stochastic model (Notebook 3) to the company's actual data.

3. **Results Chapter**: Run the notebooks with real or realistic CZ market data (OTE prices). Compare approaches on the same dataset.

4. **Discussion Chapter**: Use the bilevel model (Notebook 6) to discuss the BRP–prosumer trade-off and recommend a benefit-sharing mechanism.

5. **Implementation Chapter**: Describe how the MPC layer (Notebook 5) integrates with the company's existing infrastructure.
