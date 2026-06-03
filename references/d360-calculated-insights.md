# Data 360 Calculated Insights for Grantmaking

Calculated Insights (CIOs — `__cio` suffix) are pre-computed metrics in Data 360. They're computed on a schedule, cached, and available for instant lookup — unlike aggregation queries which run at request time.

For grantmaking, CIs are high-value, low-latency signals that the agent can surface in actions.

---

## Why CIs Matter for Grantmaking

A real-time SOQL query like "what is this organization's total lifetime grant funding?" hits the database on every agent invocation, adding latency and consuming API limits. A Calculated Insight pre-computes this nightly and serves it instantly.

CIs are especially valuable for:
- Summary statistics that would otherwise require complex aggregations
- Scores and indices that need to be consistent across channels
- Metrics used for decision-making (compliance rate, vulnerability index)

---

## High-Value CIs for Grantmaking

### 1. Lifetime Grants Received by Household
**API name (example):** `LifetimeGrantsByHousehold__cio`
**What it computes:** Total grant awards received by all members of a household, across all programs and time periods.
**Why it matters:** Prevents households from receiving grants beyond program caps. Surfaces total public benefit context.

```sql
-- Query from agent action
SELECT 
  ssot__IndividualId__c,
  LifetimeGrantAmount__c,
  GrantCount__c,
  LastGrantDate__c
FROM LifetimeGrantsByHousehold__cio
WHERE ssot__IndividualId__c = '[d360IndividualId]'
```

### 2. Compliance Rate by Organization
**API name (example):** `ComplianceRateByOrg__cio`
**What it computes:** Percentage of past FundingAwards for which the organization submitted compliance reports on time.
**Why it matters:** High-risk applicants with poor compliance history should be flagged during the eligibility or review process.

```sql
SELECT 
  ssot__AccountId__c,
  ComplianceRate__c,
  TotalAwards__c,
  LateReports__c,
  MissedReports__c
FROM ComplianceRateByOrg__cio
WHERE ssot__AccountId__c = '[d360OrgId]'
```

**Agent use:** "This organization has a compliance rate of 67% across 6 prior awards (2 late reports, 1 missed report). Consider flagging for enhanced monitoring."

### 3. Vulnerability Index
**API name (example):** `VulnerabilityIndex__cio`
**What it computes:** A composite score (0-100) indicating how economically or socially vulnerable a constituent is, based on income, housing stability, prior benefit history, and service interaction patterns.
**Why it matters:** Many grant programs prioritize the most vulnerable constituents. This score enables consistent, data-driven prioritization without manual caseworker assessment.

```sql
SELECT 
  ssot__IndividualId__c,
  VulnerabilityScore__c,
  VulnerabilityTier__c,
  LastUpdated__c
FROM VulnerabilityIndex__cio
WHERE ssot__IndividualId__c = '[d360IndividualId]'
```

**Agent use:** "This applicant has a vulnerability score of 78/100 (High tier). This program prioritizes high-vulnerability applicants."

### 4. Cross-Program Benefit Overlap (if configured)
**API name (example):** `CrossProgramOverlap__cio`
**What it computes:** Whether a constituent is currently receiving benefits from other programs that may create overlap or eligibility conflicts.

---

## Building Calculated Insights

CIs are configured in D360 → Calculated Insights → New. The definition is an ANSI SQL aggregation query.

**Example — Compliance Rate CI:**
```sql
CREATE OR REPLACE CALCULATED INSIGHT ComplianceRateByOrg AS
SELECT 
  c.ssot__AccountId__c,
  COUNT(c.ssot__Id__c) AS TotalAwards__c,
  SUM(CASE WHEN c.ssot__ComplianceStatus__c = 'Late' THEN 1 ELSE 0 END) AS LateReports__c,
  SUM(CASE WHEN c.ssot__ComplianceStatus__c = 'Missed' THEN 1 ELSE 0 END) AS MissedReports__c,
  (1 - (SUM(CASE WHEN c.ssot__ComplianceStatus__c != 'Compliant' THEN 1 ELSE 0 END) / COUNT(c.ssot__Id__c))) * 100 AS ComplianceRate__c
FROM ssot__Benefit__dlm c
WHERE c.ssot__BenefitType__c = 'Grant'
GROUP BY c.ssot__AccountId__c
```

---

## Surfacing CIs in Agent Actions

### NGA Agent Script pattern
```
action GetComplianceHistory {
  type: Flow
  flowApiName: "GetOrgComplianceCI"
  inputs {
    organizationId: @applicantOrgId
  }
  outputs {
    complianceRate -> @orgComplianceRate
    totalAwards -> @orgTotalAwards
  }
}

when @orgComplianceRate < 70 {
  response "Note: This organization has a compliance rate of {@orgComplianceRate}% across {@orgTotalAwards} prior awards. This may be relevant for the reviewer."
}
```

### Flow for CI lookup
The Flow calls a Data Cloud query (via D360 REST API or `sf data360` CLI) to retrieve the CI value for a specific constituent or org, then returns it as a Flow output variable for the agent to consume.

---

## Refresh Schedule

CIs are recomputed on a schedule (typically nightly). If a constituent just received an award today, `LifetimeGrantsByHousehold__cio` won't reflect it until tomorrow's refresh. For real-time accuracy, fall back to a live SOQL query on the PSC objects.

Tell the partner: "CIs are pre-computed — perfect for decision-making that tolerates a day's lag. For real-time status, use direct SOQL on FundingAward."
