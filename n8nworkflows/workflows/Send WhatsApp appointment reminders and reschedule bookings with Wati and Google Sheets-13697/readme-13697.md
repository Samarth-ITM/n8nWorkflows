Send WhatsApp appointment reminders and reschedule bookings with Wati and Google Sheets

https://n8nworkflows.xyz/workflows/send-whatsapp-appointment-reminders-and-reschedule-bookings-with-wati-and-google-sheets-13697


# Send WhatsApp appointment reminders and reschedule bookings with Wati and Google Sheets

This document provides a technical breakdown of the n8n workflow designed to automate medical appointment reminders and management via WhatsApp (WATI) and Google Sheets.

### 1. Workflow Overview
The workflow serves as an automated receptionist for medical clinics. It operates on two primary triggers: a scheduled daily dispatch to remind patients of upcoming appointments and a reactive webhook trigger that processes patient responses (confirmations, cancellations, and rescheduling requests).

**Logic Blocks:**
*   **1.1 Scheduled Reminder Dispatch:** Daily automation to fetch and notify patients of appointments within the next 24 hours.
*   **1.2 Inbound Routing:** A central switch that parses WhatsApp messages to trigger specific sub-flows based on keywords.
*   **1.3 Appointment Confirmation:** Logic to verify and mark appointments as "Confirmed" in the database.
*   **1.4 Cancellation & Notification:** Processes cancellations and alerts clinic administration.
*   **1.5 Rescheduling System:** Dynamic slot fetching, session management, and update logic for picking new appointment times.
*   **1.6 Status Inquiry:** Allows patients to retrieve their booking history and upcoming schedule.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Reminder Dispatch
Automatically contacts patients with upcoming appointments to reduce "no-show" rates.
*   **Nodes Involved:** `Schedule Trigger – 9AM Daily`, `Google Sheets – Read Appointments`, `Filter Upcoming Appointments`, `Build Reminder Message`, `Send a text message5`.
*   **Node Details:**
    *   **Google Sheets – Read Appointments:** Fetches the entire "Appointments" sheet.
    *   **Filter Upcoming Appointments (Code):** Uses JavaScript to filter rows where `appointmentDate` is today or tomorrow and `status` is "Pending" or "Confirmed".
    *   **Build Reminder Message (Code):** Formats a localized string (en-IN) with doctor details, clinic name, and interactive instructions (confirm/cancel/reschedule).
    *   **Edge Cases:** If no appointments match, the code returns a "No upcoming appointments" message, preventing the loop from firing errors.

#### 2.2 Inbound Routing
The entry point for all patient-initiated communication.
*   **Nodes Involved:** `WATI Trigger`, `Route Message`.
*   **Node Details:**
    *   **WATI Trigger:** Listens for the `messageReceived` event.
    *   **Route Message (Switch):** Uses regex and string normalization (`toLowerCase().trim()`) to route to one of five outputs: `confirm`, `cancel`, `reschedule`, `myappointment`, or `slot `.

#### 2.3 Confirm & Cancel Logic
Updates the system of record based on patient intent.
*   **Nodes Involved:** `Google Sheets – Read for Confirm/Cancel`, `Process Confirmation/Cancellation`, `Google Sheets – Update Status`, `Send a text message/1`.
*   **Node Details:**
    *   **Process Nodes (Code):** These nodes identify the *soonest* valid appointment for the specific phone number to ensure the status update is applied to the correct record.
    *   **Google Sheets (Update):** Uses the phone number as the matching key to update the `status` column.
    *   **Admin Notify:** (Mentioned in logic) The cancellation flow generates an `adminMessage` meant for the clinic manager.

#### 2.4 Rescheduling Flow
A multi-step process involving session state management.
*   **Nodes Involved:** `Google Sheets – Read Available Slots`, `Build Available Slots`, `Google Sheets – Save Reschedule Session`, `Process Slot Selection`, `Google Sheets – Update Rescheduled`.
*   **Node Details:**
    *   **Build Available Slots (Code):** Filters a separate "Available Slots" sheet for entries marked as "Available" that occur in the future. It generates a numbered list (1-8).
    *   **Save Reschedule Session:** Stores the list of options in a "RescheduleSessions" sheet with a timestamp. This allows the bot to remember what "Slot 2" meant when the patient replies later.
    *   **Process Slot Selection (Code):** Triggered by the keyword `slot `. It retrieves the latest session for that phone number, parses the user's number choice, and extracts the corresponding date/time.

#### 2.5 Status & Inquiry
Provides transparency to the patient regarding their records.
*   **Nodes Involved:** `Google Sheets – Read My Appointment`, `Build My Appointment Card`, `Send a text message4`.
*   **Node Details:**
    *   **Build My Appointment Card (Code):** Aggregates all records for a phone number, separating them into "Upcoming" and "Recent History" with status emojis (✅, ⏳, ❌).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger – 9AM Daily | Schedule Trigger | Periodic Trigger | None | Google Sheets – Read Appointments | 1️⃣ Scheduled Reminder Dispatch |
| Google Sheets – Read Appointments | Google Sheets | Data Retrieval | Schedule Trigger | Filter Upcoming Appointments | 1️⃣ Scheduled Reminder Dispatch |
| Filter Upcoming Appointments | Code | Data Filtering | Google Sheets – Read Appointments | Build Reminder Message | 1️⃣ Scheduled Reminder Dispatch |
| Build Reminder Message | Code | String Formatting | Filter Upcoming Appointments | Send a text message5 | 1️⃣ Scheduled Reminder Dispatch |
| Send a text message5 | WATI | Outbound Messaging | Build Reminder Message | None | 1️⃣ Scheduled Reminder Dispatch |
| Wati Trigger | WATI Trigger | Webhook Receiver | None | Route Message | 2️⃣ Inbound Routing |
| Route Message | Switch | Intent Classification | Wati Trigger | Multiple | 2️⃣ Inbound Routing |
| Google Sheets – Read for Confirm | Google Sheets | Data Retrieval | Route Message | Process Confirmation | 3️⃣ Confirm & Cancel |
| Process Confirmation | Code | Logic Processing | Google Sheets – Read for Confirm | Google Sheets – Update Status | 3️⃣ Confirm & Cancel |
| Google Sheets – Update Status Confirmed | Google Sheets | Data Update | Process Confirmation | Send a text message | 3️⃣ Confirm & Cancel |
| Google Sheets – Read for Cancel | Google Sheets | Data Retrieval | Route Message | Process Cancellation | 3️⃣ Confirm & Cancel |
| Process Cancellation | Code | Logic Processing | Google Sheets – Read for Cancel | Google Sheets – Update Status | 3️⃣ Confirm & Cancel |
| Google Sheets – Read Available Slots | Google Sheets | Data Retrieval | Route Message | Build Available Slots | 4️⃣ Reschedule Flow |
| Build Available Slots | Code | Slot Generation | Google Sheets – Read Available Slots | Google Sheets – Save Reschedule Session | 4️⃣ Reschedule Flow |
| Google Sheets – Save Reschedule Session | Google Sheets | State Management | Build Available Slots | Send a text message2 | 4️⃣ Reschedule Flow |
| Google Sheets – Read Reschedule Session | Google Sheets | State Retrieval | Route Message | Process Slot Selection | 4️⃣ Reschedule Flow |
| Process Slot Selection | Code | Slot Validation | Google Sheets – Read Reschedule Session | Google Sheets – Update Rescheduled | 4️⃣ Reschedule Flow |
| Google Sheets – Update Rescheduled | Google Sheets | Data Update | Process Slot Selection | Send a text message3 | 4️⃣ Reschedule Flow |
| Google Sheets – Read My Appointment | Google Sheets | Data Retrieval | Route Message | Build My Appointment Card | 5️⃣ My Appointment & Admin Notify |
| Build My Appointment Card | Code | Aggregation | Google Sheets – Read My Appointment | Send a text message4 | 5️⃣ My Appointment & Admin Notify |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create a Google Sheet with three tabs: `Appointments` (columns: appointmentId, phone, patientName, doctorName, appointmentDate, appointmentTime, status), `Available Slots` (date, time, status, doctorName), and `RescheduleSessions` (phone, slotsData, createdAt).
2.  **Connections:** Set up credentials for **WATI** (API Key/Endpoint) and **Google Sheets** (OAuth2).
3.  **Outbound Path:**
    *   Create a **Schedule Trigger** set to your desired reminder time.
    *   Add a **Google Sheets** node to read the `Appointments` tab.
    *   Add a **Code Node** to filter for dates matching `today` or `tomorrow`.
    *   Add a **WATI "Send Text Message"** node using expressions to map the patient's phone and the formatted message.
4.  **Inbound Path:**
    *   Create a **WATI Trigger** for "Message Received".
    *   Connect a **Switch Node** with 5 rules using "String contains" or "Equals" for: `confirm`, `cancel`, `reschedule`, `myappointment`, and `slot `.
5.  **Confirmation Logic:**
    *   Route the `confirm` branch to a **Google Sheets** node (Read).
    *   Use a **Code Node** to find the index of the first "Pending" appointment.
    *   Connect to a **Google Sheets** node (Update) to change status to "Confirmed".
6.  **Rescheduling Logic:**
    *   Route `reschedule` to read the `Available Slots` sheet.
    *   Use a **Code Node** to format the top 8 slots into a text list and stringify the JSON of these slots.
    *   Save this stringified JSON into the `RescheduleSessions` sheet indexed by the patient's phone number.
    *   Route `slot ` to read the `RescheduleSessions` sheet, match the user's number choice to the stored JSON, and update the main `Appointments` sheet with the new date/time.
7.  **Final Polish:** Add **WATI "Send Text Message"** nodes at the end of every branch to provide immediate feedback to the patient.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Core functionality includes rescheduling via temporary session storage in Sheets. | Technical Architecture |
| Requires WATI, OpenAI (if adding AI logic), Google Sheets, and Google Calendar. | Credentials Needed |
| Appointment date formatting uses 'en-IN' locale for DD/MM/YYYY consistency. | Formatting Note |
| Detailed configuration and setup instructions are managed via the WATI templates. | [WATI Documentation](https://www.wati.io/) |