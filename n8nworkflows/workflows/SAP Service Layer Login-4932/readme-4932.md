SAP Service Layer Login

https://n8nworkflows.xyz/workflows/sap-service-layer-login-4932


# SAP Service Layer Login

### 1. Workflow Overview

This workflow, titled **"SAP Service Layer Login"**, is designed to authenticate and establish a session with the SAP Service Layer API. It targets use cases where automated login to SAP is required to subsequently perform operations on SAP resources via API calls. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow execution.
- **1.2 Credential Setup:** Setting up the SAP connection parameters such as URL, username, password, and company database.
- **1.3 SAP Authentication Request:** Sending a POST request to SAP Service Layer's login endpoint.
- **1.4 Response Handling:** Branching logic to handle success (extracting session ID) or failure (capturing error details).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initializes the workflow execution manually, allowing users to trigger the SAP login process on demand.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Node Name:** When clicking ‘Execute workflow’  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters; simply triggers workflow execution when manually clicked.  
  - **Expressions/Variables:** None.  
  - **Input/Output Connections:** Output connected to "Set Login Data".  
  - **Version Specifics:** n8n core standard node, version 1.  
  - **Potential Failures:** None expected; manual trigger inherently stable.  

#### 2.2 Credential Setup

- **Overview:**  
  This block prepares the necessary login parameters for SAP Service Layer authentication, including the URL, username, password, and company database. These are set as workflow variables.

- **Nodes Involved:**  
  - Set Login Data

- **Node Details:**  
  - **Node Name:** Set Login Data  
  - **Type:** Set Node  
  - **Configuration:** Four string variables assigned without default values:  
    - `sap_url` (SAP Service Layer base URL, e.g., `https://my.sapserver:50000/b1s/v1/`)  
    - `sap_username` (SAP user login)  
    - `sap_password` (SAP user password)  
    - `sap_companydb` (SAP Company database name)  
  - **Expressions/Variables:** Values are static strings to be filled by the user before execution.  
  - **Input/Output Connections:** Input from manual trigger; output to "SAP Connection" HTTP node.  
  - **Version Specifics:** Uses Set node version 3.4.  
  - **Potential Failures:** Empty or incorrect credentials will cause login failure downstream.  

#### 2.3 SAP Authentication Request

- **Overview:**  
  This block performs the actual login request to SAP Service Layer by sending a POST request with the login data, and branches the workflow depending on the HTTP response outcome.

- **Nodes Involved:**  
  - SAP Connection

- **Node Details:**  
  - **Node Name:** SAP Connection  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - URL: `={{ $json.sap_url }}Login` — dynamically appends "Login" to the SAP base URL.  
    - Method: POST  
    - JSON Body:  
      ```json
      {
        "UserName": "{{ $json.sap_username }}",
        "Password": "{{ $json.sap_password }}",
        "CompanyDB": "{{ $json.sap_companydb }}"
      }
      ```  
    - Options: Allows unauthorized certificates (`allowUnauthorizedCerts: true`) — useful for self-signed certs in SAP environments.  
    - Send Body: true; body format JSON.  
  - **Expressions/Variables:** Uses JSON variables from "Set Login Data" node.  
  - **Input/Output Connections:** Input from "Set Login Data"; output branches to "Success" (on HTTP success) and "Failed" (on error).  
  - **Version Specifics:** HTTP Request node version 4.2; error handling set to continue on error output.  
  - **Potential Failures:**  
    - Connection timeout or network errors.  
    - Invalid credentials leading to HTTP error response.  
    - SSL certificate errors (mitigated by allowUnauthorizedCerts).  
    - Malformed URL or missing parameters.  

#### 2.4 Response Handling

- **Overview:**  
  This block captures the outcomes of the SAP login request, separating successful authentication (extracting the session ID) from failures (capturing status code and error message).

- **Nodes Involved:**  
  - Success  
  - Failed

- **Node Details:**  

  - **Node Name:** Success  
    - **Type:** Set Node  
    - **Configuration:** Extracts `SessionId` from the SAP login response JSON and sets it as `sessionID`.  
    - **Expressions/Variables:** `={{ $json.SessionId }}`  
    - **Input/Output Connections:** Input from "SAP Connection" success output; no outputs connected further.  
    - **Version Specifics:** Set node version 3.4.  
    - **Potential Failures:** Missing or malformed `SessionId` in response JSON could cause empty values.  

  - **Node Name:** Failed  
    - **Type:** Set Node  
    - **Configuration:** Captures error details from the HTTP request error object:  
      - `statusCode` from `$json.error.status`  
      - `errorMessage` from `$json.error.message`  
    - **Expressions/Variables:** Uses error object properties from the HTTP Request node.  
    - **Input/Output Connections:** Input from "SAP Connection" error output; no further outputs.  
    - **Version Specifics:** Set node version 3.4.  
    - **Potential Failures:** If error object structure changes or is missing, variables may be undefined.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role               | Input Node(s)                | Output Node(s)           | Sticky Note                          |
|---------------------------|--------------------|------------------------------|-----------------------------|--------------------------|------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Start workflow execution       | -                           | Set Login Data           |                                    |
| Set Login Data            | Set                | Assign SAP login parameters    | When clicking ‘Execute workflow’ | SAP Connection           |                                    |
| SAP Connection            | HTTP Request       | Perform SAP Service Layer login| Set Login Data              | Success, Failed          |                                    |
| Success                   | Set                | Handle successful login output | SAP Connection (success)    | -                        |                                    |
| Failed                    | Set                | Handle failed login output     | SAP Connection (error)      | -                        |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - No configuration needed. This node starts the workflow manually.

2. **Create a Set node:**  
   - Name: `Set Login Data`  
   - Add four string fields with empty default values:  
     - `sap_url` (e.g., `https://your-sap-server:50000/b1s/v1/`)  
     - `sap_username`  
     - `sap_password`  
     - `sap_companydb`  
   - Connect output of `When clicking ‘Execute workflow’` to this node.

3. **Create an HTTP Request node:**  
   - Name: `SAP Connection`  
   - Set HTTP Method to `POST`.  
   - Set URL to expression: `={{ $json.sap_url }}Login` (concatenate base URL and "Login").  
   - Under "Options", enable `Allow Unauthorized Certs` (to handle self-signed SAP certs).  
   - Set "Send Body" to `true` and specify "JSON".  
   - Enter JSON body as expression:  
     ```json
     {
       "UserName": "{{ $json.sap_username }}",
       "Password": "{{ $json.sap_password }}",
       "CompanyDB": "{{ $json.sap_companydb }}"
     }
     ```  
   - Set "On Error" to `Continue On Error Output` to branch for error handling.  
   - Connect output of `Set Login Data` to this node.

4. **Create a Set node for success case:**  
   - Name: `Success`  
   - Add a string field `sessionID` with expression: `={{ $json.SessionId }}`  
   - Connect the **first output** (success) of `SAP Connection` to this node.

5. **Create a Set node for failure case:**  
   - Name: `Failed`  
   - Add two string fields:  
     - `statusCode` with expression: `={{ $json.error.status }}`  
     - `errorMessage` with expression: `={{ $json.error.message }}`  
   - Connect the **second output** (error) of `SAP Connection` to this node.

6. **Save and activate the workflow.**  
   - Before running, fill in the `Set Login Data` node with valid SAP connection details.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                        |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Allowing unauthorized certificates is critical for SAP environments with self-signed SSL.   | Security consideration for SAP Service Layer HTTPS    |
| SAP Service Layer API documentation: https://help.sap.com/docs/SAP_BUSINESS_ONE_SERVICE_LAYER | Official SAP docs for API endpoints and usage          |
| This workflow is inactive by default; activate before use.                                  | n8n workflow status                                     |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated n8n workflow designed for SAP Service Layer login. It complies fully with content policies and contains no illegal or protected information. All data handled is legal and public.