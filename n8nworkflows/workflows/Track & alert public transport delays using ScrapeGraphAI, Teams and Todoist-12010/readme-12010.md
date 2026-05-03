Track & alert public transport delays using ScrapeGraphAI, Teams and Todoist

https://n8nworkflows.xyz/workflows/track---alert-public-transport-delays-using-scrapegraphai--teams-and-todoist-12010


# Track & alert public transport delays using ScrapeGraphAI, Teams and Todoist

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Public Transport Delay Tracker with Microsoft Teams and Todoist  
**Stated title:** Track & alert public transport delays using ScrapeGraphAI, Teams and Todoist

**Purpose:**  
This workflow receives a public transport query (line + stop) via an HTTP webhook, builds a transit “next departures” URL, uses **ScrapeGraphAI** to extract the next departures and delay minutes, then:
- Sends a **Microsoft Teams** alert only when a departure is delayed (`delayMinutes > 0`)
- Always creates a **Todoist** task for each extracted departure (delayed or not)

### 1.1 Trigger & Input Reception
Entry via webhook; validate required fields (`line`, `stop`) early to prevent downstream failures.

### 1.2 URL Building & Batching
Construct a transit authority URL for the requested line/stop and pass through batching (scales to multiple URLs later).

### 1.3 AI Scraping & Data Normalization
ScrapeGraphAI extracts structured departure data; code flattens the array into per-departure items and prepares metadata (title, Teams text).

### 1.4 Conditional Alerting (Teams) + Persistent Logging (Todoist)
If delayed: format and send Teams message, then continue to Todoist.  
If on time: skip Teams and go directly to Todoist.

---

## 2. Block-by-Block Analysis

### Block A — Trigger & Input Validation
**Overview:** Accepts inbound POST requests and enforces the presence of `line` and `stop` before doing any external calls.

**Nodes involved:**  
- Webhook – Incoming Request  
- Validate & Parse Request

#### Node: Webhook – Incoming Request
- **Type / role:** `n8n-nodes-base.webhook` — workflow entry point (HTTP trigger).
- **Key configuration:**
  - **HTTP Method:** `POST`
  - **Path:** `public-transport-tracker`
  - Produces the request body as the initial item JSON.
- **Inputs / outputs:**
  - **Input:** none (trigger)
  - **Output:** to **Validate & Parse Request**
- **Version-specific notes:** typeVersion `1.1` (standard webhook behavior).
- **Failure modes / edge cases:**
  - Wrong HTTP method (GET instead of POST) won’t trigger.
  - If request body isn’t JSON (or is empty), downstream validation will likely fail.
  - If webhook is in test mode vs production URL, callers may hit the wrong endpoint.

#### Node: Validate & Parse Request
- **Type / role:** `n8n-nodes-base.code` — validates payload schema and normalizes output.
- **Key configuration choices (interpreted):**
  - Iterates over incoming items (`$input.all()`).
  - Requires `body.line` and `body.stop`; otherwise **throws** an Error.
  - Emits normalized JSON: `{ line, stop }`.
- **Key expressions / variables:**
  - Uses `item.json` as body.
  - Throws: `Request body must include "line" and "stop"`.
- **Inputs / outputs:**
  - **Input:** Webhook body
  - **Output:** to **Generate Target URLs**
- **Version-specific notes:** typeVersion `2` (Code node runtime).
- **Failure modes / edge cases:**
  - Missing fields → hard fail (execution error).
  - `line`/`stop` present but not strings (e.g., objects) could produce an invalid URL later.

---

### Block B — URL Generation & Batching
**Overview:** Builds the transit authority endpoint using `line` and URL-encoded `stop`, then hands the URL to batching.

**Nodes involved:**  
- Generate Target URLs  
- Split URLs

#### Node: Generate Target URLs
- **Type / role:** `n8n-nodes-base.code` — constructs the target URL to scrape.
- **Key configuration choices:**
  - Encodes stop: `encodeURIComponent($json.stop)`
  - Builds: `https://www.citytransitauthority.com/api/next-departures?line=${line}&stop=${stop}`
  - Outputs `{ url, line }`
- **Key expressions / variables:**
  - `$json.line`, `$json.stop`
- **Inputs / outputs:**
  - **Input:** validated `{ line, stop }`
  - **Output:** to **Split URLs**
- **Failure modes / edge cases:**
  - If `line` contains unsafe characters, it is *not* encoded (only `stop` is encoded).
  - Domain is a placeholder-style hostname; if not real/reachable, scraping will fail.

#### Node: Split URLs
- **Type / role:** `n8n-nodes-base.splitInBatches` — processes items in batches (here effectively 1-by-1).
- **Key configuration choices:**
  - Default options (batch size not explicitly set in parameters).
- **Inputs / outputs:**
  - **Input:** URL item(s)
  - **Output:** to **Scrape Schedules & Delays**
- **Version-specific notes:** typeVersion `3`.
- **Failure modes / edge cases:**
  - With multiple URLs, you usually need a loop-back pattern to fetch next batches; this workflow does not include a “continue” loop connection, so it behaves as a pass-through for typical single-item usage.

---

### Block C — Scraping & Transformation
**Overview:** Uses ScrapeGraphAI to extract departures, then converts them into individual items enriched with fields used by Teams and Todoist steps.

**Nodes involved:**  
- Scrape Schedules & Delays  
- Clean & Enrich Data

#### Node: Scrape Schedules & Delays
- **Type / role:** `n8n-nodes-scrapegraphai.scrapegraphAi` — AI-assisted scraping/structuring.
- **Key configuration choices:**
  - **Website URL:** `={{ $json.url }}`
  - **Prompt:** “Extract the next three departures… Return JSON array named `departures`.”
- **Inputs / outputs:**
  - **Input:** `{ url, line }`
  - **Output:** to **Clean & Enrich Data**
- **Version-specific notes:** typeVersion `1` (ScrapeGraphAI community/partner node behavior may vary by package version).
- **Failure modes / edge cases:**
  - Credential/auth failure for ScrapeGraphAI.
  - Target site unreachable, blocked, rate-limited, or returns unexpected format.
  - Model may return malformed JSON or wrong key name (not `departures`), causing downstream code to break.
  - If the endpoint is already JSON, AI scraping still depends on prompt adherence.

#### Node: Clean & Enrich Data
- **Type / role:** `n8n-nodes-base.code` — transforms `departures[]` into per-departure items.
- **Key configuration choices (interpreted):**
  - Assumes `$json.departures` is an array.
  - Emits one n8n item per departure with:
    - `line` (pulled from the *original* item via `$item(0).$json.line`)
    - `destination`
    - `departureISO`
    - `delayMinutes` as number (defaults to 0)
    - `title` string
    - `msTeamsMessage` string
- **Key expressions / variables:**
  - `const data = $json.departures;`
  - `delayMinutes: Number(dep.delay_minutes) || 0`
  - Uses `$item(0).$json.line` to reference another item’s JSON.
- **Inputs / outputs:**
  - **Input:** ScrapeGraphAI output (expects `departures`)
  - **Output:** to **Delay > 0?**
- **Failure modes / edge cases:**
  - If `departures` is missing/not an array → runtime error (`data.map is not a function`).
  - `$item(0)` referencing can break if item linking doesn’t match expectations (especially if multiple items/URLs are processed).
  - Field naming mismatch: prompt expects `departure_time` and `delay_minutes`; if the model returns different keys, fields become `undefined`.

---

### Block D — Conditional Logic, Teams Alerts, and Todoist Tasks
**Overview:** Routes delayed departures to Teams alerting, then logs each departure into Todoist.

**Nodes involved:**  
- Delay > 0?  
- Prepare Teams Message  
- Send Teams Alert  
- Prepare Todoist Task  
- Create Todoist Task

#### Node: Delay > 0?
- **Type / role:** `n8n-nodes-base.if` — branch based on delay.
- **Key configuration choices:**
  - Number condition: `{{ $json.delayMinutes }}` **larger than** `0`
- **Inputs / outputs:**
  - **Input:** enriched per-departure items
  - **Output (true):** to **Prepare Teams Message**
  - **Output (false):** to **Prepare Todoist Task**
- **Failure modes / edge cases:**
  - If `delayMinutes` is missing/non-numeric string, conversion happened earlier; if still invalid it may evaluate unexpectedly.

#### Node: Prepare Teams Message
- **Type / role:** `n8n-nodes-base.set` — intended to map fields for Teams posting.
- **Key configuration choices:**
  - **No fields are defined** in the node parameters shown (only `options`), so it currently does not create/rename fields.
- **Inputs / outputs:**
  - **Input:** delayed departures from IF (true branch)
  - **Output:** to **Send Teams Alert**
- **Failure modes / edge cases:**
  - Downstream Teams node uses `$json.message`, but upstream data created `msTeamsMessage`, not `message`. Since this Set node does not map fields, Teams message expression will likely render `undefined` for `$json.message` unless callers already provided it.

#### Node: Send Teams Alert
- **Type / role:** `n8n-nodes-base.microsoftTeams` — posts a chat message (HTML content type).
- **Key configuration choices:**
  - **Resource:** `chatMessage`
  - **Content type:** `html`
  - **Message expression:** `{{ '<b>' + $json.title + ':</b> ' + $json.message }}`
  - **chatId:** configured as “list” but **value is empty** in the provided workflow.
- **Inputs / outputs:**
  - **Input:** output of Prepare Teams Message
  - **Output:** to **Prepare Todoist Task** (flow rejoins after alert)
- **Version-specific notes:** typeVersion `2`.
- **Failure modes / edge cases:**
  - Missing Teams credentials/OAuth scopes.
  - Empty `chatId` → node cannot send.
  - Uses `$json.message` but the workflow produces `$json.msTeamsMessage`; message may be blank/undefined.
  - HTML formatting may be restricted depending on Teams API context.

#### Node: Prepare Todoist Task
- **Type / role:** `n8n-nodes-base.set` — intended to map enriched data into Todoist task fields.
- **Key configuration choices:**
  - **No fields are defined** in the node parameters shown, so it currently does not set `content` or other expected values.
- **Inputs / outputs:**
  - **Input:** from IF false branch (on-time) **or** from Teams alert node (delayed)
  - **Output:** to **Create Todoist Task**
- **Failure modes / edge cases:**
  - Todoist node expects `$json.content`, but this Set node does not create it; task creation likely fails unless content already exists.

#### Node: Create Todoist Task
- **Type / role:** `n8n-nodes-base.todoist` — creates a task in Todoist.
- **Key configuration choices:**
  - **Content:** `={{ $json.content }}`
  - **Project:** list-mode but **value is empty** in the provided workflow.
- **Inputs / outputs:**
  - **Input:** output of Prepare Todoist Task
  - **Output:** terminal node (no outgoing connections)
- **Version-specific notes:** typeVersion `1`.
- **Failure modes / edge cases:**
  - Missing Todoist credentials/token.
  - Empty project selection may default to Inbox in some configurations, but with `project.value` empty it may error depending on node behavior/version.
  - If `$json.content` is undefined → API rejects (task content is required).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook – Incoming Request | Webhook | Entry point (HTTP POST trigger) | — | Validate & Parse Request | ## Trigger & Input Validation\n\nThe first two functional nodes form the entry point for the entire automation. The webhook listens for HTTP POST requests so any external system—mobile app, chatbot, or another workflow—can supply line and stop information in real time. Using a webhook keeps latency low and avoids polling schedules that might miss last-minute changes. Immediately after the trigger, the **Validate & Parse Request** Code node checks that the payload includes both a `line` and a `stop`. Throwing an error early prevents wasted API calls and ensures the workflow processes only well-formed requests. Parsing here also normalizes the stop name, converting any special characters to a URL-safe format so downstream nodes can build clean query strings for the transit API. |
| Validate & Parse Request | Code | Validate request body and output normalized `{line, stop}` | Webhook – Incoming Request | Generate Target URLs | ## Trigger & Input Validation\n\nThe first two functional nodes form the entry point for the entire automation. The webhook listens for HTTP POST requests so any external system—mobile app, chatbot, or another workflow—can supply line and stop information in real time. Using a webhook keeps latency low and avoids polling schedules that might miss last-minute changes. Immediately after the trigger, the **Validate & Parse Request** Code node checks that the payload includes both a `line` and a `stop`. Throwing an error early prevents wasted API calls and ensures the workflow processes only well-formed requests. Parsing here also normalizes the stop name, converting any special characters to a URL-safe format so downstream nodes can build clean query strings for the transit API. |
| Generate Target URLs | Code | Build transit authority “next departures” URL | Validate & Parse Request | Split URLs | ## Scraping & Transformation\n\nAfter generating the target URL, the Split In Batches node lets you scale beyond a single stop or line by queuing multiple requests one at a time. This prevents rate-limit issues with the transit authority site. **Scrape Schedules & Delays** leverages ScrapeGraphAI to interpret HTML or JSON responses using an LLM, reliably pulling departure times, destinations, and delay minutes even if the website layout changes. The following **Clean & Enrich Data** Code node flattens the scraped array into individual items and injects helpful metadata such as a human-readable title and a pre-formatted Microsoft Teams message. This transformation keeps later nodes simple because they no longer need to understand raw scrape output. |
| Split URLs | Split In Batches | Batch/queue URL processing | Generate Target URLs | Scrape Schedules & Delays | ## Scraping & Transformation\n\nAfter generating the target URL, the Split In Batches node lets you scale beyond a single stop or line by queuing multiple requests one at a time. This prevents rate-limit issues with the transit authority site. **Scrape Schedules & Delays** leverages ScrapeGraphAI to interpret HTML or JSON responses using an LLM, reliably pulling departure times, destinations, and delay minutes even if the website layout changes. The following **Clean & Enrich Data** Code node flattens the scraped array into individual items and injects helpful metadata such as a human-readable title and a pre-formatted Microsoft Teams message. This transformation keeps later nodes simple because they no longer need to understand raw scrape output. |
| Scrape Schedules & Delays | ScrapeGraphAI | Extract departures + delays into structured JSON | Split URLs | Clean & Enrich Data | ## Scraping & Transformation\n\nAfter generating the target URL, the Split In Batches node lets you scale beyond a single stop or line by queuing multiple requests one at a time. This prevents rate-limit issues with the transit authority site. **Scrape Schedules & Delays** leverages ScrapeGraphAI to interpret HTML or JSON responses using an LLM, reliably pulling departure times, destinations, and delay minutes even if the website layout changes. The following **Clean & Enrich Data** Code node flattens the scraped array into individual items and injects helpful metadata such as a human-readable title and a pre-formatted Microsoft Teams message. This transformation keeps later nodes simple because they no longer need to understand raw scrape output. |
| Clean & Enrich Data | Code | Flatten departures array into per-departure items | Scrape Schedules & Delays | Delay > 0? | ## Scraping & Transformation\n\nAfter generating the target URL, the Split In Batches node lets you scale beyond a single stop or line by queuing multiple requests one at a time. This prevents rate-limit issues with the transit authority site. **Scrape Schedules & Delays** leverages ScrapeGraphAI to interpret HTML or JSON responses using an LLM, reliably pulling departure times, destinations, and delay minutes even if the website layout changes. The following **Clean & Enrich Data** Code node flattens the scraped array into individual items and injects helpful metadata such as a human-readable title and a pre-formatted Microsoft Teams message. This transformation keeps later nodes simple because they no longer need to understand raw scrape output. |
| Delay > 0? | IF | Branch: delayed vs on-time | Clean & Enrich Data | Prepare Teams Message; Prepare Todoist Task | ## Conditional Logic & Todoist Storage\n\nThe **Delay > 0?** IF node adds smart routing without over-complicating the overall linear design. If a departure is delayed, the workflow branches to include a Teams alert before converging back to task creation. The **Prepare Todoist Task** Set node converts enriched JSON into Todoist-friendly fields—`content` for the task label, `dueDate` for reminders, and `priority` that increases when a delay is present. Finally, the **Create Todoist Task** node records each departure in the user’s Todoist inbox, giving travellers a personal itinerary that updates automatically. This storage step also serves as an audit trail, enabling users to review historical punctuality trends later. |
| Prepare Teams Message | Set | (Intended) map fields for Teams message | Delay > 0? (true) | Send Teams Alert | ## Microsoft Teams Notifications\n\nDelays often require immediate action—perhaps choosing an alternate route or informing colleagues of a late arrival. When the IF node detects any non-zero `delayMinutes`, execution continues through **Prepare Teams Message** and **Send Teams Alert**. The Set node keeps only the fields required by the Teams connector and formats HTML for improved readability. The Teams node posts directly into a designated channel, ensuring visibility for everyone who needs the update. Because the path eventually rejoins the Todoist branch, even delayed departures are still logged as tasks, giving users one consolidated view of their travel schedule while simultaneously alerting relevant stakeholders in real time. |
| Send Teams Alert | Microsoft Teams | Post delay alert message to Teams | Prepare Teams Message | Prepare Todoist Task | ## Microsoft Teams Notifications\n\nDelays often require immediate action—perhaps choosing an alternate route or informing colleagues of a late arrival. When the IF node detects any non-zero `delayMinutes`, execution continues through **Prepare Teams Message** and **Send Teams Alert**. The Set node keeps only the fields required by the Teams connector and formats HTML for improved readability. The Teams node posts directly into a designated channel, ensuring visibility for everyone who needs the update. Because the path eventually rejoins the Todoist branch, even delayed departures are still logged as tasks, giving users one consolidated view of their travel schedule while simultaneously alerting relevant stakeholders in real time. |
| Prepare Todoist Task | Set | (Intended) map fields into Todoist task format | Delay > 0? (false); Send Teams Alert | Create Todoist Task | ## Conditional Logic & Todoist Storage\n\nThe **Delay > 0?** IF node adds smart routing without over-complicating the overall linear design. If a departure is delayed, the workflow branches to include a Teams alert before converging back to task creation. The **Prepare Todoist Task** Set node converts enriched JSON into Todoist-friendly fields—`content` for the task label, `dueDate` for reminders, and `priority` that increases when a delay is present. Finally, the **Create Todoist Task** node records each departure in the user’s Todoist inbox, giving travellers a personal itinerary that updates automatically. This storage step also serves as an audit trail, enabling users to review historical punctuality trends later. |
| Create Todoist Task | Todoist | Create a task per departure | Prepare Todoist Task | — | ## Conditional Logic & Todoist Storage\n\nThe **Delay > 0?** IF node adds smart routing without over-complicating the overall linear design. If a departure is delayed, the workflow branches to include a Teams alert before converging back to task creation. The **Prepare Todoist Task** Set node converts enriched JSON into Todoist-friendly fields—`content` for the task label, `dueDate` for reminders, and `priority` that increases when a delay is present. Finally, the **Create Todoist Task** node records each departure in the user’s Todoist inbox, giving travellers a personal itinerary that updates automatically. This storage step also serves as an audit trail, enabling users to review historical punctuality trends later. |
| Workflow Overview | Sticky Note | Documentation | — | — | ## How it works\n\nThis workflow provides real-time public transport monitoring for busy commuters. A mobile app or any HTTP client sends a POST request containing a line number and stop name to the webhook trigger. The workflow validates the payload, builds the correct transit authority URL, and hands the request to ScrapeGraphAI. Using AI-powered scraping, the node extracts the next three departures along with current delay information. Cleaned results pass through a conditional check: if any departure is delayed, a Microsoft Teams message alerts the appropriate channel. Regardless of delay status, every departure becomes a Todoist task so travellers can track schedules directly within their personal productivity system.\n\n## Setup steps\n\n1. Create ScrapeGraphAI, Todoist, and Microsoft Teams credentials in n8n.\n2. Replace `{{YOUR_TEAM_ID}}` and `{{YOUR_CHANNEL_ID}}` in the Teams node.\n3. Deploy the workflow and copy the webhook URL.\n4. Send a test POST request with `{\"line\":\"A\",\"stop\":\"Central Station\"}`.\n5. Confirm Teams notifications appear when delays occur.\n6. Check Todoist for newly created tasks.\n7. Enable the workflow for continuous use. |
| Section – Trigger & Input | Sticky Note | Documentation | — | — | ## Trigger & Input Validation\n\nThe first two functional nodes form the entry point for the entire automation. The webhook listens for HTTP POST requests so any external system—mobile app, chatbot, or another workflow—can supply line and stop information in real time. Using a webhook keeps latency low and avoids polling schedules that might miss last-minute changes. Immediately after the trigger, the **Validate & Parse Request** Code node checks that the payload includes both a `line` and a `stop`. Throwing an error early prevents wasted API calls and ensures the workflow processes only well-formed requests. Parsing here also normalizes the stop name, converting any special characters to a URL-safe format so downstream nodes can build clean query strings for the transit API. |
| Section – Scraping & Transformation | Sticky Note | Documentation | — | — | ## Scraping & Transformation\n\nAfter generating the target URL, the Split In Batches node lets you scale beyond a single stop or line by queuing multiple requests one at a time. This prevents rate-limit issues with the transit authority site. **Scrape Schedules & Delays** leverages ScrapeGraphAI to interpret HTML or JSON responses using an LLM, reliably pulling departure times, destinations, and delay minutes even if the website layout changes. The following **Clean & Enrich Data** Code node flattens the scraped array into individual items and injects helpful metadata such as a human-readable title and a pre-formatted Microsoft Teams message. This transformation keeps later nodes simple because they no longer need to understand raw scrape output. |
| Section – Conditional Logic & Storage | Sticky Note | Documentation | — | — | ## Conditional Logic & Todoist Storage\n\nThe **Delay > 0?** IF node adds smart routing without over-complicating the overall linear design. If a departure is delayed, the workflow branches to include a Teams alert before converging back to task creation. The **Prepare Todoist Task** Set node converts enriched JSON into Todoist-friendly fields—`content` for the task label, `dueDate` for reminders, and `priority` that increases when a delay is present. Finally, the **Create Todoist Task** node records each departure in the user’s Todoist inbox, giving travellers a personal itinerary that updates automatically. This storage step also serves as an audit trail, enabling users to review historical punctuality trends later. |
| Section – Microsoft Teams Alerts | Sticky Note | Documentation | — | — | ## Microsoft Teams Notifications\n\nDelays often require immediate action—perhaps choosing an alternate route or informing colleagues of a late arrival. When the IF node detects any non-zero `delayMinutes`, execution continues through **Prepare Teams Message** and **Send Teams Alert**. The Set node keeps only the fields required by the Teams connector and formats HTML for improved readability. The Teams node posts directly into a designated channel, ensuring visibility for everyone who needs the update. Because the path eventually rejoins the Todoist branch, even delayed departures are still logged as tasks, giving users one consolidated view of their travel schedule while simultaneously alerting relevant stakeholders in real time. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Public Transport Delay Tracker with Microsoft Teams and Todoist* (or your preferred name).

2. **Add Trigger: Webhook**
   - Node: **Webhook**
   - Set:
     - **HTTP Method:** POST
     - **Path:** `public-transport-tracker`
   - Copy the **Test URL** for initial testing.

3. **Add Code node: Validate & Parse Request**
   - Node: **Code**
   - Paste logic that:
     - Reads incoming JSON body
     - Validates `line` and `stop` exist
     - Outputs `{ line, stop }`
   - Connect: **Webhook → Validate & Parse Request**

4. **Add Code node: Generate Target URLs**
   - Node: **Code**
   - Build URL:
     - URL-encode `stop`
     - Create `url` like: `https://www.citytransitauthority.com/api/next-departures?line=<line>&stop=<encodedStop>`
     - Output `{ url, line }`
   - Connect: **Validate & Parse Request → Generate Target URLs**

5. **Add Split In Batches: Split URLs**
   - Node: **Split In Batches**
   - Keep defaults (or set batch size = 1 explicitly).
   - Connect: **Generate Target URLs → Split URLs**

6. **Add ScrapeGraphAI node: Scrape Schedules & Delays**
   - Node: **ScrapeGraphAI** (ScrapeGraphAI integration)
   - Credentials: create/select **ScrapeGraphAI API** credentials in n8n.
   - Set:
     - **Website URL:** use expression `{{$json.url}}`
     - **Prompt:** request JSON output with `departures` array containing:
       - `departure_time` (ISO8601)
       - `destination`
       - `delay_minutes` (int, 0 if on time)
   - Connect: **Split URLs → Scrape Schedules & Delays**

7. **Add Code node: Clean & Enrich Data**
   - Node: **Code**
   - Transform `departures[]` into one item per departure, setting:
     - `delayMinutes` (number)
     - `title`
     - `msTeamsMessage`
   - Connect: **Scrape Schedules & Delays → Clean & Enrich Data**

8. **Add IF node: Delay > 0?**
   - Node: **IF**
   - Condition:
     - **Number**: `{{$json.delayMinutes}}` **larger than** `0`
   - Connect: **Clean & Enrich Data → Delay > 0?**

9. **Add Set node: Prepare Teams Message**
   - Node: **Set**
   - Create fields required by Teams node, for example:
     - `title` = `{{$json.title}}`
     - `message` = `{{$json.msTeamsMessage}}` *(important: map to `message` because Teams node references `$json.message`)*
   - Connect: **Delay > 0? (true) → Prepare Teams Message**

10. **Add Microsoft Teams node: Send Teams Alert**
   - Node: **Microsoft Teams**
   - Credentials: configure Microsoft Teams (OAuth2) in n8n.
   - Set:
     - **Resource:** Chat Message
     - Choose target **chatId** (or channel/team configuration supported by your node UI; the provided workflow leaves this blank and must be set)
     - **Content Type:** HTML
     - **Message:** `{{'<b>' + $json.title + ':</b> ' + $json.message}}`
   - Connect: **Prepare Teams Message → Send Teams Alert**

11. **Add Set node: Prepare Todoist Task**
   - Node: **Set**
   - Create at least:
     - `content` = something like `{{$json.title}}` or `{{$json.title + ' | Delay: ' + $json.delayMinutes + ' min'}}`
   - (Optional) also set Todoist fields if you plan to use them (due date, priority), matching what your Todoist node supports.
   - Connect:
     - **Delay > 0? (false) → Prepare Todoist Task**
     - **Send Teams Alert → Prepare Todoist Task** (to rejoin branches)

12. **Add Todoist node: Create Todoist Task**
   - Node: **Todoist**
   - Credentials: configure Todoist token/credentials in n8n.
   - Set:
     - **Content:** `{{$json.content}}`
     - **Project:** select a project (the provided workflow leaves it empty; set it explicitly or ensure Inbox behavior in your version)
   - Connect: **Prepare Todoist Task → Create Todoist Task**

13. **Test**
   - Execute via webhook test URL with JSON body:
     - `{"line":"A","stop":"Central Station"}`
   - Verify:
     - If delay > 0: Teams message posts.
     - All departures create Todoist tasks.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Replace `{{YOUR_TEAM_ID}}` and `{{YOUR_CHANNEL_ID}}` in the Teams node.” | Mentioned in the workflow’s overview sticky note; in the provided node config, the Teams target field is blank and must be configured. |
| Test payload example: `{"line":"A","stop":"Central Station"}` | Provided in “Workflow Overview” sticky note. |
| Important configuration gap: Teams node uses `$json.message` but workflow creates `msTeamsMessage` in code | You should map `msTeamsMessage → message` in **Prepare Teams Message** (currently empty). |
| Important configuration gap: Todoist node uses `$json.content` but no node sets it | You should set `content` in **Prepare Todoist Task** (currently empty). |