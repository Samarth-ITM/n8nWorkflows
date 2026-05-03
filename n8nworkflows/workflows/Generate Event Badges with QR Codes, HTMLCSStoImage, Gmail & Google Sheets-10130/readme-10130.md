Generate Event Badges with QR Codes, HTMLCSStoImage, Gmail & Google Sheets

https://n8nworkflows.xyz/workflows/generate-event-badges-with-qr-codes--htmlcsstoimage--gmail---google-sheets-10130


# Generate Event Badges with QR Codes, HTMLCSStoImage, Gmail & Google Sheets

---

### 1. Workflow Overview

This workflow automates the generation and distribution of personalized event badges with embedded QR codes for attendees. It targets event organizers managing registrations for conferences, seminars, corporate events, or training programs who want to streamline badge creation, tracking, and delivery.

**Logical Blocks:**

- **1.1 Input Reception and Validation:** Receive attendee registration data through a webhook, validate, and sanitize inputs.
- **1.2 QR Code Generation:** Generate a unique QR code image representing the attendee ID.
- **1.3 QR Code URL Preparation:** Convert the binary QR code output into a publicly accessible URL for embedding.
- **1.4 Badge Design & Image Generation:** Use a styled HTML template to create a badge displaying attendee info and QR code, convert this HTML to a PNG image via an external API.
- **1.5 Badge Generation Validation:** Check that the badge image URL was successfully generated.
- **1.6 Data Logging:** Append the attendee and badge info to a Google Sheets document for tracking.
- **1.7 Badge Delivery:** Email the badge image and event instructions to the attendee.
- **1.8 Error Handling:** Send alert emails to admins if badge generation fails.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:** This block captures new attendee registration via webhook, validates email and name, normalizes data, and generates a unique attendee ID with initials.
- **Nodes Involved:** Webhook, Validate & Sanitize
- **Node Details:**

  - **Webhook**
    - Type: Webhook trigger (HTTP POST)
    - Configuration: Listens at `/new-attendee` path for POST requests with JSON body containing attendee info.
    - Input: External HTTP POST with JSON payload (name, email, event, role)
    - Output: Raw JSON payload forwarded to next node.
    - Edge Cases: Missing or malformed JSON, unsupported HTTP methods.
  
  - **Validate & Sanitize (Code)**
    - Type: JavaScript code node for data validation and transformation.
    - Configuration:
      - Validates presence of `name` and properly formatted `email`.
      - Normalizes email to lowercase.
      - Capitalizes attendee name.
      - Generates a unique attendee ID using timestamp and random characters.
      - Extracts initials (up to 2 letters).
      - Sets default event title and role if missing.
      - Adds a timestamp and static event date string.
    - Key Expressions: Regex for email validation, string manipulations, unique ID generation.
    - Input: JSON from Webhook.
    - Output: Sanitized and enriched JSON with fields: name, email, event, role, attendeeId, initials, eventDate, timestamp.
    - Edge Cases: Throws error if name or email invalid, which halts the workflow.

#### 2.2 QR Code Generation

- **Overview:** Generates a binary PNG QR code encoding the unique attendee ID.
- **Nodes Involved:** Generate QR Code
- **Node Details:**

  - **Generate QR Code (HTTP Request)**
    - Type: HTTP Request node
    - Configuration:
      - Calls `https://api.qrserver.com/v1/create-qr-code/`
      - Query params: size=300x300, data=attendeeId (from previous node)
      - Response type: binary file (PNG), output property named `qrcode`
      - Continue on fail enabled to avoid workflow failure on QR API issues.
    - Input: Sanitized data with attendeeId.
    - Output: Binary PNG QR code image attached to output data.
    - Edge Cases: API downtime, rate limits, network errors.

#### 2.3 QR Code URL Preparation

- **Overview:** Converts the QR code binary into a publicly accessible URL to embed in the badge HTML.
- **Nodes Involved:** Prepare QR URL
- **Node Details:**

  - **Prepare QR URL (Code)**
    - Type: JavaScript code node
    - Configuration:
      - Receives attendee data and prepares a QR code URL using the qrserver.com service URL with the attendeeId embedded.
      - Returns all attendee info plus newly constructed `qrCodeUrl`.
    - Key Expressions: Template literal constructing QR URL.
    - Input: Binary QR code from previous node.
    - Output: JSON including qrCodeUrl for next step.
    - Edge Cases: If attendeeId missing or malformed, URL may be incorrect.

#### 2.4 Badge Design & Image Generation

- **Overview:** Builds a styled HTML badge embedding attendee details and QR code URL, converts it to a PNG image using Htmlcsstoimage API.
- **Nodes Involved:** Badge Design
- **Node Details:**

  - **Badge Design (HTML CSS to Image)**
    - Type: Htmlcsstoimage node (external API integration)
    - Configuration:
      - Uses a full HTML template with inline CSS for a visually appealing badge.
      - Injects dynamic variables: initials, event, eventDate, name, role, qrCodeUrl, attendeeId.
      - Output: URL to generated PNG badge image.
      - Uses Basic Auth with Htmlcsstoimage API credentials.
    - Input: JSON with attendee data and qrCodeUrl.
    - Output: JSON with `image_url` field containing badge PNG URL.
    - Edge Cases: API failures, invalid HTML, auth errors.
    - Continue on fail enabled to avoid total workflow halt.

#### 2.5 Badge Generation Validation

- **Overview:** Checks if the badge image URL exists, branching to success or error handling.
- **Nodes Involved:** Is Badge Generated? (If)
- **Node Details:**

  - **Is Badge Generated? (If node)**
    - Type: Conditional logic node
    - Configuration:
      - Condition: `image_url` property is not empty.
      - True: Proceed to log data and send badge email.
      - False: Trigger error alert email to admin.
    - Input: Output of Badge Design node.
    - Output: Two branches - success and error.
    - Edge Cases: Empty or missing image_url due to API failure or malformed data.

#### 2.6 Data Logging

- **Overview:** Appends attendee and badge info to a Google Sheet for record keeping.
- **Nodes Involved:** Log to Sheets, Extract Image URL
- **Node Details:**

  - **Log to Sheets (Google Sheets)**
    - Type: Google Sheets node
    - Configuration:
      - Appends a row to sheet named "BadgesLog" inside the spreadsheet "Event Badge Tracker".
      - Columns: Name, Role, Email, Event, Status ("Sent"), Badge URL, Timestamp, Attendee ID.
      - Uses Google Sheets OAuth2 credentials.
      - Continue on fail enabled.
    - Input: Data validated from previous steps plus badge image URL.
    - Output: Append confirmation JSON.
    - Edge Cases: API auth failure, spreadsheet ID incorrect, rate limits.

  - **Extract Image URL (HTTP Request)**
    - Type: HTTP Request node
    - Configuration:
      - Downloads the badge image file from the URL generated by Htmlcsstoimage.
      - Response format: file binary, output property named `badge`.
    - Input: Badge image URL.
    - Output: Binary badge image file passed to email node.
    - Edge Cases: URL inaccessible, network errors, file download failure.

#### 2.7 Badge Delivery

- **Overview:** Sends the badge as an email attachment to the attendee with personalized event info and instructions.
- **Nodes Involved:** Send Badge Email
- **Node Details:**

  - **Send Badge Email (Gmail)**
    - Type: Gmail node (OAuth2)
    - Configuration:
      - Sends to attendee's email.
      - Subject includes event name dynamically.
      - HTML email body with personalized greeting, badge details, event info, instructions.
      - Attaches PNG badge image binary.
      - Continue on fail enabled to avoid total halt.
    - Input: Binary badge image and attendee email/details.
    - Output: Email sent confirmation.
    - Edge Cases: Gmail API auth failure, invalid email, attachment size limits.

#### 2.8 Error Handling

- **Overview:** In case badge generation fails, sends an alert email to the administrator with detailed failure info and action steps.
- **Nodes Involved:** Send Error Alert
- **Node Details:**

  - **Send Error Alert (Gmail)**
    - Type: Gmail node (OAuth2)
    - Configuration:
      - Sends to admin@example.com.
      - HTML email body containing attendee details, timestamp, possible failure reasons, and remediation instructions.
      - Subject: "⚠️ Badge Generation Failed".
    - Input: Data from Validate & Sanitize node for attendee context.
    - Output: Alert email sent confirmation.
    - Edge Cases: Gmail auth issues, no admin email set, malformed data.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                         | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                     |
|---------------------|---------------------------|---------------------------------------|-----------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Webhook             | Webhook                   | Receive registration data via HTTP POST | None                  | Validate & Sanitize      | ## STEP 1: WEBHOOK TRIGGER<br>Test with cURL example provided                                  |
| Validate & Sanitize  | Code                      | Validate and normalize input data      | Webhook                | Generate QR Code         | ## STEP 2: DATA VALIDATION<br>Validates email and name, generates unique ID                    |
| Generate QR Code     | HTTP Request              | Generate QR code image (binary PNG)    | Validate & Sanitize    | Prepare QR URL           | ## STEP 3: QR CODE GENERATION<br>Uses qrserver.com API, continues on fail                      |
| Prepare QR URL       | Code                      | Create public QR code URL for HTML     | Generate QR Code       | Badge Design             | ## STEP 4: PREPARE QR URL<br>Creates URL to embed in badge template                            |
| Badge Design         | HTML CSS to Image         | Generate badge PNG from HTML+CSS       | Prepare QR URL         | Is Badge Generated?      | ## STEP 5: BUILD HTML BADGE<br>Uses Htmlcsstoimage API, basic auth, continues on fail          |
| Is Badge Generated?  | If                        | Conditional check for badge image URL  | Badge Design           | Log to Sheets, Send Error Alert | ## STEP 6: CONDITIONAL LOGIC<br>Checks if badge generated successfully                        |
| Log to Sheets        | Google Sheets             | Append attendee and badge info to sheet | Is Badge Generated? (true) | Extract Image URL       | ## STEP 9: LOG TO GOOGLE SHEETS<br>Appends data, continues on fail                            |
| Extract Image URL    | HTTP Request              | Download badge PNG file for email      | Log to Sheets          | Send Badge Email         | ## STEP 8: EXTRACT BADGE URL<br>Returns binary file named badge                               |
| Send Badge Email     | Gmail                     | Email badge with instructions to attendee | Extract Image URL      | None                    | ## STEP 9: SEND BADGE EMAIL<br>Sends personalized email with attachment, continues on fail    |
| Send Error Alert     | Gmail                     | Notify admin if badge generation fails | Is Badge Generated? (false) | None                    | ## ERROR HANDLING<br>Sends detailed failure alert email to admin                              |
| Sticky Note1         | Sticky Note               | Setup credentials instructions          | None                  | None                    | ## SETUP CREDENTIALS FIRST<br>API keys and OAuth2 setup instructions                          |
| Sticky Note2 to 11   | Sticky Note               | Step-by-step explanations and tips      | None                  | None                    | Various notes providing detailed step explanations and troubleshooting tips                   |
| Sticky Note          | Sticky Note               | Workflow overview                       | None                  | None                    | ## WORKFLOW OVERVIEW<br>Summary of purpose, steps, success and error paths                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook
   - HTTP Method: POST
   - Path: `new-attendee`
   - Purpose: Receive attendee registration JSON payload with fields: name, email, event, role.

2. **Add Code Node "Validate & Sanitize"**
   - Purpose: Validate email format and presence of name; normalize email to lowercase; capitalize name; generate unique attendee ID; extract initials; add timestamp and event date.
   - JS code as per specification:
     - Use regex to validate email.
     - Throw errors if validation fails.
     - Return enriched JSON with all required fields.

3. **Add HTTP Request Node "Generate QR Code"**
   - URL: `https://api.qrserver.com/v1/create-qr-code/`
   - Query Parameters:
     - size = `300x300`
     - data = attendeeId from previous node
   - Response Format: File binary, output property `qrcode`
   - Continue on fail: Enabled (true)
   - Purpose: Generate QR code PNG binary for attendee.

4. **Add Code Node "Prepare QR URL"**
   - JS code:
     - Construct QR code URL for embedding, e.g. `https://api.qrserver.com/v1/create-qr-code/?size=300x300&data=attendeeId`
     - Return all attendee data plus `qrCodeUrl`.
   - Purpose: Prepare accessible QR code URL for badge template.

5. **Add HTML CSS to Image Node "Badge Design"**
   - HTML content: Use provided full HTML template embedding dynamic variables via mustache syntax, including initials, name, role, event, eventDate, qrCodeUrl, attendeeId.
   - Credentials: Htmlcsstoimage API key (Basic Auth)
   - Continue on fail: Enabled (true)
   - Purpose: Convert HTML badge to PNG image URL.

6. **Add If Node "Is Badge Generated?"**
   - Condition: Check if `image_url` from Badge Design node is not empty.
   - True branch: Proceed to logging and email.
   - False branch: Trigger error alert.

7. **Add Google Sheets Node "Log to Sheets"**
   - Operation: Append
   - Spreadsheet ID: Your Google Sheet ID
   - Sheet Name: "BadgesLog" or equivalent
   - Columns: Name, Role, Email, Event, Status ("Sent"), Badge URL, Timestamp, Attendee ID
   - Credentials: Google Sheets OAuth2
   - Continue on fail: Enabled
   - Input fields mapped from previous nodes.

8. **Add HTTP Request Node "Extract Image URL"**
   - Purpose: Download badge image binary from the URL in `image_url`.
   - Response Format: File binary
   - Output Property: `badge`
   - Input: Badge URL from Log to Sheets or Badge Design.

9. **Add Gmail Node "Send Badge Email"**
   - Send To: Attendee email (dynamic)
   - Subject: "Your Event Badge for {Event} 🎫"
   - HTML Body: Use provided personalized email template with embedded variables.
   - Attachments: Attach binary badge image from previous node.
   - Credentials: Gmail OAuth2
   - Continue on fail: Enabled

10. **Add Gmail Node "Send Error Alert"**
    - Send To: admin@example.com (replace with real admin email)
    - Subject: "⚠️ Badge Generation Failed"
    - HTML Body: Use detailed failure alert template with attendee info and troubleshooting instructions.
    - Credentials: Gmail OAuth2

11. **Connect Nodes**
    - Webhook → Validate & Sanitize → Generate QR Code → Prepare QR URL → Badge Design → Is Badge Generated?
    - Is Badge Generated? TRUE → Log to Sheets → Extract Image URL → Send Badge Email
    - Is Badge Generated? FALSE → Send Error Alert

12. **Set Credentials**
    - Htmlcsstoimage API: Obtain from https://htmlcsstoimg.com, set Basic Auth credential.
    - Gmail OAuth2: Authenticate with Gmail, grant send email permissions.
    - Google Sheets OAuth2: Authenticate with Google, grant read/write access to target sheet.

13. **Test Each Credential**
    - Use credential test features in n8n to verify connectivity before running full workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| Setup credentials BEFORE running workflow: Htmlcsstoimage API (https://htmlcsstoimg.com), Gmail OAuth2 (with send permissions), Google Sheets OAuth2 (read/write). Use dedicated Gmail for automation; monitor API rate limits. | Sticky Note10 (Setup Credentials First) |
| Workflow takes approximately 5-8 seconds end-to-end to process and deliver badges.                                                                                                                                           | Sticky Note (Workflow Overview)          |
| Error handling sends detailed failure alerts to admin@example.com with attendee details and recovery steps.                                                                                                                 | Sticky Note9 (Error Handling)             |
| Sample cURL for webhook testing provided in Sticky Note1 to simulate attendee registration POST requests.                                                                                                                    | Sticky Note1 (Webhook Trigger)            |
| The badge HTML template uses dynamic variables and includes a modern, visually appealing design with gradients, shadows, and responsive layout for printing or digital use.                                                    | Badge Design Node (HTML Template)         |
| Use a new Google Sheet per event to keep logs organized and avoid rate limits.                                                                                                                                                 | Sticky Note10 (Pro Tips)                   |

---

**Disclaimer:** The provided documentation is based exclusively on an automated n8n workflow analysis. It complies with all applicable content policies and handles only lawful, public data.

---