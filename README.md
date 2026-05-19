# ⛵ Ironman 140.6: Grit vs. Wealth — A Causal Inference Study

> *Does money buy speed in endurance sports — or does it just buy entry to the start line?*

[![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)](https://python.org)
[![DoWhy](https://img.shields.io/badge/Causal%20Inference-DoWhy-orange)](https://github.com/py-why/dowhy)
[![Dataset](https://img.shields.io/badge/Dataset-Kaggle-20BEFF?logo=kaggle)](https://www.kaggle.com/datasets/miguswong/ironman-140-6-results-dataset-2002-2024)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## The Question

Ironman triathlon marketing claims the average participant earns **$247,000+** per year. Online, two theories compete:

- **Wealth Theory:** Money buys aerodynamic equipment, professional coaching, and training time autonomy — directly purchasing faster finish times.
- **Grit Theory:** Ironman self-selects for obsessive, high-achieving personalities. The income correlation is a *selection effect*, not a performance driver.

This study uses **causal inference** — not correlation — to separate them.

---

## Key Results

| Model | Estimate | Interpretation |
|---|---|---|
| **IPW ATE (primary)** | **−23.7 minutes** | High-capital athletes finish faster under full confounder control |
| **OLS with venue FE** | +3.1 minutes | Direction reverses when course conditions are fixed |
| **Mechanism** | **Bike split (61%)** | Capital advantage concentrates on the bike leg |
| **Grit correlation** | −0.38 with finish time | Strongest single predictor in the dataset |

### The IPW / OLS Direction Conflict — Honest Interpretation

The IPW model returns −23.7 min (wealth is faster) while OLS with venue fixed effects returns +3.1 min (wealth is slightly slower after controlling for course). This conflict is methodologically important:

- **IPW** matches athletes globally on biology, talent, and grit — but cannot control for which course they raced on. High-capital athletes may disproportionately race harder courses (Kona, Boulder).
- **OLS** controls for venue, eliminating course difficulty — but includes `Log_Career_Sequence` as a covariate, which partially mediates the wealth pathway (more races → more experience).

The honest conclusion: **within identical course conditions, the raw capital advantage shrinks significantly.** Grit (−0.38 correlation) is the more robust predictor of finish time across all model specifications.

### The Exchange Rate

Capital buys **23.7 minutes** via IPW.  
To earn that same advantage through Grit alone would require approximately **3 grit-score points** (OLS grit coefficient is large, making the exchange rate favorable to training).

### The Mechanism

The bike leg accounts for **~61%** of the split ATE — consistent with the "Bought Speed" hypothesis: carbon superbikes, disc wheels, and professional bike fits directly reduce bike split times.

---

## The Causal Match Architecture

When the IPW model calculates the ATE, it enforces **identical** values across:

| Dimension | Variable |
|---|---|
| Biology | Age, Age², Gender |
| Talent | Swim time (baseline physical ability) |
| Grit | Run/Bike pace ratio (late-race durability) |
| Experience | Rookie status, historical DNF count |
| Freshness | Years since last race (≤ 2 year window) |
| Conditions | Venue + Year fixed effects *(OLS only)* |

**The only variable left:** Did this athlete race 2+ times within 24 months?

---

## Dataset

**[Ironman 140.6 Results 2002–2024](https://www.kaggle.com/datasets/miguswong/ironman-140-6-results-dataset-2002-2024)** — miguswong on Kaggle

| Stat | Value |
|---|---|
| Raw records | 1,096,719 |
| Finisher records (modeled) | 752,558 |
| Unique athletes | 451,508 |
| Race years | 2003 – 2024 |
| Races | 587 |
| Series | 69 |

> **Data not included in this repo.** Download via KaggleHub (Cell 2 of the notebook handles this automatically).

---

## Treatment Variable: High_Capital_Lifestyle

An athlete is flagged **High_Capital_Lifestyle = 1** if:
- They completed their **2nd or later race in the same calendar year** *(Yearly_Sequence ≥ 2)*, **OR**
- They raced in the **prior calendar year** *(gap ≤ 1 year between consecutive entries)*

Both conditions are strictly **backward-looking** — no future race information is used.

**Financial logic:** Two full Ironmans within 24 months requires a minimum of $6k–$10k in race spend, plus the recovery infrastructure that money buys.

**Treatment prevalence:** ~27.8% of finisher records

---

## Methodology

```
Pipeline:
  1. Load results + races + series CSVs → join into single dataframe
  2. Compute sequences on FULL dataset (Finishers + DNFs) — then filter
     (DNF = financial event; dropping before sequence calculation erases history)
  3. Parse split times HH:MM:SS → float minutes
  4. Compute Grit Score: normalized inverse of Run_Pace / Bike_Pace ratio
  5. Engineer treatment (2-year race density window)
  6. Engineer confounders (Age², Swim_Min, DNF history, freshness)
  7. Filter: Years_Since_Last_Race ≤ 2 (active-fitness apples-to-apples)
  8. PRIMARY: IPW via DoWhy (propensity_score_weighting, all rows)
  9. ROBUSTNESS: PSM via DoWhy (propensity_score_matching)
  10. ROBUSTNESS: OLS with venue-year fixed effects (HC1 SEs)
  11. REFUTATION: Placebo treatment / Random common cause / Data subset
  12. MECHANISM: Split ATE by Swim / Bike / Run
  13. EXCHANGE RATE: |ATE| / |grit_coef| in grit-score points
  14. COST PER MINUTE: Capital spend / minutes saved
```

### Key Design Decisions

**Why IPW over PSM?** PSM discards unmatched observations — at 750k rows it wastes enormous statistical power. IPW reweights all observations, is more efficient, and is mathematically stable.

**Why HC1 over HC3 in OLS?** HC3 computes exact leverage for every observation. At 750k rows + venue dummies it causes CPU thrashing for minutes. HC1 is statistically sound at this sample size.

**Why exclude Career_Race_Count from IPW?** It is a mediator — wealth enables more races, more races improve performance. Controlling for it deletes the causal effect we are measuring (post-treatment bias).

**Why keep DNFs before sequence calculation?** A DNF is a financial event. The athlete paid the entry fee and shipped their bike. Dropping DNFs before cumcount() understates financial footprint and mislabels third-try finishers as rookies.

---

## Caveats

1. **Survivorship bias:** Only finishers are observed. Every athlete cleared a ~$3k entry barrier — the Control group is already pre-selected, not the general population.

2. **Attenuation bias (conservative lower bound):** Wealthy bucket-list athletes who race once every 3+ years are misclassified into Control, making Control artificially fast. Any wealth ATE found is a lower bound.

3. **No exact race dates:** Only `Race_Year` is available. The 2-year window approximates a rolling 24-month window.

4. **Intra-year sort:** `Race_ID` used as within-year chronological proxy (database key, not a timestamp). Intra-year rookie assignment is best-effort.

5. **No athlete home location:** Venue income mapping was explored and abandoned — it causes an ecological fallacy (classifying a visiting CEO as "low income" based on the host city's median).

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
│   └── ironman_grit_vs_wealth_v11.ipynb
│
└── assets/
    ├── causal_model.png
    ├── final_plot1_correlation.png
    └── final_plot2_eda.jpg
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
jupyter notebook notebooks/ironman_grit_vs_wealth_v11.ipynb
```

The notebook downloads the dataset automatically via KaggleHub (Cell 2). You will need a `kaggle.json` API key at `~/.kaggle/kaggle.json`.

---

## Stack

| Library | Role |
|---|---|
| `pandas` / `numpy` | Data manipulation |
| `dowhy` | Causal model, IPW, PSM, refutation tests |
| `statsmodels` | OLS with HC1 heteroskedasticity-robust SEs |
| `seaborn` / `matplotlib` | Visualization |
| `kagglehub` | Dataset download |

---

## Built by 雲

[@_yuncl](https://instagram.com/_yuncl) · [@yunvo.fit](https://instagram.com/yunvo.fit) · [yunvo.fit](https://yunvo.fit)

*Yunvo — protein built for people who race things.*
