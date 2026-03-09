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