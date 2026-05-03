Generate DraftKings DFS lineups with GPT-4o, Google Sheets, email and Slack alerts

https://n8nworkflows.xyz/workflows/generate-draftkings-dfs-lineups-with-gpt-4o--google-sheets--email-and-slack-alerts-12928


# Generate DraftKings DFS lineups with GPT-4o, Google Sheets, email and Slack alerts

## 1. Workflow Overview

**Purpose:** This workflow automates daily DraftKings DFS lineup generation: it retrieves contest data, filters for suitable contests, fetches player data for eligible slates, computes player “value” metrics, uses GPT‑4o via an AI Agent to generate lineups, logs results to Google Sheets, then distributes them via email and Slack.

**Target use cases:**
- Daily DFS lineup production for a team/community
- Automated slate selection based on contest attributes and start time
- Consistent distribution + audit trail (Sheets)

### 1.1 Scheduling & Run Configuration
Trigger the workflow on a schedule and set run-level variables/settings.

### 1.2 Contest Collection & Normalization
Pull contest listings from DraftKings (via HTTP) and standardize the raw response into a predictable schema.

### 1.3 Contest Filtering & Routing
Apply business rules (entry fee/prize pool, then start time). Eligible contests continue to player fetch; ineligible contests are logged as skipped.

### 1.4 Player Data Pipeline & Scoring
Fetch player data for the selected contest/slate, normalize it, and compute a “value” metric used downstream by the AI generator.

### 1.5 AI Lineup Generation (GPT‑4o)
Use an n8n LangChain Agent powered by an OpenAI Chat model node to produce DraftKings lineups from the prepared player pool.

### 1.6 Output & Notifications
Write generated lineups to Google Sheets, email subscribers, then post a Slack notification.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Run Configuration

**Overview:** Starts the workflow automatically and prepares configuration variables that downstream nodes rely on (dates, thresholds, sheet IDs, etc.).  
**Nodes involved:** `Daily DFS Schedule`, `Workflow Configuration`

#### Node: Daily DFS Schedule
- **Type / role:** `Schedule Trigger` — entry point that runs on a time schedule.
- **Configuration (interpreted):** Schedule is not specified in the JSON (parameters empty). In n8n UI this would be set to a cron/interval/time-of-day.
- **Connections:**
  - **Out:** `Workflow Configuration`
- **Edge cases / failures:**
  - Misconfigured schedule (never runs / runs too often).
  - Timezone mismatches (n8n instance timezone vs intended DFS slate timezone).
- **Version notes:** typeVersion `1.3` (modern schedule trigger behavior; UI differs from older Cron node patterns).

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines workflow constants and computed variables.
- **Configuration (interpreted):** Parameters are empty in the exported JSON; in a complete build this node typically sets fields like:
  - date/slate key
  - contest thresholds (min/max entry fee, min prize pool)
  - time window for start time filtering
  - endpoints and IDs (sheet ID, email list, Slack channel, etc.)
- **Connections:**
  - **In:** `Daily DFS Schedule`
  - **Out:** `DraftKings Contest Scraper`
- **Edge cases / failures:**
  - Missing fields referenced later by expressions (causes expression evaluation errors).
- **Version notes:** typeVersion `3.4` (current Set node behavior, including field UI and keep/remove options).

---

### Block 2 — Contest Collection & Normalization

**Overview:** Pulls DraftKings contest listing data, then restructures it into a consistent schema for filtering.  
**Nodes involved:** `DraftKings Contest Scraper`, `Normalize Contest Data`

#### Node: DraftKings Contest Scraper
- **Type / role:** `HTTP Request` — calls a DraftKings contest endpoint (or a proxy/scraper API).
- **Configuration (interpreted):** Parameters are empty in JSON; normally includes:
  - URL for contest listing
  - method `GET`
  - headers (user-agent, auth/cookies if required)
  - response format JSON
- **Connections:**
  - **In:** `Workflow Configuration`
  - **Out:** `Normalize Contest Data`
- **Edge cases / failures:**
  - 401/403 due to DraftKings protections, bot detection, missing cookies, geo restrictions.
  - Response shape changes break normalization downstream.
  - Rate limiting (429) and transient network timeouts.
- **Version notes:** typeVersion `4.3` (HTTP Request options like pagination, response parsing, proxy, retry settings).

#### Node: Normalize Contest Data
- **Type / role:** `Set` — maps raw contest response into standardized fields.
- **Configuration (interpreted):** Typically extracts fields like:
  - contest ID, contest name
  - entry fee
  - total prize pool
  - start time / slate start timestamp
  - sport/contest type
- **Connections:**
  - **In:** `DraftKings Contest Scraper`
  - **Out:** `Contest Filter: Entry Fee & Prize Pool`
- **Edge cases / failures:**
  - Missing keys in API response → undefined fields → IF filters behave unexpectedly.
- **Version notes:** typeVersion `3.4`.

---

### Block 3 — Contest Filtering & Routing

**Overview:** Applies contest eligibility rules and routes eligible contests to player fetching. Skipped contests are captured for visibility.  
**Nodes involved:** `Contest Filter: Entry Fee & Prize Pool`, `Contest Filter: Start Time`, `Log Skipped Contests`

#### Node: Contest Filter: Entry Fee & Prize Pool
- **Type / role:** `IF` — filters contests based on financial criteria.
- **Configuration (interpreted):** Parameters empty in JSON; typically conditions like:
  - Entry fee between min/max
  - Prize pool above minimum
- **Connections:**
  - **In:** `Normalize Contest Data`
  - **True out (0):** `Contest Filter: Start Time`
  - **False out (1):** `Log Skipped Contests`
- **Edge cases / failures:**
  - Entry fee/prize values as strings (e.g., “$10”) causing numeric comparison issues unless parsed.
  - Currency formatting and locale differences.
- **Version notes:** typeVersion `2.3`.

#### Node: Contest Filter: Start Time
- **Type / role:** `IF` — ensures the contest start time is within an intended window.
- **Configuration (interpreted):** Often checks:
  - Start time is today
  - Start time is after “now + buffer” (avoid late swaps / too close to lock)
- **Connections:**
  - **In:** `Contest Filter: Entry Fee & Prize Pool` (true path)
  - **Output 0:** `Fetch Player Data`
  - **Output 1:** `Log Skipped Contests`
- **Edge cases / failures:**
  - Time parsing issues (ISO vs epoch vs localized string).
  - Timezone misalignment causing wrong routing.
- **Version notes:** typeVersion `2.3`.

#### Node: Log Skipped Contests
- **Type / role:** `Set` — records why a contest was skipped (for debugging/monitoring).
- **Configuration (interpreted):** Parameters empty; typically sets fields like:
  - contestId, contestName, reason (“entry fee out of range”, “start time too early”)
  - evaluated thresholds
- **Connections:**
  - **In:** from `Contest Filter: Entry Fee & Prize Pool` (false) and `Contest Filter: Start Time` (false)
  - **Out:** none (terminal branch)
- **Edge cases / failures:**
  - If this is meant to persist logs, a Set node alone won’t store anything; it needs a downstream sink (Sheets/DB/Slack). As-is, it only reshapes data and then ends.
- **Version notes:** typeVersion `3.4`.

---

### Block 4 — Player Data Pipeline & Scoring

**Overview:** Retrieves eligible-slate player data, normalizes it, then computes derived “value” metrics used by the lineup generator.  
**Nodes involved:** `Fetch Player Data`, `Normalize Player Data`, `Calculate Player Value`

#### Node: Fetch Player Data
- **Type / role:** `HTTP Request` — pulls player pool/salaries/projections for the contest/slate.
- **Configuration (interpreted):** Parameters empty; typically:
  - URL derived from contest/slate ID
  - JSON response parsing
- **Connections:**
  - **In:** `Contest Filter: Start Time` (eligible path)
  - **Out:** `Normalize Player Data`
- **Edge cases / failures:**
  - Endpoint auth/geo restrictions, 403/401
  - Response schema drift
  - Large payload sizes causing memory/time issues
- **Version notes:** typeVersion `4.3`.

#### Node: Normalize Player Data
- **Type / role:** `Set` — standardizes player records.
- **Configuration (interpreted):** Usually outputs fields like:
  - playerId, name, team, position(s)
  - salary
  - projection/avg points (if available)
  - game info, start time
- **Connections:**
  - **In:** `Fetch Player Data`
  - **Out:** `Calculate Player Value`
- **Edge cases / failures:**
  - Multi-position handling (e.g., “PG/SG”) for DraftKings roster slots.
  - Missing projection data → value calc must handle nulls.
- **Version notes:** typeVersion `3.4`.

#### Node: Calculate Player Value
- **Type / role:** `Code` — computes metrics used for optimization (e.g., points per $1k).
- **Configuration (interpreted):** Code is not present in export (parameters empty). Common outputs:
  - `value = projection / (salary/1000)`
  - ceiling/floor heuristics
  - flags for injury/news
- **Connections:**
  - **In:** `Normalize Player Data`
  - **Out:** `AI Lineup Generator`
- **Edge cases / failures:**
  - Division by zero (salary missing/0).
  - Type coercion issues (string numbers).
  - Code node runtime errors stop execution.
- **Version notes:** typeVersion `2` (JavaScript code node).

---

### Block 5 — AI Lineup Generation (GPT‑4o)

**Overview:** Uses a LangChain Agent to generate DFS lineups based on the prepared player dataset, with GPT‑4o as the underlying chat model.  
**Nodes involved:** `AI Lineup Generator`, `OpenAI GPT-4`

#### Node: OpenAI GPT-4
- **Type / role:** `lmChatOpenAi` — provides the chat model (GPT‑4o implied by node naming; actual model is set in node parameters, not shown here).
- **Configuration (interpreted):**
  - Requires OpenAI credentials in n8n
  - Typical settings: model (e.g., `gpt-4o`), temperature, max tokens
- **Connections:**
  - **Out (AI language model):** to `AI Lineup Generator` via `ai_languageModel`
- **Edge cases / failures:**
  - Auth errors (missing/invalid API key).
  - Model not available in account/region.
  - Token limits if player pool is large; needs batching or summarization.
- **Version notes:** typeVersion `1.3` (LangChain OpenAI chat integration).

#### Node: AI Lineup Generator
- **Type / role:** `LangChain Agent` — orchestrates prompts/tools (tools not shown) to output lineups.
- **Configuration (interpreted):**
  - Uses `OpenAI GPT-4` as its language model input (wired via ai port)
  - Likely builds prompts that include roster constraints (DK salary cap, positions, max players per team, etc.)
  - No tools are defined in JSON; could be prompt-only generation.
- **Connections:**
  - **In (main):** `Calculate Player Value`
  - **In (ai_languageModel):** `OpenAI GPT-4`
  - **Out:** `Log Lineups to Google Sheets`
- **Edge cases / failures:**
  - Agent output formatting inconsistencies (needs strict schema for Sheets/email).
  - Hallucinated players if prompt doesn’t constrain to provided pool.
  - If multiple contests are processed, agent may need to iterate per contest (not visible here).
- **Version notes:** typeVersion `3` (Agent node behavior and inputs changed across versions).

---

### Block 6 — Output & Notifications

**Overview:** Persists generated lineups, emails subscribers, then alerts the team on Slack.  
**Nodes involved:** `Log Lineups to Google Sheets`, `Email Lineups to Subscribers`, `Notify Team on Slack`

#### Node: Log Lineups to Google Sheets
- **Type / role:** `Google Sheets` — writes lineup outputs to a spreadsheet.
- **Configuration (interpreted):** Parameters empty; typically:
  - operation: Append / Add row(s)
  - spreadsheet ID + sheet/tab name
  - column mapping (contest, timestamp, lineup1..N, salary, projected points)
- **Connections:**
  - **In:** `AI Lineup Generator`
  - **Out:** `Email Lineups to Subscribers`
- **Credentials:** Google OAuth2 or Service Account credentials required.
- **Edge cases / failures:**
  - Permission denied to sheet
  - Column mismatch / invalid range
  - Rate limits on Sheets API
- **Version notes:** typeVersion `4.7`.

#### Node: Email Lineups to Subscribers
- **Type / role:** `Email Send` — sends lineups to a subscriber list.
- **Configuration (interpreted):** Parameters empty; typically:
  - SMTP credentials (or integrated email provider)
  - to/cc/bcc list, subject, HTML/text body
  - includes lineup content from upstream node
- **Connections:**
  - **In:** `Log Lineups to Google Sheets`
  - **Out:** `Notify Team on Slack`
- **Credentials:** SMTP/Email account configuration in n8n.
- **Edge cases / failures:**
  - SMTP auth failures, provider blocks, rate limits
  - Invalid subscriber addresses causing partial send failures
- **Version notes:** typeVersion `2.1`.

#### Node: Notify Team on Slack
- **Type / role:** `Slack` — posts a notification to Slack after emailing.
- **Configuration (interpreted):** Parameters empty; typically:
  - operation: Post message
  - channel and message content (summary + Sheets link)
- **Connections:**
  - **In:** `Email Lineups to Subscribers`
  - **Out:** none (terminal)
- **Credentials:** Slack OAuth token / bot token in n8n.
- **Edge cases / failures:**
  - Missing channel permissions
  - Slack rate limits
  - Message formatting issues (blocks vs text)
- **Version notes:** typeVersion `2.4`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily DFS Schedule | scheduleTrigger | Scheduled entry point | — | Workflow Configuration |  |
| Workflow Configuration | set | Set run constants/variables | Daily DFS Schedule | DraftKings Contest Scraper |  |
| DraftKings Contest Scraper | httpRequest | Fetch contest list | Workflow Configuration | Normalize Contest Data |  |
| Normalize Contest Data | set | Standardize contest fields | DraftKings Contest Scraper | Contest Filter: Entry Fee & Prize Pool |  |
| Contest Filter: Entry Fee & Prize Pool | if | Filter contests by fee/pool | Normalize Contest Data | Contest Filter: Start Time; Log Skipped Contests |  |
| Contest Filter: Start Time | if | Filter contests by start time | Contest Filter: Entry Fee & Prize Pool | Fetch Player Data; Log Skipped Contests |  |
| Fetch Player Data | httpRequest | Fetch slate/player pool | Contest Filter: Start Time | Normalize Player Data |  |
| Normalize Player Data | set | Standardize player fields | Fetch Player Data | Calculate Player Value |  |
| Calculate Player Value | code | Compute derived metrics | Normalize Player Data | AI Lineup Generator |  |
| OpenAI GPT-4 | lmChatOpenAi | LLM provider for agent | — | AI Lineup Generator (ai_languageModel) |  |
| AI Lineup Generator | agent | Generate DFS lineups via LLM | Calculate Player Value; OpenAI GPT-4 (ai) | Log Lineups to Google Sheets |  |
| Log Lineups to Google Sheets | googleSheets | Persist lineups | AI Lineup Generator | Email Lineups to Subscribers |  |
| Email Lineups to Subscribers | emailSend | Email distribution | Log Lineups to Google Sheets | Notify Team on Slack |  |
| Notify Team on Slack | slack | Team alert | Email Lineups to Subscribers | — |  |
| Log Skipped Contests | set | Track contests filtered out | Contest Filter: Entry Fee & Prize Pool; Contest Filter: Start Time | — |  |
| Sticky Note | stickyNote | Comment container | — | — |  |
| Sticky Note1 | stickyNote | Comment container | — | — |  |
| Sticky Note2 | stickyNote | Comment container | — | — |  |
| Sticky Note3 | stickyNote | Comment container | — | — |  |
| Sticky Note4 | stickyNote | Comment container | — | — |  |

> Note: All sticky notes have empty content in this workflow export, so the “Sticky Note” column remains blank for all nodes.

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add **Schedule Trigger** named `Daily DFS Schedule`.  
   2) Configure the desired run time (daily before slate lock; set correct timezone).

2. **Add configuration node**
   1) Add **Set** named `Workflow Configuration`.  
   2) Add fields you will reference later, for example:
      - `minEntryFee`, `maxEntryFee`, `minPrizePool`
      - `startTimeWindowStart`, `startTimeWindowEnd` (or a “min minutes before lock”)
      - `contestListUrl`, `playerDataUrlTemplate`
      - `spreadsheetId`, `sheetName`
      - `emailTo` (or list), `slackChannel`
   3) Connect: `Daily DFS Schedule` → `Workflow Configuration`.

3. **Fetch contests**
   1) Add **HTTP Request** named `DraftKings Contest Scraper`.  
   2) Set method `GET`, URL = your contest endpoint (often from `Workflow Configuration`).  
   3) Set response to JSON; add headers/auth as needed.  
   4) Connect: `Workflow Configuration` → `DraftKings Contest Scraper`.

4. **Normalize contest output**
   1) Add **Set** named `Normalize Contest Data`.  
   2) Map raw response fields to normalized keys (contest id/name/fee/pool/startTime).  
   3) Connect: `DraftKings Contest Scraper` → `Normalize Contest Data`.

5. **Filter by entry fee & prize pool**
   1) Add **IF** named `Contest Filter: Entry Fee & Prize Pool`.  
   2) Add conditions using normalized numeric fields (parse numbers if needed).  
   3) Connect: `Normalize Contest Data` → `Contest Filter: Entry Fee & Prize Pool`.

6. **Filter by start time**
   1) Add **IF** named `Contest Filter: Start Time`.  
   2) Add conditions comparing contest start time to your desired window (ensure timezone handling).  
   3) Connect: `Contest Filter: Entry Fee & Prize Pool` (true output) → `Contest Filter: Start Time`.

7. **Skipped contest logging**
   1) Add **Set** named `Log Skipped Contests`.  
   2) Set fields like `contestId`, `contestName`, `skipReason` (you can hardcode per branch or use separate nodes per reason).  
   3) Connect:
      - `Contest Filter: Entry Fee & Prize Pool` (false output) → `Log Skipped Contests`
      - `Contest Filter: Start Time` (false output) → `Log Skipped Contests`
   4) (Recommended) Add a sink after it (Sheets/DB/Slack) if you want persistence.

8. **Fetch player data**
   1) Add **HTTP Request** named `Fetch Player Data`.  
   2) Configure URL using contest/slate identifiers from normalized contest item.  
   3) Connect: `Contest Filter: Start Time` (true output) → `Fetch Player Data`.

9. **Normalize player data**
   1) Add **Set** named `Normalize Player Data`.  
   2) Output consistent fields: `name`, `positions`, `team`, `salary`, `projection` (if available).  
   3) Connect: `Fetch Player Data` → `Normalize Player Data`.

10. **Compute player value**
   1) Add **Code** named `Calculate Player Value`.  
   2) Implement value calculation and guardrails (salary/projection null checks).  
   3) Connect: `Normalize Player Data` → `Calculate Player Value`.

11. **Add OpenAI model node**
   1) Add **OpenAI Chat Model** node (LangChain) named `OpenAI GPT-4`.  
   2) Select model **gpt-4o** (or your preferred).  
   3) Configure credentials (OpenAI API key) in n8n credentials manager.

12. **Add AI agent to generate lineups**
   1) Add **AI Agent** node named `AI Lineup Generator`.  
   2) Connect model: `OpenAI GPT-4` → `AI Lineup Generator` via the **AI Language Model** connection.  
   3) Connect data: `Calculate Player Value` → `AI Lineup Generator` (main).  
   4) In the agent prompt/instructions, enforce:
      - DraftKings roster rules (positions, salary cap, roster size)
      - “Use only players provided”
      - Output strict JSON or a table structure for downstream parsing/logging

13. **Log to Google Sheets**
   1) Add **Google Sheets** node named `Log Lineups to Google Sheets`.  
   2) Choose operation (typically **Append**).  
   3) Configure credentials (Google OAuth2/service account), spreadsheet ID, sheet name, and column mapping.  
   4) Connect: `AI Lineup Generator` → `Log Lineups to Google Sheets`.

14. **Email subscribers**
   1) Add **Email Send** node named `Email Lineups to Subscribers`.  
   2) Configure SMTP credentials, recipients, subject, body (inject lineups + sheet link).  
   3) Connect: `Log Lineups to Google Sheets` → `Email Lineups to Subscribers`.

15. **Slack notification**
   1) Add **Slack** node named `Notify Team on Slack`.  
   2) Configure Slack credentials, channel, and a short summary message.  
   3) Connect: `Email Lineups to Subscribers` → `Notify Team on Slack`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided by the user with the workflow context |

