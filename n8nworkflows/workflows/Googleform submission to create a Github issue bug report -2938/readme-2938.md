Googleform submission to create a Github issue bug report 

https://n8nworkflows.xyz/workflows/googleform-submission-to-create-a-github-issue-bug-report--2938


# Googleform submission to create a Github issue bug report 

### 1. Workflow Overview

This workflow automates the process of handling new Google Form submissions by creating GitHub issues for bug reports and sending notifications to a Discord channel. It is designed to streamline bug tracking and team communication by integrating Google Sheets, GitHub, OpenAI, and Discord.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Storage**  
  Captures new Google Form submissions via a Google Sheets Trigger and records them in a dedicated Google Sheet.

- **1.2 Duplicate Issue Filtering**  
  Checks if a GitHub issue has already been created for the submission to prevent duplicates.

- **1.3 AI-Powered Issue Formatting**  
  Uses OpenAI’s language model with memory and structured output parsing to generate a clear, structured bug report including title, description, and suggested fix.

- **1.4 GitHub Issue Creation**  
  Creates a new issue in the specified GitHub repository using the AI-generated content.

- **1.5 Discord Notification**  
  Sends a notification with a link to the newly created GitHub issue to a Discord channel via webhook.

- **1.6 Google Sheet Update**  
  Updates the original Google Sheet row with the GitHub issue link for cross-referencing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Storage

- **Overview:**  
  This block listens for new Google Form submissions and logs them into a Google Sheet for record-keeping and further processing.

- **Nodes Involved:**  
  - Google Form Trigger  
  - Add new Form submissions to Google Sheets

- **Node Details:**

  - **Google Form Trigger**  
    - Type: Google Sheets Trigger  
    - Role: Watches the Google Sheet linked to the form for new rows (submissions).  
    - Configuration: Set with the Google Sheet document ID and sheet name linked to the form responses. Polls every minute.  
    - Inputs: None (trigger node)  
    - Outputs: New form submission data as JSON.  
    - Edge Cases: Possible authentication errors with Google API; delays if polling interval is too long.  
    - Version: 1

  - **Add new Form submissions to Google Sheets**  
    - Type: Google Sheets node  
    - Role: Adds the new submission data into a dedicated Google Sheet tab for archiving and reference.  
    - Configuration: Uses the same Google Sheet document ID and target sheet name. Maps form fields to columns.  
    - Inputs: Output from Google Form Trigger  
    - Outputs: Confirmation of row addition.  
    - Edge Cases: API quota limits, permission errors, or sheet locking conflicts.  
    - Version: 4.5

---

#### 2.2 Duplicate Issue Filtering

- **Overview:**  
  Prevents creating duplicate GitHub issues by checking if a GitHub link already exists for the submission.

- **Nodes Involved:**  
  - If  
  - Filter out already posted issues (NoOp node)

- **Node Details:**

  - **If**  
    - Type: Conditional node  
    - Role: Checks if the current form submission row already contains a GitHub issue link.  
    - Configuration: Condition checks if the GitHub Link column is empty or not.  
    - Inputs: Output from "Add new Form submissions to Google Sheets"  
    - Outputs: Two branches — True (new issue needed), False (issue already exists).  
    - Edge Cases: Incorrect column mapping or empty values causing false negatives.  
    - Version: 2.2

  - **Filter out already posted issues**  
    - Type: No Operation (NoOp) node  
    - Role: Acts as a sink for submissions where issues already exist; effectively stops the workflow for those items.  
    - Inputs: False branch from If node  
    - Outputs: None (end of branch)  
    - Version: 1

---

#### 2.3 AI-Powered Issue Formatting

- **Overview:**  
  Uses OpenAI’s Chat Model with memory and structured output parsing to generate a detailed, formatted bug report from the form submission data.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Window Buffer Memory  
  - Structured Output Parser  
  - format message /output parsing (Agent node)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Generates AI-driven text based on prompts and input data.  
    - Configuration: Uses OpenAI API credentials; model and temperature settings configured for structured output.  
    - Inputs: Connected as AI language model input to the Agent node.  
    - Outputs: AI-generated text response.  
    - Edge Cases: API rate limits, invalid credentials, or prompt failures.  
    - Version: 1.2

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational context for the AI model to ensure coherent output.  
    - Configuration: Default window size; connected as AI memory input to Agent node.  
    - Inputs: None (internal memory)  
    - Outputs: Contextual memory data.  
    - Edge Cases: Memory overflow or context loss if window size is too small.  
    - Version: 1.3

  - **Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses AI output into structured JSON format with fields like title, description, suggested fix.  
    - Configuration: Predefined schema for parsing AI responses.  
    - Inputs: AI output from OpenAI Chat Model via Agent node.  
    - Outputs: Structured JSON object.  
    - Edge Cases: Parsing errors if AI output deviates from expected format.  
    - Version: 1.2

  - **format message /output parsing**  
    - Type: LangChain Agent node  
    - Role: Orchestrates AI model, memory, and output parser to produce final structured message.  
    - Configuration: Connected to OpenAI Chat Model (ai_languageModel), Window Buffer Memory (ai_memory), and Structured Output Parser (ai_outputParser).  
    - Inputs: True branch from If node (new issues only).  
    - Outputs: Structured AI-generated issue data.  
    - Edge Cases: Failures in any connected AI components propagate here.  
    - Version: 1.7

---

#### 2.4 GitHub Issue Creation

- **Overview:**  
  Creates a new issue in the specified GitHub repository using the structured data generated by the AI.

- **Nodes Involved:**  
  - Add issue to GitHub

- **Node Details:**

  - **Add issue to GitHub**  
    - Type: GitHub node  
    - Role: Creates a new issue in a GitHub repository.  
    - Configuration: Uses OAuth credentials for GitHub; repository owner and name specified; issue title and body mapped from AI output.  
    - Inputs: Output from AI formatting node ("format message /output parsing")  
    - Outputs: GitHub issue creation response including issue URL.  
    - Edge Cases: Authentication errors, permission issues, API rate limits, repository not found.  
    - Version: 1

---

#### 2.5 Discord Notification

- **Overview:**  
  Sends a notification message to a Discord channel with a link to the newly created GitHub issue.

- **Nodes Involved:**  
  - Send notification to discord w link

- **Node Details:**

  - **Send notification to discord w link**  
    - Type: HTTP Request node  
    - Role: Posts a message to Discord via webhook URL.  
    - Configuration: HTTP POST request to Discord webhook URL; message payload includes GitHub issue link and summary.  
    - Inputs: Output from GitHub issue creation node.  
    - Outputs: HTTP response from Discord API.  
    - Edge Cases: Invalid webhook URL, network errors, rate limits.  
    - Version: 4.2

---

#### 2.6 Google Sheet Update

- **Overview:**  
  Updates the original Google Sheet row with the GitHub issue link for traceability.

- **Nodes Involved:**  
  - Add Github link to the sheet

- **Node Details:**

  - **Add Github link to the sheet**  
    - Type: Google Sheets node  
    - Role: Updates the Google Sheet row corresponding to the form submission with the GitHub issue URL.  
    - Configuration: Uses the same Google Sheet document ID and sheet name; updates the GitHub Link column.  
    - Inputs: Output from Discord notification node.  
    - Outputs: Confirmation of update.  
    - Edge Cases: Row indexing errors, API limits, permission issues.  
    - Version: 4.5

---

### 3. Summary Table

| Node Name                         | Node Type                                | Functional Role                          | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                 |
|----------------------------------|-----------------------------------------|----------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------|
| Google Form Trigger               | Google Sheets Trigger                    | Detects new Google Form submissions    | None                             | Add new Form submissions to Google Sheets |                                                                             |
| Add new Form submissions to Google Sheets | Google Sheets                          | Logs new submissions into Google Sheet | Google Form Trigger              | If                                |                                                                             |
| If                               | If                                      | Checks if GitHub issue already exists  | Add new Form submissions to Google Sheets | format message /output parsing, Filter out already posted issues |                                                                             |
| Filter out already posted issues | No Operation (NoOp)                      | Stops workflow for existing issues     | If (False branch)                | None                              |                                                                             |
| OpenAI Chat Model                | LangChain OpenAI Chat Model              | Generates AI text for issue formatting | Connected internally to Agent node | format message /output parsing (ai_languageModel) |                                                                             |
| Window Buffer Memory             | LangChain Memory Buffer Window           | Maintains AI conversation context      | None (internal)                  | format message /output parsing (ai_memory) |                                                                             |
| Structured Output Parser         | LangChain Output Parser Structured       | Parses AI output into structured JSON  | OpenAI Chat Model (via Agent)    | format message /output parsing (ai_outputParser) |                                                                             |
| format message /output parsing   | LangChain Agent                          | Coordinates AI model, memory, and parsing | If (True branch)                 | Add issue to GitHub               |                                                                             |
| Add issue to GitHub              | GitHub                                  | Creates new GitHub issue                | format message /output parsing   | Send notification to discord w link |                                                                             |
| Send notification to discord w link | HTTP Request                           | Sends Discord notification with issue link | Add issue to GitHub              | Add Github link to the sheet      |                                                                             |
| Add Github link to the sheet     | Google Sheets                           | Updates Google Sheet with GitHub link  | Send notification to discord w link | None                             |                                                                             |
| Sticky Note                     | Sticky Note                             | N/A                                    | N/A                             | N/A                              |                                                                             |
| Sticky Note1                    | Sticky Note                             | N/A                                    | N/A                             | N/A                              |                                                                             |
| Sticky Note2                    | Sticky Note                             | N/A                                    | N/A                             | N/A                              |                                                                             |
| Sticky Note3                    | Sticky Note                             | N/A                                    | N/A                             | N/A                              |                                                                             |
| Sticky Note4                    | Sticky Note                             | N/A                                    | N/A                             | N/A                              |                                                                             |
| Sticky Note5                    | Sticky Note                             | N/A                                    | N/A                             | N/A                              |                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Form and Link to Google Sheet**  
   - Design your Google Form for bug reporting.  
   - Link responses to a Google Sheet.

2. **Add Google Sheets Trigger Node**  
   - Node Type: Google Sheets Trigger  
   - Configure with the Google Sheet document ID and sheet name linked to form responses.  
   - Set polling interval (e.g., every 1 minute).

3. **Add Google Sheets Node to Log Submissions**  
   - Node Type: Google Sheets  
   - Configure to append new rows to a dedicated sheet/tab for archiving submissions.  
   - Map form response fields accordingly.

4. **Add If Node to Filter Duplicates**  
   - Node Type: If  
   - Configure condition to check if the GitHub Link column in the current row is empty (issue not created).  
   - Connect True branch to AI formatting block; False branch to a NoOp node.

5. **Add NoOp Node for Duplicate Submissions**  
   - Node Type: No Operation  
   - Connect False branch of If node here to stop processing duplicates.

6. **Set Up OpenAI Chat Model Node**  
   - Node Type: LangChain OpenAI Chat Model  
   - Configure with OpenAI API credentials.  
   - Set model parameters (e.g., GPT-4 or GPT-3.5, temperature).  
   - No direct input; will be connected to Agent node.

7. **Add Window Buffer Memory Node**  
   - Node Type: LangChain Memory Buffer Window  
   - Use default window size or adjust as needed.  
   - Connect to Agent node as memory input.

8. **Add Structured Output Parser Node**  
   - Node Type: LangChain Output Parser Structured  
   - Define schema with fields: title, description, suggested fix.  
   - Connect to Agent node as output parser.

9. **Add Agent Node (format message /output parsing)**  
   - Node Type: LangChain Agent  
   - Connect OpenAI Chat Model (ai_languageModel), Window Buffer Memory (ai_memory), and Structured Output Parser (ai_outputParser) nodes as inputs.  
   - Connect True branch of If node as main input.  
   - Configure prompt to format form data into structured issue content.

10. **Add GitHub Node to Create Issue**  
    - Node Type: GitHub  
    - Configure OAuth credentials with repository access.  
    - Set repository owner and name.  
    - Map issue title and body from Agent node output.  
    - Connect Agent node output to this node.

11. **Add HTTP Request Node for Discord Notification**  
    - Node Type: HTTP Request  
    - Configure as POST request to Discord webhook URL.  
    - Set JSON payload to include GitHub issue link and summary.  
    - Connect GitHub node output to this node.

12. **Add Google Sheets Node to Update Sheet with GitHub Link**  
    - Node Type: Google Sheets  
    - Configure to update the original submission row with the GitHub issue URL in the GitHub Link column.  
    - Connect HTTP Request node output to this node.

13. **Connect Nodes Sequentially**  
    - Google Sheets Trigger → Add new Form submissions to Google Sheets → If →  
      - True → Agent node → GitHub → Discord HTTP Request → Google Sheets update  
      - False → NoOp node

14. **Test Workflow**  
    - Submit a test Google Form entry.  
    - Verify issue creation on GitHub, Discord notification, and Google Sheet update.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow uses OpenAI’s LangChain integration for advanced AI-driven formatting.             | n8n LangChain nodes documentation: https://docs.n8n.io/integrations/builtin/nodes/langchain/   |
| Discord webhook setup instructions can be found in Discord’s developer documentation.            | https://discord.com/developers/docs/resources/webhook                                           |
| Ensure Google API credentials have correct scopes for Sheets and Forms access.                    | Google Cloud Console API & Services setup                                                       |
| GitHub OAuth app must have repo permissions to create issues.                                    | GitHub OAuth Apps documentation: https://docs.github.com/en/developers/apps/building-oauth-apps  |
| Recommended Google Sheet columns: Timestamp, Issue Title, Issue Description, GitHub Link, Discord Notification Status | As per workflow description                                                                    |

---

This document provides a complete and detailed reference to understand, reproduce, and maintain the "Googleform submission to create a Github issue bug report" workflow in n8n.