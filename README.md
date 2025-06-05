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

We began by cleaning the raw `RAW_recipes.csv` and `RAW_interactions.csv` datasets:

- Converted stringified lists in `tags`, `steps`, and `ingredients` to Python lists
- Split the `nutrition` column into 7 individual nutrient features
- Converted PDV values to gram/mg equivalents using USDA standards from 2016 onward
- Merged average user ratings from `RAW_interactions.csv` into the recipes dataset
- Added a new column `is_baked_good` (based on tag keywords)

###Univariate Analysis

#### Sugar Content
- There is a strong right skew indicating that high-sugar recipes are rare, with **most recipes having sugar content less than 50g**
- There are a few outliers with extremely high-sugar levels, explaining the long tail

<iframe src="assets/sugar_distribution.html" width="600" height="400" style="border:none;"></iframe>

#### Calorie Content
- **Most recipes fall between 200 and 600 calories**, suggesting that the majority of recipes in this dataset are moderate-calorie dishes.
  
<iframe src="assets/calories_distribution.html" width="600" height="400" style="border:none;"></iframe>

### Bivariate Analysis

#### **`sugar (g)` vs `is_baked_good`**
- Baked goods tend to have **higher sugar content on average** compared to non-baked goods, as expected.
- However, the spread is tighter for baked goods, whereas non-baked goods show more variability and extreme outliers.

<iframe src="assets/sugar_by_baked.html" width="600" height="400" style="border:none;"></iframe>

#### **`protein (g)` vs `is_baked_good`**
- Baked goods generally have **lower protein content** than non-baked goods, with median protein levels visibly lower.
- There is greater variability in protein among non-baked goods, likely due to the inclusion of protein-rich entrees.

<iframe src="assets/protein_by_baked.html" width="600" height="400" style="border:none;"></iframe>


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

  <iframe src="assets/permutation_test_visual.html" width="100%" height="500px"></iframe>

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

  <iframe src="assets/baseline_confusion_matrix.html" width="100%" height="500px"></iframe>

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

The final model greatly improved performance, especially recall and F1 score, indicating better handling of baked goods.

---

## Fairness Analysis

