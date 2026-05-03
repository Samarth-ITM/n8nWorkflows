Get actionable Gmail alerts on Telegram with OpenAI

https://n8nworkflows.xyz/workflows/get-actionable-gmail-alerts-on-telegram-with-openai-13766


# Get actionable Gmail alerts on Telegram with OpenAI

This document provides a technical breakdown and implementation guide for the **AI Email Assistant** workflow. This system automates the process of monitoring Gmail, analyzing message priority via AI, and delivering actionable summaries to Telegram.

---

### 1. Workflow Overview

The **AI Email Assistant** is designed to reduce inbox fatigue by filtering noise and highlighting high-priority communications. It operates on a scheduled basis, pulling unread messages and using LLM intelligence to determine which emails require a manual response.

**Functional Logic Blocks:**
*   **1.1 Data Retrieval:** Triggers the workflow at specific times and fetches unread emails from the last 8 hours.
*   **1.2 Pre-processing:** Filters out empty results and prepares a lightweight JSON payload for AI consumption.
*   **1.3 AI Intelligence:** Utilizes OpenAI to classify emails, assess urgency, and generate summaries.
*   **1.4 Formatting & Notification:** Parses the AI output into a user-friendly Markdown format and sends it to Telegram if actionable items exist.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Retrieval
*   **Overview:** Triggers the automation and connects to the Gmail API to pull the most recent unread messages.
*   **Nodes Involved:** `Schedule 3x Daily Check`, `Gmail – Fetch Unread Emails`.
*   **Node Details:**
    *   **Schedule 3x Daily Check:** (Trigger) Set to run at 08:00, 13:00, and 18:00 daily.
    *   **Gmail – Fetch Unread Emails:** (Gmail Node) Configured to `getAll` messages. Filters: Label = `INBOX`, Status = `unread`, and Received After = `8 hours ago` (using expression: `{{ new Date(Date.now() - 8*60*60*1000).toISOString() }}`). Limit: 30 messages.

#### 2.2 Filtering & Preparation
*   **Overview:** Ensures there is data to process and strips unnecessary metadata to stay within LLM token limits.
*   **Nodes Involved:** `Unread Emails Found?`, `Prepare Email Data`.
*   **Node Details:**
    *   **Unread Emails Found?:** (If Node) Checks if the input array length is greater than 0. 
    *   **Prepare Email Data:** (Code Node) Uses JavaScript to map Gmail objects into a simplified structure: `id`, `from`, `subject`, `date`, `snippet` (limited to 500 characters), and `labelIds`.

#### 2.3 AI Intelligence
*   **Overview:** Analyzes the batch of emails to identify "real" messages vs. newsletters/spam.
*   **Nodes Involved:** `AI – Analyze & Classify Emails`.
*   **Node Details:**
    *   **AI – Analyze & Classify Emails:** (OpenAI Node) Uses `gpt-4o-mini`. 
    *   **Configuration:** Temperature set to 0.3 for consistency.
    *   **Prompting:** Instructs the AI to return a specific JSON schema identifying `needs_reply` (boolean), `summary`, and `priority` (high/medium/low).

#### 2.4 Formatting & Notification
*   **Overview:** Converts the AI's JSON output into a Telegram-compatible message and sends it.
*   **Nodes Involved:** `Format Telegram Alert`, `Action Required?`, `Telegram – Send Smart Summary`.
*   **Node Details:**
    *   **Format Telegram Alert:** (Code Node) Parses the AI string back into JSON. It filters for emails where `needs_reply` is true and maps priorities to emojis (🔴, 🟡, 🟢).
    *   **Action Required?:** (If Node) Terminates the workflow if no actionable emails were identified.
    *   **Telegram – Send Smart Summary:** (Telegram Node) Sends the text to a specific Chat ID using `Markdown` parse mode.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule 3x Daily Check | ScheduleTrigger | Workflow Trigger | None | Gmail – Fetch... | Flux: Schedule (3x/jour) → Gmail → AI Analysis → Telegram |
| Gmail – Fetch Unread Emails | Gmail | Data Ingestion | Schedule 3x... | Unread Emails Found? | Monitors your Gmail inbox for new emails. |
| Unread Emails Found? | If | Data Validation | Gmail – Fetch... | Prepare Email Data | Monitors your Gmail inbox for new emails. |
| Prepare Email Data | Code | Data Cleaning | Unread Emails Found? | AI – Analyze... | Uses AI (OpenAI) to analyze and classify each email. |
| AI – Analyze & Classify Emails | OpenAI | Intelligence | Prepare Email Data | Format Telegram Alert | Extracts key information: Urgency, Required action, Summary. |
| Format Telegram Alert | Code | Text Formatting | AI – Analyze... | Action Required? | Sends a structured, concise alert to Telegram. |
| Action Required? | If | Logic Gate | Format Telegram Alert | Telegram – Send... | Optionally highlights emails that require immediate reply. |
| Telegram – Send Smart Summary | Telegram | Final Delivery | Action Required? | None | The result: you receive smart, decision-ready notifications. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule Trigger** node. Add three rules for "Trigger at Hour": 8, 13, and 18.
2.  **Gmail Integration:** Add a **Gmail** node. 
    *   Connect your Gmail OAuth2 credentials.
    *   Set Resource to `Message`, Operation to `Get All`.
    *   Filter: `Label IDs` = `INBOX`, `Read Status` = `unread`.
    *   Use Expression for `Received After`: `{{ new Date(Date.now() - 8*60*60*1000).toISOString() }}`.
3.  **Check for Content:** Add an **If** node. Use logic: `{{ $nodes["Gmail – Fetch Unread Emails"].all().length }}` greater than `0`.
4.  **Data Scrubbing:** Add a **Code** node (JavaScript) to map the input array into a clean JSON array containing only the sender, subject, and snippet.
5.  **AI Configuration:** Add an **OpenAI** node.
    *   Model: `gpt-4o-mini`.
    *   System Prompt: Define a JSON structure including `needs_reply` and `summary`.
    *   Input: Reference the output of the "Prepare Email Data" node.
6.  **Message Formatting:** Add a **Code** node. 
    *   Parse the LLM response (`JSON.parse`).
    *   Loop through items to build a string (e.g., `msg += "From: " + e.from`). 
    *   Return a boolean `hasEmails` to signal if anything relevant was found.
7.  **Final Logic Gate:** Add an **If** node to check if `hasEmails` is `true`.
8.  **Telegram Delivery:** Add a **Telegram** node.
    *   Credentials: Use your Bot Token from @BotFather.
    *   Chat ID: Enter your personal Chat ID.
    *   Text: Use the formatted string from the previous node.
    *   **Important:** Set "Parse Mode" to `Markdown` in additional fields.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Prerequisites** | Requires Google Cloud Console (Gmail API), OpenAI API Key, and Telegram Bot Token. |
| **Target Audience** | Busy professionals, founders, freelancers, and support teams. |
| **Bot Setup** | Use [@BotFather](https://t.me/botfather) on Telegram to create your bot and get the API key. |
| **Urgency Mapping** | The workflow uses a 3-tier color system (Red/Yellow/Green) for email priority visualization. |