# Online Retail: Data Cleaning, EDA and Sales Prediction

Dataset: UCI Online Retail (about one year of transactions from a UK-based online retailer, 2010-12-01 to 2011-12-09).

This project covers three tasks: data cleaning, exploratory data analysis (EDA), and a sales prediction model.

## Folder contents

| File | Description |
|---|---|
| `Online Retail.xlsx` | Original raw dataset (UCI Online Retail Dataset, unchanged) |
| `cleaned_online_retail.csv` | Output of Task 1 (531,201 rows x 15 columns) |
| `01_Data_Cleaning.ipynb` | Task 1: data cleaning |
| `02_EDA.ipynb` | Task 2: exploratory data analysis |
| `02_EDA.ipynb - Colab.pdf` | PDF export of the Task 2 notebook |
| `03_Sales_Prediction.ipynb` | Task 3: sales prediction model |
| `README.md` | This file |

## How to run

1. Open the notebooks in Google Colab (or Jupyter) with Python 3.
2. Run the notebooks in order: `01_Data_Cleaning.ipynb`, then `02_EDA.ipynb`, then `03_Sales_Prediction.ipynb`.
3. `01_Data_Cleaning.ipynb` needs the raw dataset (`Online Retail.xlsx`). `02_EDA.ipynb` and `03_Sales_Prediction.ipynb` need `cleaned_online_retail.csv`.
4. In Colab, uploaded files are deleted when the session ends, so upload the required file again at the start of every session (each notebook has an upload cell at the top).
5. Use `Runtime > Restart session and run all` to reproduce all outputs. The grid search in Task 3 takes about 30 to 45 seconds.

Libraries: pandas, numpy, matplotlib, seaborn, scikit-learn.

## Task 1: Data cleaning

- Loaded and inspected the raw data, handled missing values, duplicates, column names (snake_case) and data types.
- 8,666 cancelled invoices (invoice numbers starting with "C") were kept and flagged with `is_cancelled`, not deleted.
- Outliers were detected with the IQR method and flagged (`is_outlier_*` columns), not removed.
- Rows without a customer ID are flagged with `is_missing_customer`.
- Final cleaned dataset: 531,201 rows x 15 columns.

## Task 2: Exploratory data analysis

- Descriptive statistics, histograms with KDE, correlation heatmap, box plots, line charts, bar charts, group-by analysis and scatter plots.
- "Genuine sales" were defined as rows where `is_cancelled` is False, excluding one sale that was later reversed by a manual entry (see "Known issues" below). This gives 522,534 rows.
- Key findings:
  - Monthly revenue rises from September to November 2011 (about 1.03M, 1.10M and 1.45M), driven by the number of invoices rather than order value.
  - The United Kingdom accounts for 84.72% of revenue. Netherlands, Australia and Japan have few but large invoices.
  - Revenue peaks between 10:00 and 15:59.
  - The top 10 customers account for 16.63% of the revenue of customers with a known ID.
  - `quantity`, `unit_price` and `total_price` are strongly right-skewed.
- Full insights, recommendations and limitations are in the notebook.

## Task 3: Sales prediction

**Goal:** predict daily revenue (`total_price` summed per day) from genuine sales.

**Design decisions**
- **Daily level:** row-level prediction has no usable history for lag features and monthly data has only about 13 points.
- **No data leakage:** `total_price = quantity x unit_price`, so same-day quantity and price are never used as features. Rolling features are computed on the series shifted by one day.
- **Features (9):** `lag_1`, `lag_7`, `lag_14`, `roll_mean_7`, `roll_mean_14`, `day_of_week`, `day_of_month`, `is_month_start`, `is_month_end`. `month` and `week_of_year` were created but excluded, because November never appears in the training period.
- **Chronological 80/20 split:** 216 training days (2010-12-15 to 2011-10-06) and 55 test days (2011-10-07 to 2011-12-09), because the goal is to predict future sales.
- **Tuning:** GridSearchCV with TimeSeriesSplit on the training data only; the test set was used once for the final score.

**Results (test set)**

| Model | RMSE | MAE | R2 |
|---|---|---|---|
| Random Forest (tuned) | 15,132 | 11,019 | 0.315 |
| Linear Regression | 15,156 | 11,317 | 0.313 |
| Random Forest (default) | 15,350 | 10,948 | 0.295 |
| Naive (7-day rolling mean) | 17,823 | 13,359 | 0.049 |

- Both machine learning models reduce RMSE by about 15% compared with the naive baseline, but explain only about 30% of the variation in daily revenue.
- The tuned Random Forest and Linear Regression are practically tied; with 55 test days the difference is within noise.
- The most important features are `lag_7`, `roll_mean_7` and `day_of_week`, which points to a weekly pattern.
- The mean absolute error is about 22% of the average daily revenue in the test period. The model under-predicts on average and misses the largest spikes (for example, 2011-11-14: actual about 111,349, predicted about 51,961).

## Limitations

- Only about one year of data (December 2011 is incomplete), so yearly seasonality cannot be learned. The test period has a much higher average revenue (about 49,900) than the training period (about 28,000).
- The test set has only 55 days, so metrics are noisy.
- Large daily spikes are not predictable from past revenue and calendar features. Outliers were flagged in Task 1, not removed.
- The model is meant for next-trading-day prediction, because lag and rolling features need recent actual revenue.
- 25.1% of rows have no customer ID. The data has no Saturdays and has a holiday gap at the end of December.
- `day_of_week` is used as a number (0 to 6), which is not ideal for Linear Regression.
- A small hyperparameter grid was searched, and cross-validation RMSE is not comparable with test RMSE because the folds come from the earlier, lower-revenue part of the year.

## Known issues

- **Task 1 inconsistency:** Task 1 has 522,535 non-cancelled product rows, while Tasks 2 and 3 use 522,534. One sale (invoice 556444, stock code 22502, quantity 60 x 649.50 = 38,970, customer 15098, 2011-06-10) was reversed by a manual entry (C556445, stock code "M"), which Task 1 removed as a non-product row. This left the original sale unreversed in the cleaned file. Tasks 2 and 3 exclude that sale manually. The cleaned CSV from Task 1 was not changed, to keep the submitted files consistent.

## Possible improvements (not tested)

- Predict the logarithm of revenue to reduce the effect of spikes.
- One-hot encode `day_of_week`; add holiday and promotion indicators.
- Add lagged order or customer counts as features (using only past values).
- Use at least two full years of data.
