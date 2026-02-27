# TXC10-GP-DEGO
TXC10 - DEGO 2606 Group Project – Credit Application Governance Analysis

## Data Engineering Pipeline: Credit Applications

This repository contains a robust Data Engineering pipeline designed to process, clean, and validate raw credit application data using **Python** and **MongoDB**. The pipeline transforms messy, unstructured data into a pristine dataset ready for machine learning and bias analysis.



### Pipeline Overview

The workflow is divided into four critical stages:

#### 1. Data Loading
* Connects to a local MongoDB instance (`novacred_db`).
* Parses raw JSON data and enforces uniqueness by dropping duplicate application `_id`s upon insertion.

#### 2. Data Cleaning
A comprehensive quality assurance process addressing six core issues:
* **Duplicate Records:** Deduplication of resubmitted applications.
* **Inconsistent Data Types:** Dynamic auditing and casting of mixed-type fields (e.g., converting string representations of income to integers).
* **Missing Data & Schema Drift:** Drops structurally invalid records (missing SSNs), imputes missing demographic data with "UNKNOWN" placeholders, and resolves schema drift by mapping legacy keys (`annual_salary`) to the official data dictionary (`annual_income`).
* **Categorical Formatting:** Standardizes inconsistent categorical codes (e.g., dynamically mapping "M"/"F" to "Male"/"Female").
* **Impossible Values:** Applies strict governance rules to correct negative numbers (e.g., applying absolute values to negative time metrics and flooring overdrawn savings accounts at zero).
* **Inconsistent Dates:** Uses regex and MongoDB string replacement to standardize non-compliant date strings to the strict ISO 8601 format (`YYYY-MM-DD`).

#### 3. Schema Validation
* Enforces a strict MongoDB `$jsonSchema` based on the official project Data Dictionary.
* Guarantees that all required nested objects, critical identifiers, and correct data types are present. The final audit confirms **0 invalid documents**.

#### 4. Data Export
Extracts the perfectly clean database into two distinct formats for different stakeholders:
* **JSON:** A nested `clean_credit_applications.json` file for system backups and application developers.
* **CSV:** A flattened, tabular `clean_credit_applications.csv` file (flattened via `pandas.json_normalize`) designed specifically for the Data Science team's analytical models.

### Key Results
* **Raw Records Processed:** 500
* **Final Clean Records:** 495 perfectly validated, structurally sound records ready for analysis.