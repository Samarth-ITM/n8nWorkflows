Turn support tickets into developer insights with OpenAI, Postgres, Slack and Jira

https://n8nworkflows.xyz/workflows/turn-support-tickets-into-developer-insights-with-openai--postgres--slack-and-jira-14684


# Turn support tickets into developer insights with OpenAI, Postgres, Slack and Jira

# 1. Workflow Overview

This workflow runs on a schedule to transform recent support tickets into a developer-focused intelligence report. It pulls bug and feature request tickets from Postgres, cleans and deduplicates them, groups semantically similar tickets using OpenAI embeddings, performs AI-based root cause analysis on each group, scores the groups by business/technical severity, generates a Markdown report, and delivers the result through Slack, email, and optionally Jira.

Typical use cases:

- Daily or periodic review of recurring support issues
- Converting raw support feedback into engineering priorities
- Detecting product pain points with severity weighting
- Producing a structured report for development or product teams

## 1.1 Trigger & Configuration

The workflow starts on a scheduled cadence and sets global configuration values such as analysis window, similarity threshold, severity weights, and delivery toggles.

## 1.2 Data Retrieval

Recent support tickets are queried from Postgres, limited to `bug` and `feature_request` categories within the configured time window.

## 1.3 Data Cleaning & Deduplication

Fetched tickets are normalized and deduplicated so repeated or empty messages do not distort clustering and scoring.

## 1.4 Embedding-Based Clustering

Each cleaned ticket text is embedded using OpenAI’s embeddings API, then tickets are grouped into similarity-based clusters using cosine similarity.

## 1.5 Root Cause Analysis

Clustered ticket groups are passed to an AI agent backed by an OpenAI chat model and a structured output parser to infer likely root causes, impacted modules, severity, debugging steps, fix direction, and impact scale.

## 1.6 Severity Scoring

Clusters are scored using a weighted formula combining recurrence, sentiment, churn risk, and enterprise-customer concentration.

## 1.7 Developer Report Generation

A second AI agent converts the scored cluster analysis into a Markdown developer intelligence report.

## 1.8 Delivery Routing

A conditional node routes the generated report toward Slack/Jira and email branches depending on configuration flags. In the current implementation, only Slack is actually controlled by the IF condition; email and Jira behavior are only partially aligned with the configuration fields.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Configuration

### Overview

This block initiates the workflow on a fixed schedule and defines all tunable runtime parameters in one place. These settings drive downstream SQL filtering, clustering sensitivity, severity calculation, and delivery behavior.

### Nodes Involved

- Schedule Trigger
- Workflow Configuration

### Node Details

#### Schedule Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry-point trigger node that starts the workflow automatically.
- **Configuration choices:** Configured with an interval rule that triggers at hour `2`, which typically means one scheduled run per day at 02:00 according to the n8n instance timezone.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Workflow Configuration`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Timezone mismatch between business expectations and server/n8n timezone
  - Missed runs if the instance is down at trigger time
- **Sub-workflow reference:** None.

#### Workflow Configuration

- **Type and technical role:** `n8n-nodes-base.set`; defines reusable configuration fields.
- **Configuration choices:** Assigns these values:
  - `timeWindowDays = 7`
  - `similarityThreshold = 0.82`
  - `frequencyWeight = 0.4`
  - `sentimentWeight = 0.2`
  - `churnWeight = 0.2`
  - `tierWeight = 0.2`
  - `enableSlack = true`
  - `enableEmail = false`
  - `enableJira = false`
  - `includeOtherFields = true`
- **Key expressions or variables used:** Referenced later via `$('Workflow Configuration').first().json...`
- **Input and output connections:**
  - Input: `Schedule Trigger`
  - Output: `Fetch Feedback Data`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Misconfigured weights can produce misleading severity rankings
  - Toggle fields are not consistently enforced in downstream routing
- **Sub-workflow reference:** None.

---

## 2.2 Data Retrieval

### Overview

This block fetches support ticket records from Postgres for the configured time window. It limits the dataset to bug reports and feature requests and provides the raw text and metadata used downstream.

### Nodes Involved

- Fetch Feedback Data

### Node Details

#### Fetch Feedback Data

- **Type and technical role:** `n8n-nodes-base.postgres`; executes a SQL query against a Postgres database.
- **Configuration choices:**
  - Operation: `executeQuery`
  - Query selects:
    - `ticket_id`
    - `category`
    - `sentiment`
    - `churn_risk`
    - `account_tier`
    - `translated_text`
    - `original_language`
    - `original_message`
  - Filters:
    - `category IN ('bug','feature_request')`
    - `created_at >= NOW() - INTERVAL '{{ timeWindowDays }} days'`
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.timeWindowDays }}`
- **Input and output connections:**
  - Input: `Workflow Configuration`
  - Output: `Preprocess & Deduplicate`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**
  - Postgres credential/authentication failure
  - SQL syntax or expression interpolation issues
  - `created_at` missing or not indexed, leading to poor performance
  - Query returns no rows, causing later blocks to process an empty dataset
  - If `timeWindowDays` is not numeric, interval expression could break
- **Sub-workflow reference:** None.

---

## 2.3 Data Cleaning & Deduplication

### Overview

This block standardizes ticket text and removes duplicate or empty messages. It also preserves key metadata for later scoring and reporting.

### Nodes Involved

- Preprocess & Deduplicate

### Node Details

#### Preprocess & Deduplicate

- **Type and technical role:** `n8n-nodes-base.code`; JavaScript transformation node.
- **Configuration choices:**
  - Loads all incoming items
  - Uses `translated_text` as the main analyzable text
  - Trims and lowercases text into `clean_text`
  - Skips:
    - empty texts
    - exact duplicate normalized texts
  - Emits one item per unique message with:
    - `ticket_id`
    - `clean_text`
    - `sentiment` default `0`
    - `churn_risk` default `0`
    - `account_tier` default `'free'`
    - `original_language`
    - `original_message`
- **Key expressions or variables used:** `$input.all()`
- **Input and output connections:**
  - Input: `Fetch Feedback Data`
  - Output: `Generate Embeddings & Cluster`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Exact-string deduplication may collapse distinct tickets that share identical translated text
  - Near-duplicates are not removed unless text is exactly equal after trim/lowercase
  - If `translated_text` is null for many records, many items will be dropped
  - Large batches may increase memory use because all items are loaded at once
- **Sub-workflow reference:** None.

---

## 2.4 Embedding-Based Clustering

### Overview

This block creates semantic embeddings for each cleaned ticket and groups similar tickets into clusters using cosine similarity. It is the main pattern-discovery stage of the workflow.

### Nodes Involved

- Generate Embeddings & Cluster
- Aggregate Clusters

### Node Details

#### Generate Embeddings & Cluster

- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript logic using `fetch()` to call the OpenAI embeddings API directly and then perform clustering in memory.
- **Configuration choices:**
  - Reads `similarityThreshold` from the configuration node
  - Uses OpenAI embeddings model `text-embedding-3-small`
  - Calls `POST https://api.openai.com/v1/embeddings`
  - Clustering algorithm:
    - for each unassigned ticket, create a new cluster
    - compare subsequent tickets only to the first ticket in the cluster
    - add ticket if cosine similarity is `>= threshold`
  - Outputs flattened per-ticket items with:
    - original ticket fields
    - `cluster_id`
    - `cluster_size`
- **Key expressions or variables used:**
  - `$('Workflow Configuration').first().json.similarityThreshold`
  - hardcoded placeholder: `OPENAI_API_KEY = '<__PLACEHOLDER_VALUE__OpenAI API Key__>'`
- **Input and output connections:**
  - Input: `Preprocess & Deduplicate`
  - Output: `Aggregate Clusters`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - OpenAI API key placeholder must be replaced; otherwise all calls fail
  - API rate limits or quota exhaustion
  - Network timeouts or non-200 responses
  - Code does not validate `response.ok`; malformed API responses can cause `data.data[0]` errors
  - Very large ticket sets can become slow and expensive because one embedding request is made per ticket
  - Clustering is simplistic: it compares each candidate only with the seed ticket, not all tickets in a cluster or a centroid
  - Memory and runtime can grow significantly with larger datasets
- **Sub-workflow reference:** None.

#### Aggregate Clusters

- **Type and technical role:** `n8n-nodes-base.aggregate`; groups or aggregates incoming cluster-tagged items.
- **Configuration choices:**
  - Aggregates by the field `cluster_id`
  - This is intended to produce per-cluster grouped data for the root cause analysis stage
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Input: `Generate Embeddings & Cluster`
  - Output: `Root Cause Analysis Agent`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - The downstream nodes assume an `aggregatedData` array exists; this depends on the exact behavior of the Aggregate node configuration in the n8n version used
  - If the node is not configured to preserve all relevant fields in a grouped array, later logic may fail or produce incomplete cluster analysis
- **Sub-workflow reference:** None.

---

## 2.5 Root Cause Analysis

### Overview

This block uses an AI agent to examine each cluster and return structured engineering insights. It relies on a chat model and a strict JSON schema parser so the output can be consumed programmatically.

### Nodes Involved

- OpenAI Chat Model
- Root Cause Schema Parser
- Root Cause Analysis Agent

### Node Details

#### OpenAI Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model provider node for the AI agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - No additional tools configured
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none on main channel
  - AI output connection: to `Root Cause Analysis Agent`
- **Version-specific requirements:** Type version `1.3`; requires compatible LangChain/OpenAI node support in the installed n8n version.
- **Edge cases or potential failure types:**
  - Missing or invalid OpenAI credentials
  - Model unavailability or account access restrictions
  - Output quality may vary with large or poorly structured cluster payloads
- **Sub-workflow reference:** None.

#### Root Cause Schema Parser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; structured output parser enforcing a manual JSON schema.
- **Configuration choices:**
  - Manual schema with required fields:
    - `root_cause` string
    - `impacted_module` string
    - `severity` enum: `low | medium | high | critical`
    - `debug_steps` array of strings
    - `fix_direction` string
    - `impact_scale` enum: `low | medium | high`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none on main channel
  - AI parser connection: to `Root Cause Analysis Agent`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Model may return data that fails schema validation
  - Missing required fields or wrong enum values will cause parser errors
- **Sub-workflow reference:** None.

#### Root Cause Analysis Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI agent that analyzes grouped issue reports.
- **Configuration choices:**
  - Prompt source: `={{ $json.aggregatedData }}`
  - Uses a system message framing the model as a senior product engineer
  - Expected tasks:
    1. identify root cause
    2. determine impacted module
    3. estimate severity
    4. suggest debug steps
    5. propose fix direction
    6. estimate impact scale
  - Structured parser enabled
- **Key expressions or variables used:**
  - `$json.aggregatedData`
- **Input and output connections:**
  - Main input: `Aggregate Clusters`
  - AI language model input: `OpenAI Chat Model`
  - AI output parser input: `Root Cause Schema Parser`
  - Main output: `Calculate Severity Scores`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If `aggregatedData` is absent or malformed, analysis quality degrades or fails
  - Large cluster payloads may exceed model context window or increase cost
  - Structured parsing failures if the model response deviates from schema
- **Sub-workflow reference:** None.

---

## 2.6 Severity Scoring

### Overview

This block computes a numerical severity score for each analyzed cluster so issues can be ranked by combined technical and business importance. It also sorts all clusters descending by score.

### Nodes Involved

- Calculate Severity Scores

### Node Details

#### Calculate Severity Scores

- **Type and technical role:** `n8n-nodes-base.code`; JavaScript scoring and sorting logic.
- **Configuration choices:**
  - Reads all cluster items
  - Pulls configuration weights from `Workflow Configuration`
  - Expects ticket list in `cluster.aggregatedData`
  - Computes:
    - `ticket_count`
    - `avg_sentiment`
    - `avg_churn_risk`
    - `enterprise_ratio`
    - `severity_score`
  - Formula:
    - `frequencyWeight * log(ticketCount + 1)`
    - `+ sentimentWeight * abs(avgSentiment)`
    - `+ churnWeight * avgChurnRisk`
    - `+ tierWeight * enterpriseRatio`
  - Returns items sorted descending by `severity_score`
- **Key expressions or variables used:**
  - `$('Workflow Configuration').first().json`
- **Input and output connections:**
  - Input: `Root Cause Analysis Agent`
  - Output: `Developer Report Generator`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `aggregatedData` is missing, `tickets.length` may become `0` and cause division by zero in averages
  - The node assumes root cause analysis output still contains original aggregated cluster data; depending on agent output behavior, this may not hold
  - `severity` from AI is not incorporated into `severity_score`
  - Sentiment absolute value assumes more negative or more extreme values increase importance
- **Sub-workflow reference:** None.

---

## 2.7 Developer Report Generation

### Overview

This block turns all scored cluster records into a single technical Markdown report intended for engineers and product teams. It uses a second LLM configured as a report writer.

### Nodes Involved

- Report Generator Model
- Developer Report Generator

### Node Details

#### Report Generator Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; chat model provider for report generation.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - No built-in tools configured
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - AI output connection: to `Developer Report Generator`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Missing OpenAI credentials
  - Long payloads can increase cost or exceed context capacity
- **Sub-workflow reference:** None.

#### Developer Report Generator

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI agent generating the final Markdown report.
- **Configuration choices:**
  - Input text: `={{ JSON.stringify($input.all().map(i => i.json)) }}`
  - System prompt instructs the model to produce:
    1. Executive Summary
    2. Ranked Recurring Bugs
    3. Feature Requests Grouped
    4. Severity Ranking Table
    5. Recommended Sprint Priorities
    6. Risk Assessment
    7. Reproducible Patterns
  - Requires sample user quotes with original language noted
  - Output format: Markdown
  - Tone: technical, concise, actionable
- **Key expressions or variables used:**
  - `JSON.stringify($input.all().map(i => i.json))`
- **Input and output connections:**
  - Main input: `Calculate Severity Scores`
  - AI language model input: `Report Generator Model`
  - Main output: `Check Delivery Options`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Large JSON payload can exceed model context
  - If prior nodes dropped original quote fields, required quote context may be incomplete
  - Output is plain model text; no structured parser is used here
- **Sub-workflow reference:** None.

---

## 2.8 Delivery Routing

### Overview

This block routes the generated report to downstream delivery channels. In practice, the current IF logic only checks the Slack toggle, while the email false branch and Jira branch are not independently controlled.

### Nodes Involved

- Check Delivery Options
- Send to Slack
- Create Jira Issues
- Send Email Report

### Node Details

#### Check Delivery Options

- **Type and technical role:** `n8n-nodes-base.if`; conditional router.
- **Configuration choices:**
  - Condition: `$('Workflow Configuration').first().json.enableSlack == true`
  - True branch sends data to:
    - `Send to Slack`
    - `Create Jira Issues`
  - False branch sends data to:
    - `Send Email Report`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.enableSlack }}`
- **Input and output connections:**
  - Input: `Developer Report Generator`
  - True outputs: `Send to Slack`, `Create Jira Issues`
  - False output: `Send Email Report`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - `enableEmail` and `enableJira` are defined but not checked here
  - If Slack is enabled, Jira always runs too, even when `enableJira` is false
  - If Slack is disabled, email always runs, even when `enableEmail` is false
  - This is the main logic inconsistency in the workflow
- **Sub-workflow reference:** None.

#### Send to Slack

- **Type and technical role:** `n8n-nodes-base.slack`; posts the generated report to a Slack channel.
- **Configuration choices:**
  - Sends formatted text to a selected channel
  - Text includes a title line with today’s ISO date and `{{ $json.output }}`
  - Channel ID/name is a placeholder and must be set
- **Key expressions or variables used:**
  - `{{ new Date().toISOString().split('T')[0] }}`
  - `{{ $json.output }}`
- **Input and output connections:**
  - Input: `Check Delivery Options` true branch
  - Output: none
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases or potential failure types:**
  - Missing Slack credentials or invalid channel
  - Message length may exceed Slack limits for long reports
  - Markdown rendering in Slack will differ from standard Markdown
- **Sub-workflow reference:** None.

#### Create Jira Issues

- **Type and technical role:** `n8n-nodes-base.jira`; creates a Jira issue from incoming data.
- **Configuration choices:**
  - Project key placeholder must be replaced
  - Issue type: `Bug`
  - Summary: `{{ $json.root_cause }}`
  - Additional fields:
    - Priority mapped from `metrics.severity`
    - Description assembled from root cause analysis fields and severity metrics
- **Key expressions or variables used:**
  - `={{ $json.root_cause }}`
  - `={{ $json.metrics.severity === 'critical' ? 'Highest' : $json.metrics.severity === 'high' ? 'High' : $json.metrics.severity === 'medium' ? 'Medium' : 'Low' }}`
  - Description references:
    - `$json.root_cause`
    - `$json.impacted_module`
    - `$json.debug_steps`
    - `$json.fix_direction`
    - `$json.metrics.ticket_count`
    - `$json.metrics.severity_score`
- **Input and output connections:**
  - Input: `Check Delivery Options` true branch
  - Output: none
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - This node is wired after the final report generator, so incoming data is likely the report object, not per-cluster root cause objects
  - Expression references `metrics.severity`, but the score node creates `metrics.severity_score`, not `metrics.severity`
  - If input is only the final report, most fields referenced here will be missing
  - Jira credentials, project key, issue type, and field mappings may not match the Jira instance
- **Sub-workflow reference:** None.

#### Send Email Report

- **Type and technical role:** `n8n-nodes-base.emailSend`; sends the report as plain text email.
- **Configuration choices:**
  - Subject includes current ISO date
  - Body uses `{{ $json.output }}`
  - Email format: `text`
  - Sender and recipient are placeholders and must be configured
- **Key expressions or variables used:**
  - `={{ $json.output }}`
  - `=Developer Intelligence Report - {{ new Date().toISOString().split('T')[0] }}`
- **Input and output connections:**
  - Input: `Check Delivery Options` false branch
  - Output: none
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - SMTP or email credential configuration issues
  - If `enableSlack` is true, email never runs regardless of `enableEmail`
  - Plain text email may reduce readability of Markdown formatting
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry point |  | Workflow Configuration | ## Trigger & Config<br>Scheduled run with scoring weights and feature toggles |
| Workflow Configuration | n8n-nodes-base.set | Central runtime configuration store | Schedule Trigger | Fetch Feedback Data | ## Trigger & Config<br>Scheduled run with scoring weights and feature toggles |
| Fetch Feedback Data | n8n-nodes-base.postgres | Query recent support tickets from Postgres | Workflow Configuration | Preprocess & Deduplicate | ## Data Fetching<br>Query recent support tickets from database) |
| Preprocess & Deduplicate | n8n-nodes-base.code | Normalize text and remove duplicates | Fetch Feedback Data | Generate Embeddings & Cluster | ## Data Cleaning<br>Normalize, deduplicate, and prepare ticket text |
| Generate Embeddings & Cluster | n8n-nodes-base.code | Generate embeddings and assign similarity clusters | Preprocess & Deduplicate | Aggregate Clusters | ## Clustering Engine<br>Generate embeddings and group similar issues |
| Aggregate Clusters | n8n-nodes-base.aggregate | Group tickets by cluster ID | Generate Embeddings & Cluster | Root Cause Analysis Agent |  |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for root cause analysis |  | Root Cause Analysis Agent | ## Root Cause Analysis<br>AI identifies cause, severity, and fix direction |
| Root Cause Schema Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON schema for root cause output |  | Root Cause Analysis Agent | ## Root Cause Analysis<br>AI identifies cause, severity, and fix direction |
| Root Cause Analysis Agent | @n8n/n8n-nodes-langchain.agent | Analyze each cluster for root cause and fixes | Aggregate Clusters; OpenAI Chat Model; Root Cause Schema Parser | Calculate Severity Scores | ## Root Cause Analysis<br>AI identifies cause, severity, and fix direction |
| Calculate Severity Scores | n8n-nodes-base.code | Compute weighted cluster severity score | Root Cause Analysis Agent | Developer Report Generator | ## Severity Scoring<br>Weighted scoring using frequency and risk signals |
| Report Generator Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for final report generation |  | Developer Report Generator | ## Report Generation<br>Generate structured developer intelligence report |
| Developer Report Generator | @n8n/n8n-nodes-langchain.agent | Produce final Markdown developer report | Calculate Severity Scores; Report Generator Model | Check Delivery Options | ## Report Generation<br>Generate structured developer intelligence report |
| Check Delivery Options | n8n-nodes-base.if | Route report to delivery channels | Developer Report Generator | Send to Slack; Create Jira Issues; Send Email Report | ## Delivery Routing |
| Send to Slack | n8n-nodes-base.slack | Post report to Slack channel | Check Delivery Options |  | ## Send report via Slack and  create Jira issues |
| Create Jira Issues | n8n-nodes-base.jira | Create Jira bug issue from analysis data | Check Delivery Options |  | ## Send report via Slack and  create Jira issues |
| Send Email Report | n8n-nodes-base.emailSend | Email the generated report | Check Delivery Options |  | ## send email report |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note |  |  | ## How it works<br>This workflow runs on a schedule and analyzes recent support tickets to identify recurring issues and product insights.<br><br>It fetches feedback data from a database, cleans and deduplicates messages, and uses embeddings to group similar tickets into clusters. Each cluster is analyzed by an AI agent to detect root causes, impacted modules, severity, and possible fixes.<br><br>The workflow then calculates a weighted severity score based on frequency, sentiment, churn risk, and customer tier. A final developer report is generated with prioritized issues, recommendations, and technical insights.<br><br>Results are automatically delivered to Slack, email, or Jira depending on configuration.<br><br>## Setup steps<br>- Configure Postgres credentials for ticket data<br>- Add OpenAI API key for embeddings and analysis<br>- Set Slack, Email, or Jira credentials<br>- Adjust weights and thresholds in config node<br>- Customize schedule trigger timing |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Trigger & Config<br>Scheduled run with scoring weights and feature toggles |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Data Fetching<br>Query recent support tickets from database) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Data Cleaning<br>Normalize, deduplicate, and prepare ticket text |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Clustering Engine<br>Generate embeddings and group similar issues |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Root Cause Analysis<br>AI identifies cause, severity, and fix direction |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Severity Scoring<br>Weighted scoring using frequency and risk signals |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Report Generation<br>Generate structured developer intelligence report |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Send report via Slack and  create Jira issues |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## send email report |
| Sticky Note11 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Delivery Routing |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   `Turn support tickets into developer insights with OpenAI, Postgres, Slack and Jira`.

2. **Add a Schedule Trigger node**.
   - Node type: `Schedule Trigger`
   - Configure it to run daily at hour `2`
   - Confirm the n8n instance timezone is correct for your reporting needs

3. **Add a Set node** after the trigger and name it `Workflow Configuration`.
   - Add the following fields:
     - `timeWindowDays` → Number → `7`
     - `similarityThreshold` → Number → `0.82`
     - `frequencyWeight` → Number → `0.4`
     - `sentimentWeight` → Number → `0.2`
     - `churnWeight` → Number → `0.2`
     - `tierWeight` → Number → `0.2`
     - `enableSlack` → Boolean → `true`
     - `enableEmail` → Boolean → `false`
     - `enableJira` → Boolean → `false`
   - Keep other incoming fields if desired

4. **Add a Postgres node** named `Fetch Feedback Data`.
   - Connect `Workflow Configuration` → `Fetch Feedback Data`
   - Operation: `Execute Query`
   - Configure Postgres credentials
   - Use a query equivalent to:
     - Select `ticket_id, category, sentiment, churn_risk, account_tier, translated_text, original_language, original_message`
     - From `support_tickets`
     - Filter category to `bug` and `feature_request`
     - Filter `created_at` to the last `timeWindowDays`
   - Use an n8n expression inside the SQL interval based on the config node

5. **Add a Code node** named `Preprocess & Deduplicate`.
   - Connect `Fetch Feedback Data` → `Preprocess & Deduplicate`
   - Paste logic that:
     - loads all items
     - reads `translated_text`
     - trims and lowercases it into `clean_text`
     - skips empty strings
     - skips exact duplicate normalized texts
     - returns one item per unique text
   - Preserve these fields:
     - `ticket_id`
     - `clean_text`
     - `sentiment` with default `0`
     - `churn_risk` with default `0`
     - `account_tier` with default `'free'`
     - `original_language`
     - `original_message`

6. **Add a Code node** named `Generate Embeddings & Cluster`.
   - Connect `Preprocess & Deduplicate` → `Generate Embeddings & Cluster`
   - Implement code that:
     - reads all items
     - pulls `similarityThreshold` from `Workflow Configuration`
     - sends each `clean_text` to OpenAI embeddings endpoint
     - uses model `text-embedding-3-small`
     - computes cosine similarity between vectors
     - groups tickets into clusters using the threshold
     - returns ticket-level items enriched with:
       - `cluster_id`
       - `cluster_size`
   - Important setup:
     - Replace the placeholder OpenAI API key with a secure method
     - Preferably store the key in credentials or environment variables rather than hardcoding it
   - Recommended improvement:
     - add `response.ok` checking and retry handling

7. **Add an Aggregate node** named `Aggregate Clusters`.
   - Connect `Generate Embeddings & Cluster` → `Aggregate Clusters`
   - Configure it to group by `cluster_id`
   - Ensure each output item includes the grouped cluster data in an array that downstream nodes can reference as `aggregatedData`
   - Validate the output shape manually by testing the node

8. **Add an OpenAI Chat Model node** named `OpenAI Chat Model`.
   - Node type: OpenAI LangChain chat model
   - Configure OpenAI credentials
   - Select model `gpt-4.1-mini`

9. **Add a Structured Output Parser node** named `Root Cause Schema Parser`.
   - Use manual JSON schema
   - Define required fields:
     - `root_cause` string
     - `impacted_module` string
     - `severity` enum `low, medium, high, critical`
     - `debug_steps` array of strings
     - `fix_direction` string
     - `impact_scale` enum `low, medium, high`

10. **Add an AI Agent node** named `Root Cause Analysis Agent`.
    - Connect `Aggregate Clusters` main output into it
    - Connect `OpenAI Chat Model` to its language model input
    - Connect `Root Cause Schema Parser` to its output parser input
    - Set prompt type to define manually
    - Set text input to the grouped cluster payload, e.g. `{{$json.aggregatedData}}`
    - Set the system message to instruct the model to:
      - identify likely root cause
      - identify impacted module
      - estimate severity
      - provide debugging steps
      - suggest fix direction
      - estimate impact scale
    - Enable structured parser handling

11. **Add a Code node** named `Calculate Severity Scores`.
    - Connect `Root Cause Analysis Agent` → `Calculate Severity Scores`
    - Implement logic that:
      - reads all items
      - retrieves scoring weights from `Workflow Configuration`
      - reads cluster tickets from `aggregatedData`
      - calculates:
        - ticket count
        - average sentiment
        - average churn risk
        - enterprise ratio
        - weighted severity score
      - attaches a `metrics` object
      - sorts descending by `metrics.severity_score`
    - Recommended guardrails:
      - handle empty arrays safely
      - ensure the root cause agent output still preserves the cluster ticket array

12. **Add another OpenAI Chat Model node** named `Report Generator Model`.
    - Configure OpenAI credentials
    - Select model `gpt-4.1-mini`

13. **Add a second AI Agent node** named `Developer Report Generator`.
    - Connect `Calculate Severity Scores` → `Developer Report Generator`
    - Connect `Report Generator Model` to its language model input
    - Set text input to a JSON string of all input items, e.g. `{{ JSON.stringify($input.all().map(i => i.json)) }}`
    - Use a system message requiring a Markdown report with:
      - Executive Summary
      - Ranked Recurring Bugs
      - Feature Requests Grouped
      - Severity Ranking Table
      - Recommended Sprint Priorities
      - Risk Assessment
      - Reproducible Patterns
      - Sample user quotes with original language

14. **Add an IF node** named `Check Delivery Options`.
    - Connect `Developer Report Generator` → `Check Delivery Options`
    - Configure a boolean condition checking whether Slack delivery is enabled:
      - left value: `{{ $('Workflow Configuration').first().json.enableSlack }}`
      - operator: equals
      - right value: `true`

15. **Add a Slack node** named `Send to Slack`.
    - Connect from the **true** output of `Check Delivery Options`
    - Configure Slack credentials
    - Action: send message to a channel
    - Channel: specify the target channel ID or name
    - Message body should include:
      - report title with current date
      - `{{$json.output}}`

16. **Add a Jira node** named `Create Jira Issues`.
    - Connect it from the **true** output of `Check Delivery Options`
    - Configure Jira credentials
    - Project: set the Jira project key/id
    - Issue type: `Bug`
    - Summary: root cause field
    - Description: include root cause, impacted module, debug steps, fix direction, ticket count, severity score
    - Important: in the current design, this node likely needs a different upstream branch if you want one Jira issue per analyzed cluster
    - Recommended fix:
      - branch Jira before the final report generator, or preserve cluster-level records separately
      - add a dedicated IF node for `enableJira`

17. **Add an Email Send node** named `Send Email Report`.
    - Connect it from the **false** output of `Check Delivery Options`
    - Configure SMTP or email service credentials
    - Set:
      - recipient emails
      - sender email
      - subject with current date
      - body as `{{$json.output}}`
      - format `text`
    - Recommended fix:
      - use a separate IF node for `enableEmail` rather than relying on Slack being disabled

18. **Connect all nodes exactly in this order**:
    - `Schedule Trigger` → `Workflow Configuration`
    - `Workflow Configuration` → `Fetch Feedback Data`
    - `Fetch Feedback Data` → `Preprocess & Deduplicate`
    - `Preprocess & Deduplicate` → `Generate Embeddings & Cluster`
    - `Generate Embeddings & Cluster` → `Aggregate Clusters`
    - `Aggregate Clusters` → `Root Cause Analysis Agent`
    - `OpenAI Chat Model` → `Root Cause Analysis Agent` via AI language model connector
    - `Root Cause Schema Parser` → `Root Cause Analysis Agent` via AI output parser connector
    - `Root Cause Analysis Agent` → `Calculate Severity Scores`
    - `Calculate Severity Scores` → `Developer Report Generator`
    - `Report Generator Model` → `Developer Report Generator` via AI language model connector
    - `Developer Report Generator` → `Check Delivery Options`
    - `Check Delivery Options` true → `Send to Slack`
    - `Check Delivery Options` true → `Create Jira Issues`
    - `Check Delivery Options` false → `Send Email Report`

19. **Configure credentials**:
    - **Postgres:** host, port, database, username, password, SSL if required
    - **OpenAI:** API key for both chat model nodes; also secure access for the custom embeddings code
    - **Slack:** OAuth/app token with channel posting permissions
    - **Jira:** cloud/server credentials and project access
    - **Email:** SMTP or provider credentials

20. **Test each stage incrementally**.
    - Run the Postgres query first
    - Inspect deduplicated items
    - Validate embedding and clustering output shape
    - Confirm `Aggregate Clusters` emits the expected grouped array
    - Check structured root cause analysis output against schema
    - Verify `metrics` object exists before report generation
    - Confirm `Developer Report Generator` returns a field like `output`
    - Test Slack/email/Jira nodes with sample data

21. **Recommended production hardening**:
    - Add empty-result handling after Postgres
    - Replace hardcoded API key in the Code node with credentials/environment variables
    - Add retry/error handling for OpenAI embedding requests
    - Split delivery logic into three independent branches:
      - Slack IF
      - Email IF
      - Jira IF
    - Rewire Jira creation to consume cluster-level analysis rather than the final report text

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow runs on a schedule and analyzes recent support tickets to identify recurring issues and product insights. | General workflow behavior |
| It fetches feedback data from a database, cleans and deduplicates messages, and uses embeddings to group similar tickets into clusters. | Processing pipeline |
| Each cluster is analyzed by an AI agent to detect root causes, impacted modules, severity, and possible fixes. | AI analysis stage |
| The workflow then calculates a weighted severity score based on frequency, sentiment, churn risk, and customer tier. | Scoring logic |
| A final developer report is generated with prioritized issues, recommendations, and technical insights. | Reporting stage |
| Results are automatically delivered to Slack, email, or Jira depending on configuration. | Delivery intent |
| Configure Postgres credentials for ticket data. | Setup requirement |
| Add OpenAI API key for embeddings and analysis. | Setup requirement |
| Set Slack, Email, or Jira credentials. | Setup requirement |
| Adjust weights and thresholds in config node. | Operational tuning |
| Customize schedule trigger timing. | Scheduling setup |

## Additional implementation observations

- The workflow has **one actual entry point**: `Schedule Trigger`.
- There are **no sub-workflows** or workflow-call nodes.
- The current delivery logic does **not fully honor** `enableEmail` and `enableJira`.
- The current Jira node appears **miswired semantically**, because it likely receives the final report item instead of per-cluster analysis data.
- The embedding step uses a **hardcoded API key placeholder** inside a Code node rather than standard n8n credentials, which should be corrected before production use.