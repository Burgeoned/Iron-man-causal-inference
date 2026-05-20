# ⛵ Ironman 140.6: Grit vs. Wealth — A Causal Inference Study

> *Does money buy speed in endurance sports — or does it just buy entry to the start line?*

[![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)](https://python.org)
[![DoWhy](https://img.shields.io/badge/Causal%20Inference-DoWhy-orange)](https://github.com/py-why/dowhy)
[![Dataset](https://img.shields.io/badge/Dataset-Kaggle-20BEFF?logo=kaggle)](https://www.kaggle.com/datasets/miguswong/ironman-140-6-results-dataset-2002-2024)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## The Question

Ironman marketing claims the average participant earns **$247,000+** per year. Online, two competing theories exist:

- **Wealth Theory:** Money buys aerodynamic equipment, coaching, and training time autonomy — directly purchasing faster finish times.
- **Grit Theory:** Ironman self-selects for obsessive, high-achieving personalities. The income correlation is a *selection effect*, not a performance driver.

This study uses **causal inference** — not correlation — to separate them.

---

## Key Results (v13 — Global, 2003–2024)

| Metric | Value |
|---|---|
| **Total finisher records** | 752,558 |
| **Unique athletes** | 451,508 |
| **Race years** | 2003–2024 |
| **Continents** | 5 (North America 44%, Europe 39%, Asia Pacific 12%, South America 3%, Africa 2%) |

### Causal Models

| Model | Estimate | Notes |
|---|---|---|
| **IPW ATE (primary, full data)** | **−23.7 min** | High-capital athletes finish faster globally |
| **PSM ATE (robustness, 50k)** | −36.3 min | Consistent direction |
| **OLS with venue FE (robustness, 50k)** | +1.4 min (p=0.44) | Not significant once course difficulty controlled |
| **Grit Score OLS coef** | **−8.578 min/point*** | Strongest predictor across all model specs |
| **Rookie penalty** | +23.2 min*** | First-race blowup variance |
| **Gender gap (OLS)** | −40.0 min*** | Men faster |
| **R-squared (OLS)** | 0.676 | Strong fit for sports performance data |

### The IPW / OLS Direction Conflict — The Honest Conclusion

The IPW model returns −23.7 min (capital is faster) while OLS with venue fixed effects returns +1.4 min (not significant). This is the most important finding in the study:

- **IPW** matches athletes on biology, talent, and grit globally — but cannot control for course difficulty.
- **OLS** forces comparison within identical course-year combinations — once you hold course constant, the raw capital advantage disappears entirely.

**High-capital athletes disproportionately race harder, more prestigious courses** (Kona, Boulder, Frankfurt). The IPW advantage is partially course selection, not raw speed. Within identical course conditions, **Grit is the dominant performance predictor** across every model specification.

### Refutation Tests (IPW)

| Test | Result | Interpretation |
|---|---|---|
| Placebo treatment | New effect: +11.2 min (p≈0) | Original effect is real — random treatment kills it |
| Random common cause | New effect: −23.73 min (p=0.16) | Stable — no lurking confounder detected |
| Data subset (80%) | New effect: −23.72 min (p=0.48) | Stable — not driven by outliers |

### Mechanism of Action (Split ATE)

| Discipline | ATE | Share |
|---|---|---|
| **Bike** | **−13.4 min** | **61%** |
| **Run** | −8.5 min | 39% |
| Swim | excluded (confounder in DAG) | — |

**61% of the capital advantage concentrates on the bike leg** — consistent with the "Bought Speed" hypothesis (carbon frames, disc wheels, aero fits). The remaining 39% on the run suggests recovery infrastructure (physio, coaching, altitude camps) also plays a role.

### Sensitivity Analyses

| Analysis | ATE | Interpretation |
|---|---|---|
| **3+ races in 24 months** | −28.9 min | Dose-response confirmed — capital effect scales with spending |
| **Veteran cohort (5+ career races)** | −6.1 min | Advantage persists even after pacing experience is maxed out — it's the equipment |
| Veteran bike split | −3.82 min (65%) | Same mechanism in veterans — bike dominates |

---

## The Causal Match Architecture

When the IPW model calculates the ATE, it enforces **identical** values across:

| Dimension | Variable |
|---|---|
| Biology | Age, Age², Gender |
| Talent | Swim time (baseline physical ability) |
| Grit | Run/Bike pace ratio (late-race durability) |
| Experience | Rookie status (DNF-corrected), Log historical DNF count |
| Freshness | Years since last race (≤ 2 year active-fitness window) |
| Conditions | Venue + Year fixed effects *(OLS robustness check only)* |

**The only variable left:** Did this athlete race 2+ times within 24 months?

---

## Exploratory Findings

| Finding | Value |
|---|---|
| First-attempt finish rate (global) | **84.0%** |
| First-attempt finish rate by continent | South America 86.4%, Asia Pacific 86.3%, Europe 85.0%, Africa 82.5%, North America 82.3% |
| Athletes who finish on their first try | **95.9%** of eventual finishers |
| Athletes who finish within 3 tries | 99.9% |
| Mean attempts to first finish | 1.05 |
| Mean race-day cost to first finish | ~$2,305 |
| Male peak performance age | 32 years |
| Female peak performance age | 27 years |
| Gender gap (2003) | +45.3 min (F−M) |
| Gender gap (2024) | +45.1 min (F−M) |
| Gender gap trend | Essentially flat over 21 years |
| DNF rate at career race 1 | ~16% |
| DNF rate at career race 5+ | ~23% (experienced athletes push harder and race harder courses) |

---

## The Exchange Rate

Capital buys **23.7 minutes** (IPW ATE).  
The OLS Grit coefficient is **−8.578 min per grit point**.  
Exchange rate: **2.8 grit points = 1 capital event**.

Speed bought by capital costs **~$331 per minute** (~$19,845 per hour of finish time reduction).

---

## Treatment Variable: High_Capital_Lifestyle

**High_Capital_Lifestyle = 1 if:**
- Athlete completed their **2nd or later race in the same calendar year** (Yearly_Sequence ≥ 2), **OR**
- Athlete raced in the **prior calendar year** (gap ≤ 1 year between consecutive entries)

Both conditions are strictly **backward-looking** — no future race information is used.

**Financial logic:** Two full Ironmans within 24 months = minimum $6k–$10k combined race spend.  
**Treatment prevalence:** 33.6% of finisher records (after active-fitness filter)

---

## Methodology

```
Pipeline:
  1. Load results + races + series CSVs → join (1,096,719 total entries)
  2. Compute sequences on FULL dataset (Finishers + DNFs + DNS)
     — DNF/DNS = financial event; filtering before sequence calculation erases history
  3. Filter to finishers → apply time range filter (8h–17h)
  4. Parse split times HH:MM:SS → float minutes
  5. Compute Grit Score: normalized inverse of Run_Pace / Bike_Pace ratio
  6. Engineer treatment (2-year race density window, strictly backward-looking)
  7. Engineer confounders (Age², Swim_Min, Log_DNF_Count, Years_Since_Last_Race)
  8. Active fitness filter: Years_Since_Last_Race ≤ 2 (dropped 6.0%)
  9. PRIMARY: IPW via DoWhy (propensity_score_weighting, full 752k dataset)
  10. ROBUSTNESS: PSM via DoWhy (50k stratified sample)
  11. ROBUSTNESS: OLS with venue-year fixed effects (50k sample, HC1 SEs)
  12. REFUTATION: Placebo treatment / Random common cause / Data subset
  13. MECHANISM: Split ATE by Bike + Run (Swim excluded — confounder in DAG)
  14. SENSITIVITY A: 3+ races in 24 months (dose-response test)
  15. SENSITIVITY B: Veteran-only cohort (Career_Sequence ≥ 5)
  16. EDA: First-attempt finish rate, trials to finish, cost model,
           DNF survival curve, age of peak performance, gender gap over time
```

### Key Design Decisions

**Why IPW over PSM as primary?** PSM discards unmatched observations — at 750k rows it wastes enormous statistical power. IPW reweights all observations and is more stable. PSM retained as robustness check on 50k sample.

**Why HC1 over HC3 in OLS?** HC3 computes exact leverage for every observation. At 750k rows + 530 venue dummies it causes CPU thrashing. HC1 is statistically sound at this sample size.

**Why exclude Career_Race_Count from IPW confounders?** It is a mediator — wealth enables more races, more races improve performance. Controlling for it deletes the causal effect we are measuring (post-treatment bias).

**Why keep DNFs before sequence calculation?** A DNF is a financial event. An athlete who paid, flew, shipped their bike, and DNF'd still spent the money. Dropping them before cumcount() understates financial footprint and mislabels third-try finishers as rookies.

**Why exclude Swim_Min from split ATE?** Swim_Min appears as a confounder in the DAG (causes both treatment and outcome). Using it as an outcome creates a directed cycle — methodologically invalid.

**Why global scope?** The treatment variable is athlete behavior, not venue geography. Restricting to North America misclassifies athletes who raced Frankfurt in June and Arizona in November — their Frankfurt race would be invisible, placing them incorrectly in Control.

---

## Caveats

1. **Survivorship bias:** Only finishers are observed. Every athlete cleared a ~$3k entry barrier — the Control group is already pre-selected.

2. **Attenuation bias (conservative lower bound):** Wealthy athletes who race infrequently are misclassified as Control, making Control artificially fast. Any positive wealth ATE found is a lower bound.

3. **No exact race dates:** Only `Race_Year` is available. The 2-year window approximates a rolling 24-month window.

4. **Intra-year sort:** `Race_ID` used as within-year chronological proxy (database key, not a timestamp). Intra-year rookie assignment is best-effort.

5. **No athlete home location:** Venue income mapping was explored and abandoned — it causes an ecological fallacy (classifying a visiting CEO as "low income" based on the host city's median).

6. **Career sequence starts at 2002:** Pre-2002 race history unavailable — some veteran athletes may be misclassified as rookies.

---

## Repository Structure

```
ironman-causal-inference/
│
├── README.md
├── .gitignore
├── requirements.txt
│
├── notebooks/
│   └── ironman_grit_vs_wealth_v13.ipynb
│
└── assets/
    ├── v13_plot1_correlation.png
    ├── v13_plot2_eda.png
    ├── v13_plot3_final.png
    ├── v13_eda_first_attempt.png
    ├── v13_eda_attempts.png
    ├── v13_eda_cost.png
    ├── v13_eda_survival.png
    ├── v13_eda_age_peak.png
    └── v13_eda_gender_gap.png
```

---

## Quickstart

```bash
git clone https://github.com/yunvo/ironman-causal-inference
cd ironman-causal-inference
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # Mac/Linux
pip install -r requirements.txt
jupyter notebook notebooks/ironman_grit_vs_wealth_v13.ipynb
```

The notebook downloads the dataset automatically via KaggleHub (Cell 2). You will need a Kaggle API key at `~/.kaggle/kaggle.json`.

---

## Stack

| Library | Version | Role |
|---|---|---|
| `pandas` | ≥2.0.0 | Data manipulation |
| `numpy` | ≥2.0.0 | Numerical operations |
| `dowhy` | ≥0.11.0 | Causal model, IPW, PSM, refutation tests |
| `statsmodels` | ≥0.14.0 | OLS with HC1 heteroskedasticity-robust SEs |
| `seaborn` | ≥0.13.0 | Statistical visualization |
| `matplotlib` | ≥3.8.0 | Plotting |
| `scikit-learn` | ≥1.3.0 | Propensity score estimation (DoWhy dependency) |
| `kagglehub` | ≥0.1.0 | Dataset download |

---

## Built by 雲

[@_yuncl](https://instagram.com/_yuncl) · [@yunvo.fit](https://instagram.com/yunvo.fit) · [yunvo.fit](https://yunvo.fit)

*Yunvo — protein built for people who race things.*
