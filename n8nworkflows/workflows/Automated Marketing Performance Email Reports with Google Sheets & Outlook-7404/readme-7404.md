Automated Marketing Performance Email Reports with Google Sheets & Outlook

https://n8nworkflows.xyz/workflows/automated-marketing-performance-email-reports-with-google-sheets---outlook-7404


# Automated Marketing Performance Email Reports with Google Sheets & Outlook

### 1. Workflow Overview

This workflow automates the generation and emailing of daily marketing performance reports using data from Google Sheets and Microsoft Outlook. It targets marketing teams or analysts who need regular, structured insights into campaign performance metrics such as customer reach, campaign counts, clicks, conversions, and spend, delivered via a polished HTML email.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Data Retrieval:** Manual trigger initiates the workflow and fetches raw marketing data from a specified Google Sheets document.
- **1.2 Data Aggregation & Summarization:** Multiple summarization nodes process the raw data to compute unique counts and sums for key marketing metrics.
- **1.3 Data Merging:** Aggregated metrics are combined into a single structured data object for reporting.
- **1.4 Email Reporting:** The combined data is formatted into a modern, responsive HTML email and sent via Microsoft Outlook.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Data Retrieval

- **Overview:** Starts the workflow manually and retrieves marketing data from Google Sheets.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Google Sheets Data (Google Sheets node)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point to start the workflow on-demand.  
    - *Configuration:* Default manual trigger with no additional parameters.  
    - *Input/Output:* No input; output triggers next node.  
    - *Failure Modes:* None typical; user must manually execute.  

  - **Get Google Sheets Data**  
    - *Type:* Google Sheets  
    - *Role:* Reads marketing data from a Google Sheets spreadsheet.  
    - *Configuration:*  
      - Document ID set to a specific Google Sheet (sample marketing data).  
      - Sheet name set to the “Data” tab by GID.  
      - OAuth2 credentials configured for Google Sheets API access.  
    - *Key Variables:* None dynamic; static document and sheet IDs.  
    - *Input/Output:* Input from manual trigger; outputs raw rows of marketing data.  
    - *Failure Modes:*  
      - Authentication errors if credentials are invalid or expired.  
      - API quota limits or connectivity issues.  
      - Sheet structure changes causing missing columns.  

---

#### 1.2 Data Aggregation & Summarization

- **Overview:** Calculates key marketing performance metrics by aggregating the raw data.
- **Nodes Involved:**  
  - Count Unique Customers (Summarize)  
  - Count Unique Campaigns (Summarize)  
  - Sum Total Clicks (Summarize)  
  - Sum Total Conversions (Summarize)  
  - Sum Total Spend (Summarize)

- **Node Details:**

  - **Count Unique Customers**  
    - *Type:* Summarize  
    - *Role:* Counts unique values in the "Customer ID" column.  
    - *Configuration:* Aggregation set to "countUnique" on "Customer ID".  
    - *Connections:* Input from Google Sheets data, output to Merge node.  
    - *Failure Modes:*  
      - Missing "Customer ID" column causes errors.  

  - **Count Unique Campaigns**  
    - *Type:* Summarize  
    - *Role:* Counts unique campaign names in "Campaign" column.  
    - *Configuration:* Aggregation "countUnique" on "Campaign".  
    - *Input/Output:* Same as above.  
    - *Failure Modes:* Missing or renamed "Campaign" column.  

  - **Sum Total Clicks**  
    - *Type:* Summarize  
    - *Role:* Sums all values in the "Clicks" column.  
    - *Configuration:* Aggregation "sum" on "Clicks".  
    - *Failure Modes:* Non-numeric or missing data in "Clicks" column.  

  - **Sum Total Conversions**  
    - *Type:* Summarize  
    - *Role:* Sums "Conversions" column values.  
    - *Failure Modes:* Same as above for "Conversions" data.  

  - **Sum Total Spend**  
    - *Type:* Summarize  
    - *Role:* Sums "Spend ($)" column values.  
    - *Configuration:* Aggregation "sum" on "Spend ($)".  
    - *Failure Modes:* Formatting issues with currency values or missing column.  

---

#### 1.3 Data Merging

- **Overview:** Combines all aggregated metrics into a single data object for reporting.
- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Combines outputs of five summarization nodes by position into one dataset.  
    - *Configuration:* Mode set to "combine" using position index; no filters or conditions.  
    - *Input/Output:* Accepts five inputs (one from each summarization node), outputs merged data.  
    - *Failure Modes:*  
      - Mismatch in input array sizes or missing inputs causes incomplete merge.  

---

#### 1.4 Email Reporting

- **Overview:** Sends a styled HTML email with the aggregated report data via Outlook.
- **Nodes Involved:**  
  - Send Email Report1 (Microsoft Outlook node)

- **Node Details:**

  - **Send Email Report1**  
    - *Type:* Microsoft Outlook  
    - *Role:* Sends the final marketing performance report email.  
    - *Configuration:*  
      - OAuth2 credentials configured for Microsoft Outlook account.  
      - Email recipient statically set to "rbreen@ynteractive.com" (should be customized).  
      - Subject: "Daily Marketing Performance".  
      - Body: Rich HTML template with glassmorphic design and responsive layout.  
        - Uses expressions like `{{ $json.unique_count_Campaign }}` to inject metrics dynamically.  
      - Body content type set to "html".  
    - *Input/Output:* Input from Merge node, no output.  
    - *Failure Modes:*  
      - Authentication errors with Outlook credentials.  
      - Recipient email invalid or blocked.  
      - HTML rendering issues if expressions fail or data missing.  

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                    | Input Node(s)                      | Output Node(s)           | Sticky Note                                                                                   |
|----------------------------|--------------------|----------------------------------|----------------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger    | Starts workflow on-demand         | -                                | Get Google Sheets Data    | ## 📬 Need Help or Want to Customize This?  📧 robert@ynteractive.com 🔗 [LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/) |
| Get Google Sheets Data      | Google Sheets      | Fetches raw marketing data        | When clicking ‘Execute workflow’  | Count Unique Customers; Count Unique Campaigns; Sum Total Clicks; Sum Total Conversions; Sum Total Spend | See Sticky Note2 for setup instructions and sample data template link                          |
| Count Unique Customers      | Summarize          | Counts unique customers           | Get Google Sheets Data            | Merge                    | See Sticky Note4 for details on summarization nodes                                           |
| Count Unique Campaigns      | Summarize          | Counts unique campaigns           | Get Google Sheets Data            | Merge                    | See Sticky Note4                                                                              |
| Sum Total Clicks            | Summarize          | Sums total clicks                 | Get Google Sheets Data            | Merge                    | See Sticky Note4                                                                              |
| Sum Total Conversions       | Summarize          | Sums total conversions            | Get Google Sheets Data            | Merge                    | See Sticky Note4                                                                              |
| Sum Total Spend             | Summarize          | Sums total marketing spend        | Get Google Sheets Data            | Merge                    | See Sticky Note4                                                                              |
| Merge                      | Merge              | Combines all summarized metrics  | Count Unique Customers; Count Unique Campaigns; Sum Total Clicks; Sum Total Conversions; Sum Total Spend | Send Email Report1        | See Sticky Note3 for merge and email node details                                            |
| Send Email Report1          | Microsoft Outlook  | Sends HTML email report           | Merge                            | -                        | See Sticky Note3 for configuration details and email template features                       |
| Sticky Note17               | Sticky Note        | Contact info and help             | -                                | -                        | ## 📬 Need Help or Want to Customize This?  📧 robert@ynteractive.com 🔗 [LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/) |
| Sticky Note2                | Sticky Note        | Instructions for Google Sheets setup | -                            | -                        | Step 1: Set Up Your Google Sheets Data Source with links and detailed instructions           |
| Sticky Note3                | Sticky Note        | Instructions for Merge & Email nodes | -                            | -                        | Details on configuring Merge and Microsoft Outlook nodes                                    |
| Sticky Note4                | Sticky Note        | Details on summarization nodes   | -                                | -                        | Explanation of summarization nodes purpose and no-setup requirement                          |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Manual Trigger Node**  
- Add **Manual Trigger** node, name it `When clicking ‘Execute workflow’`.  
- No special configuration needed.

**Step 2: Add Google Sheets Node to Retrieve Data**  
- Add **Google Sheets** node, name it `Get Google Sheets Data`.  
- Set **Document ID** to your Google Sheet ID (e.g., `19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA`).  
- Set **Sheet Name** to the "Data" tab (GID or named range).  
- Configure **Google Sheets OAuth2 API** credentials with appropriate permissions.  
- Connect output of Manual Trigger to this node.

**Step 3: Add Summarize Nodes for Metrics**  
Create five **Summarize** nodes with the following configurations, each connected from `Get Google Sheets Data` output:

- `Count Unique Customers`: Count unique on "Customer ID" column.  
- `Count Unique Campaigns`: Count unique on "Campaign" column.  
- `Sum Total Clicks`: Sum on "Clicks" column.  
- `Sum Total Conversions`: Sum on "Conversions" column.  
- `Sum Total Spend`: Sum on "Spend ($)" column.

No additional setup required; just specify field and aggregation type.

**Step 4: Add Merge Node to Combine Data**  
- Add **Merge** node named `Merge`.  
- Set **Mode** to "Combine".  
- Set **Combine By** to "Position".  
- Connect all five Summarize nodes outputs to Merge node inputs in any order but ensure consistent indexing.

**Step 5: Add Microsoft Outlook Node to Send Email**  
- Add **Microsoft Outlook** node named `Send Email Report1`.  
- Set **OAuth2 credentials** for Microsoft Outlook (create new OAuth2 credential if needed).  
- Configure email parameters:  
  - **To Recipients:** Set to desired email address (default `rbreen@ynteractive.com`).  
  - **Subject:** Set to "Daily Marketing Performance" or customize.  
  - **Body Content Type:** Set to "html".  
  - **Body Content:** Paste the provided HTML template with placeholders for metrics (e.g., `{{ $json.unique_count_Campaign }}`).  
- Connect output of `Merge` node to this node.

**Step 6: Test the workflow**  
- Manually trigger the workflow and verify the email is received with correct data and formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Step 1: Set Up Your Google Sheets Data Source with sample template and API setup instructions.                                                                  | [Marketing Performance Data Template](https://docs.google.com/spreadsheets/d/19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA/edit?usp=sharing) and Google Cloud Console |
| Merge and Microsoft Outlook node setup details including OAuth2 credential configuration and email template features.                                           | See Sticky Note3 content in workflow                                                                                |
| Contact for customization or help: Robert Breen, including email and LinkedIn link.                                                                              | [robert@ynteractive.com](mailto:robert@ynteractive.com), [LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/)                |

---

**Disclaimer:**  
The provided text is exclusively extracted from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and does not include any illegal, offensive, or protected content. All handled data is legal and public.