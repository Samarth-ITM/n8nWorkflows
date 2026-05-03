Summarize Bitcoin news in Japanese and post to Discord with Gemini AI

https://n8nworkflows.xyz/workflows/summarize-bitcoin-news-in-japanese-and-post-to-discord-with-gemini-ai-14213


# Summarize Bitcoin news in Japanese and post to Discord with Gemini AI

# 1. Workflow Overview

This workflow periodically pulls the latest Bitcoin-related news from CoinDesk, summarizes each selected article in Japanese using Google Gemini, posts the result to Discord, logs the generated content to Google Sheets, and sends a confirmation to Slack.

Although the title provided by the user mentions posting to Discord, the internal workflow name still says “post to X with Gemini AI.” The actual implementation posts to **Discord**, then logs to **Google Sheets**, then notifies **Slack**.

## 1.1 Scheduled Trigger and Centralized Configuration

The workflow starts every 6 hours via a Schedule Trigger. A Set node immediately defines reusable configuration values such as the RSS URL, article count, language, hashtags, and a Google Sheet ID placeholder.

## 1.2 News Ingestion and Limiting

The workflow reads an RSS feed from CoinDesk and limits the resulting item list to the latest 3 articles to avoid over-posting.

## 1.3 AI Summarization and Structured Parsing

Each selected RSS item is passed into a LangChain-based LLM chain powered by Google Gemini. The model is instructed to return a strict JSON object containing a Japanese summary, a short Japanese comment, the article URL, and hashtags. A Structured Output Parser is attached to enforce the expected schema.

## 1.4 Distribution, Logging, and Run Confirmation

The generated content is posted to Discord via webhook, appended to a Google Sheet for recordkeeping, and then a Slack message confirms successful posting.

## 1.5 Visual Documentation Notes

The workflow includes multiple sticky notes that describe setup, ingestion, AI processing, and distribution. These notes are useful operational annotations and are reflected later in the summary table.

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger and Configuration

### Overview

This block initializes the workflow on a recurring schedule and stores user-editable settings in one place. It acts as the control layer for timing and some intended configuration centralization, although not all downstream nodes actually consume these values dynamically.

### Nodes Involved

- Run every 6 hours
- Set config values

### Node Details

#### 1. Run every 6 hours

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a recurring interval.
- **Configuration choices:**  
  Configured to run every 6 hours using an interval rule.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none, as this is an entry point
  - Output: `Set config values`
- **Version-specific requirements:**  
  Uses `typeVersion` 1.2. Standard Schedule Trigger behavior in recent n8n versions.
- **Edge cases or potential failure types:**  
  - Workflow must be activated, otherwise it will not run automatically.
  - Instance timezone may affect perceived schedule timing.
  - If the n8n instance is down during the scheduled time, execution behavior depends on deployment/runtime behavior.
- **Sub-workflow reference:**  
  None.

#### 2. Set config values

- **Type and technical role:** `n8n-nodes-base.set`  
  Adds configuration fields into the execution data stream.
- **Configuration choices:**  
  Defines the following fields:
  - `config_rss_url` = `https://feeds.feedburner.com/CoinDesk`
  - `config_articles_to_fetch` = `3`
  - `config_tweet_language` = `Japanese`
  - `config_hashtags` = `#Bitcoin #BTC #仮想通貨`
  - `config_google_sheet_id` = `YOUR_GOOGLE_SHEET_ID_HERE `
- **Key expressions or variables used:**  
  Static values only.
- **Input and output connections:**  
  - Input: `Run every 6 hours`
  - Output: `Fetch Bitcoin RSS feed`
- **Version-specific requirements:**  
  Uses `typeVersion` 3.4.
- **Edge cases or potential failure types:**  
  - No major runtime risk here.
  - The values are not consistently used downstream. For example:
    - The RSS node uses a hardcoded URL instead of `config_rss_url`
    - The Google Sheets node uses a hardcoded document ID instead of `config_google_sheet_id`
    - The Discord webhook URL is not stored here despite the sticky note claiming it is
  - This can create maintenance confusion.
- **Sub-workflow reference:**  
  None.

---

## Block 2 — RSS Ingestion and Article Selection

### Overview

This block fetches the CoinDesk RSS feed and keeps only the latest 3 items. It reduces content volume before AI processing and posting.

### Nodes Involved

- Fetch Bitcoin RSS feed
- Get latest 3 articles

### Node Details

#### 3. Fetch Bitcoin RSS feed

- **Type and technical role:** `n8n-nodes-base.rssFeedRead`  
  Reads and parses entries from an RSS feed URL.
- **Configuration choices:**  
  - URL: `https://feeds.feedburner.com/CoinDesk`
  - `ignoreSSL`: `false`
- **Key expressions or variables used:**  
  None; the URL is hardcoded.
- **Input and output connections:**  
  - Input: `Set config values`
  - Output: `Get latest 3 articles`
- **Version-specific requirements:**  
  Uses `typeVersion` 1.1.
- **Edge cases or potential failure types:**  
  - Feed unavailable or rate-limited
  - Invalid RSS/XML response
  - Temporary network/DNS/TLS issues
  - Feed content may not be Bitcoin-only; CoinDesk is broader crypto/news media
- **Sub-workflow reference:**  
  None.

#### 4. Get latest 3 articles

- **Type and technical role:** `n8n-nodes-base.itemLists`  
  Limits the number of incoming items.
- **Configuration choices:**  
  - Operation: `limit`
  - Maximum items: `3`
- **Key expressions or variables used:**  
  None; value is hardcoded rather than using `config_articles_to_fetch`.
- **Input and output connections:**  
  - Input: `Fetch Bitcoin RSS feed`
  - Output: `Basic LLM Chain`
- **Version-specific requirements:**  
  Uses `typeVersion` 3.1.
- **Edge cases or potential failure types:**  
  - If fewer than 3 RSS items are returned, only available items are passed through.
  - The order depends on the RSS node output order; typically newest first, but this assumes the feed is correctly sorted.
- **Sub-workflow reference:**  
  None.

---

## Block 3 — AI Summarization and Structured Output

### Overview

This block transforms each RSS article into a concise Japanese social-style post. It relies on Gemini for generation and a structured parser to enforce a predictable JSON result.

### Nodes Involved

- Basic LLM Chain
- Google Gemini Chat Model
- Structured Output Parser

### Node Details

#### 5. Basic LLM Chain

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Main AI orchestration node that sends article data to the language model and expects structured output.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Output parser enabled
  - Prompt instructs the model to:
    - Act as a Bitcoin news analyst and social media expert
    - Use:
      - `{{ $json.title }}`
      - `{{ $json.contentSnippet }}`
      - `{{ $json.link }}`
    - Return exactly 4 JSON fields:
      - `summary`
      - `comment`
      - `url`
      - `hashtags`
    - Use Japanese for summary/comment
    - Keep each summary/comment under 60 characters
    - Return only JSON, no markdown or explanation
- **Key expressions or variables used:**  
  - `{{ $json.title }}`
  - `{{ $json.contentSnippet }}`
  - `{{ $json.link }}`
- **Input and output connections:**  
  - Main input: `Get latest 3 articles`
  - AI language model input: `Google Gemini Chat Model`
  - AI output parser input: `Structured Output Parser`
  - Main output: `Discord`
- **Version-specific requirements:**  
  Uses `typeVersion` 1.9. Requires LangChain-compatible n8n installation and AI nodes enabled.
- **Edge cases or potential failure types:**  
  - Model may still return malformed JSON despite instruction
  - Parser may fail if field names differ or response is not valid JSON
  - RSS items may have empty `contentSnippet`
  - Prompt says “Bitcoin news” but feed may contain broader crypto stories
  - Character count requirements are advisory to the model, not technically enforced outside parsing
  - If Gemini rate limits or quota is exceeded, the node fails
- **Sub-workflow reference:**  
  None.

#### 6. Google Gemini Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Supplies the LLM used by the Basic LLM Chain.
- **Configuration choices:**  
  - Uses Google Gemini API credentials
  - No additional model options are explicitly set in the JSON
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to: `Basic LLM Chain`
- **Version-specific requirements:**  
  Uses `typeVersion` 1.
  Requires valid Google Gemini / PaLM-compatible API credentials in n8n.
- **Edge cases or potential failure types:**  
  - Invalid API key or credential mismatch
  - Quota exhaustion
  - Regional/model availability restrictions
  - Model behavior may change over time if defaults are updated
- **Sub-workflow reference:**  
  None.

#### 7. Structured Output Parser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a structured JSON schema for the LLM response.
- **Configuration choices:**  
  Uses a JSON schema example with these fields:
  - `summary`
  - `comment`
  - `url`
  - `hashtags`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI output parser connection to: `Basic LLM Chain`
- **Version-specific requirements:**  
  Uses `typeVersion` 1.3.
- **Edge cases or potential failure types:**  
  - If the LLM returns extra text, malformed JSON, or missing fields, parsing can fail
  - Example-based schema guidance is helpful but does not guarantee semantic correctness
  - URL could still be syntactically present but incorrect
- **Sub-workflow reference:**  
  None.

---

## Block 4 — Posting to Discord

### Overview

This block takes the structured AI response and publishes it to Discord using a webhook credential. It formats the content as a simple multiline message.

### Nodes Involved

- Discord

### Node Details

#### 8. Discord

- **Type and technical role:** `n8n-nodes-base.discord`  
  Sends a message to Discord using webhook authentication.
- **Configuration choices:**  
  The content is built from the LLM output:
  - `summary`
  - `comment`
  - `url`
  - `hashtags`
  
  Message template:
  ```text
  {{$json.output.summary}}
  {{$json.output.comment}}
  {{$json.output.url}}
  {{$json.output.hashtags}}
  ```
- **Key expressions or variables used:**  
  - `{{ $json.output.summary }}`
  - `{{ $json.output.comment }}`
  - `{{ $json.output.url }}`
  - `{{ $json.output.hashtags }}`
- **Input and output connections:**  
  - Input: `Basic LLM Chain`
  - Output: `Log to Google Sheets`
- **Version-specific requirements:**  
  Uses `typeVersion` 2.
- **Edge cases or potential failure types:**  
  - Invalid or revoked Discord webhook credential
  - Discord webhook permissions/channel issues
  - Message formatting may fail if any output field is null or undefined
  - If generated text exceeds Discord limits, the request may fail or truncate depending on endpoint behavior
- **Sub-workflow reference:**  
  None.

---

## Block 5 — Logging to Google Sheets

### Overview

This block appends each posted result to a Google Sheet for tracking and historical review. It records the article URL, summary, post timestamp, AI comment, and hashtags.

### Nodes Involved

- Log to Google Sheets

### Node Details

#### 9. Log to Google Sheets

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to a specified Google Sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet document ID is set directly in the node
  - Sheet name: `シート1`
  - Column mapping:
    - `URL` = `{{ $('Basic LLM Chain').item.json.output.url }}`
    - `要約` = `{{ $('Basic LLM Chain').item.json.output.summary }}`
    - `投稿日時` = `{{ $now.toISO() }}`
    - `AIコメント` = `{{ $('Basic LLM Chain').item.json.output.comment }}`
    - `ハッシュタグ` = `{{ $('Basic LLM Chain').item.json.output.hashtags }}`
- **Key expressions or variables used:**  
  - `{{ $('Basic LLM Chain').item.json.output.url }}`
  - `{{ $('Basic LLM Chain').item.json.output.summary }}`
  - `{{ $now.toISO() }}`
  - `{{ $('Basic LLM Chain').item.json.output.comment }}`
  - `{{ $('Basic LLM Chain').item.json.output.hashtags }}`
- **Input and output connections:**  
  - Input: `Discord`
  - Output: `Notify Slack`
- **Version-specific requirements:**  
  Uses `typeVersion` 4.5. Requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Wrong spreadsheet ID or inaccessible document
  - Sheet name `シート1` not present
  - OAuth token expired or missing permissions
  - Mapped columns may not match exact sheet headers
  - Configuration inconsistency: this node does not use `config_google_sheet_id` from the Set node
- **Sub-workflow reference:**  
  None.

---

## Block 6 — Slack Confirmation

### Overview

This block sends a completion notification to Slack after a row has been written to Google Sheets. It serves as a lightweight operational confirmation that the run progressed through posting and logging.

### Nodes Involved

- Notify Slack

### Node Details

#### 10. Notify Slack

- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a message to a Slack channel using OAuth2.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Destination channel by name: `#all-マナビトエージェント`
  - Text:
    ```text
    ✅ Bitcoin news posted to Discord!

    📰 *{{ $('Get latest 3 articles').item.json.title }}*
    ```
- **Key expressions or variables used:**  
  - `{{ $('Get latest 3 articles').item.json.title }}`
- **Input and output connections:**  
  - Input: `Log to Google Sheets`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion` 2.3.
- **Edge cases or potential failure types:**  
  - Invalid Slack OAuth2 credential
  - Channel name not found or app not invited
  - Expression may not resolve as expected if item linking becomes ambiguous across multiple items
  - This node runs once per processed article, so it may send multiple confirmations per workflow run
- **Sub-workflow reference:**  
  None.

---

## Block 7 — Documentation and Visual Notes

### Overview

These sticky notes do not execute logic but document the intended behavior, setup, and customization guidance directly on the canvas. They are important for human maintainability and some statements reveal intended design choices that are only partially implemented.

### Nodes Involved

- Sticky Note - Overview
- Sticky Note - Config
- Sticky Note - Ingestion
- Sticky Note - AI
- Sticky Note - Distribution

### Node Details

#### 11. Sticky Note - Overview

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for high-level workflow explanation and setup.
- **Configuration choices:**  
  Contains overview, setup steps, and customization guidance.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion` 1.
- **Edge cases or potential failure types:**  
  None at runtime.
- **Sub-workflow reference:**  
  None.

#### 12. Sticky Note - Config

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents that settings should live in one place.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion` 1.
- **Edge cases or potential failure types:**  
  Note content does not fully match implementation, because not all settings are truly centralized.
- **Sub-workflow reference:**  
  None.

#### 13. Sticky Note - Ingestion

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes RSS fetch and limiting behavior.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion` 1.
- **Edge cases or potential failure types:**  
  None at runtime.
- **Sub-workflow reference:**  
  None.

#### 14. Sticky Note - AI

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents Gemini summarization and structured parsing.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion` 1.
- **Edge cases or potential failure types:**  
  Mentions “Gemini 2.5 Flash,” but the model node itself does not visibly specify model options in the JSON.
- **Sub-workflow reference:**  
  None.

#### 15. Sticky Note - Distribution

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents posting, logging, and Slack confirmation.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion` 1.
- **Edge cases or potential failure types:**  
  Note claims run confirmation, but in practice Slack sends one message per processed item.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run every 6 hours | Schedule Trigger | Starts the workflow every 6 hours |  | Set config values | ## Overview<br>### How it works<br>This workflow runs every 6 hours and turns Bitcoin news into structured Japanese posts for Discord.<br>1. Schedule trigger fires every 6 hours<br>2. Set config values holds all settings in one place<br>3. RSS node fetches the latest 3 CoinDesk articles<br>4. Basic LLM Chain sends each article to Gemini 2.5 Flash<br>5. Structured Output Parser extracts 4 fields — summary, comment, URL, hashtags<br>6. Post is delivered to Discord via webhook<br>7. Each post is logged to Google Sheets<br>8. Slack notification confirms every run<br>### Setup steps<br>1. Open Set config values — fill in your Google Sheet ID, Discord Webhook URL, and hashtags<br>2. Add your Google Gemini API credential to the Gemini 2.5 Flash node (free tier works)<br>3. Connect your Google Sheets OAuth2 credential<br>4. Add your Slack credential and channel ID to the Notify Slack node<br>5. Activate the workflow<br>### Customization<br>- Swap the RSS URL in Set config values for a different crypto feed<br>- Edit the prompt in Basic LLM Chain to change language or post style<br>- Adjust the schedule interval in the trigger node |
| Set config values | Set | Stores reusable configuration values | Run every 6 hours | Fetch Bitcoin RSS feed | ## ⚙️ Config<br>All user settings live here.<br>Edit RSS URL, Sheet ID,<br>Discord URL, and hashtags. |
| Fetch Bitcoin RSS feed | RSS Feed Read | Reads CoinDesk RSS articles | Set config values | Get latest 3 articles | ## 📡 Data ingestion<br>Fetches RSS and limits output<br>to the 3 newest articles to<br>avoid flooding Discord. |
| Get latest 3 articles | Item Lists | Limits feed items to 3 | Fetch Bitcoin RSS feed | Basic LLM Chain | ## 📡 Data ingestion<br>Fetches RSS and limits output<br>to the 3 newest articles to<br>avoid flooding Discord. |
| Basic LLM Chain | LangChain LLM Chain | Generates structured Japanese summaries/comments | Get latest 3 articles; Google Gemini Chat Model; Structured Output Parser | Discord | ## 🤖 AI processing<br>Gemini 2.5 Flash summarizes<br>each article. Structured Output<br>Parser extracts 4 clean fields. |
| Google Gemini Chat Model | LangChain Google Gemini Chat Model | Provides the LLM backend |  | Basic LLM Chain | ## 🤖 AI processing<br>Gemini 2.5 Flash summarizes<br>each article. Structured Output<br>Parser extracts 4 clean fields. |
| Structured Output Parser | LangChain Structured Output Parser | Enforces 4-field JSON output |  | Basic LLM Chain | ## 🤖 AI processing<br>Gemini 2.5 Flash summarizes<br>each article. Structured Output<br>Parser extracts 4 clean fields. |
| Discord | Discord | Posts generated content to Discord | Basic LLM Chain | Log to Google Sheets | ## 📤 Distribution & logging<br>Posts to Discord, logs each<br>record to Sheets, and pings<br>Slack to confirm every run. |
| Log to Google Sheets | Google Sheets | Appends posted content to a spreadsheet | Discord | Notify Slack | ## 📤 Distribution & logging<br>Posts to Discord, logs each<br>record to Sheets, and pings<br>Slack to confirm every run. |
| Notify Slack | Slack | Sends completion notification to Slack | Log to Google Sheets |  | ## 📤 Distribution & logging<br>Posts to Discord, logs each<br>record to Sheets, and pings<br>Slack to confirm every run. |
| Sticky Note - Overview | Sticky Note | Canvas documentation and setup notes |  |  |  |
| Sticky Note - Config | Sticky Note | Canvas documentation for configuration area |  |  |  |
| Sticky Note - Ingestion | Sticky Note | Canvas documentation for ingestion area |  |  |  |
| Sticky Note - AI | Sticky Note | Canvas documentation for AI processing area |  |  |  |
| Sticky Note - Distribution | Sticky Note | Canvas documentation for distribution/logging area |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for recreating this workflow manually in n8n.

## 1. Create the Schedule Trigger

1. Add a **Schedule Trigger** node.
2. Name it **Run every 6 hours**.
3. Set the trigger rule to **Interval**.
4. Configure the interval to **every 6 hours**.
5. This node will be the workflow entry point.

## 2. Create the configuration Set node

1. Add a **Set** node.
2. Name it **Set config values**.
3. Connect **Run every 6 hours → Set config values**.
4. Add these fields in the Assignments section:
   - `config_rss_url` as String: `https://feeds.feedburner.com/CoinDesk`
   - `config_articles_to_fetch` as Number: `3`
   - `config_tweet_language` as String: `Japanese`
   - `config_hashtags` as String: `#Bitcoin #BTC #仮想通貨`
   - `config_google_sheet_id` as String: `YOUR_GOOGLE_SHEET_ID_HERE`
5. Leave other options at default.

## 3. Create the RSS reader

1. Add an **RSS Feed Read** node.
2. Name it **Fetch Bitcoin RSS feed**.
3. Connect **Set config values → Fetch Bitcoin RSS feed**.
4. Set the URL to:
   - `https://feeds.feedburner.com/CoinDesk`
5. In options, keep **Ignore SSL** disabled (`false`).

> Note: The original workflow hardcodes the URL here instead of referencing the Set node. If you want better maintainability, you can replace the URL with an expression referencing `config_rss_url`.

## 4. Limit the number of articles

1. Add an **Item Lists** node.
2. Name it **Get latest 3 articles**.
3. Connect **Fetch Bitcoin RSS feed → Get latest 3 articles**.
4. Set:
   - **Operation**: `Limit`
   - **Max Items**: `3`

> Note: The original workflow hardcodes the value `3` instead of using `config_articles_to_fetch`.

## 5. Create the LLM chain

1. Add a **Basic LLM Chain** node from the LangChain nodes.
2. Name it **Basic LLM Chain**.
3. Connect **Get latest 3 articles → Basic LLM Chain**.
4. Set **Prompt Type** to **Define below**.
5. Enable **Output Parser**.
6. Paste this prompt:

   ```text
   You are a Bitcoin news analyst and social media expert.

   Here is a Bitcoin news article:
   Title: {{ $json.title }}
   Summary: {{ $json.contentSnippet }}
   URL: {{ $json.link }}

   Return a JSON object with exactly these 4 fields:
   - "summary": one-line news summary in Japanese (max 60 chars)
   - "comment": your brief AI commentary in Japanese (max 60 chars)
   - "url": the article URL as-is
   - "hashtags": "#Bitcoin #BTC #仮想通貨"

   Respond ONLY with the JSON object. No explanation, no markdown, no code blocks.
   ```

7. Leave batching at default unless you want custom AI batching behavior.

## 6. Add the Google Gemini model node

1. Add a **Google Gemini Chat Model** node.
2. Name it **Google Gemini Chat Model**.
3. Connect it to **Basic LLM Chain** using the **AI Language Model** connection.
4. Configure Google Gemini credentials:
   - Create or select a **Google Gemini / PaLM API credential**
   - Paste your API key
5. If your n8n version allows explicit model selection, choose the intended Gemini model.  
   The sticky note refers to **Gemini 2.5 Flash**, but the supplied workflow JSON does not expose a model parameter. If available in your environment, select the corresponding fast Gemini model.

## 7. Add the structured parser

1. Add a **Structured Output Parser** node.
2. Name it **Structured Output Parser**.
3. Connect it to **Basic LLM Chain** using the **AI Output Parser** connection.
4. In the schema example field, enter:

   ```json
   {
     "summary": "SBFの過去の政治献金がNY候補攻撃に利用される件",
     "comment": "クリプトの負の遺産がAI政治にも影響、長期的な教訓に。",
     "url": "https://www.coindesk.com/policy/2026/03/20/sample",
     "hashtags": "#Bitcoin #BTC #仮想通貨"
   }
   ```

5. Save the node.

## 8. Create the Discord posting node

1. Add a **Discord** node.
2. Name it **Discord**.
3. Connect **Basic LLM Chain → Discord**.
4. Set authentication to **Webhook**.
5. Create or select a **Discord Webhook credential**.
   - In Discord, create a webhook for the target channel
   - Copy the webhook URL into the credential
6. Set the message content to:

   ```text
   {{ $json.output.summary }}
   {{ $json.output.comment }}
   {{ $json.output.url }}
   {{ $json.output.hashtags }}
   ```

7. Leave additional options empty unless you need embeds or username overrides.

## 9. Create the Google Sheets logging node

1. Add a **Google Sheets** node.
2. Name it **Log to Google Sheets**.
3. Connect **Discord → Log to Google Sheets**.
4. Set:
   - **Operation**: `Append`
5. Configure Google Sheets OAuth2 credentials:
   - Create or select a **Google Sheets OAuth2** credential
   - Grant access to the spreadsheet
6. Set the spreadsheet document ID.
   - In the original workflow, the node uses a fixed document ID directly.
   - If you want to match the original exactly, paste the intended document ID into the node.
   - If you want a cleaner design, reference `config_google_sheet_id`.
7. Set the sheet name to:
   - `シート1`
8. Use manual column mapping and map the following columns:
   - `URL` → `{{ $('Basic LLM Chain').item.json.output.url }}`
   - `要約` → `{{ $('Basic LLM Chain').item.json.output.summary }}`
   - `投稿日時` → `{{ $now.toISO() }}`
   - `AIコメント` → `{{ $('Basic LLM Chain').item.json.output.comment }}`
   - `ハッシュタグ` → `{{ $('Basic LLM Chain').item.json.output.hashtags }}`
9. Make sure the destination sheet has matching headers.

## 10. Create the Slack notification node

1. Add a **Slack** node.
2. Name it **Notify Slack**.
3. Connect **Log to Google Sheets → Notify Slack**.
4. Set authentication to **OAuth2**.
5. Create or select a **Slack OAuth2 credential**.
   - Ensure the Slack app has permission to post to the target channel
   - Invite the app to the destination channel if required
6. Choose **Select by channel**.
7. Set the channel to:
   - `#all-マナビトエージェント`
8. Set the message text to:

   ```text
   ✅ Bitcoin news posted to Discord!

   📰 *{{ $('Get latest 3 articles').item.json.title }}*
   ```

## 11. Optional: Add the sticky notes for documentation

To match the original canvas layout, add five **Sticky Note** nodes with the following content.

### Sticky Note - Overview

```text
## Overview

### How it works
This workflow runs every 6 hours and turns Bitcoin news into structured Japanese posts for Discord.

1. Schedule trigger fires every 6 hours
2. Set config values holds all settings in one place
3. RSS node fetches the latest 3 CoinDesk articles
4. Basic LLM Chain sends each article to Gemini 2.5 Flash
5. Structured Output Parser extracts 4 fields — summary, comment, URL, hashtags
6. Post is delivered to Discord via webhook
7. Each post is logged to Google Sheets
8. Slack notification confirms every run

### Setup steps
1. Open Set config values — fill in your Google Sheet ID, Discord Webhook URL, and hashtags
2. Add your Google Gemini API credential to the Gemini 2.5 Flash node (free tier works)
3. Connect your Google Sheets OAuth2 credential
4. Add your Slack credential and channel ID to the Notify Slack node
5. Activate the workflow

### Customization
- Swap the RSS URL in Set config values for a different crypto feed
- Edit the prompt in Basic LLM Chain to change language or post style
- Adjust the schedule interval in the trigger node
```

### Sticky Note - Config

```text
## ⚙️ Config

All user settings live here.
Edit RSS URL, Sheet ID,
Discord URL, and hashtags.
```

### Sticky Note - Ingestion

```text
## 📡 Data ingestion

Fetches RSS and limits output
to the 3 newest articles to
avoid flooding Discord.
```

### Sticky Note - AI

```text
## 🤖 AI processing

Gemini 2.5 Flash summarizes
each article. Structured Output
Parser extracts 4 clean fields.
```

### Sticky Note - Distribution

```text
## 📤 Distribution & logging

Posts to Discord, logs each
record to Sheets, and pings
Slack to confirm every run.
```

## 12. Validate the end-to-end flow

1. Run the workflow manually once.
2. Confirm the RSS feed returns items.
3. Confirm the LLM output is valid structured JSON.
4. Confirm Discord receives 3 messages.
5. Confirm Google Sheets receives 3 appended rows.
6. Confirm Slack receives 3 confirmation messages.

## 13. Activate the workflow

1. Save the workflow.
2. Turn **Active** on.
3. The schedule will now trigger every 6 hours.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow title and internal name are inconsistent. The user-facing title refers to posting to Discord, while the workflow name still says “post to X with Gemini AI.” | Internal naming / maintenance note |
| The Set node suggests centralized configuration, but several downstream nodes still use hardcoded values instead of config expressions. | Design consistency note |
| The RSS source is CoinDesk’s FeedBurner endpoint: `https://feeds.feedburner.com/CoinDesk` | RSS source |
| The Google Sheets target sheet name is `シート1`, so the spreadsheet should contain that exact tab unless adjusted. | Google Sheets setup |
| Slack confirmation is sent after each processed article, not once per overall workflow execution. | Operational behavior |
| The sticky notes mention a Discord URL in config, but the actual Discord node uses a webhook credential, not a config field. | Credential design note |
| The sticky notes mention Gemini 2.5 Flash, but the model selection is not explicitly visible in the exported JSON. | AI model note |