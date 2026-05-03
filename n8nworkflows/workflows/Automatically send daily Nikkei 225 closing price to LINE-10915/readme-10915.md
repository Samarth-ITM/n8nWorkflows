Automatically send daily Nikkei 225 closing price to LINE

https://n8nworkflows.xyz/workflows/automatically-send-daily-nikkei-225-closing-price-to-line-10915


# Automatically send daily Nikkei 225 closing price to LINE

### 1. Workflow Overview

This n8n workflow automates the daily retrieval and dissemination of the Nikkei 225 stock index closing price to specified users on the LINE messaging platform. It is designed to run every weekday at 4 PM JST, immediately after market close, fetch the latest Nikkei 225 data from an external API, format this information into a user-friendly message, and send it to a predefined list of LINE users via the LINE Messaging API.

The workflow is logically divided into two main blocks:

- **1.1 Scheduled Data Fetching:** Triggers on a weekday schedule and retrieves Nikkei 225 data from an HTTP API.
- **1.2 Message Preparation and Delivery:** Formats the fetched data into a LINE-compatible message payload and sends it via an authenticated HTTP request.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Fetching

**Overview:**  
This block initiates the workflow on a fixed schedule and retrieves the latest Nikkei 225 closing data from a specified HTTP endpoint.

**Nodes Involved:**  
- Every Weekday at 4 PM JST (Schedule Trigger)  
- Get Nikkei 225 Data (HTTP Request)  
- Format Message (Set node, unused)

**Node Details:**

- **Every Weekday at 4 PM JST**  
  - *Type & Role:* Schedule Trigger; initiates workflow execution.  
  - *Configuration:* Cron expression set to trigger at midnight UTC (which corresponds to 4 PM JST) Monday through Friday (`0 0 0 * * 1-5`).  
  - *Inputs:* None  
  - *Outputs:* Connects to "Get Nikkei 225 Data" node.  
  - *Potential Failures:* Misconfigured cron expression could cause missed or off-time triggers. Timezone awareness is critical.  
  - *Version Specifics:* Standard schedule trigger node, version 1.

- **Get Nikkei 225 Data**  
  - *Type & Role:* HTTP Request; fetches the latest Nikkei 225 data from an external API.  
  - *Configuration:* Uses a GET request to the URL `https://8080-id15jiyrj1k5dcuz7d9nz-3e90844d.manus-asia.computer/nikkei`. No additional headers or authentication configured.  
  - *Inputs:* Triggered by the schedule node.  
  - *Outputs:* Passes JSON response to the next node.  
  - *Potential Failures:*  
    - API endpoint downtime or incorrect URL.  
    - Network timeouts or errors.  
    - Unexpected response structure leading to downstream failures.  
  - *Version Specifics:* HTTP Request node version 3.

- **Format Message**  
  - *Type & Role:* Set node; originally intended for message formatting but currently unused.  
  - *Configuration:* No parameters configured; no expressions or variables set.  
  - *Inputs:* Receives data from "Get Nikkei 225 Data."  
  - *Outputs:* Connected downstream but effectively bypassed since formatting occurs in the Code node.  
  - *Potential Failures:* None, as it is unused. Can be safely deleted.  
  - *Version Specifics:* Set node version 3.

#### 1.2 Message Preparation and Delivery

**Overview:**  
This block formats the obtained stock data into a LINE Messaging API compatible payload and sends it to a list of specified users through the LINE multicast API.

**Nodes Involved:**  
- Prepare LINE API Payload (Code node)  
- Send to LINE via HTTP (HTTP Request)

**Node Details:**

- **Prepare LINE API Payload**  
  - *Type & Role:* Code node; constructs the message text and formats the payload for the LINE Messaging API.  
  - *Configuration:*  
    - JavaScript code extracts `date`, `close`, `change`, and `change_pct` from the input JSON.  
    - Defines an array of recipient LINE user IDs (`userIds`), currently including one user ID string.  
    - Constructs a text message with emojis and formatted stock data.  
    - Creates the payload object with `to` field set to the user IDs array and `messages` array containing the text message.  
  - *Key Expressions:*  
    - Accesses data fields like `data.date`, `data.close`, etc. from the previous node.  
    - Returns the payload as JSON for the next HTTP request node.  
  - *Inputs:* Receives JSON data from "Format Message" node.  
  - *Outputs:* Produces a JSON payload for the HTTP request node.  
  - *Potential Failures:*  
    - Missing or malformed input data leading to undefined variables.  
    - Empty or incorrect `userIds` array causing message delivery failure.  
    - Syntax errors in JavaScript code.  
  - *Version Specifics:* Code node version 2.

- **Send to LINE via HTTP**  
  - *Type & Role:* HTTP Request; sends the formatted payload to the LINE Messaging API multicast endpoint.  
  - *Configuration:*  
    - POST request to `https://api.line.me/v2/bot/message/multicast`.  
    - Body sent as JSON, directly from the previous node's output.  
    - Headers:  
      - `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder token to be replaced with actual LINE API token).  
      - `Content-Type: application/json`  
    - Response expected in JSON format.  
  - *Inputs:* Receives payload JSON from the Code node.  
  - *Outputs:* The response from LINE API (success or error) can be used downstream if needed (not connected here).  
  - *Potential Failures:*  
    - Invalid or expired LINE API token causing authentication failure (401 Unauthorized).  
    - Network issues or HTTP errors.  
    - Payload malformation resulting in 400 Bad Request.  
    - Rate limiting by LINE API.  
  - *Version Specifics:* HTTP Request node version 4.2.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                          | Input Node(s)            | Output Node(s)              | Sticky Note                                                                                     |
|----------------------------|---------------------|----------------------------------------|--------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| Every Weekday at 4 PM JST  | Schedule Trigger    | Scheduled trigger to start workflow    | None                     | Get Nikkei 225 Data         | ## 1. Get Daily Stock Data This section runs on a schedule and fetches the latest stock data from an API. |
| Get Nikkei 225 Data        | HTTP Request       | Fetch Nikkei 225 data from API         | Every Weekday at 4 PM JST | Format Message              | ## 1. Get Daily Stock Data This section runs on a schedule and fetches the latest stock data from an API. |
| Format Message             | Set                | (Unused) originally for formatting     | Get Nikkei 225 Data       | Prepare LINE API Payload     | Note: This node is unused and can be deleted. All formatting is done in the "Code" node.        |
| Prepare LINE API Payload   | Code               | Format message and prepare LINE payload| Format Message            | Send to LINE via HTTP        | ## 2. Format & Send to LINE This section formats the data into a message and sends it via the LINE Messaging API. |
| Send to LINE via HTTP      | HTTP Request       | Send message payload to LINE API       | Prepare LINE API Payload  | None                       | ## 2. Format & Send to LINE This section formats the data into a message and sends it via the LINE Messaging API. |
| Sticky Note                | Sticky Note        | Workflow overview and setup instructions| None                     | None                       | This workflow automatically fetches the Nikkei 225 closing price every weekday and sends a formatted message to a list of users on LINE.\n\n### How it works\n1.  **Trigger:** Runs every weekday at 4 PM JST (market close).\n2.  **Get Data:** Fetches the latest Nikkei 225 data from an API.\n3.  **Format & Send:** A Code node formats the data and sends it to a list of LINE users.\n\n### Setup steps\n1.  **Add Credentials:** In the "Send to LINE..." node (Section 2), add your LINE API Token to the 'Authorization' header.\n2.  **Add User IDs:** In the "Prepare LINE..." node (Section 2), edit the 'userIds' array.\n3.  **Update API URL:** In the "Get Nikkei 225 Data" node (Section 1), replace the temporary URL with your own. |
| Sticky Note1               | Sticky Note        | Section 1 description                   | None                     | None                       | ## 1. Get Daily Stock Data This section runs on a schedule and fetches the latest stock data from an API. |
| Sticky Note2               | Sticky Note        | Section 2 description                   | None                     | None                       | ## 2. Format & Send to LINE This section formats the data into a message and sends it via the LINE Messaging API. |
| Sticky Note3               | Sticky Note        | Format Message node unused note        | None                     | None                       | Note: This node is unused and can be deleted. All formatting is done in the "Code" node.        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Node Type: Schedule Trigger  
   - Name: "Every Weekday at 4 PM JST"  
   - Parameters:  
     - Select "Cron Expression" mode.  
     - Set expression to `0 0 0 * * 1-5` (this triggers at midnight UTC Monday-Friday, equals 4 PM JST).  
   - No credentials needed.  
   - Connect output to next node.

2. **Create HTTP Request Node to Fetch Nikkei Data**  
   - Node Type: HTTP Request  
   - Name: "Get Nikkei 225 Data"  
   - Parameters:  
     - HTTP Method: GET  
     - URL: `https://8080-id15jiyrj1k5dcuz7d9nz-3e90844d.manus-asia.computer/nikkei` (replace with your actual Nikkei data API).  
     - No authentication or additional headers.  
   - Connect input from "Every Weekday at 4 PM JST" node.  
   - Output connects to next node.

3. **(Optional) Create Set Node for Formatting**  
   - Node Type: Set  
   - Name: "Format Message"  
   - No parameters or fields set (this node is unused and can be omitted).  
   - Connect input from "Get Nikkei 225 Data" node.  
   - Output connects to next node.

4. **Create Code Node to Prepare LINE Payload**  
   - Node Type: Code  
   - Name: "Prepare LINE API Payload"  
   - Parameters:  
     - Language: JavaScript  
     - Code snippet:  
       ```javascript
       // Define target LINE user IDs
       const userIds = [
         "U1734402c36e0ccc28ca2f40fa752e923",
       ];

       // Extract Nikkei data from input
       const data = $input.first().json;

       // Build message string
       const message = `📊 本日の日経平均株価 (${data.date})\n\n終値: ${data.close}円\n前日比: ${data.change}円 (${data.change_pct})`;

       // Construct LINE Messaging API payload
       const payload = {
         to: userIds,
         messages: [
           {
             type: "text",
             text: message
           }
         ]
       };

       return [{ json: payload }];
       ```  
   - Connect input from "Format Message" node.  
   - Output connects to next node.

5. **Create HTTP Request Node to Send to LINE**  
   - Node Type: HTTP Request  
   - Name: "Send to LINE via HTTP"  
   - Parameters:  
     - HTTP Method: POST  
     - URL: `https://api.line.me/v2/bot/message/multicast`  
     - Body Content: Select JSON and set to `{{$json}}` (pass the payload from previous node).  
     - Headers:  
       - `Authorization`: `Bearer YOUR_TOKEN_HERE` (replace with your actual LINE channel access token)  
       - `Content-Type`: `application/json`  
     - Enable "Send Body" and "Send Headers" as JSON.  
     - Expect response as JSON.  
   - Connect input from "Prepare LINE API Payload" node.

6. **Add Credentials**  
   - For the "Send to LINE via HTTP" node, no special credential profile is required if using header-based token authorization.  
   - Ensure your LINE Messaging API token is valid and has multicast messaging permission.

7. **Final Workflow Connections**  
   - "Every Weekday at 4 PM JST" → "Get Nikkei 225 Data" → "Format Message" (optional) → "Prepare LINE API Payload" → "Send to LINE via HTTP"

8. **Optional Cleanup**  
   - Delete "Format Message" node if unused.  
   - Adjust user IDs in the Code node to your target recipients.  
   - Replace API URL in "Get Nikkei 225 Data" with your data source.  
   - Replace `YOUR_TOKEN_HERE` with your actual LINE Messaging API access token.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow automatically fetches the Nikkei 225 closing price every weekday and sends a formatted message to a list of users on LINE. Setup requires editing the LINE API token, user IDs, and API URL in respective nodes. | Inline workflow sticky note with detailed setup instructions                                                |
| The schedule trigger uses a cron expression for 0:00 UTC Monday to Friday to align with 4 PM JST market close. Adjust timezone carefully if deploying outside JST context.                                                        | Sticky note explaining scheduling details                                                                  |
| The LINE Messaging API multicast endpoint allows sending messages to multiple users in a single API call, which is efficient for broadcasting updates.                                                                           | Official LINE Messaging API documentation: https://developers.line.biz/en/reference/messaging-api/#send-multicast-message |
| Ensure the target API providing Nikkei 225 data returns consistent JSON with fields: `date`, `close`, `change`, and `change_pct` for proper message formatting.                                                                   | Critical for preventing runtime errors in the Code node                                                    |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, complying strictly with content policies and containing no illegal or protected elements. All data handled is legal and public.