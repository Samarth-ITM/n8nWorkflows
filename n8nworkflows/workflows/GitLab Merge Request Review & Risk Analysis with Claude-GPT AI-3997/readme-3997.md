GitLab Merge Request Review & Risk Analysis with Claude/GPT AI

https://n8nworkflows.xyz/workflows/gitlab-merge-request-review---risk-analysis-with-claude-gpt-ai-3997


# GitLab Merge Request Review & Risk Analysis with Claude/GPT AI

---

## 1. Workflow Overview

This workflow automates the review and risk analysis of GitLab Merge Requests (MRs) by leveraging AI models Claude (Anthropic) or GPT-4o (OpenAI). It targets software development teams aiming to integrate automated code quality, security, and compliance assessments directly into their GitLab CI/CD pipelines. The workflow executes upon MR creation or update, fetches the code diff, analyzes risks and issues with AI, generates a detailed report, and notifies relevant stakeholders via email and GitLab comments.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Data Collection:** Watches GitLab for MR events, merges incoming webhook data.
- **1.2 Code Diff Extraction and Validation:** Retrieves the MR code diff and verifies if changes exist.
- **1.3 AI-Powered Risk Assessment:** Sends the diff to Claude/GPT AI to analyze risks, issues, recommendations, and test cases.
- **1.4 Output Processing and Structuring:** Cleans and parses AI output into a structured JSON format.
- **1.5 Notification Preparation:** Generates a distribution list dynamically based on project and author info.
- **1.6 Deliver Notifications:** Sends an HTML email to developers and QA, and posts a formatted comment back on the GitLab MR.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger and Data Collection

**Overview:**  
This block detects when a GitLab Merge Request is created or updated and consolidates the incoming webhook payload for processing.

**Nodes Involved:**  
- GitLab Trigger  
- Merge

**Node Details:**

- **GitLab Trigger**  
  - *Type:* gitlabTrigger  
  - *Role:* Listens for GitLab events, specifically "merge_requests" on the configured repository.  
  - *Configuration:*  
    - Owner and repository set to track the target project.  
    - Event type: `merge_requests`  
    - Uses stored GitLab API credentials for authentication.  
  - *Input/Output:* Trigger node, outputs webhook payload to Merge.  
  - *Edge Cases:* Webhook misconfiguration, GitLab API credential expiry or invalid permissions.  
  - *Sticky Note:* Reminds to add GitLab credentials and select the merge_requests event.

- **Merge**  
  - *Type:* merge  
  - *Role:* Consolidates incoming data streams (if multiple GitLab triggers are used) into a single execution path.  
  - *Configuration:* Default, executes once per trigger.  
  - *Input:* Receives webhook data from GitLab Trigger(s).  
  - *Output:* Outputs merged JSON to "Extract Diff" node.  
  - *Edge Cases:* No input data or multiple triggers firing simultaneously may cause unintended merges.

---

### 2.2 Code Diff Extraction and Validation

**Overview:**  
Fetches the code changes (diff) of the MR from GitLab API and checks if there are any changes to analyze.

**Nodes Involved:**  
- Extract Diff  
- If Some Change

**Node Details:**

- **Extract Diff**  
  - *Type:* HTTP Request  
  - *Role:* Calls GitLab API to retrieve the list of changes (diffs) for the specific MR.  
  - *Configuration:*  
    - URL dynamically constructed using project namespace and MR IID from webhook data.  
    - Authorization header with GitLab personal access token (PAT).  
    - Sends HTTP GET request with JSON headers.  
  - *Input:* Receives merged data from "Merge".  
  - *Output:* Returns diff data, including changes array and web_url.  
  - *Edge Cases:*  
    - Invalid or expired GitLab token leads to 401 Unauthorized.  
    - API rate limits or network timeout.  
    - MR may have no changes or be empty.  
  - *Sticky Note:* Advises to replace Authorization token with your GitLab API key.

- **If Some Change**  
  - *Type:* If  
  - *Role:* Checks if the "changes" array from "Extract Diff" contains any items (length > 0).  
  - *Configuration:*  
    - Condition: Array length of `changes` field > 0.  
  - *Input:* Receives diff data from "Extract Diff".  
  - *Output:* If true, proceeds to AI Agent for analysis; if false, stops workflow.  
  - *Edge Cases:* False negatives if API response malformed or empty changes array.  
  - *Sticky Note:* Explains it ensures MR contains code changes before continuing.

---

### 2.3 AI-Powered Risk Assessment

**Overview:**  
Submits the code diff to an AI agent (Claude or GPT) for comprehensive risk analysis, including identification of issues, recommendations, risk levels, and test cases.

**Nodes Involved:**  
- AI Agent

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent (AI language model agent)  
  - *Role:* Sends a prompt with the MR diff to Anthropic Claude-3.5-haiku model for analysis, requesting structured JSON output with predefined schema.  
  - *Configuration:*  
    - Model: Claude 3.5 Haiku (Anthropic).  
    - Max tokens: 1000, temperature: 0.7 for balanced creativity.  
    - Tools: Defines a single tool "record_summary" with strict JSON schema describing expected output fields (RiskLevel, Summary, Recommendations, Issues, URL, DiffTable, TestCases).  
    - Prompt includes detailed instructions and the MR diff text extracted from the changes array.  
  - *Input:* Receives changes array from "If Some Change".  
  - *Output:* Produces AI-generated structured summary with risk assessment.  
  - *Edge Cases:*  
    - API key missing or expired.  
    - AI model timeout or rate limiting.  
    - Malformed or too large diff input causing model truncation or errors.  
    - Incomplete or invalid JSON output from AI.  
  - *Sticky Note:* Requires Anthropic API key setup; explains role of node.

---

### 2.4 Output Processing and Structuring

**Overview:**  
Refines and parses the AI-generated output to ensure it adheres strictly to the defined JSON schema, enabling downstream nodes to consume structured data.

**Nodes Involved:**  
- Auto-fixing Output Parser  
- Structured Output Parser

**Node Details:**

- **Auto-fixing Output Parser**  
  - *Type:* LangChain Output Parser (Autofixing)  
  - *Role:* Automatically detects and corrects minor JSON formatting errors in AI output to ensure valid JSON.  
  - *Configuration:* Default.  
  - *Input:* Receives raw AI output from "Anthropic Chat Model".  
  - *Output:* Corrected JSON output to "AI Agent".  
  - *Edge Cases:* Cannot fix major structural errors or missing fields.  
  - *Sticky Note:* Notes cleaning and refinement of AI output; no special setup.

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser (Structured)  
  - *Role:* Validates AI output against the manual JSON schema defining all required fields and types.  
  - *Configuration:* Schema includes RiskLevel, Summary, Recommendations, Issues, URL, DiffTable, and TestCases with detailed nested properties.  
  - *Input:* From "AI Agent".  
  - *Output:* Provides parsed and validated structured data to downstream nodes.  
  - *Edge Cases:* Validation failure if AI output is incomplete or schema non-compliant.  
  - *Sticky Note:* Also cleans and refines AI output for structured reporting.

- **Anthropic Chat Model1 (Behind scenes):**  
  - Supports AI Agent with Anthropic API calls.  
  - Requires Anthropic API credentials configured.

---

### 2.5 Notification Preparation

**Overview:**  
Generates a dynamic email distribution list based on the project involved and the commit author, combining project leads and global admins.

**Nodes Involved:**  
- Distribution List Generator

**Node Details:**

- **Distribution List Generator**  
  - *Type:* Code (JavaScript)  
  - *Role:*  
    - Maps project names from the MR data to predefined developer and QA email lists.  
    - Appends a global admin list.  
    - Extracts sender email from the last commit, replacing default noreply addresses with actual emails.  
    - Removes duplicates and outputs a comma-separated string of emails.  
  - *Configuration:*  
    - Hardcoded email mappings for 10 sample projects and global admins.  
    - Uses n8n expressions to access previous node data for project and commit info.  
  - *Input:* Receives AI Agent output and MR data.  
  - *Output:* Email distribution string to be used by Gmail node.  
  - *Edge Cases:*  
    - Unknown project names fallback to global list only.  
    - If commit author email missing or malformed, may cause errors.  
  - *Sticky Note:* Advises updating email mappings per your team.

---

### 2.6 Deliver Notifications

**Overview:**  
Sends the AI-generated review report to the distribution list via email and posts the same report as a comment on the GitLab MR.

**Nodes Involved:**  
- Send to DL (Email Notification)  
- Comment Back on MR

**Node Details:**

- **Send to DL (Email Notification)**  
  - *Type:* Gmail (OAuth2)  
  - *Role:* Sends a richly formatted HTML email with the AI report to the generated distribution list.  
  - *Configuration:*  
    - Email recipients: Expression pulling from Distribution List Generator output.  
    - Subject line includes risk level, project name, user name, and MR creation date.  
    - HTML body includes tables for Summary, Risk Level, Recommendations, Test Cases, Issues, Diff Table, and MR URL.  
    - Uses Gmail OAuth2 credentials.  
  - *Input:* Receives email list and AI Agent output.  
  - *Output:* Email sent confirmation.  
  - *Edge Cases:*  
    - OAuth token expiry or invalid credentials.  
    - Email quota limits.  
    - Large email body size causing send failures.  
  - *Sticky Note:* Reminds to add Gmail credentials.

- **Comment Back on MR**  
  - *Type:* HTTP Request  
  - *Role:* Posts a markdown comment on the GitLab MR containing the AI-generated review report.  
  - *Configuration:*  
    - URL built dynamically using project namespace and MR IID.  
    - Authorization header with GitLab Personal Access Token (PAT).  
    - POST body contains markdown-formatted summary, risk level, recommendations, test cases, issues, and diff table.  
  - *Input:* Receives AI Agent output and MR data.  
  - *Output:* HTTP response from GitLab API.  
  - *Edge Cases:*  
    - Invalid or expired PAT.  
    - API rate limits or network errors.  
  - *Sticky Note:* Advises replacing Authorization token with your GitLab API key.

---

## 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                        | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                  |
|-----------------------------|---------------------------------|-------------------------------------|----------------------------------|-----------------------------------|--------------------------------------------------------------|
| GitLab Trigger              | gitlabTrigger                   | Trigger on GitLab MR events          | None                             | Merge                             | - Triggers workflow when a merge request (MR) is created or updated.  <br>- Add GitLab credentials. Select merge_requests event. |
| Merge                      | merge                           | Merges incoming webhook data         | GitLab Trigger                   | Extract Diff                     |                                                              |
| Extract Diff               | HTTP Request                   | Fetches MR diff from GitLab API      | Merge                           | If Some Change                   | - Fetches code changes (diffs) from GitLab using API.  <br>- Replace Authorization token with your GitLab API key.              |
| If Some Change             | if                             | Checks if MR has code changes        | Extract Diff                    | AI Agent                        | - Ensures that MR contains changes before proceeding. <br>- No setup required.                                                   |
| AI Agent                   | LangChain Agent                | Sends diff to Claude AI for analysis | If Some Change                  | Distribution List Generator, Comment Back on MR | - Calls Claude AI to analyze the diff and generate: Risk Level, Issues, Recommendations, Test Cases. <br>- Add Anthropic API Key (Claude AI). |
| Distribution List Generator | Code                           | Generates email distribution list    | AI Agent                       | Send to DL ( Email Notification) | - Creates a list of developers & QA testers for email notifications. <br>- Update email mappings for your team.                   |
| Send to DL ( Email Notification) | Gmail                         | Sends email with AI report           | Distribution List Generator      | None                            | - Sends an HTML-formatted MR Report to developers & QA teams. <br>- Add Gmail credentials.                                        |
| Comment Back on MR         | HTTP Request                   | Posts AI report as GitLab MR comment | AI Agent                       | None                            | - Posts AI-generated review report as a GitLab MR comment. <br>- Replace Authorization token with your GitLab API key.            |
| Auto-fixing Output Parser  | LangChain Output Parser (Auto) | Cleans and fixes AI output JSON      | Anthropic Chat Model             | AI Agent                        | - Cleans and refines AI output for structured reporting. <br>- No setup required.                                                 |
| Structured Output Parser   | LangChain Output Parser        | Validates AI output against schema   | AI Agent                       | Auto-fixing Output Parser       | - Cleans and refines AI output for structured reporting. <br>- No setup required.                                                 |
| Anthropic Chat Model       | LangChain LM Chat Model (AI)   | Calls Anthropic API with prompt      | Structured Output Parser         | Auto-fixing Output Parser       | - Requires Anthropic API key credential.                                                                                       |
| Anthropic Chat Model1      | LangChain LM Chat Model (AI)   | Supports AI Agent calls              | None                           | AI Agent                        | - Requires Anthropic API key credential.                                                                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create GitLab Trigger Node**  
   - Type: `gitlabTrigger`  
   - Configure credentials with your GitLab API key.  
   - Set event to `merge_requests`.  
   - Set repository owner and name accordingly.  
   - Position at workflow start.  

2. **Add Merge Node**  
   - Type: `merge`  
   - Connect GitLab Trigger output to Merge input.  
   - Configure to execute once per trigger.

3. **Add HTTP Request Node "Extract Diff"**  
   - Configure URL:  
     ```
     https://gitlab.com/api/v4/projects/{{ encodeURIComponent($json.body.project.path_with_namespace) }}/merge_requests/{{ $json.body.object_attributes.iid }}/changes
     ```  
   - Method: GET  
   - Headers:  
     ```
     Authorization: Bearer <Your_GitLab_PAT>
     ```  
   - Ensure JSON response is parsed.  
   - Connect Merge node output to this node.

4. **Add If Node "If Some Change"**  
   - Condition: Check if array length of `changes` field is greater than zero:  
     Expression: `={{ $json.changes.length > 0 }}`  
   - Connect Extract Diff output to If node.

5. **Add LangChain Agent Node "AI Agent"**  
   - Model: `claude-3-5-haiku-20241022` (Anthropic)  
   - Max tokens: 1000  
   - Temperature: 0.7  
   - Define a tool named "record_summary" with JSON schema including fields: RiskLevel, Summary, Recommendations, Issues, URL, DiffTable, TestCases.  
   - Construct prompt with instructions and include the MR diff text extracted from `changes`.  
   - Connect If node "true" output to AI Agent.

6. **Add LangChain Output Parser Nodes**  
   - Add “Structured Output Parser”:  
     - Paste the manual JSON schema for output validation as per AI Agent tool.  
     - Connect AI Agent output to Structured Output Parser input.  
   - Add “Auto-fixing Output Parser”:  
     - Connect Structured Output Parser output to Auto-fixing Output Parser input.  
   - Connect Auto-fixing Output Parser output back to AI Agent input (for final structured output).

7. **Add Code Node "Distribution List Generator"**  
   - Paste JavaScript code mapping project names to email lists, including global admins.  
   - Extract sender email from last commit, replace noreply with actual email.  
   - Remove duplicates and join emails as string.  
   - Connect AI Agent output to this node.

8. **Add Gmail Node "Send to DL (Email Notification)"**  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient emails to `={{ $json.emails }}` from Distribution List Generator.  
   - Compose HTML email body with tables for summary, risk level, recommendations, test cases, issues, diff table, and URL using expressions from AI Agent output.  
   - Set subject line incorporating risk level, project name, user name, and MR creation date.  
   - Connect Distribution List Generator output to this node.

9. **Add HTTP Request Node "Comment Back on MR"**  
   - Method: POST  
   - URL:  
     ```
     https://gitlab.com/api/v4/projects/{{ encodeURIComponent($json.body.project.path_with_namespace) }}/merge_requests/{{ $json.body.object_attributes.iid }}/notes
     ```  
   - Headers:  
     ```
     Authorization: Bearer <Your_GitLab_PAT>
     ```  
   - Body: JSON with markdown-formatted AI report summary, risk level, recommendations, test cases, issues, diff table.  
   - Connect AI Agent output to this node.

10. **Connect AI Agent output to both Distribution List Generator and Comment Back on MR nodes.**

11. **Test Workflow**  
    - Create or update a test MR in GitLab for monitored repo.  
    - Confirm workflow triggers, AI analysis completes, emails are sent, and MR comment is posted.

---

## 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                            |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------|
| Developed by Quantana, an AI-powered automation and software development company.                              | Branding                                  |
| Workflow heavily relies on Anthropic Claude AI; ensure API keys are active and have sufficient quotas.        | AI Service                                |
| GitLab Personal Access Token must have `api` scope to retrieve MR diffs and post comments.                     | GitLab API docs: https://docs.gitlab.com/ee/api/ |
| Gmail OAuth2 credentials setup is required for sending notification emails securely.                           | Gmail OAuth2 setup guide                   |
| Email distribution list is customizable via the "Distribution List Generator" node’s JavaScript code.          | Update emails per your team structure     |
| HTML formatting used in emails and MR comments ensures readability and structured presentation of reports.    | Email and GitLab MR comment UI            |
| Video overview and installation guide available at Quantana’s website: https://quantana.com.au/ai             | Additional resources                       |

---

This comprehensive documentation enables developers and automation professionals to understand, reproduce, and maintain the GitLab MR Auto-Review & Risk Assessment workflow, ensuring seamless integration and robust AI-driven code quality control.