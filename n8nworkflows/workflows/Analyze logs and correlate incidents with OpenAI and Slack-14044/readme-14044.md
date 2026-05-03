Analyze logs and correlate incidents with OpenAI and Slack

https://n8nworkflows.xyz/workflows/analyze-logs-and-correlate-incidents-with-openai-and-slack-14044


# Analyze logs and correlate incidents with OpenAI and Slack

# 1. Workflow Overview

This workflow automates incident investigation after an external system sends an incident event to an n8n webhook. It collects operational context from multiple APIs, filters and structures recent logs, uses OpenAI-powered components to cluster failures and generate root-cause hypotheses, then posts the resulting incident summary to Slack.

Typical use cases:
- Automated triage for production incidents
- Rapid correlation of logs with recent deployments and feature changes
- AI-assisted root-cause suggestion for on-call engineers
- Centralized incident alerting into Slack

## 1.1 Incident Intake

The workflow starts from a webhook that receives an incident payload. This payload is used mainly for ticket metadata such as incident ID and severity.

## 1.2 Runtime Configuration

A Set node centralizes environment-specific values:
- Logs API endpoint
- Metrics API endpoint
- Deployments API endpoint
- Feature flags API endpoint
- Time window for analysis
- Slack channel ID

This block allows the workflow to be adapted without changing downstream expressions.

## 1.3 Incident Context Collection

Using the configured endpoints, the workflow fetches:
- recent logs
- metrics
- recent deployments
- feature flags

These parallel data sources provide the context used later for correlation and AI reasoning.

## 1.4 Log Normalization and Denoising

Raw log payloads are filtered to remove low-value entries such as debug/info/trace/verbose logs. Remaining warning/error/fatal logs are normalized to a common structure and grouped by session or request identifier.

## 1.5 Context Aggregation and Failure Clustering

The normalized logs and the other collected context are aggregated. Error documents are then loaded, embedded with OpenAI embeddings, and inserted into an in-memory vector store. A subsequent code node attempts to identify dominant failure patterns from the clustered or vectorized error data.

## 1.6 Event Correlation

The workflow then attempts to time-align failure patterns with deployments, config changes, and traffic spikes to estimate likely causal relationships.

## 1.7 AI Root Cause Analysis

A LangChain agent uses an OpenAI chat model plus a structured output parser to generate ranked root-cause hypotheses, evidence, and recommended actions.

## 1.8 Slack Incident Ticketing

Finally, the workflow formats the AI analysis into a Slack message and posts it to the configured channel.

---

# 2. Block-by-Block Analysis

## 2.1 Incident Intake

**Overview:**  
This block exposes the workflow as an HTTP endpoint. It is the entry point for incident events coming from an external monitoring or alerting system.

**Nodes Involved:**  
- Incident Trigger

### Node Details

#### Incident Trigger
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Receives an incoming HTTP POST request and starts the workflow.
- **Configuration choices:**  
  - Path: `incident-trigger`
  - Method: `POST`
  - Response mode: `lastNode`  
  This means the webhook waits until the workflow completes and returns the output from the final node.
- **Key expressions or variables used:**  
  Downstream nodes reference:
  - `$('Incident Trigger').first().json.body.incidentId`
  - `$('Incident Trigger').first().json.body.severity`
- **Input and output connections:**  
  - Input: none
  - Output: Workflow Configuration
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - If the caller sends no `body`, downstream expressions may resolve to undefined.
  - `responseMode = lastNode` can lead to long-running webhook requests if APIs or AI steps are slow.
  - Timeout risks depend on deployment and reverse proxy settings.
- **Sub-workflow reference:** none

---

## 2.2 Runtime Configuration

**Overview:**  
This block defines all configurable endpoints and operational parameters in one place. It serves as a lightweight configuration layer for the rest of the workflow.

**Nodes Involved:**  
- Workflow Configuration

### Node Details

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates reusable static fields while preserving incoming webhook data.
- **Configuration choices:**  
  `includeOtherFields` is enabled, so the incoming incident payload remains available.
  Assigned fields:
  - `logsApiUrl`
  - `metricsApiUrl`
  - `deploymentsApiUrl`
  - `featureFlagsApiUrl`
  - `timeWindowMinutes = 30`
  - `slackChannel`
- **Key expressions or variables used:**  
  Downstream nodes use:
  - `$('Workflow Configuration').first().json.logsApiUrl`
  - `$('Workflow Configuration').first().json.metricsApiUrl`
  - `$('Workflow Configuration').first().json.deploymentsApiUrl`
  - `$('Workflow Configuration').first().json.featureFlagsApiUrl`
  - `$('Workflow Configuration').first().json.timeWindowMinutes`
  - `$('Workflow Configuration').first().json.slackChannel`
- **Input and output connections:**  
  - Input: Incident Trigger
  - Outputs:
    - Fetch Logs
    - Fetch Metrics
    - Fetch Recent Deployments
    - Fetch Feature Flags
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Placeholder values must be replaced before production use.
  - Invalid URLs will cause downstream HTTP Request failures.
  - If `slackChannel` is invalid, Slack posting fails later.
- **Sub-workflow reference:** none

---

## 2.3 Incident Context Collection

**Overview:**  
This block retrieves operational signals from external systems. The four requests run in parallel from the configuration node and form the basis for both clustering and causal analysis.

**Nodes Involved:**  
- Fetch Logs
- Fetch Metrics
- Fetch Recent Deployments
- Fetch Feature Flags

### Node Details

#### Fetch Logs
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the logs API for the configured time window.
- **Configuration choices:**  
  - URL from `logsApiUrl`
  - Query parameter `minutes` from `timeWindowMinutes`
  - Response format forced to JSON
- **Key expressions or variables used:**  
  - URL: `={{ $('Workflow Configuration').first().json.logsApiUrl }}`
  - Query `minutes`: `={{ $('Workflow Configuration').first().json.timeWindowMinutes }}`
- **Input and output connections:**  
  - Input: Workflow Configuration
  - Output: Normalize and Denoise Logs
- **Version-specific requirements:**  
  Type version `4.3`
- **Edge cases or potential failure types:**  
  - Non-JSON response will fail parsing.
  - Authentication is not configured in the JSON; if the API requires auth, credentials or headers must be added.
  - If the API returns an array nested under a property instead of as itemized records, the code node may not interpret it as intended.
- **Sub-workflow reference:** none

#### Fetch Metrics
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves metrics data for the same analysis window.
- **Configuration choices:**  
  - URL from `metricsApiUrl`
  - Query parameter `minutes`
  - Response format JSON
- **Key expressions or variables used:**  
  Same pattern as above using `metricsApiUrl`.
- **Input and output connections:**  
  - Input: Workflow Configuration
  - Output: Merge Context Data
- **Version-specific requirements:**  
  Type version `4.3`
- **Edge cases or potential failure types:**  
  - Missing auth/header config
  - Unexpected schema may reduce usefulness later because downstream code expects typed events such as `traffic_spike`
- **Sub-workflow reference:** none

#### Fetch Recent Deployments
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves recent deployments in the configured time window.
- **Configuration choices:**  
  - URL from `deploymentsApiUrl`
  - Query parameter `minutes`
  - Response format JSON
- **Key expressions or variables used:**  
  `deploymentsApiUrl`, `timeWindowMinutes`
- **Input and output connections:**  
  - Input: Workflow Configuration
  - Output: Merge Context Data
- **Version-specific requirements:**  
  Type version `4.3`
- **Edge cases or potential failure types:**  
  - Missing auth/header config
  - Downstream correlation expects events shaped like `type: deployment` plus timestamps
- **Sub-workflow reference:** none

#### Fetch Feature Flags
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves feature-flag state or changes.
- **Configuration choices:**  
  - URL from `featureFlagsApiUrl`
  - Response format JSON
  - No `minutes` query parameter is configured
- **Key expressions or variables used:**  
  `={{ $('Workflow Configuration').first().json.featureFlagsApiUrl }}`
- **Input and output connections:**  
  - Input: Workflow Configuration
  - Output: Merge Context Data
- **Version-specific requirements:**  
  Type version `4.3`
- **Edge cases or potential failure types:**  
  - Missing auth/header config
  - If the API returns current flags rather than change events, event correlation logic will not directly use them
- **Sub-workflow reference:** none

---

## 2.4 Log Normalization and Denoising

**Overview:**  
This block converts inconsistent raw logs into a structured schema and removes low-signal entries. It groups warning/error records by session/request ID so downstream analysis can focus on failure sequences rather than isolated lines.

**Nodes Involved:**  
- Normalize and Denoise Logs

### Node Details

#### Normalize and Denoise Logs
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript transformation for filtering, normalization, grouping, and summarization of logs.
- **Configuration choices:**  
  The code:
  - Reads all incoming items with `$input.all()`
  - Excludes log levels matching `/debug/i`, `/info/i`, `/trace/i`, `/verbose/i`
  - Normalizes fields across common variants:
    - timestamp/time/@timestamp
    - level/severity
    - message/msg/text
    - sessionId/session_id/requestId/request_id/traceId
    - service/serviceName/app
    - error/exception/stack
  - Keeps only levels:
    - `error`
    - `warn`
    - `warning`
    - `fatal`
    - `critical`
  - Groups logs by `sessionId`
  - Sorts grouped logs by timestamp
  - Emits one item per session with:
    - `sessionId`
    - `logCount`
    - `firstError`
    - `lastError`
    - `service`
    - `errors[]`
  - If nothing remains, returns one item with message `No error logs found after denoising`
- **Key expressions or variables used:**  
  Internal code uses `$input.all()`
- **Input and output connections:**  
  - Input: Fetch Logs
  - Output: Merge Context Data
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - If the logs API returns a single item containing an array rather than multiple items, the code may treat the array container as one log object unless preprocessed.
  - Missing timestamps are replaced with current time, which can distort chronological analysis.
  - Missing session IDs fall back to `'unknown'`, potentially grouping unrelated failures together.
  - If all logs are filtered out, downstream clustering receives a non-error placeholder message instead of actual error objects.
- **Sub-workflow reference:** none

---

## 2.5 Context Aggregation and Failure Clustering

**Overview:**  
This block aggregates all context into a single stream, loads error data as documents, embeds them with OpenAI, and inserts them into an in-memory vector store. A custom code step then attempts to infer dominant error patterns.

**Nodes Involved:**  
- Merge Context Data
- OpenAI Embeddings
- Document Loader
- Cluster Log Messages
- Identify Failure Patterns

### Node Details

#### Merge Context Data
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Aggregates all incoming item data into a combined output.
- **Configuration choices:**  
  `aggregate = aggregateAllItemData`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Inputs:
    - Fetch Metrics
    - Fetch Recent Deployments
    - Fetch Feature Flags
    - Normalize and Denoise Logs
  - Output: Cluster Log Messages
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - The exact aggregated structure depends on the item shapes from all branches.
  - Later nodes assume direct access to `$json.errors`, but aggregation may wrap source data in arrays or merged objects not matching that expectation.
  - Parallel branch completion timing is managed by n8n, but malformed incoming items can still make the merged payload unsuitable.
- **Sub-workflow reference:** none

#### OpenAI Embeddings
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  Generates embeddings for document content before vector-store insertion.
- **Configuration choices:**  
  Uses default options; no model override is specified in the JSON.
- **Key expressions or variables used:** none directly
- **Input and output connections:**  
  - AI output: Cluster Log Messages
- **Version-specific requirements:**  
  Type version `1.2`
  Requires valid OpenAI credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - Missing or invalid OpenAI credentials
  - Token or rate-limit errors
  - Input text too large depending on embedded document sizes
- **Sub-workflow reference:** none

#### Document Loader
- **Type and technical role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
  Converts JSON error arrays into LangChain documents.
- **Configuration choices:**  
  - JSON mode: `expressionData`
  - JSON data: `={{ $json.errors }}`
- **Key expressions or variables used:**  
  `$json.errors`
- **Input and output connections:**  
  - AI document output: Cluster Log Messages
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - There is no standard main input connection to this node in the workflow JSON. As configured, it is disconnected from the execution path.
  - Because it is disconnected, Cluster Log Messages may never receive documents.
  - If it were connected, `$json.errors` must exist and be an array or JSON-like structure suitable for document creation.
- **Sub-workflow reference:** none

#### Cluster Log Messages
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStoreInMemory`  
  Inserts document embeddings into an in-memory vector store.
- **Configuration choices:**  
  - Mode: `insert`
  - Memory key: `vector_store_key`
- **Key expressions or variables used:**  
  Memory key selection only
- **Input and output connections:**  
  - Main input: Merge Context Data
  - AI embedding input: OpenAI Embeddings
  - AI document input: Document Loader
  - Main output: Identify Failure Patterns
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - In-memory vector stores are ephemeral and reset between executions.
  - If no documents arrive from Document Loader, insert mode may fail or produce no meaningful output.
  - The node is being used only for insertion, not explicit similarity search; the downstream code expects clustered items but the vector-store output may not contain `cluster_id`.
- **Sub-workflow reference:** none

#### Identify Failure Patterns
- **Type and technical role:** `n8n-nodes-base.code`  
  Summarizes error distributions and pseudo-clusters from upstream records.
- **Configuration choices:**  
  The code:
  - Reads all items
  - Extracts:
    - `error_type` or `level` or `severity`
    - `message` or `log` or `text`
    - `timestamp` or `time`
  - Builds counts by error type
  - Groups by `cluster_id` or `group`, defaulting to `'default'`
  - Produces:
    - summary stats
    - top error types
    - cluster analysis
    - failure pattern stats
    - raw_data
- **Key expressions or variables used:**  
  `$input.all()`
- **Input and output connections:**  
  - Input: Cluster Log Messages
  - Output: Time-Align Events
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - If vector-store output lacks fields like `message`, `timestamp`, `cluster_id`, the analysis becomes weak or generic.
  - If upstream only emits one aggregate item, the statistics may not represent individual errors.
  - `data.timestamps.sort()` sorts lexicographically; ISO timestamps are usually safe, but non-ISO formats can misorder.
- **Sub-workflow reference:** none

---

## 2.6 Event Correlation

**Overview:**  
This block tries to correlate failures with likely causes based on timestamps. It is intended to connect failures to deployments, config changes, and traffic spikes using weighted causal scores.

**Nodes Involved:**  
- Time-Align Events

### Node Details

#### Time-Align Events
- **Type and technical role:** `n8n-nodes-base.code`  
  Performs temporal correlation and causal ranking.
- **Configuration choices:**  
  The code:
  - Splits incoming items into:
    - `failurePatterns`
    - `deployments`
    - `configChanges`
    - `trafficSpikes`
  - Uses `data.type` to determine category
  - Looks for events occurring shortly before failures
  - Time windows:
    - deployments: 60 min
    - config changes: 30 min
    - traffic spikes: 15 min
  - Computes `correlationStrength`
  - Produces:
    - `correlatedEvents`
    - `causalRelationships`
    - `topLikelyCause`
    - `timestamp`
- **Key expressions or variables used:**  
  `$input.all()`
- **Input and output connections:**  
  - Input: Identify Failure Patterns
  - Output: Root Cause Analysis Agent
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - The incoming data from `Identify Failure Patterns` is a single summary object, not obviously a list of typed events with `type` fields.
  - If no items have `type === 'failure_pattern'`, the node returns an empty array.
  - If this node returns no items, the agent will not execute.
  - Feature flags are not directly handled unless transformed into `config_change`.
- **Sub-workflow reference:** none

---

## 2.7 AI Root Cause Analysis

**Overview:**  
This block uses a chat model and a structured parser to generate machine-readable incident conclusions. It is the workflow’s decision-making layer, intended to produce ranked hypotheses with confidence and recommended actions.

**Nodes Involved:**  
- OpenAI Chat Model
- Root Cause Output Parser
- Root Cause Analysis Agent

### Node Details

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM used by the agent.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
  - No additional options configured
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - AI language model output: Root Cause Analysis Agent
- **Version-specific requirements:**  
  Type version `1.3`
  Requires OpenAI credentials.
- **Edge cases or potential failure types:**  
  - Credential/authentication issues
  - Rate limits
  - Output variability if schema enforcement is imperfect
- **Sub-workflow reference:** none

#### Root Cause Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Constrains model output to a JSON schema.
- **Configuration choices:**  
  Manual JSON schema:
  - top-level object
  - `rootCauses` array
  - each item includes:
    - `hypothesis` string
    - `confidence` number 0–100
    - `evidence` string array
    - `recommendedAction` string
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - AI output parser output: Root Cause Analysis Agent
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - If the model returns content outside schema, parsing can fail.
  - The downstream Slack node does not reference `rootCauses`; it expects different fields such as `hypotheses`, `recommendations`, `summary`, `affectedServices`.
- **Sub-workflow reference:** none

#### Root Cause Analysis Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Receives incident context and prompts the LLM to generate structured root-cause reasoning.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Output parser enabled
  - User prompt interpolates:
    - `{{ $json.failurePatterns }}`
    - `{{ $json.correlatedEvents }}`
    - `{{ $json.metrics }}`
    - `{{ $json.deployments }}`
    - `{{ $json.featureFlags }}`
  - System message frames the AI as an expert SRE and asks for:
    - probable root causes
    - confidence scores
    - evidence
    - concrete actions
- **Key expressions or variables used:**  
  The prompt references fields that must exist on the incoming item.
- **Input and output connections:**  
  - Main input: Time-Align Events
  - AI model input: OpenAI Chat Model
  - AI parser input: Root Cause Output Parser
  - Main output: Create Incident Ticket
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - If Time-Align Events emits empty results, the agent will not run.
  - The prompt references fields not produced by Time-Align Events in the current design.
  - Parsed output schema does not match what the Slack node later expects.
- **Sub-workflow reference:** none

---

## 2.8 Slack Incident Ticketing

**Overview:**  
This block posts the final incident summary to Slack. It includes incident metadata from the webhook and analysis content from the AI agent.

**Nodes Involved:**  
- Create Incident Ticket

### Node Details

#### Create Incident Ticket
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a message to a Slack channel.
- **Configuration choices:**  
  - Resource/action corresponds to sending a message
  - Channel selected by ID
  - Channel ID comes from workflow configuration
  - Includes link to the workflow
  - Message text contains sections:
    - incident metadata
    - root-cause hypotheses
    - recommended actions
    - analysis summary
    - affected services
    - correlation score
- **Key expressions or variables used:**  
  - Channel:
    `={{ $('Workflow Configuration').first().json.slackChannel }}`
  - Incident data:
    `$('Incident Trigger').first().json.body.incidentId`
    `$('Incident Trigger').first().json.body.severity`
  - AI data:
    `$('Root Cause Analysis Agent').first().json.hypotheses`
    `recommendations`
    `summary`
    `affectedServices`
    `correlationScore`
- **Input and output connections:**  
  - Input: Root Cause Analysis Agent
  - Output: none
- **Version-specific requirements:**  
  Type version `2.4`
  Requires Slack credentials.
- **Edge cases or potential failure types:**  
  - The agent’s structured schema currently outputs `rootCauses`, not `hypotheses`, so most AI sections may render fallback text.
  - Invalid channel ID or missing Slack scopes will fail posting.
  - Long Slack messages can be truncated depending on workspace/API limits.
- **Sub-workflow reference:** none

---

## 2.9 Sticky Notes / Visual Documentation

**Overview:**  
These nodes are documentation-only and do not execute. They explain workflow intent and visually label major sections.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the incident trigger area.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note1
- Same technical type and non-executing role. Documents workflow configuration.

#### Sticky Note2
- Same technical type and non-executing role. Documents context collection.

#### Sticky Note3
- Same technical type and non-executing role. Documents log normalization.

#### Sticky Note4
- Same technical type and non-executing role. Documents failure clustering.

#### Sticky Note5
- Same technical type and non-executing role. Documents event correlation.

#### Sticky Note6
- Same technical type and non-executing role. Documents Slack posting.

#### Sticky Note7
- Same technical type and non-executing role. Documents AI root-cause analysis and ticketing.

#### Sticky Note8
- Same technical type and non-executing role. Provides overall workflow summary.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Incident Trigger | n8n-nodes-base.webhook | Receives incident webhook and starts execution |  | Workflow Configuration | ## Incident Trigger\n\nThis workflow starts when an incident alert is received through a webhook. |
| Workflow Configuration | n8n-nodes-base.set | Stores API URLs, analysis window, and Slack channel | Incident Trigger | Fetch Logs; Fetch Metrics; Fetch Recent Deployments; Fetch Feature Flags | ## Workflow Configuration\nDefines the API endpoints used to collect logs, metrics, deployments, and feature flag data. It also configures the analysis time window and Slack channel used for incident notifications. |
| Fetch Logs | n8n-nodes-base.httpRequest | Retrieves recent logs | Workflow Configuration | Normalize and Denoise Logs | ## Incident Context Collection\n\nThe workflow gathers operational context from multiple sources including logs, metrics, recent deployments, and feature flag changes. |
| Fetch Metrics | n8n-nodes-base.httpRequest | Retrieves metrics for analysis | Workflow Configuration | Merge Context Data | ## Incident Context Collection\n\nThe workflow gathers operational context from multiple sources including logs, metrics, recent deployments, and feature flag changes. |
| Fetch Recent Deployments | n8n-nodes-base.httpRequest | Retrieves deployment history | Workflow Configuration | Merge Context Data | ## Incident Context Collection\n\nThe workflow gathers operational context from multiple sources including logs, metrics, recent deployments, and feature flag changes. |
| Fetch Feature Flags | n8n-nodes-base.httpRequest | Retrieves feature flag data | Workflow Configuration | Merge Context Data | ## Incident Context Collection\n\nThe workflow gathers operational context from multiple sources including logs, metrics, recent deployments, and feature flag changes. |
| Normalize and Denoise Logs | n8n-nodes-base.code | Filters and normalizes logs, groups errors by session | Fetch Logs | Merge Context Data | ## Log Normalization and Denoising\n\nRaw logs are cleaned and normalized to remove low-value entries such as debug or informational messages. |
| Merge Context Data | n8n-nodes-base.aggregate | Aggregates logs and context into a combined dataset | Fetch Metrics; Fetch Recent Deployments; Fetch Feature Flags; Normalize and Denoise Logs | Cluster Log Messages |  |
| OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates embeddings for log/error documents |  | Cluster Log Messages | ## Failure Pattern Clustering\n\nError messages are converted into embeddings and grouped using a vector store. |
| Document Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Converts JSON errors to LangChain documents |  | Cluster Log Messages | ## Failure Pattern Clustering\n\nError messages are converted into embeddings and grouped using a vector store. |
| Cluster Log Messages | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Inserts embedded documents into in-memory vector store | Merge Context Data; Document Loader; OpenAI Embeddings | Identify Failure Patterns | ## Failure Pattern Clustering\n\nError messages are converted into embeddings and grouped using a vector store. |
| Identify Failure Patterns | n8n-nodes-base.code | Summarizes error frequencies and pseudo-clusters | Cluster Log Messages | Time-Align Events | ## Event Correlation Analysis\n\nFailure patterns are time-aligned with contextual events such as deployments, configuration changes, and traffic spikes. |
| Time-Align Events | n8n-nodes-base.code | Correlates failures with deployments/config/traffic by time | Identify Failure Patterns | Root Cause Analysis Agent | ## Event Correlation Analysis\n\nFailure patterns are time-aligned with contextual events such as deployments, configuration changes, and traffic spikes. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies chat LLM for root-cause reasoning |  | Root Cause Analysis Agent | ## Root Cause Analysis and Incident Ticket\n\nAn AI agent analyzes all collected signals and generates root cause hypotheses. |
| Root Cause Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured AI output schema |  | Root Cause Analysis Agent | ## Root Cause Analysis and Incident Ticket\n\nAn AI agent analyzes all collected signals and generates root cause hypotheses. |
| Root Cause Analysis Agent | @n8n/n8n-nodes-langchain.agent | Produces root-cause hypotheses and actions | Time-Align Events; OpenAI Chat Model; Root Cause Output Parser | Create Incident Ticket | ## Root Cause Analysis and Incident Ticket\n\nAn AI agent analyzes all collected signals and generates root cause hypotheses. |
| Create Incident Ticket | n8n-nodes-base.slack | Posts incident analysis to Slack | Root Cause Analysis Agent |  | The final analysis is posted to Slack as an incident ticket for engineers to review. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Incident Trigger\n\nThis workflow starts when an incident alert is received through a webhook. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Workflow Configuration\nDefines the API endpoints used to collect logs, metrics, deployments, and feature flag data. It also configures the analysis time window and Slack channel used for incident notifications. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Incident Context Collection\n\nThe workflow gathers operational context from multiple sources including logs, metrics, recent deployments, and feature flag changes. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Log Normalization and Denoising\n\nRaw logs are cleaned and normalized to remove low-value entries such as debug or informational messages. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Failure Pattern Clustering\n\nError messages are converted into embeddings and grouped using a vector store. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Event Correlation Analysis\n\nFailure patterns are time-aligned with contextual events such as deployments, configuration changes, and traffic spikes. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation |  |  | The final analysis is posted to Slack as an incident ticket for engineers to review. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Root Cause Analysis and Incident Ticket\n\nAn AI agent analyzes all collected signals and generates root cause hypotheses. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## AI-Powered Incident Root Cause Analysis\n\nThis workflow automates incident investigation using logs, metrics, deployment data, and configuration changes.\n\nIt clusters error logs, identifies failure patterns, correlates them with system events, and uses an AI agent to generate root cause hypotheses and recommended actions. The results are automatically posted to Slack for rapid incident response. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **Analyze logs and correlate incidents with OpenAI and Slack**

2. **Add a Webhook node** named **Incident Trigger**.
   - Type: Webhook
   - HTTP Method: `POST`
   - Path: `incident-trigger`
   - Response Mode: `Last Node`
   - Expect the incoming payload to include fields such as:
     - `body.incidentId`
     - `body.severity`

3. **Add a Set node** named **Workflow Configuration** and connect it from **Incident Trigger**.
   - Enable keeping other incoming fields.
   - Add these fields:
     1. `logsApiUrl` as String
     2. `metricsApiUrl` as String
     3. `deploymentsApiUrl` as String
     4. `featureFlagsApiUrl` as String
     5. `timeWindowMinutes` as Number, default `30`
     6. `slackChannel` as String
   - Replace all placeholder values with real endpoints and channel ID.

4. **Add an HTTP Request node** named **Fetch Logs**.
   - Connect from **Workflow Configuration**
   - Method: `GET` unless your API requires otherwise
   - URL expression:  
     `{{ $('Workflow Configuration').first().json.logsApiUrl }}`
   - Enable query parameters:
     - `minutes` = `{{ $('Workflow Configuration').first().json.timeWindowMinutes }}`
   - Response format: JSON
   - If required, configure:
     - authentication
     - headers
     - bearer token
     - retry behavior

5. **Add an HTTP Request node** named **Fetch Metrics**.
   - Connect from **Workflow Configuration**
   - URL expression:  
     `{{ $('Workflow Configuration').first().json.metricsApiUrl }}`
   - Query parameter:
     - `minutes` = `{{ $('Workflow Configuration').first().json.timeWindowMinutes }}`
   - Response format: JSON
   - Add any required auth.

6. **Add an HTTP Request node** named **Fetch Recent Deployments**.
   - Connect from **Workflow Configuration**
   - URL expression:  
     `{{ $('Workflow Configuration').first().json.deploymentsApiUrl }}`
   - Query parameter:
     - `minutes` = `{{ $('Workflow Configuration').first().json.timeWindowMinutes }}`
   - Response format: JSON
   - Add any required auth.

7. **Add an HTTP Request node** named **Fetch Feature Flags**.
   - Connect from **Workflow Configuration**
   - URL expression:  
     `{{ $('Workflow Configuration').first().json.featureFlagsApiUrl }}`
   - Response format: JSON
   - Add any required auth.

8. **Add a Code node** named **Normalize and Denoise Logs** and connect it from **Fetch Logs**.
   - Paste the JavaScript logic that:
     - reads all log items
     - removes debug/info/trace/verbose levels
     - normalizes field names
     - keeps warning/error/fatal/critical entries
     - groups by session/request/trace ID
     - outputs one item per session with `errors[]`
   - Important expected output fields:
     - `sessionId`
     - `logCount`
     - `firstError`
     - `lastError`
     - `service`
     - `errors`

9. **Add an Aggregate node** named **Merge Context Data**.
   - Connect these nodes into it:
     - **Fetch Metrics**
     - **Fetch Recent Deployments**
     - **Fetch Feature Flags**
     - **Normalize and Denoise Logs**
   - Set aggregation mode to aggregate all item data.
   - Validate the resulting JSON structure manually after a test run, because later nodes assume certain fields exist directly.

10. **Create OpenAI credentials** in n8n.
    - Use an OpenAI API key with access to:
      - embeddings
      - chat completions
    - Assign these credentials later to:
      - **OpenAI Embeddings**
      - **OpenAI Chat Model**

11. **Add a Document Default Data Loader node** named **Document Loader**.
    - Configure JSON mode as expression-based
    - Set JSON data to:
      `{{ $json.errors }}`
    - Important: unlike the provided JSON, you should also connect this node in the main execution path so it receives data.  
      Practical approach:
      - Connect **Merge Context Data** to **Document Loader**
    - Expected input:
      - a field `errors` containing an array of error objects

12. **Add an OpenAI Embeddings node** named **OpenAI Embeddings**.
    - Attach OpenAI credentials
    - Keep default options unless you need a specific embedding model

13. **Add an In-Memory Vector Store node** named **Cluster Log Messages**.
    - Mode: `Insert`
    - Memory key: `vector_store_key`
    - Connect:
      - Main input from **Merge Context Data**
      - AI document input from **Document Loader**
      - AI embedding input from **OpenAI Embeddings**
   - Note: this setup inserts vectors but does not itself perform clustering analysis in a strong analytical sense. If you want true semantic grouping, add retrieval/search or explicit clustering logic after insertion.

14. **Add a Code node** named **Identify Failure Patterns** and connect it from **Cluster Log Messages**.
    - Paste the JavaScript that:
      - counts error types
      - groups items by `cluster_id` or `group`
      - computes summary statistics
      - emits:
        - `summary`
        - `top_error_types`
        - `cluster_analysis`
        - `failure_patterns`
        - `raw_data`
    - Verify the upstream vector-store output actually contains fields this code expects.

15. **Add a Code node** named **Time-Align Events** and connect it from **Identify Failure Patterns**.
    - Paste the JavaScript that:
      - separates events by `data.type`
      - correlates failures to deployments/config changes/traffic spikes by timestamp
      - computes causal scores
    - Important implementation note: in the provided design, this node likely needs richer inputs than it currently receives.  
      To make it work reliably, adjust earlier aggregation so it outputs typed events such as:
      - `type: failure_pattern`
      - `type: deployment`
      - `type: config_change`
      - `type: traffic_spike`
      each with a valid `timestamp`.

16. **Add an OpenAI Chat Model node** named **OpenAI Chat Model**.
    - Select model: `gpt-4.1-mini`
    - Attach OpenAI credentials

17. **Add a Structured Output Parser node** named **Root Cause Output Parser**.
    - Choose manual schema
    - Define a JSON schema with:
      - `rootCauses` as array
      - each object containing:
        - `hypothesis` string
        - `confidence` number 0–100
        - `evidence` array of strings
        - `recommendedAction` string

18. **Add an AI Agent node** named **Root Cause Analysis Agent**.
    - Set prompt type to manually defined
    - Enable output parser
    - Connect:
      - Main input from **Time-Align Events**
      - AI language model from **OpenAI Chat Model**
      - AI output parser from **Root Cause Output Parser**
    - Use a system message instructing the model to act as an expert SRE and generate ranked root-cause hypotheses with evidence and actions.
    - Use a user prompt referencing the incoming data, for example:
      - failure patterns
      - correlated events
      - metrics
      - deployments
      - feature flags
    - Recommended improvement: align prompt variable names with the actual structure emitted by **Time-Align Events**.

19. **Create Slack credentials** in n8n.
    - Use a Slack app or OAuth2 setup with permission to post messages to the target channel.
    - Ensure the bot is invited to that channel.

20. **Add a Slack node** named **Create Incident Ticket**.
    - Connect from **Root Cause Analysis Agent**
    - Operation: send message to channel
    - Channel selection mode: by ID
    - Channel ID expression:  
      `{{ $('Workflow Configuration').first().json.slackChannel }}`
    - Compose the message using expressions for:
      - incident ID
      - severity
      - generated hypotheses
      - recommended actions
      - summary
      - affected services
      - correlation score

21. **Align the Slack message schema with the AI output.**
    - In the provided workflow, Slack expects:
      - `hypotheses`
      - `recommendations`
      - `summary`
      - `affectedServices`
      - `correlationScore`
    - But the parser enforces:
      - `rootCauses`
    - To reproduce successfully, either:
      - change the parser schema to match the Slack fields, or
      - change the Slack expressions to read from `rootCauses`

22. **Add sticky notes** if you want the same visual layout.
    - Create notes for:
      - Incident Trigger
      - Workflow Configuration
      - Incident Context Collection
      - Log Normalization and Denoising
      - Failure Pattern Clustering
      - Event Correlation Analysis
      - Root Cause Analysis and Incident Ticket
      - Slack posting
      - Overall workflow description

23. **Test the workflow with a sample POST request** to the webhook.
    - Example payload should include:
      - `incidentId`
      - `severity`
    - Confirm each API responds with valid JSON.

24. **Validate data shape after each major block.**
    - Especially inspect:
      - output of **Normalize and Denoise Logs**
      - output of **Merge Context Data**
      - whether **Document Loader** receives `errors`
      - whether **Time-Align Events** receives typed timestamped events
      - whether **Root Cause Analysis Agent** produces fields used by Slack

25. **Harden the workflow for production.**
    - Add HTTP auth and headers where needed
    - Add retry/error handling to API nodes
    - Consider `Continue On Fail` only when partial analysis is acceptable
    - Consider replacing `responseMode = lastNode` with an earlier acknowledgment if execution may be long
    - Consider persistent storage instead of in-memory vector store if you need reuse across runs

### Credential Configuration Summary

- **OpenAI**
  - Required for:
    - OpenAI Embeddings
    - OpenAI Chat Model
  - Needs valid API key and access to selected models

- **Slack**
  - Required for:
    - Create Incident Ticket
  - Needs permission to post to the target channel

- **External APIs**
  - Not configured in the JSON, but likely needed for:
    - Fetch Logs
    - Fetch Metrics
    - Fetch Recent Deployments
    - Fetch Feature Flags
  - Add headers, bearer auth, OAuth2, or API key auth based on your systems

### Sub-workflow Setup

This workflow does **not** use any Execute Workflow / sub-workflow nodes.  
There are **no sub-workflows** to create.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## AI-Powered Incident Root Cause Analysis This workflow automates incident investigation using logs, metrics, deployment data, and configuration changes. It clusters error logs, identifies failure patterns, correlates them with system events, and uses an AI agent to generate root cause hypotheses and recommended actions. The results are automatically posted to Slack for rapid incident response. | Overall workflow purpose |
| The workflow contains several design mismatches that should be corrected before production use. | Document Loader is disconnected from the main path; Time-Align Events expects typed event items that are not clearly produced upstream; Slack output fields do not match the structured parser schema. |
| The in-memory vector store is ephemeral. | Data is available only for the current execution and is not suitable for long-term incident knowledge retention. |
| Webhook response mode is set to return the last node output. | This can increase requester wait time and may cause timeouts for long AI or API operations. |