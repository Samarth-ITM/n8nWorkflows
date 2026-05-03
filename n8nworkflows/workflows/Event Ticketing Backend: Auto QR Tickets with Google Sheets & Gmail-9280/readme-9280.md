Event Ticketing Backend: Auto QR Tickets with Google Sheets & Gmail

https://n8nworkflows.xyz/workflows/event-ticketing-backend--auto-qr-tickets-with-google-sheets---gmail-9280


# Event Ticketing Backend: Auto QR Tickets with Google Sheets & Gmail

### 1. Workflow Overview

This workflow, titled **"Event Ticketing Backend: Auto QR Tickets with Google Sheets & Gmail"**, orchestrates a complete backend system for event ticketing management. It handles participant registration, ticket generation with QR codes, ticket email delivery, and ticket validation at event check-in. The system integrates Google Sheets as its data store (for participant info and ticket records), Gmail for sending emails, and an external QR code API for generating QR images.

The workflow is logically divided into three main functional blocks:

- **1.1 Registration & Validation Block**: Accepts participant registration via a webhook, validates input, checks for duplicate emails, and stores registration data in Google Sheets.
- **1.2 Ticket Generation & Email Delivery Block**: Runs periodically to find paid registrations without sent tickets, generates unique ticket IDs and QR codes, builds HTML emails with embedded QR codes, sends emails via Gmail, and updates the Google Sheets data accordingly.
- **1.3 Ticket Check-in Scanner Block**: Provides a webhook endpoint to receive scanned QR data, parse it, verify ticket validity and check-in status, update ticket status if valid, and respond with success or error messages.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Registration & Validation Block

**Overview:**  
This block manages new participant registrations via a webhook. It validates the input data, checks for existing registrations by email, stores new valid registrations in Google Sheets, and responds with appropriate success or error messages.

**Nodes Involved:**  
- REGISTER (Webhook)  
- Validate Input (Code)  
- Valid Input? (If)  
- Get Participant (Google Sheets)  
- Email exist? (If)  
- Already Registered (Respond to Webhook)  
- Store Data (Google Sheets)  
- Tiket Booked (Respond to Webhook)  
- Validation Error (Respond to Webhook)  

**Node Details:**

- **REGISTER**  
  - Type: Webhook (POST /v1/register)  
  - Role: Entry point for registration requests with participant data in JSON (name, email, phone, ticket count, payment method, etc.)  
  - Input: HTTP POST JSON body  
  - Output: Passes data to Validate Input node  
  - Edge cases: Malformed JSON input, missing fields  

- **Validate Input**  
  - Type: Code (JavaScript)  
  - Role: Validates participant fields (name non-empty, email format, phone length, ticket count ≥ 1, payment method present)  
  - Key Expressions: Checks fields and accumulates error messages  
  - Output: Returns valid flag and either transformed data or errors  
  - Edge cases: Missing or invalid fields triggers validation error response  

- **Valid Input?**  
  - Type: If  
  - Role: Branches based on validation result  
  - Conditions: validation_error === false  
  - Output: True path proceeds to check for existing email; False path to Validation Error response  

- **Get Participant**  
  - Type: Google Sheets (read)  
  - Role: Searches “Register” sheet for existing participant by email  
  - Config: Filters by Email column matching input email  
  - Output: Participant data or empty if not found  
  - Edge cases: Google Sheets connectivity/auth errors, no matching email  

- **Email exist?**  
  - Type: If  
  - Role: Branches based on whether participant already exists (row_number defined)  
  - Output: True path to Already Registered response; False path to Store Data  

- **Already Registered**  
  - Type: Respond to Webhook  
  - Role: Sends JSON error for duplicate email with existing participant details  
  - Output: HTTP response to API client  

- **Store Data**  
  - Type: Google Sheets (append)  
  - Role: Appends validated new participant data to “Register” sheet with initial payment status "PENDING" and Email Sent "NO"  
  - Output: Passes data to Tiket Booked node  

- **Tiket Booked**  
  - Type: Respond to Webhook  
  - Role: Sends success JSON confirming registration and payment pending status  
  - Output: HTTP response to API client  

- **Validation Error**  
  - Type: Respond to Webhook  
  - Role: Sends JSON error with list of validation errors  
  - Output: HTTP response to API client  

---

#### 2.2 Ticket Generation & Email Delivery Block

**Overview:**  
A scheduled trigger runs every minute to process paid registrations with tickets not yet sent. It generates unique ticket IDs and QR codes for each ticket, builds a styled HTML email embedding the QR code, sends the email via Gmail, updates the “Tickets” and “Register” sheets, and marks tickets as emailed.

**Nodes Involved:**  
- START (Schedule Trigger)  
- Get Rows (Google Sheets)  
- Filter Paid Not Sent (Filter)  
- Generate Ticket Data (Code)  
- Generate QR Code (HTTP Request)  
- Build HTML Email (Code)  
- Send Email (Gmail)  
- Parse Data (Code)  
- Update Sheet (Tickets) (Google Sheets)  
- Update Sheet (Register) (Google Sheets)  

**Node Details:**

- **START**  
  - Type: Schedule Trigger (1-minute interval)  
  - Role: Initiates ticket generation process periodically  

- **Get Rows**  
  - Type: Google Sheets (read)  
  - Role: Fetches all rows from “Register” sheet  
  - Output: List of participants and payment data  

- **Filter Paid Not Sent**  
  - Type: Filter  
  - Role: Filters registrations where Payment Status = "PAID" and Email Sent = "NO"  
  - Output: Only participants ready for ticket generation  

- **Generate Ticket Data**  
  - Type: Code  
  - Role: For each participant, generates unique ticket IDs for each ticket requested, builds JSON data for QR code, and prepares metadata  
  - Key Logic:  
    - Ticket ID format: `TL-YYYYMMDD-RowNum-TicketNum-HASH`  
    - QR data JSON includes event info, ticket_id, name, email, ticket number, total tickets, timestamp, and hash  
  - Output: Array of ticket JSON objects with QR data  

- **Generate QR Code**  
  - Type: HTTP Request  
  - Role: Calls external QR code API (https://api.qrserver.com) with encoded QR data to generate PNG image file  
  - Output: Binary PNG file of QR code  

- **Build HTML Email**  
  - Type: Code  
  - Role: Constructs a detailed, styled HTML email for each ticket embedding the base64 QR code image and ticket info  
  - Template includes: Event name, greeting, ticket number, QR code image, ticket details, important notes, and footer  
  - Output: JSON with html_content and base64 QR code  

- **Send Email (Gmail)**  
  - Type: Gmail node  
  - Role: Sends the ticket email to participant’s email address with subject “Tiketmu Telah Tersedia!” and HTML body  
  - Credential: Gmail OAuth2 (must be configured with appropriate scope)  
  - Edge cases: Gmail API rate limits, auth errors, invalid email addresses  

- **Parse Data**  
  - Type: Code  
  - Role: Groups tickets by Email to combine all ticket IDs for update in Register sheet  
  - Output: JSON objects with Email and combined ticket_ids string  

- **Update Sheet (Tickets)**  
  - Type: Google Sheets (append)  
  - Role: Appends individual ticket records (ticket ID, ticket number, total tickets, participant info) to “Tickets” sheet  

- **Update Sheet (Register)**  
  - Type: Google Sheets (update)  
  - Role: Updates participant’s row in “Register” sheet to mark Email Sent = "YES" and store combined ticket IDs  
  - Matching Column: Email  

---

#### 2.3 Ticket Check-in Scanner Block

**Overview:**  
This block exposes a webhook endpoint for scanning tickets at event check-in. It parses the scanned QR code data, verifies ticket existence and check-in status, updates the ticket as checked in if valid, and responds with appropriate success or error messages.

**Nodes Involved:**  
- SCAN TICKET (Webhook)  
- Parse Barcode (Code)  
- Get Tickets (Google Sheets)  
- Ticket Available? (If)  
- Update Ticket Status (Google Sheets)  
- Checked IN (Respond to Webhook)  
- Parse Output (Code)  
- Already Checked IN (Respond to Webhook)  

**Node Details:**

- **SCAN TICKET**  
  - Type: Webhook (POST /v1/scanner)  
  - Role: Receives scanned QR code data from client scanner app or device  
  - Input: JSON containing a JSON-string barcode and scannedAt timestamp  

- **Parse Barcode**  
  - Type: Code  
  - Role: Parses JSON string in barcode field into ticket details (ticket_id, event, name, email, ticket_number, total_tickets) along with scannedAt time  
  - Error Handling: Returns failure response if barcode JSON is invalid  

- **Get Tickets**  
  - Type: Google Sheets (read)  
  - Role: Searches “Tickets” sheet by Ticket ID from parsed barcode  
  - Output: Matching ticket data if found  

- **Ticket Available?**  
  - Type: If  
  - Role: Checks if ticket exists and is not already checked in (Checked In != "YES")  
  - Conditions:  
    - Number of results > 0  
    - Checked In field not "YES"  
  - Output: True path updates ticket status; False path returns error response  

- **Update Ticket Status**  
  - Type: Google Sheets (update)  
  - Role: Marks the ticket as Checked In = "YES" and sets Checkin Time to scannedAt timestamp formatted as locale string  

- **Checked IN**  
  - Type: Respond to Webhook  
  - Role: Returns success JSON with ticket details and check-in time after update  

- **Parse Output**  
  - Type: Code  
  - Role: Prepares error JSON for invalid or already checked-in tickets  
  - Logic:  
    - If ticket exists but Checked In = YES, returns ALREADY_CHECKED_IN error with previous check-in time  
    - Otherwise, returns INVALID_TICKET error  

- **Already Checked IN**  
  - Type: Respond to Webhook  
  - Role: Returns error JSON for tickets already checked in  

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                              | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                                  |
|-----------------------|----------------------|----------------------------------------------|-------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| START                 | Schedule Trigger     | Triggers ticket generation every minute      |                         | Get Rows                        | # Automatic Ticket Generation (every 1 minute)                                                               |
| Get Rows              | Google Sheets        | Reads all registrations from "Register" sheet | START                   | Filter Paid Not Sent             |                                                                                                              |
| Filter Paid Not Sent  | Filter               | Filters paid registrations with Email Sent=NO | Get Rows                | Generate Ticket Data             |                                                                                                              |
| Generate Ticket Data  | Code                 | Generates unique ticket IDs and QR data       | Filter Paid Not Sent      | Generate QR Code                | # Ticket Generator & Sender                                                                                   |
| Generate QR Code      | HTTP Request         | Calls QR code API to generate QR image        | Generate Ticket Data      | Build HTML Email                |                                                                                                              |
| Build HTML Email      | Code                 | Builds HTML email content with embedded QR    | Generate QR Code          | Send Email (Gmail), Parse Data, Update Sheet (Tickets) |                                                                                                              |
| Send Email (Gmail)    | Gmail                | Sends ticket email to participant              | Build HTML Email          | Parse Data                     | # Setup Required: Gmail OAuth2 must be configured with Gmail API enabled                                      |
| Parse Data            | Code                 | Groups ticket IDs by Email for updating Register | Send Email (Gmail)       | Update Sheet (Register)         |                                                                                                              |
| Update Sheet (Tickets)| Google Sheets        | Appends individual ticket records to "Tickets" | Build HTML Email          |                                 |                                                                                                              |
| Update Sheet (Register)| Google Sheets        | Updates registration row with Email Sent=YES and ticket IDs | Parse Data                |                                 |                                                                                                              |
| REGISTER              | Webhook              | Receives participant registration data        |                         | Validate Input                  | # Registration Endpoint: POST /v1/register                                                                     |
| Validate Input        | Code                 | Validates input data fields                     | REGISTER                 | Valid Input?                   |                                                                                                              |
| Valid Input?          | If                   | Branches on validation success/failure         | Validate Input           | Get Participant, Validation Error |                                                                                                              |
| Get Participant       | Google Sheets        | Looks up participant by email                   | Valid Input?             | Email exist?                   |                                                                                                              |
| Email exist?          | If                   | Checks if email already registered              | Get Participant          | Already Registered, Store Data  |                                                                                                              |
| Already Registered    | Respond to Webhook   | Responds if email already registered             | Email exist? (true)      |                                 |                                                                                                              |
| Store Data            | Google Sheets        | Appends new registration data                    | Email exist? (false)     | Tiket Booked                   |                                                                                                              |
| Tiket Booked          | Respond to Webhook   | Responds with success confirmation               | Store Data               |                                 |                                                                                                              |
| Validation Error      | Respond to Webhook   | Responds with validation error messages          | Valid Input? (false)     |                                 |                                                                                                              |
| SCAN TICKET           | Webhook              | Receives scanned ticket QR code                  |                         | Parse Barcode                  | # How Ticket Scanner Works: POST /v1/scanner endpoint                                                         |
| Parse Barcode         | Code                 | Parses QR code JSON to extract ticket info      | SCAN TICKET              | Get Tickets                   |                                                                                                              |
| Get Tickets           | Google Sheets        | Looks up ticket by Ticket ID                      | Parse Barcode            | Ticket Available?              |                                                                                                              |
| Ticket Available?     | If                   | Checks ticket existence and check-in status      | Get Tickets              | Update Ticket Status, Parse Output |                                                                                                              |
| Update Ticket Status  | Google Sheets        | Marks ticket as checked in with time             | Ticket Available? (true) | Checked IN                    |                                                                                                              |
| Checked IN            | Respond to Webhook   | Responds with successful check-in JSON           | Update Ticket Status     |                                 |                                                                                                              |
| Parse Output          | Code                 | Prepares error message for invalid/already checked-in tickets | Ticket Available? (false) | Already Checked IN            |                                                                                                              |
| Already Checked IN    | Respond to Webhook   | Responds with already checked-in error           | Parse Output             |                                 |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node `REGISTER`**  
   - Set HTTP Method to POST, path `/v1/register`  
   - Response Mode: Response Node  

2. **Add a Code Node `Validate Input`** connected to `REGISTER`  
   - JavaScript validation for fields: `nama`, `email` (contains '@'), `no_hp` (min length 10), `jumlah_tiket` (≥1), `payment_method` present  
   - Returns `valid: true` with transformed data or `valid: false` with list of errors  

3. **Add an If Node `Valid Input?`** connected to `Validate Input`  
   - Condition: `validation_error === false`  

4. **Add a Google Sheets Node `Get Participant`** connected to `Valid Input?` (true path)  
   - Operation: Read rows from "Register" sheet  
   - Filter: `Email` column equals input email  

5. **Add an If Node `Email exist?`** connected to `Get Participant`  
   - Condition: `row_number !== undefined` (checks if participant exists)  

6. **Add Respond to Webhook Node `Already Registered`** connected to `Email exist?` (true)  
   - Return JSON error for duplicate email  

7. **Add a Google Sheets Node `Store Data`** connected to `Email exist?` (false)  
   - Operation: Append new participant data to "Register" sheet  
   - Include fields: Name, Email, Phone, Ticket Count, Payment Method, Payment Status = "PENDING", Email Sent = "NO"  

8. **Add Respond to Webhook Node `Tiket Booked`** connected to `Store Data`  
   - Return success JSON confirming registration  

9. **Add Respond to Webhook Node `Validation Error`** connected to `Valid Input?` (false)  
   - Return JSON with validation errors  

---

10. **Create a Schedule Trigger Node `START`**  
    - Interval: every 1 minute  

11. **Add Google Sheets Node `Get Rows`** connected to `START`  
    - Read all rows from "Register" sheet  

12. **Add Filter Node `Filter Paid Not Sent`** connected to `Get Rows`  
    - Condition: `Payment Status` equals "PAID" AND `Email Sent` equals "NO"  

13. **Add Code Node `Generate Ticket Data`** connected to `Filter Paid Not Sent`  
    - For each participant, generate ticket IDs for each ticket requested  
    - Format: `TL-YYYYMMDD-RowNum-TicketNum-HASH`  
    - Prepare QR data JSON string for each ticket  

14. **Add HTTP Request Node `Generate QR Code`** connected to `Generate Ticket Data`  
    - GET request to `https://api.qrserver.com/v1/create-qr-code/`  
    - Query params: size=300x300, data=URL encoded QR data string  
    - Response: File (PNG image)  

15. **Add Code Node `Build HTML Email`** connected to `Generate QR Code`  
    - Build styled HTML email embedding base64 QR code image and ticket details  

16. **Add Gmail Node `Send Email (Gmail)`** connected to `Build HTML Email`  
    - Send To: participant email  
    - Subject: "Tiketmu Telah Tersedia!"  
    - Message: HTML content from previous node  
    - Credential: Gmail OAuth2 configured with appropriate scopes  

17. **Add Code Node `Parse Data`** connected to `Send Email (Gmail)`  
    - Group tickets by email to combine ticket IDs for update  

18. **Add Google Sheets Node `Update Sheet (Register)`** connected to `Parse Data`  
    - Update participant row in "Register" sheet  
    - Set `Email Sent` to "YES" and `Ticked ID` to combined ticket IDs  
    - Match row by Email  

19. **Add Google Sheets Node `Update Sheet (Tickets)`** connected to `Build HTML Email`  
    - Append rows to "Tickets" sheet for each ticket  
    - Include Ticket ID, Ticket Number, Total Tickets, participant info  

---

20. **Create a Webhook Node `SCAN TICKET`**  
    - POST method path `/v1/scanner`  
    - Response Mode: Response Node  

21. **Add Code Node `Parse Barcode`** connected to `SCAN TICKET`  
    - Parse JSON string from body.barcode field to extract ticket_id, event, name, email, ticket_number, total_tickets, scannedAt  

22. **Add Google Sheets Node `Get Tickets`** connected to `Parse Barcode`  
    - Read rows from "Tickets" sheet filtered by `Ticket ID` equal to parsed ticket_id  

23. **Add If Node `Ticket Available?`** connected to `Get Tickets`  
    - Conditions:  
      - Number of matching rows > 0  
      - Checked In != "YES"  

24. **Add Google Sheets Node `Update Ticket Status`** connected to `Ticket Available?` (true)  
    - Update ticket row: set `Checked In` = "YES", `Checkin TIme` = scannedAt (formatted)  

25. **Add Respond to Webhook Node `Checked IN`** connected to `Update Ticket Status`  
    - Send JSON success response with ticket info and check-in time  

26. **Add Code Node `Parse Output`** connected to `Ticket Available?` (false)  
    - If ticket exists but already checked in, prepare ALREADY_CHECKED_IN error with previous check-in time  
    - Else prepare INVALID_TICKET error  

27. **Add Respond to Webhook Node `Already Checked IN`** connected to `Parse Output`  
    - Send error response JSON for already checked in or invalid tickets  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow manages the full lifecycle of event ticketing including registration, ticket generation with QR codes, email delivery, and check-in validation.                                                                 | Sticky Note3 content                                                                                     |
| Credentials required: Google Sheets OAuth2 (for reading/writing sheets), Gmail OAuth2 (for sending emails). Gmail API must be enabled in Google Cloud Console and use the same Google account as Sheets.                        | Sticky Note4 content                                                                                     |
| Registration endpoint expects POST JSON with fields: `nama`, `email`, `no_hp`, `jumlah_tiket`, `total_price`, `payment_method`.                                                                                              | Sticky Note6 content                                                                                     |
| Ticket scanner endpoint is POST `/v1/scanner`. It parses scanned QR data, verifies ticket status, and updates check-in.                                                                                                        | Sticky Note5 content                                                                                     |
| Ticket ID format: `TL-YYYYMMDD-XXXX-N-HASH` where YYYYMMDD is date, XXXX is row number, N is ticket number, HASH is random unique string.                                                                                      | Sticky Note8 content                                                                                     |
| QR codes are generated using the external API: https://api.qrserver.com/v1/create-qr-code/                                                                                                                                   | Node: Generate QR Code                                                                                   |
| The HTML email template includes embedded base64 QR code image and a styled layout with important usage notes for ticket holders.                                                                                            | Node: Build HTML Email                                                                                   |

---

This structured documentation enables advanced users and automation agents to fully understand, reproduce, and maintain the Event Ticketing Backend workflow in n8n. It covers all nodes, logic branches, data flow, and integration points with Google Sheets and Gmail.