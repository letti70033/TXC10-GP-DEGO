# Credit Application Governance Analysis
**DEGO 2606 Group Project | MSc Business Analytics | Nova SBE**

> **Scenario:** NovaCred, a fintech startup using ML for credit decisions, received a regulatory inquiry about potential discrimination in its lending practices. Our team acted as a **Data Governance Task Force**, auditing 502 raw credit applications for data quality issues, algorithmic bias, and GDPR/AI Act compliance gaps.

**Video Presentation:** [watch here](https://www.youtube.com/watch?v=Tiepuh7psn0)

---

## Repository Structure

```
TXC10-GP-DEGO/
├── README.md                              # This file — executive summary & governance report
├── data/
│   ├── raw_credit_applications.json       # Source dataset (502 records)
│   ├── clean_credit_applications.csv      # Clean output (487 records, CSV)
│   ├── clean_credit_applications.json     # Clean output (487 records, JSON)
│   └── quarantined_records.json           # Removed records with drop-reason tags
├── notebooks/
│   ├── 01 - data - quality.ipynb          # Data Engineering & cleaning pipeline
│   ├── 02 - bias - analysis.ipynb         # Bias detection & fairness metrics
│   └── 03 - privacy - demo.ipynb          # PII audit, pseudonymization & GDPR mapping
└── reports/
    ├── 01 - data - quality - findings.md  # Detailed data quality methodology & findings
    ├── 02 - bias - analysis - findings.md # Detailed bias analysis methodology & findings
    ├── 03 - privacy - demo - findings.md  # Detailed privacy & governance findings
    └── TXC10-DEGO_presentation.pdf        # Slide deck presentation
```

---

## Team

| Role | Student |
|---|---|
| **Data Engineer** | 70485 Lennart Stenzel |
| **Data Scientist** | 70033 Leticia Brendle |
| **Governance Officer** | 72120 Laura Rodrigues |

> Each team member led one analytical workstream: Lennart Stenzel (Data Engineering & quality pipeline), Leticia Brendle (Bias detection & fairness analysis), and Laura Rodrigues (Privacy assessment & governance). All members contributed to the repository and the Product Lead role was shared collectively.

---

## Executive Summary

| Area | Key Finding |
|---|---|
| Data Quality | 15 records quarantined; 6 issue categories identified across 502 records |
| Bias — Gender | DI = **0.772** (fails the four-fifths rule); 15.1 pp gap favouring men |
| Bias — Age | Applicants under 30 approved at only **41.5%** vs 60–65% for older groups | 
| Bias — Geography | NYC-area (100xx) approved at **64.3%** vs LA-area (902xx) at **52.0%** | 
| Privacy | 4 PII fields stored in plain text; no consent tracking; no retention policy |
| Governance | No audit trail for decisions; no human oversight documentation |

NovaCred's credit scoring system exhibits statistically confirmed gender and age bias, stores unprotected sensitive personal data, and lacks the transparency and oversight mechanisms required under GDPR and the EU AI Act. Immediate remediation is required to avoid regulatory liability.

---

## 1. Data Quality Analysis

**Notebook:** `01 - data - quality.ipynb`

Starting from **502 raw records**, the pipeline produced **487 clean records** and quarantined **15 records** with machine-readable drop-reason tags in `data/quarantined_records.json`.

### Issues Identified

| Dimension | Issue | Records Affected | % of Total | Remediation |
|---|---|---|---|---|
| **Uniqueness** | Duplicate `_id` fields | 2 | 0.4% | Kept most recent; quarantined older duplicate |
| **Uniqueness** | SSN conflicts (different names, same SSN) | 4 | 0.8% | Both records quarantined — unverifiable without external evidence |
| **Completeness** | Missing SSNs | 5 | 1.0% | Records with missing KYC fields dropped |
| **Completeness** | Missing date of birth | 1 | 0.2% | Dropped (KYC-required) |
| **Completeness** | Missing / malformed emails | 6 | 1.2% | Set to `null` (honest representation) |
| **Completeness** | Missing zip codes | 2 | 0.4% | Planned imputation with `"UNKNOWN_ZIP"` not required — both records were already removed during SSN drop |
| **Completeness** | Schema drift (`annual_salary` key) | 5 | 1.0% | Key renamed to `annual_income` |
| **Consistency** | Gender coded as 'M'/'F' vs 'Male'/'Female' | 109 | 21.7% | Standardized to full-word equivalents |
| **Consistency** | `annual_income` stored as string instead of number | 7 | 1.4% | Cast to integer via MongoDB aggregation pipeline |
| **Validity** | Non-standard date formats in `date_of_birth` | 156 | 31.1% | Rule-based deterministic parser (ambiguous cases default to DD/MM/YYYY) |
| **Validity** | Malformed emails (failed RFC 5321 check) | 4 | 0.8% | Set to `null` via regex audit |
| **Accuracy** | Negative `credit_history_months` | 2 | 0.4% | Quarantined |
| **Accuracy** | Negative `savings_balance` | 1 | 0.2% | Quarantined |
| **Timeliness** | Legacy `annual_salary` key (schema drift) | 5 | 1.0% | Renamed to restore structural completeness |

### Key Design Decisions
- **Quarantine over deletion:** All removed records are preserved in `quarantined_records.json` with structured `_drop_reason` tags, enabling manual review and full auditability.
- **Null over placeholder:** Missing emails set to `null` rather than `"UNKNOWN"` — a more honest and GDPR-consistent representation.
- **No silent correction:** Negative credit history months were not flipped to absolute values; silently altering impossible values would introduce unverifiable assumptions distorting downstream bias analysis.

---

## 2. Bias Detection & Fairness Analysis

**Notebook:** `02 - bias - analysis.ipynb`

Analysis performed on the 487-record clean dataset using `pandas`, `scipy`, and `fairlearn`.

### 2.1 Gender Disparate Impact

| Metric | Value | Interpretation |
|---|---|---|
| Male approval rate | 65.9% | Privileged group |
| Female approval rate | 50.8% | Unprivileged group |
| **Disparate Impact Ratio** | **0.772** | **Fails four-fifths rule (threshold: 0.8)** |
| Demographic Parity Difference | −0.151 | 15.1 pp gap in favour of male applicants |
| Statistical significance | χ², p = 0.001 | Confirmed significant |

The DI ratio of **0.772** falls below the legal four-fifths threshold, constituting evidence of potential disparate impact discrimination. Validated independently with `fairlearn.metrics.demographic_parity_difference`.

### 2.2 Age-Based Discrimination

| Age Group | Approval Rate | Notes |
|---|---|---|
| < 30 | **41.5%** | Significantly lower |
| 30–44 | ~62% | Largest group (256 records) |
| 45–59 | ~65% | |
| 60+ | ~60% | Small sample (43 records) |

Age gap confirmed statistically significant (χ², **p = 0.008**).

### 2.3 Proxy Discrimination

| Variable | Proxies For | Finding | Evidence |
|---|---|---|---|
| `credit_history_months` | Age | **Confirmed proxy** | r = 0.65 with applicant age, p < 0.001 — young applicants structurally penalised for a variable they cannot improve |
| `zip_code` | Geography/race | **Confirmed proxy** | NYC (100xx): 64.3% approval vs LA (902xx): 52.0% (χ², p = 0.025), not explained by financial profiles alone |
| `annual_income` | Gender | Hypothesis rejected | Male ($81,358) vs Female ($83,599), t-test p = 0.38 — no significant gap |
| `savings_balance` | Gender | Hypothesis rejected | No significant gender wealth gap detected |

### 2.4 Interaction Effects

Women under 30 face compounding disadvantage:

| Group | Approval Rate | DI Ratio |
|---|---|---|
| Men under 30 | 50.0% | — |
| Women under 30 | **34.1%** | **0.682** |

A DI of **0.682** — far below the 0.8 threshold — is consistent with **intersectional discrimination** where gender and age effects compound rather than add linearly.

---

## 3. Privacy Assessment & GDPR Compliance

**Notebook:** `03 - privacy - demo.ipynb`

Credit scoring systems are classified as **high-risk AI systems under the EU AI Act (Annex III)**, triggering elevated obligations for transparency, data governance, and human oversight.

### 3.1 PII Inventory

| Field | Type | Stored As | Risk Level |
|---|---|---|---|
| `applicant_info.full_name` | Direct identifier | Plain text | High |
| `applicant_info.email` | Direct identifier | Plain text | High |
| `applicant_info.ssn` | Highly sensitive identifier | **Plain text** | **Critical** |
| `applicant_info.ip_address` | Quasi-identifier | Plain text | Medium |
| `applicant_info.date_of_birth` | Quasi-identifier | Plain text | Medium |

None of these fields are hashed, encrypted, or pseudonymized in the raw dataset.

### 3.2 Pseudonymization Demonstration

SSNs were pseudonymized using **SHA-256 hashing**, converting each identifier into an irreversible fixed-length string. The same SSN produces the same hash (preserving analytical consistency), but the original value cannot be reconstructed.

> Note: Pseudonymized data **still qualifies as personal data under GDPR** if re-linkage is possible. Pseudonymization must be complemented by access controls and secure storage.

### 3.3 Governance Gap Analysis

| Gap | GDPR Article | EU AI Act | Risk |
|---|---|---|---|
| PII stored in plain text | Art. 32 (Security), Art. 25 (Privacy by Design) | Data governance requirements | Critical |
| No lawful basis / consent tracking | Art. 6 (Lawfulness), Art. 5(2) (Accountability), Art. 13 (Transparency) | — | High |
| No data retention policy | Art. 5(1)(e) (Storage Limitation) | — | High |
| Rejections justified only as "algorithm_risk_score" | Art. 22 (Automated Decision-Making), Art. 5(1)(a) (Transparency) | Logging & explainability requirements | High |
| No human oversight documentation | Art. 22 (Right to Human Intervention) | Human oversight requirements (high-risk AI) | High |
| Detailed spending behavior collected without stated necessity | Art. 5(1)(c) (Data Minimization), Art. 13 | Data quality requirements | Medium |

---

## 4. Governance Recommendations

### Immediate Actions (0–3 months)

1. **Pseudonymize / encrypt all PII at rest.** SSNs and IP addresses must be hashed before any analytical use. Implement field-level encryption for production storage.
2. **Add a lawful basis field** to every application record. Document whether processing relies on consent, legitimate interest, or contractual necessity. Enable consent withdrawal.
3. **Replace "algorithm_risk_score" with structured rejection reasons.** Each record must log the top contributing variables, model version, and decision timestamp (GDPR Art. 22 / AI Act logging requirements).

### Medium-Term Actions (3–12 months)

4. **Implement a data retention policy.** Define maximum retention periods per data category; automate deletion or anonymization at end-of-life (GDPR Art. 5(1)(e)).
5. **Introduce structured human review for borderline cases.** Document the review process and override mechanisms to satisfy EU AI Act human oversight requirements for high-risk systems.
6. **Retrain the credit scoring model** after removing or re-weighting `credit_history_months` and `zip_code` as features, or apply fairness constraints (e.g., equalized odds) to reduce their discriminatory proxy effect.

### Strategic Actions (12+ months)

7. **Conduct a Data Protection Impact Assessment (DPIA)** covering the full ML pipeline, from data collection through decision output, as required for high-risk AI under the EU AI Act.
8. **Establish an audit trail system** that logs all automated credit decisions with sufficient granularity for regulatory review and individual appeals.
9. **Review `spending_behavior` data collection.** If behavioral profiling is not demonstrably necessary for credit risk, its collection should be reduced or discontinued under the data minimization principle.

---

## 5. Technical Stack

| Purpose | Library |
|---|---|
| Data manipulation | `pandas`, `numpy` |
| Database / schema validation | `pymongo` (MongoDB) |
| Statistical testing | `scipy.stats` (χ², t-test, Pearson, Kruskal-Wallis) |
| Fairness metrics | `fairlearn` (`demographic_parity_difference`, `MetricFrame`) |
| Visualization | `matplotlib`, `seaborn` |
| Pseudonymization | `hashlib` (SHA-256) |

---
