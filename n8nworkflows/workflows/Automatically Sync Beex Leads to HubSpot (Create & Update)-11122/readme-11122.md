Automatically Sync Beex Leads to HubSpot (Create & Update)

https://n8nworkflows.xyz/workflows/automatically-sync-beex-leads-to-hubspot--create---update--11122


# Automatically Sync Beex Leads to HubSpot (Create & Update)

### 1. Workflow Overview

This workflow is designed to **automatically synchronize leads from Beex to HubSpot contacts**, handling both lead creation and update events. It is targeted at users who want to keep their HubSpot CRM contacts in sync with lead data generated or updated in the Beex platform.

The workflow is logically divided into these main functional blocks:

- **1.1 Input Reception (Beex Trigger & Format):** Receives lead events from Beex via a webhook and flattens the nested JSON data to a simpler structure for easier processing.

- **1.2 Validation & Field Preparation:** Filters out leads without valid email addresses and maps input fields to the appropriate HubSpot contact properties.

- **1.3 Event Routing:** Routes the flow depending on whether the event is a lead creation or update.

- **1.4 HubSpot API Execution:** Sends HTTP requests to HubSpot APIs to either create a new contact or update an existing contact based on the event type.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block listens for lead creation or update events from Beex and prepares the incoming data by flattening nested objects into a simplified JSON format.

**Nodes Involved:**  
- Beex Trigger  
- Format (Code node)

**Node Details:**

- **Beex Trigger**  
  - Type: `n8n-nodes-beex.beexTrigger` (community node)  
  - Role: Listens to Beex webhook events of types `contact_create` and `contact_update`.  
  - Configuration:  
    - Event Types set to `contact_create` and `contact_update`  
    - Uses OAuth2 Bearer token credential for authentication with Beex API (credential named "TokenApp Beex account").  
  - Input: Webhook requests from Beex platform.  
  - Output: Raw lead event data with nested structure.  
  - Failure cases: Invalid or expired token, network timeouts, webhook misconfiguration on Beex side.  
  - Notes: Requires community node `n8n-nodes-beex`.  

- **Format (Code)**  
  - Type: `n8n-nodes-base.code` (JavaScript node)  
  - Role: Flattens nested fields from the Beex lead payload into a simple JSON object for easier downstream processing.  
  - Configuration:  
    - Extracts `general_dates` and `custom_fields` from input data.  
    - Flattens each nested object into key-value pairs with values only.  
    - Outputs a JSON object including email, portfolio info, and all flattened fields.  
  - Key expression:  
    ```js
    const data = $json.data;
    const generalDates = data.general_dates;
    const customFields = data.custom_fields;

    const flatDates = {};
    const flatFields = {};

    for (const [key, obj] of Object.entries(generalDates)) {
      flatDates[key] = obj.value;
    }

    for (const [key, obj] of Object.entries(customFields)) {
      flatFields[obj.title] = obj.value;
    }

    return {
      email: data.email,
      portfolio_id: data.portfolio.id,
      portfolio_name: data.portfolio.name,
      ...flatDates,
      ...flatFields
    };
    ```  
  - Inputs: Beex Trigger output.  
  - Outputs: Flattened JSON object with all relevant fields.  
  - Failure cases: If input data structure changes or missing keys, the node could error or produce incomplete data.

---

#### 2.2 Validation & Field Preparation

**Overview:**  
Filters leads missing an email address and maps fields from the flattened payload to HubSpot contact property names.

**Nodes Involved:**  
- ¿Email Null? (Filter)  
- Set Fields

**Node Details:**

- **¿Email Null? (Filter)**  
  - Type: `n8n-nodes-base.filter`  
  - Role: Filters out leads that do not have a valid, non-empty email address.  
  - Configuration: Conditions require that `email` exists and is not empty (`string exists` and `notEmpty` on `$json.email`).  
  - Input: Flattened JSON from Format node.  
  - Outputs: Passes valid leads to next node, blocks leads without a proper email.  
  - Failure cases: Missing or malformed email field; leads filtered out (expected behavior).  

- **Set Fields**  
  - Type: `n8n-nodes-base.set`  
  - Role: Maps fields from Beex data to HubSpot contact property names expected by HubSpot API.  
  - Configuration:  
    - Sets `firstname` from `first_name`  
    - Sets `lastname` by concatenating `paternal_surname` and `maternal_surname`  
    - Copies `email` to `email` property  
    - Copies custom field `hubspot_custom_field` if present  
  - Input: Filtered lead data (with valid email).  
  - Outputs: JSON object formatted with the correct HubSpot property names.  
  - Failure cases: If required fields are missing or empty, HubSpot API may reject the request.

---

#### 2.3 Event Routing

**Overview:**  
Routes the workflow based on the event type received from Beex, splitting the flow into create or update branches.

**Nodes Involved:**  
- Routing (Switch)

**Node Details:**

- **Routing (Switch)**  
  - Type: `n8n-nodes-base.switch`  
  - Role: Checks the Beex event type (`contact_create` or `contact_update`) and routes accordingly.  
  - Configuration:  
    - Two outputs keyed as `CREATE` and `UPDATE`  
    - Conditions compare `$json.event` from the original Beex Trigger node to these string values.  
  - Input: Output from Set Fields node (which has lead data and event type inherited).  
  - Outputs:  
    - `CREATE` output goes to HTTP POST request node  
    - `UPDATE` output goes to HTTP PATCH request node  
  - Failure cases: Unexpected event types will not match any route, resulting in no action.

---

#### 2.4 HubSpot API Execution

**Overview:**  
Sends HTTP requests to HubSpot API to create or update contacts based on lead event type.

**Nodes Involved:**  
- Create Contact (HTTP Request)  
- Update Contact (HTTP Request)

**Node Details:**

- **Create Contact**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: Sends a POST request to HubSpot to create a new contact.  
  - Configuration:  
    - URL: `https://api.hubapi.com/crm/v3/objects/contacts`  
    - HTTP Method: POST  
    - Body: JSON with `properties` containing all mapped fields from Set Fields node (`{"properties": <lead-data-json>}`)  
    - Authentication: HubSpot App Token credential configured with required scopes (read/write contacts).  
    - Query Parameters: none required for creation.  
  - Input: From Routing node on `CREATE` output.  
  - Output: HubSpot API response with contact creation confirmation or error.  
  - Failure cases:  
    - Invalid or expired HubSpot token  
    - API rate limiting or downtime  
    - Validation errors if properties are incorrect or missing  

- **Update Contact**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: Sends a PATCH request to HubSpot to update an existing contact by email.  
  - Configuration:  
    - URL template: `https://api.hubapi.com/crm/v3/objects/contacts/{{ $json.email }}`  
    - HTTP Method: PATCH  
    - Body: JSON with `properties` containing mapped fields (`{"properties": <lead-data-json>}`)  
    - Query Parameter: `idProperty=email` to identify contact by email  
    - Authentication: Same HubSpot App Token as Create Contact  
  - Input: From Routing node on `UPDATE` output.  
  - Output: HubSpot API response with update confirmation or error.  
  - Failure cases:  
    - Contact not found with given email causing 404 error  
    - Token or permission errors  
    - Malformed data causing validation failure  

---

### 3. Summary Table

| Node Name         | Node Type                 | Functional Role                     | Input Node(s)      | Output Node(s)     | Sticky Note                                                                                                             |
|-------------------|---------------------------|-----------------------------------|--------------------|--------------------|-------------------------------------------------------------------------------------------------------------------------|
| Sticky Note1      | Sticky Note               | Overview & instructions            |                    |                    | ## Sync Beex Leads to HubSpot<br>Disclaimer about community node `n8n-nodes-beex` and setup instructions                 |
| Beex Trigger      | Beex Trigger (community)  | Input reception from Beex webhook |                    | Format             | Create/Update                                                                                                           |
| Format            | Code                      | Flatten nested JSON data           | Beex Trigger       | ¿Email Null?       | Trigger Node (Beex Trigger + Format)<br>- Link webhook URL on Beex platform<br>- Convert input data to flat JSON       |
| ¿Email Null?      | Filter                    | Filter leads without email         | Format             | Set Fields         | Filter and Set Fields<br>- Leads without email filtered out<br>- Mapped fields must match HubSpot properties            |
| Set Fields        | Set                       | Map fields to HubSpot property names | ¿Email Null?        | Routing            | Filter and Set Fields                                                                                                  |
| Routing           | Switch                    | Route based on event type          | Set Fields          | Create Contact, Update Contact | Create/Update                                                                                                           |
| Create Contact    | HTTP Request              | Create contact in HubSpot          | Routing (CREATE)    |                    | HTTP Requests (Routing + Create/Update)                                                                                 |
| Update Contact    | HTTP Request              | Update contact in HubSpot          | Routing (UPDATE)    |                    | HTTP Requests (Routing + Create/Update)                                                                                 |
| Sticky Note3      | Sticky Note               | HTTP request section overview      |                    |                    | ## HTTP Requests (Routing + Create/Update)                                                                             |
| Sticky Note4      | Sticky Note               | Trigger node explanation           |                    |                    | ## Trigger Node (Beex Trigger + Format)<br>- Link webhook URL on Beex platform<br>- Convert input data into flat JSON  |
| Sticky Note5      | Sticky Note               | Filter and field mapping notes     |                    |                    | ## Filter and Set Fields<br>- Leads without email filtered out<br>- Fields must align with HubSpot contact properties   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Beex Trigger Node**  
   - Type: `Beex Trigger` (community node)  
   - Configure: Set event types to `contact_create` and `contact_update`  
   - Assign Beex API credentials (Bearer token OAuth2)  
   - Save webhook URL generated by node and configure it in Beex platform to send lead events  

2. **Add Code Node "Format"**  
   - Type: `Code` (JavaScript)  
   - Paste the provided JS code to flatten nested fields:  
     ```js
     const data = $json.data;
     const generalDates = data.general_dates;
     const customFields = data.custom_fields;

     const flatDates = {};
     const flatFields = {};

     for (const [key, obj] of Object.entries(generalDates)) {
       flatDates[key] = obj.value;
     }

     for (const [key, obj] of Object.entries(customFields)) {
       flatFields[obj.title] = obj.value;
     }

     return {
       email: data.email,
       portfolio_id: data.portfolio.id,
       portfolio_name: data.portfolio.name,
       ...flatDates,
       ...flatFields
     };
     ```  
   - Connect Beex Trigger output to Format input  

3. **Add Filter Node "¿Email Null?"**  
   - Type: `Filter`  
   - Set conditions:  
     - `$json.email` exists (string exists)  
     - `$json.email` not empty (string notEmpty)  
   - Connect Format node output to this filter’s input  

4. **Add Set Node "Set Fields"**  
   - Type: `Set`  
   - Map fields:  
     - firstname = `{{$json.first_name}}`  
     - lastname = `{{$json.paternal_surname}} {{$json.maternal_surname}}`  
     - email = `{{$json.email}}`  
     - hubspot_custom_field = `{{$json.hubspot_custom_field}}` (optional custom field)  
   - Connect filter’s true output to Set Fields node input  

5. **Add Switch Node "Routing"**  
   - Type: `Switch`  
   - Add two outputs:  
     - `CREATE` if `$json.event` equals `"contact_create"`  
     - `UPDATE` if `$json.event` equals `"contact_update"`  
   - Connect Set Fields output to Routing input  

6. **Add HTTP Request Node "Create Contact"**  
   - Type: `HTTP Request`  
   - Configure:  
     - Method: POST  
     - URL: `https://api.hubapi.com/crm/v3/objects/contacts`  
     - Body Content Type: JSON  
     - Body: `{"properties": {{ $json.toJsonString() }}}` (use expression to convert JSON)  
     - Authentication: Select HubSpot App Token credential with read/write contact permissions  
   - Connect Routing node’s `CREATE` output to this node  

7. **Add HTTP Request Node "Update Contact"**  
   - Type: `HTTP Request`  
   - Configure:  
     - Method: PATCH  
     - URL: `https://api.hubapi.com/crm/v3/objects/contacts/{{ $json.email }}` (use expression for email in URL)  
     - Query Parameter: `idProperty=email`  
     - Body Content Type: JSON  
     - Body: `{"properties": {{ $json.toJsonString() }}}`  
     - Authentication: Same HubSpot App Token credential  
   - Connect Routing node’s `UPDATE` output to this node  

8. **Test the complete flow**  
   - Trigger events from Beex for contact creation and update  
   - Monitor workflow executions in n8n to confirm contact creation or updates in HubSpot  

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                            |
|-------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| This workflow requires the community node `n8n-nodes-beex` to listen to Beex lead events.                                                  | Node repository: https://github.com/n8n-community/n8n-nodes-beex          |
| HubSpot App Token must have read and write permissions for contacts in CRM.                                                                | HubSpot dev docs: https://developers.hubspot.com/docs/api/crm/contacts    |
| The email field is critical for contact identification and update operations; ensure it is always present and valid in Beex leads.       | Workflow logic and filter node                                            |
| HubSpot API rate limits and token expiration can cause failures; monitor API limits and refresh tokens as needed.                        | HubSpot API documentation                                                  |
| Flattening nested JSON fields simplifies mapping but requires maintaining data structure consistency from Beex.                         | Code node explanation                                                      |
| For custom HubSpot fields, ensure the field names in Set node match exactly the HubSpot custom property API names.                        | HubSpot property management in CRM                                         |

---

*Disclaimer: The content provided is exclusively derived from an automated workflow created with n8n, adhering strictly to current content policies without any illegal, offensive, or protected material. All data processed is legal and public.*