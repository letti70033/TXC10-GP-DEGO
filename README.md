# TXC10-GP-DEGO
TXC10 - DEGO 2606 Group Project – Credit Application Governance Analysis

1. Data Engineer: 70485 Lennart Stenzel
2. Data Scientist: 70033 Leticia Brendle 
3. Governance Officer: 

This workbook contains 3 notebooks, each corresponding to one Role:

## Data Quality Findings

As part of the NovaCred Data Governance Task Force, we conducted a comprehensive data quality audit on the `raw_credit_applications.json` dataset. To ensure our algorithmic bias testing is built on a reliable foundation, we evaluated the dataset across the six core dimensions of data quality. 

Out of the initial 500+ records, our automated Data Engineering pipeline successfully cleaned and retained 494 valid records. These were subsequently locked under strict MongoDB schema validation to prevent future degradation into a "data swamp".

### 1. Uniqueness
* **Issue Found:** We identified 2 duplicate records (`app_042`, `app_001`) sharing identical `_id` fields. 
* **Remediation:** Overwrote the duplicates during database insertion to ensure every application represents a unique individual, preventing skewed analytics.

### 2. Completeness
* **Issue Found:** A database-wide audit revealed missing critical identifiers (5 missing SSNs, 1 missing DOB), missing demographic data (gender), and missing contact information (2 missing emails). We also found 5 records where `annual_income` was seemingly missing, but was actually stored under the key `annual_salary` (schema drift).
* **Remediation:** Dropped 6 structurally invalid "ghost" records that lacked legally required KYC fields (SSN/DOB) or essential demographic data (gender) needed for bias testing. However, for the 2 missing emails, we imputed them with a placeholder ("UNKNOWN_EMAIL") instead of dropping the rows. We did this because email is non-critical for the credit risk algorithm; dropping those rows would mean throwing away perfectly valid financial data. Finally, we renamed the 5 drifted `annual_salary` keys to `annual_income`.

### 3. Consistency
* **Issue Found:** We found inconsistent categorical coding in the `applicant_info.gender` field (mixing 'M'/'F' with 'Male'/'Female') and inconsistent data types across the collection (8 records stored `financials.annual_income` as strings instead of numbers). 
* **Remediation:** Standardized 111 records by mapping 'M'/'F' to 'Male'/'Female'. Dynamically cast the 8 string-based income values to integers. 

### 4. Validity
* **Issue Found:** 157 records contained invalid date formats in `applicant_info.date_of_birth` (e.g., DD/MM/YYYY or YYYY/MM/DD instead of the ISO 8601 standard).
* **Remediation:** Utilized Pandas datetime parsing to intelligently convert all 157 non-standard strings into the strict `YYYY-MM-DD` format required by our schema.

### 5. Accuracy
* **Issue Found:** We detected impossible, negative values in numeric fields: 2 records had negative `credit_history_months` and 1 record had a negative `savings_balance`.
* **Remediation:** Converted the time metrics to absolute (positive) values, assuming typographical errors. Capped the negative savings balance at $0 to conservatively reflect reality without artificially inflating the applicant's wealth profile.

### 6. Timeliness
* **Issue Found:** While static datasets limit real-time timeliness checks, our audit uncovered a strong indicator of stale data. We found 5 records where income was stored under the obsolete key `annual_salary` rather than the current `annual_income`. 
* **Remediation & Governance Impact:** This schema drift strongly suggests these 5 applications originated from a legacy system or an outdated version of the application form. Consequently, the financial figures in these specific records may no longer be up-to-date, potentially violating the Timeliness dimension. We mapped the keys to salvage the records for structural completeness, but our governance policy recommendation is to flag these older records and mandate a "data refresh" to ensure NovaCred's algorithms are scoring applicants based on their current financial reality.


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
