Schedule appointments from a booking form with Google Calendar and Gmail

https://n8nworkflows.xyz/workflows/schedule-appointments-from-a-booking-form-with-google-calendar-and-gmail-13923


# Schedule appointments from a booking form with Google Calendar and Gmail

# 1. Workflow Overview

This workflow implements a self-hosted appointment booking system using **n8n**, **Google Calendar**, and **Gmail**. It exposes a public booking page, computes available time slots based on calendar occupancy, accepts appointment submissions, creates a calendar event, sends a confirmation email, and returns a branded success page.

It has **two entry points on the same path** (`/booking`), differentiated by HTTP method:

- **GET /booking** → displays the booking form with dynamically generated available slots
- **POST /booking** → processes the submitted form and creates the appointment

## 1.1 Booking Form Delivery

This block serves the HTML booking interface. It fetches upcoming calendar events, calculates free 30-minute slots for the next 30 days, and renders a complete HTML form with date navigation and slot selection.

## 1.2 Booking Submission Processing

This block receives the submitted form payload, computes the meeting end time from the chosen start slot, creates the Google Calendar event, enriches the event data for email rendering, sends a Gmail confirmation email, and returns an HTML success response.

## 1.3 Shared Design / Documentation Layer

A sticky note documents the workflow purpose, explains the two-path structure, and gives setup instructions for credentials and customization.

---

# 2. Block-by-Block Analysis

## 2.1 Booking Form Delivery

### Overview

This block handles the **GET** request used to display the booking page. It reads existing events from Google Calendar, derives open appointment slots, builds a full front-end HTML form, and returns that page through the webhook response node.

### Nodes Involved

- Webhook - Show Form
- Get Calendar Events
- Calculate Available Slots
- Build HTML Form
- Respond to Webhook

---

### Node: Webhook - Show Form

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for serving the booking form.
- **Configuration choices:**
  - Path: `booking`
  - Response mode: `responseNode`
  - HTTP method not explicitly set, so this webhook is intended for the default form-serving route; in context it acts as the **GET handler**
- **Key expressions or variables used:** none
- **Input and output connections:**
  - No input
  - Output → `Get Calendar Events`
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Path conflict risk because another webhook uses the same path with POST; this is valid only if methods differ as intended
  - If workflow is inactive, public URL will not serve production webhook
  - If response chain breaks before `Respond to Webhook`, callers may receive timeout/error
- **Sub-workflow reference:** none

---

### Node: Get Calendar Events

- **Type and technical role:** `n8n-nodes-base.googleCalendar`  
  Retrieves existing events from the selected Google Calendar so occupied slots can be excluded.
- **Configuration choices:**
  - Operation: `getAll`
  - Calendar: `primary`
  - Return all results: enabled
  - Time window:
    - `timeMin = {{$now.toISO()}}`
    - `timeMax = {{$now.plus({ days: 30 }).toISO()}}`
- **Key expressions or variables used:**
  - `$now.toISO()`
  - `$now.plus({ days: 30 }).toISO()`
- **Input and output connections:**
  - Input ← `Webhook - Show Form`
  - Output → `Calculate Available Slots`
- **Version-specific requirements:** `typeVersion: 1.2`
- **Edge cases or potential failure types:**
  - Google OAuth2 credential errors
  - Google API quota/rate limit issues
  - Empty result set is acceptable and handled downstream
  - All-day events and long events are passed through, but later filtered in code
- **Sub-workflow reference:** none

---

### Node: Calculate Available Slots

- **Type and technical role:** `n8n-nodes-base.code`  
  Generates 30-minute free slots for the next 30 days by comparing configured working hours to fetched Google Calendar events.
- **Configuration choices:**
  - JavaScript code node
  - Business rules:
    - Working days: Monday to Friday (`[1,2,3,4,5]`)
    - Working hours: `09:00` to `14:00`
    - Slot duration: `30` minutes
    - Planning horizon: `30` days
  - Timezone strategy:
    - Explicitly constructs slot timestamps with `+02:00`
    - Comments indicate intended target timezone is Greece
- **Key expressions or variables used:**
  - `$input.all()`
  - `event.start.dateTime`, `event.end.dateTime`
  - Generated slot object:
    - `value`: ISO timestamp with timezone offset
    - `label`: human-readable display text
- **Input and output connections:**
  - Input ← `Get Calendar Events`
  - Output → `Build HTML Form`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Timezone handling may be inaccurate during DST changes because the code hardcodes `+02:00`
  - Comment says Greece is 1 hour ahead of Berlin, but DST can invalidate that assumption seasonally
  - Events longer than 24 hours are intentionally ignored
  - All-day events are ignored because only events with `start.dateTime` are considered
  - If no free slots are found, returns one item with `{ noSlots: true, slots: [] }`; downstream HTML still renders but expects a compatible structure
  - If calendar returns many overlapping events, computation remains simple but could become heavier with large result sets
- **Sub-workflow reference:** none

---

### Node: Build HTML Form

- **Type and technical role:** `n8n-nodes-base.code`  
  Builds the entire front-end booking page as raw HTML, CSS, and JavaScript.
- **Configuration choices:**
  - Reads all slot items from input
  - Groups slots by date
  - Formats time labels manually to avoid browser timezone conversion
  - Produces a complete HTML document with:
    - Date navigator
    - Slot buttons
    - Hidden input for selected slot
    - First name, last name, email, and message fields
    - Client-side validation
- **Key expressions or variables used:**
  - `$input.all()`
  - `slot.value`
  - Embedded `slotsJson` in browser-side script
  - Form field names:
    - `timeslot`
    - `firstName`
    - `lastName`
    - `email`
    - `message`
- **Input and output connections:**
  - Input ← `Calculate Available Slots`
  - Output → `Respond to Webhook`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If upstream returns `{ noSlots: true, slots: [] }` instead of normal slot items, the code still treats it as slot input; this can create malformed grouping because `slot.value` is expected
  - Large embedded HTML is harder to maintain directly inside a Code node
  - Browser-side JavaScript assumes `slotsByDate.length` exists and is valid
  - Form action is empty (`action=""`), so the form posts back to the same URL; this is intended
- **Sub-workflow reference:** none

---

### Node: Respond to Webhook

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the generated HTML booking page back to the requester.
- **Configuration choices:**
  - Response type: text
  - Body: `{{$json.html}}`
  - Response header: `Content-Type: text/html`
- **Key expressions or variables used:**
  - `$json.html`
- **Input and output connections:**
  - Input ← `Build HTML Form`
  - No output
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - If `html` is missing from input, response body will be empty or invalid
  - Wrong content type would break rendering, but header is correctly set
- **Sub-workflow reference:** none

---

## 2.2 Booking Submission Processing

### Overview

This block handles the **POST** request generated when a user submits the form. It validates and transforms the chosen slot into a start/end time pair, creates the appointment in Google Calendar, builds formatted email data, sends a confirmation email, and returns an HTML confirmation page.

### Nodes Involved

- Webhook - Submit Form
- Code in JavaScript
- Create Calendar Event
- Format Email Data
- Send Confirmation Email
- Show Success Message

---

### Node: Webhook - Submit Form

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Receives form submissions from the HTML page.
- **Configuration choices:**
  - Path: `booking`
  - HTTP method: `POST`
  - Response mode: `responseNode`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - No input
  - Output → `Code in JavaScript`
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Missing expected POST form fields
  - If form encoding changes, `body` structure may differ
  - If no response node completes, request may hang or timeout
- **Sub-workflow reference:** none

---

### Node: Code in JavaScript

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the submitted timeslot and computes a 30-minute end time while preserving the original timezone offset.
- **Configuration choices:**
  - Reads `body` from the webhook payload
  - Expects `timeslot` in format like `2025-11-21T10:30:00+02:00`
  - Uses regex to extract date/time/offset parts
  - Adds 30 minutes manually
  - Returns a new `body` object including:
    - `startTime`
    - `endTime`
- **Key expressions or variables used:**
  - `$input.first().json.body`
  - `webhookData.timeslot`
  - Regex: `^(.+T)(\d{2}):(\d{2}):(\d{2})(\+\d{2}:\d{2})$`
- **Input and output connections:**
  - Input ← `Webhook - Submit Form`
  - Output → `Create Calendar Event`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Throws `Invalid time format` if the slot does not match the expected pattern
  - Regex only supports positive timezone offsets like `+02:00`; it does not support `Z` or negative offsets
  - Manual addition of 30 minutes can produce hour values beyond 23 and does not normalize date rollover properly
  - Assumes all bookings are exactly 30 minutes
- **Sub-workflow reference:** none

---

### Node: Create Calendar Event

- **Type and technical role:** `n8n-nodes-base.googleCalendar`  
  Creates the booked appointment in Google Calendar.
- **Configuration choices:**
  - Calendar: `primary`
  - Start: `{{$json.body.timeslot}}`
  - End: `{{$json.body.endTime}}`
  - Summary: `Meeting with {{ $json.body.firstName }} {{ $json.body.lastName }}`
  - Description includes:
    - customer full name
    - email
    - message fallback to `No message provided`
- **Key expressions or variables used:**
  - `$json.body.timeslot`
  - `$json.body.endTime`
  - `$json.body.firstName`
  - `$json.body.lastName`
  - `$json.body.email`
  - `$json.body.message || 'No message provided'`
- **Input and output connections:**
  - Input ← `Code in JavaScript`
  - Output → `Format Email Data`
- **Version-specific requirements:** `typeVersion: 1.2`
- **Edge cases or potential failure types:**
  - Google OAuth2 authentication failure
  - Calendar API quota/rate limit failure
  - Event creation conflict is not pre-checked at write time; race conditions are possible if two users submit the same slot near-simultaneously
  - Invalid start/end timestamps cause API rejection
- **Sub-workflow reference:** none

---

### Node: Format Email Data

- **Type and technical role:** `n8n-nodes-base.code`  
  Combines Google Calendar event data with original form fields and creates user-friendly date/time strings for email content.
- **Configuration choices:**
  - Reads event details from current input
  - Reads original form payload from `Webhook - Submit Form`
  - Formats:
    - `formattedDate`
    - `formattedTime`
  - Adds:
    - `firstName`
    - `lastName`
    - `email`
    - `message`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('Webhook - Submit Form').first().json.body`
  - `eventData.start.dateTime`
  - `eventData.end.dateTime`
  - `toLocaleDateString('en-US', ...)`
  - `toLocaleTimeString('en-US', ...)`
- **Input and output connections:**
  - Input ← `Create Calendar Event`
  - Output → `Send Confirmation Email`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Cross-node reference to `Webhook - Submit Form` assumes that node is present and reachable in execution data
  - Formatting depends on server/runtime locale support
  - Timezone display may reflect runtime interpretation of returned ISO timestamps
- **Sub-workflow reference:** none

---

### Node: Send Confirmation Email

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a styled HTML confirmation email to the user.
- **Configuration choices:**
  - Recipient: `{{$json.email}}`
  - Subject: `✅ Appointment Confirmed - {{ $json.formattedDate }}`
  - HTML body includes:
    - customer name
    - date and time
    - fixed duration of 30 minutes
    - email address
    - submitted message
    - Google Calendar link via `htmlLink`
- **Key expressions or variables used:**
  - `$json.email`
  - `$json.firstName`
  - `$json.lastName`
  - `$json.formattedDate`
  - `$json.formattedTime`
  - `$json.message`
  - `$json.htmlLink`
- **Input and output connections:**
  - Input ← `Format Email Data`
  - Output → `Show Success Message`
- **Version-specific requirements:** `typeVersion: 2.1`
- **Edge cases or potential failure types:**
  - Gmail OAuth2 authentication issues
  - Sending limits or Gmail API restrictions
  - HTML link may not be ideal as an “Add to Google Calendar” CTA depending on what `htmlLink` returned by Google Calendar represents
  - If recipient email is invalid, sending may fail
- **Sub-workflow reference:** none

---

### Node: Show Success Message

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns a final branded success page to the browser after successful booking and email sending.
- **Configuration choices:**
  - Response type: text
  - Static HTML success page
  - Header: `Content-Type: text/html`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input ← `Send Confirmation Email`
  - No output
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - If upstream email sending fails, this node is never reached
  - User receives no success page if booking is created but email step errors
- **Sub-workflow reference:** none

---

## 2.3 Shared Design / Documentation Layer

### Overview

This non-executing block contains a sticky note that explains the workflow structure and setup steps. It is useful operational documentation embedded directly in the canvas.

### Nodes Involved

- Sticky Note

---

### Node: Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation node for human operators.
- **Configuration choices:**
  - Contains workflow description, path overview, and quick-start steps
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none in execution; only maintainability concerns if note becomes outdated
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Show Form | Webhook | Public entry point for serving the booking page |  | Get Calendar Events | ## Self-Hosted Booking Form with Google Calendar<br>This workflow has **two paths**:<br>**Top row (GET):** Serves the booking form<br>Webhook → Get Calendar Events → Calculate Available Slots → Build HTML Form → Respond<br>**Bottom row (POST):** Processes a booking submission<br>Webhook → Calculate End Time → Create Calendar Event → Format Email Data → Send Confirmation → Show Success Page<br>### Quick Start<br>1. Connect Google Calendar OAuth2<br>2. Connect Gmail OAuth2<br>3. Adjust availability settings in "Calculate Available Slots"<br>4. Activate & visit: `<your-n8n-url>/webhook/booking` |
| Get Calendar Events | Google Calendar | Retrieves upcoming occupied events in the next 30 days | Webhook - Show Form | Calculate Available Slots | ## Self-Hosted Booking Form with Google Calendar<br>This workflow has **two paths**:<br>**Top row (GET):** Serves the booking form<br>Webhook → Get Calendar Events → Calculate Available Slots → Build HTML Form → Respond<br>**Bottom row (POST):** Processes a booking submission<br>Webhook → Calculate End Time → Create Calendar Event → Format Email Data → Send Confirmation → Show Success Page<br>### Quick Start<br>1. Connect Google Calendar OAuth2<br>2. Connect Gmail OAuth2<br>3. Adjust availability settings in "Calculate Available Slots"<br>4. Activate & visit: `<your-n8n-url>/webhook/booking` |
| Calculate Available Slots | Code | Computes available 30-minute appointment slots | Get Calendar Events | Build HTML Form | ## Self-Hosted Booking Form with Google Calendar<br>This workflow has **two paths**:<br>**Top row (GET):** Serves the booking form<br>Webhook → Get Calendar Events → Calculate Available Slots → Build HTML Form → Respond<br>**Bottom row (POST):** Processes a booking submission<br>Webhook → Calculate End Time → Create Calendar Event → Format Email Data → Send Confirmation → Show Success Page<br>### Quick Start<br>1. Connect Google Calendar OAuth2<br>2. Connect Gmail OAuth2<br>3. Adjust availability settings in "Calculate Available Slots"<br>4. Activate & visit: `<your-n8n-url>/webhook/booking` |
| Build HTML Form | Code | Generates the booking page HTML/CSS/JS | Calculate Available Slots | Respond to Webhook | ## Self-Hosted Booking Form with Google Calendar<br>This workflow has **two paths**:<br>**Top row (GET):** Serves the booking form<br>Webhook → Get Calendar Events → Calculate Available Slots → Build HTML Form → Respond<br>**Bottom row (POST):** Processes a booking submission<br>Webhook → Calculate End Time → Create Calendar Event → Format Email Data → Send Confirmation → Show Success Page<br>### Quick Start<br>1. Connect Google Calendar OAuth2<br>2. Connect Gmail OAuth2<br>3. Adjust availability settings in "Calculate Available Slots"<br>4. Activate & visit: `<your-n8n-url>/webhook/booking` |
| Respond to Webhook | Respond to Webhook | Returns the generated booking page to the browser | Build HTML Form |  |  |
| Webhook - Submit Form | Webhook | Public entry point for booking form submissions via POST |  | Code in JavaScript |  |
| Code in JavaScript | Code | Parses selected slot and computes end time | Webhook - Submit Form | Create Calendar Event |  |
| Create Calendar Event | Google Calendar | Creates the booked event in Google Calendar | Code in JavaScript | Format Email Data |  |
| Format Email Data | Code | Combines calendar event output with original form data for email rendering | Create Calendar Event | Send Confirmation Email |  |
| Send Confirmation Email | Gmail | Sends HTML confirmation email to the customer | Format Email Data | Show Success Message |  |
| Show Success Message | Respond to Webhook | Returns a success HTML page after booking completion | Send Confirmation Email |  |  |
| Sticky Note | Sticky Note | Embedded operational documentation |  |  | ## Self-Hosted Booking Form with Google Calendar<br>This workflow has **two paths**:<br>**Top row (GET):** Serves the booking form<br>Webhook → Get Calendar Events → Calculate Available Slots → Build HTML Form → Respond<br>**Bottom row (POST):** Processes a booking submission<br>Webhook → Calculate End Time → Create Calendar Event → Format Email Data → Send Confirmation → Show Success Page<br>### Quick Start<br>1. Connect Google Calendar OAuth2<br>2. Connect Gmail OAuth2<br>3. Adjust availability settings in "Calculate Available Slots"<br>4. Activate & visit: `<your-n8n-url>/webhook/booking` |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add a Sticky Note** and paste the following operational summary:
   - Self-hosted booking form with Google Calendar
   - Top row for GET form rendering
   - Bottom row for POST submission processing
   - Connect Google Calendar OAuth2 and Gmail OAuth2
   - Visit `/webhook/booking` after activation

## Build the GET path

3. **Add a Webhook node** named **Webhook - Show Form**.
   - Path: `booking`
   - Response Mode: `Using Respond to Webhook Node`
   - Leave it as the form-serving endpoint for browser access

4. **Add a Google Calendar node** named **Get Calendar Events**.
   - Credential: Google Calendar OAuth2
   - Operation: **Get Many / Get All events**
   - Calendar: `primary`
   - Return All: enabled
   - Set time window:
     - `Time Min`: `{{$now.toISO()}}`
     - `Time Max`: `{{$now.plus({ days: 30 }).toISO()}}`
   - Connect **Webhook - Show Form → Get Calendar Events**

5. **Add a Code node** named **Calculate Available Slots**.
   - Paste logic that:
     - reads all calendar events
     - filters out all-day and >24h events
     - scans the next 30 days
     - keeps only Monday–Friday
     - generates 30-minute slots from 09:00 to 14:00
     - marks overlapping slots unavailable
     - emits slot items with:
       - `value`: ISO datetime with timezone offset
       - `label`: display text
   - Current workflow assumptions:
     - working days `[1,2,3,4,5]`
     - `startHour = 9`
     - `endHour = 14`
     - `slotDuration = 30`
     - timezone offset hardcoded to `+02:00`
   - Connect **Get Calendar Events → Calculate Available Slots**

6. **Add a Code node** named **Build HTML Form**.
   - Paste code that:
     - reads all slot items from input
     - groups them by date
     - generates full HTML page
     - includes embedded CSS and client-side JavaScript
     - includes a `<form method="POST" action="">`
     - includes fields:
       - hidden `timeslot`
       - `firstName`
       - `lastName`
       - `email`
       - `message`
     - includes date navigation and slot selection UI
   - Connect **Calculate Available Slots → Build HTML Form**

7. **Add a Respond to Webhook node** named **Respond to Webhook**.
   - Respond With: `Text`
   - Response Body: `{{$json.html}}`
   - Add response header:
     - `Content-Type: text/html`
   - Connect **Build HTML Form → Respond to Webhook**

## Build the POST path

8. **Add a second Webhook node** named **Webhook - Submit Form**.
   - Path: `booking`
   - HTTP Method: `POST`
   - Response Mode: `Using Respond to Webhook Node`
   - Connect nothing into it; it is a second entry point

9. **Add a Code node** named **Code in JavaScript**.
   - Configure it to:
     - read `const webhookData = $input.first().json.body`
     - extract `webhookData.timeslot`
     - validate format like `YYYY-MM-DDTHH:mm:ss+02:00`
     - compute `endTime` by adding 30 minutes
     - return:
       - original body
       - `startTime`
       - `endTime`
   - Connect **Webhook - Submit Form → Code in JavaScript**

10. **Add a Google Calendar node** named **Create Calendar Event**.
    - Credential: Google Calendar OAuth2
    - Calendar: `primary`
    - Operation: create event
    - Start: `{{$json.body.timeslot}}`
    - End: `{{$json.body.endTime}}`
    - Summary:
      - `Meeting with {{ $json.body.firstName }} {{ $json.body.lastName }}`
    - Description:
      - Name, email, and message from the form
      - Fallback message: `No message provided`
    - Connect **Code in JavaScript → Create Calendar Event**

11. **Add a Code node** named **Format Email Data**.
    - Configure it to:
      - read event output from Google Calendar
      - reference original form data via `$('Webhook - Submit Form').first().json.body`
      - parse `start.dateTime` and `end.dateTime`
      - create:
        - `formattedDate`
        - `formattedTime`
      - merge these with:
        - `firstName`
        - `lastName`
        - `email`
        - `message`
    - Connect **Create Calendar Event → Format Email Data**

12. **Add a Gmail node** named **Send Confirmation Email**.
    - Credential: Gmail OAuth2
    - To: `{{$json.email}}`
    - Subject: `✅ Appointment Confirmed - {{ $json.formattedDate }}`
    - Message type: HTML
    - Email body should include:
      - greeting with first/last name
      - formatted date and time
      - 30-minute duration
      - customer message
      - CTA/button linking to `{{$json.htmlLink}}`
    - Connect **Format Email Data → Send Confirmation Email**

13. **Add a Respond to Webhook node** named **Show Success Message**.
    - Respond With: `Text`
    - Set header:
      - `Content-Type: text/html`
    - Response body: static branded success HTML
    - Connect **Send Confirmation Email → Show Success Message**

## Credentials

14. **Configure Google Calendar OAuth2 credentials**.
    - Use a Google account with calendar access
    - Ensure the selected calendar (`primary`) is writable
    - Reuse the same credential for:
      - `Get Calendar Events`
      - `Create Calendar Event`

15. **Configure Gmail OAuth2 credentials**.
    - Use a Google account authorized to send email
    - Attach this credential to `Send Confirmation Email`

## Activation and testing

16. **Activate the workflow**.
17. Open the production URL:
    - `<your-n8n-url>/webhook/booking`
18. Verify:
    - GET loads the booking page
    - slot selection works
    - POST creates a calendar event
    - user receives the confirmation email
    - browser receives the success page

## Input / Output expectations

19. **GET path input/output**
    - Input: browser request
    - Output: HTML page containing available slots and form fields

20. **POST path expected form fields**
    - `timeslot` as ISO timestamp with timezone offset
    - `firstName`
    - `lastName`
    - `email`
    - `message` optional

21. **POST path output**
    - Side effect 1: Google Calendar event created
    - Side effect 2: Gmail confirmation sent
    - HTTP response: success HTML page

## Important implementation cautions

22. If you rebuild exactly as provided, keep in mind:
    - hardcoded timezone `+02:00` may not handle DST correctly
    - no lock/recheck exists to prevent double-booking race conditions
    - no validation node exists for email format beyond browser-side form validation
    - empty-slot handling in `Build HTML Form` should ideally be improved so `{ noSlots: true }` is handled explicitly

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed as a self-hosted booking form backed by Google Calendar and Gmail. | Canvas documentation |
| Two logical paths are used: GET for form display and POST for form submission processing. | Canvas documentation |
| Quick start: connect Google Calendar OAuth2, connect Gmail OAuth2, adjust availability settings in `Calculate Available Slots`, activate the workflow, then visit `<your-n8n-url>/webhook/booking`. | Canvas documentation |
| Availability logic is currently configured in the `Calculate Available Slots` Code node. | Internal implementation note |
| The public endpoint path used by both entry points is `/booking`, with behavior split by HTTP method. | Workflow architecture note |