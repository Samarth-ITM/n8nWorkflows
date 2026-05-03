Validate GitHub Configurations with GPT-4o-mini and Log Issues to Sheets and Slack

https://n8nworkflows.xyz/workflows/validate-github-configurations-with-gpt-4o-mini-and-log-issues-to-sheets-and-slack-10329


# Validate GitHub Configurations with GPT-4o-mini and Log Issues to Sheets and Slack

### 1. Workflow Overview

This workflow automates the validation of GitHub repository configuration files against FAQ documentation references using AI, specifically the GPT-4o-mini model. It is designed for DevOps teams and configuration managers who want to prevent configuration drift, ensure documentation accuracy, and automate compliance checks.

**Logical Blocks:**

- **1.1 Input Reception:** Listens for GitHub push or pull request events related to config files.
- **1.2 File Fetching:** Retrieves the repository configuration file and the FAQ reference configuration file from GitHub.
- **1.3 Data Parsing and Merging:** Parses fetched JSON files and merges them into a combined object for AI analysis.
- **1.4 AI Processing:** Runs AI agent to compare configurations, identify discrepancies, and classify issue severities.
- **1.5 Results Processing:** Extracts and formats AI output, prepares data for logging and alerting.
- **1.6 Logging and Notification:** Logs issues to Google Sheets and sends Slack alerts for critical or high-severity problems.
- **1.7 Documentation and Setup Guidance:** Provides user instructions and notes on workflow configuration and usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Triggers workflow on relevant GitHub repository events (push or pull request), specifically when monitored config files change.
- **Nodes Involved:**  
  - GitHub Push or PR Event
- **Node Details:**
  - **GitHub Push or PR Event**
    - Type: GitHub Trigger node
    - Role: Listens for `push` and `pull_request` events on specified repository and owner (dynamic from incoming payload).
    - Config: Uses OAuth2 authentication with GitHub; webhook automatically created by n8n.
    - Inputs: None (Webhook trigger).
    - Outputs: JSON payload containing GitHub event data, repository info.
    - Edge cases: Missing webhook setup or permissions; webhook secret validation failure; event payload format changes.
    - Notes: Requires GitHub OAuth2 credentials and webhook configured in GitHub repository settings.

#### 2.2 File Fetching

- **Overview:** Fetches two JSON config files in parallel from the repository: the actual configuration and the FAQ reference.
- **Nodes Involved:**  
  - Fetch Repository Config  
  - Fetch FAQ Reference Config
- **Node Details:**
  - **Fetch Repository Config**
    - Type: GitHub node (file get operation)
    - Role: Retrieves `config/app-config.json` from the repository’s main branch.
    - Config: Owner and repository dynamically set from trigger payload; OAuth2 authentication.
    - Inputs: Trigger event output.
    - Outputs: File content in JSON format.
    - Edge cases: File missing, permission denied, API rate limit, GitHub downtime.
  - **Fetch FAQ Reference Config**
    - Type: GitHub node (file get operation)
    - Role: Retrieves `faq-config.json` from the same repository.
    - Config: Similar to above, OAuth2 authenticated.
    - Inputs: Trigger event output.
    - Outputs: File content in JSON format.
    - Edge cases: Same as above.

#### 2.3 Data Parsing and Merging

- **Overview:** Converts the raw file contents from GitHub into parsed JSON objects and merges both configs into a single composite object for AI consumption.
- **Nodes Involved:**  
  - Parse Repo Config JSON  
  - Parse FAQ Config JSON  
  - Merge Config Files
- **Node Details:**
  - **Parse Repo Config JSON**
    - Type: Extract from File node (fromJson operation)
    - Role: Parses repository config file content into JSON object.
    - Inputs: File content from Fetch Repository Config.
    - Outputs: Parsed JSON.
    - Edge cases: Invalid JSON format, empty file.
  - **Parse FAQ Config JSON**
    - Type: Extract from File node (fromJson operation)
    - Role: Parses FAQ reference file content into JSON object.
    - Inputs: File content from Fetch FAQ Reference Config.
    - Outputs: Parsed JSON.
    - Edge cases: Same as above.
  - **Merge Config Files**
    - Type: Merge node (combine mode)
    - Role: Combines parsed repo and FAQ config JSONs into one object:  
      ```json
      {
        "repoConfig": {...},
        "faqConfig": {...}
      }
      ```
    - Inputs: Parsed repo config and FAQ config.
    - Outputs: Combined JSON object.
    - Edge cases: Missing one of the inputs; unexpected JSON structures.

#### 2.4 AI Processing

- **Overview:** Uses a Langchain AI agent with GPT-4o-mini model to compare the two configs, identify discrepancies, classify severity, and provide actionable recommendations.
- **Nodes Involved:**  
  - AI Config Comparison Agent  
  - OpenAI GPT-4o-mini  
  - JSON Output Schema  
  - Conversation Memory
- **Node Details:**
  - **AI Config Comparison Agent**
    - Type: Langchain AI Agent node
    - Role: Implements a detailed prompt to instruct AI to compare configs, find mismatches, and generate structured JSON output with issues.
    - Inputs: Merged config object.
    - Outputs: Raw AI JSON response.
    - Configuration: Includes system message with expert context and severity guidelines.
    - Edge cases: AI timeout, malformed JSON output, API key errors, incomplete prompt.
  - **OpenAI GPT-4o-mini**
    - Type: Langchain OpenAI LM Chat node
    - Role: Executes GPT-4o-mini model with temperature 0.3 and max tokens 4000.
    - Inputs: Prompt from AI Agent.
    - Outputs: Chat completion with JSON.
    - Credentials: OpenAI API key required.
    - Edge cases: Rate limits, API errors, network issues.
  - **JSON Output Schema**
    - Type: Langchain Output Parser node
    - Role: Parses AI output and validates it against predefined JSON schema for issues.
    - Inputs: Raw AI output.
    - Outputs: Structured JSON with `issues` array and metadata.
    - Edge cases: Parsing errors if AI returns invalid JSON.
  - **Conversation Memory**
    - Type: Langchain Memory Buffer node
    - Role: Maintains context window of last 3 interactions keyed by repository name for continuity.
    - Inputs/Outputs: Connects to AI Agent node.
    - Edge cases: Memory overflow or session key misconfiguration.

#### 2.5 Results Processing

- **Overview:** Processes AI output to extract individual issues, format them for logging to Google Sheets, and prepare summarized data for notifications.
- **Nodes Involved:**  
  - Format Issues for Logging
- **Node Details:**
  - **Format Issues for Logging**
    - Type: Code node (JavaScript)
    - Role: Iterates over AI output, maps issues to Google Sheets columns, handles no-issue and error cases gracefully.
    - Inputs: Structured JSON from AI Output Schema.
    - Outputs: Array of issue objects formatted for Google Sheets.
    - Key logic: Adds timestamp, generates unique issue IDs, supports error fallback message.
    - Edge cases: Unexpected AI output structure, runtime exceptions.

#### 2.6 Logging and Notification

- **Overview:** Appends the formatted issues to a Google Sheets document and sends Slack alerts when critical or high-severity issues are detected.
- **Nodes Involved:**  
  - Log to Google Sheets  
  - Send Slack Alert
- **Node Details:**
  - **Log to Google Sheets**
    - Type: Google Sheets node
    - Role: Appends rows to a configured sheet named "Config Discrepancies".
    - Columns: timestamp, configKey, faqReference, actualConfig, issueType, severity, suggestion, confidence, issueId, summary.
    - Credentials: OAuth2 for Google Sheets.
    - Inputs: Formatted issue data.
    - Outputs: Confirmation of append operation.
    - Edge cases: Sheet not found, permission denied, API quota exceeded.
  - **Send Slack Alert**
    - Type: Slack node
    - Role: Sends Slack message to configured channel summarizing up to 3 issues with severity emojis and detailed info.
    - Credentials: OAuth2 for Slack workspace.
    - Inputs: Formatted issue data + GitHub event metadata for contextual info.
    - Message includes repository, branch, commit, issue details, total count, and link to Google Sheets report.
    - Edge cases: Slack API errors, invalid channel ID, message formatting issues.

#### 2.7 Documentation and Setup Guidance

- **Overview:** Provides users with essential information on workflow purpose, setup instructions, configuration details, and operational notes.
- **Nodes Involved:**  
  - Workflow Overview (sticky note)  
  - Setup Guide (sticky note)  
  - Trigger Configuration (sticky note)  
  - File Fetching Logic (sticky note)  
  - Merge Strategy (sticky note)  
  - AI Analysis Details (sticky note)  
  - Data Processing (sticky note)  
  - Sheets Configuration (sticky note)  
  - Slack Alerts (sticky note)
- **Node Details:**  
  - Sticky notes contain detailed textual explanations about each functional area, including file paths, credential requirements, webhook setup, AI model usage, Google Sheets schema, Slack alert customization, and business value.
  - These nodes do not participate in data flow but serve as in-editor documentation for operators.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                       | Input Node(s)                           | Output Node(s)                  | Sticky Note                                                                                                          |
|----------------------------|---------------------------------|-------------------------------------|---------------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Workflow Overview          | Sticky Note                     | Documentation overview               | None                                  | None                           | Provides high-level description, use cases, and business value of the workflow.                                      |
| Setup Guide               | Sticky Note                     | Setup instructions                  | None                                  | None                           | Step-by-step setup instructions for GitHub, OpenAI, Google Sheets, Slack credentials and config paths.               |
| Trigger Configuration     | Sticky Note                     | GitHub webhook trigger explanation | None                                  | None                           | Details webhook event types, security recommendations, and trigger conditions.                                       |
| File Fetching Logic       | Sticky Note                     | Explanation of fetching config files | None                                  | None                           | Describes parallel fetching of repository and FAQ config files.                                                      |
| Merge Strategy            | Sticky Note                     | Data merging explanation            | None                                  | None                           | Explains merge of repo and FAQ configs into single JSON for AI.                                                      |
| AI Analysis Details       | Sticky Note                     | AI comparison logic explanation    | None                                  | None                           | Details AI goals, severity levels, and output format.                                                                 |
| Data Processing           | Sticky Note                     | Result extraction and formatting   | None                                  | None                           | Explains how AI output is parsed and formatted for logging and alerting.                                            |
| Sheets Configuration      | Sticky Note                     | Google Sheets schema details        | None                                  | None                           | Lists required Google Sheets columns and append operation.                                                          |
| Slack Alerts              | Sticky Note                     | Slack alert content and conditions | None                                  | None                           | Describes Slack alert content, severity filtering, and customization options.                                        |
| GitHub Push or PR Event   | GitHub Trigger                  | Input trigger for GitHub events    | None                                  | Fetch Repository Config, Fetch FAQ Reference Config |                                                                                                                      |
| Fetch Repository Config   | GitHub                         | Fetch repo config file              | GitHub Push or PR Event               | Parse Repo Config JSON          |                                                                                                                      |
| Fetch FAQ Reference Config | GitHub                         | Fetch FAQ reference config file    | GitHub Push or PR Event               | Parse FAQ Config JSON           |                                                                                                                      |
| Parse Repo Config JSON    | Extract from File               | Parse repo config JSON              | Fetch Repository Config               | Merge Config Files             |                                                                                                                      |
| Parse FAQ Config JSON     | Extract from File               | Parse FAQ config JSON               | Fetch FAQ Reference Config            | Merge Config Files             |                                                                                                                      |
| Merge Config Files        | Merge                          | Combine parsed configs              | Parse Repo Config JSON, Parse FAQ Config JSON | AI Config Comparison Agent    |                                                                                                                      |
| AI Config Comparison Agent | Langchain AI Agent             | AI comparison and issue detection  | Merge Config Files                    | Format Issues for Logging       |                                                                                                                      |
| OpenAI GPT-4o-mini        | Langchain LM Chat OpenAI       | Execute GPT-4o-mini model           | AI Config Comparison Agent            | AI Config Comparison Agent (output parser) |                                                                                                                      |
| JSON Output Schema        | Langchain Output Parser        | Validate and parse AI output        | OpenAI GPT-4o-mini                   | AI Config Comparison Agent      |                                                                                                                      |
| Conversation Memory       | Langchain Memory Buffer        | Maintain AI conversation context    | AI Config Comparison Agent            | AI Config Comparison Agent      |                                                                                                                      |
| Format Issues for Logging | Code                          | Format AI issues for Sheets logging | AI Config Comparison Agent            | Log to Google Sheets            |                                                                                                                      |
| Log to Google Sheets      | Google Sheets                  | Append issues to Sheets             | Format Issues for Logging             | Send Slack Alert               |                                                                                                                      |
| Send Slack Alert          | Slack                         | Send alerts for critical issues    | Log to Google Sheets                  | None                           |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub OAuth2 Credential in n8n:**
   - Set up OAuth2 with repo and webhook permissions.
   - Store credential as “GitHub account vivek” or similar.

2. **Add GitHub Trigger Node:**
   - Type: GitHub Trigger
   - Events: `push`, `pull_request`
   - Repository and owner set dynamically from incoming webhook payload.
   - Use GitHub OAuth2 credential.
   - Enable webhook; add webhook URL to GitHub repository settings.
   - Secure with webhook secret and validate payload signatures.

3. **Add Two GitHub Nodes to Fetch Config Files in Parallel:**
   - Node 1: Fetch Repository Config
     - Operation: Get file
     - File path: `config/app-config.json`
     - Owner and repo dynamic from GitHub trigger.
     - Use same OAuth2 credential.
   - Node 2: Fetch FAQ Reference Config
     - Operation: Get file
     - File path: `faq-config.json`
     - Owner and repo dynamic.
     - OAuth2 credential same as above.

4. **Parse Both Files from JSON:**
   - Add Extract From File node for each fetched file.
   - Operation: fromJson
   - Input: Corresponding GitHub fetch node output.

5. **Merge Parsed JSON Objects:**
   - Add Merge node
   - Mode: Combine (one-to-one)
   - Connect both parsing nodes as inputs.
   - Output will be a JSON object with two keys: `repoConfig` and `faqConfig`.

6. **Add Langchain AI Agent Node for Config Comparison:**
   - Use @n8n/n8n-nodes-langchain.agent node.
   - Provide prompt to compare repoConfig and faqConfig.
   - Configure system message with expertise and severity guidelines.
   - Connect Merge node output as input.
   - Set output parser to expect structured JSON.

7. **Add OpenAI GPT-4o-mini Node:**
   - Use Langchain LM Chat OpenAI node.
   - Model: `gpt-4o-mini`
   - Temperature: 0.3
   - Max tokens: 4000
   - Credential: your OpenAI API key.
   - Connect AI Agent node to this node’s input.

8. **Add JSON Output Schema Node:**
   - Use Langchain Output Parser Structured node.
   - Define JSON schema with fields: result, summary, issues array with id, key, repo_value, faq_value, type, severity, recommendation, confidence, metadata.
   - Connect OpenAI GPT node output here.

9. **Add Conversation Memory Node:**
   - Use Langchain Memory Buffer Window node.
   - Session Key: `"config_validation_" + repository name from GitHub trigger`
   - Context window length: 3
   - Connect memory input/output to AI Agent node.

10. **Add Code Node to Format Issues for Google Sheets:**
    - Copy provided JS code that extracts issues, handles empty results, errors, and maps fields to sheet columns.
    - Input: JSON Output Schema node output.

11. **Add Google Sheets Node to Log Issues:**
    - Operation: Append
    - Sheet name: `Config Discrepancies`
    - Map columns as per schema: timestamp, configKey, faqReference, actualConfig, issueType, severity, suggestion, confidence, issueId, summary.
    - Use OAuth2 credential for Google Sheets with access to target spreadsheet.
    - Connect Code node output here.

12. **Add Slack Node for Alerts:**
    - Use Slack node.
    - Channel: configured Slack channel ID (from setup)
    - Text: Construct message summarizing top 3 issues with emojis, details, and link to Google Sheets report.
    - OAuth2 credential for Slack workspace.
    - Connect Google Sheets node output here.

13. **Add Sticky Note Nodes for Documentation:**
    - Add sticky notes to describe workflow overview, setup steps, trigger config, file fetching, merge strategy, AI analysis details, results processing, sheet schema, and Slack alerts.
    - Place them logically near related nodes for user reference.

14. **Test Workflow:**
    - Push or create PR with changes in monitored config files.
    - Verify workflow triggers, fetches files, AI comparison outputs issues.
    - Confirm issues logged in Google Sheets.
    - Confirm Slack alerts sent for critical/high issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow uses GPT-4o-mini model for cost-effective AI analysis with up to 4000 max tokens per run.                                                                                                                                                                 | OpenAI model choice and cost considerations                                                                |
| GitHub OAuth2 credential must have permissions to read repository contents and configure webhooks.                                                                                                                                                                | GitHub setup requirement                                                                                     |
| Google Sheets must have a sheet named "Config Discrepancies" with columns matching the schema for proper logging.                                                                                                                                                 | Google Sheets schema                                                                                          |
| Slack alerts include severity-based emoji indicators and link to full Google Sheets report for easy access.                                                                                                                                                       | Slack messaging customization                                                                                 |
| Workflow prevents config drift and helps maintain compliance and documentation accuracy automatically.                                                                                                                                                            | Business value                                                                                               |
| For best security, set GitHub webhook secret and validate incoming payload signatures in the GitHub Trigger node.                                                                                                                                                 | Security best practice                                                                                        |
| AI prompt enforces strict JSON output with no markdown or extraneous text to simplify parsing downstream.                                                                                                                                                         | AI prompt design                                                                                              |
| Conversation memory buffers last 3 validation runs per repository for contextual continuity.                                                                                                                                                                       | Langchain memory usage                                                                                        |
| All nodes use OAuth2 credentials; ensure tokens and scopes are kept up to date to avoid authorization failures.                                                                                                                                                    | Credential maintenance                                                                                        |
| Slack channel ID and Google Sheets document ID are dynamically referenced from Workflow Overview sticky note JSON properties for flexibility.                                                                                                                     | Dynamic references for integration endpoints                                                                 |
| Workflow is designed to handle no-issue cases gracefully by logging success entries and avoiding false alerts.                                                                                                                                                    | Edge case handling                                                                                            |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.