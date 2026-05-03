Automate WhatsApp lead follow-ups and nurturing with WATI, Airtable and OpenAI

https://n8nworkflows.xyz/workflows/automate-whatsapp-lead-follow-ups-and-nurturing-with-wati--airtable-and-openai-13723


# Automate WhatsApp lead follow-ups and nurturing with WATI, Airtable and OpenAI

This document provides a technical analysis and reproduction guide for the **WhatsApp Lead Follow-up & Nurturing** n8n workflow. This system integrates WATI, Airtable, and OpenAI to automate CRM management, A/B tested outreach, and intelligent reply processing.

---

### 1. Workflow Overview
The workflow is designed to manage the entire lifecycle of a WhatsApp lead—from enrollment and automated nurturing to intent detection and performance reporting. It is divided into five functional modules:

*   **1.1 Command Routing:** Acts as the central intake, sorting incoming WhatsApp messages into specific pipelines based on keywords (`enroll`, `report`, `pause`) or treating them as lead engagement.
*   **1.2 Campaign Setup (Enrollment):** Parses new lead data via a specific command, assigns them to an A/B testing variant, and initializes their record in Airtable.
*   **1.3 Follow-up Scheduler:** A daily automated process that identifies leads due for a touchpoint and uses OpenAI to generate personalized messages.
*   **1.4 Engagement Tracker:** Analyzes incoming replies using AI to categorize intent (e.g., "Interested", "Unsubscribe") and updates the CRM status accordingly.
*   **1.5 Analytics Dashboard:** Generates on-demand A/B testing reports and conversion statistics.

---

### 2. Block-by-Block Analysis

#### 2.1 Command Routing & Pipeline Selection
This block monitors the WATI webhook and directs traffic.
*   **Nodes Involved:** `Wati Trigger`, `Command Router`.
*   **Details:** The `Command Router` (Switch node) uses string operations (Starts With, Equals) to determine if a message is a system command or a lead's reply.
*   **Edge Cases:** Non-text messages (images/locations) might fail if the `text` property is missing.

#### 2.2 Pipeline A: Campaign Setup (Enrollment)
Extracts lead data from a formatted string and initializes the CRM.
*   **Nodes Involved:** `Parse Enroll Command`, `Airtable – Create Contact`, `WATI – Confirm Enrollment`, `WATI – Send Welcome to Contact`.
*   **Configuration:** 
    *   **Code Node:** Uses Regex/Split logic to extract `phone`, `name`, `company`, and `campaignId`. It assigns a variant ('A' or 'B') using `Math.random()`.
    *   **Airtable:** Creates a record with status "Active" and sets the first follow-up for "Tomorrow".
*   **Failure Modes:** Incorrect command formatting (handled by an error message sent back to the rep).

#### 2.3 Pipeline B: Follow-up Scheduler
The engine that drives automated nurturing based on timing and personalization.
*   **Nodes Involved:** `Schedule Trigger – 9AM Daily`, `Airtable – Read Active Contacts`, `Filter Due Follow-ups`, `Airtable – Read Campaign`, `OpenAI – Personalise Follow-up Message`, `Build Follow-up Row`, `Airtable – Log Follow-up Sent`, `Airtable – Update Contact After Send`, `WATI – Send Follow-up`.
*   **Configuration:**
    *   **Filter Node:** Compares `nextFollowUp` date with the current date.
    *   **OpenAI (HTTP Request):** Sends contact details (Name, Company, Variant) to the Chat Completion API to generate a message tailored to the A/B variant's tone.
    *   **CRM Update:** Increments the `followUpCount` and calculates the `nextFollowUp` date based on the campaign's interval.

#### 2.4 Pipeline C: Engagement Tracker (Reply Processing)
Intelligent analysis of lead replies to stop or escalate outreach.
*   **Nodes Involved:** `Airtable – Find Contact by Phone`, `Find Contact Record`, `OpenAI – Detect Reply Intent`, `Process Reply & Build Log`, `Airtable – Log Engagement`, `Airtable – Update Contact Status`, `WATI – Send Reply Acknowledgement`.
*   **Configuration:**
    *   **OpenAI:** Classifies text into: `interested`, `not_interested`, `question`, `converted`, or `unsubscribe`.
    *   **Logic:** If intent is `converted` or `unsubscribe`, the status is updated to prevent further automated messages.

#### 2.5 Pipeline D: Analytics Dashboard
Provides a snapshot of campaign health via WhatsApp.
*   **Nodes Involved:** `Airtable – Read All Follow-ups`, `Airtable – Read All Engagement`, `Build Analytics Report`, `WATI – Send Analytics Report`.
*   **Configuration:**
    *   **Code Node:** Aggregates data to calculate Reply Rate, Conversion Rate, and compares Variant A vs. Variant B performance.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Wati Trigger | WATI Trigger | Webhook Entry | None | Command Router | Branch: Command Routing |
| Command Router | Switch | Logic Branching | Wati Trigger | Parse Enroll, Read All Follow-ups, Parse Pause, Find Contact | Branch: Command Routing |
| Parse Enroll Command | Code | Data Extraction | Command Router | Airtable – Create Contact | Pipeline A · Campaign Setup |
| Airtable – Create Contact | Airtable | Record Creation | Parse Enroll | WATI – Confirm Enrollment, WATI – Send Welcome | Pipeline A · Campaign Setup |
| WATI – Confirm Enrollment | WATI | Rep Notification | Airtable – Create | None | Pipeline A · Campaign Setup |
| WATI – Send Welcome | WATI | Lead Notification | Airtable – Create | None | Pipeline A · Campaign Setup |
| Schedule Trigger – 9AM | Schedule | Timer | None | Airtable – Read Active Contacts | Pipeline B: Follow-up Scheduler |
| Filter Due Follow-ups | Code | Date Filtering | Airtable – Read Active | Airtable – Read Campaign | Pipeline B: Follow-up Scheduler |
| OpenAI – Personalise... | HTTP Request | AI Content Gen | Airtable – Read Camp. | Build Follow-up Row | Pipeline B: Follow-up Scheduler |
| Build Follow-up Row | Code | Data Formatting | OpenAI | Airtable – Log Follow-up | Pipeline B: Follow-up Scheduler |
| Airtable – Log Follow-up | Airtable | Logging | Build Follow-up Row | Airtable – Update Contact | Pipeline B: Follow-up Scheduler |
| WATI – Send Follow-up | WATI | Outreach | Airtable – Update Contact | None | Pipeline B: Follow-up Scheduler |
| OpenAI – Detect Intent | HTTP Request | AI Analysis | Find Contact Record | Process Reply & Build Log | Pipeline C: Engagement Tracker |
| Airtable – Log Engagement| Airtable | Interaction Log | Process Reply | Airtable – Update Contact | Pipeline C: Engagement Tracker |
| Build Analytics Report | Code | Data Aggregation | Read All Engagement | WATI – Send Analytics | Pipeline D · Analytics Dashboard |
| Parse Pause Command | Code | Data Parsing | Command Router | Airtable – Pause Contact | Branch: Pause Outreach |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Airtable Setup
Create a Base named "Campaign Manager" with three tables:
1.  **Contacts Table:** Columns: `Name`, `phone` (Text), `company`, `Campaign Id`, `Variant` (A/B), `followUpCount` (Number), `nextFollowUp` (Date), `status` (Select: Active, Paused, Converted, Unsubscribed), `enrolledAt` (Date).
2.  **FollowUps Table:** Columns: `contactId`, `phone`, `campaignId`, `variant`, `stepNumber`, `messageText`, `sentAt`.
3.  **Engagement Table:** Columns: `contactId`, `intent`, `replyText`, `repliedAt`.

#### Step 2: Main Entry & Routing
1.  Add a **WATI Trigger** node (Event: `messageReceived`).
2.  Connect a **Switch** node (`Command Router`):
    *   Output 1: String starts with `enroll `.
    *   Output 2: String equals `report`.
    *   Output 3: String starts with `pause `.
    *   Output 4 (Fallback): Extra.

#### Step 3: Enrollment Pipeline (Output 1)
1.  **Code Node:** Split the string `{{ $json.text }}` to extract variables. Add `Math.random()` to assign Variant A or B.
2.  **Airtable Node:** Use "Create" on the Contacts Table.
3.  **WATI Node:** Send a message to the *Sender* confirming success and a message to the *Lead* welcoming them.

#### Step 4: Daily Follow-up Engine
1.  **Schedule Trigger:** Set to `0 9 * * *`.
2.  **Airtable Node:** "List" contacts where `status` is "Active".
3.  **Code Node:** Filter records where `nextFollowUp` <= Today.
4.  **OpenAI (HTTP Request):** POST to `https://api.openai.com/v1/chat/completions`. 
    *   *Prompt:* "Write a {{variant}} follow-up for {{name}} at {{company}}."
5.  **Airtable Node:** Create a record in "FollowUps Table".
6.  **Airtable Node:** Update "Contacts Table" (set `nextFollowUp` to +3 days, increment `followUpCount`).
7.  **WATI Node:** Send the AI-generated message to the lead.

#### Step 5: Intent Processing (Fallback Output)
1.  **Airtable Node:** "List" contacts and find the record matching the incoming `waId`.
2.  **OpenAI (HTTP Request):** Send the reply text. 
    *   *Prompt:* "Classify this reply intent: [Text]. Categories: interested, not_interested, question, converted, unsubscribe."
3.  **Code Node:** Map the intent to a new status (e.g., "converted" -> "Converted").
4.  **Airtable Node:** Update contact status and Log Engagement.
5.  **WATI Node:** Send a context-aware acknowledgement (e.g., "Thanks, we'll get a rep to call you").

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **A/B Testing Logic** | Variant A is Friendly/Informal, Variant B is Professional/Direct. |
| **Security Note** | Ensure OpenAI API keys are stored in n8n credentials, not hardcoded in the HTTP node. |
| **WATI Setup** | Requires a valid API Key and Endpoint URL from the WATI Dashboard (API Docs). |
| **Formatting** | WhatsApp formatting (*bold*, _italics_) is handled within the Code nodes for better UX. |