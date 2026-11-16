# Project Plan: Predicting Run vs. Pass from Pre-Snap NFL Data

**Course:** CS 363M Machine Learning (UT Austin, Fall 2026)
**Notebook & process writeup due:** Nov 19, 2026, midnight (slip days not allowed)
**Q&A:** Dec 1, 2026

> We record every decision and its reasoning in [DECISIONS.md](DECISIONS.md). This plan refers to those entries by ID, e.g. (D3). DECISIONS.md is internal and not submitted, so the notebook explains decisions inline and never cites IDs.

---

## 1. Goal

We join the nflverse 2022 play-by-play (pbp) data with the 2022 participation data on game ID + play ID. The task is to predict whether a **scrimmage play is a pass (dropback) or a run**, using only information known **before the snap**.

- Label: nflverse `pass` flag. Sacks and scrambles count as passes (D1).
- Scope: scrimmage plays only. Special teams, kneels, spikes and 2-pt tries are excluded (D2). *This narrows the proposal's "pass vs. anything else" wording.*

## 2. How we're graded (and where each part is covered)

| Rubric item | Points | Where it's covered |
|---|---|---|
| Data Cleaning | 10 | Notebook §1–2: join diagnostics, filters, missing values, leakage audit |
| Data Exploration | 10 | Notebook §3: EDA plots, each with a takeaway |
| Feature Engineering | 20 | Notebook §4: personnel parsing, situation/clock/weather/sequence/tendency features, ablation |
| Modeling | 40 | Notebook §5–6: baselines, 6+ models, grouped CV, tuning, ensembles |
| Code & Notebook Quality | 10 | Narrated markdown, clean pipeline, runs top to bottom |
| Outcome/Results | 10 | Notebook §7: comparison, xpass benchmark, importance, error analysis |

Accuracy itself isn't graded. **Methodology is.**

## 3. Repository layout

```
ML-Project/
├── PLAN.md                    ← this file
├── DECISIONS.md               ← every decision + why (source material for the writeup)
├── nfl_pass_prediction.ipynb  ← the graded notebook
├── requirements.txt
├── .gitignore                 ← ignores data/, checkpoints
└── data/                      ← downloaded parquet files (not committed)
```

## 4. Notebook outline

### §0 Introduction
Problem statement, why it matters (defensive play-calling), dataset description, table of contents.

### §1 Data acquisition and merge
- Download `play_by_play_2022.parquet` and `pbp_participation_2022.parquet` from the nflverse-data GitHub releases. Cache them in `data/` (D3).
- Inner join `pbp.game_id = part.nflverse_game_id` AND `pbp.play_id = part.play_id` (D4).
- Join diagnostics: row counts, matched/unmatched rows from each side, which play types failed to match. Assert no duplicate keys.

### §2 Cleaning
- Scrimmage-play filters, with a before/after count for each step (D2). Aborted snaps dropped (D19).
- Target `is_pass` and the class balance (D1).
- **Leakage audit:** a whitelist of pre-snap columns, every excluded column tagged with a reason, and a univariate-AUC scan with an assert (D7, D8, D22).
- Missing-value table, handled by cause: drop NGS gaps, mark indoor games, parse outdoor weather from text (D20).
- Category cardinality: rare personnel groupings are handled by parsing into counts and by `OneHotEncoder(min_frequency)` in the Pipeline (D21).
- ✅ Done Nov 16: 36,539 plays × 38 cols, 61.0% pass.

### §3 Exploratory data analysis
Each plot gets a 1–2 line takeaway.
- Class balance overall and by team.
- Pass rate heatmap: down × distance.
- Pass rate vs. score differential by quarter; pass rate vs. time remaining.
- Pass rate by personnel, by formation, by defenders in box; shotgun × pass crosstab.
- Weather (rain, wind) vs. pass rate.
- Correlation matrix and collinearity notes.

### §4 Feature engineering (D9)
Every feature must be known before the snap. Each group gets a quick check (pass-rate plot or univariate AUC).
- **Personnel:** parse offense into `n_rb/n_te/n_wr` and defense into `n_dl/n_lb/n_db`. Add `box_minus_blockers`, `light_box`/`heavy_box`.
- **Formation:** one-hot formation, `shotgun`, `no_huddle`, `shotgun × n_wr`.
- **Situation:** distance buckets, `log_ydstogo`, `third_or_fourth_and_long`, `red_zone`, `goal_to_go`, `backed_up`.
- **Game state/clock:** `two_minute_drill`, `trailing_late`, `score_diff × time_remaining`, timeouts, `wp`, `spread_line`.
- **Weather:** `is_rain`/`is_snow` from the weather string (the proposal idea), `wind`, `temp`, `is_indoor`, roof/surface.
- **Sequence/tendency:** previous play in the drive (pass? yards?), play number within the drive, team season-to-date pass rate *before* the current game (D10).
- **Pipeline:** `ColumnTransformer` (one-hot + scaling) inside an sklearn `Pipeline`, so nothing leaks across CV folds (D11).
- **Ablation:** pbp-only → + participation → + engineered features (D12).

### §5 Train/test methodology
- Temporal split: train on weeks 1–14, test on weeks 15–18 + playoffs (D13).
- `GroupKFold` by `game_id` (5 folds) inside the training set (D14).
- Metrics: ROC-AUC is primary. Also accuracy, log-loss, F1, precision/recall, confusion matrix (D15).

### §6 Modeling (D16)
1. Baselines: always-pass; shotgun → pass.
2. Logistic regression (tune C, L1/L2).
3. Decision tree (tune depth, min leaf, criterion). Plot a shallow tree.
4. KNN (tune k). Optional.
5. Random forest.
6. Gradient boosting (XGBoost / HistGradientBoosting) with early stopping.
7. MLP. Optional.
8. Ensembles: soft voting and stacking.
- Tuning: RandomizedSearchCV/GridSearchCV with GroupKFold, scored on AUC. Validation curves and a learning curve (D17).
- Class weights and decision-threshold check (D18).

### §7 Results and analysis
- Comparison table (CV AUC ± std, test metrics) and bar chart; overlaid ROC curves; calibration curve.
- **Benchmark vs. nflverse `xpass`** on the same test plays (D8).
- Permutation importance, SHAP (GBM), LR coefficients.
- Error analysis by down, quarter, distance, formation, team; review the most confident misses.
- Ablation results; conclusions and limitations.

## 5. Schedule

| Day | Date | Work |
|---|---|---|
| 1 | Nov 16–17 | Setup, download, merge, cleaning, leakage audit, EDA |
| 2 | Nov 17–18 | Feature engineering, split/CV, baselines, LR/tree/RF/GBM + tuning |
| 3 | Nov 18–19 | Ensembles, ablation, xpass benchmark, error analysis, polish. **Team writes the PDF writeup** |

## 6. Process writeup (team-written, no AI)

The course forbids AI tools for the writeup. [DECISIONS.md](DECISIONS.md) holds the raw facts and reasoning to draw from, but the writeup itself has to be in the team's own words. Required sections: data exploration, cleaning, feature engineering, algorithm selection, hyperparameter tuning, results, and **work split** (at the end).

## 7. Verification checklist

- [ ] `pip install -r requirements.txt` → Restart & Run All in a fresh kernel, with no errors
- [ ] Asserts: join counts, no duplicate (game_id, play_id), no leakage columns in X, train/test weeks disjoint, binary label with ~60% pass
- [ ] Leakage smell test: >90% accuracy means stop and investigate (expect ~70–80% accuracy, AUC ~0.80–0.85)
- [ ] Every model beats both baselines; best model compared with `xpass`
- [ ] Every decision in the notebook is explained inline (what, why, alternatives, plan changes), with no D-ID references
