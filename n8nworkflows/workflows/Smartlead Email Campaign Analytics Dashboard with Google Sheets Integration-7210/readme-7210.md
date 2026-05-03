Smartlead Email Campaign Analytics Dashboard with Google Sheets Integration

https://n8nworkflows.xyz/workflows/smartlead-email-campaign-analytics-dashboard-with-google-sheets-integration-7210


# Smartlead Email Campaign Analytics Dashboard with Google Sheets Integration

### 1. Workflow Overview

This workflow, titled **Smartlead Email Campaign Analytics Dashboard with Google Sheets Integration**, automates the process of fetching, processing, and storing email campaign and domain health analytics data from the Smartlead API into Google Sheets. It enables users to maintain up-to-date analytics dashboards without manual entry, supporting better decision-making and monitoring of email outreach campaigns.

The workflow is logically divided into the following functional blocks:

- **1.1 Scheduled Data Fetching**: Two independent schedule triggers that initiate the data retrieval process at different intervals.
- **1.2 API Integration and Data Retrieval**: HTTP Request nodes that query various Smartlead analytics endpoints for detailed metrics.
- **1.3 Data Splitting and Transformation**: Split Out nodes that flatten API response arrays to process individual metric records.
- **1.4 Google Sheets Integration**: Append or update row nodes that insert or update the processed analytics data in designated Google Sheets tabs.
- **1.5 Ancillary Notes and Documentation**: Sticky Note nodes providing contextual information and setup help links.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Fetching

**Overview:**  
This block triggers the workflow executions on a recurring basis to ensure analytics data remains current.

**Nodes Involved:**  
- Schedule Trigger  
- Schedule Trigger1

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates data fetch every 2 hours for campaign and day-wise stats.  
  - *Configuration:* Interval set to 2 hours.  
  - *Input/Output:* No input; outputs to two HTTP Request nodes (HTTP Request, HTTP Request1).  
  - *Edge Cases:* Trigger misfires if n8n server is down or time drift occurs.

- **Schedule Trigger1**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates data fetch every 5 hours for mailbox domain and name-wise health metrics.  
  - *Configuration:* Interval set to 5 hours.  
  - *Input/Output:* No input; outputs to two HTTP Request nodes (HTTP Request5, HTTP Request6).  
  - *Edge Cases:* Similar to Schedule Trigger.

---

#### 1.2 API Integration and Data Retrieval

**Overview:**  
This block queries the Smartlead API endpoints to retrieve various analytics data: day-wise stats, campaign-wise performance, domain health metrics, and email health metrics.

**Nodes Involved:**  
- HTTP Request  
- HTTP Request1  
- HTTP Request5  
- HTTP Request6  
- HTTP Request9 (not present, so ignored)

**Node Details:**

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Fetches day-wise overall stats by sent time from Smartlead API.  
  - *Configuration:*  
    - URL: `https://server.smartlead.ai/api/v1/analytics/day-wise-overall-stats-by-sent-time`  
    - Query parameters include `api_key` (placeholder `your_API_KEY`), `start_date` fixed as `2025-08-7`, `end_date` dynamic based on current day of month extracted from Schedule Trigger JSON, and timezone offset `5:30`.  
  - *Input/Output:* Input from Schedule Trigger; outputs JSON containing `data.day_wise_stats`.  
  - *Edge Cases:* API key missing/invalid, network timeout, date expression failures.

- **HTTP Request1**  
  - *Type:* HTTP Request  
  - *Role:* Fetches overall campaign-wise performance stats.  
  - *Configuration:*  
    - URL: `https://server.smartlead.ai/api/v1/analytics/campaign/overall-stats`  
    - Query parameters similar to above with dynamic end_date.  
  - *Input/Output:* Input from Schedule Trigger; outputs JSON containing `data.campaign_wise_performance`.  
  - *Edge Cases:* Same as HTTP Request.

- **HTTP Request5**  
  - *Type:* HTTP Request  
  - *Role:* Fetches mailbox domain-wise health metrics.  
  - *Configuration:*  
    - URL: `https://server.smartlead.ai/api/v1/analytics/mailbox/domain-wise-health-metrics`  
    - Query parameters include dynamic `end_date`.  
  - *Input/Output:* Input from Schedule Trigger1; outputs JSON containing `data.domain_health_metrics`.  
  - *Edge Cases:* Same as above.

- **HTTP Request6**  
  - *Type:* HTTP Request  
  - *Role:* Fetches mailbox name-wise health metrics.  
  - *Configuration:*  
    - URL: `https://server.smartlead.ai/api/v1/analytics/mailbox/name-wise-health-metrics`  
    - Query parameters include dynamic `end_date`.  
  - *Input/Output:* Input from Schedule Trigger1; outputs JSON containing `data.email_health_metrics`.  
  - *Edge Cases:* Same as above.

---

#### 1.3 Data Splitting and Transformation

**Overview:**  
This block splits the arrays of analytics data received from API calls into individual items for row-wise processing.

**Nodes Involved:**  
- Split Out  
- Split Out1  
- Split Out5  
- Split Out6

**Node Details:**

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits `data.day_wise_stats` array into individual records.  
  - *Configuration:* Field to split: `data.day_wise_stats`.  
  - *Input:* From HTTP Request.  
  - *Output:* To Append or update row in sheet.  
  - *Edge Cases:* Empty or malformed array may cause no output.

- **Split Out1**  
  - *Type:* Split Out  
  - *Role:* Splits `data.campaign_wise_performance` array into individual campaign records.  
  - *Configuration:* Field to split: `data.campaign_wise_performance`.  
  - *Input:* From HTTP Request1.  
  - *Output:* To Append or update row in sheet1.  
  - *Edge Cases:* Same as above.

- **Split Out5**  
  - *Type:* Split Out  
  - *Role:* Splits `data.domain_health_metrics` array into individual domain health records.  
  - *Configuration:* Field to split: `data.domain_health_metrics`.  
  - *Input:* From HTTP Request5.  
  - *Output:* To Append or update row in sheet5.  
  - *Edge Cases:* Same as above.

- **Split Out6**  
  - *Type:* Split Out  
  - *Role:* Splits `data.email_health_metrics` array into individual email health records.  
  - *Configuration:* Field to split: `data.email_health_metrics`.  
  - *Input:* From HTTP Request6.  
  - *Output:* To Append or update row in sheet6.  
  - *Edge Cases:* Same as above.

---

#### 1.4 Google Sheets Integration

**Overview:**  
This block appends or updates rows in specific Google Sheets tabs to store the analytics data per record.

**Nodes Involved:**  
- Append or update row in sheet  
- Append or update row in sheet1  
- Append or update row in sheet5  
- Append or update row in sheet6

**Node Details:**

- **Append or update row in sheet**  
  - *Type:* Google Sheets  
  - *Role:* Inserts or updates day-wise stats in "Sheet1" tab of a Google Sheet document.  
  - *Configuration:*  
    - Document ID: Spreadsheet ID `1U8CC_vlv80vZeeDJc41YTEiWA-efikXBur6XDurtQoQ`  
    - Sheet Name: `Sheet1` (gid=0)  
    - Matching column: `date` (to update existing rows)  
    - Columns mapped: day, date, sent, opened, bounced, replied, unsubscribed, unique_lead_reached  
  - *Input:* From Split Out node.  
  - *Output:* None.  
  - *Edge Cases:* Google API auth failure, rate limits, schema mismatch.

- **Append or update row in sheet1**  
  - *Type:* Google Sheets  
  - *Role:* Inserts or updates campaign-wise stats in "CampaignWise" tab.  
  - *Configuration:*  
    - Document ID: Same as above  
    - Sheet Name: `CampaignWise` (gid=476539810)  
    - Matching column: `id` (unique campaign ID)  
    - Columns mapped: campaign_name, sent, opened, bounced, replied, etc.  
  - *Input:* From Split Out1 node.  
  - *Output:* None.  
  - *Edge Cases:* Same as above.

- **Append or update row in sheet5**  
  - *Type:* Google Sheets  
  - *Role:* Inserts or updates domain health metrics in "DomainHealth" tab.  
  - *Configuration:*  
    - Document ID: Same as above  
    - Sheet Name: `DomainHealth` (gid=1430156318)  
    - Matching column: `domain`  
    - Columns: sent, domain, opened, bounced, replied, open_rate, reply_rate, bounce_rate, positive_replied, unique_lead_count, unique_open_count, Last_update (date string from Schedule Trigger1)  
  - *Input:* From Split Out5 node.  
  - *Output:* None.  
  - *Edge Cases:* Same as above.

- **Append or update row in sheet6**  
  - *Type:* Google Sheets  
  - *Role:* Inserts or updates email health metrics in "EmailHealth" tab.  
  - *Configuration:*  
    - Document ID: Same as above  
    - Sheet Name: `EmailHealth` (gid=1649535361)  
    - Matching column: `from_email`  
    - Columns mapped: sent, opened, bounced, replied, open_rate, reply_rate, bounce_rate, positive_replied, unique_lead_count, unique_open_count, Last_update (from Schedule Trigger1)  
  - *Input:* From Split Out6 node.  
  - *Output:* None.  
  - *Edge Cases:* Same as above.

---

#### 1.5 Ancillary Notes and Documentation

**Overview:**  
Sticky Notes provide helpful context and contact information for workflow setup assistance.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**

- **Sticky Note**  
  - *Role:* Notes that this workflow is for getting data about email and domain health.  
  - *Position:* Visually covers part of the workflow near domain and email health nodes.

- **Sticky Note1**  
  - *Role:* Describes the workflow as Smartlead Campaignwise & Global analytics day wise.  
  - *Position:* Near campaign and day-wise analytics nodes.

- **Sticky Note2**  
  - *Role:* Provides contact link for help: [https://www.linkedin.com/in/rahiuppal/](https://www.linkedin.com/in/rahiuppal/)  
  - *Color:* Custom color (color index 5) for emphasis.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                                   | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                  |
|-----------------------------|---------------------|--------------------------------------------------|-------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger     | Triggers every 2 hours to fetch campaign & day-wise stats | None                    | HTTP Request, HTTP Request1 |                                                                                              |
| HTTP Request                | HTTP Request        | Fetches day-wise overall stats                    | Schedule Trigger        | Split Out                   |                                                                                              |
| Split Out                  | Split Out           | Splits day-wise stats array to individual records | HTTP Request            | Append or update row in sheet |                                                                                              |
| Append or update row in sheet | Google Sheets       | Updates/inserts day-wise stats into Sheet1 tab    | Split Out               | None                        |                                                                                              |
| HTTP Request1               | HTTP Request        | Fetches campaign-wise overall stats               | Schedule Trigger        | Split Out1                  |                                                                                              |
| Split Out1                 | Split Out           | Splits campaign-wise performance array             | HTTP Request1           | Append or update row in sheet1 |                                                                                              |
| Append or update row in sheet1 | Google Sheets       | Updates/inserts campaign stats into CampaignWise tab | Split Out1              | None                        |                                                                                              |
| Schedule Trigger1           | Schedule Trigger     | Triggers every 5 hours to fetch mailbox domain & name-wise metrics | None                    | HTTP Request5, HTTP Request6 |                                                                                              |
| HTTP Request5               | HTTP Request        | Fetches domain-wise health metrics                 | Schedule Trigger1       | Split Out5                  |                                                                                              |
| Split Out5                 | Split Out           | Splits domain health metrics array                  | HTTP Request5           | Append or update row in sheet5 |                                                                                              |
| Append or update row in sheet5 | Google Sheets       | Updates/inserts domain health metrics into DomainHealth tab | Split Out5              | None                        |                                                                                              |
| HTTP Request6               | HTTP Request        | Fetches name-wise health metrics                    | Schedule Trigger1       | Split Out6                  |                                                                                              |
| Split Out6                 | Split Out           | Splits email health metrics array                    | HTTP Request6           | Append or update row in sheet6 |                                                                                              |
| Append or update row in sheet6 | Google Sheets       | Updates/inserts email health metrics into EmailHealth tab | Split Out6              | None                        |                                                                                              |
| Sticky Note                | Sticky Note          | Notes on purpose: Getting data about email and domain health | None                    | None                        | For Getting data about email and domain health                                               |
| Sticky Note1               | Sticky Note          | Describes workflow: Smartlead Campaignwise & Global analytics day wise | None                    | None                        | Smartlead Campaignwise & Global analytics day wise                                          |
| Sticky Note2               | Sticky Note          | Setup help contact link                            | None                    | None                        | For help in setting up these workflows, connect me at https://www.linkedin.com/in/rahiuppal/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create two Schedule Trigger nodes:**  
   - **Schedule Trigger:** Set interval to every 2 hours.  
   - **Schedule Trigger1:** Set interval to every 5 hours.

2. **Create four HTTP Request nodes with credentials and parameters:**  
   - **HTTP Request:**  
     - URL: `https://server.smartlead.ai/api/v1/analytics/day-wise-overall-stats-by-sent-time`  
     - Query Parameters:  
       - `api_key`: Your Smartlead API key  
       - `start_date`: Fixed `2025-08-7` (adjust as needed)  
       - `end_date`: Expression `=2025-08-{{ $json['Day of month'] }}` from Schedule Trigger  
       - `timezone`: `5:30`  
     - Connect output from Schedule Trigger.

   - **HTTP Request1:**  
     - URL: `https://server.smartlead.ai/api/v1/analytics/campaign/overall-stats`  
     - Query Parameters similar to above, with `full_data=true`.  
     - Connect output from Schedule Trigger.

   - **HTTP Request5:**  
     - URL: `https://server.smartlead.ai/api/v1/analytics/mailbox/domain-wise-health-metrics`  
     - Query Parameters:  
       - `api_key`: Your API key  
       - `start_date`: Fixed `2025-07-04`  
       - `end_date`: Expression `=2025-08-{{ $json['Day of month'] }}` from Schedule Trigger1  
       - `full_data`: `true`  
     - Connect output from Schedule Trigger1.

   - **HTTP Request6:**  
     - URL: `https://server.smartlead.ai/api/v1/analytics/mailbox/name-wise-health-metrics`  
     - Query Parameters same as HTTP Request5.  
     - Connect output from Schedule Trigger1.

3. **Create four Split Out nodes to flatten data arrays:**  
   - **Split Out:** Field to split: `data.day_wise_stats` from HTTP Request.  
   - **Split Out1:** Field to split: `data.campaign_wise_performance` from HTTP Request1.  
   - **Split Out5:** Field to split: `data.domain_health_metrics` from HTTP Request5.  
   - **Split Out6:** Field to split: `data.email_health_metrics` from HTTP Request6.

4. **Create four Google Sheets nodes configured for Append or Update operation:**  
   - **Append or update row in sheet:**  
     - Document ID: Spreadsheet ID `1U8CC_vlv80vZeeDJc41YTEiWA-efikXBur6XDurtQoQ`  
     - Sheet Name: `Sheet1` (gid=0)  
     - Matching Column: `date`  
     - Map columns: day, date, sent, opened, bounced, replied, unsubscribed, unique_lead_reached  
     - Connect input from Split Out.

   - **Append or update row in sheet1:**  
     - Document ID: Same as above  
     - Sheet Name: `CampaignWise` (gid=476539810)  
     - Matching Column: `id`  
     - Map columns: id, campaign_name, sent, opened, bounced, replied, open_rate, reply_rate, bounce_rate, positive_reply_rate, unique_lead_count, unique_open_count, positive_replied, Updated at (dynamic date from Schedule Trigger JSON)  
     - Connect input from Split Out1.

   - **Append or update row in sheet5:**  
     - Document ID: Same  
     - Sheet Name: `DomainHealth` (gid=1430156318)  
     - Matching Column: `domain`  
     - Map columns: sent, domain, opened, bounced, replied, open_rate, reply_rate, bounce_rate, positive_replied, unique_lead_count, unique_open_count, Last_update (formatted date from Schedule Trigger1 JSON)  
     - Connect input from Split Out5.

   - **Append or update row in sheet6:**  
     - Document ID: Same  
     - Sheet Name: `EmailHealth` (gid=1649535361)  
     - Matching Column: `from_email`  
     - Map columns: sent, opened, bounced, replied, open_rate, reply_rate, bounce_rate, positive_replied, unique_lead_count, unique_open_count, Last_update (from Schedule Trigger1)  
     - Connect input from Split Out6.

5. **Add Sticky Note nodes for documentation at positions corresponding to your layout:**  
   - Note about email and domain health data purpose.  
   - Note about Smartlead Campaignwise & Global analytics day wise.  
   - Note with help contact link: https://www.linkedin.com/in/rahiuppal/

6. **Set credentials:**  
   - Configure Google Sheets credentials with required scopes for read/write on the specified spreadsheet.  
   - Set HTTP Request nodes with appropriate API key in query parameters or use n8n credentials management for API keys.

7. **Test workflow:**  
   - Run triggers manually or wait for scheduled execution.  
   - Verify data retrieval, splitting, and Google Sheets update correctness.

---

### 5. General Notes & Resources

| Note Content                                                          | Context or Link                                              |
|-----------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow integrates Smartlead API analytics with Google Sheets.  | Overview purpose                                              |
| For help in setting up these workflows, connect with the author: https://www.linkedin.com/in/rahiuppal/ | Sticky Note2 providing contact for support                    |
| The Google Spreadsheet used is: https://docs.google.com/spreadsheets/d/1U8CC_vlv80vZeeDJc41YTEiWA-efikXBur6XDurtQoQ | Spreadsheet URL referenced by nodes                           |

---

**Disclaimer:** The content provided is derived solely from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.