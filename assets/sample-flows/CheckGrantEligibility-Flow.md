# Flow Design: Check Grant Eligibility (Flow Fallback)

**Use when:** BRE is not configured. If BRE is configured, use the BRE invocable action instead.
**Type:** Autolaunched Flow
**API Name:** `CheckGrantEligibility_Flow`

---

## Input Variables

| Variable | Type | Required | Notes |
|---|---|---|---|
| fundingRequestId | Id | Yes | The FundingRequest being evaluated |

## Output Variables

| Variable | Type | Notes |
|---|---|---|
| eligibilityStatus | Text | ELIGIBLE / INELIGIBLE / NEEDS_REVIEW |
| eligibilityReason | Text | Human-readable explanation |

---

## Flow Elements

### Element 1: Get FundingRequest
**Type:** Get Records
**Object:** FundingRequest
**Filter:** Id = {!fundingRequestId}
**Get:** Id, RequestedAmount__c, FundingOpportunity__c, Applicant__c, Organization__c
**Get related fields:**
- `Organization__r.Type` (org type)
- `Organization__r.BillingState` (geography)
- `Organization__r.AnnualRevenue` (for revenue cap checks)
- `FundingOpportunity__r.MaxAwardAmount__c`
- `FundingOpportunity__r.EligibilityGeographies__c` (comma-delimited text)
- `FundingOpportunity__r.EligibleOrgTypes__c` (comma-delimited text)
- `FundingOpportunity__r.MaxAnnualRevenue__c`

### Element 2: Decision — FundingRequest found?
**Condition:** fundingRequest is null
- Yes → Assignment: `eligibilityStatus = 'NEEDS_REVIEW'`, `eligibilityReason = 'Application record not found'` → End

### Element 3: Decision — Amount Check
**Condition:** `{!fundingRequest.RequestedAmount__c}` <= `{!fundingRequest.FundingOpportunity__r.MaxAwardAmount__c}`
- No → Assignment: `eligibilityStatus = 'INELIGIBLE'`, append to `eligibilityReason`: "Requested amount exceeds program maximum of $[MaxAwardAmount]"
- Yes → continue to Element 4

### Element 4: Decision — Organization Type Check
**Condition:** `CONTAINS({!fundingRequest.FundingOpportunity__r.EligibleOrgTypes__c}, {!fundingRequest.Organization__r.Type})`
- No (and EligibleOrgTypes is not empty) → append to `eligibilityReason`: "Organization type [Type] is not eligible for this program"
- Yes or empty field → continue to Element 5

### Element 5: Decision — Geography Check
**Condition:** `CONTAINS({!fundingRequest.FundingOpportunity__r.EligibilityGeographies__c}, {!fundingRequest.Organization__r.BillingState})`
- No (and EligibilityGeographies is not empty) → append to `eligibilityReason`: "Organization location [State] is outside the eligible geography for this program"
- Yes or empty field → continue to Element 6

### Element 6: Final Decision
**Condition:** `{!eligibilityReason}` is empty (all checks passed)
- Yes → Assignment: `eligibilityStatus = 'ELIGIBLE'`, `eligibilityReason = 'All eligibility criteria met'`
- No → Assignment: `eligibilityStatus = 'INELIGIBLE'`

### Element 7: End

---

## Testing

```apex
Map<String, Object> inputs = new Map<String, Object>{
    'fundingRequestId' => 'YOUR_FUNDING_REQUEST_ID'
};
Flow.Interview.CheckGrantEligibility_Flow flow =
    new Flow.Interview.CheckGrantEligibility_Flow(inputs);
flow.start();
System.debug('Status: ' + flow.getVariableValue('eligibilityStatus'));
System.debug('Reason: ' + flow.getVariableValue('eligibilityReason'));
```

---

## Notes
- `EligibilityGeographies__c` and `EligibleOrgTypes__c` use CONTAINS — requires comma-delimited values with no spaces (e.g., "CA,OR,WA" not "CA, OR, WA")
- If these fields are blank/null, the check is skipped (universal eligibility for that criterion)
- For BRE path, see [../../references/bre-configuration-guide.md](../../references/bre-configuration-guide.md)
