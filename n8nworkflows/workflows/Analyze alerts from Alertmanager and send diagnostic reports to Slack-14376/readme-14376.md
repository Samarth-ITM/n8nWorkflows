Analyze alerts from Alertmanager and send diagnostic reports to Slack

https://n8nworkflows.xyz/workflows/analyze-alerts-from-alertmanager-and-send-diagnostic-reports-to-slack-14376


# Analyze alerts from Alertmanager and send diagnostic reports to Slack

# 1. Workflow Overview

This workflow receives alerts from Alertmanager, converts each alert into a compact AI-ready incident brief, lets an AI agent investigate the issue using multiple observability and infrastructure tools, then posts the generated diagnostic report into the matching Slack alert thread.

Its main use case is proactive incident triage for infrastructure or Kubernetes-based production environments. It is designed for SRE / DevOps teams who already receive Alertmanager notifications in Slack and want an automated first-pass diagnosis before a human responder begins the investigation.

## 1.1 Input Reception and Variable Initialization

The workflow starts with a secured webhook endpoint that accepts Alertmanager `POST` payloads. It then enriches the execution context with static values such as Grafana datasource UIDs used later by the AI agent.

## 1.2 Alert Preprocessing

The incoming Alertmanager payload is normalized by a Code node. For each alert in the payload, the node produces one execution item containing:
- a human-readable alert summary for the AI agent
- a session identifier
- the alert fingerprint used later to match the Slack message thread

This block is important because one Alertmanager webhook may contain multiple alerts.

## 1.3 AI Investigation and Tool Access

An AI Agent receives the prepared alert text and follows a detailed system prompt specialized for SRE incident analysis. The agent uses:
- an OpenAI chat model
- MCP tools for Kubernetes, Grafana, DigitalOcean, and GitLab
- a Qdrant vector store backed by Gemini embeddings for knowledge retrieval

The agent is instructed to investigate autonomously and produce a Slack-friendly report with sections for what happened, timeline, root cause, and troubleshooting.

## 1.4 Slack Thread Matching

After the AI report is generated, the workflow retrieves recent Slack channel messages and searches for the one corresponding to the current alert fingerprint. If found, it extracts the proper `thread_ts` so the diagnostic report can be posted as a thread reply.

## 1.5 Slack Report Delivery

The final block posts the AI-generated analysis back into Slack, ideally as a reply to the original alert thread.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Variable Initialization

### Overview
This block receives Alertmanager webhook calls and injects configuration variables required later by the AI prompt. It forms the entry point of the workflow and ensures datasource IDs are available before preprocessing begins.

### Nodes Involved
- Receive alerts
- SetVars

### Node Details

#### Receive alerts
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point webhook that accepts Alertmanager payloads over HTTP.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `alertmanager`
  - Authentication: Basic Auth
- **Key expressions or variables used:** None in parameters, but the webhook payload is later accessed as `body`.
- **Input and output connections:**
  - No input; this is the trigger node
  - Output to: `SetVars`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Basic auth failure if Alertmanager uses wrong credentials
  - Payload shape mismatch if Alertmanager sends unexpected JSON
  - Webhook not reachable from Alertmanager network
  - In production, test URL vs production URL confusion is common
- **Sub-workflow reference:** None

#### SetVars
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds static configuration values to the current item.
- **Configuration choices:**
  - Adds:
    - `prometheus_uid = cb0069a2-5028-423c-b8a6-95bf3c9d39e7`
    - `loki_uid = grafanacloud-logs`
  - `includeOtherFields: true` preserves the incoming alert payload
- **Key expressions or variables used:** None; values are hardcoded
- **Input and output connections:**
  - Input from: `Receive alerts`
  - Output to: `PreProcessAlerts`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - Incorrect datasource UIDs will cause the AI agent to query the wrong Grafana datasource or fail to retrieve metrics/logs
  - If future prompts depend on more variables, they must be added here
- **Sub-workflow reference:** None

---

## 2.2 Alert Preprocessing

### Overview
This block transforms the raw Alertmanager payload into one AI-ready item per alert. It also extracts a reusable alert fingerprint for Slack thread matching.

### Nodes Involved
- PreProcessAlerts

### Node Details

#### PreProcessAlerts
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the webhook payload and emits one item per alert with normalized fields.
- **Configuration choices:**
  - Reads payload from either:
    - `$input.first().json.body`
    - or fallback `$input.first().json`
  - Skips noisy labels:
    - `alertname`, `endpoint`, `instance`, `job`, `metrics_path`, `prometheus`, `project`, `env`
  - Renames some labels for readability:
    - `grafana_board` → `Grafana Dashboard`
    - `persistentvolumeclaim` → `PVC`
  - Builds a multi-line `chatInput` string containing:
    - status
    - alert name
    - severity
    - remaining labels
    - `startsAt`
    - `generatorURL`
    - annotations
  - Builds `sessionId` from:
    - `fingerprint`, if present
    - otherwise fallback using alert name, namespace, and service
  - Returns fields:
    - `action: "sendMessage"`
    - `chatInput`
    - `sessionId`
    - `alert_id`
- **Key expressions or variables used:**
  - `const webhookData = $input.first().json.body || $input.first().json;`
  - `alert.fingerprint`
  - `labels.namespace`
  - `labels.service`
- **Input and output connections:**
  - Input from: `SetVars`
  - Output to: `AI Agent`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If `alerts` is missing or empty, the node returns an empty array and downstream nodes will not run
  - If Alertmanager payload format changes, `body.alerts` parsing may break
  - Missing `fingerprint` reduces Slack matching reliability
  - The fallback `sessionId` may not be unique enough in noisy environments
- **Sub-workflow reference:** None

---

## 2.3 AI Investigation and Tool Access

### Overview
This is the analytical core of the workflow. The AI agent receives the formatted alert and investigates it using external tools for Kubernetes, Grafana, DigitalOcean, GitLab, and vector knowledge retrieval.

### Nodes Involved
- AI Agent
- OpenAI Chat Model
- K8S mcp
- Grafana mcp
- Digitalocean MCP
- Gitlab MCP
- Qdrant Vector Store
- Embeddings Google Gemini

### Node Details

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrates LLM reasoning and tool invocation.
- **Configuration choices:**
  - Uses a custom system prompt specialized for game studio SRE operations
  - Explicitly directs the agent to:
    - inspect Kubernetes resources and logs
    - query Grafana Prometheus and Loki
    - search dashboards
    - use vector knowledge for alert rules
    - inspect DigitalOcean resources
    - check GitLab activity/releases
  - Uses datasource IDs from `SetVars` via expressions
  - Enforces strict behavior:
    - do not retry failed tools more than once
    - acknowledge empty/error results and move on
    - do not make changes through MCP
    - do not ask questions
  - Requires output sections:
    - What happened
    - Event timeline
    - Root cause
    - Troubleshooting tips
- **Key expressions or variables used:**
  - `{{ $('SetVars').first().json.prometheus_uid }}`
  - `{{ $('SetVars').first().json.loki_uid }}`
- **Input and output connections:**
  - Main input from: `PreProcessAlerts`
  - Language model input from: `OpenAI Chat Model`
  - Tool inputs from:
    - `K8S mcp`
    - `Grafana mcp`
    - `Digitalocean MCP`
    - `Gitlab MCP`
    - `Qdrant Vector Store`
  - Main output to: `GetAlertMessages`
- **Version-specific requirements:** Type version `3.1`
- **Edge cases or potential failure types:**
  - LLM quota/auth/model access errors
  - Tool availability failures if MCP endpoints are down
  - Hallucinated assumptions if tool results are sparse
  - Agent latency may be significant when many tools are used
  - Prompt references GitHub, but the actual configured node is GitLab MCP; this is a documentation/prompt mismatch and may confuse behavior
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used by the AI agent.
- **Configuration choices:**
  - Model: `gpt-5.3-codex`
  - No extra options configured
  - Uses OpenAI API credentials
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `AI Agent` as `ai_languageModel`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - Invalid OpenAI API key
  - Model unavailable for the account
  - Regional or enterprise policy restrictions
  - Token/cost growth if prompt and tool traces are large
- **Sub-workflow reference:** None

#### K8S mcp
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes Kubernetes inspection tools to the agent through MCP.
- **Configuration choices:**
  - Endpoint: `http://mcp-kubernetes:8089/mcp`
  - Includes selected tools only:
    - configuration view
    - events listing
    - namespace listing
    - node logs/stats/top
    - pod get/list/log/top
    - generic resource get/list
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `AI Agent` as `ai_tool`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - MCP server unreachable
  - Cluster RBAC denies access
  - Resource disappears between alert time and investigation time
  - Log retrieval may fail on terminated pods
- **Sub-workflow reference:** None

#### Grafana mcp
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Gives the agent access to Grafana dashboards, alerts, Prometheus queries, and Loki logs.
- **Configuration choices:**
  - Endpoint: `http://mcp-grafana:8000/mcp`
  - Timeout: `60000 ms`
  - Includes many Grafana tools, notably:
    - `query_prometheus`
    - `query_loki_logs`
    - dashboard search and summary
    - datasource metadata
    - alert rule lookups
    - panel image/deeplink helpers
- **Key expressions or variables used:** Datasource UIDs are referenced in the AI prompt, not directly here
- **Input and output connections:**
  - Connected to `AI Agent` as `ai_tool`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - MCP server unavailable
  - Datasource UID mismatch
  - Query timeout despite 60s limit
  - Grafana API permissions insufficient
- **Sub-workflow reference:** None

#### Digitalocean MCP
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Lets the agent inspect DigitalOcean infrastructure resources.
- **Configuration choices:**
  - Endpoint: `http://mcp-digitalocean:8090/mcp`
  - Selected tools include:
    - app info/logs/deployment status
    - DOKS cluster and nodepool info
    - droplets, load balancers, VPCs, firewalls, domains, certificates, regions, sizes
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `AI Agent` as `ai_tool`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - MCP auth or backend API issues
  - Rate limits from DigitalOcean
  - Resource names/IDs in alerts may not map cleanly to DigitalOcean assets
- **Sub-workflow reference:** None

#### Gitlab MCP
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Gives the agent read-only repository, merge request, pipeline, and commit inspection capabilities.
- **Configuration choices:**
  - Endpoint expression: `http://mcp-gitlab:8091/mcp`
  - Selected tools include:
    - repository search
    - file content access
    - merge requests and diffs
    - branches and commit diffs
    - projects and namespaces
    - pipelines and jobs
- **Key expressions or variables used:**
  - Endpoint is defined as an expression string, though static in content
- **Input and output connections:**
  - Connected to `AI Agent` as `ai_tool`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - GitLab token scope too narrow
  - Organization/project naming mismatch
  - The AI prompt says “Github tools” and “playneta organization,” but the configured integration is GitLab, so prompts should be corrected
- **Sub-workflow reference:** None

#### Qdrant Vector Store
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStoreQdrant`  
  Exposes vector retrieval as a tool for the agent.
- **Configuration choices:**
  - Mode: `retrieve-as-tool`
  - Collection: `observability`
  - Tool description: generic knowledge database fetcher
  - Uses Qdrant API credentials
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Embeddings input from: `Embeddings Google Gemini`
  - Tool output to: `AI Agent`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - Missing collection
  - Embedding mismatch with indexed data
  - Qdrant auth/network failure
  - Low retrieval quality if documents are not well chunked
- **Sub-workflow reference:** None

#### Embeddings Google Gemini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`  
  Supplies embeddings used by the Qdrant retrieval tool.
- **Configuration choices:**
  - Model: `models/gemini-embedding-001`
  - Uses Google Gemini / PaLM API credentials
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to: `Qdrant Vector Store` as `ai_embedding`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Invalid API key
  - Embedding model unavailable
  - Retrieval quality degradation if indexed documents were embedded with another model
- **Sub-workflow reference:** None

---

## 2.4 Slack Thread Matching

### Overview
This block loads recent messages from the Slack alerts channel and tries to identify the original alert message using the alert fingerprint. It then prepares the thread timestamp and AI output for final posting.

### Nodes Involved
- GetAlertMessages
- FindAlert

### Node Details

#### GetAlertMessages
- **Type and technical role:** `n8n-nodes-base.slack`  
  Reads recent message history from a specific Slack channel.
- **Configuration choices:**
  - Resource: `channel`
  - Operation: `history`
  - Channel ID: `C053DG9S2EL`
  - Limit: `10`
  - OAuth2 authentication
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `AI Agent`
  - Output to: `FindAlert`
- **Version-specific requirements:** Type version `2.4`
- **Edge cases or potential failure types:**
  - If the relevant alert message is older than the last 10 messages, it will not be found
  - Slack scopes may be insufficient for channel history
  - Wrong channel ID breaks thread matching
- **Sub-workflow reference:** None

#### FindAlert
- **Type and technical role:** `n8n-nodes-base.code`  
  Searches fetched Slack messages for the alert fingerprint and prepares the final payload for posting.
- **Configuration choices:**
  - Reads current alert fingerprint from:
    - `$('PreProcessAlerts').first().json.alert_id`
  - Reads AI report from:
    - `$('AI Agent').first().json.output`
  - Iterates through Slack messages and attachments
  - Match rule:
    - if message text contains `alert_id`
    - or attachment title/text contains `alert_id`
  - If found:
    - `thread_ts = msg.thread_ts || msg.ts`
  - Returns:
    - `thread_ts`
    - `matched_ts`
    - `matched_title`
    - `alert_id`
    - `agentOutput`
- **Key expressions or variables used:**
  - `$('PreProcessAlerts').first().json.alert_id`
  - `$('AI Agent').first().json.output`
  - `msg.thread_ts || msg.ts`
- **Input and output connections:**
  - Input from: `GetAlertMessages`
  - Output to: `SendReport`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Uses `.first()` rather than per-item linkage, which may be problematic if the webhook contains multiple alerts and n8n processes multiple items in one execution
  - If no Slack message contains the fingerprint, `thread_ts` is returned empty
  - If Slack alert formatting changes and fingerprint is no longer in text or attachments, matching fails
  - If `AI Agent` output field differs from `output`, report text may be blank or default to `No data`
- **Sub-workflow reference:** None

---

## 2.5 Slack Report Delivery

### Overview
This final block posts the generated diagnosis to Slack. If a matching alert thread was found, the message is posted as a threaded reply.

### Nodes Involved
- SendReport

### Node Details

#### SendReport
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the AI-generated report to Slack.
- **Configuration choices:**
  - Sends message text from `agentOutput`
  - Target selection: channel
  - Channel ID: `C053DG9S2EL`
  - `thread_ts` populated from the previous node
  - OAuth2 authentication
- **Key expressions or variables used:**
  - `={{ $json.agentOutput }}`
  - `={{ $json.thread_ts }}`
- **Input and output connections:**
  - Input from: `FindAlert`
  - No downstream node
- **Version-specific requirements:** Type version `2.4`
- **Edge cases or potential failure types:**
  - If `thread_ts` is empty, behavior depends on Slack node handling; it may post as a normal channel message instead of a thread reply
  - Slack app may lack permission to post in the selected channel
  - Long AI responses may exceed practical Slack readability limits
- **Sub-workflow reference:** None

---

## 2.6 Documentation and In-Canvas Notes

### Overview
These sticky notes provide architectural guidance, deployment notes, and setup requirements. They do not affect execution but are important for reproducing the workflow correctly.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the input area.
- **Configuration choices:** Marks the input chain as “Receive and format alerts from Alertmanager”
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the analysis area as “Alert analysis”
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents MCP dependencies and Qdrant credential requirement, with links:
  - [grafana-mcp](https://github.com/grafana/mcp-grafana)
  - [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)
  - [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)
  - [gitlab-mcp](https://github.com/zereight/gitlab-mcp)
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** Misconfigured remote MCP URLs will break tool usage
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the output area as “Post generated report to Slack”
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Provides overall workflow purpose, usage instructions, and requirements
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** The setup instructions must be followed for the workflow to work end-to-end
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive alerts | n8n-nodes-base.webhook | Receives Alertmanager webhook events via Basic Auth |  | SetVars | # Input chain\nReceive and format alerts from Alertmanager |
| SetVars | n8n-nodes-base.set | Injects Grafana datasource UIDs into execution data | Receive alerts | PreProcessAlerts | # Input chain\nReceive and format alerts from Alertmanager |
| PreProcessAlerts | n8n-nodes-base.code | Normalizes webhook payload and emits one AI task per alert | SetVars | AI Agent | # Input chain\nReceive and format alerts from Alertmanager |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Investigates the alert with LLM reasoning and tool use | PreProcessAlerts; OpenAI Chat Model; K8S mcp; Grafana mcp; Digitalocean MCP; Gitlab MCP; Qdrant Vector Store | GetAlertMessages | # Alert analysis |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the LLM for the AI agent |  | AI Agent | # Alert analysis |
| K8S mcp | @n8n/n8n-nodes-langchain.mcpClientTool | Exposes Kubernetes inspection tools through MCP |  | AI Agent | # MCP servers\nAdd correct URLs for remote MCPs\nUse following mcp:\n* [grafana-mcp](https://github.com/grafana/mcp-grafana)\n* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)\n* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)\n* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)\nCreate QdrantAPI credentials |
| Grafana mcp | @n8n/n8n-nodes-langchain.mcpClientTool | Exposes Grafana, Prometheus, and Loki investigation tools |  | AI Agent | # MCP servers\nAdd correct URLs for remote MCPs\nUse following mcp:\n* [grafana-mcp](https://github.com/grafana/mcp-grafana)\n* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)\n* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)\n* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)\nCreate QdrantAPI credentials |
| Digitalocean MCP | @n8n/n8n-nodes-langchain.mcpClientTool | Exposes DigitalOcean infrastructure inspection tools |  | AI Agent | # MCP servers\nAdd correct URLs for remote MCPs\nUse following mcp:\n* [grafana-mcp](https://github.com/grafana/mcp-grafana)\n* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)\n* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)\n* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)\nCreate QdrantAPI credentials |
| Gitlab MCP | @n8n/n8n-nodes-langchain.mcpClientTool | Exposes GitLab repository and pipeline inspection tools |  | AI Agent | # MCP servers\nAdd correct URLs for remote MCPs\nUse following mcp:\n* [grafana-mcp](https://github.com/grafana/mcp-grafana)\n* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)\n* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)\n* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)\nCreate QdrantAPI credentials |
| Qdrant Vector Store | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Retrieves knowledge from the observability vector collection as an AI tool | Embeddings Google Gemini | AI Agent | # MCP servers\nAdd correct URLs for remote MCPs\nUse following mcp:\n* [grafana-mcp](https://github.com/grafana/mcp-grafana)\n* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)\n* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)\n* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)\nCreate QdrantAPI credentials |
| Embeddings Google Gemini | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Provides embeddings for Qdrant retrieval |  | Qdrant Vector Store | # MCP servers\nAdd correct URLs for remote MCPs\nUse following mcp:\n* [grafana-mcp](https://github.com/grafana/mcp-grafana)\n* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)\n* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)\n* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)\nCreate QdrantAPI credentials |
| GetAlertMessages | n8n-nodes-base.slack | Reads recent Slack messages to find the alert thread | AI Agent | FindAlert | # Output chain\nPost generated report to Slack |
| FindAlert | n8n-nodes-base.code | Matches the alert fingerprint with Slack history and prepares final post data | GetAlertMessages | SendReport | # Output chain\nPost generated report to Slack |
| SendReport | n8n-nodes-base.slack | Posts the AI report to the Slack alert thread or channel | FindAlert |  | # Output chain\nPost generated report to Slack |
| Sticky Note | n8n-nodes-base.stickyNote | Documents the input section |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documents the AI analysis section |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documents MCP and Qdrant dependencies |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documents the output section |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Provides overall description, usage notes, and requirements |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `AlertAssistant`.
   - Leave it inactive until all credentials and endpoints are configured.

2. **Add the webhook trigger**
   - Create a **Webhook** node named `Receive alerts`.
   - Set:
     - **HTTP Method**: `POST`
     - **Path**: `alertmanager`
     - **Authentication**: `Basic Auth`
   - Create or select Basic Auth credentials for Alertmanager.
   - Configure Alertmanager to send webhook payloads to this endpoint.

3. **Add a Set node for static variables**
   - Create a **Set** node named `SetVars`.
   - Enable keeping input fields with **Include Other Fields = true**.
   - Add two string fields:
     - `prometheus_uid` = `cb0069a2-5028-423c-b8a6-95bf3c9d39e7`
     - `loki_uid` = `grafanacloud-logs`
   - Connect `Receive alerts -> SetVars`.

4. **Add the preprocessing Code node**
   - Create a **Code** node named `PreProcessAlerts`.
   - Paste logic equivalent to:
     - read webhook payload from `body` or root JSON
     - iterate over `alerts`
     - build one outgoing item per alert
     - create a human-readable multiline `chatInput`
     - store the fingerprint in `alert_id`
     - create `sessionId`
   - Use the same skip-label strategy and label renaming if you want behavior identical to the original.
   - Connect `SetVars -> PreProcessAlerts`.

5. **Add the AI Agent**
   - Create an **AI Agent** node named `AI Agent`.
   - Set the system message to an SRE-focused prompt.
   - Include at minimum:
     - Kubernetes investigation instructions
     - Grafana/Prometheus/Loki tool usage
     - knowledge base usage
     - DigitalOcean and GitLab usage if relevant
     - “do not ask questions”
     - “do not make changes”
     - output format with sections:
       - What happened
       - Event timeline
       - Root cause
       - Troubleshooting tips
   - In the prompt, reference the Set node values with expressions:
     - `{{ $('SetVars').first().json.prometheus_uid }}`
     - `{{ $('SetVars').first().json.loki_uid }}`
   - Connect `PreProcessAlerts -> AI Agent`.

6. **Add the OpenAI model**
   - Create an **OpenAI Chat Model** node named `OpenAI Chat Model`.
   - Select model `gpt-5.3-codex` or another model available in your account.
   - Attach OpenAI credentials.
   - Connect it to `AI Agent` using the **AI Language Model** connection type.

7. **Add the Kubernetes MCP tool**
   - Create an **MCP Client Tool** node named `K8S mcp`.
   - Set endpoint URL to `http://mcp-kubernetes:8089/mcp`.
   - Use selected tools only:
     - `configuration_view`
     - `events_list`
     - `namespaces_list`
     - `nodes_log`
     - `nodes_stats_summary`
     - `nodes_top`
     - `pods_get`
     - `pods_list`
     - `pods_list_in_namespace`
     - `pods_log`
     - `pods_top`
     - `resources_get`
     - `resources_list`
   - Connect it to `AI Agent` as an **AI Tool**.

8. **Add the Grafana MCP tool**
   - Create an **MCP Client Tool** node named `Grafana mcp`.
   - Set endpoint URL to `http://mcp-grafana:8000/mcp`.
   - Set timeout to `60000 ms`.
   - Select these tools:
     - `generate_deeplink`
     - `get_alert_rule_by_uid`
     - `get_annotations`
     - `get_dashboard_by_uid`
     - `get_dashboard_panel_queries`
     - `get_dashboard_property`
     - `get_dashboard_summary`
     - `get_datasource_by_name`
     - `get_datasource_by_uid`
     - `get_panel_image`
     - `get_sift_analysis`
     - `get_sift_investigation`
     - `list_alert_rules`
     - `list_loki_label_names`
     - `list_loki_label_values`
     - `list_prometheus_label_names`
     - `list_prometheus_label_values`
     - `list_prometheus_metric_metadata`
     - `list_prometheus_metric_names`
     - `query_loki_logs`
     - `query_loki_patterns`
     - `query_loki_stats`
     - `query_prometheus`
     - `query_prometheus_histogram`
     - `search_dashboards`
   - Connect it to `AI Agent` as an **AI Tool**.

9. **Add the DigitalOcean MCP tool**
   - Create an **MCP Client Tool** node named `Digitalocean MCP`.
   - Set endpoint URL to `http://mcp-digitalocean:8090/mcp`.
   - Select the needed DigitalOcean read-only tools, including:
     - app info, logs, deployment status
     - DOKS cluster and nodepool queries
     - droplets, load balancers, VPCs, firewalls, domains, certificates, images
   - Connect it to `AI Agent` as an **AI Tool**.

10. **Add the GitLab MCP tool**
    - Create an **MCP Client Tool** node named `Gitlab MCP`.
    - Set endpoint URL to `http://mcp-gitlab:8091/mcp`.
    - Select tools for:
      - repository search
      - file content
      - merge requests
      - commits
      - branches
      - pipelines and jobs
      - namespaces and projects
    - Connect it to `AI Agent` as an **AI Tool**.
    - Recommended improvement: update the system prompt so it says **GitLab** instead of **GitHub**.

11. **Add the embedding model**
    - Create a **Google Gemini Embeddings** node named `Embeddings Google Gemini`.
    - Select model `models/gemini-embedding-001`.
    - Attach Google Gemini / PaLM API credentials.

12. **Add the Qdrant vector store tool**
    - Create a **Qdrant Vector Store** node named `Qdrant Vector Store`.
    - Set mode to **Retrieve as Tool**.
    - Set collection to `observability`.
    - Add a meaningful tool description such as “Use this tool to fetch alert rules and observability knowledge.”
    - Attach Qdrant API credentials.
    - Connect `Embeddings Google Gemini -> Qdrant Vector Store` with the **AI Embedding** connection.
    - Connect `Qdrant Vector Store -> AI Agent` with the **AI Tool** connection.

13. **Add Slack history lookup**
    - Create a **Slack** node named `GetAlertMessages`.
    - Configure:
      - **Resource**: `Channel`
      - **Operation**: `History`
      - **Channel ID**: your alerts channel, e.g. `C053DG9S2EL`
      - **Limit**: `10`
    - Use Slack OAuth2 credentials with permissions to read channel history.
    - Connect `AI Agent -> GetAlertMessages`.

14. **Add Slack message matching logic**
    - Create a **Code** node named `FindAlert`.
    - Implement logic to:
      - get `alert_id` from `PreProcessAlerts`
      - get AI output from `AI Agent`
      - iterate through Slack history items
      - check message text and attachments for the alert fingerprint
      - if matched, extract `thread_ts`
      - output:
        - `thread_ts`
        - `matched_ts`
        - `matched_title`
        - `alert_id`
        - `agentOutput`
    - Connect `GetAlertMessages -> FindAlert`.

15. **Add Slack send node**
    - Create a **Slack** node named `SendReport`.
    - Configure:
      - send a message to the same alerts channel
      - **Text**: `={{ $json.agentOutput }}`
      - set **thread_ts** from `={{ $json.thread_ts }}`
    - Use the same Slack OAuth2 credentials, ensuring write permission is granted.
    - Connect `FindAlert -> SendReport`.

16. **Configure credentials**
    - Required credentials:
      - **HTTP Basic Auth** for webhook protection
      - **OpenAI API**
      - **Slack OAuth2**
      - **Google Gemini / PaLM API**
      - **Qdrant API**
    - External MCP servers may also require their own environment-level tokens or API keys:
      - Kubernetes access
      - Grafana access
      - DigitalOcean access
      - GitLab access

17. **Deploy and verify MCP servers**
    - Ensure the following endpoints are reachable from n8n:
      - `http://mcp-kubernetes:8089/mcp`
      - `http://mcp-grafana:8000/mcp`
      - `http://mcp-digitalocean:8090/mcp`
      - `http://mcp-gitlab:8091/mcp`
    - If running remotely, replace hostnames with actual reachable URLs.

18. **Prepare Slack alert formatting**
    - Ensure the original Slack alert message includes the Alertmanager fingerprint somewhere in:
      - message text
      - or attachment title/text
    - Without the fingerprint, `FindAlert` cannot reliably locate the thread.

19. **Prepare Qdrant collection contents**
    - Create the `observability` collection.
    - Index relevant operational knowledge such as:
      - alert rules
      - runbooks
      - troubleshooting notes
      - service topology descriptions
    - For best results, use the same embedding model during indexing as during retrieval.

20. **Test with a sample Alertmanager payload**
    - Send a test POST request to the webhook.
    - Confirm:
      - `Receive alerts` accepts the request
      - `PreProcessAlerts` emits one item per alert
      - `AI Agent` produces a structured report
      - `GetAlertMessages` can see the target channel
      - `FindAlert` resolves the right `thread_ts`
      - `SendReport` posts in-thread

21. **Activate the workflow**
    - Once credentials, MCP endpoints, and Slack message formatting are confirmed, activate the workflow.

## Reproduction Constraints and Important Notes
- The workflow has **one trigger entry point**: the `Receive alerts` webhook.
- It does **not** invoke any n8n sub-workflow node.
- It does rely on **external MCP services**, which are operational dependencies and must be installed/configured separately.
- If you expect batched Alertmanager payloads with multiple alerts, review the `.first()` usage in `FindAlert` and the AI prompt references, because this can cause per-item mismatches.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow helps automatically analyze alerts occurring in the infrastructure and suggest solutions even before the on-duty engineer sees the alert. | Overview note from canvas |
| The workflow receives an alert from Alertmanager via Webhook. The variables required for operation are set. A prompt is prepared for the agent containing only the data necessary for analysis. The agent performs diagnostics as described in the system prompt. During operation, it can access various systems via MCP to obtain additional information. A message in a Slack channel corresponding to the processed alert is found. A report is sent to the Slack thread. | Overview note from canvas |
| Generate webhook credentials and use them in Alertmanager. Add the alert fingerprint to the Slack message template. Set variables in the SetVars node. Add your own rules and recommendations to the system prompt. Run MCP servers. Choose the Slack channel with alerts. | Usage note from canvas |
| Requirements: OpenAI or Anthropic API key; Slack API key; Google Gemini API key; Webhook receiver in Alertmanager; Webhook token; Qdrant with token; Gitlab Access key. | Requirements note from canvas |
| Add correct URLs for remote MCPs. | MCP setup note |
| grafana-mcp | https://github.com/grafana/mcp-grafana |
| kubernetes-mcp-server | https://github.com/containers/kubernetes-mcp-server |
| digitalocean-mcp | https://github.com/digitalocean/digitalocean-mcp |
| gitlab-mcp | https://github.com/zereight/gitlab-mcp |

## Additional implementation observations
- The prompt mentions **Github tools**, but the workflow actually connects to **GitLab MCP**.
- Slack thread matching currently inspects only the last **10** channel messages.
- The workflow depends on the alert fingerprint being present in the Slack alert message.
- The workflow is currently **inactive** in the exported JSON.
- Execution order is set to `v1`, and binary mode is `separate`.