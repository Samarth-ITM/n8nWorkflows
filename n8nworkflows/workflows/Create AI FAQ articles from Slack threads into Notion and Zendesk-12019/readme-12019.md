Create AI FAQ articles from Slack threads into Notion and Zendesk

https://n8nworkflows.xyz/workflows/create-ai-faq-articles-from-slack-threads-into-notion-and-zendesk-12019


# Create AI FAQ articles from Slack threads into Notion and Zendesk

## 1. Workflow Overview

**Purpose:**  
Automatically convert useful Slack thread conversations into an AI-generated FAQ article, store it in **Notion** (required) and optionally **Zendesk**, then post a confirmation back into the originating Slack thread.

**Primary use cases:**  
- Turning internal support Q&A into a searchable knowledge base  
- Capturing onboarding/technical discussions as structured documentation  
- Reducing manual copy/paste from Slack into Notion/Zendesk

### 1.1 Trigger & Validation
Detects a **📚 (book)** reaction on a Slack message and ensures the reaction is applied to a **message** item type.

### 1.2 Conversation Retrieval
Fetches the **parent message** and then retrieves **all messages in the thread** to reconstruct full context.

### 1.3 AI Processing & Structuring
Formats thread messages into a Q/A transcript, sends it to OpenAI to generate a structured FAQ output, then prepares a normalized data object for Notion (including a Slack thread URL).

### 1.4 Storage & Notification
Creates a Notion database page (the canonical record), optionally creates a Zendesk Help Center article, and posts Slack confirmation containing the created links.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Validation
**Overview:**  
Starts the workflow on Slack reaction events, then filters to proceed only when the emoji is `book` and the reacted item is a Slack message.

**Nodes involved:**  
- Slack Trigger - :book: Reaction  
- Workflow Configuration1  
- IF - :book: Reaction Check

#### Node: **Slack Trigger - :book: Reaction**
- **Type / role:** `slackTrigger` — entry point listening to Slack events.
- **Configuration (interpreted):**
  - Triggers on Slack events (intended: reaction added).
  - `channelId` is present but empty in the JSON (meaning: not restricted or not configured yet depending on n8n UI behavior).
- **Key data produced:** typically includes fields like `reaction`, `item.type`, message/channel references.
- **Connections:**
  - Output → **Workflow Configuration1**
- **Potential failures / edge cases:**
  - Slack app not subscribed to reaction events, missing scopes, bot not in channel.
  - If `channelId` must be set (workspace policy / node behavior), trigger may not fire.
  - Event payload differences depending on Slack event subtype.

#### Node: **Workflow Configuration1**
- **Type / role:** `set` — injects required configuration constants (IDs).
- **Configuration (interpreted):**
  - Sets:
    - `slackWorkspaceId` (placeholder)
    - `notionDatabaseId` (placeholder)
    - `zendeskSectionId` (placeholder, optional)
- **Key variables used later:**
  - `$('Workflow Configuration1').item.json.slackWorkspaceId`
  - `$('Workflow Configuration1').item.json.notionDatabaseId`
  - `$('Workflow Configuration1').item.json.zendeskSectionId` (but note: not actually referenced by the Zendesk node in this workflow as provided)
- **Connections:**
  - Input ← **Slack Trigger - :book: Reaction**
  - Output → **IF - :book: Reaction Check**
- **Potential failures / edge cases:**
  - Placeholders not replaced → broken Slack thread URLs and Notion create failures.
  - This node assumes a single-item flow; if the trigger produces multiple items, “item” referencing could become ambiguous.

#### Node: **IF - :book: Reaction Check**
- **Type / role:** `if` — gatekeeper.
- **Configuration (interpreted):**
  - Continues only if:
    - `{{$json.reaction}} == "book"`
    - `{{$json.item.type}} == "message"`
- **Connections:**
  - Input ← **Workflow Configuration1**
  - True output → **Slack - Get Parent Message**
  - False output → not connected (workflow ends silently)
- **Potential failures / edge cases:**
  - If Slack sends emoji name differently (e.g., `:book:` vs `book`) depending on payload, condition may fail.
  - If reaction is on a file/comment/etc., it will correctly stop.

---

### Block 2 — Conversation Retrieval
**Overview:**  
Retrieves the base message and the full thread to build a complete conversation history for AI processing.

**Nodes involved:**  
- Slack - Get Parent Message  
- Slack - Get Thread

#### Node: **Slack - Get Parent Message**
- **Type / role:** `slack` — reads a single message.
- **Configuration (interpreted):**
  - Operation: **get**
  - The JSON does not show mapped parameters (channel, timestamp); in n8n these are usually required and should be mapped from the trigger event (e.g., channel + message ts).
- **Connections:**
  - Input ← **IF - :book: Reaction Check** (true path)
  - Output → **Slack - Get Thread**
- **Potential failures / edge cases:**
  - Missing required fields mapping (channel/ts) will cause runtime errors.
  - Permissions: `channels:history` / `groups:history` / `im:history` depending on channel type.
  - If message was deleted or inaccessible, Slack API will return errors.

#### Node: **Slack - Get Thread**
- **Type / role:** `slack` — retrieves multiple messages (thread).
- **Configuration (interpreted):**
  - Operation: **getAll**
  - Intended to fetch all replies in the thread (and possibly include parent depending on Slack API method used by the node).
- **Connections:**
  - Input ← **Slack - Get Parent Message**
  - Output → **Code - Format Conversation**
- **Potential failures / edge cases:**
  - Pagination/limits for long threads.
  - Thread identifier mapping (thread_ts) must be correct.
  - Private channels require the bot to be a member.

---

### Block 3 — AI Processing & Structuring
**Overview:**  
Formats the Slack thread into a Q/A transcript, asks OpenAI to generate an FAQ payload, and prepares a Notion-ready object including Slack thread URL.

**Nodes involved:**  
- Code - Format Conversation  
- OpenAI - Generate FAQ  
- Code - Prepare Notion Data

#### Node: **Code - Format Conversation**
- **Type / role:** `code` — transforms Slack messages into an AI-friendly prompt + metadata.
- **Configuration (interpreted):**
  - Mode: **Run once for each item**
  - Code behavior:
    - `messages = $input.all()` collects all incoming items (the thread messages).
    - Treats `messages[0]` as the **question** and formats it as: `Q: <text>`
    - Formats subsequent messages as: `A1: <user> - <text>`, `A2: ...`
    - Builds `metadata`:
      - `question_author`, `channel_id`, `thread_ts`, `created_at`, `channel_name`
      - `slack_workspace_id` from **Workflow Configuration1**
    - Returns:
      - `conversation_for_ai` plus metadata fields
- **Key expressions/variables:**
  - `$('Workflow Configuration1').item.json` for config
  - Uses `$json.channel` and `$json.channel_name` from the current item context (may not exist depending on Slack node output)
- **Connections:**
  - Input ← **Slack - Get Thread**
  - Output → **OpenAI - Generate FAQ**
- **Potential failures / edge cases:**
  - If Slack “Get Thread” does not include the parent message as the first element, the “question” assumption may be wrong.
  - `channel_name` is defaulted to `'unknown'` because it may not exist.
  - `created_at` uses `firstMsg.ts`; if missing/non-numeric → invalid date.
  - Running “once per item” while also using `$input.all()` is unusual; it can cause repeated identical outputs if multiple input items trigger this node. Typically this should be “Run once for all items”.

#### Node: **OpenAI - Generate FAQ**
- **Type / role:** `openAi` — generates structured FAQ content from the transcript.
- **Configuration (interpreted):**
  - Operation: **message**
  - The prompt/system instructions are not visible in the JSON; however downstream code expects the model to return **JSON in `content`** (string).
- **Expected output contract (implied by downstream parsing):**
  - `content` should be a JSON string, e.g.:
    - `faq_question` (string)
    - `faq_answer` (string; markdown allowed)
    - `tags` (array of strings)
    - `summary` (string)
- **Connections:**
  - Input ← **Code - Format Conversation**
  - Output → **Code - Prepare Notion Data**
- **Potential failures / edge cases:**
  - If the model returns non-JSON text → `JSON.parse` will fail later.
  - Rate limits, auth errors, model availability.
  - Large threads could exceed token limits unless trimmed/summarized.

#### Node: **Code - Prepare Notion Data**
- **Type / role:** `code` — merges AI output and metadata and creates final fields for Notion.
- **Configuration (interpreted):**
  - Parses AI response: `JSON.parse($json.content || '{}')`
  - Reads previous metadata: `const prevData = $input.item.json;`
    - **Important:** At this point `$input.item.json` is the OpenAI node’s item, not the “Format Conversation” item. Unless n8n is preserving paired input in a special way, `prevData.channel_id` etc. may not exist here. This is a common logical bug.
  - Builds Slack thread URL:
    - Uses `workspaceId` from config
    - Uses `channelId` from `prevData.channel_id`
    - Uses `thread_ts` cleaned by removing `.`
    - URL format:
      `https://app.slack.com/client/${workspaceId}/${channelId}/thread/${channelId}-${threadTsClean}`
  - Outputs:
    - `title`, `answer_markdown`, `tags`, `summary`
    - `slack_thread_url`, `channel_name`, `created_at`
    - `notion_database_id` from config
- **Connections:**
  - Input ← **OpenAI - Generate FAQ**
  - Output → **Notion - Create FAQ Page**
- **Potential failures / edge cases:**
  - **JSON parsing failure** if OpenAI output isn’t strict JSON.
  - Slack URL invalid if workspace/channel/thread ids are missing.
  - Metadata loss due to `prevData` referencing the wrong node’s data (see note above).

---

### Block 4 — Storage & Notification
**Overview:**  
Creates a Notion page with the generated FAQ, optionally creates a Zendesk article, then posts a Slack message with result links.

**Nodes involved:**  
- Notion - Create FAQ Page  
- Zendesk - Create Article (Optional)  
- Slack - Notify Completion

#### Node: **Notion - Create FAQ Page**
- **Type / role:** `notion` — creates a page in a Notion database.
- **Configuration (interpreted):**
  - Resource: **databasePage**
  - Database ID: `{{$json.notion_database_id}}`
  - Properties set:
    - `Name` (title) = `{{$json.title}}`
    - `Summary` (rich text) = `{{$json.summary}}`
    - `Tags` (present but no value mapped in this workflow JSON)
    - `Source` (present but no value mapped; presumably should store `slack_thread_url`)
    - `Channel` (present but no value mapped; presumably should store `channel_name`)
  - Page content/body:
    - Adds a block with rich text = `{{$json.answer_markdown}}`
- **Connections:**
  - Input ← **Code - Prepare Notion Data**
  - Output → **Zendesk - Create Article (Optional)**
- **Potential failures / edge cases:**
  - Notion auth/scopes, database access.
  - Database property names must match exactly: `Name`, `Summary`, `Tags`, `Source`, `Channel`.
  - Tags not actually written because no mapping is configured.
  - Rich text length limits and markdown rendering differences.

#### Node: **Zendesk - Create Article (Optional)**
- **Type / role:** `zendesk` — intended to create a Help Center article.
- **Configuration (interpreted):**
  - Resource: **article**
  - Operation/required fields are not shown; as-is, this node likely requires additional configuration (section ID, title, body, locale, etc.).
- **Connections:**
  - Input ← **Notion - Create FAQ Page**
  - Output → **Slack - Notify Completion**
- **Potential failures / edge cases:**
  - If left unconfigured, execution will fail (not “optional” in routing terms—there is no IF/branch to skip).
  - Missing section ID mapping (a `zendeskSectionId` exists in config but is not used here).
  - Zendesk permissions, Help Center enabled, locale required.

#### Node: **Slack - Notify Completion**
- **Type / role:** `slack` — posts a message back to Slack.
- **Configuration (interpreted):**
  - Posts text:
    - Notion link from: `$('Notion - Create FAQ Page').item.json.url`
    - Zendesk link from: `$('Zendesk - Create Article (Optional)').item.json.html_url || 'Skipped'`
- **Connections:**
  - Input ← **Zendesk - Create Article (Optional)**
  - Output → none (end)
- **Potential failures / edge cases:**
  - Node does not show which channel/thread it posts to; it likely must be configured to reply in the thread using `channel` + `thread_ts`.
  - If Zendesk node fails earlier, this notification never runs.
  - If Zendesk node runs but doesn’t return `html_url`, the expression falls back to “Skipped”.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Slack Trigger - :book: Reaction | slackTrigger | Entry point: capture Slack reaction events | — | Workflow Configuration1 | Trigger & Validation: Detects 📚 reaction on messages and validates it's a proper message (not a file or other item type). |
| Workflow Configuration1 | set | Stores workspace/database/section IDs used across nodes | Slack Trigger - :book: Reaction | IF - :book: Reaction Check | Trigger & Validation: Detects 📚 reaction on messages and validates it's a proper message (not a file or other item type). |
| IF - :book: Reaction Check | if | Filter: proceed only for `reaction=book` and `item.type=message` | Workflow Configuration1 | Slack - Get Parent Message | Trigger & Validation: Detects 📚 reaction on messages and validates it's a proper message (not a file or other item type). |
| Slack - Get Parent Message | slack | Fetch the reacted parent message | IF - :book: Reaction Check | Slack - Get Thread | Conversation Retrieval: Fetches the parent message and all thread replies to capture the full Q&A context. |
| Slack - Get Thread | slack | Fetch all messages in the thread | Slack - Get Parent Message | Code - Format Conversation | Conversation Retrieval: Fetches the parent message and all thread replies to capture the full Q&A context. |
| Code - Format Conversation | code | Build Q/A transcript + metadata for AI | Slack - Get Thread | OpenAI - Generate FAQ | AI Processing: Formats the conversation into Q&A structure, sends to OpenAI for FAQ generation, then prepares structured data for storage. |
| OpenAI - Generate FAQ | openAi | Generate structured FAQ JSON from transcript | Code - Format Conversation | Code - Prepare Notion Data | AI Processing: Formats the conversation into Q&A structure, sends to OpenAI for FAQ generation, then prepares structured data for storage. |
| Code - Prepare Notion Data | code | Merge AI output + build Slack URL + Notion fields | OpenAI - Generate FAQ | Notion - Create FAQ Page | AI Processing: Formats the conversation into Q&A structure, sends to OpenAI for FAQ generation, then prepares structured data for storage. |
| Notion - Create FAQ Page | notion | Persist FAQ into Notion database | Code - Prepare Notion Data | Zendesk - Create Article (Optional) | Storage & Notification: Saves the FAQ to Notion (required) and Zendesk (optional), then notifies the Slack thread with links to the created articles. |
| Zendesk - Create Article (Optional) | zendesk | (Intended) Create Zendesk Help Center article | Notion - Create FAQ Page | Slack - Notify Completion | Storage & Notification: Saves the FAQ to Notion (required) and Zendesk (optional), then notifies the Slack thread with links to the created articles. |
| Slack - Notify Completion | slack | Notify Slack with Notion/Zendesk links | Zendesk - Create Article (Optional) | — | Storage & Notification: Saves the FAQ to Notion (required) and Zendesk (optional), then notifies the Slack thread with links to the created articles. |
| Workflow Overview | stickyNote | Documentation / explanation | — | — | ## How it works\nWhen someone reacts to a Slack thread with 📚, this workflow captures the entire conversation, uses AI to transform it into a well-formatted FAQ article, then saves it to your Notion knowledge base and optionally to Zendesk. A confirmation message is posted back to the original Slack thread.\n\nPerfect for building internal documentation from support conversations, technical discussions, or onboarding questions.\n\n## Setup steps\n1. Configure Slack app with reaction permissions and add the bot to your channels\n2. Create a Notion database with properties: Name (title), Summary (text), Tags (multi-select), Source (URL), Channel (text)\n3. Get your Slack Workspace ID from the URL when logged into Slack web\n4. Add your API credentials: Slack, OpenAI, Notion, and optionally Zendesk\n5. Update the Workflow Configuration node with your workspace and database IDs\n6. Test by reacting to a message with the 📚 emoji |
| Group 1 | stickyNote | Visual grouping label | — | — | ## Trigger & Validation\nDetects 📚 reaction on messages and validates it's a proper message (not a file or other item type). |
| Group 2 | stickyNote | Visual grouping label | — | — | ## Conversation Retrieval\nFetches the parent message and all thread replies to capture the full Q&A context. |
| Group 3 | stickyNote | Visual grouping label | — | — | ## AI Processing\nFormats the conversation into Q&A structure, sends to OpenAI for FAQ generation, then prepares structured data for storage. |
| Group 4 | stickyNote | Visual grouping label | — | — | ## Storage & Notification\nSaves the FAQ to Notion (required) and Zendesk (optional), then notifies the Slack thread with links to the created articles. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create node: Slack Trigger**
   - Node: **Slack Trigger**
   - Event: Reaction added (or equivalent reaction event)
   - (Optional) Restrict to a channel if desired.
   - Credentials: Slack OAuth (bot token)
   - Ensure Slack app scopes/events include reaction events and the bot is in target channels.

2. **Create node: Set (Workflow Configuration1)**
   - Node: **Set**
   - Add fields:
     - `slackWorkspaceId` = your Slack workspace ID (from Slack web URL)
     - `notionDatabaseId` = Notion database ID
     - `zendeskSectionId` = Zendesk section ID (optional)
   - Connect: **Slack Trigger → Set**

3. **Create node: IF (reaction check)**
   - Node: **IF**
   - Conditions (AND):
     - `{{$json.reaction}}` equals `book`
     - `{{$json.item.type}}` equals `message`
   - Connect: **Set → IF (true path continues)**

4. **Create node: Slack (Get Parent Message)**
   - Node: **Slack**
   - Operation: **Get** message
   - Map required identifiers from trigger payload (commonly):
     - Channel: from trigger event (e.g., `$json.item.channel` or similar)
     - Timestamp/message ID: from trigger event (e.g., `$json.item.ts`)
   - Connect: **IF (true) → Slack Get Parent**

5. **Create node: Slack (Get Thread)**
   - Node: **Slack**
   - Operation: **Get All** thread messages
   - Map:
     - Channel: from parent message output
     - Thread timestamp: parent message `thread_ts` or `ts`
   - Connect: **Slack Get Parent → Slack Get Thread**

6. **Create node: Code (Format Conversation)**
   - Node: **Code**
   - Prefer setting mode to **Run once for all items** (recommended) if the input is multiple messages.
   - Implement logic to:
     - Build `conversation_for_ai` with `Q:` and `A1/A2...`
     - Output metadata: `channel_id`, `thread_ts`, `created_at`, `channel_name`, `slack_workspace_id`
   - Connect: **Slack Get Thread → Code Format**

7. **Create node: OpenAI (Generate FAQ)**
   - Node: **OpenAI**
   - Operation: **Message**
   - Configure prompt so the assistant returns **strict JSON** with keys:
     - `faq_question`, `faq_answer`, `tags` (array), `summary`
   - Credentials: OpenAI API key
   - Connect: **Code Format → OpenAI**

8. **Create node: Code (Prepare Notion Data)**
   - Node: **Code**
   - Parse OpenAI result JSON safely (ideally with try/catch).
   - Merge with metadata from the earlier step (best practice: pass metadata forward explicitly, or use a Merge node).
   - Build `slack_thread_url` using:
     - workspace ID + channel ID + cleaned thread ts
   - Output fields:
     - `title`, `answer_markdown`, `summary`, `tags`, `slack_thread_url`, `channel_name`, `created_at`, `notion_database_id`
   - Connect: **OpenAI → Code Prepare**

9. **Create node: Notion (Create FAQ Page)**
   - Node: **Notion**
   - Resource: **Database Page**
   - Database ID: from `{{$json.notion_database_id}}`
   - Map properties (match your Notion database schema):
     - Name (Title) ← `{{$json.title}}`
     - Summary (Rich text) ← `{{$json.summary}}`
     - Tags (Multi-select) ← map `{{$json.tags}}` properly
     - Source (URL) ← `{{$json.slack_thread_url}}`
     - Channel (Text) ← `{{$json.channel_name}}`
   - Add page content block containing `answer_markdown`.
   - Credentials: Notion integration token with access to the database.
   - Connect: **Code Prepare → Notion**

10. **(Optional) Create node: Zendesk (Create Article)**
   - Node: **Zendesk**
   - Resource: **Article**
   - Configure required fields (typical):
     - Section ID ← `{{$('Workflow Configuration1').item.json.zendeskSectionId}}`
     - Title ← `{{$json.title}}`
     - Body ← `{{$json.answer_markdown}}` (or HTML-converted)
     - Locale, Draft/Published, etc.
   - Credentials: Zendesk API token/OAuth depending on setup.
   - If truly optional, add an **IF** node to skip Zendesk when `zendeskSectionId` is empty, and route around it.
   - Connect: **Notion → Zendesk** (or **Notion → IF → Zendesk**)

11. **Create node: Slack (Notify Completion)**
   - Node: **Slack**
   - Operation: Post message (ideally as a thread reply)
   - Map:
     - Channel: original channel id
     - Thread ts: original thread timestamp
   - Message text should include:
     - Notion page URL from Notion output
     - Zendesk article URL if created
   - Connect: **Zendesk (or Notion/IF branch) → Slack Notify**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Notion database with properties: Name (title), Summary (text), Tags (multi-select), Source (URL), Channel (text). | Mentioned in “Workflow Overview” sticky note. |
| Get your Slack Workspace ID from the URL when logged into Slack web. | Required for building the Slack thread URL. |
| Ensure Slack app has reaction permissions and the bot is added to channels. | Required for Slack Trigger and message/thread retrieval. |
| OpenAI must return strict JSON in `content` for downstream parsing. | Required by “Code - Prepare Notion Data” (`JSON.parse`). |
| Zendesk step is labeled optional but is not conditional in the current wiring. | Add an IF/branch if Zendesk should be skippable without failing the run. |