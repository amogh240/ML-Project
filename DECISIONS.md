# Decision Log

Every methodological choice in the project, with **what** we decided, **why**, and **what we considered instead**. This file is internal (not submitted). The notebook never cites these IDs; it explains each decision in its own words where the decision is made.

**How to use this file**
- Add an entry whenever you make a choice someone could ask "why?" about. Expect those questions at the Dec 1 Q&A.
- When results change a decision, **update the same entry**. The top of the entry always states the current decision, and a dated **How it evolved** list underneath records what we planned first, what changed it, and why. Never delete history: the changes are themselves evidence of methodology.
- Results that were later re-run keep their old numbers as **Earlier result** lines, so you can see what moved.
- Use this as source material only. AI tools aren't allowed for the process writeup, so the team writes it in its own words.

Status values: **Decided** · **Tentative** (revisit once we see data/results). "(revised <date>)" means the decision changed after it was first made; see its history.

**Merged IDs.** Some decisions were first logged as separate entries that replaced older ones. They have been folded into the original entry:

| Old ID | Now part of |
|---|---|
| D20 | D5 Missing values |
| D21 | D6 Rare categories |
| D22, D23 | D7 Leakage control |
| D25 | D9 Engineered features |
| D24 | D10 Sequence and tendency features |
| D34 | D14 Cross-validation scheme |
| D27 | D16 Model lineup |
| D28 | D17 Hyperparameter tuning |

---

## Problem definition

### D1: Label: `is_pass` = nflverse `pass` flag (dropbacks)
- **Status:** Decided (team, Nov 16)
- **Decision:** A play is a pass if nflverse's `pass` column = 1. Sacks and QB scrambles therefore count as passes. A penalty-nullified play (`play_type == "no_play"`) is kept when nflverse's `pass`/`rush` flag shows which play was called.
- **Why:** We are predicting the play call, not the outcome. A sack or scramble began as a called pass, so labeling it "run" would add label noise the model can't learn from pre-snap data.
- **Alternatives:** (a) only actual throws (`play_type == "pass"`) count as pass; rejected because sacks/scrambles become mislabeled. (b) drop sacks, scrambles and no_plays; rejected because it discards valid called plays and shrinks the data.
- **Checked later:** keeping the nullified plays doesn't change conclusions (D32).

### D2: Scope: scrimmage plays only (run vs. pass)
- **Status:** Decided (team, Nov 16). **Changes the proposal**, which said "pass vs. anything else."
- **Decision:** Keep plays where `pass == 1` or `rush == 1`. Drop special-teams plays (punts, FGs, kickoffs, XPs), 2-pt tries, QB kneels, QB spikes, and non-plays (timeouts, end of quarter, null `down`). Aborted snaps are also dropped (D19).
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

### D19: Drop aborted snaps
- **Status:** Decided (Nov 16)
- **Decision:** Drop the 113 plays with `aborted_play == 1` (fumbled or botched snaps).
- **Why:** nflverse labels 112 of the 113 as `rush` whatever the offense called, because the play never developed. Those labels reflect the botched snap rather than the call, so they are label noise. They're 0.3% of the data, so dropping them costs nothing meaningful.
- **Alternatives:** keep them as runs (adds noise); relabel from the description text (the called play usually can't be recovered).

### D5: Missing values, handled by cause
- **Status:** Decided (revised Nov 16)
- **Decision:**
  - Formation / personnel / `defenders_in_box` missing (77 plays, 0.21%, NGS tracking gaps): **drop rows**.
  - Dome/closed roof: `is_indoor = 1`, temp = 70°F, wind = 0 mph.
  - Outdoor/open roof with missing numbers: **parse temp and wind from the `weather` string**. This recovered 11,721 plays, with zero missing afterwards. Parsed ranges look realistic: 6–92°F, 0–27 mph.
  - `surface` empty string → "unknown"; missing `weather` text → "" (used for the rain/snow flags in §4).
  - A median imputer still sits inside the Pipeline as a safeguard (D11), fitted on training folds only.
- **Why:** For formation/personnel, imputing would invent exactly the data we merged to get. For weather, the true values exist in the text, so recovering them beats imputing a constant.
- **Alternatives:** treat all missing weather as indoor (the original plan, below); median imputation of temp/wind; model-based imputation (overkill); dropping all rows with any NaN (would wipe out all dome games); dropping weather features.
- **How it evolved:**
  1. **Nov 16, plan (before seeing the data):** drop rows with missing formation/personnel/box if under ~2% are missing, otherwise impute (median / "UNKNOWN"). Assume `temp`/`wind` are NaN only in domes and closed roofs, so fill 70°F / 0 mph and add `is_indoor`. Reasoning: there, a missing value means "no weather," not "unknown."
  2. **Nov 16, after seeing the data (current):** `temp`/`wind` are missing for 62% of plays. That covers all dome/closed games, but also all 1,313 open-roof plays and 10,438 outdoor plays from 81 games. For every one of those, the free-text `weather` column still holds the values ("Temp: 73° F, … Wind: W 3 mph"). The original plan would have marked 11,721 outdoor plays as "indoor 70°F, no wind," which is wrong and would have corrupted the weather analysis, so we parse the text instead. The formation/personnel rule held: only 0.21% missing, so we drop.

### D6: Rare categories: handled inside the Pipeline; personnel parsed into counts
- **Status:** Decided (revised Nov 16)
- **Decision:** (1) Parse personnel strings into position counts in §4. (2) Wherever a grouping is one-hot encoded, use `OneHotEncoder(min_frequency=0.005, handle_unknown="infrequent_if_exist")` inside the Pipeline.
- **Why:** Very rare one-hot columns add noise and invite overfitting. Choosing the rare-category cut-off from the full dataset uses test-set frequencies, a small leak. Doing it in the encoder fits the cut-off on each training fold. Position counts generalize across rare groupings (an unusual "6 OL, 1 RB, 2 TE, 1 WR" still reads as "heavy").
- **Alternatives:** a manual pre-split merge to "OTHER" (the original plan).
- **Finding (Nov 16):** All 7 offensive formations are ≥ 0.5% of plays (the rarest, WILDCAT, is 0.71%), so no formation merging is needed. Personnel has 64 distinct strings on each side. The top 5 cover 94.8% (offense) / 82.8% (defense), and 56 / 51 groupings are each under 0.5%. There are also odd strings ("2 QB, …", "…,1 LB" on offense).
- **How it evolved:**
  1. **Nov 16, plan:** group formation and personnel groupings under ~0.5% of plays into "OTHER" before modeling, with the threshold to be set after EDA.
  2. **Nov 16, after seeing the data (current):** formations didn't need merging at all. For personnel, we moved the merge into the encoder (fit per training fold) to avoid using test-set frequencies, and added count parsing so rare groupings still carry meaning.

### D7: Leakage control: whitelist of pre-snap columns, verified by a univariate-AUC scan
- **Status:** Decided (revised Nov 16, 17, 18)
- **Decision:** Keep an explicit whitelist of 26 pre-snap feature candidates, 9 ID columns (including `order_sequence`, for sorting only) and 1 benchmark (`xpass`, D8). Exclude everything else. As a safety net, compute every numeric column's univariate AUC against the label on **training weeks only**, scored by value and by missingness, and assert that no whitelisted feature exceeds 0.90.
- **Why:** The model is meant to predict before the snap, and any post-snap column hands it the answer (e.g. `air_yards` only exists for passes). With about 390 columns, a blacklist fails silently when one is forgotten; a whitelist fails safe. The AUC scan catches leaks nobody thought of, including missingness leaks that a name-based review would miss.
- **Alternatives:** a blacklist of known leaky columns (the original plan); manual review only.
- **Other calls made here:** `wp` is allowed as a candidate (pre-snap win probability from game state; it doesn't model the play call), though we later chose not to use it (D26). `n_offense`/`n_defense` are excluded (tracking counts, almost always 11, no football meaning). `drive` is excluded in favor of nflverse's corrected `fixed_drive`.
- **Current scan result (Nov 18, weeks 1–14, 26,866 plays at that point in cleaning):** `defenders_in_box` 0.710, `shotgun` 0.692, `down` 0.619, `wp` 0.612; `xpass` 0.787. Excluded leaky columns still score 0.92–1.00.
- **How it evolved:**
  1. **Nov 16, plan:** explicitly drop every post-snap column: pbp outcome columns (`yards_gained`, `epa`, `wpa`, `air_yards`, `complete_pass`, `sack`, `qb_scramble`, `desc`, `play_type`, `rush`, etc.) and participation post-snap columns (`number_of_pass_rushers`, `time_to_throw`, `was_pressure`, `route`, `ngs_air_yards`, `defense_coverage_type`, `defense_man_zone_type`, player ID lists), with an assert that none reach the feature matrix.
  2. **Nov 16, implementation:** switched from a blacklist to a whitelist plus the AUC scan, because a blacklist fails silently. The whitelist then had 26 candidates, 8 IDs, 1 auxiliary column (`yards_gained`, to build the previous play's yardage, D10) and 1 benchmark. The 353 excluded columns were tagged: post-snap outcome 224; player identity 36; drive/series summary 18; label-derived 13; NGS post-snap 7; not needed 55. The near-perfect columns were all excluded: `pass_oe`, `qb_dropback`, `pass_attempt`, `number_of_pass_rushers`, `air_yards`, jersey numbers. Several leak only through missingness (e.g. `passer_jersey_number` is non-null only on passes). Strongest whitelisted features: `defenders_in_box` 0.71, `shotgun` 0.69, `down` 0.62; `xpass` 0.79. This scan used all weeks.
  3. **Nov 17, whitelist roles changed:** `play_id` turned out not to be chronological in 12 of 284 games (e.g. 2022_01_BAL_NYJ has play 3037 before 2844). nflverse's `order_sequence` is the true order, with no gaps or duplicates within a game, so it was added as an ID (sorting only). `yards_gained` is 0 on every penalty-nullified play that has a value (1,337 rows), so "previous play's yards" would be misleading after a nullified play. It was dropped (nothing used it anymore) and replaced by the auxiliary `is_nullified` flag. Feature ranking unchanged; with the re-downloaded data `wp` went from 0.608434 to 0.608055. Alternatives considered: sort by `play_id` (wrong order in 12 games); keep `prev_play_yards` (noisy after nullified plays).
  4. **Nov 18, final cleanup:** the scan now uses training weeks only, so no test-week labels are inspected during the leakage audit. The whitelist and the 0.90 assert are unchanged, and no feature choice depended on the scan.
  5. **Trim pass (current):** the notebook was cut down to match the course guideline ("brief" markdown; rationale goes in the writeup). The auxiliary `is_nullified` column was removed along with the nullified-play check (D32). The 28-line regex table that tagged every excluded column with a reason was also removed: the whitelist and AUC scan already show the point. The reason counts recorded in step 2 remain here for the writeup.

### D8: Exclude `xpass`/`pass_oe` as features; use `xpass` as a benchmark
- **Status:** Decided
- **Decision:** `xpass` (nflverse's own pass-probability model) and `pass_oe` (pass minus xpass) are never used as inputs. We report `xpass`'s AUC and accuracy on our test set next to our models.
- **Why:** `pass_oe` contains the label. Using `xpass` as a feature would mean copying someone else's model. As a benchmark, though, it gives an honest outside reference point.
- **Result (Nov 18), §7.3:** `xpass` has no missing values on the test plays. Test AUC 0.789, accuracy 70.8% (threshold 0.5) vs. the selected AdaBoost 0.828 / 74.8% (tree 0.810, RF 0.831). The gap plausibly comes from participation data (CV: situation-only set A 0.771); the Nov 17 version cited a test AUC for set A, which was removed under D35. We don't know which seasons nflverse trained `xpass` on, so this is a reference point, not a clean head-to-head.

## Feature engineering

### D9: Engineered features (all known before the snap)
- **Status:** Decided (revised Nov 17)
- **Decision (as built, §4.1–4.5):** personnel counts `n_rb, n_te, n_wr, n_ol` (OL absent from string → 5) and `n_dl, n_lb, n_db`; `box_minus_blockers = defenders_in_box − (n_ol + n_te)`; `ydstogo_bin` (1–2 / 3–6 / 7–10 / 11+, one-hot); `red_zone` (≤ 20 yards); `third_fourth_short` (down ≥ 3, ≤ 2 yards); `third_fourth_long` (down ≥ 3, ≥ 7 yards); `two_minute` (≤ 120 s left in half); `trailing_late` (behind with ≤ 900 game seconds left); `is_home`; `posteam_spread` (D26); `is_rain` (outdoor/open games, "rain|shower" in the weather text); the sequence features from D10.
- **Final set C:** 37 numeric + 4 categorical = 41 input columns → 72 after one-hot encoding.
- **Why:** Personnel strings like "1 RB, 2 TE, 2 WR" are useless to a model as raw text but very informative as counts. Football knowledge says passing is more likely with more WRs, a lighter box, long distance, trailing late, and the two-minute drill, so explicit flags make those patterns easy to learn. Rain/no-rain follows the proposal's plan to keep the weather columns few.
- **Excluded on purpose:** raw `spread_line` (wrong perspective, D26); `posteam_type`, `roof`, raw `weather` text (each replaced by a flag); `surface` (no clear football reason); team identities (would memorize teams; tendency is carried by `team_prior_pass_rate`); `wp` (D26).
- **Personnel parsing observed:** 35,249 offensive strings have 5 RB+TE+WR; 1,165 have 4 (e.g. a 2-QB or defender-on-offense look). On defense, 36,489 strings sum to 11, 36 to 10 and 14 to 12. Irregular positions are ignored, not dropped.
- **Univariate AUC on training weeks (not used for selection):** `n_db` 0.649, `box_minus_blockers` 0.644, `n_wr` 0.637 … `is_rain` 0.504, `posteam_spread` 0.501.
- **How it evolved:**
  1. **Nov 16, plan:** personnel counts parsed from strings; box-vs-blockers; formation one-hots; down/distance buckets and flags; clock/score flags (two-minute drill, trailing late); weather (rain/snow parsed, wind, temp, indoor); interactions (`shotgun × n_wr`, `score_diff × time_remaining`).
  2. **Nov 17, as built:** mostly as planned, with three differences. The two interaction terms were not built, because the lineup at the time was all trees (D16), which find interactions themselves. `is_snow` was built but not used: the only snow game (2022_20_CIN_BUF) is a playoff game in the test weeks, so the flag is all zeros in training (an assert in §5 documented this). `spread_line` was converted to the offense's perspective, and `wp` was left out (D26).
  3. **Trim pass (current):** `is_snow` is no longer built at all (it was unusable in training). Logistic regression joined the lineup (D16); the interaction terms were still not added, since the explicit flags already give it the key thresholds, and the ablation shows the engineered features help it more than they help AdaBoost (D12). The separate univariate-AUC table for engineered features (§5.2) was removed, since the ablation and permutation importance cover it.

### D10: Sequence and team-tendency features, computed without leakage
- **Status:** Decided (revised Nov 17, 18)
- **Decision (as built, §4.4):**
  - `prev_play` ∈ {pass, run, none}: the label of the previous kept scrimmage play in the same `(game_id, fixed_drive)`, after sorting by `order_sequence`. "none" = first play of the drive. One-hot encoded.
  - `drive_play_number`: 1, 2, … within the drive.
  - `team_prior_pass_rate`: the offense's pass rate over all earlier **weeks** (cumulative sums minus the current week). NaN in week 1, filled by the Pipeline's median imputer (fit on training folds).
- **Why:** Coaches and defenses use these signals in real life, and all of them are known before the snap. The previous play's type is public before the next snap, and a play never sees its own label. A team pass rate that included the current game would leak the label, hence "earlier weeks only."
- **Checks in the notebook:** first play of every drive has `prev_play == "none"`; each team plays at most one game per week (so "earlier week" = "earlier game"); KC week-10 value equals a direct mean over weeks 1–9 (0.707); week 1 is all NaN.
- **Result (Nov 18, forward-chaining CV):** removing all three lowers CV AUC from 0.8267 to 0.8238 (within ~1 fold std). Permutation importance (AdaBoost, test): `prev_play` and `team_prior_pass_rate` small but non-zero; `drive_play_number` ≈ 0.
- **Earlier result (Nov 17, GroupKFold CV):** 0.8273 → 0.8240 (test 0.8312 → 0.8268).
- **How it evolved:**
  1. **Nov 16, plan:** `prev_play_was_pass` and `prev_play_yards` (lag within the same game and drive, ordered by `play_id`), `drive_play_number`, and the team's season-to-date pass rate from previous games only (expanding mean shifted one game).
  2. **Nov 17, as built:** `prev_play_yards` dropped, because `yards_gained` is 0 on penalty-nullified plays (see D7). Ordering switched to `order_sequence`, because `play_id` is out of order in 12 games. The tendency feature uses earlier weeks, checked to equal earlier games.
  3. **Nov 18, audit (current):** under the original `GroupKFold` CV, `team_prior_pass_rate` let validation-fold labels into training-fold features (62.6% of training-fold plays affected). The feature itself was kept; the CV scheme was changed to forward-chaining by week to remove the leak (D14).

### D11: sklearn Pipeline + ColumnTransformer
- **Status:** Decided
- **Decision:** One-hot encoding and scaling are fitted inside a `Pipeline`, so they are refit on each CV training fold.
- **Why:** Fitting a scaler or encoder on all the data before splitting leaks test-set information. Scaling is needed for LR, KNN and MLP. It's harmless for trees, so we use one pipeline everywhere for consistency.
- **Implemented (Nov 17):** `make_pipeline()` in §5.1 = `ColumnTransformer`(median `SimpleImputer` on numeric; `OneHotEncoder(min_frequency=0.005, handle_unknown="infrequent_if_exist")` on categorical) + model. 41 input columns → 72 model columns after encoding.
- **How it evolved:** (1) Nov 16 plan: scaler for every model. (2) Nov 17: no scaler, since the lineup was tree-only and tree splits are invariant to monotone rescaling. (3) Trim pass (current): `make_pipeline(..., scale=True)` adds a `StandardScaler` after the numeric imputer for logistic regression only; trees stay unscaled. One-hot columns are left as 0/1.

### D12: Ablation study to justify the merge and the feature engineering
- **Status:** Decided
- **Decision:** Train the same model on three feature sets: (a) pbp situational only, (b) + participation (formation/personnel/box), (c) + engineered features.
- **Why:** This measures directly whether merging the second dataset and our feature engineering helped, instead of assuming it did.
- **Current (trim pass), §6.7:** tuned logistic regression and the selected model (AdaBoost) on sets A (16 situation columns), B (+ shotgun, no_huddle, box, formation, raw personnel = 22), C (+ engineered = 41). CV only. Hyperparameters were tuned on C, which slightly favors C.
- **Result (trim pass, forward-chaining CV):** logistic regression A 0.718 → B 0.809 → C 0.818; AdaBoost A 0.771 → B 0.821 → C 0.827. The merge helps both (+0.091 LR, +0.050 AdaBoost). Engineered features help LR more (+0.009, ~1.5 fold std) than AdaBoost (+0.005, ~1 fold std), which fits the original reason for building explicit flags: a linear model can't learn thresholds or interactions itself.
- **Dropped variants:** "C without sequence/tendency" and "C + `wp`" were removed from the notebook in the trim pass to keep the ablation to one clear question. Their Nov 18 results stay below and in D10/D26.
- **Implemented earlier (Nov 17), §6.6 + §7.6:** selected model (tuned RF) on sets A, B, C plus the two variants.
- **Result (Nov 18, forward-chaining CV, training weeks only):** CV AUC A 0.771 → B 0.821 → C 0.827; C without sequence/tendency 0.824; C + `wp` 0.827. Participation merge = +0.050 AUC (+4.9 pts CV accuracy); engineered features = +0.005 AUC, about one fold standard deviation (~0.004), so weak evidence. No test-set scores for feature sets (D35).
- **Earlier result (Nov 17, GroupKFold CV, see D14):** CV AUC A 0.777 → B 0.823 → C 0.827; test AUC A 0.785 → B 0.826 → C 0.831. Participation merge = +0.046 CV AUC (+4.5 pts CV accuracy); engineered features = +0.004 CV AUC, most of it from sequence/tendency (C without them: 0.824). The test-set scores were removed under D35.

### D26: Offense-relative spread and exclusion of `wp`
- **Status:** Decided, implemented (§4.2, §6.6)
- **Finding:** `spread_line` is the home team's expected margin. Over all plays, raw `spread_line` has correlation 0.00 with the offense's score differential; `posteam_spread = spread_line if home else −spread_line` has +0.31 (the flipped sign would give −0.31). An assert (> 0.2) guards the sign. (A first attempt checked the sign against `wp` on each game's first play and got only +0.29, so we switched to the score check.)
- **Decision on `wp`:** keep it out of the main set. It is a pre-snap nflverse model output, nearly redundant with `score_differential` (r = 0.92 in training weeks), and including it would make our model partly a wrapper around nflverse's model.
- **Result (Nov 18, forward-chaining CV):** C + `wp`: CV AUC 0.8270 vs. 0.8267 without. Negligible, so the exclusion costs nothing.
- **Earlier result (Nov 17, GroupKFold CV):** 0.8283 vs. 0.8273 (test 0.8319 vs. 0.8312).
- **How it evolved:** on Nov 16, `wp` passed the leakage audit and was allowed as a candidate (D7). On Nov 17, when building the feature set, we left it out for the redundancy/wrapper reasons above, and checked that this costs nothing.

## Evaluation methodology

### D13: Temporal train/test split
- **Status:** Decided
- **Decision:** Train on regular-season weeks 1–14; test on weeks 15–18 and the playoffs.
- **Why:** A real deployment predicts future games from past ones. A random split puts plays from the same game/drive in both train and test (a leakage risk) and conflicts with the season-to-date tendency features.
- **Alternatives:** a random stratified 80/20 split (simpler, but optimistic).
- **Result (Nov 17):** nflverse numbers POST weeks 19–22, so the split is `week <= 14` vs. `week >= 15`. Train 26,799 plays / 208 games (61.4% pass); test 9,740 plays / 76 games (59.9% pass). Asserts: no shared games, both classes present, no duplicate plays.

### D14: Cross-validation: forward-chaining by week
- **Status:** Decided (revised Nov 18)
- **Decision (current, §5):** 4 forward-chaining folds inside weeks 1–14. Validation blocks are weeks 7–8, 9–10, 11–12 and 13–14; each trains on all earlier weeks (12,130 → 23,166 training plays; 3,495–3,848 validation plays). Whole weeks go to one side, so games are never split.
- **Why:** Plays within a game are correlated (same teams, same game script), so games must not be split across folds. Folds must also be chronological, because `team_prior_pass_rate` (D10) uses labels from earlier weeks.
- **Checks (asserted per fold):** training weeks < validation weeks; no shared games; recomputing `team_prior_pass_rate` from the training rows alone reproduces the training rows' values exactly, so no validation label reaches a training feature. Validation rows may use earlier validation-block weeks' labels, which matches deployment (those games have been played).
- **Not affected:** `prev_play` and `drive_play_number` use only the same game's earlier plays, and a game is never split across folds. Test-week rows use labels from earlier test weeks for their own features; no model is fitted on them, and in real use those games have already been played.
- **Cost:** 4 folds, the first trained on only 6 weeks, so CV scores are slightly pessimistic for early folds and fold std is larger (~0.004 vs. ~0.003).
- **Alternatives:** `GroupKFold` by game (the original choice, below); recompute `team_prior_pass_rate` inside every fold from training-fold games only (more complex, and still trains on future weeks); drop the feature (it carries a small amount of signal).
- **How it evolved:**
  1. **Nov 16, plan:** 5-fold `GroupKFold` on the training weeks, grouped by `game_id`, so a game's correlated plays never land on both sides.
  2. **Nov 17, implemented:** `GroupKFold(n_splits=5)`, folds of 5,297–5,404 plays. We noticed that folds aren't chronological, so a training-fold row's `team_prior_pass_rate` can include labels of an earlier validation-fold game. We accepted it at the time: that information is available before the snap in real life, a validation game never sees its own labels, the final test set is strictly later, and CV and test AUCs ended up within ~0.005 of each other.
  3. **Nov 18, audit (current):** we measured the effect and decided it was leakage after all. **62.6%** of training-fold plays carried a pass rate computed partly from a validation-fold game. That puts validation-label information into training features, which can make CV optimistic and contaminates tuning and model selection. We switched to forward-chaining folds and re-ran everything in §6–§7. CV AUCs moved by ≤ 0.005; the selected model changed from random forest to AdaBoost (D31); tree and RF were re-tuned to different settings, while AdaBoost kept the same ones (D17).

### D15: ROC-AUC is the primary metric
- **Status:** Decided
- **Decision:** Tune and compare on ROC-AUC. Also report accuracy, log-loss, F1, precision/recall, and confusion matrices.
- **Why:** The classes are imbalanced (~60/40), so accuracy alone flatters "always pass." AUC measures ranking quality across all thresholds, and log-loss/calibration matter if the output is used as a probability (as `xpass` is).
- **Implemented (Nov 17):** positive class = pass. CV tracks AUC, accuracy, F1; test reports accuracy, precision/recall/F1 for pass, recall for run, macro F1, ROC-AUC (from probabilities), confusion matrices. Log-loss and calibration curves were **not** done (time; not required).
- **Trim pass:** §5.2 of the notebook now has a table defining every metric (ROC curve vs. ROC-AUC, accuracy, precision/recall, F1, macro F1) and the CV columns (CV mean, CV AUC std, train-fold AUC), at the team's request.

### D29: EDA restricted to training weeks
- **Status:** Decided, implemented (§3)
- **Decision:** all §3 plots and tables use weeks 1–14 only.
- **Why:** EDA findings shaped the engineered features; looking only at training data keeps test weeks fully unseen until §7.

### D35: Test set used only for final evaluation (audit)
- **Status:** Decided, implemented (Nov 18 audit)
- **Issue found:** the Nov 17 notebook (§7.6) refit the selected model on each ablation feature set and on training-without-nullified-plays, and scored those **alternative models on the test set**. They were not used for selection, but this is model-development information computed on the final test set. The §7.3 text also cited set A's test AUC.
- **Decision:** feature-set comparisons and the nullified-play training choice are now CV-only (training weeks). The test set is scored only for the three final tuned models and the two baselines, plus subgroup views of the final model (error analysis, nullified subset, permutation importance) that change nothing.
- **Also confirmed:** model selection (§6.5) uses `best_score_` from the searches; no §6 cell reads `X_test`/`y_test`.
- **Wording:** removed "statistically tied" claims (notebook and D31). No significance tests were run; differences are compared to the fold standard deviation only.

## Modeling

### D16: Model lineup: logistic regression + three tree-based models
- **Status:** Decided (revised Nov 17, revised again in the trim pass)
- **Decision (current, §6):** `DummyClassifier(most_frequent)` and a shotgun rule as baselines (D30); then `LogisticRegression` (linear, scaled inputs), `DecisionTreeClassifier`, `RandomForestClassifier` (bagging + random feature subsets), `AdaBoostClassifier` (boosting, SAMME, sklearn 1.9.1; default base learner = decision stump).
- **Why:** One linear model, one single tree, one bagging ensemble and one boosting ensemble give a clean comparison across the course's main model families, and all four are explainable in the Q&A.
- **Alternatives:** the broader original lineup (below); `HistGradientBoostingClassifier` instead of AdaBoost (not needed, AdaBoost worked).
- **How it evolved:**
  1. **Nov 16, plan:** baselines (always-pass, shotgun rule) → logistic regression → decision tree → KNN → random forest → gradient boosting → MLP → voting/stacking ensembles. Reasoning: baselines set the floor; LR and the tree come from the proposal and are interpretable; RF and GBM test whether nonlinearity helps; KNN and MLP add different inductive biases; ensembles test whether combining models beats the best single one.
  2. **Nov 17:** cut to three tree-based models to match the project requirements. Logistic regression, KNN, MLP and stacking were dropped. A side effect: no scaler was needed in the Pipeline (D11), and interaction features weren't needed (D9).
  3. **Trim pass (current, team decision):** **logistic regression added back** as a fourth model. It was in the original proposal, it's the only linear model, and it's the most interpretable (coefficients). It needs a scaler (D11). Result: CV AUC 0.818, between the tuned tree (0.812) and the ensembles (0.827); test AUC 0.818, accuracy 74.3%. It isn't selected.

### D17: Hyperparameter tuning: search spaces and results
- **Status:** Decided (revised Nov 17, 18)
- **Decision (current, §6.2–6.5):** all searches use 4-fold forward-chaining CV (D14), `scoring={roc_auc, accuracy, f1}`, `refit="roc_auc"`, `return_train_score=True`, seed 363. Results below are from Nov 18, with the Nov 17 `GroupKFold` values in brackets.
  - **Logistic regression (added in the trim pass):** `GridSearchCV` over C {0.001, 0.01, 0.1, 1, 10} × `l1_ratio` {0 (L2), 1 (L1)} = 10 configs, `solver="liblinear"`. (`penalty=` is deprecated in sklearn 1.9, hence `l1_ratio`.) Best: C = 1, L2, which is the sklearn default. CV AUC 0.8184 (train 0.8202); every C ≥ 0.1 is within 0.0005. Regularization barely matters with ~27k plays and 72 inputs, and the tiny train–CV gap shows the linear model doesn't overfit.
  - **Decision tree:** `validation_curve` over `max_depth` ∈ {1,…,30} (best 6, CV AUC 0.791 [8, 0.793]), then `GridSearchCV` over depth {4,6,8,10,12} × min_samples_leaf {1,25,100} × min_samples_split {2,200} × criterion {gini, entropy} = 60 configs. Best: entropy, depth 12, leaf 100 (split 2 and 200 tie). CV AUC 0.651 (default, train 1.00) → **0.812** (train 0.831). [gini, depth 10, leaf 25, split 200; 0.646 → 0.807]
  - **Random forest:** `RandomizedSearchCV`, 12 of 192 configs: n_estimators {150,300} × max_depth {8,12,16,None} × min_samples_leaf {1,5,20,50} × max_features {sqrt, 0.25, 0.4} × class_weight {None, balanced}. Best: 300 trees, depth 16, leaf 20, max_features 0.4, balanced. CV AUC 0.820 (default) → **0.8265** (train 0.892); best unweighted config 0.8260. [150 trees, depth 12, leaf 5, 0.4, no weighting; 0.821 → 0.8273]
  - **AdaBoost:** `GridSearchCV`, base-tree depth {1,2,3} × n_estimators {100,300} × learning_rate {0.1,0.5,1.0} = 18 configs. Best: depth 3, 300 trees, learning rate 0.5. CV AUC 0.807 (default) → **0.8267** (train 0.838). [same settings; 0.8270]
- **Why:** Depth/leaf ranges bracket the validation-curve optimum. RF uses random search because the full grid is ~16× larger. A validation curve shows why the chosen depth works (the bias–variance tradeoff), not just that it does. Full notebook runtime ~6 min on 8 cores.
- **How it evolved:**
  1. **Nov 16, plan:** GridSearchCV for small grids (LR, tree, KNN), RandomizedSearchCV for large ones (RF, GBM, MLP), all with GroupKFold and AUC scoring. Validation curves for tree depth and GBM learning rate, plus a learning curve for the best model.
  2. **Nov 17, implemented:** searches for the three models that remained (D16), with 5-fold `GroupKFold`. Validation curve done for tree depth only; no learning curve (time).
  3. **Nov 18 (current):** all searches re-run with forward-chaining CV after the leakage audit (D14). Tree and RF picked different settings; AdaBoost kept the same ones.

### D18: Class imbalance handling
- **Status:** Decided (Nov 17), partly done.
- **Decision:** Compare `class_weight="balanced"` with no weighting, and tune the decision threshold against the default 0.5.
- **Why:** The imbalance is mild (~60/40), so we expect little change, but we'll check rather than assume.
- **What was actually done:** `class_weight` ∈ {None, "balanced"} is part of the random-forest search (D17). Threshold tuning was **not** done; all label metrics use 0.5.
- **Result (Nov 18, forward-chaining CV):** the RF search selects `class_weight="balanced"` (CV AUC 0.8265 vs. 0.8260 for the best unweighted config). Balanced weighting lowers CV accuracy at threshold 0.5 (0.738 vs. 0.750) and on test gives run recall 0.81 / pass recall 0.69.
- **Earlier result (Nov 17, GroupKFold CV, see D14):** In the RF search the best unweighted and best balanced configurations tie on CV AUC (0.8273 vs. 0.8272). In an earlier run (before `is_snow` was removed, D9) the balanced one won by a similar margin. Weighting does not change ranking quality (AUC); at threshold 0.5 it shifts predictions toward "run" (earlier run: test run recall 0.82 / pass recall 0.69 vs. final 0.63 / 0.83, similar accuracy). The majority-class pass share (61%) is mild, so we kept AUC as the selection metric and did not tune the threshold.

### D30: Baselines (professor requirement)
- **Status:** Decided, implemented (§6.1, §7.1)
- **Result:** majority class = pass (61.4% of training plays). Majority baseline: CV accuracy 0.607 (validation weeks 7–14; 0.615 under GroupKFold), test accuracy 0.599, test pass-F1 0.749, run recall 0, AUC 0.5; test confusion matrix = all 9,740 plays predicted pass (3,904 runs wrong). Shotgun rule (pass iff shotgun): test accuracy 0.712, AUC 0.679.
- **Every trained model beats both:** +14.0 (tree), +14.3 (RF), +14.9 (AdaBoost) test-accuracy points over the majority baseline (Nov 18). [Nov 17: +14.1, +15.0, +14.9]

### D31: Final model selection
- **Status:** Decided (§6.5)
- **Decision:** pre-declared rule: highest CV ROC-AUC. `best_name = max(searches, key=best_score_)` in §6.5 uses only CV scores; no test data is touched before §7 (checked Nov 18).
- **Result (Nov 18, forward-chaining CV):** **AdaBoost** 0.8267 vs. random forest 0.8265 and decision tree 0.8117. After logistic regression was added (trim pass) it scored 0.8184, so the selection is unchanged.
- **Caveat:** the AdaBoost–RF margin (0.0002) is far below the fold standard deviation (~0.004), and the order flipped between CV schemes (Nov 17 GroupKFold: RF 0.8273 vs. AdaBoost 0.8270). No significance test was run; we treat the two as indistinguishable and keep the rule rather than choose after seeing test scores.
- **Test result (Nov 18):** AdaBoost accuracy 0.748, pass precision 0.781 / recall 0.806, run recall 0.661, ROC-AUC 0.828. RF 0.742 / AUC 0.831; tree 0.739 / AUC 0.810. CV → test AUC differences ≤ 0.005.
- **Earlier result (Nov 17, GroupKFold CV, see D14):** RF selected; RF test accuracy 0.749 / AUC 0.831.

### D32: Penalty-nullified plays: sensitivity check
- **Status:** Decided (keep, as in D1). Checked on Nov 18; the check was **removed from the notebook in the trim pass** because it is a lot of code for a decision affecting ~1,400 plays, and its result (below) changes nothing. Use the numbers here for the writeup/Q&A.
- **Result (Nov 18):** 1,062 training and 313 test plays are nullified. Training choice (CV only, D35): trained with them 0.8268 vs. without 0.8255 CV AUC (validation excludes them in both). Final model, test subgroup: AUC 0.8281 all plays vs. 0.8292 without; accuracy 0.7482 vs. 0.7483. Keeping them does not affect conclusions.
- **Earlier result (Nov 17, GroupKFold CV, see D14):** the retrain-without-nullified variant was scored on the test set (test AUC 0.8312 / 0.8321 / 0.8317); replaced by the CV comparison under D35.

## Environment

### D33: Environment and reproducibility
- **Status:** Decided (Nov 17)
- **Decision:** project-local `.venv` (gitignored) created with `uv` on Python 3.11. `requirements.txt` now pins the verified versions (pandas 3.1.1, pyarrow 26.0.1, numpy 2.4.6, scikit-learn 1.9.1, matplotlib 3.11.2, jupyter 1.1.1). `seaborn` was removed (never imported).
- **Observed:** with pandas 3 the existing §0–§2 code ran unchanged and gave the same 36,539 rows; only displayed missing values changed (`None` → `NaN`). The freshly downloaded nflverse files differ slightly from the Nov 16 download (D2 note). sklearn 1.9 no longer accepts `color=` in `RocCurveDisplay.from_predictions`, so §7.3 plots ROC curves with `roc_curve` + matplotlib.
- **Verification:** the full notebook runs top to bottom with `nbclient` in a fresh kernel (~6–7 min); random seeds are fixed (`RANDOM_STATE = 363`). Re-verified Nov 18 after the D14 and D35 changes.

---

## Changes from the original proposal
| Proposal said | We're doing | Entry |
|---|---|---|
| Predict pass vs. anything else (incl. punts, FGs) | Pass vs. run on scrimmage plays only | D2 |
| Decision tree, maybe logistic regression | Logistic regression + decision tree + random forest (bagging) + AdaBoost (boosting), against a majority-class and a shotgun baseline | D16 |
| Rain / not-rain weather feature | Kept, plus wind, temp, indoor flag (snow built but unusable in training) | D9 |
