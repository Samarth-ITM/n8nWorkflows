Summarize Gmail Support Emails with Gemini and Post to Slack

https://n8nworkflows.xyz/workflows/summarize-gmail-support-emails-with-gemini-and-post-to-slack-7594


# Summarize Gmail Support Emails with Gemini and Post to Slack

---

### 1. Workflow Overview

This workflow automates the process of monitoring Gmail for new support emails, summarizing their content using Google Gemini’s large language model (LLM), and posting concise summaries to a Slack user. It targets customer support teams who want to efficiently stay updated on incoming support requests without reading full-length emails.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Detect new incoming support emails via Gmail Trigger.
- **1.2 AI Processing:** Use Langchain’s chain LLM node combined with Google Gemini to generate a summary of the email content.
- **1.3 Output Posting:** Post the generated summary to a specified Slack user.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block detects new support emails arriving in a Gmail inbox. It triggers the workflow execution immediately upon receiving new messages, passing the full email data downstream.

**Nodes Involved:**  
- Check Support Emails

**Node Details:**

- **Node:** Check Support Emails  
  - **Type:** Gmail Trigger  
  - **Technical Role:** Watches Gmail inbox for new emails, triggering workflow on arrival.  
  - **Configuration:**  
    - Polling every minute (near real-time detection).  
    - No filters configured (captures all incoming emails).  
    - Full email data retrieved (`simple` mode set to false).  
  - **Key Expressions/Variables:** Passes full JSON email data including `subject`, `from`, and `text` fields downstream.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connected to "Summarize Email and sent to Slack".  
  - **Version Requirements:** n8n v1.2 or later for Gmail Trigger node version 1.2.  
  - **Potential Failures:**  
    - Gmail OAuth token expiration or invalid credentials.  
    - Gmail API rate limits or downtime.  
    - Large email size or malformed emails.  

---

#### 1.2 AI Processing

**Overview:**  
This block takes the raw email content and uses Langchain’s chain LLM node with Google Gemini to produce a concise summary. It constructs a prompt incorporating key email metadata and sends it to the Gemini LLM.

**Nodes Involved:**  
- Summarize Email and sent to Slack  
- Google Gemini To Summarize Email

**Node Details:**

- **Node:** Summarize Email and sent to Slack  
  - **Type:** Langchain Chain LLM  
  - **Technical Role:** Defines the prompt and chains the input to the language model for response generation.  
  - **Configuration:**  
    - Prompt text: `"Summarize the following email: Subject - {{ $json.subject }}, From - {{ $json.from }}, Body - {{ $json.text }}."`  
    - Defines the summarization task explicitly in prompt.  
    - Uses `promptType` set to `define` for customized prompt.  
  - **Key Expressions:** Uses JSON path expressions to extract email subject, sender, and body dynamically.  
  - **Input Connections:** Receives from "Check Support Emails" node (email JSON).  
  - **Output Connections:** Outputs to "Post Summary to Slack User".  
  - **Version Requirements:** Requires n8n version supporting Langchain nodes v1.7 or higher.  
  - **Potential Failures:**  
    - Expression evaluation errors if input JSON fields missing or malformed.  
    - Downstream model invocation failures.  
  - **Sub-Workflow:** Invokes Google Gemini model via ai_languageModel connection.

- **Node:** Google Gemini To Summarize Email  
  - **Type:** Langchain LM Chat Google Gemini  
  - **Technical Role:** Connects to Google Gemini PaLM API to execute the LLM prompt for summarization.  
  - **Configuration:**  
    - Model name set to `models/gemini-1.5-flash` for a fast, high-quality summarization.  
    - No additional options configured.  
    - Uses Google Palm API credentials configured in n8n.  
  - **Key Expressions:** Receives prompt text from Langchain chain LLM node.  
  - **Input Connections:** Receives AI prompt from "Summarize Email and sent to Slack" node via ai_languageModel input.  
  - **Output Connections:** Returns summarized text to "Summarize Email and sent to Slack" node.  
  - **Version Requirements:** Requires configured Google Gemini (PaLM) API credentials.  
  - **Potential Failures:**  
    - API authentication errors due to expired or invalid credentials.  
    - Model invocation timeouts or quota limits.  
    - Network errors impacting API calls.  
  - **Sub-Workflow:** None (single node usage).

---

#### 1.3 Output Posting

**Overview:**  
This block posts the summarized email text into a Slack user’s direct message or channel, enabling the support team to receive concise updates instantly.

**Nodes Involved:**  
- Post Summary to Slack User

**Node Details:**

- **Node:** Post Summary to Slack User  
  - **Type:** Slack node  
  - **Technical Role:** Sends text messages to Slack users or channels.  
  - **Configuration:**  
    - Message text is set dynamically to the summarized text: `={{ $json.text }}`  
    - Target mode is set to "user" selection, but user ID is left empty (requires configuration to specify recipient).  
    - No other options enabled.  
    - Utilizes Slack API credentials stored in n8n (`Interview_feedback_Slack`).  
  - **Key Expressions:** Uses JSON field `text` from previous node output.  
  - **Input Connections:** Connected from "Summarize Email and sent to Slack".  
  - **Output Connections:** None (terminal node).  
  - **Version Requirements:** Requires Slack API credentials with permission to send messages. Node uses Slack node v2.3.  
  - **Potential Failures:**  
    - Slack OAuth token expiration or invalid credentials.  
    - Target user ID not configured or invalid, causing message send failure.  
    - Slack API rate limits or downtime.  

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                         | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                               |
|-------------------------------|--------------------------------|----------------------------------------|----------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------|
| Sticky Note                   | Sticky Note                   | Informational header                    | None                       | None                           | ## Summarize Gmail Support Emails with Gemini and Post to Slack                                           |
| Sticky Note1                  | Sticky Note                   | Node breakdown and descriptions        | None                       | None                           | # **Node Breakdown & Descriptions:** Workflow starts with Gmail Trigger, then Langchain + Gemini summarization, and finally Slack posting. |
| Check Support Emails          | Gmail Trigger                 | Detect new Gmail support emails         | None                       | Summarize Email and sent to Slack |                                                                                                           |
| Summarize Email and sent to Slack | Langchain Chain LLM           | Prepare prompt and send to Gemini LLM  | Check Support Emails        | Post Summary to Slack User      |                                                                                                           |
| Google Gemini To Summarize Email | Langchain LM Chat Google Gemini | Execute Gemini LLM summarization        | Summarize Email and sent to Slack (ai_languageModel) | Summarize Email and sent to Slack (ai_languageModel output) |                                                                                                           |
| Post Summary to Slack User    | Slack                        | Post summarized text to Slack user      | Summarize Email and sent to Slack | None                           |                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Add node: **Gmail Trigger**  
   - Name: `Check Support Emails`  
   - Configure credentials: Use OAuth2 credentials for Gmail account with support inbox access.  
   - Set polling interval: Every minute.  
   - Leave filters empty to capture all new emails.  
   - Set `Simple` mode to false to retrieve full email content.  

2. **Create Langchain Chain LLM Node:**  
   - Add node: **Langchain Chain LLM** (type: `@n8n/n8n-nodes-langchain.chainLlm`)  
   - Name: `Summarize Email and sent to Slack`  
   - Set prompt text to:  
     `="Summarize the following email: Subject - {{ $json.subject }}, From - {{ $json.from }}, Body - {{ $json.text }}."`  
   - Set `promptType` to `define`.  
   - Connect input from `Check Support Emails`.  

3. **Create Langchain LM Chat Google Gemini Node:**  
   - Add node: **Langchain LM Chat Google Gemini** (type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)  
   - Name: `Google Gemini To Summarize Email`  
   - Set `modelName` to `models/gemini-1.5-flash`.  
   - Configure Google Palm API credentials with valid Google Gemini (PaLM) API key.  
   - Connect this node as the `ai_languageModel` input of the `Summarize Email and sent to Slack` node (via Langchain chain LLM ai_languageModel input).  

4. **Create Slack Node:**  
   - Add node: **Slack**  
   - Name: `Post Summary to Slack User`  
   - Configure Slack API credentials with appropriate OAuth token allowing message posting.  
   - Set message text to dynamic expression: `={{ $json.text }}`  
   - Set target user mode to `user`. Select or enter the Slack user ID to receive the message.  
   - Connect input from `Summarize Email and sent to Slack`.  

5. **Connect Nodes:**  
   - Connect `Check Support Emails` → `Summarize Email and sent to Slack` (main output)  
   - Connect `Google Gemini To Summarize Email` → `Summarize Email and sent to Slack` (ai_languageModel input)  
   - Connect `Summarize Email and sent to Slack` → `Post Summary to Slack User` (main output)  

6. **Test the Workflow:**  
   - Trigger a new email to the Gmail inbox.  
   - Confirm that the email is detected, summarized, and the summary is posted to Slack user.  
   - Adjust Slack user ID or Gmail filters as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow starts with Gmail Trigger node for near real-time email detection.                         | Workflow overview in Sticky Note1 node.                                                                 |
| Uses Google Gemini LLM API via Langchain integration for high-quality text summarization.           | Requires Google Palm API credentials setup in n8n.                                                      |
| Slack node requires valid OAuth token with permissions to send direct messages or channel posts.    | Slack API credential: `Interview_feedback_Slack`.                                                       |
| The Langchain Chain LLM node’s prompt can be customized to change summarization style or detail.    | Prompt expression uses mustache templating syntax for dynamic email fields.                             |
| Google Gemini model used is `models/gemini-1.5-flash` optimized for fast summarization tasks.       | Model selection can be changed based on desired speed/quality tradeoffs.                                |
| Slack user ID must be configured manually to specify the message recipient.                          | Slack user selection is currently empty and requires manual input.                                      |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---