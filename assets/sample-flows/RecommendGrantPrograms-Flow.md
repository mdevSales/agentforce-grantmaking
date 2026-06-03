# Flow Design: Recommend Grant Programs

**Type:** Autolaunched Flow
**API Name:** `RecommendGrantPrograms`
**Purpose:** Returns active grant programs a constituent is likely eligible for, ranked by deadline.

---

## Input Variables

| Variable | Type | Required | Notes |
|---|---|---|---|
| applicantType | Text | No | 'Individual', 'Small Business', 'Nonprofit', 'Government' |
| geographyCode | Text | No | State abbreviation (e.g., 'CA') |
| requestedAmountMin | Currency | No | Minimum award needed |
| requestedAmountMax | Currency | No | Maximum desired |

## Output Variables

| Variable | Type | Notes |
|---|---|---|
| recommendedPrograms | Record Collection (FundingOpportunity) | Matching programs, max 5 |
| programCount | Number | How many matched |
| noMatchReason | Text | If 0 results, why |

---

## Flow Elements

### Element 1: Get Active FundingOpportunities
**Type:** Get Records
**Object:** FundingOpportunity
**Filter:**
- Status__c = 'Active'
- SubmissionDeadline__c > TODAY (or null — open-ended programs)
**Sort:** SubmissionDeadline__c Ascending (soonest deadline first)
**Limit:** 20 (filter down in loop)
**Fields:** Id, Name, ProgramDescription__c, MaxAwardAmount__c, MinAwardAmount__c, SubmissionDeadline__c, EligibleOrgTypes__c, EligibilityGeographies__c, FundingType__c

### Element 2: Decision — Any active programs?
- No → Assignment: `noMatchReason = 'No active grant programs are currently accepting applications'`, `programCount = 0` → End

### Element 3: Loop — Filter to matching programs
**Loop variable:** currentProgram
**For each program:**

- **Geography check:**
  `IF geographyCode is not blank AND EligibilityGeographies__c is not blank:`
  `IF NOT CONTAINS(EligibilityGeographies__c, geographyCode): skip (add to excludedList, not matchedList)`

- **Org type check:**
  `IF applicantType is not blank AND EligibleOrgTypes__c is not blank:`
  `IF NOT CONTAINS(EligibleOrgTypes__c, applicantType): skip`

- **Amount check:**
  `IF requestedAmountMin is not blank AND MaxAwardAmount__c < requestedAmountMin: skip`
  `IF requestedAmountMax is not blank AND MinAwardAmount__c > requestedAmountMax: skip`

- **If all checks passed:** Add to `matchedPrograms` collection

### Element 4: Decision — Any matches?
- No → Assignment: `noMatchReason = 'No programs currently match your criteria. Programs accepting your region/org type may open soon.'`
- Yes → Element 5

### Element 5: Assignment — Limit to top 5
`recommendedPrograms = FIRST 5 from matchedPrograms` (using loop counter or subflow)
`programCount = COUNT(matchedPrograms)` (total matches before limit)

### Element 6: End

---

## Agent Response Pattern

After calling this Flow:
```
"I found {programCount} grant programs that match your profile. Here are the top options:

1. {program1.Name} — Up to ${program1.MaxAwardAmount__c} — Deadline: {program1.SubmissionDeadline__c}
   {program1.ProgramDescription__c (first 100 chars)}

2. {program2.Name} — ...

Would you like to check your eligibility for any of these programs?"
```

---

## Testing

```apex
Map<String, Object> inputs = new Map<String, Object>{
    'applicantType' => 'Small Business',
    'geographyCode' => 'CA',
    'requestedAmountMin' => 10000
};
Flow.Interview.RecommendGrantPrograms flow =
    new Flow.Interview.RecommendGrantPrograms(inputs);
flow.start();
System.debug('Count: ' + flow.getVariableValue('programCount'));
```
