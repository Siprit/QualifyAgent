# Agentforce Lead Qualification Agent — Equity Builder Finance

## Context

Equity Builder Finance needs a conversational Agentforce Agent to qualify loan leads. The agent collects identity info, verifies via email OTP, and gathers comprehensive financial data (income, assets, documents) to produce a qualification decision. Everything must be **100% declarative** — Flows, Sub-flows, Custom Metadata Types, and OOTB Salesforce features only. No Apex.

**Key design decisions:**
- Financial data is stored as individual records on a child object `Financial_Asset__c` (Master-Detail to Lead)
- OTP verification is a **standalone reusable custom object** (`OTP_Verification__c`) — polymorphic, works with Lead, Case, Contact, or any object
- An **Experience Cloud portal** allows customers to self-serve: view application status, upload/re-upload documents, and update stale financial data

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      AGENTFORCE AGENT                           │
│              Equity_Builder_Qualification_Agent                  │
├──────────┬──────────────────┬───────────────────────────────────┤
│ Topic 1  │     Topic 2      │       Topic 3                     │
│ Identity │ OTP Verification │ Financial Qualification            │
│Collection│                  │                                    │
└────┬─────┴────────┬─────────┴──────────┬────────────────────────┘
     │              │                    │
     ▼              ▼                    ▼
┌─────────┐  ┌───────────┐  ┌──────────────────────────┐
│Create/   │  │Send OTP   │  │Create Financial Asset    │
│Update    │  │Verify OTP │  │Request Document Upload   │
│Lead Flow │  │(2 Flows)  │  │Run Qualification         │
└─────────┘  └─────┬─────┘  └──────────┬───────────────┘
                    │                   │
     ┌──────────────┼───────────────────┼──────────────────┐
     ▼              ▼                   ▼                  ▼
┌────────────┐ ┌──────────────────┐ ┌──────────────┐ ┌─────────────────┐
│Lead        │ │OTP_Verification  │ │Financial     │ │Experience Cloud │
│(identity + │ │__c               │ │_Asset__c     │ │Portal           │
│ qual       │ │(reusable across  │ │(child records│ │(self-service    │
│ summary)   │ │ Lead/Case/etc.)  │ │+ documents)  │ │ view + upload + │
│            │ │                  │ │              │ │ update)         │
└────────────┘ └──────────────────┘ └──────────────┘ └─────────────────┘
```

---

## 1. Agentforce Agent — 3 Topics

### Topic 1: Identity Collection
- **Instructions:** Greet user, collect First Name, Last Name, Phone, Email. Call Create_Or_Update_Lead action.
- **Action:** `Action_Create_Or_Update_Lead` → Flow `EBF_Create_Or_Update_Lead`

### Topic 2: OTP Verification
- **Instructions:** Send 6-digit OTP to email. Ask user to enter it. Up to 3 retries. On failure, direct to support.
- **Actions:**
  - `Action_Send_OTP` → Flow `OTP_Send` (generic, reusable)
  - `Action_Verify_OTP` → Flow `OTP_Verify` (generic, reusable)

### Topic 3: Financial Qualification
- **Instructions:** Collect data one question at a time: (1) cash on hand, (2) term deposits, (3) 2 payslips, (4) partner? → partner payslips, (5) dependents, (6) 2 tax returns, (7) super, (8) savings, (9) crypto, (10) investment properties. Each item creates a Financial_Asset__c record. Then run qualification.
- **Actions:**
  - `Action_Create_Financial_Asset` → Flow `EBF_Create_Financial_Asset`
  - `Action_Request_Document_Upload` → Flow `EBF_Request_Document_Upload`
  - `Action_Run_Qualification` → Flow `EBF_Run_Qualification`

---

## 2. Reusable OTP System (Custom Object + Custom Metadata)

### Why a Custom Object (not fields on Lead)

OTP fields on Lead only work for Lead verification. By creating `OTP_Verification__c`, the same OTP flows can verify:
- **Leads** — initial qualification (current use case)
- **Cases** — customer identity before sharing sensitive info
- **Contacts** — portal login verification, account changes
- **Any future object** — via the polymorphic `Related_Record_Id__c` field

### Custom Object: `OTP_Verification__c`

| Field | Type | Purpose |
|---|---|---|
| `Name` | Auto Number (OTP-{0000}) | Record identifier |
| `Related_Record_Id__c` | Text(18) | Polymorphic — stores any record Id (Lead, Case, Contact, etc.) |
| `Related_Object_Type__c` | Text(50) | "Lead", "Case", "Contact" — for filtering/reporting |
| `Recipient_Email__c` | Email | Where the OTP was sent |
| `Recipient_Phone__c` | Phone | For future SMS OTP |
| `OTP_Code__c` | Text(6) | The generated code (FLS: system-only) |
| `Expiry_DateTime__c` | DateTime | When the OTP expires |
| `Attempts__c` | Number(1,0) | Failed verification attempts |
| `Status__c` | Picklist | Pending / Verified / Expired / Failed |
| `Channel__c` | Picklist | Email / SMS (future) |
| `Config_Name__c` | Text(50) | Which OTP_Config__mdt record to use (default: "Default") |
| `Verified_DateTime__c` | DateTime | When verification succeeded |

### Custom Metadata: `OTP_Config__mdt` (renamed — no EBF prefix, generic)

Configurable settings per use case. Multiple records for different contexts.

| Field | Type |
|---|---|
| `OTP_Length__c` | Number(1,0) |
| `Expiry_Minutes__c` | Number(2,0) |
| `Max_Attempts__c` | Number(1,0) |
| `Email_Template_Name__c` | Text(100) |
| `Channel__c` | Text(20) — Email / SMS |
| `Is_Active__c` | Checkbox |

**Records:**
| DeveloperName | Expiry_Minutes | Max_Attempts | Email_Template | Use Case |
|---|---|---|---|---|
| `Default` | 10 | 3 | `OTP_Verification_Email` | General purpose |
| `Lead_Qualification` | 10 | 3 | `EBF_OTP_Verification` | Lead qualification |
| `Case_Verification` | 5 | 3 | `Case_OTP_Verification` | Case identity check |
| `Portal_Login` | 15 | 5 | `Portal_OTP_Verification` | Experience Cloud portal |

### Reusable OTP Flows (2 generic flows)

#### `OTP_Send` (Autolaunched Flow)

**Inputs:** `relatedRecordId` (Text), `relatedObjectType` (Text), `recipientEmail` (Email), `configName` (Text, default "Default")

1. **Get Records** — Fetch `OTP_Config__mdt` where DeveloperName = `configName`
2. **Assignment** — Generate OTP: `TEXT(FLOOR(RANDOM() * 900000) + 100000)`
3. **Assignment** — Set expiry: CurrentDateTime + config's Expiry_Minutes
4. **Create Records** — Create `OTP_Verification__c` record with all fields, Status = "Pending"
5. **Send Email** — To `recipientEmail` using the template from config, with OTP in body
6. **Output:** `otpRecordId` (the new OTP_Verification__c Id), `otpSent` (Boolean)

#### `OTP_Verify` (Autolaunched Flow)

**Inputs:** `otpRecordId` (Text), `enteredCode` (Text)

1. **Get Records** — Fetch `OTP_Verification__c` by Id
2. **Decision: Already verified?** — Status = "Verified" → return "ALREADY_VERIFIED"
3. **Decision: Expired?** — CurrentDateTime > Expiry_DateTime → update Status = "Expired" → return "EXPIRED"
4. **Get Records** — Fetch `OTP_Config__mdt` using `Config_Name__c`
5. **Decision: Too many attempts?** — Attempts >= config Max_Attempts → update Status = "Failed" → return "MAX_ATTEMPTS"
6. **Decision: Code match?**
   - **Match:** Update Status = "Verified", Verified_DateTime = now → return "SUCCESS"
   - **No match:** Increment Attempts → return "INVALID"
7. **Output:** `verificationResult` (Text)

### How Lead Qualification Uses OTP

The Agentforce `Action_Send_OTP` passes:
- `relatedRecordId` = Lead Id
- `relatedObjectType` = "Lead"
- `recipientEmail` = Lead's email
- `configName` = "Lead_Qualification"

The returned `otpRecordId` is passed to `Action_Verify_OTP`. The Lead no longer stores any OTP fields — they all live on `OTP_Verification__c`.

### Lead Fields Removed (moved to OTP_Verification__c)

`OTP_Code__c`, `OTP_Expiry__c`, `OTP_Attempts__c` are **removed from Lead**. Only `OTP_Verified__c` remains on Lead as a convenience flag (updated by a small helper flow or by the qualification flow after OTP_Verify returns SUCCESS).

---

## 3. Custom Object: `Financial_Asset__c`

**Relationship:** Master-Detail to Lead (Lead is the master)

### Fields on Financial_Asset__c

| Field | Type | Purpose |
|---|---|---|
| `Lead__c` | Master-Detail(Lead) | Parent Lead record |
| `Asset_Category__c` | Picklist | Payslip / Partner_Payslip / Tax_Return / Cash / Term_Deposit / Superannuation / Savings_Account / Cryptocurrency / Investment_Property / Gross_Income / Partner_Income |
| `Asset_Value__c` | Currency(14,2) | Dollar value |
| `Description__c` | Text(255) | Free-text (e.g. "March 2026 payslip") |
| `Period__c` | Text(50) | Time period (e.g. "2025-2026") |
| `Sequence__c` | Number(2,0) | Ordering (Payslip 1 vs 2) |
| `Document_Uploaded__c` | Checkbox | Whether document attached |
| `Document_Required__c` | Checkbox | Whether upload is mandatory (from CMT) |
| `Status__c` | Picklist | Pending / Document_Uploaded / Verified / Rejected / Update_Requested |
| `Property_Address__c` | Text(255) | Investment properties only |
| `Property_Count__c` | Number(2,0) | Investment properties only |
| `Is_Stale__c` | Checkbox | Flagged by verifier as not recent enough |
| `Stale_Reason__c` | Text(255) | Why it was flagged (shown to customer in portal) |
| `Updated_By_Customer__c` | Checkbox | Whether customer updated via portal |
| `Last_Customer_Update__c` | DateTime | When customer last modified via portal |

### Document Linking

```
Lead
 └── Financial_Asset__c ("Payslip 1 - March 2026")
      └── ContentDocumentLink → ContentVersion (PDF)
 └── Financial_Asset__c ("Tax Return FY2025")
      └── ContentDocumentLink → ContentVersion (PDF)
 └── Financial_Asset__c ("Superannuation Balance")
      (value only — no document)
```

---

## 4. Custom Fields on Lead (7 fields only)

### Qualification Fields
| Field | Type | Purpose |
|---|---|---|
| `OTP_Verified__c` | Checkbox | Convenience flag — true when OTP verified |
| `Qualification_Status__c` | Picklist | Not Started / Identity Collected / OTP Verified / Documents Pending / Qualified / Not Qualified / Customer Update Requested |
| `Has_Partner__c` | Checkbox | Whether applicant has a partner |
| `Number_Of_Dependents__c` | Number(2,0) | Number of dependents |
| `Qualification_Score__c` | Number(5,1) | Computed score |
| `Qualification_Result__c` | Text(255) | Outcome text |
| `Qualification_Date__c` | DateTime | When qualification was run |

### Roll-Up Summary Fields (OOTB Master-Detail)
| Field | Type | Filter |
|---|---|---|
| `Total_Assets__c` | Roll-Up Summary (SUM) | SUM of Asset_Value__c on all Financial_Asset__c |
| `Total_Income__c` | Roll-Up Summary (SUM) | SUM of Asset_Value__c WHERE Asset_Category__c IN (Gross_Income, Partner_Income) |
| `Documents_Pending_Count__c` | Roll-Up Summary (COUNT) | WHERE Document_Required__c = true AND Document_Uploaded__c = false |
| `Stale_Assets_Count__c` | Roll-Up Summary (COUNT) | WHERE Is_Stale__c = true AND Updated_By_Customer__c = false |

---

## 5. Experience Cloud Self-Service Portal

### Purpose

Customers access a portal to:
1. **View** their application status and qualification progress
2. **Upload** documents for financial assets (payslips, tax returns)
3. **Update** financial data that has been flagged as stale/not recent by a verifier
4. **Re-upload** documents if previous ones were rejected

### Portal Pages (OOTB Lightning Components + Screen Flows)

#### Page 1: Application Dashboard (`/s/my-application`)

**Components:**
- **Record Detail (Lead)** — read-only view of: Name, Email, Qualification_Status__c, Qualification_Score__c, Qualification_Result__c
- **Related List (Financial_Asset__c)** — all financial assets with columns: Category, Value, Status, Document Uploaded, Is Stale
- **Custom Banner/Alert** — if `Stale_Assets_Count__c > 0`, show: "Some of your financial information needs updating. Please review the items marked below."

#### Page 2: Financial Asset Detail (`/s/financial-asset/{recordId}`)

**Components:**
- **Record Detail (Financial_Asset__c)** — shows category, value, period, status
- **Files component** — shows attached documents
- **Embedded Screen Flow: `EBF_Portal_Update_Asset`** — allows customer to update value, re-upload document

#### Page 3: Document Upload (`/s/upload-documents`)

**Components:**
- **Embedded Screen Flow: `EBF_Portal_Document_Upload`** — accepts URL params `leadId` and `assetId`, shows file upload component

### Portal Screen Flows (3 flows)

#### `EBF_Portal_Update_Asset` (Screen Flow)

Triggered when customer clicks "Update" on a stale financial asset.

1. **Screen 1: Update Financial Information**
   - Input field: New `Asset_Value__c` (pre-filled with current value)
   - Input field: New `Description__c`
   - Input field: New `Period__c`
   - File Upload component (optional — for re-uploading documents)
2. **Update Records** — Update Financial_Asset__c:
   - `Asset_Value__c` = new value
   - `Description__c` = new description
   - `Is_Stale__c` = false
   - `Status__c` = "Pending" (back in review queue)
   - `Updated_By_Customer__c` = true
   - `Last_Customer_Update__c` = NOW()
3. **Decision** — If file was uploaded → update `Document_Uploaded__c` = true
4. **Screen 2: Confirmation** — "Your information has been updated and will be reviewed."

#### `EBF_Portal_Document_Upload` (Screen Flow)

For initial document uploads (linked from Agentforce chat).

1. **Get Records** — Fetch Financial_Asset__c by `assetId` URL param
2. **Screen 1** — Show asset details (category, description) + Lightning File Upload (`relatedRecordId` = Financial_Asset__c Id)
3. **Update Records** — Set `Document_Uploaded__c` = true, `Status__c` = "Document_Uploaded"
4. **Screen 2: Confirmation** — "Document uploaded successfully. You can return to the chat."

#### `EBF_Portal_OTP_Login` (Screen Flow)

OTP-based portal authentication (reuses the generic OTP system).

1. **Screen 1** — Collect email address
2. **Get Records** — Find Lead by email
3. **Sub-flow** — Call `OTP_Send` with `relatedRecordId` = Lead Id, `configName` = "Portal_Login"
4. **Screen 2** — Enter OTP code
5. **Sub-flow** — Call `OTP_Verify`
6. **Decision** — SUCCESS → redirect to Application Dashboard; INVALID → show error + retry

### Portal Access Control

- **Guest User Profile** — read-only access to Lead (own record), Financial_Asset__c (own Lead's children)
- **OTP verification required** before any data is shown (via `EBF_Portal_OTP_Login` flow on landing page)
- **Sharing Rules** — Lead owner shares with community user; Financial_Asset__c inherits via Master-Detail
- **FLS** — `OTP_Code__c` on OTP_Verification__c hidden from portal profile; sensitive Lead fields hidden

### Stale Data Workflow

When an internal verifier reviews financial assets and finds them outdated:

1. Verifier sets `Is_Stale__c` = true and fills `Stale_Reason__c` on the Financial_Asset__c record
2. `Stale_Assets_Count__c` roll-up on Lead increments
3. `Qualification_Status__c` updated to "Customer Update Requested"
4. **Record-Triggered Flow: `EBF_Notify_Customer_Stale`** — sends email to Lead with portal link, listing which items need updating
5. Customer logs into portal → sees flagged items → updates → flow resets `Is_Stale__c`
6. Agent or verifier can re-run qualification once all stale items are resolved

---

## 6. Custom Metadata Types (3 types)

### `EBF_Qualification_Threshold__mdt`
Admin-configurable scoring thresholds and weights.

| Record | Minimum_Value | Score_Weight |
|---|---|---|
| Min_Total_Income | 60,000 | 0.30 |
| Min_Total_Assets | 50,000 | 0.25 |
| Max_Dependents_Tier1 | 2 (uses Maximum_Value) | 0.10 |
| Min_Savings_Ratio | 0.20 | 0.15 |
| Investment_Property_Bonus | 1 | 0.20 |
| Qualification_Pass_Score | 60 | — |

Fields: `Minimum_Value__c`, `Maximum_Value__c`, `Score_Weight__c`, `Is_Active__c`

### `EBF_Asset_Category__mdt`
Defines each asset category's properties.

| DeveloperName | Label | Requires_Document | Max_Records |
|---|---|---|---|
| Cash | Cash on Hand | No | 1 |
| Term_Deposit | Term Deposit | No | 1 |
| Payslip | Payslip | Yes | 2 |
| Partner_Payslip | Partner Payslip | Yes | 2 |
| Tax_Return | Tax Return | Yes | 2 |
| Gross_Income | Annual Gross Income | No | 1 |
| Partner_Income | Partner Annual Income | No | 1 |
| Superannuation | Superannuation | No | 1 |
| Savings_Account | Savings Account | No | 1 |
| Cryptocurrency | Cryptocurrency | No | 1 |
| Investment_Property | Investment Property | No | 1 |

Fields: `Category_Label__c`, `Requires_Document__c`, `Display_Order__c`, `Is_Active__c`, `Max_Records__c`

### `OTP_Config__mdt` (generic — no EBF prefix)
Per-context OTP settings.

| DeveloperName | Expiry_Minutes | Max_Attempts | Email_Template |
|---|---|---|---|
| Default | 10 | 3 | OTP_Verification_Email |
| Lead_Qualification | 10 | 3 | EBF_OTP_Verification |
| Case_Verification | 5 | 3 | Case_OTP_Verification |
| Portal_Login | 15 | 5 | Portal_OTP_Verification |

Fields: `OTP_Length__c`, `Expiry_Minutes__c`, `Max_Attempts__c`, `Email_Template_Name__c`, `Channel__c`, `Is_Active__c`

---

## 7. Flow Architecture (18 Flows)

### Reusable OTP Flows (generic — used everywhere)
1. **`OTP_Send`** — Creates OTP_Verification__c record, sends email. Inputs: relatedRecordId, relatedObjectType, recipientEmail, configName
2. **`OTP_Verify`** — Validates entered code against OTP_Verification__c. Inputs: otpRecordId, enteredCode. Returns: verificationResult

### Agentforce Action Flows
3. **`EBF_Create_Or_Update_Lead`** — Dedup by email → create or update Lead
4. **`EBF_Create_Financial_Asset`** — Creates Financial_Asset__c record for any category. Sets Document_Required from CMT lookup.
5. **`EBF_Request_Document_Upload`** — Returns Experience Cloud upload URL
6. **`EBF_Run_Qualification`** — Reads roll-ups + CMT thresholds → scores → updates Lead → calls `EBF_Lead_Assignment` with Stage 3
7. **`EBF_Save_Lead_Details`** — Saves Has_Partner__c and Number_Of_Dependents__c
8. **`EBF_Update_OTP_Status`** — After OTP_Verify returns SUCCESS, sets Lead.OTP_Verified__c = true

### Lead Assignment Flows
9. **`EBF_Lead_Assignment`** — **Reusable sub-flow.** Inputs: leadId, assignmentStage. Reads CMT rules → matches by stage/score/partner → assigns to Queue or round-robin user → creates Task → sends notification
10. **`EBF_Round_Robin_Assign`** — Sub-flow. Inputs: queueApiName. Gets queue members, reads EBF_Assignment_Counter__c, calculates next user via MOD, increments counter. Returns: assignedUserId
11. **`EBF_Lead_Assign_Stage1`** — Record-Triggered on Lead: OTP_Verified__c changes to true → calls EBF_Lead_Assignment (Stage_1_Triage)
12. **`EBF_Lead_Docs_Complete`** — Record-Triggered on Lead: Documents_Pending_Count__c changes to 0 → calls EBF_Lead_Assignment (Stage_2_Doc_Review)
13. **`EBF_Lead_Assign_Stage3`** — Record-Triggered on Lead: Qualification_Status__c changes to Qualified/Not Qualified → calls EBF_Lead_Assignment (Stage_3)

### Supporting / Record-Triggered Flows
14. **`EBF_After_Document_Upload`** — Record-Triggered on ContentDocumentLink → sets Document_Uploaded__c on parent Financial_Asset__c
15. **`EBF_Notify_Customer_Stale`** — Record-Triggered on Financial_Asset__c when Is_Stale__c changes to true → sends email with portal link

### Experience Cloud Portal Flows
16. **`EBF_Portal_OTP_Login`** — Screen Flow: email → OTP send → OTP verify → redirect to dashboard
17. **`EBF_Portal_Document_Upload`** — Screen Flow: file upload linked to Financial_Asset__c
18. **`EBF_Portal_Update_Asset`** — Screen Flow: update value/description, re-upload doc, clear stale flag

---

## 8. Qualification Scoring

The `EBF_Run_Qualification` flow:

1. **Get Records** — Read `Total_Assets__c`, `Total_Income__c`, `Documents_Pending_Count__c`, `Stale_Assets_Count__c` from Lead
2. **Get Records** — All active `EBF_Qualification_Threshold__mdt` records
3. **Decision: Stale check** — `Stale_Assets_Count__c > 0` → return "Qualification Blocked — Customer update pending"
4. **Decision: Document gate** — `Documents_Pending_Count__c > 0` → return "Documents Pending"
5. **Decision elements** per criterion:

| Criterion | Weight | Points |
|---|---|---|
| Total Income >= $60K | 0.30 | 30 |
| Total Assets >= $50K | 0.25 | 25 |
| Dependents <= 2 | 0.10 | 10 |
| Savings/Income >= 20% | 0.15 | 15 |
| Has Investment Property | 0.20 | 20 |

6. **Pass/Fail** — Score >= `Qualification_Pass_Score` CMT value (default 60)
7. **Update Records** — Save score, result, date, status on Lead

---

## 9. Lead Handover & Assignment Plan

### Overview

Lead assignment happens at **three trigger points** during the qualification lifecycle, routing the Lead to the right Queue or individual owner based on stage, score, and financial profile. All routing rules are driven by **Custom Metadata** so admins can adjust without touching Flows.

### Trigger Points

```
 ┌──────────────┐     ┌───────────────┐     ┌────────────────────┐
 │ STAGE 1      │     │ STAGE 2       │     │ STAGE 3            │
 │ Lead Created │────►│ Docs Submitted│────►│ Qualification Run  │
 │ (Identity +  │     │ (All required │     │ (Score computed,   │
 │  OTP verified│     │  docs uploaded│     │  Qualified or Not) │
 │  )           │     │  )            │     │                    │
 └──────┬───────┘     └──────┬────────┘     └──────┬─────────────┘
        │                    │                     │
        ▼                    ▼                     ▼
 ┌──────────────┐     ┌───────────────┐     ┌────────────────────┐
 │ Triage Queue │     │ Document      │     │ Score-based routing│
 │ (initial     │     │ Review Queue  │     │ to Loan Officer    │
 │  screening)  │     │ (verify docs) │     │ Queue or individual│
 └──────────────┘     └───────────────┘     └────────────────────┘
```

### Stage 1: After Lead Creation + OTP Verification

**When:** `EBF_Update_OTP_Status` flow sets `OTP_Verified__c = true`

**Routing:**
- All newly verified Leads → **`EBF_Triage_Queue`**
- This is a holding queue for the pre-qualification team to monitor incoming applications
- The Lead stays here while the customer provides financial data via Agentforce

**Flow:** `EBF_Lead_Assignment` (sub-flow called at each stage)

### Stage 2: After All Documents Submitted

**When:** `Documents_Pending_Count__c` drops to 0 (flow-based roll-up update detects this)

**Routing:**
- Lead moves from Triage → **`EBF_Document_Review_Queue`**
- Document reviewers verify authenticity, flag stale items
- If stale items found → customer notified, Lead stays in this queue until resolved

**Trigger:** Record-Triggered Flow `EBF_Lead_Docs_Complete` fires when `Documents_Pending_Count__c` changes to 0

### Stage 3: After Qualification Scoring

**When:** `EBF_Run_Qualification` flow completes and sets `Qualification_Status__c` to "Qualified" or "Not Qualified"

**Routing:** Score-based assignment using `EBF_Lead_Assignment_Rule__mdt`:

| Scenario | Score Range | Route To | Reason |
|---|---|---|---|
| Premium Qualified | 80–100 | `EBF_Senior_Loan_Officers` Queue | High-value, fast-track |
| Standard Qualified | 60–79 | `EBF_Loan_Officers` Queue | Standard processing |
| Borderline | 50–59 | `EBF_Assessment_Review` Queue | Manual review needed |
| Not Qualified | 0–49 | `EBF_Nurture_Queue` | Nurture campaign / follow-up |
| Joint Application (has partner) | Any qualified | `EBF_Joint_Application_Officers` Queue | Specialist for joint apps |

### Custom Metadata: `EBF_Lead_Assignment_Rule__mdt`

Configurable routing rules — admins change assignment without modifying Flows.

| Field | Type | Purpose |
|---|---|---|
| `Rule_Name__c` | Text(100) | Descriptive name |
| `Stage__c` | Picklist | Stage_1_Triage / Stage_2_Doc_Review / Stage_3_Qualified / Stage_3_Not_Qualified |
| `Min_Score__c` | Number(5,1) | Minimum qualification score (null for stages 1-2) |
| `Max_Score__c` | Number(5,1) | Maximum qualification score (null for stages 1-2) |
| `Has_Partner__c` | Checkbox | If true, rule only applies to joint applications |
| `Priority__c` | Number(2,0) | Rule evaluation order (lower = higher priority) |
| `Target_Queue_API_Name__c` | Text(100) | Queue DeveloperName to assign to |
| `Use_Round_Robin__c` | Checkbox | If true, assign to individual user within queue via round-robin |
| `Notification_Template__c` | Text(100) | Email template for handover notification |
| `Is_Active__c` | Checkbox | Toggle rule on/off |
| `SLA_Hours__c` | Number(3,0) | Expected response time (for reporting/escalation) |

**Records:**

| DeveloperName | Stage | Min Score | Max Score | Has Partner | Target Queue | Round Robin | SLA Hours |
|---|---|---|---|---|---|---|---|
| `Triage_Default` | Stage_1_Triage | — | — | false | EBF_Triage_Queue | No | 24 |
| `Doc_Review_Default` | Stage_2_Doc_Review | — | — | false | EBF_Document_Review_Queue | No | 48 |
| `Premium_Qualified` | Stage_3_Qualified | 80 | 100 | false | EBF_Senior_Loan_Officers | Yes | 4 |
| `Standard_Qualified` | Stage_3_Qualified | 60 | 79 | false | EBF_Loan_Officers | Yes | 8 |
| `Joint_Qualified` | Stage_3_Qualified | 60 | 100 | true | EBF_Joint_Application_Officers | Yes | 8 |
| `Borderline_Review` | Stage_3_Qualified | 50 | 59 | false | EBF_Assessment_Review | No | 24 |
| `Not_Qualified` | Stage_3_Not_Qualified | 0 | 49 | false | EBF_Nurture_Queue | No | 72 |

### Queues (5 Lead Queues)

| Queue API Name | Label | Members | Purpose |
|---|---|---|---|
| `EBF_Triage_Queue` | Triage | All loan admin staff | New applications awaiting financial data |
| `EBF_Document_Review_Queue` | Document Review | Document verification team | Verify payslips, tax returns, flag stale data |
| `EBF_Senior_Loan_Officers` | Senior Loan Officers | Senior staff | Premium/high-score applications |
| `EBF_Loan_Officers` | Loan Officers | All loan officers | Standard qualified applications |
| `EBF_Joint_Application_Officers` | Joint Application Specialists | Joint-app trained officers | Partner/joint applications |
| `EBF_Assessment_Review` | Assessment Review | Senior assessors | Borderline scores needing manual review |
| `EBF_Nurture_Queue` | Nurture | Marketing / follow-up team | Not qualified — nurture or re-engage |

### Round-Robin Assignment (Declarative)

For queues with `Use_Round_Robin__c = true`, the assignment flow distributes Leads evenly among queue members.

**Mechanism:** A custom field `Round_Robin_Counter__c` (Number) on a helper custom object `EBF_Assignment_Counter__c` tracks the last-assigned index per queue.

**`EBF_Assignment_Counter__c`** (simple helper object):
| Field | Type | Purpose |
|---|---|---|
| `Name` | Text | Queue API Name |
| `Last_Assigned_Index__c` | Number(4,0) | Rotating counter |

**How it works in Flow:**
1. Get the `EBF_Assignment_Counter__c` record for the target queue
2. Get all active Users who are members of the queue (via GroupMember where GroupId = Queue Id)
3. Calculate next index: `MOD(Last_Assigned_Index__c, total_members)`
4. Get the User at that index position
5. Assign Lead.OwnerId to that User
6. Increment `Last_Assigned_Index__c` + 1, update counter record

### Lead Assignment Flow: `EBF_Lead_Assignment` (Autolaunched Sub-flow)

**Inputs:** `leadId` (Text), `assignmentStage` (Text — "Stage_1_Triage" / "Stage_2_Doc_Review" / "Stage_3_Qualified" / "Stage_3_Not_Qualified")

This is the **single reusable assignment flow** called from multiple trigger points.

**Steps:**

1. **Get Records** — Fetch Lead (Score, Has_Partner, Qualification_Status)
2. **Get Records** — All active `EBF_Lead_Assignment_Rule__mdt` where `Stage__c` = `assignmentStage`, ordered by `Priority__c` ASC
3. **Loop** through rules:
   - **Decision: Score in range?** — If Stage 3: check `Min_Score <= Lead score <= Max_Score`. Stages 1-2: skip score check.
   - **Decision: Partner match?** — If rule `Has_Partner__c` = true, check `Lead.Has_Partner__c` = true. If rule is false, always matches.
   - **First matching rule wins** (lowest priority number) → store `Target_Queue_API_Name__c`
4. **Get Records** — Fetch the Queue record by DeveloperName
5. **Decision: Round Robin?**
   - **Yes:** Call sub-flow `EBF_Round_Robin_Assign` with Queue Id → returns specific User Id → set Lead.OwnerId
   - **No:** Set Lead.OwnerId = Queue Id
6. **Update Records** — Update Lead OwnerId
7. **Assignment** — Set `Assignment_Stage__c` on Lead to current stage
8. **Create Records** — Create a Task for the new owner:
   - Subject: "New Lead Assignment — {Lead Name} — {Stage}"
   - Description: "Qualification Score: {score}. SLA: {SLA_Hours} hours."
   - Due Date: NOW() + SLA_Hours
   - Priority: High (for Premium), Normal (for Standard), Low (for Nurture)
9. **Send Email** — Notification to new owner using the template from the matching CMT rule

### New Lead Fields for Assignment

| Field | Type | Purpose |
|---|---|---|
| `Assignment_Stage__c` | Picklist | Stage_1_Triage / Stage_2_Doc_Review / Stage_3_Qualified / Stage_3_Not_Qualified |
| `Assigned_DateTime__c` | DateTime | When last assignment occurred |
| `Assignment_SLA_Due__c` | DateTime | When SLA expires (NOW + SLA_Hours from rule) |
| `Previous_Owner__c` | Text(18) | Previous OwnerId before handover (audit trail) |

### Trigger Flows for Assignment

#### `EBF_Lead_Assign_Stage1` (Record-Triggered Flow on Lead)

- **Trigger:** After Update, where `OTP_Verified__c` changes to `true`
- **Action:** Call sub-flow `EBF_Lead_Assignment` with stage = "Stage_1_Triage"

#### `EBF_Lead_Docs_Complete` (Record-Triggered Flow on Lead)

- **Trigger:** After Update, where `Documents_Pending_Count__c` changes to `0` AND was previously > 0
- **Action:** Call sub-flow `EBF_Lead_Assignment` with stage = "Stage_2_Doc_Review"

#### `EBF_Lead_Assign_Stage3` (Record-Triggered Flow on Lead)

- **Trigger:** After Update, where `Qualification_Status__c` changes to `Qualified` or `Not Qualified`
- **Decision:**
  - Qualified → Call `EBF_Lead_Assignment` with stage = "Stage_3_Qualified"
  - Not Qualified → Call `EBF_Lead_Assignment` with stage = "Stage_3_Not_Qualified"

### Handover Notifications

Each stage transition sends an email + creates a Task:

| Stage | Notification To | Email Template | Task Subject |
|---|---|---|---|
| Stage 1 → Triage | Triage Queue members | `EBF_Assignment_Triage` | "New Application: {Lead Name}" |
| Stage 2 → Doc Review | Doc Review Queue/User | `EBF_Assignment_Doc_Review` | "Documents Ready for Review: {Lead Name}" |
| Stage 3 → Loan Officer | Assigned Loan Officer | `EBF_Assignment_Qualified` | "Qualified Lead: {Lead Name} — Score {Score}" |
| Stage 3 → Nurture | Nurture Queue | `EBF_Assignment_Nurture` | "Follow-up Lead: {Lead Name}" |

### Escalation (OOTB Escalation Rules)

Use standard **Salesforce Escalation Rules** on Lead to handle SLA breaches:

- **Rule:** If `Assignment_SLA_Due__c` < NOW() AND `Qualification_Status__c` != "Qualified" AND `Qualification_Status__c` != "Not Qualified"
- **Action:** Reassign to manager queue, send escalation email
- Configured in Setup → Lead → Escalation Rules (point-and-click, no metadata file needed)

### Lead Handover Lifecycle Summary

```
Customer starts Agentforce conversation
        │
        ▼
┌─────────────────────────┐
│ 1. Identity + OTP       │  Owner: Default (Agentforce system user)
└────────┬────────────────┘
         │ OTP_Verified__c = true
         ▼
┌─────────────────────────┐
│ 2. Triage Queue         │  Owner: EBF_Triage_Queue
│    SLA: 24 hours        │  Monitor while customer provides financial data
└────────┬────────────────┘
         │ Documents_Pending_Count = 0
         ▼
┌─────────────────────────┐
│ 3. Document Review      │  Owner: EBF_Document_Review_Queue
│    SLA: 48 hours        │  Verify docs, flag stale items
│                         │  ─── if stale → customer updates via portal ───┐
│                         │  ◄── stale resolved, re-enters this stage ─────┘
└────────┬────────────────┘
         │ Verifier runs qualification
         ▼
┌─────────────────────────┐
│ 4a. Qualified           │  Score-based routing:
│     80-100: Senior LO   │  Round-robin to individual loan officer
│     60-79: Standard LO  │  SLA: 4-8 hours
│     Joint: Specialist   │  Task + email notification
├─────────────────────────┤
│ 4b. Borderline (50-59)  │  Owner: EBF_Assessment_Review
│     Manual review       │  SLA: 24 hours
├─────────────────────────┤
│ 4c. Not Qualified       │  Owner: EBF_Nurture_Queue
│     (0-49)              │  SLA: 72 hours (follow-up / re-engagement)
└─────────────────────────┘
         │
         │ Lead Conversion (if qualified)
         ▼
┌─────────────────────────┐
│ Account + Contact       │  Financial_Asset__c reparented to Account
│ Opportunity created     │  Assignment fields copied
└─────────────────────────┘
```

---

## 10. Metadata File Structure

```
force-app/main/default/
├── objects/
│   ├── Lead/fields/                          — 11 fields + 4 roll-ups + 4 assignment + 7 attribution = 26 .field-meta.xml
│   ├── Financial_Asset__c/
│   │   ├── Financial_Asset__c.object-meta.xml
│   │   └── fields/                           — 14 .field-meta.xml
│   ├── OTP_Verification__c/
│   │   ├── OTP_Verification__c.object-meta.xml
│   │   └── fields/                           — 11 .field-meta.xml
│   ├── EBF_Assignment_Counter__c/
│   │   ├── EBF_Assignment_Counter__c.object-meta.xml
│   │   └── fields/                           — 1 .field-meta.xml (Last_Assigned_Index__c)
│   ├── EBF_Qualification_Threshold__mdt/     — object + 4 fields
│   ├── EBF_Asset_Category__mdt/              — object + 5 fields
│   ├── EBF_Lead_Assignment_Rule__mdt/        — object + 11 fields
│   ├── EBF_Campaign_Mapping__mdt/            — object + 4 fields
│   └── OTP_Config__mdt/                      — object + 6 fields (includes WhatsApp config)
│
├── customMetadata/                           — 33+ records (6 thresholds + 11 categories + 5 OTP + 7 assignment + 3+ campaigns)
├── queues/                                   — 7 .queue-meta.xml files
├── flows/                                    — 21 .flow-meta.xml (20 + 1 scheduled)
├── email/
│   ├── OTP_Email_Templates/                  — generic OTP templates
│   ├── EBF_Email_Templates/                  — EBF qualification templates
│   └── EBF_Assignment_Templates/             — handover notification templates (4)
├── genAiPlugins/                             — 3 Topic definitions
├── genAiFunctions/                           — 8 Action definitions
├── genAiPlanners/                            — 1 Agent planner config
├── sites/                                    — Experience Cloud site definition
├── networks/                                 — Experience Cloud network config
├── experiences/                              — Experience Cloud pages/theme
│   └── EBF_Customer_Portal/
│       ├── views/
│       │   ├── myApplication.json            — dashboard page
│       │   ├── financialAssetDetail.json     — asset detail page
│       │   ├── uploadDocuments.json          — upload page
│       │   └── login.json                    — OTP login page
│       └── config/
│           └── mainNavigationMenu.json
└── profiles/
    └── EBF_Portal_User.profile-meta.xml      — portal user profile with FLS
```

---

## 10. Implementation Order

1. **OTP_Verification__c custom object + fields** — standalone, no dependencies
2. **OTP_Config__mdt** — CMT definition + 4 config records
3. **OTP Flows** — OTP_Send + OTP_Verify (generic, testable immediately)
4. **Financial_Asset__c custom object + fields**
5. **EBF_Assignment_Counter__c custom object** — round-robin helper
6. **Lead custom fields** (7 qual + 4 roll-ups + 4 assignment = 15 fields)
7. **EBF Custom Metadata Types** — Qualification_Threshold + Asset_Category + records
8. **EBF_Lead_Assignment_Rule__mdt** — CMT definition + 7 rule records
9. **Queues** — 7 Lead queues
10. **Email templates** (OTP + stale notification + 4 assignment handover templates)
11. **Agentforce flows:** Create_Or_Update_Lead, Create_Financial_Asset, Save_Lead_Details, Update_OTP_Status
12. **Document flows:** Request_Document_Upload, After_Document_Upload
13. **Assignment sub-flows:** EBF_Round_Robin_Assign, EBF_Lead_Assignment
14. **Assignment trigger flows:** EBF_Lead_Assign_Stage1, EBF_Lead_Docs_Complete, EBF_Lead_Assign_Stage3
15. **Qualification flow:** Run_Qualification
16. **Stale notification flow:** Notify_Customer_Stale
17. **Experience Cloud portal:** site setup, pages, portal flows
18. **Agentforce metadata:** genAiFunctions → genAiPlugins → genAiPlanners

---

## 10a. Campaign Entry — Meta (Facebook) Ads

### Overview

Meta/Facebook ads drive traffic to an Experience Cloud landing page where the Agentforce agent is embedded. The full qualification happens in web chat. UTM parameters are captured for attribution and Salesforce Campaign membership.

```
┌───────────────────────────────────────────────────────────────────┐
│                     META ADS MANAGER                               │
│     Campaign: "Equity Builder — Pre-Qualify Now"                   │
│     Audience: Homebuyers, 25-55, AU, Interest: property/finance    │
│     Placements: Facebook Feed / Instagram Stories / Reels          │
└───────────────┬───────────────────────────────────────────────────┘
                │
                │  CTA: "Check If You Qualify"
                │  URL: https://ebf-portal.my.site.com/s/apply
                │       ?utm_source=facebook
                │       &utm_medium=cpc
                │       &utm_campaign=equity_builder_spring_2026
                │       &utm_content=video_ad_v2
                │
                ▼
┌───────────────────────────────────────────────────────────────────┐
│  Experience Cloud Landing Page: /s/apply                          │
│                                                                    │
│  ┌──────────────────────────────────────────────────────┐         │
│  │  Hero: "Find out if you qualify for equity building   │         │
│  │        in under 10 minutes"                           │         │
│  │                                                       │         │
│  │  Trust signals: AFCA member / Licensed / 4.8★         │         │
│  └──────────────────────────────────────────────────────┘         │
│                                                                    │
│  ┌──────────────────────────────────────────────┐                 │
│  │  Embedded Agentforce Chat (auto-opens)       │                 │
│  │                                               │                 │
│  │  "Welcome to Equity Builder Finance!          │                 │
│  │   I'll help you check if you qualify.         │                 │
│  │   What's your first name?"                    │                 │
│  └──────────────────────────────────────────────┘                 │
│                                                                    │
│  FAQ accordion / How it works steps (below fold)                  │
└───────────────────────────────────────────────────────────────────┘
```

### Meta Ads Configuration

#### Ad Account Setup

| Setting | Value |
|---|---|
| Campaign Objective | Lead Generation (optimise for conversions) |
| Conversion Event | Custom: `EBF_Qualification_Started` (Meta Pixel event fired when agent opens) |
| Audience | Lookalike based on existing qualified Leads (exported from SF) |
| Placements | Facebook Feed, Instagram Feed, Instagram Stories, Reels |
| Landing Page | `https://ebf-portal.my.site.com/s/apply?utm_source=facebook&utm_medium=cpc&utm_campaign={campaign_name}&utm_content={ad_name}` |

#### Meta Pixel on Experience Cloud

Install the **Meta Pixel** on the Experience Cloud site to track conversions:

1. **Experience Builder → Settings → Head Markup** — add Meta Pixel base code
2. **Custom events** fired by the landing page (via Experience Cloud Lightning component or page-level JS):
   - `EBF_PageView` — when `/s/apply` loads
   - `EBF_Qualification_Started` — when the Agentforce chat widget opens
   - `EBF_OTP_Verified` — when OTP verification succeeds (fired by flow updating a hidden page element)
   - `EBF_Qualification_Complete` — when qualification result is returned

These events feed back to Meta for **conversion optimisation** — Meta learns which audiences complete qualification and targets similar users.

#### Ad Creative Strategy

| Ad Format | Hook | CTA |
|---|---|---|
| Video (15s) | "In under 10 minutes, find out if you can build equity" | Check If You Qualify |
| Carousel | Step 1: Tell us about yourself → Step 2: Upload payslips → Step 3: Get your result | Get Started |
| Static Image | "Your home equity journey starts with a 10-minute chat" | Pre-Qualify Now |
| Instagram Story | Swipe-up with countdown timer animation | Apply Now |

### Salesforce-Side Setup

#### Experience Cloud Landing Page: `/s/apply`

**Setup (all OOTB, no code):**

1. **Embedded Service Deployment** (Setup → Embedded Service Deployments):
   - Create deployment → connect to `Equity_Builder_Qualification_Agent`
   - Enable for unauthenticated (Guest) users
   - Configure **Hidden Pre-Chat Fields** to capture URL params:

   | Pre-Chat Field | URL Parameter | Maps To |
   |---|---|---|
   | `utm_source` | `utm_source` | Passed to agent flow as input |
   | `utm_medium` | `utm_medium` | Passed to agent flow as input |
   | `utm_campaign` | `utm_campaign` | Passed to agent flow as input |
   | `utm_content` | `utm_content` | Passed to agent flow as input |

2. **Experience Builder** → Edit `/s/apply` page:
   - Add "Embedded Service" component → select the deployment
   - Configure: auto-open on page load = true, position = bottom-right
   - Add hero section, trust signals, FAQ content (standard CMS components)

3. **Guest User Profile** → grant:
   - Read on Lead (limited fields)
   - Create on Lead, Financial_Asset__c, OTP_Verification__c (via Flow — runs in system context)

#### UTM Capture: How Pre-Chat Fields Flow to the Lead

```
Meta Ad Click
    │
    ▼
/s/apply?utm_source=facebook&utm_campaign=equity_builder_spring_2026
    │
    │ Experience Cloud reads URL params
    │ Embedded Service passes them as hidden pre-chat fields
    │
    ▼
Agentforce Session starts (pre-chat fields available)
    │
    │ Agent collects Name, Phone, Email
    │ Calls EBF_Create_Or_Update_Lead flow
    │ Flow receives pre-chat UTM fields as inputs
    │
    ▼
Lead record created with:
  UTM_Source__c = "facebook"
  UTM_Medium__c = "cpc"
  UTM_Campaign__c = "equity_builder_spring_2026"
  UTM_Content__c = "video_ad_v2"
  Entry_Channel__c = "Web_Chat"
  + CampaignMember linking Lead → SF Campaign
```

### Meta Lead Ads (Alternative — Native Facebook Form)

For users who don't want to leave Facebook, **Meta Lead Ads** capture basic info natively:

```
Meta Lead Ad (in-app form)
    │
    │ Collects: Name, Email, Phone (pre-filled from Facebook profile)
    │
    ▼
Salesforce Connector (Meta → SF integration)
    │ OR: Zapier / Marketing Cloud Connect
    │
    ▼
Web-to-Lead creates Lead in Salesforce
    │
    ▼
Record-Triggered Flow: EBF_Meta_Lead_Follow_Up
    │
    │ Sends email: "Thanks for your interest! Continue your
    │ application here: [portal link with lead token]"
    │
    ▼
Customer clicks link → /s/apply?leadId={id}&token={otp}
    │
    ▼
Agentforce agent resumes qualification
(skips identity collection — Lead already exists)
```

**New Flow: `EBF_Meta_Lead_Follow_Up`** (Record-Triggered on Lead):
- **Trigger:** After Insert where `UTM_Source__c` = "facebook" AND `LeadSource` = "Web"
- Sends email with portal link to continue application
- Sets `Qualification_Status__c` = "Identity Collected"

### UTM / Campaign Attribution

**New Lead Fields:**

| Field | Type | Purpose |
|---|---|---|
| `UTM_Source__c` | Text(100) | facebook / google / email |
| `UTM_Medium__c` | Text(100) | cpc / organic / email |
| `UTM_Campaign__c` | Text(100) | Campaign identifier |
| `UTM_Content__c` | Text(100) | Ad variant / creative name |
| `Entry_Channel__c` | Picklist | Web_Chat / Portal / Meta_Lead_Ad |
| `Campaign_Landing_URL__c` | URL | Full landing page URL |

### Custom Metadata: `EBF_Campaign_Mapping__mdt`

Maps UTM campaign strings to Salesforce Campaign records for automatic CampaignMember creation.

| Field | Type |
|---|---|
| `UTM_Campaign_Value__c` | Text(100) |
| `SF_Campaign_Id__c` | Text(18) |
| `Channel__c` | Text(50) |
| `Is_Active__c` | Checkbox |

**Records:**
| DeveloperName | UTM Value | Channel |
|---|---|---|
| `EBF_Spring_2026_FB` | equity_builder_spring_2026 | facebook |
| `EBF_Spring_2026_IG` | equity_builder_spring_2026_ig | instagram |
| `EBF_Retarget_FB` | ebf_retarget_q2 | facebook |
| `EBF_Lead_Ad_FB` | ebf_lead_ad_spring | facebook |

### Updated `EBF_Create_Or_Update_Lead` Flow

Extended with attribution:

1. Dedup Lead by email (existing)
2. Create or update Lead with identity fields (existing)
3. **NEW:** Set UTM fields + Entry_Channel from pre-chat inputs
4. **NEW:** Get `EBF_Campaign_Mapping__mdt` where `UTM_Campaign_Value__c` = utm_campaign input
5. **NEW:** If match → Create `CampaignMember` (Lead → Campaign)
6. Return Lead Id

### Retargeting: Abandoned Qualification

For users who start but don't complete qualification:

**Scheduled Flow: `EBF_Abandoned_Qualification_Reminder`**
- **Runs:** Daily
- **Criteria:** Leads where `Qualification_Status__c` IN (Identity Collected, OTP Verified, Documents Pending) AND `LastModifiedDate` < 3 days ago
- **Action:** Send email with portal link to continue
- **After 7 days:** Add to a Salesforce Campaign "Retarget — Abandoned" → export to Meta Custom Audience for retargeting ads

**Meta Custom Audience sync:**
- Salesforce Campaign "Retarget — Abandoned" → export contacts to Meta Custom Audience (via Marketing Cloud or manual CSV export)
- Meta shows retargeting ads to these users: "You're almost there — finish your equity builder application"

### Experience Cloud Page Architecture

```
/s/apply              — Public landing page with embedded agent (Meta ad destination)
/s/login              — OTP login for returning customers
/s/my-application     — Dashboard (authenticated)
/s/financial-asset    — Asset detail + update (authenticated)
/s/upload-documents   — Document upload (authenticated)
```

---

### PHASE 2 (On Hold): WhatsApp Channel

WhatsApp is planned as a future channel that uses the same Agentforce agent. Key advantage: native file sharing eliminates the document upload redirect. Requires Digital Engagement license + Meta Business Account + WhatsApp Business API. Architecture is designed to support it — `OTP_Config__mdt` already has a `Channel__c` field, and `Entry_Channel__c` on Lead accommodates WhatsApp as a value. Implementation deferred.

---

## 10b. Complete Flow Architecture (20 Flows)

### Reusable OTP Flows
1. **`OTP_Send`** — Creates OTP_Verification__c, sends email. Channel field ready for WhatsApp Phase 2.
2. **`OTP_Verify`** — Validates code against OTP_Verification__c

### Agentforce Action Flows
3. **`EBF_Create_Or_Update_Lead`** — Dedup by email → create/update Lead → capture UTM/channel → create CampaignMember
4. **`EBF_Create_Financial_Asset`** — Creates Financial_Asset__c record, sets Document_Required from CMT
5. **`EBF_Request_Document_Upload`** — Returns Experience Cloud upload URL
6. **`EBF_Run_Qualification`** — Scores Lead → updates status → triggers Stage 3 assignment
7. **`EBF_Save_Lead_Details`** — Saves Has_Partner__c and Number_Of_Dependents__c
8. **`EBF_Update_OTP_Status`** — Sets Lead.OTP_Verified__c = true after successful OTP

### Lead Assignment Flows
9. **`EBF_Lead_Assignment`** — Reusable sub-flow: reads CMT rules → matches → assigns to Queue/user
10. **`EBF_Round_Robin_Assign`** — Sub-flow: round-robin within a Queue using counter object
11. **`EBF_Lead_Assign_Stage1`** — Record-Triggered: OTP verified → Triage Queue
12. **`EBF_Lead_Docs_Complete`** — Record-Triggered: all docs uploaded → Document Review Queue
13. **`EBF_Lead_Assign_Stage3`** — Record-Triggered: qualification complete → score-based routing

### Supporting / Record-Triggered Flows
14. **`EBF_After_Document_Upload`** — Record-Triggered on ContentDocumentLink → updates Financial_Asset__c checkbox
15. **`EBF_Notify_Customer_Stale`** — Record-Triggered: Is_Stale changes → sends email notification with portal link
16. **`EBF_Rollup_Financial_Assets`** — Record-Triggered on Financial_Asset__c: computes roll-up totals on Lead/Account
17. **`EBF_Meta_Lead_Follow_Up`** — Record-Triggered on Lead: when UTM_Source = facebook + new Lead → sends email to continue on portal

### Experience Cloud Portal Flows
18. **`EBF_Portal_OTP_Login`** — Screen Flow: OTP-based portal authentication
19. **`EBF_Portal_Document_Upload`** — Screen Flow: file upload for web users
20. **`EBF_Portal_Update_Asset`** — Screen Flow: update stale data + re-upload

### Campaign Flows
21. **`EBF_Abandoned_Qualification_Reminder`** — Scheduled Flow (daily): emails Leads stuck in partial qualification > 3 days; after 7 days adds to retargeting Campaign

---

## 11. Verification

- Deploy to scratch org via `sf project deploy start`
- **OTP system:** Create OTP for a Lead → verify email received → verify code → confirm OTP_Verification__c status. Repeat for a Case to prove reusability.
- **Agentforce:** Test full conversation: identity → OTP → financial data → documents → qualification
- **Lead Assignment:**
  - Verify Stage 1: OTP verified → Lead assigned to Triage Queue
  - Verify Stage 2: All docs uploaded → Lead moves to Document Review Queue
  - Verify Stage 3 (score 85): Lead round-robined to individual Senior Loan Officer
  - Verify Stage 3 (score 65): Lead round-robined to standard Loan Officer
  - Verify Stage 3 (score 55): Lead assigned to Assessment Review Queue
  - Verify Stage 3 (score 40): Lead assigned to Nurture Queue
  - Verify joint application (Has_Partner = true, score 75): Routes to Joint Application Officers
  - Verify Task created on each handover with correct SLA due date
  - Verify email notification sent to new owner
  - Verify round-robin distributes evenly (create 4 Leads, check 4 different officers assigned)
- **Meta Ads Campaign Flow:**
  - Click Meta ad with UTM params → land on `/s/apply` → verify agent auto-opens
  - Complete full qualification → verify UTM fields populated on Lead (Source, Medium, Campaign, Content)
  - Verify CampaignMember created linking Lead to correct Salesforce Campaign via CMT mapping
  - Verify `Entry_Channel__c` = "Web_Chat" on Lead
  - Test Meta Lead Ad path: form submit → Lead created in SF → follow-up email sent → click portal link → agent resumes
  - Verify Meta Pixel fires: PageView, Qualification_Started, OTP_Verified, Qualification_Complete
- **Abandoned Qualification:**
  - Create Lead, complete OTP but stop → wait 3 days → verify reminder email sent
  - Wait 7 days → verify Lead added to "Retarget — Abandoned" Campaign
- **Retargeting:**
  - Export "Retarget — Abandoned" Campaign → verify it contains the right Leads for Meta Custom Audience upload
- **Portal:** Log in via OTP → view dashboard → upload document → verify link → flag stale → customer updates → confirm cleared
- **Qualification:** Test edge cases: partner/no partner, 0 dependents, missing docs, stale assets blocking
- **Roll-ups:** Verify Total_Assets, Total_Income, Documents_Pending_Count, Stale_Assets_Count compute correctly
- **CMT changes:** Modify a threshold → re-run qualification → confirm new threshold applies. Change assignment rule → verify Lead routes to new queue.
- **Escalation:** Let SLA expire → verify escalation rule fires

---

## Key Trade-offs

| Decision | Trade-off |
|---|---|
| OTP_Verification__c (custom object) | More flexible than fields on Lead; slight overhead of extra object, but scales to Case/Contact/any object |
| Polymorphic via Text(18) not Lookup | Can reference any object without multiple lookup fields; trade-off is no referential integrity (acceptable — OTP records are transient) |
| Experience Cloud portal | Requires Digital Experience license; provides proper self-service and solves the file upload limitation |
| Stale data workflow | Adds complexity but enables round-trip verification: internal team flags → customer updates → re-qualify |
| Financial_Asset__c Dual Lookup (not Master-Detail) | Supports Lead conversion to Account; trade-off is Flow-based roll-ups instead of native Roll-Up Summary |
| OTP via RANDOM() | Not cryptographically secure — acceptable for initial launch with FLS lockdown |
| CMT-driven assignment rules | Admins can change routing without modifying Flows; trade-off is rule evaluation in Flow loops (acceptable at Lead volumes) |
| Round-robin via counter object | Simple declarative round-robin; not as sophisticated as Omni-Channel skills-based routing, but zero license cost |
| Assignment at 3 stages | Lead ownership changes 3 times during lifecycle; each handover is audited via Previous_Owner__c + Tasks |
| Meta ads as primary acquisition channel | Requires Meta Pixel on Experience Cloud for conversion tracking; strong ROI visibility via UTM + CampaignMember attribution |
| Meta Lead Ads (native form) as alternative | Lower friction (no page exit) but requires follow-up email to continue in Agentforce — two-step funnel |
| Abandoned qualification retargeting | Adds a scheduled flow + Campaign export overhead; high value for converting drop-offs back via Meta Custom Audiences |
| WhatsApp deferred to Phase 2 | Architecture supports it (Channel fields, OTP Config ready); avoids Digital Engagement license cost in Phase 1 |
