Manage WhatsApp vehicle service reminders and bookings with WATI and Google Sheets

https://n8nworkflows.xyz/workflows/manage-whatsapp-vehicle-service-reminders-and-bookings-with-wati-and-google-sheets-13733


# Manage WhatsApp vehicle service reminders and bookings with WATI and Google Sheets

This document provides a technical analysis of the n8n workflow designed to manage vehicle service lifecycles, integrating **WATI** (WhatsApp Business API) with **Google Sheets** for data persistence.

---

### 1. Workflow Overview
The workflow serves as an automated service advisor. It proactively identifies vehicles due for maintenance based on time or mileage, manages interactive appointment bookings via WhatsApp, and provides users with their service history and current vehicle status.

**Logical Blocks:**
*   **1.1 Scheduled Service Reminders:** Daily check for vehicles nearing service thresholds.
*   **1.2 Appointment Reminders:** Daily check for appointments scheduled for the following day.
*   **1.3 Inbound Command Routing:** A central gateway for all incoming WhatsApp messages.
*   **1.4 Booking & Confirmation Engine:** Interactive multi-step flow to select and log service slots.
*   **1.5 Maintenance Tracking:** User-driven mileage updates and garage-driven service logging.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Service Reminder
*   **Overview:** Every morning at 9 AM, the system scans the "Vehicles" database to find cars needing service based on date or distance.
*   **Nodes Involved:** 
    *   `Schedule Trigger – 9AM Daily1`
    *   `Sheets – Read All Vehicles1`
    *   `Check Service Due1`
    *   `Sheets – Mark Reminder Sent1`
    *   `WATI – Send Service Reminder1`
*   **Node Details:**
    *   **Check Service Due1 (Code):** Evaluates two conditions: `nextServiceDate` $\le$ 7 days from now, OR `nextServiceMileage` - `currentMileage` $\le$ 500km. It excludes vehicles already marked as "Booked" or where a reminder was already sent.
    *   **Sheets – Mark Reminder Sent1:** Updates the "Vehicles" tab to set `reminderSent` to "Yes" to avoid duplicate alerts.
    *   **Edge Cases:** If no vehicles are due, the code returns a summary item to stop the execution flow.

#### 2.2 Day-Before Appointment Reminder
*   **Overview:** A dedicated routine at 8 AM to reduce "no-shows" for scheduled appointments.
*   **Nodes Involved:** 
    *   `Schedule Trigger – 8AM Daily`
    *   `Sheets – Read Appointments1`
    *   `Check Tomorrow Appointments1`
    *   `WATI – Send Appointment Reminder1`
*   **Node Details:**
    *   **Check Tomorrow Appointments1 (Code):** Filters rows where `appointmentDate` matches tomorrow's date and the status is "Confirmed".

#### 2.3 Inbound Routing & Command Center
*   **Overview:** The entry point for all user interactions. It parses the incoming text and directs the user to the specific functional path.
*   **Nodes Involved:** 
    *   `Wati Trigger`
    *   `Command Router1`
*   **Node Details:**
    *   **Command Router1 (Switch):** Uses string matching (equals/startsWith) to route to 7 possible outputs: `book`, `mileage`, `history`, `status`, `logservice`, `confirm`, and a fallback `help`.

#### 2.4 Booking & Confirmation Flow
*   **Overview:** Handles the "Choose your slot" logic.
*   **Nodes Involved:** `Sheets – Find Vehicle for Booking1`, `Parse Book & Generate Slots1`, `WATI – Send Available Slots1`, `Sheets – Find Vehicle for Confirm1`, `Handle Slot Confirmation1`, `Sheets – Log Appointment1`, `WATI – Confirm Appointment1`.
*   **Node Details:**
    *   **Parse Book & Generate Slots1 (Code):** Dynamically calculates the next 4 available slots (Mon-Fri, 10 AM and 2 PM) starting from the next day.
    *   **Handle Slot Confirmation1 (Code):** Maps the user's numeric input (e.g., "1") back to the specific date/time generated in the previous step and creates a unique `apptId`.

#### 2.5 Mileage & Service Logging
*   **Overview:** Allows users to update their odometer and staff to close out service records.
*   **Nodes Involved:** `Parse Mileage Update1`, `Sheets – Update Mileage1`, `Parse Log Service1`, `Sheets – Append Service Record1`.
*   **Node Details:**
    *   **Parse Mileage Update1 (Code):** Extracts the number from a "mileage 50000" message, calculates distance remaining to the next service, and determines if the vehicle is now "Overdue".

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Wati Trigger | watiTrigger | Inbound Webhook | None | Command Router1 | 3️⃣ Inbound Reply Routing: Wati Trigger receives all inbound WhatsApp messages... |
| Schedule Trigger – 9AM Daily1 | scheduleTrigger | Daily Timer | None | Sheets – Read All Vehicles1 | 1️⃣ Scheduled Service Reminder: Schedule Trigger – 9AM Daily fires every morning. |
| Sheets – Read All Vehicles1 | googleSheets | Fetch Database | Schedule Trigger | Check Service Due1 | 1️⃣ Scheduled Service Reminder: Sheets – Read All Vehicles fetches every row... |
| Check Service Due1 | code | Logic Filter | Sheets – Read All Vehicles1 | Sheets – Mark Reminder Sent1 | 1️⃣ Scheduled Service Reminder: Check Service Due Code applies two checks... |
| Command Router1 | switch | Logic Gate | Wati Trigger | Multiple (7 Paths) | 3️⃣ Inbound Reply Routing: Command Router Switch detects the reply keyword... |
| Parse Book & Generate Slots1 | code | Data Generation | Sheets – Find Vehicle... | WATI – Send Available... | 4️⃣ Book Appointment & Confirm Slot: Parse Book & Generate Slots Code finds... |
| Handle Slot Confirmation1 | code | Data Mapping | Sheets – Find Vehicle... | Sheets – Log Appointment1 | 4️⃣ Book Appointment & Confirm Slot: Handle Slot Confirmation Code maps... |
| Sheets – Update Mileage1 | googleSheets | Update Database | Parse Mileage Update1 | WATI – Send Mileage Ack1 | 5️⃣ Mileage Update: Sheets – Update Mileage updates currentMileage... |
| Sheets – Append Service Record1| googleSheets | Log Entry | Parse Log Service1 | WATI – Confirm Service... | 6️⃣ History · Status · Log Service: Parse Log Service Code parses... |
| WATI – Send Help1 | wati | Fallback Message | Command Router1 | None | 6️⃣ History · Status · Log Service: WATI – Send Help delivers the command menu... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Database Preparation (Google Sheets):**
    *   Create a Sheet with 3 Tabs:
        *   **Vehicles:** Columns: `phone`, `ownerName`, `vehicleReg`, `make`, `model`, `currentMileage`, `nextServiceMileage`, `nextServiceDate`, `serviceIntervalKm`, `status`, `reminderSent`.
        *   **Appointments:** Columns: `phone`, `apptId`, `vehicleReg`, `serviceType`, `appointmentDate`, `timeSlot`, `status`, `bookedAt`.
        *   **ServiceHistory:** Columns: `historyId`, `phone`, `vehicleReg`, `serviceDate`, `mileageAtService`, `serviceType`, `workDone`, `cost`.

2.  **Inbound Configuration:**
    *   Create a **WATI Trigger** (Event: `messageReceived`).
    *   Connect a **Switch Node** with rules for: `book` (Equals), `mileage ` (StartsWith), `history` (Equals), `status` (Equals), `logservice ` (StartsWith), and `confirm` (StartsWith).

3.  **Scheduled Reminder Path:**
    *   Create a **Schedule Trigger** (9 AM).
    *   Add a **Google Sheets (Read)** node for the "Vehicles" tab.
    *   Add a **Code Node** to check if `Date.now()` is near `nextServiceDate` or if `currentMileage` is near `nextServiceMileage`.
    *   Add a **Google Sheets (Update)** node to set `reminderSent = Yes`.
    *   Add a **WATI (Send Message)** node for the alert.

4.  **Booking Logic:**
    *   Add a **Google Sheets (Read)** node to fetch the specific vehicle by `waId`.
    *   Add a **Code Node** to generate a text list of 4 future dates (skipping weekends).
    *   Add a **WATI (Send Message)** node displaying the slots.
    *   Add a **Confirmation Code Node** that takes the `confirm <n>` message, calculates the chosen date, and outputs an object for the **Google Sheets (Append)** node.

5.  **Mileage Path:**
    *   Add a **Code Node** to parse `parseFloat(text.split(' ')[1])`.
    *   Add a **Google Sheets (Update)** node to save the new `currentMileage` and recalculated `nextServiceMileage`.

6.  **Staff Logging Path:**
    *   Configure the `logservice` path to parse: `<reg> <km> <work> <cost>`.
    *   Use **Google Sheets (Append)** to save to the "ServiceHistory" tab.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Credential Setup** | Requires WATI API Key and Google Sheets OAuth2 credentials. |
| **Sheet ID Placeholder** | Replace `1eTlVJbSiz8P8PpjaqXgkRsZF8oeBTH_4kDtC5oaoTpI` with your unique ID. |
| **Localization** | Date formatting in code nodes is currently set to `en-IN` (Indian Standard). |
| **Slot Logic** | The booking engine assumes 2 slots per day (10:00 AM and 02:00 PM). |