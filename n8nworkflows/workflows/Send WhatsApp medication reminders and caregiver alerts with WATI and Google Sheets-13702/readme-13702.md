Send WhatsApp medication reminders and caregiver alerts with WATI and Google Sheets

https://n8nworkflows.xyz/workflows/send-whatsapp-medication-reminders-and-caregiver-alerts-with-wati-and-google-sheets-13702


# Send WhatsApp medication reminders and caregiver alerts with WATI and Google Sheets

# Reference Document: WhatsApp Medication Reminder & Adherence Tracking System

This document provides a technical breakdown of the n8n workflow designed to automate medication reminders via WhatsApp (WATI) and track patient adherence using Google Sheets.

---

### 1. Workflow Overview

The workflow is a comprehensive health-tech automation designed to improve medication adherence. It functions through three primary mechanisms:
1.  **Proactive Dispatch:** Sends personalized medication lists to patients three times daily.
2.  **Reactive Logging:** Processes patient replies (taken, skip, snooze) to update a centralized Google Sheets database.
3.  **Safety Monitoring:** Automatically alerts caregivers if doses are skipped multiple times or if a patient fails to respond within a two-hour window.

**Logical Blocks:**
*   **1.1 Scheduled Dispatch:** Triggered at 8 AM, 1 PM, and 8 PM to filter and send reminders.
*   **1.2 Inbound Interaction:** Routes incoming WhatsApp messages based on keywords.
*   **1.3 Adherence Logic:** Updates logs for "Taken" or "Skipped" doses and manages "Snooze" requests.
*   **1.4 Automated Monitoring:** Periodically checks for missed doses or pending snoozes.
*   **1.5 Reporting Engine:** Generates on-demand stats for patients and summary reports for caregivers.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Reminder Dispatch
Automatically identifies which medications are due for which patients based on the current time of day and sends a consolidated WhatsApp message.

*   **Nodes Involved:** 
    *   `Schedule Trigger – 8AM 1PM 8PM`
    *   `Google Sheets – Read Medications`
    *   `Filter Medications by Time Slot`
    *   `Build Medication Reminder`
    *   `Google Sheets – Log Reminder Sent`
    *   `Send a text message`
*   **Node Details:**
    *   **Filter Medications by Time Slot (Code):** Determines the "current slot" (morning/afternoon/night) based on the execution hour. It filters rows from the "Medications" sheet where the status is `active` and groups meds by patient phone number to avoid sending multiple messages to one person.
    *   **Build Medication Reminder (Code):** Formats the WhatsApp message with emojis and structured text. It generates a unique `logKey` (phone + date + slot) used for tracking responses.
    *   **Google Sheets – Log Reminder Sent:** Appends a row to the `AdherenceLog` sheet with a "Pending" status.
    *   **Failure Modes:** API limit reached on Google Sheets; WATI credential expiration; incorrect time zone settings in n8n environment.

#### 2.2 Inbound Routing
The entry point for all patient and caregiver interactions.

*   **Nodes Involved:**
    *   `Wati Trigger`
    *   `Route Message`
*   **Node Details:**
    *   **Route Message (Switch):** Uses string matching on the incoming text (converted to lowercase). It directs the flow to six distinct paths: `taken`, `skip`, `snooze`, `mystats`, `mymeds`, and `report`.
    *   **Edge Case:** If no keyword is matched, the "extra" output can be used to send a "Help" message (though currently unconfigured in the provided JSON).

#### 2.3 Dose Logging (Taken & Skip)
Handles the outcome of a medication reminder.

*   **Nodes Involved:**
    *   `Google Sheets – Read Log (Taken/Skip)`
    *   `Process Taken/Skip (Code)`
    *   `Google Sheets – Update Taken/Skipped`
    *   `Send a text message (1 & 2)`
*   **Node Details:**
    *   **Process Taken (Code):** Calculates the patient's current "streak" (consecutive days of taking meds) and returns a congratulatory message.
    *   **Process Skip (Code):** Checks for consecutive skips. If $\ge$ 2 skips are detected, it flags `alertCaregiver = true`.
    *   **Update Nodes:** Uses the `logKey` to find the specific "Pending" row in Google Sheets and change its status to "Taken" or "Skipped".

#### 2.4 Monitoring & Alerts
Background checks to ensure patient safety when no response is received.

*   **Nodes Involved:**
    *   `Schedule Trigger – Every 30 Min / 1 Hour`
    *   `Check Snooze Queue / Detect Missed Doses`
*   **Node Details:**
    *   **Detect Missed Doses (Code):** Runs hourly. It looks for any log entries that remained "Pending" for more than 2 hours after being sent. It groups these by caregiver and sends a critical alert.
    *   **Check Snooze Queue (Code):** Resends reminders to patients who requested a 30-minute delay.

#### 2.5 Reporting
Provides transparency to both the patient and the caregiver.

*   **Nodes Involved:**
    *   `Build My Stats`
    *   `Build My Meds Schedule`
    *   `Build Caregiver Report`
*   **Node Details:**
    *   **Build My Stats (Code):** Generates a visual 7-day adherence report using ASCII bar charts (e.g., `████░░░░░░ 40%`).
    *   **Build Caregiver Report (Code):** Allows a caregiver to text `report <phone>` to receive the full adherence history of a specific patient.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger – 8AM 1PM 8PM | Schedule Trigger | Timing Trigger | None | GS – Read Medications | 1️⃣ Scheduled Reminder Dispatch |
| Google Sheets – Read Medications | Google Sheets | Data Fetching | Schedule Trigger | Filter Meds by Slot | 1️⃣ Scheduled Reminder Dispatch |
| Wati Trigger | WATI Trigger | Webhook Entry | None | Route Message | 2️⃣ Inbound Reply Routing |
| Route Message | Switch | Keyword Router | WATI Trigger | Various (Taken, Skip, etc.) | 2️⃣ Inbound Reply Routing |
| Process Taken | Code | Streak Calculation | GS – Read Log (Taken) | GS – Update Taken | 3️⃣ Dose Logging — Taken & Skip |
| Process Skip | Code | Alert Logic | GS – Read Log (Skip) | GS – Update Skipped | 3️⃣ Dose Logging — Taken & Skip |
| Process Snooze | Code | Delay Logic | Route Message | GS – Log Snooze | 4️⃣ Snooze & Missed Dose Alert |
| Detect Missed Doses | Code | Safety Check | GS – Read Log (Missed) | Send a text message8 | 4️⃣ Snooze & Missed Dose Alert |
| Build My Stats | Code | Visualization | GS – Read Log (Stats) | Send a text message4 | 5️⃣ Reports & Stats |
| Build Caregiver Report | Code | Caregiver Audit | GS – Read Log (Caregiver) | Send a text message6 | 5️⃣ Reports & Stats |

---

### 4. Reproducing the Workflow from Scratch

1.  **Database Preparation:**
    *   Create a Google Sheet with three tabs:
        *   `Medications`: Columns: `medId`, `patientName`, `phone`, `medName`, `dosage`, `instructions`, `frequency`, `timeSlots`, `status`, `caregiverPhone`.
        *   `AdherenceLog`: Columns: `date`, `slot`, `phone`, `logKey`, `sentAt`, `status`, `patientName`, `caregiverPhone`.
        *   `SnoozeQueue`: Columns: `phone`, `status`, `snoozedAt`, `followUpAt`.

2.  **Setup Scheduled Reminders:**
    *   Add a **Schedule Trigger** with a Cron expression `0 8,13,20 * * *`.
    *   Connect a **Google Sheets (Read)** node to fetch the `Medications` tab.
    *   Connect a **Code Node** to filter rows where `status == 'active'` and `timeSlots` match the current hour.
    *   Connect a **WATI (Send Text)** node using expressions to map the patient phone and the reminder message.

3.  **Setup Inbound Routing:**
    *   Add a **WATI Trigger** (Event: `messageReceived`).
    *   Connect a **Switch Node** with rules (String: Equals): `taken`, `skip`, `snooze`, `mystats`, `mymeds`. Add one (String: Starts With) for `report `.

4.  **Implement Logging Logic:**
    *   For the `taken` branch: Fetch logs from Google Sheets, use a **Code Node** to calculate the streak and find the current `Pending` record, then use **Google Sheets (Update)** to mark it "Taken".
    *   For the `skip` branch: Similar to taken, but include logic in the **Code Node** to count recent skips and trigger a caregiver alert if needed.

5.  **Implement Safety Alerts:**
    *   Add a **Schedule Trigger** (Every 1 Hour).
    *   Connect **Google Sheets (Read)** for `AdherenceLog`.
    *   Add a **Code Node** to filter items where `status == 'Pending'` and `sentAt < (now - 2 hours)`.
    *   Connect a **WATI (Send Text)** node directed to the `caregiverPhone`.

6.  **Credential Configuration:**
    *   **WATI:** Requires API Endpoint and Access Token.
    *   **Google Sheets:** Requires OAuth2 authentication with `https://www.googleapis.com/auth/spreadsheets` scope.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Workflow Capabilities** | 3x daily reminders, adherence streaks, caregiver escalation, and visual reports. |
| **Data Retention** | Adherence is calculated based on the `AdherenceLog` tab history. |
| **Language** | Messages are currently formatted for English speakers in the IST time zone (`en-IN`). |
| **Requirements** | Active WATI account with a verified WhatsApp Business API number. |