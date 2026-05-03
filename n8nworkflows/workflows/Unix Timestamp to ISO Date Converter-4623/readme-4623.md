Unix Timestamp to ISO Date Converter

https://n8nworkflows.xyz/workflows/unix-timestamp-to-iso-date-converter-4623


# Unix Timestamp to ISO Date Converter

### 1. Workflow Overview

This workflow is designed to convert a Unix timestamp (expressed in seconds) received via a webhook into an ISO 8601 formatted date string. It is intended for use cases where an external system sends a Unix timestamp and requires a standardized, human-readable date-time string in response. The workflow is structured into three logical blocks:

- **1.1 Input Reception:** Receives the incoming POST request with the Unix timestamp payload.
- **1.2 Timestamp Conversion:** Processes the timestamp by converting it from seconds to milliseconds, then formats it as an ISO 8601 string.
- **1.3 Response Dispatch:** Sends back the converted ISO date string as a response to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for incoming HTTP POST requests containing a JSON object with a Unix timestamp. Validates and prepares the data for conversion.

- **Nodes Involved:**  
  - Receive Timestamp Webhook  
  - Note: Webhook Input (sticky note)

- **Node Details:**

  - **Receive Timestamp Webhook**  
    - *Type:* Webhook  
    - *Role:* Entry point of the workflow, listens for POST requests at the path `/convert-timestamp`.  
    - *Configuration:*  
      - HTTP Method: POST  
      - Path: `convert-timestamp`  
      - Response Mode: `responseNode` (the workflow will respond using a downstream Respond node)  
    - *Key Expressions:* None (standard webhook)  
    - *Input:* External HTTP POST request with JSON body containing a `timestamp` field (Unix timestamp in seconds).  
    - *Output:* JSON object with the original request body under `$json.body`.  
    - *Potential Failures:*  
      - Missing or malformed `timestamp` field in request body may cause downstream errors.  
      - HTTP request timeout or network issues.  
    - *Version Requirements:* Uses Webhook node version 2.

  - **Note: Webhook Input** (Sticky Note)  
    - *Purpose:* Documents expected input format for clarity.  
    - *Content:* Expects a JSON body with property `timestamp` (Unix timestamp in seconds, e.g., 1678886400 for March 15, 2023, 00:00:00 UTC).

---

#### 2.2 Timestamp Conversion

- **Overview:**  
  Converts the Unix timestamp from seconds to milliseconds, then formats it as an ISO 8601 string and adds this as a new field `convertedTime`.

- **Nodes Involved:**  
  - Convert to ISO 8601  
  - Note: Conversion Logic (sticky note)

- **Node Details:**

  - **Convert to ISO 8601**  
    - *Type:* Set  
    - *Role:* Performs the timestamp conversion and assigns the result to a new JSON property.  
    - *Configuration:*  
      - Adds a new field `convertedTime` as a string.  
      - Expression used: `={{ new Date($json.body.timestamp * 1000).toISOString() }}`  
        - Multiplies the incoming timestamp by 1000 to convert seconds to milliseconds.  
        - Uses JavaScript `Date` object to format the timestamp as ISO 8601.  
    - *Input:* JSON containing `body.timestamp` from the webhook node.  
    - *Output:* JSON with original data plus `convertedTime` field.  
    - *Potential Failure Cases:*  
      - If `timestamp` is missing or not a valid number, the conversion will fail or produce `Invalid Date`.  
      - Expression evaluation errors if `$json.body.timestamp` is undefined.  
    - *Version Requirements:* Set node version 3.4.

  - **Note: Conversion Logic** (Sticky Note)  
    - *Purpose:* Explains the conversion formula and output format.  
    - *Content:* Describes multiplying the timestamp by 1000 and converting to ISO 8601 string, stored as `convertedTime`.

---

#### 2.3 Response Dispatch

- **Overview:**  
  Sends the converted ISO 8601 date back as the HTTP response to the original POST request.

- **Nodes Involved:**  
  - Respond with Converted Time  
  - Note: Webhook Response (sticky note)

- **Node Details:**

  - **Respond with Converted Time**  
    - *Type:* Respond to Webhook  
    - *Role:* Returns the workflow output to the webhook caller.  
    - *Configuration:*  
      - `respondWith`: all incoming items (i.e., the entire JSON including `convertedTime`)  
    - *Input:* Receives data from the Set node with the converted ISO date.  
    - *Output:* HTTP response containing JSON with the `convertedTime` field.  
    - *Potential Failures:*  
      - If upstream data is missing or malformed, the response may be incomplete or invalid.  
      - Network or timeout errors in response delivery.  
    - *Version Requirements:* Node version 1.2.

  - **Note: Webhook Response** (Sticky Note)  
    - *Purpose:* Clarifies that this node outputs the final converted time as the webhook response.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role               | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                          |
|-------------------------|-----------------------|------------------------------|-------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------|
| Note: Webhook Input     | Sticky Note           | Documents expected webhook input | —                       | —                       | This node listens for incoming POST requests. It expects a JSON body with a single property: 'timestamp' (a Unix timestamp in seconds, e.g., 1678886400 for March 15, 2023, 12:00:00 AM UTC). |
| Receive Timestamp Webhook | Webhook               | Receives incoming timestamp POST requests | —                       | Convert to ISO 8601     |                                                                                                                      |
| Note: Conversion Logic  | Sticky Note           | Documents conversion logic    | —                       | —                       | This node takes the 'timestamp' from the webhook, multiplies it by 1000 (to convert seconds to milliseconds), and then converts it to an ISO 8601 formatted string (e.g., '2023-03-15T00:00:00.000Z'). This new string is added as 'convertedTime'. |
| Convert to ISO 8601     | Set                   | Converts timestamp to ISO 8601 | Receive Timestamp Webhook | Respond with Converted Time |                                                                                                                      |
| Note: Webhook Response  | Sticky Note           | Documents webhook response    | —                       | —                       | This node sends the 'convertedTime' back as the response to the original webhook caller. It's the final output of the workflow. |
| Respond with Converted Time | Respond to Webhook     | Sends back converted ISO date | Convert to ISO 8601     | —                       |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Receive Timestamp Webhook`  
   - Type: Webhook (version 2)  
   - HTTP Method: POST  
   - Path: `convert-timestamp`  
   - Response Mode: `responseNode` (to delegate response to a downstream node)

2. **Add Set Node:**  
   - Name: `Convert to ISO 8601`  
   - Type: Set (version 3.4)  
   - Configuration:  
     - Add new field: `convertedTime` (type: string)  
     - Value (Expression): `={{ new Date($json.body.timestamp * 1000).toISOString() }}`  
   - Connect the output of `Receive Timestamp Webhook` to this node.

3. **Add Respond to Webhook Node:**  
   - Name: `Respond with Converted Time`  
   - Type: Respond to Webhook (version 1.2)  
   - Configuration:  
     - Respond with: `allIncomingItems` (to send the entire JSON including `convertedTime`)  
   - Connect the output of `Convert to ISO 8601` node to this node.

4. **(Optional) Add Sticky Notes for Documentation:**  
   - Add a sticky note near the webhook node describing the expected input JSON format (property `timestamp` in seconds).  
   - Add a sticky note near the Set node explaining the conversion logic from Unix timestamp to ISO 8601 string.  
   - Add a sticky note near the response node clarifying the response content.

5. **Save and Activate Workflow:**  
   - Ensure the workflow is activated to start listening for incoming webhook requests.

6. **Credential Configuration:**  
   - No credentials are required since this is an open webhook and no external API calls are made.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                           |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| The workflow handles only timestamps in seconds. Unix timestamps in milliseconds must be converted externally before sending. | Workflow design constraint                |
| For testing, use tools like `curl` or Postman to POST JSON payloads such as `{"timestamp": 1678886400}` to the webhook URL. | Testing instructions                      |
| ISO 8601 strings produced are in UTC timezone with millisecond precision, e.g., `2023-03-15T00:00:00.000Z`.            | Output format specification               |

---

**Disclaimer:** The provided text results solely from an automated workflow created with n8n, fully compliant with current content policies and free of any illegal or protected elements. All processed data are legal and public.