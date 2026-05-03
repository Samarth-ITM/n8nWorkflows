Daily Business Ideas from IdeaBrowser to Telegram

https://n8nworkflows.xyz/workflows/daily-business-ideas-from-ideabrowser-to-telegram-8922


# Daily Business Ideas from IdeaBrowser to Telegram

### 1. Workflow Overview

This workflow automates the daily scraping of a business idea from the website https://www.ideabrowser.com/idea-of-the-day and sends a formatted summary message to a specified Telegram chat. It is designed for entrepreneurs, business enthusiasts, or content curators who want to receive fresh business ideas every morning via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Block:** Initiates the workflow either manually or on a daily schedule at 9:00 AM.
- **1.2 Scraping Block:** Performs an HTTP request to fetch the HTML content from the IdeaBrowser daily idea page.
- **1.3 Content Extraction Block:** Parses the fetched HTML content and extracts key pieces of information such as title, description, pricing, market, and features.
- **1.4 Message Formatting Block:** Constructs a structured and human-readable message containing the extracted data.
- **1.5 Message Length Check Block:** Evaluates if the formatted message fits within Telegram’s message length limits.
- **1.6 Conditional Sending Block:** Sends the message to Telegram directly if within limits or truncates and then sends if too long.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** Initiates the workflow execution either manually for testing or automatically every day at 9 AM.
- **Nodes Involved:**  
  - Daily Schedule  
  - Manual Test Trigger

- **Node Details:**

  - **Daily Schedule**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow daily at 9:00 AM local time.  
    - Configuration: Set to trigger at hour 9 daily.  
    - Inputs: None  
    - Outputs: Connected to "Scrape Idea of the Day" node.  
    - Edge Cases: Timezone issues could affect trigger time; ensure n8n server timezone matches expectations.

  - **Manual Test Trigger**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow for testing or debugging.  
    - Configuration: Default, no parameters.  
    - Inputs: None  
    - Outputs: Connected to "Scrape Idea of the Day" node.  
    - Edge Cases: None significant; manual trigger only runs on user action.

#### 2.2 Scraping Block

- **Overview:** Retrieves the raw HTML content of the daily business idea page.
- **Nodes Involved:**  
  - Scrape Idea of the Day

- **Node Details:**

  - **Scrape Idea of the Day**  
    - Type: HTTP Request  
    - Role: Sends a GET request to the URL https://www.ideabrowser.com/idea-of-the-day to fetch the page content.  
    - Configuration:  
      - URL: https://www.ideabrowser.com/idea-of-the-day  
      - Method: GET (default)  
      - No authentication or special headers configured.  
    - Inputs: From either "Daily Schedule" or "Manual Test Trigger"  
    - Outputs: HTML content passed to "Extract Content" node.  
    - Edge Cases:  
      - Website downtime or request failures (timeouts, 4xx/5xx status codes).  
      - Changes in website structure or URL could break scraping downstream.

#### 2.3 Content Extraction Block

- **Overview:** Parses the HTML content and extracts specific fields relevant to the business idea.
- **Nodes Involved:**  
  - Extract Content

- **Node Details:**

  - **Extract Content**  
    - Type: HTML Extract  
    - Role: Uses CSS selectors to extract title, description, pricing, market, and features from the HTML.  
    - Configuration:  
      - Trim values enabled to clean whitespace.  
      - Extraction keys and their CSS selectors:  
        - title: `h1, .idea-title, .main-title`  
        - description: `.idea-description, .main-content, p`  
        - pricing: `.pricing, .offer, .price`  
        - market: `.market, .target, .analysis`  
        - features: `.features, .benefits, ul li`  
    - Inputs: Raw HTML from "Scrape Idea of the Day"  
    - Outputs: JSON with extracted fields to "Format Message"  
    - Edge Cases:  
      - If CSS selectors no longer match due to website redesign, extraction will fail or return empty fields.  
      - Multiple matches for selectors (especially for features list) may result in arrays or concatenated strings; verify expected format.

#### 2.4 Message Formatting Block

- **Overview:** Constructs a readable message summarizing the extracted business idea details, with date and truncation to avoid overly long content.
- **Nodes Involved:**  
  - Format Message

- **Node Details:**

  - **Format Message**  
    - Type: Code (JavaScript)  
    - Role: Creates the final Telegram message text with formatting, emojis, and substring truncation for each section.  
    - Configuration:  
      - Uses current date formatted as "Weekday, Month Day, Year" (e.g., Monday, June 12, 2024)  
      - Fields fallback to default phrases if empty or missing.  
      - Limits description to 500 chars, pricing, market, and features to 300 chars each.  
      - Appends a permanent link to the original page.  
    - Inputs: Extracted JSON fields from "Extract Content"  
    - Outputs: JSON containing `formattedMessage`, original URL, and scraped date to "Check Message Length"  
    - Edge Cases:  
      - If fields are unexpectedly null or non-string, substring operations could fail; the code assumes string or fallback values.  
      - Date formatting depends on environment locale support.

#### 2.5 Message Length Check Block

- **Overview:** Checks if the formatted message fits within Telegram’s maximum message length of 4096 characters.
- **Nodes Involved:**  
  - Check Message Length

- **Node Details:**

  - **Check Message Length**  
    - Type: If (Conditional)  
    - Role: Routes message either to direct sending or to truncation based on length condition.  
    - Configuration:  
      - Condition: `$json.formattedMessage.length <= 4096`  
      - True branch: Send message directly to Telegram  
      - False branch: Pass message to "Truncate Message"  
    - Inputs: From "Format Message"  
    - Outputs:  
      - True: "Send to Telegram"  
      - False: "Truncate Message"  
    - Edge Cases:  
      - Expression evaluation depends on presence and type of `formattedMessage`.  
      - Length exactly 4096 passes as valid.

#### 2.6 Conditional Sending Block

- **Overview:** Sends the Telegram message, truncating if necessary to fit length limits and appending a "Read more" link.
- **Nodes Involved:**  
  - Send to Telegram  
  - Truncate Message  
  - Send Truncated Message

- **Node Details:**

  - **Send to Telegram**  
    - Type: Telegram  
    - Role: Sends the formatted message to the specified Telegram chat.  
    - Configuration:  
      - Text: Uses expression to inject `$json.formattedMessage`  
      - Chat ID: Uses credential variable `telegramChatId`  
      - Append Attribution: Disabled (no 'via n8n' footer)  
    - Credentials: Telegram Bot with Bot Token provided in n8n credentials  
    - Inputs: True branch from "Check Message Length"  
    - Outputs: None (workflow ends)  
    - Edge Cases:  
      - Telegram API errors such as invalid token, wrong chat ID, or network issues.  
      - Message length must be within Telegram limits.

  - **Truncate Message**  
    - Type: Code (JavaScript)  
    - Role: Shortens the message to 4000 characters max and appends a "Read more" link to the original idea page.  
    - Configuration:  
      - Checks message length; if >4000, truncates to 3900 chars and appends `\n\n...\n\n🔗 Read more: https://www.ideabrowser.com/idea-of-the-day`  
      - Otherwise passes original message as is.  
    - Inputs: False branch from "Check Message Length"  
    - Outputs: Modified JSON with truncated `formattedMessage` to "Send Truncated Message"  
    - Edge Cases:  
      - Assumes input has a valid `formattedMessage`.  
      - Truncation at 3900 chars leaves room for appended link without exceeding Telegram max length.

  - **Send Truncated Message**  
    - Type: Telegram  
    - Role: Sends the truncated message to Telegram chat.  
    - Configuration: Same as "Send to Telegram" node.  
    - Credentials: Telegram Bot Token in credentials  
    - Inputs: From "Truncate Message"  
    - Outputs: None (workflow ends)  
    - Edge Cases: Same as "Send to Telegram".

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                      | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                         |
|---------------------|---------------------|------------------------------------|------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Daily Schedule      | Schedule Trigger    | Triggers workflow daily at 9 AM    | None                         | Scrape Idea of the Day       |                                                                                                   |
| Manual Test Trigger | Manual Trigger      | Allows manual execution for testing | None                         | Scrape Idea of the Day       |                                                                                                   |
| Scrape Idea of the Day | HTTP Request       | Fetches the daily idea HTML page   | Daily Schedule, Manual Test Trigger | Extract Content            |                                                                                                   |
| Extract Content     | HTML Extract        | Parses and extracts idea details   | Scrape Idea of the Day        | Format Message              |                                                                                                   |
| Format Message      | Code                | Builds formatted Telegram message  | Extract Content               | Check Message Length        |                                                                                                   |
| Check Message Length | If                  | Routes message based on length     | Format Message                | Send to Telegram, Truncate Message |                                                                                                   |
| Send to Telegram    | Telegram             | Sends message to Telegram chat     | Check Message Length (true)   | None                        |                                                                                                   |
| Truncate Message    | Code                 | Truncates message if too long      | Check Message Length (false)  | Send Truncated Message      |                                                                                                   |
| Send Truncated Message | Telegram            | Sends truncated message to Telegram | Truncate Message             | None                        |                                                                                                   |
| Workflow Context    | Sticky Note          | Provides high-level workflow summary | None                        | None                        | # Daily Idea Scraper to Telegram\n\nScrapes https://www.ideabrowser.com/idea-of-the-day daily at 9 AM and sends formatted content to Telegram. |
| Setup Instructions  | Sticky Note          | Details setup steps for credentials and testing | None                        | None                        | ## Setup Required:\n\n1. **Telegram Bot Token**: Create bot via @BotFather\n2. **Chat ID**: Get your chat ID (use @userinfobot)\n3. **Credentials**: Add Telegram credentials in n8n\n4. **Test**: Use manual trigger to test before scheduling\n\n**Daily Schedule**: Runs at 9:00 AM |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Daily Idea Scraper to Telegram".

2. **Add a Schedule Trigger node**  
   - Name: `Daily Schedule`  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 9:00 AM local time (set `triggerAtHour` to 9).  
   - No credentials required.

3. **Add a Manual Trigger node**  
   - Name: `Manual Test Trigger`  
   - Type: Manual Trigger  
   - No configuration needed.

4. **Add an HTTP Request node**  
   - Name: `Scrape Idea of the Day`  
   - Type: HTTP Request  
   - Configure:  
     - HTTP Method: GET  
     - URL: `https://www.ideabrowser.com/idea-of-the-day`  
     - No authentication or special headers needed.

5. **Connect both `Daily Schedule` and `Manual Test Trigger` nodes** to the `Scrape Idea of the Day` node outputs.

6. **Add an HTML Extract node**  
   - Name: `Extract Content`  
   - Type: HTML Extract  
   - Configure:  
     - Operation: Extract HTML Content  
     - Options: Enable "Trim Values" to remove whitespace.  
     - Extraction Values:  
       - `title` → CSS Selector: `h1, .idea-title, .main-title`  
       - `description` → CSS Selector: `.idea-description, .main-content, p`  
       - `pricing` → CSS Selector: `.pricing, .offer, .price`  
       - `market` → CSS Selector: `.market, .target, .analysis`  
       - `features` → CSS Selector: `.features, .benefits, ul li`  

7. **Connect `Scrape Idea of the Day` output to `Extract Content` input.**

8. **Add a Code node**  
   - Name: `Format Message`  
   - Type: Code (JavaScript)  
   - Paste the following code (adapted for n8n v2 code node):  
     ```javascript
     const today = new Date().toLocaleDateString('en-US', {
       weekday: 'long',
       year: 'numeric',
       month: 'long',
       day: 'numeric'
     });

     const title = $json.title || 'Business Idea';
     const description = $json.description || 'No description available';
     const pricing = $json.pricing || 'Pricing not specified';
     const market = $json.market || 'Market analysis not available';
     const features = $json.features || 'Features not listed';

     const message = `🚀 Daily Business Idea - ${today}\n\n📋 ${title}\n\n💡 Description:\n${description.substring(0, 500)}${description.length > 500 ? '...' : ''}\n\n💰 Revenue Model:\n${pricing.substring(0, 300)}${pricing.length > 300 ? '...' : ''}\n\n🎯 Target Market:\n${market.substring(0, 300)}${market.length > 300 ? '...' : ''}\n\n⚡ Key Features:\n${features.substring(0, 300)}${features.length > 300 ? '...' : ''}\n\n🔗 View Details: https://www.ideabrowser.com/idea-of-the-day`;

     return {
       json: {
         formattedMessage: message,
         originalUrl: 'https://www.ideabrowser.com/idea-of-the-day',
         scrapedDate: new Date().toISOString()
       }
     };
     ```
   
9. **Connect `Extract Content` output to `Format Message` input.**

10. **Add an If node**  
    - Name: `Check Message Length`  
    - Type: If  
    - Configure condition:  
      - Condition type: Number  
      - Left Value: Expression `{{ $json.formattedMessage.length }}`  
      - Operation: Less than or equal (<=)  
      - Right Value: `4096`  

11. **Connect `Format Message` output to `Check Message Length` input.**

12. **Add a Telegram node**  
    - Name: `Send to Telegram`  
    - Type: Telegram  
    - Configure:  
      - Text: Expression `{{ $json.formattedMessage }}`  
      - Chat ID: Use credential variable `{{ $credentials.telegramChatId }}`  
      - Additional Fields: Disable append attribution  
      - Set credentials by creating a Telegram API credential in n8n with your Bot Token (from @BotFather).  
    - Connect the **true** output of `Check Message Length` to this node.

13. **Add another Code node**  
    - Name: `Truncate Message`  
    - Type: Code (JavaScript)  
    - Paste this code:  
      ```javascript
      const originalMessage = $json.formattedMessage;
      const maxLength = 4000;
      
      let truncatedMessage = originalMessage;
      if (originalMessage.length > maxLength) {
        truncatedMessage = originalMessage.substring(0, maxLength - 100) + '\n\n...\n\n🔗 Read more: https://www.ideabrowser.com/idea-of-the-day';
      }
      
      return {
        json: {
          ...input.json,
          formattedMessage: truncatedMessage
        }
      };
      ```
    - Connect the **false** output of `Check Message Length` to this node.

14. **Add another Telegram node**  
    - Name: `Send Truncated Message`  
    - Type: Telegram  
    - Configure identically to `Send to Telegram` node (same chat ID and credentials).  
    - Connect the output of `Truncate Message` node to this node.

15. **Add two Sticky Note nodes for documentation:**  
    - One near the start, titled `Workflow Context` with content:  
      ```
      # Daily Idea Scraper to Telegram

      Scrapes https://www.ideabrowser.com/idea-of-the-day daily at 9 AM and sends formatted content to Telegram.
      ```
    - One near the end, titled `Setup Instructions` with content:  
      ```
      ## Setup Required:

      1. **Telegram Bot Token**: Create bot via @BotFather  
      2. **Chat ID**: Get your chat ID (use @userinfobot)  
      3. **Credentials**: Add Telegram credentials in n8n  
      4. **Test**: Use manual trigger to test before scheduling

      **Daily Schedule**: Runs at 9:00 AM
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                       |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Telegram bot token must be generated using the official @BotFather Telegram bot.                                                | Telegram Bot Creation                                |
| To find the chat ID for your Telegram conversation, use the @userinfobot Telegram bot.                                         | Telegram Chat ID Retrieval                           |
| The workflow respects Telegram’s maximum message length of 4096 characters and truncates messages if necessary.                | Telegram API Documentation: https://core.telegram.org/bots/api#sendmessage |
| The date formatting uses the 'en-US' locale; adjust if your timezone or language preferences differ.                           | JavaScript Date.toLocaleDateString docs              |
| Scheduled trigger time depends on the timezone of your n8n instance/server.                                                    | n8n Scheduling Documentation                          |

---

This documentation fully describes the “Daily Idea Scraper to Telegram” workflow, enabling expert users or AI agents to understand, reproduce, and maintain it effectively.