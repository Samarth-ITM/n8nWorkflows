Track Calendly Schedule Changes with Google Sheets & Slack Notifications

https://n8nworkflows.xyz/workflows/track-calendly-schedule-changes-with-google-sheets---slack-notifications-11247


# Track Calendly Schedule Changes with Google Sheets & Slack Notifications

### 1. Workflow Overview

This n8n workflow automates tracking and notifications for Calendly scheduling events, specifically bookings and cancellations. It is designed to:

- Listen to Calendly webhook events for new bookings and cancellations.
- Route incoming events to dedicated processing streams.
- Extract and transform relevant event data into structured formats.
- Log all events into separate Google Sheets tabs for bookings and cancellations.
- Send contextual Slack notifications with priority flags based on event timing.

**Target use cases**: Teams using Calendly for appointment scheduling who want automated real-time tracking and communication of booking status changes via Google Sheets logging and Slack alerts.

**Logical blocks:**

- **1.1 Event Reception and Routing**: Receive Calendly webhook events, route by event type (booking or cancellation).
- **1.2 Booking Data Processing**: Extract and format booking details, assess urgency, and label urgency status.
- **1.3 Booking Data Logging & Notification**: Log booking data to Google Sheets and notify via Slack with urgency label.
- **1.4 Cancellation Data Processing**: Extract cancellation details, analyze timing, and categorize cancellation urgency.
- **1.5 Cancellation Data Logging & Notification**: Log cancellation data to Google Sheets and notify Slack with cancellation context.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Reception and Routing

**Overview:**  
This block listens for Calendly webhook events (bookings and cancellations) and routes the data flow based on event type.

**Nodes Involved:**  
- Calendly Webhook Trigger1  
- Route Event Type  
- Sticky Note4 (documentation aid)

**Node Details:**

- **Calendly Webhook Trigger1**  
  - Type: Calendly Trigger  
  - Role: Receives webhook events for `invitee.created` (booking) and `invitee.canceled` (cancellation) using OAuth2 authentication.  
  - Config: Monitors the whole organization scope.  
  - Input: Incoming HTTP webhook from Calendly.  
  - Output: Passes event JSON to Route Event Type node.  
  - Failure modes: OAuth token expiration, webhook misconfiguration, network timeouts.  
  - Notes: Requires Calendly OAuth2 credentials with webhook event subscriptions.

- **Route Event Type**  
  - Type: Switch  
  - Role: Routes based on `event` field (`invitee.created`, `invitee.canceled`, or other).  
  - Config: Three output paths for booking, cancellation, and others.  
  - Input: JSON from Calendly Webhook Trigger.  
  - Output: Routes to Transform Booking Data or Transform Cancellation Data.  
  - Failure modes: Unexpected event type, missing event field.  
  - Notes: Critical for splitting logic into bookings vs cancellations.

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Documentation for event type detection logic.

---

#### 2.2 Booking Data Processing

**Overview:**  
Extracts detailed booking information, computes fields like urgency and formatted dates, and flags urgent bookings.

**Nodes Involved:**  
- Transform Booking Data  
- Check Urgency  
- Mark as Urgent  
- Mark as Normal  
- Merge Booking Data  
- Sticky Note (Capture Booked Appointments)

**Node Details:**

- **Transform Booking Data**  
  - Type: Set  
  - Role: Extracts and formats booking payload into structured fields: event ID, invitee details, event timing, meeting URL, guests, urgency markers, and status.  
  - Config: Uses JavaScript expressions to parse dates, compute durations, and handle missing names gracefully.  
  - Key expressions:  
    - `event_id` from URI last segment  
    - `invitee_name` fallback to first + last name or 'Guest'  
    - `formatted_date` and `formatted_time` localized with timezone  
    - `duration_minutes` computed from start/end time difference  
    - `days_until_event`, `is_same_day` as urgency indicators  
  - Input: JSON from Route Event Type for booking events.  
  - Output: Passes enriched booking JSON to Check Urgency.  
  - Edge cases: Missing scheduled_event object, malformed dates, empty guest lists.

- **Check Urgency**  
  - Type: If  
  - Role: Determines if booking is urgent (same-day or within next day).  
  - Config: Checks boolean `is_same_day` or if `days_until_event` <= 1.  
  - Input: Booking data from Transform Booking Data.  
  - Output: Routes to Mark as Urgent if true, else Mark as Normal.  
  - Edge cases: Timezone differences affecting date calculations.

- **Mark as Urgent**  
  - Type: Set  
  - Role: Flags booking as urgent with boolean `is_urgent` and label `🚨 URGENT`.  
  - Config: Preserves all other fields.  
  - Input: From Check Urgency true branch.  
  - Output: To Merge Booking Data.

- **Mark as Normal**  
  - Type: Set  
  - Role: Flags booking as normal (non-urgent) with `is_urgent`=false and label `📅 Scheduled`.  
  - Input: From Check Urgency false branch.  
  - Output: To Merge Booking Data.

- **Merge Booking Data**  
  - Type: Merge  
  - Role: Combines outputs from urgent and normal branches into one stream.  
  - Input: From Mark as Urgent and Mark as Normal.  
  - Output: Passes merged booking data downstream for logging and notification.

- **Sticky Note (Capture Booked Appointments)**  
  - Type: Sticky Note  
  - Role: Documentation for booking data extraction block.

---

#### 2.3 Booking Data Logging & Notification

**Overview:**  
Logs the processed booking data into Google Sheets and sends a formatted Slack message with urgency status.

**Nodes Involved:**  
- Log to Bookings Sheet1  
- Slack Booking Notification  
- Sticky Note2 (Log the Booked Appointments)

**Node Details:**

- **Log to Bookings Sheet1**  
  - Type: Google Sheets  
  - Role: Appends booking data as a new row to the `Bookings` tab in a Google spreadsheet.  
  - Config: Auto-maps all fields from input JSON to respective sheet columns.  
  - Input: Booking data from Merge Booking Data.  
  - Output: Passes data forward regardless of errors (onError: continue).  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: API quota limits, sheet access permissions, schema mismatch.

- **Slack Booking Notification**  
  - Type: Slack  
  - Role: Sends a Slack message to channel `#general` summarizing booking details with markdown.  
  - Config: Message includes urgency label, event name, invitee info, date/time, duration, location, guests, and meeting/reschedule/cancel links.  
  - Input: From Log to Bookings Sheet1.  
  - Output: Final node for booking pipeline.  
  - Credentials: Slack OAuth2 bot token with chat:write scope.  
  - Edge cases: Slack API rate limits, invalid channel, message formatting errors.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Documentation for booking data logging and notification.

---

#### 2.4 Cancellation Data Processing

**Overview:**  
Processes cancellation events by extracting cancellation details, computing timing relative to the original event, and categorizing urgency for follow-up.

**Nodes Involved:**  
- Transform Cancellation Data  
- Categorize Cancellation  
- Label: Last Minute  
- Label: Standard  
- Label: Past Event  
- Merge Cancellation Data  
- Sticky Note1 (Capture Cancelled Appointments)

**Node Details:**

- **Transform Cancellation Data**  
  - Type: Set  
  - Role: Extracts cancellation payload including invitee, event details, cancellation reason, who canceled, timing metrics (hours before event), and flags (last minute).  
  - Key computations:  
    - `hours_before_event` rounded difference in hours between event start and now  
    - `is_last_minute` flag if cancellation is within 24 hours before event  
  - Input: Cancellation event JSON from Route Event Type.  
  - Output: To Categorize Cancellation.  
  - Edge cases: Missing cancellation data, event start in the past.

- **Categorize Cancellation**  
  - Type: Switch  
  - Role: Routes cancellations into three categories: last minute, standard, or past event cancellations.  
  - Conditions:  
    - If `is_last_minute` = true → last minute branch  
    - Else if event is still upcoming (`was_upcoming` = true) → standard branch  
    - Else → past event branch  
  - Input: From Transform Cancellation Data.  
  - Output: To corresponding Label nodes.

- **Label: Last Minute**  
  - Type: Set  
  - Role: Flags cancellation as `last_minute` with label 🚨 LAST MINUTE and high follow-up priority.  
  - Output: To Merge Cancellation Data.

- **Label: Standard**  
  - Type: Set  
  - Role: Flags as `standard` cancellation with label ❌ Cancelled and normal priority.  
  - Output: To Merge Cancellation Data.

- **Label: Past Event**  
  - Type: Set  
  - Role: Flags as `past_event` cancellation with label 📋 Past Event Cancelled and low priority.  
  - Output: To Merge Cancellation Data.

- **Merge Cancellation Data**  
  - Type: Merge  
  - Role: Combines all cancellation labeling branches into one stream.  
  - Output: To logging node.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Documentation for cancellation data extraction and categorization.

---

#### 2.5 Cancellation Data Logging & Notification

**Overview:**  
Logs cancellation data into a dedicated Google Sheets tab and sends a Slack notification with cancellation details and follow-up priority.

**Nodes Involved:**  
- Log to Cancellations Sheet  
- Slack Cancellation Alert  
- Sticky Note3 (Log the Cancelled Appointments)

**Node Details:**

- **Log to Cancellations Sheet**  
  - Type: Google Sheets  
  - Role: Appends cancellation event data to the `Cancellations` tab in the tracking spreadsheet.  
  - Config: Auto-maps fields from cancellation JSON to sheet columns.  
  - Input: From Merge Cancellation Data.  
  - Output: Passes data to Slack notification node.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Same as booking logging.

- **Slack Cancellation Alert**  
  - Type: Slack  
  - Role: Posts a formatted message to Slack channel `#general` detailing cancellation specifics, including reason, who canceled, notice timing, and follow-up priority.  
  - Input: From Log to Cancellations Sheet.  
  - Output: Final node in cancellation pipeline.  
  - Credentials: Slack OAuth2 bot token.  
  - Edge cases: Slack API limits, message formatting.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Documentation for cancellation logging and notification.

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                           | Input Node(s)                | Output Node(s)                | Sticky Note                                             |
|----------------------------|--------------------|-----------------------------------------|-----------------------------|------------------------------|---------------------------------------------------------|
| 📋 WORKFLOW OVERVIEW       | Sticky Note        | General documentation                    |                             |                              | Explains workflow overview and Calendly webhook details |
| Calendly Webhook Trigger1  | Calendly Trigger   | Receives webhook events                  |                             | Route Event Type              |                                                         |
| Route Event Type           | Switch             | Routes events by type                    | Calendly Webhook Trigger1   | Transform Booking Data, Transform Cancellation Data | Sticky Note4 covers this node                            |
| Sticky Note4               | Sticky Note        | Documentation for event routing          |                             |                              | ## Detect the Event Type                                |
| Transform Booking Data     | Set                | Extracts and formats booking data       | Route Event Type (booking)  | Check Urgency                | Sticky Note (Capture Booked Appointments)               |
| Check Urgency              | If                 | Flags urgent bookings                    | Transform Booking Data      | Mark as Urgent, Mark as Normal |                                                         |
| Mark as Urgent             | Set                | Labels urgent bookings                   | Check Urgency (true branch) | Merge Booking Data           |                                                         |
| Mark as Normal             | Set                | Labels normal bookings                   | Check Urgency (false branch)| Merge Booking Data           |                                                         |
| Merge Booking Data         | Merge              | Combines urgent and normal bookings     | Mark as Urgent, Mark as Normal | Log to Bookings Sheet1      |                                                         |
| Log to Bookings Sheet1     | Google Sheets      | Logs booking data to spreadsheet        | Merge Booking Data          | Slack Booking Notification   | Sticky Note2                                             |
| Slack Booking Notification | Slack              | Sends booking notification to Slack     | Log to Bookings Sheet1      |                              |                                                         |
| Sticky Note                | Sticky Note        | Documentation for booked appointments   |                             |                              | ## Capture the Booked Appointments                       |
| Transform Cancellation Data| Set                | Extracts cancellation info and timing   | Route Event Type (cancellation) | Categorize Cancellation    | Sticky Note1                                             |
| Categorize Cancellation    | Switch             | Routes cancellations by urgency         | Transform Cancellation Data | Label: Last Minute, Label: Standard, Label: Past Event |                                                         |
| Label: Last Minute         | Set                | Labels last-minute cancellations        | Categorize Cancellation     | Merge Cancellation Data      |                                                         |
| Label: Standard            | Set                | Labels standard cancellations            | Categorize Cancellation     | Merge Cancellation Data      |                                                         |
| Label: Past Event          | Set                | Labels cancellations for past events    | Categorize Cancellation     | Merge Cancellation Data      |                                                         |
| Merge Cancellation Data    | Merge              | Combines cancellation categories        | Label: Last Minute, Label: Standard, Label: Past Event | Log to Cancellations Sheet |                                                         |
| Log to Cancellations Sheet | Google Sheets      | Logs cancellation data to spreadsheet   | Merge Cancellation Data     | Slack Cancellation Alert     | Sticky Note3                                             |
| Slack Cancellation Alert   | Slack              | Sends cancellation notification to Slack| Log to Cancellations Sheet  |                              |                                                         |
| Sticky Note1               | Sticky Note        | Documentation for cancellation handling |                             |                              | ## Capture the Cancelled Appointments                    |
| Sticky Note2               | Sticky Note        | Documentation for booking logging       |                             |                              | ## Log the Booked Appointments                           |
| Sticky Note3               | Sticky Note        | Documentation for cancellation logging  |                             |                              | ## Log the Cancelled Appointments                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Calendly Webhook Trigger node**  
   - Type: Calendly Trigger  
   - Configure OAuth2 credentials for Calendly API (register app at developer.calendly.com)  
   - Scope: Organization  
   - Events: `invitee.created`, `invitee.canceled`  
   - Position: Start node

2. **Add Switch node: Route Event Type**  
   - Input from Calendly Trigger node  
   - Conditions:  
     - `event` equals `invitee.created` → output 1 (booking)  
     - `event` equals `invitee.canceled` → output 2 (cancellation)  
     - Otherwise → output 3 (others)  
   - Purpose: route between booking and cancellation workflows

3. **Booking branch: Transform Booking Data (Set node)**  
   - Extract and compute fields from payload: event_id, invitee_email, invitee_name, first_name, event_name, start/end ISO dates, timezone, formatted date/time strings, duration_minutes, meeting_url, reschedule and cancel URLs, location_type, guests_count, guests_emails, questions_json, days_until_event, is_same_day, event_type (`new_booking`), processed_at (ISO now), status (`confirmed`).  
   - Use JS expressions for date/time formatting and default fallbacks.

4. **Check Urgency (If node)**  
   - Condition: `is_same_day` is true OR `days_until_event` <= 1  
   - True: urgent branch  
   - False: normal branch

5. **Mark as Urgent (Set node)**  
   - Assign `is_urgent` = true  
   - Assign `urgency_label` = "🚨 URGENT"  
   - Include other fields from previous node

6. **Mark as Normal (Set node)**  
   - Assign `is_urgent` = false  
   - Assign `urgency_label` = "📅 Scheduled"  
   - Include other fields from previous node

7. **Merge Booking Data (Merge node)**  
   - Inputs from Mark as Urgent and Mark as Normal  
   - Combine streams for unified downstream processing

8. **Log to Bookings Sheet (Google Sheets node)**  
   - Operation: Append  
   - Document: Your Google Sheet ID (create spreadsheet with tab named `Bookings`)  
   - Sheet: Tab `Bookings` (gid=0)  
   - Auto-map all booking data fields to sheet columns  
   - Use Google Sheets OAuth2 credentials with write access

9. **Slack Booking Notification (Slack node)**  
   - Channel: `#general` (or your preferred channel)  
   - Message text: Markdown with urgency label, event name, invitee info, formatted date/time, duration, timezone, location, guests, meeting link, reschedule and cancel links  
   - Use Slack OAuth2 bot credentials with chat permissions  
   - Connect output of Log to Bookings Sheet node

10. **Cancellation branch: Transform Cancellation Data (Set node)**  
    - Extract fields: event_id, invitee_email, invitee_name, event_name, original_date/time (formatted), cancellation_reason, canceled_by, canceler_type, was_upcoming (event start > now), hours_before_event (rounded hours), is_last_minute (hours_before_event < 24 and event in future), event_type (`cancellation`), canceled_at (ISO now)  
    - Use JS expressions for date/time and boolean calculations

11. **Categorize Cancellation (Switch node)**  
    - Conditions:  
      - `is_last_minute` = true → last minute branch  
      - `was_upcoming` = true → standard branch  
      - Else → past event branch

12. **Label: Last Minute (Set node)**  
    - Assign `cancel_category` = "last_minute"  
    - Assign `cancel_label` = "🚨 LAST MINUTE"  
    - Assign `follow_up_priority` = "high"  
    - Include other fields

13. **Label: Standard (Set node)**  
    - Assign `cancel_category` = "standard"  
    - Assign `cancel_label` = "❌ Cancelled"  
    - Assign `follow_up_priority` = "normal"  
    - Include other fields

14. **Label: Past Event (Set node)**  
    - Assign `cancel_category` = "past_event"  
    - Assign `cancel_label` = "📋 Past Event Cancelled"  
    - Assign `follow_up_priority` = "low"  
    - Include other fields

15. **Merge Cancellation Data (Merge node)**  
    - Inputs from all three labeling nodes  
    - Combine into single stream for logging and notification

16. **Log to Cancellations Sheet (Google Sheets node)**  
    - Operation: Append  
    - Document: Same Google Sheet ID as booking, tab named `Cancellations` (gid=272696534 or create accordingly)  
    - Auto-map cancellation data fields to sheet columns  
    - Use Google Sheets OAuth2 credentials

17. **Slack Cancellation Alert (Slack node)**  
    - Channel: `#general`  
    - Message text: Markdown including cancel_label, event name, invitee info, original scheduled date/time, cancellation reason, who canceled, notice timing (hours before event or after), and follow-up priority  
    - Slack OAuth2 bot credentials

18. **Connect streams properly**:  
    - Calendly Webhook Trigger → Route Event Type  
    - Route Event Type output 1 → Transform Booking Data → Check Urgency → Mark as Urgent/Normal → Merge Booking Data → Log to Bookings Sheet → Slack Booking Notification  
    - Route Event Type output 2 → Transform Cancellation Data → Categorize Cancellation → Label nodes → Merge Cancellation Data → Log to Cancellations Sheet → Slack Cancellation Alert  

19. **Add sticky notes for documentation** as per original workflow to enhance clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                        |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Calendly Webhook API Documentation and event details                                                        | https://developer.calendly.com                         |
| Slack API app setup with Bot Token Scopes and OAuth2 credentials required for sending messages                | https://api.slack.com                                  |
| Google Sheets: Create spreadsheet with tabs named `Bookings` and `Cancellations`                             |                                                       |
| OAuth2 recommended for Calendly API (API keys deprecated May 2025)                                          |                                                       |
| Workflow designed for organizational-level Calendly accounts with webhook scope set to "organization"       |                                                       |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated n8n workflow. All data processed is legal and public, and content complies with current content policies without illegal or offensive elements.