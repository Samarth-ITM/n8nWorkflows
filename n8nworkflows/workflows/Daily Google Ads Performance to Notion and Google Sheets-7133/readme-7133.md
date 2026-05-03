Daily Google Ads Performance to Notion and Google Sheets

https://n8nworkflows.xyz/workflows/daily-google-ads-performance-to-notion-and-google-sheets-7133


# Daily Google Ads Performance to Notion and Google Sheets

### 1. Workflow Overview

This workflow automates the daily extraction, aggregation, and storage of Google Ads campaign performance data, focusing on both click-level (top-of-funnel) and conversion-level (bottom-of-funnel) metrics. It targets marketing teams and analysts who require an automated daily report integrating Google Ads data into Notion databases and Google Sheets for campaign-level tracking and summary analytics.

The workflow is logically organized into these blocks:

- **1.1 Scheduled Trigger & Date Setup:** Initiates the workflow daily at 8:00 AM and sets the reporting date to yesterday.
- **1.2 Google Ads Data Query:** Executes two separate HTTP POST requests to Google Ads API using GAQL:
  - Query Click Metrics (impressions, clicks, cost)
  - Query Conversion Metrics (conversions, conversion actions)
- **1.3 Data Splitting and Merging:** Splits API responses into individual campaign records and merges click and conversion data on campaign ID and date.
- **1.4 Data Storage per Campaign:** Stores each campaign’s detailed metrics into a Notion database and appends rows to a Google Sheets tab.
- **1.5 Daily Aggregation and Summary:** Aggregates totals and unique conversion types across all campaigns, then stores the summary into another Notion database and Google Sheets tab.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Date Setup

- **Overview:**  
  This block triggers the workflow every day at 08:00 AM and sets a variable for the reporting date (`yesterday`), which is used in subsequent API queries to Google Ads.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set Yesterday Date2  

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Role: Starts workflow execution daily at hour 8 (8:00 AM).  
    - Configuration: Interval set to trigger once daily at 8 AM.  
    - Inputs: None (start node)  
    - Outputs: Connects to `Set Yesterday Date2`  
    - Potential failures: Scheduling misconfiguration, timezone mismatch.

  - **Set Yesterday Date2**  
    - Type: `Set`  
    - Role: Defines a workflow variable `yesterday` with the date string of the previous day (`yyyy-MM-dd` format).  
    - Configuration: Uses n8n expression `$now.minus({ days: 1 }).toFormat('yyyy-MM-dd')` for dynamic date calculation.  
    - Inputs: From `Schedule Trigger`  
    - Outputs: Feeds two Google Ads query nodes (`G-Ads Query Conversion` and `G-Ads Query Click`)  
    - Potential issues: Incorrect timezone may lead to wrong date; expression parsing errors if $now is undefined.

#### 2.2 Google Ads Data Query

- **Overview:**  
  Executes two GAQL queries via HTTP POST requests to Google Ads API to fetch campaign-level metrics: one for clicks/impressions/cost, and the other for conversions and conversion types.

- **Nodes Involved:**  
  - G-Ads Query Click  
  - G-Ads Query Conversion  

- **Node Details:**  

  - **G-Ads Query Click**  
    - Type: `HTTP Request`  
    - Role: Sends POST request to Google Ads API to retrieve campaign impressions, clicks, cost, and date for `yesterday`.  
    - Configuration:  
      - URL: `https://googleads.googleapis.com/v20/customers/{{YOUR_CUSTOMER_ID}}/googleAds:search`  
      - Method: POST  
      - Body: GAQL query selecting `campaign.id`, `campaign.name`, `metrics.impressions`, `metrics.clicks`, `metrics.cost_micros`, `segments.date` filtered by `segments.date = yesterday`.  
      - Headers: `developer-token`, `login-customer-id`, `Content-Type: application/json`  
      - Authentication: Google Ads OAuth2 credential  
    - Inputs: From `Set Yesterday Date2` (to get date)  
    - Outputs: Connects to `Split Click2`  
    - Failure modes: Auth errors, invalid credentials, API quota limits, malformed query, network timeout.

  - **G-Ads Query Conversion**  
    - Type: `HTTP Request`  
    - Role: Sends POST request to Google Ads API to retrieve conversions, conversion action names, and date for `yesterday`.  
    - Configuration:  
      - Same URL and headers as above  
      - GAQL query selecting `campaign.id`, `campaign.name`, `metrics.conversions`, `segments.conversion_action_name`, `segments.date` filtered by `segments.date = yesterday`.  
      - `onError` set to continue workflow (non-blocking failures)  
    - Inputs: From `Set Yesterday Date2`  
    - Outputs: Connects to `Split Conversion2`  
    - Failure modes: Same as `G-Ads Query Click`, plus potential partial data missing if conversions are low or absent.

#### 2.3 Data Splitting and Merging

- **Overview:**  
  Splits the array of campaign results from each API response into individual items and merges the click and conversion datasets on campaign ID and date.

- **Nodes Involved:**  
  - Split Click2  
  - Split Conversion2  
  - Merge2  

- **Node Details:**  

  - **Split Click2**  
    - Type: `Code`  
    - Role: Takes `results` array from `G-Ads Query Click` response and outputs each campaign as a separate item for processing.  
    - Configuration: JavaScript code `return items[0].json.results.map(r => ({ json: r }));`  
    - Inputs: From `G-Ads Query Click`  
    - Outputs: Connects to `Merge2` (main input 1)  
    - Failures: If results array missing or malformed, code may throw or output empty.

  - **Split Conversion2**  
    - Type: `Code`  
    - Role: Similar to `Split Click2`, splits conversion results into individual campaign items.  
    - Configuration: Same code as above.  
    - Inputs: From `G-Ads Query Conversion`  
    - Outputs: Connects to `Merge2` (main input 2)  
    - Failures: Same as above.

  - **Merge2**  
    - Type: `Merge`  
    - Role: Combines click and conversion data items into enriched records based on matching `campaign.id` and `segments.date`.  
    - Configuration:  
      - Mode: Combine (enrich input 1 with input 2)  
      - Join fields: `campaign.id` and `segments.date`  
      - Advanced join enabled  
    - Inputs: From `Split Click2` (input 1) and `Split Conversion2` (input 2)  
    - Outputs: Connects to `Google Sheets10` and `Daily Recap2`  
    - Failures: Mismatched keys lead to missing data enrichment; handling of missing conversion data is critical.

#### 2.4 Data Storage per Campaign

- **Overview:**  
  Stores each merged campaign record into the Notion database "Google Ads Campaign Tracker" and appends it to the "Campaign Daily Report" Google Sheets tab.

- **Nodes Involved:**  
  - Google Sheets10  
  - Notion1  

- **Node Details:**  

  - **Google Sheets10**  
    - Type: `Google Sheets`  
    - Role: Appends each campaign’s metrics as a new row in the Google Sheets "Campaign Daily Report" tab.  
    - Configuration:  
      - Operation: Append  
      - Document ID and Sheet Name provided as parameters (credentials required)  
    - Inputs: From `Merge2`  
    - Outputs: None (end node for this path)  
    - Failures: OAuth2 token expiry, sheet access permission issues, invalid document or sheet IDs.

  - **Notion1**  
    - Type: `Notion`  
    - Role: Creates a new page in the Notion database "Google Ads Campaign Tracker," mapping campaign metrics into database fields.  
    - Configuration:  
      - Resource: databasePage  
      - Database ID: input required  
      - Properties mapped with expressions converting metric data, e.g., parsing cost micros to standard units, rounding conversions.  
      - `onError`: continue workflow (non-blocking)  
    - Inputs: From `Merge2`  
    - Outputs: None (end node for this path)  
    - Failures: API rate limits, incorrect database ID, missing or misconfigured integration permissions.

#### 2.5 Daily Aggregation and Summary

- **Overview:**  
  Aggregates total clicks, impressions, cost, conversions, and unique conversion types across all campaigns for the day. Stores the summary in the Notion database "Google Ads Daily Summary" and the "Summary Report" Google Sheets tab.

- **Nodes Involved:**  
  - Daily Recap2  
  - Notion2  
  - Google Sheets11  

- **Node Details:**  

  - **Daily Recap2**  
    - Type: `Code`  
    - Role: Aggregates totals and collects unique conversion types from all campaign records.  
    - Configuration:  
      - JavaScript code loops through input items, sums metrics, deduplicates conversion types, and returns a single summary object.  
    - Inputs: From `Merge2` (after Google Sheets10)  
    - Outputs: Connects to `Notion2` and `Google Sheets11`  
    - Failures: Empty input may cause zero totals; data type inconsistencies may cause NaN.

  - **Notion2**  
    - Type: `Notion`  
    - Role: Creates a new page in the Notion database "Google Ads Daily Summary" with aggregated daily totals and conversion types.  
    - Configuration:  
      - Maps aggregated fields: date, total impressions, clicks, conversions, cost, and conversion types.  
      - `onError`: continue workflow  
    - Inputs: From `Daily Recap2`  
    - Outputs: Connects to `Google Sheets11`  
    - Failures: Same as other Notion node.

  - **Google Sheets11**  
    - Type: `Google Sheets`  
    - Role: Appends the daily summary as a new row to the "Summary Report" tab of the Google Sheets document.  
    - Configuration:  
      - Operation: Append  
      - Document ID and Sheet Name provided  
    - Inputs: From `Notion2`  
    - Outputs: None  
    - Failures: Same as other Google Sheets node.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                             | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                                       |
|---------------------|-------------------|--------------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger  | Trigger workflow daily at 08:00 AM          | None                          | Set Yesterday Date2          | Setup steps: Schedule workflow daily at 08:00 AM                                                                                |
| Set Yesterday Date2  | Set               | Define `yesterday` date variable             | Schedule Trigger              | G-Ads Query Conversion, G-Ads Query Click | Setup steps: Define variable for yesterday's date                                                                               |
| G-Ads Query Click    | HTTP Request      | Fetch campaign impressions, clicks, cost    | Set Yesterday Date2           | Split Click2                | Google Ads Click vs Conversion (HTTP POST): top-of-funnel metrics query                                                        |
| G-Ads Query Conversion | HTTP Request    | Fetch campaign conversions and conversion types | Set Yesterday Date2           | Split Conversion2           | Google Ads Click vs Conversion (HTTP POST): bottom-of-funnel metrics query                                                     |
| Split Click2         | Code              | Split click query results into individual campaigns | G-Ads Query Click            | Merge2                     |                                                                                                                                |
| Split Conversion2    | Code              | Split conversion query results into individual campaigns | G-Ads Query Conversion       | Merge2                     |                                                                                                                                |
| Merge2               | Merge             | Merge click and conversion data on campaign ID and date | Split Click2, Split Conversion2 | Google Sheets10, Daily Recap2 | G-Ads Query Conversion: fetches bottom-of-funnel metrics (results and ROI)                                                     |
| Google Sheets10      | Google Sheets     | Append each campaign's metrics to sheet      | Merge2                       | None                        | Output Summary: Campaign Daily Report tab                                                                                      |
| Notion1              | Notion            | Store each campaign's metrics into Notion   | Merge2                       | None                        | Notion database: Google Ads Campaign Tracker                                                                                   |
| Daily Recap2         | Code              | Aggregate daily totals and unique conversion types | Merge2                       | Notion2                    |                                                                                                                                |
| Notion2              | Notion            | Store daily aggregated summary into Notion  | Daily Recap2                 | Google Sheets11             | Notion database: Google Ads Daily Summary                                                                                      |
| Google Sheets11      | Google Sheets     | Append daily summary to Google Sheets        | Notion2                     | None                        | Output Summary: Summary Report tab                                                                                            |
| Sticky Note          | Sticky Note       | Workflow description and explanation         | None                        | None                        | How It Works: detailed step-by-step workflow explanation                                                                      |
| Sticky Note7         | Sticky Note       | Workflow description                          | None                        | None                        | Workflow description: purpose and overview                                                                                   |
| Sticky Note8         | Sticky Note       | Setup instructions for schedule and API      | None                        | None                        | Setup steps: scheduling and Google Ads API setup                                                                              |
| Sticky Note9         | Sticky Note       | Notion and Google Sheets database setup      | None                        | None                        | Notion and Google Sheets setup instructions                                                                                   |
| Sticky Note10        | Sticky Note       | Output summary of data destinations           | None                        | None                        | Output Summary: destinations in Notion and Google Sheets                                                                      |
| Sticky Note11        | Sticky Note       | Explanation of Google Ads queries             | None                        | None                        | Google Ads Click vs Conversion (HTTP POST) description                                                                        |
| Sticky Note12        | Sticky Note       | Explanation of conversion query                | None                        | None                        | G-Ads Query Conversion explanation                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: `Schedule Trigger`  
   - Parameters: Set to trigger daily at 08:00 AM (triggerAtHour = 8)  
   - Output: Connect to next node.

2. **Create Set Node "Set Yesterday Date2"**  
   - Type: `Set`  
   - Add string field named `yesterday`  
   - Value: Use expression `{{$now.minus({ days: 1 }).toFormat('yyyy-MM-dd')}}`  
   - Connect input from Schedule Trigger.

3. **Create HTTP Request Node "G-Ads Query Click"**  
   - Type: `HTTP Request`  
   - Configure:  
     - URL: `https://googleads.googleapis.com/v20/customers/{{YOUR_CUSTOMER_ID}}/googleAds:search` (replace `YOUR_CUSTOMER_ID`)  
     - Method: POST  
     - Body type: JSON  
     - JSON Body:  
       ```json
       {
         "query": "SELECT campaign.id, campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros, segments.date FROM campaign WHERE segments.date = '{{ $json.yesterday }}'"
       }
       ```  
     - Headers: Include `developer-token`, `login-customer-id`, `Content-Type: application/json`  
     - Authentication: Google Ads OAuth2 API credential (configure OAuth2 with appropriate scopes and tokens)  
   - Connect input from Set Yesterday Date2.

4. **Create HTTP Request Node "G-Ads Query Conversion"**  
   - Same setup as "G-Ads Query Click" but change query to:  
     ```json
     {
       "query": "SELECT campaign.id, campaign.name, metrics.conversions, segments.conversion_action_name, segments.date FROM campaign WHERE segments.date = '{{ $json.yesterday }}'"
     }
     ```  
   - Set `onError` to continue workflow (non-blocking).  
   - Connect input from Set Yesterday Date2.

5. **Create Code Node "Split Click2"**  
   - Type: `Code`  
   - JavaScript code:  
     ```js
     return items[0].json.results.map(r => ({ json: r }));
     ```  
   - Connect input from G-Ads Query Click.

6. **Create Code Node "Split Conversion2"**  
   - Same as Split Click2 with identical code.  
   - Connect input from G-Ads Query Conversion.

7. **Create Merge Node "Merge2"**  
   - Type: `Merge`  
   - Mode: Combine (enrich input 1 with input 2)  
   - Join mode: Advanced mode enabled  
   - Set join fields:  
     - Field1: `campaign.id` (left), Field2: `campaign.id` (right)  
     - Field1: `segments.date` (left), Field2: `segments.date` (right)  
   - Connect inputs:  
     - Input 1: Split Click2  
     - Input 2: Split Conversion2

8. **Create Google Sheets Node "Google Sheets10"**  
   - Operation: Append  
   - Document ID: URL or ID of your Google Sheets document  
   - Sheet Name: Tab name for campaign daily report  
   - Connect input from Merge2.

9. **Create Notion Node "Notion1"**  
   - Resource: databasePage  
   - Database ID: your "Google Ads Campaign Tracker" database ID  
   - Map properties:  
     - Campaign Name (title): `={{ $json.campaign.name }}`  
     - Campaign ID (number): `={{ parseInt($json.campaign.id) }}`  
     - Impressions (number): `={{ parseInt($json.metrics.impressions) }}`  
     - Clicks (number): `={{ parseInt($json.metrics.clicks) }}`  
     - Cost (number): `={{ Math.floor($json.metrics.costMicros / 1000000) }}`  
     - Conversion Type (rich_text): `={{ $json.segments.conversionActionName || "N/A" }}`  
     - Conversions (number): `={{ Math.round(Number($json.metrics.conversions || 0) * 100) / 100 }}`  
     - Date (date): `={{ $json.segments.date }}`  
   - Connect input from Merge2.  
   - Set `onError` to continue.

10. **Create Code Node "Daily Recap2"**  
    - JavaScript code:  
      ```js
      let totalClicks = 0;
      let totalImpressions = 0;
      let totalCost = 0;
      let totalConversions = 0;
      let conversionTypes = new Set();
      let date = null;

      for (const item of items) {
        const d = item.json;

        totalClicks += parseInt(d.metrics?.clicks || 0);
        totalImpressions += parseInt(d.metrics?.impressions || 0);
        totalCost += parseInt(d.metrics?.costMicros || 0) / 1_000_000;
        totalConversions += parseFloat(d.metrics?.conversions || 0);

        const convType = d.segments?.conversionActionName;
        if (convType) conversionTypes.add(convType);

        date = d.segments?.date || date;
      }

      return [
        {
          json: {
            date,
            total_clicks: totalClicks,
            total_impressions: totalImpressions,
            total_cost: Number(totalCost.toFixed(2)),
            total_conversions: Number(totalConversions.toFixed(2)),
            conversion_types: Array.from(conversionTypes).join(', ') || 'N/A'
          }
        }
      ];
      ```  
    - Connect input from Merge2 (after Google Sheets10).

11. **Create Notion Node "Notion2"**  
    - Resource: databasePage  
    - Database ID: your "Google Ads Daily Summary" database ID  
    - Map properties:  
      - Date (date): `={{ $json.date }}`  
      - Total Impressions (number): `={{ $json.total_impressions }}`  
      - Total Clicks (number): `={{ $json.total_clicks }}`  
      - Total Conversions (number): `={{ $json.total_conversions }}`  
      - Total Cost (number): `={{ $json.total_cost }}`  
      - Conversion Types (rich_text): `={{ $json.conversion_types }}`  
    - Connect input from Daily Recap2.  
    - Set `onError` to continue.

12. **Create Google Sheets Node "Google Sheets11"**  
    - Operation: Append  
    - Document ID: same or different Google Sheets document URL or ID  
    - Sheet Name: Tab name for summary report  
    - Connect input from Notion2.

13. **Configure Credentials:**  
    - Google Ads OAuth2 API credentials with developer token and login-customer-id headers configured for HTTP requests.  
    - Google Sheets OAuth2 credentials for Google Sheets nodes.  
    - Notion credentials with integration access to databases.

14. **Test the workflow end-to-end and adjust database IDs, sheet names, and customer IDs as needed.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow is designed to run daily at 08:00 AM, pulling data for the previous day to ensure complete reporting.                                                              | Setup instructions in Sticky Note8 and Sticky Note9                                                    |
| Google Ads developer token and OAuth2 credentials must be correctly configured with appropriate permissions for API access.                                                | https://developers.google.com/google-ads/api/docs/oauth/overview                                       |
| Notion databases require fields matching the mapped properties exactly and must be shared with the integration used for API access.                                       | https://developers.notion.com/docs/getting-started                                                     |
| Google Sheets document must have two tabs named as per the workflow and matching columns for seamless data append operations.                                              |                                                                                                         |
| The workflow gracefully handles partial failures on conversion data queries and Notion writes to avoid full workflow failure.                                             | Nodes configured with `onError: continue`                                                              |
| Conversion cost is calculated by dividing micros (`cost_micros`) by 1,000,000 and rounding for readability.                                                                |                                                                                                         |
| For best results, ensure the Google Ads customer ID used is the client account (not the MCC manager account).                                                              |                                                                                                         |
| Explanation of Google Ads queries and data fields documented in Sticky Note11 and Sticky Note12 within the workflow.                                                        |                                                                                                         |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.