# Investigation Report: Stopping an Azure AI Foundry Agent (Temporary Suspension)

**Prepared for:** Infrastructure Team  
**Date:** May 2026  
**API Version Referenced:** `2026-01-15-preview`

---

## Executive Summary

This report investigates all available methods to **temporarily stop or suspend** an Azure AI Foundry agent — without deleting it — in response to a security incident scenario. The goal is to render a specific agent inaccessible (e.g., from M365 Copilot) while preserving the ability to restart it later.

**Key finding:** Due to a platform architecture shift away from standalone deployment objects, the available methods differ depending on the agent type (Prompt/Workflow vs. Hosted) and whether the agent has been published as an Agent Application. A true per-agent "stop without delete" is currently only possible in two scenarios: (1) a published Agent Application with a deployment, or (2) a Hosted agent. For unpublished prompt/workflow agents, access must be blocked at a higher scope.

---

## Background: Agent Architecture (Current State)

As of the `2026-01-15-preview` API, the publishing model has been restructured:

| Concept | Description |
|---|---|
| **Agent (version)** | An immutable snapshot of prompt, model, and tools inside a Foundry project. Development-scope only. |
| **Agent Application** | A published ARM resource wrapping an agent version with its own endpoint, RBAC scope, and identity. Required for M365/Teams/external access. |
| **Deployment** | A child resource of an Agent Application representing a running instance. Supports `start` / `stop` lifecycle. |

> **Note:** The "deployment" concept referenced in the `az cognitiveservices agent stop` CLI command applies specifically to **Hosted agents** and to **Agent Application deployments**. For unpublished prompt/workflow agents inside a project, there is no standalone deployment object to stop.

---

## Method 1: REST API — Stop Agent Deployment (Published Agent Applications)

### Applicability
- ✅ Published Prompt/Workflow agents (deployed via Agent Application)  
- ✅ Published Hosted agents (deployed via Agent Application)  
- ❌ Unpublished / development-scope agents

### Endpoint

```http
POST https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CognitiveServices/accounts/{accountName}/projects/{projectName}/applications/{appName}/agentDeployments/{deploymentName}/stop?api-version=2026-01-15-preview
Authorization: Bearer <ARM token>
```

### How to Acquire the Token

```bash
az login
az account get-access-token --resource https://management.azure.com --query accessToken -o tsv
```

### Sample Request

```http
POST https://management.azure.com/subscriptions/00000000-1111-2222-3333-444444444444/resourceGroups/my-rg/providers/Microsoft.CognitiveServices/accounts/my-account/projects/my-project/applications/agent-app-1/agentDeployments/deployment-1/stop?api-version=2026-01-15-preview
Authorization: Bearer <token>
```

### Expected Response

```
HTTP 200 OK
```

The deployment `state` field will transition: `Running → Stopping → Stopped`.

### Restart (Resume) the Same Deployment

```http
POST .../agentDeployments/{deploymentName}/start?api-version=2026-01-15-preview
Authorization: Bearer <ARM token>
```

### Verify Current State

```http
GET .../agentDeployments/{deploymentName}?api-version=2026-01-15-preview
```

Inspect the `state` property in the response. Valid values: `Starting`, `Running`, `Stopping`, `Failed`, `Deleting`, `Deleted`, `Updating`.

---

## Method 2: Azure CLI — Stop Agent Deployment (Hosted Agents)

### Applicability
- ✅ Hosted agents (containerized, running on Foundry Agent Service)  
- ❌ Prompt/Workflow agents (the CLI `agent stop` command targets hosted agent versions)

### Prerequisites

```bash
az extension add --name cognitiveservices --upgrade  # Azure CLI >= 2.80 required
```

### Stop Command

```bash
az cognitiveservices agent stop \
    --account-name "<account-name>" \
    --project-name "<project-name>" \
    --name "<agent-name>" \
    --agent-version 1
```

### State Transitions

```
Running → Stopping → Stopped   (success)
Running → Stopping → Running   (error / rollback)
```

### Restart Command

```bash
az cognitiveservices agent start \
    --account-name "<account-name>" \
    --project-name "<project-name>" \
    --name "<agent-name>" \
    --agent-version 1 \
    --min-replicas 1 \
    --max-replicas 2
```

> **Important:** The stopped agent version is preserved and can be restarted at any time. No data or configuration is lost.

---

## Method 3: Disable Agent Application via `isEnabled` Flag (Prompt/Workflow/Hosted)

### Applicability
- ✅ All published Agent Applications (Prompt, Workflow, Hosted)  
- ❌ Unpublished agents

### Overview

The `AgentApplication` ARM resource exposes an `isEnabled` boolean property. Setting it to `false` disables the application without stopping the underlying deployment or deleting any resource. This is a lightweight toggle.

### REST API — Disable

```http
PATCH https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CognitiveServices/accounts/{accountName}/projects/{projectName}/applications/{applicationName}?api-version=2026-01-15-preview
Authorization: Bearer <ARM token>
Content-Type: application/json

{
  "properties": {
    "isEnabled": false
  }
}
```

### REST API — Re-enable

```http
PATCH .../applications/{applicationName}?api-version=2026-01-15-preview
...
{
  "properties": {
    "isEnabled": true
  }
}
```

> **Effect:** Callers attempting to invoke the endpoint will receive an error (endpoint is inactive). The agent, its versions, and the deployment are untouched.

---

## Method 4: RBAC Removal — Block Access Without Stopping

This is the **most flexible and immediate** method and works across all agent types. Per the updated investigation, this is the recommended approach when the deployment concept is absent.

### Scope Options

| Scope | Effect | Granularity |
|---|---|---|
| **Agent Application resource** | Blocks invocation of a single published agent | Per-agent ✅ |
| **Foundry Project** | Blocks access to all agents in the project | Project-level ⚠️ |
| **Foundry Account (resource)** | Blocks all projects and agents under the account | Account-level ⚠️ |

### Per-Agent RBAC Removal (Recommended for Security Incidents)

When a specific agent is published as an Agent Application, invocation requires the `Azure AI User` role (or a custom role with `Microsoft.CognitiveServices/accounts/projects/applications/invoke/action`) on the **Agent Application ARM resource**.

Removing this role assignment from all users/groups immediately blocks invocation without touching the agent itself.

#### Via Azure Portal
1. Navigate to the Agent Application resource in the Azure portal.
2. Open **Access Control (IAM)**.
3. Remove all `Azure AI User` role assignments for the affected identities.

#### Via Azure CLI

```bash
# List current role assignments on the Agent Application
az role assignment list \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>/applications/<app-name>"

# Remove a specific assignment
az role assignment delete \
  --assignee "<user-or-group-object-id>" \
  --role "Azure AI User" \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>/applications/<app-name>"
```

#### Via REST API

```http
DELETE https://management.azure.com/{scope}/providers/Microsoft.Authorization/roleAssignments/{roleAssignmentName}?api-version=2022-04-01
Authorization: Bearer <ARM token>
```

### Project-Level RBAC Removal

If the agent is not published (development-scope only) or if you need to cut off all access to a project:

```bash
az role assignment delete \
  --assignee "<identity>" \
  --role "Azure AI User" \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>"
```

> ⚠️ **Caveat:** This blocks **all agents** within the project, not just the affected one. Use Agent Application-scoped RBAC removal for surgical per-agent control.

---

## Method 5: Hosted Agent — Stop via Python SDK

### Applicability
- ✅ Hosted agents only

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project = AIProjectClient(
    endpoint="https://<account>.services.ai.azure.com/api/projects/<project>",
    credential=DefaultAzureCredential(),
    allow_preview=True,
)

# Stop the hosted agent version
project.agents.stop_version(agent_name="<agent-name>", agent_version="1")

# Restart later
project.agents.start_version(
    agent_name="<agent-name>",
    agent_version="1",
    min_replicas=1,
    max_replicas=2,
)
```

---

## Summary Comparison Table

| Method | Prompt/Workflow | Hosted | Scope | Reversible | Notes |
|---|:---:|:---:|---|:---:|---|
| **REST API deployment stop** | ✅ (if published) | ✅ (if published) | Per-deployment | ✅ | Requires Agent Application |
| **`az cognitiveservices agent stop`** | ❌ | ✅ | Per-agent version | ✅ | Hosted agents only |
| **`isEnabled: false` on Agent Application** | ✅ | ✅ | Per-Agent Application | ✅ | Lightweight toggle; deployment stays running |
| **RBAC removal (Agent Application scope)** | ✅ | ✅ | Per-Agent Application | ✅ | **Best for security incidents; per-agent precision** |
| **RBAC removal (Project scope)** | ✅ | ✅ | All agents in project | ✅ | Blunt; affects all agents in the project |
| **APIM policy** | ✅ | ✅ | Configurable | ✅ | Requires APIM in front of the endpoint |
| **Deletion** | ✅ | ✅ | Per-agent | ❌ | Irreversible at version level |

---

## Decision Guide for Security Incident Response

```
Is the agent published as an Agent Application?
│
├── YES ──► Is fast per-agent access revocation needed?
│           │
│           ├── YES ──► Remove Azure AI User role on the Agent Application resource (Method 4)
│           │           (Immediate effect; no downtime for other agents)
│           │
│           └── NO  ──► Use isEnabled: false (Method 3) or deployment stop (Method 1)
│
└── NO  ──► Agent is in development scope only (Foundry project)
            │
            ├── Hosted agent? ──► az cognitiveservices agent stop (Method 2)
            │
            └── Prompt/Workflow? ──► Remove RBAC at project level (Method 4, project scope)
                                     ⚠️ Affects all agents in the project
```

---

## Current Limitations & Observations

1. **No native "pause" for unpublished prompt/workflow agents.** An agent that has not been published as an Agent Application has no individual stop mechanism — only RBAC at the project scope or deletion are available.

2. **Deployment concept has shifted.** The `agentDeployments` resource now lives under an Agent Application (`/applications/{appName}/agentDeployments/`), not directly under the project. Teams still using legacy flows may need to migrate.

3. **Per-agent stop requires publishing.** To enable fine-grained start/stop per agent without affecting other agents, the agent must first be published as an Agent Application. This is the recommended production pattern.

4. **APIM as a complementary control layer.** Azure API Management (APIM) placed in front of the Foundry endpoint can provide policy-based blocking (e.g., return 403 for a specific agent app name) without touching the Azure resource itself. This is useful when IAM changes have propagation latency concerns.

5. **M365 Copilot access is controlled via the Agent Application + Activity Protocol.** Removing `Azure AI User` RBAC on the Agent Application resource will prevent M365 Copilot from invoking the agent, as Copilot uses the Activity Protocol through the Agent Application endpoint.

---

## Recommended Action Plan (Security Incident)

| Priority | Action | Method | Time to Effect |
|---|---|---|---|
| **Immediate** | Remove `Azure AI User` role on the Agent Application | Method 4 (RBAC) | Near-immediate (~minutes) |
| **Short-term** | Set `isEnabled: false` on Agent Application | Method 3 | Immediate API call |
| **If hosted** | Run `az cognitiveservices agent stop` | Method 2 | Minutes (state transition) |
| **Review** | Audit all role assignments using `az role assignment list` | — | — |
| **Restore** | Re-add RBAC / set `isEnabled: true` / run `agent start` | Reverse of above | Minutes |

---

## References

- [Agent Deployments – Stop REST API](https://learn.microsoft.com/en-us/rest/api/aifoundry/accountmanagement/agent-deployments/stop?view=rest-aifoundry-accountmanagement-2026-01-15-preview)
- [Publish your agent as an Agent Application](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/agent-applications)
- [Manage hosted agent lifecycle](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/manage-hosted-agent)
- [Role-based access control for Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/concepts/rbac-foundry)
- [High availability and resiliency for Foundry Agent Service](https://learn.microsoft.com/en-us/azure/foundry/how-to/high-availability-resiliency)
