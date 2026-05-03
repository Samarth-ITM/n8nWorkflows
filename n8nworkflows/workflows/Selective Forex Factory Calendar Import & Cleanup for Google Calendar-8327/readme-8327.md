Selective Forex Factory Calendar Import & Cleanup for Google Calendar

https://n8nworkflows.xyz/workflows/selective-forex-factory-calendar-import---cleanup-for-google-calendar-8327


# Selective Forex Factory Calendar Import & Cleanup for Google Calendar

### 1. Workflow Overview

This workflow automates the selective importation and cleanup of Forex Factory calendar events into a Google Calendar. It targets users who want to maintain an up-to-date calendar of Forex Factory economic events filtered by impact level (High, Medium, Low) and ensure obsolete or duplicate events from ForexFactory.com are deleted. The workflow is designed to run weekly on Sundays at 6 PM and consists of the following logical blocks:

- **1.1 Scheduled Trigger & Data Acquisition:** Initiates the workflow weekly and fetches the current week's Forex Factory calendar data in ICS format.
- **1.2 Data Extraction & Event Splitting:** Converts the ICS calendar data into JSON events and splits them into individual event items for processing.
- **1.3 Event Impact Classification & Google Calendar Integration:** Routes events based on their impact level (High, Medium, Low) and inserts them into corresponding Google Calendar entries or discards them if low impact.
- **1.4 Cleanup of Old ForexFactory.com Events:** Retrieves existing Google Calendar events from the past 10 days, identifies those originating from ForexFactory.com, and deletes them to prevent duplicates or outdated entries.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Acquisition

- **Overview:** This block triggers the workflow weekly on Sundays at 6 PM and fetches the Forex Factory ICS calendar file for the current week.
- **Nodes Involved:** `Sunday 6 PM`, `HTTP Request`, `Get All Event 10 Days Before`
  
- **Node Details:**

  - **Sunday 6 PM**
    - Type: Schedule Trigger  
    - Role: Starts the workflow every Sunday at 18:00 (6 PM).
    - Configuration: Interval set to weekly on Sundays, trigger hour 18.
    - Inputs: None (trigger node).
    - Outputs: Triggers `HTTP Request` and `Get All Event 10 Days Before`.
    - Edge Cases: Workflow will not run if the n8n instance is down at scheduled time.

  - **HTTP Request**
    - Type: HTTP Request  
    - Role: Downloads the Forex Factory calendar ICS file for the current week.
    - Configuration: URL set to `https://nfs.faireconomy.media/ff_calendar_thisweek.ics`.
    - Inputs: Trigger from `Sunday 6 PM`.
    - Outputs: Sends downloaded ICS data to `Extract from File`.
    - Edge Cases: Possible network errors, HTTP timeouts, or 404 if the ICS URL changes.

  - **Get All Event 10 Days Before**
    - Type: Google Calendar (Get All)  
    - Role: Retrieves all Google Calendar events from the last 10 days.
    - Configuration: Time range from now minus 10 days to now; returns all events.
    - Inputs: Trigger from `Sunday 6 PM`.
    - Outputs: Sends data to `If ForexFactory.com event`.
    - Credentials: Requires valid Google Calendar OAuth2 credentials.
    - Edge Cases: API rate limits, authentication failures, or empty calendar returns.

#### 2.2 Data Extraction & Event Splitting

- **Overview:** Extracts events from the ICS file into a JSON structure and splits the events array into separate event items for processing.
- **Nodes Involved:** `Extract from File`, `Split Out`

- **Node Details:**

  - **Extract from File**
    - Type: Extract From File  
    - Role: Converts ICS format data to JSON event objects.
    - Configuration: Operation set to 'fromIcs' to parse ICS files.
    - Inputs: ICS file content from `HTTP Request`.
    - Outputs: JSON object with `data.events` array to `Split Out`.
    - Edge Cases: Malformed ICS files may cause parsing errors.

  - **Split Out**
    - Type: Split Out  
    - Role: Splits the `data.events` array into individual event JSON items.
    - Configuration: Field to split out is `data.events`; includes all other fields.
    - Inputs: JSON with `data.events` array from `Extract from File`.
    - Outputs: Sends individual event JSON objects to `Switch`.
    - Edge Cases: Empty or undefined `data.events` field will result in no output.

#### 2.3 Event Impact Classification & Google Calendar Integration

- **Overview:** Classifies each event by impact (High, Medium, Low) based on the `description` field and inserts High and Medium impact events into Google Calendar with appropriate reminders. Low impact events are ignored.
- **Nodes Involved:** `Switch`, `High Impact`, `Medium Impact`, `No Operation, do nothing`

- **Node Details:**

  - **Switch**
    - Type: Switch  
    - Role: Routes events to different paths based on impact keyword in `description`.
    - Configuration: Three outputs:
      - "HIgh" if `description` contains "High"
      - "Medium" if `description` contains "Medium"
      - "Low" if `description` contains "Low"
    - Inputs: Individual event JSON from `Split Out`.
    - Outputs:
      - High impact events to `High Impact`
      - Medium impact events to `Medium Impact`
      - Low impact events to `No Operation, do nothing`
    - Edge Cases: If `description` is missing or does not contain any keyword, event is ignored as no matching case.

  - **High Impact**
    - Type: Google Calendar (Create Event)  
    - Role: Creates Google Calendar events for high impact events.
    - Configuration:
      - Uses event start and end dates from `$json['data.events'].start.date` and `.end.date`.
      - Sets event summary and description from JSON.
      - Adds a popup reminder 30 minutes before event.
      - Does not use default reminders.
    - Inputs: High impact events from `Switch`.
    - Outputs: None (terminal node).
    - Credentials: Requires Google Calendar OAuth2.
    - Edge Cases: Google API authentication errors, invalid date formats, or rate limits.

  - **Medium Impact**
    - Type: Google Calendar (Create Event)  
    - Role: Creates Google Calendar events for medium impact events.
    - Configuration:
      - Similar to High Impact but no reminders set.
      - Uses event start/end dates and summary/description.
    - Inputs: Medium impact events from `Switch`.
    - Outputs: None (terminal node).
    - Credentials: Requires Google Calendar OAuth2.
    - Edge Cases: Same as High Impact.

  - **No Operation, do nothing**
    - Type: No Operation  
    - Role: Drops low impact events without action.
    - Inputs: Low impact events from `Switch`.
    - Outputs: None.
    - Edge Cases: None.

#### 2.4 Cleanup of Old ForexFactory.com Events

- **Overview:** Inspects Google Calendar events from the last 10 days and deletes those identified as originating from ForexFactory.com, preventing duplicate or outdated entries.
- **Nodes Involved:** `If ForexFactory.com event`, `Delete an event`, `No Operation, do nothing`

- **Node Details:**

  - **If ForexFactory.com event**
    - Type: If  
    - Role: Filters events whose description contains "forexfactory.com".
    - Configuration:
      - Condition checks if `description` field contains the string "forexfactory.com".
      - Outputs:
        - True: to `Delete an event`
        - False: to `No Operation, do nothing`
    - Inputs: Events from `Get All Event 10 Days Before`.
    - Outputs: Conditional branch based on description content.
    - Edge Cases: Events without a description field might be misclassified.

  - **Delete an event**
    - Type: Google Calendar (Delete Event)  
    - Role: Deletes the identified ForexFactory.com events from Google Calendar.
    - Configuration:
      - Uses event ID from `$json.id`.
      - Deletes event from specified calendar.
    - Inputs: True branch from `If ForexFactory.com event`.
    - Outputs: None.
    - Credentials: Requires Google Calendar OAuth2.
    - Edge Cases: Deletion may fail if event was already deleted, or due to API errors.

  - **No Operation, do nothing**
    - Type: No Operation  
    - Role: Does nothing for events not matching the ForexFactory.com condition.
    - Inputs: False branch from `If ForexFactory.com event`.
    - Outputs: None.
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                                  | Input Node(s)              | Output Node(s)                 | Sticky Note                                   |
|----------------------------|----------------------------|-------------------------------------------------|----------------------------|-------------------------------|-----------------------------------------------|
| Sunday 6 PM                | Schedule Trigger           | Starts workflow weekly on Sundays at 6 PM       | None                       | HTTP Request, Get All Event 10 Days Before |                                               |
| HTTP Request               | HTTP Request              | Downloads Forex Factory ICS calendar              | Sunday 6 PM                | Extract from File              |                                               |
| Extract from File          | Extract From File          | Parses ICS file into JSON events                   | HTTP Request               | Split Out                     |                                               |
| Split Out                 | Split Out                 | Splits events array into individual events       | Extract from File           | Switch                       |                                               |
| Switch                    | Switch                    | Routes events by impact level                      | Split Out                  | High Impact, Medium Impact, No Operation, do nothing |                                               |
| High Impact               | Google Calendar            | Creates calendar entries for high impact events  | Switch (High)              | None                         |                                               |
| Medium Impact             | Google Calendar            | Creates calendar entries for medium impact events| Switch (Medium)            | None                         |                                               |
| No Operation, do nothing  | No Operation               | Drops low impact events                            | Switch (Low)               | None                         |                                               |
| Get All Event 10 Days Before | Google Calendar           | Retrieves last 10 days of events                   | Sunday 6 PM                | If ForexFactory.com event     |                                               |
| If ForexFactory.com event | If                         | Filters events based on description                | Get All Event 10 Days Before| Delete an event, No Operation, do nothing |                                               |
| Delete an event           | Google Calendar            | Deletes ForexFactory.com events                     | If ForexFactory.com event  | None                         |                                               |
| No Operation, do nothing  | No Operation               | Does nothing for non-matching events               | If ForexFactory.com event  | None                         |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Sunday 6 PM`  
   - Type: Schedule Trigger  
   - Set schedule to weekly, trigger on Sundays, at hour 18 (6 PM).

2. **Add an HTTP Request node**  
   - Name: `HTTP Request`  
   - Type: HTTP Request  
   - Set URL to `https://nfs.faireconomy.media/ff_calendar_thisweek.ics`.  
   - Connect output of `Sunday 6 PM` to this node.

3. **Add an Extract From File node**  
   - Name: `Extract from File`  
   - Type: Extract From File  
   - Set operation to `fromIcs` to parse ICS format.  
   - Connect output of `HTTP Request` to this node.

4. **Add a Split Out node**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Set fieldToSplitOut to `data.events`.  
   - Configure to include all other fields.  
   - Connect output of `Extract from File` to this node.

5. **Add a Switch node**  
   - Name: `Switch`  
   - Type: Switch  
   - Create three outputs with conditions on the field `$json['data.events'].description`:
     - Output "HIgh": contains string "High" (case sensitive)
     - Output "Medium": contains string "Medium"
     - Output "Low": contains string "Low"
   - Connect output of `Split Out` to this node.

6. **Add a Google Calendar node for High Impact**  
   - Name: `High Impact`  
   - Type: Google Calendar (Create)  
   - Set calendar to desired target calendar (select from list).  
   - Set start date to `={{ $json['data.events'].start.date }}`  
   - Set end date to `={{ $json['data.events'].end.date }}`  
   - Set summary to `={{ $json['data.events'].summary }}`  
   - Set description to `={{ $json['data.events'].description }}`  
   - Disable useDefaultReminders.  
   - Add a popup reminder 30 minutes before event.  
   - Connect output "HIgh" of `Switch` to this node.  
   - Assign appropriate Google Calendar OAuth2 credentials.

7. **Add a Google Calendar node for Medium Impact**  
   - Name: `Medium Impact`  
   - Type: Google Calendar (Create)  
   - Configure similarly to `High Impact` but do not add reminders.  
   - Connect output "Medium" of `Switch` to this node.  
   - Assign Google Calendar OAuth2 credentials.

8. **Add a No Operation node for Low impact events**  
   - Name: `No Operation, do nothing`  
   - Type: No Operation  
   - Connect output "Low" of `Switch` to this node.

9. **Add a Google Calendar node to get past events**  
   - Name: `Get All Event 10 Days Before`  
   - Type: Google Calendar (Get All)  
   - Set calendar to the same as above.  
   - Set timeMin to `={{ $now.minus(10, 'days') }}`  
   - Set timeMax to `={{ $now }}`  
   - Set operation to get all events in range.  
   - Connect output of `Sunday 6 PM` to this node.  
   - Assign Google Calendar OAuth2 credentials.

10. **Add an If node to filter ForexFactory.com events**  
    - Name: `If ForexFactory.com event`  
    - Type: If  
    - Add condition: `$json.description` contains "forexfactory.com".  
    - Connect output of `Get All Event 10 Days Before` to this node.

11. **Add a Google Calendar node to delete events**  
    - Name: `Delete an event`  
    - Type: Google Calendar (Delete)  
    - Set calendar to the same calendar.  
    - Set operation to `delete`.  
    - Set eventId to `={{ $json.id }}`.  
    - Connect True output of `If ForexFactory.com event` to this node.  
    - Assign Google Calendar OAuth2 credentials.

12. **Add a No Operation node for non-ForexFactory events**  
    - Name: `No Operation, do nothing`  
    - Type: No Operation  
    - Connect False output of `If ForexFactory.com event` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                               |
|----------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow requires Google Calendar OAuth2 credentials configured for all Google nodes.    | n8n Credential Setup Documentation                             |
| The ICS calendar URL `https://nfs.faireconomy.media/ff_calendar_thisweek.ics` is used for weekly Forex Factory data. | Forex Factory Calendar ICS URL                                |
| The workflow runs on Sundays at 6 PM to update the calendar weekly with recent Forex events. | Scheduling best practice for weekly updates                   |
| Event classification is done by matching keywords "High", "Medium", and "Low" in event descriptions, which assumes consistent formatting from Forex Factory. | May need adjustment if Forex Factory changes event descriptions |
| Deletion logic ensures removal of events containing "forexfactory.com" in their description, avoiding duplicates or stale entries. | Important for calendar cleanup and data accuracy              |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.