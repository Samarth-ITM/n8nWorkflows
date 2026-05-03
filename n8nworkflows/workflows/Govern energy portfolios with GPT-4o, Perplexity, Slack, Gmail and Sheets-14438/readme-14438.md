Govern energy portfolios with GPT-4o, Perplexity, Slack, Gmail and Sheets

https://n8nworkflows.xyz/workflows/govern-energy-portfolios-with-gpt-4o--perplexity--slack--gmail-and-sheets-14438


# Govern energy portfolios with GPT-4o, Perplexity, Slack, Gmail and Sheets

# 1. Workflow Overview

This workflow automates energy portfolio governance by collecting three external data streams, combining them into a unified energy context, analyzing them with a GPT-4o-based orchestration agent plus specialist AI tools, and distributing the resulting sustainability KPIs through Slack, Gmail, and Google Sheets.

Typical use cases include:

- Monitoring renewable versus non-renewable energy usage
- Forecasting renewable availability from weather and generation signals
- Checking sustainability and policy compliance
- Producing stakeholder-facing alerts and dashboards
- Logging KPI snapshots for historical tracking

## 1.1 Scheduled Input Reception

The workflow starts on a recurring schedule and launches three parallel API calls to gather:

- Weather data
- Energy demand data
- Renewable generation data

## 1.2 Energy Data Consolidation

The three inbound datasets are merged into a single combined item so downstream AI analysis can use a synchronized snapshot.

## 1.3 AI Governance Orchestration

A central **Energy Governance Agent** receives the merged dataset and uses:

- A GPT-4o chat model as its reasoning engine
- Three specialist agent tools:
  - Renewable Forecasting Agent
  - Weather Analysis Agent
  - Policy Compliance Agent
- Four additional tools:
  - Calculator
  - Custom code optimization tool
  - External HTTP API tool
  - Perplexity research tool
- A structured output parser to force a KPI-shaped result

## 1.4 Reporting and Persistence

Once the governance agent produces structured output, the workflow fans out into three delivery channels:

- Slack sustainability alert
- Gmail HTML dashboard
- Google Sheets KPI append

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

### Overview

This block triggers the workflow on a recurring basis and initiates three parallel fetches for the required upstream energy inputs. It is the workflow’s only execution entry point.

### Nodes Involved

- Energy Analysis Schedule
- Fetch Weather Data
- Fetch Energy Demand
- Fetch Renewable Generation

### Node Details

#### Energy Analysis Schedule
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — scheduled workflow trigger
- **Configuration choices:** Configured to run at an hourly interval.
- **Key expressions or variables:** None
- **Input and output connections:** No input; outputs to:
  - Fetch Weather Data
  - Fetch Energy Demand
  - Fetch Renewable Generation
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**
  - Misconfigured interval may trigger too frequently or not frequently enough
  - Workflow must be activated for schedule execution
- **Sub-workflow reference:** None

#### Fetch Weather Data
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches weather conditions relevant to renewable forecasting
- **Configuration choices:** Simple HTTP request using a placeholder weather API endpoint; no advanced options configured
- **Key expressions or variables:**
  - URL is currently a placeholder: `weather_api_endpoint`
- **Input and output connections:** Input from Energy Analysis Schedule; output to Combine Energy Data input 0
- **Version-specific requirements:** Type version `4.4`
- **Edge cases / failures:**
  - Placeholder URL not replaced
  - API auth may be required but is not configured
  - Non-JSON or unexpected response shape may impair downstream analysis
  - Timeouts, rate limits, DNS failures
- **Sub-workflow reference:** None

#### Fetch Energy Demand
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches current or forecast energy demand data
- **Configuration choices:** Simple HTTP request using a placeholder energy demand endpoint
- **Key expressions or variables:**
  - URL placeholder: `energy_demand_api_endpoint`
- **Input and output connections:** Input from Energy Analysis Schedule; output to Combine Energy Data input 1
- **Version-specific requirements:** Type version `4.4`
- **Edge cases / failures:**
  - Placeholder URL not replaced
  - Auth requirements not configured
  - Demand payload may not match expected semantics
  - Rate limiting or service unavailability
- **Sub-workflow reference:** None

#### Fetch Renewable Generation
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches renewable generation capacity or current generation data
- **Configuration choices:** Simple HTTP request using a placeholder renewable generation API endpoint
- **Key expressions or variables:**
  - URL placeholder: `renewable_generation_api_endpoint`
- **Input and output connections:** Input from Energy Analysis Schedule; output to Combine Energy Data input 2
- **Version-specific requirements:** Type version `4.4`
- **Edge cases / failures:**
  - Placeholder URL not replaced
  - Missing credentials if source requires authentication
  - Unstable response schema or null fields
  - Service timeout
- **Sub-workflow reference:** None

---

## 2.2 Energy Data Consolidation

### Overview

This block merges the three parallel API responses into a single item by position. The downstream AI block depends on all three responses being present in a synchronized execution.

### Nodes Involved

- Combine Energy Data

### Node Details

#### Combine Energy Data
- **Type / role:** `n8n-nodes-base.merge` — combines weather, demand, and generation responses into one composite dataset
- **Configuration choices:**
  - Mode: `combine`
  - Combine method: `combineByPosition`
  - Number of inputs: `3`
- **Key expressions or variables:** None
- **Input and output connections:**
  - Input 0: Fetch Weather Data
  - Input 1: Fetch Energy Demand
  - Input 2: Fetch Renewable Generation
  - Output: Energy Governance Agent
- **Version-specific requirements:** Type version `3.2`
- **Edge cases / failures:**
  - If one branch returns no item, combine-by-position can produce incomplete or no merged output
  - If APIs return multiple items with mismatched counts, records may pair incorrectly by index
  - Large payloads may increase token consumption for the AI block
- **Sub-workflow reference:** None

---

## 2.3 AI Governance Orchestration

### Overview

This is the core analytical block. A central governance agent receives the merged energy snapshot, uses GPT-4o for reasoning, calls specialized tools and specialist agents as needed, and emits a structured KPI object.

### Nodes Involved

- Energy Governance Agent
- Governance Model
- Structured KPI Output
- Renewable Forecasting Agent
- Forecasting Model
- Weather Analysis Agent
- Weather Model
- Policy Compliance Agent
- Policy Model
- Energy Calculator
- Optimization Algorithm Tool
- External Data API Tool
- Sustainability Research Tool

### Node Details

#### Energy Governance Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — central orchestration agent
- **Configuration choices:**
  - Prompt type: defined manually
  - User text input: entire merged JSON via `={{ $json }}`
  - System prompt defines the role as an energy governance orchestrator focused on allocation, compliance, and sustainability reporting
  - Output parser enabled
- **Key expressions or variables:**
  - `={{ $json }}`
- **Input and output connections:**
  - Main input: Combine Energy Data
  - AI language model input: Governance Model
  - AI tools:
    - Renewable Forecasting Agent
    - Weather Analysis Agent
    - Policy Compliance Agent
    - Energy Calculator
    - Optimization Algorithm Tool
    - External Data API Tool
    - Sustainability Research Tool
  - AI output parser: Structured KPI Output
  - Main outputs:
    - Send Sustainability Alert
    - Email Performance Dashboard
    - Prepare Sheet Data
- **Version-specific requirements:** Type version `3.1`
- **Edge cases / failures:**
  - Model may fail to produce output conforming to schema
  - Oversized merged JSON can exceed model context
  - Tool calls may fail individually and degrade final output
  - Ambiguous or sparse API payloads can lead to hallucinated assumptions
- **Sub-workflow reference:** None

#### Governance Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model backend for the governance agent
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for relatively deterministic outputs
- **Key expressions or variables:** None
- **Input and output connections:** AI language model connection into Energy Governance Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**
  - OpenAI credential failures
  - Model availability changes
  - Rate limits / token quotas
- **Sub-workflow reference:** None

#### Structured KPI Output
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured JSON output
- **Configuration choices:**
  - Manual JSON schema with fields:
    - renewablePercentage
    - nonRenewablePercentage
    - energyEfficiencyScore
    - carbonFootprintKg
    - policyComplianceStatus
    - recommendations
    - alerts
    - forecastedDemandKwh
    - availableRenewableKwh
    - optimizationOpportunities
- **Key expressions or variables:** Manual schema only
- **Input and output connections:** AI output parser connection into Energy Governance Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**
  - Model may omit required values or output wrong types
  - Arrays may come back empty or malformed
  - Numeric fields may be inferred from non-numeric source text and fail parsing
- **Sub-workflow reference:** None

#### Renewable Forecasting Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist forecasting tool callable by the governance agent
- **Configuration choices:**
  - Prompt text uses AI-provided task variable:
    - `={{ $fromAI('forecasting_task', 'The forecasting analysis task to perform') }}`
  - System message specializes the agent in demand trends, weather patterns, and renewable capacity forecasting
  - Tool description clarifies purpose for the orchestrator
- **Key expressions or variables:**
  - `$fromAI('forecasting_task', ...)`
- **Input and output connections:**
  - AI language model input from Forecasting Model
  - AI tool output into Energy Governance Agent
- **Version-specific requirements:** Type version `3`
- **Edge cases / failures:**
  - If the parent agent sends a vague task, specialist output may be weak
  - Same OpenAI credential risks as other model nodes
- **Sub-workflow reference:** None

#### Forecasting Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — model for Renewable Forecasting Agent
- **Configuration choices:** `gpt-4o`, temperature `0.2`
- **Key expressions or variables:** None
- **Input and output connections:** AI language model connection into Renewable Forecasting Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:** Credential errors, rate limits, model issues
- **Sub-workflow reference:** None

#### Weather Analysis Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist weather impact analysis tool
- **Configuration choices:**
  - Task input:
    - `={{ $fromAI('weather_task', 'The weather analysis task to perform') }}`
  - System prompt focused on solar radiation, wind speed, cloud cover, temperature, precipitation, and impact on generation
- **Key expressions or variables:**
  - `$fromAI('weather_task', ...)`
- **Input and output connections:**
  - AI language model from Weather Model
  - AI tool into Energy Governance Agent
- **Version-specific requirements:** Type version `3`
- **Edge cases / failures:**
  - Poorly structured weather source data may limit usefulness
  - General LLM/tool-call instability
- **Sub-workflow reference:** None

#### Weather Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — model for Weather Analysis Agent
- **Configuration choices:** `gpt-4o`, temperature `0.2`
- **Key expressions or variables:** None
- **Input and output connections:** AI language model connection into Weather Analysis Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:** Credential or quota issues
- **Sub-workflow reference:** None

#### Policy Compliance Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist compliance-checking tool
- **Configuration choices:**
  - Task input:
    - `={{ $fromAI('compliance_task', 'The policy compliance task to perform') }}`
  - System prompt focused on sustainability rules, environmental regulations, carbon targets, and renewable mandates
- **Key expressions or variables:**
  - `$fromAI('compliance_task', ...)`
- **Input and output connections:**
  - AI language model from Policy Model
  - AI tool into Energy Governance Agent
- **Version-specific requirements:** Type version `3`
- **Edge cases / failures:**
  - If actual policy thresholds are not present in source data, the model may infer generic compliance
  - Better results may require explicit thresholds or policy documents
- **Sub-workflow reference:** None

#### Policy Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — model for Policy Compliance Agent
- **Configuration choices:** `gpt-4o`, temperature `0.2`
- **Key expressions or variables:** None
- **Input and output connections:** AI language model connection into Policy Compliance Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:** Credential, quota, or provider-side availability issues
- **Sub-workflow reference:** None

#### Energy Calculator
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCalculator` — arithmetic tool for exact calculations
- **Configuration choices:** Default calculator tool
- **Key expressions or variables:** None directly exposed
- **Input and output connections:** AI tool into Energy Governance Agent
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:**
  - Limited to calculator-style operations
  - Parent agent must formulate valid calculation requests
- **Sub-workflow reference:** None

#### Optimization Algorithm Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCode` — custom JavaScript tool for energy optimization calculations
- **Configuration choices:**
  - JS code accepts AI-supplied numbers:
    - `demand`
    - `renewable`
  - Computes:
    - `efficiency = Math.min(renewable / demand * 100, 100)`
    - `surplus = Math.max(renewable - demand, 0)`
  - Tool description states it supports optimization/statistical modeling
- **Key expressions or variables:**
  - `$fromAI('demand', 'Energy demand in kWh', 'number')`
  - `$fromAI('renewable', 'Available renewable energy in kWh', 'number')`
- **Input and output connections:** AI tool into Energy Governance Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**
  - Division by zero if `demand = 0` leads to `Infinity`
  - Missing or non-numeric AI-supplied values may break execution or yield invalid outputs
  - The description suggests broader capability than the implemented code actually provides
- **Sub-workflow reference:** None

#### External Data API Tool
- **Type / role:** `n8n-nodes-base.httpRequestTool` — AI-callable HTTP tool for ad hoc external enrichment
- **Configuration choices:**
  - URL is supplied by AI:
    - `={{ $fromAI('url', 'API endpoint URL', 'string') }}`
  - Method is AI-supplied with default GET:
    - `={{ $fromAI('method', 'HTTP method (GET, POST, etc.)', 'string', 'GET') }}`
  - Tool description indicates use for grid data, sensors, or sustainability metrics
- **Key expressions or variables:**
  - `$fromAI('url', ...)`
  - `$fromAI('method', ...)`
- **Input and output connections:** AI tool into Energy Governance Agent
- **Version-specific requirements:** Type version `4.4`
- **Edge cases / failures:**
  - Risky if unrestricted AI-generated URLs are allowed
  - External APIs may need auth headers not configured here
  - Could hit unavailable or slow endpoints
  - Security review recommended before production use
- **Sub-workflow reference:** None

#### Sustainability Research Tool
- **Type / role:** `n8n-nodes-base.perplexityTool` — research tool for policy and best-practice lookup
- **Configuration choices:**
  - Query is AI-provided:
    - `={{ $fromAI('sustainability_research', 'Research query about renewable energy policies, sustainability regulations, or industry best practices') }}`
  - Manual tool description clarifies intended use
- **Key expressions or variables:**
  - `$fromAI('sustainability_research', ...)`
- **Input and output connections:** AI tool into Energy Governance Agent
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:**
  - Perplexity credential/auth issues
  - Research answers may be current but not jurisdiction-specific
  - Network or provider quotas may interrupt execution
- **Sub-workflow reference:** None

---

## 2.4 Reporting and Persistence

### Overview

This block takes the governance agent’s structured output and distributes it to stakeholders via Slack and Gmail while also transforming selected fields into a tabular format for storage in Google Sheets.

### Nodes Involved

- Send Sustainability Alert
- Email Performance Dashboard
- Prepare Sheet Data
- Store Energy KPIs

### Node Details

#### Send Sustainability Alert
- **Type / role:** `n8n-nodes-base.slack` — posts KPI summary and alerts to a Slack channel
- **Configuration choices:**
  - OAuth2 authentication
  - Channel selected by ID placeholder
  - Message template includes renewable percentage, efficiency score, carbon footprint, compliance status, recommendations, and alerts
- **Key expressions or variables:**
  - `{{ $json.output.renewablePercentage }}`
  - `{{ $json.output.energyEfficiencyScore }}`
  - `{{ $json.output.carbonFootprintKg }}`
  - `{{ $json.output.policyComplianceStatus }}`
  - `{{ $json.output.recommendations.join('\n• ') }}`
  - `{{ $json.output.alerts.join('\n⚠️ ') }}`
- **Input and output connections:** Input from Energy Governance Agent; no downstream node
- **Version-specific requirements:** Type version `2.4`
- **Edge cases / failures:**
  - Placeholder channel ID must be replaced
  - If `recommendations` or `alerts` are null instead of arrays, `.join()` fails
  - Slack OAuth scopes must permit posting to the target channel
- **Sub-workflow reference:** None

#### Email Performance Dashboard
- **Type / role:** `n8n-nodes-base.gmail` — sends an HTML dashboard email
- **Configuration choices:**
  - Sends to placeholder sustainability team email
  - HTML body uses data from `$('Energy Governance Agent').item.json.output`
  - Attribution disabled
  - Subject includes current date using `$now.format('yyyy-MM-dd')`
- **Key expressions or variables:**
  - `{{ $('Energy Governance Agent').item.json.output.renewablePercentage }}`
  - `{{ $('Energy Governance Agent').item.json.output.nonRenewablePercentage }}`
  - `{{ $('Energy Governance Agent').item.json.output.energyEfficiencyScore }}`
  - `{{ $('Energy Governance Agent').item.json.output.carbonFootprintKg }}`
  - `{{ $('Energy Governance Agent').item.json.output.forecastedDemandKwh }}`
  - `{{ $('Energy Governance Agent').item.json.output.availableRenewableKwh }}`
  - `{{ $('Energy Governance Agent').item.json.output.policyComplianceStatus }}`
  - `{{ $('Energy Governance Agent').item.json.output.optimizationOpportunities.map(...) }}`
  - `{{ $('Energy Governance Agent').item.json.output.recommendations.map(...) }}`
  - Subject: `{{ $now.format('yyyy-MM-dd') }}`
- **Input and output connections:** Input from Energy Governance Agent; no downstream node
- **Version-specific requirements:** Type version `2.2`
- **Edge cases / failures:**
  - Placeholder email must be replaced
  - `.map()` on null or missing arrays will fail
  - Gmail OAuth must allow sending
  - HTML rendering may be degraded if values contain unexpected markup
- **Sub-workflow reference:** None

#### Prepare Sheet Data
- **Type / role:** `n8n-nodes-base.set` — maps structured AI output into flat columns for storage
- **Configuration choices:**
  - Creates fields:
    - timestamp
    - renewablePercentage
    - nonRenewablePercentage
    - energyEfficiencyScore
    - carbonFootprintKg
    - policyComplianceStatus
    - forecastedDemandKwh
    - availableRenewableKwh
- **Key expressions or variables:**
  - `={{ $now.toISO() }}`
  - `={{ $json.output.renewablePercentage }}`
  - `={{ $json.output.nonRenewablePercentage }}`
  - `={{ $json.output.energyEfficiencyScore }}`
  - `={{ $json.output.carbonFootprintKg }}`
  - `={{ $json.output.policyComplianceStatus }}`
  - `={{ $json.output.forecastedDemandKwh }}`
  - `={{ $json.output.availableRenewableKwh }}`
- **Input and output connections:** Input from Energy Governance Agent; output to Store Energy KPIs
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:**
  - Numeric fields may fail if parser output contains strings or nulls
  - Some useful arrays such as recommendations are not persisted
- **Sub-workflow reference:** None

#### Store Energy KPIs
- **Type / role:** `n8n-nodes-base.googleSheets` — appends KPI rows to a Google Sheet
- **Configuration choices:**
  - Operation: append
  - Document ID and sheet name are still unset placeholders/empty selections
- **Key expressions or variables:** None
- **Input and output connections:** Input from Prepare Sheet Data; no downstream node
- **Version-specific requirements:** Type version `4.7`
- **Edge cases / failures:**
  - Sheet document ID and tab name must be configured
  - Sheet columns should match the prepared field names
  - OAuth permissions must include edit access
  - Append may fail if sheet structure changes
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Energy Analysis Schedule | n8n-nodes-base.scheduleTrigger | Hourly workflow trigger |  | Fetch Weather Data, Fetch Energy Demand, Fetch Renewable Generation | ## How It Works<br>This workflow automates energy portfolio governance for energy managers, sustainability teams, and policy compliance officers. It eliminates the manual effort of aggregating multi-source energy data, applying forecasting and optimisation logic, and distributing performance outcomes to stakeholders. Three scheduled API feeds, weather data, energy demand, and renewable generation, are combined into a unified dataset. The Energy Governance Agent, backed by shared memory and a governance model, coordinates three specialist agents: a Renewable Forecasting Agent (predictive generation modelling), a Weather Analysis Agent (climate-adjusted demand assessment), and a Policy Compliance Agent (regulatory alignment checking). Five tools support the orchestration: an Energy Calculator, Optimisation Algorithm Tool, External Data API Tool, Sustainability Research Tool, and Structured KPI Output parser. Results are distributed across three outputs, a Slack sustainability alert, a Gmail performance dashboard email, and a Google Sheets KPI store, giving stakeholders immediate, channel-appropriate visibility into energy governance outcomes. |
| Fetch Weather Data | n8n-nodes-base.httpRequest | Retrieve weather data | Energy Analysis Schedule | Combine Energy Data | ## How It Works<br>This workflow automates energy portfolio governance for energy managers, sustainability teams, and policy compliance officers. It eliminates the manual effort of aggregating multi-source energy data, applying forecasting and optimisation logic, and distributing performance outcomes to stakeholders. Three scheduled API feeds, weather data, energy demand, and renewable generation, are combined into a unified dataset. The Energy Governance Agent, backed by shared memory and a governance model, coordinates three specialist agents: a Renewable Forecasting Agent (predictive generation modelling), a Weather Analysis Agent (climate-adjusted demand assessment), and a Policy Compliance Agent (regulatory alignment checking). Five tools support the orchestration: an Energy Calculator, Optimisation Algorithm Tool, External Data API Tool, Sustainability Research Tool, and Structured KPI Output parser. Results are distributed across three outputs, a Slack sustainability alert, a Gmail performance dashboard email, and a Google Sheets KPI store, giving stakeholders immediate, channel-appropriate visibility into energy governance outcomes. |
| Fetch Energy Demand | n8n-nodes-base.httpRequest | Retrieve energy demand data | Energy Analysis Schedule | Combine Energy Data | ## How It Works<br>This workflow automates energy portfolio governance for energy managers, sustainability teams, and policy compliance officers. It eliminates the manual effort of aggregating multi-source energy data, applying forecasting and optimisation logic, and distributing performance outcomes to stakeholders. Three scheduled API feeds, weather data, energy demand, and renewable generation, are combined into a unified dataset. The Energy Governance Agent, backed by shared memory and a governance model, coordinates three specialist agents: a Renewable Forecasting Agent (predictive generation modelling), a Weather Analysis Agent (climate-adjusted demand assessment), and a Policy Compliance Agent (regulatory alignment checking). Five tools support the orchestration: an Energy Calculator, Optimisation Algorithm Tool, External Data API Tool, Sustainability Research Tool, and Structured KPI Output parser. Results are distributed across three outputs, a Slack sustainability alert, a Gmail performance dashboard email, and a Google Sheets KPI store, giving stakeholders immediate, channel-appropriate visibility into energy governance outcomes. |
| Fetch Renewable Generation | n8n-nodes-base.httpRequest | Retrieve renewable generation data | Energy Analysis Schedule | Combine Energy Data | ## How It Works<br>This workflow automates energy portfolio governance for energy managers, sustainability teams, and policy compliance officers. It eliminates the manual effort of aggregating multi-source energy data, applying forecasting and optimisation logic, and distributing performance outcomes to stakeholders. Three scheduled API feeds, weather data, energy demand, and renewable generation, are combined into a unified dataset. The Energy Governance Agent, backed by shared memory and a governance model, coordinates three specialist agents: a Renewable Forecasting Agent (predictive generation modelling), a Weather Analysis Agent (climate-adjusted demand assessment), and a Policy Compliance Agent (regulatory alignment checking). Five tools support the orchestration: an Energy Calculator, Optimisation Algorithm Tool, External Data API Tool, Sustainability Research Tool, and Structured KPI Output parser. Results are distributed across three outputs, a Slack sustainability alert, a Gmail performance dashboard email, and a Google Sheets KPI store, giving stakeholders immediate, channel-appropriate visibility into energy governance outcomes. |
| Combine Energy Data | n8n-nodes-base.merge | Combine three data feeds into one item | Fetch Weather Data, Fetch Energy Demand, Fetch Renewable Generation | Energy Governance Agent | ## Energy Governance Agent & Specialist Agents<br>**Why** — Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. |
| Energy Governance Agent | @n8n/n8n-nodes-langchain.agent | Central AI orchestrator for governance decisions | Combine Energy Data | Prepare Sheet Data, Send Sustainability Alert, Email Performance Dashboard | ## Energy Governance Agent & Specialist Agents<br>**Why** — Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. |
| Governance Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main LLM for orchestration agent |  | Energy Governance Agent | ## Setup Steps<br>1. Import workflow; configure the Energy Analysis Schedule trigger interval.<br>2. Set API endpoint URLs for Fetch Weather Data, Fetch Energy Demand, and Fetch Renewable Generation nodes.<br>3. Add AI model credentials to the Agents.<br>4. Connect Slack credentials to the Send Sustainability Alert node.<br>5. Link Gmail credentials to the Email Performance Dashboard node.<br>6. Link Google Sheets credentials; set the sheet ID for the Energy KPIs tab.<br><br>## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Slack workspace with bot credentials<br>- Gmail account with OAuth credentials<br>- Google Sheets with Energy KPIs tab pre-created<br>## Use Cases<br>- Energy managers automating renewable generation forecasting against real-time demand data<br>## Customisation<br>- Swap the Optimisation Algorithm Tool parameters to target carbon intensity, cost, or grid stability objectives<br>## Benefits<br>- Parallel triple-source ingestion ensures governance decisions are based on complete, synchronised energy data |
| Structured KPI Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured KPI schema |  | Energy Governance Agent | ## Energy Calculator, Optimisation & External Data Tools<br>**Why** — Applies quantitative calculation, algorithmic optimisation, and live external data enrichment to produce governance-grade energy recommendations. |
| Renewable Forecasting Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist AI tool for renewable forecasting |  | Energy Governance Agent | ## Energy Governance Agent & Specialist Agents<br>**Why** — Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. |
| Forecasting Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for forecasting specialist |  | Renewable Forecasting Agent | ## Energy Governance Agent & Specialist Agents<br>**Why** — Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. |
| Weather Analysis Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist AI tool for weather impact analysis |  | Energy Governance Agent | ## Energy Governance Agent & Specialist Agents<br>**Why** — Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. |
| Weather Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for weather specialist |  | Weather Analysis Agent | ## Energy Governance Agent & Specialist Agents<br>**Why** — Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. |
| Policy Compliance Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist AI tool for policy/regulatory analysis |  | Energy Governance Agent | ## Energy Governance Agent & Specialist Agents<br>**Why** — Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. |
| Policy Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for policy specialist |  | Policy Compliance Agent | ## Energy Governance Agent & Specialist Agents<br>**Why** — Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. |
| Energy Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Exact arithmetic support tool |  | Energy Governance Agent | ## Energy Calculator, Optimisation & External Data Tools<br>**Why** — Applies quantitative calculation, algorithmic optimisation, and live external data enrichment to produce governance-grade energy recommendations. |
| Optimization Algorithm Tool | @n8n/n8n-nodes-langchain.toolCode | Custom optimization calculation tool |  | Energy Governance Agent | ## Energy Calculator, Optimisation & External Data Tools<br>**Why** — Applies quantitative calculation, algorithmic optimisation, and live external data enrichment to produce governance-grade energy recommendations. |
| External Data API Tool | n8n-nodes-base.httpRequestTool | AI-callable external enrichment API tool |  | Energy Governance Agent | ## Energy Calculator, Optimisation & External Data Tools<br>**Why** — Applies quantitative calculation, algorithmic optimisation, and live external data enrichment to produce governance-grade energy recommendations. |
| Sustainability Research Tool | n8n-nodes-base.perplexityTool | Policy and best-practice research tool |  | Energy Governance Agent | ## Energy Calculator, Optimisation & External Data Tools<br>**Why** — Applies quantitative calculation, algorithmic optimisation, and live external data enrichment to produce governance-grade energy recommendations. |
| Send Sustainability Alert | n8n-nodes-base.slack | Post KPI alert to Slack | Energy Governance Agent |  | ## Send Sustainability Alert, Email Dashboard & Store Energy KPIs<br>**Why** — Distributes governance outcomes via Slack alerts, Gmail dashboards, and Google Sheets KPI logs to keep all stakeholders informed without manual reporting. |
| Email Performance Dashboard | n8n-nodes-base.gmail | Send HTML dashboard email | Energy Governance Agent |  | ## Send Sustainability Alert, Email Dashboard & Store Energy KPIs<br>**Why** — Distributes governance outcomes via Slack alerts, Gmail dashboards, and Google Sheets KPI logs to keep all stakeholders informed without manual reporting. |
| Prepare Sheet Data | n8n-nodes-base.set | Flatten structured output into sheet columns | Energy Governance Agent | Store Energy KPIs | ## Send Sustainability Alert, Email Dashboard & Store Energy KPIs<br>**Why** — Distributes governance outcomes via Slack alerts, Gmail dashboards, and Google Sheets KPI logs to keep all stakeholders informed without manual reporting. |
| Store Energy KPIs | n8n-nodes-base.googleSheets | Append KPI row to Google Sheets | Prepare Sheet Data |  | ## Send Sustainability Alert, Email Dashboard & Store Energy KPIs<br>**Why** — Distributes governance outcomes via Slack alerts, Gmail dashboards, and Google Sheets KPI logs to keep all stakeholders informed without manual reporting. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for setup |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for workflow behavior |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for prerequisites and customization |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note for reporting outputs |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note for tools layer |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note for agent coordination |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:
   - `AI-powered energy governance with renewable forecasting and policy compliance`

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Energy Analysis Schedule`
   - Set the trigger rule to run every hour.
   - This is the only entry point.

3. **Add the weather HTTP request**
   - Node type: **HTTP Request**
   - Name: `Fetch Weather Data`
   - Method: GET unless your weather API requires otherwise
   - URL: your weather API endpoint
   - If required, add authentication, headers, or query parameters
   - Connect `Energy Analysis Schedule -> Fetch Weather Data`

4. **Add the energy demand HTTP request**
   - Node type: **HTTP Request**
   - Name: `Fetch Energy Demand`
   - Method: GET unless the API requires another method
   - URL: your energy demand API endpoint
   - Add auth if needed
   - Connect `Energy Analysis Schedule -> Fetch Energy Demand`

5. **Add the renewable generation HTTP request**
   - Node type: **HTTP Request**
   - Name: `Fetch Renewable Generation`
   - Method: GET unless needed otherwise
   - URL: your renewable generation API endpoint
   - Add auth if needed
   - Connect `Energy Analysis Schedule -> Fetch Renewable Generation`

6. **Add a Merge node**
   - Node type: **Merge**
   - Name: `Combine Energy Data`
   - Set:
     - Mode: `Combine`
     - Combine By: `Position`
     - Number of inputs: `3`
   - Connect:
     - `Fetch Weather Data -> Combine Energy Data` input 1
     - `Fetch Energy Demand -> Combine Energy Data` input 2
     - `Fetch Renewable Generation -> Combine Energy Data` input 3

7. **Add the main orchestration agent**
   - Node type: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Name: `Energy Governance Agent`
   - Set prompt mode to define manually
   - Set the user text input to:
     - `={{ $json }}`
   - Add this system message:
     - “You are the Energy Governance Agent, responsible for orchestrating smart energy allocation, policy compliance, and sustainability reporting. Analyze the combined energy data (weather, demand, renewable generation) and coordinate with specialized sub-agents to optimize renewable energy usage, reduce non-renewable reliance, ensure policy compliance, and generate comprehensive sustainability insights. Provide strategic recommendations for energy allocation and identify compliance issues.”
   - Enable structured output parsing
   - Connect `Combine Energy Data -> Energy Governance Agent`

8. **Add the main OpenAI chat model**
   - Node type: **OpenAI Chat Model**
   - Name: `Governance Model`
   - Choose model: `gpt-4o`
   - Set temperature to `0.2`
   - Configure OpenAI credentials
   - Connect this model to `Energy Governance Agent` via the AI language model connection

9. **Add the structured output parser**
   - Node type: **Structured Output Parser**
   - Name: `Structured KPI Output`
   - Choose manual schema
   - Define a schema with:
     - `renewablePercentage` number
     - `nonRenewablePercentage` number
     - `energyEfficiencyScore` number
     - `carbonFootprintKg` number
     - `policyComplianceStatus` string
     - `recommendations` array of strings
     - `alerts` array of strings
     - `forecastedDemandKwh` number
     - `availableRenewableKwh` number
     - `optimizationOpportunities` array of strings
   - Connect this parser to `Energy Governance Agent` as the AI output parser

10. **Add the Renewable Forecasting specialist**
    - Node type: **AI Agent Tool**
    - Name: `Renewable Forecasting Agent`
    - Set text input to:
      - `={{ $fromAI('forecasting_task', 'The forecasting analysis task to perform') }}`
    - Set tool description to explain that it forecasts renewable availability and allocation strategies
    - Set system message to the renewable forecasting specialist prompt from the workflow
    - Connect later to its own model and then to the main governance agent as an AI tool

11. **Add the forecasting model**
    - Node type: **OpenAI Chat Model**
    - Name: `Forecasting Model`
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Reuse the same OpenAI credential or another valid one
    - Connect `Forecasting Model -> Renewable Forecasting Agent` with AI language model connection
    - Connect `Renewable Forecasting Agent -> Energy Governance Agent` with AI tool connection

12. **Add the Weather Analysis specialist**
    - Node type: **AI Agent Tool**
    - Name: `Weather Analysis Agent`
    - Text:
      - `={{ $fromAI('weather_task', 'The weather analysis task to perform') }}`
    - Add the weather-focused system prompt
    - Add a tool description explaining it analyzes renewable generation impact from weather
    - This will be callable by the governance agent

13. **Add the weather model**
    - Node type: **OpenAI Chat Model**
    - Name: `Weather Model`
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Connect `Weather Model -> Weather Analysis Agent` as AI language model
    - Connect `Weather Analysis Agent -> Energy Governance Agent` as AI tool

14. **Add the Policy Compliance specialist**
    - Node type: **AI Agent Tool**
    - Name: `Policy Compliance Agent`
    - Text:
      - `={{ $fromAI('compliance_task', 'The policy compliance task to perform') }}`
    - Add the compliance-focused system prompt
    - Add a tool description explaining regulatory and sustainability evaluation
    - This will be callable by the governance agent

15. **Add the policy model**
    - Node type: **OpenAI Chat Model**
    - Name: `Policy Model`
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Connect `Policy Model -> Policy Compliance Agent` as AI language model
    - Connect `Policy Compliance Agent -> Energy Governance Agent` as AI tool

16. **Add the calculator tool**
    - Node type: **Calculator Tool**
    - Name: `Energy Calculator`
    - Default configuration is sufficient
    - Connect it to `Energy Governance Agent` as an AI tool

17. **Add the custom optimization code tool**
    - Node type: **Code Tool**
    - Name: `Optimization Algorithm Tool`
    - Description: indicate it executes optimization logic
    - Paste this JavaScript logic conceptually:
      - Accept AI inputs `demand` and `renewable` as numbers
      - Compute efficiency as renewable divided by demand times 100, capped at 100
      - Compute surplus as renewable minus demand, floored at 0
    - In n8n, use:
      - `$fromAI('demand', 'Energy demand in kWh', 'number')`
      - `$fromAI('renewable', 'Available renewable energy in kWh', 'number')`
    - Connect it to `Energy Governance Agent` as an AI tool
    - Recommended improvement: add a guard for `demand === 0`

18. **Add the external HTTP tool**
    - Node type: **HTTP Request Tool**
    - Name: `External Data API Tool`
    - URL:
      - `={{ $fromAI('url', 'API endpoint URL', 'string') }}`
    - Method:
      - `={{ $fromAI('method', 'HTTP method (GET, POST, etc.)', 'string', 'GET') }}`
    - Add a tool description indicating external enrichment
    - Connect it to `Energy Governance Agent` as an AI tool
    - For production, consider restricting domains or predefining safe endpoints

19. **Add the Perplexity research tool**
    - Node type: **Perplexity Tool**
    - Name: `Sustainability Research Tool`
    - Message content:
      - `={{ $fromAI('sustainability_research', 'Research query about renewable energy policies, sustainability regulations, or industry best practices') }}`
    - Add Perplexity API credentials
    - Set a manual description explaining its policy and best-practice lookup role
    - Connect it to `Energy Governance Agent` as an AI tool

20. **Add the Slack output**
    - Node type: **Slack**
    - Name: `Send Sustainability Alert`
    - Authentication: OAuth2
    - Operation: send message to channel
    - Select channel by ID
    - Replace the placeholder sustainability channel with your actual channel
    - Use a message template containing:
      - renewable percentage
      - energy efficiency score
      - carbon footprint
      - policy compliance status
      - recommendations list
      - alerts list
    - Connect `Energy Governance Agent -> Send Sustainability Alert`

21. **Add the Gmail output**
    - Node type: **Gmail**
    - Name: `Email Performance Dashboard`
    - Operation: send email
    - To: sustainability team email address
    - Subject:
      - `Environmental Performance Dashboard - {{ $now.format('yyyy-MM-dd') }}`
    - Body: HTML dashboard using fields from `$('Energy Governance Agent').item.json.output`
    - Disable attribution if desired
    - Configure Gmail OAuth2 credentials
    - Connect `Energy Governance Agent -> Email Performance Dashboard`

22. **Add the Set node for sheet formatting**
    - Node type: **Set**
    - Name: `Prepare Sheet Data`
    - Add fields:
      - `timestamp = {{ $now.toISO() }}`
      - `renewablePercentage = {{ $json.output.renewablePercentage }}`
      - `nonRenewablePercentage = {{ $json.output.nonRenewablePercentage }}`
      - `energyEfficiencyScore = {{ $json.output.energyEfficiencyScore }}`
      - `carbonFootprintKg = {{ $json.output.carbonFootprintKg }}`
      - `policyComplianceStatus = {{ $json.output.policyComplianceStatus }}`
      - `forecastedDemandKwh = {{ $json.output.forecastedDemandKwh }}`
      - `availableRenewableKwh = {{ $json.output.availableRenewableKwh }}`
    - Connect `Energy Governance Agent -> Prepare Sheet Data`

23. **Add the Google Sheets node**
    - Node type: **Google Sheets**
    - Name: `Store Energy KPIs`
    - Operation: `Append`
    - Select your spreadsheet document
    - Select or create the target sheet/tab, such as `Energy KPIs`
    - Ensure the sheet has matching columns
    - Configure Google Sheets OAuth2 credentials
    - Connect `Prepare Sheet Data -> Store Energy KPIs`

24. **Optionally add sticky notes**
    - Add notes for setup, prerequisites, agent roles, tool roles, and output roles for maintainability.

25. **Test each upstream API independently**
    - Confirm all three HTTP requests return valid, structured data
    - Validate that Merge produces one combined item

26. **Test the agent block**
    - Execute through the governance agent
    - Confirm the structured parser returns valid values for all KPI fields

27. **Test each reporting branch**
    - Slack formatting
    - Gmail HTML rendering
    - Google Sheets append behavior

28. **Activate the workflow**
    - Once credentials, endpoints, channel IDs, email addresses, and spreadsheet details are configured, activate the workflow so the schedule can run.

## Credential Configuration Summary

- **OpenAI credentials**
  - Required for:
    - Governance Model
    - Forecasting Model
    - Weather Model
    - Policy Model

- **Perplexity credentials**
  - Required for:
    - Sustainability Research Tool

- **Slack OAuth2 credentials**
  - Required for:
    - Send Sustainability Alert

- **Gmail OAuth2 credentials**
  - Required for:
    - Email Performance Dashboard

- **Google Sheets OAuth2 credentials**
  - Required for:
    - Store Energy KPIs

## Important Build Constraints

- Replace all placeholder values before activation:
  - weather API endpoint
  - energy demand API endpoint
  - renewable generation API endpoint
  - Slack channel ID
  - sustainability team email
  - Google Sheet document and tab
- Ensure the three API branches return compatible item counts if using combine-by-position.
- Consider adding fallback/default handling for missing arrays before Slack/Gmail formatting.
- Consider guarding against divide-by-zero in the optimization code tool.
- Consider restricting AI-generated URL access in the external HTTP tool.

## Sub-workflow Setup

This workflow does **not** invoke any sub-workflow nodes and does not require separate child workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Setup Steps: Import workflow; configure the Energy Analysis Schedule trigger interval. Set API endpoint URLs for Fetch Weather Data, Fetch Energy Demand, and Fetch Renewable Generation nodes. Add AI model credentials to the Agents. Connect Slack credentials to the Send Sustainability Alert node. Link Gmail credentials to the Email Performance Dashboard node. Link Google Sheets credentials; set the sheet ID for the Energy KPIs tab. | Workflow setup guidance |
| How It Works: This workflow automates energy portfolio governance for energy managers, sustainability teams, and policy compliance officers. It eliminates the manual effort of aggregating multi-source energy data, applying forecasting and optimisation logic, and distributing performance outcomes to stakeholders. Three scheduled API feeds, weather data, energy demand, and renewable generation, are combined into a unified dataset. The Energy Governance Agent, backed by shared memory and a governance model, coordinates three specialist agents: a Renewable Forecasting Agent, a Weather Analysis Agent, and a Policy Compliance Agent. Five tools support the orchestration: an Energy Calculator, Optimisation Algorithm Tool, External Data API Tool, Sustainability Research Tool, and Structured KPI Output parser. Results are distributed across Slack, Gmail, and Google Sheets. | Workflow behavior summary |
| Prerequisites: OpenAI API key or compatible LLM; Slack workspace with bot credentials; Gmail account with OAuth credentials; Google Sheets with Energy KPIs tab pre-created. | Required services |
| Use case: Energy managers automating renewable generation forecasting against real-time demand data. | Primary business context |
| Customisation: Swap the Optimisation Algorithm Tool parameters to target carbon intensity, cost, or grid stability objectives. | Extension idea |
| Benefit: Parallel triple-source ingestion ensures governance decisions are based on complete, synchronised energy data. | Architectural rationale |
| Send Sustainability Alert, Email Dashboard & Store Energy KPIs — Why: Distributes governance outcomes via Slack alerts, Gmail dashboards, and Google Sheets KPI logs to keep all stakeholders informed without manual reporting. | Output design intent |
| Energy Calculator, Optimisation & External Data Tools — Why: Applies quantitative calculation, algorithmic optimisation, and live external data enrichment to produce governance-grade energy recommendations. | Tooling design intent |
| Energy Governance Agent & Specialist Agents — Why: Coordinates Renewable Forecasting, Weather Analysis, and Policy Compliance agents using shared memory, enabling context-aware optimisation across all energy dimensions simultaneously. | AI orchestration intent |