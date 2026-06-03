# Decision Explainer Guide

PSC includes built-in Decision Explainer functionality that provides a legally defensible, structured audit trail for grant approval and denial decisions. This is especially critical for compliance-heavy customers and any agency that may face appeals or legal challenges on grant decisions.

---

## What Decision Explainer Does

Decision Explainer records:
- **What decision was made** (approved, denied, more info needed)
- **Why it was made** (which criteria were evaluated, what values were assessed, what the outcome was for each)
- **When it was made** (timestamp)
- **Who made it** (the user or automated process)
- **What data was used** (record snapshot at time of decision)

This creates a tamper-resistant, structured audit trail that can be surfaced in reports, shared with applicants, and used in appeals processes.

---

## Enabling Decision Explainer

### On FundingRequest
1. Setup → Object Manager → FundingRequest → Fields → look for `DecisionRationale__c` or `DecisionExplanation__c`
2. If not present: Setup → Decision Explainer → Enable on FundingRequest
3. Configure which fields are included in the explanation snapshot
4. Configure explanation templates for each decision type (Approved / Denied / More Info Needed)

### On FundingAward
1. Setup → Object Manager → FundingAward → Decision Explainer settings
2. Enable explanation capture for AwardStatus changes
3. Configure the fields that feed the explanation

---

## Agent Integration

### Surfacing Decision Explanation to Applicants
When a constituent asks "why was my application denied?", the agent queries `DecisionRationale__c` from the FundingRequest and returns the explanation in plain language.

**Flow for retrieving explanation:**
```
1. Get Records: FundingRequest WHERE Applicant__c = :contactId AND Status__c IN ('Approved','Denied')
2. Return: Status__c, DecisionRationale__c, DecisionDate__c
```

**Agent response pattern:**
> "Your application for [Program Name] was [Status] on [Date]. Here's the explanation: [DecisionRationale]"

### Populating Decision Explanation During Review (Employee Agent)
When a reviewer uses the Score Application (Action 9) or makes a decision, the agent can trigger a Flow that writes the AI-generated rationale to `DecisionRationale__c`:

```
1. Prompt Template scores the application and returns per-criterion rationale
2. Agent triggers a "SaveDecisionRationale" Flow
3. Flow: Update FundingRequest.DecisionRationale__c = [PT output]
4. Decision Explainer captures the full record snapshot at that moment
```

---

## Recommended Configuration for Compliance-Heavy Customers

| Setting | Value | Reason |
|---|---|---|
| Enable snapshot capture | Yes | Freeze record state at time of decision |
| Include related records in snapshot | Yes (FundingOpportunity, GrantApplicationReview) | Full context captured |
| Configure explanation templates | Yes | Consistent, legally defensible language |
| Enable audit log | Yes | Immutable record of all decision changes |
| Grant applicant read access to DecisionRationale__c | Yes (via Experience Cloud) | Transparency |

---

## Appeals Process Integration

If the customer has an appeals process:
1. When an applicant says "I want to appeal my denial", route to the EscalationSubagent
2. The escalation Case is created with DecisionRationale__c included in the case description
3. The appeals team reviewer can see the original decision explanation alongside the constituent's appeal
4. If the appeals decision differs from the original, Decision Explainer captures both decisions

---

## Legal Considerations

Decision Explainer provides the technical foundation for defensible decisions, but the explanation templates should be reviewed by the agency's legal team. Ensure:
- Explanations don't inadvertently reveal protected classification criteria
- Language is clear to a non-technical audience
- Explanation includes a reference to the applicable program requirements
- Denials include information about the appeals process

Note this to the partner: "Decision Explainer provides the audit trail, but the explanation templates should be reviewed by your client's legal/compliance team before going live."
