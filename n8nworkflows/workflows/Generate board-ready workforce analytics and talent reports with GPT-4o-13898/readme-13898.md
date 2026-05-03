Generate board-ready workforce analytics and talent reports with GPT-4o

https://n8nworkflows.xyz/workflows/generate-board-ready-workforce-analytics-and-talent-reports-with-gpt-4o-13898


# Generate board-ready workforce analytics and talent reports with GPT-4o

# 1. Workflow Overview

This workflow automates recurring workforce analytics and board-level talent reporting. It ingests employee records, computes a skill similarity index, prepares a consolidated analytics dataset, then uses a multi-agent GPT-4o setup to generate an executive-ready report containing attrition analysis, talent strategy recommendations, and structured board-report JSON. The final report is stored in an n8n Data Table and can optionally be delivered to an external webhook endpoint.

Typical use cases include:
- Weekly or monthly CHRO / HR leadership reporting
- Board-ready workforce transformation summaries
- Attrition risk monitoring with explainability
- Internal mobility, succession planning, and bias detection support

## 1.1 Scheduled Data Ingestion
The workflow starts on a schedule and loads employee records from an n8n Data Table.

## 1.2 Workforce Data Aggregation and Skill Indexing
Loaded employee records are aggregated and transformed into a skill similarity index and a normalized analytics dataset.

## 1.3 Multi-Agent AI Orchestration
A main orchestrator agent coordinates two specialized AI tools:
- Workforce Analytics Agent
- Talent Strategy Agent

These agents use dedicated chat models and supporting tools such as SHAP-style risk scoring, statistics, and skill similarity search.

## 1.4 Structured Report Generation
The orchestrator is constrained by a structured JSON schema so its output is machine-usable and consistently formatted for downstream storage.

## 1.5 Report Persistence and Optional Delivery
The generated board report is reformatted for storage, inserted into a report table, and optionally POSTed to an external endpoint.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Data Ingestion

### Overview
This block triggers the reporting process on a recurring schedule and retrieves the employee dataset from an n8n Data Table. It is the workflow entry point and establishes the raw input used by all downstream analytics and AI operations.

### Nodes Involved
- Schedule Workforce Analysis
- Load Employee Dataset

### Node Details

#### Schedule Workforce Analysis
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow trigger node.
- **Configuration choices:** Configured with a weekly interval. It triggers at hour 6 on day 1 of the configured weekly rule, which in n8n usually represents a scheduled weekday slot.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Load Employee Dataset**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Timezone mismatch between expected business schedule and n8n instance timezone
  - Misinterpretation of weekday setting if instance locale/team assumptions differ
  - Workflow inactive state prevents trigger execution
- **Sub-workflow reference:** None.

#### Load Employee Dataset
- **Type and technical role:** `n8n-nodes-base.dataTable`; reads records from an n8n Data Table.
- **Configuration choices:** Uses operation `get` against a configured Data Table ID placeholder for employee data.
- **Key expressions or variables used:** Data Table ID is externalized as a placeholder: `<__PLACEHOLDER_VALUE__employee_data_table__>`.
- **Input and output connections:** Input from **Schedule Workforce Analysis**; output to **Aggregate Employee Records**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Invalid or missing Data Table ID
  - Table schema mismatch with downstream expectations
  - Empty table producing no useful analytics
  - Missing expected fields such as `employee_id`, `skills`, `engagement_score`
- **Sub-workflow reference:** None.

---

## 2.2 Workforce Data Aggregation and Skill Indexing

### Overview
This block transforms raw employee rows into analytics-ready structures. It aggregates all records, computes a pairwise skill similarity matrix, and packages employee and similarity data into a single JSON object for AI analysis.

### Nodes Involved
- Aggregate Employee Records
- Build Skill Similarity Index
- Prepare Analytics Dataset

### Node Details

#### Aggregate Employee Records
- **Type and technical role:** `n8n-nodes-base.aggregate`; combines incoming items.
- **Configuration choices:** Uses `aggregateAllItemData`, meaning all incoming employee rows are grouped for downstream processing.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Load Employee Dataset**; output to **Build Skill Similarity Index**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If input is empty, downstream code may still run but produce empty indexes
  - Large datasets can increase memory use because all items are collected
- **Sub-workflow reference:** None.

#### Build Skill Similarity Index
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript transform.
- **Configuration choices:** Iterates over all employee items, normalizes skills to lowercase, builds:
  - `skill_index`: employee-centric metadata and skills
  - `similarity_matrix`: pairwise Jaccard similarity scores between employees
  - `indexed_at`: generation timestamp
- **Key expressions or variables used:**
  - `$input.all()`
  - Jaccard similarity based on skills intersection / union
- **Input and output connections:** Input from **Aggregate Employee Records**; output to **Prepare Analytics Dataset**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Non-array `skills` fields are treated as empty arrays
  - Missing `employee_id` can corrupt index keys
  - Duplicate employee IDs will overwrite previous entries in `skill_index`
  - Large employee populations create an O(n²) similarity matrix and may become expensive
- **Sub-workflow reference:** None.

#### Prepare Analytics Dataset
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript consolidation node.
- **Configuration choices:** Pulls original employee records directly from **Load Employee Dataset** and merges them with the computed skill index / similarity matrix from the immediate input.
- **Key expressions or variables used:**
  - `$('Load Employee Dataset').all()`
  - `$input.first().json`
  - Produces:
    - `employees`
    - `skill_similarity_index`
    - `similarity_matrix`
    - `total_count`
    - `analysis_date`
- **Input and output connections:** Input from **Build Skill Similarity Index**; output to **Main Orchestrator Agent**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If **Load Employee Dataset** name changes, expression lookup breaks
  - Missing expected employee fields propagate as `undefined`
  - If upstream returns no item, `$input.first()` can fail
  - Large employee arrays may exceed model context limits later
- **Sub-workflow reference:** None.

---

## 2.3 Multi-Agent AI Orchestration

### Overview
This is the core intelligence layer. A main orchestrator agent receives the prepared dataset and is instructed to invoke both a workforce analytics specialist and a talent strategy specialist, then synthesize their outputs into one board-ready report.

### Nodes Involved
- Main Orchestrator Agent
- Orchestrator Chat Model
- Board Report JSON Schema
- Workforce Analytics Agent
- Analytics Agent Chat Model
- SHAP Value Calculator Tool
- Statistical Calculator Tool
- Talent Strategy Agent
- Strategy Agent Chat Model
- Skill Similarity Search Tool1

### Node Details

#### Main Orchestrator Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; top-level LLM agent.
- **Configuration choices:**
  - Prompt type is explicitly defined.
  - Main input text is `={{ $json.employees }}`, so it sends the employee array as the primary user content.
  - System message assigns the role of Chief Workforce Intelligence Officer.
  - It is explicitly instructed to always invoke both specialized agents and integrate their outputs.
  - Structured output parsing is enabled.
- **Key expressions or variables used:**
  - `={{ $json.employees }}`
- **Input and output connections:**
  - Main input from **Prepare Analytics Dataset**
  - AI language model from **Orchestrator Chat Model**
  - AI tools: **Workforce Analytics Agent**, **Talent Strategy Agent**
  - AI output parser: **Board Report JSON Schema**
  - Main output to **Prepare Report Storage**
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain agent support in n8n.
- **Edge cases or potential failure types:**
  - Model may ignore instruction to use both tools unless the node/tool behavior enforces it strongly
  - Input token size may be too large for big employee datasets
  - Structured parser may fail if output is incomplete or malformed
  - Tool outputs may be internally inconsistent
- **Sub-workflow reference:** None; this is not a separate workflow call.

#### Orchestrator Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend for the orchestrator.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for relatively stable executive synthesis
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model connection into **Main Orchestrator Agent**.
- **Version-specific requirements:** Type version `1.3`; requires OpenAI credentials.
- **Edge cases or potential failure types:**
  - Invalid API credentials
  - Model access unavailable in the OpenAI account
  - Rate limits or quota exhaustion
  - API timeout on large prompts/tool loops
- **Sub-workflow reference:** None.

#### Board Report JSON Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces a JSON schema on the final orchestrator output.
- **Configuration choices:** Manual JSON schema defining:
  - `executive_summary`
  - `attrition_analysis`
  - `talent_strategy`
  - `strategic_recommendations`
  - `analysis_metadata`
  Required fields are the first four.
- **Key expressions or variables used:** None.
- **Input and output connections:** AI output parser connected into **Main Orchestrator Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - LLM output may not satisfy required schema
  - Optional nested objects can still vary in shape because internal child schemas are broad
  - If downstream expects consistent nested keys, schema may be too permissive
- **Sub-workflow reference:** None.

#### Workforce Analytics Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; agent exposed as a tool to another agent.
- **Configuration choices:**
  - Receives `employee_data` via `$fromAI(...)`
  - System message focuses on predictive HR analytics and explainable AI
  - Instructed to use SHAP tool and analyze tenure, performance, engagement, skills, promotion delays
- **Key expressions or variables used:**
  - `={{ $fromAI('employee_data', 'Complete employee dataset including skills, tenure, performance, engagement, and promotion history') }}`
- **Input and output connections:**
  - AI tool connection into **Main Orchestrator Agent**
  - AI language model from **Analytics Agent Chat Model**
  - AI tools from **SHAP Value Calculator Tool** and **Statistical Calculator Tool**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Parent agent may pass incomplete `employee_data`
  - Tool-use plans may be weak if the prompt is underspecified
  - Long employee datasets may exceed context or tool invocation practicality
- **Sub-workflow reference:** None.

#### Analytics Agent Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend for the workforce analytics agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1` for deterministic analytical behavior
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model connection into **Workforce Analytics Agent**.
- **Version-specific requirements:** Type version `1.3`; requires OpenAI credentials.
- **Edge cases or potential failure types:** same family as other OpenAI chat model nodes: credentials, rate limiting, unavailable model, timeout.
- **Sub-workflow reference:** None.

#### SHAP Value Calculator Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`; code-based AI tool.
- **Configuration choices:**
  - Expects an `employee_data` object via `$fromAI`
  - Computes simplified SHAP-like feature contributions from:
    - `tenure_years`
    - `performance_rating`
    - `engagement_score`
    - `last_promotion_months`
    - `skills.length`
  - Uses a baseline risk of `0.15`
  - Returns:
    - `risk_score`
    - `risk_level`
    - `shap_values`
    - `top_risk_factors`
    - `employee_id`
- **Key expressions or variables used:**
  - `$fromAI('employee_data', ..., 'object')`
- **Input and output connections:** AI tool into **Workforce Analytics Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - This is a heuristic calculator, not a true SHAP implementation
  - Negative/positive contribution logic may not align with real attrition science
  - Missing `employee_id` or malformed employee objects reduce usefulness
  - If passed the entire dataset instead of a single record, outputs may be nonsensical
- **Sub-workflow reference:** None.

#### Statistical Calculator Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCalculator`; built-in calculator tool.
- **Configuration choices:** No custom parameters; available to both specialist agents.
- **Key expressions or variables used:** None.
- **Input and output connections:** AI tool into **Workforce Analytics Agent** and **Talent Strategy Agent**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - LLM may formulate invalid expressions
  - Useful only for relatively simple arithmetic/statistical reasoning unless carefully prompted
- **Sub-workflow reference:** None.

#### Talent Strategy Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; strategy-focused agent exposed as a tool.
- **Configuration choices:**
  - Receives `analytics_insights` via `$fromAI`
  - System message focuses on org design, succession planning, internal mobility, and diversity analytics
  - Explicitly told to pass `similarity_matrix` and `skill_index` to the skill search tool
- **Key expressions or variables used:**
  - `={{ $fromAI('analytics_insights', 'Workforce analytics insights including attrition risks and employee profiles') }}`
- **Input and output connections:**
  - AI tool into **Main Orchestrator Agent**
  - AI language model from **Strategy Agent Chat Model**
  - AI tools from **Statistical Calculator Tool** and **Skill Similarity Search Tool1**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - The parent orchestrator only sends `employees` directly as main text, so the strategy agent depends on proper inter-agent context formulation
  - If `similarity_matrix` and `skill_index` are not included in the passed insights, the skill search tool cannot function well
  - Bias detection quality depends on presence and quality of demographic fields
- **Sub-workflow reference:** None.

#### Strategy Agent Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend for strategy analysis.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3` for slightly more generative strategic recommendations
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model connection into **Talent Strategy Agent**.
- **Version-specific requirements:** Type version `1.3`; requires OpenAI credentials.
- **Edge cases or potential failure types:** same OpenAI issues as above.
- **Sub-workflow reference:** None.

#### Skill Similarity Search Tool1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`; code-based AI tool for mobility/succession matching.
- **Configuration choices:**
  - Expects:
    - `employee_id`
    - `top_n` default `5`
    - `similarity_matrix`
    - `skill_index`
  - Validates availability of similarity data
  - Returns top similar employees with metadata
- **Key expressions or variables used:**
  - `$fromAI('employee_id', ..., 'string')`
  - `$fromAI('top_n', ..., 'number', 5)`
  - `$fromAI('similarity_matrix', ..., 'object')`
  - `$fromAI('skill_index', ..., 'object')`
- **Input and output connections:** AI tool into **Talent Strategy Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Employee not found in matrix
  - Missing or malformed matrix/index objects
  - Similarity scores based only on skills overlap, not proficiency or recency
  - Duplicate/incorrect employee IDs undermine lookup integrity
- **Sub-workflow reference:** None.

---

## 2.4 Structured Report Generation and Storage Preparation

### Overview
This block converts the AI output into a storage-friendly shape. It extracts the parsed structured report, adds reporting metadata, and serializes nested objects for insertion into a table.

### Nodes Involved
- Prepare Report Storage

### Node Details

#### Prepare Report Storage
- **Type and technical role:** `n8n-nodes-base.set`; maps and formats output fields.
- **Configuration choices:** Creates:
  - `report_id` from formatted current timestamp
  - `report_date` as ISO timestamp
  - `executive_summary`
  - `attrition_analysis` as JSON string
  - `talent_strategy` as JSON string
  - `strategic_recommendations` as JSON string
  - `full_report_json` as JSON string
- **Key expressions or variables used:**
  - `={{ $now.format('yyyyMMdd-HHmmss') }}`
  - `={{ $now.toISO() }}`
  - `={{ $json.output.executive_summary }}`
  - `={{ JSON.stringify($json.output.attrition_analysis) }}`
  - `={{ JSON.stringify($json.output.talent_strategy) }}`
  - `={{ JSON.stringify($json.output.strategic_recommendations) }}`
  - `={{ JSON.stringify($json.output) }}`
- **Input and output connections:** Input from **Main Orchestrator Agent**; output to **Store Workforce Report**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If parser output is absent, `$json.output...` expressions fail
  - Large report JSON may exceed storage field limits depending on table design
  - Stringifying nested objects reduces queryability unless storage schema is designed for JSON strings
- **Sub-workflow reference:** None.

---

## 2.5 Report Persistence and Optional Delivery

### Overview
This block persists the final report and optionally sends it to an external system. The delivery step is non-blocking because its error behavior is configured to continue regular output.

### Nodes Involved
- Store Workforce Report
- Optional Report Delivery

### Node Details

#### Store Workforce Report
- **Type and technical role:** `n8n-nodes-base.dataTable`; inserts or maps prepared fields into an n8n Data Table.
- **Configuration choices:**
  - Uses auto-mapping of input data to columns
  - Writes to a report table placeholder: `<__PLACEHOLDER_VALUE__workforce_reports_table__>`
- **Key expressions or variables used:** Table ID placeholder.
- **Input and output connections:** Input from **Prepare Report Storage**; output to **Optional Report Delivery**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Report table missing required columns
  - Auto-map may fail or silently mismatch if column names differ
  - Long JSON strings can exceed storage constraints
- **Sub-workflow reference:** None.

#### Optional Report Delivery
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends the final report to an external endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL placeholder: `<__PLACEHOLDER_VALUE__report_delivery_endpoint__>`
  - Sends body parameters:
    - `report_id`
    - `report_date`
    - `report_data`
  - `onError` is set to continue regular output
- **Key expressions or variables used:**
  - `={{ $json.report_id }}`
  - `={{ $json.report_date }}`
  - `={{ $json.full_report_json }}`
- **Input and output connections:** Input from **Store Workforce Report**; no downstream nodes.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Invalid endpoint URL
  - 4xx/5xx remote API failures
  - Authentication may be required but is not configured here
  - Large payload rejection by receiving service
  - Because errors are tolerated, delivery failures may go unnoticed unless execution logs are monitored
- **Sub-workflow reference:** None.

---

## 2.6 Documentation / Sticky Note Layer

### Overview
These nodes are purely informational and do not execute logic. They document prerequisites, setup, workflow purpose, and the meaning of major visual sections.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Lists prerequisites, use cases, customization, and benefits.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None functionally.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Lists setup steps for credentials, data source, schedule, storage, delivery, and JSON schema alignment.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None functionally.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Explains the end-to-end business purpose and multi-agent design.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None functionally.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual grouping note.
- **Configuration choices:** Labels the aggregation/indexing section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual grouping note.
- **Configuration choices:** Labels the orchestrator section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual grouping note.
- **Configuration choices:** Labels analytics and strategy tools/agents section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual grouping note.
- **Configuration choices:** Labels reporting, storage, and delivery section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Workforce Analysis | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry point |  | Load Employee Dataset | ## How It Works<br>This workflow automates end-to-end workforce analytics and board-level talent strategy reporting using a multi-agent AI orchestration system. Designed for HR leaders, people analytics teams, and CHROs, it eliminates manual effort in compiling workforce insights and translating them into executive-ready reports. The pipeline begins with a scheduled trigger that loads employee datasets and aggregates HR records. It then builds a skill similarity index and prepares a structured analytics dataset. A Main Orchestrator Agent coordinates two specialised sub-agents: a Workforce Analytics Agent (using SHAP value analysis and statistical tools) and a Talent Strategy Agent (leveraging skill similarity search). Results are parsed into a Board Report JSON schema, stored in a report repository, and optionally delivered via webhook. The system enables data-driven talent decisions at scale. |
| Load Employee Dataset | n8n-nodes-base.dataTable | Reads employee records from Data Table | Schedule Workforce Analysis | Aggregate Employee Records | ## How It Works<br>This workflow automates end-to-end workforce analytics and board-level talent strategy reporting using a multi-agent AI orchestration system. Designed for HR leaders, people analytics teams, and CHROs, it eliminates manual effort in compiling workforce insights and translating them into executive-ready reports. The pipeline begins with a scheduled trigger that loads employee datasets and aggregates HR records. It then builds a skill similarity index and prepares a structured analytics dataset. A Main Orchestrator Agent coordinates two specialised sub-agents: a Workforce Analytics Agent (using SHAP value analysis and statistical tools) and a Talent Strategy Agent (leveraging skill similarity search). Results are parsed into a Board Report JSON schema, stored in a report repository, and optionally delivered via webhook. The system enables data-driven talent decisions at scale.<br>## Aggregate & Index<br>**What** — Aggregates employee records and builds a skill similarity index.<br>**Why** — Structures raw data into analytics-ready format for accurate modelling. |
| Aggregate Employee Records | n8n-nodes-base.aggregate | Aggregates all employee rows for indexing | Load Employee Dataset | Build Skill Similarity Index | ## Aggregate & Index<br>**What** — Aggregates employee records and builds a skill similarity index.<br>**Why** — Structures raw data into analytics-ready format for accurate modelling. |
| Build Skill Similarity Index | n8n-nodes-base.code | Builds skill index and pairwise similarity matrix | Aggregate Employee Records | Prepare Analytics Dataset | ## Aggregate & Index<br>**What** — Aggregates employee records and builds a skill similarity index.<br>**Why** — Structures raw data into analytics-ready format for accurate modelling. |
| Prepare Analytics Dataset | n8n-nodes-base.code | Merges employee dataset with similarity artifacts | Build Skill Similarity Index | Main Orchestrator Agent | ## Aggregate & Index<br>**What** — Aggregates employee records and builds a skill similarity index.<br>**Why** — Structures raw data into analytics-ready format for accurate modelling. |
| Main Orchestrator Agent | @n8n/n8n-nodes-langchain.agent | Coordinates specialist agents and produces final structured report | Prepare Analytics Dataset | Prepare Report Storage | ## Prerequisites<br>- OpenAI or compatible LLM API credentials<br>- Employee dataset (CSV, Google Sheets, or DB)<br>- Webhook endpoint or email (optional delivery)<br>## Use Cases<br>- Automated monthly board talent reports for CHROs<br>## Customisation<br>- Swap LLM models per agent for cost/performance balance<br>## Benefits<br>- Eliminates manual HR reporting effort<br>## Orchestrator Agent<br>**What** — Coordinates Workforce Analytics and Talent Strategy sub-agents.<br>**Why** — Decomposes complex analysis into specialised tasks for better accuracy. |
| Orchestrator Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for orchestrator |  | Main Orchestrator Agent | ## Setup Steps<br>1. Add OpenAI or compatible LLM credentials for all Chat Model nodes.<br>2. Configure employee dataset source (e.g., Google Sheets, database, or CSV node).<br>3. Set the Schedule Trigger interval (daily/weekly) to match reporting cadence.<br>4. Update the `Prepare Report Storage` node with your target storage path or bucket.<br>5. Configure `Optional Report Delivery` webhook URL or email endpoint if needed.<br>6. Verify the Board Report JSON Schema matches your organisation's reporting fields. |
| Board Report JSON Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured board-report output |  | Main Orchestrator Agent | ## Setup Steps<br>1. Add OpenAI or compatible LLM credentials for all Chat Model nodes.<br>2. Configure employee dataset source (e.g., Google Sheets, database, or CSV node).<br>3. Set the Schedule Trigger interval (daily/weekly) to match reporting cadence.<br>4. Update the `Prepare Report Storage` node with your target storage path or bucket.<br>5. Configure `Optional Report Delivery` webhook URL or email endpoint if needed.<br>6. Verify the Board Report JSON Schema matches your organisation's reporting fields. |
| Workforce Analytics Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for attrition and explainability analysis |  | Main Orchestrator Agent | ## Orchestrator Agent<br>**What** — Coordinates Workforce Analytics and Talent Strategy sub-agents.<br>**Why** — Decomposes complex analysis into specialised tasks for better accuracy.<br>## Analytics & Strategy<br>**What** — Runs SHAP analysis, statistical tools, and skill search.<br>**Why** — Generates explainable, evidence-based workforce and talent insights. |
| Analytics Agent Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for analytics agent |  | Workforce Analytics Agent | ## Analytics & Strategy<br>**What** — Runs SHAP analysis, statistical tools, and skill search.<br>**Why** — Generates explainable, evidence-based workforce and talent insights. |
| SHAP Value Calculator Tool | @n8n/n8n-nodes-langchain.toolCode | Computes heuristic SHAP-like attrition explanations |  | Workforce Analytics Agent | ## Analytics & Strategy<br>**What** — Runs SHAP analysis, statistical tools, and skill search.<br>**Why** — Generates explainable, evidence-based workforce and talent insights. |
| Statistical Calculator Tool | @n8n/n8n-nodes-langchain.toolCalculator | Shared calculator for quantitative reasoning |  | Workforce Analytics Agent; Talent Strategy Agent | ## Analytics & Strategy<br>**What** — Runs SHAP analysis, statistical tools, and skill search.<br>**Why** — Generates explainable, evidence-based workforce and talent insights. |
| Talent Strategy Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for mobility, succession, and bias analysis |  | Main Orchestrator Agent | ## Orchestrator Agent<br>**What** — Coordinates Workforce Analytics and Talent Strategy sub-agents.<br>**Why** — Decomposes complex analysis into specialised tasks for better accuracy.<br>## Analytics & Strategy<br>**What** — Runs SHAP analysis, statistical tools, and skill search.<br>**Why** — Generates explainable, evidence-based workforce and talent insights. |
| Strategy Agent Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for strategy agent |  | Talent Strategy Agent | ## Analytics & Strategy<br>**What** — Runs SHAP analysis, statistical tools, and skill search.<br>**Why** — Generates explainable, evidence-based workforce and talent insights. |
| Skill Similarity Search Tool1 | @n8n/n8n-nodes-langchain.toolCode | Finds employees with similar skills for mobility/succession use cases |  | Talent Strategy Agent | ## Analytics & Strategy<br>**What** — Runs SHAP analysis, statistical tools, and skill search.<br>**Why** — Generates explainable, evidence-based workforce and talent insights. |
| Prepare Report Storage | n8n-nodes-base.set | Formats structured AI output for storage | Main Orchestrator Agent | Store Workforce Report | ## Report & Deliver<br>**What** — Formats output as board JSON, stores it, and optionally sends via webhook.<br>**Why** — Produces board-ready reports with zero manual formatting. |
| Store Workforce Report | n8n-nodes-base.dataTable | Persists final report in Data Table | Prepare Report Storage | Optional Report Delivery | ## Report & Deliver<br>**What** — Formats output as board JSON, stores it, and optionally sends via webhook.<br>**Why** — Produces board-ready reports with zero manual formatting. |
| Optional Report Delivery | n8n-nodes-base.httpRequest | Sends report to external endpoint | Store Workforce Report |  | ## Report & Deliver<br>**What** — Formats output as board JSON, stores it, and optionally sends via webhook.<br>**Why** — Produces board-ready reports with zero manual formatting. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for prerequisites and benefits |  |  | ## Prerequisites<br>- OpenAI or compatible LLM API credentials<br>- Employee dataset (CSV, Google Sheets, or DB)<br>- Webhook endpoint or email (optional delivery)<br>## Use Cases<br>- Automated monthly board talent reports for CHROs<br>## Customisation<br>- Swap LLM models per agent for cost/performance balance<br>## Benefits<br>- Eliminates manual HR reporting effort |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual setup guidance |  |  | ## Setup Steps<br>1. Add OpenAI or compatible LLM credentials for all Chat Model nodes.<br>2. Configure employee dataset source (e.g., Google Sheets, database, or CSV node).<br>3. Set the Schedule Trigger interval (daily/weekly) to match reporting cadence.<br>4. Update the `Prepare Report Storage` node with your target storage path or bucket.<br>5. Configure `Optional Report Delivery` webhook URL or email endpoint if needed.<br>6. Verify the Board Report JSON Schema matches your organisation's reporting fields. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual overview of workflow purpose and flow |  |  | ## How It Works<br>This workflow automates end-to-end workforce analytics and board-level talent strategy reporting using a multi-agent AI orchestration system. Designed for HR leaders, people analytics teams, and CHROs, it eliminates manual effort in compiling workforce insights and translating them into executive-ready reports. The pipeline begins with a scheduled trigger that loads employee datasets and aggregates HR records. It then builds a skill similarity index and prepares a structured analytics dataset. A Main Orchestrator Agent coordinates two specialised sub-agents: a Workforce Analytics Agent (using SHAP value analysis and statistical tools) and a Talent Strategy Agent (leveraging skill similarity search). Results are parsed into a Board Report JSON schema, stored in a report repository, and optionally delivered via webhook. The system enables data-driven talent decisions at scale. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual section label for aggregation and indexing |  |  | ## Aggregate & Index<br>**What** — Aggregates employee records and builds a skill similarity index.<br>**Why** — Structures raw data into analytics-ready format for accurate modelling. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual section label for orchestrator |  |  | ## Orchestrator Agent<br>**What** — Coordinates Workforce Analytics and Talent Strategy sub-agents.<br>**Why** — Decomposes complex analysis into specialised tasks for better accuracy. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual section label for analytics and strategy tools |  |  | ## Analytics & Strategy<br>**What** — Runs SHAP analysis, statistical tools, and skill search.<br>**Why** — Generates explainable, evidence-based workforce and talent insights. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual section label for reporting and delivery |  |  | ## Report & Deliver<br>**What** — Formats output as board JSON, stores it, and optionally sends via webhook.<br>**Why** — Produces board-ready reports with zero manual formatting. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Intelligent workforce analytics and talent strategy report automation`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Schedule Workforce Analysis`
   - Configure a weekly schedule.
   - Set it to run at hour `6`.
   - Set the target day according to your reporting cadence.
   - Keep in mind the n8n instance timezone.

3. **Add the employee source node**
   - Node type: **Data Table**
   - Name: `Load Employee Dataset`
   - Operation: `Get`
   - Select your employee Data Table.
   - Ensure the table contains at minimum fields equivalent to:
     - `employee_id`
     - `name`
     - `department`
     - `role`
     - `tenure_years`
     - `performance_rating`
     - `engagement_score`
     - `promotion_history`
     - `skills`
     - `demographics`
     - `last_promotion_months`
   - Connect `Schedule Workforce Analysis -> Load Employee Dataset`.

4. **Add an Aggregate node**
   - Node type: **Aggregate**
   - Name: `Aggregate Employee Records`
   - Aggregate mode: `Aggregate all item data`
   - Connect `Load Employee Dataset -> Aggregate Employee Records`.

5. **Add a Code node to build a skill similarity index**
   - Node type: **Code**
   - Name: `Build Skill Similarity Index`
   - Paste logic that:
     - Reads all aggregated employee items
     - Builds `skill_index`
     - Converts skills to lowercase
     - Computes pairwise Jaccard similarity
     - Returns one item containing `skill_index`, `similarity_matrix`, and timestamp
   - Connect `Aggregate Employee Records -> Build Skill Similarity Index`.

6. **Add a second Code node to prepare the analytics payload**
   - Node type: **Code**
   - Name: `Prepare Analytics Dataset`
   - Configure code so it:
     - Reads all original employees using `$('Load Employee Dataset').all()`
     - Reads similarity output from the previous node via `$input.first().json`
     - Returns one consolidated JSON item with:
       - `employees`
       - `skill_similarity_index`
       - `similarity_matrix`
       - `total_count`
       - `analysis_date`
   - Connect `Build Skill Similarity Index -> Prepare Analytics Dataset`.

7. **Add the main AI agent**
   - Node type: **AI Agent** / LangChain Agent
   - Name: `Main Orchestrator Agent`
   - Prompt mode: `Define`
   - Input text: `={{ $json.employees }}`
   - Enable structured output parsing.
   - Set the system message to define the agent as a board-level workforce intelligence orchestrator that must always call both specialist agents and synthesize their results.
   - Connect `Prepare Analytics Dataset -> Main Orchestrator Agent`.

8. **Add the orchestrator chat model**
   - Node type: **OpenAI Chat Model**
   - Name: `Orchestrator Chat Model`
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Add OpenAI credentials.
   - Connect it to the AI language model port of `Main Orchestrator Agent`.

9. **Add the structured output parser**
   - Node type: **Structured Output Parser**
   - Name: `Board Report JSON Schema`
   - Schema type: `Manual`
   - Define a JSON schema with these top-level fields:
     - `executive_summary` string
     - `attrition_analysis` object
     - `talent_strategy` object
     - `strategic_recommendations` array of strings
     - `analysis_metadata` object
   - Mark as required:
     - `executive_summary`
     - `attrition_analysis`
     - `talent_strategy`
     - `strategic_recommendations`
   - Connect it to the parser port of `Main Orchestrator Agent`.

10. **Add the workforce analytics specialist**
    - Node type: **AI Agent Tool**
    - Name: `Workforce Analytics Agent`
    - Tool input text:
      - `={{ $fromAI('employee_data', 'Complete employee dataset including skills, tenure, performance, engagement, and promotion history') }}`
    - Add a system message instructing it to:
      - predict 12-month attrition risk
      - use explainable AI
      - use the SHAP calculator tool
      - identify high-risk employees and reasons
    - Add a clear tool description.
    - Connect this node to the AI tool port of `Main Orchestrator Agent`.

11. **Add the workforce analytics chat model**
    - Node type: **OpenAI Chat Model**
    - Name: `Analytics Agent Chat Model`
    - Model: `gpt-4o`
    - Temperature: `0.1`
    - Use the same OpenAI credentials or a dedicated credential set.
    - Connect it to `Workforce Analytics Agent` as its language model.

12. **Add the SHAP-style code tool**
    - Node type: **AI Code Tool**
    - Name: `SHAP Value Calculator Tool`
    - Add code that:
      - accepts one `employee_data` object via `$fromAI`
      - derives feature values
      - calculates heuristic feature impacts
      - sums them with a baseline risk
      - returns `risk_score`, `risk_level`, `shap_values`, `top_risk_factors`, `employee_id`
    - Add a description explaining that it provides explainability for attrition scoring.
    - Connect it to the AI tool port of `Workforce Analytics Agent`.

13. **Add a calculator tool**
    - Node type: **Calculator Tool**
    - Name: `Statistical Calculator Tool`
    - No special parameters required.
    - Connect it to the AI tool port of `Workforce Analytics Agent`.

14. **Add the talent strategy specialist**
    - Node type: **AI Agent Tool**
    - Name: `Talent Strategy Agent`
    - Tool input text:
      - `={{ $fromAI('analytics_insights', 'Workforce analytics insights including attrition risks and employee profiles') }}`
    - System message should instruct it to:
      - formulate role realignment recommendations
      - create succession plans
      - identify internal mobility opportunities
      - detect demographic bias
      - use the skill similarity search tool
      - pass `similarity_matrix` and `skill_index` into that tool
    - Connect this node to the AI tool port of `Main Orchestrator Agent`.

15. **Add the strategy chat model**
    - Node type: **OpenAI Chat Model**
    - Name: `Strategy Agent Chat Model`
    - Model: `gpt-4o`
    - Temperature: `0.3`
    - Add OpenAI credentials.
    - Connect it to `Talent Strategy Agent` as its language model.

16. **Connect the calculator tool to the strategy agent as well**
    - Connect `Statistical Calculator Tool` to the AI tool port of `Talent Strategy Agent`.

17. **Add the skill similarity search code tool**
    - Node type: **AI Code Tool**
    - Name: `Skill Similarity Search Tool1`
    - Add code that:
      - accepts `employee_id`
      - optionally accepts `top_n` with default `5`
      - accepts `similarity_matrix`
      - accepts `skill_index`
      - validates employee existence
      - returns top similar employees with role, department, skills, similarity score, tenure, and performance
    - Add a description stating it is used for internal mobility, succession planning, and role realignment.
    - Connect it to the AI tool port of `Talent Strategy Agent`.

18. **Add a Set node for report storage preparation**
    - Node type: **Set**
    - Name: `Prepare Report Storage`
    - Add these fields:
      - `report_id` = formatted current timestamp
      - `report_date` = current ISO datetime
      - `executive_summary` = final parsed executive summary
      - `attrition_analysis` = JSON stringified attrition analysis
      - `talent_strategy` = JSON stringified talent strategy
      - `strategic_recommendations` = JSON stringified recommendations
      - `full_report_json` = JSON stringified full structured output
    - Use expressions equivalent to:
      - `$now.format('yyyyMMdd-HHmmss')`
      - `$now.toISO()`
      - `$json.output.executive_summary`
      - `JSON.stringify($json.output.attrition_analysis)`
      - `JSON.stringify($json.output.talent_strategy)`
      - `JSON.stringify($json.output.strategic_recommendations)`
      - `JSON.stringify($json.output)`
    - Connect `Main Orchestrator Agent -> Prepare Report Storage`.

19. **Add the report storage node**
    - Node type: **Data Table**
    - Name: `Store Workforce Report`
    - Select your report output table.
    - Configure column mapping in auto-map mode.
    - Ensure the target table supports at least:
      - `report_id`
      - `report_date`
      - `executive_summary`
      - `attrition_analysis`
      - `talent_strategy`
      - `strategic_recommendations`
      - `full_report_json`
    - Connect `Prepare Report Storage -> Store Workforce Report`.

20. **Add an optional outbound delivery node**
    - Node type: **HTTP Request**
    - Name: `Optional Report Delivery`
    - Method: `POST`
    - URL: your external webhook/reporting endpoint
    - Enable request body sending
    - Add body parameters:
      - `report_id` = `={{ $json.report_id }}`
      - `report_date` = `={{ $json.report_date }}`
      - `report_data` = `={{ $json.full_report_json }}`
    - Set node error handling to continue workflow execution on failure.
    - Connect `Store Workforce Report -> Optional Report Delivery`.

21. **Configure credentials**
    - For each OpenAI Chat Model node:
      - add valid OpenAI API credentials
      - confirm access to `gpt-4o`
    - If the HTTP endpoint requires authentication:
      - configure headers, token, or OAuth in `Optional Report Delivery`
      - this is not included in the original workflow and must be added manually if required

22. **Validate data assumptions**
    - Test with a small employee dataset first.
    - Confirm `skills` is stored as an array.
    - Confirm demographic data exists if bias analysis is expected.
    - Confirm employee IDs are unique.

23. **Run a manual execution**
    - Execute the workflow manually.
    - Inspect:
      - skill index output
      - final structured JSON
      - stored table row
      - optional HTTP delivery response

24. **Activate the workflow**
    - Once validated, activate it so the schedule trigger runs automatically.

## Reproduction Notes for AI Agents / Developers
- There are **multiple AI tool entry points**, but only one workflow trigger.
- There are **no sub-workflows** invoked via Execute Workflow nodes.
- The specialist agents are not standalone workflow branches; they are tool endpoints attached to the orchestrator.
- The current design passes the employee array directly into the orchestrator text. If datasets become large, consider:
  - summarizing upstream
  - chunking by department
  - storing embeddings/indexes externally
  - moving heavy computation out of prompt context

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: OpenAI or compatible LLM API credentials; Employee dataset (CSV, Google Sheets, or DB); Webhook endpoint or email (optional delivery) | Workflow note |
| Use case: Automated monthly board talent reports for CHROs | Workflow note |
| Customisation: Swap LLM models per agent for cost/performance balance | Workflow note |
| Benefit: Eliminates manual HR reporting effort | Workflow note |
| Setup steps: Add LLM credentials, configure employee dataset source, set trigger cadence, update report storage target, configure report delivery endpoint, verify JSON schema fields | Workflow note |
| Workflow purpose: End-to-end workforce analytics and board-level talent strategy reporting using a multi-agent AI orchestration system | Workflow note |
| Aggregate & Index: Aggregates employee records and builds a skill similarity index for analytics-ready modelling | Workflow note |
| Orchestrator Agent: Coordinates Workforce Analytics and Talent Strategy sub-agents for improved specialization | Workflow note |
| Analytics & Strategy: Runs SHAP analysis, statistics, and skill search to generate explainable insights | Workflow note |
| Report & Deliver: Formats as board JSON, stores it, and optionally sends via webhook | Workflow note |

## Additional Implementation Notes
- The workflow is currently **inactive** (`active: false`), so it will not run until activated.
- The Data Table IDs and report delivery endpoint are placeholders and must be replaced before use.
- The SHAP calculator is a simplified heuristic explainer, not a mathematically rigorous SHAP implementation.
- The largest technical risk is prompt/context size if the employee dataset grows substantially.
- The workflow assumes n8n supports the LangChain agent/tool node set used here.