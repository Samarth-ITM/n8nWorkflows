Get Meta Ads insights and save them into Google Sheets

https://n8nworkflows.xyz/workflows/get-meta-ads-insights-and-save-them-into-google-sheets-2714


# Get Meta Ads insights and save them into Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of Meta (Facebook) Ads insights via the Facebook Graph API and saves the data into Google Sheets for further analysis. It targets marketing professionals who want to track key advertising metrics such as impressions, spend, reach, conversions, CTR, and CPC on a daily basis. The workflow runs automatically every day at 3 AM, pulling data from the previous day, but can also be triggered manually for testing.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Block:** Initiates the workflow either on a schedule (daily at 3 AM) or manually.
- **1.2 Data Retrieval Block:** Calls the Facebook Graph API to fetch Meta Ads insights for the specified date range (primarily yesterday).
- **1.3 Data Processing Block:** Splits and filters the raw API data into general metrics, non-monetary actions, and monetary actions.
- **1.4 Data Storage Block:** Inserts the processed data into designated Google Sheets tabs for general metrics, non-monetary actions, and monetary actions.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:**  
  This block starts the workflow either automatically every day at 3 AM or manually via a button click for testing purposes.

- **Nodes Involved:**  
  - Everyday at 3am (Schedule Trigger)  
  - When clicking ‘Test workflow’ (Manual Trigger)

- **Node Details:**

  - **Everyday at 3am**  
    - Type: Schedule Trigger  
    - Configuration: Triggers the workflow daily at 3 AM server time.  
    - Inputs: None  
    - Outputs: Connects to "Ad insights from yesterday" node.  
    - Edge Cases: Timezone misconfiguration could cause unexpected trigger times.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Configuration: Allows manual execution of the workflow for testing.  
    - Inputs: None  
    - Outputs: Connects to "Ad insights from yesterday" node.  
    - Edge Cases: None significant; manual trigger bypasses schedule.

---

#### 2.2 Data Retrieval Block

- **Overview:**  
  This block fetches Meta Ads insights from the Facebook Graph API for the previous day.

- **Nodes Involved:**  
  - Ad insights from yesterday (Facebook Graph API)

- **Node Details:**

  - **Ad insights from yesterday**  
    - Type: Facebook Graph API  
    - Configuration:  
      - API call configured to retrieve Meta Ads insights for the date range of "yesterday".  
      - Metrics include impressions, spend, reach, conversions, CTR, CPC, and actions.  
      - Uses OAuth2 credentials for Facebook API authentication.  
    - Inputs: Trigger nodes ("Everyday at 3am" and "When clicking ‘Test workflow’")  
    - Outputs: Connects to "data column only" node.  
    - Edge Cases:  
      - API rate limits or authentication failures.  
      - Missing or incomplete data for the date range.  
      - Network timeouts.

---

#### 2.3 Data Processing Block

- **Overview:**  
  Processes the raw API response by splitting data arrays and filtering action types into general metrics, non-monetary actions, and monetary actions.

- **Nodes Involved:**  
  - data column only (Split Out)  
  - split actions (Split Out)  
  - split action values (Split Out)  
  - filter by action type (Filter)  
  - Only monetary actions (Filter)  
  - filter by monetary action type (Filter)

- **Node Details:**

  - **data column only**  
    - Type: Split Out  
    - Configuration: Splits the main data array from the API response into individual items for processing.  
    - Inputs: "Ad insights from yesterday"  
    - Outputs: Connects to "split actions" and "Add General Metrics".  
    - Edge Cases: Empty or malformed data arrays.

  - **split actions**  
    - Type: Split Out  
    - Configuration: Further splits the "actions" array inside each data item to process individual action entries.  
    - Inputs: "data column only"  
    - Outputs: Connects to "split action values" and "filter by action type".  
    - Edge Cases: Missing "actions" array or empty actions.

  - **split action values**  
    - Type: Split Out  
    - Configuration: Splits nested action values for detailed filtering.  
    - Inputs: "split actions"  
    - Outputs: Connects to "Only monetary actions".  
    - Edge Cases: Empty nested arrays.

  - **filter by action type**  
    - Type: Filter  
    - Configuration: Filters actions to select only non-monetary action types (e.g., clicks, page views).  
    - Inputs: "split actions"  
    - Outputs: Connects to "Add Non-Monetary actions".  
    - Edge Cases: No matching non-monetary actions.

  - **Only monetary actions**  
    - Type: Filter  
    - Configuration: Filters actions to select only monetary action entries (e.g., purchases, leads).  
    - Inputs: "split action values"  
    - Outputs: Connects to "filter by monetary action type".  
    - Edge Cases: No monetary actions present.

  - **filter by monetary action type**  
    - Type: Filter  
    - Configuration: Further filters monetary actions by specific types for detailed reporting.  
    - Inputs: "Only monetary actions"  
    - Outputs: Connects to "Add Monetary actions".  
    - Edge Cases: No matching monetary action types.

---

#### 2.4 Data Storage Block

- **Overview:**  
  Saves the processed data into Google Sheets tabs corresponding to general metrics, non-monetary actions, and monetary actions.

- **Nodes Involved:**  
  - Add General Metrics (Google Sheets)  
  - Add Non-Monetary actions (Google Sheets)  
  - Add Monetary actions (Google Sheets)

- **Node Details:**

  - **Add General Metrics**  
    - Type: Google Sheets  
    - Configuration: Appends general metrics data (impressions, spend, reach, CTR, CPC, etc.) into a designated Google Sheets spreadsheet and tab.  
    - Inputs: "data column only"  
    - Outputs: None (end node)  
    - Credentials: Requires Google Sheets OAuth2 credentials.  
    - Edge Cases:  
      - Google Sheets API quota limits.  
      - Sheet or tab not existing.  
      - Data format mismatches.

  - **Add Non-Monetary actions**  
    - Type: Google Sheets  
    - Configuration: Appends filtered non-monetary action data into a specific Google Sheets tab.  
    - Inputs: "filter by action type"  
    - Outputs: None (end node)  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Edge Cases: Same as above.

  - **Add Monetary actions**  
    - Type: Google Sheets  
    - Configuration: Appends filtered monetary action data into a dedicated Google Sheets tab.  
    - Inputs: "filter by monetary action type"  
    - Outputs: None (end node)  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Edge Cases: Same as above.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                          |
|-------------------------|----------------------|----------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Everyday at 3am         | Schedule Trigger     | Starts workflow daily at 3 AM           | None                          | Ad insights from yesterday      |                                                                                                    |
| When clicking ‘Test workflow’ | Manual Trigger       | Manual start for testing                 | None                          | Ad insights from yesterday      |                                                                                                    |
| Ad insights from yesterday | Facebook Graph API   | Fetches Meta Ads insights for yesterday | Everyday at 3am, Manual Trigger | data column only                |                                                                                                    |
| data column only         | Split Out            | Splits main data array                   | Ad insights from yesterday     | split actions, Add General Metrics |                                                                                                    |
| split actions            | Split Out            | Splits actions array                     | data column only               | split action values, filter by action type |                                                                                                    |
| split action values      | Split Out            | Splits nested action values              | split actions                 | Only monetary actions           |                                                                                                    |
| filter by action type    | Filter               | Filters non-monetary actions             | split actions                 | Add Non-Monetary actions        |                                                                                                    |
| Only monetary actions    | Filter               | Filters monetary actions                  | split action values           | filter by monetary action type  |                                                                                                    |
| filter by monetary action type | Filter           | Filters specific monetary action types   | Only monetary actions         | Add Monetary actions            |                                                                                                    |
| Add General Metrics      | Google Sheets        | Saves general metrics to Google Sheets  | data column only               | None                           |                                                                                                    |
| Add Non-Monetary actions | Google Sheets        | Saves non-monetary actions to Google Sheets | filter by action type          | None                           |                                                                                                    |
| Add Monetary actions     | Google Sheets        | Saves monetary actions to Google Sheets | filter by monetary action type | None                           |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node named "Everyday at 3am".
     - Set to trigger daily at 3 AM.
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".
     - No special configuration needed.

2. **Create Facebook Graph API Node:**
   - Add a **Facebook Graph API** node named "Ad insights from yesterday".
   - Configure to use Facebook OAuth2 credentials.
   - Set the API call to retrieve Meta Ads insights for the date range "yesterday".
   - Select metrics: impressions, spend, reach, conversions, CTR, CPC, and actions.
   - Connect both trigger nodes ("Everyday at 3am" and "When clicking ‘Test workflow’") to this node.

3. **Create Data Splitting Nodes:**
   - Add a **Split Out** node named "data column only".
     - Connect "Ad insights from yesterday" to this node.
     - Configure to split the main data array from the API response.
   - Add a **Split Out** node named "split actions".
     - Connect "data column only" to this node.
     - Configure to split the "actions" array inside each data item.
   - Add a **Split Out** node named "split action values".
     - Connect "split actions" to this node.
     - Configure to split nested action values.

4. **Create Filter Nodes:**
   - Add a **Filter** node named "filter by action type".
     - Connect "split actions" to this node.
     - Configure to allow only non-monetary action types (e.g., clicks, page views).
   - Add a **Filter** node named "Only monetary actions".
     - Connect "split action values" to this node.
     - Configure to allow only monetary action entries (e.g., purchases, leads).
   - Add a **Filter** node named "filter by monetary action type".
     - Connect "Only monetary actions" to this node.
     - Configure to filter specific monetary action types for detailed reporting.

5. **Create Google Sheets Nodes:**
   - Add a **Google Sheets** node named "Add General Metrics".
     - Connect "data column only" to this node.
     - Configure with Google Sheets OAuth2 credentials.
     - Set to append rows to the spreadsheet tab for general metrics.
   - Add a **Google Sheets** node named "Add Non-Monetary actions".
     - Connect "filter by action type" to this node.
     - Configure similarly, targeting the non-monetary actions tab.
   - Add a **Google Sheets** node named "Add Monetary actions".
     - Connect "filter by monetary action type" to this node.
     - Configure similarly, targeting the monetary actions tab.

6. **Verify Connections:**
   - Connect nodes as per the flow:
     - "Everyday at 3am" and "When clicking ‘Test workflow’" → "Ad insights from yesterday"
     - "Ad insights from yesterday" → "data column only"
     - "data column only" → "split actions" and "Add General Metrics"
     - "split actions" → "split action values" and "filter by action type"
     - "split action values" → "Only monetary actions"
     - "Only monetary actions" → "filter by monetary action type"
     - "filter by action type" → "Add Non-Monetary actions"
     - "filter by monetary action type" → "Add Monetary actions"

7. **Set Default Values and Constraints:**
   - Ensure date range in Facebook Graph API node is dynamic for "yesterday".
   - Confirm Google Sheets tabs exist and have appropriate headers matching data fields.
   - Handle empty data gracefully by enabling error handling or conditional checks if desired.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow is designed for marketing professionals to automate Meta Ads data collection.     | Workflow purpose                                    |
| You can replace Google Sheets nodes with database nodes (MySQL, Postgres) for BI tool integration. | Alternative data storage options                     |
| Check out other templates by the creator Solomon at: https://n8n.io/creators/solomon/            | Creator’s template collection                        |
| The workflow pulls data daily at 3 AM for the previous day’s Meta Ads insights.                  | Scheduling and data freshness                        |
| Metrics collected include impressions, spend, reach, conversions, CTR, CPC, and detailed actions.| Key metrics overview                                 |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling users and automation agents to reproduce, modify, or troubleshoot it effectively.