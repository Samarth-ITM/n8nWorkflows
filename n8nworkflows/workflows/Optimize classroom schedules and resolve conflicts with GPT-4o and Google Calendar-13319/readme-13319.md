Optimize classroom schedules and resolve conflicts with GPT-4o and Google Calendar

https://n8nworkflows.xyz/workflows/optimize-classroom-schedules-and-resolve-conflicts-with-gpt-4o-and-google-calendar-13319


# Optimize classroom schedules and resolve conflicts with GPT-4o and Google Calendar

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Optimize classroom schedules and resolve conflicts with GPT-4o and Google Calendar  
**Workflow name (in JSON):** AI-Driven Classroom Scheduling and Conflict Resolution System

This workflow runs on a schedule (daily at 06:00) to analyze classroom scheduling requests, detect resource/faculty/equipment conflicts using multiple GPT-4o agents, optionally auto-resolve low-risk conflicts with high confidence, and route outcomes to Slack (critical escalation) or email (standard recommendations). It also logs execution summaries.

### 1.1 Automated Input & Configuration
Defines operational parameters (Slack channel, scheduling team email, calendar ID, thresholds) and prepares sample datasets (classrooms, faculty, timetable requests).

### 1.2 Multi-Agent AI Analysis (Resource + Operations)
- **Resource Analysis Agent:** checks feasibility, capacity buffers, equipment match, workload constraints, and conflicts.
- **Operations Agent:** produces an “optimal schedule” proposal, conflict list with severity, alternatives, and escalation decision.

### 1.3 Historical Context & Synthesis
A code node simulates historical utilization/preferences; results are merged with AI analyses.

### 1.4 Master Orchestration (Tools + Decision)
A master agent uses three tools:
- schedule optimization tool (GPT-4o + structured output),
- conflict scoring tool (code),
- Google Calendar “getAll” tool (availability check),
then returns a structured final decision.

### 1.5 Auto-Resolution vs Manual Escalation + Routing
If the final decision allows auto-resolution at sufficient confidence, it marks actions as auto-applied; otherwise it generates a conflict report for human review. Then it routes to Slack (critical) or email (standard) and logs results.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Automated Input & Configuration
**Overview:** Triggers the workflow daily and establishes configuration constants plus baseline scheduling datasets used by all later blocks.  
**Nodes involved:** Schedule Trigger; Workflow Configuration; Prepare Scheduling Data.

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point on a time rule.
- **Config choices:** Runs at **06:00** (interval rule with `triggerAtHour: 6`).
- **Connections:**  
  - Output → **Workflow Configuration**
- **Edge cases / failures:** Timezone differences (n8n instance timezone), missed runs if instance is down.

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes parameters used across nodes.
- **Config choices (key fields):**
  - `slackChannelId` (placeholder)
  - `schedulingTeamEmail` (placeholder)
  - `maxClassroomCapacity` = 50
  - `minBreakBetweenClasses` = 15 (minutes)
  - `conflictThreshold` = "high"
  - `googleCalendarId` (placeholder)
  - `autoResolutionThreshold` = 0.85
  - `historicalDataDays` = 90
  - **Include other fields:** enabled (passes through upstream data too).
- **Expressions used downstream:** e.g. `$('Workflow Configuration').first().json.googleCalendarId`
- **Connections:**  
  - Input ← Schedule Trigger  
  - Output → Prepare Scheduling Data
- **Edge cases / failures:** Placeholder values not replaced → Slack/email/calendar nodes fail; type mismatch risks if edited (numbers stored as strings).

#### Node: Prepare Scheduling Data
- **Type / role:** `Set` — constructs arrays of classrooms/faculty/requests (currently hardcoded examples).
- **Config choices (arrays):**
  - `classrooms`: rooms with `id`, `capacity`, `equipment`, `building`
  - `faculty`: faculty with `maxHoursPerDay`, `preferredTimeSlots`
  - `timetableRequests`: course requests with `requiredCapacity`, `duration`, `preferredTime`, `equipment`
  - **Include other fields:** enabled.
- **Connections:**  
  - Input ← Workflow Configuration  
  - Outputs → Resource Analysis Agent, Calculate Historical Patterns (parallel)
- **Edge cases / failures:** Invalid JSON in set expressions; missing required fields leads to weaker/incorrect AI output.

**Sticky note context (applies to this block’s nodes):**  
“## Automated Data Collection — **Why**: Schedule trigger initiates competitor data fetching…” (Note: text is generic/misaligned with classroom scheduling; still attached in the workflow canvas.)

---

### Block 2.2 — Resource Feasibility & Constraint Analysis (AI)
**Overview:** Uses GPT-4o to evaluate capacity buffers, equipment availability, faculty constraints, utilization, and identify conflicts; output is forced into a structured JSON schema.  
**Nodes involved:** OpenAI Model - Resource Agent; Resource Analysis Output Parser; Resource Analysis Agent.

#### Node: OpenAI Model - Resource Agent
- **Type / role:** `lmChatOpenAi` — provides GPT-4o chat model for the agent.
- **Config choices:** model = **gpt-4o**, temperature = **0.2** (more deterministic).
- **Credentials:** OpenAI API credential named “OpenAi account”.
- **Connections:**  
  - ai_languageModel → Resource Analysis Agent
- **Edge cases / failures:** OpenAI auth errors, rate limits, model name availability, timeouts.

#### Node: Resource Analysis Output Parser
- **Type / role:** `outputParserStructured` — validates/coerces agent output into a JSON schema.
- **Config choices:** Manual JSON schema with:
  - `resourceAnalysis.classroomAvailability[]` (slots, utilizationRate, conflicts)
  - `resourceAnalysis.facultyConstraints[]` (availableHours, violations)
  - `resourceAnalysis.equipmentMatches[]`
  - `resourceAnalysis.overallFeasibility`, `criticalIssues[]`
- **Connections:**  
  - ai_outputParser → Resource Analysis Agent
- **Edge cases / failures:** Model outputs non-JSON or schema-mismatched JSON → parser errors; missing fields may cause downstream expressions to fail.

#### Node: Resource Analysis Agent
- **Type / role:** `langchain agent` — performs the resource analysis task with a strict system message.
- **Prompt inputs (expressions):**
  - `Classroom Data: {{ JSON.stringify($json.classrooms) }}`
  - `Faculty Data: {{ JSON.stringify($json.faculty) }}`
  - `Timetable Requests: {{ JSON.stringify($json.timetableRequests) }}`
  - Constraints pulled from config:
    - `Max Capacity={{ $('Workflow Configuration').first().json.maxClassroomCapacity }}`
    - `Min Break={{ $('Workflow Configuration').first().json.minBreakBetweenClasses }} minutes`
- **System message highlights:**
  - Capacity must exceed required capacity by **10%** (safety buffer)
  - Enforce max faculty hours/day, min breaks, equipment completeness, building proximity
  - Return in the defined JSON structure
- **Connections:**  
  - Input ← Prepare Scheduling Data  
  - Output → Operations Agent; Merge Analysis Results (input 1)  
  - Uses: OpenAI Model - Resource Agent; Resource Analysis Output Parser
- **Edge cases / failures:** If data arrays empty, analysis becomes low-value; hallucinated time slots if not grounded; parser failures if model deviates.

**Sticky note context (applies to this block’s nodes):**  
“## Multi-Agent AI Analysis — **Why**: Delivers expert-level analysis across multiple dimensions…”

---

### Block 2.3 — Operations Scheduling Recommendations (AI)
**Overview:** Generates an actionable schedule proposal, conflict classification, alternatives, escalation recommendation, and action items, in structured output.  
**Nodes involved:** OpenAI Model - Operations Agent; Operations Output Parser; Operations Agent.

#### Node: OpenAI Model - Operations Agent
- **Type / role:** `lmChatOpenAi` — GPT-4o model for operations planning.
- **Config choices:** model = **gpt-4o**, temperature = **0.2**.
- **Credentials:** “OpenAi account”.
- **Connections:** ai_languageModel → Operations Agent
- **Edge cases / failures:** Same as other OpenAI model nodes.

#### Node: Operations Output Parser
- **Type / role:** `outputParserStructured` — enforces `schedulingRecommendations` schema.
- **Schema highlights:**
  - `optimalSchedule[]` includes `confidence` per scheduled item
  - `conflicts[]` includes `severity`, `resolutionOptions[]`
  - `escalationRequired` boolean + reason
  - `overallStatus` and `actionItems[]`
- **Connections:** ai_outputParser → Operations Agent
- **Edge cases / failures:** If conflicts are output as strings vs objects → schema mismatch.

#### Node: Operations Agent
- **Type / role:** `langchain agent` — turns resource analysis into an operational schedule plan.
- **Prompt inputs (expressions):**
  - `Resource Analysis Results: {{ JSON.stringify($json.output) }}`
  - `Original Requests: {{ JSON.stringify($('Prepare Scheduling Data').first().json.timetableRequests) }}`
  - Note: `$json.output` here is the Resource Analysis Agent’s parsed output (as connected).
- **System message highlights:**
  - Severity levels: CRITICAL/HIGH/MEDIUM/LOW
  - Escalation if any CRITICAL, >3 HIGH, impossible schedule, unresolvable workload violations
  - Output in defined JSON structure
- **Connections:**  
  - Input ← Resource Analysis Agent  
  - Outputs → Route by Severity; Merge Analysis Results (input 3)  
  - Uses: OpenAI Model - Operations Agent; Operations Output Parser
- **Edge cases / failures:** EscalationRequired may be inconsistent with conflicts list; downstream routing depends on `escalationRequired`.

**Sticky note context (applies to this block’s nodes):**  
“## Multi-Agent AI Analysis — **Why**: Delivers expert-level analysis…”

---

### Block 2.4 — Historical Patterns & Merge
**Overview:** Adds historical context (simulated) and merges it with Resource Analysis and Operations output to feed the Master Orchestrator.  
**Nodes involved:** Calculate Historical Patterns; Merge Analysis Results.

#### Node: Calculate Historical Patterns
- **Type / role:** `Code` — generates simulated historical utilization and trends.
- **Config choices / logic:**
  - Reads `historicalDataDays` from Workflow Configuration (default 90).
  - Creates `historicalPatterns`:
    - `classroomUtilization[]` with random utilization/conflict frequency
    - `facultyPreferences[]` with random satisfaction/cancellation metrics
    - `conflictTrends`, `seasonalPatterns`
  - Output shape: `{ historicalPatterns, originalData }`
- **Connections:**  
  - Input ← Prepare Scheduling Data  
  - Output → Merge Analysis Results (input 2)
- **Edge cases / failures:** Randomness makes outputs non-deterministic; in production you’d replace with DB queries—missing credentials/DB nodes are not present.

#### Node: Merge Analysis Results
- **Type / role:** `Merge` — combines **3 inputs** into one item for orchestration.
- **Config choices:** `numberInputs = 3`.
- **Expected inputs (by connection order):**
  1. From Resource Analysis Agent
  2. From Calculate Historical Patterns
  3. From Operations Agent
- **Output:** An array-like merged JSON where the Master Orchestrator references:
  - `$json[0].output` (resource)
  - `$json[1].historicalPatterns` (history)
  - `$json[2].output` (operations)
- **Connections:**  
  - Inputs ← Resource Analysis Agent, Calculate Historical Patterns, Operations Agent  
  - Output → Master Orchestrator Agent
- **Edge cases / failures:** If any upstream branch produces 0 items, merge may not behave as expected; index-based access in the orchestrator prompt can break if item ordering differs.

**Sticky note context (applies to this block’s nodes):**  
“## Multi-Agent AI Analysis — **Why**: Delivers expert-level analysis…”

---

### Block 2.5 — Master Orchestration with Tools (Optimization, Scoring, Calendar)
**Overview:** A master GPT-4o agent synthesizes analysis + history + operations, calls tools to optimize and score conflicts and check Google Calendar, and returns a structured final decision.  
**Nodes involved:** OpenAI Model - Master Orchestrator; Master Orchestrator Output Parser; Master Orchestrator Agent; Schedule Optimization Agent Tool; OpenAI Model - Optimization Agent; Optimization Output Parser; Conflict Score Calculator Tool; Google Calendar Tool.

#### Node: OpenAI Model - Master Orchestrator
- **Type / role:** `lmChatOpenAi` — primary model for orchestration.
- **Config choices:** model = **gpt-4o**, temperature = **0.1** (more deterministic).
- **Connections:** ai_languageModel → Master Orchestrator Agent
- **Edge cases / failures:** Same OpenAI risks; tool-calling behavior may vary by model/version.

#### Node: Master Orchestrator Output Parser
- **Type / role:** `outputParserStructured` — enforces `finalDecision` schema:
  - `canAutoResolve` (boolean), `confidence` (number)
  - `recommendedActions[]`, `riskAssessment`
  - `calendarIntegration` (boolean), `optimizationApplied` (boolean)
  - `conflictScoreTotal` (number), `reasoning` (string)
- **Connections:** ai_outputParser → Master Orchestrator Agent
- **Edge cases / failures:** If the agent doesn’t return `finalDecision` exactly, downstream `IF` node expressions fail.

#### Node: Schedule Optimization Agent Tool
- **Type / role:** `agentTool` — exposes a GPT-powered optimization capability as a callable tool for the master agent.
- **Config choices:**
  - Input mapping: `{{ $fromAI('scheduleData', 'Current schedule and constraints to optimize', 'json') }}`
  - System message: “Schedule Optimization specialist… maximize efficiency…”
  - **Has output parser:** enabled (see Optimization Output Parser)
- **Connections:**  
  - ai_tool → Master Orchestrator Agent  
  - Uses OpenAI Model - Optimization Agent and Optimization Output Parser
- **Edge cases / failures:** The master agent must provide `scheduleData` in the expected JSON shape; otherwise tool call fails.

#### Node: OpenAI Model - Optimization Agent
- **Type / role:** `lmChatOpenAi` — model backing the optimization tool.
- **Config choices:** model = **gpt-4o**, temperature = **0.3** (slightly more exploratory).
- **Connections:** ai_languageModel → Schedule Optimization Agent Tool
- **Edge cases / failures:** Same OpenAI risks.

#### Node: Optimization Output Parser
- **Type / role:** `outputParserStructured` — enforces:
  - `optimizedSchedule[]` with `optimizationScore` and `reasoning`
  - `efficiencyGains`, `conflictsResolved`
- **Connections:** ai_outputParser → Schedule Optimization Agent Tool
- **Edge cases / failures:** Model output not matching schema → tool failure, which can cascade to orchestrator failure.

#### Node: Conflict Score Calculator Tool
- **Type / role:** `toolCode` — computes weighted conflict scores to prioritize risk.
- **Logic summary:**
  - Accepts `query` (stringified JSON or object) representing `conflicts[]`.
  - Severity weights: CRITICAL=10, HIGH=5, MEDIUM=2, LOW=1
  - Score per conflict: `severityScore * affectedCoursesCount * complexityFactor` (1.5 if >2 courses)
  - Outputs JSON string with `totalScore`, `breakdown[]`, `riskLevel`, counts.
- **Connections:** ai_tool → Master Orchestrator Agent
- **Edge cases / failures:** If `query` is not valid JSON → returns an error string (not JSON), which can confuse the agent; conflict objects missing `severity` or `affectedCourses` reduces accuracy.

#### Node: Google Calendar Tool
- **Type / role:** `googleCalendarTool` — callable tool to fetch calendar events.
- **Config choices:**
  - Operation: `getAll`
  - `returnAll: true`
  - Calendar ID from config: `{{ $('Workflow Configuration').first().json.googleCalendarId }}`
- **Credentials:** Google Calendar OAuth2 credential “Google Calendar account”.
- **Connections:** ai_tool → Master Orchestrator Agent
- **Edge cases / failures:** Invalid calendar ID, OAuth scope issues, token expiration, large calendars causing timeouts or big payloads.

#### Node: Master Orchestrator Agent
- **Type / role:** `langchain agent` — central decision-maker that calls tools and emits final decision.
- **Prompt inputs (index-based after Merge):**
  - Resource: `{{ JSON.stringify($json[0].output) }}`
  - Historical: `{{ JSON.stringify($json[1].historicalPatterns) }}`
  - Operations: `{{ JSON.stringify($json[2].output) }}`
- **System message decision criteria (auto-resolution):**
  - Total conflict score < 15
  - Optimization confidence > 85%
  - No CRITICAL conflicts
  - Calendar slots available
  - Historical success rate > 75%
- **Connections:**  
  - Input ← Merge Analysis Results  
  - Output → Check Auto-Resolution Possible  
  - Uses: OpenAI Model - Master Orchestrator, Output Parser, and all three tools
- **Edge cases / failures:**
  - Merge ordering changes break `$json[0]/[1]/[2]` references.
  - Tool failures (calendar, parsers) can prevent finalDecision creation.
  - The workflow’s later IF uses `finalDecision` fields strictly.

**Sticky note context (applies to this block’s nodes):**  
“## Intelligent Routing & Validation — **Why**: Ensures data quality and prioritizes critical insights…”

---

### Block 2.6 — Resolution Paths (Auto vs Manual) + Merge
**Overview:** Uses the orchestrator decision to either mark the result as auto-resolved or generate a structured escalation report; then merges both paths.  
**Nodes involved:** Check Auto-Resolution Possible; Apply Auto-Resolution; Generate Conflict Report; Merge Resolution Paths.

#### Node: Check Auto-Resolution Possible
- **Type / role:** `If` — branches by orchestrator decision.
- **Conditions (AND):**
  1. `{{ $json.output.finalDecision.canAutoResolve }}` equals `true`
  2. `{{ $json.output.finalDecision.confidence }}` >= `0.85`
     - Note: constant is hardcoded at 0.85 here (not reading `autoResolutionThreshold`).
- **Connections:**  
  - Input ← Master Orchestrator Agent  
  - True → Apply Auto-Resolution  
  - False → Generate Conflict Report
- **Edge cases / failures:** If `output.finalDecision` missing, expressions evaluate to null → branch may default unexpectedly or error depending on n8n settings.

#### Node: Apply Auto-Resolution
- **Type / role:** `Set` — annotates the item as auto-resolved.
- **Fields set:**
  - `resolutionType` = `AUTO_RESOLVED`
  - `resolutionTimestamp` = `{{ $now.toISO() }}`
  - `appliedActions` = `{{ $json.output.finalDecision.recommendedActions }}`
  - `autoResolutionConfidence` = `{{ $json.output.finalDecision.confidence }}`
  - Pass-through: include other fields enabled
- **Connections:** Output → Merge Resolution Paths (input 1)
- **Edge cases / failures:** This node does not actually write to Google Calendar; it only labels the result. If you expect calendar event creation/updates, additional nodes are required.

#### Node: Generate Conflict Report
- **Type / role:** `Code` — creates a human-facing escalation report object.
- **Logic summary:**
  - Builds `conflictReport` with:
    - `resolutionType: MANUAL_ESCALATION`
    - `decision` = `data.output.finalDecision`
    - `conflictSummary` (score, riskLevel, reasoning)
    - `recommendedActions`
    - `escalationPriority` (URGENT if score > 30)
- **Connections:** Output → Merge Resolution Paths (input 2)
- **Edge cases / failures:** If `finalDecision` incomplete, report has fallbacks; score may be undefined → defaults to 0.

#### Node: Merge Resolution Paths
- **Type / role:** `Merge` — reunites auto and manual paths.
- **Config choices:** default merge behavior (no explicit mode set in JSON excerpt).
- **Connections:**  
  - Inputs ← Apply Auto-Resolution, Generate Conflict Report  
  - Output → Route by Severity
- **Edge cases / failures:** If only one branch runs, merge behavior must be compatible; otherwise it can stall waiting for both inputs depending on merge mode. (Review merge mode in UI if executions hang.)

**Sticky note context (applies to this block’s nodes):**  
“## Intelligent Routing & Validation — **Why**: Ensures data quality…”

---

### Block 2.7 — Routing, Notifications, and Logging
**Overview:** Routes results based on escalation flag to Slack (critical) or email (standard), then logs a summary.  
**Nodes involved:** Route by Severity; Notify Critical Conflicts; Email Scheduling Recommendations; Log Results.

#### Node: Route by Severity
- **Type / role:** `Switch` — routes by `escalationRequired`.
- **Rules:**
  - Output “Critical” if `{{ $json.output.schedulingRecommendations.escalationRequired }}` is `true`
  - Output “Standard” if it is `false`
- **Connections:**  
  - Input ← Operations Agent (direct) AND Merge Resolution Paths (later path)  
  - Critical → Notify Critical Conflicts  
  - Standard → Email Scheduling Recommendations
- **Edge cases / failures (important):**
  - **Field mismatch risk:** After Master Orchestrator, the data contains `output.finalDecision...` but routing still checks `output.schedulingRecommendations...`. If the merged resolution path item does not include `output.schedulingRecommendations`, routing may fail or misroute.
  - Additionally, because **Operations Agent** is also connected directly into this switch, the workflow can notify/email **before** orchestration finishes, causing duplicate or inconsistent outputs.

#### Node: Notify Critical Conflicts
- **Type / role:** `Slack` — sends critical alert message.
- **Config choices:**
  - Auth: OAuth2 Slack credential “Slack account”
  - Channel: dynamic `{{ $('Workflow Configuration').first().json.slackChannelId }}`
  - Message text renders:
    - overallStatus, escalationReason
    - list of conflicts (`map`)
    - action items
    - affected courses (flatten)
- **Connections:** Output → Log Results
- **Edge cases / failures:** Invalid channel ID, missing scopes, message formatting errors if conflicts/actionItems are undefined; Slack rate limits.

#### Node: Email Scheduling Recommendations
- **Type / role:** `Email Send` — sends an HTML report to scheduling team.
- **Config choices:**
  - To: `{{ $('Workflow Configuration').first().json.schedulingTeamEmail }}`
  - Subject: `Classroom Scheduling Recommendations - {{ overallStatus }}`
  - Rich HTML including:
    - status badge (CSS class based on lowercased overallStatus)
    - schedule table
    - conflict blocks with resolution options
    - action items list
    - generated timestamp `{{ $now.toISO() }}`
  - From: placeholder sender email
- **Connections:** Output → Log Results
- **Edge cases / failures:** SMTP/transport not configured, invalid from address, large HTML, undefined arrays causing template errors.

#### Node: Log Results
- **Type / role:** `Code` — logs summary to console and returns a structured log entry.
- **Logic summary:**
  - Aggregates items and prints:
    - status, escalationRequired, conflictsCount, scheduledCourses
  - Returns `{ timestamp, workflowRun, totalItems, results[] }`
- **Connections:** Terminal node (no outputs).
- **Edge cases / failures:** If upstream output differs, optional chaining prevents most crashes; but logs may not reflect real scheduling if earlier routing was inconsistent.

**Sticky note context (applies to this block’s nodes):**  
“## Report Generation & Distribution — **Why**: Delivers actionable intelligence…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Time-based entry point (06:00) | — | Workflow Configuration | ## Automated Data Collection  \n**Why**: Schedule trigger initiates competitor data fetching at defined intervals to ensure continuous market monitoring without manual intervention, capturing time-sensitive competitive moves |
| Workflow Configuration | Set | Central configuration constants and thresholds | Schedule Trigger | Prepare Scheduling Data | ## Automated Data Collection  \n**Why**: Schedule trigger initiates competitor data fetching at defined intervals to ensure continuous market monitoring without manual intervention, capturing time-sensitive competitive moves |
| Prepare Scheduling Data | Set | Creates classrooms/faculty/requests datasets | Workflow Configuration | Resource Analysis Agent; Calculate Historical Patterns | ## Automated Data Collection  \n**Why**: Schedule trigger initiates competitor data fetching at defined intervals to ensure continuous market monitoring without manual intervention, capturing time-sensitive competitive moves |
| OpenAI Model - Resource Agent | OpenAI Chat Model | LLM backend for Resource Analysis Agent | — | Resource Analysis Agent | ## Multi-Agent AI Analysis  \n**Why**: Delivers expert-level analysis across multiple dimensions simultaneously, uncovering hidden patterns humans miss |
| Resource Analysis Output Parser | Structured Output Parser | Enforces resource analysis JSON schema | — | Resource Analysis Agent | ## Multi-Agent AI Analysis  \n**Why**: Delivers expert-level analysis across multiple dimensions simultaneously, uncovering hidden patterns humans miss |
| Resource Analysis Agent | LangChain Agent | Capacity/equipment/faculty feasibility + conflict detection | Prepare Scheduling Data | Operations Agent; Merge Analysis Results | ## Multi-Agent AI Analysis  \n**Why**: Delivers expert-level analysis across multiple dimensions simultaneously, uncovering hidden patterns humans miss |
| OpenAI Model - Operations Agent | OpenAI Chat Model | LLM backend for Operations Agent | — | Operations Agent | ## Multi-Agent AI Analysis  \n**Why**: Delivers expert-level analysis across multiple dimensions simultaneously, uncovering hidden patterns humans miss |
| Operations Output Parser | Structured Output Parser | Enforces scheduling recommendations schema | — | Operations Agent | ## Multi-Agent AI Analysis  \n**Why**: Delivers expert-level analysis across multiple dimensions simultaneously, uncovering hidden patterns humans miss |
| Operations Agent | LangChain Agent | Creates schedule proposal, conflicts, escalation, actions | Resource Analysis Agent | Route by Severity; Merge Analysis Results | ## Multi-Agent AI Analysis  \n**Why**: Delivers expert-level analysis across multiple dimensions simultaneously, uncovering hidden patterns humans miss |
| Calculate Historical Patterns | Code | Simulated historical utilization/preferences/trends | Prepare Scheduling Data | Merge Analysis Results | ## Multi-Agent AI Analysis  \n**Why**: Delivers expert-level analysis across multiple dimensions simultaneously, uncovering hidden patterns humans miss |
| Merge Analysis Results | Merge | Combines resource + history + operations into one payload | Resource Analysis Agent; Calculate Historical Patterns; Operations Agent | Master Orchestrator Agent | ## Multi-Agent AI Analysis  \n**Why**: Delivers expert-level analysis across multiple dimensions simultaneously, uncovering hidden patterns humans miss |
| OpenAI Model - Master Orchestrator | OpenAI Chat Model | LLM backend for master orchestration | — | Master Orchestrator Agent | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Master Orchestrator Output Parser | Structured Output Parser | Enforces finalDecision schema | — | Master Orchestrator Agent | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Schedule Optimization Agent Tool | Agent Tool | Tool callable by master agent for optimization | — | Master Orchestrator Agent | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| OpenAI Model - Optimization Agent | OpenAI Chat Model | LLM backend for optimization tool | — | Schedule Optimization Agent Tool | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Optimization Output Parser | Structured Output Parser | Enforces optimizedSchedule schema | — | Schedule Optimization Agent Tool | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Conflict Score Calculator Tool | Code Tool | Computes weighted conflict scores + risk level | — | Master Orchestrator Agent | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Google Calendar Tool | Google Calendar Tool | Tool callable by master agent to fetch calendar events | — | Master Orchestrator Agent | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Master Orchestrator Agent | LangChain Agent | Synthesizes, calls tools, decides auto-resolve vs escalate | Merge Analysis Results | Check Auto-Resolution Possible | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Check Auto-Resolution Possible | If | Branches based on finalDecision.canAutoResolve & confidence | Master Orchestrator Agent | Apply Auto-Resolution; Generate Conflict Report | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Apply Auto-Resolution | Set | Annotates result as auto-resolved with actions/confidence | Check Auto-Resolution Possible (true) | Merge Resolution Paths | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Generate Conflict Report | Code | Builds escalation report for human review | Check Auto-Resolution Possible (false) | Merge Resolution Paths | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Merge Resolution Paths | Merge | Rejoins auto/manual paths | Apply Auto-Resolution; Generate Conflict Report | Route by Severity | ## Intelligent Routing & Validation  \n**Why**: Ensures data quality and prioritizes critical insights requiring immediate strategic response |
| Route by Severity | Switch | Routes to Slack vs Email by escalationRequired | Operations Agent; Merge Resolution Paths | Notify Critical Conflicts; Email Scheduling Recommendations | ## Report Generation & Distribution  \n**Why**: Delivers actionable intelligence to decision-makers instantly for rapid strategic pivots |
| Notify Critical Conflicts | Slack | Sends critical conflicts alert to Slack channel | Route by Severity (Critical) | Log Results | ## Report Generation & Distribution  \n**Why**: Delivers actionable intelligence to decision-makers instantly for rapid strategic pivots |
| Email Scheduling Recommendations | Email Send | Emails HTML schedule report to scheduling team | Route by Severity (Standard) | Log Results | ## Report Generation & Distribution  \n**Why**: Delivers actionable intelligence to decision-makers instantly for rapid strategic pivots |
| Log Results | Code | Console logs execution summary and returns log object | Notify Critical Conflicts; Email Scheduling Recommendations | — | ## Report Generation & Distribution  \n**Why**: Delivers actionable intelligence to decision-makers instantly for rapid strategic pivots |
| Sticky Note | Sticky Note | Canvas note (setup steps) | — | — |  |
| Sticky Note1 | Sticky Note | Canvas note (prerequisites/use cases) | — | — |  |
| Sticky Note2 | Sticky Note | Canvas note (how it works) | — | — |  |
| Sticky Note3 | Sticky Note | Canvas note (report distribution) | — | — |  |
| Sticky Note4 | Sticky Note | Canvas note (multi-agent analysis) | — | — |  |
| Sticky Note5 | Sticky Note | Canvas note (automated collection) | — | — |  |
| Sticky Note6 | Sticky Note | Canvas note (routing & validation) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: *AI-Driven Classroom Scheduling and Conflict Resolution System* (or your preferred name).
   - Keep it inactive until credentials and placeholders are set.

2. **Add trigger**
   - Node: **Schedule Trigger**
   - Configure: run daily at **06:00** (set the appropriate interval/time in the node).
   - Connect → **Workflow Configuration**

3. **Add configuration node**
   - Node: **Set** → name it **Workflow Configuration**
   - Add fields:
     - `slackChannelId` (string)
     - `schedulingTeamEmail` (string)
     - `maxClassroomCapacity` (number, e.g. 50)
     - `minBreakBetweenClasses` (number, e.g. 15)
     - `conflictThreshold` (string, e.g. "high")
     - `googleCalendarId` (string)
     - `autoResolutionThreshold` (number, e.g. 0.85)
     - `historicalDataDays` (number, e.g. 90)
   - Enable “Include Other Fields” (pass-through).
   - Connect → **Prepare Scheduling Data**

4. **Add scheduling data preparation**
   - Node: **Set** → name **Prepare Scheduling Data**
   - Add fields (type “Array”) and populate:
     - `classrooms[]` objects with `id`, `capacity`, `equipment[]`, `building`
     - `faculty[]` objects with `id`, `name`, `department`, `maxHoursPerDay`, `preferredTimeSlots[]`
     - `timetableRequests[]` objects with `courseId`, `facultyId`, `requiredCapacity`, `duration`, `preferredTime`, `equipment[]`
   - Enable “Include Other Fields”.
   - Create two outgoing connections:
     - → **Resource Analysis Agent**
     - → **Calculate Historical Patterns**

5. **Add Resource Analysis Agent (LangChain)**
   - Node: **AI Agent (LangChain Agent)** → name **Resource Analysis Agent**
   - Set “Prompt type” to define your prompt.
   - In the **text** prompt, include:
     - JSON.stringify of classrooms, faculty, timetableRequests
     - constraints from config using expressions like `$('Workflow Configuration').first().json.minBreakBetweenClasses`
   - Add **System Message** enforcing the rules (10% safety capacity buffer, breaks, equipment, etc.).
   - Attach:
     - **OpenAI Chat Model** node named **OpenAI Model - Resource Agent** (model gpt-4o, temperature 0.2) via the agent’s *Language Model* connection.
     - **Structured Output Parser** node named **Resource Analysis Output Parser** via the agent’s *Output Parser* connection; paste the resource schema.
   - Connect outputs:
     - Resource Analysis Agent → **Operations Agent**
     - Resource Analysis Agent → **Merge Analysis Results** (will create later)

6. **Add Operations Agent (LangChain)**
   - Node: **AI Agent** → name **Operations Agent**
   - Prompt includes:
     - Resource analysis results from current `$json`
     - Original requests from `$('Prepare Scheduling Data').first().json.timetableRequests`
   - System message: define severity scheme, escalation rules, output requirements.
   - Attach:
     - **OpenAI Model - Operations Agent** (gpt-4o, temp 0.2)
     - **Operations Output Parser** (structured schema for schedulingRecommendations)
   - Connect outputs:
     - Operations Agent → **Route by Severity** (create later)
     - Operations Agent → **Merge Analysis Results** (input 3)

7. **Add historical analysis**
   - Node: **Code** → name **Calculate Historical Patterns**
   - Implement logic to produce `historicalPatterns` (or replace with DB query in production).
   - Read `historicalDataDays` from Workflow Configuration using an expression.
   - Connect → **Merge Analysis Results** (input 2)

8. **Merge the three analysis streams**
   - Node: **Merge** → name **Merge Analysis Results**
   - Set **Number of Inputs = 3**
   - Connect:
     1. Resource Analysis Agent → Merge (input 1)
     2. Calculate Historical Patterns → Merge (input 2)
     3. Operations Agent → Merge (input 3)
   - Connect Merge → **Master Orchestrator Agent**

9. **Create tool nodes for the orchestrator**
   - **Conflict Score Calculator Tool**
     - Node: **AI Tool (Code Tool)** (LangChain tool code)
     - Implement the weighted scoring logic.
   - **Google Calendar Tool**
     - Node: **Google Calendar Tool**
     - Operation: **Get All**
     - Calendar: expression `{{ $('Workflow Configuration').first().json.googleCalendarId }}`
     - Credentials: set Google OAuth2 credentials with Calendar read scope.
   - **Schedule Optimization Agent Tool**
     - Node: **AI Agent Tool**
     - Configure input mapping using `$fromAI('scheduleData', ..., 'json')`
     - Add tool system message describing optimization behavior.
     - Attach:
       - **OpenAI Model - Optimization Agent** (gpt-4o, temp 0.3)
       - **Optimization Output Parser** (optimizedSchedule schema)

10. **Add Master Orchestrator Agent**
   - Node: **AI Agent** → name **Master Orchestrator Agent**
   - Prompt text should reference merged inputs (resource/history/operations). If using a Merge node that outputs an array, use index references consistently.
   - System message: instruct it to call the three tools, validate calendar availability, and produce a `finalDecision`.
   - Attach:
     - **OpenAI Model - Master Orchestrator** (gpt-4o, temp 0.1)
     - **Master Orchestrator Output Parser** (finalDecision schema)
   - Connect tool nodes to the agent through their **ai_tool** connections:
     - Schedule Optimization Agent Tool → Master Orchestrator Agent
     - Conflict Score Calculator Tool → Master Orchestrator Agent
     - Google Calendar Tool → Master Orchestrator Agent
   - Connect Master Orchestrator Agent → **Check Auto-Resolution Possible**

11. **Add decision branch**
   - Node: **IF** → name **Check Auto-Resolution Possible**
   - Conditions (AND):
     - `output.finalDecision.canAutoResolve` equals true
     - `output.finalDecision.confidence` >= 0.85 (or reference config threshold if you improve it)
   - True output → **Apply Auto-Resolution**
   - False output → **Generate Conflict Report**

12. **Add auto-resolution marker**
   - Node: **Set** → name **Apply Auto-Resolution**
   - Set: `resolutionType`, `resolutionTimestamp`, `appliedActions`, `autoResolutionConfidence`
   - Include other fields enabled.
   - Connect → **Merge Resolution Paths**

13. **Add escalation report**
   - Node: **Code** → name **Generate Conflict Report**
   - Build `conflictReport` object with score/risk/reason/actions/priority.
   - Connect → **Merge Resolution Paths**

14. **Merge resolution paths**
   - Node: **Merge** → name **Merge Resolution Paths**
   - Connect:
     - Apply Auto-Resolution → Merge input 1
     - Generate Conflict Report → Merge input 2
   - Connect Merge → **Route by Severity**

15. **Add routing**
   - Node: **Switch** → name **Route by Severity**
   - Rule 1 (“Critical”): `output.schedulingRecommendations.escalationRequired` is true
   - Rule 2 (“Standard”): same field is false
   - Connect:
     - Critical → **Notify Critical Conflicts**
     - Standard → **Email Scheduling Recommendations**

16. **Add Slack notification**
   - Node: **Slack**
   - Auth: OAuth2 Slack credential
   - Channel: expression `{{ $('Workflow Configuration').first().json.slackChannelId }}`
   - Message: format from `output.schedulingRecommendations` (conflicts/actionItems etc.)
   - Connect → **Log Results**

17. **Add email report**
   - Node: **Email Send**
   - Configure SMTP/Email credentials as required by your n8n setup.
   - To: expression `{{ $('Workflow Configuration').first().json.schedulingTeamEmail }}`
   - From: set a valid sender.
   - Subject and HTML: render schedule/conflicts/action items from `output.schedulingRecommendations`.
   - Connect → **Log Results**

18. **Add logging**
   - Node: **Code** → name **Log Results**
   - Implement console summary and return `logEntry`.

19. **Credentials checklist**
   - OpenAI API credential added to all OpenAI model nodes.
   - Slack OAuth2 credential for Slack node.
   - Google Calendar OAuth2 credential for Google Calendar Tool.
   - Email transport configured for Email Send node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Setup Steps… Connect Schedule Trigger… Add OpenAI API keys… Link Google Sheets… Configure Gmail… Set up Slack/Discord…” | Canvas sticky note (generic setup guidance; mentions nodes not present like Google Sheets/Gmail). |
| “## Prerequisites… Use Cases… Benefits…” | Canvas sticky note (generic competitor-intel use case; not aligned to classroom scheduling). |
| “## How It Works… competitive intelligence gathering…” | Canvas sticky note (describes competitor monitoring, not classroom scheduling). |
| “## Report Generation & Distribution — Why: Delivers actionable intelligence…” | Canvas sticky note near notification/report nodes. |
| “## Multi-Agent AI Analysis — Why: …uncovering hidden patterns…” | Canvas sticky note near AI analysis nodes. |
| “## Intelligent Routing & Validation — Why: Ensures data quality…” | Canvas sticky note near orchestration/decision/routing nodes. |

