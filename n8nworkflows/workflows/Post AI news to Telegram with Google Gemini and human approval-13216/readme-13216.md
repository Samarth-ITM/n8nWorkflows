Post AI news to Telegram with Google Gemini and human approval

https://n8nworkflows.xyz/workflows/post-ai-news-to-telegram-with-google-gemini-and-human-approval-13216


# Post AI news to Telegram with Google Gemini and human approval

## 1. Workflow Overview

**Purpose:** Automatically fetch the latest AI news from two RSS feeds, select the most recent article, generate a Telegram-ready summary using **Google Gemini**, and **publish it only after a human approves** the draft via Telegram.

**Typical use cases:**
- Daily/recurring AI news digest for a Telegram channel
- Human-in-the-loop content automation (draft → approval → publish)
- Lightweight editorial workflow for social/news channels

### Logical Blocks
**1.1 Scheduling & Feed Fetching**
- Triggers on a schedule and pulls items from two RSS feeds.

**1.2 Aggregation & Selection**
- Merges the feeds and keeps only the most recent item.

**1.3 AI Draft Generation (Gemini via LangChain)**
- Prepares a prompt and uses Gemini as the LLM behind a chain node to generate a Telegram post draft.

**1.4 Human Approval & Publishing**
- Sends the draft to Telegram using “send and wait”, checks approval, then publishes if approved.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Feed Fetching
**Overview:** Runs daily at a configured time, then pulls recent articles from VentureBeat’s AI category and The AI Blog RSS feed.

**Nodes Involved:**
- **Schedule trigger**
- **Fetch VentureBeat AI feed**
- **Fetch AI Blog feed**

#### Node: Schedule trigger
- **Type / Role:** `scheduleTrigger` — workflow entry point.
- **Config choices:** Runs at **15:56** (server/workflow timezone).
- **Connections:**
  - Outputs to **Fetch VentureBeat AI feed** and **Fetch AI Blog feed** in parallel.
- **Edge cases / failures:**
  - Timezone mismatch (server vs expected local time).
  - If the instance is sleeping/down at trigger time, the run may be missed depending on hosting.

#### Node: Fetch VentureBeat AI feed
- **Type / Role:** `rssFeedRead` — reads RSS and outputs feed items.
- **Config choices:** URL = `https://venturebeat.com/category/ai/feed/`
- **Outputs:** Items containing typical RSS fields (e.g., `title`, `link`, `contentSnippet`, `isoDate` depending on parser output).
- **Connections:** To **Merge Feeds** (input index 0).
- **Edge cases / failures:**
  - Feed temporarily unavailable (HTTP errors, timeouts).
  - RSS schema differences; some items may miss `isoDate` or `contentSnippet`.
  - Rate limiting or bot protection.

#### Node: Fetch AI Blog feed
- **Type / Role:** `rssFeedRead`
- **Config choices:** URL = `https://feeds.feedburner.com/TheAIBlog`
- **Connections:** To **Merge Feeds** (input index 1).
- **Edge cases / failures:** Same as above; Feedburner feeds can sometimes return cached/limited results.

---

### 2.2 Aggregation & Selection
**Overview:** Combines items from both feeds and selects the single most recent article based on `isoDate`.

**Nodes Involved:**
- **Merge Feeds**
- **Keep Top 1 Article**

#### Node: Merge Feeds
- **Type / Role:** `merge` — combines two input streams.
- **Config choices:** Uses default Merge behavior (no explicit mode shown in parameters).
- **Inputs:**
  - Input 0: VentureBeat feed items
  - Input 1: AI Blog feed items
- **Output:** A combined set of items (behavior depends on default merge mode).
- **Connections:** To **Keep Top 1 Article**
- **Edge cases / failures:**
  - With default merge settings, output shape may differ from a simple “append”; if the merge mode is not “append”, you can unexpectedly lose items or get paired/merged objects rather than a concatenated list.
  - If one feed returns zero items, merge behavior may produce empty output depending on configuration.

#### Node: Keep Top 1 Article
- **Type / Role:** `code` — sorts and slices items.
- **Config choices (interpreted):**
  - Sorts descending by `new Date(item.json.isoDate)`
  - Returns only the first item (most recent)
- **Key code logic:**
  - `items.sort((a,b) => new Date(b.json.isoDate) - new Date(a.json.isoDate))`
  - `return sorted.slice(0, 1)`
- **Connections:** To **Prepare Gemini Prompt**
- **Edge cases / failures:**
  - If `isoDate` is missing/invalid, `new Date(...)` becomes invalid and sorting can behave unpredictably.
  - If `items` is empty, output will be empty and downstream nodes may fail due to missing fields.

---

### 2.3 AI Draft Generation (Gemini via LangChain)
**Overview:** Formats a prompt from the selected article and generates a short Telegram post draft using a LangChain LLM chain node backed by Google Gemini.

**Nodes Involved:**
- **Prepare Gemini Prompt**
- **Generate Telegram post (Gemini)** (LLM provider node)
- **Generate post summary** (LLM chain node)

#### Node: Prepare Gemini Prompt
- **Type / Role:** `code` — builds a `chatInput` string.
- **Config choices (interpreted):**
  - For each item, outputs JSON with:
    - `chatInput: "Summarize this article for a LinkedIn post:\nTitle: ...\nSummary: ...\nLink: ..."`
- **Key variables used:**
  - `item.json.title`
  - `item.json.contentSnippet`
  - `item.json.link`
- **Connections:** To **Generate post summary**
- **Edge cases / failures:**
  - If `contentSnippet` is missing, prompt quality degrades (summary becomes “undefined”).
  - Prompt says “LinkedIn post” but downstream prompt asks for a Telegram post—this mismatch can reduce consistency.

#### Node: Generate Telegram post (Gemini)
- **Type / Role:** `lmChatGoogleGemini` — provides Gemini chat model to LangChain nodes.
- **Config choices:** Uses attached **Google Gemini(PaLM) API** credential; default model/options not explicitly shown.
- **Connections:**
  - Connected to **Generate post summary** via the special `ai_languageModel` connection.
- **Version-specific notes:**
  - Requires n8n’s LangChain integration nodes (`@n8n/n8n-nodes-langchain...`) and a configured Google Gemini/PaLM credential.
- **Edge cases / failures:**
  - Credential issues (invalid key, quota exceeded).
  - Model refusal or empty output if prompt triggers safety filters.
  - Latency/timeouts on LLM calls.

#### Node: Generate post summary
- **Type / Role:** `chainLlm` — runs an LLM chain using the connected language model node.
- **Config choices (interpreted):**
  - Uses one AI message template that instructs:
    - “creative social media manager”
    - Summarize “today’s top AI news from these articles: {{JSON.stringify($json)}}”
    - Tone + 2–3 hashtags + CTA “Read more below 👇”
    - Output suitable for posting
  - `promptType` set to auto.
- **Important detail about inputs:**
  - This node is fed by **Prepare Gemini Prompt**, which produces `{ chatInput: ... }`.
  - The prompt in this node uses `{{JSON.stringify($json)}}`, so it will stringify the incoming JSON (which likely includes `chatInput`, not the full article object unless you pass it through).
- **Connections:** To **Request approval via Telegram**
- **Edge cases / failures:**
  - If the node expects a specific input field (commonly `chatInput`) but the prompt ignores it, output may be suboptimal.
  - Output field naming: downstream assumes `{{$json.text}}` exists.
  - If the chain returns content under a different field (e.g., `response`, `output`, etc.), Telegram approval message will be blank.

---

### 2.4 Human Approval & Publishing
**Overview:** Sends the generated draft to Telegram and waits for human approval. Only if approved, publishes the final post to the target chat/channel.

**Nodes Involved:**
- **Request approval via Telegram**
- **Check approval result**
- **Publish approved post to Telegram**

#### Node: Request approval via Telegram
- **Type / Role:** `telegram` — “send and wait” approval interaction.
- **Config choices (interpreted):**
  - **Operation:** `sendAndWait`
  - **Chat ID:** `123456789`
  - **Message:** `={{ $json.text }}`
  - **Approval options:** `approvalType: double` (typically means explicit approve/reject flow)
  - Uses Telegram bot credential “Telegram account 3”
- **Input expectations:**
  - Expects upstream to provide `$json.text` containing the draft post.
- **Output:**
  - Produces a result object including approval data at `$json.data.approved` (as referenced later).
- **Connections:** To **Check approval result**
- **Edge cases / failures:**
  - Wrong chat/channel ID (bot can’t post or can’t receive responses).
  - Bot permissions missing (especially for channels vs private chats).
  - If `$json.text` is missing, approval request will send an empty message.
  - “Send and wait” can stall runs until timeout if no human responds.

#### Node: Check approval result
- **Type / Role:** `if` — gates publishing.
- **Condition:** `={{ $json.data.approved }}` is **true** (boolean true check).
- **Connections:**
  - **True branch:** to **Publish approved post to Telegram**
  - False branch is not connected (workflow ends without publishing).
- **Edge cases / failures:**
  - If `data.approved` is missing or not boolean, strict validation may evaluate to false or error depending on runtime.
  - If Telegram node changes output schema in a future version, condition may break.

#### Node: Publish approved post to Telegram
- **Type / Role:** `telegram` — final send message.
- **Config choices:**
  - **Chat ID:** `123456789`
  - **Text expression:** `={{ $('Generate post summary').item.json.text }}`
    - Explicitly references the output of **Generate post summary** (not the approval node output).
- **Connections:** End node.
- **Edge cases / failures:**
  - If “Generate post summary” did not produce `.json.text`, publish will send empty text or fail.
  - If the approval step involved edits/feedback, those are not incorporated—this always posts the original generated text unless manually changed elsewhere.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule trigger | Schedule Trigger | Entry point / timed execution | — | Fetch VentureBeat AI feed; Fetch AI Blog feed | Post AI news to Telegram with Google Gemini and human approval… (setup steps and explanation) |
| Fetch VentureBeat AI feed | RSS Feed Read | Fetch RSS items | Schedule trigger | Merge Feeds | Data Collection: Fetches and merges recent AI articles from RSS feeds, then selects the top one. |
| Fetch AI Blog feed | RSS Feed Read | Fetch RSS items | Schedule trigger | Merge Feeds | Data Collection: Fetches and merges recent AI articles from RSS feeds, then selects the top one. |
| Merge Feeds | Merge | Combine feed items | Fetch VentureBeat AI feed; Fetch AI Blog feed | Keep Top 1 Article | Data Collection: Fetches and merges recent AI articles from RSS feeds, then selects the top one. |
| Keep Top 1 Article | Code | Sort by date and keep most recent | Merge Feeds | Prepare Gemini Prompt | Data Collection: Fetches and merges recent AI articles from RSS feeds, then selects the top one. |
| Prepare Gemini Prompt | Code | Build LLM input prompt (`chatInput`) | Keep Top 1 Article | Generate post summary | AI Summarization: Prepares prompt and uses Google Gemini to generate a Telegram-friendly post draft. |
| Generate Telegram post (Gemini) | Google Gemini Chat Model (LangChain) | LLM provider for chain | — (model connection) | Generate post summary (ai_languageModel) | AI Summarization: Prepares prompt and uses Google Gemini to generate a Telegram-friendly post draft. |
| Generate post summary | LangChain Chain LLM | Generate Telegram draft text | Prepare Gemini Prompt + Gemini model | Request approval via Telegram | AI Summarization: Prepares prompt and uses Google Gemini to generate a Telegram-friendly post draft. |
| Request approval via Telegram | Telegram | Send draft + wait for approval | Generate post summary | Check approval result | Approval & Publishing: Requests human approval in Telegram, checks result, and posts if approved. |
| Check approval result | IF | Gate publishing based on approval | Request approval via Telegram | Publish approved post to Telegram (true) | Approval & Publishing: Requests human approval in Telegram, checks result, and posts if approved. |
| Publish approved post to Telegram | Telegram | Publish final approved message | Check approval result | — | Approval & Publishing: Requests human approval in Telegram, checks result, and posts if approved. |
| Workflow Explanation | Sticky Note | Documentation / context | — | — | Post AI news to Telegram with Google Gemini and human approval… (setup steps and explanation) |
| Group: Data Collection | Sticky Note | Visual grouping | — | — | Data Collection: Fetches and merges recent AI articles from RSS feeds, then selects the top one. |
| Group: AI Summarization | Sticky Note | Visual grouping | — | — | AI Summarization: Prepares prompt and uses Google Gemini to generate a Telegram-friendly post draft. |
| Group: Approval & Publishing | Sticky Note | Visual grouping | — | — | Approval & Publishing: Requests human approval in Telegram, checks result, and posts if approved. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule trigger” (Schedule Trigger node)**
   - Set one interval entry: **Hour = 15**, **Minute = 56** (adjust timezone/time as needed).

2) **Add two RSS nodes**
   - Node A: **RSS Feed Read** named “Fetch VentureBeat AI feed”
     - URL: `https://venturebeat.com/category/ai/feed/`
   - Node B: **RSS Feed Read** named “Fetch AI Blog feed”
     - URL: `https://feeds.feedburner.com/TheAIBlog`
   - Connect **Schedule trigger →** both RSS nodes.

3) **Add “Merge Feeds” (Merge node)**
   - Keep default configuration (or explicitly set mode to “Append” to ensure concatenation).
   - Connect:
     - VentureBeat RSS → Merge input 0
     - AI Blog RSS → Merge input 1

4) **Add “Keep Top 1 Article” (Code node)**
   - Paste code (adapt if your RSS node uses different date field):
     - Sort by `isoDate` descending and `slice(0,1)`.
   - Connect **Merge Feeds → Keep Top 1 Article**.

5) **Add “Prepare Gemini Prompt” (Code node)**
   - Build a `chatInput` string using fields like `title`, `contentSnippet`, `link`.
   - Connect **Keep Top 1 Article → Prepare Gemini Prompt**.

6) **Add Gemini model node**
   - Add **“Google Gemini Chat Model”** node (LangChain) named “Generate Telegram post (Gemini)”.
   - Configure credentials:
     - Create/attach **Google Gemini (PaLM) API** credential in n8n (API key).
   - Leave model options default unless you need a specific model/version.

7) **Add LLM chain node**
   - Add **LangChain “Chain LLM”** node named “Generate post summary”.
   - In its message/prompt template, include your Telegram-post requirements (hashtags + CTA).
   - Connect:
     - **Prepare Gemini Prompt → Generate post summary** (main)
     - **Generate Telegram post (Gemini) → Generate post summary** (ai_languageModel connection)

8) **Add Telegram approval node**
   - Add Telegram node named “Request approval via Telegram”
   - Operation: **Send and Wait**
   - Chat ID: your target (private chat ID, group ID, or channel ID where the bot can interact)
   - Message: use the generated draft field (ensure it matches your chain output, e.g. `{{$json.text}}`)
   - Approval type: **double**
   - Credentials:
     - Create a **Telegram Bot** via BotFather, paste token into n8n Telegram credentials.
   - Connect **Generate post summary → Request approval via Telegram**.

9) **Add “Check approval result” (IF node)**
   - Condition: `{{$json.data.approved}}` **is true** (boolean).
   - Connect **Request approval via Telegram → Check approval result**.

10) **Add final publish node**
   - Add Telegram node named “Publish approved post to Telegram”
   - Operation: send message
   - Chat ID: same target chat/channel (or a channel ID if publishing to channel)
   - Text: reference the draft text (here it references the earlier node):  
     - `{{$('Generate post summary').item.json.text}}`
   - Connect **Check approval result (true) → Publish approved post to Telegram**.

11) **(Optional) Add sticky notes**
   - Add notes for “Data Collection”, “AI Summarization”, “Approval & Publishing”, and a general workflow explanation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Post AI news to Telegram with Google Gemini and human approval… Setup steps: configure RSS URLs, add Google Gemini API credentials, set up Telegram bot and target chat ID, adjust schedule, activate.” | Workflow Explanation sticky note content |
| Prompt inconsistency: “Prepare Gemini Prompt” asks for a LinkedIn post, while “Generate post summary” asks for a Telegram post. | Consider aligning both prompts to improve output consistency |
| Merge node default mode may not be “Append” in all contexts; explicitly setting append/concatenate avoids losing items. | Reliability note for feed aggregation |
| Approval publishes the original generated text (from “Generate post summary”), not a potentially edited text from approver feedback. | If you want edits, capture the approved/edited message content and publish that instead |