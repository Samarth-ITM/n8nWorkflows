Automated Google Ads Campaign Reporting to Google Sheets with Airtable Client Management

https://n8nworkflows.xyz/workflows/automated-google-ads-campaign-reporting-to-google-sheets-with-airtable-client-management-7387


# Automated Google Ads Campaign Reporting to Google Sheets with Airtable Client Management

### 1. Workflow Overview

This n8n workflow automates monthly reporting of Google Ads campaign performance data into Google Sheets, leveraging Airtable for client and project management. It targets digital marketing agencies or teams managing multiple client Google Ads accounts, enabling efficient, error-resilient, and scheduled budget tracking.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger & Client Data Retrieval**: Automatically starts on the 3rd of each month at 10 AM, fetching active client projects from Airtable.
- **1.2 Client Filtering and Batch Processing**: Filters clients by active status and processes them sequentially in batches to maintain system stability.
- **1.3 Campaign Type Classification and API Querying**: Classifies each client’s Google Ads account as Ecommerce or Lead type, and queries Google Ads API accordingly.
- **1.4 Google Ads Data Processing**: Processes raw API responses, aggregates costs and conversions by campaign type, and formats data.
- **1.5 Dynamic Google Sheets Integration**: Calculates the correct spreadsheet column for the previous month, extracts spreadsheet IDs, and prepares update requests.
- **1.6 Automated Spreadsheet Update and Rate Limiting**: Sends updates to Google Sheets with retry and wait logic to manage API rate limits and ensure data integrity.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Client Data Retrieval

**Overview:**  
This block initiates the workflow monthly, retrieves client projects from Airtable, and filters for active projects to be processed.

**Nodes Involved:**  
- 10 du mois (Schedule Trigger)  
- Récupération info airtable (Airtable)  
- Actif (If)

**Node Details:**  
- **10 du mois**  
  - Type: Schedule Trigger  
  - Role: Fires the workflow on the 3rd day of each month at 10:00 AM (cron expression: `0 10 3 * *`).  
  - Input: None (trigger node)  
  - Output: Trigger event to next node  
  - Edge cases: Scheduling misconfiguration could cause missed executions.

- **Récupération info airtable**  
  - Type: Airtable  
  - Role: Retrieves all records from the "Projets" table in the specified Airtable base.  
  - Config: Uses API token credential; operation is "search" without filters (retrieves all).  
  - Input: Trigger from schedule node  
  - Output: List of client project records  
  - Edge cases: API authentication failures, Airtable service downtime, or API rate limits.

- **Actif**  
  - Type: If  
  - Role: Filters projects for which the field "Status - Prévisionnel budgétaire" equals "Actif".  
  - Input: Airtable data  
  - Output: Passes active projects downstream  
  - Edge cases: Missing or misspelled status field, case sensitivity issues.

---

#### 2.2 Client Filtering and Batch Processing

**Overview:**  
Processes each active client project one by one in batches to avoid system overload and maintain data integrity.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Ecommerce/Lead (Switch)

**Node Details:**  
- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes client records sequentially (defaults to batch size 1).  
  - Input: Active clients from If node  
  - Output: Individual client data items sent downstream  
  - Edge cases: Large client lists may cause long execution times; batch size can be adjusted.

- **Ecommerce/Lead**  
  - Type: Switch  
  - Role: Routes client data based on "Typologie ADS" field: "Ecommerce" or "Lead".  
  - Input: Individual client record from Loop Over Items  
  - Output: Two branches — one for Ecommerce, one for Lead  
  - Edge cases: Typology field missing or misconfigured, unrecognized values.

---

#### 2.3 Campaign Type Classification and API Querying

**Overview:**  
Depending on the client’s campaign typology, this block sends tailored queries to the Google Ads API to retrieve last month’s campaign data.

**Nodes Involved:**  
- Query Ecommerce (HTTP Request)  
- Query Lead (HTTP Request)

**Node Details:**  
- **Query Ecommerce**  
  - Type: HTTP Request  
  - Role: POST query to Google Ads API to fetch cost and conversion value metrics for Ecommerce accounts for the previous month.  
  - Config:  
    - URL dynamically built with client's Google Ads ID  
    - Query selects campaign.advertising_channel_type, metrics.cost_micros, metrics.conversions_value  
    - Headers include Content-Type, developer-token, login-customer-id  
    - Authentication via Google Ads OAuth2 credentials  
  - Input: Ecommerce branch from Switch node  
  - Output: Raw Google Ads API response  
  - Edge cases: API authentication failure, invalid Google Ads ID, quota exceeded, network timeouts.

- **Query Lead**  
  - Type: HTTP Request  
  - Role: Similar to Query Ecommerce but queries for metrics.cost_micros and metrics.conversions (lead count).  
  - Config: Similar to Query Ecommerce but different query fields.  
  - Input: Lead branch from Switch node  
  - Output: Raw API response for lead campaigns  
  - Edge cases: Same as Query Ecommerce.

---

#### 2.4 Google Ads Data Processing

**Overview:**  
Transforms raw Google Ads API responses into structured, aggregated campaign data grouped by advertising channel type with cost and conversion metrics.

**Nodes Involved:**  
- Tri données ecommerce (Code)  
- Tri données lead (Code)

**Node Details:**  
- **Tri données ecommerce**  
  - Type: Code  
  - Role: Parses Ecommerce API data; aggregates cost and conversion value for campaign types: PERFORMANCE_MAX, SEARCH, DISPLAY, VIDEO, DEMAND_GEN, SHOPPING.  
  - Key logic: Converts micros to euros, sums conversions value, rounds to two decimals, returns structured JSON with campaign types.  
  - Input: Google Ads API response from Query Ecommerce  
  - Output: Aggregated campaign data object  
  - Edge cases: Unexpected response structure, missing fields, zero-cost campaigns.

- **Tri données lead**  
  - Type: Code  
  - Role: Similar to ecommerce node, but aggregates cost and lead conversions count instead of conversion value.  
  - Input: Query Lead output  
  - Output: Aggregated structured data  
  - Edge cases: Same as ecommerce node.

---

#### 2.5 Dynamic Google Sheets Integration

**Overview:**  
Determines the correct spreadsheet column based on the previous month, extracts the target Google Sheets spreadsheet ID from the client’s URL, and formats the data payload for API update.

**Nodes Involved:**  
- Formatage requete ecommerce (Code)  
- Formatage requete lead (Code)

**Node Details:**  
- **Formatage requete ecommerce**  
  - Type: Code  
  - Role:  
    - Calculates spreadsheet column for previous month (e.g., B=Jan, C=Feb, etc.)  
    - Extracts spreadsheet ID from client’s Google Sheets URL ("Automation budget" field)  
    - Prepares Google Sheets API URL for range update (rows 2 to 11 in the calculated column)  
    - Constructs JSON body with campaign cost and conversion values in correct order for Ecommerce data  
  - Input: Output from Tri données ecommerce  
  - Output: JSON object with URL, method, body, range, and metadata for Google Sheets update  
  - Edge cases: Missing or malformed spreadsheet URL, inability to extract spreadsheet ID, date calculation errors.

- **Formatage requete lead**  
  - Type: Code  
  - Role: Same as ecommerce formatting node but uses lead campaign data structure.  
  - Input: Output from Tri données lead  
  - Output: JSON with Google Sheets update details  
  - Edge cases: Same as ecommerce formatting node.

---

#### 2.6 Automated Spreadsheet Update and Rate Limiting

**Overview:**  
Sends the update requests to Google Sheets API to write campaign data, with retry logic and wait nodes to manage API quotas and ensure stable processing.

**Nodes Involved:**  
- Remplissage ecommerce (HTTP Request)  
- Wait (Wait)  
- Remplissage lead (HTTP Request)  
- Wait1 (Wait)

**Node Details:**  
- **Remplissage ecommerce**  
  - Type: HTTP Request  
  - Role: Sends PUT request to Google Sheets API to update campaign data for Ecommerce clients.  
  - Config:  
    - URL and body taken from previous formatting node  
    - Uses Google Sheets OAuth2 credential  
    - Retries up to 5 times with 5 seconds wait between tries on failure  
  - Input: Formatage requete ecommerce output  
  - Output: API response confirming update  
  - Edge cases: API quota exceeded, invalid spreadsheet ID, network errors.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow 1 minute after each Ecommerce client update to avoid rate limit errors.  
  - Input: Remplissage ecommerce output  
  - Output: Passes control to next batch iteration  
  - Edge cases: Long workflow duration if many clients.

- **Remplissage lead**  
  - Type: HTTP Request  
  - Role: Same as Remplissage ecommerce but for Lead clients.  
  - Config: Same retry logic and credentials.  
  - Input: Formatage requete lead output  
  - Output: API response for Google Sheets update  
  - Edge cases: Same as Remplissage ecommerce.

- **Wait1**  
  - Type: Wait  
  - Role: Same as Wait node but for Lead client updates.  
  - Input: Remplissage lead output  
  - Output: Passes control to next batch iteration  
  - Edge cases: Same as Wait node.

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                            | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                          |
|-----------------------|----------------------|--------------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------|
| 10 du mois            | Schedule Trigger     | Monthly workflow initiation                 | None                    | Récupération info airtable| 10 du mois                                                                                                           |
| Récupération info airtable | Airtable           | Fetch all client project records             | 10 du mois              | Actif                    | # Phase 2: Client Database Retrieval and Filtering                                                                   |
| Actif                 | If                   | Filter active projects                        | Récupération info airtable | Loop Over Items          | # Phase 2: Client Database Retrieval and Filtering                                                                   |
| Loop Over Items        | SplitInBatches        | Batch processing of active clients           | Actif                   | Ecommerce/Lead           | # Phase 3: Sequential Account Processing                                                                              |
| Ecommerce/Lead         | Switch                | Route per campaign typology                   | Loop Over Items          | Query Ecommerce, Query Lead | # Phase 4: Campaign Type Classification and Data Extraction                                                          |
| Query Ecommerce        | HTTP Request          | Query Google Ads API for Ecommerce data      | Ecommerce/Lead           | Tri données ecommerce     | # Phase 5: Google Ads API Data Retrieval                                                                               |
| Query Lead             | HTTP Request          | Query Google Ads API for Lead data           | Ecommerce/Lead           | Tri données lead          | # Phase 5: Google Ads API Data Retrieval                                                                               |
| Tri données ecommerce  | Code                  | Process and aggregate Ecommerce data         | Query Ecommerce          | Formatage requete ecommerce | # Phase 6: Data Processing and Campaign Aggregation                                                                   |
| Tri données lead       | Code                  | Process and aggregate Lead data              | Query Lead               | Formatage requete lead    | # Phase 6: Data Processing and Campaign Aggregation                                                                   |
| Formatage requete ecommerce | Code              | Prepare Google Sheets API request for Ecommerce | Tri données ecommerce    | Remplissage ecommerce     | # Phase 7: Dynamic Spreadsheet Integration Setup                                                                      |
| Formatage requete lead | Code                  | Prepare Google Sheets API request for Lead   | Tri données lead         | Remplissage lead          | # Phase 7: Dynamic Spreadsheet Integration Setup                                                                      |
| Remplissage ecommerce  | HTTP Request          | Update Google Sheets with Ecommerce data     | Formatage requete ecommerce | Wait                     | # Phase 8: Automated Spreadsheet Updates and Rate Limiting                                                           |
| Wait                  | Wait                  | Pause after Ecommerce update to avoid rate limits | Remplissage ecommerce    | Loop Over Items           | # Phase 8: Automated Spreadsheet Updates and Rate Limiting                                                           |
| Remplissage lead       | HTTP Request          | Update Google Sheets with Lead data          | Formatage requete lead   | Wait1                    | # Phase 8: Automated Spreadsheet Updates and Rate Limiting                                                           |
| Wait1                 | Wait                  | Pause after Lead update to avoid rate limits | Remplissage lead         | Loop Over Items           | # Phase 8: Automated Spreadsheet Updates and Rate Limiting                                                           |
| Sticky Note1           | Sticky Note           | Phase 1: Monthly Schedule Activation         | None                    | None                     | # Phase 1: Monthly Schedule Activation                                                                                |
| Sticky Note3           | Sticky Note           | Explanation of monthly scheduling            | None                    | None                     | Describes automated monthly execution                                                                                 |
| Sticky Note4           | Sticky Note           | Phase 2: Client Database Retrieval and Filtering | None                    | None                     |                                                                                                                      |
| Sticky Note5           | Sticky Note           | Details about Airtable client data retrieval | None                    | None                     |                                                                                                                      |
| Sticky Note6           | Sticky Note           | Phase 3: Sequential Account Processing       | None                    | None                     |                                                                                                                      |
| Sticky Note7           | Sticky Note           | Details about batch processing                | None                    | None                     |                                                                                                                      |
| Sticky Note8           | Sticky Note           | Phase 4: Campaign Type Classification        | None                    | None                     |                                                                                                                      |
| Sticky Note9           | Sticky Note           | Campaign type-specific data extraction details | None                    | None                     |                                                                                                                      |
| Sticky Note10          | Sticky Note           | Phase 5: Google Ads API Data Retrieval       | None                    | None                     |                                                                                                                      |
| Sticky Note11          | Sticky Note           | Google Ads API data retrieval explanation    | None                    | None                     |                                                                                                                      |
| Sticky Note12          | Sticky Note           | Phase 6: Data Processing and Aggregation     | None                    | None                     |                                                                                                                      |
| Sticky Note13          | Sticky Note           | Details on data processing logic              | None                    | None                     |                                                                                                                      |
| Sticky Note14          | Sticky Note           | Phase 7: Dynamic Spreadsheet Integration Setup | None                    | None                     |                                                                                                                      |
| Sticky Note15          | Sticky Note           | Spreadsheet column and ID extraction details | None                    | None                     |                                                                                                                      |
| Sticky Note16          | Sticky Note           | Phase 8: Automated Spreadsheet Updates       | None                    | None                     |                                                                                                                      |
| Sticky Note17          | Sticky Note           | Spreadsheet update retry and rate limiting details | None                    | None                     |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Name: `10 du mois`
   - Type: Schedule Trigger
   - Set cron expression to run at 10:00 AM on the 3rd day of each month (`0 10 3 * *`).
   - Connect output to next node.

2. **Create an Airtable node**
   - Name: `Récupération info airtable`
   - Credentials: Airtable Personal Access Token
   - Operation: Search (retrieve all records)
   - Base: Select your Airtable base containing client projects.
   - Table: Select the table with project records.
   - Connect input from `10 du mois`.
   - Connect output to next node.

3. **Create an If node**
   - Name: `Actif`
   - Set condition: Check if field `Status - Prévisionnel budgétaire` equals the string "Actif".
   - Input: Airtable node output
   - True output connects to batch processing node.

4. **Create a SplitInBatches node**
   - Name: `Loop Over Items`
   - Default batch size (1)
   - Input: If node's True output
   - Output: Connect to Switch node.

5. **Create a Switch node**
   - Name: `Ecommerce/Lead`
   - Set rules based on field `Typologie ADS`:
     - If equals "Ecommerce" → output key "Ecommerce"
     - If equals "Lead" → output key "Lead"
   - Input: SplitInBatches output
   - Output: Two branches, one for Ecommerce, one for Lead.

6. **Create HTTP Request nodes for Google Ads queries**

   - **For Ecommerce:**
     - Name: `Query Ecommerce`
     - Method: POST
     - URL: Use expression to build URL with client Google Ads ID `https://googleads.googleapis.com/v21/customers/{{ $json['ID GADS'] }}/googleAds:searchStream`
     - Headers: Content-Type: application/json, developer-token, login-customer-id (set your tokens)
     - Body: JSON query selecting `campaign.advertising_channel_type, metrics.cost_micros, metrics.conversions_value` for last month where cost > 0
     - Authentication: Google Ads OAuth2 credential
     - Input: Ecommerce output from Switch

   - **For Lead:**
     - Name: `Query Lead`
     - Same as Ecommerce node but query selects `campaign.advertising_channel_type, metrics.cost_micros, metrics.conversions`

7. **Create Code nodes to process API responses**

   - **For Ecommerce:**
     - Name: `Tri données ecommerce`
     - Input: Output from `Query Ecommerce`
     - Paste provided JavaScript code that:
       - Checks response structure
       - Aggregates cost and conversion value by campaign type
       - Converts micros to euros
       - Outputs structured JSON with campaign types

   - **For Lead:**
     - Name: `Tri données lead`
     - Input: Output from `Query Lead`
     - Paste provided JavaScript code that aggregates cost and conversions count similarly.

8. **Create Code nodes to format Google Sheets update requests**

   - **For Ecommerce:**
     - Name: `Formatage requete ecommerce`
     - Input: Output from `Tri données ecommerce`
     - Paste provided JavaScript code that:
       - Calculates previous month column
       - Extracts spreadsheet ID from URL in `Automation budget` field
       - Constructs Google Sheets API URL and JSON body with campaign data

   - **For Lead:**
     - Name: `Formatage requete lead`
     - Input: Output from `Tri données lead`
     - Similar code structure to ecommerce formatter.

9. **Create HTTP Request nodes to update Google Sheets**

   - **For Ecommerce:**
     - Name: `Remplissage ecommerce`
     - Method: PUT
     - URL and body from previous formatter node (`Formatage requete ecommerce`)
     - Headers: Content-Type: application/json
     - Authentication: Google Sheets OAuth2 credential
     - Set retry on fail: 5 retries, 5 seconds wait between retries
     - Input: Output from `Formatage requete ecommerce`

   - **For Lead:**
     - Name: `Remplissage lead`
     - Same configuration as above but input from `Formatage requete lead`

10. **Create Wait nodes to avoid rate limits**

    - **For Ecommerce:**
      - Name: `Wait`
      - Wait 1 minute after each Google Sheets update
      - Input: Output from `Remplissage ecommerce`
      - Output: Connect back to `Loop Over Items` for next batch

    - **For Lead:**
      - Name: `Wait1`
      - Same as above but connected to `Remplissage lead` output

11. **Connect nodes according to the sequence:**

    - `10 du mois` → `Récupération info airtable` → `Actif` → `Loop Over Items`  
    - `Loop Over Items` → `Ecommerce/Lead`  
    - `Ecommerce/Lead` → `Query Ecommerce` → `Tri données ecommerce` → `Formatage requete ecommerce` → `Remplissage ecommerce` → `Wait` → back to `Loop Over Items`  
    - `Ecommerce/Lead` → `Query Lead` → `Tri données lead` → `Formatage requete lead` → `Remplissage lead` → `Wait1` → back to `Loop Over Items`

12. **Credential Setup:**

    - Airtable: Personal Access Token with read access to client base  
    - Google Ads: OAuth2 credentials with appropriate scopes and tokens for API access  
    - Google Sheets: OAuth2 credentials with edit permissions on client spreadsheets

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| The workflow is designed to run automatically monthly, eliminating manual data entry and ensuring consistent reporting for client Google Ads budgets.                                  | Scheduling node configuration                                                                                       |
| Batch processing with Wait nodes prevents Google Sheets API rate limit errors or quota exceeded issues, maintaining stable operation even with many clients.                            | Rate limiting strategy                                                                                              |
| Campaign metrics differ by client typology: Ecommerce accounts track conversion value, Lead accounts track conversion counts. This distinction is critical for accurate reporting.     | Campaign classification and data processing logic                                                                   |
| Airtable serves as the single source of truth for client configuration, including active status, Google Ads IDs, campaign typology, and target Google Sheets URLs.                      | Client management and filtering                                                                                      |
| Google Sheets columns are dynamically calculated based on previous month, supporting year-end transitions and avoiding manual column updates.                                         | Spreadsheet integration logic                                                                                        |
| Retry logic in HTTP nodes ensures resilience against transient API errors, improving workflow reliability.                                                                              | HTTP Request node configuration                                                                                      |
| For further customization, adjust the batch size in `Loop Over Items` and monitor API quotas on Google Ads and Google Sheets to optimize throughput and avoid throttling.               | Workflow scalability considerations                                                                                  |
| Useful for agencies or marketers managing multiple client accounts requiring regular budget tracking automation.                                                                          | Target use case                                                                                                      |
| Google Ads API developer tokens and customer IDs must be kept secure and updated in credentials to maintain uninterrupted access.                                                        | Credential security best practices                                                                                   |

---

**Disclaimer:**  
The provided content is exclusively generated based on an automated n8n workflow designed for legal, public data processing. It complies strictly with applicable content policies and does not include illegal or protected information.