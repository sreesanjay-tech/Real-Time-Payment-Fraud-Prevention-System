# Real-Time Payment Fraud Prevention System (RTPFPS)
## Process Flows & Use Cases
### Step-by-Step Process Flows, Use Case Specifications & Actor Interaction Narratives

---

## Table of Contents

1. [Introduction & Document Purpose](#1-introduction--document-purpose)
2. [Use Case Catalogue](#2-use-case-catalogue)
3. [Use Case Specifications](#3-use-case-specifications)
4. [Detailed Process Flows](#4-detailed-process-flows)
5. [Swim Lane Narratives](#5-swim-lane-narratives)
6. [Approval & Sign-Off](#6-approval--sign-off)
7. [Document Control](#7-document-control)

---

## 1. Introduction & Document Purpose

This document describes the end-to-end process flows and use cases for the RTPFPS. It bridges the gap between the requirements defined in the BRD and FRD and the visual process models that guide system design, UI design, and test case creation.

- **Use cases** describe complete actor-system interactions for specific business scenarios
- **Process flows** describe step-by-step sequences of actions, decisions, and system responses
- **Swim lane narratives** define cross-actor handoffs for conversion into BPMN or Visio diagrams

These artefacts are designed to be directly usable as the basis for **BPMN diagrams**, **Visio swim lane diagrams**, and **Lucidchart flows**.

---

### Actor Catalogue

| Actor ID | Actor Name | Description | System Role |
|---|---|---|---|
| ACT-01 | Customer | End customer who initiates payment transactions via mobile, web, POS, or other channels | External — Initiates transactions |
| ACT-02 | Fraud Analyst | Fraud operations team member who reviews and decisions flagged fraud cases in the CMS dashboard | Internal User — Case Review |
| ACT-03 | Senior Analyst | Senior fraud operations member who handles escalated cases and supervisor-level case reassignment | Internal User — Escalations |
| ACT-04 | Fraud Ops Manager | Head of fraud operations — approves fraud rules, monitors KPI dashboard, manages team workload | Internal Admin — Configuration & MI |
| ACT-05 | Compliance Officer | Reviews audit logs, regulatory reports, and SAR drafts; validates compliance of system outputs | Internal User — Compliance |
| ACT-06 | RTPFPS (System) | The fraud prevention platform — automated actor performing scoring, decisions, and case creation | System — Automated Processing |
| ACT-07 | Core Banking System | External system providing account data and receiving hold/release instructions | External System — Integration |
| ACT-08 | Card Platform | External card management system receiving block, unblock, and SCA challenge triggers | External System — Integration |
| ACT-09 | Notification Service | Delivers SMS and push notifications to customers about blocked transactions | External System — Integration |
| ACT-10 | ML Model Service | Internal microservice providing ML-based fraud anomaly scores for each transaction | Internal System — Scoring |

---

## 2. Use Case Catalogue

| UC ID | Use Case Name | Primary Actor | Priority | BRD Ref | Category |
|---|---|---|---|---|---|
| UC-01 | Customer Initiates Payment Transaction | Customer / Payment Channel | 🔴 High | BR-01, BR-04 | Transaction |
| UC-02 | System Scores Transaction for Fraud Risk | RTPFPS (System) | 🔴 High | BR-02, BR-05, BR-06 | Scoring |
| UC-03 | System Blocks High-Risk Transaction | RTPFPS (System) | 🔴 High | BR-03, BRU-01 | Decision |
| UC-04 | System Triggers SCA Challenge | RTPFPS / Card Platform | 🔴 High | BRU-02, BR-09 | Decision |
| UC-05 | Fraud Analyst Reviews and Decisions a Case | Fraud Analyst | 🔴 High | BR-11, BR-12, BR-13 | Case Mgmt |
| UC-06 | Fraud Analyst Escalates Case to Senior Analyst | Fraud Analyst | 🟡 Medium | BR-17 | Case Mgmt |
| UC-07 | Fraud Ops Manager Configures a Fraud Rule | Fraud Ops Manager | 🟡 Medium | BR-05, BRU-12 | Configuration |
| UC-08 | Compliance Officer Reviews Audit Log & SAR | Compliance Officer | 🟡 Medium | BR-30, BR-31 | Compliance |

---

## 3. Use Case Specifications

---

### UC-01 — Customer Initiates Payment Transaction
> **BRD:** BR-01, BR-04 | **FRD:** FR-01, FR-02

| Field | Detail |
|---|---|
| **Primary Actor** | Customer (ACT-01) |
| **Priority** | 🔴 High |
| **Secondary Actors** | Payment Channel (Mobile/Web/POS/ATM), RTPFPS System (ACT-06), Core Banking System (ACT-07) |
| **Pre-conditions** | Customer has an active account. Payment channel is operational. RTPFPS is active and monitoring all channels. |
| **Trigger** | Customer submits a payment request on any supported channel (tap to pay, online checkout, mobile banking transfer, ATM withdrawal). |

**Main Success Flow**

1. Customer initiates a payment transaction on the payment channel
2. Payment channel sends transaction event to RTPFPS API Gateway (`POST /v1/transaction/event`)
3. RTPFPS validates the transaction payload for mandatory fields
4. RTPFPS assigns a system transaction ID (UUID) and normalises the currency to GBP equivalent
5. RTPFPS enriches the event with device, channel, and geolocation metadata
6. RTPFPS publishes the enriched event to the fraud scoring queue within 50ms
7. Scoring begins — see UC-02
8. Transaction outcome (Pass / Challenge / Block) is returned within 300ms
9. If outcome = `PASS`: transaction is approved and customer receives confirmation

**Alternative Flows**

- **4a.** If mandatory payload fields are missing: RTPFPS returns `HTTP 400` to payment channel. Payment channel displays error to customer. Transaction does not proceed.
- **8a.** If outcome = `CHALLENGE`: customer is presented with SCA authentication — see UC-04
- **8b.** If outcome = `BLOCK`: transaction is declined — see UC-03

**Post-conditions**

Transaction event is ingested, scored, and a decision is made within 300ms. Customer receives transaction approval, SCA challenge, or decline message. All events are logged to the audit trail.

**Exceptions**

If RTPFPS scoring engine is unavailable: apply failsafe rule — all transactions above £500 are held; others are approved. IT-On-Call is alerted. Core Banking system receives hold instruction for high-value transactions.

---

### UC-02 — System Scores Transaction for Fraud Risk
> **BRD:** BR-02, BR-05, BR-06, BR-07 | **FRD:** FR-03, FR-04, FR-05

| Field | Detail |
|---|---|
| **Primary Actor** | RTPFPS System (ACT-06) |
| **Priority** | 🔴 High |
| **Secondary Actors** | Rules Engine (internal), ML Model Service (ACT-10), Customer Profile Store, Transaction History DB |
| **Pre-conditions** | A normalised transaction event has been published to the fraud scoring queue. Active fraud rules exist in the Rules DB. ML model is deployed and available. |
| **Trigger** | RTPFPS receives a normalised transaction event on `Kafka: txn-score-queue`. |

**Main Success Flow**

1. RTPFPS retrieves the active rules library from Rules DB (cached, refreshed every 60 seconds)
2. RTPFPS retrieves the customer's 90-day behavioural profile from Customer Profile Store
3. RTPFPS evaluates the transaction against all active fraud rules in priority order
4. For each matching rule: rule weight is added to the running rules score
5. Velocity checks applied: transactions in 1-min, 5-min, and 60-min windows vs. customer baseline
6. Geographic anomaly check: if transaction country differs from registered country → `score += 15`
7. New device check: if `device_id` not in customer's registered device list → `score += 20`
8. Rules score capped at 100 and passed to the ML scoring module
9. RTPFPS constructs the ML feature vector (32 features) from transaction data and customer profile
10. RTPFPS calls ML Model Service via gRPC (timeout: 150ms) and receives `ml_score` (0–100) + top-3 contributing features
11. Final score calculated: `final_score = (rules_score × 0.4) + (ml_score × 0.6)`
12. Final score passed to the Decision Engine

**Alternative Flows**

- **9a.** If customer profile is not found: use transaction-only scoring (no behavioural component). Apply +10 score uplift for unrecognised customer. Log as `PROFILE_NOT_FOUND`.
- **10a.** If ML Model Service times out (>150ms): use rules-only score with `ml_unavailable` flag set. Alert Data Science team if ML is unavailable for > 5 minutes.

**Post-conditions**

Final fraud risk score (0–100) calculated and passed to Decision Engine within 300ms of transaction receipt. Score breakdown logged: `rules_score`, `ml_score`, `final_score`, `rules_fired`, top ML features. All scoring events logged to audit trail.

**Exceptions**

If Rules DB is unavailable: use last cached rules snapshot (max 5 min old). If both rules and ML are unavailable: route transaction to `MANUAL_HOLD` queue and alert Fraud Ops Manager.

---

### UC-03 — System Blocks High-Risk Transaction
> **BRD:** BR-03, BRU-01 | **FRD:** FR-05, FR-06, FR-10

| Field | Detail |
|---|---|
| **Primary Actor** | RTPFPS System (ACT-06) |
| **Priority** | 🔴 High |
| **Secondary Actors** | Core Banking System (ACT-07), Case Management System (internal), Notification Service (ACT-09), Fraud Analyst (ACT-02) |
| **Pre-conditions** | Final fraud risk score has been calculated (≥ 85). Customer is not on the whitelist. Core Banking API is available. |
| **Trigger** | Decision Engine receives `final_score >= 85` for a transaction. |

**Main Success Flow**

1. Decision Engine sets `outcome = BLOCK`
2. RTPFPS calls Core Banking API: `POST /accounts/{account_id}/holds` to place a transaction hold. Core Banking confirms hold within 5 seconds.
3. RTPFPS publishes `BLOCK` decision event to `Kafka topic: txn-decisions`
4. Case Management System receives the `BLOCK` event and creates a new fraud case record (`case_id` assigned)
5. Case is assigned to the fraud analyst with the lowest current open case count
6. Case SLA timer is started (30-minute analyst review clock)
7. Notification Service triggered: SMS and push notification sent to customer within 60 seconds
8. Analyst receives real-time dashboard notification of new case
9. Transaction decline signal returned to the originating payment channel within 300ms
10. All events logged to the audit trail

**Alternative Flows**

- **2a.** If Core Banking hold API fails after 3 retries: route to `MANUAL_REVIEW` queue. Alert IT-On-Call.
- **5a.** If no analysts are currently available: assign case to `UNASSIGNED` queue. Alert Fraud Ops Manager.
- **7a.** If customer notification fails: log failure; queue for retry. Case closure is not blocked by notification failure.

**Post-conditions**

Transaction hold placed on Core Banking. Fraud case created and assigned to analyst within 30 seconds. Customer notified within 60 seconds. Decline signal returned to payment channel within 300ms. All actions logged.

**Exceptions**

If all downstream systems fail simultaneously: RTPFPS falls back to a safe-decline state — transaction is declined at the channel level even if hold cannot be confirmed. All failures logged and IT-On-Call immediately alerted.

---

### UC-04 — System Triggers SCA Challenge
> **BRD:** BRU-02, BR-09, BR-36 | **FRD:** FR-05

| Field | Detail |
|---|---|
| **Primary Actor** | RTPFPS System (ACT-06) |
| **Priority** | 🔴 High |
| **Secondary Actors** | Card Platform / SCA Service (ACT-08), Customer (ACT-01), Core Banking System (ACT-07) |
| **Pre-conditions** | Final fraud risk score is between 60 and 84. SCA Service is available. Customer is registered for SCA. |
| **Trigger** | Decision Engine receives `final_score` in range 60–84 for a payment transaction. |

**Main Success Flow**

1. Decision Engine sets `outcome = CHALLENGE`
2. RTPFPS calls SCA Service API to trigger a Strong Customer Authentication challenge
3. SCA Service sends a push notification or OTP SMS to the customer's registered device
4. Customer receives the SCA challenge and completes authentication within 60-second timeout window
5. SCA Service returns authentication result to RTPFPS: `AUTHENTICATED` or `FAILED`
6. If `AUTHENTICATED`: RTPFPS sets `outcome = PASS`. Transaction approved. Event logged.
7. If `FAILED` or timeout: RTPFPS sets `outcome = BLOCK`. Flow continues as per UC-03.

**Alternative Flows**

- **4a.** If customer does not respond within 60 seconds: timeout. Outcome = `BLOCK`. Proceed as UC-03.
- **5a.** If SCA Service is unavailable: default outcome = `BLOCK`. Set `sca_unavailable` flag. Log and alert IT-On-Call.

**Post-conditions**

SCA challenge delivered to customer within 5 seconds of score decision. Customer either authenticates successfully (transaction proceeds) or fails/times out (transaction blocked). All SCA events logged to audit trail.

**Exceptions**

> ⚠️ SCA Service unavailability must **never** result in a `CHALLENGE` transaction being approved without authentication. All SCA service failures trigger automatic `BLOCK` and IT-On-Call alert.

---

### UC-05 — Fraud Analyst Reviews and Decisions a Case
> **BRD:** BR-11, BR-12, BR-13, BR-14, BR-18 | **FRD:** FR-06, FR-07, FR-08

| Field | Detail |
|---|---|
| **Primary Actor** | Fraud Analyst (ACT-02) |
| **Priority** | 🔴 High |
| **Secondary Actors** | RTPFPS Case Management System (ACT-06), Core Banking System (ACT-07), Customer Data API, Notification Service (ACT-09) |
| **Pre-conditions** | A fraud case has been created and assigned to the analyst. Analyst is logged in with ROLE-01 permissions. The transaction hold is in place on Core Banking. |
| **Trigger** | Fraud analyst receives a real-time dashboard notification of a new assigned case, or selects an open case from their queue. |

**Main Success Flow**

1. Analyst opens the Case Investigation View from their dashboard
2. System loads full case context within 3 seconds: transaction details, customer profile, account history (last 20 transactions), device and location data, fraud score breakdown, rules that fired, ML feature contributions
3. Analyst reviews all available evidence and makes a decision
4. Analyst selects one of three decision actions: **APPROVE**, **DECLINE**, or **ESCALATE**

| Decision | System Action |
|---|---|
| ✅ **APPROVE** | Core Banking hold released. Case status = `CLOSED`. Tag = `FALSE_POSITIVE` or `SUSPICIOUS_APPROVED`. Customer notified. |
| ❌ **DECLINE** | Block confirmed on Core Banking. Customer notified. Case status = `CLOSED`. Tag = `CONFIRMED_FRAUD`. SAR auto-generated if amount ≥ £10,000. |
| ↗️ **ESCALATE** | Case transferred to next available Senior Analyst. Full case history retained. Status = `ESCALATED`. |

5. Analyst optionally adds free-text case notes (max 1,000 characters) before closing
6. System writes an **immutable audit log entry**: `case_id`, `analyst_id`, `decision`, `outcome_tag`, `notes`, `timestamp`
7. If `outcome = FALSE_POSITIVE`: feedback event published to ML retraining pipeline within 24 hours

**Alternative Flows**

- **3a.** If analyst SLA (30 minutes) is reached without a decision: system auto-escalates case to Senior Analyst. Fraud Ops Manager is notified.
- **5a.** If Core Banking release API fails on `APPROVE`: system displays error. Retry option available. Case remains `OPEN` until release is confirmed.

**Post-conditions**

Case reviewed and decisioned within SLA. Transaction hold released or confirmed. Customer notified. Immutable audit log entry created. SAR created where applicable. ML feedback published for false positives.

**Exceptions**

> ⚠️ Audit log write failure is a **critical system error** — case closure is blocked until the log entry is successfully written. This ensures full traceability and compliance at all times.

---

### UC-06 — Fraud Analyst Escalates Case to Senior Analyst
> **BRD:** BR-17 | **FRD:** FR-08

| Field | Detail |
|---|---|
| **Primary Actor** | Fraud Analyst (ACT-02) |
| **Priority** | 🟡 Medium |
| **Secondary Actors** | Senior Analyst (ACT-03), RTPFPS Case Management System (ACT-06) |
| **Pre-conditions** | Fraud Analyst has an open assigned case. At least one Senior Analyst is available. The original analyst has reviewed the case and determined it requires senior review. |
| **Trigger** | Fraud Analyst selects 'Escalate' on a case they cannot confidently decision, or where the transaction is complex or high-value. |

**Main Success Flow**

1. Fraud Analyst selects `ESCALATE` action on the case
2. System prompts analyst to select an escalation reason from a predefined list: *Complex fraud pattern / High value transaction / Conflicting evidence / Customer vulnerability concern*
3. Analyst optionally adds escalation notes
4. System identifies the next available Senior Analyst based on current open case count
5. Case ownership transferred to Senior Analyst. Full case history, analyst notes, and original fraud score are retained and visible.
6. Case status updated to `ESCALATED`. SLA timer reset to new 60-minute senior review window.
7. Senior Analyst receives a real-time dashboard notification of the escalated case
8. Original Fraud Analyst receives confirmation that case has been transferred
9. Escalation event logged to audit trail: `original_analyst_id`, `escalation_reason`, `new_analyst_id`, `timestamp`

**Alternative Flows**

- **4a.** If no Senior Analyst is available: case placed in `ESCALATED-UNASSIGNED` queue. Fraud Ops Manager immediately notified to manually assign.

**Post-conditions**

Case transferred to Senior Analyst within 30 seconds. Full case history retained. New SLA timer started. All parties notified. Escalation logged to audit trail.

**Exceptions**

> ⚠️ An escalated case must never be left unassigned for more than **15 minutes** without a Fraud Ops Manager alert being triggered.

---

### UC-07 — Fraud Ops Manager Configures a Fraud Detection Rule
> **BRD:** BR-05, BRU-12 | **FRD:** FR-03

| Field | Detail |
|---|---|
| **Primary Actor** | Fraud Ops Manager (ACT-04) |
| **Priority** | 🟡 Medium |
| **Secondary Actors** | RTPFPS Rules Management UI (ACT-06), Rules Engine DB |
| **Pre-conditions** | Fraud Ops Manager is authenticated with ROLE-03 permissions. Manager has identified a new fraud pattern requiring a detection rule. |
| **Trigger** | Fraud Ops Manager navigates to the Rules Management section and selects 'Create New Rule' or 'Edit Existing Rule'. |

**Main Success Flow**

1. Fraud Ops Manager opens the Rule Builder UI
2. Manager defines the rule:
   - Rule name and description
   - Trigger condition(s) *(e.g., `transaction_amount > 5000 AND merchant_category = 'crypto exchange'`)*
   - Rule weight (score increment: 1–50 points)
   - Rule status (Active / Inactive)
   - Applicable channels
3. Manager clicks **'Test Rule'** — system runs rule against the last 24 hours of historical transactions and returns: number of matches, estimated false positive impact, sample matched transactions
4. Manager reviews test results. If acceptable: proceeds to activation.
5. Manager submits rule for activation. System validates: ROLE-03 permission confirmed; rule has been tested.
6. System activates the rule in the Rules Engine DB. Active rules cache refreshed within 60 seconds.
7. New rule immediately applied to all incoming transactions
8. Rule creation event logged to audit trail: `manager_id`, `rule_id`, `rule_definition`, `activation_timestamp`

**Alternative Flows**

- **3a.** If test results show > 5% estimated false positive rate: system displays a high false positive warning. Manager must confirm before activation.
- **5a.** If activation fails due to rule syntax error: system returns detailed error message identifying the invalid condition. Manager corrects and retests.

**Post-conditions**

New fraud detection rule active in the Rules Engine within 60 seconds. Test results reviewed. Audit log records rule creation with full definition. No IT ticket or code change was required.

**Exceptions**

> ⚠️ A rule with a syntax error must never be activated silently. All rule changes — including deactivation — must be logged.

---

### UC-08 — Compliance Officer Reviews Audit Log and SAR
> **BRD:** BR-30, BR-31, BRU-09 | **FRD:** FR-12, FR-13

| Field | Detail |
|---|---|
| **Primary Actor** | Compliance Officer (ACT-05) |
| **Priority** | 🟡 Medium |
| **Secondary Actors** | RTPFPS Compliance Portal (ACT-06), SAR Management Module, Audit Log DB |
| **Pre-conditions** | Compliance Officer is authenticated with ROLE-04 permissions. A SAR draft has been auto-generated for a qualifying confirmed fraud case (amount ≥ £10,000). |
| **Trigger** | Compliance Officer receives an email and dashboard notification of a new SAR draft, OR proactively queries the audit log for a specified date range. |

**Main Success Flow**

**SAR Review:**
1. Officer navigates to SAR Review Queue in the Compliance Portal
2. Opens pending SAR draft — reviews pre-populated fields: reporter details, subject details, suspicious activity narrative, transaction details, case timeline
3. Officer edits narrative if required and adds additional context
4. Officer submits SAR for filing or rejects and flags for analyst follow-up
5. System logs SAR submission: `sar_id`, `officer_id`, `submission_timestamp`, `decision`

**Audit Log Review:**
1. Officer navigates to Audit Log Query screen
2. Enters query parameters: date range, event type, `actor_id`, `case_id`
3. System returns paginated audit log entries
4. Officer can export results as CSV or PDF

**Regulatory Report Review:**
1. Officer navigates to Reports section
2. Selects report type (FCA Monthly Return / PSD2 Quarterly / PCI-DSS)
3. Reviews auto-generated report content; submits or downloads for manual submission

> All Compliance Officer actions are themselves logged to the audit trail.

**Alternative Flows**

- **SAR 2a.** If SAR SLA (24 hours) is reached without Officer review: escalation alert sent to Chief Compliance Officer.
- **Audit 3a.** If audit log query returns > 10,000 records: system prompts Officer to narrow the date range before running the full query.

**Post-conditions**

SAR reviewed and submitted (or flagged) within 24-hour SLA. Audit log queries return accurate, immutable records. Regulatory reports available for submission. All Compliance Officer actions logged.

**Exceptions**

> ⚠️ Compliance Officer must **not** be able to edit or delete audit log entries. Any attempt to modify the audit log must be rejected and a security alert sent to the CISO and IT-On-Call.

---

## 4. Detailed Process Flows

---

### Process Flow 1 — Transaction Ingestion to Fraud Decision *(Happy Path)*

| Step | Actor | Action | System Response | Decision / Next |
|---|---|---|---|---|
| 1 | Customer (ACT-01) | Initiates payment on any channel | Payment channel sends `POST /v1/transaction/event` to API Gateway | → Step 2 |
| 2 | API Gateway (ACT-06) | Receives and validates transaction payload | If valid: `202 Accepted`. Event normalised, UUID assigned, enriched with metadata. | If invalid: `HTTP 400`. End flow. |
| 3 | Deduplication Engine | Checks Redis cache for duplicate event (60s window) | If duplicate: event discarded, log `DUPLICATE`. If new: proceed. | → Step 4 |
| 4 | Rules Engine (ACT-06) | Evaluates transaction against all active fraud rules | Rules score (0–100) calculated. Matched rules list returned. | → Step 5 |
| 5 | ML Model Service (ACT-10) | Scores transaction using 32-feature ML model | ML score (0–100) returned with top-3 contributing features | → Step 6 |
| 6 | Score Blending (ACT-06) | Calculates final score: `(rules × 0.4) + (ML × 0.6)` | Final fraud risk score (integer, 0–100) produced | → Step 7 |
| 7 | Decision Engine (ACT-06) | Applies threshold rules to final score | Score < 60: `PASS` \| Score 60–84: `CHALLENGE` \| Score ≥ 85: `BLOCK` | Branch by outcome |
| 8a | System (`PASS`) | Returns approval signal to payment channel | Transaction approved. Audit log entry created. | ✅ End — Transaction complete |
| 8b | System (`CHALLENGE`) | Calls SCA Service to trigger authentication challenge | SCA challenge sent to customer. Awaiting response (60s timeout). | → See Process Flow 2 |
| 8c | System (`BLOCK`) | Calls Core Banking API to place transaction hold | Hold placed. Decline signal returned to channel. Case created in CMS. | → See Process Flow 3 |

---

### Process Flow 2 — SCA Challenge Flow

| Step | Actor | Action | System Response | Decision / Next |
|---|---|---|---|---|
| 1 | Decision Engine (ACT-06) | Sets `outcome = CHALLENGE` for score 60–84 | SCA trigger event published | → Step 2 |
| 2 | SCA Service (ACT-08) | Receives SCA trigger. Identifies customer's SCA method (push / OTP SMS). | SCA challenge sent to customer's registered device. 60-second timer starts. | → Step 3 |
| 3 | Customer (ACT-01) | Receives SCA challenge notification on mobile device | Customer sees: *'Confirm payment of [amount] to [merchant]?'* | Customer responds or times out |
| 4a | Customer — Authenticates | Customer approves SCA challenge via biometric or PIN | SCA Service receives `AUTHENTICATED` response | → Step 5a |
| 4b | Customer — Rejects / Timeout | Customer declines or does not respond within 60 seconds | SCA Service receives `FAILED` or `TIMEOUT` response | → Step 5b |
| 5a | SCA Service | Returns `AUTHENTICATED` to Decision Engine | Decision Engine sets `outcome = PASS`. Transaction approved. | ✅ End — Transaction approved |
| 5b | SCA Service | Returns `FAILED`/`TIMEOUT` to Decision Engine | Decision Engine sets `outcome = BLOCK`. Flow continues as Process Flow 3. | → Process Flow 3 |
| 6 | RTPFPS (ACT-06) | Logs SCA event to audit trail | Audit log entry: `challenge_timestamp`, `response_timestamp`, `outcome` | End of SCA flow |

---

### Process Flow 3 — Transaction Block & Case Management Flow

| Step | Actor | Action | System Response | Decision / Next |
|---|---|---|---|---|
| 1 | Decision Engine (ACT-06) | Sets `outcome = BLOCK` (score ≥ 85 or SCA failure) | `BLOCK` event published to `Kafka: txn-decisions` | → Step 2 |
| 2 | Core Banking API (ACT-07) | Receives hold instruction from RTPFPS | Transaction hold placed. Confirmation returned within 5 seconds. | → Step 3 |
| 3 | Case Mgmt System (ACT-06) | Creates fraud case record. Assigns unique `case_id`. | Case record created with `status = OPEN`. SLA timer (30 mins) started. | → Step 4 |
| 4 | Assignment Engine (ACT-06) | Identifies analyst with lowest open case count | Case assigned to selected Fraud Analyst. Dashboard notification sent. | → Step 5 |
| 5 | Notification Service (ACT-09) | Receives customer notification trigger | SMS and push notification sent to customer within 60 seconds of block. | → Step 6 |
| 6 | Fraud Analyst (ACT-02) | Opens Case Investigation View. Reviews all transaction context. | Case view rendered within 3 seconds. Full context displayed. | Analyst makes decision |
| 7a | Analyst — Approve | Analyst selects `APPROVE`. Tags outcome as `FALSE_POSITIVE`. | Core Banking hold released. Customer notified. ML feedback published. | ✅ Case closed — PASS |
| 7b | Analyst — Decline | Analyst selects `DECLINE`. Tags outcome as `CONFIRMED_FRAUD`. | Block confirmed. Customer notified. SAR triggered if amount ≥ £10K. | ✅ Case closed — FRAUD |
| 7c | Analyst — Escalate | Analyst selects `ESCALATE`. Adds escalation reason. | Case transferred to Senior Analyst. New 60-min SLA timer started. | → See UC-06 |
| 8 | Audit Trail (ACT-06) | Logs all case events, analyst decisions, and outcomes | Immutable audit log entry created for every event in the flow. | End of flow |

---

### Process Flow 4 — Fraud Rule Configuration by Fraud Ops Manager

| Step | Actor | Action | System Response | Decision / Next |
|---|---|---|---|---|
| 1 | Fraud Ops Manager (ACT-04) | Logs in to RTPFPS dashboard. Navigates to Rules Management. | Rules Management screen loaded. Existing rules displayed. | → Step 2 |
| 2 | Manager | Selects 'Create New Rule'. Defines rule conditions, weight, and channels. | Rule Builder UI validates condition syntax in real time. | → Step 3 |
| 3 | Manager | Clicks 'Test Rule Against Last 24 Hours' | System returns: # of matches, estimated FP rate, sample matched transactions. | → Step 4 |
| 4 | Manager | Reviews test results. Confirms FP rate is acceptable (< 5%). | If FP rate > 5%: warning displayed. Manager confirms to proceed or adjusts rule. | → Step 5 |
| 5 | Manager | Submits rule for activation | System validates ROLE-03 permission. Checks rule has been tested. Activates rule in Rules DB. | → Step 6 |
| 6 | Rules Engine (ACT-06) | Active rules cache refreshed within 60 seconds | New rule now applied to all incoming transactions in real time. | → Step 7 |
| 7 | Audit Trail | Logs rule creation event | Audit entry: `manager_id`, `rule_id`, full rule definition, `activation_timestamp`. | End of flow |

---

### Process Flow 5 — Compliance Officer SAR Review Flow

| Step | Actor | Action | System Response | Decision / Next |
|---|---|---|---|---|
| 1 | System (ACT-06) | Confirmed fraud case closed with `amount >= £10,000` | SAR draft auto-generated and placed in Compliance Officer review queue. | → Step 2 |
| 2 | Compliance Officer (ACT-05) | Receives email + dashboard notification of pending SAR | SAR draft accessible in Compliance Portal. | → Step 3 |
| 3 | Compliance Officer | Opens SAR draft. Reviews pre-populated fields. | SAR draft displays: reporter details, subject details, suspicious activity narrative, transaction data, case timeline. | → Step 4 |
| 4 | Compliance Officer | Edits narrative if needed. Adds additional context. | SAR editor allows free-text editing of narrative section only. | → Step 5 |
| 5a | Compliance Officer — Submit | Officer is satisfied. Submits SAR. | SAR marked as `SUBMITTED`. Record archived with 7-year retention. Audit log entry created. | ✅ End — SAR filed |
| 5b | Compliance Officer — Reject | Officer identifies issue. Rejects SAR, adds rejection notes. | SAR flagged for analyst follow-up. Analyst notified. Audit log entry created. | Case re-opened for review |
| 6 | Audit Trail | All Compliance Officer actions logged | Immutable log entry: `officer_id`, `sar_id`, `action`, `timestamp`. | End of flow |

---

### Process Flow 6 — Automatic SLA Breach Escalation Flow

| Step | Actor | Action | System Response | Decision / Next |
|---|---|---|---|---|
| 1 | SLA Monitor (ACT-06) | Monitors all open cases against their 30-minute SLA deadline | System checks all `OPEN` cases every 60 seconds against SLA deadline. | Continuous monitoring |
| 2 | SLA Monitor | Detects case approaching SLA breach (25-minute mark) | System sends an early warning notification to the assigned analyst. | → Step 3 |
| 3 | Assigned Analyst (ACT-02) | Receives early warning. Has 5 minutes remaining to decision the case. | Analyst reviews case urgently. | Analyst decisions OR time expires |
| 4a | Analyst decisions in time | Analyst makes a decision within 30-minute SLA | Case closed or escalated normally. No breach recorded. | ✅ End — Normal flow |
| 4b | 30-minute SLA expires | No decision made by assigned analyst | System automatically escalates to Senior Analyst. Fraud Ops Manager notified of SLA breach. | → Step 5 |
| 5 | Senior Analyst (ACT-03) | Receives escalated case with SLA breach flag visible | New 60-minute SLA timer started for Senior Analyst. | → Step 6 |
| 6 | Fraud Ops Manager (ACT-04) | Receives SLA breach notification. Reviews analyst performance. | Manager can reassign or add additional capacity. | → Step 7 |
| 7 | Audit Trail | SLA breach event logged with full context | Audit entry: `case_id`, `original_analyst_id`, `breach_timestamp`, `escalation_action`. | End of flow |

---

## 5. Swim Lane Narratives

The following narratives define cross-actor handoffs for the three highest-priority scenarios. Each is designed for direct conversion into swim lane diagrams in **Visio**, **Lucidchart**, or **draw.io**.

---

### 5.1 Swim Lane: Transaction Scoring and Decision

| Lane (Actor) | Actions & Handoffs |
|---|---|
| **Customer** | Initiates payment transaction on chosen channel. Receives final transaction outcome (approved / SCA challenge / declined). Responds to SCA challenge if presented. |
| **Payment Channel** | Sends transaction event payload to RTPFPS API Gateway. Receives and displays final outcome to customer. |
| **RTPFPS — Ingestion Layer** | Validates and normalises event. Assigns transaction ID. Enriches with metadata. Publishes to scoring queue. Runs deduplication check. |
| **RTPFPS — Scoring Engine** | Retrieves active rules. Retrieves customer profile. Evaluates rules scoring. Calls ML Model Service. Blends scores. Passes final score to Decision Engine. |
| **ML Model Service** | Receives feature vector from Scoring Engine. Returns ML fraud score and top contributing features within 150ms. |
| **RTPFPS — Decision Engine** | Applies threshold decision logic. Routes to `PASS`, `CHALLENGE`, or `BLOCK`. Executes appropriate action on downstream systems. Publishes decision event to Kafka. |
| **Core Banking System** | Receives transaction hold instruction (`BLOCK`) or approval signal (`PASS`). Executes hold or confirms approval. Returns response to RTPFPS. |
| **SCA Service / Card Platform** | Receives SCA challenge trigger (`CHALLENGE`). Sends authentication request to customer. Returns authentication result to Decision Engine. |
| **Notification Service** | Receives customer notification trigger (`BLOCK`). Sends SMS and push notification to customer within 60 seconds. |
| **Audit Trail** | Receives event log entries from every actor at every step. Stores immutably with timestamp and actor ID. |

---

### 5.2 Swim Lane: Fraud Analyst Case Review

| Lane (Actor) | Actions & Handoffs |
|---|---|
| **RTPFPS — Case Mgmt System** | Creates fraud case on `BLOCK` event. Assigns to analyst with lowest queue. Sets 30-min SLA timer. Sends dashboard notification. Monitors SLA clock continuously. |
| **Fraud Analyst** | Receives dashboard notification. Opens Case Investigation View. Reviews all transaction context. Makes decision (Approve / Decline / Escalate). Adds case notes. Submits decision. |
| **RTPFPS — Case Mgmt System** | Receives analyst decision. Routes action to appropriate downstream system. Updates case status. Writes audit log entry. Triggers ML feedback if false positive. |
| **Core Banking System** | Receives release instruction (`APPROVE`) or hold confirmation (`DECLINE`). Executes and confirms. |
| **Notification Service** | Receives customer notification trigger. Sends customer outcome notification (approved or permanently declined). |
| **ML Feedback Pipeline** | Receives false positive feedback event. Queues for model retraining within 24 hours. |
| **Compliance — SAR Module** | Receives SAR trigger for `CONFIRMED_FRAUD` cases ≥ £10,000. Creates pre-populated SAR draft. Routes to Compliance Officer review queue. |
| **Senior Analyst** *(SLA breach path)* | If 30-min SLA expires without analyst decision: receives escalated case. Starts 60-min SLA timer. Decisions case. |
| **Fraud Ops Manager** *(SLA breach path)* | Receives SLA breach notification. Reviews and acts as needed. Can reassign cases manually. |
| **Audit Trail** | Records all case creation, assignment, analyst actions, decisions, SLA events, and escalation events with full immutable logs. |

---

### 5.3 Swim Lane: Regulatory Compliance Flow

| Lane (Actor) | Actions & Handoffs |
|---|---|
| **RTPFPS — Case Mgmt** | On `CONFIRMED_FRAUD` case closure with `amount >= £10,000`: triggers SAR auto-generation. Publishes SAR trigger event. |
| **RTPFPS — SAR Module** | Receives SAR trigger. Retrieves case data, customer data, analyst notes. Pre-populates NCA SAR template. Routes draft to Compliance Officer queue. Starts 24-hour SLA timer. |
| **Compliance Officer** | Receives SAR notification. Opens draft in Compliance Portal. Reviews and edits narrative. Submits or rejects SAR. Also runs ad-hoc audit log queries and reviews scheduled regulatory reports. |
| **RTPFPS — Report Scheduler** | Runs on configured schedule (daily / weekly / monthly). Queries Reporting DB. Applies FCA / PSD2 / PCI-DSS templates. Generates PDF and CSV outputs. Distributes to configured recipients. |
| **Audit Trail DB** | Stores all SAR events, compliance officer actions, report generation events, and regulatory submission records. All records retained for **minimum 7 years**. Append-only — no modifications or deletions permitted. |
| **Chief Compliance Officer** | Receives escalation if Compliance Officer does not review SAR within 24-hour SLA. Receives copies of all regulatory reports. |
| **CISO / IT-On-Call** | Receives security alert if any user — including Compliance Officer — attempts to modify or delete an audit log entry. |

---

## 6. Approval & Sign-Off

By signing below, the named stakeholders confirm that the process flows and use cases accurately represent the intended system behaviour and operational workflows for the RTPFPS.

| Name | Title | Signature | Date |
|---|---|---|---|
| [Name] | Head of Fraud Operations | | |
| [Name] | IT / Engineering Lead | | |
| [Name] | Compliance Officer | | |
| [Name] | QA / Test Lead | | |
| [Name] | Business Analyst (Author) | | March 06, 2026 |

---

## 7. Document Control

| Version | Date | Changes | Author |
|---|---|---|---|
| v1.0 | 06 March 2026 | Initial release — 8 use cases, 6 process flows, 3 swim lane narratives, 10-actor catalogue | BA Portfolio Author |

---

*End of Document — RTPFPS Process Flows & Use Cases v1.0*
