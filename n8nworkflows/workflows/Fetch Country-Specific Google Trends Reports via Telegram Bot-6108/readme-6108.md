Fetch Country-Specific Google Trends Reports via Telegram Bot

https://n8nworkflows.xyz/workflows/fetch-country-specific-google-trends-reports-via-telegram-bot-6108


# Fetch Country-Specific Google Trends Reports via Telegram Bot

---

### 1. Workflow Overview

This workflow enables users to fetch country-specific Google Trends reports via a Telegram bot. A user sends a country code (e.g., "EG", "US", "SA") as a message through Telegram; the workflow queries Google Trends RSS feeds for that country, processes the top trending topics, formats a detailed report in Arabic with rich Markdown, and sends it back to the user on Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Input Reception:** Receives user messages containing country codes through a Telegram Trigger node.
- **1.2 Fetching Google Trends Data:** Uses an HTTP Request node to fetch Google Trends RSS feed for the specified country.
- **1.3 Parsing and Converting Data:** Converts the XML RSS feed to JSON using the XML node.
- **1.4 Report Preparation:** Processes and formats the top 5 Google Trends entries into a Markdown report with trend titles, search traffic, and related news.
- **1.5 Sending the Report:** Sends the formatted trends report back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

- **Overview:**  
Receives incoming messages from Telegram users, filtering by configured chat IDs, to extract the country code input.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: `telegramTrigger`  
    - Role: Entry point for messages sent to the Telegram bot.  
    - Configuration: Listens for "message" updates only; filters messages to those from specified chat IDs (via environment variable `$env.chat_id`).  
    - Key expressions: Uses `$env.chat_id` to restrict access.  
    - Inputs: None (trigger node).  
    - Outputs: Emits incoming message JSON object, including the text containing the country code and chat details.  
    - Edge cases: Invalid or missing chat IDs in environment variable, unsupported or malformed messages, bot not authorized in chat.  
    - Credentials: Requires Telegram API credentials with bot token and webhook configured.  
    - Sticky Note: "## Send country code\nEG | US | SA | ..."

#### 1.2 Fetching Google Trends Data

- **Overview:**  
Constructs and sends a GET request to Google Trends RSS feed, passing the country code from Telegram as the `geo` query parameter.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **HTTP Request**  
    - Type: `httpRequest`  
    - Role: Performs HTTP GET to `https://trends.google.com/trending/rss` with dynamic query parameter `geo`.  
    - Configuration:  
      - URL fixed as trends.google.com RSS feed.  
      - Sends query parameter `geo` dynamically set from the Telegram message text: `={{ $json.message.text }}`.  
      - No additional headers or authentication required.  
    - Inputs: Receives Telegram message JSON.  
    - Outputs: Raw XML response from Google Trends RSS feed.  
    - Edge cases: Invalid country codes leading to empty or error responses, network timeouts, rate limiting, malformed responses.  
    - Sticky Note: "## Fetch trends \nGets trends for provided country code"

#### 1.3 Parsing and Converting Data

- **Overview:**  
Converts the XML RSS feed response into JSON format for easier manipulation downstream.

- **Nodes Involved:**  
  - XML

- **Node Details:**  
  - **XML**  
    - Type: `xml`  
    - Role: Parses XML string from HTTP response into JSON.  
    - Configuration: Default XML parsing options (no special options set).  
    - Inputs: Receives HTTP Request node output (raw XML).  
    - Outputs: JSON object representing the RSS feed structure.  
    - Edge cases: Malformed XML causing parse failures, empty responses, unexpected data structures.  
    - Sticky Note: "## Convert to json"

#### 1.4 Report Preparation

- **Overview:**  
Processes the JSON data to extract the top 5 trending topics, formats them into a Markdown report in Arabic including search traffic and top 2 related news articles per trend.

- **Nodes Involved:**  
  - Code (JavaScript)

- **Node Details:**  
  - **Code**  
    - Type: `code` (JavaScript)  
    - Role: Data transformation and report formatting.  
    - Configuration:  
      - Extracts trend items from JSON path `$json.rss.channel.item`.  
      - Handles empty trends by returning a friendly message.  
      - Formats the current date in Arabic locale (`ar-EG`).  
      - Builds a header with trend report title including the country code from Telegram input.  
      - Processes the first 5 trend items, extracting title, approximate traffic, and up to 2 related news items per trend, formatting all with Markdown syntax optimized for RTL display.  
      - Combines all formatted blocks with separators (`---`).  
      - Outputs JSON with keys `text` (final report) and `parse_mode: 'Markdown'` for Telegram formatting.  
    - Key expressions: Accesses `$('Telegram Trigger').first().json.message.text` for country code; uses array slicing, mapping, and string templates.  
    - Inputs: JSON from XML parser.  
    - Outputs: JSON with formatted message text and parse mode.  
    - Edge cases: Missing or empty trend items; missing traffic or news entries; localization or encoding issues; expression errors if input structure changes.  
    - Sticky Note: "## Prepare report"

#### 1.5 Sending the Report

- **Overview:**  
Sends the formatted trends report back to the appropriate Telegram chat.

- **Nodes Involved:**  
  - Send a text message (Telegram node)

- **Node Details:**  
  - **Send a text message**  
    - Type: `telegram`  
    - Role: Sends text message to Telegram user/chat.  
    - Configuration:  
      - Text parameter set dynamically from the Code node output: `={{ $json.text }}`.  
      - Chat ID dynamically set from the original Telegram Trigger node: `={{ $('Telegram Trigger').item.json.message.chat.id }}`.  
      - Additional fields: `parse_mode` is set to HTML (note: the code output uses Markdown; slight mismatch but Telegram supports both), and `appendAttribution` is disabled.  
    - Inputs: JSON with text content from Code node.  
    - Outputs: Telegram API response (not used further).  
    - Credentials: Telegram API credentials.  
    - Edge cases: Chat ID invalid or missing; Telegram API rate limits; message formatting errors; network issues.  
    - Sticky Note: "## Send report"

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                 | Input Node(s)         | Output Node(s)           | Sticky Note                             |
|---------------------|---------------------|--------------------------------|-----------------------|--------------------------|---------------------------------------|
| Telegram Trigger     | telegramTrigger     | Receive user country code input | None                  | HTTP Request             | ## Send country code<br>EG \| US \| SA \| ... |
| HTTP Request        | httpRequest         | Fetch Google Trends RSS feed    | Telegram Trigger      | XML                      | ## Fetch trends <br>Gets trends for provided country code |
| XML                 | xml                 | Parse XML to JSON               | HTTP Request          | Code                     | ## Convert to json                    |
| Code                | code                | Format trends report in Markdown | XML                   | Send a text message      | ## Prepare report                    |
| Send a text message  | telegram            | Send formatted trends report    | Code                  | None                     | ## Send report                      |
| Sticky Note          | stickyNote          | Informational                  | None                  | None                     | ## Fetch trends <br>Gets trends for provided country code |
| Sticky Note1         | stickyNote          | Informational                  | None                  | None                     | ## Convert to json                   |
| Sticky Note2         | stickyNote          | Informational                  | None                  | None                     | ## Prepare report                   |
| Sticky Note3         | stickyNote          | Informational                  | None                  | None                     | ## Send report                     |
| Sticky Note4         | stickyNote          | Informational                  | None                  | None                     | ## Send country code <br>EG \| US \| SA \| ... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure to listen to "message" updates only.  
   - Set `chatIds` filter to `={{ $env.chat_id }}` (requires setting environment variable `chat_id` with allowed chat IDs).  
   - Connect Telegram API credentials (bot token).  
   - Position it as the workflow start.

2. **Create HTTP Request node:**  
   - Type: HTTP Request  
   - Set URL to `https://trends.google.com/trending/rss`.  
   - Enable sending query parameters.  
   - Add query parameter:  
     - Name: `geo`  
     - Value: `={{ $json.message.text }}` (extracts country code from Telegram message).  
   - Connect input from Telegram Trigger node.

3. **Create XML node:**  
   - Type: XML  
   - Use default options.  
   - Connect input from HTTP Request node to parse RSS XML response.

4. **Create Code node:**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the following logic (adjusted as per below):  
     ```javascript
     const trends = $json.rss?.channel?.item || [];
     if (trends.length === 0) {
       return [{ json: { text: "Could not fetch Google Trends data at this time." } }];
     }
     const today = new Date().toLocaleDateString('ar-EG', { day: 'numeric', month: 'long', year: 'numeric' });
     const countryCode = $('Telegram Trigger').item.json.message.text;
     const header = `🔥 *Trends report for ${countryCode}*\n_${today}_\n\n`;
     const trendBlocks = trends.slice(0,5).map((trend, index) => {
       const rank = index + 1;
       const title = trend.title;
       const traffic = trend['ht:approx_traffic'] || 'Not available';
       const newsItems = (trend['ht:news_item'] || []).slice(0,2).map(news => {
         const newsTitle = news['ht:news_item_title'];
         const newsUrl = news['ht:news_item_url'];
         const newsSource = news['ht:news_item_source'];
         return `• [${newsTitle}](${newsUrl}) - _${newsSource}_`;
       }).join('\n');
       return `*${title}* (${rank})\n📈 *Searches:* ${traffic}\n🗞️ *Related:*\n${newsItems}`;
     }).join('\n\n---\n\n');
     const finalReport = header + trendBlocks;
     return [{ json: { text: finalReport, parse_mode: 'Markdown' } }];
     ```
   - Connect input from XML node.

5. **Create Telegram node to send message:**  
   - Type: Telegram  
   - Set "Text" parameter to `={{ $json.text }}` (from Code node output).  
   - Set "Chat ID" parameter to `={{ $('Telegram Trigger').item.json.message.chat.id }}` to reply to the original user.  
   - In Additional Fields, set `parse_mode` to `"HTML"` (note: the message is Markdown, Telegram supports both but consider switching to MarkdownV2 if issues arise).  
   - Disable append attribution.  
   - Connect input from Code node.  
   - Attach Telegram API credentials.

6. **Arrange nodes sequentially:**  
   Telegram Trigger → HTTP Request → XML → Code → Send a text message.

7. **Set environment variable:**  
   - Add environment variable `chat_id` with the list of allowed Telegram chat IDs (comma-separated or as needed).

8. **Test:**  
   - Send a country code like "EG" to the Telegram bot.  
   - Confirm the bot replies with the formatted Google Trends report.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow uses Google Trends RSS feeds with dynamic `geo` query parameter to localize trends by country code. | Google Trends RSS documentation (unofficial)    |
| The formatted report uses Arabic localization for date and RTL-friendly Markdown formatting.                    | Date localization uses `ar-EG` locale             |
| Telegram bot requires proper webhook setup and valid API credentials (bot token).                               | Telegram Bot API docs: https://core.telegram.org/bots/api |
| The code node outputs text in Markdown format, but Telegram node sends with `parse_mode` set to HTML; consider aligning formats if rendering issues occur. | Telegram message formatting notes                 |
| Environment variable `chat_id` restricts usage to authorized Telegram chats for security.                        | n8n environment variable management               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.