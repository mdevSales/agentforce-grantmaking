# Data 360 Identity Resolution for Grantmaking

## Why Identity Resolution Is the Core Value Prop

**Without Identity Resolution:** The agent only sees grants data. A constituent who applied for a housing grant through one system, received food assistance through a different system, and opened a case management record through a third system is three separate records — unknown to each other.

**With Identity Resolution:** D360 recognizes that the person with email "j.smith@gmail.com" in the grants system is the same person as "John Smith" at "123 Main St" in the benefits system, who matches the SSN ending in 4821 in the case management system. One unified `Individual__dlm` profile with all interactions, benefits, and cases attached.

For grantmaking specifically, this means:
- "Has this household already received grants from similar programs?" — answerable without manual lookup
- "Is this applicant already receiving overlapping benefits?" — instant cross-program check
- "What is this person's full service history with this agency?" — complete context for the reviewer
- Prioritization by vulnerability index (a composite of data from multiple systems)

---

## How Identity Resolution Works in D360

Identity Resolution (IR) uses configurable **match rules** to find records across data sources that represent the same real-world person or organization.

### Match rules (examples for grants context)
| Rule | Match criteria | Confidence |
|---|---|---|
| Exact Email | email1 == email2 (case-insensitive) | High |
| Exact SSN Last 4 + DOB | last4SSN match AND dateOfBirth match | High |
| Name + Address | firstName + lastName + zip code fuzzy match | Medium |
| Phone + DOB | mobilePhone match AND dateOfBirth match | Medium |

### What IR produces
- A unified `ssot__Individual__dlm` record with a D360-generated ID
- `ssot__ContactPoint*__dlm` records linking all emails, phones, addresses
- A match confidence score per resolved identity
- A "source records" list showing which systems contributed to the profile

---

## Setting Up Identity Resolution

1. D360 → Identity Resolution → New Ruleset
2. Configure match rules (start with Exact Email + Name+Address)
3. Run the ruleset: D360 → Identity Resolution → Run
4. Review unmatched/low-confidence records in the IR dashboard
5. Schedule recurring runs (nightly recommended)

**Important:** IR is not instant. First run on a large org can take hours. Plan for it in the project timeline.

---

## Connecting PSC Systems to IR

For grants to benefit from IR, ALL relevant systems must be ingested into D360:

| System | D360 Connection type | What it provides |
|---|---|---|
| PSC Grantmaking (Salesforce org) | SalesforceDotCom connector | FundingRequest, Contact, Account |
| Case Management (if separate Salesforce org) | SalesforceDotCom connector (second connection) | Cases, Contacts |
| Benefits system (external) | Ingestion API or Snowflake connector | Benefit records |
| Housing / social services (external) | Ingestion API | Service records |

The more systems connected, the richer the unified profile.

---

## What the Agent Can Do With IR

Once IR is running and profiles are unified:

### Cross-program overlap check
```sql
-- All active benefits for this unified constituent
SELECT 
  b.ssot__BenefitType__c,
  b.ssot__Program__c,
  b.ssot__Amount__c,
  b.ssot__Status__c
FROM ssot__Benefit__dlm b
WHERE b.ssot__IndividualId__c = '[d360UnifiedId]'
AND b.ssot__Status__c = 'Active'
```

### Full service history
```sql
SELECT 
  si.ssot__InteractionType__c,
  si.ssot__Channel__c,
  si.ssot__Program__c,
  si.ssot__InteractionDate__c
FROM ssot__ServiceInteraction__dlm si
WHERE si.ssot__IndividualId__c = '[d360UnifiedId]'
ORDER BY si.ssot__InteractionDate__c DESC
LIMIT 50
```

### Look up the D360 unified ID from a Salesforce Contact ID
```bash
bash ~/.claude/skills/salesforce-data-360/scripts/dc-query.sh \
  "SELECT ssot__Id__c, ssot__FirstName__c, ssot__LastName__c FROM ssot__Individual__dlm WHERE ssot__SourceRecordId__c = '[sfContactId]' LIMIT 1"
```

---

## Communicating IR Value to the Partner

When presenting D360 to a partner, lead with this framing:

> "Right now, your grants system doesn't know if the organization applying for a small business grant is the same organization that defaulted on compliance for an infrastructure grant two years ago — because those are in different records or different systems. Data 360 Identity Resolution solves this. Once connected, your grantmaking agent has the same complete view of every constituent and organization that your best caseworker would build manually — but instantly, for every application."

This is the 'why D360 matters' story — not 'it's possible to connect data,' but 'without it, your agent is operating blind on 70% of the context that should inform these decisions.'

---

## IR Timing in the Project Plan

IR is not a day-one feature — it needs:
1. All source systems connected and ingesting (1-2 weeks typically)
2. First IR run to complete (hours to days for large datasets)
3. Match rule tuning after reviewing low-confidence matches (1 week)
4. Partner sign-off on unified profiles before going live

Recommend starting IR setup in parallel with agent development so it's ready when the agent goes live.
