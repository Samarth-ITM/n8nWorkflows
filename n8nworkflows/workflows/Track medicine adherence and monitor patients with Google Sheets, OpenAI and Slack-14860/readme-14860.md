Track medicine adherence and monitor patients with Google Sheets, OpenAI and Slack

https://n8nworkflows.xyz/workflows/track-medicine-adherence-and-monitor-patients-with-google-sheets--openai-and-slack-14860


# Track medicine adherence and monitor patients with Google Sheets, OpenAI and Slack

### 1. Workflow Overview

The **Patient Reminder System** is an automated healthcare monitoring solution designed to track medicine adherence and ensure patient safety. The workflow integrates a database (Google Sheets), an intelligence layer (OpenAI), and a communication layer (Slack) to create a closed-loop reminder and monitoring system.

The logic is divided into two primary operational streams:
- **Proactive Reminder Loop:** Periodically checks for scheduled medication times and notifies patients.
- **Reactive Monitoring Loop:** Processes patient responses via webhook, classifies the intent using AI, updates medical records, and escalates critical cases to healthcare providers.

#### Logical Blocks:
- **1.1 Reminder Scheduling:** Automates the trigger and retrieval of patient schedules.
- **1.2 Notification Delivery:** Validates the current time against the schedule and sends Slack alerts.
- **1.3 Response Intake & AI Analysis:** Captures patient replies and uses GPT-4 to determine adherence and medical urgency.
- **1.4 Patient Matching & Tracking:** Links the response to a specific patient record and calculates missed doses.
- **1.5 Risk Evaluation:** Determines if the patient's status is "Critical" based on AI flags or repeated missed doses.
- **1.6 Final Action & Feedback:** Updates the database and sends either a doctor alert or a patient-facing confirmation.

---

### 2. Block-by-Block Analysis

#### 2.1 Reminder Scheduling
- **Overview:** This block acts as the heartbeat of the system, ensuring patient records are checked every minute.
- **Nodes Involved:** `Trigger: Check Reminder Schedule`, `Fetch Patient Records`.
- **Node Details:**
    - **Trigger: Check Reminder Schedule** (Schedule Trigger): Set to run every minute.
    - **Fetch Patient Records** (Google Sheets): Reads the "Patients" sheet from a specific Document ID.
    - **Failure Types:** Google API quota limits or credential expiration.

#### 2.2 Notification Delivery
- **Overview:** Checks if any patient is due for their medication at the current moment.
- **Nodes Involved:** `Match Reminder Time`, `Send Medicine Reminder`.
- **Node Details:**
    - **Match Reminder Time** (If): Compares the `Reminder Time` column from the sheet with the current time using the expression `{{ $now.format('HH:mm') }}`.
    - **Send Medicine Reminder** (Slack): Sends a personalized message with options (Taken, Not Taken, Feeling Unwell).
    - **Edge Cases:** Timezone mismatches between the n8n server and the Google Sheet timestamps.

#### 2.3 Response Intake & AI Analysis
- **Overview:** Handles the incoming communication from patients and interprets the meaning of the text.
- **Nodes Involved:** `Receive Patient Response`, `AI Response Classification`, `Parse & Normalize AI Output`.
- **Node Details:**
    - **Receive Patient Response** (Webhook): Listens for POST requests containing `phone` and `message`.
    - **AI Response Classification** (OpenAI): Uses `gpt-4-turbo` to classify the message into `TAKEN`, `NOT_TAKEN`, `LATER`, or `CONFUSED`. It also determines if the case is `is_critical`.
    - **Parse & Normalize AI Output** (Code): A JavaScript node that uses `JSON.parse()` to convert the AI's string response into a structured n8n object. It includes a try-catch block to provide a "CONFUSED" fallback if the AI returns invalid JSON.

#### 2.4 Patient Matching & Tracking
- **Overview:** Identifies the specific patient associated with the phone number provided in the webhook.
- **Nodes Involved:** `Load Patient Data`, `Match Patient by Phone`, `Update Missed Count`.
- **Node Details:**
    - **Load Patient Data** (Google Sheets): Retrieves the full patient list to find the matching record.
    - **Match Patient by Phone** (If): Validates if the phone number from the webhook matches the phone number in the sheet.
    - **Update Missed Count** (Code): Logic to increment `Missed_Count` if classification is `NOT_TAKEN` or `LATER`, and reset it to `0` if `TAKEN`.

#### 2.5 Risk Evaluation
- **Overview:** Evaluates whether the patient needs immediate medical intervention.
- **Nodes Involved:** `Check Critical Condition`.
- **Node Details:**
    - **Check Critical Condition** (If): Triggers a "True" path if:
        1. The AI flagged the response as `is_critical = true`.
        2. OR the `Missed_Count` is greater than or equal to 3.

#### 2.6 Final Action & Feedback
- **Overview:** Performs the final database update and sends the appropriate notification to either the doctor or the patient.
- **Nodes Involved:** `Update Critical Status`, `Update Patient Record`, `Route Based on Response`, `Alert Doctor (Critical Case)`, `Notify Missed Dose`, `Reminder for Later`, `Acknowledge Compliance`.
- **Node Details:**
    - **Update Critical Status / Update Patient Record** (Google Sheets): Updates the sheet using the `Phone` as the matching column.
    - **Route Based on Response** (Switch): Routes the workflow to different Slack nodes based on the AI classification (`TAKEN` $\rightarrow$ Acknowledge; `NOT_TAKEN` $\rightarrow$ Notify Missed; `LATER` $\rightarrow$ Reminder for Later).
    - **Alert Doctor (Critical Case)** (Slack): High-priority notification containing the AI's `doctor_alert` and the patient's reason for criticality.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Trigger: Check Reminder Schedule | Schedule Trigger | Workflow Entry (Time) | - | Fetch Patient Records | Reminder Scheduler & Patient Fetch |
| Fetch Patient Records | Google Sheets | Data Retrieval | Trigger: Check... | Match Reminder Time | Reminder Scheduler & Patient Fetch |
| Match Reminder Time | If | Time Validation | Fetch Patient Records | Send Medicine Reminder | Reminder Time Validation |
| Send Medicine Reminder | Slack | Patient Notification | Match Reminder Time | - | Reminder Time Validation |
| Receive Patient Response | Webhook | Workflow Entry (Event) | - | AI Response Classification | Patient Response Intake & AI Processing |
| AI Response Classification | OpenAI | Intent Analysis | Receive Patient Response | Parse & Normalize... | Patient Response Intake & AI Processing |
| Parse & Normalize AI Output | Code | Data Structuring | AI Response Class... | Load Patient Data | Patient Response Intake & AI Processing |
| Load Patient Data | Google Sheets | Record Lookup | Parse & Normalize... | Match Patient by Phone | Patient Identification & Tracking |
| Match Patient by Phone | If | Identity Verification | Load Patient Data | Update Missed Count | Patient Identification & Tracking |
| Update Missed Count | Code | Metric Calculation | Match Patient by Phone | Check Critical Condition | Missed Dose Calculation |
| Check Critical Condition | If | Risk Assessment | Update Missed Count | Update Critical Status / Update Patient Record | Critical Condition Detection |
| Update Critical Status | Google Sheets | DB Update (Critical) | Check Critical Condition | Alert Doctor... | Data Update & Doctor Alert |
| Alert Doctor (Critical Case) | Slack | Emergency Notification | Update Critical Status | - | Data Update & Doctor Alert |
| Update Patient Record | Google Sheets | DB Update (Standard) | Check Critical Condition | Route Based on Response | Standard Update & Patient Feedback |
| Route Based on Response | Switch | Logic Branching | Update Patient Record | Notify Missed / Reminder Later / Acknowledge | Standard Update & Patient Feedback |
| Notify Missed Dose | Slack | Warning Message | Route Based on Response | - | Standard Update & Patient Feedback |
| Reminder for Later | Slack | Gentle Reminder | Route Based on Response | - | Standard Update & Patient Feedback |
| Acknowledge Compliance | Slack | Positive Reinforcement | Route Based on Response | - | Standard Update & Patient Feedback |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Database Setup
1. Create a Google Sheet with a tab named **Patients**.
2. Define columns: `Name`, `Phone`, `Medicine`, `Reminder Time` (format HH:mm), `Last Response`, `Last Patient Message`, `Status`, `Missed_Count`.

#### Step 2: The Reminder Loop
1. Add a **Schedule Trigger** $\rightarrow$ Set interval to "Minutes".
2. Add a **Google Sheets Node** $\rightarrow$ Operation: "Get Many" $\rightarrow$ Select the Patients sheet.
3. Add an **If Node** $\rightarrow$ Condition: `{{ $json['Reminder Time'] }}` equals `{{ $now.format('HH:mm') }}`.
4. Add a **Slack Node** $\rightarrow$ Connect to the "True" output $\rightarrow$ Configure a message template mentioning the patient's name and medicine.

#### Step 3: The Response Intake Loop
1. Add a **Webhook Node** $\rightarrow$ Set path to `patient-reply` $\rightarrow$ HTTP Method: `POST`.
2. Add an **OpenAI Node** $\rightarrow$ Resource: "Chat" $\rightarrow$ Model: `gpt-4-turbo`. 
   - Paste the system prompt provided in the JSON to ensure classification into JSON format (TAKEN, NOT_TAKEN, LATER, CONFUSED).
3. Add a **Code Node** $\rightarrow$ Use JavaScript to `JSON.parse()` the OpenAI output. Include a fallback object in case of parsing errors.

#### Step 4: Patient Matching & Metric Update
1. Add a **Google Sheets Node** $\rightarrow$ Operation: "Get Many" to fetch current records.
2. Add an **If Node** $\rightarrow$ Match the webhook `body.phone` with the sheet's `Phone` column.
3. Add a **Code Node** $\rightarrow$ Implement logic: 
   - If classification is `NOT_TAKEN` or `LATER`: `Missed_Count + 1`.
   - If classification is `TAKEN`: `Missed_Count = 0`.

#### Step 5: Criticality & Final Action
1. Add an **If Node** $\rightarrow$ Condition: `is_critical == true` OR `Missed_Count >= 3`.
2. **Critical Path (True):**
   - **Google Sheets Node** $\rightarrow$ Operation: "Update" $\rightarrow$ Set `Status` to "CRITICAL".
   - **Slack Node** $\rightarrow$ Send emergency alert to the Doctor's channel.
3. **Standard Path (False):**
   - **Google Sheets Node** $\rightarrow$ Operation: "Update" $\rightarrow$ Map `Status` and `Missed_Count`.
   - **Switch Node** $\rightarrow$ Route by `classification` value.
   - **Slack Nodes** $\rightarrow$ Create three separate nodes for "Missed Dose", "Later", and "Compliance" messages.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| High-level operational guide and setup steps | See "How Workflow Works + Setup" sticky note in workflow |
| AI Prompting | Strict JSON output is required for the "Parse & Normalize" node to function |
| Connectivity | Requires OAuth2 for Google Sheets and Bot Token for Slack |