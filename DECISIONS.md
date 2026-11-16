# Decision Log

Every methodological choice in the project, with **what** we decided, **why**, and **what we considered instead**. This file is internal (not submitted). The notebook never cites these IDs; it explains each decision in its own words where the decision is made.

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
- **Result (Nov 16):** 49,434 merged rows → 36,870 with a pass/rush flag → drop 128 flagged 2-pt tries → drop 13 more 2-pt tries that were wiped out by a penalty (these have no `down` and no `two_point_attempt` flag; a notebook assert caught them) → 36,729. The 1,423 penalty-nullified plays with a known call are kept (D1). The 959 QB scrambles (play_type "run") are labeled pass (D1). Final pass rate after all cleaning: **61.0%** (REG 60.9%, POST 63.9%).

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
- **Result (Nov 16):** pbp has 49,434 rows and participation has 50,150, across the same 284 games (47,157 REG + 2,277 POST plays). Neither table has duplicate keys. **All 49,434 pbp plays matched.** The 716 participation-only rows are empty placeholders (no formation, no personnel, `n_offense == 0`), so the inner join loses nothing. Implementation details: pbp's `play_id` is a float and gets cast to int64; participation's duplicate `old_game_id` is dropped; the merge uses `validate="one_to_one"`. The merged frame is 49,434 × 389.

## Cleaning

### D5: Missing values
- **Status:** **Superseded by D20** (Nov 16). The data showed that a missing `temp`/`wind` does *not* always mean indoors.
- **Decision (planned):**
  - `defenders_in_box`, `offense_formation`, `offense_personnel`: drop rows if under ~2% are missing. Otherwise impute (median / "UNKNOWN" category).
  - `temp`, `wind`: these are NaN in domes and closed roofs, so fill with neutral values (e.g. 70°F, 0 mph) and add an `is_indoor` flag.
- **Why:** For `temp`/`wind`, a missing value means "no weather," not "unknown," so a flag plus a neutral fill keeps that meaning. For a tiny share of rows, dropping is simpler and cleaner than guessing personnel.
- **Alternatives:** model-based imputation (overkill here); dropping all rows with any NaN (would wipe out all dome games).

### D6: Merge rare categories
- **Status:** **Superseded by D21** (Nov 16). The merge moves inside the Pipeline, and personnel gets parsed into counts.
- **Decision:** Formation and personnel groupings that appear in fewer than ~0.5% of plays get grouped into "OTHER".
- **Why:** Very rare one-hot columns add noise and invite overfitting, especially for trees and KNN.

### D19: Drop aborted snaps
- **Status:** Decided (Nov 16)
- **Decision:** Drop the 113 plays with `aborted_play == 1` (fumbled or botched snaps).
- **Why:** nflverse labels 112 of the 113 as `rush` whatever the offense called, because the play never developed. Those labels reflect the botched snap rather than the call, so they are label noise. They're 0.3% of the data, so dropping them costs nothing meaningful.
- **Alternatives:** keep them as runs (adds noise); relabel from the description text (the called play usually can't be recovered).

### D20: Missing values, by cause (supersedes D5)
- **Status:** Decided (Nov 16)
- **Finding:** `temp`/`wind` are missing for 62% of plays. That covers all dome/closed games, but also all 1,313 open-roof plays and 10,438 *outdoor* plays from 81 games. For every one of those, the free-text `weather` column still holds the values ("Temp: 73° F, … Wind: W 3 mph").
- **Decision:**
  - Formation / personnel / `defenders_in_box` missing (77 plays, 0.21%, NGS tracking gaps): **drop rows**.
  - Dome/closed roof: `is_indoor = 1`, temp = 70°F, wind = 0 mph.
  - Outdoor/open roof with missing numbers: **parse temp and wind from the `weather` string**. This recovered 11,721 plays, with zero missing afterwards. Parsed ranges look realistic: 6–92°F, 0–27 mph.
  - `surface` empty string → "unknown"; missing `weather` text → "" (used for the rain/snow flags in §4).
  - A median imputer still sits inside the Pipeline as a safeguard (D11), fitted on training folds only.
- **Why:** For formation/personnel, imputing would invent exactly the data we merged to get. For weather, the true values exist in the text, so recovering them beats imputing a constant. The old plan would have marked 11,721 outdoor plays as "indoor 70°F, no wind," which is wrong and would have corrupted the weather analysis.
- **Alternatives:** the D5 plan (treat all missing as indoor); median imputation of temp/wind; dropping weather features.

### D21: Rare categories handled inside the Pipeline; personnel parsed into counts (supersedes D6)
- **Status:** Decided (Nov 16)
- **Finding:** All 7 offensive formations are ≥ 0.5% of plays (the rarest, WILDCAT, is 0.71%), so no formation merging is needed. Personnel has 64 distinct strings on each side. The top 5 cover 94.8% (offense) / 82.8% (defense), and 56 / 51 groupings are each under 0.5%. There are also odd strings ("2 QB, …", "…,1 LB" on offense).
- **Decision:** (1) Parse personnel strings into position counts in §4. (2) Wherever a grouping is one-hot encoded, use `OneHotEncoder(min_frequency=0.005, handle_unknown="infrequent_if_exist")` inside the Pipeline.
- **Why:** Choosing the rare-category cut-off from the full dataset uses test-set frequencies, a small leak. Doing it in the encoder fits the cut-off on each training fold. Position counts generalize across rare groupings (an unusual "6 OL, 1 RB, 2 TE, 1 WR" still reads as "heavy").
- **Alternatives:** the D6 plan (a manual pre-split merge to "OTHER").

### D22: Leakage control by whitelist, verified by a univariate-AUC scan
- **Status:** Decided (Nov 16)
- **Decision:** Keep an explicit whitelist of 26 pre-snap feature candidates, 8 ID columns, 1 auxiliary column (`yards_gained`, used only to build the *previous* play's yardage, D10) and 1 benchmark (`xpass`). Exclude the other 353 columns, each tagged with a reason (post-snap outcome 224; player identity 36; drive/series summary 18; label-derived 13; NGS post-snap 7; not needed 55). As an empirical safety net, compute every numeric column's univariate AUC against the label, scored by value *and* by missingness, and assert that no whitelisted feature exceeds 0.90.
- **Result:** The near-perfect columns are all excluded: `pass_oe`, `qb_dropback`, `pass_attempt`, `number_of_pass_rushers`, `air_yards`, jersey numbers. Several leak *only through missingness* (e.g. `passer_jersey_number` is non-null only on passes). The strongest whitelisted features are `defenders_in_box` (0.71), `shotgun` (0.69) and `down` (0.62). `xpass` = 0.79.
- **Why:** With about 390 columns, a blacklist fails silently when one is forgotten. A whitelist fails safe. The AUC scan catches leaks nobody thought of, including missingness leaks that a name-based review would miss.
- **Also decided here:** `wp` is kept (pre-snap win probability from game state; it doesn't model the play call). `n_offense`/`n_defense` are excluded (tracking counts, almost always 11, no football meaning). `drive` is excluded in favor of nflverse's corrected `fixed_drive`.
- **Alternatives:** a blacklist of known leaky columns (the D7 wording); manual review only.

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
