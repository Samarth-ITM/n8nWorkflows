Secure Web Form to Odoo CRM Lead Creation with UTM Tracking

https://n8nworkflows.xyz/workflows/secure-web-form-to-odoo-crm-lead-creation-with-utm-tracking-7289


# Secure Web Form to Odoo CRM Lead Creation with UTM Tracking

### 1. Workflow Overview

This workflow automates the secure creation of CRM leads in Odoo from data submitted via a web form, including tracking marketing campaign parameters (UTM tags). It is designed for organizations using Odoo CRM who want to capture leads with enriched marketing attribution data, ensuring data validation and secure webhook access.

**Logical blocks:**

- **1.1 Input Reception and Validation:** Receives incoming lead data securely via a webhook with header authentication, and validates required fields.
- **1.2 UTM Data Resolution:** Queries Odoo to resolve marketing UTM parameters (source, medium, campaign) to their internal IDs.
- **1.3 Data Consolidation:** Merges incoming lead data and resolved UTM IDs into a single, clean JSON object suitable for Odoo lead creation.
- **1.4 Lead Creation in Odoo:** Creates a new lead record in Odoo CRM using the consolidated data.
- **1.5 Response Handling:** Sends an HTTP 200 success response on lead creation or 400 bad request if validation fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

- **Overview:**  
  This block manages the secure reception of incoming lead data via an HTTP POST webhook with header authentication and performs validation on required fields.

- **Nodes Involved:**  
  - `Webhook - Lead Webform`  
  - `Required data missing?` (If node)  
  - `Bad Request` (Respond to Webhook)  
  - `Sticky Note1`

- **Node Details:**

  - **Webhook - Lead Webform**  
    - *Type:* Webhook  
    - *Role:* Receives HTTP POST requests at path `/lead-webform`.  
    - *Configuration:*  
      - HTTP method: POST  
      - Response Mode: Response Node (delays response until workflow completes)  
      - Authentication: Header Authentication requiring a secret token in headers (`x-webhook-token`)  
    - *Inputs:* External HTTP POST requests  
    - *Outputs:* Payload JSON from request body  
    - *Edge cases:* Missing or incorrect authentication header will reject request; unexpected payload shape; timeout if downstream nodes error.  
    - *Sticky note:* Example payload JSON illustrating expected fields (`firstname`, `lastname`, `email`, `phone`, `source`, `medium`, `campaign`, `notes`).

  - **Required data missing?**  
    - *Type:* If  
    - *Role:* Validates presence of mandatory fields: `firstname`, `lastname`, and `email` in the webhook payload.  
    - *Configuration:* Checks if any required field does not exist or is empty.  
    - *Inputs:* JSON from webhook node  
    - *Outputs:*  
      - True branch: missing required data → leads to `Bad Request` node  
      - False branch: all required data present → leads to UTM lookup nodes  
    - *Edge cases:* Payload without required fields triggers immediate 400 response.

  - **Bad Request**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends HTTP 400 response with no data when validation fails.  
    - *Configuration:* HTTP status code 400, no response body.

  - **Sticky Note1**  
    - Provides an example JSON payload for users to understand the expected input format.

---

#### 1.2 UTM Data Resolution

- **Overview:**  
  This block queries Odoo to retrieve internal IDs for marketing UTM parameters (`source`, `medium`, `campaign`) by their names, allowing proper CRM attribution.

- **Nodes Involved:**  
  - `UTM: Get Source ID`  
  - `UTM: Get Medium ID`  
  - `UTM: Get Campaign ID`  
  - `Source ID Validation`  
  - `Medium ID Validation`  
  - `Campaign ID Validation`  
  - `Sticky Note3`

- **Node Details:**

  - **UTM: Get Source ID**  
    - *Type:* Odoo node (custom resource)  
    - *Role:* Fetches UTM Source ID from Odoo by filtering on the `name` field with value from webhook payload (`body.source`).  
    - *Configuration:*  
      - Resource: custom  
      - Custom Resource: `utm.source`  
      - Operation: `getAll` (limit 1)  
      - Filter: name equals `{{ $json.body.source }}`  
      - Fields returned: `id`, `name`  
    - *Credentials:* Odoo API credentials  
    - *Error Handling:* On error, continues normal output to avoid workflow failure if UTM source not found.  
    - *Outputs:* Array of matching UTM source records or empty.

  - **UTM: Get Medium ID**  
    - *Type:* Odoo node (custom resource)  
    - *Role:* Fetches UTM Medium ID similarly by filtering `utm.medium` with `body.medium`.  
    - *Configuration:* Same as above, targeting `utm.medium`.

  - **UTM: Get Campaign ID**  
    - *Type:* Odoo node (custom resource)  
    - *Role:* Fetches UTM Campaign ID similarly by filtering `utm.campaign` with `body.campaign`.  
    - *Configuration:* Same as above, targeting `utm.campaign`.

  - **Source ID Validation**  
    - *Type:* Code  
    - *Role:* Extracts the first matched UTM Source ID or null if none found.  
    - *Code snippet:* Returns `{ source_id: firstItemIdOrNull }`.  
    - *Inputs:* Output from `UTM: Get Source ID` node.  
    - *Outputs:* JSON with `source_id`.

  - **Medium ID Validation**  
    - *Type:* Code  
    - *Role:* Extracts the first matched UTM Medium ID or null.  
    - *Similar structure to Source ID Validation.*

  - **Campaign ID Validation**  
    - *Type:* Code  
    - *Role:* Extracts the first matched UTM Campaign ID or null.  
    - *Similar structure to Source ID Validation.*

  - **Sticky Note3**  
    - Labels this block as "Get Marketing Data".

---

#### 1.3 Data Consolidation

- **Overview:**  
  This block merges the webhook lead data and the resolved UTM IDs into a single, clean JSON object formatted for Odoo lead creation.

- **Nodes Involved:**  
  - `Merge`  
  - `Prepare Request` (Code)  
  - `Sticky Note` (Odoo - Lead Create)

- **Node Details:**

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Joins four input streams: webhook lead data, source ID JSON, medium ID JSON, campaign ID JSON by appending them into one combined set for processing.  
    - *Configuration:* Number of inputs: 4, operation: append.  
    - *Inputs:*  
      - Input 1: Source ID Validation output  
      - Input 2: Medium ID Validation output  
      - Input 3: Campaign ID Validation output  
      - Input 4: Webhook lead data (passed from `Required data missing?`)  
    - *Outputs:* Combined array of four JSON objects.

  - **Prepare Request**  
    - *Type:* Code  
    - *Role:* Processes the merged input array to build a single object representing the Odoo lead fields.  
    - *Key logic:*  
      - Extracts lead’s first name, last name, email, phone, notes (as description).  
      - Combines first and last name into contact_name and name fields.  
      - Includes UTM IDs (campaign_id, source_id, medium_id), safely converted to numbers or null.  
      - Sets lead type to `"lead"`.  
      - Throws error if webhook lead payload is missing after merge.  
    - *Input:* Four JSON objects from merged inputs.  
    - *Output:* One JSON object with clean lead data ready for Odoo.  
    - *Edge cases:* Missing webhook payload causes explicit error; UTM IDs default to null if missing or invalid.

  - **Sticky Note**  
    - Notes that this block prepares the request body to create an Odoo Lead custom object.

---

#### 1.4 Lead Creation in Odoo

- **Overview:**  
  Sends the consolidated lead data to Odoo CRM to create a new lead record.

- **Nodes Involved:**  
  - `Create Lead` (Odoo node)  
  - `Execution Data`  
  - `Success` (Respond to Webhook)  

- **Node Details:**

  - **Create Lead**  
    - *Type:* Odoo node (custom resource)  
    - *Role:* Creates a new `crm.lead` record in Odoo with the prepared data.  
    - *Configuration:*  
      - Resource: custom  
      - Custom Resource: `crm.lead`  
      - Fields to create:  
        - `name`: lead full name or email fallback  
        - `contact_name`  
        - `email_from`  
        - `phone`  
        - `description`  
        - `type`: fixed "lead"  
        - UTM many2one fields: `source_id`, `medium_id`, `campaign_id`  
    - *Credentials:* Odoo API credentials  
    - *Inputs:* JSON lead data from `Prepare Request`  
    - *Outputs:* Created lead record details including ID  
    - *Edge cases:* Odoo API errors; network failures.

  - **Execution Data**  
    - *Type:* Execution Data  
    - *Role:* Captures execution metadata from previous node (Create Lead) for response.  
    - *Inputs:* Output of `Create Lead` node.  
    - *Outputs:* Passes execution info to respond node.

  - **Success**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends HTTP 200 response to webhook client indicating success with no response data.  
    - *Configuration:* Status code 200, no body.

---

#### 1.5 Response Handling

- **Overview:**  
  This block sends HTTP responses back to the webhook caller depending on success or validation errors.

- **Nodes Involved:**  
  - `Bad Request` (400 response)  
  - `Success` (200 response)

- **Node Details:**  
  Covered above in relevant blocks.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                                | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                                  |
|-------------------------|------------------------|-----------------------------------------------|--------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook - Lead Webform   | Webhook                | Receives incoming lead data securely          | External HTTP request                | Required data missing?                 | Example payload JSON demonstrating expected input fields                                                    |
| Required data missing?   | If                     | Validates mandatory lead fields                | Webhook - Lead Webform               | Bad Request (if missing), UTM: Get Campaign ID (if valid) |                                                                                                              |
| Bad Request             | Respond to Webhook      | Sends HTTP 400 if validation fails             | Required data missing? (true branch) | -                                     |                                                                                                              |
| UTM: Get Campaign ID     | Odoo                   | Retrieves UTM Campaign ID from Odoo            | Required data missing? (false branch) | Campaign ID Validation                |                                                                                                              |
| UTM: Get Medium ID       | Odoo                   | Retrieves UTM Medium ID from Odoo               | Required data missing? (false branch) | Medium ID Validation                  |                                                                                                              |
| UTM: Get Source ID       | Odoo                   | Retrieves UTM Source ID from Odoo               | Required data missing? (false branch) | Source ID Validation                  |                                                                                                              |
| Campaign ID Validation   | Code                   | Extracts Campaign ID or null                    | UTM: Get Campaign ID                 | Merge                                |                                                                                                              |
| Medium ID Validation     | Code                   | Extracts Medium ID or null                      | UTM: Get Medium ID                   | Merge                                |                                                                                                              |
| Source ID Validation     | Code                   | Extracts Source ID or null                      | UTM: Get Source ID                   | Merge                                |                                                                                                              |
| Merge                   | Merge                  | Combines lead data and UTM IDs into one object | Source ID Validation, Medium ID Validation, Campaign ID Validation, Required data missing? | Prepare Request                   |                                                                                                              |
| Prepare Request         | Code                   | Creates final lead JSON for Odoo creation      | Merge                              | Create Lead                         | Preparation of request to create Odoo Lead custom object                                                    |
| Create Lead             | Odoo                   | Creates lead record in Odoo CRM                 | Prepare Request                     | Execution Data                      |                                                                                                              |
| Execution Data          | Execution Data          | Captures execution metadata for response       | Create Lead                        | Success                            |                                                                                                              |
| Success                 | Respond to Webhook      | Sends HTTP 200 success response                 | Execution Data                     | -                                    |                                                                                                              |
| Sticky Note             | Sticky Note            | Explains lead request preparation               | -                                  | -                                    | Preparation of the request creating the right body for Lead custom object                                   |
| Sticky Note1            | Sticky Note            | Shows example webhook payload                    | -                                  | -                                    | Example: JSON with firstname, lastname, phone, email, source, medium, campaign, notes                        |
| Sticky Note2            | Sticky Note            | Usage instructions and prerequisites             | -                                  | -                                    | Detailed How-To with prerequisites, testing example, and notes on usage                                     |
| Sticky Note3            | Sticky Note            | Describes marketing data retrieval block         | -                                  | -                                    | "Get Marketing Data"                                                                                         |
| Sticky Note4            | Sticky Note            | Explains workflow logic, steps, and notes        | -                                  | -                                    | Detailed explanation of each workflow step and notes on Odoo version compatibility, credentials, and testing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Webhook - Lead Webform`  
   - Type: Webhook  
   - Path: `lead-webform`  
   - HTTP Method: POST  
   - Authentication: Header Auth with a secret token header (e.g., `x-webhook-token`)  
   - Response Mode: Response Node

2. **Create If Node for Validation**  
   - Name: `Required data missing?`  
   - Type: If (condition)  
   - Conditions: Check if any of `body.firstname`, `body.lastname`, or `body.email` do not exist or are empty (string not exists).  
   - Connect input from `Webhook - Lead Webform`.

3. **Create Bad Request Respond Node**  
   - Name: `Bad Request`  
   - Type: Respond to Webhook  
   - Response Code: 400  
   - Respond with no data  
   - Connect from `Required data missing?` node's True output.

4. **Create Odoo Nodes to Get UTM IDs**  
   - Create three Odoo nodes named:  
     - `UTM: Get Source ID`  
     - `UTM: Get Medium ID`  
     - `UTM: Get Campaign ID`  
   - For each:  
     - Resource: Custom  
     - Custom Resource: `utm.source`, `utm.medium`, `utm.campaign` respectively  
     - Operation: Get All, limit 1  
     - Filter: `name` equals corresponding webhook JSON field (`body.source`, `body.medium`, `body.campaign`)  
     - Fields List: `name`, `id`  
     - On Error: Continue Regular Output (to avoid workflow failure if no record found)  
   - Connect all from the False output of `Required data missing?`.

5. **Create Code Nodes to Extract IDs**  
   - Create three code nodes named:  
     - `Source ID Validation`  
     - `Medium ID Validation`  
     - `Campaign ID Validation`  
   - Each node's JS code:  
     ```js
     const f = items[0]?.json;
     return [{ json: { key_id: f?.id ?? null } }];
     ```  
     Replace `key_id` with `source_id`, `medium_id`, or `campaign_id` accordingly.  
   - Connect each to the corresponding UTM Get node.

6. **Create Merge Node**  
   - Name: `Merge`  
   - Type: Merge  
   - Number of Inputs: 4  
   - Mode: Append  
   - Connect inputs in order:  
     1. Output of `Source ID Validation`  
     2. Output of `Medium ID Validation`  
     3. Output of `Campaign ID Validation`  
     4. False output of `Required data missing?` (original webhook payload)

7. **Create Code Node to Prepare Lead Request**  
   - Name: `Prepare Request`  
   - Type: Code  
   - JavaScript code to:  
     - Extract webhook payload and UTM IDs from merged inputs  
     - Compose final JSON lead object with fields: `name`, `contact_name`, `email_from`, `phone`, `description`, `type` (fixed "lead"), `campaign_id`, `source_id`, `medium_id`  
     - Throw error if webhook payload missing  
   - Connect from `Merge` node.

8. **Create Odoo Node to Create Lead**  
   - Name: `Create Lead`  
   - Type: Odoo  
   - Resource: Custom  
   - Custom Resource: `crm.lead`  
   - Operation: Create  
   - Fields to create: `name`, `contact_name`, `email_from`, `phone`, `description`, `type`, `campaign_id`, `source_id`, `medium_id` with expressions from input JSON.  
   - Connect from `Prepare Request`.

9. **Create Execution Data Node**  
   - Name: `Execution Data`  
   - Type: Execution Data  
   - Connect from `Create Lead`.

10. **Create Respond to Webhook Node for Success**  
    - Name: `Success`  
    - Type: Respond to Webhook  
    - Response Code: 200  
    - Respond with no data  
    - Connect from `Execution Data`.

11. **Credential Setup:**  
    - Create n8n credentials for:  
      - Odoo API (URL, DB name, user login, API key)  
      - HTTP Header Auth (secret token for webhook security)

12. **Add Sticky Notes as Documentation:**  
    - Add notes explaining input JSON format, usage instructions, and workflow logic as described in sticky notes in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| **How To Use**: Prerequisites include enabling Leads in Odoo CRM, obtaining API keys, setting up n8n credentials, and securing webhook with header authentication. Example curl command provided for testing. Payload requires firstname, lastname, and email; phone, notes, source, medium, campaign are optional. | Sticky Note2 content inside workflow; essential user setup and testing instructions.                  |
| Recent Odoo versions do not support the `mobile` field for leads; use `phone` instead.                                                                                                                                                              | Sticky Note4 content; important for field mapping compatibility.                                       |
| If UTM records are missing in Odoo, the workflow currently passes null IDs; to auto-create missing UTM entries, add conditional logic after each `getAll` node and create missing records accordingly.                                                | Sticky Note4 content; advice for customization and extension.                                         |
| Ensure that the public URL, webhook path, and authentication headers are consistent in your deployment environment (ngrok, Cloudflare, reverse proxy, etc.).                                                                                        | Sticky Note2 and Sticky Note4; deployment and security notes.                                         |
| Example webhook payload JSON: `{ "firstname": "John", "lastname": "Doe", "phone": "+393331212123", "email": "test@outlook.com", "source": "Ads", "medium": "Website", "notes": "test notes", "campaign": "WebMassive" }`                                    | Sticky Note1 content; sample payload to simulate webhook POST requests.                               |

---

*Disclaimer: The text provided results exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*