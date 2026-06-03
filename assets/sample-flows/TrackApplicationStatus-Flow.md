# Flow Design: Track Application Status

**Type:** Autolaunched Flow
**API Name:** `TrackApplicationStatus`
**Auth gate required:** Yes — for service agents. Only call this after identity is verified.

---

## Input Variables

| Variable | Type | Required | Notes |
|---|---|---|---|
| contactId | Id | Yes* | Verified constituent Contact ID |
| specificApplicationId | Id | No | If user asks about a specific application |

*One of contactId or specificApplicationId must be provided.

## Output Variables

| Variable | Type | Notes |
|---|---|---|
| applications | Record Collection | FundingRequest records |
| statusSummary | Text | Formatted summary string for agent response |
| applicationCount | Number | Total applications found |

---

## Flow Elements

### Element 1: Decision — Specific application requested?
**Condition:** `{!specificApplicationId}` is not null
- Yes → Element 2 (get specific app)
- No → Element 3 (get all apps for contact)

### Element 2: Get Specific FundingRequest
**Type:** Get Records
**Object:** FundingRequest
**Filter:** Id = {!specificApplicationId}
**Fields:** All status fields + related FundingOpportunity.Name, FundingAward.AwardAmount__c

### Element 3: Get All FundingRequests for Contact
**Type:** Get Records
**Object:** FundingRequest
**Filter:** Applicant__c = {!contactId}
**Sort:** SubmissionDate__c Descending
**Limit:** 5
**Fields:** Name, Status__c, SubmissionDate__c, FundingOpportunity__r.Name, RequestedAmount__c, DecisionRationale__c

### Element 4: Decision — Any applications found?
- No records → Assignment: `statusSummary = 'No applications found for your account'`, `applicationCount = 0` → End

### Element 5: Loop — Build Status Summary
**Loop over:** applications collection
**For each application:**
- Build `singleAppStatus` text:
  - Base: "[FundingOpportunity.Name] — Status: [Status__c] — Submitted: [SubmissionDate__c]"
  - If Status = 'Approved': append " — Award: $[FundingAward.AwardAmount__c]"
  - If Status = 'Denied': append " — Reason: [first 150 chars of DecisionRationale__c]"
  - If Status = 'Under Review': append " — Estimated decision: [ReviewCompletionDate__c if present]"
- Append `singleAppStatus` + newline to `statusSummary`

### Element 6: Assignment — Set applicationCount
`applicationCount = COUNT({!applications})`

### Element 7: End

---

## Testing

```apex
Map<String, Object> inputs = new Map<String, Object>{
    'contactId' => 'YOUR_CONTACT_ID'
};
Flow.Interview.TrackApplicationStatus flow =
    new Flow.Interview.TrackApplicationStatus(inputs);
flow.start();
System.debug('Count: ' + flow.getVariableValue('applicationCount'));
System.debug('Summary: ' + flow.getVariableValue('statusSummary'));
```
