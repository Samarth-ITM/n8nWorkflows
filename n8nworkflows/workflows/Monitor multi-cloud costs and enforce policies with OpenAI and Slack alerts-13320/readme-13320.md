Monitor multi-cloud costs and enforce policies with OpenAI and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-multi-cloud-costs-and-enforce-policies-with-openai-and-slack-alerts-13320


# Monitor multi-cloud costs and enforce policies with OpenAI and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name (internal):** Cloud Cost Intelligence and Governance Orchestration  
**Title (provided):** Monitor multi-cloud costs and enforce policies with OpenAI and Slack alerts

**Purpose:**  
Runs a daily cloud cost check, analyzes spend using a first OpenAI-powered “Cost Intelligence” agent (optimization + risk), then passes results to a second OpenAI-powered “Governance” agent (policy enforcement + actions). The workflow routes notifications to Slack (critical/high) and optionally escalates to finance via email, while logging decisions to a Data Table for auditability.

**Target use cases:**
- FinOps daily spend monitoring and anomaly detection
- Budget policy enforcement (threshold-based)
- Automated optimization recommendations and governance actions
- Stakeholder alerting and finance escalation
- Audit trail of daily analyses/decisions

### 1.1 Automated Cost Monitoring (Trigger + Configuration)
Daily schedule trigger → set governance thresholds/channels → (in this sample) simulate spend data.

### 1.2 Dual AI Analysis (Cost Intelligence → Governance)
Agent #1 creates structured cost intelligence (trend, budget status, drivers, savings, risk).  
Agent #2 converts that into governance severity, escalation requirement, and actionable alert text.

### 1.3 Severity-Based Routing (Slack/Email) + Audit Logging
Switch routes by severity/escalation → Slack critical/high messages and finance email if needed → write record to a Data Table.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Automated Cost Monitoring (Trigger, Config, Spend Input)
**Overview:** Triggers daily at a configured hour, loads thresholds and notification targets, then provides spend data (here simulated) to the AI analysis pipeline.

**Nodes involved:**
- Daily Cloud Cost Check
- Workflow Configuration
- Simulate Cloud Spend Data

#### Node: Daily Cloud Cost Check
- **Type / role:** `Schedule Trigger` — entry point, runs workflow on a time schedule.
- **Configuration choices:** Daily schedule with `triggerAtHour = 8` (server timezone of n8n instance).
- **Inputs / outputs:** No inputs; outputs one item to **Workflow Configuration**.
- **Version:** typeVersion `1.3`.
- **Edge cases / failures:**
  - Timezone mismatch vs. expected billing refresh time.
  - If instance is down at trigger time, execution may be missed depending on n8n setup.

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes configurable parameters (thresholds, channels, finance email).
- **Configuration choices (interpreted):**
  - Sets:
    - `costThresholdCritical = 10000`
    - `costThresholdHigh = 5000`
    - `slackChannelCritical = <placeholder>`
    - `slackChannelHigh = <placeholder>`
    - `financeEmail = <placeholder>`
  - **Include other fields:** enabled (passes through any incoming fields too).
- **Inputs / outputs:** From **Daily Cloud Cost Check** → to **Simulate Cloud Spend Data**.
- **Version:** typeVersion `3.4`.
- **Edge cases / failures:**
  - Placeholders not replaced → Slack node fails with invalid channel; Email node fails with invalid recipient.
  - Threshold logic is used later by expressions referencing this node; renaming the node breaks those expressions.

#### Node: Simulate Cloud Spend Data
- **Type / role:** `Set` — sample spend dataset used in place of real cloud billing APIs.
- **Configuration choices (interpreted):**
  - Produces a single spend snapshot:
    - `cloudProvider = "AWS"`
    - `totalSpend = 12500`
    - `previousMonthSpend = 8000`
    - `budgetLimit = 10000`
    - `topServices` (array of service cost + trend)
    - `unusedResources` (array of resourceId/type/monthlyCost)
    - `timestamp = {{$now.toISO()}}`
  - **Include other fields:** enabled.
- **Inputs / outputs:** From **Workflow Configuration** → to **Cost Intelligence Agent**.
- **Version:** typeVersion `3.4`.
- **Edge cases / failures:**
  - `topServices`/`unusedResources` are provided as arrays; if converted accidentally to strings, the AI prompt still prints but downstream assumptions may degrade.
  - In production, replace with AWS Cost Explorer / Azure Cost Management / GCP Billing export calls; API pagination and time-window alignment become key risks.

**Sticky notes covering this block (context):**
- **Automated Cost Monitoring**: “Continuous oversight prevents cost surprises and catches anomalies before they impact budgets significantly”
- **How It Works**: full business context description (multi-cloud cost governance)
- **Setup Steps**: includes aligning schedule, configuring cloud provider APIs, OpenAI keys, Slack, Gmail/email

---

### Block 2.2 — Dual AI Analysis (Cost Intelligence Agent + Parser + Model)
**Overview:** Uses OpenAI (via LangChain agent node) to convert raw spend metrics into structured cost intelligence: budget status, trend, drivers, savings, and risk.

**Nodes involved:**
- Cost Intelligence Agent
- OpenAI Model - Cost Intelligence
- Cost Analysis Output Parser

#### Node: Cost Intelligence Agent
- **Type / role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt + model + structured parsing.
- **Configuration choices (interpreted):**
  - **Prompt (`text`)** formats the spend snapshot into readable sections:
    - Provider, Total Spend, Previous Month, Budget Limit
    - `Top Services` and `Unused Resources` rendered with `JSON.stringify(..., null, 2)`
  - **System message** defines responsibilities:
    - Calculate % change MoM
    - Determine `budgetStatus` rules (within/over/approaching within 10%)
    - Identify top 3 cost drivers + unused resources + potential savings
    - Produce optimization list with estimated savings and effort
    - Assess risk level using provided guidelines
    - “Return structured JSON output with all required fields.”
  - **Output parser enabled:** `hasOutputParser = true`
- **Key expressions / variables:**
  - `{{ $json.cloudProvider }}`, `{{ $json.totalSpend }}`, etc.
  - `{{ JSON.stringify($json.topServices, null, 2) }}`
  - `{{ JSON.stringify($json.unusedResources, null, 2) }}`
- **Inputs / outputs:**
  - Main input: from **Simulate Cloud Spend Data**
  - AI language model input: from **OpenAI Model - Cost Intelligence** via `ai_languageModel`
  - AI output parser input: from **Cost Analysis Output Parser** via `ai_outputParser`
  - Main output: to **Governance Agent**
- **Version:** typeVersion `3.1`.
- **Edge cases / failures:**
  - Model returns non-JSON or schema-mismatched JSON → parser error.
  - Ambiguous numeric formatting (commas, currency symbols) can break schema number fields.
  - If incoming spend fields are null/undefined, prompt still renders but analysis may be inconsistent.

#### Node: OpenAI Model - Cost Intelligence
- **Type / role:** `ChatOpenAI` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM used by the agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1` (low randomness, more deterministic structure)
- **Credentials:** OpenAI API credential “OpenAi account”.
- **Connections:** Outputs to **Cost Intelligence Agent** on `ai_languageModel`.
- **Version:** typeVersion `1.3`.
- **Edge cases / failures:**
  - Invalid/expired OpenAI key, quota exceeded, model not available in account/region.
  - Rate limits on daily runs across many workflows.

#### Node: Cost Analysis Output Parser
- **Type / role:** `Structured Output Parser` — enforces JSON schema for the agent output.
- **Configuration choices:**
  - Manual JSON schema requiring (at minimum):  
    `totalSpend, budgetStatus, spendTrend, percentageChange, topCostDrivers, optimizationOpportunities, riskLevel, analysisReasoning`
  - Optional but defined: `unusedResourcesCount, potentialSavings`
- **Connections:** Feeds parser to **Cost Intelligence Agent** via `ai_outputParser`.
- **Version:** typeVersion `1.3`.
- **Edge cases / failures:**
  - If the model omits required fields or outputs wrong types (e.g., “10%” as string), parsing fails and the workflow stops unless error handling is added.

**Sticky note covering this block (context):**
- **Dual AI Analysis**: “Combines optimization insights with compliance enforcement…”

---

### Block 2.3 — Governance Decisioning (Governance Agent + Parser + Model)
**Overview:** Converts the cost intelligence output into governance actions: severity classification, whether finance escalation is required, and formatted alert content.

**Nodes involved:**
- Governance Agent
- OpenAI Model - Governance
- Governance Output Parser

#### Node: Governance Agent
- **Type / role:** `LangChain Agent` — governance orchestration using the cost intelligence JSON plus configured thresholds.
- **Configuration choices (interpreted):**
  - **Prompt (`text`)** includes:
    - `Cost Intelligence Analysis` as `{{ JSON.stringify($json.output, null, 2) }}`
    - Threshold values pulled from **Workflow Configuration**:
      - `{{ $('Workflow Configuration').first().json.costThresholdCritical }}`
      - `{{ $('Workflow Configuration').first().json.costThresholdHigh }}`
  - **System message** defines:
    - Severity mapping rules (critical/high/medium/low)
    - Finance escalation conditions (critical/high OR >10% budget overrun)
    - Alert message format (mentions “severity emoji” requirement)
    - Return structured JSON
  - **Output parser enabled:** `hasOutputParser = true`
- **Inputs / outputs:**
  - Main input: from **Cost Intelligence Agent**
  - AI language model: from **OpenAI Model - Governance**
  - AI output parser: from **Governance Output Parser**
  - Main output: to **Route by Severity** and **Log Cost Analysis**
- **Version:** typeVersion `3.1`.
- **Edge cases / failures:**
  - Relies on `Cost Intelligence Agent` output being under `$json.output`. If agent node output structure changes, expressions break.
  - Threshold references use node name string; renaming **Workflow Configuration** breaks them.
  - The system message requests emojis, while downstream routing relies on `severity` field (string). Ensure the agent returns `severity` strictly as `critical|high|medium|low` regardless of emojis in `alertMessage`.

#### Node: OpenAI Model - Governance
- **Type / role:** `ChatOpenAI` — LLM for governance agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Credentials:** OpenAI API credential “OpenAi account”.
- **Connections:** to **Governance Agent** on `ai_languageModel`.
- **Version:** typeVersion `1.3`.
- **Edge cases / failures:** same as other OpenAI model node (auth/quota/rate limits).

#### Node: Governance Output Parser
- **Type / role:** `Structured Output Parser` — schema enforcement for governance output.
- **Configuration choices:**
  - Required: `severity`, `requiresFinanceEscalation`, `alertMessage`, `recommendedActions`, `governanceDecision`
  - Optional: `escalationReason`
- **Connections:** to **Governance Agent** on `ai_outputParser`.
- **Version:** typeVersion `1.3`.
- **Edge cases / failures:**
  - If `requiresFinanceEscalation` returned as `"true"` (string) instead of boolean → parse failure.
  - If `recommendedActions` is not a string array → parse failure.

---

### Block 2.4 — Severity-Based Routing + Multi-Channel Alerts + Logging
**Overview:** Routes governance output to Slack and email based on severity/escalation flags and writes a record to a Data Table for audit trail.

**Nodes involved:**
- Route by Severity
- Slack Alert - Critical
- Slack Alert - High
- Email Finance Team
- Log Cost Analysis

#### Node: Route by Severity
- **Type / role:** `Switch` — conditional routing based on severity/escalation.
- **Configuration choices (interpreted):**
  - Rules (with renamed outputs):
    1. Output “Critical” if `{{$json.severity}} == "critical"`
    2. Output “High” if `{{$json.severity}} == "high"`
    3. Output “Finance Escalation” if `{{$json.requiresFinanceEscalation}} == true`
  - **Ignore case:** enabled
  - **Fallback output:** enabled, renamed to “Low Priority”
- **Inputs / outputs:**
  - Input: from **Governance Agent**
  - Outputs: connected (in a single `main` output wiring) to:
    - **Slack Alert - Critical**
    - **Slack Alert - High**
    - **Email Finance Team**
- **Version:** typeVersion `3.4`.
- **Important behavioral note:**  
  The workflow JSON shows **all three alert nodes connected to the same switch output index**, which may cause **all three to receive the same items** depending on how n8n interprets that connection set. Typically, each rule output should connect to its specific downstream node (Critical → Slack Critical, High → Slack High, Finance Escalation → Email). You should verify and correct the connections in the canvas.
- **Edge cases / failures:**
  - If severity is “medium/low” and escalation is false, it goes to fallback (“Low Priority”) which currently has **no connected node**, so nothing happens (besides logging).
  - If the agent returns unexpected severity strings, items may all go to fallback.

#### Node: Slack Alert - Critical
- **Type / role:** `Slack` — posts critical alert message to a channel.
- **Configuration choices:**
  - Auth: OAuth2 Slack
  - Channel selection: by channel ID pulled from config:  
    `{{ $('Workflow Configuration').first().json.slackChannelCritical }}`
  - Message text uses governance output:
    - `{{$json.output.alertMessage}}`
    - Formats `recommendedActions` into numbered lines
    - Includes `governanceDecision`
- **Inputs / outputs:** From **Route by Severity** (intended: Critical route).
- **Credentials:** Slack OAuth2 credential “Slack account”.
- **Version:** typeVersion `2.4`.
- **Edge cases / failures:**
  - Invalid channel ID / missing scopes (e.g., `chat:write`, channel access).
  - Message formatting expression errors if `recommendedActions` missing or not an array.
  - If the upstream governance output isn’t under `$json.output` (structure mismatch), message becomes blank or fails.

#### Node: Slack Alert - High
- **Type / role:** `Slack` — posts high-priority alert.
- **Configuration choices:**
  - Channel: `{{ $('Workflow Configuration').first().json.slackChannelHigh }}`
  - Message includes alertMessage + numbered recommended actions (no governanceDecision section here).
- **Inputs / outputs:** From **Route by Severity** (intended: High route).
- **Credentials:** Slack OAuth2 “Slack account”.
- **Version:** typeVersion `2.4`.
- **Edge cases / failures:** same as critical Slack node.

#### Node: Email Finance Team
- **Type / role:** `Email Send` — escalates to finance via HTML email.
- **Configuration choices:**
  - `toEmail`: `{{ $('Workflow Configuration').first().json.financeEmail }}`
  - `fromEmail`: placeholder (must be replaced)
  - Subject: `Cloud Cost Governance Alert - <SEVERITY>`
  - HTML body includes:
    - severity (uppercased)
    - alert summary (`$json.output.alertMessage`)
    - escalation reason (fallback string if missing)
    - ordered list from `recommendedActions`
    - governanceDecision
- **Inputs / outputs:** From **Route by Severity** (intended: Finance Escalation route).
- **Version:** typeVersion `2.1`.
- **Edge cases / failures:**
  - SMTP configuration/credentials not set in n8n (this node depends on instance email setup or node credentials depending on n8n mode).
  - Invalid `fromEmail` causes delivery rejection.
  - HTML building expression errors if `recommendedActions` is not an array.

#### Node: Log Cost Analysis
- **Type / role:** `Data Table` — stores execution results for audit trail.
- **Configuration choices:**
  - Data table: `cloud_cost_analysis_log` (selected by name)
  - Column mapping: auto-map input data
- **Inputs / outputs:** Directly from **Governance Agent** (runs regardless of severity routing).
- **Version:** typeVersion `1.1`.
- **Edge cases / failures:**
  - Data table missing/not created → node errors.
  - Auto-map can create inconsistent columns across changing JSON structures; governance output stability is important.

**Sticky notes covering this block (context):**
- **Severity-Based Routing**: urgent overruns routed quickly + audit trails
- **Multi-Channel Alerts**: context-appropriate communications

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Cloud Cost Check | Schedule Trigger | Daily entry point at 08:00 | — | Workflow Configuration | ## Automated Cost Monitoring<br>**Why**: Continuous oversight prevents cost surprises and catches anomalies before they impact budgets significantly |
| Workflow Configuration | Set | Central thresholds + notification targets | Daily Cloud Cost Check | Simulate Cloud Spend Data | ## Automated Cost Monitoring<br>**Why**: Continuous oversight prevents cost surprises and catches anomalies before they impact budgets significantly |
| Simulate Cloud Spend Data | Set | Sample cost dataset (replace with cloud billing APIs) | Workflow Configuration | Cost Intelligence Agent | ## Automated Cost Monitoring<br>**Why**: Continuous oversight prevents cost surprises and catches anomalies before they impact budgets significantly |
| Cost Intelligence Agent | LangChain Agent | AI cost analysis + optimization + risk | Simulate Cloud Spend Data | Governance Agent | ## Dual AI Analysis<br>**Why**: Combines optimization insights with compliance enforcement, balancing efficiency and control simultaneously |
| OpenAI Model - Cost Intelligence | OpenAI Chat Model | LLM backing for cost intelligence agent | — | Cost Intelligence Agent | ## Dual AI Analysis<br>**Why**: Combines optimization insights with compliance enforcement, balancing efficiency and control simultaneously |
| Cost Analysis Output Parser | Structured Output Parser | Enforce schema for cost intelligence output | — | Cost Intelligence Agent | ## Dual AI Analysis<br>**Why**: Combines optimization insights with compliance enforcement, balancing efficiency and control simultaneously |
| Governance Agent | LangChain Agent | Severity decisioning + actions + escalation | Cost Intelligence Agent | Route by Severity; Log Cost Analysis | ## Dual AI Analysis<br>**Why**: Combines optimization insights with compliance enforcement, balancing efficiency and control simultaneously |
| OpenAI Model - Governance | OpenAI Chat Model | LLM backing for governance agent | — | Governance Agent | ## Dual AI Analysis<br>**Why**: Combines optimization insights with compliance enforcement, balancing efficiency and control simultaneously |
| Governance Output Parser | Structured Output Parser | Enforce schema for governance output | — | Governance Agent | ## Dual AI Analysis<br>**Why**: Combines optimization insights with compliance enforcement, balancing efficiency and control simultaneously |
| Route by Severity | Switch | Route notifications by severity/escalation | Governance Agent | Slack Alert - Critical; Slack Alert - High; Email Finance Team | ## Severity-Based Routing<br>**Why**: Ensures urgent overruns reach teams instantly while maintaining comprehensive audit trails |
| Slack Alert - Critical | Slack | Send critical Slack alert | Route by Severity | — | ## Multi-Channel Alerts<br>**Why**: Delivers context-appropriate communications enabling rapid response and informed decision-making |
| Slack Alert - High | Slack | Send high-priority Slack alert | Route by Severity | — | ## Multi-Channel Alerts<br>**Why**: Delivers context-appropriate communications enabling rapid response and informed decision-making |
| Email Finance Team | Email Send | Escalate to finance by email | Route by Severity | — | ## Multi-Channel Alerts<br>**Why**: Delivers context-appropriate communications enabling rapid response and informed decision-making |
| Log Cost Analysis | Data Table | Persist governance output for audit | Governance Agent | — | ## Severity-Based Routing<br>**Why**: Ensures urgent overruns reach teams instantly while maintaining comprehensive audit trails |
| Sticky Note | Sticky Note | Comment / prerequisites and benefits | — | — | ## Prerequisites<br>Cloud provider API access (AWS/Azure/GCP billing), OpenAI API account<br>## Use Cases<br>Multi-cloud cost optimization, budget compliance enforcement<br>## Customization<br>Modify AI prompts for company-specific cost policies, adjust severity thresholds for alerts<br>## Benefits<br>Prevents budget overruns through proactive alerts, reduces cloud waste by 30-50% |
| Sticky Note1 | Sticky Note | Comment / setup steps | — | — | ## Setup Steps<br>1. Connect **Daily Trigger** (schedule time aligned with billing cycle updates)<br>2. Configure **Cloud Provider APIs**<br>3. Add **OpenAI API keys** to Cost Intelligence Agent and Governance Agent nodes<br>4. Set budget thresholds and cost policies in Governance Agent prompts<br>5. Configure **Slack** webhooks for critical and high-priority alerts<br>6. Link **Gmail** credentials for finance team report distribution |
| Sticky Note2 | Sticky Note | Comment / how it works | — | — | ## How It Works<br>This workflow automates cloud cost intelligence and governance... (full text in workflow) |
| Sticky Note3 | Sticky Note | Comment / dual AI analysis rationale | — | — | ## Dual AI Analysis<br>**Why**: Combines optimization insights with compliance enforcement, balancing efficiency and control simultaneously |
| Sticky Note4 | Sticky Note | Comment / routing rationale | — | — | ## Severity-Based Routing<br>**Why**: Ensures urgent overruns reach teams instantly while maintaining comprehensive audit trails |
| Sticky Note5 | Sticky Note | Comment / monitoring rationale | — | — | ## Automated Cost Monitoring<br>**Why**: Continuous oversight prevents cost surprises and catches anomalies before they impact budgets significantly |
| Sticky Note6 | Sticky Note | Comment / multi-channel rationale | — | — | ## Multi-Channel Alerts<br>**Why**: Delivers context-appropriate communications enabling rapid response and informed decision-making |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Cloud Cost Intelligence and Governance Orchestration** (or your preferred name).
- Keep workflow **Inactive** until credentials and channels are validated.

2) **Add Trigger: “Daily Cloud Cost Check”**
- Node type: **Schedule Trigger**
- Configure: Daily at **08:00** (adjust timezone/instance settings to match billing refresh).

3) **Add “Workflow Configuration” (Set node)**
- Node type: **Set**
- Add fields:
  - `costThresholdCritical` (Number) = `10000`
  - `costThresholdHigh` (Number) = `5000`
  - `slackChannelCritical` (String) = your Slack channel ID (e.g., `C0123...`)
  - `slackChannelHigh` (String) = your Slack channel ID
  - `financeEmail` (String) = finance distribution email
- Enable: **Include Other Fields**
- Connect: **Daily Cloud Cost Check → Workflow Configuration**

4) **Add spend input node**
- For this exact workflow, add **Simulate Cloud Spend Data** (Set node):
  - `cloudProvider` (String) = `AWS`
  - `totalSpend` (Number) = `12500`
  - `previousMonthSpend` (Number) = `8000`
  - `budgetLimit` (Number) = `10000`
  - `topServices` (Array) = list of objects `{service,cost,trend}`
  - `unusedResources` (Array) = list of objects `{resourceId,type,monthlyCost}`
  - `timestamp` (String) = `{{$now.toISO()}}`
  - Enable: **Include Other Fields**
- Connect: **Workflow Configuration → Simulate Cloud Spend Data**
- (Production replacement) Swap this node for HTTP/AWS/Azure/GCP billing nodes that output the same fields.

5) **Add OpenAI model node for cost analysis**
- Node type: **OpenAI Chat Model** (LangChain) / `lmChatOpenAi`
- Name: **OpenAI Model - Cost Intelligence**
- Model: **gpt-4o**
- Temperature: **0.1**
- Credentials: configure **OpenAI API** credential in n8n and select it here.

6) **Add “Cost Analysis Output Parser”**
- Node type: **Structured Output Parser**
- Schema: paste/define the manual JSON schema with required fields:
  - `totalSpend` (number), `budgetStatus` (string), `spendTrend` (string), `percentageChange` (number),
  - `topCostDrivers` (array of objects), `optimizationOpportunities` (array of objects),
  - `riskLevel` (string), `analysisReasoning` (string)
- Save.

7) **Add “Cost Intelligence Agent”**
- Node type: **AI Agent** (LangChain) / `agent`
- Prompt Type: **Define**
- In **Text**, format the spend input using expressions (provider, totals, JSON.stringify for arrays).
- In **System Message**, paste the cost intelligence instructions (budget status rules, risk guidelines, structured JSON requirement).
- Enable/ensure it uses:
  - **Language Model input** (connect from **OpenAI Model - Cost Intelligence**)
  - **Output Parser input** (connect from **Cost Analysis Output Parser**)
- Connect:
  - **Simulate Cloud Spend Data → Cost Intelligence Agent (main)**
  - **OpenAI Model - Cost Intelligence → Cost Intelligence Agent (ai_languageModel)**
  - **Cost Analysis Output Parser → Cost Intelligence Agent (ai_outputParser)**

8) **Add OpenAI model node for governance**
- Node type: **OpenAI Chat Model** (`lmChatOpenAi`)
- Name: **OpenAI Model - Governance**
- Model: **gpt-4o**
- Temperature: **0.1**
- Credentials: same OpenAI credential (or another, if separated).

9) **Add “Governance Output Parser”**
- Node type: **Structured Output Parser**
- Schema requiring:
  - `severity` (string), `requiresFinanceEscalation` (boolean), `alertMessage` (string),
  - `recommendedActions` (array of strings), `governanceDecision` (string)
  - optional: `escalationReason` (string)

10) **Add “Governance Agent”**
- Node type: **AI Agent** (`agent`)
- Text should include:
  - `{{ JSON.stringify($json.output, null, 2) }}` for cost intelligence
  - thresholds referencing the Set node by name:
    - `{{ $('Workflow Configuration').first().json.costThresholdCritical }}`
    - `{{ $('Workflow Configuration').first().json.costThresholdHigh }}`
- System message: paste governance rules + alert formatting instructions + JSON requirement.
- Connect:
  - **Cost Intelligence Agent → Governance Agent (main)**
  - **OpenAI Model - Governance → Governance Agent (ai_languageModel)**
  - **Governance Output Parser → Governance Agent (ai_outputParser)**

11) **Add “Log Cost Analysis”**
- Node type: **Data Table**
- Data Table name: `cloud_cost_analysis_log` (create it in n8n Data Tables if it doesn’t exist).
- Mapping mode: Auto-map input.
- Connect: **Governance Agent → Log Cost Analysis**

12) **Add “Route by Severity”**
- Node type: **Switch**
- Add rules:
  - If `{{$json.severity}} equals "critical"` → output “Critical”
  - If `{{$json.severity}} equals "high"` → output “High”
  - If `{{$json.requiresFinanceEscalation}} equals true` → output “Finance Escalation”
- Enable: ignore case; fallback output renamed “Low Priority”.
- Connect: **Governance Agent → Route by Severity**

13) **Add Slack nodes**
- Node type: **Slack**
- Create **Slack OAuth2** credential with scopes allowing posting to channels (commonly `chat:write`, plus channel access).
- **Slack Alert - Critical**
  - Post to channel ID: `{{ $('Workflow Configuration').first().json.slackChannelCritical }}`
  - Text: use `{{$json.output.alertMessage}}`, plus formatted recommended actions, plus governance decision.
- **Slack Alert - High**
  - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannelHigh }}`
  - Text: alert message + formatted actions.
- Connect **each Slack node to the correct Switch output** (recommended):
  - Switch “Critical” output → Slack Alert - Critical
  - Switch “High” output → Slack Alert - High

14) **Add finance email node**
- Node type: **Email Send**
- Configure email transport (SMTP) in n8n as required by your deployment.
- Set:
  - To: `{{ $('Workflow Configuration').first().json.financeEmail }}`
  - From: a valid sender address for your domain
  - Subject: `Cloud Cost Governance Alert - {{ $json.output.severity.toUpperCase() }} Severity`
  - HTML: include alertMessage, escalationReason fallback, recommendedActions list, governanceDecision.
- Connect: Switch “Finance Escalation” output → **Email Finance Team**

15) **Test**
- Manually execute with simulated data.
- Validate:
  - Cost agent output parses successfully
  - Governance agent output parses successfully
  - Switch routes only the intended path
  - Slack posts to correct channels
  - Email sends successfully
  - Data Table record is created

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Cloud provider API access (AWS/Azure/GCP billing), OpenAI API account | Prerequisites (Sticky Note) |
| Multi-cloud cost optimization, budget compliance enforcement | Use cases (Sticky Note) |
| Modify AI prompts for company-specific cost policies; adjust severity thresholds for alerts | Customization (Sticky Note) |
| Prevents budget overruns through proactive alerts, reduces cloud waste by 30–50% | Benefits (Sticky Note) |
| Setup steps include aligning schedule with billing updates, configuring Slack and email credentials | Setup Steps (Sticky Note1) |
| Workflow concept: daily checks → dual AI agents → severity routing → Slack/email → audit log | How It Works (Sticky Note2) |

If you want, I can propose a production-grade replacement for “Simulate Cloud Spend Data” for AWS Cost Explorer / Azure Cost Management / GCP Billing Export while keeping the downstream AI and routing unchanged.