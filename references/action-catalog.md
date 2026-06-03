# Action Catalog — 11 Grantmaking Actions

## Backing Logic Types

| Tag | Meaning |
|---|---|
| `STANDARD` | PSC out-of-the-box component (BRE, DocGen, OmniScript) — no custom code needed |
| `FLOW` | Autolaunched Flow — preferred for most data operations |
| `PROMPT_TEMPLATE` | LLM-powered via GenAiPromptTemplate flex type — for generate/summarize/score |
| `APEX` | Only when Flow genuinely cannot express the logic (e.g., external callout with complex auth) |

---

## Action 1: Check Eligibility

**Backing:** `STANDARD` (BRE) | `FLOW` fallback if BRE not configured

**What it does:** Evaluates whether a constituent or organization meets the criteria for a specific FundingOpportunity.

**BRE path:**
- BRE rule set evaluates FundingRequest fields against configured eligibility rules
- Invoked as an invocable action: `BusinessRulesEngine.executeRules(ruleSetApiName, inputRecord)`
- Returns: `ELIGIBLE`, `INELIGIBLE`, `NEEDS_REVIEW`, with an explanation array

**Flow fallback path (BRE not configured):**
- Get Records: FundingOpportunity (eligibility criteria fields)
- Decision element: evaluate requestedAmount, applicantType, geography, organizationType
- Output variable: `eligibilityStatus` (Text), `eligibilityReason` (Text)

**Agent input:** `fundingOpportunityId`, `applicantId`, `requestedAmount`
**Agent output:** eligibility status + reason for decision

See [flow-backing-logic-guide.md](flow-backing-logic-guide.md) for full Flow design.
See [bre-configuration-guide.md](bre-configuration-guide.md) for BRE rule set setup.

---

## Action 2: Recommend Grant Programs

**Backing:** `FLOW`

**What it does:** Queries active FundingOpportunity records and returns a ranked list of programs the constituent is likely eligible for.

**Flow design:**
- Get Records: FundingOpportunity WHERE Status = 'Active' AND SubmissionDeadline > TODAY
- Filter: match applicant type, geography, funding range to opportunity criteria
- Loop: build recommendation list with Name, Description, MaxAwardAmount, Deadline
- Output variable: `recommendedPrograms` (Collection of FundingOpportunity)

**Agent input:** `applicantType`, `organizationType`, `geographyCode`, `requestedAmountRange`
**Agent output:** list of 3-5 matching programs with brief descriptions

---

## Action 3: Guide Application Completion

**Backing:** `FLOW` (multi-turn stateful)

**What it does:** Walks a constituent through completing a FundingRequest step by step. Persists progress across turns.

**State management:** Required — see [multi-turn-state-guide.md](multi-turn-state-guide.md).
- NGA: `@applicationStep` (current step name), `@completedSections` (completed section IDs), `@applicantId`
- Legacy: Conversation Variables set via Flow output at each step completion

**Flow design:** See [sample-flows/TrackApplicationStatus-Flow.md](../assets/sample-flows/TrackApplicationStatus-Flow.md) for pattern.

**Steps:**
1. Personal/organization info → create/update Contact or Account
2. Program narrative → update FundingRequest narrative fields
3. Required documents → checklist validation (see Action 5)
4. Review and submit → set Status to 'Submitted', stamp SubmissionDate

**Agent input (per turn):** current `@applicationStep`, user-provided field values
**Agent output:** confirmation of saved data, next step instruction, progress summary

---

## Action 4: Answer Program Questions

**Backing:** ADL Knowledge Grounding (not a Flow or Apex action)

**What it does:** Answers constituent questions about grant program requirements, eligibility rules, deadlines, and policy.

**ADL setup:**
- **PDF program guides / policy docs:** use retriever-based indexing. Guide: chunk size 512 tokens, overlap 50 tokens, use `text-embedding-ada-002`. Delegate to `developing-agentforce` skill.
- **Knowledge Articles:** add Knowledge Article data source in ADL configuration. Different ingestion path — does not use retriever.

**Agent instruction:** Ground the answer in the indexed content. If the question isn't answerable from the knowledge base, say so and offer to connect to a caseworker.

**Note:** This action does NOT query Salesforce records — it answers from documents. For status queries, use Action 6.

---

## Action 5: Validate Required Documents

**Backing:** `FLOW`

**What it does:** Checks whether all required documents have been uploaded to a FundingRequest.

**Flow design:**
- Get Records: ContentDocumentLink WHERE LinkedEntityId = fundingRequestId
- Get Records: Document checklist for the FundingOpportunity (custom metadata or custom object)
- Loop: compare uploaded documents against required checklist
- Decision: all required docs present?
- Output: `missingDocuments` (List), `validationStatus` (Complete / Incomplete)

**Agent input:** `fundingRequestId`
**Agent output:** list of missing documents (if any) or confirmation all required docs are present

---

## Action 6: Track Application Status

**Backing:** `FLOW`

**What it does:** Returns the current status of a constituent's FundingRequest.

**IMPORTANT:** For service agents, this action requires the Verification Gate before it can run. See [verification-gate-guide.md](verification-gate-guide.md).

**Flow design:**
- Get Records: FundingRequest WHERE Applicant__c = :contactId (or Organization__c = :accountId)
- Return: Status__c, SubmissionDate__c, ReviewScore__c, FundingOpportunity__r.Name
- If multiple applications: return list sorted by SubmissionDate DESC

**Agent input:** `contactId` (or `applicationId` if known)
**Agent output:** status, submission date, program name, next steps based on status

---

## Action 7: Support Compliance Reporting

**Backing:** `FLOW`

**What it does:** Retrieves compliance status, budget utilization, and upcoming reporting deadlines for an active FundingAward.

**Flow design:**
- Get Records: FundingAward WHERE Applicant/Org matches AND AwardStatus = 'Active'
- Get Records: Budget WHERE FundingAward__c = awardId
- Calculate: sum(ActualAmount) / sum(AllocatedAmount) = utilization rate
- Subflow or Decision: flag at-risk if utilization < 50% and ReportingDueDate within 30 days
- Output: award summary, budget utilization %, at-risk flag, next reporting deadline

**Agent input:** `contactId` or `organizationId`
**Agent output:** compliance status dashboard — award name, utilization %, deadline, at-risk flag

---

## Action 8: Summarize Application for Reviewer (NEW)

**Backing:** `PROMPT_TEMPLATE`

**What it does:** Generates a structured 3-5 sentence summary of a FundingRequest for a reviewer, highlighting key details: applicant, program, requested amount, stated purpose, and any flags.

**Template design:**
- Input: `FundingRequest__c` SObject
- Prompt: "Summarize this grant application for a reviewer. Include: applicant name, organization, program applied for, amount requested, stated purpose (from narrative fields), any eligibility flags or missing documents. Be factual and concise."
- Model: `sfdc_ai__DefaultVertexAIGemini25Flash001` (fast, cost-effective)

**Delegate to `salesforce-prompt-templates` skill** for full XML metadata authoring.
See [prompt-template-actions.md](prompt-template-actions.md) for complete XML pattern.

**Agent input:** `fundingRequestId`
**Agent output:** structured summary text (3-5 sentences)

---

## Action 9: Score Application Against Criteria (NEW)

**Backing:** `PROMPT_TEMPLATE`

**What it does:** Evaluates a FundingRequest against the scoring criteria defined for a FundingOpportunity and returns a structured score with rationale.

**Template design:**
- Inputs: `FundingRequest__c` SObject + scoring criteria text (from FundingOpportunity or custom metadata)
- Prompt: "Score this grant application against the following criteria: [criteria]. For each criterion, provide a score (1-5) and a one-sentence rationale. Return as a structured list."
- Model: `sfdc_ai__DefaultBedrockAnthropicClaude45Sonnet` (better for structured reasoning + scoring)

**Important:** This is AI-assisted scoring, not binding. Final scores are reviewed and confirmed by a human reviewer. Note this in the agent's response.

**Delegate to `salesforce-prompt-templates` skill** for full XML metadata.
See [prompt-template-actions.md](prompt-template-actions.md) for complete XML pattern.

**Agent input:** `fundingRequestId`, `scoringCriteriaText`
**Agent output:** per-criterion scores with rationale, suggested total score

---

## Action 10: Notify Applicant (NEW)

**Backing:** `FLOW`

**What it does:** Sends an outbound notification to an applicant when their FundingRequest status changes.

**Flow design:** See [sample-flows/NotifyApplicant-Flow.md](../assets/sample-flows/NotifyApplicant-Flow.md).
- Trigger: Record-Changed Flow on FundingRequest.Status__c
- Get Related: Contact.Email, Contact.MobilePhone from Applicant__c
- Decision: status → message template selection (Submitted / Approved / Denied / More Info Needed)
- Action: Send Email (standard) or Messaging Notification (if Messaging licensed)

**Message templates by status:**
- Submitted: "Your application [Name] has been received. Expected review timeline: [X] days."
- Approved: "Congratulations! Your application [Name] has been approved for $[Amount]."
- Denied: "We regret to inform you... [DecisionRationale summary]"
- More Info Needed: "Additional information is required... [specific items]"

---

## Action 11: Check for Duplicate Applications (NEW)

**Backing:** `FLOW`

**What it does:** Detects if the same organization or constituent has already submitted an application for the same FundingOpportunity in the current cycle.

**Flow design:**
- Get Records: FundingRequest WHERE FundingOpportunity__c = :opportunityId AND (Organization__c = :orgId OR Applicant__c = :contactId) AND Status__c != 'Withdrawn'
- Decision: count > 0?
- Output: `duplicateFound` (Boolean), `existingApplicationId` (Id), `existingApplicationStatus` (Text)

**Agent input:** `fundingOpportunityId`, `applicantId` or `organizationId`
**Agent output:** whether a prior application exists, its status, and the application ID

---

## Action 12: Budget Utilization Check (NEW)

**Backing:** `FLOW`

**What it does:** For post-award compliance, returns how much of an awarded budget has been spent and flags at-risk awards.

**Flow design:**
- Get Records: Budget WHERE FundingAward__c = :awardId
- Loop: sum AllocatedAmount, sum ActualAmount per category
- Formula: utilizationRate = totalActual / totalAllocated * 100
- Decision: utilizationRate < 50% AND ReportingDueDate within 30 days → atRisk = true
- Output: `utilizationRate` (Number), `atRisk` (Boolean), `budgetLines` (collection), `nextReportingDeadline`

**Agent input:** `fundingAwardId`
**Agent output:** utilization percentage, at-risk flag, per-category breakdown, next deadline
