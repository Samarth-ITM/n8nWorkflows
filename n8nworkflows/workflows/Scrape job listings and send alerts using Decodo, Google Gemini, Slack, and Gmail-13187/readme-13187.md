Scrape job listings and send alerts using Decodo, Google Gemini, Slack, and Gmail

https://n8nworkflows.xyz/workflows/scrape-job-listings-and-send-alerts-using-decodo--google-gemini--slack--and-gmail-13187


# Scrape job listings and send alerts using Decodo, Google Gemini, Slack, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow periodically monitors multiple company career pages, extracts job listings using Decodo + Google Gemini, filters them by a target department/category (e.g., “Engineering”), and sends alerts to **Slack** and **Gmail** when matching jobs are found.

**Typical use cases:**
- Job seekers tracking many company career pages
- Recruiters monitoring competitor hiring
- Teams wanting lightweight job-change notifications

### 1.1 Scheduling & Target Criteria Setup
Defines when the workflow runs and what department/category to match.

### 1.2 Source URL Retrieval (Airtable)
Fetches the list of career page URLs (and website names) to process.

### 1.3 Page Scraping + Rate Control
Scrapes the career page content via Decodo and (optionally) throttles requests to avoid rate limits.

### 1.4 Job Extraction (LLM)
Uses Google Gemini to extract job titles + application URLs from raw scraped content.

### 1.5 Normalization / Structuring
Cleans the extracted text and converts it into structured “job” items.

### 1.6 Department Filtering + Notifications
Uses a second Gemini-powered agent to filter by the target criteria and sends matching roles to Slack and Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Target Criteria Setup
**Overview:**  
Triggers the workflow on a schedule and defines the target department/category used later for filtering.

**Nodes involved:**  
- Run Daily (Schedule Trigger)  
- Job Name (Set)

#### Node: Run Daily
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point on a time schedule.
- **Configuration (interpreted):** Runs on an interval rule (configured in the UI; JSON shows a default interval placeholder).
- **Connections:**
  - **Output →** Job Name
- **Edge cases / failures:**
  - Misconfigured schedule rule (runs too frequently/rarely).
  - n8n instance time zone differences can cause unexpected trigger times.
- **Version notes:** typeVersion `1.2`

#### Node: Job Name
- **Type / role:** `n8n-nodes-base.set` — defines a constant variable for downstream prompts.
- **Configuration:**
  - Sets field `Position` (string) to **"Engineering"**.
- **Key variables used downstream:**
  - Referenced later as: `{{ $('Job Name').item.json.Position }}`
- **Connections:**
  - **Input ←** Run Daily
  - **Output →** Search records
- **Edge cases / failures:**
  - Renaming the node “Job Name” without updating expressions will break the filter prompt.
- **Version notes:** typeVersion `3.4`

**Sticky note coverage (applies to nodes in this block):**  
“Controls when the workflow runs and defines the target job category (e.g., Engineering). Change the schedule interval or target department without touching the rest of the workflow.”

---

### Block 2 — Source URL Retrieval (Airtable)
**Overview:**  
Pulls the list of company career page links from Airtable; only records with non-empty links are returned.

**Nodes involved:**  
- Search records (Airtable)

#### Node: Search records
- **Type / role:** `n8n-nodes-base.airtable` — queries Airtable for URLs to scrape.
- **Configuration:**
  - **Base:** “Career Pages Tracker”
  - **Table:** “Table 1”
  - **Operation:** Search
  - **Return all:** true
  - **Filter formula:** `NOT({Link} = '')` (ensures Link is not empty)
  - **Fields returned:** `Link`, `Website`
  - **Auth:** Airtable OAuth2 / Personal Access Token credential
- **Connections:**
  - **Input ←** Job Name
  - **Output →** Decodo
- **Edge cases / failures:**
  - Airtable auth expired / revoked.
  - Base/table IDs changed.
  - Records missing the expected `Link` field (schema mismatch).
  - `filterByFormula` syntax errors if field names differ.
- **Version notes:** typeVersion `2.1`

**Sticky note coverage:**  
“Fetches company career page URLs from Airtable… Only records with valid links are processed…”

---

### Block 3 — Page Scraping + Rate Control
**Overview:**  
Scrapes each career page via Decodo. A disabled batching node exists for iterative processing and waiting between scrapes, but it is currently disabled—this materially affects how the workflow behaves.

**Nodes involved:**  
- Decodo  
- Loop Over Items (Split in Batches) **(disabled)**  
- Wait 5 seconds

#### Node: Decodo
- **Type / role:** `@decodo/n8n-nodes-decodo.decodo` — performs scraping / retrieval from a URL.
- **Configuration:**
  - Parameters are empty in JSON (likely configured by defaults or via UI in a way not serialized here; verify in n8n).
  - Credentials are empty in JSON (template flag indicates creds setup completed, but no credential reference is embedded).
- **Connections:**
  - **Input ←** Search records
  - **Output →** Loop Over Items
- **Edge cases / failures:**
  - Missing/invalid Decodo credentials (if required).
  - Target sites blocking scraping, dynamic JS rendering issues, captchas.
  - Output shape differences (downstream expects `results[0].content` later).
- **Version notes:** typeVersion `1`

#### Node: Loop Over Items (disabled)
- **Type / role:** `n8n-nodes-base.splitInBatches` — intended to process items in controlled batches.
- **Status:** **Disabled**. When disabled, n8n typically does not execute it; this can break the intended flow (because downstream nodes may receive no input).
- **Configuration:** default options.
- **Connections (as designed):**
  - **Input ←** Decodo
  - **Output (batch loop) →** AI Agent
  - **Output (next batch) →** Wait 5 seconds
- **Edge cases / failures:**
  - With the node disabled, the workflow likely stops after Decodo (unless n8n bypass behavior is configured; generally it won’t).
  - If enabled, wrong batch size can trigger rate limits or slow processing.
- **Version notes:** typeVersion `3`

#### Node: Wait 5 seconds
- **Type / role:** `n8n-nodes-base.wait` — throttling/delay between batches/requests.
- **Configuration:**
  - Amount: `50` (despite the name “Wait 5 seconds”; in Wait node this amount is commonly seconds, but confirm the unit in UI—this mismatch is a maintenance hazard).
- **Connections:**
  - **Input ←** Loop Over Items
  - **Output →** Job extractor
- **Edge cases / failures:**
  - If the unit is seconds, 50s delay may be unintended.
  - If used heavily, it can increase runtime and queue pressure.
- **Version notes:** typeVersion `1.1`

**Sticky note coverage:**  
“Scrapes each career page and processes URLs in batches. The wait node prevents rate limits…”

---

### Block 4 — Job Extraction (LLM)
**Overview:**  
Uses a Gemini chat model wired into a LangChain Agent node to extract job titles and application URLs from scraped page content.

**Nodes involved:**  
- Google Gemini Chat Model  
- Job extractor (LangChain Agent)

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the LLM to the agent.
- **Configuration:**
  - Uses credential: “Google Gemini(PaLM) Api account”
  - Options: empty/default
- **Connections:**
  - **ai_languageModel →** Job extractor
- **Edge cases / failures:**
  - Invalid/expired API key, quota exceeded.
  - Model output variability (may include formatting that downstream code must clean).
- **Version notes:** typeVersion `1`

#### Node: Job extractor
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent to extract jobs from raw content.
- **Configuration:**
  - Prompt (key portion):  
    - “Extract all job titles and their application URLs… Output ONLY a raw list… If no jobs are found, return "No jobs".”
  - Feeds the scraped content with: `{{ $json.results[0].content }}`
  - Prompt type: “define”
- **Inputs/Outputs:**
  - **Input ←** Wait 5 seconds (so it expects the Wait output to contain `results[0].content` from Decodo’s scrape result; effectively this assumes Wait is just passing through Decodo output)
  - **Output →** Code in JavaScript
- **Edge cases / failures:**
  - If Decodo output does not contain `results[0].content`, expression resolves to `undefined` and extraction quality collapses.
  - If the page is heavy HTML, token limits may truncate.
  - “No jobs” exact phrase must be handled downstream (currently code checks “No open positions”, not “No jobs”).
- **Version notes:** typeVersion `2.2`

**Sticky note coverage:**  
“Uses Google Gemini to extract clean job titles and application links from raw page content…”

---

### Block 5 — Normalization / Structuring
**Overview:**  
Converts the agent’s raw textual output into individual items, stripping numbering, bullets, “Job Title:” prefixes, and skipping empty/no-position results.

**Nodes involved:**  
- Code in JavaScript

#### Node: Code in JavaScript
- **Type / role:** `n8n-nodes-base.code` — transforms LLM output into structured items.
- **Configuration (logic):**
  - Iterates over all incoming items: `$input.all()`
  - Reads job text from (in priority order):  
    1) `item.json.output?.Job`  
    2) `item.json.Job`  
    3) `item.json.output`  
    else empty string
  - Skips items if:
    - Empty / not a string
    - Contains `"No open positions"`
  - Splits by newline and cleans each line:
    - Removes `Job Title:` prefix (case-insensitive)
    - Removes leading numbering (`1. `) or bullet (`- `)
  - Outputs new items shaped as: `{ json: { job: cleanJob } }`
- **Connections:**
  - **Input ←** Job extractor
  - **Output →** AI Agent
- **Edge cases / failures:**
  - Upstream agent output structure mismatch: LangChain Agent nodes in n8n often store text in `json.output` (string). This code also looks for `output.Job`, which may not exist unless the agent outputs JSON.
  - Does **not** skip `"No jobs"` even though Job extractor may return that; this may create a bogus job item `"No jobs"`.
  - If Gemini returns multi-line entries where title and URL are on separate lines, splitting by newline may separate them incorrectly.
- **Version notes:** typeVersion `2`

**Sticky note coverage:**  
“Cleans AI output by removing formatting, empty results, and ‘No open positions.’ Converts jobs into structured items…”

---

### Block 6 — Department Filtering + Notifications
**Overview:**  
A second Gemini-powered agent filters the cleaned jobs against the target department. The result is sent to Slack and Gmail.

**Nodes involved:**  
- Google Gemini Chat Model1  
- AI Agent  
- Send a message1 (Slack)  
- Send a message (Gmail)

#### Node: Google Gemini Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — LLM for the filtering agent.
- **Configuration:**
  - Credential: “Google Gemini(PaLM) Api account”
  - Options: default
- **Connections:**
  - **ai_languageModel →** AI Agent
- **Edge cases / failures:** same as other Gemini node (quota/auth/latency).
- **Version notes:** typeVersion `1`

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — filters job items to only those matching the target criteria.
- **Configuration (prompt highlights):**
  - Target criteria: `{{ $('Job Name').item.json.Position }}`
  - Jobs list: `{{ $json.job }}` (one job per item from Code node)
  - Instructions: return ONLY matching jobs; if none: exactly `"No matching positions found"`.
  - `hasOutputParser: true` (output parsing enabled; parser details not shown—assume default).
- **Connections:**
  - **Input ←** Code in JavaScript
  - **Output →** Send a message1 (Slack) AND Send a message (Gmail) (fan-out)
- **Edge cases / failures:**
  - Because items are per-job, the agent is effectively filtering **one job at a time**, not a “list”. It may return the job text or “No matching positions found” for each item—this can lead to multiple Slack/email messages and many “no match” alerts unless filtered out.
  - Expression `$('Job Name').item...` depends on cross-node item matching; if multiple Airtable records run, item-linking can be ambiguous. Consider using a single merged context or setting Position globally.
- **Version notes:** typeVersion `2.2`

#### Node: Send a message1 (Slack)
- **Type / role:** `n8n-nodes-base.slack` — sends Slack DM/message to a selected user.
- **Configuration:**
  - Auth: OAuth2
  - Recipient selection: user (ID `U09BDCAHRBN`, cached name “hafsa.karim”)
  - Text:  
    ```
    Here is the new listed job:
    {{ $json.output }}
    ```
- **Connections:**
  - **Input ←** AI Agent
  - **No outputs** (terminal)
- **Edge cases / failures:**
  - Slack OAuth token revoked / missing scopes (chat:write, users:read).
  - If `$json.output` is “No matching positions found”, it will still notify unless you add a conditional node.
- **Version notes:** typeVersion `2.3`

#### Node: Send a message (Gmail)
- **Type / role:** `n8n-nodes-base.gmail` — sends an email alert.
- **Configuration:**
  - To: `user@example.com`
  - Subject: `New job listed`
  - Body: `{{ $json.output }}`
  - Append attribution disabled
- **Connections:**
  - **Input ←** AI Agent
  - **No outputs** (terminal)
- **Edge cases / failures:**
  - Gmail OAuth expired; Google may require re-consent.
  - Same “No matching positions found” spam risk as Slack.
- **Version notes:** typeVersion `2.1`

**Sticky note coverage:**  
“Filters jobs based on the target department and sends alerts. Only relevant roles trigger Slack and email notifications.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Daily | Schedule Trigger | Entry point on schedule | — | Job Name | Controls when the workflow runs and defines the target job category (e.g., Engineering). Change the schedule interval or target department without touching the rest of the workflow. |
| Job Name | Set | Defines target department/category (`Position`) | Run Daily | Search records | Controls when the workflow runs and defines the target job category (e.g., Engineering). Change the schedule interval or target department without touching the rest of the workflow. |
| Search records | Airtable | Fetch career page URLs from Airtable | Job Name | Decodo | Fetches company career page URLs from Airtable. Only records with valid links are processed, making it easy to manage multiple companies in one place. |
| Decodo | Decodo | Scrape career pages | Search records | Loop Over Items | Scrapes each career page and processes URLs in batches. The wait node prevents rate limits and ensures stable scraping across multiple websites. |
| Loop Over Items | Split in Batches | Batch/loop control for multiple URLs (**disabled**) | Decodo | AI Agent; Wait 5 seconds | Scrapes each career page and processes URLs in batches. The wait node prevents rate limits and ensures stable scraping across multiple websites. |
| Wait 5 seconds | Wait | Rate limiting / throttling | Loop Over Items | Job extractor | Scrapes each career page and processes URLs in batches. The wait node prevents rate limits and ensures stable scraping across multiple websites. |
| Google Gemini Chat Model | Gemini Chat Model | LLM for extraction agent | — (AI link) | Job extractor (AI) | Uses Google Gemini to extract clean job titles and application links from raw page content while ignoring navigation and non-job text. |
| Job extractor | LangChain Agent | Extract job titles + URLs from scraped content | Wait 5 seconds | Code in JavaScript | Uses Google Gemini to extract clean job titles and application links from raw page content while ignoring navigation and non-job text. |
| Code in JavaScript | Code | Clean/normalize extracted jobs into items | Job extractor | AI Agent | Cleans AI output by removing formatting, empty results, and “No open positions.” Converts jobs into structured items for reliable filtering. |
| Google Gemini Chat Model1 | Gemini Chat Model | LLM for filtering agent | — (AI link) | AI Agent (AI) | Filters jobs based on the target department and sends alerts. Only relevant roles trigger Slack and email notifications. |
| AI Agent | LangChain Agent | Filter jobs by target criteria | Code in JavaScript | Send a message1; Send a message | Filters jobs based on the target department and sends alerts. Only relevant roles trigger Slack and email notifications. |
| Send a message1 | Slack | Send Slack alert | AI Agent | — | Filters jobs based on the target department and sends alerts. Only relevant roles trigger Slack and email notifications. |
| Send a message | Gmail | Send email alert | AI Agent | — | Filters jobs based on the target department and sends alerts. Only relevant roles trigger Slack and email notifications. |
| Sticky Note | Sticky Note | Comment | — | — | (Node comment itself) ## Nodes: Schedule Trigger, Edit Fields… |
| Sticky Note1 | Sticky Note | Comment | — | — | (Node comment itself) ## Nodes: Airtable Search Records… ## Nodes: Decodo, Loop Over Items, Wait… |
| Sticky Note2 | Sticky Note | Comment | — | — | (Node comment itself) ### Title: Automated Job Scraping & Alerts with Decodo and Google Gemini… |
| Sticky Note3 | Sticky Note | Comment | — | — | (Node comment itself) ## Nodes: Code in JavaScript… |
| Sticky Note4 | Sticky Note | Comment | — | — | (Node comment itself) ## Nodes: AI Agent 1, Google Gemini Chat Model… |
| Sticky Note5 | Sticky Note | Comment | — | — | (Node comment itself) ## Nodes: AI Agent, Google Gemini Chat Model 1, Slack, Gmail… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Run Daily”**
   - Node: **Schedule Trigger**
   - Set your desired schedule (daily or interval).
   - Connect **Run Daily → Job Name**.

2) **Create “Job Name”**
   - Node: **Set**
   - Add a string field:
     - `Position = "Engineering"` (or your target department, e.g., “Data”, “Marketing”).
   - Connect **Job Name → Search records**.

3) **Create “Search records” (Airtable)**
   - Node: **Airtable**
   - Authentication: **Airtable OAuth2 / Personal Access Token**
   - Select:
     - Base: *Career Pages Tracker* (or your base)
     - Table: *Table 1* (or your table)
   - Operation: **Search**
   - Fields to return: `Link`, `Website`
   - Filter formula: `NOT({Link} = '')`
   - Return All: **true**
   - Connect **Search records → Decodo**.

4) **Create “Decodo”**
   - Node: **Decodo**
   - Configure it to fetch/scrape the URL from Airtable records (typically map Airtable’s `Link` field to the Decodo target URL in the node UI).
   - Add required **Decodo credentials** (API key/account) if prompted by the node.
   - Connect **Decodo → Loop Over Items**.

5) **Create “Loop Over Items”** (important)
   - Node: **Split In Batches**
   - Choose a batch size (e.g., 1–5) depending on rate limits.
   - Ensure it is **enabled** (in the provided workflow it is disabled; enabling is required for the designed looping + wait behavior).
   - Connect:
     - **Loop Over Items (main output 0) → Wait 5 seconds**
     - **Loop Over Items (main output 1 / “continue”) →** (back into processing path as needed)
   - In this workflow JSON, output 0 goes to **AI Agent** and output 1 goes to **Wait**; if you keep that design, ensure the data you want reaches Job extractor. Commonly you would do:
     - Loop Over Items → Wait → Job extractor → Code → Filter → Notify
     - Then connect the end of the chain back to **Loop Over Items** “Next batch” input (standard batching pattern).

6) **Create “Wait 5 seconds”**
   - Node: **Wait**
   - Set delay (confirm unit in UI; set to 5 seconds if that’s intended).
   - Connect **Wait 5 seconds → Job extractor**.

7) **Create “Google Gemini Chat Model”**
   - Node: **Google Gemini Chat Model**
   - Credential: **Google Gemini(PaLM) API**
   - Connect its **AI Language Model** output to **Job extractor** (AI port).

8) **Create “Job extractor”**
   - Node: **AI Agent (LangChain)**
   - Prompt type: **Define**
   - Prompt text (adapt as needed) that:
     - extracts job titles and application URLs
     - returns “No jobs” when none found
     - uses the scraped content variable (map the Decodo content field, e.g. `{{$json.results[0].content}}`)
   - Connect **Job extractor → Code in JavaScript**.

9) **Create “Code in JavaScript”**
   - Node: **Code**
   - Paste transformation logic to:
     - read the agent output string
     - split lines
     - clean prefixes/numbering
     - output items as `{ job: "..." }`
   - Connect **Code in JavaScript → AI Agent** (filtering agent).

10) **Create “Google Gemini Chat Model1”**
   - Node: **Google Gemini Chat Model**
   - Credential: same (or separate) Gemini credential
   - Connect **AI Language Model → AI Agent** (AI port).

11) **Create “AI Agent” (filtering)**
   - Node: **AI Agent (LangChain)**
   - Prompt should:
     - read `Target Criteria` from `$('Job Name').item.json.Position`
     - read the job text from `$json.job`
     - output only matching jobs or exactly `No matching positions found`
   - Connect **AI Agent → Slack** and **AI Agent → Gmail**.

12) **Create “Send a message1” (Slack)**
   - Node: **Slack**
   - Authentication: **OAuth2**
   - Select target (user/channel).
   - Message text uses `{{$json.output}}`.

13) **Create “Send a message” (Gmail)**
   - Node: **Gmail**
   - Authentication: **Gmail OAuth2**
   - To/Subject/Body (body uses `{{$json.output}}`)
   - Optional: disable attribution if desired.

**Recommended extra step (to prevent spam):**
- Insert an **IF** node between “AI Agent” and notifications to only send when `{{$json.output}}` is not equal to `"No matching positions found"` (and also not `"No jobs"` if you keep that phrase).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Automated Job Scraping & Alerts with Decodo and Google Gemini… Add career page URLs to Airtable… Set the target job category… Connect Decodo, Google Gemini, Slack, and Gmail credentials… Adjust the schedule… Activate the workflow.” | Workflow description (Sticky Note content) |
| **Important operational note:** “Loop Over Items” is **disabled** in the provided workflow, which likely prevents the intended batching/wait/extraction flow from running end-to-end. | Execution correctness / maintenance |
| **Data-shape dependency:** Job extractor prompt expects `{{$json.results[0].content}}` to exist (Decodo output must match). | Integration constraint |
| **Text mismatch:** extractor returns `"No jobs"` but the cleaning code only checks `"No open positions"`; consider harmonizing. | Reliability / edge case prevention |