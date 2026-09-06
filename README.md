# The Rise of Protein Obsession

## Did protein-focused recipes earn better ratings?

*Leo Wong*

The rise of the “protein crave” had become increasingly apparent. Companies had begun adding protein to nearly everything, including protein ice cream, protein chips, and even protein-infused water. This positive traction raised an interesting question: if food companies could market ice cream and chips as healthier alternatives because they contained protein, how did that trend relate to complete meals? I therefore asked: did a recipe being protein-focused affect its rating?

---

## Introduction

I analyzed Food.com recipes and user interactions. The recipes file contained 83,782 recipes, and the interactions file contained 731,927 user-recipe interactions. The central question was: did recipes whose title or tags mentioned protein, while excluding `low-protein`, receive different average ratings than other recipes? This question was useful because popularity claims around protein were easy to make, while community feedback allowed me to test whether a measurable rating difference accompanied the label.

Each row of the cleaned analysis data represented one recipe. I merged the interaction data into the recipe data to obtain each recipe's mean nonzero user rating.

| Column | Meaning |
|---|---|
| `id` | Unique recipe identifier used to join ratings to recipes |
| `rating` | Mean of the recipe's nonzero user ratings (1–5) |
| `name`, `tags` | Recipe text used to construct the protein-focus indicator |
| `protein`, `calories` | Nutrition values supplied with a recipe |
| `n_steps`, `n_ingredients`, `minutes` | Recipe complexity and preparation metadata |
| `is_protein_focused` | Whether `protein` occurs in the name or tags, excluding `low-protein` |

## Data Cleaning and Exploratory Data Analysis

I left-merged recipes with interactions (`id` ↔ `recipe_id`) so recipes without usable feedback remained identifiable. Food.com used a rating of 0 for an interaction where no rating was supplied; because the rating scale itself was 1–5, I replaced these zeros with missing values before computing each recipe's mean rating. This prevented a missing rating from being treated as an extremely negative review.

The original `nutrition` column was a string representation of a seven-item list. I split it into numeric `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `saturated_fat`, and `carbohydrates` columns. I also converted `submitted` to datetime. Finally, I created `is_protein_focused`: it was true when *protein* appeared in the recipe name or tags, except when the mention was *low-protein*. That exclusion mattered because a low-protein dietary tag was conceptually the opposite of a protein-forward recipe.

This was a compact view of the cleaned recipe-level data.

| id | minutes | n_steps | n_ingredients | rating | calories | protein | is_protein_focused |
|---:|---:|---:|---:|---:|---:|---:|:---|
| 333281 | 40 | 10 | 9 | 4.000 | 138.4 | 3.0 | False |
| 453467 | 45 | 12 | 11 | 5.000 | 595.1 | 13.0 | False |
| 306168 | 40 | 6 | 9 | 5.000 | 194.8 | 22.0 | False |
| 286009 | 120 | 7 | 7 | 5.000 | 878.3 | 20.0 | False |
| 475785 | 90 | 17 | 13 | 5.000 | 267.0 | 29.0 | False |

### Rating distribution

<iframe src="assets/rating-distribution.html" width="100%" height="520" frameborder="0" title="Interactive histogram of average recipe ratings"></iframe>

Average ratings were concentrated near the top of the 1–5 scale, especially at 5. This ceiling-heavy distribution meant that even a real difference between recipe groups could have been small in raw rating points.

### Protein and calories

<iframe src="assets/protein-calories.html" width="100%" height="560" frameborder="0" title="Interactive scatter plot of protein and calories"></iframe>

Among recipes below 1,500 calories and 120 g of protein, protein tended to rise with calories. Protein-focused recipes appeared more often in the higher-protein region, but the substantial overlap showed that the text/tag indicator was not a substitute for the numeric nutrition label.

### Grouped recipe profile

| Recipe group | Mean rating | Mean calories | Mean protein (g) | Mean ingredients |
|---|---:|---:|---:|---:|
| Other recipes | 4.625 | 427.766 | 31.439 | 9.266 |
| Protein-focused | 4.629 | 498.352 | 86.783 | 7.559 |

Protein-focused recipes contained much more listed protein and somewhat more calories, yet their mean rating was almost identical to that of other recipes. They also used fewer ingredients on average, suggesting that protein-oriented dishes in this collection were not necessarily more ingredient-heavy.

## Assessment of Missingness

There were 2,609 recipes without a usable average rating after zero ratings were treated as missing. I believed `rating` may have been MNAR: users' willingness to leave a rating could plausibly have depended on their unobserved opinion of the recipe. For example, an indifferent user may have been less motivated to rate at all. Review-prompt exposure, whether a user cooked the recipe, and user-level rating habits would have been valuable additional data; conditioning on such variables could have helped explain the missingness and made an MAR explanation more plausible.

For dependency testing, I used the absolute difference in means between recipes whose rating was missing and present, then compared it with 1,000 permutations of the missingness indicator.

<iframe src="assets/missingness-permutation.html" width="100%" height="520" frameborder="0" title="Interactive permutation distribution for rating missingness and ingredients"></iframe>

Rating missingness depended on `n_ingredients`: the observed difference in mean ingredient count was 0.2542, with permutation p < 0.001. The observed value lay far beyond the simulated null differences, so at α = 0.05 I rejected independence for these variables.

As a contrast, rating missingness did not show evidence of dependence on `is_protein_focused`. The absolute difference in the proportion of protein-focused recipes was 0.0039, with permutation p = 0.289. At α = 0.05, I failed to reject independence; the small difference was compatible with random assignment of missingness labels.

## Hypothesis Testing

I tested whether protein-forward labeling was associated with a different average rating.

- Null hypothesis: protein-focused recipes and other recipes had the same mean average rating; any observed difference was due to random assignment of the protein-focus labels.
- Alternative hypothesis: protein-focused recipes and other recipes had different mean average ratings.
- Test statistic: absolute difference in group mean ratings.
- Significance level: α = 0.05.

The protein-focused mean was 4.6286 and the other-recipe mean was 4.6253, giving an observed difference of 0.0034. A 1,000-permutation test gave p = 0.783. Because this was much larger than 0.05, I failed to reject the null hypothesis. In this dataset, the tiny observed difference was consistent with random variation; it was not evidence that protein-focused recipes received systematically different average ratings.

## Framing a Prediction Problem

The prediction task was to estimate a recipe's average user `rating`, so this was a regression problem. Ratings were continuous recipe-level averages between 1 and 5, and the target directly captured the outcome discussed in the central question.

I reported mean squared error (MSE) as the primary metric and also reported R². MSE was appropriate because larger misses on a compact 1–5 scale should have counted more heavily than small misses; R² complemented it by measuring performance against predicting the overall mean. At prediction time, a recipe's posted metadata—nutrition, minutes, steps, ingredient count, tags, and name—was available. I excluded later user reviews, interaction dates, rating counts, and the rating itself to avoid leakage.

## Baseline Model

The baseline was a `Pipeline` with `StandardScaler` followed by `LinearRegression`. It used two quantitative features: `n_steps` and `n_ingredients`. Standardizing put the two counts on comparable scales before the linear model estimated their relationship with average rating.

On an 80/20 train/test split (random state 42), the baseline obtained test MSE = 0.4045 and test R² = −0.0004. This was not a useful predictor: its negative R² meant it performed marginally worse than simply predicting the training-set mean rating. Recipe complexity alone contained almost no linear signal for these highly concentrated ratings.

## Final Model

The final `Pipeline` applied a `ColumnTransformer` and a `RandomForestRegressor`. It retained the complexity variables and added the engineered quantitative feature protein per calorie (`protein / (calories + 1)`), which distinguished calorie-dense recipes from recipes whose calories were more protein-dense. It also added the nominal protein-focus indicator, one-hot encoded, to capture a text/tag-based dietary signal that was not identical to the nutrition value. Minutes were restricted to 300 or fewer before modeling to prevent a small number of implausibly long durations from dominating the scale.

I tuned `n_estimators` (50, 100), `max_depth` (5, 10), and `min_samples_split` (5, 10) with 3-fold `GridSearchCV` on the training set using negative MSE. The best model used 100 trees, maximum depth 5, and minimum split size 5. Its held-out performance was MSE = 0.4116 and R² = 0.0034.

The final model captured a tiny amount of variance (R² rose from −0.0004 to 0.0034), but its MSE was not lower than the baseline's; moreover, the final preprocessing removed extreme-duration rows, so the two reported scores were not a perfectly like-for-like comparison. The responsible conclusion was that these pre-submission metadata features provided extremely limited rating predictability, not that the final model was practically strong.

## Fairness Analysis

I assessed whether the final model's errors differed for protein-focused recipes (name/tags included protein, excluding low-protein) and other recipes. The group metric was test-set MSE, and the test statistic was the absolute difference between the groups' MSE values.

- Null hypothesis: the model was fair with respect to these groups; any MSE difference was due to chance.
- Alternative hypothesis: the model was not fair with respect to these groups; the groups had different MSE.
- Significance level: α = 0.05.

The protein-focused group had MSE 0.3856 and the other-recipe group had MSE 0.4124, an absolute difference of 0.0267. A 1,000-permutation test gave p = 0.677. I therefore failed to reject the null hypothesis: this analysis did not provide evidence that the model's squared errors differed by protein-focus group. This was not proof of fairness; it meant the observed gap was plausible under the permutation null with this sample and metric.

---

*Data: Food.com recipes and interactions.*
