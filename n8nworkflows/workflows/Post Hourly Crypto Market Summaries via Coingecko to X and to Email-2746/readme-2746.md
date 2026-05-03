Post Hourly Crypto Market Summaries via Coingecko to X and to Email

https://n8nworkflows.xyz/workflows/post-hourly-crypto-market-summaries-via-coingecko-to-x-and-to-email-2746


# Post Hourly Crypto Market Summaries via Coingecko to X and to Email

### 1. Workflow Overview

This workflow automates the hourly delivery of cryptocurrency market summaries, focusing by default on Bitcoin, by leveraging the CoinGecko API. It fetches real-time market data, formats it into an engaging message, and distributes the update via two channels: posting on X (formerly Twitter) and sending an email notification. The workflow is modular and customizable, allowing adaptation for different cryptocurrencies, schedules, or additional notification channels.

Logical blocks:

- **1.1 Trigger Block:** Initiates the workflow on a scheduled hourly basis.
- **1.2 Data Retrieval Block:** Fetches current cryptocurrency market data from CoinGecko.
- **1.3 Message Formatting Block:** Processes raw data into a structured, human-readable message.
- **1.4 Distribution Block:** Posts the formatted message on X and sends it via email.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:**  
  This block triggers the entire workflow on a recurring schedule, set by default to every hour.

- **Nodes Involved:**  
  - Crypto Hourly Trigger

- **Node Details:**

  - **Crypto Hourly Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow execution at defined intervals.  
    - Configuration: Default hourly trigger (exact cron or interval not explicitly specified but implied hourly).  
    - Inputs: None (start node).  
    - Outputs: Connects to "Fetch Bitcoin Data" node.  
    - Version: 1.2  
    - Edge Cases:  
      - If the scheduler is disabled or misconfigured, the workflow will not run.  
      - Timezone considerations may affect trigger timing.  
    - Notes: Can be customized to different intervals or timezones as needed.

#### 2.2 Data Retrieval Block

- **Overview:**  
  Retrieves up-to-date cryptocurrency market data from the CoinGecko API, focusing on Bitcoin by default.

- **Nodes Involved:**  
  - Fetch Bitcoin Data

- **Node Details:**

  - **Fetch Bitcoin Data**  
    - Type: HTTP Request  
    - Role: Calls the CoinGecko API to fetch market data for Bitcoin.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: CoinGecko API endpoint for Bitcoin market data (exact URL not shown but typically something like `https://api.coingecko.com/api/v3/coins/bitcoin`).  
      - Authentication: None required (CoinGecko API is public).  
      - Response Format: JSON.  
    - Inputs: Receives trigger from "Crypto Hourly Trigger".  
    - Outputs: Passes data to "Format Crypto Message".  
    - Version: 4.2  
    - Edge Cases:  
      - API rate limiting by CoinGecko may cause failures or delays.  
      - Network errors or downtime of CoinGecko API.  
      - Unexpected API response structure changes.  
    - Notes: Can be adapted to fetch data for other cryptocurrencies by modifying the API endpoint or parameters.

#### 2.3 Message Formatting Block

- **Overview:**  
  Converts raw JSON data from CoinGecko into a well-structured, visually engaging message suitable for social media posting and email content.

- **Nodes Involved:**  
  - Format Crypto Message

- **Node Details:**

  - **Format Crypto Message**  
    - Type: Code (JavaScript)  
    - Role: Processes and formats the raw market data into a human-readable message string.  
    - Configuration:  
      - Contains custom JavaScript code that extracts key data points (e.g., current price, market cap, 24h change) and formats them into a text message.  
      - May include emojis, line breaks, and other formatting for readability and engagement.  
    - Inputs: Receives JSON data from "Fetch Bitcoin Data".  
    - Outputs: Sends formatted message to both "Post Crypto Update on X" and "Send Crypto Update Email".  
    - Version: 2  
    - Edge Cases:  
      - If the input data structure changes, the code may fail or produce incorrect messages.  
      - Potential runtime errors in JavaScript code (e.g., null references).  
    - Notes: This node is the central transformation point and can be extended to include more data or different formatting styles.

#### 2.4 Distribution Block

- **Overview:**  
  Distributes the formatted cryptocurrency update message by posting it on X and sending it via email.

- **Nodes Involved:**  
  - Post Crypto Update on X  
  - Send Crypto Update Email

- **Node Details:**

  - **Post Crypto Update on X**  
    - Type: Twitter (X) node  
    - Role: Posts the formatted message as a tweet/status update on X.  
    - Configuration:  
      - Uses OAuth2 credentials for authentication with X API.  
      - Message content is set dynamically from the output of "Format Crypto Message".  
    - Inputs: Receives formatted message from "Format Crypto Message".  
    - Outputs: None (end node).  
    - Version: 2  
    - Edge Cases:  
      - Authentication failures due to expired or invalid OAuth2 tokens.  
      - API rate limits or posting restrictions on X.  
      - Message length exceeding X limits.  
    - Notes: OAuth2 credentials must be properly configured in n8n credentials manager.

  - **Send Crypto Update Email**  
    - Type: Gmail node  
    - Role: Sends the formatted message as an email to specified recipients.  
    - Configuration:  
      - Uses Gmail OAuth2 credentials for authentication.  
      - Email fields (To, Subject, Body) are dynamically populated with the formatted message.  
    - Inputs: Receives formatted message from "Format Crypto Message".  
    - Outputs: None (end node).  
    - Version: 2.1  
    - Edge Cases:  
      - Authentication errors due to invalid or expired Gmail OAuth2 tokens.  
      - Email delivery failures (e.g., invalid recipient address).  
      - Gmail API rate limits.  
    - Notes: Requires Gmail OAuth2 credentials setup in n8n.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                 | Input Node(s)           | Output Node(s)                         | Sticky Note                                  |
|-------------------------|---------------------|--------------------------------|------------------------|---------------------------------------|----------------------------------------------|
| Crypto Hourly Trigger   | Schedule Trigger    | Initiates workflow hourly       | —                      | Fetch Bitcoin Data                    |                                              |
| Fetch Bitcoin Data      | HTTP Request       | Retrieves crypto market data    | Crypto Hourly Trigger   | Format Crypto Message                 |                                              |
| Format Crypto Message   | Code               | Formats raw data into message   | Fetch Bitcoin Data      | Post Crypto Update on X, Send Crypto Update Email |                                              |
| Post Crypto Update on X | Twitter (X) node   | Posts message on X              | Format Crypto Message   | —                                     |                                              |
| Send Crypto Update Email| Gmail node         | Sends message via email         | Format Crypto Message   | —                                     |                                              |
| Sticky Note             | Sticky Note        | —                              | —                      | —                                     |                                              |
| Sticky Note1            | Sticky Note        | —                              | —                      | —                                     |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Crypto Hourly Trigger`  
   - Type: Schedule Trigger  
   - Set to trigger every hour (default). Adjust cron or interval as needed.  
   - No credentials required.  

2. **Create HTTP Request Node**  
   - Name: `Fetch Bitcoin Data`  
   - Type: HTTP Request  
   - Connect input from `Crypto Hourly Trigger`.  
   - Configure:  
     - HTTP Method: GET  
     - URL: `https://api.coingecko.com/api/v3/coins/bitcoin` (or appropriate CoinGecko endpoint)  
     - Response Format: JSON  
     - No authentication needed.  

3. **Create Code Node**  
   - Name: `Format Crypto Message`  
   - Type: Code (JavaScript)  
   - Connect input from `Fetch Bitcoin Data`.  
   - Write JavaScript code to:  
     - Parse JSON data from CoinGecko.  
     - Extract relevant fields (e.g., current price, market cap, 24h change).  
     - Format a message string with line breaks, emojis, and labels for readability.  
   - Output the formatted message as a string property (e.g., `message`).  

4. **Create Twitter Node**  
   - Name: `Post Crypto Update on X`  
   - Type: Twitter (X) node  
   - Connect input from `Format Crypto Message`.  
   - Configure OAuth2 credentials for X API access.  
   - Set the tweet content dynamically from the formatted message output of the previous node.  
   - Ensure message length complies with X limits.  

5. **Create Gmail Node**  
   - Name: `Send Crypto Update Email`  
   - Type: Gmail node  
   - Connect input from `Format Crypto Message`.  
   - Configure Gmail OAuth2 credentials.  
   - Set email parameters:  
     - To: recipient email address(es)  
     - Subject: e.g., "Hourly Crypto Market Update"  
     - Body: use the formatted message content.  

6. **Connect Nodes**  
   - `Crypto Hourly Trigger` → `Fetch Bitcoin Data`  
   - `Fetch Bitcoin Data` → `Format Crypto Message`  
   - `Format Crypto Message` → `Post Crypto Update on X`  
   - `Format Crypto Message` → `Send Crypto Update Email`  

7. **Test Workflow**  
   - Run manually or wait for scheduled trigger.  
   - Verify data retrieval, message formatting, posting on X, and email delivery.  

8. **Optional Customizations**  
   - Modify HTTP Request URL to track other cryptocurrencies.  
   - Adjust schedule trigger for different intervals.  
   - Add additional notification nodes (Slack, Telegram) connected from `Format Crypto Message`.  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                  |
|------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| CoinGecko API is public and does not require authentication but has rate limits. | https://www.coingecko.com/en/api                                                                |
| X (Twitter) API requires OAuth2 credentials configured in n8n for posting.   | https://developer.twitter.com/en/docs/authentication/oauth-2-0                                      |
| Gmail node requires OAuth2 credentials with appropriate scopes for sending email. | https://developers.google.com/identity/protocols/oauth2                                            |
| Workflow can be extended to include other cryptocurrencies by changing API endpoint. |                                                                                                 |
| Consider timezone settings in Schedule Trigger to align posting times.       |                                                                                                 |
| For message formatting, ensure code node handles API response changes gracefully. |                                                                                                 |