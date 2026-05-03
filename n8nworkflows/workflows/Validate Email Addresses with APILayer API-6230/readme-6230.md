Validate Email Addresses with APILayer API

https://n8nworkflows.xyz/workflows/validate-email-addresses-with-apilayer-api-6230


# Validate Email Addresses with APILayer API

### 1. Workflow Overview

This workflow validates email addresses using the APILayer email verification API. It is designed for on-demand use, allowing users to input an email and an API access key, then send a request to APILayer to verify the email’s validity. The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Triggering the workflow manually.
- **1.2 Data Preparation:** Setting the email address and API access key dynamically.
- **1.3 API Request Execution:** Constructing and sending the HTTP request to APILayer and receiving the validation response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow manually, enabling execution at any time on demand.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Node Name:** When clicking ‘Execute workflow’  
    - Type: Manual Trigger  
    - Purpose: Start the workflow manually by user action.  
    - Configuration: No parameters; standard manual trigger.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Set Email & Access Key" node.  
    - Edge cases: None expected; manual trigger may require user to initiate.  
    - Version specific: Standard manual trigger node, version 1.

#### 2.2 Data Preparation

- **Overview:**  
  This block assigns the email address to validate and the APILayer API access key. These values are then passed forward to build the API request.

- **Nodes Involved:**  
  - Set Email & Access Key

- **Node Details:**  
  - **Node Name:** Set Email & Access Key  
    - Type: Set node  
    - Purpose: Define variables for `email` and `access_key` used in the API call.  
    - Configuration: Two string fields assigned —  
      - `email`: The email address to validate (user input required here).  
      - `access_key`: The APILayer API access key (user input required here).  
    - Key expressions: Static string assignment; no expressions in the current config.  
    - Inputs: Receives trigger from manual execution node.  
    - Outputs: Passes JSON containing `email` and `access_key` to the HTTP Request node.  
    - Edge cases: Empty or invalid `email` or `access_key` will cause API errors downstream.  
    - Version specific: Set node version 3.4.

#### 2.3 API Request Execution

- **Overview:**  
  This block constructs the API request URL dynamically using the variables set previously and sends the GET request to APILayer’s email validation endpoint.

- **Nodes Involved:**  
  - Make Request to APILayer

- **Node Details:**  
  - **Node Name:** Make Request to APILayer  
    - Type: HTTP Request node  
    - Purpose: Perform a GET request to APILayer’s email verification API.  
    - Configuration:  
      - URL constructed dynamically with expressions:  
        `https://apilayer.net/api/check?access_key={{ $json.access_key }}&email={{ $json.email }}`  
      - HTTP Method: GET (default)  
      - No additional headers or body.  
    - Inputs: Receives JSON with `email` and `access_key` from Set node.  
    - Outputs: Returns API response containing validation results (JSON).  
    - Edge cases:  
      - HTTP errors (e.g., 401 Unauthorized if access key invalid).  
      - Network timeouts or connectivity issues.  
      - API rate limiting or quota exceeded responses.  
      - Invalid email format errors returned from API.  
    - Version specific: HTTP Request node version 4.2.

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role              | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                         |
|---------------------------|-------------------|------------------------------|----------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger    | Workflow start trigger        | —                          | Set Email & Access Key    | The manual trigger allows for execution on-demand.                                                                  |
| Set Email & Access Key     | Set               | Assign email and API key      | When clicking ‘Execute workflow’ | Make Request to APILayer | The "Edit Fields" node sets the email and uses a specific access key for authentication from "https://apilayer.com/" |
| Make Request to APILayer   | HTTP Request      | Call APILayer email API       | Set Email & Access Key      | —                        | The HTTP Request node dynamically constructs the URL using these set fields.                                        |
| Sticky Note               | Sticky Note       | Documentation and notes       | —                          | —                        | ## Validate Email Addresses with APILayer API Notes: - The "Edit Fields" node sets the email and uses a specific access key for authentication from "https://apilayer.com/" - The HTTP Request node dynamically constructs the URL using these set fields. - The manual trigger allows for execution on-demand. - The API response can be further processed or stored based on user needs. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name it `When clicking ‘Execute workflow’`.  
   - No configuration needed.

3. **Add a Set node:**  
   - Name it `Set Email & Access Key`.  
   - Under the "Values to Set" section, add two string fields:  
     - `email` (leave empty for user input or fill with test email).  
     - `access_key` (leave empty for user input or fill with your APILayer API key).  
   - Connect the Manual Trigger node’s output to this Set node’s input.

4. **Add an HTTP Request node:**  
   - Name it `Make Request to APILayer`.  
   - Set HTTP Method to `GET`.  
   - For the URL, enter the expression mode (click the gears icon, then "Add Expression") and input:  
     `https://apilayer.net/api/check?access_key={{ $json.access_key }}&email={{ $json.email }}`  
   - No authentication configured here because the access key is included in the URL.  
   - Connect the Set node’s output to this HTTP Request node’s input.

5. **(Optional) Add nodes downstream to process the response if needed.**

6. **Save and activate the workflow.**

7. **Run the workflow manually by clicking 'Execute workflow'.**  
   - Input the email and access key in the Set node before execution or modify the Set node to read from an external source.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------|
| This workflow requires a valid API key from APILayer, obtainable at https://apilayer.com/.                                                               | APILayer API documentation      |
| The API response includes detailed validation information such as format validity, MX record status, and SMTP check results.                            | APILayer Email Validation API   |
| The workflow is designed for manual execution but can be integrated with other triggers or data sources for automation.                                  | n8n Workflow flexibility         |
| Ensure network connectivity and that API rate limits are respected to avoid request failures.                                                            | APILayer usage policies          |

---

**Disclaimer:** The text provided is based exclusively on a workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.