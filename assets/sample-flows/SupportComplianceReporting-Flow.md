# Flow Design: Support Compliance Reporting

**Type:** Autolaunched Flow
**API Name:** `SupportComplianceReporting`
**Purpose:** Returns compliance status, budget utilization, and upcoming reporting deadlines for active grant awards.

---

## Input Variables

| Variable | Type | Required | Notes |
|---|---|---|---|
| organizationId | Id | Yes* | For org grantees |
| contactId | Id | Yes* | For individual grantees |

*One of organizationId or contactId required.

## Output Variables

| Variable | Type | Notes |
|---|---|---|
| awardSummaryText | Text | Formatted compliance summary |
| atRiskCount | Number | Number of at-risk awards |
| nextReportingDeadline | Date | Nearest upcoming deadline |
| hasActiveAwards | Boolean | Whether any active awards found |

---

## Flow Elements

### Element 1: Get Active FundingAwards
**Type:** Get Records
**Object:** FundingAward
**Filter (OR conditions):**
- Organization__c = {!organizationId} (if not null)
- Applicant__c = {!contactId} (if not null)
- AwardStatus__c = 'Active'
**Fields:** Id, Name, AwardAmount__c, AwardDate__c, ComplianceStatus__c, ReportingDueDate__c, FundingRequest__r.Name, FundingRequest__r.FundingOpportunity__r.Name

### Element 2: Decision — Any active awards?
- No → `hasActiveAwards = false`, `awardSummaryText = 'No active grant awards found for your account'` → End
- Yes → `hasActiveAwards = true` → Element 3

### Element 3: Loop — Process each award
For each FundingAward in `awards`:

**Get Budget records:**
- Get Records: Budget WHERE FundingAward__c = {!currentAward.Id}
- Loop budget records: sum `totalAllocated` += AllocatedAmount__c, sum `totalActual` += ActualAmount__c

**Calculate utilization:**
```
IF totalAllocated > 0:
    utilizationRate = (totalActual / totalAllocated) * 100
ELSE:
    utilizationRate = 0
```

**Determine at-risk status:**
```
atRisk = false
IF utilizationRate < 50 AND ReportingDueDate__c <= TODAY + 30:
    atRisk = true
    atRiskCount += 1
IF ComplianceStatus__c == 'At Risk' OR ComplianceStatus__c == 'Non-Compliant':
    atRisk = true
    atRiskCount += 1
```

**Find nearest deadline:**
```
IF nextReportingDeadline is null OR currentAward.ReportingDueDate__c < nextReportingDeadline:
    nextReportingDeadline = currentAward.ReportingDueDate__c
```

**Build award summary text (append per award):**
```
"• " + currentAward.Name + " (" + FundingOpportunity.Name + ")"
+ "\n  Award: $" + AwardAmount__c
+ "\n  Budget utilization: " + utilizationRate + "%"
+ "\n  Compliance status: " + ComplianceStatus__c
+ "\n  Next report due: " + ReportingDueDate__c
+ IF atRisk: "\n  ⚠ AT RISK — immediate attention needed"
+ "\n"
```

### Element 4: End

---

## Agent Response Pattern

```
"Here's your compliance summary:

{awardSummaryText}

{IF atRiskCount > 0}: ⚠ You have {atRiskCount} award(s) at compliance risk. 
Would you like help preparing your next compliance report?

Your next reporting deadline is: {nextReportingDeadline}"
```

---

## Testing

```apex
Map<String, Object> inputs = new Map<String, Object>{
    'organizationId' => 'YOUR_ACCOUNT_ID'
};
Flow.Interview.SupportComplianceReporting flow =
    new Flow.Interview.SupportComplianceReporting(inputs);
flow.start();
System.debug('Summary: ' + flow.getVariableValue('awardSummaryText'));
System.debug('At Risk Count: ' + flow.getVariableValue('atRiskCount'));
System.debug('Next Deadline: ' + flow.getVariableValue('nextReportingDeadline'));
```
