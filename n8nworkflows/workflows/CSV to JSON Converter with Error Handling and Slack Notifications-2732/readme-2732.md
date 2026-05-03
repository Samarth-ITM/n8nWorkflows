CSV to JSON Converter with Error Handling and Slack Notifications

https://n8nworkflows.xyz/workflows/csv-to-json-converter-with-error-handling-and-slack-notifications-2732


# CSV to JSON Converter with Error Handling and Slack Notifications

### 1. Workflow Overview

This workflow provides a robust API endpoint to convert CSV data into JSON format, supporting multiple input formats and comprehensive error handling with Slack notifications. It is designed for developers or teams needing to automate CSV-to-JSON transformations via HTTP POST requests, either by uploading CSV files or sending raw CSV text.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives POST requests at a webhook endpoint and routes the input based on content type.
- **1.2 CSV Data Processing:** Extracts CSV data from files or raw text, converting it into JSON objects.
- **1.3 Data Aggregation and Validation:** Aggregates processed data and checks for errors.
- **1.4 Response Handling:** Returns success or error responses to the API caller.
- **1.5 Error Notification:** Sends detailed error notifications to a configured Slack channel for monitoring and debugging.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives incoming POST requests at the webhook endpoint `/tool/csv-to-json` and uses a Switch node to route the data based on its format: file upload (binary), raw CSV text, or JSON.

**Nodes Involved:**  
- POST (Webhook)  
- Switch  

**Node Details:**

- **POST (Webhook)**  
  - Type: Webhook  
  - Role: Entry point for POST requests at `/tool/csv-to-json`. Accepts binary data under property `data`.  
  - Configuration: HTTP method POST, response mode set to response node (delays response until downstream nodes complete).  
  - Inputs: External HTTP request  
  - Outputs: Connected to Switch node  
  - Edge Cases: Missing or malformed requests; unsupported HTTP methods are not handled here.  

- **Switch**  
  - Type: Switch  
  - Role: Routes input based on content type or presence of binary data.  
  - Configuration:  
    - Output "File": If binary data exists (non-empty `$binary`).  
    - Output "Data/Text": If `content-type` header equals `text/plain`.  
    - Output "appJSON": If `content-type` header equals `application/json`.  
    - Fallback output: "extra" (not connected, so unhandled inputs lead to error).  
  - Inputs: From POST node  
  - Outputs:  
    - "File" → Extract From File  
    - "Data/Text" → Change Field  
    - "appJSON" → Error Response (unsupported input)  
    - "extra" → Error Response  
  - Edge Cases: Missing or unexpected content-type headers; binary data with incorrect format.

---

#### 1.2 CSV Data Processing

**Overview:**  
Processes CSV data depending on input type: extracts CSV from uploaded files or converts raw CSV text to JSON objects. Handles both comma and semicolon delimiters.

**Nodes Involved:**  
- Extract From File  
- Change Field  
- Convert Raw Text To CSV  

**Node Details:**

- **Extract From File**  
  - Type: Extract From File  
  - Role: Extracts CSV content from uploaded binary file under property `data0`.  
  - Configuration: Default options, binary property name set to `data0`.  
  - Inputs: From Switch output "File"  
  - Outputs: Aggregates data downstream or sends error response on failure.  
  - Edge Cases: File format errors, empty files, extraction failures.  
  - Error Handling: Continues on error output to allow error response node to handle.  

- **Change Field**  
  - Type: Set  
  - Role: Prepares raw CSV text for conversion by assigning the raw CSV string from the request body to a new field `csv`.  
  - Configuration: Sets field `csv` to `{{$json.body}}` (raw text content).  
  - Inputs: From Switch output "Data/Text"  
  - Outputs: Connects to Convert Raw Text To CSV or Error Response on failure.  
  - Edge Cases: Empty or missing body content.  

- **Convert Raw Text To CSV**  
  - Type: Code (JavaScript)  
  - Role: Parses raw CSV text into JSON objects, supporting both comma and semicolon delimiters.  
  - Configuration:  
    - Splits input CSV string by newline.  
    - Splits headers and each line by regex `/,|;/`.  
    - Constructs JSON objects mapping headers to values.  
    - Throws error if no data rows found.  
  - Inputs: From Change Field node  
  - Outputs: Connects to Check if Value node for error checking.  
  - Edge Cases: Malformed CSV, inconsistent delimiters, empty input, missing headers.  
  - Error Handling: Throws error to trigger error response downstream.  

---

#### 1.3 Data Aggregation and Validation

**Overview:**  
Aggregates JSON data arrays and checks for errors before sending the final response.

**Nodes Involved:**  
- Aggregate  
- Aggregate1  
- Check if Value  

**Node Details:**

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates all items from Extract From File node into a single JSON array under field `jsondata`.  
  - Inputs: From Extract From File  
  - Outputs: Connects to Success Response node.  
  - Edge Cases: Empty input arrays.  

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates all items from Check if Value node into a single JSON array under field `jsondata`.  
  - Inputs: From Check if Value  
  - Outputs: Connects to Success Response2 node.  
  - Edge Cases: Empty input arrays.  

- **Check if Value**  
  - Type: If  
  - Role: Checks if the JSON data contains an `error` field.  
  - Configuration: Condition tests if `$json.error` does not exist (i.e., no error).  
  - Inputs: From Convert Raw Text To CSV  
  - Outputs:  
    - True (no error) → Aggregate1  
    - False (error present) → Error Response  
  - Edge Cases: Unexpected JSON structure, missing fields.

---

#### 1.4 Response Handling

**Overview:**  
Sends HTTP responses back to the API caller, either success with JSON data or error with message.

**Nodes Involved:**  
- Success Response  
- Success Response2  
- Error Response  

**Node Details:**

- **Success Response**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP 200 response with JSON data from Aggregate node (file upload path).  
  - Configuration:  
    - Response code: 200  
    - Response body: JSON with status "OK" and stringified JSON data from `jsondata`.  
  - Inputs: From Aggregate  
  - Outputs: On error, triggers Slack notification.  
  - Edge Cases: Serialization errors.  

- **Success Response2**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP 200 response with JSON data from Aggregate1 node (raw text path).  
  - Configuration:  
    - Response code: 200  
    - Response body: JSON stringified `jsondata`.  
  - Inputs: From Aggregate1  
  - Outputs: On error, triggers Slack notification.  
  - Edge Cases: Serialization errors.  

- **Error Response**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP 500 error response with a generic error message.  
  - Configuration:  
    - Response code: 500  
    - Response body: JSON with status "error" and a user-friendly message.  
  - Inputs: From multiple nodes on error paths (Switch, Extract From File, Change Field, Check if Value)  
  - Outputs: Connects to Slack notification node.  
  - Edge Cases: Multiple error sources funnel here.

---

#### 1.5 Error Notification

**Overview:**  
Sends detailed error notifications to a Slack channel for monitoring and debugging, including execution time and a link to the execution log.

**Nodes Involved:**  
- Send to Error Channel  

**Node Details:**

- **Send to Error Channel**  
  - Type: Slack  
  - Role: Posts a formatted message to a Slack channel when an error occurs.  
  - Configuration:  
    - OAuth2 authentication for Slack.  
    - Channel ID set to `"C0832GBAEN4"`.  
    - Message type: Block Kit with sections including:  
      - Error alert emoji and message.  
      - Timestamp of error occurrence.  
      - Execution ID.  
      - Button linking to the workflow execution error page (URL placeholder to be replaced).  
  - Inputs: From Error Response and error outputs of Success Response nodes.  
  - Outputs: None (terminal node).  
  - Edge Cases: Slack API authentication failures, network issues.

---

### 3. Summary Table

| Node Name             | Node Type             | Functional Role                          | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                   |
|-----------------------|-----------------------|----------------------------------------|-----------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| POST                  | Webhook               | Receives POST requests at webhook      | External HTTP request  | Switch                      |                                                                                                               |
| Switch                | Switch                | Routes input by content type            | POST                  | Extract From File, Change Field, Error Response |                                                                                                               |
| Extract From File      | Extract From File     | Extracts CSV from uploaded file         | Switch (File output)   | Aggregate, Error Response    |                                                                                                               |
| Change Field          | Set                   | Assigns raw CSV text to field `csv`    | Switch (Data/Text)     | Convert Raw Text To CSV, Error Response |                                                                                                               |
| Convert Raw Text To CSV| Code                  | Parses raw CSV text to JSON objects     | Change Field           | Check if Value               |                                                                                                               |
| Check if Value        | If                     | Checks for error field in JSON          | Convert Raw Text To CSV| Aggregate1, Error Response   |                                                                                                               |
| Aggregate             | Aggregate              | Aggregates JSON data from file path     | Extract From File      | Success Response             |                                                                                                               |
| Aggregate1            | Aggregate              | Aggregates JSON data from raw text path | Check if Value         | Success Response2            |                                                                                                               |
| Success Response      | Respond to Webhook     | Sends success response for file input   | Aggregate              | (Error output) Send to Error Channel |                                                                                                               |
| Success Response2     | Respond to Webhook     | Sends success response for raw text input | Aggregate1             | (Error output) Send to Error Channel |                                                                                                               |
| Error Response        | Respond to Webhook     | Sends error response (500)               | Switch, Extract From File, Change Field, Check if Value | Send to Error Channel          |                                                                                                               |
| Send to Error Channel | Slack                  | Sends error notification to Slack       | Error Response, Success Response error outputs | None                        |                                                                                                               |
| Sticky Note1          | Sticky Note            | Testing instructions for CURL            | None                  | None                        | Testing can be done with CURL or similar. For file posting using Form Data: curl -X POST "https://yoururl.com/webhook-test/tool/csv-to-json" -H "Content-Type: text/csv" --data-binary @path/to/your/file.csv This can also be tested using the Test workflow |
| Sticky Note3          | Sticky Note            | Response format explanation              | None                  | None                        | Response: Where possible we will be returning a binary object. If there is an error: {"status": "error", "data": "error message to display"} |
| Sticky Note4          | Sticky Note            | Empty, no content                        | None                  | None                        |                                                                                                               |
| Sticky Note           | Sticky Note            | Sample raw CSV data send example         | None                  | None                        | Sample of Raw CSV Data Send: Use the HTTP request node below to see how to send the Raw CSV data into this workflow. Don't forget to include the \n's |
| Send Raw CSV          | HTTP Request           | Example HTTP request sending raw CSV    | None                  | None                        |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `POST`  
   - HTTP Method: POST  
   - Path: `tool/csv-to-json`  
   - Response Mode: `Response Node`  
   - Binary Property Name: `data`  
   - Save credentials if needed (none required here).  

2. **Create Switch Node**  
   - Name: `Switch`  
   - Add three outputs with keys: `File`, `Data/Text`, `appJSON`  
   - Configure rules:  
     - `File`: Condition - `$binary` is not empty (object not empty).  
     - `Data/Text`: Condition - `$json.headers['content-type']` equals `text/plain`.  
     - `appJSON`: Condition - `$json.headers['content-type']` equals `application/json`.  
   - Set fallback output to `extra`.  
   - Connect `POST` node output to `Switch` input.  

3. **Create Extract From File Node**  
   - Name: `Extract From File`  
   - Binary Property Name: `data0` (default)  
   - Connect `Switch` output `File` to this node.  
   - Set error handling to continue on error output.  

4. **Create Change Field Node**  
   - Name: `Change Field`  
   - Add assignment: Set field `csv` to expression `{{$json.body}}` (raw text from request body).  
   - Connect `Switch` output `Data/Text` to this node.  
   - Set error handling to continue on error output.  

5. **Create Code Node**  
   - Name: `Convert Raw Text To CSV`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const csvData = $input.all()[0]?.json?.csv;

     // Use a regex to split on either ',' or ';'
     const lines = csvData.split("\n");
     const headers = lines[0].split(/,|;/);

     const jsonData = lines.slice(1).map((line) => {
       const data = line.split(/,|;/);
       let obj = {};
       headers.forEach((header, i) => {
         obj[header] = data[i];
       });
       return obj;
     });

     if (jsonData.length === 0) {
       throw new Error("No data to process");
     }

     return jsonData;
     ```
   - Connect `Change Field` output to this node.  
   - Set error handling to continue regular output.  

6. **Create If Node**  
   - Name: `Check if Value`  
   - Condition: Check if `$json.error` does **not** exist (string operation `notExists`).  
   - Connect `Convert Raw Text To CSV` output to this node.  

7. **Create Aggregate Nodes**  
   - Name: `Aggregate`  
     - Aggregate all item data into field `jsondata`.  
     - Connect `Extract From File` output to this node.  
   - Name: `Aggregate1`  
     - Aggregate all item data into field `jsondata`.  
     - Connect `Check if Value` true output to this node.  

8. **Create Respond to Webhook Nodes**  
   - Name: `Success Response`  
     - Response code: 200  
     - Response body:  
       ```json
       {
         "status": "OK",
         "data": {{ JSON.stringify($json.jsondata) }}
       }
       ```  
     - Connect `Aggregate` output to this node.  
     - Set error handling to continue on error output.  
   - Name: `Success Response2`  
     - Response code: 200  
     - Response body: `={{ JSON.stringify($json.jsondata) }}`  
     - Connect `Aggregate1` output to this node.  
     - Set error handling to continue on error output.  
   - Name: `Error Response`  
     - Response code: 500  
     - Response body:  
       ```json
       {
         "status": "error",
         "data": "There was a problem converting your CSV. Please refresh the page and try again."
       }
       ```  
     - Connect error outputs from `Switch` (appJSON and extra), `Extract From File`, `Change Field`, and `Check if Value` false output to this node.  
     - Set error handling to continue on error output.  

9. **Create Slack Node**  
   - Name: `Send to Error Channel`  
   - Authentication: OAuth2 (Slack) - configure OAuth2 credentials for Slack API.  
   - Channel ID: `C0832GBAEN4` (update as needed).  
   - Message Type: Block  
   - Message Blocks:  
     ```json
     {
       "blocks": [
         {
           "type": "section",
           "text": {
             "type": "mrkdwn",
             "text": ":interrobang: Error in CSV to JSON tool"
           }
         },
         {
           "type": "section",
           "text": {
             "type": "mrkdwn",
             "text": "*Time:*\n{{ $now.format('dd/MM/yyyy HH:mm:ss') }}\n*Execution ID:*\n{{ $execution.id }}\n"
           },
           "accessory": {
             "type": "button",
             "text": {
               "type": "plain_text",
               "text": "Go to Error",
               "emoji": true
             },
             "value": "error",
             "url": "[insert URL here]{{ $workflow.id }}/executions/{{ $execution.id }}",
             "action_id": "button-action",
             "style": "primary"
           }
         }
       ]
     }
     ```  
   - Connect error outputs from `Error Response`, and error outputs from `Success Response` and `Success Response2` nodes to this Slack node.  

10. **Connect all nodes as per the described flow:**  
    - `POST` → `Switch`  
    - `Switch` "File" → `Extract From File` → `Aggregate` → `Success Response`  
    - `Switch` "Data/Text" → `Change Field` → `Convert Raw Text To CSV` → `Check if Value`  
    - `Check if Value` true → `Aggregate1` → `Success Response2`  
    - `Check if Value` false → `Error Response`  
    - `Switch` "appJSON" and fallback → `Error Response`  
    - `Extract From File` error → `Error Response`  
    - `Change Field` error → `Error Response`  
    - `Error Response` → `Send to Error Channel`  
    - `Success Response` error → `Send to Error Channel`  
    - `Success Response2` error → `Send to Error Channel`  

11. **Deploy the workflow.**

12. **Test using CURL or HTTP clients:**  
    - File upload:  
      ```bash
      curl -X POST "https://yoururl.com/webhook-test/tool/csv-to-json" \
           -H "Content-Type: text/csv" \
           --data-binary @path/to/your/file.csv
      ```  
    - Raw CSV text:  
      Send POST with `Content-Type: text/plain` and raw CSV in body.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Testing can be done with CURL or similar. For file posting using Form Data: `curl -X POST "https://yoururl.com/webhook-test/tool/csv-to-json" -H "Content-Type: text/csv" --data-binary @path/to/your/file.csv` | Sticky Note1                                                                                           |
| Response format explanation: On error, returns JSON `{ "status": "error", "data": "error message to display" }`.     | Sticky Note3                                                                                           |
| Sample raw CSV data send example: Use HTTP request node with raw CSV text including newline characters (`\n`).       | Sticky Note                                                                                             |
| Slack OAuth2 credentials must be configured properly for the Slack node to send notifications.                       | Slack node configuration                                                                               |
| Replace `[insert URL here]` in Slack message URL with your actual n8n instance URL for direct error execution access.| Slack node message configuration                                                                       |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the "CSV to JSON Converter with Error Handling and Slack Notifications" workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and automation agents to work effectively with the workflow.