# Real-Time Payment Fraud Prevention System (RTPFPS)
## Business Requirements Document

> **Version:** 1.0 — Baseline Draft
> **Status:** Baselined – Awaiting Sign-Off
> **Date:** March 06, 2026
> **Project:** RTPFPS – Phase 1
> **Total Requirements:** 38 Business Requirements

---

## Table of Contents

1. [Introduction & Document Purpose](#1-introduction--document-purpose)
2. [Project Overview & Context](#2-project-overview--context)
3. [Stakeholder Analysis](#3-stakeholder-analysis)
4. [Business Requirements](#4-business-requirements)
5. [Non-Functional Requirements](#5-non-functional-requirements-nfrs)
6. [Business Rules](#6-business-rules)
7. [Assumptions, Dependencies & Constraints](#7-assumptions-dependencies--constraints)
8. [Requirements Traceability Matrix](#8-requirements-traceability-matrix-rtm)
9. [Sign-Off & Approval](#9-sign-off--approval)
10. [Document Control & Glossary](#10-document-control--glossary)

---

## 1. Introduction & Document Purpose

This Business Requirements Document (BRD) captures the complete set of business, stakeholder, and non-functional requirements for the **Real-Time Payment Fraud Prevention System (RTPFPS)**. It defines **what the business needs** the system to achieve — not how it will be technically built (addressed in the Functional Requirements Document).

This document is the primary reference for all stakeholders throughout the project lifecycle and forms the contractual baseline between the business and the delivery team.

### 1.1 Document Objectives

- Capture all business and stakeholder requirements in a structured, traceable format
- Define the scope boundary — what is and is not required from the system
- Establish measurable acceptance criteria for each requirement
- Serve as the baseline for change control throughout the project
- Enable the development of the Functional Requirements Document (FRD) and test plans

### 1.2 How to Use This Document

| Audience | How to Use |
|---|---|
| Business Stakeholders / Sponsor | Review and sign off requirements in Sections 4–7 to confirm they accurately represent your needs |
| Business Analyst | Use as the baseline document for all requirements; manage changes via change control |
| Development / Engineering Team | Use as input to technical design and the FRD; raise queries via formal change request |
| Test / QA Team | Derive test cases from the acceptance criteria in each requirement |
| Compliance & Legal | Review Section 7 (Regulatory Requirements) for audit and compliance sign-off |

---

## 2. Project Overview & Context

### 2.1 Project Background

The organisation currently processes over **2 million payment transactions per day** across mobile banking, online banking, POS terminals, and API-connected third-party services. Fraud detection is currently handled by static rule sets and manual review — unable to detect fraud in real time or adapt to evolving patterns.

The RTPFPS is a strategic initiative to replace this capability with an **intelligent, automated, and scalable platform** that monitors every transaction at the point of processing and intervenes when fraud is detected.

### 2.2 Business Objectives

| Ref | Business Objective | Success Metric |
|---|---|---|
| BO-01 | Detect and prevent fraudulent transactions in real time | < 300ms average detection latency |
| BO-02 | Reduce direct fraud financial losses | 40% reduction in Year 1; 65% by Year 2 |
| BO-03 | Reduce false-positive rate on flagged transactions | < 2% false positive rate |
| BO-04 | Increase fraud analyst operational efficiency | 30% reduction in manual case handling workload |
| BO-05 | Achieve full regulatory compliance | 100% audit trail coverage across all flagged transactions |
| BO-06 | Improve customer trust and experience | 25% reduction in fraud-related customer complaints |

### 2.3 Scope

#### ✅ In Scope
- Real-time transaction monitoring (all digital channels)
- Rules-based fraud detection engine
- ML-augmented anomaly detection
- Fraud analyst case management dashboard
- Automated alerts (email, SMS, in-app)
- Integration with core banking, card, and identity systems
- Compliance audit logging and reporting
- UAT and fraud operations team training

#### ❌ Out of Scope
- Chargeback and dispute management
- Core banking system replacement
- Non-payment fraud (loans, insurance)
- Customer-facing fraud self-report portal *(Phase 2)*
- International regulatory compliance outside UK/EU

---

## 3. Stakeholder Analysis

| Stakeholder | Role | Primary Needs / Goals | Engagement Method | Req. Refs |
|---|---|---|---|---|
| Chief Risk Officer | Sponsor | Real-time loss prevention, regulatory assurance, board-level KPI reporting | 1:1 Interview | BR-01–05 |
| Head of Fraud Operations | Business Owner | Efficient analyst workflow, accurate fraud scoring, manageable alert volumes, audit trails | Workshop ×3 | BR-06–15 |
| Fraud Analysts (8 users) | Primary End Users | Clear case queue, transaction context, decision tools, fast resolution workflow, notes & audit log | Focus Group | BR-06–12 |
| Compliance & Legal | Advisor | Full regulatory data retention, PCI-DSS/PSD2 compliance, GDPR-safe data handling | Document Review + Interview | BR-28–35 |
| IT / Engineering Lead | Technical Owner | Clear API specs, low-latency SLAs, scalability requirements, integration standards | Workshop ×2 | BR-16–22 |
| Data Science Team | SME | Access to clean historical data, model retraining cadence, champion-challenger capability | Workshop ×1 | BR-23–27 |
| Customer Experience Team | Stakeholder | Minimal false positives, clear customer communication on blocked transactions | Survey + Interview | BR-13–14 |
| Finance Team | Stakeholder | Cost tracking, fraud loss reporting, chargeback data feed | 1:1 Interview | BR-36–38 |

---

## 4. Business Requirements

> **MoSCoW Priority Key:**
> - 🔴 **Must Have** — Critical; system cannot go live without this
> - 🟡 **Should Have** — Important; strong business value, include if possible
> - 🟢 **Could Have** — Desirable; included if time and budget allow

### 4.1 Fraud Detection & Monitoring

| Req. ID | Requirement Statement | Priority | Source | Acceptance Criteria |
|---|---|---|---|---|
| BR-01 | The system must monitor 100% of payment transactions across all digital channels at the point of processing, before a transaction is completed. | 🔴 Must Have | CRO / BO-01 | All transactions flagged or cleared within 300ms; zero transactions bypass the system |
| BR-02 | The system must assign a fraud risk score (0–100) to every transaction, based on a combination of rules and ML model outputs. | 🔴 Must Have | Fraud Ops | Every transaction has a risk score; score updates in real time as new signals are received |
| BR-03 | The system must automatically block or hold transactions that exceed a configurable fraud risk threshold, pending analyst review. | 🔴 Must Have | CRO / Fraud Ops | Transactions above threshold are blocked within 300ms; threshold is configurable by authorised admin |
| BR-04 | The system must cover all payment channels including mobile app, online banking, POS, ATM, and open banking API payments. | 🔴 Must Have | BO-01 | Test transactions on each channel are correctly monitored and scored in UAT |
| BR-05 | The system must include a configurable rules engine enabling fraud operations to create, edit, activate, and deactivate fraud detection rules without requiring IT changes. | 🔴 Must Have | Head of Fraud Ops | Fraud analyst can create and activate a new rule within 10 minutes with no IT involvement |
| BR-06 | The system should build and maintain a behavioural profile per customer to detect anomalies such as unusual transaction times, locations, amounts, and merchant categories. | 🟡 Should Have | Data Science | Anomaly triggers fire correctly when transaction deviates from 90-day behavioural baseline |
| BR-07 | The system must detect and flag velocity-based fraud patterns, including multiple transactions in a short time window, rapid account balance drain, and repeated declined transactions. | 🔴 Must Have | Fraud Ops | Velocity rules fire within configured time windows in UAT with 100% accuracy |
| BR-08 | The system should incorporate device fingerprinting and geolocation data to flag transactions from unrecognised devices or geographically impossible journeys. | 🟡 Should Have | IT Lead / Fraud Ops | Device and location signals are captured and included in fraud score for relevant channels |
| BR-09 | The system must include specific detection logic for Card-Not-Present (CNP) fraud, including 3DS challenge triggers for suspicious online card transactions. | 🔴 Must Have | Fraud Ops / Compliance | CNP fraud scenarios from UAT test pack are correctly flagged with appropriate 3DS challenge |
| BR-10 | The system should flag accounts exhibiting synthetic identity fraud signals such as no transaction history, rapid credit utilisation, and mismatched identity data. | 🟡 Should Have | Fraud Ops | Test scenarios for synthetic identity fraud are flagged correctly in UAT |

### 4.2 Case Management & Analyst Workflow

| Req. ID | Requirement Statement | Priority | Source | Acceptance Criteria |
|---|---|---|---|---|
| BR-11 | The system must provide fraud analysts with a real-time case management dashboard showing all flagged transactions, prioritised by risk score. | 🔴 Must Have | Fraud Analysts | Dashboard loads within 3 seconds; displays all open cases with risk score, amount, and channel |
| BR-12 | Analysts must be able to view full transaction context including customer profile, account history, device data, location, and the fraud signals that triggered the alert. | 🔴 Must Have | Fraud Analysts | All context data visible in a single case view; no need to access other systems |
| BR-13 | Analysts must be able to approve, decline, or escalate a flagged transaction directly from the case management interface. | 🔴 Must Have | Fraud Analysts | All three actions available in case view; decision is applied to transaction within 60 seconds |
| BR-14 | Every analyst decision must be logged with a timestamp, analyst ID, decision made, and optional free-text note. This log must be immutable. | 🔴 Must Have | Compliance | Audit log entries cannot be edited or deleted; all decisions captured in real time |
| BR-15 | When a transaction is blocked, the system should automatically trigger a customer notification (SMS and in-app message) within 60 seconds of the block. | 🟡 Should Have | CX Team | Customer notification received within 60 seconds of block in UAT test scenarios |
| BR-16 | The system should support rule-based automatic assignment of fraud cases to analysts based on workload, specialisation, and case type. | 🟡 Should Have | Head of Fraud Ops | Cases are distributed evenly across active analysts; reassignment is possible by supervisors |
| BR-17 | Analysts must be able to escalate complex cases to a senior analyst or supervisor, with case ownership transferred and full history retained. | 🔴 Must Have | Fraud Ops | Escalation completes within 30 seconds; full case history visible to receiving analyst |
| BR-18 | Analysts must tag every resolved case with a fraud outcome (confirmed fraud, false positive, suspicious but approved) to support model training and reporting. | 🔴 Must Have | Data Science / Fraud Ops | Outcome tag is mandatory before case can be closed; all tag options available in UAT |

### 4.3 Alerting, Notification & Integration

| Req. ID | Requirement Statement | Priority | Source | Acceptance Criteria |
|---|---|---|---|---|
| BR-19 | The system must generate real-time alerts to the fraud operations team for all high-risk transactions (score > 80) within 30 seconds of detection. | 🔴 Must Have | Fraud Ops | All high-risk transactions generate alerts within 30 seconds in performance testing |
| BR-20 | Fraud operations managers must be able to configure alert delivery channels (email, SMS, dashboard notification, Slack/Teams) per alert type and severity. | 🟡 Should Have | Head of Fraud Ops | Configuration UI allows channel selection per rule; changes are effective immediately |
| BR-21 | The system must integrate with the core banking platform via REST API to access account balance, transaction history, and to place transaction holds. | 🔴 Must Have | IT Lead | Integration passes 100% of API integration test cases; data returned within 200ms |
| BR-22 | The system must integrate with the card management system to enable real-time card block, unblock, and 3DS challenge triggers. | 🔴 Must Have | IT Lead / Fraud Ops | Card block/unblock actions are reflected in card management system within 5 seconds |
| BR-23 | The system should integrate with the identity verification and KYC platform to cross-reference customer identity data during fraud scoring. | 🟡 Should Have | Compliance | KYC data is returned and used in fraud score within 500ms; integration tested in UAT |
| BR-24 | The system should integrate with Visa/Mastercard fraud data feeds to enrich transaction risk scoring with network-level intelligence. | 🟢 Could Have | IT Lead | Network data feed is consumed and contributes to fraud score when available |
| BR-25 | The system must publish real-time fraud events to an event streaming platform (e.g., Kafka) to enable downstream system integration and audit logging. | 🔴 Must Have | IT Lead | All fraud events are published to event stream within 100ms of decision; format is documented |

### 4.4 Reporting, Analytics & Management Information

| Req. ID | Requirement Statement | Priority | Source | Acceptance Criteria |
|---|---|---|---|---|
| BR-26 | The system must provide a management information dashboard showing daily, weekly, and monthly fraud KPIs including fraud volume, value, catch rate, and false positive rate. | 🔴 Must Have | CRO / Fraud Ops | Dashboard available to senior users; data refreshes every 15 minutes; all 6 KPIs displayed |
| BR-27 | The system must support scheduled automated reports (daily, weekly, monthly) delivered by email to configurable recipient lists. | 🔴 Must Have | Finance / Compliance | Reports generated on schedule without manual intervention; recipient list is configurable |
| BR-28 | Fraud analysts and managers must be able to run ad-hoc queries on transaction data and export results in CSV and PDF formats. | 🟡 Should Have | Fraud Ops / Finance | Query results exportable in both formats; export of 10,000 rows completes within 60 seconds |
| BR-29 | The system should provide a model performance report showing precision, recall, F1 score, and false positive rate for each ML model in production. | 🟡 Should Have | Data Science | Model performance report available to data science team; updates daily after batch scoring run |
| BR-30 | The system must generate pre-formatted regulatory reports aligned to FCA, PSD2, and PCI-DSS requirements, including Suspicious Activity Report (SAR) support. | 🔴 Must Have | Compliance | All regulatory report templates validated by Compliance team during UAT |

### 4.5 Regulatory, Compliance & Data

| Req. ID | Requirement Statement | Priority | Source | Acceptance Criteria |
|---|---|---|---|---|
| BR-31 | All fraud decisions, alerts, analyst actions, and system events must be stored in an immutable audit log retained for a minimum of 7 years. | 🔴 Must Have | Compliance / FCA | Audit log entries pass immutability test; 7-year retention policy confirmed by storage architecture review |
| BR-32 | The system must only collect and process personal data that is necessary for fraud detection; all personal data must be identifiable as a specific data category. | 🔴 Must Have | Legal / Compliance | Data audit confirms no excess personal data collected; data categories mapped in DPIA |
| BR-33 | All personal and financial data must be encrypted at rest (AES-256) and in transit (TLS 1.3 or above). | 🔴 Must Have | IT / Compliance | Encryption standards confirmed by security penetration test; no unencrypted data found |
| BR-34 | The system must be designed and operated in compliance with PCI-DSS v4.0 standards for all cardholder data environments. | 🔴 Must Have | Compliance | PCI-DSS compliance confirmed by QSA audit prior to go-live |
| BR-35 | The system must support the ability to erase an individual customer's personal data upon a valid GDPR erasure request, except where legal retention obligations apply. | 🔴 Must Have | Legal | Erasure workflow tested end-to-end; audit trail of erasure retained; legal hold categories correctly excluded |
| BR-36 | For transactions that trigger a fraud hold, the system must support Strong Customer Authentication (SCA) challenges compliant with PSD2 RTS requirements. | 🔴 Must Have | Compliance / IT | SCA flows pass PSD2 RTS compliance checklist validated by Compliance team |

### 4.6 Financial Reporting & Operational

| Req. ID | Requirement Statement | Priority | Source | Acceptance Criteria |
|---|---|---|---|---|
| BR-37 | The system must produce a daily fraud loss report showing total value and volume of confirmed fraud by channel, fraud type, and customer segment. | 🔴 Must Have | Finance / CRO | Daily report produced automatically by 08:00; Finance team validates content in UAT |
| BR-38 | The system should provide a data feed of confirmed fraud cases to the chargeback processing team to initiate recovery actions. | 🟡 Should Have | Finance | Chargeback data feed tested end-to-end with Finance team's receiving system in UAT |

---

## 5. Non-Functional Requirements (NFRs)

> NFRs define **how well** the system performs — quality attributes and constraints verified during testing.

| NFR ID | Category | Attribute | Requirement | Test Approach |
|---|---|---|---|---|
| NFR-01 | Performance | Transaction Latency | Process and score each transaction within **300ms (P95)** under normal operating conditions | Performance testing at 2× peak load |
| NFR-02 | Performance | Throughput | Support a minimum of **2,000 TPS** at peak load | Load testing with simulated peak traffic |
| NFR-03 | Availability | System Uptime | Achieve **99.95% availability** (max ~4.4 hours downtime/year) | Monitoring reports; SLA verified over 30-day UAT |
| NFR-04 | Availability | Recovery Time | **RTO < 30 minutes; RPO < 5 minutes** | DR failover test in testing environment |
| NFR-05 | Scalability | Horizontal Scaling | Automatically scale to handle a **3× increase** in transaction volume without manual intervention | Stress test at 3× peak load; auto-scale verified |
| NFR-06 | Security | Access Control | **RBAC enforced**; analysts can only access cases in their queue | Security test; pen test of RBAC model |
| NFR-07 | Security | Multi-Factor Auth | All users must authenticate via **MFA**; SSO with organisation's identity provider required | MFA enforcement tested for all user roles |
| NFR-08 | Security | Penetration Testing | Must pass external pen test with **no Critical or High severity findings** before go-live | External pen test report accepted by CISO |
| NFR-09 | Usability | Analyst UI Response | Dashboard loads within **3 seconds**; all analyst actions respond within **2 seconds** | UI performance test with 50 concurrent sessions |
| NFR-10 | Usability | Accessibility | UI must comply with **WCAG 2.1 AA** accessibility standards | Automated scan + manual review |
| NFR-11 | Maintainability | Rule Configuration | Fraud rules configurable by trained staff **without code changes or IT tickets** | Fraud analyst creates and activates rule within 10 minutes unaided |
| NFR-12 | Auditability | Log Completeness | System logs must capture every transaction decision, user action, and system event with timestamp, user ID, and outcome | Log completeness verified against audit checklist by Compliance |
| NFR-13 | Portability | Data Export | All data must be exportable in a standard open format **(CSV, JSON, XML)** to prevent vendor lock-in | Full data export performed and validated in UAT |

---

## 6. Business Rules

| Rule ID | Rule Name | Business Rule Statement | Applies To |
|---|---|---|---|
| BRU-01 | High Risk Auto-Block | Any transaction with a fraud risk score of **≥ 85** must be automatically blocked and placed in the analyst queue before processing can proceed. | All channels |
| BRU-02 | Medium Risk Challenge | Transactions with a fraud risk score of **60–84** must trigger an SCA challenge before processing is permitted. | Card & digital channels |
| BRU-03 | Low Risk Pass-Through | Transactions with a fraud risk score **< 60** may proceed without interruption but must still be logged and retained. | All channels |
| BRU-04 | Analyst Review SLA | All blocked transactions must be reviewed by a fraud analyst within **30 minutes**. Breaches are automatically escalated to a supervisor. | Case management |
| BRU-05 | Repeated Decline Rule | **3 or more consecutive declines** on the same card within a **10-minute window** must trigger a card block and analyst case creation. | Card channels |
| BRU-06 | Whitelist Exemption | Transactions from accounts on an approved whitelist are exempt from automated blocking but must still be scored and logged. | All channels |
| BRU-07 | International Transaction Flag | All transactions from a country not matching the customer's registered address must receive a **minimum risk score uplift of +15 points**. | All channels |
| BRU-08 | New Device Rule | The first transaction on a previously unregistered device must receive a **minimum risk score uplift of +20 points**. | Mobile / Online |
| BRU-09 | SAR Filing Trigger | Confirmed fraud cases exceeding **£10,000** in value must automatically generate a SAR draft for Compliance review within **24 hours**. | Case management |
| BRU-10 | Data Retention Rule | Fraud case records, decision logs, and audit trails must be retained for a **minimum of 7 years**, then securely deleted. | All data stores |
| BRU-11 | False Positive Feedback Loop | Cases tagged as false positives must be automatically fed back to the ML model training pipeline within **24 hours**. | ML model |
| BRU-12 | Rule Activation Approval | New or changed fraud detection rules must be reviewed and approved by the **Head of Fraud Operations** before production activation. | Rules engine |

---

## 7. Assumptions, Dependencies & Constraints

### 7.1 Assumptions

| Ref | Assumption | Impact if Incorrect |
|---|---|---|
| ASS-01 | All transaction data is accessible via REST API from the core banking system in real time. | BR-21 and all real-time detection requirements would need re-scoping |
| ASS-02 | A minimum of 24 months of historical transaction data is available in a clean, labelled format for ML model training. | BR-06 and BR-29 could not be delivered as specified; model accuracy may be lower |
| ASS-03 | The organisation's cloud infrastructure (AWS/Azure) can support sub-300ms processing at 2,000 TPS. | NFR-01 and NFR-02 may require infrastructure uplift investment |
| ASS-04 | Fraud operations team members will be available for requirements workshops, UAT, and training during the project. | Delays to requirements sign-off and UAT completion |
| ASS-05 | The Head of Fraud Operations has authority to approve business rules and sign off requirements on behalf of the Fraud Operations team. | Additional approval steps may delay requirements baseline |
| ASS-06 | GDPR, PCI-DSS v4.0, and PSD2 regulatory requirements remain stable throughout the project delivery period. | Section 5 compliance requirements may need revision; regulatory gap analysis required |

### 7.2 Dependencies

| Ref | Dependency | Owner | Required By |
|---|---|---|---|
| DEP-01 | Core banking API specification and sandbox environment available for integration design | IT Lead | Phase 2 (May 2026) |
| DEP-02 | Historical transaction dataset (24 months, labelled) provided by Data Engineering team | Data Engineering | Phase 3 (July 2026) |
| DEP-03 | Card management system API documentation and test environment available | IT Lead / Card Platform | Phase 3 (Aug 2026) |
| DEP-04 | Identity/KYC platform integration specification confirmed by Compliance | Compliance / IT | Phase 3 (Aug 2026) |
| DEP-05 | Fraud operations team available for UAT (minimum 4 analysts, 3 weeks) | Head of Fraud Ops | Phase 4 (Oct 2026) |
| DEP-06 | External penetration test booked and completed before go-live | IT Security / CISO | Phase 4 (Oct 2026) |

---

## 8. Requirements Traceability Matrix (RTM)

| Req. ID | Requirement (Short) | Business Obj. | Priority | Category | Delivery Phase |
|---|---|---|---|---|---|
| BR-01 | Real-time monitoring of all transactions | BO-01 | Must Have | Detection | Phase 3 – Build |
| BR-02 | Fraud risk scoring (0–100) per transaction | BO-01 | Must Have | Detection | Phase 3 – Build |
| BR-03 | Threshold-based automatic transaction blocking | BO-02 | Must Have | Detection | Phase 3 – Build |
| BR-05 | Configurable fraud rules engine | BO-04 | Must Have | Detection | Phase 3 – Build |
| BR-06 | Customer behavioural analytics | BO-03 | Should Have | Detection | Phase 3 – Build |
| BR-11 | Real-time case management dashboard | BO-04 | Must Have | Case Mgmt | Phase 3 – Build |
| BR-14 | Immutable analyst decision audit log | BO-05 | Must Have | Compliance | Phase 3 – Build |
| BR-15 | Automated customer notification on block | BO-06 | Should Have | Alerting | Phase 3 – Build |
| BR-19 | Real-time high-risk transaction alerting | BO-01 | Must Have | Alerting | Phase 3 – Build |
| BR-21 | Core banking system API integration | BO-01 | Must Have | Integration | Phase 3 – Build |
| BR-26 | Fraud KPI management dashboard | BO-02 | Must Have | Reporting | Phase 3 – Build |
| BR-30 | Regulatory report generation (FCA/PSD2) | BO-05 | Must Have | Compliance | Phase 3 – Build |
| BR-31 | 7-year immutable audit trail retention | BO-05 | Must Have | Compliance | Phase 3 – Build |
| BR-33 | AES-256 and TLS 1.3 data encryption | BO-05 | Must Have | Security | Phase 3 – Build |
| BR-36 | PSD2 SCA challenge integration | BO-05 | Must Have | Compliance | Phase 3 – Build |

---

## 9. Sign-Off & Approval

By signing below, the named stakeholders confirm that they have reviewed this BRD and that the requirements accurately represent the business needs for the RTPFPS. Signed approval triggers the baseline and activates change control for all requirements.

| Name | Title | Signature | Date |
|---|---|---|---|
| [Name] | Chief Risk Officer (Sponsor) | | |
| [Name] | Head of Fraud Operations | | |
| [Name] | Compliance Officer | | |
| [Name] | IT / Engineering Lead | | |
| [Name] | Business Analyst (Author) | | March 06, 2026 |

---

## 10. Document Control & Glossary

### 10.1 Version History

| Version | Date | Changes | Author |
|---|---|---|---|
| v1.0 | 06 March 2026 | Initial BRD created — 38 requirements, 12 business rules, NFRs and RTM documented | BA Portfolio Author |

### 10.2 Glossary

| Term / Acronym | Definition |
|---|---|
| ATO | Account Takeover — fraudulent access to a customer's account by a third party |
| BRD | Business Requirements Document — captures what the business needs, independent of technical design |
| CNP | Card-Not-Present — transactions where the physical card is not used (e.g., online payments) |
| FRD | Functional Requirements Document — describes how the system will behave to meet the BRD requirements |
| GDPR | General Data Protection Regulation — EU/UK data protection legislation |
| ML | Machine Learning — AI technique used to identify patterns and anomalies in transaction data |
| MoSCoW | Prioritisation method: Must Have, Should Have, Could Have, Won't Have |
| NFR | Non-Functional Requirement — quality attribute (performance, security, availability, etc.) |
| PCI-DSS | Payment Card Industry Data Security Standard — security standard for cardholder data |
| PSD2 | Payment Services Directive 2 — EU regulation governing payment services and fraud prevention |
| RBAC | Role-Based Access Control — access permissions assigned based on user role |
| RTM | Requirements Traceability Matrix — maps requirements to objectives, tests, and delivery phases |
| SAR | Suspicious Activity Report — mandatory regulatory report filed for suspected financial crime |
| SCA | Strong Customer Authentication — PSD2 requirement for two-factor authentication on payments |
| TPS | Transactions Per Second — system throughput metric |
| UAT | User Acceptance Testing — testing performed by business users to validate the system meets requirements |

---

*End of Document — RTPFPS BRD v1.0*
