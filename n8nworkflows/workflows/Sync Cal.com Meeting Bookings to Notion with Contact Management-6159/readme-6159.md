Sync Cal.com Meeting Bookings to Notion with Contact Management

https://n8nworkflows.xyz/workflows/sync-cal-com-meeting-bookings-to-notion-with-contact-management-6159


# Sync Cal.com Meeting Bookings to Notion with Contact Management

### 1. Workflow Overview

This workflow automates the synchronization of meeting bookings from Cal.com into a Notion workspace, including contact management. It listens to booking events (creation, cancellation, rescheduling) from Cal.com and updates corresponding records in Notion’s Meetings and Contacts databases accordingly. The workflow is designed for users who want to keep their Notion workspace updated with real-time meeting data and maintain an organized contact list linked to these meetings.

**Logical Blocks:**

- **1.1 Event Reception and Routing:** Captures Cal.com booking events and directs workflow paths based on the event type (creation, cancellation, rescheduling).
- **1.2 Contact Management:** Searches for the meeting attendee in the Notion Contacts database, creates a new contact if none exists.
- **1.3 Meeting Management:** Depending on the event type, creates, updates, or deletes meeting records in the Notion Meetings database.
- **1.4 User Guidance:** Sticky Notes provide instructions and usage guidance for setup and testing.

---

### 2. Block-by-Block Analysis

#### 1.1 Event Reception and Routing

**Overview:**  
This block triggers the workflow when a Cal.com meeting booking event occurs and routes execution based on the specific event type.

**Nodes Involved:**  
- Cal.com Trigger  
- Route based on trigger event type (Switch)

**Node Details:**

- **Cal.com Trigger**  
  - *Type:* Cal.com Trigger node (webhook-based event listener)  
  - *Configuration:* Listens for three events — BOOKING_CREATED, BOOKING_CANCELLED, BOOKING_RESCHEDULED  
  - *Key variables:* `$json.triggerEvent` holds event type, other booking details in `$json`  
  - *Input:* External webhook call from Cal.com  
  - *Output:* Emits event data forwarded to Switch node  
  - *Failures:* Possible webhook misconfiguration, API key invalid, connectivity issues.

- **Route based on trigger event type (Switch)**  
  - *Type:* Switch node (conditional routing)  
  - *Configuration:* Routes data based on `$json.triggerEvent` matching 'BOOKING_CREATED', 'BOOKING_CANCELLED', or 'BOOKING_RESCHEDULED'  
  - *Input:* Output from Cal.com Trigger  
  - *Output:* Three outputs corresponding to each event type  
  - *Failures:* Expression evaluation errors if event type missing or malformed

---

#### 1.2 Contact Management

**Overview:**  
Checks if the meeting attendee exists in Notion contacts by email; if not found, creates a new contact record.

**Nodes Involved:**  
- get contact  
- doesn't exist (If)  
- create contact

**Node Details:**

- **get contact**  
  - *Type:* Notion node (databasePage, getAll operation)  
  - *Configuration:* Queries Contacts database filtering by attendee’s email (`$json.attendees[0].email`)  
  - *Input:* Output from “Route based on trigger event type” (new event path)  
  - *Output:* Returns matching contacts or empty array  
  - *Failures:* Notion API access issues, missing database ID, malformed email data

- **doesn't exist (If)**  
  - *Type:* If node (boolean condition)  
  - *Configuration:* Checks if output from “get contact” is empty (`$('get contact').isEmpty()`)  
  - *Input:* Output from “get contact”  
  - *Output:* True if contact does not exist, false if found  
  - *Failures:* Expression errors if “get contact” output format changes

- **create contact**  
  - *Type:* Notion node (databasePage, create operation)  
  - *Configuration:* Creates a new contact page with title = attendee name and email set from Cal.com data  
  - *Input:* True branch of “doesn't exist”  
  - *Output:* New contact’s Notion page data  
  - *Failures:* Notion API permission or quota issues, invalid data format for email or title

---

#### 1.3 Meeting Management

**Overview:**  
Handles creation, updating, or deletion of meeting records in Notion depending on the booking event type.

**Nodes Involved:**  
- create meeting  
- get meeting  
- update meeting  
- get meeting1  
- delete

**Node Details:**

- **create meeting**  
  - *Type:* Notion node (databasePage, create operation)  
  - *Configuration:* Creates meeting entry with title, linked contact relation, event time, and custom properties (camera_choice, notes, cal id) from Cal.com data  
  - *Input:* Output from “create contact” or false branch of “doesn't exist” (contact exists)  
  - *Output:* Created meeting page data  
  - *Failures:* Notion API issues, relation property misconfiguration, invalid date/time or timezone formats

- **get meeting**  
  - *Type:* Notion node (databasePage, getAll operation)  
  - *Configuration:* Searches Meetings database by 'cal id' matching Cal.com bookingId (string)  
  - *Input:* Rescheduled event path from Switch node  
  - *Output:* Meeting(s) matching booking ID  
  - *Failures:* Errors if database ID missing or Notion API issues

- **update meeting**  
  - *Type:* Notion node (databasePage, update operation)  
  - *Configuration:* Updates meeting’s Event Time date property with new start time and timezone from reschedule event  
  - *Input:* Output from “get meeting”  
  - *Output:* Updated meeting page data  
  - *Failures:* Page ID missing, API permission errors, invalid date/time formats

- **get meeting1**  
  - *Type:* Notion node (databasePage, getAll operation)  
  - *Configuration:* Similar to “get meeting” node, used for cancellations  
  - *Input:* Cancelled event path from Switch node  
  - *Output:* Meeting(s) matching booking ID  
  - *Failures:* Same as “get meeting”

- **delete**  
  - *Type:* Notion node (databasePage, archive operation)  
  - *Configuration:* Archives (soft deletes) the meeting page identified by pageId  
  - *Input:* Output from “get meeting1”  
  - *Output:* Confirmation of archive operation  
  - *Failures:* Missing pageId, permission issues, API rate limiting

---

#### 1.4 User Guidance

**Overview:**  
Provides detailed instructions and helpful links to assist users in setup, testing, and understanding the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**

- **Sticky Note**  
  - *Type:* Sticky Note node  
  - *Content:* Overview usage guide with links to essay and video walkthroughs  
  - *Note:* Contains links:  
    - Essay: https://www.simonesmerilli.com/business/cal-notion-automation  
    - Video: https://docs.n8n.io/workflows/sticky-notes/  
    - Author credit to simo (https://www.simosme.com)

- **Sticky Note1**  
  - *Type:* Sticky Note node  
  - *Content:* Step 1 instructions about connecting Cal.com account, API key creation, testing trigger events with execute step or workflow run, and reminders about testing cancellation and reschedule events.  
  - *Link:* https://app.cal.com/settings/developer/api-keys

- **Sticky Note2**  
  - *Type:* Sticky Note node  
  - *Content:* Step 2 instructions about connecting Notion workspace, granting integration access, adding “cal id” property to meetings database, acquiring database IDs, and configuring nodes accordingly.  
  - *Links:*  
    - Notion integration docs: https://developers.notion.com/docs/authorization#internal-integration-auth-flow-set-up  
    - Database ID info: https://developers.notion.com/docs/working-with-databases#adding-pages-to-a-database

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                  | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                                                           |
|------------------------------|---------------------|---------------------------------|----------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Cal.com Trigger              | Cal.com Trigger     | Event reception from Cal.com     | (Webhook external)          | Route based on trigger event type | See Sticky Note1 for Cal.com connection and testing instructions                                                                                       |
| Route based on trigger event type (Switch) | Switch              | Routes by event type             | Cal.com Trigger            | get contact / get meeting1 / get meeting |                                                                                                                                                       |
| get contact                  | Notion              | Searches Contacts by email       | Route based on trigger event type (new event) | doesn't exist              | See Sticky Note2 for Notion connection and database setup                                                                                            |
| doesn't exist (If)           | If                  | Checks if contact exists         | get contact                | create contact / create meeting |                                                                                                                                                       |
| create contact               | Notion              | Creates new contact in Notion    | doesn't exist (true branch) | create meeting             |                                                                                                                                                       |
| create meeting               | Notion              | Creates new meeting in Notion    | create contact / doesn't exist (false branch) | (end)                     | See Sticky Note2 for Notion connection and database setup                                                                                            |
| get meeting                 | Notion              | Finds meeting to update          | Route based on trigger event type (rescheduled) | update meeting             |                                                                                                                                                       |
| update meeting              | Notion              | Updates meeting event time       | get meeting                | (end)                      |                                                                                                                                                       |
| get meeting1                | Notion              | Finds meeting to delete          | Route based on trigger event type (cancelled)   | delete                     |                                                                                                                                                       |
| delete                      | Notion              | Archives (deletes) meeting       | get meeting1               | (end)                      |                                                                                                                                                       |
| Sticky Note                 | Sticky Note         | Usage guide and credits          |                            |                            | Usage guide with links to essay and video walkthrough                                                                                                 |
| Sticky Note1                | Sticky Note         | Cal.com connection and testing   |                            |                            | Cal.com API key and testing instructions                                                                                                            |
| Sticky Note2                | Sticky Note         | Notion connection and setup      |                            |                            | Notion integration and database setup instructions                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cal.com Trigger Node**  
   - Type: Cal.com Trigger  
   - Configure: Connect your Cal.com account via API key (create key at https://app.cal.com/settings/developer/api-keys)  
   - Set events to listen: BOOKING_CREATED, BOOKING_CANCELLED, BOOKING_RESCHEDULED  
   - Save webhook ID generated by n8n

2. **Add Switch Node (Route based on trigger event type)**  
   - Type: Switch  
   - Input: Connect from Cal.com Trigger output  
   - Add 3 rules based on expression `{{$json.triggerEvent}}`:  
     - Equals "BOOKING_CREATED" → output "new event"  
     - Equals "BOOKING_CANCELLED" → output "cancelled"  
     - Equals "BOOKING_RESCHEDULED" → output "rescheduled"

3. **Set up Contact Management for New Events:**

   3.1. Add Notion node named "get contact"  
   - Resource: databasePage  
   - Operation: getAll  
   - Database ID: Contacts database ID from Notion  
   - Filter: Email equals `{{$json.attendees[0].email}}`  
   - Credentials: Notion integration with access to Contacts database  
   - Connect from Switch node "new event" output

   3.2. Add If node named "doesn't exist"  
   - Condition: Expression `{{$node["get contact"].isEmpty()}}` is true  
   - Connect from "get contact"

   3.3. Add Notion node named "create contact"  
   - Resource: databasePage  
   - Operation: create  
   - Database ID: Contacts database ID  
   - Title: `{{$json.attendees[0].name}}`  
   - Email property: `{{$json.attendees[0].email}}`  
   - Credentials: Notion integration  
   - Connect from "doesn't exist" true output

4. **Create Meeting Record:**

   4.1. Add Notion node named "create meeting"  
   - Resource: databasePage  
   - Operation: create  
   - Database ID: Meetings database ID  
   - Title: `{{$json.title}}` (from Cal.com Trigger)  
   - Properties:  
     - Contacts (relation): Use expression to get ID from either "get contact" or "create contact" node outputs `{{$ifEmpty($node["get contact"].item.json.id, $node["create contact"].id)}}`  
     - Event Time (date): `{{$json.startTime}}` with timezone `{{$json.organizer.timeZone}}`  
     - cal id (text): `{{$json.bookingId.toString()}}`  
   - Blocks (rich text): include camera_choice and notes responses from Cal.com Trigger data  
   - Credentials: Notion integration  
   - Connect from "create contact" output and also from "doesn't exist" false output (contact exists)

5. **Handle Rescheduled Events:**

   5.1. Add Notion node "get meeting"  
   - Resource: databasePage  
   - Operation: getAll  
   - Database ID: Meetings database ID  
   - Filter: cal id equals `{{$json.bookingId.toString()}}`  
   - Connect from Switch node "rescheduled" output

   5.2. Add Notion node "update meeting"  
   - Resource: databasePage  
   - Operation: update  
   - Page ID: `{{$json.id}}` (from "get meeting")  
   - Property to update: Event Time (date) set to `{{$json.rescheduleStartTime}}` with timezone `{{$json.organizer.timeZone}}`  
   - Credentials: Notion integration  
   - Connect from "get meeting"

6. **Handle Cancelled Events:**

   6.1. Add Notion node "get meeting1"  
   - Same config as "get meeting"  
   - Connect from Switch node "cancelled" output

   6.2. Add Notion node "delete"  
   - Operation: archive page  
   - Page ID: `{{$json.id}}` (from "get meeting1")  
   - Credentials: Notion integration  
   - Connect from "get meeting1"

7. **Add Sticky Notes for User Guidance:**

   7.1. Add Sticky Note with usage guide and links to essay and video  
   7.2. Add Sticky Note with step 1 instructions for Cal.com connection and testing  
   7.3. Add Sticky Note with step 2 instructions for Notion connection, database setup, and configuration

8. **Confirm all credentials are set correctly:**

   - Cal.com credentials via API key for trigger node  
   - Notion integration credentials with database access rights for Meetings and Contacts databases

9. **Test workflow:**

   - Test individual nodes by executing steps in n8n manually  
   - Schedule, reschedule, and cancel meetings in Cal.com to verify correct Notion synchronization  
   - Monitor logs for errors (e.g., API permission issues, missing data)

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                               |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Usage guide with comprehensive written and video walkthroughs of the workflow.                                | Essay: https://www.simonesmerilli.com/business/cal-notion-automation<br>Video: https://docs.n8n.io/workflows/sticky-notes/ |
| To connect your Cal.com account, create an API key in the dedicated settings page.                            | https://app.cal.com/settings/developer/api-keys                                                              |
| For Notion integration, follow the internal integration auth flow and ensure the integration has database access. | https://developers.notion.com/docs/authorization#internal-integration-auth-flow-set-up                        |
| To find your Notion database IDs, see the official documentation.                                             | https://developers.notion.com/docs/working-with-databases#adding-pages-to-a-database                          |
| Add a "cal id" text property to your Notion Meetings database to identify Cal.com bookings uniquely.          | Manual setup step; no direct link                                                                              |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.