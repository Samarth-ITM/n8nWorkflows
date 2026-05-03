Monitor hotel competitor rates and answer WhatsApp Q&A using OpenAI GPT-4.1

https://n8nworkflows.xyz/workflows/monitor-hotel-competitor-rates-and-answer-whatsapp-q-a-using-openai-gpt-4-1-12246


# Monitor hotel competitor rates and answer WhatsApp Q&A using OpenAI GPT-4.1

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow automates hotel competitor rate monitoring (via Amadeus Hotel Offers) for the next N nights, detects significant competitor price changes vs the previous snapshot, enriches alerts with **your own hotel’s rate** and **local demand context** (Vancouver Convention Centre events), then uses an **OpenAI agent** to produce an executive, action-oriented revenue recommendation and sends it via **WhatsApp**. A second branch provides an **interactive WhatsApp Q&A** experience grounded in the latest alert summary plus access to stored historical rate snapshots.

**Typical use cases**
- Revenue managers automating rate-shopping and intraday competitor move detection (09:00/15:00/21:00).
- Leadership receiving concise, decision-ready alerts on significant competitor changes.
- Follow-up questions on WhatsApp (“Is this trend persistent?”, “What happened last week?”) answered using stored rate history.

### 1.1 Scheduled Monitoring & Rate Collection (Top branch)
- Triggered 3× daily (cron).
- Fetch OAuth token, generate hotel×date windows, query Amadeus for offers (batched), normalize responses.

### 1.2 Snapshoting, History Archiving, and Change Detection
- Create stable keys (rateKey/historyKey), look up previous snapshot, compute deltas and “significant change” flags.
- Persist latest snapshot and append history.

### 1.3 Alert Enrichment (Our hotel + Events)
- For significant competitor changes only:
  - Fetch your hotel’s rate for the same date from latest snapshots.
  - Scrape Vancouver Convention Centre events (current month + next month) and extract/normalize event list.

### 1.4 AI Analysis, WhatsApp Alerting, and Persistence
- Bundle alerts + event summary into a compact agent input.
- OpenAI agent produces a structured JSON analysis (validated by a structured output parser).
- Send executive summary on WhatsApp and upsert the “latest alert” into a datastore.

### 1.5 WhatsApp Follow-Up Q&A (Bottom branch)
- WhatsApp trigger receives messages (optionally filtered by sender).
- Loads latest alert context and uses an OpenAI agent with memory + a Data Table tool to answer questions.
- Sends answer back via WhatsApp.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Intraday Trigger & Configuration
**Overview:** Defines monitoring parameters (hotels, window size, thresholds) and runs on a fixed schedule.  
**Nodes involved:** `Daily Market Check`, `Config`, `Amadeus OAuth`

#### Node: Daily Market Check
- **Type / role:** Schedule Trigger — entry point for the monitoring branch.
- **Configuration:** Three cron expressions: `09:00`, `15:00`, `21:00` (seconds included: `0 0 9 * * *`, etc.).
- **Outputs:** Connects to `Config`.
- **Edge cases:** Timezone depends on n8n instance settings; if instance timezone differs from Vancouver, schedule times may not align with local intent.

#### Node: Config
- **Type / role:** Set — central configuration object.
- **Key fields:**
  - `currency` (CAD), `adults` (2), `roomQuantity` (1)
  - `thresholdPct` (0.1 = 10% significance)
  - `daysAhead` (30)
  - `competitors` (array of `{name, hotelId}`)
  - `ourHotel` (object `{name, hotelId}`)
- **Outputs:** Connects to `Amadeus OAuth`.
- **Edge cases:** If `competitors` is not an array or contains missing `hotelId`, those entries are skipped later in code.

#### Node: Amadeus OAuth
- **Type / role:** HTTP Request — obtain access token for Amadeus API.
- **Configuration choices:**
  - POST `https://test.api.amadeus.com/v1/security/oauth2/token`
  - Form-URL-encoded body: `grant_type=client_credentials`, plus `client_id`, `client_secret` (expected to be provided as credentials or environment variables mapped into the node).
- **Outputs:** Connects to `Generate 30 night windows`.
- **Edge cases / failures:**
  - 401/403 if client_id/secret invalid.
  - Using `test.api.amadeus.com` implies sandbox; production uses a different base URL.
  - Token missing/expired will break downstream calls.

---

### Block 2.2 — Generate Hotel×Date Windows & Fetch Offers
**Overview:** Builds 30 one-night windows for each hotel (competitors + your hotel) and queries Amadeus Hotel Offers for each window with batching to control rate limits.  
**Nodes involved:** `Generate 30 night windows`, `Amadeus Hotel Offers`, `Combine 30-Day Rates`, `Clean Amadeus Data`

#### Node: Generate 30 night windows
- **Type / role:** Code — expands configuration into many request items.
- **Key logic:**
  - Reads config via `$('Config').first().json`
  - Reads token via `$('Amadeus OAuth').first().json.access_token`
  - Generates dates in `America/Vancouver` to avoid local drift.
  - Emits items for each `hotelId` and each date:
    - `hotelRole`: `"competitor"` or `"ourHotel"`
    - `checkInDate`, `checkOutDate`
    - config passthrough: `currency`, `adults`, `roomQuantity`, `thresholdPct`
    - `access_token` for downstream headers
- **Outputs:**
  - To `Amadeus Hotel Offers` (main API calls)
  - Also to `Combine 30-Day Rates` input 0 (used to preserve context when merging)
- **Edge cases:**
  - If token is missing, downstream Authorization header becomes invalid.
  - `cfg.timezone` is optional; defaults to `America/Vancouver`.

#### Node: Amadeus Hotel Offers
- **Type / role:** HTTP Request — fetch hotel offers per hotel×date item.
- **Configuration choices:**
  - GET `https://test.api.amadeus.com/v3/shopping/hotel-offers`
  - Query params from item: `hotelIds`, `checkInDate`, `checkOutDate`, `adults`, `roomQuantity`, `currency`, `paymentPolicy=NONE`
  - Header: `Authorization: Bearer {{$json.access_token}}`
  - **Batching enabled:** batch size 1, interval 1200ms (throttling).
  - `onError: continueRegularOutput` to keep pipeline running on errors.
- **Outputs:** To `Combine 30-Day Rates` input 1.
- **Edge cases / failures:**
  - 429 rate limiting (handled later by normalization).
  - 3664 “no offers” style errors (handled later).
  - Amadeus may return different structures; normalization node tries to handle stringified JSON too.

#### Node: Combine 30-Day Rates
- **Type / role:** Merge (Combine by Position) — reattaches request-context items with their corresponding API responses.
- **Configuration:** `mode=combine`, `combineByPosition`.
- **Inputs:**
  - Input 0: items from `Generate 30 night windows`
  - Input 1: responses from `Amadeus Hotel Offers`
- **Outputs:** To `Clean Amadeus Data`.
- **Edge cases:** Combine-by-position assumes both streams align 1:1 in the same order. If batching/retries reorder or drop items unexpectedly, misalignment can occur.

#### Node: Clean Amadeus Data
- **Type / role:** Code — normalize Amadeus response to a stable schema.
- **Key outputs produced:**
  - Preserved context: `hotelRole`, `hotelId`, `hotelName`, `checkInDate`, `checkOutDate`, `currency`, etc.
  - Normalized pricing: `minRate` (min of offer totals), `hasOffer`
  - Error semantics: `httpStatus`, `amadeusCode`, `errorTitle`
  - `outcome`: one of `OK`, `NO_OFFERS`, `RATE_LIMIT`, `AUTH_ERROR`, `ERROR`
- **Outputs:** To `New Snapshot`.
- **Edge cases:**
  - Handles “full response” vs “body-only”.
  - Attempts to parse stringified JSON fields.
  - If offers array missing or empty -> `NO_OFFERS` or `OK` with null rate, depending on status.

---

### Block 2.3 — Snapshot Keys, Previous Snapshot Lookup, and Storage
**Overview:** Creates stable keys for latest and historical tables, enriches with previous snapshot values, then persists latest snapshot and appends history.  
**Nodes involved:** `New Snapshot`, `Get Prev Rates`, `Prev Snapshot`, `Combine New Rates vs Prev Rates`, `Upsert Latest Rates`, `Insert History Rates`

#### Node: New Snapshot
- **Type / role:** Code (per item) — compute keys and timestamps.
- **Key logic:**
  - Timezone bucketing in `America/Vancouver`
  - `timeSlot` determined by local hour: `"09"`, `"15"`, `"21"`
  - `bestTotal` derived from `minRate`
  - Keys:
    - `rateKey = hotelId|checkInDate` (latest snapshot)
    - `historyKey = hotelId|checkInDate|timeSlot` (append-only)
  - Adds `observedAtUtc` (ISO) and `observedAtLocal` (string with tz).
- **Outputs:**
  - To `Get Prev Rates` (lookup)
  - Also directly to `Combine New Rates vs Prev Rates` input 0 (as “new” data)
- **Edge cases:**
  - If `minRate` null -> `bestTotal` null, later change computation yields no pctChange.
  - TimeSlot calculation depends on Vancouver time parsing via `Intl`.

#### Node: Get Prev Rates
- **Type / role:** Data Table (get) — fetch previous latest snapshot by `rateKey`.
- **Configuration:**
  - Data table: `Hotel_Rates`
  - Filter: `rateKey == {{$json.rateKey}}`
  - `limit: 1`
  - `alwaysOutputData: true` ensures downstream receives an item even when no match.
- **Outputs:** To `Prev Snapshot`.
- **Edge cases:** If table empty or no match, output may have no `id`; handled by `Prev Snapshot`.

#### Node: Prev Snapshot
- **Type / role:** Code (per item) — standardizes “previous snapshot” fields.
- **Adds fields:**
  - `prevExists` boolean
  - `prevId`, `prevBestTotal`, `prevCurrency`, `prevUpdatedAt`, `prevRateKey`
- **Outputs:** To `Combine New Rates vs Prev Rates` input 1.
- **Edge cases:** Coerces numeric `bestTotal`; safe when no row exists.

#### Node: Combine New Rates vs Prev Rates
- **Type / role:** Merge (Enrich Input1 by key) — combines current snapshot with previous snapshot fields.
- **Configuration:**
  - `joinMode: enrichInput1`
  - Match field: `rateKey`
  - Clash handling: shallow merge, prefer input1.
- **Inputs:**
  - Input 0: `New Snapshot` stream
  - Input 1: `Prev Snapshot` stream
- **Outputs:**
  - To `Upsert Latest Rates` (store latest)
  - To `Insert History Rates` (append history)
  - To `Compute Change` (change detection)
- **Edge cases:** If `rateKey` missing, join may not enrich; downstream change detection may be incomplete.

#### Node: Upsert Latest Rates
- **Type / role:** Data Table (upsert) — stores the latest snapshot per `rateKey`.
- **Table:** `Hotel_Rates`
- **Data written:** `hotelId`, `hotelName`, `checkInDate`, `timeSlot`, `bestTotal`, `currency`, `rateKey`.
- **Matching filter:** `rateKey == {{$json.rateKey}}` (so one row per date per hotel).
- **Outputs:** none (terminal for that branch path).
- **Edge cases:** Schema mismatch or type conversion disabled; ensure `bestTotal` is numeric or null.

#### Node: Insert History Rates
- **Type / role:** Data Table (insert) — append-only storage for time series.
- **Table:** `Hotel_Rates_History`
- **Data written:** includes `historyKey`, timestamps, and snapshot details.
- **Outputs:** none (terminal).
- **Edge cases:** If `historyKey` repeats (same slot rerun), table may allow duplicates unless constrained externally.

---

### Block 2.4 — Change Detection and Our-Hotel Comparison
**Overview:** Identifies significant competitor changes (vs previous snapshot) and loads your hotel’s comparable rate for the same check-in date.  
**Nodes involved:** `Compute Change`, `Significant Competitor Change`, `Get Our Hotel Rates`, `Prefix Our Hotel Fields`, `Combine Competitor Rates vs Our Rates`

#### Node: Compute Change
- **Type / role:** Code (per item) — compute delta metrics and significance.
- **Key calculations:**
  - `absChange = bestTotal - prevBestTotal`
  - `pctChange = absChange / prevBestTotal` and `pctChangePct = pctChange*100`
  - `isSignificant`: competitor only, and `|pctChange| >= thresholdPct`
  - role flags: `isCompetitor`, `isOurHotel`
- **Outputs:** To `Significant Competitor Change`.
- **Edge cases:**
  - No pctChange if `prevBestTotal` null/0 or if current `bestTotal` null.
  - Significance requires `prevExists` and non-null pctChange.

#### Node: Significant Competitor Change
- **Type / role:** IF — gates alerting pipeline.
- **Conditions (AND):**
  - `hotelRole == competitor`
  - `isSignificant == true`
  - `prevExists == true`
  - `outcome == OK`
- **True-path outputs:**
  - To `Get Our Hotel Rates` (load our hotel baseline)
  - To `Combine Competitor Rates vs Our Rates` input 0 (competitor item)
- **Edge cases:** If Amadeus returns `NO_OFFERS` or errors, competitor changes won’t alert.

#### Node: Get Our Hotel Rates
- **Type / role:** Data Table (get) — fetch your hotel’s latest rate snapshot for the same check-in date.
- **Table:** `Hotel_Rates`
- **Filter:** `rateKey == (ourHotel.hotelId + '|' + competitor.checkInDate)`
- **Outputs:** To `Prefix Our Hotel Fields`.
- **Edge cases:** If your hotel has no offer/snapshot stored for that date/time, comparison fields will be missing downstream.

#### Node: Prefix Our Hotel Fields
- **Type / role:** Set — prefixes fields from the retrieved “our hotel” row to avoid collisions.
- **Writes:** `ourHotelID`, `ourHotelName`, `ourHotelRateKey`, `ourHotelCheckInDate`, `ourHotelTimeSlot`, `ourHotelBestTotal`, `ourHotelCurrency`, `ourHotelUpdatedAt`
- **Outputs:** To `Combine Competitor Rates vs Our Rates` input 1.
- **Edge cases:** If `Get Our Hotel Rates` returned no row, these fields may be null/undefined.

#### Node: Combine Competitor Rates vs Our Rates
- **Type / role:** Merge (Combine by Position) — attaches “our hotel” prefixed fields to each competitor alert.
- **Inputs:**
  - Input 0: competitor alert items (from IF true path)
  - Input 1: prefixed our-hotel item(s)
- **Outputs:**
  - To `Build VCC URLs` (event scraping requests)
  - To `Combine Rates & Events` input 0 (alerts stream)
- **Edge cases:** Combine-by-position again requires aligned ordering; if multiple alerts and DB lookups yield mismatched counts, mapping can drift.

---

### Block 2.5 — Market Context (VCC Event Scraping & Normalization)
**Overview:** Scrapes Vancouver Convention Centre monthly event pages for months relevant to alert dates, extracts event metadata, and normalizes/sorts it into a compact summary used by the AI agent.  
**Nodes involved:** `Build VCC URLs`, `Fetch VCC Month HTML`, `Extract Event Details`, `Events Extract`

#### Node: Build VCC URLs
- **Type / role:** Code (run once for all items) — creates unique month URLs.
- **Key logic:**
  - For each alert’s `checkInDate`, adds month and next month to a set.
  - Produces one item per unique month:
    - `vccMonth` (`YYYY-MM`)
    - `vccUrl` like `https://www.vancouverconventioncentre.com/events/YYYY/MM`
    - `alertItems` array preserving all alert items for context (though later the pipeline uses the events bundle separately).
- **Outputs:** To `Fetch VCC Month HTML`.
- **Edge cases:** If alerts span many months, number of scrapes increases.

#### Node: Fetch VCC Month HTML
- **Type / role:** HTTP Request — fetches monthly event listing HTML.
- **Configuration:** Response format `text`, stored in property `events`.
- **onError:** continue regular output (pipeline continues even if scrape fails).
- **Outputs:** To `Extract Event Details`.
- **Edge cases:** Site structure changes, bot protection, 403s, timeouts.

#### Node: Extract Event Details
- **Type / role:** HTML Extract — parse monthly page into arrays.
- **Selectors extracted (arrays):**
  - `title`: `a.event-item .event-details h2`
  - `href`: attribute `href` on `a.event-item`
  - date parts: start/end day/month using `.event-date` selectors
- **Outputs:** To `Events Extract`.
- **Edge cases:** Selector changes will yield empty arrays, leading to empty event summary.

#### Node: Events Extract
- **Type / role:** Code (run once for all items) — normalize, infer years, dedupe, sort, summarize.
- **Key logic:**
  - Converts relative URLs to absolute.
  - Attempts to detect event year from title or URL slug.
  - Establishes an `anchorYear` (prefers Jan event years to solve Dec/Jan crossover).
  - Assigns year for events missing explicit year (Dec/Nov heuristic).
  - Produces `eventsSummary` as bullet list with dates + links.
- **Outputs:** To `Combine Rates & Events` input 1.
- **Edge cases:** If neither title nor URL includes year and the heuristic is wrong, `startISO` may be mis-assigned.

---

### Block 2.6 — Bundle Preparation for AI Analysis
**Overview:** Merges alert rows with the events bundle, then produces a compact JSON payload for the analysis agent.  
**Nodes involved:** `Combine Rates & Events`, `Prepare AI Agent Input`

#### Node: Combine Rates & Events
- **Type / role:** Merge — combines competitor alerts stream with events bundle stream.
- **Configuration:** default merge parameters (acts like an “append/merge” depending on n8n defaults).
- **Inputs:**
  - Input 0: enriched alert items (competitor + our hotel prefixed)
  - Input 1: events bundle from `Events Extract`
- **Outputs:** To `Prepare AI Agent Input`.
- **Edge cases:** If merge mode is not “append” in your n8n version, confirm behavior; the next node expects a mixed set of items (alerts + one events bundle).

#### Node: Prepare AI Agent Input
- **Type / role:** Code (run once for all items) — constructs the agent input bundle.
- **Key outputs:**
  - `alertsCount`, `alerts` (slimmed fields)
  - `vccEventsSummary` (only summary to avoid prompt bloat)
  - `relevantEventsByAlert`: computed overlaps where `checkInDate` falls within event start/end window
  - `context`: convenience info (first alert’s date/slot/ourHotelName)
- **Outputs:**
  - To `AI Agent: Analyze`
  - Also to `Combine Summary & Alert` input 1 (for persistence)
- **Edge cases:** If events bundle missing, overlap list remains empty; agent still works with alerts alone.

---

### Block 2.7 — AI Analysis, WhatsApp Alerting, and Persistence
**Overview:** Uses an OpenAI-powered agent to generate structured revenue recommendations, sends executive summary via WhatsApp, and stores the latest summary for later Q&A.  
**Nodes involved:** `AI Agent: Analyze`, `OpenAI Chat Model`, `Structured Output Parser`, `Send WhatsApp Alert`, `Combine Summary & Alert`, `Upsert Hotel Price Alerts`

#### Node: OpenAI Chat Model
- **Type / role:** LangChain Chat Model (OpenAI).
- **Model:** `gpt-4.1-mini`
- **Outputs:** Wired to `AI Agent: Analyze` as `ai_languageModel`.
- **Edge cases:** API key missing, model unavailable in region/account, token limits if prompt grows.

#### Node: Structured Output Parser
- **Type / role:** LangChain structured output parser — validates agent output against a JSON Schema.
- **Schema:** `RevenueManagerAlertAnalysis` with:
  - `executiveSummary` (string)
  - `alertsReviewed` (int)
  - `findings[]` with competitor and our rate fields + recommendation object
  - `vccEventsUsedSummary` (string)
- **Outputs:** Wired to `AI Agent: Analyze` as `ai_outputParser`.
- **Edge cases:** If agent returns non-JSON or violates schema, parsing fails (agent has `hasOutputParser: true`, so it should be constrained).

#### Node: AI Agent: Analyze
- **Type / role:** LangChain Agent — produces the final structured analysis.
- **Inputs:** Bundle from `Prepare AI Agent Input` (stringified into the prompt).
- **System message:** Revenue manager persona + explicit analysis steps and required output JSON.
- **Output parser:** Uses `Structured Output Parser` to enforce schema.
- **Outputs:**
  - To `Send WhatsApp Alert` (executive summary)
  - To `Combine Summary & Alert` input 0 (store full results)
- **Edge cases:** If no significant alerts exist, this path may never run (because earlier IF gate blocks it).

#### Node: Send WhatsApp Alert
- **Type / role:** WhatsApp (send message).
- **Message:** `{{$json.output.executiveSummary}}`
- **Config:** `phoneNumberId` is set; `recipientPhoneNumber` is a placeholder (`x`).
- **Edge cases:** Wrong credentials, invalid phoneNumberId, recipient not permitted, template/session requirements.

#### Node: Combine Summary & Alert
- **Type / role:** Merge (combine by position) — pairs the analysis output with the original prepared bundle.
- **Inputs:**
  - Input 0: AI agent output
  - Input 1: prepared bundle
- **Outputs:** To `Upsert Hotel Price Alerts`.
- **Edge cases:** Combine-by-position depends on 1:1 alignment (usually one bundle per run). If multiple AI outputs exist unexpectedly, pairing may break.

#### Node: Upsert Hotel Price Alerts
- **Type / role:** Data Table (upsert) — stores the “latest” analysis context for Q&A.
- **Table:** `Hotel_Price_Alert`
- **Upsert key:** `key == "latest"`
- **Fields written:**
  - `alertSummary` = executive summary
  - `eventSummary` = `vccEventsSummary`
  - `alertStrategy` = concatenation of tactics + monitoring from first finding  
    (expression uses `findings[0]`; assumes at least one finding exists)
- **Edge cases:**
  - If `findings` is empty, `findings[0]` will be undefined and expression may fail or write blank.
  - Consider guarding this with conditional expressions.

---

### Block 2.8 — WhatsApp Q&A Entry, Context Load, Agent, and Reply
**Overview:** Receives WhatsApp messages, loads latest stored alert context, then answers using an OpenAI agent with memory and a tool to query the historical rate table.  
**Nodes involved:** `WhatsApp Trigger`, `Filter Text Messages`, `Normalize Whatsapp Input`, `Get Alert Summary`, `AI Agent: Q&A`, `OpenAI Chat Model2`, `Simple Memory`, `Get Hotel_Rates_History`, `Q&A`

#### Node: WhatsApp Trigger
- **Type / role:** WhatsApp Trigger — entry point for Q&A branch.
- **Updates:** listens to `messages`.
- **Credentials:** placeholder “Empty API: For Template”.
- **Outputs:** To `Filter Text Messages`.
- **Edge cases:** Webhook setup, verification, and Meta/WhatsApp Cloud API permissions.

#### Node: Filter Text Messages
- **Type / role:** IF — allows only a specific sender.
- **Condition:** `$json.messages[0].from == "16727551224"`
- **Outputs:** True path to `Normalize Whatsapp Input`.
- **Edge cases:** Blocks all other senders; for production, replace with allowlist logic or remove.

#### Node: Normalize Whatsapp Input
- **Type / role:** Set — normalizes inbound WhatsApp payload shapes.
- **Output field:** `chatInput` extracted from multiple possible paths:
  - `$json.messages[0].text.body`
  - `$json.value.messages[0].text.body`
  - deeply nested webhook structure under `entry[0].changes[0].value.messages[0].text.body`
- **Outputs:** To `Get Alert Summary`.
- **Edge cases:** Non-text messages yield `""` which may confuse the agent; consider filtering for message type.

#### Node: Get Alert Summary
- **Type / role:** Data Table (get) — loads the latest stored analysis context.
- **Table:** `Hotel_Price_Alert`
- **Filter:** `key == "latest"`
- **Outputs:** To `AI Agent: Q&A`.
- **Edge cases:** If no “latest” row exists yet, agent system message will have missing variables.

#### Node: OpenAI Chat Model2
- **Type / role:** LangChain Chat Model (OpenAI)
- **Model:** `gpt-4.1-mini`
- **Outputs:** To `AI Agent: Q&A` as `ai_languageModel`.

#### Node: Simple Memory
- **Type / role:** LangChain memory buffer window — keeps conversational context per WhatsApp sender.
- **Session key:** `$('WhatsApp Trigger').item.json.messages[0].from`
- **Outputs:** Connected to `AI Agent: Q&A` as `ai_memory`.
- **Edge cases:** If trigger payload differs, sessionKey expression may fail; could reuse `Normalize Whatsapp Input` context.

#### Node: Get Hotel_Rates_History
- **Type / role:** DataTable Tool — exposes `Hotel_Rates_History` to the agent as a callable tool.
- **Operation:** `get`
- **Outputs:** Connected to `AI Agent: Q&A` as `ai_tool`.
- **Edge cases:** Tool returns entire rows; agent may need prompting to filter by hotelId/date to avoid token bloat.

#### Node: AI Agent: Q&A
- **Type / role:** LangChain Agent — answers user questions using latest alert context + optional tool access.
- **User text:** from `Normalize Whatsapp Input` → `chatInput`
- **System message:** embeds:
  - `alertSummary`, `eventSummary`, `alertStrategy` from `Hotel_Price_Alert`
  - instructs it can query `Hotel_Rates_History` for other dates/trends
- **Outputs:** To `Q&A`.
- **Edge cases:** If `chatInput` is empty, output may be generic; you may want a guard clause.

#### Node: Q&A
- **Type / role:** WhatsApp (send message) — sends the agent’s answer.
- **Message body:** `{{$json.output}}`
- **phoneNumberId / recipient:** placeholders (`1234`, `x`) in this template.
- **Edge cases:** Same WhatsApp constraints as alert sending.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Market Check | scheduleTrigger | Intraday scheduler (09/15/21) | — | Config | ## Data Collection Engine<br>This section triggers on a schedule (09:00, 15:00, 21:00) to ensure intraday coverage. It generates 30 individual date windows and queries the Amadeus API to fetch live pricing for competitor hotels. |
| Config | set | Monitoring parameters (hotels, thresholds) | Daily Market Check | Amadeus OAuth | ## Data Collection Engine<br>This section triggers on a schedule (09:00, 15:00, 21:00) to ensure intraday coverage. It generates 30 individual date windows and queries the Amadeus API to fetch live pricing for competitor hotels. |
| Amadeus OAuth | httpRequest | Get Amadeus access token | Config | Generate 30 night windows | ## Data Collection Engine<br>This section triggers on a schedule (09:00, 15:00, 21:00) to ensure intraday coverage. It generates 30 individual date windows and queries the Amadeus API to fetch live pricing for competitor hotels. |
| Generate 30 night windows | code | Expand hotels × dates, attach token/context | Amadeus OAuth | Amadeus Hotel Offers; Combine 30-Day Rates | ## Data Collection Engine<brThis section triggers on a schedule (09:00, 15:00, 21:00) to ensure intraday coverage. It generates 30 individual date windows and queries the Amadeus API to fetch live pricing for competitor hotels. |
| Amadeus Hotel Offers | httpRequest | Fetch offers per hotel/date | Generate 30 night windows | Combine 30-Day Rates | ## Data Collection Engine<brThis section triggers on a schedule (09:00, 15:00, 21:00) to ensure intraday coverage. It generates 30 individual date windows and queries the Amadeus API to fetch live pricing for competitor hotels. |
| Combine 30-Day Rates | merge | Reattach request context to API response | Generate 30 night windows; Amadeus Hotel Offers | Clean Amadeus Data | ## Data Collection Engine<brThis section triggers on a schedule (09:00, 15:00, 21:00) to ensure intraday coverage. It generates 30 individual date windows and queries the Amadeus API to fetch live pricing for competitor hotels. |
| Clean Amadeus Data | code | Normalize Amadeus response into minRate/outcome | Combine 30-Day Rates | New Snapshot | ## Data Consolidation & Archiving<brConsolidates the results with the previous pricing snapshot into a single dataset for further processing and archives into the data bases. |
| New Snapshot | code | Compute rateKey/historyKey/timeSlot/bestTotal | Clean Amadeus Data | Get Prev Rates; Combine New Rates vs Prev Rates | ## Data Consolidation & Archiving<brConsolidates the results with the previous pricing snapshot into a single dataset for further processing and archives into the data bases. |
| Get Prev Rates | dataTable | Load previous latest snapshot by rateKey | New Snapshot | Prev Snapshot | ## Data Consolidation & Archiving<brConsolidates the results with the previous pricing snapshot into a single dataset for further processing and archives into the data bases. |
| Prev Snapshot | code | Standardize previous snapshot fields | Get Prev Rates | Combine New Rates vs Prev Rates | ## Data Consolidation & Archiving<brConsolidates the results with the previous pricing snapshot into a single dataset for further processing and archives into the data bases. |
| Combine New Rates vs Prev Rates | merge | Enrich current snapshot with previous snapshot | New Snapshot; Prev Snapshot | Upsert Latest Rates; Insert History Rates; Compute Change | ## Data Consolidation & Archiving<brConsolidates the results with the previous pricing snapshot into a single dataset for further processing and archives into the data bases. |
| Upsert Latest Rates | dataTable | Persist latest snapshot (1 row per rateKey) | Combine New Rates vs Prev Rates | — | ## Data Consolidation & Archiving<brConsolidates the results with the previous pricing snapshot into a single dataset for further processing and archives into the data bases. |
| Insert History Rates | dataTable | Append time series history snapshot | Combine New Rates vs Prev Rates | — | ## Data Consolidation & Archiving<brConsolidates the results with the previous pricing snapshot into a single dataset for further processing and archives into the data bases. |
| Compute Change | code | Compute abs/pct change + significance flags | Combine New Rates vs Prev Rates | Significant Competitor Change | ## Change Detection & Comparison<brFilters for competitor's significant price changes (>10%) and fetch our hotel's prices for further analysis. |
| Significant Competitor Change | if | Gate: competitor + significant + prevExists + OK | Compute Change | Get Our Hotel Rates; Combine Competitor Rates vs Our Rates | ## Change Detection & Comparison<brFilters for competitor's significant price changes (>10%) and fetch our hotel's prices for further analysis. |
| Get Our Hotel Rates | dataTable | Load our hotel snapshot for same date | Significant Competitor Change | Prefix Our Hotel Fields | ## Change Detection & Comparison<brFilters for competitor's significant price changes (>10%) and fetch our hotel's prices for further analysis. |
| Prefix Our Hotel Fields | set | Prefix our-hotel fields to avoid collisions | Get Our Hotel Rates | Combine Competitor Rates vs Our Rates | ## Change Detection & Comparison<brFilters for competitor's significant price changes (>10%) and fetch our hotel's prices for further analysis. |
| Combine Competitor Rates vs Our Rates | merge | Attach our-hotel comparison fields to competitor alerts | Significant Competitor Change; Prefix Our Hotel Fields | Build VCC URLs; Combine Rates & Events | ## Change Detection & Comparison<brFilters for competitor's significant price changes (>10%) and fetch our hotel's prices for further analysis. |
| Build VCC URLs | code | Create unique monthly VCC URLs from alert dates | Combine Competitor Rates vs Our Rates | Fetch VCC Month HTML | ## Market Context<brScrapes the Vancouver Convention Centre (VCC) website for the specific alert dates to identify if a conference or event is driving the competitor's price surge. |
| Fetch VCC Month HTML | httpRequest | Fetch VCC month listing HTML | Build VCC URLs | Extract Event Details | ## Market Context<brScrapes the Vancouver Convention Centre (VCC) website for the specific alert dates to identify if a conference or event is driving the competitor's price surge. |
| Extract Event Details | html | Extract event titles/links/date parts | Fetch VCC Month HTML | Events Extract | ## Market Context<brScrapes the Vancouver Convention Centre (VCC) website for the specific alert dates to identify if a conference or event is driving the competitor's price surge. |
| Events Extract | code | Normalize/dedupe/sort events + build summary | Extract Event Details | Combine Rates & Events | ## Market Context<brScrapes the Vancouver Convention Centre (VCC) website for the specific alert dates to identify if a conference or event is driving the competitor's price surge. |
| Combine Rates & Events | merge | Merge alert items + events bundle for bundling | Combine Competitor Rates vs Our Rates; Events Extract | Prepare AI Agent Input | ## Analysis by Revenue AI Agent<brConducts analysis to generate a strategic "Executive Summary" recommending specific revenue actions. Delivers the result and archives into the data base. |
| Prepare AI Agent Input | code | Build compact JSON payload (alerts + events summary + overlaps) | Combine Rates & Events | AI Agent: Analyze; Combine Summary & Alert | ## Analysis by Revenue AI Agent<brConducts analysis to generate a strategic "Executive Summary" recommending specific revenue actions. Delivers the result and archives into the data base. |
| OpenAI Chat Model | lmChatOpenAi | LLM backend for analysis agent | — | AI Agent: Analyze | ## Analysis by Revenue AI Agent<brConducts analysis to generate a strategic "Executive Summary" recommending specific revenue actions. Delivers the result and archives into the data base. |
| Structured Output Parser | outputParserStructured | Enforce agent JSON schema | — | AI Agent: Analyze | ## Analysis by Revenue AI Agent<brConducts analysis to generate a strategic "Executive Summary" recommending specific revenue actions. Delivers the result and archives into the data base. |
| AI Agent: Analyze | agent | Produce structured revenue recommendations | Prepare AI Agent Input (+ model/parser) | Send WhatsApp Alert; Combine Summary & Alert | ## Analysis by Revenue AI Agent<brConducts analysis to generate a strategic "Executive Summary" recommending specific revenue actions. Delivers the result and archives into the data base. |
| Send WhatsApp Alert | whatsApp | Send executive summary alert | AI Agent: Analyze | — | ## Analysis by Revenue AI Agent<brConducts analysis to generate a strategic "Executive Summary" recommending specific revenue actions. Delivers the result and archives into the data base. |
| Combine Summary & Alert | merge | Combine analysis output + prepared bundle | AI Agent: Analyze; Prepare AI Agent Input | Upsert Hotel Price Alerts | ## Analysis by Revenue AI Agent<brConducts analysis to generate a strategic "Executive Summary" recommending specific revenue actions. Delivers the result and archives into the data base. |
| Upsert Hotel Price Alerts | dataTable | Store latest alert context for Q&A | Combine Summary & Alert | — | ## Analysis by Revenue AI Agent<brConducts analysis to generate a strategic "Executive Summary" recommending specific revenue actions. Delivers the result and archives into the data base. |
| WhatsApp Trigger | whatsAppTrigger | Incoming WhatsApp message entry point | — | Filter Text Messages | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| Filter Text Messages | if | Allowlist sender filter | WhatsApp Trigger | Normalize Whatsapp Input | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| Normalize Whatsapp Input | set | Extract user text from multiple payload shapes | Filter Text Messages | Get Alert Summary | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| Get Alert Summary | dataTable | Load latest saved analysis context | Normalize Whatsapp Input | AI Agent: Q&A | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| OpenAI Chat Model2 | lmChatOpenAi | LLM backend for Q&A agent | — | AI Agent: Q&A | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| Simple Memory | memoryBufferWindow | Per-sender conversational memory | — | AI Agent: Q&A (ai_memory) | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| Get Hotel_Rates_History | dataTableTool | Tool access to rate history for agent | — | AI Agent: Q&A (ai_tool) | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| AI Agent: Q&A | agent | Answer questions using latest context + tool | Get Alert Summary (+ model/memory/tool) | Q&A | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| Q&A | whatsApp | Send agent answer back to user | AI Agent: Q&A | — | ## Follow-Up Q&A with Revenue AI Agent<brRetrieves the latest alert context and uses database tools to answer our follow-up questions with historical trend data. |
| Sticky Note | stickyNote | Documentation | — | — |  |
| Sticky Note2 | stickyNote | Documentation | — | — |  |
| Sticky Note3 | stickyNote | Documentation | — | — |  |
| Sticky Note6 | stickyNote | Documentation | — | — |  |
| Sticky Note7 | stickyNote | Documentation | — | — |  |
| Sticky Note8 | stickyNote | Documentation | — | — |  |
| Sticky Note9 | stickyNote | Documentation | — | — |  |
| Sticky Note10 | stickyNote | Documentation | — | — |  |
| Sticky Note11 | stickyNote | Documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

### A) Create required Data Tables
1. **Create Data Table: `Hotel_Rates`** (latest snapshot)
   - Columns (at minimum): `rateKey` (string), `hotelId` (string), `hotelName` (string), `checkInDate` (string), `timeSlot` (string), `bestTotal` (number), `currency` (string).
   - Ensure `rateKey` is used as the logical unique key (workflow enforces it via upsert filter).

2. **Create Data Table: `Hotel_Rates_History`** (append-only)
   - Columns: `historyKey` (string), `hotelID` (string), `hotelName` (string), `checkInDate` (string), `bestTotal` (number), `currency` (string), `timeSlot` (string), `observedAtUtc` (string), `observedAtLocal` (string).

3. **Create Data Table: `Hotel_Price_Alert`** (latest analysis context)
   - Columns: `key` (string), `alertSummary` (string), `eventSummary` (string), `alertStrategy` (string).
   - Store latest under `key="latest"`.

### B) Monitoring branch (scheduled)
4. Add **Schedule Trigger** node named **Daily Market Check**
   - Add three cron rules: `0 0 9 * * *`, `0 0 15 * * *`, `0 0 21 * * *`.

5. Add **Set** node named **Config**
   - Add fields: `currency`, `adults`, `roomQuantity`, `thresholdPct`, `daysAhead`, `competitors` (array), `ourHotel` (object).
   - Populate competitor + your hotel IDs.

6. Add **HTTP Request** node named **Amadeus OAuth**
   - Method: POST
   - URL: `https://test.api.amadeus.com/v1/security/oauth2/token`
   - Body: form-urlencoded
   - Params: `grant_type=client_credentials`, plus `client_id`, `client_secret` (bind via credentials or environment variables).

7. Add **Code** node named **Generate 30 night windows**
   - Paste logic to:
     - Read config from `Config`
     - Read token from `Amadeus OAuth`
     - Emit items for each hotel×date with `access_token`, `hotelRole`, `hotelId`, dates, and config fields.

8. Add **HTTP Request** node named **Amadeus Hotel Offers**
   - GET `https://test.api.amadeus.com/v3/shopping/hotel-offers`
   - Query parameters from item: hotelIds, checkInDate, checkOutDate, adults, roomQuantity, currency, paymentPolicy=NONE
   - Header: `Authorization: Bearer {{$json.access_token}}`
   - Enable batching: size 1, interval ~1200ms
   - Set **On Error** → “Continue (regular output)”.

9. Add **Merge** node named **Combine 30-Day Rates**
   - Mode: Combine
   - Combine by: Position
   - Connect:
     - Generate 30 night windows → Merge input 0
     - Amadeus Hotel Offers → Merge input 1

10. Add **Code** node named **Clean Amadeus Data**
    - Normalize response to `minRate`, `outcome`, `httpStatus`, etc., while preserving request context.

11. Add **Code** node named **New Snapshot**
    - Compute: `timeSlot`, `bestTotal`, `rateKey`, `historyKey`, `observedAtUtc`, `observedAtLocal`.

12. Add **Data Table (Get)** node named **Get Prev Rates** (Hotel_Rates)
    - Filter: `rateKey == {{$json.rateKey}}`
    - Limit: 1
    - Enable “Always output data”.

13. Add **Code** node named **Prev Snapshot**
    - Add `prevExists` and normalized previous fields.

14. Add **Merge** node named **Combine New Rates vs Prev Rates**
    - Join mode: Enrich Input1
    - Match field: `rateKey`
    - Clash handling: prefer current (input1) values

15. Add **Data Table (Upsert)** node named **Upsert Latest Rates** (Hotel_Rates)
    - Filter: `rateKey == {{$json.rateKey}}`
    - Map fields from current item.

16. Add **Data Table (Insert)** node named **Insert History Rates** (Hotel_Rates_History)
    - Insert `historyKey`, timestamps, and snapshot fields.

17. Add **Code** node named **Compute Change**
    - Compute deltas and `isSignificant` for competitor items only.

18. Add **IF** node named **Significant Competitor Change**
    - Conditions:
      - hotelRole == competitor
      - isSignificant == true
      - prevExists == true
      - outcome == OK

### C) Enrichment branch (our hotel + events) and AI analysis
19. Add **Data Table (Get)** node named **Get Our Hotel Rates** (Hotel_Rates)
    - Filter: `rateKey == {{ourHotelId}}|{{$json.checkInDate}}` (pull `ourHotelId` from Config).

20. Add **Set** node named **Prefix Our Hotel Fields**
    - Create prefixed fields (ourHotelBestTotal etc.) from the fetched row.

21. Add **Merge** node named **Combine Competitor Rates vs Our Rates**
    - Mode: Combine by Position
    - Input 0: competitor alert item(s)
    - Input 1: prefixed our hotel row(s)

22. Add **Code** node named **Build VCC URLs**
    - Run once for all items
    - Create unique month URLs: `https://www.vancouverconventioncentre.com/events/YYYY/MM`

23. Add **HTTP Request** node named **Fetch VCC Month HTML**
    - URL: `{{$json.vccUrl}}`
    - Response: Text, output property name `events`
    - On Error: Continue regular output

24. Add **HTML Extract** node named **Extract Event Details**
    - Extract arrays: title, href, startDay/startMonth, endDay/endMonth using CSS selectors.

25. Add **Code** node named **Events Extract**
    - Normalize, infer year, dedupe, sort; output `eventsSummary`.

26. Add **Merge** node named **Combine Rates & Events**
    - Ensure it outputs a mixed set: alert items + single events bundle item.

27. Add **Code** node named **Prepare AI Agent Input**
    - Create compact JSON bundle with `alerts[]`, `vccEventsSummary`, and overlaps.

28. Add **OpenAI Chat Model** node named **OpenAI Chat Model**
    - Model: `gpt-4.1-mini`
    - Configure OpenAI credentials.

29. Add **Structured Output Parser** node
    - Paste the JSON schema used to validate the agent output.

30. Add **AI Agent** node named **AI Agent: Analyze**
    - Prompt: stringify the input bundle and ask for “ONLY valid JSON”
    - System message: revenue manager instructions and required output schema
    - Attach:
      - Chat model as `ai_languageModel`
      - Structured parser as `ai_outputParser`

31. Add **WhatsApp (Send)** node named **Send WhatsApp Alert**
    - Text: `{{$json.output.executiveSummary}}`
    - Configure WhatsApp Cloud API credentials, `phoneNumberId`, recipient number.

32. Add **Merge** node named **Combine Summary & Alert**
    - Combine by position: AI output + prepared bundle.

33. Add **Data Table (Upsert)** node named **Upsert Hotel Price Alerts** (Hotel_Price_Alert)
    - Filter: `key == "latest"`
    - Write: `alertSummary`, `eventSummary`, `alertStrategy` and set `key="latest"`.

### D) WhatsApp Q&A branch
34. Add **WhatsApp Trigger** node named **WhatsApp Trigger**
    - Subscribe to `messages`
    - Configure webhook/credentials.

35. Add **IF** node named **Filter Text Messages**
    - Optional: restrict by sender (`messages[0].from`).
    - For production, replace with allowlist logic or remove.

36. Add **Set** node named **Normalize Whatsapp Input**
    - Build `chatInput` from the possible inbound payload shapes.

37. Add **Data Table (Get)** node named **Get Alert Summary** (Hotel_Price_Alert)
    - Filter: `key == "latest"`

38. Add **OpenAI Chat Model** node named **OpenAI Chat Model2**
    - Model: `gpt-4.1-mini`

39. Add **Simple Memory** node (buffer window)
    - Session key: sender phone (`messages[0].from`)

40. Add **Data Table Tool** node named **Get Hotel_Rates_History**
    - Expose `Hotel_Rates_History` as a tool (operation: get)

41. Add **AI Agent** node named **AI Agent: Q&A**
    - User input: `{{$('Normalize Whatsapp Input').item.json.chatInput}}`
    - System message includes the loaded `alertSummary`, `eventSummary`, `alertStrategy`
    - Attach:
      - `OpenAI Chat Model2` as language model
      - `Simple Memory` as memory
      - `Get Hotel_Rates_History` as tool

42. Add **WhatsApp (Send)** node named **Q&A**
    - Text: `{{$json.output}}`
    - Configure WhatsApp credentials, `phoneNumberId`, recipient (typically from inbound payload).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **How It Works / Setup Steps** (top branch monitoring, context enrichment, AI analysis & alerting; bottom branch interactive analyst) | Sticky note “How It Works” (embedded in workflow) |
| **Prerequisites:** OTA API key, LLM key, database, message tool key | Sticky note “Prerequisites” |
| **Use cases & benefits:** Revenue managers, sales/marketing, leadership; “Zero-Touch Efficiency”, “Contextual Intelligence”, “Actionable Strategy”, “Long-Term Vision” | Sticky note “Use Cases & Benefits” |
| **Market context source:** Vancouver Convention Centre events pages | `https://www.vancouverconventioncentre.com/events/YYYY/MM` (scraped by workflow) |

