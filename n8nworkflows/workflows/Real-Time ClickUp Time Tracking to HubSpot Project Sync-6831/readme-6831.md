Real-Time ClickUp Time Tracking to HubSpot Project Sync

https://n8nworkflows.xyz/workflows/real-time-clickup-time-tracking-to-hubspot-project-sync-6831


# Real-Time ClickUp Time Tracking to HubSpot Project Sync

### 1. Workflow Overview

This workflow automates the real-time synchronization of time tracked on ClickUp tasks with a custom project object in HubSpot. It is designed for teams managing detailed task time tracking in ClickUp while maintaining high-level project progress and sprint hour tracking in HubSpot. The workflow triggers whenever time tracked on a ClickUp task is updated, fetches detailed task and time entry information, extracts sprint and project-related data, and updates corresponding HubSpot project records accordingly.

**Logical Blocks:**

- **1.1 Input Reception:** Trigger node listens for ClickUp time-tracked updates.
- **1.2 Data Retrieval:** Fetch detailed task and time entry information from ClickUp.
- **1.3 Data Merging:** Combine task and time entry data for unified processing.
- **1.4 Data Processing:** Extract and calculate sprint, project, time, and status details from merged data.
- **1.5 HubSpot Project Lookup:** Search HubSpot custom objects for the matching project record.
- **1.6 Conditional Routing:** Decide update path based on sprint type (Sprints 1-4 or Additional Requests).
- **1.7 HubSpot Update:** Update total and sprint-specific hours in HubSpot custom object.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens to ClickUp events for any updates to tracked time on tasks within a specified team and space.
- **Nodes Involved:** 
  - Time Tracked Update Trigger
- **Node Details:**

  - **Time Tracked Update Trigger**
    - Type: ClickUp Trigger (Webhook)
    - Role: Initiates workflow on time tracked updates (`taskTimeTrackedUpdated` event) in a specified ClickUp space.
    - Configuration:
      - Team ID set to `9015730766`.
      - Event filter: `taskTimeTrackedUpdated`.
      - Space filter: `90153635535`.
      - Uses OAuth2 authentication linked to a ClickUp account.
    - Input: webhook event from ClickUp.
    - Output: JSON containing time entry interval ID and task ID.
    - Potential Failures:
      - Authentication errors if OAuth token expires.
      - Missing or incorrect team/space ID causing no triggers.
      - Webhook registration issues.
    - Version-Specific:
      - Uses typeVersion 1; ensure compatibility with current n8n ClickUp nodes.

#### 1.2 Data Retrieval

- **Overview:** Retrieves full task details and specific time entry details from ClickUp API for the triggered event.
- **Nodes Involved:** 
  - ClickUp: Get Task Details
  - ClickUp: Get Time Entry Details
- **Node Details:**

  - **ClickUp: Get Task Details**
    - Type: ClickUp node (get operation on task)
    - Role: Fetches complete task information using task ID from trigger.
    - Configuration:
      - `id` parameter set dynamically from previous node’s JSON (`{{$json.task_id}}`).
      - OAuth2 authentication.
    - Inputs:
      - Receives task ID from trigger.
    - Outputs:
      - Detailed task JSON including custom fields, project info, time spent, estimates, dates.
    - Failures/Potential Issues:
      - Authentication token expiry.
      - Task ID not found or deleted task.
      - API rate limits.
    - Version: 1

  - **ClickUp: Get Time Entry Details**
    - Type: ClickUp node (get operation on timeEntry resource)
    - Role: Fetches details of the specific time entry update.
    - Configuration:
      - Team ID `9015730766`.
      - Time entry ID from trigger (`{{$json.data.interval_id}}`).
      - OAuth2 authentication.
    - Inputs:
      - Receives time interval ID from trigger.
    - Outputs:
      - Time entry details JSON.
    - Failures:
      - Invalid time entry ID.
      - Authentication issues.
      - API rate limits.
    - Version: 1

#### 1.3 Data Merging

- **Overview:** Combines the outputs of the task details and time entry details nodes into a single data object for further processing.
- **Nodes Involved:** 
  - Merge: Task & Time Data
- **Node Details:**

  - **Merge: Task & Time Data**
    - Type: Merge node
    - Role: Combines task details (left input) and time entry details (right input) into one single item.
    - Configuration:
      - Mode: `combineBySql` to merge based on SQL-like matching.
    - Inputs:
      - Left input: task details.
      - Right input: time entry details.
    - Outputs:
      - Single JSON object combining both datasets.
    - Failures:
      - Mismatched or missing data causing merge failures.
      - Empty inputs.
    - Version: 3.1

#### 1.4 Data Processing

- **Overview:** Extracts relevant sprint, project, and time tracking data from the merged task and time entry information for subsequent HubSpot update.
- **Nodes Involved:** 
  - Code: Extract Sprint & Task Data
- **Node Details:**

  - **Code: Extract Sprint & Task Data**
    - Type: Code node (Python)
    - Role: Parses merged data to:
      - Extract sprint name from ClickUp custom fields.
      - Extract HubSpot Deal ID.
      - Convert timestamps to dates.
      - Calculate actual vs scoped hours (in hours).
      - Determine project start/end dates.
      - Calculate project status (on time, at risk, closed late, etc.).
      - Map sprint names to normalized keys.
      - Determine numeric sprint ID for routing.
      - Assemble output JSON suitable for HubSpot update.
    - Configuration:
      - Uses Python code with robust date parsing and error handling.
      - Utilizes predefined mappings for sprint names to keys.
      - Returns a structured dictionary with all required fields.
    - Inputs:
      - Single merged JSON item (task + time entry).
    - Outputs:
      - JSON with processed fields such as:
        - `project_name`, `project_start_date`, `project_end_date`
        - `total_scoped_hours`, `actual_hours_tracked`, `total_time_remaining`
        - `project_status`, `clickup_list_id`, `hs_object_id`
        - `target_actual_sprint_field_name`, `numeric_sprint_id`
        - Sprint-specific scoped and actual hours.
    - Failures:
      - Code exceptions (caught and logged; sets error flags).
      - Missing or malformed date fields.
      - Unexpected custom field structures.
    - Version: 2

#### 1.5 HubSpot Project Lookup

- **Overview:** Searches HubSpot custom objects for the project record matching the task’s project name to retrieve current stored hours and IDs.
- **Nodes Involved:** 
  - OnProjectFolder (HTTP Request)
- **Node Details:**

  - **OnProjectFolder**
    - Type: HTTP Request
    - Role: Queries HubSpot CRM custom objects via API to find project matching `project_name`.
    - Configuration:
      - POST request to HubSpot search API for custom object (`objectTypeId` placeholder).
      - Filters on `project_name` equals task project name.
      - Requests properties relevant to time tracking and project metadata.
      - Auth: HTTP Header Auth with Private App Token.
    - Inputs:
      - Receives processed JSON from code node.
    - Outputs:
      - HubSpot search results JSON with matching project records.
    - Failures:
      - Authentication errors.
      - Invalid or missing custom object ID in URL.
      - Network or API downtime.
      - No matching HubSpot project found.
    - Version: 4.2

#### 1.6 Conditional Routing

- **Overview:** Routes the flow based on whether the sprint is one of the main Sprints 1 to 4 or categorized as "Additional Requests" (or others).
- **Nodes Involved:** 
  - Route by Sprint Type (If node)
- **Node Details:**

  - **Route by Sprint Type**
    - Type: If node
    - Role: Checks if numeric sprint ID > 4 to distinguish additional requests from main sprints.
    - Configuration:
      - Condition: numeric_sprint_id > 4
      - True branch: Additional Requests update path.
      - False branch: Sprints 1-4 update path.
    - Inputs:
      - Receives project data from HubSpot search node.
    - Outputs:
      - Two outputs for conditional routing.
    - Failures:
      - Missing or invalid numeric sprint ID field.
      - Type mismatches in comparison.
    - Version: 2.2

#### 1.7 HubSpot Update

- **Overview:** Updates the HubSpot project record’s total and sprint-specific actual hours and remaining time based on the task time update.
- **Nodes Involved:** 
  - HubSpot: Update Project Hours (actual_additional_requests_hours)
  - HubSpot: Update Project Hours Sprint 1-4
- **Node Details:**

  - **HubSpot: Update Project Hours (actual_additional_requests_hours)**
    - Type: HTTP Request
    - Role: PATCHes the HubSpot project record for additional requests sprint hours.
    - Configuration:
      - URL uses HubSpot custom object ID and result record ID.
      - Updates:
        - `actual_hours_tracked` by adding the new tracked hours.
        - `total_time_remaining` by subtracting the new tracked hours.
      - Auth: HTTP Header Auth.
      - Retries: 2 attempts on failure.
    - Inputs:
      - Project search results + processed sprint hours.
    - Outputs:
      - Updated project record confirmation.
    - Failures:
      - API errors.
      - Incorrect object or property IDs.
      - Data type mismatches causing JSON parse errors.
    - Version: 4.2

  - **HubSpot: Update Project Hours Sprint 1-4**
    - Type: HTTP Request
    - Role: PATCHes the HubSpot project record for sprints 1 to 4 with sprint-specific actual hours.
    - Configuration:
      - Similar URL structure as above.
      - Dynamically updates sprint-specific field (`target_actual_sprint_field_name`) to current actual hours.
      - Updates total actual hours and remaining time similarly.
      - Auth: HTTP Header Auth.
      - Retries: 2 attempts.
    - Inputs:
      - Project search results + processed sprint hours.
    - Outputs:
      - Updated project record confirmation.
    - Failures:
      - Same as above.
      - Potential formatting or interpolation issues in expressions (noted incomplete JSON in raw).
    - Version: 4.2

---

### 3. Summary Table

| Node Name                                 | Node Type               | Functional Role                                   | Input Node(s)                       | Output Node(s)                               | Sticky Note                                                                                                                      |
|-------------------------------------------|-------------------------|--------------------------------------------------|------------------------------------|----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Time Tracked Update Trigger                | ClickUp Trigger         | Starts workflow on ClickUp task time updates     | -                                  | ClickUp: Get Task Details<br>ClickUp: Get Time Entry Details | See “Real-Time ClickUp Time Tracking to HubSpot Project Sync” sticky note for workflow overview and use case details.          |
| ClickUp: Get Task Details                  | ClickUp Node            | Fetches full task details from ClickUp API       | Time Tracked Update Trigger         | Merge: Task & Time Data                       |                                                                                                                                 |
| ClickUp: Get Time Entry Details            | ClickUp Node            | Fetches specific time entry details               | Time Tracked Update Trigger         | Merge: Task & Time Data                       |                                                                                                                                 |
| Merge: Task & Time Data                    | Merge Node              | Combines task and time entry details              | ClickUp: Get Task Details<br>ClickUp: Get Time Entry Details | Code: Extract Sprint & Task Data              |                                                                                                                                 |
| Code: Extract Sprint & Task Data           | Code (Python)           | Extracts, parses, and computes sprint/project info | Merge: Task & Time Data              | OnProjectFolder                               | See “Explanation Box” sticky note for detailed processing explanation and sync notes.                                          |
| OnProjectFolder                            | HTTP Request            | Searches HubSpot for matching project record      | Code: Extract Sprint & Task Data    | Route by Sprint Type                          |                                                                                                                                 |
| Route by Sprint Type                       | If Node                 | Routes based on sprint type (1-4 vs additional)   | OnProjectFolder                     | HubSpot: Update Project Hours (additional) <br> HubSpot: Update Project Hours Sprint 1-4 |                                                                                                                                 |
| HubSpot: Update Project Hours (actual_additional_requests_hours) | HTTP Request            | Updates HubSpot custom object for additional requests sprint | Route by Sprint Type (true branch) | -                                            |                                                                                                                                 |
| HubSpot: Update Project Hours Sprint 1-4 | HTTP Request            | Updates HubSpot custom object for Sprints 1-4     | Route by Sprint Type (false branch) | -                                            |                                                                                                                                 |
| Sticky Note4                              | Sticky Note             | Workflow overview, use case, requirements, setup | -                                  | -                                            | Contains full workflow overview, use case, requirements, and setup instructions.                                                |
| Sticky Note5                              | Sticky Note             | Summary explanation box and sync note             | -                                  | -                                            | Notes real-time sync and recommends initial full sync workflow if data is out of sync.                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create ClickUp Trigger Node**  
   - Type: ClickUp Trigger  
   - Configure to listen for `taskTimeTrackedUpdated` event.  
   - Set team ID to your ClickUp team (e.g., `9015730766`).  
   - Set space ID filter to your ClickUp space (e.g., `90153635535`).  
   - Use OAuth2 credentials linked to your ClickUp account.

2. **Add ClickUp Get Task Details Node**  
   - Type: ClickUp Node with `get` operation on task resource.  
   - Set task ID parameter to `{{$json.task_id}}` from trigger.  
   - Use same OAuth2 credentials.

3. **Add ClickUp Get Time Entry Details Node**  
   - Type: ClickUp Node with `get` operation on `timeEntry` resource.  
   - Set team ID to your team.  
   - Set time entry ID to `{{$json.data.interval_id}}` from trigger.  
   - Use same OAuth2 credentials.

4. **Add Merge Node**  
   - Type: Merge  
   - Mode: `combineBySql`  
   - Connect outputs of both ClickUp detail nodes to this node’s left and right inputs respectively.

5. **Add Code Node (Python)**  
   - Paste the provided Python code for extracting sprint and task data (logic includes date parsing, sprint mapping, hours calculations).  
   - Input: output of Merge node.  
   - Output: structured JSON including project and sprint details, hours, status, and field names for HubSpot.

6. **Add HTTP Request Node for HubSpot Search (OnProjectFolder)**  
   - Method: POST  
   - URL: `https://api.hubapi.com/crm/v3/objects/{YOUR_CUSTOM_OBJECT_ID}/search`  
   - Body: JSON payload filtering on `project_name` equal to `{{ $json.project_name }}` from code node output.  
   - Request properties: include all relevant time and project fields (total scoped, actual hours, sprint hours, status, etc.).  
   - Authentication: HTTP Header Auth with HubSpot Private App token.

7. **Add If Node (Route by Sprint Type)**  
   - Condition: check if `{{$json.numeric_sprint_id}} > 4`.  
   - True branch for additional requests sprint.  
   - False branch for sprints 1-4.

8. **Add HTTP Request Node for HubSpot Update - Additional Requests**  
   - Method: PATCH  
   - URL: `https://api.hubapi.com/crm/v3/objects/{YOUR_CUSTOM_OBJECT_ID}/{{ $json.results[0].id }}`  
   - Body: JSON that updates `actual_hours_tracked` and `total_time_remaining` by adding/subtracting the new tracked hours from code node.  
   - Authentication: HTTP Header Auth with HubSpot token.  
   - Enable retry on failure (max 2 tries).

9. **Add HTTP Request Node for HubSpot Update - Sprints 1-4**  
   - Method: PATCH  
   - URL: `https://api.hubapi.com/crm/v3/objects/{YOUR_CUSTOM_OBJECT_ID}/{{ $json.results[0].id }}`  
   - Body: JSON dynamically updating sprint-specific actual hours field (from `target_actual_sprint_field_name`) and total actual hours and remaining time.  
   - Authentication: HTTP Header Auth with HubSpot token.  
   - Enable retry on failure (max 2 tries).

10. **Connect Nodes**  
    - Trigger → ClickUp Get Task Details & ClickUp Get Time Entry Details (parallel).  
    - Both ClickUp nodes → Merge node.  
    - Merge → Code node.  
    - Code node → HubSpot Search node.  
    - HubSpot Search → Conditional If node.  
    - If node true output → HubSpot Update Additional Requests node.  
    - If node false output → HubSpot Update Sprints 1-4 node.

11. **Add Sticky Notes (Optional)**  
    - Add descriptive sticky notes summarizing the workflow purpose, instructions, and notes on sync behavior.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Real-Time ClickUp Time Tracking to HubSpot Project Sync: This workflow automates syncing ClickUp task time updates to a custom HubSpot object, ensuring project metrics (scoped vs actual hours per sprint) remain accurate and up to date. Requires ClickUp OAuth2 and HubSpot Private App credentials configured. Custom fields on ClickUp tasks (`Sprint` dropdown, `HubSpot Deal ID` short text) and HubSpot custom object with relevant properties must be set up. It triggers on ClickUp time update events, fetches task/time data, processes sprint info, and updates corresponding HubSpot project records. | Workflow overview and use case (Sticky Note4 content)                                                  |
| The workflow instantly syncs ClickUp task time updates to HubSpot, fetching task details, calculating sprint hours, and updating actual hours fields in HubSpot. For initial full sync or if data is out of sync, run the separate "ClickUp Client Project Data Stream: Automated Sync" workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Summary and important note (Sticky Note5 content)                                                     |
| Official ClickUp API documentation: https://clickup.com/api | Reference for API endpoints and authentication                                                     |
| HubSpot Custom Objects API documentation: https://developers.hubspot.com/docs/api/crm/custom-objects | Reference for API usage to query and update custom objects                                        |
| Date parsing in Python code handles multiple formats including UNIX timestamps (seconds and milliseconds) and ISO dates, with fallback and error handling to avoid workflow failure. | Code node processing detail                                                                        |
| Ensure OAuth2 tokens for ClickUp are refreshed periodically to avoid authentication failures. | Operational consideration                                                                         |
| HubSpot API tokens should be stored securely and have appropriate scopes to read and write custom objects and properties. | Security and credential setup                                                                       |

---

This document enables a developer or automation engineer to fully understand, recreate, and maintain the workflow, including anticipating potential failure modes and integration points.