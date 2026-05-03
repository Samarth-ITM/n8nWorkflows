Sync Google Calendar tasks to Trello every day

https://n8nworkflows.xyz/workflows/sync-google-calendar-tasks-to-trello-every-day-1118


# Sync Google Calendar tasks to Trello every day

### 1. Workflow Overview

This workflow automates the daily synchronization of Google Calendar events into Trello cards. Its main purpose is to create Trello task cards from calendar events each morning, enabling users to track meetings and tasks on Trello with additional notes, labels, and automations.

The workflow comprises the following logical blocks:

- **1.1 Scheduled Trigger:** Starts the workflow every day at 8:00 AM.
- **1.2 Date Range Determination:** Calculates the start and end timestamps for the current day.
- **1.3 Google Calendar Event Retrieval:** Fetches all Google Calendar events occurring within the day.
- **1.4 Event Processing & Filtering:** Splits events into individual items, sets Trello card details, and filters out specific recurring tasks.
- **1.5 Trello Card Creation:** Creates Trello cards for the filtered events with customized details.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution daily at 8:00 AM.

- **Nodes Involved:**  
  - *Trigger Every Day at 8am*

- **Node Details:**

  - **Trigger Every Day at 8am**  
    - *Type:* Cron Trigger  
    - *Role:* Initiates workflow every day at the specified hour.  
    - *Configuration:* Set to trigger at 8 AM daily.  
    - *Input/Output:* No input; outputs a single trigger event.  
    - *Edge cases:*  
      - Timezone misconfiguration could cause incorrect trigger time.  
      - Cron node downtime or workflow inactive status will prevent execution.

---

#### 1.2 Date Range Determination

- **Overview:**  
  Computes the beginning and end timestamps of the current day in ISO format to query calendar events within this range.

- **Nodes Involved:**  
  - *Get Start & End of day*

- **Node Details:**

  - **Get Start & End of day**  
    - *Type:* Function Node  
    - *Role:* Calculate and output the current day's start (00:00:00.000) and end (23:59:59.099) timestamps.  
    - *Configuration:*  
      - Uses JavaScript Date object to set date boundaries.  
      - Outputs two properties: `from` and `to` (ISO strings).  
      - Adds an unused variable `myVariable` set to 1 (no effect).  
    - *Input:* Trigger output (empty input).  
    - *Output:* JSON object with `from` and `to` timestamps for the day.  
    - *Edge cases:*  
      - Date calculations assume system timezone; could cause issues if calendar events are in another timezone.  
      - The millisecond for end time is set to 99 which is unusual; typical maximum is 999. Minor impact only.  
    - *Version:* Compatible with n8n v0.150+ (no special demands).

---

#### 1.3 Google Calendar Event Retrieval

- **Overview:**  
  Retrieves all Google Calendar events occurring between the computed start and end of the day.

- **Nodes Involved:**  
  - *Get Todays Events*

- **Node Details:**

  - **Get Todays Events**  
    - *Type:* Google Calendar Node  
    - *Role:* Fetch calendar events within time boundaries.  
    - *Configuration:*  
      - Uses `timeMin` and `timeMax` parameters from previous node (`Get Start & End of day`).  
      - `singleEvents` set to true to expand recurring events into individual instances.  
      - Calendar email specified explicitly (`amenendez@threatconnect.com`).  
      - Operation: `getAll` to retrieve all matching events.  
    - *Credentials:* OAuth2 credentials named “Angel TC Calendar API”.  
    - *Input:* Date range from previous function node.  
    - *Output:* Array of event objects with fields like summary, description, start time, htmlLink.  
    - *Edge cases:*  
      - Authentication failure if credentials expire or revoked.  
      - API rate limits or network issues may cause errors.  
      - Empty event list if no events are scheduled.  
      - Timezone mismatch if calendar events have localized times.  
    - *Version:* Requires n8n version with Google Calendar OAuth2 support (v0.115+).

---

#### 1.4 Event Processing & Filtering

- **Overview:**  
  Splits the batch of events into individual items for processing, sets Trello card fields, and filters out certain recurring tasks by event summary.

- **Nodes Involved:**  
  - *Split Events In Batches*  
  - *Set Trello Card Details*  
  - *Remove Recurring Tasks*

- **Node Details:**

  - **Split Events In Batches**  
    - *Type:* SplitInBatches Node  
    - *Role:* Processes one event per batch to handle events individually.  
    - *Configuration:* Batch size set to 1.  
    - *Input:* Array of events from Google Calendar node.  
    - *Output:* Single event JSON object passed downstream per batch.  
    - *Edge cases:*  
      - Very large event lists will cause many batches and longer execution time.  
      - If input is empty, no batches processed.

  - **Set Trello Card Details**  
    - *Type:* Set Node  
    - *Role:* Prepare Trello card data fields from the current event.  
    - *Configuration:*  
      - Sets `name` to event summary.  
      - Sets `description` to event description (notes currently used; can be changed to calendar description).  
      - Sets `duedate` to event start datetime.  
      - Sets `URL` to event HTML link.  
    - *Expressions:* Uses direct references to `$node["Split Events In Batches"].json` event properties.  
    - *Input:* Single event JSON from SplitInBatches.  
    - *Output:* Structured card details JSON for Trello creation.  
    - *Edge cases:*  
      - Missing event fields could produce undefined values.  
      - Malformed date strings may cause downstream errors.

  - **Remove Recurring Tasks**  
    - *Type:* If Node  
    - *Role:* Filters out events whose summaries match specific recurring tasks to avoid creating Trello cards for them.  
    - *Configuration:*  
      - Checks if event summary equals any of the following strings:  
        - "Check email and start day"  
        - "Lunch"  
        - "Wrap Up & Clear Desk"  
        - "Beers and Griping"  
      - Combines conditions with OR logic (`combineOperation: "any"`).  
    - *Input:* Set Trello Card Details output.  
    - *Output:*  
      - If TRUE (matches one of the summaries), routes to *Delete Task* (no operation).  
      - If FALSE (no match), routes to *Create Trello Cards* node.  
    - *Edge cases:*  
      - Case-sensitive match may cause unintended passes/fails.  
      - If event summary is missing, expression may fail or default to no match.

---

#### 1.5 Trello Card Creation

- **Overview:**  
  Creates Trello cards for filtered calendar events with custom fields, due dates, and links.

- **Nodes Involved:**  
  - *Create Trello Cards*  
  - *Delete Task* (NoOp node for ignored events)

- **Node Details:**

  - **Create Trello Cards**  
    - *Type:* Trello Node  
    - *Role:* Adds a new card on Trello using event details.  
    - *Configuration:*  
      - `name`: Event summary (from Set Trello Card Details).  
      - `description`: A static multiline template designed for meeting notes, next steps, decisions, and discussion points.  
        (Note: The description is currently static and does not pull calendar event description.)  
      - `due`: Event start datetime.  
      - `idLabels`: Currently empty. User must configure label ID for meeting type (see sticky note).  
      - `urlSource`: Event HTML link (to Google Calendar event).  
    - *Credentials:* Trello API credentials named "Angel Work Trello".  
    - *Input:* Filtered event card details.  
    - *Output:* Created Trello card info.  
    - *Edge cases:*  
      - Missing or invalid label ID may prevent proper labeling.  
      - API errors due to auth failure, rate limits, or incorrect board/list IDs.  
      - Card name or due date missing or invalid may cause creation failure.

  - **Delete Task**  
    - *Type:* NoOp Node  
    - *Role:* Placeholder node for events filtered out (to discard them quietly).  
    - *Configuration:* None.  
    - *Input:* Events matching recurring tasks.  
    - *Output:* None (ends branch).

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                            | Input Node(s)            | Output Node(s)                | Sticky Note                                                                                                                                                                                                                                         |
|--------------------------|---------------------|--------------------------------------------|--------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger Every Day at 8am | Cron Trigger        | Starts workflow daily at 8:00 AM           | None                     | Get Start & End of day        |                                                                                                                                                                                                                                                     |
| Get Start & End of day    | Function            | Calculates current day start and end times | Trigger Every Day at 8am | Get Todays Events             |                                                                                                                                                                                                                                                     |
| Get Todays Events         | Google Calendar     | Retrieves all events for the day            | Get Start & End of day    | Split Events In Batches       |                                                                                                                                                                                                                                                     |
| Split Events In Batches   | SplitInBatches      | Splits event array into single items        | Get Todays Events         | Set Trello Card Details, Get Todays Events (retry) |                                                                                                                                                                                                                                                     |
| Set Trello Card Details   | Set                 | Prepares Trello card fields from event data | Split Events In Batches   | Remove Recurring Tasks        |                                                                                                                                                                                                                                                     |
| Remove Recurring Tasks    | If                  | Filters out specific recurring tasks        | Set Trello Card Details   | Delete Task (if true), Create Trello Cards (if false) |                                                                                                                                                                                                                                                     |
| Delete Task              | NoOp                | Discards filtered events                     | Remove Recurring Tasks    | None                         |                                                                                                                                                                                                                                                     |
| Create Trello Cards       | Trello              | Creates Trello cards from calendar events   | Remove Recurring Tasks    | None                         | When deploying, change Label ID for meeting type under "Create Trello Cards". Instructions [Here](https://stackoverflow.com/questions/42713100/trello-label-id-in-microsoft-flow#:~:text=Go%20to%20any%20board%20and,info%20should%20be%20right%20there.) to find label ID. Also adjust description if desired. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger Node**  
   - Name: `Trigger Every Day at 8am`  
   - Type: Cron Trigger  
   - Configure to trigger daily at 8:00 AM (set hour to 8, minutes and seconds to 0).  
   - No credentials needed.  

2. **Create a Function Node**  
   - Name: `Get Start & End of day`  
   - Type: Function  
   - Paste the following JavaScript code to calculate today's start and end ISO timestamps:  
     ```javascript
     var curr = new Date();
     var first = curr.getDate();
     var last = first;
     var firstday = new Date(curr.setDate(first));
     var lastday = new Date(curr.setDate(last));
     var beginning = new Date(firstday.setHours(0,0,0,0));
     var ending = new Date(lastday.setHours(23,59,59,99));
     items[0].json.from = beginning.toISOString();
     items[0].json.to = ending.toISOString();
     return items;
     ```  
   - Connect input from `Trigger Every Day at 8am`.

3. **Create a Google Calendar Node**  
   - Name: `Get Todays Events`  
   - Type: Google Calendar  
   - Operation: `getAll`  
   - Set `timeMin` to expression: `{{$node["Get Start & End of day"].json["from"]}}`  
   - Set `timeMax` to expression: `{{$node["Get Start & End of day"].json["to"]}}`  
   - Enable `singleEvents` option to true (expand recurring events).  
   - Specify your calendar email address.  
   - Credentials: Create and select OAuth2 credentials for Google Calendar API (e.g., "Angel TC Calendar API").  
   - Connect input from `Get Start & End of day`.

4. **Create a SplitInBatches Node**  
   - Name: `Split Events In Batches`  
   - Type: SplitInBatches  
   - Batch size: 1  
   - Connect input from `Get Todays Events`.

5. **Create a Set Node**  
   - Name: `Set Trello Card Details`  
   - Type: Set  
   - Create fields with expressions referencing the current event in batch:  
     - `name`: `={{$node["Split Events In Batches"].json["summary"]}}`  
     - `description`: `={{$node["Split Events In Batches"].json["description"]}}`  
     - `duedate`: `={{$node["Split Events In Batches"].json["start"]["dateTime"]}}`  
     - `URL`: `={{$node["Split Events In Batches"].json["htmlLink"]}}`  
   - Connect input from `Split Events In Batches`.

6. **Create an If Node**  
   - Name: `Remove Recurring Tasks`  
   - Type: If  
   - Configure string equals conditions for the event summary field (`{{$node["Split Events In Batches"].json["summary"]}}`):  
     - "Check email and start day"  
     - "Lunch"  
     - "Wrap Up & Clear Desk"  
     - "Beers and Griping"  
   - Combine operation: OR (any)  
   - Connect input from `Set Trello Card Details`.

7. **Create a NoOp Node**  
   - Name: `Delete Task`  
   - Type: NoOp (does nothing)  
   - Connect from the TRUE output of `Remove Recurring Tasks` (events to discard).

8. **Create a Trello Node**  
   - Name: `Create Trello Cards`  
   - Type: Trello  
   - Configure to create cards on your Trello board:  
     - `name`: `={{$node["Set Trello Card Details"].json["name"]}}`  
     - `description`: (Use multiline static template or customize as needed)  
       ```
       **Meeting purpose (*Integrations, Playbooks, UI Issues, Project*):**

       - Task

       **Next Steps (*Task, Assigned to, Checkpoint Date*):**

       - Task

       **Decisions Made: (*What, Why, Impacts*):**

       - Task

       **Discussion: (*Items/Knowledge Shared*):**

       - Task
       ```  
     - `due`: `={{$node["Set Trello Card Details"].json["duedate"]}}`  
     - `idLabels`: (Set your label ID here for meeting type)  
     - `urlSource`: `={{$node["Set Trello Card Details"].json["URL"]}}`  
   - Credentials: Set Trello API credentials (e.g., "Angel Work Trello").  
   - Connect input from FALSE output of `Remove Recurring Tasks`.

9. **Verify all connections:**  
   ```
   Trigger Every Day at 8am → Get Start & End of day → Get Todays Events → Split Events In Batches → Set Trello Card Details → Remove Recurring Tasks  
   Remove Recurring Tasks TRUE → Delete Task  
   Remove Recurring Tasks FALSE → Create Trello Cards
   ```

10. **Additional configuration:**  
    - Update the Trello `idLabels` field in the `Create Trello Cards` node with your Trello label ID for meeting types.  
    - Adjust the card description template in the `Create Trello Cards` node if you want to pull event descriptions or other dynamic content from Google Calendar.  
    - Change the Cron trigger time if you want the sync to run at a different hour.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                                                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When deploying, remember to change the Trello Label ID for meeting types under "Create Trello Cards" node.                  | Instructions to find Trello Label ID: [Stack Overflow Link](https://stackoverflow.com/questions/42713100/trello-label-id-in-microsoft-flow#:~:text=Go%20to%20any%20board%20and,info%20should%20be%20right%20there.)                             |
| The Trello card description is currently a static template focused on meeting notes; modify it to pull Google Calendar descriptions if desired. | This allows better customization of Trello cards according to your workflow needs.                                                                                                                                                           |
| The workflow assumes Google Calendar and Trello OAuth2 credentials are properly set up and authorized before use.           | Missing or invalid credentials will cause runtime failures.                                                                                                                                                                                  |
| Timezone considerations: The function node uses system timezone for date calculations; ensure consistency with your calendar timezone to avoid missing or duplicated events. | Consider adjusting the function code or Google Calendar node timezone settings if needed.                                                                                                                                                     |

---

This completes the detailed reference documentation for the "Sync Google Calendar tasks to Trello every day" workflow. It provides a clear understanding of the nodes, their logic, and how to recreate or modify the workflow confidently.