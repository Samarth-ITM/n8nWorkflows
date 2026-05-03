Track Marketing Channel Performance with HighLevel, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/track-marketing-channel-performance-with-highlevel--google-sheets--and-slack-10149


# Track Marketing Channel Performance with HighLevel, Google Sheets, and Slack

### 1. Workflow Overview

This workflow, named **Lead Source Quality Analyzer**, is designed to track and analyze the performance of marketing channels by processing sales opportunities from HighLevel CRM. It focuses on identifying successful ("won") deals, calculating conversion metrics per lead source, and distributing insights via Google Sheets and Slack. The logic is structured into these main blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Fetching:** Retrieve all opportunities from HighLevel CRM.
- **1.3 Filtering Logic:** Separate won deals from others.
- **1.4 Analytics Pipeline:**  
  - Log won deals to Google Sheets  
  - Calculate lead source metrics (counts and total amounts)  
  - Send summarized report to Slack
- **1.5 Non-Won Status Alerts:** Send Slack notifications for opportunities that are not won.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  The workflow begins manually via a trigger node, allowing controlled execution on demand.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters needed; simple manual start  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Starts the workflow flow to fetch HighLevel data  
  - **Edge Cases:** None (manual start)  
  - **Sub-workflow:** None

---

#### 1.2 Data Fetching

- **Overview:**  
  Retrieves the full list of opportunities from HighLevel CRM to serve as the data source for analysis.

- **Nodes Involved:**  
  - Fetch All HighLevel Opportunities  
  - Sticky Note - Fetch Data

- **Node Details:**  
  - **Fetch All HighLevel Opportunities**  
    - Type: HighLevel node (API integration)  
    - Configuration:  
      - Resource: opportunity  
      - Operation: getAll  
      - Return All: true (fetch all records without pagination limits)  
      - Filters: none (fetches all opportunities)  
    - Credentials: OAuth2 for HighLevel account  
    - Input: Trigger node output  
    - Output: Full list of opportunities with fields including status, amount, lead source, and id  
    - Edge Cases: API rate limits, authentication failures, empty datasets  
  - Sticky Note - Fetch Data  
    - Provides a summary of the role of this block for users

---

#### 1.3 Filtering Logic

- **Overview:**  
  Splits opportunities into two paths based on their status: won deals are sent for analysis; others trigger individual Slack alerts.

- **Nodes Involved:**  
  - Check If Status = Won  
  - Sticky Note - Filtering Logic

- **Node Details:**  
  - **Check If Status = Won**  
    - Type: If node (conditional branching)  
    - Configuration:  
      - Condition: `$json.status` equals the string `"won"`  
      - Case sensitive: true  
    - Input: Opportunities from HighLevel fetch node  
    - Outputs:  
      - TRUE path: opportunities with status "won"  
      - FALSE path: all other statuses  
    - Edge Cases: Missing or null `status` fields, case mismatches, unexpected status values  
  - Sticky Note - Filtering Logic  
    - Explains the branching rationale and destination of each path

---

#### 1.4 Analytics Pipeline

- **Overview:**  
  Processes won deals by logging raw data into Google Sheets, calculating aggregated lead source metrics, and sending a report summary to Slack.

- **Nodes Involved:**  
  - Log Won Deals to Sheets  
  - Calculate Lead Source Metrics  
  - Send Analytics to Slack  
  - Sticky Note - Analytics

- **Node Details:**  
  - **Log Won Deals to Sheets**  
    - Type: Google Sheets node  
    - Configuration:  
      - Sheet ID: User must specify their Google Sheet ID  
      - Range: `Sheet1!A1:D1` (starting range for data logging)  
      - Authentication: OAuth2 Google Sheets credential  
    - Input: Won deals from If node TRUE path  
    - Output: Data passed to function node for metrics calculation  
    - Edge Cases: Invalid sheet ID, authentication errors, quota limits, range conflicts  
  - **Calculate Lead Source Metrics**  
    - Type: Function node (JavaScript code execution)  
    - Configuration:  
      - Code calculates total number of deals and sum of amounts per lead source by iterating over input items  
      - Outputs a single JSON object summarizing deals and amounts by lead source  
    - Input: Logged won deals data  
    - Output: Aggregated metrics JSON  
    - Edge Cases: Missing `leadSource` or `amount` fields, non-numeric amounts, empty input arrays  
  - **Send Analytics to Slack**  
    - Type: Slack node (message sender)  
    - Configuration:  
      - Channel: `#lead-source-report`  
      - Text: JSON stringified summary from previous node, formatted with indentation for readability  
      - Authentication: OAuth2 Slack account credential  
    - Input: Metrics JSON  
    - Edge Cases: Slack API rate limits, invalid channel, authentication issues  
  - Sticky Note - Analytics  
    - Describes this multi-step analytics and reporting pipeline

---

#### 1.5 Non-Won Status Alerts

- **Overview:**  
  For opportunities not marked as won, individual Slack alerts are sent to notify the team of lost, pending, or other deal statuses.

- **Nodes Involved:**  
  - Alert Non-Won Status to Slack  
  - Sticky Note - Status Alerts

- **Node Details:**  
  - **Alert Non-Won Status to Slack**  
    - Type: Slack node  
    - Configuration:  
      - Channel: Slack channel with ID `C09GNB90TED` (likely a general or alerts channel)  
      - Text: Template message including opportunity id and status, e.g., "Opportunity id: 1234 is lost"  
      - Authentication: OAuth2 Slack credential  
    - Input: Opportunities from If node FALSE path  
    - Edge Cases: Missing opportunity id or status, Slack message send failures, incorrect channel ID  
  - Sticky Note - Status Alerts  
    - Intent behind sending real-time notifications about non-won deals

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                         | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                              |
|-------------------------------|---------------------|---------------------------------------|----------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Manual start of workflow               | None                             | Fetch All HighLevel Opportunities | Workflow Overview: Explains overall workflow start and purpose                                        |
| Fetch All HighLevel Opportunities | HighLevel          | Fetch all opportunities from CRM      | When clicking ‘Execute workflow’ | Check If Status = Won           | Sticky Note - Fetch Data: Describes data source fetching block                                        |
| Check If Status = Won          | If                  | Filter opportunities by "won" status  | Fetch All HighLevel Opportunities | Log Won Deals to Sheets (True path), Alert Non-Won Status to Slack (False path) | Sticky Note - Filtering Logic: Explains splitting based on deal status                                |
| Log Won Deals to Sheets        | Google Sheets       | Log won deals into Google Sheet       | Check If Status = Won (True)     | Calculate Lead Source Metrics   | Sticky Note - Analytics: Part of analytics pipeline with logging and metrics                          |
| Calculate Lead Source Metrics  | Function            | Aggregate deals and amounts by source | Log Won Deals to Sheets          | Send Analytics to Slack         | Sticky Note - Analytics                                                                                 |
| Send Analytics to Slack        | Slack               | Send summary report to Slack channel  | Calculate Lead Source Metrics    | None                           | Sticky Note - Analytics                                                                                 |
| Alert Non-Won Status to Slack  | Slack               | Notify Slack about non-won opportunities | Check If Status = Won (False)    | None                           | Sticky Note - Status Alerts: Explains non-won deal alerts                                             |
| Sticky Note - Workflow Overview | Sticky Note         | Documentation note                    | None                             | None                           | See content                                                                                             |
| Sticky Note - Fetch Data       | Sticky Note         | Documentation note                    | None                             | None                           | See content                                                                                             |
| Sticky Note - Filtering Logic  | Sticky Note         | Documentation note                    | None                             | None                           | See content                                                                                             |
| Sticky Note - Analytics        | Sticky Note         | Documentation note                    | None                             | None                           | See content                                                                                             |
| Sticky Note - Status Alerts    | Sticky Note         | Documentation note                    | None                             | None                           | See content                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: When clicking ‘Execute workflow’  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create HighLevel Node to Fetch Opportunities:**  
   - Name: Fetch All HighLevel Opportunities  
   - Type: HighLevel  
   - Resource: opportunity  
   - Operation: getAll  
   - Return All: true  
   - No filters configured (fetch all)  
   - Credentials: Assign your configured HighLevel OAuth2 credentials  
   - Connect output of Manual Trigger to this node.

3. **Add If Node to Check Deal Status:**  
   - Name: Check If Status = Won  
   - Type: If  
   - Condition: `$json.status` equals `"won"` (case sensitive)  
   - Connect output of HighLevel node as input.

4. **Create Google Sheets Node to Log Won Deals:**  
   - Name: Log Won Deals to Sheets  
   - Type: Google Sheets  
   - Sheet ID: Enter your Google Sheet ID here  
   - Range: `Sheet1!A1:D1` (adjust as needed)  
   - Authentication: Use your Google Sheets OAuth2 credentials  
   - Connect TRUE output of If node to this node.

5. **Add Function Node to Calculate Metrics:**  
   - Name: Calculate Lead Source Metrics  
   - Type: Function  
   - Copy and paste this JavaScript code:
     ```javascript
     const sourceData = items.map(i => i.json);
     const result = {};

     sourceData.forEach(deal => {
       const source = deal.leadSource;
       if (!result[source]) result[source] = { deals: 0, totalAmount: 0 };
       result[source].deals += 1;
       result[source].totalAmount += parseFloat(deal.amount || 0);
     });

     return [{ json: result }];
     ```
   - Connect output of Google Sheets node to this node.

6. **Create Slack Node to Send Analytics Report:**  
   - Name: Send Analytics to Slack  
   - Type: Slack  
   - Channel: `#lead-source-report` (create or confirm this channel exists in Slack workspace)  
   - Text: `={{JSON.stringify($json, null, 2)}}` (stringify the JSON from previous node)  
   - Credentials: Use your Slack OAuth2 credentials  
   - Connect output of Function node to this node.

7. **Create Slack Node for Non-Won Notifications:**  
   - Name: Alert Non-Won Status to Slack  
   - Type: Slack  
   - Channel ID: Use the ID of the Slack channel for alerts (e.g., `C09GNB90TED`)  
   - Text template: `=Opportunity id: {{ $json.id }} is {{ $json.status }}`  
   - Credentials: Same Slack OAuth2 credentials as above  
   - Connect FALSE output of If node to this node.

8. **Add Sticky Notes for Documentation (Optional):**  
   - For each logical block, add sticky notes with summaries matching those in the original workflow for clarity and maintainability.

9. **Activate and Test:**  
   - Ensure credentials are properly configured and authorized.  
   - Run the manual trigger and verify data flows correctly through all nodes.  
   - Check Google Sheets for logged deals and Slack for messages in both channels.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                               |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow analyzes lead source quality by fetching opportunities, filtering won deals, computing metrics, and sending reports. | Workflow Overview Sticky Note                                  |
| Google Sheets node requires valid Sheet ID and OAuth2 credentials for data logging.              | Google Sheets node configuration                               |
| Slack nodes require OAuth2 credentials and correct channel IDs/names to send messages properly. | Slack node configuration                                       |
| HighLevel API integration depends on OAuth2 credentials and may be subject to API rate limits.   | HighLevel node configuration                                   |
| For further info on n8n Slack integration see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/ | Official n8n Slack Node Documentation                          |
| For details on Google Sheets integration see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.google-sheets/ | Official n8n Google Sheets Node Documentation                  |
| For HighLevel API usage within n8n, refer to HighLevel documentation and OAuth2 setup guides.    | HighLevel API and OAuth2 setup                                 |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. It fully complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.