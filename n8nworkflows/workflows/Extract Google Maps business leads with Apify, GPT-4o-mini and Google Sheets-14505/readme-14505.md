Extract Google Maps business leads with Apify, GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/extract-google-maps-business-leads-with-apify--gpt-4o-mini-and-google-sheets-14505


# Extract Google Maps business leads with Apify, GPT-4o-mini and Google Sheets

Good, now I have all the information needed. Let me write the comprehensive documentation.### 1. Workflow Overview

This workflow automates the extraction of business lead data from Google Maps using Apify's Google Maps Scraper actor, enriches that data with AI-generated company descriptions and website-extracted email addresses, stores everything in Google Sheets, and notifies operators across multiple messaging platforms (Telegram, WhatsApp, Microsoft Teams, Slack) upon completion.

The logic is organized into six functional blocks:

| Block | Name | Purpose |
|-------|------|---------|
| 1 | **Trigger / Initial Data Processing** | Manually trigger the workflow, fetch the list of Apify actors, and split them into individual items. |
| 2 | **Get the Specific Scraper Task** | Loop through actors and tasks to locate a specific Apify actor and its task by configurable IDs/names. |
| 3 | **Run the Specific Scraper Task** | Execute the Apify scraper task via its API, wait for completion, retrieve the dataset results, deduplicate, and begin iterating over leads. |
| 4 | **Create Company Description & Store Lead Data** | Generate a natural-language company summary with GPT-4.1-mini, then append all lead fields plus the summary to Google Sheets. |
| 5 | **Extract Business Emails from Websites & Save to Sheet** | For each lead, fetch the business website HTML, extract the primary email with GPT-4o-mini, and update the corresponding Google Sheets row. |
| 6 | **Notifying the Results** | After all leads are processed, count them and broadcast a summary message across four notification channels. |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Trigger / Initial Data Processing

**Overview:** The workflow starts on manual click. It immediately calls the Apify API to list all actors available in the user's account, then splits the paginated `data.items` array into individual items for downstream filtering.

**Nodes Involved:**
- Start by clicking
- Get list of actors
- Split Out

**Node Details:**

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|----------------|---------------|-----------------------------|-------|--------|------------|
| **Start by clicking** | Manual Trigger | Entry point; emits a single empty item to kick off execution. | No parameters. | — | None | Get list of actors | None. |
| **Get list of actors** | Apify (n8n-nodes-apify.apify) | Calls the Apify API to retrieve all actors in the authenticated account. | Resource: `Actors`, Operation: `Get list of actors`. Uses Apify API credential. | — | Start by clicking | Split Out | **Auth error** if Apify API token is invalid or expired. **Rate limit** if account has many actors; default pagination may not fetch all. |
| **Split Out** | Split Out (n8n-nodes-base.splitOut) | Expands the `data.items` array from the Apify response into one item per actor. | Field to split: `data.items`. | — | Get list of actors | Loop Over Items (actor) | If `data.items` is missing or not an array, the node will fail. Empty arrays produce zero items. |

---

#### Block 2 — Get the Specific Scraper Task

**Overview:** This block iterates through all Apify actors to find one matching a user-configured title. Once found, it fetches all tasks for that actor, then iterates through tasks to find one matching a user-configured task ID. This two-level lookup allows the workflow to target a specific pre-configured Google Maps Scraper task.

**Nodes Involved:**
- Loop Over Items (actor)
- If (Check specific actor)
- Get list of tasks
- Split Out1
- Loop Over Items (task)
- If (Check specific task)

**Node Details:**

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|----------------|---------------|-----------------------------|-------|--------|------------|
| **Loop Over Items (actor)** | Split In Batches (v3) | Batches actor items for sequential processing; each iteration feeds one actor into the condition check. | Default options (batch size auto). | — | Split Out | Output 0 (done) → nothing; Output 1 → If (Check specific actor) | If no actors exist, the loop completes immediately with no output on batch branch. |
| **If (Check specific actor)** | IF (v2.3) | Filters for a single actor whose `title` matches a hardcoded string. | Condition: `$json.title` equals `"add_your_title/actore_name"` (string, case-sensitive, strict type validation). | `{{ $json.title }}` vs `"add_your_title/actore_name"` | Loop Over Items (actor) | True → Get list of tasks; False → Loop Over Items (actor) (continue looping) | **Must be updated** with the actual actor title before first run. Case-sensitive. If no actor matches, the loop exhausts all items with no True branch ever firing. |
| **Get list of tasks** | Apify (n8n-nodes-apify.apify) | Retrieves all tasks belonging to the matched actor. | Resource: `Actor tasks`, Operation: `Get list of tasks`. Uses same Apify credential. | — | If (Check specific actor) — true | Split Out1 | Auth errors. If the actor has no tasks, `data.items` will be empty. |
| **Split Out1** | Split Out (v1) | Splits the `data.items` array of tasks into individual items. | Field to split: `data.items`. | — | Get list of tasks | Loop Over Items (task) | Same edge case as Split Out — empty or missing array. |
| **Loop Over Items (task)** | Split In Batches (v3) | Batches task items for sequential processing. | Default options. | — | Split Out1 | Output 0 (done) → nothing; Output 1 → If (Check specific task) | Same as Loop Over Items (actor). |
| **If (Check specific task)** | IF (v2.3) | Filters for a single task whose `id` matches a hardcoded value. | Condition: `$json.id` equals `"add_you_task_ID"` (string, case-sensitive, strict type validation). | `{{ $json.id }}` vs `"add_you_task_ID"` | Loop Over Items (task) | True → HTTP (Run Scraper Task); False → Loop Over Items (task) (continue looping) | **Must be updated** with the actual task ID. If no task matches, the loop ends with no True branch ever firing. |

---

#### Block 3 — Run the Specific Scraper Task

**Overview:** Once the correct task is identified, this block triggers the task run via a direct HTTP POST to the Apify API, waits 20 seconds for the actor to finish, then fetches the resulting dataset. Duplicate leads (by title) are removed, and the remaining items are fed into a batch loop for per-lead processing.

**Nodes Involved:**
- HTTP (Run Scraper Task)
- Wait 20s for run successfully and get data
- HTTP (Get Specific Task Result)
- Remove Duplicates
- Loop Over Items (lead)

**Node Details:**

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|----------------|---------------|-----------------------------|-------|--------|------------|
| **HTTP (Run Scraper Task)** | HTTP Request (v4.3) | Sends a POST request to the Apify Actor Task Run endpoint to start the Google Maps Scraper. | URL: `https://api.apify.com/v2/actor-tasks/{{ $json.id }}/runs`; Method: POST; Headers: `Accept: application/json`, `Authorization: Bearer YOUR_TOKEN_HERE`. | `{{ $json.id }}` — the matched task ID from the previous If node. | If (Check specific task) — true | Wait 20s for run successfully and get data | **Token placeholder** must be replaced with a real Apify API token. **401** if token is wrong. **404** if task ID is invalid. **429** on rate limit. No request body is sent — the task runs with its saved input configuration in Apify. |
| **Wait 20s for run successfully and get data** | Wait (v1.1) | Pauses workflow execution for 20 seconds to give the Apify actor time to complete. | Amount: 20 seconds. Webhook-based resumption. | — | HTTP (Run Scraper Task) | HTTP (Get Specific Task Result) | 20 seconds may be insufficient for large scrapes; the dataset may not be ready, resulting in partial or empty results. The workflow does **not** poll for completion — it assumes 20s is enough. |
| **HTTP (Get Specific Task Result)** | HTTP Request (v4.3) | Retrieves the dataset items produced by the completed actor run. | URL: `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items?format=json&view=overview&clean=true`. | `{{ $json.data.defaultDatasetId }}` — comes from the run response. | Wait 20s | Remove Duplicates | **404** if dataset ID is invalid. **Empty results** if the actor hasn't finished. The `clean=true` parameter strips Apify metadata. No authentication header is set on this request — public datasets will work; private datasets will return 401. |
| **Remove Duplicates** | Remove Duplicates (v2) | Eliminates duplicate lead entries based on the `title` field. | Compare mode: `selectedFields`; Fields to compare: `title`. | — | HTTP (Get Specific Task Result) | Loop Over Items (lead) | If `title` is null/empty for some items, those may all collapse into one or be treated as unique depending on null handling. Case-sensitive comparison by default. |
| **Loop Over Items (lead)** | Split In Batches (v3) | Processes each deduplicated lead one at a time. Output 1 (each batch) goes to AI description generation; Output 0 (loop done) goes to the notification block. | Default options. | — | Remove Duplicates | Output 0 → Code (Count Total Incoming Data); Output 1 → AI Company Description Generator | Also receives loop-continuation signals from the error output of the HTTP request for website HTML and from Wait 5s (after email extraction), forming a processing loop. |

---

#### Block 4 — Create Company Description & Store Lead Data

**Overview:** For each lead, GPT-4.1-mini generates a professional one-paragraph company summary from the structured Google Maps data. This summary, along with all other lead fields, is then appended as a new row in a Google Sheet.

**Nodes Involved:**
- AI Company Description Generator
- Sheet (Store google maps/lead data)

**Node Details:**

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|----------------|---------------|-----------------------------|-------|--------|------------|
| **AI Company Description Generator** | OpenAI LangChain (v1.8) | Calls GPT-4.1-mini to produce a natural-language company summary from structured fields. | Model: `gpt-4.1-mini` (list mode). JSON output enabled (output schema key: `companySummary`). System prompt defines the assistant's role and output rules. User prompt includes `title`, `categoryName`, `street`, `city`, `countryCode`, `phone`, `url` from the current lead item. Third message instructs JSON output format. | `{{ $json.title }}`, `{{ $json.categoryName }}`, `{{ $json.street }}`, `{{ $json.city }}`, `{{ $json.countryCode }}`, `{{ $json.phone }}`, `{{ $json.url }}` | Loop Over Items (lead) — batch output | Sheet (Store google maps/lead data) | **OpenAI API key** must be configured. **Rate limit / quota** on the OpenAI account. If fields are null, the prompt will include literal "null" strings. JSON output mode may occasionally fail to produce valid JSON, causing downstream errors. |
| **Sheet (Store google maps/lead data)** | Google Sheets (v4.7) | Appends a row with 19 columns to the target spreadsheet. | Operation: `append`. Document: `Scrape Google Maps Business Leads` (ID: `1K-i-GXjv3J1k-t_JC9LA0_gGHT7F1ONLUsTpwcqyIt0`). Sheet: `Sheet1`. Columns mapped via expressions referencing `Loop Over Items (lead)` item context. | All column expressions reference `$('Loop Over Items (lead)').item.json.<field>`. The `COMPANY SUMMARY IN` column uses `$json.message.content` from the AI node output. The `PHONE` column has an inline JavaScript function to strip non-digits. | AI Company Description Generator | Get Website URLs | **⚠ Bug**: The `ADDRESS` column expression uses `('Loop Over Items')` instead of `$('Loop Over Items')` — missing the `$` prefix, which will cause a runtime expression error. **Google Sheets OAuth2** credential required. **Sheet permissions** must allow the OAuth account to edit. If the spreadsheet ID is incorrect or inaccessible, the node will fail. |

**Column mapping detail for the Sheet node:**

| Column | Source Expression | Notes |
|--------|-------------------|-------|
| TITLE | `$('Loop Over Items (lead)').item.json.title` | Business name |
| CATEGORY NAME | `$('Loop Over Items (lead)').item.json.categoryName` | Primary category |
| ADDRESS | `('Loop Over Items').item.json.address` | **BUG** — missing `$` prefix |
| STREET | `$('Loop Over Items (lead)').item.json.street` | |
| CITY | `$('Loop Over Items (lead)').item.json.city` | |
| POSTAL CODE | `$('Loop Over Items (lead)').item.json.postalCode` | |
| COUNTRY CODE | `$('Loop Over Items (lead)').item.json.countryCode` | |
| PHONE | Inline JS: strips non-digits, reformats | Regex `(\d{3})(\d{4})(\d{4})` → `$1$2$3` (no separators) |
| PHONE UNFORMATTED | `phonesUncertain \|\| phoneUnformatted \|\| 'No phone found'` | Fallback chain |
| WEBSITE | `$('Loop Over Items (lead)').item.json.website` | |
| EMAIL | `$('Loop Over Items (lead)').item.json.email` | Initially from Apify; later updated |
| TOTAL SCORE | `$('Loop Over Items (lead)').item.json.totalScore` | |
| TOTAL REVIEW | `$('Loop Over Items (lead)').item.json.reviewsCount` | |
| CATEGORIES | `$('Loop Over Items (lead)').item.json.categories` | Full category list |
| URL | `$('Loop Over Items (lead)').item.json.url` | Google Maps link |
| RANK | `$('Loop Over Items (lead)').item.json.rank` | |
| IS ADVERTISEMENT | `$('Loop Over Items (lead)').item.json.isAdvertisement` | |
| IMAGE URL | `$('Loop Over Items (lead)').item.json.imageUrl` | |
| COMPANY SUMMARY IN | `$json.message.content` | AI-generated summary |

---

#### Block 5 — Extract Business Emails from Websites & Save to Sheet

**Overview:** After a lead's data is written to the sheet, the workflow fetches the business's website HTML, passes it to GPT-4o-mini to extract the best contact email, and updates the same Google Sheet row with the extracted email. A 5-second wait prevents API rate limiting before looping back to process the next lead.

**Nodes Involved:**
- Get Website URLs
- Https (Get Raw HTML Content from Business Website)
- Extract Business Email from Website HTML (GPT-4)
- Sheet (Update Email form website)
- Wait 5s

**Node Details:**

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|----------------|---------------|-----------------------------|-------|--------|------------|
| **Get Website URLs** | Set (v3.4) | Extracts the website URL from the current lead item and exposes it as a named field `Site internet`. | Assignment: `Site internet` = `{{ $('Loop Over Items (lead)').first().json.website }}`. | `$('Loop Over Items (lead)').first().json.website` | Sheet (Store google maps/lead data) | Https (Get Raw HTML Content from Business Website) | If `website` is null/empty, the downstream HTTP request will have an empty URL and fail. The `.first()` call always references the first item in the batch context, which is correct for single-item batches. |
| **Https (Get Raw HTML Content from Business Website)** | HTTP Request (v4.2) | Fetches the raw HTML source of the business's website. | URL: `{{ $json['Site internet'] }}`. Error handling: `continueErrorOutput` — errors do not stop the workflow. | `{{ $json['Site internet'] }}` | Get Website URLs | Success → Extract Business Email from Website HTML (GPT-4); Error → Loop Over Items (lead) (skip this lead) | **SSL errors**, DNS failures, timeouts, 403/404 responses, or very large HTML pages may cause issues. The error output gracefully routes back to the loop to continue with the next lead. No authentication headers. |
| **Extract Business Email from Website HTML (GPT-4)** | OpenAI LangChain (v1.8) | Sends the fetched HTML to GPT-4o-mini with a prompt to extract the single most relevant business email. | Model: `gpt-4o-mini` (list mode). Single user message containing the extraction instructions and `{{ $json.data }}` (the HTML body). No JSON output mode — plain text response expected. | `{{ $json.data }}` — the full HTML content of the website. | Https (Get Raw HTML) — success output | Sheet (Update Email form website) | **Large HTML** may exceed the model's context window or token limit. The model may return "Null" if no email is found. The response is raw text (no JSON wrapping). OpenAI API key required. |
| **Sheet (Update Email form website)** | Google Sheets (v4.7) | Updates the existing row in Google Sheets by matching on the TITLE column, setting the EMAIL column to the extracted value. | Operation: `appendOrUpdate`. Matching column: `TITLE`. Document & sheet same as the store node. Columns written: `TITLE` (for matching) and `EMAIL` (from AI output). | `EMAIL` = `{{ $json.message.content }}`; `TITLE` = `{{ $('Loop Over Items (lead)').item.json.title }}` | Extract Business Email from Website HTML (GPT-4) | Wait 5s | If the TITLE doesn't match an existing row, a new row is appended instead of updated. The AI may return "Null" as text, which would be written literally. Google Sheets OAuth2 required. |
| **Wait 5s** | Wait (v1.1) | Pauses 5 seconds before looping back, providing a throttle to avoid OpenAI and Google Sheets rate limits. | Amount: default (5 seconds based on node name; parameter not explicitly set, defaults to 1 second — the node name suggests 5s but the parameter `amount` is not set, meaning default of 1 second). | — | Sheet (Update Email form website) | Loop Over Items (lead) (continue to next lead) | **Important**: The Wait node's `amount` parameter is not explicitly set, so it defaults to 1 second, not 5 seconds as the node name implies. Users should explicitly set `amount: 5` to match the intended behavior. |

---

#### Block 6 — Notifying the Results

**Overview:** Once all leads have been processed (the Loop Over Items (lead) batch output is done), the workflow counts total leads and broadcasts a formatted notification message to four channels simultaneously: Telegram, WhatsApp (via Rapiwa), Microsoft Teams, and Slack.

**Nodes Involved:**
- Code (Count Total Incoming Data)
- Notification message (Telegram)
- Create message (Microsoft Teams)
- Rapiwa (WhatsApp)
- Send a message (Slack)

**Node Details:**

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|------|------|----------------|---------------|-----------------------------|-------|--------|------------|
| **Code (Count Total Incoming Data)** | Code (v2) | Counts the total number of items that reached the "done" branch of the lead loop and outputs it as `total_execution`. | JavaScript: `const total = $input.all().length; return [{ json: { total_execution: total } }];` | `$input.all().length` | Loop Over Items (lead) — output 0 (done) | Notification message, Create message, Rapiwa, Send a message | If no leads were processed, `total_execution` will be 0. |
| **Notification message** | Telegram (v1.2) | Sends a text message to a Telegram chat with the scrape summary. | Chat ID: `add_your_telegram_ID` (placeholder). Message template includes date and total count. | `{{ $today.format('dd-MMM-yyyy') }}`, `{{ $json.total_execution }}` | Code (Count Total) | None (terminal) | **Chat ID must be configured**. **Telegram Bot API token** credential required. Attribution append disabled. |
| **Create message** | Microsoft Teams (v2) | Posts a channel message in Microsoft Teams with the same summary. | Team ID: `team_ID` (placeholder). Channel ID: `channel_ID` (placeholder). Resource: `channelMessage`. | Same date/count template as Telegram. | Code (Count Total) | None (terminal) | **Team ID and Channel ID** must be replaced. **Microsoft Teams OAuth2** credential required. Bot must be added to the target team/channel. |
| **Rapiwa** | Rapiwa (v1) | Sends a WhatsApp message via the Rapiwa API. | Number: `your_whatsapp_number` (placeholder). Message: same template. | Same date/count template. | Code (Count Total) | None (terminal) | **Rapiwa API credential** required. Phone number must include country code. |
| **Send a message** | Slack (v2.4) | Sends a direct message to a Slack user with the summary. | Select: `user`; User: `user_name` (placeholder, username mode). Authentication: OAuth2. | Same date/count template. | Code (Count Total) | None (terminal) | **Slack OAuth2** credential required. Bot must have `chat:write` scope. The username must exist and the bot must have permission to DM them. |

**Notification message template (shared across all channels):**
```
Map Scrape Successfully
Date: {{ $today.format('dd-MMM-yyyy') }}
Total Collected Lead: {{ $json.total_execution }}
```

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|---------------|----------------|-------------|
| Start by clicking | Manual Trigger | Workflow entry point | — | Get list of actors | # Trigger/start and Initial Data Processing Group |
| Get list of actors | Apify | Fetch all actors from Apify account | Start by clicking | Split Out | # Trigger/start and Initial Data Processing Group |
| Split Out | Split Out | Expand actor list array into individual items | Get list of actors | Loop Over Items (actor) | # Trigger/start and Initial Data Processing Group |
| Loop Over Items (actor) | Split In Batches | Batch-process actors one by one | Split Out | If (Check specific actor) | # Get the Specific Scraper Task Group |
| If (Check specific actor) | IF | Filter for a specific actor by title | Loop Over Items (actor) | True → Get list of tasks; False → Loop Over Items (actor) | # Get the Specific Scraper Task Group |
| Get list of tasks | Apify | Fetch all tasks for the matched actor | If (Check specific actor) | Split Out1 | # Get the Specific Scraper Task Group |
| Split Out1 | Split Out | Expand task list array into individual items | Get list of tasks | Loop Over Items (task) | # Get the Specific Scraper Task Group |
| Loop Over Items (task) | Split In Batches | Batch-process tasks one by one | Split Out1 | If (Check specific task) | # Get the Specific Scraper Task Group |
| If (Check specific task) | IF | Filter for a specific task by ID | Loop Over Items (task) | True → HTTP (Run Scraper Task); False → Loop Over Items (task) | # Get the Specific Scraper Task Group |
| HTTP (Run Scraper Task) | HTTP Request | Trigger the Apify actor task run via API | If (Check specific task) | Wait 20s for run successfully and get data | # Run the Specific Scraper Task Group |
| Wait 20s for run successfully and get data | Wait | Pause 20s for the actor to finish | HTTP (Run Scraper Task) | HTTP (Get Specific Task Result) | # Run the Specific Scraper Task Group |
| HTTP (Get Specific Task Result) | HTTP Request | Retrieve the dataset from the completed actor run | Wait 20s for run successfully and get data | Remove Duplicates | # Run the Specific Scraper Task Group |
| Remove Duplicates | Remove Duplicates | Deduplicate leads by title | HTTP (Get Specific Task Result) | Loop Over Items (lead) | # Run the Specific Scraper Task Group |
| Loop Over Items (lead) | Split In Batches | Process each lead sequentially; loop back after email extraction or website fetch error | Remove Duplicates; Https (Get Raw HTML) error; Wait 5s | Done → Code (Count Total Incoming Data); Batch → AI Company Description Generator | # Run the Specific Scraper Task Group |
| AI Company Description Generator | OpenAI LangChain | Generate a company summary paragraph using GPT-4.1-mini | Loop Over Items (lead) | Sheet (Store google maps/lead data) | ## Create Description for Company Information base on Task group data & update in sheet |
| Sheet (Store google maps/lead data) | Google Sheets | Append lead data + AI summary as a new row | AI Company Description Generator | Get Website URLs | ## Create Description for Company Information base on Task group data & update in sheet |
| Get Website URLs | Set | Extract the website URL field for the current lead | Sheet (Store google maps/lead data) | Https (Get Raw HTML Content from Business Website) | # Extract Business Emails from Websites and Save to Sheet |
| Https (Get Raw HTML Content from Business Website) | HTTP Request | Fetch the raw HTML of the business website | Get Website URLs | Success → Extract Business Email from Website HTML (GPT-4); Error → Loop Over Items (lead) | # Extract Business Emails from Websites and Save to Sheet |
| Extract Business Email from Website HTML (GPT-4) | OpenAI LangChain | Extract the primary business email from HTML using GPT-4o-mini | Https (Get Raw HTML Content from Business Website) | Sheet (Update Email form website) | # Extract Business Emails from Websites and Save to Sheet |
| Sheet (Update Email form website) | Google Sheets | Update the lead row in Sheets with the extracted email | Extract Business Email from Website HTML (GPT-4) | Wait 5s | # Extract Business Emails from Websites and Save to Sheet |
| Wait 5s | Wait | Pause before processing the next lead | Sheet (Update Email form website) | Loop Over Items (lead) | # Extract Business Emails from Websites and Save to Sheet |
| Code (Count Total Incoming Data) | Code | Count total leads processed | Loop Over Items (lead) — done | Notification message; Create message; Rapiwa; Send a message | # Notifying the Results Group |
| Notification message | Telegram | Send completion notification via Telegram | Code (Count Total Incoming Data) | — | # Notifying the Results Group |
| Create message | Microsoft Teams | Send completion notification via Teams | Code (Count Total Incoming Data) | — | # Notifying the Results Group |
| Rapiwa | Rapiwa | Send completion notification via WhatsApp | Code (Count Total Incoming Data) | — | # Notifying the Results Group |
| Send a message | Slack | Send completion notification via Slack | Code (Count Total Incoming Data) | — | # Notifying the Results Group |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Automated Google Maps Business Lead Scraper".

2. **Add a Manual Trigger node.**  
   - Type: `Manual Trigger`  
   - Name: "Start by clicking"  
   - No configuration needed.

3. **Add an Apify node.**  
   - Type: `Apify` (n8n-nodes-apify.apify, version 1)  
   - Name: "Get list of actors"  
   - Resource: `Actors`  
   - Operation: `Get list of actors`  
   - Credential: Create an Apify API credential with your API token.  
   - Connect: "Start by clicking" → "Get list of actors"

4. **Add a Split Out node.**  
   - Type: `Split Out` (version 1)  
   - Name: "Split Out"  
   - Field to split: `data.items`  
   - Connect: "Get list of actors" → "Split Out"

5. **Add a Split In Batches node.**  
   - Type: `Split In Batches` (version 3)  
   - Name: "Loop Over Items (actor)"  
   - Default options  
   - Connect: "Split Out" → "Loop Over Items (actor)" (input)

6. **Add an IF node.**  
   - Type: `IF` (version 2.3)  
   - Name: "If (Check specific actor)"  
   - Condition combinator: `and`  
   - Condition: Left value = `{{ $json.title }}`, Operator = `equals`, Right value = your actor title (e.g., `compass/crawler-google-places`)  
   - Type validation: strict, case-sensitive  
   - Connect: "Loop Over Items (actor)" → output 1 (each batch) → "If (Check specific actor)"  
   - Connect: "If (Check specific actor)" → true → "Get list of tasks"  
   - Connect: "If (Check specific actor)" → false → "Loop Over Items (actor)" (loop back)

7. **Add an Apify node.**  
   - Type: `Apify` (version 1)  
   - Name: "Get list of tasks"  
   - Resource: `Actor tasks`  
   - Operation: `Get list of tasks`  
   - Same Apify credential  
   - Connect: "If (Check specific actor)" → true → "Get list of tasks"

8. **Add a Split Out node.**  
   - Type: `Split Out` (version 1)  
   - Name: "Split Out1"  
   - Field to split: `data.items`  
   - Connect: "Get list of tasks" → "Split Out1"

9. **Add a Split In Batches node.**  
   - Type: `Split In Batches` (version 3)  
   - Name: "Loop Over Items (task)"  
   - Connect: "Split Out1" → "Loop Over Items (task)"

10. **Add an IF node.**  
    - Type: `IF` (version 2.3)  
    - Name: "If (Check specific task)"  
    - Condition: Left value = `{{ $json.id }}`, Operator = `equals`, Right value = your Apify task ID  
    - Connect: "Loop Over Items (task)" → output 1 → "If (Check specific task)"  
    - Connect: "If (Check specific task)" → true → "HTTP (Run Scraper Task)"  
    - Connect: "If (Check specific task)" → false → "Loop Over Items (task)" (loop back)

11. **Add an HTTP Request node.**  
    - Type: `HTTP Request` (version 4.3)  
    - Name: "HTTP (Run Scraper Task)"  
    - Method: `POST`  
    - URL: `https://api.apify.com/v2/actor-tasks/{{ $json.id }}/runs`  
    - Send Headers: enabled  
    - Header 1: `Accept` = `application/json`  
    - Header 2: `Authorization` = `Bearer YOUR_APIFY_TOKEN`  
    - No body  
    - Connect: "If (Check specific task)" → true → "HTTP (Run Scraper Task)"

12. **Add a Wait node.**  
    - Type: `Wait` (version 1.1)  
    - Name: "Wait 20s for run successfully and get data"  
    - Amount: `20` (seconds)  
    - Connect: "HTTP (Run Scraper Task)" → "Wait 20s"

13. **Add an HTTP Request node.**  
    - Type: `HTTP Request` (version 4.3)  
    - Name: "HTTP (Get Specific Task Result)"  
    - Method: `GET` (default)  
    - URL: `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items?format=json&view=overview&clean=true`  
    - No authentication headers (adjust if your dataset is private)  
    - Connect: "Wait 20s" → "HTTP (Get Specific Task Result)"

14. **Add a Remove Duplicates node.**  
    - Type: `Remove Duplicates` (version 2)  
    - Name: "Remove Duplicates"  
    - Compare: `selectedFields`  
    - Fields to compare: `title`  
    - Connect: "HTTP (Get Specific Task Result)" → "Remove Duplicates"

15. **Add a Split In Batches node.**  
    - Type: `Split In Batches` (version 3)  
    - Name: "Loop Over Items (lead)"  
    - Connect: "Remove Duplicates" → "Loop Over Items (lead)"  
    - Output 0 (done) will later connect to the Code node.  
    - Output 1 (each batch) will later connect to the AI Company Description Generator.

16. **Add an OpenAI LangChain node.**  
    - Type: `OpenAI` (@n8n/n8n-nodes-langchain.openAi, version 1.8)  
    - Name: "AI Company Description Generator"  
    - Model: `gpt-4.1-mini` (select from list)  
    - JSON Output: enabled (schema key: `companySummary`)  
    - System message: "You are an AI assistant specialized in generating professional, concise, and natural company summaries from structured data scraped from Google Maps…" (full prompt as in original)  
    - User message: includes `{{ $json.title }}`, `{{ $json.categoryName }}`, `{{ $json.street }}`, `{{ $json.city }}`, `{{ $json.countryCode }}`, `{{ $json.phone }}`, `{{ $json.url }}` with the example paragraph format.  
    - Third message: "Output the result in the JSON format companySummary"  
    - Credential: Create an OpenAI API credential with your API key.  
    - Connect: "Loop Over Items (lead)" → output 1 → "AI Company Description Generator"

17. **Add a Google Sheets node.**  
    - Type: `Google Sheets` (version 4.7)  
    - Name: "Sheet (Store google maps/lead data)"  
    - Operation: `append`  
    - Document ID: Your Google Sheets spreadsheet ID (or select from list)  
    - Sheet Name: `Sheet1` (or select from list)  
    - Mapping mode: `Define below`  
    - Columns: Map all 19 columns as described in the Block 4 analysis above, using expressions referencing `$('Loop Over Items (lead)').item.json.<field>`  
    - **Important fix**: Set the ADDRESS column expression to `={{ $('Loop Over Items (lead)').item.json.address }}` (include the `$` prefix)  
    - Credential: Create a Google Sheets OAuth2 credential.  
    - Connect: "AI Company Description Generator" → "Sheet (Store google maps/lead data)"

18. **Add a Set node.**  
    - Type: `Set` (version 3.4)  
    - Name: "Get Website URLs"  
    - Assignment: Name = `Site internet`, Type = `string`, Value = `={{ $('Loop Over Items (lead)').first().json.website }}`  
    - Connect: "Sheet (Store google maps/lead data)" → "Get Website URLs"

19. **Add an HTTP Request node.**  
    - Type: `HTTP Request` (version 4.2)  
    - Name: "Https (Get Raw HTML Content from Business Website)"  
    - URL: `={{ $json['Site internet'] }}`  
    - On Error: `Continue Error Output`  
    - Connect: "Get Website URLs" → "Https (Get Raw HTML)"  
    - Connect the error output of this node → "Loop Over Items (lead)" (to continue with the next lead on failure)

20. **Add an OpenAI LangChain node.**  
    - Type: `OpenAI` (@n8n/n8n-nodes-langchain.openAi, version 1.8)  
    - Name: "Extract Business Email from Website HTML (GPT-4)"  
    - Model: `gpt-4o-mini` (select from list)  
    - JSON Output: disabled  
    - Single user message with extraction instructions and `{{ $json.data }}`  
    - Same OpenAI credential  
    - Connect: "Https (Get Raw HTML)" → success output → "Extract Business Email"

21. **Add a Google Sheets node.**  
    - Type: `Google Sheets` (version 4.7)  
    - Name: "Sheet (Update Email form website)"  
    - Operation: `appendOrUpdate`  
    - Matching columns: `TITLE`  
    - Same document ID and sheet name as step 17  
    - Columns mapped: `TITLE` = `{{ $('Loop Over Items (lead)').item.json.title }}`, `EMAIL` = `{{ $json.message.content }}`  
    - Same Google Sheets credential  
    - Connect: "Extract Business Email" → "Sheet (Update Email form website)"

22. **Add a Wait node.**  
    - Type: `Wait` (version 1.1)  
    - Name: "Wait 5s"  
    - Amount: `5` (seconds) — **set this explicitly**, as the default is 1 second  
    - Connect: "Sheet (Update Email form website)" → "Wait 5s"  
    - Connect: "Wait 5s" → "Loop Over Items (lead)" (continue to next lead)

23. **Add a Code node.**  
    - Type: `Code` (version 2)  
    - Name: "Code (Count Total Incoming Data)"  
    - JavaScript:  
      ```
      const total = $input.all().length;
      return [{ json: { total_execution: total } }];
      ```  
    - Connect: "Loop Over Items (lead)" → output 0 (done) → "Code (Count Total)"

24. **Add a Telegram node.**  
    - Type: `Telegram` (version 1.2)  
    - Name: "Notification message"  
    - Chat ID: your Telegram chat ID  
    - Text: `=Map Scrape Successfully\nDate: {{ $today.format('dd-MMM-yyyy') }}\nTotal Collected Lead: {{ $json.total_execution }}\n`  
    - Append Attribution: disabled  
    - Credential: Create a Telegram API credential with your Bot token.  
    - Connect: "Code (Count Total)" → "Notification message"

25. **Add a Microsoft Teams node.**  
    - Type: `Microsoft Teams` (version 2)  
    - Name: "Create message"  
    - Resource: `channelMessage`  
    - Team ID: your Team ID  
    - Channel ID: your Channel ID  
    - Message: same template as Telegram  
    - Credential: Create a Microsoft Teams OAuth2 credential.  
    - Connect: "Code (Count Total)" → "Create message"

26. **Add a Rapiwa node.**  
    - Type: `Rapiwa` (n8n-nodes-rapiwa.rapiwa, version 1)  
    - Name: "Rapiwa"  
    - Number: your WhatsApp number (with country code)  
    - Message: same template  
    - Credential: Create a Rapiwa API credential.  
    - Connect: "Code (Count Total)" → "Rapiwa"

27. **Add a Slack node.**  
    - Type: `Slack` (version 2.4)  
    - Name: "Send a message"  
    - Select: `user`  
    - User: your Slack username  
    - Authentication: OAuth2  
    - Text: same template  
    - Credential: Create a Slack OAuth2 credential with `chat:write` scope.  
    - Connect: "Code (Count Total)" → "Send a message"

28. **Activate the workflow** (optional — currently set to inactive).

**Credential checklist:**

| Credential | Service | Required For |
|-----------|---------|--------------|
| Apify API Token | Apify | Get list of actors, Get list of tasks |
| OpenAI API Key | OpenAI | AI Company Description Generator, Extract Business Email |
| Google Sheets OAuth2 | Google | Sheet (Store), Sheet (Update Email) |
| Telegram Bot API | Telegram | Notification message |
| Microsoft Teams OAuth2 | Microsoft Teams | Create message |
| Rapiwa API | Rapiwa | WhatsApp notification |
| Slack OAuth2 | Slack | Send a message |

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| **Apify Platform** — Required for running the Google Maps Scraper actor. An active account and API token are mandatory. | [https://apify.com/](https://apify.com/) |
| **Google Maps Scraper Actor** — The specific Apify actor used to scrape business data from Google Maps. Must be pre-configured as an Apify Task with search parameters. | [https://apify.com/compass/crawler-google-places](https://apify.com/compass/crawler-google-places) |
| **Telegram Bot API** — Used for sending completion notifications. Create a bot via BotFather and obtain the Bot Token and Chat ID. | [https://core.telegram.org/bots/api](https://core.telegram.org/bots/api) |
| **n8n Telegram Node Documentation** | [https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/) |
| **Bug in ADDRESS expression** — The Sheet (Store google maps/lead data) node uses `('Loop Over Items')` instead of `$('Loop Over Items')` for the ADDRESS column. This will cause a runtime expression error and must be fixed by adding the `$` prefix. | N/A |
| **Wait 5s default mismatch** — The "Wait 5s" node name implies a 5-second pause, but the `amount` parameter is not explicitly set, defaulting to 1 second. Set `amount: 5` explicitly to match the intended throttle. | N/A |
| **Apify dataset authentication** — The HTTP (Get Specific Task Result) node does not include an Authorization header. If the Apify dataset is not public, the request will fail with a 401. Add the same `Authorization: Bearer <token>` header used in the Run node. | N/A |
| **20-second wait assumption** — The workflow assumes the Apify actor finishes within 20 seconds. For large scrapes (many places, deep pagination), this may be insufficient. Consider replacing the Wait node with a polling loop or increasing the wait time. | N/A |
| **Token placeholder** — The HTTP (Run Scraper Task) node contains `YOUR_TOKEN_HERE` as a placeholder. Replace with your actual Apify API Bearer token before running. | N/A |
| **All notification placeholders** — Telegram Chat ID, Teams Team/Channel ID, Rapiwa phone number, and Slack username are all set to placeholder values. Replace each before first execution. | N/A |
| **Search customization** — The Apify task's search parameters (searchString, locationString, maxCrawlingDepth, etc.) are configured in the Apify platform, not in this workflow. Modify them directly in the Apify Task settings. | N/A |
| **PHONE field regex** — The inline JavaScript for the PHONE column strips non-digits and re-groups them as `(\d{3})(\d{4})(\d{4})`, but the replacement `$1$2$3` produces no separators (e.g., `12345678901`). Adjust the replacement pattern (e.g., `$1-$2-$3`) if formatting is desired. | N/A |
| **Email extraction fallback** — When GPT-4o-mini cannot find an email, it returns the literal string "Null". This string will be written to the Google Sheets EMAIL column as-is. Consider adding a post-processing step to convert "Null" to an actual empty value. | N/A |