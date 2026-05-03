Track Play Store App Rankings with SerpApi, Baserow & Slack Alerts

https://n8nworkflows.xyz/workflows/track-play-store-app-rankings-with-serpapi--baserow---slack-alerts-8036


# Track Play Store App Rankings with SerpApi, Baserow & Slack Alerts

### 1. Workflow Overview

This workflow automates the tracking of an app's ranking on the Google Play Store based on a set of keywords stored in a Baserow database. It periodically fetches keywords, queries the Play Store ranking via SerpApi, updates rankings back in Baserow, and sends Slack alerts to notify stakeholders of ranking updates.  
The logical blocks are:

- **1.1 Scheduled Trigger & Keyword Fetching:** Periodically start the workflow and retrieve keywords from Baserow.  
- **1.2 Query Preparation:** Prepare URLs and data needed to query SerpApi for Play Store rankings.  
- **1.3 Context Preservation:** Maintain relevant data context through workflow steps.  
- **1.4 Play Store Search via SerpApi:** Perform HTTP requests to SerpApi with search queries.  
- **1.5 Data Merging & Rank Extraction:** Combine API results with context and extract app rank.  
- **1.6 Update Baserow & Slack Notification:** Update the database with new rankings and notify via Slack.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger & Keyword Fetching

**Overview:**  
This block initiates the workflow on a weekly schedule and fetches all relevant keywords from a Baserow database table for rank tracking.

**Nodes Involved:**  
- Schedule Trigger  
- Fetch Keywords from Baserow

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow execution weekly (interval field set to weeks).  
  - Configuration: Runs once per week, exact day/time configured in n8n UI (not shown in JSON).  
  - Inputs: None (trigger node)  
  - Outputs: Triggers next node (Fetch Keywords from Baserow)  
  - Edge Cases: Workflow will not run if the trigger is disabled or misconfigured.

- **Fetch Keywords from Baserow**  
  - Type: Baserow node  
  - Role: Retrieves all rows (keywords) from a specific Baserow database table (ID: 568021).  
  - Configuration: Returns all rows, uses credential named "Arunava" for Baserow API access.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Array of keyword records to next node (Prepare Search Queries)  
  - Edge Cases: Possible failures include API authentication errors, rate limits, or empty data if no rows exist.

---

#### 2.2 Query Preparation

**Overview:**  
This block constructs SerpApi search URLs per keyword, embedding API keys, fixed app package name, and query parameters.

**Nodes Involved:**  
- Prepare Search Queries

**Node Details:**  

- **Prepare Search Queries**  
  - Type: Code node (JavaScript)  
  - Role: For each keyword item, prepares a URL to query SerpApi with proper parameters for Play Store app search.  
  - Configuration:  
    - Hardcoded `apiKey` for SerpApi (string placeholder `{api_key}` to be replaced).  
    - Fixed app package ID: `"practice.test.daily.testline"` (to be adapted).  
    - Builds search URL including keyword, API key, geo (`gl=in`), language (`hl=en`), and store (`apps`).  
  - Key Expressions: Uses `items.map()` to transform input array, preserving all Baserow fields and adding new `serpapi_url`.  
  - Inputs: Rows from Baserow (keywords)  
  - Outputs: Each item enriched with SerpApi URL and context fields to next node (Keep Context)  
  - Edge Cases:  
    - If input data is missing keywords or IDs, URLs might be malformed.  
    - Hardcoded API key and package ID require secure management and updating.  
    - No error handling in code, so malformed inputs might cause failures.

---

#### 2.3 Context Preservation

**Overview:**  
This block sets and preserves the row context (IDs, previous and current ranks, keywords) for downstream processing and updates.

**Nodes Involved:**  
- Keep Context

**Node Details:**  

- **Keep Context**  
  - Type: Set node  
  - Role: Extracts and assigns specific fields to keep track of app ranking context (id, Keywords, currentRank, Previous Rank).  
  - Configuration: Assigns values from input JSON directly to new named fields for clarity and downstream use.  
  - Inputs: Prepared search query items from previous step  
  - Outputs: Passes enriched items to Search PlayStore via SerpAPI and Merge Data nodes  
  - Edge Cases: Missing fields in input JSON could lead to undefined values in context.

---

#### 2.4 Play Store Search via SerpApi

**Overview:**  
This block queries the SerpApi Play Store endpoint to fetch organic search results for each keyword.

**Nodes Involved:**  
- Search PlayStore via SerpAPI

**Node Details:**  

- **Search PlayStore via SerpAPI**  
  - Type: HTTP Request node  
  - Role: Sends a GET request to SerpApi's Google Play search API endpoint with query parameters including keyword, geo, language, and API key.  
  - Configuration:  
    - URL: `https://serpapi.com/search.json`  
    - Query parameters dynamically set: engine=google_play, q=keyword, api_key (from credentials), gl=in, hl=en, store=apps.  
    - Always outputs data to ensure downstream nodes receive output even if empty.  
  - Inputs: Items with context and keywords (from Keep Context)  
  - Outputs: API response JSON to Merge Data node  
  - Version: Uses n8n version 4.2 HTTP request node features.  
  - Edge Cases:  
    - API key invalid/missing leads to auth errors.  
    - Rate limiting or network errors.  
    - Empty or malformed JSON responses.

---

#### 2.5 Data Merging & Rank Extraction

**Overview:**  
Merges the original context with API response data, then extracts the app's ranking position based on package ID.

**Nodes Involved:**  
- Merge Data  
- Get App Rank

**Node Details:**  

- **Merge Data**  
  - Type: Merge node  
  - Role: Combines original context (from Keep Context) with SerpApi search results by position to align data per keyword.  
  - Configuration: Mode "combine" by position (index in the array).  
  - Inputs:  
    - First input: Context items (Keep Context output)  
    - Second input: SerpApi responses (Search PlayStore via SerpAPI output)  
  - Outputs: Merged items containing full context and API data to Get App Rank node  
  - Edge Cases: Mismatch in array lengths could cause incorrect merging.

- **Get App Rank**  
  - Type: Code node (JavaScript)  
  - Role: Parses SerpApi JSON results to find the position (rank) of the fixed app package ID in organic results.  
  - Configuration:  
    - Hardcoded app package ID placeholder `{app_package_name}` to be replaced.  
    - Iterates over organic results, increments position counter, and identifies rank if app matches.  
    - If not found, sets rank to 0 (indicates no rank).  
    - Returns JSON with id, keyword, currentRank (new), and previousRank (old).  
  - Inputs: Merged data from Merge Data node  
  - Outputs: App rank data to Baserow update node  
  - Edge Cases:  
    - If SerpApi response structure changes, parsing might fail.  
    - Null or missing organic results handled by default empty array.  
    - Hardcoded package ID must match exactly the app searched.

---

#### 2.6 Update Baserow & Slack Notification

**Overview:**  
Updates the app rank fields in Baserow and sends a Slack alert notifying about the latest ranking update.

**Nodes Involved:**  
- Update Rank in Baserow  
- Slack

**Node Details:**  

- **Update Rank in Baserow**  
  - Type: Baserow node  
  - Role: Updates specific fields in a Baserow row with newly calculated ranks.  
  - Configuration:  
    - Uses row ID from JSON (`$json.id`).  
    - Updates fields 4563204 with currentRank and 4563209 with previousRank.  
    - Uses credential "Arunava" for API access.  
    - Operates on database ID 239215 and table ID 568021.  
  - Inputs: Rank data from Get App Rank  
  - Outputs: Passes to Slack node  
  - Edge Cases: Possible API errors, invalid row IDs, or permission issues.

- **Slack**  
  - Type: Slack node  
  - Role: Sends a notification message to a specified Slack channel.  
  - Configuration:  
    - Sends a fixed text message announcing updated app rankings with a link to the Baserow public grid.  
    - Uses Slack API credentials "Slack account".  
    - Channel specified by URL `https://app.slack.com/client/T053HKKDY12/C08AGRQ9GBF`.  
  - Inputs: From Baserow update node  
  - Outputs: Terminal node (no further outputs)  
  - Edge Cases: Slack API authentication errors or network issues.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                    | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                                 |
|---------------------------|---------------------|----------------------------------|--------------------------|-------------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger    | Start workflow weekly            | —                        | Fetch Keywords from Baserow    |                                                                                             |
| Fetch Keywords from Baserow| Baserow             | Retrieve keywords from Baserow   | Schedule Trigger          | Prepare Search Queries         | ## Fetch Keywords from Baserow                                                              |
| Prepare Search Queries    | Code                | Build SerpApi search URLs        | Fetch Keywords from Baserow| Keep Context                  | ## Prepare Search Queries                                                                   |
| Keep Context             | Set                  | Preserve row context             | Prepare Search Queries    | Search PlayStore via SerpAPI, Merge Data | ## Store Row Context                                                                        |
| Search PlayStore via SerpAPI| HTTP Request        | Query SerpApi Play Store search  | Keep Context             | Merge Data                    | ## Search Play Store via SerpApi                                                            |
| Merge Data               | Merge                 | Combine context and API results  | Keep Context, Search PlayStore via SerpAPI | Get App Rank         | ## Merge Results + Context                                                                  |
| Get App Rank             | Code                  | Extract app rank from results    | Merge Data               | Update Rank in Baserow         | ## Extract App Rank                                                                         |
| Update Rank in Baserow   | Baserow                | Update app rank in Baserow       | Get App Rank             | Slack                         | ## Update Rank in Baserow                                                                   |
| Slack                    | Slack                  | Send Slack alert notification    | Update Rank in Baserow   | —                             | ## Send Slack Alert                                                                         |
| Sticky Note33            | Sticky Note            | Visual comment                   | —                        | —                             | ## Check for Rank Updates                                                                   |
| Sticky Note16            | Sticky Note            | Visual comment                   | —                        | —                             | ## Fetch Keywords from Baserow                                                              |
| Sticky Note17            | Sticky Note            | Visual comment                   | —                        | —                             | ## Prepare Search Queries                                                                   |
| Sticky Note18            | Sticky Note            | Visual comment                   | —                        | —                             | ## Store Row Context                                                                        |
| Sticky Note19            | Sticky Note            | Visual comment                   | —                        | —                             | ## Search Play Store via SerpApi                                                            |
| Sticky Note34            | Sticky Note            | Visual comment                   | —                        | —                             | ## Merge Results + Context                                                                  |
| Sticky Note35            | Sticky Note            | Visual comment                   | —                        | —                             | ## Extract App Rank                                                                         |
| Sticky Note36            | Sticky Note            | Visual comment                   | —                        | —                             | ## Update Rank in Baserow                                                                   |
| Sticky Note37            | Sticky Note            | Visual comment                   | —                        | —                             | ## Send Slack Alert                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to run weekly (set interval to weeks).  

2. **Create Baserow node to fetch keywords:**  
   - Type: Baserow  
   - Set database ID to `239215`, table ID to `568021`.  
   - Enable "Return All" to fetch all rows.  
   - Connect Schedule Trigger → Fetch Keywords from Baserow.  
   - Add Baserow credential named (e.g.) "Arunava" with appropriate API key.  

3. **Create Code node to prepare search queries:**  
   - Type: Code (JavaScript)  
   - Paste code that:  
     - Defines SerpApi API key (replace `{api_key}` with real key).  
     - Uses fixed app package ID (e.g., `"practice.test.daily.testline"`).  
     - Constructs SerpApi query URLs per keyword.  
     - Preserves all Baserow fields.  
   - Connect Fetch Keywords from Baserow → Prepare Search Queries.  

4. **Create Set node to keep context:**  
   - Type: Set  
   - Assign fields:  
     - id: `{{$json.id}}` (row ID)  
     - Keywords: `{{$json.Keywords}}`  
     - currentRank: `{{$json.currentRank}}`  
     - Previous Rank: `{{$json["Previous Rank"]}}`  
   - Connect Prepare Search Queries → Keep Context.  

5. **Create HTTP Request node to query SerpApi:**  
   - Type: HTTP Request  
   - URL: `https://serpapi.com/search.json`  
   - Method: GET  
   - Query Parameters:  
     - engine: "google_play"  
     - q: `{{$json.Keywords}}`  
     - api_key: (use credential or static value)  
     - gl: "in"  
     - hl: "en"  
     - store: "apps"  
   - Use proper credential for SerpApi API key if possible.  
   - Connect Keep Context → Search PlayStore via SerpAPI.  

6. **Create Merge node:**  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Position  
   - Connect Keep Context → Merge (input 1)  
   - Connect Search PlayStore via SerpAPI → Merge (input 2)  

7. **Create Code node to extract rank:**  
   - Type: Code (JavaScript)  
   - Logic:  
     - Hardcode app package name (replace `{app_package_name}` with your app’s package).  
     - Parse SerpApi `organic_results`, iterate over app items, find position of package ID.  
     - Return JSON with id, keyword, currentRank, previousRank.  
   - Connect Merge → Get App Rank.  

8. **Create Baserow node to update rank:**  
   - Type: Baserow  
   - Operation: Update  
   - Database ID: `239215`  
   - Table ID: `568021`  
   - Row ID: `{{$json.id}}`  
   - Fields to update:  
     - Field ID `4563204`: `{{$json.currentRank}}`  
     - Field ID `4563209`: `{{$json.previousRank}}`  
   - Connect Get App Rank → Update Rank in Baserow.  
   - Use same Baserow credential as before.  

9. **Create Slack node for notification:**  
   - Type: Slack  
   - Set channel by URL: `https://app.slack.com/client/T053HKKDY12/C08AGRQ9GBF`  
   - Message text:  
     ```
     🚀 RPSC App Rankings Just Updated!
     See how we're performing 👉 https://baserow.io/public/grid/mjkYq2wWb6vIE3MsB1RannZgclvdHKacNE3zEVrC5Bo
     ```  
   - Use Slack API credential with necessary permissions.  
   - Connect Update Rank in Baserow → Slack.  

10. **Add Sticky Notes for documentation (optional):**  
    - Add notes summarizing each block as per the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow tags include "ASO" indicating its use for App Store Optimization monitoring.      | Tag in workflow metadata.                                                                                |
| Public Baserow grid link in Slack message: https://baserow.io/public/grid/mjkYq2wWb6vIE3MsB1RannZgclvdHKacNE3zEVrC5Bo | Useful for stakeholders to view live ranking data.                                                     |
| SerpApi documentation for Google Play Store queries: https://serpapi.com/play-store-api         | For understanding and customizing API queries.                                                          |
| Ensure API keys and package IDs are securely managed and updated in code nodes and HTTP requests.| Critical for security and correctness.                                                                   |
| Slack bot requires chat:write permissions to post messages in the specified channel.             | Credential setup detail.                                                                                  |

---

**Disclaimer:** The provided content is generated exclusively from an n8n automation workflow JSON. It adheres to current content policies, contains no illegal or offensive material, and operates on publicly accessible data.