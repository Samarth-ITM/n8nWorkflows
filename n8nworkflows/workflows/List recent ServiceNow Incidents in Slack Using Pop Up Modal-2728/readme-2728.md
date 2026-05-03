List recent ServiceNow Incidents in Slack Using Pop Up Modal

https://n8nworkflows.xyz/workflows/list-recent-servicenow-incidents-in-slack-using-pop-up-modal-2728


# List recent ServiceNow Incidents in Slack Using Pop Up Modal

### 1. Workflow Overview

This workflow automates the retrieval and reporting of recent ServiceNow incidents directly within Slack, targeting IT teams and incident management professionals. It enables users to interactively specify incident search parameters via a Slack modal, queries ServiceNow for matching incidents, and delivers formatted results back to Slack either in a selected channel or via direct message. The workflow also includes robust error handling to notify users when no incidents match or when input parameters are incomplete.

The workflow is logically divided into the following blocks:

- **1.1 Slack Event Reception and Parsing**: Captures Slack events via webhook and parses incoming data to identify the type of interaction.
- **1.2 Slack Modal Interaction**: Responds to Slack modal requests by opening a modal form for user input.
- **1.3 Modal Submission Handling and ServiceNow Query**: Closes the modal, queries ServiceNow incidents based on user input, and checks if incidents are found.
- **1.4 Incident Results Processing and Formatting**: Sorts, limits, formats, and aggregates incident data for Slack presentation.
- **1.5 Results Delivery and Notifications**: Sends formatted incident results or no-match notifications to Slack channels or users.
- **1.6 Slack Button Interaction Acknowledgment**: Sends HTTP 200 responses to Slack for button press events to prevent UI errors.

---

### 2. Block-by-Block Analysis

#### 2.1 Slack Event Reception and Parsing

- **Overview:**  
  This block receives incoming Slack events via a webhook, parses the payload to extract relevant details, and routes the message based on the interaction type.

- **Nodes Involved:**  
  - Webhook  
  - Parse Webhook  
  - Route Message

- **Node Details:**

  - **Webhook**  
    - *Type:* Webhook  
    - *Role:* Entry point for Slack events (modal submissions, button presses, etc.)  
    - *Configuration:* HTTP POST method, path set to a unique webhook ID, response mode set to use a response node  
    - *Input:* Incoming HTTP POST from Slack  
    - *Output:* Raw Slack event JSON  
    - *Edge Cases:* Missing or malformed payloads, unauthorized requests (Slack verification not shown)  

  - **Parse Webhook**  
    - *Type:* Set node  
    - *Role:* Extracts the `payload` object from the Slack event body for easier access  
    - *Configuration:* Assigns `response` variable to `body.payload` from incoming JSON  
    - *Input:* Output from Webhook node  
    - *Output:* JSON with `response` object containing Slack event details  
    - *Edge Cases:* Payload missing or malformed, expression evaluation failure  

  - **Route Message**  
    - *Type:* Switch node  
    - *Role:* Routes the workflow based on Slack interaction type or callback ID  
    - *Configuration:*  
      - Routes to "Request Modal" if `response.callback_id` equals `"search_recent_incidents"`  
      - Routes to "Submit Data" if `response.type` equals `"view_submission"`  
      - Routes to "Block Actions" if `response.type` equals `"block_actions"`  
      - Fallback output is none (no route)  
    - *Input:* Parsed Slack event JSON  
    - *Output:* Routes to one of three paths for further processing  
    - *Edge Cases:* Unknown or unexpected interaction types, missing callback IDs  

---

#### 2.2 Slack Modal Interaction

- **Overview:**  
  This block responds to Slack modal requests by acknowledging the webhook and opening a modal form for users to input incident search parameters.

- **Nodes Involved:**  
  - Respond to Slack Webhook  
  - ServiceNow Modal

- **Node Details:**

  - **Respond to Slack Webhook**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends an empty acknowledgment to Slack to prevent UI errors after modal request  
    - *Configuration:* Respond with no data  
    - *Input:* Routed from "Request Modal" output of Route Message  
    - *Output:* HTTP 200 response to Slack  
    - *Edge Cases:* Slack timeout if no response sent  

  - **ServiceNow Modal**  
    - *Type:* HTTP Request  
    - *Role:* Opens a Slack modal via Slack API to collect incident search parameters  
    - *Configuration:*  
      - POST to `https://slack.com/api/views.open`  
      - Uses Slack API credentials  
      - JSON body includes:  
        - `trigger_id` from parsed Slack event  
        - Modal title "Search SNOW Incidents"  
        - Input blocks for selecting incident priority and state (external selects)  
        - Channel selector for output destination  
        - Submit and cancel buttons  
    - *Input:* Triggered after Respond to Slack Webhook  
    - *Output:* Slack modal displayed to user  
    - *Edge Cases:* Invalid or expired `trigger_id`, Slack API errors, missing credentials  

---

#### 2.3 Modal Submission Handling and ServiceNow Query

- **Overview:**  
  After the user submits the modal, this block closes the modal popup, queries ServiceNow for incidents matching the selected priority and state, and checks if any incidents were found.

- **Nodes Involved:**  
  - Close Modal Popup  
  - ServiceNow  
  - Were Incidents Found?

- **Node Details:**

  - **Close Modal Popup**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends a response to Slack to close the modal after submission  
    - *Configuration:* Respond with no data  
    - *Input:* Routed from "Submit Data" output of Route Message  
    - *Output:* HTTP 200 response to Slack closing the modal  
    - *Edge Cases:* Slack timeout if no response sent  

  - **ServiceNow**  
    - *Type:* ServiceNow node  
    - *Role:* Queries ServiceNow incidents based on user-selected priority and state  
    - *Configuration:*  
      - Operation: `getAll` incidents  
      - Query: `incident_state={{state}}^priority={{priority}}` using values from modal submission  
      - Return all results  
      - Authentication: Basic Auth with ServiceNow credentials  
    - *Input:* Triggered after modal closure  
    - *Output:* List of matching incidents from ServiceNow  
    - *Edge Cases:* Authentication failure, API timeout, no incidents found, malformed query  

  - **Were Incidents Found?**  
    - *Type:* If node  
    - *Role:* Checks if the ServiceNow query returned any incidents  
    - *Configuration:* Condition checks if the `number` field exists in any returned incident  
    - *Input:* Output from ServiceNow node  
    - *Output:* Routes to incident processing if found, or to no-match handling if none found  
    - *Edge Cases:* Empty results, unexpected data structure  

---

#### 2.4 Incident Results Processing and Formatting

- **Overview:**  
  This block sorts incidents by most recent, limits to the first five, formats each incident into Slack Block Kit sections, concatenates the formatted details, and prepares the final Slack message.

- **Nodes Involved:**  
  - Sort by Most Recent  
  - Retain First 5 Incidents  
  - Loop Over Items  
  - Format Incident Details  
  - Concatenate Incident Details  
  - Format Slack Message

- **Node Details:**

  - **Sort by Most Recent**  
    - *Type:* Sort node  
    - *Role:* Sorts incidents descending by incident number (assumed proxy for recency)  
    - *Configuration:* Sort by `number.value` descending  
    - *Input:* Incidents from ServiceNow node (when incidents found)  
    - *Output:* Sorted incident list  
    - *Edge Cases:* Missing or malformed `number` field  

  - **Retain First 5 Incidents**  
    - *Type:* Limit node  
    - *Role:* Limits the incident list to the first 5 items for concise reporting  
    - *Configuration:* Max items = 5  
    - *Input:* Sorted incidents  
    - *Output:* Top 5 incidents  
    - *Edge Cases:* Less than 5 incidents (passes all)  

  - **Loop Over Items**  
    - *Type:* SplitInBatches node  
    - *Role:* Iterates over each incident to process individually  
    - *Configuration:* Default batch size (1)  
    - *Input:* Limited incident list  
    - *Output:* Single incident per iteration  
    - *Edge Cases:* Empty input (no iteration)  

  - **Format Incident Details**  
    - *Type:* Set node  
    - *Role:* Formats each incident into a Slack Block Kit section with details and a link to ServiceNow  
    - *Configuration:*  
      - Creates a JSON string with:  
        - Incident number linked to ServiceNow incident page  
        - Short description  
        - Opened by, date opened, severity, priority, state, category  
        - Divider block  
    - *Input:* Single incident from Loop Over Items  
    - *Output:* JSON string field `incident_details` for each incident  
    - *Edge Cases:* Missing fields, expression evaluation errors  

  - **Concatenate Incident Details**  
    - *Type:* Summarize node  
    - *Role:* Concatenates all `incident_details` strings into one block for Slack message  
    - *Configuration:* Aggregates `incident_details` field by concatenation  
    - *Input:* Formatted incidents from Loop Over Items  
    - *Output:* Single concatenated string of all incident blocks  
    - *Edge Cases:* No incidents to concatenate  

  - **Format Slack Message**  
    - *Type:* Set node  
    - *Role:* Prepares the final Slack message JSON with greeting, summary, and concatenated incident details  
    - *Configuration:*  
      - Creates a Block Kit JSON string with:  
        - Greeting section including search parameters and total incident count  
        - Divider  
        - Concatenated incident details  
        - Link to view all matching incidents in ServiceNow  
    - *Input:* Concatenated incident details  
    - *Output:* JSON string `slack_output` ready for Slack message  
    - *Edge Cases:* Expression errors, missing concatenated details  

---

#### 2.5 Results Delivery and Notifications

- **Overview:**  
  This block determines whether a Slack channel was selected for results delivery and sends the formatted incident message or no-match notification accordingly, either to the channel or as a direct message to the user.

- **Nodes Involved:**  
  - Was a Channel Selected?  
  - Channel - Send Matching Incidents  
  - DM - Send Matching Incidents  
  - No Matches - Was a Channel Selected?  
  - Channel - Notify User no Incidents Matched  
  - DM - Notify User no Incidents Matched

- **Node Details:**

  - **Was a Channel Selected?**  
    - *Type:* If node  
    - *Role:* Checks if the user selected a Slack channel in the modal for results delivery  
    - *Configuration:* Checks existence of selected channel ID in modal response  
    - *Input:* From Format Slack Message node (incidents found)  
    - *Output:* Routes to channel or DM send nodes  
    - *Edge Cases:* Missing or empty channel selection  

  - **Channel - Send Matching Incidents**  
    - *Type:* Slack node  
    - *Role:* Sends the formatted incident message to the selected Slack channel using Block Kit  
    - *Configuration:*  
      - Message type: block  
      - Channel ID from modal input  
      - Blocks JSON from `slack_output` field  
      - Uses Slack API credentials  
    - *Input:* From "Was a Channel Selected?" true branch  
    - *Output:* Slack message posted to channel  
    - *Edge Cases:* Invalid channel ID, Slack API errors  

  - **DM - Send Matching Incidents**  
    - *Type:* Slack node  
    - *Role:* Sends the formatted incident message as a direct message to the user who initiated the search  
    - *Configuration:*  
      - Message type: block  
      - User ID from Slack event user  
      - Blocks JSON from `slack_output` field  
      - Uses Slack API credentials  
    - *Input:* From "Was a Channel Selected?" false branch  
    - *Output:* Slack DM sent to user  
    - *Edge Cases:* User ID invalid, Slack API errors  

  - **No Matches - Was a Channel Selected?**  
    - *Type:* If node  
    - *Role:* Checks if a channel was selected when no incidents matched the query  
    - *Configuration:* Checks existence of selected channel ID in modal response  
    - *Input:* From "Were Incidents Found?" false branch  
    - *Output:* Routes to channel or DM no-match notification nodes  
    - *Edge Cases:* Missing or empty channel selection  

  - **Channel - Notify User no Incidents Matched**  
    - *Type:* Slack node  
    - *Role:* Sends a message to the selected Slack channel notifying no incidents matched the search criteria  
    - *Configuration:*  
      - Text includes selected priority and state from modal  
      - Channel ID from modal input  
      - Uses Slack API credentials  
    - *Input:* From "No Matches - Was a Channel Selected?" true branch  
    - *Output:* Slack message posted to channel  
    - *Edge Cases:* Invalid channel ID, Slack API errors  

  - **DM - Notify User no Incidents Matched**  
    - *Type:* Slack node  
    - *Role:* Sends a direct message to the user notifying no incidents matched the search criteria  
    - *Configuration:*  
      - Text includes selected priority and state from modal  
      - User ID from Slack event user  
      - Uses Slack API credentials  
    - *Input:* From "No Matches - Was a Channel Selected?" false branch  
    - *Output:* Slack DM sent to user  
    - *Edge Cases:* User ID invalid, Slack API errors  

---

#### 2.6 Slack Button Interaction Acknowledgment

- **Overview:**  
  This block sends HTTP 200 responses to Slack for button press events to prevent Slack UI errors.

- **Nodes Involved:**  
  - Send 200

- **Node Details:**

  - **Send 200**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends HTTP 200 response to Slack for button press events  
    - *Configuration:* Response code 200  
    - *Input:* Routed from "Block Actions" output of Route Message  
    - *Output:* HTTP 200 response to Slack  
    - *Edge Cases:* Slack timeout if no response sent  

---

### 3. Summary Table

| Node Name                               | Node Type               | Functional Role                                         | Input Node(s)               | Output Node(s)                         | Sticky Note                                                                                                                                            |
|----------------------------------------|-------------------------|---------------------------------------------------------|-----------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                                | Webhook                 | Receives Slack events                                   |                             | Parse Webhook                         | Receive and Parse Slack Event via Webhook                                                                                                             |
| Parse Webhook                          | Set                     | Extracts Slack payload                                  | Webhook                     | Route Message                        | Receive and Parse Slack Event via Webhook                                                                                                             |
| Route Message                         | Switch                  | Routes based on Slack interaction type                  | Parse Webhook               | Respond to Slack Webhook, Close Modal Popup, Send 200 | Receive and Parse Slack Event via Webhook                                                                                                             |
| Respond to Slack Webhook              | Respond to Webhook      | Acknowledges Slack modal request                         | Route Message (Request Modal) | ServiceNow Modal                    | Respond to Modal request                                                                                                                              |
| ServiceNow Modal                     | HTTP Request            | Opens Slack modal for user input                         | Respond to Slack Webhook    |                                   | Respond to Modal request                                                                                                                              |
| Close Modal Popup                    | Respond to Webhook      | Closes Slack modal after submission                      | Route Message (Submit Data) | ServiceNow                         | Close Modal and Search Service Now                                                                                                                    |
| ServiceNow                         | ServiceNow              | Queries ServiceNow incidents based on user input        | Close Modal Popup           | Were Incidents Found?               | Close Modal and Search Service Now                                                                                                                    |
| Were Incidents Found?               | If                      | Checks if ServiceNow returned any incidents             | ServiceNow                 | Sort by Most Recent, No Matches - Was a Channel Selected? | Close Modal and Search Service Now                                                                                                                    |
| Sort by Most Recent                 | Sort                    | Sorts incidents descending by number                     | Were Incidents Found?       | Retain First 5 Incidents            | Sort and format Results for block kit                                                                                                                 |
| Retain First 5 Incidents           | Limit                   | Limits incidents to first 5                              | Sort by Most Recent         | Loop Over Items                    | Sort and format Results for block kit                                                                                                                 |
| Loop Over Items                   | SplitInBatches          | Iterates over incidents                                  | Retain First 5 Incidents    | Format Incident Details, Concatenate Incident Details | Sort and format Results for block kit                                                                                                                 |
| Format Incident Details           | Set                     | Formats each incident into Slack Block Kit section       | Loop Over Items             | Loop Over Items                    | Sort and format Results for block kit                                                                                                                 |
| Concatenate Incident Details      | Summarize               | Concatenates formatted incident details                  | Loop Over Items             | Format Slack Message               | Sort and format Results for block kit                                                                                                                 |
| Format Slack Message              | Set                     | Prepares final Slack message with greeting and incidents | Concatenate Incident Details | Was a Channel Selected?            | Sort and format Results for block kit                                                                                                                 |
| Was a Channel Selected?           | If                      | Checks if user selected a Slack channel                   | Format Slack Message        | Channel - Send Matching Incidents, DM - Send Matching Incidents | Check if Slack channel selected and send Incident results in block kit                                                                                 |
| Channel - Send Matching Incidents | Slack                   | Sends incident results to selected Slack channel         | Was a Channel Selected?     |                                   | Check if Slack channel selected and send Incident results in block kit                                                                                 |
| DM - Send Matching Incidents      | Slack                   | Sends incident results as DM to user                      | Was a Channel Selected?     |                                   | Check if Slack channel selected and send Incident results in block kit                                                                                 |
| No Matches - Was a Channel Selected? | If                      | Checks if channel selected when no incidents found       | Were Incidents Found?       | Channel - Notify User no Incidents Matched, DM - Notify User no Incidents Matched | No Incidents found, respond to Slack                                                                                                                  |
| Channel - Notify User no Incidents Matched | Slack                   | Notifies selected channel no incidents matched           | No Matches - Was a Channel Selected? |                                   | No Incidents found, respond to Slack                                                                                                                  |
| DM - Notify User no Incidents Matched | Slack                   | Notifies user via DM no incidents matched                | No Matches - Was a Channel Selected? |                                   | No Incidents found, respond to Slack                                                                                                                  |
| Send 200                         | Respond to Webhook      | Sends HTTP 200 to Slack for button presses               | Route Message (Block Actions) |                                   | Respond to Slack Button Press with 200 Response                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `e03c7d39-1dce-4ac5-8db8-2b4511a85a07`)  
   - Response Mode: Use response node  

2. **Create Set Node "Parse Webhook"**  
   - Assign variable `response` to `{{$json["body"]["payload"]}}`  
   - Connect Webhook → Parse Webhook  

3. **Create Switch Node "Route Message"**  
   - Add rules:  
     - Output "Request Modal": if `response.callback_id` equals `"search_recent_incidents"`  
     - Output "Submit Data": if `response.type` equals `"view_submission"`  
     - Output "Block Actions": if `response.type` equals `"block_actions"`  
   - Connect Parse Webhook → Route Message  

4. **Create Respond to Webhook Node "Respond to Slack Webhook"**  
   - Respond with no data  
   - Connect Route Message "Request Modal" → Respond to Slack Webhook  

5. **Create HTTP Request Node "ServiceNow Modal"**  
   - Method: POST  
   - URL: `https://slack.com/api/views.open`  
   - Authentication: Slack API credentials (OAuth2 or token)  
   - Headers: Content-Type: application/json  
   - Body (JSON): Slack modal JSON with:  
     - `trigger_id` from `{{$node["Parse Webhook"].json["response"]["trigger_id"]}}`  
     - Modal title, submit, close buttons  
     - Input blocks: external selects for priority and state  
     - Channel select block for output channel  
   - Connect Respond to Slack Webhook → ServiceNow Modal  

6. **Create Respond to Webhook Node "Close Modal Popup"**  
   - Respond with no data  
   - Connect Route Message "Submit Data" → Close Modal Popup  

7. **Create ServiceNow Node "ServiceNow"**  
   - Resource: Incident  
   - Operation: Get All  
   - Query: `incident_state={{$json.response.view.state.values.state_selector.state_select.selected_option.value}}^priority={{$json.response.view.state.values.priority_selector.priority_select.selected_option.value}}`  
   - Return All: true  
   - Authentication: ServiceNow Basic Auth credentials  
   - Connect Close Modal Popup → ServiceNow  

8. **Create If Node "Were Incidents Found?"**  
   - Condition: Check if `number` field exists in any incident  
   - Connect ServiceNow → Were Incidents Found?  

9. **Create Sort Node "Sort by Most Recent"**  
   - Sort by `number.value` descending  
   - Connect Were Incidents Found? (true) → Sort by Most Recent  

10. **Create Limit Node "Retain First 5 Incidents"**  
    - Max Items: 5  
    - Connect Sort by Most Recent → Retain First 5 Incidents  

11. **Create SplitInBatches Node "Loop Over Items"**  
    - Default batch size (1)  
    - Connect Retain First 5 Incidents → Loop Over Items  

12. **Create Set Node "Format Incident Details"**  
    - Assign field `incident_details` with Slack Block Kit JSON string including:  
      - Link to incident in ServiceNow using `sys_id` and `task_effective_number`  
      - Short description, opened by, date opened, severity, priority, state, category  
      - Divider block  
    - Connect Loop Over Items → Format Incident Details  

13. **Connect Format Incident Details → Loop Over Items** (to continue looping)  
    - Connect Loop Over Items → Concatenate Incident Details (for aggregation)  

14. **Create Summarize Node "Concatenate Incident Details"**  
    - Aggregate field `incident_details` by concatenation  
    - Connect Format Incident Details → Concatenate Incident Details  

15. **Create Set Node "Format Slack Message"**  
    - Assign field `slack_output` with Slack Block Kit JSON string including:  
      - Greeting with search parameters and total incident count  
      - Divider  
      - Concatenated incident details  
      - Link to view all incidents in ServiceNow  
    - Connect Concatenate Incident Details → Format Slack Message  

16. **Create If Node "Was a Channel Selected?"**  
    - Condition: Check if selected channel ID exists in modal response  
    - Connect Format Slack Message → Was a Channel Selected?  

17. **Create Slack Node "Channel - Send Matching Incidents"**  
    - Send message to channel ID from modal input  
    - Message type: block  
    - Blocks JSON from `slack_output`  
    - Use Slack API credentials  
    - Connect Was a Channel Selected? (true) → Channel - Send Matching Incidents  

18. **Create Slack Node "DM - Send Matching Incidents"**  
    - Send message as DM to user ID from Slack event  
    - Message type: block  
    - Blocks JSON from `slack_output`  
    - Use Slack API credentials  
    - Connect Was a Channel Selected? (false) → DM - Send Matching Incidents  

19. **Create If Node "No Matches - Was a Channel Selected?"**  
    - Condition: Check if selected channel ID exists in modal response  
    - Connect Were Incidents Found? (false) → No Matches - Was a Channel Selected?  

20. **Create Slack Node "Channel - Notify User no Incidents Matched"**  
    - Send text message to selected channel indicating no incidents found with given priority and state  
    - Use Slack API credentials  
    - Connect No Matches - Was a Channel Selected? (true) → Channel - Notify User no Incidents Matched  

21. **Create Slack Node "DM - Notify User no Incidents Matched"**  
    - Send text message as DM to user indicating no incidents found with given priority and state  
    - Use Slack API credentials  
    - Connect No Matches - Was a Channel Selected? (false) → DM - Notify User no Incidents Matched  

22. **Create Respond to Webhook Node "Send 200"**  
    - Respond with HTTP 200  
    - Connect Route Message (Block Actions) → Send 200  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Slack modal interface allows users to specify priority and state, and select output channel for results.       | ![Slack Modal](https://uploads.n8n.io/templates/servicenowmodalinterface.png)                       |
| Webhook node captures Slack events and routes based on interaction type for seamless Slack integration.       | ![Slack](https://uploads.n8n.io/templates/slack.png)                                               |
| Modal submission closes modal and queries ServiceNow for incidents matching user criteria.                     | ![ServiceNow](https://uploads.n8n.io/templates/servicenow.png)                                     |
| Incident results are sorted, limited to 5, formatted for Slack Block Kit, and concatenated for messaging.      | ![n8n](https://uploads.n8n.io/templates/n8n.png)                                                   |
| If no incidents found, user is notified either in selected channel or via DM with search parameters included.  | ![Slack](https://uploads.n8n.io/templates/slack.png)                                               |
| Slack button presses are acknowledged with HTTP 200 to prevent UI errors.                                      | ![Slack](https://uploads.n8n.io/templates/slack.png)                                               |
| Setup video for Slack modal apps: [YouTube - Slack Modal Setup](https://youtu.be/z4JuK4qPJ1E)                   | Setup Instructions                                                                                  |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the "List recent ServiceNow Incidents in Slack Using Pop Up Modal" workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with this integration.