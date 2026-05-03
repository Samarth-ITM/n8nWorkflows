Route and prioritize Gmail support tickets to Notion with Gemini AI and Slack alerts

https://n8nworkflows.xyz/workflows/route-and-prioritize-gmail-support-tickets-to-notion-with-gemini-ai-and-slack-alerts-13770


# Route and prioritize Gmail support tickets to Notion with Gemini AI and Slack alerts

# Workflow Reference: Route and Prioritize Gmail Support Tickets to Notion with Gemini AI

This document provides a technical breakdown of an automated n8n workflow designed to streamline customer support by using AI to categorize, prioritize, and log incoming emails.

---

### 1. Workflow Overview
The workflow automates the transition from a raw support email to a structured entry in a Notion database, complemented by real-time Slack notifications. It reduces manual triage time by leveraging Large Language Models (LLMs) to understand intent and urgency.

**Logical Blocks:**
*   **1.1 Email Intake:** Monitors Gmail for new messages and prepares the data.
*   **1.2 AI Processing:** Uses Google Gemini to analyze the content, categorize the request, and draft a response.
*   **1.3 Data Persistence:** Logs the structured data and AI analysis into a Notion database.
*   **1.4 Routing & Notification:** Filters for high-priority issues to send urgent Slack alerts, while logging a standard summary for all other tickets.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Intake
**Overview:** This block acts as the entry point, polling Gmail for new messages and isolating relevant fields (sender, subject, body).
*   **Nodes Involved:** `Watch for support emails`, `Extract email fields`.
*   **Node Details:**
    *   **Watch for support emails (Gmail Trigger):** Polls the Gmail account every minute. Configured to ignore spam/trash.
    *   **Extract email fields (Set):** Normalizes the trigger data. It maps variables: `emailFrom`, `emailSubject`, `emailBody` (using a fallback to snippet if text is missing), and `emailDate`.
*   **Failure Modes:** Gmail OAuth2 expiry, connection timeouts.

#### 2.2 AI Analysis
**Overview:** Sends the extracted email text to Google Gemini for classification and summarization based on a specific JSON schema.
*   **Nodes Involved:** `Classify ticket with AI`, `Google Gemini`, `Parse AI classification`.
*   **Node Details:**
    *   **Classify ticket with AI (Basic LLM Chain):** Takes the email fields and prompts the AI to provide a Category (billing/tech/etc.), Priority (critical to low), a Summary, and a Suggested Response.
    *   **Google Gemini (Language Model):** Connected to the chain using the `gemini-2.0-flash` model for high-speed analysis.
    *   **Parse AI classification (Code):** A Javascript snippet that handles the AI output. It attempts to parse the JSON returned by the AI and merges it with the original email data.
*   **Edge Cases:** If the AI returns malformed JSON, the `try/catch` block defaults the ticket to "medium" priority and "general" category to prevent workflow failure.

#### 2.3 Route and Notify
**Overview:** Finalizes the process by updating external tools and alerting the human team.
*   **Nodes Involved:** `Create ticket in Notion`, `Is critical priority`, `Alert team on Slack`, `Post ticket summary to Slack`.
*   **Node Details:**
    *   **Create ticket in Notion (Notion):** Creates a new page in a specified Database. Requires mapping the AI-generated fields to Notion columns.
    *   **Is critical priority (If):** A logic gate checking if `{{ $json.priority }}` equals "critical".
    *   **Alert team on Slack (Slack):** Triggered only for "critical" tickets. Sends a formatted message with a rotating light emoji.
    *   **Post ticket summary to Slack (Slack):** Triggered for all non-critical tickets as a standard log.
*   **Failure Modes:** Missing Notion Database permissions, Slack channel ID mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Watch for support emails** | Gmail Trigger | Input Trigger | None | Extract email fields | Email intake: Fetches new support emails... |
| **Extract email fields** | Set | Data Normalization | Watch for support emails | Classify ticket with AI | Email intake: Fetches new support emails... |
| **Classify ticket with AI** | LLM Chain | AI Processing | Extract email fields | Parse AI classification | AI analysis: Gemini classifies the ticket... |
| **Google Gemini** | Chat Gemini | AI Model | None (Connection) | Classify ticket with AI | AI analysis: Gemini classifies the ticket... |
| **Parse AI classification** | Code | Data Sanitization | Classify ticket with AI | Create ticket in Notion | AI analysis: Gemini classifies the ticket... |
| **Create ticket in Notion** | Notion | Data Storage | Parse AI classification | Is critical priority | Route and notify: Creates a Notion page... |
| **Is critical priority** | If | Logic Filter | Create ticket in Notion | Alert team on Slack / Post ticket summary | Route and notify: Creates a Notion page... |
| **Alert team on Slack** | Slack | Urgent Notification | Is critical priority (True) | None | Route and notify: Creates a Notion page... |
| **Post ticket summary to Slack**| Slack | General Notification | Is critical priority (False) | None | Route and notify: Creates a Notion page... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Gmail Trigger** node. Set to poll "Every Minute". Authenticate with Gmail OAuth2.
2.  **Field Extraction:** Add a **Set** node. Define four String variables:
    *   `emailFrom`: `{{ $json.from.text }}`
    *   `emailSubject`: `{{ $json.subject }}`
    *   `emailBody`: `{{ $json.text || $json.snippet }}`
    *   `emailDate`: `{{ $json.date }}`
3.  **AI Integration:** 
    *   Add a **Basic LLM Chain** node. 
    *   Connect a **Google Gemini Chat Model** node to its sub-input. Set model to `gemini-2.0-flash`.
    *   Set the Chain Prompt to "Define" and include instructions to output JSON with keys: `category`, `priority`, `summary`, `suggestedResponse`.
4.  **JSON Parsing:** Add a **Code** node. Use `JSON.parse()` on the AI output to turn the string response into usable data objects. Include a `try/catch` to handle formatting errors.
5.  **Notion Storage:** Add a **Notion** node. Select "Database Page" -> "Create". Map the parsed JSON fields to your Notion database columns (ensure your Notion database has matching headers: Category, Priority, Summary, etc.).
6.  **Logic Gate:** Add an **If** node. Set the condition to: `{{ $json.priority }}` Equals `critical`.
7.  **Slack (High Priority):** Connect a **Slack** node to the "True" branch. Configure a message: `:rotating_light: CRITICAL TICKET: {{ $json.emailSubject }}`.
8.  **Slack (Standard):** Connect a second **Slack** node to the "False" branch. Configure a message: `New ticket logged: {{ $json.emailSubject }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires columns: Title, Category, Priority, Status, Summary, Suggested Response. | Notion Database Setup Requirement |
| AI analysis covers billing, technical, feature-request, and general categories. | Logic Scope |
| The `gemini-2.0-flash` model is recommended for balancing speed and cost. | Model Recommendation |