Create an AI image remix and design bot for Telegram with BrowserAct and Gemini

https://n8nworkflows.xyz/workflows/create-an-ai-image-remix-and-design-bot-for-telegram-with-browseract-and-gemini-12354


# Create an AI image remix and design bot for Telegram with BrowserAct and Gemini

## 1. Workflow Overview

**Workflow name:** AI Image Remix & Design Bot for Telegram with BrowserAct & Gemini  
**Purpose:** A Telegram bot that can (1) chat casually, (2) fetch “inspiration” (top trending AI images/prompts via BrowserAct scraping PromptHero and saving them into Google Sheets), and (3) “remix” images by analyzing either a chosen PromptHero image or a user-uploaded photo, then generating a new image with Google Gemini Image.

**Primary use cases**
- Users send a text like “Start” / “give me ideas” → the bot scrapes PromptHero trends, stores results, and lets the user browse images with inline buttons.
- Users upload a photo → the bot performs forensic image description, asks whether they want changes, and later uses that description to generate a redesigned image.
- Users chat casually → the bot responds without invoking heavy automations.

### 1.1 Entry & Traffic Control (Telegram event routing)
Separates Telegram **callback queries** (button clicks) from **new messages**.

### 1.2 Interactive State Logic (button clicks)
Handles “regenerate”, “change”, numeric indexes for “next image”, and “no”.

### 1.3 Session State Management (Google Sheets)
Stores a global `UserState` in a Google Sheet tab (“Current State”) to control multi-step interactions.

### 1.4 Text Intent Classification (LLM)
If the user sends text, classifies it as either:
- **Start** (inspiration flow), or
- **Chat** (conversational response)

### 1.5 Inspiration Engine (BrowserAct → Google Sheets)
Scrapes PromptHero via BrowserAct, splits results, and saves them to a Google Sheets “PromptHero” tab, then offers browsing.

### 1.6 Visual Forensics (user uploads an image)
Downloads the photo from Telegram, performs deep image analysis, stores the description, and asks if user wants changes.

### 1.7 Prompt Engineering (remix instructions)
Builds an ultra-detailed generation prompt using:
- PromptHero image + prompt (chosen via browsing), or
- User image description stored in Google Sheets

### 1.8 Production & Delivery (Gemini image generation → Telegram)
Generates a thumbnail via Gemini image model and sends it back to Telegram; supports “next/regenerate” via inline buttons.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Traffic Control
**Overview:** Receives Telegram updates and routes execution depending on whether the event is a callback query (inline keyboard button click) or a regular message.  
**Nodes involved:** `Telegram Trigger`, `Check for Query Callback`

#### Node: Telegram Trigger
- **Type/role:** Telegram Trigger (`n8n-nodes-base.telegramTrigger`) — workflow entrypoint via Telegram webhook.
- **Configuration (interpreted):**
  - Subscribes to `message` and `callback_query` updates.
  - Uses a Telegram bot credential (“Telegram account”).
- **Inputs/outputs:**
  - Output → `Check for Query Callback` (main).
- **Key fields used later:**
  - Text: `$('Telegram Trigger').first().json.message.text`
  - Photo: `$('Telegram Trigger').first().json.message.photo`
  - Callback payload: `$('Telegram Trigger').first().json.callback_query.data`
- **Potential failures/edge cases:**
  - Telegram credential misconfigured or webhook not set/blocked.
  - Telegram payload shape differs for some message types (stickers, documents, etc.).

#### Node: Check for Query Callback
- **Type/role:** Switch (`n8n-nodes-base.switch`) — routes by presence of `callback_query`.
- **Configuration (interpreted):**
  - Rule 1: `callback_query` **exists** → treat as button click.
  - Rule 2: `callback_query` **not exists** → treat as normal message.
- **Connections:**
  - Callback branch → `Switch Query`
  - Non-callback branch → `Get User State`
- **Potential failures/edge cases:**
  - If Telegram sends an update type neither contains `.message` nor `.callback_query`, downstream expressions may fail.

**Sticky note context:** “### 🚦 Step 1: The Traffic Controller …” (Gateway)

---

### Block 2 — Interactive State Logic (Callback Queries)
**Overview:** Interprets inline keyboard button presses to either regenerate, request changes, browse next PromptHero images by numeric index, or set a “dodge” state.  
**Nodes involved:** `Switch Query`, `Extract and Save PromptHero Data`, `Notify User2`, `Update Current Image`, `Ask for changes - HeroPrompt`, `Ask for changes - Custom image`, `Set user State`

#### Node: Switch Query
- **Type/role:** Switch — routes based on `callback_query.data`.
- **Configuration (interpreted) rules:**
  1. Equals `"regenerate"` → send to `Ask for changes - HeroPrompt` (current wiring implies “regenerate” leads to asking for changes on HeroPrompt path).
  2. Equals `"change"` → send to `Ask for changes - Custom image`.
  3. Numeric `>= 0` → treat as index into PromptHero sheet; goes to `Extract and Save PromptHero Data`.
  4. Equals `"no"` → goes to `Set user State` (sets state to `dodge`).
- **Inputs/outputs:**
  - Input from `Check for Query Callback`
  - Outputs as above.
- **Edge cases:**
  - Callback values that are numeric strings (e.g. `"0"`) rely on “loose type validation”; usually OK but can misroute if non-numeric text is sent.
  - Rule order matters: numeric rule can accidentally match unexpected strings if coercion occurs (though rule 1/2 checked first).

#### Node: Extract and Save PromptHero Data
- **Type/role:** Google Sheets “Read” — fetch a specific row from “PromptHero” tab based on numeric index.
- **Configuration (interpreted):**
  - Reads values starting at row: `= (Number($json.callback_query.data) || 0) + 2`
    - Index `0` → row 2 (first data row).
  - `alwaysOutputData: true` to avoid stopping if empty.
- **Output:** → `Notify User2`
- **Edge cases:**
  - If the row doesn’t exist, output may be empty; downstream uses `Image Url` and may fail or send blank image.
  - Spreadsheet/tab ID must match user’s sheet; permissions required.

#### Node: Notify User2
- **Type/role:** Telegram “Send Photo” — shows PromptHero image and adds inline buttons.
- **Configuration (interpreted):**
  - Sends photo from URL: `={{ $json["Image Url"] }}`
  - Chat id: callback message chat: `={{ $('Telegram Trigger').first().json.callback_query.message.chat.id }}`
  - Inline keyboard:
    - “go next” → callback_data = current numeric + 1
    - “I Like it let's regenerate.” → callback_data = `regenerate`
  - `executeOnce: true` (important in loops/multi-item contexts).
- **Output:** → `Update Current Image`
- **Edge cases:**
  - Telegram may reject invalid/blocked image URLs.
  - If the workflow ever has multiple items, `executeOnce` ensures only one send occurs (may hide other items).

#### Node: Update Current Image
- **Type/role:** Google Sheets “Update” — stores the currently displayed PromptHero selection into “Current Image” tab.
- **Configuration (interpreted):**
  - Updates row_number = 2 with:
    - `Model`, `Ptompt`, `Image Url` from `Extract and Save PromptHero Data`.
  - `executeOnce: true`
- **Edge cases:**
  - Column name typo: `Ptompt` is used throughout (must match your sheet).
  - If `Extract and Save PromptHero Data` output is empty, updates may write blanks.

#### Node: Ask for changes - HeroPrompt
- **Type/role:** Telegram “Send Message” — prompts user for next instructions (HeroPrompt path).
- **Configuration:**
  - Message: “I'm so thrilled to hear it! Now show me the way.”
  - Chat id from callback query message.
- **Output:** → `Set user State4` (sets `HeroPrompt` state)
- **Edge cases:** minimal; mainly Telegram credential/chat id.

#### Node: Ask for changes - Custom image
- **Type/role:** Telegram “Send Message” — prompts user for next instructions (custom uploaded image path).
- **Configuration:**
  - Message: “Got it. show me the directions.”
  - Chat id from callback query message.
- **Output:** → `Set user State3` (sets `User Image` state)

#### Node: Set user State
- **Type/role:** Google Sheets “Update” — sets global `UserState` to `dodge` at row 2.
- **Connection:** From `Switch Query` “no” path.
- **Edge cases:**
  - This design stores **one global state** (row 2), not per Telegram user. Multiple users will overwrite each other.

**Sticky note context:** “### 🕹️ Step 2: Interactive State Logic …” (Interaction)

---

### Block 3 — Session State Management (Google Sheets-based)
**Overview:** Reads and writes `UserState` to decide whether incoming messages should be treated as image changes for a specific mode (HeroPrompt vs User Image) or handled normally.  
**Nodes involved:** `Get User State`, `Switch User State`, `Set user State1`, `Set user State2`, `Set user State3`, `Set user State4`, `Set user State5`, `Set user State`, `Set user State1/2/3/4/5` (all state setters)

#### Node: Get User State
- **Type/role:** Google Sheets “Read” — reads Current State tab (from row 2 onward).
- **Output:** → `Switch User State`
- **Edge cases:**
  - If sheet returns multiple rows, `Switch User State` may run multiple times; workflow seems to assume a single state row.

#### Node: Switch User State
- **Type/role:** Switch — routes by `UserState`.
- **Rules:**
  1. `UserState == "User Image"` → `Get Description`
  2. `UserState == "HeroPrompt"` (note: rightValue is `=HeroPrompt` in JSON; effectively literal “HeroPrompt” is intended) → `Get Prompt`
  3. `UserState == "dodge"` → `Check for Image or Text`
- **Edge cases:**
  - The second rule’s `rightValue` includes a leading `=` in the JSON (`"=HeroPrompt"`). If treated literally, it may never match unless the cell includes “=HeroPrompt”. Verify in n8n UI.

#### Node: Set user State1
- **Type/role:** Google Sheets “Update” — sets `UserState = dodge` row 2.
- **Used in:** `Check for Image or Text` (text path) before intent classification.
- **Purpose:** resets state so the next text is treated normally unless explicitly changed.

#### Node: Set user State2
- **Type/role:** Google Sheets “Update” — sets `UserState = dodge` row 2.
- **Used in:** `Check for Image or Text` (photo path) before downloading the photo.

#### Node: Set user State3
- **Type/role:** Google Sheets “Update” — sets `UserState = "User Image"` row 2.
- **Used in:** after “Ask for changes - Custom image” to indicate future text should be treated as change requests for the uploaded image.

#### Node: Set user State4
- **Type/role:** Google Sheets “Update” — sets `UserState = "HeroPrompt"` row 2.
- **Used in:** after “Ask for changes - HeroPrompt” to indicate future text should be treated as change requests to apply to the chosen PromptHero image.

#### Node: Set user State5
- **Type/role:** Google Sheets “Update” — sets `UserState = dodge` row 2.
- **Used in:** after sending the generated thumbnail to reset state.

**Sticky note context:** “### 💾 Step 3: Session State Management …” (State)

---

### Block 4 — Text vs Image Detection (Normal Messages)
**Overview:** For non-callback messages, decides whether the user sent text (intent classification) or a photo (visual forensics).  
**Nodes involved:** `Check for Image or Text`

#### Node: Check for Image or Text
- **Type/role:** Switch — detects `message.text` vs `message.photo`.
- **Rules (interpreted):**
  - If `Telegram Trigger … message.text` exists → `Set user State1` (reset to dodge) → then intent classification.
  - If `Telegram Trigger … message.photo` exists → `Set user State2` (reset to dodge) → download & analyze image.
- **Edge cases:**
  - Telegram photo is an array; existence check on `message.photo` is OK.
  - Messages containing both text and photo/caption can occur; this switch checks text existence first.

---

### Block 5 — Intent Classification (Text → Start vs Chat)
**Overview:** Classifies user text into “Start” (inspiration flow) vs “Chat” (conversational reply).  
**Nodes involved:** `Google Gemini`, `Validate user input`, `Structured Output Parser`, `Validation Type Switch`

#### Node: Google Gemini
- **Type/role:** LangChain Chat Model (`lmChatGoogleGemini`) used as the language model for the classifier agent and output parser.
- **Connections:** feeds as `ai_languageModel` into:
  - `Validate user input`
  - `Structured Output Parser`
- **Edge cases:**
  - Credential (“Google Gemini(PaLM) Api account”) must be valid and enabled.
  - Model defaults are unspecified; Gemini model availability depends on account/region.

#### Node: Validate user input
- **Type/role:** LangChain Agent — classification engine.
- **Configuration:**
  - Input text: `={{ $json.message.text }}`
  - System message: strict rules to output ONLY JSON: `{"Type":"Start"}` or `{"Type":"Chat"}`
  - `hasOutputParser: true` (expects structured parsing).
- **Output:** → `Validation Type Switch`
- **Failure modes:**
  - If the LLM outputs non-JSON, parser must fix; may still fail.
  - `$json.message.text` assumes the incoming item structure includes `message.text` at that stage.

#### Node: Structured Output Parser
- **Type/role:** Structured output parser — enforces JSON schema.
- **Configuration:**
  - Example schema: `{"Type":"Chat"}`
  - `autoFix: true` (attempts to repair malformed JSON).
- **Connection:** attached to `Validate user input` via `ai_outputParser`.
- **Edge cases:** If output is too malformed, execution fails.

#### Node: Validation Type Switch
- **Type/role:** Switch — routes based on `output.Type`.
- **Rules:**
  - If `output.Type == "Start"` → `Notify User` and `Clear PromptHero Database` (both connected as separate outputs in same branch order)
  - If `output.Type == "Chat"` → `Chatting Agent`
- **Edge cases:**
  - If parser output path differs, `{{ $json.output.Type }}` may be undefined.

**Sticky note context:** “### 🧠 Step 4: Intent Classification …” (Intent)

---

### Block 6 — Conversational AI (Chat branch)
**Overview:** For casual chat, generates a lightweight response and sends it back to Telegram.  
**Nodes involved:** `Google Gemini1`, `Chatting Agent`, `Chat With User`

#### Node: Google Gemini1
- **Type/role:** LangChain Chat Model used by `Chatting Agent`.
- **Connection:** `ai_languageModel` → `Chatting Agent`

#### Node: Chatting Agent
- **Type/role:** LangChain Agent — produces Telegram-safe HTML text.
- **Configuration:**
  - Input: `Input type : {{ $json.output.Type }} | User Input : {{ ...message.text }}`
  - System instructions: no JSON, no Markdown, only limited Telegram HTML tags.
- **Output:** → `Chat With User`
- **Edge cases:**
  - Telegram HTML parsing errors if model outputs unsupported tags; prompt tries to prevent that.

#### Node: Chat With User
- **Type/role:** Telegram “Send Message”
- **Configuration:**
  - Text: `={{ $json.output }}`
  - chatId: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
  - parse mode: HTML
- **Edge cases:** If the agent output includes invalid HTML, Telegram may reject.

**Sticky note context:** “### 💬 Step 5: Conversational AI …” (Chat)

---

### Block 7 — Inspiration Engine (Start branch: BrowserAct → DB)
**Overview:** When user asks for inspiration, clears PromptHero sheet, runs BrowserAct workflow to fetch top images/prompts, parses results into items, saves to Google Sheets, then asks user to browse.  
**Nodes involved:** `Notify User`, `Clear PromptHero Database`, `Extract Top AI-Generated Images`, `Splitting Generated Items`, `Save Images To Database`, `Send Finishing Alert`

#### Node: Notify User
- **Type/role:** Telegram message — informs user scraping is starting.
- **Configuration:**
  - “Got it. I'll get a few ideas from PromptHero.”
  - chatId from `Telegram Trigger … message.chat.id`
- **Edge cases:** none major.

#### Node: Clear PromptHero Database
- **Type/role:** Google Sheets “Clear”
- **Configuration:**
  - Clears range `A2:F` in “PromptHero” tab (gid=0).
- **Output:** → `Extract Top AI-Generated Images`
- **Edge cases:**
  - If sheet has different structure, clearing A2:F may remove needed columns.

#### Node: Extract Top AI-Generated Images
- **Type/role:** BrowserAct node — runs a BrowserAct hosted workflow (template referenced in sticky note).
- **Configuration:**
  - `type: WORKFLOW`
  - `workflowId: 69279796243400430`
  - `timeout: 7200` seconds
- **Output:** → `Splitting Generated Items`
- **Failure modes:**
  - BrowserAct API key invalid.
  - Workflow ID not found / template not present in account.
  - Timeouts or scraping site changes (PromptHero layout).
  - Output format changes break downstream parser (see next node).

#### Node: Splitting Generated Items
- **Type/role:** Code node — parses JSON string returned by BrowserAct into multiple n8n items.
- **Key logic:**
  - Reads: `$input.first().json.output.string`
  - `JSON.parse` → must be an array of objects.
  - Emits each element as `{ json: item }`.
- **Output:** → `Save Images To Database`
- **Failure modes:**
  - Throws explicit errors if string missing or parse fails.
  - If BrowserAct returns an object instead of array, errors.

#### Node: Save Images To Database
- **Type/role:** Google Sheets “Append” — stores each scraped image entry.
- **Columns appended:** `Model`, `Ptompt`, `Image Url` from `$json.model`, `$json.prompt`, `$json.image`.
- **Output:** → `Send Finishing Alert`
- **Edge cases:** Rate limits on Sheets API when appending many rows.

#### Node: Send Finishing Alert
- **Type/role:** Telegram message with inline keyboard — invites user to browse.
- **Buttons:**
  - “Yes, Continue Please.” → callback_data `"0"` (start browsing at index 0)
  - “No, I don't have credits in my Gemini account” → callback_data `"no"`
- **Edge cases:** This branch depends on callbacks being handled by Block 2.

**Sticky note context:** “### 🌍 Step 6: The Inspiration Engine …” (BrowserAct)

---

### Block 8 — Visual Forensics (User uploaded image)
**Overview:** Downloads the user’s Telegram photo, clears prior stored user image data, analyzes the image with an LLM vision agent, saves description into Google Sheets, then asks if the user wants changes.  
**Nodes involved:** `Get Picture from User`, `Clear UserData`, `OpenRouter`, `Structured Output`, `Analyze the user image input`, `Update Image Descriptions`, `Ask User for Continuation`, `Set user State2` (already used earlier)

#### Node: Get Picture from User
- **Type/role:** Telegram “Get File” — downloads the last (highest resolution) photo.
- **Configuration:**
  - `fileId = {{ message.photo.last().file_id }}`
  - resource: file
- **Output:** → `Clear UserData`
- **Edge cases:**
  - If user sends non-photo media, `.photo.last()` fails.
  - Large file downloads may be slow.

#### Node: Clear UserData
- **Type/role:** Google Sheets “Clear” — clears “UserImage” tab while keeping header.
- **Configuration:** `keepFirstRow: true`
- **Output:** → `Analyze the user image input`
- **Edge cases:** Clears shared global storage (not per user).

#### Node: OpenRouter
- **Type/role:** OpenRouter Chat Model (`openai/gpt-4o`) used as vision-capable LLM for analyzing images.
- **Connection:** `ai_languageModel` → `Analyze the user image input`
- **Edge cases:** OpenRouter key/quota; model availability.

#### Node: Structured Output
- **Type/role:** Structured output parser (array-of-objects schema `[{ "output": "..." }]`) for image analysis agent.
- **Connection:** `ai_outputParser` → `Analyze the user image input`
- **Edge cases:** Output mismatch; parser auto-fix may fail.

#### Node: Analyze the user image input
- **Type/role:** LangChain Agent (vision) — forensic image description.
- **Configuration highlights:**
  - Input: binary from Telegram download: `={{ $('Get Picture from User').first().binary }}`
  - `passthroughBinaryImages: true` (required for vision input handling)
  - System message: extremely strict “raw descriptive text only”.
- **Output:** → `Update Image Descriptions`
- **Failures:**
  - If binary not passed correctly, model gets no image.
  - Vision model may refuse or produce nonconforming output; parser attempts repair.

#### Node: Update Image Descriptions
- **Type/role:** Google Sheets “Update” — stores:
  - `Images ID` = Telegram `file_id`
  - `Image Description` = `$json.output[0].output`
  - row_number = 2
- **Output:** → `Ask User for Continuation`

#### Node: Ask User for Continuation
- **Type/role:** Telegram message with inline keyboard
- **Text:** “Okay, I got the image. Do you want me to change it?”
- **Buttons:**
  - Yes → callback_data `change`
  - No → callback_data `no`
- **Edge cases:** Subsequent routing depends on `Switch Query`.

**Sticky note context:** “### 👁️ Step 7: Visual Forensics Lab …” (Forensics)

---

### Block 9 — Prompt Engineering (from PromptHero selection OR user image description)
**Overview:** Builds a hyper-detailed generation prompt. There are two prompt-engineering paths:
- **HeroPrompt-based remix:** Analyze chosen PromptHero image → generate a huge prompt using image analysis + PromptHero prompt + user requested changes.
- **UserImage-based remix:** Use stored user image description + user requested changes → generate huge prompt.  
**Nodes involved:** `Get Prompt`, `OpenRouter1`, `Structured Output Parser4`, `Analyze the Chosen image`, `Generate Image From PromptHero`, `Get Description`, `Generate Image From UserData`, `Google Gemini2`, `Structured Output1`, `Structured Output2`, `Fix Output`, `Fix Output1`, plus shared state routing via `Switch User State`.

#### Node: Get Prompt
- **Type/role:** Google Sheets “Read” — reads “Current Image” tab (PromptHero selected image/prompt).
- **Output:** → `Analyze the Chosen image`
- **Edge cases:** If no current image is stored, later nodes may have missing `Image Url` / `Ptompt`.

#### Node: OpenRouter1
- **Type/role:** OpenRouter Chat Model (`openai/gpt-4o`) used by `Analyze the Chosen image`.
- **Connection:** `ai_languageModel` → `Analyze the Chosen image`

#### Node: Structured Output Parser4
- **Type/role:** Structured output parser (expects `[{ "output": "..." }]`) for image analysis of chosen PromptHero image.
- **Connection:** `ai_outputParser` → `Analyze the Chosen image`

#### Node: Analyze the Chosen image
- **Type/role:** LangChain Agent (vision via image URL) — forensic analysis of PromptHero image.
- **Input (multi-part message):**
  - Text: “Analyze this image in extreme detail.”
  - Image URL: `{{ $json["Image Url"] }}` with `detail: "high"`
- **Output:** → `Generate Image From PromptHero`
- **Edge cases:** Invalid image URL or blocked host → vision fails.

#### Node: Google Gemini2
- **Type/role:** Gemini chat model (explicit `models/gemini-2.5-pro`) used as LLM for both prompt-engineering agents.
- **Connection:** `ai_languageModel` → `Generate Image From PromptHero` and `Generate Image From UserData`

#### Node: Structured Output2
- **Type/role:** Parser enforcing `{ "Prompt": "Prompt" }` output for PromptHero prompt engineering.
- **Connection:** `ai_outputParser` → `Generate Image From PromptHero`

#### Node: Generate Image From PromptHero
- **Type/role:** LangChain Agent — master prompt engineer for PromptHero path.
- **Input:** Combines:
  - Image description from analysis: `{{ $json.output[0].output }}`
  - PromptHero prompt: `{{ $('Get Prompt').first().json.Ptompt }}`
  - User change request: `{{ $('Telegram Trigger').first().json.message.text }}`
- **System constraints:** “Rule of Multiplication”, no markdown, output must be JSON: `{"Prompt":"..."}`
- **Output:** → `Generate Thumbnail`
- **Edge cases:** If user text is absent (because flow was triggered by callback), `message.text` might be undefined.

#### Node: Get Description
- **Type/role:** Google Sheets “Read” — reads “UserImage” tab to get stored `Image Description`.
- **Output:** → `Generate Image From UserData`

#### Node: Structured Output1
- **Type/role:** Parser enforcing `{ "Prompt": "Prompt" }` for user image prompt engineering.
- **Connection:** `ai_outputParser` → `Generate Image From UserData`

#### Node: Generate Image From UserData
- **Type/role:** LangChain Agent — master prompt engineer for user image path.
- **Input:** Combines:
  - Stored description: `{{ $json["Image Description"] }}`
  - User change request: `{{ $('Telegram Trigger').first().json.message.text }}`
- **Output:** → `Generate Thumbnail`

#### Node: Fix Output / Fix Output1
- **Type/role:** Gemini chat models (`models/gemini-2.5-pro`) connected as `ai_languageModel` into parsers.
- **Observed wiring:**
  - `Fix Output` provides `ai_languageModel` to `Structured Output Parser4`.
  - `Fix Output1` provides `ai_languageModel` to `Structured Output1` and `Structured Output2`.
- **Purpose (interpreted):** used by n8n’s structured parser auto-fix mechanism (LLM-assisted correction).
- **Edge cases:** If these models are not available, structured parsing auto-fix may fail.

**Sticky note context:** “### 🎨 Step 8: Master Prompt Engineering …” (Creation)

---

### Block 10 — Production & Delivery (Gemini Image → Telegram)
**Overview:** Generates an image thumbnail using Gemini image model from whichever prompt-engineering path executed, then sends it to Telegram and resets user state.  
**Nodes involved:** `Generate Thumbnail`, `Send Thumbnail Back to Chatbot`, `Set user State5`

#### Node: Generate Thumbnail
- **Type/role:** Google Gemini Image node (`@n8n/n8n-nodes-langchain.googleGemini`) — renders an image from prompt.
- **Configuration:**
  - Model: `models/gemini-3-pro-image-preview` (listed as “Nano Banana Pro” in UI)
  - `sampleCount: 1`
  - outputs binary to property: `data`
  - Prompt expression:
    - If `Generate Image From PromptHero` executed → use its `output.Prompt`
    - Else → use `Generate Image From UserData` `output.Prompt`
- **Output:** → `Send Thumbnail Back to Chatbot`
- **Edge cases:**
  - Requires Gemini image-generation access/credits.
  - Prompt may be too long; model limits can reject.
  - Expression uses `$("Generate Image From PromptHero").isExecuted`—works only within same execution context.

#### Node: Send Thumbnail Back to Chatbot
- **Type/role:** Telegram “Send Photo” (binary)
- **Configuration:**
  - `binaryData: true`
  - chatId: from `Telegram Trigger … message.chat.id`
- **Output:** → `Set user State5`
- **Edge cases:**
  - If initial trigger was callback-only, `message.chat.id` might be absent; but typically callback queries also contain message; still verify.

#### Node: Set user State5
- **Type/role:** Google Sheets “Update” — resets state to `dodge` after image delivery.

**Sticky note context:** “### 🖼️ Step 9: Production & Delivery …” (Production)

---

### Block 11 — Unused / Orphaned Nodes (present but not connected in main flow)
**Overview:** These nodes exist in the workflow but are not connected into the active execution path (based on the provided connections). They still must be documented because they may be intended for future steps or were left behind from iterations.  
**Nodes involved:** `OpenRouter (already used)`, `OpenRouter (used)`, `0edea536 Structured Output` (used), plus **these orphaned**: `f072d5c5 OpenRouter` is used; `0edea536 Structured Output` is used; the true orphans are:

- `07c502d9 Google Gemini` is used.
- `5802722b Structured Output Parser` is used.
- **Orphaned nodes identified by no main connections:**
  - `75390f65 Google Gemini2` is used as AI model link.
  - `7101cde9 Fix Output`, `0af8b812 Fix Output1`, `644d8633 Fix Output3` (Fix Output3 is connected to `Structured Output`, but `Structured Output` is used; Fix Output3 itself is not otherwise used.)
  - `dc7e52b6 Google Gemini1` used.
  - `ff0b6a4f Structured Output Parser4` used.
  - `f072d5c5 OpenRouter` used.
  - **Important true orphan:** `Fix Output3` is connected as AI model to `Structured Output` parser; so it is indirectly used.  
  - There is **no node** named “Fix Output2”; none missing.

Additionally, note a structural issue:
- `cadbc9d7` + `5802722b` output parsing is set, but `Structured Output Parser` is attached; OK.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram updates (message/callback_query) | — | Check for Query Callback | ### 🚦 Step 1: The Traffic Controller |
| Check for Query Callback | switch | Route between callback query vs normal message | Telegram Trigger | Switch Query; Get User State | ### 🚦 Step 1: The Traffic Controller |
| Switch Query | switch | Handle inline button payloads (regenerate/change/index/no) | Check for Query Callback | Ask for changes - HeroPrompt; Ask for changes - Custom image; Extract and Save PromptHero Data; Set user State | ### 🕹️ Step 2: Interactive State Logic |
| Extract and Save PromptHero Data | googleSheets (read) | Fetch PromptHero row by numeric index | Switch Query | Notify User2 | ### 🕹️ Step 2: Interactive State Logic |
| Notify User2 | telegram (sendPhoto) | Show PromptHero image with inline keyboard | Extract and Save PromptHero Data | Update Current Image | ### 🖼️ Step 9: Production & Delivery |
| Update Current Image | googleSheets (update) | Store currently selected PromptHero item | Notify User2 | — | ### 🕹️ Step 2: Interactive State Logic |
| Ask for changes - HeroPrompt | telegram (sendMessage) | Ask for change directions (HeroPrompt path) | Switch Query | Set user State4 | ### 🕹️ Step 2: Interactive State Logic |
| Ask for changes - Custom image | telegram (sendMessage) | Ask for change directions (User Image path) | Switch Query | Set user State3 | ### 🕹️ Step 2: Interactive State Logic |
| Set user State | googleSheets (update) | Set UserState=dodge on “no” | Switch Query | — | ### 💾 Step 3: Session State Management |
| Get User State | googleSheets (read) | Read current UserState | Check for Query Callback | Switch User State | ### 💾 Step 3: Session State Management |
| Switch User State | switch | Route logic by UserState (User Image/HeroPrompt/dodge) | Get User State | Get Description; Get Prompt; Check for Image or Text | ### 💾 Step 3: Session State Management |
| Check for Image or Text | switch | For normal messages: text vs photo | Switch User State | Set user State1; Set user State2 |  |
| Set user State1 | googleSheets (update) | Reset UserState=dodge before intent classification | Check for Image or Text | Validate user input | ### 🧠 Step 4: Intent Classification |
| Google Gemini | lmChatGoogleGemini | LLM for intent classification + parser | — (AI model link) | (AI model to Validate user input, Structured Output Parser) | ### 🧠 Step 4: Intent Classification |
| Structured Output Parser | outputParserStructured | Enforce `{Type}` JSON for classifier | — (AI parser link) | (attached to Validate user input) | ### 🧠 Step 4: Intent Classification |
| Validate user input | langchain agent | Classify text into Start vs Chat | Set user State1 | Validation Type Switch | ### 🧠 Step 4: Intent Classification |
| Validation Type Switch | switch | Branch: Start → scraping; Chat → chatbot | Validate user input | Notify User + Clear PromptHero Database; Chatting Agent | ### 🧠 Step 4: Intent Classification |
| Notify User | telegram (sendMessage) | Inform scraping PromptHero ideas will start | Validation Type Switch | — | ### 🌍 Step 6: The Inspiration Engine |
| Clear PromptHero Database | googleSheets (clear) | Clear PromptHero tab data range | Validation Type Switch | Extract Top AI-Generated Images | ### 🌍 Step 6: The Inspiration Engine |
| Extract Top AI-Generated Images | browserAct | Scrape PromptHero trending images/prompts | Clear PromptHero Database | Splitting Generated Items | ### 🌍 Step 6: The Inspiration Engine |
| Splitting Generated Items | code | Parse BrowserAct JSON string into items | Extract Top AI-Generated Images | Save Images To Database | ### 🌍 Step 6: The Inspiration Engine |
| Save Images To Database | googleSheets (append) | Append scraped items to PromptHero tab | Splitting Generated Items | Send Finishing Alert | ### 🌍 Step 6: The Inspiration Engine |
| Send Finishing Alert | telegram (sendMessage) | Ask user to browse the saved results | Save Images To Database | — | ### 🌍 Step 6: The Inspiration Engine |
| Chatting Agent | langchain agent | Generate conversational response | Validation Type Switch | Chat With User | ### 💬 Step 5: Conversational AI |
| Google Gemini1 | lmChatGoogleGemini | LLM backing the chat agent | — (AI model link) | (AI model to Chatting Agent) | ### 💬 Step 5: Conversational AI |
| Chat With User | telegram (sendMessage) | Send chat response to Telegram | Chatting Agent | — | ### 💬 Step 5: Conversational AI |
| Set user State2 | googleSheets (update) | Reset UserState=dodge before processing uploaded photo | Check for Image or Text | Get Picture from User | ### 👁️ Step 7: Visual Forensics Lab |
| Get Picture from User | telegram (file) | Download user photo binary from Telegram | Set user State2 | Clear UserData | ### 👁️ Step 7: Visual Forensics Lab |
| Clear UserData | googleSheets (clear) | Clear UserImage tab (keep header) | Get Picture from User | Analyze the user image input | ### 👁️ Step 7: Visual Forensics Lab |
| OpenRouter | lmChatOpenRouter | Vision-capable LLM (gpt-4o) for user image analysis | — (AI model link) | (AI model to Analyze the user image input) | ### 👁️ Step 7: Visual Forensics Lab |
| Structured Output | outputParserStructured | Enforce `[{output:...}]` schema for image analysis | — (AI parser link) | (attached to Analyze the user image input) | ### 👁️ Step 7: Visual Forensics Lab |
| Fix Output3 | lmChatGoogleGemini | LLM used by structured parser auto-fix (user image analysis) | — (AI model link) | (AI model to Structured Output) |  |
| Analyze the user image input | langchain agent | Forensic description of uploaded image | Clear UserData | Update Image Descriptions | ### 👁️ Step 7: Visual Forensics Lab |
| Update Image Descriptions | googleSheets (update) | Save image file_id + description to UserImage tab | Analyze the user image input | Ask User for Continuation | ### 👁️ Step 7: Visual Forensics Lab |
| Ask User for Continuation | telegram (sendMessage) | Ask if user wants changes (change/no buttons) | Update Image Descriptions | — | ### 👁️ Step 7: Visual Forensics Lab |
| Set user State3 | googleSheets (update) | Set UserState=User Image | Ask for changes - Custom image | — | ### 💾 Step 3: Session State Management |
| Set user State4 | googleSheets (update) | Set UserState=HeroPrompt | Ask for changes - HeroPrompt | — | ### 💾 Step 3: Session State Management |
| Get Description | googleSheets (read) | Read stored user image description | Switch User State | Generate Image From UserData | ### 🎨 Step 8: Master Prompt Engineering |
| Get Prompt | googleSheets (read) | Read current PromptHero selection (image/prompt) | Switch User State | Analyze the Chosen image | ### 🎨 Step 8: Master Prompt Engineering |
| OpenRouter1 | lmChatOpenRouter | Vision-capable LLM for analyzing chosen PromptHero image | — (AI model link) | (AI model to Analyze the Chosen image) | ### 🎨 Step 8: Master Prompt Engineering |
| Structured Output Parser4 | outputParserStructured | Enforce `[{output:...}]` schema for chosen image analysis | — (AI parser link) | (attached to Analyze the Chosen image) |  |
| Fix Output | lmChatGoogleGemini | LLM used by parser auto-fix (chosen image analysis) | — (AI model link) | (AI model to Structured Output Parser4) |  |
| Analyze the Chosen image | langchain agent | Forensic analysis of PromptHero chosen image URL | Get Prompt | Generate Image From PromptHero | ### 🎨 Step 8: Master Prompt Engineering |
| Google Gemini2 | lmChatGoogleGemini | LLM (gemini-2.5-pro) for prompt engineering | — (AI model link) | (AI model to Generate Image From PromptHero/UserData) | ### 🎨 Step 8: Master Prompt Engineering |
| Structured Output1 | outputParserStructured | Enforce `{Prompt:...}` for user-image prompt engineer | — (AI parser link) | (attached to Generate Image From UserData) |  |
| Structured Output2 | outputParserStructured | Enforce `{Prompt:...}` for PromptHero prompt engineer | — (AI parser link) | (attached to Generate Image From PromptHero) |  |
| Fix Output1 | lmChatGoogleGemini | LLM used by `{Prompt}` parsers auto-fix | — (AI model link) | (AI model to Structured Output1 & Structured Output2) |  |
| Generate Image From UserData | langchain agent | Build hyper-detailed prompt from stored user image description | Get Description | Generate Thumbnail | ### 🎨 Step 8: Master Prompt Engineering |
| Generate Image From PromptHero | langchain agent | Build hyper-detailed prompt from chosen PromptHero image+prompt | Analyze the Chosen image | Generate Thumbnail | ### 🎨 Step 8: Master Prompt Engineering |
| Generate Thumbnail | googleGemini (image) | Generate image binary via Gemini image model | Generate Image From UserData / Generate Image From PromptHero | Send Thumbnail Back to Chatbot | ### 🖼️ Step 9: Production & Delivery |
| Send Thumbnail Back to Chatbot | telegram (sendPhoto) | Send generated image to user | Generate Thumbnail | Set user State5 | ### 🖼️ Step 9: Production & Delivery |
| Set user State5 | googleSheets (update) | Reset UserState=dodge after delivery | Send Thumbnail Back to Chatbot | — | ### 💾 Step 3: Session State Management |
| Structured Output Parser4 (already listed) |  |  |  |  |  |
| Sticky Note - Intro | stickyNote | Documentation panel | — | — |  |
| Sticky Note - Gateway | stickyNote | Documentation panel | — | — |  |
| Sticky Note - Interaction | stickyNote | Documentation panel | — | — |  |
| Sticky Note - State | stickyNote | Documentation panel | — | — |  |
| Sticky Note - Intent | stickyNote | Documentation panel | — | — |  |
| Sticky Note - Chat | stickyNote | Documentation panel | — | — |  |
| Sticky Note - BrowserAct | stickyNote | Documentation panel | — | — |  |
| Sticky Note - Forensics | stickyNote | Documentation panel | — | — |  |
| Sticky Note - Creation | stickyNote | Documentation panel | — | — |  |
| Sticky Note - Production | stickyNote | Documentation panel | — | — |  |
| Sticky Note | stickyNote | Video embed panel | — | — | @[youtube](GqeKd9aYjW4) |
| Sticky Note1 | stickyNote | Complexity warning | — | — | ## ⚠️ Complex Workflow / Please proceed using the tutorial video. |

> Note: sticky notes are included as nodes in the workflow but do not affect execution.

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Telegram Bot credential (token).
   2. Google Sheets OAuth2 credential (access to your target spreadsheet).
   3. Google Gemini (PaLM/Gemini) API credential (must support both chat + image generation if you use the image node).
   4. OpenRouter API credential (for `openai/gpt-4o`).
   5. BrowserAct API credential (and ensure the template/workflow exists in your BrowserAct account).

2) **Prepare Google Sheet**
   - Create a spreadsheet with tabs (exact names expected by node configuration):
     - `PromptHero`
     - `Current State`
     - `UserImage`
     - `Current Image`
   - Ensure columns exist exactly as referenced:
     - **PromptHero**: `Model`, `Ptompt`, `Image Url`
     - **Current State**: `UserState`, `row_number` (row_number used for matching row 2)
     - **UserImage**: `Images ID`, `Image Description`, `row_number`
     - **Current Image**: `Model`, `Ptompt`, `Image Url`, `row_number`
   - Put headers in row 1; the workflow uses row 2 as the main storage row for state/current selections.

3) **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Updates: enable `message` and `callback_query`.
   - Connect to next node.

4) **Add “Check for Query Callback” switch**
   - Node: **Switch**
   - Rule A: `{{$json.callback_query}}` exists → output 1 (callback path)
   - Rule B: `{{$json.callback_query}}` not exists → output 2 (message path)
   - Connect:
     - callback path → `Switch Query`
     - message path → `Get User State`

5) **Callback logic (“Switch Query”)**
   - Node: **Switch**
   - Evaluate `{{$json.callback_query.data}}`
   - Create rules in order:
     1. equals `regenerate` → `Ask for changes - HeroPrompt`
     2. equals `change` → `Ask for changes - Custom image`
     3. number >= 0 → `Extract and Save PromptHero Data`
     4. equals `no` → `Set user State` (set dodge)
   - Connect each output accordingly.

6) **PromptHero browse pipeline**
   1. Node: **Google Sheets (Read)** named `Extract and Save PromptHero Data`
      - Tab: `PromptHero`
      - Range/first data row formula: `(Number(callback_data)||0)+2`
   2. Node: **Telegram (Send Photo)** named `Notify User2`
      - Photo URL from `Image Url`
      - Inline keyboard:
        - Next: callback_data = current numeric + 1
        - Regenerate: callback_data = `regenerate`
   3. Node: **Google Sheets (Update)** named `Update Current Image`
      - Tab: `Current Image`
      - Match `row_number = 2`
      - Set `Model`, `Ptompt`, `Image Url` from the read row.

7) **State setters for change prompts**
   - Node: **Telegram (Send Message)** `Ask for changes - HeroPrompt` → then **Google Sheets (Update)** `Set user State4` setting `UserState="HeroPrompt"` at row 2.
   - Node: **Telegram (Send Message)** `Ask for changes - Custom image` → then **Google Sheets (Update)** `Set user State3` setting `UserState="User Image"` at row 2.
   - Node: **Google Sheets (Update)** `Set user State` sets `UserState="dodge"` at row 2.

8) **User state read + routing**
   - Node: **Google Sheets (Read)** `Get User State` from tab `Current State`.
   - Node: **Switch** `Switch User State` on `{{$json.UserState}}`:
     - “User Image” → `Get Description`
     - “HeroPrompt” → `Get Prompt`
     - “dodge” → `Check for Image or Text`

9) **Detect text vs photo**
   - Node: **Switch** `Check for Image or Text`
     - If `Telegram Trigger.message.text` exists → `Set user State1`
     - If `Telegram Trigger.message.photo` exists → `Set user State2`

10) **Intent classification (text path)**
   1. Node: **Google Sheets (Update)** `Set user State1` sets `UserState="dodge"`.
   2. Node: **LangChain Agent** `Validate user input`
      - Text: `{{$json.message.text}}`
      - System message: classification rules producing ONLY `{"Type":"Start"}` or `{"Type":"Chat"}`
      - Attach **Structured Output Parser** expecting `{"Type":"Chat"}` (autoFix on).
      - Attach a **Gemini chat model** node as the language model.
   3. Node: **Switch** `Validation Type Switch`
      - If `output.Type == "Start"` → `Notify User` and `Clear PromptHero Database`
      - If `output.Type == "Chat"` → `Chatting Agent`

11) **Chat branch**
   1. Node: **LangChain Chat Model (Gemini)** `Google Gemini1`
   2. Node: **LangChain Agent** `Chatting Agent` with Telegram-HTML-safe constraints
   3. Node: **Telegram (Send Message)** `Chat With User` sending `{{$json.output}}`

12) **Start/Inspiration branch (BrowserAct scrape)**
   1. Node: **Telegram (Send Message)** `Notify User`
   2. Node: **Google Sheets (Clear)** `Clear PromptHero Database` clearing `PromptHero!A2:F`
   3. Node: **BrowserAct** `Extract Top AI-Generated Images`
      - Type: WORKFLOW
      - Set your BrowserAct workflowId (must exist)
      - Timeout up to 7200 seconds if needed
   4. Node: **Code** `Splitting Generated Items`
      - Parse `$input.first().json.output.string` as JSON array
      - Return each element as an item
   5. Node: **Google Sheets (Append)** `Save Images To Database` mapping `model/prompt/image` → `Model/Ptompt/Image Url`
   6. Node: **Telegram** `Send Finishing Alert` with buttons callback `0` and `no`

13) **User uploads image (forensics path)**
   1. Node: **Google Sheets (Update)** `Set user State2` sets `UserState="dodge"`
   2. Node: **Telegram (File)** `Get Picture from User` using `message.photo.last().file_id`
   3. Node: **Google Sheets (Clear)** `Clear UserData` clears `UserImage` keeping header
   4. Node: **LangChain Agent (vision)** `Analyze the user image input`
      - Input: binary from the Telegram file node
      - Enable `passthroughBinaryImages`
      - Use **OpenRouter chat model** (`openai/gpt-4o`)
      - Attach **Structured Output** parser expecting `[{output:"..."}]`
   5. Node: **Google Sheets (Update)** `Update Image Descriptions` writes description into `UserImage` row 2
   6. Node: **Telegram** `Ask User for Continuation` with buttons callback `change` and `no`

14) **Prompt engineering & image generation**
   - **HeroPrompt path**
     1. **Get Prompt** (Google Sheets read `Current Image`)
     2. **Analyze the Chosen image** (vision agent using image URL) with OpenRouter model + structured parser
     3. **Generate Image From PromptHero** (Gemini 2.5 Pro) → outputs `{"Prompt":"..."}` (structured parser)
   - **UserImage path**
     1. **Get Description** (Google Sheets read `UserImage`)
     2. **Generate Image From UserData** (Gemini 2.5 Pro) → outputs `{"Prompt":"..."}` (structured parser)
   - Both connect to:
     1. **Generate Thumbnail** (Gemini image model `models/gemini-3-pro-image-preview`) using conditional prompt expression
     2. **Send Thumbnail Back to Chatbot** (Telegram sendPhoto, binary)
     3. **Set user State5** (Google Sheets update UserState=dodge)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Try It Out! AI Image Remix & Design Bot… Requirements: BrowserAct API (Template: Image Remix & Design Bot), Telegram Bot Token, Google Sheets tabs…, Google Gemini & OpenRouter credentials.” | From “Sticky Note - Intro” |
| Discord | https://discord.com/invite/UpnCKd7GaU |
| Blog | https://www.browseract.com/blog |
| How to Find Your BrowserAct API Key & Workflow ID | https://www.youtube.com/watch?v=pDjoZWEsZlE |
| How to Connect n8n to BrowserAct | https://www.youtube.com/watch?v=RoYMdJaRdcQ |
| How to Use & Customize BrowserAct Templates | https://www.youtube.com/watch?v=CPZHFUASncY |
| Embedded video reference | @[youtube](GqeKd9aYjW4) |
| Complexity warning: “This workflow is complex. Please proceed using the tutorial video.” | From “Sticky Note1” |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.