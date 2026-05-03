Telegram Appointment Scheduler Bot with Google Calendar & Sheets

https://n8nworkflows.xyz/workflows/telegram-appointment-scheduler-bot-with-google-calendar---sheets-8809


# Telegram Appointment Scheduler Bot with Google Calendar & Sheets

### 1. Workflow Overview

This workflow implements a **Telegram Appointment Scheduler Bot** integrated with **Google Calendar** and **Google Sheets**. It enables users to interact via Telegram to schedule, cancel, and check appointments efficiently. The workflow supports natural command parsing, event creation/deletion in Google Calendar, record management in Google Sheets, and conversational responses.

The workflow is organized into the following logical blocks:

- **1.1 Start Point & Command Dispatch**: Listens to Telegram messages and routes them based on command type using a Switch node.
- **1.2 Help / Initial Bot Response**: Responds to the `/start` command with usage instructions.
- **1.3 Schedule Appointment ("Agendar")**: Parses appointment requests, checks calendar availability, creates events in Google Calendar, appends data to Google Sheets, and confirms scheduling.
- **1.4 Cancel Appointment ("Cancelar")**: Parses cancellation requests, verifies existing appointments, deletes corresponding Google Calendar events and Google Sheets records, and confirms cancellation.
- **1.5 List Appointments ("Citas")**: Retrieves and filters user appointments from Google Sheets, returning upcoming appointments or a no-appointments message.
- **1.6 Default Response**: Handles unrecognized messages with a fallback informative response.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Start Point & Command Dispatch

- **Overview:**  
  Initiates the workflow by monitoring Telegram messages and routing based on user commands using a Switch node.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Switch  

- **Node Details:**  

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point webhook to capture Telegram user messages (updates of type "message").  
    - *Config:* Authenticated with Telegram API credentials. Listens for new messages.  
    - *Input:* Incoming Telegram webhook calls.  
    - *Output:* Message JSON payloads to Switch node.  
    - *Edge Cases:* Telegram API connectivity issues, webhook misconfiguration.

  - **Switch**  
    - *Type:* Switch (conditional routing)  
    - *Role:* Parses the message text to route commands: `/start`, messages starting with "agendar ", "cancelar ", "citas ", or fallback ("extra").  
    - *Config:* Multiple string equals or startsWith conditions on `$json.message.text`.  
    - *Input:* Telegram Trigger output.  
    - *Output:* Directs flow to relevant handling blocks for help, scheduling, cancelling, listing, or default response.  
    - *Edge Cases:* Case sensitivity or unexpected command formats leading to fallback route.  

---

#### 2.2 Help / Initial Bot Response

- **Overview:**  
  Provides introductory instructions and usage guidelines when user sends `/start`.

- **Nodes Involved:**  
  - Respuesta Bot Inicial (Telegram)  

- **Node Details:**  

  - **Respuesta Bot Inicial**  
    - *Type:* Telegram node (message sender)  
    - *Role:* Sends a welcome/help message with detailed instructions on command formats (`agendar`, `cancelar`, `citas`).  
    - *Config:* Markdown parse mode, chatId taken from incoming message.  
    - *Input:* Switch node route for `/start`.  
    - *Output:* None (end of this branch).  
    - *Edge Cases:* Telegram send message failure, invalid chatId.  

---

#### 2.3 Schedule Appointment ("Agendar") Module

- **Overview:**  
  Parses scheduling commands, checks calendar availability, creates Google Calendar events with 1-hour duration, stores appointment in Google Sheets, and confirms to user.

- **Nodes Involved:**  
  - Code in JavaScript (parse and format date/time)  
  - Get many events (check availability)  
  - If (availability decision)  
  - Create an event (Google Calendar)  
  - Agregar cita en Google Sheets  
  - Respuesta de Cita Agendada (Telegram)  
  - Respuesta Hora no disponible (Telegram)  

- **Node Details:**  

  - **Code in JavaScript**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Parses the "agendar" command message, extracts date, time, and client name; generates ISO start and end datetime with 1-hour slot using timezone "America/Guayaquil".  
    - *Key Logic:* Uses Luxon DateTime.fromFormat and plus(1 hour).  
    - *Input:* Switch output for "agendar" commands.  
    - *Output:* JSON with `name`, `summary`, `startDateTime`, `endDateTime`.  
    - *Edge Cases:* Invalid date/time format, missing parameters, timezone mismatches.

  - **Get many events**  
    - *Type:* Google Calendar node (getAll events)  
    - *Role:* Queries existing events in the calendar during requested slot to detect conflicts.  
    - *Config:* Uses `startDateTime` and `endDateTime` from previous node as timeMin and timeMax; calendar ID specified.  
    - *Input:* From Code node.  
    - *Output:* Array of events found.  
    - *Edge Cases:* API limits, connectivity, permission errors.

  - **If**  
    - *Type:* If node  
    - *Role:* Checks if the returned event list is empty (no conflicts).  
    - *Input:* Output from Get many events.  
    - *Output:*  
      - True branch (no conflicts): Continue to create event.  
      - False branch: Send "hour not available" message.  
    - *Edge Cases:* Empty data detection reliability.

  - **Create an event**  
    - *Type:* Google Calendar node (create event)  
    - *Role:* Creates the appointment event on Google Calendar with summary, reminders, conference data (Google Meet link).  
    - *Config:* Use start/end datetime and summary from the Code node; reminders set to 30 min popup; calendar OAuth credentials.  
    - *Input:* If node's true branch.  
    - *Output:* Event creation confirmation and event ID.  
    - *Edge Cases:* API failures, quota limits.

  - **Agregar cita en Google Sheets**  
    - *Type:* Google Sheets node (append or update row)  
    - *Role:* Logs the appointment details (event ID, ISO date, formatted date, client name) in Google Sheets for record-keeping.  
    - *Config:* Document and sheet IDs fixed; columns mapped with event data and formatted date/time string.  
    - *Input:* Output from Create an event.  
    - *Output:* Confirmation of row append/update.  
    - *Edge Cases:* Sheet access permissions, write conflicts.

  - **Respuesta de Cita Agendada**  
    - *Type:* Telegram node (message sender)  
    - *Role:* Sends a confirmation message to the user with appointment details in Spanish, formatted nicely with date/time localized in Spanish.  
    - *Config:* Markdown parse mode, chatId from initial Telegram message.  
    - *Input:* From Google Sheets node.  
    - *Output:* None (end of flow).  
    - *Edge Cases:* Telegram API send failures.

  - **Respuesta Hora no disponible**  
    - *Type:* Telegram node (message sender)  
    - *Role:* Sends a polite refusal message if the requested slot is already booked.  
    - *Config:* Markdown parse mode, chatId from Telegram Trigger.  
    - *Input:* If node's false branch.  
    - *Output:* None (end of flow).  
    - *Edge Cases:* Telegram API failures.

---

#### 2.4 Cancel Appointment ("Cancelar") Module

- **Overview:**  
  Parses cancellation requests, verifies event existence, deletes event from Google Calendar and corresponding row from Google Sheets, then confirms cancellation.

- **Nodes Involved:**  
  - Code in JavaScript1 (parse cancellation command)  
  - Get many events1 (search for appointment)  
  - If1 (check if appointment exists)  
  - Delete an event (Google Calendar)  
  - Buscar cita agendada (Google Sheets search)  
  - Delete rows or columns from sheet  
  - Respuesta Cita Cancelada (Telegram)  
  - Respuesta Cita no encontrada (Telegram)  

- **Node Details:**  

  - **Code in JavaScript1**  
    - *Type:* Code node  
    - *Role:* Parses "cancelar" command text, extracts date, time, and client name; formats start and end ISO datetime for event search.  
    - *Input:* Switch output for "cancelar" commands.  
    - *Output:* JSON with `name`, `summary` (title for search), `startDateTime`, `endDateTime`.  
    - *Edge Cases:* Input format errors, timezone.

  - **Get many events1**  
    - *Type:* Google Calendar node (getAll events)  
    - *Role:* Retrieves events matching the cancellation time window.  
    - *Input:* From Code in JavaScript1.  
    - *Output:* List of events found.  
    - *Edge Cases:* API errors.

  - **If1**  
    - *Type:* If node  
    - *Role:* Checks if event list is empty.  
    - *Output:*  
      - True branch (empty): Send "appointment not found" message.  
      - False branch: Proceed to delete event.  
    - *Edge Cases:* Reliability of empty check.

  - **Delete an event**  
    - *Type:* Google Calendar node (delete event)  
    - *Role:* Deletes the matched appointment event from Google Calendar using event ID.  
    - *Input:* If1 false branch.  
    - *Output:* Confirmation of deletion.  
    - *Edge Cases:* Permissions, event no longer existing.

  - **Buscar cita agendada**  
    - *Type:* Google Sheets node (search rows)  
    - *Role:* Searches Google Sheets for the row with matching event ID to delete.  
    - *Input:* Output from Delete an event node (event ID).  
    - *Output:* Row data with row number.  
    - *Edge Cases:* Sheet access, multiple matches.

  - **Delete rows or columns from sheet**  
    - *Type:* Google Sheets node (delete operation)  
    - *Role:* Deletes the row corresponding to the cancelled appointment.  
    - *Input:* Row number from Buscar cita agendada.  
    - *Edge Cases:* Index errors, permission issues.

  - **Respuesta Cita Cancelada**  
    - *Type:* Telegram node  
    - *Role:* Confirms the cancellation to the user via Telegram.  
    - *Input:* From Delete rows node.  
    - *Edge Cases:* Telegram API failures.

  - **Respuesta Cita no encontrada**  
    - *Type:* Telegram node  
    - *Role:* Informs user if no appointment matches cancellation request.  
    - *Input:* If1 true branch.  
    - *Edge Cases:* Telegram send issues.

---

#### 2.5 List Appointments ("Citas") Module

- **Overview:**  
  Retrieves all appointments for a user from Google Sheets, filters for future appointments, and sends a summary message.

- **Nodes Involved:**  
  - Code in JavaScript2 (parse "citas" command)  
  - Buscar citas de cliente (Google Sheets search)  
  - If2 (check if results empty)  
  - Respuesta Citas no encontradas (Telegram)  
  - Code in JavaScript3 (filter future appointments and build message)  
  - Respuesta de Cita Agendada1 (Telegram)  

- **Node Details:**  

  - **Code in JavaScript2**  
    - *Type:* Code node  
    - *Role:* Extracts client name from the "citas" command text and prepares a search term.  
    - *Input:* Switch output for "citas" commands.  
    - *Output:* JSON with `name` and `searchTerm`.  
    - *Edge Cases:* Missing name parameter.

  - **Buscar citas de cliente**  
    - *Type:* Google Sheets node (search rows)  
    - *Role:* Finds all rows where "Nombre Cliente" matches the name provided.  
    - *Input:* From Code in JavaScript2.  
    - *Output:* Appointment rows for the client.  
    - *Edge Cases:* No matches found, large result sets.

  - **If2**  
    - *Type:* If node  
    - *Role:* Checks if any appointments were found.  
    - *Output:*  
      - True branch (empty): Send no appointments message.  
      - False branch: Process appointments.  
    - *Edge Cases:* Empty detection.

  - **Respuesta Citas no encontradas**  
    - *Type:* Telegram node  
    - *Role:* Sends a message indicating no future appointments for the user.  
    - *Input:* If2 true branch.  
    - *Edge Cases:* Telegram API issues.

  - **Code in JavaScript3**  
    - *Type:* Code node  
    - *Role:* Filters appointments to only those in the future and constructs a formatted message listing them. If none are in the future, sends a special message.  
    - *Input:* If2 false branch (appointments found).  
    - *Output:* JSON containing the message text.  
    - *Edge Cases:* Date parsing errors, empty filtered lists.

  - **Respuesta de Cita Agendada1**  
    - *Type:* Telegram node  
    - *Role:* Sends the constructed list of upcoming appointments to the user.  
    - *Input:* From Code in JavaScript3.  
    - *Edge Cases:* Telegram API failures.

---

#### 2.6 Default Response

- **Overview:**  
  Handles any message that does not match defined commands, providing usage hints.

- **Nodes Involved:**  
  - Respuesta Bot Defecto (Telegram)  

- **Node Details:**  

  - **Respuesta Bot Defecto**  
    - *Type:* Telegram node  
    - *Role:* Sends a polite message indicating the command was not understood and reminds user of available commands and formats.  
    - *Config:* Markdown parse mode, chatId from incoming message.  
    - *Input:* Switch fallback branch.  
    - *Edge Cases:* Telegram API delivery errors.

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                                            | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                     |
|------------------------------|----------------------------|------------------------------------------------------------|-----------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger           | Entry point listening to Telegram messages                  | -                           | Switch                          | ## Start Point \nTelegram Trigger to listen the new messages from the users                     |
| Switch                      | Switch                    | Routes commands based on message text                       | Telegram Trigger            | Respuesta Bot Inicial, Code in JS (agendar), Code in JS1 (cancelar), Code in JS2 (citas), Respuesta Bot Defecto | ## Commands Options \n**Switch** for control the command send for the user (/help, agendar, cancelar, citas) |
| Respuesta Bot Inicial        | Telegram                  | Sends welcome/help instructions for `/start`                | Switch                     | -                                | ## Help Response\nInformation about the Bot's functions                                        |
| Code in JavaScript           | Code                      | Parses "agendar" command, formats date/time for scheduling  | Switch                     | Get many events                  | ## "Agendar" Module \nFlow to schedule a appointment with Telegram Bot and register on Google Calendar and Google Sheets |
| Get many events              | Google Calendar           | Checks calendar availability for requested slot             | Code in JavaScript          | If                             |                                                                                                |
| If                          | If                        | Decision node: slot availability                             | Get many events             | Create an event, Respuesta Hora no disponible |                                                                                                |
| Create an event              | Google Calendar           | Creates calendar event                                       | If                         | Agregar cita en Google Sheets   |                                                                                                |
| Agregar cita en Google Sheets| Google Sheets             | Logs appointment details in Google Sheets                    | Create an event             | Respuesta de Cita Agendada      |                                                                                                |
| Respuesta de Cita Agendada   | Telegram                  | Confirms appointment scheduling to user                      | Agregar cita en Google Sheets| -                              |                                                                                                |
| Respuesta Hora no disponible | Telegram                  | Notifies user if requested slot is unavailable               | If                         | -                              |                                                                                                |
| Code in JavaScript1          | Code                      | Parses "cancelar" command, formats date/time for search      | Switch                     | Get many events1                 | ## "Cancelar" Module \nFlow to cancel a appointment with Telegram Bot and register this change deleting on Google Calendar and Google Sheets |
| Get many events1             | Google Calendar           | Searches calendar for matching appointment                   | Code in JavaScript1         | If1                            |                                                                                                |
| If1                         | If                        | Checks if appointment exists                                 | Get many events1            | Respuesta Cita no encontrada, Delete an event |                                                                                                |
| Delete an event              | Google Calendar           | Deletes appointment event                                    | If1                        | Buscar cita agendada            |                                                                                                |
| Buscar cita agendada         | Google Sheets             | Finds appointment row in Google Sheets                       | Delete an event             | Delete rows or columns from sheet |                                                                                                |
| Delete rows or columns from sheet | Google Sheets         | Deletes appointment row from sheet                           | Buscar cita agendada        | Respuesta Cita Cancelada        |                                                                                                |
| Respuesta Cita Cancelada     | Telegram                  | Confirms cancellation to user                                | Delete rows or columns from sheet | -                            |                                                                                                |
| Respuesta Cita no encontrada | Telegram                  | Notifies user if no appointment matches cancellation request| If1 (true branch)           | -                              |                                                                                                |
| Code in JavaScript2          | Code                      | Parses "citas" command, extracts client name                 | Switch                     | Buscar citas de cliente          | ## "Citas" Module \nFlow to check all the appointments from a user                             |
| Buscar citas de cliente      | Google Sheets             | Retrieves all appointments for the client                    | Code in JavaScript2         | If2                            |                                                                                                |
| If2                         | If                        | Checks if any appointments found                             | Buscar citas de cliente     | Respuesta Citas no encontradas, Code in JavaScript3 |                                                                                                |
| Respuesta Citas no encontradas | Telegram                | Notifies user when no future appointments are found         | If2 (true branch)           | -                              |                                                                                                |
| Code in JavaScript3          | Code                      | Filters future appointments and constructs response message | If2 (false branch)          | Respuesta de Cita Agendada1     |                                                                                                |
| Respuesta de Cita Agendada1  | Telegram                  | Sends upcoming appointments list to user                    | Code in JavaScript3         | -                              |                                                                                                |
| Respuesta Bot Defecto        | Telegram                  | Sends fallback message for unrecognized commands             | Switch (fallback)           | -                              | ## Default Response \nBot's Response for unexpected messages                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Credentials: Connect your Telegram Bot API credentials.  
   - Parameters: Listen for `message` updates only.

2. **Create Switch node**  
   - Connect Telegram Trigger output to Switch input.  
   - Define rules:  
     - `/start` exact match  
     - Starts with `agendar `  
     - Starts with `cancelar `  
     - Starts with `citas `  
     - Fallback route for others.

3. **Setup Help / Initial Bot Response**  
   - Create Telegram node named "Respuesta Bot Inicial".  
   - Connect `/start` output from Switch here.  
   - Configure text with detailed usage instructions (Markdown enabled).  
   - Use chatId from Telegram Trigger input.

4. **Build "Agendar" Module**  
   - Create JavaScript Code node, parse message for date, time, name; compute start/end ISO datetime with timezone "America/Guayaquil".  
   - Connect Switch "agendar" output to this node.  
   - Create Google Calendar node "Get many events" to check availability between start and end datetime.  
   - Connect JavaScript Code node to this.  
   - Add If node to check if event list is empty (no conflicts).  
   - True branch: create Google Calendar event with 1-hour slot, popup reminder 30 min, conference data enabled.  
   - Connect output to Google Sheets node "Agregar cita en Google Sheets": append or update row with event ID, ISO date, formatted date/time, client name.  
   - Connect to Telegram node "Respuesta de Cita Agendada" sending confirmation message with localized Spanish date/time and client name.  
   - False branch: connect to Telegram node "Respuesta Hora no disponible" notifying user slot is taken.

5. **Build "Cancelar" Module**  
   - Create JavaScript Code node parsing cancellation command for date, time, name, and preparation of search parameters.  
   - Connect Switch "cancelar" output here.  
   - Create Google Calendar node "Get many events1" to search events in the time window.  
   - Connect Code node output to this.  
   - Add If node "If1" to check for empty event list.  
   - True branch: connect to Telegram node "Respuesta Cita no encontrada".  
   - False branch: connect to Google Calendar node "Delete an event" with eventId from event found.  
   - Connect output to Google Sheets node "Buscar cita agendada" to find matching row by event ID.  
   - Connect to Google Sheets node "Delete rows or columns from sheet" to delete that row by index.  
   - Connect to Telegram node "Respuesta Cita Cancelada" confirming deletion.

6. **Build "Citas" Module**  
   - Create JavaScript Code node parsing "citas" command to extract client name.  
   - Connect Switch "citas" output here.  
   - Add Google Sheets node "Buscar citas de cliente" to find all rows matching client name.  
   - Add If node "If2" to check if results are empty.  
   - True branch: connect to Telegram node "Respuesta Citas no encontradas" informing no upcoming appointments.  
   - False branch: connect to JavaScript Code node "Code in JavaScript3" that filters future appointments and constructs a message.  
   - Connect to Telegram node "Respuesta de Cita Agendada1" to send the list.

7. **Build Default Response**  
   - Connect Switch fallback output to Telegram node "Respuesta Bot Defecto" sending usage reminder for unknown commands.

8. **Credentials Setup**  
   - Telegram API credentials linked to all Telegram nodes.  
   - Google OAuth2 credentials linked to Google Calendar and Google Sheets nodes.  
   - Set calendar ID to your target calendar.  
   - Set Google Sheets document and sheet IDs to your target spreadsheet.

9. **Timezone and Formatting**  
   - Use timezone "America/Guayaquil" in all date/time parsing and formatting.  
   - Use Luxon library for date operations in Code nodes.  
   - Use Markdown parse mode in Telegram messages for formatting.

10. **Testing and Validation**  
    - Test each command with expected and edge-case inputs.  
    - Verify Google Calendar event creation, deletion, and data integrity in Google Sheets.  
    - Ensure Telegram messages arrive correctly with proper formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow uses **Luxon** library for date-time operations inside Code nodes (`DateTime.fromFormat`, etc.). | Native in n8n code nodes with JavaScript.                                                               |
| Telegram commands must be exact or start with specified keywords for correct routing.                           | Switch node conditions are case-sensitive string comparisons.                                            |
| Google Calendar event creation includes automatic Google Meet conferencing link via `conferenceDataUi`.       | Requires appropriate Google Calendar API permissions.                                                   |
| Google Sheets are used as a secondary data store to maintain appointment records with columns: id, date, name. | Document ID and sheet name are hardcoded; update for your environment.                                  |
| Reminder is configured as a 30-minute popup in Google Calendar events.                                         | Adjust in "Create an event" node under remindersUi if needed.                                           |
| Markdown parse mode is used in Telegram messages for better formatting and links.                              | Be cautious with user input to avoid markdown injection issues.                                         |
| The bot responds in Spanish, reflecting the target audience language and date formatting localized accordingly.| Date formatting uses Spanish locale with Luxon in code nodes and Telegram messages.                      |
| Workflow inactive by default; activate after configuring credentials and testing.                              |                                                                                                         |
| Telegram Bot token and Google OAuth2 credentials must be set up before running the workflow.                   | Credential setup required in n8n credentials manager.                                                   |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.