# League of Legends CS vs. Kills Analysis (2022 Esports Data)

## Step 1: Introduction

The dataset comes from the game League of Legends. This dataset is from the year 2022 and contains 150,588 rows of data of 12,549 unique matches.

To provide some context, in matches items are bought by players and items cost gold. There are 2 main ways to earn gold:

1. Kill enemies
2. Kill minions (Creep Score or CS)

There is a third way which is to break down structures but this is trivial and not considered.

Items make players more stronger, so as you get more gold, you have a larger opportunity to get stronger. However I always get into a dispute with my friends about if the creepscore (killing minions) is a good proxy for the gold. I primarily focus on the kills. This is actually a real debate I have with my hometown friends in Korea when I say look I have 10 kills and they say you only have 3 cs per minute(this is bad apparently). They argue good cs means good at being able to kill and I just stole other peoples kills. So I've taken it upon myself to analyze if higher creepscore is truly correlated with higher ability to be a threat to kill the enemy team.

"The central question of this project is: Is higher creep score (CS) a stronger predictor of winning than kills at 15 minutes?"

Each match has 10 players, with 5 players on each team.

Here are the columns that I will concern myself with:

### Early Game Performance Metrics

**Creep Score (CS) Metrics**

- `csat10`: The creep score (number of minions/monsters killed) by 10 minutes. Reflects early farming efficiency in the early game otherwise known as the laning phase.
- `csat15`: The creep score by 15 minutes. Shows whether early CS advantages are maintained or improved as the game progresses.
- `csat20`: The creep score at 20 minutes. Provides insight into sustained farming performance beyond the initial phase.
- `opp_csat10`: The opponent's creep score at 10 minutes. Allows calculation of the CS differential. Same for the following columns.
- `opp_csat15`: The opponent's creep score at 15 minutes.
- `opp_csat20`: The opponent's creep score at 20 minutes.

**Kill Metrics**

- `killsat10`: Number of kills secured by the team within the first 10 minutes.
- `killsat15`: Number of kills at 15 minutes. Helps compare how kill pressure evolves alongside CS.
- `killsat20`: Number of kills at 20 minutes.
- `opp_killsat10`: Opponent's kills at 10 minutes, used to calculate the kill differential.
- `opp_killsat15`: Opponent's kills at 15 minutes.
- `opp_killsat20`: Opponent's kills at 20 minutes.

"While I preserved data from 10, 15, and 20 minutes to enable comprehensive analysis, I focus on the 15-minute mark because it represents the end of the laning phase while still being predictive of the final outcome."

---

## Step 2: Data Cleaning and Imputation

### Explanation of Data Cleaning Steps

1.  **Column Selection**: We specify a list of columns that are directly relevant to the analysis:
    - General identifiers and context: `gameid`, `teamid`, `side`, `patch`, `result`.
    - CS metrics: `csat10`, `csat15`, `csat20` and their opponent counterparts (`opp_csat10`, etc.).
    - Kill metrics: `killsat10`, `killsat15`, `killsat20` and corresponding opponent metrics.
    - Overall metrics: `total cs`, `teamkills`.
    - Position: Included so that we can filter out support players (who don't typically prioritize kills or CS) and help the ADC, which is a position that is considered. We will use the omission of support as a means of imputation.
      This ensures that we only load the data necessary for hypothesis testing.
2.  **Data Type Conversion**: Many columns in the raw data that should be numeric might be read as strings. We convert these columns to numeric using `pd.to_numeric` with `errors='coerce'`, which automatically replaces any non-numeric entries with `NaN`. This is critical for numerical analyses.
3.  **Filtering Rows**: The dataset includes rows for all positions, including aggregated 'team' rows. We filter out 'team' rows initially. We also filter out 'sup' (support) position rows _after_ imputation, as their CS/kill patterns differ significantly. Also I noticed there were an obscene amount of rows where CS was 0, so we will also remove these rows as these likely represent incomplete game records.
4.  **Imputation**: Missing values in key numeric columns (`csat*`, `killsat*`, `total cs`, etc.) are imputed using a cascading median strategy (details below).
5.  **Index Reset and Renaming**: After filtering and imputation, the index is reset for easier handling. The column `'total cs'` is renamed to `'total_cs'` for consistency.

### Head of Cleaned DataFrame

```markdown
| position | result | csat10 | opp_csat10 | csat15 | opp_csat15 | killsat10 | opp_killsat10 | killsat15 | opp_killsat15 | total_cs | teamkills |
| :------- | -----: | -----: | ---------: | -----: | ---------: | --------: | ------------: | --------: | ------------: | -------: | --------: |
| top      |      0 |     89 |         81 |    135 |        121 |         0 |             0 |         0 |             0 |      231 |         9 |
| jng      |      0 |     58 |         63 |     89 |        100 |         1 |             0 |         2 |             0 |      148 |         9 |
| mid      |      0 |     81 |         81 |    120 |        119 |         0 |             0 |         0 |             3 |      193 |         9 |
| bot      |      0 |     78 |         90 |    115 |        149 |         1 |             0 |         2 |             3 |      226 |         9 |
| top      |      1 |     81 |         89 |    121 |        135 |         0 |             0 |         0 |             0 |      229 |        19 |
```

### Univariate Analysis

Here we look at the distributions of key team-level metrics: Total CS and Total Kills, separated by match outcome (Win/Loss).

**Distribution of Team-Level Total CS by Outcome**

<iframe
 src="assets/team_cs_distribution.html"
 width="800"
 height="600"
 frameborder="0"
 ></iframe>

_Interpretation:_ This histogram shows that CS distributions form a relatively normal curve centered around 900-1000 total CS per team. Winning teams (green) generally have higher CS than losing teams (red). Wow, I guess I might be on the path towards being wrong. The box plots at the top confirm that the median CS for winning teams is clearly higher than for losing teams. There appears to be a CS advantage associated with winning, supporting the idea that farming efficiency correlates with success. Ouch.

**Distribution of Team-Level Total Kills by Outcome**

<iframe
 src="assets/team_kills_distribution.html"
 width="800"
 height="600"
 frameborder="0"
 ></iframe>

_Interpretation:_ This histogram shows the distribution of total team kills. Similar to CS, winning teams (green) tend to have higher kill counts than losing teams (red), although the distributions overlap significantly. The median number of kills is higher for winning teams, suggesting that securing more kills is also associated with winning.

### Bivariate Analysis

This plot examines the relationship between a player's role (position), their CS at 15 minutes, and the match outcome. Note: the reason why I will be looking at the 15 minute mark for the following graphs is because 15 minutes usually marks the end of the laning phase where most CS is collected. After 15 minutes players roam to other parts of the map and get into team fights and whatnot.

**CS at 15 Minutes by Position and Outcome**

<iframe
 src="assets/cs_by_position_outcome.html"
 width="800"
 height="600"
 frameborder="0"
 ></iframe>

_Interpretation:_ This box plot reveals that Mid and Top laners generally achieve the highest CS by 15 minutes, followed closely by Bot lane ADCs. Junglers have significantly lower CS, as expected. Within each role, winning players (green) tend to have a higher median CS than losing players (red). The difference appears most pronounced in the Top lane, suggesting that a CS advantage in this lane might be particularly impactful for securing a win. This aligns with competitive LoL dynamics where lane dominance through CS creates pressure.

### Interesting Aggregates

This combined chart shows the relationship between CS brackets at 15 minutes and the overall win rate for players within those brackets.

**Win Rate by CS at 15 Minutes Brackets**

<iframe
 src="assets/win_rate_by_cs_bracket.html"
 width="800"
 height="600"
 frameborder="0"
 ></iframe>

_Interpretation:_ This chart strongly demonstrates a positive correlation between CS at 15 minutes and win rate. The win rate climbs steadily from around 50% in the lower CS brackets to nearly 70% in the highest bracket. Most games fall into the middle brackets (e.g., 115-153 CS), but achieving exceptionally high CS significantly increases the likelihood of winning. This highlights that strong early-game farming is a robust predictor of match success in this dataset. But I will address that the small amount of sample size in the beginning of the graph signifying a high win rate respite having low CS could indicate that there are a decent amount of games despite having low CS there is usually another carry on the team, and doing this bad at 15 minutes could mean there is an x factor on your own team. But the trend for 38.4 cs + remains true.

**Average CS at Different Timestamps by Position and Outcome**

```markdown
| position | csat10_0 | csat10_1 | csat15_0 | csat15_1 | csat20_0 | csat20_1 | cs_diff_10 | cs_diff_15 | cs_diff_20 |
| :------- | -------: | -------: | -------: | -------: | -------: | -------: | ---------: | ---------: | ---------: |
| bot      |  77.6326 |  79.7207 |  124.943 |  128.997 |  171.942 |  178.213 |    2.08813 |    4.05387 |    6.27169 |
| jng      |  62.0141 |   63.216 |   93.571 |  96.0065 |  122.054 |  126.875 |    1.20197 |    2.43545 |    4.82102 |
| mid      |  83.3052 |  85.1219 |  134.034 |  137.689 |  177.165 |  182.729 |    1.81672 |    3.65451 |    5.56431 |
| top      |  75.6975 |  77.8944 |  122.852 |  126.971 |  165.039 |  171.271 |    2.19683 |    4.11893 |    6.23265 |
```

_Interpretation:_ This table provides the average CS values at 10, 15, and 20 minutes for each position, split by whether the team won (1) or lost (0). The `cs_diff` columns show the average CS advantage for winning players in each role at each time point. This numerically reinforces the observations from the box plot, showing consistent CS leads for winners across most roles and time points.

### Imputation

A significant portion of the data had missing values, particularly for the CS and Kill metrics at specific timestamps.

**Missing Data Summary:**

- Total rows: 150,588
- Rows with at least one missing value in key columns: 45,933 (~30.5%)
- Missing percentages for `csat*` / `killsat*` columns: ~15-16%

Dropping these rows would discard too much information. Therefore, imputation was necessary.

**Imputation Strategy:**

At first I did median imputation but because such a large amount of rows were missing data the graph looked really awkward as the median value frequency count was so so high. So with my background knowledge in League of Legends, I decided to use a different impute strategy where I get the median by role and team average. Because some teams like to play aggressive and some play passively. Furthermore ADC's typically have more kills than other roles so I thought this would be a good strategy.

A more nuanced, cascaded median imputation strategy was adopted:

1.  For a missing value, first try imputing using the median for that specific `position` and `champion` combination.
2.  If that combination isn't available or its median is NaN, fall back to the median for the player's `position`.
3.  If that fails, use the median for the player's `teamid`.
4.  As a final fallback, use the overall median for the column across all non-support players.

This strategy leverages domain knowledge (different roles/champions/teams have different typical stats) to provide more realistic imputed values than a simple overall median. The reason why I chose the position + champion combination is because some positions have an opportunity to get more CS than others, and some champions have better farming mechanics(i.e it's easier to kill minons with their abilities) than others!

**Imputation Quality Report**

<!--
INSTRUCTIONS:
1. In your Python script/notebook, after creating `imputation_report`, add:
   `print(imputation_report.to_markdown(index=False))`
2. Copy the Markdown table output from the console.
3. Paste the copied Markdown table below this comment block.
-->

```markdown
| column        | missing_values | original_mean | original_median | imputed_mean | imputed_median | percent_change |
| :------------ | -------------: | ------------: | --------------: | -----------: | -------------: | -------------: |
| total_cs      |          25098 |       198.968 |             214 |      239.868 |            237 |        20.5561 |
| csat10        |          22716 |        104.83 |              77 |      75.5753 |             78 |        27.9071 |
| csat15        |          22716 |       167.489 |             124 |      120.633 |            126 |        27.9758 |
| csat20        |          23172 |       224.578 |             167 |      161.911 |            169 |        27.9045 |
| opp_csat10    |          22716 |        104.83 |              77 |      75.5935 |             77 |        27.8898 |
| opp_csat15    |          22716 |       167.489 |             124 |      120.677 |            125 |        27.9493 |
| opp_csat20    |          23172 |       224.578 |             167 |      161.948 |            169 |        27.8879 |
| killsat10     |          22716 |       0.72826 |               0 |      0.42483 |              0 |        41.6651 |
| killsat15     |          22716 |       1.35872 |               1 |       0.8852 |              1 |        34.8504 |
| killsat20     |          23172 |       2.15704 |               1 |      1.43643 |              1 |        33.4075 |
| opp_killsat10 |          22716 |       0.72826 |               0 |     0.417548 |              0 |        42.6649 |
| opp_killsat15 |          22716 |       1.35872 |               1 |     0.883681 |              1 |        34.9622 |
| opp_killsat20 |          23172 |       2.15704 |               1 |       1.4194 |              1 |        34.1972 |
```

_Interpretation:_ This table compares the mean and median of key columns before and after imputation (calculated on the non-missing original values vs. the fully imputed column). The goal is for the imputed statistics to be reasonably close to the original ones, indicating the imputation hasn't drastically changed the central tendency of the data. Small percentage changes suggest the imputation was successful in preserving the overall data characteristics.

**Visualizing Imputation Effect (Example: CS at 15 Minutes)**

<!--
INSTRUCTIONS:
Option 1 (Interactive Plot - Recommended if possible):
1. Modify your `visualize_imputation_effect` function (or create a new one) to generate interactive Plotly histograms comparing distributions before (using `df_original[col].dropna()`) and after imputation (`df_cleaned[col]`).
2. Save these plots to HTML files in the `assets` folder (e.g., `assets/imputation_csat10_compare.html`).
3. Embed using the iframe structure below. Repeat for other key columns if desired.

Option 2 (Static PNG Image):
1. Ensure your current `visualize_imputation_effect` saves `imputation_effect.png` to the `assets` folder.
2. Use the standard Markdown image tag: `![Imputation Effect](assets/imputation_effect.png)` instead of the iframe.
-->

<iframe
 src="assets/imputation_csat10_compare.html"
 width="800"
 height="400"
 frameborder="0"
 ></iframe>

_Interpretation:_ Comparing the distribution of `csat10` before imputation (orange, excluding NaNs) and after imputation (blue) shows how the imputation strategy filled the gaps. Ideally, the shape of the blue histogram should look like a "filled-in" version of the orange one, without introducing extreme artificial peaks, demonstrating that the imputation maintained the original data's general distribution. Getting rid of these artificial peaks at the bottom and the top help us better predict, as these are not a typically representative of a normal League of Legends game. I don't think blowouts are good to consider here.

## Step 3: Framing a Prediction Problem

**Prediction Problem:** Can we predict the final outcome (win or loss) of a League of Legends match based _only_ on performance metrics measured at the 15-minute mark?

**Motivation:** This problem aims to test the hypothesis that the early-to-mid game phase (specifically, the state at 15 minutes) is significantly predictive of the final match result. It explores whether early advantages in creep score (CS), kills, gold, or experience translate reliably into victories, addressing the common debate about the importance of the "laning phase" versus later team fights.

**Prediction Type:** This is a **Binary Classification** problem.

**Response Variable:**

- `result`: A binary variable indicating whether the team won (`1`) or lost (`0`) the match.
- _Justification:_ This variable directly represents the match outcome we want to predict.

**Evaluation Metric:**

- **F1-Score:** We will primarily use the F1-score to evaluate our models.
- _Justification:_ The F1-score provides a balance between Precision (the proportion of predicted wins that were actual wins) and Recall (the proportion of actual wins that were correctly predicted). This is useful because we care about both accurately identifying winning scenarios and not misclassifying losses as wins (and vice-versa). It's also generally more robust to potential class imbalance (where wins might be slightly more or less frequent than losses) than simple accuracy. We choose it over accuracy because F1 also balances the csot of false-wins and false-losses, so we won't sacrifice catching true wins just to boost overall percent correct.

I chose f1 over Auroc as well because auroc is good for understanding separability before pickinga deciison threshold, but doesn't tell us how our final win loss cutoff actually performs. Since the end goal is a concrete classifier, we need a metric that reflects performance at the cutoff, and this is exactly what F1 does. I will still include auroc and accuracy just to show overall performance of the model however.

**Features at Prediction Time:**

- We will only use features derived from data available _at_ the 15-minute mark (e.g., `csat15`, `opp_csat15`, `killsat15`, `opp_killsat15`, `side`). The final match `result` is unknown at this point, making it a valid prediction target based on these early-game indicators.

---

## Step 4: Baseline Model

To establish a performance baseline, we start with a simple Logistic Regression model.

**Model Description:**

- An `sklearn` Pipeline combining preprocessing and a Logistic Regression classifier.

**Features Used:**

- The baseline model uses **two** features known at 15 minutes:
  1.  `cs_diff_15`: **Quantitative**. Represents the difference between a player's CS and their direct opponent's CS at 15 minutes (`csat15 - opp_csat15`). This feature was left as-is (no scaling applied in this baseline).
  2.  `side`: **Nominal**. Represents the side the player's team played on ('Blue' or 'Red'). This was encoded using `OneHotEncoder`, dropping one category to prevent multicollinearity. This results in one binary feature (e.g., `side_Red`).
  - In total, the model uses **1 quantitative feature** and **1 nominal feature** (0 ordinal features).
  - To those not familiar with league of legends, this actually kind of matters because of the way the map is set up red side can have less vision than blue at certain spots in the map.

**Performance:**

- The model was trained on 80% of the (non-support) player data and evaluated on the remaining 20%.
- **Accuracy:** 0.5817
- **F1-Score (Weighted Avg):** 0.5805
- **ROC AUC:** 0.6141

- **Classification Report:**

  ```text
               precision    recall  f1-score   support

            0       0.58      0.58      0.58     10043
            1       0.58      0.58      0.58     10036

     accuracy                           0.58     20079
    macro avg       0.58      0.58      0.58     20079
  weighted avg       0.58      0.58      0.58     20079
  ```

- **Feature Importance:** The coefficients from the Logistic Regression indicate the importance and direction of influence for each feature. A higher absolute coefficient suggests stronger influence.

  | Feature    | Coefficient | Abs_Coefficient |
  | :--------- | ----------: | --------------: |
  | side_Red   |   -0.201498 |        0.201498 |
  | cs_diff_15 |    0.015363 |        0.015363 |

  _(Interpretation: The model gives slightly more weight (based on absolute coefficient) to the `side` played than the `cs_diff_15`. A positive coefficient for `cs_diff_15` suggests a higher CS difference increases the log-odds of winning. A negative coefficient for `side_Red` suggests playing on the Red side decreases the log-odds of winning compared to the Blue side, according to this model.)_

**Assessment:**

- The baseline model achieves an F1-score of `0.5805` and an ROC AUC of `0.6141`. Given that a random guessing model would achieve an F1/AUC around 0.5, this baseline performance indicates that even just the CS difference at 15 minutes and the side played on have _some_ predictive power regarding the match outcome.
- The feature importance shows that both `cs_diff_15` and `side` contribute to the prediction, with `side` having a slightly larger coefficient magnitude.
- However, the overall performance (`0.5805` F1-score) is quite low, indicating the model is only slightly better than random guessing. I guess the positive correlation between cs and winning demonstrated by figure 1.4 indicates late-game cs beyond the laning phase also have influence as well...There is significant room for improvement by incorporating more relevant features (like kill differences, gold differences, etc.) which will be explored in Step 5.

---

## Step 5: Final Model

Building upon the baseline, the final model aims to improve prediction accuracy by incorporating more sophisticated features and optimizing model parameters.

**Feature Engineering:**

In addition to the baseline features (`cs_diff_15`, `side`), the final model includes:

- **Original Difference Metrics:**

  - `kills_diff_15`: Difference in kills at 15 minutes. This is what I believe contributes to direct combat success.
  - `gold_diff_15`: Difference in gold earned at 15 minutes. Reflects overall economic advantage.
  - `xp_diff_15`: Difference in experience points at 15 minutes. Indicates champion level and power disparities.

  I reimpute to get the above mentioned metrics.

- **Newly Engineered Features:** To capture more nuanced game dynamics, two interaction/ratio features were created:
  1.  `gold_efficiency`: Calculated as `gold_diff_15 / (abs(cs_diff_15) + 10)`.
      - _Justification:_ This feature aims to measure how effectively a player translates their farming advantage (or lack thereof) into a _gold_ advantage. A high ratio might indicate a player securing gold through objectives, kills, or efficient trading beyond just CS, reflecting strong map play or combat effectiveness relative to their farming. The `+ 10` avoids division by zero and dampens the ratio for very small CS differences.
  2.  `gold_xp_interaction`: Calculated as `gold_diff_15 * xp_diff_15`.
      - _Justification:_ In League of Legends, gold (items) and experience (levels/skills) advantages often work synergistically. This interaction term captures that combined effect. A large positive value signifies a strong dual advantage (high gold _and_ high XP diff), while a large negative value indicates a severe deficit in both. This should provide a stronger signal of dominance or disadvantage than either metric alone.

These features were chosen because they represent key aspects of early-game performance (combat, economy, levels) and interactions between them that are commonly understood to influence match outcomes in League of Legends.

**Modeling Algorithm and Hyperparameter Tuning:**

- **Algorithm:** Logistic Regression was retained for the final model due to its suitability for binary classification and the interpretability of its coefficients.
- **Hyperparameter Tuning Method:** `GridSearchCV` with 5-fold cross-validation was used to systematically search for the best model configuration based on f1 (the chosen `scoring` metric for `GridSearchCV`). Also I binarized predictions at the default 0.5 probability cutoff to compute F1. Furthermore after CV I refit the best estimator on the full training split before evaluating on the test set.””
- **Parameters Tuned:**
  - `C` (Inverse of regularization strength): `[0.01, 0.1, 1.0, 10.0, 100.0]`
  - `penalty` (Regularization type): `['l1', 'l2']`
  - `solver` (Optimization algorithm): `['liblinear', 'saga']`
- **Best Hyperparameters Found:**
  - `C`: `1.0`
  - `penalty`: `'l1'`
  - `solver`: `'saga'`
    _(Interpretation: The optimal C value of `1.0` suggests a moderate level of regularization was optimal. The L1 penalty ('lasso') was selected, which can sometimes perform feature selection by shrinking some coefficients exactly to zero, although that didn't happen significantly here.)_
- **Feature Transformation:** A `ColumnTransformer` was used within the pipeline:
  - Original numeric features (`cs_diff_15`, `kills_diff_15`, etc.) were scaled using `StandardScaler`.
  - Engineered numeric features (`gold_efficiency`, `gold_xp_interaction`) were transformed using `QuantileTransformer` (outputting to a normal distribution) to handle potential skewness or outliers.
  - The categorical `side` feature was processed using `OneHotEncoder` (dropping the first category).

**Final Model Performance:**

- **Accuracy:** 0.6272
- **F1-Score (Weighted Avg):** 0.6272 _(Note: Macro avg F1 is 0.6272, value for class 1 is 0.6274)_
- **ROC AUC:** 0.6870

- **Classification Report:**

  ```text
               precision    recall  f1-score   support

            0       0.63      0.63      0.63     12535
            1       0.63      0.63      0.63     12563

     accuracy                           0.63     25098
    macro avg       0.63      0.63      0.63     25098
  weighted avg       0.63      0.63      0.63     25098
  ```

- **Feature Importance:**
  | Feature | Coefficient | Abs_Coefficient |
  |:----------------------|--------------:|------------------:|
  | gold_diff_15 | 0.703971 | 0.703971 |
  | cs_diff_15 | -0.308589 | 0.308589 |
  | xp_diff_15 | 0.235688 | 0.235688 |
  | side_Red | -0.154401 | 0.154401 |
  | gold_efficiency | 0.125449 | 0.125449 |
  | kills_diff_15 | -0.098009 | 0.098009 |
  | gold_xp_interaction | 0.005903 | 0.005903 |

  *(Interpretation: Based on the coefficients' absolute values, the most influential features in predicting a win at 15 minutes were `gold_diff_15`, `cs_diff_15`, and `xp_diff_15`. The positive coefficient for `gold_diff_15` strongly aligns with game knowledge – having more gold directly translates to better items and thus a higher chance of winning. Interestingly, `cs_diff_15` has a negative coefficient, suggesting that *after accounting for gold and XP differences*, a higher CS difference alone might slightly decrease the odds of winning in this model's view (perhaps capturing scenarios where a player focuses too much on CS without converting it effectively, though this requires careful interpretation). The `xp_diff_15` also positively contributes as expected. `side_Red` maintains its negative coefficient, and the engineered features contribute, but less strongly than the primary difference metrics.)*

**Comparison to Baseline:**

The final model demonstrated a clear improvement over the baseline:

| Metric         | Baseline Model | Final Model | Improvement |
| :------------- | -------------: | ----------: | ----------: |
| Accuracy       |         0.5817 |      0.6272 |     +0.0455 |
| F1 Score (Wgt) |         0.5805 |      0.6272 |     +0.0467 |
| ROC AUC        |         0.6141 |      0.6870 |     +0.0729 |

- _Discussion:_ The increase in F1-score (from `0.5805` to `0.6272`) and ROC AUC (from `0.6141` to `0.6870`) shows that incorporating the additional difference features (kills, gold, XP) and the engineered features, along with hyperparameter tuning, allowed the model to capture the game state at 15 minutes more effectively than the simple two-feature baseline. The improvement, while modest, highlights the predictive value of a more comprehensive view of early-game performance, particularly the economic (`gold_diff_15`) and level (`xp_diff_15`) advantages.

**Conclusion:**

The final Logistic Regression model, enhanced with additional and engineered features reflecting key game interactions and tuned hyperparameters, outperforms the baseline model in predicting match outcomes based on the 15-minute mark. While still far from perfect (accuracy ~63%), the model achieves an F1-score of `0.6272` and ROC AUC of `0.6870`, demonstrating that early-game advantages, particularly gold difference, are statistically significant indicators of eventual success. With my previous of League of Legends, one thing that may contribute early laning phase being a modest predictor for winning is the fact that these are data from pro games. And in pro games there are a lot of upsets in the late stages because pro's know how to capitalize on mistakes much more. In casual games there is more snowballing meaning that if someone has a big advantage early on, inexprerienced players have low probability of coming back.

The feature importance aligns with game knowledge, emphasizing the impact of gold and experience differentials. The negative coefficient for CS difference warrants further investigation but might suggest complexities beyond simple farm counts when other advantages are considered.
