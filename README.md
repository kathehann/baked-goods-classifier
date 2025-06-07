# Tastes Like Data: Using Nutritional Info to Spot Baked Goods

by Katie Hannigan (khannigan@ucsd.edu) and Alyvia Vaughan (alvaughan@ucsd.edu)

## Introduction

In this project, we aim to answer the question:

**Can we classify whether a recipe is a baked good using nutritional and recipe metadata?**

We analyze a dataset of recipes and user reviews from Food.com, applying exploratory data analysis, statistical inference, and predictive modeling techniques. The ultimate goal is to determine whether simple nutritional features can spot the difference bewteen a baked good and other types of recipes.

### Dataset Overview

Our project uses two datasets from Food.com:

**`recipes`**: Contains 83,782 rows and includes metadata and nutritional information for each recipe.  
Relevant columns include:

| Column Name      | Description                                      |
|------------------|--------------------------------------------------|
| `name`           | Recipe name                                      |
| `id`             | Unique recipe ID                                 |
| `minutes`        | Minutes to prepare the recipe                    |
| `submitted`      | Date the recipe was submitted                    |
| `nutrition`      | Nutritional info as a list: [calories, fat, sugar, sodium, protein, saturated fat, carbs] |
| `n_steps`        | Number of steps in the recipe                    |
| `n_ingredients`  | Number of ingredients                            |
| `description`    | User-provided recipe description                 |

**`interactions`**: Contains 731,927 rows of user engagement data.  
Relevant columns include:

| Column Name   | Description                                           |
|---------------|-------------------------------------------------------|
| `recipe_id`   | Recipe ID (links to `id` in the `recipes` dataset)    |
| `date`        | Date the rating or review was submitted               |
| `rating`      | Rating given by a user                                |

---

## Data Cleaning and Exploratory Data Analysis

We began by cleaning and preparing the raw `RAW_recipes.csv` and `RAW_interactions.csv` datasets from Food.com to make them usable for analysis and modeling. The original datasets contained nested and stringified fields, as well as compact nutritional information that required transformation. Here are the major steps we performed:

- **Parsing Stringified Lists**: The `tags`, `steps`, and `ingredients` columns in the recipes dataset were stored as stringified Python lists (e.g., `"['tag1', 'tag2']"`). We used `ast.literal_eval` to convert these strings into actual Python lists to enable filtering and counting operations.

- **Nutritional Feature Extraction**: The `nutrition` column held seven numeric values as a single list representing: `[calories, total fat, sugar, sodium, protein, saturated fat, carbohydrates]`. We split this list into separate columns, each with meaningful units (e.g., `sugar (g)`, `sodium (mg)`), and renamed them accordingly.

- **Unit Standardization**: To ensure consistency across our dataset, we cross-referenced nutrient values against USDA standards and converted them into standardized units. For example, we represented sodium in milligrams and all other values in grams where appropriate.

- **User Ratings Integration**: From `RAW_interactions.csv`, we calculated the average rating for each recipe using the `rating` values provided by users. We then merged these average ratings into the main recipe dataset using the shared `recipe_id`.

- **Creating a Target Column (`is_baked_good`)**: We introduced a new binary column, `is_baked_good`, which flags whether a recipe is likely to be a baked good. This was determined by checking for the presence of specific baking-related keywords (e.g., 'bake', 'cookie', 'cake', 'bread') in the `tags` and `name` columns.

## Nutritional Value Filtering

To ensure our dataset only includes realistic and representative recipes, we applied a series of filters to remove extreme nutritional values. This step was necessary to improve the quality of our analysis and model performance. Here's how and why we did it:

**Why We Filtered Nutritional Columns**: Nutritional values like sugar, fat, or sodium can sometimes contain extreme outliers due to data entry errors, bulk recipes, or misreported units. These outliers distort summary statistics, reduce model reliability, and make visualizations harder to interpret.

**Thresholds We Used**:
  We limited each key nutrient to a maximum value that aligns with typical human consumption for a single recipe serving:

  * `sugar (g) < 300`: Equivalent to \~75 teaspoons of sugar.
  * `protein (g) < 100`: Higher than most meals but within reason.
  * `saturated fat (g) < 100`: Exceeds daily recommended intake.
  * `carbohydrates (g) < 300`: Represents \~1200 calories of carbs.
  * `sodium (mg) < 4000`: Roughly double the recommended daily max.
  * `calories (#) < 2000`: Reflects a full day’s intake in one dish.

**Why These Limits Are Justified**:
These thresholds were based on dietary guidelines (USDA) and an inspection of the data distributions. They helped us retain the vast majority of recipes while removing unrealistic entries that could harm our modeling or mislead interpretations.

Overall, this filtering process allowed us to focus our analysis on standard recipes intended for individual or household consumption—improving the quality, clarity, and trustworthiness of all subsequent results.

After cleaning, we performed exploratory data analysis (EDA) using univariate and bivariate visualizations. For instance:

- We plotted the distribution of sugar, calories, and protein content across all recipes.
- We compared nutrient distributions between baked and non-baked goods using box plots.
- We created aggregate tables summarizing the average nutritional content for each group.

These steps helped us better understand the structure and trends in the dataset, and informed the direction of our modeling efforts.

### Univariate Analysis

#### Sugar Content
- There is a strong right skew indicating that high-sugar recipes are rare, with **most recipes having sugar content less than 50g**
- There are a few outliers with extremely high-sugar levels, explaining the long tail

<iframe src="assets/sugar_distribution1.html" width="900px" height="600px" style="border:none;"></iframe>

#### Calorie Content
- **Most recipes fall between 200 and 600 calories**, suggesting that the majority of recipes in this dataset are moderate-calorie dishes.
  
<iframe src="assets/calories_distribution1.html" width="900px" height="600px" style="border:none;"></iframe>

### Bivariate Analysis

#### **`sugar (g)` vs `is_baked_good`**
- Baked goods tend to have **higher sugar content on average** compared to non-baked goods, as expected.
- However, the spread is tighter for baked goods, whereas non-baked goods show more variability and extreme outliers.

<iframe src="assets/sugar_by_baked1.html" width="900px" height="600px" style="border:none;"></iframe>

#### **`protein (g)` vs `is_baked_good`**
- Baked goods generally have **lower protein content** than non-baked goods, with median protein levels visibly lower.
- There is greater variability in protein among non-baked goods, likely due to the inclusion of protein-rich entrees.

<iframe src="assets/protein_by_baked1.html" width="900px" height="600px" style="border:none;"></iframe>


Exploratory plots showed that:
- Baked goods generally have higher sugar content
- Non-baked goods typically have higher protein content
- The distribution of calories is right-skewed, with many high-calorie baked items

---

## Assessment of Missingness

### NMAR Analysis

We examined whether any of the missing data in the dataset is likely **Not Missing at Random (NMAR).**

We believe the missingness in the rating column may be NMAR, since it's likely that the decision to leave a rating depends on the user's sentiment, which we do not observe. For example, users with neutral experiences may be less likely to leave ratings. Additionally, if the data scraping pulled both comments and formal ratings without a clear distinction, the missingness may reflect underlying behavior patterns not captured in the observed data. Therefore, the missingness depends on unobserved factors related to the rating itself.

Because these underlying factors aren’t captured in the dataset, the missingness in rating is not related to other columns. Thus, we believe that description is likely NMAR where its missingness may be directly related to its own value, not other columns.

---

### Missingness Dependency: Permutation Tests

We focused on the `description` column, which was missing for a subset of recipes. To evaluate whether the missingness in `description` is related to other observable variables, we performed **permutation tests** on the following:

- `n_steps` — the number of procedural steps in a recipe  
- `n_ingredients` — the number of ingredients  
- `avg_rating` — the average user rating  

These tests compared the means of each variable between two groups:
- Recipes **with** a description
- Recipes **without** a description

The null hypothesis for each test was that the missingness of the `description` is unrelated to the variable in question (i.e., missingness is random).

---

#### `n_steps`  
- **Observed difference**: 0.7194  
- **p-value**: 0.241  
There is no significant difference in the number of steps between recipes with and without descriptions, suggesting that missingness in `description` is not strongly dependent on recipe complexity.

<iframe src="assets/n_steps_description_missing.html" width="100%" height="500px" style="border:none;"></iframe>

---

#### `n_ingredients`  
- **Observed difference**: −1.1335  
- **p-value**: 0.001  
Recipes missing a description tend to have fewer ingredients. This relationship is statistically significant, indicating that simpler recipes are more likely to have missing descriptions.

<iframe src="assets/n_ingredients_description_missing.html" width="100%" height="500px" style="border:none;"></iframe>

---

#### `avg_rating`  
- **Observed difference**: −0.1903  
- **p-value**: 0.000  
Recipes with missing descriptions receive lower ratings on average. This may reflect user behavior patterns—users may be less likely to leave comments or descriptions when their experience was neutral or negative.

<iframe src="assets/avg_rating_description_missing.html" width="100%" height="500px"style="border:none;"></iframe>

---

### Conclusion

- The missingness in the `description` column **is not MCAR** (Missing Completely At Random).
- The significant associations with `n_ingredients` and `avg_rating` suggest the missingness is **at least MAR** (Missing At Random).
- Because user sentiment is likely unobserved and influences both ratings and the inclusion of descriptions, the data may also reflect **NMAR** (Not Missing At Random) behavior.

Understanding the nature of missingness helps guide how we handle such data during modeling. For example, standard imputation may not be appropriate when data is NMAR, and more cautious strategies like sensitivity analysis should be considered.

---

## Hypothesis Testing

To statistically test whether baked goods contain more sugar than non-baked goods, we ran a one-sided permutation test.

**Null Hypothesis (H₀):**  
The average sugar content of baked goods is less than or equal to that of non-baked goods.  

**Alternative Hypothesis (H₁):**  
The average sugar content of baked goods is greater than that of non-baked goods.

- **Test statistic:** Difference in means

<iframe src="assets/difference_in_means_sugar.html" width="100%" height="500px" style="border:none;"></iframe>

  <iframe src="assets/permutation_test_visual3.html" width="100%" height="500px" style="border:none;"></iframe>

- **Observed statistic:** 21.4139 grams
- **p-value:** < 0.001

**Conclusion:**  
We reject the null hypothesis. There is strong statistical evidence that baked goods have higher sugar content on average.

---

## Framing a Prediction Problem

We define the prediction task as a binary classification problem:

- **Goal:** Predict whether a recipe is a baked good based on its nutritional content and metadata.
- **Target Variable:** `is_baked_good` (True/False)
- **Features:** Nutritional information such as sugar, protein, and recipe metadata

This is a **classification problem**, as the output variable is categorical.

---

## Baseline Model

The baseline model is a logistic regression using only two features: `sugar (g)` and `protein (g)`.

**Results:**

- Precision (Baked): 66%
- Recall (Baked): 22%
- F1-score (Baked): 33%  
- Accuracy: 75%

### Confusion Matrix: Baseline Model

The confusion matrix below summarizes the performance of our baseline logistic regression model using only **sugar** and **protein** as features.

- **True Positives (Top-left):** Correctly predicted baked goods.  
- **False Positives (Top-right):** Incorrectly predicted non-baked goods as baked.  
- **False Negatives (Bottom-left):** Missed baked goods (predicted as non-baked).  
- **True Negatives (Bottom-right):** Correctly predicted non-baked goods.

<iframe src="assets/baseline_confusion_matrix.html" width="100%" height="500px" style="border:none;"></iframe>

This model serves as a baseline for comparison with more complex models using additional features and engineered variables.


**Note:** Precision, Recall, and F1-score reported here refer to the *baked good* class (i.e., `is_baked_good = True`).

While it classifies non-baked goods well, it performs poorly on baked goods due to class imbalance and limited features.

---

## Final Model

We improved our model in several ways:
- Added features: `n_steps`, `calories (#)`, `n_ingredients`, `description`
- Engineered features: calories per ingredient, description word count
- Used a Random Forest Classifier with class weighting
- Performed 5-fold cross-validation with grid search tuning

To improve our model beyond just using `sugar (g)` and `protein (g)`, we added features that reflect the structure and richness of a recipe. `n_steps`, `calories (#)`, `n_ingredients` capture how complex or dense a recipe is—baked goods often involve structured, multi-step processes and include many calorie-dense ingredients.

We also engineered two features:

- **calories per ingredient** measures how rich or dense a recipe is per component—baked goods often have a higher ratio due to ingredients like butter and sugar.

- **description word count** captures how detailed the recipe description is—baked goods tend to have longer, more descriptive write-ups that highlight texture, flavor, and preparation.

These features align with how baked goods are typically composed and described, making them valuable for improving prediction accuracy.

**Best Parameters:**
- `n_estimators`: 100  
- `max_depth`: 10  
- `bootstrap`: True  
- `class_weight`: 'balanced'

**Cross-Validation Performance:** 
- Average Precision: 67.1%  
- Average Recall: 74.1%  
- Average F1 Score: 70.4%
- Average Accuracy: 82.4%

The plot below displays the average performance across the folds in terms of Accuracy, Precision, Recall, and F1-score as compared to our baseline model results.

<iframe src="assets/baseline_vs_final.html" width="100%" height="500px" style="border:none;"></iframe>

- **Accuracy** reflects the overall proportion of correctly classified observations.
- **Precision** measures how many of the predicted baked goods were actually baked goods.
- **Recall** indicates how many of the actual baked goods were correctly identified.
- **F1-score** balances precision and recall, useful when classes are imbalanced.

The final model greatly improved performance, especially recall and F1 score, indicating better handling of baked goods.

---

## Fairness Analysis

To assess whether our final model performs equitably across different subgroups, we conducted a fairness analysis based on the year a recipe was submitted. Specifically, we explored whether the model's performance varies between recipes submitted before 2010 and those submitted in or after 2010.

### Year Distribution

We began by plotting the distribution of recipes by year. Most recipes in our dataset were submitted between 2008 and 2012, with a steep drop-off in later years.

### Methodology

We binarized the `year` column at the threshold of 2010 to define two groups:

- **Group 0**: Recipes submitted before 2010
- **Group 1**: Recipes submitted in or after 2010

We then calculated the **F1 score** for each group using the model’s predictions and conducted a **permutation test** to determine whether the observed difference in F1 scores is statistically significant.

<iframe src="assets/recipes_by_year.html" width="100%" height="500px" style="border:none;"></iframe>

### Results

- **Observed F1 score difference (group 1 − group 0)**: -0.0209  
- **p-value**: 0.002

This result indicates that the model performs slightly better on older recipes (pre-2010), and the observed difference in F1 scores is statistically significant at the 0.01 level. This suggests that there may be temporal bias in the model's performance.

<iframe src="assets/fairness_f1_permutation.html" width="100%" height="500px" style="border:none;"></iframe>

### Interpretation

While the performance difference is relatively small in magnitude, its statistical significance prompts further investigation. One possible explanation could be changes in how recipes are written or tagged in more recent years, leading to a mismatch between newer formats and the model's learned patterns.

