# QualifyAgent

Salesforce Agentforce agent for **Equity Builder Finance** — a declarative lead qualification system that collects identity, verifies via email OTP, gathers comprehensive financial data and documents, scores applicants, and routes qualified leads to the right team.

**100% declarative. No Apex. Flows + Custom Metadata + OOTB features only.**

---

## Architecture

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

### Qualification Flow

```
Customer starts Agentforce conversation
        │
        ▼
  1. Identity + OTP Verification
        │
        ▼
  2. Financial Data Collection
     (cash, income, super, savings, crypto, property)
        │
        ▼
  3. Document Upload
     (payslips, tax returns — via Experience Cloud portal)
        │
        ▼
  4. Weighted Scoring (0-100)
     Thresholds configurable via Custom Metadata
        │
        ▼
  5. Lead Assignment
     Score-based routing to Loan Officer queues
```

---

## Prerequisites

### Salesforce Licenses

| License | Required For | Mandatory |
|---|---|---|
| **Agentforce for Sales / Service** | Running the Agentforce agent, genAiPlugins, genAiFunctions, genAiPlanners metadata | Yes |
| **Einstein 1 Edition** or **Einstein for Sales/Service add-on** | Einstein AI platform that powers Agentforce agent reasoning | Yes |
| **Sales Cloud** or **Service Cloud** | Lead object, Queues, Assignment Rules, Escalation Rules | Yes |
| **Digital Experiences (Community) license** | Experience Cloud portal for document upload and customer self-service | Yes |
| **Digital Engagement add-on** | WhatsApp channel (Phase 2 — not required for initial launch) | No |
| **Marketing Cloud** or **Account Engagement** | Campaign management, Meta Lead Ad sync, nurture journeys | Recommended |

### Salesforce Edition Compatibility

| Edition | Compatible |
|---|---|
| Enterprise Edition + Agentforce add-on | Yes |
| Unlimited Edition | Yes (Agentforce included) |
| Einstein 1 Edition | Yes (Agentforce included) |
| Professional Edition | No — does not support Custom Metadata Types or Agentforce |
| Developer Edition | Yes (for development/testing, Agentforce may need enablement) |

### Permission Sets & Profiles

The following Permission Sets must be assigned to relevant users before deployment:

#### For Agentforce Agent Execution

| Permission Set | Assign To | Purpose |
|---|---|---|
| **AgentforceServiceAgent** (or `Agentforce_Service_Agent_User`) | The Agentforce service user / integration user | Allows the agent to execute genAiFunction actions and invoke Flows |
| **Einstein Agent User** | All users who configure or test the agent in Agent Builder | Access to Agent Builder, topic/action configuration |
| **Flow User** | Agentforce service user | Required for the agent to execute Autolaunched Flows |

#### For Internal Users (Loan Officers, Verifiers, Admins)

| Permission Set / Profile | Assign To | Purpose |
|---|---|---|
| **EBF Loan Officer** (custom — create this) | Loan officers, senior officers, joint app specialists | Read/Edit access to Lead, Financial_Asset__c, OTP_Verification__c. Visibility into assigned Leads via Queue membership |
| **EBF Document Reviewer** (custom — create this) | Document verification team | Read/Edit on Financial_Asset__c (to set Is_Stale__c, verify docs). Read on Lead |
| **EBF Admin** (custom — create this) | System administrators | Full CRUD on all custom objects + Custom Metadata Types. Access to modify EBF_Qualification_Threshold__mdt, EBF_Lead_Assignment_Rule__mdt records |
| **Queue membership** | Add users to the 7 Lead Queues via Setup | Controls which users receive assigned Leads at each stage |

#### For Experience Cloud Portal Users

| Permission Set / Profile | Assign To | Purpose |
|---|---|---|
| **EBF Portal User** (custom — create this) | Customer portal profile / Guest User | Read on own Lead record (limited fields). Read/Edit on own Financial_Asset__c records. Create on ContentVersion (file upload). No access to OTP_Code__c or internal fields |
| **Guest User Profile** | Unauthenticated visitors on `/s/apply` | Interact with embedded Agentforce agent. Create Lead, OTP_Verification__c via Flow (system context). No direct object access |

#### Field-Level Security (Critical)

| Field | Visibility |
|---|---|
| `OTP_Verification__c.OTP_Code__c` | **Hidden from all profiles** except system/integration user. Flows run in system context and can still read/write it |
| `Lead.Previous_Owner__c` | Read-only for all non-admin profiles |
| `Financial_Asset__c.Is_Stale__c` | Editable only by EBF Document Reviewer and EBF Admin |
| `Lead.Qualification_Score__c` | Read-only for portal users (visible but not editable) |

### Object Permissions Summary

| Object | Loan Officer | Doc Reviewer | Portal User | Agentforce User |
|---|---|---|---|---|
| Lead | Read, Edit | Read | Read (own) | Create, Read, Edit |
| Financial_Asset__c | Read | Read, Edit | Read, Edit (own) | Create, Read, Edit |
| OTP_Verification__c | — | — | — | Create, Read, Edit |
| EBF_Assignment_Counter__c | — | — | — | Read, Edit |
| ContentVersion | Read | Read | Create, Read | Create, Read |

### Salesforce Features to Enable

Enable these in **Setup** before deploying:

| Feature | Setup Path | Purpose |
|---|---|---|
| **Agentforce** | Setup > Einstein > Agentforce | Enable the Agentforce platform |
| **Einstein Generative AI** | Setup > Einstein > Einstein Setup | Required for genAi metadata types |
| **Embedded Service Deployments** | Setup > Embedded Service | Deploy chat widget on Experience Cloud |
| **Digital Experiences** | Setup > Digital Experiences > Settings | Enable Experience Cloud sites |
| **Email Deliverability** | Setup > Email > Deliverability | Set to "All Email" (for OTP delivery) |
| **Allow Flow to send emails** | Setup > Process Automation > Flow Settings | Enable "Let Flows send emails" |
| **Escalation Rules** | Setup > Lead > Escalation Rules | For SLA breach handling |

---

## Repository Structure

```
QualifyAgent/
├── ARCHITECTURE.md                    # Full architecture plan (detailed)
├── README.md                          # This file
│
├── OTP_Verification__c/               # Reusable OTP object (polymorphic)
│   ├── OTP_Verification__c.object-meta.xml
│   └── fields/                        # 11 fields
│
├── Financial_Asset__c/                # Financial data child records
│   ├── Financial_Asset__c.object-meta.xml
│   └── fields/                        # 16 fields (incl. Lead + Account lookups)
│
├── EBF_Assignment_Counter__c/         # Round-robin helper object
│   ├── EBF_Assignment_Counter__c.object-meta.xml
│   └── fields/                        # 1 field
│
├── Lead_Fields/                       # 21 new custom fields on Lead
│   ├── OTP_Verified__c.field-meta.xml
│   ├── Qualification_Status__c.field-meta.xml
│   ├── Qualification_Score__c.field-meta.xml
│   ├── Has_Partner__c.field-meta.xml
│   ├── Total_Assets__c.field-meta.xml
│   ├── UTM_Source__c.field-meta.xml
│   ├── Assignment_Stage__c.field-meta.xml
│   └── ... (21 total)
│
├── OTP_Config__mdt/                   # OTP settings (per-context)
│   ├── OTP_Config__mdt.object-meta.xml
│   └── fields/                        # 6 fields
│
├── EBF_Qualification_Threshold__mdt/  # Scoring thresholds & weights
│   ├── ...object-meta.xml
│   └── fields/                        # 4 fields
│
├── EBF_Asset_Category__mdt/           # Asset category config
│   ├── ...object-meta.xml
│   └── fields/                        # 5 fields
│
├── EBF_Lead_Assignment_Rule__mdt/     # Lead routing rules
│   ├── ...object-meta.xml
│   └── fields/                        # 11 fields
│
├── EBF_Campaign_Mapping__mdt/         # UTM → SF Campaign mapping
│   ├── ...object-meta.xml
│   └── fields/                        # 4 fields
│
├── customMetadata/                    # 31 Custom Metadata records
│   ├── OTP_Config__mdt.Default.md-meta.xml
│   ├── OTP_Config__mdt.Lead_Qualification.md-meta.xml
│   ├── EBF_Qualification_Threshold__mdt.Min_Total_Income.md-meta.xml
│   ├── EBF_Asset_Category__mdt.Payslip.md-meta.xml
│   ├── EBF_Lead_Assignment_Rule__mdt.Premium_Qualified.md-meta.xml
│   ├── EBF_Campaign_Mapping__mdt.EBF_Spring_2026_FB.md-meta.xml
│   └── ... (31 total)
│
├── queues/                            # 7 Lead queues
│   ├── EBF_Triage_Queue.queue-meta.xml
│   ├── EBF_Document_Review_Queue.queue-meta.xml
│   ├── EBF_Senior_Loan_Officers.queue-meta.xml
│   ├── EBF_Loan_Officers.queue-meta.xml
│   ├── EBF_Joint_Application_Officers.queue-meta.xml
│   ├── EBF_Assessment_Review.queue-meta.xml
│   └── EBF_Nurture_Queue.queue-meta.xml
│
├── email/                             # 6 email templates
│   ├── OTP_Email_Templates/
│   │   └── EBF_OTP_Verification.email-meta.xml
│   └── EBF_Email_Templates/
│       ├── EBF_Assignment_Triage.email-meta.xml
│       ├── EBF_Assignment_Doc_Review.email-meta.xml
│       ├── EBF_Assignment_Qualified.email-meta.xml
│       ├── EBF_Assignment_Nurture.email-meta.xml
│       └── EBF_Stale_Notification.email-meta.xml
│
├── genAiFunctions/                    # 8 Agentforce actions
│   ├── Action_Create_Or_Update_Lead.genAiFunction-meta.xml
│   ├── Action_Send_OTP.genAiFunction-meta.xml
│   ├── Action_Verify_OTP.genAiFunction-meta.xml
│   ├── Action_Update_OTP_Status.genAiFunction-meta.xml
│   ├── Action_Create_Financial_Asset.genAiFunction-meta.xml
│   ├── Action_Save_Lead_Details.genAiFunction-meta.xml
│   ├── Action_Request_Document_Upload.genAiFunction-meta.xml
│   └── Action_Run_Qualification.genAiFunction-meta.xml
│
├── genAiPlugins/                      # 3 Agentforce topics
│   ├── Topic_Identity_Collection.genAiPlugin-meta.xml
│   ├── Topic_OTP_Verification.genAiPlugin-meta.xml
│   └── Topic_Financial_Qualification.genAiPlugin-meta.xml
│
└── genAiPlanners/                     # 1 Agent planner
    └── Equity_Builder_Qualification_Agent.genAiPlanner-meta.xml
```

**144 metadata files total.**

---

## Deployment

### 1. Deploy metadata to a scratch org or sandbox

Copy files into a Salesforce DX project structure:

```bash
# From your SFDX project root
cp -R QualifyAgent/OTP_Verification__c force-app/main/default/objects/
cp -R QualifyAgent/Financial_Asset__c force-app/main/default/objects/
cp -R QualifyAgent/EBF_Assignment_Counter__c force-app/main/default/objects/
cp -R QualifyAgent/OTP_Config__mdt force-app/main/default/objects/
cp -R QualifyAgent/EBF_Qualification_Threshold__mdt force-app/main/default/objects/
cp -R QualifyAgent/EBF_Asset_Category__mdt force-app/main/default/objects/
cp -R QualifyAgent/EBF_Lead_Assignment_Rule__mdt force-app/main/default/objects/
cp -R QualifyAgent/EBF_Campaign_Mapping__mdt force-app/main/default/objects/
cp -R QualifyAgent/Lead_Fields/* force-app/main/default/objects/Lead/fields/
cp -R QualifyAgent/customMetadata force-app/main/default/
cp -R QualifyAgent/queues force-app/main/default/
cp -R QualifyAgent/email force-app/main/default/
cp -R QualifyAgent/genAiFunctions force-app/main/default/
cp -R QualifyAgent/genAiPlugins force-app/main/default/
cp -R QualifyAgent/genAiPlanners force-app/main/default/

# Deploy
sf project deploy start --source-dir force-app
```

### 2. Build Flows in Flow Builder

The 21 Flows must be built in Salesforce Flow Builder (visual designer). Flow XML is not included because it is extremely verbose and best authored visually. See [ARCHITECTURE.md](ARCHITECTURE.md) for complete flow specifications including:

- Inputs/outputs for each flow
- Step-by-step logic (Get Records, Decisions, Assignments, Updates)
- Record-Triggered Flow entry criteria
- Sub-flow call patterns

### 3. Configure Experience Cloud

1. Create a Digital Experience site (Customer Service template)
2. Build pages: `/s/apply`, `/s/login`, `/s/my-application`, `/s/financial-asset`, `/s/upload-documents`
3. Embed Agentforce agent on `/s/apply` via Embedded Service component
4. Configure Guest User access for unauthenticated chat

### 4. Set up Meta Ads integration

1. Install Meta Pixel on Experience Cloud site (Head Markup)
2. Configure UTM parameter capture via Embedded Service pre-chat fields
3. Update `EBF_Campaign_Mapping__mdt` records with real Salesforce Campaign IDs

---

## Qualification Scoring

Weighted additive model configured via `EBF_Qualification_Threshold__mdt`:

| Criterion | Default Threshold | Weight | Max Points |
|---|---|---|---|
| Total Income | >= $60,000 | 0.30 | 30 |
| Total Assets | >= $50,000 | 0.25 | 25 |
| Dependents | <= 2 | 0.10 | 10 |
| Savings/Income Ratio | >= 20% | 0.15 | 15 |
| Investment Property | >= 1 | 0.20 | 20 |
| **Pass Score** | **>= 60** | | **100** |

All thresholds and weights are admin-configurable without modifying Flows.

---

## Lead Assignment (3-Stage Handover)

| Stage | Trigger | Routes To | SLA |
|---|---|---|---|
| 1. Triage | OTP verified | Triage Queue | 24h |
| 2. Document Review | All docs uploaded | Document Review Queue | 48h |
| 3a. Premium Qualified | Score 80-100 | Senior Loan Officers (round-robin) | 4h |
| 3b. Standard Qualified | Score 60-79 | Loan Officers (round-robin) | 8h |
| 3c. Joint Application | Score 60+ with partner | Joint App Specialists (round-robin) | 8h |
| 3d. Borderline | Score 50-59 | Assessment Review Queue | 24h |
| 3e. Not Qualified | Score 0-49 | Nurture Queue | 72h |

Routing rules are in `EBF_Lead_Assignment_Rule__mdt` — admins change queue targets, score ranges, and SLAs without touching Flows.

---

## Phase 2 (Planned)

- **WhatsApp channel** — native document upload in chat (no portal redirect needed). Architecture is pre-wired with `Channel__c` fields on OTP_Config__mdt and `Entry_Channel__c` on Lead.
- **Lead conversion handler** — Record-Triggered Flow to reparent Financial_Asset__c records from Lead to Account on conversion.
- **Account summary fields** — Mirror qualification fields on Account post-conversion.

---

## License

This project is provided as-is for Salesforce implementation reference.
