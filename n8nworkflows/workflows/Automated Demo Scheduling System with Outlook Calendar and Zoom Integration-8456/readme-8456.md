Automated Demo Scheduling System with Outlook Calendar and Zoom Integration

https://n8nworkflows.xyz/workflows/automated-demo-scheduling-system-with-outlook-calendar-and-zoom-integration-8456


# Automated Demo Scheduling System with Outlook Calendar and Zoom Integration

### 1. Workflow Overview

This workflow automates the scheduling of live product demos by integrating Microsoft Outlook Calendar and Zoom. It is designed for companies that want to streamline demo requests, automatically manage available time slots, and finalize bookings with calendar events and Zoom meetings.

**Target Use Cases:**  
- Businesses offering product demos on-demand.  
- Automating client scheduling without manual calendar checks.  
- Ensuring no double-booking via Outlook event updates.  
- Providing clients with a seamless scheduling experience including confirmation and meeting links.

**Logical Blocks:**

- **1.1 Input Reception**  
  Collect client and demo request details via forms.

- **1.2 Date Availability Check (Outlook Calendar)**  
  Verify if requested demo date has available pre-created "Online Meeting Slot" events.

- **1.3 Time Slot Selection and Validation**  
  Present available time slots to the client for selection or request a new date if none available.

- **1.4 Booking Finalization (Zoom & Outlook Update)**  
  Create Zoom meeting for the booked slot, update the Outlook event with client info and Zoom link.

- **1.5 Confirmation and Error Handling**  
  Show confirmation form to client or an error form if any step fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Collects client company and contact details, along with a preferred demo date, via a web form.

- **Nodes Involved:**  
  - `On form submission` (Form Trigger)  
  - `Client Details` (Form)

- **Node Details:**

  - **On form submission:**  
    - Type: Form Trigger  
    - Role: Entry point webhook that listens for new demo requests at path `/yourcompany-demo-request`.  
    - Config: Custom CSS styling for UI, form title "Live Demo Request", description included.  
    - Inputs: HTTP request from client form.  
    - Outputs: Passes form data to `Client Details`.  
    - Failure cases: Network issues, malformed form data.

  - **Client Details:**  
    - Type: Form node  
    - Role: Collects detailed info: company name, contact name, role, email, phone, and demo date.  
    - Config: Fields are all required; custom CSS for UI styling consistent with branding.  
    - Inputs: From `On form submission`.  
    - Outputs: Passes collected data to `Selected Date Has Slot Available?` for validation.  
    - Failure cases: Missing required fields, invalid email format.

---

#### 1.2 Date Availability Check (Outlook Calendar)

- **Overview:**  
  Validates whether the selected demo date has available pre-created Outlook events titled "Online Meeting Slot". If available, fetches up to 3 nearest slots.

- **Nodes Involved:**  
  - `Selected Date Has Slot Available?` (If)  
  - `Outlook Calendar CheckAvailability` (HTTP Request)  
  - `New Slot Available?` (If)  
  - `Set Events` (Code)

- **Node Details:**

  - **Selected Date Has Slot Available?:**  
    - Type: If node  
    - Role: Checks if the selected date is today or later (date not in past).  
    - Condition: Boolean check that selected date timestamp >= current timestamp.  
    - Inputs: `Client Details` form data.  
    - Outputs: True → `Outlook Calendar CheckAvailability`, False → `Select New Date` (ask for a new date).  
    - Failure cases: Date parsing errors, client selecting past dates.

  - **Outlook Calendar CheckAvailability:**  
    - Type: HTTP Request  
    - Role: Calls Microsoft Graph API to fetch events on the selected date with subject starting with "Online Meeting Slot".  
    - Config: Queries using the selected date as startDateTime, with endDateTime set 30 days later, limits to 3 results, orders ascending. Uses timezone Asia/Dubai.  
    - Authentication: Microsoft Outlook OAuth2 credential required.  
    - Inputs: Date from previous node.  
    - Outputs: List of available events or empty list.  
    - Failure cases: Auth errors, Microsoft Graph API downtime, rate limits.

  - **New Slot Available?:**  
    - Type: If node  
    - Role: Checks if the HTTP response contains non-empty event values.  
    - Outputs: True → `Set Events`, False → `Select New Date`.  
    - Failure cases: Unexpected API response formats.

  - **Set Events:**  
    - Type: Code node (JavaScript)  
    - Role: Parses Outlook events, formats date/time for display and prepares objects for form dropdown.  
    - Key logic: Formats date string like "Wednesday 20 August 2025 11:00".  
    - Inputs: Events array from Outlook API.  
    - Outputs: Each available slot as individual item with time, subject, start/end, id.  
    - Failure cases: Date parsing issues, empty events array.

---

#### 1.3 Time Slot Selection and Validation

- **Overview:**  
  Presents the client with a dropdown of up to 3 available time slots to pick from. Merges selected slot with client data for next steps.

- **Nodes Involved:**  
  - `Set Form Dates` (Code)  
  - `Select Date&Time` (Form)  
  - `Get Date` (Merge)  
  - `Persist Selected Date&Time` (Merge)  
  - `Selected Date Has Slot Available?` (If for validation on new date, if needed)  
  - `Select New Date` (Form, fallback for unavailable dates)

- **Node Details:**

  - **Set Form Dates:**  
    - Type: Code node  
    - Role: Builds a dropdown form field from available slots for user to pick.  
    - Inputs: Array of event slots from `Set Events`.  
    - Outputs: First item is dropdown form field, subsequent items are event objects.  
    - Failure cases: Empty input array.

  - **Select Date&Time:**  
    - Type: Form node  
    - Role: Displays dropdown of available time slots with submit button for user selection.  
    - Config: Custom CSS for branding, title "Select your preferred time".  
    - Inputs: Form fields from `Set Form Dates`.  
    - Outputs: Selected slot to `Get Date`.  
    - Failure cases: User submits no selection.

  - **Get Date:**  
    - Type: Merge node (combine mode)  
    - Role: Combines selected time slot with event details from `Set Form Dates`.  
    - Inputs: From `Select Date&Time` and from `Set Form Dates`.  
    - Outputs: Merged JSON to `Persist Selected Date&Time`.  
    - Failure cases: Incorrect merge due to mismatched keys.

  - **Persist Selected Date&Time:**  
    - Type: Merge node (combine mode)  
    - Role: Merges client details with selected event info for final booking steps.  
    - Inputs: From `Get Date` and client details.  
    - Outputs: Combined data to `Create Zoom meeting`.  
    - Failure cases: Data mismatch, incomplete info.

  - **Selected Date Has Slot Available?:** (Used again for validation when user picks a new date)  
    - Role: Validates if newly selected date has available slots or asks user to reselect.

  - **Select New Date:**  
    - Type: Form node  
    - Role: Requests user to select a new date if no slots available or invalid date is chosen.  
    - Config: Title "Previous selected date is unavailable".  
    - Inputs: From previous validation nodes.  
    - Outputs: Loops back to availability check.  
    - Failure cases: User repeatedly submits invalid dates.

---

#### 1.4 Booking Finalization (Zoom & Outlook Update)

- **Overview:**  
  Creates a Zoom meeting for the selected slot, updates the Outlook event with meeting details and client info, marking it as booked.

- **Nodes Involved:**  
  - `Create Zoom meeting` (Zoom node)  
  - `Set Event Fields` (Set node)  
  - `Outlook Update Event` (HTTP Request)  
  - `Final Form` (Form)

- **Node Details:**

  - **Create Zoom meeting:**  
    - Type: Zoom node  
    - Role: Creates a Zoom meeting with topic "Live Demo - {Company trade name}", duration 40 minutes, timezone Asia/Dubai, starting at selected event start time.  
    - Auth: OAuth2 Zoom credentials required.  
    - Inputs: Merged client and slot data.  
    - Outputs: Zoom meeting join_url and details.  
    - Failure cases: Zoom API errors, auth failure, rate limits.

  - **Set Event Fields:**  
    - Type: Set node  
    - Role: Prepares the event update body with:  
      - HTML content including client details and Zoom link  
      - Static strings for email and name (placeholders "John" and "Smith") — this may be a bug or placeholder  
    - Inputs: Zoom meeting details and merged client info.  
    - Outputs: JSON for Outlook event update.  
    - Failure cases: Missing Zoom data, incomplete client info.

  - **Outlook Update Event:**  
    - Type: HTTP Request  
    - Role: PATCH request to Microsoft Graph API to update the selected Outlook event:  
      - Changes subject to "Live Demo Booked"  
      - Marks event as busy  
      - Adds attendees: client and internal contact  
      - Adds Zoom meeting link in body content  
    - Auth: Microsoft Outlook OAuth2 required.  
    - Inputs: From `Set Event Fields` and `Persist Selected Date&Time` (for event ID and attendees).  
    - Outputs: Success confirmation or error.  
    - Failure cases: Auth failure, event not found, invalid patch data.

  - **Final Form:**  
    - Type: Form node (completion)  
    - Role: Shows confirmation message with booked date/time formatted nicely, confirming booking success.  
    - Inputs: From successful `Outlook Update Event`.  
    - Outputs: Final client confirmation page.  
    - Failure cases: None expected if upstream succeeds.

---

#### 1.5 Confirmation and Error Handling

- **Overview:**  
  Provides user feedback on success or failure of the scheduling process.

- **Nodes Involved:**  
  - `Form Failed` (Form node)  
  - Error branches from multiple nodes

- **Node Details:**

  - **Form Failed:**  
    - Type: Form node (completion)  
    - Role: Displays an error message if any critical step fails (e.g., no slots, API failure).  
    - Content: Asks user to try again later or contact support with provided phone/email.  
    - Trigger: On error branches from key nodes such as `Outlook Calendar CheckAvailability`, `Create Zoom meeting`, `Outlook Update Event`.  
    - Failure cases: N/A (final fallback).

---

### 3. Summary Table

| Node Name                    | Node Type             | Functional Role                                | Input Node(s)                       | Output Node(s)                   | Sticky Note                                                                                     |
|------------------------------|-----------------------|----------------------------------------------|-----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note16                | Sticky Note           | Title banner for the workflow                 |                                   |                                 | # 🚀 Live Demo Request                                                                         |
| On form submission           | Form Trigger          | Entry point for demo request form             |                                   | Client Details                  | Custom styled demo request form with branding                                                 |
| Client Details               | Form                  | Collects client and requested demo date info | On form submission                | Selected Date Has Slot Available? | Custom styled form with required fields                                                       |
| Selected Date Has Slot Available? | If                   | Validates if selected date is today or future | Client Details                   | Outlook Calendar CheckAvailability / Select New Date | See Sticky Note about Date Availability                                                       |
| Outlook Calendar CheckAvailability | HTTP Request          | Checks Outlook calendar for available slots   | Selected Date Has Slot Available?  | New Slot Available?             | Authenticated Microsoft Graph API call, queries events with subject "Online Meeting Slot"     |
| New Slot Available?          | If                    | Checks if Outlook returned any slot events    | Outlook Calendar CheckAvailability | Set Events / Select New Date    |                                                                                                |
| Set Events                  | Code                  | Formats available event slots for dropdown    | New Slot Available?               | Set Form Dates                 |                                                                                                |
| Set Form Dates              | Code                  | Creates dropdown form fields for slot selection | Set Events                    | Select Date&Time               | See Sticky Note about Time Availability                                                       |
| Select Date&Time            | Form                  | Shows available time slots for client selection | Set Form Dates                 | Get Date                      | Custom styled form with dropdown                                                             |
| Get Date                   | Merge                 | Combines selected time with event details     | Select Date&Time, Set Form Dates  | Persist Selected Date&Time     |                                                                                                |
| Persist Selected Date&Time  | Merge                 | Merges client info and selected slot for booking | Get Date, Client Details        | Create Zoom meeting            |                                                                                                |
| Create Zoom meeting         | Zoom                  | Creates Zoom meeting for booked slot           | Persist Selected Date&Time       | Set Event Fields / Form Failed | See Sticky Note about Zoom & Outlook integration                                              |
| Set Event Fields            | Set                   | Prepares Outlook event update content          | Create Zoom meeting              | Outlook Update Event           |                                                                                                |
| Outlook Update Event        | HTTP Request          | Updates Outlook event with booking details     | Set Event Fields                | Final Form / Form Failed       |                                                                                                |
| Final Form                 | Form                  | Confirmation page after successful booking     | Outlook Update Event            |                                 |                                                                                                |
| Select New Date            | Form                  | Requests new date if no slots available or invalid date | New Slot Available?, Selected Date Has Slot Available? | Selected Date Has Slot Available? | Custom styled form prompting new date                                                        |
| Form Failed               | Form                  | Error message page on failure                   | Multiple error branches         |                                 | Shows contact info for support                                                                |
| Sticky Note                | Sticky Note           | Date Availability explanation                   |                               |                               | See detailed note on Date Availability                                                        |
| Sticky Note1               | Sticky Note           | Time Availability explanation                   |                               |                               | See detailed note on Time Availability                                                        |
| Sticky Note2               | Sticky Note           | Zoom meeting creation and Outlook event update |                               |                               | See detailed note on Zoom and Outlook integration                                            |
| Sticky Note3               | Sticky Note           | Overall workflow explanation and requirements  |                               |                               | High-level overview of the workflow steps and requirements                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `On form submission`  
   - Path: `yourcompany-demo-request`  
   - Add form title "Live Demo Request" and description "Request a live demo of our software"  
   - Style with custom CSS as per branding (optional)  
   - This node is the workflow entry point.

2. **Create a Form Node for Client Details**  
   - Name: `Client Details`  
   - Fields (all required):  
     - Company trade name (text)  
     - Full Contact Name (text)  
     - Contact Role (text)  
     - Contact Email (email)  
     - Contact Phone (text)  
     - Select the date you would require a demo (date)  
   - Connect `On form submission` output to `Client Details` input.

3. **Add an If Node to Validate Selected Date**  
   - Name: `Selected Date Has Slot Available?`  
   - Condition: Check if selected date ≥ today (no past dates)  
   - True output → `Outlook Calendar CheckAvailability`  
   - False output → `Select New Date` (form to ask for a new date)

4. **Add HTTP Request Node for Outlook Calendar Check**  
   - Name: `Outlook Calendar CheckAvailability`  
   - Method: GET  
   - URL: `https://graph.microsoft.com/v1.0/me/calendarView`  
   - Query Parameters:  
     - startDateTime = selected demo date + "T00:00:00"  
     - endDateTime = selected date + 30 days + "T23:59:59"  
     - $top = 3  
     - $orderby = start/dateTime asc  
     - $filter = startsWith(subject,'Online Meeting Slot')  
   - Header: Prefer = `outlook.timezone="Asia/Dubai"`  
   - Authentication: Microsoft Outlook OAuth2  
   - Connect `Selected Date Has Slot Available?` true output here.

5. **Add If Node to Check if Slots Are Available**  
   - Name: `New Slot Available?`  
   - Condition: Response JSON contains non-empty `value` array  
   - True → `Set Events`  
   - False → `Select New Date` (ask user for new date)

6. **Add Code Node to Format Available Events**  
   - Name: `Set Events`  
   - Input: HTTP response from Outlook  
   - JS logic: Map events, parse start time, format to readable string (e.g., "Wednesday 20 August 2025 11:00"), output each slot as separate item.  
   - Connect true output of `New Slot Available?` here.

7. **Add Code Node to Build Dropdown Form**  
   - Name: `Set Form Dates`  
   - Input: Items from `Set Events`  
   - Build a dropdown field with label "Select one of our available time" and options from event times.  
   - Output: First item is dropdown field, followed by event objects.  
   - Connect `Set Events` output here.

8. **Add Form Node for Time Selection**  
   - Name: `Select Date&Time`  
   - Display dropdown from previous node, button label "Submit", title "Select your preferred time".  
   - Connect `Set Form Dates` output here.

9. **Add Merge Node to Combine Selected Time with Event Details**  
   - Name: `Get Date`  
   - Mode: Combine by position  
   - Inputs: user selection from `Select Date&Time` and event details from `Set Form Dates`  
   - Connect `Select Date&Time` and `Set Form Dates` outputs here.

10. **Add Merge Node to Combine Client Details and Selected Slot**  
    - Name: `Persist Selected Date&Time`  
    - Mode: Combine by position  
    - Inputs: `Get Date` and `Client Details`  
    - Connect both inputs accordingly.

11. **Add Zoom Node to Create Meeting**  
    - Name: `Create Zoom meeting`  
    - Authentication: OAuth2 Zoom credentials setup required  
    - Parameters:  
      - Topic: `"Live Demo - {{ $json['Company trade name'] }}"`  
      - Duration: 40 minutes  
      - Timezone: Asia/Dubai  
      - Start time: event start time from merged data  
    - Connect `Persist Selected Date&Time` output here.

12. **Add Set Node to Prepare Outlook Event Update**  
    - Name: `Set Event Fields`  
    - Assign fields:  
      - `content` (HTML with client details and Zoom join URL)  
      - `your_email` and `your_name` placeholders (should be replaced with real data or removed)  
    - Connect Zoom meeting output here.

13. **Add HTTP Request Node to Update Outlook Event**  
    - Name: `Outlook Update Event`  
    - Method: PATCH  
    - URL: `https://graph.microsoft.com/v1.0/me/events/{{ eventId }}` (eventId from selected slot)  
    - Body (JSON):  
      - categories: ["Booked", "Online meeting"]  
      - subject: "Live Demo Booked"  
      - showAs: "busy"  
      - body: HTML content from `Set Event Fields`  
      - attendees: client and internal contact with email and name  
    - Authentication: Microsoft Outlook OAuth2  
    - Connect `Set Event Fields` here.

14. **Add Form Node for Final Confirmation**  
    - Name: `Final Form`  
    - Operation: Completion  
    - Title: "Live Demo Scheduled"  
    - Message: Confirmation with formatted date/time from selected event  
    - Connect `Outlook Update Event` success output here.

15. **Add Form Node for Failures**  
    - Name: `Form Failed`  
    - Operation: Completion  
    - Title: "Submission Failed"  
    - Message: Inform user of failure and provide contact info  
    - Connect error outputs from:  
      - `Outlook Calendar CheckAvailability`  
      - `New Slot Available?` false branch  
      - `Create Zoom meeting` error  
      - `Outlook Update Event` error  

16. **Add Form Node for New Date Selection**  
    - Name: `Select New Date`  
    - Field: Date picker labeled "Select the date you would you require a demo"  
    - Title: "Previous selected date is unavailable"  
    - Connect false branches or fallback paths from date availability checks.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The Outlook Calendar requires pre-created events with the subject "Online Meeting Slot" for available demo times. This workflow does not create these slots; they must be managed manually or by a separate system.                 | Sticky Note near `Selected Date Has Slot Available?` and `Outlook Calendar CheckAvailability`     |
| OAuth2 credentials for Microsoft Outlook and Zoom must be configured in n8n credentials prior to running this workflow.                                                                                                         | Workflow requirement                                                                             |
| Custom CSS is applied to all forms to ensure branding consistency and responsive design for mobile devices.                                                                                                                    | Present in all form nodes                                                                         |
| The Zoom meeting duration is fixed at 40 minutes and uses Asia/Dubai timezone. Adjust these parameters if your organization operates in a different timezone.                                                                   | `Create Zoom meeting` node parameters                                                           |
| The placeholder email and name in `Set Event Fields` ("John" and "Smith") should be replaced with dynamic client or internal user data to avoid confusion in calendar invites.                                                     | Potential bug or placeholder in `Set Event Fields` node                                          |
| Documentation on Microsoft Graph API for calendar events: https://learn.microsoft.com/en-us/graph/api/resources/calendar?view=graph-rest-1.0                                                                                   | Reference for Outlook API usage                                                                  |
| Zoom API documentation for creating meetings: https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/meetingCreate                                                                                          | Reference for Zoom node configuration                                                           |
| This workflow is designed for n8n versions supporting webhook form triggers and OAuth2 integrations (tested on n8n 0.195+).                                                                                                    | Version recommendation                                                                           |

---

*Disclaimer: The text provided is exclusively extracted from an automated n8n workflow. All content complies with legal and content policies, with no illegal or offensive data included.*