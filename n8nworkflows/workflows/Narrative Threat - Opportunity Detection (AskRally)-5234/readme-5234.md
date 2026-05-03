Narrative Threat / Opportunity Detection (AskRally)

https://n8nworkflows.xyz/workflows/narrative-threat---opportunity-detection--askrally--5234


# Narrative Threat / Opportunity Detection (AskRally)

### 1. Workflow Overview

This workflow, titled **Narrative Threat / Opportunity Detection (AskRally)**, is designed to monitor news content from an RSS feed, extract and clean article content, analyze audience sentiment via the AskRally API, and send alert emails summarizing narrative trends and sample responses.  
It targets use cases such as real-time monitoring of emerging narratives in media to detect shifts in public interest or opinion relevant to synthetic research spending, providing actionable insights through automated polling and reporting.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled RSS Feed Polling and Preprocessing:** Periodically reads an RSS feed, filters out old/duplicate items, and extracts clean URLs.  
- **1.2 Content Fetching and Preparation:** Retrieves full article HTML content and processes it to extract a clean text snippet and metadata.  
- **1.3 AskRally API Interaction:** Sends prepared content to the AskRally API for synthetic audience polling and receives simulated response data.  
- **1.4 Response Analysis:** Parses and aggregates the AskRally API responses to quantify narrative sentiment and select representative voter comments.  
- **1.5 Alert Generation and Email Sending:** Determines whether an alert should be triggered based on narrative thresholds and sends a rich HTML email summarizing the findings.

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled RSS Feed Polling and Preprocessing

**Overview:**  
This block initiates the workflow by polling an RSS feed every 2 hours, filters out news items older than 3 hours, removes duplicates, and extracts clean URLs for further processing.

**Nodes Involved:**  
- Schedule Trigger  
- RSS Read  
- Dedup + url extraction (Code)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow every 2 hours automatically.  
  - Configuration: Interval set to 2 hours.  
  - Inputs: None (trigger).  
  - Outputs: Emits trigger event to RSS Read node.  
  - Edge cases: Scheduling misfires if n8n server is down or delayed.

- **RSS Read**  
  - Type: RSS Feed Read  
  - Role: Reads RSS feed entries from Google Alerts RSS URL.  
  - Configuration: URL set to a specific Google Alerts RSS feed.  
  - Inputs: Trigger from Schedule Trigger.  
  - Outputs: List of RSS items.  
  - Edge cases: RSS feed could be empty, unreachable, or malformed.

- **Dedup + url extraction**  
  - Type: Code (JavaScript)  
  - Role: Filters out RSS items older than 3 hours (threshold configurable), and extracts clean URLs, decoding Google redirect URLs when present.  
  - Key logic:  
    - Parse publication date to compute age.  
    - Filter items by age threshold.  
    - Decode URLs embedded in Google redirect links.  
  - Inputs: RSS items from RSS Read.  
  - Outputs: Filtered and cleaned list of RSS items with `cleanUrl` field added.  
  - Edge cases:  
    - Malformed date fields.  
    - URL decoding errors.  
    - Items with missing or invalid links filtered out.  
  - Logs detailed processing info for debugging.

---

#### 2.2 Content Fetching and Preparation

**Overview:**  
Fetches full HTML content of each filtered news article and processes it to extract a clean textual snippet, preparing a payload for AskRally API analysis.

**Nodes Involved:**  
- Get Content (HTTP Request)  
- Code (named "Code")

**Node Details:**

- **Get Content**  
  - Type: HTTP Request  
  - Role: Requests full article HTML from the clean URL extracted earlier.  
  - Configuration: URL dynamically constructed from item’s `link` or cleaned URL; handles Google redirect URLs by decoding them.  
  - Inputs: Items output by Dedup + url extraction.  
  - Outputs: HTTP response including raw HTML content in `data` or `body`.  
  - Edge cases:  
    - HTTP errors or timeouts.  
    - 404 or inaccessible pages.  
    - Pages with no or minimal content.

- **Code ("Code")**  
  - Type: Code (JavaScript)  
  - Role:  
    - Extracts metadata from RSS and HTTP content.  
    - Cleans HTML tags and entities from the article body.  
    - Limits text snippet length to 3000 characters.  
    - Prepares the AskRally API payload including the cleaned content and voting query.  
  - Key expressions:  
    - Extracts `title`, `snippet`, `articleUrl` from RSS data.  
    - Cleans HTML by regex removing scripts/styles and tags.  
    - Decides whether to use extracted content or fallback snippet based on length threshold (200 chars).  
    - Constructs `rallyPayload` with audience ID, query, voting mode, and memory content.  
  - Inputs: HTTP content from Get Content and RSS metadata from Dedup + url extraction.  
  - Outputs: JSON including cleaned text, metadata, and AskRally payload.  
  - Edge cases:  
    - Missing or empty content.  
    - HTML parsing anomalies.  
    - Expression failures if expected fields are missing.

---

#### 2.3 AskRally API Interaction

**Overview:**  
Submits the prepared content and query to the AskRally API endpoint using an authenticated HTTP request to receive simulated audience voting responses.

**Nodes Involved:**  
- HTTP Request (AskRally POST)

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Sends POST request to AskRally API with the JSON body prepared in the previous step.  
  - Configuration:  
    - URL: Placeholder for Rally API endpoint (to be replaced).  
    - Method: POST  
    - Authentication: HTTP Bearer token (configured via n8n credentials).  
    - Headers: Content-Type application/json.  
    - Body: JSON from previous Code node’s `rallyPayload`.  
  - Inputs: Payload from Code node.  
  - Outputs: JSON response containing audience voting simulations.  
  - Edge cases:  
    - Auth token invalid or expired.  
    - API endpoint unreachable or returns errors.  
    - Network timeouts.  
    - Unexpected API response format.

---

#### 2.4 Response Analysis

**Overview:**  
Parses the AskRally API response to aggregate voting results, calculate narrative percentages, classify narrative type, and select sample voter responses for reporting.

**Nodes Involved:**  
- analyze simulation results (Code)  
- alert trigger (Code)

**Node Details:**

- **analyze simulation results**  
  - Type: Code (JavaScript)  
  - Role:  
    - Parses each persona response from API.  
    - Counts votes per option (A-E).  
    - Calculates vote percentages and narrative aggregates (pro, contra, neutral).  
    - Selects up to 5 sample responses from dominant narrative groups or mixed if no clear majority.  
  - Inputs: Raw API response from HTTP Request node.  
  - Outputs: Structured summary of votes, percentages, narrative type, sample responses, and debug info.  
  - Edge cases:  
    - Parsing JSON response strings fails.  
    - Missing or malformed response data.  
    - No responses returned by API.

- **alert trigger**  
  - Type: Code (JavaScript)  
  - Role:  
    - Further processes summarized vote counts and percentages.  
    - Determines if alert should be sent based on narrative thresholds (≥75% pro or contra).  
    - Prepares message string describing narrative detection.  
    - Filters sample responses according to detected narrative for email content.  
  - Inputs: Output from analyze simulation results.  
  - Outputs: Data including message, notification type, and flag `shouldNotify`.  
  - Edge cases:  
    - No responses or insufficient data to trigger alert.  
    - Logical branch failures in narrative classification.

---

#### 2.5 Alert Generation and Email Sending

**Overview:**  
If a significant narrative shift is detected, sends a styled HTML email via Gmail summarizing the article details, full content, rally results, and sample voter comments.

**Nodes Involved:**  
- Gmail

**Node Details:**

- **Gmail**  
  - Type: Gmail  
  - Role: Sends an alert email to a configured recipient with detailed narrative analysis.  
  - Configuration:  
    - Recipient email to be set manually.  
    - Email subject uses the narrative summary message.  
    - Email body is a rich HTML template displaying article info, full content, rally voting stats, and categorized sample responses with color-coded sections.  
  - Inputs: Data from alert trigger node.  
  - Credentials: OAuth2 Gmail account required.  
  - Edge cases:  
    - Email sending failures (auth issues, quota limits).  
    - Missing or incomplete data causing empty email sections.

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                               | Input Node(s)           | Output Node(s)           | Sticky Note                                                       |
|-----------------------|----------------------------|-----------------------------------------------|-------------------------|--------------------------|------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger           | Starts workflow every 2 hours                  | -                       | RSS Read                 |                                                                  |
| RSS Read              | RSS Feed Read              | Reads RSS feed entries                         | Schedule Trigger        | Dedup + url extraction   | ## Dedup + Extract content from RSS alert                        |
| Dedup + url extraction| Code                       | Filters recent items, dedupes, extracts URLs | RSS Read                | Get Content              | ## Dedup + Extract content from RSS alert                        |
| Get Content           | HTTP Request               | Fetches full article content                   | Dedup + url extraction  | Code                     | ## Dedup + Extract content from RSS alert                        |
| Code                  | Code                       | Cleans HTML, prepares Rally API payload       | Get Content             | HTTP Request             | ## Ask Rally                                                     |
| HTTP Request          | HTTP Request               | Sends content to AskRally API                   | Code                    | analyze simulation results| ## Ask Rally                                                     |
| analyze simulation results | Code                    | Aggregates and interprets AskRally responses  | HTTP Request            | alert trigger            | ## Send an Alert                                                |
| alert trigger         | Code                       | Decides if alert triggers and prepares message| analyze simulation results | Gmail                  | ## Send an Alert                                                |
| Gmail                 | Gmail                      | Sends alert email with narrative summary      | alert trigger           | -                        | ## Send an Alert                                                |
| Sticky Note           | Sticky Note                | Annotates RSS preprocessing                    | -                       | -                        | ## Dedup + Extract content from RSS alert                        |
| Sticky Note4          | Sticky Note                | Annotates AskRally processing                   | -                       | -                        | ## Ask Rally                                                     |
| Sticky Note1          | Sticky Note                | Annotates alert sending                         | -                       | -                        | ## Send an Alert                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to 2 hours.

2. **Create RSS Read node**  
   - Type: RSS Feed Read  
   - URL: Set to your Google Alerts RSS feed URL (e.g. `https://www.google.com/alerts/feeds/...`).  
   - Connect Schedule Trigger → RSS Read.

3. **Create Code node named "Dedup + url extraction"**  
   - Type: Code  
   - Paste the JavaScript code that filters items less than 3 hours old, decodes Google redirect URLs, and outputs cleaned items.  
   - Connect RSS Read → Dedup + url extraction.

4. **Create HTTP Request node named "Get Content"**  
   - Type: HTTP Request  
   - URL: Use an expression to fetch `link` or `cleanUrl` from previous node output, decoding Google redirect URLs if present.  
   - Connect Dedup + url extraction → Get Content.

5. **Create Code node named "Code"**  
   - Type: Code  
   - Paste JavaScript that extracts article metadata from RSS and HTTP content, cleans HTML tags/entities, truncates content, and prepares AskRally API payload.  
   - Connect Get Content → Code.

6. **Create HTTP Request node for AskRally API**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Set to your AskRally API endpoint URL.  
   - Authentication: HTTP Bearer token (create credentials in n8n with your API token).  
   - Headers: Content-Type application/json.  
   - Body: Use JSON mode and set to `{{$json.rallyPayload}}` from previous node.  
   - Connect Code → HTTP Request.

7. **Create Code node named "analyze simulation results"**  
   - Type: Code  
   - Paste JavaScript that parses AskRally responses, counts votes, calculates percentages, categorizes narrative types, and selects sample responses.  
   - Connect HTTP Request → analyze simulation results.

8. **Create Code node named "alert trigger"**  
   - Type: Code  
   - Paste JavaScript that determines if alert should be sent based on narrative thresholds, prepares notification message and sample responses.  
   - Connect analyze simulation results → alert trigger.

9. **Create Gmail node**  
   - Type: Gmail  
   - Credentials: Set up OAuth2 Gmail credentials.  
   - To: Your email address.  
   - Subject: Use expression `{{$json.message}}` from alert trigger node.  
   - Message: Paste the provided rich HTML template, referencing data from alert trigger (`title`, `url`, `fullContent`, `sampleResponses`, etc.) using expressions like  
     `{{ $node["Code"].json.title }}`, `{{$node["Code"].json.url}}`, etc.  
   - Connect alert trigger → Gmail.

10. **Test the workflow end-to-end**  
    - Verify RSS feed is read correctly, content fetched, API requests sent and responses parsed.  
    - Confirm emails received with correct narrative summaries and sample responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| The workflow uses Google Alerts RSS feeds, which may redirect URLs via Google’s redirector. Decoding URLs is essential for accuracy.   | RSS feed URL handling in Dedup + url extraction code.                |
| AskRally API requires an authenticated HTTP Bearer token configured in n8n credentials.                                                | API token setup in HTTP Request node credentials.                    |
| Gmail node uses OAuth2 credentials; ensure your Gmail account is authorized for n8n email sending.                                      | Gmail OAuth2 credential configuration in n8n.                        |
| The email template is a custom HTML design incorporating narrative colors and dynamic sample responses for clear visualization.       | Email body configuration in Gmail node.                              |
| Narrative thresholds (75% for pro or contra) and sample response limits (up to 5) are configurable in the analyze and alert code nodes.| These parameters can be adjusted to tune sensitivity and verbosity.  |
| Logs and debug info are included in code nodes for troubleshooting unexpected data or processing issues.                               | Console logs in Code nodes for debugging during execution.           |

---

**Disclaimer:**  
The provided content is extracted exclusively from an n8n workflow automation. It complies strictly with content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.