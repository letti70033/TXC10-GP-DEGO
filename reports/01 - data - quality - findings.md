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