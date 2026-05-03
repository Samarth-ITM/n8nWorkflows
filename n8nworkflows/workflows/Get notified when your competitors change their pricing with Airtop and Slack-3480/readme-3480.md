Get notified when your competitors change their pricing with Airtop and Slack

https://n8nworkflows.xyz/workflows/get-notified-when-your-competitors-change-their-pricing-with-airtop-and-slack-3480


# Get notified when your competitors change their pricing with Airtop and Slack

### 1. Workflow Overview

This workflow automates competitor pricing monitoring by extracting pricing data from competitor websites, comparing it with previously stored pricing information, and notifying users via Slack when significant changes occur. It is designed for businesses that want to stay updated on competitor pricing without manual tracking.

Logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval:** Fetch competitor pricing URLs and baseline pricing data from Google Sheets.
- **1.3 Pricing Extraction & Comparison:** Use Airtop AI to extract and summarize pricing from competitor pages, comparing current data with previous baseline.
- **1.4 Data Processing:** Parse AI response and merge with baseline data.
- **1.5 Change Filtering:** Filter out unchanged or similar pricing results.
- **1.6 Data Update & Notification:** Update Google Sheets with new pricing data and notify via Slack if changes are detected.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block initiates the workflow manually for testing or on-demand runs.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Configuration: No parameters; triggers workflow execution on manual action.  
    - Inputs: None  
    - Outputs: Connected to "Get Pricing URLs" node.  
    - Edge cases: None; manual trigger ensures controlled execution.

#### 1.2 Data Retrieval

- **Overview:** Retrieves competitor pricing URLs and baseline pricing data from a Google Sheets document to use as input for pricing comparison.
- **Nodes Involved:**  
  - Get Pricing URLs

- **Node Details:**

  - **Get Pricing URLs**  
    - Type: Google Sheets  
    - Role: Reads competitor pricing URLs and baseline pricing data from a specified Google Sheet.  
    - Configuration:  
      - Document ID: Google Sheet containing competitor pricing data.  
      - Sheet Name: "Sheet1" (gid=0).  
      - Credentials: Google Drive OAuth2 credentials.  
    - Key Expressions: None explicitly; outputs rows with pricing URLs and baseline pricing.  
    - Inputs: From manual trigger node.  
    - Outputs: To "Check pricing" and "Merge" nodes.  
    - Edge cases:  
      - Authentication errors if Google credentials expire or are invalid.  
      - Empty or malformed sheet data causing downstream errors.

#### 1.3 Pricing Extraction & Comparison

- **Overview:** Uses Airtop AI node to extract pricing details from competitor URLs and compare with previous baseline pricing, generating summaries and status flags.
- **Nodes Involved:**  
  - Check pricing

- **Node Details:**

  - **Check pricing**  
    - Type: Airtop (AI extraction)  
    - Role: Extracts pricing plans, prices, and top features from competitor pricing pages and compares with previous data.  
    - Configuration:  
      - URL: Dynamic, from current item’s "Pricing URL".  
      - Prompt: Custom prompt instructing Airtop to summarize pricing plans, list prices and top 3 features, and compare with previous pricing.  
      - Output Schema: JSON object with fields: `pricing_summary`, `differences_summary`, `status` (values: [DIFF], [SIMILAR], [NEW]).  
      - Session Mode: New session per request.  
      - Credentials: Airtop API key.  
    - Inputs: From "Get Pricing URLs".  
    - Outputs: To "Merge" node.  
    - Edge cases:  
      - API authentication failure or rate limits.  
      - Inaccurate or incomplete extraction if pricing page structure is complex or dynamic.  
      - Prompt misinterpretation causing incorrect summaries.  
      - Network timeouts or errors.

#### 1.4 Data Processing

- **Overview:** Parses the JSON response from Airtop, combines it with baseline data, and prepares it for filtering.
- **Nodes Involved:**  
  - Merge  
  - Parse response

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from "Check pricing" and "Get Pricing URLs" by position to align baseline and new data.  
    - Configuration:  
      - Mode: Combine  
      - Combine By: Position (index-based)  
    - Inputs: Two inputs: from "Check pricing" and "Get Pricing URLs".  
    - Outputs: To "Parse response".  
    - Edge cases:  
      - Mismatched array lengths causing misaligned merges.  
      - Missing data in either input causing incomplete merges.

  - **Parse response**  
    - Type: Code (JavaScript)  
    - Role: Parses Airtop JSON response string into JSON object and appends metadata like row number and pricing URL.  
    - Configuration:  
      - Runs once per item.  
      - Code extracts `modelResponse` field, parses JSON, and merges with existing fields.  
    - Inputs: From "Merge".  
    - Outputs: To "Filter out similar".  
    - Edge cases:  
      - JSON parse errors if Airtop response is malformed.  
      - Missing fields causing runtime errors.

#### 1.5 Change Filtering

- **Overview:** Filters out pricing entries where the status indicates no significant change ([SIMILAR]), allowing only new or different pricing to proceed.
- **Nodes Involved:**  
  - Filter out similar

- **Node Details:**

  - **Filter out similar**  
    - Type: Filter  
    - Role: Passes only items where `status` is not "SIMILAR".  
    - Configuration:  
      - Condition: `status` does NOT contain "SIMILAR" (case-sensitive).  
    - Inputs: From "Parse response".  
    - Outputs: To "Update pricing" and "Notify pricing change".  
    - Edge cases:  
      - Case sensitivity may cause unexpected filtering if status values vary.  
      - Missing or empty `status` field causing all items to pass or fail.

#### 1.6 Data Update & Notification

- **Overview:** Updates the Google Sheet with new pricing summaries and timestamps, then sends a Slack notification about detected pricing changes.
- **Nodes Involved:**  
  - Update pricing  
  - Notify pricing change

- **Node Details:**

  - **Update pricing**  
    - Type: Google Sheets  
    - Role: Updates existing rows in Google Sheet with new pricing summary and timestamp.  
    - Configuration:  
      - Operation: Update  
      - Matching Column: `row_number` to identify the row to update.  
      - Columns updated: Time (current timestamp), Pricing (new summary), Pricing URL, row_number (read-only).  
      - Document ID and Sheet Name same as "Get Pricing URLs".  
      - Credentials: Google Drive OAuth2.  
    - Inputs: From "Filter out similar".  
    - Outputs: None (end of branch).  
    - Edge cases:  
      - Row not found if `row_number` is incorrect or missing.  
      - Authentication or permission errors.  
      - Concurrent updates causing race conditions.

  - **Notify pricing change**  
    - Type: Slack  
    - Role: Sends a Slack message to a specified channel with pricing URL and summary of differences.  
    - Configuration:  
      - Text: Concatenates Pricing URL and differences summary.  
      - Channel: Fixed channel ID "pricing-changes".  
      - Credentials: Slack OAuth token.  
    - Inputs: From "Filter out similar".  
    - Outputs: None (end of branch).  
    - Edge cases:  
      - Slack API rate limits or authentication failures.  
      - Channel ID invalid or user lacks permission to post.  
      - Message length limits.

---

### 3. Summary Table

| Node Name             | Node Type         | Functional Role                          | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                      |
|-----------------------|-------------------|----------------------------------------|-----------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Entry point to start workflow manually | None                        | Get Pricing URLs                |                                                                                                 |
| Get Pricing URLs      | Google Sheets     | Retrieve competitor URLs and baseline data | When clicking ‘Test workflow’ | Check pricing, Merge            |                                                                                                 |
| Check pricing         | Airtop            | Extract and compare pricing data       | Get Pricing URLs             | Merge                          |                                                                                                 |
| Merge                 | Merge             | Combine baseline and new pricing data  | Get Pricing URLs, Check pricing | Parse response                 |                                                                                                 |
| Parse response        | Code              | Parse AI JSON response and enrich data | Merge                       | Filter out similar             |                                                                                                 |
| Filter out similar    | Filter            | Filter out unchanged pricing entries   | Parse response              | Update pricing, Notify pricing change |                                                                                                 |
| Update pricing        | Google Sheets     | Update sheet with new pricing data     | Filter out similar          | None                          |                                                                                                 |
| Notify pricing change | Slack             | Notify Slack channel about pricing changes | Filter out similar          | None                          |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Test workflow’".  
   - No parameters needed.

2. **Add Google Sheets Node to Retrieve Pricing URLs**  
   - Add "Google Sheets" node named "Get Pricing URLs".  
   - Set operation to "Read Rows" (default).  
   - Configure:  
     - Document ID: Use your Google Sheet ID containing competitor pricing data.  
     - Sheet Name: "Sheet1" or appropriate sheet.  
   - Connect "When clicking ‘Test workflow’" output to this node input.  
   - Set Google Drive OAuth2 credentials.

3. **Add Airtop Node to Extract and Compare Pricing**  
   - Add "Airtop" node named "Check pricing".  
   - Set resource to "extraction", operation to "query".  
   - Set "URL" parameter to `={{ $json["Pricing URL"] }}` (dynamic from input).  
   - Use the provided prompt to instruct Airtop to summarize pricing plans and compare with previous data.  
   - Set output schema to expect `pricing_summary`, `differences_summary`, and `status`.  
   - Use a new session mode.  
   - Connect "Get Pricing URLs" main output to "Check pricing" main input.  
   - Set Airtop API credentials.

4. **Add Merge Node to Combine Baseline and New Data**  
   - Add "Merge" node named "Merge".  
   - Set mode to "Combine".  
   - Set combine by "Position".  
   - Connect "Get Pricing URLs" second output (or same output as needed) to second input of "Merge".  
   - Connect "Check pricing" output to first input of "Merge".

5. **Add Code Node to Parse Airtop Response**  
   - Add "Code" node named "Parse response".  
   - Set mode to "Run Once For Each Item".  
   - Paste JavaScript code to parse JSON from `data.modelResponse` and merge with existing fields (`row_number`, `Pricing URL`).  
   - Connect "Merge" output to "Parse response" input.

6. **Add Filter Node to Filter Out Similar Pricing**  
   - Add "Filter" node named "Filter out similar".  
   - Set condition: `status` does NOT contain "SIMILAR" (case-sensitive).  
   - Connect "Parse response" output to "Filter out similar" input.

7. **Add Google Sheets Node to Update Pricing Data**  
   - Add "Google Sheets" node named "Update pricing".  
   - Set operation to "Update".  
   - Configure columns to update:  
     - Time: `={{ $now }}` (current timestamp)  
     - Pricing: `={{ $json.pricing_summary }}`  
     - row_number: `={{ $json.row_number }}` (used as matching column)  
     - Pricing URL: `={{ $json["Pricing URL"] }}`  
   - Set matching column to "row_number".  
   - Use same Document ID and Sheet Name as "Get Pricing URLs".  
   - Connect "Filter out similar" output to "Update pricing" input.  
   - Set Google Drive OAuth2 credentials.

8. **Add Slack Node to Notify Pricing Changes**  
   - Add "Slack" node named "Notify pricing change".  
   - Set operation to "Send Message".  
   - Set channel to the Slack channel ID for pricing notifications (e.g., "pricing-changes").  
   - Set message text to: `={{ $json["Pricing URL"] + " - " + $json.differences_summary }}`.  
   - Connect "Filter out similar" output to "Notify pricing change" input.  
   - Set Slack API credentials.

9. **Verify Connections**  
   - Connect nodes as follows:  
     - When clicking ‘Test workflow’ → Get Pricing URLs  
     - Get Pricing URLs → Check pricing (main output)  
     - Get Pricing URLs → Merge (second input)  
     - Check pricing → Merge (first input)  
     - Merge → Parse response  
     - Parse response → Filter out similar  
     - Filter out similar → Update pricing  
     - Filter out similar → Notify pricing change

10. **Set Credentials**  
    - Configure Google Drive OAuth2 credentials for Google Sheets nodes.  
    - Configure Airtop API credentials for Airtop node.  
    - Configure Slack API credentials for Slack node.

11. **Test Workflow**  
    - Run manual trigger to test end-to-end functionality.  
    - Verify Google Sheets updates and Slack notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Free Airtop API Key required to use Airtop node for pricing extraction.                                        | https://portal.airtop.ai/?utm_campaign=n8n                                                        |
| Google Sheets template for competitor pricing baseline data.                                                  | https://docs.google.com/spreadsheets/d/1qXuUU2RuxPkwoEqPk0hwQufNZvwXXIAAHKnn9SxW12s/edit?usp=sharing |
| Slack channel ID must be set to your pricing notification channel.                                            | Slack workspace configuration                                                                     |
| Best practices include maintaining detailed baseline data, specifying pricing tiers, and updating baseline regularly. | See workflow description section                                                                  |
| Consider extending notifications to other channels like Email or Telegram for broader alerting.               | Customization options in workflow description                                                     |
| Workflow designed for manual trigger but can be scheduled with Cron node for periodic monitoring.             | n8n scheduling capabilities                                                                        |

---

This document provides a complete and detailed reference for understanding, reproducing, and maintaining the "Monitor Competitor Pricing" workflow in n8n.