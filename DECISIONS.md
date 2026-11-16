# Decision Log

Every methodological choice in the project, with **what** we decided, **why**, and **what we considered instead**. The notebook cites these by ID (e.g. "see D7").

**How to use this file**
- Add an entry whenever you make a choice someone could ask "why?" about. Expect those questions at the Dec 1 Q&A.
- When results change a decision, don't delete the old entry. Mark it **Superseded by Dx** and add the new one. The history of changes is itself evidence of methodology.
- Use this as *source material only*. AI tools aren't allowed for the process writeup, so the team writes it in its own words.

Status values: **Decided** · **Tentative** (revisit once we see data/results) · **Superseded**

---

## Problem definition

### D1: Label: `is_pass` = nflverse `pass` flag (dropbacks)
- **Status:** Decided (team, Nov 16)
- **Decision:** A play is a pass if nflverse's `pass` column = 1. Sacks and QB scrambles therefore count as passes. A penalty-nullified play (`play_type == "no_play"`) is kept when nflverse's `pass`/`rush` flag shows which play was called.
- **Why:** We are predicting the *play call*, not the outcome. A sack or scramble began as a called pass, so labeling it "run" would add label noise the model can't learn from pre-snap data.
- **Alternatives:** (a) only actual throws (`play_type == "pass"`) count as pass; rejected because sacks/scrambles become mislabeled. (b) drop sacks, scrambles and no_plays; rejected because it discards valid called plays and shrinks the data.

### D2: Scope: scrimmage plays only (run vs. pass)
- **Status:** Decided (team, Nov 16). **Changes the proposal**, which said "pass vs. anything else."
- **Decision:** Keep plays where `pass == 1` or `rush == 1`. Drop special-teams plays (punts, FGs, kickoffs, XPs), 2-pt tries, QB kneels, QB spikes, and non-plays (timeouts, end of quarter, null `down`).
- **Why:** Special-teams plays are almost perfectly predictable from down and formation (4th down plus a punt formation means a punt). Including them inflates accuracy without showing that the model learned anything. Kneels and spikes are clock management, not strategic play calls. The interesting and realistic question is run vs. pass.
- **Alternatives:** keep everything (the proposal's version); keep it and add a run-vs-pass sub-analysis.

## Data acquisition and merging

### D3: Use the parquet files from nflverse-data releases, cached locally
- **Status:** Decided
- **Decision:** Download `play_by_play_2022.parquet` and `pbp_participation_2022.parquet` once into a gitignored `data/` folder.
- **Why:** Parquet keeps the dtypes and loads much faster than CSV for about 370 columns. Caching keeps re-runs fast and reproducible. Leaving data out of git keeps the repo small.
- **Alternatives:** the CSV versions; the `nflreadpy` package (another dependency to install).

### D4: Inner join on (game_id, play_id)
- **Status:** Decided
- **Decision:** `pbp.game_id = participation.nflverse_game_id` and `pbp.play_id = participation.play_id`, using an inner join.
- **Why:** Both tables come from nflverse and share play IDs (stated in the proposal). An inner join keeps only plays that have both situational and personnel/formation data, and our key features need both. Unmatched rows get reported, not hidden.
- **Alternatives:** a left join from pbp with imputed participation fields; rejected because it would invent formation/personnel data for plays where we have none.

## Cleaning

### D5: Missing values
- **Status:** Tentative (finalize after seeing the missingness table)
- **Decision (planned):**
  - `defenders_in_box`, `offense_formation`, `offense_personnel`: drop rows if under ~2% are missing. Otherwise impute (median / "UNKNOWN" category).
  - `temp`, `wind`: these are NaN in domes and closed roofs, so fill with neutral values (e.g. 70°F, 0 mph) and add an `is_indoor` flag.
- **Why:** For `temp`/`wind`, a missing value means "no weather," not "unknown," so a flag plus a neutral fill keeps that meaning. For a tiny share of rows, dropping is simpler and cleaner than guessing personnel.
- **Alternatives:** model-based imputation (overkill here); dropping all rows with any NaN (would wipe out all dome games).

### D6: Merge rare categories
- **Status:** Tentative (threshold set after EDA)
- **Decision:** Formation and personnel groupings that appear in fewer than ~0.5% of plays get grouped into "OTHER".
- **Why:** Very rare one-hot columns add noise and invite overfitting, especially for trees and KNN.

### D7: Leakage audit: use only pre-snap features
- **Status:** Decided
- **Decision:** Explicitly drop every column that is only known after the snap:
  - pbp outcome columns: `yards_gained`, `epa`, `wpa`, `air_yards`, `complete_pass`, `sack`, `qb_scramble`, `desc`, `play_type`, `rush`, etc.
  - Participation post-snap columns: `number_of_pass_rushers`, `time_to_throw`, `was_pressure`, `route`, `ngs_air_yards`, `defense_coverage_type`, `defense_man_zone_type`, player ID lists.
- **Why:** The model is meant to predict *before* the snap. Any post-snap column hands it the answer (e.g. `air_yards` only exists for passes). The notebook asserts that none of these columns reach the feature matrix.

### D8: Exclude `xpass`/`pass_oe` as features; use `xpass` as a benchmark
- **Status:** Decided
- **Decision:** `xpass` (nflverse's own pass-probability model) and `pass_oe` (pass minus xpass) are never used as inputs. We report `xpass`'s AUC and accuracy on our test set next to our models.
- **Why:** `pass_oe` contains the label. Using `xpass` as a feature would mean copying someone else's model. As a benchmark, though, it gives an honest outside reference point.

## Feature engineering

### D9: Feature groups (all known before the snap)
- **Status:** Decided (individual features may be dropped based on validation)
- **Decision:** Personnel counts parsed from strings; box-vs-blockers; formation one-hots; down/distance buckets and flags; clock/score flags (two-minute drill, trailing late); weather (rain/snow parsed, wind, temp, indoor); interactions (`shotgun × n_wr`, `score_diff × time_remaining`).
- **Why:** Personnel strings like "1 RB, 2 TE, 2 WR" are useless to a model as raw text but very informative as counts. Football knowledge says passing is more likely with more WRs, a lighter box, long distance, trailing late, and the two-minute drill, so explicit flags make those patterns easy to learn, especially for linear models. Rain/no-rain follows the proposal's plan to keep the weather columns few.

### D10: Sequence and team-tendency features, computed without leakage
- **Status:** Decided
- **Decision:** `prev_play_was_pass` and `prev_play_yards` (lag within the same game and drive), `drive_play_number`, and the team's season-to-date pass rate computed **only from previous games** (expanding mean shifted one game).
- **Why:** Coaches and defenses use these signals in real life, and all of them are known before the snap. A team pass rate that included the current game would leak the label, hence the shift.

### D11: sklearn Pipeline + ColumnTransformer
- **Status:** Decided
- **Decision:** One-hot encoding and scaling are fitted inside a `Pipeline`, so they are refit on each CV training fold.
- **Why:** Fitting a scaler or encoder on all the data before splitting leaks test-set information. Scaling is needed for LR, KNN and MLP. It's harmless for trees, so we use one pipeline everywhere for consistency.

### D12: Ablation study to justify the merge and the feature engineering
- **Status:** Decided
- **Decision:** Train the same model on three feature sets: (a) pbp situational only, (b) + participation (formation/personnel/box), (c) + engineered features.
- **Why:** This measures directly whether merging the second dataset and our feature engineering helped, instead of assuming it did.

## Evaluation methodology

### D13: Temporal train/test split
- **Status:** Decided
- **Decision:** Train on regular-season weeks 1–14; test on weeks 15–18 and the playoffs.
- **Why:** A real deployment predicts future games from past ones. A random split puts plays from the same game/drive in both train and test (a leakage risk) and conflicts with the season-to-date tendency features.
- **Alternatives:** a random stratified 80/20 split (simpler, but optimistic).

### D14: GroupKFold by game for cross-validation
- **Status:** Decided
- **Decision:** 5-fold `GroupKFold` on the training weeks, grouped by `game_id`.
- **Why:** Plays within a game are correlated (same teams, same game script). Grouping keeps CV scores honest.

### D15: ROC-AUC is the primary metric
- **Status:** Decided
- **Decision:** Tune and compare on ROC-AUC. Also report accuracy, log-loss, F1, precision/recall, and confusion matrices.
- **Why:** The classes are imbalanced (~60/40), so accuracy alone flatters "always pass." AUC measures ranking quality across all thresholds, and log-loss/calibration matter if the output is used as a probability (as `xpass` is).

## Modeling

### D16: Model lineup
- **Status:** Decided
- **Decision:** Baselines (always-pass, shotgun rule) → logistic regression → decision tree → KNN → random forest → gradient boosting → MLP → voting/stacking ensembles.
- **Why:** Baselines set the floor. LR and the decision tree come from the proposal and are interpretable. RF and GBM are the strongest tabular learners, so they test whether nonlinearity helps. KNN and MLP add different inductive biases. Ensembles test whether combining models beats the best single one. The course asks us to compare a variety of models.

### D17: Hyperparameter tuning approach
- **Status:** Decided
- **Decision:** GridSearchCV for small grids (LR, tree, KNN) and RandomizedSearchCV for large ones (RF, GBM, MLP), all with GroupKFold and AUC scoring. Show validation curves (tree depth, GBM learning rate) and a learning curve for the best model.
- **Why:** Randomized search covers big spaces efficiently given the 3-day deadline. Validation and learning curves show *why* the chosen values work (the bias–variance tradeoff), not just *that* they do.

### D18: Class imbalance handling
- **Status:** Tentative
- **Decision:** Compare `class_weight="balanced"` with no weighting, and tune the decision threshold against the default 0.5.
- **Why:** The imbalance is mild (~60/40), so we expect little change, but we'll check rather than assume.

---

## Changes from the original proposal
| Proposal said | We're doing | Entry |
|---|---|---|
| Predict pass vs. anything else (incl. punts, FGs) | Pass vs. run on scrimmage plays only | D2 |
| Decision tree, maybe logistic regression | Full model comparison incl. ensembles | D16 |
| Rain / not-rain weather feature | Kept, plus snow, wind, temp, indoor flag | D9 |
