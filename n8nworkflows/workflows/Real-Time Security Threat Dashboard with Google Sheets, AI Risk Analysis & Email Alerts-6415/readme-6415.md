Real-Time Security Threat Dashboard with Google Sheets, AI Risk Analysis & Email Alerts

https://n8nworkflows.xyz/workflows/real-time-security-threat-dashboard-with-google-sheets--ai-risk-analysis---email-alerts-6415


# Real-Time Security Threat Dashboard with Google Sheets, AI Risk Analysis & Email Alerts

### 1. Workflow Overview

This workflow, titled **"Real-Time Security Threat Dashboard with Google Sheets, AI Risk Analysis & Email Alerts"**, automates the daily ingestion, analysis, triage, and alerting of security threat data. It targets security operations teams aiming to maintain up-to-date threat intelligence dashboards and receive prioritized alerts for critical vulnerabilities or indicators of compromise (IOC). The workflow integrates multiple external data feeds, applies AI-driven risk assessment and triage, and routes alerts and logging to appropriate channels.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Scheduled daily trigger to fetch external threat intelligence feeds (CVE and IOC).
- **1.2 Data Aggregation & Preparation**: Merging and combining the threat data from multiple feeds into a single dataset.
- **1.3 AI-Driven Risk Analysis**: Using AI-powered code nodes to evaluate and triage the combined threat data.
- **1.4 Alerting & Reporting**: Conditional alert triggering, email notifications, Google Sheets logging, and routing to response sub-processes.
- **1.5 Incident Playbook Selection & Response Routing**: AI selection of incident playbooks and routing outputs for further action such as sending emails, logging, or HTTP requests.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow daily and retrieves security threat data from two external sources: a CVE feed and an IOC feed.

- **Nodes Involved:**  
  - ⏰ Cron – Daily Trigger  
  - 🌐 Get CVE Feed  
  - 🛡️ Get IOC Feed

- **Node Details:**  

  - **⏰ Cron – Daily Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow execution once per day.  
    - Configuration: Default daily schedule (exact timing not specified).  
    - Inputs: None (start node).  
    - Outputs: Triggers two parallel HTTP request nodes.  
    - Edge Cases: Cron misfire or scheduler downtime could delay feed updates.  

  - **🌐 Get CVE Feed**  
    - Type: HTTP Request  
    - Role: Fetches Common Vulnerabilities and Exposures (CVE) threat data from an external API or feed.  
    - Configuration: URL and authentication not explicit here but expected to be configured in node parameters.  
    - Inputs: Trigger from Cron node.  
    - Outputs: Data forwarded to the merge node.  
    - Edge Cases: HTTP errors, timeout, or malformed feed data could disrupt processing.  

  - **🛡️ Get IOC Feed**  
    - Type: HTTP Request  
    - Role: Retrieves Indicators of Compromise (IOC) from a different external source.  
    - Configuration: Similar to CVE feed with HTTP settings for the IOC data source.  
    - Inputs: Trigger from Cron node.  
    - Outputs: Data forwarded to the merge node.  
    - Edge Cases: Similar to CVE feed node (connection errors, invalid responses).

---

#### 2.2 Data Aggregation & Preparation

- **Overview:**  
  Aggregates the CVE and IOC data streams into a unified dataset and prepares it for AI analysis.

- **Nodes Involved:**  
  - 🧠 Merge Threat Data  
  - 🧠Combine Threat Data

- **Node Details:**  

  - **🧠 Merge Threat Data**  
    - Type: Merge  
    - Role: Combines the two input data streams (CVE and IOC feeds) into one stream.  
    - Configuration: Default merge mode, likely "Merge by Index" or "Append".  
    - Inputs: Outputs from both feed HTTP request nodes.  
    - Outputs: Merged data sent to a code node for further combination.  
    - Edge Cases: Data format inconsistencies may cause merge issues.  

  - **🧠Combine Threat Data**  
    - Type: Code (JavaScript)  
    - Role: Performs custom data processing to combine and normalize merged threat data for AI input.  
    - Configuration: Contains code logic to unify data schema, filter, or enrich records.  
    - Inputs: Data from the merge node.  
    - Outputs: Cleaned combined dataset forwarded to AI risk evaluation.  
    - Edge Cases: Code errors or unexpected data structures could break execution.

---

#### 2.3 AI-Driven Risk Analysis

- **Overview:**  
  Applies AI logic to evaluate risk levels of threats, triage vulnerabilities, and prepare data for alerting and incident handling.

- **Nodes Involved:**  
  - 🧠 AI – Risk Evaluation  
  - 🧠 AI – Triage Vulnerabilities  
  - Split Out  

- **Node Details:**  

  - **🧠 AI – Risk Evaluation**  
    - Type: Code (JavaScript)  
    - Role: Implements AI-driven algorithms or heuristics to score and rank threats by risk severity.  
    - Configuration: Custom code, potentially invoking AI APIs or local logic for risk scoring.  
    - Inputs: Combined threat data.  
    - Outputs: Evaluated threat data with risk metadata.  
    - Edge Cases: AI service unavailability, API limits, or code exceptions.  

  - **🧠 AI – Triage Vulnerabilities**  
    - Type: Code (JavaScript)  
    - Role: Further triages vulnerabilities based on risk evaluation to identify actionable threats.  
    - Configuration: Custom triage logic.  
    - Inputs: Risk-evaluated threat data.  
    - Outputs: Triage results forwarded to alert trigger and split node.  
    - Edge Cases: Logic errors or missing data fields could impair triage.  

  - **Split Out**  
    - Type: SplitOut  
    - Role: Splits the triaged threat data into individual items for parallel incident playbook selection.  
    - Configuration: Default split one item per output.  
    - Inputs: Output from triage node.  
    - Outputs: Feeds into Incident Playbook Selector code node.  
    - Edge Cases: Empty input or malformed arrays.

---

#### 2.4 Alerting & Reporting

- **Overview:**  
  Conditionally triggers alerts based on triage results, sends notification emails, and logs incidents to Google Sheets.

- **Nodes Involved:**  
  - 🚨 ALERT – LEV Trigger  
  - 📧 Send Alert Email  
  - Google Sheets

- **Node Details:**  

  - **🚨 ALERT – LEV Trigger**  
    - Type: If  
    - Role: Checks for conditions (e.g., risk above threshold) to decide if an alert should be sent.  
    - Configuration: Conditional expression evaluating threat risk level or severity.  
    - Inputs: Output from triage node.  
    - Outputs: True branch triggers email and logging; false branch is not connected (no action).  
    - Edge Cases: Incorrect condition logic may cause missed or false alerts.  

  - **📧 Send Alert Email**  
    - Type: Email Send  
    - Role: Sends email alerts to designated recipients with threat details.  
    - Configuration: SMTP or other email credentials configured externally. Email content likely templated with threat info.  
    - Inputs: True output from alert trigger node.  
    - Outputs: None downstream within this block.  
    - Edge Cases: Email sending failures, SMTP errors, or invalid recipient addresses.  

  - **Google Sheets**  
    - Type: Google Sheets  
    - Role: Logs alert/incident data into a Google Sheets document for dashboarding and record keeping.  
    - Configuration: Google Sheets API credentials configured; target spreadsheet and worksheet specified.  
    - Inputs: True output from alert trigger node.  
    - Outputs: None downstream in this block.  
    - Edge Cases: API quota limits, permission issues, or invalid sheet identifiers.

---

#### 2.5 Incident Playbook Selection & Response Routing

- **Overview:**  
  Uses AI to select an appropriate incident response playbook per threat and routes the response actions accordingly, including sending emails, logging, or making HTTP requests.

- **Nodes Involved:**  
  - 🧠 AI – Incident Playbook Selector  
  - Code  
  - 🧭 Response Router  
  - Send Alert Email (second email node)  
  - Log to Google Sheet (second Google Sheets node)  
  - HTTP Request (second HTTP Request node)  
  - Split Out (from prior block feeds here)

- **Node Details:**  

  - **🧠 AI – Incident Playbook Selector**  
    - Type: Code (JavaScript)  
    - Role: AI logic to determine the most suitable incident playbook based on individual threat details.  
    - Configuration: Custom code using threat data input.  
    - Inputs: Split Out node output (individual threat items).  
    - Outputs: Sends results to the next Code node.  
    - Edge Cases: AI logic errors or incomplete data could misclassify playbooks.  

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Processes playbook selection results, possibly formatting or enriching data for routing.  
    - Configuration: Custom logic.  
    - Inputs: Output from Incident Playbook Selector.  
    - Outputs: Routes to the Response Router switch node.  
    - Edge Cases: Script errors or unexpected input format.  

  - **🧭 Response Router**  
    - Type: Switch  
    - Role: Routes execution flow based on playbook or threat type, directing to email, logging, or HTTP request nodes.  
    - Configuration: Multiple case conditions based on playbook names or threat categories.  
    - Inputs: Output from Code node.  
    - Outputs:  
      - Case 1: Send Alert Email  
      - Case 2: Log to Google Sheet  
      - Case 3: HTTP Request  
    - Edge Cases: Missing or unmatched cases may cause dropped messages.

  - **Send Alert Email (second)**  
    - Type: Email Send  
    - Role: Sends incident-specific alert emails based on routed playbook cases.  
    - Configuration: Email credentials set; templated content based on playbook data.  
    - Inputs: Routed from Response Router.  
    - Outputs: None downstream.  
    - Edge Cases: Similar to earlier email node.  

  - **Log to Google Sheet (second)**  
    - Type: Google Sheets  
    - Role: Logs incident or response details to a Google Sheet for tracking.  
    - Configuration: Configured for a different sheet or tab than earlier logging node.  
    - Inputs: Routed from Response Router.  
    - Outputs: None downstream.  
    - Edge Cases: Similar to earlier Google Sheets node.  

  - **HTTP Request (second)**  
    - Type: HTTP Request  
    - Role: Sends data to an external system or webhook as part of the incident response.  
    - Configuration: URL and authentication parameters set accordingly.  
    - Inputs: Routed from Response Router.  
    - Outputs: None downstream.  
    - Edge Cases: Network errors, invalid URLs, or authentication failures.

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                         | Input Node(s)                   | Output Node(s)                        | Sticky Note |
|----------------------------|--------------------|---------------------------------------|--------------------------------|-------------------------------------|-------------|
| ⏰ Cron – Daily Trigger      | Schedule Trigger   | Daily workflow start                   | None                           | 🌐 Get CVE Feed, 🛡️ Get IOC Feed    |             |
| 🌐 Get CVE Feed             | HTTP Request       | Fetch CVE threat feed                  | ⏰ Cron – Daily Trigger          | 🧠 Merge Threat Data                 |             |
| 🛡️ Get IOC Feed            | HTTP Request       | Fetch IOC threat feed                  | ⏰ Cron – Daily Trigger          | 🧠 Merge Threat Data                 |             |
| 🧠 Merge Threat Data        | Merge              | Combine CVE and IOC feeds              | 🌐 Get CVE Feed, 🛡️ Get IOC Feed| 🧠Combine Threat Data                |             |
| 🧠Combine Threat Data       | Code               | Normalize and combine threat data     | 🧠 Merge Threat Data            | 🧠 AI – Risk Evaluation              |             |
| 🧠 AI – Risk Evaluation     | Code               | AI-based risk scoring                  | 🧠Combine Threat Data           | 🧠 AI – Triage Vulnerabilities, Split Out |             |
| 🧠 AI – Triage Vulnerabilities | Code            | AI-based triage of vulnerabilities    | 🧠 AI – Risk Evaluation         | 🚨 ALERT – LEV Trigger               |             |
| Split Out                  | SplitOut           | Split triaged data into individual items | 🧠 AI – Triage Vulnerabilities | 🧠 AI – Incident Playbook Selector   |             |
| 🚨 ALERT – LEV Trigger      | If                 | Conditional alert triggering           | 🧠 AI – Triage Vulnerabilities  | 📧 Send Alert Email, Google Sheets  |             |
| 📧 Send Alert Email         | Email Send         | Send alert email notifications         | 🚨 ALERT – LEV Trigger          | None                               |             |
| Google Sheets              | Google Sheets      | Log alerts/incidents to Google Sheets  | 🚨 ALERT – LEV Trigger          | None                               |             |
| 🧠 AI – Incident Playbook Selector | Code         | AI selects incident response playbook | Split Out                      | Code                               |             |
| Code                       | Code               | Process playbook selection result      | 🧠 AI – Incident Playbook Selector | 🧭 Response Router               |             |
| 🧭 Response Router          | Switch             | Route response based on playbook       | Code                          | Send Alert Email, Log to Google Sheet, HTTP Request |             |
| Send Alert Email (second)  | Email Send         | Send playbook-based alert emails       | 🧭 Response Router              | None                               |             |
| Log to Google Sheet (second)| Google Sheets      | Log playbook-based incident data       | 🧭 Response Router              | None                               |             |
| HTTP Request (second)      | HTTP Request       | Send data to external system/webhook   | 🧭 Response Router              | None                               |             |
| Sticky Note                | Sticky Note        | Visual note                            | None                           | None                               |             |
| Sticky Note1               | Sticky Note        | Visual note                            | None                           | None                               |             |
| Sticky Note2               | Sticky Note        | Visual note                            | None                           | None                               |             |
| Sticky Note3               | Sticky Note        | Visual note                            | None                           | None                               |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: `⏰ Cron – Daily Trigger`  
   - Type: Schedule Trigger  
   - Set to trigger once daily at a desired time (e.g., midnight).

2. **Create two HTTP Request nodes:**  
   - Name: `🌐 Get CVE Feed`  
     - Type: HTTP Request  
     - Configure URL for CVE feed API endpoint.  
     - Set method (GET), authentication if required.  
   - Name: `🛡️ Get IOC Feed`  
     - Type: HTTP Request  
     - Configure URL for IOC feed API endpoint.  
     - Set method and authentication similarly.

3. **Connect:**  
   - Connect output of `⏰ Cron – Daily Trigger` to both `🌐 Get CVE Feed` and `🛡️ Get IOC Feed` nodes.

4. **Add a Merge node:**  
   - Name: `🧠 Merge Threat Data`  
   - Type: Merge  
   - Configure to merge inputs from `🌐 Get CVE Feed` and `🛡️ Get IOC Feed`.  
   - Use "Append" or "Merge By Index" depending on data.

5. **Add a Code node:**  
   - Name: `🧠Combine Threat Data`  
   - Type: Code (JavaScript)  
   - Configure with logic to unify and normalize merged threat data (e.g., map fields, filter irrelevant items).  
   - Connect output of `🧠 Merge Threat Data` to this node.

6. **Add a Code node for AI risk evaluation:**  
   - Name: `🧠 AI – Risk Evaluation`  
   - Type: Code (JavaScript)  
   - Configure with AI risk scoring logic, optionally call external AI API or embed heuristic scoring.  
   - Connect output of `🧠Combine Threat Data` here.

7. **Add a Code node for AI triage:**  
   - Name: `🧠 AI – Triage Vulnerabilities`  
   - Type: Code (JavaScript)  
   - Configure to triage vulnerabilities based on risk scores (e.g., filter high-risk ones).  
   - Connect output of `🧠 AI – Risk Evaluation`.

8. **Add an If node:**  
   - Name: `🚨 ALERT – LEV Trigger`  
   - Type: If  
   - Configure condition to check if risk score or severity exceeds alert threshold.  
   - Connect output of `🧠 AI – Triage Vulnerabilities`.

9. **Add Email Send node:**  
   - Name: `📧 Send Alert Email`  
   - Type: Email Send  
   - Configure SMTP or other email credentials.  
   - Design email template with threat details.  
   - Connect "true" output of alert trigger node here.

10. **Add Google Sheets node:**  
    - Name: `Google Sheets`  
    - Type: Google Sheets  
    - Configure with Google API credentials and select target spreadsheet and worksheet.  
    - Connect "true" output of alert trigger node here.

11. **Add SplitOut node:**  
    - Name: `Split Out`  
    - Type: SplitOut  
    - Connect output of `🧠 AI – Triage Vulnerabilities` (parallel to alert trigger).  
    - Purpose: split data for individual incident processing.

12. **Add Incident Playbook Selector Code node:**  
    - Name: `🧠 AI – Incident Playbook Selector`  
    - Type: Code (JavaScript)  
    - Configure to select incident response playbook based on individual threat data.  
    - Connect output of `Split Out`.

13. **Add Code node:**  
    - Name: `Code`  
    - Type: Code (JavaScript)  
    - Implement any necessary formatting or enrichment of playbook selection results.  
    - Connect output of Incident Playbook Selector here.

14. **Add Switch node:**  
    - Name: `🧭 Response Router`  
    - Type: Switch  
    - Configure cases based on playbook or threat type to route to different response nodes.  
    - Connect output of `Code` node here.

15. **Add Email Send node for playbook alerts:**  
    - Name: `Send Alert Email`  
    - Type: Email Send  
    - Configure email credentials and template.  
    - Connect one case output of switch node here.

16. **Add Google Sheets node for playbook logging:**  
    - Name: `Log to Google Sheet`  
    - Type: Google Sheets  
    - Configure with Google credentials and spreadsheet for logging.  
    - Connect another case output of switch node here.

17. **Add HTTP Request node for external system notification:**  
    - Name: `HTTP Request`  
    - Type: HTTP Request  
    - Configure with URL and authentication for external API/webhook.  
    - Connect third case output of switch node here.

18. **Test and Activate:**  
    - Ensure all credentials (email, Google Sheets, HTTP API) are correctly configured.  
    - Test each node individually and run the workflow manually to verify data flow and outputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow requires valid credentials for Google Sheets API, SMTP email sending, and any external API feeds. | Credential setup in n8n nodes is essential.     |
| The AI logic nodes are custom JavaScript code nodes; integration with external AI APIs (e.g., OpenAI) may require API keys and rate limits management. | Consider API quota and error handling.          |
| Use descriptive email templates and Google Sheets layouts to enhance usability of alerts and dashboards.        | Customize as per organizational needs.          |
| Ensure network access and firewall permissions allow HTTP requests to external threat feeds and APIs.            | Network configuration is critical for operation.|
| Sticky notes in the workflow are placeholders for additional documentation or instructions if needed.           | No specific content provided in current workflow.|

---

This documentation provides a detailed blueprint of the workflow's architecture, node-by-node roles, and instructions to recreate it fully within n8n, enabling efficient maintenance, customization, or extension.