# OmniStudio Dependency Guide

OmniStudio (formerly Vlocity) is the standard PSC component for building rich, multi-step application forms. If it's not licensed, the standard PSC application intake form won't exist and you need an alternative.

---

## Checking OmniStudio Status

```bash
# Check if OmniStudio is installed
sf package installed list --target-org YOUR_ORG_ALIAS | grep -i "omni\|vlocity"

# Check for OmniStudio objects
sf data query \
  --query "SELECT QualifiedApiName FROM EntityDefinition WHERE QualifiedApiName LIKE 'OmniScript%'" \
  --target-org YOUR_ORG_ALIAS 2>/dev/null
```

---

## Path A: OmniStudio Licensed

The standard PSC application intake form is an OmniScript — a guided, multi-step web form that integrates directly with FundingRequest creation.

**What exists out-of-the-box:**
- Grant Application OmniScript (wizard-style form with validation)
- Document upload integration
- Mobile-responsive layout
- Experience Cloud embedding via FlexCard

**Agent integration:**
The "Guide Application Completion" action (Action 3) can launch the OmniScript via a deep link or embedded Experience Cloud component, then track progress using the OmniScript's built-in state management.

For OmniScript development and configuration, delegate to `sf-industry-commoncore-omniscript` skill.

---

## Path B: OmniStudio Not Licensed — Alternative Approaches

### Option 1: Flow Screen (Recommended for simple intake)

Best for: straightforward applications with < 10 fields per step.

**How to build:**
1. Create a Screen Flow: `GrantApplicationIntake`
2. Add Screen elements for each step:
   - Screen 1: Personal/Organization Info
   - Screen 2: Program selection (from FundingOpportunity records)
   - Screen 3: Application narrative
   - Screen 4: Document upload (use `lightning:fileUpload` component)
   - Screen 5: Review and submit
3. Between screens: Get Records, Decision, Assignment elements to validate and save
4. Final step: Create FundingRequest record, update status to Submitted

**Deploy as an Experience Cloud page** via Digital Experiences → Pages → new Flow-embedded page.

**Limitation:** Less visual polish than OmniScript. No built-in resume/save-progress for partial applications.

### Option 2: Experience Cloud LWC (Recommended for complex intake)

Best for: complex forms with conditional sections, calculated fields, document requirements by program type.

**How to build:**
1. Create a custom LWC: `grantApplicationForm`
2. Use `@wire(getRecord)` to load the FundingOpportunity details
3. Render conditional sections based on program requirements
4. On submit: call `createRecord` to create FundingRequest, then call `uploadFile` for documents
5. Deploy to Experience Cloud page

**Delegate to `sf-lwc` skill** for LWC development.

---

## OmniStudio Delegation

If OmniStudio questions arise during the build, delegate to:
- `sf-industry-commoncore-omniscript` — OmniScript authoring and configuration
- `sf-industry-commoncore-flexcard` — FlexCard for displaying application status in Experience Cloud
- `sf-industry-commoncore-omnistudio-analyze` — analyzing existing OmniStudio deployments

---

## Agent Impact of OmniStudio Absence

| Agent action | OmniStudio present | OmniStudio absent |
|---|---|---|
| Guide Application Completion | Launch OmniScript (deep link or embedded) | Step-by-step conversational Flow |
| Validate Required Documents | OmniScript checklist built-in | Separate document validation Flow |
| Application form rendering | Rich wizard UI | Conversational turns or Flow screen |

The agent can still guide application completion conversationally (Action 3 multi-turn state pattern) without OmniStudio. The experience is less visual but functionally equivalent for simple programs.
