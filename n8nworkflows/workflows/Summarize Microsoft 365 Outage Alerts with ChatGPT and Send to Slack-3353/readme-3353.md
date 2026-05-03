Summarize Microsoft 365 Outage Alerts with ChatGPT and Send to Slack

https://n8nworkflows.xyz/workflows/summarize-microsoft-365-outage-alerts-with-chatgpt-and-send-to-slack-3353


# Summarize Microsoft 365 Outage Alerts with ChatGPT and Send to Slack

### 1. Workflow Overview

This workflow is designed to automate the monitoring and summarization of Microsoft 365 (M365) outage alert emails, and send concise, actionable notifications to a dedicated Slack channel. It targets IT administrators and small Managed Service Providers (MSPs) who want to streamline M365 alert management using ChatOps principles rather than traditional email handling.

**Use Cases:**
- Consolidate M365 service health alerts from one or multiple mailboxes.
- Provide clear, summarized incident updates to Slack channels for rapid awareness.
- Automate email cleanup by deleting processed alert emails.
- Incorporate geographic user impact assessment based on configured regions.

**Logical Blocks:**

- **1.1 Trigger & Input Reception:** Poll Outlook mailbox every minute for new M365 outage alert emails.
- **1.2 Incident Data Extraction:** Parse the alert email’s HTML content to extract relevant text and incident link.
- **1.3 AI-Powered Summarization:** Use OpenAI GPT-4O-MINI model to generate a concise Markdown summary tailored for Slack, including user impact by region.
- **1.4 Slack Notification Construction:** Format the AI summary into a Slack Block Kit message with interactive button linking to the incident.
- **1.5 Slack Delivery & Email Cleanup:** Post the message to Slack, then delete the original alert email upon successful posting.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

- **Overview:**  
  This block periodically checks the configured Outlook mailbox for new M365 service health alert emails sent from Microsoft's official alert sender address.

- **Nodes Involved:**  
  - "Check for 365 Service Alert" (Microsoft Outlook Trigger)

- **Node Details:**

  **Check for 365 Service Alert**  
  - Type: Microsoft Outlook Trigger  
  - Technical Role: Watches mailbox for incoming emails matching specific criteria.  
  - Configuration:  
    - Polling interval: Every 1 minute  
    - Filters: Sender email must be "o365mc@microsoft.com" (Microsoft 365 service health alert sender)  
    - Output mode: Raw email data  
  - Expressions / Variables: None explicitly used here.  
  - Inputs: None (trigger node)  
  - Outputs: Emits new email items matching criteria for downstream processing.  
  - Edge Cases / Failures:  
    - Authentication failures due to invalid or expired OAuth2 credentials.  
    - No new emails found — workflow simply waits.  
    - Rate limits or temporary outages on Microsoft Graph API.  
  - Credentials: Outlook OAuth2 credential required (configured by user).

---

#### 2.2 Incident Data Extraction

- **Overview:**  
  Parses the HTML content of the received alert email to extract the incident summary text and the incident link URL for later summarization and notification.

- **Nodes Involved:**  
  - "Extract M365 Incident text & link" (Code node)

- **Node Details:**

  **Extract M365 Incident text & link**  
  - Type: Code (JavaScript) node  
  - Technical Role: Extract and clean relevant incident information from raw HTML email body.  
  - Configuration:  
    - Uses Cheerio library to parse HTML content.  
    - Extracts first anchor tag containing "servicehealth?message=" to find incident link.  
    - Removes extraneous HTML elements like scripts, styles, headers, footers, images.  
    - Aggregates text from headings, paragraphs, list items, tables into a clean plain text string with simple formatting.  
  - Expressions / Variables:  
    - Input: `items[0].json.body.content` (email HTML content)  
    - Output JSON keys:  
      - `extractedText`: cleaned, formatted alert text  
      - `incidentLink`: URL to the incident details  
  - Inputs: Output from "Check for 365 Service Alert"  
  - Outputs: JSON object with extracted text and link for summarization.  
  - Edge Cases / Failures:  
    - Missing or malformed HTML content triggers an error.  
    - Incident link may be absent or malformed, resulting in null.  
    - Parsing errors if email structure changes.  
  - Version Requirements: n8n version supporting external libraries in Code node.  
  - Credentials: None.

---

#### 2.3 AI-Powered Summarization

- **Overview:**  
  Sends the extracted incident text to OpenAI GPT-4O-MINI model to generate a concise, Slack-ready summary including user impact based on configured regions.

- **Nodes Involved:**  
  - "Summarize service alert" (LangChain OpenAI node)

- **Node Details:**

  **Summarize service alert**  
  - Type: LangChain OpenAI node (n8n native integration)  
  - Technical Role: Calls OpenAI API to produce structured summary output.  
  - Configuration:  
    - Model: "gpt-4o-mini" (cost-effective summarization model)  
    - Messages:  
      - System prompt defines strict formatting rules for Slack markdown, emoji status indicators, user impact logic (default regions US & Australia), and output JSON structure with `summaryText`, `incidentId`, `incidentLink`.  
      - User prompt injects extracted alert text and incident link for summarization.  
    - JSON output enabled for structured response parsing.  
  - Expressions / Variables:  
    - Injects extracted text and incident link via expression in user message.  
  - Inputs: Output from "Extract M365 Incident text & link"  
  - Outputs: JSON object with summarized Slack message text and incident metadata.  
  - Edge Cases / Failures:  
    - OpenAI API key invalid or quota exceeded.  
    - Model response format deviates, causing JSON parsing errors.  
    - Latency or timeout on API calls.  
  - Credentials: OpenAI API key credential required.  
  - Notes: User should customize system prompt’s region list to match organizational user base.

---

#### 2.4 Slack Notification Construction

- **Overview:**  
  Transforms the AI-generated summary JSON into a Slack Block Kit message with a button linking to the full incident details.

- **Nodes Involved:**  
  - "Generate Slack Block" (Code node)

- **Node Details:**

  **Generate Slack Block**  
  - Type: Code (JavaScript) node  
  - Technical Role: Creates a Slack message payload using the AI summary fields.  
  - Configuration:  
    - Constructs a single Slack section block with markdown text from `summaryText`.  
    - Adds a button accessory labeled with incident ID, linking to incident URL.  
  - Expressions / Variables:  
    - Reads from `$json.message.content` (output from AI node)  
  - Inputs: Output from "Summarize service alert" node  
  - Outputs: JSON formatted Slack message block  
  - Edge Cases / Failures:  
    - Missing or malformed AI output fields cause incomplete Slack message.  
    - Slack API may reject malformed blocks (handled downstream).  
  - Credentials: None.

---

#### 2.5 Slack Delivery & Email Cleanup

- **Overview:**  
  Posts the constructed Slack Block message to the configured Slack channel, then deletes the original alert email from the mailbox if delivery succeeds.

- **Nodes Involved:**  
  - "Post outage to Slack" (Slack node)  
  - "Clear email alert from mailbox" (Microsoft Outlook node)

- **Node Details:**

  **Post outage to Slack**  
  - Type: Slack node  
  - Technical Role: Sends a Slack message using Block Kit format to a selected channel.  
  - Configuration:  
    - Message type: Block  
    - Channel: User must specify Slack channel ID or select from list.  
    - Message blocks populated from previous node.  
    - Markdown enabled.  
    - Workflow link inclusion disabled.  
  - Expressions / Variables:  
    - Text fallback uses first block’s text for plain text display.  
  - Inputs: Output from "Generate Slack Block"  
  - Outputs: Message confirmation for next step  
  - Edge Cases / Failures:  
    - Slack API errors due to invalid token, missing permissions, or invalid channel.  
    - Network failures or rate limits.  
  - Credentials: Slack bot OAuth token with channel posting rights required.

  **Clear email alert from mailbox**  
  - Type: Microsoft Outlook node  
  - Technical Role: Deletes the processed alert email from mailbox to prevent reprocessing.  
  - Configuration:  
    - Operation: Delete  
    - Message ID: Pulled dynamically from original trigger email’s `id` field.  
  - Expressions / Variables:  
    - Uses expression `={{ $('Check for 365 Service Alert').item.json.id }}` to dynamically target message.  
  - Inputs: Output from "Post outage to Slack" (workflow logic assumes success before deletion)  
  - Outputs: None (final cleanup)  
  - Edge Cases / Failures:  
    - Message may already be deleted or inaccessible.  
    - Outlook permissions or connectivity issues.  
  - Credentials: Same Outlook OAuth credential as trigger.

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                           | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                                                                                                                                                                                                                                                                          |
|------------------------------|-------------------------------|------------------------------------------|-----------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Check for 365 Service Alert   | Microsoft Outlook Trigger      | Poll mailbox for new M365 alert emails   | None                        | Extract M365 Incident text & link |                                                                                                                                                                                                                                                                                                                                                                    |
| Extract M365 Incident text & link | Code (JavaScript)             | Parse HTML email, extract alert text & link | Check for 365 Service Alert | Summarize service alert    |                                                                                                                                                                                                                                                                                                                                                                    |
| Summarize service alert       | LangChain OpenAI node          | Generate Slack Markdown summary with incident info | Extract M365 Incident text & link | Generate Slack Block       |                                                                                                                                                                                                                                                                                                                                                                    |
| Generate Slack Block          | Code (JavaScript)              | Format AI output into Slack Block Kit message | Summarize service alert      | Post outage to Slack       |                                                                                                                                                                                                                                                                                                                                                                    |
| Post outage to Slack          | Slack node                    | Post formatted incident notification to Slack channel | Generate Slack Block         | Clear email alert from mailbox |                                                                                                                                                                                                                                                                                                                                                                    |
| Clear email alert from mailbox | Microsoft Outlook node         | Delete processed alert email from mailbox | Post outage to Slack         | None                      |                                                                                                                                                                                                                                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Check for 365 Service Alert"**  
   - Node Type: Microsoft Outlook Trigger  
   - Configure OAuth2 credential for mailbox receiving M365 alerts.  
   - Set polling interval: every 1 minute.  
   - Add filter: Sender equals "o365mc@microsoft.com".  
   - Output mode: Raw.

2. **Add "Extract M365 Incident text & link"**  
   - Node Type: Code (JavaScript)  
   - Paste the provided JS code that parses HTML content using Cheerio to extract `extractedText` and `incidentLink`.  
   - Connect input from "Check for 365 Service Alert".

3. **Add "Summarize service alert"**  
   - Node Type: LangChain OpenAI node  
   - Configure OpenAI credential with access to GPT-4O-MINI model.  
   - System prompt: Use the provided prompt specifying Slack markdown formatting, emoji codes for status, and user impact based on regions (customize region names as needed).  
   - User prompt: Inject extracted alert text and incident link via expression.  
   - Enable JSON output for structured parsing.  
   - Connect input from "Extract M365 Incident text & link".

4. **Add "Generate Slack Block"**  
   - Node Type: Code (JavaScript)  
   - Insert JS code that builds Slack Block Kit JSON with a section block and a button linking to the incident.  
   - Connect input from "Summarize service alert".

5. **Add "Post outage to Slack"**  
   - Node Type: Slack node  
   - Configure Slack bot OAuth2 credential with permissions to post messages to the target channel.  
   - Set message type to "block".  
   - Bind channelId to the desired Slack channel (select or enter ID).  
   - Set message blocks to expression output of previous node.  
   - Enable Markdown formatting.  
   - Connect input from "Generate Slack Block".

6. **Add "Clear email alert from mailbox"**  
   - Node Type: Microsoft Outlook node  
   - Configure same Outlook OAuth2 credential as trigger.  
   - Operation: Delete  
   - Message ID: Use expression to retrieve original email ID from "Check for 365 Service Alert" node: `={{ $('Check for 365 Service Alert').item.json.id }}`  
   - Connect input from "Post outage to Slack".

7. **Ensure execution order:**  
   - Trigger → Extract → Summarize → Generate Block → Post Slack → Delete Email.

8. **Test end-to-end:**  
   - Confirm Outlook credential access and Slack bot posting rights.  
   - Send sample M365 alert email to mailbox and verify Slack message formatting and deletion of email.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| OpenAI Projects allow budget control to limit monthly usage costs; recommended to use for GPT API key.                                                                        | https://help.openai.com/en/articles/9186755-managing-projects-in-the-api-platform                |
| Customize the system prompt in OpenAI node to reflect the user base regions for accurate impact statements in the summary.                                                  | Workflow Setup Instructions                                                                     |
| Slack message formatting uses Block Kit with a button linking to incident details for quick access.                                                                           | Slack Block Kit documentation: https://api.slack.com/block-kit                                   |
| Outlook OAuth2 credentials must have mailbox read and delete permissions.                                                                                                     | Microsoft Graph API / Outlook OAuth2 setup                                                       |
| The workflow runs every minute to ensure timely alerting with minimal delay.                                                                                                  | Polling interval configuration                                                                  |

---

This reference document fully covers all nodes, logic, and configuration details for the "Summarize Microsoft 365 Outage Alerts with ChatGPT and Send to Slack" n8n workflow, enabling reproduction, modification, and troubleshooting.