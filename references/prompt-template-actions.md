# Prompt Template-Backed Actions

LLM-powered actions use `GenAiPromptTemplate` metadata (flex type). These are distinct from Flow and STANDARD actions — they call an LLM at runtime and return generated text.

**Delegate to `salesforce-prompt-templates` skill** for full XML authoring, deployment, and activation. This file provides the input/output contracts and recommended XML patterns.

---

## When to Use Prompt Templates

Use `PROMPT_TEMPLATE` backing when the action needs to:
- **Generate** content (narratives, summaries, letters)
- **Evaluate** or score against criteria using natural language judgment
- **Transform** structured data into prose

Do NOT use for: data queries, status lookups, record creation, eligibility rule evaluation (use BRE or Flow instead).

---

## Action 8: Summarize Application for Reviewer

**Purpose:** Generates a concise structured summary of a FundingRequest for a human reviewer.

**Template XML:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<GenAiPromptTemplate xmlns="http://soap.sforce.com/2006/04/metadata">
    <description>Summarizes a grant application for a reviewer — applicant, program, amount, purpose, flags.</description>
    <developerName>Grantmaking_Summarize_Application</developerName>
    <masterLabel>Grantmaking Summarize Application</masterLabel>
    <overridable>false</overridable>
    <templateVersions>
        <content>You are a grant review assistant. Summarize the following grant application in 3-5 sentences for a human reviewer.

Application details:
- Applicant: {!$Input:Application.Applicant__r.Name}
- Organization: {!$Input:Application.Organization__r.Name}
- Program applied for: {!$Input:Application.FundingOpportunity__r.Name}
- Amount requested: {!$Input:Application.RequestedAmount__c}
- Submission date: {!$Input:Application.SubmissionDate__c}
- Current status: {!$Input:Application.Status__c}
- Eligibility status: {!$Input:Application.EligibilityStatus__c}

Focus on: who is applying, what they are asking for, why (stated purpose from the application), and any flags (ineligibility, missing documents, duplicate application indicator).

Be factual. Do not make recommendations. Do not add information not present in the record.</content>
        <fileDroppingStrategy>{"dropFilesBy":"LAST_MODIFIED_DATE","dropFileStartingWith":"OLDEST"}</fileDroppingStrategy>
        <inputs>
            <apiName>Application</apiName>
            <definition>SOBJECT://FundingRequest__c</definition>
            <masterLabel>Application</masterLabel>
            <referenceName>Input:Application</referenceName>
            <required>true</required>
        </inputs>
        <isCitationEnabled>false</isCitationEnabled>
        <primaryModel>sfdc_ai__DefaultVertexAIGemini25Flash001</primaryModel>
        <status>Published</status>
        <versionIdentifier>grantmake_summ_1</versionIdentifier>
    </templateVersions>
    <type>einstein_gpt__flex</type>
    <visibility>Global</visibility>
</GenAiPromptTemplate>
```

**File path:** `force-app/main/default/genAiPromptTemplates/Grantmaking_Summarize_Application.genAiPromptTemplate-meta.xml`

**Deploy:**
```bash
sf project deploy start \
  --metadata "GenAiPromptTemplate:Grantmaking_Summarize_Application" \
  --target-org YOUR_ORG_ALIAS
```

**Agent wiring (NGA):**
```
action SummarizeApplication {
  type: PromptTemplate
  templateApiName: "Grantmaking_Summarize_Application"
  inputs {
    Application: @fundingRequestId
  }
  output: @applicationSummary
}
```

**Agent wiring (legacy):**
- Agent Action → Action Type: Prompt Template → select `Grantmaking_Summarize_Application`
- Map `Application` input to the FundingRequest record variable

---

## Action 9: Score Application Against Criteria

**Purpose:** Evaluates a FundingRequest against scoring criteria and returns per-criterion scores with rationale.

**Template XML:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<GenAiPromptTemplate xmlns="http://soap.sforce.com/2006/04/metadata">
    <description>Scores a grant application against defined criteria and returns per-criterion scores with rationale.</description>
    <developerName>Grantmaking_Score_Application</developerName>
    <masterLabel>Grantmaking Score Application</masterLabel>
    <overridable>false</overridable>
    <templateVersions>
        <content>You are a grant scoring assistant. Score the following grant application against the provided criteria.

Application:
- Applicant: {!$Input:Application.Applicant__r.Name}
- Program: {!$Input:Application.FundingOpportunity__r.Name}
- Requested amount: {!$Input:Application.RequestedAmount__c}

Scoring criteria:
{!$Input:ScoringCriteria}

Instructions:
- For each criterion, provide a score from 1 (does not meet) to 5 (exceeds expectations)
- Provide a one-sentence rationale for each score
- Provide a suggested total score (sum of all criteria scores)
- End with: "NOTE: This is AI-assisted scoring. All scores must be reviewed and confirmed by a human reviewer before use."

Format your response as:
Criterion 1 - [Criterion Name]: [Score]/5 — [Rationale]
Criterion 2 - [Criterion Name]: [Score]/5 — [Rationale]
...
Suggested total: [X]/[max]</content>
        <fileDroppingStrategy>{"dropFilesBy":"LAST_MODIFIED_DATE","dropFileStartingWith":"OLDEST"}</fileDroppingStrategy>
        <inputs>
            <apiName>Application</apiName>
            <definition>SOBJECT://FundingRequest__c</definition>
            <masterLabel>Application</masterLabel>
            <referenceName>Input:Application</referenceName>
            <required>true</required>
        </inputs>
        <inputs>
            <apiName>ScoringCriteria</apiName>
            <definition>primitive://String</definition>
            <masterLabel>Scoring Criteria</masterLabel>
            <referenceName>Input:ScoringCriteria</referenceName>
            <required>true</required>
        </inputs>
        <isCitationEnabled>false</isCitationEnabled>
        <primaryModel>sfdc_ai__DefaultBedrockAnthropicClaude45Sonnet</primaryModel>
        <status>Published</status>
        <versionIdentifier>grantmake_score_1</versionIdentifier>
    </templateVersions>
    <type>einstein_gpt__flex</type>
    <visibility>Global</visibility>
</GenAiPromptTemplate>
```

**File path:** `force-app/main/default/genAiPromptTemplates/Grantmaking_Score_Application.genAiPromptTemplate-meta.xml`

**Model note:** Uses Claude Sonnet 4.5 (`sfdc_ai__DefaultBedrockAnthropicClaude45Sonnet`) rather than Gemini Flash because structured multi-criterion scoring benefits from stronger reasoning. Acceptable higher cost — reviewer-only action, not constituent-facing.

---

## Action (Optional): Draft Award Decision Narrative

**Purpose:** Generates a legally-phrased approval or denial letter narrative for a grant decision.

**Template design:**
- Inputs: `FundingRequest__c` SObject + `decisionType` (String: "Approved" or "Denied") + `decisionRationale` (String: reviewer notes)
- Model: `sfdc_ai__DefaultBedrockAnthropicClaude45Sonnet`
- Output: formal letter narrative (1-2 paragraphs)

**Note:** Decision Explainer (see [decision-explainer-guide.md](decision-explainer-guide.md)) populates the structured audit trail separately. This Prompt Template generates the human-facing letter body.

---

## Deployment Checklist

Before deploying Prompt Template actions:
- [ ] `developerName` matches filename (without extension)
- [ ] `type` is `einstein_gpt__flex`
- [ ] `visibility` is `Global`
- [ ] `overridable` is `false`
- [ ] All SObject inputs reference valid PSC object API names
- [ ] `activeVersionIdentifier` matches the `versionIdentifier` of the Published version
- [ ] `fileDroppingStrategy` present on each version
- [ ] No grounding dependencies (these templates are standalone — no Flow or Apex grounding needed)

---

## Testing

After deployment:
```bash
# Verify template is deployed and active
sf project retrieve start \
  --metadata "GenAiPromptTemplate:Grantmaking_Summarize_Application" \
  --target-org YOUR_ORG_ALIAS

# Test via Prompt Builder UI: Setup → Prompt Builder → select template → Preview
```
