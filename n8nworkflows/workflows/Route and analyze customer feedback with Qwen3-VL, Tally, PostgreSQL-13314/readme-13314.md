Route and analyze customer feedback with Qwen3-VL, Tally, PostgreSQL

https://n8nworkflows.xyz/workflows/route-and-analyze-customer-feedback-with-qwen3-vl--tally--postgresql-13314


# Route and analyze customer feedback with Qwen3-VL, Tally, PostgreSQL

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Collect customer feedback from a Tally.so form (optionally with an image), run parallel AI analysis (sentiment, text category, image labels) using Qwen3-VL via an OpenAI-compatible endpoint, store enriched results in PostgreSQL, then route and post a formatted Discord embed to the right channel.

**Target use cases:**
- Automating triage of customer feedback/support tickets
- Prioritizing urgent complaints, surfacing praise, and capturing general inquiries
- Adding computer-vision context when customers attach photos (damage, wrong item, etc.)

### 1.1 Logical Blocks
1. **Input & Prep**: Receive Tally webhook; map raw answers into normalized fields; decide whether to fetch image binary.
2. **Multi-Track AI Analysis**: In parallel, compute sentiment, classify feedback category, and extract image keywords (if an attachment exists).
3. **Storage & Routing Logic**: Merge AI outputs, aggregate, save to PostgreSQL; run a decision LLM to choose Discord channel + craft a one-line summary.
4. **Discord Distribution**: Switch to the right Discord channel branch, build an embed payload, and send a Discord message.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Prep
**Overview:** Captures a Tally form submission, maps the form’s question IDs into human-readable fields, and conditionally fetches an attached image so downstream vision processing only runs when needed.

**Nodes involved:**
- Tally Trigger
- Field Mapping
- If
- Fetch Image
- Empty Keywords Handler
- Image Results Merge

#### Node: **Tally Trigger**
- **Type / role:** `tallyTrigger` (webhook trigger) — entry point for Tally submissions.
- **Configuration choices:**
  - Form ID: `xXVN5G`
  - `executeOnce: true` (useful for testing; prevents repeated runs from multiple webhook events)
  - `retryOnFail: true`, `alwaysOutputData: true`
- **Inputs/outputs:**
  - **Input:** none (trigger)
  - **Output:** raw Tally payload containing `question_*` fields.
- **Credentials:** Tally API credential (“Tally account”).
- **Edge cases / failures:**
  - Invalid/expired Tally token; webhook not registered; form ID mismatch.
  - Tally “private storage” image URLs may be short-lived; fetching later may fail.

#### Node: **Field Mapping**
- **Type / role:** `Set` — normalizes Tally question IDs to stable field names.
- **Key mappings (expressions):**
  - `Customer Name` ← `{{$json.question_NoKEOG}}`
  - `Email Address` ← `{{$json.question_qVxgN2}}`
  - `Message` ← `{{$json.question_QVgjEg}}`
  - `Image URL` ← `{{$json.question_9QLxxG}}`
- **Connections:**
  - **In:** Tally Trigger
  - **Out (parallel fan-out):** Sentiment Analysis, Text Classification, If, and also directly into AI Results Merge input #3 (index 2 in connections)
- **Edge cases:**
  - Missing/renamed Tally fields will result in empty values; downstream LLM prompts may receive empty text.

#### Node: **If**
- **Type / role:** `If` — checks whether an image URL was provided.
- **Condition:**
  - `{{$json["Image URL"]}}` **is not empty**
- **Connections:**
  - **True:** Fetch Image
  - **False:** Empty Keywords Handler
- **Edge cases:**
  - Image URL may exist but be invalid/expired (true branch proceeds, then Fetch Image fails).

#### Node: **Fetch Image**
- **Type / role:** `HTTP Request` — downloads the image as binary for vision processing.
- **Configuration choices:**
  - URL: `{{$json["Image URL"]}}`
  - Authentication: “predefinedCredentialType” using `tallyApi` (unusual for generic image download; works only if Tally credential is accepted/needed)
- **Connections:**
  - **In:** If (true branch)
  - **Out:** Image Keyword Extraction (Code)
- **Failures:**
  - 403/404 if signed URL expired; non-image content; large file timeouts.
  - If binary is not present as `binary.data`, the downstream code node will throw.

#### Node: **Empty Keywords Handler**
- **Type / role:** `Set` — produces a fallback image keyword payload when no image exists.
- **Output field:**
  - `keywords` set to string value `=[]` (note: stored as a string, not an actual array)
- **Connections:**
  - **In:** If (false branch)
  - **Out:** Image Results Merge
- **Edge cases:**
  - Since `keywords` is a string literal, later DB insert into an array column may fail depending on PostgreSQL type casting and n8n settings. Consider setting it as an actual array instead.

#### Node: **Image Results Merge**
- **Type / role:** `Merge` — merges either vision keywords OR empty keywords.
- **Configuration:** default merge behavior (n8n Merge v3.2).
- **Connections:**
  - **In1:** Image Keyword Extraction
  - **In2:** Empty Keywords Handler
  - **Out:** AI Results Merge input #4 (index 3)
- **Edge cases:**
  - If both branches somehow produce items (mis-execution), merge behavior may duplicate/conflict.

---

### Block 2 — Multi-Track AI Analysis
**Overview:** Runs sentiment labeling and category classification on the text, and extracts predefined image labels via a vision-capable model. Each track is validated with structured parsers (except vision which parses JSON in code).

**Nodes involved:**
- Sentiment LLM
- Sentiment Analysis
- Sentiment Parser
- Classification LLM
- Text Classification
- Classification Parser
- Image Keyword Extraction
- AI Results Merge

#### Node: **Sentiment LLM**
- **Type / role:** `lmChatOpenAi` — provides the model backend for Sentiment Analysis chain.
- **Configuration:**
  - Model: `qwen/qwen3-vl-4b`
  - Temperature: `0.3`
  - “responsesApiEnabled”: false
- **Credentials:** OpenAI API credential (“OpenAi account”), expected to point to a **local OpenAI-compatible API** per sticky note (LM Studio / similar).
- **Connections:**
  - **Out (ai_languageModel):** Sentiment Analysis
- **Failures:**
  - Incorrect base URL in credential, model not loaded, 404 model not found, timeouts.

#### Node: **Sentiment Analysis**
- **Type / role:** `chainLlm` — prompt-based sentiment classifier.
- **Input text:** `{{$json.Message}}`
- **Prompt intent:** Return exactly one label: `Positive|Negative|Neutral`, with special rule: ambiguous/empty/punctuation-only ⇒ Neutral.
- **Output parsing:** `hasOutputParser: true` with Sentiment Parser.
- **Connections:**
  - **In:** Field Mapping
  - **ai_languageModel:** Sentiment LLM
  - **ai_outputParser:** Sentiment Parser
  - **Out:** AI Results Merge input #1 (index 0)
- **Edge cases:**
  - If the model returns extra text, parser will fail.
  - Empty message handled by prompt rule, but only if model complies.

#### Node: **Sentiment Parser**
- **Type / role:** `outputParserStructured` — enforces schema: `{ sentiment: "Positive"|"Negative"|"Neutral" }`.
- **Failures:**
  - Any deviation from enum causes parse error and stops the chain.

#### Node: **Classification LLM**
- **Type / role:** `lmChatOpenAi` — model backend for Text Classification chain.
- **Configuration:** same as Sentiment LLM (Qwen3-VL-4B, temp 0.3).
- **Connections:** `ai_languageModel` → Text Classification
- **Failures:** same as other LLM nodes.

#### Node: **Text Classification**
- **Type / role:** `chainLlm` — categorizes feedback into exactly one of 8 categories.
- **Input text:** `{{$json.Message}}`
- **Output parsing:** `hasOutputParser: true` with Classification Parser.
- **Connections:**
  - **In:** Field Mapping
  - **ai_languageModel:** Classification LLM
  - **ai_outputParser:** Classification Parser
  - **Out:** AI Results Merge input #2 (index 1)
- **Edge cases:**
  - Strong instruction “no explanation” reduces parse errors; still possible if model drifts.

#### Node: **Classification Parser**
- **Type / role:** `outputParserStructured` — enforces `category` in a fixed enum of 8 categories.
- **Failures:** invalid category → parse error.

#### Node: **Image Keyword Extraction**
- **Type / role:** `Code` — sends base64 image to a local OpenAI-compatible `/v1/chat/completions` endpoint and forces JSON-schema output with predefined labels.
- **Key implementation details:**
  - Reads binary from: `$input.first().binary.data`
  - Constructs data URL: ``data:${mime_type};base64,${img_data}``
  - Calls: `http://192.168.56.1:1234/v1/chat/completions`
  - Uses `response_format: json_schema` with strict schema:
    - Output: `{ "keywords": [<one or more predefined labels>] }`
  - Parses: `JSON.parse(response_data.choices[0].message.content)`
- **Connections:**
  - **In:** Fetch Image
  - **Out:** Image Results Merge
- **Version-specific notes:**
  - Requires Code node v2 behavior and availability of `axios` in n8n’s runtime (commonly available in n8n Code node, but can vary by deployment).
- **Failures / edge cases:**
  - If Fetch Image does not produce `binary.data`, code throws.
  - If local endpoint does not support `response_format: json_schema`, model may return non-JSON; `JSON.parse` throws.
  - Large images may exceed server limits; consider resizing or using multipart uploads.

#### Node: **AI Results Merge**
- **Type / role:** `Merge` (multi-input) — collects outputs from sentiment, classification, original mapped fields, and image keywords into one stream.
- **Configuration:** `numberInputs: 4`
- **Inputs (as wired):**
  1. Sentiment Analysis output
  2. Text Classification output
  3. Field Mapping output (raw mapped data)
  4. Image Results Merge output
- **Output:** to Data Aggregation
- **Edge cases:**
  - This pattern depends on all four inputs arriving as expected. If any upstream branch errors, merge may stall or produce partial data.

---

### Block 3 — Storage & Routing Logic
**Overview:** Aggregates the merged multi-track data, inserts a row into PostgreSQL, then uses an LLM decision chain to select a Discord channel and produce a one-line routing message.

**Nodes involved:**
- Data Aggregation
- Save to PostgreSQL
- Routing LLM
- Route Parser
- Decision Logic

#### Node: **Data Aggregation**
- **Type / role:** `Aggregate` — aggregates “all item data” into a single structure for easier referencing.
- **Configuration:** `aggregateAllItemData`
- **Connections:**
  - **In:** AI Results Merge
  - **Out:** Save to PostgreSQL
- **Output shape:** `$('Data Aggregation').item.json.data[...]` is used downstream.
- **Edge cases:**
  - The workflow later relies on fixed indexes (0..3) to refer to sentiment/classification/mapping/image results; if merge ordering changes, DB mapping breaks.

#### Node: **Save to PostgreSQL**
- **Type / role:** `Postgres` — inserts feedback + AI enrichment into `public.customer_feedback`.
- **Configuration choices:**
  - Table: `customer_feedback` (schema `public`)
  - Mapping mode: “define below”
  - `attemptToConvertTypes: false` (type mismatches will error instead of being coerced)
- **Column mappings (key expressions):**
  - `name` = `$('Data Aggregation').item.json.data[2]['Customer Name']`
  - `email_address` = `...data[2]['Email Address']`
  - `feedback_message` = `...data[2]['Message']`
  - `image_url` = `...data[2]['Image URL']`
  - `sentiment` = `...data[0].output.sentiment`
  - `category` = `...data[1].output.category`
  - `img_keywords` = `...data[3].keywords`
- **Credentials:** Postgres credential (“Postgres account”).
- **Failures / edge cases:**
  - `img_keywords` column is configured as `array`; but “Empty Keywords Handler” emits `keywords` as a **string** (`"=[]"`). With `attemptToConvertTypes: false`, inserts may fail.
  - If `image_url` is empty string rather than NULL, downstream embed logic checks only `!== null`, so it may attach an empty image URL.
  - If the table schema differs (e.g., `img_keywords` is JSONB), mapping must be adjusted.

#### Node: **Routing LLM**
- **Type / role:** `lmChatOpenAi` — model backend for the Decision Logic chain.
- **Configuration:** model `qwen/qwen3-vl-4b`, temperature 0.3.
- **Connections:** `ai_languageModel` → Decision Logic
- **Failures:** same as other LLM nodes.

#### Node: **Route Parser**
- **Type / role:** `outputParserStructured` — intended to enforce routing output schema.
- **Schema issue (important):**
  - `properties` defines `category` and `message`
  - but `required` specifies `["channel_name","message"]`
  - and `additionalProperties: false`
  - This schema is internally inconsistent: it requires `channel_name` but does not allow it in `properties`, so parsing will fail if strict.
- **Connections:** `ai_outputParser` → Decision Logic
- **Recommended fix:** Change schema to:
  - either rename `category` to `channel_name`, or change `required` to `["category","message"]`, and align downstream Switch checks accordingly.

#### Node: **Decision Logic**
- **Type / role:** `chainLlm` — chooses the Discord channel and crafts a one-line summary.
- **Input text (constructed JSON-like string):**
  - Uses variables such as `$json.name`, `$json.email_address`, `$json.feedback_message`, `$json.sentiment`, `$json.category`, `$json.img_keywords`
- **Prompt intent:**
  - Decide which channel among `#general-inquiries`, `#support-urgent`, `#happy-customers`
  - Produce a concise, informative one-line message for posting
- **Connections:**
  - **In:** Save to PostgreSQL
  - **ai_languageModel:** Routing LLM
  - **ai_outputParser:** Route Parser
  - **Out:** Channel Router
- **Edge cases:**
  - If Route Parser fails (schema mismatch), this node fails and Discord routing stops.
  - The prompt describes “channel_name” conceptually, but downstream uses `$json.output.category` for switching.

---

### Block 4 — Discord Distribution
**Overview:** Switches based on the chosen channel, injects per-channel embed styling (title/description/color/channel_id), merges with decision output, formats a Discord embed object, and posts it.

**Nodes involved:**
- Channel Router
- #general-inquiries
- #happy-customers
- #support-urgent
- Build Discord Message
- Format Embed Data
- Send Discord Notification

#### Node: **Channel Router**
- **Type / role:** `Switch` — routes to one of three branches.
- **Rules (all compare `{{$json.output.category}}`):**
  - equals `#general-inquiries`
  - equals `#happy-customers`
  - equals `#support-urgent`
- **Connections:**
  - **In:** Decision Logic
  - **Out:** three Set nodes (#general-inquiries / #happy-customers / #support-urgent)
- **Failures / edge cases:**
  - If Decision Logic output uses a different field name (e.g., `channel_name`), routing will not match.
  - No default case configured; unexpected values drop the item.

#### Node: **#general-inquiries**
- **Type / role:** `Set` — defines embed styling + Discord channel ID.
- **Outputs:**
  - `title`: “🔹 General Feedback Alert”
  - `description`: “A new general feedback or inquiry has been received.”
  - `color`: `11393254`
  - `channel_id`: `1408795476555206827`
- **Connections:** to Build Discord Message (input 1)

#### Node: **#happy-customers**
- **Type / role:** `Set`
- **Outputs:**
  - `title`: “🎉 Happy Customer Alert”
  - `description`: “Positive feedback from a satisfied customer.”
  - `color`: `4682522`
  - `channel_id`: `1408795413766733844`
- **Connections:** to Build Discord Message (input 2)

#### Node: **#support-urgent**
- **Type / role:** `Set`
- **Outputs:**
  - `title`: “⚠️ Urgent Support Required”
  - `description`: “Immediate attention needed for this customer feedback.”
  - `color`: `16711680`
  - `channel_id`: `1408795367281131621`
- **Connections:** to Build Discord Message (input 3)

#### Node: **Build Discord Message**
- **Type / role:** `Merge` (3 inputs) — combines the selected channel styling with other required data for formatting.
- **Configuration:** `numberInputs: 3`
- **Connections:**
  - **In:** from one of the 3 channel Set nodes (but the merge expects 3 inputs; in practice, this can be fragile unless the other inputs are also connected)
  - **Out:** Format Embed Data
- **Important wiring observation:**
  - In the provided connections, only the three Set nodes connect to the three merge inputs. That means only **one** input receives data per execution, while the other two inputs are empty. Depending on merge mode defaults, this may or may not output anything.
  - If this works in your environment, it’s likely because Merge node is configured to pass through incoming data in a mode that tolerates missing inputs; otherwise, add explicit connections for the other required streams (e.g., Decision Logic output, Save to PostgreSQL output).
  
#### Node: **Format Embed Data**
- **Type / role:** `Code` — constructs final Discord embed JSON.
- **Key logic:**
  - Reads `channel_id`, `title`, `description`, `color` from the merge input.
  - Builds `embed.fields` using data from:
    - `$('Save to PostgreSQL').first().json.*` (name/email/feedback/image_url)
    - `$('Decision Logic').first().json.output.message` (model’s one-line analysis)
  - Adds `embed.image` if `image_url !== null`
- **Connections:**
  - **In:** Build Discord Message
  - **Out:** Send Discord Notification
- **Edge cases:**
  - Uses cross-node references (`$('Save to PostgreSQL')...`). If the workflow runs with multiple items/batches, `.first()` may mismatch the current item.
  - `image_url` check should also exclude empty string.

#### Node: **Send Discord Notification**
- **Type / role:** `Discord` — posts a message to a channel with embeds.
- **Configuration choices:**
  - Resource: `message`
  - Guild: `Artisan Corner` (ID `1408795203615326341`)
  - Channel ID: expression `{{$json.channel_id}}`
  - Embeds JSON: `{{ JSON.stringify($json.embed) }}`
- **Credentials:** Discord Bot token (“Discord Bot account”).
- **Failures:**
  - Missing permissions in channel; invalid channel ID; embed payload invalid; rate limiting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Tally Trigger | n8n-nodes-tallyforms.tallyTrigger | Entry webhook from Tally form | — | Field Mapping | ## 1. Input & Prep; Captures the webhook from Tally and maps the fields. The **Fetch Image** branch ensures binary data is handled only if an attachment exists. |
| Field Mapping | n8n-nodes-base.set | Normalize Tally fields to stable names | Tally Trigger | Sentiment Analysis; If; Text Classification; AI Results Merge | ## 1. Input & Prep; Captures the webhook from Tally and maps the fields. The **Fetch Image** branch ensures binary data is handled only if an attachment exists. |
| If | n8n-nodes-base.if | Check if Image URL exists | Field Mapping | Fetch Image (true); Empty Keywords Handler (false) | ## 1. Input & Prep; Captures the webhook from Tally and maps the fields. The **Fetch Image** branch ensures binary data is handled only if an attachment exists. |
| Fetch Image | n8n-nodes-base.httpRequest | Download image as binary | If (true) | Image Keyword Extraction | ## 1. Input & Prep; Captures the webhook from Tally and maps the fields. The **Fetch Image** branch ensures binary data is handled only if an attachment exists. |
| Image Keyword Extraction | n8n-nodes-base.code | Vision inference + keyword labels via local OpenAI-compatible API | Fetch Image | Image Results Merge | ## 2. Multi-Track AI Analysis; Parallel processing for sentiment, category, and visual content. **AI Results Merge** combines these three streams into a single JSON object. |
| Empty Keywords Handler | n8n-nodes-base.set | Fallback keywords when no image | If (false) | Image Results Merge | ## 1. Input & Prep; Captures the webhook from Tally and maps the fields. The **Fetch Image** branch ensures binary data is handled only if an attachment exists. |
| Image Results Merge | n8n-nodes-base.merge | Merge “vision keywords” vs “no image” result | Image Keyword Extraction; Empty Keywords Handler | AI Results Merge | ## 2. Multi-Track AI Analysis; Parallel processing for sentiment, category, and visual content. **AI Results Merge** combines these three streams into a single JSON object. |
| Sentiment LLM | @n8n/n8n-nodes-langchain.lmChatOpenAi | Model backend for sentiment chain | — | Sentiment Analysis (ai_languageModel) | ### ⚠️ Local Inference Setup; The nodes **Sentiment LLM**, **Classification LLM**, **Image Keyword Extraction**, and **Routing LLM** all point to a local OpenAI-compatible API. If running n8n in Docker, do **not** use `localhost`. Use your machine's internal IP address to allow the container to communicate with LM Studio. |
| Sentiment Analysis | @n8n/n8n-nodes-langchain.chainLlm | Sentiment classification (Positive/Negative/Neutral) | Field Mapping | AI Results Merge | ## 2. Multi-Track AI Analysis; Parallel processing for sentiment, category, and visual content. **AI Results Merge** combines these three streams into a single JSON object. |
| Sentiment Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce sentiment enum schema | — | Sentiment Analysis (ai_outputParser) | ## 2. Multi-Track AI Analysis; Parallel processing for sentiment, category, and visual content. **AI Results Merge** combines these three streams into a single JSON object. |
| Classification LLM | @n8n/n8n-nodes-langchain.lmChatOpenAi | Model backend for classification chain | — | Text Classification (ai_languageModel) | ### ⚠️ Local Inference Setup; The nodes **Sentiment LLM**, **Classification LLM**, **Image Keyword Extraction**, and **Routing LLM** all point to a local OpenAI-compatible API. If running n8n in Docker, do **not** use `localhost`. Use your machine's internal IP address to allow the container to communicate with LM Studio. |
| Text Classification | @n8n/n8n-nodes-langchain.chainLlm | Classify feedback into 1 of 8 categories | Field Mapping | AI Results Merge | ## 2. Multi-Track AI Analysis; Parallel processing for sentiment, category, and visual content. **AI Results Merge** combines these three streams into a single JSON object. |
| Classification Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce category enum schema | — | Text Classification (ai_outputParser) | ## 2. Multi-Track AI Analysis; Parallel processing for sentiment, category, and visual content. **AI Results Merge** combines these three streams into a single JSON object. |
| AI Results Merge | n8n-nodes-base.merge | Combine sentiment + classification + mapped fields + image keywords | Sentiment Analysis; Text Classification; Field Mapping; Image Results Merge | Data Aggregation | ## 2. Multi-Track AI Analysis; Parallel processing for sentiment, category, and visual content. **AI Results Merge** combines these three streams into a single JSON object. |
| Data Aggregation | n8n-nodes-base.aggregate | Aggregate merged data for stable referencing | AI Results Merge | Save to PostgreSQL | ## 3. Storage & Logic; The aggregated data is committed to the database. The **Decision Logic** and **Routing LLM** then determine which team needs to see this specific feedback. |
| Save to PostgreSQL | n8n-nodes-base.postgres | Persist enriched feedback into DB | Data Aggregation | Decision Logic | ## 3. Storage & Logic; The aggregated data is committed to the database. The **Decision Logic** and **Routing LLM** then determine which team needs to see this specific feedback. |
| Routing LLM | @n8n/n8n-nodes-langchain.lmChatOpenAi | Model backend for decision routing | — | Decision Logic (ai_languageModel) | ### ⚠️ Local Inference Setup; The nodes **Sentiment LLM**, **Classification LLM**, **Image Keyword Extraction**, and **Routing LLM** all point to a local OpenAI-compatible API. If running n8n in Docker, do **not** use `localhost`. Use your machine's internal IP address to allow the container to communicate with LM Studio. |
| Route Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce routing output schema (currently inconsistent) | — | Decision Logic (ai_outputParser) | ## 3. Storage & Logic; The aggregated data is committed to the database. The **Decision Logic** and **Routing LLM** then determine which team needs to see this specific feedback. |
| Decision Logic | @n8n/n8n-nodes-langchain.chainLlm | Decide channel + craft one-line summary | Save to PostgreSQL | Channel Router | ## 3. Storage & Logic; The aggregated data is committed to the database. The **Decision Logic** and **Routing LLM** then determine which team needs to see this specific feedback. |
| Channel Router | n8n-nodes-base.switch | Route to correct Discord branch by channel name | Decision Logic | #general-inquiries; #happy-customers; #support-urgent | ## 4. Discord Distribution; The **Channel Router** sends the data to the correct branch where the final message is formatted as a rich embed for Discord. |
| #general-inquiries | n8n-nodes-base.set | Set embed styling + channel ID | Channel Router | Build Discord Message | ## 4. Discord Distribution; The **Channel Router** sends the data to the correct branch where the final message is formatted as a rich embed for Discord. |
| #happy-customers | n8n-nodes-base.set | Set embed styling + channel ID | Channel Router | Build Discord Message | ## 4. Discord Distribution; The **Channel Router** sends the data to the correct branch where the final message is formatted as a rich embed for Discord. |
| #support-urgent | n8n-nodes-base.set | Set embed styling + channel ID | Channel Router | Build Discord Message | ## 4. Discord Distribution; The **Channel Router** sends the data to the correct branch where the final message is formatted as a rich embed for Discord. |
| Build Discord Message | n8n-nodes-base.merge | Combine channel styling and other payload parts | #general-inquiries; #happy-customers; #support-urgent | Format Embed Data | ## 4. Discord Distribution; The **Channel Router** sends the data to the correct branch where the final message is formatted as a rich embed for Discord. |
| Format Embed Data | n8n-nodes-base.code | Build Discord embed object | Build Discord Message | Send Discord Notification | ## 4. Discord Distribution; The **Channel Router** sends the data to the correct branch where the final message is formatted as a rich embed for Discord. |
| Send Discord Notification | n8n-nodes-base.discord | Post embed message into Discord channel | Format Embed Data | — | ## 4. Discord Distribution; The **Channel Router** sends the data to the correct branch where the final message is formatted as a rich embed for Discord. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block | — | — | ## 4. Discord Distribution; The **Channel Router** sends the data to the correct branch where the final message is formatted as a rich embed for Discord. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment block | — | — | ## 3. Storage & Logic; The aggregated data is committed to the database. The **Decision Logic** and **Routing LLM** then determine which team needs to see this specific feedback. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment block | — | — | ## 2. Multi-Track AI Analysis; Parallel processing for sentiment, category, and visual content. **AI Results Merge** combines these three streams into a single JSON object. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment block | — | — | ## 1. Input & Prep; Captures the webhook from Tally and maps the fields. The **Fetch Image** branch ensures binary data is handled only if an attachment exists. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment block | — | — | ### ⚠️ Local Inference Setup; The nodes **Sentiment LLM**, **Classification LLM**, **Image Keyword Extraction**, and **Routing LLM** all point to a local OpenAI-compatible API. If running n8n in Docker, do **not** use `localhost`. Use your machine's internal IP address to allow the container to communicate with LM Studio. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment block | — | — | ### How it works; This workflow provides end-to-end automation for customer feedback… (includes Setup steps/Customization text as provided). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger: Tally Trigger**
   1. Add node **Tally Trigger**.
   2. Set **Form ID** to `xXVN5G`.
   3. Connect **Tally API** credential.

2. **Add Set node: Field Mapping**
   1. Add **Set** node named “Field Mapping”.
   2. Add fields:
      - `Customer Name` = `{{$json.question_NoKEOG}}`
      - `Email Address` = `{{$json.question_qVxgN2}}`
      - `Message` = `{{$json.question_QVgjEg}}`
      - `Image URL` = `{{$json.question_9QLxxG}}`
   3. Connect: **Tally Trigger → Field Mapping**

3. **Conditional image branch**
   1. Add **If** node.
      - Condition: `{{$json["Image URL"]}}` → **String** → **is not empty**
   2. Connect: **Field Mapping → If**
   3. **True branch:** add **HTTP Request** node “Fetch Image”
      - URL: `{{$json["Image URL"]}}`
      - Ensure it outputs **binary** (default behavior usually does when “Download” is enabled; if not, enable “Download”/“Response: File” depending on n8n version).
      - (Optional) remove Tally auth unless required for the URL.
      - Connect: **If(true) → Fetch Image**
   4. **False branch:** add **Set** node “Empty Keywords Handler”
      - Preferred: set `keywords` to an **array** `[]` (not a string) to match Postgres array typing.
      - Connect: **If(false) → Empty Keywords Handler**

4. **Vision processing**
   1. Add **Code** node “Image Keyword Extraction”
      - Implement call to your local OpenAI-compatible endpoint (`/v1/chat/completions`) and parse `{keywords:[...]}`.
      - Ensure it reads binary from the same property your HTTP Request produces (commonly `binary.data`).
   2. Connect: **Fetch Image → Image Keyword Extraction**
   3. Add **Merge** node “Image Results Merge”
      - Connect **Image Keyword Extraction → Image Results Merge (Input 1)**
      - Connect **Empty Keywords Handler → Image Results Merge (Input 2)**

5. **Set up local LLM credentials**
   1. Create/adjust an **OpenAI API** credential to point to your **local OpenAI-compatible base URL** (example from workflow: `http://192.168.56.1:1234/v1`).
   2. If n8n runs in Docker: use your host machine LAN/IP, not `localhost`.

6. **Sentiment track**
   1. Add **OpenAI Chat Model** node (LangChain) “Sentiment LLM”
      - Model: `qwen/qwen3-vl-4b`, temperature 0.3
      - Use the local OpenAI credential
   2. Add **Chain LLM** node “Sentiment Analysis”
      - Text: `{{$json.Message}}`
      - Prompt: sentiment rules (Positive/Negative/Neutral only)
      - Enable output parser
   3. Add **Structured Output Parser** “Sentiment Parser”
      - Schema: `{ sentiment: enum[Positive,Negative,Neutral] }`
   4. Wire:
      - **Field Mapping → Sentiment Analysis**
      - **Sentiment LLM → Sentiment Analysis (ai_languageModel)**
      - **Sentiment Parser → Sentiment Analysis (ai_outputParser)**

7. **Classification track**
   1. Add model node “Classification LLM” (same settings/model/credential).
   2. Add chain node “Text Classification”
      - Text: `{{$json.Message}}`
      - Prompt: choose exactly one of the 8 categories
      - Enable output parser
   3. Add parser “Classification Parser” enforcing the category enum.
   4. Wire:
      - **Field Mapping → Text Classification**
      - **Classification LLM → Text Classification (ai_languageModel)**
      - **Classification Parser → Text Classification (ai_outputParser)**

8. **Merge AI streams**
   1. Add **Merge** node “AI Results Merge” with **4 inputs**.
   2. Connect:
      - **Sentiment Analysis → AI Results Merge (Input 1)**
      - **Text Classification → AI Results Merge (Input 2)**
      - **Field Mapping → AI Results Merge (Input 3)**
      - **Image Results Merge → AI Results Merge (Input 4)**

9. **Aggregate for DB insert**
   1. Add **Aggregate** node “Data Aggregation”
      - Mode: aggregate all item data
   2. Connect: **AI Results Merge → Data Aggregation**

10. **PostgreSQL storage**
   1. Add **Postgres** node “Save to PostgreSQL”
      - Schema: `public`
      - Table: `customer_feedback`
      - Map columns (either by field references or, preferably, by restructuring data before insert to avoid index-based references).
   2. Create table with columns consistent with:
      - `name` (text), `email_address` (text), `feedback_message` (text), `image_url` (text nullable),
      - `sentiment` (text), `category` (text), `img_keywords` (text[] or jsonb)
   3. Connect Postgres credential.
   4. Connect: **Data Aggregation → Save to PostgreSQL**

11. **Decision routing LLM**
   1. Add model node “Routing LLM” (same local OpenAI credential/model).
   2. Add chain node “Decision Logic”
      - Provide a structured input string including name/email/message/sentiment/category/img_keywords.
      - Prompt: choose channel (#general-inquiries / #support-urgent / #happy-customers) and craft one-line summary.
      - Enable output parser.
   3. Add parser node “Route Parser”
      - **Fix schema** so it matches what you want downstream. Example:
        - Output: `{ "category": "#support-urgent", "message": "..." }`
        - Required: `["category","message"]`
   4. Wire:
      - **Save to PostgreSQL → Decision Logic**
      - **Routing LLM → Decision Logic (ai_languageModel)**
      - **Route Parser → Decision Logic (ai_outputParser)**

12. **Discord routing and formatting**
   1. Add **Switch** node “Channel Router”
      - Switch value: `{{$json.output.category}}`
      - Rules: equals `#general-inquiries`, equals `#happy-customers`, equals `#support-urgent`
   2. Add three **Set** nodes named exactly:
      - “#general-inquiries” with `title/description/color/channel_id`
      - “#happy-customers” with `title/description/color/channel_id`
      - “#support-urgent” with `title/description/color/channel_id`
   3. Connect: **Decision Logic → Channel Router**, then each output to the matching Set node.

13. **Build embed and send**
   1. Add **Code** node “Format Embed Data”
      - Build an `embed` object with title/description/color, fields for customer info and feedback, and include Decision Logic summary.
      - Set output to `{ channel_id, embed }`
      - Improve image condition: only attach if `image_url` is non-empty.
   2. Add **Discord** node “Send Discord Notification”
      - Guild: your server
      - Channel ID: `{{$json.channel_id}}`
      - Embeds: JSON stringified `{{$json.embed}}`
      - Add Discord bot credential (bot must have access to target channels).
   3. Connect: **each Set node → (directly) Format Embed Data** (recommended), then **Format Embed Data → Send Discord Notification**.
      - If you keep the “Build Discord Message” Merge node, ensure it receives all required inputs consistently; otherwise remove it and connect Set → Format directly.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Local inference requirement: Sentiment LLM, Classification LLM, Image Keyword Extraction, Routing LLM point to a local OpenAI-compatible API; in Docker do not use `localhost` | From sticky note “### ⚠️ Local Inference Setup” |
| Setup steps mention LM Studio + model name `Qwen2-VL-7B-Instruct`, while workflow uses `qwen/qwen3-vl-4b` | From sticky note “### How it works”; reconcile model naming to avoid “model not found” |
| Ensure Postgres table matches expected schema and types (especially `img_keywords` array vs JSONB and empty array handling) | From sticky note “### How it works” + observed mapping behavior |
| Discord channel IDs must be customized for your server | From sticky note “### How it works” |
| Prompts for Sentiment Analysis / Text Classification are designed to minimize extra text and enforce strict labels | From the chain prompts in nodes |

If you want, I can propose a corrected “Route Parser” schema and a safer, non-index-based data structure (so the Postgres insert doesn’t depend on `data[0]..data[3]`).