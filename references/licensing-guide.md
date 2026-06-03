# Licensing Guide: 3 SKUs for Grantmaking Agents

**Surface this in Phase 0 — before any discovery or spec work.** Partners need to confirm all required SKUs are on the order before starting a POC. A missed SKU discovered during build can stall a project for weeks.

---

## The 3 Required SKUs

### SKU 1: PSC Grantmaking (required)
**Full name:** Public Sector Solutions — Grantmaking
**What it provides:**
- FundingOpportunity, FundingRequest, GrantApplicationReview, FundingAward, Budget objects
- BRE integration for eligibility checking
- DocGen for award/denial letters
- Outcome Management for compliance
- Decision Explainer
- OmniScript application forms (if OmniStudio also licensed)
- Experience Cloud for Grantmaking permission set

**Who needs it:** The Salesforce org must have this license. Not user-level — org-level.
**Cannot build without this.** If missing, stop and escalate to AE.

---

### SKU 2: Agentforce for Public Sector (required for Agentforce)
**Full name:** Agentforce for Public Sector
**What it provides:**
- Permission to use Agentforce / Einstein Bots features in a PSC context
- Agentforce for Public Sector permission set (user-level)
- Access to NGA Agent Script for government orgs
- Agentforce conversations quota

**Who needs it:** Every user who interacts with the agent. Also required on the integration user.
**This is separate from base Agentforce licensing.** A customer with Agentforce (non-PS) does NOT automatically have Agentforce for Public Sector.

---

### SKU 3: Data 360 for Public Sector (required only if D360 is in scope)
**Full name:** Data 360 for Government / Data 360 for Public Sector
**What it provides:**
- The harmonized Customer 360 Data Model for Government (Individual, Case, Benefit, ServiceInteraction DMOs)
- Identity Resolution across systems
- Calculated Insights
- Data 360 CLI plugin compatibility (`sf data360` commands)
- `sf-pi` `/sf-data360` extension support

**Who needs it:** Org-level license. Only required if the partner is building constituent profile features, cross-program overlap checking, or vulnerability index / CI-based prioritization.
**Cannot use D360 features without this.** If partner wants D360 but doesn't have this SKU, set the D360 branch aside and revisit when the license is active.

---

## Optional / Related Licenses

| License | When needed | Not required for basic agent |
|---|---|---|
| OmniStudio | OmniScript application forms | Yes — fallback is Flow screen |
| Experience Cloud | Public-facing / constituent portal | Yes — only for Service agent pattern |
| Messaging (SMS/WhatsApp) | Notify Applicant via text message | Yes — email notification works without it |
| Einstein GPT / Prompt Builder | Prompt Template-backed actions | Required for Summarize + Score actions |

---

## Partner Conversation Guide

When a partner says "we want to build a grantmaking agent":

> "Before we design anything, let's confirm you have all three licenses active in this sandbox:
> 1. PSC Grantmaking — this is the data model foundation, everything else depends on it
> 2. Agentforce for Public Sector — this is what makes the agent work in a PSC context, separate SKU from base Agentforce
> 3. Data 360 for Public Sector — only if you want to bring in constituent profiles and cross-program data; we can start without this and add it later
>
> Can you check Setup → Installed Packages and Company Information → Permission Set Licenses and confirm these are active?"

---

## Org Check

```bash
# Check installed packages for PSC
sf package installed list --target-org YOUR_ORG_ALIAS | grep -i "public sector\|grantmaking"

# Check permission set licenses
sf data query \
  --query "SELECT MasterLabel, TotalLicenses, UsedLicenses FROM PermissionSetLicense WHERE MasterLabel LIKE '%Agentforce%' OR MasterLabel LIKE '%Einstein%' OR MasterLabel LIKE '%Data%'" \
  --target-org YOUR_ORG_ALIAS
```
