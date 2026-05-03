Score and route qualified leads to Notion and Matrix

https://n8nworkflows.xyz/workflows/score-and-route-qualified-leads-to-notion-and-matrix-13116


# Score and route qualified leads to Notion and Matrix

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Lead Scoring Pipeline – Matrix + Notion  
**Purpose:** On a schedule, fetch new form leads, validate and enrich them (Clearbit), compute a numeric lead score, then:
- **Qualified (score ≥ 75):** create a Notion CRM record + notify a Matrix room
- **Not qualified:** still create a Notion record as “Disqualified” for archive/history

**Primary use cases**
- Sales/marketing qualification automation
- ICP-based scoring using firmographic and role/title signals
- Centralized storage in Notion + real-time team alerting via Matrix

### Logical blocks
1. **1.1 Orchestration & Lead Intake**: schedule trigger → fetch lead list → early exit if empty → batch iteration  
2. **1.2 Validation & Rate Limiting**: basic email presence check → wait to reduce API pressure  
3. **1.3 Enrichment & Scoring**: normalize lead fields → Clearbit combined enrichment → merge → score calculation  
4. **1.4 Storage & Routing + Notification**: build Notion properties → qualified gate → Notion create + Matrix message (qualified) OR Notion archive (disqualified)

---

## 2. Block-by-Block Analysis

### 2.1 Orchestration & Lead Intake

**Overview:** Triggers every 15 minutes, pulls “new” leads from an external form provider API, and avoids downstream work when nothing is returned. Processes items sequentially via batching.

**Nodes involved:**
- Schedule – Every 15 min
- Fetch New Leads
- IF – Any Leads?
- Split In Batches

#### Node: Schedule – Every 15 min
- **Type / role:** `Schedule Trigger` — workflow entry point on a time interval.
- **Config choices:** Runs every **15 minutes** (`minutesInterval: 15`).
- **Input/Output:** No inputs; outputs to **Fetch New Leads**.
- **Edge cases:** None typical; if n8n instance is down, runs are missed (unless using queued/external orchestration).

#### Node: Fetch New Leads
- **Type / role:** `HTTP Request` — fetch lead submissions from provider.
- **Config choices:**
  - **URL:** `https://api.yourformprovider.com/v1/leads?status=new`
  - **Auth:** `predefinedCredentialType` using `httpHeaderAuth` (expects an API key or token in headers).
- **Input/Output:** Receives trigger signal; outputs response items to **IF – Any Leads?**.
- **Key variables/expressions:** none in URL besides fixed endpoint.
- **Edge cases / failures:**
  - 401/403 if header auth misconfigured
  - Non-JSON or unexpected schema (e.g., object instead of array)
  - Pagination not handled (if API paginates, you may miss leads)
  - Rate limits/timeouts from provider

#### Node: IF – Any Leads?
- **Type / role:** `IF` — gate to stop the run when the fetched list is empty.
- **Config choices:**
  - Condition: **Number**: `{{ $items.length }} > 0`
- **Input/Output:**
  - Input from **Fetch New Leads**
  - **True path** → **Split In Batches**
  - **False path** not connected (workflow ends)
- **Edge cases:**
  - If the HTTP node returns a single item containing an array field (instead of returning N items), `$items.length` might be 1 even when there are “0 leads” inside the payload. In that case, you’d need to check the array length inside `$json`.
  - If upstream errors are returned as items, this may incorrectly pass.

#### Node: Split In Batches
- **Type / role:** `Split In Batches` — iterates through leads one-by-one.
- **Config choices:** Uses default options (batch size default in n8n UI; not explicitly set here).
- **Input/Output:** Input from **IF – Any Leads?**; output to **IF – Valid Email?**.
- **Edge cases:**
  - No “loop-back” connection is defined to request the next batch. In many designs, you connect the last node back into Split In Batches to continue. As provided, it may only process the first batch (depending on n8n behavior/version and whether defaults are used). If you expect to process *all leads*, add the standard continuation connection.

---

### 2.2 Validation & Rate Limiting

**Overview:** Ensures an email exists before enrichment and adds a short wait to reduce rate-limit risk on downstream APIs.

**Nodes involved:**
- IF – Valid Email?
- Wait 1 s (Rate-limit)

#### Node: IF – Valid Email?
- **Type / role:** `IF` — basic validation gate.
- **Config choices:**
  - Condition: **String**: `{{ $json.email }}` **isNotEmpty**
- **Input/Output:**
  - Input from **Split In Batches**
  - **True path** → **Wait 1 s (Rate-limit)**
  - **False path** not connected (lead is effectively dropped)
- **Edge cases:**
  - Checks only presence, not format. Invalid emails (“abc”) will pass.
  - If your lead field name differs (e.g., `Email` or nested path), condition fails and lead is dropped.

#### Node: Wait 1 s (Rate-limit)
- **Type / role:** `Wait` — simple throttling between leads.
- **Config choices:** Wait **1 second**.
- **Input/Output:** Input from **IF – Valid Email?**; output to **Set – Lead Basics**.
- **Edge cases:** In high-volume scenarios, this slows processing linearly; consider batch-size tuning or API-specific rate-limiting strategies.

---

### 2.3 Enrichment & Scoring

**Overview:** Normalizes lead data, calls Clearbit Combined API for person+company enrichment, merges results, and computes a score (0–100) with a `qualified` boolean.

**Nodes involved:**
- Set – Lead Basics
- Clearbit Enrichment
- Merge – Combine Lead & Enrichment
- Code – Calculate Lead Score

#### Node: Set – Lead Basics
- **Type / role:** `Set` — intended to shape/standardize lead fields before merge/scoring.
- **Config choices:** No explicit fields configured (options empty). As-is, it effectively passes through the input.
- **Input/Output:** Input from **Wait 1 s**; output to **Merge – Combine Lead & Enrichment** (input 0).
- **Edge cases:**
  - Since no fields are set, fields like `full_name` expected later may not exist unless provided by the source or Clearbit.
  - If you intended to map `name → full_name`, you should add explicit set/mapping here.

#### Node: Clearbit Enrichment
- **Type / role:** `HTTP Request` — calls Clearbit Combined endpoint.
- **Config choices:**
  - **URL:** `https://person.clearbit.com/v2/combined/find?email={{$json.email}}`
  - **Auth:** `httpHeaderAuth` credential (should include `Authorization: Bearer <CLEARBIT_KEY>` or Clearbit-required header format).
- **Input/Output:**
  - Input from upstream item stream (implicitly from the same lead item)
  - Output to **Merge – Combine Lead & Enrichment** (input 1)
- **Key expressions:** `{{$json.email}}` injected into URL.
- **Edge cases / failures:**
  - 404/202-like “not found” behavior (Clearbit often returns not found for unknown emails)
  - 401 if key missing/invalid
  - Rate limiting (429)
  - If Clearbit returns an error body, merge/scoring may break unless handled

#### Node: Merge – Combine Lead & Enrichment
- **Type / role:** `Merge` — stitches lead basics with Clearbit response.
- **Config choices:** `mergeByIndex` (assumes both inputs are aligned by item index).
- **Input/Output:**
  - Input 0: **Set – Lead Basics**
  - Input 1: **Clearbit Enrichment**
  - Output: **Code – Calculate Lead Score**
- **Edge cases:**
  - With async failures (e.g., Clearbit returns fewer items or errors), index merge can misalign. Alternative: merge by a key (email) if needed.
  - If Clearbit returns `null` fields, later code must guard.

#### Node: Code – Calculate Lead Score
- **Type / role:** `Code` — computes numeric score and qualified flag.
- **Config choices (logic):**
  - Reads from `item.json` and uses:
    - `company.metrics.employees` (default 0)
    - `company.metrics.alexa` (default 1,000,000)
    - `title` regex match for seniority (`director|vp|chief|head`)
  - Scoring:
    - employees > 50 → +30
    - employees > 200 → +20 (cumulative; so >200 gives 50)
    - senior title → +20
    - alexa < 200000 → +20
    - cap at 100
  - Outputs: adds `score` and `qualified: score >= 75`
- **Input/Output:** Input from Merge; output to **Set – Build Notion Props**
- **Edge cases:**
  - If Clearbit response structure differs (e.g., metrics not present), defaults prevent crashes, but scoring may be too low.
  - `data.title` might be missing if not provided by Clearbit or lead source.
  - If `company` is a string instead of object, `company.metrics` access may fail—current code uses `const company = data.company || {};` but later uses `const metrics = company.metrics || {};` which will throw if `company` is a string. (Because `"Acme".metrics` is undefined but not a throw; however `company.metrics` on a string returns undefined in JS without throwing—still okay. The bigger risk is later parts of workflow expecting `company.name`.)

---

### 2.4 Storage & Routing + Notification

**Overview:** Prepares Notion fields, routes by qualification, creates the appropriate Notion database page, and (for qualified leads) sends a Matrix message.

**Nodes involved:**
- Set – Build Notion Props
- IF – Qualified?
- Notion – Create Qualified Lead Page
- Code – Build Matrix Message
- Matrix Notify – New Qualified Lead
- Notion – Archive Disqualified Lead

#### Node: Set – Build Notion Props
- **Type / role:** `Set` — intended to assemble/standardize fields used by Notion nodes.
- **Config choices:** No explicit fields configured (options empty). Currently pass-through.
- **Input/Output:** Input from **Code – Calculate Lead Score**; output to **IF – Qualified?**
- **Edge cases:** If you expect to set `notion_database_id`, `full_name`, or flatten `company`, you must configure it here; otherwise Notion mapping relies on upstream fields existing.

#### Node: IF – Qualified?
- **Type / role:** `IF` — routes qualified vs disqualified.
- **Config choices:** Boolean: `{{ $json.qualified }}` isTrue
- **Input/Output:**
  - True → **Notion – Create Qualified Lead Page** AND **Code – Build Matrix Message** (two parallel outputs)
  - False → **Notion – Archive Disqualified Lead**
- **Edge cases:**
  - If `qualified` is missing, it evaluates false → disqualified path.
  - Parallel true-path plus an additional connection from Notion node to Matrix code means Matrix message can be built twice (see below).

#### Node: Notion – Create Qualified Lead Page
- **Type / role:** `Notion` — create a page in a database (CRM entry).
- **Config choices:**
  - Resource: `databasePage` (create database page)
  - Database ID: `{{ $json.notion_database_id || 'YOUR_NOTION_DATABASE_ID' }}`
  - Properties:
    - **Name** (Title): `{{ $json.full_name }}`
    - **Email**: uses value from `$json.Email`/`$json.email` depending on Notion node mapping; here it is listed as key only, implying it takes `$json.Email` by default—this may require explicit mapping in UI.
    - **Company** (rich text): `{{ $json.company.name || $json.company }}`
    - **Score**: similar to Email—key present without explicit expression
    - **Status** (select): “Qualified”
- **Input/Output:** Input from **IF – Qualified?**; output to **Code – Build Matrix Message**
- **Edge cases / failures:**
  - Notion credential missing/invalid → auth error
  - Database properties must match exactly: `Name`, `Email`, `Company`, `Score`, `Status`
  - If `full_name` missing → Notion title empty (often invalid)
  - If Email/Score aren’t explicitly mapped, Notion may write blanks depending on node UI configuration
  - Rate limits (Notion API), especially on bursts

#### Node: Code – Build Matrix Message
- **Type / role:** `Code` — constructs a Matrix message body.
- **Config choices (logic):**
  - Uses first input item:
    - `full_name`
    - `company.name || company`
    - `score`
  - Message format:
    - `🎯 *New Qualified Lead!*`
    - Name/Company/Score with markdown-like styling
- **Input/Output:**
  - Receives input from **IF – Qualified?** (direct) *and* from **Notion – Create Qualified Lead Page**
  - Outputs to **Matrix Notify – New Qualified Lead**
- **Important edge case (duplication risk):**
  - Because it is connected both from **IF – Qualified?** and from **Notion – Create Qualified Lead Page**, it may execute twice per qualified lead (depending on how n8n schedules parallel branches). If you intended to notify only after Notion creation, remove the direct connection from the IF node and keep only the Notion → Code connection.

#### Node: Matrix Notify – New Qualified Lead
- **Type / role:** `Matrix` — sends chat message to a Matrix room.
- **Config choices:** Operation `sendMessage` (room ID and credential are configured in node credentials/UI, not visible in JSON parameters here).
- **Input/Output:** Input from **Code – Build Matrix Message**; no downstream nodes.
- **Edge cases / failures:**
  - Credential/room misconfiguration
  - Homeserver connectivity issues
  - Message formatting differences (Matrix clients may not render markdown the same way)

#### Node: Notion – Archive Disqualified Lead
- **Type / role:** `Notion` — creates a Notion database page for non-qualified leads.
- **Config choices:**
  - Same database ID expression as qualified node
  - Same properties, except **Status** = “Disqualified”
- **Input/Output:** Input from **IF – Qualified?** false branch; end of branch.
- **Edge cases:** Same as qualified Notion node (property schema, missing title/email, rate limits).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📌 Lead Scoring Pipeline – Overview | Sticky Note | Documentation / setup notes | — | — | ## How it works; Setup steps 1–7 (credentials, Notion DB props, Matrix room, scoring logic, activation) |
| 🗂️ Data Collection | Sticky Note | Documentation for intake block | — | — | Describes schedule, fetch, empty gate, batching |
| 🔍 Enrichment & Scoring | Sticky Note | Documentation for enrichment/scoring | — | — | Describes validation, Clearbit, merge, scoring customization |
| 💾 Storage & Notification | Sticky Note | Documentation for storage/notify | — | — | Describes Notion storage for both, Matrix notify, message customization |
| Schedule – Every 15 min | Schedule Trigger | Time-based entry point | — | Fetch New Leads | Describes schedule, fetch, empty gate, batching |
| Fetch New Leads | HTTP Request | Pull new leads from form provider API | Schedule – Every 15 min | IF – Any Leads? | Describes schedule, fetch, empty gate, batching |
| IF – Any Leads? | IF | Stop if no leads | Fetch New Leads | Split In Batches | Describes schedule, fetch, empty gate, batching |
| Split In Batches | Split In Batches | Process leads sequentially | IF – Any Leads? | IF – Valid Email? | Describes schedule, fetch, empty gate, batching |
| IF – Valid Email? | IF | Validate presence of email | Split In Batches | Wait 1 s (Rate-limit) | Describes validation, Clearbit, merge, scoring customization |
| Wait 1 s (Rate-limit) | Wait | Throttle requests | IF – Valid Email? | Set – Lead Basics | Describes validation, Clearbit, merge, scoring customization |
| Set – Lead Basics | Set | Normalize lead fields (currently pass-through) | Wait 1 s (Rate-limit) | Merge – Combine Lead & Enrichment | Describes validation, Clearbit, merge, scoring customization |
| Clearbit Enrichment | HTTP Request | Enrich lead via Clearbit Combined API | (same item stream) | Merge – Combine Lead & Enrichment | Describes validation, Clearbit, merge, scoring customization |
| Merge – Combine Lead & Enrichment | Merge | Merge lead and enrichment payloads | Set – Lead Basics; Clearbit Enrichment | Code – Calculate Lead Score | Describes validation, Clearbit, merge, scoring customization |
| Code – Calculate Lead Score | Code | Compute `score` and `qualified` | Merge – Combine Lead & Enrichment | Set – Build Notion Props | Describes validation, Clearbit, merge, scoring customization |
| Set – Build Notion Props | Set | Prepare Notion-ready fields (currently pass-through) | Code – Calculate Lead Score | IF – Qualified? | Describes Notion storage, Matrix notify, message customization |
| IF – Qualified? | IF | Route qualified vs disqualified | Set – Build Notion Props | Notion – Create Qualified Lead Page; Code – Build Matrix Message; Notion – Archive Disqualified Lead | Describes Notion storage, Matrix notify, message customization |
| Notion – Create Qualified Lead Page | Notion | Create Notion page for qualified lead | IF – Qualified? | Code – Build Matrix Message | Describes Notion storage, Matrix notify, message customization |
| Code – Build Matrix Message | Code | Build message text payload | IF – Qualified?; Notion – Create Qualified Lead Page | Matrix Notify – New Qualified Lead | Describes Notion storage, Matrix notify, message customization |
| Matrix Notify – New Qualified Lead | Matrix | Send Matrix room notification | Code – Build Matrix Message | — | Describes Notion storage, Matrix notify, message customization |
| Notion – Archive Disqualified Lead | Notion | Create Notion page for disqualified lead | IF – Qualified? | — | Describes Notion storage, Matrix notify, message customization |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Lead Scoring Pipeline – Matrix + Notion”**
   - (Optional) Add sticky notes with the same section titles and descriptions.

2. **Add trigger**
   - Add node: **Schedule Trigger**
   - Configure: **Every 15 minutes**

3. **Fetch new leads (form provider)**
   - Add node: **HTTP Request** named **Fetch New Leads**
   - Method: GET
   - URL: `https://api.yourformprovider.com/v1/leads?status=new`
   - Authentication: **Predefined Credential Type → HTTP Header Auth**
   - Create credential:
     - Add required header(s) for your provider (e.g., `Authorization: Bearer <token>` or `X-API-Key: <key>`).
   - Connect: **Schedule Trigger → Fetch New Leads**

4. **Gate if empty**
   - Add node: **IF** named **IF – Any Leads?**
   - Condition (Number):
     - Value 1: `{{ $items.length }}`
     - Operation: **larger**
     - Value 2: `0`
   - Connect: **Fetch New Leads → IF – Any Leads?**
   - Connect **true** output to next step (leave false unconnected to end run).

5. **Batching**
   - Add node: **Split In Batches**
   - (Recommended) set **Batch Size = 1** for strict sequential processing
   - Connect: **IF – Any Leads? (true) → Split In Batches**
   - Note: if you need to process all leads, add a loop-back from the end of each branch to **Split In Batches** “Next batch”.

6. **Validate email**
   - Add node: **IF** named **IF – Valid Email?**
   - Condition (String):
     - Value 1: `{{ $json.email }}`
     - Operation: **isNotEmpty**
   - Connect: **Split In Batches → IF – Valid Email?** (true continues)

7. **Rate limit**
   - Add node: **Wait** named **Wait 1 s (Rate-limit)**
   - Unit: seconds, Value: **1**
   - Connect: **IF – Valid Email? (true) → Wait**

8. **Normalize lead fields**
   - Add node: **Set** named **Set – Lead Basics**
   - Add fields as needed (recommended):
     - `full_name` = `{{ $json.name || $json.full_name }}`
     - `email` = `{{ $json.email }}`
     - `company` = `{{ $json.company || '' }}`
   - Connect: **Wait → Set – Lead Basics**

9. **Clearbit enrichment**
   - Add node: **HTTP Request** named **Clearbit Enrichment**
   - Method: GET
   - URL: `https://person.clearbit.com/v2/combined/find?email={{$json.email}}`
   - Authentication: **HTTP Header Auth**
   - Credential: set Clearbit auth header as required (commonly `Authorization: Bearer <CLEARBIT_API_KEY>`)
   - Connect it so it runs for each lead item (typically from **Set – Lead Basics** in the main chain, or simply place it downstream and merge later).

10. **Merge lead + enrichment**
    - Add node: **Merge** named **Merge – Combine Lead & Enrichment**
    - Mode: **Merge By Index**
    - Connect:
      - **Set – Lead Basics → Merge (Input 0)**
      - **Clearbit Enrichment → Merge (Input 1)**

11. **Score calculation**
    - Add node: **Code** named **Code – Calculate Lead Score**
    - Paste the provided scoring logic (adjust weights/thresholds):
      - Employees, Alexa rank, senior title regex
      - Output `score` and `qualified: score >= 75`
    - Connect: **Merge → Code – Calculate Lead Score**

12. **Prepare Notion fields**
    - Add node: **Set** named **Set – Build Notion Props**
    - Recommended explicit fields (to avoid Notion blanks):
      - `notion_database_id` = your database ID (or keep a constant)
      - `full_name`, `email`, `company_name` (flatten), `score`, `qualified`
    - Connect: **Code – Calculate Lead Score → Set – Build Notion Props**

13. **Routing**
    - Add node: **IF** named **IF – Qualified?**
    - Boolean condition:
      - Value 1: `{{ $json.qualified }}`
      - Operation: **isTrue**
    - Connect: **Set – Build Notion Props → IF – Qualified?**

14. **Notion: qualified lead page**
    - Add node: **Notion** named **Notion – Create Qualified Lead Page**
    - Resource: **Database Page**
    - Operation: **Create**
    - Database ID: `{{ $json.notion_database_id || 'YOUR_NOTION_DATABASE_ID' }}`
    - Map properties (must exist in the Notion DB):
      - Name (Title): `{{ $json.full_name }}`
      - Email: map to `{{ $json.email }}`
      - Company: `{{ $json.company.name || $json.company }}` (or your flattened field)
      - Score: `{{ $json.score }}`
      - Status (Select): `Qualified`
    - Credentials: configure Notion integration credential in n8n.
    - Connect: **IF – Qualified? (true) → Notion – Create Qualified Lead Page**

15. **Build Matrix message**
    - Add node: **Code** named **Code – Build Matrix Message**
    - Use logic:
      - Build `message` using `full_name`, `company`, `score`
    - Connect (recommended): **Notion – Create Qualified Lead Page → Code – Build Matrix Message**
    - Important: avoid also connecting **IF – Qualified? (true)** directly to this node unless you want duplicate sends.

16. **Matrix notification**
    - Add node: **Matrix** named **Matrix Notify – New Qualified Lead**
    - Operation: **sendMessage**
    - Configure credentials and **room ID** in node settings.
    - Message content: map from `{{ $json.message }}`
    - Connect: **Code – Build Matrix Message → Matrix Notify – New Qualified Lead**

17. **Notion: disqualified archive**
    - Add node: **Notion** named **Notion – Archive Disqualified Lead**
    - Same Database ID and property mapping as qualified, but:
      - Status = `Disqualified`
    - Connect: **IF – Qualified? (false) → Notion – Archive Disqualified Lead**

18. **Activate**
    - Test with a sample lead payload.
    - Activate workflow once credentials (Form API, Clearbit, Notion, Matrix) and database schema are confirmed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Configure Leads API credentials + URL in “Fetch New Leads”. | From workflow overview sticky note |
| Add Clearbit API key as HTTP Header Auth credential. | From workflow overview sticky note |
| Notion DB must include properties: Name, Email, Company, Score, Status. | From workflow overview sticky note |
| Configure Notion credential and set database ID in Notion nodes. | From workflow overview sticky note |
| Supply Matrix credential + target room ID in Matrix node. | From workflow overview sticky note |
| Adjust scoring logic in “Code – Calculate Lead Score”. | From workflow overview and enrichment sticky note |
| Activate only after credentials/IDs are in place. | From workflow overview sticky note |