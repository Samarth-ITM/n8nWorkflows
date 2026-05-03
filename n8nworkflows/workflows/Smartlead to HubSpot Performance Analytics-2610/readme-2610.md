Smartlead to HubSpot Performance Analytics

https://n8nworkflows.xyz/workflows/smartlead-to-hubspot-performance-analytics-2610


# Smartlead to HubSpot Performance Analytics

### 1. Workflow Overview

This workflow, titled **"Smartlead to HubSpot Performance Analytics,"** automates the process of tracking and analyzing outreach campaign performance by integrating data from Smartlead and HubSpot, storing it in PostgreSQL, and generating reports in Google Sheets.

**Target Use Cases:**  
Outbound automation agencies, sales, and marketing teams using Smartlead for outreach campaigns who want to track lead progression and campaign effectiveness via HubSpot lifecycle stages without manual data consolidation.

**Key Logical Blocks:**

- **1.1 Schedule and Initialization**  
  Initiates the workflow at a scheduled interval and sets the Smartlead API key.

- **1.2 Campaign Data Extraction and Update**  
  Fetches all Smartlead campaigns, extracts detailed lead data per campaign, parses CSV data into structured JSON, and upserts campaign and activity data into PostgreSQL.

- **1.3 HubSpot Lifecycle Enrichment**  
  Queries PostgreSQL for campaign activities needing lifecycle updates, calls HubSpot API to retrieve lifecycle stages, and updates PostgreSQL with HubSpot lifecycle data.

- **1.4 Aggregation and Reporting**  
  Aggregates lifecycle and campaign data from PostgreSQL, then updates or appends this data into a Google Sheets report for visualization.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Initialization

- **Overview:**  
  This block triggers the workflow on a scheduled basis and initializes the Smartlead API key for subsequent requests.

- **Nodes Involved:**  
  - Schedule Trigger  
  - SET SMARTLEAD API KEY

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution based on time intervals (default is every minute but configurable).  
    - Config: Uses interval scheduling with no advanced filters.  
    - Inputs: None (trigger node)  
    - Outputs: Triggers the "SET SMARTLEAD API KEY" node.  
    - Potential Failures: None typical; workflow won’t start if scheduling misconfigured.

  - **SET SMARTLEAD API KEY**  
    - Type: Set  
    - Role: Sets a workflow variable named `API KEY` containing the Smartlead API key string.  
    - Config: Static string assignment; user must replace placeholder with actual API key.  
    - Inputs: Trigger from Schedule Trigger node.  
    - Outputs: Three parallel outputs to FETCH ALL CAMPAIGNS, Postgres (analytics), and SEARCH nodes.  
    - Potential Failures: Misconfigured or missing API key will cause subsequent HTTP requests to Smartlead to fail authentication.

---

#### 2.2 Campaign Data Extraction and Update

- **Overview:**  
  This block retrieves all campaigns from Smartlead, loops through each campaign to extract detailed lead data, parses CSV data into structured JSON, and inserts or updates campaign and campaign activity data in PostgreSQL.

- **Nodes Involved:**  
  - FETCH ALL CAMPAIGNS  
  - Loop Over Items  
  - EXTRACT CAMPAIGN DATA  
  - Code (CSV parsing)  
  - UPSERT CAMPAIGN ACTIVITY  
  - UPDATE CAMPAIGN  
  - Merge

- **Node Details:**  

  - **FETCH ALL CAMPAIGNS**  
    - Type: HTTP Request  
    - Role: Retrieves all campaigns from Smartlead API using the stored API key query parameter.  
    - Config: GET request to `https://server.smartlead.ai/api/v1/campaigns` with API key as query parameter.  
    - Inputs: API key from SET SMARTLEAD API KEY node.  
    - Outputs: JSON array of campaigns.  
    - Potential Failures: API key invalid, network issues, rate limiting.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each campaign item to process individually.  
    - Config: Batch size set to 1 (default behavior).  
    - Inputs: Output array from FETCH ALL CAMPAIGNS.  
    - Outputs: Each campaign item to EXTRACT CAMPAIGN DATA and UPDATE CAMPAIGN nodes.  
    - Potential Failures: Context reset misconfiguration may cause infinite loops or skips.

  - **EXTRACT CAMPAIGN DATA**  
    - Type: HTTP Request  
    - Role: For each campaign, requests detailed lead export data as CSV from Smartlead API.  
    - Config: Uses dynamic URL with campaign ID; API key passed as query parameter.  
    - Inputs: Campaign id and API key from Loop Over Items.  
    - Outputs: CSV text data under `data` field.  
    - Potential Failures: API key invalid, campaign id invalid, network errors.

  - **Code (CSV parsing)**  
    - Type: Code (JavaScript)  
    - Role: Parses CSV string from EXTRACT CAMPAIGN DATA into structured JSON objects. Handles empty and missing fields robustly.  
    - Config: Custom JavaScript with detailed parsing logic expecting 22 columns. Converts booleans and numbers appropriately.  
    - Inputs: Raw CSV string from EXTRACT CAMPAIGN DATA node.  
    - Outputs: Array of JSON objects representing campaign leads.  
    - Potential Failures: Unexpected CSV format, missing data fields, parsing errors.

  - **UPSERT CAMPAIGN ACTIVITY**  
    - Type: PostgreSQL  
    - Role: Upserts each parsed campaign lead data into the `ce_campaign_activity` table in PostgreSQL.  
    - Config: Mapping JSON fields to database columns with type conversions; matches on primary key `id`.  
    - Inputs: Parsed lead JSON from Code node. Also uses campaign id from Loop Over Items.  
    - Outputs: Upsert result, merged downstream.  
    - Potential Failures: DB connection issues, data type mismatches, constraint violations.

  - **UPDATE CAMPAIGN**  
    - Type: PostgreSQL  
    - Role: Upserts campaign metadata into `ce_campaign` table with latest fields and timestamps.  
    - Config: Maps campaign fields (name, status, user_id, timestamps, settings) with match on `campaign_id`.  
    - Inputs: Campaign JSON from Loop Over Items.  
    - Outputs: Result merged downstream.  
    - Potential Failures: DB connection, data mismatches.

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from UPSERT CAMPAIGN ACTIVITY and UPDATE CAMPAIGN nodes to synchronize batch processing.  
    - Config: Combines by position (index order).  
    - Inputs: Outputs from UPSERT CAMPAIGN ACTIVITY and UPDATE CAMPAIGN.  
    - Outputs: Merged results to Loop Over Items for next batch iteration.  
    - Potential Failures: Mismatched array lengths or data inconsistency.

---

#### 2.3 HubSpot Lifecycle Enrichment

- **Overview:**  
  This block identifies campaign activities requiring lifecycle stage updates, queries HubSpot for contact lifecycle stages via OAuth2, and updates the PostgreSQL `hubspot` and `ce_campaign_activity` tables accordingly.

- **Nodes Involved:**  
  - SEARCH (PostgreSQL Query)  
  - Loop Over Items1  
  - HubSpot (API)  
  - If  
  - HUBSPOT TABLE (PostgreSQL Upsert)  
  - UPDATE HUBSPOT ACTIVITY TABLE (PostgreSQL Update)  
  - Postgres1 (PostgreSQL Update)

- **Node Details:**  

  - **SEARCH**  
    - Type: PostgreSQL  
    - Role: Executes a query to retrieve campaign activity records missing recent lifecycle stage checks or older than 24 hours.  
    - Config: Complex SQL joining `ce_campaign_activity` and `ce_campaign` filtering on lifecycle timestamp. Limits to 5000 records.  
    - Inputs: API key triggered flow from SET SMARTLEAD API KEY.  
    - Outputs: Campaign activity records to Loop Over Items1.  
    - Potential Failures: DB connection, query timeouts, large result sets.

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Processes each campaign activity record individually for HubSpot enrichment.  
    - Config: Default batch processing, resets context after completion.  
    - Inputs: Output records from SEARCH.  
    - Outputs: Each record to HubSpot node.  
    - Potential Failures: Batch reset misconfiguration.

  - **HubSpot**  
    - Type: HubSpot  
    - Role: Retrieves all contact properties for the current item via HubSpot API using OAuth2 credentials.  
    - Config: Operation: `getAll` contacts; authentication via connected OAuth2 credential.  
    - Inputs: Current campaign activity item.  
    - Outputs: HubSpot contact data for conditional processing.  
    - Potential Failures: OAuth token expiration, API rate limits, network issues.

  - **If**  
    - Type: If  
    - Role: Checks if the HubSpot response contains a valid companyId property to proceed with database update or skip.  
    - Config: Boolean condition checking if `companyId` is empty or not.  
    - Inputs: HubSpot API output.  
    - Outputs: True branch to HUBSPOT TABLE; False branch to Postgres1 (update timestamp only).  
    - Potential Failures: Expression evaluation errors.

  - **HUBSPOT TABLE**  
    - Type: PostgreSQL (Upsert)  
    - Role: Inserts or updates HubSpot lifecycle stage and deal count for the contact related to the campaign in the `hubspot` table.  
    - Config: Upserts on `hubspot_company_id` with lifecycle stage, open deals, campaign id.  
    - Inputs: HubSpot API data, Loop Over Items1 campaign id.  
    - Outputs: Updates downstream.  
    - Potential Failures: DB constraints, type mismatch.

  - **UPDATE HUBSPOT ACTIVITY TABLE**  
    - Type: PostgreSQL (Update)  
    - Role: Updates the `hb_lifecyclestage_check_timestamp` field in `ce_campaign_activity` to current time for the updated records.  
    - Config: Update based on `id` matching current item id.  
    - Inputs: Loop Over Items1 campaign activity id.  
    - Outputs: Loop Over Items1 for next batch iteration.  
    - Potential Failures: DB connection issues.

  - **Postgres1**  
    - Type: PostgreSQL (Update)  
    - Role: Updates `hb_lifecyclestage_check_timestamp` for records where no HubSpot company id was found, marking them as checked.  
    - Config: Update by `id`.  
    - Inputs: False branch from If node.  
    - Outputs: Loop Over Items1 for next batch iteration.  
    - Potential Failures: DB issues.

---

#### 2.4 Aggregation and Reporting

- **Overview:**  
  This block aggregates campaign and lifecycle stage data from PostgreSQL, then appends or updates this data into a Google Sheet for reporting purposes.

- **Nodes Involved:**  
  - Postgres (Aggregation Query)  
  - Google Sheets

- **Node Details:**  

  - **Postgres**  
    - Type: PostgreSQL (Execute Query)  
    - Role: Runs an aggregation SQL query to count companies, leads, MQL, SQL, opportunities, customers, and specific lifecycle stages grouped by campaign.  
    - Config: Joins `hubspot` and `ce_campaign` tables, groups by campaign id, filters for relevant records.  
    - Inputs: Triggered after SET SMARTLEAD API KEY node.  
    - Outputs: Aggregated performance data.  
    - Potential Failures: Long query execution times, DB connection errors.

  - **Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends or updates the aggregated campaign performance data into a specified Google Sheets document and sheet.  
    - Config: Maps all aggregated fields (campaign id, counts, statuses) to sheet columns; uses OAuth2 credentials.  
    - Inputs: Aggregated data from Postgres node.  
    - Outputs: None (end node).  
    - Potential Failures: Google API quota limits, credential expiration, sheet permissions.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                                   | Input Node(s)                    | Output Node(s)                          | Sticky Note                                                                                                              |
|-------------------------|----------------------|-------------------------------------------------|---------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger     | Starts workflow on schedule                      | None                            | SET SMARTLEAD API KEY                  |                                                                                                                          |
| SET SMARTLEAD API KEY   | Set                  | Initializes Smartlead API key                     | Schedule Trigger                | FETCH ALL CAMPAIGNS, Postgres, SEARCH | See note: "Search for your smartlead API key [here](https://app.smartlead.ai/app/settings/profile)"                      |
| FETCH ALL CAMPAIGNS     | HTTP Request         | Retrieves all Smartlead campaigns                 | SET SMARTLEAD API KEY           | Loop Over Items                       |                                                                                                                          |
| Loop Over Items         | SplitInBatches       | Iterates campaigns for detailed extraction       | FETCH ALL CAMPAIGNS             | EXTRACT CAMPAIGN DATA, UPDATE CAMPAIGN |                                                                                                                          |
| EXTRACT CAMPAIGN DATA   | HTTP Request         | Pulls lead export CSV per campaign                | Loop Over Items                 | Code (CSV Parsing)                    |                                                                                                                          |
| Code                    | Code (JS)            | Parses CSV lead data into structured JSON         | EXTRACT CAMPAIGN DATA           | UPSERT CAMPAIGN ACTIVITY              |                                                                                                                          |
| UPSERT CAMPAIGN ACTIVITY| PostgreSQL           | Inserts/updates campaign lead activity data       | Code                          | Merge                                |                                                                                                                          |
| UPDATE CAMPAIGN         | PostgreSQL           | Inserts/updates campaign metadata                  | Loop Over Items                 | Merge                                |                                                                                                                          |
| Merge                   | Merge                | Combines UPSERT CAMPAIGN ACTIVITY & UPDATE CAMPAIGN outputs | UPSERT CAMPAIGN ACTIVITY, UPDATE CAMPAIGN | Loop Over Items                   |                                                                                                                          |
| SEARCH                  | PostgreSQL           | Queries campaign activity needing lifecycle update| SET SMARTLEAD API KEY           | Loop Over Items1                     |                                                                                                                          |
| Loop Over Items1        | SplitInBatches       | Iterates activity records for HubSpot enrichment  | SEARCH                        | HubSpot                             |                                                                                                                          |
| HubSpot                 | HubSpot API          | Retrieves lifecycle stage data for contacts       | Loop Over Items1               | If                                  |                                                                                                                          |
| If                      | If                   | Checks if HubSpot companyId exists                 | HubSpot                       | HUBSPOT TABLE (true), Postgres1 (false) | See note: "## HUBSPOT LIFECYCLESTAGE (LEAD STATUS)"                                                                       |
| HUBSPOT TABLE           | PostgreSQL           | Upserts HubSpot lifecycle data                      | If (true)                    | UPDATE HUBSPOT ACTIVITY TABLE        |                                                                                                                          |
| UPDATE HUBSPOT ACTIVITY TABLE | PostgreSQL     | Updates lifecycle check timestamp                   | HUBSPOT TABLE                | Loop Over Items1                    |                                                                                                                          |
| Postgres1               | PostgreSQL           | Updates lifecycle check timestamp for no HubSpot id | If (false)                   | Loop Over Items1                    |                                                                                                                          |
| Postgres                | PostgreSQL           | Aggregates lifecycle and campaign data for report | SET SMARTLEAD API KEY           | Google Sheets                      | See note: "## POSTGRES INSTALATION [Guide](https://github.com/wukimidaire/postgres_table_templates)"                      |
| Google Sheets           | Google Sheets        | Appends or updates performance report              | Postgres                     | None                               |                                                                                                                          |
| Sticky Notes (multiple) | Sticky Note          | Documentation and explanations                      | None                         | None                               | See detailed notes in sections 5 and inline with relevant blocks                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set desired interval (e.g., every hour or daily) for report update frequency.

2. **Create a Set node ("SET SMARTLEAD API KEY"):**  
   - Assign a string variable named `API KEY` with your Smartlead API key value.

3. **Create an HTTP Request node ("FETCH ALL CAMPAIGNS"):**  
   - Method: GET  
   - URL: `https://server.smartlead.ai/api/v1/campaigns`  
   - Query Parameters: `api_key={{ $json["API KEY"] }}`  
   - Connect output from "SET SMARTLEAD API KEY".

4. **Create a SplitInBatches node ("Loop Over Items"):**  
   - Input: Output from "FETCH ALL CAMPAIGNS"  
   - Batch size: Default (1)  
   - Purpose: Iterate through each campaign.

5. **Create an HTTP Request node ("EXTRACT CAMPAIGN DATA"):**  
   - Method: GET  
   - URL: `https://server.smartlead.ai/api/v1/campaigns/{{ $json.id }}/leads-export`  
   - Query Parameters: `api_key={{ $json["API KEY"] }}`  
   - Input: From "Loop Over Items".

6. **Create a Code node ("Code"):**  
   - JavaScript code to parse CSV string in `data` field into JSON array with 22 fields, converting booleans and numbers properly.  
   - Input: Output from "EXTRACT CAMPAIGN DATA".

7. **Create a PostgreSQL node ("UPSERT CAMPAIGN ACTIVITY"):**  
   - Operation: Upsert  
   - Table: `ce_campaign_activity`  
   - Schema: `outbound_activities`  
   - Map each JSON property from parsed leads to corresponding table columns, matching on `id`.  
   - Input: From Code node.

8. **Create a PostgreSQL node ("UPDATE CAMPAIGN"):**  
   - Operation: Upsert  
   - Table: `ce_campaign`  
   - Schema: `outbound_activities`  
   - Map campaign metadata from "Loop Over Items" node to table columns, matching on `campaign_id`.

9. **Create a Merge node ("Merge"):**  
   - Mode: Combine by position  
   - Inputs: Outputs from "UPSERT CAMPAIGN ACTIVITY" and "UPDATE CAMPAIGN"  
   - Output: Back to "Loop Over Items" to continue batches.

10. **Create a PostgreSQL node ("SEARCH"):**  
    - Operation: Execute SQL query (provided in the workflow) to select campaign activities needing lifecycle stage refresh.  
    - Use your PostgreSQL credentials.

11. **Create a SplitInBatches node ("Loop Over Items1"):**  
    - Input: Output from "SEARCH"  
    - Default batch size.

12. **Create a HubSpot node ("HubSpot"):**  
    - Operation: getAll contacts  
    - Authentication: OAuth2 (set up HubSpot OAuth2 credential)  
    - Input: From "Loop Over Items1".

13. **Create an If node ("If"):**  
    - Condition: Check if `companyId` property from HubSpot response is empty or not.

14. **Create a PostgreSQL node ("HUBSPOT TABLE"):**  
    - Operation: Upsert into `hubspot` table  
    - Schema: `outbound_activities`  
    - Map lifecycle stage, open deals, campaign id, etc.  
    - Input: True branch of "If".

15. **Create a PostgreSQL node ("UPDATE HUBSPOT ACTIVITY TABLE"):**  
    - Operation: Update `ce_campaign_activity`  
    - Set `hb_lifecyclestage_check_timestamp` to current time for updated records.  
    - Input: Output from "HUBSPOT TABLE".

16. **Create a PostgreSQL node ("Postgres1"):**  
    - Operation: Update `ce_campaign_activity` for records without HubSpot company id, setting timestamp.  
    - Input: False branch of "If".

17. **Connect "UPDATE HUBSPOT ACTIVITY TABLE" and "Postgres1" outputs back to "Loop Over Items1" to continue batch processing.**

18. **Create a PostgreSQL node ("Postgres"):**  
    - Operation: Execute Query  
    - SQL: Aggregate lifecycle and campaign data, summing counts grouped by campaign id.  
    - Input: From "SET SMARTLEAD API KEY" (parallel branch).

19. **Create a Google Sheets node ("Google Sheets"):**  
    - Operation: appendOrUpdate  
    - Document ID and Sheet name: Use your Google Sheets document for reporting.  
    - Map fields from aggregation query to columns (campaign name, status, lead count, customer count, etc.)  
    - Authentication: Google Sheets OAuth2 credentials.

20. **Connect "Postgres" output to "Google Sheets" node to finalize reporting.**

21. **Add Sticky Notes (optional) to document grouping and instructions as needed.**

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Search for your Smartlead API key [here](https://app.smartlead.ai/app/settings/profile) | SET SMARTLEAD API KEY node comment |
| PostgreSQL installation and table setup guide [GitHub](https://github.com/wukimidaire/postgres_table_templates) | Setup requirements for PostgreSQL tables: ce_campaign_activity, ce_campaign, hubspot |
| HubSpot lifecycle stage (lead status) explanation | Context for lifecycle enrichment block |
| Campaign Analytics Report Documentation included in workflow notes | Explains report metrics, status, and usage insights |
| Google Account integration guide | [n8n Google OAuth2 Credentials](https://docs.n8n.io/integrations/builtin/credentials/google/#compatible-nodes) |
| Assistance and suggestions contact | [LinkedIn: Victor Decoster](https://www.linkedin.com/in/victordecoster/) |

---

This detailed documentation enables advanced users and AI agents to understand, reproduce, and modify the **Smartlead to HubSpot Performance Analytics** workflow completely, anticipate integration points, and handle potential errors effectively.