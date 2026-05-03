Bitrix24 Task Form Widget Application Workflow with Webhook Integration

https://n8nworkflows.xyz/workflows/bitrix24-task-form-widget-application-workflow-with-webhook-integration-2974


# Bitrix24 Task Form Widget Application Workflow with Webhook Integration

### 1. Workflow Overview

This workflow implements a Bitrix24 Task Form Widget Application integrated via webhook. It extends Bitrix24 task interfaces by adding a custom widget tab that displays detailed task information and supports seamless interaction. The workflow handles incoming webhook requests from Bitrix24, manages authentication tokens, processes application installation events, registers widget placements, and retrieves and formats task data for display. It also persistently stores configuration settings to maintain state across sessions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Credential Extraction:** Receives webhook POST requests from Bitrix24 and extracts authentication and domain credentials.
- **1.2 Event Type Determination:** Determines if the incoming request is an installation event or a regular task view event.
- **1.3 Installation Handling:** Manages installation events, including registration of widget placement and saving installation settings.
- **1.4 Settings Management:** Reads, processes, and validates persistent settings from storage.
- **1.5 Task Data Retrieval & Formatting:** Fetches task details from Bitrix24 REST API and formats them into an HTML view.
- **1.6 Response Handling:** Sends appropriate HTML responses back to Bitrix24, including installation confirmations, task views, or error messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Credential Extraction

- **Overview:**  
  This block receives incoming webhook requests from Bitrix24 and extracts necessary credentials and parameters for further processing.

- **Nodes Involved:**  
  - Bitrix24 Handler  
  - Extract Credentials

- **Node Details:**

  - **Bitrix24 Handler**  
    - Type: Webhook  
    - Role: Entry point for POST requests from Bitrix24 task widget.  
    - Configuration: Listens on path `bitrix24/widgethandler.php` with POST method, responds via response nodes.  
    - Inputs: HTTP POST request from Bitrix24.  
    - Outputs: JSON containing request body, query parameters, headers, and webhook URL.  
    - Edge Cases: Invalid or malformed requests, missing required parameters.

  - **Extract Credentials**  
    - Type: Set  
    - Role: Extracts and assigns authentication tokens, domain, client credentials, and file paths from incoming request data.  
    - Configuration: Uses expressions to extract `CLIENT_ID`, `CLIENT_SECRET`, `domain`, `access_token`, `refresh_token`, `application_token`, `expires_in`, and constructs `client_endpoint` URL. Also sets a file path for settings storage.  
    - Inputs: Output from Bitrix24 Handler.  
    - Outputs: JSON with extracted credentials and parameters.  
    - Edge Cases: Missing or undefined tokens or domain parameters may cause downstream failures.

#### 2.2 Event Type Determination

- **Overview:**  
  Determines whether the incoming request is related to application installation or a regular widget invocation.

- **Nodes Involved:**  
  - Check Event Type  
  - Is Installation?

- **Node Details:**

  - **Check Event Type**  
    - Type: Code  
    - Role: Parses request JSON to identify if the event is `ONAPPINSTALL` or if the placement is `DEFAULT` (indicating installation). Also checks if installation is finished.  
    - Configuration: JavaScript code inspects `body.event` and `body.PLACEMENT` fields, sets boolean flags `isInstallation` and `isInstallationFinished`.  
    - Inputs: Extract Credentials output.  
    - Outputs: JSON enriched with installation flags.  
    - Edge Cases: Missing or malformed event or placement fields.

  - **Is Installation?**  
    - Type: If  
    - Role: Branches workflow based on whether the request is an installation event.  
    - Configuration: Checks if `isInstallation` is true.  
    - Inputs: Output from Check Event Type.  
    - Outputs: Two branches: true (installation) and false (non-installation).  
    - Edge Cases: Incorrect flag evaluation may misroute processing.

#### 2.3 Installation Handling

- **Overview:**  
  Handles the installation process by saving settings, registering widget placement, and responding to Bitrix24 with installation status.

- **Nodes Involved:**  
  - If Installation finished  
  - Installation finished Response  
  - Register Placement  
  - Set Settings Data  
  - Create Settings File  
  - Save Installation Settings  
  - Merge Installation info  
  - Installation Response

- **Node Details:**

  - **If Installation finished**  
    - Type: If  
    - Role: Checks if installation has been fully completed (`isInstallationFinished` flag).  
    - Inputs: Is Installation? node output.  
    - Outputs: True branch for finished installation, false branch for ongoing installation.  
    - Edge Cases: Flag misinterpretation.

  - **Installation finished Response**  
    - Type: Respond to Webhook  
    - Role: Sends a simple HTML confirmation that installation is fully finished.  
    - Configuration: HTTP 200 with HTML body indicating completion.  
    - Inputs: True branch of If Installation finished.  
    - Outputs: HTTP response to Bitrix24.

  - **Register Placement**  
    - Type: HTTP Request  
    - Role: Calls Bitrix24 REST API to bind the widget placement (`TASK_VIEW_TAB`) with the webhook handler URL.  
    - Configuration: POST to `https://{domain}/rest/placement.bind` with auth token and parameters `PLACEMENT`, `HANDLER`, and `TITLE`.  
    - Inputs: False branch of If Installation finished.  
    - Outputs: API response.  
    - Edge Cases: Auth failures, network errors, invalid parameters.

  - **Set Settings Data**  
    - Type: Set  
    - Role: Prepares JSON object with installation tokens, domain, client credentials, and timestamp for persistent storage.  
    - Inputs: False branch of If Installation finished.  
    - Outputs: JSON with `data` object and `settingsFilePath`.  
    - Edge Cases: Missing tokens or incorrect data structure.

  - **Create Settings File**  
    - Type: Convert to File  
    - Role: Converts JSON settings data to a JSON file format for storage.  
    - Inputs: Output of Set Settings Data.  
    - Outputs: File object representing settings JSON.  
    - Edge Cases: File path issues.

  - **Save Installation Settings**  
    - Type: Read/Write File  
    - Role: Writes the settings JSON file to disk at the specified path.  
    - Inputs: Output of Create Settings File.  
    - Outputs: Confirmation of file write.  
    - Edge Cases: File system permissions, disk space.

  - **Merge Installation info**  
    - Type: Merge  
    - Role: Combines installation API response and file write confirmation for final response.  
    - Inputs: Outputs from Save Installation Settings and Register Placement.  
    - Outputs: Combined JSON.  

  - **Installation Response**  
    - Type: Respond to Webhook  
    - Role: Sends HTML response indicating installation has finished and widget is registered.  
    - Inputs: Output of Merge Installation info.  
    - Outputs: HTTP 200 response with HTML body including Bitrix24 JS API call to finalize installation.  
    - Edge Cases: Response formatting errors.

#### 2.4 Settings Management

- **Overview:**  
  Reads and processes stored installation settings, validates their presence, and extracts task ID from placement options.

- **Nodes Involved:**  
  - Read Installation Settings  
  - Extract Installation Settings  
  - Merge request data with installation settings  
  - Process Settings  
  - Has Valid Settings?

- **Node Details:**

  - **Read Installation Settings**  
    - Type: Read/Write File  
    - Role: Reads the stored settings JSON file from disk.  
    - Inputs: Non-installation branch from Is Installation? node.  
    - Outputs: Raw file content as JSON string.  
    - Edge Cases: File not found, read errors.

  - **Extract Installation Settings**  
    - Type: Extract From File  
    - Role: Parses JSON content from the file into usable JSON object.  
    - Inputs: Output of Read Installation Settings.  
    - Outputs: Parsed JSON object with settings.  
    - Edge Cases: Invalid JSON format.

  - **Merge request data with installation settings**  
    - Type: Merge  
    - Role: Combines incoming request data with stored installation settings for unified processing.  
    - Inputs: Extract Installation Settings and output from Is Installation? node.  
    - Outputs: Combined JSON.

  - **Process Settings**  
    - Type: Function  
    - Role: Parses settings data and extracts `taskId` from `PLACEMENT_OPTIONS` if available. Returns success flag and original request data.  
    - Inputs: Output of Merge request data with installation settings.  
    - Outputs: JSON with settings, `taskId`, success flag, and original request.  
    - Edge Cases: JSON parse errors on `PLACEMENT_OPTIONS`.

  - **Has Valid Settings?**  
    - Type: If  
    - Role: Checks if settings were successfully processed (`success` flag).  
    - Inputs: Output of Process Settings.  
    - Outputs: True branch proceeds to task data retrieval; false branch triggers error response.  
    - Edge Cases: False negatives due to parsing errors.

#### 2.5 Task Data Retrieval & Formatting

- **Overview:**  
  Retrieves task details from Bitrix24 REST API and formats the data into an HTML table for display in the widget.

- **Nodes Involved:**  
  - Get Task Data  
  - Format Task Data

- **Node Details:**

  - **Get Task Data**  
    - Type: HTTP Request  
    - Role: Calls Bitrix24 REST API endpoint `tasks.task.get` to fetch task details using auth token and domain.  
    - Configuration: POST request with JSON body containing `PLACEMENT_OPTIONS` (which includes `taskId`).  
    - Inputs: True branch of Has Valid Settings? node.  
    - Outputs: JSON containing task data under `result.task`.  
    - Edge Cases: API errors, expired tokens, invalid task IDs.

  - **Format Task Data**  
    - Type: Function  
    - Role: Converts task JSON data into an HTML table string for rendering in the widget. Handles arrays and null values gracefully.  
    - Inputs: Output of Get Task Data.  
    - Outputs: JSON with `taskHtml` containing HTML string.  
    - Edge Cases: Missing or malformed task data.

#### 2.6 Response Handling

- **Overview:**  
  Sends appropriate HTML responses back to Bitrix24 depending on the workflow branch: task view, installation confirmation, or error.

- **Nodes Involved:**  
  - Task View Response  
  - Error Response  
  - Installation Response  
  - Installation finished Response

- **Node Details:**

  - **Task View Response**  
    - Type: Respond to Webhook  
    - Role: Sends the formatted task HTML wrapped in a full HTML page to Bitrix24 for display in the widget tab.  
    - Configuration: HTTP 200 with `Content-Type: text/html`. Includes CSS and jQuery references.  
    - Inputs: Output of Format Task Data.  
    - Outputs: HTTP response.  
    - Edge Cases: Rendering issues if HTML is malformed.

  - **Error Response**  
    - Type: Respond to Webhook  
    - Role: Sends an error HTML page indicating missing settings or expired tokens, prompting reinstallation.  
    - Inputs: False branch of Has Valid Settings? node.  
    - Outputs: HTTP 200 with error message HTML.  
    - Edge Cases: None.

  - **Installation Response**  
    - See above in Installation Handling.

  - **Installation finished Response**  
    - See above in Installation Handling.

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                                  | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                          |
|-------------------------------|------------------------|-------------------------------------------------|----------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------|
| Bitrix24 Handler               | Webhook                | Receives webhook POST requests from Bitrix24    | -                                | Extract Credentials                     |                                                                                                    |
| Extract Credentials            | Set                    | Extracts auth tokens, domain, and credentials   | Bitrix24 Handler                 | Check Event Type                       |                                                                                                    |
| Check Event Type               | Code                   | Determines if request is installation event     | Extract Credentials              | Is Installation?                      |                                                                                                    |
| Is Installation?               | If                     | Branches based on installation flag              | Check Event Type                 | If Installation finished, Read Installation Settings |                                                                                                    |
| If Installation finished       | If                     | Checks if installation is fully finished         | Is Installation?                 | Installation finished Response, Register Placement, Set Settings Data |                                                                                                    |
| Installation finished Response | Respond to Webhook     | Sends installation completion confirmation       | If Installation finished         | -                                      |                                                                                                    |
| Register Placement            | HTTP Request           | Registers widget placement in Bitrix24           | If Installation finished (false) | Merge Installation info                |                                                                                                    |
| Set Settings Data             | Set                    | Prepares installation settings for storage       | If Installation finished (false) | Create Settings File                   |                                                                                                    |
| Create Settings File          | Convert to File        | Converts settings JSON to file                     | Set Settings Data                | Save Installation Settings             |                                                                                                    |
| Save Installation Settings    | Read/Write File        | Saves settings file to disk                        | Create Settings File             | Merge Installation info                |                                                                                                    |
| Merge Installation info       | Merge                  | Combines installation API and file write results | Save Installation Settings, Register Placement | Installation Response                  |                                                                                                    |
| Installation Response         | Respond to Webhook     | Sends installation finished HTML response         | Merge Installation info          | -                                      |                                                                                                    |
| Read Installation Settings    | Read/Write File        | Reads stored settings file                         | Is Installation? (false branch) | Extract Installation Settings          |                                                                                                    |
| Extract Installation Settings | Extract From File      | Parses JSON settings file                          | Read Installation Settings       | Merge request data with installation settings |                                                                                                    |
| Merge request data with installation settings | Merge                  | Combines request and stored settings              | Extract Installation Settings, Is Installation? (false) | Process Settings                      |                                                                                                    |
| Process Settings             | Function               | Parses settings and extracts task ID              | Merge request data with installation settings | Has Valid Settings?                   |                                                                                                    |
| Has Valid Settings?           | If                     | Checks if settings are valid                       | Process Settings                | Get Task Data, Error Response          |                                                                                                    |
| Get Task Data                | HTTP Request           | Fetches task details from Bitrix24 API             | Has Valid Settings? (true)       | Format Task Data                      |                                                                                                    |
| Format Task Data             | Function               | Formats task data into HTML table                   | Get Task Data                   | Task View Response                    |                                                                                                    |
| Task View Response           | Respond to Webhook     | Sends formatted task HTML response                  | Format Task Data                | -                                      |                                                                                                    |
| Error Response              | Respond to Webhook     | Sends error HTML response if settings invalid       | Has Valid Settings? (false)      | -                                      |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Bitrix24 Handler"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `bitrix24/widgethandler.php`  
   - Response Mode: Response Node  
   - Purpose: Entry point for Bitrix24 webhook requests.

2. **Create Set Node: "Extract Credentials"**  
   - Extract from incoming JSON:  
     - CLIENT_ID: static string `"local.67b8a796e92127.82791242"`  
     - CLIENT_SECRET: static string `"BylHzv4eBw2JuDm7QXOP0C25qzEwf7ATGh79JeOn1iY5lmIRC2"`  
     - domain: from `$json.query.DOMAIN` or `$json.body.domain`  
     - access_token: from `$json.body.AUTH_ID` or `$json.body.access_token`  
     - refresh_token: from `$json.body.REFRESH_ID` or `$json.body.refresh_token`  
     - application_token: from `$json.query.APP_SID` or `$json.body.APP_SID`  
     - expires_in: from `$json.body.AUTH_EXPIRES` or default `3600`  
     - client_endpoint: construct as `https://{domain}/rest/`  
     - settingsFilePath: set to `/data/files/hotline_files/`  
   - Connect output from "Bitrix24 Handler".

3. **Create Code Node: "Check Event Type"**  
   - JavaScript code to check if `body.event === 'ONAPPINSTALL'` or `body.PLACEMENT === 'DEFAULT'`  
   - Sets boolean flags `isInstallation` and `isInstallationFinished` (checks `PLACEMENT_OPTIONS.install_finished === 'Y'`)  
   - Connect output from "Extract Credentials".

4. **Create If Node: "Is Installation?"**  
   - Condition: `$json.isInstallation === true`  
   - Connect output from "Check Event Type".

5. **Create If Node: "If Installation finished"**  
   - Condition: `$json.isInstallationFinished === true`  
   - Connect true branch from "Is Installation?".

6. **Create Respond to Webhook Node: "Installation finished Response"**  
   - HTTP 200  
   - Content-Type: text/html  
   - Body: simple HTML indicating installation fully finished  
   - Connect true branch from "If Installation finished".

7. **Create HTTP Request Node: "Register Placement"**  
   - Method: POST  
   - URL: `https://{domain}/rest/placement.bind?auth={access_token}`  
   - Body parameters:  
     - PLACEMENT: `TASK_VIEW_TAB`  
     - HANDLER: webhook URL from incoming request (`$json.webhookUrl`)  
     - TITLE: `My App`  
   - Connect false branch from "If Installation finished".

8. **Create Set Node: "Set Settings Data"**  
   - Prepare JSON object with keys:  
     - access_token, refresh_token, domain, expires_in, application_token (from extracted credentials)  
     - client_endpoint: `https://{domain}/rest/`  
     - C_REST_CLIENT_ID: static `"app.644f4956606e88.45725320"`  
     - C_REST_CLIENT_SECRET: static `"lUb7WU81Wc4UVCWBJBh0xX5sKYWM4nKmsJl0m4vWb2XR6ByRGF"`  
     - updated_at: current timestamp (`{{$now}}`)  
   - Include `settingsFilePath` from previous node.  
   - Connect false branch from "If Installation finished".

9. **Create Convert to File Node: "Create Settings File"**  
   - Operation: to JSON file  
   - File name: `{settingsFilePath}/widget-app-settings.json`  
   - Connect output from "Set Settings Data".

10. **Create Read/Write File Node: "Save Installation Settings"**  
    - Operation: write  
    - File name: `{settingsFilePath}/widget-app-settings.json`  
    - Append: false  
    - Connect output from "Create Settings File".

11. **Create Merge Node: "Merge Installation info"**  
    - Mode: Combine  
    - Combine all inputs  
    - Connect outputs from "Save Installation Settings" and "Register Placement".

12. **Create Respond to Webhook Node: "Installation Response"**  
    - HTTP 200  
    - Content-Type: text/html  
    - Body: HTML with Bitrix24 JS API call `BX24.installFinish()` to finalize installation  
    - Connect output from "Merge Installation info".

13. **Create Read/Write File Node: "Read Installation Settings"**  
    - Operation: read  
    - File selector: `{settingsFilePath}/widget-app-settings.json`  
    - Connect false branch from "Is Installation?".

14. **Create Extract From File Node: "Extract Installation Settings"**  
    - Operation: from JSON  
    - Connect output from "Read Installation Settings".

15. **Create Merge Node: "Merge request data with installation settings"**  
    - Mode: Combine  
    - Combine all inputs  
    - Connect outputs from "Extract Installation Settings" and false branch of "Is Installation?".

16. **Create Function Node: "Process Settings"**  
    - JavaScript code:  
      - Parses settings JSON from file content  
      - Attempts to parse `PLACEMENT_OPTIONS` from request body to extract `taskId`  
      - Returns JSON with settings, `taskId`, success flag, and original request  
    - Connect output from "Merge request data with installation settings".

17. **Create If Node: "Has Valid Settings?"**  
    - Condition: `$json.success === true`  
    - Connect output from "Process Settings".

18. **Create HTTP Request Node: "Get Task Data"**  
    - Method: POST  
    - URL: `https://{originalRequest.query.DOMAIN}/rest/tasks.task.get?auth={originalRequest.access_token}`  
    - Body: JSON from `originalRequest.body.PLACEMENT_OPTIONS`  
    - Connect true branch from "Has Valid Settings?".

19. **Create Function Node: "Format Task Data"**  
    - JavaScript code:  
      - Extracts task data from API response  
      - Builds an HTML table string with task fields and values  
      - Handles arrays and null values gracefully  
    - Connect output from "Get Task Data".

20. **Create Respond to Webhook Node: "Task View Response"**  
    - HTTP 200  
    - Content-Type: text/html  
    - Body: Full HTML page embedding the formatted task HTML, includes CSS and jQuery references  
    - Connect output from "Format Task Data".

21. **Create Respond to Webhook Node: "Error Response"**  
    - HTTP 200  
    - Content-Type: text/html  
    - Body: HTML error message indicating missing settings or expired token, suggesting reinstallation  
    - Connect false branch from "Has Valid Settings?".

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is designed to be deployed on a server accessible by Bitrix24 via webhook URL.                   | Ensure your n8n instance or server is publicly reachable and secured.                           |
| The workflow uses persistent file storage at `/data/files/hotline_files/` for saving installation settings.    | Adjust file paths according to your server environment and permissions.                         |
| Bitrix24 REST API requires valid OAuth tokens; token refresh logic is not included in this workflow.           | Consider implementing token refresh if tokens expire frequently.                               |
| The installation response includes Bitrix24 JavaScript API call `BX24.installFinish()` to finalize installation.| See https://training.bitrix24.com/rest_help/applications/placement_bind.php for placement API.  |
| Task data is displayed in a simple HTML table; customize the `Format Task Data` node to change UI presentation. |                                                                                               |
| For more information on Bitrix24 REST API and widget development, visit: https://training.bitrix24.com/rest_help/ |                                                                                               |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the Bitrix24 Task Form Widget Application Workflow with Webhook Integration in n8n.