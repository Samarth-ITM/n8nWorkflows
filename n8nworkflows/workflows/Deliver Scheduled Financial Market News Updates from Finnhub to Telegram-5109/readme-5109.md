Deliver Scheduled Financial Market News Updates from Finnhub to Telegram

https://n8nworkflows.xyz/workflows/deliver-scheduled-financial-market-news-updates-from-finnhub-to-telegram-5109


# Deliver Scheduled Financial Market News Updates from Finnhub to Telegram

### 1. Workflow Overview

This workflow automates the delivery of scheduled financial market news updates fetched from Finnhub’s API to a Telegram channel or chat. It targets users interested in receiving automated, timely financial news summaries every four hours, optionally including related images.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every four hours.
- **1.2 API Key Setup:** Prepares the necessary authentication credential for Finnhub API calls.
- **1.3 Fetch Latest News:** Retrieves the most recent financial news from Finnhub.
- **1.4 Optional Image Download:** Downloads associated images from the news items if available.
- **1.5 Data Processing:** Processes and formats the news data into a Telegram-friendly message.
- **1.6 Telegram Notification:** Sends the formatted news update as a text message and optionally as an image post to Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow execution every four hours, ensuring periodic updates without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger - Every 4 Hours

- **Node Details:**  
  - **Schedule Trigger - Every 4 Hours**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger the workflow cyclically every 4 hours.  
    - Inputs: None (trigger node).  
    - Outputs: Connected downstream to "Set API Key for Finhubb".  
    - Edge Cases: Misconfiguration could cause missed or excessive triggers; time zone settings may affect exact timing.  
    - Notes: Requires n8n version supporting Schedule Trigger v1.2 or newer.

#### 2.2 API Key Setup

- **Overview:**  
  This block sets the Finnhub API key required for subsequent HTTP requests to authenticate and authorize data retrieval.

- **Nodes Involved:**  
  - Set API Key for Finhubb

- **Node Details:**  
  - **Set API Key for Finhubb**  
    - Type: Set Node  
    - Configuration: Sets a variable/parameter containing the Finnhub API key (likely stored securely as a credential or environment variable).  
    - Inputs: Triggered by the schedule node.  
    - Outputs: Passes the API key parameter to "Get Latest News".  
    - Edge Cases: API key missing, expired, or invalid will cause HTTP request failures downstream.  
    - Notes: Ensure the API key is securely stored and referenced.

#### 2.3 Fetch Latest News

- **Overview:**  
  This block makes an authenticated HTTP request to Finnhub to fetch the latest financial market news.

- **Nodes Involved:**  
  - Get Latest News

- **Node Details:**  
  - **Get Latest News**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET  
      - URL: Finnhub endpoint for latest news (exact URL not provided here)  
      - Authentication: Uses the API key set previously, likely as a query parameter or header.  
    - Inputs: Receives API key from previous node.  
    - Outputs: Provides JSON data about latest news to "Download Image (optional)".  
    - Edge Cases: Network issues, API rate limits, invalid API key, malformed response.  
    - Version: HTTP Request node version 4.2 used.  
    - Notes: Proper error handling should be implemented for failed requests.

#### 2.4 Optional Image Download

- **Overview:**  
  This block attempts to download images associated with the news, if any, to enrich the Telegram message.

- **Nodes Involved:**  
  - Download Image (optional)

- **Node Details:**  
  - **Download Image (optional)**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET  
      - URL: Dynamically set from news data image URLs (via expressions).  
      - Response Format: Binary data (image file).  
    - Inputs: Receives news data from "Get Latest News".  
    - Outputs: Passes image binary data to "Code" node.  
    - Edge Cases: News items without images, broken URLs, slow responses, unsupported image formats.  
    - Notes: This node is optional; workflow can continue without images.

#### 2.5 Data Processing

- **Overview:**  
  Processes and formats the news data and image into messages suitable for Telegram posting.

- **Nodes Involved:**  
  - Code

- **Node Details:**  
  - **Code**  
    - Type: Code Node (JavaScript)  
    - Configuration:  
      - Custom code to parse news JSON, construct text summaries, and prepare message payloads for Telegram.  
      - Handles optional image data integration.  
    - Inputs: Receives image binary data and news JSON.  
    - Outputs: Passes formatted message data to "Send Text Updates via Telegram".  
    - Edge Cases: Code errors, unexpected data structure changes, missing fields.  
    - Version: Code node v2 used.  
    - Notes: Robustness depends on code quality; consider error catching.

#### 2.6 Telegram Notification

- **Overview:**  
  Sends the financial news updates as Telegram messages, first as text, then optionally as an image with caption.

- **Nodes Involved:**  
  - Send Text Updates via Telegram  
  - Send Image (with text update) - Optional

- **Node Details:**  
  - **Send Text Updates via Telegram**  
    - Type: Telegram Node  
    - Configuration:  
      - Uses Telegram Bot API credentials (OAuth2 or token) for authorization.  
      - Sends text messages with the news summary.  
    - Inputs: Receives formatted text from "Code" node.  
    - Outputs: Connects to "Send Image (with text update) - Optional" node.  
    - Edge Cases: Telegram API rate limits, bot permissions, network errors.  
    - Notes: Ensure bot is added to target chat/channel with send message rights.  

  - **Send Image (with text update) - Optional**  
    - Type: Telegram Node  
    - Configuration:  
      - Sends photo messages, including image binary and caption text.  
    - Inputs: Receives message and image data from previous Telegram node.  
    - Outputs: None (end of workflow).  
    - Edge Cases: Image size limits, unsupported formats, Telegram API errors.  
    - Notes: Optional step; can be disabled if no images available or desired.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                       | Input Node(s)                   | Output Node(s)                     | Sticky Note |
|-------------------------------|--------------------|------------------------------------|--------------------------------|----------------------------------|-------------|
| Schedule Trigger - Every 4 Hours | Schedule Trigger   | Initiate workflow every 4 hours    | None                           | Set API Key for Finhubb           |             |
| Set API Key for Finhubb         | Set                | Set Finnhub API key parameter      | Schedule Trigger - Every 4 Hours | Get Latest News                   |             |
| Get Latest News                 | HTTP Request       | Fetch latest financial news        | Set API Key for Finhubb          | Download Image (optional)          |             |
| Download Image (optional)       | HTTP Request       | Download news images if present    | Get Latest News                 | Code                             |             |
| Code                           | Code               | Format news and prepare messages   | Download Image (optional)       | Send Text Updates via Telegram    |             |
| Send Text Updates via Telegram  | Telegram           | Send news text updates              | Code                           | Send Image (with text update) - Optional |             |
| Send Image (with text update) - Optional | Telegram           | Send news image with caption        | Send Text Updates via Telegram  | None                             |             |
| Sticky Note                    | Sticky Note        | Visual note (empty)                 | None                           | None                            |             |
| Sticky Note1                   | Sticky Note        | Visual note (empty)                 | None                           | None                            |             |
| Sticky Note2                   | Sticky Note        | Visual note (empty)                 | None                           | None                            |             |
| Sticky Note3                   | Sticky Note        | Visual note (empty)                 | None                           | None                            |             |
| Sticky Note4                   | Sticky Note        | Visual note (empty)                 | None                           | None                            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Name: *Schedule Trigger - Every 4 Hours*  
   - Set to trigger every 4 hours (using the Cron or Interval mode).  
   - No inputs; outputs connected to next node.

2. **Add a Set node:**
   - Name: *Set API Key for Finhubb*  
   - Configure to set a key-value pair for the Finnhub API key.  
   - Use an environment variable or parameter to store the key securely.  
   - Connect the output of the Schedule Trigger node to this node’s input.

3. **Add an HTTP Request node:**
   - Name: *Get Latest News*  
   - Method: GET  
   - URL: Finnhub API endpoint for latest news, e.g., `https://finnhub.io/api/v1/news?category=general` (adjust as needed).  
   - Authentication: Add API key either as a query parameter or header using the value from the previous Set node.  
   - Connect input from *Set API Key for Finhubb*.  
   - Output to next node.

4. **Add an HTTP Request node for image download (optional):**
   - Name: *Download Image (optional)*  
   - Method: GET  
   - URL: Use an expression to dynamically extract image URLs from the news data (e.g., `{{$json["image"]}}`).  
   - Response format: Set to return binary data for images.  
   - Connect input from *Get Latest News*.  
   - Output to next node.

5. **Add a Code node:**
   - Name: *Code*  
   - Use JavaScript to process input JSON from news and optionally include downloaded image data.  
   - Format the message text for Telegram, including headlines, summaries, and links.  
   - Prepare message payload(s) for Telegram nodes.  
   - Connect input from *Download Image (optional)*.  
   - Output to next node.

6. **Add a Telegram node:**
   - Name: *Send Text Updates via Telegram*  
   - Configure credentials with your Telegram Bot API token.  
   - Set action to send a text message to the desired chat/channel.  
   - Use message payload from the Code node.  
   - Connect input from *Code*.  
   - Output to next node.

7. **Add a Telegram node (optional):**
   - Name: *Send Image (with text update) - Optional*  
   - Use same Telegram credentials.  
   - Set action to send photo with caption.  
   - Use binary image data and text caption from previous node.  
   - Connect input from *Send Text Updates via Telegram*.  
   - No output (end node).

8. **Credentials Setup:**
   - Create and store Finnhub API key securely in n8n credentials or environment variables.  
   - Create Telegram Bot credentials (token-based).  
   - Assign credentials to the respective nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                      |
|----------------------------------------------------------------------------------------------------|------------------------------------|
| Finnhub API documentation: https://finnhub.io/docs/api                                            | API Reference                      |
| Telegram Bot API documentation: https://core.telegram.org/bots/api                                | Telegram Integration               |
| Ensure Telegram bot is added to the target channel or group with permission to send messages       | Telegram Bot Setup                 |
| Consider adding error handling and alerting on API failures or rate limits                         | Workflow robustness                |
| Optional image download can be disabled if bandwidth or storage concerns arise                      | Performance consideration          |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.