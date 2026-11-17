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
- **Leakage audit:** a whitelist of pre-snap columns, every excluded column tagged with a reason, and a univariate-AUC scan with an assert (D7, D8).
- Missing-value table, handled by cause: drop NGS gaps, mark indoor games, parse outdoor weather from text (D5).
- Category cardinality: rare personnel groupings are handled by parsing into counts and by `OneHotEncoder(min_frequency)` in the Pipeline (D6).
- ✅ Done Nov 16: 36,539 plays, 61.0% pass. (Nov 17: whitelist roles updated, D7 → 39 columns.)

### §3 Exploratory data analysis ✅ Done Nov 17 (training weeks only, D29)
- Class balance; run/pass counts by down.
- Pass rate by down × yards-to-go bin (grouped bars).
- Formation (stacked counts + pass rate); pass rate by top-6 offensive personnel; shotgun crosstab.
- Pass rate by score differential × quarter; pass rate by minutes left in the half.
- Pass rate by defenders in box; by wind speed vs. indoor.
- Class-conditional histograms (ydstogo, yardline_100, score_differential, temp); correlation heatmap.
- *Not done (planned earlier):* class balance by team; rain-specific plot.

### §4 Feature engineering ✅ Done Nov 17 (D23–D26; D9/D10 refined)
- Personnel counts `n_rb/n_te/n_wr/n_ol`, `n_dl/n_lb/n_db`, `box_minus_blockers`.
- `ydstogo_bin`, `red_zone`, `third_fourth_short/long`, `two_minute`, `trailing_late`, `is_home`, offense-relative `posteam_spread`, `is_rain`.
- `prev_play`, `drive_play_number`, `team_prior_pass_rate` (chronology asserted, ordered by `order_sequence`).
- Nested feature sets A (situation) ⊂ B (+ alignment/personnel) ⊂ C (+ engineered); leakage asserts on the final list.
- *Not done:* `light_box/heavy_box`, `log_ydstogo`, `backed_up`, interaction terms (trees handle interactions); `is_snow` built but unusable (all zeros in training).

### §5 Train/test methodology ✅ Done Nov 17
- Temporal split weeks 1–14 / 15–18 + playoffs (26,799 / 9,740 plays), disjoint-games assert (D13).
- 4-fold forward-chaining CV by week (validation weeks 7–8, 9–10, 11–12, 13–14; D14: replaced the original `GroupKFold`, which leaked validation labels through `team_prior_pass_rate`); Pipeline = median imputer + `OneHotEncoder(min_frequency=0.005)`, no scaler (tree-only, D11).
- Metrics: ROC-AUC primary; accuracy, F1, precision/recall, run recall, macro F1, confusion matrices (D15). *Log-loss/calibration not done.*

### §6 Modeling ✅ Done Nov 17 (D16, D17)
1. Baselines: majority class (`DummyClassifier`), shotgun rule (D30).
2. Decision tree: depth validation curve + 60-config grid; depth-3 tree plotted.
3. Random forest: 12-config randomized search (incl. `class_weight`, D18).
4. AdaBoost: 18-config grid (base depth, n_estimators, learning rate).
5. Tuning summary + selection by CV AUC → AdaBoost (0.8267), effectively indistinguishable from the random forest (0.8265) (D31).
6. Feature-set ablation A/B/C, C − sequence, C + `wp`, CV only (D12, D26, D35).
- *Dropped:* logistic regression, KNN, gradient boosting, MLP, voting/stacking; learning curve; threshold tuning.

### §7 Results and analysis ✅ Done Nov 17
- Test comparison table (with CV AUC column and Δ accuracy vs. majority) and bar charts; confusion matrices; ROC curves.
- `xpass` benchmark on the same test plays (D8).
- Permutation importance (test set, selected model).
- Error analysis by down, yards-to-go, quarter, formation; down × distance error shares; most confident misses.
- Sensitivity: nullified plays (CV for the training choice; test only as a subgroup view of the final model) (D32, D35).
- *Not done:* calibration curve, SHAP, error analysis by team.

## 5. Schedule

| Day | Date | Work |
|---|---|---|
| 1 | Nov 16–17 | Setup, download, merge, cleaning, leakage audit, EDA |
| 2 | Nov 17 | ✅ EDA, feature engineering, split/CV, baselines, tree/RF/AdaBoost + tuning, ablation, xpass benchmark, error analysis (notebook complete) |
| 3 | Nov 18–19 | Team reviews the notebook and DECISIONS.md, prepares for Q&A. **Team writes the PDF writeup** |

## 6. Process writeup (team-written, no AI)

The course forbids AI tools for the writeup. [DECISIONS.md](DECISIONS.md) holds the raw facts and reasoning to draw from, but the writeup itself has to be in the team's own words. Required sections: data exploration, cleaning, feature engineering, algorithm selection, hyperparameter tuning, results, and **work split** (at the end).

## 7. Verification checklist

- [x] `pip install -r requirements.txt` → Restart & Run All in a fresh kernel, with no errors (Nov 17: fresh uv venv + nbclient, two consecutive runs gave identical outputs; Nov 18: re-run after the CV/test-set audit, D34–D35)
- [x] Asserts: join counts, no duplicate (game_id, play_id), no leakage columns in X, train/test weeks disjoint, binary label with ~60% pass, valid prediction probabilities
- [x] Leakage smell test: >90% accuracy means stop and investigate (expect ~70–80% accuracy, AUC ~0.80–0.85) → test accuracy 74–75%, AUC 0.81–0.83
- [x] Every model beats both baselines; best model compared with `xpass` (0.831 vs. 0.789 test AUC)
- [x] Every decision in the notebook is explained inline, briefly (the detailed rationale belongs in the team's writeup), with no D-ID references
