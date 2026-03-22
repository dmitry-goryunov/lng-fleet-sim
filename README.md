# LNG Fleet Portfolio Optimiser

**Stochastic portfolio optimisation for an 8-vessel LNG carrier fleet**  
Version 7 · March 2026 · Python 3.11

---

## What this is

A Python simulation framework for optimising a six-vessel LNG fleet (with optional two-vessel acquisition extension) across three supply basins (US, West Africa, Pacific) and two destination markets (Europe via TTF, Asia via JKM).

Built in layers:

1. **Deterministic route matrix** — base-case economics for every origin-destination pair
2. **FOB vs DES contract logic** — clean separation of pricing, routing flexibility, and optionality
3. **Initial hedging layer** — locked vs open P&L by vessel category
4. **Stochastic simulation** — correlated GBM for Brent/HH/TTF/JKM + mean-reverting OU freight, with two-regime Markov switching
5. **Two-stage optimisation** — committed ships fixed, swing FOB ships re-assigned per scenario
6. **Real options valuation** — Margrabe exchange option per swing ship
7. **Acquisition analysis** — marginal portfolio value of adding C2 vessels W1 and W2
8. **Decision support outputs** — route explanation, contract dashboard, desk-ready summary

---

## Quickstart

```bash
git clone https://github.com/your-username/lng-fleet-sim.git
cd lng-fleet-sim
pip install -r requirements.txt
jupyter notebook notebooks/lng_fleet_optimiser.ipynb
```

---

## Results summary (v7, March 2026)

| Scenario | E[P&L] | Std Dev | CVaR |
|---|---|---|---|
| Sum of parts (ρ=1 naive) | ~$84M | ~$21M | ~$45M |
| Portfolio fixed routes | $88.34M | $16.82M | $59.14M |
| Portfolio two-stage | $94.75M | $17.70M | $63.80M |
| 8-ship (C1+C2) fixed | $115.90M | $20.10M | $81.20M |
| 8-ship two-stage | $126.40M | — | — |

**Two-stage uplift:** +$6.41M E[P&L], +$4.66M CVaR  
**Acquisition benefit:** +$7.86M CVaR, +$3.55M two-stage uplift from adding C2 vessels

---

## Fleet

| # | Vessel | m³ | Contract | Supply origin | Hedge | AIS position (Mar 2026) |
|---|--------|----|----------|---------------|-------|--------------------------|
| 1 | GasLog Italy | 174k | DES | US / HH | 85% | NE Atlantic → Plaquemine LA |
| 2 | Maran Gas Kalymnos | 174k | FOB swing | Nigeria / Brent | 45% | Alboran Sea → Asia |
| 3 | Maran Gas Kastelorizo | 174k | DES | US / HH | 80% | GoM → Destrehan LA |
| 4 | Maran Gas Achilles | 174k | DES | Australia / Brent | 90% | Arabian Sea → Dahej India |
| 5 | Maran Gas Delphi | 160k | FOB swing | Nigeria / Brent | 40% | S.China Sea → Gladstone AUS |
| 6 | Seapeak Galicia | 140k | FOB swing | US / HH | 50% | Moroccan EEZ → Europe |
| 7 | W1 (C2 target) | 156k | Spot TC | Atlantic / Mixed | — | Mid-Atlantic → Ravenna Italy |
| 8 | W2 (C2 target) | 156k | Spot TC | Atlantic / Mixed | — | 35°N 13°W → Australia |

---

## Key parameters

| Parameter | Value | Description |
|---|---|---|
| `BRENT` | 82.0 $/bbl | Spot Brent |
| `HH` | 2.40 $/MMBtu | Henry Hub |
| `TTF` | 11.50 $/MMBtu | European gas |
| `JKM` | 13.20 $/MMBtu | Japan Korea Marker |
| `KAPPA` | 3.0 | OU freight mean-reversion (half-life ≈ 84 days) |
| `F_SIGMA` | 0.35 | Annualised freight vol |
| `N_SCENARIOS` | 5,000 | Monte Carlo paths (v7; was 2,000 in v6) |
| `HORIZON_DAYS` | 90 | Planning horizon |
| `PI_NORMAL` | 0.85 | Stationary probability of normal regime |
| `P_TO_STRESS` | 0.03/month | Markov transition: normal → stress |
| `P_TO_NORMAL` | 0.20/month | Markov transition: stress → normal |

---

## Model architecture

### Layer 1 — Route economics

Per-cargo margin formula (v7):

```
Margin = DES_price − FOB_cost − Freight − Boil-off(DES price) − Port − Canal − E[Demurrage]
```

**v7 fixes vs v6:**
- Boil-off valued at DES price (not flat $12/MMBtu)
- Vessel-specific freight (Galicia 140k pays ~20% more per MMBtu than 174k vessels)
- Full round-trip cycle: laden + discharge + ballast + waiting
- Demurrage as expected value per port call
- Canal disruption as Bernoulli events (Suez 8%, Panama 3% per voyage)

### Layer 2 — Ship assignment

Each vessel is assigned to one of two buckets:

- **Committed (DES):** Destination fixed by contract. Ships 1, 3, 4. Hedged at 80–90%.
- **FOB swing:** Flexible destination. Ships 2, 5, 6, W1, W2. Hedged at 35–50% to preserve optionality.

### Layer 3 — Price simulation

| Variable | Model | Key params |
|---|---|---|
| Brent | GBM | σ = 28% annual |
| Henry Hub | GBM | σ = 45% annual |
| TTF | GBM | σ = 35% annual |
| JKM | GBM | σ = 30% annual |
| LNG Freight | Ornstein-Uhlenbeck | κ=3.0, θ=1.0, σ=0.35 |

Prices are simulated jointly via **Cholesky decomposition** of the regime-dependent correlation matrix.

### Layer 4 — Regime-switching correlation (v7)

Two-state Markov model with separate correlation matrices:

**Normal regime (π = 85%):**

|  | Brent | HH | TTF | JKM |
|---|---|---|---|---|
| Brent | 1.00 | 0.15 | 0.40 | 0.35 |
| HH | 0.15 | 1.00 | 0.25 | 0.20 |
| TTF | 0.40 | 0.25 | 1.00 | 0.75 |
| JKM | 0.35 | 0.20 | 0.75 | 1.00 |

**Stress regime (π = 15%):**

|  | Brent | HH | TTF | JKM |
|---|---|---|---|---|
| Brent | 1.00 | 0.05 | **0.65** | 0.55 |
| HH | 0.05 | 1.00 | 0.10 | 0.08 |
| TTF | **0.65** | 0.10 | 1.00 | **0.40** |
| JKM | 0.55 | 0.08 | **0.40** | 1.00 |

**Key insight:** TTF-JKM drops from 0.75 → 0.40 in stress. This amplifies Margrabe option values by ~+55% in stress scenarios, partially offsetting wider tail risk.

### Layer 5 — Two-stage optimisation

**Stage 1 (today):** Fix committed ships and hedge ratios. Place swing ships on standby.

**Stage 2 (day 30–45):** Each FOB swing ship observes realised prices and routes to whichever destination pays more:

```python
payoff(swing) = max(margin_EU, margin_Asia)  # per scenario
```

Switch triggers (current JKM-TTF = $1.70/MMBtu):
- Ship 2 (Kalymnos): switch to EU if JKM-TTF < $1.01
- Ships 5, 6 (Delphi, Galicia): switch to Asia if JKM-TTF > $0.96

### Layer 6 — Real options: Margrabe exchange option (v7)

Each FOB swing ship holds an option to exchange one destination margin for another:

```
V = Q × [M₁·N(d₁) − M₂·N(d₂)]
d₁ = [ln(M₁/M₂) + ½σ²T] / (σ√T)
σ² = σ₁² + σ₂² − 2ρ·σ₁·σ₂   (spread vol)
```

Total option value across 5 swing ships: **~$4.65M** (normal regime).

---

## Design decisions

**Boil-off uses destination price (v7)** — v6 used a fixed $12/MMBtu reference to avoid circular dependency. v7 resolves this by computing boil-off cost at the realised destination price per scenario, correctly penalising long-haul routes when destination prices are high.

**Freight modelled as Ornstein-Uhlenbeck** — LNG freight rates are mean-reverting. GBM with high vol produces unrealistic extreme values. OU with κ=3.0 anchors freight near equilibrium while allowing co-movement with TTF/JKM shocks.

**Hedge sweep is tiered by bucket** — committed ships scale 70–100%, swing ships 30–60%. A uniform scalar sweep ignores the bucket structure and produces misleading frontiers.

**FOB flex value via Margrabe (v7)** — v6 modelled destination optionality explicitly in the two-stage optimiser without pricing it separately. v7 prices it via Margrabe, revealing +$2.8M of option value previously unaccounted for.

**Regime-switching rather than static correlations (v7)** — a single static correlation matrix misses the TTF-JKM decoupling that occurs during supply crises. Two-regime Markov switching captures this dynamic and increases model realism at moderate computational cost.

---

## Structure

```
lng-fleet-sim/
├── notebooks/
│   └── lng_fleet_optimiser.ipynb
├── docs/
│   └── index.html              ← Interactive dashboard (GitHub Pages)
├── requirements.txt
├── .gitignore
└── README.md
```

### Interactive dashboard

The `docs/index.html` file is a standalone single-page application with 12 tabs:

1. **Overview** — Fleet composition, AIS positions, KPI cards
2. **Acquisition** — C2 marginal value analysis
3. **Portfolio Effect** — Diversification decomposition
4. **Correlations** — Three-scenario correlation sensitivity table
5. **Model Overview** — All 7 model layers documented inline
6. **Route Matrix** — Deterministic margin table for all routes
7. **Fleet Matrix** — Ship-by-ship assignment and option premiums
8. **Hedge Dashboard** — Locked vs open P&L by benchmark
9. **Real Options** — Margrabe valuations by swing ship
10. **Canal Risk** — Suez/Panama disruption overlay
11. **Regime Switch** — Two-regime model detail
12. **Scenarios / Two-Stage / Action List** — Operational outputs

Deploy: push `docs/index.html` to `main` branch and enable GitHub Pages on `main/docs`.

---

## Requirements

```
numpy >= 1.24
pandas >= 1.5
scipy >= 1.10
matplotlib >= 3.6
seaborn >= 0.12
```

Install:
```bash
pip install numpy pandas scipy matplotlib seaborn
```

No external data feeds required — all market parameters are hardcoded to March 2026 values. To update, modify the constants in Section 1 of the notebook.

---

## Known limitations

| Limitation | Status |
|---|---|
| Single cargo per ship per cycle | Open |
| Canal availability deterministic | Fixed in v7 (Bernoulli disruption events) |
| Freight hedge instruments not modelled | Open — flagged as priority gap |
| No multi-period rolling horizon | Open |
| Cross-fleet correlation ρ=0.18 (freight vs commodity) uncalibrated | Open — most sensitive parameter |
| Partial cargo loading not modelled (ballasting ships) | Open |
| Normal approximation for CVaR in correlation tab | Acceptable for sensitivity analysis only |

---

## Model evolution (v6 → v7)

| Fix | Description | Impact |
|---|---|---|
| FIX 1 | Boil-off at DES price (was flat $12) | Route-specific; most material on long hauls |
| FIX 2 | Vessel-specific freight | Galicia disadvantaged on long routes |
| FIX 3 | Margrabe real-options pricing | +$2.8M option value identified |
| FIX 4 | Canal disruption (Bernoulli overlay) | Wider CVaR tail |
| FIX 5 | Regime-switching correlations | +55% option value in stress |
| FIX 6 | Dynamic hedge triggers | Improved CVaR in tail scenarios |
| FIX 7 | Full round-trip cycle times | $/day ranking changes |
| FIX 8 | Demurrage as expected value | Reduces all margins ~$0.05–0.15/MMBtu |

E[P&L] roughly flat v6→v7 ($88.04M → $88.34M) but CVaR worsens ($63.64M → $59.14M) because v7 captures tail risks v6 ignored. Two-stage uplift increases from +$5.21M to +$6.41M because regime-switching creates more spread divergence scenarios.

---

*All numbers are model outputs based on March 2026 market data. Not investment advice.*
