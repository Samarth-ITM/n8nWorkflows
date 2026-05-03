Automated Crypto News Digest to Telegram with RSS Feeds and GPT-4

https://n8nworkflows.xyz/workflows/automated-crypto-news-digest-to-telegram-with-rss-feeds-and-gpt-4-7930


# Automated Crypto News Digest to Telegram with RSS Feeds and GPT-4

### 1. Workflow Overview

This workflow automates the process of collecting, analyzing, summarizing, and distributing cryptocurrency news via Telegram. It targets users who want a curated digest of the most relevant crypto news in Russian, delivered regularly to a Telegram channel or group. The workflow comprises the following logical blocks:

- **1.1 Scheduling & RSS Collection:** Periodically trigger news fetches from multiple major crypto news RSS feeds.
- **1.2 Data Aggregation & Filtering:** Merge incoming feeds, filter out old or duplicate news, and prepare a clean dataset.
- **1.3 AI Analysis & Summarization:** Use OpenAI GPT-4 models to analyze the news, select the top 10 most important events, and create a styled digest in Russian with HTML formatting.
- **1.4 Message Preparation & Delivery:** Split the digest into Telegram-compatible message chunks and post them automatically to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & RSS Collection

**Overview:**  
This block triggers the workflow every 3 hours and fetches fresh news articles from five different cryptocurrency news sources via their RSS feeds.

**Nodes Involved:**  
- Scheduler  
- Nulltx  
- Coindesk  
- Cointelegraph  
- Decrypt  
- Cryptobriefing  

**Node Details:**

- **Scheduler**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow every 3 hours (configurable interval).  
  - Configuration: Trigger rule set to every 3 hours.  
  - Input: None (trigger node).  
  - Output: Triggers downstream RSS nodes simultaneously.  
  - Failure Modes: Misconfiguration of interval, system time errors.  

- **Nulltx, Coindesk, Cointelegraph, Decrypt, Cryptobriefing**  
  - Type: RSS Feed Read  
  - Role: Fetch latest RSS feed items from each news source.  
  - Configuration: Each node configured with respective RSS URL:  
    - Nulltx: https://nulltx.com/feed/  
    - Coindesk: https://www.coindesk.com/arc/outboundfeeds/rss/  
    - Cointelegraph: https://cointelegraph.com/rss  
    - Decrypt: https://decrypt.co/feed  
    - Cryptobriefing: https://cryptobriefing.com/feed/  
  - Input: Trigger from Scheduler.  
  - Output: RSS feed items emitted downstream.  
  - Failure Modes: Network errors, invalid feed URLs, feed format changes, rate limits.

---

#### 2.2 Data Aggregation & Filtering

**Overview:**  
Merges RSS feeds into a unified stream, removes duplicates, filters out news older than 24 hours, cleans HTML content, and formats data for further processing.

**Nodes Involved:**  
- Merge  
- News Filter & Sorter  
- News Formatter  

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Combines the 5 RSS feed inputs into a single stream for unified processing.  
  - Configuration: Set to merge 5 inputs.  
  - Input: RSS feeds from Nulltx, Coindesk, Cointelegraph, Decrypt, Cryptobriefing.  
  - Output: Single combined list of news items.  
  - Failure Modes: Missing inputs, inconsistent item formats.  

- **News Filter & Sorter**  
  - Type: Code (JavaScript)  
  - Role:  
    - Filters out news items older than 24 hours.  
    - Deduplicates based on article URLs.  
    - Cleans RSS content from HTML tags and entities.  
    - Preserves key data fields (title, link, snippet, date, source).  
  - Key Expressions:  
    - Date filtering using JavaScript date comparisons.  
    - Uses Sets to track and skip duplicates.  
    - Regex for HTML stripping.  
  - Input: Merged news items.  
  - Output: Filtered and cleaned unique news array.  
  - Failure Modes: Parsing errors if feed fields missing or malformed, timezone inconsistencies.  

- **News Formatter**  
  - Type: Code (JavaScript)  
  - Role: Converts the array of filtered news items into a single object with a "data" key containing the array, preparing input for AI nodes.  
  - Input: Filtered news items array.  
  - Output: One JSON item with `json.data` as an array of news.  
  - Failure Modes: Empty input arrays, malformed data structures.

---

#### 2.3 AI Analysis & Summarization

**Overview:**  
This block uses two AI agents sequentially: one to select the 10 most important news events with the best representative articles, and another to translate and format the digest in Russian with strict HTML styling for Telegram.

**Nodes Involved:**  
- Crypto Analyst (GPT-4o-mini)  
- SMM Editor (GPT-4.1-mini)  

**Node Details:**

- **Crypto Analyst**  
  - Type: Langchain Agent (OpenAI GPT)  
  - Role: Analyze all news items and select the top 10 most significant events, choosing the best article per event.  
  - Configuration: Model set to "gpt-4o-mini" with OpenAI credentials.  
  - Key Prompt Elements:  
    - Prioritizes finance, technology, regulation, scientific breakthroughs.  
    - Output must be a JSON array of 10 news objects only.  
  - Input: Formatted news array from "News Formatter".  
  - Output: JSON array of 10 selected news items.  
  - Failure Modes: API errors, prompt misformatting, exceeding token limits.  
  - Credential: OpenAI API.  

- **SMM Editor**  
  - Type: Langchain Agent (OpenAI GPT)  
  - Role: Translate selected news into Russian, format headlines and summaries with strict HTML tags compatible with Telegram, and structure as a numbered digest with hashtags.  
  - Configuration: Model "gpt-4.1-mini" with same OpenAI credentials.  
  - Key Prompt Rules:  
    - HTML only, no Markdown.  
    - Headlines bold and underlined.  
    - Dates formatted as DD month YYYY, HH:MM in <code> tags.  
    - Links as HTML anchors.  
    - Numbered list with double line breaks separating news.  
    - Hashtags including #CryptoDigest for branding.  
  - Input: JSON array output from Crypto Analyst.  
  - Output: Long formatted text string.  
  - Failure Modes: Formatting errors, translation inaccuracies, API timeouts.  
  - Credential: OpenAI API.

---

#### 2.4 Message Preparation & Delivery

**Overview:**  
Prepares the AI-generated long digest text for Telegram by splitting it into message-sized chunks (max 4096 characters), then posts them sequentially to a Telegram chat or channel with HTML parsing.

**Nodes Involved:**  
- News Preparer  
- Post to Group  

**Node Details:**

- **News Preparer**  
  - Type: Code (JavaScript)  
  - Role:  
    - Splits the long digest text into chunks ≤4096 characters (Telegram message limit).  
    - Attempts to split at double line breaks for clean message boundaries.  
    - Falls back to single line break or hard cut if necessary.  
    - Outputs array of messages, each with `text` field for Telegram.  
  - Input: Text output from SMM Editor.  
  - Output: Array of text chunks as separate items for downstream processing.  
  - Failure Modes: Unexpected text format, empty input.  

- **Post to Group**  
  - Type: Telegram node  
  - Role: Sends the prepared message chunks to the configured Telegram chat/channel using HTML parse mode.  
  - Configuration:  
    - `chatId` must be set to the target Telegram group or channel ID (e.g., @channelname or numeric ID).  
    - `parse_mode` set to HTML to interpret formatting.  
    - Attribution disabled to keep messages clean.  
  - Input: Message chunks from News Preparer.  
  - Output: Sent messages on Telegram.  
  - Failure Modes: Invalid chat ID, Telegram API errors, network issues, message size errors.

---

### 3. Summary Table

| Node Name          | Node Type                       | Functional Role                         | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                                             |
|--------------------|--------------------------------|---------------------------------------|----------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Scheduler          | Schedule Trigger               | Triggers workflow every 3 hours       | None                             | Coindesk, Cointelegraph, Decrypt, Cryptobriefing, Nulltx |                                                                                                                                         |
| Nulltx             | RSS Feed Read                 | Fetches Nulltx RSS feed                | Scheduler                       | Merge                          |                                                                                                                                         |
| Coindesk           | RSS Feed Read                 | Fetches Coindesk RSS feed              | Scheduler                       | Merge                          |                                                                                                                                         |
| Cointelegraph      | RSS Feed Read                 | Fetches Cointelegraph RSS feed         | Scheduler                       | Merge                          |                                                                                                                                         |
| Decrypt            | RSS Feed Read                 | Fetches Decrypt RSS feed               | Scheduler                       | Merge                          |                                                                                                                                         |
| Cryptobriefing     | RSS Feed Read                 | Fetches Cryptobriefing RSS feed        | Scheduler                       | Merge                          |                                                                                                                                         |
| Merge              | Merge                         | Combines all RSS feeds                 | Nulltx, Coindesk, Cointelegraph, Decrypt, Cryptobriefing | News Filter & Sorter           |                                                                                                                                         |
| News Filter & Sorter| Code (JavaScript)             | Filters, deduplicates, cleans news    | Merge                          | News Formatter                 |                                                                                                                                         |
| News Formatter     | Code (JavaScript)             | Formats news array for AI input        | News Filter & Sorter            | Crypto Analyst                 |                                                                                                                                         |
| Crypto Analyst     | Langchain Agent (OpenAI GPT)  | Selects top 10 important news events   | News Formatter                 | SMM Editor                    |                                                                                                                                         |
| SMM Editor         | Langchain Agent (OpenAI GPT)  | Translates and formats digest in Russian with HTML | Crypto Analyst                | News Preparer                 |                                                                                                                                         |
| News Preparer      | Code (JavaScript)             | Splits long text into Telegram message chunks | SMM Editor                    | Post to Group                 |                                                                                                                                         |
| Post to Group      | Telegram                      | Posts digest messages to Telegram chat | News Preparer                 | None                         | In the Telegram node “Post to Group”, set your `chatId` (e.g., @your_channel or numeric ID). Adjust scheduler interval if needed. Run once manually to verify, then activate. |

| Sticky Note | Covers Nodes                                                                                  |
|-------------|----------------------------------------------------------------------------------------------|
| Description of workflow, purpose, setup instructions, usage notes | Entire workflow overview and usage (Sticky Note near nodes on UI) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add "Scheduler" node:**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 3 hours.

3. **Add RSS Feed nodes for each source:**

   - Nulltx: RSS Feed Read node with URL `https://nulltx.com/feed/`  
   - Coindesk: RSS Feed Read node with URL `https://www.coindesk.com/arc/outboundfeeds/rss/`  
   - Cointelegraph: RSS Feed Read node with URL `https://cointelegraph.com/rss`  
   - Decrypt: RSS Feed Read node with URL `https://decrypt.co/feed`  
   - Cryptobriefing: RSS Feed Read node with URL `https://cryptobriefing.com/feed/`

4. **Connect Scheduler node outputs to all five RSS Feed nodes (parallel).**

5. **Add a "Merge" node:**  
   - Type: Merge  
   - Set "Number of Inputs" to 5.  
   - Connect all five RSS Feed nodes as inputs to Merge.

6. **Add a "Code" node named "News Filter & Sorter":**  
   - Paste the provided JavaScript code that filters news older than 24 hours, removes duplicates by URL, and cleans HTML from content.  
   - Connect Merge output to this node.

7. **Add another "Code" node named "News Formatter":**  
   - Paste the JavaScript code that converts the filtered news array into a single JSON object with a `data` property (array of news).  
   - Connect "News Filter & Sorter" output to this node.

8. **Add Langchain Agent node "Crypto Analyst":**  
   - Set model to `gpt-4o-mini`.  
   - Add OpenAI credentials (API key).  
   - Use the prompt to analyze and select the 10 most important crypto news events from the input data.  
   - Connect "News Formatter" output to this node.

9. **Add Langchain Agent node "SMM Editor":**  
   - Set model to `gpt-4.1-mini`.  
   - Use same OpenAI credentials.  
   - Use the detailed prompt to translate and format news into Russian with specific HTML formatting for Telegram.  
   - Connect "Crypto Analyst" output to this node.

10. **Add a "Code" node named "News Preparer":**  
    - Paste the JavaScript code that splits the long digest text into ≤4096 character chunks, ideally splitting at double line breaks.  
    - Connect "SMM Editor" output to this node.

11. **Add "Telegram" node named "Post to Group":**  
    - Configure credentials for Telegram Bot API OAuth2 or token access.  
    - Set `chatId` to your Telegram channel or group (e.g., `@your_channel` or numeric ID).  
    - Set message text to `={{ $json.text }}`.  
    - Set `parse_mode` to HTML and disable attribution.  
    - Connect "News Preparer" output to this node.

12. **Connect nodes in the following order:**  
    Scheduler → (all RSS nodes) → Merge → News Filter & Sorter → News Formatter → Crypto Analyst → SMM Editor → News Preparer → Post to Group.

13. **Test the workflow manually:**  
    - Run once to verify proper data flow and output.  
    - Check Telegram channel for digest message delivery with correct formatting.  

14. **Activate the workflow:**  
    - Once verified, activate to run automatically every 3 hours.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                     |
|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| The digest strictly uses HTML formatting compatible with Telegram; Markdown or other formatting is disallowed by design. | Critical for Telegram message compatibility and proper display.  |
| Setup requires valid OpenAI API credentials and Telegram Bot credentials with proper permissions for the target chat.    | Credential configuration instructions included in node details.  |
| The workflow is designed for Russian-language output but can be adapted by modifying AI prompts.                         | Prompt customization possible for other languages or styles.     |
| The scheduler interval is set to every 3 hours by default but can be adjusted as needed.                                  | Change in Scheduler node parameters.                              |
| The workflow includes a sticky note with detailed description, purpose, and setup steps for user reference.               | Visible in the workflow UI for onboarding and documentation.      |

---

**Disclaimer:**  
This workflow was created entirely using n8n automation and respects all applicable content policies. It processes only publicly available and legal data sources.