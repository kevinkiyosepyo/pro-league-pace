# Tempo of the Game: Which Pro League Plays the Bloodiest League of Legends?

By Kevin Pyo

A DSC 80 project at UC San Diego, built on pro match data from [Oracle's Elixir](https://oracleselixir.com/tools/downloads).

## Introduction

Professional League of Legends is played across regional leagues with very different reputations. The LPL (China) is known for chaotic, fight-heavy games; the LCK (Korea) for slow, methodical macro play. This project asks which league actually plays the most action-packed games, and in particular whether the LPL really is bloodier than the LCK. I measure action as combined kills per minute (CKPM): total kills by both teams in a game, divided by the game's length in minutes.

The data covers professional matches from the 2022 season (2022-01-10 to 2022-02-11), 18,143 rows and 165 columns. Each game contributes 12 rows, one per player plus a summary row per team, and after cleaning the analysis works with 1,512 games. The columns that matter here: `league`, `side` (Blue or Red), `result` (win or loss), `kills` and `deaths` (together, a game's total kills), `gamelength` in seconds, `datacompleteness` (whether detailed stats were recorded), and the 15-minute timeline stats `golddiffat15`, `xpdiffat15`, `csdiffat15`, `killsat15`, and `deathsat15`, which power the prediction task in the second half. Pace is a real part of the viewing experience, so quantifying it tells fans what each league serves up. And measuring how well 15-minute leads predict wins says how snowbally the pro game really is.

## Data Cleaning and Exploratory Data Analysis

### Cleaning

Five steps get the raw file ready. Keep only the two team summary rows per game (`position == 'team'`), because keeping player rows would count every kill five extra times. Convert 0/1 columns like `result`, `firstblood`, and `firstdragon` to booleans that preserve missing values. Parse `date` as a timestamp. Derive `gamelength_min`, `total_kills = kills + deaths` (one team's deaths are the other's kills), and the action metric `kpm = total_kills / gamelength_min`. Finally, cut to one row per game by keeping the Blue-side row, since game-level stats are identical on both rows and each game should count once. The head of the cleaned game-level DataFrame:

| gameid                | date                | league   | playoffs   |   game |   gamelength_min |   total_kills |      kpm | datacompleteness   |
|:----------------------|:--------------------|:---------|:-----------|-------:|-----------------:|--------------:|---------:|:-------------------|
| ESPORTSTMNT01_2690210 | 2022-01-10 07:44:08 | LCKC     | False      |      1 |          28.55   |            28 | 0.980736 | complete           |
| ESPORTSTMNT01_2690219 | 2022-01-10 08:38:24 | LCKC     | False      |      1 |          35.2333 |            19 | 0.539262 | complete           |
| 8401-8401_game_1      | 2022-01-10 09:24:26 | LPL      | False      |      1 |          22.75   |            19 | 0.835165 | partial            |
| ESPORTSTMNT01_2690227 | 2022-01-10 09:51:16 | LCKC     | False      |      1 |          32.8667 |            19 | 0.578093 | complete           |
| 8401-8401_game_2      | 2022-01-10 10:09:22 | LPL      | False      |      2 |          24.0667 |            31 | 1.28809  | partial            |

### Univariate analysis

The distribution of combined kills per minute is roughly bell-shaped, centered around 0.7, with a right tail of true bloodbaths.

<iframe src="assets/kpm-dist.html" width="100%" height="500" frameborder="0"></iframe>

### Bivariate analysis

Sorting leagues by their kills-per-minute distributions puts the LPL near the top of the major regions and the LCK at the bottom. The gap between those two boxes is what the hypothesis test below checks formally.

<iframe src="assets/kpm-by-league.html" width="100%" height="500" frameborder="0"></iframe>

### Aggregates

Grouping by league confirms the picture numerically. The LPL plays shorter and bloodier games than the LCK, which sits last in mean kills per minute among the major leagues shown.

| league   |   games |   mean_length_min |   mean_total_kills |   mean_kpm |
|:---------|--------:|------------------:|-------------------:|-----------:|
| VCS      |       3 |             26.55 |              31    |       1.18 |
| PCS      |       4 |             30.6  |              30.25 |       0.98 |
| LCO      |      24 |             31.3  |              27.5  |       0.89 |
| LPL      |     116 |             31.83 |              26.11 |       0.83 |
| CBLOL    |      30 |             32.43 |              26.67 |       0.83 |
| TCL      |      30 |             34.52 |              28    |       0.82 |
| LEC      |      45 |             33.02 |              26.27 |       0.8  |
| LJL      |       4 |             31.78 |              25.25 |       0.79 |
| LLA      |      16 |             32.79 |              24.94 |       0.75 |
| LCS      |      49 |             32.55 |              23.53 |       0.73 |
| LCK      |      87 |             35.25 |              22.64 |       0.66 |

## Assessment of Missingness

My best MNAR candidate is the ban columns, `ban1` through `ban5`. A ban slot is blank when no champion was banned there, and in pro play skipping a ban is usually a deliberate team decision, occasionally a draft error. The chance a value is missing depends on the team's own decision process, the thing that would have generated the value, not on anything else recorded in the dataset. That is MNAR. Draft-phase logs recording whether a ban was skipped on purpose, timed out, or never got captured would explain the missingness and turn it into MAR.

For missingness dependency I test `golddiffat15`, the gold lead at 15 minutes, missing for 8.1% of team rows. The statistic is the total variation distance (TVD), with 5,000 permutations at the 0.05 level.

Missingness depends on league: TVD = 1.000, p = 0.0000, reject the null. Rows without 15-minute stats come almost entirely from leagues whose games are marked `partial`, mostly the LPL and its academy league the LDL, which don't publish timeline data. Missingness explained by an observed column is MAR.

<iframe src="assets/missingness-league.html" width="100%" height="500" frameborder="0"></iframe>

Missingness does not depend on side: TVD = 0.0002, p = 1.00, fail to reject. Timeline coverage is a property of the event, both of a game's team rows share it, and every game has one Blue and one Red row, so missing rows split evenly between sides.

<iframe src="assets/missingness-side-null.html" width="100%" height="500" frameborder="0"></iframe>

## Hypothesis Testing

Question: are LPL games bloodier than LCK games?

Null hypothesis: LPL and LCK games come from the same distribution of combined kills per minute, and any difference in observed means is due to random chance.

Alternative hypothesis: LPL games have higher combined kills per minute than LCK games.

The test is a permutation test, since there are two observed samples and no known population: 10,000 shuffles of the league labels across games, at the 0.05 significance level. The test statistic is the difference in mean kills per minute, LPL minus LCK. Signed, not absolute, because the alternative points one way.

Result: across 116 LPL and 87 LCK games, the observed gap is 0.177 kills per minute in the LPL's favor, with p < 0.0001. I reject the null. The data back the stereotype: LPL games produce more kills per minute than LCK games. League isn't randomly assigned, though, so the test doesn't say why; team parity, drafts, and playstyle norms are all tangled up in the league label.

<iframe src="assets/hypothesis-null.html" width="100%" height="500" frameborder="0"></iframe>

## Framing a Prediction Problem

I predict whether a team wins the game (the `result` column): binary classification. The response variable is the outcome everyone in the scene cares about, and it extends the tempo question, because if 15-minute leads predict winners well, the early game decides matches. Features are restricted to what is knowable at the 15-minute mark: `golddiffat15`, `xpdiffat15`, `csdiffat15`, `killsat15`, `deathsat15`, `firstblood`, `firstdragon`, `side`, and `league`. Anything decided later, like total kills, first baron, or game length, is excluded because it would leak the answer. The metric is accuracy: each game produces exactly one winner and one loser, so the classes are perfectly balanced and accuracy reads directly as the share of games called correctly, with no asymmetric cost that would favor precision or recall. I report F1 as a check. Modeling uses the 2,779 team rows from games with complete timeline data.

## Baseline Model

The baseline is a logistic regression in a single sklearn `Pipeline` with two features from the original data: `golddiffat15` (quantitative, used as-is) and `side` (nominal, one-hot encoded). Data are split 75/25 into training and test sets, and evaluation happens on the held-out test set. It scores 0.734 train accuracy and 0.728 test accuracy (F1 = 0.727). Guessing one class blindly gets 0.500, so calling about 73% of games from a gold number and a side label is a real result: pro games snowball. Train and test scores sit close together, so it barely overfits. Decent but incomplete, since it ignores kills, experience, and objectives, which any analyst would consult.

## Final Model

I added four things on top of the baseline features, each with a reason from the game itself. `kill_diff_15 = killsat15 - deathsat15`, because kills carry momentum value that raw gold understates, like shutdown bounties and free objectives while the enemy is down players. Standardized `xpdiffat15` and `csdiffat15`, because experience gates levels and ultimates and CS measures lane control, with standardization putting gold (thousands), XP (thousands), and CS (tens) on one scale so regularization treats them evenly. One-hot encoded `league`, because the same lead converts to wins at different rates across regions. And `firstdragon`, an early map-control signal the scalar diffs don't capture. I compared three algorithms (logistic regression, random forest, gradient boosting) in identical pipelines, tuning hyperparameters stated in advance (`C` for logistic regression; tree depth, tree count, and learning rate for the ensembles) with `GridSearchCV` and 5-fold cross-validation on the training set only.

The best cross-validated model was gradient boosting (learning_rate: 0.1, max_depth: 3), CV accuracy 0.7419. On the same held-out test set as the baseline it scores 0.731 accuracy (F1 = 0.725) against the baseline's 0.728, a gain of +0.003. Small, and the smallness is the finding: the 15-minute gold lead already carries most of what there is to know about who wins, and the extra features mostly help with games near the margin.

## Fairness Analysis

Question: does the final model perform worse for Red-side teams (Group X) than for Blue-side teams (Group Y)? Blue side wins more often in pro play, so a model could plausibly make lower-quality win calls for Red. The evaluation metric is precision: when the model predicts a win, how often is it right?

Null hypothesis: the model is fair; precision for Blue-side and Red-side teams is roughly the same, and any difference is due to chance.

Alternative hypothesis: the model is unfair; precision for Blue-side teams is higher than for Red-side teams.

The test statistic is precision(Blue) minus precision(Red), with 10,000 permutations of the side labels within the test set at the 0.05 level, using the fixed final model.

Result: precision is 0.721 for Blue and 0.691 for Red, a gap of 0.030, with p = 0.2662. I fail to reject the null: no evidence the model's win calls are less trustworthy for Red-side teams. That doesn't certify the model as fair, but this particular test found nothing.

<iframe src="assets/fairness-null.html" width="100%" height="500" frameborder="0"></iframe>
