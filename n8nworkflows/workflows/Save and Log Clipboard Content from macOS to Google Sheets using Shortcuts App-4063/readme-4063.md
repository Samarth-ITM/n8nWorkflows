Save and Log Clipboard Content from macOS to Google Sheets using Shortcuts App

https://n8nworkflows.xyz/workflows/save-and-log-clipboard-content-from-macos-to-google-sheets-using-shortcuts-app-4063


# Save and Log Clipboard Content from macOS to Google Sheets using Shortcuts App

### 1. Workflow Overview

This n8n workflow, named **macOS Clipboard Sync Agent**, provides a seamless method to save and log clipboard content from a macOS device into a Google Sheets spreadsheet. It is designed to work in tandem with the macOS Shortcuts app, which triggers the workflow by sending clipboard data via an HTTP POST request to an n8n webhook. The workflow’s main utility is to create a searchable, timestamped history of clipboard entries without relying on third-party clipboard managers.

The workflow is logically divided into three functional blocks:

- **1.1 Input Reception:** Receiving clipboard data from the macOS Shortcut via an HTTP Webhook.
- **1.2 Data Formatting:** Preparing and formatting the incoming data, including timestamp creation.
- **1.3 Data Logging:** Appending the formatted data as a new row into a specified Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming HTTP POST requests sent by the macOS Shortcuts app. It acts as the entry point for clipboard data into the workflow.

**Nodes Involved:**  
- Clipboard Webhook

**Node Details:**  

- **Clipboard Webhook**  
  - **Type:** `Webhook` (HTTP Trigger Node)  
  - **Role:** Listens for HTTP POST requests at a specified path to receive clipboard content as JSON.  
  - **Configuration:**  
    - HTTP Method: POST  
    - Path: `copy-paste`  
    - Authentication: None (open endpoint)  
    - Responds immediately to incoming requests to avoid timeout issues on the client side.  
  - **Key Expressions / Variables:** None, receives raw JSON body with clipboard text.  
  - **Input Connections:** None (trigger node)  
  - **Output Connections:** Connects to the Data Formatter node.  
  - **Version Requirements:** n8n version supporting Webhook node v2 (since typeVersion is 2)  
  - **Potential Failures:**  
    - Network connectivity issues causing missed webhook calls.  
    - Malformed JSON input from the Shortcut app.  
    - Unauthorized access (no auth configured, so open endpoint).  
  - **Sub-workflow:** None

---

#### 2.2 Data Formatting

**Overview:**  
Formats received clipboard data by adding a localized timestamp and structuring the data fields for Google Sheets insertion.

**Nodes Involved:**  
- Data Formatter (Set node)

**Node Details:**  

- **Data Formatter**  
  - **Type:** `Set` (Data manipulation node)  
  - **Role:** Creates two output fields: `Timestamp` and `Text`, formatting the current date/time and extracting clipboard text.  
  - **Configuration:**  
    - Adds field `Timestamp` with expression:  
      ```js
      {{ new Date().toLocaleString("ro-RO", { day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit' }) }}
      ```  
      This formats the timestamp in Romanian locale with specific date/time components.  
    - Adds field `Text` with expression:  
      ```js
      {{ $json.body.text }}
      ```  
      Extracts the clipboard text sent in the JSON body.  
  - **Input Connections:** Receives data from Clipboard Webhook node.  
  - **Output Connections:** Passes formatted data to Clipboard Logger Sheet node.  
  - **Version Requirements:** Uses Set node version 3.4 features (latest expression syntax).  
  - **Potential Failures:**  
    - If incoming JSON does not contain `body.text`, `$json.body.text` will be undefined or cause errors.  
    - Timestamp generation depends on server locale settings; can differ if server locale is not consistent.  
  - **Sub-workflow:** None

---

#### 2.3 Data Logging

**Overview:**  
Appends the formatted clipboard content with a timestamp to a Google Sheets spreadsheet, creating a persistent log.

**Nodes Involved:**  
- Clipboard Logger Sheet (Google Sheets node)

**Node Details:**  

- **Clipboard Logger Sheet**  
  - **Type:** `Google Sheets` (Integration node)  
  - **Role:** Appends or updates a row in the specified Google Sheets document with `Timestamp` and `Text` columns.  
  - **Configuration:**  
    - Operation: `appendOrUpdate` (adds new row; updates if matching)  
    - Mapping Mode: Defines columns explicitly: `Timestamp` and `Text`  
    - Fields mapped:  
      - `Timestamp`: `={{ $json.Timestamp }}`  
      - `Text`: `={{ $json.Text }}`  
    - Sheet Name and Document ID are set as URLs (masked in the JSON but required to specify the target spreadsheet and sheet tab).  
    - Authentication: Uses a Google Service Account credential configured with appropriate access to the target Google Sheets.  
  - **Input Connections:** Connected from Data Formatter node, receives formatted data.  
  - **Output Connections:** None (terminal node)  
  - **Version Requirements:** Uses Google Sheets node version 4.5 with service account support.  
  - **Potential Failures:**  
    - Authentication failures if Google Service Account credentials are misconfigured or revoked.  
    - Permission errors if the service account lacks access to the specified spreadsheet.  
    - Network errors or API limits from Google Sheets.  
    - Mismatched column names in the Google Sheet causing append failures.  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name             | Node Type      | Functional Role           | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                             |
|-----------------------|----------------|---------------------------|---------------------|------------------------|-------------------------------------------------------------------------------------------------------|
| Clipboard Webhook      | Webhook        | Receives clipboard JSON    | None                | Data Formatter         | Listening for incoming requests from the macOS Shortcut. See image: ![Webhook](https://raw.githubusercontent.com/TuguiDragos/macos-clipboard-sync-agent/refs/heads/main/ClipboardHook.png) |
| Data Formatter        | Set            | Adds timestamp & formats data | Clipboard Webhook    | Clipboard Logger Sheet  |                                                                                                       |
| Clipboard Logger Sheet | Google Sheets  | Logs data into Google Sheets | Data Formatter      | None                   | Google Sheet logs all copied content with timestamps. See image: ![Sheet](https://raw.githubusercontent.com/TuguiDragos/macos-clipboard-sync-agent/refs/heads/main/GoogleSheet.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheet**  
   - Open Google Sheets and create a new spreadsheet.  
   - Create two columns: `Timestamp` and `Text`.  
   - Name the sheet tab (e.g., `n8n-test`).  
   - Share the sheet with the Google Service Account email you will use for authentication.

2. **In n8n, create a new workflow named "macOS Clipboard Sync Agent".**

3. **Add the `Webhook` node:**  
   - Name it `Clipboard Webhook`.  
   - Set HTTP Method to `POST`.  
   - Set Path to `copy-paste`.  
   - Authentication: None (leave empty).  
   - Enable "Respond Immediately" (to quickly acknowledge requests).  
   - This node will receive clipboard content JSON from the macOS Shortcut.  

4. **Add the `Set` node:**  
   - Name it `Data Formatter`.  
   - Add two fields:  
     - `Timestamp` (type: string) with expression:  
       ```js
       {{ new Date().toLocaleString("ro-RO", { day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit' }) }}
       ```  
     - `Text` (type: string) with expression:  
       ```js
       {{ $json.body.text }}
       ```  
   - This node formats the incoming data for logging.  

5. **Add the `Google Sheets` node:**  
   - Name it `Clipboard Logger Sheet`.  
   - Set Operation to `appendOrUpdate`.  
   - Configure:  
     - Document ID: your Google Sheets document ID (found in the URL).  
     - Sheet Name: the specific tab name (e.g., `n8n-test`).  
     - Define columns explicitly: `Timestamp` and `Text`.  
     - Map fields:  
       - Timestamp: `={{ $json.Timestamp }}`  
       - Text: `={{ $json.Text }}`  
   - In Credentials, select or create a Google Service Account credential with access to your spreadsheet.  
   - This node appends the clipboard data to your Google Sheet.  

6. **Connect nodes in sequence:**  
   - `Clipboard Webhook` → `Data Formatter` → `Clipboard Logger Sheet`.

7. **Activate the workflow and copy the webhook URL (Test or Production URL).**

8. **Setup macOS Shortcut:**  
   - Open the Shortcuts app on macOS.  
   - Create a new shortcut with two actions:  
     1. `Get Clipboard`  
     2. `Get contents of URL`:  
        - Method: POST  
        - URL: Paste the webhook URL from n8n (e.g., `http://localhost:5678/webhook-test/copy-paste` or your network IP and port).  
        - Request Body: JSON  
        - Under JSON parameters, add key `text` with value set to the Clipboard content.  
   - Save the shortcut, e.g., `Copy-Paste`.  
   - Optionally assign a keyboard shortcut to run it quickly.

9. **Test by running the macOS Shortcut:**  
   - Trigger the shortcut manually or via keyboard.  
   - Check the Google Sheet for a new row with the copied text and timestamp.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow requires n8n installed locally or on a VPS with reachable webhook URL from macOS device. | Setup instruction preamble.                                                                         |
| Apple Shortcuts app is built-in on macOS for easy automation integration without extra software. | macOS native integration.                                                                           |
| No third-party clipboard managers needed; preserves privacy and offline operation.               | Benefits of this approach.                                                                          |
| Workflow can be expanded with AI, text filters, or categorization for advanced clipboard management. | Future extension ideas.                                                                             |
| MIT License applies; credit to [@TuguiDragos](https://tuguidragos.com) if reused or extended.    | Licensing and attribution.                                                                          |
| Visual references for nodes and Shortcuts setup are available at:                                | [Webhook Image](https://raw.githubusercontent.com/TuguiDragos/macos-clipboard-sync-agent/refs/heads/main/ClipboardHook.png)  
[Shortcuts Setup](https://raw.githubusercontent.com/TuguiDragos/macos-clipboard-sync-agent/refs/heads/main/ShortrtcutsApple.png)  
[Google Sheet Image](https://raw.githubusercontent.com/TuguiDragos/macos-clipboard-sync-agent/refs/heads/main/GoogleSheet.png) |

---

**Disclaimer:** The provided workflow is an automated solution created exclusively with n8n, adhering strictly to content policies with no illegal or protected data. All data processed is legal and public.