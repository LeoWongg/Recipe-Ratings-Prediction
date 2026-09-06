---
layout: default
---

# The Rise of Protein Obsession

## Do protein-forward recipes earn better ratings?

*Leo Wong · DSC 80, UC San Diego*

Protein is everywhere in contemporary food culture: recipe titles, meal plans, and nutrition labels all treat it as a selling point. This project asks whether that label actually corresponds to a difference in how Food.com users rate a recipe—and whether recipe metadata can predict those ratings at all.

---

## Introduction

I analyze Food.com recipe metadata and user interactions. The recipes file contains **83,782 recipes**, and the interactions file contains **731,927 user-recipe interactions**. The central question is: **do recipes whose title or tags mention protein (while excluding `low-protein`) receive different average ratings than other recipes?** It is useful because popularity claims around protein are easy to make, while community feedback lets us test whether a measurable rating difference accompanies the label.

Each row of the cleaned analysis data represents one recipe. I merge the interaction data into the recipe data to obtain each recipe's mean nonzero user rating.

| Column | Meaning |
|---|---|
| `id` | Unique recipe identifier used to join ratings to recipes |
| `rating` | Mean of the recipe's nonzero user ratings (1–5) |
| `name`, `tags` | Recipe text used to construct the protein-focus indicator |
| `protein`, `calories` | Nutrition values supplied with a recipe |
| `n_steps`, `n_ingredients`, `minutes` | Recipe complexity and preparation metadata |
| `is_protein_focused` | Whether `protein` occurs in the name or tags, excluding `low-protein` |

## Data Cleaning and Exploratory Data Analysis

I left-merge recipes with interactions (`id` ↔ `recipe_id`) so recipes without usable feedback remain identifiable. Food.com uses a rating of **0** for an interaction where no rating was supplied; because the rating scale itself is 1–5, I replace these zeros with missing values before computing each recipe's mean rating. This prevents a missing rating from being treated as an extremely negative review.

The original `nutrition` column is a string representation of a seven-item list. I split it into numeric `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `saturated_fat`, and `carbohydrates` columns. I also convert `submitted` to datetime. Finally, I create `is_protein_focused`: it is true when *protein* appears in the recipe name or tags, except when the mention is *low-protein*. That exclusion matters because a low-protein dietary tag is conceptually the opposite of a protein-forward recipe.

Here is a compact view of the cleaned recipe-level data.

| id | minutes | n_steps | n_ingredients | rating | calories | protein | is_protein_focused |
|---:|---:|---:|---:|---:|---:|---:|:---|
| 333281 | 40 | 10 | 9 | 4.000 | 138.4 | 3.0 | False |
| 453467 | 45 | 12 | 11 | 5.000 | 595.1 | 13.0 | False |
| 306168 | 40 | 6 | 9 | 5.000 | 194.8 | 22.0 | False |
| 286009 | 120 | 7 | 7 | 5.000 | 878.3 | 20.0 | False |
| 475785 | 90 | 17 | 13 | 5.000 | 267.0 | 29.0 | False |

### Rating distribution

<iframe src="assets/rating-distribution.html" width="100%" height="520" frameborder="0" title="Interactive histogram of average recipe ratings"></iframe>

Average ratings are concentrated near the top of the 1–5 scale, especially at 5. This ceiling-heavy distribution means that even a real difference between recipe groups may be small in raw rating points.

### Protein and calories

<iframe src="assets/protein-calories.html" width="100%" height="560" frameborder="0" title="Interactive scatter plot of protein and calories"></iframe>

Among recipes below 1,500 calories and 120 g of protein, protein tends to rise with calories. Protein-focused recipes appear more often in the higher-protein region, but the substantial overlap shows that the text/tag indicator is not a substitute for the numeric nutrition label.

### Grouped recipe profile

| Recipe group | Mean rating | Mean calories | Mean protein (g) | Mean ingredients |
|---|---:|---:|---:|---:|
| Other recipes | 4.625 | 427.766 | 31.439 | 9.266 |
| Protein-focused | 4.629 | 498.352 | 86.783 | 7.559 |

Protein-focused recipes contain much more listed protein and somewhat more calories, yet their mean rating is almost identical to that of other recipes. They also use fewer ingredients on average, suggesting that protein-oriented dishes in this collection are not necessarily more ingredient-heavy.

## Assessment of Missingness

There are **2,609 recipes** without a usable average rating after zero ratings are treated as missing. I believe `rating` may be **MNAR**: users' willingness to leave a rating can plausibly depend on their unobserved opinion of the recipe. For example, an indifferent user may be less motivated to rate at all. Review-prompt exposure, whether a user cooked the recipe, and user-level rating habits would be valuable additional data; conditioning on such variables could help explain the missingness and make an MAR explanation more plausible.

For dependency testing, I use the absolute difference in means between recipes whose rating is missing and present, then compare it with 1,000 permutations of the missingness indicator.

<iframe src="assets/missingness-permutation.html" width="100%" height="520" frameborder="0" title="Interactive permutation distribution for rating missingness and ingredients"></iframe>

Rating missingness **depends on** `n_ingredients`: the observed difference in mean ingredient count is **0.2542**, with permutation **p < 0.001**. The observed value lies far beyond the simulated null differences, so at α = 0.05 I reject independence for these variables.

As a contrast, rating missingness **does not show evidence of dependence on** `is_protein_focused`. The absolute difference in the proportion of protein-focused recipes is **0.0039**, with permutation **p = 0.289**. At α = 0.05, I fail to reject independence; the small difference is compatible with random assignment of missingness labels.

## Hypothesis Testing

I test whether protein-forward labeling is associated with a different average rating.

- **Null hypothesis:** protein-focused recipes and other recipes have the same mean average rating; any observed difference is due to random assignment of the protein-focus labels.
- **Alternative hypothesis:** protein-focused recipes and other recipes have different mean average ratings.
- **Test statistic:** absolute difference in group mean ratings.
- **Significance level:** α = 0.05.

The protein-focused mean is **4.6286** and the other-recipe mean is **4.6253**, giving an observed difference of **0.0034**. A 1,000-permutation test gives **p = 0.783**. Because this is much larger than 0.05, I fail to reject the null hypothesis. In this dataset, the tiny observed difference is consistent with random variation; it is not evidence that protein-focused recipes receive systematically different average ratings.

## Framing a Prediction Problem

The prediction task is to estimate a recipe's average user `rating`, so this is a **regression** problem. Ratings are continuous recipe-level averages between 1 and 5, and the target directly captures the outcome discussed in the central question.

I report **mean squared error (MSE)** as the primary metric and also report R². MSE is appropriate because larger misses on a compact 1–5 scale should count more heavily than small misses; R² complements it by measuring performance against predicting the overall mean. At prediction time, a recipe's posted metadata—nutrition, minutes, steps, ingredient count, tags, and name—is available. I exclude later user reviews, interaction dates, rating counts, and the rating itself to avoid leakage.

## Baseline Model

The baseline is a `Pipeline` with `StandardScaler` followed by `LinearRegression`. It uses two quantitative features: `n_steps` and `n_ingredients`. Standardizing puts the two counts on comparable scales before the linear model estimates their relationship with average rating.

On an 80/20 train/test split (random state 42), the baseline obtains **test MSE = 0.4045** and **test R² = −0.0004**. This is not a useful predictor: its negative R² means it performs marginally worse than simply predicting the training-set mean rating. Recipe complexity alone contains almost no linear signal for these highly concentrated ratings.

## Final Model

The final `Pipeline` applies a `ColumnTransformer` and a `RandomForestRegressor`. It retains the complexity variables and adds the engineered quantitative feature **protein per calorie** (`protein / (calories + 1)`), which distinguishes calorie-dense recipes from recipes whose calories are more protein-dense. It also adds the nominal **protein-focus indicator**, one-hot encoded, to capture a text/tag-based dietary signal that is not identical to the nutrition value. Minutes are restricted to 300 or fewer before modeling to prevent a small number of implausibly long durations from dominating the scale.

I tune `n_estimators` (50, 100), `max_depth` (5, 10), and `min_samples_split` (5, 10) with 3-fold `GridSearchCV` on the training set using negative MSE. The best model uses **100 trees**, **maximum depth 5**, and **minimum split size 5**. Its held-out performance is **MSE = 0.4116** and **R² = 0.0034**.

The final model captures a tiny amount of variance (R² rises from −0.0004 to 0.0034), but its MSE is not lower than the baseline's; moreover, the final preprocessing removes extreme-duration rows, so the two reported scores are not a perfectly like-for-like comparison. The responsible conclusion is that these pre-submission metadata features provide extremely limited rating predictability, not that the final model is practically strong.

## Fairness Analysis

I assess whether the final model's errors differ for **protein-focused recipes** (name/tags include protein, excluding low-protein) and **other recipes**. The group metric is test-set **MSE**, and the test statistic is the absolute difference between the groups' MSE values.

- **Null hypothesis:** the model is fair with respect to these groups; any MSE difference is due to chance.
- **Alternative hypothesis:** the model is not fair with respect to these groups; the groups have different MSE.
- **Significance level:** α = 0.05.

The protein-focused group has MSE **0.3856** and the other-recipe group has MSE **0.4124**, an absolute difference of **0.0267**. A 1,000-permutation test gives **p = 0.677**. I therefore fail to reject the null hypothesis: this analysis does not provide evidence that the model's squared errors differ by protein-focus group. This is not proof of fairness; it means the observed gap is plausible under the permutation null with this sample and metric.

---

*Data: Food.com recipes and interactions. Built as a DSC 80 project at UC San Diego.*
