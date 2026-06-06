# Do Hawaiian Businesses That Respond to Reviews Get Better Ratings?

**By Ashwin Kannan & Abhishai Ganta** | DSC 80 Final Project, UC San Diego

---

## Introduction

Online reviews on platforms like Google Maps shape where people eat, stay, and spend their money, especially in a tourism-heavy destination like Hawaii. Whether or not a business actively engages with its reviewers is a behavior we can check, and the question of whether that engagement is associated with higher ratings has actual implication for business owners, platform designers, and consumers as well.

**Our central question: Do businesses that respond to customer reviews tend to receive higher ratings than businesses that do not?**

We work with two datasets from the Google Maps Reviews collection, filtered to Hawaii:

- **`review-Hawaii_10.json.gz`**: individual customer reviews, containing **1,504,347 rows**.
- **`meta-Hawaii.json.gz`**: business metadata, containing **21,507 rows** (21,421 after removing 86 duplicates).

After merging and aggregating to the business level, our primary analysis DataFrame contains **11,686 rows**, one per unique business.

**Relevant columns from the reviews dataset:**

| Column    | Description                                                                 |
| --------- | --------------------------------------------------------------------------- |
| `gmap_id` | Unique identifier for each business; used to join with metadata             |
| `rating`  | Star rating (1–5) left by a reviewer                                        |
| `resp`    | The business's text response to the review; `null` if no response was given |

**Relevant columns from the metadata dataset:**

| Column                   | Description                                                                       |
| ------------------------ | --------------------------------------------------------------------------------- |
| `gmap_id`                | Unique business identifier; primary key for the join                              |
| `category`               | List of business category labels (e.g., `['Restaurant']`, `['Park']`)             |
| `price`                  | Google Maps price tier (`$`, `$$`, `$$$`, `$$$$`); missing for ~81% of businesses |
| `latitude` / `longitude` | Geographic coordinates of the business                                            |

**Derived columns created during analysis:**

| Column          | Description                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------ |
| `avg_rating`    | Mean star rating across all reviews for a given business                                               |
| `num_reviews`   | Total number of reviews a business has received                                                        |
| `response_rate` | Fraction of reviews for which the business left a response (0 = never responded, 1 = responded to all) |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

Steps taken:

1. **Removed duplicate businesses** from `meta` based on the `gmap_id` column, which uniquely identifies each business. There were 86 duplicate rows, likely caused by the data collection process re-scraping the same business at different times.
2. **Merged** the review and meta datasets on `gmap_id` using a left join, keeping all reviews and attaching available business metadata.
3. **Created** a `has_response` column indicating whether a business responded to a given customer review (`True` if `resp` was present and `False` otherwise).
4. **Aggregated to the business level** by grouping on `gmap_id` and computing each business's `avg_rating`, `num_reviews`, and `response_rate`.

The resulting `business` DataFrame has one row per business and is the primary dataset for all subsequent analysis. Here are the first few rows:

| gmap_id                               | avg_rating | num_reviews | response_rate |
| :------------------------------------ | ---------: | ----------: | ------------: |
| 0x0:0x9edcb14b0cf1ec04                |      3.909 |          22 |         0.955 |
| 0x1150a1a2df43910f:0xc3419d135191bbca |      4.071 |          14 |         0.286 |
| 0x115668f3c694d673:0x73bfd141c6e1669b |      4.684 |          19 |         0.105 |
| 0x11566d436f0c4cc1:0xb0d0dbfba1351beb |      4.594 |          69 |         0.000 |
| 0x11566dad4a2c7545:0xed4746f5f86444a2 |      4.680 |          25 |         0.000 |

### Univariate Analysis

Most businesses have an average rating above 3.5 stars, with ratings concentrated between 4.0 and 5.0 stars. The distribution is left-skewed, indicating that highly rated businesses are far more common than poorly rated ones in this dataset.

<iframe
  src="assets/fig_rating.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

Most businesses have very low response rates, with approximately **71.7%** of businesses never responding to any customer review. Only a small minority exhibit substantial engagement. This is important context for our research question.

<iframe
  src="assets/fig_resp_rate.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

### Bivariate Analysis

There is no strong linear relationship between `response_rate` and `avg_rating`. Businesses at every response level span a wide range of ratings, though businesses with the highest response rates tend to cluster slightly higher. Thus, motivating our hypothesis test.

<iframe
  src="assets/fig_scatter_resp.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

Businesses that respond to more than 50% of their reviews show a noticeably higher median rating than businesses that never respond. The interquartile ranges for the active-responder groups are also slightly tighter, suggesting that engagement may be associated with more consistent ratings.

<iframe
  src="assets/fig_box.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

### Interesting Aggregates

The table below groups businesses by how actively they respond to reviews and shows their aggregate rating statistics. The 0% group (businesses that never respond) contains the vast majority of businesses (8,384 out of 11,686). Businesses in the 51–100% response bin have both the highest mean and median average ratings, offering an initial insight consistent with our research question.

| Response Rate Group | # Businesses | Mean Avg Rating | Median Avg Rating | Mean # Reviews |
| :------------------ | -----------: | --------------: | ----------------: | -------------: |
| 0% (no response)    |         8384 |           4.364 |             4.407 |        121.012 |
| 1–5%                |          760 |           4.382 |             4.409 |        263.468 |
| 6–15%               |          752 |           4.361 |             4.406 |        136.864 |
| 16–50%              |          996 |           4.381 |             4.419 |         99.677 |
| 51–100%             |          794 |           4.476 |             4.500 |        110.009 |

---

## Assessment of Missingness

### NMAR Analysis

We believe the `price` column in the metadata is likely **NMAR** (Not Missing At Random). The `price` field records a Google Maps price tier (e.g., `$`, `$$`, `$$$`), and about 81% of businesses are missing this value.

The reason this is likely NMAR rather than MAR or MCAR comes from the data-generating process. A business's `price` label is typically assigned manually by the business owner or inferred from Google's own category systems. Businesses that are harder to categorize by price, like free public attractions, parks, or municipal services, may be structurally unlikely to ever receive a price label. In other words, the reason the price is missing is related to the nature of the business itself, not any other observed column.

To potentially make this missingness MAR, we would want additional information about whether a business is a commercial establishment vs. a public facility, since that information could fully explain why certain businesses never receive a price tier.

### Missingness Dependency

We examined whether the missingness of the `resp` column (operationalized as `response_rate = 0`) depends on other columns.

**Test 1: Missingness of `resp` vs. `avg_rating`**

- **Null hypothesis:** The missingness of `resp` is independent of a business's average rating; any observed difference is due to random chance.
- **Alternative hypothesis:** The missingness of `resp` depends on average rating.
- **Test statistic:** Difference in mean `avg_rating` between businesses with `response_rate = 0` and `response_rate > 0`.
- **Significance level:** α = 0.05

The p-value is approximately **0.0000**, well below 0.05. We reject the null hypothesis. This provides strong evidence that the missingness of `resp` is associated with `avg_rating`. Businesses that never respond tend to have slightly lower average ratings. The plot below shows the empirical null distribution; the observed statistic falls far outside what would be expected under the null.

<iframe
  src="assets/fig_miss_null.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The overlay histogram below shows the rating distributions directly. Therefore, businesses that never respond (red) are shifted slightly lower than those that do respond (blue).

<iframe
  src="assets/fig_miss_overlay.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

**Test 2: Missingness of `resp` vs. `longitude`**

- **Null hypothesis:** The missingness of `resp` is independent of geographic longitude.
- **Alternative hypothesis:** The missingness of `resp` depends on longitude.
- **Test statistic:** Difference in mean longitude between businesses with `response_rate = 0` and `response_rate > 0`.
- **Significance level:** α = 0.05

The p-value is approximately **0.0536**, above our significance level of 0.05. We fail to reject the null hypothesis. There is no statistically significant evidence that the missingness of `resp` depends on longitude.

---

## Hypothesis Testing

### Null Hypothesis (H₀)

Businesses that respond to reviews and businesses that do not respond have the same average rating. Any observed difference in ratings is due to random chance.

### Alternative Hypothesis (H₁)

Businesses that respond to reviews have **higher** average ratings than businesses that do not respond.

### Test Statistic

Mean rating of responders − Mean rating of non-responders. We use a one-sided test because our research question specifically highlights that responders have _higher_ ratings, not just _different_ ones.

### Significance Level

α = 0.05

### Justification

The difference in means is the natural test statistic for comparing two groups on a continuous variable (`avg_rating`). We use a permutation test rather than a t-test because we make no distributional assumptions about the underlying ratings data. The one-sided direction is justified by our a prior hypothesis that responsive businesses should be better, not just different.

### Results

The observed test statistic was **+0.036 stars**. The p-value from our permutation test (5,000 shuffles) was approximately **0.0000**, far below our significance level of 0.05. We reject the null hypothesis. This provides strong statistical evidence that businesses that respond to reviews tend to have higher average ratings than those that do not.

Note: since this is a permutation test on observational data, we cannot conclude a causal relationship. It is possible that higher-rated businesses are simply more motivated to respond, rather than responding being the cause of higher ratings.

<iframe
  src="assets/fig_hyp.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

---

## Framing a Prediction Problem

We aim to predict a business's **average rating** (`avg_rating`) using information about its review engagement and activity. This is a **regression** problem, since the response variable is continuous.

**Response variable:** `avg_rating`: the mean star rating across all reviews for a given business. We chose this variable because it is the most direct, interpretable measure of business quality available in the dataset, and it is the variable our entire analysis has been centered around.

**Metric:** We evaluate model performance using **Root Mean Squared Error (RMSE)**, which measures average prediction error in the original star units. We prefer RMSE over MAE because it penalizes larger errors more heavily. Therefore, a prediction off by 1.5 stars is much worse than one off by 0.5, and RMSE reflects that asymmetry.

**Features available at time of prediction:** We only use features that would be observable without already knowing the business's true average rating:

- `num_reviews`: how many reviews the business has received
- `response_rate`: what fraction of those reviews received a business response
- `category`: what type of business it is (from metadata)

---

## Baseline Model

We trained a simple **Linear Regression** baseline using two features:

- `num_reviews`: **quantitative** (continuous count; passed through as-is)
- `response_rate`: **quantitative** (continuous proportion 0-1; passed through as-is)

There are **0 ordinal** and **0 nominal** features in this baseline, so no encoding is necessary. Both features are passed directly into a `LinearRegression` estimator inside a single `sklearn` `Pipeline`. We split the data 75/25 into training and test sets with `random_state=42`.

The model achieved a **test RMSE of ~0.3417 stars**. Given that ratings span roughly 1.2–5.0 (a range of ~3.8 stars), a 0.34-star average error is moderate. The model captures the broad trend but linear regression cannot capture nonlinear interactions between engagement features and ratings. We consider this a reasonable but not great baseline, as it gives us a floor to beat.

---

## Final Model

We built a more flexible model using a **RandomForestRegressor**. On top of the original `num_reviews` and `response_rate` features, we engineered two new features and added one categorical feature:

1. **`response_volume`** (`num_reviews × response_rate`, quantitative): This is the absolute number of reviews that received a business response. A 5% response rate means very different things for a business with 20 reviews vs. one with 2,000.

2. **`category_group`** (nominal, one-hot encoded): Different business types have systematically different rating distributions, as a park or beach is rated on a completely different implicit scale than a restaurant or gas station. Including category lets the model account for these structural differences. We grouped categories into the top 5 most common plus an "Other" bucket, then one-hot encoded them via a `ColumnTransformer` inside the pipeline.

We tuned the `max_depth` hyperparameter of the RandomForest using `GridSearchCV` with 5-fold cross-validation and RMSE as the scoring metric. We chose `max_depth` because it directly controls the bias-variance tradeoff. We searched over `[2, 5, 10, 20, 30, 50, 100]`.

The best-performing model used `max_depth = 5`. The final model achieved a **test RMSE of ~0.3383 stars**, compared to the baseline's **0.3417**, showing a consistent improvement. Moderately shallow trees generalize best here: deeper trees overfit the training data while shallower trees miss real structure in the features.

<iframe
  src="assets/fig_pred.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

---

## Fairness Analysis

We evaluated whether the final model performs equally well across two groups that arise naturally from our research question:

- **Group X:** Businesses that **respond** to at least one review (`response_rate > 0`)
- **Group Y:** Businesses that **never respond** (`response_rate = 0`)

This is a meaningful fairness split because `response_rate` is a feature in our model, and we want to know whether the model is systematically more or less accurate for one engagement group.

**Evaluation metric:** RMSE (required for regression models)

**Test statistic:** RMSE(Group X) - RMSE(Group Y)

**Null hypothesis (H₀):** Our model is fair. Its RMSE for businesses that respond and those that do not are roughly equal; any observed difference is due to random chance.

**Alternative hypothesis (H₁):** Our model is unfair. Its RMSE differs meaningfully between the two groups.

**Significance level:** α = 0.05

The resulting p-value was approximately **0.657**. Since this is much larger than 0.05, we fail to reject the null hypothesis. There is not sufficient statistical evidence that the model performs differently across the two groups. Any observed difference in error is likely due to random variation. Our model does not appear to show meaningful predictive bias between businesses that engage with reviews and those that don't.

<iframe
  src="assets/fig_fair.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>
