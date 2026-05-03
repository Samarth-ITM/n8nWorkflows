Real-time Public Transport Delay Tracking with ScrapeGraphAI, Teams & Dropbox

https://n8nworkflows.xyz/workflows/real-time-public-transport-delay-tracking-with-scrapegraphai--teams---dropbox-11634


# Real-time Public Transport Delay Tracking with ScrapeGraphAI, Teams & Dropbox

### 1. Workflow Overview

This workflow provides a real-time monitoring system for public transport delays by scraping official transit authority websites, analyzing delays, sending alerts to Microsoft Teams, and archiving structured data to Dropbox. It targets transit operations teams or commuters who want timely updates on specific transit lines.

The workflow is organized into these logical blocks:

- **1.1 Trigger & URL Preparation:** Accepts POST requests via webhook specifying transit routes, converts these into official schedule URLs, and splits them for parallel processing.
- **1.2 Scraping Layer:** Uses ScrapeGraphAI to extract timetable and delay information from each URL, then merges the results.
- **1.3 Processing & Alerting:** Normalizes and deduplicates scraped data, evaluates delay severity, and sends alerts to Microsoft Teams if delays exceed a threshold.
- **1.4 Storage & Archiving:** Formats the data into JSON files and archives them to Dropbox for historical analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & URL Preparation

- **Overview:**  
Starts the workflow on a POST webhook call with requested transit routes. It builds a list of URLs for each requested route and splits them for individual scraping.

- **Nodes Involved:**  
  - Webhook Listener  
  - Prepare Source URLs (Code)  
  - Split URLs (Split In Batches)

- **Node Details:**

1. **Webhook Listener**  
   - Type: Webhook  
   - Role: HTTP POST endpoint to receive client requests with desired routes.  
   - Configuration: HTTP POST on path `/public-transit-update`, response mode set to last node (acknowledges request after downstream processing).  
   - Expressions: None.  
   - Inputs: External POST request.  
   - Outputs: Passes JSON body to next node.  
   - Version: 1  
   - Edge Cases: Invalid POST payloads, missing or malformed routes array.  
   - Sub-workflow: None.

2. **Prepare Source URLs**  
   - Type: Code (JavaScript)  
   - Role: Converts requested route names into official MBTA schedule URLs; defaults to RedLine and BlueLine if none given.  
   - Configuration: Maps incoming JSON `routes` array to an array of objects: `{ route, url }`. Uses hard-coded base URLs for RedLine and BlueLine.  
   - Expressions: Uses `$json` to access input data.  
   - Inputs: JSON from webhook.  
   - Outputs: Array of `{route, url}` objects.  
   - Version: 2  
   - Edge Cases: Unknown routes result in empty URLs; no explicit validation or error handling.  
   - Sub-workflow: None.

3. **Split URLs**  
   - Type: SplitInBatches  
   - Role: Divides the array of URLs into individual items to enable parallel scraping.  
   - Configuration: Default options (batch size = 1 implied).  
   - Inputs: Array of route URL objects.  
   - Outputs: Single URL object per execution pass.  
   - Version: 3  
   - Edge Cases: Large number of routes could trigger API rate limits downstream.  
   - Sub-workflow: None.

---

#### 2.2 Scraping Layer

- **Overview:**  
Scrapes each URL with ScrapeGraphAI’s LLM-powered parser to extract next departures, delays, and alerts. Merges all scraped results into one stream.

- **Nodes Involved:**  
  - Scrape Transit Data (ScrapeGraphAI)  
  - Merge Results (Merge)

- **Node Details:**

1. **Scrape Transit Data**  
   - Type: ScrapeGraphAI  
   - Role: Extracts structured JSON of next 5 departures, delay times, and alerts from transit schedule pages.  
   - Configuration: User prompt instructs extraction of `line_name`, `next_departures` (list of time and destination), `max_delay_minutes`, and `alert_text`. URL is dynamic from input JSON.  
   - Expressions: `websiteUrl` set to `={{ $json.url }}`.  
   - Inputs: Single URL object from Split node.  
   - Outputs: Structured JSON with transit data.  
   - Version: 1  
   - Edge Cases: Scraper may fail if page structure changes, or network errors occur; LLM parsing errors possible if page content is ambiguous.  
   - Sub-workflow: None.

2. **Merge Results**  
   - Type: Merge  
   - Role: Combines parallel scraping outputs back into one unified stream.  
   - Configuration: Pass-through mode (simply merges without filtering or joining).  
   - Inputs: Multiple Scrape Transit Data outputs.  
   - Outputs: Single stream of all scraped data.  
   - Version: 2  
   - Edge Cases: None significant; preserves all incoming items.  
   - Sub-workflow: None.

---

#### 2.3 Processing & Alerting

- **Overview:**  
Normalizes and deduplicates the merged data, checks if any delays exceed 10 minutes, sends alerts to Microsoft Teams if so, otherwise bypasses alert but continues to archiving.

- **Nodes Involved:**  
  - Normalize & Deduplicate (Code)  
  - Significant Delay? (IF)  
  - Send Teams Alert (Microsoft Teams)  
  - Prepare File for Dropbox (Set)

- **Node Details:**

1. **Normalize & Deduplicate**  
   - Type: Code (JavaScript)  
   - Role: Standardizes field names (`line_name` vs `route`), adds a timestamp, removes duplicate entries by line name.  
   - Configuration: Iterates all inputs, uses a Set to track seen line names, outputs unique entries with `checked_at` timestamp.  
   - Expressions: Uses `$input.all()` to gather all items.  
   - Inputs: Merged scrape results.  
   - Outputs: Deduplicated array of normalized JSON objects.  
   - Version: 2  
   - Edge Cases: Duplicate line names may be missed if inconsistently named; assumes `line_name` or `route` present.  
   - Sub-workflow: None.

2. **Significant Delay?**  
   - Type: IF  
   - Role: Tests if `max_delay_minutes` exceeds 10 (default threshold).  
   - Configuration: Numeric condition `max_delay_minutes > 10`.  
   - Expressions: `={{ $json.max_delay_minutes }}`  
   - Inputs: Normalized data.  
   - Outputs: TRUE branch to alert, FALSE branch to archiving.  
   - Version: 2  
   - Edge Cases: Missing or non-numeric `max_delay_minutes` fields may cause evaluation errors or skip alerts.  
   - Sub-workflow: None.

3. **Send Teams Alert**  
   - Type: Microsoft Teams  
   - Role: Posts delay alert messages into a specified Teams chat/channel with HTML formatting.  
   - Configuration:  
     - `chatId`: Configured via credential or list mode (empty here to be filled).  
     - Message composes bold alert with line name, delay minutes, and optional alert text.  
   - Expressions: Message uses concatenation: `<b>Transit delay alert:</b> ...`  
   - Inputs: TRUE branch from IF node.  
   - Outputs: Passes output to archiving node.  
   - Version: 2  
   - Edge Cases: Auth failures, invalid chatId, message formatting issues.  
   - Sub-workflow: None.

4. **Prepare File for Dropbox**  
   - Type: Set  
   - Role: Constructs JSON file content and timestamped filename for archiving.  
   - Configuration: Creates properties `fileName` and `fileContent` (content is JSON string of transit data).  
   - Inputs: Both from alert node and FALSE branch of IF node (unified archiving).  
   - Outputs: JSON object ready for upload.  
   - Version: 3  
   - Edge Cases: None significant; assumes correct input data.  
   - Sub-workflow: None.

---

#### 2.4 Storage & Archiving

- **Overview:**  
Uploads the generated JSON file into Dropbox under `/transit-updates` folder for persistent storage and later analysis.

- **Nodes Involved:**  
  - Archive to Dropbox (Dropbox)

- **Node Details:**

1. **Archive to Dropbox**  
   - Type: Dropbox  
   - Role: Uploads JSON file to Dropbox path `/transit-updates/{fileName}` with file content from previous node.  
   - Configuration:  
     - Path assembled from JSON property `fileName`.  
     - File content from `fileContent`.  
     - Creates folder if doesn't exist.  
   - Expressions:  
     - `path` = `={{ '/transit-updates/' + $json.fileName }}`  
     - `fileContent` = `={{ $json.fileContent }}`  
   - Inputs: From Prepare File for Dropbox node.  
   - Outputs: None downstream.  
   - Version: 1  
   - Edge Cases: Network errors, insufficient permissions, rate limits, folder creation failure.  
   - Sub-workflow: None.

---

### 3. Summary Table

| Node Name              | Node Type               | Functional Role                                 | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                          |
|------------------------|-------------------------|------------------------------------------------|-------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Webhook Listener       | Webhook                 | Accepts POST requests with transit routes      | —                       | Prepare Source URLs        | Trigger & URL Builder: Starts automation on POST, acknowledges client, passes body downstream                                         |
| Prepare Source URLs    | Code                    | Builds official transit URLs per requested route | Webhook Listener        | Split URLs                | Trigger & URL Builder: Centralizes URL construction for easy city support                                                             |
| Split URLs            | SplitInBatches           | Splits URL array into single items for parallel scraping | Prepare Source URLs     | Scrape Transit Data        | Trigger & URL Builder: Allows parallel scraping per URL without rate limit issues                                                     |
| Scrape Transit Data    | ScrapeGraphAI            | Extracts structured transit delay data          | Split URLs               | Merge Results             | Scraping Layer: LLM-powered parsing of live timetables and alerts                                                                     |
| Merge Results          | Merge                    | Merges parallel scrape results into one stream  | Scrape Transit Data      | Normalize & Deduplicate   | Scraping Layer: Aggregates multiple URL scrape outputs                                                                                 |
| Normalize & Deduplicate| Code                    | Normalizes field names, timestamps, removes duplicates | Merge Results          | Significant Delay?         | Processing & Alerting: Cleans data for alerting and archiving                                                                          |
| Significant Delay?     | IF                       | Checks if delay exceeds threshold (default 10 min) | Normalize & Deduplicate | Send Teams Alert / Prepare File for Dropbox | Processing & Alerting: Branches alerts for significant delays versus normal processing                                                   |
| Send Teams Alert       | Microsoft Teams          | Posts delay alerts to Teams channel in HTML format | Significant Delay? (TRUE) | Prepare File for Dropbox   | Processing & Alerting: Immediate notification of significant delays                                                                    |
| Prepare File for Dropbox| Set                      | Assembles JSON file content and filename for archiving | Significant Delay? (FALSE), Send Teams Alert | Archive to Dropbox          | Storage & Archiving: Prepares data for persistent storage                                                                              |
| Archive to Dropbox     | Dropbox                  | Uploads JSON file to Dropbox folder               | Prepare File for Dropbox  | —                         | Storage & Archiving: Stores historical transit delay data for analysis                                                                |
| Workflow Overview      | Sticky Note              | Explains overall workflow logic and setup steps  | —                       | —                         | Complete workflow explanation                                                                                                         |
| Section – Trigger & URLs| Sticky Note             | Describes webhook and URL preparation logic      | —                       | —                         | Trigger & URL Builder: Explains initial data reception and URL construction                                                            |
| Section – Scraping Layer| Sticky Note             | Describes scraping logic                          | —                       | —                         | Scraping Layer: Details ScrapeGraphAI usage and merging                                                                                |
| Section – Processing & Alerts| Sticky Note          | Describes normalization, alerting, and branching | —                      | —                         | Processing & Alerting: Details delay evaluation and Teams alert process                                                               |
| Section – Storage & Archiving| Sticky Note          | Describes file preparation and Dropbox archiving | —                       | —                         | Storage & Archiving: Explains data archival strategy                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Listener Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `public-transit-update`  
   - Response Mode: Last Node  
   - Purpose: Receive JSON with `routes` array from client apps.

2. **Add Code Node "Prepare Source URLs"**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const body = $json;
     const routes = body.routes || ['RedLine', 'BlueLine'];
     const baseUrls = {
       RedLine: 'https://www.mbta.com/schedules/Red',
       BlueLine: 'https://www.mbta.com/schedules/Blue'
     };
     return routes.map(route => ({
       json: {
         route,
         url: baseUrls[route] || ''
       }
     }));
     ```  
   - Purpose: Map requested routes to official MBTA URLs.

3. **Add SplitInBatches Node "Split URLs"**  
   - Batch Size: 1 (default)  
   - Purpose: Split URL array for parallel scraping.

4. **Add ScrapeGraphAI Node "Scrape Transit Data"**  
   - Credential: Configure ScrapeGraphAI API credentials.  
   - Website URL: `={{ $json.url }}`  
   - User Prompt:  
     ```
     Extract the next five departures for the requested line along with any reported delay or service alerts. Return JSON with: line_name, next_departures:[{time, destination}], max_delay_minutes (number), alert_text (string).
     ```  
   - Purpose: Extract live transit schedule and delay data.

5. **Add Merge Node "Merge Results"**  
   - Mode: Pass Through  
   - Purpose: Combine parallel scrape results into one stream.

6. **Add Code Node "Normalize & Deduplicate"**  
   - Code:  
     ```javascript
     const items = $input.all();
     const seen = new Set();
     const output = [];
     items.forEach(item => {
       const data = item.json;
       const key = data.line_name || data.route;
       if (!seen.has(key)) {
         seen.add(key);
         output.push({
           json: {
             ...data,
             checked_at: new Date().toISOString()
           }
         });
       }
     });
     return output;
     ```  
   - Purpose: Standardize keys, timestamp, and remove duplicate lines.

7. **Add IF Node "Significant Delay?"**  
   - Condition: Numeric `max_delay_minutes` > 10  
   - Purpose: Determine if alert should be sent.

8. **Add Microsoft Teams Node "Send Teams Alert"**  
   - Credential: Microsoft Teams OAuth2 with appropriate permissions.  
   - Chat ID: Fill with target team/channel ID list.  
   - Message:  
     ```html
     <b>Transit delay alert:</b> {{$json.line_name}} experiencing {{$json.max_delay_minutes}} minute delay.<br/>{{$json.alert_text || ''}}
     ```  
   - Resource: Chat Message  
   - Content Type: HTML  
   - Purpose: Post alert messages to Teams when delay is significant.

9. **Add Set Node "Prepare File for Dropbox"**  
   - Parameters: Define properties:  
     - `fileName`: A timestamped and descriptive filename (implementation detail not in JSON, but usually like `transit_update_YYYYMMDDTHHMMSS.json`).  
     - `fileContent`: JSON string of transit data to archive.  
   - Purpose: Prepare archival JSON file data.

10. **Add Dropbox Node "Archive to Dropbox"**  
    - Credential: Dropbox account with write access.  
    - Path: `=/transit-updates/{{$json.fileName}}`  
    - File Content: `={{ $json.fileContent }}`  
    - Purpose: Upload JSON file for historical record.

11. **Connect Nodes in Sequence:**  
    - Webhook Listener → Prepare Source URLs → Split URLs → Scrape Transit Data → Merge Results → Normalize & Deduplicate → Significant Delay?  
    - Significant Delay? TRUE → Send Teams Alert → Prepare File for Dropbox → Archive to Dropbox  
    - Significant Delay? FALSE → Prepare File for Dropbox → Archive to Dropbox

12. **Add Sticky Notes:**  
    - Add descriptive sticky notes at strategic positions explaining each logical section: Trigger & URLs, Scraping Layer, Processing & Alerts, Storage & Archiving, and Workflow Overview with setup instructions.

13. **Credentials Setup:**  
    - ScrapeGraphAI API credentials: Required for Scrape Transit Data node.  
    - Microsoft Teams OAuth2 credentials: For posting messages.  
    - Dropbox credentials: For file upload.

14. **Adjust Parameters:**  
    - Update `Prepare Source URLs` to reflect any new transit routes or cities.  
    - Modify delay threshold in IF node if needed (default 10 minutes).  
    - Fill Microsoft Teams chat ID with correct channel ID.

15. **Activate Workflow:**  
    - Enable workflow.  
    - Copy webhook URL and configure client apps to POST route requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses ScrapeGraphAI, an LLM-based scraping service, which removes dependence on brittle CSS selectors and adapts to page layout changes.                                                                          | ScrapeGraphAI official documentation                                                            |
| Microsoft Teams node requires OAuth2 credentials and proper permissions to post messages to channels. Ensure your app is authorized.                                                                                           | Microsoft Teams API and OAuth2 setup guides                                                     |
| Dropbox integration requires write permissions and will create the `/transit-updates` folder if absent. JSON files enable easy import into BI tools later for historical delay analysis.                                        | Dropbox API documentation                                                                        |
| The workflow’s webhook enables real-time triggering from mobile apps or other services that specify transit lines of interest.                                                                                                  | n8n Webhook node documentation                                                                  |
| Setup steps are summarized in the Workflow Overview sticky note for quick reference.                                                                                                                                            | Sticky note content in workflow                                                                 |

---

This document fully describes the “Real-time Public Transport Delay Tracking with ScrapeGraphAI, Teams & Dropbox” workflow, enabling understanding, reproduction, and modification with attention to possible edge cases and integration points.