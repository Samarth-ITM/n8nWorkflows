Create Executive Security Briefings with NixGuard AI & Wazuh Alerts

https://n8nworkflows.xyz/workflows/create-executive-security-briefings-with-nixguard-ai---wazuh-alerts-5895


# Create Executive Security Briefings with NixGuard AI & Wazuh Alerts

### 1. Workflow Overview

This workflow automates the generation and delivery of a daily executive security briefing using security alerts aggregated by NixGuard and Wazuh. It is designed for non-technical executives who need a clear, concise summary of the organization's security posture based on high-severity alerts from the last 24 hours.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow automatically every day at 8 AM.
- **1.2 Initial Security Data Retrieval:** Calls a child workflow to fetch all recent security alerts in a structured JSON format.
- **1.3 Alert Parsing and Conditional Branching:** Cleans and parses the returned data to verify if any alerts exist, then routes accordingly.
- **1.4 Executive Summary Generation:** If alerts exist, sends raw alert data to a second AI summarization stage (via the same child workflow) to generate a business-friendly briefing.
- **1.5 Formatting and Email Delivery:** Converts the AI-generated Markdown summary to HTML and sends it as an email to designated recipients.
- **1.6 Setup and Documentation:** Sticky notes provide setup instructions, workflow overview, and support links.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

**Overview:**  
Triggers the workflow automatically every day at 8 AM to ensure timely generation of the security briefing.

**Nodes Involved:**  
- Run Daily at 8 AM

**Node Details:**

- **Run Daily at 8 AM**  
  - Type: Schedule Trigger  
  - Configuration: Set to trigger hourly (interval field set to hours, but position suggests daily at 8 AM; likely requires refinement to daily 8 AM in actual use).  
  - Input: None (trigger node)  
  - Output: Triggers the next node to start data retrieval  
  - Edge Cases: If the schedule is misconfigured (e.g., interval too frequent), could cause excessive API calls or emails.  
  - Version: 1.1  

---

#### 1.2 Initial Security Data Retrieval

**Overview:**  
Prepares the API key and initial prompt for the AI to fetch recent security alerts and calls a child workflow that interfaces with NixGuard and Wazuh to retrieve raw security event data.

**Nodes Involved:**  
- Set API Key & Initial Prompt  
- Execute: Get Daily Events as JSON (child workflow)

**Node Details:**

- **Set API Key & Initial Prompt**  
  - Type: Set  
  - Configuration: Sets two string variables:  
    - `apiKey`: Placeholder for user’s NixGuard API key (must be replaced with valid key).  
    - `chatInput`: A prompt instructing the AI to review last 24 hours security data and return a minified JSON array of significant alerts or an empty array if none found.  
  - Input: Trigger node output  
  - Output: Passes variables to child workflow node  
  - Edge Cases: Missing or invalid API key will cause downstream failures.  
  - Version: 2  

- **Execute: Get Daily Events as JSON (Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration)**  
  - Type: Execute Workflow  
  - Configuration: Calls child workflow by ID `I0nUORqYTwDFZa51` which handles the actual API requests and data aggregation from NixGuard and Wazuh. Inputs mapped from prior node.  
  - Input: Receives API key and prompt  
  - Output: Returns raw AI output as a string (usually JSON wrapped in Markdown)  
  - Edge Cases: Child workflow failure, incorrect workflow ID, API communication errors, or invalid response format.  
  - Version: 1.2  
  - Sub-workflow Reference: `Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration`

---

#### 1.3 Alert Parsing and Conditional Branching

**Overview:**  
Extracts and parses JSON data from AI output, which often includes JSON wrapped in Markdown code fences. Determines if there are any alerts and routes workflow accordingly.

**Nodes Involved:**  
- Parse Alert Array (Code Node)  
- If (Conditional Node)

**Node Details:**

- **Parse Alert Array**  
  - Type: Code  
  - Configuration:  
    - Extracts JSON string from Markdown code fences using regex.  
    - Attempts JSON.parse on cleaned string.  
    - Returns alerts array in JSON if valid and non-empty; otherwise returns empty to trigger the false branch.  
  - Input: Raw AI string output from child workflow  
  - Output: Structured alerts array or empty (which affects conditional routing)  
  - Edge Cases: Parsing errors if AI returns invalid JSON, empty arrays, or unexpected formatting. Logs errors to console.  
  - Version: 2  

- **If**  
  - Type: If (Conditional)  
  - Configuration: Checks if `alerts` array exists and is non-empty.  
  - Input: Parsed alerts from previous node  
  - Output: True branch if alerts exist; false branch if no alerts or parsing failed  
  - Edge Cases: False negatives if parsing logic fails or alerts are empty.  
  - Version: 2.2  

---

#### 1.4 Executive Summary Generation

**Overview:**  
If alerts were found, prompts the AI to generate a high-level executive summary from the raw alerts array. Then calls the same child workflow again for summarization.

**Nodes Involved:**  
- Set Prompt for Summary  
- Execute: Generate Executive Summary (child workflow)  
- Set Final Briefing

**Node Details:**

- **Set Prompt for Summary**  
  - Type: Set  
  - Configuration:  
    - Sets a detailed prompt instructing the AI to act as a senior security analyst and produce:  
      1. One sentence summary of overall risk.  
      2. Number of critical alerts.  
      3. 3-4 bullet points in Markdown focusing on business impact.  
      4. A clear recommendation.  
    - Embeds raw alert data as JSON string in the prompt.  
    - `apiKey` is also set but empty (likely inherited or set elsewhere).  
  - Input: Alerts array from If node true branch  
  - Output: Structured prompt for child workflow summarization  
  - Edge Cases: Large alert arrays could cause prompt length issues; malformed alert data affects summary quality.  
  - Version: 2  

- **Execute: Generate Executive Summary (Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration)**  
  - Type: Execute Workflow  
  - Configuration: Calls the same child workflow ID `I0nUORqYTwDFZa51` to generate a textual executive summary from the prompt.  
  - Input: Prompt from previous node  
  - Output: AI-generated summary text in Markdown  
  - Edge Cases: Same as prior child workflow; API failures, workflow ID mismatch, or unexpected AI output format.  
  - Version: 1.2  
  - Sub-workflow Reference: `Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration`

- **Set Final Briefing**  
  - Type: Set  
  - Configuration: Stores the AI output summary text into `executive_summary` field for downstream processing.  
  - Input: AI summary text from child workflow  
  - Output: Prepares data for Markdown to HTML conversion  
  - Edge Cases: Empty or malformed summary text.  
  - Version: 2  

---

#### 1.5 Formatting and Email Delivery

**Overview:**  
Converts the AI-generated Markdown summary into simple HTML, then sends it via email to recipients.

**Nodes Involved:**  
- Convert Markdown to HTML (Code Node)  
- Send Email

**Node Details:**

- **Convert Markdown to HTML**  
  - Type: Code  
  - Configuration:  
    - Implements a lightweight custom Markdown to HTML converter supporting headings, bold, italics, unordered lists, and paragraphs.  
    - Escapes HTML to prevent injection.  
    - Converts Markdown bullet points to `<ul><li>` structures.  
    - Converts line breaks and basic inline formatting.  
    - Attaches resulting HTML as `html_summary` field.  
  - Input: Markdown summary from `executive_summary` field  
  - Output: HTML version of the summary for email body  
  - Edge Cases: Complex Markdown formatting will not render correctly; use of a dedicated Markdown library is recommended for robustness.  
  - Version: 2  

- **Send Email**  
  - Type: Email Send  
  - Configuration:  
    - Subject: "Daily AI Cyber Security Briefing"  
    - Body: Uses HTML from previous node (`html_summary`)  
    - Requires valid email credentials set in n8n and recipient email addresses configured in the node (not shown here).  
  - Input: HTML summary  
  - Output: Sends email; ends workflow  
  - Edge Cases: Email delivery failures due to incorrect credentials, network issues, or invalid recipient addresses.  
  - Version: 2.1  

---

#### 1.6 Setup and Documentation (Sticky Notes)

**Overview:**  
Provides user-facing guidance on setup, workflow purpose, and support resources.

**Nodes Involved:**  
- Workflow Overview1  
- Setup Guide1  
- Setup Guide

**Node Details:**

- **Workflow Overview1**  
  - Type: Sticky Note  
  - Content: Summarizes the workflow’s purpose and the two-stage AI process (data retrieval and summarization).  
  - Position: Central location for quick reference.  

- **Setup Guide1**  
  - Type: Sticky Note  
  - Content: Four-step setup instructions including importing the child workflow, setting API key, verifying workflow ID, and configuring email credentials.  
  - Position: Near initial setup nodes.  

- **Setup Guide**  
  - Type: Sticky Note  
  - Content: Prerequisites and setup instructions including API key setup, trigger configuration, agent installation, and support links:  
    - NixGuard Documentation: https://nixguard.thenex.world  
    - Community Discord: https://discord.com/invite/ajCYwYCwHb  
  - Position: Near trigger node for initial user guidance.  

---

### 3. Summary Table

| Node Name                                                                | Node Type                 | Functional Role                          | Input Node(s)                                        | Output Node(s)                                                       | Sticky Note                                                                                                                   |
|--------------------------------------------------------------------------|---------------------------|----------------------------------------|-----------------------------------------------------|----------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Run Daily at 8 AM                                                        | Schedule Trigger          | Initiate workflow daily at 8 AM        | None                                                | Set API Key & Initial Prompt                                         | See "Setup Guide" and "Setup Guide1" for setup instructions                                                                  |
| Set API Key & Initial Prompt                                             | Set                       | Prepare API key and initial AI prompt  | Run Daily at 8 AM                                   | Execute: Get Daily Events as JSON (child workflow)                   | See "Setup Guide1" for API key and workflow ID configuration                                                                 |
| Execute: Get Daily Events as JSON (Get Real-Time Security Insights...)   | Execute Workflow          | Retrieve raw security alerts            | Set API Key & Initial Prompt                         | Parse Alert Array                                                    | Relies on child workflow ID `I0nUORqYTwDFZa51`; ensure child workflow is activated                                            |
| Parse Alert Array                                                        | Code                      | Extract and parse JSON from AI output  | Execute: Get Daily Events as JSON                    | If                                                                  | Error logged if parsing fails; routes to If false branch if no alerts                                                        |
| If                                                                       | If                        | Check if alerts exist                   | Parse Alert Array                                   | Set Prompt for Summary (true), ends (false branch)                  |                                                                                                                              |
| Set Prompt for Summary                                                   | Set                       | Prepare prompt for executive summary   | If                                                  | Execute: Generate Executive Summary (child workflow)                |                                                                                                                              |
| Execute: Generate Executive Summary (Get Real-Time Security Insights...)| Execute Workflow          | Generate executive summary from alerts | Set Prompt for Summary                              | Set Final Briefing                                                  | Uses same child workflow as data retrieval; requires valid API key                                                           |
| Set Final Briefing                                                      | Set                       | Store AI summary text for formatting   | Execute: Generate Executive Summary                  | Convert Markdown to HTML                                            |                                                                                                                              |
| Convert Markdown to HTML                                                 | Code                      | Convert markdown summary to HTML       | Set Final Briefing                                  | Send Email                                                        | Lightweight converter; not suitable for complex Markdown                                                                     |
| Send Email                                                              | Email Send                | Deliver final briefing via email       | Convert Markdown to HTML                             | None                                                             | Requires configured email credentials and recipient addresses                                                                |
| Workflow Overview1                                                      | Sticky Note               | Workflow overview and process summary  | None                                                | None                                                             | Summarizes workflow purpose and stages                                                                                        |
| Setup Guide1                                                           | Sticky Note               | Step-by-step setup instructions        | None                                                | None                                                             | Detailed 4-step guide for API key, child workflow, and email setup                                                           |
| Setup Guide                                                           | Sticky Note               | Prerequisites and support info          | None                                                | None                                                             | Includes NixGuard docs and Discord community links                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**

   - Add a **Schedule Trigger** node named `Run Daily at 8 AM`.
   - Set trigger interval to daily at 8:00 AM (adjust hours field accordingly).

2. **Prepare Initial API Request:**

   - Add a **Set** node named `Set API Key & Initial Prompt`.
   - Add two string fields:  
     - `apiKey`: Enter your valid NixGuard API key here.  
     - `chatInput`: Enter the prompt text:  
       ```
       Review all security data from the last 24 hours. List all significant security alerts found. Your response MUST be a single, valid, minified JSON array of objects. Each object in the array should represent a distinct alert. If no significant alerts are found, return an empty array [].
       ```
   - Connect `Run Daily at 8 AM` output to this node.

3. **Execute Child Workflow for Data Retrieval:**

   - Add an **Execute Workflow** node named `Execute: Get Daily Events as JSON (Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration)`.
   - Select or enter the workflow ID of the child workflow that fetches real-time security insights (default ID: `I0nUORqYTwDFZa51`).
   - Map inputs to pass `apiKey` and `chatInput` from previous node.
   - Connect `Set API Key & Initial Prompt` to this node.

4. **Parse AI Output:**

   - Add a **Code** node named `Parse Alert Array`.
   - Paste the provided JavaScript code to extract JSON from Markdown code fences and parse it into an alerts array.
   - Connect output of `Execute: Get Daily Events as JSON ...` to this node.

5. **Conditional Check for Alerts:**

   - Add an **If** node named `If`.
   - Configure condition to check if `alerts` array exists and is non-empty (`{{$json.alerts}}` exists).
   - Connect `Parse Alert Array` to this node.

6. **Prepare Prompt for Executive Summary (True Branch):**

   - Add a **Set** node named `Set Prompt for Summary`.
   - Set string fields:  
     - `chatInput` with detailed prompt instructing AI to summarize alerts for executives, embedding `{{ JSON.stringify($json.alerts) }}` into the prompt.  
     - `apiKey`: (leave empty or set as needed).  
   - Connect the true output of `If` node to this node.

7. **Execute Child Workflow for Summary Generation:**

   - Add an **Execute Workflow** node named `Execute: Generate Executive Summary (Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration)`.
   - Use same child workflow ID as before (`I0nUORqYTwDFZa51`).
   - Map inputs from `Set Prompt for Summary`.
   - Connect `Set Prompt for Summary` to this node.

8. **Store AI Summary Output:**

   - Add a **Set** node named `Set Final Briefing`.
   - Set a string field `executive_summary` to the AI output `{{$json.output}}`.
   - Connect output of summary execution node to this node.

9. **Convert Markdown Summary to HTML:**

   - Add a **Code** node named `Convert Markdown to HTML`.
   - Paste the provided JavaScript code implementing a simple Markdown to HTML converter.
   - Connect `Set Final Briefing` to this node.

10. **Send Email with Summary:**

    - Add an **Email Send** node named `Send Email`.
    - Set the subject to “Daily AI Cyber Security Briefing”.
    - Use the HTML field from the previous node (`{{$json.html_summary}}`) as the email body.
    - Configure email credentials (SMTP or other) in n8n settings.
    - Set recipient email addresses.
    - Connect `Convert Markdown to HTML` to this node.

11. **Add Sticky Notes for Documentation (Optional):**

    - Add sticky note nodes containing the workflow overview, setup guide, API key instructions, and support links as per original workflow for user reference.

12. **Validate Workflow:**

    - Verify all node connections reflect the above steps.
    - Ensure the child workflow with the given ID is imported, activated, and accessible.
    - Replace all placeholders (API key, email recipients) with actual values.
    - Test the workflow manually and schedule.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| NixGuard Documentation and Community Support: Visit https://nixguard.thenex.world and join the Discord at https://discord.com/invite/ajCYwYCwHb for help and updates. | Setup Guide sticky note and general user support                                                    |
| The child workflow `Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration` must be imported and activated for this workflow to function correctly. | Setup Guide1 sticky note                                                                             |
| The Markdown to HTML converter included is simplified and does not support complex Markdown features. For robust needs, use a dedicated library like 'marked'. | Convert Markdown to HTML node comments                                                              |
| Email credentials must be configured securely in n8n. Ensure proper SMTP or email service integration with valid recipient addresses. | Send Email node                                                                                      |
| Schedule Trigger node interval should be adjusted to daily at 8 AM as intended; current config may require fine-tuning.         | Run Daily at 8 AM node                                                                                |

---

This completes the detailed, structured documentation of the workflow "Create Executive Security Briefings with NixGuard AI & Wazuh Alerts." It is designed to facilitate understanding, reproduction, and troubleshooting by advanced users and automation agents alike.