Curate daily tech news for Slack and Telegram using BrowserAct and OpenRouter

https://n8nworkflows.xyz/workflows/curate-daily-tech-news-for-slack-and-telegram-using-browseract-and-openrouter-12349


# Curate daily tech news for Slack and Telegram using BrowserAct and OpenRouter

## 1. Workflow Overview

**Purpose:** This workflow runs daily to scrape the latest tech headlines and product launches (via BrowserAct), then uses two platform-specific AI agents (via OpenRouter) to format the same input into (1) Slack-friendly messages and (2) Telegram Bot API–compatible HTML messages. It automatically splits long briefings into multiple posts and publishes them to a Slack channel and a Telegram channel.

**Primary use cases:**
- Daily internal team “morning brief” in Slack
- Public/community digest posted to a Telegram channel
- A reusable pattern for “scrape → summarize → format per platform → post with split handling”

### 1.1 Schedule & Data Extraction
Runs on a daily schedule and triggers BrowserAct to fetch/compile news items.

### 1.2 Slack Brief Generation (AI + Structured Output)
Sends the extracted data to a Slack-specific agent that formats content using Slack link/bold conventions and returns a JSON `messages[]` array.

### 1.3 Telegram Brief Generation (AI + Structured Output)
Sends the same extracted data to a Telegram-specific agent that outputs HTML-formatted content, enforces safe splitting at ~2500 chars, and returns a JSON `messages[]` array.

### 1.4 Split & Publish
Splits `messages[]` into individual items and publishes each as a separate Slack/Telegram post.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule & Scrape
**Overview:** Triggers once per day and runs a BrowserAct workflow to gather the latest items from sources like The Verge and Product Hunt.

**Nodes involved:**
- `Daily Schedule`
- `Extract Latest News and Products`

#### Node: Daily Schedule
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration:** Runs daily at **10:00** (hour-based rule).
- **Outputs:** Main output to `Extract Latest News and Products`.
- **Edge cases / failures:**
- If n8n instance timezone differs from expectation, posts may run at an unexpected local time (timezone is set at instance/workflow level, not shown here).

#### Node: Extract Latest News and Products
- **Type / role:** `n8n-nodes-browseract.browserAct` — executes a BrowserAct cloud workflow.
- **Configuration choices:**
  - **Mode:** WORKFLOW
  - **BrowserAct workflowId:** `70367719194706940`
  - Uses a BrowserAct template referenced in sticky notes: **“Automated Multi-Site Morning Brief”** (must exist in the BrowserAct account).
  - Input schema includes `theverge` and `producthunt`, but both are marked “removed”; the node effectively relies on BrowserAct’s defaults.
  - `open_incognito_mode`: false
- **Outputs / downstream:**
  - Sends its main output to **both**:
    - `Telegram Content Generation`
    - `Slack Content Generation`
- **Key data contract (assumed by downstream):**
  - Both AI agent nodes read: `{{ $json.output.string }}`  
    This implies BrowserAct returns an object containing `output.string` with the scraped items serialized as a JSON string (or at least a string representation of the list).
- **Edge cases / failures:**
  - BrowserAct auth issues (invalid API key), workflow not found, template missing, execution timeouts.
  - Output shape mismatch (e.g., if BrowserAct returns `output` in a different field) will break the AI nodes’ prompt input.
  - Source website layout changes may reduce scrape quality or return empty results.

---

### Block 2 — Slack Path (AI Formatting → Split → Publish)
**Overview:** Uses a Slack-specialized AI agent to transform the scraped dataset into Slack-formatted text, potentially split into multiple message parts, then posts each part to Slack.

**Nodes involved:**
- `Open Router1`
- `Structured Output1`
- `Slack Content Generation`
- `Split  Data for Slack`
- `Publish to Slack Channel`

#### Node: Open Router1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — LLM provider for the Slack agent and structured parser.
- **Configuration choices:**
  - **Model:** `anthropic/claude-sonnet-4.5`
  - Uses OpenRouter credentials: `OpenRouter account`
- **Connections:**
  - Provides `ai_languageModel` to:
    - `Slack Content Generation`
    - `Structured Output1`
- **Edge cases / failures:**
  - OpenRouter auth / quota / billing errors.
  - Model availability changes (OpenRouter model names can change or be retired).
  - Rate limits during scheduled runs.

#### Node: Structured Output1
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces a JSON schema-like output.
- **Configuration choices:**
  - `autoFix: true` (attempts to correct near-valid JSON produced by the model)
  - Example schema:
    - Output must be an object with `messages: string[]`
- **Connections:**
  - Receives LLM capability from `Open Router1` (as `ai_languageModel`)
  - Feeds into `Slack Content Generation` via `ai_outputParser`
- **Edge cases / failures:**
  - If the model outputs content too far from JSON, auto-fix may fail.
  - If it outputs valid JSON but not matching the expected structure (e.g., `message` instead of `messages`), downstream split will fail.

#### Node: Slack Content Generation
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + LLM + structured output.
- **Configuration choices:**
  - **Prompt input text:** `Data Input : {{ $json.output.string }}`
  - **System message:** Defines “SlackBrief” persona and strict output JSON format:
    - Must return:
      ```json
      { "messages": ["...", "..."] }
      ```
    - Slack syntax rules:
      - Bold: `*bold*`
      - Links: `<url|text>`
      - Bullets: `•`
    - Sorting priorities and section grouping
    - Splitting guidance: aim under **3000 chars**; if too long, split into two strings in `messages[]`
  - `hasOutputParser: true` and connected to `Structured Output1`
- **Outputs / downstream:**
  - Main output to `Split  Data for Slack`
  - The agent output is expected to be available under a field used later as: `output.messages`
- **Edge cases / failures:**
  - If `Data Input` is not valid JSON (or is too large), the model may hallucinate structure or produce malformed output.
  - Character-length guidance is advisory; the model may exceed it unless strongly enforced.
  - Slack formatting issues if the model fails to use `<url|text>` format.
  - If BrowserAct returns empty content, output may be empty or generic.

#### Node: Split  Data for Slack
- **Type / role:** `n8n-nodes-base.splitOut` — converts an array field into multiple items (one per message).
- **Configuration choices:**
  - `fieldToSplitOut: "output.messages"`
- **Connections:**
  - Input from `Slack Content Generation`
  - Output to `Publish to Slack Channel`
- **Edge cases / failures:**
  - If `output.messages` is missing or not an array, node errors.
  - If array contains empty strings, Slack may post blank/near-blank messages.

#### Node: Publish to Slack Channel
- **Type / role:** `n8n-nodes-base.slack` — posts each split message to a Slack channel.
- **Configuration choices:**
  - Operation: send a message to a **channel**
  - `channelId`: `C09KLV9DJSX` (label cached: `all-browseract-workflow-test`)
  - `text`: `{{ $json["output.messages"] }}`  
    After splitting, each item still exposes the single message value at the same path.
- **Edge cases / failures:**
  - Slack auth/token revoked, missing scopes (typically `chat:write`).
  - Posting to a channel where the bot/app isn’t a member.
  - Slack API errors due to rate limits if many message parts are produced.

---

### Block 3 — Telegram Path (AI Formatting → Split → Publish)
**Overview:** Uses a Telegram-specialized AI agent that outputs Telegram-safe HTML, enforces splitting under ~2500 characters, then posts each part to a Telegram channel.

**Nodes involved:**
- `Open Router`
- `Structured Output`
- `Telegram Content Generation`
- `Split Data for Telegram`
- `Publish to Telegram Channel`

#### Node: Open Router
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — LLM provider for Telegram agent and structured parser.
- **Configuration choices:**
  - **Model:** `google/gemini-2.5-pro`
  - OpenRouter credentials: `OpenRouter account`
- **Connections:**
  - Provides `ai_languageModel` to:
    - `Telegram Content Generation`
    - `Structured Output`
- **Edge cases / failures:** Same class as Slack LLM node (auth, quota, model availability, rate limits).

#### Node: Structured Output
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — ensures valid JSON output structure.
- **Configuration choices:**
  - `autoFix: true`
  - Example schema requires:
    - `{ "messages": [ "string...", "string..." ] }`
- **Connections:**
  - Receives LLM capability from `Open Router`
  - Feeds into `Telegram Content Generation` via `ai_outputParser`
- **Edge cases / failures:** As in Slack structured output.

#### Node: Telegram Content Generation
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — platform-specific content generation for Telegram.
- **Configuration choices:**
  - **Prompt input text:** `Data Input : {{ $json.output.string }}`
  - **System message:** Defines “TelegramBriefBot” and strict constraints:
    - Output JSON: `{ "messages": [ ... ] }`
    - Telegram HTML rules: only `<b>`, `<a href="">`, `<code>`, `<i>`
    - Must escape `<`, `>`, `&` inside visible text
    - “Safe Split” limit: **2500 chars** per message
    - Splitting rules: do not split within sentences/tags; split between list items
    - If split occurs, next message starts with: `⬇️ <i>Continued...</i>`
- **Outputs / downstream:**
  - Main output to `Split Data for Telegram`
- **Edge cases / failures:**
  - Telegram is strict about HTML validity; unescaped characters or malformed tags can cause message rejection.
  - The model may still exceed 2500 chars if not carefully bounded.
  - If it includes unsupported tags (e.g., `<br>`), Telegram may reject or strip formatting.

#### Node: Split Data for Telegram
- **Type / role:** `n8n-nodes-base.splitOut` — turns `messages[]` into multiple posting items.
- **Configuration choices:**
  - `fieldToSplitOut: "output.messages"`
- **Connections:** Output to `Publish to Telegram Channel`
- **Edge cases / failures:** Same as Slack split node.

#### Node: Publish to Telegram Channel
- **Type / role:** `n8n-nodes-base.telegram` — sends each message to a Telegram chat/channel.
- **Configuration choices:**
  - `text`: `{{ $json["output.messages"] }}`
  - `parse_mode`: `HTML`
  - `chatId`: **misconfigured placeholder expression**: `chatId=="Your Telegram Channel ID"`
    - This is not a valid n8n expression for a chat ID. It should be a string/number like `"-1001234567890"` or an expression that evaluates to it.
- **Edge cases / failures:**
  - Invalid chat ID will fail every run.
  - Bot not added to channel or lacks admin rights for posting (common requirement for channels).
  - HTML parse errors if the content violates Telegram HTML constraints.

---

### Block 4 — Documentation / Annotations
**Overview:** Sticky notes provide setup requirements, usage instructions, and a YouTube link.

**Nodes involved:**
- `Documentation` (sticky note)
- `Step 1 Explanation` (sticky note)
- `Step 2a Explanation` (sticky note)
- `Step 3 Explanation` (sticky note)
- `Sticky Note` (YouTube link)

#### Sticky Note: Documentation
- **Role:** Human guidance: required credentials, BrowserAct template requirement, schedule customization, links to BrowserAct docs.
- **Links:**
  - https://docs.browseract.com (API key & workflow ID)
  - https://docs.browseract.com (connect n8n)
  - https://docs.browseract.com (templates)

#### Sticky Note: Step 1 Explanation
- Describes schedule + BrowserAct scraping.

#### Sticky Note: Step 2a Explanation
- Describes platform-specific formatting via AI agent.

#### Sticky Note: Step 3 Explanation
- Describes splitting and delivery behavior.

#### Sticky Note: Sticky Note (YouTube)
- Contains: `@[youtube](MIvUFkpobvc)` (reference to a video ID)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Documentation | Sticky Note | Setup/overview annotation | — | — | ## ⚡ Workflow Overview & Setup; Requirements: BrowserAct, OpenRouter, Slack, Telegram; Mandatory BrowserAct template “Automated Multi-Site Morning Brief”; schedule default 10:00 AM; Links: https://docs.browseract.com |
| Sticky Note | Sticky Note | Video reference | — | — | @[youtube](MIvUFkpobvc) |
| Step 1 Explanation | Sticky Note | Annotation for scheduling/scraping | — | — | ### 📅 Step 1: Schedule & Scrape — runs BrowserAct to gather headlines/launches |
| Step 2a Explanation | Sticky Note | Annotation for AI formatting | — | — | ### 💬 Step 2: Apply Platform-Specific Formatting — AI groups items by category |
| Step 3 Explanation | Sticky Note | Annotation for split/delivery | — | — | ### 🚀 Step 3: Split & Deliver — AI decides split points; Split node executes multiple posts |
| Daily Schedule | Schedule Trigger | Daily entry point | — | Extract Latest News and Products | ### 📅 Step 1: Schedule & Scrape — runs BrowserAct to gather headlines/launches |
| Extract Latest News and Products | BrowserAct | Scrape/compile news via BrowserAct workflow | Daily Schedule | Telegram Content Generation; Slack Content Generation | ### 📅 Step 1: Schedule & Scrape — runs BrowserAct to gather headlines/launches |
| Open Router1 | OpenRouter Chat Model | LLM provider for Slack path | — | Slack Content Generation (AI model); Structured Output1 (AI model) | ### 💬 Step 2: Apply Platform-Specific Formatting — AI groups items by category |
| Structured Output1 | Structured Output Parser | Enforces JSON `{messages: []}` for Slack | — (AI model from Open Router1) | Slack Content Generation (AI output parser) | ### 💬 Step 2: Apply Platform-Specific Formatting — AI groups items by category |
| Slack Content Generation | LangChain Agent | Generates Slack-formatted briefing JSON | Extract Latest News and Products; (AI model from Open Router1); (parser from Structured Output1) | Split  Data for Slack | ### 💬 Step 2: Apply Platform-Specific Formatting — AI groups items by category |
| Split  Data for Slack | Split Out | Split `output.messages[]` into separate items | Slack Content Generation | Publish to Slack Channel | ### 🚀 Step 3: Split & Deliver — AI decides split points; Split node executes multiple posts |
| Publish to Slack Channel | Slack | Post each message part to Slack channel | Split  Data for Slack | — | ### 🚀 Step 3: Split & Deliver — AI decides split points; Split node executes multiple posts |
| Open Router | OpenRouter Chat Model | LLM provider for Telegram path | — | Telegram Content Generation (AI model); Structured Output (AI model) | ### 💬 Step 2: Apply Platform-Specific Formatting — AI groups items by category |
| Structured Output | Structured Output Parser | Enforces JSON `{messages: []}` for Telegram | — (AI model from Open Router) | Telegram Content Generation (AI output parser) | ### 💬 Step 2: Apply Platform-Specific Formatting — AI groups items by category |
| Telegram Content Generation | LangChain Agent | Generates Telegram HTML briefing JSON | Extract Latest News and Products; (AI model from Open Router); (parser from Structured Output) | Split Data for Telegram | ### 💬 Step 2: Apply Platform-Specific Formatting — AI groups items by category |
| Split Data for Telegram | Split Out | Split `output.messages[]` into separate items | Telegram Content Generation | Publish to Telegram Channel | ### 🚀 Step 3: Split & Deliver — AI decides split points; Split node executes multiple posts |
| Publish to Telegram Channel | Telegram | Post each message part to Telegram channel | Split Data for Telegram | — | ### 🚀 Step 3: Split & Deliver — AI decides split points; Split node executes multiple posts |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `Curate daily tech news for Slack and Telegram using BrowserAct & OpenRouter`

2. **Add node: “Daily Schedule”**
   - Type: **Schedule Trigger**
   - Configure: run daily at **10:00** (adjust as desired).

3. **Add node: “Extract Latest News and Products”**
   - Type: **BrowserAct**
   - Resource/Mode: **WORKFLOW**
   - Set **Workflow ID** to: `70367719194706940`
   - Leave inputs empty so BrowserAct uses template defaults (ensure your BrowserAct workflow/template already defines targets like The Verge and Product Hunt).
   - Credential: **BrowserAct API** key in n8n (create BrowserAct credential).

4. **Connect:** `Daily Schedule` → `Extract Latest News and Products`

---

### Slack branch

5. **Add node: “Open Router1”**
   - Type: **OpenRouter Chat Model**
   - Model: `anthropic/claude-sonnet-4.5`
   - Credential: **OpenRouter API** (create OpenRouter credential in n8n)

6. **Add node: “Structured Output1”**
   - Type: **Structured Output Parser**
   - Enable **Auto-fix**
   - Provide schema/example:
     - Object with `messages` as an array of strings (two parts max).

7. **Add node: “Slack Content Generation”**
   - Type: **AI Agent (LangChain Agent)**
   - Prompt/Text field: `Data Input : {{ $json.output.string }}`
   - System message: paste the SlackBrief instructions (Slack formatting, grouping, split guidance, strict JSON output).
   - Enable/Use output parser (so it produces structured `output`).
   - Connect AI sockets:
     - Connect **Open Router1** to the agent’s **Language Model** input.
     - Connect **Structured Output1** to the agent’s **Output Parser** input.

8. **Add node: “Split  Data for Slack”**
   - Type: **Split Out**
   - Field to split out: `output.messages`

9. **Add node: “Publish to Slack Channel”**
   - Type: **Slack**
   - Operation: send message to **channel**
   - Channel: pick your target channel (or set channel ID)
   - Text: `{{ $json["output.messages"] }}`
   - Credential: Slack OAuth credential with `chat:write` and access to the channel.

10. **Connect Slack path:**
   - `Extract Latest News and Products` → `Slack Content Generation`
   - `Slack Content Generation` → `Split  Data for Slack`
   - `Split  Data for Slack` → `Publish to Slack Channel`

---

### Telegram branch

11. **Add node: “Open Router”**
   - Type: **OpenRouter Chat Model**
   - Model: `google/gemini-2.5-pro`
   - Credential: same OpenRouter credential (or another if desired)

12. **Add node: “Structured Output”**
   - Type: **Structured Output Parser**
   - Enable **Auto-fix**
   - Provide schema/example requiring `{ "messages": [ ... ] }`

13. **Add node: “Telegram Content Generation”**
   - Type: **AI Agent (LangChain Agent)**
   - Prompt/Text: `Data Input : {{ $json.output.string }}`
   - System message: paste the TelegramBriefBot instructions (HTML-only tags, escaping rules, 2500 char safe split, “Continued…” prefix).
   - Connect AI sockets:
     - **Open Router** → agent **Language Model**
     - **Structured Output** → agent **Output Parser**

14. **Add node: “Split Data for Telegram”**
   - Type: **Split Out**
   - Field to split out: `output.messages`

15. **Add node: “Publish to Telegram Channel”**
   - Type: **Telegram**
   - Operation: send message
   - Text: `{{ $json["output.messages"] }}`
   - Additional field: `parse_mode = HTML`
   - **Chat ID:** set to your real channel/chat id (example: `-1001234567890`)
     - Replace the placeholder currently shown in the workflow.
   - Credential: Telegram bot token credential in n8n.
   - Ensure the bot is allowed to post in the channel (often requires adding bot as admin).

16. **Connect Telegram path:**
   - `Extract Latest News and Products` → `Telegram Content Generation`
   - `Telegram Content Generation` → `Split Data for Telegram`
   - `Split Data for Telegram` → `Publish to Telegram Channel`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| BrowserAct setup: requires BrowserAct, OpenRouter, Slack, Telegram credentials; BrowserAct template “Automated Multi-Site Morning Brief”; schedule default 10:00 AM | https://docs.browseract.com |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | `@[youtube](MIvUFkpobvc)` |

**Important correction to apply:** The Telegram node’s `chatId` is currently not a usable value; replace it with the real numeric chat/channel ID (or a valid expression that evaluates to it).