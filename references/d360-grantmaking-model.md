# Data 360 for Public Sector: Grantmaking Semantic Model

Data 360 for Public Sector (formerly "Data Cloud for Public Sector") has its own harmonized semantic data model called the **Customer 360 Data Model for Government**. This is NOT the same as generic Data Cloud — it includes government-specific DMOs (Data Model Objects) like `Case`, `Benefit`, `ServiceInteraction`, and `Individual` that map directly to PSC objects.

---

## PSC Object → D360 DMO Mapping

| PSC Object | D360 DMO | Notes |
|---|---|---|
| Contact / Individual | `ssot__Individual__dlm` | Constituent profile — unified across systems via Identity Resolution |
| Account (Organization) | `ssot__Account__dlm` | Applicant organization |
| FundingRequest | `ssot__Case__dlm` | Grant application treated as a "case" in the constituent journey |
| FundingAward | `ssot__Benefit__dlm` | Award treated as a "benefit received" — enables cross-program view |
| FundingOpportunity | `ssot__ServiceCatalogItem__dlm` | Grant program as a "service" offered to constituents |
| Service interactions (calls, portal sessions) | `ssot__ServiceInteraction__dlm` | Constituent touchpoints across all service channels |
| Case (for escalations) | `ssot__Case__dlm` | Escalation cases also map here |

---

## Why This Mapping Matters

Without D360: the agent only knows about grants.

With D360 + this mapping: the agent can see the **full constituent profile** — whether this individual is also receiving housing benefits, has open case management cases, has interacted with other programs. This enables:
- "Has this household already received grants from other programs?" (cross-program duplicate/overlap check)
- "What is this organization's compliance history across all benefit programs?"
- "Does this constituent have open case management issues that should be resolved before awarding a grant?"

---

## D360 Activation Sequence

**Always in this order:**

### 1. Activate (`sf-datacloud-act` skill)
Set up activation targets and downstream delivery first. The activation target defines where D360 outputs flow — typically back into Salesforce CRM for agent consumption.
```bash
# Use sf-datacloud-act skill
# Key commands:
sf data360 activation-target create -o YOUR_ORG -f target.json
sf data360 activation create -o YOUR_ORG -f activation.json
```

### 2. Connect (`sf-datacloud-connect` skill)
Set up the Salesforce org connector so D360 can ingest PSC data.
```bash
# Use sf-datacloud-connect skill
# Key commands:
sf data360 connection list -o YOUR_ORG --connector-type SalesforceDotCom
sf data360 connection create -o YOUR_ORG -f salesforce-connection.json
```

### 3. Harmonize (`sf-datacloud-harmonize` skill)
Map the PSC objects to D360 DMOs using the table above. This step creates the semantic links that enable unified constituent profiles.
```bash
# Use sf-datacloud-harmonize skill
# Map FundingRequest__c → ssot__Case__dlm
# Map FundingAward__c → ssot__Benefit__dlm
# Map Contact → ssot__Individual__dlm (usually auto-mapped)
```

---

## Schema Exploration

After activation, explore what DMOs are available and what fields they have:

**Option A — `salesforce-data-360` skill (bash scripts via Anonymous Apex):**
```bash
# List all DMOs
bash ~/.claude/skills/salesforce-data-360/scripts/dc-list-objects.sh

# Describe the Case DMO (FundingRequest mapping)
bash ~/.claude/skills/salesforce-data-360/scripts/dc-describe.sh ssot__Case__dlm

# Query grant applications as Cases
bash ~/.claude/skills/salesforce-data-360/scripts/dc-query.sh \
  "SELECT ssot__Id__c, ssot__IndividualId__c, ssot__Status__c FROM ssot__Case__dlm LIMIT 10"
```

**Option B — `sf-pi` `/sf-data360` extension (if installed):**
```bash
# Install sf-pi
pi install git:github.com/salesforce/sf-pi

# Then use:
# /sf-data360 — opens Data 360 panel
# data360_discover — probes org availability
# data360_query — run ANSI SQL against DMOs
# data360_api — REST escape hatch
```

---

## Key D360 ANSI SQL Patterns for Grantmaking

```sql
-- Constituent's full benefit history (grants + other benefits)
SELECT 
  i.ssot__FirstName__c, 
  i.ssot__LastName__c,
  b.ssot__BenefitType__c,
  b.ssot__Amount__c,
  b.ssot__StartDate__c
FROM ssot__Individual__dlm i
JOIN ssot__Benefit__dlm b ON i.ssot__Id__c = b.ssot__IndividualId__c
WHERE i.ssot__Id__c = '[constituentD360Id]'
ORDER BY b.ssot__StartDate__c DESC;

-- All grant cases for an organization (cross-program overlap check)
SELECT 
  a.ssot__Name__c,
  c.ssot__Subject__c,
  c.ssot__Status__c,
  c.ssot__CreatedDate__c
FROM ssot__Account__dlm a
JOIN ssot__Case__dlm c ON a.ssot__Id__c = c.ssot__AccountId__c
WHERE a.ssot__Id__c = '[orgD360Id]'
ORDER BY c.ssot__CreatedDate__c DESC;

-- Service interactions for a constituent (full touchpoint history)
SELECT 
  ssot__IndividualId__c,
  ssot__InteractionType__c,
  ssot__Channel__c,
  ssot__InteractionDate__c
FROM ssot__ServiceInteraction__dlm
WHERE ssot__IndividualId__c = '[constituentD360Id]'
ORDER BY ssot__InteractionDate__c DESC
LIMIT 20;
```

---

## Field Namespace Note

D360 field names carry `ssot__` prefixes that **vary by org**. Always run `dc-describe.sh` or `data360_discover` before writing SQL — never guess field names.

---

## Licensing and Enablement

Data 360 for Public Sector is a separate SKU. Before any D360 work:
1. Confirm the license is active (see [preflight-checklist.md](preflight-checklist.md))
2. Confirm Data Spaces are configured for your data access model
3. Confirm Identity Resolution is set up (see [d360-identity-resolution.md](d360-identity-resolution.md))
