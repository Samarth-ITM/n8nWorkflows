Score telematics driving risk with Claude and adjust insurance premiums via HTTP, Gmail, and Slack

https://n8nworkflows.xyz/workflows/score-telematics-driving-risk-with-claude-and-adjust-insurance-premiums-via-http--gmail--and-slack-12279


# Score telematics driving risk with Claude and adjust insurance premiums via HTTP, Gmail, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Score telematics driving risk with Claude and adjust insurance premiums via HTTP, Gmail, and Slack  
**Workflow name (in JSON):** Smart Telematics Behavioral Risk Scoring and Premium Adjustment Engine

**Purpose:**  
This workflow runs on a schedule, fetches telematics driving behavior data via HTTP, uses Anthropic Claude (via the LangChain Agent node) to compute a structured driving risk score (0–100), calculates an insurance premium adjustment, updates the premium via an HTTP API, and—if the risk is high—sends alerts via Gmail and Slack. Finally, it syncs the outcome to an underwriting rules system.

**Target use cases:**
- Usage-based insurance (UBI) programs automating underwriting adjustments
- Continuous driver risk monitoring and proactive intervention
- Reducing manual review of telematics patterns (speeding, harsh braking, etc.)

### 1.1 Scheduling & Configuration
- Scheduled trigger (daily and weekly variant)
- Centralized configuration (API URLs, thresholds, alert targets)

### 1.2 Data Ingestion (Telematics)
- HTTP fetch from telematics platform endpoint

### 1.3 AI Risk Scoring (Claude + Structured Parsing)
- LangChain Agent prompts Claude with telematics payload
- Structured output enforced via JSON schema parser

### 1.4 Premium Adjustment & Policy System Update
- Compute adjustment % and multiplier from risk score
- POST update to premium system API

### 1.5 High-Risk Alerting & Underwriting Sync
- IF gate for high risk threshold
- Gmail + Slack notifications for high-risk cases
- POST sync to underwriting rules system

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Workflow Configuration
**Overview:** Triggers the workflow periodically and sets all configurable runtime parameters (API endpoints, thresholds, alert targets) in one place for reuse via expressions.

**Nodes involved:**
- Daily/Weekly Schedule
- Workflow Configuration

#### Node: Daily/Weekly Schedule
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):**
  - Runs on an interval rule set containing:
    - A daily trigger at **02:00**
    - A weekly trigger on **day 1** (typically Monday in n8n schedule semantics) at **02:00**
- **Inputs:** None (trigger node).
- **Outputs / connections:**
  - Main → **Workflow Configuration**
- **Version notes:** typeVersion **1.3**.
- **Edge cases / failures:**
  - Timezone implications depend on n8n instance timezone setting.
  - Overlapping executions if runtime exceeds schedule frequency (consider concurrency settings).

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines reusable config variables.
- **Configuration (interpreted):**
  - Adds fields (while keeping existing fields): `includeOtherFields: true`
  - Creates:
    - `telematicsApiUrl` (placeholder)
    - `premiumApiUrl` (placeholder)
    - `underwritingApiUrl` (placeholder)
    - `riskThreshold` = **75**
    - `alertEmail` (placeholder)
    - `slackChannel` (placeholder channel ID)
- **Key expressions / variables used elsewhere:**
  - Referenced via `$('Workflow Configuration').first().json.<field>`
- **Inputs:** From Schedule Trigger.
- **Outputs / connections:**
  - Main → **Fetch Telematics Data**
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**
  - If placeholders are not replaced, downstream HTTP and notifications will fail (invalid URL/email/channel).
  - `riskThreshold` must be numeric for the IF comparison (it is set as number here).

---

### Block 2 — Telematics Data Ingestion
**Overview:** Pulls driving/vehicle behavior data from a telematics platform via HTTP and passes the JSON to the AI scoring agent.

**Nodes involved:**
- Fetch Telematics Data

#### Node: Fetch Telematics Data
- **Type / role:** `HTTP Request` — retrieves telematics data.
- **Configuration (interpreted):**
  - URL: `={{ $('Workflow Configuration').first().json.telematicsApiUrl }}`
  - Response format: JSON
  - Sends `Authorization` header with placeholder token
- **Inputs:** From Workflow Configuration.
- **Outputs / connections:**
  - Main → **Risk Scoring AI Agent**
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - 401/403 if token invalid or missing.
  - Non-JSON responses will break downstream assumptions (agent receives raw/unexpected data).
  - API pagination or batching not handled—if the endpoint returns multiple drivers/vehicles, ensure the payload structure matches what the agent expects.

---

### Block 3 — AI Risk Scoring (Claude + Structured Output)
**Overview:** Uses a LangChain Agent powered by Anthropic Claude to analyze telematics data and produce a structured risk assessment enforced by a schema.

**Nodes involved:**
- Risk Scoring AI Agent
- Anthropic Chat Model
- Risk Score Output Parser

#### Node: Anthropic Chat Model
- **Type / role:** `LangChain Chat Model (Anthropic)` — provides Claude model to the agent.
- **Configuration (interpreted):**
  - Model: `claude-3-5-sonnet-20241022`
  - Credentials: **Anthropic account** (API key in n8n credentials store)
- **Inputs / outputs:**
  - Connected to **Risk Scoring AI Agent** via `ai_languageModel`
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**
  - Credential missing/invalid → auth errors.
  - Model availability/region restrictions or quota limits.
  - Latency/timeouts on large telematics payloads.

#### Node: Risk Score Output Parser
- **Type / role:** `Structured Output Parser` — enforces JSON schema for the agent’s final output.
- **Configuration (interpreted):**
  - Manual JSON schema requiring an object with properties:
    - `riskScore` (number)
    - `riskLevel` (string: low/moderate/elevated/high per prompt)
    - `riskyBehaviors` (array of strings)
    - `reasoning` (string)
    - `recommendations` (array of strings)
    - `vehicleId` (string)
    - `driverId` (string)
- **Inputs / outputs:**
  - Connected to **Risk Scoring AI Agent** via `ai_outputParser`
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**
  - If Claude returns non-conforming JSON, parsing fails and downstream nodes won’t run.
  - Schema does not specify “required” fields; however, downstream expressions assume these fields exist (risk of undefined values).

#### Node: Risk Scoring AI Agent
- **Type / role:** `LangChain Agent` — orchestrates prompt + model + parser to produce structured risk scoring.
- **Configuration (interpreted):**
  - Input text: `={{ $json }}` (passes the entire telematics JSON payload as the prompt input)
  - System message instructs:
    - Analyze telematics behavior (speed, braking, cornering, acceleration)
    - Output risk score 0–100 and bucket it into levels
    - Provide risky behaviors, reasoning, recommendations
  - Prompt type: “define”
  - Output parser enabled (wired to Risk Score Output Parser)
  - Language model provided by Anthropic Chat Model
- **Inputs:** From Fetch Telematics Data.
- **Outputs / connections:**
  - Main → **Calculate Premium Adjustment**
- **Version notes:** typeVersion **3.1**
- **Edge cases / failures:**
  - If telematics data is too large, token limits may truncate important details.
  - If telematics payload lacks `vehicleId`/`driverId`, the model might hallucinate or omit—breaking premium update and alerts.
  - Inconsistent risk level casing (e.g., “High” vs “high”)—downstream uses `.toUpperCase()` for email, which is safe, but comparisons are numeric only.

---

### Block 4 — Premium Adjustment Calculation & Update
**Overview:** Converts the risk score into a premium adjustment and pushes the change to the premium/policy system via HTTP.

**Nodes involved:**
- Calculate Premium Adjustment
- Update Premium in System

#### Node: Calculate Premium Adjustment
- **Type / role:** `Set` — computes premium adjustment fields from AI output.
- **Configuration (interpreted):**
  - Keeps original fields (`includeOtherFields: true`) and adds:
    - `premiumAdjustmentPercent`:
      - `-10` if `riskScore <= 25`
      - `0` if `riskScore <= 50`
      - `15` if `riskScore <= 75`
      - `30` otherwise
    - `newPremiumMultiplier` = `1 + (premiumAdjustmentPercent/100)`
    - `adjustmentReason` = `"Risk score: <score> (<level>)"`
    - `effectiveDate` = current timestamp ISO (`$now.toISO()`)
- **Key expressions:**
  - Ternary ladder on `$json.riskScore`
- **Inputs:** From Risk Scoring AI Agent.
- **Outputs / connections:**
  - Main → **Update Premium in System**
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**
  - If `riskScore` is missing or non-numeric, expressions may evaluate unexpectedly (NaN comparisons).
  - If `riskScore` is outside 0–100, logic still applies but may be semantically invalid—consider clamping.

#### Node: Update Premium in System
- **Type / role:** `HTTP Request` — writes premium adjustments to policy/premium system.
- **Configuration (interpreted):**
  - URL: `={{ $('Workflow Configuration').first().json.premiumApiUrl }}`
  - Method: POST
  - JSON body includes:
    - `vehicleId`, `driverId`, `riskScore`
    - `premiumAdjustment` (percent)
    - `newMultiplier`
    - `reason`, `effectiveDate`
  - Headers:
    - `Authorization`: placeholder token
    - `Content-Type: application/json`
- **Inputs:** From Calculate Premium Adjustment.
- **Outputs / connections:**
  - Main → **Check for High Risk**
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - Premium API may require idempotency keys; not implemented.
  - If the API expects different field names/types, update fails (400).
  - Network timeouts or 5xx errors should be handled with retries (not configured here).

---

### Block 5 — High-Risk Handling, Notifications, and Underwriting Sync
**Overview:** After premium update, checks if the risk score meets/exceeds the configured threshold. If high risk, sends Gmail and Slack alerts, then syncs results to underwriting rules.

**Nodes involved:**
- Check for High Risk
- Send Risk Alert Email
- Send Slack Notification
- Sync with Underwriting Rules

#### Node: Check for High Risk
- **Type / role:** `IF` — conditional gate for high-risk cases.
- **Configuration (interpreted):**
  - Condition: `riskScore >= riskThreshold`
    - Left: `={{ $('Calculate Premium Adjustment').item.json.riskScore }}`
    - Right: `={{ $('Workflow Configuration').first().json.riskThreshold }}`
- **Inputs:** From Update Premium in System.
- **Outputs / connections:**
  - **True** path (index 0) → **Send Risk Alert Email** and **Send Slack Notification** (fan-out)
  - False path is not connected (no action for non-high-risk cases)
- **Version notes:** typeVersion **2.3**
- **Edge cases / failures:**
  - Uses `typeValidation: loose`; still, missing riskScore may cause unexpected comparisons.
  - Since false branch is unused, underwriting sync happens only for high-risk (as wired). If you intended sync for all cases, connection must be adjusted.

#### Node: Send Risk Alert Email
- **Type / role:** `Gmail` — sends detailed high-risk email.
- **Configuration (interpreted):**
  - To: config `alertEmail`
  - Subject includes driverId and riskScore
  - Body includes:
    - Driver/Vehicle IDs
    - Risk score & uppercase risk level
    - Risky behaviors list
    - Reasoning text
    - Recommendations list
    - Premium adjustment percent and effective date
- **Key expressions / assumptions:**
  - `$json.riskyBehaviors.join(...)` and `$json.recommendations.join(...)` assume arrays exist.
  - Risk level string supports `.toUpperCase()`.
- **Inputs:** From IF (true branch).
- **Outputs / connections:**
  - Main → **Sync with Underwriting Rules**
- **Credentials:** Gmail OAuth2
- **Version notes:** typeVersion **2.2**
- **Edge cases / failures:**
  - OAuth token expiration / permission issues.
  - If `riskyBehaviors` is empty/undefined, `.join()` fails—consider defaulting to `[]`.
  - Email deliverability depends on Gmail account limits and policies.

#### Node: Send Slack Notification
- **Type / role:** `Slack` — posts high-risk alert to a channel.
- **Configuration (interpreted):**
  - OAuth2 authentication
  - Channel: `={{ $('Workflow Configuration').first().json.slackChannel }}`
  - Message highlights: driver, vehicle, score/level, premium adjustment, top 3 risky behaviors
  - Message text includes a `:warning:` token (Slack renders it if allowed)
- **Inputs:** From IF (true branch).
- **Outputs / connections:**
  - Main → **Sync with Underwriting Rules**
- **Credentials:** Slack OAuth2
- **Version notes:** typeVersion **2.4**
- **Edge cases / failures:**
  - Channel ID must be valid for the connected workspace.
  - If `riskyBehaviors` is missing, `.slice()` fails.
  - Slack rate limits or posting restrictions.

#### Node: Sync with Underwriting Rules
- **Type / role:** `HTTP Request` — syncs high-risk outcome to underwriting rules/compliance system.
- **Configuration (interpreted):**
  - URL: `={{ $('Workflow Configuration').first().json.underwritingApiUrl }}`
  - Method: POST
  - JSON body includes:
    - `vehicleId`, `driverId`, `riskScore`, `riskLevel`, `riskyBehaviors`
    - `premiumAdjustment`
    - `syncTimestamp` = `$now.toISO()`
  - Headers: Authorization placeholder + JSON content-type
- **Inputs:** From **Send Risk Alert Email** and from **Send Slack Notification** (two inbound connections).
- **Outputs:** None configured.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - This node may execute **twice** for the same high-risk event (once after Gmail, once after Slack), because both feed into it independently. If the underwriting API is not idempotent, this can create duplicate sync records.
  - Consider merging the alert branches (e.g., Merge node) before syncing, or connect sync directly from IF true branch.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily/Weekly Schedule | scheduleTrigger | Scheduled entry point | — | Workflow Configuration | ## **Document Analysis & Risk Scoring** / **What:** Schedule triggers periodic execution; HTTP node fetches telematics data. / **Why:** AI detects subtle risk patterns missed by rule-based systems, and automation |
| Workflow Configuration | set | Central config variables (URLs, thresholds, targets) | Daily/Weekly Schedule | Fetch Telematics Data | ## Setup Steps / 1. Configure Schedule node for desired analysis frequency / 2. Set up HTTP node with telematics platform API / 3. Add Anthropic API key to Chat Model node for behavioral risk analysis / 4. Connect policy management system API credentials in HTTP nodes / 5. Integrate Gmail and Slack with underwriting team addresses |
| Fetch Telematics Data | httpRequest | Pull telematics JSON via API | Workflow Configuration | Risk Scoring AI Agent | ## **Document Analysis & Risk Scoring** / **What:** Schedule triggers periodic execution; HTTP node fetches telematics data. / **Why:** AI detects subtle risk patterns missed by rule-based systems, and automation |
| Risk Scoring AI Agent | @n8n/n8n-nodes-langchain.agent | AI orchestration: prompt + model + structured output | Fetch Telematics Data | Calculate Premium Adjustment | ## How It Works / This workflow automates insurance premium adjustments by analyzing telematics data with AI-driven risk assessment and syncing changes across underwriting systems... |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model provider for agent | — (model connection) | Risk Scoring AI Agent | ## Prerequisites / Anthropic API key, telematics data platform API access / ## Use Cases / Auto insurance carriers implementing usage-based insurance programs / ## Customization / Modify AI prompts to incorporate additional risk factors like weather conditions / ## Benefits / Reduces premium calculation time from days to minutes |
| Risk Score Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce schema for AI result | — (parser connection) | Risk Scoring AI Agent | ## Prerequisites / Anthropic API key, telematics data platform API access / ## Use Cases / Auto insurance carriers implementing usage-based insurance programs / ## Customization / Modify AI prompts to incorporate additional risk factors like weather conditions / ## Benefits / Reduces premium calculation time from days to minutes |
| Calculate Premium Adjustment | set | Compute adjustment %, multiplier, reason, effective date | Risk Scoring AI Agent | Update Premium in System | ## **Premium Update** / **What:** Calculator node applies risk scores to premium formulas; / **Why:** Automated calculations eliminate manual errors . |
| Update Premium in System | httpRequest | POST premium update to policy system | Calculate Premium Adjustment | Check for High Risk | ## **Premium Update** / **What:** Calculator node applies risk scores to premium formulas; / **Why:** Automated calculations eliminate manual errors . |
| Check for High Risk | if | Gate for high-risk threshold | Update Premium in System | Send Risk Alert Email; Send Slack Notification | ## **High-Risk Handling** / **What:** IF node flags high-risk drivers, triggering Gmail alerts to underwriters and Slack / **Why:** Proactive flagging enables early intervention, while synchronized rule compliance |
| Send Risk Alert Email | gmail | Email alert for high-risk drivers | Check for High Risk (true) | Sync with Underwriting Rules | ## **High-Risk Handling** / **What:** IF node flags high-risk drivers, triggering Gmail alerts to underwriters and Slack / **Why:** Proactive flagging enables early intervention, while synchronized rule compliance |
| Send Slack Notification | slack | Slack alert for high-risk drivers | Check for High Risk (true) | Sync with Underwriting Rules | ## **High-Risk Handling** / **What:** IF node flags high-risk drivers, triggering Gmail alerts to underwriters and Slack / **Why:** Proactive flagging enables early intervention, while synchronized rule compliance |
| Sync with Underwriting Rules | httpRequest | Sync outcome to underwriting rules/compliance system | Send Risk Alert Email; Send Slack Notification | — | ## **High-Risk Handling** / **What:** IF node flags high-risk drivers, triggering Gmail alerts to underwriters and Slack / **Why:** Proactive flagging enables early intervention, while synchronized rule compliance |
| Sticky Note | stickyNote | Commentary | — | — | ## Prerequisites / Anthropic API key, telematics data platform API access / ## Use Cases / Auto insurance carriers implementing usage-based insurance programs / ## Customization / Modify AI prompts to incorporate additional risk factors like weather conditions / ## Benefits / Reduces premium calculation time from days to minutes |
| Sticky Note1 | stickyNote | Commentary | — | — | ## Setup Steps / 1. Configure Schedule node for desired analysis frequency / 2. Set up HTTP node with telematics platform API / 3. Add Anthropic API key to Chat Model node for behavioral risk analysis / 4. Connect policy management system API credentials in HTTP nodes / 5. Integrate Gmail and Slack with underwriting team addresses |
| Sticky Note2 | stickyNote | Commentary | — | — | ## How It Works / This workflow automates insurance premium adjustments by analyzing telematics data with AI-driven risk assessment and syncing changes across underwriting systems... |
| Sticky Note3 | stickyNote | Commentary | — | — | ## **High-Risk Handling** / **What:** IF node flags high-risk drivers, triggering Gmail alerts to underwriters and Slack / **Why:** Proactive flagging enables early intervention, while synchronized rule compliance |
| Sticky Note4 | stickyNote | Commentary | — | — | ## **Premium Update** / **What:** Calculator node applies risk scores to premium formulas; / **Why:** Automated calculations eliminate manual errors . |
| Sticky Note5 | stickyNote | Commentary | — | — | ## **Document Analysis & Risk Scoring** / **What:** Schedule triggers periodic execution; HTTP node fetches telematics data. / **Why:** AI detects subtle risk patterns missed by rule-based systems, and automation |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Smart Telematics Behavioral Risk Scoring and Premium Adjustment Engine** (or your preferred name).

2. **Add Trigger: “Schedule Trigger”**
   - Node name: **Daily/Weekly Schedule**
   - Configure two rules:
     - Daily at **02:00**
     - Weekly on **day 1** at **02:00**
   - Save.

3. **Add “Set” node for configuration**
   - Node name: **Workflow Configuration**
   - Enable **Include Other Fields**
   - Add fields:
     - `telematicsApiUrl` (string): your telematics API endpoint
     - `premiumApiUrl` (string): your premium/policy system endpoint
     - `underwritingApiUrl` (string): underwriting rules endpoint
     - `riskThreshold` (number): **75**
     - `alertEmail` (string): recipient email
     - `slackChannel` (string): Slack channel ID
   - Connect: **Daily/Weekly Schedule → Workflow Configuration**

4. **Add “HTTP Request” to fetch telematics**
   - Node name: **Fetch Telematics Data**
   - URL expression: `$('Workflow Configuration').first().json.telematicsApiUrl`
   - Response format: **JSON**
   - Add header `Authorization: <your token or expression>`
   - Connect: **Workflow Configuration → Fetch Telematics Data**

5. **Add Anthropic model node**
   - Node name: **Anthropic Chat Model**
   - Select model: **claude-3-5-sonnet-20241022**
   - Create/select **Anthropic API credentials** in n8n (API key)
   - This node will be connected to the agent via the AI model connector (not main flow).

6. **Add “Structured Output Parser”**
   - Node name: **Risk Score Output Parser**
   - Schema type: **Manual**
   - Paste/define a schema with fields:
     - `riskScore` (number), `riskLevel` (string),
     - `riskyBehaviors` (string array),
     - `reasoning` (string),
     - `recommendations` (string array),
     - `vehicleId` (string), `driverId` (string)

7. **Add “AI Agent” (LangChain Agent)**
   - Node name: **Risk Scoring AI Agent**
   - Text input: `{{$json}}`
   - System message: instruct analysis and require the structured fields (risk score 0–100, categories, behaviors, reasoning, recommendations).
   - Enable structured output (output parser).
   - Connect AI ports:
     - **Anthropic Chat Model → Risk Scoring AI Agent** (language model connection)
     - **Risk Score Output Parser → Risk Scoring AI Agent** (output parser connection)
   - Connect main flow: **Fetch Telematics Data → Risk Scoring AI Agent**

8. **Add “Set” node to compute premium adjustment**
   - Node name: **Calculate Premium Adjustment**
   - Enable **Include Other Fields**
   - Add fields:
     - `premiumAdjustmentPercent` (number expression):  
       `{{$json.riskScore <= 25 ? -10 : $json.riskScore <= 50 ? 0 : $json.riskScore <= 75 ? 15 : 30}}`
     - `newPremiumMultiplier` (number): `{{1 + ($json.premiumAdjustmentPercent / 100)}}`
     - `adjustmentReason` (string): `{{'Risk score: ' + $json.riskScore + ' (' + $json.riskLevel + ')'}}`
     - `effectiveDate` (string): `{{$now.toISO()}}`
   - Connect: **Risk Scoring AI Agent → Calculate Premium Adjustment**

9. **Add “HTTP Request” to update premium system**
   - Node name: **Update Premium in System**
   - Method: **POST**
   - URL: `{{$('Workflow Configuration').first().json.premiumApiUrl}}`
   - Body type: **JSON**
   - JSON body expression containing:
     - `vehicleId`, `driverId`, `riskScore`,
     - `premiumAdjustment`, `newMultiplier`,
     - `reason`, `effectiveDate`
   - Headers:
     - `Authorization: <premium system token>`
     - `Content-Type: application/json`
   - Connect: **Calculate Premium Adjustment → Update Premium in System**

10. **Add “IF” node for high-risk check**
    - Node name: **Check for High Risk**
    - Condition (Number):  
      Left: `{{$('Calculate Premium Adjustment').item.json.riskScore}}`  
      Operator: **>=**  
      Right: `{{$('Workflow Configuration').first().json.riskThreshold}}`
    - Connect: **Update Premium in System → Check for High Risk**

11. **Add Gmail node for email alert**
    - Node name: **Send Risk Alert Email**
    - Configure Gmail OAuth2 credentials in n8n.
    - To: `{{$('Workflow Configuration').first().json.alertEmail}}`
    - Subject/body: include driverId, vehicleId, score, behaviors, reasoning, recommendations, premium adjustment, effective date (as per your preferred formatting).
    - Connect: **Check for High Risk (true) → Send Risk Alert Email**

12. **Add Slack node for Slack alert**
    - Node name: **Send Slack Notification**
    - Configure Slack OAuth2 credentials in n8n.
    - Post to channel ID: `{{$('Workflow Configuration').first().json.slackChannel}}`
    - Message: include driverId, vehicleId, score/level, adjustment, top behaviors.
    - Connect: **Check for High Risk (true) → Send Slack Notification**

13. **Add “HTTP Request” to sync underwriting rules**
    - Node name: **Sync with Underwriting Rules**
    - Method: **POST**
    - URL: `{{$('Workflow Configuration').first().json.underwritingApiUrl}}`
    - Body: JSON with vehicleId/driverId/riskScore/riskLevel/riskyBehaviors/premiumAdjustment + `syncTimestamp: $now.toISO()`
    - Headers: Authorization + JSON content type
    - Connect:
      - **Send Risk Alert Email → Sync with Underwriting Rules**
      - **Send Slack Notification → Sync with Underwriting Rules**
    - Important: this wiring causes **two sync calls** per high-risk case (see notes above). If undesired, insert a Merge node or sync directly from the IF node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: Anthropic API key, telematics data platform API access | Sticky note “Prerequisites” |
| Use case: auto insurance carriers implementing usage-based insurance programs | Sticky note “Use Cases” |
| Customization: expand AI prompts with additional risk factors (e.g., weather) | Sticky note “Customization” |
| Benefit: reduces premium calculation time from days to minutes | Sticky note “Benefits” |
| High-risk handling triggers both Gmail and Slack, then syncs underwriting | Sticky note “High-Risk Handling” |
| Current wiring may sync underwriting twice per high-risk event (once after Gmail, once after Slack) | Integration/behavior note (derived from connections) |
| Non-high-risk cases do not sync underwriting (false branch unused) | Integration/behavior note (derived from connections) |