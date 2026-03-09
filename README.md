# TXC10-GP-DEGO
TXC10 - DEGO 2606 Group Project – Credit Application Governance Analysis

1. Data Engineer: 70485 Lennart Stenzel
2. Data Scientist: 70033 Leticia Brendle 
3. Governance Officer: 72120 Laura Rodrigues

This workbook contains 3 notebooks, each corresponding to one Role:

## Data Quality Findings

As part of the NovaCred Data Governance Task Force, we conducted a comprehensive data quality audit on the `raw_credit_applications.json` dataset. To ensure our algorithmic bias testing is built on a reliable foundation, we evaluated the dataset across the six core dimensions of data quality.

Out of the initial 502 raw records, our automated Data Engineering pipeline cleaned and retained **487 valid records**. All removed records were preserved in `data/quarantined_records.json` with machine-readable drop reason tags, ensuring nothing is permanently lost. The clean dataset was then locked under strict MongoDB schema validation to prevent future degradation.

### 1. Uniqueness
* **Issue Found:** Two records shared identical `_id` fields, indicating resubmissions (0.4% of 502). A secondary SSN audit revealed two additional conflicts where different individuals shared the same national identifier (4 records, 0.8%).
* **Remediation:** Deduplicated `_id` conflicts during initial loading, keeping the most recent submission. For SSN conflicts with different names, both records in each pair were removed as neither could be verified without external evidence. In total, 6 records (1.2%) were quarantined under the uniqueness stage (2 duplicate IDs, 4 SSN conflicts).

### 2. Completeness
* **Issue Found:** A database-wide audit revealed 5 missing SSNs (1.0%), 1 missing date of birth (0.2%), 2 missing emails (0.4%), and 2 missing zip codes (0.4%). An additional 4 emails were present but structurally invalid (0.8%, e.g. `mike johnson@gmail.com`, `sarah.smith@`). We also found 5 records where `annual_income` was stored under the legacy key `annual_salary` (1.0%, schema drift).
* **Remediation:** Dropped 6 structurally invalid records (1.2%) lacking KYC-required fields (SSN, DOB) that are also essential for bias testing. For emails, both missing and malformed values were set to `null` rather than a placeholder string. This is more honest: `null` means no valid address is known, and the schema explicitly allows it. Missing zip codes were to be imputed with the placeholder `"UNKNOWN_ZIP"` to preserve records for geographic bias analysis; however, both affected records turned out to be the same ghost records already removed during the SSN drop, so no imputation was required. The 5 drifted `annual_salary` keys were renamed to `annual_income`.

### 3. Consistency
* **Issue Found:** The `applicant_info.gender` field mixed abbreviations and full words ('M'/'F' alongside 'Male'/'Female'), affecting 109 records (21.7%). Additionally, 7 records (1.4%) stored `financials.annual_income` as strings instead of numbers.
* **Remediation:** Standardized 109 gender records by mapping abbreviations to their full-word equivalents. Cast the 7 string-typed income values to integers using a MongoDB aggregation pipeline update.

### 4. Validity
* **Issue Found:** 156 records (31.1%) contained non-standard date formats in `applicant_info.date_of_birth` (slash-delimited variants such as DD/MM/YYYY, MM/DD/YYYY, and YYYY/MM/DD). Four emails (0.8%) were present but failed a standard RFC 5321 structural check.
* **Remediation:** Replaced Pandas date inference (which emits silent warnings on ambiguous input) with a deterministic, rule-based parser. The parser resolves format by inspecting segment magnitude: a first segment above 31 is a year, above 12 is a day, and so on. Truly ambiguous cases (39 records where both day and month are 12 or below) were defaulted to European DD/MM/YYYY, which is the dominant format in the dataset. Invalid emails were nulled via regex audit.

### 5. Accuracy
* **Issue Found:** Two records (0.4%) had negative `credit_history_months` and one record (0.2%) had a negative `savings_balance`, both of which are impossible in practice. Combined, 3 records (0.6%) contained impossible numeric values.
* **Remediation:** All 3 records (0.6%) were quarantined and removed from the clean dataset. Silently correcting negative credit history to its absolute value, or capping a negative savings balance at $0, would introduce unverifiable assumptions that could distort the downstream bias analysis. Each record is tagged with a machine-readable `_drop_reason` (`impossible_negative_credit_history` or `impossible_negative_savings_balance`) and preserved in `data/quarantined_records.json` for manual review.

### 6. Timeliness
* **Issue Found:** Five records (1.0%) stored income under the obsolete key `annual_salary` rather than the current `annual_income`, a strong indicator of schema drift from a legacy intake form.
* **Remediation:** The keys were renamed to restore structural completeness.


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


## Privacy and Governance Analysis


Credit scoring systems are classified as **high-risk AI systems under the EU AI Act**, because they significantly affect individuals’ access to financial opportunities. As a result, such systems must meet strict requirements related to **transparency, traceability, and human oversight**.

The previous analysis identified significant bias in NovaCred’s credit approval system, including disparities related to **gender, age, and geographic location**. Rejections are primarily justified using a generic label **“algorithm_risk_score,”** which does not explain how decisions are made.

From a governance perspective, it is therefore necessary to evaluate how NovaCred’s data practices align with **GDPR requirements and EU AI Act obligations**.

The dataset stores several forms of **personally identifiable information (PII)** in plain text format, including:

* Full names  
* Email addresses  
* Social Security Numbers (SSNs)  
* IP addresses  

These identifiers can directly reveal the identity of individuals. Because they are neither hashed nor encrypted, they increase the risk of identity theft or fraud. The dataset itself does not demonstrate the presence of **privacy-by-design safeguards**, suggesting weak implementation of preventive data protection controls.


### 1. Sensitive Personal Data Exposure

The dataset stores highly sensitive identifiers such as **SSNs and IP addresses in plain text format**.

* **Governance Risk:**  
Unprotected personal identifiers increase the likelihood of unauthorized access and potential identity theft. From a governance perspective, the dataset does not demonstrate embedded technical safeguards that would reduce the exposure of sensitive personal data.

* **GDPR Mapping:**

  * **Article 5(1)(c) – Data Minimization**  
  Personal data must be adequate, relevant, and limited to what is necessary for the stated purpose. In a credit scoring context, it must be questioned whether storing full SSNs and IP addresses is necessary for risk modelling.

  * **Article 32 – Security of Processing**  
  Organizations must implement appropriate technical and organizational measures to ensure the confidentiality and integrity of personal data.

  * **Article 25 – Data Protection by Design and by Default**  
  Privacy safeguards should be integrated into system architecture from the outset.

* **Recommendation:**  
NovaCred should implement a **layered data protection strategy** for sensitive identifiers. SSNs should be **pseudonymized (hashed)** before analytical use and encrypted at rest. The organization should also assess whether storing full SSNs and IP addresses is necessary for credit risk modelling.

* **Pseudonymization Demonstration:**  
The SSN field was pseudonymized using a **SHA-256 hashing function**. This process converts the original identifier into an irreversible string while maintaining consistency across records. The same SSN produces the same hashed value, but the original identifier cannot be reconstructed from the hash.

This reduces the risk of direct identification in the event of unauthorized access while preserving analytical usefulness. However, pseudonymized data **still qualifies as personal data under GDPR** if it can potentially be re-linked to individuals using additional information. Therefore, pseudonymization must be complemented by **access controls and secure storage mechanisms**.


### 2. Absence of Consent Tracking Mechanism

The dataset does not contain any field indicating **consent or lawful basis for processing personal data**.

* **Governance Risk:**  
Without a documented lawful basis, NovaCred cannot demonstrate that personal data processing complies with GDPR requirements.

* **GDPR Mapping:**

  * **Article 6 – Lawfulness of Processing**  
  Personal data may only be processed where a valid legal basis exists.

  * **Article 5(2) – Accountability**  
  Organizations must demonstrate compliance with GDPR obligations.

  * **Article 13 – Information to Data Subjects**  
  Individuals must be informed about what personal data is collected and how it will be used.

* **Recommendation:**  
NovaCred should implement a **formal lawful-basis documentation framework**. If consent is relied upon, the system should record consent status, timestamps, and allow withdrawal.


### 3. Missing Data Retention Policy

The dataset contains no fields indicating **retention limits, deletion timestamps, or lifecycle management controls**.

* **Governance Risk:**  
Without defined retention periods, personal data may be stored indefinitely, increasing regulatory risk.

* **GDPR Mapping:**

  * **Article 5(1)(e) – Storage Limitation**  
  Personal data must not be kept longer than necessary for the purpose for which it is processed.

* **Recommendation:**  
NovaCred should define **clear retention periods** for each category of personal data and implement automated deletion or anonymization mechanisms.


### 4. Lack of Audit Trail for Automated Decisions

Rejected applications contain only the generic label **“algorithm_risk_score.”**

* **Governance Risk:**  
The absence of detailed explanations prevents individuals and regulators from understanding how automated credit decisions are produced.

* **GDPR Mapping:**

  * **Article 22 – Automated Decision-Making**  
  Individuals must receive meaningful information about the logic involved in automated decisions.

  * **Article 5(1)(a) – Transparency**  
  Personal data must be processed in a lawful, fair, and transparent manner.

* **EU AI Act Mapping:**

  * **High-Risk Classification**  
  Credit scoring systems fall within the high-risk AI category.

  * **Logging and Documentation Requirements**  
  High-risk AI systems must maintain records allowing automated decisions to be reviewed and audited.

  * **Human Oversight Requirements**  
  High-risk systems must allow effective human supervision.

* **Recommendation:**  
NovaCred should implement **decision explanation and logging mechanisms** that record risk score calculations, key variables influencing the outcome, and the model version used.


### 5. Lack of Documented Human Oversight

The dataset contains no fields indicating that automated credit decisions are subject to **human review or intervention**.

* **Governance Risk:**  
Fully automated credit decisions affecting individuals’ financial opportunities may conflict with regulatory safeguards.

* **GDPR Mapping:**

  * **Article 22 – Automated Decision-Making**  
  Individuals must have the possibility of meaningful human intervention.

  * **Article 5(2) – Accountability**  
  Organizations must demonstrate that automated decisions are properly supervised.

* **EU AI Act Mapping:**

  * **Human Oversight Requirements for High-Risk AI Systems**  
  High-risk AI systems must allow human supervision and intervention.

* **Recommendation:**  
NovaCred should introduce **structured human oversight procedures**, including manual review processes and override mechanisms.


### 6. Sensitive Behavioral Data Collection

The dataset includes a **spending_behavior** variable containing detailed information about individuals’ consumption patterns.

* **Governance Risk:**  
Detailed behavioral profiling may exceed what is necessary for credit risk assessment and raises privacy concerns.

* **GDPR Mapping:**

  * **Article 5(1)(c) – Data Minimization**  
  Only data necessary for the intended purpose should be collected.

  * **Article 5(1)(a) – Fairness and Transparency**  
  Individuals must understand how their data influences automated decisions.

  * **Article 13 – Information to Data Subjects**  
  Applicants must be informed if behavioral data is used in credit scoring.

* **EU AI Act Mapping:**

  * **Data Governance and Data Quality Requirements**  
  High-risk AI systems must use relevant and appropriate data.

* **Recommendation:**  
NovaCred should review whether detailed behavioral spending data is necessary for credit risk assessment. If used, the company should clearly document its relevance, limit the level of detail collected, and inform applicants that spending behavior may influence credit decisions.