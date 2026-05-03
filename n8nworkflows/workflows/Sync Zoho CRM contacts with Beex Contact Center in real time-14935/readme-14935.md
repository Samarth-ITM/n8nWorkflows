Sync Zoho CRM contacts with Beex Contact Center in real time

https://n8nworkflows.xyz/workflows/sync-zoho-crm-contacts-with-beex-contact-center-in-real-time-14935


# Sync Zoho CRM contacts with Beex Contact Center in real time

# Workflow Documentation: Sync Zoho CRM contacts with Beex Contact Center

### 1. Workflow Overview
This workflow provides real-time synchronization between Zoho CRM and the Beex Contact Center. It listens for contact-related events (creation and updates) triggered by Zoho via webhooks and ensures that the corresponding contact data is mirrored in Beex.

The logic is divided into three functional blocks:
- **1.1 Input Reception & Parsing:** Captures the incoming webhook from Zoho and transforms the raw payload into structured fields.
- **1.2 Event Routing:** Determines whether the contact is being created for the first time or updated based on the event type.
- **1.3 Beex Synchronization:** Executes the final API call to Beex to either create a new lead (after validating the phone number) or update an existing client record.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parsing
**Overview:** This block acts as the entry point, receiving data from Zoho CRM and preparing it for processing by cleaning and splitting specific strings (like phone numbers).

- **Nodes Involved:** `On Listen Event`, `Set Fields`.
- **Node Details:**
    - **On Listen Event** (Webhook): 
        - **Role:** Listens for HTTP POST requests from Zoho.
        - **Configuration:** Set to `POST` method with a specific unique path.
        - **Failure Risks:** Incorrect URL configuration in Zoho or network timeouts.
    - **Set Fields** (Set):
        - **Role:** Maps raw JSON from the webhook to specific internal variables.
        - **Key Expressions:** 
            - `event`: Extracted from `{{ $json.headers.event }}`.
            - `countryCode`: Extracted using `{{ $json.body.phone.split(' ')[0].slice(1) }}` (removes the leading '+' and takes the first part of the split).
            - `phone`: Extracted using `{{ $json.body.phone.split(' ')[1] }}`.
            - Static Values: `portfolio_id` is set to `"81"`, `sequence_id` is set to `"26"`.
        - **Failure Risks:** If the phone number format in Zoho does not contain a space, the `split` expression will fail or produce unexpected results.

#### 2.2 Event Routing
**Overview:** This block analyzes the event type to decide which synchronization path (Create vs. Update) to follow.

- **Nodes Involved:** `Routing`.
- **Node Details:**
    - **Routing** (Switch):
        - **Role:** Directs the flow based on the `event` variable.
        - **Logic:**
            - If `event` equals `insert` $\rightarrow$ Route to **CREATED** branch.
            - If `event` equals `update` $\rightarrow$ Route to **UPDATED** branch.
        - **Failure Risks:** If Zoho sends an event type other than `insert` or `update`, the workflow will stop at this node.

#### 2.3 Beex Synchronization
**Overview:** The final execution phase where data is written to the Beex API. It handles two distinct scenarios: new lead creation and existing client updates.

- **Nodes Involved:** `Filter Phone`, `Create Contact`, `Get Client`, `Update Client`.
- **Node Details:**
    - **Filter Phone** (Filter):
        - **Role:** Prevents the creation of contacts without phone numbers.
        - **Configuration:** Validates that `phone` exists and is not empty.
    - **Create Contact** (Beex):
        - **Role:** Creates a new lead in Beex.
        - **Configuration:** Maps `first_name`, `paternal_surname`, `phone_number`, and `code_country`.
    - **Get Client** (Beex):
        - **Role:** Retrieves the Beex internal ID of a client using the Zoho Contact ID (`code`).
        - **Operation:** `get-one`.
    - **Update Client** (Beex):
        - **Role:** Updates the retrieved client's details.
        - **Configuration:** Uses the ID found in the `Get Client` step and updates `first_name` and `paternal_surname` using data passed from the `Routing` node.
        - **Failure Risks:** Beex API authentication errors or failure to find a matching client in the `Get Client` step.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On Listen Event | Webhook | Entry point for Zoho events | None | Set Fields | Link the webhook URL on Zoho in Instant Actions |
| Set Fields | Set | Data normalization & mapping | On Listen Event | Routing | The fields to be extracted are defined in Zoho CRM by Workflow Rules |
| Routing | Switch | Logic branch (Create vs Update) | Set Fields | Filter Phone, Get Client | Routing + Create/Update. Possible Branches: Creation or Update. |
| Filter Phone | Filter | Phone number validation | Routing | Create Contact | Routing + Create/Update. Possible Branches: Creation or Update. |
| Create Contact | Beex | Post new lead to Beex | Filter Phone | None | Routing + Create/Update. Possible Branches: Creation or Update. |
| Get Client | Beex | Fetch Beex ID via Zoho ID | Routing | Update Client | Routing + Create/Update. Possible Branches: Creation or Update. |
| Update Client | Beex | Update existing Beex record | Get Client | None | Routing + Create/Update. Possible Branches: Creation or Update. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Install Dependency:** Install the community node `n8n-nodes-beex` via the n8n Community Nodes settings.
2.  **Webhook Setup:**
    - Create a **Webhook** node. Set HTTP Method to `POST`. 
    - Note the production URL and configure it in Zoho CRM "Instant Actions" to trigger on contact creation and updates.
3.  **Data Mapping:**
    - Add a **Set** node. Create the following assignments:
        - `event` $\rightarrow$ `{{ $json.headers.event }}`
        - `code` $\rightarrow$ `{{ $json.body.id }}`
        - `name` $\rightarrow$ `{{ $json.body.name }}`
        - `lastName` $\rightarrow$ `{{ $json.body.lastName }}`
        - `countryCode` $\rightarrow$ `{{ $json.body.phone.split(' ')[0].slice(1) }}`
        - `phone` $\rightarrow$ `{{ $json.body.phone.split(' ')[1] }}`
        - `email` $\rightarrow$ `{{ $json.body.email }}`
        - `portfolio_id` $\rightarrow$ `81` (String)
        - `sequence_id` $\rightarrow$ `26` (String)
4.  **Routing Logic:**
    - Add a **Switch** node. Create two rules:
        - Rule 1: If `{{ $json.event }}` equals `insert`, route to output 0.
        - Rule 2: If `{{ $json.event }}` equals `update`, route to output 1.
5.  **Create Branch (Output 0):**
    - Add a **Filter** node: Ensure `phone` is not empty and exists.
    - Add a **Beex** node: Operation `leads` (post). Map the fields from the Set node (Phone, Country Code, Email, etc.).
6.  **Update Branch (Output 1):**
    - Add a **Beex** node: Operation `get-one` using the `code` (Zoho ID) to find the Beex internal ID.
    - Add a second **Beex** node: Operation `put` (update). Set the ID to `{{ $json.results[0].id }}` and map the updated name and surname.
7.  **Credentials:**
    - Configure the **Beex API** credentials using a valid Bearer Token.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| This workflow requires the community node `n8n-nodes-beex` to function. | Technical Dependency |
| Ensure the Zoho CRM Webhook includes the `event` header for routing to work correctly. | Configuration Requirement |
| The phone number parsing logic assumes a format like "+33 612345678". | Data Format Assumption |