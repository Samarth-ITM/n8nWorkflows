BIN Code Lookup with apilayer API

https://n8nworkflows.xyz/workflows/bin-code-lookup-with-apilayer-api-6306


# BIN Code Lookup with apilayer API

### 1. Workflow Overview

This workflow is designed to perform a BIN (Bank Identification Number) code lookup using the apilayer API. Its primary use case is to retrieve detailed information about a BIN code, which typically identifies issuing banks or financial institutions linked to credit or debit cards. The workflow logically consists of three main blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Parameter Setup:** Setting the BIN code and API key needed to query the external API.
- **1.3 API Request and Response:** Executing the HTTP request to apilayer's BIN lookup endpoint and retrieving the response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This initial block triggers the workflow execution manually. It serves as the entry point and activates the subsequent nodes.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Node Name:** When clicking ‘Execute workflow’  
  - **Type:** Manual Trigger  
  - **Technical Role:** Initiates workflow execution on user command.  
  - **Configuration Choices:** No parameters set; default manual trigger.  
  - **Key Expressions / Variables:** None  
  - **Input Connections:** None (start node)  
  - **Output Connections:** Connects to "Set API Key & BIN Number to Lookup" node.  
  - **Version Requirements:** Compatible with n8n versions supporting manual triggers (all recent versions).  
  - **Edge Cases / Potential Failures:** None, as this is a trigger node.  
  - **Sub-workflow Reference:** None

#### 1.2 Parameter Setup

- **Overview:**  
  This block sets the input parameters required for the API call: the BIN code to look up and the API key for authentication.

- **Nodes Involved:**  
  - Set API Key & BIN Number to Lookup

- **Node Details:**  
  - **Node Name:** Set API Key & BIN Number to Lookup  
  - **Type:** Set Node  
  - **Technical Role:** Assigns workflow variables for subsequent HTTP request.  
  - **Configuration Choices:**  
    - Sets two string fields:  
      - `bin_code`: Hardcoded example value "JH4KA7560RC003647" (should be replaced with the actual BIN code to query).  
      - `apikey`: Placeholder for the user’s apilayer API key (empty string by default).  
  - **Key Expressions / Variables:** None; static string assignments.  
  - **Input Connections:** Receives input from manual trigger node.  
  - **Output Connections:** Connects to "Lookup BIN" HTTP request node.  
  - **Version Requirements:** Uses Set node version 3.4 (ensure n8n supports this or higher).  
  - **Edge Cases / Potential Failures:**  
    - Missing or invalid API key will cause errors in the HTTP request.  
    - Incorrect BIN code format may yield no results or API errors.  
  - **Sub-workflow Reference:** None

#### 1.3 API Request and Response

- **Overview:**  
  This block performs the HTTP GET request to the apilayer BIN lookup API using parameters set in the previous block and fetches the data for the specified BIN code.

- **Nodes Involved:**  
  - Lookup BIN

- **Node Details:**  
  - **Node Name:** Lookup BIN  
  - **Type:** HTTP Request  
  - **Technical Role:** Sends a GET request to the external apilayer API to retrieve BIN information.  
  - **Configuration Choices:**  
    - HTTP Method: GET  
    - URL: Dynamically constructed using expression `https://api.apilayer.com/bincheck/{{ $json.bin_code }}`  
    - Headers: Includes `apiKey` header set via expression `{{ $json.apikey }}`  
    - No additional options or authentication set beyond the header.  
  - **Key Expressions / Variables:**  
    - URL uses `bin_code` from previous node.  
    - Header `apiKey` uses `apikey` from previous node.  
  - **Input Connections:** Receives input from the Set node.  
  - **Output Connections:** None (end node)  
  - **Version Requirements:** Uses HTTP Request node version 4.2; ensure compatibility.  
  - **Edge Cases / Potential Failures:**  
    - HTTP errors due to invalid or missing API key (401 Unauthorized).  
    - Network or timeout errors.  
    - API may return empty or error payload if BIN code is invalid or not found.  
    - Expression failures if input variables are missing or malformed.  
  - **Sub-workflow Reference:** None

---

### 3. Summary Table

| Node Name                      | Node Type       | Functional Role                     | Input Node(s)                 | Output Node(s)                      | Sticky Note                                                                                       |
|-------------------------------|-----------------|-----------------------------------|------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger  | Starts the workflow manually      | None                         | Set API Key & BIN Number to Lookup | ## BIN Code Lookup with apilayer API Steps in n8n: Step 1: Manual Trigger - Node Type: Manual Trigger - Purpose: Starts the workflow manually |
| Set API Key & BIN Number to Lookup | Set             | Assigns BIN code and API Key      | When clicking ‘Execute workflow’ | Lookup BIN                        | Step 2: Set BIN Code and API Key - Node Type: Set - Fields to set: bin_code, apikey               |
| Lookup BIN                    | HTTP Request    | Performs the BIN lookup via API   | Set API Key & BIN Number to Lookup | None                             | Step 3: HTTP Request - Method: GET - URL: https://api.apilayer.com/bincheck/{{ $json.bin_code }} - Headers: apiKey header with API key       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: Start the workflow manually  
   - No special parameters needed  
   - Position: (0,0) or preferred location

2. **Create Set Node for Parameters**  
   - Node Type: Set  
   - Node Name: "Set API Key & BIN Number to Lookup"  
   - Add two string fields under "Values to Set":  
     - `bin_code`: Set to a sample BIN code or leave empty for dynamic input (e.g., "JH4KA7560RC003647")  
     - `apikey`: Set your apilayer API key here (string)  
   - Connect Manual Trigger node output to this Set node input  
   - Ensure the Set node version supports these features (v3.4 or later)

3. **Create HTTP Request Node for BIN Lookup**  
   - Node Type: HTTP Request  
   - Node Name: "Lookup BIN"  
   - HTTP Method: GET  
   - URL: Use expression mode and enter:  
     `https://api.apilayer.com/bincheck/{{ $json.bin_code }}`  
   - Under Headers, add a header:  
     - Name: `apiKey`  
     - Value: Use expression mode: `{{ $json.apikey }}`  
   - Connect output of Set node to input of this HTTP Request node  
   - Use HTTP Request node version 4.2 or later

4. **Save and Activate Workflow**  
   - Test execution by manually triggering the workflow  
   - Inspect the output of the HTTP Request node for BIN lookup results

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                    |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow uses the apilayer BIN check API; an API key is required from https://apilayer.com/ | Official apilayer BIN check API documentation is recommended for further details on response schema and rate limits. |
| The BIN code used in Set node is a placeholder; replace with actual BIN codes or make dynamic input | For production use, consider replacing the Set node with input parameters or integrations to fetch BIN codes dynamically. |

---

**Disclaimer:** The text provided is solely derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All data processed is legal and publicly accessible.