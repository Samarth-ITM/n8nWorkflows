Estimate construction costs from text, photos and PDFs with Telegram, GPT‑4/Gemini and DDC CWICR

https://n8nworkflows.xyz/workflows/estimate-construction-costs-from-text--photos-and-pdfs-with-telegram--gpt-4-gemini-and-ddc-cwicr-12176


# Estimate construction costs from text, photos and PDFs with Telegram, GPT‑4/Gemini and DDC CWICR

## 1. Workflow Overview

**Workflow name:** `DDC CWICR v10.9 - Construction Cost Estimator Bot`  
**Purpose:** A Telegram bot that accepts **photos**, **PDF drawings**, or **text descriptions** and returns a **construction cost estimate** by:
1) extracting work items with AI (Gemini Vision or OpenAI Vision / Gemini text),  
2) matching each work item to a **Qdrant vector database** of ~55k construction rates using **OpenAI embeddings**,  
3) re-ranking candidates with an LLM,  
4) computing unit and total costs plus resource breakdown,  
5) generating **HTML report** + **CSV** and “PDF” export (HTML file).

### 1.1 Entry & Session Management (Telegram → Action routing)
- Receives Telegram updates (messages + callback queries).
- Maintains per-chat session state in **workflow static data**.
- Determines `action` (language menu, ask photo, analyze, edit menu, calculate, exports, PDF processing, etc.).

### 1.2 Localization & Configuration
- Loads language pack (9 languages) and selects the appropriate **Qdrant collection** and currency settings.
- Provides all UI messages and labels used in Telegram.

### 1.3 Photo Vision Analysis (Photo → Works list)
- Downloads Telegram photo(s), converts to base64 (or reuses stored base64 on “Refine”).
- Calls **Gemini 2.0 Flash** or **OpenAI GPT‑4o** vision chat endpoint.
- Parses strict JSON response into normalized works list.
- Shows works list with edit buttons in Telegram.

### 1.4 Text Analysis (Text → Works list)
- Sends user text to LLM (Gemini or OpenAI) to extract a JSON array of works.
- Saves works into session and shows works list for editing.

### 1.5 PDF Floor Plan Processing (PDF → Rooms + Works list)
- Downloads PDF from Telegram, estimates/splits into up to **3 pages** (heuristic by size).
- Calls Gemini “generateContent” with inline PDF for each page.
- Parses rooms/areas and derived works, accumulates across pages, deduplicates, and shows works list.

### 1.6 Cost Calculation Loop (Works → Rate match → Cost)
For each work item (limited to 5 in FREE DEMO mode):
- LLM transforms query → OpenAI embedding → Qdrant vector search → LLM rerank → cost compute.
- Updates a Telegram “progress/work” message per item.
- Accumulates results in static data.

### 1.7 Aggregation & Reporting (Results → Summary + HTML/Exports)
- Deletes progress messages (best effort).
- Aggregates totals and breakdowns, stores `lastResults` and HTML report in static data.
- Sends final Telegram summary with export buttons.
- Exports: CSV via Telegram document, “PDF” via HTML file, and separate HTML report file.

---

## 2. Block-by-Block Analysis

### Block A — Project / Setup Notes (Sticky Notes)
**Overview:** Documentation and setup guidance embedded as sticky notes.  
**Nodes involved:** `Sticky Note1`, `🔐 Credentials Setup`, `Intro`, `Checklist`, `Telegram Credentials`, `UI Messages`, `Route Switch`, `Config & Localization`, `Main Router`, `Edit Menu` (sticky), `Block 4 - Vision`, `Block 5 - PDF`, `Block 6 - Calculation`, `Block 7 - Reports`, `Block 8 - Export`, `Qdrant Info`  
**Node details:** These are n8n Sticky Note nodes; they do not execute. They describe repo link, credential JSON, features, and block diagrams.

**Edge cases:** None (non-executing).

---

### Block B — Entry Point & Credentials Injection
**Overview:** Receives Telegram updates and injects API keys/bot token into the execution context.  
**Nodes involved:** `Telegram Trigger`, `🔑 TOKEN`

#### Node: Telegram Trigger
- **Type/Role:** `telegramTrigger` — workflow entrypoint, listens to Telegram updates.
- **Config choices:** Subscribes to `message` and `callback_query`. Uses a Telegram API credential (webhook mode implied by notes).
- **Outputs:** Raw Telegram update JSON.
- **Failure modes:** Telegram credential misconfigured, webhook not registered, bot token invalid, Telegram downtime.

#### Node: 🔑 TOKEN
- **Type/Role:** `set` (raw JSON output) — central “secrets/config” injector.
- **Configuration:** Outputs:
  - `bot_token`
  - `AI_PROVIDER` (`gemini` or `openai`)
  - `GEMINI_API_KEY`, `OPENAI_API_KEY`
  - `QDRANT_URL`, `QDRANT_API_KEY`
- **Connections:** `Telegram Trigger → 🔑 TOKEN → Main`
- **Failure modes:** Missing/placeholder keys causes downstream errors (vision/text rerank/embedding/Qdrant).

**Version requirements:** `Set` node v3.4 supports `mode=raw` JSON output as used.

---

### Block C — Main Router + Localization
**Overview:** Interprets Telegram messages/callbacks, manages per-chat session state, sets `action`, then applies localization and routes to the appropriate branch.  
**Nodes involved:** `Main`, `Config`, `Route`

#### Node: Main
- **Type/Role:** `code` — central message handler/state machine.
- **Key behavior:**
  - Reads update from `Telegram Trigger`.
  - Initializes/reads `sd.sess[cid]` in global static data.
  - Detects:
    - `/start`, `/help`
    - language selection callbacks `lang_XX`
    - photos (including media groups), voice (placeholder action), PDFs (`application/pdf`)
    - callback actions: analyze, add more photos, edit work, qty adjustments, delete work, add work, calculate, exports, restart, help.
  - Produces normalized output: `chatId`, `action`, `lang`, `photos`, `description`, `pdfFileId`, `pdfFileName`, etc.
- **Key variables:** `sd = $getWorkflowStaticData('global')`, `sd.sess[cid]` session object.
- **Outputs:** Single item JSON used by `Config`.
- **Edge cases:**
  - Voice route sets `action='process_voice'` but **no downstream route exists** in `Route` switch rules → would go to fallback output (likely `📤 Fallback` via Route extra).
  - Media group photo collection sets `action='photo_added'` / state `collecting_photos`, but there is no explicit “wait end of album”; user must hit “analyze”.
  - If user sends text while not in `adding_work`, text triggers `analyze_text` only in `wait_photo` state.

#### Node: Config
- **Type/Role:** `code` — localization, db mapping, currency/region settings, PDF fields passthrough.
- **Configuration choices:**
  - Provides `LANGS` dictionary for 9 languages with UI strings and Qdrant collection names.
  - Chooses `L = LANGS[lang]` and sets `db = L.db`.
  - Pulls `voiceFileId`, `pdfFileId`, `pdfFileName` from session if present.
- **Outputs:** `...input + L + db + pdf/voice fields`.
- **Edge cases/failures:**
  - If `input.lang` missing, defaults to EN.
  - Some strings are mixed-language (e.g., EN help text includes “*3. Расчёт*”).
  - If session not initialized, it still returns L; session update only occurs if `sd.sess[chatId]` exists.

#### Node: Route
- **Type/Role:** `switch` — action router.
- **Configuration:** Named outputs for actions:
  - `show_lang`, `lang_selected`, `ask_photo`, `show_analyze_options`, `photo_added`, `analyze`, `works_updated`, `show_edit_menu`, `ask_new_work`, `start_calc`, `export_excel`, `export_pdf`, `show_help`, `view_details`, `refine_analysis`, `analyze_text`, `process_pdf`, plus fallback output `extra`.
- **Connections:** Each output goes to relevant block nodes.
- **Edge cases:** Any action not mapped (notably `process_voice`, `update_categories`) goes to `extra` → `📤 Fallback`.

---

### Block D — Telegram UI: Language, Help, Basic Prompts
**Overview:** Sends language menu, language confirmation, photo request, help text, and fallback message.  
**Nodes involved:** `📤 Lang Menu`, `Answer Lang CB`, `📤 Lang OK`, `Answer Photo CB`, `📤 Ask Photo`, `📤 Help`, `📤 Fallback`

#### Node: 📤 Lang Menu
- **Type:** `httpRequest` (Telegram Bot API `sendMessage`)
- **Role:** Shows language selection inline keyboard.
- **Key expressions:** `bot_token` from `🔑 TOKEN`.
- **Failure modes:** Telegram API errors, invalid token.

#### Node: Answer Lang CB / Answer Photo CB / Answer Calc CB
- **Type:** `httpRequest` (Telegram `answerCallbackQuery`)
- **Role:** Removes Telegram “loading spinner” for callbacks.
- **Config:** `onError=continueRegularOutput` to avoid breaking flow.
- **Edge cases:** Missing `callback_query_id` leads to no-op/Telegram error (ignored).

#### Node: 📤 Lang OK
- Confirms selected language and asks for photo/PDF/text via localized `L.ok` + `L.photo`.

#### Node: 📤 Ask Photo
- Sends localized instruction `L.photo`.

#### Node: 📤 Help
- Sends static help message (not localized beyond button label).
- Has inline “Back” button `back_to_lang` but **Main router does not handle `back_to_lang`** → will fall back.

#### Node: 📤 Fallback
- Sends `L.fallback_start` or “Use /start to begin”.

---

### Block E — Vision (Photo) Analysis Pipeline
**Overview:** Downloads first photo, converts to base64, calls Gemini/OpenAI vision, parses JSON into works, shows editable list. Also supports “Refine analysis” using stored base64.  
**Nodes involved:** `Refine Analysis`, `IF Skip Refine`, `📤 Ask Photo Refine`, `Prep Photo Download`, `IF Skip Download`, `IF No Photos1`, `📤 No Photos Msg1`, `Get File Path1`, `Download Photo File1`, `Convert To Base`, `Use Stored Base64`, `Merge To Vision1`, `Prep Vision1`, `Call Vision1`, `Merge Vision1`, `Parse AI`, `📊 Show Works`, `📤 Send Works`, `📤 Analyze Options`

#### Node: Refine Analysis
- **Type:** `code`
- **Role:** Determines if refine is possible; sets `use_advanced_prompt` flag in session.
- **Logic:** Checks:
  - stored `session.photos_base64` OR photo fileIds in config/session.
  - If none, sets `skipRefine=true` and forces state `wait_photo`.
- **Failure modes:** None critical; safe fallback to asking photo.

#### Node: IF Skip Refine → 📤 Ask Photo Refine
- **IF condition:** `skipRefine == true`
- Sends “Please send photo first” (localized first line of `L.photo`).

#### Node: Prep Photo Download
- **Type:** `code`
- **Role:** If stored base64 exists, skip download; else pick first photo `fileId` to download from Telegram.
- **Outputs:** `skipDownload`, `useStoredBase64`, `noPhotosError`.
- **Edge cases:** Multi-photo support exists in session but pipeline downloads only the **first** photo unless stored base64 contains multiple.

#### Node: IF Skip Download
- Routes to:
  - If `skipDownload=true` → `IF No Photos1`
  - Else → `Get File Path1`

#### Node: IF No Photos1 → 📤 No Photos Msg1 / Use Stored Base64
- If `noPhotosError=true` send error; else use stored base64 and proceed.

#### Node: Get File Path1 / Download Photo File1
- **Type:** `httpRequest`
- **Role:** Telegram `getFile` then download actual file from `file_path` as binary.
- **Failure modes:** Telegram file expired, wrong file_id, bot token invalid.

#### Node: Convert To Base
- **Type:** `code`
- **Role:** Reads binary file, extracts base64, stores to `sd.sess[cid].photos_base64` for later refine.
- **Output:** `{photos:[{base64,...}]}`

#### Node: Merge To Vision1
- Pass-through merge point (mainly logging).

#### Node: Prep Vision1
- **Type:** `code`
- **Role:** Builds provider-specific request:
  - **OpenAI:** `POST /v1/chat/completions`, model `gpt-4o`, content includes text + `image_url` base64.
  - **Gemini:** `generateContent` with multiple `inline_data` images + text prompt, model `gemini-2.0-flash`.
- **Language control:** Forces output ONLY in `L.search_lang`.
- **Failure modes:**
  - Missing API keys returns error in output; downstream still calls API unless guarded elsewhere (not guarded here).
  - Large images may exceed provider limits; Gemini/OpenAI may reject.

#### Node: Call Vision1
- **Type:** `httpRequest`
- **Role:** Executes the vision API call. Adds `Authorization` header only for OpenAI.
- **Config:** `neverError=true` so response errors appear in payload.

#### Node: Merge Vision1
- **Type:** `code`
- **Role:** Normalizes response fields: `candidates` (Gemini) and `choices` (OpenAI).

#### Node: Parse AI
- **Type:** `code`
- **Role:** Extracts JSON object `{description, items:[...]}` from model response, normalizes items.
- **Session write:** `sd.sess[cid].works`, `.description`, `.state='wait_edit'`, `.db`, `.L`
- **Edge cases:** If model returns non-JSON or partial JSON → parse error → empty works.

#### Node: 📤 Analyze Options
- **Role:** After photo added, shows menu “Add more / Analyze now / Help”.

#### Node: 📊 Show Works → 📤 Send Works
- **Show Works (code):** Formats room/work summary and inline keyboard:
  - edit buttons `edit_work_i`
  - add work, calculate, new project
- **Send Works (httpRequest):** Sends message with inline keyboard.

---

### Block F — Text-to-Works Pipeline
**Overview:** Converts free-form text into structured works list via LLM, then shows in edit list.  
**Nodes involved:** `Prep Text LLM`, `Call Text LLM`, `Parse Text LLM`, `📊 Show Works`, `📤 Send Works`

#### Node: Prep Text LLM
- **Type:** `code`
- **Role:** Builds prompt and chooses provider based on `AI_PROVIDER` and key availability.
- **Provider behavior:**
  - Gemini: `gemini-2.0-flash-exp:generateContent`
  - OpenAI: `gpt-4o-mini chat/completions`
- **Output expectation:** JSON array only (no markdown).
- **Edge cases:** If provider key missing, `_llm_api_url` may be undefined → `Call Text LLM` fails.

#### Node: Call Text LLM
- **Type:** `httpRequest`
- **Role:** Calls `_llm_api_url` with `_llm_request_body`.
- **Headers:** Authorization only for OpenAI.

#### Node: Parse Text LLM
- **Type:** `code`
- **Role:** Extracts JSON array from response (Gemini `candidates[0].content.parts[0].text` or OpenAI `choices[0].message.content`).
- **Normalizes works:** id/seq, name/query, qty/unit defaults.
- **Session write:** stores works, description, state, db, L.

---

### Block G — PDF Floor Plan Processing
**Overview:** Downloads PDF from Telegram, estimates up to 3 pages, calls Gemini vision on the PDF, parses rooms/works per page, accumulates & deduplicates, then shows works list.  
**Nodes involved:** `📄 PDF Download Prep`, `📄 Get PDF Path`, `📄 Download PDF`, `📄 Split PDF Pages`, `📝 Prep PDF Message`, `📤 PDF Received`, `📄 Prep Pages Loop`, `🔁 Loop PDF Pages`, `👁️ Prep Vision PDF`, `👁️ Call Vision PDF`, `🏠 Parse PDF Page`, `📦 Accumulate Pages`, `🧹 Deduplicate & Merge`, `📊 Show Works`, `📤 Send Works`

#### Node: 📄 PDF Download Prep
- **Type:** `code`
- **Role:** Ensures `pdfFileId` and `pdfFileName` are present from `Config`.

#### Node: 📄 Get PDF Path / 📄 Download PDF
- **Type:** `httpRequest`
- **Role:** Telegram `getFile` then download the PDF binary.

#### Node: 📄 Split PDF Pages
- **Type:** `code`
- **Role:** Stores base64 PDF in `sd.pdfData[cid]`, estimates `totalPages` by size (`~150KB/page`) capped at **3**.
- **Note:** It does **not** actually split the PDF; it just loops “pageNum” values.

#### Node: 📝 Prep PDF Message → 📤 PDF Received
- Sends “PDF received / analyzing / ~minutes”.

#### Node: 📄 Prep Pages Loop → 🔁 Loop PDF Pages
- **Prep Pages Loop (code):** Creates items `{currentPage, totalPages, pdfBase64}`.
- **Loop PDF Pages:** `splitInBatches` iterates pages.
- **Important:** `👁️ Prep Vision PDF` currently ignores `currentPage` and sends the **full PDF** each time. So looping repeats analysis rather than per-page extraction.

#### Node: 👁️ Prep Vision PDF
- **Type:** `code`
- **Role:** Builds Gemini request for PDF inline_data (`mime_type: application/pdf`), model `gemini-2.0-flash-exp`.
- **Failure modes:** Missing PDF base64 results in `_skip_vision=true`.

#### Node: 👁️ Call Vision PDF
- **Type:** `httpRequest`
- **Role:** Calls Gemini with 120s timeout. `neverError=true`.

#### Node: 🏠 Parse PDF Page
- **Type:** `code`
- **Role:** Parses JSON `{total_area_m2, rooms, works}` from Gemini response.
- **Output:** `pageRooms`, `pageWorks`.

#### Node: 📦 Accumulate Pages
- **Type:** `code`
- **Role:** Appends rooms/works into `sd.pdfAcc[cid]`.

#### Node: 🧹 Deduplicate & Merge
- **Type:** `code`
- **Role:** Deduplicates:
  - rooms by lowercase name (keeps bigger area)
  - works by `(name, unit, room)` (sums qty)
- **Session write:** `sd.sess[cid].works`, `rooms`, `totalArea`, `state='wait_edit'`
- **Cleanup:** deletes `sd.pdfAcc[cid]` and `sd.pdfData[cid]`.

**Key edge cases:**
- “Per-page” loop does not truly split PDF; results may duplicate and rely on dedupe.
- Large/complex drawings may exceed model limits or yield non-JSON.
- Only Gemini is used for PDF; `AI_PROVIDER` does not switch PDF provider.

---

### Block H — Editing Works in Telegram
**Overview:** Allows editing quantities, deleting items, adding new work via text, and returning updated list.  
**Nodes involved:** `Edit Menu` (code), `📤 Edit Menu`, `Works Updated`, `📤 Works Updated`, `📤 Ask New Work`

#### Node: Edit Menu (code)
- **Role:** Builds inline keyboard for qty changes and deletion for `editingWorkIndex`.
- **Failure modes:** Missing or invalid index returns `_skip=true` (but nothing prevents send attempt unless guarded elsewhere).

#### Node: 📤 Edit Menu
- Sends edit menu message with inline keyboard.

#### Node: Works Updated (code) → 📤 Works Updated
- Rebuilds works list message with edit buttons and calculate/new options.

#### Node: 📤 Ask New Work
- Prompts user to enter new work in text format.

**Edge cases:**
- Add-work parsing in `Main` expects “name, qty unit” (comma-separated). If user sends “Drywall 10m2” without comma it becomes name only, qty defaults to 1.

---

### Block I — Calculation Loop (Qdrant + LLM rerank + cost compute)
**Overview:** For each work item, transforms query, embeds, searches Qdrant, re-ranks, computes costs/resources, updates Telegram status, accumulates results. Includes FREE DEMO limit of 5 items.  
**Nodes involved:** `Answer Calc CB`, `📝 Prep Progress`, `📤 Send Progress`, `Save Progress ID`, `Prep Works`, `Loop`, `📝 Prep Work Msg`, `🗑️ Delete Prev`, `📤 Send Work`, `💾 Save Work Msg`, `1️⃣ Prep Query`, `1.5️⃣ LLM Transform`, `2️⃣ Extract Transform`, `3️⃣ OpenAI Embedding`, `4️⃣ Extract Embedding`, `5️⃣ Qdrant Search`, `6️⃣ Prep Rerank`, `7️⃣ LLM Rerank`, `8️⃣ Apply Rerank`, `9️⃣ Calculate`, `📊 Update Result`, `📤 Edit Result`, `Acc`

#### Node: Answer Calc CB → 📝 Prep Progress → 📤 Send Progress → Save Progress ID
- Shows initial progress message.
- Stores progress message id (`sd.progress[cid]`) and resets calculation tracker (`sd.calcProgress[cid].lastMsgId=null`).
- Implements FREE DEMO logic: stores `session.isLimited`, `session.totalWorks`, and `session.limitedWorks`.

#### Node: Prep Works
- Builds loop items from `session.limitedWorks` or first 5 of `session.works`.
- Initializes `sd.res[cid]=[]` accumulator.

#### Node: Loop (splitInBatches)
- Iterates works; `options.reset=false` means it continues until exhausted.

#### Node: 📝 Prep Work Msg → 🗑️ Delete Prev → 📤 Send Work → 💾 Save Work Msg
- Sends a minimal “current work…” message.
- Deletes previous work message (best effort).
- Saves the new message id to `sd.calcProgress[cid].lastMsgId` for later editing.

#### Node: 1️⃣ Prep Query
- Prepares query transformation prompt and injects credentials for OpenAI + Qdrant.
- Detects DB language from collection name to instruct transform.
- If missing `OPENAI_API_KEY` / query / collection → sets `_skip=true`.

#### Node: 1.5️⃣ LLM Transform (OpenAI)
- Calls OpenAI `gpt-4o-mini` to transform the query into better search keywords.
- **Note:** This step always uses OpenAI; Gemini is not used here.

#### Node: 2️⃣ Extract Transform
- Cleans LLM output, combines original query with new keywords.
- Produces `_query` for embeddings.

#### Node: 3️⃣ OpenAI Embedding
- Calls OpenAI embeddings `text-embedding-3-large`, `dimensions=3072`.

#### Node: 4️⃣ Extract Embedding
- Extracts vector, warns if length not 3072.
- Passes Qdrant credentials.

#### Node: 5️⃣ Qdrant Search
- POST `{vector, limit:10, with_payload:true}` to `QDRANT_URL/collections/{collection}/points/search`.
- Header `api-key` used if provided.
- Failure modes: wrong URL, collection missing, API key invalid, timeout.

#### Node: 6️⃣ Prep Rerank
- Builds prompt with top 5 candidates (includes code/name/unit/scope/material hints).
- If no results returns `NOT_FOUND`.
- If Qdrant error returns `QDRANT_ERROR`.

#### Node: 7️⃣ LLM Rerank
- OpenAI `gpt-4o-mini` ranks candidates with JSON `{rankings:[{index,score,reason}]}`.

#### Node: 8️⃣ Apply Rerank
- Parses rerank JSON; fallback uses Qdrant scores.
- Combines LLM (70%) + Qdrant (30%) into `combined_score`.
- Selects `_best_result` and `_best_payload` and sets `_quality_level`.

#### Node: 9️⃣ Calculate
- Extracts payload from multiple possible nesting patterns.
- Computes:
  - total cost from `cost_summary.total_cost_position` or sum of resource costs
  - unit divisor for “100 m² / 10 m …” units
  - unit cost `uc` and total cost `tc`
  - resource scaling and categorization (labor/material/machine heuristics)
  - labor hours from resource units (`ч`, `чел.-ч`, etc.)
  - scope_of_work from `work_steps`
- Failure modes:
  - If payload missing → returns `PAYLOAD_NOT_FOUND` with debug keys.
  - Unit parsing may not cover all formats.

#### Node: 📊 Update Result → 📤 Edit Result → Acc
- Formats per-item “✓ found / not found” summary and edits the last work message.
- Accumulates result into `sd.res[cid]`.
- Loops back to `Loop`.

---

### Block J — Cleanup, Aggregation & Report Generation
**Overview:** After loop completes, deletes progress messages, aggregates totals and quality metrics, generates HTML report, sends final Telegram summary and optional HTML file.  
**Nodes involved:** `🧹 Prep Cleanup`, `🗑️ Delete Work Msg`, `🗑️ Delete Progress Msg`, `Agg`, `Generate HTML`, `Final`, `📤 Final`, `Prep HTML File`, `📤 Send HTML`

#### Node: 🧹 Prep Cleanup → 🗑️ Delete Work Msg → 🗑️ Delete Progress Msg
- Prepares message IDs for deletion from `sd.calcProgress` and `sd.progress`.
- Deletes messages with `neverError=true`.

#### Node: Agg
- Aggregates all results from `sd.res[cid]`:
  - totals, sums, found percentage, labor/material/machine totals, labor hours
  - FREE DEMO info: limited flag and skipped works count
- Cleans static data: deletes `sd.res[cid]`, progress trackers.
- Stores `sd.lastResults = {...}` for exports/details.

#### Node: Generate HTML
- Builds an interactive HTML report with expandable resources and scope of work.
- Stores `sd.html_report`.

#### Node: Final
- Creates compact Telegram Markdown summary with quality markers and resource previews.
- Sets session state to `done`.

#### Node: 📤 Final
- Sends the final Telegram message with inline buttons:
  - “Resources” (details), “Excel”, “PDF”, “New”

#### Node: Prep HTML File → 📤 Send HTML
- Packages HTML report as Telegram document.

**Edge cases:**
- Telegram Markdown formatting is fragile; `Final` uses `escMd()` but not full escaping (Telegram “Markdown” mode is limited).
- HTML report can become large; Telegram file size limits may apply.

---

### Block K — Details View & Exports
**Overview:** Generates CSV (“Excel”), “PDF” (HTML file), and detailed per-work resource breakdown message.  
**Nodes involved:** `View Details`, `📤 Details`, `Generate Excel`, `📤 Send Excel`, `Generate PDF`, `IF PDF`, `📤 Send PDF`

#### Node: View Details → 📤 Details
- Builds a long Markdown message including:
  - each work item, code/name, qty×rate, breakdown, resource lines, scope of work.
- Sends with inline export buttons.

#### Node: Generate Excel → 📤 Send Excel
- Produces CSV (semicolon-separated, BOM included) as binary document.
- Uses `sd.lastResults` data.
- Failure modes: if `lastResults` missing (no calculation yet) → empty file but still produced.

#### Node: Generate PDF → IF PDF → 📤 Send PDF
- Despite name, it sends **HTML file** as “PDF alternative”:
  - Requires `sd.html_report` to exist.
  - If missing, sends Telegram message “No report to export. Please calculate first.” and returns `{skip:true}`.
- `IF PDF` checks `skip != true` before sending document.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram updates | — | 🔑 TOKEN | ## ⚙️ Telegram Credentials… / ## ✅ CHECKLIST… / ## 🧠 Main Router… |
| 🔑 TOKEN | set | Provide bot/API/Qdrant credentials | Telegram Trigger | Main | ## 🔐 API CREDENTIALS SETUP… |
| Main | code | Session/state machine; produce action | 🔑 TOKEN | Config | ## 🧠 Main Router… |
| Config | code | Localization + DB mapping | Main | Route | ## 🌐 Config… / ## 🌍 UI Messages… |
| Route | switch | Route by action | Config | Many | ## 🔀 Route Switch… |
| 📤 Lang Menu | httpRequest | Send language menu | Route(LANG) / Route(extra fallback) | — | ## 🌍 UI Messages… |
| Answer Lang CB | httpRequest | answerCallbackQuery for lang | Route(LANG_OK) | 📤 Lang OK |  |
| 📤 Lang OK | httpRequest | Confirm language + ask input | Answer Lang CB | — |  |
| Answer Photo CB | httpRequest | answerCallbackQuery for ask photo | Route(PHOTO) | 📤 Ask Photo |  |
| 📤 Ask Photo | httpRequest | Ask for photo/PDF/text | Answer Photo CB | — |  |
| 📤 Analyze Options | httpRequest | Show “add more / analyze now” | Route(ANALYZE_OPT), Route(PHOTO_ADDED) | — | ## 👁️ Block 4: Vision Analysis Pipeline… |
| Refine Analysis | code | Decide refine path | Route(REFINE), Route(ANALYZE) | IF Skip Refine | ## 👁️ Block 4: Vision Analysis Pipeline… |
| IF Skip Refine | if | Branch: ask photo vs proceed | Refine Analysis | 📤 Ask Photo Refine / Prep Photo Download |  |
| 📤 Ask Photo Refine | httpRequest | Ask photo when refine not possible | IF Skip Refine(true) | — |  |
| Prep Photo Download | code | Choose stored base64 vs download | IF Skip Refine(false) | IF Skip Download |  |
| IF Skip Download | if | Branch: download vs no download | Prep Photo Download | IF No Photos1 / Get File Path1 |  |
| IF No Photos1 | if | Branch: error vs use stored | IF Skip Download(true) | 📤 No Photos Msg1 / Use Stored Base64 |  |
| 📤 No Photos Msg1 | httpRequest | “Send photo first” message | IF No Photos1(true) | — |  |
| Use Stored Base64 | code | Use stored base64 photos | IF No Photos1(false) | Merge To Vision1 |  |
| Get File Path1 | httpRequest | Telegram getFile for photo | IF Skip Download(false) | Download Photo File1 |  |
| Download Photo File1 | httpRequest | Download photo binary | Get File Path1 | Convert To Base |  |
| Convert To Base | code | Convert photo to base64; store in session | Download Photo File1 | Merge To Vision1 |  |
| Merge To Vision1 | code | Merge path before vision call | Convert To Base / Use Stored Base64 | Prep Vision1 |  |
| Prep Vision1 | code | Build request for Gemini/OpenAI vision | Merge To Vision1 | Call Vision1 |  |
| Call Vision1 | httpRequest | Call vision API | Prep Vision1 | Merge Vision1 |  |
| Merge Vision1 | code | Normalize vision response | Call Vision1 | Parse AI |  |
| Parse AI | code | Parse JSON to works; save session | Merge Vision1 | 📊 Show Works |  |
| 📊 Show Works | code | Format works list + edit keyboard | Parse AI / Parse Text LLM / 🧹 Deduplicate & Merge | 📤 Send Works |  |
| 📤 Send Works | httpRequest | Send works list message | 📊 Show Works | — |  |
| Prep Text LLM | code | Build LLM request from text | Route(ANALYZE_TEXT) | Call Text LLM |  |
| Call Text LLM | httpRequest | Call text LLM | Prep Text LLM | Parse Text LLM |  |
| Parse Text LLM | code | Parse works array; save session | Call Text LLM | 📊 Show Works |  |
| 📄 PDF Download Prep | code | Prep pdfFileId/name | Route(PDF_PROCESS) | 📄 Get PDF Path | ## 📄 Block 5: PDF Floor Plan Processing… |
| 📄 Get PDF Path | httpRequest | Telegram getFile for PDF | 📄 PDF Download Prep | 📄 Download PDF |  |
| 📄 Download PDF | httpRequest | Download PDF binary | 📄 Get PDF Path | 📄 Split PDF Pages |  |
| 📄 Split PDF Pages | code | Store PDF base64; estimate up to 3 pages | 📄 Download PDF | 📝 Prep PDF Message |  |
| 📝 Prep PDF Message | code | Create “PDF received” message text | 📄 Split PDF Pages | 📤 PDF Received |  |
| 📤 PDF Received | httpRequest | Send PDF received message | 📝 Prep PDF Message | 📄 Prep Pages Loop |  |
| 📄 Prep Pages Loop | code | Create loop items for pages | 📤 PDF Received | 🔁 Loop PDF Pages |  |
| 🔁 Loop PDF Pages | splitInBatches | Iterate pages | 📄 Prep Pages Loop / 📦 Accumulate Pages | 👁️ Prep Vision PDF / 🧹 Deduplicate & Merge |  |
| 👁️ Prep Vision PDF | code | Build Gemini request with inline PDF | 🔁 Loop PDF Pages | 👁️ Call Vision PDF |  |
| 👁️ Call Vision PDF | httpRequest | Call Gemini PDF vision | 👁️ Prep Vision PDF | 🏠 Parse PDF Page |  |
| 🏠 Parse PDF Page | code | Parse rooms/works JSON | 👁️ Call Vision PDF | 📦 Accumulate Pages |  |
| 📦 Accumulate Pages | code | Accumulate rooms/works in static data | 🏠 Parse PDF Page | 🔁 Loop PDF Pages |  |
| 🧹 Deduplicate & Merge | code | Dedupe rooms/works; write session | 🔁 Loop PDF Pages | 📊 Show Works |  |
| Edit Menu (code) | code | Build qty edit keyboard | Route(EDIT_MENU) | 📤 Edit Menu | ## ✏️ Edit Menu… |
| 📤 Edit Menu | httpRequest | Send edit menu message | Edit Menu (code) | — |  |
| Works Updated | code | Re-render works list after edit | Route(WORKS_UPD) | 📤 Works Updated |  |
| 📤 Works Updated | httpRequest | Send updated works list | Works Updated | — |  |
| 📤 Ask New Work | httpRequest | Prompt user for new work text | Route(ADD_WORK) | — |  |
| 📤 Help | httpRequest | Send help message | Route(HELP) | — |  |
| 📤 Fallback | httpRequest | Fallback guidance | Route(extra) | — |  |
| Answer Calc CB | httpRequest | answerCallbackQuery for calc start | Route(CALC) | 📝 Prep Progress | ## 🔄 Block 6: Calculation Loop… |
| 📝 Prep Progress | code | Compose initial progress (FREE DEMO) | Answer Calc CB | 📤 Send Progress |  |
| 📤 Send Progress | httpRequest | Send progress message | 📝 Prep Progress | Save Progress ID |  |
| Save Progress ID | code | Store progress msg id; init trackers | 📤 Send Progress | Prep Works |  |
| Prep Works | code | Prepare loop items; init accumulator | Save Progress ID | Loop |  |
| Loop | splitInBatches | Iterate work items | Prep Works / Acc | 🧹 Prep Cleanup / 📝 Prep Work Msg |  |
| 📝 Prep Work Msg | code | Compose “searching…” msg per work | Loop | 🗑️ Delete Prev |  |
| 🗑️ Delete Prev | httpRequest | Delete previous work msg | 📝 Prep Work Msg | 📤 Send Work |  |
| 📤 Send Work | httpRequest | Send current work msg | 🗑️ Delete Prev | 💾 Save Work Msg |  |
| 💾 Save Work Msg | code | Store lastMsgId for edits | 📤 Send Work | 1️⃣ Prep Query |  |
| 1️⃣ Prep Query | code | Build transform prompt & inject keys | 💾 Save Work Msg | 1.5️⃣ LLM Transform |  |
| 1.5️⃣ LLM Transform | httpRequest | OpenAI query transform | 1️⃣ Prep Query | 2️⃣ Extract Transform |  |
| 2️⃣ Extract Transform | code | Combine original + transformed keywords | 1.5️⃣ LLM Transform | 3️⃣ OpenAI Embedding |  |
| 3️⃣ OpenAI Embedding | httpRequest | Create embedding vector | 2️⃣ Extract Transform | 4️⃣ Extract Embedding |  |
| 4️⃣ Extract Embedding | code | Extract embedding array | 3️⃣ OpenAI Embedding | 5️⃣ Qdrant Search |  |
| 5️⃣ Qdrant Search | httpRequest | Vector search in Qdrant | 4️⃣ Extract Embedding | 6️⃣ Prep Rerank |  |
| 6️⃣ Prep Rerank | code | Build rerank prompt from candidates | 5️⃣ Qdrant Search | 7️⃣ LLM Rerank |  |
| 7️⃣ LLM Rerank | httpRequest | OpenAI rerank candidates | 6️⃣ Prep Rerank | 8️⃣ Apply Rerank |  |
| 8️⃣ Apply Rerank | code | Combine scores; select best payload | 7️⃣ LLM Rerank | 9️⃣ Calculate |  |
| 9️⃣ Calculate | code | Compute uc/tc + resources breakdown | 8️⃣ Apply Rerank | 📊 Update Result |  |
| 📊 Update Result | code | Format per-item status text | 9️⃣ Calculate | 📤 Edit Result |  |
| 📤 Edit Result | httpRequest | Edit Telegram message with result | 📊 Update Result | Acc |  |
| Acc | code | Accumulate results; loop next | 📤 Edit Result | Loop |  |
| 🧹 Prep Cleanup | code | Prepare msg IDs to delete | Loop(done branch) | 🗑️ Delete Work Msg | ## 📊 Block 7: Aggregation & Reports… |
| 🗑️ Delete Work Msg | httpRequest | Delete last work msg | 🧹 Prep Cleanup | 🗑️ Delete Progress Msg |  |
| 🗑️ Delete Progress Msg | httpRequest | Delete progress msg | 🗑️ Delete Work Msg | Agg |  |
| Agg | code | Aggregate totals; store lastResults | 🗑️ Delete Progress Msg | Generate HTML |  |
| Generate HTML | code | Build interactive HTML report; store in sd | Agg | Final |  |
| Final | code | Build final Telegram summary message | Generate HTML | Prep HTML File / 📤 Final |  |
| 📤 Final | httpRequest | Send final message with buttons | Final | — |  |
| Prep HTML File | code | Package HTML as Telegram document | Final | 📤 Send HTML |  |
| 📤 Send HTML | telegram | Send HTML report file | Prep HTML File | — |  |
| View Details | code | Build detailed resources/scope message | Route(DETAILS) | 📤 Details | ## 📥 Block 8: Export Options… |
| 📤 Details | httpRequest | Send details message | View Details | — |  |
| Generate Excel | code | Create CSV binary from lastResults | Route(EXCEL) | 📤 Send Excel |  |
| 📤 Send Excel | telegram | Send CSV file | Generate Excel | — |  |
| Generate PDF | code | Package HTML report as “PDF” file | Route(PDF) | IF PDF |  |
| IF PDF | if | Skip send if no report | Generate PDF | 📤 Send PDF |  |
| 📤 Send PDF | telegram | Send HTML file as “PDF export” | IF PDF | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n. Set execution order to `v1` (Workflow settings).

2) **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Updates: `message`, `callback_query`
   - Credential: create **Telegram API** credential with bot token (from @BotFather).
   - Activate later to register webhook.

3) **Add Set node for secrets**
   - Node: **Set** named `🔑 TOKEN`
   - Mode: raw JSON output
   - Paste fields:
     - `bot_token`
     - `AI_PROVIDER`: `gemini` or `openai`
     - `GEMINI_API_KEY`
     - `OPENAI_API_KEY`
     - `QDRANT_URL`
     - `QDRANT_API_KEY`
   - Connect: `Telegram Trigger → 🔑 TOKEN`

4) **Add Main Router (Code)**
   - Node: **Code** named `Main`
   - Implement the session-based action selection as in the provided workflow:
     - Read update from `Telegram Trigger`
     - Maintain `sd.sess[cid]`
     - Output `{bot_token, chatId, action, lang, photos, description, pdfFileId, pdfFileName, callbackQueryId, ...}`
   - Connect: `🔑 TOKEN → Main`

5) **Add Localization (Code)**
   - Node: **Code** named `Config`
   - Add the `LANGS` dictionary (9 languages) with:
     - UI strings
     - `db` (Qdrant collection name)
     - `cur`, `sym`, `loc`, `region`, `search_lang`
   - Output `{...input, L, db, pdfFileId, pdfFileName}`
   - Connect: `Main → Config`

6) **Add Action Router (Switch)**
   - Node: **Switch** named `Route`
   - Switch on `{{$json.action}}`
   - Create outputs for:
     - `show_lang`, `lang_selected`, `ask_photo`, `show_analyze_options`, `photo_added`, `analyze`,
     - `show_edit_menu`, `works_updated`, `ask_new_work`, `start_calc`,
     - `show_help`, `view_details`, `export_excel`, `export_pdf`,
     - `process_pdf`, `analyze_text`, `refine_analysis`
   - Add fallback output `extra`.
   - Connect: `Config → Route`

7) **Implement Telegram UI senders (HTTP Request nodes)**
   - Create HTTP Request nodes to Telegram Bot API:
     - `📤 Lang Menu` → `sendMessage` with inline keyboard languages
     - `Answer Lang CB` → `answerCallbackQuery`
     - `📤 Lang OK` → `sendMessage`
     - `Answer Photo CB` → `answerCallbackQuery`
     - `📤 Ask Photo` → `sendMessage`
     - `📤 Analyze Options` → `sendMessage` with “add more/analyze/help”
     - `📤 Help` → `sendMessage`
     - `📤 Fallback` → `sendMessage`
   - Each uses URL form: `https://api.telegram.org/bot{{ $('🔑 TOKEN').first().json.bot_token }}/METHOD`

8) **Build Works display**
   - Add Code node `📊 Show Works` that reads works/rooms from input or session, builds message and inline keyboard.
   - Add HTTP node `📤 Send Works` calling `sendMessage` with `reply_markup.inline_keyboard`.
   - Connect analysis outputs to `📊 Show Works → 📤 Send Works`.

9) **Photo Vision pipeline**
   - Add nodes:
     - `Refine Analysis` (Code) → `IF Skip Refine` (If) → `📤 Ask Photo Refine` (HTTP)
     - `Prep Photo Download` (Code) → `IF Skip Download` (If)
     - `IF No Photos1` (If) → `📤 No Photos Msg1` (HTTP) / `Use Stored Base64` (Code)
     - `Get File Path1` (HTTP getFile) → `Download Photo File1` (HTTP download file as binary)
     - `Convert To Base` (Code) storing base64 into `sd.sess[cid].photos_base64`
     - `Merge To Vision1` (Code pass-through)
     - `Prep Vision1` (Code builds request for Gemini/OpenAI)
     - `Call Vision1` (HTTP call)
     - `Merge Vision1` (Code unify response)
     - `Parse AI` (Code parse JSON; store session works)
     - then `📊 Show Works`
   - Connect the `Route(ANALYZE)` and `Route(REFINE)` outputs to start at `Refine Analysis`.
   - Connect `Route(ANALYZE_OPT)` and `Route(PHOTO_ADDED)` to `📤 Analyze Options`.

10) **Text analysis pipeline**
   - Add `Prep Text LLM` (Code) → `Call Text LLM` (HTTP) → `Parse Text LLM` (Code) → `📊 Show Works`.
   - Connect `Route(ANALYZE_TEXT)` to `Prep Text LLM`.

11) **PDF pipeline**
   - Add:
     - `📄 PDF Download Prep` (Code)
     - `📄 Get PDF Path` (HTTP getFile)
     - `📄 Download PDF` (HTTP binary)
     - `📄 Split PDF Pages` (Code store base64, estimate pages ≤3)
     - `📝 Prep PDF Message` (Code) → `📤 PDF Received` (HTTP)
     - `📄 Prep Pages Loop` (Code) → `🔁 Loop PDF Pages` (SplitInBatches)
     - `👁️ Prep Vision PDF` (Code Gemini request with inline PDF)
     - `👁️ Call Vision PDF` (HTTP, 120s timeout)
     - `🏠 Parse PDF Page` (Code parse JSON)
     - `📦 Accumulate Pages` (Code)
     - loop back to `🔁 Loop PDF Pages` until done
     - done branch: `🧹 Deduplicate & Merge` (Code) → `📊 Show Works`
   - Connect `Route(PDF_PROCESS)` to `📄 PDF Download Prep`.

12) **Edit menu**
   - Add `Edit Menu` (Code) → `📤 Edit Menu` (HTTP sendMessage with keyboard).
   - Add `Works Updated` (Code) → `📤 Works Updated` (HTTP).
   - Add `📤 Ask New Work` (HTTP).
   - Connect `Route(EDIT_MENU)` → `Edit Menu`
   - Connect `Route(WORKS_UPD)` → `Works Updated`
   - Connect `Route(ADD_WORK)` → `📤 Ask New Work`

13) **Calculation loop**
   - Add nodes:
     - `Answer Calc CB` (HTTP answerCallbackQuery)
     - `📝 Prep Progress` (Code) → `📤 Send Progress` (HTTP) → `Save Progress ID` (Code)
     - `Prep Works` (Code) → `Loop` (SplitInBatches)
     - Per item: `📝 Prep Work Msg` (Code) → `🗑️ Delete Prev` (HTTP) → `📤 Send Work` (HTTP) → `💾 Save Work Msg` (Code)
     - Search chain: `1️⃣ Prep Query` (Code) → `1.5️⃣ LLM Transform` (HTTP OpenAI) → `2️⃣ Extract Transform` (Code) → `3️⃣ OpenAI Embedding` (HTTP) → `4️⃣ Extract Embedding` (Code) → `5️⃣ Qdrant Search` (HTTP) → `6️⃣ Prep Rerank` (Code) → `7️⃣ LLM Rerank` (HTTP OpenAI) → `8️⃣ Apply Rerank` (Code) → `9️⃣ Calculate` (Code)
     - Progress: `📊 Update Result` (Code) → `📤 Edit Result` (HTTP editMessageText) → `Acc` (Code) → back to `Loop`
   - After loop completes:
     - `🧹 Prep Cleanup` (Code) → `🗑️ Delete Work Msg` (HTTP) → `🗑️ Delete Progress Msg` (HTTP) → `Agg` (Code)

14) **Reports**
   - Add:
     - `Generate HTML` (Code) that stores `sd.html_report`
     - `Final` (Code) builds final Telegram message
     - `📤 Final` (HTTP sendMessage with export buttons)
     - `Prep HTML File` (Code) creates binary html
     - `📤 Send HTML` (Telegram node: sendDocument, binary property `html`)
   - Connect: `Agg → Generate HTML → Final → (📤 Final and Prep HTML File) → 📤 Send HTML`

15) **Exports & Details**
   - Add:
     - `View Details` (Code) → `📤 Details` (HTTP)
     - `Generate Excel` (Code) → `📤 Send Excel` (Telegram sendDocument, binary property `excel`)
     - `Generate PDF` (Code) → `IF PDF` (If skip) → `📤 Send PDF` (Telegram sendDocument, binary property `pdf`)
   - Connect `Route(DETAILS)`, `Route(EXCEL)`, `Route(PDF)` accordingly.

16) **Credentials**
   - Telegram:
     - Telegram Trigger credential
     - Telegram “sendDocument” nodes (`📤 Send HTML`, `📤 Send Excel`, `📤 Send PDF`) require Telegram credential as well.
   - OpenAI:
     - This workflow uses raw HTTP with bearer headers from `🔑 TOKEN` (no n8n OpenAI credential node needed).
   - Gemini:
     - Also raw HTTP with API key in URL (from `🔑 TOKEN`).
   - Qdrant:
     - URL and optional `api-key` header (from `🔑 TOKEN`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “If you find our tools helpful, please consider starring our repository…” | https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR |
| DDC CWICR database and project | https://DataDrivenConstruction.io |
| Qdrant setup guidance (collections per language, upload datasets, configure URL/key) | Mentioned in “Qdrant Info” sticky note |
| FREE DEMO behavior: calculation limited to 5 work items | Implemented in `📝 Prep Progress` and `Prep Works` |
| “PDF export” currently sends HTML file, not a true PDF | Implemented in `Generate PDF` |
| Some callback buttons are not handled (`back_to_lang`, `update_categories`, `process_voice`) | Would route to `Route` fallback output |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.