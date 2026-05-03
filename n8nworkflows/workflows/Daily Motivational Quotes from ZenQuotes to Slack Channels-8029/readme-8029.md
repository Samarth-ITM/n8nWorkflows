Daily Motivational Quotes from ZenQuotes to Slack Channels

https://n8nworkflows.xyz/workflows/daily-motivational-quotes-from-zenquotes-to-slack-channels-8029


# Daily Motivational Quotes from ZenQuotes to Slack Channels

### 1. Workflow Overview

This workflow automates the daily posting of motivational quotes from the ZenQuotes.io API to a Slack channel. Targeted at teams or communities seeking daily inspiration, it fetches a random quote each day at 8 AM (configured for America/New_York timezone), formats the quote suitably for Slack, and posts it to a specified Slack channel using a Slack app with OAuth2 authentication.

The workflow logic is divided into these blocks:

- **1.1 Scheduling Trigger:** Initiates the workflow execution daily at 8 AM.
- **1.2 Quote Retrieval:** Calls the ZenQuotes.io API to fetch a random motivational quote.
- **1.3 Quote Formatting:** Processes and formats the API response into a Slack message.
- **1.4 Slack Posting:** Sends the formatted motivational quote to the designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling Trigger

- **Overview:**  
  This block schedules the workflow to run automatically once per day at 8 AM Eastern Time.

- **Nodes Involved:**  
  - Daily 8AM Trigger

- **Node Details:**

  **Daily 8AM Trigger**  
  - Type: Cron Trigger  
  - Configuration: Default cron settings (assumed 8 AM daily based on workflow description and timezone settings)  
  - Key Expressions/Variables: None  
  - Input: None (trigger node)  
  - Output: Triggers downstream "Fetch Random Quote" node  
  - Version Requirements: None specific; standard cron node  
  - Potential Failures: None typical; misconfiguration of timezone can cause timing issues  
  - Notes: The workflow timezone is set to America/New_York, ensuring timing is consistent with Eastern Time.

#### 1.2 Quote Retrieval

- **Overview:**  
  Fetches a random motivational quote by making an HTTP GET request to the ZenQuotes.io API.

- **Nodes Involved:**  
  - Fetch Random Quote

- **Node Details:**

  **Fetch Random Quote**  
  - Type: HTTP Request  
  - Configuration:  
    - Method: GET  
    - URL: https://zenquotes.io/api/random  
    - No authentication or API key required (free public API)  
  - Input: Trigger from "Daily 8AM Trigger"  
  - Output: JSON response containing an array with one quote object  
  - Version Requirements: Node version 4.1 or higher (for HTTP Request)  
  - Potential Failures:  
    - HTTP errors (timeouts, 5xx errors from API)  
    - Network connectivity issues  
    - Unexpected API response structure changes  
  - Notes: ZenQuotes.io API is free and requires no key, simplifying setup.

#### 1.3 Quote Formatting

- **Overview:**  
  Normalizes the API response and formats it into a rich Slack message containing the quote and author, including fallback defaults.

- **Nodes Involved:**  
  - Format Quote for Slack

- **Node Details:**

  **Format Quote for Slack**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Reads the first JSON item from input (the API response).  
    - Extracts quote text and author, supports multiple field naming conventions (`q` or `quote` for quote, `a` or `author` for author).  
    - Constructs a Slack message with emoji, formatted text, username "MotivationBot", and star icon emoji.  
    - Sets default channel as `#general` (modifiable in code).  
  - Key Expressions:  
    - Uses `$input.first().json` for input JSON  
    - Template literal for message text with emoji and quote  
  - Input: JSON output from "Fetch Random Quote"  
  - Output: JSON with Slack message fields (`text`, `channel`, `username`, `icon_emoji`) plus raw quote/author for reference  
  - Version Requirements: Code node v2 or higher  
  - Potential Failures:  
    - If API response format changes significantly, code may fail to find quote/author fields  
    - Expression or runtime JS errors in code node  
  - Notes: Channel name is hardcoded in code; user must update to desired Slack channel.

#### 1.4 Slack Posting

- **Overview:**  
  Sends the formatted motivational quote as a message to the configured Slack channel using OAuth2 authentication.

- **Nodes Involved:**  
  - Send to Slack

- **Node Details:**

  **Send to Slack**  
  - Type: Slack node  
  - Configuration:  
    - Authentication: OAuth2 (Slack app credentials with required scopes)  
    - Channel: "#general" (by name, can be updated)  
    - Text: Uses expression `={{ $json.text }}` to post formatted message text  
    - Other options: default  
  - Input: Output JSON from "Format Quote for Slack"  
  - Output: Slack API response confirming message sent  
  - Version Requirements: Slack node v2.1 or higher  
  - Potential Failures:  
    - OAuth token invalid or expired  
    - Missing Slack app permissions (requires `chat:write` and `channels:read`)  
    - Invalid channel name or channel access issues  
    - Slack API rate limiting or downtime  
  - Notes: Requires Slack app configured and installed in workspace with proper scopes as per setup instructions.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role           | Input Node(s)         | Output Node(s)       | Sticky Note                                              |
|---------------------|-------------------|---------------------------|-----------------------|----------------------|----------------------------------------------------------|
| Setup Instructions  | Sticky Note       | Setup guidance for users  | None                  | None                 | ⭐ **SETUP REQUIRED:** 1. Connect Slack App with scopes chat:write, channels:read. 2. Update Slack channel in 'Send to Slack' node. 3. Workflow timezone America/New_York. Uses free ZenQuotes.io API - no key needed! |
| Daily 8AM Trigger   | Cron Trigger      | Schedule daily execution  | None                  | Fetch Random Quote    |                                                          |
| Fetch Random Quote  | HTTP Request      | Retrieve random quote     | Daily 8AM Trigger     | Format Quote for Slack|                                                          |
| Format Quote for Slack | Code            | Format quote for Slack    | Fetch Random Quote    | Send to Slack        |                                                          |
| Send to Slack       | Slack             | Post message to Slack     | Format Quote for Slack | None                 |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Cron Trigger Node:**  
   - Add a "Cron" node named "Daily 8AM Trigger".  
   - Configure it to run daily at 8:00 AM.  
   - Ensure the workflow timezone is set to America/New_York under workflow settings.

2. **Create the HTTP Request Node:**  
   - Add an "HTTP Request" node named "Fetch Random Quote".  
   - Set Method to GET.  
   - Set URL to `https://zenquotes.io/api/random`.  
   - No authentication required.  
   - Connect "Daily 8AM Trigger" output to this node's input.

3. **Create the Code Node:**  
   - Add a "Code" node named "Format Quote for Slack".  
   - Use JavaScript mode.  
   - Paste the following logic:

   ```javascript
   const response = $input.first().json;
   const quoteData = Array.isArray(response) ? response[0] : response;

   const formattedQuote = {
     text: `🌟 *Daily Motivation* 🌟\n\n"${quoteData.q || quoteData.quote || 'Stay positive and keep moving forward!'}"\n\n— ${quoteData.a || quoteData.author || 'Unknown'}`,
     channel: '#general', // Change to your Slack channel if needed
     username: 'MotivationBot',
     icon_emoji: ':star2:',
     raw_quote: quoteData.q || quoteData.quote,
     raw_author: quoteData.a || quoteData.author
   };

   return { json: formattedQuote };
   ```

   - Connect "Fetch Random Quote" output to this node.

4. **Create the Slack Node:**  
   - Add a "Slack" node named "Send to Slack".  
   - Set authentication to OAuth2.  
   - Configure OAuth2 credentials with Slack app that has `chat:write` and `channels:read` scopes, installed to your workspace.  
   - Set "Channel" mode to "Name" and value to `general` (or your preferred channel).  
   - Set Text field to expression `={{ $json.text }}` to use formatted message text.  
   - Connect "Format Quote for Slack" output to this node.

5. **Configure Workflow Settings:**  
   - Set timezone to America/New_York (or your local zone).  
   - Activate the workflow.

6. **Slack App Setup (Outside n8n):**  
   - Create a Slack app at https://api.slack.com/apps.  
   - Add OAuth scopes: `chat:write`, `channels:read`.  
   - Install the app into your Slack workspace.  
   - Create OAuth2 credentials in n8n with the app's Client ID/Secret.

7. **Optional Adjustments:**  
   - Modify the channel in the Code node or Slack node to target a different Slack channel.  
   - Adjust the message formatting in the Code node as desired.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                           |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Workflow uses free ZenQuotes.io API - no API key required.                                           | https://zenquotes.io/api                                  |
| Slack app requires OAuth scopes: chat:write, channels:read and must be installed in the workspace.  | https://api.slack.com/authentication/oauth-v2            |
| Workflow timezone set to America/New_York; change in workflow settings if needed for local timezone.| n8n workflow settings                                     |
| Setup instructions summarized in a sticky note node within the workflow for user convenience.       | Included in workflow "Setup Instructions" sticky note    |

---

**Disclaimer:** The provided content is solely derived from an n8n workflow automation. All processing adheres strictly to valid content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.