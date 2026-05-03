Context-Aware Google Calendar Management with MCP Protocol

https://n8nworkflows.xyz/workflows/context-aware-google-calendar-management-with-mcp-protocol-4231


# Context-Aware Google Calendar Management with MCP Protocol

### 1. Workflow Overview

This workflow, titled **Context-Aware Google Calendar Management with MCP Protocol**, is designed to manage Google Calendar events dynamically and contextually via the MCP (Multi-Channel Protocol) server trigger. It targets scenarios where calendar events need to be created, updated, deleted, or queried with intelligent validation of time availability and event data retrieval. The workflow integrates n8n's LangChain MCP trigger nodes, Google Calendar nodes, and custom tool workflows to deliver context-aware calendar management.

The logic is grouped into the following blocks:

- **1.1 Input Reception and Operation Routing:** Accepts incoming requests via MCP or from another workflow, normalizes inputs, and routes to the appropriate operation (create, update, delete, get event, get event data).

- **1.2 Operation Execution and Validation:** For each operation, calls sub-workflows or directly uses Google Calendar nodes to execute tasks, including validating time availability before creating or updating events.

- **1.3 Availability Checking and Conditional Event Creation:** Checks calendar availability for the requested time slot and decides whether to create the event or raise an error if the slot is busy.

- **1.4 Event Data Retrieval and Response Preparation:** Fetches event data within specified time gaps and prepares structured output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Operation Routing

**Overview:**  
This block receives external triggers either from the MCP Server or another workflow, maps input data into a normalized format, then routes the request based on the operation type.

**Nodes Involved:**  
- MCP Server Trigger  
- When Executed by Another Workflow  
- map_data  
- Operation (Switch)

**Node Details:**

- **MCP Server Trigger**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Entry point for MCP protocol-based external requests via webhook.  
  - Configuration: Listens on a specific webhook path to receive JSON payloads containing operation instructions and calendar event parameters.  
  - Inputs: Incoming HTTP webhook request.  
  - Outputs: Passes JSON data downstream.  
  - Edge cases: Webhook authentication or rate limiting issues; malformed JSON payloads.

- **When Executed by Another Workflow**  
  - Type: `n8n-nodes-base.executeWorkflowTrigger`  
  - Role: Allows this workflow to be triggered by another workflow with specific input parameters (operation, startDate, endDate, eventId, timeZone).  
  - Configuration: Defines expected input schema for downstream processing.  
  - Inputs: Trigger from parent workflow.  
  - Outputs: Forwards inputs to map_data node.  
  - Edge cases: Missing or invalid input parameters; version compatibility with n8n.

- **map_data**  
  - Type: `n8n-nodes-base.set`  
  - Role: Normalizes and formats input data fields, including converting dates to ISO strings with timezone adjustments via `DateTime.fromISO(...).setZone(...)`.  
  - Configuration: Uses expressions to map and adjust input JSON fields (`operation`, `startDate`, `endDate`, `eventId`, `timeZone`).  
  - Inputs: Data from previous trigger nodes.  
  - Outputs: Structured JSON for switch routing.  
  - Edge cases: Invalid date formats, missing timeZone causing incorrect conversions.

- **Operation (Switch)**  
  - Type: `n8n-nodes-base.switch`  
  - Role: Routes workflow execution based on the `operation` field value (e.g., getEvent, deleteEvent, createEvent, updateEvent, getEventData).  
  - Configuration: Multiple rules matching exact string values from the operation field to different output branches.  
  - Inputs: Normalized input JSON.  
  - Outputs: Downstream to operation-specific sub-workflows or nodes.  
  - Edge cases: Unrecognized operation strings or case sensitivity mismatches.

---

#### 2.2 Operation Execution and Validation

**Overview:**  
Executes the specific calendar operation by invoking either sub-workflows or Google Calendar nodes, ensuring time slot availability is validated before event creation or updates.

**Nodes Involved:**  
- validate_busy_time (Tool Workflow)  
- create_new_event (Tool Workflow)  
- update_event (Tool Workflow)  
- delete_event (Tool Workflow)  
- get_events_in_gap_time (Tool Workflow)  
- validate_availability_event (Google Calendar node)  
- delete_event1 (Google Calendar node)  
- update_calendar (Google Calendar node)  
- get_event_in_time_gap (Google Calendar node)  
- response_data_get_data (Set node)

**Node Details:**

- **validate_busy_time**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Validates if a given time gap is busy to prevent scheduling conflicts before creating or updating events.  
  - Configuration: Calls a sub-workflow named "validate_busy_time" with parameters for startDate, endDate, eventId, timeZone, and operation "getEvent".  
  - Inputs: Normalized input data.  
  - Outputs: Validation status, used downstream to permit or block event creation/update.  
  - Edge cases: API rate limits, incorrect time ranges, sub-workflow failure.

- **create_new_event**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Sub-workflow to create a new calendar event, used only after validating the time slot availability.  
  - Configuration: Passes event parameters with operation "createEvent".  
  - Edge cases: Failure due to invalid event data or conflicts, improper validation skipping.

- **update_event**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Updates existing event details such as name, startDate, or endDate after validating availability.  
  - Configuration: Passes event and time data with operation "updateEvent".  
  - Edge cases: Event not found, conflicting updates, validation failures.

- **delete_event**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Deletes an event using the eventId.  
  - Configuration: Requires eventId parameter; operation "deleteEvent".  
  - Edge cases: Deleting non-existent event, permission issues.

- **get_events_in_gap_time**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Retrieves event data within a specified time range. Intended for detailed event data retrieval beyond availability checks.  
  - Configuration: Operation "getEventData", inputs include time range and eventId.  
  - Edge cases: Large data sets, API limits.

- **validate_availability_event**  
  - Type: `n8n-nodes-base.googleCalendar`  
  - Role: Directly queries Google Calendar API to check availability for the requested time slot.  
  - Configuration: Uses Google OAuth2 credentials, queries calendar with timeMin and timeMax parameters.  
  - Inputs: Time range and calendar identifier.  
  - Outputs: Availability status.  
  - Edge cases: Authentication errors, API rate limits, invalid date formats.

- **delete_event1**  
  - Type: `n8n-nodes-base.googleCalendar`  
  - Role: Deletes an event directly via Google Calendar API using eventId.  
  - Configuration: Requires valid eventId and Google OAuth2 credentials.  
  - Edge cases: Event not found, insufficient permissions.

- **update_calendar**  
  - Type: `n8n-nodes-base.googleCalendar`  
  - Role: Updates event start and end times via Google Calendar API.  
  - Configuration: Uses eventId and updated date fields with OAuth2 credentials.  
  - Edge cases: Conflicting updates, invalid dates.

- **get_event_in_time_gap**  
  - Type: `n8n-nodes-base.googleCalendar`  
  - Role: Retrieves all events within a specified time gap.  
  - Configuration: Operates with "getAll" operation, OAuth2 credentials.  
  - Edge cases: Large result sets, pagination issues.

- **response_data_get_data**  
  - Type: `n8n-nodes-base.set`  
  - Role: Prepares structured JSON response indicating if the time is available and includes event data ID.  
  - Configuration: Uses expressions to determine availability (empty array means available) and extracts event ID.  
  - Edge cases: Empty or malformed input data.

---

#### 2.3 Availability Checking and Conditional Event Creation

**Overview:**  
Validates availability before event creation and either creates the event or stops the workflow with an error if the time slot is busy.

**Nodes Involved:**  
- check_availability_to_create (Google Calendar)  
- If (Conditional)  
- create_event (Google Calendar)  
- Stop and Error

**Node Details:**

- **check_availability_to_create**  
  - Type: `n8n-nodes-base.googleCalendar`  
  - Role: Checks if the calendar time slot is free before attempting to create a new event.  
  - Configuration: Uses startDate and endDate from mapped input to query availability.  
  - Edge cases: API failures, inconsistent time zones.

- **If**  
  - Type: `n8n-nodes-base.if`  
  - Role: Conditional branching based on the availability boolean from the previous node.  
  - Configuration: Checks if `available` property is true.  
  - Outputs: If true → create_event; else → Stop and Error.

- **create_event**  
  - Type: `n8n-nodes-base.googleCalendar`  
  - Role: Creates a new event in Google Calendar using validated time slots.  
  - Configuration: Uses startDate and endDate from input; OAuth2 credentials required.  
  - Edge cases: Creation failures, quota exceeded.

- **Stop and Error**  
  - Type: `n8n-nodes-base.stopAndError`  
  - Role: Stops workflow execution and returns a custom error message indicating time slot conflict.  
  - Configuration: Error message: "The time you are trying to create the event is busy".  
  - Edge cases: None; halts workflow on error.

---

#### 2.4 Event Data Retrieval and Response Preparation

**Overview:**  
Retrieves detailed event data within a specified time gap and prepares a response object indicating availability and event information.

**Nodes Involved:**  
- get_event_in_time_gap (Google Calendar)  
- response_data_get_data (Set)

**Node Details:**

- **get_event_in_time_gap**  
  - Type: `n8n-nodes-base.googleCalendar`  
  - Role: Retrieves all events in the specified time range from the calendar.  
  - Configuration: Uses OAuth2 credentials, timeMin and timeMax parameters.  
  - Outputs: Event list for processing.  
  - Edge cases: Pagination, API limits.

- **response_data_get_data**  
  - Type: `n8n-nodes-base.set`  
  - Role: Processes retrieved events to determine if the time is free (no events found) and formats a response JSON including event ID info.  
  - Configuration: Uses expression `{{$json.values().length === 0}}` to check availability.  
  - Edge cases: Empty dataset or unexpected data structure.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                                | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                  |
|---------------------------|----------------------------------------|-----------------------------------------------|--------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| MCP Server Trigger        | @n8n/n8n-nodes-langchain.mcpTrigger    | Entry point for MCP external requests         | -                              | map_data (indirect via executeWorkflowTrigger) |                                                                                              |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger | Entry point for internal workflow trigger      | -                              | map_data                       |                                                                                              |
| map_data                  | n8n-nodes-base.set                      | Normalizes and formats input data              | MCP Server Trigger, When Executed by Another Workflow | Operation                     |                                                                                              |
| Operation                 | n8n-nodes-base.switch                   | Routes to operation-specific branches          | map_data                       | validate_availability_event, delete_event1, check_availability_to_create, update_calendar, get_event_in_time_gap |                                                                                              |
| validate_busy_time        | @n8n/n8n-nodes-langchain.toolWorkflow  | Validates if time slot is busy                  | Operation (getEvent branch)    | MCP Server Trigger             |                                                                                              |
| create_new_event          | @n8n/n8n-nodes-langchain.toolWorkflow  | Creates new event after validation              | Operation (createEvent branch) | MCP Server Trigger             |                                                                                              |
| update_event              | @n8n/n8n-nodes-langchain.toolWorkflow  | Updates existing event after validation         | Operation (updateEvent branch) | MCP Server Trigger             |                                                                                              |
| delete_event              | @n8n/n8n-nodes-langchain.toolWorkflow  | Deletes event by eventId                         | Operation (deleteEvent branch) | MCP Server Trigger             |                                                                                              |
| get_events_in_gap_time    | @n8n/n8n-nodes-langchain.toolWorkflow  | Retrieves event data within time gap            | Operation (getEventData branch) | MCP Server Trigger             |                                                                                              |
| validate_availability_event | n8n-nodes-base.googleCalendar          | Checks availability of calendar time slot      | Operation                      | Edit Fields                   |                                                                                              |
| delete_event1             | n8n-nodes-base.googleCalendar          | Deletes event via Google Calendar API           | Operation                      | -                            |                                                                                              |
| update_calendar           | n8n-nodes-base.googleCalendar          | Updates event details                            | Operation                      | -                            |                                                                                              |
| get_event_in_time_gap     | n8n-nodes-base.googleCalendar          | Gets all events in time gap                      | Operation                      | response_data_get_data         |                                                                                              |
| response_data_get_data    | n8n-nodes-base.set                      | Prepares response with availability and event ID | get_event_in_time_gap          | -                            |                                                                                              |
| check_availability_to_create | n8n-nodes-base.googleCalendar          | Checks if slot is free before creating event    | Operation                      | If                           |                                                                                              |
| If                        | n8n-nodes-base.if                       | Conditional check on availability               | check_availability_to_create   | create_event, Stop and Error   |                                                                                              |
| create_event              | n8n-nodes-base.googleCalendar          | Creates event if time slot is available          | If (true branch)               | -                            |                                                                                              |
| Stop and Error            | n8n-nodes-base.stopAndError             | Stops workflow with error if slot busy           | If (false branch)              | -                            |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Configure webhook path (e.g., `c734cf98-3d5a-4cbd-af97-8ad2da760944`)  
   - No credentials needed.

2. **Create Execute Workflow Trigger Node**  
   - Type: `n8n-nodes-base.executeWorkflowTrigger`  
   - Define expected inputs: `operation`, `startDate`, `endDate`, `eventId`, `timeZone`.

3. **Create Set Node "map_data"**  
   - Type: `n8n-nodes-base.set`  
   - Mode: Raw  
   - JSON Output: Map inputs normalizing dates to ISO with timezone adjustment using expressions like:  
     ```javascript
     {
       "operation": "{{ $json.operation }}",
       "startDate": "{{ DateTime.fromISO($json.startDate).setZone($json.timeZone).toISO() }}",
       "endDate": "{{ DateTime.fromISO($json.endDate).setZone($json.timeZone).toISO() }}",
       "eventId": "{{ $json.eventId }}",
       "timeZone": "{{ $json.timeZone }}"
     }
     ```
   - Connect MCP Server Trigger and Execute Workflow Trigger nodes to this node.

4. **Create Switch Node "Operation"**  
   - Type: `n8n-nodes-base.switch`  
   - Add rules to route based on `operation` string exactly matching: `getEvent`, `deleteEvent`, `createEvent`, `updateEvent`, `getEventData`.  
   - Connect output to corresponding nodes or sub-workflows.

5. **Create Tool Workflows for Operations**  
   For each operation, create a tool workflow node:  
   - `validate_busy_time` (operation: getEvent)  
   - `create_new_event` (operation: createEvent)  
   - `update_event` (operation: updateEvent)  
   - `delete_event` (operation: deleteEvent)  
   - `get_events_in_gap_time` (operation: getEventData)  
   - Configure each tool workflow node to call the sub-workflow with appropriate input parameters and mapping.  
   - Connect each operation output from the Switch node to corresponding tool workflow node.

6. **Create Google Calendar Nodes for Direct API Calls**  
   - `validate_availability_event`: Query calendar availability between `startDate` and `endDate`.  
   - `delete_event1`: Delete event by `eventId`.  
   - `update_calendar`: Update event's start and end time by `eventId`.  
   - `get_event_in_time_gap`: Retrieve all events in the given time range.  
   - Set Google OAuth2 credentials for these nodes.  
   - Connect outputs from Switch node accordingly.

7. **Create Set Node "response_data_get_data"**  
   - Format output to indicate if time is available and include event ID.  
   - Use expression to check if events array is empty.

8. **Availability Check and Conditional Event Creation Branch**  
   - Create Google Calendar node `check_availability_to_create` to check if slot is free.  
   - Create If node to branch on availability boolean (`available === true`).  
   - If true, connect to `create_event` Google Calendar node to create the event.  
   - If false, connect to `Stop and Error` node with message "The time you are trying to create the event is busy".

9. **Set Credentials**  
   - Configure Google OAuth2 credentials for all Google Calendar nodes.

10. **Connect Nodes According to Workflow**  
    - Connect `map_data` to `Operation` switch node.  
    - Connect each Switch output to respective tool workflow or Google Calendar node.  
    - Connect availability checking nodes and conditional nodes as described.

11. **Test and Validate**  
    - Test each operation via triggering the MCP webhook or workflow trigger node.  
    - Verify correct event creation, updating, deletion, and event data retrieval with proper error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses the MCP Protocol for contextual multi-channel communication integration.                    | MCP Protocol integration via LangChain MCP Trigger nodes.                                       |
| Google Calendar OAuth2 credentials must be properly configured with necessary scopes (calendar.events).       | Google Cloud Console → OAuth2 setup → Google Calendar API access.                               |
| Time zone handling uses Luxon DateTime conversions inside Set nodes for accurate ISO timestamp formatting.    | Luxon library expressions in n8n Set nodes.                                                     |
| For detailed event validation, prefer using the `validate_busy_time` tool workflow before creating/updating.  | Avoid conflicts by pre-checking availability.                                                  |
| Workflow supports both external HTTP webhook triggering and internal workflow invocation for flexibility.     | Use MCP Server Trigger or Execute Workflow Trigger nodes depending on context.                   |

---

**Disclaimer:** The content provided is derived exclusively from an automated workflow constructed with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected material and manipulates only legal and public data.