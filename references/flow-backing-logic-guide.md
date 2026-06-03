# Flow Backing Logic Guide

All `FLOW`-backed actions use autolaunched Flows (no screens, no user interaction — they run in the background when called by the agent).

**Core pattern for all flows:**
1. Get Records (query PSC objects)
2. Decision (branch based on query results)
3. Assignment (build output variables)
4. Return output to agent

---

## General Rules

- Always use **Autolaunched Flow** type — not Screen Flow, not Record-Triggered Flow
- Expose as an **Invocable Action** so the agent can call it
- Input/output variables must be marked `Available for input` / `Available for output`
- Use `Get Records` with SOQL-style filters — not hardcoded IDs
- Handle null / no-records cases with a Decision element — never let a null propagate

---

## Flow 1: Check Grant Eligibility (Flow Fallback)

**For:** Action 1 when BRE is not configured.
Also see [bre-configuration-guide.md](bre-configuration-guide.md) for the BRE path.

```
Flow: CheckGrantEligibility_Flow
Type: Autolaunched

Inputs:
  fundingRequestId (Id, required)

Steps:
  1. Get FundingRequest (with Organization, Contact, FundingOpportunity related fields)
  2. Get FundingOpportunity (MaxAwardAmount, eligible org types, geographies)
  3. Decision: requestedAmount <= maxAwardAmount?
  4. Decision: orgType in eligibleOrgTypes? (text CONTAINS check)
  5. Decision: billingState in eligibilityGeographies? (text CONTAINS check)
  6. Assignment: build eligibilityReason text
  7. Final Assignment:
     - All passed: eligibilityStatus = 'ELIGIBLE'
     - Any failed: eligibilityStatus = 'INELIGIBLE'
     - Ambiguous/missing fields: eligibilityStatus = 'NEEDS_REVIEW'

Outputs:
  eligibilityStatus (Text)
  eligibilityReason (Text)
```

---

## Flow 2: Recommend Grant Programs

```
Flow: RecommendGrantPrograms
Type: Autolaunched

Inputs:
  applicantType (Text) — e.g., 'Individual', 'Small Business', 'Nonprofit'
  requestedAmount (Currency)
  geographyCode (Text) — state or county code

Steps:
  1. Get Records: FundingOpportunity
     WHERE Status = 'Active'
     AND SubmissionDeadline > TODAY
     AND MaxAwardAmount >= requestedAmount (optional filter)
  2. Loop: for each opportunity
     - Check: EligibleOrgTypes CONTAINS applicantType
     - Check: EligibilityGeographies CONTAINS geographyCode
     - If both match: add to recommendedList collection
  3. Sort recommendedList by SubmissionDeadline ASC (prioritize soonest deadline)
  4. Limit to top 5 (Subflow or loop counter)

Outputs:
  recommendedPrograms (Record Collection of FundingOpportunity)
  programCount (Number)
```

---

## Flow 3: Track Application Status

```
Flow: TrackApplicationStatus
Type: Autolaunched

Inputs:
  contactId (Id, required) — the verified constituent
  specificApplicationId (Id, optional) — if user wants a specific app

Steps:
  1. If specificApplicationId is provided:
     - Get FundingRequest by Id
  2. Else:
     - Get Records: FundingRequest
       WHERE Applicant__c = contactId
       ORDER BY SubmissionDate__c DESC
       LIMIT 5
  3. Get related: FundingOpportunity names, award details
  4. Assignment: build statusSummary text per application
     - Format: "[Program Name] — [Status] — Submitted [Date]"
     - If Approved: include award amount
     - If Denied: include DecisionRationale summary (first 200 chars)

Outputs:
  applications (Record Collection of FundingRequest)
  statusSummary (Text) — formatted summary for agent response
  applicationCount (Number)
```

---

## Flow 4: Support Compliance Reporting

```
Flow: SupportComplianceReporting
Type: Autolaunched

Inputs:
  organizationId (Id, required)
  contactId (Id, optional) — for individual grantees

Steps:
  1. Get Records: FundingAward
     WHERE (Organization__c = organizationId OR Applicant__c = contactId)
     AND AwardStatus = 'Active'
  2. Loop: for each award
     - Get Records: Budget WHERE FundingAward__c = awardId
     - Calculate: totalAllocated (sum AllocatedAmount), totalActual (sum ActualAmount)
     - Calculate: utilizationRate = totalActual / totalAllocated * 100
     - Decision: utilizationRate < 50 AND ReportingDueDate <= TODAY + 30 → atRisk = true
     - Add to awardSummaryList
  3. Assignment: build complianceSummary text

Outputs:
  awardSummaryList (Collection)
  complianceSummary (Text)
  atRiskCount (Number)
  nextReportingDeadline (Date)
```

---

## Flow 5: Validate Required Documents

```
Flow: ValidateRequiredDocuments
Type: Autolaunched

Inputs:
  fundingRequestId (Id, required)

Steps:
  1. Get Records: FundingRequest (to get FundingOpportunity)
  2. Get Records: Required Document list
     (Custom metadata type or custom object: GrantRequiredDocument__mdt
      WHERE Program__c = fundingOpportunityId)
  3. Get Records: ContentDocumentLink WHERE LinkedEntityId = fundingRequestId
  4. Loop: for each required document
     - Check: uploaded files contain a match for this document type
     - If missing: add to missingDocumentsList
  5. Decision: missingDocumentsList is empty?

Outputs:
  missingDocuments (Text Collection)
  validationStatus (Text) — 'Complete' or 'Incomplete'
  missingDocumentCount (Number)
```

---

## Exposing a Flow as an Invocable Action for the Agent

All of the above Flows need to be callable by the agent. They already are by default (autolaunched Flows are automatically available as invocable actions in Agent Builder and Agent Script).

**In Agent Builder:** Topic → Actions → New Action → Flow → select your Flow → map inputs/outputs

**In Agent Script:**
```
action CheckEligibility {
  type: Flow
  flowApiName: "CheckGrantEligibility_Flow"
  inputs {
    fundingRequestId: @currentDraftId
  }
  outputs {
    eligibilityStatus -> @eligibilityStatus
    eligibilityReason -> @eligibilityReason
  }
}
```

**Test a Flow before wiring it:**
```bash
sf apex run --file test-flow.apex --target-org YOUR_ALIAS
```

Where `test-flow.apex` contains:
```apex
Map<String, Object> inputs = new Map<String, Object>{
    'fundingRequestId' => 'YOUR_TEST_FUNDING_REQUEST_ID'
};
Flow.Interview.CheckGrantEligibility_Flow flow = new Flow.Interview.CheckGrantEligibility_Flow(inputs);
flow.start();
System.debug('Status: ' + flow.getVariableValue('eligibilityStatus'));
System.debug('Reason: ' + flow.getVariableValue('eligibilityReason'));
```
