Daily Website Data Extraction with Firecrawl and Telegram Alerts

https://n8nworkflows.xyz/workflows/daily-website-data-extraction-with-firecrawl-and-telegram-alerts-5591


# Daily Website Data Extraction with Firecrawl and Telegram Alerts

### 1. Workflow Overview

This workflow automates daily web data extraction from a target website using the Firecrawl AI-powered scraping API and delivers structured results directly to a Telegram chat. It is designed to run once per day at a specified time, extracting specific data points framed by a custom JSON schema. The workflow incorporates a polling mechanism to wait for Firecrawl’s asynchronous extraction process to complete before fetching the results. If no data is returned initially, it retries after a short wait, ensuring reliable data retrieval. Finally, the extracted data is formatted and sent as a message via Telegram.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 6 PM.
- **1.2 Firecrawl Extraction Request:** Sends a POST request to Firecrawl with a custom extraction prompt and schema.
- **1.3 Processing Delay:** Waits 30 seconds to allow Firecrawl to process the extraction request.
- **1.4 Result Retrieval Loop:** Repeatedly requests extraction results from Firecrawl until data is available or a retry condition is met.
- **1.5 Data Formatting:** Prepares and formats the extracted data for messaging.
- **1.6 Telegram Notification:** Sends the formatted data to a specified Telegram chat.
- **1.7 Control Logic:** Uses IF and Wait nodes to manage retries and conditional flows during result retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Triggers the entire workflow once per day at 6 PM.
- **Nodes Involved:** `Schedule Trigger`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Scheduled initiation of the workflow.
    - Configuration:
      - Interval set to trigger daily at hour 18 (6 PM).
      - No additional parameters.
    - Input: None (start node)
    - Output: Connects to `Extract`
    - Edge cases:
      - Time zone considerations may affect trigger time.
      - Workflow inactive status will prevent triggering.
  
#### 1.2 Firecrawl Extraction Request

- **Overview:** Sends a POST request to Firecrawl API to start the extraction job with a specified URL, prompt, and JSON schema.
- **Nodes Involved:** `Extract`
- **Node Details:**

  - **Extract**
    - Type: HTTP Request (POST)
    - Role: Initiates Firecrawl extraction with a structured prompt and schema.
    - Configuration:
      - URL: `https://api.firecrawl.dev/v1/extract`
      - Method: POST
      - Body (JSON): Includes:
        - `urls`: array with `"https://www.website.com"` (target site)
        - `prompt`: Extraction instruction text ("Extract the [insert] information from the page")
        - `schema`: JSON schema defining expected data structure (array of notable trades with details such as congress member name, party, stock, amount, date)
      - Authentication: HTTP Header Auth (Firecrawl API key), must be configured in credentials.
      - Sends body as JSON.
    - Input: From `Schedule Trigger`
    - Output: To `30 Secs` node
    - Edge cases:
      - Authentication failure if API key missing or invalid.
      - HTTP errors (timeouts, 4xx/5xx responses).
      - Schema mismatch or invalid prompt causing extraction failure.
  
- **Sticky Note:** "Firecrawl Extract POST" (indicates this node's primary function)

#### 1.3 Processing Delay

- **Overview:** Waits 30 seconds to allow Firecrawl to process the extraction asynchronously.
- **Nodes Involved:** `30 Secs`
- **Node Details:**

  - **30 Secs**
    - Type: Wait
    - Role: Delay node to pause workflow execution for 30 seconds.
    - Configuration:
      - Amount: 30 seconds.
      - Webhook ID used internally for execution control.
    - Input: From `Extract`
    - Output: To `Get Results`
    - Edge cases:
      - Delays workflow but no error expected.
      - If Firecrawl takes longer than 30 seconds, next steps may get empty results.
  
- **Sticky Note:** "Wait 30 secs"

#### 1.4 Result Retrieval Loop

- **Overview:** Polls Firecrawl API to fetch extraction results using the request ID and loops with conditional waiting if no data is returned.
- **Nodes Involved:** `Get Results`, `If`, `Wait 15 secs`
- **Node Details:**

  - **Get Results**
    - Type: HTTP Request (GET)
    - Role: Retrieves extraction results by ID from Firecrawl.
    - Configuration:
      - URL: `https://api.firecrawl.dev/v1/extract/{{ $('Extract').item.json.id }}`
        - Uses expression to dynamically insert extraction job ID from `Extract` node response.
      - Method: GET
      - Authentication: HTTP Header Auth (Firecrawl API key).
    - Input: From `30 Secs` or `Wait 15 secs`
    - Output: To `If`
    - Edge cases:
      - Possible errors: invalid ID, API errors, network issues.
      - If response contains empty or no data, triggers retry logic.
  
  - **If**
    - Type: If (Conditional)
    - Role: Checks if the fetched results contain any trades or are empty.
    - Configuration:
      - Condition: Checks if `trades` array is empty (empty array check).
      - If True (empty): proceeds to `Wait 15 secs`.
      - If False (data present): proceeds to `Edit Fields`.
    - Input: From `Get Results`
    - Output: Two branches:
      - True → `Wait 15 secs`
      - False → `Edit Fields`
    - Edge cases:
      - Expression failure if `trades` field is missing or malformed.
  
  - **Wait 15 secs**
    - Type: Wait
    - Role: Delays 15 seconds before retrying the `Get Results` call.
    - Configuration:
      - Amount: 15 seconds.
    - Input: From `If` (True branch)
    - Output: Loops back to `Get Results`
    - Edge cases:
      - Infinite loops if results never arrive; no max retry logic present.
  
- **Sticky Notes:**
  - "GET Result Loop" (covers `Get Results`, `If`, and `Wait 15 secs` nodes)

#### 1.5 Data Formatting

- **Overview:** Prepares and sets the extracted data into a format suitable for messaging.
- **Nodes Involved:** `Edit Fields`
- **Node Details:**

  - **Edit Fields**
    - Type: Set
    - Role: Extracts the `trades` data field from the JSON response and assigns it to a new field `data` as string.
    - Configuration:
      - Assigns `data` = `{{$json.trades}}` (stringifies the trades array).
      - Includes other existing fields unchanged.
    - Input: From `If` (False branch)
    - Output: To `Telegram`
    - Edge cases:
      - If `trades` is missing or empty, may result in empty message.
  
- **Sticky Note:** "Send Result to Telegram"

#### 1.6 Telegram Notification

- **Overview:** Sends the formatted extracted data as a message to a specified Telegram chat.
- **Nodes Involved:** `Telegram`
- **Node Details:**

  - **Telegram**
    - Type: Telegram node
    - Role: Delivers the extracted data message to Telegram chat/group.
    - Configuration:
      - Text: Uses expression `{{$json.data}}` to send formatted data.
      - Chat ID: Pre-configured chat identifier (redacted in JSON).
      - Credentials: Telegram API credentials (bot token) must be set.
    - Input: From `Edit Fields`
    - Output: End of workflow
    - Edge cases:
      - Authentication failure if bot token invalid or missing.
      - Chat ID invalid or bot not authorized to send messages to that chat.
  
- **Sticky Note:** "Send Result to Telegram"

---

### 3. Summary Table

| Node Name        | Node Type             | Functional Role                        | Input Node(s)               | Output Node(s)         | Sticky Note                                                                                  |
|------------------|-----------------------|-------------------------------------|----------------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger | Schedule Trigger      | Starts workflow daily at 6 PM        | None                       | Extract               | Scheduled Trigger                                                                           |
| Extract          | HTTP Request (POST)   | Sends extraction request to Firecrawl| Schedule Trigger           | 30 Secs               | Firecrawl Extract POST                                                                      |
| 30 Secs          | Wait                  | Waits 30 seconds for extraction processing| Extract                | Get Results           | Wait 30 secs                                                                               |
| Get Results      | HTTP Request (GET)    | Fetches extraction results by ID     | 30 Secs, Wait 15 secs      | If                    | GET Result Loop                                                                            |
| If               | If (Conditional)      | Checks if extraction result is empty | Get Results                | Wait 15 secs (True), Edit Fields (False) | GET Result Loop                                                                            |
| Wait 15 secs     | Wait                  | Waits 15 seconds before retrying     | If (True branch)           | Get Results           | GET Result Loop                                                                            |
| Edit Fields      | Set                   | Formats extracted data for messaging | If (False branch)          | Telegram              | Send Result to Telegram                                                                    |
| Telegram         | Telegram              | Sends data message to Telegram chat  | Edit Fields                | None                  | Send Result to Telegram                                                                    |
| Sticky Note      | Sticky Note           | Visual annotation for Extract node   | None                       | None                  | Firecrawl Extract POST                                                                     |
| Sticky Note1     | Sticky Note           | Visual annotation for Schedule Trigger| None                      | None                  | Scheduled Trigger                                                                          |
| Sticky Note2     | Sticky Note           | Visual annotation for 30 Secs node   | None                       | None                  | Wait 30 secs                                                                              |
| Sticky Note3     | Sticky Note           | Visual annotation for Result retrieval loop| None                   | None                  | GET Result Loop                                                                           |
| Sticky Note4     | Sticky Note           | Visual annotation for Telegram node  | None                       | None                  | Send Result to Telegram                                                                   |
| Sticky Note5     | Sticky Note           | Workflow overview and detailed description| None                   | None                  | 🔥 Automated Daily Firecrawl Scraper with Telegram Alerts... (full content in JSON)        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**
   - Type: Schedule Trigger
   - Configure to trigger daily at 18:00 (6 PM).
   - This node starts the workflow.

2. **Create `Extract` node:**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://api.firecrawl.dev/v1/extract`
   - Authentication: HTTP Header Auth (configure Firecrawl API key credential)
   - Body Content Type: JSON
   - JSON Body (enter as expression or raw JSON):
     ```json
     {
       "urls": ["https://www.website.com"],
       "prompt": "Extract the [insert] information from the page",
       "schema": {
         "type": "object",
         "properties": {
           "notable_trades": {
             "type": "array",
             "items": {
               "type": "object",
               "properties": {
                 "congress_member_name": {"type": "string"},
                 "party": {"type": "string"},
                 "stock_or_asset": {"type": "string"},
                 "amount": {"type": "number"},
                 "transaction_date": {"type": "string"}
               },
               "required": ["congress_member_name", "stock_or_asset", "amount", "transaction_date"]
             }
           }
         },
         "required": ["notable_trades"]
       }
     }
     ```
   - Connect `Schedule Trigger` → `Extract`.

3. **Create `30 Secs` node:**
   - Type: Wait
   - Parameters: Wait time set to 30 seconds.
   - Connect `Extract` → `30 Secs`.

4. **Create `Get Results` node:**
   - Type: HTTP Request
   - Method: GET
   - URL: `https://api.firecrawl.dev/v1/extract/{{ $('Extract').item.json.id }}`
     - Use expression to inject extraction job ID from `Extract` node.
   - Authentication: HTTP Header Auth (same Firecrawl API key credential).
   - Connect `30 Secs` → `Get Results`.

5. **Create `If` node:**
   - Type: If
   - Condition: Check if `trades` array is empty.
     - Use expression: `{{$json.trades}}`
     - Operator: Array is empty
   - True branch (empty data): Connect to `Wait 15 secs`.
   - False branch (data present): Connect to `Edit Fields`.
   - Connect `Get Results` → `If`.

6. **Create `Wait 15 secs` node:**
   - Type: Wait
   - Parameters: Wait time set to 15 seconds.
   - Connect `If` (True) → `Wait 15 secs`.
   - Connect `Wait 15 secs` → `Get Results` (loop back to retry).

7. **Create `Edit Fields` node:**
   - Type: Set
   - Add a field assignment:
     - Field Name: `data`
     - Value: `={{ $json.trades }}`
   - Keep other fields unchanged.
   - Connect `If` (False) → `Edit Fields`.

8. **Create `Telegram` node:**
   - Type: Telegram
   - Text: `={{ $json.data }}`
   - Chat ID: Enter your Telegram chat ID.
   - Credentials: Configure Telegram API credentials (bot token).
   - Connect `Edit Fields` → `Telegram`.

9. **Add Sticky Notes (optional for clarity):**
   - Add descriptive sticky notes near key nodes for documentation purposes.

10. **Activate and test the workflow:**
    - Ensure credentials are valid.
    - Confirm Firecrawl API key and Telegram bot token have proper permissions.
    - Run manually or wait for scheduled trigger.
    - Monitor logs for errors or infinite loops.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| 🔥 Automated Daily Firecrawl Scraper with Telegram Alerts. Get structured insights scraped daily from the web using Firecrawl’s AI extraction engine — then send them directly to your Telegram chat. Use cases include compliance monitoring, intelligence gathering, market alert bots, and more. Customizable to integrate with Gmail, Discord, Slack, or Google Sheets. | Workflow overview and use case description from workflow's Sticky Note5.                        |
| For step-by-step video tutorials of n8n builds and automation ideas, visit: https://www.youtube.com/@Automatewithmarc                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | YouTube channel link from workflow's Sticky Note5.                                            |
| Firecrawl API key is required and must be set in HTTP Header Auth credentials for the HTTP Request nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Authentication detail for Firecrawl API.                                                       |
| Telegram Bot Token and Chat ID must be configured in Telegram node credentials and parameters.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Authentication detail for Telegram node.                                                      |
| No maximum retry limit implemented in the result polling loop; consider adding to prevent infinite loops if Firecrawl fails to return data.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Suggested improvement for error handling and robustness.                                      |
| Time zone awareness: Scheduled trigger time depends on n8n instance time zone settings; verify to match intended daily run time.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | General scheduling consideration.                                                            |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.