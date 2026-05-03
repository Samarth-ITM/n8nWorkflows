Generate social content pillars, calendars and posts using Google Sheets and OpenAI

https://n8nworkflows.xyz/workflows/generate-social-content-pillars--calendars-and-posts-using-google-sheets-and-openai-12294


# Generate social content pillars, calendars and posts using Google Sheets and OpenAI

## 1. Workflow Overview

**Purpose:** This n8n workflow automates social media planning end-to-end: it reads a brand’s planning inputs from Google Sheets, uses OpenAI (GPT-4o) to generate **content pillars**, then creates a **full posting calendar**, then generates **publish-ready posts** (hooks, main copy, CTA, hashtags) and stores everything back into structured Google Sheets tabs.

**Target use cases:**
- B2B/B2C teams producing repeatable LinkedIn/Instagram content plans
- Agencies generating deliverables (pillars + calendar + copy) from a client intake sheet
- Automated content ops pipelines where Sheets is the source of truth

### 1.1 Input Reception & Planning Data Fetch
Triggered by Google Sheets polling; fetches the relevant input row(s) containing brand and scheduling constraints.

### 1.2 Pillar Calculation & Pillar Storage
Uses OpenAI to compute posting metrics and generate platform-specific pillars (with strict weight rules), parses/validates JSON, and appends pillar rows into a “Pillars” sheet/tab.

### 1.3 Calendar Generation & Calendar Storage
Uses OpenAI to build a day-by-day calendar from pillars + constraints, parses it into sheet-row format, and appends calendar rows into a “Calendar” sheet/tab.

### 1.4 Post Creation Loop (format routing + throttling)
Routes calendar items by format (Video vs Non-Video), loops through items one-by-one to avoid rate limits, generates post copy with OpenAI, parses into final row format, appends results into a “Final Posts” sheet/tab, then waits and continues.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Capture & Trigger
**Overview:** Detects new/updated planning rows and retrieves sheet row data that becomes the single source of truth for all downstream prompts and calculations.

**Nodes involved:**
- Google Sheets Trigger
- Get row(s) in sheet

#### Node: Google Sheets Trigger
- **Type / Role:** `n8n-nodes-base.googleSheetsTrigger` — polling trigger that fires on schedule.
- **Configuration (interpreted):**
  - Polling mode: **everyHour**
  - Document: `your_google_sheet_id_here`
  - Sheet (tab): `gid=0` (first tab)
  - Output: `includeInOutput: both` (returns both old/new row snapshots when applicable)
- **Connections:**
  - Output → **Get row(s) in sheet**
- **Edge cases / failures:**
  - Google OAuth credential expiry / missing permissions
  - Polling may miss very rapid changes between intervals
  - If the trigger delivers multiple changed rows, downstream nodes must handle multiple items (this workflow largely assumes a single planning context per execution)

#### Node: Get row(s) in sheet
- **Type / Role:** `n8n-nodes-base.googleSheets` (v4.7) — reads rows from the planning sheet.
- **Configuration (interpreted):**
  - Same document/tab as trigger (`gid=0`)
  - Operation implied by node name: “Get row(s)” (read)
- **Key data consumed downstream (fields referenced in prompts):**
  - `Brand Name`, `Industry`, `Platform`, `Target Audience`, `Content Goal`, `Tone`
  - `Start Date`, `Duration (Days)`, `Posting Frequency / Week`
  - `Promotion Level`, `Max Promo Posts / Week`, `Min Educational Posts / Week`, `CTA Cooldown (Days)`, `Max Same Pillar Streak`
- **Connections:**
  - Output → **Message a model** (pillar generation)
- **Edge cases / failures:**
  - Empty sheet / no matching rows returned
  - Column names must match exactly (including spaces and punctuation) or expressions will resolve to `undefined`
  - Date fields are passed as strings; inconsistent formats can break the model’s deterministic date logic

**Sticky note covering this block:**
- **“Step 1: Input Capture & Trigger”** (explains the intent of trigger + capture)

---

### Block 2 — Pillar Generation, Parsing, Validation, and Storage
**Overview:** Calls GPT-4o to compute calendar metrics and generate 3–5 content pillars per resolved platform, then parses JSON, validates pillar structure and weight totals per platform, and stores pillars in a dedicated sheet.

**Nodes involved:**
- Message a model
- Code in JavaScript
- Append row in sheet

#### Node: Message a model
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` (v2.1) — OpenAI chat call producing pillar JSON + calculation metadata.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
  - System message enforces:
    - Output **only valid JSON**
    - Weight totals **must equal 100 per platform**
    - Platform resolution strictly: LinkedIn / Instagram / Both → array
    - Educational must dominate unless promotion is High
  - User message includes deterministic rules:
    - `totalWeeks = ceil(Duration/7)`
    - `endDate = start + Duration`
    - `totalPostsPerPlatform = postsPerWeek * totalWeeks`
    - promotional ratio based on Promotion Level (Low/Medium/High)
    - min/max constraints for promo and educational posts per week
    - pillar distribution requirements and platform considerations
  - Uses expressions like `{{ $json['Brand Name'] }}` from the row item.
- **Connections:**
  - Output → **Code in JavaScript**
- **Edge cases / failures:**
  - Model returns non-JSON despite instructions (handled by downstream parser)
  - Model produces weights not summing to 100 (explicitly validated downstream)
  - Rate limits / API auth errors / timeouts from OpenAI

#### Node: Code in JavaScript
- **Type / Role:** `n8n-nodes-base.code` (v2) — parses the OpenAI response structure and converts pillars to Google Sheets row items.
- **Configuration choices / logic:**
  - Extracts response text from LangChain/OpenAI node structure:
    - `items[0].json.output[0].content[...]` where `type === 'output_text'`
  - Removes possible code fences (```json … ```), trims, then `JSON.parse`
  - Validates:
    - `contentPillars` exists and is a non-empty array
    - `platforms` exists and is an array
    - each pillar has `pillar_name`, `platform`, and `Weight (%)`
  - Maps each pillar into a row with columns:
    - `Pillar Name` ← `pillar.pillar_name`
    - `Primary Purpose` ← `pillar.description`
    - `Platform` ← `pillar.platform`
    - `Weight (%)` ← `pillar['Weight (%)']`
    - `Allowed Formats` ← `pillar.pillar_type` (note: this is a semantic mismatch—“Allowed Formats” is being filled with pillar type)
    - `CTA Allowed` ← `pillar['CTA Allowed']` defaulting to `Soft`
  - Validates weights: sums weights per platform (using `parseInt`) must equal **100**
- **Connections:**
  - Output → **Append row in sheet**
- **Edge cases / failures:**
  - If OpenAI node output format changes (e.g., content array not containing `output_text`), extraction fails
  - `parseInt` on weights: if weights include `%` or decimals as strings, validation may fail or truncate
  - “Allowed Formats” column is populated with `pillar_type`; if the destination sheet expects actual formats, downstream logic may become inconsistent

#### Node: Append row in sheet
- **Type / Role:** `n8n-nodes-base.googleSheets` (v4.7) — appends generated pillar rows to a dedicated tab.
- **Configuration (interpreted):**
  - Operation: **append**
  - Sheet/tab: `value: 467688910` (a specific gid/tab for pillars)
  - Column mapping (expressions):
    - `Platform` = `{{$json.Platform}}`, `Weight (%)` = `{{$json["Weight (%)"]}}`, etc.
- **Connections:**
  - Output → **Message a model7** (calendar generation)
- **Edge cases / failures:**
  - Schema mismatch: destination sheet must have headers exactly:
    - Pillar Name, Primary Purpose, Platform, Weight (%), Allowed Formats, CTA Allowed
  - Permissions / quota limits in Google Sheets API
  - Append order: if multiple pillars become multiple items, Sheets will receive multiple append operations (expected)

**Sticky note covering this block:**
- General workflow description note (the large “Smart Content Calendar Orchestration Workflow” note)
- **“Step 2: Calendar Generation & Routing”** partly overlaps conceptually (pillars+calendar)

---

### Block 3 — Calendar Generation, Parsing, and Storage
**Overview:** Uses GPT-4o to generate a complete calendar (post dates, pillar assignment, intent, hooks angles, CTA type, format), parses it into rows, and appends to the calendar sheet.

**Nodes involved:**
- Message a model7
- Code in JavaScript7
- Append row in sheet6
- Switch By Format

#### Node: Message a model7
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` (v2.1) — generates calendar JSON.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
  - System message enforces:
    - Output only JSON
    - Do not exceed `Max Promo Posts / Week`
    - Respect `CTA Cooldown (Days)` (days since last hard CTA)
    - Respect `Max Same Pillar Streak`
    - Keep within duration date range
    - `pillarID` must match exact pillar names provided
  - User message:
    - Pulls planning inputs explicitly via expressions referencing **Get row(s) in sheet**:
      - `Start Date: {{ $('Get row(s) in sheet').item.json['Start Date'] }}`
      - etc.
    - Provides pillars list via `{{ JSON.stringify($input.all()) }}` (this relies on the upstream items being the pillar rows coming from “Append row in sheet” output).
- **Connections:**
  - Output → **Code in JavaScript7**
- **Edge cases / failures:**
  - The prompt references `$('Get row(s) in sheet').item...` which assumes that node has an accessible item in this execution context; if multiple items or execution branches occur, you can get unexpected row references.
  - `JSON.stringify($input.all())` will stringify the current input items; after “Append row in sheet”, the incoming items may be the append operation results, not the original pillar objects (depends on n8n node output behavior). If it doesn’t contain pillar data, the model will not receive the actual pillars.

#### Node: Code in JavaScript7
- **Type / Role:** `n8n-nodes-base.code` (v2) — parses calendar JSON and maps to exact sheet columns.
- **Configuration / logic:**
  - Extracts `output_text` similarly to the pillar parser
  - Removes code fences, `JSON.parse`
  - Validates `contentCalendar` exists and is non-empty
  - Maps each entry to:
    - `Sr No` (uses `entry.srNo` or index+1)
    - `Date` `entry.date`
    - `Platform` `entry.platform`
    - `Pillar ID` `entry.pillarID`
    - `Format` `entry.format` default `Text Post`
    - `Post Intent` `entry.postIntent`
    - `Hook Angle` `entry.hookAngle`
    - `CTA Type` `entry.ctaType` default `None`
    - `Status` `entry.status` default `Pending`
  - Contains additional validations/logging:
    - Pillar distribution logging
    - CTA spacing validation uses a **hard-coded** `ctaCooldownDays = 5` and checks spacing by **index** not date
- **Connections:**
  - Output → **Append row in sheet6**
- **Edge cases / failures:**
  - Hard-coded cooldown (5) may contradict sheet input `CTA Cooldown (Days)`
  - CTA spacing check uses item index, not calendar date difference; if frequency isn’t daily, it’s not actually “days”
  - If model outputs keys in different casing (e.g., `srNo` vs `srno`), mapping may miss fields

#### Node: Append row in sheet6
- **Type / Role:** `n8n-nodes-base.googleSheets` (v4.7) — appends calendar rows to a calendar tab.
- **Configuration (interpreted):**
  - Operation: **append**
  - Sheet/tab: `value: 1387629362` (calendar gid)
  - Column mapping:
    - `Date`, `Sr No`, `Format`, `Status`, `CTA Type`, `Platform`, `Pillar ID`, `Hook Angle`, `Post Intent`
- **Connections:**
  - Output → **Switch By Format**
- **Edge cases / failures:**
  - Destination headers must exactly match those keys
  - Date format should be consistent (`YYYY-MM-DD`) to keep sorting/filters reliable

#### Node: Switch By Format
- **Type / Role:** `n8n-nodes-base.switch` (v3.4) — routes calendar entries by format.
- **Configuration (interpreted):**
  - Rule 1: if `{{$json.Format}} == "Video"` → first output
  - Rule 2: if `{{$json.Format}} != "Video"` → second output
- **Connections:**
  - Output 0 (Video): **not connected** (currently does nothing for video posts)
  - Output 1 (Non-Video): → **Loop Over Items**
- **Edge cases / failures:**
  - Only exact match `"Video"` is treated as video; formats like “Reel” or “Short Video” will go to non-video path
  - Video path is unimplemented; video items are effectively dropped

**Sticky note covering this block:**
- **“Step 2: Calendar Generation & Routing”** (pillars + calendar + routing)

---

### Block 4 — Post Creation Loop, Parsing, Final Storage, Throttling
**Overview:** Iterates through each non-video calendar item, generates final post copy with GPT-4o, parses it into a row including the original calendar date, appends to the final posts sheet, then waits and continues the loop.

**Nodes involved:**
- Loop Over Items
- Message a model6
- Code in JavaScript6
- Append row in sheet7
- Wait

#### Node: Loop Over Items
- **Type / Role:** `n8n-nodes-base.splitInBatches` (v3) — processes items in batches (effectively a loop).
- **Configuration (interpreted):**
  - No explicit batch size set in parameters (n8n default is typically 1 unless configured; behavior should be confirmed in UI)
- **Connections:**
  - Output 1 (current batch) → **Message a model6**
  - Output 0 (done) → not connected
  - Also receives input from **Wait** to continue looping
- **Edge cases / failures:**
  - If batch size is not 1, OpenAI calls may be made in bursts (rate limit risk)
  - If incoming items are empty, nothing is generated (silent completion)

#### Node: Message a model6
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` (v2.1) — generates final post content JSON per calendar row.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
  - System message enforces:
    - Output only JSON
    - Two hooks (8–15 words)
    - Main content must be complete and format-specific
    - CTA rules: None → empty string
    - Hashtags are space-separated string
  - User message uses the loop item fields:
    - `Sr No`, `Platform`, `Pillar ID`, `Post Intent`, `Format`, `Hook Angle`, `CTA Type`
- **Connections:**
  - Output → **Code in JavaScript6**
- **Edge cases / failures:**
  - If loop item lacks one of those fields, content quality drops and validation may fail downstream
  - Instagram guidance mentions emojis; system rules don’t forbid them, but your brand may
  - Rate limits; mitigated by Wait + batching strategy

#### Node: Code in JavaScript6
- **Type / Role:** `n8n-nodes-base.code` (v2) — parses post JSON and merges in the original calendar date.
- **Configuration / logic:**
  - Extracts text via `items[0].json.output?.[0]?.content?.[0]?.text`
    - Note: this differs from the other parsers (it assumes `[0]` is text directly, not searching for `type === 'output_text'`)
  - Removes code fences, parses JSON
  - Pulls original calendar data from: `$('Loop Over Items').item.json`
  - Outputs one row with:
    - `Sr No`, `Date` (from originalData), `Hook Option 1/2`, `Main Content`, `CTA`, `Hashtags`, `Platform Variant`
- **Connections:**
  - Output → **Append row in sheet7**
- **Edge cases / failures:**
  - If OpenAI node output structure changes, `...content?.[0]?.text` may be undefined (and it throws “No text found”)
  - If `Loop Over Items` context item is not the intended one (multi-item complexities), the `Date` could mismatch

#### Node: Append row in sheet7
- **Type / Role:** `n8n-nodes-base.googleSheets` (v4.7) — appends final post content to the final posts tab.
- **Configuration (interpreted):**
  - Operation: **append**
  - Sheet/tab: `value: 1093465678` (final posts gid)
  - Column mapping:
    - `CTA`, `Date`, `Sr No`, `Hashtags`, `Main Content`,
    - `Hook (Option 1)`, `Hook (Option 2)`,
    - `Platform Variant`
- **Connections:**
  - Output → **Wait**
- **Edge cases / failures:**
  - Headers must match exactly; note that the code outputs `Hook Option 1/2` but the append mapping expects `Hook (Option 1)` / `Hook (Option 2)`. In this workflow, the mapping uses:
    - `Hook (Option 1)` = `{{$json["Hook Option 1"]}}`
    - `Hook (Option 2)` = `{{$json["Hook Option 2"]}}`
  - Large content may exceed cell limits; Google Sheets has per-cell character limits (~50k)

#### Node: Wait
- **Type / Role:** `n8n-nodes-base.wait` (v1.1) — throttling/pausing mechanism to control pacing.
- **Configuration (interpreted):**
  - No explicit wait duration configured in parameters (must be set in UI or defaults; as-is, it may behave as “resume later” style wait depending on node settings)
- **Connections:**
  - Output → **Loop Over Items** (continues the loop)
- **Edge cases / failures:**
  - If wait is not configured with a duration or resume condition, executions may stall indefinitely
  - Webhook-based resume can be misconfigured if not used intentionally

**Sticky note covering this block:**
- **“Step 3: Post Creation & Final Storage”** (looping, post generation, storage)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheets Trigger | googleSheetsTrigger | Polling trigger on planning sheet changes | — | Get row(s) in sheet | Step 1: Input Capture & Trigger |
| Get row(s) in sheet | googleSheets | Read planning inputs | Google Sheets Trigger | Message a model | Step 1: Input Capture & Trigger |
| Message a model | langchain.openAi | Generate calculations + content pillars JSON | Get row(s) in sheet | Code in JavaScript | Smart Content Calendar Orchestration Workflow |
| Code in JavaScript | code | Parse/validate pillars; map to sheet rows; weight checks | Message a model | Append row in sheet | Smart Content Calendar Orchestration Workflow |
| Append row in sheet | googleSheets | Append pillars to Pillars tab | Code in JavaScript | Message a model7 | Step 2: Calendar Generation & Routing |
| Message a model7 | langchain.openAi | Generate calendar JSON from pillars + constraints | Append row in sheet | Code in JavaScript7 | Step 2: Calendar Generation & Routing |
| Code in JavaScript7 | code | Parse/validate calendar; map to calendar rows | Message a model7 | Append row in sheet6 | Step 2: Calendar Generation & Routing |
| Append row in sheet6 | googleSheets | Append calendar rows to Calendar tab | Code in JavaScript7 | Switch By Format | Step 2: Calendar Generation & Routing |
| Switch By Format | switch | Route items by Format (Video vs not Video) | Append row in sheet6 | Loop Over Items (non-video); (video path unused) | Step 2: Calendar Generation & Routing |
| Loop Over Items | splitInBatches | Iterate calendar items one-by-one (rate-limit control) | Switch By Format; Wait | Message a model6 | Step 3: Post Creation & Final Storage |
| Message a model6 | langchain.openAi | Generate final post JSON per calendar item | Loop Over Items | Code in JavaScript6 | Step 3: Post Creation & Final Storage |
| Code in JavaScript6 | code | Parse post JSON; merge in original Date; shape final row | Message a model6 | Append row in sheet7 | Step 3: Post Creation & Final Storage |
| Append row in sheet7 | googleSheets | Append final post content to Final Posts tab | Code in JavaScript6 | Wait | Step 3: Post Creation & Final Storage |
| Wait | wait | Throttle/resume loop execution | Append row in sheet7 | Loop Over Items | Step 3: Post Creation & Final Storage |
| Sticky Note4 | stickyNote | Documentation (workflow purpose + setup) | — | — |  |
| Sticky Note5 | stickyNote | Documentation (Step 1) | — | — |  |
| Sticky Note6 | stickyNote | Documentation (Step 2) | — | — |  |
| Sticky Note7 | stickyNote | Documentation (Step 3) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: *Smart Content Calendar Orchestration Workflow* (or your choice)
   - Ensure workflow settings use default execution order (v1) unless you need v2.

2. **Prepare Google Sheets structure (3 tabs + 1 planning tab)**
   - **Planning tab (gid=0)**: must include columns referenced by expressions:
     - Brand Name, Industry, Platform, Target Audience, Content Goal, Tone
     - Start Date, Duration (Days), Posting Frequency / Week
     - Promotion Level, Max Promo Posts / Week, Min Educational Posts / Week
     - CTA Cooldown (Days), Max Same Pillar Streak
   - **Pillars tab** headers:
     - Pillar Name, Primary Purpose, Platform, Weight (%), Allowed Formats, CTA Allowed
   - **Calendar tab** headers:
     - Sr No, Date, Platform, Pillar ID, Format, Post Intent, Hook Angle, CTA Type, Status
   - **Final Posts tab** headers:
     - Sr No, Date, Hook (Option 1), Hook (Option 2), Main Content, CTA, Hashtags, Platform Variant

3. **Add node: Google Sheets Trigger**
   - Type: **Google Sheets Trigger**
   - Credentials: connect Google account (OAuth2) with access to the spreadsheet
   - Set:
     - Document: your spreadsheet
     - Sheet: Planning tab (gid=0)
     - Poll time: Every hour
     - Include in output: Both

4. **Add node: Get row(s) in sheet**
   - Type: **Google Sheets**
   - Operation: **Get row(s)** (read)
   - Same Document + Planning tab
   - Connect: **Google Sheets Trigger → Get row(s) in sheet**

5. **Add node: Message a model (Pillars)**
   - Type: **OpenAI (LangChain) → Message a model**
   - Credentials: OpenAI API key configured in n8n
   - Model: **gpt-4o**
   - Add 2 messages:
     - System: the pillar architect JSON-only constraints (as in workflow)
     - User: includes the calculation rules and maps `{{$json[...]}}` fields from the planning row
   - Connect: **Get row(s) in sheet → Message a model**

6. **Add node: Code (Parse pillars)**
   - Type: **Code**
   - Language: JavaScript
   - Paste logic to:
     - Extract `output_text` from `items[0].json.output[0].content`
     - Strip code fences, parse JSON
     - Validate `contentPillars` and platform weights sum to 100 per platform
     - Output one item per pillar with keys matching your Pillars tab headers
   - Connect: **Message a model → Code**

7. **Add node: Google Sheets (Append pillars)**
   - Type: **Google Sheets**
   - Operation: **Append**
   - Sheet: Pillars tab
   - Map columns from the Code node output:
     - Platform, Weight (%), CTA Allowed, Pillar Name, Allowed Formats, Primary Purpose
   - Connect: **Code → Append pillars**

8. **Add node: Message a model (Calendar)**
   - Type: **OpenAI (LangChain) → Message a model**
   - Model: **gpt-4o**
   - System: calendar strategist JSON-only + constraints
   - User:
     - Reference planning values via `$('Get row(s) in sheet').item.json[...]`
     - Provide pillar list via `{{ JSON.stringify($input.all()) }}`
   - Connect: **Append pillars → Message a model (Calendar)**

9. **Add node: Code (Parse calendar)**
   - Type: **Code**
   - Parse `output_text`, strip fences, `JSON.parse`
   - Map `contentCalendar[]` into items with headers for Calendar tab:
     - Sr No, Date, Platform, Pillar ID, Format, Post Intent, Hook Angle, CTA Type, Status
   - Connect: **Message a model (Calendar) → Code (Parse calendar)**

10. **Add node: Google Sheets (Append calendar)**
   - Type: **Google Sheets**
   - Operation: **Append**
   - Sheet: Calendar tab
   - Map fields from parsed calendar items
   - Connect: **Code (Parse calendar) → Append calendar**

11. **Add node: Switch (By Format)**
   - Type: **Switch**
   - Add rules:
     - If `{{$json.Format}}` equals `Video` → Output 0
     - If `{{$json.Format}}` not equals `Video` → Output 1
   - Connect: **Append calendar → Switch**
   - (Optional) Implement the Video branch later; currently only non-video continues.

12. **Add node: Split in Batches (Loop Over Items)**
   - Type: **Split in Batches**
   - Set batch size to **1** (recommended for rate limiting)
   - Connect: **Switch Output 1 → Loop Over Items**

13. **Add node: Message a model (Post generation)**
   - Type: **OpenAI (LangChain) → Message a model**
   - Model: **gpt-4o**
   - System: copywriter JSON-only constraints (two hooks, full content, hashtags string, CTA rules)
   - User: map from loop item:
     - Sr No, Platform, Pillar ID, Post Intent, Format, Hook Angle, CTA Type
   - Connect: **Loop Over Items → Message a model (Post generation)**

14. **Add node: Code (Parse post JSON + add Date)**
   - Type: **Code**
   - Parse AI JSON and merge `Date` from `$('Loop Over Items').item.json.Date`
   - Output fields matching Final Posts headers (or intermediate keys with mapping expressions)
   - Connect: **Message a model (Post generation) → Code (Parse post)**

15. **Add node: Google Sheets (Append final post)**
   - Type: **Google Sheets**
   - Operation: **Append**
   - Sheet: Final Posts tab
   - Map:
     - Sr No, Date, Hook (Option 1/2), Main Content, CTA, Hashtags, Platform Variant
   - Connect: **Code (Parse post) → Append final post**

16. **Add node: Wait (Throttle)**
   - Type: **Wait**
   - Configure a fixed delay (e.g., 1–3 seconds) or a rate-limit-safe interval suitable for your OpenAI quota
   - Connect: **Append final post → Wait**
   - Connect: **Wait → Loop Over Items** to continue next batch

17. **Credentials checklist**
   - Google Sheets OAuth2 credential with edit access to the spreadsheet
   - OpenAI credential authorized to use `gpt-4o`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Smart Content Calendar Orchestration Workflow” note explains end-to-end behavior and setup steps (trigger → pillars → calendar → routing → looped post generation → Sheets storage). | Embedded as a sticky note inside the workflow |
| Step notes: “Step 1: Input Capture & Trigger”, “Step 2: Calendar Generation & Routing”, “Step 3: Post Creation & Final Storage”. | Embedded as sticky notes inside the workflow |
| Important functional gap: Video branch exists in Switch but is not connected (video items are dropped). | Switch By Format node behavior |
| Potential mismatch: Pillar parser writes `Allowed Formats` using `pillar_type` (Educational/Promotional/…). | Code in JavaScript (pillar mapping) |
| CTA cooldown validation in calendar parser uses a hard-coded 5 (by index), not the sheet’s `CTA Cooldown (Days)` and not actual date deltas. | Code in JavaScript7 |

Disclaimer (provided by user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.