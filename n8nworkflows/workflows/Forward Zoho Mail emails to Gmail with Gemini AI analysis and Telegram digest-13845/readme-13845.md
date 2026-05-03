Forward Zoho Mail emails to Gmail with Gemini AI analysis and Telegram digest

https://n8nworkflows.xyz/workflows/forward-zoho-mail-emails-to-gmail-with-gemini-ai-analysis-and-telegram-digest-13845


# Forward Zoho Mail emails to Gmail with Gemini AI analysis and Telegram digest

# Reference Document: Smart Email Forwarder with AI Analysis

## 1. Workflow Overview
This workflow automates the process of monitoring a **Zoho Mail** inbox, analyzing incoming messages using **Google Gemini AI**, and forwarding them to a **Gmail** address. It features a sophisticated AI-driven analysis that classifies emails by priority and type, extracts deadlines and action items, and attaches an easy-to-read summary panel to the forwarded email. Additionally, it sends a real-time digest to a **Telegram** chat for instant mobile oversight.

### Logical Blocks:
*   **1.1 Input & AI Intelligence:** Monitors the inbox, sets global configurations, and utilizes LLMs to parse and categorize email content.
*   **1.2 Content Formatting:** Generates a custom HTML email body containing the AI's findings (Priority, Summary, Action Required).
*   **1.3 Attachment Handling (Conditional):** A specialized pipeline that interacts with the Zoho Mail API to download and aggregate binary files if the email contains attachments.
*   **1.4 Delivery & Notification:** Forwards the enriched email via Gmail and sends a structured summary to Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Input & AI Intelligence
This block captures the raw email and transforms it into structured data.
*   **Nodes Involved:** `Zoho Mail Trigger`, `Set Context`, `AI Agent`, `Google Gemini Chat Model`, `Structured Output Parser`.
*   **Node Details:**
    *   **Zoho Mail Trigger:** Captures new emails. Configured to filter by specific "from" addresses.
    *   **Set Context (Set Node):** Centralized configuration. Defines variables like `forward_to` email, `telegram_chat_id`, and `ai_type_options`.
    *   **AI Agent:** Uses the Gemini model to process the email subject and body. It is instructed via a System Message to return a specific JSON structure.
    *   **Google Gemini Chat Model:** The brain of the analysis (Google Palm API).
    *   **Structured Output Parser:** Ensures the AI output matches the schema (type, priority, summary, deadline, action_required, key_numbers).

### 2.2 Content Formatting
Converts AI data into a visual layout.
*   **Nodes Involved:** `Build Email HTML`.
*   **Node Details:**
    *   **Build Email HTML (Code Node):** A JavaScript node that constructs a responsive HTML template. It maps priorities (High/Medium/Low) to colored badges and emojis and prepends the AI analysis to the original email body.

### 2.3 Attachment Handling Pipeline
Only executes if the "Has Attachment?" condition is met.
*   **Nodes Involved:** `Has Attachment?`, `Get AccountsID`, `Get Attachment Info`, `Split Out Attachments`, `Get Attachment Content`, `Aggregate Attachments`, `Build Attachment List`.
*   **Node Details:**
    *   **Has Attachment? (If Node):** Checks the `hasAttachment` boolean from the Zoho trigger.
    *   **Get AccountsID (HTTP Request):** Calls Zoho API to find the user's Account ID required for file paths.
    *   **Get Attachment Info (HTTP Request):** Retrieves metadata for specific attachments linked to the message ID.
    *   **Split Out Attachments:** Handles cases with multiple files by creating individual items for each.
    *   **Get Attachment Content (HTTP Request):** Downloads the actual binary data for each file.
    *   **Aggregate/Build List:** Reassembles individual files into a single object that the Gmail node can process as a list of attachments.

### 2.4 Delivery & Notification
The final stage for user alerts.
*   **Nodes Involved:** `Forward Email with Attachments`, `Forward Email (No Attachments)`, `Notify Admin via Telegram`.
*   **Node Details:**
    *   **Gmail Nodes:** Sends the HTML generated in Block 2.2. The attachment-enabled version maps the aggregated binary data.
    *   **Notify Admin via Telegram:** Sends a Markdown-formatted message to the admin's Telegram ID with a summary of the AI's findings.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Zoho Mail Trigger** | Zoho Mail Trigger | Entry Point | - | Set Context | Section 1: Trigger & AI Analysis; Edit these nodes! |
| **Set Context** | Set | Global Config | Zoho Mail Trigger | AI Agent | Section 1: Trigger & AI Analysis; Edit these nodes! |
| **AI Agent** | AI Agent | AI Processing | Set Context | Build Email HTML | Section 1: Trigger & AI Analysis |
| **Google Gemini Chat Model** | Google Gemini | AI Model | - | AI Agent | Section 1: Trigger & AI Analysis |
| **Structured Output Parser** | Output Parser | Data Validation | - | AI Agent | Section 1: Trigger & AI Analysis |
| **Build Email HTML** | Code | UI/UX Generation | AI Agent | Has Attachment? | Section 1: Trigger & AI Analysis |
| **Has Attachment?** | If | Logic Routing | Build Email HTML | Get AccountsID, Forward Email (No Attach) | Section 1: Trigger & AI Analysis |
| **Get AccountsID** | HTTP Request | API Discovery | Has Attachment? | Get Attachment Info | Section 2: Attachment Download Pipeline; ⚠️ Important! Enter Zoho credentials. |
| **Get Attachment Info** | HTTP Request | Metadata Fetch | Get AccountsID | Split Out Attachments | Section 2: Attachment Download Pipeline; ⚠️ Important! Enter Zoho credentials. |
| **Split Out Attachments** | Split Out | Data Unrolling | Get Attachment Info | Get Attachment Content | Section 2: Attachment Download Pipeline |
| **Get Attachment Content** | HTTP Request | Binary Download | Split Out Attachments | Aggregate Attachments | Section 2: Attachment Download Pipeline; ⚠️ Important! Enter Zoho credentials. |
| **Aggregate Attachments** | Aggregate | Data Rolling | Get Attachment Content | Build Attachment List | Section 2: Attachment Download Pipeline |
| **Build Attachment List** | Code | Binary Mapping | Aggregate Attachments | Forward Email with Attachments | Section 2: Attachment Download Pipeline |
| **Forward Email...** | Gmail | Delivery | Build Attachment List / Has Attachment? | Notify Admin via Telegram | Section 3: Forward & Smart Notify |
| **Notify Admin...** | Telegram | Notification | Gmail Nodes | - | Section 3: Forward & Smart Notify |

---

## 4. Reproducing the Workflow from Scratch

1.  **Authentication Setup:**
    *   Configure **Zoho Mail OAuth2** (Requires Client ID, Secret, and appropriate Scopes for mail read/access).
    *   Configure **Gmail OAuth2** with `https://www.googleapis.com/auth/gmail.send` scope.
    *   Generate a **Google Gemini API Key** (Google AI Studio).
    *   Create a **Telegram Bot** via `@BotFather` and obtain the Chat ID.

2.  **Configuration Node:**
    *   Create a `Set` node named "Set Context". Add string fields for: `forward_to`, `subject_prefix`, `telegram_chat_id`, `footer_note`, `ai_context`, and `ai_type_options`.

3.  **AI Integration:**
    *   Add an `AI Agent` node. Set the System Message to use the `ai_context` from the Set node.
    *   Connect a `Google Gemini Chat Model` and a `Structured Output Parser`.
    *   Define the JSON schema in the parser: `type`, `priority`, `summary`, `deadline`, `action_required`, `key_numbers`.

4.  **HTML Generation:**
    *   Add a `Code` node. Use JavaScript to wrap the AI output in a `<!DOCTYPE html>` template with CSS for styles like `.ai-box` and `.badge`.

5.  **Conditional logic:**
    *   Add an `If` node to check if `hasAttachment` is "Yes".
    *   **False Path:** Connect directly to a Gmail "Send" node.
    *   **True Path:** Create a sequence of HTTP nodes:
        *   `GET https://mail.zoho.com/api/accounts`
        *   `GET .../attachmentinfo`
        *   `GET .../attachments/{{$json.attachmentId}}` (Response: File).

6.  **Binary Processing:**
    *   Use the `Aggregate` node to combine binary items.
    *   Use a `Code` node to map filenames into a format Gmail understands for the "Attachments" field.

7.  **Finalize Outputs:**
    *   Setup the Gmail node to use the constructed HTML as the "Email Body".
    *   Setup the Telegram node using Markdown to display the AI summary.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Author / Creator** | Nguyen Thieu Toan (Jay Nguyen) - [n8n Profile](https://n8n.io/creators/nguyenthieutoan/) |
| **Support / Donations** | Help the creator: [Donate Website](https://nguyenthieutoan.com/payment/) |
| **Company** | GenStaff - [genstaff.net](https://genstaff.net) |
| **License** | Internal GenStaff use; requires attribution to Nguyen Thieu Toan. |
| **Customization** | Swap Zoho for Gmail or Outlook triggers easily by updating the Input node. |