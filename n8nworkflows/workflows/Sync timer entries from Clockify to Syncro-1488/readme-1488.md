Sync timer entries from Clockify to Syncro

https://n8nworkflows.xyz/workflows/sync-timer-entries-from-clockify-to-syncro-1488


# Sync timer entries from Clockify to Syncro

### 1. Workflow Overview

This workflow synchronizes time entries from Clockify to tickets in Syncro MSP. It is designed to listen for updates on time entries in Clockify via a webhook, then either create or update corresponding timer entries on Syncro tickets. The workflow maintains synchronization state by storing paired Clockify and Syncro timer entry IDs in a Google Sheet, enabling idempotent updates. It also matches technicians by name using a predefined mapping.

Logical blocks:

- **1.1 Input Reception and Validation:** Receives webhook calls from Clockify when time entries are updated and verifies that the time entry is associated with a project containing "Ticket" in its name.

- **1.2 Environment and Ticket Identification:** Loads environment variables (Syncro base URL) and extracts the Syncro ticket ID from the Clockify project name.

- **1.3 Technician Matching:** Matches the Clockify user (technician) name to a Syncro user ID using a predefined mapping stored in a Set node.

- **1.4 Existing Timer Entry Lookup:** Checks the Google Sheet to see if the incoming Clockify time entry already has a mapped Syncro timer entry.

- **1.5 Conditional Timer Entry Creation or Update:** Depending on whether a match exists, either creates a new timer entry on Syncro or updates the existing one.

- **1.6 State Persistence:** Records new mappings of Clockify and Syncro timer entry IDs in Google Sheets after successful creation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

- **Overview:** Accepts incoming webhook POST requests from Clockify triggered on time entry updates. Validates that the time entry is related to a project containing the string "Ticket" to filter relevant entries.

- **Nodes Involved:** `Webhook`, `IF1`, `EnvVariables`, `NoOp`

- **Node Details:**

  - **Webhook**
    - Type: Webhook (HTTP POST endpoint)
    - Config: Listens on path `82b654d7-aeb2-4cc1-97a8-0ebd1a729202`
    - Outputs all received data downstream.
    - Input: External HTTP POST from Clockify webhook.
    - Output: JSON payload of the time entry update.
    - Edge cases: Missing or malformed webhook payload; unauthorized calls (no auth configured).

  - **IF1**
    - Type: IF
    - Config: Checks if `body.project.name` contains "Ticket".
    - Input: Webhook output.
    - Output: True branch continues workflow; False branch terminates with NoOp.
    - Edge cases: Missing or empty project name; case sensitivity.

  - **EnvVariables**
    - Type: Set
    - Config: Defines Syncro base URL (`https://subdomain.syncromsp.com`).
    - Input: True output of IF1.
    - Output: Sets environment variables for later HTTP requests.
    - Edge cases: Hardcoded URL requires manual update per deployment.

  - **NoOp**
    - Type: No operation
    - Config: Ends workflow for irrelevant entries.
    - Input: False output of IF1.
    - Output: None.
    - Edge cases: None.

#### 1.2 Environment and Ticket Identification

- **Overview:** Extracts Syncro ticket ID from Clockify project name using regex and prepares for technician matching.

- **Nodes Involved:** `ForSyncro`, `SetTechnicians`, `MatchTechnician`

- **Node Details:**

  - **ForSyncro**
    - Type: Set
    - Config: Extracts ticket ID from `body.project.name` using regex `\[(\d+)]`.
    - Input: Output of `EnvVariables`.
    - Output: JSON with key `id` = extracted Syncro ticket ID.
    - Edge cases: Project name missing bracketed ticket ID; regex fails; no fallback.

  - **SetTechnicians**
    - Type: Set
    - Config: Hardcoded mapping of technician names to Syncro user IDs, e.g., `"Tech 1": "1234"`.
    - Input: Output of `ForSyncro`.
    - Output: JSON object with key-value pairs of technician names and IDs.
    - Edge cases: Requires exact name matching; no support for multiple identical names.

  - **MatchTechnician**
    - Type: Function
    - Config: Matches Clockify user name from webhook payload against keys in the technician map.
    - Input: Output of `SetTechnicians` plus `Webhook` node context.
    - Output: JSON with matched technician ID and name.
    - Edge cases: No match found results in empty output; no fallback or error handling.

#### 1.3 Existing Timer Entry Lookup

- **Overview:** Searches Google Sheets to determine if this Clockify time entry has an existing Syncro timer entry mapping.

- **Nodes Involved:** `FindMatch`, `IF`

- **Node Details:**

  - **FindMatch**
    - Type: Google Sheets (Lookup)
    - Config: Searches columns A:B for Clockify time entry ID; returns all matches.
    - Input: Output of `MatchTechnician`.
    - Output: Matching rows with `Clockify` and `Syncro` timer IDs.
    - Edge cases: Google API errors; multiple matches; empty results.

  - **IF**
    - Type: IF
    - Config: Checks if `FindMatch` returned any data (non-empty).
    - Input: Output of `FindMatch`.
    - Output: True branch for existing mapping (update path), False branch for new entry creation.
    - Edge cases: Data format issues; expression failures.

#### 1.4 Conditional Timer Entry Creation or Update

- **Overview:** Depending on prior lookup, either creates a new timer entry on Syncro or updates an existing one.

- **Nodes Involved:** `UpdateSyncroTimer`, `NewSyncroTimer`, `ForGoogle`

- **Node Details:**

  - **UpdateSyncroTimer**
    - Type: HTTP Request (PUT)
    - Config:
      - URL: `${syncro_baseurl}/api/v1/tickets/{ticket_id}/update_timer_entry`
      - Body: timer_entry_id, start_time, end_time, notes, user_id
      - Auth: Header Auth with Syncro credentials
    - Input: True branch of `IF` (existing timer)
    - Output: Updated Syncro timer entry response
    - Edge cases: HTTP errors; auth failures; invalid timer_entry_id.

  - **NewSyncroTimer**
    - Type: HTTP Request (POST)
    - Config:
      - URL: `${syncro_baseurl}/api/v1/tickets/{ticket_id}/timer_entry`
      - Body: start_at, end_at, notes, user_id
      - Auth: Header Auth with Syncro credentials
    - Input: False branch of `IF` (new timer)
    - Output: Newly created Syncro timer entry response
    - Edge cases: HTTP errors; auth failures; invalid ticket ID.

  - **ForGoogle**
    - Type: Set
    - Config: Extracts Syncro timer entry ID from new timer response and Clockify timer ID from webhook; prepares data for Google Sheets append.
    - Input: Output of `NewSyncroTimer`.
    - Output: JSON with `Syncro` and `Clockify` IDs.
    - Edge cases: Missing IDs in response; expression errors.

#### 1.5 State Persistence

- **Overview:** Appends the new time entry mapping to Google Sheets to maintain synchronization state.

- **Nodes Involved:** `Google Sheets`

- **Node Details:**

  - **Google Sheets**
    - Type: Google Sheets (Append)
    - Config:
      - Range: `A:B`
      - Operation: Append new row with Syncro and Clockify IDs
      - Value Input Mode: User Entered
      - Credentials: Google API OAuth2
    - Input: Output of `ForGoogle`.
    - Output: Append confirmation.
    - Edge cases: API quota limits; network errors; invalid sheet ID.

---

### 3. Summary Table

| Node Name         | Node Type            | Functional Role                        | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                              |
|-------------------|----------------------|-------------------------------------|-----------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook           | Webhook              | Receives Clockify webhook POST       | External HTTP         | IF1                      | Setup webhook in Clockify on "Time entry updated (anyone)" pointing here.                              |
| IF1               | IF                   | Filters entries with "Ticket" in project name | Webhook              | EnvVariables, NoOp        |                                                                                                        |
| EnvVariables      | Set                  | Sets Syncro base URL                  | IF1                   | ForSyncro                |                                                                                                        |
| ForSyncro         | Set                  | Extracts Syncro ticket ID from project name | EnvVariables         | SetTechnicians           |                                                                                                        |
| SetTechnicians    | Set                  | Defines technician name to Syncro ID mapping | ForSyncro             | MatchTechnician          | If multiple technicians share the same name or names don't match exactly, this fails.                  |
| MatchTechnician   | Function             | Matches Clockify user name to Syncro ID | SetTechnicians        | FindMatch                |                                                                                                        |
| FindMatch         | Google Sheets (Lookup) | Finds existing Syncro timer entries by Clockify ID | MatchTechnician      | IF                       |                                                                                                        |
| IF                | IF                   | Checks if existing Syncro timer entry exists | FindMatch             | UpdateSyncroTimer, NewSyncroTimer |                                                                                                        |
| UpdateSyncroTimer | HTTP Request (PUT)    | Updates existing Syncro timer entry  | IF (true)              |                          |                                                                                                        |
| NewSyncroTimer    | HTTP Request (POST)   | Creates new Syncro timer entry       | IF (false)             | ForGoogle                |                                                                                                        |
| ForGoogle         | Set                  | Prepares data to append new mapping  | NewSyncroTimer         | Google Sheets            |                                                                                                        |
| Google Sheets     | Google Sheets (Append)| Stores Clockify-Syncro timer ID pairs | ForGoogle             |                          |                                                                                                        |
| NoOp              | No Operation         | Ends workflow for irrelevant entries | IF1 (false)            |                          |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook
   - Method: POST
   - Path: Unique identifier (e.g., `82b654d7-aeb2-4cc1-97a8-0ebd1a729202`)
   - Response: Return all data, response mode last node
   - Purpose: Receive Clockify time entry updated webhook

2. **Add IF Node (IF1)**
   - Type: IF
   - Condition: Check if `{{$json["body"]["project"]["name"]}}` contains substring `"Ticket"`
   - Connect Webhook output to IF1 input

3. **Add NoOp Node**
   - Type: No Operation
   - Connect IF1 false output to NoOp

4. **Add Set Node (EnvVariables)**
   - Type: Set
   - Set string variable `syncro_baseurl` to your Syncro base URL (e.g., `https://subdomain.syncromsp.com`)
   - Connect IF1 true output to EnvVariables

5. **Add Set Node (ForSyncro)**
   - Type: Set
   - Add string variable `id` with expression: `{{$json["body"]["project"]["name"].match(/\[(\d+)]/)[1]}}`
   - Connect EnvVariables output to ForSyncro

6. **Add Set Node (SetTechnicians)**
   - Type: Set
   - Add string key-value pairs for technician name to Syncro user ID, e.g., `"Tech 1": "1234"`, `"Tech 2": "5678"`
   - Connect ForSyncro output to SetTechnicians

7. **Add Function Node (MatchTechnician)**
   - Type: Function
   - Paste code:

```javascript
const results = [];

const user = $node["Webhook"].json["body"]["user"];
const persons = items[0].json;

for (const key of Object.keys(persons)) {
  if (key === user.name) {
    results.push({ json: { id: persons[key], name: key } });
  }
}

return results;
```
   - Connect SetTechnicians output to MatchTechnician

8. **Add Google Sheets Node (FindMatch)**
   - Type: Google Sheets
   - Operation: Lookup
   - Lookup Column: `Clockify` (assumed column name)
   - Lookup Value: `{{$node["Webhook"].json["body"]["id"]}}`
   - Range: `A:B`
   - Sheet ID: Your Google Sheet storing mappings
   - Credentials: Google API OAuth2
   - Connect MatchTechnician output to FindMatch

9. **Add IF Node (IF)**
   - Type: IF
   - Condition: Check boolean `{{$boolean(!!Object.keys($node["FindMatch"].data).length)}}` equals `true`
   - Connect FindMatch output to IF

10. **Add HTTP Request Node (UpdateSyncroTimer)**
    - Type: HTTP Request
    - Method: PUT
    - URL: `{{$node["EnvVariables"].json["syncro_baseurl"]}}/api/v1/tickets/{{$node["ForSyncro"].json["id"]}}/update_timer_entry`
    - Authentication: Header Auth (Syncro credentials)
    - Body Parameters:
      - `timer_entry_id`: `{{$node["IF"].json["Syncro"]}}`
      - `start_time`: `{{$node["Webhook"].json["body"]["timeInterval"]["start"]}}`
      - `end_time`: `{{$node["Webhook"].json["body"]["timeInterval"]["end"]}}`
      - `notes`: `{{$node["Webhook"].json["body"]["description"]}}`
      - `user_id`: `{{$node["MatchTechnician"].json["id"]}}`
    - Connect IF true output to UpdateSyncroTimer

11. **Add HTTP Request Node (NewSyncroTimer)**
    - Type: HTTP Request
    - Method: POST
    - URL: `{{$node["EnvVariables"].json["syncro_baseurl"]}}/api/v1/tickets/{{$node["ForSyncro"].json["id"]}}/timer_entry`
    - Authentication: Header Auth (Syncro credentials)
    - Body Parameters:
      - `start_at`: `{{$node["Webhook"].json["body"]["timeInterval"]["start"]}}`
      - `end_at`: `{{$node["Webhook"].json["body"]["timeInterval"]["end"]}}`
      - `notes`: `{{$node["Webhook"].json["body"]["description"]}}`
      - `user_id`: `{{$node["MatchTechnician"].json["id"]}}`
    - Connect IF false output to NewSyncroTimer

12. **Add Set Node (ForGoogle)**
    - Type: Set
    - Variables:
      - `Syncro`: `{{$json["id"]}}` (from NewSyncroTimer response)
      - `Clockify`: `{{$node["Webhook"].json["body"]["id"]}}`
    - Connect NewSyncroTimer output to ForGoogle

13. **Add Google Sheets Node (Append)**
    - Type: Google Sheets
    - Operation: Append
    - Range: `A:B`
    - Value Input Mode: User Entered
    - Sheet ID: same as FindMatch
    - Credentials: Google API OAuth2
    - Connect ForGoogle output to Google Sheets append

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow requires exact matching of technician names between Clockify and SetTechnicians node; duplicates unsupported. | Technician mapping limitation                                                                  |
| Must configure Clockify webhook to trigger on "Time entry updated (anyone)" and point to the Webhook node URL.      | Clockify webhook setup instructions                                                            |
| Original workflow is part of an MSP collection available at https://github.com/bionemesis/n8nsyncro                 | Source repository and further MSP workflows                                                    |