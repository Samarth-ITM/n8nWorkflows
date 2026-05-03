Automatically Detect & Classify GitHub API Errors with GPT-4o to Airtable, Notion & Slack

https://n8nworkflows.xyz/workflows/automatically-detect---classify-github-api-errors-with-gpt-4o-to-airtable--notion---slack-10335


# Automatically Detect & Classify GitHub API Errors with GPT-4o to Airtable, Notion & Slack

### 1. Workflow Overview

This workflow automates the detection, classification, and routing of GitHub API error issues using OpenAI’s GPT-4o model, storing structured data in Airtable and Notion, and notifying relevant teams on Slack. It is designed to support SaaS engineering teams by streamlining error triage, documentation, and communication.

**Target Use Cases:**
- Automatically processing new or updated GitHub issues tagged as "bug" or "error"
- Leveraging AI (GPT-4o) for error classification, root cause analysis, severity assessment, and FAQ matching
- Routing issues to the appropriate internal teams (DevOps, Backend, Support, API)
- Logging detailed error records in Airtable and Notion databases
- Sending Slack alerts to notify teams about newly classified errors
- Handling workflow errors gracefully with Slack notifications

**Logical Blocks:**

- **1.1 Input Reception & Preparation**  
  Receives GitHub issue webhook events and prepares clean, structured JSON data for AI processing.

- **1.2 AI Classification & Parsing**  
  Uses GPT-4o to analyze issue text, classify the error, determine root cause and severity, and produce structured JSON output.

- **1.3 Intelligent Routing**  
  Based on the AI classification, routes the issue data to the correct internal team branch.

- **1.4 Database Logging & Slack Alerts**  
  Each team branch creates records in Airtable and Notion, then sends Slack notifications with summarized error details.

- **1.5 Error Handling**  
  Catches any workflow execution errors and sends detailed error alerts to Slack for debugging.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preparation

- **Overview:**  
  This block listens for GitHub webhook events related to issues (creation or update), filters for relevant issues, and extracts key issue metadata into a structured format.

- **Nodes Involved:**  
  - GitHub Issue Created/Updated  
  - Prepare GitHub Issue Data  
  - Trigger Configuration (Sticky Note)

- **Node Details:**

  - **GitHub Issue Created/Updated**  
    - Type: GitHub Trigger  
    - Role: Listens for GitHub "issues" events via webhook (configured webhook ID: `api-error-catalog-webhook`)  
    - Configuration: OAuth2 authentication, repository and owner dynamically set (URL mode)  
    - Inputs: Webhook event (issue created/updated)  
    - Outputs: Raw GitHub issue JSON  
    - Edge Cases: Webhook misconfiguration, OAuth token expiry, network failure

  - **Prepare GitHub Issue Data**  
    - Type: Set Node  
    - Role: Extracts and assigns key issue fields (title, body, URL, number, repo name, creation date) into simplified fields  
    - Key expressions: Uses expressions like `={{ $json.body.issue.title }}` to extract nested data  
    - Inputs: From GitHub trigger  
    - Outputs: Structured JSON with flattened issue data  
    - Edge Cases: Missing fields in webhook payload, malformed JSON

  - **Trigger Configuration (Sticky Note)**  
    - Type: Sticky Note  
    - Role: Documentation block describing the GitHub trigger behavior and data preparation

#### 1.2 AI Classification & Parsing

- **Overview:**  
  Invokes GPT-4o to analyze the issue text, classify the error type, root cause, severity, and suggest fixes, then parses the AI JSON output for downstream use.

- **Nodes Involved:**  
  - AI: Classify API Error (GPT-4o)  
  - LLM Model: GPT-4o  
  - Output Parser: Structured JSON Schema  
  - AI Memory Buffer  
  - Parse AI Classification Output (Code node)  
  - AI Classification (Sticky Note)

- **Node Details:**

  - **AI: Classify API Error (GPT-4o)**  
    - Type: LangChain Agent Node  
    - Role: Sends issue data to GPT-4o with a detailed system prompt instructing classification schema and output format  
    - Configuration: Custom system message detailing classification responsibilities and desired JSON output schema  
    - Inputs: Structured issue data from "Prepare GitHub Issue Data"  
    - Outputs: Raw AI response JSON  
    - Version-specific: Requires LangChain nodes integration  
    - Edge Cases: API rate limits, malformed AI output, timeout, invalid JSON response

  - **LLM Model: GPT-4o**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides language model for AI node (GPT-4o with temperature 0.7)  
    - Configuration: OpenAI API credentials required  
    - Inputs: Prompt from AI agent node  
    - Outputs: Model response to AI node

  - **Output Parser: Structured JSON Schema**  
    - Type: LangChain structured output parser  
    - Role: Enforces AI response to comply with expected JSON schema example  
    - Inputs: AI response text  
    - Outputs: Parsed JSON object

  - **AI Memory Buffer**  
    - Type: LangChain memory buffer window  
    - Role: Maintains session context keyed by "AI Agent - Error Analyzer" to enable conversational context if needed  
    - Inputs/Outputs: Connects with AI node for memory management

  - **Parse AI Classification Output**  
    - Type: Code Node (JavaScript)  
    - Role: Re-formats AI JSON output along with original issue data to build objects formatted for Airtable, Notion, and Slack consumption  
    - Key variables: Extracts AI fields such as errorCode, category, severity, assignedTeam, and constructs formatted messages and field objects  
    - Inputs: AI classification JSON, original issue data  
    - Outputs: Enriched JSON with multi-platform formatted fields  
    - Edge Cases: Missing or incomplete AI fields, data inconsistency

  - **AI Classification (Sticky Note)**  
    - Type: Sticky Note  
    - Role: Documentation describing AI classification, fields extracted, and output formatting

#### 1.3 Intelligent Routing

- **Overview:**  
  Validates AI output and routes the enriched data to the appropriate internal team branch for further logging and notification.

- **Nodes Involved:**  
  - Check: Valid AI Response or Data Missing (If node)  
  - Route by Assigned Team (Switch node)  
  - Response Parsing (Sticky Note)

- **Node Details:**

  - **Check: Valid AI Response or Data Missing**  
    - Type: If Node  
    - Role: Checks if AI output contains valid ID or non-empty Airtable fields before routing  
    - Logic: Routes only if data is valid; otherwise, could drop or handle differently (not shown)  
    - Inputs: Parsed AI classification output  
    - Outputs: True (valid data) or False (invalid/missing data)  
    - Edge Cases: Empty or malformed AI output

  - **Route by Assigned Team**  
    - Type: Switch Node  
    - Role: Routes data based on `assignedTeam` field (DevOps Team, Backend Team, Support Team, API Team)  
    - Configuration: Exact string match conditions for team routing  
    - Inputs: Valid AI classification output  
    - Outputs: Four branches for each team

  - **Response Parsing (Sticky Note)**  
    - Type: Sticky Note  
    - Role: Describes intelligent routing function and team branches

#### 1.4 Database Logging & Slack Alerts

- **Overview:**  
  Each team branch creates a new record in Airtable, logs the issue in Notion, then sends a Slack notification with summarized key info.

- **Nodes Involved:**  
  For each team branch (DevOps, Backend, Support, API):

  - Airtable: Create Record (Team)  
  - Notion: Log Issue (Team)  
  - Slack Notify: Team

- **Node Details:**

  - **Airtable: Create Record (Team)**  
    - Type: Airtable Node  
    - Role: Creates a new record in a defined Airtable table with error details mapped to columns such as Error Code, Category, Severity, Suggested Action, etc.  
    - Configuration: Uses personal access token credentials; fields defined with expressions from input JSON  
    - Inputs: Routed data for the team  
    - Outputs: Created Airtable record JSON  
    - Edge Cases: API key expiry, invalid base/table IDs, rate limits, data type mismatches

  - **Notion: Log Issue (Team)**  
    - Type: Notion Node  
    - Role: Creates a page in a Notion database with mapped properties reflecting error details  
    - Configuration: Requires Notion API integration with correct database ID and property mappings  
    - Inputs: Airtable created record fields  
    - Outputs: Created Notion page JSON  
    - Edge Cases: Invalid token, missing database, property mismatches

  - **Slack Notify: Team**  
    - Type: Slack Node  
    - Role: Sends a formatted alert message to a configured Slack channel with key error info and GitHub issue link  
    - Configuration: Uses Slack OAuth2 credentials and channel IDs (set via expressions)  
    - Inputs: Notion or Airtable data, original issue URL  
    - Outputs: Slack message post status  
    - Edge Cases: Invalid webhook/channel, token expiry, Slack API rate limits

  - **Slack Alerts (Sticky Note)**  
    - Type: Sticky Note  
    - Role: Describes the purpose of Slack alerts for quick team awareness

#### 1.5 Error Handling

- **Overview:**  
  Catches any errors in workflow execution and posts detailed error alerts to a designated Slack channel for immediate attention.

- **Nodes Involved:**  
  - Error Handler Trigger  
  - Slack: Send Error Alert  
  - Sticky Note1 (Error Handling description)

- **Node Details:**

  - **Error Handler Trigger**  
    - Type: Error Trigger Node  
    - Role: Listens for any node failures or errors in the workflow  
    - Inputs: N/A (system-triggered on error)  
    - Outputs: Error details JSON

  - **Slack: Send Error Alert**  
    - Type: Slack Node  
    - Role: Sends a formatted error alert message with node name, error message, and timestamp to Slack channel  
    - Inputs: Error data from error trigger  
    - Outputs: Slack message confirmation  
    - Edge Cases: Slack API issues, token expiry

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains error handling mechanism and Slack alert content

---

### 3. Summary Table

| Node Name                          | Node Type                     | Functional Role                         | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                                   |
|-----------------------------------|-------------------------------|---------------------------------------|----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| GitHub Issue Created/Updated       | GitHub Trigger                | Listens for GitHub issue events       | N/A                              | Prepare GitHub Issue Data            | Trigger Configuration: Listens for new/updated issues labeled bug or error, prepares JSON for AI             |
| Prepare GitHub Issue Data          | Set                           | Extracts key issue fields              | GitHub Issue Created/Updated     | AI: Classify API Error (GPT-4o)     |                                                                                                              |
| AI: Classify API Error (GPT-4o)   | LangChain Agent               | AI classification of issues            | Prepare GitHub Issue Data        | Parse AI Classification Output      | AI Classification: Analyzes issue text with GPT-4o, extracts structured error info                            |
| LLM Model: GPT-4o                 | LangChain LLM OpenAI Model   | Provides GPT-4o language model         | AI: Classify API Error (GPT-4o)  | AI: Classify API Error (GPT-4o)     |                                                                                                              |
| Output Parser: Structured JSON Schema | LangChain Output Parser     | Enforces AI response to JSON schema    | AI: Classify API Error (GPT-4o)  | AI: Classify API Error (GPT-4o)     |                                                                                                              |
| AI Memory Buffer                  | LangChain Memory Buffer       | Maintains session context for AI       | AI: Classify API Error (GPT-4o)  | AI: Classify API Error (GPT-4o)     |                                                                                                              |
| Parse AI Classification Output    | Code                          | Formats AI output for Airtable/Notion/Slack | AI: Classify API Error (GPT-4o)  | Check: Valid AI Response or Data Missing |                                                                                                              |
| Check: Valid AI Response or Data Missing | If                        | Validates AI response presence         | Parse AI Classification Output  | Route by Assigned Team               |                                                                                                              |
| Route by Assigned Team             | Switch                        | Routes data to team branches           | Check: Valid AI Response or Data Missing | Airtable: Create Record (DevOps), Notion: Log Issue (Backend), Airtable: Create Record (Support), Airtable: Create Record (API Team) | Response Parsing: Routes issues to DevOps, Backend, Support, or API teams                                     |
| Airtable: Create Record (DevOps)  | Airtable                      | Creates error record for DevOps        | Route by Assigned Team           | Notion: Log Issue (DevOps)           |                                                                                                              |
| Notion: Log Issue (DevOps)        | Notion                        | Logs error in Notion for DevOps        | Airtable: Create Record (DevOps) | Slack Notify: DevOps Team            |                                                                                                              |
| Slack Notify: DevOps Team          | Slack                         | Sends Slack alert to DevOps channel    | Notion: Log Issue (DevOps)       | N/A                                | Slack Alerts: Sends team-specific alerts with error summary                                                |
| Notion: Log Issue (Backend)       | Notion                        | Logs error in Notion for Backend       | Route by Assigned Team           | Slack Notify: Backend Team           |                                                                                                              |
| Slack Notify: Backend Team         | Slack                         | Sends Slack alert to Backend channel   | Notion: Log Issue (Backend)      | N/A                                |                                                                                                              |
| Airtable: Create Record (Support) | Airtable                      | Creates error record for Support       | Route by Assigned Team           | Notion: Log Issue (Support Team)     |                                                                                                              |
| Notion: Log Issue (Support Team)  | Notion                        | Logs error in Notion for Support       | Airtable: Create Record (Support) | Slack Notify: Support Team           |                                                                                                              |
| Slack Notify: Support Team         | Slack                         | Sends Slack alert to Support channel   | Notion: Log Issue (Support Team) | N/A                                |                                                                                                              |
| Airtable: Create Record (API Team)| Airtable                      | Creates error record for API Team      | Route by Assigned Team           | Notion: Log Issue (API Team)         |                                                                                                              |
| Notion: Log Issue (API Team)      | Notion                        | Logs error in Notion for API Team      | Airtable: Create Record (API Team) | Notify API Team                     |                                                                                                              |
| Notify API Team                   | Slack                         | Sends Slack alert to API team channel  | Notion: Log Issue (API Team)     | N/A                                |                                                                                                              |
| Error Handler Trigger             | Error Trigger                 | Catches workflow errors                 | N/A                              | Slack: Send Error Alert              | Error Handling: Catches failures and posts alert to Slack                                                  |
| Slack: Send Error Alert           | Slack                         | Sends error alert to Slack channel     | Error Handler Trigger            | N/A                                |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger Node**  
   - Type: GitHub Trigger  
   - Configure OAuth2 credentials for GitHub  
   - Set to listen for "issues" events on the target repository  
   - Enable webhook with unique webhook ID  
   - Purpose: Trigger workflow on new or updated GitHub issues

2. **Add Set Node to Prepare GitHub Issue Data**  
   - Extract fields: Issue Title, Issue Body, Issue URL, Issue Number, Repository Name, Created At  
   - Use expressions to map from webhook JSON: e.g. `{{$json.body.issue.title}}`  
   - Connect GitHub Trigger output to this node’s input

3. **Insert LangChain Agent Node for AI Classification**  
   - Type: LangChain Agent  
   - Use OpenAI GPT-4o model with configured OpenAI credentials  
   - Set system message prompt with detailed instructions for error classification, JSON output schema, and team assignment rules  
   - Input: Structured issue data from Set node  
   - Connect output to output parser node

4. **Add LangChain OpenAI Chat Model Node (GPT-4o)**  
   - Configure with OpenAI credentials  
   - Set temperature to 0.7  
   - Connect to LangChain Agent node as the language model provider

5. **Add LangChain Structured Output Parser Node**  
   - Define JSON schema example matching expected AI output  
   - Connect AI Agent output to this parser node

6. **Add LangChain Memory Buffer Node**  
   - Set session key like `"AI Agent - Error Analyzer"`  
   - Connect memory buffer input/output to AI Agent node for session context

7. **Create Code Node to Parse AI Output**  
   - Write JavaScript to merge AI results with original issue data  
   - Format fields for Airtable, Notion, and Slack notifications (include formatted messages)  
   - Connect AI Agent output parser to this node

8. **Add If Node to Validate AI Response**  
   - Condition: Check if AI output has valid ID or non-empty Airtable fields  
   - Connect Code node output to this If node

9. **Add Switch Node to Route by Assigned Team**  
   - Define cases for: DevOps Team, Backend Team, Support Team, API Team  
   - Connect If node’s "true" output to this Switch node

10. **For Each Team Branch, Add Airtable Create Record Node**  
    - Configure Airtable credentials (Personal Access Token)  
    - Select base and table with fields matching error data (Error Code, Category, Severity, etc.)  
    - Map input JSON fields to Airtable columns using expressions  
    - Connect team branch output from Switch node

11. **Add Notion Create Page Node for Each Team**  
    - Configure Notion API credentials  
    - Select Notion database for issue logging  
    - Map Airtable output fields to Notion database properties (Status, Category, FAQ, Root Cause, etc.)  
    - Connect Airtable node output to Notion node input

12. **Add Slack Send Message Node for Each Team**  
    - Configure Slack OAuth2 credentials  
    - Set target Slack channel (channel ID)  
    - Compose message with error summary, FAQ, suggested actions, and GitHub issue link using expressions from Airtable or Notion data  
    - Connect Notion output to Slack node input

13. **Add Error Trigger Node**  
    - Configure to catch any workflow error events

14. **Add Slack Node for Error Alerts**  
    - Configure Slack credentials and channel for error notifications  
    - Format message to include node name, error message, and timestamp from error trigger data  
    - Connect Error Trigger node to this Slack node

15. **Add Descriptive Sticky Notes**  
    - Add notes to document trigger, AI classification, routing, Slack alerts, error handling, and credentials usage  
    - Arrange visually near related nodes

16. **Finalize Connections**  
    - Ensure proper execution order: GitHub Trigger → Prepare Data → AI Classification chain → Parse Output → Validate → Route → Airtable → Notion → Slack → End  
    - Error trigger node runs in parallel

17. **Test and Deploy**  
    - Run workflow manually to initialize GitHub webhook  
    - Test with sample GitHub issues labeled bug/error  
    - Monitor Slack for alerts and check Airtable/Notion for records

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow automatically listens for GitHub issues labeled “bug” or “error,” classifies with GPT-4o, and logs to Airtable, Notion, Slack | Sticky Note3: Setup steps and overall workflow description                                                   |
| OAuth2 used for GitHub and Slack authentication; API keys for OpenAI, Airtable, and Notion; remove personal IDs before sharing | Sticky Note2: Credentials & Security guidelines                                                               |
| Slack alerts provide team-specific notifications summarizing error code, category, FAQ match, suggested action, and GitHub link | Sticky Note (Slack Alerts): Explains purpose of Slack notifications                                             |
| Error handling node captures any workflow error and sends Slack alerts including node name, error message, and timestamp  | Sticky Note1: Error Handling explanation                                                                       |

---

This detailed reference document enables users or AI agents to fully understand, reproduce, modify, and troubleshoot the "Automatically Detect & Classify GitHub API Errors with GPT-4o to Airtable, Notion & Slack" n8n workflow without requiring the original JSON export.