Simple SMS Messaging System with Textbelt API Integration

https://n8nworkflows.xyz/workflows/simple-sms-messaging-system-with-textbelt-api-integration-6228


# Simple SMS Messaging System with Textbelt API Integration

### 1. Workflow Overview

This workflow, titled **"Simple SMS Messaging System with Textbelt API Integration"**, is designed for sending SMS messages through the Textbelt API. It is intended for users who want to manually trigger SMS sending by specifying a recipient phone number, a message, and an API key, then dispatch the SMS via a simple HTTP POST request.

The workflow consists of three main logical blocks:

- **1.1 Manual Trigger:** Initiates the workflow manually by user action.
- **1.2 Data Preparation:** Defines and sets the necessary input parameters for the SMS (phone number, message content, API key).
- **1.3 SMS Sending via API:** Sends the prepared data to the Textbelt API using an HTTP POST request and handles the response.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger Block

- **Overview:**  
  This block starts the workflow execution manually when the user clicks the ‘Execute workflow’ button in the n8n interface.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  

  - **Node Name:** When clicking ‘Execute workflow’  
  - **Type:** Manual Trigger  
  - **Technical Role:** Entry point node for manually initiating the workflow  
  - **Configuration Choices:** No parameters configured; default manual trigger behavior  
  - **Key Expressions/Variables:** None  
  - **Input Connections:** None (start node)  
  - **Output Connections:** Connects to the “Set Data” node  
  - **Version Requirements:** n8n version supporting Manual Trigger nodes (standard feature)  
  - **Potential Failures:** None typical; only manual user action required

#### 2.2 Data Preparation Block

- **Overview:**  
  This block sets up the SMS sending parameters: the phone number, message text, and Textbelt API key. It provides placeholders for these values, which the user must populate before executing.

- **Nodes Involved:**  
  - Set Data

- **Node Details:**  

  - **Node Name:** Set Data  
  - **Type:** Set  
  - **Technical Role:** Defines and assigns static values or variables for SMS sending parameters  
  - **Configuration Choices:**  
    - Assigns three string fields:  
      - `phone` (empty string by default)  
      - `message` (empty string by default)  
      - `key` (empty string by default)  
    - These fields are intended to be manually filled before execution or replaced by dynamic input if extended  
  - **Key Expressions/Variables:** None by default; values are static empty strings (`""`)  
  - **Input Connections:** Receives trigger from “When clicking ‘Execute workflow’”  
  - **Output Connections:** Connects to “HTTP Request” node  
  - **Version Requirements:** Compatible with Set node features in n8n version 3.4+  
  - **Potential Failures:**  
    - Failure if required fields remain empty at runtime (no validation present)  
    - Potential misconfiguration if non-string or malformed values are inserted

#### 2.3 SMS Sending via API Block

- **Overview:**  
  Sends the SMS by making a POST HTTP request to Textbelt's API endpoint with the prepared parameters. It handles the API call and receives confirmation or error response.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  

  - **Node Name:** HTTP Request  
  - **Type:** HTTP Request  
  - **Technical Role:** Executes a POST request to the Textbelt API endpoint to send the SMS  
  - **Configuration Choices:**  
    - **URL:** `https://textbelt.com/tex` (Note: The correct Textbelt endpoint is typically `https://textbelt.com/text` — this may be a typo)  
    - **Method:** POST  
    - **Body Parameters:** Sent as form parameters  
      - `phone`: Value taken from the incoming JSON (`{{$json.phone}}`)  
      - `message`: Value taken from the incoming JSON (`{{$json.message}}`)  
      - `key`: Value taken from the incoming JSON (`{{$json.key}}`)  
    - **Send Body:** Enabled to include parameters in the request body  
  - **Key Expressions/Variables:**  
    - Uses expressions to inject dynamic values for phone, message, and API key from prior node output  
  - **Input Connections:** Receives data from “Set Data” node  
  - **Output Connections:** None (end node)  
  - **Version Requirements:** HTTP Request node version 4.2 features used; standard in recent n8n versions  
  - **Potential Failures:**  
    - HTTP errors (e.g., network issues, endpoint unavailable)  
    - API errors like invalid API key, phone number format errors, message length limits  
    - Typo in URL (`/tex` vs `/text`) may cause 404 errors or failure to send  
    - No error handling or retries configured in current workflow

---

### 3. Summary Table

| Node Name                   | Node Type       | Functional Role                    | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                                                     |
|-----------------------------|-----------------|----------------------------------|------------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger  | Starts workflow manually          | None                         | Set Data                | ## Sending SMS via Textbelt API Step: 1. Manual Trigger: Starts the workflow manually by clicking ‘Execute workflow’.          |
| Set Data                    | Set             | Defines SMS parameters (phone, message, key) | When clicking ‘Execute workflow’ | HTTP Request            | 2. Set Data Node: Defines the required input parameters (phone, message, and key) that will be sent to the SMS API.           |
| HTTP Request                | HTTP Request    | Sends SMS via Textbelt API        | Set Data                     | None                    | 3. HTTP Request Node: Sends a POST request to https://textbelt.com/tex with the phone number, message, and API key in the body. The response confirms success.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Add a node of type **Manual Trigger**.  
   - Leave default settings. This node will start the workflow on manual execution.  

2. **Create a Set Node:**  
   - Add a node of type **Set**.  
   - Configure three new fields with type **String**:  
     - `phone` (leave value empty by default; user will enter phone number)  
     - `message` (leave value empty; user will enter SMS text)  
     - `key` (leave value empty; user will enter Textbelt API key)  
   - Connect output of **Manual Trigger** node to input of this **Set** node.

3. **Create an HTTP Request Node:**  
   - Add a node of type **HTTP Request**.  
   - Configure:  
     - **HTTP Method:** POST  
     - **URL:** `https://textbelt.com/tex` (note: verify endpoint correctness; typically `https://textbelt.com/text`)  
     - **Send Body:** Enable to send form parameters  
     - **Body Parameters:** Add three parameters with these names and values:  
       - `phone` = `={{ $json.phone }}` (expression referencing Set node output)  
       - `message` = `={{ $json.message }}`  
       - `key` = `={{ $json.key }}`  
   - Connect output of **Set** node to input of **HTTP Request** node.

4. **Set Credentials:**  
   - No special credentials node needed because Textbelt uses API key sent in the body.  
   - Ensure the API key is correctly entered in the `key` field in the Set node before execution.

5. **Save and Test:**  
   - Save the workflow.  
   - Populate the `phone`, `message`, and `key` fields in the **Set Data** node with valid values before executing.  
   - Click ‘Execute workflow’ to trigger the SMS sending process.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow is a simple example to demonstrate sending SMS using the Textbelt API. Users must ensure they have a valid API key from Textbelt.               | https://textbelt.com/                               |
| The Textbelt endpoint URL in the workflow is `https://textbelt.com/tex` which appears to be a typo; the correct endpoint is usually `https://textbelt.com/text` | Textbelt API documentation                          |
| No error handling or retries are implemented; for production use, consider adding error capture, validation, and retry mechanisms to ensure robustness.       | Best practices in API integrations                   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.