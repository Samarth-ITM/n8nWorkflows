Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration

https://n8nworkflows.xyz/workflows/get-real-time-security-insights-with-nixguard-rag-and-wazuh-integration-4693


# Get Real-Time Security Insights with NixGuard RAG and Wazuh Integration

### 1. Workflow Overview

This workflow is designed to integrate NixGuard’s Retrieval-Augmented Generation (RAG) AI with Wazuh security data, delivering real-time actionable security insights. It targets cybersecurity teams and automated monitoring systems that require correlating multiple security data sources and generating intelligent responses to security queries.

The workflow is composed of three main logical blocks:

- **1.1 Trigger and Input Preparation**: Receives security queries via triggers and prepares unified input data.
- **1.2 Data Aggregation and Combination**: Merges and aggregates security data from various sources into a single payload.
- **1.3 NixGuard API Interaction and Response Processing**: Sends the aggregated data as a prompt to NixGuard’s API, processes the response, and formats the output for end users.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Preparation

**Overview:**  
This block handles the reception of security-related queries or triggers, preparing the foundational data for the workflow. It supports different trigger types (chat messages, manual triggers) and formats initial parameters such as API keys and session information.

**Nodes Involved:**  
- When chat message received (disabled)  
- Execute Workflow Trigger  
- Prepare API Request Data (disabled)  
- Merge Input Data

**Node Details:**

- **When chat message received**  
  - *Type:* Chat Trigger (n8n-nodes-langchain.chatTrigger)  
  - *Role:* Intended to receive chat-based security queries as triggers.  
  - *Configuration:* Disabled in this workflow; webhook ID set for future use.  
  - *Inputs:* External chat messages (when enabled)  
  - *Outputs:* JSON data including `apiKey`, `chatInput`, `sessionId`, `action`  
  - *Failure Modes:* Disabled; if enabled, potential failures include webhook misconfiguration or invalid chat payloads.  
  - *Sub-workflow:* None

- **Execute Workflow Trigger**  
  - *Type:* Execute Workflow Trigger (n8n-nodes-base.executeWorkflowTrigger)  
  - *Role:* Acts as a manual or programmatic trigger to start the workflow.  
  - *Configuration:* Default; no parameters set.  
  - *Inputs:* External manual trigger or API call  
  - *Outputs:* Passes trigger data downstream  
  - *Failure Modes:* None expected unless upstream trigger fails

- **Prepare API Request Data**  
  - *Type:* Set Node (n8n-nodes-base.set)  
  - *Role:* Intended to format and prepare essential API request parameters such as `apiKey`, `sessionId`, `action`, and `chatInput`.  
  - *Configuration:* Disabled; contains placeholder for API key (empty string) and copies other fields from input JSON.  
  - *Inputs:* Trigger data from chat or manual trigger  
  - *Outputs:* Structured JSON with API request fields  
  - *Failure Modes:* Disabled; if enabled, failure could occur if required fields are missing or empty.

- **Merge Input Data**  
  - *Type:* Merge Node (n8n-nodes-base.merge)  
  - *Role:* Combines inputs from different trigger nodes into one unified JSON object for processing.  
  - *Configuration:* Default merge settings, no specific keys set.  
  - *Inputs:* Output from Execute Workflow Trigger and potentially Prepare API Request Data  
  - *Outputs:* Merged JSON data downstream to aggregation node  
  - *Failure Modes:* Potential issues if input data formats differ significantly or are empty

---

#### 1.2 Data Aggregation and Combination

**Overview:**  
This block consolidates security event data, potentially from Wazuh and other sources, and prepares a unified payload to be sent as a prompt to the NixGuard API. It aggregates multiple security events and merges their data into a comprehensive object.

**Nodes Involved:**  
- Aggregate Security Data  
- Combine Security Data

**Node Details:**

- **Aggregate Security Data**  
  - *Type:* Aggregate Node (n8n-nodes-base.aggregate)  
  - *Role:* Aggregates all incoming items into a single combined dataset for unified processing.  
  - *Configuration:* Uses `aggregateAllItemData` option to aggregate all item data.  
  - *Inputs:* Merged input data from previous block  
  - *Outputs:* Aggregated array of security data objects  
  - *Failure Modes:* May fail if input data is malformed or too large; performance impact if data volume is high.

- **Combine Security Data**  
  - *Type:* Code Node (n8n-nodes-base.code)  
  - *Role:* Merges properties from multiple security data objects into a single object to be sent as the request payload.  
  - *Configuration:* Uses JavaScript `Object.assign` to combine all objects in the aggregated data array into one.  
  - *Inputs:* Aggregated data from previous node  
  - *Outputs:* Unified JSON object containing combined security data  
  - *Failure Modes:* Could fail if input is not an array or contains non-object items.

---

#### 1.3 NixGuard API Interaction and Response Processing

**Overview:**  
This block handles the interaction with the NixGuard API, sending the constructed security query and receiving AI-generated insights. It parses the API response and formats it for downstream consumption.

**Nodes Involved:**  
- Send Request to NixGuard API  
- Parse NixGuard Response  
- Format API Response  
- Prepare Final Output

**Node Details:**

- **Send Request to NixGuard API**  
  - *Type:* HTTP Request (n8n-nodes-base.httpRequest)  
  - *Role:* Sends a POST request to the NixGuard API endpoint, authenticating via API key and sending the security query prompt.  
  - *Configuration:*  
    - URL: `https://nix.thenex.world`  
    - Method: POST  
    - Headers: `Content-Type: application/json`  
    - Body: JSON containing `apiKey` and `prompt` extracted from input JSON fields  
  - *Inputs:* Combined security data object from previous block  
  - *Outputs:* Raw JSON response from NixGuard API  
  - *Failure Modes:*  
    - Network timeouts or unreachable endpoint  
    - Invalid/expired API key resulting in authentication errors  
    - Malformed request body or headers leading to 4xx errors  
    - API rate limits exceeded

- **Parse NixGuard Response**  
  - *Type:* Code Node (n8n-nodes-base.code)  
  - *Role:* Parses the raw JSON string response from the API into a usable JavaScript object and extracts the `result` field.  
  - *Configuration:* Uses `JSON.parse` on `$input.first().json.data`.  
  - *Inputs:* Raw API response from HTTP Request node  
  - *Outputs:* Parsed JSON object with AI-generated security insights  
  - *Failure Modes:* Parsing errors if the response is not valid JSON or is empty

- **Format API Response**  
  - *Type:* Set Node (n8n-nodes-base.set)  
  - *Role:* Extracts the relevant `content` field from the parsed response to prepare it for final output.  
  - *Configuration:* Assigns `content` field from input JSON to output.  
  - *Inputs:* Parsed NixGuard response  
  - *Outputs:* Structured JSON with AI-generated content  
  - *Failure Modes:* Missing or undefined `content` field in response

- **Prepare Final Output**  
  - *Type:* Code Node (n8n-nodes-base.code)  
  - *Role:* Wraps the extracted content into a final output object for downstream consumption or display.  
  - *Configuration:* Returns JSON with property `output` containing the content string.  
  - *Inputs:* Formatted API response content  
  - *Outputs:* Final structured output JSON  
  - *Failure Modes:* Input content missing or empty

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                            | Input Node(s)                 | Output Node(s)             | Sticky Note                                                                                                                     |
|-----------------------------|----------------------------------|--------------------------------------------|------------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| When chat message received   | Chat Trigger                     | Receive chat-based security queries (disabled) | External webhook              | Prepare API Request Data    |                                                                                                                                |
| Execute Workflow Trigger     | Execute Workflow Trigger         | Manual or programmatic workflow start      | External trigger             | Merge Input Data            |                                                                                                                                |
| Prepare API Request Data     | Set                             | Format API request parameters (disabled)  | When chat message received    | Merge Input Data            |                                                                                                                                |
| Merge Input Data             | Merge                           | Combine inputs from triggers                | Execute Workflow Trigger, Prepare API Request Data | Aggregate Security Data     | See sticky note "Data Aggregation"                                                                                             |
| Aggregate Security Data      | Aggregate                       | Aggregate security event data               | Merge Input Data              | Combine Security Data       | See sticky note "Data Aggregation"                                                                                             |
| Combine Security Data        | Code                            | Combine aggregated security objects         | Aggregate Security Data       | Send Request to NixGuard API | See sticky note "Data Aggregation"                                                                                             |
| Send Request to NixGuard API | HTTP Request                    | Send security query to NixGuard API         | Combine Security Data         | Parse NixGuard Response     | See sticky note "API Request Configuration"                                                                                    |
| Parse NixGuard Response      | Code                            | Parse JSON response from NixGuard API       | Send Request to NixGuard API  | Format API Response         | See sticky note "Response Processing"                                                                                          |
| Format API Response          | Set                             | Extract relevant content from response      | Parse NixGuard Response       | Prepare Final Output        | See sticky note "Response Processing"                                                                                          |
| Prepare Final Output         | Code                            | Structure final output for end user         | Format API Response           |                            | See sticky note "Response Processing"                                                                                          |
| Data Aggregation             | Sticky Note                     | Documentation of aggregation block          |                              |                            | Describes the logic of merging, aggregating, and combining security data                                                      |
| Setup Guide                 | Sticky Note                     | Setup and prerequisites guide                |                              |                            | Includes prerequisites, setup steps, and links to documentation and community                                                 |
| Workflow Overview1           | Sticky Note                     | High-level workflow description               |                              |                            | Describes workflow purpose, key features, and authentication requirements                                                    |
| API Request Explanation1     | Sticky Note                     | Explains API request node configuration       |                              |                            | Details headers, body format, and endpoint for sending queries                                                                |
| Response Processing1         | Sticky Note                     | Explains response handling nodes               |                              |                            | Describes parsing, formatting, and error handling of API responses                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add an **Execute Workflow Trigger** node to act as the main workflow starter. No special configuration required.  
   - Optionally, add a **When chat message received** node (type: Chat Trigger) if chat-based input is needed. Configure webhook ID and enable it if chat interaction is desired.

2. **Set Up Input Preparation**  
   - Add a **Set** node named `Prepare API Request Data`.  
     - Add fields:  
       - `apiKey` (string): set to your NixGuard API key (must be non-empty)  
       - `sessionId` (string): map from incoming data (e.g., `{{$json.sessionId}}`)  
       - `action` (string): map from incoming data (e.g., `{{$json.action}}`)  
       - `chatInput` (string): map from incoming data (e.g., `{{$json.chatInput}}`)  
     - Initially disable this node if not used.

3. **Merge Inputs**  
   - Add a **Merge** node named `Merge Input Data`.  
   - Connect the outputs from the trigger nodes (`Execute Workflow Trigger` and optionally `Prepare API Request Data`) into this node to combine input data streams.

4. **Aggregate Security Data**  
   - Add an **Aggregate** node named `Aggregate Security Data`.  
   - Configure to aggregate all item data (`aggregateAllItemData`).  
   - Connect input from `Merge Input Data`.

5. **Combine Security Data**  
   - Add a **Code** node named `Combine Security Data`.  
   - Enter JavaScript code:  
     ```js
     const combinedObject = Object.assign({}, ...$input.first().json.data);
     return [ { json: combinedObject } ];
     ```  
   - Connect input from `Aggregate Security Data`.

6. **Configure HTTP Request to NixGuard API**  
   - Add an **HTTP Request** node named `Send Request to NixGuard API`.  
   - Set:  
     - Method: POST  
     - URL: `https://nix.thenex.world`  
     - Headers: `Content-Type: application/json`  
     - Body Parameters (JSON):  
       - `apiKey`: map from incoming JSON field `apiKey`  
       - `prompt`: map from incoming JSON field `chatInput`  
   - Connect input from `Combine Security Data`.  
   - Enable sending headers and body as JSON.

7. **Parse NixGuard API Response**  
   - Add a **Code** node named `Parse NixGuard Response`.  
   - JavaScript code:  
     ```js
     const nixResponse = JSON.parse($input.first().json.data);
     // Optionally extract nixResponse.result if needed
     return [ { json: nixResponse } ];
     ```  
   - Connect input from `Send Request to NixGuard API`.

8. **Format API Response**  
   - Add a **Set** node named `Format API Response`.  
   - Assign the field `content` with value: `{{$json["content"]}}` (extracted from parsed response).  
   - Connect input from `Parse NixGuard Response`.

9. **Prepare Final Output**  
   - Add a **Code** node named `Prepare Final Output`.  
   - JavaScript code:  
     ```js
     const output = items[0].json.content;
     return [{ json: { output } }];
     ```  
   - Connect input from `Format API Response`.

10. **Add Sticky Notes for Documentation**  
    - Add sticky notes describing:  
      - Workflow overview and authentication needs  
      - Data aggregation logic  
      - API request configuration details  
      - Response processing and error handling  
      - Setup instructions and prerequisites including API key and Wazuh data access  
    - Position them near relevant nodes for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Valid NixGuard API key and access to Wazuh security data are prerequisites for operation.                                   | Setup Guide sticky note                                                                           |
| NixGuard Documentation and Community Discord are available for support and additional information.                         | [NixGuard Documentation](https://nixguard.thenex.world), [Community Discord](https://discord.com/invite/ajCYwYCwHb) |
| Workflow integrates AI-powered security query processing with real-time security data aggregation to enhance incident response. | Workflow Overview1 sticky note                                                                   |
| API requests require proper JSON formatting and authentication headers to communicate with NixGuard endpoint securely.     | API Request Explanation1 sticky note                                                             |
| Response processing includes JSON parsing and content extraction; error handling should be implemented in extended workflows.| Response Processing1 sticky note                                                                 |

---

**Disclaimer:**  
The provided text derives exclusively from an automated n8n workflow configuration. It complies fully with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.