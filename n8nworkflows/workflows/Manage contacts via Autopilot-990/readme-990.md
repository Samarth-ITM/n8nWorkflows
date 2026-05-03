Manage contacts via Autopilot

https://n8nworkflows.xyz/workflows/manage-contacts-via-autopilot-990


# Manage contacts via Autopilot

### 1. Workflow Overview

This workflow automates managing contacts within Autopilot, a marketing automation platform. It sequentially creates a new contact list, adds a contact to that list, updates contact details, and finally retrieves all contacts from the created list. The workflow is designed for use cases involving automated contact list management and synchronization within Autopilot.

Logical blocks:

- **1.1 List Creation:** Create a new contact list in Autopilot.
- **1.2 Contact Creation:** Add a new contact to the newly created list.
- **1.3 Contact Update:** Update the newly added contact’s information.
- **1.4 Retrieve Contacts:** Fetch all contacts from the created list.

---

### 2. Block-by-Block Analysis

#### Block 1.1: List Creation

- **Overview:**  
  Creates a new contact list named `n8n-docs` in Autopilot, which serves as the container for subsequent contacts.

- **Nodes Involved:**  
  - Autopilot

- **Node Details:**

  - **Autopilot**  
    - *Type & Role:* Autopilot node configured for list resource creation.  
    - *Configuration:*  
      - Operation: Create list  
      - List name: `n8n-docs`  
      - Resource: List management  
    - *Expressions / Variables:* None; static list name  
    - *Input / Output:*  
      - Input: None (starting node)  
      - Output: Returns list metadata including `list_id` used downstream  
    - *Version:* Compatible with Autopilot API v1 node version  
    - *Potential Failures:*  
      - Authentication errors if credentials are invalid  
      - API rate limiting or network timeouts  
      - Duplicate list name errors if `n8n-docs` already exists  
    - *Sub-workflow:* None

---

#### Block 1.2: Contact Creation

- **Overview:**  
  Adds a new contact to the list created in the previous step. The contact’s email is expected to be provided dynamically.

- **Nodes Involved:**  
  - Autopilot1

- **Node Details:**

  - **Autopilot1**  
    - *Type & Role:* Autopilot node for contact creation.  
    - *Configuration:*  
      - Resource: Contact  
      - Email: Empty string by default; expected to be set dynamically or via input data  
      - Additional Fields:  
        - `autopilotList`: Set dynamically using the `list_id` from the previous node (`={{$json["list_id"]}}`) ensuring the contact is added to the newly created list  
    - *Expressions / Variables:*  
      - `={{$json["list_id"]}}` dynamically links the contact to the created list  
    - *Input / Output:*  
      - Input: Output from the Autopilot node (list creation)  
      - Output: Returns created contact details, including email  
    - *Version:* Requires Autopilot API v1 node version  
    - *Potential Failures:*  
      - Missing or invalid email (empty string default may cause API rejections)  
      - Auth errors or API timeouts  
      - Failure if `list_id` is invalid or list creation failed  
    - *Sub-workflow:* None

---

#### Block 1.3: Contact Update

- **Overview:**  
  Updates the contact’s information (e.g., company name) created in the previous step, identified by their email.

- **Nodes Involved:**  
  - Autopilot2

- **Node Details:**

  - **Autopilot2**  
    - *Type & Role:* Autopilot node for updating contact details.  
    - *Configuration:*  
      - Resource: Contact  
      - Email: Dynamically set from the previous node’s parameter: `={{$node["Autopilot1"].parameter["email"]}}`  
      - Additional Fields:  
        - Company: Set to static value `"n8n"`  
    - *Expressions / Variables:*  
      - Email dynamically references the contact just created to ensure the correct contact is updated  
    - *Input / Output:*  
      - Input: Output of Autopilot1 (contact creation)  
      - Output: Returns updated contact details  
    - *Version:* Autopilot API v1  
    - *Potential Failures:*  
      - Email expression could fail if Autopilot1 returns empty or invalid email  
      - API errors if the contact does not exist or network issues arise  
    - *Sub-workflow:* None

---

#### Block 1.4: Retrieve Contacts

- **Overview:**  
  Fetches all contacts associated with the `n8n-docs` list created initially.

- **Nodes Involved:**  
  - Autopilot3

- **Node Details:**

  - **Autopilot3**  
    - *Type & Role:* Autopilot node to retrieve contact list details.  
    - *Configuration:*  
      - Resource: `contactList`  
      - Operation: `getAll` contacts  
      - List ID: Dynamically retrieved from the first node’s output (`={{$node["Autopilot"].json["list_id"]}}`)  
      - Return All: `true` (fetch all contacts without pagination)  
    - *Expressions / Variables:*  
      - Uses `list_id` from the originally created list to scope the retrieval  
    - *Input / Output:*  
      - Input: Output from Autopilot2 (contact update)  
      - Output: Returns array of all contacts in the list  
    - *Version:* Autopilot API v1  
    - *Potential Failures:*  
      - Invalid or missing `list_id`  
      - API errors or network timeouts  
      - Large data sets could cause performance delays if many contacts exist  
    - *Sub-workflow:* None

---

### 3. Summary Table

| Node Name    | Node Type                      | Functional Role         | Input Node(s) | Output Node(s) | Sticky Note                                           |
|--------------|--------------------------------|------------------------|---------------|----------------|-------------------------------------------------------|
| Autopilot    | Autopilot (List resource)       | Create contact list    | None          | Autopilot1     | Creates a new list called `n8n-docs` in Autopilot.    |
| Autopilot1   | Autopilot (Contact resource)    | Create contact in list | Autopilot     | Autopilot2     | Creates a new contact and adds to the list created.   |
| Autopilot2   | Autopilot (Contact resource)    | Update contact info    | Autopilot1    | Autopilot3     | Updates the contact created in the previous node.     |
| Autopilot3   | Autopilot (ContactList resource)| Retrieve all contacts  | Autopilot2    | None           | Returns all contacts of the `n8n-docs` list created.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Autopilot List Node**  
   - Add a node: Type `Autopilot`  
   - Set resource to `list`  
   - Set operation to create a list  
   - Set list name to `n8n-docs` (static)  
   - Configure Autopilot credentials with valid API key  
   - This node is the workflow start point

2. **Create Contact Node**  
   - Add a node: Type `Autopilot` (rename to Autopilot1)  
   - Set resource to `contact`  
   - Set operation to create a contact  
   - Set email field to empty string or connect to input data dynamically as needed  
   - In additional fields, set `autopilotList` to expression: `{{$json["list_id"]}}` referencing the output of the List Creation node  
   - Connect Autopilot node output to this node input  
   - Use the same Autopilot credentials

3. **Update Contact Node**  
   - Add a node: Type `Autopilot` (rename to Autopilot2)  
   - Set resource to `contact`  
   - Set operation to update contact  
   - Set email field using expression: `{{$node["Autopilot1"].parameter["email"]}}` to reference the created contact’s email  
   - In additional fields, add a static field `Company` with value `"n8n"`  
   - Connect Autopilot1 output to this node input  
   - Use the same Autopilot credentials

4. **Get All Contacts Node**  
   - Add a node: Type `Autopilot` (rename to Autopilot3)  
   - Set resource to `contactList`  
   - Set operation to `getAll` contacts  
   - Set List ID field using expression: `{{$node["Autopilot"].json["list_id"]}}` referencing the original list creation node  
   - Set `Return All` to true  
   - Connect Autopilot2 output to this node input  
   - Use the same Autopilot credentials

5. **Connect the Nodes Sequentially:**  
   - Autopilot → Autopilot1 → Autopilot2 → Autopilot3

6. **Test the Workflow:**  
   - Ensure Autopilot API credentials are correctly configured with required permissions  
   - Provide valid email input to Autopilot1 or modify the workflow to accept input dynamically  
   - Execute to verify list creation, contact creation, update, and retrieval of contacts

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                          |
|------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| Autopilot API credentials must be configured with appropriate permissions to manage lists and contacts.    | n8n Autopilot node documentation        |
| Email field in contact creation node is empty by default; ensure to provide a valid email to avoid errors. | Workflow input requirement               |
| The list name `n8n-docs` is static; modify if needed to prevent duplication errors in Autopilot.           | List naming constraint                   |
| The workflow demonstrates sequential dependency chaining using expressions referencing prior node outputs. | n8n expression syntax                    |

---

This reference document fully describes the workflow "Manage contacts via Autopilot" for understanding, reproduction, and modification purposes.