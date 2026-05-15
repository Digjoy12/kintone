# Method 1 Investigation — Azure Logic Apps Kintone Connector

**Date:** May 2026  
**Purpose:** Evaluate feasibility, cost, and speed of using the native Azure Logic Apps Kintone connector as an AI agent tool in Azure AI Foundry.

---

## 1. Capability Mapping

### What the Native Connector Provides

The Kintone managed connector (shared between Power Automate and Azure Logic Apps) exposes the following prebuilt operations:

**Triggers (inbound events from Kintone):**
- When a record is added to an app
- When a record is updated
- When a record is deleted
- When a record comment is posted
- When a process management status is updated

**Actions (outbound calls to Kintone):**
- Add a new record to an app (`POST /k/v1/record.json`)
- Update a record in an app (`PUT /k/v1/record.json`)

### Gap Analysis Against Required Endpoints

| Endpoint | Method | Native Connector | Verdict |
|---|---|---|---|
| `/k/v1/record.json` | GET | ❌ Not exposed | **Gap** |
| `/k/v1/record.json` | POST | ✅ "Add a new record" | Covered |
| `/k/v1/record.json` | PUT | ✅ "Update a record" | Covered |
| `/k/v1/file.json` | GET | ❌ Not exposed | **Gap** |
| `/k/v1/file.json` | POST | ❌ Not exposed | **Gap** |

**Summary:** The native connector covers only 2 of the 5 required operations. GET record retrieval and both file upload/download endpoints are entirely absent. Kintone's own developer documentation explicitly acknowledges this ceiling, advising engineers to use Power Automate's HTTP connector (a Premium connector) "to bypass these limitations" when they need endpoints or parameters outside the prebuilt set.

### Bridging the Gaps

To cover all five operations within a Logic App workflow, the recommended approach is:

- Use the native Kintone connector actions for `POST` and `PUT` record operations where convenient.
- Replace the missing operations with the **HTTP action** (built-in, free in Consumption plan), calling Kintone's REST API directly with the `X-Cybozu-API-Token` header. This gives full endpoint coverage including `/k/v1/record.json` (GET) and `/k/v1/file.json` (GET/POST).

This hybrid approach is fully supported by Logic Apps and does not require a custom connector. File uploads via `/k/v1/file.json POST` require a `multipart/form-data` body, which the HTTP action can construct using workflow variables and inline expressions, though it adds authoring complexity.

---

## 2. Authentication

### What the Native Connector Uses

The Kintone managed connector authenticates using **OAuth / Kintone domain user credentials** — the connection is established by signing in with a Kintone account (username and password for the subdomain). This means:

- The connector operates under a user identity, inheriting that user's App-level permissions.
- The connection is **not shareable**: if a flow is shared with another user, that person must re-establish their own connection.
- A maximum of 60 webhook notifications per minute per domain apply to trigger-based flows.

### App-Specific API Tokens

Kintone's API token system (`X-Cybozu-API-Token` header) is **not natively supported by the managed connector**. API tokens are scoped per App, allowing fine-grained permission control (e.g., read-only on App A, read-write on App B). This is the preferred authentication model for service-to-service integrations.

**To use API tokens in a Logic App**, you must bypass the managed connector and call Kintone's REST API through the HTTP action, passing the token in the request header:

```
X-Cybozu-API-Token: <your-app-api-token>
```

Multiple tokens can be comma-separated in a single header when the request involves Lookup or Related Record fields across multiple Apps. Tokens are stored securely as Logic App parameters or referenced from Azure Key Vault using a managed identity connection.

**Important scoping constraint:** API tokens are App-specific. A single workflow calling records from multiple Kintone Apps must manage multiple tokens — one per App. This is manageable but requires deliberate token lifecycle governance.

---

## 3. Foundry Integration — Exposing a Logic App as an Agent Tool

### Prerequisites

To appear in the Azure AI Foundry agent tool gallery, a Logic App workflow must meet all of the following criteria:

| Requirement | Detail |
|---|---|
| **Plan** | Consumption SKU only (Standard Logic Apps not yet supported) |
| **Trigger** | Must begin with a **Request (HTTP)** trigger |
| **Description** | Each workflow must include a clear natural-language description for agent reasoning |
| **Foundry Project** | An Azure AI Foundry project with GPT-4.1 or earlier deployed |

### Integration Path

There are two ways to expose the workflow as an agent tool:

**Option A — Foundry Portal (No-Code):**
1. Open the Foundry portal and navigate to the target agent.
2. In the Setup pane, select **Actions → Add**.
3. Choose **Azure Logic Apps** from the gallery.
4. Select from pre-authored Microsoft templates or import an existing workflow by pointing to the subscription/resource group where the Consumption Logic App lives.
5. Foundry automatically reads the workflow's Request trigger schema and constructs the tool definition the agent uses for tool selection.

**Option B — Programmatic (SDK):**
```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from user_logic_apps import AzureLogicAppTool, create_send_email_function

project_client = AIProjectClient(
    endpoint=os.environ["PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential()
)

logic_app_tool = AzureLogicAppTool(subscription_id, resource_group)
logic_app_tool.register_logic_app(logic_app_name, trigger_name)
# register_logic_app internally calls Workflows - List callback Url via LogicManagementClient
# to retrieve the SAS-signed callback URL

kintone_func = create_send_email_function(logic_app_tool, logic_app_name)
# Wrap as a FunctionTool and attach to agent ToolSet
```

### Authentication Between Foundry and Logic Apps

When an agent invokes the Logic App tool, Foundry sends a POST request to the workflow's callback URL. Two authentication modes are supported:

- **SAS-based (default):** The callback URL includes an embedded Shared Access Signature. SAS tokens can be scoped with a validity period and rotated as needed.
- **Microsoft Entra ID OAuth:** A stricter enterprise option where Logic Apps validates an OAuth bearer token issued by Entra ID before executing the workflow.

### Workflow Design Constraints

The Request trigger and its response schema become the tool's input/output contract. Do not modify or remove the Request trigger after the agent action is registered — doing so breaks the agent-workflow relationship without warning. Treat the trigger schema as an API contract; version it by creating new workflows for breaking changes.

---

## 4. Cost & Latency

### Billing Model — Consumption Plan

Logic Apps Consumption uses pure **pay-per-execution** pricing. There are no reserved instances and no idle costs.

| Component | Rate |
|---|---|
| Built-in actions (HTTP, conditions, loops, etc.) | First 4,000/month free per subscription; then **$0.000025 per execution** |
| Standard managed connectors | **$0.000125 per call** |
| Enterprise managed connectors | **$0.001 per call** |
| Data retention (run history) | **$0.12 per GB/month** |

**Kintone connector classification:** The Kintone managed connector is a **Standard** connector, billing at $0.000125 per call.

**Per-invocation cost estimate for a typical agent-triggered workflow:**

Assume a workflow triggered by a Foundry agent that performs: 1 Request trigger + 1 HTTP GET (built-in) to `/k/v1/record.json` + 1 Kintone PUT (standard connector) + 1 Response action (built-in):

| Step | Type | Cost |
|---|---|---|
| Request trigger (HTTP) | Built-in | ~$0.000025 |
| HTTP GET to Kintone | Built-in | ~$0.000025 |
| Kintone PUT (managed connector) | Standard | $0.000125 |
| Response action | Built-in | ~$0.000025 |
| **Total per invocation** | | **~$0.0002** |

At 10,000 agent-triggered invocations per month this equates to roughly **$2/month** from Logic Apps alone (LLM inference costs are separate).

**Billing caution:** Every execution is billed regardless of success or failure. A workflow that errors at step 2 of 5 still incurs charges for steps 1 and 2.

### Execution Latency

The Consumption plan runs in Azure's shared multitenant infrastructure. Key latency characteristics:

**Warm invocation (recent execution):**  
End-to-end latency from agent POST to Kintone API response is typically **1–4 seconds**, dominated by: HTTP round-trip to Azure Logic Apps (~100–200 ms), internal workflow engine scheduling (~200–500 ms), and the outbound REST call to Kintone (~100–500 ms depending on region proximity).

**Cold start (workflow idle for several minutes):**  
Unlike Azure Functions, Consumption Logic Apps do not fully scale to zero in the Functions sense — the workflow orchestration infrastructure is shared and always running. However, the first invocation after an extended idle period can add **1–3 seconds** of additional scheduler warm-up latency. This is meaningfully lower than typical Azure Functions cold starts.

**Foundry agent overhead:**  
The agent's tool selection (LLM call) and the HTTP round-trip from Foundry to the Logic App callback URL adds another layer. In practice, agent-triggered tool invocations from Foundry typically add **200–600 ms** over the base Logic App execution time.

**Latency risk factors:**
- Kintone's REST API enforces a limit of **10,000 API requests per day per App**, resetting at 09:00 JST. Rate limiting responses from Kintone will cause Logic App retry attempts, each adding 20–30 seconds depending on the configured retry policy.
- Workflows handling large Kintone record sets using pagination will execute more calls and incur proportionally higher latency and cost.

---

## Summary Assessment

| Dimension | Finding | Risk Level |
|---|---|---|
| Native endpoint coverage | Only 2/5 required endpoints covered (POST, PUT record only) | 🔴 High — HTTP action workaround required |
| File API support | Completely absent from native connector | 🔴 High — full custom HTTP action required |
| API token authentication | Not supported by managed connector; requires HTTP action | 🟡 Medium — solvable, adds authoring steps |
| Foundry integration | Well-supported via portal gallery and SDK; Consumption SKU required | 🟢 Low |
| Cost | Very low per-invocation (~$0.0002); no idle cost | 🟢 Low |
| Latency (warm) | 1–4 seconds end-to-end; acceptable for async agent tools | 🟢 Low |
| Latency (cold) | 2–7 seconds worst-case; acceptable for non-latency-critical agents | 🟡 Medium |

**Feasibility verdict:** The Logic Apps path is feasible but the native Kintone connector is insufficient on its own. A hybrid design — using the managed connector for event triggers and replacing all read/file operations with direct HTTP actions — is required to meet the full API surface. This increases workflow authoring complexity but keeps the solution within standard Logic Apps tooling with no custom connector development required.
