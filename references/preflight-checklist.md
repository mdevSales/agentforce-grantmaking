# Pre-Flight Checklist: Org Readiness

Run this checklist **before** any discovery, spec work, or agent building. A missing item caught here takes 10 minutes to flag. A missing item caught during build takes hours to unblock.

---

## Checklist

### 1. PSC Grantmaking Installed
**Check:** Setup → Installed Packages → look for "Public Sector Solutions" or "Grantmaking"
**CLI check:**
```bash
sf package installed list --target-org YOUR_ORG_ALIAS | grep -i "grantmaking\|public sector"
```
**If missing:** Requires PSC Grantmaking license. Contact AE to provision. Cannot proceed without this.

---

### 2. Agentforce for Public Sector License Active
**Check:** Setup → Company Information → Permission Set Licenses → look for "Agentforce for Public Sector"
**If missing:** Separate SKU from PSC Grantmaking. Must be ordered and provisioned separately. Contact AE.

---

### 3. Einstein Agent User / Einstein Agent Access
**Check:**
```bash
sf data query --query "SELECT Id, PermissionSet.Name FROM PermissionSetAssignment WHERE Assignee.Name = 'YOUR_INTEGRATION_USER' AND PermissionSet.Name IN ('EinsteinAgentUser', 'EinsteinAgentAccess')" --target-org YOUR_ORG_ALIAS
```
**If missing:** Assign via `sf-permissions` skill or manually:
- Setup → Permission Sets → Einstein Agent User → Manage Assignments → Add integration user
- Setup → Permission Sets → Einstein Agent Access → Manage Assignments → Add integration user

---

### 4. Business Rules Engine (BRE) Configured
**Check:** Setup → Business Rules Engine → Rule Sets → look for eligibility-related rule sets
**If not configured:** Ask partner: "Do you want to use BRE for eligibility checking, or should we build a Flow-based eligibility check?"
- BRE = Yes → follow [bre-configuration-guide.md](bre-configuration-guide.md)
- BRE = No → use Flow fallback (Action 1 in [action-catalog.md](action-catalog.md))

**This check determines which eligibility action pattern to use — resolve before Spec work.**

---

### 5. OmniStudio Licensed
**Check:** Setup → Installed Packages → look for "OmniStudio" or "Vlocity"
**If not licensed:**
- The standard PSC application form (OmniScript) will not be available
- Use Flow screen or Experience Cloud LWC for application intake instead
- See [omnistudio-dependency-guide.md](omnistudio-dependency-guide.md)

---

### 6. Data 360 Licensed (if D360 is in scope)
**Check:** Setup → Data Cloud → look for active license OR:
```bash
sf data query --query "SELECT Id, PermissionSet.Name FROM PermissionSetAssignment WHERE PermissionSet.Name LIKE '%DataCloud%'" --target-org YOUR_ORG_ALIAS
```
**If not licensed:** Skip the entire D360 branch. Do not recommend D360 features. Revisit when the license is active.

---

### 7. Einstein Generative AI / Prompt Builder Enabled (for Prompt Template actions)
**Check:** Setup → Einstein Generative AI → verify it is enabled
**If not enabled:** Prompt Template-backed actions (Summarize, Score) will not work. Enable via Setup → Einstein Generative AI → toggle ON → accept terms.

---

### 8. Experience Cloud Enabled (for service/public-facing agents)
**Check:** Setup → Digital Experiences → verify at least one active community site
**If not set up:** Service agent cannot be deployed to constituents. Either set up Experience Cloud or scope to employee-facing agent only.

---

## Quick CLI Preflight Script

Run this to check the most common blockers at once:

```bash
ORG_ALIAS="YOUR_ORG_ALIAS"

echo "=== PSC Packages ==="
sf package installed list --target-org $ORG_ALIAS 2>/dev/null | grep -i "grantmaking\|public sector\|omni"

echo "=== Einstein Agent Perm Sets ==="
sf data query \
  --query "SELECT Id, PermissionSet.Name, Assignee.Name FROM PermissionSetAssignment WHERE PermissionSet.Name IN ('EinsteinAgentUser', 'EinsteinAgentAccess', 'AgentforcePublicSector')" \
  --target-org $ORG_ALIAS 2>/dev/null

echo "=== Data Cloud Perm Sets ==="
sf data query \
  --query "SELECT Id, PermissionSet.Name, Assignee.Name FROM PermissionSetAssignment WHERE PermissionSet.Name LIKE '%DataCloud%' OR PermissionSet.Name LIKE '%Data360%'" \
  --target-org $ORG_ALIAS 2>/dev/null

echo "=== BRE Rule Sets ==="
sf data query \
  --query "SELECT Id, Name, Status FROM BusinessRulesEngineRuleSet WHERE Status = 'Active'" \
  --target-org $ORG_ALIAS 2>/dev/null
```

---

## What to Do with Gaps

| Gap found | Action |
|---|---|
| PSC not installed | Stop. Cannot build without this. Escalate to AE. |
| Agentforce for PS not licensed | Stop. Cannot build agent features without this. Escalate to AE. |
| Einstein Agent User not assigned | Fix now via `sf-permissions` skill — 5 minutes. |
| BRE not configured | Choose path (BRE or Flow) before Spec. |
| OmniStudio not licensed | Choose intake alternative before Spec. |
| Data 360 not licensed | Remove D360 from scope for this build cycle. |
| Einstein GenAI not enabled | Enable in Setup → 2 minutes. Do it now. |
| Experience Cloud not set up | Scope to employee-facing agent only, or set up Experience Cloud first. |
