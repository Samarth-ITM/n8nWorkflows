Facebook Ads Reporting Automation with Facebook Graph API and Google Sheets

https://n8nworkflows.xyz/workflows/facebook-ads-reporting-automation-with-facebook-graph-api-and-google-sheets-6328


# Facebook Ads Reporting Automation with Facebook Graph API and Google Sheets

### 1. Workflow Overview

This workflow automates the reporting of Facebook Ads campaign performance by integrating the Facebook Graph API with Google Sheets. It is designed to periodically fetch active campaigns and their statistics from a Facebook Ads account, process and aggregate the data, and update a Google Sheet to maintain an up-to-date report.

Logical blocks:

- **1.1 Schedule Triggers:** Initiate the workflow at defined intervals.
- **1.2 Facebook API Authentication and Campaign Retrieval:** Authenticate with Facebook Graph API and fetch campaigns from the ad account.
- **1.3 Campaign Filtering and Storage:** Filter active campaigns and store campaign data in Google Sheets.
- **1.4 Campaign Statistics Retrieval and Validation:** Retrieve campaign statistics, check for data presence, and handle campaign data accordingly.
- **1.5 Data Processing and Aggregation:** Process conversions and aggregate statistics.
- **1.6 Update Google Sheets with Campaign Statistics:** Write the aggregated data back into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Triggers

**Overview:**  
Two schedule trigger nodes initiate the workflow at scheduled times: one for fetching campaigns and another for updating campaign statistics.

**Nodes Involved:**  
- Set Schedule  
- Set Schedule1

**Node Details:**

- **Set Schedule**  
  - Type: Schedule Trigger  
  - Configuration: Trigger set to run at specific intervals (not detailed in JSON)  
  - Outputs to: Add credentials to the Facebook Graph API  
  - Failure modes: Missed schedules if n8n service is down; misconfigured cron expressions can cause trigger failures.

- **Set Schedule1**  
  - Type: Schedule Trigger  
  - Configuration: Another schedule trigger also set to run at intervals (likely offset or different schedule)  
  - Outputs to: Get Campaign ID  
  - Failure modes: Same as above.

---

#### 1.2 Facebook API Authentication and Campaign Retrieval

**Overview:**  
This block authenticates to the Facebook Graph API and fetches campaigns from the specified ad account.

**Nodes Involved:**  
- Add credentials to the Facebook Graph API  
- Get Campaign from AD Account  
- Code1  
- Get Only Active Campaign  
- Add your campaign to file

**Node Details:**

- **Add credentials to the Facebook Graph API**  
  - Type: Facebook Graph API node  
  - Role: Handles OAuth2 or token-based authentication for Facebook API requests  
  - Outputs to: Get Campaign from AD Account  
  - Failure modes: Authentication errors, expired tokens, permission issues.

- **Get Campaign from AD Account**  
  - Type: HTTP Request  
  - Role: Calls Facebook Graph API endpoint to retrieve campaigns associated with the ad account  
  - Input: Authenticated context from previous node  
  - Output: Campaign data JSON  
  - Failure modes: API rate limits, network errors, invalid ad account ID.

- **Code1**  
  - Type: Code (JavaScript)  
  - Role: Processes raw campaign data, likely for formatting or filtering (details not specified)  
  - Input: Campaign JSON data  
  - Output: Processed campaign data  
  - Failure modes: Script errors, unexpected data formats.

- **Get Only Active Campaign**  
  - Type: If Condition  
  - Role: Filters campaigns to include only those with active status  
  - Input: Processed campaign data  
  - Output: Active campaigns  
  - Failure modes: Logic errors if campaign status fields are missing or changed.

- **Add your campaign to file**  
  - Type: Google Sheets  
  - Role: Adds or updates campaign details into a Google Sheet file for record keeping  
  - Input: Active campaign list  
  - Output: Confirmation or updated rows  
  - Failure modes: Google Sheets API limits, authentication failures, write conflicts.

---

#### 1.3 Campaign Filtering and Storage

**Overview:**  
This block ensures only active campaigns are processed and saved for reporting.

**Nodes Involved:**  
- Get Only Active Campaign  
- Add your campaign to file

**Details:**  
Already covered above; these nodes form a sub-block focused on filtering and persisting campaign metadata.

---

#### 1.4 Campaign Statistics Retrieval and Validation

**Overview:**  
Using the stored campaign IDs, this block fetches detailed statistics, checks if campaigns have data, and routes them based on data presence.

**Nodes Involved:**  
- Get Campaign ID  
- Get Campaign statistics  
- Checking if Campaign have data  
- Update row in sheet4  
- Checking Conversion

**Node Details:**

- **Get Campaign ID**  
  - Type: Google Sheets  
  - Role: Reads campaign IDs from the Google Sheet to query statistics  
  - Output: List of campaign IDs  
  - Failure modes: Sheet access issues, missing data.

- **Get Campaign statistics**  
  - Type: HTTP Request  
  - Role: Calls Facebook Graph API to retrieve detailed statistics for each campaign ID  
  - Input: Campaign IDs  
  - Output: Statistics data  
  - Failure modes: API errors, missing permissions, invalid IDs.

- **Checking if Campaign have data**  
  - Type: If Condition  
  - Role: Determines if statistics data is present for each campaign  
  - Output True: Update row in sheet4 (update with data)  
  - Output False: Checking Conversion (possibly some alternative processing)  
  - Failure modes: Logic errors if data format changes.

- **Update row in sheet4**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet with campaign statistics when data is present  
  - Input: Valid campaign statistics  
  - Failure modes: API write errors, concurrent updates.

- **Checking Conversion**  
  - Type: Code (JavaScript)  
  - Role: Processes conversion-related data from campaign statistics, possibly for further aggregation  
  - Output: Merged with other data flows for aggregation  
  - Failure modes: Script errors, unexpected data structures.

---

#### 1.5 Data Processing and Aggregation

**Overview:**  
This block merges processed data streams and aggregates campaign statistics before final update.

**Nodes Involved:**  
- Merge  
- Aggregate  
- Update Campaign statistics

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Combines multiple input data streams (from Checking Conversion and Get Campaign ID) for unified processing  
  - Input: Multiple data branches  
  - Output: Merged dataset  
  - Failure modes: Data mismatch, empty inputs.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Performs aggregation functions (sum, average, count) on merged campaign data to prepare summary statistics  
  - Input: Merged data  
  - Output: Aggregated results  
  - Failure modes: Incorrect aggregation parameters, empty data sets.

- **Update Campaign statistics**  
  - Type: Google Sheets  
  - Role: Writes aggregated statistics back to the Google Sheet for reporting  
  - Input: Aggregated data  
  - Output: Confirmation of update  
  - Failure modes: API write limits, authentication issues.

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                               | Input Node(s)              | Output Node(s)                | Sticky Note                  |
|----------------------------|-----------------------|-----------------------------------------------|----------------------------|------------------------------|------------------------------|
| Set Schedule               | Schedule Trigger      | Initiates workflow for campaign retrieval     |                            | Add credentials to the Facebook Graph API |                              |
| Add credentials to the Facebook Graph API | Facebook Graph API   | Authenticates Facebook API requests           | Set Schedule              | Get Campaign from AD Account  |                              |
| Get Campaign from AD Account| HTTP Request          | Retrieves campaigns from Facebook Ads account | Add credentials to the Facebook Graph API | Code1                       |                              |
| Code1                      | Code                  | Processes raw campaign data                    | Get Campaign from AD Account | Get Only Active Campaign      |                              |
| Get Only Active Campaign    | If Condition          | Filters active campaigns                        | Code1                      | Add your campaign to file     |                              |
| Add your campaign to file   | Google Sheets         | Stores active campaign data into Google Sheets| Get Only Active Campaign    |                              |                              |
| Set Schedule1              | Schedule Trigger      | Initiates workflow for updating campaign stats|                            | Get Campaign ID              |                              |
| Get Campaign ID            | Google Sheets         | Reads campaign IDs to fetch statistics        | Set Schedule1              | Get Campaign statistics, Merge |                              |
| Get Campaign statistics    | HTTP Request          | Fetches detailed stats for campaigns           | Get Campaign ID            | Checking if Campaign have data|                              |
| Checking if Campaign have data | If Condition       | Checks presence of stats data                   | Get Campaign statistics    | Update row in sheet4 (True), Checking Conversion (False) |                              |
| Update row in sheet4       | Google Sheets         | Updates sheet with campaign stats               | Checking if Campaign have data | Get Campaign ID             |                              |
| Checking Conversion        | Code                  | Processes campaign conversion data              | Checking if Campaign have data | Merge                      |                              |
| Merge                     | Merge                  | Combines data flows                              | Checking Conversion, Get Campaign ID | Aggregate                 |                              |
| Aggregate                 | Aggregate              | Aggregates campaign statistics                   | Merge                      | Update Campaign statistics    |                              |
| Update Campaign statistics | Google Sheets          | Writes aggregated campaign statistics to sheet | Aggregate                   | Get Campaign ID              |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node named "Set Schedule"**  
   - Type: Schedule Trigger  
   - Configure desired interval (e.g., daily at 1 AM)  
   - This will trigger the flow to fetch campaigns.

2. **Create Facebook Graph API node named "Add credentials to the Facebook Graph API"**  
   - Configure with Facebook OAuth2 credentials (App ID, App Secret, Access Token)  
   - Connect "Set Schedule" output to this node input.

3. **Create HTTP Request node named "Get Campaign from AD Account"**  
   - Method: GET  
   - URL: Facebook Graph API endpoint for campaigns, e.g. `https://graph.facebook.com/v14.0/<AD_ACCOUNT_ID>/campaigns`  
   - Use credentials from previous node  
   - Connect output of "Add credentials to the Facebook Graph API" to this node.

4. **Create Code node named "Code1"**  
   - JavaScript logic to parse and process the raw campaign data from Facebook API  
   - Connect "Get Campaign from AD Account" output to this node.

5. **Create If node named "Get Only Active Campaign"**  
   - Condition: Filter campaigns where status is "ACTIVE" (e.g., `item.status === 'ACTIVE'`)  
   - Connect "Code1" output to this node.

6. **Create Google Sheets node named "Add your campaign to file"**  
   - Operation: Append or Update row  
   - Sheet: Select or create a Google Sheet dedicated to campaign metadata  
   - Connect "Get Only Active Campaign" output to this node.

7. **Create second Schedule Trigger node named "Set Schedule1"**  
   - Configure to trigger at intervals for updating campaign statistics (e.g., daily at 2 AM)  

8. **Create Google Sheets node named "Get Campaign ID"**  
   - Operation: Read rows  
   - Sheet: Same sheet storing campaign IDs  
   - Connect "Set Schedule1" output to this node.

9. **Create HTTP Request node named "Get Campaign statistics"**  
   - Method: GET  
   - URL: Facebook Graph API endpoint to get campaign insights, e.g. `https://graph.facebook.com/v14.0/<CAMPAIGN_ID>/insights`  
   - Connect "Get Campaign ID" output to this node.

10. **Create If node named "Checking if Campaign have data"**  
    - Condition: Check if response contains data array with length > 0  
    - Connect "Get Campaign statistics" output to this node.

11. **Create Google Sheets node named "Update row in sheet4"**  
    - Operation: Update row  
    - Sheet: Same or another sheet for detailed stats  
    - Connect "Checking if Campaign have data" True output to this node.

12. **Create Code node named "Checking Conversion"**  
    - JavaScript to process conversion metrics from statistics  
    - Connect "Checking if Campaign have data" False output to this node.

13. **Create Merge node named "Merge"**  
    - Mode: Merge by index or property  
    - Connect "Checking Conversion" output and "Get Campaign ID" output to this node (multiple inputs).

14. **Create Aggregate node named "Aggregate"**  
    - Configure aggregation operations (sum, average) on merged campaign data  
    - Connect "Merge" output to this node.

15. **Create Google Sheets node named "Update Campaign statistics"**  
    - Operation: Update or append rows with aggregated statistics  
    - Connect "Aggregate" output to this node.

16. **Connect "Update Campaign statistics" output back to "Get Campaign ID"** (loop for continuous update or batch processing).

**Credential Setup:**  
- Facebook Graph API node requires Facebook OAuth2 credentials with appropriate permissions (ads_read, ads_management).  
- Google Sheets nodes require Google OAuth2 credentials with read/write access to the targeted sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow depends on having a correctly configured Facebook App with Ads API permissions enabled. | Facebook for Developers: https://developers.facebook.com/docs/marketing-api/                        |
| Google Sheets API quotas and rate limits may affect update frequency and volume.                     | Google Sheets API docs: https://developers.google.com/sheets/api/limits                             |
| For efficient data handling, ensure that campaign IDs and stats are stored consistently in Sheets.  |                                                                                                     |
| This workflow can be adapted to other ad platforms by replacing API calls and adapting data parsing.|                                                                                                     |

---

**Disclaimer:** The provided workflow JSON was processed and interpreted according to the current n8n capabilities and Facebook Ads API specifications. The documented flow respects all applicable content policies and uses only public, legal data sources.