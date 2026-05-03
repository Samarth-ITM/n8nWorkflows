Analyze global supply chain sustainability and risk with GPT-4o and email alerts

https://n8nworkflows.xyz/workflows/analyze-global-supply-chain-sustainability-and-risk-with-gpt-4o-and-email-alerts-12988


# Analyze global supply chain sustainability and risk with GPT-4o and email alerts

## 1. Workflow Overview

**Title:** Smart Global Supply Chain Sustainability Analyzer and Optimizer  
**User-provided title:** Analyze global supply chain sustainability and risk with GPT-4o and email alerts

**Purpose:**  
This workflow runs on a schedule, pulls supply chain operational/sustainability data from an external API, performs *parallel* AI analyses (emissions, circular economy, logistics) using GPT‑4o plus dedicated tools/parsers, merges the results, performs a final sustainability/risk assessment, then routes outcomes by risk level to send email alerts (critical/high) and store all analyses in an external storage API.

**Target use cases (from sticky notes + node intent):**
- Daily supply chain sustainability monitoring and risk alerts
- Supplier risk assessment (environmental impact + bottlenecks)
- Logistics efficiency and circular economy maturity tracking
- Centralized storage of structured analysis results for dashboards/BI

### 1.1 Scheduled Trigger & Configuration
Runs daily at a configured hour, loads workflow-level settings (API URLs, thresholds, recipients).

### 1.2 Data Acquisition
Fetches supply chain data from a configurable API endpoint.

### 1.3 Parallel Specialized AI Analysis (3 agents)
Runs three specialized AI agents in parallel:
- Emissions calculation (with benchmark + calculator tools)
- Circular economy evaluation
- Logistics optimization analysis  
Each agent is constrained to structured output via an output parser.

### 1.4 Merge + Final Sustainability/Risk Scoring (Aggregator agent)
Merges the three agent outputs, then runs a final “Sustainability Analyzer” agent (GPT‑4o) producing a single structured record including risk_level, priority_score, recommendations, and scores.

### 1.5 Risk Routing, Notification, and Storage
Switches by `risk_level` to:
- format + email CRITICAL
- format + email HIGH
- format a MEDIUM summary (no email)  
All paths store results via an HTTP POST to the storage API.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Trigger & Configuration

**Overview:**  
Starts the workflow on a schedule and defines key environment/config variables used by later nodes (API endpoints, email recipients, thresholds).

**Nodes involved:**  
- Schedule Trigger  
- Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration choices:** Runs at **06:00** (based on `triggerAtHour: 6`). Interval rule is hourly-based scheduling.
- **Inputs/outputs:** No input; outputs to **Workflow Configuration**.
- **Version notes:** typeVersion 1.3.
- **Failure/edge cases:** n8n instance timezone affects the actual 06:00. If instance time changes (DST), execution time shifts.

#### Node: Workflow Configuration
- **Type / role:** `Set` (`n8n-nodes-base.set`) — centralizes configuration.
- **Configuration choices (interpreted):**
  - `supplyChainApiUrl`: placeholder for ERP/SCM API endpoint
  - `storageApiUrl`: placeholder for storage/database API endpoint
  - `alertEmailRecipients`: comma-separated recipients
  - Thresholds: `criticalThreshold=90`, `highThreshold=70`, `mediumThreshold=40` (note: these thresholds are defined but **not used** anywhere else in current logic)
  - **Include other fields:** enabled (preserves incoming JSON, though trigger provides minimal fields).
- **Key expressions:** none (static placeholders + numeric values).
- **Outputs:** to **Fetch Supply Chain Data**.
- **Version notes:** typeVersion 3.4.
- **Failure/edge cases:** Placeholder values must be replaced; if left as placeholders, downstream HTTP nodes fail due to invalid URL.

---

### Block 2 — Data Acquisition

**Overview:**  
Fetches the latest supply chain dataset from an external API.

**Nodes involved:**  
- Fetch Supply Chain Data

#### Node: Fetch Supply Chain Data
- **Type / role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — pulls supply chain JSON payload.
- **Configuration choices:**
  - **URL:** `={{ $('Workflow Configuration').first().json.supplyChainApiUrl }}`
  - Sends header `Content-Type: application/json`
  - Defaults to GET (method not specified)
- **Input:** From **Workflow Configuration**.
- **Output connections:** Fan-out to three agents in parallel:
  - Emissions Calculator Agent
  - Circular Economy Evaluator Agent
  - Logistics Optimizer Agent
- **Version notes:** typeVersion 4.3.
- **Failure/edge cases:**
  - Authentication is not configured here; many APIs require auth headers (Bearer/API key/OAuth2). Missing auth will yield 401/403.
  - Non-JSON responses can break agent expectations.
  - Large payloads may increase latency and token usage for GPT nodes.

---

### Block 3 — Parallel AI Analysis with Specialized Tools

**Overview:**  
Three specialist GPT‑4o-driven agents analyze the same supply chain data in parallel. Each agent is paired with a structured output parser to force a predictable JSON shape. The emissions agent additionally uses two “tools” (a code tool and an HTTP tool) available to the LLM during reasoning.

**Nodes involved:**  
- OpenAI Chat Model 2, Emissions Calculator Agent, Emissions Output Parser, Carbon Footprint Calculator Tool, Industry Benchmark API Tool  
- OpenAI Chat Model 3, Circular Economy Evaluator Agent, Circular Economy Output Parser  
- OpenAI Chat Model 4, Logistics Optimizer Agent, Logistics Output Parser

#### Node: OpenAI Chat Model 2
- **Type / role:** `OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides GPT‑4o for the Emissions agent.
- **Configuration choices:** model `gpt-4o`, temperature `0.1` (low variance).
- **Credentials:** OpenAI API credential “OpenAi account”.
- **Output:** Connected via `ai_languageModel` to **Emissions Calculator Agent**.
- **Failure/edge cases:** invalid/expired OpenAI key; quota; model access restrictions; transient 5xx.

#### Node: Emissions Output Parser
- **Type / role:** `Structured Output Parser` — enforces emissions JSON schema.
- **Schema outputs:** `total_emissions_co2e`, scopes 1/2/3, `carbon_intensity`, `benchmark_comparison`, arrays of `high_emission_sources`, `reduction_opportunities`.
- **Output:** Connected via `ai_outputParser` to **Emissions Calculator Agent**.
- **Failure/edge cases:** If the agent returns non-conforming JSON, parsing fails and the workflow errors.

#### Node: Carbon Footprint Calculator Tool
- **Type / role:** `Code Tool` (`toolCode`) — callable by the emissions agent to compute CO2e breakdown.
- **Behavior summary:**
  - Accepts `query` (expects JSON string/object)
  - Uses fixed emission factors:
    - electricity: 0.5 kg CO2e/kWh
    - fuel types: diesel/gasoline/electric/hybrid/air_freight/sea_freight
    - materials: steel/aluminum/plastic/paper/glass/default
  - Returns JSON string with totals and breakdown
- **Connection:** Provided to **Emissions Calculator Agent** as `ai_tool`.
- **Failure/edge cases:**
  - If `query` is not valid JSON, returns an error string (not JSON) which can confuse the LLM and break structured output downstream.
  - Emission factors are simplistic; may not match region-specific grids or supplier-specific factors.

#### Node: Industry Benchmark API Tool
- **Type / role:** `HTTP Request Tool` — callable by emissions agent to fetch benchmarks.
- **Configuration choices:** placeholder URL `<Industry benchmark API endpoint>`.
- **Connection:** Provided to **Emissions Calculator Agent** as `ai_tool`.
- **Failure/edge cases:** Missing auth; placeholder URL; API latency; non-JSON results.

#### Node: Emissions Calculator Agent
- **Type / role:** `LangChain Agent` — performs emissions + benchmarking analysis.
- **Prompt/system message:** Emissions specialist; instructed to compute totals, scopes, carbon intensity; compare to benchmarks; use both tools; produce reduction strategies.
- **Input mapping:** `text: ={{ $json }}` (passes entire fetched supply chain payload as text/object to agent).
- **Dependencies:**
  - Language model: **OpenAI Chat Model 2**
  - Output parser: **Emissions Output Parser**
  - Tools: **Carbon Footprint Calculator Tool**, **Industry Benchmark API Tool**
- **Output:** to **Merge Agent Results**.
- **Failure/edge cases:**
  - Input payload too large → token overrun or truncation.
  - Tools may return unexpected formats; agent might hallucinate benchmarks if tool fails.
  - Parser schema is permissive (no required fields), but downstream merge expects `.json` object exists.

---

#### Node: OpenAI Chat Model 3
- **Role:** GPT‑4o for Circular Economy agent (temp 0.1).
- **Credential:** same OpenAI credential.
- **Connection:** `ai_languageModel` → Circular Economy Evaluator Agent.

#### Node: Circular Economy Output Parser
- **Role:** Enforces schema with `circular_economy_score` plus rates, lifecycle score, arrays of opportunities/recommendations.
- **Connection:** `ai_outputParser` → Circular Economy Evaluator Agent.

#### Node: Circular Economy Evaluator Agent
- **Role:** Circular economy specialist assessment and scoring.
- **Input:** `text: ={{ $json }}`
- **Dependencies:** OpenAI Chat Model 3 + Circular Economy Output Parser.
- **Output:** to **Merge Agent Results**.
- **Failure/edge cases:** Same token/format risks; if supplier data lacks waste/material fields, model may infer values (risk of non-data-driven output).

---

#### Node: OpenAI Chat Model 4
- **Role:** GPT‑4o for Logistics agent (temp 0.1).
- **Connection:** `ai_languageModel` → Logistics Optimizer Agent.

#### Node: Logistics Output Parser
- **Role:** Enforces schema with `logistics_efficiency_score`, footprint, route %, modal_split object, packaging waste, arrays of opportunities/improvements.
- **Connection:** `ai_outputParser` → Logistics Optimizer Agent.

#### Node: Logistics Optimizer Agent
- **Role:** Logistics sustainability/optimization assessment.
- **Input:** `text: ={{ $json }}`
- **Dependencies:** OpenAI Chat Model 4 + Logistics Output Parser.
- **Output:** to **Merge Agent Results**.
- **Failure/edge cases:** modal_split is an untyped object; may vary shape across runs; be cautious if storage schema expects fixed keys.

---

### Block 4 — Merge + Final Sustainability/Risk Scoring

**Overview:**  
Combines the three specialist outputs into one payload, then runs a final agent that generates a unified sustainability assessment with a risk level, priority score, bottlenecks, and recommendations in a strictly structured format.

**Nodes involved:**  
- Merge Agent Results  
- OpenAI Chat Model  
- Structured Output Parser  
- Supply Chain Sustainability Analyzer

#### Node: Merge Agent Results
- **Type / role:** `Set` — creates a merged object containing each agent’s output.
- **Key expressions/variables:**
  - `emissions_data = {{ $('Emissions Calculator Agent').first().json }}`
  - `circular_economy_data = {{ $('Circular Economy Evaluator Agent').first().json }}`
  - `logistics_data = {{ $('Logistics Optimizer Agent').first().json }}`
  - `combined_analysis_timestamp = {{ $now.toISO() }}`
- **Include other fields:** true (so it also carries forward original supply chain payload from the incoming branch that reached this node).
- **Inputs:** It receives items from **Emissions**, **Circular Economy**, and **Logistics** agents (all connect to it).
- **Important behavior note:** With multiple incoming connections, n8n will execute this node once per incoming item unless you manage merges explicitly. Because this is a **Set node** (not a Merge node), the resulting behavior can be:
  - Multiple runs producing duplicated/partial merged results if agents return at different times or multiple items.
  - `.first()` usage reduces some variability but can still mismatch across executions.
- **Output:** to **Supply Chain Sustainability Analyzer**.
- **Failure/edge cases:**
  - If any agent fails upstream, `.first()` may be undefined → expression error.
  - Parallel branch synchronization is not guaranteed; consider using a dedicated **Merge** node (e.g., “Wait/merge by position”) if deterministic pairing is required.

#### Node: OpenAI Chat Model
- **Role:** GPT‑4o for the final sustainability analyzer agent (temp 0.1).
- **Connection:** `ai_languageModel` → Supply Chain Sustainability Analyzer.

#### Node: Structured Output Parser
- **Role:** Enforces final unified schema:
  - IDs: `supply_chain_id`, `supplier_id`
  - `emissions_metric`, `resource_usage{energy_kwh, water_liters, materials_kg}`
  - `risk_level` enum: critical/high/medium/low
  - `recommended_action`, `priority_score` (0–100), `confidence_level` (0–1)
  - `analysis_timestamp`, `bottlenecks[]`
  - `circular_economy_score` (0–100), `logistics_efficiency_score` (0–100)
- **Connection:** `ai_outputParser` → Supply Chain Sustainability Analyzer.
- **Edge cases:** If the model outputs risk_level outside enum (e.g., “Critical”), parsing fails.

#### Node: Supply Chain Sustainability Analyzer
- **Type / role:** LangChain Agent — final synthesis and scoring.
- **System message:** “green economics AI agent” with deterministic, data-driven analysis; specifies risk bands:
  - CRITICAL 90–100, HIGH 70–89, MEDIUM 40–69, LOW 0–39
- **Input:** `text = {{ $json }}` (the merged payload from Merge Agent Results).
- **Output:** to **Route by Risk Level**.
- **Failure/edge cases:**
  - The workflow defines thresholds in configuration, but the agent uses fixed thresholds in prompt. This can cause inconsistency if thresholds are changed in config but not in prompt.
  - If merged payload doesn’t include expected fields, model may invent values to satisfy schema.

---

### Block 5 — Risk Routing, Email Notifications, and Storage

**Overview:**  
Routes items by `risk_level`, formats risk-specific messages, sends emails for critical/high, and stores the final structured result in an external storage API.

**Nodes involved:**  
- Route by Risk Level  
- Format Critical Risk Alert → Send Critical Alert Email  
- Format High Risk Alert → Send High Risk Email  
- Format Medium Risk Report  
- Store Analysis Results

#### Node: Route by Risk Level
- **Type / role:** `Switch` — routes based on `risk_level`.
- **Rules:**
  - Output “Critical” when `{{$json.risk_level}} == "critical"`
  - Output “High” when equals `"high"`
  - Output “Medium” when equals `"medium"`
  - Fallback output: `"extra"` (covers `"low"` and anything else)
- **Connections:** The node is wired to three downstream nodes only (Critical/High/Medium). **No node is connected to fallback**, so LOW (and unexpected values) are effectively dropped.
- **Failure/edge cases:**
  - If `risk_level` is `"low"`, the result is not stored nor reported (data loss).
  - If case differs (e.g., “Critical”), rule won’t match; it will go to fallback and be dropped.

#### Node: Format Critical Risk Alert
- **Type / role:** `Set` — builds `alert_type`, `subject`, `message_body`.
- **Key expressions:**
  - Subject includes supplier: `🚨 CRITICAL... {{ $json.supplier_id }}`
  - Body uses: IDs, score, emissions, resource usage, `bottlenecks.join("\n- ")`, recommended action, CE/logistics scores, `Math.round(confidence_level*100)`
- **Failure/edge cases:**
  - If `bottlenecks` is missing or not an array, `.join()` fails.
  - Text includes emoji in subject; some systems handle it fine, some legacy SMTP setups may not.

#### Node: Send Critical Alert Email
- **Type / role:** `Email Send` — sends critical alert to recipients.
- **Configuration choices:**
  - To: `={{ $('Workflow Configuration').first().json.alertEmailRecipients }}`
  - From: placeholder sender email
  - Subject/text from formatted fields; text-only email
- **Failure/edge cases:**
  - Requires SMTP credential/config in n8n environment (not shown in JSON).
  - If recipients are comma-separated, ensure the Email node supports that format in your n8n version/SMTP provider; sometimes it requires array or semicolon.

#### Node: Format High Risk Alert
- **Type / role:** `Set` — builds HIGH email subject/body (similar structure to critical).
- **Edge cases:** Same `.join()` risk.

#### Node: Send High Risk Email
- **Type / role:** `Email Send` — sends high-priority email.
- **Output:** connected to **Store Analysis Results**.
- **Failure/edge cases:** same SMTP concerns.

#### Node: Format Medium Risk Report
- **Type / role:** `Set` — creates a one-line `summary` plus `report_type=MEDIUM`.
- **Note:** No email is sent for medium; it proceeds to storage only.
- **Output:** to **Store Analysis Results**.

#### Node: Store Analysis Results
- **Type / role:** `HTTP Request` — persists the current JSON to storage API.
- **Configuration choices:**
  - POST to `={{ $('Workflow Configuration').first().json.storageApiUrl }}`
  - JSON body: `={{ $json }}`
  - Header Content-Type: application/json
- **Inputs:** From Send Critical Alert Email, Send High Risk Email, and Format Medium Risk Report.
- **Failure/edge cases:**
  - If storage API requires auth, it’s not configured here.
  - Different incoming shapes: critical/high path includes `subject/message_body`, medium path includes `summary`. If you expect a consistent stored schema, you may want to store the original final structured analysis separately from email formatting fields.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled workflow entry point | — | Workflow Configuration | ## Scheduled Data Acquisition and Multi-Agent Deployment<br>**Why:** Triggers automated supply chain data retrieval at configured intervals, then simultaneously deploys four specialized AI agents (Enterprise, Provider, Circular Economy, Logistics) each analyzing distinct supply chain dimensions |
| Workflow Configuration | set | Central config variables (URLs, recipients, thresholds) | Schedule Trigger | Fetch Supply Chain Data | ## Scheduled Data Acquisition and Multi-Agent Deployment<br>**Why:** Triggers automated supply chain data retrieval at configured intervals, then simultaneously deploys four specialized AI agents (Enterprise, Provider, Circular Economy, Logistics) each analyzing distinct supply chain dimensions |
| Fetch Supply Chain Data | httpRequest | Pull supply chain JSON data from external API | Workflow Configuration | Emissions Calculator Agent; Circular Economy Evaluator Agent; Logistics Optimizer Agent | ## Scheduled Data Acquisition and Multi-Agent Deployment<br>**Why:** Triggers automated supply chain data retrieval at configured intervals, then simultaneously deploys four specialized AI agents (Enterprise, Provider, Circular Economy, Logistics) each analyzing distinct supply chain dimensions |
| OpenAI Chat Model 2 | lmChatOpenAi | GPT‑4o model provider (emissions agent) | — | Emissions Calculator Agent (ai_languageModel) | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Carbon Footprint Calculator Tool | toolCode | LLM tool: compute CO2e from energy/transport/materials | — | Emissions Calculator Agent (ai_tool) | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Industry Benchmark API Tool | httpRequestTool | LLM tool: fetch industry sustainability benchmarks | — | Emissions Calculator Agent (ai_tool) | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Emissions Output Parser | outputParserStructured | Enforce structured emissions output | — | Emissions Calculator Agent (ai_outputParser) | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Emissions Calculator Agent | langchain.agent | Specialist emissions/scopes/benchmarks analysis | Fetch Supply Chain Data | Merge Agent Results | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| OpenAI Chat Model 3 | lmChatOpenAi | GPT‑4o model provider (circular economy agent) | — | Circular Economy Evaluator Agent (ai_languageModel) | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Circular Economy Output Parser | outputParserStructured | Enforce structured circular economy output | — | Circular Economy Evaluator Agent (ai_outputParser) | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Circular Economy Evaluator Agent | langchain.agent | Circular economy maturity scoring and recommendations | Fetch Supply Chain Data | Merge Agent Results | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| OpenAI Chat Model 4 | lmChatOpenAi | GPT‑4o model provider (logistics agent) | — | Logistics Optimizer Agent (ai_languageModel) | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Logistics Output Parser | outputParserStructured | Enforce structured logistics output | — | Logistics Optimizer Agent (ai_outputParser) | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Logistics Optimizer Agent | langchain.agent | Logistics footprint + route/packaging optimization | Fetch Supply Chain Data | Merge Agent Results | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |
| Merge Agent Results | set | Consolidate 3 agent outputs into one payload | Emissions Calculator Agent; Circular Economy Evaluator Agent; Logistics Optimizer Agent | Supply Chain Sustainability Analyzer | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| OpenAI Chat Model | lmChatOpenAi | GPT‑4o model provider (final analyzer) | — | Supply Chain Sustainability Analyzer (ai_languageModel) | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Structured Output Parser | outputParserStructured | Enforce final unified structured analysis | — | Supply Chain Sustainability Analyzer (ai_outputParser) | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Supply Chain Sustainability Analyzer | langchain.agent | Final risk level, priority, bottlenecks, actions | Merge Agent Results | Route by Risk Level | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Route by Risk Level | switch | Route by `risk_level` to correct handling | Supply Chain Sustainability Analyzer | Format Critical Risk Alert; Format High Risk Alert; Format Medium Risk Report | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Format Critical Risk Alert | set | Build critical email subject/body | Route by Risk Level (Critical) | Send Critical Alert Email | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Send Critical Alert Email | emailSend | Send CRITICAL email alert | Format Critical Risk Alert | Store Analysis Results | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Format High Risk Alert | set | Build high-risk email subject/body | Route by Risk Level (High) | Send High Risk Email | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Send High Risk Email | emailSend | Send HIGH email alert | Format High Risk Alert | Store Analysis Results | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Format Medium Risk Report | set | Create medium-risk summary (no email) | Route by Risk Level (Medium) | Store Analysis Results | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Store Analysis Results | httpRequest | Persist results to storage API | Send Critical Alert Email; Send High Risk Email; Format Medium Risk Report | — | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Sticky Note | stickyNote | Comment block | — | — | ## Prerequisites<br>Active OpenAI API account with sufficient credits, supply chain management system with API access<br>## Use Cases<br>Daily supply chain health monitoring, supplier risk assessment, inventory shortage prediction<br>## Customization<br>Modify agent prompts for industry-specific analysis, adjust risk scoring algorithms<br>## Benefits<br>Provides 360-degree supply chain visibility, enables proactive risk mitigation |
| Sticky Note1 | stickyNote | Comment block | — | — | ## Setup Steps<br>1. Configure Schedule Trigger with desired monitoring frequency<br>2. Set up OpenAI API credentials for all four AI agent nodes<br>3. Configure Fetch Supply Chain Data node with your ERP/SCM system API endpoint<br>4. Customize Enterprise Executor Agent tools with your strategic KPIs<br>5. Update Provider Generator Agent with supplier evaluation criteria<br>6. Configure Circular Economy Agent with sustainability metrics and targets |
| Sticky Note2 | stickyNote | Comment block | — | — | ## How It Works<br>This workflow automates supply chain monitoring and risk management by deploying multiple specialized AI agents to analyze different supply chain dimensions simultaneously. Designed for supply chain managers, procurement teams, and logistics coordinators, it solves the critical challenge of real-time supply chain visibility and proactive risk mitigation across complex global networks. The system triggers on schedule, fetches current supply chain data, then deploys four specialized AI agents—Enterprise Executor for strategic coordination, Provider Generator for supplier assessment, Circular Economy analyzer for sustainability metrics, and Logistics Optimizer for distribution efficiency. Each agent leverages OpenAI models with dedicated tools for calculations and data parsing. Results are merged, analyzed for risk levels (critical, high, normal), and routed to appropriate stakeholders via email with risk-specific formatting and urgency levels. |
| Sticky Note4 | stickyNote | Comment block | — | — | ## Scheduled Data Acquisition and Multi-Agent Deployment<br>**Why:** Triggers automated supply chain data retrieval at configured intervals, then simultaneously deploys four specialized AI agents (Enterprise, Provider, Circular Economy, Logistics) each analyzing distinct supply chain dimensions |
| Sticky Note5 | stickyNote | Comment block | — | — | ## Risk-Based Alert Routing and Notification<br>**Why:** Consolidates multi-agent findings, determines risk severity levels, routes critical alerts to executives via formatted email, high-risk issues to managers, and standard reports to operations teams, ensuring appropriate stakeholders receive actionable intelligence with urgency matching threat level. |
| Sticky Note6 | stickyNote | Comment block | — | — | ## Parallel AI Analysis with Specialized Tools<br>**Why:** Each agent processes supply chain data using OpenAI models equipped with domain-specific tools including calculators for metrics, output parsers for structured results, and benchmark tools, generating insights on supplier performance |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Smart Global Supply Chain Sustainability Analyzer and Optimizer* (or your title).

2. **Add “Schedule Trigger”**
   - Node type: *Schedule Trigger*
   - Configure rule to run daily at **06:00** (or your preferred hour).

3. **Add “Workflow Configuration” (Set node)**
   - Node type: *Set*
   - Add fields:
     - `supplyChainApiUrl` (string) → your ERP/SCM endpoint
     - `storageApiUrl` (string) → your database/storage ingestion endpoint
     - `alertEmailRecipients` (string) → “a@x.com,b@y.com”
     - `criticalThreshold` (number) = 90
     - `highThreshold` (number) = 70
     - `mediumThreshold` (number) = 40
   - Enable **Include Other Fields**.

4. **Connect:** Schedule Trigger → Workflow Configuration

5. **Add “Fetch Supply Chain Data” (HTTP Request)**
   - Node type: *HTTP Request*
   - URL expression: `{{ $('Workflow Configuration').first().json.supplyChainApiUrl }}`
   - Add header `Content-Type: application/json`
   - Configure authentication as required by your API (API key/Bearer/OAuth2).

6. **Connect:** Workflow Configuration → Fetch Supply Chain Data

7. **Create the Emissions analysis chain**
   1) Add **OpenAI Chat Model 2**
      - Node type: *OpenAI Chat Model (LangChain)*
      - Model: `gpt-4o`, Temperature: `0.1`
      - Credential: configure **OpenAI API** in n8n Credentials.
   2) Add **Emissions Output Parser**
      - Node type: *Structured Output Parser*
      - Schema: use the emissions schema from the workflow (fields for totals, scopes, arrays).
   3) Add **Carbon Footprint Calculator Tool**
      - Node type: *Code Tool*
      - Paste the provided JS calculator code.
   4) Add **Industry Benchmark API Tool**
      - Node type: *HTTP Request Tool*
      - Set URL to your benchmark endpoint; configure auth if needed.
   5) Add **Emissions Calculator Agent**
      - Node type: *AI Agent (LangChain)*
      - Prompt type: “Define”
      - System message: emissions specialist instructions (as provided)
      - Text input: `{{ $json }}`
      - Attach:
        - Language model: OpenAI Chat Model 2
        - Output parser: Emissions Output Parser
        - Tools: Carbon Footprint Calculator Tool + Industry Benchmark API Tool
   6) **Connect main data:** Fetch Supply Chain Data → Emissions Calculator Agent

8. **Create the Circular Economy chain**
   1) Add **OpenAI Chat Model 3** (gpt-4o, temp 0.1, OpenAI credential)
   2) Add **Circular Economy Output Parser** (schema with circular score, rates, arrays)
   3) Add **Circular Economy Evaluator Agent**
      - System message: circular economy specialist instructions
      - Text: `{{ $json }}`
      - Attach model + output parser
   4) **Connect:** Fetch Supply Chain Data → Circular Economy Evaluator Agent

9. **Create the Logistics chain**
   1) Add **OpenAI Chat Model 4** (gpt-4o, temp 0.1)
   2) Add **Logistics Output Parser** (schema with logistics score, footprint, modal_split, arrays)
   3) Add **Logistics Optimizer Agent**
      - System message: logistics specialist instructions
      - Text: `{{ $json }}`
      - Attach model + output parser
   4) **Connect:** Fetch Supply Chain Data → Logistics Optimizer Agent

10. **Add “Merge Agent Results” (Set node)**
    - Node type: *Set*
    - Enable **Include Other Fields**
    - Add fields:
      - `emissions_data` (object) = `{{ $('Emissions Calculator Agent').first().json }}`
      - `circular_economy_data` (object) = `{{ $('Circular Economy Evaluator Agent').first().json }}`
      - `logistics_data` (object) = `{{ $('Logistics Optimizer Agent').first().json }}`
      - `combined_analysis_timestamp` (string) = `{{ $now.toISO() }}`
    - **Connect:** each agent → Merge Agent Results  
      (For stricter synchronization, consider using a dedicated Merge node pattern.)

11. **Add final analyzer components**
    1) Add **OpenAI Chat Model** (gpt-4o, temp 0.1, OpenAI credential)
    2) Add **Structured Output Parser** (final schema with risk_level enum, priority_score, etc.)
    3) Add **Supply Chain Sustainability Analyzer** (AI Agent)
       - System message: green economics agent instructions + risk levels
       - Text: `{{ $json }}`
       - Attach model + output parser
    4) **Connect:** Merge Agent Results → Supply Chain Sustainability Analyzer

12. **Add “Route by Risk Level” (Switch)**
    - Node type: *Switch*
    - Rule 1: `{{$json.risk_level}} equals "critical"` → output “Critical”
    - Rule 2: equals `"high"` → “High”
    - Rule 3: equals `"medium"` → “Medium”
    - Fallback: “extra”
    - **Connect:** Supply Chain Sustainability Analyzer → Route by Risk Level

13. **Critical path (format + email)**
    1) Add **Format Critical Risk Alert** (Set)
       - Create fields: `alert_type="CRITICAL"`, plus `subject` and `message_body` using the provided expressions.
    2) Add **Send Critical Alert Email** (Email Send)
       - To: `{{ $('Workflow Configuration').first().json.alertEmailRecipients }}`
       - From: set your sender address
       - Subject/Text from formatted fields
       - Configure SMTP/email credentials in n8n (node will require it).
    3) **Connect:** Route Critical → Format Critical Risk Alert → Send Critical Alert Email

14. **High path (format + email)**
    1) Add **Format High Risk Alert** (Set)
    2) Add **Send High Risk Email** (Email Send) with same recipient expression
    3) **Connect:** Route High → Format High Risk Alert → Send High Risk Email

15. **Medium path (format only)**
    1) Add **Format Medium Risk Report** (Set) to create `summary`
    2) **Connect:** Route Medium → Format Medium Risk Report

16. **Add “Store Analysis Results” (HTTP Request)**
    - Node type: *HTTP Request*
    - Method: POST
    - URL: `{{ $('Workflow Configuration').first().json.storageApiUrl }}`
    - Body: JSON, value `{{ $json }}`
    - Header: Content-Type application/json
    - Add auth as required.
    - **Connect:** Send Critical Alert Email → Store Analysis Results  
      Send High Risk Email → Store Analysis Results  
      Format Medium Risk Report → Store Analysis Results

17. **(Recommended) Handle LOW/fallback**
    - Add a node connected to Switch fallback (“extra”) to store LOW risk results too, otherwise they are dropped.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Active OpenAI API account with sufficient credits, supply chain management system with API access | Prerequisites (Sticky Note) |
| Daily supply chain health monitoring, supplier risk assessment, inventory shortage prediction | Use cases (Sticky Note) |
| Modify agent prompts for industry-specific analysis, adjust risk scoring algorithms | Customization (Sticky Note) |
| Provides 360-degree supply chain visibility, enables proactive risk mitigation | Benefits (Sticky Note) |
| Setup Steps: configure schedule, OpenAI credentials for AI nodes, API endpoint in Fetch node, customize agent tools/criteria/metrics | Setup Steps (Sticky Note1) |
| How it works: scheduled fetch → multiple AI agents → merged → risk routing → email formatting by urgency | How It Works (Sticky Note2) |

**Notable consistency gaps to be aware of (implementation-level):**
- Workflow config thresholds are not used in routing/scoring; risk bands are embedded in the analyzer prompt.
- Switch fallback (“low” / unexpected) is not connected → results can be silently lost.
- Merge logic uses a Set node and `.first()` across parallel branches; for strict correlation, use explicit merge/synchronization patterns.