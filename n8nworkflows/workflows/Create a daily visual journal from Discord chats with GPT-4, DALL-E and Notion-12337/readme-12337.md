Create a daily visual journal from Discord chats with GPT-4, DALL-E and Notion

https://n8nworkflows.xyz/workflows/create-a-daily-visual-journal-from-discord-chats-with-gpt-4--dall-e-and-notion-12337


# Create a daily visual journal from Discord chats with GPT-4, DALL-E and Notion

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create a daily visual journal from Discord chats with GPT-4, DALL-E and Notion

**Purpose:**  
This workflow generates a daily “visual journal” entry in Notion from a Discord channel’s messages. It runs on a daily schedule, fetches Discord messages, filters those from the current day, summarizes and structures them via OpenAI Chat Completions (GPT-4o) into JSON, generates an image with DALL·E 3, uploads the image to Cloudinary to obtain a public URL, and finally creates a Notion database page.

**Target use cases:**
- Personal journaling from community or friend-group Discord discussions
- Daily reflection logs for teams or study groups
- Automated “day recap” pages in Notion with an illustrative image

### Logical blocks
**1.1 Scheduling & Message Retrieval**
- Runs daily and pulls Discord channel messages.

**1.2 Day Filtering & Guard Clause**
- Filters only today’s messages and stops the pipeline if none exist.

**1.3 Conversation Preparation**
- Formats messages into a structured text transcript and participant stats.

**1.4 AI Analysis (GPT-4o)**
- Produces a structured diary entry payload (title, summary, mood, tags, highlights, image prompt) using JSON Schema output.

**1.5 Image Generation & Hosting**
- Generates an image via DALL·E 3 and uploads it to Cloudinary (for a public URL usable by Notion).

**1.6 Notion Entry Creation & Final Response**
- Creates a Notion database page and outputs a success payload (or a “no messages” payload).

---

## 2. Block-by-Block Analysis

### 1.1 Scheduling & Message Retrieval
**Overview:** Triggers once per day and retrieves messages from a specified Discord channel using the official Discord node.

**Nodes involved:**
- Daily Trigger (11pm)
- Get Discord Messages

#### Node: Daily Trigger (11pm)
- **Type / role:** `Schedule Trigger` — workflow entry point, time-based execution.
- **Configuration (interpreted):** Runs every 24 hours (interval-based schedule). Despite the name “11pm”, the JSON shows an *every 24 hours* interval; the actual start time depends on when it was activated or configured in UI.
- **Inputs / outputs:** No inputs. Output → **Get Discord Messages**.
- **Edge cases / failures:**
  - Timezone differences (n8n instance timezone vs expected local timezone).
  - Interval scheduling may drift vs a strict “at 23:00 daily” cron rule.

#### Node: Get Discord Messages
- **Type / role:** `Discord` node (`resource: message`, `operation: getAll`) — fetches messages.
- **Configuration (interpreted):**
  - Uses Discord Bot credentials (preconfigured).
  - `guildId` and `channelId` are set via UI lists but currently blank in the JSON export (must be selected).
  - Default options; likely fetches the most recent batch (Discord node typically supports limits/pagination; not explicitly set here).
- **Connections:** Input from **Daily Trigger (11pm)** → Output to **Filter Today's Messages**.
- **Edge cases / failures:**
  - Missing/invalid bot token or insufficient permissions (needs *Read Message History*).
  - Channel not selected (blank `guildId`/`channelId`).
  - Rate limiting or pagination limitations (may not cover an entire day if the channel is very active).

---

### 1.2 Day Filtering & Guard Clause
**Overview:** Converts the Discord message list into a “today-only” bundle and branches: proceed if there are messages, otherwise stop with a “No Messages” result.

**Nodes involved:**
- Filter Today's Messages
- Has Messages?
- No Messages Today

#### Node: Filter Today's Messages
- **Type / role:** `Code` — filters and normalizes message data.
- **Configuration choices:**
  - Reads all incoming items: `const allMessages = $input.all();`
  - Defines “today” using local server time:
    - `todayStart` = 00:00:00 local
    - `todayEnd` = 23:59:59 local
  - Filters by `item.json.timestamp` parsed as `Date`.
  - Maps each message to a reduced schema: `id, content, author, timestamp, attachments`.
  - Reverses array to chronological order (assumes Discord returns newest-first).
  - Outputs a **single item**:  
    `json: { messages: [...], messageCount, date: YYYY-MM-DD, fetchedAt }`
- **Connections:** Input from **Get Discord Messages** → Output to **Has Messages?**
- **Edge cases / failures:**
  - Timestamp format differences/timezone mismatches (Discord timestamps are ISO; comparisons are in local timezone).
  - If Discord node outputs nested structures or nonstandard timestamp fields, filtering could break.
  - If the Discord node outputs multiple pages/items, `$input.all()` is correct; output is collapsed into one item.

#### Node: Has Messages?
- **Type / role:** `IF` — branch based on message count.
- **Condition:** `{{ $json.messageCount }} > 0`
- **Outputs:**
  - **True** → Format for GPT-4
  - **False** → No Messages Today
- **Edge cases / failures:**
  - If `messageCount` is missing or not numeric, strict validation may fail or route incorrectly.

#### Node: No Messages Today
- **Type / role:** `Code` — returns a graceful “skip” result.
- **Behavior:** Outputs:
  - `success: false`
  - `message: 'No messages found for today. Skipping diary entry.'`
  - `date` derived from input or current date.
- **Connections:** Input from **Has Messages? (false)**; no further outputs.
- **Edge cases / failures:** Minimal; depends on upstream `date` existence.

---

### 1.3 Conversation Preparation
**Overview:** Builds a chat transcript string and basic stats for GPT analysis.

**Nodes involved:**
- Format for GPT-4

#### Node: Format for GPT-4
- **Type / role:** `Code` — creates a clean prompt payload.
- **Key logic:**
  - Expects `messages` array from previous node.
  - Builds `conversationText` in format:  
    `[HH:MM] Author: message`
  - Extracts unique participants and per-author message counts.
  - Outputs: `conversationText, participants[], messageCounts{}, totalMessages, date, dateFormatted`
- **Connections:** Input from **Has Messages? (true)** → Output to **GPT-4 Analyze Day**
- **Edge cases / failures:**
  - Empty messages array: returns `{ error: 'No messages to process' }` but the workflow still proceeds to GPT unless additional guard exists (none here). This can cause GPT call to be nonsensical or fail due to missing fields.

---

### 1.4 AI Analysis (GPT-4o)
**Overview:** Calls OpenAI Chat Completions (gpt-4o) and forces structured JSON output via JSON Schema.

**Nodes involved:**
- GPT-4 Analyze Day
- Parse Analysis

#### Node: GPT-4 Analyze Day
- **Type / role:** `HTTP Request` — OpenAI Chat Completions API call.
- **Authentication:** `predefinedCredentialType` with `openAiApi` credentials in n8n.
- **Request:**
  - `POST https://api.openai.com/v1/chat/completions`
  - Model: `gpt-4o`
  - System prompt: journaling assistant tone.
  - User prompt: includes `dateFormatted`, `conversationText`, participants list, total message count.
  - Uses `response_format: json_schema` with `strict: true` and required fields:
    - `title` (string)
    - `summary` (string)
    - `mood` (string)
    - `tags` (string[])
    - `imagePrompt` (string)
    - `highlights` (string[])
- **Connections:** Input from **Format for GPT-4** → Output to **Parse Analysis**
- **Version-specific notes:**
  - Requires an n8n HTTP Request node version that supports the shown auth method and JSON body expressions (here `typeVersion: 4.3`).
  - OpenAI “structured outputs” behavior depends on API capabilities; schema strictness may cause refusals if the model can’t comply (rare but possible).
- **Edge cases / failures:**
  - OpenAI auth errors (missing/invalid API key in credential).
  - Rate limits or timeouts.
  - Model returning unexpected structure; although schema is requested, code downstream assumes `choices[0].message.content` is JSON text.

#### Node: Parse Analysis
- **Type / role:** `Code` — parses GPT JSON and merges with prior context.
- **Key expressions / variables:**
  - Reads OpenAI response: `response.choices[0].message.content`
  - Parses JSON: `JSON.parse(...)`
  - Pulls prior values from another node: `$('Format for GPT-4').first().json`
  - Adds `moodWithEmoji` mapping (Great/Good/Neutral/Low/Productive).
- **Outputs:** A consolidated object with:
  - `title, summary, mood, moodWithEmoji, tags, imagePrompt, highlights`
  - `date, dateFormatted, totalMessages, participants`
- **Connections:** Output → **Generate Image (DALL-E)**
- **Edge cases / failures:**
  - `JSON.parse` failure if content is not valid JSON (even with schema, still possible on upstream errors).
  - Mood not in mapping: defaults to 😐 Neutral emoji fallback.

---

### 1.5 Image Generation & Hosting
**Overview:** Generates an image using DALL·E 3 and uploads it to Cloudinary to obtain a public `secure_url`.

**Nodes involved:**
- Generate Image (DALL-E)
- Upload to Cloudinary

#### Node: Generate Image (DALL-E)
- **Type / role:** `Code` — direct API call using `fetch` to OpenAI Images API.
- **Configuration choices:**
  - **Hardcoded API key constant**: `OPENAI_API_KEY = 'YOUR_OPENAI_API_KEY'` (must be replaced).
  - Enhances prompt:
    - `${data.imagePrompt}. Artistic digital illustration, dreamy atmosphere, soft lighting, no text or words...`
  - Calls: `POST https://api.openai.com/v1/images/generations`
    - model: `dall-e-3`
    - size: `1024x1024`
    - quality: `standard`
    - response_format: `b64_json` (returns base64 image)
- **Outputs:**
  - On success: `imageBase64`, `imageError: null`
  - On failure: `imageBase64: null`, `imageError: <message>`
- **Connections:** Output → **Upload to Cloudinary**
- **Version-specific notes:**
  - Requires n8n Code node runtime that supports `fetch` (modern n8n versions do).
- **Edge cases / failures:**
  - If API key is not replaced, will always fail (401).
  - OpenAI Images endpoint errors (policy, rate limits, prompt too long).
  - Large payload handling; base64 increases item size.

#### Node: Upload to Cloudinary
- **Type / role:** `Code` — uploads base64 to Cloudinary with signed request.
- **Configuration choices:**
  - **Hardcoded credentials**:
    - `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`
  - If no `imageBase64`, returns `imageUrl: null` with `uploadError`.
  - Creates signature via SHA1:
    - signature string: `folder=...&public_id=...&timestamp=...<API_SECRET>`
  - Upload endpoint:  
    `POST https://api.cloudinary.com/v1_1/<cloud>/image/upload`
  - Uses folder `visual-journal`, public_id `diary_<YYYY-MM-DD>`.
- **Outputs:**
  - `imageUrl` (Cloudinary `secure_url`) and `imagePublicId` on success.
  - `uploadError` on failure.
- **Connections:** Output → **Create Notion Entry**
- **Edge cases / failures:**
  - Hardcoded secrets are risky; prefer n8n credentials or environment variables.
  - Signature mismatch if parameters change.
  - Overwrite behavior: same `public_id` daily will overwrite if Cloudinary allows; could be intended or not.

---

### 1.6 Notion Entry Creation & Final Response
**Overview:** Creates a Notion database page with structured properties and content blocks, then emits a success payload.

**Nodes involved:**
- Create Notion Entry
- Success Response

#### Node: Create Notion Entry
- **Type / role:** `Notion` — creates a page in a Notion database (`resource: databasePage`).
- **Configuration choices:**
  - `databaseId` is blank in JSON export (must be selected).
  - **Properties set:**
    - Title (title): `{{ $json.title }}`
    - Date (date): `{{ $json.date }}`
    - Summary (rich text): `{{ $json.summary }}`
    - Mood (select): `{{ $json.moodWithEmoji }}`
    - Message Count (number): `{{ $json.totalMessages }}`
  - **Page content blocks** (in order):
    - `image` block (configured but **no URL binding is shown**)
    - Heading “📝 Summary”
    - Summary paragraph
    - Heading “✨ Highlights”
    - 3 bulleted list items from `highlights[0..2]`
    - Divider
    - Participants line
    - Total messages line
- **Connections:** Input from **Upload to Cloudinary** → Output to **Success Response**
- **Important integration note (likely issue):**
  - The image block is present but the workflow does not map `imageUrl` into the image block configuration. As-is, the created page may contain an empty image block or fail validation depending on Notion node behavior/version. You typically must set the image block source URL to `{{ $json.imageUrl }}`.
- **Edge cases / failures:**
  - Notion auth or database permissions (integration must be shared with the database).
  - “Mood” select must match an existing select option in the database, otherwise Notion may reject or create new option depending on API behavior and n8n node capabilities.
  - If `highlights` has fewer than 3 entries, empty strings are used (safe).

#### Node: Success Response
- **Type / role:** `Code` — constructs final output.
- **Key expressions / variables:**
  - Uses Notion node output (`data.id`, `data.url`)
  - Pulls prior data explicitly from Cloudinary node: `$('Upload to Cloudinary').first().json`
- **Output:**
  - `success: true`
  - Entry summary object: title/date/mood/messageCount/hasImage/notionPageId/notionUrl
- **Edge cases / failures:**
  - If Notion node fails, this node won’t run.
  - If Cloudinary node didn’t produce `imageUrl`, `hasImage` becomes false (handled).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Main | Sticky Note | Documentation / workflow description |  |  | ## Visual Daily Journal … (full setup + steps content) |
| Sticky Note1 | Sticky Note | Visual label: schedule step |  |  | **Step 1: Schedule** Daily trigger |
| Sticky Note2 | Sticky Note | Visual label: Discord step |  |  | **Step 2: Discord** Official node |
| Sticky Note3 | Sticky Note | Visual label: filtering step |  |  | **Step 3: Filter** Today's messages |
| Sticky Note4 | Sticky Note | Visual label: guard clause step |  |  | **Step 4: Check** Has messages? |
| Sticky Note5 | Sticky Note | Visual label: formatting step |  |  | **Step 5: Format** Prepare text |
| Sticky Note6 | Sticky Note | Visual label: GPT step |  |  | **Step 6: GPT-4** Analyze day |
| Sticky Note7 | Sticky Note | Visual label: parsing step |  |  | **Step 7: Parse** Extract data |
| Sticky Note8 | Sticky Note | Visual label: image generation step |  |  | **Step 8: DALL-E** Generate image |
| Sticky Note9 | Sticky Note | Visual label: upload step |  |  | **Step 9: Upload** Cloudinary |
| Sticky Note10 | Sticky Note | Visual label: Notion step |  |  | **Step 10: Notion** Create entry |
| Daily Trigger (11pm) | Schedule Trigger | Daily workflow trigger |  | Get Discord Messages | **Step 1: Schedule** Daily trigger |
| Get Discord Messages | Discord | Fetch channel messages | Daily Trigger (11pm) | Filter Today's Messages | **Step 2: Discord** Official node |
| Filter Today's Messages | Code | Keep only today’s messages + normalize | Get Discord Messages | Has Messages? | **Step 3: Filter** Today's messages |
| Has Messages? | IF | Branch if messageCount > 0 | Filter Today's Messages | Format for GPT-4; No Messages Today | **Step 4: Check** Has messages? |
| Format for GPT-4 | Code | Build transcript + participants/stats | Has Messages? (true) | GPT-4 Analyze Day | **Step 5: Format** Prepare text |
| GPT-4 Analyze Day | HTTP Request | OpenAI Chat Completions with JSON schema | Format for GPT-4 | Parse Analysis | **Step 6: GPT-4** Analyze day |
| Parse Analysis | Code | Parse JSON, add mood emoji, merge fields | GPT-4 Analyze Day | Generate Image (DALL-E) | **Step 7: Parse** Extract data |
| Generate Image (DALL-E) | Code | Call OpenAI Images (DALL·E 3) to get base64 | Parse Analysis | Upload to Cloudinary | **Step 8: DALL-E** Generate image |
| Upload to Cloudinary | Code | Upload image to Cloudinary, get public URL | Generate Image (DALL-E) | Create Notion Entry | **Step 9: Upload** Cloudinary |
| Create Notion Entry | Notion | Create Notion database page with blocks/properties | Upload to Cloudinary | Success Response | **Step 10: Notion** Create entry |
| Success Response | Code | Final output payload with Notion link | Create Notion Entry |  |  |
| No Messages Today | Code | Final output payload when no messages | Has Messages? (false) |  |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: *Create a daily visual journal from Discord chats with GPT-4, DALL-E and Notion*.

2) **Add Schedule Trigger**
- Node: **Schedule Trigger**
- Configure to run daily:
  - Prefer “Every Day at” (e.g., 23:00) rather than “every 24 hours” if you want a stable daily time.
- Connect to the next node.

3) **Add Discord node (Get messages)**
- Node: **Discord**
- Resource: **Message**
- Operation: **Get All**
- Select:
  - **Guild (Server)**
  - **Channel**
- Credentials:
  - Create Discord application → Bot token
  - Add bot to server with **Read Message History**
  - In n8n create **Discord Bot API** credential
- Connect from Schedule Trigger → Discord.

4) **Add Code node: Filter Today’s Messages**
- Node: **Code**
- Paste logic to:
  - `$input.all()` to gather messages
  - Filter by timestamp to today’s range
  - Output a single item `{ messages, messageCount, date, fetchedAt }`
- Connect Discord → Filter code.

5) **Add IF node: Has Messages?**
- Node: **IF**
- Condition: Number → `{{$json.messageCount}}` **greater than** `0`
- True output → next processing; False output → “No Messages Today”
- Connect Filter → IF.

6) **Add Code node (False branch): No Messages Today**
- Node: **Code**
- Return `{ success:false, message:'...', date:'...' }`
- Connect IF (false) → this node.

7) **Add Code node (True branch): Format for GPT-4**
- Node: **Code**
- Create:
  - `conversationText` transcript lines
  - `participants` unique list
  - `messageCounts` dictionary
  - `totalMessages`, `date`, `dateFormatted`
- Connect IF (true) → Format code.

8) **Add HTTP Request node: GPT-4 Analyze Day**
- Node: **HTTP Request**
- Method: **POST**
- URL: `https://api.openai.com/v1/chat/completions`
- Authentication: **Predefined credentials** → **OpenAI API**
- Body: JSON containing:
  - model: `gpt-4o`
  - system/user messages using expressions for `conversationText`, `participants`, `totalMessages`
  - `response_format: json_schema` with required fields (`title, summary, mood, tags, imagePrompt, highlights`)
- Connect Format → HTTP Request.

9) **Add Code node: Parse Analysis**
- Node: **Code**
- Parse `choices[0].message.content` as JSON
- Add `moodWithEmoji`
- Merge in `date`, `participants`, `totalMessages` from the “Format for GPT-4” node using:  
  `$('Format for GPT-4').first().json`
- Connect GPT node → Parse.

10) **Add Code node: Generate Image (DALL·E)**
- Node: **Code**
- Call OpenAI Images API:
  - Endpoint: `https://api.openai.com/v1/images/generations`
  - model: `dall-e-3`
  - `response_format: b64_json`
- **Important:** Do not hardcode keys in code in production. Prefer:
  - n8n credentials, or
  - environment variable access.
- Connect Parse → DALL·E code.

11) **Add Code node: Upload to Cloudinary**
- Node: **Code**
- Implement signed Cloudinary upload:
  - Upload base64 as `data:image/png;base64,...`
  - Return `imageUrl` from `secure_url`
- Configure Cloudinary credentials securely (ideally via env vars).
- Connect DALL·E code → Cloudinary code.

12) **Add Notion node: Create Notion Entry**
- Node: **Notion**
- Resource: **Database Page**
- Choose your **Database ID**
- Map properties:
  - Title = `{{$json.title}}`
  - Date = `{{$json.date}}`
  - Summary = `{{$json.summary}}`
  - Mood = `{{$json.moodWithEmoji}}`
  - Message Count = `{{$json.totalMessages}}`
- Add page blocks:
  - Image block: set source URL to `{{$json.imageUrl}}` (this mapping is required for the image to appear)
  - Headings + summary text
  - Highlights list items using `highlights[i]`
  - Participants + total messages lines
- Credentials:
  - Create Notion integration
  - Share the database with the integration
  - Use Notion API credential in n8n
- Connect Cloudinary → Notion.

13) **Add Code node: Success Response**
- Node: **Code**
- Build final payload with:
  - Notion page ID and URL from Notion output
  - Fields from Cloudinary output (title/date/mood/message count/imageUrl)
- Connect Notion → Success Response.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Discord bot setup: create application → bot token → add bot to server with Read Message History permission | https://discord.com/developers/applications |
| Notion integration must be shared with the target database | https://www.notion.so/my-integrations |
| Cloudinary required because Notion needs a publicly accessible image URL (base64 is not directly usable) | https://cloudinary.com |
| Current workflow uses hardcoded secrets for DALL·E and Cloudinary in Code nodes; prefer n8n credentials or environment variables | Security / maintainability note |
| The Notion “image” block is present but not visibly mapped to `imageUrl` in the exported configuration; ensure the image block uses `{{$json.imageUrl}}` | Potential functional issue to fix |