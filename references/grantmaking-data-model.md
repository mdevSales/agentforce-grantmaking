# PSC Grantmaking Data Model

## Core Object Chain

```
FundingOpportunity
  └── FundingRequest (the application)
        ├── GrantApplicationReview (scoring / review records)
        └── FundingAward (the award)
              └── Budget (spend tracking)
```

---

## Object Reference

### FundingOpportunity
The grant program itself — what money is available, for what purpose, to whom.

| Field | Type | Purpose |
|---|---|---|
| Name | Text | Program name |
| Status__c | Picklist | Draft / Active / Closed |
| SubmissionDeadline__c | DateTime | Application cutoff |
| MaxAwardAmount__c | Currency | Maximum grant size |
| EligibilityCriteria__c | Long Text | Human-readable criteria (used when BRE not configured) |
| FundingType__c | Picklist | Grant / Loan / Voucher |
| ProgramDescription__c | Long Text | Agent knowledge grounding source |

### FundingRequest
The constituent's application for a specific FundingOpportunity.

| Field | Type | Purpose |
|---|---|---|
| Name | Text | Auto-generated application ID |
| Status__c | Picklist | Draft / Submitted / Under Review / Approved / Denied / Withdrawn |
| EligibilityStatus__c | Picklist | Eligible / Ineligible / Pending Review |
| FundingOpportunity__c | Lookup | Parent program |
| Applicant__c | Lookup(Contact) | Constituent applying |
| Organization__c | Lookup(Account) | Applying organization (if applicable) |
| RequestedAmount__c | Currency | Amount requested |
| SubmissionDate__c | Date | When submitted |
| ReviewScore__c | Number | Aggregate score from reviews |
| DecisionRationale__c | Long Text | Decision explanation (Decision Explainer populates this) |

### GrantApplicationReview
Individual reviewer scoring records linked to a FundingRequest.

| Field | Type | Purpose |
|---|---|---|
| FundingRequest__c | Lookup | Application being reviewed |
| Reviewer__c | Lookup(User) | Assigned reviewer |
| Score__c | Number | Numeric score |
| ReviewNotes__c | Long Text | Reviewer comments |
| Status__c | Picklist | Assigned / In Progress / Complete |
| ReviewCriteria__c | Long Text | Criteria used for scoring |

### FundingAward
The approved grant award linked to an approved FundingRequest.

| Field | Type | Purpose |
|---|---|---|
| FundingRequest__c | Lookup | Source application |
| AwardAmount__c | Currency | Actual awarded amount |
| AwardDate__c | Date | Date of award |
| AwardStatus__c | Picklist | Active / Closed / Suspended |
| ComplianceStatus__c | Picklist | Compliant / At Risk / Non-Compliant |
| ReportingDueDate__c | Date | Next compliance report due |

### Budget
Tracks spend against a FundingAward.

| Field | Type | Purpose |
|---|---|---|
| FundingAward__c | Lookup | Parent award |
| BudgetCategory__c | Text | Spend category |
| AllocatedAmount__c | Currency | Budget line amount |
| ActualAmount__c | Currency | Actual spend to date |
| Variance__c | Formula | AllocatedAmount - ActualAmount |

---

## Permission Sets

| Permission Set | Purpose | Assigned To |
|---|---|---|
| Grantmaking User | Read/write FundingRequest, FundingOpportunity | Caseworkers, applicants (internal) |
| Grantmaking Manager | Full CRUD + approve/award/manage all objects | Grants managers, program officers |
| Grantmaking External Reviewer | Review-only access to FundingRequest + GrantApplicationReview | External review panel |
| Experience Cloud for Grantmaking | Constituent portal access | Applicants via Experience Cloud |
| Einstein Agent User | Required for any user interacting with Agentforce | All agent users |
| Einstein Agent Access | Required on the agent's integration user | Agent integration user |
| Agentforce for Public Sector | License-level access for Agentforce PS features | All agent users |

---

## Integration Points

### Business Rules Engine (BRE)
- Used for: **Check Eligibility** action
- How it works: BRE rule sets evaluate FundingRequest fields against FundingOpportunity eligibility criteria
- Fallback when not configured: autolaunched Flow queries EligibilityCriteria__c fields directly
- Setup: see [bre-configuration-guide.md](bre-configuration-guide.md)

### DocGen (Document Generation)
- Used for: award letters, denial letters, compliance reports
- Object integration: FundingAward → Award Letter template
- Triggers: Flow element on FundingAward status change to "Active"

### OmniStudio / OmniScript
- Used for: **Guide Application Completion** — the application intake form
- Dependency: OmniStudio license required
- Fallback when not licensed: Flow screen or Experience Cloud LWC
- See [omnistudio-dependency-guide.md](omnistudio-dependency-guide.md)

### Outcome Management
- Used for: **Support Compliance Reporting** action
- Links FundingAward to program outcomes, tracks indicator results

### Decision Explainer
- Used for: legally defensible audit trail on FundingRequest and FundingAward decisions
- Populates DecisionRationale__c with a structured explanation of the approval/denial
- See [decision-explainer-guide.md](decision-explainer-guide.md)

---

## SOQL Quick Reference

```soql
-- Active grant programs accepting applications
SELECT Id, Name, MaxAwardAmount__c, SubmissionDeadline__c
FROM FundingOpportunity
WHERE Status__c = 'Active' AND SubmissionDeadline__c > TODAY

-- Applications for a specific constituent
SELECT Id, Name, Status__c, FundingOpportunity__r.Name, RequestedAmount__c
FROM FundingRequest
WHERE Applicant__c = :contactId

-- Open reviews assigned to current user
SELECT Id, FundingRequest__r.Name, Score__c, Status__c
FROM GrantApplicationReview
WHERE Reviewer__c = :userId AND Status__c != 'Complete'

-- Awards at compliance risk
SELECT Id, FundingRequest__r.Name, AwardAmount__c, ReportingDueDate__c
FROM FundingAward
WHERE ComplianceStatus__c = 'At Risk' AND ReportingDueDate__c <= NEXT_N_DAYS:30

-- Budget utilization for an award
SELECT BudgetCategory__c, AllocatedAmount__c, ActualAmount__c, Variance__c
FROM Budget
WHERE FundingAward__c = :awardId
```
