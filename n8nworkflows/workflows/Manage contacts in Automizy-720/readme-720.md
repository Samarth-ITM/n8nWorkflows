Manage contacts in Automizy

https://n8nworkflows.xyz/workflows/manage-contacts-in-automizy-720


# Manage contacts in Automizy

### 1. Workflow Overview

This workflow automates managing contacts within Automizy, a marketing automation platform. It is designed to:

- Create a new contact list in Automizy
- Add a new contact with an active status to the newly created list
- Update the contact’s tags
- Retrieve all contacts from the list

The workflow is structured linearly, with each step dependent on the previous one’s output. The logical blocks are:

- **1.1 Trigger & List Creation:** Manual trigger to start the workflow and create a new contact list.
- **1.2 Contact Addition:** Add a new contact to the created list with a specified email and active status.
- **1.3 Contact Update:** Update the newly added contact by adding tags.
- **1.4 Retrieve Contacts:** Fetch all contacts from the created list to review or process further.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & List Creation

- **Overview:**  
  This block initiates the workflow manually and creates a new Automizy contact list named "n8n-docs."

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Automizy (Create List)

- **Node Details:**  

  - **On clicking 'execute'**  
    - *Type:* Manual Trigger  
    - *Role:* Initiates the workflow manually on demand.  
    - *Configuration:* Default manual trigger with no parameters.  
    - *Input:* None (user initiated)  
    - *Output:* Triggers the next node (Automizy list creation).  
    - *Potential Failures:* None typical; user must manually execute.  
    - *Version:* Compatible with n8n v1.x+.

  - **Automizy**  
    - *Type:* Automizy Node  
    - *Role:* Creates a new contact list in Automizy.  
    - *Configuration:*  
      - Resource: List  
      - Operation: Create (implicit by specifying list name)  
      - Name: "n8n-docs" (static string)  
    - *Input:* Trigger output  
    - *Output:* JSON containing the newly created list’s details including its `id`.  
    - *Expression Usage:* The list `id` from this node is used downstream to add contacts.  
    - *Potential Failures:* API authentication errors, network issues, duplicate list name handling.  
    - *Credentials:* Requires valid Automizy API credentials.

#### 1.2 Contact Addition

- **Overview:**  
  Adds a new contact with the email `example@n8n.io` to the created list, setting the contact’s status as active.

- **Nodes Involved:**  
  - Automizy1 (Add Contact)

- **Node Details:**  

  - **Automizy1**  
    - *Type:* Automizy Node  
    - *Role:* Adds a contact to a specified list.  
    - *Configuration:*  
      - Resource: Contact  
      - Operation: Add  
      - Email: "example@n8n.io" (static string)  
      - List Id: Dynamic expression referencing `{{$node["Automizy"].json["id"]}}` (from previous list creation)  
      - Additional Fields: Status set to "ACTIVE"  
    - *Input:* Output from Automizy (list creation)  
    - *Output:* JSON representing the added contact, including the email used.  
    - *Potential Failures:* Invalid email format, API rate limits, list not found if ID is invalid or deleted.  
    - *Credentials:* Automizy API credentials required.

#### 1.3 Contact Update

- **Overview:**  
  Updates the newly added contact by adding a tag "reviewer."

- **Nodes Involved:**  
  - Automizy2 (Update Contact)

- **Node Details:**  

  - **Automizy2**  
    - *Type:* Automizy Node  
    - *Role:* Updates contact details, specifically adding tags.  
    - *Configuration:*  
      - Resource: Contact  
      - Operation: Update  
      - Email: Dynamic expression `{{$node["Automizy1"].json["email"]}}` (email from contact add node)  
      - Update Fields: Tags set to ["reviewer"] (static array)  
    - *Input:* Output from Automizy1  
    - *Output:* JSON of updated contact details  
    - *Potential Failures:* Contact not found (if add failed), API errors, tag validation errors.  
    - *Credentials:* Automizy API credentials.

#### 1.4 Retrieve Contacts

- **Overview:**  
  Retrieves all contacts from the newly created list to confirm or process the list contents.

- **Nodes Involved:**  
  - Automizy3 (Get All Contacts)

- **Node Details:**  

  - **Automizy3**  
    - *Type:* Automizy Node  
    - *Role:* Fetches all contacts associated with the specified list.  
    - *Configuration:*  
      - Resource: Contact  
      - Operation: Get All  
      - List Id: Dynamic expression `{{$node["Automizy"].json["id"]}}`  
      - Return All: true (fetch all contacts without limit)  
      - Additional Fields: None  
    - *Input:* Output from Automizy2  
    - *Output:* Array of all contacts in the list  
    - *Potential Failures:* API timeouts on large lists, authentication errors, list not found.  
    - *Credentials:* Automizy API credentials.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                      |
|---------------------|--------------------|------------------------------------|------------------------|-------------------------|---------------------------------|
| On clicking 'execute'| Manual Trigger     | Starts workflow manually            | None                   | Automizy                |                                 |
| Automizy            | Automizy Node       | Creates a new contact list          | On clicking 'execute'   | Automizy1               |                                 |
| Automizy1           | Automizy Node       | Adds a new contact to the list      | Automizy               | Automizy2               |                                 |
| Automizy2           | Automizy Node       | Updates contact with tags            | Automizy1              | Automizy3               |                                 |
| Automizy3           | Automizy Node       | Retrieves all contacts from the list| Automizy2              | None                    |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a *Manual Trigger* node named "On clicking 'execute'."  
   - No parameters needed.

2. **Create Automizy Node to Create List**  
   - Add an *Automizy* node named "Automizy."  
   - Set *Resource* to "List."  
   - Set *Operation* to create a list by entering the *Name* field with the static value `"n8n-docs"`.  
   - Connect "On clicking 'execute'" node output to this node input.  
   - Configure Automizy API credentials.

3. **Create Automizy Node to Add Contact**  
   - Add an *Automizy* node named "Automizy1."  
   - Set *Resource* to "Contact."  
   - Set *Operation* to "Add."  
   - Set Email to `"example@n8n.io"` (static).  
   - Set *List Id* with expression referencing the list created: `{{$node["Automizy"].json["id"]}}`.  
   - In Additional Fields, set *Status* to `"ACTIVE"`.  
   - Connect "Automizy" node output to "Automizy1" input.  
   - Use the same Automizy API credentials.

4. **Create Automizy Node to Update Contact**  
   - Add an *Automizy* node named "Automizy2."  
   - Set *Resource* to "Contact."  
   - Set *Operation* to "Update."  
   - Set Email with expression: `{{$node["Automizy1"].json["email"]}}` to target the newly added contact.  
   - In Update Fields, add *Tags* with value `["reviewer"]`.  
   - Connect "Automizy1" node output to "Automizy2" input.

5. **Create Automizy Node to Get All Contacts**  
   - Add an *Automizy* node named "Automizy3."  
   - Set *Resource* to "Contact."  
   - Set *Operation* to "Get All."  
   - Set *List Id* to `{{$node["Automizy"].json["id"]}}`.  
   - Enable *Return All* to true to fetch all contacts.  
   - Connect "Automizy2" output to "Automizy3" input.

6. **Credential Setup**  
   - Configure the Automizy API credentials in each Automizy node using valid API keys.

7. **Activate and Test**  
   - Save and activate the workflow.  
   - Manually execute the workflow and monitor each node output to confirm successful list creation, contact addition, update, and retrieval.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                     |
|-------------------------------------------------------------------------------------------------|----------------------------------|
| Automizy API node documentation for reference on operations and parameters.                      | https://docs.n8n.io/integrations/builtin/credentials/automizy/ |
| Workflow screenshot available as an attachment (fileId:276) for visual reference.                | Workflow description asset         |
| Ensure API rate limits and quota are respected when adding/updating contacts in Automizy.        | Automizy API best practices        |

---

This document enables full understanding, modification, and reconstruction of the Automizy contacts management workflow within n8n.