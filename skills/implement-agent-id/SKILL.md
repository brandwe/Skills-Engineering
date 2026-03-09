---
name: implement-agent-id
description: Guide for integrating with Microsoft Entra Agent Identity APIs (Graph beta). Covers authentication, blueprint creation, agent identity provisioning, sponsors, permissions, and known pitfalls. Use when implementing Entra Agent IDs, Agent Identity Blueprints, AgentIdentityBlueprintPrincipal, or working with the Graph beta Agent Identity endpoints.
---

# Integrating with Microsoft Entra Agent Identity APIs

## Overview

Microsoft Entra Agent Identity (preview, Nov 2025) provides a new identity primitive for AI agents in Microsoft Entra ID. It creates OAuth2-capable identities (service principals) that represent individual agent instances, organized under an "Agent Identity Blueprint" (application registration).

**Conceptual Model:**
```
Agent Identity Blueprint (application)     ← one per agent "kind" or project
  └─ AgentIdentityBlueprintPrincipal (SP)  ← must be created explicitly
      ├─ Agent Identity (SP): agent-1      ← one per agent instance
      ├─ Agent Identity (SP): agent-2
      └─ Agent Identity (SP): agent-3
```

**Graph beta API base:** `https://graph.microsoft.com/beta`

---

## Critical Pitfalls (Read First)

### 1. Azure CLI Tokens Are Rejected

**Problem:** Azure CLI tokens always include the `Directory.AccessAsUser.All` delegated permission. The Agent Identity APIs **explicitly reject** any token containing this permission, returning a generic 403.

**Solution:** You MUST use a dedicated app registration with `client_credentials` flow:

```python
from azure.identity import ClientSecretCredential

credential = ClientSecretCredential(
    tenant_id="<tenant-id>",
    client_id="<app-client-id>",
    client_secret="<app-secret>",
)
token = credential.get_token("https://graph.microsoft.com/.default")
```

**DO NOT** use `DefaultAzureCredential` or `AzureCliCredential` — they will produce tokens with `Directory.AccessAsUser.All` and every Agent Identity API call will fail with 403.

Auto-provisioning the app registration via `az ad app create` is the recommended approach. See the reference implementation in [microsoft/aim-foundry-poc](https://github.com/microsoft/aim-foundry-poc) at `scripts/create-entra-agent-ids.py`.

### 2. Sponsors Are Required

**Problem:** Both Blueprint and Agent Identity creation require a `sponsors@odata.bind` field. Without it, you get: `400: No sponsor specified. Please provide at least one sponsor.`

**Rules:**
- Sponsors must be **User** references — ServicePrincipals are NOT valid
- Use the `/users/{objectId}` URL format (not `/directoryObjects/` or `/servicePrincipals/`)
- Since you're using `client_credentials` (no user context), you CANNOT use `GET /me` to get the user ID. Use `az ad signed-in-user show --query id -o tsv` instead.

```python
# Get sponsor user ID (az CLI has the user's auth context)
result = subprocess.run(
    ["az", "ad", "signed-in-user", "show", "--query", "id", "-o", "tsv"],
    capture_output=True, text=True,
)
user_id = result.stdout.strip()

# Add to any Blueprint or Agent Identity creation body
body["sponsors@odata.bind"] = [
    f"https://graph.microsoft.com/beta/users/{user_id}"
]
```

### 3. BlueprintPrincipal Must Be Created Separately

**Problem:** Creating a Blueprint (`POST /applications`) does NOT auto-create its BlueprintPrincipal (SP). Without the BlueprintPrincipal, all Agent Identity creation fails with: `400: The Agent Blueprint Principal for the Agent Blueprint does not exist.`

**Solution:** Always create the BlueprintPrincipal immediately after the Blueprint:

```python
# Step 1: Create Blueprint
blueprint_body = {
    "@odata.type": "Microsoft.Graph.AgentIdentityBlueprint",
    "displayName": "My Agent Blueprint",
    "sponsors@odata.bind": [f"https://graph.microsoft.com/beta/users/{user_id}"],
}
resp = requests.post(f"{GRAPH_BASE}/applications", headers=headers, json=blueprint_body)
app_id = resp.json()["appId"]

# Step 2: Create BlueprintPrincipal (REQUIRED — not auto-created)
sp_body = {
    "@odata.type": "Microsoft.Graph.AgentIdentityBlueprintPrincipal",
    "appId": app_id,
}
requests.post(f"{GRAPH_BASE}/servicePrincipals", headers=headers, json=sp_body)
```

**Also important:** If you're implementing idempotent scripts that skip Blueprint creation when it already exists, you MUST check for and create the BlueprintPrincipal on the skip path too. A previous run may have created the Blueprint but crashed before creating the SP.

### 4. Permission Propagation Takes 30-120+ Seconds

After `az ad app permission admin-consent`, newly-granted Agent Identity permissions don't appear in tokens immediately. The token endpoint serves cached claims.

**Solution:** Retry with fresh tokens:

```python
for attempt in range(5):
    token = credential.get_token("https://graph.microsoft.com/.default")
    # Try the actual operation
    resp = requests.post(url, headers=auth_header(token), json=body)
    if resp.status_code == 403:
        wait = 20 * (attempt + 1)
        time.sleep(wait)
        continue
    break
```

Key insight: `credential.get_token()` returns cached tokens. For `ClientSecretCredential`, the cache is based on token lifetime (usually 1hr). But Entra's token endpoint itself may serve tokens with stale claims for 30-120s after a permission change. The retry loop handles this.

---

## Required Permissions

### Minimum for Blueprint + Agent Identity Creation

There are **18 Agent Identity-specific** Graph application permissions. They can be discovered dynamically:

```bash
az ad sp show --id 00000003-0000-0000-c000-000000000000 \
  --query "appRoles[?contains(value, 'AgentIdentity')].{id:id, value:value}" -o json
```

**Core permissions needed:**
| Permission | Purpose |
|-----------|---------|
| `Application.ReadWrite.All` | Read/write applications (for Blueprint CRUD) |
| `AgentIdentityBlueprint.Create` | Create new Blueprints |
| `AgentIdentityBlueprint.ReadWrite.All` | Read/update Blueprints |
| `AgentIdentityBlueprintPrincipal.Create` | Create BlueprintPrincipals |
| `AgentIdentity.Create.All` | Create Agent Identities |
| `AgentIdentity.ReadWrite.All` | Read/update Agent Identities |

**Microsoft Graph API ID** (constant across all tenants): `00000003-0000-0000-c000-000000000000`

**Application.ReadWrite.All role ID**: `1bfefb4e-e0b5-418b-a88f-73c46d2cc8e9`

In practice, granting all 18 Agent Identity permissions plus `Application.ReadWrite.All` is the safest approach — the granular permission set is underdocumented and it's unclear which exact subset is needed for which operations.

### Admin Consent

All these are **Application permissions** (not delegated), so they require tenant admin consent:

```bash
az ad app permission admin-consent --id <client-id>
```

Admin consent may fail with 404 if the service principal hasn't replicated yet. Retry with 10-40s backoff:

```python
for attempt in range(4):
    wait = 10 * (attempt + 1)
    time.sleep(wait)
    rc, _, err = run_az(["ad", "app", "permission", "admin-consent", "--id", client_id])
    if rc == 0:
        break
```

---

## API Reference

### Create Agent Identity Blueprint

```
POST https://graph.microsoft.com/beta/applications
```

```json
{
    "@odata.type": "Microsoft.Graph.AgentIdentityBlueprint",
    "displayName": "My Agent Blueprint",
    "description": "Optional description",
    "sponsors@odata.bind": [
        "https://graph.microsoft.com/beta/users/{user-object-id}"
    ]
}
```

Returns: Application object with `appId` (GUID) and `id` (object ID).

### Create BlueprintPrincipal

```
POST https://graph.microsoft.com/beta/servicePrincipals
```

```json
{
    "@odata.type": "Microsoft.Graph.AgentIdentityBlueprintPrincipal",
    "appId": "{blueprint-appId-from-step-1}"
}
```

### Create Agent Identity

```
POST https://graph.microsoft.com/beta/servicePrincipals
```

```json
{
    "@odata.type": "Microsoft.Graph.AgentIdentity",
    "displayName": "my-agent-instance",
    "agentIdentityBlueprintId": "{blueprint-appId}",
    "sponsors@odata.bind": [
        "https://graph.microsoft.com/beta/users/{user-object-id}"
    ]
}
```

Returns: ServicePrincipal object. The `appId` or `id` field is the Entra Agent ID (UUID).

### Find Existing Blueprint

```
GET https://graph.microsoft.com/beta/applications?$filter=displayName eq 'My Agent Blueprint'
```

### Find Existing Agent Identity

```
GET https://graph.microsoft.com/beta/servicePrincipals?$filter=displayName eq 'my-agent-instance'
```

### Check BlueprintPrincipal Exists

```
GET https://graph.microsoft.com/beta/servicePrincipals?$filter=appId eq '{blueprint-appId}'
```

---

## Complete Integration Sequence

The correct order of operations:

1. **Create dedicated app registration** (`az ad app create`)
2. **Create its service principal** (`az ad sp create --id <appId>`)
3. **Add permissions** (`az ad app permission add --id <appId> --api 00000003-... --api-permissions <id>=Role <id>=Role ...`)
   - Note: Each `<id>=Role` must be a **separate argument** to `--api-permissions`, not a joined string
4. **Grant admin consent** (`az ad app permission admin-consent --id <appId>`) — retry with backoff
5. **Wait 30s** for permission propagation to token endpoint
6. **Acquire token** via `ClientSecretCredential` with `client_credentials` flow
7. **Verify permissions** by attempting a test blueprint creation, retry with fresh tokens if 403
8. **Get sponsor user ID** via `az ad signed-in-user show --query id -o tsv`
9. **Create Blueprint** (`POST /applications` with `AgentIdentityBlueprint` type + sponsors)
10. **Create BlueprintPrincipal** (`POST /servicePrincipals` with `AgentIdentityBlueprintPrincipal` type)
11. **Create Agent Identities** (`POST /servicePrincipals` with `AgentIdentity` type + sponsors + `agentIdentityBlueprintId`)

### Idempotency

All steps should be idempotent — check for existing resources before creating:
- Blueprint: filter `/applications` by `displayName`
- BlueprintPrincipal: filter `/servicePrincipals` by `appId`
- Agent Identity: filter `/servicePrincipals` by `displayName`

Agent Identities are **durable** — they should survive environment teardowns (infra destroy/recreate). Only delete them when decommissioning the project entirely.

---

## Reference Implementation

See [microsoft/aim-foundry-poc](https://github.com/microsoft/aim-foundry-poc) at `scripts/create-entra-agent-ids.py` for a battle-tested implementation that handles all of the above, including:
- Auto-provisioning the dedicated app registration via `az ad` CLI
- Dynamic discovery of all 18 Agent Identity permissions
- Admin consent with retry and SP propagation handling
- Token verification with actual blueprint creation probe
- Idempotent blueprint, BlueprintPrincipal, and agent identity creation
- Sponsor assignment from `az ad signed-in-user show`
- azd env integration for credential and ID storage

---

## Known Limitations (as of March 2026)

1. **Preview API only** — all endpoints are under `/beta`, not `/v1.0`
2. **Sponsors must be Users** — ServicePrincipals and Groups are not accepted
3. **`/me` endpoint unavailable** in `client_credentials` flow — must use CLI for user context
4. **No "quick start" permission bundle** — must discover and grant 18+ individual permissions
5. **BlueprintPrincipal not auto-created** — requires explicit `POST /servicePrincipals`
6. **Permission propagation delay** — 30-120s after admin consent before tokens include new claims
7. **`Directory.AccessAsUser.All` hard rejection** — makes Azure CLI tokens (the most common auth method) unusable
