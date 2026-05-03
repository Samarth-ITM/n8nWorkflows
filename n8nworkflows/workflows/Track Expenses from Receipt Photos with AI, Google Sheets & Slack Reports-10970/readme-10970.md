Track Expenses from Receipt Photos with AI, Google Sheets & Slack Reports

https://n8nworkflows.xyz/workflows/track-expenses-from-receipt-photos-with-ai--google-sheets---slack-reports-10970


# Track Expenses from Receipt Photos with AI, Google Sheets & Slack Reports

---

### 1. Workflow Overview

This workflow automates household expense tracking by converting receipt photos into structured budget data, maintaining a Google Sheets budget ledger, and reporting spending summaries to Slack. It covers both immediate updates after each receipt upload and comprehensive daily/monthly budget reports.

Logical blocks:

- **1.1 Receipt Input & AI Parsing**  
  Receive receipt photos via webhook, extract structured expense data (date, store, items, amount) using AI.

- **1.2 Budget Data Management**  
  Append parsed receipt data into a Google Sheets budget ledger.

- **1.3 Daily Expense Summary & Slack Notification**  
  After each receipt entry, aggregate current month spending, calculate remaining budget, and send a concise Slack message.

- **1.4 Scheduled Monthly Analysis & Detailed Report**  
  Triggered daily by a cron job, reads all budget data for the month, analyzes trends (top stores/items/days, averages), generates a detailed AI-written report, and sends it to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Receipt Input & AI Parsing

**Overview:**  
Receives a receipt photo upload via HTTP POST, extracts textual data, and uses an AI agent to parse the receipt details into structured JSON.

**Nodes Involved:**  
- Receipt Photo Upload (Webhook)  
- Parse Receipt (AI Agent)  
- OpenRouter Chat Model2 (AI Language Model)  
- Code in JavaScript (Parse AI output into JSON items)

**Node Details:**

- **Receipt Photo Upload**  
  - *Type:* Webhook (HTTP POST endpoint)  
  - *Configuration:* Path = `receipt Text`, method = POST  
  - *Input:* Incoming HTTP POST with receipt text payload (expects text at `$json.body[''].text`)  
  - *Output:* Raw receipt text forwarded to Parse Receipt  
  - *Edge cases:* Missing or malformed POST body, empty text field, webhook not activated  
  - *Version:* 2.1

- **Parse Receipt**  
  - *Type:* LangChain Agent node (AI agent)  
  - *Configuration:*  
    - System message instructs to extract date, store, items, amount in JSON array format  
    - Input text: `={{ $json.body[''].text }}` (receipt text from webhook)  
    - No extra commentary, output only JSON  
  - *Input:* Receipt text from webhook  
  - *Output:* AI-generated JSON string detailing receipt data  
  - *Edge cases:* AI output malformed or not JSON, incomplete data extraction  
  - *Version:* 3

- **OpenRouter Chat Model2**  
  - *Type:* LangChain language model node  
  - *Configuration:* Uses OpenRouter API key credential (must be set up)  
  - *Input:* Receives prompt from Parse Receipt node  
  - *Output:* Chat model response containing extracted receipt data  
  - *Edge cases:* API key missing/invalid, rate limiting, network timeout  
  - *Version:* 1

- **Code in JavaScript**  
  - *Type:* Code node  
  - *Configuration:* Parses AI output string to JSON, removes markdown formatting, ensures array format, maps each entry into n8n item structure with keys: Date, Store, Items, Amount  
  - *Key expressions:* Uses JS regex and JSON.parse with error handling  
  - *Input:* AI JSON string from OpenRouter Chat Model2  
  - *Output:* Array of structured JSON items suitable for Google Sheets append  
  - *Edge cases:* JSON parsing errors, unexpected AI output format  
  - *Version:* 2

---

#### 2.2 Budget Data Management

**Overview:**  
Appends the parsed receipt expense entries as new rows into a specified Google Sheets document.

**Nodes Involved:**  
- Add to Budget Sheet (Google Sheets)

**Node Details:**

- **Add to Budget Sheet**  
  - *Type:* Google Sheets node (Append rows)  
  - *Configuration:*  
    - Document ID: specific Google Sheet for household budget  
    - Sheet Name: gid=0 (default sheet)  
    - Columns mapped: Date, Store, Items, Amount from incoming JSON  
    - Operation: Append new rows  
  - *Input:* JSON items from Code in JavaScript node (parsed receipt entries)  
  - *Output:* Confirmation of row append, passes data downstream for further processing  
  - *Edge cases:* Credential issues, sheet access permission denied, invalid data types, Google API rate limits  
  - *Version:* 4.7

---

#### 2.3 Daily Expense Summary & Slack Notification

**Overview:**  
After adding new receipt data, calculates the current month's total spending, remaining budget, and sends a summary message to Slack.

**Nodes Involved:**  
- Code in JavaScript2 (Calculate monthly totals)  
- Report Budget (AI agent for Slack message formatting)  
- OpenRouter Chat Model (AI language model)  
- Send a message (Slack)

**Node Details:**

- **Code in JavaScript2**  
  - *Type:* Code node  
  - *Configuration:*  
    - Reads all items passed (assumed from Google Sheets append node)  
    - Parses dates, sums amounts for current month  
    - Applies fixed monthly budget (30000 JPY, user-editable)  
    - Creates a plain text summary message with budget, total spent, remaining, and alert if over budget  
  - *Input:* Items including newly appended rows  
  - *Output:* JSON with keys: month, budget, totalSpent, remainingBudget, message  
  - *Edge cases:* Empty dataset, invalid date or amount formats, zero or negative budget scenarios  
  - *Version:* 2

- **Report Budget**  
  - *Type:* LangChain agent node  
  - *Configuration:*  
    - System prompt: formats a polished Slack message summarizing total spent and remaining budget  
    - Input text: `={{ $json["message"] }}` (from previous node)  
  - *Input:* Text summary from Code in JavaScript2  
  - *Output:* Polished Slack message text in `output` JSON key  
  - *Edge cases:* AI model errors, network/API issues  
  - *Version:* 3

- **OpenRouter Chat Model**  
  - *Type:* LangChain language model node  
  - *Configuration:* Uses OpenRouter API key  
  - *Input:* Receives prompt from Report Budget node  
  - *Output:* Slack-ready formatted message  
  - *Edge cases:* API key issues, rate limits  
  - *Version:* 1

- **Send a message (Slack)**  
  - *Type:* Slack node  
  - *Configuration:*  
    - Auth: OAuth2 (Slack workspace connection)  
    - Channel: specific Slack channel ID (e.g., "C09SMMZGXMM")  
    - Text: `={{ $json.output }}` (message from OpenRouter Chat Model)  
  - *Input:* Slack message text  
  - *Output:* Confirmation of message sent  
  - *Edge cases:* Slack auth token expired, channel ID invalid, network errors  
  - *Version:* 2.3

---

#### 2.4 Scheduled Monthly Analysis & Detailed Report

**Overview:**  
Triggered daily by a cron job, reads the entire budget sheet, analyzes spending patterns for the current month, generates a detailed AI-written report, and sends it to Slack.

**Nodes Involved:**  
- Daily Report Trigger (Cron)  
- Get Budget Sheet (Daily) (Google Sheets read)  
- Monthly Analysis (Code in JavaScript)  
- Monthly Report (AI agent)  
- OpenRouter Chat Model1 (AI language model)  
- Send monthly report (Slack)

**Node Details:**

- **Daily Report Trigger**  
  - *Type:* Cron node  
  - *Configuration:* Default cron to trigger once per day (customizable)  
  - *Output:* Triggers workflow execution branch for daily budget report  
  - *Edge cases:* Timezone issues, misconfiguration causing no trigger  
  - *Version:* 1

- **Get Budget Sheet (Daily)**  
  - *Type:* Google Sheets node (Read rows)  
  - *Configuration:*  
    - Document ID & Sheet name same as used elsewhere  
    - Reads all rows (no filters)  
  - *Output:* All budget entries to downstream analysis  
  - *Edge cases:* Empty sheet, permission denied, API limits  
  - *Version:* 4.7

- **Monthly Analysis**  
  - *Type:* Code node  
  - *Configuration:*  
    - Parses all rows, filters entries for the latest month based on last row’s date  
    - Aggregates totals: total spending, daily average, top 3 stores, items, and spending days  
    - Outputs structured JSON with analytics data  
  - *Edge cases:* Empty data, malformed data, invalid dates or amounts  
  - *Version:* 2

- **Monthly Report**  
  - *Type:* LangChain agent node  
  - *Configuration:*  
    - System prompt requests a polite, informal English summary of monthly budget data with trends and advice  
    - Input text includes monthly totals, top spending entities serialized as JSON strings  
  - *Edge cases:* AI model failures, malformed input data  
  - *Version:* 3

- **OpenRouter Chat Model1**  
  - *Type:* LangChain language model node  
  - *Configuration:* Uses OpenRouter API key  
  - *Edge cases:* API authentication or connectivity issues  
  - *Version:* 1

- **Send monthly report (Slack)**  
  - *Type:* Slack node  
  - *Configuration:*  
    - OAuth2 authentication  
    - Channel set to same Slack channel as daily report  
    - Text from AI-generated monthly report output  
  - *Edge cases:* Slack API errors, invalid token, channel misconfiguration  
  - *Version:* 2.3

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                           | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                            |
|-------------------------|-----------------------------------|-----------------------------------------|-----------------------|-----------------------|------------------------------------------------------------------------------------------------------------------------|
| Receipt Photo Upload     | Webhook                           | Receive receipt text via HTTP POST      | —                     | Parse Receipt          |                                                                                                                        |
| Parse Receipt           | LangChain Agent                   | Extract structured data from receipt AI | Receipt Photo Upload   | Code in JavaScript     |                                                                                                                        |
| OpenRouter Chat Model2   | LangChain Language Model          | AI model for receipt parsing             | Parse Receipt         | Code in JavaScript     |                                                                                                                        |
| Code in JavaScript       | Code                              | Parse AI output to JSON items            | OpenRouter Chat Model2 | Add to Budget Sheet    |                                                                                                                        |
| Add to Budget Sheet      | Google Sheets                     | Append extracted receipt data            | Code in JavaScript     | Code in JavaScript2    |                                                                                                                        |
| Code in JavaScript2      | Code                              | Calculate monthly totals & remaining budget | Add to Budget Sheet    | Report Budget          |                                                                                                                        |
| Report Budget            | LangChain Agent                   | Format budget summary message for Slack | Code in JavaScript2    | OpenRouter Chat Model  |                                                                                                                        |
| OpenRouter Chat Model    | LangChain Language Model          | AI model for budget summary formatting   | Report Budget          | Send a message         |                                                                                                                        |
| Send a message           | Slack                            | Send daily budget message to Slack       | OpenRouter Chat Model  | —                     |                                                                                                                        |
| Daily Report Trigger     | Cron                             | Trigger daily budget report branch       | —                     | Get Budget Sheet (Daily) |                                                                                                                        |
| Get Budget Sheet (Daily) | Google Sheets                    | Read all budget entries                   | Daily Report Trigger   | Monthly Analysis       |                                                                                                                        |
| Monthly Analysis         | Code                             | Analyze monthly spending data             | Get Budget Sheet (Daily) | Monthly Report         |                                                                                                                        |
| Monthly Report           | LangChain Agent                  | Generate detailed monthly budget report  | Monthly Analysis       | OpenRouter Chat Model1 |                                                                                                                        |
| OpenRouter Chat Model1   | LangChain Language Model          | AI model for monthly report generation    | Monthly Report         | Send monthly report    |                                                                                                                        |
| Send monthly report      | Slack                           | Send monthly budget report to Slack      | OpenRouter Chat Model1 | —                     |                                                                                                                        |
| Sticky Note             | Sticky Note                      | Workflow overview and setup instructions | —                     | —                     | ## What it does\nThis workflow automates your household budget tracking in several steps: ...                          |
| Sticky Note1            | Sticky Note                      | Setup instructions and credentials guide | —                     | —                     | ### Steps\n1. Google Sheets Setup: Create/use a Google Sheet, share with n8n credentials, and select it in the workflow nodes. ... |
| Sticky Note3            | Sticky Note                      | Customization tips for triggers, AI, formatting | —                     | —                     | ## How to customize the workflow\n- Daily Report Trigger: Adjust the cron settings for frequency. ...                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receipt Photo Upload"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `receipt Text`  
   - Purpose: To receive receipt text payloads via HTTP POST.

2. **Create LangChain Agent Node: "Parse Receipt"**  
   - Type: LangChain Agent  
   - Input Text: `={{ $json.body[''].text }}` (extract receipt text from webhook payload)  
   - System Message: Instruct AI to extract date, store, items, amount in JSON array format; no extra commentary.  
   - Connect output of "Receipt Photo Upload" to this node's input.

3. **Create LangChain Language Model Node: "OpenRouter Chat Model2"**  
   - Type: LangChain Language Model  
   - Credentials: Set OpenRouter API key credential  
   - Connect "Parse Receipt" agent node's AI language model input to this node.

4. **Create Code Node: "Code in JavaScript"**  
   - Purpose: Parse AI output string to JSON items array, remove markdown if present, enforce array format.  
   - Code: Implement JSON parse with error handling, map each item to object with keys Date, Store, Items, Amount.  
   - Connect output of "OpenRouter Chat Model2" to this node.

5. **Create Google Sheets Node: "Add to Budget Sheet"**  
   - Operation: Append rows  
   - Document ID: Your Google Sheet ID for budget tracking  
   - Sheet Name: `gid=0` or actual sheet name  
   - Columns: Map Date, Store, Items, Amount from incoming JSON  
   - Credential: Google Sheets credential with access to your sheet  
   - Connect output of "Code in JavaScript" to this node.

6. **Create Code Node: "Code in JavaScript2"**  
   - Purpose: Calculate current month total spending and remaining budget (default 30000 JPY)  
   - Logic: Parse dates, sum amounts for current month, calculate remaining budget, prepare summary text.  
   - Connect output of "Add to Budget Sheet" to this node.

7. **Create LangChain Agent Node: "Report Budget"**  
   - Input Text: `={{ $json["message"] }}` (summary text from previous node)  
   - System Message: Format a Slack-friendly polished summary message (total spent, remaining budget).  
   - Connect output of "Code in JavaScript2" to this node.

8. **Create LangChain Language Model Node: "OpenRouter Chat Model"**  
   - Credentials: OpenRouter API key  
   - Connect AI language model input of "Report Budget" to this node.

9. **Create Slack Node: "Send a message"**  
   - Authentication: OAuth2 with Slack workspace  
   - Channel: Set your Slack channel ID  
   - Text: `={{ $json.output }}` from "OpenRouter Chat Model"  
   - Connect output of "OpenRouter Chat Model" to this node.

10. **Create Cron Node: "Daily Report Trigger"**  
    - Set to trigger once daily at desired time.

11. **Create Google Sheets Node: "Get Budget Sheet (Daily)"**  
    - Operation: Read rows  
    - Document ID and Sheet: Same as "Add to Budget Sheet"  
    - Connect output of "Daily Report Trigger" to this node.

12. **Create Code Node: "Monthly Analysis"**  
    - Purpose: Analyze all rows for current month: total spending, daily average, top 3 stores/items/days.  
    - Connect output of "Get Budget Sheet (Daily)" to this node.

13. **Create LangChain Agent Node: "Monthly Report"**  
    - Input Text: Provide JSON stringified data from Monthly Analysis node with a prompt to generate a polite, informal English report with bullet points and advice.  
    - System Message: Assistant that analyzes household budget data and writes clear English reports.  
    - Connect output of "Monthly Analysis" to this node.

14. **Create LangChain Language Model Node: "OpenRouter Chat Model1"**  
    - Credentials: OpenRouter API key  
    - Connect AI language model input of "Monthly Report" to this node.

15. **Create Slack Node: "Send monthly report"**  
    - OAuth2 Slack credential  
    - Channel: Same Slack channel as daily report  
    - Text: `={{ $json.output }}` from "OpenRouter Chat Model1"  
    - Connect output of "OpenRouter Chat Model1" to this node.

16. **Connect all nodes as described in the logical blocks**, enabling the flow:

    - Webhook -> AI Parsing -> JSON parse -> Google Sheets append -> Monthly total calculation -> Slack summary  
    - Cron -> Google Sheets read -> Monthly analysis -> AI report -> Slack monthly report

17. **Credential Setup**:  
    - Configure Google Sheets credentials with access to your sheet.  
    - Configure Slack OAuth2 credentials with permissions to post messages in desired channel.  
    - Configure OpenRouter API key credentials for all LangChain language model nodes.

18. **Adjust constants and parameters:**  
    - Edit monthly budget value in "Code in JavaScript2" node (`const budget = 30000;`) as needed.  
    - Customize Slack channel IDs in Slack nodes.  
    - Set cron schedule in "Daily Report Trigger" node for desired timing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow automates household budget tracking: receipt upload, AI parsing, Google Sheets ledger, daily and monthly Slack reports.                                                                                                                                       | Sticky Note at position [-656,-272] (Workflow overview)                                                  |
| Setup instructions: create Google Sheet with columns Date, Store, Items, Amount; share with n8n credentials; configure OpenRouter and Slack credentials; copy webhook URL for receipt upload; set monthly budget in code node. | Sticky Note1 at position [48,64]                                                                          |
| Customization tips: Adjust cron schedule for daily report trigger; swap AI models in LangChain nodes; customize Slack message content and tone; add integrations for other accounting or notification tools.                      | Sticky Note3 at position [288,-496]                                                                       |
| OpenRouter API key needed for all LangChain AI nodes.                                                                                                                                                                            | Credential requirement for all AI language model nodes                                                  |
| Google Sheets must have a sheet with columns exactly named: Date, Store, Items, Amount.                                                                                                                                         | Critical for correct data mapping in Append and Read nodes                                              |
| Slack OAuth2 credential must have permission to post messages in the configured channel.                                                                                                                                         | Required for Slack message nodes                                                                         |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---