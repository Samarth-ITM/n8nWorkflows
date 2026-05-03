Analyze Alertmanager incidents and post diagnostic reports to Mattermost

https://n8nworkflows.xyz/workflows/analyze-alertmanager-incidents-and-post-diagnostic-reports-to-mattermost-14347


# Analyze Alertmanager incidents and post diagnostic reports to Mattermost

## 1. Workflow Overview

This workflow receives Alertmanager webhook events, converts each alert into a compact investigation prompt, lets an AI agent analyze the incident using multiple external toolsets, then posts the resulting diagnostic report into the corresponding Mattermost alert thread.

Its main use case is incident triage for infrastructure or application alerts. It is designed for environments using Alertmanager, Mattermost, and observability/tooling sources such as Grafana, Kubernetes, GitHub, DigitalOcean, and a Qdrant knowledge base.

### 1.1 Input Reception and Runtime Variable Setup
This block receives POST requests from Alertmanager through a protected webhook, then injects required runtime variables such as Mattermost host and channel ID.

### 1.2 Alert Preprocessing
This block transforms incoming Alertmanager payloads into one item per alert and builds a structured `chatInput` string for the AI agent.

### 1.3 AI Incident Analysis
This block runs a LangChain agent with a configured chat model and several tool providers:
- Kubernetes MCP
- Grafana MCP
- DigitalOcean MCP
- GitHub MCP
- Qdrant vector store retrieval

The agent is instructed to investigate independently and produce a Mattermost-friendly diagnostic report.

### 1.4 Mattermost Thread Lookup
This block retrieves recent posts from the target Mattermost channel and tries to identify the original Alertmanager post by matching the alert name in attachment titles.

### 1.5 Report Posting
This block posts the generated analysis into the discovered Mattermost thread using the matched root post ID.

### 1.6 Documentation / Visual Guidance
Several sticky notes document requirements, configuration expectations, and external MCP references. These do not execute but are important for maintenance and reproduction.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Runtime Variable Setup

**Overview:**  
This block serves as the workflow entry point. It accepts Alertmanager webhook calls over HTTP Basic Auth and appends environment-specific variables used later by Mattermost and AI prompt expressions.

**Nodes Involved:**  
- Receive alerts
- SetVars

### Node: Receive alerts
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point webhook that receives Alertmanager payloads.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `alertmanager`
  - Authentication: Basic Auth
- **Key expressions or variables used:**  
  None in parameters, but the incoming payload is expected in request body.
- **Input and output connections:**  
  - No input; workflow trigger
  - Output → `SetVars`
- **Version-specific requirements:**  
  Uses typeVersion `2.1`; standard webhook behavior in modern n8n.
- **Edge cases or potential failure types:**  
  - Invalid/missing Basic Auth credentials
  - Alertmanager sending payload in unexpected structure
  - Webhook path not exposed or reverse proxy misconfiguration
  - Alertmanager timeout if downstream execution is too slow
- **Sub-workflow reference:**  
  None

### Node: SetVars
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds static variables needed later in HTTP requests and AI system prompt expressions.
- **Configuration choices:**  
  - Sets `MATTERMOST_HOST` to `mattermost.example.com`
  - Sets `MATTERMOST_CHANNEL_ID` to `dr87mpsdmbfrbxeag7i11acuyw`
  - `includeOtherFields: true`, so original webhook data is preserved
- **Key expressions or variables used:**  
  Produces fields referenced by downstream expressions such as:
  - `$('SetVars').first().json.MATTERMOST_HOST`
  - `$('SetVars').first().json.MATTERMOST_CHANNEL_ID`
  The AI system prompt also references `prometheus_uid` and `loki_uid`, but those variables are not actually set here.
- **Input and output connections:**  
  - Input ← `Receive alerts`
  - Output → `PreProcessAlerts`
- **Version-specific requirements:**  
  Uses typeVersion `3.4`
- **Edge cases or potential failure types:**  
  - Mattermost values left as placeholders
  - Missing additional expected variables such as `prometheus_uid` and `loki_uid`
  - If channel ID is wrong, later Mattermost lookup/posting will fail
- **Sub-workflow reference:**  
  None

---

## 2.2 Alert Preprocessing

**Overview:**  
This block converts the Alertmanager payload into one output item per alert. It builds a concise but information-rich prompt for the AI agent and generates a session identifier based on the alert fingerprint.

**Nodes Involved:**  
- PreProcessAlerts

### Node: PreProcessAlerts
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that normalizes incoming Alertmanager data.
- **Configuration choices:**  
  The code:
  - Reads from `$input.first().json.body` or fallback `$input.first().json`
  - Extracts `alerts`
  - Skips noisy labels such as `alertname`, `endpoint`, `instance`, `job`, `metrics_path`, `prometheus`, `project`, `env`
  - Renames selected labels:
    - `grafana_board` → `Grafana Dashboard`
    - `persistentvolumeclaim` → `PVC`
  - Formats labels and annotations into a text block
  - Creates:
    - `action: "sendMessage"`
    - `chatInput`
    - `sessionId`
- **Key expressions or variables used:**  
  Internal code references:
  - `webhookData.alerts`
  - `alert.labels`
  - `alert.annotations`
  - `alert.fingerprint`
  - `alert.startsAt`
  - `alert.generatorURL`
- **Input and output connections:**  
  - Input ← `SetVars`
  - Output → `AI Agent`
- **Version-specific requirements:**  
  Uses code node typeVersion `2`
- **Edge cases or potential failure types:**  
  - If incoming payload has no `alerts`, output is empty and downstream nodes do not execute meaningfully
  - If labels/annotations are malformed or not objects, formatting may degrade
  - If multiple alerts arrive in one webhook, this node emits multiple items; all downstream steps will execute per alert
  - Session ID fallback may be less unique when fingerprint is absent
- **Sub-workflow reference:**  
  None

---

## 2.3 AI Incident Analysis

**Overview:**  
This block is the analytical core of the workflow. The AI agent receives the formatted alert text and is allowed to use several tool providers to inspect observability, infrastructure, deployment, and stored rule knowledge before generating a structured incident report.

**Nodes Involved:**  
- AI Agent
- OpenAI Chat Model
- Github MCP
- Digitalocean MCP
- K8S mcp
- Grafana mcp
- Qdrant Vector Store
- Embeddings Google Gemini

### Node: AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-style tool-using agent that orchestrates LLM reasoning and external tool calls.
- **Configuration choices:**  
  - Custom system message defines:
    - Role: experienced SRE assistant
    - Tool usage rules
    - Investigation flow
    - Required output format:
      - What happened
      - Event timeline
      - Root cause
      - Troubleshooting tips
  - Explicitly instructs the model not to ask follow-up questions
  - Instructs the model not to retry failed tool calls more than once
- **Key expressions or variables used:**  
  The system prompt includes:
  - `{{ $('SetVars').first().json.prometheus_uid }}`
  - `{{ $('SetVars').first().json.loki_uid }}`
  These variables are referenced but not set in `SetVars`, so they will likely resolve to empty/undefined unless added.
- **Input and output connections:**  
  - Main input ← `PreProcessAlerts`
  - Language model input ← `OpenAI Chat Model`
  - Tool inputs ← `Github MCP`, `Digitalocean MCP`, `K8S mcp`, `Grafana mcp`, `Qdrant Vector Store`
  - Main output → `GetAlertMessages`
- **Version-specific requirements:**  
  Uses typeVersion `3.1`; requires compatible n8n LangChain/AI node support.
- **Edge cases or potential failure types:**  
  - Missing or invalid model credentials
  - Tool endpoint unavailable
  - Prompt expression failure if referenced variables are absent
  - Agent may produce long output exceeding Mattermost preferences
  - External tool latency can significantly slow execution
  - If one tool fails, behavior depends on model compliance with prompt instructions
- **Sub-workflow reference:**  
  None

### Node: OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM used by the AI Agent.
- **Configuration choices:**  
  - Model: `gpt-5.3-codex`
  - No extra options configured
  - Uses OpenAI credentials named `OpenAi account`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output (AI language model) → `AI Agent`
- **Version-specific requirements:**  
  Uses typeVersion `1.3`; model availability depends on the connected OpenAI account and n8n version.
- **Edge cases or potential failure types:**  
  - Model not available for the account
  - API quota/rate limit
  - Authentication failure
  - Cost escalation if used on many alerts
- **Sub-workflow reference:**  
  None

### Node: Github MCP
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes GitHub repository and PR/issue inspection tools to the AI agent.
- **Configuration choices:**  
  - Endpoint URL: `http://mcp-github:8091/mcp`
  - Included tools:
    - repository search
    - file retrieval
    - commit listing
    - issues and PR inspection
- **Key expressions or variables used:**  
  Endpoint URL is expression-based but static in effect.
- **Input and output connections:**  
  - Tool output → `AI Agent`
- **Version-specific requirements:**  
  Uses typeVersion `1.2`; requires accessible MCP server compatible with n8n MCP client tooling.
- **Edge cases or potential failure types:**  
  - MCP server unavailable
  - GitHub token missing at MCP server level
  - Tool responses too broad if org scoping is not enforced server-side
- **Sub-workflow reference:**  
  None

### Node: Digitalocean MCP
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes DigitalOcean inspection functions for apps, droplets, Kubernetes clusters, networking, and related assets.
- **Configuration choices:**  
  - Endpoint URL: `http://mcp-digitalocean:8090/mcp`
  - Timeout: `60000 ms`
  - Includes a large set of DigitalOcean discovery and status tools
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Tool output → `AI Agent`
- **Version-specific requirements:**  
  Uses typeVersion `1.2`
- **Edge cases or potential failure types:**  
  - MCP server unavailable
  - Timeouts on slow API operations
  - Missing DigitalOcean token on MCP server
  - Large/irrelevant results may confuse agent reasoning
- **Sub-workflow reference:**  
  None

### Node: K8S mcp
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes Kubernetes cluster inspection capabilities.
- **Configuration choices:**  
  - Endpoint URL: `http://mcp-kubernetes:8089/mcp`
  - Includes tools for:
    - namespaces
    - resource listing/get
    - pods list/get/log/top
    - nodes stats/top/log
    - events
    - configuration view
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Tool output → `AI Agent`
- **Version-specific requirements:**  
  Uses typeVersion `1.2`
- **Edge cases or potential failure types:**  
  - MCP service unavailable
  - Kubernetes RBAC restrictions
  - Logs unavailable because pod already terminated
  - Resource names inferred incorrectly by the LLM
- **Sub-workflow reference:**  
  None

### Node: Grafana mcp
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes Grafana-related querying capabilities for metrics, logs, dashboards, alerts, panel queries, and deep links.
- **Configuration choices:**  
  - Endpoint URL: `http://mcp-grafana:8000/mcp`
  - Timeout: `60000 ms`
  - Includes Prometheus, Loki, alert rule, dashboard, and investigation tools
- **Key expressions or variables used:**  
  Indirectly depends on the AI system prompt references to datasource UIDs:
  - `prometheus_uid`
  - `loki_uid`
- **Input and output connections:**  
  - Tool output → `AI Agent`
- **Version-specific requirements:**  
  Uses typeVersion `1.2`
- **Edge cases or potential failure types:**  
  - Grafana MCP server unavailable
  - Wrong datasource UID in prompt guidance
  - Prometheus/Loki query errors
  - Dashboard search returning too many irrelevant results
- **Sub-workflow reference:**  
  None

### Node: Qdrant Vector Store
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStoreQdrant`  
  Makes a Qdrant collection available as a retrieval tool for the AI agent.
- **Configuration choices:**  
  - Mode: `retrieve-as-tool`
  - Collection: `observability`
  - Tool description: “Use this tool to fetch knowledge from the database.”
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Embedding input ← `Embeddings Google Gemini`
  - Tool output → `AI Agent`
- **Version-specific requirements:**  
  Uses typeVersion `1.3`
- **Edge cases or potential failure types:**  
  - Invalid Qdrant credentials
  - Missing collection
  - Embedding mismatch with indexed vectors
  - Retrieval quality depends on collection content quality
- **Sub-workflow reference:**  
  None

### Node: Embeddings Google Gemini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`  
  Supplies embedding generation to the Qdrant retrieval node.
- **Configuration choices:**  
  - Model: `models/gemini-embedding-001`
  - Uses Google Gemini / PaLM credentials
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output (AI embedding) → `Qdrant Vector Store`
- **Version-specific requirements:**  
  Uses typeVersion `1`
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Rate limit
  - Embedding model mismatch with pre-indexed content representation strategy
- **Sub-workflow reference:**  
  None

---

## 2.4 Mattermost Thread Lookup

**Overview:**  
This block finds the likely original alert post in Mattermost so the generated report can be posted as a threaded reply instead of a standalone message.

**Nodes Involved:**  
- GetAlertMessages
- FindAlert

### Node: GetAlertMessages
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries the Mattermost REST API for recent channel posts.
- **Configuration choices:**  
  - Method implied by node default for retrieval; configured with URL only
  - URL:
    `https://<MATTERMOST_HOST>/api/v4/channels/<MATTERMOST_CHANNEL_ID>/posts`
  - Query parameter: `per_page=10`
  - Authentication: predefined credential type `mattermostApi`
- **Key expressions or variables used:**  
  - `$('SetVars').first().json.MATTERMOST_HOST`
  - `$('SetVars').first().json.MATTERMOST_CHANNEL_ID`
- **Input and output connections:**  
  - Input ← `AI Agent`
  - Output → `FindAlert`
- **Version-specific requirements:**  
  Uses typeVersion `4.4`
- **Edge cases or potential failure types:**  
  - Invalid Mattermost API credentials
  - Wrong hostname or channel ID
  - Only last 10 posts are searched, which may miss the target alert thread
  - Network/TLS issues
- **Sub-workflow reference:**  
  None

### Node: FindAlert
- **Type and technical role:** `n8n-nodes-base.code`  
  Correlates AI output with Mattermost posts and extracts the thread root ID.
- **Configuration choices:**  
  The code:
  - Reads AI output from `$('AI Agent').first().json`
  - Reads Mattermost posts from direct input
  - Parses `chatInput` to extract the `Alertname`
  - Scans post attachments for titles containing that alert name
  - Uses `post.root_id || postId` as `thread_root_id`
  - Returns:
    - `thread_root_id`
    - `message`
    - `alertname`
- **Key expressions or variables used:**  
  - `$('AI Agent').first().json`
  - `$input.first().json`
- **Input and output connections:**  
  - Input ← `GetAlertMessages`
  - Output → `Post in thread`
- **Version-specific requirements:**  
  Uses typeVersion `2`
- **Edge cases or potential failure types:**  
  - If `AI Agent` did not run, code catches error and proceeds with empty values
  - Matching based only on `alertname` may collide across repeated alerts
  - If Mattermost alert posts do not use attachments or title format differs, no thread will be found
  - If `per_page=10` does not include the alert post, `thread_root_id` remains empty
- **Sub-workflow reference:**  
  None

---

## 2.5 Report Posting

**Overview:**  
This block sends the final AI-generated diagnostic report to Mattermost, targeting the matched thread when available.

**Nodes Involved:**  
- Post in thread

### Node: Post in thread
- **Type and technical role:** `n8n-nodes-base.mattermost`  
  Posts a message into a Mattermost channel, optionally as a threaded reply.
- **Configuration choices:**  
  - Message: `{{ $('AI Agent').item }}`
  - Channel ID: `{{ $('SetVars').first().json.MATTERMOST_CHANNEL_ID }}`
  - `root_id`: `{{ $json.thread_root_id }}`
  - Attachments: empty
- **Key expressions or variables used:**  
  - `$('AI Agent').item`
  - `$('SetVars').first().json.MATTERMOST_CHANNEL_ID`
  - `$json.thread_root_id`
- **Input and output connections:**  
  - Input ← `FindAlert`
  - No downstream connection
- **Version-specific requirements:**  
  Uses typeVersion `1`
- **Edge cases or potential failure types:**  
  - The message expression is likely incorrect or at least imprecise. Usually the intended field would be something like `$('AI Agent').first().json.output` or `$json.message`. As written, it may stringify the whole item object rather than the generated text.
  - If `thread_root_id` is empty, behavior depends on Mattermost node/API handling; it may post as a normal channel message or fail
  - Invalid channel ID or credentials
- **Sub-workflow reference:**  
  None

---

## 2.6 Documentation / Visual Guidance

**Overview:**  
These nodes are non-executable notes that explain configuration requirements, workflow flow, and MCP server references. They are important for operators rebuilding or maintaining the workflow.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation for the input section.
- **Configuration choices:**  
  Content says the section is the input chain and instructs the user to add Mattermost server address and channel ID into `SetVars`.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Uses typeVersion `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the output chain area.
- **Configuration choices:**  
  Content: `# Output chain`
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  typeVersion `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the AI analysis section.
- **Configuration choices:**  
  Content: `# Alert analysis`
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  typeVersion `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents MCP server prerequisites and links.
- **Configuration choices:**  
  Content instructs to add correct URLs for remote MCPs and references:
  - [grafana-mcp](https://github.com/grafana/mcp-grafana)
  - [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)
  - [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)
  - github-mcp
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  typeVersion `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  High-level operational note covering requirements and usage.
- **Configuration choices:**  
  Documents:
  - workflow purpose
  - required credentials/services
  - how it works
  - how to use it
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  typeVersion `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive alerts | Webhook | Receives Alertmanager POST payloads with Basic Auth protection |  | SetVars | # Input chain<br>Add your Mattermost server address and channel id into SetVars<br># Overview<br>Workflow receives alert from Alertmanager, analize it and send report to Mattermost<br>## Requirements:<br>1. OpenAI  or Anthropic API key<br>2. Mattermost API key<br>3. Google Gemini API key<br>4. Webhook receiver in Alertmanager<br>5. Webhook token<br>6. Qdrant with token<br>7. Github token<br>8. Digitalocean token<br>## How it work<br>1. Workflow receives alert from Alertmanager via Webhook.<br>2. The variables required for operation are set<br>3. Preparing a prompt for the agent containing only the data necessary for analysis<br>4. The agent performs diagnostics as described in the system prompt. During operation, it can access various systems via MCP to obtain additional information.<br>5. Search for a message in a Mattermost channel corresponding to a processed alert<br>6. Send report to Mattermost thread.<br>## How to use<br>1. Generate webhook credentials and use it in Alertmanager<br>2. Add Alert fingerprint into Slack message template<br>3. Set variables it SetVars node<br>4. Add your own Rules and recomendations to system promt<br>5 Run mcp servers<br>6. Choose Mattermost channel with alerts |
| SetVars | Set | Injects Mattermost host and channel settings while preserving input payload | Receive alerts | PreProcessAlerts | # Input chain<br>Add your Mattermost server address and channel id into SetVars<br># Overview<br>Workflow receives alert from Alertmanager, analize it and send report to Mattermost<br>## Requirements:<br>1. OpenAI  or Anthropic API key<br>2. Mattermost API key<br>3. Google Gemini API key<br>4. Webhook receiver in Alertmanager<br>5. Webhook token<br>6. Qdrant with token<br>7. Github token<br>8. Digitalocean token<br>## How it work<br>1. Workflow receives alert from Alertmanager via Webhook.<br>2. The variables required for operation are set<br>3. Preparing a prompt for the agent containing only the data necessary for analysis<br>4. The agent performs diagnostics as described in the system prompt. During operation, it can access various systems via MCP to obtain additional information.<br>5. Search for a message in a Mattermost channel corresponding to a processed alert<br>6. Send report to Mattermost thread.<br>## How to use<br>1. Generate webhook credentials and use it in Alertmanager<br>2. Add Alert fingerprint into Slack message template<br>3. Set variables it SetVars node<br>4. Add your own Rules and recomendations to system promt<br>5 Run mcp servers<br>6. Choose Mattermost channel with alerts |
| PreProcessAlerts | Code | Converts webhook payload into one AI-ready item per alert | SetVars | AI Agent | # Input chain<br>Add your Mattermost server address and channel id into SetVars<br># Overview<br>Workflow receives alert from Alertmanager, analize it and send report to Mattermost<br>## Requirements:<br>1. OpenAI  or Anthropic API key<br>2. Mattermost API key<br>3. Google Gemini API key<br>4. Webhook receiver in Alertmanager<br>5. Webhook token<br>6. Qdrant with token<br>7. Github token<br>8. Digitalocean token<br>## How it work<br>1. Workflow receives alert from Alertmanager via Webhook.<br>2. The variables required for operation are set<br>3. Preparing a prompt for the agent containing only the data necessary for analysis<br>4. The agent performs diagnostics as described in the system prompt. During operation, it can access various systems via MCP to obtain additional information.<br>5. Search for a message in a Mattermost channel corresponding to a processed alert<br>6. Send report to Mattermost thread.<br>## How to use<br>1. Generate webhook credentials and use it in Alertmanager<br>2. Add Alert fingerprint into Slack message template<br>3. Set variables it SetVars node<br>4. Add your own Rules and recomendations to system promt<br>5 Run mcp servers<br>6. Choose Mattermost channel with alerts |
| AI Agent | LangChain Agent | Performs incident investigation using LLM and external tools | PreProcessAlerts, OpenAI Chat Model, Github MCP, Digitalocean MCP, K8S mcp, Grafana mcp, Qdrant Vector Store | GetAlertMessages | # Alert analysis |
| OpenAI Chat Model | OpenAI Chat Model | Supplies the LLM backend for the agent |  | AI Agent | # Alert analysis |
| Github MCP | MCP Client Tool | Provides GitHub repository, commit, issue, and PR inspection tools |  | AI Agent | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana)<br>* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)<br>* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)<br>* [github-mcp] |
| Digitalocean MCP | MCP Client Tool | Provides DigitalOcean environment inspection tools |  | AI Agent | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana)<br>* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)<br>* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)<br>* [github-mcp] |
| K8S mcp | MCP Client Tool | Provides Kubernetes cluster inspection tools |  | AI Agent | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana)<br>* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)<br>* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)<br>* [github-mcp] |
| Grafana mcp | MCP Client Tool | Provides Grafana, Prometheus, Loki, and dashboard investigation tools |  | AI Agent | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana)<br>* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)<br>* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)<br>* [github-mcp] |
| Qdrant Vector Store | Qdrant Vector Store | Exposes knowledge retrieval as an AI tool | Embeddings Google Gemini | AI Agent | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana)<br>* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)<br>* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)<br>* [github-mcp] |
| Embeddings Google Gemini | Google Gemini Embeddings | Generates embeddings for vector retrieval |  | Qdrant Vector Store | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana)<br>* [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server)<br>* [digitalocean-mcp](https://github.com/digitalocean/digitalocean-mcp)<br>* [github-mcp] |
| GetAlertMessages | HTTP Request | Fetches recent Mattermost posts from the alert channel | AI Agent | FindAlert | # Output chain |
| FindAlert | Code | Matches the analyzed alert to an existing Mattermost alert thread | GetAlertMessages | Post in thread | # Output chain |
| Post in thread | Mattermost | Posts the AI diagnostic message into the Mattermost thread | FindAlert |  | # Output chain |
| Sticky Note | Sticky Note | Visual documentation for input area |  |  |  |
| Sticky Note1 | Sticky Note | Visual documentation for output area |  |  |  |
| Sticky Note2 | Sticky Note | Visual documentation for AI section |  |  |  |
| Sticky Note3 | Sticky Note | Visual documentation for MCP prerequisites |  |  |  |
| Sticky Note4 | Sticky Note | Visual documentation for overview, requirements, and usage |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `AlertAssistant`.
   - Keep it inactive until credentials, URLs, and outputs are validated.

2. **Add the webhook trigger**
   - Create a **Webhook** node named `Receive alerts`.
   - Set:
     - HTTP Method: `POST`
     - Path: `alertmanager`
     - Authentication: `Basic Auth`
   - Create Basic Auth credentials for Alertmanager, for example `alertmanager-webhook`.
   - In Alertmanager, configure the webhook receiver to call this n8n endpoint.

3. **Add a Set node for environment variables**
   - Create a **Set** node named `SetVars`.
   - Enable “Include Other Input Fields”.
   - Add at minimum:
     - `MATTERMOST_HOST` as string, e.g. `mattermost.example.com`
     - `MATTERMOST_CHANNEL_ID` as string, e.g. your channel ID
   - Recommended: also add
     - `prometheus_uid`
     - `loki_uid`
     because the AI system prompt references them.
   - Connect `Receive alerts` → `SetVars`.

4. **Add the alert preprocessing logic**
   - Create a **Code** node named `PreProcessAlerts`.
   - Connect `SetVars` → `PreProcessAlerts`.
   - Paste JavaScript that:
     - reads from `body` or root json
     - loops over `alerts`
     - builds one output item per alert
     - constructs `chatInput`
     - constructs `sessionId` using `fingerprint` if available
   - The output structure should include:
     - `action`
     - `chatInput`
     - `sessionId`

5. **Add the AI agent**
   - Create an **AI Agent** node named `AI Agent`.
   - Connect `PreProcessAlerts` → `AI Agent`.
   - In the system message, paste the SRE-oriented instruction set from the workflow:
     - explain available tools
     - explain investigation flow
     - define strict behavior on tool failures
     - require Mattermost-friendly output sections:
       - What happened
       - Event timeline
       - Root cause
       - Troubleshooting tips
   - If you want Grafana datasource hints to work, make sure your prompt references valid values from `SetVars`, e.g. `prometheus_uid` and `loki_uid`.

6. **Add the LLM node**
   - Create an **OpenAI Chat Model** node named `OpenAI Chat Model`.
   - Select model `gpt-5.3-codex` or another supported model available in your account.
   - Configure OpenAI credentials.
   - Connect this node to the AI Agent using the **AI Language Model** connection.

7. **Add the GitHub MCP tool**
   - Create an **MCP Client Tool** node named `Github MCP`.
   - Set endpoint URL to your GitHub MCP server, e.g. `http://mcp-github:8091/mcp`.
   - Include these tools:
     - `search_repositories`
     - `get_file_contents`
     - `list_commits`
     - `list_issues`
     - `search_issues`
     - `get_issue`
     - `get_pull_request`
     - `list_pull_requests`
     - `get_pull_request_files`
     - `get_pull_request_status`
     - `get_pull_request_comments`
     - `get_pull_request_reviews`
   - Connect it to `AI Agent` as an **AI Tool** input.

8. **Add the DigitalOcean MCP tool**
   - Create another **MCP Client Tool** node named `Digitalocean MCP`.
   - Set endpoint URL to your DigitalOcean MCP server, e.g. `http://mcp-digitalocean:8090/mcp`.
   - Set timeout to `60000 ms`.
   - Include the listed infrastructure and app tools from the source workflow.
   - Connect it to `AI Agent` as an **AI Tool** input.

9. **Add the Kubernetes MCP tool**
   - Create an **MCP Client Tool** node named `K8S mcp`.
   - Set endpoint URL to your Kubernetes MCP server, e.g. `http://mcp-kubernetes:8089/mcp`.
   - Include:
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
   - Connect it to `AI Agent` as an **AI Tool** input.

10. **Add the Grafana MCP tool**
    - Create an **MCP Client Tool** node named `Grafana mcp`.
    - Set endpoint URL to your Grafana MCP server, e.g. `http://mcp-grafana:8000/mcp`.
    - Set timeout to `60000 ms`.
    - Include:
      - dashboard tools
      - alert rule tools
      - datasource lookup tools
      - Prometheus query tools
      - Loki query tools
      - deep link / investigation tools
    - Connect it to `AI Agent` as an **AI Tool** input.

11. **Add the embeddings node**
    - Create a **Google Gemini Embeddings** node named `Embeddings Google Gemini`.
    - Choose model `models/gemini-embedding-001`.
    - Configure Google Gemini credentials.

12. **Add the Qdrant vector retrieval node**
    - Create a **Qdrant Vector Store** node named `Qdrant Vector Store`.
    - Set:
      - Mode: `Retrieve as Tool`
      - Collection: `observability`
      - Tool description: a short phrase like “Use this tool to fetch knowledge from the database.”
    - Configure Qdrant credentials.
    - Connect `Embeddings Google Gemini` to `Qdrant Vector Store` via the **AI Embedding** connection.
    - Connect `Qdrant Vector Store` to `AI Agent` via the **AI Tool** connection.

13. **Add Mattermost channel post retrieval**
    - Create an **HTTP Request** node named `GetAlertMessages`.
    - Connect `AI Agent` → `GetAlertMessages`.
    - Configure:
      - URL: `https://{{$('SetVars').first().json.MATTERMOST_HOST}}/api/v4/channels/{{$('SetVars').first().json.MATTERMOST_CHANNEL_ID}}/posts`
      - Authentication: `Predefined Credential Type`
      - Credential type: `Mattermost API`
      - Query parameter:
        - `per_page = 10`
    - Configure Mattermost credentials with an API token.

14. **Add thread correlation logic**
    - Create a **Code** node named `FindAlert`.
    - Connect `GetAlertMessages` → `FindAlert`.
    - Add JavaScript that:
      - reads AI output from the `AI Agent`
      - extracts the `Alertname:` line from `chatInput`
      - scans recent Mattermost posts
      - checks `props.attachments[].title`
      - if the attachment title contains the alert name, sets `thread_root_id`
      - returns:
        - `thread_root_id`
        - `message`
        - `alertname`

15. **Add Mattermost posting**
    - Create a **Mattermost** node named `Post in thread`.
    - Connect `FindAlert` → `Post in thread`.
    - Configure:
      - Channel ID: `{{$('SetVars').first().json.MATTERMOST_CHANNEL_ID}}`
      - Root ID: `{{$json.thread_root_id}}`
    - For the message, prefer:
      - `{{$json.message}}`
      instead of the original workflow’s `{{$('AI Agent').item}}`, which may serialize the whole item incorrectly.
    - Use the same Mattermost credentials as the HTTP node if supported.

16. **Optional but strongly recommended fixes**
    - In `SetVars`, add:
      - `prometheus_uid`
      - `loki_uid`
    - In `GetAlertMessages`, increase `per_page` if your channel is busy.
    - In `FindAlert`, improve matching by using:
      - alert fingerprint
      - labels like namespace/service
      - creation time
    - In `Post in thread`, ensure message uses the exact AI text field.

17. **Create and configure credentials**
    - **HTTP Basic Auth** for the webhook
    - **OpenAI API**
    - **Mattermost API**
    - **Google Gemini API**
    - **Qdrant API**
    - MCP server-side credentials for:
      - GitHub
      - DigitalOcean
      - Kubernetes access
      - Grafana / Prometheus / Loki access

18. **Prepare external services**
    - Deploy or expose these MCP services:
      - Grafana MCP
      - Kubernetes MCP server
      - DigitalOcean MCP
      - GitHub MCP
    - Ensure n8n can reach them over the configured URLs.
    - Ensure each MCP server is already authenticated to its underlying platform.

19. **Prepare Alertmanager formatting**
    - Configure Alertmanager to send alerts to the n8n webhook.
    - The workflow notes recommend including the alert fingerprint in the originating chat message template so threading correlation is more reliable.

20. **Test with a sample alert**
    - Send a realistic Alertmanager payload containing:
      - `alerts[]`
      - `labels.alertname`
      - `labels.severity`
      - `annotations`
      - `fingerprint`
      - `startsAt`
      - `generatorURL`
    - Verify:
      - one item is created per alert
      - AI output is generated
      - Mattermost recent posts are fetched
      - correct thread is identified
      - the report appears in the desired thread

21. **Activate the workflow**
    - Only activate after:
      - webhook auth is confirmed
      - Mattermost posting works
      - MCP URLs resolve
      - AI output formatting is acceptable

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Add your Mattermost server address and channel id into SetVars | Workflow configuration note |
| Output chain | Visual grouping note |
| Alert analysis | Visual grouping note |
| Add correct URLs for remote MCPs | MCP server configuration note |
| grafana-mcp | https://github.com/grafana/mcp-grafana |
| kubernetes-mcp-server | https://github.com/containers/kubernetes-mcp-server |
| digitalocean-mcp | https://github.com/digitalocean/digitalocean-mcp |
| github-mcp | Referenced in sticky note, no URL provided in workflow |
| Workflow receives alert from Alertmanager, analize it and send report to Mattermost | Overview note |
| Requirements: OpenAI or Anthropic API key, Mattermost API key, Google Gemini API key, webhook receiver in Alertmanager, webhook token, Qdrant with token, Github token, Digitalocean token | Project requirements note |
| How it work: receive alert, set variables, prepare prompt, run diagnostics through MCP tools, search Mattermost message, send report to thread | Operational summary |
| How to use: generate webhook credentials, add alert fingerprint into Slack message template, set variables in SetVars, add your own rules and recommendations to system prompt, run MCP servers, choose Mattermost channel with alerts | Usage note |

### Important implementation observations
- The workflow title provided by the user is “Analyze Alertmanager incidents and post diagnostic reports to Mattermost”, while the internal workflow name in JSON is `AlertAssistant`.
- The workflow currently references `prometheus_uid` and `loki_uid` in the AI system prompt, but these values are not set anywhere.
- The final Mattermost message expression should likely be corrected from `{{$('AI Agent').item}}` to a concrete field such as `{{$json.message}}` or `{{$('AI Agent').first().json.output}}`.
- Thread lookup only scans the latest 10 Mattermost posts, which may be insufficient in active channels.
- The sticky note mentions adding an alert fingerprint into a “Slack message template”; operationally this appears intended to improve source alert correlation, but the actual destination in this workflow is Mattermost.