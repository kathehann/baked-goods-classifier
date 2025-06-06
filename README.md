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

- **Creating a Target Column (`is_baked_good`)**: We introduced a new binary column, `is_baked_good`, which flags whether a recipe is likely to be a baked good. This was determined by checking for the presence of specific baking-related keywords (e.g., 'bake', 'cookie', 'cake', 'bread') in the `tags` column.

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

We assessed missing values across the dataset:

- `avg_rating` had missing values due to recipes without reviews — classified as Missing at Random (MAR) as lower rated foods were more likely to have missing descriptions
- `nutrition`, `tags`, and `description` had sporadic missing values — handled with row drops or safe defaults

Since missingness was generally limited and non-informative, we chose not to impute values aggressively. Instead, we focused our modeling on the subset with full nutritional info.

---

## Hypothesis Testing

To statistically test whether baked goods contain more sugar than non-baked goods, we ran a one-sided permutation test.

**Null Hypothesis (H₀):**  
The average sugar content of baked goods is less than or equal to that of non-baked goods.  

**Alternative Hypothesis (H₁):**  
The average sugar content of baked goods is greater than that of non-baked goods.

- **Test statistic:** Difference in means

  <iframe src="assets/permutation_test_visual1.html" width="100%" height="500px" style="border:none;"></iframe>

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
- Added features: `n_steps`, `calories (#)`, `n_ingredients`, description word count
- Engineered features: calories per ingredient
- Used a Random Forest Classifier with class weighting
- Performed 5-fold cross-validation with grid search tuning

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

<iframe src="assets/recipes_by_year.html" width="100%" height="500px" border:none></iframe>

### Results

- **Observed F1 score difference (group 1 − group 0)**: -0.0209  
- **p-value**: 0.002

This result indicates that the model performs slightly better on older recipes (pre-2010), and the observed difference in F1 scores is statistically significant at the 0.01 level. This suggests that there may be temporal bias in the model's performance.

<iframe src="assets/fairness_f1_permutation.html" width="100%" height="500px" border:none></iframe>

### Interpretation

While the performance difference is relatively small in magnitude, its statistical significance prompts further investigation. One possible explanation could be changes in how recipes are written or tagged in more recent years, leading to a mismatch between newer formats and the model's learned patterns.

