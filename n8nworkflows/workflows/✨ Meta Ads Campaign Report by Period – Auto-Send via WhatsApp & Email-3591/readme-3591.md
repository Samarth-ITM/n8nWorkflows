✨ Meta Ads Campaign Report by Period – Auto-Send via WhatsApp & Email

https://n8nworkflows.xyz/workflows/--meta-ads-campaign-report-by-period---auto-send-via-whatsapp---email-3591


# ✨ Meta Ads Campaign Report by Period – Auto-Send via WhatsApp & Email

### 1. Workflow Overview

This workflow automates the generation and distribution of Meta Ads campaign reports by a specified period. It is designed for digital marketers or agencies managing multiple Meta Ads accounts who want to streamline their daily reporting process. The workflow fetches campaign data, calculates key performance metrics, filters relevant campaigns, formats the report, and sends it automatically via WhatsApp or Email.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Account Retrieval:** Starts the workflow on a schedule and retrieves the list of Meta Ad accounts from a Google Sheet.
- **1.2 Campaign Data Collection:** Iterates through each ad account, fetching campaign performance data for the specified period using the Facebook Graph API.
- **1.3 Data Filtering & Processing:** Filters campaigns with meaningful results, processes creative-level data, and calculates key metrics.
- **1.4 Report Formatting:** Renames and aggregates data fields to prepare a clear, professional report.
- **1.5 Report Distribution:** Sends the formatted report either via Gmail or WhatsApp (using Evolution API), based on user selection.
- **1.6 Loop Control:** Manages iteration over multiple accounts and ensures the workflow continues until all accounts are processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Account Retrieval

- **Overview:** This block initiates the workflow on a schedule and retrieves the list of Meta Ads accounts from a Google Sheet to process.
- **Nodes Involved:** `Start Time`, `Filter`, `Ad Accounts`, `Click Next`

##### Node Details:

- **Start Time**
  - Type: Schedule Trigger
  - Role: Triggers the workflow automatically at configured times (e.g., every morning).
  - Configuration: Default schedule or user-defined time.
  - Inputs: None (trigger node).
  - Outputs: Connects to `Filter`.
  - Failure Modes: Misconfigured schedule or disabled workflow may prevent triggering.

- **Filter**
  - Type: Filter Node
  - Role: Applies initial filtering logic to decide whether to proceed.
  - Configuration: Custom conditions (not specified in JSON).
  - Inputs: From `Start Time`.
  - Outputs: Passes filtered data to `Ad Accounts`.
  - Failure Modes: Expression errors if conditions reference missing data.

- **Ad Accounts**
  - Type: Google Sheets
  - Role: Reads the list of Meta Ads accounts from a Google Sheet.
  - Configuration: Connected to a Google Sheet template managing accounts.
  - Inputs: From `Filter`.
  - Outputs: Passes account data to `Click Next`.
  - Failure Modes: Authentication errors, sheet access issues, or empty data.

- **Click Next**
  - Type: SplitInBatches
  - Role: Processes accounts one by one in batches to avoid API limits.
  - Configuration: Batch size likely set to 1 (default).
  - Inputs: From `Ad Accounts`.
  - Outputs: On first output, proceeds to `Set Data`; on second output, loops back or ends.
  - Failure Modes: Batch processing errors, infinite loops if not properly controlled.

---

#### 2.2 Campaign Data Collection

- **Overview:** For each ad account, this block fetches campaign data for the specified period from Meta Ads via Facebook Graph API.
- **Nodes Involved:** `Set Data`, `Ads by Period`, `So if`

##### Node Details:

- **Set Data**
  - Type: Set
  - Role: Prepares or formats data needed for the API request.
  - Configuration: Sets parameters such as date range, account ID, or filters.
  - Inputs: From `Click Next`.
  - Outputs: Passes prepared data to `Ads by Period`.
  - Failure Modes: Missing or incorrect parameters can cause API call failures.

- **Ads by Period**
  - Type: Facebook Graph API
  - Role: Queries Meta Ads API to retrieve campaign performance data for the given period.
  - Configuration: Uses Facebook Graph API credentials, queries metrics like impressions, clicks, spend.
  - Inputs: From `Set Data`.
  - Outputs: Passes raw campaign data to `So if`.
  - Failure Modes: API rate limits, expired tokens, permission errors.

- **So if**
  - Type: If
  - Role: Checks if the API returned data or if further processing is needed.
  - Configuration: Condition to verify if data exists.
  - Inputs: From `Ads by Period`.
  - Outputs: If true, proceeds to `Data by Creative`; if false, loops back to `Click Next`.
  - Failure Modes: Expression errors or unexpected empty data.

---

#### 2.3 Data Filtering & Processing

- **Overview:** Processes campaign data to filter campaigns with conversions, calculates metrics, and prepares data for reporting.
- **Nodes Involved:** `Data by Creative`, `Filter if there was Conversion`, `Rename Titles`, `Agregar`, `Dismember Text`

##### Node Details:

- **Data by Creative**
  - Type: Code (JavaScript)
  - Role: Processes raw campaign data at the creative level, extracting and calculating detailed metrics.
  - Configuration: Custom JavaScript code to parse and aggregate data.
  - Inputs: From `So if` (true branch).
  - Outputs: Passes processed data to `Save Data` and `Filter if there was Conversion`.
  - Failure Modes: Code errors, unexpected data formats.

- **Filter if there was Conversion**
  - Type: If
  - Role: Filters campaigns that had conversions (e.g., leads, purchases).
  - Configuration: Condition checks conversion count > 0.
  - Inputs: From `Data by Creative`.
  - Outputs: If true, proceeds to `Rename Titles`; if false, loops back to `Click Next`.
  - Failure Modes: Expression errors, missing conversion data.

- **Rename Titles**
  - Type: Set
  - Role: Renames data fields to user-friendly titles for the report.
  - Configuration: Maps internal field names to readable column headers.
  - Inputs: From `Filter if there was Conversion`.
  - Outputs: Passes renamed data to `Agregar`.
  - Failure Modes: Misalignment of fields or missing data.

- **Agregar**
  - Type: Aggregate
  - Role: Aggregates data, possibly summing or averaging metrics for the report.
  - Configuration: Aggregation rules (sum, average) on key metrics.
  - Inputs: From `Rename Titles`.
  - Outputs: Passes aggregated data to `Dismember Text`.
  - Failure Modes: Incorrect aggregation logic.

- **Dismember Text**
  - Type: Code (JavaScript)
  - Role: Formats aggregated data into text blocks or tables suitable for messaging.
  - Configuration: Custom code to convert data into formatted strings.
  - Inputs: From `Agregar`.
  - Outputs: Passes formatted text to `Switch`.
  - Failure Modes: Code errors, formatting issues.

---

#### 2.4 Report Formatting & Distribution

- **Overview:** Decides the delivery method and sends the report via Email or WhatsApp.
- **Nodes Involved:** `Switch`, `Send by Email`, `Send by WhatsApp`, `Return to the loop`

##### Node Details:

- **Switch**
  - Type: Switch
  - Role: Routes the workflow based on user choice of delivery method (Email or WhatsApp).
  - Configuration: Condition on delivery method parameter.
  - Inputs: From `Dismember Text`.
  - Outputs: To `Send by Email` or `Send by WhatsApp`.
  - Failure Modes: Missing or invalid delivery method.

- **Send by Email**
  - Type: Gmail
  - Role: Sends the formatted report via Gmail.
  - Configuration: Uses Gmail OAuth2 credentials, email template with report content.
  - Inputs: From `Switch`.
  - Outputs: Passes control to `Return to the loop`.
  - Failure Modes: Authentication errors, email quota limits.

- **Send by WhatsApp**
  - Type: HTTP Request
  - Role: Sends the report via WhatsApp using the Evolution API.
  - Configuration: HTTP POST request with message payload, uses API keys.
  - Inputs: From `Switch`.
  - Outputs: Passes control to `Return to the loop`.
  - Failure Modes: API errors, network issues, invalid credentials.
  - On Error: Continues workflow to avoid stopping on failure.

- **Return to the loop**
  - Type: NoOp
  - Role: Acts as a loop control node to continue processing next account.
  - Configuration: No parameters.
  - Inputs: From `Send by Email` and `Send by WhatsApp`.
  - Outputs: Loops back to `Click Next`.
  - Failure Modes: None.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                         |
|-------------------------|---------------------|---------------------------------------|-------------------------|-------------------------|-----------------------------------|
| Start Time              | Schedule Trigger    | Initiates workflow on schedule        | None                    | Filter                  |                                   |
| Filter                  | Filter              | Applies initial filter condition      | Start Time              | Ad Accounts             |                                   |
| Ad Accounts             | Google Sheets       | Retrieves Meta Ads accounts list      | Filter                  | Click Next              |                                   |
| Click Next              | SplitInBatches      | Processes accounts one by one         | Ad Accounts, Return to the loop | Set Data, (loop)    |                                   |
| Set Data                | Set                 | Prepares API request parameters       | Click Next              | Ads by Period           |                                   |
| Ads by Period           | Facebook Graph API  | Fetches campaign data from Meta Ads   | Set Data                | So if                   |                                   |
| So if                   | If                  | Checks if campaign data exists         | Ads by Period           | Data by Creative, Click Next |                                   |
| Data by Creative        | Code                | Processes creative-level data          | So if                   | Save Data, Filter if there was Conversion |                                   |
| Filter if there was Conversion | If            | Filters campaigns with conversions     | Data by Creative        | Rename Titles, Click Next |                                   |
| Rename Titles           | Set                 | Renames fields for report clarity      | Filter if there was Conversion | Agregar             |                                   |
| Agregar                 | Aggregate           | Aggregates key metrics                  | Rename Titles           | Dismember Text          |                                   |
| Dismember Text          | Code                | Formats data into text for messaging   | Agregar                 | Switch                  |                                   |
| Switch                  | Switch              | Routes report delivery method          | Dismember Text          | Send by Email, Send by WhatsApp |                                   |
| Send by Email           | Gmail               | Sends report via email                  | Switch                  | Return to the loop      |                                   |
| Send by WhatsApp        | HTTP Request        | Sends report via WhatsApp (Evolution API) | Switch               | Return to the loop      | On error: continue workflow       |
| Return to the loop      | NoOp                | Controls loop to process next account  | Send by Email, Send by WhatsApp | Click Next          |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named `Start Time`:
   - Set the desired schedule (e.g., daily at 7 AM).
   - This node triggers the workflow automatically.

2. **Add a Filter node** named `Filter`:
   - Connect `Start Time` output to `Filter` input.
   - Configure filter conditions to control workflow execution (e.g., only run on weekdays).

3. **Add a Google Sheets node** named `Ad Accounts`:
   - Connect `Filter` output to `Ad Accounts` input.
   - Configure credentials for Google Sheets.
   - Set to read the sheet containing Meta Ads account IDs and related info.
   - Use the provided Google Sheet template URL for structure.

4. **Add a SplitInBatches node** named `Click Next`:
   - Connect `Ad Accounts` output to `Click Next` input.
   - Set batch size to 1 to process accounts sequentially.

5. **Add a Set node** named `Set Data`:
   - Connect `Click Next` first output to `Set Data`.
   - Configure to set parameters for Facebook Graph API calls, including:
     - Account ID from batch data
     - Date range for report (e.g., last 7 days)
     - Filters for campaign types or actions

6. **Add a Facebook Graph API node** named `Ads by Period`:
   - Connect `Set Data` output to `Ads by Period`.
   - Configure with Facebook Graph API credentials.
   - Set endpoint to fetch campaign insights for the specified account and period.
   - Request metrics such as impressions, clicks, spend, conversions.

7. **Add an If node** named `So if`:
   - Connect `Ads by Period` output to `So if`.
   - Configure condition to check if data was returned (e.g., check if response array length > 0).
   - True branch connects to `Data by Creative`.
   - False branch connects back to `Click Next` to process next account.

8. **Add a Code node** named `Data by Creative`:
   - Connect `So if` true output to `Data by Creative`.
   - Write JavaScript code to:
     - Parse campaign data at the creative level.
     - Calculate metrics like CPC, CTR, CPV, conversions, spend.
     - Prepare data for filtering and reporting.

9. **Add a Google Sheets node** named `Save Data`:
   - Connect `Data by Creative` output to `Save Data`.
   - Configure to save processed data back to a Google Sheet for record-keeping.

10. **Add an If node** named `Filter if there was Conversion`:
    - Connect `Data by Creative` output to `Filter if there was Conversion`.
    - Configure to check if conversions > 0.
    - True branch connects to `Rename Titles`.
    - False branch connects back to `Click Next`.

11. **Add a Set node** named `Rename Titles`:
    - Connect `Filter if there was Conversion` true output to `Rename Titles`.
    - Configure to rename data fields to user-friendly report headers.

12. **Add an Aggregate node** named `Agregar`:
    - Connect `Rename Titles` output to `Agregar`.
    - Configure aggregation rules (sum, average) on key metrics.

13. **Add a Code node** named `Dismember Text`:
    - Connect `Agregar` output to `Dismember Text`.
    - Write JavaScript code to format the aggregated data into a text message or table.

14. **Add a Switch node** named `Switch`:
    - Connect `Dismember Text` output to `Switch`.
    - Configure to route based on a parameter indicating delivery method (`email` or `whatsapp`).

15. **Add a Gmail node** named `Send by Email`:
    - Connect `Switch` output for email to `Send by Email`.
    - Configure Gmail OAuth2 credentials.
    - Set email recipient, subject, and body with the formatted report.

16. **Add an HTTP Request node** named `Send by WhatsApp`:
    - Connect `Switch` output for WhatsApp to `Send by WhatsApp`.
    - Configure HTTP POST request to Evolution API endpoint.
    - Include API key and message payload with the formatted report.
    - Set error handling to continue on failure.

17. **Add a NoOp node** named `Return to the loop`:
    - Connect outputs of `Send by Email` and `Send by WhatsApp` to `Return to the loop`.
    - Connect `Return to the loop` output back to the second output of `Click Next` to continue batch processing.

18. **Test the workflow**:
    - Verify Google Sheets access and data structure.
    - Confirm Facebook Graph API permissions and token validity.
    - Test sending via Gmail and WhatsApp.
    - Adjust filters and parameters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Google Sheet template to manage accounts: Use this to organize Meta Ads accounts for the workflow.  | https://docs.google.com/spreadsheets/d/1c5QdBtyiYRW8DPmYxHWcfKa0Lg54CVHSrQJ6kXbiz40/edit?usp=sharing |
| WhatsApp contact for customization requests: Reach out for tailored workflow adaptations.           | https://wa.me/5517991557874 (+55 17 99155-7874)                                                  |
| Workflow automates daily reporting, saving time by integrating data retrieval, processing, and delivery. | General workflow purpose and benefits.                                                            |
| Requires Facebook Graph API access and Google account connected to n8n.                             | Setup prerequisites.                                                                              |
| WhatsApp sending uses Evolution API with error handling to avoid workflow interruption.             | Integration detail and robustness feature.                                                       |

---

This documentation provides a complete, structured reference to understand, reproduce, and maintain the "Meta Ads Campaign Report by Period – Auto-Send via WhatsApp & Email" workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with the workflow.