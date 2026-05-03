Analyze workforce data and generate planning reports with GPT-4o

https://n8nworkflows.xyz/workflows/analyze-workforce-data-and-generate-planning-reports-with-gpt-4o-13428


# Analyze workforce data and generate planning reports with GPT-4o

## 1. Workflow Overview

**Purpose:**  
This workflow runs on a weekly schedule to analyze workforce metrics (headcount, attrition, utilization, skill gaps) and produce structured workforce intelligence plus planning recommendations using GPT-4o-based agents, with an optional **human-approval gate** when critical signals are detected.

**Typical use cases:** strategic workforce planning, quarterly planning cycles, early warning signals for attrition/capacity gaps, internal mobility planning, executive reporting.

### 1.1 Scheduled Run + Configuration
A weekly trigger starts the workflow, loads configuration constants (org name, thresholds, reporting period), and generates synthetic workforce data (mock departments).

### 1.2 Workforce Intelligence (Analysis + Forecasting Tools)
A dedicated “Workforce Intelligence Agent” (GPT-4o) analyzes each department item and can call two code tools:
- capacity forecasting (capacity gap + hiring need estimate)
- attrition risk scoring (risk level and actions)

Outputs are forced into a structured JSON shape via an output parser.

### 1.3 Aggregation + Planning Orchestration (Mobility + Executive Reporting)
All intelligence outputs are aggregated, then a “Planning Orchestration Agent” (GPT-4o) synthesizes recommendations and calls:
- a mobility specialist tool (GPT-4o-mini) to propose internal mobility opportunities
- an executive reporting tool (GPT-4o) to produce an executive-ready report

Planning output is also structured via a JSON schema parser.

### 1.4 Critical Signal Check → Human Approval (Optional) → Final Report Formatting
If critical findings exist or the plan flags `requiresHumanReview`, the workflow pauses for a manual approval (webhook resume). Finally, a “Format Final Workforce Report” node builds the final report object.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Run + Baseline Configuration + Data Generation

**Overview:**  
Triggers weekly on Monday morning, sets global parameters, and generates mock department workforce items.

**Nodes involved:**
- Weekly Workforce Analysis Trigger
- Workflow Configuration
- Generate Mock Workforce Data

#### Node: Weekly Workforce Analysis Trigger
- **Type / role:** `Schedule Trigger` — entry point.
- **Config:** Runs **every 1 week**, **Monday (day 1)** at **09:00**.
- **Outputs:** One execution item into **Workflow Configuration**.
- **Edge cases / failures:** n8n instance timezone affects “09:00”; missed executions depend on n8n queue/execution settings.

#### Node: Workflow Configuration
- **Type / role:** `Set` — stores workflow constants used later.
- **Config choices:**
  - `organizationName`: `"Acme Corporation"`
  - `criticalAttritionThreshold`: `15` (not directly referenced later, but intended for logic/prompting)
  - `capacityGapThreshold`: `20` (not directly referenced later)
  - `reportingPeriod`: `"Q1 2024"`
  - **Include other fields:** enabled (`includeOtherFields: true`)
- **Outputs:** Passes config + original trigger fields to **Generate Mock Workforce Data**.
- **Edge cases:** If later expressions assume these values exist, removing them can break report formatting.

#### Node: Generate Mock Workforce Data
- **Type / role:** `Code` — creates synthetic dataset (one item per department).
- **Config choices (interpreted):**
  - Returns 5 items: Engineering, Sales, Marketing, Operations, HR.
  - Each item includes: `headcount`, `attritionRate`, `capacityUtilization`, `criticalSkills`, `skillGaps`, `openPositions`, `avgTenure`, `performanceScore`.
- **Outputs:** Each department item goes to **Workforce Intelligence Agent** (agent runs per item).
- **Edge cases / failures:** JS runtime errors stop execution; changes to field names can break agent tools that expect fields like `capacityUtilization`, `headcount`, etc.

---

### Block 2 — Workforce Intelligence Agent (LLM + Structured Output + Tool Calls)

**Overview:**  
Per department item, GPT-4o analyzes workforce health and can call forecasting/risk tools. Output is forced into a structured schema.

**Nodes involved:**
- OpenAI Model - Intelligence Agent
- Workforce Intelligence Agent
- Capacity Forecasting Tool
- Attrition Risk Calculator Tool
- Intelligence Analysis Output Parser

#### Node: OpenAI Model - Intelligence Agent
- **Type / role:** `lmChatOpenAi` — provides the chat model to the intelligence agent.
- **Config:** model `gpt-4o`, temperature `0.3`.
- **Credential:** OpenAI API credential (“OpenAi account”).
- **Connections:** Supplies language model to **Workforce Intelligence Agent** via `ai_languageModel`.
- **Failures:** Invalid API key, model access restrictions, rate limits, network timeouts.

#### Node: Workforce Intelligence Agent
- **Type / role:** `LangChain Agent` — performs analysis, can invoke tools, returns structured JSON.
- **Input text:** `={{ $json }}` (the current department item as context).
- **System message highlights:**
  - Analyze headcount distribution, skills inventory, attrition risk, utilization bottlenecks.
  - Must call **Capacity Forecasting Tool** and **Attrition Risk Calculator Tool**.
  - Must **not** make hiring decisions; focus on analysis/forecasting.
- **Output parser:** enabled; connected to **Intelligence Analysis Output Parser**.
- **Tool connections:** can call:
  - **Capacity Forecasting Tool** (`ai_tool`)
  - **Attrition Risk Calculator Tool** (`ai_tool`)
- **Edge cases:**
  - If tools return error JSON strings, the agent may still attempt to incorporate them; parser may fail if output deviates from schema.
  - Since input is one department at a time, “overall” metrics are approximations unless the agent infers cross-department context.

#### Node: Capacity Forecasting Tool
- **Type / role:** `toolCode` — computes capacity gap and hiring need estimate.
- **Key variables / expressions:**
  - `const workforceData = $fromAI('workforceData', ..., 'json');`
  - Parses `capacityUtilization`, `headcount`, `growthRate`, `attritionRate` (defaults used if missing).
- **Output:** Returns a **stringified JSON** forecast (via `JSON.stringify`).
- **Connections:** Only callable by **Workforce Intelligence Agent**.
- **Edge cases / failures:**
  - If `workforceData` is neither valid JSON nor object → parse error handled; returns `{"error":"Capacity forecast failed"...}`.
  - Uses defaults (`headcount:100`, `capacityUtilization:85`, `growthRate:0.15`, `attritionRate:0.10`) which may distort results if agent doesn’t pass expected fields.

#### Node: Attrition Risk Calculator Tool
- **Type / role:** `toolCode` — computes attrition risk score/level.
- **Key variables / expressions:**
  - `const employeeData = $fromAI('employeeData', ..., 'json');`
  - Reads `avgTenure`, `performanceScore`, `attritionRate`, `department`.
- **Output:** Returns **stringified JSON** with `riskScore`, `riskLevel`, recommendations.
- **Connections:** Only callable by **Workforce Intelligence Agent**.
- **Edge cases / failures:**
  - Same JSON parsing risks as above.
  - `performanceScore` default is `75`, but mock data uses ~`4.x` (department `performanceScore: 4.3`). If the agent passes 4.3 into a 0–100 logic, risk scoring becomes skewed (likely “low performance” falsely). This is a key data/unit mismatch to address.

#### Node: Intelligence Analysis Output Parser
- **Type / role:** `outputParserStructured` — enforces a JSON structure for intelligence output.
- **Schema expectations (high-level):**
  - `overallHealthScore` (number)
  - `criticalFindings` (array of strings)
  - `departmentAnalysis` (array of per-dept objects with risk/capacity statuses and recommendations)
  - `capacityForecast` and `attritionInsights` objects
- **Connections:** Receives agent output; structured result goes to **Aggregate Intelligence Insights**.
- **Edge cases / failures:** If the agent outputs non-matching fields/types, parsing fails and stops the workflow.

---

### Block 3 — Aggregate Intelligence + Planning Orchestration (Mobility + Executive Reporting)

**Overview:**  
Aggregates intelligence from all departments into a single object, then a planning agent synthesizes hiring forecast recommendations, internal mobility opportunities, and executive summary by calling two sub-tools.

**Nodes involved:**
- Aggregate Intelligence Insights
- OpenAI Model - Planning Agent
- Planning Orchestration Agent
- Mobility Analysis Agent Tool
- OpenAI Model - Mobility Agent
- Mobility Workflow Output Parser
- Executive Reporting Agent Tool
- OpenAI Model - Reporting Agent
- Executive Report Output Parser
- Planning Recommendations Output Parser

#### Node: Aggregate Intelligence Insights
- **Type / role:** `Aggregate` — merges all department-level intelligence items.
- **Config:** `aggregateAllItemData` into `intelligenceInsights`.
- **Input:** Multiple items from **Workforce Intelligence Agent**.
- **Output:** Single item to **Planning Orchestration Agent**.
- **Edge cases:** If upstream produces zero items, aggregation can output empty/undefined depending on n8n behavior; downstream expressions may fail or LLM context becomes insufficient.

#### Node: OpenAI Model - Planning Agent
- **Type / role:** `lmChatOpenAi` — model for planning agent.
- **Config:** model `gpt-4o`, temperature `0.4`.
- **Connections:** `ai_languageModel` → **Planning Orchestration Agent**.

#### Node: Planning Orchestration Agent
- **Type / role:** `LangChain Agent` — coordinates planning, calls mobility + executive reporting tools, produces structured recommendations.
- **Input text:** `={{ $json }}` (contains `intelligenceInsights` from aggregation).
- **System message highlights:**
  - Review intelligence insights.
  - Call **Mobility Analysis Agent Tool** and **Executive Reporting Agent Tool**.
  - Provide hiring forecast recommendations but require human approval when appropriate.
  - Must set `requiresHumanReview`.
- **Output parser:** enabled; connected to **Planning Recommendations Output Parser**.
- **Connections:** Main output to **Check Critical Workforce Signals**.
- **Edge cases:**
  - If agent fails to call tools, recommendations may be shallow.
  - Parser failure if output doesn’t match schema example (types/fields missing).

#### Node: Mobility Analysis Agent Tool
- **Type / role:** `agentTool` — tool wrapper that runs a mobility-specialist LLM.
- **Input expression:**  
  `={{ $fromAI("intelligenceData", "Workforce intelligence analysis including skill gaps and capacity needs", "json") }}`
- **System message:** internal mobility specialist; produce mobility opportunities, training plans, cost savings, success probabilities.
- **Output parser:** enabled; connected to **Mobility Workflow Output Parser**.
- **Connections:** Callable by **Planning Orchestration Agent** (`ai_tool`).
- **Edge cases:** If `intelligenceData` is not provided in the call context, tool may receive incomplete data; output parser may fail.

#### Node: OpenAI Model - Mobility Agent
- **Type / role:** `lmChatOpenAi` — model for mobility tool.
- **Config:** model `gpt-4o-mini`, temperature `0.3`.
- **Connections:** `ai_languageModel` → **Mobility Analysis Agent Tool**.

#### Node: Mobility Workflow Output Parser
- **Type / role:** `outputParserStructured` — enforces mobility tool JSON.
- **Schema expectations:** `mobilityRecommendations[]`, `totalOpportunities`, `estimatedTimeToFill`, `retentionImpact`.
- **Connections:** `ai_outputParser` → **Mobility Analysis Agent Tool**.

#### Node: Executive Reporting Agent Tool
- **Type / role:** `agentTool` — tool wrapper that runs an executive-report specialist LLM.
- **Input expression:**  
  `={{ $fromAI("workforceData", "Comprehensive workforce data and planning recommendations", "json") }}`
- **System message:** C-suite report with metrics, strategic recommendations, financial impact, timeline.
- **Output parser:** enabled; connected to **Executive Report Output Parser**.
- **Connections:** Callable by **Planning Orchestration Agent** (`ai_tool`).
- **Edge cases:** Same `$fromAI` context risks; ensure the planning agent passes the right data under the expected key.

#### Node: OpenAI Model - Reporting Agent
- **Type / role:** `lmChatOpenAi` — model for executive reporting tool.
- **Config:** model `gpt-4o`, temperature `0.5`.
- **Connections:** `ai_languageModel` → **Executive Reporting Agent Tool**.

#### Node: Executive Report Output Parser
- **Type / role:** `outputParserStructured` — enforces executive report JSON.
- **Schema expectations:** `executiveReport.title`, `keyMetrics`, `strategicRecommendations`, `financialImpact`, `timeline`.
- **Connections:** `ai_outputParser` → **Executive Reporting Agent Tool**.

#### Node: Planning Recommendations Output Parser
- **Type / role:** `outputParserStructured` — enforces planning agent output JSON.
- **Schema expectations (high-level):**
  - `hiringForecast` (per department recommendations + yearly summary)
  - `internalMobilityOpportunities`
  - `executiveSummary`
  - `riskMitigation[]`
  - `requiresHumanReview` (boolean)
- **Connections:** `ai_outputParser` → **Planning Orchestration Agent**.

---

### Block 4 — Critical Signal Routing + Human Approval Gate + Final Report Object

**Overview:**  
Routes execution depending on criticality: either generate the report immediately or pause for human approval, then generate the report.

**Nodes involved:**
- Check Critical Workforce Signals
- Flag for Human Review
- Wait for Human Approval
- Format Final Workforce Report

#### Node: Check Critical Workforce Signals
- **Type / role:** `IF` — determines if human review is needed.
- **Conditions (OR):**
  1. `={{ $json.requiresHumanReview }}` is **true**
  2. `={{ $json.criticalFindings }}` is **not empty** (array)
- **Outputs:**
  - **True branch (index 1):** → Flag for Human Review
  - **False branch (index 0):** → Format Final Workforce Report
- **Edge cases:**
  - `criticalFindings` may not exist at this stage (planning output schema has it? the *intelligence* schema has it). If missing, the array check may behave unexpectedly under “loose” validation.
  - If the planning agent does not include `requiresHumanReview`, condition may evaluate false and skip approval unintentionally.

#### Node: Flag for Human Review
- **Type / role:** `Set` — creates review metadata.
- **Fields set:**
  - `reviewRequired: true`
  - `reviewReason: "Critical workforce signals detected requiring human approval"`
  - `reviewInstructions: ...`
  - `pendingApproval: true`
- **Output:** → Wait for Human Approval
- **Edge cases:** This node does not merge back with the planning recommendation object; if you need the original recommendations after approval, you typically must carry them forward explicitly or store them externally.

#### Node: Wait for Human Approval
- **Type / role:** `Wait` — pauses workflow until webhook resume.
- **Config:** `resume: webhook`, HTTP method `POST`.
- **Output:** → Format Final Workforce Report
- **Edge cases / failures:**
  - Webhook URL must be called to resume; otherwise execution remains waiting.
  - Authentication/authorization for the resume webhook is not configured here (consider adding a secret/token check).

#### Node: Format Final Workforce Report
- **Type / role:** `Set` — builds final consolidated report fields.
- **Key expressions:**
  - `reportDate: ={{ $now.toISO() }}`
  - `organization: ={{ $('Workflow Configuration').first().json.organizationName }}`
  - `reportingPeriod: ={{ $('Workflow Configuration').first().json.reportingPeriod }}`
  - `intelligenceAnalysis: ={{ $json.intelligenceInsights }}`
  - `planningRecommendations: ={{ $json }}`
  - `humanReviewStatus: ={{ $json.requiresHumanReview ? "Approved" : "Auto-approved" }}`
- **Input:** Comes either directly from IF false branch (planning output) or from Wait node after approval.
- **Important edge case:** If execution goes through the human approval path, the incoming `$json` is whatever the webhook resumes with (often minimal), so:
  - `$json.intelligenceInsights` may be missing
  - `$json.requiresHumanReview` may be missing
  - `planningRecommendations` may be replaced by approval payload  
  To preserve original data, you’d typically store the planning output (e.g., in DB, static data, or workflow execution data) and reload it after approval.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Workforce Analysis Trigger | scheduleTrigger | Weekly entry point | — | Workflow Configuration | ## How It Works (full note): This workflow automates workforce intelligence analysis and strategic planning... |
| Workflow Configuration | set | Store org constants and thresholds | Weekly Workforce Analysis Trigger | Generate Mock Workforce Data | ## How It Works (full note): This workflow automates workforce intelligence analysis and strategic planning... |
| Generate Mock Workforce Data | code | Create synthetic workforce dataset | Workflow Configuration | Workforce Intelligence Agent | ## How It Works (full note): This workflow automates workforce intelligence analysis and strategic planning... |
| OpenAI Model - Intelligence Agent | lmChatOpenAi | LLM for intelligence agent | — | Workforce Intelligence Agent | ## Workforce Intelligence\nIdentifies operational inefficiencies and capability shortfalls to optimize resource allocation decisions. |
| Workforce Intelligence Agent | agent | Analyze workforce data; call tools; structured output | Generate Mock Workforce Data | Aggregate Intelligence Insights | ## Workforce Intelligence\nIdentifies operational inefficiencies and capability shortfalls to optimize resource allocation decisions. |
| Capacity Forecasting Tool | toolCode | Tool: forecast capacity gap & hiring need | (Tool call from Workforce Intelligence Agent) | (Tool result to Workforce Intelligence Agent) | ## Cognitive Forecasting \nEnables proactive hiring and training strategies to prevent capacity bottlenecks during scaling. |
| Attrition Risk Calculator Tool | toolCode | Tool: compute attrition risk score/level | (Tool call from Workforce Intelligence Agent) | (Tool result to Workforce Intelligence Agent) | ## Attrition Risk \nTriggers retention interventions before critical talent exits, reducing replacement costs and knowledge loss. |
| Intelligence Analysis Output Parser | outputParserStructured | Enforce intelligence JSON schema | Workforce Intelligence Agent | Aggregate Intelligence Insights | ## Workforce Intelligence\nIdentifies operational inefficiencies and capability shortfalls to optimize resource allocation decisions. |
| Aggregate Intelligence Insights | aggregate | Combine all dept intelligence into one object | Workforce Intelligence Agent | Planning Orchestration Agent |  |
| OpenAI Model - Planning Agent | lmChatOpenAi | LLM for planning orchestration | — | Planning Orchestration Agent |  |
| Planning Orchestration Agent | agent | Synthesize planning; call mobility + exec reporting tools | Aggregate Intelligence Insights | Check Critical Workforce Signals |  |
| Planning Recommendations Output Parser | outputParserStructured | Enforce planning recommendations schema | Planning Orchestration Agent | Check Critical Workforce Signals |  |
| OpenAI Model - Mobility Agent | lmChatOpenAi | LLM for mobility tool | — | Mobility Analysis Agent Tool |  |
| Mobility Analysis Agent Tool | agentTool | Internal mobility specialist tool | (Tool call from Planning Orchestration Agent) | (Tool result to Planning Orchestration Agent) |  |
| Mobility Workflow Output Parser | outputParserStructured | Enforce mobility tool JSON schema | Mobility Analysis Agent Tool | Mobility Analysis Agent Tool |  |
| OpenAI Model - Reporting Agent | lmChatOpenAi | LLM for executive reporting tool | — | Executive Reporting Agent Tool |  |
| Executive Reporting Agent Tool | agentTool | Executive report specialist tool | (Tool call from Planning Orchestration Agent) | (Tool result to Planning Orchestration Agent) |  |
| Executive Report Output Parser | outputParserStructured | Enforce executive report JSON schema | Executive Reporting Agent Tool | Executive Reporting Agent Tool |  |
| Check Critical Workforce Signals | if | Route to approval or auto proceed | Planning Orchestration Agent | Format Final Workforce Report; Flag for Human Review |  |
| Flag for Human Review | set | Prepare approval metadata | Check Critical Workforce Signals (true) | Wait for Human Approval |  |
| Wait for Human Approval | wait | Pause until webhook approval | Flag for Human Review | Format Final Workforce Report |  |
| Format Final Workforce Report | set | Create final report object | Check Critical Workforce Signals (false) OR Wait for Human Approval | — |  |
| Sticky Note | stickyNote | Notes: prerequisites/use cases/benefits | — | — | ## Prerequisites\nAPI key, HR data access (anonymized employee metrics) ... |
| Sticky Note1 | stickyNote | Notes: setup steps | — | — | ## Setup Steps\n1. Configure API credentials with Llama-3.1-70B-Instruct model access ... |
| Sticky Note2 | stickyNote | Notes: how it works | — | — | ## How It Works\nThis workflow automates workforce intelligence analysis... |
| Sticky Note4 | stickyNote | Notes: attrition risk | — | — | ## Attrition Risk ... |
| Sticky Note5 | stickyNote | Notes: cognitive forecasting | — | — | ## Cognitive Forecasting ... |
| Sticky Note6 | stickyNote | Notes: workforce intelligence | — | — | ## Workforce Intelligence ... |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: *AI-driven workforce intelligence and planning optimization system* (or your preferred name).
   - (Optional) Keep it inactive until credentials and approval flow are tested.

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure: Interval = **Weeks (1)**, Day = **Monday**, Hour = **09:00**.
   - This is the entry node.

3. **Add “Workflow Configuration” (Set node)**
   - Node: **Set**
   - Add fields:
     - `organizationName` (String) = `Acme Corporation`
     - `criticalAttritionThreshold` (Number) = `15`
     - `capacityGapThreshold` (Number) = `20`
     - `reportingPeriod` (String) = `Q1 2024`
   - Enable **Include Other Fields**.
   - Connect: Schedule Trigger → Workflow Configuration.

4. **Add “Generate Mock Workforce Data” (Code node)**
   - Node: **Code**
   - Paste JS that returns one item per department (Engineering/Sales/Marketing/Operations/HR) with the fields used in the JSON.
   - Connect: Workflow Configuration → Generate Mock Workforce Data.

5. **Create OpenAI credential**
   - Add credentials: **OpenAI API**
   - Provide API key with access to `gpt-4o` and `gpt-4o-mini`.

6. **Add “OpenAI Model - Intelligence Agent”**
   - Node: **OpenAI Chat Model (LangChain)** (`lmChatOpenAi`)
   - Model: `gpt-4o`
   - Temperature: `0.3`
   - Select your OpenAI credentials.

7. **Add “Capacity Forecasting Tool”**
   - Node: **Code Tool** (`toolCode`)
   - Description: “Forecasts workforce capacity needs…”
   - Code: uses `$fromAI('workforceData', ..., 'json')` and returns JSON string forecast.

8. **Add “Attrition Risk Calculator Tool”**
   - Node: **Code Tool** (`toolCode`)
   - Description: “Calculates attrition risk scores…”
   - Code: uses `$fromAI('employeeData', ..., 'json')` and returns JSON string risk analysis.
   - **Important:** Align performance score scale (0–100 vs 1–5) before production use.

9. **Add “Intelligence Analysis Output Parser”**
   - Node: **Structured Output Parser** (`outputParserStructured`)
   - Provide the JSON schema example (overallHealthScore, criticalFindings, departmentAnalysis, capacityForecast, attritionInsights).

10. **Add “Workforce Intelligence Agent”**
   - Node: **AI Agent** (`agent`)
   - Input text: `{{ $json }}`
   - System message: workforce intelligence analyst instructions (analysis only; no hiring decisions).
   - Enable **Output Parser** and attach **Intelligence Analysis Output Parser**.
   - Connect language model: OpenAI Model - Intelligence Agent → Workforce Intelligence Agent (`ai_languageModel`).
   - Connect tools:
     - Capacity Forecasting Tool → Workforce Intelligence Agent (`ai_tool`)
     - Attrition Risk Calculator Tool → Workforce Intelligence Agent (`ai_tool`)
   - Connect: Generate Mock Workforce Data → Workforce Intelligence Agent (main).

11. **Add “Aggregate Intelligence Insights”**
   - Node: **Aggregate**
   - Mode: **Aggregate All Item Data**
   - Destination field: `intelligenceInsights`
   - Connect: Workforce Intelligence Agent → Aggregate Intelligence Insights.

12. **Add planning model and agent**
   - Add **OpenAI Model - Planning Agent** (`lmChatOpenAi`):
     - Model `gpt-4o`, temperature `0.4`
   - Add **Planning Recommendations Output Parser** (`outputParserStructured`) with the provided planning schema example.
   - Add **Planning Orchestration Agent** (`agent`):
     - Input text: `{{ $json }}`
     - System message: coordinator instructions; must call mobility + executive reporting; set `requiresHumanReview`.
     - Enable output parser and attach Planning Recommendations Output Parser.
   - Connect:
     - Aggregate Intelligence Insights → Planning Orchestration Agent (main)
     - OpenAI Model - Planning Agent → Planning Orchestration Agent (`ai_languageModel`)

13. **Add Mobility tool chain**
   - Add **OpenAI Model - Mobility Agent** (`lmChatOpenAi`):
     - Model `gpt-4o-mini`, temperature `0.3`
   - Add **Mobility Workflow Output Parser** (`outputParserStructured`) with mobility schema example.
   - Add **Mobility Analysis Agent Tool** (`agentTool`):
     - Text: `{{ $fromAI("intelligenceData", "Workforce intelligence analysis including skill gaps and capacity needs", "json") }}`
     - System message: mobility specialist instructions
     - Enable output parser and attach Mobility Workflow Output Parser
   - Connect:
     - OpenAI Model - Mobility Agent → Mobility Analysis Agent Tool (`ai_languageModel`)
     - Mobility Workflow Output Parser → Mobility Analysis Agent Tool (`ai_outputParser`)
     - Mobility Analysis Agent Tool → Planning Orchestration Agent (`ai_tool`)

14. **Add Executive reporting tool chain**
   - Add **OpenAI Model - Reporting Agent** (`lmChatOpenAi`):
     - Model `gpt-4o`, temperature `0.5`
   - Add **Executive Report Output Parser** (`outputParserStructured`) with executive report schema example.
   - Add **Executive Reporting Agent Tool** (`agentTool`):
     - Text: `{{ $fromAI("workforceData", "Comprehensive workforce data and planning recommendations", "json") }}`
     - System message: executive reporting instructions
     - Enable output parser and attach Executive Report Output Parser
   - Connect:
     - OpenAI Model - Reporting Agent → Executive Reporting Agent Tool (`ai_languageModel`)
     - Executive Report Output Parser → Executive Reporting Agent Tool (`ai_outputParser`)
     - Executive Reporting Agent Tool → Planning Orchestration Agent (`ai_tool`)

15. **Add “Check Critical Workforce Signals”**
   - Node: **IF**
   - Condition group: **OR**
     - Boolean is true: `{{ $json.requiresHumanReview }}`
     - Array not empty: `{{ $json.criticalFindings }}`
   - Connect: Planning Orchestration Agent → Check Critical Workforce Signals.

16. **Add Human approval path**
   - Add **Flag for Human Review** (Set):
     - `reviewRequired` = true
     - `reviewReason` = “Critical workforce signals…”
     - `reviewInstructions` = your instruction text
     - `pendingApproval` = true
   - Add **Wait for Human Approval** (Wait):
     - Resume: **Webhook**
     - HTTP method: **POST**
   - Connect:
     - IF (true) → Flag for Human Review → Wait for Human Approval

17. **Add “Format Final Workforce Report”**
   - Node: **Set**
   - Fields:
     - `reportTitle` = “Workforce Intelligence and Planning Report”
     - `reportDate` = `{{ $now.toISO() }}`
     - `organization` = `{{ $('Workflow Configuration').first().json.organizationName }}`
     - `reportingPeriod` = `{{ $('Workflow Configuration').first().json.reportingPeriod }}`
     - `intelligenceAnalysis` = `{{ $json.intelligenceInsights }}`
     - `planningRecommendations` (Object) = `{{ $json }}`
     - `humanReviewStatus` = `{{ $json.requiresHumanReview ? "Approved" : "Auto-approved" }}`
   - Connect:
     - IF (false) → Format Final Workforce Report
     - Wait for Human Approval → Format Final Workforce Report

18. **(Optional but recommended) Add a distribution step**
   - Email/Slack/Drive/Notion export of the final report to stakeholders (not present in JSON but referenced by sticky notes).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Prerequisites: API key, HR data access (anonymized employee metrics)… Benefits: Reduces planning cycle time by 70%…” | Sticky note “Prerequisites / Use Cases / Customization / Benefits” |
| “## Setup Steps: 1. Configure API credentials with Llama-3.1-70B-Instruct model access…” | Sticky note “Setup Steps” — **Note:** current workflow actually uses **OpenAI gpt-4o / gpt-4o-mini**, not Llama 3.1 |
| “## How It Works … Weekly triggers initiate the analysis cycle… specialized AI agents… human oversight…” | Sticky note “How It Works” |
| “## Workforce Intelligence … optimize resource allocation decisions.” | Sticky note “Workforce Intelligence” |
| “## Cognitive Forecasting … prevent capacity bottlenecks during scaling.” | Sticky note “Cognitive Forecasting” |
| “## Attrition Risk … reducing replacement costs and knowledge loss.” | Sticky note “Attrition Risk” |

Dispositif: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.