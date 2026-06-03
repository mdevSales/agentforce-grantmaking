# agentforce-grantmaking

A Claude skill that guides end users through designing, building, and deploying Agentforce agents for Public Sector grantmaking. Covers the full lifecycle: org readiness, PSC data model, action selection, Prompt Template-backed LLM actions, multi-turn state, human handoff, authentication gates, Data 360 integration, and agent deployment.

---

## Install

```bash
# Clone into your Claude skills directory
git clone https://github.com/salesforce-gps/agentforce-grantmaking ~/.claude/skills/agentforce-grantmaking
```

Then start a new Claude Code session — the skill auto-loads.

---

## Invoke

```
I want to build a grantmaking agent for a county running PSC in a sandbox.
```

or

```
Help me build an Agentforce agent for our grant management program. We have PSC installed.
```

---

## What This Skill Does

1. **Orients** you on the PSC grantmaking data model and 3 deployment patterns (employee-facing, workflow, service/public-facing)
2. **Runs a pre-flight checklist** — confirms licenses, BRE, OmniStudio, Einstein Agent User are in place before any work begins
3. **Qualifies the use case** — 9 structured discovery questions covering lifecycle stage, Gov Cloud, BRE, OmniStudio, Data 360, escalation, and compliance requirements
4. **Recommends an action set** — 11 actions with backing logic tags (STANDARD, FLOW, PROMPT_TEMPLATE) and rationale
5. **Handles advanced patterns** — multi-turn state management, authentication gates for service agents, human handoff via Case creation Flow
6. **Produces an Agent Spec** — complete spec requiring explicit approval before any build work starts
7. **Builds the agent** — NGA Agent Script (via `developing-agentforce` skill) or legacy Agent Builder guidance with step-by-step instructions
8. **Wires in Data 360** — D360 for Public Sector semantic model mapping, Identity Resolution, Calculated Insights, delegating to `sf-datacloud-act` / `sf-pi` `/sf-data360`

---

## 3 Required SKUs

Before starting any work, confirm these are licensed:

| SKU | Purpose | Required |
|---|---|---|
| **PSC Grantmaking** | Data model (FundingOpportunity, FundingRequest, etc.) | Always |
| **Agentforce for Public Sector** | Agent features in PSC context | Always |
| **Data 360 for Public Sector** | Constituent profiles, Identity Resolution, Calculated Insights | Only if D360 features needed |

---

## Key Features

- **11-action catalog** — 7 original PSC defaults + 4 new: Summarize Application (Prompt Template), Score Application (Prompt Template), Notify Applicant, Check for Duplicates
- **3 backing logic types** — STANDARD (BRE/DocGen/OmniScript), FLOW, PROMPT_TEMPLATE
- **BRE-aware** — guides BRE rule set configuration for eligibility OR builds a Flow fallback when BRE isn't configured
- **Verification Gate** — authentication pattern for public-facing service agents before showing personal data
- **Multi-turn state** — `@variables` (NGA) and Conversation Variables (legacy) guidance for "Guide Application Completion"
- **Async escalation** — Case creation Flow + Grant Review Queue routing for complex cases, appeals, fraud flags
- **Data 360 semantic model** — maps PSC objects to D360 DMOs (FundingRequest → Case DMO, FundingAward → Benefit DMO, etc.)
- **Calculated Insights** — LifetimeGrants, ComplianceRate, VulnerabilityIndex patterns for grantmaking
- **Decision Explainer** — audit trail guidance for compliance-heavy customers

---

## SI Partners for Grantmaking Implementations

- [Agile Cloud Consulting](https://www.agilecloudconsulting.com) — PSC Grantmaking, Federal grants, OmniStudio
- [Huron Consulting](https://www.huronconsultinggroup.com) — Higher ed / nonprofit grantmaking, compliance-heavy
- [Slalom](https://www.slalom.com) — Digital government, Experience Cloud, large agency builds
- [ImagineCRM](https://www.imaginecrm.com) — Nonprofit and community foundation grantmaking

---

## Compatibility

- PSC Grantmaking license required
- Salesforce CLI (`sf`) required for NGA path
- API v62+
- Gov Cloud: legacy agent path only (NGA not FedRAMP-authorized)

---

## License

Apache-2.0 — see [LICENSE](LICENSE)
