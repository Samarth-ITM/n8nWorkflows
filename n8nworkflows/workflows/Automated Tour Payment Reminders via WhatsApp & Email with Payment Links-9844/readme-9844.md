Automated Tour Payment Reminders via WhatsApp & Email with Payment Links

https://n8nworkflows.xyz/workflows/automated-tour-payment-reminders-via-whatsapp---email-with-payment-links-9844


# Automated Tour Payment Reminders via WhatsApp & Email with Payment Links

### 1. Workflow Overview

This workflow automates the process of sending payment reminders to travelers who have pending payments for their tours. It targets travel agencies aiming to streamline payment follow-up by delivering timely and personalized reminders via WhatsApp and Email. The workflow runs twice daily—once at 7 AM and once at 7 PM—to ensure reminders are sent 3 days before the payment due date.

Logical blocks include:

- **1.1 Morning Flow Trigger and Data Retrieval (7 AM)**: Scheduled trigger initiates reading pending payments from Microsoft Excel.
- **1.2 Morning Reminder Preparation and Sending via WhatsApp**: Filters records due soon, prepares WhatsApp messages, sends reminders, and updates status.
- **1.3 Evening Flow Trigger and Data Retrieval (7 PM)**: Scheduled trigger initiates reading pending payments again.
- **1.4 Evening Reminder Preparation and Sending via Email**: Filters due records, prepares email reminders, sends emails, and updates status.
- **1.5 Reminder Status Update**: Updates records in Excel to prevent duplicate reminders.
- **1.6 Supporting Sticky Notes**: Documentation nodes outlining features, description, purpose, and flow summaries.

---

### 2. Block-by-Block Analysis

#### 1.1 Morning Flow Trigger and Data Retrieval (7 AM)

- **Overview**: This block triggers at 7 AM daily to start the reminder process by reading all pending payments from the Excel workbook.
- **Nodes Involved**:
  - Daily Payment Check - 7 AM
  - Read Pending Travel Payment
  - Process Payment Reminders

- **Node Details**:

  - **Daily Payment Check - 7 AM**
    - Type: Schedule Trigger
    - Role: Initiates workflow daily at 7:00 AM.
    - Configuration: Trigger at hour 7.
    - Input: None (trigger node).
    - Output: Starts next node.
    - Edge cases: Missed trigger if n8n instance is down.
  
  - **Read Pending Travel Payment**
    - Type: Microsoft Excel node (worksheet read)
    - Role: Reads records with "Status" equal to "Pending" from a configured workbook and worksheet.
    - Configuration: Filter on column "Status" equals "Pending"; workbook and worksheet referenced by ID.
    - Credentials: Microsoft Excel OAuth2.
    - Input: Trigger output.
    - Output: List of pending payment records.
    - Edge cases: Excel API errors, authentication failure, empty dataset.
  
  - **Process Payment Reminders**
    - Type: Code node (JavaScript)
    - Role: Filters records to those whose tour date is within 3 days, generates payment link and calculates days remaining.
    - Configuration: Custom JS code that:
      - Gets current date.
      - Sets reminderDate = current date + 3 days.
      - Iterates over all records, filters those with tourDate ≤ reminderDate and tourDate > currentDate.
      - Constructs a payment link with travelerId, amountDue, bookingId.
      - Returns filtered records with calculated fields.
    - Input: Pending payments list.
    - Output: Filtered reminders list.
    - Edge cases: Date parsing errors, empty input, invalid data fields.

#### 1.2 Morning Reminder Preparation and Sending via WhatsApp

- **Overview**: Prepares personalized WhatsApp messages with payment details and sends them; finally updates reminder status.
- **Nodes Involved**:
  - Prepare WhatsApp Reminder
  - Send Message
  - Update Reminder Status

- **Node Details**:

  - **Prepare WhatsApp Reminder**
    - Type: Code node
    - Role: Formats the WhatsApp message with traveler/tour/payment details.
    - Configuration: JS code dynamically builds message string including travelerName, tourName, amountDue, tourDate, daysRemaining, paymentLink.
    - Input: Filtered reminders from previous code node.
    - Output: JSON containing phone, message, and key fields.
    - Edge cases: Missing phone number, malformed message.
  
  - **Send Message**
    - Type: WhatsApp node (API)
    - Role: Sends WhatsApp message to the traveler.
    - Configuration:
      - Operation: send
      - Text body from JSON message field
      - Recipient phone number from JSON phone field
      - PhoneNumberId configured (likely WhatsApp Business API number)
    - Credentials: WhatsApp API OAuth2.
    - Input: Prepared WhatsApp messages.
    - Output: API response.
    - Edge cases: API rate limits, invalid phone number, WhatsApp API downtime.
  
  - **Update Reminder Status**
    - Type: Microsoft Excel node (update)
    - Role: Updates the traveler’s record in Excel to mark reminder as sent.
    - Configuration:
      - Workbook and worksheet by ID.
      - Operation: Update.
      - Column to match on: "name" (likely traveler or booking identifier).
      - Data mode: autoMap (maps JSON fields to Excel columns).
    - Credentials: Microsoft Excel OAuth2.
    - Input: Output of Send Message (to ensure update after successful send).
    - Output: Update confirmation.
    - Edge cases: Update failures, concurrency issues.

#### 1.3 Evening Flow Trigger and Data Retrieval (7 PM)

- **Overview**: This block triggers at 7 PM daily to capture any new pending payments and begin the email reminder flow.
- **Nodes Involved**:
  - Daily Payment Check - 7 PM
  - Check Pending Payment
  - Create Payment Reminders

- **Node Details**:

  - **Daily Payment Check - 7 PM**
    - Type: Schedule Trigger
    - Role: Initiate workflow daily at 7 PM.
    - Configuration: Trigger at hour 19.
    - Edge cases: Similar to morning trigger.
  
  - **Check Pending Payment**
    - Type: Microsoft Excel node (read)
    - Role: Reads all records with "Status" = "Pending" (same as morning read).
    - Configuration: Same filtering on "Status".
    - Credentials: Microsoft Excel OAuth2.
    - Edge cases: Same as morning read.
  
  - **Create Payment Reminders**
    - Type: Code node
    - Role: Filters records for those with payments due in ≤3 days, similar filtering logic as morning.
    - Configuration: JS code identical in logic to "Process Payment Reminders" node.
    - Edge cases: Same as morning code node.

#### 1.4 Evening Reminder Preparation and Sending via Email

- **Overview**: Prepares customized email messages and sends payment reminders via email; updates status afterward.
- **Nodes Involved**:
  - Make Reminder For Email
  - Send Email Reminder
  - Update Status Of Reminder

- **Node Details**:

  - **Make Reminder For Email**
    - Type: Code node
    - Role: Formats an email reminder message with payment details.
    - Configuration: JS constructs the email body with travelerName, tourName, amountDue, tourDate, daysRemaining, and paymentLink.
    - Input: Filtered reminders from previous code node.
    - Output: JSON with email message and metadata.
    - Edge cases: Missing email address, malformed message.
  
  - **Send Email Reminder**
    - Type: Email Send node
    - Role: Sends the email reminder.
    - Configuration:
      - Subject and body injected dynamically from JSON.
      - ToEmail from JSON.
      - FromEmail statically set as "tour@gmail.com".
      - Email format: text.
    - Credentials: SMTP.
    - Edge cases: SMTP failures, invalid email, spam filtering.
  
  - **Update Status Of Reminder**
    - Type: Microsoft Excel node (update)
    - Role: Updates Excel record to mark email reminder sent.
    - Configuration: Matches on "name" column to update status.
    - Credentials: Microsoft Excel OAuth2.
    - Edge cases: Update failures.

#### 1.5 Reminder Status Update

- **Overview**: Both morning and evening flows end with updating the status in the Excel workbook to prevent duplicate reminders.
- **Nodes Involved**:
  - Update Reminder Status (morning)
  - Update Status Of Reminder (evening)

- **See node details above.**

#### 1.6 Supporting Sticky Notes

- **Overview**: Provides documentation inside the workflow for clarity on features, purpose, description, and the flows at 7 AM and 7 PM.
- **Nodes Involved**:
  - Sticky Note
  - Sticky Note1
  - Sticky Note2
  - Sticky Note3
  - Sticky Note4

- **Content Summary**:
  - Feature highlights: scheduled execution, due-date filtering, multi-channel notifications, secure payment links, reminder tracking.
  - Description: automation for notifying travelers with payment links via WhatsApp and Email twice daily.
  - Purpose: ensuring timely payment reminders to maintain cash flow and professionalism.
  - Detailed morning and evening flow steps.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                         | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                          |
|---------------------------|-------------------------|---------------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Daily Payment Check - 7 AM | Schedule Trigger        | Morning trigger at 7 AM                | -                          | Read Pending Travel Payment | Morning Flow (7 AM) detailed in Sticky Note3                                                       |
| Read Pending Travel Payment| Microsoft Excel         | Reads pending payments from Excel     | Daily Payment Check - 7 AM  | Process Payment Reminders  | Morning Flow (7 AM) detailed in Sticky Note3                                                       |
| Process Payment Reminders  | Code                    | Filters payments due within 3 days    | Read Pending Travel Payment | Prepare WhatsApp Reminder  | Morning Flow (7 AM) detailed in Sticky Note3                                                       |
| Prepare WhatsApp Reminder  | Code                    | Formats WhatsApp message content      | Process Payment Reminders   | Send Message              | Morning Flow (7 AM) detailed in Sticky Note3                                                       |
| Send Message              | WhatsApp                | Sends WhatsApp reminder messages      | Prepare WhatsApp Reminder   | Update Reminder Status    | Morning Flow (7 AM) detailed in Sticky Note3                                                       |
| Update Reminder Status     | Microsoft Excel         | Updates record status after WhatsApp  | Send Message               | -                        | Morning Flow (7 AM) detailed in Sticky Note3                                                       |
| Daily Payment Check - 7 PM | Schedule Trigger        | Evening trigger at 7 PM                | -                          | Check Pending Payment      | Evening Flow (7 PM) detailed in Sticky Note4                                                       |
| Check Pending Payment      | Microsoft Excel         | Reads pending payments from Excel     | Daily Payment Check - 7 PM  | Create Payment Reminders   | Evening Flow (7 PM) detailed in Sticky Note4                                                       |
| Create Payment Reminders   | Code                    | Filters payments due within 3 days    | Check Pending Payment       | Make Reminder For Email    | Evening Flow (7 PM) detailed in Sticky Note4                                                       |
| Make Reminder For Email    | Code                    | Formats email message content          | Create Payment Reminders    | Send Email Reminder        | Evening Flow (7 PM) detailed in Sticky Note4                                                       |
| Send Email Reminder        | Email Send              | Sends email reminders                  | Make Reminder For Email     | Update Status Of Reminder  | Evening Flow (7 PM) detailed in Sticky Note4                                                       |
| Update Status Of Reminder  | Microsoft Excel         | Updates record status after email      | Send Email Reminder         | -                        | Evening Flow (7 PM) detailed in Sticky Note4                                                       |
| Sticky Note                | Sticky Note             | Highlights key features                | -                          | -                        | ## 🚀 Key Features - Scheduled execution, multi-channel, payment links, reminder tracking          |
| Sticky Note1               | Sticky Note             | Provides workflow description          | -                          | -                        | Workflow description for travel agencies automating payment reminders                              |
| Sticky Note2               | Sticky Note             | Explains workflow purpose              | -                          | -                        | Purpose: ensure timely reminders to maintain cash flow and professionalism                         |
| Sticky Note3               | Sticky Note             | Explains morning flow in detail        | -                          | -                        | Morning Flow (7 AM) steps including reading, processing, sending WhatsApp reminders                |
| Sticky Note4               | Sticky Note             | Explains evening flow in detail        | -                          | -                        | Evening Flow (7 PM) steps including reading, processing, sending email reminders                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node (Morning)**
   - Name: Daily Payment Check - 7 AM
   - Type: Schedule Trigger
   - Configuration: Set to trigger daily at 7:00 AM.

2. **Create Microsoft Excel Read Node (Morning)**
   - Name: Read Pending Travel Payment
   - Type: Microsoft Excel
   - Operation: Read worksheet rows
   - Workbook: Select appropriate Excel workbook (by ID or name)
   - Worksheet: Select worksheet with payment records
   - Filter: Add filter where "Status" column equals "Pending"
   - Credentials: Connect Microsoft Excel OAuth2

3. **Create Code Node for Filtering (Morning)**
   - Name: Process Payment Reminders
   - Type: Code
   - Code: Implement logic to filter records with tourDate within 3 days, generate payment link, calculate days remaining (see node code description in 2.1)
   - Input: Incoming records from Excel read node.

4. **Create Code Node for WhatsApp Message Preparation**
   - Name: Prepare WhatsApp Reminder
   - Type: Code
   - Code: Compose WhatsApp message including travelerName, tourName, amountDue, tourDate, daysRemaining, paymentLink.
   - Input: Output from filtering code node.

5. **Create WhatsApp Node for Sending Message**
   - Name: Send Message
   - Type: WhatsApp
   - Operation: send
   - Parameters:
     - Text Body: Expression from the JSON message field.
     - Recipient Phone Number: Expression from JSON phone field.
     - PhoneNumberId: Set your WhatsApp Business phone number ID.
   - Credentials: Configure WhatsApp API OAuth2.

6. **Create Microsoft Excel Update Node (Morning)**
   - Name: Update Reminder Status
   - Type: Microsoft Excel
   - Operation: Update rows
   - Workbook and worksheet: Same as the read node.
   - Column to match on: "name" (or appropriate identifier)
   - Data Mode: Auto Map fields from input JSON.
   - Credentials: Microsoft Excel OAuth2
   - Connect this node after WhatsApp node.

7. **Create Schedule Trigger Node (Evening)**
   - Name: Daily Payment Check - 7 PM
   - Type: Schedule Trigger
   - Configuration: Trigger daily at 19:00.

8. **Create Microsoft Excel Read Node (Evening)**
   - Name: Check Pending Payment
   - Same configuration as the morning read node (filters "Status" = "Pending").

9. **Create Code Node for Filtering (Evening)**
   - Name: Create Payment Reminders
   - Same filtering logic as morning filtering node.

10. **Create Code Node for Email Message Preparation**
    - Name: Make Reminder For Email
    - Type: Code
    - Code: Compose email text with traveler and payment info, similar to WhatsApp message but formatted for email.

11. **Create Email Send Node**
    - Name: Send Email Reminder
    - Type: Email Send
    - Parameters:
      - Subject: Dynamic expression from JSON (e.g., "Tour Payment Reminder")
      - Text: Dynamic email body from JSON
      - To Email: From JSON email field
      - From Email: Static email (e.g., tour@gmail.com)
      - Email Format: Text
    - Credentials: Configure SMTP credentials for sending email.

12. **Create Microsoft Excel Update Node (Evening)**
    - Name: Update Status Of Reminder
    - Same configuration principle as morning update node.
    - Connect after email send node.

13. **Connect Nodes Appropriately**
    - Morning flow: Schedule Trigger → Read Pending Travel Payment → Process Payment Reminders → Prepare WhatsApp Reminder → Send Message → Update Reminder Status.
    - Evening flow: Schedule Trigger → Check Pending Payment → Create Payment Reminders → Make Reminder For Email → Send Email Reminder → Update Status Of Reminder.

14. **Add Sticky Notes (optional)**
    - Create sticky note nodes to document the workflow: features, description, purpose, and detailed flow explanations.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automates payment reminders for travel agencies via WhatsApp and Email, running twice daily at 7 AM and 7 PM. | General workflow information and purpose.                                                       |
| Uses Microsoft Excel as source and status tracking for payments.                                                           | Integration detail: Microsoft Excel API with OAuth2 credentials.                                 |
| WhatsApp messages require WhatsApp Business API credentials; email uses SMTP.                                               | Credential requirements for communication nodes.                                                |
| Payment links are dynamically generated URLs with travelerId, amount, and bookingId as query parameters.                   | Payment link construction logic embedded in code nodes.                                        |
| Reminder status updates prevent duplicate notifications for the same payment.                                              | Data consistency and idempotency mechanism.                                                    |
| Scheduled triggers rely on n8n instance uptime; consider external monitoring for reliability.                               | Operational note for deployment environments.                                                   |
| Sticky notes embedded in workflow provide detailed explanations and usage notes.                                            | Helpful for users modifying or auditing the workflow.                                          |

---

**Disclaimer:** The provided content is derived exclusively from an automated n8n workflow. It respects all applicable content policies and contains no illegal or offensive elements. All handled data is legal and publicly accessible.