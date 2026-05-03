Daily Competitor WordPress Analysis with Charts, Gemini & Telegram Reporting

https://n8nworkflows.xyz/workflows/daily-competitor-wordpress-analysis-with-charts--gemini---telegram-reporting-8443


# Daily Competitor WordPress Analysis with Charts, Gemini & Telegram Reporting

### 1. Workflow Overview

This workflow automates a daily competitive analysis for WordPress-related opponents by aggregating their RSS feed data, processing it into structured data, generating visual insights, and sending reports via Telegram. It is designed for digital marketers, SEO analysts, or content strategists seeking a daily snapshot of competitor activities alongside graphical and AI-enhanced analysis.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Data Collection:** Triggering the process daily and fetching RSS feeds from three competitor sources.
- **1.2 Data Limiting and Merging:** Limiting the volume of fetched RSS items and merging them into a consolidated dataset.
- **1.3 Data Aggregation and Storage:** Aggregating merged data, appending or updating it in a Google Sheets document for record-keeping.
- **1.4 Visual & AI Processing:** Generating screenshots of the data via an external API, analyzing images with Google Gemini AI, and preparing content for messaging.
- **1.5 Reporting:** Sending the processed summary and insights as a Telegram message to a configured chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Collection

- **Overview:**  
This block initiates the workflow on a schedule and fetches competitor data from three RSS feeds in parallel.

- **Nodes Involved:**  
  - Schedule Trigger  
  - RSS opponent 1  
  - RSS opponent 2  
  - RSS opponent 3

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution daily at a configured time (time config not explicitly shown but implied).  
    - Input: None (trigger node)  
    - Output: Triggers three HTTP Request nodes in parallel.  
    - Failure Modes: Misconfiguration of schedule or node failure may cause missed runs.

  - **RSS opponent 1 / 2 / 3**  
    - Type: HTTP Request  
    - Role: Fetch RSS feeds from three different competitor URLs (likely WordPress blogs).  
    - Configuration: Each node uses GET request to respective RSS feed URLs (URLs not visible in JSON).  
    - Input: Triggered by Schedule Trigger.  
    - Output: Response containing RSS feed data passed to respective Limit nodes.  
    - Failure Modes: HTTP errors (404, 500), network issues, invalid RSS format.  
    - Notes: Three separate nodes allow parallel data collection.

#### 2.2 Data Limiting and Merging

- **Overview:**  
Limits the number of RSS items processed per feed to control data volume, then merges the limited datasets into one.

- **Nodes Involved:**  
  - Limit (for RSS opponent 1)  
  - Limit2 (for RSS opponent 2)  
  - Limit1 (for RSS opponent 3)  
  - Merge

- **Node Details:**

  - **Limit / Limit1 / Limit2**  
    - Type: Limit  
    - Role: Limits the number of items passed downstream (exact limit parameter not specified).  
    - Input: RSS feed data from respective HTTP Request nodes.  
    - Output: Limited data sets forwarded to Merge.  
    - Failure Modes: Expression errors if input data is malformed.

  - **Merge**  
    - Type: Merge  
    - Role: Combines the three limited data streams into one consolidated dataset.  
    - Configuration: Likely using 'Merge by Append' mode to stack items.  
    - Input: Three inputs from the Limit nodes.  
    - Output: Combined dataset sent to Google Sheets node.  
    - Failure Modes: Merge conflicts or mismatched data structures.

#### 2.3 Data Aggregation and Storage

- **Overview:**  
Aggregates the merged data possibly to summarize or reformat and appends or updates the result in a Google Sheets spreadsheet for persistent storage.

- **Nodes Involved:**  
  - Append or update row in sheet  
  - Aggregate

- **Node Details:**

  - **Append or update row in sheet**  
    - Type: Google Sheets  
    - Role: Inserts or updates rows in a designated Google Sheets document.  
    - Configuration: Specific spreadsheet ID, sheet name, and key columns (not explicitly shown).  
    - Input: Combined dataset from Merge node.  
    - Output: Passes data to Aggregate node, possibly to prepare data for next step.  
    - Failure Modes: Authentication errors, Google API limits, incorrect sheet configuration.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Processes rows to calculate aggregates or transform data for visualization.  
    - Input: Output from Google Sheets node.  
    - Output: Feeds data into the screenshot generation API.  
    - Failure Modes: Data inconsistency or empty input.

#### 2.4 Visual & AI Processing

- **Overview:**  
Generates a screenshot of the aggregated data using an external HTTP API (APIFlash), analyzes the image via Google Gemini AI language model, and prepares content for messaging.

- **Nodes Involved:**  
  - APIFlash  
  - Analyze image  
  - Send a text message

- **Node Details:**

  - **APIFlash**  
    - Type: HTTP Request  
    - Role: Calls APIFlash to generate a screenshot image of the aggregated data or dashboard.  
    - Configuration: HTTP GET or POST with URL and API credentials (API key).  
    - Input: Receives aggregated data from Aggregate node.  
    - Output: Image or image URL passed to Analyze image.  
    - Failure Modes: API request failures, invalid parameters, rate limiting.

  - **Analyze image**  
    - Type: Google Gemini (LangChain node)  
    - Role: Uses Google Gemini AI to analyze the generated screenshot image and extract insights or text.  
    - Configuration: Linked to Google Gemini credentials, configured for image analysis.  
    - Input: Image data from APIFlash node.  
    - Output: Textual AI analysis sent to Telegram node.  
    - Failure Modes: API quota exceeded, authentication errors, malformed image data.

  - **Send a text message**  
    - Type: Telegram  
    - Role: Sends the final report (text + insights) to a Telegram chat.  
    - Configuration: Uses Telegram bot credentials, chat ID configured in parameters.  
    - Input: AI-analyzed text from Analyze image node.  
    - Output: Message delivery status.  
    - Failure Modes: Invalid bot token, chat ID, network issues.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                          | Input Node(s)                   | Output Node(s)                  | Sticky Note                                |
|---------------------------|------------------------|----------------------------------------|--------------------------------|--------------------------------|--------------------------------------------|
| Schedule Trigger           | Schedule Trigger       | Initiates daily workflow execution     | None                           | RSS opponent 1,2,3             |                                            |
| RSS opponent 1             | HTTP Request           | Fetch RSS feed from competitor 1       | Schedule Trigger               | Limit                         |                                            |
| RSS opponent 2             | HTTP Request           | Fetch RSS feed from competitor 2       | Schedule Trigger               | Limit2                        |                                            |
| RSS opponent 3             | HTTP Request           | Fetch RSS feed from competitor 3       | Schedule Trigger               | Limit1                        |                                            |
| Limit                     | Limit                  | Limit RSS opponent 1 items              | RSS opponent 1                 | Merge                        |                                            |
| Limit2                    | Limit                  | Limit RSS opponent 2 items              | RSS opponent 2                 | Merge                        |                                            |
| Limit1                    | Limit                  | Limit RSS opponent 3 items              | RSS opponent 3                 | Merge                        |                                            |
| Merge                     | Merge                  | Merge limited RSS data streams          | Limit, Limit2, Limit1          | Append or update row in sheet |                                            |
| Append or update row in sheet | Google Sheets         | Append or update data in Google Sheet   | Merge                         | Aggregate                    |                                            |
| Aggregate                 | Aggregate              | Aggregate or transform sheet data       | Append or update row in sheet  | APIFlash                     |                                            |
| APIFlash                  | HTTP Request           | Generate data screenshot via API        | Aggregate                     | Analyze image                |                                            |
| Analyze image             | Google Gemini AI       | Analyze screenshot image with AI        | APIFlash                     | Send a text message          |                                            |
| Send a text message       | Telegram               | Send report via Telegram bot             | Analyze image                 | None                        |                                            |
| Sticky Note               | Sticky Note            | (Empty / no content)                     | None                         | None                        |                                            |
| Sticky Note1              | Sticky Note            | (Empty / no content)                     | None                         | None                        |                                            |
| Sticky Note2              | Sticky Note            | (Empty / no content)                     | None                         | None                        |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at desired time (e.g., 07:00 AM).  
   - No input, output triggers next nodes.

2. **Create three HTTP Request nodes for RSS feeds**  
   - Names: RSS opponent 1, RSS opponent 2, RSS opponent 3  
   - Method: GET  
   - URL: Enter respective competitor RSS URLs (e.g., https://competitor1.com/feed)  
   - Connect Schedule Trigger output to all three HTTP Request nodes in parallel.

3. **Add three Limit nodes**  
   - Names: Limit, Limit1, Limit2  
   - Purpose: Limit the number of items from each RSS feed (set e.g., to 5 items).  
   - Connect RSS opponent 1 to Limit, RSS opponent 2 to Limit2, RSS opponent 3 to Limit1.

4. **Add Merge node**  
   - Type: Merge  
   - Mode: Append  
   - Connect outputs of Limit, Limit2, and Limit1 to inputs 0,1,2 of Merge node respectively.

5. **Add Google Sheets node**  
   - Name: Append or update row in sheet  
   - Operation: Append or update rows  
   - Credentials: Configure Google Sheets OAuth2 credentials with access to target spreadsheet  
   - Parameters: Set spreadsheet ID, sheet name, and key columns for update/append  
   - Connect Merge output to this node.

6. **Add Aggregate node**  
   - Type: Aggregate  
   - Configure aggregation operations as needed (e.g., group by date, count, sum)  
   - Connect Google Sheets node output to Aggregate node.

7. **Add HTTP Request node for APIFlash**  
   - Name: APIFlash  
   - Method: GET or POST depending on API requirements  
   - URL: APIFlash endpoint for screenshot generation (e.g., https://apiflash.com/api/urltoimage)  
   - Authentication: Add API key in headers or query parameters  
   - Pass aggregated data or dashboard URL as parameter  
   - Connect Aggregate output to APIFlash node.

8. **Add Google Gemini node**  
   - Name: Analyze image  
   - Node type: @n8n/n8n-nodes-langchain.googleGemini  
   - Credentials: Configure Google Gemini API credentials  
   - Configure node to analyze image from APIFlash response  
   - Connect APIFlash output to Analyze image node.

9. **Add Telegram node**  
   - Name: Send a text message  
   - Credentials: Configure Telegram bot token and chat ID  
   - Message: Use output from Analyze image node to compose message content  
   - Connect Analyze image output to Telegram node.

10. **Connect nodes in the sequence as per described connections**  
    - Schedule Trigger → RSS opponents → Limit nodes → Merge → Google Sheets → Aggregate → APIFlash → Analyze image → Telegram

11. **Test workflow end-to-end**  
    - Validate credentials and API keys  
    - Verify RSS feed URLs and limits  
    - Confirm Google Sheets access and sheet structure  
    - Confirm Telegram bot functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                 |
|-----------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow leverages Google Gemini AI via LangChain node for advanced AI image analysis.   | n8n marketplace node: @n8n/n8n-nodes-langchain |
| APIFlash is used as an external service to generate screenshots of web pages or dashboards.   | https://apiflash.com                            |
| Telegram node requires setting up a Telegram bot and obtaining bot token and chat ID.         | https://core.telegram.org/bots/api              |
| Google Sheets integration requires setting up OAuth2 credentials with Google Cloud Console.   | https://developers.google.com/sheets/api       |
| Scheduling can be customized to any desired daily run time in the Schedule Trigger node.      | n8n documentation on Schedule Trigger          |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.