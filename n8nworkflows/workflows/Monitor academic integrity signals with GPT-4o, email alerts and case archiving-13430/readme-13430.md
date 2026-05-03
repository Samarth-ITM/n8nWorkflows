Monitor academic integrity signals with GPT-4o, email alerts and case archiving

https://n8nworkflows.xyz/workflows/monitor-academic-integrity-signals-with-gpt-4o--email-alerts-and-case-archiving-13430


# Monitor academic integrity signals with GPT-4o, email alerts and case archiving

## 1. Workflow Overview

**Purpose:** This workflow runs on a schedule to assess student submissions for potential academic integrity issues using GPT‑4o, routes cases by risk level, performs deeper AI-led investigation for non-low risk cases, and then either (a) queues them for human review with an email alert or (b) processes them automatically. Finally, it merges and archives all outcomes for auditability.

**Primary use cases:**
- Daily integrity monitoring (plagiarism + behavioral anomaly signals)
- Triage into low-risk “monitor only” vs. cases requiring investigation
- Human-in-the-loop decision gating for sensitive cases
- Central archiving of all case outcomes

### 1.1 Scheduled Initiation & Global Configuration
A daily trigger starts the run and sets global thresholds and reviewer contact settings.

### 1.2 Assessment Data Ingestion (Simulated)
Creates a synthetic “submission” payload (student, assessment, text, timing/behavior metrics).

### 1.3 AI Integrity Signal Detection (GPT‑4o + Structured Output)
An AI agent scores plagiarism/anomalies, assigns a risk level, and returns structured signals.

### 1.4 Risk Routing
Switches the flow into Low vs. Medium vs. High/Critical paths.

### 1.5 Orchestrated Investigation & Case Decisioning (GPT‑4o + Tool Call)
For Medium/High/Critical, an orchestration agent calls an investigation tool agent, synthesizes findings, and decides whether human review is required.

### 1.6 Human Review Queue + Email Alert + Webhook Wait
If human review is required, the case is stored in a queue table and an email is sent with links to a webhook-based “decision” endpoint; the workflow then waits for the webhook callback.

### 1.7 Automated Case Storage (No Human Review)
If human review is not required, the case is stored as automatically processed.

### 1.8 Merge & Archive
All branches converge, then the workflow archives cases into a central table.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduled Initiation & Global Configuration
**Overview:** Starts the workflow daily and defines run-wide parameters (email + thresholds) used later in AI prompts and routing.

**Nodes involved:**
- Schedule Trigger
- Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — entry point; runs on a time schedule.
- **Configuration:** Interval schedule configured to trigger at **09:00** (triggerAtHour: 9). (Minute not specified; n8n defaults apply.)
- **Outputs:** Connects to **Workflow Configuration**.
- **Potential failures / edge cases:**
  - Timezone ambiguity: schedule runs in the n8n instance timezone; ensure it matches operational needs.
  - If workflows overlap (long execution time), consider concurrency settings.

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines constants used across the workflow.
- **Key fields set:**
  - `reviewerEmail`: placeholder `<__PLACEHOLDER_VALUE__Academic Integrity Reviewer Email__>`
  - `plagiarismThreshold`: `75`
  - `anomalyThreshold`: `80`
  - `highRiskThreshold`: `85` (not actually used in routing logic as implemented; see notes below)
- **Configuration choice:** `includeOtherFields: true` (passes through incoming fields if any).
- **Outputs:** Connects to **Simulate Assessment Data**.
- **Edge cases:**
  - Missing/invalid `reviewerEmail` causes email send failures downstream.
  - `highRiskThreshold` is unused; can confuse maintainers.

---

### Block 2.2 — Assessment Data Ingestion (Simulated)
**Overview:** Generates a synthetic assessment submission payload used to demonstrate the integrity analysis pipeline.

**Nodes involved:**
- Simulate Assessment Data

#### Node: Simulate Assessment Data
- **Type / role:** `Set` — generates mock submission/behavioral metrics.
- **Key expressions/fields:**
  - `studentId`: `STU-2024-{{ Math.floor(Math.random() * 10000) }}`
  - `assessmentId`: `ASSESS-{{ Math.floor(Math.random() * 1000) }}`
  - `submissionText`: static sample text
  - `submissionTimestamp`: `{{ $now.toISO() }}`
  - `previousSubmissions`: `5`
  - `averageGrade`: `78.5`
  - `timeSpentMinutes`: `45`
  - `wordCount`: `850`
  - `browserFingerprint`: `FP-{{ Math.floor(Math.random() * 100000) }}`
- **Outputs:** Connects to **Integrity Signal Agent**.
- **Edge cases:**
  - Because this is synthetic, signals may not be stable run-to-run; random IDs/fingerprints can affect anomaly narratives.
  - In a real implementation, this block would be replaced by LMS/assessment platform ingestion (API, DB, webhook).

---

### Block 2.3 — AI Integrity Signal Detection (GPT‑4o + Structured Output)
**Overview:** Uses GPT‑4o to analyze the submission for plagiarism indicators and behavioral anomalies, producing a structured “integrity signals” object.

**Nodes involved:**
- OpenAI Model - Integrity Agent
- Structured Output - Integrity Signals
- Integrity Signal Agent

#### Node: OpenAI Model - Integrity Agent
- **Type / role:** `lmChatOpenAi` (LangChain) — provides the chat model to the agent.
- **Model:** `gpt-4o`
- **Options:** `temperature: 0.1` (more deterministic scoring/labels)
- **Credentials:** OpenAI API credential “OpenAi account”
- **Connections:** Feeds the **Integrity Signal Agent** via `ai_languageModel`.
- **Failure modes:**
  - Invalid/expired OpenAI credential; quota/rate limit.
  - Model availability changes (ensure `gpt-4o` is accessible on the account).

#### Node: Structured Output - Integrity Signals
- **Type / role:** `outputParserStructured` — enforces JSON schema-like structured output.
- **Expected output shape (example schema):**
  - `riskLevel`: `LOW | MEDIUM | HIGH | CRITICAL`
  - `plagiarismScore` (0–100), `plagiarismIndicators` (array)
  - `anomalyScore` (0–100), `anomalyIndicators` (array)
  - `behavioralFlags` (array)
  - `overallConfidence` (0.0–1.0)
  - `reasoning` (string)
  - `recommendedAction`: `MONITOR | INVESTIGATE | ESCALATE`
- **Connections:** Feeds **Integrity Signal Agent** via `ai_outputParser`.
- **Edge cases:**
  - Model output may not match schema; parser can fail or coerce unexpectedly depending on node behavior/version.
  - Consider adding retry/fallback logic if parser errors occur.

#### Node: Integrity Signal Agent
- **Type / role:** `agent` (LangChain) — prompts the model and returns structured integrity signals.
- **Prompt content:** Builds a contextual text block using:
  - Submission fields from current item: `studentId`, `assessmentId`, `submissionText`, `submissionTimestamp`, etc.
  - Thresholds pulled from **Workflow Configuration**:
    - `$('Workflow Configuration').first().json.plagiarismThreshold`
    - `$('Workflow Configuration').first().json.anomalyThreshold`
- **System message:** Defines scoring rules and risk mapping:
  - LOW: both scores below thresholds
  - MEDIUM: one score >= threshold
  - HIGH: both >= threshold
  - CRITICAL: significantly above thresholds
- **Output:** Under `$json.output` (as referenced later), containing the parsed structured object.
- **Connections:** Main output -> **Route by Risk Level**
- **Failure modes / edge cases:**
  - If `Workflow Configuration` doesn’t run (or is renamed), expressions referencing it will fail.
  - The agent defines CRITICAL qualitatively (“significantly exceed thresholds”)—this can lead to inconsistent CRITICAL classification.
  - Threshold `highRiskThreshold` is never supplied to the agent (only plagiarism/anomaly thresholds are).

---

### Block 2.4 — Risk Routing
**Overview:** Branches execution based on `riskLevel` from the integrity signals.

**Nodes involved:**
- Route by Risk Level

#### Node: Route by Risk Level
- **Type / role:** `Switch` — conditional routing.
- **Rules (on `={{ $json.output.riskLevel }}`):**
  - Output “Low Risk”: equals `LOW`
  - Output “Medium Risk”: equals `MEDIUM`
  - Output “High/Critical Risk”: equals `HIGH` OR `CRITICAL`
- **Fallback:** `Unclassified` (renamed fallback output)
- **Connections:**
  - Low Risk -> **Prepare Low Risk Response**
  - Medium Risk -> **Case Orchestration Agent**
  - High/Critical Risk -> **Case Orchestration Agent**
- **Edge cases:**
  - If `riskLevel` missing or unexpected (e.g., “low”), route goes to **Unclassified**—but **Unclassified is not connected**, so items would stop there silently.

---

### Block 2.5 — Low Risk Handling (Store + Later Merge)
**Overview:** For LOW risk, creates a simple case record, stores it, then merges for archiving.

**Nodes involved:**
- Prepare Low Risk Response
- Store Low Risk Case

#### Node: Prepare Low Risk Response
- **Type / role:** `Set` — builds a low-risk case record.
- **Fields set:**
  - `caseId`: `CASE-LOW-{{ Math.floor(Math.random() * 10000) }}`
  - `studentId`, `assessmentId` from **Simulate Assessment Data**
  - `riskLevel`: `{{ $json.output.riskLevel }}`
  - `action`: “No action required - monitoring only”
  - `status`: “LOW_RISK_CLEARED”
  - `processedAt`: `{{ $now.toISO() }}`
- **Connections:** -> **Store Low Risk Case**
- **Edge cases:**
  - Random caseId risks collisions; use a UUID or timestamp-based ID for real systems.

#### Node: Store Low Risk Case
- **Type / role:** `Data Table` — persists low-risk case record.
- **Target table:** `LowRiskCases`
- **Mapping mode:** “defineBelow” with explicit column mapping from `$json` fields.
- **Connections:** -> **Merge All Cases** (input index 2)
- **Failure modes:**
  - Data table not created / permissions / schema mismatch.

---

### Block 2.6 — Investigation Orchestration & Decisioning (GPT‑4o + Tool)
**Overview:** For MEDIUM/HIGH/CRITICAL cases, an orchestration agent decides what to do; it can call an investigation tool agent to produce deeper evidence, then returns a structured orchestration decision.

**Nodes involved:**
- OpenAI Model - Orchestration Agent
- Case Orchestration Agent
- Structured Output - Orchestration
- Investigation Agent Tool
- OpenAI Model - Investigation Tool
- Structured Output - Investigation

#### Node: OpenAI Model - Orchestration Agent
- **Type / role:** `lmChatOpenAi` — model provider for orchestration agent.
- **Model:** `gpt-4o`
- **Options:** `temperature: 0.2`
- **Connections:** `ai_languageModel` -> **Case Orchestration Agent**
- **Failure modes:** Same OpenAI credential/quota/model availability issues.

#### Node: Structured Output - Orchestration
- **Type / role:** `outputParserStructured` — enforces structured orchestration decision.
- **Expected fields:**
  - `caseId`
  - `orchestrationDecision`: `AUTOMATED_ACTION | HUMAN_REVIEW_REQUIRED`
  - `actionsTaken` (array)
  - `investigationSummary`, `documentationGenerated`
  - `humanReviewRequired` (boolean), `humanReviewReason`
  - `recommendedSanction`: `WARNING | GRADE_REDUCTION | COURSE_FAILURE | DISCIPLINARY_ACTION`
  - `confidenceLevel` (0.0–1.0)
  - `nextSteps` (array)
- **Connections:** `ai_outputParser` -> **Case Orchestration Agent**
- **Edge cases:** Parser/schema mismatch if the agent returns inconsistent booleans/fields.

#### Node: Investigation Agent Tool
- **Type / role:** `agentTool` — a callable tool the orchestration agent can invoke.
- **Input mapping:** `text: {{ $fromAI("caseData", "Academic integrity case data requiring investigation", "json") }}`
  - This expects the orchestration agent to pass a JSON object argument named `caseData`.
- **System message:** Simulates deeper checks: sources, timeline, style comparison; returns structured results.
- **Tool description:** “Conducts detailed investigation…”
- **Connections:**
  - Receives model via `ai_languageModel` from **OpenAI Model - Investigation Tool**
  - Receives parser via `ai_outputParser` from **Structured Output - Investigation**
  - Exposed to **Case Orchestration Agent** via `ai_tool`
- **Failure modes / edge cases:**
  - If the orchestration agent calls the tool with wrong argument name/type, `$fromAI` extraction can fail.
  - Tool may be invoked for medium/high/critical per instructions, but this is not technically enforced—depends on agent behavior.

#### Node: OpenAI Model - Investigation Tool
- **Type / role:** `lmChatOpenAi` — model provider for the tool agent.
- **Model/options:** `gpt-4o`, `temperature: 0.1`
- **Connections:** `ai_languageModel` -> **Investigation Agent Tool**

#### Node: Structured Output - Investigation
- **Type / role:** `outputParserStructured` — enforces structured investigation results.
- **Expected fields:**
  - `investigationFindings`
  - `evidenceStrength`: `WEAK | MODERATE | STRONG | CONCLUSIVE`
  - `sourcesChecked` (array), `matchPercentage` (number)
  - `timelineAnalysis`, `comparisonWithPreviousWork`
  - `recommendedNextSteps` (array)
- **Connections:** `ai_outputParser` -> **Investigation Agent Tool**

#### Node: Case Orchestration Agent
- **Type / role:** `agent` — synthesizes integrity signals + investigation results and decides on human review.
- **Prompt text includes:**
  - `{{ $json.output.riskLevel }}`, `plagiarismScore`, `anomalyScore`, `recommendedAction`, `reasoning`
  - Student + assessment IDs from **Simulate Assessment Data**
- **System message rules:**
  - Call the investigation tool for MEDIUM/HIGH/CRITICAL
  - Human review required if:
    - evidence strength is WEAK/MODERATE (CONCLUSIVE = automated)
    - CRITICAL always requires human review
    - confidence < 0.85 requires human review
- **Connections:** Main output -> **Check if Human Review Required**
- **Edge cases:**
  - The CRITICAL rule (“always requires”) competes with “CONCLUSIVE = automated”; depending on agent interpretation, it may still set automated. If you need strict enforcement, implement with deterministic IF logic outside the agent.
  - References to `$('Simulate Assessment Data').first()` assume a single-item context and that the node name remains unchanged.

---

### Block 2.7 — Human Review Gate (Queue + Email + Wait)
**Overview:** If human review is required, the workflow stores the case, emails the reviewer, and waits for an inbound webhook to resume.

**Nodes involved:**
- Check if Human Review Required
- Store Case for Human Review
- Send Human Review Alert
- Wait for Human Decision

#### Node: Check if Human Review Required
- **Type / role:** `IF` — branches on orchestration output.
- **Condition:** `={{ $json.output.humanReviewRequired }} equals true` (boolean comparison)
- **Connections:**
  - True -> **Store Case for Human Review**
  - False -> **Store Automated Case**
- **Edge cases:**
  - If `humanReviewRequired` is returned as `"true"` (string) instead of boolean, strictness may matter; this IF uses “loose” validation, which helps, but do not rely on that for production.

#### Node: Store Case for Human Review
- **Type / role:** `Data Table` — persists a pending case record for reviewers.
- **Target table:** `HumanReviewQueue`
- **Fields written:**
  - `caseId`: from orchestration output
  - `status`: `PENDING_REVIEW`
  - `createdAt`: now
  - `riskLevel`: from **Integrity Signal Agent** output
  - `studentId`, `assessmentId`: from **Simulate Assessment Data**
  - `reviewReason`, `documentation`, `recommendedSanction`: from orchestration output
- **Connections:** -> **Send Human Review Alert**
- **Failure modes:** Data table missing, write permissions, schema mismatch.

#### Node: Send Human Review Alert
- **Type / role:** `Email Send` — notifies reviewer with case summary and links.
- **To:** `{{ $('Workflow Configuration').first().json.reviewerEmail }}`
- **From:** placeholder `<__PLACEHOLDER_VALUE__Sender Email Address__>`
- **Subject:** `Academic Integrity Case Requires Human Review - {{ $json.output.caseId }}`
- **HTML body:** Includes case details, investigation summary, documentation, next steps list.
- **Important implementation detail:** The email builds links using:
  - `https://cschin.app.n8n.cloud/webhook/{{ $('Wait for Human Decision').first().webhookId }}`
  - Both “Approve Action” and “Reject Action” use the same URL (no decision parameter is included).
- **Connections:** -> **Wait for Human Decision**
- **Failure modes / edge cases:**
  - SMTP/credential issues (node depends on n8n email configuration).
  - The “Approve/Reject” links are identical and do not transmit a choice; without additional parameters/body, the workflow cannot distinguish decisions.
  - Hard-coded base URL `https://cschin.app.n8n.cloud` must match your actual n8n public URL.

#### Node: Wait for Human Decision
- **Type / role:** `Wait` — pauses execution until a webhook resumes it.
- **Resume mode:** `webhook`
- **HTTP method:** `POST`
- **Output:** After resume, sends data to **Merge All Cases** (input index 0)
- **Edge cases:**
  - No handler for decision payload: there is no downstream logic that updates status based on approval/rejection.
  - Webhook must be reachable publicly; behind VPN/firewall will break resume.

---

### Block 2.8 — Automated Case Path (Store + Later Merge)
**Overview:** Stores orchestrated cases that do not require human review.

**Nodes involved:**
- Store Automated Case

#### Node: Store Automated Case
- **Type / role:** `Data Table` — persists an automatically processed case.
- **Target table:** `AutomatedCases`
- **Fields written:**
  - `caseId`, `documentation`, `confidenceLevel`, `recommendedSanction` from orchestration output
  - `status`: `AUTOMATED_PROCESSED`
  - `riskLevel`: from Integrity Signal Agent
  - `studentId`, `assessmentId`: from Simulate Assessment Data
  - `processedAt`: now
  - `actionsTaken`: `{{ $json.output.actionsTaken.join(', ') }}`
- **Connections:** -> **Merge All Cases** (input index 1)
- **Edge cases:**
  - If `actionsTaken` is missing or not an array, `.join()` will throw an expression error. Consider guarding: `Array.isArray(...) ? ... : ''`.

---

### Block 2.9 — Merge & Archive
**Overview:** Collects results from the human-review wait path, automated path, and low-risk path, then archives all items.

**Nodes involved:**
- Merge All Cases
- Archive All Cases

#### Node: Merge All Cases
- **Type / role:** `Merge` — combines multiple inputs.
- **Configuration:** `numberInputs: 3`
  - Input 0: from **Wait for Human Decision**
  - Input 1: from **Store Automated Case**
  - Input 2: from **Store Low Risk Case**
- **Connections:** -> **Archive All Cases**
- **Edge cases:**
  - Merge behavior depends on node settings and arrival patterns; with 3 inputs, if one branch produces no item, the merge may wait/behave unexpectedly depending on merge mode defaults (not explicitly configured here).
  - If the Wait node never resumes, the entire workflow execution may not reach archiving for that run.

#### Node: Archive All Cases
- **Type / role:** `Data Table` — archives merged cases.
- **Target table:** `AllCasesArchive`
- **Configuration:** No explicit column mapping shown; likely writes incoming item fields as-is (depending on Data Table node behavior/version).
- **Edge cases:**
  - Heterogeneous items: low-risk vs orchestration outputs have different shapes; ensure the archive table schema can store them (or store as JSON blob).
  - If the node expects defined columns, lack of mapping can cause failures.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily workflow entry point | — | Workflow Configuration | ## Integrity Assessment  \nPrioritizes investigation resources on high-risk matters requiring immediate attention versus routine monitoring. |
| Workflow Configuration | n8n-nodes-base.set | Defines reviewer email + thresholds | Schedule Trigger | Simulate Assessment Data | ## Integrity Assessment  \nPrioritizes investigation resources on high-risk matters requiring immediate attention versus routine monitoring. |
| Simulate Assessment Data | n8n-nodes-base.set | Generates synthetic submission payload | Workflow Configuration | Integrity Signal Agent | ## Integrity Assessment  \nPrioritizes investigation resources on high-risk matters requiring immediate attention versus routine monitoring. |
| OpenAI Model - Integrity Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o model for integrity analysis agent | — | Integrity Signal Agent (ai_languageModel) | ## Integrity Assessment  \nPrioritizes investigation resources on high-risk matters requiring immediate attention versus routine monitoring. |
| Structured Output - Integrity Signals | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured integrity signal output | — | Integrity Signal Agent (ai_outputParser) | ## Integrity Assessment  \nPrioritizes investigation resources on high-risk matters requiring immediate attention versus routine monitoring. |
| Integrity Signal Agent | @n8n/n8n-nodes-langchain.agent | Produces risk level + plagiarism/anomaly scores | Simulate Assessment Data | Route by Risk Level | ## Integrity Assessment  \nPrioritizes investigation resources on high-risk matters requiring immediate attention versus routine monitoring. |
| Route by Risk Level | n8n-nodes-base.switch | Branches flow by risk level | Integrity Signal Agent | Prepare Low Risk Response; Case Orchestration Agent | ## Data Orchestration  \nReveals systemic patterns and repeat offenders that isolated signal analysis would miss. |
| Prepare Low Risk Response | n8n-nodes-base.set | Builds minimal low-risk case record | Route by Risk Level (Low Risk) | Store Low Risk Case | ## Data Orchestration  \nReveals systemic patterns and repeat offenders that isolated signal analysis would miss. |
| Store Low Risk Case | n8n-nodes-base.dataTable | Persists low-risk cases | Prepare Low Risk Response | Merge All Cases | ## Data Orchestration  \nReveals systemic patterns and repeat offenders that isolated signal analysis would miss. |
| OpenAI Model - Orchestration Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o model for orchestration agent | — | Case Orchestration Agent (ai_languageModel) | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Structured Output - Orchestration | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured orchestration decision output | — | Case Orchestration Agent (ai_outputParser) | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Case Orchestration Agent | @n8n/n8n-nodes-langchain.agent | Calls investigation tool and decides human vs automated | Route by Risk Level (Medium Risk / High/Critical Risk) | Check if Human Review Required | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Investigation Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Deep investigation tool callable by orchestration agent | — | Case Orchestration Agent (ai_tool) | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| OpenAI Model - Investigation Tool | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o model for investigation tool | — | Investigation Agent Tool (ai_languageModel) | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Structured Output - Investigation | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured investigation output | — | Investigation Agent Tool (ai_outputParser) | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Check if Human Review Required | n8n-nodes-base.if | Splits to human review queue vs automated store | Case Orchestration Agent | Store Case for Human Review; Store Automated Case | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Store Case for Human Review | n8n-nodes-base.dataTable | Stores pending cases for reviewer | Check if Human Review Required (true) | Send Human Review Alert | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Send Human Review Alert | n8n-nodes-base.emailSend | Emails reviewer with case details + webhook links | Store Case for Human Review | Wait for Human Decision | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Wait for Human Decision | n8n-nodes-base.wait | Pauses until webhook callback resumes | Send Human Review Alert | Merge All Cases | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Store Automated Case | n8n-nodes-base.dataTable | Stores automatically processed orchestration cases | Check if Human Review Required (false) | Merge All Cases | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Merge All Cases | n8n-nodes-base.merge | Converges low-risk + automated + human-review-resumed items | Wait for Human Decision; Store Automated Case; Store Low Risk Case | Archive All Cases |  |
| Archive All Cases | n8n-nodes-base.dataTable | Archives final case items | Merge All Cases | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Comment container | — | — | ## Prerequisites\nAPI key, Gmail account with app password\n## Use Cases\nFinancial fraud detection, employee misconduct investigation\n## Customization\nIntegrate case management systems, add industry-specific risk models\n## Benefits\nReduces investigation triage time by 65%, ensures consistent risk assessment methodology |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Setup Steps\n1. Configure Llama-3.1-70B-Instruct model access\n2. Set up schedule trigger for daily or continuous monitoring cycles\n3. Configure risk-based routing logic (Low/High thresholds)\n4. Connect Gmail for human review alerts to compliance officers\n5. Set up Google Sheets for case storage and automated archival |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment container | — | — | ## How It Works\nThis workflow automates integrity signal detection and investigation orchestration for compliance officers, ethics teams, and risk managers in financial services, healthcare, and regulated industries. It solves the challenge of identifying potential misconduct while ensuring human judgment governs sensitive investigations. Scheduled triggers initiate assessments on synthetic integrity signals, which flow to an AI agent for severity classification based on risk indicators. High-risk signals route to parallel AI investigation agents: data correlation analysis to uncover patterns and anomaly detection to flag statistical outliers. Results converge at mandatory human review gates where compliance professionals evaluate findings before case creation. Approved investigations generate structured case records, while cleared signals archive automatically with full audit trails. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Investigation Analysis\nIdentifies unusual activities warranting deeper investigation through quantitative evidence discovery. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Data Orchestration \nReveals systemic patterns and repeat offenders that isolated signal analysis would miss. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Integrity Assessment \nPrioritizes investigation resources on high-risk matters requiring immediate attention versus routine monitoring. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow**
   - Name: *AI-powered academic integrity monitoring and investigation system* (title can be your French title; name is cosmetic).

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Set to run daily at **09:00** (or your desired time/timezone).
   - Connect to the next node.

3. **Add “Workflow Configuration” (Set)**
   - Node: **Set**
   - Add fields:
     - `reviewerEmail` (string) = your integrity reviewer mailbox
     - `plagiarismThreshold` (number) = 75
     - `anomalyThreshold` (number) = 80
     - `highRiskThreshold` (number) = 85 (optional unless you wire it into logic)
   - Enable **Include Other Fields**
   - Connect from Schedule Trigger -> this node.

4. **Add “Simulate Assessment Data” (Set)** (replace later with real ingestion)
   - Node: **Set**
   - Add fields (as expressions where indicated):
     - `studentId` = `STU-2024-{{ Math.floor(Math.random() * 10000) }}`
     - `assessmentId` = `ASSESS-{{ Math.floor(Math.random() * 1000) }}`
     - `submissionText` = sample text or your real submission body
     - `submissionTimestamp` = `{{ $now.toISO() }}`
     - `previousSubmissions` = 5
     - `averageGrade` = 78.5
     - `timeSpentMinutes` = 45
     - `wordCount` = 850
     - `browserFingerprint` = `FP-{{ Math.floor(Math.random() * 100000) }}`
   - Connect Workflow Configuration -> Simulate Assessment Data.

5. **Add Integrity AI components**
   1) Node: **OpenAI Model - Integrity Agent** (`lmChatOpenAi`)
      - Model: `gpt-4o`
      - Temperature: `0.1`
      - Configure **OpenAI API credentials** in n8n and select them here.
   2) Node: **Structured Output - Integrity Signals** (`outputParserStructured`)
      - Provide the schema example for: riskLevel, plagiarismScore, anomalyScore, indicators arrays, confidence, reasoning, recommendedAction.
   3) Node: **Integrity Signal Agent** (`agent`)
      - Set “Prompt type” to **Define**
      - Paste the system message describing scoring rules and thresholds.
      - In the “text” field, compose the prompt using expressions referencing:
        - current item submission fields (`$json.studentId`, etc.)
        - thresholds from `$('Workflow Configuration').first().json...`
      - Enable **Has Output Parser**
      - Connect:
        - Simulate Assessment Data -> Integrity Signal Agent (main)
        - OpenAI Model - Integrity Agent -> Integrity Signal Agent (`ai_languageModel`)
        - Structured Output - Integrity Signals -> Integrity Signal Agent (`ai_outputParser`)

6. **Add “Route by Risk Level” (Switch)**
   - Node: **Switch**
   - Condition value: `={{ $json.output.riskLevel }}`
   - Create outputs:
     - LOW -> output “Low Risk”
     - MEDIUM -> output “Medium Risk”
     - HIGH or CRITICAL -> output “High/Critical Risk”
   - (Optional) Connect fallback “Unclassified” to logging/alerting.
   - Connect Integrity Signal Agent -> Route by Risk Level.

7. **Low-risk branch**
   1) Node: **Prepare Low Risk Response** (`Set`)
      - Create `caseId` (use UUID ideally), plus `studentId`, `assessmentId`, `riskLevel`, `action`, `status`, `processedAt`.
   2) Node: **Store Low Risk Case** (`Data Table`)
      - Create/select Data Table: `LowRiskCases`
      - Map columns from the set fields.
   - Connect Route “Low Risk” -> Prepare Low Risk Response -> Store Low Risk Case.

8. **Investigation + orchestration branch (for Medium and High/Critical)**
   1) Node: **OpenAI Model - Orchestration Agent** (`lmChatOpenAi`)
      - Model: `gpt-4o`
      - Temperature: `0.2`
   2) Node: **Structured Output - Orchestration** (`outputParserStructured`)
      - Provide schema example for caseId, decision, actionsTaken, documentationGenerated, humanReviewRequired, etc.
   3) Node: **OpenAI Model - Investigation Tool** (`lmChatOpenAi`)
      - Model: `gpt-4o`
      - Temperature: `0.1`
   4) Node: **Structured Output - Investigation** (`outputParserStructured`)
      - Provide schema example for evidenceStrength, matchPercentage, sourcesChecked, etc.
   5) Node: **Investigation Agent Tool** (`agentTool`)
      - Set the tool system message (deep investigation simulation).
      - Set input text to: `{{ $fromAI("caseData", "Academic integrity case data requiring investigation", "json") }}`
      - Enable **Has Output Parser**
      - Connect:
        - OpenAI Model - Investigation Tool -> Investigation Agent Tool (`ai_languageModel`)
        - Structured Output - Investigation -> Investigation Agent Tool (`ai_outputParser`)
   6) Node: **Case Orchestration Agent** (`agent`)
      - System message: orchestration rules (call tool, decide human review).
      - Text: include integrity signals and identifiers.
      - Enable **Has Output Parser**
      - Connect:
        - Route outputs “Medium Risk” and “High/Critical Risk” -> Case Orchestration Agent (main)
        - OpenAI Model - Orchestration Agent -> Case Orchestration Agent (`ai_languageModel`)
        - Structured Output - Orchestration -> Case Orchestration Agent (`ai_outputParser`)
        - Investigation Agent Tool -> Case Orchestration Agent (`ai_tool`)

9. **Add Human review decision IF**
   - Node: **IF** (“Check if Human Review Required”)
   - Condition: `={{ $json.output.humanReviewRequired }} equals true`
   - Connect Case Orchestration Agent -> IF.

10. **Human review path**
   1) Node: **Store Case for Human Review** (`Data Table`)
      - Table: `HumanReviewQueue`
      - Map: caseId, status=PENDING_REVIEW, createdAt, riskLevel, studentId, assessmentId, reviewReason, documentation, recommendedSanction.
   2) Node: **Send Human Review Alert** (`Email Send`)
      - Configure email sending (SMTP/Gmail app password or provider).
      - To: reviewerEmail from Workflow Configuration
      - Subject/body: include case details.
      - Ensure your n8n public base URL is correct in links.
   3) Node: **Wait for Human Decision** (`Wait`)
      - Resume: **Webhook**
      - Method: **POST**
   - Connect IF true -> Store Case for Human Review -> Send Human Review Alert -> Wait for Human Decision.

11. **Automated path**
   - Node: **Store Automated Case** (`Data Table`)
     - Table: `AutomatedCases`
     - Map: caseId, status, riskLevel, studentId, assessmentId, processedAt, actionsTaken (guard `.join`), documentation, confidenceLevel, recommendedSanction.
   - Connect IF false -> Store Automated Case.

12. **Merge + archive**
   1) Node: **Merge All Cases** (`Merge`)
      - Set number of inputs: **3**
      - Connect:
        - Wait for Human Decision -> Merge (input 0)
        - Store Automated Case -> Merge (input 1)
        - Store Low Risk Case -> Merge (input 2)
   2) Node: **Archive All Cases** (`Data Table`)
      - Table: `AllCasesArchive`
      - Decide whether to map columns explicitly or store a JSON blob.
   - Connect Merge -> Archive All Cases.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Prerequisites: API key, Gmail account with app password” | Sticky note content (prereqs) |
| “Setup Steps: Configure Llama-3.1-70B-Instruct model access…” | Sticky note content (note: workflow actually uses GPT‑4o, not Llama) |
| Cross-industry framing (financial services/healthcare) | Sticky note “How It Works” describes a broader compliance use case than academic integrity |
| Approve/Reject links are identical and do not encode a decision | Email node: both buttons point to the same webhook URL; add query params or distinct endpoints if you need actionable decisions |
| `highRiskThreshold` exists but is unused | Workflow Configuration; consider integrating it into agent prompt/routing for consistency |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.