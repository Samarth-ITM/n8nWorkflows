Send LinkedIn connection requests to competitor post commenters via ConnectSafely.ai

https://n8nworkflows.xyz/workflows/send-linkedin-connection-requests-to-competitor-post-commenters-via-connectsafely-ai-13094


# Send LinkedIn connection requests to competitor post commenters via ConnectSafely.ai

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** *Send LinkedIn connection requests to competitor post commenters via ConnectSafely.ai*  
**Workflow name (JSON):** *Competitor Poach - Auto Daily Limit*

**Purpose:**  
Collect commenters from a specified LinkedIn post (typically a competitor’s post), deduplicate them into unique profiles, then send LinkedIn connection requests through the ConnectSafely.ai API while enforcing a **daily send limit**. When the daily limit is reached, the workflow **waits until just after midnight** and automatically resumes the next day. It also **filters out competitor employees** before sending.

**Primary use cases:**
- Sales: connect with engaged commenters on competitor posts
- Recruiting: identify active, relevant professionals
- Marketing/BD: grow a targeted network with light personalization

### 1.1 Input Reception (Form)
User submits the post URL, competitor name, and optional message preferences via an n8n form trigger.

### 1.2 Configuration + Comment Collection
Workflow normalizes inputs (lowercase competitor, boolean sendMessage, default template, dailyLimit) and calls ConnectSafely.ai to fetch up to 500 post comments.

### 1.3 Profile Extraction + Deduplication
Transforms raw comments into a list of unique LinkedIn profiles (by profileUrl) to process one-by-one.

### 1.4 Processing Loop with Daily Limit + Auto-Resume
Iterates through profiles sequentially:
- checks daily quota stored in workflow static data
- if quota exceeded → calculates seconds to next midnight → Wait → continue

### 1.5 Profile Enrichment + Filtering + Sending
Fetches each profile’s details, filters competitor employees, builds a message (optional), sends the connection request, increments the daily counter on success, then rate-limits with a short wait.

### 1.6 Aggregation + Final Summary
Aggregates per-profile results and outputs a final summary containing counts (sent/skipped/failed) and the updated daily total.

---

## 2. Block-by-Block Analysis

### Block 1 — Setup & Data Collection
**Overview:** Collects user inputs from a form, normalizes them into a consistent config object, then requests all comments from the target LinkedIn post via ConnectSafely.ai.

**Nodes involved:**
- Form Trigger
- Set Config
- Get Comments

#### Node: Form Trigger
- **Type / role:** `Form Trigger` — interactive entry point; starts the workflow on form submission.
- **Configuration choices:**
  - Form title: “Competitor Post Scraper”
  - Description: indicates scraping + auto-send with daily max and auto-continue
  - Fields:
    - **LinkedIn Post URL** (required)
    - **Competitor Name** (required)
    - **Send Message?** dropdown: Yes/No (required)
    - **Message Template** textarea (optional)
- **Key variables produced:**  
  `$json['LinkedIn Post URL']`, `$json['Competitor Name']`, `$json['Send Message?']`, `$json['Message Template']`
- **Connections:**
  - Output → **Set Config**
- **Potential failures / edge cases:**
  - User enters non-Linkedin URL or malformed post URL; downstream API call may fail or return no comments.
  - Empty template is allowed; workflow applies a default later.

#### Node: Set Config
- **Type / role:** `Set` — normalizes input and sets defaults used across the workflow.
- **Configuration choices (interpreted):**
  - `dailyLimit` (number): **10** (note: sticky note mentions default 8; actual value here is 10)
  - `postUrl`: from form field
  - `competitor`: lowercased competitor name for case-insensitive matching
  - `sendMessage`: boolean derived from dropdown (`Yes` → true)
  - `template`: uses form template or default:  
    `Hi {firstName}, saw your comment. Would love to connect!`
- **Key expressions:**
  - `postUrl = {{ $json['LinkedIn Post URL'] }}`
  - `competitor = {{ $json['Competitor Name'].toLowerCase() }}`
  - `sendMessage = {{ $json['Send Message?'] === 'Yes' }}`
  - `template = {{ $json['Message Template'] || 'Hi {firstName}, saw your comment. Would love to connect!' }}`
- **Connections:**
  - Input ← Form Trigger
  - Output → Get Comments
- **Edge cases / failures:**
  - If competitor name is missing (should be required by the form), `.toLowerCase()` would throw. Form requirement prevents this unless bypassed via webhook testing.

#### Node: Get Comments
- **Type / role:** `HTTP Request` — calls ConnectSafely.ai to retrieve comments.
- **Configuration choices:**
  - POST `https://api.connectsafely.ai/linkedin/posts/comments/all`
  - JSON body includes:
    - `postUrl` from config
    - `maxComments: 500`
  - Auth: `httpBearerAuth` (ConnectSafely.ai API key)
- **Key expressions:**
  - Body: `{{ JSON.stringify({ postUrl: $json.postUrl, maxComments: 500 }) }}`
- **Connections:**
  - Input ← Set Config
  - Output → Extract Profiles
- **Potential failures / edge cases:**
  - 401/403 if bearer token invalid/expired.
  - 429 if API rate-limits.
  - Non-standard response shape; next node partially handles array-wrapped responses.
  - LinkedIn post has comments disabled or inaccessible to the API agent → empty comments.

---

### Block 2 — Comment Processing (Extract + Deduplicate)
**Overview:** Converts the comments payload into a list of unique target profiles and passes them into a batch loop.

**Nodes involved:**
- Extract Profiles
- Loop

#### Node: Extract Profiles
- **Type / role:** `Code` — parses API response, extracts and deduplicates profiles, and attaches config to each profile item.
- **Configuration choices (logic):**
  - Reads config from `Set Config` via node lookup.
  - Supports response either as `{ comments: [...] }` or `[{ comments: [...] }]`.
  - Deduplicates by `profileUrl` using a `Set`.
  - Emits one item per unique profile with:
    - `profileId` from `publicIdentifier`
    - `name`, `profileUrl`
    - `competitor`, `sendMessage`, `template`, `dailyLimit`
- **Key variables / lookups:**
  - `const config = $('Set Config').first().json;`
  - `const response = $input.first().json;`
- **Connections:**
  - Input ← Get Comments
  - Output → Loop
- **Edge cases / failures:**
  - If comments are missing or empty, returns one item: `{ error: 'No comments found', profiles: [] }`  
    However, it still outputs an item, which will enter the loop and may later fail because `profileId` is missing.
  - If `publicIdentifier` is missing, downstream “Get Profile” may fail.
  - If ConnectSafely.ai changes response schema, parsing may break.

#### Node: Loop
- **Type / role:** `Split In Batches` — iterates through items sequentially.
- **Configuration choices:**
  - Batch options not explicitly set (defaults; in modern n8n this usually means batch size = 1 unless configured otherwise).
- **Connections (important):**
  - Input ← Extract Profiles
  - **Output 0 (per batch item)** → Check Limit
  - **Output 1 (when no items left)** → Aggregate
- **Edge cases / failures:**
  - If the prior node emitted an error item without `profileId`, the loop still processes it unless explicitly filtered.

---

### Block 3 — Daily Limit Enforcement + Auto-Wait
**Overview:** Before attempting any send, checks a persistent daily counter stored in workflow static data. If at limit, schedules a wait until just after midnight and resets the counter.

**Nodes involved:**
- Check Limit
- Limit Reached?
- Wait Until Tomorrow
- Wait

#### Node: Check Limit
- **Type / role:** `Code` — reads and updates daily quota state (date/count) in global workflow static data.
- **Configuration choices (logic):**
  - `today = YYYY-MM-DD` in UTC based on `toISOString()`
  - Resets count if static date != today
  - `DAILY_LIMIT = item.dailyLimit || 8`
  - Outputs `limitReached`, `sentToday`, and `dailyLimit`
- **Key variables:**
  - `const staticData = $getWorkflowStaticData('global');`
- **Connections:**
  - Input ← Loop (per item)
  - Output → Limit Reached?
- **Edge cases / failures:**
  - Timezone: uses UTC midnight; may not match your business day local time.
  - In clustered/multi-instance n8n, static data persistence depends on execution mode; may not behave like a shared counter across replicas.
  - If item lacks `dailyLimit`, defaults to 8 (but Set Config sets 10).

#### Node: Limit Reached?
- **Type / role:** `IF` — routes execution depending on `limitReached`.
- **Condition:** `{{ $json.limitReached }} equals true`
- **Connections:**
  - **True** → Wait Until Tomorrow
  - **False** → Get Profile
- **Edge cases:**
  - If `limitReached` is undefined due to upstream unexpected item shape, strict boolean comparison may evaluate to false; it will proceed to Get Profile.

#### Node: Wait Until Tomorrow
- **Type / role:** `Code` — computes sleep duration and resets daily counter for the next date.
- **Logic:**
  - Sets “tomorrow” to next day at **00:00:05**
  - Computes `waitSeconds = ceil((tomorrow - now)/1000)`
  - Writes static data:
    - `staticData.date = newDate` (tomorrow’s date)
    - `staticData.count = 0`
  - Outputs `waitSeconds`, `resumeAt`, and resets `sentToday` to 0
- **Connections:**
  - Input ← Limit Reached? (true)
  - Output → Wait
- **Edge cases / failures:**
  - Long waits rely on n8n Wait node behavior and instance uptime.
  - Because it resets the counter *immediately* for tomorrow’s date before the wait completes, concurrent executions could send earlier than intended if they share static data.

#### Node: Wait
- **Type / role:** `Wait` — pauses execution for `waitSeconds`.
- **Configuration:** `amount = {{ $json.waitSeconds }}`
- **Connections:**
  - Input ← Wait Until Tomorrow
  - Output → Get Profile
- **Edge cases / failures:**
  - If n8n restarts, wait resumption behavior depends on n8n configuration and persistence.
  - If `waitSeconds` is negative due to clock skew, node may fail.

---

### Block 4 — Profile Enrichment, Filtering, Sending, Rate Limiting
**Overview:** Retrieves full profile details, skips competitor employees, optionally generates a personalized message, sends a connection request, increments the daily counter on success, and throttles via a 3-second wait.

**Nodes involved:**
- Get Profile
- Prepare & Filter
- Should Send?
- Send Connection
- Log & Increment
- Wait 3s

#### Node: Get Profile
- **Type / role:** `HTTP Request` — fetches a LinkedIn profile’s details from ConnectSafely.ai.
- **Configuration choices:**
  - POST `https://api.connectsafely.ai/linkedin/profile`
  - Body: `{ profileId }`
  - Auth: bearer token
- **Key expressions:**
  - `{{ JSON.stringify({ profileId: $json.profileId }) }}`
- **Connections:**
  - Input ← Limit Reached? (false) OR Wait (after sleeping)
  - Output → Prepare & Filter
- **Edge cases / failures:**
  - Missing `profileId` → API error.
  - Profile not accessible / private → incomplete data; downstream checks rely on `currentCompany`/`headline`.

#### Node: Prepare & Filter
- **Type / role:** `Code` — decides whether to skip, and builds message.
- **Configuration choices (logic):**
  - Reads profile from HTTP response (`$input.first().json`)
  - Reads current loop item using: `$('Loop').first().json` (important coupling)
  - Competitor employee check:
    - `company = (profile.currentCompany || profile.headline || '').toLowerCase()`
    - skip if `company.includes(item.competitor)`
  - Message generation:
    - only if `item.sendMessage` true and template present
    - replaces `{firstName}` with `profile.firstName` OR first token of `item.name` OR `"there"`
    - enforces max length 300 chars (truncates to 297 + `...`)
  - Output schema (send path): `{ skip:false, profileId, name, firstName, message }`
  - Output schema (skip path): `{ skip:true, reason:'competitor_employee', profileId, name }`
- **Connections:**
  - Input ← Get Profile
  - Output → Should Send?
- **Edge cases / failures:**
  - `$('Loop').first()` references the first item in the Loop context; in some n8n scenarios this can be fragile. Prefer `$json` propagation or paired-item usage if refactoring.
  - `includes` substring match can create false positives (e.g., “Not Salesforce”).
  - If profile has neither currentCompany nor headline, competitor filtering is bypassed.

#### Node: Should Send?
- **Type / role:** `IF` — routes based on `skip`.
- **Condition:** `{{ $json.skip }} equals false`
- **Connections:**
  - **True** → Send Connection
  - **False** → Wait 3s (then continues loop)
- **Edge cases:**
  - If `skip` undefined, strict comparison to `false` fails; it routes to “false” output (skipping send).

#### Node: Send Connection
- **Type / role:** `HTTP Request` — sends a LinkedIn connection request via ConnectSafely.ai.
- **Configuration choices:**
  - POST `https://api.connectsafely.ai/linkedin/connect`
  - Body always includes `profileId`
  - If message exists, adds `customMessage`
  - Auth: bearer token
- **Key expression (body builder):**
  - Builds an object then `JSON.stringify(b)`
- **Connections:**
  - Input ← Should Send? (true)
  - Output → Log & Increment
- **Edge cases / failures:**
  - LinkedIn weekly invitation limit, account restrictions, or already-connected state.
  - Message may be rejected by API if too long or invalid characters (workflow truncates length but does not sanitize).
  - 429 or transient errors; no retries configured.

#### Node: Log & Increment
- **Type / role:** `Code` — interprets send result, increments daily static counter on success, and emits a simplified result.
- **Logic:**
  - `sent = res.success === true`
  - If sent:
    - `staticData.count = (staticData.count || 0) + 1`
  - Outputs: `{ profileId, name, sent, error }`
- **Connections:**
  - Input ← Send Connection
  - Output → Wait 3s
- **Edge cases / failures:**
  - Assumes response has boolean `success`; if API returns different schema, `sent` becomes false.
  - Does not capture skip events; those are not aggregated into results in the current wiring (see Aggregation block).

#### Node: Wait 3s
- **Type / role:** `Wait` — throttles requests to reduce rate-limit risk.
- **Configuration:** fixed `amount = 3`
- **Connections:**
  - Input ← Log & Increment OR Should Send? (false)
  - Output → Loop (to request next batch item)
- **Edge cases:**
  - Fixed wait may be insufficient if API or LinkedIn rate-limits aggressively.

---

### Block 5 — Results & Response (Aggregation + Summary)
**Overview:** After the loop finishes, aggregates all items that reach the loop’s “done” output, then produces a summary response.

**Nodes involved:**
- Aggregate
- Summary

#### Node: Aggregate
- **Type / role:** `Aggregate` — collects all item data into a single array field.
- **Configuration choices:**
  - Mode: aggregate all item data
  - Output field: `results`
- **Connections:**
  - Input ← Loop (when completed / no items left)
  - Output → Summary
- **Edge cases / failures:**
  - With current connections, only items on the loop completion path are aggregated; ensure all per-profile result items actually continue the loop correctly (they do via Wait 3s → Loop).
  - Skip items currently go to Wait 3s but do not produce a `{skip:true}` item for aggregation; only the “Prepare & Filter” output contains skip info, but it is not routed into aggregation. As a result, `skipped` count in Summary will likely remain 0.

#### Node: Summary
- **Type / role:** `Code` — computes totals and emits final structured output.
- **Logic:**
  - `results = $input.first().json.results || []`
  - `sent = results.filter(r => r.sent).length`
  - `skipped = results.filter(r => r.skip).length` (likely always 0 with current wiring)
  - `failed = results.length - sent - skipped`
  - reads `staticData.count` as `dailyCountNow`
- **Connections:**
  - Input ← Aggregate
  - Output → (end)
- **Edge cases / failures:**
  - If results contain only `{profileId, name, sent, error}`, there is no `skip` field; skipped count will be 0 and “failed” may be inflated.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form Trigger | Form Trigger | Workflow entry; collect LinkedIn post URL, competitor, and messaging prefs | — | Set Config | ## 1. Setup & Data Collection |
| Set Config | Set | Normalize inputs; set daily limit and message defaults | Form Trigger | Get Comments | ## 1. Setup & Data Collection |
| Get Comments | HTTP Request | Fetch up to 500 comments from ConnectSafely.ai | Set Config | Extract Profiles | ## 1. Setup & Data Collection |
| Extract Profiles | Code | Parse API response; dedupe commenters into unique profiles | Get Comments | Loop | ## 2. Processing Loop with Daily Limit |
| Loop | Split In Batches | Iterate profiles sequentially; route completion to aggregation | Extract Profiles; Wait 3s | Check Limit; Aggregate | ## 2. Processing Loop with Daily Limit |
| Check Limit | Code | Enforce daily sending quota via static data | Loop | Limit Reached? | ## 2. Processing Loop with Daily Limit |
| Limit Reached? | IF | Branch: wait until tomorrow vs proceed | Check Limit | Wait Until Tomorrow; Get Profile | ## 2. Processing Loop with Daily Limit |
| Wait Until Tomorrow | Code | Compute seconds to midnight+5s; reset quota | Limit Reached? (true) | Wait | ## 2. Processing Loop with Daily Limit |
| Wait | Wait | Sleep until next day then resume | Wait Until Tomorrow | Get Profile | ## 2. Processing Loop with Daily Limit |
| Get Profile | HTTP Request | Fetch profile data from ConnectSafely.ai | Limit Reached? (false); Wait | Prepare & Filter | ## 2. Processing Loop with Daily Limit |
| Prepare & Filter | Code | Skip competitor employees; build optional message | Get Profile | Should Send? | ## 2. Processing Loop with Daily Limit |
| Should Send? | IF | Branch: send connection vs skip | Prepare & Filter | Send Connection; Wait 3s | ## 2. Processing Loop with Daily Limit |
| Send Connection | HTTP Request | Send LinkedIn connection request (optional custom message) | Should Send? (true) | Log & Increment | ## 2. Processing Loop with Daily Limit |
| Log & Increment | Code | Record send result and increment daily counter if success | Send Connection | Wait 3s | ## 2. Processing Loop with Daily Limit |
| Wait 3s | Wait | Throttle between iterations | Log & Increment; Should Send? (false) | Loop | ## 2. Processing Loop with Daily Limit |
| Aggregate | Aggregate | Collect all processed results into `results` array | Loop (done output) | Summary | ## 3. Results & Response |
| Summary | Code | Compute sent/skipped/failed totals; output final report | Aggregate | — | ## 3. Results & Response |
| Sticky Note | Sticky Note | Documentation / purpose / setup notes | — | — | ## Competitor Poach - Auto Daily Limit; Automatically scrape LinkedIn post commenters and send connection requests with built-in daily limits and auto-scheduling.; Setup: Add ConnectSafely.ai API credentials (Header Auth); Configure dailyLimit in Set Config (default mentioned: 8); Adjust Wait 3s for rate limiting. |
| Sticky Note1 | Sticky Note | Section label: Setup & Data Collection | — | — | ## 1. Setup & Data Collection |
| Sticky Note2 | Sticky Note | Section label: Processing Loop with Daily Limit | — | — | ## 2. Processing Loop with Daily Limit |
| Sticky Note3 | Sticky Note | Section label: Results & Response | — | — | ## 3. Results & Response |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: **“Competitor Poach - Auto Daily Limit”**.

2. **Add node: Form Trigger** (`Form Trigger`)
   - Title: `Competitor Post Scraper`
   - Description: `Scrape competitor post and auto-send connections (max 8/day, auto-continues next day)`
   - Fields:
     1) `LinkedIn Post URL` (required, text)  
     2) `Competitor Name` (required, text)  
     3) `Send Message?` (required, dropdown): options `Yes`, `No`  
     4) `Message Template` (textarea, optional)

3. **Add node: Set Config** (`Set`)
   - Add fields:
     - `dailyLimit` = number `10` (set to your desired daily cap)
     - `postUrl` = expression: `{{ $json['LinkedIn Post URL'] }}`
     - `competitor` = expression: `{{ $json['Competitor Name'].toLowerCase() }}`
     - `sendMessage` = expression: `{{ $json['Send Message?'] === 'Yes' }}`
     - `template` = expression:  
       `{{ $json['Message Template'] || 'Hi {firstName}, saw your comment. Would love to connect!' }}`
   - Connect: **Form Trigger → Set Config**

4. **Create credentials for ConnectSafely.ai**
   - Create an **HTTP Bearer Auth** credential in n8n containing your ConnectSafely.ai API key.
   - (Optional/legacy) The JSON lists an additional `httpHeaderAuth` credential, but the nodes are configured to use **Bearer** auth (`genericAuthType: httpBearerAuth`). Ensure the HTTP Request nodes use the bearer credential.

5. **Add node: Get Comments** (`HTTP Request`)
   - Method: `POST`
   - URL: `https://api.connectsafely.ai/linkedin/posts/comments/all`
   - Authentication: **Generic Credential Type → HTTP Bearer Auth**
   - Body: JSON
     - Use expression for the JSON body:
       `{{ JSON.stringify({ postUrl: $json.postUrl, maxComments: 500 }) }}`
   - Connect: **Set Config → Get Comments**

6. **Add node: Extract Profiles** (`Code`)
   - Paste logic implementing:
     - read config from `Set Config`
     - parse response comments (support array wrapper)
     - dedupe by `profileUrl`
     - output one item per profile with `profileId`, `name`, `profileUrl`, plus config fields
   - Connect: **Get Comments → Extract Profiles**

7. **Add node: Loop** (`Split In Batches`)
   - Keep defaults (or set batch size to 1 for strict sequential processing).
   - Connect: **Extract Profiles → Loop**

8. **Add node: Check Limit** (`Code`)
   - Implement:
     - `staticData = $getWorkflowStaticData('global')`
     - reset `count` when date changes (UTC date string)
     - determine `limitReached = count >= dailyLimit`
     - output augmented item
   - Connect: **Loop (main output) → Check Limit**

9. **Add node: Limit Reached?** (`IF`)
   - Condition: Boolean equals
     - Left: `{{ $json.limitReached }}`
     - Right: `true`
   - Connect: **Check Limit → Limit Reached?**

10. **Add node: Wait Until Tomorrow** (`Code`) for the *true* branch
   - Compute:
     - next day 00:00:05
     - `waitSeconds`
   - Reset in static data:
     - set `date` to tomorrow
     - set `count` to 0
   - Connect: **Limit Reached? (true) → Wait Until Tomorrow**

11. **Add node: Wait** (`Wait`)
   - Amount: expression `{{ $json.waitSeconds }}`
   - Connect: **Wait Until Tomorrow → Wait**

12. **Add node: Get Profile** (`HTTP Request`)
   - Method: `POST`
   - URL: `https://api.connectsafely.ai/linkedin/profile`
   - Auth: HTTP Bearer Auth (ConnectSafely.ai)
   - Body JSON expression:
     - `{{ JSON.stringify({ profileId: $json.profileId }) }}`
   - Connect:
     - **Limit Reached? (false) → Get Profile**
     - **Wait → Get Profile**

13. **Add node: Prepare & Filter** (`Code`)
   - Implement:
     - competitor employee filter using `currentCompany` or `headline`
     - message generation if enabled:
       - replace `{firstName}`
       - truncate to 300 chars
     - output `{skip:false,...}` or `{skip:true, reason:'competitor_employee', ...}`
   - Connect: **Get Profile → Prepare & Filter**

14. **Add node: Should Send?** (`IF`)
   - Condition: `{{ $json.skip }}` equals `false`
   - Connect: **Prepare & Filter → Should Send?**

15. **Add node: Send Connection** (`HTTP Request`) for the *true* branch
   - Method: `POST`
   - URL: `https://api.connectsafely.ai/linkedin/connect`
   - Auth: HTTP Bearer Auth
   - Body JSON expression:
     - Build `{ profileId }` and include `customMessage` only if message exists.
   - Connect: **Should Send? (true) → Send Connection**

16. **Add node: Log & Increment** (`Code`)
   - If API response indicates success, increment `staticData.count`.
   - Output a result object `{profileId, name, sent, error}`.
   - Connect: **Send Connection → Log & Increment**

17. **Add node: Wait 3s** (`Wait`)
   - Amount: `3` seconds
   - Connect:
     - **Log & Increment → Wait 3s**
     - **Should Send? (false) → Wait 3s** (so skips also advance the loop)

18. **Close the loop**
   - Connect: **Wait 3s → Loop** (to process the next profile)

19. **Add node: Aggregate** (`Aggregate`) for completion
   - Configure to aggregate all item data into field `results`.
   - Connect: **Loop (done output / second output) → Aggregate**

20. **Add node: Summary** (`Code`)
   - Compute totals and include `dailyCountNow` from static data.
   - Connect: **Aggregate → Summary**

21. **(Optional but recommended) Fix skipped counting**
   - If you want `skipped` to be accurate, ensure skipped items are also emitted into the same results stream that reaches `Aggregate` (e.g., by standardizing outputs from both send and skip paths and merging before returning to the loop).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Competitor Poach - Auto Daily Limit: Automatically scrape LinkedIn post commenters and send connection requests with built-in daily limits and auto-scheduling.” | Workflow purpose (sticky overview) |
| “Setup steps: Add ConnectSafely.ai API credentials (Header Auth). Configure dailyLimit in Set Config node (default: 8).” | Sticky overview (note: current Set Config uses 10; Check Limit defaults to 8 if missing) |
| “Customization: Change dailyLimit, modify message template, adjust Wait 3s for different rate limiting.” | Sticky overview |
| “Requirements: ConnectSafely.ai account with API access; n8n credentials.” | Sticky overview |
| Sections labeled: “1. Setup & Data Collection”, “2. Processing Loop with Daily Limit”, “3. Results & Response” | Sticky section notes |