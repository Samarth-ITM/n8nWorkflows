Track Zoom attendance and create AI summaries with Google Sheets, ClickUp and Telegram

https://n8nworkflows.xyz/workflows/track-zoom-attendance-and-create-ai-summaries-with-google-sheets--clickup-and-telegram-14959


# Track Zoom attendance and create AI summaries with Google Sheets, ClickUp and Telegram

# Technical Documentation: Zoom Attendance Tracker & AI Summarizer

### 1. Workflow Overview
This workflow automates the post-meeting administrative process for Zoom calls. It triggers immediately after a meeting ends to log attendance, identify late participants, and generate an AI-powered executive summary for the host.

The logic is divided into five functional blocks:
- **1.1 Input & Configuration:** Captures the Zoom event and initializes global settings.
- **1.2 Participant Processing:** Parses raw Zoom data and classifies participants based on a time threshold.
- **1.3 Data Archiving:** Records comprehensive attendance data into Google Sheets.
- **1.4 AI Summary Generation:** Uses LLM logic to create a concise summary for the host (triggered only once).
- **1.5 Follow-up Automation:** Identifies late joiners and creates targeted tasks in ClickUp.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration
**Overview:** Handles the entry point from Zoom and defines the environment variables used throughout the workflow.
- **Nodes Involved:** `1. Webhook — Zoom Meeting Ended`, `2. Set — Config Values`.
- **Node Details:**
    - **Webhook:** Listens for `POST` requests on `/zoom-meeting-ended`. This is the trigger.
    - **Set (Config Values):** A centralized configuration hub. It defines `telegramChatId`, `sheetId`, `sheetName`, `hostName`, `lateThresholdMinutes` (default: 5), and `clickupListId`.
    - **Failure types:** Webhook authentication failure or incorrect ID formats in the Set node.

#### 2.2 Participant Extraction
**Overview:** Transforms the Zoom payload into a structured list of participants and determines their attendance status.
- **Nodes Involved:** `3. Code — Extract and Classify Participants`.
- **Node Details:**
    - **Code (JavaScript):** This node calculates the `joinDelayMin` by comparing the meeting start time with the participant's actual join time.
    - **Logic:** If `joinDelayMin > lateThresholdMinutes`, status is set to `"Late"`; otherwise, it is `"Attended"`.
    - **Output:** An array of items (one per participant), each enriched with meeting metadata and a boolean `isFirstRow` to prevent duplicate summaries.

#### 2.3 Sheet Logging
**Overview:** Maintains a permanent record of every single participant for auditing purposes.
- **Nodes Involved:** `4. Google Sheets — Log Participant Row`.
- **Node Details:**
    - **Google Sheets (Append):** Maps 14 specific columns (Topic, Participant Name, Email, Join/Leave times, Status, etc.).
    - **Configuration:** Uses `USER_ENTERED` cell formatting to ensure dates and times are handled correctly by Sheets.
    - **Edge Case:** If the Sheet ID in the config is incorrect, all subsequent rows will fail to log.

#### 2.4 Telegram Summary
**Overview:** Generates a professional summary of the meeting's attendance metrics using AI.
- **Nodes Involved:** `5. IF — First Row Check`, `6. AI Agent — Write Telegram Summary`, `7. OpenAI — GPT-4o-mini Model`, `11. Set — Prepare Telegram Fields`, `12. Telegram — Send Meeting Summary`.
- **Node Details:**
    - **IF Node:** Acts as a gate; only the item where `isFirstRow === true` passes to the AI agent.
    - **AI Agent:** Uses a detailed prompt to format a 11-line summary (Topic, Date, Counts, Observation, and Action Tip).
    - **OpenAI Model:** GPT-4o-mini with a low temperature (0.4) for consistency.
    - **Telegram Node:** Sends the final formatted string to the configured Chat ID.

#### 2.5 Late Participant Actions
**Overview:** Triggers a remediation process for anyone classified as "Late".
- **Nodes Involved:** `8. IF — Is Participant Late?`, `9. ClickUp — Create Late Participant Task`, `10. Set — No Action Needed`.
- **Node Details:**
    - **IF Node:** Filters for items where `status === "Late"`.
    - **ClickUp Node:** Creates a task in the specified list. The task description includes full meeting context and a specific "Action Required" list.
    - **Due Date:** Set automatically to `now + 1 day`.
    - **Set (No Action):** A "sink" node for on-time participants to ensure the workflow completes cleanly.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Webhook — Zoom Meeting Ended | Webhook | Trigger | None | 2. Set — Config Values | Webhook and Config |
| 2. Set — Config Values | Set | Global Config | 1. Webhook | 3. Code | Webhook and Config |
| 3. Code — Extract... | Code | Data Parsing | 2. Set | 4. Google Sheets, 5. IF | Participant Extraction |
| 4. Google Sheets — Log... | Google Sheets | Archiving | 3. Code | None | Sheet Logging |
| 5. IF — First Row Check | IF | Logic Gate | 3. Code | 6. AI Agent, 8. IF | First Row Gate |
| 6. AI Agent — Write... | AI Agent | Content Gen | 5. IF | 11. Set | Telegram Summary |
| 7. OpenAI — GPT-4o-mini | Language Model | LLM Engine | None (Linked) | 6. AI Agent | Telegram Summary |
| 8. IF — Is Participant Late? | IF | Logic Gate | 5. IF | 9. ClickUp, 10. Set | Late Participant Actions |
| 9. ClickUp — Create... | ClickUp | Task Creation | 8. IF | None | Late Participant Actions |
| 10. Set — No Action Needed | Set | Termination | 8. IF | None | Late Participant Actions |
| 11. Set — Prepare... | Set | Data Mapping | 6. AI Agent | 12. Telegram | Telegram Summary |
| 12. Telegram — Send... | Telegram | Notification | 11. Set | None | Telegram Summary |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    - Create a **Webhook** node. Set the path to `zoom-meeting-ended` and method to `POST`.
2.  **Configuration Setup:**
    - Add a **Set** node. Define 6 string values: `telegramChatId`, `sheetId`, `sheetName`, `hostName`, `lateThresholdMinutes`, and `clickupListId`.
3.  **Data Processing:**
    - Add a **Code** node. Paste the JavaScript logic provided in the workflow to calculate join delays and create the `attendanceRows` array.
4.  **Logging:**
    - Add a **Google Sheets** node. Set operation to "Append". Map the columns manually based on the expressions (e.g., `{{ $json.participantName }}`).
5.  **Flow Branching (AI Summary):**
    - Add an **IF** node. Condition: `{{ $json.isFirstRow }}` is true.
    - Connect the **True** output to an **AI Agent** node.
    - Attach an **OpenAI Chat Model** node (gpt-4o-mini) to the AI Agent.
    - Connect the AI Agent to a **Set** node (to clean the output) and then to a **Telegram** node.
6.  **Flow Branching (Late Tasks):**
    - Connect the **False** output of the "First Row Check" IF node to a second **IF** node.
    - Condition: `{{ $json.status }}` equals `Late`.
    - Connect the **True** output to a **ClickUp** node. Configure it to "Create Task" using the `clickupListId` and map the task content to a detailed string including participant and meeting info.
    - Connect the **False** output to a **Set** node titled "No Action Needed".
7.  **Credentials:**
    - Connect OAuth2 for Google Sheets.
    - Connect API Key for OpenAI.
    - Connect Bot Token for Telegram.
    - Connect OAuth2 for ClickUp.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zoom Webhook Setup | Requires a "Webhook Only App" created in the Zoom Marketplace. |
| GPT-4o-mini | Selected for high speed and low cost for short text summaries. |
| Sheet Formatting | Ensure the Google Sheet has a tab name matching the `sheetName` config. |