Book, manage, and check appointments using Vapi and Google Calendar

https://n8nworkflows.xyz/workflows/book--manage--and-check-appointments-using-vapi-and-google-calendar-12831


# Book, manage, and check appointments using Vapi and Google Calendar

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Book, manage, and check appointments using Vapi and Google Calendar  
**Workflow name (in JSON):** Voice AI Appointment Booking System with Vapi & Google Calendar - Template

This workflow exposes **three HTTP webhook endpoints** designed to be used as **Vapi “custom tools”** for a voice assistant. It integrates with **Google Calendar** to (1) check availability, (2) book appointments, and (3) cancel/reschedule existing appointments. Each endpoint returns a **Vapi-compatible tool response** (a `results[]` array containing `toolCallId` and a `result` message).

### 1.1 Check Availability (Tool 1)
Receives a requested date/time, queries Google Calendar events for that day, determines whether the requested time slot is free, and returns either “available” or “UNAVAILABLE + suggested alternatives”.

### 1.2 Book Appointment (Tool 2)
Receives booking details (name, phone, date, time, etc.), validates required fields, creates a Google Calendar event, and returns a success message (or a validation error message).

### 1.3 Manage Appointments (Tool 3)
Receives an action (`update` or `cancel`) plus the original appointment date/time (and new date/time for rescheduling), locates the corresponding calendar event, updates or deletes it, and returns a status message.

---

## 2. Block-by-Block Analysis

### Block 1 — Tool 1: Check Availability

**Overview:**  
This block is called by Vapi to verify whether a requested appointment time is available on a given date. It pulls all calendar events for the day and compares them to the requested timestamp, returning either “available” or alternative slot suggestions.

**Nodes involved:**
- Check Availability Webhook
- Extract Availability Request
- Get Calendar Events for Date
- Check Time Slot Availability
- Format Available Time Response
- Respond - Time Available
- Format Unavailable Time Response
- Respond - Time Unavailable
- Sticky Note1
- Sticky Note5

#### 2.1 Check Availability Webhook
- **Type / role:** Webhook trigger; entry point for Tool 1.
- **Configuration:**
  - `POST /availability`
  - Response mode: **Respond via “Respond to Webhook” node** (`responseNode`)
  - Webhook ID placeholder: `YOUR_WEBHOOK_ID_CHECK_AVAILABILITY`
- **Input / output:**
  - Output → Extract Availability Request
- **Key expectations (payload):**
  - Expects Vapi-like structure under `body.message.toolCalls[0].function.arguments`
  - Elsewhere in the block, code assumes `body.message.toolCallList[0].id` exists (note the difference).
- **Common failures / edge cases:**
  - If Vapi sends `toolCallList` but not `toolCalls` (or vice versa), downstream expressions may break.
  - If endpoint is hit outside Vapi format, Set node expressions may evaluate to `undefined`.

#### 2.2 Extract Availability Request
- **Type / role:** Set node; extracts requested date/time and builds a formatted datetime string.
- **Configuration choices:**
  - Extracts:
    - `appointmentDate` = `body.message.toolCalls[0].function.arguments.appointmentDate`
    - `appointmentTime` = `...appointmentTime`
    - `requestedDatetime` = Luxon `DateTime.fromISO(date + 'T' + time + ':00', zone: 'America/Vancouver')` formatted as `yyyy-MM-dd'T'HH:mm:ssZZ`
- **Input / output:**
  - Input ← Check Availability Webhook
  - Output → Get Calendar Events for Date
- **Key expressions / variables:**
  - Uses **Luxon DateTime** (n8n supports Luxon in expressions when enabled by n8n environment).
- **Edge cases:**
  - No validation of date/time format; invalid ISO composition can throw or produce invalid strings.
  - Hard-coded timezone: `America/Vancouver` (may not match business location).

#### 2.3 Get Calendar Events for Date
- **Type / role:** Google Calendar node; fetches all events in the target day.
- **Configuration choices:**
  - Operation: `getAll`
  - Calendar: `user@example.com` (placeholder)
  - Time window:
    - `timeMin` = `{{ $json.appointmentDate }}T00:00:00-08:00`
    - `timeMax` = `{{ $json.appointmentDate }}T23:59:59-08:00`
  - Time zone option: `America/Vancouver`
  - `alwaysOutputData: true` (important: ensures downstream node runs even if no events)
- **Input / output:**
  - Input ← Extract Availability Request
  - Output → Check Time Slot Availability
- **Edge cases / failures:**
  - OAuth/credential errors (expired token, missing scopes).
  - DST mismatch risk: hard-coded `-08:00` offset may be wrong in summer (Vancouver often `-07:00`).
  - All-day events may not have `start.dateTime` (they use `start.date`); downstream nodes assume `start.dateTime`.

#### 2.4 Check Time Slot Availability
- **Type / role:** IF node; compares each returned event’s start to requested datetime.
- **Configuration choices:**
  - Condition: `{{ $json.start.dateTime }}` equals `{{ $items('Edit Fields1')[0].json.requestedDatetime }}`
- **Input / output:**
  - Input ← Get Calendar Events for Date (iterates over events)
  - True/False outputs:
    - Output 0 → Format Unavailable Time Response
    - Output 1 → Format Available Time Response
- **Critical issue (high impact):**
  - References node name **`Edit Fields1`**, which does **not exist** in this workflow. It likely should reference **`Extract Availability Request`**.
  - As written, this IF condition will fail evaluation or always be false depending on n8n behavior.
- **Logic issue (functional):**
  - Availability should be based on **overlap**, not exact equality of `start.dateTime`. A conflict could exist even if event starts earlier/later but overlaps the requested time slot.
- **Edge cases:**
  - If there are **0 events**, the IF node won’t receive items unless `alwaysOutputData` produces an empty item; behavior depends on node output structure.
  - All-day events missing `dateTime`.

#### 2.5 Format Available Time Response
- **Type / role:** Code node; formats a Vapi-compatible success response.
- **Configuration choices:**
  - Reads:
    - `editFieldsData` from `Extract Availability Request`
    - `toolCallId` from `Check Availability Webhook` at `body.message.toolCallList[0].id` (fallback `'available-check'`)
  - Returns:
    ```js
    [{ json: { results: [{ toolCallId, result: "..." }] } }]
    ```
- **Input / output:**
  - Input ← Check Time Slot Availability (available branch)
  - Output → Respond - Time Available
- **Edge cases:**
  - Mixed usage of `toolCalls` vs `toolCallList` across the workflow may cause missing toolCallId.
  - If webhook doesn’t include `toolCallList`, fallback id is used (may be unacceptable to Vapi).

#### 2.6 Respond - Time Available
- **Type / role:** Respond to Webhook; sends the final HTTP response.
- **Configuration:** default options.
- **Input / output:**
  - Input ← Format Available Time Response
  - Output: ends request.

#### 2.7 Format Unavailable Time Response
- **Type / role:** Code node; generates alternative time slot suggestions and returns Vapi response.
- **Configuration choices & logic:**
  - Pulls all events: `$('Get Calendar Events for Date').all().map(item => item.json)`
  - Reads:
    - `requestedDate`, `requestedTime` from Extract Availability Request
    - `duration`, `timeZone` from Extract Availability Request **(but these fields are not set there)**
    - `toolCallId` from `Check Availability Webhook` at `body.message.toolCallList[0].id`
  - Generates slots from **09:00 to 18:00**, 30-minute increments, and filters those that do not overlap existing events.
  - Suggests first 3 available slots.
  - Returns an object shaped like:
    ```js
    return { results: [{ toolCallId, result: "UNAVAILABLE - ..." }] };
    ```
    (Note: unlike other code nodes, this does **not** wrap in `[{ json: ... }]`.)
- **Input / output:**
  - Input ← Check Time Slot Availability (unavailable branch)
  - Output → Respond - Time Unavailable
- **Critical issues:**
  - **Return shape inconsistency:** n8n Code node should usually return an **array of items** (`[{json: ...}]`). Returning a plain object can fail or be auto-wrapped depending on n8n version; it is not consistent with other code nodes here.
  - `duration` and `timeZone` are referenced but never created in `Extract Availability Request`. `timeZone` becomes `undefined`, causing `toLocaleTimeString` with `timeZone: undefined` to fall back to server timezone (inconsistent results).
  - Assumes all events have `start.dateTime` and `end.dateTime`.
- **Edge cases:**
  - If calendar has many events, slot checking is O(slots × events) (still fine for a single day).
  - If requested date is invalid, `new Date(`${date}T...`)` becomes invalid.

#### 2.8 Respond - Time Unavailable
- **Type / role:** Respond to Webhook; returns the “unavailable + suggestions” response.
- **Input / output:**
  - Input ← Format Unavailable Time Response

**Sticky notes in this block:**
- **Sticky Note1:** “Tool 1: Check Availability …”
- **Sticky Note5:** Explains the two code nodes’ roles and alternative suggestions behavior.

---

### Block 2 — Tool 2: Book Appointment

**Overview:**  
This block books a new appointment by validating required fields from Vapi and then creating a Google Calendar event. It returns a Vapi-formatted success or validation error response.

**Nodes involved:**
- Book Appointment Webhook
- Extract Booking Data
- Validate Required Fields
- Create Calendar Event
- Format Booking Success Response
- Respond - Booking Success
- Format Validation Error Response
- Respond - Validation Error
- Sticky Note
- Sticky Note6

#### 2.9 Book Appointment Webhook
- **Type / role:** Webhook trigger; entry point for booking.
- **Configuration:**
  - `POST /book-appointment`
  - Response mode: `responseNode`
  - Webhook ID placeholder: `YOUR_WEBHOOK_ID_BOOK_APPOINTMENT`
- **Input / output:**
  - Output → Extract Booking Data
- **Edge cases:**
  - Payload format must match expressions in Extract Booking Data (`body.message.toolCalls[0]...`).

#### 2.10 Extract Booking Data
- **Type / role:** Set node; extracts booking fields from Vapi tool call arguments.
- **Configuration choices (created fields):**
  - `name`, `email`, `phone` (from `phoneNumber`), `appointmentDate`, `appointmentTime`, `duration`, `service`, `notes`
  - `toolCallId` = `body.message.toolCalls[0].id`
- **Input / output:**
  - Input ← Book Appointment Webhook
  - Output → Validate Required Fields
- **Edge cases:**
  - If any nested paths are missing, fields become empty/undefined; validation handles some.

#### 2.11 Validate Required Fields
- **Type / role:** IF node; ensures required values exist before creating calendar event.
- **Configuration choices:**
  - Checks **not empty**: `name`, `phone`, `appointmentDate`, `appointmentTime`
- **Routing:**
  - True → Create Calendar Event
  - False → Format Validation Error Response
- **Edge cases:**
  - Does not validate format (YYYY-MM-DD, HH:MM), only non-empty.

#### 2.12 Create Calendar Event
- **Type / role:** Google Calendar node; creates the booking event.
- **Configuration choices:**
  - Operation: create (implicit by node fields)
  - Calendar: `user@example.com` (placeholder, displayed as “Bookings” cached name)
  - Start: Luxon parse `appointmentDate + 'T' + appointmentTime + ':00'` with zone `America/Vancouver`
  - End: start + `parseInt(duration) || 30` minutes
  - Summary: `${service} - ${name}`
  - Description includes customer name/phone/service/notes
- **Input / output:**
  - Input ← Validate Required Fields (true)
  - Output → Format Booking Success Response
- **Edge cases / failures:**
  - Credential/scopes issues (Google Calendar).
  - If `duration` is non-numeric, defaults to 30.
  - If date/time invalid, Luxon may produce invalid ISO.
  - No conflict check here; relies on Tool 1 being called first by assistant prompt, but nothing enforces it.

#### 2.13 Format Booking Success Response
- **Type / role:** Code node; formats response to Vapi.
- **Configuration choices:**
  - Reads:
    - booking data from `Extract Booking Data`
    - webhook data from `Book Appointment Webhook`
  - ToolCallId source:
    - `webhookData.body?.message?.toolCallList?.[0]?.id`
    - (Note: Extract Booking Data stored `toolCallId` from `toolCalls[0].id`, but this code does not use it.)
  - Returns `[{ json: { results: [{ toolCallId, result }] } }]`
- **Input / output:**
  - Input ← Create Calendar Event
  - Output → Respond - Booking Success
- **Edge cases:**
  - toolCallId may be undefined due to toolCalls/toolCallList mismatch.

#### 2.14 Respond - Booking Success
- **Type / role:** Respond to Webhook; returns final booking success response.

#### 2.15 Format Validation Error Response
- **Type / role:** Code node; formats missing-required-fields response.
- **Configuration choices:**
  - Reads toolCallId from `Book Appointment Webhook` using `toolCallList`
  - Returns `[{ json: { results: [...] } }]`
- **Input / output:**
  - Input ← Validate Required Fields (false)
  - Output → Respond - Validation Error
- **Edge cases:**
  - toolCallId path mismatch again.

#### 2.16 Respond - Validation Error
- **Type / role:** Respond to Webhook; returns validation error.

**Sticky notes in this block:**
- **Sticky Note:** “Tool 2: Book Appointment …”
- **Sticky Note6:** Mentions the formatting code nodes return Vapi response format.

---

### Block 3 — Tool 3: Manage Appointments (Cancel / Reschedule)

**Overview:**  
This block receives a management request from Vapi, routes by action (`update` vs `cancel`), finds the matching calendar event around the original date/time, then updates or deletes it.

**Nodes involved:**
- Manage Appointment Webhook
- Extract Appointment Management Data
- Route by Action Type
- Find Appointment to Update
- Update Calendar Event
- Format Reschedule Success Response
- Respond - Reschedule Success
- Find Appointment to Cancel
- Delete Calendar Event
- Format Cancellation Success Response
- Respond - Cancellation Success
- Sticky Note2
- Sticky Note7

#### 2.17 Manage Appointment Webhook
- **Type / role:** Webhook trigger; entry point for cancel/reschedule.
- **Configuration:**
  - `POST /manage-appointment`
  - Response mode: `responseNode`
  - Webhook ID placeholder: `YOUR_WEBHOOK_ID_MANAGE_APPOINTMENT`
- **Output:** → Extract Appointment Management Data

#### 2.18 Extract Appointment Management Data
- **Type / role:** Set node; normalizes tool arguments for downstream routing.
- **Configuration choices (fields):**
  - `action` = `body.message.toolCallList[0].function.arguments.action`
  - `email`
  - `originalDate` / `originalTime` from arguments `appointmentDate` / `appointmentTime`
  - `newDate` / `newTime`
  - `phoneNumber`
  - `name` = `body.message.toolCalls[0].function.arguments.name` (**note mismatch**)
- **Input / output:**
  - Input ← Manage Appointment Webhook
  - Output → Route by Action Type
- **Edge cases / critical issues:**
  - Mixed use of `toolCallList` and `toolCalls` again; `name` might not populate.
  - Email/phone/name are not actually used to locate the event; only time window is used.

#### 2.19 Route by Action Type
- **Type / role:** Switch node; splits flow by `action`.
- **Configuration:**
  - Output “Update” when `action == "update"`
  - Output “Cancel” when `action == "cancel"`
- **Input / output:**
  - Input ← Extract Appointment Management Data
  - Output “Update” → Find Appointment to Update
  - Output “Cancel” → Find Appointment to Cancel
- **Edge cases:**
  - If action is anything else, no output path triggers (silent drop unless you add a default route).

#### 2.20 Find Appointment to Update
- **Type / role:** Google Calendar `getAll`; finds the event occurring at the original date/time.
- **Configuration choices:**
  - Time window:
    - `timeMin` = original datetime
    - `timeMax` = original datetime + 1 minute
  - Calendar: `user@example.com`
  - Time zone option set: `America/Vancouver`
- **Input / output:**
  - Input ← Route by Action Type (Update)
  - Output → Update Calendar Event (iterates through matched events, possibly multiple)
- **Edge cases:**
  - If no event matches exactly in that 1-minute window, nothing to update (no explicit handling).
  - If multiple events match, will update multiple events.

#### 2.21 Update Calendar Event
- **Type / role:** Google Calendar `update`; reschedules the found event.
- **Configuration choices:**
  - Event ID: `{{ $json.id }}` (from items emitted by Find Appointment to Update)
  - New start/end:
    - uses `$('Extract Appointment Management Data').item.json.newDate/newTime`
    - end = start + 30 minutes (fixed; does not preserve original duration)
- **Input / output:**
  - Input ← Find Appointment to Update
  - Output → Format Reschedule Success Response
- **Edge cases:**
  - If `newDate/newTime` missing (user forgot to provide), Luxon parse fails.
  - No availability check before moving time.

#### 2.22 Format Reschedule Success Response
- **Type / role:** Code node; produces Vapi response.
- **Configuration:**
  - `toolCallId` = `$('Manage Appointment Webhook').first().json.body.message.toolCallList[0].id`
  - Returns a plain object `{ results: [...] }` (not wrapped in `[{json: ...}]`)
- **Input / output:**
  - Input ← Update Calendar Event
  - Output → Respond - Reschedule Success
- **Edge cases:**
  - Return shape inconsistency (same risk as the Unavailable formatter).

#### 2.23 Respond - Reschedule Success
- **Type / role:** Respond to Webhook; returns success response.

#### 2.24 Find Appointment to Cancel
- **Type / role:** Google Calendar `getAll`; finds matching event to delete.
- **Configuration:**
  - Time window: original datetime to +1 minute
- **Input / output:**
  - Input ← Route by Action Type (Cancel)
  - Output → Delete Calendar Event

#### 2.25 Delete Calendar Event
- **Type / role:** Google Calendar `delete`; deletes the found event.
- **Configuration:**
  - Event ID: `{{ $json.id }}`
- **Input / output:**
  - Input ← Find Appointment to Cancel
  - Output → Format Cancellation Success Response
- **Edge cases:**
  - If multiple events match, it deletes multiple.
  - If none match, nothing happens (no explicit “not found” response).

#### 2.26 Format Cancellation Success Response
- **Type / role:** Code node; formats cancellation response for Vapi.
- **Configuration:**
  - Reads toolCallId from Manage Appointment Webhook `toolCallList[0].id`
  - Returns plain object `{ results: [...] }` (not wrapped)
- **Input / output:**
  - Input ← Delete Calendar Event
  - Output → Respond - Cancellation Success
- **Edge cases:**
  - Return shape inconsistency risk.

#### 2.27 Respond - Cancellation Success
- **Type / role:** Respond to Webhook; returns success response.

**Sticky notes in this block:**
- **Sticky Note2:** “Tool 3: Manage Appointments …”
- **Sticky Note7:** Mentions code nodes format results for easy debugging in Vapi terminal.

---

### Block 4 — Documentation / Prompt Notes (Non-executing)
**Overview:**  
These sticky notes provide instructions for configuring Vapi and using the tools.

**Nodes involved:**
- Sticky Note3
- Sticky Note4

#### 2.28 Sticky Note3 (Vapi voice assistant prompt)
- **Type / role:** Sticky note; contains a full prompt/guidelines for the voice assistant.
- **Key content highlights:**
  - Use `checkAvailability` before booking
  - Required booking fields
  - Manage appointment process with `manageAppointment` action `cancel` / `update`
  - Confirmation rules and response guidelines

#### 2.29 Sticky Note4 (Workflow description & setup steps)
- **Type / role:** Sticky note; general overview and prerequisites.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Book Appointment Webhook | Webhook | Entry point for booking tool | — | Extract Booking Data | ## Tool 2: Book Appointment / Receives booking data from Vapi, validates required fields, creates Google Calendar event, and returns formatted response. |
| Extract Booking Data | Set | Extract booking arguments from Vapi payload | Book Appointment Webhook | Validate Required Fields | ## Tool 2: Book Appointment / Receives booking data from Vapi, validates required fields, creates Google Calendar event, and returns formatted response. |
| Validate Required Fields | If | Ensure required booking fields are present | Extract Booking Data | Create Calendar Event; Format Validation Error Response | ## Tool 2: Book Appointment / Receives booking data from Vapi, validates required fields, creates Google Calendar event, and returns formatted response. |
| Create Calendar Event | Google Calendar | Create appointment event in calendar | Validate Required Fields | Format Booking Success Response | The top and bottom code node format the results in the required vapi response format |
| Format Booking Success Response | Code | Build Vapi tool response (booking success) | Create Calendar Event | Respond - Booking Success | The top and bottom code node format the results in the required vapi response format |
| Respond - Booking Success | Respond to Webhook | Return HTTP response to Vapi | Format Booking Success Response | — | The top and bottom code node format the results in the required vapi response format |
| Format Validation Error Response | Code | Build Vapi tool response (validation error) | Validate Required Fields | Respond - Validation Error | The top and bottom code node format the results in the required vapi response format |
| Respond - Validation Error | Respond to Webhook | Return HTTP response to Vapi | Format Validation Error Response | — | The top and bottom code node format the results in the required vapi response format |
| Check Availability Webhook | Webhook | Entry point for availability tool | — | Extract Availability Request | ## Tool 1: Check Availability / Queries Google Calendar for requested date, checks if time slot is available, and suggests alternatives if booked. |
| Extract Availability Request | Set | Extract date/time and compute requested datetime | Check Availability Webhook | Get Calendar Events for Date | ## Tool 1: Check Availability / Queries Google Calendar for requested date, checks if time slot is available, and suggests alternatives if booked. |
| Get Calendar Events for Date | Google Calendar | Fetch all events for the requested date | Extract Availability Request | Check Time Slot Availability | ## Tool 1: Check Availability / Queries Google Calendar for requested date, checks if time slot is available, and suggests alternatives if booked. |
| Check Time Slot Availability | If | Decide available vs unavailable | Get Calendar Events for Date | Format Unavailable Time Response; Format Available Time Response | ## Tool 1: Check Availability / Queries Google Calendar for requested date, checks if time slot is available, and suggests alternatives if booked. |
| Format Unavailable Time Response | Code | Suggest 3 alternate slots and return Vapi response | Check Time Slot Availability | Respond - Time Unavailable | The top code node pulls all events, checks that time and data requested does not overlap with any events, returns time periods as unavailable and then suggests 3 alternate times closest to the requested time and returns a result in the required vapi response format. The bottom code node returns the result as available in the required vapi response format. |
| Respond - Time Unavailable | Respond to Webhook | Return HTTP response (unavailable) | Format Unavailable Time Response | — | The top code node pulls all events, checks that time and data requested does not overlap with any events, returns time periods as unavailable and then suggests 3 alternate times closest to the requested time and returns a result in the required vapi response format. The bottom code node returns the result as available in the required vapi response format. |
| Format Available Time Response | Code | Return Vapi response (available) | Check Time Slot Availability | Respond - Time Available | The top code node pulls all events, checks that time and data requested does not overlap with any events, returns time periods as unavailable and then suggests 3 alternate times closest to the requested time and returns a result in the required vapi response format. The bottom code node returns the result as available in the required vapi response format. |
| Respond - Time Available | Respond to Webhook | Return HTTP response (available) | Format Available Time Response | — | The top code node pulls all events, checks that time and data requested does not overlap with any events, returns time periods as unavailable and then suggests 3 alternate times closest to the requested time and returns a result in the required vapi response format. The bottom code node returns the result as available in the required vapi response format. |
| Manage Appointment Webhook | Webhook | Entry point for cancel/reschedule tool | — | Extract Appointment Management Data | ## Tool 3: Manage Appointments / Handles cancellations and rescheduling. Routes by action type, finds existing appointment, then updates or deletes calendar event. |
| Extract Appointment Management Data | Set | Extract action and appointment details | Manage Appointment Webhook | Route by Action Type | ## Tool 3: Manage Appointments / Handles cancellations and rescheduling. Routes by action type, finds existing appointment, then updates or deletes calendar event. |
| Route by Action Type | Switch | Route to cancel or update branch | Extract Appointment Management Data | Find Appointment to Update; Find Appointment to Cancel | ## Tool 3: Manage Appointments / Handles cancellations and rescheduling. Routes by action type, finds existing appointment, then updates or deletes calendar event. |
| Find Appointment to Update | Google Calendar | Locate event by original time window | Route by Action Type (Update) | Update Calendar Event | ## Tool 3: Manage Appointments / Handles cancellations and rescheduling. Routes by action type, finds existing appointment, then updates or deletes calendar event. |
| Update Calendar Event | Google Calendar | Update event time (reschedule) | Find Appointment to Update | Format Reschedule Success Response | ## Tool 3: Manage Appointments / Handles cancellations and rescheduling. Routes by action type, finds existing appointment, then updates or deletes calendar event. |
| Format Reschedule Success Response | Code | Build Vapi response (reschedule success) | Update Calendar Event | Respond - Reschedule Success | The code nodes format results in the required vapi response format for easy debugging on vapi terminal |
| Respond - Reschedule Success | Respond to Webhook | Return HTTP response (reschedule) | Format Reschedule Success Response | — | The code nodes format results in the required vapi response format for easy debugging on vapi terminal |
| Find Appointment to Cancel | Google Calendar | Locate event by original time window | Route by Action Type (Cancel) | Delete Calendar Event | ## Tool 3: Manage Appointments / Handles cancellations and rescheduling. Routes by action type, finds existing appointment, then updates or deletes calendar event. |
| Delete Calendar Event | Google Calendar | Delete the found event | Find Appointment to Cancel | Format Cancellation Success Response | ## Tool 3: Manage Appointments / Handles cancellations and rescheduling. Routes by action type, finds existing appointment, then updates or deletes calendar event. |
| Format Cancellation Success Response | Code | Build Vapi response (cancellation success) | Delete Calendar Event | Respond - Cancellation Success | The code nodes format results in the required vapi response format for easy debugging on vapi terminal |
| Respond - Cancellation Success | Respond to Webhook | Return HTTP response (cancellation) | Format Cancellation Success Response | — | The code nodes format results in the required vapi response format for easy debugging on vapi terminal |
| Sticky Note | Sticky Note | Documentation (Tool 2 description) | — | — | ## Tool 2: Book Appointment / Receives booking data from Vapi, validates required fields, creates Google Calendar event, and returns formatted response. |
| Sticky Note1 | Sticky Note | Documentation (Tool 1 description) | — | — | ## Tool 1: Check Availability / Queries Google Calendar for requested date, checks if time slot is available, and suggests alternatives if booked. |
| Sticky Note2 | Sticky Note | Documentation (Tool 3 description) | — | — | ## Tool 3: Manage Appointments / Handles cancellations and rescheduling. Routes by action type, finds existing appointment, then updates or deletes calendar event. |
| Sticky Note3 | Sticky Note | Vapi assistant prompt text | — | — | (Contains the full “VAPI VOICE ASSISTANT PROMPT” content) |
| Sticky Note4 | Sticky Note | Overall workflow explanation and setup prerequisites | — | — | (Contains “Book Appointments using Vapi and Google Calendar” overview and setup steps) |
| Sticky Note5 | Sticky Note | Notes about availability formatting logic | — | — | The top code node pulls all events, checks that time and data requested does not overlap with any events, returns time periods as unavailable and then suggests 3 alternate times closest to the requested time and returns a result in the required vapi response format. The bottom code node returns the result as available in the required vapi response format. |
| Sticky Note6 | Sticky Note | Notes about booking response formatting | — | — | The top and bottom code node format the results in the required vapi response format |
| Sticky Note7 | Sticky Note | Notes about management response formatting | — | — | The code nodes format results in the required vapi response format for easy debugging on vapi terminal |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
1. **n8n** instance (cloud or self-hosted).
2. **Google Calendar** account with a calendar to store bookings.
3. **Vapi.ai** assistant that can call custom tools via HTTP.

### Credentials
1. In n8n, create **Google Calendar OAuth2** credentials:
   - Ensure scopes include Calendar read/write (commonly `.../auth/calendar`).
   - Connect the Google account that owns (or has access to) the target calendar.
2. Attach these credentials to **all Google Calendar nodes** in the workflow.

---

### A) Tool 1 — Check Availability endpoint

1. **Create Webhook node** named **“Check Availability Webhook”**
   - Type: *Webhook*
   - Method: **POST**
   - Path: **availability**
   - Response mode: **Using “Respond to Webhook” node**

2. **Create Set node** named **“Extract Availability Request”**
   - Add string fields:
     - `appointmentDate` = from webhook body tool call arguments (Vapi)
     - `appointmentTime` = same
     - `requestedDatetime` = Luxon expression combining date + time in `America/Vancouver`
   - Connect: Webhook → Set

3. **Create Google Calendar node** named **“Get Calendar Events for Date”**
   - Operation: **Get All**
   - Calendar: select your bookings calendar (replace `user@example.com`)
   - Timezone option: `America/Vancouver`
   - `timeMin`: `${appointmentDate}T00:00:00-08:00`
   - `timeMax`: `${appointmentDate}T23:59:59-08:00`
   - Enable **Always Output Data**
   - Connect: Set → Google Calendar

4. **Create IF node** named **“Check Time Slot Availability”**
   - Condition should compare the event time to the requested time.
   - Important: fix the broken reference:
     - Replace `$items('Edit Fields1')...` with `$items('Extract Availability Request')...` (or use `$node[...]`)
   - Connect: Google Calendar → IF

5. **Create Code node** named **“Format Available Time Response”**
   - Output format: return an array with `{json:{results:[...]}}`
   - Include:
     - `toolCallId` from webhook body
     - `result` string confirming availability
   - Connect: IF (available path) → Code

6. **Create Respond to Webhook** node named **“Respond - Time Available”**
   - Connect: Format Available Time Response → Respond

7. **Create Code node** named **“Format Unavailable Time Response”**
   - Implement:
     - Pull day events
     - Generate business-hour slots
     - Filter out overlaps
     - Suggest first 3
   - Ensure the Code node returns **`[{ json: { results: [...] } }]`** for consistency.
   - Connect: IF (unavailable path) → Code

8. **Create Respond to Webhook** node named **“Respond - Time Unavailable”**
   - Connect: Format Unavailable Time Response → Respond

---

### B) Tool 2 — Book Appointment endpoint

9. **Create Webhook node** named **“Book Appointment Webhook”**
   - POST path: **book-appointment**
   - Response mode: **Respond to Webhook node**

10. **Create Set node** named **“Extract Booking Data”**
   - Map fields from Vapi tool call arguments:
     - `name`, `email`, `phoneNumber` → store as `phone`
     - `appointmentDate`, `appointmentTime`, `duration`, `service`, `notes`
     - capture `toolCallId`
   - Connect: Webhook → Set

11. **Create IF node** named **“Validate Required Fields”**
   - Require non-empty:
     - `name`, `phone`, `appointmentDate`, `appointmentTime`
   - Connect: Set → IF

12. **Create Google Calendar node** named **“Create Calendar Event”**
   - Operation: **Create**
   - Calendar: your bookings calendar
   - Start: Luxon parse date/time in `America/Vancouver`
   - End: start + `duration` minutes (default 30)
   - Summary: `${service} - ${name}`
   - Description: include customer details
   - Connect: IF (true) → Google Calendar

13. **Create Code node** named **“Format Booking Success Response”**
   - Return Vapi `results[]` with `toolCallId` and success string
   - Connect: Google Calendar → Code

14. **Create Respond to Webhook** named **“Respond - Booking Success”**
   - Connect: Code → Respond

15. **Create Code node** named **“Format Validation Error Response”**
   - Return Vapi `results[]` with message “missing required fields”
   - Connect: IF (false) → Code

16. **Create Respond to Webhook** named **“Respond - Validation Error”**
   - Connect: Code → Respond

---

### C) Tool 3 — Manage Appointment endpoint (Cancel/Reschedule)

17. **Create Webhook node** named **“Manage Appointment Webhook”**
   - POST path: **manage-appointment**
   - Response mode: Respond to Webhook node

18. **Create Set node** named **“Extract Appointment Management Data”**
   - Extract:
     - `action` (“cancel” or “update”)
     - `originalDate`/`originalTime` (from appointmentDate/appointmentTime)
     - `newDate`/`newTime` (for update)
     - optional: `name`, `email`, `phoneNumber`
   - Connect: Webhook → Set

19. **Create Switch node** named **“Route by Action Type”**
   - Rule 1: if `action == "update"` → output “Update”
   - Rule 2: if `action == "cancel"` → output “Cancel”
   - Connect: Set → Switch

**Update path**
20. **Google Calendar node** “Find Appointment to Update”
   - Operation: Get All
   - TimeMin = original datetime
   - TimeMax = original datetime + 1 minute
   - Connect: Switch(Update) → Find

21. **Google Calendar node** “Update Calendar Event”
   - Operation: Update
   - EventId = `{{$json.id}}`
   - New start/end computed from `newDate/newTime` (consider preserving duration if needed)
   - Connect: Find → Update

22. **Code node** “Format Reschedule Success Response”
   - Return `[{ json: { results: [...] } }]`
   - Connect: Update → Code

23. **Respond to Webhook** “Respond - Reschedule Success”
   - Connect: Code → Respond

**Cancel path**
24. **Google Calendar node** “Find Appointment to Cancel”
   - Operation: Get All
   - Same time window logic
   - Connect: Switch(Cancel) → Find

25. **Google Calendar node** “Delete Calendar Event”
   - Operation: Delete
   - EventId = `{{$json.id}}`
   - Connect: Find → Delete

26. **Code node** “Format Cancellation Success Response”
   - Return `[{ json: { results: [...] } }]`
   - Connect: Delete → Code

27. **Respond to Webhook** “Respond - Cancellation Success”
   - Connect: Code → Respond

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Book Appointments using Vapi and Google Calendar… Key Features… Set up steps…” | Sticky Note4 (overall workflow description and prerequisites) |
| “VAPI VOICE ASSISTANT PROMPT… Always use checkAvailability first… Never book UNAVAILABLE…” | Sticky Note3 (assistant behavior + tool usage rules) |
| “The top code node pulls all events… suggests 3 alternate times…” | Sticky Note5 (availability formatting logic explanation) |
| “The top and bottom code node format the results in the required vapi response format” | Sticky Note6 (booking block formatting note) |
| “The code nodes format results in the required vapi response format for easy debugging on vapi terminal” | Sticky Note7 (manage block formatting note) |

--- 

### Important implementation warnings (based on the provided workflow)
- **Broken node reference:** `Check Time Slot Availability` references `Edit Fields1` (non-existent). Must be updated to `Extract Availability Request`.
- **Inconsistent Vapi payload paths:** mixture of `toolCalls` and `toolCallList` is used across nodes. Standardize to the exact structure Vapi sends in your environment.
- **Code node return shapes inconsistent:** some Code nodes return plain objects instead of `[{ json: ... }]`. Standardize to the array-of-items format to avoid runtime issues.
- **Timezone/DST risk:** hard-coded `-08:00` may be wrong during DST. Prefer using timezone-based boundaries computed with Luxon rather than fixed offsets.
- **Event overlap logic:** the IF availability check uses equality on start time; real availability requires **overlap detection** (which the “unavailable” code attempts, but the routing logic should be aligned with it).