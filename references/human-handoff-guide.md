# Human Handoff / Escalation Guide

## When to Escalate

The agent should escalate to a human caseworker when it encounters:
- Complex eligibility edge cases the BRE rules don't cover
- Appeals or contested eligibility decisions
- Fraud flags (duplicate applications from different organizations, suspicious document patterns)
- Constituent requests to "speak to someone" or "escalate my application"
- Any situation where the agent cannot confidently answer and the stakes are high

## Pattern: Async Case Creation + Queue Routing

The escalation pattern creates a Case, routes it to the Grant Review Queue, and notifies a caseworker. The constituent receives a confirmation message. There is no live agent transfer — this is async escalation.

### Flow Design: EscalateToReviewQueue

See [../assets/sample-flows/EscalateToReviewQueue-Flow.md](../assets/sample-flows/EscalateToReviewQueue-Flow.md) for the full Flow design.

**Flow inputs:**
- `contactId` (Id) — the constituent
- `fundingRequestId` (Id) — the application being escalated (if known)
- `escalationReason` (Text) — what triggered the escalation
- `conversationSummary` (Text) — brief summary of what the agent discussed

**Flow logic:**
1. Get Related: Contact record (for name, email)
2. Get Related: FundingRequest record (if provided)
3. Create Record: Case
   - Subject: "Grant Review Escalation — [FundingRequest.Name or 'New Inquiry']"
   - Type: "Grant Review Escalation"
   - Status: "New"
   - Origin: "Agentforce"
   - ContactId: contactId
   - Description: escalationReason + "\n\nConversation summary:\n" + conversationSummary
4. Get Record: Grant Review Queue (Queue.DeveloperName = 'Grant_Review_Queue')
5. Create Record: CaseTeamMember (assign Case to queue) OR set Case.OwnerId = queueId
6. Create Record: Task (follow-up task for queue)
7. Send Email / Messaging Notification to caseworker queue (if Messaging licensed)
8. Output: `caseNumber` (Text), `caseId` (Id)

**Agent response after escalation:**
> "I've created a case for you (Case #[caseNumber]). A grant specialist from our team will review your situation and contact you within 2 business days. You can reference this case number when following up."

---

## NGA Agent Script: Escalation Wiring

```
action EscalateToReviewer {
  type: Flow
  flowApiName: "EscalateToReviewQueue"
  inputs {
    contactId: @applicantId
    fundingRequestId: @currentDraftId
    escalationReason: {userInput.escalationContext}
    conversationSummary: @conversationSummary
  }
  outputs {
    caseNumber -> @escalationCaseNumber
  }
}

when @escalationCaseNumber != "" {
  response "I've created case #{@escalationCaseNumber} for you. A grant specialist will contact you within 2 business days."
  end
}
```

**Escalation trigger patterns in Agent Script:**
```
// User explicitly requests human
when {userInput} matches ["speak to someone", "talk to a person", "escalate", "this isn't right", "I want to appeal"] {
  action EscalateToReviewer { ... }
}

// Agent detects complexity it can't handle
when @eligibilityStatus == "NEEDS_REVIEW" {
  action EscalateToReviewer { ... }
}

// Fraud flag from duplicate check
when @duplicateFound == "true" {
  action EscalateToReviewer {
    inputs { escalationReason: "Potential duplicate application detected" }
  }
}
```

---

## Legacy Agent Builder: Escalation Wiring

1. In the Bot Dialog where escalation triggers:
   - Add a Flow element → `EscalateToReviewQueue`
   - Map input `contactId` ← conversation variable `applicantId`
   - Map input `fundingRequestId` ← conversation variable `currentDraftId`
   - Map output `caseNumber` → conversation variable `escalationCaseNumber`

2. After the Flow element, add a Text message:
   > "I've created case #{!$Bot.escalationCaseNumber} for you. A grant specialist will contact you within 2 business days."

3. Add an "End Bot Conversation" element after the message.

4. To trigger escalation from a topic, add a "Route to Agent" or "Redirect to Flow" element with the escalation condition in the dialog's Decision node.

---

## Grant Review Queue Setup

Before the escalation Flow works, the queue must exist:

1. Setup → Queues → New
2. Label: "Grant Review Queue"
3. Queue Name: `Grant_Review_Queue`
4. Object: Case
5. Add queue members: caseworkers who handle grant escalations
6. Email: set queue email address for notifications

---

## Decision Explainer Integration

If Decision Explainer is enabled on the Case object, the escalation reason and conversation summary are automatically logged in the audit trail. See [decision-explainer-guide.md](decision-explainer-guide.md).
