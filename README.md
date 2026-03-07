# TXC10-GP-DEGO
TXC10 - DEGO 2606 Group Project – Credit Application Governance Analysis

1. Data Engineer: 70485 Lennart Stenzel
2. Data Scientist: 70033 Leticia Brendle 
3. Governance Officer: 

This workbook contains 3 notebooks, each corresponding to one Role:

## Data Quality Findings

As part of the NovaCred Data Governance Task Force, we conducted a comprehensive data quality audit on the `raw_credit_applications.json` dataset. To ensure our algorithmic bias testing is built on a reliable foundation, we evaluated the dataset across the six core dimensions of data quality.

Out of the initial 502 raw records, our automated Data Engineering pipeline cleaned and retained **490 valid records**. All removed records were preserved in `data/quarantined_records.json` with machine-readable drop reason tags, ensuring nothing is permanently lost. The clean dataset was then locked under strict MongoDB schema validation to prevent future degradation.

### 1. Uniqueness
* **Issue Found:** Two records (`app_042`, `app_001`) shared identical `_id` fields, indicating resubmissions (0.4% of 502). A secondary SSN audit revealed two additional conflicts: `937-72-8731` and `780-24-9300`, where different individuals shared the same national identifier (4 records, 0.8%).
* **Remediation:** Deduplicated `_id` conflicts during initial loading, keeping the most recent submission. For SSN conflicts with different names, both records in each pair were removed as neither could be verified without external evidence. In total, 6 records (1.2%) were quarantined under the uniqueness stage (2 duplicate IDs, 4 SSN conflicts).

### 2. Completeness
* **Issue Found:** A database-wide audit revealed 5 missing SSNs (1.0%), 1 missing date of birth (0.2%), and 2 missing emails (0.4%). An additional 4 emails were present but structurally invalid (0.8%, e.g. `mike johnson@gmail.com`, `sarah.smith@`). We also found 5 records where `annual_income` was stored under the legacy key `annual_salary` (1.0%, schema drift).
* **Remediation:** Dropped 6 structurally invalid records (1.2%) lacking KYC-required fields (SSN, DOB) that are also essential for bias testing. For emails, both missing and malformed values were set to `null` rather than a placeholder string. This is more honest: `null` means no valid address is known, and the schema explicitly allows it. The 5 drifted `annual_salary` keys were renamed to `annual_income`.

### 3. Consistency
* **Issue Found:** The `applicant_info.gender` field mixed abbreviations and full words ('M'/'F' alongside 'Male'/'Female'), affecting 109 records (21.7%). Additionally, 7 records (1.4%) stored `financials.annual_income` as strings instead of numbers.
* **Remediation:** Standardized 109 gender records by mapping abbreviations to their full-word equivalents. Cast the 7 string-typed income values to integers using a MongoDB aggregation pipeline update.

### 4. Validity
* **Issue Found:** 156 records (31.1%) contained non-standard date formats in `applicant_info.date_of_birth` (slash-delimited variants such as DD/MM/YYYY, MM/DD/YYYY, and YYYY/MM/DD). Four emails (0.8%) were present but failed a standard RFC 5321 structural check.
* **Remediation:** Replaced Pandas date inference (which emits silent warnings on ambiguous input) with a deterministic, rule-based parser. The parser resolves format by inspecting segment magnitude: a first segment above 31 is a year, above 12 is a day, and so on. Truly ambiguous cases (39 records where both day and month are 12 or below) were defaulted to European DD/MM/YYYY, which is the dominant format in the dataset. Invalid emails were nulled via regex audit.

### 5. Accuracy
* **Issue Found:** Two records (0.4%) had negative `credit_history_months` and one record (0.2%) had a negative `savings_balance`, both of which are impossible in practice. Combined, 3 records (0.6%) contained impossible numeric values.
* **Remediation:** Converted negative credit history values to their absolute equivalents (assumed typographical errors). Capped the negative savings balance at $0 to conservatively reflect reality without inflating the applicant's financial profile.

### 6. Timeliness
* **Issue Found:** Five records (1.0%) stored income under the obsolete key `annual_salary` rather than the current `annual_income`, a strong indicator of schema drift from a legacy intake form.
* **Remediation:** The keys were renamed to restore structural completeness. From a governance perspective, these records should be flagged for a data refresh, as the income figures may reflect an outdated financial snapshot and could skew the credit scoring algorithm.


## Data Scientist Pipeline: Bias Detection & Proxy Discrimination Analysis

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
