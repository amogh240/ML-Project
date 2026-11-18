# Project handoff (internal, not for submission)

**What we're doing.** We're trying to guess whether an NFL play will be a run or a pass using only what you could know before the ball is snapped: down, distance, field position, score, clock, weather, and how both teams lined up (formation, personnel, how many defenders are "in the box" near the line). We use two public nflverse datasets from the 2022 season and join them on game and play ID. We only look at normal offensive plays, so punts, field goals, kneels and spikes are dropped. Sacks and QB scrambles count as passes because the offense called a pass. Everything lives in `nfl_pass_prediction.ipynb`. `DECISIONS.md` logs every choice and why we made it (it's internal, not something we submit).

**What's done.** The whole notebook is finished and runs top to bottom without errors (about 6–7 minutes):
- **Data cleaning:** 36,539 plays, 61% passes. We kept only columns we're sure are known before the snap and ran a check that nothing "gives away" the answer.
- **Charts:** charts of how pass rate changes with down, distance, formation, score and clock.
- **Our own features:** e.g. counting WRs/TEs/RBs from the personnel text, a "3rd/4th and long" flag, the offense's point spread, what the previous play in the drive was, and each team's pass rate in earlier weeks.
- **Training and testing:** we trained on weeks 1–14 and saved weeks 15–18 plus playoffs as a test set we only looked at once, at the very end.
- **Three models:** a single decision tree, a random forest (many trees voting), and AdaBoost (small trees built one after another, each fixing the last one's mistakes).
- **Tuning:** we tuned each model's settings using cross-validation, i.e. repeatedly training on earlier weeks and checking on the next couple of weeks.
- **One fix to know about:** our first setup let later weeks' results sneak into the "team pass rate" feature during tuning. We switched to the week-by-week setup and re-ran everything.

**Results.** Always guessing "pass" (the most common play) gets **59.9%** right on the test weeks. All three models beat that easily:

| Model | Test accuracy | Gain over always-pass |
|---|---|---|
| Decision tree | 73.9% | +14.0 points |
| Random forest | 74.2% | +14.3 points |
| AdaBoost | 74.8% | +14.9 points |

A simple "pass if the QB is in shotgun" rule gets 71.2%, and nflverse's own pass-probability model gets 70.8% on the same plays. We picked AdaBoost because it had the best tuning score, but its lead over the random forest is tiny (0.8267 vs. 0.8265 on a 0.5–1.0 ranking score called ROC-AUC), so treat them as tied. The biggest boost came from adding the formation/personnel data. Our own extra features helped only a little. The model struggles most on 1st-and-10 and on runs called out of passing formations.

**What you should review before we submit:**
- Set up the environment and run the notebook yourself: `uv venv --python 3.11 .venv && uv pip install -r requirements.txt`, then Restart & Run All. The data downloads automatically.
- Read §2 (what we excluded and why), §5 (how we split the data), and the results in §6–§7. Make sure you can explain them at the Dec 1 Q&A.
- Things we haven't fully settled or that someone could push back on:
  - **Defenders in the box:** this is measured at the snap and partly reflects the defense reacting to the offense.
  - **AdaBoost vs. random forest:** basically a coin flip.
  - **The comparison with nflverse's model** isn't apples-to-apples, since we don't know how theirs was trained.
  - **AdaBoost's scores** rank plays well but shouldn't be read as real probabilities.
  - **The first tuning round** only trained on 6 weeks of data.
  - **Not done:** decision-threshold tuning or calibration.
- Skim `DECISIONS.md` and tell me if you disagree with anything.
- **The 5-page process writeup PDF is still completely unwritten.** We have to write it ourselves (no AI allowed), including the work-split section.
- Nothing is committed yet. Once you've looked it over, we'll commit and push together.
