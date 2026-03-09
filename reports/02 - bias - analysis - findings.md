## Bias Detection & Proxy Discrimination Analysis

This notebook applies a structured bias analysis to the cleaned dataset (`clean_credit_applications.csv`), using **Python**, **pandas**, **scipy**, and **fairlearn** to detect fairness violations, identify proxy discrimination, and quantify interaction effects across protected attributes.

### Pipeline Overview

#### 1. Data Loading
* Loads `clean_credit_applications.csv` from the Data Engineering export directly, without MongoDB dependency.
* Derives applicant age from `date_of_birth` and bins into four groups (`<30`, `30–44`, `45–59`, `60+`).
* Parses `spending_behavior` from stringified list format for category-level analysis.

#### 2. Overview of Protected Attributes
* **Gender Distribution:** Balanced split (Male: 246, Female: 248), ensuring sufficient statistical power for group-level comparisons.
* **Age Distribution:** 30–44 dominates (256 records), while 60+ has only 43, making results for that group statistically limited.
* **Overall Approval Rate:** 58.3% baseline against which all group disparities are measured.

#### 3. Bias Detection

##### 3.1 Gender Disparate Impact
* Applied the four-fifths rule (DI = approval rate female / approval rate male). Female approval (50.8%) versus male approval (65.9%) produces a **DI of 0.772**, below the 0.8 legal threshold, confirmed statistically significant (χ², p = 0.001).
* **Fairlearn validation:** `demographic_parity_difference` (DPD) computed via `fairlearn.metrics` confirms a **−0.151 absolute gap** (15.1 percentage points) in favour of male applicants. A `MetricFrame` breakdown provides audit-ready per-group rates for regulatory reporting under the EU AI Act.

##### 3.2 Age-Based Discrimination Patterns
* Applicants under 30 are approved at only 41.5%, compared to 60–65% for all older groups, a statistically significant gap (χ², p = 0.008).

#### 4. Proxy Discrimination
* **Correlation Heatmap:** Mapped the relationship between all financial features and the approval outcome to identify proxy candidates.
* **`annual_income` → gender proxy:** Tested but not confirmed — male ($81,358) and female ($83,599) incomes are virtually identical (t-test, p = 0.38).
* **`savings_balance` → gender proxy:** Also tested via independent t-test to check for a gender wealth gap that could serve as a second indirect discrimination channel.
* **`credit_history_months` → age proxy:** Strong positive correlation with applicant age (r = 0.65, p < 0.001), meaning young applicants are structurally penalised for a variable they cannot meaningfully improve.
* **`zip_code` → geographic proxy:** NYC-area applicants (100xx) approved at 64.3% versus 52.0% for LA-area (902xx), a significant difference (χ², p = 0.025) not explained by financial profiles alone.
* **Spending behavior → age proxy:** Spending categories analysed by age group via correlation heatmap and **Kruskal-Wallis tests** (non-parametric, appropriate for right-skewed amounts). Formal statistical tests determine which categories differ significantly across age groups; spending is a weaker proxy than credit history overall.

##### 5. Interaction Effects
* Disparate impact ratios computed within each age subgroup. Women under 30 face a DI of **0.682** (34.1% vs 50.0%), significantly worse than either group in isolation, consistent with intersectional discrimination.

#### 6. Statistical Summary
* All 8 tests — chi-squared, t-tests, Pearson correlation, Kruskal-Wallis, and Fairlearn DPD — consolidated into a single reference table covering every hypothesis tested.

### Key Results
* **Disparate Impact Ratio (gender):** as 0.772 < 0.8 —> four-fifths rule **FAILS**
* **Demographic Parity Difference (fairlearn):** 15.1 % absolute gap in favour of male applicants
* **Most affected group:** Women under 30 (DI = 0.682, approval rate 34.1%)
* **Confirmed proxies:** `credit_history_months` (age), `zip_code` (geography)
* **Rejected proxy hypotheses:** `annual_income` and `savings_balance` — no significant gender gap detected in either financial attribute