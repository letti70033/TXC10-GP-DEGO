# TXC10-GP-DEGO
TXC10 - DEGO 2606 Group Project – Credit Application Governance Analysis


## Data Engineering Pipeline: Credit Applications

This repository contains a robust Data Engineering pipeline designed to process, clean, and validate raw credit application data using **Python** and **MongoDB**. The pipeline transforms messy, unstructured NoSQL data into a pristine, validated dataset.

### Pipeline Overview

The workflow is divided into four critical stages:

#### 1. Data Loading & Deduplication
* Connects to a local MongoDB instance (`novacred_db`).
* Parses raw JSON data and enforces uniqueness by dropping duplicate application `_id`s upon insertion (removed 2 resubmitted duplicates).
* Successfully loaded 500 unique applications into the database for processing.

#### 2. Data Cleaning
A comprehensive quality assurance process addressing six core data quality dimensions:
* **Inconsistent Data Types:** Dynamically audited the nested structures and cast mixed-type fields (e.g., converted 8 string representations of `annual_income` into integers).
* **Missing Data & KYC Compliance:** * **Dropped:** 6 structurally invalid records (5 missing SSN, 1 missing Date of Birth) to ensure strict KYC (Know Your Customer) and age verification compliance.
    * **Imputed:** Filled missing demographic and contact data (e.g., emails) with "UNKNOWN" placeholders to preserve valid financial data for downstream analysis.
* **Schema Drift:** Identified and resolved drifting schema keys by mapping legacy `annual_salary` fields to the official `annual_income` key without data loss.
* **Categorical Formatting:** Standardized inconsistent categorical codes using direct mapping (e.g., mapped 111 instances of "M"/"F" to "Male"/"Female" for accurate demographic grouping).
* **Impossible Values:** Applied strict domain-specific rules to correct negative numbers:
    * *Time Metrics:* Converted negative `credit_history_months` (2 records) to their absolute values.
    * *Financial Metrics:* Floored negative `savings_balance` amounts (1 record) to `$0` to prevent artificial wealth inflation.
* **Inconsistent Dates:** Utilized `pandas` datetime parsing to intelligently interpret chaotic date strings (e.g., `DD/MM/YYYY`) and standardized 157 records to the strict ISO 8601 format (`YYYY-MM-DD`).

#### 3. Schema Validation
* Locked the collection against future bad inserts by enforcing a strict MongoDB `$jsonSchema` based on the official project Data Dictionary.
* Guarantees all required nested objects (`applicant_info`, `financials`), critical identifiers, BSON data types, and ISO 8601 regex patterns are perfectly compliant. 
* **Final Audit Result:** 0 invalid documents found.

#### 4. Data Export
Extracts the validated database into two distinct formats for cross-functional teams:
* **JSON:** A nested `clean_credit_applications.json` file for system backups and software engineering consumption.
* **CSV:** A flattened, tabular `clean_credit_applications.csv` file (using `pandas.json_normalize`) designed specifically for the Data Science team's models.

### Key Results
* **Raw Records Evaluated:** 502
* **Final Clean Records:** 494 perfectly validated, structurally sound records ready for analysis.

## Data Scientist Pipeline: Bias Detection & Proxy Discrimination Analysis

This notebook applies a structured bias analysis to the cleaned dataset (`clean_credit_applications.csv`), using **Python**, **pandas**, and **scipy** to detect fairness violations, identify proxy discrimination, and quantify interaction effects across protected attributes.

### Pipeline Overview

#### 1 - Data Loading
* Loads `clean_credit_applications.csv` from the Data Engineering export directly, without MongoDB dependency.
* Derives applicant age from `date_of_birth` and bins into four groups (`<30`, `30–44`, `45–59`, `60+`).
* Parses `spending_behavior` from stringified list format for category-level analysis.
* Excludes 3 records with `gender = UNKNOWN` from protected-attribute comparisons.

#### 2 - Overview of Protected Attributes
* **Gender Distribution:** Balanced split (Male: 246, Female: 248), ensuring sufficient statistical power for group-level comparisons.
* **Age Distribution:** 30–44 dominates (256 records), while 60+ has only 43, making results for that group statistically limited.
* **Overall Approval Rate:** 58.3% baseline against which all group disparities are measured.

#### 3 - Bias Patterns

##### 1. Gender Disparate Impact
* Applied the four-fifths rule (DI = approval rate female / approval rate male). Female approval (50.8%) versus male approval (65.9%) produces a **DI of 0.772**, below the 0.8 legal threshold, confirmed statistically significant (χ², p = 0.001).

##### 2. Age-Based Discrimination Patterns
* Applicants under 30 are approved at only 41.5%, compared to 60–65% for all older groups, a statistically significant gap (χ², p = 0.008).

##### 3. Proxy Variable Analysis
* **Correlation Heatmap:** Mapped the relationship between all financial features and the approval outcome to identify proxy candidates.
* **`annual_income` → gender proxy:** Tested but not confirmed, male ($81,358) and female ($83,599) incomes are virtually identical (t-test, p = 0.38).
* **`credit_history_months` → age proxy:** Strong positive correlation with applicant age (r = 0.65, p < 0.001), meaning young applicants are structurally penalised for a variable they cannot meaningfully improve.
* **`zip_code` → geographic proxy:** NYC-area applicants (100xx) approved at 64.3% versus 52.0% for LA-area (902xx), a significant difference (χ², p = 0.025) not explained by financial profiles alone.
* **Spending behavior → age proxy:** Spending categories analysed by age group, confirmed as a weaker proxy than credit history in this dataset.

##### 4. Interaction Effects
* Disparate impact ratios computed within each age subgroup. Women under 30 face a DI of **0.682** (34.1% vs 50.0%), significantly worse than either group in isolation, consistent with intersectional discrimination.

#### 4 - Statistical Summary
* All chi-squared, t-test, and Pearson correlation results consolidated into a single reference table covering every hypothesis tested.

#### 5 - Key Findings
* **Disparate Impact Ratio (gender):** 0.772 — four-fifths rule **FAIL**
* **Most affected group:** Women under 30 (DI = 0.682, approval rate 34.1%)
* **Confirmed proxies:** `credit_history_months` (age), `zip_code` (geography)
* **Rejected proxy hypothesis:** `annual_income` — no significant gender income gap detected (p = 0.38)
* **Governance flag:** Dominant rejection label `algorithm_risk_score` provides no explanation of decision drivers, conflicting with GDPR right to explanation and EU AI Act transparency requirements.