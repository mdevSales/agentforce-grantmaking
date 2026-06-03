# Permission Sets & User Setup for Grantmaking Agents

All permission sets are assumed to be already created when PSC Grantmaking is installed. This guide covers assignment, not creation.

---

## Permission Set Matrix

### For Human Users

| Permission Set | API Name | Assigned To | Grants Access To |
|---|---|---|---|
| Grantmaking User | `GrantmakingUser` | Caseworkers, internal applicants | Read/write FundingRequest, FundingOpportunity |
| Grantmaking Manager | `GrantmakingManager` | Program officers, grants directors | Full CRUD + approve, award, all objects |
| Grantmaking External Reviewer | `GrantmakingExternalReviewer` | Panel reviewers | Read-only FundingRequest + write GrantApplicationReview |
| Experience Cloud for Grantmaking | `ExperienceCloudGrantmaking` | Constituents via portal | Limited portal access to their own applications |

### For Agent Users

| Permission Set | API Name | Assigned To | Purpose |
|---|---|---|---|
| Einstein Agent User | `EinsteinAgentUser` | All users who interact with the agent | Required for any user to see/use Agentforce |
| Einstein Agent Access | `EinsteinAgentAccess` | The agent's **integration user** | Required for the agent to execute actions |
| Agentforce for Public Sector | `AgentforcePublicSector` | All users + integration user | License-level access to Agentforce PS features |

---

## The Integration User

The agent runs actions as a **dedicated integration user** — not as the end user. This user needs:
1. A Salesforce license (minimum: Salesforce or Service Cloud)
2. `Einstein Agent Access` permission set
3. `Agentforce for Public Sector` permission set
4. Object-level permissions for all PSC objects the agent reads/writes

**Create the integration user:**
```bash
# Query existing users to find or verify the bot user
sf data query \
  --query "SELECT Id, Name, Username, IsActive FROM User WHERE Name LIKE '%Bot%' OR Name LIKE '%Agent%'" \
  --target-org YOUR_ORG_ALIAS
```

If no dedicated bot user exists, create one in Setup → Users → New User with profile "Salesforce" (or minimal license), then assign the perm sets below.

---

## Assigning Permission Sets (CLI)

Use the `sf-permissions` skill for guided assignment, or run directly:

```bash
# Find the permission set IDs
sf data query \
  --query "SELECT Id, Name FROM PermissionSet WHERE Name IN ('EinsteinAgentUser', 'EinsteinAgentAccess', 'AgentforcePublicSector', 'GrantmakingUser', 'GrantmakingManager')" \
  --target-org YOUR_ORG_ALIAS

# Assign Einstein Agent User to all relevant users (replace USER_ID)
sf data create record \
  --sobject PermissionSetAssignment \
  --values "AssigneeId=USER_ID PermissionSetId=PERM_SET_ID" \
  --target-org YOUR_ORG_ALIAS

# Assign Einstein Agent Access to the integration user
sf data create record \
  --sobject PermissionSetAssignment \
  --values "AssigneeId=BOT_USER_ID PermissionSetId=EINSTEIN_ACCESS_PERM_SET_ID" \
  --target-org YOUR_ORG_ALIAS
```

---

## Verifying Assignments

```bash
# Check perm set assignments for a specific user
sf data query \
  --query "SELECT PermissionSet.Name, Assignee.Name FROM PermissionSetAssignment WHERE AssigneeId = 'YOUR_USER_ID'" \
  --target-org YOUR_ORG_ALIAS

# Check all Einstein Agent Access assignments
sf data query \
  --query "SELECT Assignee.Name, Assignee.Username FROM PermissionSetAssignment WHERE PermissionSet.Name = 'EinsteinAgentAccess'" \
  --target-org YOUR_ORG_ALIAS
```

---

## Object-Level Permissions for the Integration User

The integration user's profile or permission set must grant access to:

| Object | Read | Create | Edit | Delete |
|---|---|---|---|---|
| FundingOpportunity | Yes | No | No | No |
| FundingRequest | Yes | Yes | Yes | No |
| GrantApplicationReview | Yes | Yes | Yes | No |
| FundingAward | Yes | No | No | No |
| Budget | Yes | No | No | No |
| Case (for escalations) | Yes | Yes | Yes | No |
| Contact | Yes | Yes | Yes | No |
| Account | Yes | No | Yes | No |
| ContentDocument | Yes | Yes | No | No |
| ContentDocumentLink | Yes | Yes | No | No |

**Add these via a custom permission set** on the integration user — do not expand its profile.

---

## Experience Cloud Guest User (for unauthenticated access)

If the service agent needs to serve unauthenticated visitors (before login):
- The Experience Cloud Guest User profile handles this
- Grant read access to FundingOpportunity only (public program info)
- Do NOT grant FundingRequest access to guest users — requires authentication first
- See [verification-gate-guide.md](verification-gate-guide.md) for the auth gate pattern

---

## Quick Preflight Check

```bash
ORG="YOUR_ORG_ALIAS"
BOT_USER="your-bot-user@example.com"

echo "=== Bot User Perm Sets ==="
sf data query \
  --query "SELECT PermissionSet.Name FROM PermissionSetAssignment WHERE Assignee.Username = '$BOT_USER'" \
  --target-org $ORG

echo "=== Einstein Agent Access Holders ==="
sf data query \
  --query "SELECT Assignee.Name FROM PermissionSetAssignment WHERE PermissionSet.Name = 'EinsteinAgentAccess'" \
  --target-org $ORG
```

Expected: integration user has `EinsteinAgentAccess` + `AgentforcePublicSector` + object-level access perm set.
