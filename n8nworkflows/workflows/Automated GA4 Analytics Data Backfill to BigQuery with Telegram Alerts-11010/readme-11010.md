Automated GA4 Analytics Data Backfill to BigQuery with Telegram Alerts

https://n8nworkflows.xyz/workflows/automated-ga4-analytics-data-backfill-to-bigquery-with-telegram-alerts-11010


# Automated GA4 Analytics Data Backfill to BigQuery with Telegram Alerts

### 1. Workflow Overview

This workflow automates the backfilling of Google Analytics 4 (GA4) data into Google BigQuery (BQ) on a scheduled basis, with Telegram notifications for status alerts. It targets data engineers, analysts, and marketers who want to maintain historical GA4 datasets in BigQuery for analysis or reporting.

The workflow is logically divided into the following functional blocks:

- **1.1 Configuration & Trigger**: Defines date ranges and GA4/BQ project info; triggers workflow execution on schedule.
- **1.2 GA4 Data Extraction**: Fetches multiple GA4 reports using Google Analytics API nodes, each targeting specific dimensions and metrics.
- **1.3 BigQuery Data Insertion**: Corresponding BigQuery nodes create/replace tables and insert the fetched GA4 data.
- **1.4 Data Merging & Status Aggregation**: Merges output of all BigQuery insertions to prepare an aggregate success/failure report.
- **1.5 Telegram Alerting**: Sends a summary message via Telegram indicating overall workflow success or any failed steps.
- **1.6 Documentation Notes**: Sticky notes providing user instructions and project references.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration & Trigger

- **Overview**: Initializes workflow parameters (project ID, dataset, GA4 Property ID, date range) and triggers the workflow execution periodically.
- **Nodes Involved**:  
  - Schedule Trigger  
  - Backfill Config

- **Node Details**:

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Executes the workflow automatically every day at minute 27 of the hour.  
    - Configuration: Interval set to trigger at minute 27 of each hour (daily use assumed).  
    - Inputs: None (entry point)  
    - Outputs: Connects to Backfill Config node  
    - Edge cases: If workflow is manually started outside schedule, config still applies.

  - **Backfill Config**  
    - Type: Set  
    - Role: Defines all critical configuration variables used downstream: GA4 Property ID, BigQuery project and dataset IDs, and the date range (auto set to two days before current date).  
    - Configuration:  
      - `GA4 Property ID`: Static string (461978355)  
      - `project_id`: Google Cloud project string  
      - `dataset_id`: BigQuery dataset name  
      - `startDate` and `endDate`: Both set dynamically to two days ago’s date in `yyyy-MM-dd` format using n8n’s DateTime expression.  
    - Inputs: From Schedule Trigger  
    - Outputs: Feeds all GA4 nodes for API queries  
    - Edge cases: Misconfiguration or invalid date format could cause API or SQL query failures.

---

#### 1.2 GA4 Data Extraction

- **Overview**: Queries GA4 API for multiple datasets, each with specific dimensions and metrics, for the configured date range.
- **Nodes Involved**:  
  - GA4 - Transaction Items  
  - GA4 - Session Channel Group  
  - GA4 - Session Source/Campaign/Medium  
  - GA4 - Country/Language/City  
  - GA4 - Item Name  
  - GA4 - Browser/OS/Device  
  - GA4 - First User Source/Medium  
  - GA4 - First User Channel Group  
  - GA4 - Ads Data  
  - GA4 - All Metrics  
  - GA4 - Event Metrics  
  - GA4 - Page Location  
  - GA4 - Landing Page

- **Node Details (common characteristics)**:

  - Type: Google Analytics (GA4) node  
  - Role: Extracts GA4 metrics and dimensions for a custom date range  
  - Configuration:  
    - `startDate` and `endDate` passed from Backfill Config node via expressions  
    - `propertyId` dynamically set to the GA4 Property ID from Backfill Config  
    - `metricsGA4` and `dimensionsGA4` configured per node to retrieve required data slices (sessions, users, ecommerce metrics, source/medium, device info, etc.)  
    - `returnAll`: true to fetch all available data  
  - Credentials: Google Analytics OAuth2 (shared credential)  
  - Inputs: From Backfill Config node  
  - Outputs: Connect to corresponding BigQuery nodes  
  - Edge cases:  
    - API rate limits or quota exceeded  
    - Invalid Property ID or date range  
    - No data returned for given range (handled gracefully)  
  - Version: n8n Google Analytics node version 2 (some nodes 2.1 for BigQuery integration)

---

#### 1.3 BigQuery Data Insertion

- **Overview**: For each GA4 dataset, creates or replaces a dedicated BigQuery table named with the `startDate` suffix and inserts the GA4 data into this table.
- **Nodes Involved**:  
  - BQ - ga4_transaction_items  
  - BQ - ga4_data_session_channel_group  
  - BQ - ga4_data_session_source_campaign_medium  
  - BQ - ga4_data_country_language_city  
  - BQ - ga4_data_item_name  
  - BQ - ga4_data_browser_os_device  
  - BQ - ga4_data_first_user_source_medium  
  - BQ - ga4_data_first_user_channel_group  
  - BQ - ga4_ads_data  
  - BQ - ga4_all_metrics_data  
  - BQ - ga4_event_metrics_data  
  - BQ - ga4_page_location_data  
  - BQ - ga4_landing_page_data

- **Node Details (common characteristics)**:

  - Type: Google BigQuery node  
  - Role: Create or replace tables and insert GA4 data using parameterized SQL queries  
  - Configuration:  
    - Uses `CREATE OR REPLACE TABLE` with a dynamic table name suffixed by the date (formatted as `yyyyMMdd`) from Backfill Config  
    - Inserts rows from the associated GA4 node by mapping GA4 API response fields into SQL values using JavaScript expressions  
    - `projectId` dynamically set from Backfill Config  
  - Credentials: Google BigQuery OAuth2 (shared credential)  
  - Inputs: Connected from corresponding GA4 nodes  
  - Outputs: Connected to Merge nodes for aggregation  
  - On error: Set to continue workflow to avoid full failure on partial BigQuery errors  
  - Edge cases:  
    - SQL syntax errors due to unexpected data or escaping issues  
    - BigQuery quota or permission errors  
    - Null or missing fields handled via `|| 'NULL'` fallback  
  - Version: 2.1 for enhanced SQL support

---

#### 1.4 Data Merging & Status Aggregation

- **Overview**: Combines outputs of all BQ insertion nodes to produce a consolidated success/failure report.
- **Nodes Involved**:  
  - Merge (10 inputs)  
  - Merge1 (4 inputs)  
  - Code

- **Node Details**:

  - **Merge**  
    - Type: Merge  
    - Role: Aggregates outputs from 10 BQ nodes into one stream  
    - Inputs: Outputs from 10 BQ nodes (e.g., transaction items, session data, ads data)  
    - Outputs: Connects to Merge1 node  
    - Edge cases: Missing inputs or partial failures are handled by the merge node.

  - **Merge1**  
    - Type: Merge  
    - Role: Further aggregates outputs from 4 remaining BQ nodes with output of first Merge node  
    - Inputs: Outputs from Merge and 4 BQ nodes (event metrics, page location, landing page)  
    - Outputs: Connects to Code node  
    - Edge cases: Similar to above.

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Analyzes all merged input items to detect which BQ nodes succeeded or failed; builds a Telegram message with counts and failed node names.  
    - Configuration:  
      - Hardcoded node names array in order expected from merged inputs  
      - Checks each input item for `json.error` property to determine failure  
      - Escapes underscores in node names to prevent Telegram Markdown issues  
      - Constructs a multiline summary message reporting successes and failures  
    - Inputs: From Merge1 node  
    - Outputs: JSON object with `message` property ready for Telegram notification  
    - Edge cases: If input order changes or nodes rename, message may mismatch outputs.

---

#### 1.5 Telegram Alerting

- **Overview**: Sends a summary status message to a Telegram chat/group to notify workflow run results.
- **Nodes Involved**:  
  - Send a text message

- **Node Details**:

  - **Send a text message**  
    - Type: Telegram node  
    - Role: Posts a message to a specific Telegram chat ID  
    - Configuration:  
      - Message text is dynamically set from the Code node’s output `message` field.  
      - Chat ID: `-1002962247686` (Telegram group or channel)  
      - Attribution disabled to keep message clean  
    - Credentials: Telegram API key configured under "GA4 Backfill"  
    - Inputs: From Code node  
    - Edge cases: Telegram API failures, invalid chat ID, or network issues could prevent alert delivery.

---

#### 1.6 Documentation Notes

- **Overview**: Provides embedded user guidance and project links in the workflow canvas.
- **Nodes Involved**:  
  - Sticky Note  
  - Sticky Note1 (empty)

- **Node Details**:

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Contains detailed instructions on how to configure and use the workflow, including credential and parameter setup, and a link to the GitHub repository.  
    - Content highlights:  
      - How to set project properties and date ranges  
      - Credential setup for GA and BQ nodes  
      - Telegram credential and chat ID configuration  
      - Link to GitHub repo: https://github.com/aliasoblomov/N8N-GA4-Backfill-Workflow  
    - Visible on canvas for user reference.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Empty placeholder, possibly for future notes.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                             | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                   |
|-----------------------------------|----------------------------|---------------------------------------------|--------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger                  | Schedule Trigger           | Triggers workflow on schedule                | None                                 | Backfill Config                   |                                                                                               |
| Backfill Config                  | Set                        | Sets config vars: GA4 IDs, project/dataset, date range | Schedule Trigger                    | All GA4 nodes                    | See Sticky Note for usage instructions                                                        |
| GA4 - Transaction Items          | Google Analytics           | Fetch transaction item metrics from GA4     | Backfill Config                      | BQ - ga4_transaction_items       |                                                                                               |
| BQ - ga4_transaction_items       | Google BigQuery            | Create table & insert transaction items     | GA4 - Transaction Items              | Merge                           |                                                                                               |
| GA4 - Session Channel Group      | Google Analytics           | Fetch session metrics by channel group      | Backfill Config                      | BQ - ga4_data_session_channel_group |                                                                                               |
| BQ - ga4_data_session_channel_group | Google BigQuery            | Create table & insert session channel group data | GA4 - Session Channel Group          | Merge                           |                                                                                               |
| GA4 - Session Source/Campaign/Medium | Google Analytics           | Fetch session metrics by source, campaign, medium | Backfill Config                      | BQ - ga4_data_session_source_campaign_medium |                                                                                               |
| BQ - ga4_data_session_source_campaign_medium | Google BigQuery            | Create table & insert session source/campaign/medium data | GA4 - Session Source/Campaign/Medium | Merge                           |                                                                                               |
| GA4 - Country/Language/City      | Google Analytics           | Fetch user metrics by country, language, city | Backfill Config                      | BQ - ga4_data_country_language_city |                                                                                               |
| BQ - ga4_data_country_language_city | Google BigQuery            | Create table & insert country/language/city data | GA4 - Country/Language/City          | Merge                           |                                                                                               |
| GA4 - Item Name                  | Google Analytics           | Fetch item-level purchase metrics            | Backfill Config                      | BQ - ga4_data_item_name           |                                                                                               |
| BQ - ga4_data_item_name          | Google BigQuery            | Create table & insert item name metrics      | GA4 - Item Name                     | Merge                           |                                                                                               |
| GA4 - Browser/OS/Device          | Google Analytics           | Fetch session metrics by browser, OS, device | Backfill Config                      | BQ - ga4_data_browser_os_device   |                                                                                               |
| BQ - ga4_data_browser_os_device  | Google BigQuery            | Create table & insert browser/OS/device data | GA4 - Browser/OS/Device              | Merge                           |                                                                                               |
| GA4 - First User Source/Medium   | Google Analytics           | Fetch first user acquisition source/medium   | Backfill Config                      | BQ - ga4_data_first_user_source_medium |                                                                                               |
| BQ - ga4_data_first_user_source_medium | Google BigQuery            | Create table & insert first user source/medium data | GA4 - First User Source/Medium       | Merge                           |                                                                                               |
| GA4 - First User Channel Group   | Google Analytics           | Fetch first user acquisition by channel group | Backfill Config                      | BQ - ga4_data_first_user_channel_group |                                                                                               |
| BQ - ga4_data_first_user_channel_group | Google BigQuery            | Create table & insert first user channel group data | GA4 - First User Channel Group       | Merge                           |                                                                                               |
| GA4 - Ads Data                  | Google Analytics           | Fetch ad performance metrics                  | Backfill Config                      | BQ - ga4_ads_data                |                                                                                               |
| BQ - ga4_ads_data               | Google BigQuery            | Create table & insert ads data                 | GA4 - Ads Data                     | Merge                           |                                                                                               |
| GA4 - All Metrics              | Google Analytics           | Fetch all key metrics daily summary            | Backfill Config                      | BQ - ga4_all_metrics_data        |                                                                                               |
| BQ - ga4_all_metrics_data       | Google BigQuery            | Create table & insert all metrics data         | GA4 - All Metrics                  | Merge                           |                                                                                               |
| GA4 - Event Metrics            | Google Analytics           | Fetch event-related metrics                     | Backfill Config                      | BQ - ga4_event_metrics_data      |                                                                                               |
| BQ - ga4_event_metrics_data     | Google BigQuery            | Create table & insert event metrics             | GA4 - Event Metrics                | Merge1                          |                                                                                               |
| GA4 - Page Location            | Google Analytics           | Fetch metrics by page location                   | Backfill Config                      | BQ - ga4_page_location_data      |                                                                                               |
| BQ - ga4_page_location_data     | Google BigQuery            | Create table & insert page location data         | GA4 - Page Location                | Merge1                          |                                                                                               |
| GA4 - Landing Page             | Google Analytics           | Fetch landing page metrics                        | Backfill Config                      | BQ - ga4_landing_page_data       |                                                                                               |
| BQ - ga4_landing_page_data      | Google BigQuery            | Create table & insert landing page data           | GA4 - Landing Page                | Merge1                          |                                                                                               |
| Merge                          | Merge                      | Aggregate outputs from 10 BQ nodes               | Multiple BQ nodes                   | Merge1                          |                                                                                               |
| Merge1                         | Merge                      | Aggregate outputs from Merge and 4 BQ nodes      | Merge + 4 BQ nodes                 | Code                           |                                                                                               |
| Code                           | Code                       | Analyze all outputs, prepare summary message     | Merge1                            | Send a text message             |                                                                                               |
| Send a text message            | Telegram                   | Send workflow run status to Telegram chat         | Code                             | None                           |                                                                                               |
| Sticky Note                    | Sticky Note                | User instructions and GitHub repo link            | None                             | None                           | See content for setup and usage instructions: https://github.com/aliasoblomov/N8N-GA4-Backfill-Workflow |
| Sticky Note1                   | Sticky Note                | Empty placeholder note                             | None                             | None                           |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Parameters: Set interval to trigger at minute 27 every hour or daily as needed.

2. **Create Backfill Config Node (Set Node)**  
   - Connect Schedule Trigger → Backfill Config  
   - Assign variables:  
     - `GA4 Property ID`: Your GA4 property numeric ID (e.g., "461978355")  
     - `project_id`: Your Google Cloud project ID for BigQuery  
     - `dataset_id`: BigQuery dataset name (e.g., "GA4")  
     - `startDate`: Expression `={{ DateTime.now().minus({ days: 2 }).toFormat('yyyy-MM-dd') }}`  
     - `endDate`: Same expression as startDate

3. **For Each GA4 Report: Create Google Analytics (GA4) Nodes**  
   - Connect Backfill Config → Each GA4 node  
   - Configure each node with:  
     - `propertyId`: Expression `={{ $json['GA4 Property ID'] }}`  
     - Date range: Custom with `startDate` and `endDate` from Backfill Config  
     - Metrics and dimensions as per original node (refer to block 1.2 analysis for exact metrics/dimensions)  
     - Set `returnAll` to true  
     - Use the same Google Analytics OAuth2 credential for all GA4 nodes

   GA4 nodes to create (with key dimension/metric highlights):  
   - Transaction Items (dimensions: transactionId, itemName; metrics: itemPurchaseQuantity, itemRevenue)  
   - Session Channel Group (dimension: sessionDefaultChannelGroup; metrics: sessions, newUsers, ecommercePurchases, purchaseRevenue)  
   - Session Source/Campaign/Medium (dimensions: sessionSource, sessionCampaignName, sessionMedium; metrics similar to above)  
   - Country/Language/City (dimensions: city, language, country; metrics: sessions, screenPageViews, newUsers, ecommercePurchases, purchaseRevenue)  
   - Item Name (dimension: itemName; metrics: itemPurchaseQuantity, itemRevenue)  
   - Browser/OS/Device (dimensions: browser, operatingSystem, deviceCategory; metrics: sessions, screenPageViews, newUsers, ecommercePurchases, purchaseRevenue)  
   - First User Source/Medium (dimensions: firstUserMedium, firstUserCampaignName, firstUserSource; metrics: newUsers, ecommercePurchases, purchaseRevenue)  
   - First User Channel Group (dimension: firstUserDefaultChannelGroup; metrics: newUsers, ecommercePurchases, purchaseRevenue)  
   - Ads Data (dimensions: sessionSource, sessionMedium, sessionCampaignName; metrics: ecommercePurchases, averagePurchaseRevenue, purchaseRevenue, advertiserAdClicks, advertiserAdCost, advertiserAdCostPerClick, returnOnAdSpend)  
   - All Metrics (dimension: none; metrics: sessions, totalUsers, userEngagementDuration, newUsers, engagementRate, engagedSessions, screenPageViews, purchaseRevenue, ecommercePurchases)  
   - Event Metrics (dimension: eventName; metrics: eventCount, eventValue, eventCountPerUser)  
   - Page Location (dimension: pageLocation; metrics: totalUsers, ecommercePurchases, purchaseRevenue, screenPageViews, eventCount, engagementRate)  
   - Landing Page (dimension: landingPage; metrics: totalUsers, sessions, eventCount, ecommercePurchases, engagementRate, purchaseRevenue)

4. **For Each GA4 Node, Create Corresponding BigQuery Node**  
   - Connect each GA4 node → corresponding BQ node  
   - Use Google BigQuery OAuth2 credential  
   - Configure SQL Query as per the original nodes:  
     - `CREATE OR REPLACE TABLE` with table name suffixed by `startDate` in `yyyyMMdd` format  
     - Insert data using mapped GA4 rows and dimensions/metrics  
   - Set `projectId` and dataset dynamically from Backfill Config fields  
   - Set `onError` to "continueRegularOutput" to allow partial failures.

5. **Create Merge Node**  
   - Type: Merge  
   - Parameters: Number of inputs: 10  
   - Connect 10 BigQuery nodes outputs to this Merge node inputs (transaction items, session channel group, session source/campaign/medium, country/language/city, item name, browser/os/device, first user source/medium, first user channel group, ads data, all metrics)

6. **Create Merge1 Node**  
   - Type: Merge  
   - Parameters: Number of inputs: 4  
   - Connect Merge node output and 3 other BigQuery nodes (event metrics, page location, landing page) into this node inputs.

7. **Create Code Node**  
   - Connect Merge1 → Code  
   - Paste the provided JavaScript code that:  
     - Enumerates all nodes in expected order  
     - Checks for errors in each node’s output  
     - Builds a summary message of successes and failures for Telegram

8. **Create Telegram Node**  
   - Connect Code → Telegram node  
   - Set chat ID to your Telegram group/channel ID  
   - Use the Telegram API credential  
   - Configure text message with expression: `={{ $json.message }}`  
   - Disable attribution for clean message

9. **Optional: Add Sticky Note Node**  
   - Add a sticky note with instructions for user reference  
   - Include GitHub repo link: https://github.com/aliasoblomov/N8N-GA4-Backfill-Workflow

10. **Activate the workflow**  
    - Test manually or wait for scheduled trigger  
    - Ensure all credentials are valid and permissions granted  
    - Monitor Telegram notifications for status

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                        |
|---------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| This workflow automates GA4 data backfill into BigQuery and sends Telegram alerts on status.                              | Workflow purpose                                                      |
| Configure GA4 Property ID, BigQuery project/dataset, and date ranges in the Backfill Config node.                         | Workflow setup                                                        |
| Google Analytics and BigQuery OAuth2 credentials must be set and authorized properly.                                     | Credential setup                                                      |
| Telegram credential and target chat ID must be configured for alerting.                                                  | Telegram node setup                                                  |
| GitHub repository with detailed instructions and source code: https://github.com/aliasoblomov/N8N-GA4-Backfill-Workflow | Official project repository                                           |
| Edge cases include API rate limits, data availability gaps, SQL errors, and network issues which are partially handled.  | Operational considerations                                           |
| The workflow uses dynamic SQL and JavaScript expressions to map GA4 data to BigQuery tables.                              | Implementation details                                               |

---

**Disclaimer:**  
The text above is fully derived from an automated n8n workflow designed for legal and public data integration. It complies with content policies and contains no illegal or protected elements.