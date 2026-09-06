# The Rise of Protein Obsession

## Do protein-focused recipes earn better ratings?

*Leo Wong*

The rise of the “protein crave” had become increasingly apparent. Companies had begun adding protein to nearly everything, including protein ice cream, protein chips, and even protein-infused water. This positive traction raised an interesting question: if food companies could market ice cream and chips as healthier alternatives because they contained protein, how did that trend relate to complete meals? I therefore asked: did a recipe being protein-focused affect its rating?

---

## Introduction

I analyzed Food.com recipes and user interactions. The recipes dataset comprised 83,782 recipes, while the user interaction data had 731,927 user and recipe interactions. My hypothesis was: do recipes whose titles or tags include protein but exclude low-protein recipes have higher average ratings compared to other recipes? This question was helpful to explore because claims about the popularity of protein are easy to find, while the user reviews gave room to see if there was any real rating difference.

Each record in the cleaned data for analysis represents one recipe. To calculate the average rating of each recipe, the interaction data was merged into the recipes data.

| Column | Meaning |
|---|---|
| id | Unique recipe identifier used to join ratings to recipes |
| rating | Mean of the recipe's nonzero user ratings (1 to 5) |
| name, tags | Recipe text used to construct the protein-focus indicator |
| protein, calories | Nutrition values supplied with a recipe |
| n_steps, n_ingredients, minutes | Recipe complexity and preparation metadata |
| is_protein_focused | Whether protein occurs in the name or tags, excluding low-protein |

## Data Cleaning and Exploratory Data Analysis

I left-merged with the interactions dataset (id matching recipe_id) so that recipes without ratings were still identifiable. Food.com used a 0 for an interaction that had no rating. Because the actual ratings scale was 1 to 5, I converted these zeros to null values to make sure that a missing value was not mistaken for an especially bad review.

The original nutrition row was stored as a string representation of a list with seven elements. I parsed out these elements into numeric columns for calories, total_fat, sugar, sodium, protein, saturated_fat, and carbohydrates. I also converted the submitted date into a standard datetime format. Lastly, I created a feature called is_protein_focused that was set to true when protein occurred in either the name or tags, but only when it was not low-protein. This distinction was necessary because a low-protein dietary preference is the exact opposite of a protein-focused recipe.

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

Average ratings were concentrated near the top of the 1 to 5 scale, especially at 5. This ceiling-heavy distribution meant that even a real difference between recipe groups could have been small in raw rating points.

### Protein and calories

<iframe src="assets/protein-calories.html" width="100%" height="560" frameborder="0" title="Interactive scatter plot of protein and calories"></iframe>

Among recipes below 1,500 calories and 120 g of protein, protein tended to rise with calories. Protein-focused recipes appeared more often in the higher-protein region, but the substantial overlap showed that the text and tag indicator was not a substitute for the numeric nutrition label.

### Grouped recipe profile

| Recipe group | Mean rating | Mean calories | Mean protein (g) | Mean ingredients |
|---|---:|---:|---:|---:|
| Other recipes | 4.625 | 427.766 | 31.439 | 9.266 |
| Protein-focused | 4.629 | 498.352 | 86.783 | 7.559 |

Protein-focused recipes contained much more listed protein and somewhat more calories, yet their mean rating was almost identical to that of other recipes. They also used fewer ingredients on average, suggesting that protein-oriented dishes in this collection were not necessarily more ingredient-heavy.

## Assessment of Missingness

There were 2,609 recipes without useful mean ratings after treating zero ratings as missing. I hypothesized that rating could be MNAR, meaning the willingness of the user to leave a rating could depend on their unobserved perception of the recipe. For instance, user indifference could lead someone not to rate the recipe at all. To better understand this mechanism, it would have been helpful to know whether there was a review prompt, whether the user actually prepared the recipe, and their individual tendencies when rating food.

To test for dependency, I took the absolute difference of means between recipes with missing and non-missing ratings, and I compared it against 1,000 permutations of the missingness indicator.

<iframe src="assets/missingness-permutation.html" width="100%" height="520" frameborder="0" title="Interactive permutation distribution for rating missingness and ingredients"></iframe>

Missingness on the rating depended on the number of ingredients, with an observed difference in the means of 0.2542 and a permutation p-value less than 0.001. Because the observed value was far from the null distribution of differences, I rejected independence between these two variables at a significance level of 0.05.

On the other hand, there was no sign of dependence between missingness on the rating and is_protein_focused. The observed absolute difference in the proportion of protein-focused recipes was 0.0039 with a permutation p-value of 0.289. Therefore, at a significance level of 0.05, I failed to reject independence.

## Hypothesis Testing

I tested whether protein-forward labeling was associated with a different average rating.

- Null hypothesis: protein-focused recipes and other recipes had the same mean average rating, and any observed difference was due to random assignment of the protein-focus labels.
- Alternative hypothesis: protein-focused recipes and other recipes had different mean average ratings.
- Test statistic: absolute difference in group mean ratings.
- Significance level: α = 0.05.

The protein-focused mean was 4.6286 and the other-recipe mean was 4.6253, giving an observed difference of 0.0034. A 1,000-permutation test gave p = 0.783. Because this was much larger than 0.05, I failed to reject the null hypothesis. In this dataset, the tiny observed difference was consistent with random variation. It was not evidence that protein-focused recipes received systematically different average ratings.

## Framing a Prediction Problem

The goal was to predict the average user rating for a recipe, making this a regression problem. The ratings were continuous recipe averages on a scale from 1 to 5, and the target variable represented the exact output of interest in our guiding question.

Mean squared error (MSE) was the evaluation metric chosen alongside R². This choice was justified because larger errors on the tighter 1 to 5 scale should be penalized more heavily than smaller errors, while R² helped assess how well the model performed compared to simply predicting the mean rating. During prediction, only the metadata of the recipe was used, including nutrition facts, minutes, steps, number of ingredients, tags, and name.

## Baseline Model

The baseline model was an sklearn Pipeline that first scaled the predictors using StandardScaler and then trained a LinearRegression model. The two predictors used were n_steps and n_ingredients.

Using an 80/20 train/test split with random state 42, the baseline model yielded an MSE of 0.4045 and an R² of -0.0004 on the test set. This was not a strong predictor, as indicated by the negative R² value, which showed that the model was slightly less accurate than simply predicting the mean rating of the training set.

## Final Model

The final Pipeline used ColumnTransformer and RandomForestRegressor. It retained all complexity attributes and added a new quantitative attribute called protein per calories, calculated as protein divided by calories plus one, to distinguish calorie-dense recipes from protein-dense recipes. I also retained the binary protein-focus indicator, which was one-hot encoded to include a text and tag-based nutrition signal separate from the raw nutritional value. Recipe duration was filtered to 300 minutes or less before fitting.

I optimized n_estimators (50, 100), max_depth (5, 10), and min_samples_split (5, 10) with 3-fold GridSearchCV on the training set using negative MSE. The best model had 100 trees, a maximum depth of 5, and a minimum split size of 5. This optimization produced a test MSE of 0.4116 and an R² of 0.0034.

Although this model explained very little variance (with R² moving from -0.0004 to 0.0034), its MSE was not an improvement over the baseline. Furthermore, the pre-processing used in the final model removed rows with extreme duration values, meaning the metrics cannot be compared identically. The main takeaway is that these submission-time metadata features predict recipe ratings very weakly.

## Fairness Analysis

An analysis was performed to evaluate whether model errors differed for protein recipes (where name or tags contained protein, excluding low protein) versus other recipes. The evaluation metric was test set MSE, and the test statistic was the absolute difference in MSE between the two groups.

- Null hypothesis: the model was fair with respect to these groups, and any MSE difference was due to chance.
- Alternative hypothesis: the model was not fair with respect to these groups, and the groups had different MSE.
- Significance level: α = 0.05.

The MSE for the protein recipe group was 0.3856, while the MSE for the other recipe group was 0.4124, resulting in an absolute difference of 0.0267. A 1,000-iteration permutation test yielded a p-value of 0.677. Because this value is well above 0.05, we fail to reject the null hypothesis, and there is no evidence that the model squared errors differ between protein-focused and standard recipes.

---

*Data: Food.com recipes and interactions.*
