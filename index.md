---
layout: default
title: Does Side Matter? Blue vs Red Win Rates in Professional League of Legends
---

# Does Side Matter? Blue vs Red Win Rates in Professional League of Legends

**By Jun Tao** — DSC 80, UC San Diego (Spring 2026)

---

## Introduction

*League of Legends* is the most-played competitive video game in the world, and at the professional level every structural advantage matters. One of the only built-in asymmetries in the game is **which side of the map a team plays on**: Blue side or Red side. The two halves of Summoner's Rift are mirror images, but they are not identical — Blue side drafts first, and the Dragon pit (a key neutral objective) sits closer to the bottom lane on the Blue side. Casters and analysts have discussed a perceived "blue-side advantage" for years.

This project investigates a single question:

> **Does the side a team is assigned to — Blue or Red — have a statistically significant effect on their probability of winning the game?**

The dataset comes from [Oracle's Elixir](https://oracleselixir.com/tools/downloads) and contains detailed post-game statistics from professional matches played during the 2022 competitive season. The raw file has **150,348 rows and 165 columns**. Each match (`gameid`) appears as 12 rows: one for each of the 10 players plus 2 team-summary rows. After filtering to team-level rows, we are left with **25,058 rows representing 12,529 unique games**.

This question matters because if side meaningfully affects win probability, tournament organizers, draft strategists, and analysts all need to treat it as a non-trivial prior — before a single champion is even picked.

The columns most relevant to our question:

| Column | Description |
| --- | --- |
| `gameid` | Unique identifier for each match. |
| `position` | Player role (`top`, `jng`, `mid`, `bot`, `sup`) or `'team'` for the per-team summary row. Used to filter to team-level data. |
| `side` | Which side of the map the team played on — `'Blue'` or `'Red'`. |
| `result` | Binary outcome: `1` if the team won, `0` if they lost. |
| `gamelength` | Length of the game in seconds (converted to minutes during cleaning). |
| `goldat25` | Team's total gold at the 25-minute mark. Used in the missingness analysis. |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

We performed the following cleaning steps:

1. **Filtered to team-level rows.** Each game appears 12 times — once per player and twice as team summaries. Since our question is about team outcomes, we kept only rows where `position == 'team'`, reducing the data from 150,348 to 25,058 rows.
2. **Selected relevant columns:** `gameid`, `league`, `side`, `result`, `gamelength`, and `goldat25`.
3. **Converted `result` to boolean** since it represents a true/false outcome stored as 0/1.
4. **Converted `gamelength` from seconds to minutes** for readability.

These steps reflect the data generating process: the raw file mixes player-level and team-level records, and our question lives entirely at the team level, so collapsing to team rows is the natural first move. The head of the cleaned DataFrame:

{% raw %}
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>gameid</th><th>league</th><th>side</th><th>result</th><th>gamelength</th><th>goldat25</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>ESPORTSTMNT01_2690210</td><td>LCKC</td><td>Blue</td><td>False</td><td>28.55</td><td>40224.0</td></tr>
    <tr><td>ESPORTSTMNT01_2690210</td><td>LCKC</td><td>Red</td><td>True</td><td>28.55</td><td>40136.0</td></tr>
    <tr><td>ESPORTSTMNT01_2690219</td><td>LCKC</td><td>Blue</td><td>False</td><td>35.23</td><td>39335.0</td></tr>
    <tr><td>ESPORTSTMNT01_2690219</td><td>LCKC</td><td>Red</td><td>True</td><td>35.23</td><td>46615.0</td></tr>
    <tr><td>8401-8401_game_1</td><td>LPL</td><td>Blue</td><td>True</td><td>22.75</td><td>NaN</td></tr>
  </tbody>
</table>
{% endraw %}

### Univariate Analysis

<iframe src="assets/univariate_side.html" width="100%" height="420" frameborder="0"></iframe>

The two sides are perfectly balanced (12,529 games each). This is by design — every game produces exactly one Blue and one Red team — and it is a precondition for a fair comparison of win rates.

<iframe src="assets/univariate_gamelength.html" width="100%" height="420" frameborder="0"></iframe>

Game length is roughly bell-shaped and centered around 31 minutes, with most games lasting between 25 and 40 minutes and a slight right skew from occasional long games.

### Bivariate Analysis

<iframe src="assets/bivariate_winrate.html" width="100%" height="420" frameborder="0"></iframe>

Blue side teams win about 52.5% of their games while Red side teams win about 47.5% — a roughly 5 percentage point gap that motivates the formal hypothesis test below.

<iframe src="assets/bivariate_gamelength.html" width="100%" height="420" frameborder="0"></iframe>

Game length is essentially identical across sides (the medians differ by a fraction of a minute). This is an important clue: Blue's advantage does **not** come from winning faster — it shows up somewhere other than game duration.

### Interesting Aggregates

{% raw %}
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;"><th>side</th><th>win_rate</th><th>avg_gamelength</th><th>n_games</th></tr>
  </thead>
  <tbody>
    <tr><th>Blue</th><td>0.525</td><td>31.584</td><td>12529</td></tr>
    <tr><th>Red</th><td>0.475</td><td>31.584</td><td>12529</td></tr>
  </tbody>
</table>
{% endraw %}

Grouping by side makes the core tension clear: win rate differs by side, but average game length is identical to three decimals. Whatever drives Blue's edge, it isn't reflected in how long games take.

---

## Assessment of Missingness

### NMAR Analysis

We believe the `playername` column for substitute players could plausibly be **NMAR** (Not Missing At Random). When a player is brought in as a last-minute substitute (e.g. due to visa or health issues), their identity may be deliberately withheld from the public match record because of contract or roster restrictions — so the reason the name is missing depends on the missing value itself. If we could obtain each team's official weekly substitute roster, the missingness would become MAR conditional on roster status.

### Missingness Dependency

We analyze `goldat25`, which is missing in about 20% of games. We hypothesize its missingness depends on `gamelength` (games ending before 25 minutes can't record 25-minute stats) but not on `side`.

<iframe src="assets/missingness.html" width="100%" height="420" frameborder="0"></iframe>

**Dependency on game length.** Test statistic: absolute difference in mean `gamelength` between rows where `goldat25` is missing vs. present. Observed difference = **3.636 minutes**, p ≈ 0. We reject independence — missingness depends strongly on game length. This is **MAR**: games that end before 25 minutes simply cannot record a 25-minute statistic, exactly as the plot above shows (missing rows cluster below 25 minutes).

**Dependency on side.** Test statistic: total variation distance between the side distributions of missing vs. present rows. Observed TVD = **0**, p = **1.0**. We fail to reject independence. This is structural: pro games produce paired Blue/Red team rows, so whenever a game ends early both sides lose their `goldat25` value together, keeping the missingness perfectly balanced across sides.

---

## Hypothesis Testing

We test whether side affects win rate.

- **Null Hypothesis (H₀):** There is no difference in win rate between Blue side and Red side teams; any observed difference is due to chance.
- **Alternative Hypothesis (H₁):** There is a difference in win rate between Blue side and Red side teams.
- **Test Statistic:** Absolute difference in win proportions, |p_blue − p_red|.
- **Significance Level:** α = 0.05.

A permutation test is appropriate because under H₀ the side label can be shuffled across rows without changing the outcome distribution, and side assignment in pro play is determined by seeding rather than team strength, so exchangeability is reasonable. We use the absolute difference because the alternative is two-sided.

<iframe src="assets/hypothesis_perm.html" width="100%" height="420" frameborder="0"></iframe>

Across 1,000 permutations, the observed statistic of **0.0496** was never matched, giving p ≈ 0. We reject the null hypothesis. The observed Blue-side win rate (~52.5%) versus Red-side (~47.5%) is highly unlikely to occur by chance under the assumption that side has no effect. This provides strong statistical evidence of a Blue-side advantage in 2022 professional *League of Legends*, consistent with longstanding community observation about pick-order and dragon-pit asymmetries. (As with any statistical test, this is strong evidence rather than absolute proof.)

---

## Framing a Prediction Problem

We frame a **binary classification** problem: predict whether a team wins (`result`) using only information knowable **before the game ends**.

We chose `result` as the response variable because it directly extends the question above — having shown that side affects win probability, we now ask how well wins can actually be predicted. We evaluate with **accuracy**, which is appropriate because the classes are perfectly balanced (every game has exactly one winner and one loser), the cost of the two error types is symmetric, and accuracy is easy to interpret. We also report F1 as a secondary check.

A key concern is the **time of prediction**: most columns (kills, gold totals, towers, dragons, game length) are post-game and would leak the outcome. Our baseline uses only pre-game features (`side`, `league`), and our final model adds features observable within the first 15 minutes — still legitimate because they precede the game's outcome.

---

## Baseline Model

Our baseline is a logistic regression in a single scikit-learn `Pipeline` using two **nominal** features known before the game starts: `side` and `league` (0 quantitative, 0 ordinal, 2 nominal). Both are one-hot encoded inside a `ColumnTransformer` so encoding happens within the pipeline and does not leak across the train/test split. We split the data 75/25 and evaluate accuracy on the held-out test set.

The baseline achieves **51.19% test accuracy** — only marginally above a 50% coin flip. This is expected: side and league are coarse signals that say little about which specific team in a matchup will win. It establishes a floor for the final model to improve on.

---

## Final Model

For the final model we add three **early-game** features that capture how the match is unfolding:

- **`golddiffat15`** — the team's gold lead at 15 minutes. Gold is the game's primary resource, so an early lead reflects a real power advantage that tends to snowball.
- **`firstblood`** — whether the team scored the first kill, a proxy for winning the opening skirmishes.
- **`firstdragon`** — whether the team took the first dragon, an objective that grants compounding buffs.

All three are observable within the first 15 minutes, so they respect the time of prediction. We keep the baseline's one-hot encoded nominal features, median-impute and standardize the three numeric features, and use a `RandomForestClassifier` (chosen because the relationship between early-game state and winning is non-linear and interaction-heavy).

We tuned `n_estimators` (100, 200), `max_depth` (5, 10, 20, None), and `min_samples_split` (2, 10) using `GridSearchCV` with 5-fold cross-validation, on the **same** train/test split as the baseline. The best hyperparameters were `max_depth=10`, `min_samples_split=10`, `n_estimators=100`.

The final model achieves **71.75% test accuracy**, up from the baseline's 51.19% — an improvement of roughly **20 percentage points**. From the data generating process perspective this makes sense: the gold difference at 15 minutes directly measures in-game dominance, which the static pre-game labels could never capture.

<iframe src="assets/confusion_matrix.html" width="100%" height="450" frameborder="0"></iframe>

The confusion matrix shows balanced errors across both classes, consistent with the balanced 50/50 outcome distribution.

---

## Fairness Analysis

We ask whether the model performs differently for two groups of games split by length:

- **Group X — Long games** (`gamelength` > 30 minutes)
- **Group Y — Short games** (`gamelength` ≤ 30 minutes)

- **Evaluation metric:** Precision.
- **Null Hypothesis (H₀):** The model is fair; its precision is the same for long and short games, and any difference is due to chance.
- **Alternative Hypothesis (H₁):** The model is unfair; its precision differs between long and short games.
- **Test Statistic:** Absolute difference in precision between the two groups.
- **Significance Level:** α = 0.05.

<iframe src="assets/fairness_perm.html" width="100%" height="420" frameborder="0"></iframe>

The model's precision on short games (~82.6%) is much higher than on long games (~62.5%), an observed difference of ~0.20. Across 1,000 permutations this gap was never matched, giving p ≈ 0, so we reject the null hypothesis. There is strong statistical evidence that the model is **not fair** with respect to game length. This aligns with the data generating process: the model leans heavily on the 15-minute gold difference, which reliably predicts short, decisive games but is frequently overturned by comebacks and late-game scaling in long games. A production model would need late-game features to predict long games as reliably as short ones.
