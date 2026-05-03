Generate concert ticket PDFs with QR codes using PDF Generator API

https://n8nworkflows.xyz/workflows/generate-concert-ticket-pdfs-with-qr-codes-using-pdf-generator-api-14216


# Generate concert ticket PDFs with QR codes using PDF Generator API

# 1. Workflow Overview

This workflow receives a concert ticket request through an n8n-hosted form, prepares ticket metadata, generates a personalized PDF ticket through PDF Generator API, sends the attendee a confirmation email with a link to the PDF, logs the issued ticket to Google Sheets, and finally posts a notification to Slack.

Typical use cases:
- Event ticket issuance for concerts or live shows
- Simple attendee intake with automated PDF delivery
- Internal ticket tracking and organizer notifications
- Low-code generation of branded tickets with QR codes embedded in the PDF template

The workflow is linear and consists of one main execution path.

## 1.1 Input Reception

The workflow starts with a form trigger that collects attendee and event details directly from the end user.

## 1.2 Ticket Data Preparation

A Code node generates a unique ticket ID, normalizes tier-related values, prepares display fields for email styling, and builds the payload expected by the PDF Generator API template.

## 1.3 PDF Ticket Generation

The prepared template data is sent to PDF Generator API, which renders a hosted PDF ticket URL. The QR code is expected to be part of the PDF template itself, not created by n8n.

## 1.4 Confirmation Email Composition and Delivery

A second Code node builds an HTML email using both the original ticket data and the PDF URL. A Gmail node then sends the message to the attendee.

## 1.5 Logging and Internal Notification

After the email is sent, the workflow appends or updates a Google Sheets row for tracking, then sends a Slack notification to the organizer with the ticket details and PDF link.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block exposes a public form endpoint that collects the information required to issue a concert ticket. It is the only entry point in the workflow.

**Nodes Involved:**  
- Concert Ticket Form

### Node: Concert Ticket Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that creates a hosted form and starts the workflow when submitted.
- **Configuration choices:**
  - Custom form path is configured.
  - Form title: “🎵 Concert Ticket Request”
  - Description explains that the user will receive a personalized PDF ticket with a QR code.
  - Required fields:
    - Full Name
    - Email Address
    - Event Name
    - Venue
    - Event Date
    - Seat Number
    - Ticket Tier
  - Ticket Tier is a dropdown with:
    - General Admission
    - VIP
    - Backstage Pass
- **Key expressions or variables used:**  
  None inside this node; it emits form fields as JSON keys using the field labels.
- **Input and output connections:**
  - **Input:** none
  - **Output:** Prepare Ticket Data
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`; this requires a recent n8n version supporting Form Trigger node v2.x behavior and field configuration.
- **Edge cases or potential failure types:**
  - Public form exposure may allow unwanted submissions if the URL is shared broadly.
  - Field labels are used later as JSON keys; renaming a field here without updating downstream Code logic will break expressions.
  - Invalid or fake email addresses are not deeply validated beyond email field format.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Ticket Data Preparation

**Overview:**  
This block transforms raw form data into a structured ticket payload. It generates a unique ticket ID, derives tier display values, and constructs the variables expected by the PDF template.

**Nodes Involved:**  
- Prepare Ticket Data

### Node: Prepare Ticket Data
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node used to enrich incoming form data.
- **Configuration choices:**
  - Reads the submitted form data from `$input.item.json`.
  - Generates an 8-character suffix from a limited character set that avoids ambiguous characters.
  - Builds a ticket ID with format `TKT-XXXXXXXX`.
  - Creates a human-readable `issuedDate` using `toLocaleDateString('en-US', ...)`.
  - Maps ticket tier into:
    - `tierBadge` for display/template values:
      - VIP → `VIP`
      - Backstage Pass → `BACKSTAGE`
      - otherwise → `GENERAL`
    - `tierColor` for the email theme
  - Sets a hard-coded PDF Generator API template ID:
    - `const TEMPLATE_ID = 1615251;`
  - Builds:
    - `emailSubject`
    - `outputName`
    - `templateData` object for PDF rendering
- **Key expressions or variables used:**
  - Input fields:
    - `item['Full Name']`
    - `item['Email Address']`
    - `item['Event Name']`
    - `item['Venue']`
    - `item['Event Date']`
    - `item['Seat Number']`
    - `item['Ticket Tier']`
  - Output fields:
    - `ticketId`
    - `attendeeName`
    - `attendeeEmail`
    - `eventName`
    - `venue`
    - `eventDate`
    - `seatNumber`
    - `ticketTier`
    - `tierBadge`
    - `tierColor`
    - `issuedDate`
    - `emailSubject`
    - `templateId`
    - `outputName`
    - `templateData`
- **Input and output connections:**
  - **Input:** Concert Ticket Form
  - **Output:** Generate Concert Ticket PDF
- **Version-specific requirements:**  
  Uses Code node `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Ticket ID collision is unlikely but possible because uniqueness is random and not checked against prior records.
  - If form field labels change, this node will return undefined values.
  - `toLocaleDateString('en-US', ...)` depends on runtime locale support, though this is generally safe in n8n environments.
  - The embedded template ID must exist in the connected PDF Generator API account; otherwise generation fails.
  - The code comment mentions replacing `123456`, but the actual configured value is `1615251`; documentation and implementation must stay aligned.
- **Sub-workflow reference:**  
  None.

---

## 2.3 PDF Ticket Generation

**Overview:**  
This block submits the prepared variable set to PDF Generator API and gets back a hosted PDF URL. The actual QR code is expected to be rendered by the PDF template using `ticket_id`.

**Nodes Involved:**  
- Generate Concert Ticket PDF

### Node: Generate Concert Ticket PDF
- **Type and technical role:** `@pdfgeneratorapi/n8n-nodes-pdf-generator-api.pdfGeneratorApi`  
  Third-party integration node that renders a document from a PDF Generator API template.
- **Configuration choices:**
  - `templateId` is taken dynamically from `{{$json.templateId}}`
  - `data` is sent as a JSON string from `{{$json.templateData}}`
  - Output mode is `url`
  - Additional field `outputName` is set dynamically from `{{$json.outputName}}`
- **Key expressions or variables used:**
  - `={{ JSON.stringify($json.templateData) }}`
  - `={{ $json.templateId }}`
  - `={{ $json.outputName }}`
- **Input and output connections:**
  - **Input:** Prepare Ticket Data
  - **Output:** Build Confirmation Email
- **Version-specific requirements:**  
  Uses node `typeVersion: 1`. The custom/community node package for PDF Generator API must be installed and compatible with the running n8n version.
- **Edge cases or potential failure types:**
  - Authentication failure if PDF Generator API credentials are missing or invalid.
  - Template ID not found or inaccessible under the connected account.
  - Invalid template data structure or template variables mismatching component bindings.
  - Network timeout or API rate limiting.
  - If output remains URL mode, the returned link is temporary according to the sticky note and may expire after 30 days.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Confirmation Email Composition and Delivery

**Overview:**  
This block constructs a styled HTML email using ticket data and the generated PDF link, then sends it to the attendee through Gmail.

**Nodes Involved:**  
- Build Confirmation Email
- Send Ticket to Attendee

### Node: Build Confirmation Email
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that builds an HTML email body from prior node outputs.
- **Configuration choices:**
  - Reads current input from PDF Generator API response.
  - Reads original prepared ticket data via cross-node access:
    - `$('Prepare Ticket Data').item.json`
  - Extracts PDF URL from:
    - `pdfData.response || ''`
  - Builds a complete HTML email string with:
    - Tier-specific color theme
    - Event summary card
    - Ticket ID
    - Download button linking to the PDF
    - Footer credit links to PDF Generator API and n8n
  - Returns:
    - `attendeeEmail`
    - `emailSubject`
    - `emailHtml`
    - `pdfUrl`
    - `ticketId`
    - `eventName`
    - `tierBadge`
- **Key expressions or variables used:**
  - `$('Prepare Ticket Data').item.json`
  - `pdfData.response`
  - `t.tierColor`
  - `t.attendeeName`
  - `t.ticketTier`
  - `t.eventName`
  - `t.venue`
  - `t.eventDate`
  - `t.seatNumber`
  - `t.ticketId`
  - `t.attendeeEmail`
  - `t.emailSubject`
- **Input and output connections:**
  - **Input:** Generate Concert Ticket PDF
  - **Output:** Send Ticket to Attendee
- **Version-specific requirements:**  
  Uses Code node `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - If the PDF node returns a different response shape than expected, `pdfUrl` may be empty.
  - HTML is manually concatenated; malformed escaping could occur if user input contains unusual characters. Most inputs will still render, but HTML-safe encoding is not applied.
  - Cross-node reference assumes one item flow and stable execution order.
- **Sub-workflow reference:**  
  None.

### Node: Send Ticket to Attendee
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the composed message through a connected Gmail account.
- **Configuration choices:**
  - Recipient: `={{ $json.attendeeEmail }}`
  - Subject: `={{ $json.emailSubject }}`
  - Message body: `={{ $json.emailHtml }}`
  - No extra options configured
  - The message is intended as HTML content
- **Key expressions or variables used:**
  - `={{ $json.attendeeEmail }}`
  - `={{ $json.emailHtml }}`
  - `={{ $json.emailSubject }}`
- **Input and output connections:**
  - **Input:** Build Confirmation Email
  - **Output:** Log Ticket Sale
- **Version-specific requirements:**  
  Uses Gmail node `typeVersion: 2.1`; requires valid Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - OAuth token expiration or revoked Gmail access.
  - Gmail sending limits or anti-spam restrictions.
  - If HTML mode is not interpreted as expected in the environment, message rendering could differ.
  - Email delivery success in Gmail does not guarantee inbox delivery.
  - If switching PDF output from URL to file, this node must be reconfigured to send attachments.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Logging and Internal Notification

**Overview:**  
This block records ticket issuance in Google Sheets and notifies the organizer in Slack. It provides auditability and real-time operational visibility.

**Nodes Involved:**  
- Log Ticket Sale
- Notify Event Organizer

### Node: Log Ticket Sale
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Writes workflow output into a Google Sheet for tracking.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document: a specific spreadsheet named “Concert Tickets”
  - Sheet/tab selected by `gid=0`, cached result name shown as `Sheet1`
  - Mapping mode: `autoMapInputData`
  - Type conversion disabled
- **Key expressions or variables used:**  
  No explicit expressions in parameters; the node relies on incoming JSON fields and automatic column mapping.
- **Input and output connections:**
  - **Input:** Send Ticket to Attendee
  - **Output:** Notify Event Organizer
- **Version-specific requirements:**  
  Uses Google Sheets node `typeVersion: 4.5`; requires OAuth2 credentials.
- **Edge cases or potential failure types:**
  - The sticky note says the tab should be named `Tickets`, but the current node points to `Sheet1` via `gid=0`. This mismatch can confuse reproduction if not corrected.
  - Auto-mapping only works if incoming keys match column names or supported mapping behavior. In this workflow, incoming data after Gmail may not contain the full ticket record unless the Gmail node preserves fields as expected in that n8n version.
  - `appendOrUpdate` typically requires clear matching behavior to avoid duplicates or unintended updates; here `matchingColumns` is empty.
  - Spreadsheet permission errors or wrong document selection will fail execution.
- **Sub-workflow reference:**  
  None.

### Node: Notify Event Organizer
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack message with ticket issuance details.
- **Configuration choices:**
  - Message text is dynamically composed using cross-node references to:
    - Prepare Ticket Data
    - Generate Concert Ticket PDF
  - `select: user` with a user ID configured
  - No channel is explicitly shown in the node parameters; this configuration appears aimed at a Slack user target rather than a channel
- **Key expressions or variables used:**
  - `$('Prepare Ticket Data').item.json.tierBadge`
  - `$('Prepare Ticket Data').item.json.eventName`
  - `$('Prepare Ticket Data').item.json.attendeeName`
  - `$('Prepare Ticket Data').item.json.attendeeEmail`
  - `$('Prepare Ticket Data').item.json.seatNumber`
  - `$('Prepare Ticket Data').item.json.ticketId`
  - `$('Generate Concert Ticket PDF').item.json.response`
- **Input and output connections:**
  - **Input:** Log Ticket Sale
  - **Output:** none
- **Version-specific requirements:**  
  Uses Slack node `typeVersion: 2.3`; requires Slack API credentials and compatible workspace permissions.
- **Edge cases or potential failure types:**
  - Slack auth or permission issues.
  - The sticky note refers to setting a channel like `#tickets`, but the actual node is configured with `select: user`, so documentation and implementation are inconsistent.
  - If upstream nodes fail or return empty values, the message may contain blank fields.
  - User ID may not be valid in another workspace after import.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Concert Ticket Form | n8n-nodes-base.formTrigger | Public form entry point for attendee and event data collection |  | Prepare Ticket Data | ## 🎵 Concert Ticket Generator<br>### Powered by PDF Generator API<br><br>This workflow accepts ticket requests via a web form, generates a personalized **PDF ticket with a built-in QR code** using **PDF Generator API**, emails it to the attendee, logs the sale to **Google Sheets**, and notifies the event organizer via **Slack**.<br><br>---<br><br>### ⚙️ Setup (4 steps)<br><br>**1. Create a PDF ticket template in PDF Generator API**<br>Sign up at [pdfgeneratorapi.com](https://pdfgeneratorapi.com) → Templates → New Template.<br>Design your concert ticket and use these data variables:<br>- `ticket_id` — unique ticket identifier *(also the QR code data — see note →)*<br>- `attendee_name` · `event_name` · `venue` · `event_date` · `seat_number`<br>- `ticket_tier` — tier badge (GENERAL / VIP / BACKSTAGE)<br>- `issued_date` — ticket issue date<br><br>**2. Set your Template ID**<br>In the **Generate Concert Ticket PDF** node, replace `123456` with your actual template ID (found in the PDF Generator API dashboard).<br><br>**3. Add credentials**<br>- **Generate Concert Ticket PDF** node → add your PDF Generator API credentials<br>- **Send Ticket to Attendee** node → connect your Gmail account<br>- **Log Ticket Sale** node → connect your Google Sheets account<br>- **Notify Event Organizer** node → connect your Slack account<br><br>**4. Configure Google Sheets & Slack**<br>See the sticky notes on the right for details. |
| Prepare Ticket Data | n8n-nodes-base.code | Generate ticket ID and prepare PDF/email payload | Concert Ticket Form | Generate Concert Ticket PDF | ## 🎵 Concert Ticket Generator<br>### Powered by PDF Generator API<br><br>This workflow accepts ticket requests via a web form, generates a personalized **PDF ticket with a built-in QR code** using **PDF Generator API**, emails it to the attendee, logs the sale to **Google Sheets**, and notifies the event organizer via **Slack**.<br><br>---<br><br>### ⚙️ Setup (4 steps)<br><br>**1. Create a PDF ticket template in PDF Generator API**<br>Sign up at [pdfgeneratorapi.com](https://pdfgeneratorapi.com) → Templates → New Template.<br>Design your concert ticket and use these data variables:<br>- `ticket_id` — unique ticket identifier *(also the QR code data — see note →)*<br>- `attendee_name` · `event_name` · `venue` · `event_date` · `seat_number`<br>- `ticket_tier` — tier badge (GENERAL / VIP / BACKSTAGE)<br>- `issued_date` — ticket issue date<br><br>**2. Set your Template ID**<br>In the **Generate Concert Ticket PDF** node, replace `123456` with your actual template ID (found in the PDF Generator API dashboard).<br><br>**3. Add credentials**<br>- **Generate Concert Ticket PDF** node → add your PDF Generator API credentials<br>- **Send Ticket to Attendee** node → connect your Gmail account<br>- **Log Ticket Sale** node → connect your Google Sheets account<br>- **Notify Event Organizer** node → connect your Slack account<br><br>**4. Configure Google Sheets & Slack**<br>See the sticky notes on the right for details. |
| Generate Concert Ticket PDF | @pdfgeneratorapi/n8n-nodes-pdf-generator-api.pdfGeneratorApi | Render ticket PDF from template data | Prepare Ticket Data | Build Confirmation Email | ## 🎵 Concert Ticket Generator<br>### Powered by PDF Generator API<br><br>This workflow accepts ticket requests via a web form, generates a personalized **PDF ticket with a built-in QR code** using **PDF Generator API**, emails it to the attendee, logs the sale to **Google Sheets**, and notifies the event organizer via **Slack**.<br><br>---<br><br>### ⚙️ Setup (4 steps)<br><br>**1. Create a PDF ticket template in PDF Generator API**<br>Sign up at [pdfgeneratorapi.com](https://pdfgeneratorapi.com) → Templates → New Template.<br>Design your concert ticket and use these data variables:<br>- `ticket_id` — unique ticket identifier *(also the QR code data — see note →)*<br>- `attendee_name` · `event_name` · `venue` · `event_date` · `seat_number`<br>- `ticket_tier` — tier badge (GENERAL / VIP / BACKSTAGE)<br>- `issued_date` — ticket issue date<br><br>**2. Set your Template ID**<br>In the **Generate Concert Ticket PDF** node, replace `123456` with your actual template ID (found in the PDF Generator API dashboard).<br><br>**3. Add credentials**<br>- **Generate Concert Ticket PDF** node → add your PDF Generator API credentials<br>- **Send Ticket to Attendee** node → connect your Gmail account<br>- **Log Ticket Sale** node → connect your Google Sheets account<br>- **Notify Event Organizer** node → connect your Slack account<br><br>**4. Configure Google Sheets & Slack**<br>See the sticky notes on the right for details.<br><br>### 🔲 QR Code — Built into the Template<br><br>The QR code is **not generated by this workflow**. Instead, it is a native **QR Code component** inside your PDF Generator API template.<br><br>**How to set it up in the template editor:**<br>1. Open your template in PDF Generator API<br>2. Add a **QR Code** component to the ticket design<br>3. Set the QR code data field to: `{{ ticket_id }}`<br>4. The QR will automatically encode the unique ticket ID for each generated PDF<br><br>This approach is simpler, more reliable, and gives you full design control over QR code size and position.<br><br>---<br>**Output format:** `URL` — the node returns a hosted PDF link valid for 30 days. Change to `File` to attach the PDF directly to the email (update the Gmail node accordingly). |
| Build Confirmation Email | n8n-nodes-base.code | Build styled HTML email with download link | Generate Concert Ticket PDF | Send Ticket to Attendee |  |
| Send Ticket to Attendee | n8n-nodes-base.gmail | Email the attendee their ticket link | Build Confirmation Email | Log Ticket Sale |  |
| Log Ticket Sale | n8n-nodes-base.googleSheets | Record issued ticket in spreadsheet | Send Ticket to Attendee | Notify Event Organizer | ### 📊 Google Sheets Tracker<br><br>Logs every issued ticket for event management.<br><br>**Setup:**<br>1. Create a Google Sheet with a tab named **Tickets**<br>2. Add these column headers in row 1:<br>   `Ticket ID` · `Attendee` · `Email` · `Event` · `Venue` · `Date` · `Seat` · `Tier` · `PDF URL` · `Issued At`<br>3. In the **Log Ticket Sale** node, set your Spreadsheet ID and connect your Google account<br><br>💡 Use this sheet to track attendance, validate tickets at the door, or export attendee lists. |
| Notify Event Organizer | n8n-nodes-base.slack | Send organizer notification in Slack | Log Ticket Sale |  | ### 🔔 Slack Notification<br><br>Sends a real-time alert to the event organizer whenever a new ticket is issued.<br><br>**Setup:**<br>1. Connect your Slack workspace in the **Notify Event Organizer** node<br>2. Set the channel name (e.g. `#tickets` or `#events`)<br>3. Optional: remove this node if you don't use Slack |
| 📋 Setup Instructions | n8n-nodes-base.stickyNote | Documentation / canvas note |  |  |  |
| 🔲 QR Code & PDF Output | n8n-nodes-base.stickyNote | Documentation / canvas note |  |  |  |
| 📊 Google Sheets Setup | n8n-nodes-base.stickyNote | Documentation / canvas note |  |  |  |
| 🔔 Slack Setup | n8n-nodes-base.stickyNote | Documentation / canvas note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Open n8n and create a blank workflow.
   - Name it something like: `Concert Ticket Generator with QR Code`.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Name: `Concert Ticket Form`
   - Set a custom path if desired.
   - Configure:
     - Title: `🎵 Concert Ticket Request`
     - Description: explain that users receive a personalized concert ticket via email with a QR code.
   - Add these required fields:
     1. `Full Name` — text
     2. `Email Address` — email
     3. `Event Name` — text
     4. `Venue` — text
     5. `Event Date` — text
     6. `Seat Number` — text
     7. `Ticket Tier` — dropdown with:
        - `General Admission`
        - `VIP`
        - `Backstage Pass`

3. **Add a Code node after the form**
   - Node type: **Code**
   - Name: `Prepare Ticket Data`
   - Connect: `Concert Ticket Form` → `Prepare Ticket Data`
   - Paste JavaScript that:
     - Reads form fields from `$input.item.json`
     - Generates a random ticket ID like `TKT-ABCDEFGH`
     - Creates `issuedDate`
     - Maps ticket tiers to:
       - `tierBadge`: `GENERAL`, `VIP`, `BACKSTAGE`
       - `tierColor`: suitable hex colors
     - Sets a `templateId`
     - Creates `outputName`
     - Creates `emailSubject`
     - Creates `templateData` with:
       - `ticket_id`
       - `attendee_name`
       - `event_name`
       - `venue`
       - `event_date`
       - `seat_number`
       - `ticket_tier`
       - `issued_date`
   - Use the same field names as the form labels, or adjust the code accordingly.

4. **Prepare your PDF Generator API template**
   - In PDF Generator API, create a concert ticket template.
   - Include placeholders for:
     - `ticket_id`
     - `attendee_name`
     - `event_name`
     - `venue`
     - `event_date`
     - `seat_number`
     - `ticket_tier`
     - `issued_date`
   - Add a **QR Code component** in the template editor.
   - Bind the QR code content to:
     - `{{ ticket_id }}`
   - Copy the numeric template ID from PDF Generator API.

5. **Set the template ID in the Code node**
   - In `Prepare Ticket Data`, replace the hard-coded template ID with your real template ID.
   - Keep it as a number or adapt the expression format consistently.

6. **Add the PDF Generator API node**
   - Node type: **PDF Generator API** community/custom node
   - Name: `Generate Concert Ticket PDF`
   - Connect: `Prepare Ticket Data` → `Generate Concert Ticket PDF`
   - Configure:
     - Template ID: expression using the incoming `templateId`
     - Data: expression that stringifies `templateData`
       - `={{ JSON.stringify($json.templateData) }}`
     - Output type: `URL`
     - Output name: `={{ $json.outputName }}`
   - Add PDF Generator API credentials.

7. **Add a second Code node to build the email**
   - Node type: **Code**
   - Name: `Build Confirmation Email`
   - Connect: `Generate Concert Ticket PDF` → `Build Confirmation Email`
   - Configure JavaScript to:
     - Read the generated PDF URL from the PDF node response, typically `response`
     - Read prepared ticket values from `$('Prepare Ticket Data').item.json`
     - Build an HTML email body
     - Return:
       - `attendeeEmail`
       - `emailSubject`
       - `emailHtml`
       - `pdfUrl`
       - `ticketId`
       - `eventName`
       - `tierBadge`

8. **Add a Gmail node**
   - Node type: **Gmail**
   - Name: `Send Ticket to Attendee`
   - Connect: `Build Confirmation Email` → `Send Ticket to Attendee`
   - Configure:
     - To: `={{ $json.attendeeEmail }}`
     - Subject: `={{ $json.emailSubject }}`
     - Message: `={{ $json.emailHtml }}`
   - Add Gmail OAuth2 credentials.
   - Ensure your Gmail node is configured to send the message body as intended for HTML email in your n8n version.

9. **Add a Google Sheets node**
   - Node type: **Google Sheets**
   - Name: `Log Ticket Sale`
   - Connect: `Send Ticket to Attendee` → `Log Ticket Sale`
   - Create a Google Sheet beforehand.
   - Add a header row such as:
     - `Ticket ID`
     - `Attendee`
     - `Email`
     - `Event`
     - `Venue`
     - `Date`
     - `Seat`
     - `Tier`
     - `PDF URL`
     - `Issued At`
   - In the node:
     - Select your spreadsheet
     - Select the target tab
     - Operation: `Append or Update`
   - Add Google Sheets OAuth2 credentials.
   - Important: because the current workflow uses auto-mapping, you may need to add an intermediate Set or Code node if you want exact column-name alignment. As configured in the provided workflow, this part may rely on output preservation from previous nodes and may need adjustment.

10. **Add a Slack node**
    - Node type: **Slack**
    - Name: `Notify Event Organizer`
    - Connect: `Log Ticket Sale` → `Notify Event Organizer`
    - Configure the message text with expressions referencing:
      - `Prepare Ticket Data`
      - `Generate Concert Ticket PDF`
    - Example content:
      - Tier badge
      - Event name
      - Attendee name
      - Email
      - Seat number
      - Ticket ID
      - PDF URL
    - Add Slack credentials.
    - Decide whether to send to:
      - a channel such as `#tickets`
      - or a specific user
    - Note that the provided workflow’s sticky note suggests channel usage, but the actual node is configured for a user target. Choose one explicitly.

11. **Connect nodes in this exact order**
    - `Concert Ticket Form` → `Prepare Ticket Data`
    - `Prepare Ticket Data` → `Generate Concert Ticket PDF`
    - `Generate Concert Ticket PDF` → `Build Confirmation Email`
    - `Build Confirmation Email` → `Send Ticket to Attendee`
    - `Send Ticket to Attendee` → `Log Ticket Sale`
    - `Log Ticket Sale` → `Notify Event Organizer`

12. **Add credentials**
    - PDF Generator API credentials for the PDF node
    - Gmail OAuth2 credentials for the email node
    - Google Sheets OAuth2 credentials for the Sheets node
    - Slack credentials for the Slack node

13. **Test with sample form data**
    - Submit the form with realistic sample values.
    - Confirm:
      - Ticket ID is generated
      - PDF URL is returned
      - Email is delivered
      - Google Sheets receives a row
      - Slack notification is posted

14. **Validate data consistency**
    - Ensure your PDF template variable names exactly match `templateData`.
    - Ensure your form field labels still match what `Prepare Ticket Data` expects.
    - Ensure the Google Sheet tab and column names match your logging strategy.
    - Ensure Slack destination type matches your intended target.

15. **Optional improvements when rebuilding**
    - Add a dedicated Set/Code node before Google Sheets to map exact sheet columns.
    - Add error handling branches for failed PDF generation or email send.
    - Store generated PDFs as files instead of URLs if long-term retention is needed.
    - Add uniqueness checks for `ticketId`.
    - Encode HTML entities in user-provided values before inserting them into the email.

**Sub-workflow setup:**  
There are no sub-workflows or Execute Workflow nodes in this workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PDF Generator API is used for ticket rendering and hosts the generated PDF when output mode is set to URL. | https://pdfgeneratorapi.com |
| The QR code is expected to be created inside the PDF Generator API template using `{{ ticket_id }}` as the encoded value. | PDF Generator API template editor |
| The workflow automation platform referenced in the email footer is n8n. | https://n8n.io |
| The setup note says to replace `123456` with your template ID, but the current Code node already uses `1615251`. Keep documentation and implementation synchronized. | Internal consistency note |
| The Google Sheets sticky note says the tab should be named `Tickets`, but the configured node points to `Sheet1`/`gid=0`. | Reconcile this during setup |
| The Slack sticky note suggests channel configuration, but the current Slack node is configured with a user target. | Reconcile this during setup |