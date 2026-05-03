MCP AI Agent Google Calendar - Create, Update & Manage Events

https://n8nworkflows.xyz/workflows/mcp-ai-agent-google-calendar---create--update---manage-events-3589


# MCP AI Agent Google Calendar - Create, Update & Manage Events

### 1. Workflow Overview

This workflow, titled **MCP AI Agent Google Calendar - Create, Update & Manage Events**, is designed to provide a natural language interface for managing Google Calendar events through an AI-powered agent connected via MCP (Multi-Channel Protocol). It enables users to create, update, delete, and retrieve calendar events by interpreting natural language commands such as “book a meeting tomorrow at 3pm” or “reschedule my call to Monday.” The workflow integrates Google Calendar with an AI interface (e.g., GPT-4o) via MCP, automating calendar management seamlessly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception via MCP Server Trigger:** Receives incoming MCP messages containing natural language commands.
- **1.2 Google Calendar Event Operations:** Executes calendar operations (create, update, delete, retrieve) based on parsed instructions from the AI.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception via MCP Server Trigger

- **Overview:**  
  This block listens for incoming MCP messages at a webhook endpoint. It acts as the entry point for all user commands sent through the MCP protocol, which are expected to be natural language requests related to calendar management.

- **Nodes Involved:**  
  - MCP Server Trigger

- **Node Details:**  

  **MCP Server Trigger**  
  - *Type & Role:* MCP Trigger node from the LangChain package, designed to receive MCP protocol messages via HTTP POST.  
  - *Configuration:*  
    - Webhook ID set to `23a95c27-5a91-49b1-8d99-1f64dfe04b2d`  
    - No additional parameters configured, default listening on `POST /mcp/calendar` endpoint.  
  - *Expressions/Variables:* Receives raw MCP message payloads containing user natural language commands.  
  - *Input/Output:* No input connections; outputs to all Google Calendar operation nodes via the `ai_tool` output.  
  - *Version Requirements:* Requires n8n environment with MCP Trigger node support (LangChain integration).  
  - *Potential Failures:*  
    - Webhook authentication or permission errors if MCP is not properly configured.  
    - Payload format errors if incoming messages do not conform to MCP standards.  
    - Network or timeout issues on webhook reception.  
  - *Sub-workflow:* None.

---

#### 1.2 Google Calendar Event Operations

- **Overview:**  
  This block performs the core calendar management tasks by interacting with Google Calendar via the Google Calendar Tool nodes. Each node corresponds to a specific calendar operation: creating events (with or without attendees), updating events, deleting events, and retrieving events. All these nodes receive parsed instructions from the MCP Server Trigger.

- **Nodes Involved:**  
  - Create Event  
  - Create Event with Attendee  
  - Update Event  
  - Delete Event  
  - Get Events

- **Node Details:**  

  **Create Event**  
  - *Type & Role:* Google Calendar Tool node for creating new calendar events without attendees.  
  - *Configuration:* Uses OAuth2 credentials linked to Google Calendar. Parameters like event title, start/end time, and description are expected to be dynamically populated from MCP-triggered data.  
  - *Expressions/Variables:* Inputs are mapped from MCP payload fields parsed by AI (e.g., event title, date/time).  
  - *Input/Output:* Input from MCP Server Trigger’s `ai_tool` output; no further outputs connected.  
  - *Version Requirements:* Google Calendar Tool version 1.3 or higher recommended.  
  - *Potential Failures:*  
    - OAuth token expiration or invalid credentials.  
    - Invalid date/time formats causing API errors.  
    - API quota limits or network issues.  
  - *Sub-workflow:* None.

  **Create Event with Attendee**  
  - *Type & Role:* Google Calendar Tool node for creating events including guests/attendees.  
  - *Configuration:* Similar to Create Event but includes attendee email addresses parsed from MCP input.  
  - *Expressions/Variables:* Attendee list dynamically populated from AI-parsed data.  
  - *Input/Output:* Input from MCP Server Trigger; no outputs connected.  
  - *Version Requirements:* Same as Create Event.  
  - *Potential Failures:*  
    - Invalid or missing attendee emails.  
    - Permission issues if calendar sharing settings restrict adding guests.  
  - *Sub-workflow:* None.

  **Update Event**  
  - *Type & Role:* Google Calendar Tool node to update existing events with new details or times.  
  - *Configuration:* Requires event ID and updated fields (time, title, description) from MCP input.  
  - *Expressions/Variables:* Event ID and update parameters extracted from AI-parsed MCP message.  
  - *Input/Output:* Input from MCP Server Trigger; no outputs connected.  
  - *Version Requirements:* Version 1.3 or higher.  
  - *Potential Failures:*  
    - Event ID not found or invalid.  
    - Conflicts with calendar permissions or event ownership.  
  - *Sub-workflow:* None.

  **Delete Event**  
  - *Type & Role:* Google Calendar Tool node to delete events specified by event ID.  
  - *Configuration:* Event ID provided dynamically from MCP input.  
  - *Expressions/Variables:* Event ID parsed from natural language command.  
  - *Input/Output:* Input from MCP Server Trigger; no outputs connected.  
  - *Version Requirements:* Version 1.3 or higher.  
  - *Potential Failures:*  
    - Event ID invalid or event already deleted.  
    - Permission denied errors.  
  - *Sub-workflow:* None.

  **Get Events**  
  - *Type & Role:* Google Calendar Tool node to retrieve events for a specified date or date range.  
  - *Configuration:* Date or date range parameters extracted from MCP input.  
  - *Expressions/Variables:* Date filters dynamically set from AI-parsed commands.  
  - *Input/Output:* Input from MCP Server Trigger; no outputs connected.  
  - *Version Requirements:* Version 1.3 or higher.  
  - *Potential Failures:*  
    - Invalid date formats.  
    - API quota or network issues.  
  - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                      | Input Node(s)         | Output Node(s)                       | Sticky Note                                                                                          |
|-------------------------|----------------------------------|------------------------------------|-----------------------|------------------------------------|----------------------------------------------------------------------------------------------------|
| MCP Server Trigger      | MCP Trigger (LangChain)           | Receives MCP natural language input| None                  | Create Event, Create Event with Attendee, Update Event, Delete Event, Get Events | This webhook is ready for MCP messages at POST /mcp/calendar endpoint.                              |
| Create Event            | Google Calendar Tool              | Creates calendar events without guests | MCP Server Trigger    | None                               |                                                                                                    |
| Create Event with Attendee | Google Calendar Tool            | Creates calendar events with guests | MCP Server Trigger    | None                               |                                                                                                    |
| Update Event            | Google Calendar Tool              | Updates existing calendar events   | MCP Server Trigger    | None                               |                                                                                                    |
| Delete Event            | Google Calendar Tool              | Deletes specified calendar events  | MCP Server Trigger    | None                               |                                                                                                    |
| Get Events              | Google Calendar Tool              | Retrieves calendar events by date  | MCP Server Trigger    | None                               |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Add node: MCP Server Trigger (LangChain package)  
   - Configure webhook path: `/mcp/calendar` (default)  
   - No additional parameters needed  
   - This node will receive POST requests containing MCP messages with natural language commands.

2. **Create Google Calendar Tool Nodes**  
   For each calendar operation, create a separate Google Calendar Tool node:

   - **Create Event**  
     - Node type: Google Calendar Tool  
     - Operation: Create Event  
     - Configure OAuth2 credentials for Google Calendar  
     - Map input fields (title, start time, end time, description) dynamically from MCP Server Trigger output.

   - **Create Event with Attendee**  
     - Node type: Google Calendar Tool  
     - Operation: Create Event  
     - Include attendee emails field, mapped dynamically from MCP input  
     - Use same OAuth2 credentials.

   - **Update Event**  
     - Node type: Google Calendar Tool  
     - Operation: Update Event  
     - Map event ID and updated fields from MCP input  
     - Use OAuth2 credentials.

   - **Delete Event**  
     - Node type: Google Calendar Tool  
     - Operation: Delete Event  
     - Map event ID from MCP input  
     - Use OAuth2 credentials.

   - **Get Events**  
     - Node type: Google Calendar Tool  
     - Operation: Get Events  
     - Map date or date range from MCP input  
     - Use OAuth2 credentials.

3. **Connect Nodes**  
   - Connect the `ai_tool` output of the MCP Server Trigger node to the input of each Google Calendar Tool node (Create Event, Create Event with Attendee, Update Event, Delete Event, Get Events). This allows the trigger to send parsed commands to all operation nodes.

4. **Credential Setup**  
   - Configure Google Calendar OAuth2 credentials in n8n credentials manager.  
   - Ensure the OAuth2 token has calendar read/write permissions.

5. **Parameter Mapping**  
   - Use expressions or JSON parsing to extract event details (title, date/time, attendees, event ID) from the MCP Server Trigger output.  
   - These mappings depend on the AI parsing logic external to this workflow but must be consistent with the expected MCP message format.

6. **Testing**  
   - Test the webhook by sending MCP messages with natural language commands (e.g., “schedule a meeting tomorrow at 3pm”).  
   - Verify that the correct Google Calendar operation node executes and the event is created, updated, deleted, or retrieved accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed for therapists, consultants, coaches, and small teams wanting AI-powered calendar management. | Workflow description and target audience.                                                        |
| Connect your AI interface (e.g., LangChain, Typebot) to MCP Server Trigger for natural language parsing. | Setup instructions for AI integration.                                                          |
| Chat with Amanda for customizations: [WhatsApp Chat](https://wa.me/5517991557874) (+55 17 99155-7874) | Contact for personalized workflow adaptations.                                                  |
| Requires Google Calendar OAuth2 credentials with calendar read/write access.                   | Credential setup requirement.                                                                    |
| MCP Trigger API must be enabled in your n8n environment for webhook reception.                 | Technical prerequisite for MCP Server Trigger node.                                             |
| The workflow supports GPT-4o or any AI model via MCP for natural language understanding.       | AI model flexibility note.                                                                       |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the MCP AI Agent Google Calendar workflow.