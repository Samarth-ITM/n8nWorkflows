Create and publish AI videos with Sora 2 Cameos, Gemini, and Blotato

https://n8nworkflows.xyz/workflows/create-and-publish-ai-videos-with-sora-2-cameos--gemini--and-blotato-13085


# Create and publish AI videos with Sora 2 Cameos, Gemini, and Blotato

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Create and publish AI videos with Sora 2 Cameos, Gemini, and Blotato  
Workflow name (internal): **AI videos with Sora 2 Cameos**

---

## 1. Workflow Overview

This workflow automates an end-to-end pipeline that:
1) collects viral TikTok content ideas,  
2) analyzes a selected viral video using **Gemini**,  
3) generates two **Sora 2 Cameos**-ready prompts with a fixed voice/persona,  
4) generates two 15s portrait clips via a Sora generation API, waits for render completion,  
5) concatenates clips into one final video,  
6) auto-publishes to **TikTok, Facebook, Instagram** via **Blotato**,  
7) updates **Google Sheets** with success/error status.

### 1.1 Viral Content Collector (scheduled intake → Google Sheets)
Runs on a schedule, scrapes a TikTok profile’s popular videos using Apify, normalizes fields, and appends rows to a Google Sheet as “Pending”.

### 1.2 Prompt & Script Generation (Gemini video analysis → JSON prompts)
Loads one “Pending” row from Google Sheets, sends its video URL to Gemini for analysis, then uses a LangChain Agent (Gemini chat model + structured output parser) to produce **exactly two prompts** with strict persona/voice constraints.

### 1.3 Video Generation (Sora API job → polling → collect clip URLs)
Splits the two prompts, calls a Sora generation endpoint for each, waits, polls status until complete, then aggregates the output clip URLs.

### 1.4 Video Merges (concatenate clips → final media URL)
Transforms the aggregated list into the JSON format required by a video concatenation service, then calls an HTTP endpoint to merge.

### 1.5 Auto Publishing (Blotato multi-platform publishing)
Publishes the merged video to TikTok, Facebook Page, and Instagram using Blotato.

### 1.6 Logging & Status Tracking (Google Sheets update)
Marks the source row as **Done** if publishing succeeds, or **Error** if publishing fails (per platform node error output).

---

## 2. Block-by-Block Analysis

## Block 1 — Viral Content Collector
**Overview:** Scrapes viral content from a target TikTok profile every 6 hours, normalizes key metrics/links, and stores them as “Pending” rows in Google Sheets for later processing.

**Nodes involved:**
- Daily Content Scheduler
- Fetch Viral Content Dataset
- Normalize Content Fields
- Save Viral Content to Sheet

### Node: Daily Content Scheduler
- **Type / role:** Schedule Trigger; starts the collector pipeline.
- **Config choices:** Runs every **6 hours**.
- **Inputs/outputs:** Entry node → outputs to **Fetch Viral Content Dataset**.
- **Edge cases:** If n8n instance is paused/offline, runs may be delayed/missed.

### Node: Fetch Viral Content Dataset
- **Type / role:** Apify node (`Run actor and get dataset`) to scrape TikTok.
- **Config choices:**
  - Actor: **TikTok Scraper (clockworks/tiktok-scraper)**.
  - Scrapes profile `@donal.store`, section `videos`, sorting `popular`, `resultsPerPage: 10`.
  - Downloads videos enabled (`shouldDownloadVideos: true`), others disabled.
  - Proxy country code `VN`.
  - Stores videos in kv store `my-tiktok-ASMR`.
- **Credentials:** `apifyOAuth2Api` required.
- **Outputs:** Dataset items → **Normalize Content Fields**.
- **Edge cases / failures:**
  - Apify rate limits, actor failures, or TikTok anti-bot responses.
  - Dataset schema changes: fields like `text`, `createTimeISO`, `authorMeta`, `mediaUrls` may differ.

### Node: Normalize Content Fields
- **Type / role:** Set node; maps scraped fields to a consistent schema for Sheets.
- **Key expressions/variables:**
  - `Caption = {{$json.text}}`
  - `Date Post = {{$json.createTimeISO.toDateTime().format('dd/LL/yyyy')}}`
  - `Username = {{$json.authorMeta.name}}`
  - `Link user = {{$json.authorMeta.profileUrl}}`
  - `Link video = {{$json.webVideoUrl}}`
  - `VideoURL = {{$json.mediaUrls}}` (array)
  - Metrics: `Likes`, `Shares`, `View`, `Save`
- **Outputs:** Normalized item → **Save Viral Content to Sheet**.
- **Edge cases:**
  - `createTimeISO` missing or not parseable → date formatting expression error.
  - `mediaUrls` empty → later nodes expect `VideoURL[0]`.

### Node: Save Viral Content to Sheet
- **Type / role:** Google Sheets append; stores new ideas.
- **Config choices:**
  - Spreadsheet: **Sora Cameos** (documentId `12gO8...`).
  - Sheet `gid=0`.
  - Appends columns including `Status: "Pending"` and `VideoURL: {{$json.VideoURL[0]}}`.
- **Credentials:** Google Sheets OAuth/Service account as configured in n8n.
- **Outputs:** Appended row result (not used downstream here).
- **Edge cases:**
  - Duplicate entries: no deduplication logic; repeated scheduled scrapes can add duplicates.
  - If `VideoURL[0]` is undefined, Sheets receives blank URL, breaking downstream analysis.

---

## Block 2 — Prompt & Script Generation
**Overview:** Periodically pulls one “Pending” row, analyzes its video content with Gemini, then generates a strictly formatted JSON payload containing a title, description, and two Sora-ready prompts using a LangChain Agent with a fixed persona/voice profile.

**Nodes involved:**
- Video Processing Scheduler
- Load Viral Content
- Analyze Viral Idea
- Gemini LLM
- Reasoning Engine
- Parse Script Output
- AI Script & Prompt Agent
- Split Video Prompt

### Node: Video Processing Scheduler
- **Type / role:** Schedule Trigger; starts the “process pending row” pipeline.
- **Config choices:** Interval is defined but empty in JSON (`interval:[{}]`), which typically means misconfigured/default.
- **Outputs:** → **Load Viral Content**.
- **Edge cases:**
  - If schedule rule is invalid, the node may not fire. Verify in UI and set an explicit interval (e.g., every hour).

### Node: Load Viral Content
- **Type / role:** Google Sheets read; fetches a row with `Status = Pending`.
- **Config choices:**
  - `returnFirstMatch: true` (processes a single row per run).
  - Filters show `Status = Pending` twice (redundant but harmless).
- **Outputs:** Selected row → **Analyze Viral Idea**.
- **Edge cases:**
  - If no “Pending” rows exist, node returns no items; downstream nodes won’t run.
  - Requires that the sheet includes `row_number` for later update nodes.

### Node: Analyze Viral Idea
- **Type / role:** LangChain Google Gemini “video analyze”; extracts a textual analysis from a video URL.
- **Config choices:**
  - Resource: `video`, Operation: `analyze`
  - Model: `models/gemini-3-pro-preview`
  - `videoUrls = {{$json.VideoURL}}`
  - Prompt text: “Detailed video description including detailed analysis of setting and characters…”
- **Credentials:** Google PaLM/Gemini (`googlePalmApi`).
- **Outputs:** Gemini analysis object → **AI Script & Prompt Agent**.
- **Edge cases:**
  - Video URL not publicly accessible / geo-blocked → analysis fails.
  - Large/unsupported media → timeout or API error.
  - Output shape assumed later: `{{$json.content.parts[0].text}}`.

### Node: Gemini LLM
- **Type / role:** LangChain Chat LLM provider for the Agent.
- **Config choices:** `models/gemini-3-pro-preview`.
- **Connections:** Provides the LLM to:
  - **AI Script & Prompt Agent** (ai_languageModel)
  - **Parse Script Output** (ai_languageModel)
- **Edge cases:** Model availability/preview deprecations; adjust model name when Google updates preview versions.

### Node: Reasoning Engine
- **Type / role:** LangChain “toolThink”; allows the agent to do internal reasoning/tool use.
- **Connections:** Tool → **AI Script & Prompt Agent** (ai_tool).
- **Edge cases:** Typically low risk; if agent/tool wiring is broken, the agent may fail to execute planned steps.

### Node: Parse Script Output
- **Type / role:** Structured Output Parser; enforces JSON structure and can auto-fix invalid JSON.
- **Config choices:**
  - `autoFix: true`
  - Schema example expects:
    ```json
    { "title":"string","description":"string","prompts":[{"prompt":"string"}] }
    ```
  - Note: The agent system message requests `prompts` as an array of strings, but the parser example expects objects `{prompt:string}`. Because `autoFix` is enabled, it may coerce; still, this mismatch is a key fragility.
- **Connections:** Parser → **AI Script & Prompt Agent** (ai_outputParser).
- **Edge cases:** If Gemini returns malformed JSON or mismatched types, parser may fail or “fix” into an unexpected structure.

### Node: AI Script & Prompt Agent
- **Type / role:** LangChain Agent; transforms analysis into two prompts.
- **Config choices:**
  - Input text: `Reference analysis extracted from the video: {{$json.content.parts[0].text}}`
  - System message: strict persona:
    - One consistent main character `@cameo`
    - Fixed voice profile: gentle/calm/warm, first-person, slow pace, consistent emotion
    - Output must be **valid JSON only** with `title`, `description`, `prompts` (two entries)
- **Connections:**
  - Receives LLM from **Gemini LLM**
  - Receives tool from **Reasoning Engine**
  - Receives parser from **Parse Script Output**
  - Main output → **Split Video Prompt**
- **Edge cases:**
  - Output schema mismatch with parser can break downstream `Split Video Prompt` (`output.prompts` path).
  - If analysis text is empty, the agent may produce generic prompts.

### Node: Split Video Prompt
- **Type / role:** Split Out; splits the two prompts into separate items for clip generation.
- **Config choices:** `fieldToSplitOut = output.prompts`
- **Outputs:** Each prompt item → **Generate Sora Video**.
- **Edge cases:** If `output.prompts` is not an array, node errors. Ensure agent/parser produce `output.prompts`.

---

## Block 3 — Video Generation (Sora 2 Cameos)
**Overview:** For each prompt, requests video generation via an HTTP API, waits, polls job status, and routes based on completion/failure. Completed clips are collected for concatenation.

**Nodes involved:**
- Generate Sora Video
- Wait for Render
- Get Video
- Render Status Check1
- Aggregate Video Parts

### Node: Generate Sora Video
- **Type / role:** HTTP Request; submits a Sora generation job.
- **Config choices:**
  - POST `https://api.geminigen.ai/uapi/v1/video-gen/sora`
  - Multipart form-data body:
    - `prompt = {{$json.prompt}}`
    - `model = sora-2`
    - `resolution = small`
    - `duration = 15`
    - `aspect_ratio = portrait`
  - Auth: `httpHeaderAuth` via “genericCredentialType”
- **Credentials:** `httpHeaderAuth` named **Vbee API** (must include required header key/value, e.g., Authorization).
- **Outputs:** Expected to return a job identifier like `uuid` used later.
- **Edge cases:**
  - API may return different fields (e.g., `id` vs `uuid`).
  - 429/rate limiting; large queues; intermittent 5xx.
  - If `prompt` is missing (schema mismatch), request fails.

### Node: Wait for Render
- **Type / role:** Wait node; pauses before status polling.
- **Config choices:** No explicit wait duration configured in JSON; in UI it may be set or default behavior may be “resume via webhook”.
- **Connections:** → **Get Video**.
- **Edge cases:** If no time or resume trigger is configured, executions may remain waiting indefinitely.

### Node: Get Video
- **Type / role:** HTTP Request; fetches render status and outputs.
- **Config choices:**
  - GET `https://api.geminigen.ai/uapi/v1/history/{{$json.uuid}}`
  - Auth configured as generic header auth (credentials not explicitly attached in JSON here—ensure it uses the same header auth as generation, if required by the API).
- **Outputs:** → **Render Status Check1**.
- **Edge cases:**
  - If upstream doesn’t provide `$json.uuid`, URL becomes invalid.
  - Auth missing/mismatched leads to 401/403.

### Node: Render Status Check1
- **Type / role:** Switch; routes by render status.
- **Config choices / rules:**
  - `Processing` if `status == 1` (number equals)
  - `Completed` if `status == 2`
  - `Failed` if `status == "3"` (string equals) — inconsistent typing vs earlier numeric checks.
- **Connections:**
  - Processing → **Wait for Render** (poll again)
  - Completed → **Aggregate Video Parts**
  - Failed: not connected (implicitly stops for that item)
- **Edge cases:**
  - Status type inconsistencies can misroute (e.g., numeric 3 won’t match `"3"`).
  - If API uses different status codes, switch won’t work and flow may loop forever.

### Node: Aggregate Video Parts
- **Type / role:** Aggregate; collects generated clip URLs across items.
- **Config choices:** Aggregates field `generated_video[0].video_url`.
- **Outputs:** Single aggregated item → **Prepare Video Metadata**.
- **Edge cases:**
  - If the API output path differs, aggregation yields empty list.
  - When one of the two clips fails, you may end with 1 URL; concatenation service behavior should be checked.

---

## Block 4 — Video Merges
**Overview:** Converts the aggregated URL list into the expected payload and calls a concatenation endpoint to produce one final publishable video URL.

**Nodes involved:**
- Prepare Video Metadata
- Merge Video Clips

### Node: Prepare Video Metadata
- **Type / role:** Code node; builds final JSON payload string for concatenation API.
- **Key logic (interpreted):**
  - Reads first aggregated item: `const item = $input.first()`
  - Takes `item.json.video_url` (array of strings)
  - Converts into `video_urls: [{video_url: link1}, ...]`
  - Adds static `id: "2323"`
  - Outputs `{ output: JSON.stringify(finalObject) }`
- **Outputs:** → **Merge Video Clips**
- **Edge cases:**
  - Assumes aggregated field is `video_url` but aggregate node aggregates `generated_video[0].video_url` into a field that may not be named `video_url` depending on n8n aggregate output; verify actual output key.
  - Static `id` may need to be unique depending on the merge service.

### Node: Merge Video Clips
- **Type / role:** HTTP Request; concatenates clips.
- **Config choices:**
  - POST `https://no-code-architects-toolkit-.../v1/video/concatenate`
  - Content-Type JSON
  - Body: `{{$json.output}}` (stringified JSON)
  - Auth: header auth (credential referenced generically; must be configured)
- **Outputs:** Response → three publish nodes (TikTok/Facebook/Instagram).
- **Edge cases:**
  - Service may expect an object, not a JSON string; if so, remove `JSON.stringify` and send raw JSON.
  - Response shape assumed by publishers as `{{$json.response}}` (must match actual merge API response).

---

## Block 5 — Auto Publishing
**Overview:** Publishes the merged video URL to three platforms using Blotato. Each publisher continues on error and routes to either success or error sheet updates.

**Nodes involved:**
- Publish to TikTok
- Publish to Facebook
- Publish to Instagram

### Node: Publish to TikTok
- **Type / role:** Blotato publish node.
- **Config choices:**
  - Platform: TikTok
  - Account: `23867` (kytagiang)
  - Text: `{{ $('AI Agent1').item.json.output.title }}`
  - Media URLs: `{{ $json.response }}`
  - `onError: continueErrorOutput` (so it emits on an error output as well)
- **Credentials:** Blotato API credential `Blotato GiangxAI`.
- **Connections:** Success → **Update Success Status**; Error → **Update Error Status**.
- **Edge cases:**
  - **Broken reference:** node references `$('AI Agent1')` but the actual agent node name is **AI Script & Prompt Agent**. This will fail unless a node named “AI Agent1” exists in the real environment. Must update expressions to `$('AI Script & Prompt Agent')...`.
  - TikTok publishing restrictions, media format limits, rate limits.

### Node: Publish to Facebook
- **Type / role:** Blotato publish node.
- **Config choices:**
  - Platform: Facebook
  - Account: `16978` (Giang VT)
  - Page: `884148598111681` (Quỳnh Hoa Trần AI)
  - Text: `{{ $('AI Agent1').item.json.output.title }}`
  - Media: `{{ $json.response }}`
  - `onError: continueErrorOutput`
- **Connections:** Success/Error → status updates.
- **Edge cases:** Same broken `AI Agent1` reference; Facebook page permissions, Reels/video constraints.

### Node: Publish to Instagram
- **Type / role:** Blotato publish node.
- **Config choices:**
  - Account: `25299`
  - Text:
    ```
    {{ $('AI Agent1').item.json.output.title }}.
    {{ $('AI Agent1').item.json.output.description }}
    ```
  - Media: `{{ $json.response }}`
  - `onError: continueErrorOutput`
- **Connections:** Success/Error → status updates.
- **Edge cases:** Same broken reference; IG media requirements; caption length/hashtag strategy not implemented.

---

## Block 6 — Logging & Status Tracking
**Overview:** Updates the processed Google Sheets row to “Done” or “Error” using the row_number from the loaded Pending row.

**Nodes involved:**
- Update Success Status
- Update Error Status

### Node: Update Success Status
- **Type / role:** Google Sheets update.
- **Config choices:**
  - Sets `Status = "Done"`
  - Matches row using `row_number = {{ $('Load Viral Content').item.json.row_number }}`
  - Matching columns: `row_number`
- **Edge cases:**
  - If `row_number` is missing from Load Viral Content output, update will fail.
  - If multiple platform publishes run, each may trigger an update; first success updates to Done, later errors could flip to Error depending on execution timing (see note below).

### Node: Update Error Status
- **Type / role:** Google Sheets update.
- **Config choices:** Same as success but sets `Status = "Error"`.
- **Edge cases:** Same as above; additionally, a single platform failure marks the whole row as Error even if other platforms succeed.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation: video generation block | — | — | ### Video Generation<br>Automatically creates AI videos using Sora 2 Cameos based on generated prompts, with retry and status handling. |
| Sticky Note2 | Sticky Note | Documentation: merge block | — | — | ### Video Merges<br>Merges, processes, and finalizes video outputs into a publish-ready format. |
| Sticky Note3 | Sticky Note | Documentation: publishing block | — | — | ### Auto Publishing<br>Automatically publishes videos to multiple social platforms (TikTok, Facebook, Instagram). |
| Sticky Note4 | Sticky Note | Documentation: logging block | — | — | ### Logging & Status Tracking<br>Updates Google Sheets with success or error status for monitoring and analytics. |
| Sticky Note5 | Sticky Note | Documentation: collector block | — | — | ### Viral Content Collector<br>Automatically collects viral content ideas for AI video creation. |
| Sticky Note6 | Sticky Note | Documentation: prompt/script block | — | — | ### Prompt & Script Generation<br>Analyzes viral content and uses AI to generate structured video prompts and scripts optimized for Sora 2 Cameos. |
| Daily Content Scheduler | Schedule Trigger | Triggers viral content scraping | — | Fetch Viral Content Dataset | # 🛠️ Workflow Setup Guide<br>Author: [GiangxAI](https://www.youtube.com/@giangxai.official)<br>… |
| Fetch Viral Content Dataset | Apify | Scrape TikTok viral videos dataset | Daily Content Scheduler | Normalize Content Fields | ### Viral Content Collector<br>Automatically collects viral content ideas for AI video creation. |
| Normalize Content Fields | Set | Normalize fields for sheet storage | Fetch Viral Content Dataset | Save Viral Content to Sheet | ### Viral Content Collector<br>Automatically collects viral content ideas for AI video creation. |
| Save Viral Content to Sheet | Google Sheets | Append “Pending” ideas to sheet | Normalize Content Fields | — | ### Viral Content Collector<br>Automatically collects viral content ideas for AI video creation. |
| Video Processing Scheduler | Schedule Trigger | Triggers processing of pending ideas | — | Load Viral Content | # 🛠️ Workflow Setup Guide<br>Author: [GiangxAI](https://www.youtube.com/@giangxai.official)<br>… |
| Load Viral Content | Google Sheets | Load first row where Status=Pending | Video Processing Scheduler | Analyze Viral Idea | ### Prompt & Script Generation<br>Analyzes viral content and uses AI… |
| Analyze Viral Idea | Google Gemini (LangChain) | Analyze a viral video from URL | Load Viral Content | AI Script & Prompt Agent | ### Prompt & Script Generation<br>Analyzes viral content and uses AI… |
| Gemini LLM | Gemini Chat Model (LangChain) | LLM provider for agent/parser | — | AI Script & Prompt Agent; Parse Script Output | ### Prompt & Script Generation<br>Analyzes viral content and uses AI… |
| Reasoning Engine | Tool Think (LangChain) | Agent reasoning tool | — | AI Script & Prompt Agent | ### Prompt & Script Generation<br>Analyzes viral content and uses AI… |
| Parse Script Output | Structured Output Parser (LangChain) | Enforce/auto-fix JSON structure | Gemini LLM (AI) | AI Script & Prompt Agent (AI parser) | ### Prompt & Script Generation<br>Analyzes viral content and uses AI… |
| AI Script & Prompt Agent | Agent (LangChain) | Produce title/description + 2 prompts | Analyze Viral Idea (+ Gemini LLM/tool/parser) | Split Video Prompt | ### Prompt & Script Generation<br>Analyzes viral content and uses AI… |
| Split Video Prompt | Split Out | Split two prompts into separate items | AI Script & Prompt Agent | Generate Sora Video | ### Video Generation<br>Automatically creates AI videos… |
| Generate Sora Video | HTTP Request | Submit Sora generation job | Split Video Prompt | Wait for Render | ### Video Generation<br>Automatically creates AI videos… |
| Wait for Render | Wait | Delay/pause before polling | Generate Sora Video; Render Status Check1 (processing loop) | Get Video | ### Video Generation<br>Automatically creates AI videos… |
| Get Video | HTTP Request | Poll render status/result | Wait for Render | Render Status Check1 | ### Video Generation<br>Automatically creates AI videos… |
| Render Status Check1 | Switch | Route by render status | Get Video | Wait for Render; Aggregate Video Parts | ### Video Generation<br>Automatically creates AI videos… |
| Aggregate Video Parts | Aggregate | Collect clip URLs | Render Status Check1 (completed) | Prepare Video Metadata | ### Video Merges<br>Merges, processes… |
| Prepare Video Metadata | Code | Build concatenate payload | Aggregate Video Parts | Merge Video Clips | ### Video Merges<br>Merges, processes… |
| Merge Video Clips | HTTP Request | Concatenate clips into final video | Prepare Video Metadata | Publish to Facebook; Publish to TikTok; Publish to Instagram | ### Video Merges<br>Merges, processes… |
| Publish to TikTok | Blotato | Publish final video to TikTok | Merge Video Clips | Update Success Status; Update Error Status | ### Auto Publishing<br>Automatically publishes… |
| Publish to Facebook | Blotato | Publish final video to Facebook page | Merge Video Clips | Update Success Status; Update Error Status | ### Auto Publishing<br>Automatically publishes… |
| Publish to Instagram | Blotato | Publish final video to Instagram | Merge Video Clips | Update Success Status; Update Error Status | ### Auto Publishing<br>Automatically publishes… |
| Update Success Status | Google Sheets | Mark row Done | Publish nodes (success outputs) | — | ### Logging & Status Tracking<br>Updates Google Sheets… |
| Update Error Status | Google Sheets | Mark row Error | Publish nodes (error outputs) | — | ### Logging & Status Tracking<br>Updates Google Sheets… |
| Sticky Note1 | Sticky Note | Global setup notes/credits | — | — | # 🛠️ Workflow Setup Guide<br>Author: [GiangxAI](https://www.youtube.com/@giangxai.official)<br>… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Google Sheet**
   1. Create a spreadsheet (e.g., “Sora Cameos”).
   2. Add columns at least: `Caption, Date Post, Username, Link user, Link video, VideoURL, Likes, Shares, View, Save, Status`.
   3. Ensure your Google Sheets node in n8n returns a `row_number` field (n8n typically includes it when reading rows).

2) **Collector pipeline (viral intake)**
   1. Add **Schedule Trigger** named *Daily Content Scheduler* → set to **every 6 hours**.
   2. Add **Apify** node named *Fetch Viral Content Dataset*:
      - Operation: *Run actor and get dataset*
      - Actor: `clockworks/tiktok-scraper` (or the listed actor)
      - Input JSON: set profile(s), sorting popular, results per page, download videos true.
      - Credentials: configure **Apify OAuth2**.
   3. Add **Set** node *Normalize Content Fields*:
      - Map fields to: Caption, Date Post (format), Username, links, VideoURL array, Likes/Shares/View/Save.
   4. Add **Google Sheets** node *Save Viral Content to Sheet*:
      - Operation: *Append*
      - Map columns and set `Status = Pending`
      - Use `VideoURL = {{$json.VideoURL[0]}}`.
   5. Connect: Scheduler → Apify → Set → Sheets append.

3) **Processing pipeline (pick pending row)**
   1. Add **Schedule Trigger** *Video Processing Scheduler* and set an explicit interval (e.g., every 30 minutes).
   2. Add **Google Sheets** node *Load Viral Content*:
      - Operation: *Read / Get many* (as per UI)
      - Filter: `Status` equals `Pending`
      - Option: `Return First Match` enabled.
   3. Connect: Processing Scheduler → Load Viral Content.

4) **Gemini video analysis**
   1. Add **Google Gemini** (LangChain) node *Analyze Viral Idea*:
      - Resource: `video`, Operation: `analyze`
      - Model: pick a current Gemini model (the workflow uses `gemini-3-pro-preview`)
      - `videoUrls = {{$json.VideoURL}}`
      - Credentials: configure Google Gemini/PaLM API in n8n.
   2. Connect: Load Viral Content → Analyze Viral Idea.

5) **Agent to generate 2 prompts (structured JSON)**
   1. Add **Gemini Chat Model** node *Gemini LLM* (LangChain LLM provider) with the same model.
   2. Add **Structured Output Parser** node *Parse Script Output*:
      - Enable **Auto-fix**
      - Use a schema that matches what you want downstream. Recommended schema for this workflow:
        - `title: string`
        - `description: string`
        - `prompts: string[]` (two strings)
   3. Add **Tool Think** node *Reasoning Engine* (optional but included).
   4. Add **Agent** node *AI Script & Prompt Agent*:
      - Prompt type: define
      - System message: paste the fixed persona/voice rules and JSON-only output rules.
      - Input text: use `{{$json.content.parts[0].text}}` from the analysis.
      - Attach **Gemini LLM** as language model, **Parse Script Output** as output parser, and **Reasoning Engine** as tool.
   5. Connect: Analyze Viral Idea → AI Script & Prompt Agent.

6) **Split prompts and generate Sora clips**
   1. Add **Split Out** node *Split Video Prompt*:
      - Field to split: `output.prompts`
   2. Add **HTTP Request** node *Generate Sora Video*:
      - POST `https://api.geminigen.ai/uapi/v1/video-gen/sora`
      - Content-Type: multipart/form-data
      - Body params: `prompt`, `model=sora-2`, `resolution=small`, `duration=15`, `aspect_ratio=portrait`
      - Credentials: create **HTTP Header Auth** (e.g., `Authorization: Bearer <token>`), name it as you like.
   3. Add **Wait** node *Wait for Render*:
      - Configure a wait time (e.g., 30–60 seconds) to poll; avoid indefinite waits.
   4. Add **HTTP Request** node *Get Video*:
      - GET `https://api.geminigen.ai/uapi/v1/history/{{$json.uuid}}`
      - Use same header auth if required.
   5. Add **Switch** node *Render Status Check1*:
      - Processing: `status == 1`
      - Completed: `status == 2`
      - Failed: `status == 3` (make type consistent—prefer numeric).
      - Processing → back to Wait node (poll loop)
      - Completed → aggregation
   6. Connect: Agent → Split → Generate → Wait → Get → Switch → (Processing back to Wait; Completed onward).

7) **Aggregate URLs and concatenate**
   1. Add **Aggregate** node *Aggregate Video Parts*:
      - Aggregate field: `generated_video[0].video_url` (confirm actual API response path).
   2. Add **Code** node *Prepare Video Metadata*:
      - Build payload:
        - `video_urls: [{video_url: <url>}, ...]`
        - any required `id`
      - Prefer output as a proper JSON object if the merge API expects JSON (not a string).
   3. Add **HTTP Request** node *Merge Video Clips*:
      - POST to your concatenate endpoint
      - Send JSON body with the structure required by that service
      - Add header auth if required.
   4. Connect: Switch (Completed) → Aggregate → Code → Merge.

8) **Publish via Blotato**
   1. Add 3 **Blotato** nodes:
      - *Publish to TikTok* (platform tiktok, choose account)
      - *Publish to Facebook* (platform facebook, choose account + page)
      - *Publish to Instagram* (choose account)
   2. Set each node:
      - `postContentMediaUrls` to the merged video URL field returned by *Merge Video Clips* (verify exact response field; workflow uses `{{$json.response}}`)
      - Caption/text from agent output.
   3. Important: fix expression references to the correct agent node name, e.g.:
      - `{{$('AI Script & Prompt Agent').item.json.output.title}}`
   4. Set each Blotato node **On Error** to “Continue (error output)”.
   5. Connect: Merge Video Clips → all three Blotato nodes.

9) **Update Google Sheets status**
   1. Add **Google Sheets** node *Update Success Status*:
      - Operation: Update
      - Match by `row_number`
      - Set `Status = Done`
      - `row_number = {{$('Load Viral Content').item.json.row_number}}`
   2. Add **Google Sheets** node *Update Error Status* similarly with `Status = Error`.
   3. Connect each publish node:
      - Main success output → Update Success Status
      - Error output → Update Error Status

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: GiangxAI | https://www.youtube.com/@giangxai.official |
| Workflow operating concept: collect viral ideas → choose Sora2 cameo → generate prompts → generate clips → merge → publish → update status | Sticky note “Workflow Setup Guide” |
| Key implementation warning: Blotato captions reference `$('AI Agent1')` which does not exist in the provided workflow; update to the actual agent node name. | Expressions inside Publish nodes |
| Schedule warning: “Video Processing Scheduler” interval appears empty; set a real schedule in the UI. | Video processing entrypoint |

