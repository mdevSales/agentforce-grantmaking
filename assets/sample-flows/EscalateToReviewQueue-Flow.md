# Flow Design: Escalate to Review Queue

**Type:** Autolaunched Flow
**API Name:** `EscalateToReviewQueue`
**Purpose:** Creates a Case and routes to the Grant Review Queue when the agent encounters an issue it can't resolve (complex eligibility, appeal, fraud flag, user request).

---

## Prerequisites

Before this Flow works:
1. The Grant Review Queue must exist: Setup → Queues → Grant_Review_Queue
2. The queue must be configured for the Case object
3. Queue members (caseworkers) must be added

---

## Input Variables

| Variable | Type | Required | Notes |
|---|---|---|---|
| contactId | Id | Yes | The constituent |
| fundingRequestId | Id | No | The application being escalated (if known) |
| escalationReason | Text | Yes | What triggered the escalation (from agent) |
| conversationSummary | Text | No | Brief summary of agent conversation so far |

## Output Variables

| Variable | Type | Notes |
|---|---|---|
| caseNumber | Text | The created Case number (e.g., "00001234") |
| caseId | Id | Salesforce Id of the created Case |

---

## Flow Elements

### Element 1: Get Contact
**Type:** Get Records
**Object:** Contact
**Filter:** Id = {!contactId}
**Fields:** Id, FirstName, LastName, Email, MobilePhone

### Element 2: Decision — FundingRequest provided?
- Yes → Element 3 (get application name)
- No → continue with blank applicationName

### Element 3: Get FundingRequest Name
**Type:** Get Records
**Object:** FundingRequest
**Filter:** Id = {!fundingRequestId}
**Fields:** Name, Status__c, FundingOpportunity__r.Name

### Element 4: Assignment — Build Case Subject
```
caseSubject = 'Grant Review Escalation'
IF fundingRequest is not null:
    caseSubject = 'Grant Review Escalation — ' + fundingRequest.Name
```

### Element 5: Assignment — Build Case Description
```
caseDescription = 'Escalation reason: ' + escalationReason
+ '\n\nConversation summary:\n' + conversationSummary
+ '\n\nApplicant: ' + contact.FirstName + ' ' + contact.LastName
+ '\nEmail: ' + contact.Email
IF fundingRequest is not null:
    caseDescription += '\nApplication: ' + fundingRequest.Name
    caseDescription += '\nProgram: ' + fundingRequest.FundingOpportunity__r.Name
    caseDescription += '\nApplication Status: ' + fundingRequest.Status__c
```

### Element 6: Get Grant Review Queue
**Type:** Get Records
**Object:** Group (the Queue object)
**Filter:** Type = 'Queue' AND DeveloperName = 'Grant_Review_Queue'
**Fields:** Id

### Element 7: Create Case Record
**Type:** Create Records
**Object:** Case
**Field values:**
- Subject: {!caseSubject}
- Type: 'Grant Review Escalation'
- Status: 'New'
- Origin: 'Agentforce'
- ContactId: {!contactId}
- Description: {!caseDescription}
- OwnerId: {!grantReviewQueue.Id}
- Priority: 'Normal' (or 'High' if fraud flag)

### Element 8: Assignment — Extract Case Number
Get the created Case record:
```
Get Records: Case WHERE Id = {!newCaseId}
caseNumber = case.CaseNumber
```

*(Note: CaseNumber is auto-generated and only readable after creation — requires a second Get Records.)*

### Element 9: Optional — Send Messaging Notification
*If Messaging is licensed:*
- Create Messaging Notification to the Grant Review Queue email
- Subject: "New Escalation Case — {!caseNumber}"
- Body: brief summary of the escalation

*If email only:*
- Send Email action to the queue email address

### Element 10: End — Return outputs
- `caseNumber` = case.CaseNumber
- `caseId` = case.Id

---

## Agent Response After This Flow

```
"I've created case #{caseNumber} for you. A grant specialist from our team 
will review your situation and contact you at {contact.Email} within 2 business days. 
Please keep this case number for your records: {caseNumber}"
```

---

## Testing

```apex
Map<String, Object> inputs = new Map<String, Object>{
    'contactId' => 'YOUR_CONTACT_ID',
    'escalationReason' => 'Test escalation from Apex',
    'conversationSummary' => 'Constituent asked about eligibility, edge case detected'
};
Flow.Interview.EscalateToReviewQueue flow =
    new Flow.Interview.EscalateToReviewQueue(inputs);
flow.start();
System.debug('Case Number: ' + flow.getVariableValue('caseNumber'));
System.debug('Case Id: ' + flow.getVariableValue('caseId'));
```
