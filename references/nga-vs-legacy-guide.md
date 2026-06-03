# NGA vs Legacy Agent Builder Decision Guide

## Quick Decision

```
Is this a Gov Cloud (FedRAMP) org?
  YES → Legacy agent (Agent Builder UI). Stop. NGA is not FedRAMP-authorized.
  NO  → Does the partner want version control, CLI deployment, and deterministic behavior?
          YES → NGA Agent Script (developing-agentforce skill)
          NO  → Legacy agent (Agent Builder UI)
```

---

## Feature Comparison

| Feature | NGA (Agent Script) | Legacy (Agent Builder) |
|---|---|---|
| Authoring tool | CLI + `.agent` file in version control | Agent Builder UI in Setup |
| Deployment | `sf agent generate/preview/publish` | Manual UI configuration |
| Version control | Git — full diff, PR review, rollback | No native version control |
| Determinism | Deterministic routing via `when` blocks | LLM-driven topic routing (less predictable) |
| Multi-turn state | `@variables` native to the DSL | Conversation Variables (more manual) |
| Authentication gate | `when` guard blocks — clean, readable | Topic precondition Flows (more complex) |
| Subagents | Native in Agent Script | Bot dialogs (equivalent, different syntax) |
| Prompt Template actions | Supported | Supported |
| Flow actions | Supported | Supported |
| ADL knowledge grounding | Supported via `knowledge:` block | Supported via Knowledge topic |
| Gov Cloud support | Not FedRAMP-authorized | Supported |
| Debugging | `--use-live-actions` preview + CLI traces | Conversation Simulator in UI |
| Partner technical skill required | Moderate (CLI comfort) | Low (UI-based) |

---

## NGA Path: Agent Script

**Trigger this path when:**
- Gov Cloud = No
- Partner is comfortable with CLI and wants version-controlled agents
- Agent has complex routing logic that benefits from deterministic `when` blocks
- Multi-turn state management is needed (cleaner with `@variables`)
- Partner wants to deploy via CI/CD pipeline

**Delegate to:** `developing-agentforce` skill for all NGA work.

**Key Agent Script files:**
- `.agent` file — the main agent definition (subagents, actions, routing, variables)
- `.aiAuthoringBundle/` — metadata bundle for deployment
- Validate: `sf agent validate --file my-agent.agent`
- Preview: `sf agent preview --file my-agent.agent --use-live-actions`
- Publish: `sf agent publish --target-org YOUR_ALIAS`

---

## Legacy Path: Agent Builder

**Trigger this path when:**
- Gov Cloud = Yes (mandatory)
- Partner prefers UI-based configuration
- Simple agent with straightforward topic routing
- Time constraints favor UI speed over code rigor

**Steps:**
1. Setup → Einstein Bots → New Bot
2. Select "Einstein Bot" (service agent) or "Einstein Agent" (employee)
3. Configure Bot Details: name, language, greeting
4. Create Topics matching the subagents in your Agent Spec
5. For each Topic: add Actions (Flows, invocable actions, Prompt Templates)
6. Configure Conversation Variables (Settings tab → Variables)
7. Add Topic precondition Flows for authentication gate
8. Configure Escalation dialogs
9. Test via Conversation Simulator
10. Deploy: Bot → Activate → Embed in Experience Cloud or Salesforce console

---

## Gov Cloud Detection

To detect whether an org is Gov Cloud:
```bash
sf org display --target-org YOUR_ALIAS --json | jq '.result.instanceUrl'
```

Gov Cloud instance URLs typically contain:
- `.mil` domain
- `government.` subdomain
- `usgovcloudapi.net` suffix
- Explicit `gov` in the instance URL

When in doubt, ask the partner directly: "Is this org on a FedRAMP-authorized instance?"

---

## Migrating from Legacy to NGA (Future State)

If a customer builds on legacy now and wants to migrate to NGA later:
1. Each Bot Topic → becomes a subagent in Agent Script
2. Each Bot Action → becomes an `action` block (same Flow/Prompt Template references)
3. Conversation Variables → become `@variables`
4. Topic precondition Flows → become `when` guard blocks
5. The Flows and Prompt Templates themselves do NOT change — only the orchestration layer changes

This migration is a rewrite of the `.agent` file, not a data migration. The underlying PSC Flows, Prompt Templates, and records remain unchanged.
