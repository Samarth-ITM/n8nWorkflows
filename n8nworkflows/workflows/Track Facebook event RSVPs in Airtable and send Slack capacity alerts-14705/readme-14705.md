Track Facebook event RSVPs in Airtable and send Slack capacity alerts

https://n8nworkflows.xyz/workflows/track-facebook-event-rsvps-in-airtable-and-send-slack-capacity-alerts-14705


# Track Facebook event RSVPs in Airtable and send Slack capacity alerts

# 1. Workflow Overview

This workflow receives Facebook Event RSVP notifications through a webhook, normalizes the payload, stores attendee RSVP data in Airtable, counts confirmed attendees for the event, and sends a Slack alert once capacity is exceeded.

Primary use case:
- Track live RSVP changes for Facebook events
- Maintain attendee state in Airtable
- Notify a team in Slack when attendance goes beyond a fixed threshold
- Avoid sending duplicate capacity alerts

Logical blocks:

## 1.1 Input Reception and Normalization
The workflow starts from an HTTP webhook that receives Facebook RSVP event payloads. A Set node extracts only the relevant fields needed downstream: event ID, user ID, RSVP status, and processing timestamp.

## 1.2 RSVP Persistence in Airtable
The normalized RSVP data is inserted or updated in Airtable using an upsert operation. This ensures the attendee’s latest RSVP status is stored without creating unnecessary duplicates.

## 1.3 Attending Count and Capacity Check
After persistence, the workflow queries Airtable for all records for the same event where RSVP status is `attending`. It then compares the count of those records against a hardcoded event capacity threshold of `50`.

## 1.4 Alert Deduplication
If capacity is exceeded, the workflow checks whether a capacity alert has already been marked as sent. This logic is intended to prevent repeated Slack alerts for the same event.

## 1.5 Team Notification and Airtable State Update
If no prior alert is detected, the workflow posts a Slack message to a selected channel and then updates Airtable to mark that the capacity alert has been sent.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Normalization

### Overview
This block receives the Facebook webhook payload and extracts the key fields needed for storage and later capacity logic. It transforms a nested Facebook event structure into a flatter item shape.

### Nodes Involved
- Facebook Event RSVP Webhook
- Edit Fields

### Node Details

#### Facebook Event RSVP Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that accepts HTTP POST requests from Facebook or another sender emulating Facebook RSVP events.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `facebook-event-rsvp`
  - No special response options configured
- **Key expressions or variables used:** None in the node itself
- **Input and output connections:**
  - Input: none, this is a trigger node
  - Output: `Edit Fields`
- **Version-specific requirements:**
  - Uses `typeVersion 2.1`
  - Webhook URL must be active in n8n and reachable from Facebook
- **Edge cases or potential failure types:**
  - Facebook webhook verification is not implemented here; this node appears to only handle POST event delivery
  - Payload shape mismatches will break downstream expressions
  - Missing `entry[0].changes[0].value` fields will cause undefined values in the next node
  - If the workflow is inactive, production webhook calls will not run
- **Sub-workflow reference:** None

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes the incoming webhook payload by assigning simplified top-level fields.
- **Configuration choices:**
  - Creates four fields:
    - `eventId` from `{{$json.entry[0].changes[0].value.event_id}}`
    - `userId` from `{{$json.entry[0].changes[0].value.user_id}}`
    - `rsvpStatus` from `{{$json.entry[0].changes[0].value.rsvp_status}}`
    - `receivedAt` from `{{new Date().toISOString()}}`
- **Key expressions or variables used:**
  - `$json.entry[0].changes[0].value.event_id`
  - `$json.entry[0].changes[0].value.user_id`
  - `$json.entry[0].changes[0].value.rsvp_status`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input: `Facebook Event RSVP Webhook`
  - Output: `Upsert RSVP in Airtable`
- **Version-specific requirements:**
  - Uses `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - If Facebook sends multiple entries or multiple changes in one payload, this node only processes the first one
  - Expression failures can occur if `entry`, `changes`, or `value` are absent
  - `receivedAt` reflects processing time, not necessarily Facebook event creation time
- **Sub-workflow reference:** None

---

## 2.2 RSVP Persistence in Airtable

### Overview
This block stores the normalized RSVP data in Airtable. The intent is to keep one logical record per attendee/event combination, but the actual matching configuration does not fully enforce that.

### Nodes Involved
- Upsert RSVP in Airtable

### Node Details

#### Upsert RSVP in Airtable
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Writes RSVP data to Airtable using an upsert operation.
- **Configuration choices:**
  - Operation: `upsert`
  - Base: `Fake Review Detector`
  - Table: `LogAtendeeData`
  - Mapped fields:
    - `user_Id` = `{{$json.userId}}`
    - `event_Id` = `{{$json.eventId}}`
    - `receivedAt` = `{{$json.receivedAt}}`
    - `rsvpStatus` = `{{$json.rsvpStatus}}`
  - Matching column: `event_Id`
- **Key expressions or variables used:**
  - `$json.userId`
  - `$json.eventId`
  - `$json.receivedAt`
  - `$json.rsvpStatus`
- **Input and output connections:**
  - Input: `Edit Fields`
  - Output: `Fetch Attending RSVPs for Event`
- **Version-specific requirements:**
  - Uses `typeVersion 2.1`
  - Requires Airtable Personal Access Token credentials
- **Edge cases or potential failure types:**
  - **Important logic issue:** Matching only on `event_Id` means all attendees for the same event may target the same Airtable record instead of unique attendee-event combinations
  - If the goal is one row per attendee per event, matching should likely include both `event_Id` and `user_Id`
  - Airtable auth failure or revoked token will stop execution
  - Field names are case-sensitive and must exist in Airtable
  - Type conversion is disabled, so incompatible data types may fail
- **Sub-workflow reference:** None

---

## 2.3 Attending Count and Capacity Check

### Overview
This block retrieves all Airtable rows for the current event where the RSVP status is `attending`, then checks whether the total exceeds the configured capacity threshold of 50.

### Nodes Involved
- Fetch Attending RSVPs for Event
- Is Event Capacity Exceeded?

### Node Details

#### Fetch Attending RSVPs for Event
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Searches Airtable for all attendee rows matching the event and having an `attending` status.
- **Configuration choices:**
  - Operation: `search`
  - Base: `Fake Review Detector`
  - Table: `LogAtendeeData`
  - `filterByFormula`:
    ```text
    =AND(
      {event_Id} = '{{ $json.fields.event_Id }}',
      {rsvpStatus} = 'attending'
    )
    ```
- **Key expressions or variables used:**
  - `$json.fields.event_Id`
- **Input and output connections:**
  - Input: `Upsert RSVP in Airtable`
  - Output: `Is Event Capacity Exceeded?`
- **Version-specific requirements:**
  - Uses `typeVersion 2.1`
  - Requires Airtable credentials
- **Edge cases or potential failure types:**
  - Depends on the output shape of the previous Airtable upsert node
  - This expression assumes the upsert result contains `fields.event_Id`; that is usually valid for Airtable record output, but should be verified in the target n8n version
  - If the upsert configuration is wrong and only one record exists per event, the count will be inaccurate
  - Airtable formula escaping may be an issue if event IDs contain apostrophes or unusual characters
- **Sub-workflow reference:** None

#### Is Event Capacity Exceeded?
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the number of matching Airtable items is greater than 50.
- **Configuration choices:**
  - Condition: `{{$items().length}} > 50`
  - Number comparison using strict validation
- **Key expressions or variables used:**
  - `$items().length`
- **Input and output connections:**
  - Input: `Fetch Attending RSVPs for Event`
  - Output true branch: `Is Capacity Alert Already Sent?`
  - Output false branch: none
- **Version-specific requirements:**
  - Uses `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - The condition uses `gt 50`, so exactly 50 attendees does **not** trigger the alert; only 51 or more does
  - `$items().length` counts all items entering the IF node, which is appropriate here
  - If Airtable pagination/search behavior changes, count may not reflect the full table without proper handling
- **Sub-workflow reference:** None

---

## 2.4 Alert Deduplication

### Overview
This block attempts to determine whether a capacity alert has already been sent. If the alert flag is true or empty, one branch is taken; otherwise Slack notification proceeds.

### Nodes Involved
- Is Capacity Alert Already Sent?

### Node Details

#### Is Capacity Alert Already Sent?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branching node intended to prevent duplicate Slack alerts.
- **Configuration choices:**
  - OR condition:
    - `{{$json.capacity_alert_sent}}` is `true`
    - `{{$json.capacity_alert_sent}}` is empty
  - True branch has no downstream node
  - False branch sends Slack alert
- **Key expressions or variables used:**
  - `$json.capacity_alert_sent`
- **Input and output connections:**
  - Input: `Is Event Capacity Exceeded?` true branch
  - Output true branch: none
  - Output false branch: `Send Slack Capacity Alert`
- **Version-specific requirements:**
  - Uses `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - **Important logic issue:** The current logic appears inverted for the empty case
    - If `capacity_alert_sent` is empty, the node evaluates true and stops, which prevents Slack from sending
    - Usually, empty should mean “no alert has been sent yet,” so Slack should likely be sent on that path
  - Since `Fetch Attending RSVPs for Event` returns attendee records, many or all items may not have a `capacity_alert_sent` field populated
  - This can suppress alerts unintentionally
  - Multi-item input may lead to item-wise branching that does not reflect event-level state cleanly
- **Sub-workflow reference:** None

---

## 2.5 Team Notification and Airtable State Update

### Overview
If the workflow decides an alert is required, it posts a Slack message and then updates Airtable to record that a capacity alert has been sent.

### Nodes Involved
- Send Slack Capacity Alert
- Mark Capacity Alert as Sent

### Node Details

#### Send Slack Capacity Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a message to a Slack channel notifying the team that event capacity has been exceeded.
- **Configuration choices:**
  - Message text:
    `Event Capacity Alert Event ID: {{ $json.event_Id }} Attendees: {{$items().length}} / 50`
  - Target: specific Slack channel
  - Channel selected: `n8n`
- **Key expressions or variables used:**
  - `$json.event_Id`
  - `$items().length`
- **Input and output connections:**
  - Input: `Is Capacity Alert Already Sent?` false branch
  - Output: `Mark Capacity Alert as Sent`
- **Version-specific requirements:**
  - Uses `typeVersion 2.3`
  - Requires Slack API credentials
- **Edge cases or potential failure types:**
  - Slack credential issues or insufficient bot permissions can block message delivery
  - If multiple items reach this node, multiple Slack messages may be sent
  - `$items().length` reflects all items available in the current execution context, not necessarily a deduplicated event count at this stage
  - Message formatting is minimal and may render without clear spacing because the configured text is compact
- **Sub-workflow reference:** None

#### Mark Capacity Alert as Sent
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Updates an Airtable record to set `capacity_alert_sent` to `true`.
- **Configuration choices:**
  - Operation: `upsert`
  - Base: `Fake Review Detector`
  - Table: `LogAtendeeData`
  - Mapped fields:
    - `id` = `{{ $('Is Event Capacity Exceeded?').item.json.id }}`
    - `capacity_alert_sent` = `true`
  - Matching column: `id`
- **Key expressions or variables used:**
  - `$('Is Event Capacity Exceeded?').item.json.id`
- **Input and output connections:**
  - Input: `Send Slack Capacity Alert`
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion 2.1`
  - Requires Airtable credentials
- **Edge cases or potential failure types:**
  - **Important logic issue:** `Is Event Capacity Exceeded?` receives search results, meaning there may be many items; referencing `.item.json.id` may select only one record
  - Marking only one attendee row as `capacity_alert_sent = true` does not reliably mark the whole event as alerted
  - If the referenced item lacks `id`, the upsert fails
  - Event-level deduplication is being stored on attendee-level records, which is structurally fragile
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Facebook Event RSVP Webhook | Webhook | Receives Facebook RSVP POST payloads |  | Edit Fields | ## Receive Event RSVP<br>This section captures new RSVP responses from a Facebook event and prepares the attendee data in a clean, usable format for further processing. |
| Edit Fields | Set | Extracts and normalizes eventId, userId, rsvpStatus, receivedAt | Facebook Event RSVP Webhook | Upsert RSVP in Airtable | ## Receive Event RSVP<br>This section captures new RSVP responses from a Facebook event and prepares the attendee data in a clean, usable format for further processing. |
| Upsert RSVP in Airtable | Airtable | Stores or updates RSVP data in Airtable | Edit Fields | Fetch Attending RSVPs for Event | ## Save Attendee Details<br>Stores or updates attendee RSVP information in Airtable, ensuring no duplicate records are created and the latest RSVP status is always saved. |
| Fetch Attending RSVPs for Event | Airtable | Searches all attending RSVPs for the same event | Upsert RSVP in Airtable | Is Event Capacity Exceeded? | ## Event Capacity Calculation<br>Counts how many people are attending the event and checks whether the total number has reached or crossed the maximum allowed capacity. |
| Is Event Capacity Exceeded? | If | Checks whether attending count is greater than 50 | Fetch Attending RSVPs for Event | Is Capacity Alert Already Sent? | ## Event Capacity Calculation<br>Counts how many people are attending the event and checks whether the total number has reached or crossed the maximum allowed capacity. |
| Is Capacity Alert Already Sent? | If | Attempts to suppress duplicate capacity alerts | Is Event Capacity Exceeded? | Send Slack Capacity Alert | ## Alert Control & Deduplication<br>Checks whether a capacity alert has already been sent for the event to prevent duplicate notifications when additional RSVPs arrive after capacity is reached. |
| Send Slack Capacity Alert | Slack | Sends Slack notification when capacity is exceeded | Is Capacity Alert Already Sent? | Mark Capacity Alert as Sent | ## Team Notification & State Update<br>Sends a real-time Slack alert to the team when event capacity is crossed and updates Airtable to mark the alert as sent for future executions. |
| Mark Capacity Alert as Sent | Airtable | Marks alert state in Airtable | Send Slack Capacity Alert |  | ## Team Notification & State Update<br>Sends a real-time Slack alert to the team when event capacity is crossed and updates Airtable to mark the alert as sent for future executions. |
| Sticky Note | Sticky Note | Documentation note for input block |  |  |  |
| Sticky Note1 | Sticky Note | Documentation note for Airtable persistence block |  |  |  |
| Sticky Note2 | Sticky Note | Documentation note for capacity calculation block |  |  |  |
| Sticky Note3 | Sticky Note | Documentation note for alert deduplication block |  |  |  |
| Sticky Note4 | Sticky Note | Documentation note for notification block |  |  |  |
| Sticky Note5 | Sticky Note | Global explanatory note with setup summary |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Facebook Event RSVP Tracking with Airtable Capacity Control & Slack Alerts`.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - Name: `Facebook Event RSVP Webhook`
   - HTTP Method: `POST`
   - Path: `facebook-event-rsvp`
   - Leave response handling at default unless you want a custom response for Facebook.
   - Activate the workflow later so the production webhook URL becomes available.

3. **Add a Set node after the webhook**
   - Node type: `Set`
   - Name: `Edit Fields`
   - Create these fields:
     - `eventId` as String: `={{ $json.entry[0].changes[0].value.event_id }}`
     - `userId` as String: `={{ $json.entry[0].changes[0].value.user_id }}`
     - `rsvpStatus` as String: `={{ $json.entry[0].changes[0].value.rsvp_status }}`
     - `receivedAt` as String: `={{ new Date().toISOString() }}`
   - Connect: `Facebook Event RSVP Webhook -> Edit Fields`

4. **Prepare Airtable**
   - In Airtable, create or use:
     - A base
     - A table named `LogAtendeeData`
   - Add at minimum these fields:
     - `event_Id` text
     - `user_Id` text
     - `rsvpStatus` text
     - `receivedAt` text or date-time
     - `capacity_alert_sent` checkbox or boolean
   - Strongly recommended:
     - Add a unique event-attendee key or configure matching on both `event_Id` and `user_Id` if your n8n Airtable node supports it cleanly.

5. **Add an Airtable node to store the RSVP**
   - Node type: `Airtable`
   - Name: `Upsert RSVP in Airtable`
   - Credentials: Airtable Personal Access Token
   - Operation: `Upsert`
   - Select your base and table
   - Map fields:
     - `user_Id` = `={{ $json.userId }}`
     - `event_Id` = `={{ $json.eventId }}`
     - `receivedAt` = `={{ $json.receivedAt }}`
     - `rsvpStatus` = `={{ $json.rsvpStatus }}`
   - Matching columns in the original workflow: `event_Id`
   - Better practical option: use both `event_Id` and `user_Id`, or a composite unique field
   - Connect: `Edit Fields -> Upsert RSVP in Airtable`

6. **Add an Airtable search node to count attendees**
   - Node type: `Airtable`
   - Name: `Fetch Attending RSVPs for Event`
   - Credentials: same Airtable credential
   - Operation: `Search`
   - Select same base and table
   - Filter formula:
     ```text
     =AND(
       {event_Id} = '{{ $json.fields.event_Id }}',
       {rsvpStatus} = 'attending'
     )
     ```
   - Connect: `Upsert RSVP in Airtable -> Fetch Attending RSVPs for Event`

7. **Add an IF node for capacity**
   - Node type: `If`
   - Name: `Is Event Capacity Exceeded?`
   - Condition type: Number
   - Left value: `={{ $items().length }}`
   - Operator: `Greater Than`
   - Right value: `50`
   - Connect: `Fetch Attending RSVPs for Event -> Is Event Capacity Exceeded?`

8. **Add an IF node for deduplication**
   - Node type: `If`
   - Name: `Is Capacity Alert Already Sent?`
   - Configure OR logic with these conditions:
     - `={{ $json.capacity_alert_sent }}` is `true`
     - `={{ $json.capacity_alert_sent }}` is `empty`
   - Connect the **true** output of `Is Event Capacity Exceeded?` to this node
   - In the original workflow:
     - true branch stops
     - false branch sends Slack
   - Important: this logic is likely flawed if empty should mean “not yet sent.” If rebuilding for production, invert this behavior.

9. **Create Slack credentials**
   - Add a Slack app/bot with permission to post into the target channel
   - Connect it in n8n as a Slack credential

10. **Add a Slack node**
    - Node type: `Slack`
    - Name: `Send Slack Capacity Alert`
    - Operation: send message to channel
    - Select target channel: `n8n` or your chosen channel
    - Message text:
      ```text
      = Event Capacity Alert  Event ID:  {{ $json.event_Id }}Attendees: {{$items().length}} / 50
      ```
    - You may want to improve formatting, for example:
      ```text
      Event Capacity Alert
      Event ID: {{ $json.event_Id }}
      Attendees: {{ $items().length }} / 50
      ```
    - Connect the **false** output of `Is Capacity Alert Already Sent?` to this node

11. **Add a final Airtable node to mark alert state**
    - Node type: `Airtable`
    - Name: `Mark Capacity Alert as Sent`
    - Credentials: same Airtable credential
    - Operation: `Upsert`
    - Base/table: same as earlier
    - Map fields:
      - `id` = `={{ $('Is Event Capacity Exceeded?').item.json.id }}`
      - `capacity_alert_sent` = `true`
    - Matching column: `id`
    - Connect: `Send Slack Capacity Alert -> Mark Capacity Alert as Sent`

12. **Add sticky notes if desired**
    - Add documentation notes matching the workflow sections:
      - Receive Event RSVP
      - Save Attendee Details
      - Event Capacity Calculation
      - Alert Control & Deduplication
      - Team Notification & State Update
      - Global “How This Workflow Works” note

13. **Test with sample payload**
    - Example structure:
      ```json
      {
        "entry": [
          {
            "id": "1234567890",
            "time": 1734856789,
            "changes": [
              {
                "field": "events",
                "value": {
                  "user_id": "9988776655",
                  "event_id": "61584948043418",
                  "rsvp_status": "attending"
                }
              }
            ]
          }
        ],
        "object": "page"
      }
      ```
   - Send it to the test webhook URL first.

14. **Verify Airtable behavior**
   - Confirm whether each RSVP creates or updates the intended record
   - Check especially whether matching on `event_Id` alone overwrites prior attendees for the same event

15. **Verify capacity logic**
   - Ensure the Airtable search returns one item per attendee with `rsvpStatus = attending`
   - Confirm whether the threshold should be:
     - `> 50` as in the current workflow, or
     - `>= 50` if alert should fire exactly at capacity

16. **Verify deduplication logic**
   - Check whether `capacity_alert_sent` exists on searched records
   - If not, the current IF logic may suppress alerts unexpectedly
   - For production, a better design is:
     - store alert state in a dedicated event-level table, or
     - update all rows for an event, or
     - search specifically for an event-level alert flag record

17. **Activate the workflow**
   - Once credentials, Airtable schema, and webhook endpoint are confirmed, activate the workflow so the production webhook URL can receive live events.

### Credential configuration summary
- **Airtable**
  - Use a Personal Access Token with read/write access to the target base/table
- **Slack**
  - Use a Slack credential with permission to post messages to the selected channel
- **Facebook**
  - No native Facebook node is used here; Facebook must be configured externally to send POST requests to the n8n webhook URL

### Input/output expectations
- **Input:** Facebook-style RSVP webhook payload
- **Intermediate storage:** Airtable attendee record(s)
- **Output:** Optional Slack alert plus Airtable update of `capacity_alert_sent`

### No sub-workflows
- This workflow does not invoke any sub-workflow and has only one entry point: the webhook node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow starts automatically whenever a new RSVP is received for a Facebook Event. The RSVP data is captured through a webhook and cleaned to keep only important details like event ID, user ID, RSVP status and time. Attendee information is saved in Airtable using upsert logic, which prevents duplicate records and keeps RSVP status up to date. The workflow then counts how many people are marked as attending for the event. This count is compared against the event’s maximum capacity. If the capacity is reached or exceeded, the workflow checks whether the team has already been notified. If no alert was sent earlier, a Slack message is sent to inform the team that the event is full. Finally, Airtable is updated to record that the capacity alert has been sent, preventing duplicate notifications. | Global workflow description |
| Create a webhook to receive Facebook RSVP events. Normalize the incoming data using a Set node. Store or update attendee records in Airtable. Count confirmed attendees and compare with event capacity. Send a Slack alert when capacity is reached and mark it as sent. | Global setup summary |

## Implementation warnings
- The current upsert key likely does not support one-record-per-attendee correctly because it matches only on `event_Id`.
- The current deduplication IF logic likely prevents alerts when `capacity_alert_sent` is empty.
- The current alert-state storage is attendee-level, not event-level, which makes deduplication unreliable.
- The capacity check triggers only when count is strictly greater than 50, not equal to 50.