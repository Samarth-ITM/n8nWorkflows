Manage contacts in Google Contacts

https://n8nworkflows.xyz/workflows/manage-contacts-in-google-contacts-637


# Manage contacts in Google Contacts

### 1. Workflow Overview

This workflow automates the management of a single contact in Google Contacts by performing three sequential operations: creating a new contact, updating the created contact with additional company information, and retrieving the updated contact details. It is designed for users who need to programmatically manage contacts within Google Contacts using OAuth2 authentication. The workflow is structured into three main logical blocks:

- **1.1 Input Trigger**: Manual initiation of the workflow.
- **1.2 Contact Creation**: Creating a new contact with basic name information.
- **1.3 Contact Update and Retrieval**: Updating the contact’s company information and fetching the updated record.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block serves as the manual start point of the workflow, allowing users to trigger the contact management process on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Role:** Initiates workflow execution manually.  
  - **Configuration:** No parameters required; simply a trigger button.  
  - **Key Expressions/Variables:** None.  
  - **Input/Output:** No inputs; outputs trigger event to the next node.  
  - **Version-specific Requirements:** None.  
  - **Potential Failures:** None; manual triggers rarely fail but workflow downstream errors may occur after this node.  
  - **Sub-workflow Reference:** None.

---

#### 1.2 Contact Creation

- **Overview:**  
  Creates a new Google Contact with given name "n8n" and family name "n8n" using the Google Contacts API.

- **Nodes Involved:**  
  - Google Contacts

- **Node Details:**  
  - **Node Name:** Google Contacts  
  - **Type:** Google Contacts Node  
  - **Role:** Creates a new contact in the authenticated Google Contacts account.  
  - **Configuration:**  
    - Operation: Create (implicit by providing givenName and familyName without specifying an update or get operation)  
    - Given Name: "n8n"  
    - Family Name: "n8n"  
    - Additional Fields: None specified.  
  - **Key Expressions/Variables:** None (static name values).  
  - **Input/Output:** Input from manual trigger; output includes `contactId` of the newly created contact.  
  - **Version-specific Requirements:** Requires OAuth2 credentials for Google Contacts API.  
  - **Potential Failures:**  
    - Authentication failure if OAuth2 token is invalid or expired.  
    - API rate limits or quota exceeded errors.  
    - Network timeout or connectivity issues.  
  - **Sub-workflow Reference:** None.

---

#### 1.3 Contact Update and Retrieval

- **Overview:**  
  Takes the created contact’s ID and updates the contact’s company information, then retrieves the updated contact to confirm changes.

- **Nodes Involved:**  
  - Google Contacts1 (Update)  
  - Google Contacts2 (Get)

- **Node Details:**  

  - **Node Name:** Google Contacts1  
    - **Type:** Google Contacts Node  
    - **Role:** Updates the existing contact’s company information.  
    - **Configuration:**  
      - Operation: Update  
      - Contact ID: Dynamically retrieved from previous node output `{{$node["Google Contacts"].json["contactId"]}}`  
      - Update Fields: Company details with:  
        - Name: "n8n"  
        - Title: "n8n"  
        - Domain: "n8n.io"  
        - Marked as current company  
    - **Key Expressions/Variables:** Expression for contactId from previous node.  
    - **Input/Output:** Input from Google Contacts (creation node); output includes updated contact data.  
    - **Version-specific Requirements:** OAuth2 for Google Contacts API.  
    - **Potential Failures:**  
      - Invalid `contactId` if creation failed.  
      - Authentication or permission errors.  
      - API limits or malformed update fields.  
    - **Sub-workflow Reference:** None.

  - **Node Name:** Google Contacts2  
    - **Type:** Google Contacts Node  
    - **Role:** Retrieves the updated contact details, specifically requesting the “organizations” field to verify update.  
    - **Configuration:**  
      - Operation: Get  
      - Contact ID: Same dynamic expression as update node  
      - Fields: ["organizations"] to limit data fetched.  
    - **Key Expressions/Variables:** Same contactId expression.  
    - **Input/Output:** Input from update node; outputs full contact data with organizations field.  
    - **Version-specific Requirements:** OAuth2 credentials required.  
    - **Potential Failures:**  
      - Contact not found if previous update failed or contact deleted.  
      - API errors or authentication issues.  
    - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                   | Input Node(s)           | Output Node(s)       | Sticky Note                                           |
|---------------------|---------------------|---------------------------------|------------------------|----------------------|------------------------------------------------------|
| On clicking 'execute'| Manual Trigger      | Start workflow manually          | —                      | Google Contacts       |                                                      |
| Google Contacts      | Google Contacts Node| Create a new contact             | On clicking 'execute'   | Google Contacts1      |                                                      |
| Google Contacts1     | Google Contacts Node| Update contact's company info    | Google Contacts         | Google Contacts2      |                                                      |
| Google Contacts2     | Google Contacts Node| Retrieve updated contact details | Google Contacts1        | —                    |                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "On clicking 'execute'".  
   - No parameters required.

2. **Create Google Contacts Node for Contact Creation**  
   - Add a **Google Contacts** node named "Google Contacts".  
   - Set operation to **Create** (default when providing contact fields).  
   - Enter **Given Name**: `n8n`  
   - Enter **Family Name**: `n8n`  
   - Leave Additional Fields empty.  
   - Set credentials to a valid **Google Contacts OAuth2** credential.

3. **Create Google Contacts Node for Updating Contact**  
   - Add another **Google Contacts** node named "Google Contacts1".  
   - Set operation to **Update**.  
   - For **Contact ID**, use expression: `{{$node["Google Contacts"].json["contactId"]}}` to dynamically reference the created contact's ID.  
   - Under **Update Fields**, configure:  
     - Company Name: `n8n`  
     - Company Title: `n8n`  
     - Company Domain: `n8n.io`  
     - Mark company as current.  
   - Use the same Google Contacts OAuth2 credentials.

4. **Create Google Contacts Node for Retrieving Contact**  
   - Add a third **Google Contacts** node named "Google Contacts2".  
   - Set operation to **Get**.  
   - For **Contact ID**, use the same expression referencing the previous nodes: `{{$node["Google Contacts"].json["contactId"]}}` or from "Google Contacts1" output.  
   - Specify fields to retrieve: `organizations`  
   - Use the same Google Contacts OAuth2 credentials.

5. **Connect the Nodes**  
   - Connect the output of "On clicking 'execute'" to "Google Contacts".  
   - Connect "Google Contacts" output to "Google Contacts1".  
   - Connect "Google Contacts1" output to "Google Contacts2".

6. **Save and Test**  
   - Ensure OAuth2 credentials are authorized and valid.  
   - Execute the workflow manually to verify contact creation, update, and retrieval.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|-----------------------------------------------------------------------------------------------|----------------------------------------|
| This workflow requires a valid Google Contacts OAuth2 credential with appropriate scopes.     | Google Contacts OAuth2 setup in n8n    |
| The contact fields updated include company details, which are part of the organizations data. | Google Contacts API documentation      |
| Testing workflow execution requires internet connectivity and Google API availability.        |                                        |

---

This documentation provides a comprehensive understanding and reproduction guide for the "Create, update and get a contact in Google Contacts" workflow, ensuring users and AI agents can effectively utilize and adapt it.