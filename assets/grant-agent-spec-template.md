# Agent Spec: [AGENT NAME]
*Fill in brackets. Delete sections that don't apply. Get explicit approval before building.*

---

## Overview

| Field | Value |
|---|---|
| Agent name | [e.g., "Grant Navigator"] |
| Agent type | [Employee-Facing / Workflow/Automation / Service/Public-Facing] |
| Deployment channel | [Salesforce Console / Experience Cloud / Both] |
| NGA or Legacy | [NGA Agent Script / Legacy Agent Builder] |
| Gov Cloud | [Yes / No] |
| PSC Grantmaking installed | [Yes / No] |
| BRE configured | [Yes — rule set: _______ / No — using Flow fallback] |
| OmniStudio licensed | [Yes / No — using: Flow screen / LWC] |
| Data 360 in scope | [Yes / No] |

---

## Subagent Graph

```
[AgentName]
  ├── [Subagent 1 name] — [one-sentence purpose]
  ├── [Subagent 2 name] — [one-sentence purpose]
  ├── [Subagent 3 name] — [one-sentence purpose]
  └── [Subagent 4 name — EscalationSubagent if escalation is in scope]
```

*Standard subagent sets by agent type:*

**Employee-Facing:**
```
GrantsManagerAgent
  ├── EligibilitySubagent
  ├── ApplicationReviewSubagent (Summarize + Score Prompt Templates)
  ├── AwardManagementSubagent
  └── EscalationSubagent
```

**Service/Public-Facing:**
```
PublicGrantsAgent
  ├── IdentityVerificationSubagent (REQUIRED FIRST)
  ├── EligibilitySubagent
  ├── ApplicationSubagent (multi-turn stateful)
  ├── StatusSubagent (gated by @verifiedIdentity)
  └── FAQSubagent (ADL grounding)
```

---

## State Management Section

*Fill in if any stateful action (e.g., Guide Application Completion) is in scope. Delete if not.*

**NGA @variables:**
```
@applicationStep: String = "start"
@completedSections: String = ""
@applicantId: String = ""
@currentDraftId: String = ""
@verifiedIdentity: String = "false"    ← for service agents
@verifiedContactId: String = ""         ← for service agents
```

**Legacy Conversation Variables:**
| Variable | Type | Default | Used by |
|---|---|---|---|
| applicationStep | Text | start | ApplicationSubagent |
| completedSections | Text | (empty) | ApplicationSubagent |
| applicantId | Text | (empty) | All authenticated topics |
| currentDraftId | Text | (empty) | ApplicationSubagent |

---

## Authentication Gate Section

*Fill in if this is a service/public-facing agent. Delete if employee-facing.*

**Verification method:** [Email + DOB / Email + Last 4 SSN / Community login session]

**NGA guard:**
```
when @verifiedIdentity != "true" {
  route to IdentityVerificationSubagent
}
```

**Protected actions (require auth):**
- [ ] Track Application Status
- [ ] Guide Application Completion (after contact creation)
- [ ] Support Compliance Reporting
- [ ] [Add others]

**Public actions (no auth needed):**
- [ ] Check Eligibility
- [ ] Recommend Grant Programs
- [ ] Answer Program Questions (FAQ)

---

## Action Set

*For each action: name, backing logic type, Flow/Template API name, inputs, outputs.*

### [Subagent 1 Name]

| Action | Type | Implementation | Inputs | Outputs |
|---|---|---|---|---|
| Check Eligibility | STANDARD (BRE) / FLOW | `CheckGrantEligibility_BRE` / `CheckGrantEligibility_Flow` | fundingRequestId | eligibilityStatus, eligibilityReason |
| Recommend Programs | FLOW | `RecommendGrantPrograms` | applicantType, geographyCode | recommendedPrograms |

### [Subagent 2 Name — if ApplicationReviewSubagent]

| Action | Type | Implementation | Inputs | Outputs |
|---|---|---|---|---|
| Summarize Application | PROMPT_TEMPLATE | `Grantmaking_Summarize_Application` | Application (FundingRequest__c SObject) | applicationSummary |
| Score Application | PROMPT_TEMPLATE | `Grantmaking_Score_Application` | Application (FundingRequest__c SObject), ScoringCriteria | scoringOutput |
| Track Review Status | FLOW | `TrackReviewAssignments` | fundingRequestId | reviewers, scores, completionStatus |

### [EscalationSubagent — if escalation is in scope]

| Action | Type | Implementation | Inputs | Outputs |
|---|---|---|---|---|
| Escalate to Review Queue | FLOW | `EscalateToReviewQueue` | contactId, fundingRequestId, escalationReason | caseNumber |

---

## Knowledge Grounding Section

*Fill in if Answer Program Questions (ADL) is in scope. Delete if not.*

**ADL name:** [e.g., "Grant Programs Knowledge Base"]
**Data source type:** [PDF documents / Knowledge Articles]
**Documents to index:**
- [ ] [Program guide PDF name]
- [ ] [Eligibility policy PDF name]
- [ ] [FAQ document name]

**Retriever configuration:**
- Chunk size: [512 / 1024] tokens
- Overlap: [50] tokens
- Top-K: [3 / 5]

**Indexing timeline:** Start indexing by [date] — takes [X] minutes.

---

## Web Search Section

*Fill in if web search action is in scope. Delete if not.*

- [ ] Search scope: [All web / .gov domains only / Other]
- [ ] Max results: [3 / 5]
- [ ] Safe search: [Enabled]

---

## Data 360 Section

*Fill in if D360 is in scope. Delete if not.*

**D360 feature | Status:**
- [ ] Activation target configured (`sf-datacloud-act`)
- [ ] Salesforce org connector set up (`sf-datacloud-connect`)
- [ ] DMO mapping completed (`sf-datacloud-harmonize`):
  - FundingRequest → `ssot__Case__dlm`
  - FundingAward → `ssot__Benefit__dlm`
  - Contact → `ssot__Individual__dlm`
- [ ] Identity Resolution ruleset configured
- [ ] Calculated Insights in scope:
  - [ ] `LifetimeGrantsByHousehold__cio`
  - [ ] `ComplianceRateByOrg__cio`
  - [ ] `VulnerabilityIndex__cio`

---

## Decision Explainer Section

*Fill in if compliance mode is active. Delete if not.*

- [ ] Decision Explainer enabled on FundingRequest
- [ ] Decision Explainer enabled on FundingAward
- [ ] Explanation templates configured for: Approved / Denied / More Info Needed
- [ ] DecisionRationale__c surfaced in agent response when applicant asks "why was my application denied?"

---

## Permission Sets Checklist

| Permission Set | Assigned to |
|---|---|
| Grantmaking User | Caseworkers: [list users / user groups] |
| Grantmaking Manager | Program officers: [list users] |
| Einstein Agent User | All users who see the agent |
| Einstein Agent Access | Integration user: [username] |
| Agentforce for Public Sector | All users + integration user |
| Experience Cloud for Grantmaking | Constituent portal users (if service agent) |

---

## SI Partner

*Fill in if SI is involved.*
- **SI:** [Agile Cloud / Huron / Slalom / ImagineCRM / Other]
- **Contact:** [name + email]
- **Scope:** [Flows / BRE / OmniStudio / Full build / Support only]

---

## Approval

**Agent Spec reviewed by:** [partner name]
**Approved:** [ ] Yes
**Date:** ___________

*Do not proceed to Phase 4 (Build) without explicit approval.*
