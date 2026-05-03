Doctor Appointment Management System with Gemini AI, WhatsApp, Stripe & Google Sheets

https://n8nworkflows.xyz/workflows/doctor-appointment-management-system-with-gemini-ai--whatsapp--stripe---google-sheets-8825


# Doctor Appointment Management System with Gemini AI, WhatsApp, Stripe & Google Sheets

### 1. Workflow Overview

This workflow implements a comprehensive **Doctor Appointment Management System** integrating Google Sheets, WhatsApp Business Cloud, Stripe payment processing, and Google Gemini AI. It automates appointment bookings, reminders, rescheduling, cancellations, and payment handling through a WhatsApp conversational AI assistant powered by Google Gemini.

The workflow is logically divided into the following blocks:

- **1.1 WhatsApp AI Conversational Assistant**  
  Handles user interactions on WhatsApp for booking, rescheduling, cancelling appointments, and managing patient data using Google Gemini AI and simple memory.

- **1.2 Appointment Reminders**  
  Scheduled daily at 8 AM, this block retrieves appointments for the day, generates personalized reminder messages, and sends them via WhatsApp.

- **1.3 Payment Processing and Verification**  
  Manages Stripe payment sessions, generates payment links on new appointments, verifies successful payments via Stripe webhooks, updates appointment status, and sends payment confirmations.

- **1.4 Appointment Cancellation with Refund**  
  Detects appointment cancellations from Google Sheets, sends cancellation messages, verifies payment status, triggers Stripe refunds if applicable, and updates refund status in the sheet.

- **1.5 Google Sheets Data Management**  
  Reads and writes to Google Sheets for Patients, Appointments, and Configuration data supporting all above workflows.

---

### 2. Block-by-Block Analysis

#### 2.1 WhatsApp AI Conversational Assistant

**Overview:**  
This block manages all WhatsApp user interactions using an AI agent powered by Google Gemini. It processes incoming WhatsApp messages, maintains conversation context, and manipulates patient and appointment data in Google Sheets.

**Nodes Involved:**  
- WhatsApp Trigger  
- Simple Memory  
- AI Agent  
- Google Gemini Chat Model  
- Add Patient (Google Sheets)  
- Get Patient List For User (Google Sheets)  
- Add Appointment (Google Sheets)  
- Get User Appointments (Google Sheets)  
- Reschedule Appointment (Google Sheets)  
- Cancel Appointment (Google Sheets)  
- Date & Time

**Node Details:**

- **WhatsApp Trigger**  
  - *Type:* WhatsApp Trigger node  
  - *Role:* Entry point for incoming WhatsApp messages, triggering the workflow on new messages.  
  - *Config:* Listens for message updates with webhook ID configured.  
  - *Inputs:* Incoming WhatsApp messages with sender contact info.  
  - *Outputs:* Passes message data downstream for AI processing.  
  - *Failures:* Possible webhook connectivity or authorization issues.

- **Simple Memory**  
  - *Type:* Langchain memory buffer window  
  - *Role:* Maintains session context per user WhatsApp ID for conversational continuity.  
  - *Config:* Uses sessionKey as WhatsApp ID, context window length of 6 messages.  
  - *Inputs:* Messages from WhatsApp Trigger.  
  - *Outputs:* Attaches memory context to AI Agent input.  
  - *Failures:* Memory overflow or session key mismatch.

- **AI Agent**  
  - *Type:* Langchain Agent node with Google Gemini Chat Model  
  - *Role:* Core conversational AI implementing appointment logic, patient lookup, booking flows, rescheduling, cancellation, and payment options.  
  - *Config:* System prompt defines detailed user flows and conversational style (e.g. polite, WhatsApp-friendly, menu-driven).  
  - *Inputs:* User messages + memory context.  
  - *Outputs:* Response text which is sent back via WhatsApp node.  
  - *Failures:* AI response errors, rate limits, or prompt misconfiguration.

- **Google Gemini Chat Model**  
  - *Type:* Language model node  
  - *Role:* Provides LLM capabilities to AI Agent.  
  - *Config:* Default options, linked to AI Agent.  
  - *Failures:* API quota or auth errors.

- **Add Patient**  
  - *Type:* Google Sheets Tool (Append Operation)  
  - *Role:* Adds new patient records with fields: patient_id (generated timestamp), name, age, gender, whatsapp_number.  
  - *Config:* Pulls patient details from AI Agent variables and WhatsApp contact.  
  - *Inputs:* Data from AI Agent after patient info collection.  
  - *Outputs:* Confirmation to AI Agent.  
  - *Failures:* Google Sheets API errors, schema mismatch.

- **Get Patient List For User**  
  - *Type:* Google Sheets Tool (Read/List Operation)  
  - *Role:* Retrieves all patients linked to the WhatsApp user's number.  
  - *Config:* Filters patients by whatsapp_number column.  
  - *Inputs:* WhatsApp ID from AI Agent.  
  - *Outputs:* Patient list to AI Agent for selection.  
  - *Failures:* Sheet access issues, filter errors.

- **Add Appointment**  
  - *Type:* Google Sheets Tool (Append Operation)  
  - *Role:* Stores new appointment data including date, time, payment method, status, payment status, patient_id, whatsapp_number.  
  - *Config:* Uses AI Agent variables and WhatsApp ID, status defaults to "Confirmed", payment_status "Pending".  
  - *Failures:* API or data type conflicts.

- **Get User Appointments**  
  - *Type:* Google Sheets Tool (Filtered Read)  
  - *Role:* Fetches upcoming appointments related to the user by whatsapp_number.  
  - *Config:* Filter on whatsapp_number column equal to sender's WhatsApp ID.  
  - *Failures:* Sheet read errors.

- **Reschedule Appointment**  
  - *Type:* Google Sheets Tool (Update Operation)  
  - *Role:* Updates appointment row with new date and time based on AI Agent input.  
  - *Config:* Matches on appointment_id to update specific row.  
  - *Failures:* Concurrency or row matching issues.

- **Cancel Appointment**  
  - *Type:* Google Sheets Tool (Update Operation)  
  - *Role:* Marks appointment status as "Cancelled" for given appointment_id.  
  - *Config:* Matches on appointment_id.  
  - *Failures:* Same as above.

- **Date & Time**  
  - *Type:* Date Time Tool  
  - *Role:* Provides current date/time in Asia/Kolkata timezone for appointment scheduling logic and reminders.  
  - *Failures:* Timezone misconfiguration.

---

#### 2.2 Appointment Reminder Workflow

**Overview:**  
Scheduled daily at 8:00 AM, this block fetches all appointments for the current day and sends personalized WhatsApp reminders to patients.

**Nodes Involved:**  
- Schedule Trigger1  
- Get Appointment sheet1 (Google Sheets)  
- Date & Time2  
- Appointment Reminder AI Agent1  
- Google Gemini Chat Model3  
- Send message in WhatsApp Business Cloud

**Node Details:**

- **Schedule Trigger1**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers the reminder workflow once daily at 8:00 AM IST.  
  - *Failures:* Scheduling misfires.

- **Get Appointment sheet1**  
  - *Type:* Google Sheets Tool (Read)  
  - *Role:* Reads all appointment records to identify today's bookings.  
  - *Config:* Uses configured spreadsheet and tab IDs.  
  - *Failures:* Sheet access issues.

- **Date & Time2**  
  - *Type:* Date Time Tool  
  - *Role:* Provides current date/time for filtering today's appointments.  
  - *Config:* Asia/Kolkata timezone.  
  - *Failures:* As above.

- **Appointment Reminder AI Agent1**  
  - *Type:* Langchain Agent  
  - *Role:* Filters today's appointments and generates reminder messages.  
  - *Config:* System message instructs to send short WhatsApp reminders including name, date, and time.  
  - *Failures:* AI errors.

- **Google Gemini Chat Model3**  
  - *Type:* Language Model  
  - *Role:* Supports AI Agent1 with LLM capabilities.  
  - *Failures:* API quota or auth.

- **Send message in WhatsApp Business Cloud**  
  - *Type:* WhatsApp node  
  - *Role:* Sends reminder messages to patients.  
  - *Config:* Uses appointment data and configured phone number.  
  - *Failures:* WhatsApp API issues.

---

#### 2.3 Payment Processing and Verification

**Overview:**  
Handles Stripe payment integration: generates payment links for new appointments, receives payment success webhooks, updates status in Google Sheets, and notifies users.

**Nodes Involved:**  
- Look For New Appointment1 (Google Sheets Trigger)  
- Check Appointment Payment Mode1 (If)  
- Generate Stripe Payment Link1 (HTTP Request)  
- Send Payment Link1 (WhatsApp)  
- Stripe Trigger1 (Stripe Trigger)  
- Retrieve Payment Session1 (HTTP Request)  
- Mark Payment Paid1 (Google Sheets Update)  
- Send Payment Confirmation1 (WhatsApp)

**Node Details:**

- **Look For New Appointment1**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Detects newly added appointment rows to start payment processing.  
  - *Failures:* Polling delays or API errors.

- **Check Appointment Payment Mode1**  
  - *Type:* If node  
  - *Role:* Checks if the payment method is Stripe to decide payment link generation.  
  - *Failures:* Logic errors in matching payment method string.

- **Generate Stripe Payment Link1**  
  - *Type:* HTTP Request  
  - *Role:* Calls Stripe API to create a Checkout Session payment link with metadata including appointmentId and WhatsApp number.  
  - *Config:* Uses Stripe secret key, line item price set to $50.00 USD, success and cancel URLs redirect to WhatsApp link.  
  - *Failures:* HTTP errors, Stripe API limits, authorization failures.

- **Send Payment Link1**  
  - *Type:* WhatsApp node  
  - *Role:* Sends the Stripe payment link to the user via WhatsApp.  
  - *Failures:* WhatsApp API issues.

- **Stripe Trigger1**  
  - *Type:* Stripe Trigger  
  - *Role:* Receives Stripe webhook events for payment_intent.succeeded.  
  - *Failures:* Webhook misconfiguration, security issues.

- **Retrieve Payment Session1**  
  - *Type:* HTTP Request  
  - *Role:* Fetches checkout session details from Stripe to get metadata including appointmentId and WhatsApp number.  
  - *Failures:* API failures or mismatches.

- **Mark Payment Paid1**  
  - *Type:* Google Sheets Update  
  - *Role:* Updates appointment status to "Confirmed," payment_status to "Paid," and saves Stripe Payment Intent for future refund references.  
  - *Failures:* Sheet update errors.

- **Send Payment Confirmation1**  
  - *Type:* WhatsApp node  
  - *Role:* Sends payment confirmation message to user.  
  - *Failures:* WhatsApp API issues.

---

#### 2.4 Appointment Cancellation and Refund

**Overview:**  
Triggered by changes in Google Sheets appointment status to "Cancelled." Sends cancellation messages via WhatsApp, checks payment status, triggers Stripe refunds if paid, and updates refund status.

**Nodes Involved:**  
- Google Sheets Trigger1  
- Check status "Cancelled"1 (If)  
- Check Is Amount Paid1 (If)  
- Stripe Refund API1 (HTTP Request)  
- Send Cancellation Message (STRIPE) (WhatsApp)  
- Update Refund Status1 (Google Sheets)  
- Send Cancellation Message (CASH) (WhatsApp)

**Node Details:**

- **Google Sheets Trigger1**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Watches for row updates on "status" column to detect cancellations.  
  - *Failures:* Polling delays.

- **Check status "Cancelled"1**  
  - *Type:* If node  
  - *Role:* Checks if the updated status equals "Cancelled" (case-insensitive).  
  - *Failures:* Logic errors.

- **Check Is Amount Paid1**  
  - *Type:* If node  
  - *Role:* Checks if `stripe_payment_intent` exists (meaning payment was made online).  
  - *Failures:* Missing or malformed data.

- **Stripe Refund API1**  
  - *Type:* HTTP Request  
  - *Role:* Calls Stripe API to initiate refund for the associated payment intent.  
  - *Config:* Uses Stripe secret key and payment_intent from sheet data.  
  - *Failures:* Refund API errors, rate limits.

- **Send Cancellation Message (STRIPE)**  
  - *Type:* WhatsApp node  
  - *Role:* Notifies user that appointment is cancelled and refund initiated.  
  - *Failures:* WhatsApp API errors.

- **Update Refund Status1**  
  - *Type:* Google Sheets Update  
  - *Role:* Updates payment_status to "Refunded" for appointment.  
  - *Failures:* Sheet update errors.

- **Send Cancellation Message (CASH)**  
  - *Type:* WhatsApp node  
  - *Role:* Sends cancellation confirmation if payment was cash (no refund).  
  - *Failures:* WhatsApp API errors.

---

#### 2.5 Google Sheets Data Management

**Overview:**  
Manages patient and appointment data storage, retrieval, and updates in Google Sheets supporting all other blocks.

**Nodes Involved:**  
- Add Patient  
- Get Patient List For User  
- Add Appointment  
- Get User Appointments  
- Get All Appointments  
- Reschedule Appointment  
- Cancel Appointment  
- Config (Google Sheets for configuration parameters)

**Node Details:**

- All nodes use Google Sheets Tool or Google Sheets node type with operations like append, read, update.  
- Document IDs and Sheet Names must be configured for respective sheets: Patients, Appointments, Config.  
- Matching columns vary per operation (e.g. appointment_id for updates).  
- Failures include API access, schema mismatches, data concurrency.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                              | Input Node(s)                      | Output Node(s)                                | Sticky Note                                                                                         |
|--------------------------------|----------------------------------|----------------------------------------------|----------------------------------|----------------------------------------------|---------------------------------------------------------------------------------------------------|
| WhatsApp Trigger               | WhatsApp Trigger                 | Entry point for WhatsApp messages             | -                                | AI Agent                                     |                                                                                                   |
| Simple Memory                 | Langchain Memory                 | Maintains session conversational context     | WhatsApp Trigger                 | AI Agent                                     |                                                                                                   |
| AI Agent                     | Langchain Agent                  | Conversational AI assistant                    | WhatsApp Trigger, Simple Memory  | Send message, Google Sheets Nodes            | "Appointment Booking AI Assistant: 1. Create Booking 2. View Upcoming Appointment 3. Rescheduling 4. Cancellation" |
| Google Gemini Chat Model       | Language Model                   | Provides AI language model capabilities       | AI Agent                        | AI Agent                                     |                                                                                                   |
| Add Patient                   | Google Sheets Tool               | Adds new patient records                       | AI Agent                        | AI Agent                                     | "This nodes allows your agent create and get data from Google Sheets"                             |
| Get Patient List For User     | Google Sheets Tool               | Retrieves user’s patient list                  | AI Agent                        | AI Agent                                     |                                                                                                   |
| Add Appointment              | Google Sheets Tool               | Adds new appointment data                      | AI Agent                        | AI Agent                                     |                                                                                                   |
| Get User Appointments         | Google Sheets Tool               | Retrieves user’s upcoming appointments        | AI Agent                        | AI Agent                                     |                                                                                                   |
| Reschedule Appointment       | Google Sheets Tool               | Updates appointment date/time for rescheduling| AI Agent                        | AI Agent                                     |                                                                                                   |
| Cancel Appointment           | Google Sheets Tool               | Marks appointment as cancelled                 | AI Agent                        | AI Agent                                     |                                                                                                   |
| Date & Time                  | DateTime Tool                   | Provides current date/time                      | -                                | AI Agent                                     |                                                                                                   |
| Schedule Trigger1             | Schedule Trigger                | Triggers daily appointment reminder workflow  | -                                | Get Appointment sheet1                        | "Appointment Reminder Workflow: Scheduled daily at 8:00 AM"                                       |
| Get Appointment sheet1        | Google Sheets Tool               | Reads scheduled appointments                   | Schedule Trigger1               | Appointment Reminder AI Agent1                |                                                                                                   |
| Date & Time2                 | DateTime Tool                   | Provides current date/time                      | -                                | Appointment Reminder AI Agent1                |                                                                                                   |
| Appointment Reminder AI Agent1 | Langchain Agent                  | Generates personalized WhatsApp reminders     | Get Appointment sheet1, Date & Time2 | Send message in WhatsApp Business Cloud      |                                                                                                   |
| Google Gemini Chat Model3      | Language Model                   | Supports reminder AI agent                      | Appointment Reminder AI Agent1  | Appointment Reminder AI Agent1                |                                                                                                   |
| Send message in WhatsApp Business Cloud | WhatsApp node                 | Sends appointment reminders via WhatsApp      | Appointment Reminder AI Agent1  | -                                            |                                                                                                   |
| Look For New Appointment1     | Google Sheets Trigger           | Detects new appointments added                  | -                                | Check Appointment Payment Mode1               | "Payment Link Generation: Triggered on new appointment added in Sheet"                            |
| Check Appointment Payment Mode1 | If Node                        | Checks if payment method is Stripe             | Look For New Appointment1       | Generate Stripe Payment Link1                  |                                                                                                   |
| Generate Stripe Payment Link1 | HTTP Request                   | Creates Stripe Checkout session for payment    | Check Appointment Payment Mode1 | Send Payment Link1                             |                                                                                                   |
| Send Payment Link1            | WhatsApp node                   | Sends Stripe payment link via WhatsApp         | Generate Stripe Payment Link1   | -                                            |                                                                                                   |
| Stripe Trigger1               | Stripe Trigger                 | Receives Stripe payment success webhook        | -                                | Retrieve Payment Session1                      | "Payment Verification Workflow: Stripe webhook triggers on payment success"                      |
| Retrieve Payment Session1     | HTTP Request                   | Retrieves Stripe payment metadata               | Stripe Trigger1                | Mark Payment Paid1, Send Payment Confirmation1|                                                                                                  |
| Mark Payment Paid1            | Google Sheets Update           | Updates appointment status to Paid              | Retrieve Payment Session1       | Send Payment Confirmation1                     |                                                                                                   |
| Send Payment Confirmation1    | WhatsApp node                   | Sends payment confirmation via WhatsApp        | Retrieve Payment Session1       | -                                            |                                                                                                   |
| Google Sheets Trigger1        | Google Sheets Trigger          | Watches for appointment status updates          | -                                | Check status "Cancelled"1                       | "Cancel Booking Workflow: Triggered on sheet status update"                                      |
| Check status "Cancelled"1      | If Node                        | Checks if appointment status changed to Cancelled | Google Sheets Trigger1          | Check Is Amount Paid1                           |                                                                                                   |
| Check Is Amount Paid1          | If Node                        | Checks if payment was made (Stripe payment intent exists) | Check status "Cancelled"1       | Stripe Refund API1, Send Cancellation Message (CASH) |                                                                                           |
| Stripe Refund API1            | HTTP Request                   | Initiates Stripe refund for cancelled payment   | Check Is Amount Paid1           | Send Cancellation Message (STRIPE), Update Refund Status1 |                                                                                                   |
| Send Cancellation Message (STRIPE) | WhatsApp node                 | Notifies user of cancellation and refund        | Stripe Refund API1             | -                                            |                                                                                                   |
| Update Refund Status1         | Google Sheets Update           | Updates refund status in sheet                   | Stripe Refund API1             | -                                            |                                                                                                   |
| Send Cancellation Message (CASH) | WhatsApp node                 | Notifies user of cancellation for cash payment | Check Is Amount Paid1           | -                                            |                                                                                                   |
| Config                       | Google Sheets Tool               | Reads config parameters (e.g., working hours)  | -                                | AI Agent                                     |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up credentials:**  
   - Google Sheets API credentials with access to your Patients, Appointments, and Config sheets.  
   - WhatsApp Business Cloud API credentials.  
   - Stripe API secret key for payment and refund APIs.  
   - Google Gemini API credentials for AI processing.

2. **Create WhatsApp Trigger node:**  
   - Configure webhook for WhatsApp message updates.

3. **Add Simple Memory node:**  
   - Use WhatsApp sender ID as session key.  
   - Set context window length to 6.

4. **Add Google Gemini Chat Model node:**  
   - Configure with default options.

5. **Add AI Agent node:**  
   - Set system prompt to detailed appointment assistant logic (booking, rescheduling, cancellation, payments).  
   - Connect WhatsApp Trigger and Simple Memory as inputs.  
   - Link Google Gemini Chat Model as language model.

6. **Add Google Sheets nodes for Patients:**  
   - "Add Patient" node: Append new patient with fields patient_id (timestamp), whatsapp_number, name, age, gender.  
   - "Get Patient List For User" node: Read patients filtered by whatsapp_number.

7. **Add Google Sheets nodes for Appointments:**  
   - "Add Appointment" node: Append new appointment with fields appointment_id (timestamp), patient_id, whatsapp_number, date, time, payment_method, payment_status ("Pending"), status ("Confirmed").  
   - "Get User Appointments" node: Read appointments filtered by whatsapp_number.  
   - "Reschedule Appointment" node: Update appointment date/time by appointment_id.  
   - "Cancel Appointment" node: Update appointment status to "Cancelled" by appointment_id.  
   - "Get All Appointments" node: Read all appointments (used for availability checks).

8. **Add Date & Time nodes:**  
   - One for general use in Asia/Kolkata timezone.  
   - Another for appointment reminder workflow.

9. **Add Send Message node for WhatsApp:**  
   - Configure to send responses from AI Agent back to user.

10. **Appointment Reminder Workflow:**  
    - Create Schedule Trigger node firing daily at 8:00 AM.  
    - Add Google Sheets node to read all appointments.  
    - Add AI Agent node to filter today's appointments and generate reminders.  
    - Connect Google Gemini Chat Model.  
    - Add WhatsApp node to send reminder messages.

11. **Payment Workflow:**  
    - Add Google Sheets Trigger node to detect new appointments (rows added).  
    - Add If node to check if payment_method contains "Stripe".  
    - Add HTTP Request node to call Stripe API and create Checkout Session with appointment metadata.  
    - Add WhatsApp node to send payment link.  
    - Add Stripe Trigger node to handle payment_intent.succeeded webhook events.  
    - Add HTTP Request node to retrieve payment session details from Stripe.  
    - Add Google Sheets Update node to mark appointment as "Paid" and confirm payment.  
    - Add WhatsApp node to send payment confirmation.

12. **Cancellation and Refund Workflow:**  
    - Add Google Sheets Trigger node watching "status" column updates.  
    - Add If node checking if status equals "Cancelled".  
    - Add If node checking if stripe_payment_intent exists (payment made online).  
    - Add HTTP Request node to call Stripe refund API.  
    - Add WhatsApp nodes to send cancellation messages (different for cash and Stripe).  
    - Add Google Sheets Update node to mark payment_status as "Refunded".

13. **Configure Google Sheets nodes:**  
    - Replace all placeholder spreadsheet IDs and sheet/tab names with actual Google Sheets IDs.  
    - Define schemas properly for Patients, Appointments, and Config sheets.

14. **Test the entire workflow:**  
    - Send WhatsApp messages to test booking, rescheduling, cancellations.  
    - Verify Google Sheets data updates.  
    - Test Stripe payment link generation and payment confirmation.  
    - Test cancellation refund process.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| © 2025 Lucas Peyrin                                                                                            | Creator credit on sticky notes                  |
| "Appointment Booking AI Assistant" system prompt is highly detailed and governs AI behavior and conversation. | Adjust this prompt in AI Agent for customization|
| Setup Instructions: Replace placeholder IDs and numbers, add credentials for Google Sheets, WhatsApp, Stripe.  | Sticky Note "Setup Instructions"                |
| Video on building this workflow: https://youtube.com                                                           | Sticky Note with link for help and learning     |
| Project by GreatStack: https://greatstack.dev                                                                 | Sticky Note5                                     |
| Use numbers (1, 2, 3, ...) for user option replies for clarity and WhatsApp friendliness.                      | Style guide in AI Agent system prompt            |
| Timezone throughout is Asia/Kolkata; adjust as needed for your locale.                                         | Date & Time nodes configuration                   |

---

This document enables understanding, replication, and modification of the Doctor Appointment Management System workflow integrating advanced AI conversational capabilities with practical scheduling, payment, and notification automation.