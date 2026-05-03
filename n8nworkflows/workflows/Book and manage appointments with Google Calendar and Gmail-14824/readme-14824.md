Book and manage appointments with Google Calendar and Gmail

https://n8nworkflows.xyz/workflows/book-and-manage-appointments-with-google-calendar-and-gmail-14824


# Book and manage appointments with Google Calendar and Gmail

# 1. Workflow Overview

This workflow handles appointment booking requests received through an HTTP webhook, checks Google Calendar for availability, creates a calendar event when the requested slot is free, sends confirmation emails through Gmail, and schedules reminder emails 24 hours and 1 hour before the appointment. If the requested time is not available, it proposes alternative slots and informs the requester.

The workflow is organized into the following logical blocks:

## 1.1 Input Reception
Receives a booking request via webhook and passes it into the workflow.

## 1.2 Configuration
Defines reusable workflow settings such as Google Calendar ID, appointment duration, business hours, and sender email.

## 1.3 Data Parsing and Validation
Extracts booking fields from the webhook payload and validates required fields such as name, email, requested date, and requested time.

## 1.4 Calendar Availability Check
Reads events from Google Calendar and compares them with the requested appointment time to detect conflicts.

## 1.5 Availability Decision
Branches the workflow depending on whether the slot is available.

## 1.6 Booking Confirmation Flow
Creates the Google Calendar event, sends a confirmation email, and returns a success response to the webhook caller.

## 1.7 Reminder Scheduling
After a successful booking, schedules and sends reminder emails 24 hours and 1 hour before the appointment.

## 1.8 Alternative Slot Flow
If the requested slot is unavailable, generates alternative slots, emails them to the requester, and returns an unavailable response.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block exposes the workflow as an HTTP endpoint. It accepts booking requests and starts the process.

**Nodes Involved:**  
- Booking Request Webhook

### Node Details

#### Booking Request Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point node that receives HTTP POST requests.
- **Configuration choices:**  
  - Path: `booking`
  - HTTP Method: `POST`
  - Response Mode: `lastNode`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Workflow Configuration`
- **Version-specific requirements:**  
  Uses node type version `2.1`.
- **Edge cases or potential failure types:**  
  - Incorrect HTTP method will not trigger the workflow.
  - Invalid or unexpected body shape may later cause parsing/validation issues.
  - Because `responseMode` is `lastNode`, response behavior may be confusing since `Respond to Webhook` nodes are also present later.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Configuration

**Overview:**  
This block centralizes workflow-level settings. It ensures later nodes can read key operational values without hardcoding them repeatedly.

**Nodes Involved:**  
- Workflow Configuration

### Node Details

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds configuration values to the current item.
- **Configuration choices:**  
  Sets:
  - `calendarId`: placeholder for the Google Calendar ID
  - `appointmentDuration`: `60`
  - `businessHoursStart`: `9`
  - `businessHoursEnd`: `17`
  - `senderEmail`: placeholder sender email
  Keeps existing fields with `includeOtherFields: true`.
- **Key expressions or variables used:**  
  Later nodes reference:
  - `$('Workflow Configuration').first().json.calendarId`
  - `$('Workflow Configuration').first().json.appointmentDuration`
  - `$('Workflow Configuration').first().json.senderEmail`
- **Input and output connections:**  
  - Input: `Booking Request Webhook`
  - Output: `Parse Booking Data`
- **Version-specific requirements:**  
  Uses Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Placeholder values must be replaced before production use.
  - Invalid calendar ID causes Google Calendar authentication/API lookup failures.
  - Sender email is used as sender name in Gmail nodes, which may not produce the intended email display name.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Data Parsing and Validation

**Overview:**  
This block extracts booking data from the incoming request and performs basic validation. It prepares a normalized structure for downstream nodes.

**Nodes Involved:**  
- Parse Booking Data

### Node Details

#### Parse Booking Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript logic to parse and validate incoming booking data.
- **Configuration choices:**  
  Reads either `item.json.body` or `item.json`, then extracts:
  - `name`
  - `email`
  - `requestedDate`
  - `requestedTime`
  - `notes`
  
  Validation rules:
  - name required
  - email required and regex-validated
  - requestedDate required
  - requestedTime required

  Returns:
  - `name`
  - `email`
  - `requestedDate`
  - `requestedTime`
  - `notes`
  - `isValid`
  - `validationErrors`
  - `rawData`
- **Key expressions or variables used:**  
  Internal JavaScript uses `item.json.body || item.json`.
- **Input and output connections:**  
  - Input: `Workflow Configuration`
  - Output: `Check Calendar Availability`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - Major structural issue: this node does **not** produce fields such as `requestedDateTime`, `startTime`, `endTime`, or `bookingTime`, but later nodes expect them.
  - Validation result is not used for branching; invalid requests still continue into the calendar check flow.
  - Date and time are not converted into a single ISO timestamp.
  - Time zone handling is absent.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Calendar Availability Check

**Overview:**  
This block fetches calendar events and compares them against the requested appointment interval to determine whether the requested slot is free.

**Nodes Involved:**  
- Check Calendar Availability
- Check for Conflicts

### Node Details

#### Check Calendar Availability
- **Type and technical role:** `n8n-nodes-base.googleCalendar`  
  Retrieves calendar events from a specified Google Calendar.
- **Configuration choices:**  
  - Operation: `getAll`
  - Return All: `true`
  - Calendar ID is read dynamically from the configuration node
- **Key expressions or variables used:**  
  - `={{ $('Workflow Configuration').first().json.calendarId }}`
- **Input and output connections:**  
  - Input: `Parse Booking Data`
  - Output: `Check for Conflicts`
- **Version-specific requirements:**  
  Uses Google Calendar node version `1.3`.
- **Edge cases or potential failure types:**  
  - Requires valid Google Calendar credentials.
  - Fetches all events without an explicit time window; this can be inefficient and may hit performance limits on large calendars.
  - Missing start/end range filtering increases conflict-check complexity and API volume.
- **Sub-workflow reference:**  
  None.

#### Check for Conflicts
- **Type and technical role:** `n8n-nodes-base.code`  
  Compares requested time against fetched calendar events.
- **Configuration choices:**  
  Reads:
  - all events from `Check Calendar Availability`
  - `startTime` and `endTime` from `Parse Booking Data`

  Logic:
  - Converts event and requested times to `Date`
  - Detects overlap with condition:
    - requested start `<` event end
    - requested end `>` event start
  Returns:
  - `isAvailable`
  - `hasConflict`
  - `requestedStart`
  - `requestedEnd`
  - `conflictingEvents`
  - `totalEventsChecked`
- **Key expressions or variables used:**  
  - `$('Check Calendar Availability').all()`
  - `$('Parse Booking Data').item.json.startTime`
  - `$('Parse Booking Data').item.json.endTime`
- **Input and output connections:**  
  - Input: `Check Calendar Availability`
  - Output: `Is Available?`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - Critical issue: `startTime` and `endTime` are never created upstream, so `new Date(undefined)` will produce invalid dates.
  - All-day events may behave differently because they may use `date` rather than `dateTime`.
  - Time zone mismatches may cause false positives or negatives.
  - Empty calendar event lists are handled safely, but only if requested times are valid.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Availability Decision

**Overview:**  
This block routes execution either toward booking confirmation or toward alternative slot generation based on the conflict check result.

**Nodes Involved:**  
- Is Available?

### Node Details

#### Is Available?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branch node.
- **Configuration choices:**  
  Checks whether `$('Check for Conflicts').item.json.isAvailable` is `true`.
- **Key expressions or variables used:**  
  - `={{ $('Check for Conflicts').item.json.isAvailable }}`
- **Input and output connections:**  
  - Input: `Check for Conflicts`
  - True output: `Create Calendar Event`
  - False output: `Find Alternative Slots`
- **Version-specific requirements:**  
  Uses If node version `2.3`.
- **Edge cases or potential failure types:**  
  - If `isAvailable` is missing or invalid, branch behavior may be incorrect.
  - Since earlier validation is not enforced, malformed requests may still reach this decision point.
- **Sub-workflow reference:**  
  None.

---

## 2.6 Booking Confirmation Flow

**Overview:**  
This block creates the appointment in Google Calendar, sends a confirmation email to the requester, and returns a success response.

**Nodes Involved:**  
- Create Calendar Event
- Send Confirmation Email
- Respond Success

### Node Details

#### Create Calendar Event
- **Type and technical role:** `n8n-nodes-base.googleCalendar`  
  Creates a new event in Google Calendar.
- **Configuration choices:**  
  - Start: `={{ $json.requestedDateTime }}`
  - End: calculated from `requestedDateTime + appointmentDuration`
  - Calendar ID from configuration
  - Summary: `Appointment with {{ $json.name }}`
  - Attendees: requester email
  - Description: notes
- **Key expressions or variables used:**  
  - `={{ $json.requestedDateTime }}`
  - `={{ new Date(new Date($json.requestedDateTime).getTime() + $('Workflow Configuration').first().json.appointmentDuration*60*1000).toISOString() }}`
  - `={{ $('Workflow Configuration').first().json.calendarId }}`
- **Input and output connections:**  
  - Input: `Is Available?` (true branch)
  - Output: `Send Confirmation Email`
- **Version-specific requirements:**  
  Uses Google Calendar node version `1.3`.
- **Edge cases or potential failure types:**  
  - Critical issue: `requestedDateTime` is not created upstream.
  - Invalid date format causes event creation failure.
  - Missing/invalid Google Calendar credentials or insufficient permissions.
  - Attendee invitation behavior depends on calendar settings and Google account permissions.
- **Sub-workflow reference:**  
  None.

#### Send Confirmation Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends confirmation email through Gmail.
- **Configuration choices:**  
  - Recipient: requester email
  - Subject includes localized appointment datetime
  - HTML body contains appointment details and link to the created calendar event
  - Uses configured sender email as `senderName`
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `{{ new Date($json.requestedDateTime).toLocaleString() }}`
  - `{{ $json.duration || '30' }}`
  - `{{ $('Create Calendar Event').item.json.htmlLink }}`
  - `={{ $('Workflow Configuration').first().json.senderEmail }}`
- **Input and output connections:**  
  - Input: `Create Calendar Event`
  - Outputs:
    - `Respond Success`
    - `Wait 24 Hours Before`
- **Version-specific requirements:**  
  Uses Gmail node version `2.2`.
- **Edge cases or potential failure types:**  
  - Gmail credentials required.
  - The node relies on fields like `email`, `name`, and `requestedDateTime`, but after Google Calendar creation the active item may not contain all original fields unless preserved as expected.
  - `duration` is referenced but is not explicitly set upstream.
  - `senderName` is populated from an email address, not a human-readable display name.
- **Sub-workflow reference:**  
  None.

#### Respond Success
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends a JSON success response back to the HTTP caller.
- **Configuration choices:**  
  Returns:
  ```json
  {
    "success": true,
    "message": "Booking confirmed successfully"
  }
  ```
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Send Confirmation Email`
  - Output: none
- **Version-specific requirements:**  
  Uses Respond to Webhook version `1.5`.
- **Edge cases or potential failure types:**  
  - Potential design inconsistency with webhook `responseMode: lastNode`.
  - If reminder branch continues longer, the “last node” behavior may not align with intended immediate webhook response.
- **Sub-workflow reference:**  
  None.

---

## 2.7 Reminder Scheduling

**Overview:**  
This block delays execution until specific times before the appointment and sends reminder emails. It is only triggered after a successful booking confirmation.

**Nodes Involved:**  
- Wait 24 Hours Before
- Send 24h Reminder
- Wait 1 Hour Before
- Send 1h Reminder

### Node Details

#### Wait 24 Hours Before
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution until 24 hours before the appointment.
- **Configuration choices:**  
  - Resume at specific time:
    `={{ new Date(new Date($json.requestedDateTime).getTime() - 24*60*60*1000).toISOString() }}`
- **Key expressions or variables used:**  
  - `requestedDateTime`
- **Input and output connections:**  
  - Input: `Send Confirmation Email`
  - Output: `Send 24h Reminder`
- **Version-specific requirements:**  
  Uses Wait node version `1.1`.
- **Edge cases or potential failure types:**  
  - If the appointment is less than 24 hours away, the calculated time may be in the past.
  - Requires persistent execution support in n8n.
  - Again depends on missing `requestedDateTime`.
- **Sub-workflow reference:**  
  None.

#### Send 24h Reminder
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the reminder email one day before the appointment.
- **Configuration choices:**  
  - Recipient: requester email
  - Plain-text style message
  - Includes date/time, service, and duration
  - Sender name uses configured sender email
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `{{ new Date($json.requestedDateTime).toLocaleString() }}`
  - `{{ $json.service }}`
  - `{{ $json.duration }}`
  - `={{ $('Workflow Configuration').first().json.senderEmail }}`
- **Input and output connections:**  
  - Input: `Wait 24 Hours Before`
  - Output: `Wait 1 Hour Before`
- **Version-specific requirements:**  
  Uses Gmail node version `2.2`.
- **Edge cases or potential failure types:**  
  - `service` and `duration` are not guaranteed to exist.
  - Gmail auth/quotas may block sending.
  - Missing `email` or malformed carried-forward item data will fail message delivery.
- **Sub-workflow reference:**  
  None.

#### Wait 1 Hour Before
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution until one hour before the appointment.
- **Configuration choices:**  
  - Resume at specific time:
    `={{ new Date(new Date($json.requestedDateTime).getTime() - 60*60*1000).toISOString() }}`
- **Key expressions or variables used:**  
  - `requestedDateTime`
- **Input and output connections:**  
  - Input: `Send 24h Reminder`
  - Output: `Send 1h Reminder`
- **Version-specific requirements:**  
  Uses Wait node version `1.1`.
- **Edge cases or potential failure types:**  
  - If the appointment is less than one hour away, scheduled time may be in the past.
  - Requires persisted wait execution support.
- **Sub-workflow reference:**  
  None.

#### Send 1h Reminder
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the final reminder one hour before the appointment.
- **Configuration choices:**  
  - Recipient: requester email
  - HTML message
  - Includes date/time, duration, and service
  - Uses `senderName` from configuration
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `{{ new Date($json.requestedDateTime).toLocaleString() }}`
  - `{{ $json.duration }}`
  - `{{ $json.service }}`
  - `{{ $('Workflow Configuration').first().json.senderName }}`
- **Input and output connections:**  
  - Input: `Wait 1 Hour Before`
  - Output: none
- **Version-specific requirements:**  
  Uses Gmail node version `2.2`.
- **Edge cases or potential failure types:**  
  - Critical issue: configuration node defines `senderEmail`, but this node references `senderName`, which does not exist there.
  - `service` and `duration` may be undefined.
  - HTML rendering is email-client dependent.
- **Sub-workflow reference:**  
  None.

---

## 2.8 Alternative Slot Flow

**Overview:**  
If the requested appointment conflicts with existing calendar events, this block proposes up to three alternative time slots and notifies the requester by email, then responds to the webhook caller.

**Nodes Involved:**  
- Find Alternative Slots
- Send Alternative Slots Email
- Respond Unavailable

### Node Details

#### Find Alternative Slots
- **Type and technical role:** `n8n-nodes-base.code`  
  Generates alternative appointment slots within business hours.
- **Configuration choices:**  
  - Reads requested booking time from `$('Parse Booking Data').item.json.bookingTime`
  - Hardcodes business hours:
    - start `9`
    - end `17`
  - Hardcodes slot duration: `60`
  - Iterates forward from the next hour until 3 slots are found or 7 days have passed
  - Restricts slots to weekdays and business hours
- **Key expressions or variables used:**  
  - `$('Parse Booking Data').item.json.bookingTime`
- **Input and output connections:**  
  - Input: `Is Available?` (false branch)
  - Output: `Send Alternative Slots Email`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - Critical issue: `bookingTime` is not created upstream.
  - Does not actually check Google Calendar availability for alternative slots; it only generates candidate hours.
  - Ignores configured business hours from `Workflow Configuration`.
  - Starts from current time rather than from the originally requested time.
- **Sub-workflow reference:**  
  None.

#### Send Alternative Slots Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email listing alternative appointment options.
- **Configuration choices:**  
  - Recipient: requester email
  - Subject: appointment unavailable
  - Body includes requested datetime and alternative slots
  - Uses configured sender email as sender name
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `{{ $json.name }}`
  - `{{ $json.requestedDateTime }}`
  - `{{ $json.alternativeSlots }}`
  - `={{ $('Workflow Configuration').first().json.senderEmail }}`
- **Input and output connections:**  
  - Input: `Find Alternative Slots`
  - Output: `Respond Unavailable`
- **Version-specific requirements:**  
  Uses Gmail node version `2.2`.
- **Edge cases or potential failure types:**  
  - Input mismatch: `Find Alternative Slots` returns `alternativeSlots`, `originalRequest`, and `message`, but not necessarily `email`, `name`, or `requestedDateTime`.
  - Alternative slots array may render poorly in plain text unless formatted explicitly.
- **Sub-workflow reference:**  
  None.

#### Respond Unavailable
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns a JSON response indicating the slot is unavailable and includes alternative slots.
- **Configuration choices:**  
  Returns JSON with:
  - `status: "unavailable"`
  - message
  - `alternativeSlots`
- **Key expressions or variables used:**  
  - `{{ $json.alternativeSlots }}`
- **Input and output connections:**  
  - Input: `Send Alternative Slots Email`
  - Output: none
- **Version-specific requirements:**  
  Uses Respond to Webhook version `1.5`.
- **Edge cases or potential failure types:**  
  - Expression-based JSON body may break if `alternativeSlots` is not serializable in the expected format.
  - Same webhook response mode inconsistency applies here.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Booking Request Webhook | Webhook | Receives booking requests via HTTP POST |  | Workflow Configuration | ## Input Layer<br>Receive booking request via webhook |
| Workflow Configuration | Set | Stores reusable configuration values | Booking Request Webhook | Parse Booking Data | ## Configuration<br>Set calendar, duration, and business hours |
| Parse Booking Data | Code | Parses and validates incoming booking fields | Workflow Configuration | Check Calendar Availability | ## Data Validation<br>Parse and validate booking request fields |
| Check Calendar Availability | Google Calendar | Fetches calendar events to inspect availability | Parse Booking Data | Check for Conflicts | ## Availability Check<br>Fetch calendar events and check conflicts |
| Check for Conflicts | Code | Detects overlaps between requested slot and existing events | Check Calendar Availability | Is Available? | ## Availability Check<br>Fetch calendar events and check conflicts |
| Is Available? | If | Routes based on availability result | Check for Conflicts | Create Calendar Event; Find Alternative Slots | ## Decision Logic<br>Route based on availability status |
| Create Calendar Event | Google Calendar | Creates the appointment event in Google Calendar | Is Available? | Send Confirmation Email | ## Booking Flow<br>Create calendar event and confirm booking |
| Send Confirmation Email | Gmail | Sends booking confirmation email | Create Calendar Event | Respond Success; Wait 24 Hours Before | ## Confirmation<br>Send confirmation email and response |
| Respond Success | Respond to Webhook | Returns success JSON response | Send Confirmation Email |  | ## Confirmation<br>Send confirmation email and response |
| Wait 24 Hours Before | Wait | Delays execution until 24 hours before appointment | Send Confirmation Email | Send 24h Reminder | ## Reminder System<br>Send 24h and 1h reminder emails. After a booking is confirmed, the workflow schedules two timed reminders:<br>- A 24-hour reminder to notify the user one day before<br>- A 1-hour reminder for last-minute confirmation |
| Send 24h Reminder | Gmail | Sends reminder email 24 hours before appointment | Wait 24 Hours Before | Wait 1 Hour Before | ## Reminder System<br>Send 24h and 1h reminder emails. After a booking is confirmed, the workflow schedules two timed reminders:<br>- A 24-hour reminder to notify the user one day before<br>- A 1-hour reminder for last-minute confirmation |
| Wait 1 Hour Before | Wait | Delays execution until 1 hour before appointment | Send 24h Reminder | Send 1h Reminder | ## Reminder System<br>Send 24h and 1h reminder emails. After a booking is confirmed, the workflow schedules two timed reminders:<br>- A 24-hour reminder to notify the user one day before<br>- A 1-hour reminder for last-minute confirmation |
| Send 1h Reminder | Gmail | Sends reminder email 1 hour before appointment | Wait 1 Hour Before |  | ## Reminder System<br>Send 24h and 1h reminder emails. After a booking is confirmed, the workflow schedules two timed reminders:<br>- A 24-hour reminder to notify the user one day before<br>- A 1-hour reminder for last-minute confirmation |
| Find Alternative Slots | Code | Generates fallback appointment options | Is Available? | Send Alternative Slots Email | ## Alternative Flow<br>Suggest available time slots if unavailable |
| Send Alternative Slots Email | Gmail | Sends alternative slot suggestions by email | Find Alternative Slots | Respond Unavailable | ## Alternative Flow<br>Suggest available time slots if unavailable |
| Respond Unavailable | Respond to Webhook | Returns unavailable response with alternatives | Send Alternative Slots Email |  | ## Alternative Flow<br>Suggest available time slots if unavailable |
| Sticky Note1 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note2 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note3 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note4 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note5 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note6 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note7 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note8 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note9 | Sticky Note | Documentation/comment node |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is the exact build order needed to recreate this workflow in n8n, while also highlighting the adjustments needed to make it operational.

## 1. Create the webhook trigger
1. Add a **Webhook** node.
2. Name it **Booking Request Webhook**.
3. Set:
   - **HTTP Method**: `POST`
   - **Path**: `booking`
   - **Response Mode**: preferably `Using Respond to Webhook Node` if you want explicit response control.  
     The provided workflow uses `Last Node`, but this conflicts conceptually with the presence of `Respond to Webhook` nodes.
4. Expect incoming JSON such as:
   - `name`
   - `email`
   - `requestedDate`
   - `requestedTime`
   - `notes`

## 2. Add configuration values
5. Add a **Set** node after the webhook.
6. Name it **Workflow Configuration**.
7. Enable **Include Other Input Fields**.
8. Add fields:
   - `calendarId` as string
   - `appointmentDuration` as number, default `60`
   - `businessHoursStart` as number, default `9`
   - `businessHoursEnd` as number, default `17`
   - `senderEmail` as string
9. Replace placeholders with real values:
   - Google Calendar ID
   - Gmail sender address

## 3. Parse and validate the booking payload
10. Add a **Code** node.
11. Name it **Parse Booking Data**.
12. Connect `Workflow Configuration -> Parse Booking Data`.
13. Paste logic equivalent to the supplied script, but to make the workflow actually work, also generate:
   - `requestedDateTime`
   - `startTime`
   - `endTime`
   - `bookingTime`
   - `duration`
14. Recommended computed values:
   - `requestedDateTime = new Date(\`${requestedDate}T${requestedTime}:00\`).toISOString()`
   - `startTime = requestedDateTime`
   - `endTime = requestedDateTime + appointmentDuration`
   - `bookingTime = requestedDateTime`
15. Preserve:
   - `name`
   - `email`
   - `notes`
   - `validationErrors`
   - `isValid`

## 4. Add validation branch if you want production reliability
16. Add an **If** node after parsing if you want to stop invalid requests.
17. Condition:
   - `isValid` is `true`
18. On false branch, add **Respond to Webhook** with HTTP-style JSON error content.
19. Note: the provided workflow does not include this branch, but it should.

## 5. Check Google Calendar events
20. Add a **Google Calendar** node.
21. Name it **Check Calendar Availability**.
22. Connect it from **Parse Booking Data** or from the validation-success branch.
23. Configure credentials using Google Calendar OAuth2.
24. Set:
   - **Operation**: `Get Many` / `Get All`
   - **Calendar**: expression using the configuration node:
     `$('Workflow Configuration').first().json.calendarId`
25. For a better implementation, add time filtering around the requested day.  
   The provided workflow does not do this, but it should.

## 6. Detect conflicts
26. Add a **Code** node named **Check for Conflicts**.
27. Connect `Check Calendar Availability -> Check for Conflicts`.
28. Use logic that:
   - reads all fetched calendar events
   - reads `startTime` and `endTime` from parsed booking data
   - compares intervals for overlap
29. Return:
   - `isAvailable`
   - `hasConflict`
   - `conflictingEvents`
   - requested times

## 7. Add routing decision
30. Add an **If** node named **Is Available?**
31. Connect `Check for Conflicts -> Is Available?`
32. Condition:
   - boolean true on `$('Check for Conflicts').item.json.isAvailable`

## 8. Create the booking event
33. Add a **Google Calendar** node named **Create Calendar Event** on the true branch.
34. Connect `Is Available?` true output to it.
35. Configure Google Calendar credentials.
36. Set:
   - **Calendar ID** from configuration
   - **Start**: `{{$json.requestedDateTime}}`
   - **End**: calculated from start plus appointment duration
   - **Summary**: `Appointment with {{$json.name}}`
   - **Description**: `{{$json.notes}}`
   - **Attendees**: requester email
37. Ensure item data still includes `name`, `email`, and `requestedDateTime`.

## 9. Send confirmation email
38. Add a **Gmail** node named **Send Confirmation Email**.
39. Connect `Create Calendar Event -> Send Confirmation Email`.
40. Configure Gmail OAuth2 credentials.
41. Set:
   - **To**: requester email
   - **Subject**: appointment confirmed
   - **Message**: HTML email containing:
     - name
     - appointment date/time
     - duration
     - event link from the calendar node
42. Set sender display carefully.  
   The provided workflow uses `senderEmail` as `senderName`, which is workable but not ideal.

## 10. Respond to the webhook with success
43. Add a **Respond to Webhook** node named **Respond Success**.
44. Connect it from **Send Confirmation Email**.
45. Set response type to **JSON**.
46. Use a body like:
   ```json
   {
     "success": true,
     "message": "Booking confirmed successfully"
   }
   ```

## 11. Schedule the 24-hour reminder
47. Add a **Wait** node named **Wait 24 Hours Before**.
48. Also connect it from **Send Confirmation Email** as a second outgoing branch.
49. Set **Resume** to **Specific Time**.
50. Use:
   - appointment time minus 24 hours
51. Confirm your n8n instance supports persisted waiting executions.

## 12. Send the 24-hour reminder email
52. Add a **Gmail** node named **Send 24h Reminder**.
53. Connect `Wait 24 Hours Before -> Send 24h Reminder`.
54. Configure recipient as requester email.
55. Compose reminder body with:
   - appointment datetime
   - duration
   - optional service name
56. If you want `service`, add it to the original parsed payload first.  
   The current workflow references it without defining it.

## 13. Schedule the 1-hour reminder
57. Add a **Wait** node named **Wait 1 Hour Before**.
58. Connect `Send 24h Reminder -> Wait 1 Hour Before`.
59. Set **Resume** to **Specific Time**.
60. Use:
   - appointment time minus 1 hour

## 14. Send the 1-hour reminder
61. Add a **Gmail** node named **Send 1h Reminder**.
62. Connect `Wait 1 Hour Before -> Send 1h Reminder`.
63. Configure Gmail credentials.
64. Build an HTML reminder message.
65. Correct the configuration reference:
   - use `senderEmail` or add a real `senderName` field in the configuration node.
66. The provided workflow references `$('Workflow Configuration').first().json.senderName`, which is not defined and should be fixed.

## 15. Build the unavailable branch
67. Add a **Code** node named **Find Alternative Slots** on the false branch from **Is Available?**
68. Connect `Is Available?` false output to it.
69. Use logic to generate up to 3 alternative slots.
70. For a proper build, use:
   - original requested datetime
   - configured business hours from `Workflow Configuration`
   - optional additional Google Calendar checks on generated alternatives
71. The supplied workflow does not verify that alternatives are truly free in Google Calendar.

## 16. Send alternative slots by email
72. Add a **Gmail** node named **Send Alternative Slots Email**.
73. Connect `Find Alternative Slots -> Send Alternative Slots Email`.
74. Configure:
   - recipient email
   - unavailable subject
   - body listing alternative slots in a human-readable format
75. Make sure `Find Alternative Slots` outputs:
   - `email`
   - `name`
   - `requestedDateTime`
   - `alternativeSlots`
   Otherwise the email expressions will fail.

## 17. Respond to the webhook with unavailable status
76. Add a **Respond to Webhook** node named **Respond Unavailable**.
77. Connect `Send Alternative Slots Email -> Respond Unavailable`.
78. Return JSON:
   - status unavailable
   - message
   - alternativeSlots array

## 18. Add documentation sticky notes
79. Add **Sticky Note** nodes for:
   - Input Layer
   - Configuration
   - Data Validation
   - Availability Check
   - Decision Logic
   - Booking Flow
   - Confirmation
   - Reminder System
   - Alternative Flow
   - Overall explanation and setup steps

## 19. Configure credentials
80. **Google Calendar credential**
   - Use Google OAuth2
   - Grant calendar read/write access
   - Ensure the configured account has access to the target calendar ID
81. **Gmail credential**
   - Use Gmail OAuth2
   - Grant send email permissions
   - Confirm sending quotas and sender account restrictions

## 20. Test input payload
82. Send a POST request to the webhook with a body like:
   ```json
   {
     "name": "Jane Doe",
     "email": "jane@example.com",
     "requestedDate": "2026-04-15",
     "requestedTime": "10:00",
     "notes": "Initial consultation"
   }
   ```
83. Verify:
   - validation passes
   - requested datetime is generated
   - conflict logic works
   - event is created
   - confirmation email is sent
   - wait nodes are scheduled

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates appointment booking by validating requests, checking calendar availability, and scheduling confirmed slots. When a booking request is received via webhook, the system validates input data and checks for conflicts in Google Calendar. If the requested time is available, an event is created, and a confirmation email is sent. If unavailable, alternative time slots are generated and shared with the user. The workflow also sends automated reminders 24 hours and 1 hour before the appointment. | General workflow description |
| Configure webhook endpoint for booking requests | Setup note |
| Add Google Calendar credentials | Setup note |
| Set Gmail credentials for notifications | Setup note |
| Define business hours and appointment duration | Setup note |
| Customize email templates and reminders | Setup note |

## Important implementation observations

- The current workflow structure is good conceptually, but the JSON contains several data-mapping inconsistencies that must be corrected before it can run reliably.
- Missing upstream fields:
  - `requestedDateTime`
  - `startTime`
  - `endTime`
  - `bookingTime`
  - `duration`
  - `service`
- Configuration mismatch:
  - `senderEmail` exists
  - `senderName` is referenced later but not defined
- The alternative slot generator does not verify Google Calendar availability for its proposed slots.
- The validation result is computed but never used to reject invalid requests.
- Using `Webhook.responseMode = lastNode` together with `Respond to Webhook` nodes is not ideal; choose one response strategy and keep it consistent.