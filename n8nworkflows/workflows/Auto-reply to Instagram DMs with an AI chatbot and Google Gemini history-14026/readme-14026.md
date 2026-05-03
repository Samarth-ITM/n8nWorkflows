Auto-reply to Instagram DMs with an AI chatbot and Google Gemini history

https://n8nworkflows.xyz/workflows/auto-reply-to-instagram-dms-with-an-ai-chatbot-and-google-gemini-history-14026


# Auto-reply to Instagram DMs with an AI chatbot and Google Gemini history

# 1. Workflow Overview

This workflow automates replies to Instagram Direct Messages using an AI agent powered by Google Gemini, while keeping lightweight conversation history in an n8n Data Table. It receives Instagram webhook events, validates Meta’s webhook challenge, ignores non-text and self-sent messages, stores incoming user messages, builds a history-aware prompt, generates an AI response, splits long replies into Instagram-safe chunks, sends them sequentially, and updates stored conversation state.

## 1.1 Webhook Reception and Event Filtering
This block handles incoming Instagram webhook traffic. It supports Meta webhook verification, filters for actual text messages, and ignores echo events sent by the Instagram page itself.

## 1.2 Context Extraction and Message Persistence
This block extracts the key runtime fields from the webhook payload, identifies the external user, marks the conversation as seen, and stores the incoming message as an unprocessed row in the Data Table.

## 1.3 Simple Batching and History Retrieval
This block merges all currently unprocessed user messages into one combined prompt, fetches processed history for that user and page, keeps only the 15 newest rows, and formats them into “old session” and “current session” history strings.

## 1.4 AI Response Generation
This block sends a typing indicator, invokes the Gemini-backed AI agent with merged input plus history, and prepares the response in Facebook/Instagram-compatible formatting.

## 1.5 Chunk Delivery and State Updates
This block splits long AI output into ≤2000-character chunks, sends each chunk in sequence with a delay, then marks pending messages as processed, saves the AI reply in the table, and deletes older history rows outside the retention window.

---

# 2. Block-by-Block Analysis

## 2.1 Webhook Reception and Event Filtering

### Overview
This block receives both Meta verification requests and live Instagram messaging events. It ensures that only incoming user text messages proceed into the chatbot logic.

### Nodes Involved
- Instagram Webhook
- Webhook Verification
- Is Message?
- Set Context
- Is from page?
- User is Sender

### Node Details

#### Instagram Webhook
- **Type and role:** `n8n-nodes-base.webhook`; entry point for Instagram webhook events.
- **Configuration choices:**
  - Webhook path: `nguyenthieutoan-instagram`
  - Response mode: uses a separate response node
  - Multiple HTTP methods enabled, allowing Meta verification and event delivery
- **Key expressions/variables used:** none internally; payload is consumed downstream as `$json.body` and `$json.query`.
- **Input and output connections:**
  - Entry point node, no inputs
  - Output 1 goes to `Webhook Verification`
  - Output 2 goes to both `Webhook Verification` and `Is Message?`
- **Version-specific requirements:** typeVersion 2.1; compatible with response-node mode in modern n8n versions.
- **Edge cases/failures:**
  - Meta verification fails if the workflow is not active on the production URL
  - Payload structure may differ for non-message events
  - If Instagram sends unsupported event shapes, downstream expressions can fail unless guarded
- **Sub-workflow reference:** none

#### Webhook Verification
- **Type and role:** `n8n-nodes-base.respondToWebhook`; returns the challenge token for Meta webhook verification.
- **Configuration choices:**
  - HTTP 200 response
  - Plain text response body from `hub.challenge`
- **Key expressions/variables used:**
  - `{{ $json.query['hub.challenge'] }}`
- **Input and output connections:**
  - Triggered by `Instagram Webhook`
  - No downstream nodes
- **Version-specific requirements:** typeVersion 1.5.
- **Edge cases/failures:**
  - If `hub.challenge` is missing, response body will be empty
  - Works only when Meta sends GET verification or request contains expected query fields
- **Sub-workflow reference:** none

#### Is Message?
- **Type and role:** `n8n-nodes-base.if`; filters events to only those containing a text message.
- **Configuration choices:**
  - Checks existence of `body.entry[0].messaging[0].message.text`
  - Strict type validation
- **Key expressions/variables used:**
  - `{{ $json.body.entry?.[0]?.messaging?.[0]?.message?.text }}`
- **Input and output connections:**
  - Input from `Instagram Webhook`
  - True branch goes to `Set Context`
  - False branch unused
- **Version-specific requirements:** typeVersion 2.3.
- **Edge cases/failures:**
  - Non-text message types are ignored
  - If Instagram sends attachment-only events, they stop here
- **Sub-workflow reference:** none

#### Set Context
- **Type and role:** `n8n-nodes-base.set`; centralizes extracted runtime variables.
- **Configuration choices:**
  - Creates:
    - `message`
    - `ig_id`
    - `is_from_bot`
    - `ig_access_token`
    - `received_timestamp`
  - `ig_access_token` is hardcoded placeholder and must be replaced manually
- **Key expressions/variables used:**
  - `{{ $json.body.entry[0].messaging[0].message.text }}`
  - `{{ $json.body.entry[0].id }}`
  - `{{ $json.body.entry[0].messaging[0].message.metadata }}`
  - `{{ $json.body.entry[0].messaging[0].timestamp }}`
- **Input and output connections:**
  - Input from `Is Message?`
  - Output to `Is from page?`
- **Version-specific requirements:** typeVersion 3.4.
- **Edge cases/failures:**
  - If webhook payload shape changes, field extraction may break
  - Placeholder token will cause Graph API auth failures until replaced
  - `is_from_bot` is extracted but not used elsewhere
- **Sub-workflow reference:** none

#### Is from page?
- **Type and role:** `n8n-nodes-base.if`; prevents the bot from responding to its own page-sent messages.
- **Configuration choices:**
  - Compares sender ID with page/account ID
  - If equal, message is from the page and is blocked
- **Key expressions/variables used:**
  - Left: `{{ $('Instagram Webhook').item.json.body.entry[0].messaging[0].sender.id }}`
  - Right: `{{ $('Instagram Webhook').item.json.body.entry[0].id }}`
- **Input and output connections:**
  - Input from `Set Context`
  - False branch goes to `User is Sender`
  - True branch unused
- **Version-specific requirements:** typeVersion 2.3.
- **Edge cases/failures:**
  - Assumes Instagram event structure where page/account id is in `entry[0].id`
  - If echo semantics differ, filtering may miss some bot-originated messages
- **Sub-workflow reference:** none

#### User is Sender
- **Type and role:** `n8n-nodes-base.set`; stores the external sender’s Instagram user ID for later API calls and table operations.
- **Configuration choices:**
  - Sets `user_id` from webhook sender ID
- **Key expressions/variables used:**
  - `{{ $('Instagram Webhook').item.json.body.entry[0].messaging[0].sender.id }}`
- **Input and output connections:**
  - Input from `Is from page?`
  - Outputs to `Insert To Unprocessed` and `Seen`
- **Version-specific requirements:** typeVersion 3.4.
- **Edge cases/failures:**
  - Depends on original webhook payload remaining available by node reference
- **Sub-workflow reference:** none

---

## 2.2 Context Extraction and Message Persistence

### Overview
This block stores the new incoming message as unprocessed and immediately marks the conversation as seen via the Instagram Graph API. It establishes the raw input queue used for simple batching.

### Nodes Involved
- Seen
- Insert To Unprocessed

### Node Details

#### Seen
- **Type and role:** `n8n-nodes-base.httpRequest`; sends `mark_seen` to Instagram.
- **Configuration choices:**
  - POST to `https://graph.instagram.com/v24.0/{ig_id}/messages`
  - JSON body with recipient ID and `sender_action: mark_seen`
  - Access token passed as query parameter
  - `onError` set to continue regular output
- **Key expressions/variables used:**
  - URL: `{{ $('Set Context').item.json.ig_id }}`
  - Recipient: `{{ $('User is Sender').item.json.user_id }}`
  - Access token: `{{ $('Set Context').item.json.ig_access_token }}`
- **Input and output connections:**
  - Input from `User is Sender`
  - No downstream connection
- **Version-specific requirements:** typeVersion 4.3.
- **Edge cases/failures:**
  - Invalid or expired Instagram token
  - Missing permissions in Meta app
  - Recipient mismatch or unsupported action
  - Continues even if the API call fails, so failures may go unnoticed unless monitored
- **Sub-workflow reference:** none

#### Insert To Unprocessed
- **Type and role:** `n8n-nodes-base.dataTable`; inserts incoming user text into persistent storage.
- **Configuration choices:**
  - Data Table: `insert_message`
  - Inserts columns:
    - `page_id`
    - `user_id`
    - `processed = false`
    - `user_text`
  - Uses explicit schema mapping
- **Key expressions/variables used:**
  - `page_id = {{ $('Set Context').item.json.ig_id }}`
  - `user_id = {{ $('User is Sender').item.json.user_id }}`
  - `user_text = {{ $('Set Context').item.json.message }}`
- **Input and output connections:**
  - Input from `User is Sender`
  - Output to `Get MaxID and Merged Mess`
- **Version-specific requirements:** Data Table node typeVersion 1.1; requires n8n with Data Tables feature enabled.
- **Edge cases/failures:**
  - Data Table missing or misconfigured schema
  - Type mismatch if `processed` column is not Boolean
  - Insert errors if permissions/project context are wrong
- **Sub-workflow reference:** none

---

## 2.3 Simple Batching and History Retrieval

### Overview
This block merges all pending unprocessed messages into one AI input, then retrieves prior processed rows to provide conversational memory. It retains only the latest 15 rows and groups them into date-based session summaries.

### Nodes Involved
- Get MaxID and Merged Mess
- Get history message
- Get 15 newest rows
- Merge History and Find Min_ID

### Node Details

#### Get MaxID and Merged Mess
- **Type and role:** `n8n-nodes-base.code`; performs simple batching of unprocessed user messages.
- **Configuration choices:**
  - Sorts input rows by numeric `id`
  - Concatenates all non-null `user_text` values into `merged_message`
  - Returns only the row with the largest `id`, augmented with `merged_message`
- **Key expressions/variables used:** uses `items` in JavaScript.
- **Input and output connections:**
  - Input from `Insert To Unprocessed`
  - Output to `Get history message`
- **Version-specific requirements:** code node typeVersion 2.
- **Edge cases/failures:**
  - If input is empty, returns empty array
  - Assumes `id` exists and is numeric
  - Joins messages with spaces only; may blur message boundaries
- **Sub-workflow reference:** none

#### Get history message
- **Type and role:** `n8n-nodes-base.dataTable`; retrieves processed historical rows for this page-user pair.
- **Configuration choices:**
  - Operation: get
  - Filters:
    - `user_id = current user`
    - `processed is true`
    - `page_id = current row page_id`
  - Returns all matching rows
- **Key expressions/variables used:**
  - `{{ $('User is Sender').item.json.user_id }}`
  - `{{ $json.page_id }}`
- **Input and output connections:**
  - Input from `Get MaxID and Merged Mess`
  - Output to `Get 15 newest rows`
- **Version-specific requirements:** Data Table node typeVersion 1.1.
- **Edge cases/failures:**
  - If no history exists, `alwaysOutputData` ensures downstream still runs
  - Large histories may affect performance before trimming
- **Sub-workflow reference:** none

#### Get 15 newest rows
- **Type and role:** `n8n-nodes-base.code`; limits history to the 15 newest records while preserving chronological order.
- **Configuration choices:**
  - Sorts by `createdAt` descending
  - Takes 15 newest
  - Re-sorts ascending for conversation order
  - `alwaysOutputData` enabled
- **Key expressions/variables used:** JavaScript over `items`.
- **Input and output connections:**
  - Input from `Get history message`
  - Output to `Merge History and Find Min_ID`
- **Version-specific requirements:** code node typeVersion 2.
- **Edge cases/failures:**
  - If `createdAt` is missing or malformed, sorting may become invalid
  - Empty input returns empty array
- **Sub-workflow reference:** none

#### Merge History and Find Min_ID
- **Type and role:** `n8n-nodes-base.code`; formats processed history into session text and computes the minimum retained row ID.
- **Configuration choices:**
  - Sorts ascending by `createdAt`
  - Splits sessions by Vietnam-local calendar date
  - Builds:
    - `old_session_history`
    - `now_session_history`
    - `min_id`
  - Includes user, page, and AI lines when available
- **Key expressions/variables used:**
  - Uses `createdAt`, `updatedAt`, `user_text`, `page_text`, `bot_rep`, `id`
  - Time zone: `Asia/Ho_Chi_Minh`
- **Input and output connections:**
  - Input from `Get 15 newest rows`
  - Outputs to `Send Typing` and `Process Merged Message`
- **Version-specific requirements:** code node typeVersion 2.
- **Edge cases/failures:**
  - If input is empty, returns empty array, which can stop downstream AI generation
  - Uses `bot_rep`, but current Data Table inserts/updates do not clearly persist a `bot_rep` column; this field may remain unused
  - Date-grouping is day-based, not inactivity-gap based
- **Sub-workflow reference:** none

---

## 2.4 AI Response Generation

### Overview
This block indicates typing presence to the user, runs the AI agent with persona and history, and reformats the output for Instagram/Facebook-compatible message delivery.

### Nodes Involved
- Send Typing
- Google Gemini Chat Model
- Process Merged Message
- Format Output

### Node Details

#### Send Typing
- **Type and role:** `n8n-nodes-base.httpRequest`; sends `typing_on` action to Instagram.
- **Configuration choices:**
  - POST to Graph API messages endpoint
  - Recipient is current user
  - Access token in query parameter
  - `onError` set to continue regular output
- **Key expressions/variables used:**
  - URL: `{{ $('Set Context').first().json.ig_id }}`
  - Recipient: `{{ $('User is Sender').first().json.user_id }}`
  - Token: `{{ $('Set Context').first().json.ig_access_token }}`
- **Input and output connections:**
  - Input from `Merge History and Find Min_ID`
  - No downstream connections
- **Version-specific requirements:** typeVersion 4.3.
- **Edge cases/failures:**
  - Same Graph API auth and permission risks as `Seen`
  - Failure does not halt workflow
- **Sub-workflow reference:** none

#### Google Gemini Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; LLM backend for the agent.
- **Configuration choices:**
  - Uses connected `googlePalmApi` credential
  - Default options left mostly empty
- **Key expressions/variables used:** none directly in parameters.
- **Input and output connections:**
  - Connected as AI language model to `Process Merged Message`
- **Version-specific requirements:** typeVersion 1; requires supported n8n LangChain AI nodes and valid Google Gemini/PaLM credential setup.
- **Edge cases/failures:**
  - Credential/auth issues
  - Model quota/rate limits
  - Regional/API availability differences
- **Sub-workflow reference:** none

#### Process Merged Message
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; generates the chatbot reply using merged user input and history.
- **Configuration choices:**
  - Prompt text is `merged_message`
  - Uses a large system prompt defining:
    - Persona: Jenix
    - Service scope: AI and automation consulting for Nguyen Thieu Toan / GenStaff
    - First-message introduction behavior
    - Same-language reply rule
    - Brevity and tone constraints
    - Facebook Markdown output requirement
    - Embedded profile/contact info
    - Old and current session history
  - Retry on fail enabled
  - Wait between tries: 100 ms
- **Key expressions/variables used:**
  - Input text: `{{ $('Get MaxID and Merged Mess').first().json.merged_message }}`
  - History:
    - `{{ $json.old_session_history }}`
    - `{{ $json.now_session_history }}`
  - Current time: `{{ $now }}`
- **Input and output connections:**
  - Main input from `Merge History and Find Min_ID`
  - AI model input from `Google Gemini Chat Model`
  - Output to `Format Output`
- **Version-specific requirements:** typeVersion 3.1; requires AI Agent support in n8n.
- **Edge cases/failures:**
  - If history block is empty due to upstream no-data behavior, agent may not run
  - Prompt relies on conversation history text formatting consistency
  - Generated markdown/HTML may still need downstream cleanup
- **Sub-workflow reference:** none

#### Format Output
- **Type and role:** `n8n-nodes-base.code`; normalizes model output and splits it into chunks for sending.
- **Configuration choices:**
  - Max length: 2000 characters
  - Converts markdown or HTML to Facebook-style formatting
  - Removes unsupported syntax
  - Splits by line boundaries
- **Key expressions/variables used:**
  - Reads `output` from the previous AI node
- **Input and output connections:**
  - Input from `Process Merged Message`
  - Output to `Loop Over Items`
- **Version-specific requirements:** code node typeVersion 2.
- **Edge cases/failures:**
  - If AI output is empty, returns no items
  - Long single lines may still exceed ideal UX boundaries, though splitting occurs by newline-based buffering
  - Link markdown is reduced to link text only
- **Sub-workflow reference:** none

---

## 2.5 Chunk Delivery and State Updates

### Overview
This block loops through each formatted output chunk, sends them sequentially with a delay, then updates persistent conversation state by marking pending messages as processed, saving the page reply, and cleaning excess history.

### Nodes Involved
- Loop Over Items
- Delay Between Messages
- Send Text
- Update FALSE to TRUE
- Update Page Rep
- Clean History

### Node Details

#### Loop Over Items
- **Type and role:** `n8n-nodes-base.splitInBatches`; iterates through each message chunk one by one.
- **Configuration choices:**
  - Default sequential batching behavior
- **Key expressions/variables used:** none.
- **Input and output connections:**
  - Input from `Format Output`
  - Loop branch goes to `Delay Between Messages`
  - Done branch goes to `Update FALSE to TRUE`
- **Version-specific requirements:** typeVersion 3.
- **Edge cases/failures:**
  - If there are zero chunks, post-send update path may not behave as intended
- **Sub-workflow reference:** none

#### Delay Between Messages
- **Type and role:** `n8n-nodes-base.wait`; spaces out chunk sending.
- **Configuration choices:**
  - Amount set to `1`
  - Note says 100 ms, sticky note says 1 second, but node parameter does not explicitly show unit here; verify actual wait unit in your n8n instance
- **Key expressions/variables used:** none.
- **Input and output connections:**
  - Input from `Loop Over Items`
  - Output to `Send Text`
- **Version-specific requirements:** typeVersion 1.1.
- **Edge cases/failures:**
  - Wait node behavior/unit depends on n8n configuration UI
  - Mismatch between note and actual timing can affect message ordering assumptions
- **Sub-workflow reference:** none

#### Send Text
- **Type and role:** `n8n-nodes-base.httpRequest`; sends each text chunk to Instagram DM.
- **Configuration choices:**
  - POST to Graph API messages endpoint
  - `messaging_type = RESPONSE`
  - Sends current chunk in `message.text`
  - Sets `metadata = "bot_rep"`
- **Key expressions/variables used:**
  - URL: `{{ $('Set Context').first().json.ig_id }}`
  - Recipient: `{{ $('User is Sender').first().json.user_id }}`
  - Text: `{{ JSON.stringify($('Format Output').item.json.text) }}`
  - Token: `{{ $('Set Context').first().json.ig_access_token }}`
- **Input and output connections:**
  - Input from `Delay Between Messages`
  - Output back to `Loop Over Items` to continue the loop
- **Version-specific requirements:** typeVersion 4.3.
- **Edge cases/failures:**
  - API auth/permission errors
  - Recipient delivery failure
  - Chunk text escaping issues if malformed
- **Sub-workflow reference:** none

#### Update FALSE to TRUE
- **Type and role:** `n8n-nodes-base.dataTable`; marks all pending rows for this user/page as processed.
- **Configuration choices:**
  - Update rows where:
    - `user_id = current user`
    - `processed is false`
    - `page_id = current page`
  - Sets `processed = true`
- **Key expressions/variables used:**
  - `{{ $('User is Sender').first().json.user_id }}`
  - `{{ $('Set Context').first().json.ig_id }}`
- **Input and output connections:**
  - Input from `Loop Over Items` done branch
  - Output to `Update Page Rep`
- **Version-specific requirements:** Data Table node typeVersion 1.1.
- **Edge cases/failures:**
  - If send failed but this node still runs, messages may be marked processed incorrectly
  - Relies on all pending rows being part of the current response cycle
- **Sub-workflow reference:** none

#### Update Page Rep
- **Type and role:** `n8n-nodes-base.dataTable`; stores the AI-generated reply as page text in the latest merged row.
- **Configuration choices:**
  - Updates `page_text`
  - Filters by:
    - `user_id`
    - current merged row `id`
    - `page_id`
- **Key expressions/variables used:**
  - `page_text = {{ $('Process Merged Message').item.json.output }}`
  - target row id from `{{ $('Get MaxID and Merged Mess').first().json.id }}`
- **Input and output connections:**
  - Input from `Update FALSE to TRUE`
  - Output to `Clean History`
- **Version-specific requirements:** Data Table node typeVersion 1.1.
- **Edge cases/failures:**
  - Only one row receives the page response even if several user rows were merged
  - If the target row no longer exists or id typing mismatches, update may fail silently
- **Sub-workflow reference:** none

#### Clean History
- **Type and role:** `n8n-nodes-base.dataTable`; deletes old processed rows older than the current retained 15-row window.
- **Configuration choices:**
  - Deletes rows where:
    - `user_id = current user`
    - `processed is true`
    - `page_id = current page`
    - `id < min_id`
  - Dry run disabled
- **Key expressions/variables used:**
  - `{{ $('Merge History and Find Min_ID').first().json.min_id }}`
- **Input and output connections:**
  - Input from `Update Page Rep`
  - No downstream nodes
- **Version-specific requirements:** Data Table node typeVersion 1.1.
- **Edge cases/failures:**
  - If `min_id` is missing because history retrieval returned no rows, cleanup may not behave correctly
  - Deletes permanently
- **Sub-workflow reference:** none

---

## 2.6 Documentation and Sticky Notes

### Overview
These nodes do not participate in execution. They document setup, production cautions, workflow sections, and author information.

### Nodes Involved
- Main Overview
- Author Message
- Upgrade Note
- Section 1
- Section 2
- Section 3
- Section 4
- Warning Activate
- Warning Token

### Node Details

#### Main Overview
- **Type and role:** `n8n-nodes-base.stickyNote`; explains purpose, setup, customization, and license.
- **Configuration choices:** large descriptive note.
- **Input and output connections:** none.
- **Edge cases/failures:** none; documentation only.

#### Author Message
- **Type and role:** sticky note; author attribution and support links.
- **Input and output connections:** none.

#### Upgrade Note
- **Type and role:** sticky note; suggests batching and human takeover companion workflows for production.
- **Input and output connections:** none.

#### Section 1
- **Type and role:** sticky note; labels webhook/filtering area.
- **Input and output connections:** none.

#### Section 2
- **Type and role:** sticky note; labels storage/history area.
- **Input and output connections:** none.

#### Section 3
- **Type and role:** sticky note; labels AI processing area.
- **Input and output connections:** none.

#### Section 4
- **Type and role:** sticky note; labels delivery/update area.
- **Input and output connections:** none.

#### Warning Activate
- **Type and role:** sticky note; warns to activate workflow before webhook verification.
- **Input and output connections:** none.

#### Warning Token
- **Type and role:** sticky note; warns to replace the placeholder Instagram access token.
- **Input and output connections:** none.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Instagram Webhook | Webhook | Receives Instagram webhook verification and messaging events |  | Webhook Verification, Is Message? | ## Section 1: Webhook & Validation<br>Receives Instagram DM events via webhook → returns GET verification challenge to Meta → **Is Message?** filters text-only events → **Is from page?** blocks echo messages sent by the page itself → extracts config into **Set Context**. |
| Webhook Verification | Respond to Webhook | Returns Meta verification challenge | Instagram Webhook |  | ## Section 1: Webhook & Validation<br>Receives Instagram DM events via webhook → returns GET verification challenge to Meta → **Is Message?** filters text-only events → **Is from page?** blocks echo messages sent by the page itself → extracts config into **Set Context**. |
| Is Message? | If | Filters for text message events | Instagram Webhook | Set Context | ## Section 1: Webhook & Validation<br>Receives Instagram DM events via webhook → returns GET verification challenge to Meta → **Is Message?** filters text-only events → **Is from page?** blocks echo messages sent by the page itself → extracts config into **Set Context**. |
| Set Context | Set | Extracts message, page id, timestamp, token | Is Message? | Is from page? | ## Section 1: Webhook & Validation<br>Receives Instagram DM events via webhook → returns GET verification challenge to Meta → **Is Message?** filters text-only events → **Is from page?** blocks echo messages sent by the page itself → extracts config into **Set Context**.<br>## Edit this node!<br><br>Replace `ig_access_token` with your **long-lived Instagram Access Token** from:<br>Meta App > Instagram > API Setup with credentials. |
| Is from page? | If | Blocks self-sent page echo messages | Set Context | User is Sender | ## Section 1: Webhook & Validation<br>Receives Instagram DM events via webhook → returns GET verification challenge to Meta → **Is Message?** filters text-only events → **Is from page?** blocks echo messages sent by the page itself → extracts config into **Set Context**. |
| User is Sender | Set | Stores external sender user ID | Is from page? | Insert To Unprocessed, Seen | ## Section 1: Webhook & Validation<br>Receives Instagram DM events via webhook → returns GET verification challenge to Meta → **Is Message?** filters text-only events → **Is from page?** blocks echo messages sent by the page itself → extracts config into **Set Context**. |
| Seen | HTTP Request | Sends mark_seen action to Instagram | User is Sender |  | ## Section 2: Store Message & Load History<br>Inserts new user message into **Data Table** as `unprocessed` → marks as **Seen** → merges all unprocessed rows into one prompt (simple batching) → retrieves last 15 **processed** rows → formats them into old-session / current-session history blocks for the AI. |
| Insert To Unprocessed | Data Table | Stores incoming DM as unprocessed | User is Sender | Get MaxID and Merged Mess | ## Section 2: Store Message & Load History<br>Inserts new user message into **Data Table** as `unprocessed` → marks as **Seen** → merges all unprocessed rows into one prompt (simple batching) → retrieves last 15 **processed** rows → formats them into old-session / current-session history blocks for the AI. |
| Get MaxID and Merged Mess | Code | Merges pending user messages and keeps latest row | Insert To Unprocessed | Get history message | ## Section 2: Store Message & Load History<br>Inserts new user message into **Data Table** as `unprocessed` → marks as **Seen** → merges all unprocessed rows into one prompt (simple batching) → retrieves last 15 **processed** rows → formats them into old-session / current-session history blocks for the AI. |
| Get history message | Data Table | Retrieves processed history for current conversation | Get MaxID and Merged Mess | Get 15 newest rows | ## Section 2: Store Message & Load History<br>Inserts new user message into **Data Table** as `unprocessed` → marks as **Seen** → merges all unprocessed rows into one prompt (simple batching) → retrieves last 15 **processed** rows → formats them into old-session / current-session history blocks for the AI. |
| Get 15 newest rows | Code | Keeps newest 15 history rows in chronological order | Get history message | Merge History and Find Min_ID | ## Section 2: Store Message & Load History<br>Inserts new user message into **Data Table** as `unprocessed` → marks as **Seen** → merges all unprocessed rows into one prompt (simple batching) → retrieves last 15 **processed** rows → formats them into old-session / current-session history blocks for the AI. |
| Merge History and Find Min_ID | Code | Formats session history and computes cleanup threshold | Get 15 newest rows | Send Typing, Process Merged Message | ## Section 2: Store Message & Load History<br>Inserts new user message into **Data Table** as `unprocessed` → marks as **Seen** → merges all unprocessed rows into one prompt (simple batching) → retrieves last 15 **processed** rows → formats them into old-session / current-session history blocks for the AI. |
| Send Typing | HTTP Request | Sends typing indicator to Instagram user | Merge History and Find Min_ID |  | ## Section 3: AI Processing<br>Sends `typing_on` indicator to the user → **Gemini AI Agent** generates a reply using the merged message + full session history → **Format Output** normalizes markdown, strips unsupported syntax, and splits the reply into ≤2000-character chunks. |
| Google Gemini Chat Model | Google Gemini Chat Model | Provides LLM backend for AI agent |  | Process Merged Message | ## Section 3: AI Processing<br>Sends `typing_on` indicator to the user → **Gemini AI Agent** generates a reply using the merged message + full session history → **Format Output** normalizes markdown, strips unsupported syntax, and splits the reply into ≤2000-character chunks. |
| Process Merged Message | AI Agent | Generates response from merged input and history | Merge History and Find Min_ID; Google Gemini Chat Model | Format Output | ## Section 3: AI Processing<br>Sends `typing_on` indicator to the user → **Gemini AI Agent** generates a reply using the merged message + full session history → **Format Output** normalizes markdown, strips unsupported syntax, and splits the reply into ≤2000-character chunks. |
| Format Output | Code | Converts AI output to Facebook-style markdown and splits chunks | Process Merged Message | Loop Over Items | ## Section 3: AI Processing<br>Sends `typing_on` indicator to the user → **Gemini AI Agent** generates a reply using the merged message + full session history → **Format Output** normalizes markdown, strips unsupported syntax, and splits the reply into ≤2000-character chunks. |
| Loop Over Items | Split In Batches | Iterates through message chunks sequentially | Format Output, Send Text | Delay Between Messages, Update FALSE to TRUE | ## Section 4: Deliver & Update<br>Loops through each message chunk → waits 1 second between sends → delivers via **Instagram Graph API** → marks all unprocessed rows `processed = true` → saves AI reply into `page_text` → deletes old history rows beyond the 15-message window. |
| Delay Between Messages | Wait | Adds delay between chunk sends | Loop Over Items | Send Text | ## Section 4: Deliver & Update<br>Loops through each message chunk → waits 1 second between sends → delivers via **Instagram Graph API** → marks all unprocessed rows `processed = true` → saves AI reply into `page_text` → deletes old history rows beyond the 15-message window. |
| Send Text | HTTP Request | Sends DM text chunk to Instagram | Delay Between Messages | Loop Over Items | ## Section 4: Deliver & Update<br>Loops through each message chunk → waits 1 second between sends → delivers via **Instagram Graph API** → marks all unprocessed rows `processed = true` → saves AI reply into `page_text` → deletes old history rows beyond the 15-message window. |
| Update FALSE to TRUE | Data Table | Marks pending rows as processed | Loop Over Items | Update Page Rep | ## Section 4: Deliver & Update<br>Loops through each message chunk → waits 1 second between sends → delivers via **Instagram Graph API** → marks all unprocessed rows `processed = true` → saves AI reply into `page_text` → deletes old history rows beyond the 15-message window. |
| Update Page Rep | Data Table | Saves generated reply into latest row | Update FALSE to TRUE | Clean History | ## Section 4: Deliver & Update<br>Loops through each message chunk → waits 1 second between sends → delivers via **Instagram Graph API** → marks all unprocessed rows `processed = true` → saves AI reply into `page_text` → deletes old history rows beyond the 15-message window. |
| Clean History | Data Table | Deletes processed rows older than retained window | Update Page Rep |  | ## Section 4: Deliver & Update<br>Loops through each message chunk → waits 1 second between sends → delivers via **Instagram Graph API** → marks all unprocessed rows `processed = true` → saves AI reply into `page_text` → deletes old history rows beyond the 15-message window. |
| Main Overview | Sticky Note | Documents workflow purpose, setup, and license |  |  | ## Instagram DM Chatbot with AI & Conversation History<br><br>This workflow turns your **Instagram Business/Creator account** into a fully automated AI chatbot using **Google Gemini** and **n8n Data Table** for persistent conversation history. Every incoming Direct Message is stored, processed, and replied to automatically — with long replies split and delivered sequentially.<br><br>> ⚠️ **Simplified version** — ideal for learning and low-traffic deployments. For production, integrate the Smart Batching and Human Takeover workflows linked in the Upgrade Note.<br><br>### How it works<br>1. **Instagram Webhook** receives incoming DMs and responds to Meta's GET verification challenge.<br>2. **Is Message?** filters text-only events; **Is from page?** blocks echo from the page itself.<br>3. **Set Context** extracts `user_id`, `ig_id`, and `access_token` — single config point.<br>4. New message is saved to **Data Table** as `unprocessed`. Unprocessed rows are merged into one prompt.<br>5. Last 15 **processed rows** are loaded and formatted as old/current-session history for context.<br>6. **Gemini AI Agent** generates a reply using the merged message + full session history.<br>7. Reply is **split** into ≤2000-character chunks and sent sequentially.<br>8. Data Table is **updated**: all rows marked `processed`, AI reply saved, old rows cleaned up.<br><br>### Setup<br>* [ ] Create a **Meta App** → add **Instagram** product.<br>* [ ] Go to Instagram > **API Setup with credentials** → log in to your IG Business/Creator account → copy the **long-lived Access Token**.<br>* [ ] Paste the token into **Set Context** (`ig_access_token` field).<br>* [ ] In Meta App > Instagram > **Webhooks**: paste your n8n production webhook URL + a Verify Token → click Verify.<br>* [ ] Connect **Google Gemini** (`googlePalmApi`) credential in the AI Agent and LLM nodes.<br>* [ ] Create the **n8n Data Table** `insert_message` with columns: `user_id`, `page_id`, `user_text`, `page_text`, `processed` (Boolean).<br>* [ ] Activate the workflow in **production mode BEFORE** verifying the webhook in Meta.<br><br>### Customization tips<br>* **Change AI persona:** Edit the system prompt inside `Process Merged Message` — nothing else needs changing.<br>* **Switch model:** Swap `Google Gemini Chat Model` for any supported LLM sub-node.<br>* **Scale up:** Combine with the Smart Batching or Human Takeover workflows for production readiness.<br><br>### LICENCE<br>This template is shared free of charge. Copyright belongs to Nguyen Thieu Toan (Jay Nguyen). Any copying or modification must credit the author. |
| Author Message | Sticky Note | Author attribution and support links |  |  | ## Author Message<br><br>Hi! I am **Nguyen Thieu Toan (Jay Nguyen)** — a Verified n8n Creator. Thank you for using this template!<br><br>This workflow is shared with you for free. If it brings value to your work, optimizes your operations, or saves you time, you can buy me a coffee here: **[My Donate Website](https://nguyenthieutoan.com/payment/)** *(PayPal, Momo, Bank Transfer)*<br><br>* Website: [nguyenthieutoan.com](https://nguyenthieutoan.com)<br>* Email: me@nguyenthieutoan.com<br>* Company: GenStaff ([genstaff.net](https://genstaff.net))<br>* Socials (Facebook / X / LinkedIn): @nguyenthieutoan<br><br>*Discover more of my automation solutions:* **[Click here](https://n8n.io/creators/nguyenthieutoan/)** |
| Upgrade Note | Sticky Note | Production upgrade guidance |  |  | ## IMPORTANT: Upgrade for Production<br><br>This is a **simplified version** — suitable for learning, testing, or low-traffic use.<br><br>For production, combine with these **100% compatible** workflows (originally built for Facebook Messenger, but fully cross-platform — works with Instagram too):<br><br>* **[Smart message batching](https://n8n.io/workflows/9192):** Waits for the user to finish typing before responding. Prevents duplicate or out-of-order replies.<br>* **[Smart human takeover](https://n8n.io/workflows/11920):** Automatically pauses the bot when an admin replies. Resumes when the admin is done.<br><br>Integrate **one or both** for a production-ready Instagram chatbot. |
| Section 1 | Sticky Note | Visual section label |  |  | ## Section 1: Webhook & Validation<br>Receives Instagram DM events via webhook → returns GET verification challenge to Meta → **Is Message?** filters text-only events → **Is from page?** blocks echo messages sent by the page itself → extracts config into **Set Context**. |
| Section 2 | Sticky Note | Visual section label |  |  | ## Section 2: Store Message & Load History<br>Inserts new user message into **Data Table** as `unprocessed` → marks as **Seen** → merges all unprocessed rows into one prompt (simple batching) → retrieves last 15 **processed** rows → formats them into old-session / current-session history blocks for the AI. |
| Section 3 | Sticky Note | Visual section label |  |  | ## Section 3: AI Processing<br>Sends `typing_on` indicator to the user → **Gemini AI Agent** generates a reply using the merged message + full session history → **Format Output** normalizes markdown, strips unsupported syntax, and splits the reply into ≤2000-character chunks. |
| Section 4 | Sticky Note | Visual section label |  |  | ## Section 4: Deliver & Update<br>Loops through each message chunk → waits 1 second between sends → delivers via **Instagram Graph API** → marks all unprocessed rows `processed = true` → saves AI reply into `page_text` → deletes old history rows beyond the 15-message window. |
| Warning Activate | Sticky Note | Production activation warning |  |  | ## Activate before verifying!<br><br>The webhook verification in Meta App **only works with the production URL**. Activate this workflow in production mode first, then go to Meta App > Instagram > Webhooks to verify. |
| Warning Token | Sticky Note | Token replacement warning |  |  | ## Edit this node!<br><br>Replace `ig_access_token` with your **long-lived Instagram Access Token** from:<br>Meta App > Instagram > API Setup with credentials. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the webhook entry node**
   - Add a **Webhook** node named `Instagram Webhook`.
   - Set path to `nguyenthieutoan-instagram` or your preferred path.
   - Enable **multiple methods**.
   - Set **Response Mode** to **Using Respond to Webhook node**.

2. **Create the verification response node**
   - Add **Respond to Webhook** named `Webhook Verification`.
   - Set response code to `200`.
   - Set response type to **Text**.
   - Set body to:
     - `{{ $json.query['hub.challenge'] }}`
   - Connect `Instagram Webhook` to `Webhook Verification`.
   - If you want to mirror the source workflow, connect it from both webhook outputs.

3. **Create the text-message filter**
   - Add an **If** node named `Is Message?`.
   - Condition: field exists.
   - Expression:
     - `{{ $json.body.entry?.[0]?.messaging?.[0]?.message?.text }}`
   - Connect `Instagram Webhook` to `Is Message?`.

4. **Create the context extraction node**
   - Add a **Set** node named `Set Context`.
   - Create these fields:
     - `message` = `{{ $json.body.entry[0].messaging[0].message.text }}`
     - `ig_id` = `{{ $json.body.entry[0].id }}`
     - `is_from_bot` = `{{ $json.body.entry[0].messaging[0].message.metadata }}`
     - `ig_access_token` = your long-lived Instagram access token
     - `received_timestamp` = `{{ $json.body.entry[0].messaging[0].timestamp }}`
   - Connect the **true** output of `Is Message?` to `Set Context`.

5. **Create the page-echo filter**
   - Add an **If** node named `Is from page?`.
   - Compare:
     - Left: `{{ $('Instagram Webhook').item.json.body.entry[0].messaging[0].sender.id }}`
     - Right: `{{ $('Instagram Webhook').item.json.body.entry[0].id }}`
   - If equal, it means the message originated from the page/account.
   - Connect `Set Context` to `Is from page?`.
   - Use the **false** branch for user-originated messages.

6. **Create the sender ID node**
   - Add a **Set** node named `User is Sender`.
   - Set:
     - `user_id` = `{{ $('Instagram Webhook').item.json.body.entry[0].messaging[0].sender.id }}`
   - Connect the **false** branch of `Is from page?` to this node.

7. **Create the Data Table**
   - In n8n, create a Data Table named `insert_message`.
   - Add these columns:
     - `user_id` — String
     - `user_text` — String
     - `page_id` — String
     - `page_text` — String
     - `processed` — Boolean
   - Optional but recommended: add a `bot_rep` column if you want the history formatter to use it fully, because the code references it.

8. **Create the “seen” API request**
   - Add an **HTTP Request** node named `Seen`.
   - Method: `POST`
   - URL:
     - `https://graph.instagram.com/v24.0/{{ $('Set Context').item.json.ig_id }}/messages`
   - Query parameter:
     - `access_token` = `{{ $('Set Context').item.json.ig_access_token }}`
   - Send JSON body:
     ```json
     {
       "recipient": {
         "id": "{{ $('User is Sender').item.json.user_id }}"
       },
       "sender_action": "mark_seen"
     }
     ```
   - Set **On Error** to continue.
   - Connect `User is Sender` to `Seen`.

9. **Create the incoming-message insert node**
   - Add a **Data Table** node named `Insert To Unprocessed`.
   - Operation: insert/create row.
   - Map:
     - `page_id` = `{{ $('Set Context').item.json.ig_id }}`
     - `user_id` = `{{ $('User is Sender').item.json.user_id }}`
     - `processed` = `false`
     - `user_text` = `{{ $('Set Context').item.json.message }}`
   - Connect `User is Sender` to `Insert To Unprocessed`.

10. **Create the simple batching code node**
    - Add a **Code** node named `Get MaxID and Merged Mess`.
    - Use JavaScript that:
      - sorts incoming items by `id`
      - concatenates `user_text`
      - stores result in `merged_message`
      - returns only the latest row
    - Connect `Insert To Unprocessed` to this node.

11. **Create the processed-history lookup**
    - Add a **Data Table** node named `Get history message`.
    - Operation: get rows.
    - Return all rows.
    - Filter by:
      - `user_id = {{ $('User is Sender').item.json.user_id }}`
      - `processed is true`
      - `page_id = {{ $json.page_id }}`
    - Enable **Always Output Data**.
    - Connect `Get MaxID and Merged Mess` to this node.

12. **Create the “latest 15 rows” code node**
    - Add a **Code** node named `Get 15 newest rows`.
    - Logic:
      - if no rows, return empty
      - sort by `createdAt` descending
      - keep 15
      - resort ascending
    - Enable **Always Output Data**.
    - Connect `Get history message` to it.

13. **Create the session-history formatter**
    - Add a **Code** node named `Merge History and Find Min_ID`.
    - Implement logic to:
      - sort by `createdAt`
      - group messages by Vietnam-local day
      - build `old_session_history`
      - build `now_session_history`
      - compute `min_id`
    - Use timezone `Asia/Ho_Chi_Minh`.
    - Connect `Get 15 newest rows` to it.

14. **Create the typing indicator request**
    - Add an **HTTP Request** node named `Send Typing`.
    - Method: `POST`
    - URL:
      - `https://graph.instagram.com/v24.0/{{ $('Set Context').first().json.ig_id }}/messages`
    - Query parameter:
      - `access_token = {{ $('Set Context').first().json.ig_access_token }}`
    - JSON body:
      ```json
      {
        "recipient": {
          "id": "{{ $('User is Sender').first().json.user_id }}"
        },
        "sender_action": "typing_on"
      }
      ```
    - Set **On Error** to continue.
    - Connect `Merge History and Find Min_ID` to `Send Typing`.

15. **Create the Gemini model node**
    - Add **Google Gemini Chat Model** named `Google Gemini Chat Model`.
    - Attach a valid Google Gemini/PaLM credential.
    - Leave model options default unless you want custom temperature/model selection.

16. **Create the AI agent node**
    - Add an **AI Agent** node named `Process Merged Message`.
    - Prompt type: define manually.
    - Main text input:
      - `{{ $('Get MaxID and Merged Mess').first().json.merged_message }}`
    - System prompt should include:
      - assistant persona
      - language mirroring
      - brevity rules
      - first-message introduction rule
      - company/contact info
      - current time
      - old/current session history
    - Connect:
      - `Merge History and Find Min_ID` to the main input
      - `Google Gemini Chat Model` to the AI language model input
    - Enable retry on fail if desired.

17. **Create the output formatting node**
    - Add a **Code** node named `Format Output`.
    - Logic should:
      - read AI output text
      - normalize escaped newlines
      - convert markdown/HTML into Facebook-style markdown
      - strip unsupported syntax
      - split into chunks of up to 2000 characters
      - output one item per chunk as `{ text: chunk }`
    - Connect `Process Merged Message` to it.

18. **Create the sequential loop**
    - Add **Split In Batches** named `Loop Over Items`.
    - Keep default one-at-a-time behavior.
    - Connect `Format Output` to it.

19. **Create the delay node**
    - Add a **Wait** node named `Delay Between Messages`.
    - Set amount to `1`.
    - Confirm the unit in your n8n version; the workflow notes imply one second between chunks.
    - Connect the loop output of `Loop Over Items` to `Delay Between Messages`.

20. **Create the send message request**
    - Add an **HTTP Request** node named `Send Text`.
    - Method: `POST`
    - URL:
      - `https://graph.instagram.com/v24.0/{{ $('Set Context').first().json.ig_id }}/messages`
    - Query parameter:
      - `access_token = {{ $('Set Context').first().json.ig_access_token }}`
    - JSON body:
      ```json
      {
        "recipient": {
          "id": "{{ $('User is Sender').first().json.user_id }}"
        },
        "messaging_type": "RESPONSE",
        "message": {
          "text": {{JSON.stringify($('Format Output').item.json.text)}},
          "metadata": "bot_rep"
        }
      }
      ```
    - Connect `Delay Between Messages` to `Send Text`.
    - Connect `Send Text` back to `Loop Over Items` to continue iteration.

21. **Create the processed-flag update**
    - Add a **Data Table** node named `Update FALSE to TRUE`.
    - Operation: update rows.
    - Filter:
      - `user_id = {{ $('User is Sender').first().json.user_id }}`
      - `processed is false`
      - `page_id = {{ $('Set Context').first().json.ig_id }}`
    - Set:
      - `processed = true`
    - Connect the **done/no-more-items** output of `Loop Over Items` to this node.

22. **Create the reply persistence node**
    - Add a **Data Table** node named `Update Page Rep`.
    - Operation: update rows.
    - Filter:
      - `user_id = {{ $('User is Sender').first().json.user_id }}`
      - `id = {{ $('Get MaxID and Merged Mess').first().json.id }}`
      - `page_id = {{ $('Set Context').first().json.ig_id }}`
    - Set:
      - `page_text = {{ $('Process Merged Message').item.json.output }}`
    - Connect `Update FALSE to TRUE` to it.

23. **Create the cleanup node**
    - Add a **Data Table** node named `Clean History`.
    - Operation: delete rows.
    - Filter:
      - `user_id = {{ $('User is Sender').first().json.user_id }}`
      - `processed is true`
      - `page_id = {{ $('Set Context').first().json.ig_id }}`
      - `id < {{ $('Merge History and Find Min_ID').first().json.min_id }}`
    - Disable dry run.
    - Connect `Update Page Rep` to `Clean History`.

24. **Add optional sticky notes**
    - Add notes for setup, warnings, and production guidance if you want the workflow documented visually.

25. **Configure credentials and external systems**
    - **Meta / Instagram**
      - Create a Meta App.
      - Add Instagram product.
      - Generate a long-lived Instagram access token.
      - Subscribe the webhook URL in Meta App.
      - Ensure the Instagram account is Business or Creator.
    - **Google Gemini**
      - Create or connect Gemini API credentials in n8n.
      - Attach them to `Google Gemini Chat Model`.

26. **Activate before webhook verification**
    - Turn the workflow on in production.
    - Copy the production webhook URL from `Instagram Webhook`.
    - Register that URL in Meta App > Instagram > Webhooks.
    - Complete verification.

27. **Test with a real Instagram DM**
    - Send a text DM to the connected Instagram account.
    - Confirm:
      - message is inserted into Data Table
      - AI responds
      - processed flag becomes true
      - `page_text` gets stored
      - old rows are cleaned only after enough history accumulates

28. **Recommended improvements for a real deployment**
    - Move `ig_access_token` into credentials or environment variables instead of a Set node
    - Add explicit handling for empty history
    - Add error alerting for failed Graph API calls
    - Add batching debounce and human takeover logic for production traffic

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Main workflow description and setup guidance, including Data Table schema and activation requirement | Embedded in the workflow’s “Main Overview” sticky note |
| Activate the workflow in production before verifying the webhook in Meta | Meta App > Instagram > Webhooks |
| Replace the placeholder `ig_access_token` in `Set Context` with a long-lived Instagram Access Token | Meta App > Instagram > API Setup with credentials |
| Production upgrade: Smart message batching | https://n8n.io/workflows/9192 |
| Production upgrade: Smart human takeover | https://n8n.io/workflows/11920 |
| Author website | https://nguyenthieutoan.com |
| Donate page | https://nguyenthieutoan.com/payment/ |
| Company website | https://genstaff.net |
| Creator profile and more solutions | https://n8n.io/creators/nguyenthieutoan/ |

## Additional implementation notes
- The workflow is a simplified low-traffic design. It does not debounce rapid multi-message bursts beyond merging currently unprocessed rows.
- The history formatter references `bot_rep`, but the provided Data Table schema in the notes does not include that field. If you want AI-specific history lines separate from `page_text`, add a `bot_rep` column and update it explicitly.
- The wait node note and the sticky note disagree slightly with the node note text; verify the real delay unit in your n8n instance.
- Because `Seen` and `Send Typing` continue on error, delivery indicators can fail silently while the workflow still replies.
- Only text messages are processed. Attachments, reactions, story replies without `message.text`, and other event types are ignored by design.