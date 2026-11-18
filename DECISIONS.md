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
- **Note (Nov 17):** a fresh download of the nflverse files gives 1,426 kept `no_play` rows (not 1,423) and slightly different `play_type` counts among dropped rows. nflverse re-publishes the files. Every row count in the cleaning pipeline is unchanged (36,539 final plays). See D33.

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
- **Update (Nov 17, D23):** IDs now include `order_sequence` (9 ID columns); the auxiliary column is now `is_nullified` instead of `yards_gained`. The scan's ranking of whitelisted features is unchanged; with the re-downloaded data `wp` scores 0.608 (was 0.608434, now 0.608055).
- **Update (Nov 18, final cleanup):** the univariate-AUC scan now uses **training weeks 1–14 only** (26,866 plays at that point in cleaning), so no test-week labels are inspected during the leakage audit. The whitelist and the 0.90 assert are unchanged, and no feature choice depended on the scan. New values: `defenders_in_box` 0.710, `shotgun` 0.692, `down` 0.619, `wp` 0.612; `xpass` 0.787. Excluded leaky columns still score 0.92–1.00.

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
- **Result (Nov 18), §7.3:** `xpass` has no missing values on the test plays. Test AUC 0.789, accuracy 70.8% (threshold 0.5) vs. the selected AdaBoost 0.828 / 74.8% (tree 0.810, RF 0.831). The gap plausibly comes from participation data (CV: situation-only set A 0.771); the Nov 17 version cited a *test* AUC for set A, which was removed under D35. We don't know which seasons nflverse trained `xpass` on, so this is a reference point, not a clean head-to-head.

## Feature engineering

### D9: Feature groups (all known before the snap)
- **Status:** Implemented (Nov 17); final list differs, see **D25**. The two interaction terms were not built (all our models are trees, which find interactions themselves); `is_snow` was built but not used.
- **Decision:** Personnel counts parsed from strings; box-vs-blockers; formation one-hots; down/distance buckets and flags; clock/score flags (two-minute drill, trailing late); weather (rain/snow parsed, wind, temp, indoor); interactions (`shotgun × n_wr`, `score_diff × time_remaining`).
- **Why:** Personnel strings like "1 RB, 2 TE, 2 WR" are useless to a model as raw text but very informative as counts. Football knowledge says passing is more likely with more WRs, a lighter box, long distance, trailing late, and the two-minute drill, so explicit flags make those patterns easy to learn, especially for linear models. Rain/no-rain follows the proposal's plan to keep the weather columns few.

### D10: Sequence and team-tendency features, computed without leakage
- **Status:** **Superseded by D24** (Nov 17). `prev_play_yards` dropped; ordering uses `order_sequence`, not `play_id`.
- **Decision:** `prev_play_was_pass` and `prev_play_yards` (lag within the same game and drive), `drive_play_number`, and the team's season-to-date pass rate computed **only from previous games** (expanding mean shifted one game).
- **Why:** Coaches and defenses use these signals in real life, and all of them are known before the snap. A team pass rate that included the current game would leak the label, hence the shift.

### D11: sklearn Pipeline + ColumnTransformer
- **Status:** Decided
- **Decision:** One-hot encoding and scaling are fitted inside a `Pipeline`, so they are refit on each CV training fold.
- **Why:** Fitting a scaler or encoder on all the data before splitting leaks test-set information. Scaling is needed for LR, KNN and MLP. It's harmless for trees, so we use one pipeline everywhere for consistency.
- **Implemented (Nov 17):** `make_pipeline()` in §5.1 = `ColumnTransformer`(median `SimpleImputer` on numeric; `OneHotEncoder(min_frequency=0.005, handle_unknown="infrequent_if_exist")` on categorical) + model. **No scaler**: the final lineup is tree-only (D27), and tree splits are invariant to monotone rescaling. 41 input columns → 72 model columns after encoding.

### D12: Ablation study to justify the merge and the feature engineering
- **Status:** Decided
- **Decision:** Train the same model on three feature sets: (a) pbp situational only, (b) + participation (formation/personnel/box), (c) + engineered features.
- **Why:** This measures directly whether merging the second dataset and our feature engineering helped, instead of assuming it did.
- **Implemented (Nov 17), §6.6 + §7.6:** selected model (tuned RF) on sets A (16 situation columns), B (+ shotgun, no_huddle, box, formation, raw personnel = 22), C (+ engineered = 41), plus two variants. Hyperparameters were tuned on C, which slightly favors C.
- **Result (Nov 18, forward-chaining CV, training weeks only):** CV AUC A 0.771 → B 0.821 → C 0.827; C without sequence/tendency 0.824; C + `wp` 0.827. Participation merge = +0.050 AUC (+4.9 pts CV accuracy); engineered features = +0.005 AUC, about one fold standard deviation (~0.004), so weak evidence. No test-set scores for feature sets (D35).
- **Superseded result (Nov 17, GroupKFold CV, see D34):** CV AUC A 0.777 → B 0.823 → C 0.827; test AUC A 0.785 → B 0.826 → C 0.831. Participation merge = +0.046 CV AUC (+4.5 pts CV accuracy); engineered features = +0.004 CV AUC, most of it from sequence/tendency (C without them: 0.824).

## Evaluation methodology

### D13: Temporal train/test split
- **Status:** Decided
- **Decision:** Train on regular-season weeks 1–14; test on weeks 15–18 and the playoffs.
- **Why:** A real deployment predicts future games from past ones. A random split puts plays from the same game/drive in both train and test (a leakage risk) and conflicts with the season-to-date tendency features.
- **Alternatives:** a random stratified 80/20 split (simpler, but optimistic).
- **Result (Nov 17):** nflverse numbers POST weeks 19–22, so the split is `week <= 14` vs. `week >= 15`. Train 26,799 plays / 208 games (61.4% pass); test 9,740 plays / 76 games (59.9% pass). Asserts: no shared games, both classes present, no duplicate plays.

### D14: GroupKFold by game for cross-validation
- **Status:** **Superseded by D34** (Nov 18). The caveat below turned out to be validation-label leakage into training features.
- **Decision:** 5-fold `GroupKFold` on the training weeks, grouped by `game_id`.
- **Why:** Plays within a game are correlated (same teams, same game script). Grouping keeps CV scores honest.
- **Implemented (Nov 17):** `GroupKFold(n_splits=5)`, folds of 5,297–5,404 plays. Caveat (observed in design, documented in §5): folds are not chronological, so a training-fold row's `team_prior_pass_rate` can include labels of an *earlier* validation-fold game. That information is genuinely available before the snap in real life, and a validation game never sees its own labels, so we accept it. The final test set is strictly later. CV and test AUCs ended up within ~0.005 of each other.

### D15: ROC-AUC is the primary metric
- **Status:** Decided
- **Decision:** Tune and compare on ROC-AUC. Also report accuracy, log-loss, F1, precision/recall, and confusion matrices.
- **Why:** The classes are imbalanced (~60/40), so accuracy alone flatters "always pass." AUC measures ranking quality across all thresholds, and log-loss/calibration matter if the output is used as a probability (as `xpass` is).
- **Implemented (Nov 17):** positive class = pass. CV tracks AUC, accuracy, F1; test reports accuracy, precision/recall/F1 for pass, recall for run, macro F1, ROC-AUC (from probabilities), confusion matrices. Log-loss and calibration curves were **not** done (time; not required).

## Modeling

### D16: Model lineup
- **Status:** **Superseded by D27** (Nov 17). Reduced to three tree-based models.
- **Decision:** Baselines (always-pass, shotgun rule) → logistic regression → decision tree → KNN → random forest → gradient boosting → MLP → voting/stacking ensembles.
- **Why:** Baselines set the floor. LR and the decision tree come from the proposal and are interpretable. RF and GBM are the strongest tabular learners, so they test whether nonlinearity helps. KNN and MLP add different inductive biases. Ensembles test whether combining models beats the best single one. The course asks us to compare a variety of models.

### D17: Hyperparameter tuning approach
- **Status:** **Superseded by D28** (Nov 17). Validation curve done for tree depth only; no learning curve.
- **Decision:** GridSearchCV for small grids (LR, tree, KNN) and RandomizedSearchCV for large ones (RF, GBM, MLP), all with GroupKFold and AUC scoring. Show validation curves (tree depth, GBM learning rate) and a learning curve for the best model.
- **Why:** Randomized search covers big spaces efficiently given the 3-day deadline. Validation and learning curves show *why* the chosen values work (the bias–variance tradeoff), not just *that* they do.

### D18: Class imbalance handling
- **Status:** Decided (Nov 17), partly done.
- **Decision:** Compare `class_weight="balanced"` with no weighting, and tune the decision threshold against the default 0.5.
- **Why:** The imbalance is mild (~60/40), so we expect little change, but we'll check rather than assume.
- **What was actually done:** `class_weight` ∈ {None, "balanced"} is part of the random-forest search (D28). Threshold tuning was **not** done; all label metrics use 0.5.
- **Result (Nov 18, forward-chaining CV):** the RF search selects `class_weight="balanced"` (CV AUC 0.8265 vs. 0.8260 for the best unweighted config). Balanced weighting lowers CV accuracy at threshold 0.5 (0.738 vs. 0.750) and on test gives run recall 0.81 / pass recall 0.69.
- **Superseded result (Nov 17, GroupKFold CV, see D34):** In the RF search the best unweighted and best balanced configurations tie on CV AUC (0.8273 vs. 0.8272). In an earlier run (before `is_snow` was removed, D25) the balanced one won by a similar margin. Weighting does not change ranking quality (AUC); at threshold 0.5 it shifts predictions toward "run" (earlier run: test run recall 0.82 / pass recall 0.69 vs. final 0.63 / 0.83, similar accuracy). The majority-class pass share (61%) is mild, so we kept AUC as the selection metric and did not tune the threshold.

## Implementation decisions (Nov 17)

### D23: Whitelist roles changed: `order_sequence` added as an ID, `yards_gained` replaced by `is_nullified`
- **Status:** Decided, implemented (§2.2–2.3)
- **Finding:** `play_id` is not chronological in 12 of 284 games (e.g. 2022_01_BAL_NYJ has play 3037 before 2844). nflverse's `order_sequence` is the true order, with no gaps or duplicates within a game. `yards_gained` is 0 on every penalty-nullified play in the cleaned data that has a value (1,337 rows), so "previous play's yards" would be misleading after a nullified play.
- **Decision:** Add `order_sequence` to the ID columns (sorting only). Drop `yards_gained` from the whitelist, since nothing uses it anymore. Add an auxiliary `is_nullified` flag (`play_type == "no_play"`), used only for the §7.6 sensitivity check and asserted to never be in X.
- **Alternatives:** sort by `play_id` (wrong order in 12 games); keep `prev_play_yards` (noisy after nullified plays).

### D24: Sequence and tendency features as built (supersedes D10)
- **Status:** Decided, implemented (§4.4)
- **Decision:**
  - `prev_play` ∈ {pass, run, none}: the label of the previous kept scrimmage play in the same `(game_id, fixed_drive)`, after sorting by `order_sequence`. "none" = first play of the drive. One-hot encoded.
  - `drive_play_number`: 1, 2, … within the drive.
  - `team_prior_pass_rate`: the offense's pass rate over all earlier **weeks** (cumulative sums minus the current week). NaN in week 1, filled by the Pipeline's median imputer (fit on training folds).
- **Checks in the notebook:** first play of every drive has `prev_play == "none"`; each team plays at most one game per week (so "earlier week" = "earlier game"); KC week-10 value equals a direct mean over weeks 1–9 (0.707); week 1 is all NaN.
- **Why previous labels are allowed:** the previous play's type is public before the next snap. A play never sees its own label.
- **Result (Nov 18, forward-chaining CV):** removing all three lowers CV AUC from 0.8267 to 0.8238 (within ~1 fold std). Permutation importance (AdaBoost, test): `prev_play` and `team_prior_pass_rate` small but non-zero; `drive_play_number` ≈ 0.
- **Superseded result (Nov 17, GroupKFold CV, see D34):** 0.8273 → 0.8240 (test 0.8312 → 0.8268).
- **Note:** under `GroupKFold` this feature leaked validation labels into training rows; see D34.

### D25: Final engineered features and exclusions (refines D9)
- **Status:** Decided, implemented (§4.1–4.5)
- **Built:** personnel counts `n_rb, n_te, n_wr, n_ol` (OL absent from string → 5) and `n_dl, n_lb, n_db`; `box_minus_blockers = defenders_in_box − (n_ol + n_te)`; `ydstogo_bin` (1–2 / 3–6 / 7–10 / 11+, one-hot); `red_zone` (≤ 20 yards); `third_fourth_short` (down ≥ 3, ≤ 2 yards); `third_fourth_long` (down ≥ 3, ≥ 7 yards); `two_minute` (≤ 120 s left in half); `trailing_late` (behind with ≤ 900 game seconds left); `is_home`; `posteam_spread` (D26); `is_rain` (outdoor/open games, "rain|shower" in the weather text); D24's sequence features.
- **Personnel parsing observed:** 35,249 offensive strings have 5 RB+TE+WR; 1,165 have 4 (e.g. a 2-QB or defender-on-offense look). On defense, 36,489 strings sum to 11, 36 to 10 and 14 to 12. Irregular positions are ignored, not dropped.
- **Excluded on purpose:** raw `spread_line` (wrong perspective, D26); `posteam_type`, `roof`, raw `weather` text (each replaced by a flag); `surface` (no clear football reason); team identities (would memorize teams; tendency is carried by `team_prior_pass_rate`); `wp` (D26).
- **`is_snow` built but not used:** the only snow game (2022_20_CIN_BUF) is a playoff game in the test weeks, so the flag is all zeros in training. An assert in §5 documents this.
- **Not built:** D9's interaction terms (trees model interactions directly).
- **Final set C:** 37 numeric + 4 categorical = 41 input columns → 72 after one-hot encoding.
- **Univariate AUC on training weeks (not used for selection):** `n_db` 0.649, `box_minus_blockers` 0.644, `n_wr` 0.637 … `is_rain` 0.504, `posteam_spread` 0.501.

### D26: Offense-relative spread and exclusion of `wp`
- **Status:** Decided, implemented (§4.2, §6.6)
- **Finding:** `spread_line` is the home team's expected margin. Over all plays, raw `spread_line` has correlation 0.00 with the offense's score differential; `posteam_spread = spread_line if home else −spread_line` has +0.31 (the flipped sign would give −0.31). An assert (> 0.2) guards the sign. (A first attempt checked the sign against `wp` on each game's first play and got only +0.29, so we switched to the score check.)
- **Decision on `wp`:** keep it out of the main set. It is a pre-snap nflverse model output, nearly redundant with `score_differential` (r = 0.92 in training weeks), and including it would make our model partly a wrapper around nflverse's model.
- **Result (Nov 18, forward-chaining CV):** C + `wp`: CV AUC 0.8270 vs. 0.8267 without. Negligible, so the exclusion costs nothing.
- **Superseded result (Nov 17, GroupKFold CV, see D34):** 0.8283 vs. 0.8273 (test 0.8319 vs. 0.8312).

### D27: Model lineup: three tree-based models (supersedes D16)
- **Status:** Decided, implemented (§6)
- **Decision:** `DummyClassifier(most_frequent)` and a shotgun rule as baselines; then `DecisionTreeClassifier`, `RandomForestClassifier` (bagging + random feature subsets), `AdaBoostClassifier` (boosting, SAMME, sklearn 1.9.1; default base learner = decision stump).
- **Why:** The project requirements ask for three algorithms aligned with the course (trees, bagging, boosting), not a broad sweep. One single model, one bagging ensemble and one boosting ensemble give a clean comparison, and all three are explainable in the Q&A. Logistic regression, KNN, MLP and stacking were dropped.
- **Alternatives:** the D16 lineup; `HistGradientBoostingClassifier` instead of AdaBoost (not needed, AdaBoost worked).

### D28: Hyperparameter search spaces and results (supersedes D17)
- **Status:** Decided, implemented (§6.2–6.5). Re-run Nov 18 with forward-chaining CV (D34); the results below are the Nov 18 ones, with Nov 17 GroupKFold values in brackets. All searches: 4-fold forward-chaining CV (originally 5-fold `GroupKFold` by game), `scoring={roc_auc, accuracy, f1}`, `refit="roc_auc"`, `return_train_score=True`, seed 363.
- **Decision tree:** `validation_curve` over `max_depth` ∈ {1,…,30} (best 6, CV AUC 0.791 [8, 0.793]), then `GridSearchCV` over depth {4,6,8,10,12} × min_samples_leaf {1,25,100} × min_samples_split {2,200} × criterion {gini, entropy} = 60 configs. Best: entropy, depth 12, leaf 100 (split 2 and 200 tie). CV AUC 0.651 (default, train 1.00) → **0.812** (train 0.831). [gini, depth 10, leaf 25, split 200; 0.646 → 0.807]
- **Random forest:** `RandomizedSearchCV`, 12 of 192 configs: n_estimators {150,300} × max_depth {8,12,16,None} × min_samples_leaf {1,5,20,50} × max_features {sqrt, 0.25, 0.4} × class_weight {None, balanced}. Best: 300 trees, depth 16, leaf 20, max_features 0.4, balanced. CV AUC 0.820 (default) → **0.8265** (train 0.892); best unweighted config 0.8260. [150 trees, depth 12, leaf 5, 0.4, no weighting; 0.821 → 0.8273]
- **AdaBoost:** `GridSearchCV`, base-tree depth {1,2,3} × n_estimators {100,300} × learning_rate {0.1,0.5,1.0} = 18 configs. Best: depth 3, 300 trees, learning rate 0.5. CV AUC 0.807 (default) → **0.8267** (train 0.838). [same settings; 0.8270]
- **Why these ranges:** depth/leaf ranges bracket the validation-curve optimum; RF uses random search because the full grid is ~16× larger; full notebook runtime ~6 min on 8 cores.

### D29: EDA restricted to training weeks
- **Status:** Decided, implemented (§3)
- **Decision:** all §3 plots and tables use weeks 1–14 only.
- **Why:** EDA findings shaped the engineered features; looking only at training data keeps test weeks fully unseen until §7.

### D30: Baselines (professor requirement)
- **Status:** Decided, implemented (§6.1, §7.1)
- **Result:** majority class = pass (61.4% of training plays). Majority baseline: CV accuracy 0.607 (validation weeks 7–14; 0.615 under GroupKFold), test accuracy 0.599, test pass-F1 0.749, run recall 0, AUC 0.5; test confusion matrix = all 9,740 plays predicted pass (3,904 runs wrong). Shotgun rule (pass iff shotgun): test accuracy 0.712, AUC 0.679.
- **Every trained model beats both:** +14.0 (tree), +14.3 (RF), +14.9 (AdaBoost) test-accuracy points over the majority baseline (Nov 18). [Nov 17: +14.1, +15.0, +14.9]

### D31: Final model selection
- **Status:** Decided (§6.5)
- **Decision:** pre-declared rule: highest CV ROC-AUC. `best_name = max(searches, key=best_score_)` in §6.5 uses only CV scores; no test data is touched before §7 (checked Nov 18).
- **Result (Nov 18, forward-chaining CV):** **AdaBoost** 0.8267 vs. random forest 0.8265 and decision tree 0.8117.
- **Caveat:** the AdaBoost–RF margin (0.0002) is far below the fold standard deviation (~0.004), and the order flipped between CV schemes (Nov 17 GroupKFold: RF 0.8273 vs. AdaBoost 0.8270). No significance test was run; we treat the two as indistinguishable and keep the rule rather than choose after seeing test scores.
- **Test result (Nov 18):** AdaBoost accuracy 0.748, pass precision 0.781 / recall 0.806, run recall 0.661, ROC-AUC 0.828. RF 0.742 / AUC 0.831; tree 0.739 / AUC 0.810. CV → test AUC differences ≤ 0.005.
- **Superseded result (Nov 17, GroupKFold CV, see D34):** RF selected; RF test accuracy 0.749 / AUC 0.831.

### D32: Penalty-nullified plays: sensitivity check
- **Status:** Decided (keep, as in D1), checked (§7.6)
- **Result (Nov 18):** 1,062 training and 313 test plays are nullified. *Training choice (CV only, D35):* trained with them 0.8268 vs. without 0.8255 CV AUC (validation excludes them in both). *Final model, test subgroup:* AUC 0.8281 all plays vs. 0.8292 without; accuracy 0.7482 vs. 0.7483. Keeping them does not affect conclusions.
- **Superseded result (Nov 17, GroupKFold CV, see D34):** the retrain-without-nullified variant was scored on the test set (test AUC 0.8312 / 0.8321 / 0.8317); replaced by the CV comparison under D35.

### D33: Environment and reproducibility
- **Status:** Decided (Nov 17)
- **Decision:** project-local `.venv` (gitignored) created with `uv` on Python 3.11. `requirements.txt` now pins the verified versions (pandas 3.1.1, pyarrow 26.0.1, numpy 2.4.6, scikit-learn 1.9.1, matplotlib 3.11.2, jupyter 1.1.1). `seaborn` was removed (never imported).
- **Observed:** with pandas 3 the existing §0–§2 code ran unchanged and gave the same 36,539 rows; only displayed missing values changed (`None` → `NaN`). The freshly downloaded nflverse files differ slightly from the Nov 16 download (D2 note). sklearn 1.9 no longer accepts `color=` in `RocCurveDisplay.from_predictions`, so §7.3 plots ROC curves with `roc_curve` + matplotlib.
- **Verification:** the full notebook runs top to bottom with `nbclient` in a fresh kernel (~6–7 min); random seeds are fixed (`RANDOM_STATE = 363`). Re-verified Nov 18 after D34/D35.

### D34: Forward-chaining cross-validation by week (supersedes D14)
- **Status:** Decided, implemented (Nov 18 audit, §5)
- **Issue found:** `team_prior_pass_rate` uses labels from earlier weeks. With `GroupKFold` (non-chronological folds), a training-fold play could carry a pass rate computed partly from a validation-fold game. Measured: **62.6%** of training-fold plays were affected. That is validation-label information entering training features, which can make CV optimistic and contaminates tuning/selection.
- **Decision:** 4 forward-chaining folds inside weeks 1–14. Validation blocks are weeks 7–8, 9–10, 11–12 and 13–14; each trains on all earlier weeks (12,130 → 23,166 training plays; 3,495–3,848 validation plays). Whole weeks go to one side, so games are never split.
- **Checks (asserted per fold):** training weeks < validation weeks; no shared games; recomputing `team_prior_pass_rate` from the training rows alone reproduces the training rows' values exactly, so no validation label reaches a training feature. Validation rows may use earlier *validation-block* weeks' labels, which matches deployment (those games have been played).
- **Not affected:** `prev_play` and `drive_play_number` use only the same game's earlier plays, and a game is never split across folds. Test-week rows use labels from earlier test weeks for their own features; no model is fitted on them, and in real use those games have already been played.
- **Effect:** everything in §6–§7 was re-run. CV AUCs moved by ≤ 0.005; the selected model changed from random forest to AdaBoost (D31); test metrics for each model's tuned configuration changed slightly (tree and RF were re-tuned to different settings; AdaBoost kept the same settings).
- **Alternatives:** recompute `team_prior_pass_rate` inside every fold from training-fold games only (more complex, and still trains on future weeks); drop the feature (it carries a small amount of signal).
- **Cost:** 4 folds, the first trained on only 6 weeks, so CV scores are slightly pessimistic for early folds and fold std is larger (~0.004 vs. ~0.003).

### D35: Test set used only for final evaluation (audit)
- **Status:** Decided, implemented (Nov 18 audit)
- **Issue found:** the Nov 17 notebook (§7.6) refit the selected model on each ablation feature set and on training-without-nullified-plays, and scored those **alternative models on the test set**. They were not used for selection, but this is model-development information computed on the final test set. The §7.3 text also cited set A's test AUC.
- **Decision:** feature-set comparisons and the nullified-play training choice are now CV-only (training weeks). The test set is scored only for the three final tuned models and the two baselines, plus subgroup views of the final model (error analysis, nullified subset, permutation importance) that change nothing.
- **Also confirmed:** model selection (§6.5) uses `best_score_` from the searches; no §6 cell reads `X_test`/`y_test`.
- **Wording:** removed "statistically tied" claims (notebook and D31). No significance tests were run; differences are compared to the fold standard deviation only.

---

## Changes from the original proposal
| Proposal said | We're doing | Entry |
|---|---|---|
| Predict pass vs. anything else (incl. punts, FGs) | Pass vs. run on scrimmage plays only | D2 |
| Decision tree, maybe logistic regression | Decision tree + random forest (bagging) + AdaBoost (boosting), against a majority-class and a shotgun baseline | D27 (supersedes D16) |
| Rain / not-rain weather feature | Kept, plus wind, temp, indoor flag (snow built but unusable in training) | D9, D25 |
