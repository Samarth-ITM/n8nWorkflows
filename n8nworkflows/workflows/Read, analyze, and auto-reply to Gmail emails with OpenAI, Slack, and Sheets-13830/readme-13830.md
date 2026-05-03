Read, analyze, and auto-reply to Gmail emails with OpenAI, Slack, and Sheets

https://n8nworkflows.xyz/workflows/read--analyze--and-auto-reply-to-gmail-emails-with-openai--slack--and-sheets-13830


# Read, analyze, and auto-reply to Gmail emails with OpenAI, Slack, and Sheets

# Workflow Reference: n8n OpenCWL AI Email Automation

This document provides a comprehensive technical breakdown of the n8n workflow designed to read, analyze, and automatically respond to Gmail messages using OpenAI.

---

### 1. Workflow Overview
The **n8n OpenCWL AI Email Automation** workflow is an end-to-end intelligent assistant. It monitors a Gmail inbox for unread messages, uses AI to categorize and determine the sentiment of the email, and then either drafts and sends an automated reply or escalates the message to a human operator via Slack if the content is deemed urgent or negative. Finally, every interaction is logged in Google Sheets for auditing and analytics.

**Logical Blocks:**
*   **1.1 Trigger & Extraction:** Polling the inbox and cleaning the raw email data.
*   **1.2 Configuration:** Defining business-specific rules and variables.
*   **1.3 AI Analysis:** Classifying the intent and sentiment using LLMs.
*   **1.4 Reply Generation:** Drafting a context-aware professional response.
*   **1.5 Escalation Gate:** Deciding between automated response or human intervention.
*   **1.6 Output & Logging:** Sending the email, applying labels, logging to Sheets, and notifying the team.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Extraction
**Overview:** This block identifies new emails and prepares the text for AI processing by removing HTML tags and limiting length.
*   **Nodes Involved:** `Gmail: Watch Inbox`, `Schedule (fallback)`, `Extract & Clean Email`.
*   **Node Details:**
    *   **Gmail: Watch Inbox:** Triggers every 5 minutes for "unread" emails. Uses Gmail OAuth2.
    *   **Extract & Clean Email (Code):** 
        *   *Role:* Uses JavaScript to extract `subject`, `body`, `sender`, and `threadId`. 
        *   *Logic:* Truncates body to 2000 characters and uses Regex `/<[^>]*>/g` to strip HTML.
        *   *Failure Point:* Empty email bodies or unusual character encoding.

#### 2.2 Configuration
**Overview:** A centralized node to manage settings without touching the logic of individual nodes.
*   **Nodes Involved:** `Set Email Config`.
*   **Node Details:**
    *   **Set Email Config (Set):** 
        *   *Parameters:* Sets `sender_name`, `auto_reply_categories`, `escalate_sentiments`, and the `log_sheet_id`.
        *   *Role:* Acts as a "Settings" panel for the user.

#### 2.3 AI Analysis
**Overview:** Uses OpenAI to understand the nature of the communication.
*   **Nodes Involved:** `OpenAI: Classify & Analyze`, `Parse Analysis`.
*   **Node Details:**
    *   **OpenAI: Classify & Analyze:** Sends the cleaned email text to OpenAI to determine Category (Support, Sales, etc.) and Sentiment (Urgent, Negative, etc.).
    *   **Parse Analysis (Code):** 
        *   *Logic:* Cleans the AI's markdown response (removing backticks) and parses the string into a JSON object.
        *   *Edge Case:* If OpenAI fails to return valid JSON, this node will throw an error.

#### 2.4 Reply Generation
**Overview:** Crafts the actual response based on the previous analysis.
*   **Nodes Involved:** `OpenAI: Draft Reply`, `Prepare Reply Data`.
*   **Node Details:**
    *   **OpenAI: Draft Reply:** Generates a professional response based on the sender's intent.
    *   **Prepare Reply Data (Code):** 
        *   *Logic:* Consolidates the draft with metadata. It calculates a Boolean `needs_escalation` if sentiment is "Urgent"/"Negative" or priority is "Critical".

#### 2.5 Escalation Gate & Output
**Overview:** Routes the workflow based on the AI's findings and executes external actions.
*   **Nodes Involved:** `Needs Escalation?`, `Slack: Escalate to Human`, `Gmail: Send Auto-Reply`, `Gmail: Apply Label`, `Google Sheets: Log Email`, `Slack: Notify Team`.
*   **Node Details:**
    *   **Needs Escalation? (IF):** Checks `needs_escalation === true`.
    *   **Slack Nodes:** Sends formatted blocks to a specific channel. One is for manual review (Escalation), the other for a success summary (Notify Team).
    *   **Gmail: Send Auto-Reply:** Sends the AI-generated text back to the `sender`.
    *   **Google Sheets: Log Email:** Appends a row to the specified Sheet with fields like `Sentiment`, `Priority`, and `Escalated (Yes/No)`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Gmail: Watch Inbox | gmailTrigger | Entry Point | None | Extract & Clean Email | 1. Trigger & Fetch Email |
| Schedule (fallback) | scheduleTrigger | Secondary Trigger | None | Set Email Config | 1. Trigger & Fetch Email |
| Extract & Clean Email | code | Data Normalization | Gmail: Watch Inbox | Set Email Config | 1. Trigger & Fetch Email |
| Set Email Config | set | Configuration | Extract & Clean Email | OpenAI: Classify & Analyze | 1. Trigger & Fetch Email |
| OpenAI: Classify & Analyze | openAi | AI Understanding | Set Email Config | Parse Analysis | 2. AI Analysis |
| Parse Analysis | code | Data Parsing | OpenAI: Classify & Analyze | OpenAI: Draft Reply | 2. AI Analysis |
| OpenAI: Draft Reply | openAi | Content Generation | Parse Analysis | Prepare Reply Data | 3. Draft AI Reply |
| Prepare Reply Data | code | Logic Preparation | OpenAI: Draft Reply | Needs Escalation? | 3. Draft AI Reply |
| Needs Escalation? | if | Routing | Prepare Reply Data | Slack: Escalate, Gmail: Send | 4. Escalation Gate |
| Slack: Escalate to Human | slack | Escalation | Needs Escalation? | None | 4. Escalation Gate |
| Gmail: Send Auto-Reply | gmail | Execution | Needs Escalation? | Gmail: Apply Label | 5. Send, Log & Notify |
| Gmail: Apply Label | gmail | Organization | Gmail: Send Auto-Reply | Google Sheets: Log Email | 5. Send, Log & Notify |
| Google Sheets: Log Email | googleSheets | Auditing | Gmail: Apply Label | Slack: Notify Team | 5. Send, Log & Notify |
| Slack: Notify Team | slack | Team Update | Google Sheets: Log Email | None | 5. Send, Log & Notify |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Ensure you have OAuth2 credentials for **Gmail**, **Google Sheets**, and **Slack**, plus an **OpenAI** API key.
2.  **Trigger Layer:**
    *   Add a **Gmail Trigger** node. Set to poll every 5 mins for "Unread" messages.
    *   Add a **Code Node** (Extract & Clean) to take the input and output a clean JSON with `subject`, `body` (stripped of HTML), and `sender`.
3.  **Configuration Layer:**
    *   Add a **Set Node**. Create variables for `sender_name`, `company_name`, `slack_channel`, and `log_sheet_id`. This allows easy global changes.
4.  **Intelligence Layer:**
    *   Add an **OpenAI Node**. Configure it to analyze the email body and return a JSON format containing `category`, `sentiment`, and `priority`.
    *   Add a **Code Node** (Parse) to convert the AI string into accessible JSON properties.
    *   Add a second **OpenAI Node** to draft the reply based on the parsed data.
5.  **Routing Logic:**
    *   Add a **Code Node** (Prepare Reply Data) to create a Boolean flag: `needs_escalation`.
    *   Add an **IF Node** to check if `needs_escalation` is true.
6.  **Action & Logging Layer:**
    *   **True Branch:** Add a **Slack Node** to send an escalation alert to the human team.
    *   **False Branch:** 
        1. Add a **Gmail Node** (Send) using the AI-drafted reply.
        2. Add a **Gmail Node** (Add Label) to mark the email as "AI-Processed".
        3. Add a **Google Sheets Node** (Append) to log all metadata.
        4. Add a **Slack Node** to post a confirmation message to the team channel.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Contact for Customization | [OneClick IT Solution Contact](https://www.oneclickitsolution.com/contact-us/) |
| Recommended Polling Interval | 5 Minutes (to avoid API rate limits) |
| Mandatory Gmail Scope | Must include `https://www.googleapis.com/auth/gmail.modify` to apply labels |
| Data Sanitization | Uses Regex in Code nodes to ensure AI doesn't process raw HTML tags |