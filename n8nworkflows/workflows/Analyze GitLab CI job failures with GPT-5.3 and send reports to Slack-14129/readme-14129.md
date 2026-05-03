Analyze GitLab CI job failures with GPT-5.3 and send reports to Slack

https://n8nworkflows.xyz/workflows/analyze-gitlab-ci-job-failures-with-gpt-5-3-and-send-reports-to-slack-14129


# Analyze GitLab CI job failures with GPT-5.3 and send reports to Slack

# 1. Workflow Overview

This workflow listens for GitLab pipeline/job events, detects failed jobs, gathers the most relevant failure context, asks an AI agent to analyze the failure, and posts the resulting diagnostic report to Slack.

Its primary use case is automated first-line CI/CD incident analysis for GitLab pipelines, so DevOps engineers do not need to manually inspect every failed job. The workflow is designed to focus on actual failed jobs, collect the job trace and CI configuration, optionally let the AI query GitLab and Grafana through MCP tools, and then deliver a concise report to a Slack channel.

## 1.1 Input Reception and Failure Filtering
The workflow starts with a secured webhook that receives GitLab events. It immediately checks whether the incoming event represents a failed pipeline/job status and stops early if it does not.

## 1.2 Context Preparation
For failed events, the workflow sets a few reusable variables and transforms the GitLab webhook payload into a normalized structure. It extracts pipeline metadata and identifies failed jobs that should be investigated.

## 1.3 Failure Evidence Collection
For each failed job, the workflow fetches the raw GitLab job trace and the `.gitlab-ci.yml` file used for the commit. It trims the job log down to the last 50 relevant lines and combines all inputs into a single payload.

## 1.4 AI-Based Analysis
The merged payload is sent to an AI Agent backed by an OpenAI chat model. The agent is instructed to analyze failed CI/CD jobs, suggest up to two likely root causes, and provide troubleshooting steps. It can also use MCP tools for GitLab and Grafana, plus a generic HTTP request tool, if it needs more context.

## 1.5 Notification Output
The AI-generated report is sent to a predefined Slack channel.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Failure Filtering

### Overview
This block receives GitLab webhook events and ensures the rest of the workflow only runs for failed jobs/pipelines. Non-failed events are routed to a no-op endpoint so the execution ends without additional processing.

### Nodes Involved
- Get Job Events
- Is job failed?
- No Operation, do nothing

### Node Details

#### Get Job Events
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for incoming GitLab webhook POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: auto-generated webhook path
  - Authentication: header-based authentication
- **Key expressions or variables used:** None in parameters; the payload is consumed downstream from `$json.body`.
- **Input and output connections:**
  - No input; this is the trigger node.
  - Output goes to **Is job failed?**
- **Version-specific requirements:** `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Invalid or missing `X-Gitlab-Token` header
  - GitLab not configured to send the expected event payload
  - Webhook path mismatch between GitLab and n8n
  - Payload structure changes could break downstream expressions
- **Sub-workflow reference:** None

#### Is job failed?
- **Type and technical role:** `n8n-nodes-base.if`  
  Filters events based on GitLab status.
- **Configuration choices:**
  - Compares `{{$json.body.object_attributes.status}}` to the string `failed`
  - Strict type validation enabled
  - Single AND condition
- **Key expressions or variables used:**
  - `={{ $json.body.object_attributes.status }}`
- **Input and output connections:**
  - Input from **Get Job Events**
  - True output to **SetVars**
  - False output to **No Operation, do nothing**
- **Version-specific requirements:** `typeVersion 2.3`
- **Edge cases or potential failure types:**
  - If `body.object_attributes.status` is missing, the condition simply evaluates false
  - Event types other than pipeline/job events may be silently ignored
  - If GitLab sends a status other than exactly `failed` for relevant incidents, the workflow will not continue
- **Sub-workflow reference:** None

#### No Operation, do nothing
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Explicit terminal path for non-failed events.
- **Configuration choices:** No parameters
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from the false branch of **Is job failed?**
  - No output connections
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None meaningful; this node intentionally does nothing
- **Sub-workflow reference:** None

---

## 2.2 Context Preparation

### Overview
This block enriches the incoming event with reusable settings and converts the GitLab webhook payload into one item per failed job. It also filters out jobs that are failed but marked `allow_failure`, so the analysis targets failures that actually matter.

### Nodes Involved
- SetVars
- Extract event data

### Node Details

#### SetVars
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds environment-specific constants used later in the AI system prompt.
- **Configuration choices:**
  - Preserves all incoming fields
  - Adds:
    - `prometheus_name = "dev"`
    - `loki_name = "grafanacloud-logs"`
    - `grafana_group = "TL"`
- **Key expressions or variables used:** Static assignments only
- **Input and output connections:**
  - Input from the true branch of **Is job failed?**
  - Output to **Extract event data**
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - Values are hardcoded; incorrect Grafana datasource names will mislead the AI
  - The AI prompt also references `gitlab_group`, but this node does not define it
- **Sub-workflow reference:** None

#### Extract event data
- **Type and technical role:** `n8n-nodes-base.code` using native Python  
  Parses the webhook body, extracts pipeline context, and emits one output item per relevant failed job.
- **Configuration choices:**
  - Reads from `_items[0]["json"]["body"]`
  - Pulls data from:
    - `object_attributes`
    - `project`
    - `commit`
    - `user`
    - `merge_request`
    - `builds`
  - Filters `builds` to include only:
    - `status == "failed"`
    - `allow_failure != true`
  - Returns one item per failed job with:
    - `pipeline`
    - `failed_job`
- **Key expressions or variables used:**
  - Python-native item access rather than n8n expressions
  - Builds `gitlab_base_url` from `project.web_url`
- **Input and output connections:**
  - Input from **SetVars**
  - Outputs in parallel to:
    - **GetJobLogs**
    - **GetCIFile**
    - **Merge** input 1
- **Version-specific requirements:** `typeVersion 2`
  - Requires Python-native code support in the n8n instance
- **Edge cases or potential failure types:**
  - If `builds` is missing or empty, the code returns `[]`, ending processing
  - If `project.web_url` is absent, defaults to `https://gitlab.com`
  - The node extracts `commit`, `user`, and `merge_request`, but only `merge_request` is conditionally used
  - If the webhook event format differs from GitLab Pipeline Hook expectations, fields may be empty
  - The output drops some potentially useful context such as commit message or user identity
- **Sub-workflow reference:** None

---

## 2.3 Failure Evidence Collection

### Overview
This block gathers the technical evidence needed for analysis: the raw job trace and the `.gitlab-ci.yml` content for the failing commit. It then reduces the trace to the most useful tail section and merges all information together.

### Nodes Involved
- GetJobLogs
- GetLastLogLines
- GetCIFile
- Merge
- FormatData

### Node Details

#### GetJobLogs
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the GitLab API to retrieve the full trace of the failed job.
- **Configuration choices:**
  - URL:
    - `{{$json.pipeline.gitlab_base_url}}/api/v4/projects/{{$json.pipeline.project_id}}/jobs/{{$json.failed_job.job_id}}/trace`
  - Authentication: predefined credential type
  - Credential type selected: `gitlabApi`
- **Key expressions or variables used:**
  - `pipeline.gitlab_base_url`
  - `pipeline.project_id`
  - `failed_job.job_id`
- **Input and output connections:**
  - Input from **Extract event data**
  - Output to **GetLastLogLines**
- **Version-specific requirements:** `typeVersion 4.4`
- **Edge cases or potential failure types:**
  - Invalid GitLab token or insufficient permissions
  - Logs may no longer be available if retention expired
  - Very large logs can increase memory use before trimming
  - URL encoding issues are unlikely for numeric IDs but possible if base URL is malformed
  - The node contains an unused `githubApi` credential entry in JSON; operationally the configured GitLab credential is what matters
- **Sub-workflow reference:** None

#### GetLastLogLines
- **Type and technical role:** `n8n-nodes-base.code` using native Python  
  Cleans ANSI escape sequences and GitLab section markers from the trace, then keeps only the last 50 non-empty lines.
- **Configuration choices:**
  - Reads raw log from `_items[0]["json"].get("data", "")`
  - Removes:
    - ANSI CSI sequences
    - ANSI OSC hyperlink sequences
    - `section_start:` / `section_end:` lines
    - blank lines
  - Produces:
    - `job_log`
- **Key expressions or variables used:** Python-native parsing
- **Input and output connections:**
  - Input from **GetJobLogs**
  - Output to **Merge** input 0
- **Version-specific requirements:** `typeVersion 2`
  - Requires Python-native code support
- **Edge cases or potential failure types:**
  - If the HTTP response shape changes and `data` is absent, the result becomes an empty log
  - Only the last 50 lines are kept, so root cause details earlier in the log may be lost
  - ANSI stripping is manual and may not catch all escape patterns
  - The node writes `job_log_total_lines` in FormatData logic, but this node does not provide it
- **Sub-workflow reference:** None

#### GetCIFile
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the raw `.gitlab-ci.yml` file content for the commit associated with the failed pipeline.
- **Configuration choices:**
  - URL:
    - `{{$json.pipeline.gitlab_base_url}}/api/v4/projects/{{$json.pipeline.project_id}}/repository/files/.gitlab-ci.yml/raw`
  - Query parameter:
    - `ref = {{$json.pipeline.commit_sha}}`
  - Authentication: predefined GitLab API credential
- **Key expressions or variables used:**
  - `pipeline.gitlab_base_url`
  - `pipeline.project_id`
  - `pipeline.commit_sha`
- **Input and output connections:**
  - Input from **Extract event data**
  - Output to **Merge** input 2
- **Version-specific requirements:** `typeVersion 4.4`
- **Edge cases or potential failure types:**
  - Missing `.gitlab-ci.yml` at that revision
  - Monorepo or included CI setup may make the raw root file insufficient
  - GitLab permission issues
  - The pipeline may use includes, remote configs, or generated child pipelines not visible in this single file
- **Sub-workflow reference:** None

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines three parallel input streams: cleaned log, original failed-job context, and CI file content.
- **Configuration choices:**
  - Number of inputs: `3`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input 0 from **GetLastLogLines**
  - Input 1 from **Extract event data**
  - Input 2 from **GetCIFile**
  - Output to **FormatData**
- **Version-specific requirements:** `typeVersion 3.2`
- **Edge cases or potential failure types:**
  - Merging multiple-item branches can produce unexpected item pairing if counts differ
  - If one branch fails or returns fewer items, downstream formatting may miss fields
  - Behavior depends on n8n merge semantics for multi-input synchronization in this version
- **Sub-workflow reference:** None

#### FormatData
- **Type and technical role:** `n8n-nodes-base.code` using native Python  
  Converts the merged set of items into one consolidated JSON object suitable for the AI Agent.
- **Configuration choices:**
  - Iterates over `_items`
  - Detects item types by field presence:
    - `job_log`
    - `pipeline`
    - `failed_job`
    - raw CI file under `data`
    - optional `diff`
  - Outputs one object with keys:
    - `job_log`
    - `job_log_total_lines`
    - `pipeline`
    - `failed_job`
    - `ci_config`
    - optional `commit_diff`
- **Key expressions or variables used:** Python-native field inspection
- **Input and output connections:**
  - Input from **Merge**
  - Output to **AI Agent**
- **Version-specific requirements:** `typeVersion 2`
  - Requires Python-native code support
- **Edge cases or potential failure types:**
  - `job_log_total_lines` defaults to `0` because upstream node does not provide it
  - If multiple merged items are not aligned as expected, values may overwrite each other
  - `commit_diff` logic is currently unused by upstream nodes
  - If GitLab API responses return binary or unexpected formats, the `data` field assumptions may fail
- **Sub-workflow reference:** None

---

## 2.4 AI-Based Analysis

### Overview
This block sends the assembled failure context to a language-model-powered agent. The agent is guided by a detailed system prompt and can optionally use MCP tools and an HTTP request tool to retrieve more diagnostic context before generating a final report.

### Nodes Involved
- OpenAI Chat Model
- AI Agent
- Gitlab MCP
- MCP Grafana
- HTTP Request

### Node Details

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the underlying LLM for the AI Agent.
- **Configuration choices:**
  - Model: `gpt-5.3-codex`
  - No extra model options configured
  - Uses OpenAI API credentials
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to **AI Agent** through the `ai_languageModel` port
- **Version-specific requirements:** `typeVersion 1.3`
  - Requires compatible n8n LangChain integration
  - Requires the specified model to be available on the OpenAI account
- **Edge cases or potential failure types:**
  - Invalid OpenAI credentials
  - Model unavailability or account access restrictions
  - Token/context limits if logs or CI config are large
  - API latency or rate limits
- **Sub-workflow reference:** None

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Central reasoning node that analyzes the failure payload and generates a structured report.
- **Configuration choices:**
  - Prompt source text:
    - `={{ JSON.stringify($json, null, 2) }}`
  - Prompt type: defined directly in the node
  - System message instructs the agent to:
    - analyze unsuccessful CI/CD jobs
    - produce 1 to 3 probable causes and solutions
    - not modify anything through MCP
    - inspect `job_log` first
    - output sections:
      - What happened
      - Root cause
      - Troubleshooting tips
  - Prompt references:
    - `{{ $('SetVars').first().json.gitlab_group }}`
    - `{{ $('SetVars').first().json.prometheus_name }}`
    - `{{ $('SetVars').first().json.loki_name }}`
- **Key expressions or variables used:**
  - `JSON.stringify($json, null, 2)`
  - `$('SetVars').first().json.*`
- **Input and output connections:**
  - Main input from **FormatData**
  - Language model input from **OpenAI Chat Model**
  - Tool inputs from:
    - **Gitlab MCP**
    - **MCP Grafana**
    - **HTTP Request**
  - Main output to **Send a message**
- **Version-specific requirements:** `typeVersion 3.1`
  - Requires AI Agent support in the n8n instance
- **Edge cases or potential failure types:**
  - The prompt references `gitlab_group`, but **SetVars** does not define it; this expression may resolve to undefined/empty
  - The system prompt says input format is an array, but the actual payload is a single JSON object
  - If MCP endpoints are unavailable, tool calls may fail
  - If the model generates output under a field other than `output`, Slack mapping may fail depending on AI Agent output schema
  - Long CI files plus logs can cause context overrun
- **Sub-workflow reference:** None

#### Gitlab MCP
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes selected GitLab MCP tools to the AI Agent for read-only repository and pipeline investigation.
- **Configuration choices:**
  - Endpoint URL: `http://gitlab-mcp:8091/mcp`
  - Includes selected tools such as:
    - repository search and listing
    - file retrieval
    - merge request data
    - branch diffs
    - projects and namespaces
    - commits
    - repository tree
    - pipelines and jobs
    - pipeline job output
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to **AI Agent** through `ai_tool`
- **Version-specific requirements:** `typeVersion 1.2`
  - Requires MCP-compatible n8n environment
  - Requires the external GitLab MCP server to be running and reachable
- **Edge cases or potential failure types:**
  - MCP server unreachable or misconfigured
  - Tool list and server capabilities may drift over time
  - Auth/permission constraints on the MCP backend
  - Despite prompt restrictions, actual write protection depends on MCP server/tool design
- **Sub-workflow reference:** None

#### MCP Grafana
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes Grafana/Prometheus/Loki diagnostic tools to the AI Agent.
- **Configuration choices:**
  - Endpoint URL: `http://mcp-grafana:8000/mcp`
  - Includes selected tools such as:
    - datasource listing
    - dashboard search and summaries
    - Prometheus queries
    - Loki logs and patterns
    - panel images
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to **AI Agent** through `ai_tool`
- **Version-specific requirements:** `typeVersion 1.2`
  - Requires external Grafana MCP server availability
- **Edge cases or potential failure types:**
  - MCP server unreachable
  - Datasource names in **SetVars** may not match real Grafana names
  - Grafana token/backend permissions may be insufficient
  - Agent may query irrelevant metrics if prompts are too broad
- **Sub-workflow reference:** None

#### HTTP Request
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`  
  AI-callable generic web request tool for supplemental checks.
- **Configuration choices:**
  - URL is provided dynamically by the AI with `$fromAI`
- **Key expressions or variables used:**
  - `={{ $fromAI('URL', '', 'string') }}`
- **Input and output connections:**
  - Connected to **AI Agent** through `ai_tool`
- **Version-specific requirements:** `typeVersion 4.4`
  - Requires support for AI tool parameter generation
- **Edge cases or potential failure types:**
  - Agent may attempt invalid or unreachable URLs
  - Network egress restrictions
  - DNS, TLS, timeout, or rate-limit failures
  - Security concerns if unrestricted outbound requests are allowed
- **Sub-workflow reference:** None

---

## 2.5 Notification Output

### Overview
This block posts the final AI-generated analysis to Slack. It is the workflow’s only user-facing output channel.

### Nodes Involved
- Send a message

### Node Details

#### Send a message
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the AI Agent result to a designated Slack channel.
- **Configuration choices:**
  - Operation: send message to a channel
  - Channel: `devops-ai-tests`
  - Message text: `={{ $json.output }}`
  - Authentication: OAuth2
- **Key expressions or variables used:**
  - `={{ $json.output }}`
- **Input and output connections:**
  - Input from **AI Agent**
  - No downstream output
- **Version-specific requirements:** `typeVersion 2.4`
- **Edge cases or potential failure types:**
  - Slack OAuth credential invalid or expired
  - Bot lacks permission to post to the selected channel
  - If AI Agent output is not under `output`, the message body may be empty
  - Slack formatting or length limits may affect very long reports
- **Sub-workflow reference:** None

---

## 2.6 Documentation and In-Canvas Notes

### Overview
These nodes do not affect execution but document the workflow structure, requirements, and MCP configuration expectations directly in the n8n canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the data collection area.
- **Configuration choices:** Content describes collecting job logs and pipeline file
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the output chain.
- **Configuration choices:** Notes that the generated report is posted to Slack
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual label for the AI analysis area.
- **Configuration choices:** Content is `# Error analysis`
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for MCP setup.
- **Configuration choices:** Instructs users to add correct URLs for remote MCP servers and references:
  - [gitlab-mcp](https://github.com/zereight/gitlab-mcp)
  - [grafana-mcp](https://github.com/grafana/mcp-grafana)
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  High-level workflow description, usage instructions, and requirement list.
- **Configuration choices:** Documents purpose, flow, setup, and external dependencies
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the input chain.
- **Configuration choices:** Notes that the workflow receives a pipeline event from GitLab
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get Job Events | n8n-nodes-base.webhook | Receives GitLab webhook events |  | Is job failed? | # Input Chain<br>Receving pipeline event from Gitlab |
| Is job failed? | n8n-nodes-base.if | Filters only failed events | Get Job Events | SetVars; No Operation, do nothing | # Input Chain<br>Receving pipeline event from Gitlab |
| No Operation, do nothing | n8n-nodes-base.noOp | Ends execution for non-failed events | Is job failed? |  | # Input Chain<br>Receving pipeline event from Gitlab |
| SetVars | n8n-nodes-base.set | Adds static Grafana/prompt variables | Is job failed? | Extract event data | # Collect data<br>* Job logs<br>* Pipeline file |
| Extract event data | n8n-nodes-base.code | Parses webhook payload and emits one item per failed job | SetVars | GetJobLogs; GetCIFile; Merge | # Collect data<br>* Job logs<br>* Pipeline file |
| GetJobLogs | n8n-nodes-base.httpRequest | Fetches failed job trace from GitLab API | Extract event data | GetLastLogLines | # Collect data<br>* Job logs<br>* Pipeline file |
| GetLastLogLines | n8n-nodes-base.code | Cleans trace and keeps the last 50 useful lines | GetJobLogs | Merge | # Collect data<br>* Job logs<br>* Pipeline file |
| GetCIFile | n8n-nodes-base.httpRequest | Fetches `.gitlab-ci.yml` for the failing commit | Extract event data | Merge | # Collect data<br>* Job logs<br>* Pipeline file |
| Merge | n8n-nodes-base.merge | Combines failed-job context, cleaned log, and CI config | GetLastLogLines; Extract event data; GetCIFile | FormatData | # Collect data<br>* Job logs<br>* Pipeline file |
| FormatData | n8n-nodes-base.code | Builds one AI-ready payload from merged inputs | Merge | AI Agent | # Error analysis |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the LLM for the AI Agent |  | AI Agent | # Error analysis |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Analyzes failure context and generates the final report | FormatData; OpenAI Chat Model; Gitlab MCP; MCP Grafana; HTTP Request | Send a message | # Error analysis |
| Gitlab MCP | @n8n/n8n-nodes-langchain.mcpClientTool | AI tool for GitLab repository/pipeline lookups |  | AI Agent | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana) |
| MCP Grafana | @n8n/n8n-nodes-langchain.mcpClientTool | AI tool for Grafana, Prometheus, and Loki diagnostics |  | AI Agent | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana) |
| HTTP Request | n8n-nodes-base.httpRequestTool | AI tool for generic external HTTP lookups |  | AI Agent | # MCP servers<br>Add correct URLs for remote MCPs<br>Use following mcp:<br>* [gitlab-mcp](https://github.com/zereight/gitlab-mcp)<br>* [grafana-mcp](https://github.com/grafana/mcp-grafana) |
| Send a message | n8n-nodes-base.slack | Posts the AI report to Slack | AI Agent |  | # Output Chain<br>Post generated report to Slack |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for data collection |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for output area |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for AI analysis area |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for MCP setup |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas overview, usage notes, and requirements |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation for input area |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Analyze GitLab CI job failures with GPT-5.3 and send reports to Slack`.

2. **Add the webhook trigger**
   - Create a **Webhook** node named **Get Job Events**.
   - Set:
     - Method: `POST`
     - Authentication: `Header Auth`
   - Create or select an **HTTP Header Auth** credential.
     - Configure it so GitLab can send a secret header such as `X-Gitlab-Token`.
   - Copy the production webhook URL later into GitLab webhook settings.

3. **Add a failure filter**
   - Create an **If** node named **Is job failed?**.
   - Connect **Get Job Events** → **Is job failed?**
   - Add one string condition:
     - Left value: `{{ $json.body.object_attributes.status }}`
     - Operator: `equals`
     - Right value: `failed`

4. **Add the non-failure exit branch**
   - Create a **No Operation** node named **No Operation, do nothing**.
   - Connect the **false** output of **Is job failed?** to this node.

5. **Add a Set node for static variables**
   - Create a **Set** node named **SetVars**.
   - Connect the **true** output of **Is job failed?** to **SetVars**.
   - Enable keeping other incoming fields.
   - Add these string fields:
     - `prometheus_name = dev`
     - `loki_name = grafanacloud-logs`
     - `grafana_group = TL`
   - Recommended improvement:
     - Also add `gitlab_group` because the AI prompt references it but the source workflow does not define it.

6. **Add payload extraction logic**
   - Create a **Code** node named **Extract event data**.
   - Set language to **Python (Native)**.
   - Connect **SetVars** → **Extract event data**.
   - Paste logic that:
     - Reads `body = _items[0]["json"]["body"]`
     - Extracts `object_attributes`, `project`, `merge_request`, `builds`
     - Filters builds to failed ones where `allow_failure` is false
     - Builds a `pipeline` object with:
       - `project_id`
       - `pipeline_id`
       - `pipeline_iid`
       - `commit_sha`
       - `gitlab_base_url`
       - `pipeline_duration_sec`
       - `pipeline_url`
       - `project_url`
       - `default_branch`
     - Builds a `failed_job` object with:
       - `job_id`
       - `job_name`
       - `job_stage`
       - `failure_reason`
       - `duration_sec`
       - `queued_duration_sec`
       - `started_at`
       - `finished_at`
       - `env_name`
       - `env_action`
       - `has_artifacts`
     - Returns one item per failed job:
       - `{ "pipeline": ..., "failed_job": ... }`
     - Returns `[]` if no actionable failed jobs exist.

7. **Add GitLab API credentials**
   - Create a **GitLab API** credential in n8n.
   - Use a token with permission to:
     - read project files
     - read pipelines/jobs
     - read job traces
   - Make sure it can access the target repositories/projects.

8. **Fetch the job logs**
   - Create an **HTTP Request** node named **GetJobLogs**.
   - Connect **Extract event data** → **GetJobLogs**.
   - Configure:
     - Method: `GET`
     - URL:
       `{{ $json.pipeline.gitlab_base_url }}/api/v4/projects/{{ $json.pipeline.project_id }}/jobs/{{ $json.failed_job.job_id }}/trace`
     - Authentication: `Predefined Credential Type`
     - Credential type: `GitLab API`
   - Keep default response handling unless your n8n version requires explicitly returning text.

9. **Trim and clean the logs**
   - Create a **Code** node named **GetLastLogLines**.
   - Set language to **Python (Native)**.
   - Connect **GetJobLogs** → **GetLastLogLines**.
   - Add logic that:
     - Reads the raw log from the HTTP response
     - Removes ANSI control sequences
     - Drops GitLab `section_start:` and `section_end:` markers
     - Removes empty lines
     - Keeps only the last 50 lines
     - Outputs:
       - `job_log`
   - Optional improvement:
     - Also output total line count for better prompting.

10. **Fetch the CI file**
    - Create another **HTTP Request** node named **GetCIFile**.
    - Connect **Extract event data** → **GetCIFile**.
    - Configure:
      - Method: `GET`
      - URL:
        `{{ $json.pipeline.gitlab_base_url }}/api/v4/projects/{{ $json.pipeline.project_id }}/repository/files/.gitlab-ci.yml/raw`
      - Query parameter:
        - `ref = {{ $json.pipeline.commit_sha }}`
      - Authentication: `Predefined Credential Type`
      - Credential type: `GitLab API`

11. **Add a Merge node**
    - Create a **Merge** node named **Merge**.
    - Set number of inputs to `3`.
    - Connect:
      - **GetLastLogLines** → Merge input 1
      - **Extract event data** → Merge input 2
      - **GetCIFile** → Merge input 3
   - Verify your n8n version’s merge behavior and item pairing. If multi-job runs behave incorrectly, consider using item matching keys or restructuring branches.

12. **Add formatting logic for the AI payload**
    - Create a **Code** node named **FormatData**.
    - Set language to **Python (Native)**.
    - Connect **Merge** → **FormatData**.
    - Add code that iterates through merged items and builds one object containing:
      - `job_log`
      - `pipeline`
      - `failed_job`
      - `ci_config`
      - optional `commit_diff`
    - Return a single-item array with that consolidated object.

13. **Add the language model**
    - Create an **OpenAI Chat Model** node named **OpenAI Chat Model**.
    - Configure OpenAI credentials.
    - Select model: `gpt-5.3-codex`
    - Leave model options at default unless you need deterministic behavior or token tuning.

14. **Add the AI Agent**
    - Create an **AI Agent** node named **AI Agent**.
    - Connect **FormatData** → **AI Agent**.
    - Connect **OpenAI Chat Model** to the agent’s language model port.
    - Set the input text to:
      - `{{ JSON.stringify($json, null, 2) }}`
    - Set prompt type to manually define the system message.
    - Add a system message that:
      - states the agent understands CI/CD systems
      - instructs it to analyze unsuccessful jobs only
      - asks for 1 to 3 likely causes and solutions
      - tells it not to make changes through MCP
      - says it should inspect `job_log` first
      - defines output sections:
        - `What happened`
        - `Root cause`
        - `Troubleshooting tips`
    - Include environment references such as:
      - `{{ $('SetVars').first().json.prometheus_name }}`
      - `{{ $('SetVars').first().json.loki_name }}`
    - If you also reference `gitlab_group`, make sure it exists in **SetVars**.

15. **Add the GitLab MCP tool**
    - Create an **MCP Client Tool** node named **Gitlab MCP**.
    - Connect it to the **AI Agent** tool port.
    - Set endpoint URL to your GitLab MCP server, for example:
      - `http://gitlab-mcp:8091/mcp`
    - Include only the tools you want exposed. In the source workflow these include read-oriented tools such as:
      - `search_repositories`
      - `get_file_contents`
      - `get_merge_request`
      - `get_merge_request_diffs`
      - `list_merge_request_versions`
      - `get_merge_request_version`
      - `get_branch_diffs`
      - `list_labels`
      - `get_label`
      - `list_projects`
      - `get_project`
      - `get_namespace`
      - `list_namespaces`
      - `get_commit`
      - `get_commit_diff`
      - `list_commits`
      - `get_repository_tree`
      - `list_merge_requests`
      - `list_group_projects`
      - `list_events`
      - `get_pipeline`
      - `list_pipelines`
      - `list_pipeline_jobs`
      - `list_pipeline_trigger_jobs`
      - `get_pipeline_job`
      - `get_pipeline_job_output`
    - Ensure the MCP backend itself is configured read-only where possible.

16. **Add the Grafana MCP tool**
    - Create another **MCP Client Tool** node named **MCP Grafana**.
    - Connect it to the **AI Agent** tool port.
    - Set endpoint URL to your Grafana MCP server, for example:
      - `http://mcp-grafana:8000/mcp`
    - Include tools such as:
      - `get_annotations`
      - `get_annotation_tags`
      - `get_dashboard_by_uid`
      - `get_dashboard_panel_queries`
      - `get_dashboard_property`
      - `get_dashboard_summary`
      - `get_datasource`
      - `get_panel_image`
      - `list_datasources`
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
      - `search_folders`
    - Configure Grafana-side credentials or service account access at the MCP server level.

17. **Add the generic HTTP tool**
    - Create an **HTTP Request Tool** node named **HTTP Request**.
    - Connect it to the **AI Agent** tool port.
    - Configure the URL parameter to be AI-provided using `$fromAI`.
    - Consider constraining this tool in production if unrestricted outbound web access is a concern.

18. **Add Slack output**
    - Create a **Slack** node named **Send a message**.
    - Connect **AI Agent** → **Send a message**.
    - Configure:
      - Resource/operation to send a message
      - Authentication: `OAuth2`
      - Select your channel, for example `devops-ai-tests`
      - Message text:
        `{{ $json.output }}`
    - Create or select a Slack OAuth2 credential with permission to post to the channel.

19. **Configure GitLab webhook on the GitLab side**
    - In your GitLab project or group:
      - Add the n8n webhook URL from **Get Job Events**
      - Set the secret token to match the header auth credential
      - Enable the relevant event type, typically pipeline-related events that include failed jobs/builds
    - Confirm the webhook payload contains:
      - `object_attributes.status`
      - `project`
      - `builds`

20. **Add canvas notes if desired**
    - Add **Sticky Note** nodes to document:
      - input chain
      - collection block
      - AI analysis block
      - MCP configuration block
      - output block
      - high-level requirements

21. **Test with a real failed pipeline**
    - Trigger a failing GitLab pipeline.
    - Verify:
      - the webhook reaches n8n
      - the IF node routes to the true branch
      - `Extract event data` emits items
      - job trace and CI file are successfully fetched
      - the AI agent returns structured text
      - Slack receives the report

22. **Recommended hardening before production**
    - Add explicit error handling around GitLab API failures
    - Add fallback behavior when `.gitlab-ci.yml` is missing or uses includes
    - Add length controls for CI config and logs
    - Add `gitlab_group` to **SetVars** or remove its prompt reference
    - Restrict AI tool access where necessary
    - Consider posting richer Slack blocks instead of plain text

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow helps automatically analyze the causes of build failures in Gitlab CI and propose solutions without involving DevOps engineers. | Canvas overview |
| Workflow logic described in the canvas: checks whether a job crashed, gets logs and pipeline description, lets the agent analyze data, optionally retrieves extra data from GitLab and checks endpoint availability, then sends a report to Slack. | Canvas overview |
| Usage notes from the canvas: generate webhook credentials and use them in GitLab; add your own rules and recommendations to the system prompt; run MCP servers; choose Slack channel. | Canvas overview |
| Requirements listed in the canvas: OpenAI or Anthropic API key, Slack API key, webhook with X-Gitlab-Token, GitLab access key, Grafana service account token. | Canvas overview |
| MCP setup note: add correct URLs for remote MCPs. | Canvas note |
| GitLab MCP reference | https://github.com/zereight/gitlab-mcp |
| Grafana MCP reference | https://github.com/grafana/mcp-grafana |