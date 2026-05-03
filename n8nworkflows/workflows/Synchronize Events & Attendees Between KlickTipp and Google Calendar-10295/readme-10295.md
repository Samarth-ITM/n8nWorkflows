Synchronize Events & Attendees Between KlickTipp and Google Calendar

https://n8nworkflows.xyz/workflows/synchronize-events---attendees-between-klicktipp-and-google-calendar-10295


# Synchronize Events & Attendees Between KlickTipp and Google Calendar

### 1. Workflow Overview

This workflow establishes a comprehensive two-way synchronization between KlickTipp (an email marketing and CRM platform) and Google Calendar. Its primary purpose is to keep event information and attendee participation statuses consistent between both platforms automatically. When contacts in KlickTipp are tagged for sending event invitations, corresponding Google Calendar events are created. Conversely, changes in Google Calendar events (such as creation, updates, cancellations, or RSVP status changes) trigger updates in KlickTipp contacts and tags to reflect the latest event and attendee information.

**Target Use Cases:**  
- Marketers, coaches, and event organizers who need automated event invitations and instant capture of attendee responses without manual data entry.  
- Maintaining up-to-date contact information and event statuses across KlickTipp and Google Calendar to optimize campaign targeting and event management.

**Logical Blocks:**

- **1.1 Event Data Reception:** Nodes that watch Google Calendar for events created, updated, or canceled, and KlickTipp for tagging to send invitations.
- **1.2 Attendee Segregation & Filtering:** Splitting and filtering incoming attendee lists based on email domains to exclude internal or unwanted addresses.
- **1.3 Routing by Attendee Status:** Routing attendees according to their RSVP status (accepted, declined, tentative, needsAction).
- **1.4 Data Transfer to KlickTipp:** Creating or updating KlickTipp contacts and tagging them based on event status and attendee responses.
- **1.5 Data Transfer to Google Calendar:** Creating Google Calendar events triggered by KlickTipp tagging.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Data Reception

**Overview:**  
This block monitors Google Calendar for new, updated, or canceled events and listens for KlickTipp tagging events to trigger Google Calendar event creation.

**Nodes Involved:**  
- Watch new Google Calendar events  
- Watch updated Google Calendar events  
- Watch cancelled Google Calendar events  
- Watch Tagging in KlickTipp

**Node Details:**

- **Watch new Google Calendar events**  
  - *Type:* Google Calendar Trigger  
  - *Role:* Triggers workflow when a new event is created on a specified Google Calendar.  
  - *Configuration:* Polls every minute on a designated calendar; triggers on "eventCreated".  
  - *Connections:* Outputs to "Iterate through attendees (event created)".  
  - *Failure Cases:* Google API auth errors, calendar permissions, polling delays.

- **Watch updated Google Calendar events**  
  - *Type:* Google Calendar Trigger  
  - *Role:* Triggers on event updates on the same Google Calendar.  
  - *Configuration:* Polls every minute; triggers on "eventUpdated".  
  - *Connections:* Outputs to "Iterate through attendees (status changed)".  
  - *Failure Cases:* Same as above, plus potential inconsistent update data.

- **Watch cancelled Google Calendar events**  
  - *Type:* Google Calendar Trigger  
  - *Role:* Triggers on event cancellations.  
  - *Configuration:* Polls every minute; triggers on "eventCancelled".  
  - *Connections:* Outputs to "Iterate through attendees (event canceled)".  
  - *Failure Cases:* Same as above.

- **Watch Tagging in KlickTipp**  
  - *Type:* KlickTipp Trigger  
  - *Role:* Triggers when a contact is tagged in KlickTipp with the activation tag to send an event invitation via Google Calendar.  
  - *Configuration:* Uses webhook with configured credentials.  
  - *Connections:* Outputs to "Create a Google Calendar Event".  
  - *Failure Cases:* KlickTipp API auth failure, webhook delivery issues.

---

#### 2.2 Attendee Segregation & Filtering

**Overview:**  
This block splits the attendees array from Google Calendar events into individual attendees and filters out any attendees whose email domains are on an exclusion list (e.g., internal test domains).

**Nodes Involved:**  
- Iterate through attendees (event created)  
- Iterate through attendees (status changed)  
- Iterate through attendees (event canceled)  
- Filter email domain (new event)  
- Filter email domain (updated event)  
- Filter email domain (canceled event)

**Node Details:**

- **Iterate through attendees (event created)**  
  - *Type:* SplitOut  
  - *Role:* Splits the array of attendees from a new event into individual items for processing.  
  - *Field Split:* "attendees"  
  - *Connections:* Outputs to "Filter email domain (new event)".  
  - *Edge cases:* Empty attendee lists.

- **Iterate through attendees (status changed)**  
  - *Type:* SplitOut  
  - *Role:* Same as above but for updated events.  
  - *Connections:* Outputs to "Filter email domain (updated event)".

- **Iterate through attendees (event canceled)**  
  - *Type:* SplitOut  
  - *Role:* Same as above but for canceled events.  
  - *Connections:* Outputs to "Filter email domain (canceled event)".

- **Filter email domain (new event)**  
  - *Type:* Filter  
  - *Role:* Excludes attendees whose email contains "@example.com" (example domain).  
  - *Condition:* Email NOT containing "@example.com".  
  - *Connections:* Passes filtered data to "Create or update contact for attendee (new event)".  
  - *Notes:* Protects against test/internal emails.

- **Filter email domain (updated event)**  
  - *Type:* Filter  
  - *Role:* Same as above but for updated events.  
  - *Connections:* Outputs to "Route by attendees' status".

- **Filter email domain (canceled event)**  
  - *Type:* Filter  
  - *Role:* Same as above but for canceled events.  
  - *Connections:* Outputs to "Transfer attendees cancellations".

---

#### 2.3 Routing by Attendee Status

**Overview:**  
This block routes attendees based on their RSVP status from Google Calendar, enabling different processing paths for declined, accepted, tentative, or no response.

**Nodes Involved:**  
- Route by attendees' status  
- Transfer the attendees' declines  
- Transfer the attendees' acceptances  
- Transfer the attendees' considerations  
- Create or update contact for attendee (updated event)

**Node Details:**

- **Route by attendees' status**  
  - *Type:* Switch  
  - *Role:* Routes data based on the attendee's `responseStatus` field.  
  - *Cases:*  
    - "declined" → Transfer the attendees' declines  
    - "accepted" → Transfer the attendees' acceptances  
    - "tentative" → Transfer the attendees' considerations  
    - "needsAction" → Create or update contact for attendee (updated event)  
  - *Connections:* Outputs to respective transfer/update nodes.  
  - *Edge cases:* Unknown or missing responseStatus values.

- **Transfer the attendees' declines**  
  - *Type:* KlickTipp  
  - *Role:* Tags contact in KlickTipp with "event declined" tag.  
  - *Parameters:* Uses attendee email; tagId corresponds to decline tag.  
  - *Connections:* Terminal node for declined attendees.  
  - *Failure Cases:* KlickTipp API errors, invalid emails.

- **Transfer the attendees' acceptances**  
  - *Type:* KlickTipp  
  - *Role:* Tags contact with "event confirmed" tag.  
  - *Parameters:* Uses attendee email; tagId corresponds to acceptance tag.

- **Transfer the attendees' considerations**  
  - *Type:* KlickTipp  
  - *Role:* Tags contact with "event considered" tag for tentative attendees.  
  - *Parameters:* Uses attendee email; tagId corresponds to tentative tag.

- **Create or update contact for attendee (updated event)**  
  - *Type:* KlickTipp  
  - *Role:* Adds/updates contact in KlickTipp with event details and applies update tag.  
  - *Fields:* Maps calendar event summary, location, description, and timestamps to KlickTipp custom fields. Splits displayName into first and last names.  
  - *Tag:* "event created/updated".  
  - *Connections:* Terminal node for attendees with "needsAction" or after routing.

---

#### 2.4 Data Transfer to KlickTipp (Event Cancellations)

**Overview:**  
Handles attendees of canceled Google Calendar events by tagging them with "event canceled" in KlickTipp.

**Nodes Involved:**  
- Iterate through attendees (event canceled)  
- Filter email domain (canceled event)  
- Transfer attendees cancellations

**Node Details:**

- **Transfer attendees cancellations**  
  - *Type:* KlickTipp  
  - *Role:* Tags contacts with "event canceled" tag.  
  - *Parameters:* Uses attendee email; tagId corresponds to event canceled tag.

---

#### 2.5 Data Transfer to KlickTipp (New Events)

**Overview:**  
Processes attendees of newly created Google Calendar events by creating or updating their contacts in KlickTipp and tagging them as event created/updated.

**Nodes Involved:**  
- Iterate through attendees (event created)  
- Filter email domain (new event)  
- Create or update contact for attendee (new event)

**Node Details:**

- **Create or update contact for attendee (new event)**  
  - *Type:* KlickTipp  
  - *Role:* Creates or updates KlickTipp contacts with event details and applies the event created/updated tag.  
  - *Fields:* Similar mapping as updated event node.  
  - *Connections:* Terminal node.

---

#### 2.6 Data Transfer to Google Calendar (From KlickTipp)

**Overview:**  
Creates a Google Calendar event when a contact in KlickTipp is tagged with the activation tag to send an event invitation.

**Nodes Involved:**  
- Watch Tagging in KlickTipp  
- Create a Google Calendar Event

**Node Details:**

- **Create a Google Calendar Event**  
  - *Type:* Google Calendar  
  - *Role:* Creates an event in Google Calendar using KlickTipp custom fields.  
  - *Parameters:*  
    - Start and end times parsed from KlickTipp date strings with timezone (Europe/Berlin).  
    - Summary, description, location mapped from KlickTipp custom fields.  
    - Attendees list includes the contact email.  
    - Calendar selected is the same as watched calendars.  
  - *Connections:* Terminal node.  
  - *Failure Cases:* Date parsing errors, Google API auth issues, invalid attendee emails.

---

### 3. Summary Table

| Node Name                              | Node Type                     | Functional Role                                 | Input Node(s)                         | Output Node(s)                          | Sticky Note                                                                                           |
|--------------------------------------|-------------------------------|------------------------------------------------|-------------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| 1. Get event data.                    | Sticky Note                   | Block label for event data reception            |                                     |                                       | ## 1. Get event data.                                                                              |
| 2. Segregate attendees.               | Sticky Note                   | Block label for attendee splitting              |                                     |                                       | ## 2. Segregate attendees.                                                                         |
| 3. Filter attendees.                  | Sticky Note                   | Block label for attendee filtering               |                                     |                                       | ## 3. Filter attendees.                                                                            |
| 4. Route by status.                   | Sticky Note                   | Block label for routing attendees by status     |                                     |                                       | ## 4. Route by status.                                                                             |
| 5. Transfer event data.               | Sticky Note                   | Block label for transferring event data         |                                     |                                       | ## 5. Transfer event data.                                                                         |
| Documentation                        | Sticky Note                   | Workflow introduction and setup instructions     |                                     |                                       | Community Node Disclaimer and detailed introduction including setup and customization instructions |
| Watch new Google Calendar events      | Google Calendar Trigger       | Triggers on new events                            |                                     | Iterate through attendees (event created) |                                                                                                    |
| Iterate through attendees (event created) | SplitOut                    | Splits new event attendees                        | Watch new Google Calendar events    | Filter email domain (new event)         |                                                                                                    |
| Filter email domain (new event)       | Filter                       | Filters out specified email domains               | Iterate through attendees (event created) | Create or update contact for attendee (new event) | Filter out a specific email domain.                                                               |
| Create or update contact for attendee (new event) | KlickTipp                  | Creates/updates KlickTipp contacts on new event | Filter email domain (new event)     |                                       |                                                                                                    |
| Watch updated Google Calendar events  | Google Calendar Trigger       | Triggers on event updates                         |                                     | Iterate through attendees (status changed) |                                                                                                    |
| Iterate through attendees (status changed) | SplitOut                    | Splits updated event attendees                    | Watch updated Google Calendar events | Filter email domain (updated event)     |                                                                                                    |
| Filter email domain (updated event)   | Filter                       | Filters out specified email domains               | Iterate through attendees (status changed) | Route by attendees' status              | Filter out a specific email domain.                                                               |
| Route by attendees' status            | Switch                       | Routes attendees by RSVP status                   | Filter email domain (updated event) | Transfer the attendees' declines, Transfer the attendees' acceptances, Transfer the attendees' considerations, Create or update contact for attendee (updated event) |                                                                                                    |
| Transfer the attendees' declines      | KlickTipp                    | Tags declined attendees                           | Route by attendees' status          |                                       |                                                                                                    |
| Transfer the attendees' acceptances   | KlickTipp                    | Tags accepted attendees                           | Route by attendees' status          |                                       |                                                                                                    |
| Transfer the attendees' considerations| KlickTipp                    | Tags tentative attendees                          | Route by attendees' status          |                                       |                                                                                                    |
| Create or update contact for attendee (updated event) | KlickTipp                  | Creates/updates KlickTipp contacts on updates    | Route by attendees' status          |                                       |                                                                                                    |
| Watch cancelled Google Calendar events| Google Calendar Trigger       | Triggers on event cancellations                   |                                     | Iterate through attendees (event canceled) |                                                                                                    |
| Iterate through attendees (event canceled) | SplitOut                    | Splits canceled event attendees                   | Watch cancelled Google Calendar events | Filter email domain (canceled event)     |                                                                                                    |
| Filter email domain (canceled event)  | Filter                       | Filters out specified email domains               | Iterate through attendees (event canceled) | Transfer attendees cancellations        | Filter out a specific email domain.                                                               |
| Transfer attendees cancellations      | KlickTipp                    | Tags attendees of canceled events                 | Filter email domain (canceled event) |                                       |                                                                                                    |
| Watch Tagging in KlickTipp             | KlickTipp Trigger            | Triggers on KlickTipp tagging for event invites  |                                     | Create a Google Calendar Event           |                                                                                                    |
| Create a Google Calendar Event         | Google Calendar              | Creates Google Calendar event from KlickTipp data| Watch Tagging in KlickTipp          |                                       |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger Nodes:**

   - *Watch new Google Calendar events*  
     - Type: Google Calendar Trigger  
     - Poll every 1 minute  
     - Trigger on eventCreated  
     - Select the target Google Calendar (via OAuth2 credentials)  

   - *Watch updated Google Calendar events*  
     - Same setup as above but trigger on eventUpdated  

   - *Watch cancelled Google Calendar events*  
     - Same setup but trigger on eventCancelled  

2. **Add KlickTipp Trigger Node:**

   - *Watch Tagging in KlickTipp*  
     - Type: KlickTipp Trigger  
     - Configure webhook URL in KlickTipp to trigger on tag "Send an event invitation via Google Calendar"  
     - Use KlickTipp API credentials  

3. **Add SplitOut Nodes to Separate Attendees:**

   - For each Google Calendar trigger, add a SplitOut node splitting on the `attendees` field.  
   - Name accordingly: e.g., "Iterate through attendees (event created)".  

4. **Add Filter Nodes to Exclude Internal/Test Emails:**

   - Create Filter nodes after each SplitOut node.  
   - Condition: Exclude emails containing "@example.com" (replace with actual domains to exclude).  
   - Name accordingly for each event type (new, updated, canceled).  

5. **Add Switch Node to Route by Attendee Status:**

   - After filtering updated event attendees, add a Switch node.  
   - Condition on `$json.attendees.responseStatus` with cases: "declined", "accepted", "tentative", "needsAction".  
   - Route each case to corresponding KlickTipp tag nodes or update contact.  

6. **Add KlickTipp Nodes to Tag or Update Contacts:**

   - For new events: Add KlickTipp node to create/update contact with event details and tag "event created/updated".  
   - For updated events, create KlickTipp nodes for each RSVP status to tag accordingly:  
     - Declined → tag "event declined"  
     - Accepted → tag "event confirmed"  
     - Tentative → tag "event considered"  
     - NeedsAction → update contact with event details & tag  
   - For canceled events: KlickTipp node to tag "event canceled".  
   - Use KlickTipp API credentials for all.  
   - Map required custom fields: event summary, description, location, start/end timestamps, first and last names from displayName.  

7. **Add Google Calendar Node to Create Events from KlickTipp Data:**

   - Connect from KlickTipp Trigger node.  
   - Configure to create event in the same Google Calendar.  
   - Map start and end date/time from KlickTipp custom fields, parsing the datetime strings with appropriate timezone (e.g., Europe/Berlin).  
   - Map summary, location, description, and attendees email from KlickTipp data.  
   - Ensure OAuth2 credentials for Google Calendar are set.  

8. **Add Sticky Notes at Logical Blocks:**

   - Add sticky notes to label blocks:  
     - "1. Get event data." around triggers  
     - "2. Segregate attendees." near SplitOut nodes  
     - "3. Filter attendees." near Filter nodes  
     - "4. Route by status." near Switch node  
     - "5. Transfer event data." near KlickTipp and Google Calendar nodes  
   - Add a detailed Documentation sticky note with introduction and setup instructions for user reference.  

9. **Configure Credentials:**

   - Google Calendar OAuth2 credentials with Client ID and Secret from Google Cloud Console.  
   - KlickTipp API credentials with username and password having API access enabled.  

10. **Set Polling Frequency:**

    - Recommended 1-5 minutes for near real-time sync.  
    - Adjust as needed based on API limits and usage.  

11. **Test Workflow:**

    - Validate Google Calendar triggers fire correctly.  
    - Confirm KlickTipp tagging triggers event creation.  
    - Check attendee routing and tagging in KlickTipp matches RSVP status.  
    - Verify date/time parsing correctness and timezone handling.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow uses KlickTipp community nodes.                                                                                                                                                                                  | Community Node Disclaimer                                                                                |
| Prepare KlickTipp custom fields and tags as described for proper mapping and tagging.                                                                                                                                          | Setup Instructions in Documentation sticky note                                                        |
| Recommended polling frequency: every 1–5 minutes for near real-time updates.                                                                                                                                                    | Customization section of Documentation sticky note                                                     |
| Ensure event End datetime is after Start datetime; if fixed duration is desired, compute End from Start plus duration in a Date & Time node.                                                                                   | Customization section of Documentation sticky note                                                     |
| To add Google Meet links, enable “Add Google Meet video conferencing” in the Google Calendar node instead of pasting a URL in the location field.                                                                             | Customization section of Documentation sticky note                                                     |
| For detailed KlickTipp API setup, see KlickTipp developer docs to create API credentials and enable webhook triggers.                                                                                                          | External resource (KlickTipp API documentation)                                                        |
| Google Calendar API requires OAuth2 credentials with Calendar scope enabled. Setup via Google Cloud Console.                                                                                                                   | Google Cloud Console documentation                                                                      |

---

This documentation fully describes the "KlickTipp + Google Calendar: Two-Way Event & Attendee Sync" workflow, enabling reproduction, customization, and troubleshooting.