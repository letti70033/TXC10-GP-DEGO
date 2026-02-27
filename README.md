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