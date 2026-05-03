Google Calendar MCP server for AI Agent with Dynamic Scheduling

https://n8nworkflows.xyz/workflows/google-calendar-mcp-server-for-ai-agent-with-dynamic-scheduling-3677


# Google Calendar MCP server for AI Agent with Dynamic Scheduling

### 1. Workflow Overview

This workflow, titled **Google Calendar MCP server for AI Agent with Dynamic Scheduling**, is designed to automate comprehensive Google Calendar event management through an AI-powered, dynamic parameter-driven system connected to an MCP (Model Control Plane) server. Its primary focus is enabling CRUD (Create, Read, Update, Delete) operations on Google Calendar events, availability checking with timezone awareness, and integration with AI-generated inputs for parameterization.

**Target Use Cases:**  
- Automate scheduling and booking of events based on AI recommendations.  
- Coordinate meetings and synchronize calendars with other systems.  
- Check resource (room, equipment) availability dynamically before creating events.

**Logical Blocks:**

- **1.1 AI Trigger / Input Reception:** Receive input commands and parameters dynamically from the MCP server using an MCP trigger node configured with a webhook path.  
- **1.2 Availability & Data Retrieval:** Check user availability within specified time bounds and retrieve calendar events (single or multiple) for downstream processing.  
- **1.3 Calendar Modifications:** Perform event creation, update, and deletion operations with injected AI parameters such as event ID, start/end time, description, and reminder settings.

Each block is connected directly via the MCP trigger node (acting as the gateway), allowing flexible calls for different calendar operations dynamically at runtime.

---

### 2. Block-by-Block Analysis

#### 2.1 AI Trigger / Input Reception

**Overview:**  
This block initiates the workflow by listening for HTTP requests at the specified webhook path, capturing dynamic input parameters from the MCP server or external AI agents.

**Nodes Involved:**  
- MCP_CALENDAR

**Node Details:**  

- **MCP_CALENDAR**  
  - Type: MCP Trigger (@n8n/n8n-nodes-langchain.mcpTrigger)  
  - Role: Accepts HTTP webhook trigger requests on path `/mcp/:tool/calendar`  
  - Configuration: Webhook path parameterized by `:tool` for flexible routing; uses version 1 of the MCP trigger node  
  - Inputs: External HTTP webhook calls from the MCP or AI agents  
  - Outputs: Forwards parsed parameters and authentication context to downstream nodes  
  - Edge Cases: Webhook authorization missing or invalid, malformed requests, parameter injection errors  
  - Assumes correct registration of the webhook in n8n and stable MCP server connectivity

---

#### 2.2 Availability & Data Retrieval

**Overview:**  
This block handles reading data from Google Calendar. It includes checking availability over a specified time range with timezone consideration and retrieving single or multiple calendar events.

**Nodes Involved:**  
- AVALIABILITY_CALENDAR  
- GET_ALL_CALENDAR  
- GET_CALENDAR

**Node Details:**  

- **AVALIABILITY_CALENDAR**  
  - Type: Google Calendar Tool (n8n-nodes-base.googleCalendarTool)  
  - Role: Checks calendar availability within `Start_Time` and `End_Time` bounds using AI-injected parameters  
  - Configuration:  
    - `timeMin` and `timeMax` parameters are populated with `$fromAI('Start_Time')` and `$fromAI('End_Time')` respectively  
    - Timezone fixed to `America/Sao_Paulo` for accurate local time consideration  
    - Targets a specified calendar ID: `a57a3781407f42b1ad7fe24ce76f558dc6c86fea5f349b7fd39747a2294c1654@group.calendar.google.com` (`ODONTOLOGIA`)  
  - Inputs: Data from MCP trigger node  
  - Outputs: Availability status, busy slots  
  - Potential Failures: Incorrect or missing time parameters, OAuth token expiry, API rate limits  
  - Requires Google Calendar OAuth2 credentials

- **GET_ALL_CALENDAR**  
  - Type: Google Calendar Tool  
  - Role: Retrieves all calendar events between AI-supplied `After` and `Before` times  
  - Configuration:  
    - `timeMin` and `timeMax` from AI parameters using `$fromAI('After')` and `$fromAI('Before')`  
    - `orderBy` set to `startTime`, recurring events expanded  
    - Uses same calendar ID and credentials as above  
  - Outputs: Array of calendar events matching the range  
  - Error Types: Invalid times, permissions, Google API rate limits

- **GET_CALENDAR**  
  - Type: Google Calendar Tool  
  - Role: Retrieves a single calendar event by AI-provided `Event_ID`  
  - Configuration:  
    - `eventId` from `$fromAI('Event_ID')`  
    - Fixed calendar ID and OAuth2  
  - Outputs: Data of single event  
  - Errors: Event not found, invalid ID, credential issues

---

#### 2.3 Calendar Modifications

**Overview:**  
This block performs calendar event creation, deletion, and update operations based on AI-provided dynamic input values.

**Nodes Involved:**  
- CREATE_CALENDAR  
- DELETE_CALENDAR  
- UPDATE_CALENDAR

**Node Details:**  

- **CREATE_CALENDAR**  
  - Type: Google Calendar Tool  
  - Role: Create new event using AI-injected dynamic fields: `Start`, `End`, `Description`, and reminder flags  
  - Configuration:  
    - `start` and `end` receive date/time strings via `$fromAI('Start')` and `$fromAI('End')`  
    - `description` is dynamically injected from AI using `$fromAI('Description')`  
    - Reminders controlled by AI boolean `$fromAI('Use_Default_Reminders')`  
    - Target calendar ID fixed to `ODONTOLOGIA`  
  - Outputs: Newly created event data  
  - Edge Cases: Invalid date formats, overlapping events, quota exceeded, auth errors

- **UPDATE_CALENDAR**  
  - Type: Google Calendar Tool  
  - Role: Update existing event identified by AI `Event_ID` parameter  
  - Configuration:  
    - `eventId` from `$fromAI('Event_ID')`  
    - Dynamic use of default reminders flag via AI input  
    - Currently no additional fields configured to update (empty `updateFields`) but can be extended easily  
    - Uses same calendar ID and OAuth2  
  - Outputs: Updated event info  
  - Edge Cases: Non-existent event, permissions, invalid parameters

- **DELETE_CALENDAR**  
  - Type: Google Calendar Tool  
  - Role: Delete event based on AI-injected `Event_ID`  
  - Configuration:  
    - `eventId` from `$fromAI('Event_ID')`  
    - Uses fixed calendar ID and OAuth2 credentials  
  - Outputs: Deletion success/failure response  
  - Edge Cases: Invalid ID, not found event, API errors

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                                   | Input Node(s) | Output Node(s) | Sticky Note                                                                                              |
|----------------------|----------------------------------|-------------------------------------------------|---------------|----------------|---------------------------------------------------------------------------------------------------------|
| MCP_CALENDAR         | MCP Trigger                      | Receives external webhook calls for MCP inputs  | -             | All GoogleCalendar nodes | Webhook path `/mcp/:tool/calendar` used for MCP integration and AI agent control                        |
| AVALIABILITY_CALENDAR | Google Calendar Tool             | Checks calendar availability in a time range   | MCP_CALENDAR  | -              | Timezone aware using `America/Sao_Paulo`, dynamic times from AI params                                  |
| GET_ALL_CALENDAR      | Google Calendar Tool             | Retrieves all events between two dates          | MCP_CALENDAR  | -              | Supports recurring event expansion, ordered by start time                                              |
| GET_CALENDAR          | Google Calendar Tool             | Retrieves a single event by `Event_ID`          | MCP_CALENDAR  | -              | Uses dynamic `Event_ID` injected from AI                                                               |
| CREATE_CALENDAR       | Google Calendar Tool             | Creates new calendar event with AI parameters   | MCP_CALENDAR  | -              | Injects `Start`, `End`, `Description`, and reminder flag dynamically via `$fromAI()`                    |
| UPDATE_CALENDAR       | Google Calendar Tool             | Updates calendar event by `Event_ID`            | MCP_CALENDAR  | -              | `Use_Default_Reminders` controlled by AI boolean                                                        |
| DELETE_CALENDAR       | Google Calendar Tool             | Deletes calendar event by `Event_ID`            | MCP_CALENDAR  | -              | Direct event deletion                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: `MCP_CALENDAR`  
   - Configure webhook path as `/mcp/:tool/calendar`  
   - Version: 1  
   - No credentials needed here; acts as HTTP webhook receiver.

2. **Create Google Calendar Node: Check Availability**  
   - Type: `Google Calendar Tool`  
   - Name: `AVALIABILITY_CALENDAR`  
   - Operation: `calendar/v3.freebusy` or equivalent availability check (built-in in n8n Google Calendar tool)  
   - Calendar ID: Set to your target calendar (e.g., `ODONTOLOGIA` group calendar ID)  
   - Parameters:  
     - `timeMin` = `={{ $fromAI('Start_Time', '', 'string') }}`  
     - `timeMax` = `={{ $fromAI('End_Time', '', 'string') }}`  
     - `options.timezone` = `America/Sao_Paulo`  
   - Credentials: Attach Google Calendar OAuth2 credentials.

3. **Create Google Calendar Node: Get All Events**  
   - Type: `Google Calendar Tool`  
   - Name: `GET_ALL_CALENDAR`  
   - Operation: `getAll` events, with parameters:  
     - `timeMin` = `={{ $fromAI('After', '', 'string') }}`  
     - `timeMax` = `={{ $fromAI('Before', '', 'string') }}`  
     - `orderBy` = `startTime`  
     - `recurringEventHandling` = `expand`  
     - `returnAll` = `true`  
   - Set calendar ID as before  
   - Credentials: Google Calendar OAuth2

4. **Create Google Calendar Node: Get Single Event**  
   - Type: `Google Calendar Tool`  
   - Name: `GET_CALENDAR`  
   - Operation: `get` event by ID  
   - Parameter: `eventId` = `={{ $fromAI('Event_ID', '', 'string') }}`  
   - Calendar ID and credentials as above

5. **Create Google Calendar Node: Create Event**  
   - Type: `Google Calendar Tool`  
   - Name: `CREATE_CALENDAR`  
   - Operation: `create`  
   - Parameters:  
     - `start` = `={{ $fromAI('Start', '', 'string') }}`  
     - `end` = `={{ $fromAI('End', '', 'string') }}`  
     - `additionalFields.description` = `={{ $fromAI('Description', '', 'string') }}`  
     - `useDefaultReminders` = `={{ $fromAI('Use_Default_Reminders', '', 'boolean') }}`  
   - Set calendar ID and credentials

6. **Create Google Calendar Node: Update Event**  
   - Type: `Google Calendar Tool`  
   - Name: `UPDATE_CALENDAR`  
   - Operation: `update`  
   - Parameters:  
     - `eventId` = `={{ $fromAI('Event_ID', '', 'string') }}`  
     - `useDefaultReminders` = `={{ $fromAI('Use_Default_Reminders', '', 'boolean') }}`  
     - `updateFields` can be configured as needed (currently empty)  
   - Calendar ID and credentials as above

7. **Create Google Calendar Node: Delete Event**  
   - Type: `Google Calendar Tool`  
   - Name: `DELETE_CALENDAR`  
   - Operation: `delete`  
   - Parameter: `eventId` = `={{ $fromAI('Event_ID', '', 'string') }}`  
   - Calendar ID and credentials

8. **Connect All Nodes**  
   - Connect the output of `MCP_CALENDAR` to the input of all Google Calendar nodes (`AVALIABILITY_CALENDAR`, `GET_ALL_CALENDAR`, `GET_CALENDAR`, `CREATE_CALENDAR`, `UPDATE_CALENDAR`, `DELETE_CALENDAR`)  
   - This way, any incoming request at the MCP trigger will forward data to all relevant calendar operations, to be picked/executed based on parameters

9. **Credential Setup**  
   - In n8n Settings, add Google Calendar OAuth2 credentials authorized to access the target calendar, especially the `ODONTOLOGIA` group calendar.  
   - Ensure the OAuth2 token has scopes for full calendar CRUD.

10. **Testing**  
    - Enable the workflow  
    - Send HTTP requests to `/mcp/:tool/calendar` with JSON body parameters reflecting AI-generated keys such as `Start_Time`, `End_Time`, `Event_ID`, `Description`, etc.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                              |
|----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| Extend multi-tenancy support by adding a `:userId` path parameter (e.g., `/mcp/:userId/calendar`).       | See project note to support multiple users dynamically.                     |
| Always specify option `timezone` in availability queries for accurate local time interpretation.         | Ensures correct handling in freebusy checks.                                |
| For advanced field mappings and additional Google Calendar features, refer to the official n8n docs.     | [https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/) |
| Uses MIT license, enabling free reuse and modification.                                                  | License section of the workflow.                                            |
| MCP (Model Control Plane) integration allows centralized AI agent control and dynamic parameter injection.| Key for coordinating AI-generated scheduling tasks at scale.                |
| Screenshots and UI documentation included in project assets for quick setup guidance.                    | Refer to attached images in original template deployment.                   |

---

This completes the comprehensive documentation for the **Google Calendar MCP server for AI Agent with Dynamic Scheduling** workflow.