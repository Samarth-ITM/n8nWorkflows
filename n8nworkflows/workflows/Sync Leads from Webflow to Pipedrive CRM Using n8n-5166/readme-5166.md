Sync Leads from Webflow to Pipedrive CRM Using n8n

https://n8nworkflows.xyz/workflows/sync-leads-from-webflow-to-pipedrive-crm-using-n8n-5166


# Sync Leads from Webflow to Pipedrive CRM Using n8n

---

### 1. Workflow Overview

This workflow automates the synchronization of new leads submitted via Webflow forms with a Pipedrive CRM account. It is designed to streamline CRM data entry by ensuring that each lead submission is properly associated with the relevant organization and contact, thereby minimizing duplicates and manual errors.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & Website Extraction:** Captures form submissions from Webflow and extracts the domain from the lead’s email address to identify the organization.
- **1.2 Organization Verification & Creation:** Searches Pipedrive for an existing organization by domain; creates a new organization if none exists.
- **1.3 Person Verification, Creation, and Lead Management:** Checks if the person (lead) exists in Pipedrive; adds notes for existing persons or creates new persons and leads accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Website Extraction

- **Overview:**  
  This block triggers the workflow upon a Webflow form submission and extracts the email domain from the submitted lead’s email address for later use in organization identification.

- **Nodes Involved:**  
  - Webflow Trigger  
  - Website (Code)  
  - Sticky Note (explaining this block)

- **Node Details:**

  - **Webflow Trigger**  
    - Type: Webhook Trigger node specialized for Webflow form submissions.  
    - Configuration: Listens for form submissions on a specific Webflow site (site ID configured). Uses OAuth2 credentials for authentication.  
    - Expressions: None directly, but outputs the entire form submission JSON payload.  
    - Input: External Webflow form submission.  
    - Output: JSON containing form data including Name, Email, Company, and submission metadata.  
    - Potential Failures: Webhook connectivity issues, OAuth token expiration, malformed payloads.  

  - **Website (Code)**  
    - Type: Code node (JavaScript) used for data transformation.  
    - Configuration: Extracts the domain from the email by splitting the email string at '@' and returning the domain part as `website`.  
    - Expression: `return {website: $input.first().json.payload.data.Email.split("@")[1]}`  
    - Input: Output from Webflow Trigger node.  
    - Output: JSON containing the extracted website domain.  
    - Potential Failures: Missing or malformed email address in payload causing split errors.  

  - **Sticky Note**  
    - Purpose: Explains that the email is split to extract the website domain for use in subsequent organization lookup.  
    - No technical role.

---

#### 2.2 Organization Verification & Creation

- **Overview:**  
  This block uses the extracted website domain to search for an existing organization in Pipedrive. If no matching organization is found, a new organization is created using the company name and website from the form submission.

- **Nodes Involved:**  
  - Get Organization  
  - Organization Exists? (If node)  
  - Add Organization  
  - Pipedrive (search person node, reused later)  
  - Sticky Notes (explaining logic)

- **Node Details:**

  - **Get Organization**  
    - Type: Pipedrive node configured to search organizations.  
    - Configuration: Searches organizations by the extracted website domain (`term` set to `{{$json.website}}`). Uses Pipedrive API credentials.  
    - Input: Output from Website node with domain.  
    - Output: JSON array of organizations matching the domain.  
    - Potential Failures: API rate limits, network issues, invalid credentials.  

  - **Organization Exists? (If node)**  
    - Type: If node to check if organization search returned results.  
    - Configuration: Checks if the JSON output from Get Organization is not empty (object not empty).  
    - Input: Output of Get Organization node.  
    - Output: Two branches: true (organization exists) and false (organization does not exist).  
    - Potential Failures: Expression errors if input JSON is malformed.  

  - **Add Organization**  
    - Type: Pipedrive node to create a new organization.  
    - Configuration: Creates organization with the company name from Webflow form (`{{$node["Webflow Trigger"].json.payload.data.Company}}`) and custom property to store the website domain.  
    - Input: Triggered if no organization found.  
    - Output: JSON of created organization with its ID.  
    - Potential Failures: Missing company name, API failures, validation errors.  

  - **Pipedrive (search person) node**  
    - Note: This node is also used later but is connected here for organization branch continuation.  
    - No direct action in this block; serves as next step in workflow.  

  - **Sticky Note1**  
    - Explains the organization check process: search by domain, create if absent.  

---

#### 2.3 Person Verification, Creation, and Lead Management

- **Overview:**  
  This block checks if the person (lead) already exists in Pipedrive by searching with the submitted email. If the person exists, a note is added documenting the form submission. If not, the workflow creates the person, adds a note, and creates a lead associated with that person.

- **Nodes Involved:**  
  - Pipedrive (Person Search)  
  - Person Exists? (If node)  
  - Add Note 2  
  - Create Person  
  - Add Note  
  - Create Lead  
  - Sticky Note2  

- **Node Details:**

  - **Pipedrive (Person Search)**  
    - Type: Pipedrive node searching for existing person records by email (`term` expression: `{{$node["Webflow Trigger"].item.json.payload.data.Email}}`).  
    - Configuration: Search operation under “person” resource.  
    - Input: From Organization Exists? node (both true and false branches).  
    - Output: JSON array of matched persons or empty.  
    - Always outputs data to allow downstream conditional checks.  
    - Potential Failures: API limits, invalid email format, network errors.  

  - **Person Exists? (If node)**  
    - Type: Conditional node checking if person search returned results.  
    - Configuration: Checks if JSON output is not empty (person found).  
    - Input: Output of Pipedrive person search node.  
    - Output: Two branches: true (person exists), false (person does not exist).  

  - **Add Note 2**  
    - Type: Pipedrive node adding a note linked to an existing person.  
    - Configuration: Creates a note with content including the form name and page path, linked to the existing person’s ID (`{{$json.id}}`).  
    - Input: True branch of Person Exists? node.  
    - Output: Confirmation of note creation.  
    - Potential Failures: Missing person ID, API errors.  

  - **Create Person**  
    - Type: Pipedrive node creating a new person.  
    - Configuration: Uses form submission data for person’s name and email. Associates with organization ID from earlier organization check or creation (`{{$node["Organization Exists?"].item.json.id}}`).  
    - Input: False branch of Person Exists? node.  
    - Output: JSON of created person with ID.  
    - Potential Failures: Missing required fields, API failures.  

  - **Add Note**  
    - Type: Pipedrive node adding a note linked to the newly created person.  
    - Configuration: Similar content format as Add Note 2, linked to new person ID (`{{$json.id}}`).  
    - Input: Output of Create Person node.  
    - Output: Confirmation of note creation.  

  - **Create Lead**  
    - Type: Pipedrive node creating a new lead.  
    - Configuration: Title set to the person’s name, associates the lead with the newly created person ID (`{{$node["Create Person"].item.json.id}}`).  
    - Input: Output of Add Note node.  
    - Output: Confirmation of lead creation.  
    - Potential Failures: Missing person ID, API errors.  

  - **Sticky Note2**  
    - Explains person existence check: if found, add note; if not, create person, add note, create lead.

---

### 3. Summary Table

| Node Name          | Node Type               | Functional Role                               | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                 |
|--------------------|-------------------------|-----------------------------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------|
| Webflow Trigger    | Webflow Trigger          | Entry point: receives Webflow form submissions| -                     | Website               | This n8n template automates capture of leads from Webflow to Pipedrive CRM... (Sticky Note3)|
| Website            | Code                    | Extracts domain from email to identify org    | Webflow Trigger       | Get Organization      | ## 1. Extract website from email... (Sticky Note)                                          |
| Get Organization   | Pipedrive (Search Org)  | Searches org by extracted domain               | Website               | Organization Exists?  | ## 2. Check if the organization already exists... (Sticky Note1)                           |
| Organization Exists?| If                      | Branches based on org existence                 | Get Organization      | Pipedrive, Add Organization | ## 2. Check if the organization already exists... (Sticky Note1)                    |
| Add Organization   | Pipedrive (Create Org)  | Creates org if not found                        | Organization Exists?  | Pipedrive             | ## 2. Check if the organization already exists... (Sticky Note1)                           |
| Pipedrive          | Pipedrive (Search Person)| Searches person by email                        | Organization Exists?, Add Organization | Person Exists?       | ## 3. Check if the person already exists... (Sticky Note2)                                |
| Person Exists?     | If                      | Branches on person existence                    | Pipedrive             | Add Note 2, Create Person | ## 3. Check if the person already exists... (Sticky Note2)                              |
| Add Note 2         | Pipedrive (Create Note) | Adds submission note to existing person        | Person Exists? (true) | -                     | ## 3. Check if the person already exists... (Sticky Note2)                                |
| Create Person      | Pipedrive (Create Person)| Creates person if not found                     | Person Exists? (false)| Add Note               | ## 3. Check if the person already exists... (Sticky Note2)                                |
| Add Note           | Pipedrive (Create Note) | Adds submission note to new person              | Create Person         | Create Lead            | ## 3. Check if the person already exists... (Sticky Note2)                                |
| Create Lead        | Pipedrive (Create Lead) | Creates a new lead linked to the person         | Add Note              | -                      | ## 3. Check if the person already exists... (Sticky Note2)                                |
| Sticky Note        | Sticky Note             | Explains domain extraction step                | -                     | -                      | ## 1. Extract website from email                                                           |
| Sticky Note1       | Sticky Note             | Explains organization lookup and creation     | -                     | -                      | ## 2. Check if the organization already exists                                             |
| Sticky Note2       | Sticky Note             | Explains person existence and lead creation   | -                     | -                      | ## 3. Check if the person already exists                                                   |
| Sticky Note3       | Sticky Note             | General workflow overview and use cases       | -                     | -                      | This n8n template automates the process of capturing leads from Webflow form submissions...|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webflow Trigger node:**  
   - Type: Webflow Trigger  
   - Configure with your Webflow site ID where the form is hosted.  
   - Authenticate with OAuth2 credentials for your Webflow account.  
   - This node will trigger when a Webflow form is submitted.  

2. **Add a Code node named "Website":**  
   - Type: Code (JavaScript)  
   - Paste code to extract domain from the email:  
     ```js
     return {website: $input.first().json.payload.data.Email.split("@")[1]};
     ```  
   - Connect the output of the Webflow Trigger node into this node.  

3. **Add a Pipedrive node named "Get Organization":**  
   - Type: Pipedrive  
   - Operation: Search  
   - Resource: Organization  
   - Set search term to: `{{$json.website}}` (value from previous node)  
   - Authenticate using your Pipedrive API credentials.  
   - Connect the output of the Website node to this node.  

4. **Add an If node named "Organization Exists?":**  
   - Type: If  
   - Condition: Check if the JSON response from "Get Organization" is not empty (object not empty).  
   - Connect the output of Get Organization to this node.  

5. **Add a Pipedrive node named "Add Organization":**  
   - Type: Pipedrive  
   - Operation: Create  
   - Resource: Organization  
   - Set "Name" to: `{{$node["Webflow Trigger"].json.payload.data.Company}}`  
   - Add custom property "website" or similar, set to website domain from "Website" node.  
   - Authenticate with Pipedrive credentials.  
   - Connect the 'false' output of "Organization Exists?" to this node.  

6. **Add a Pipedrive node named "Pipedrive" (Person Search):**  
   - Type: Pipedrive  
   - Operation: Search  
   - Resource: Person  
   - Set term to: `{{$node["Webflow Trigger"].item.json.payload.data.Email}}`  
   - Authenticate with Pipedrive credentials.  
   - Connect both 'true' output from "Organization Exists?" and output of "Add Organization" node into this node.  

7. **Add an If node named "Person Exists?":**  
   - Type: If  
   - Condition: Check if the JSON output of the "Pipedrive" person search node is not empty.  
   - Connect output of Pipedrive (Person Search) node to this node.  

8. **Add a Pipedrive node named "Add Note 2":**  
   - Type: Pipedrive  
   - Operation: Create  
   - Resource: Note  
   - Content:  
     ```
     Form: {{$node["Webflow Trigger"].json.payload.name}}  
     Page: {{$node["Webflow Trigger"].json.payload.publishedPath}}
     ```  
   - Link this note to the existing person by setting `person_id` to `{{$json.id}}`.  
   - Authenticate with Pipedrive credentials.  
   - Connect the 'true' output of "Person Exists?" to this node.  

9. **Add a Pipedrive node named "Create Person":**  
   - Type: Pipedrive  
   - Operation: Create  
   - Resource: Person  
   - Name: `{{$node["Webflow Trigger"].item.json.payload.data.Name}}`  
   - Email: `[{{$node["Webflow Trigger"].item.json.payload.data.Email}}]`  
   - Organization ID: `{{$node["Organization Exists?"].item.json.id}}`  
   - Authenticate with Pipedrive credentials.  
   - Connect the 'false' output of "Person Exists?" to this node.  

10. **Add a Pipedrive node named "Add Note":**  
    - Type: Pipedrive  
    - Operation: Create  
    - Resource: Note  
    - Content:  
      ```
      Form: {{$node["Webflow Trigger"].json.payload.name}}  
      Page: {{$node["Webflow Trigger"].json.payload.publishedPath}}
      ```  
    - Link note to new person by setting `person_id` to `{{$json.id}}`.  
    - Authenticate with Pipedrive credentials.  
    - Connect output of "Create Person" node to this node.  

11. **Add a Pipedrive node named "Create Lead":**  
    - Type: Pipedrive  
    - Operation: Create  
    - Resource: Lead  
    - Title: `{{$json.person.name}}` (or `{{$node["Create Person"].item.json.name}}`)  
    - Associate lead with the person by setting `person_id` to `{{$node["Create Person"].item.json.id}}`  
    - Authenticate with Pipedrive credentials.  
    - Connect output of "Add Note" node to this node.  

12. **Add Sticky Notes (optional) for documentation:**  
    - Add notes explaining each major step for user clarity.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This n8n template automates the process of capturing leads from Webflow form submissions and syncing them with your Pipedrive CRM. It ensures that each submission is accurately associated with the correct organization and contact in Pipedrive, streamlining lead management and minimizing duplicates. Use cases include sales teams automating CRM entry, marketing teams capturing leads from landing pages, or any business improving Webflow-to-CRM integration workflows. | General workflow context and use cases (Sticky Note3 in workflow)                                 |
| The workflow assumes Webflow form submissions include the lead’s email address, extracts the domain from the email to match or create the organization in Pipedrive, but does not handle lead scoring or enrichment by default. It can be extended to include Slack notifications, email follow-ups, or enrichment APIs.                                                                                                                                           | Workflow assumptions and extensibility notes                                                     |
| Webflow Trigger node requires OAuth2 credentials linked to your Webflow account. Similarly, all Pipedrive nodes require valid API credentials with appropriate permissions to search, create, and update organizations, persons, notes, and leads.                                                                                                                                                                                                           | Credentials configuration note                                                                  |
| For more information on Pipedrive API and n8n Pipedrive node usage, consult: https://pipedrive.readme.io/docs/api-overview and https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.pipedrive/                                                                                                                                                                                                                                                  | Documentation links for Pipedrive API and n8n integration                                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---