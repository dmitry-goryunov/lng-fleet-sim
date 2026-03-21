# LNG Fleet Portfolio Optimization

Six-vessel LNG portfolio simulation — deterministic routing, FOB/DES contract logic, stochastic price & freight modelling, two-stage optimization.

---

## What this is

A Python simulation framework for optimizing a six-vessel LNG fleet across three supply basins (US, West Africa, Pacific) and two destination markets (Europe via TTF, Asia via JKM).

Built in layers:

1. **Deterministic route matrix** — base-case economics for every origin-destination pair
2. **FOB vs DES contract logic** — clean separation of pricing, routing flexibility, and optionality
3. **Initial hedging layer** — locked vs open P&L by vessel category
4. **Stochastic simulation** — correlated GBM for Brent/HH/TTF/JKM + mean-reverting OU freight
5. **Two-stage optimization** — committed ships fixed, swing FOB ships re-assigned per scenario
6. **Decision support outputs** — route explanation, contract dashboard, desk-ready summary

---

## Quickstart

```bash
git clone https://github.com/your-username/lng-fleet-sim.git
cd lng-fleet-sim
pip install -r requirements.txt
jupyter notebook notebooks/lng_fleet_optimization.ipynb
```

---

## Key Parameters

| Parameter | Value | Description |
|---|---|---|
| `BRENT` | 82.0 $/bbl | Spot Brent |
| `HH` | 2.40 $/MMBtu | Henry Hub |
| `TTF` | 11.50 $/MMBtu | European gas |
| `JKM` | 13.20 $/MMBtu | Japan Korea Marker |
| `BOILOFF_REF_PRICE` | 12.0 $/MMBtu | Fixed boil-off reference |
| `KAPPA` | 3.0 | OU freight mean-reversion (~85-day half-life) |
| `F_SIGMA` | 0.35 | Annualised freight vol |
| `N_SCENARIOS` | 2,000 | Monte Carlo paths |
| `HORIZON_DAYS` | 90 | Planning horizon |

---

## Design Decisions

**Boil-off uses a fixed reference price** — valuing boil-off at `dest_price` creates a circular dependency where the destination price appears on both sides of the margin equation. A fixed $12/MMBtu reference eliminates this bias.

**Freight modelled as Ornstein-Uhlenbeck** — LNG freight rates are mean-reverting. GBM with high vol produces unrealistic extreme values. OU with κ=3.0 anchors freight near equilibrium while allowing co-movement with TTF/JKM shocks.

**Hedge sweep is tiered by bucket** — committed ships scale 70–100%, swing ships 30–60%. A uniform scalar sweep ignores the bucket structure and produces misleading frontiers.

**FOB flex value = 0** — destination optionality for FOB ships is modelled explicitly in the two-stage optimizer, not embedded in route margin formulas.

---

## Structure

```
lng-fleet-sim/
├── notebooks/
│   └── lng_fleet_optimization.ipynb
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Known Limitations

- Single cargo per ship per cycle
- Canal availability treated as deterministic
- Freight hedge instruments not modelled (flagged as open gap)
- No multi-period rolling horizon
