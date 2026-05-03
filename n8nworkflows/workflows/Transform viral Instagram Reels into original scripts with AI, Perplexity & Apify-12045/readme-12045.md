Transform viral Instagram Reels into original scripts with AI, Perplexity & Apify

https://n8nworkflows.xyz/workflows/transform-viral-instagram-reels-into-original-scripts-with-ai--perplexity---apify-12045


# Transform viral Instagram Reels into original scripts with AI, Perplexity & Apify

## 1. Workflow Overview

**Purpose:** This workflow periodically scrapes viral Instagram Reels from a set of target accounts (via Apify), stores new Reel metadata in Google Sheets, downloads each Reel video, transcribes its audio with OpenAI, extracts/filters tool- or tech-related topics and generates structured content ideas, enriches the topic with Perplexity research, then generates an original ~100-word YouTube-style script and writes it back to Google Sheets.

**Target use cases:**
- Content creators repurposing viral Reels into original scripts
- Agencies/social media managers scaling idea generation and scripting
- Trend monitoring + lightweight research + script drafting pipeline

### Logical blocks
**1.1 Scheduled ingestion (Trigger → Scrape → Limit)**  
Runs daily at a specific time, scrapes Reels, and limits batch size.

**1.2 Deduplication + storage (Check sheet → Remove duplicates → Append)**  
Looks up scraped Reel IDs in Google Sheets, keeps only new entries, and appends them.

**1.3 Media processing (Download → Transcribe)**  
Downloads the Reel video file and transcribes audio to text.

**1.4 Topic filtering + idea structuring (LLM JSON extraction)**  
Determines whether the transcript is about tools/tech/AI; extracts tool list, step-by-step guide, and an internet search prompt (JSON).

**1.5 Research enrichment (Perplexity)**  
Uses the extracted search prompt to fetch “four interesting peculiar things” about the tool/topic.

**1.6 Final script generation + persistence (LLM → Update sheet)**  
Combines transcript, idea JSON, and Perplexity output to generate a short script (JSON), then updates the corresponding row in Google Sheets with transcript + final script.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled ingestion (Trigger → Scrape → Limit)
**Overview:** Starts on a schedule, calls an Apify endpoint to scrape Reels for multiple usernames, then restricts the number processed per run for safe testing/throughput control.

**Nodes involved:**  
- **Schedule Trigger**  
- ** Run Apify Scraper**  
- **Limit**

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point on a time schedule.
- **Configuration (interpreted):** Runs every day at **08:00** (server/workspace timezone).
- **Inputs/outputs:** No input; outputs one trigger event to ** Run Apify Scraper**.
- **Edge cases / failures:**
  - Timezone mismatch (n8n instance timezone vs expected).
  - Missed triggers if instance is down at trigger time.
- **Version notes:** typeVersion `1.2` (schedule UI differs slightly across n8n versions; semantics unchanged).

#### Node:  Run Apify Scraper
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Apify API (or an Apify Actor run endpoint) to scrape Instagram Reels.
- **Configuration (interpreted):**
  - **POST** to `https://api.apify.com/` (placeholder base URL in this export).
  - Sends JSON body containing:
    - `resultsLimit: 5`
    - `username: [ ... list of Instagram handles ... ]`
  - Headers:
    - `Accept: application/json`
    - `Authorization: Bearer YOUR_TOKEN_HERE` (must be replaced with a real Apify token)
  - Redirects enabled (maxRedirects 35).
- **Inputs/outputs:** Input from trigger; output to **Limit**. Output is expected to be a list/array of scraped reel objects (each with fields like `id`, `url`, `videoUrl`, etc.).
- **Edge cases / failures:**
  - Wrong endpoint: Apify usually requires a specific Actor run endpoint (e.g., `/v2/acts/{actorId}/runs?token=...`) or dataset endpoint; using only `https://api.apify.com/` will likely fail unless completed elsewhere.
  - Auth failure if token is invalid/expired.
  - Scraper can return empty results, throttling, or structure changes.
  - Large payloads/timeouts depending on actor runtime.
- **Version notes:** typeVersion `4.2` (HTTP Request node).

#### Node: Limit
- **Type / role:** `n8n-nodes-base.limit` — limits number of items passed downstream.
- **Configuration (interpreted):**
  - Keeps **last 5 items** (`maxItems: 5`, `keep: lastItems`).
- **Inputs/outputs:**
  - Input from ** Run Apify Scraper**.
  - Two outgoing connections (important):
    - Output **index 0 → Check Existing Entries**
    - Output **index 1 → Remove Duplicate Reels**
- **Important behavior note:** The Limit node itself does not “fork” items into two logical streams; in n8n, multiple outgoing connections typically replicate the same items to both downstream nodes. Here, it’s using **different output indices** which is unusual for `Limit` (it generally has one main output). This may be an artifact of the export or UI wiring; validate in your n8n instance that both downstream nodes receive the intended data.
- **Edge cases / failures:**
  - If upstream output is not itemized (e.g., one JSON object containing an array), Limit may not behave as expected; you may need an **Item Lists** or **Split Out** step.
- **Version notes:** typeVersion `1`.

---

### 2.2 Deduplication + storage (Check sheet → Remove duplicates → Append)
**Overview:** Prevents reprocessing the same Reels by checking each scraped Reel `id` against a Google Sheet, keeping only non-matching IDs, then appending those new reels to the sheet.

**Nodes involved:**  
- ** Check Existing Entries**  
- ** Remove Duplicate Reels**  
- ** Add New Reels to Sheet**

#### Node:  Check Existing Entries
- **Type / role:** `n8n-nodes-base.googleSheets` — lookup/filter rows by Reel ID.
- **Configuration (interpreted):**
  - Uses **Google Sheets OAuth2** credential.
  - Targets document: **“Instagram Reels”**
  - Sheet/tab: **“Reels”** (gid=0)
  - **Filter:** lookup `id` column equals `={{ $json.id }}` (the incoming scraped reel item’s `id`).
- **Inputs/outputs:** Input from **Limit**. Output to ** Remove Duplicate Reels**.
- **Edge cases / failures:**
  - Column header mismatch: sheet must contain an `id` column exactly.
  - Empty/undefined `$json.id` will cause lookup to fail or match nothing (leading to duplicates being appended).
  - Google auth errors, permission issues, rate limits.
- **Version notes:** typeVersion `4.5`.

#### Node:  Remove Duplicate Reels
- **Type / role:** `n8n-nodes-base.merge` — joins scraped items with lookup results to keep only non-matching IDs.
- **Configuration (interpreted):**
  - **Mode:** combine
  - **Join mode:** `keepNonMatches`
  - **Match field:** `id`
- **Inputs/outputs:**
  - **Input 0:** from ** Check Existing Entries** (the sheet lookup results)
  - **Input 1:** from **Limit** (the original scraped reels)
  - Output to ** Add New Reels to Sheet**
- **Expected logic:** Keep items from the scraped set that did **not** match existing sheet entries by `id`.
- **Edge cases / failures:**
  - If either input stream has different item shapes (e.g., the lookup returns rows with different `id` formatting), matching can fail.
  - If “Check Existing Entries” returns no items at all (node outputs nothing), merge behavior depends on node settings; confirm it still passes scraped items correctly.
- **Version notes:** typeVersion `3`.

#### Node:  Add New Reels to Sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — append new Reel metadata as rows.
- **Configuration (interpreted):**
  - Operation: **append**
  - Document: **“Instagram Reels”**, Sheet: **“Reels”**
  - Maps many columns from each scraped reel item, including:
    - `id`, `url`, `caption`, `hashtags`, `username` (from `ownerUsername`), `videoUrl`, `shortCode`, `timestamp`, `displayUrl`, `likesCount`, `firstComment`, `commentsCount`, `videoDuration`, `videoPlayCount`, `videoViewCount`
  - Matching columns configured as `id` (sheet schema includes other removed columns used later: `scrapedTranscript`, `newTranscript`).
- **Inputs/outputs:** Input from ** Remove Duplicate Reels**; output to ** Download Reel Video**.
- **Edge cases / failures:**
  - If incoming items lack fields (e.g., `ownerUsername` missing), the cell may be blank.
  - Google Sheets API limits and throttling on large runs.
  - Data types: counts/durations are written as strings per schema; downstream expects URLs to be valid.
- **Version notes:** typeVersion `4.5`.

---

### 2.3 Media processing (Download → Transcribe)
**Overview:** Downloads each Reel’s video file from the scraped `videoUrl` and transcribes the audio into text using OpenAI’s audio transcription capability.

**Nodes involved:**  
- ** Download Reel Video**  
- ** Transcribe Reel Audio**

#### Node:  Download Reel Video
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches the binary video file.
- **Configuration (interpreted):**
  - URL: `={{ $json.videoUrl }}`
  - Method defaults to GET.
- **Inputs/outputs:** Input from ** Add New Reels to Sheet**; output to ** Transcribe Reel Audio**.
- **Edge cases / failures:**
  - `videoUrl` can be short-lived or geo/permission restricted.
  - Response might be HTML (blocked) instead of video binary.
  - Ensure node is configured to **download as binary** in your n8n instance if the transcription node expects binary input; the exported JSON doesn’t show explicit “download” toggles.
- **Version notes:** typeVersion `4.2`.

#### Node:  Transcribe Reel Audio
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI audio transcription (Whisper-like).
- **Configuration (interpreted):**
  - Resource: **audio**
  - Operation: **transcribe**
  - Uses **OpenAI API** credential.
- **Inputs/outputs:** Expects binary audio/video input from ** Download Reel Video**; outputs a JSON including transcribed text (used later as `$json.text`).
- **Edge cases / failures:**
  - If previous node didn’t provide binary data, transcription will fail.
  - File too large/unsupported format.
  - OpenAI quota/rate limits; timeouts on long videos.
- **Version notes:** typeVersion `1.6` (LangChain OpenAI node; requires compatible n8n + node package versions).

---

### 2.4 Topic filtering + idea structuring (LLM JSON extraction)
**Overview:** Uses an LLM prompt to decide if the transcript is about a tool/technology/AI. If yes, it extracts tool names, step-by-step instructions, a content suggestion, and a `searchPrompt`—all as strict JSON.

**Nodes involved:**  
- ** Filter + Generate Script Ideas**

#### Node:  Filter + Generate Script Ideas
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — chat completion with JSON output enforcement.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
  - `jsonOutput: true` (expects valid JSON)
  - Messages:
    - System: “helpful, intelligent admin assistant”
    - User: long instruction block requiring output JSON with fields:
      - `verdict` ("true or false")
      - `tools` (list)
      - `stepByStep` (string)
      - `suggestion` (string)
      - `searchPrompt` (string)
      - If verdict false → other fields empty
    - Additional user message passes transcript as JSON: `{"transcript":"{{ $json.text }}"}` where `$json.text` comes from transcription.
- **Inputs/outputs:** Input from ** Transcribe Reel Audio**; output to ** Research with Perplexity**. Output JSON is available under `message.content` (as used later).
- **Edge cases / failures:**
  - Model may output non-JSON or invalid JSON → node fails when `jsonOutput` is enforced.
  - Transcript may be empty/low-quality → verdict false; downstream Perplexity still runs unless you add an IF node.
  - Prompt mismatch: verdict is required as a string `"true or false"`; sometimes models output boolean `true/false`.
- **Version notes:** typeVersion `1.6`.

---

### 2.5 Research enrichment (Perplexity)
**Overview:** Calls Perplexity’s Chat Completions API to fetch “four interesting peculiar things” about the detected tool/topic using the earlier `searchPrompt`.

**Nodes involved:**  
- ** Research with Perplexity**

#### Node:  Research with Perplexity
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Perplexity API.
- **Configuration (interpreted):**
  - POST `https://api.perplexity.ai/chat/completions`
  - Body:
    - model: `sonar-pro`
    - system: “Be precise and concise.”
    - user: `Tell me four interesting (peculiar) things about {{ $json.message.content.searchPrompt }}`
  - Headers:
    - `accept: application/json`
    - `Authorization: Bearer YOUR_TOKEN_HERE` (replace with Perplexity API key)
- **Inputs/outputs:** Input from ** Filter + Generate Script Ideas**; output to ** Generate Final Script**. Response used later as `$json.choices[0].message.content`.
- **Edge cases / failures:**
  - If `searchPrompt` is empty (verdict false), the research prompt becomes meaningless.
  - API auth/quota failures; rate limiting.
  - Response structure changes; if `choices[0]...` missing, downstream expression fails.
- **Version notes:** typeVersion `4.2`.

---

### 2.6 Final script generation + persistence (LLM → Update sheet)
**Overview:** Generates the final short script (JSON) using OpenAI, combining transcript, tool names, step-by-step, suggestions, and Perplexity output, then updates the corresponding Google Sheet row (matched by `id`) with both the raw transcript and final script.

**Nodes involved:**  
- ** Generate Final Script**  
- ** Update Sheet with Script**

#### Node:  Generate Final Script
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — creates final script as JSON.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
  - Temperature: **0.9** (more creative variation)
  - `jsonOutput: true`
  - Messages:
    - System: writing assistant
    - User: instructions to write a ~100-word script with a CTA at end.
    - A prefilled “assistant” example JSON script (acts as an in-context example).
    - Final user message provides a JSON payload composed via expressions:
      - `toolNames`: from **Filter + Generate Script Ideas** → `tools.join()`
      - `roughDraftScript`: from **Transcribe Reel Audio** → first item `.json.text`
      - `perplexityOutput`: from Perplexity response → `$json.choices[0].message.content`
      - `stepByStepGuide`: from ideas node → `stepByStep`
      - `suggestionsForImprovement`: from ideas node → `suggestion`
- **Inputs/outputs:** Input from ** Research with Perplexity**; output to ** Update Sheet with Script** (script located at `$json.message.content.script`).
- **Edge cases / failures:**
  - Any referenced node path missing causes expression failures (e.g., `.choices[0]...`).
  - If `tools` is empty, `join()` returns empty string; still okay but may reduce script quality.
  - JSON enforcement failure if model outputs extra text.
- **Version notes:** typeVersion `1.6`.

#### Node:  Update Sheet with Script
- **Type / role:** `n8n-nodes-base.googleSheets` — updates an existing row by matching `id`.
- **Configuration (interpreted):**
  - Operation: **update**
  - Document: **“Instagram Reels”**, Sheet: **“Reels”**
  - Matching column: `id`
  - Writes:
    - `id`: `={{ $(' Add New Reels to Sheet').item.json.id }}` (the id from the append step)
    - `newTranscript`: `={{ $json.message.content.script }}` (final script)
    - `scrapedTranscript`: `={{ $(' Transcribe Reel Audio').item.json.text }}` (raw transcript)
- **Inputs/outputs:** Input from ** Generate Final Script**; end of workflow.
- **Edge cases / failures:**
  - If append step didn’t run for a given item (e.g., was duplicate), referencing `Add New Reels to Sheet` item may be invalid.
  - Matching by `id` requires the row to exist; if not found, update may fail or do nothing depending on node behavior.
  - Concurrency: parallel runs could update wrong rows if item linking is misaligned.
- **Version notes:** typeVersion `4.5`.

---

### 2.7 Documentation / annotation block (Sticky Note)
**Overview:** Provides human guidance on intent, setup requirements, and a LinkedIn link.

**Nodes involved:**  
- **Sticky Note**

#### Node: Sticky Note
- **Type / role:** `n8n-nodes-base.stickyNote` — comment/annotation.
- **Content highlights:** Explains “InstaReel Script Builder”, setup requirements (Apify, Google Sheet, Perplexity API, OpenAI), and points to the **Limit** node for controlling batch size. Includes link:  
  https://www.linkedin.com/in/nealjmcleod/
- **Execution:** Not connected; no runtime effect.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Scheduled entry point (daily run) | — |  Run Apify Scraper | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Run Apify Scraper | HTTP Request | Invoke Apify scraper to fetch Reels | Schedule Trigger | Limit | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
| Limit | Limit | Restrict number of reels processed per run |  Run Apify Scraper |  Check Existing Entries;  Remove Duplicate Reels | # 🎥 InstaReel Script Builder… (Tip: adjust Limit) https://www.linkedin.com/in/nealjmcleod/ |
|  Check Existing Entries | Google Sheets | Lookup existing Reel IDs in sheet | Limit |  Remove Duplicate Reels | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Remove Duplicate Reels | Merge | Keep only non-duplicate reels by `id` |  Check Existing Entries; Limit |  Add New Reels to Sheet | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Add New Reels to Sheet | Google Sheets | Append new reels metadata to sheet |  Remove Duplicate Reels |  Download Reel Video | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Download Reel Video | HTTP Request | Download reel video from `videoUrl` |  Add New Reels to Sheet |  Transcribe Reel Audio | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Transcribe Reel Audio | OpenAI (LangChain) | Transcribe audio to text |  Download Reel Video |  Filter + Generate Script Ideas | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Filter + Generate Script Ideas | OpenAI (LangChain) | Classify/extract tools + steps + search prompt (JSON) |  Transcribe Reel Audio |  Research with Perplexity | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Research with Perplexity | HTTP Request | Enrich topic with Perplexity research |  Filter + Generate Script Ideas |  Generate Final Script | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Generate Final Script | OpenAI (LangChain) | Generate final ~100-word script (JSON) |  Research with Perplexity |  Update Sheet with Script | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
|  Update Sheet with Script | Google Sheets | Update row with transcript + generated script |  Generate Final Script | — | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |
| Sticky Note | Sticky Note | Documentation/annotation | — | — | # 🎥 InstaReel Script Builder… https://www.linkedin.com/in/nealjmcleod/ |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set its name to:  
   *Transform viral Instagram Reels into original scripts with AI, Perplexity & Apify*

2. **Add “Schedule Trigger”** (`Schedule Trigger` node)  
   - Set schedule to **Every day at 08:00** (adjust timezone as needed).  
   - Connect to the next node.

3. **Add “Run Apify Scraper”** (`HTTP Request` node)  
   - Method: **POST**  
   - URL: **Your Apify Actor run endpoint** (recommended), not just `https://api.apify.com/`. Common patterns:
     - `https://api.apify.com/v2/acts/<user>~<actor>/runs?waitForFinish=60`
     - or a task endpoint if you use Apify Tasks.
   - Headers:
     - `Accept: application/json`
     - `Authorization: Bearer <APIFY_TOKEN>`
   - Body: JSON with at least:
     - `resultsLimit` (e.g., 5)
     - `username` array with target Instagram accounts
   - Ensure the response is a list of items or convert it later (depending on the actor response).

4. **Add “Limit”** (`Limit` node)  
   - Max items: **5** (or desired batch size)  
   - Keep: **last items** (or “first” depending on preference).  
   - Connect **Run Apify Scraper → Limit**.

5. **Add “Check Existing Entries”** (`Google Sheets` node)  
   - Credentials: **Google Sheets OAuth2** (select/create).  
   - Document: create/select a spreadsheet named **Instagram Reels**.  
   - Sheet/tab: **Reels** with headers including at least:
     - `id`, `videoUrl`, `scrapedTranscript`, `newTranscript` (plus any metadata columns you want)
   - Operation: use a **lookup/filter** that checks:
     - Column: `id`
     - Lookup value: `{{$json.id}}`
   - Connect **Limit → Check Existing Entries**.

6. **Add “Remove Duplicate Reels”** (`Merge` node)  
   - Mode: **Combine**
   - Join mode: **Keep Non-Matches**
   - Field to match: **id**
   - Connect:
     - **Check Existing Entries → Merge Input 1**
     - **Limit → Merge Input 2** (original scraped items)
   - This should output only reels not already in the sheet.

7. **Add “Add New Reels to Sheet”** (`Google Sheets` node)  
   - Credentials: same Google credential.
   - Operation: **Append**
   - Map fields from the scraped item to your sheet columns:
     - `id: {{$json.id}}`
     - `videoUrl: {{$json.videoUrl}}`
     - `url`, `caption`, `hashtags`, `ownerUsername`, etc. as desired
   - Connect **Remove Duplicate Reels → Add New Reels to Sheet**.

8. **Add “Download Reel Video”** (`HTTP Request` node)  
   - Method: **GET**
   - URL: `{{$json.videoUrl}}`
   - Configure response to **download binary** (in node options: “Response Format: File” / “Download” depending on your n8n version).
   - Connect **Add New Reels to Sheet → Download Reel Video**.

9. **Add “Transcribe Reel Audio”** (`OpenAI` LangChain node: `@n8n/n8n-nodes-langchain.openAi`)  
   - Credentials: **OpenAI API** (create/select).
   - Resource: **Audio**
   - Operation: **Transcribe**
   - Ensure it reads from the binary property produced by the previous node (often `data` unless renamed).
   - Connect **Download Reel Video → Transcribe Reel Audio**.

10. **Add “Filter + Generate Script Ideas”** (OpenAI LangChain chat)  
   - Model: **gpt-4o**
   - Enable **JSON output**.
   - Prompt: instruct the model to output strictly:
     - verdict, tools, stepByStep, suggestion, searchPrompt
   - Pass transcript from transcription node: `{{$json.text}}`
   - Connect **Transcribe Reel Audio → Filter + Generate Script Ideas**.

11. **Add “Research with Perplexity”** (`HTTP Request` node)  
   - Method: **POST**
   - URL: `https://api.perplexity.ai/chat/completions`
   - Headers:
     - `accept: application/json`
     - `Authorization: Bearer <PERPLEXITY_API_KEY>`
   - Body JSON:
     - model: `sonar-pro`
     - messages including user content referencing `{{$json.message.content.searchPrompt}}`
   - Connect **Filter + Generate Script Ideas → Research with Perplexity**.

12. **Add “Generate Final Script”** (OpenAI LangChain chat)  
   - Model: **gpt-4o**
   - Temperature: **0.9**
   - Enable **JSON output** with format: `{"script":"..."}` (~100 words + CTA).
   - Provide inputs via expressions:
     - Tool names: from ideas node `tools`
     - Rough draft: from transcription node `text`
     - Perplexity content: from Perplexity response `choices[0].message.content`
     - Step-by-step + suggestion: from ideas node
   - Connect **Research with Perplexity → Generate Final Script**.

13. **Add “Update Sheet with Script”** (`Google Sheets` node)  
   - Operation: **Update**
   - Match on column: `id`
   - Set values:
     - `id`: reference the current reel id (ideally from the same item stream as the row you appended)
     - `scrapedTranscript`: transcription text
     - `newTranscript`: generated script text
   - Connect **Generate Final Script → Update Sheet with Script**.

14. **(Recommended hardening) Add an IF node after “Filter + Generate Script Ideas”**  
   - Condition: `{{$json.message.content.verdict}}` equals `"true"`  
   - True path → Perplexity + script generation  
   - False path → optionally update sheet with transcript only or skip

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “InstaReel Script Builder” overview, setup requirements, and tip to adjust the Limit node for batch size. | Included in the workflow Sticky Note. Link: https://www.linkedin.com/in/nealjmcleod/ |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.