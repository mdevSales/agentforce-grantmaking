# Business Rules Engine (BRE) Configuration Guide

BRE powers the **Check Eligibility** action. If BRE is not configured, the action falls back to a SOQL-based Flow.

---

## What BRE Does for Grantmaking

Business Rules Engine evaluates a set of configurable rules against input data and returns a decision. For grantmaking:
- Input: applicant data (organization type, geography, requested amount, program type)
- Rules: eligibility criteria configured by the program officer
- Output: ELIGIBLE / INELIGIBLE / NEEDS_REVIEW + explanation array

BRE rules are maintained by program officers in the UI — no code changes needed when eligibility criteria change.

---

## Path A: BRE Configured — Setup Steps

### 1. Enable BRE
Setup → Business Rules Engine → Enable Business Rules Engine

### 2. Create a Rule Set for each FundingOpportunity
1. Setup → Business Rules Engine → Rule Sets → New
2. Name: `GrantEligibility_[ProgramName]` (e.g., `GrantEligibility_SmallBusinessRelief`)
3. Status: Active
4. Object: `FundingRequest__c`

### 3. Add Rules to the Rule Set

Example rule set for a small business grant:

| Rule | Condition | Action |
|---|---|---|
| Organization Type Check | `FundingRequest.Organization__r.Type = 'Small Business'` | Set `EligibilityStatus = ELIGIBLE` |
| Revenue Cap | `FundingRequest.Organization__r.AnnualRevenue <= 1000000` | Set `EligibilityStatus = ELIGIBLE` |
| Geography Check | `FundingRequest.Organization__r.BillingState IN ('CA','OR','WA')` | Set `EligibilityStatus = ELIGIBLE` |
| Amount Check | `FundingRequest.RequestedAmount__c <= 50000` | Set `EligibilityStatus = ELIGIBLE` |
| Default — all rules failed | (no condition — default) | Set `EligibilityStatus = INELIGIBLE` |

### 4. Create an Invocable Apex wrapper (if needed for agent action)

```apex
public class CheckGrantEligibilityBRE {
    @InvocableMethod(label='Check Grant Eligibility via BRE')
    public static List<EligibilityResult> checkEligibility(List<EligibilityRequest> requests) {
        List<EligibilityResult> results = new List<EligibilityResult>();
        for (EligibilityRequest req : requests) {
            // Call BRE rule set
            DecisionTable.DecisionTableResult breResult =
                DecisionTable.evaluate(req.ruleSetApiName, req.fundingRequestId);
            EligibilityResult result = new EligibilityResult();
            result.eligibilityStatus = (String) breResult.outputs.get('EligibilityStatus');
            result.eligibilityReason = (String) breResult.outputs.get('EligibilityReason');
            results.add(result);
        }
        return results;
    }

    public class EligibilityRequest {
        @InvocableVariable public String ruleSetApiName;
        @InvocableVariable public Id fundingRequestId;
    }

    public class EligibilityResult {
        @InvocableVariable public String eligibilityStatus;
        @InvocableVariable public String eligibilityReason;
    }
}
```

### 5. Wire as autolaunched Flow action

Alternatively, call BRE from within a Flow:
1. Create Autolaunched Flow: `CheckGrantEligibility_BRE`
2. Input variable: `fundingRequestId` (Id), `ruleSetApiName` (Text)
3. Action element: Apex Action → `CheckGrantEligibilityBRE`
4. Output variables: `eligibilityStatus`, `eligibilityReason`

### 6. Connect to agent action
The agent calls the Flow as an invocable action. The Flow calls BRE. This keeps the agent decoupled from the BRE invocation mechanism.

---

## Path B: BRE Not Configured — Flow Fallback

When BRE is not set up, eligibility checking uses a SOQL-based Flow that queries criteria fields directly on the FundingOpportunity.

**This is less powerful than BRE** — criteria are hardcoded in the Flow, not configurable by program officers. Recommend BRE as a future-state migration.

### Fallback Flow: CheckGrantEligibility_Flow

**Inputs:** `fundingRequestId` (Id)

**Logic:**
1. Get Records: FundingRequest (with related FundingOpportunity, Organization, Contact)
2. Get Records: FundingOpportunity (MaxAwardAmount, EligibilityGeographies, EligibleOrgTypes)
3. Decision 1: RequestedAmount <= MaxAwardAmount?
4. Decision 2: Organization.Type in EligibleOrgTypes? (stored as comma-delimited text field)
5. Decision 3: Organization.BillingState in EligibilityGeographies?
6. Assignment: build `eligibilityReason` based on which decisions passed/failed
7. Final Assignment:
   - All passed → `eligibilityStatus = 'ELIGIBLE'`
   - Any failed → `eligibilityStatus = 'INELIGIBLE'`
   - Ambiguous → `eligibilityStatus = 'NEEDS_REVIEW'`

**Output variables:** `eligibilityStatus` (Text), `eligibilityReason` (Text)

See [../assets/sample-flows/CheckGrantEligibility-Flow.md](../assets/sample-flows/CheckGrantEligibility-Flow.md) for full design.

---

## Salesforce BRE Developer Documentation

For detailed BRE rule set configuration, invoke:
```
WebFetch: https://developer.salesforce.com/docs/atlas.en-us.industries_ref.meta/industries_ref/industries_common_bre_about.htm
```
This fetches the official PSC BRE reference guide with syntax for conditions, decision tables, and invocable actions.

---

## When to Recommend BRE vs Flow

| Factor | Use BRE | Use Flow Fallback |
|---|---|---|
| Program officers need to change criteria without IT | Yes — BRE | No advantage |
| Multiple programs with different criteria | Yes — one rule set per program | Requires multiple Flows |
| Criteria are stable and managed by IT | Either | Flow is simpler |
| Complex scoring (weighted criteria) | Yes — BRE tables | Difficult in Flow |
| BRE licensed and installed | Yes | Only if BRE not available |
