Enhance Security Operations with the Qualys Slack Shortcut Bot!

https://n8nworkflows.xyz/workflows/enhance-security-operations-with-the-qualys-slack-shortcut-bot--2510


# Enhance Security Operations with the Qualys Slack Shortcut Bot!

### 1. Workflow Overview

This n8n workflow, titled **"Enhance Security Operations with the Qualys Slack Shortcut Bot!"**, is designed to enable Slack users to initiate Qualys vulnerability scans and generate security reports directly via Slack interactions. The workflow leverages Slack’s interactive modals for input collection and dynamically triggers sub-workflows to conduct scans or create reports in Qualys, providing real-time feedback within Slack channels.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception and Parsing**: Captures Slack events via webhook and extracts actionable payload data.
- **1.2 Interaction Routing**: Routes incoming Slack interactions (commands, modal submissions) to appropriate processing paths based on callback IDs and modal titles.
- **1.3 Modal Presentation**: Opens interactive Slack modals to collect user inputs for either launching a vulnerability scan or generating a scan report.
- **1.4 Submission Processing**: Extracts user inputs from modal submissions and prepares variables for downstream Qualys operations.
- **1.5 Execution of Qualys Sub-Workflows**: Invokes dedicated sub-workflows responsible for starting vulnerability scans or creating reports.
- **1.6 Slack Response Management**: Handles Slack API responses to acknowledge commands and close modals, ensuring smooth UX.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parsing

- **Overview:**  
Receives incoming POST requests from Slack via webhook. The payload containing user interaction details is extracted and transformed into a format usable by subsequent nodes.

- **Nodes Involved:**  
  - Webhook  
  - Parse Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook Trigger  
    - Role: Entry point for Slack event subscriptions and shortcut commands via POST requests.  
    - Configuration: Listens on specific webhook path; HTTP POST method; response managed by downstream nodes.  
    - Connections: Output to Parse Webhook.  
    - Failure Modes: Network errors, invalid payloads, unexpected Slack request formats.

  - **Parse Webhook**  
    - Type: Set  
    - Role: Extracts and assigns the `payload` object from the Slack POST body (`$json.body.payload`) into a top-level `response` JSON attribute.  
    - Configuration: Uses expression to parse `payload` for easier access.  
    - Input: Receives raw webhook data. Output: Structured Slack interaction data.  
    - Failure Modes: Missing or malformed payload, JSON parse errors.

---

#### 2.2 Interaction Routing

- **Overview:**  
Determines the type of Slack interaction received—whether it is a shortcut trigger to open a modal or a modal submission—and routes the flow accordingly to handle scan initiation or report generation.

- **Nodes Involved:**  
  - Route Message (Switch)  
  - Respond to Slack Webhook - Vulnerability (RespondToWebhook)  
  - Respond to Slack Webhook - Report (RespondToWebhook)  
  - Close Modal Popup (RespondToWebhook)  
  - Route Submission (Switch)

- **Node Details:**

  - **Route Message**  
    - Type: Switch  
    - Role: Routes based on `callback_id` or `type` in the parsed Slack payload:  
      - `trigger-qualys-vmscan` → triggers Vulnerability Scan Modal  
      - `qualys-scan-report` → triggers Scan Report Modal  
      - `view_submission` → processes modal submissions  
    - Configuration: Multiple condition checks on JSON properties.  
    - Input: Parsed Slack payload. Output: One of three distinct routes.  
    - Failure Modes: Unknown or missing `callback_id`, unexpected interaction types.

  - **Respond to Slack Webhook - Vulnerability**  
    - Type: RespondToWebhook  
    - Role: Sends an empty 200 OK response (no data) to Slack upon receiving a vulnerability scan shortcut command, acknowledging receipt immediately.  
    - Failure Modes: Slack API errors if webhook response is delayed.

  - **Respond to Slack Webhook - Report**  
    - Type: RespondToWebhook  
    - Role: Sends a 200 OK with no data for scan report shortcut commands to acknowledge Slack.  
    - Failure Modes: Same as above.

  - **Close Modal Popup**  
    - Type: RespondToWebhook  
    - Role: Sends a 204 No Content response to Slack to close modal after submission.  
    - Output: Routes to Route Submission for further processing.  
    - Failure Modes: Slack modal closure failures, timeout.

  - **Route Submission**  
    - Type: Switch  
    - Role: Routes modal submission based on modal title text:  
      - "Vulnerability Scan" → Vuln Scan path  
      - "Scan Report Generator" → Scan Report path  
    - Input: Parsed modal submission data. Output: One of two paths.  
    - Failure Modes: Unrecognized modal title causing routing failure.

---

#### 2.3 Modal Presentation

- **Overview:**  
Upon shortcut trigger, opens a Slack modal for either vulnerability scan initiation or scan report generation, collecting necessary parameters interactively.

- **Nodes Involved:**  
  - Vuln Scan Modal (HTTP Request)  
  - Scan Report Task Modal (HTTP Request)

- **Node Details:**

  - **Vuln Scan Modal**  
    - Type: HTTP Request  
    - Role: Calls Slack API `views.open` to display a modal titled "Vulnerability Scan" with input blocks for:  
      - Option Title (text input)  
      - Scan Title (text input with default)  
      - Asset Groups (text input)  
    - Configuration: Uses `trigger_id` from parsed Slack payload to open modal. JSON body contains modal structure including Qualys logo and input hints.  
    - Credentials: Slack API credential with required scopes.  
    - Failure Modes: Invalid `trigger_id`, Slack API rate limits, malformed JSON, auth errors.

  - **Scan Report Task Modal**  
    - Type: HTTP Request  
    - Role: Opens a Slack modal titled "Scan Report Generator" with inputs for:  
      - Report Template (external select from Qualys templates)  
      - Report Title (text input)  
      - Output Format (static select: PDF, HTML, CSV)  
    - Configuration: Similar to Vuln Scan Modal, using Slack API with proper JSON modal payload.  
    - Credentials: Slack API.  
    - Failure Modes: Same as Vuln Scan Modal, plus potential issues fetching external select options.

---

#### 2.4 Submission Processing

- **Overview:**  
Extracts user input variables from modal submission payloads and prepares structured data objects necessary for downstream Qualys API operations.

- **Nodes Involved:**  
  - Required Scan Variables (Set)  
  - Required Report Variables (Set)

- **Node Details:**

  - **Required Scan Variables**  
    - Type: Set  
    - Role: Extracts and assigns variables such as:  
      - `platformurl`: Qualys API base URL (static)  
      - `option_title`: from modal input  
      - `scan_title`: from modal input  
      - `asset_groups`: from modal input  
    - Input: Modal submission JSON (`response.view.state.values`)  
    - Output: Structured data for starting a vulnerability scan.  
    - Failure Modes: Missing inputs, malformed submission data.

  - **Required Report Variables**  
    - Type: Set  
    - Role: Extracts variables for report generation:  
      - `report_title` (custom text)  
      - `base_url` (Qualys API base URL)  
      - `output_format` (selected value)  
      - `template_name` (selected template text)  
    - Input: Modal submission JSON.  
    - Output: Data for report creation workflow.  
    - Failure Modes: Same as above.

---

#### 2.5 Execution of Qualys Sub-Workflows

- **Overview:**  
Invokes external sub-workflows dedicated to executing the actual Qualys API calls for either starting vulnerability scans or generating reports.

- **Nodes Involved:**  
  - Qualys Start Vulnerability Scan (Execute Workflow)  
  - Qualys Create Report (Execute Workflow)

- **Node Details:**

  - **Qualys Start Vulnerability Scan**  
    - Type: Execute Workflow  
    - Role: Triggers the sub-workflow responsible for launching a Qualys vulnerability scan using the prepared scan variables.  
    - Configuration: References workflow ID `"pYPh5FlGZgb36xZO"`.  
    - Input: Receives variables from Required Scan Variables node.  
    - Failure Modes: Sub-workflow errors, invalid scan parameters, Qualys API failures.

  - **Qualys Create Report**  
    - Type: Execute Workflow  
    - Role: Triggers sub-workflow to create a scan report with user-defined settings.  
    - Configuration: References workflow ID `"icSLX102kSS9zNdK"`.  
    - Input: Receives variables from Required Report Variables node.  
    - Failure Modes: Sub-workflow errors, invalid report parameters, Qualys API errors.

---

#### 2.6 Slack Response Management

- **Overview:**  
Ensures Slack receives proper acknowledgments and modal closure signals, maintaining a smooth and responsive user experience.

- **Nodes Involved:**  
  - Respond to Slack Webhook - Vulnerability  
  - Respond to Slack Webhook - Report  
  - Close Modal Popup

- **Node Details:**  
(Previously described in section 2.2)

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                                | Input Node(s)            | Output Node(s)                            | Sticky Note                                                                                     |
|-------------------------------|---------------------|-----------------------------------------------|--------------------------|------------------------------------------|------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook             | Entry point for Slack events                   | -                        | Parse Webhook                            | ![Imgur](https://uploads.n8n.io/templates/slack.png) Explains Slack Events API subscription usage|
| Parse Webhook                 | Set                 | Extracts Slack payload from webhook            | Webhook                  | Route Message                           | See above                                                                                      |
| Route Message                 | Switch              | Routes Slack interaction by callback_id/type  | Parse Webhook             | Respond to Slack Webhook - Vulnerability, Respond to Slack Webhook - Report, Close Modal Popup | Describes dynamic routing of Slack messages and modal submissions                              |
| Respond to Slack Webhook - Vulnerability | RespondToWebhook    | Acknowledges vulnerability scan shortcut      | Route Message             | Vuln Scan Modal                         |                                                                                                |
| Respond to Slack Webhook - Report | RespondToWebhook    | Acknowledges report generation shortcut        | Route Message             | Scan Report Task Modal                  |                                                                                                |
| Close Modal Popup             | RespondToWebhook    | Closes Slack modal after submission            | Route Message             | Route Submission                        |                                                                                                |
| Route Submission             | Switch              | Routes modal submission by modal title         | Close Modal Popup         | Required Scan Variables, Required Report Variables |                                                                                                |
| Required Scan Variables       | Set                 | Extracts variables for vulnerability scan      | Route Submission          | Qualys Start Vulnerability Scan         |                                                                                                |
| Required Report Variables     | Set                 | Extracts variables for report generation       | Route Submission          | Qualys Create Report                    |                                                                                                |
| Qualys Start Vulnerability Scan | Execute Workflow    | Triggers vulnerability scan sub-workflow       | Required Scan Variables   | -                                      |                                                                                                |
| Qualys Create Report          | Execute Workflow    | Triggers report creation sub-workflow           | Required Report Variables | -                                      |                                                                                                |
| Vuln Scan Modal              | HTTP Request         | Opens Slack modal for vulnerability scan input | Respond to Slack Webhook - Vulnerability | -                                  | ![Imgur](https://uploads.n8n.io/templates/qualysmodalscan.png) Explains modal popup usage      |
| Scan Report Task Modal       | HTTP Request         | Opens Slack modal for report generation input  | Respond to Slack Webhook - Report | -                                    | See above                                                                                      |
| Sticky Note                  | Sticky Note          | Workflow documentation and explanatory notes   | -                        | -                                      | Various notes covering multiple nodes, including setup instructions and tips                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `4f86c00d-ceb4-4890-84c5-850f8e5dec05`)  
   - Response Mode: Response Node

2. **Add Set Node "Parse Webhook"**  
   - Purpose: Parse Slack `payload` from incoming webhook body.  
   - Assign `response` to `{{$json.body.payload}}`.  
   - Connect Webhook output to this node.

3. **Add Switch Node "Route Message"**  
   - Conditions based on `response.callback_id` and `response.type`:  
     - If `callback_id` equals `trigger-qualys-vmscan` → output "Vuln Scan Modal"  
     - If `callback_id` equals `qualys-scan-report` → output "Scan Report Modal"  
     - If `type` equals `view_submission` → output "Process Submission"  
   - Connect Parse Webhook output to this node.

4. **Add RespondToWebhook Nodes**  
   - "Respond to Slack Webhook - Vulnerability": Respond with no data (200 OK)  
   - "Respond to Slack Webhook - Report": Respond with 200 OK, no data  
   - "Close Modal Popup": Respond with 204 No Content  
   - Connect Route Message outputs accordingly.

5. **Add HTTP Request Nodes for Modals**  
   - "Vuln Scan Modal":  
     - Method: POST  
     - URL: `https://slack.com/api/views.open`  
     - Authentication: Slack API credentials with appropriate scopes  
     - Body: JSON modal object for vulnerability scan inputs (Option Title, Scan Title, Asset Groups)  
     - Use expression for `trigger_id` from `Parse Webhook` node.  
     - Connect from "Respond to Slack Webhook - Vulnerability".

   - "Scan Report Task Modal":  
     - Similar to above, but modal includes Report Template (external select), Report Title, Output Format.  
     - Connect from "Respond to Slack Webhook - Report".

6. **Add Switch Node "Route Submission"**  
   - Condition on `response.view.title.text`:  
     - "Vulnerability Scan" → output "Vuln Scan"  
     - "Scan Report Generator" → output "Scan Report"  
   - Connect from "Close Modal Popup".

7. **Add Set Nodes for Variable Extraction**  
   - "Required Scan Variables":  
     - Set static `platformurl` to Qualys API base URL.  
     - Extract `option_title`, `scan_title`, `asset_groups` from modal submission values.  
     - Connect from "Vuln Scan" output of Route Submission.

   - "Required Report Variables":  
     - Set static `base_url` to Qualys API base URL.  
     - Extract `report_title`, `output_format`, `template_name` from modal submission.  
     - Connect from "Scan Report" output of Route Submission.

8. **Add Execute Workflow Nodes**  
   - "Qualys Start Vulnerability Scan":  
     - Reference the sub-workflow that handles launching scans (`workflowId`: `pYPh5FlGZgb36xZO`).  
     - Connect from "Required Scan Variables".

   - "Qualys Create Report":  
     - Reference the sub-workflow for creating reports (`workflowId`: `icSLX102kSS9zNdK`).  
     - Connect from "Required Report Variables".

9. **Credential Setup**  
   - Create Slack API credentials with scopes: `commands`, `channels:join`, `channels:history`, `channels:read`, `chat:write`, `chat:write.customize`, `files:read`, `files:write`.  
   - Add Qualys API credentials as required in the sub-workflows.

10. **Slack App Configuration**  
    - Use provided Slack App manifest JSON to set up the bot with global shortcuts and bot user.  
    - Update the interactivity request URLs in Slack app settings to point to the n8n webhook URLs.

11. **Testing and Validation**  
    - Test shortcut triggers in Slack to confirm modals open correctly.  
    - Submit modals to verify workflows execute and responses are sent.  
    - Confirm Qualys scans and reports are generated as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow designed to enhance security operations by integrating Slack with Qualys vulnerability scans and reporting. | Workflow description and operational flow section.                                                |
| Slack Modals provide mobile-friendly, interactive UI elements for user input.                        | Slack Modals documentation: https://api.slack.com/surfaces/modals                                  |
| Slack Events API subscription required to receive user interactions.                                | Slack Events API info: https://api.slack.com/apis/connections/events-api                           |
| Qualys API base URL used: `https://qualysapi.qg3.apps.qualys.com`                                   | Used in variable assignments for API calls.                                                       |
| Sub-workflows required for executing scans and report creation must be deployed separately.          | Links: https://n8n.io/workflows/2511-qualys-vulnerability-trigger-scan-subworkflow/ and https://n8n.io/workflows/2512-qualys-scan-slack-report-subworkflow/ |
| Slack App manifest JSON provided for easy app setup with correct permissions and shortcuts.          | Manifest includes bot user, OAuth scopes, shortcut definitions, and interactivity configuration.  |
| Remember to update Slack channel configurations within the sub-workflows to ensure messages post correctly. | Sticky note reminder within workflow.                                                             |
| Ensure Slack credentials are correctly configured in n8n HTTP request nodes for API access.          | Sticky note highlights ease of credential reuse.                                                  |
| Triggering the workflow in Slack is done via global shortcuts accessible through Slack UI (backslash command). | Sticky note with Slack shortcut usage screenshot.                                                 |

---

This comprehensive documentation enables users and automation agents alike to understand, reproduce, and maintain the **Qualys Slack Shortcut Bot** workflow in n8n, ensuring effective integration between Slack and Qualys for streamlined security operations.