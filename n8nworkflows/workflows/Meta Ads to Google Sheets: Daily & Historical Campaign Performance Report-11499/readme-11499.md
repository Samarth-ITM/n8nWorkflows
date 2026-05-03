Meta Ads to Google Sheets: Daily & Historical Campaign Performance Report

https://n8nworkflows.xyz/workflows/meta-ads-to-google-sheets--daily---historical-campaign-performance-report-11499


# Meta Ads to Google Sheets: Daily & Historical Campaign Performance Report

### 1. Workflow Overview

This workflow automates the extraction of Meta (Facebook) Ads campaign performance data and appends it to a Google Sheet for reporting and analysis. It supports two main use cases:

- **1.1 Daily Incremental Export:** Automatically runs daily at 06:00 to fetch and append the previous day's campaign-level performance metrics.
- **1.2 Manual Historical Backfill:** Allows manual execution to backfill historical Meta Ads data for custom date ranges, such as quarterly or yearly periods, preventing data gaps or to initialize reporting.

The workflow is logically divided into two functional blocks corresponding to these use cases, each comprising configuration, data fetching from the Facebook Graph API, data transformation (flattening and KPI calculation), and appending results to Google Sheets. The workflow also includes descriptive sticky notes for usage guidance and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Incremental Export Block

**Overview:**  
This block schedules a daily run at 06:00 to fetch Meta Ads campaign performance data for the previous day and appends it to a specified Google Sheet. It calculates key performance indicators such as CPL (Cost Per Lead), CPA (Cost Per Acquisition), and ROAS (Return on Ad Spend).

**Nodes Involved:**  
- Daily schedule (06:00)  
- Set config (Meta + Sheets)  
- Fetch Meta Insights (yesterday)  
- Transform Meta data for Sheets  
- Append daily rows to Google Sheet  

**Node Details:**  

- **Daily schedule (06:00):**  
  - Type: Schedule Trigger  
  - Role: Initiates the daily data fetch workflow every day at 06:00.  
  - Configuration: Cron-based trigger set to 06:00 hours daily.  
  - Inputs: None  
  - Outputs: Connects to "Set config (Meta + Sheets)" node.  
  - Edge cases: Misconfigured schedule may cause missed runs.  

- **Set config (Meta + Sheets):**  
  - Type: Set  
  - Role: Sets workflow parameters such as Meta ad account ID, date preset (`yesterday`), level (`campaign`), time increment (`1`), Google Sheet ID, and sheet name for daily data.  
  - Configuration:  
    - `adAccountId`: Placeholder for Meta ad account string (e.g., `act_<YOUR_AD_ACCOUNT_ID>`)  
    - `datePreset`: `yesterday` (fetches data for the previous day)  
    - `level`: `campaign` (aggregates data at campaign level)  
    - `timeIncrement`: `1` (daily granularity)  
    - `sheetId`: Google Sheet document ID placeholder  
    - `sheetNameData`: Sheet name for daily data (`Meta_Daily_Data`)  
  - Inputs: From schedule trigger  
  - Outputs: To "Fetch Meta Insights (yesterday)"  
  - Edge cases: Incorrect ad account or sheet ID leads to API or Google Sheets errors.  

- **Fetch Meta Insights (yesterday):**  
  - Type: Facebook Graph API Node  
  - Role: Calls Meta Ads Insights API for the ad account with parameters set in "Set config".  
  - Configuration:  
    - Edge: `insights`  
    - Node (ad account): Dynamic from previous node's `adAccountId`  
    - Fields: Date, account info, campaign info, adset/ad info, impressions, clicks, spend, frequency, actions, etc.  
    - Query parameters:  
      - Level: campaign  
      - Date preset: yesterday  
      - Time increment: 1  
      - Limit: 1000  
      - Attribution windows: 1d_click, 7d_click, 1d_view  
      - Breakdowns: publisher_platform  
    - Graph API version: v23.0  
  - Inputs: From config node  
  - Outputs: To "Transform Meta data for Sheets"  
  - Edge cases: API limits, auth token expiration, network issues.  

- **Transform Meta data for Sheets:**  
  - Type: Code (JavaScript)  
  - Role: Flattens and normalizes the nested Meta Ads API response into a single-level object per campaign per day, calculates derived KPIs (CPL, CPA, ROAS).  
  - Key logic:  
    - Loops through API data array  
    - Extracts and assigns fields such as date, account IDs, campaign/ad identifiers  
    - Parses standard metrics (impressions, reach, clicks, spend, etc.)  
    - Extracts action counts and values (leads, purchases, add to cart)  
    - Calculates CPL (spend/leads), CPA (spend/purchases), ROAS (purchase_value/spend) with null fallbacks  
  - Inputs: API response data  
  - Outputs: Cleaned, KPI-enhanced data for Google Sheets  
  - Edge cases: Missing or empty API data arrays, zero division handled gracefully, unexpected data types.  

- **Append daily rows to Google Sheet:**  
  - Type: Google Sheets Node  
  - Role: Appends transformed daily campaign data as rows to the specified Google Sheet and sheet tab.  
  - Configuration:  
    - Operation: Append  
    - Document ID and Sheet Name: Dynamically taken from config node  
    - Columns: Auto-mapped from input data fields to sheet columns (date, account_id, impressions, CPL, ROAS, etc.)  
  - Inputs: Transformed data from prior node  
  - Outputs: None (end of flow)  
  - Edge cases: Google Sheets API quota limits, incorrect sheet ID or permissions, schema mismatch with sheet headers.

---

#### 2.2 Manual Historical Backfill Block

**Overview:**  
This block allows manual execution to backfill Meta Ads campaign data for custom historical date ranges. It uses the Facebook Insights API `time_range` parameter to fetch data over the specified date window, transform it, and append it to the same Google Sheet.

**Nodes Involved:**  
- Manual backfill trigger  
- Set backfill config  
- Fetch Meta Insights (time_range)  
- Transform backfill data for Sheets  
- Append backfill rows to Google Sheet  

**Node Details:**  

- **Manual backfill trigger:**  
  - Type: Manual Trigger  
  - Role: Entry point for manual runs to backfill historical data.  
  - Configuration: None (user triggers execution manually)  
  - Inputs: None  
  - Outputs: To "Set backfill config"  
  - Edge cases: User must run manually; no scheduling.  

- **Set backfill config:**  
  - Type: Set  
  - Role: Defines parameters for the backfill fetch, including ad account, sheet ID, sheet tab, and crucially `backfillSince` and `backfillUntil` dates for the data window.  
  - Configuration:  
    - `adAccountId`: Meta ad account string placeholder  
    - `level`: `campaign`  
    - `timeIncrement`: `1` (daily granularity)  
    - `sheetId`: Google Sheet document ID placeholder  
    - `sheetNameData`: Sheet name (`Meta_Daily_Data`)  
    - `backfillSince`: Start date string (e.g., "2024-01-01")  
    - `backfillUntil`: End date string (e.g., "2024-03-31")  
  - Inputs: Manual trigger  
  - Outputs: To "Fetch Meta Insights (time_range)"  
  - Edge cases: Overlapping date ranges may cause duplicate rows; date format must be valid.  

- **Fetch Meta Insights (time_range):**  
  - Type: Facebook Graph API Node  
  - Role: Calls Meta Ads Insights API with `time_range` set to the backfill dates, fetching campaign-level data in the range.  
  - Configuration:  
    - Edge: `insights`  
    - Node (ad account): from previous node  
    - Fields: Similar to daily fetch, includes `platform` field additionally  
    - Query parameters include:  
      - level: campaign  
      - time_increment: 1  
      - limit: 1000  
      - action_attribution_windows: 1d_click, 7d_click, 1d_view  
      - time_range: JSON string with `since` and `until` from backfill config  
    - Graph API version: v23.0  
  - Inputs: Backfill config node  
  - Outputs: To "Transform backfill data for Sheets"  
  - Edge cases: Long date ranges may cause large data sets and API pagination issues; API limits and auth errors possible.  

- **Transform backfill data for Sheets:**  
  - Type: Code (JavaScript)  
  - Role: Same logic as daily transform; flattens and normalizes data, computes KPIs.  
  - Inputs: API response from backfill fetch  
  - Outputs: Transformed data for Google Sheets  
  - Edge cases: Handles empty or missing data gracefully.  

- **Append backfill rows to Google Sheet:**  
  - Type: Google Sheets Node  
  - Role: Appends historical backfill data rows to the Google Sheet tab specified (same as daily)  
  - Configuration:  
    - Operation: Append  
    - Document ID and Sheet Name: From backfill config node  
    - Columns: Auto-mapped to sheet columns, matching daily export  
  - Inputs: Transformed backfill data  
  - Outputs: None  
  - Edge cases: Duplicate data risk if overlapping backfill ranges; Google Sheets API limits.

---

#### 2.3 Informational Sticky Notes

- **Sticky Note (Manual Backfill):** Provides instructions on how to use the manual backfill block, emphasizing careful date range selection and avoiding overlaps to prevent duplicates.

- **Sticky Note1 (Daily Incremental Export):** Describes the daily export flow, setup hints, and reminders to configure sheet headers and credentials.

- **Sticky Note2 (General Overview):** Summarizes the workflow purpose, target audience, how it works, and Google Sheets setup requirements.

- **Sticky Note3 (Support Info):** Contains a link for professional help with ecommerce, marketing, or analytics setups related to this workflow: [Serendipity Technologies](https://www.serendipity.at)

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                               | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                          |
|-------------------------------|---------------------------|-----------------------------------------------|-----------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| Daily schedule (06:00)         | Schedule Trigger          | Triggers daily incremental export at 06:00  | None                        | Set config (Meta + Sheets)        | See Sticky Note1 for detailed daily export flow and setup                                         |
| Set config (Meta + Sheets)     | Set                       | Configures parameters for daily fetch        | Daily schedule (06:00)       | Fetch Meta Insights (yesterday)   | See Sticky Note1 and Sticky Note2                                                                 |
| Fetch Meta Insights (yesterday)| Facebook Graph API        | Fetches Meta Ads insights for yesterday      | Set config (Meta + Sheets)   | Transform Meta data for Sheets     | API call with campaign-level metrics, uses Facebook Graph credentials                             |
| Transform Meta data for Sheets  | Code                      | Flattens API data, computes KPIs              | Fetch Meta Insights (yesterday)| Append daily rows to Google Sheet | Handles possible missing data, calculates CPL, CPA, ROAS                                          |
| Append daily rows to Google Sheet | Google Sheets             | Appends daily transformed data to Google Sheet| Transform Meta data for Sheets| None                            | Requires correct sheet ID and headers matching columns                                            |
| Manual backfill trigger         | Manual Trigger            | Starts manual backfill for historical data    | None                        | Set backfill config               | See Sticky Note for manual backfill instructions                                                  |
| Set backfill config             | Set                       | Sets parameters including custom date range for backfill | Manual backfill trigger      | Fetch Meta Insights (time_range)  | Dates must be carefully defined to avoid duplicate data                                           |
| Fetch Meta Insights (time_range)| Facebook Graph API        | Fetches Meta Ads insights for custom date range | Set backfill config          | Transform backfill data for Sheets | Uses time_range parameter to fetch historical data                                                |
| Transform backfill data for Sheets | Code                      | Transforms and computes KPIs for backfill data | Fetch Meta Insights (time_range) | Append backfill rows to Google Sheet | Same logic as daily transform, handles large data sets                                            |
| Append backfill rows to Google Sheet | Google Sheets             | Appends backfill data to Google Sheet          | Transform backfill data for Sheets | None                            | Same sheet and columns as daily append                                                           |
| Sticky Note                    | Sticky Note               | Explains manual backfill usage                 | None                        | None                             | Content about manual backfill usage and precautions                                              |
| Sticky Note1                   | Sticky Note               | Explains daily incremental export              | None                        | None                             | Content about daily export flow and setup                                                        |
| Sticky Note2                   | Sticky Note               | General overview and instructions              | None                        | None                             | Workflow purpose, audience, setup instructions                                                   |
| Sticky Note3                   | Sticky Note               | Support & consultancy link                       | None                        | None                             | Link to [Serendipity Technologies](https://www.serendipity.at)                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node: Schedule Trigger**  
   - Type: Schedule Trigger  
   - Set to run once daily at 06:00 (hour 6, no minutes specified)  
   - This triggers the daily incremental data export.  

2. **Create Node: Set (Daily Config)**  
   - Type: Set  
   - Assign values:  
     - `adAccountId`: `"act_<YOUR_AD_ACCOUNT_ID>"` (replace with your Meta ad account ID)  
     - `datePreset`: `"yesterday"`  
     - `level`: `"campaign"`  
     - `timeIncrement`: `"1"`  
     - `sheetId`: `"YOUR_SHEET_ID"` (replace with your Google Sheet ID)  
     - `sheetNameData`: `"Meta_Daily_Data"` (or your preferred sheet tab)  
   - Connect Schedule Trigger output to this node.  

3. **Create Node: Facebook Graph API (Fetch Meta Insights - Daily)**  
   - Type: Facebook Graph API  
   - Credentials: Use configured Facebook Graph API OAuth credentials.  
   - Parameters:  
     - Edge: `insights`  
     - Node: `={{ $json["adAccountId"] }}` (dynamic from Set node)  
     - Fields: date_start, date_stop, account_id, account_name, campaign_id, campaign_name, objective, adset_id, adset_name, ad_id, ad_name, impressions, reach, clicks, inline_link_clicks, spend, ctr, cpc, cpm, actions, action_values, frequency, publisher_platform  
     - Query parameters:  
       - `level`: `={{ $json["level"] }}`  
       - `date_preset`: `={{ $json["datePreset"] }}`  
       - `time_increment`: `={{ $json["timeIncrement"] }}`  
       - `limit`: 1000  
       - `action_attribution_windows`: `["1d_click","7d_click","1d_view"]`  
       - `breakdowns`: `publisher_platform`  
     - API Version: v23.0  
   - Connect Set node output to this node.  

4. **Create Node: Code (Transform Meta Data for Sheets - Daily)**  
   - Type: Code  
   - JavaScript code to iterate over API response data, flatten, extract metrics, compute CPL, CPA, ROAS.  
   - See provided code in workflow (adapted for daily data).  
   - Connect Facebook Graph API node output to this node.  

5. **Create Node: Google Sheets (Append Daily Rows)**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2 credentials configured.  
   - Operation: Append  
   - Document ID: Dynamic from Set config node `sheetId`  
   - Sheet Name: Dynamic from Set config node `sheetNameData`  
   - Columns: Auto-map input JSON fields to sheet columns. Ensure the Google Sheet has headers matching field names:  
     `date`, `account_id`, `account_name`, `publisher_platform`, `campaign_id`, `campaign_name`, `objective`, `adset_id`, `adset_name`, `ad_id`, `ad_name`, `impressions`, `reach`, `frequency`, `spend`, `clicks`, `inline_link_clicks`, `ctr`, `cpc`, `cpm`, `leads`, `on_facebook_lead`, `purchases`, `purchase_value`, `add_to_cart`, `initiate_checkout`, `cpl`, `cpa`, `roas`  
   - Connect Code node output to this node.  

6. **Create Node: Manual Trigger (Backfill Trigger)**  
   - Type: Manual Trigger  
   - Used to start manual backfill of historical data.  

7. **Create Node: Set (Backfill Config)**  
   - Type: Set  
   - Assign values:  
     - `adAccountId`: `"act_<YOUR_AD_ACCOUNT_ID>"`  
     - `level`: `"campaign"`  
     - `timeIncrement`: `"1"`  
     - `sheetId`: `"YOUR_SHEET_ID"`  
     - `sheetNameData`: `"Meta_Daily_Data"`  
     - `backfillSince`: e.g., `"2024-01-01"` (start date for backfill)  
     - `backfillUntil`: e.g., `"2024-03-31"` (end date for backfill)  
   - Connect Manual Trigger output to this node.  

8. **Create Node: Facebook Graph API (Fetch Meta Insights - Time Range)**  
   - Type: Facebook Graph API  
   - Credentials: Same as above.  
   - Parameters:  
     - Edge: `insights`  
     - Node: `={{ $json["adAccountId"] }}`  
     - Fields: Same as daily fetch plus `platform`  
     - Query parameters:  
       - level: `={{ $json["level"] }}`  
       - time_increment: `={{ $json["timeIncrement"] }}`  
       - limit: 1000  
       - action_attribution_windows: `["1d_click","7d_click","1d_view"]`  
       - time_range: `={{ '{"since":"' + $json["backfillSince"] + '","until":"' + $json["backfillUntil"] + '"}' }}`  
     - API Version: v23.0  
   - Connect Set backfill config output to this node.  

9. **Create Node: Code (Transform Backfill Data for Sheets)**  
   - Type: Code  
   - Same JavaScript logic as daily transform.  
   - Connect Facebook Graph API backfill node output to this node.  

10. **Create Node: Google Sheets (Append Backfill Rows)**  
    - Type: Google Sheets  
    - Credentials: Same as daily append node.  
    - Operation: Append  
    - Document ID and Sheet Name: From backfill config node  
    - Columns: Same mapping as daily append node.  
    - Connect Code backfill transform node output to this node.  

11. **Add Sticky Notes**  
    - Add informational sticky notes at appropriate positions describing:  
      - Manual backfill usage and precautions  
      - Daily incremental export details  
      - General overview and setup instructions  
      - Support link to Serendipity Technologies  

**Credential Setup:**  
- Facebook Graph API: OAuth2 credentials with necessary permissions to read Ads Insights for the ad account.  
- Google Sheets OAuth2: Credentials authorized to edit the target Google Sheet.  

**Default Values & Constraints:**  
- Replace placeholder values (`<YOUR_AD_ACCOUNT_ID>`, `YOUR_SHEET_ID`) before enabling the workflow.  
- Google Sheet must contain a header row exactly matching the column names used in the workflow.  
- Avoid overlapping date ranges in manual backfill to prevent duplicate entries.  
- The daily schedule must be enabled for automated daily exports.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                              | Context or Link                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Before running, add a header row to your Google Sheet with the following columns: `date | account_id | account_name | publisher_platform | campaign_id | campaign_name | objective | adset_id | adset_name | ad_id | ad_name | impressions | reach | frequency | spend | clicks | inline_link_clicks | ctr | cpc | cpm | leads | on_facebook_lead | purchases | purchase_value | add_to_cart | initiate_checkout | cpl | cpa | roas`                         | Sheet setup requirement                             |
| Workflow target audience includes Meta/Facebook Ads managers, performance marketers, agencies, and ecommerce businesses who want clean daily campaign data for dashboards and analysis.                                                                                                                                      | Sticky Note2 overview                              |
| Use the Google Sheets output as a data source for Looker Studio, Power BI, or Excel dashboards for further analysis and visualization.                                                                                                                                                                                     | Sticky Note2 overview                              |
| For professional help with ecommerce, digital marketing, or data & analytics setups based on this workflow, visit: [Serendipity Technologies](https://www.serendipity.at)                                                                                                                                                      | Sticky Note3 support link                          |
| When performing manual backfill, run the workflow once per desired time block (e.g. quarterly), and avoid overlapping date ranges to prevent duplicate data rows.                                                                                                                                                           | Sticky Note (Manual Backfill)                       |

---

_Disclaimer: The provided text is exclusively generated from an automated n8n workflow and complies strictly with content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public._