Google Calendar to KlickTipp: Sync Event & Attendee Status for Contact Management

https://n8nworkflows.xyz/workflows/google-calendar-to-klicktipp--sync-event---attendee-status-for-contact-management-10088


# Google Calendar to KlickTipp: Sync Event & Attendee Status for Contact Management

### 1. Workflow Overview

This workflow synchronizes event and attendee statuses from Google Calendar to KlickTipp contact management. It automates the creation and updating of KlickTipp contacts based on Google Calendar events and attendees, reflecting real-time RSVP statuses and event changes. The workflow is ideal for coaches, consultants, and event organizers who want to maintain up-to-date contact lists for marketing automation triggered by calendar events.

Logical blocks:

- **1.1 Event Data Acquisition:** Listening to Google Calendar for new, updated, or cancelled events.
- **1.2 Attendee Segregation:** Splitting the list of attendees from events into individual records.
- **1.3 Attendee Filtering:** Filtering out attendees based on email domain exclusions.
- **1.4 Status Routing:** Routing attendees by their RSVP status (accepted, declined, tentative, etc.).
- **1.5 KlickTipp Contact Management:** Creating or updating contacts and tagging them in KlickTipp according to event status and attendee response.
- **1.6 Event Data Saving:** (Implied by sticky note, no explicit separate node) Mapping event metadata to KlickTipp custom fields.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Data Acquisition

- **Overview:**  
  Listens for Google Calendar events with different triggers: new event creation, event updates, and event cancellations. Each trigger independently initiates processing of the relevant event data.

- **Nodes Involved:**  
  - Watch new Google Calendar events  
  - Watch updated Google Calendar events  
  - Watch cancelled Google Calendar events

- **Node Details:**

  1. **Watch new Google Calendar events**  
     - Type: Google Calendar Trigger  
     - Role: Triggers on newly created events in a specified Google Calendar.  
     - Configuration: Polls every minute, listens for "eventCreated" trigger on a specific calendar ID.  
     - Input: N/A (trigger node)  
     - Output: Emits event data including event details and attendees.  
     - Edge Cases: API rate limits, calendar access permissions, empty attendee lists.  
     - Credentials: Google Calendar OAuth2 API.

  2. **Watch updated Google Calendar events**  
     - Type: Google Calendar Trigger  
     - Role: Triggers on updates to existing events.  
     - Configuration: Polls every minute, listens for "eventUpdated" trigger on the same calendar ID.  
     - Output includes updated event metadata and attendees.  
     - Edge Cases: Partial updates, status changes, concurrency conflicts.  
     - Credentials: Google Calendar OAuth2 API.

  3. **Watch cancelled Google Calendar events**  
     - Type: Google Calendar Trigger  
     - Role: Triggers on event cancellations.  
     - Configuration: Polls every minute, triggers on "eventCancelled" for the same calendar.  
     - Edge Cases: Cancellation propagation delays, missing attendees on cancelled events.  
     - Credentials: Google Calendar OAuth2 API.

---

#### 2.2 Attendee Segregation

- **Overview:**  
  Splits the attendees array from each event into individual attendee items to process them separately.

- **Nodes Involved:**  
  - Iterate through attendees (event created)  
  - Iterate through attendees (status changed)  
  - Iterate through attendees (event canceled)

- **Node Details:**

  1. **Iterate through attendees (event created)**  
     - Type: Split Out (Split Array Field)  
     - Role: Iterates over the "attendees" array in new events, outputting one attendee per execution.  
     - Input: Event data from new event trigger.  
     - Output: Single attendee JSON with event context.  
     - Edge Cases: Events without attendees, empty arrays.

  2. **Iterate through attendees (status changed)**  
     - Same as above but for updated events.

  3. **Iterate through attendees (event canceled)**  
     - Same as above but for cancelled events.

---

#### 2.3 Attendee Filtering

- **Overview:**  
  Filters out attendees whose email addresses contain blacklisted domains, e.g., internal or test domains like "@example.com".

- **Nodes Involved:**  
  - Filter email domain (new event)  
  - Filter email domain (updated event)  
  - Filter email domain (canceled event)

- **Node Details:**

  Each is a Filter node with a simple condition:

  - Checks if attendee email does **not** contain "@example.com" (case-sensitive, strict validation).  
  - Input: Individual attendee JSON from split nodes.  
  - Output: Passes attendees with allowed email domains; filters out others.  
  - Edge Cases: Missing or malformed emails, case sensitivity issues.

---

#### 2.4 Status Routing

- **Overview:**  
  Routes attendees based on their RSVP responseStatus to apply appropriate KlickTipp tags and updates.

- **Nodes Involved:**  
  - Route by attendees' status (Switch node)

- **Node Details:**

  - Type: Switch  
  - Role: Determines attendee responseStatus:  
    - "declined" → output "declined"  
    - "accepted" → output "accepted"  
    - "tentative" → output "considered"  
    - "needsAction" → output "event updated" (e.g., no response yet)  
  - Input: Filtered attendee data from updated events.  
  - Output: Routes to corresponding KlickTipp tagging nodes or contact update.  
  - Edge Cases: Unknown or missing responseStatus, unexpected status values.

---

#### 2.5 KlickTipp Contact Management

- **Overview:**  
  Creates or updates contacts in KlickTipp for each attendee, applying tags based on event and attendee status, and mapping event details into custom fields.

- **Nodes Involved:**  
  - Create or update contact for attendee (new event)  
  - Create or update contact for attendee (updated event)  
  - Transfer attendees cancellations  
  - Transfer the attendees' declines  
  - Transfer the attendees' acceptances  
  - Transfer the attendees' considerations

- **Node Details:**

  1. **Create or update contact for attendee (new event)**  
     - Type: KlickTipp node (subscriber subscribe operation)  
     - Role: Creates or updates KlickTipp subscriber for new event attendees.  
     - Fields mapped include: event summary, description, location, start/end timestamps (Unix seconds), first and last name parsed from displayName.  
     - Tag applied: "Google Calendar | event created/updated" (tagId "13642952").  
     - Input: Filtered attendee JSON.  
     - Edge Cases: KlickTipp API errors, missing fields, name parsing edge cases.  
     - Credentials: KlickTipp API (username/password).

  2. **Create or update contact for attendee (updated event)**  
     - Same configuration and fields as above, triggered for updated events.

  3. **Transfer attendees cancellations**  
     - Type: KlickTipp node (contact-tagging)  
     - Role: Adds "Google Calendar | event canceled" tag (tagId "13642439") to attendees of cancelled events.  
     - Input: Filtered attendees from cancelled events.  
     - Edge Cases: Tagging failures, API errors.

  4. **Transfer the attendees' declines**  
     - Type: KlickTipp node (contact-tagging)  
     - Role: Tags declined attendees with "Google Calendar | event declined" (tagId "13642445").  
     - Input: Routed attendees with "declined" status.

  5. **Transfer the attendees' acceptances**  
     - Type: KlickTipp node (contact-tagging)  
     - Role: Tags accepted attendees with "Google Calendar | event confirmed" (tagId "13667113").  
     - Input: Routed attendees with "accepted" status.

  6. **Transfer the attendees' considerations**  
     - Type: KlickTipp node (contact-tagging)  
     - Role: Tags tentative attendees with "Google Calendar | event considered" (tagId "13667652").  
     - Input: Routed attendees with "tentative" status.

---

#### 2.6 Event Data Saving

- **Overview:**  
  While not a distinct node, mapping event details to KlickTipp custom fields and tagging contacts occurs during contact creation or updating nodes.

- **Nodes Involved:**  
  - Covered in "Create or update contact" nodes.

- **Details:**  
  Custom fields in KlickTipp correspond to Google Calendar event properties such as summary, location, description, start and end datetime (stored as Unix timestamps), and attendee names.

---

### 3. Summary Table

| Node Name                             | Node Type                     | Functional Role                        | Input Node(s)                         | Output Node(s)                                            | Sticky Note                                                                                                    |
|-------------------------------------|-------------------------------|-------------------------------------|-------------------------------------|----------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Documentation                       | Sticky Note                   | Workflow introduction and instructions | N/A                                 | N/A                                                      | Community Node Disclaimer: This workflow uses KlickTipp community nodes. Introduction, setup, customization info. |
| Watch new Google Calendar events    | Google Calendar Trigger       | Trigger on new calendar events       | N/A                                 | Iterate through attendees (event created)                 |                                                                                                                |
| Watch updated Google Calendar events| Google Calendar Trigger       | Trigger on updated calendar events   | N/A                                 | Iterate through attendees (status changed)                |                                                                                                                |
| Watch cancelled Google Calendar events | Google Calendar Trigger    | Trigger on cancelled calendar events | N/A                                 | Iterate through attendees (event canceled)                |                                                                                                                |
| Iterate through attendees (event created) | Split Out                 | Split attendees array from new events | Watch new Google Calendar events    | Filter email domain (new event)                            |                                                                                                                |
| Iterate through attendees (status changed) | Split Out                 | Split attendees array from updated events | Watch updated Google Calendar events| Filter email domain (updated event)                        |                                                                                                                |
| Iterate through attendees (event canceled) | Split Out                 | Split attendees array from cancelled events | Watch cancelled Google Calendar events | Filter email domain (canceled event)                        |                                                                                                                |
| Filter email domain (new event)     | Filter                       | Filter out emails with specific domains | Iterate through attendees (event created) | Create or update contact for attendee (new event)          | Filter out a specific email domain.                                                                            |
| Filter email domain (updated event) | Filter                       | Filter out emails with specific domains | Iterate through attendees (status changed) | Route by attendees' status                                  | Filter out a specific email domain.                                                                            |
| Filter email domain (canceled event)| Filter                       | Filter out emails with specific domains | Iterate through attendees (event canceled) | Transfer attendees cancellations                            | Filter out a specific email domain.                                                                            |
| Route by attendees' status          | Switch                       | Route attendees based on RSVP status  | Filter email domain (updated event) | Transfer the attendees' declines, acceptances, considerations, Create or update contact (updated event) |                                                                                                                |
| Create or update contact for attendee (new event) | KlickTipp (subscriber) | Create or update KlickTipp contact for new event attendees | Filter email domain (new event)      | N/A                                                      |                                                                                                                |
| Create or update contact for attendee (updated event) | KlickTipp (subscriber) | Create or update KlickTipp contact for updated event attendees | Route by attendees' status           | N/A                                                      |                                                                                                                |
| Transfer attendees cancellations    | KlickTipp (contact-tagging)  | Tag attendees on cancelled events     | Filter email domain (canceled event)| N/A                                                      |                                                                                                                |
| Transfer the attendees' declines    | KlickTipp (contact-tagging)  | Tag attendees who declined events     | Route by attendees' status (declined)| N/A                                                      |                                                                                                                |
| Transfer the attendees' acceptances | KlickTipp (contact-tagging)  | Tag attendees who accepted events     | Route by attendees' status (accepted)| N/A                                                      |                                                                                                                |
| Transfer the attendees' considerations | KlickTipp (contact-tagging) | Tag attendees with tentative status  | Route by attendees' status (considered) | N/A                                                      |                                                                                                                |
| 1. Get event data                   | Sticky Note                   | Logical block marker for event data acquisition | N/A                               | N/A                                                      | ## 1. Get event data.                                                                                           |
| 2. Segregate attendees             | Sticky Note                   | Logical block marker for attendee splitting | N/A                               | N/A                                                      | ## 2. Segregate attendees.                                                                                      |
| 3. Filter attendees                | Sticky Note                   | Logical block marker for attendee filtering | N/A                               | N/A                                                      | ## 3. Filter attendees.                                                                                          |
| 4. Route by status                 | Sticky Note                   | Logical block marker for status routing | N/A                               | N/A                                                      | ## 4. Route by status.                                                                                           |
| 5. Save event data                 | Sticky Note                   | Logical block marker for saving event data | N/A                               | N/A                                                      | ## 5. Save event data.                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**  
   - Configure Google Calendar OAuth2 credentials with Client ID and Client Secret from Google Cloud.  
   - Configure KlickTipp API credentials using username/password with API access enabled.

2. **Add Google Calendar Trigger nodes:**  
   - Add three Google Calendar Trigger nodes:  
     a. "Watch new Google Calendar events"  
        - Trigger on "eventCreated"  
        - Poll every minute  
        - Select the relevant Google Calendar ID  
     b. "Watch updated Google Calendar events"  
        - Trigger on "eventUpdated"  
        - Poll every minute  
        - Same calendar ID  
     c. "Watch cancelled Google Calendar events"  
        - Trigger on "eventCancelled"  
        - Poll every minute  
        - Same calendar ID

3. **Add Split Out nodes to iterate attendees:**  
   - For each trigger, add a Split Out node named accordingly:  
     - "Iterate through attendees (event created)" connected to "Watch new Google Calendar events"  
     - "Iterate through attendees (status changed)" connected to "Watch updated Google Calendar events"  
     - "Iterate through attendees (event canceled)" connected to "Watch cancelled Google Calendar events"  
   - Configure the Split Out node to split by the "attendees" field, including all other fields.

4. **Add Filter nodes to exclude specific email domains:**  
   - For each Split Out node, add a Filter node immediately downstream:  
     - "Filter email domain (new event)" connected to "Iterate through attendees (event created)"  
     - "Filter email domain (updated event)" connected to "Iterate through attendees (status changed)"  
     - "Filter email domain (canceled event)" connected to "Iterate through attendees (event canceled)"  
   - Configure each Filter node with condition: attendee email does NOT contain "@example.com" (case-sensitive).

5. **Add a Switch node for routing by RSVP status:**  
   - Connect "Filter email domain (updated event)" output to "Route by attendees' status" Switch node.  
   - Configure Switch node rules:  
     - Output "declined" if responseStatus equals "declined"  
     - Output "accepted" if responseStatus equals "accepted"  
     - Output "considered" if responseStatus equals "tentative"  
     - Output "event updated" if responseStatus equals "needsAction"

6. **Add KlickTipp nodes for contact subscription and tagging:**  
   - For new event attendees passing the filter:  
     - Add KlickTipp subscriber node "Create or update contact for attendee (new event)" connected after "Filter email domain (new event)". Configure to:  
       - Use attendee email as subscriber email.  
       - Map event fields to custom data fields: summary, location (optional), description (optional), start datetime, end datetime as Unix timestamps, first name and last name parsed from attendee displayName.  
       - Apply tag "Google Calendar | event created/updated" (tagId "13642952").  
   - For updated event attendees:  
     - Connect Switch node outputs:  
       - "declined" → KlickTipp tagging node "Transfer the attendees' declines" with tag "Google Calendar | event declined" (tagId "13642445").  
       - "accepted" → KlickTipp tagging node "Transfer the attendees' acceptances" with tag "Google Calendar | event confirmed" (tagId "13667113").  
       - "considered" → KlickTipp tagging node "Transfer the attendees' considerations" with tag "Google Calendar | event considered" (tagId "13667652").  
       - "event updated" → KlickTipp subscriber node "Create or update contact for attendee (updated event)" configured same as new event contact node.  
   - For cancelled event attendees:  
     - Connect "Filter email domain (canceled event)" output to KlickTipp tagging node "Transfer attendees cancellations" with tag "Google Calendar | event canceled" (tagId "13642439").

7. **Set workflow execution settings:**  
   - Set timezone to your local timezone (e.g., Europe/Berlin).  
   - Set execution order to “v1” (sequential).  
   - Configure polling frequency as needed, recommended every 1-5 minutes for near real-time sync.

8. **Test and activate the workflow:**  
   - Run test events with different statuses and verify KlickTipp contact creation, updates, and tagging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow uses KlickTipp community nodes requiring API access with username/password for authentication.                                                                                                                                                        | KlikcTipp API documentation                                                                               |
| Recommended to prepare KlickTipp custom fields to store Google Calendar event data: "event summary", "event description", "event location", "event start datetime", "event end datetime" configured as single line or datetime types accordingly.                       | Workflow Documentation sticky note                                                                        |
| Prepare KlickTipp tags for event lifecycle states: "event created/updated", "event canceled", "event declined", "event confirmed", "event considered".                                                                                                              | Workflow Documentation sticky note                                                                        |
| Poll frequency can be adjusted for your use case, balancing real-time needs and API quota limits.                                                                                                                                                                    | Workflow Documentation sticky note                                                                        |
| This workflow supports partial deployment by independently enabling triggers for created, updated, or cancelled events.                                                                                                                                             | Workflow Documentation sticky note                                                                        |
| Event attendee names are split into first and last name by splitting on the first space character. This may require adjustment for complex or non-Western names.                                                                                                     | Node "Create or update contact" configuration notes                                                       |
| Filtering out "@example.com" domain attendees avoids syncing internal test accounts; adjust filter condition to your internal domains as needed.                                                                                                                     | Filter nodes notes                                                                                        |
| Event start and end datetimes are converted to Unix timestamps in seconds, considering timezone. This ensures compatibility with KlickTipp datetime fields.                                                                                                        | "Create or update contact" nodes expressions                                                             |
| The workflow is designed to handle edge cases such as missing attendees, API failures, and responseStatus variations, but users should monitor logs for any failures due to permissions or network issues.                                                           | General best practice                                                                                      |

---

**Disclaimer:**  
The provided text is extracted exclusively from an n8n automated workflow designed with adherence to content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.