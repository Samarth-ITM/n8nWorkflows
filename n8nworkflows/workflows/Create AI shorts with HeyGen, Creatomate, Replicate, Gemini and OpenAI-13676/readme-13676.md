Create AI shorts with HeyGen, Creatomate, Replicate, Gemini and OpenAI

https://n8nworkflows.xyz/workflows/create-ai-shorts-with-heygen--creatomate--replicate--gemini-and-openai-13676


# Create AI shorts with HeyGen, Creatomate, Replicate, Gemini and OpenAI

## 1. Workflow Overview

**Workflow name:** *Shorts Creation v10 - Telegram Filming*  
**Purpose:** End-to-end automated production of **3 vertical short videos** per source video, including: concept ideation, AI avatar narration (HeyGen), optional AI flash B-roll (Replicate), AI-directed storyboard and layout plan (Gemini), final composition/render (Creatomate), upload to Google Drive, social copy generation (OpenAI), lead-magnet document creation (Google Docs), and tracking in Google Sheets.

**Primary use case:** This workflow is designed to run as a **sub-workflow**. A parent workflow provides pre-analyzed inputs: transcript structure, key moments / B-roll timestamps, overview metadata, and links.

### 1.1 Logical Blocks (high-level)

1. **Block A — Input Reception & Normalization (Stage 1)**  
   Receives the parent workflow payload, extracts sections/snippets and available screen B-roll clips, and composes prompts for the AI ideation agent.

2. **Block B — Concept Ideation (Stage 2)**  
   Gemini Agent generates exactly **3 structured concepts** (pain/curiosity/transformation), each with storyboard segments, flash B-roll prompts, and a shared global CTA/deliverable prompt.

3. **Block C — Per-Concept Loop Orchestration (Stage 2 → Stage 5)**  
   For each concept:  
   - Generate lead magnet Google Doc  
   - Generate 3 “flash” AI B-roll clips via Replicate and aggregate results  
   - Generate a HeyGen avatar video and poll until ready  
   - Run AI Video Director (Gemini) to convert concept + assets into a beat-by-beat storyboard compatible with Creatomate effects rules  
   - Build Creatomate render payload and render/poll/download

4. **Block D — Delivery & Tracking**  
   Uploads the final MP4 to Google Drive, generates social copy via OpenAI, and appends rows into Google Sheets (video metadata + captions + links + magnet URL).

---

## 2. Block-by-Block Analysis

### Block A — Input Reception & Normalization (Stage 1)

**Overview:**  
Receives the parent workflow payload and restructures it into a single “shorts-ready” object: overview, sections, best clips, and enriched screen B-roll clip references. Then builds system/user prompts used by the concept ideation agent.

**Nodes involved:**  
- Shorts Trigger  
- Extract Snippets  
- Edit Fields  

#### Node: **Shorts Trigger**
- **Type / Role:** `Execute Workflow Trigger` — sub-workflow entry point (passthrough input).
- **Config highlights:** `inputSource: passthrough` (takes parent payload as-is).
- **Inputs:** Parent workflow via Execute Workflow.
- **Outputs:** JSON payload containing `source`, `overview`, `transcript`, `key_moments`, `visual_analysis`, `links`, etc.
- **Edge cases / failures:**
  - Missing expected keys (e.g., `transcript.sections`, `source.video_url`) will cause downstream expression or logic issues.
  - Large payload size may increase execution memory/time.
- **Sub-workflow reference:** This is the entrypoint; it expects a parent workflow call.

#### Node: **Extract Snippets**
- **Type / Role:** `Code` — transforms/normalizes the input structure into fields the AI needs.
- **Key configuration choices:**
  - Creates `key_moments` derived from transcript sections with `hook_potential`.
  - Creates `best_clips` from `best_sections_for_shorts`. **Note:** In pinned sample data, `best_sections_for_shorts` is an array of objects, but the code assumes indices; ensure parent workflow matches what this code expects.
  - Enriches `broll_clips` (from `input.key_moments`) with section context.
  - Outputs a compact object including: video metadata, overview fields, `sections`, `broll_clips`, and `links`.
- **Inputs:** `Shorts Trigger`
- **Outputs:** One item with the normalized structure used by prompts and video director.
- **Edge cases / failures:**
  - If `best_sections_for_shorts` is not numeric indices, `bestClips` may be empty or wrong.
  - If `transcript.sections` missing, many derived arrays become empty.
  - If `input.key_moments` missing or empty, B-roll will be limited to screen or AI-only depending on director decisions.

#### Node: **Edit Fields**
- **Type / Role:** `Set` — builds the **System Prompt**, **User Prompt**, and **Examples** JSON string for ideation.
- **Key configuration choices:**
  - System prompt is extremely detailed (speech rules, segment duration constraints, B-roll rules, no-null requirements, etc.).
  - User prompt injects:
    - overview fields
    - broll clip list (with heuristic “good for topics about …”)
    - section summaries
    - explicit instructions: narration must not refer to B-roll.
- **Inputs:** `Extract Snippets`
- **Outputs:** JSON with fields: `System Prompt`, `User Prompt`, `Examples`.
- **Edge cases / failures:**
  - Expression failures if arrays are missing (e.g., `.map(...)` on undefined). The node uses `$json.broll_clips && $json.broll_clips.length > 0 ? ...` for that section, but other fields like `$json.sections.map(...)` assume sections exists.

---

### Block B — Concept Ideation (Stage 2)

**Overview:**  
Gemini Agent generates **exactly 3** short concepts with strict structured output, including storyboard segments, flash B-roll plan, full script, hashtags, and one shared global CTA/deliverable.

**Nodes involved:**  
- Poll Short Form Concept Ideator  
- Google Gemini Chat Model  
- Structured Output Parser  
- Google Gemini Chat Model1  
- Split Out  
- Loop Through Concepts  

#### Node: **Google Gemini Chat Model**
- **Type / Role:** `lmChatGoogleGemini` — Gemini model for the ideation agent.
- **Config highlights:** `modelName: models/gemini-pro-latest`
- **Outputs:** Used by the Agent node via LangChain connector.
- **Edge cases:** API key limits, quota, blocked content, latency.

#### Node: **Structured Output Parser**
- **Type / Role:** `outputParserStructured` — enforces JSON schema for ideation output.
- **Config highlights:**
  - `schemaType: manual`, `autoFix: true`
  - Requires:
    - `concepts` array (exactly 3 implied by description)
    - per concept: storyboard (12–20 segments), flash_broll (exactly 3), full_script, etc.
    - `global_cta` object
  - `additionalProperties: false` on most objects to keep output clean.
- **Connections:** Connected to the ideation agent as output parser.
- **Edge cases:**
  - If the model returns invalid JSON or violates constraints, autoFix will attempt repair; may still fail if output is too malformed.

#### Node: **Google Gemini Chat Model1**
- **Type / Role:** Additional Gemini model node wired to the Structured Output Parser (LangChain requirement).
- **Config highlights:** default options.
- **Edge cases:** same as Gemini nodes.

#### Node: **Poll Short Form Concept Ideator**
- **Type / Role:** `agent` — main concept ideation agent.
- **Config highlights:**
  - `text`: uses `User Prompt`
  - `systemMessage`: uses `System Prompt` plus injected “Examples”
  - `hasOutputParser: true` (Structured Output Parser attached)
- **Inputs:** `Edit Fields`
- **Outputs:** One item containing `output.concepts[]` and `output.global_cta`.
- **Edge cases / failures:**
  - Token/context size: prompts are very large (examples + long system prompt + sections + broll list).
  - Strict segment timing rules may cause schema violations if model can’t comply.

#### Node: **Split Out**
- **Type / Role:** `Split Out` — splits `output.concepts` so each concept becomes its own item.
- **Config highlights:** `fieldToSplitOut: output.concepts`
- **Inputs:** `Poll Short Form Concept Ideator`
- **Outputs:** Items = 3 concepts.
- **Edge cases:** If `output.concepts` missing or not an array, node fails.

#### Node: **Loop Through Concepts**
- **Type / Role:** `Split In Batches` — iterates over concept items.
- **Config highlights:** `reset: false` (keeps cursor state; expects loop-back connections).
- **Inputs:** `Split Out`
- **Outputs:**
  - Output 0: loop “done/empty” (unused)
  - Output 1: per-item branch feeding doc/B-roll and effects library generation, and later loop-back after sheet append.
- **Edge cases:**
  - If loop-back path fails before returning to this node, remaining concepts won’t process.
  - If parallel branches return at different times, you must ensure the merge logic still aligns (see Block C).

---

### Block C — Per-Concept Production (Docs + B-Roll + HeyGen + Video Director + Creatomate)

**Overview:**  
For each concept, the workflow creates the lead magnet document, generates flash AI B-roll clips, generates the avatar video, then uses Gemini to plan the final video composition which is rendered by Creatomate.

**Nodes involved (grouped):**

**C1: Lead Magnet Doc creation**
- Document Generator  
- Google Gemini Chat Model2  
- Convert Document to HTML  
- Create Google Doc  
- Share file  
- Append row in sheet1  

**C2: Flash AI B-roll generation**
- Split Out1  
- Get AI B-Roll  
- Extract Flash B-Roll Result1  
- Aggregate Flash B-Roll  

**C3: Effects library injection**
- Creatomate Effects Library  

**C4: Merge for HeyGen kickoff**
- Merge  
- HeyGen - Generate Full Avatar  
- Set Avatar Video ID  
- Wait for Avatar  
- Poll Avatar Status  
- Avatar Done?  
- Set Avatar URL  

**C5: AI Video Director → Creatomate**
- AI Video Director  
- Google Gemini Chat Model3  
- Structured Output Parser1  
- Google Gemini Chat Model4  
- Creatomate Template Builder Code1  
- Creatomate - Render  
- Set Render ID  
- Wait for Render  
- Poll Render Status  
- Render Done?  
- Download Rendered Video  

#### C1 Node: **Document Generator**
- **Type / Role:** `agent` — generates the deliverable document content in Markdown.
- **Config highlights:**
  - `text`: uses `output.global_cta.deliverable_prompt` (from ideator).
  - Strong system message requiring Markdown output with specific callout syntax.
- **Inputs:** `Poll Short Form Concept Ideator` (not the split concept item; uses ideator output directly).
- **Outputs:** Markdown in `output`.
- **Edge cases:**
  - If ideator output lacks `global_cta.deliverable_prompt`, this node fails.
  - Output might not start with `# Title`, which affects later title extraction.

#### C1 Node: **Google Gemini Chat Model2**
- **Role:** Gemini model for Document Generator (models/gemini-pro-latest).

#### C1 Node: **Convert Document to HTML**
- **Type / Role:** `Code` — converts Markdown output into styled HTML and builds a **multipart upload body** for Google Drive “create Google Doc from HTML”.
- **Key configuration choices:**
  - Hardcoded Google Drive folder ID: `FOLDER_ID = '1RYvQEGfapC6g665JUS-XpdaEHWPpaB9l'`
  - Extracts title from first Markdown `# ...` line.
  - Implements a basic Markdown-to-HTML conversion (headings, lists, emphasis, code fences).
  - Builds multipart body with boundary `docboundary`.
- **Inputs:** `Document Generator`
- **Outputs:** `{ document_name, rawData }`
- **Edge cases:**
  - Markdown conversion is simplistic: list parsing uses a custom `<uli>` intermediate; malformed markdown may generate unexpected HTML.
  - Boundary string must match the HTTP node’s content type.
  - Very large documents may exceed API payload limits.

#### C1 Node: **Create Google Doc**
- **Type / Role:** `HTTP Request` — uses Google Drive “multipart upload” endpoint to create a Google Doc.
- **Config highlights:**
  - `POST https://www.googleapis.com/upload/drive/v3/files`
  - Query: `uploadType=multipart`, `supportsAllDrives=true`
  - `rawContentType: multipart/related; boundary=docboundary`
  - OAuth2 credential: Google Drive.
- **Inputs:** `Convert Document to HTML`
- **Outputs:** Google Drive file object, including `id`.
- **Edge cases:**
  - OAuth scopes must allow Drive file creation.
  - Boundary mismatch breaks upload.
  - Some HTML features may be stripped by Google Docs import.

#### C1 Node: **Share file**
- **Type / Role:** `Google Drive` — shares created doc publicly.
- **Config highlights:** permissions: `role=reader`, `type=anyone`, `allowFileDiscovery=true`
- **Inputs:** `Create Google Doc`
- **Outputs:** Share result.
- **Edge cases:** domain policies might block public sharing.

#### C1 Node: **Append row in sheet1**
- **Type / Role:** `Google Sheets` — appends Magnet URL.
- **Config highlights:**
  - Writes `Magnet URL = https://docs.google.com/document/d/{{ $json.id }}/edit`
  - Uses the same spreadsheet/sheet as the main tracker.
- **Inputs:** `Share file`
- **Outputs:** append result.
- **Edge cases:** This appends a **new row**, not updating the row created later/earlier; you may end up with disjoint rows unless the sheet is designed for it.

---

#### C2 Node: **Split Out1**
- **Type / Role:** `Split Out` — splits `flash_broll` array for per-clip generation.
- **Config highlights:** `fieldToSplitOut: flash_broll`
- **Inputs:** `Loop Through Concepts` (output 1)
- **Outputs:** 3 items (flash clips).
- **Edge cases:** If concept has missing/invalid `flash_broll`, generation stops.

#### C2 Node: **Get AI B-Roll**
- **Type / Role:** `HTTP Request` — calls Replicate to generate a 2-second vertical clip.
- **Config highlights:**
  - `POST https://api.replicate.com/v1/models/bytedance/seedance-1-lite/predictions`
  - Input fixed: `fps=24`, `duration=2`, `resolution=720p`, `aspect_ratio=9:16`, `camera_fixed=false`
  - Uses `webhook: {{$execution.resumeUrl}}` with `webhook_events_filter: ["completed"]`
  - Header includes `Prefer: wait` and timeout up to 300s.
  - Auth: HTTP Header Auth (Replicate token).
- **Inputs:** `Split Out1` (each flash prompt object)
- **Outputs:** Replicate prediction result (webhook completion).
- **Edge cases:**
  - Replicate queue delays; may exceed timeouts depending on plan.
  - Output schema changes could break downstream mapping.
  - `Prefer: wait` + webhook usage can be redundant depending on Replicate behavior.

#### C2 Node: **Extract Flash B-Roll Result1**
- **Type / Role:** `Set` — normalizes Replicate output and joins it with flash metadata.
- **Key expressions:**
  - copies `flash_number`, `prompt`, `duration_seconds`, `insert_at_timestamp`, etc. from `Split Out1`
  - `video_url = {{$json.output}}`, `status = {{$json.status}}`, `replicate_id = {{$json.id}}`
- **Inputs:** `Get AI B-Roll`
- **Outputs:** standardized object per clip.
- **Edge cases:** Replicate output field names may differ (`output` can be array/url depending on model).

#### C2 Node: **Aggregate Flash B-Roll**
- **Type / Role:** `Aggregate` — aggregates all flash clip outputs into one item.
- **Config highlights:** `aggregateAllItemData` into `flash_broll_generated`
- **Inputs:** `Extract Flash B-Roll Result1`
- **Outputs:** one item containing `flash_broll_generated[]`
- **Edge cases:** If any clip fails and doesn’t pass through, aggregation may produce fewer than 3 results.

---

#### C3 Node: **Creatomate Effects Library**
- **Type / Role:** `Code` — returns a library of layout IDs, transitions, motions, avatar animations, text styles, and audio tracks to constrain AI Director and Template Builder.
- **Config highlights:**
  - Provides layout IDs like `AVATAR_FULLSCREEN`, `SPLIT_50_50`, `SCREEN_FULLSCREEN`, etc.
  - Provides transitions including overlay types (FLASH, RGB_GLITCH, etc.) and non-overlay transitions (scale/slide/fade).
  - Provides guidelines that the AI director is instructed to follow.
- **Inputs:** `Loop Through Concepts` (output 1)
- **Outputs:** library object.
- **Edge cases:** If AI Director uses IDs not present here, downstream template builder may fallback or fail.

---

#### C4 Node: **Merge**
- **Type / Role:** `Merge` (combine by position) — combines the two parallel streams required before HeyGen generation:
  - Stream A: effects library → merge input 1
  - Stream B: aggregated flash B-roll → merge input 0
- **Config highlights:** `mode: combine`, `combineBy: position`
- **Inputs:** `Aggregate Flash B-Roll` (main index 0), `Creatomate Effects Library` (main index 1)
- **Outputs:** combined items, forwarded to HeyGen generation.
- **Edge cases:** If the two branches produce mismatched item counts or timing, `combineByPosition` can misalign or drop data. In this workflow, both are concept-scoped but one is aggregated and one is not; it relies on consistent per-concept execution.

#### C4 Node: **HeyGen - Generate Full Avatar**
- **Type / Role:** `HTTP Request` — generates an avatar narration video in HeyGen.
- **Config highlights:**
  - `POST https://api.heygen.com/v2/video/generate`
  - Body includes:
    - `avatar_id` selected from a hardcoded list using `concept_number - 1`
    - `input_text`: `full_script` converted to JSON string
    - voice settings (voice_id, stability, similarity_boost, etc.)
    - `dimension`: **note**: the payload sets `{ width: 1920, height: 1080 }` which is landscape; sticky note says vertical 1080×1920. If vertical is required, swap.
  - Timeout: 60s
  - Auth: HTTP Header Auth.
- **Inputs:** `Merge` result (but the request actually uses data from `Loop Through Concepts` via expressions).
- **Outputs:** returns `data.video_id`.
- **Edge cases:**
  - HeyGen API version mismatch: uses `v2/video/generate` but status uses `v1/video_status.get`.
  - Avatar ID list must exist in HeyGen account.
  - Wrong dimension may produce wrong orientation.

#### C4 Node: **Set Avatar Video ID**
- **Type / Role:** `Set` — stores `avatar_video_id = $json.data.video_id`.
- **Inputs:** HeyGen generate response.
- **Outputs:** item with `avatar_video_id`.

#### C4 Node: **Wait for Avatar**
- **Type / Role:** `Wait` — delays 45 seconds before polling HeyGen.
- **Inputs:** `Set Avatar Video ID` OR `Avatar Done?` false branch loop.
- **Outputs:** triggers `Poll Avatar Status`.

#### C4 Node: **Poll Avatar Status**
- **Type / Role:** `HTTP Request` — checks HeyGen status.
- **Config highlights:** `GET https://api.heygen.com/v1/video_status.get?video_id={{avatar_video_id}}`
- **Inputs:** `Wait for Avatar`
- **Outputs:** status payload.
- **Edge cases:** transient “processing” states; API rate limits.

#### C4 Node: **Avatar Done?**
- **Type / Role:** `If` — checks `{{$json.data.status}} == "completed"`.
- **True path:** Set Avatar URL  
- **False path:** Wait and repoll
- **Edge cases:** other terminal states (failed, canceled) are not handled explicitly; will loop forever unless status becomes “completed”.

#### C4 Node: **Set Avatar URL**
- **Type / Role:** `Set` — stores:
  - `avatar_video_url = $json.data.video_url`
  - `avatar_duration = $json.data.duration`
- **Inputs:** Avatar Done? true.
- **Outputs:** used by AI Video Director and Creatomate builder.
- **Edge cases:** missing duration; downstream director requires exact duration matching.

---

#### C5 Node: **AI Video Director**
- **Type / Role:** `agent` — converts the concept into a **production-ready beat storyboard** using effects library + available B-roll assets.
- **Config highlights:**
  - Input text includes:
    - avatar duration (must be matched exactly)
    - concept metadata, full script, current storyboard
    - available screen-share broll clips from `Extract Snippets`
    - available AI B-roll (flash clips) filtered by `status === 'succeeded'`
    - full effects library JSON + guidelines
  - Has structured output parser attached (`Structured Output Parser1`).
  - Retry enabled (`maxTries: 3`).
- **Inputs:** `Set Avatar URL`
- **Outputs:** JSON with `enhanced_storyboard[]`, audio track selection, etc.
- **Edge cases / failures:**
  - The system prompt mentions a transition `"intro"` for first beat, but the effects library does **not** define an `"intro"` transition ID; if the director outputs `"intro"`, template builder may not handle it (it will treat unknown transitions as none).
  - Director constraints require: first beat split 50/50 with screen B-roll; last beat avatar-only. If not followed, video can render with black screens.

#### C5 Node: **Google Gemini Chat Model3 / Model4**
- **Role:** Gemini models backing the AI Video Director and its output parser.

#### C5 Node: **Structured Output Parser1**
- **Type / Role:** `outputParserStructured` — validates director output.
- **Config highlights:** requires:
  - `total_duration_seconds` equals avatar duration (enforced by prompts, not by schema math)
  - each beat has strict required fields
  - `duration_seconds` must be <= 4 per instructions (schema doesn’t enforce max)
- **Edge cases:** schema mismatch, missing fields, wrong IDs.

#### C5 Node: **Creatomate Template Builder Code1**
- **Type / Role:** `Code` — constructs Creatomate render JSON (elements array) combining:
  - director storyboard
  - avatar video URL and duration
  - flash AI B-roll clip URLs mapping by `flash_number`
  - effects library layouts/transitions/motions/audio tracks
  - source screen recording video URL from trigger (`source.video_url`)
- **Key configuration choices:**
  - Sets output to vertical 1080×1920, 30 fps.
  - Adds:
    - background music track from R2 CDN
    - “avatar-master” hidden video track for transcript captions sync
    - stepped flash intro overlays (workaround for Creatomate fade-out limitations)
    - auto-captions using transcript_source = `avatar-master`
  - Uses storyboard beats to:
    - trim avatar segments sequentially
    - place screen B-roll (screen recording) or AI B-roll (flash clips) depending on `broll_type`
    - apply transitions: overlay vs animation transitions
- **Inputs (via expressions):**
  - AI Video Director, Loop Through Concepts, Set Avatar URL, Aggregate Flash B-Roll, Creatomate Effects Library, Shorts Trigger
- **Outputs:** Creatomate render payload JSON.
- **Edge cases / failures:**
  - If AI director outputs unknown layout/transition/motion IDs, builder falls back to defaults or skips transitions.
  - If `broll_type` is set incorrectly (e.g., `none` with a screen layout), output may show black frames.
  - If AI B-roll URLs missing for requested indices, those beats may have no valid source.

#### C5 Node: **Creatomate - Render**
- **Type / Role:** `HTTP Request` — submits render to Creatomate.
- **Config highlights:** `POST https://api.creatomate.com/v2/renders`, body is the entire JSON from builder.
- **Timeout:** 120s.
- **Auth:** HTTP Header Auth (Creatomate API key).
- **Outputs:** render job object (sometimes array; handled downstream).

#### C5 Node: **Set Render ID**
- **Type / Role:** `Set` — `render_id = Array.isArray($json) ? $json[0].id : $json.id`
- **Inputs:** Creatomate render response.
- **Outputs:** render_id for polling.

#### C5 Node: **Wait for Render**
- **Type / Role:** `Wait` — waits 30 seconds between polls.

#### C5 Node: **Poll Render Status**
- **Type / Role:** `HTTP Request` — `GET https://api.creatomate.com/v2/renders/{{render_id}}`
- **Config highlights:** retry enabled with 5s wait.
- **Outputs:** render status payload with `status`, `url`.
- **Edge cases:** rate limits, transient 5xx, long renders.

#### C5 Node: **Render Done?**
- **Type / Role:** `If` — checks `{{$json.status}} == "succeeded"`.
- **True path:** Download Rendered Video  
- **False path:** Wait and repoll
- **Edge cases:** failure states are not handled; may loop indefinitely.

#### C5 Node: **Download Rendered Video**
- **Type / Role:** `HTTP Request` — downloads the MP4 as binary.
- **Config highlights:** response format is `file`.
- **Inputs:** Render Done? true.
- **Outputs:** binary file for Google Drive upload.

---

### Block D — Upload, Social Copy, Tracking (Stage 5)

**Overview:**  
Uploads the rendered MP4 to Google Drive, generates platform-optimized copy, and appends a tracking row.

**Nodes involved:**  
- Upload to Google Drive  
- Social Media Copyrighter  
- Append row in sheet  

#### Node: **Upload to Google Drive**
- **Type / Role:** `Google Drive` — uploads the rendered MP4.
- **Config highlights:**
  - Filename: `{{ AI Video Director title }}_{{ concept_index }}.mp4`
  - FolderId: fixed ID `1KnH7S9ltO1VMhahtNQcKLkUnFwekbOqL`
- **Inputs:** `Download Rendered Video`
- **Outputs:** file metadata including `webViewLink`.
- **Edge cases:** missing permissions, folder ID wrong, file size limits, rate limits.

#### Node: **Social Media Copyrighter**
- **Type / Role:** `OpenAI` (LangChain) — generates YouTube/Instagram/TikTok captions and hashtag sets.
- **Config highlights:**
  - Model: `gpt-5.2`
  - Output enforced via JSON Schema (title, descriptions, IG short/long, TikTok caption, hashtags arrays, CTA object).
  - System prompt forces CTA-first formatting.
- **Inputs:** `Upload to Google Drive` (but uses many expressions from other nodes).
- **Outputs:** structured JSON in `output[0].content[0].text...`
- **Edge cases:** schema mismatch, model availability, content policy filters, long prompt context.

#### Node: **Append row in sheet**
- **Type / Role:** `Google Sheets` — writes the tracking row for the short.
- **Config highlights:**
  - Status = "Uploaded"
  - Includes: Created Date, titles/captions, hashtags, drive link, source video info.
  - `Created Date` uses `DateTime.fromSeconds($json.created_at)` — depends on `created_at` existing in the OpenAI node output payload.
- **Inputs:** `Social Media Copyrighter`
- **Outputs:** append result.
- **Loop-back:** connects back to `Loop Through Concepts` to process the next concept.
- **Edge cases:**
  - If `created_at` missing or not epoch seconds, date formatting breaks.
  - Appending rows creates separate entries from magnet URL rows (Append row in sheet1) unless sheet is designed to reconcile.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shorts Trigger | Execute Workflow Trigger | Entry point (sub-workflow trigger) | — | Extract Snippets | 🟢 Stage 1: Trigger & Extract… |
| Extract Snippets | Code | Normalize transcript/sections/B-roll for downstream AI | Shorts Trigger | Edit Fields | 🟢 Stage 1: Trigger & Extract… |
| Edit Fields | Set | Build System Prompt / User Prompt / Examples for ideation | Extract Snippets | Poll Short Form Concept Ideator | 🟢 Stage 1: Trigger & Extract… |
| Google Gemini Chat Model | Google Gemini Chat Model | LLM for concept ideation | — | Poll Short Form Concept Ideator (ai_languageModel) | 🟢 Stage 2: AI Concept Ideation… |
| Google Gemini Chat Model1 | Google Gemini Chat Model | LLM for structured parser support | — | Structured Output Parser (ai_languageModel) | 🟢 Stage 2: AI Concept Ideation… |
| Structured Output Parser | Structured Output Parser | Enforce ideation JSON schema | — | Poll Short Form Concept Ideator (ai_outputParser) | 🟢 Stage 2: AI Concept Ideation… |
| Poll Short Form Concept Ideator | Agent (Gemini) | Generate 3 concepts + global CTA | Edit Fields | Split Out; Document Generator | 🟢 Stage 2: AI Concept Ideation… |
| Split Out | Split Out | Split concepts into items | Poll Short Form Concept Ideator | Loop Through Concepts | 🟢 Stage 2: AI Concept Ideation… |
| Loop Through Concepts | Split In Batches | Iterate each concept item | Split Out; Append row in sheet (loopback) | Split Out1; Creatomate Effects Library | 🟢 Stage 2: AI Concept Ideation… |
| Document Generator | Agent (Gemini) | Generate lead magnet Markdown | Poll Short Form Concept Ideator | Convert Document to HTML | 🟢 Stage 2b: Documentation & B-Roll… |
| Google Gemini Chat Model2 | Google Gemini Chat Model | LLM for document generation | — | Document Generator (ai_languageModel) | 🟢 Stage 2b: Documentation & B-Roll… |
| Convert Document to HTML | Code | Convert Markdown to HTML + multipart upload body | Document Generator | Create Google Doc | 🟢 Stage 2b: Documentation & B-Roll… |
| Create Google Doc | HTTP Request | Create Google Doc from HTML (Drive multipart upload) | Convert Document to HTML | Share file | 🟢 Stage 2b: Documentation & B-Roll… |
| Share file | Google Drive | Make Doc public | Create Google Doc | Append row in sheet1 | 🟢 Stage 2b: Documentation & B-Roll… |
| Append row in sheet1 | Google Sheets | Append Magnet URL row | Share file | — | 🟢 Stage 2b: Documentation & B-Roll… |
| Split Out1 | Split Out | Split flash_broll items (3) | Loop Through Concepts | Get AI B-Roll | 🟢 Stage 2b: Documentation & B-Roll… |
| Get AI B-Roll | HTTP Request | Replicate Seedance 1 Lite video generation | Split Out1 | Extract Flash B-Roll Result1 | 🟢 Stage 2b: Documentation & B-Roll… |
| Extract Flash B-Roll Result1 | Set | Normalize Replicate result + flash metadata | Get AI B-Roll | Aggregate Flash B-Roll | 🟢 Stage 2b: Documentation & B-Roll… |
| Aggregate Flash B-Roll | Aggregate | Collect flash clips into flash_broll_generated | Extract Flash B-Roll Result1 | Merge | 🟢 Stage 2b: Documentation & B-Roll… |
| Creatomate Effects Library | Code | Provide layouts/transitions/audio/motions constraints | Loop Through Concepts | Merge | 🟢 Stage 4: AI Video Director & Creatomate Render… |
| Merge | Merge | Combine effects library + flash b-roll aggregation | Aggregate Flash B-Roll; Creatomate Effects Library | HeyGen - Generate Full Avatar | 🟢 Stage 3: HeyGen Avatar Generation… |
| HeyGen - Generate Full Avatar | HTTP Request | Generate avatar narration video | Merge | Set Avatar Video ID | 🟢 Stage 3: HeyGen Avatar Generation… |
| Set Avatar Video ID | Set | Store HeyGen video_id | HeyGen - Generate Full Avatar | Wait for Avatar | 🟢 Stage 3: HeyGen Avatar Generation… |
| Wait for Avatar | Wait | Delay before polling HeyGen | Set Avatar Video ID; Avatar Done? (false) | Poll Avatar Status | 🟢 Stage 3: HeyGen Avatar Generation… |
| Poll Avatar Status | HTTP Request | Poll HeyGen status | Wait for Avatar | Avatar Done? | 🟢 Stage 3: HeyGen Avatar Generation… |
| Avatar Done? | If | Check HeyGen completion | Poll Avatar Status | Set Avatar URL; Wait for Avatar | 🟢 Stage 3: HeyGen Avatar Generation… |
| Set Avatar URL | Set | Store avatar video URL + duration | Avatar Done? (true) | AI Video Director | 🟢 Stage 3: HeyGen Avatar Generation… |
| AI Video Director | Agent (Gemini) | Build beat-by-beat storyboard (render plan) | Set Avatar URL | Creatomate Template Builder Code1 | 🟢 Stage 4: AI Video Director & Creatomate Render… |
| Google Gemini Chat Model3 | Google Gemini Chat Model | LLM for AI Video Director | — | AI Video Director (ai_languageModel) | 🟢 Stage 4: AI Video Director & Creatomate Render… |
| Google Gemini Chat Model4 | Google Gemini Chat Model | LLM for output parser support | — | Structured Output Parser1 (ai_languageModel) | 🟢 Stage 4: AI Video Director & Creatomate Render… |
| Structured Output Parser1 | Structured Output Parser | Enforce director JSON schema | — | AI Video Director (ai_outputParser) | 🟢 Stage 4: AI Video Director & Creatomate Render… |
| Creatomate Template Builder Code1 | Code | Build Creatomate render payload | AI Video Director | Creatomate - Render | 🟢 Stage 4: AI Video Director & Creatomate Render… |
| Creatomate - Render | HTTP Request | Submit render job to Creatomate | Creatomate Template Builder Code1 | Set Render ID | 🟢 Stage 4: AI Video Director & Creatomate Render… |
| Set Render ID | Set | Extract render_id (array/object safe) | Creatomate - Render | Wait for Render | 🟢 Stage 5: Render Polling & Export… |
| Wait for Render | Wait | Delay before polling render status | Set Render ID; Render Done? (false) | Poll Render Status | 🟢 Stage 5: Render Polling & Export… |
| Poll Render Status | HTTP Request | Poll Creatomate render job status | Wait for Render | Render Done? | 🟢 Stage 5: Render Polling & Export… |
| Render Done? | If | Check Creatomate success | Poll Render Status | Download Rendered Video; Wait for Render | 🟢 Stage 5: Render Polling & Export… |
| Download Rendered Video | HTTP Request | Download rendered MP4 as file | Render Done? (true) | Upload to Google Drive | 🟢 Stage 5: Render Polling & Export… |
| Upload to Google Drive | Google Drive | Upload final MP4, get share link | Download Rendered Video | Social Media Copyrighter | 🟢 Stage 5: Render Polling & Export… |
| Social Media Copyrighter | OpenAI | Generate social titles/captions/hashtags JSON | Upload to Google Drive | Append row in sheet |  |
| Append row in sheet | Google Sheets | Append final tracker row + loopback | Social Media Copyrighter | Loop Through Concepts |  |
| Stage 3 | Sticky Note | Comment / stage label | — | — |  |
| Stage 4 | Sticky Note | Comment / stage label | — | — |  |
| Stage 5 | Sticky Note | Comment / stage label | — | — |  |
| Stage 3: AI Analysis | Sticky Note | Comment / stage label | — | — |  |
| Sticky Note | Sticky Note | 🟢 Stage 1 description | — | — |  |
| Sticky Note1 | Sticky Note | 🟢 Stage 2 description | — | — |  |
| Sticky Note2 | Sticky Note | 🟢 Stage 2b description | — | — |  |
| Sticky Note3 | Sticky Note | 🟢 Stage 3 description | — | — |  |
| Sticky Note4 | Sticky Note | 🟢 Stage 4 description | — | — |  |
| Sticky Note5 | Sticky Note | 🟢 Stage 5 description | — | — |  |

> Note: Sticky note nodes are included for completeness, but they do not connect to logic.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: `Shorts Creation v10 - Telegram Filming`.

2. **Add Trigger: “Execute Workflow Trigger”**  
   - Node name: `Shorts Trigger`  
   - Set **Input Source** to **Passthrough**.  
   - This workflow must be called from a parent workflow via **Execute Workflow**.

3. **Add Code node: “Extract Snippets”**  
   - Paste the snippet extraction code (logic: read `source/overview/transcript/key_moments`, build `broll_clips`, `sections`, etc.).  
   - Ensure the parent payload matches expectations:
     - `transcript.sections` array exists
     - `input.key_moments` is an array of objects with `start_seconds/end_seconds` (screen clips)

4. **Add Set node: “Edit Fields”**  
   - Create 3 string fields:
     - `System Prompt` (the long strategist + rules prompt)
     - `User Prompt` (inject video overview, broll clips, section summaries)
     - `Examples` (JSON array string of example creators)
   - Use expressions to insert `$json.video_summary`, `$json.sections`, `$json.broll_clips`, etc.

5. **Add Gemini model nodes (LangChain):**
   - `Google Gemini Chat Model` with `models/gemini-pro-latest` and Gemini API credentials.
   - `Google Gemini Chat Model1` (can be default, used for parser wiring).
   - Create **Google Gemini API credential** in n8n (Google AI Studio key).

6. **Add “Structured Output Parser” (LangChain)**  
   - Schema type: Manual  
   - Auto-fix: enabled  
   - Paste the ideation JSON schema requiring:
     - `concepts[]` (3 concepts)
     - `global_cta` object

7. **Add Agent: “Poll Short Form Concept Ideator”**  
   - Prompt type: Define  
   - Text: expression `{{$json["User Prompt"]}}`  
   - System message: `{{$json["System Prompt"]}}` plus `Examples` injected.  
   - Attach:
     - **ai_languageModel**: `Google Gemini Chat Model`
     - **ai_outputParser**: `Structured Output Parser`

8. **Add Split Out node: “Split Out”**  
   - Field to split out: `output.concepts`

9. **Add Split In Batches node: “Loop Through Concepts”**  
   - Keep defaults, set `reset: false`  
   - Connect `Split Out → Loop Through Concepts`.

10. **Lead Magnet branch (per concept):**
   1. Add Agent `Document Generator` using Gemini:
      - Text: `{{ $('Poll Short Form Concept Ideator').item.json.output.global_cta.deliverable_prompt }}`
      - System message: document generation rules (Markdown).
      - Attach Gemini model `Google Gemini Chat Model2`.
   2. Add Code `Convert Document to HTML`:
      - Set your target Drive folder ID (`FOLDER_ID`).
      - Keep boundary string `docboundary`.
   3. Add HTTP Request `Create Google Doc`:
      - POST `https://www.googleapis.com/upload/drive/v3/files`
      - Query: `uploadType=multipart`, `supportsAllDrives=true`
      - Body: `{{$json.rawData}}`
      - Content type: Raw
      - Raw content type: `multipart/related; boundary=docboundary`
      - Auth: Google Drive OAuth2 credential (must allow Drive upload).
   4. Add Google Drive node `Share file`:
      - Operation: Share
      - File ID: `{{$json.id}}`
      - Permissions: anyone/reader
   5. Add Google Sheets node `Append row in sheet1`:
      - Append row to your tracker sheet
      - Set `Magnet URL` to `https://docs.google.com/document/d/{{ $json.id }}/edit`

11. **Flash B-roll generation branch (per concept):**
   1. Add Split Out node `Split Out1`:
      - Field: `flash_broll`
   2. Add HTTP Request node `Get AI B-Roll`:
      - POST Replicate Seedance endpoint
      - Auth: HTTP Header Auth with Replicate token
      - Body includes prompt, 2 seconds, 9:16, 720p, webhook to `{{$execution.resumeUrl}}`
   3. Add Set node `Extract Flash B-Roll Result1` mapping the flash fields + replicate result.
   4. Add Aggregate node `Aggregate Flash B-Roll` to create `flash_broll_generated`.

12. **Add Code node: “Creatomate Effects Library”**  
   - Paste the effects library code returning `layouts`, `transitions`, `screen_motions`, `avatar_animations`, `audio_tracks`, guidelines.

13. **Add Merge node: “Merge”**  
   - Mode: Combine  
   - Combine by position  
   - Input 0: `Aggregate Flash B-Roll`  
   - Input 1: `Creatomate Effects Library`

14. **HeyGen avatar generation & polling:**
   1. Add HTTP Request `HeyGen - Generate Full Avatar`:
      - POST `https://api.heygen.com/v2/video/generate`
      - Auth: HTTP Header Auth (HeyGen API key)
      - Body uses:
        - selected `avatar_id` per concept
        - `input_text` from concept `full_script`
        - dimension: set to **1080×1920** if you truly want vertical (currently reversed in the provided node).
   2. Add Set `Set Avatar Video ID` (`avatar_video_id = $json.data.video_id`)
   3. Add Wait `Wait for Avatar` (45 seconds)
   4. Add HTTP Request `Poll Avatar Status`:
      - GET `https://api.heygen.com/v1/video_status.get?video_id={{avatar_video_id}}`
   5. Add If `Avatar Done?` checking status == `completed`
      - false → back to `Wait for Avatar`
      - true → `Set Avatar URL`
   6. Add Set `Set Avatar URL` to store `avatar_video_url` and `avatar_duration`

15. **AI Video Director (Gemini) + output parser:**
   1. Add Gemini model node `Google Gemini Chat Model3`
   2. Add Structured Output Parser node `Structured Output Parser1` (director schema)
   3. Add Gemini model node `Google Gemini Chat Model4` (parser wiring)
   4. Add Agent `AI Video Director`:
      - Provide full prompt including:
        - avatar duration
        - concept script + storyboard
        - available screen B-roll (`Extract Snippets` output)
        - available AI B-roll (`Aggregate Flash B-Roll` succeeded)
        - effects library JSON + guidelines
      - Attach model + parser.

16. **Creatomate render build + render/poll/download:**
   1. Add Code node `Creatomate Template Builder Code1` (build Creatomate elements).
   2. Add HTTP Request `Creatomate - Render`:
      - POST `https://api.creatomate.com/v2/renders`
      - Auth: HTTP Header Auth (Creatomate API key)
      - Body: the JSON from builder
   3. Add Set `Set Render ID`
   4. Add Wait `Wait for Render` (30 seconds)
   5. Add HTTP Request `Poll Render Status`
   6. Add If `Render Done?` (`status == succeeded`)
      - false → back to `Wait for Render`
      - true → Download
   7. Add HTTP Request `Download Rendered Video` with response format = file.

17. **Upload & social copy & tracking:**
   1. Add Google Drive node `Upload to Google Drive`:
      - Operation: Upload
      - Folder ID: your shorts folder
      - Name: `{{ AI Video Director title }}_{{ concept_index }}.mp4`
   2. Add OpenAI node `Social Media Copyrighter`:
      - Credentials: OpenAI API key
      - Model: `gpt-5.2`
      - Enforce JSON schema for outputs (YT/IG/TikTok/hashtags/cta).
   3. Add Google Sheets node `Append row in sheet`:
      - Append row with captions + drive link + concept metadata.
   4. **Connect `Append row in sheet` back to `Loop Through Concepts`** to process the next concept.

18. **Credentials to configure**
   - **HeyGen**: HTTP Header Auth (API key)
   - **Replicate**: HTTP Header Auth (token)
   - **Creatomate**: HTTP Header Auth (API key)
   - **Google Gemini (Palm)**: Google AI Studio key
   - **OpenAI**: OpenAI API key
   - **Google Drive OAuth2**: must allow Drive uploads + permissions changes
   - **Google Sheets OAuth2**: must allow Sheets append operations

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Built by @adamgoodyer — The Anti-Guru Technical Educator | https://youtube.com/@adamgoodyer |
| Replicate model used: Seedance 1 Lite (ByteDance) | https://replicate.com/bytedance/seedance-1-lite |
| Workflow stage notes in-canvas | Stage 1–5 sticky notes describe each major pipeline segment |
| Disclaimer (source text provenance) | “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” |

