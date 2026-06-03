# Flow Design: Notify Applicant

**Type:** Autolaunched Flow (can also be triggered as a Record-Changed Flow on FundingRequest.Status__c)
**API Name:** `NotifyApplicant`
**Purpose:** Sends an outbound notification to an applicant when their FundingRequest status changes.

---

## Two Invocation Patterns

**Pattern A — Agent-initiated:** Agent detects status change and calls this Flow to notify the applicant.
**Pattern B — Record-triggered:** Attach as a Record-Changed Triggered Flow on FundingRequest, triggered when Status__c changes. No agent invocation needed.

Both use the same Flow logic — Pattern B is preferred for production as it fires automatically.

---

## Input Variables

| Variable | Type | Required | Notes |
|---|---|---|---|
| fundingRequestId | Id | Yes | The application whose status changed |
| newStatus | Text | No | Override status (for agent pattern) — if blank, reads from record |

## Output Variables

| Variable | Type | Notes |
|---|---|---|
| notificationSent | Boolean | Whether notification was successfully sent |
| notificationMethod | Text | 'Email' / 'Messaging' / 'None' |

---

## Message Templates by Status

| Status | Subject | Body template |
|---|---|---|
| Submitted | "Your application has been received" | "Your application [Name] for [Program] has been received. Expected review timeline: [X] days. We will notify you when a decision has been made." |
| Under Review | "Your application is under review" | "Your application [Name] is currently being reviewed by our grants team. No action is needed from you at this time." |
| Approved | "Congratulations — Your grant application has been approved" | "Congratulations! Your application [Name] for [Program] has been approved for $[AwardAmount]. A grant specialist will contact you within 5 business days to discuss next steps." |
| Denied | "Update on your grant application" | "We have carefully reviewed your application [Name] for [Program]. We regret to inform you that your application was not approved at this time. [DecisionRationale summary]. You have the right to appeal this decision within 30 days." |
| More Info Needed | "Additional information required for your application" | "To continue processing your application [Name], we need additional information. Please log in to [portal URL] or contact us at [phone]." |
| Withdrawn | "Your application has been withdrawn" | "Your application [Name] has been marked as withdrawn. If this was an error, please contact us at [phone] within 10 days." |

---

## Flow Elements

### Element 1: Get FundingRequest
**Fields:** Name, Status__c, Applicant__c, DecisionRationale__c, RequestedAmount__c
**Related:** FundingOpportunity__r.Name, FundingAward.AwardAmount__c, Applicant__r.Email, Applicant__r.MobilePhone, Applicant__r.FirstName

### Element 2: Assignment — Resolve Status
```
statusToNotify = newStatus (if provided)
IF newStatus is blank:
    statusToNotify = fundingRequest.Status__c
```

### Element 3: Decision — Is notification needed for this status?
**Statuses that trigger notification:** Submitted, Under Review, Approved, Denied, More Info Needed, Withdrawn
- Not in list → Assignment: `notificationSent = false`, `notificationMethod = 'None'` → End

### Element 4: Assignment — Build Message
Based on `statusToNotify`, select and populate the message template (using Assignment elements with multi-line text or a Custom Metadata lookup for templates).

### Element 5: Decision — Messaging licensed?
*(Check via custom metadata or a configuration custom setting)*
- Yes → Element 6 (send Messaging notification)
- No → Element 7 (send email)

### Element 6: Send Messaging Notification
**Action:** Messaging → Send Notification
- To: Applicant's MobilePhone
- Message: {!messageBody}
- Assignment: `notificationMethod = 'Messaging'`

### Element 7: Send Email
**Action:** Send Email
- To: {!fundingRequest.Applicant__r.Email}
- Subject: {!messageSubject}
- Body: {!messageBody}
- Assignment: `notificationMethod = 'Email'`

### Element 8: Assignment
`notificationSent = true`

### Element 9: End

---

## Record-Triggered Flow Setup (Pattern B)

1. In Flow Builder → New → Record-Triggered Automation
2. Object: FundingRequest
3. Trigger: A record is updated
4. Entry condition: Status__c IS CHANGED
5. Optimize for: Actions and related records
6. Add: Call subflow (or inline logic from above)
7. Activate: Save and Activate

For Pattern B, the `fundingRequestId` comes from the triggering record (`{!$Record.Id}`) — no input variable needed.

---

## Testing

```apex
Map<String, Object> inputs = new Map<String, Object>{
    'fundingRequestId' => 'YOUR_FUNDING_REQUEST_ID',
    'newStatus' => 'Approved'
};
Flow.Interview.NotifyApplicant flow =
    new Flow.Interview.NotifyApplicant(inputs);
flow.start();
System.debug('Sent: ' + flow.getVariableValue('notificationSent'));
System.debug('Method: ' + flow.getVariableValue('notificationMethod'));
```
