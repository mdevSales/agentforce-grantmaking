# Multi-Turn State Management

Multi-turn state is required for **Guide Application Completion** (Action 3) and any action that needs to persist progress across conversation turns.

---

## NGA Agent Script: @variables

Agent Script `@variables` are declared in the `.agent` file and persist for the lifetime of the conversation session.

### Declaration (at agent or subagent level)
```
variables {
  @applicationStep: String = "personal_info"
  @completedSections: String = ""
  @applicantId: String = ""
  @fundingOpportunityId: String = ""
  @currentDraftId: String = ""
}
```

### Reading a variable
```
when @applicationStep == "personal_info" {
  response "Let's start with your personal information. What is your full name?"
}

when @applicationStep == "org_info" {
  response "Now let's collect your organization details. What is your organization's legal name?"
}
```

### Writing a variable (via action output)
```
action SavePersonalInfo {
  type: Flow
  flowApiName: "SaveApplicantPersonalInfo"
  inputs {
    firstName: {userInput.firstName}
    lastName: {userInput.lastName}
    contactId: @applicantId
  }
  outputs {
    nextStep -> @applicationStep
    savedContactId -> @applicantId
    updatedSections -> @completedSections
  }
}
```

### Progress tracking pattern
```
action GetApplicationProgress {
  type: Flow
  flowApiName: "GetApplicationStepProgress"
  inputs {
    draftId: @currentDraftId
  }
  outputs {
    currentStep -> @applicationStep
    completedSections -> @completedSections
  }
}
```

### Full Guide Application Completion subagent pattern
```
subagent ApplicationSubagent {
  description: "Guides a constituent through completing a grant application step by step."

  variables {
    @applicationStep: String = "start"
    @completedSections: String = ""
    @applicantId: String = ""
    @currentDraftId: String = ""
  }

  when @applicationStep == "start" {
    action InitializeApplication {
      type: Flow
      flowApiName: "InitializeGrantApplication"
      inputs { fundingOpportunityId: @fundingOpportunityId }
      outputs {
        draftId -> @currentDraftId
        firstStep -> @applicationStep
        applicantId -> @applicantId
      }
    }
  }

  when @applicationStep == "personal_info" {
    response "Let's start with your personal information."
    // collect fields, call SavePersonalInfo flow, advance @applicationStep
  }

  when @applicationStep == "org_info" {
    response "Now let's collect your organization details."
    // collect fields, call SaveOrgInfo flow, advance @applicationStep
  }

  when @applicationStep == "narrative" {
    response "Please describe how you plan to use this grant."
    // collect narrative, call SaveNarrative flow, advance @applicationStep
  }

  when @applicationStep == "documents" {
    action CheckDocuments {
      type: Flow
      flowApiName: "ValidateRequiredDocuments"
      inputs { fundingRequestId: @currentDraftId }
    }
    // show missing docs, advance when complete
  }

  when @applicationStep == "review_submit" {
    action SubmitApplication {
      type: Flow
      flowApiName: "SubmitGrantApplication"
      inputs { fundingRequestId: @currentDraftId }
      outputs { confirmationNumber -> @submissionConfirmation }
    }
    response "Your application has been submitted! Confirmation number: {@submissionConfirmation}"
  }
}
```

---

## Legacy Agent Builder: Conversation Variables

Conversation Variables are set via Flow outputs and persist for the bot session.

### Step 1: Create variables in Agent Builder
1. Open Agent Builder → select your Bot → Settings tab → Variables
2. Click "New Variable"
3. Create these variables:

| Variable Name | Type | Default Value |
|---|---|---|
| `applicationStep` | Text | `start` |
| `completedSections` | Text | *(empty)* |
| `applicantId` | Text | *(empty)* |
| `currentDraftId` | Text | *(empty)* |

### Step 2: Initialize in the welcome message
In the Welcome Message dialog node, add a Flow Action before the greeting:
- Flow: `InitializeConversationState`
- Map output `defaultStep` → conversation variable `applicationStep`

Always initialize variables before first use — uninitialized variables return empty string which can cause unexpected `when` branches.

### Step 3: Set variable from Flow output
In each step's Bot Dialog → add a Flow element:
1. Add Flow element: `SavePersonalInfo`
2. Map input: `applicantId` ← conversation variable `applicantId`
3. Map output: `nextStep` → conversation variable `applicationStep`
4. Add output to conversation variable `completedSections` (append comma-delimited)

### Step 4: Read variable in a condition
In a Bot Dialog Decision node:
- Condition: `{!$Bot.applicationStep}` equals `personal_info`
- Route to the Personal Info dialog branch

### Key tips for legacy variable management
- **Initialize to a default** — never leave a variable unset when first used
- **Use Set Variable element** in the Flow before returning to the agent, not after — the agent reads variables as soon as the Flow exits
- **Comma-delimited list pattern** for `completedSections`: append `"personal_info,"` then `"org_info,"` — check `CONTAINS({!$Bot.completedSections}, 'personal_info')` in conditions
- **Clear variables on abandon** — add a Flow that clears variables when the user says "start over" or "cancel"
- Variables persist for the session, not across sessions — a user returning the next day starts fresh
