# Prototyping and running an AI agent on Red Hat OpenShift AI 3.5.1

This guide is for Red Hat OpenShift AI Self-Managed 3.5.1. It covers prototyping in Gen AI Studio, running a tool-calling agent in a workbench against a model served on the cluster, and, when a separate registry is deployed, publishing an Agent-to-Agent (A2A) agent card.

OpenShift AI 3.5.1 does not install an Agent Registry, and the dashboard Model Registry does not accept agent cards. A2A discovery is provided by Red Hat build of Apicurio Registry, which is a separate product.

## Support level

| Capability | Status in 3.5.1 | Where it lives |
| --- | --- | --- |
| Gen AI Studio playground, including **View code** | Technology Preview | **Gen AI studio → Playground** |
| Saved playground agents | Developer Preview, off by default (`agentConfigManagement`) | **Gen AI studio → AI asset endpoints → Agents** |
| Agent Catalog starter kits | Developer Preview, off by default (`agentsCatalog`) | **AI hub → Agents** |
| OpenShell agent sandboxes | Developer Preview | Separate from this guide |
| Model registry | Generally available | **AI hub → Registry**. Registers models only |
| A2A agent-card registry | Not part of OpenShift AI | Red Hat build of Apicurio Registry 3.2 or later, with A2A enabled |

Technology Preview and Developer Preview features are not production-supported.

## Before you start

- A project exists, and you can open it from **Projects** in the dashboard.
- A tool-calling model is deployed and marked **Add as AI asset endpoint**. Models as a Service (MaaS) models that are already exposed to the project also appear on **Gen AI studio → AI asset endpoints**.
- The serving runtime enables tool calling. On the vLLM runtime included with OpenShift AI, that means additional arguments such as `--enable-auto-tool-choice` and `--tool-call-parser=<parser>`. The parser and chat template depend on the model family. The playground docs use this Qwen example: `--enable-auto-tool-choice --tool-call-parser=hermes`. Without these arguments, tool calls fail or are ignored, often with no clear error.
- The value you send as `model` is the served model id, not necessarily the Hugging Face repository id. The default vLLM ServingRuntime sets `--served-model-name` from the InferenceService name. Confirm it with `GET <endpoint>/v1/models`.
- MCP servers in the playground are created by a cluster administrator in the `gen-ai-aa-mcp-servers` ConfigMap in the `redhat-ods-applications` namespace. The playground does not accept ad hoc function definitions.

## Phase 1: Prototype in the Gen AI Studio playground

The playground is a chat session with a **Configure** panel. Tabs are **Model**, **Prompt**, **Knowledge**, **MCP**, and **Guardrails** when an administrator has enabled guardrails. Chat history is discarded when the browser session ends. Prompts saved from the **Prompt** tab are stored in MLflow and remain available to the project.

1. From the dashboard, click **Gen AI studio → Playground**.
2. Select the project, click **Create playground**, choose the model, set its type to **Inference**, and click **Create**.

   You can also start from **Gen AI studio → AI asset endpoints**, open the **Models** tab, and click **Add to playground**.
3. On the **Prompt** tab, set the system instructions for the agent's role and guardrails.
4. On the **MCP** tab, select a server that an administrator has already configured, click **Authorize**, and use **View tools** to see that server's tools. Authorization lasts for the browser session.
5. For RAG, open the **Knowledge** tab and upload files, or select an external vector store. OpenShift AI 3.5 provides a default PostgreSQL vector store with pgvector for playground experiments. Uploading a file is the playground RAG path. There is no separate "vector index retriever" toggle, and you cannot type raw tool schemas into the playground.
6. Send a prompt that should call a tool or use the uploaded documents. The model must support tool calling.
7. Click **View code**, then **Copy code**.

**View code** returns one Python template. There is no language picker and no execution-style picker. Red Hat documents this output as a starting point that records the model, parameters, MCP tools, and RAG files. It is not a runnable script. Paste it into the workbench and adapt it.

### Optional: save the playground configuration

Saving agents is off until an administrator sets `spec.dashboardConfig.agentConfigManagement` to `"true"` on the `OdhDashboardConfig` custom resource (typically `odh-dashboard-config` in `redhat-ods-applications`).

A saved agent stores the model, inference parameters, MLflow prompt reference, RAG sources, and MCP server connections as a Kubernetes resource in the project namespace. Project members can load, rename, and delete it from the **Agents** tab on **AI asset endpoints**. **Save as new agent** copies the configuration. This is project-scoped playground state. It is not an A2A agent card and it does not register the agent for cluster-wide discovery.

The Agent Catalog (**AI hub → Agents**, flag `agentsCatalog: "true"`) lists starter kits such as LangGraph, CrewAI, and LlamaIndex. It does not publish an agent you built in the playground.

## Phase 2: Create a workbench

1. Click **Projects**, open the project, open the **Workbenches** tab, and click **Create workbench**.
2. Set the name to `agent-dev-workbench`. The Kubernetes resource name is fixed after creation. Use lowercase letters, numbers, and hyphens.
3. Under **Image selection**, choose **Standard Data Science** and the latest image version. Images marked 2025.2 are still listed in 3.5 and are marked outdated.

   Use **Standard Data Science** when the workbench only calls a remote model. Choose the **CUDA** image and a hardware profile that requests an NVIDIA GPU when the workbench itself runs a local model. **PyTorch** is for PyTorch training and NLP or vision work, not for calling a vLLM or MaaS endpoint.
4. Under **Deployment size**, select a **Hardware profile**. Hardware profiles are generally available in 3.5 and are the deployment-size control. A small CPU profile is enough for the sample in this guide. Use **Customize resource requests and limits** only when the profile defaults are too small.
5. Under **Cluster storage**, create persistent storage or attach an existing volume. The workbench home directory is on this volume. Packages installed with a plain `pip install` into the image are lost when the workbench restarts. Put a virtual environment on the persistent volume if the packages must survive a restart:

   ```bash
   python -m venv /opt/app-root/src/agent-venv
   source /opt/app-root/src/agent-venv/bin/activate
   pip install --upgrade openai
   ```

   The sample below uses the OpenAI client against the served model's `/v1` API. LangChain, LangGraph, and the MCP Python SDK are not required for this sample. Use them only when you are building on one of those frameworks.
6. Add environment variables before you start the workbench. Click **Add environment variable** for each entry.

   | Variable | Type | Value |
   | --- | --- | --- |
   | `MODEL_ENDPOINT` | Config Map | OpenAI-compatible base URL, including `/v1` |
   | `MODEL_NAME` | Config Map | Served model id from `GET /v1/models` |
   | `AUTH_TOKEN` | Secret, or Existing secret | Token required by that endpoint. Use an existing Opaque secret when the platform team already manages the credential |

   Take `MODEL_ENDPOINT` from the model deployment page. For a KServe raw Deployment of the vLLM runtime, the in-cluster address is usually:

   ```text
   http://<inference-service-name>-predictor.<project>.svc.cluster.local:8080/v1
   ```

   When the predictor Service is headless (`clusterIP: None`), use container port `8080`. The Service port mapping does not apply. A MaaS model uses the gateway URL and an API key from **Gen AI studio → API keys** instead of the predictor Service.

   There is no `AGENT_REGISTRY_URL` service in an OpenShift AI install. Add that variable only after you deploy Apicurio Registry, and set it to that registry's Service.
7. Click **Create workbench**. Open the IDE when the status is **Ready**. If you change a secret after the workbench has started, restart the workbench so the container picks up the new value.

## Phase 3: Run the agent in the workbench

Create `agent.py` on the workbench storage. This is a tool-calling loop for the OpenAI-compatible API. It is independent of the playground template. It runs in the workbench process. It does not listen on a port and it is not an A2A server.

The tool handlers below return static JSON so you can confirm the loop. Replace them with calls to your own APIs before anyone relies on the answers.

```python
import json
import os

from openai import OpenAI

MODEL_ENDPOINT = os.environ["MODEL_ENDPOINT"]
MODEL_NAME = os.environ["MODEL_NAME"]
# vLLM accepts any non-empty key when the runtime was not started with --api-key.
AUTH_TOKEN = os.getenv("AUTH_TOKEN", "EMPTY")

client = OpenAI(base_url=MODEL_ENDPOINT, api_key=AUTH_TOKEN)

SYSTEM_PROMPT = """You are an automated OpenShift operations assistant.
Always use tools to query cluster state or calculate capacity before answering."""

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_pod_status",
            "description": "Checks the operational status of pods in a namespace.",
            "parameters": {
                "type": "object",
                "properties": {
                    "namespace": {
                        "type": "string",
                        "description": "Target Kubernetes namespace",
                    }
                },
                "required": ["namespace"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calculate_resource_quota",
            "description": "Calculates total CPU cores for a number of pods.",
            "parameters": {
                "type": "object",
                "properties": {
                    "num_pods": {"type": "integer", "description": "Number of pods"},
                    "cpu_per_pod": {
                        "type": "number",
                        "description": "CPU cores required per pod",
                    },
                },
                "required": ["num_pods", "cpu_per_pod"],
            },
        },
    },
]


def execute_tool(name: str, arguments: dict) -> str:
    if name == "get_pod_status":
        namespace = arguments.get("namespace", "default")
        return json.dumps(
            {
                "namespace": namespace,
                "running_pods": 12,
                "failing_pods": 0,
                "status": "Healthy",
            }
        )
    if name == "calculate_resource_quota":
        num_pods = arguments.get("num_pods", 1)
        cpu_per_pod = arguments.get("cpu_per_pod", 1.0)
        return json.dumps(
            {
                "num_pods": num_pods,
                "cpu_per_pod": cpu_per_pod,
                "total_cpu_cores": num_pods * cpu_per_pod,
            }
        )
    return json.dumps({"error": f"Tool '{name}' not found"})


def run_agent(query: str, max_rounds: int = 5) -> str:
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": query},
    ]

    for _ in range(max_rounds):
        response = client.chat.completions.create(
            model=MODEL_NAME,
            messages=messages,
            tools=tools,
            tool_choice="auto",
            temperature=0.0,
        )
        message = response.choices[0].message
        if not message.tool_calls:
            return message.content or ""

        messages.append(message.model_dump(exclude_none=True))
        for tool_call in message.tool_calls:
            fn_args = json.loads(tool_call.function.arguments or "{}")
            print(f"[Invoking Tool]: {tool_call.function.name} with {fn_args}")
            messages.append(
                {
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": execute_tool(tool_call.function.name, fn_args),
                }
            )

    return "Stopped after the maximum number of tool rounds."


if __name__ == "__main__":
    prompt = (
        "Check the health of pods in 'production' and calculate CPU needed "
        "for 6 pods taking 2.5 cores each."
    )
    print(f"\n[Agent Response]:\n{run_agent(prompt)}")
```

Run it from the activated virtual environment:

```bash
python agent.py
```

A successful run prints two tool invocations and a final answer that uses both results. If the model never calls a tool, check `MODEL_NAME` against `/v1/models` and confirm the runtime was deployed with `--enable-auto-tool-choice` and a parser for that model family.

## Phase 4: Describe the agent with an A2A agent card

Create `agent-card.json` only when an A2A client will discover this agent. The card describes a deployed A2A endpoint. Point `supportedInterfaces[].url` at that endpoint. The workbench script in Phase 3 is not that endpoint, so leave this phase until the agent is served over HTTP.

Apicurio Registry 3.2 validates `AGENT_CARD` artifacts against the A2A card shape. With validation level `FULL`, each interface needs `url`, `protocolBinding`, and `protocolVersion`, and each skill needs `id` and `name`. Skill tool parameters are not fields on the card. Capabilities used by the registry are booleans such as `streaming` and `pushNotifications`.

```json
{
  "name": "cluster-ops-agent",
  "description": "Cluster operations agent prototyped in Gen AI Studio and run on OpenShift AI.",
  "version": "1.0.0",
  "supportedInterfaces": [
    {
      "url": "http://cluster-ops-agent.my-project.svc.cluster.local:8000",
      "protocolBinding": "http+json",
      "protocolVersion": "1.0"
    }
  ],
  "provider": {
    "organization": "Platform Engineering",
    "url": "https://internal.company.io"
  },
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"],
  "skills": [
    {
      "id": "get_pod_status",
      "name": "Get Pod Status",
      "description": "Checks the operational status of pods in a namespace.",
      "tags": ["openshift", "operations"],
      "examples": ["Check the health of pods in production"]
    },
    {
      "id": "calculate_resource_quota",
      "name": "Calculate Resource Quota",
      "description": "Calculates total CPU cores required across pods.",
      "tags": ["openshift", "capacity"],
      "examples": ["Calculate CPU needed for 6 pods at 2.5 cores each"]
    }
  ]
}
```

Replace the interface URL, provider URL, and version before registering. `provider.url` must be an absolute URI.

## Phase 5: Register the card in Apicurio Registry

Skip this phase unless Red Hat build of Apicurio Registry is already running and A2A is enabled (`apicurio.features.experimental.enabled=true` and `apicurio.a2a.enabled=true`, or the matching `APICURIO_FEATURES_EXPERIMENTAL_ENABLED` and `APICURIO_A2A_ENABLED` environment variables). If A2A is disabled, card creation and `/.well-known/agents` return HTTP 404.

Do not register the card through **AI hub → Registry**. That UI registers a model name, format, and storage location. It has no `AGENT_CARD` type and no agent-card upload action.

Set `REGISTRY_URL` to the registry Service, for example `http://<apicurio-service>.<namespace>.svc.cluster.local:8080`. Port `8080` is the registry API port used by the Apicurio 3.2 quick start. Confirm the Service name and port in the namespace where the registry is installed.

The Registry REST API v3 stores the card inside a create-artifact document. The card JSON is a string in `firstVersion.content.content`. Posting the raw card, or setting `X-Registry-ArtifactType`, is not the documented 3.2 create call.

```python
import json
import os

import requests

REGISTRY_URL = os.environ["REGISTRY_URL"].rstrip("/")
GROUP_ID = "ai-agents"
ARTIFACT_ID = "cluster-ops-agent"

with open("agent-card.json", encoding="utf-8") as handle:
    agent_card = json.load(handle)

session = requests.Session()
session.headers["Content-Type"] = "application/json"
token = os.getenv("REGISTRY_TOKEN")
if token:
    session.headers["Authorization"] = f"Bearer {token}"

group_response = session.post(
    f"{REGISTRY_URL}/apis/registry/v3/groups",
    json={"groupId": GROUP_ID, "description": "Operational and platform AI agents"},
)
if group_response.status_code not in (200, 201, 409):
    raise SystemExit(
        f"Group create failed: HTTP {group_response.status_code} {group_response.text}"
    )

create_body = {
    "artifactId": ARTIFACT_ID,
    "artifactType": "AGENT_CARD",
    "name": agent_card["name"],
    "description": agent_card["description"],
    "firstVersion": {
        "version": agent_card["version"],
        "content": {
            "contentType": "application/json",
            "content": json.dumps(agent_card),
        },
    },
}
create_response = session.post(
    f"{REGISTRY_URL}/apis/registry/v3/groups/{GROUP_ID}/artifacts",
    json=create_body,
)
if create_response.status_code in (200, 201):
    print(f"Registered {GROUP_ID}/{ARTIFACT_ID} version {agent_card['version']}")
elif create_response.status_code == 409:
    # The artifact id already exists. Bump agent_card["version"] before re-running,
    # then create a new version instead of another artifact.
    version_body = {
        "version": agent_card["version"],
        "content": {
            "contentType": "application/json",
            "content": json.dumps(agent_card),
        },
    }
    version_response = session.post(
        f"{REGISTRY_URL}/apis/registry/v3/groups/{GROUP_ID}/artifacts/{ARTIFACT_ID}/versions",
        json=version_body,
    )
    print(
        f"New version response: HTTP {version_response.status_code} {version_response.text}"
    )
else:
    raise SystemExit(
        f"Registration failed: HTTP {create_response.status_code} {create_response.text}"
    )
```

The same calls from a shell:

```bash
curl -sS -X POST "$REGISTRY_URL/apis/registry/v3/groups" \
  -H "Content-Type: application/json" \
  -d '{"groupId":"ai-agents","description":"Operational and platform AI agents"}'

python - <<'PY'
import json
with open("agent-card.json", encoding="utf-8") as handle:
    card = json.load(handle)
body = {
    "artifactId": "cluster-ops-agent",
    "artifactType": "AGENT_CARD",
    "name": card["name"],
    "description": card["description"],
    "firstVersion": {
        "version": card["version"],
        "content": {"contentType": "application/json", "content": json.dumps(card)},
    },
}
with open("create-artifact.json", "w", encoding="utf-8") as handle:
    json.dump(body, handle)
PY

curl -sS -X POST "$REGISTRY_URL/apis/registry/v3/groups/ai-agents/artifacts" \
  -H "Content-Type: application/json" \
  -d @create-artifact.json
```

A second registration of the same artifact id returns HTTP 409 until you post a new version whose `version` string is different from the versions already stored.

## Phase 6: Discover the registered card

These endpoints are served by Apicurio Registry, at `REGISTRY_URL`. They are not served by the OpenShift AI dashboard or by a Service named `agent-registry` in `redhat-ods-applications`.

`GET /.well-known/agent.json` returns the registry's own agent card. The list of cards you published is `GET /.well-known/agents`.

```bash
curl -sS "$REGISTRY_URL/.well-known/agents"
curl -sS "$REGISTRY_URL/.well-known/agents?skill=calculate_resource_quota"
curl -sS "$REGISTRY_URL/.well-known/agents/ai-agents/cluster-ops-agent"
```

The skill query matches the skill `id`. The search response is a summary. The last request returns the stored agent card. Pass `?version=` on that request when you need a version other than the latest.

When registry authentication is enabled, the search result includes only cards the caller is allowed to read. Send the same bearer token used to create the artifact.
