When specific event created in Google Calendar, duplicate & rename Google File

https://n8nworkflows.xyz/workflows/when-specific-event-created-in-google-calendar--duplicate---rename-google-file-2145


# When specific event created in Google Calendar, duplicate & rename Google File

### 1. Workflow Overview

This workflow automates the creation of a personalized Google Document template when a specific event is created in Google Calendar. It targets professionals who frequently take notes during calls or interviews—such as recruiters, HR teams, sales/customer success teams, and product or UX researchers—by pre-generating a structured document tailored to each calendar event.

The logical flow is organized into these main blocks:

- **1.1 Event Trigger:** Detects new Google Calendar events matching a specific term.
- **1.2 Event Filtering:** Filters events based on creator email and event source URL to focus on targeted meeting types.
- **1.3 Folder Search:** Locates the Google Drive folder where the template document resides.
- **1.4 Template File Search:** Finds the specific Google Document template to duplicate.
- **1.5 File Duplication and Renaming:** Copies the template and renames it dynamically using event details.

This modular structure ensures that only relevant calendar events trigger document creation, and that the duplicated document is organized and named according to event-specific variables.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Trigger

- **Overview:**  
  Listens continuously for new events created in a specific Google Calendar, filtering them by a keyword in the event title.

- **Nodes Involved:**  
  - "New event in Google Calendar"

- **Node Details:**

  - **Node Name:** New event in Google Calendar  
  - **Type:** Google Calendar Trigger  
  - **Technical Role:** Initiates the workflow upon creation of matching calendar events.  
  - **Configuration:**  
    - Triggers on event creation only (`triggerOn: eventCreated`).  
    - Polls every minute for new events to ensure near real-time response.  
    - Filters events where the title contains the phrase "Interview with" (case-sensitive match).  
    - Watches a specific calendar identified by the email "candice@n8n.io".  
  - **Input/Output:**  
    - No input (trigger node).  
    - Outputs event data as JSON to next node.  
  - **Version-specific:** Version 1 of Google Calendar Trigger node is used.  
  - **Potential Failures:**  
    - Authentication errors if Google OAuth2 credentials expire.  
    - API rate limits or quota issues from Google Calendar API.  
    - Missed events if polling interval is too long or API downtime.  
  - **Sticky Notes:**  
    - Advises possibility to replace Google Calendar trigger with other scheduling tools like Calendly or Hubspot.

---

#### 2.2 Event Filtering

- **Overview:**  
  Filters incoming calendar events to select only those created by a specific user and originating from a specific scheduling tool URL.

- **Nodes Involved:**  
  - "Filter specific event"

- **Node Details:**

  - **Node Name:** Filter specific event  
  - **Type:** Filter  
  - **Technical Role:** Conditional gate to allow only targeted events through.  
  - **Configuration:**  
    - Condition 1: Event creator email must equal "candice@n8n.io".  
    - Condition 2: Event source URL must contain "https://n8n.workable.com/events/".  
    - Both conditions combined with logical AND.  
  - **Expressions Used:**  
    - `{{$json.creator.email}}` for creator email.  
    - `{{$json.source.url}}` for event source URL.  
  - **Input/Output:**  
    - Input from "New event in Google Calendar".  
    - Outputs event JSON only if conditions are met, otherwise stops the flow.  
  - **Version-specific:** Uses typeVersion 2 of Filter node for advanced expressions.  
  - **Potential Failures:**  
    - Key errors if event JSON structure changes or fields missing.  
    - Logic errors if conditions do not match intended events, causing no output.  
  - **Sticky Notes:**  
    - Explains that filtering can be customized for any pattern (title, organizer, attendees).

---

#### 2.3 Folder Search

- **Overview:**  
  Searches Google Drive for a folder matching the keyword "SCREENING" where the template document is stored.

- **Nodes Involved:**  
  - "Search folder"

- **Node Details:**

  - **Node Name:** Search folder  
  - **Type:** Google Drive node (Search)  
  - **Technical Role:** Retrieves folder metadata to use for subsequent file search and copy.  
  - **Configuration:**  
    - Search query: `"SCREENING"` (case-insensitive).  
    - Resource type: fileFolder (to include folders).  
    - Limit results to 1 folder.  
  - **Input/Output:**  
    - Input from "Filter specific event".  
    - Outputs folder metadata (id, name, etc.).  
  - **Version-specific:** Uses Google Drive node version 3.  
  - **Potential Failures:**  
    - Folder not found if name or location changes.  
    - Permission errors if OAuth2 credentials lack sufficient Drive scopes.  
    - API rate limits.  
  - **Sticky Notes:**  
    - Notes to search exactly for the folder where the file to duplicate is located.

---

#### 2.4 Template File Search

- **Overview:**  
  Searches within Google Drive for the designated template document named "Template | M1 | Senior AE | ".

- **Nodes Involved:**  
  - "Search file to duplicate"

- **Node Details:**

  - **Node Name:** Search file to duplicate  
  - **Type:** Google Drive node (Search)  
  - **Technical Role:** Finds the Google Doc template file that will be duplicated.  
  - **Configuration:**  
    - Search query: `"Template | M1 | Senior AE | "` (exact phrase, case-insensitive).  
    - Resource: fileFolder (includes files and folders).  
    - Limit to 1 search result.  
  - **Input/Output:**  
    - Input from "Search folder" node.  
    - Outputs file metadata including file ID necessary for duplication.  
  - **Version-specific:** Google Drive node version 3.  
  - **Potential Failures:**  
    - Template file missing or renamed, causing no results.  
    - Permission issues with Google Drive access.  
    - API issues or rate limits.  
  - **Sticky Notes:**  
    - Reminder to search explicitly for the template file to duplicate.

---

#### 2.5 File Duplication and Renaming

- **Overview:**  
  Copies the template document and renames the copy dynamically using the calendar event title or other event variables.

- **Nodes Involved:**  
  - "Create and rename Google File"

- **Node Details:**

  - **Node Name:** Create and rename Google File  
  - **Type:** Google Drive node (Copy file operation)  
  - **Technical Role:** Performs the file duplication and assigns a new name.  
  - **Configuration:**  
    - Operation: Copy  
    - File ID: taken from the output of the previous "Search file to duplicate" node.  
    - New file name: uses the event title from the Filter node’s JSON (`{{ $('Filter specific event').item.json.source.title }}`).  
    - No additional options configured.  
  - **Input/Output:**  
    - Input comes from "Search file to duplicate".  
    - Output is the metadata of the newly created file in Google Drive.  
  - **Version-specific:** Google Drive node version 3.  
  - **Potential Failures:**  
    - Copy operation fails if file ID invalid or permissions insufficient.  
    - Expression resolving the new file name fails if expected JSON path is missing.  
    - Name conflicts or restrictions by Google Drive naming rules.  
  - **Sticky Notes:**  
    - Describes how to rename the file using variables from the calendar event (e.g., title, date, attendees).

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                          | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                           |
|---------------------------|---------------------------|----------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| New event in Google Calendar | Google Calendar Trigger   | Triggers workflow on new matching event | None (Trigger)               | Filter specific event        | You can exchange with a trigger from another scheduling tool (Calendly, Hubspot)                    |
| Filter specific event      | Filter                    | Filters events by creator and source URL | New event in Google Calendar | Search folder               | You can filter with any pattern that matches the type of event you're trying to retrieve (title, organizer, attendees...) |
| Search folder             | Google Drive (Search)      | Searches for folder named "SCREENING"  | Filter specific event        | Search file to duplicate     | Search the folder where the file you want to duplicate is located.                                  |
| Search file to duplicate   | Google Drive (Search)      | Searches template file to duplicate    | Search folder                | Create and rename Google File | Search the file you want to duplicate.                                                             |
| Create and rename Google File | Google Drive (Copy file) | Copies and renames the template file   | Search file to duplicate     | None                        | Rename the new file whichever way you want, using variables from the first step (Calendar event). Ex: {Title of the event} \| {date of the event} \| {attendees} |
| Sticky Note                | Sticky Note               | Setup instructions and notes           | None                        | None                        | Set up credentials for Google Calendar, Google Drive, and Google File. You’ll need a Google Workspace account. Setup Filter, Drive, File, and renaming variables. |
| Sticky Note1               | Sticky Note               | Scheduling tool replacement suggestion | None                        | None                        | You can exchange with a trigger from another scheduling tool (Calendly, Hubspot)                    |
| Sticky Note2               | Sticky Note               | Filter customization guidance          | None                        | None                        | You can filter with any pattern that matches the type of event you're trying to retrieve (title, description, organiser, attendees...) |
| Sticky Note3               | Sticky Note               | Folder search reminder                  | None                        | None                        | Search the folder where the file you want to duplicate is located.                                 |
| Sticky Note4               | Sticky Note               | File search reminder                    | None                        | None                        | Search the file you want to duplicate.                                                             |
| Sticky Note5               | Sticky Note               | File rename guidance                    | None                        | None                        | Rename the new file whichever way you want, using variables from the first step (Calendar event). Ex: {Title of the event} | {date of the event} | {attendees} |
| Sticky Note6               | Sticky Note               | Suggests multiple event-type paths     | None                        | None                        | You can create multiple path for different type of events                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger Node:**
   - Node Type: Google Calendar Trigger  
   - Name: "New event in Google Calendar"  
   - Configure credentials using your Google OAuth2 account with Calendar scope.  
   - Set trigger on "eventCreated".  
   - Poll every minute.  
   - Set calendarId to your target calendar email (e.g., candice@n8n.io).  
   - Set "Match Term" to `"Interview with"` to filter events by title.

2. **Add Filter Node:**
   - Node Type: Filter  
   - Name: "Filter specific event"  
   - Configure two conditions combined by AND:  
     - `creator.email` equals `"candice@n8n.io"`  
     - `source.url` contains `"https://n8n.workable.com/events/"`  
   - Use the JSON path expressions for these values.  
   - Connect output of "New event in Google Calendar" to this Filter node.

3. **Add Google Drive Search Node to find Folder:**
   - Node Type: Google Drive  
   - Name: "Search folder"  
   - Configure credentials with Google Drive OAuth2 (with Drive scopes).  
   - Set resource to "fileFolder".  
   - Set query string to `"SCREENING"` (case-insensitive).  
   - Limit results to 1.  
   - Connect output of "Filter specific event" to this node.

4. **Add Google Drive Search Node to find Template File:**
   - Node Type: Google Drive  
   - Name: "Search file to duplicate"  
   - Use same Google Drive credentials.  
   - Resource: fileFolder (to search files and folders).  
   - Query string: `"Template | M1 | Senior AE | "` (exact match string to your template file name).  
   - Limit results to 1.  
   - Connect output of "Search folder" to this node.

5. **Add Google Drive Copy Node to duplicate and rename file:**
   - Node Type: Google Drive  
   - Name: "Create and rename Google File"  
   - Credentials: same Google Drive OAuth2 credentials.  
   - Operation: Copy file.  
   - File ID: Use expression to reference the template file ID from previous node (`{{$json["id"]}}` or via expression editor).  
   - Name: Use expression to set the new file name dynamically from the event data, e.g., `{{$node["Filter specific event"].json["source"]["title"]}}`.  
   - Connect output of "Search file to duplicate" to this node.

6. **Configure Credentials:**
   - Google Calendar OAuth2 with scope to read calendar events.  
   - Google Drive OAuth2 with scope to search and copy files.  
   - Ensure the credentials belong to a Google Workspace account with access to the calendars and Drive folders/files involved.

7. **Test the Workflow:**
   - Create a calendar event matching "Interview with" and originating from the specified source.  
   - Confirm the workflow triggers and creates a duplicate document with the expected name in Google Drive.

8. **Optional Enhancements:**
   - Add parallel paths or multiple filters for different event types.  
   - Replace Google Calendar trigger with other tools (Calendly, Hubspot).  
   - Add steps to send the new document link via Slack or integrate with Notion/ATS.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow template is primarily designed for professionals taking notes during calls like recruiters, HR, sales. | Workflow description                                                                                         |
| To extend functionality, consider integrating with Notion databases, Slack notifications, or ATS/CRM tools.          | "To go further" section in description                                                                      |
| Requires Google Workspace account with proper OAuth2 credentials for Google Calendar and Drive services.             | Setup instructions                                                                                           |
| Video resources and community forums on n8n can help customize triggers and expressions further.                     | Official n8n documentation and community (https://docs.n8n.io/)                                            |

---

This completes the comprehensive reference for the "When specific event created in Google Calendar, duplicate & rename Google File" workflow, enabling full understanding, reproduction, and modification.