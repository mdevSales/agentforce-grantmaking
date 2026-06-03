# Verification Gate: Authentication for Public-Facing Agents

Before a service/public-facing agent shows a constituent their application status, award information, or any personal data, it **must** verify who the constituent is. Skipping this is a security and compliance failure.

---

## Why This Is Required

Grant applications contain personally identifiable information (PII): name, address, income, organization details. A service agent that returns application status without identity verification could expose one constituent's data to another. This is especially critical in government/public sector contexts.

---

## NGA Agent Script: `when` Guard Block

The Verification Gate uses a `when` guard on the `@verifiedIdentity` variable. No constituent-data action can run until this variable is set to `"true"`.

### Setup

```
// Declare the variable
variables {
  @verifiedIdentity: String = "false"
  @verifiedContactId: String = ""
}

// Verification subagent (runs first, before any other subagent)
subagent IdentityVerificationSubagent {
  description: "Verifies the identity of the constituent before allowing access to personal data."

  action VerifyIdentity {
    type: Flow
    flowApiName: "VerifyConstituentIdentity"
    inputs {
      enteredEmail: {userInput.email}
      enteredDateOfBirth: {userInput.dateOfBirth}
      enteredLastFourSSN: {userInput.lastFourSSN}
    }
    outputs {
      verified -> @verifiedIdentity
      contactId -> @verifiedContactId
    }
  }

  when @verifiedIdentity == "false" {
    response "I wasn't able to verify your identity. Please check the information you provided and try again, or contact our office for assistance."
    end
  }
}
```

### Guard pattern on protected actions

```
// Track Application Status — gated
action GetApplicationStatus {
  // Guard: only runs if identity is verified
  when @verifiedIdentity != "true" {
    response "I need to verify your identity before I can show you your application status."
    transfer to IdentityVerificationSubagent
  }

  type: Flow
  flowApiName: "TrackApplicationStatus"
  inputs {
    contactId: @verifiedContactId
  }
}
```

### Routing — verify first, always

In the orchestrator, route to IdentityVerificationSubagent before any data-access subagent:

```
agent PublicGrantsAgent {
  routing {
    // Identity verification must happen before any data access
    when {userInput} matches ["my application", "my status", "my award", "my account"] {
      when @verifiedIdentity != "true" {
        route to IdentityVerificationSubagent
      }
      route to StatusSubagent
    }

    // Eligibility checking and FAQ are public — no auth gate needed
    when {userInput} matches ["am I eligible", "what programs", "how do I apply"] {
      route to EligibilitySubagent
    }
  }
}
```

---

## VerifyConstituentIdentity Flow Design

**Inputs:**
- `enteredEmail` (Text)
- `enteredDateOfBirth` (Date)
- `enteredLastFourSSN` (Text) — or whichever verification fields your agency uses

**Logic:**
1. Get Records: Contact WHERE Email = :enteredEmail
2. Decision: Contact found?
3. If found: check dateOfBirth match AND (last4SSN match OR other verification field)
4. If both match: output `verified = "true"`, `contactId = contact.Id`
5. If no match: output `verified = "false"`, `contactId = ""`
6. Log the verification attempt (for audit)

**Output variables:**
- `verified` (Text) — "true" or "false"
- `contactId` (Id)

**Security note:** Do not return the reason for failure (don't say "email not found" vs "DOB doesn't match" — say "unable to verify" generically to prevent enumeration).

---

## Legacy Agent Builder: Topic Precondition Flow

Legacy agents cannot use `when` guard blocks. Instead, configure a precondition Flow on the status-lookup topic.

### Setup

1. In Agent Builder → Topics → select your status-lookup topic
2. Click "Preconditions"
3. Add precondition: Flow → `CheckConstituentVerificationStatus`

**CheckConstituentVerificationStatus Flow:**
- Input: conversation variable `applicantId`
- Get Records: Contact WHERE Id = :applicantId AND VerifiedIdentity__c = true
- Output: `isVerified` (Boolean)
- Decision: if `isVerified = false` → route to verification dialog first

4. If precondition fails, redirect to the Identity Verification dialog
5. After verification, set conversation variable `applicantId` = verified ContactId

### Alternative: Custom field on Contact
Add `VerifiedIdentity__c` (Checkbox) to Contact. The verification Flow sets this to `true` when the constituent successfully verifies. The precondition Flow checks this field.

**Clear this field** when the session ends (add a "session end" Flow trigger if using Chat).

---

## What Needs Verification (Access Control Matrix)

| Action | Needs Auth Gate | Reason |
|---|---|---|
| Check Eligibility | No | Public information — no personal data returned |
| Recommend Grant Programs | No | Public information |
| Answer Program Questions (FAQ) | No | Public information |
| Guide Application Completion | Yes (after initial contact creation) | PII collected and stored |
| Track Application Status | Yes | Returns personal application data |
| Support Compliance Reporting | Yes | Returns award and financial data |
| Check for Duplicate Applications | Yes (internal only) | Not exposed to constituents |

---

## Experience Cloud Context

If the service agent is deployed on Experience Cloud and the constituent logs into the community site, their session already provides authentication. In this case:

- The `User.ContactId` from the community session is the verified constituent
- You may be able to skip the manual verification step and set `@verifiedContactId = {$User.ContactId}` directly
- Still validate that the ContactId has the Experience Cloud for Grantmaking permission set before allowing data access
