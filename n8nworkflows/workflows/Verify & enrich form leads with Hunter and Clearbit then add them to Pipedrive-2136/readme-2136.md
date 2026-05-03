Verify & enrich form leads with Hunter and Clearbit then add them to Pipedrive

https://n8nworkflows.xyz/workflows/verify---enrich-form-leads-with-hunter-and-clearbit-then-add-them-to-pipedrive-2136


# Verify & enrich form leads with Hunter and Clearbit then add them to Pipedrive

### 1. Workflow Overview

This workflow automates the process of verifying, enriching, and adding new leads collected from an online form into the Pipedrive CRM system. It is designed to save time and reduce errors when handling lead data by integrating email verification and enrichment services before CRM entry.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures lead data from an online form submission in n8n.
- **1.2 Email Verification:** Validates the submitted email address using Hunter.io.
- **1.3 Person Existence Check:** Searches Pipedrive to verify if the person already exists.
- **1.4 Data Enrichment:** Uses Clearbit to enrich the lead data if the person is new.
- **1.5 Organization Handling:** Checks if the organization exists in Pipedrive, creating it if needed.
- **1.6 Person Creation:** Adds the new person record to Pipedrive.
- **1.7 Lead Creation:** Creates a lead in Pipedrive associated with the person and organization.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a form submission occurs, capturing the lead's email address and other basic data.

- **Nodes Involved:**  
  - `n8n Form Trigger`  
  - `Sticky Note1`

- **Node Details:**

  - **n8n Form Trigger**  
    - *Type:* Trigger node  
    - *Role:* Starts workflow on form submission  
    - *Configuration:*  
      - Webhook path set to a unique identifier  
      - Form titled "Contact us" with a single field "What's your business email?"  
      - Form description: "We'll get back to you soon"  
    - *Input/Output:* No input; outputs form data JSON with field values  
    - *Edge Cases:* If form structure changes or webhook misconfigured, trigger may fail  
    - *Sticky Note1:* "You can exchange this with any form you like (e.g., Typeform, Google Forms, Survey Monkey...)"  

  - **Sticky Note1**  
    - *Type:* Informational note  
    - *Content:* Guidance on replacing the form trigger if desired  

#### 2.2 Email Verification

- **Overview:**  
  Validates the email address submitted via the form by querying Hunter.io's email verification API.

- **Nodes Involved:**  
  - `Verify email with Hunter`  
  - `Check if the email is valid`  
  - `Email is not valid, do nothing`

- **Node Details:**

  - **Verify email with Hunter**  
    - *Type:* Hunter node  
    - *Role:* Validates email deliverability and status  
    - *Configuration:*  
      - Email taken from form submission field `"What's your business email?"`  
      - Operation: `emailVerifier`  
      - Credentials: Hunter.io API credentials required  
    - *Input/Output:* Input from form trigger; output includes email verification status  
    - *Edge Cases:*  
      - API authentication errors  
      - Rate limiting from Hunter.io  
      - Invalid or malformed email inputs  
 
  - **Check if the email is valid**  
    - *Type:* IF node  
    - *Role:* Branches workflow based on Hunter.io email verification result  
    - *Configuration:* Condition checks if `status` field equals `"valid"`  
    - *Input:* Output from Hunter node  
    - *Output:*  
      - True branch: proceeds if email is valid  
      - False branch: skips processing if invalid  
    - *Edge Cases:* Expression failures if `status` field missing or unexpected values  

  - **Email is not valid, do nothing**  
    - *Type:* NoOp node  
    - *Role:* Terminates workflow path for invalid emails without error  
    - *Input:* False branch of email validity IF node  
    - *Output:* None  

#### 2.3 Person Existence Check

- **Overview:**  
  Searches Pipedrive to determine if a person with the validated email already exists, preventing duplicates.

- **Nodes Involved:**  
  - `Search for person in Pipedrive`  
  - `Is this a new person?`  
  - `Person already exists in Pipedrive, do nothing`

- **Node Details:**

  - **Search for person in Pipedrive**  
    - *Type:* Pipedrive node  
    - *Role:* Searches Pipedrive persons by email address  
    - *Configuration:*  
      - Resource: `person`  
      - Operation: `search`  
      - Search term: Email from Hunter node output  
      - Credentials: Pipedrive API credentials required  
      - Always outputs data to allow empty results  
    - *Input:* From valid email branch  
    - *Output:* Search results for persons matching email  
    - *Edge Cases:* API limit or auth errors; empty results if person not found  

  - **Is this a new person?**  
    - *Type:* IF node  
    - *Role:* Checks if search returned any existing person (checks if `id` exists)  
    - *Configuration:* Condition checks if `id` does *not* exist (new person)  
    - *Input:* From Pipedrive search node  
    - *Output:*  
      - True branch: person not found, proceed to enrichment  
      - False branch: person exists, stop processing  
    - *Edge Cases:* Expression failures if search output malformed  

  - **Person already exists in Pipedrive, do nothing**  
    - *Type:* NoOp node  
    - *Role:* Ends workflow branch if person exists to avoid duplication  

#### 2.4 Data Enrichment

- **Overview:**  
  Enriches the new lead's information using Clearbit's person enrichment API to obtain detailed profile and employment data.

- **Nodes Involved:**  
  - `Clearbit`

- **Node Details:**

  - **Clearbit**  
    - *Type:* Clearbit node  
    - *Role:* Retrieves enriched person data based on email  
    - *Configuration:*  
      - Resource: `person`  
      - Email: from the email field of the `Check if the email is valid` node output  
      - Credentials: Clearbit API credentials required  
    - *Input:* True branch of new person IF node  
    - *Output:* Enriched person data including name, employment, and email  
    - *Edge Cases:* API errors, rate limits, missing data for some emails  

#### 2.5 Organization Handling

- **Overview:**  
  Checks if the organization associated with the enriched person exists in Pipedrive, creating it if absent.

- **Nodes Involved:**  
  - `Search for organization in Pipedrive`  
  - `Is this a new organization?`  
  - `Create Organization`

- **Node Details:**

  - **Search for organization in Pipedrive**  
    - *Type:* Pipedrive node  
    - *Role:* Searches organizations by name from Clearbit employment data  
    - *Configuration:*  
      - Resource: `organization`  
      - Operation: `search`  
      - Search term: Employment organization name from Clearbit data  
      - Credentials: Pipedrive API credentials required  
      - Always outputs data  
    - *Input:* Output of Clearbit node  
    - *Output:* Organization search results  
    - *Edge Cases:* API errors, empty results if org not found  

  - **Is this a new organization?**  
    - *Type:* IF node  
    - *Role:* Checks if org search returned no organization (checks if `id` exists)  
    - *Configuration:* Condition tests absence of `id` (new organization)  
    - *Input:* From org search node  
    - *Output:*  
      - True: org not found, proceed to create org  
      - False: org exists, proceed to person creation with existing org  
    - *Edge Cases:* Expression errors if search output malformed  

  - **Create Organization**  
    - *Type:* Pipedrive node  
    - *Role:* Creates a new organization in Pipedrive  
    - *Configuration:*  
      - Resource: `organization`  
      - Name: Employment organization name from Clearbit data  
      - Credentials: Pipedrive API  
    - *Input:* True branch of new org IF node  
    - *Output:* Newly created organization data  
    - *Edge Cases:* API errors, name conflicts  

#### 2.6 Person Creation

- **Overview:**  
  Creates a new person in Pipedrive, associating them with the organization found or created previously.

- **Nodes Involved:**  
  - `Create Person`

- **Node Details:**

  - **Create Person**  
    - *Type:* Pipedrive node  
    - *Role:* Adds a new person record to Pipedrive  
    - *Configuration:*  
      - Resource: `person`  
      - Name: Full name from Clearbit data  
      - Email: Email from Clearbit data  
      - Organization ID: ID from either existing or newly created org  
      - Credentials: Pipedrive API  
    - *Input:* Output of either `Create Organization` or False branch of `Is this a new organization?`  
    - *Output:* Newly created person data  
    - *Edge Cases:* API errors, missing mandatory fields  

#### 2.7 Lead Creation

- **Overview:**  
  Creates a new lead in Pipedrive linked to the newly created person and their organization.

- **Nodes Involved:**  
  - `Create lead`

- **Node Details:**

  - **Create lead**  
    - *Type:* Pipedrive node  
    - *Role:* Creates a lead associated with the person and organization in Pipedrive  
    - *Configuration:*  
      - Resource: `lead`  
      - Title: Concatenation of person’s name and organization name  
      - Person ID: ID of created person  
      - Organization ID: ID of organization  
      - Associate with person as primary  
      - Credentials: Pipedrive API  
    - *Input:* Output of `Create Person` node  
    - *Output:* Newly created lead data  
    - *Edge Cases:* API errors, missing references  

---

### 3. Summary Table

| Node Name                       | Node Type           | Functional Role                         | Input Node(s)                | Output Node(s)                      | Sticky Note                                                                                       |
|--------------------------------|---------------------|---------------------------------------|-----------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note                    | Sticky Note         | Setup instructions                    |                             |                                   | 1. Add your **Hunter.io**, **Clearbit** and **Pipedrive** credentials 2. Click test 3. Activate  |
| n8n Form Trigger               | Form Trigger        | Triggers workflow on form submission  |                             | Verify email with Hunter           | You can exchange this with any form you like (e.g., Typeform, Google Forms, Survey Monkey...)    |
| Verify email with Hunter       | Hunter              | Validates email via Hunter.io         | n8n Form Trigger             | Check if the email is valid        |                                                                                                 |
| Check if the email is valid    | IF                  | Checks if email status is "valid"     | Verify email with Hunter     | Search for person in Pipedrive, Email is not valid, do nothing |                                                                                                 |
| Email is not valid, do nothing | NoOp                | Ends workflow for invalid emails      | Check if the email is valid  |                                   |                                                                                                 |
| Search for person in Pipedrive | Pipedrive           | Searches for existing person by email | Check if the email is valid  | Is this a new person?              |                                                                                                 |
| Is this a new person?          | IF                  | Determines if person exists            | Search for person in Pipedrive | Clearbit, Person already exists in Pipedrive, do nothing |                                                                                                 |
| Person already exists in Pipedrive, do nothing | NoOp                | Ends workflow for existing person      | Is this a new person?        |                                   |                                                                                                 |
| Clearbit                      | Clearbit            | Enriches person data                   | Is this a new person?        | Search for organization in Pipedrive |                                                                                                 |
| Search for organization in Pipedrive | Pipedrive           | Searches for existing organization    | Clearbit                    | Is this a new organization?        |                                                                                                 |
| Is this a new organization?   | IF                  | Determines if organization exists      | Search for organization in Pipedrive | Create Organization, Create Person |                                                                                                 |
| Create Organization           | Pipedrive           | Creates new organization if needed     | Is this a new organization?  | Create Person                     |                                                                                                 |
| Create Person                 | Pipedrive           | Creates new person linked to org       | Create Organization / Is this a new organization? (False branch) | Create lead                       |                                                                                                 |
| Create lead                   | Pipedrive           | Creates lead linked to person and org  | Create Person               |                                   |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Type: `n8n Form Trigger`  
   - Configure webhook path (e.g., unique ID)  
   - Set form title "Contact us"  
   - Add one field labeled "What's your business email?"  
   - Add form description "We'll get back to you soon"  

2. **Add a Hunter node to verify email:**  
   - Type: `Hunter`  
   - Operation: `emailVerifier`  
   - Email: use expression to get form field `{{ $json["What's your business email?"] }}`  
   - Set Hunter API credentials  

3. **Add IF node to check email validity:**  
   - Type: `IF`  
   - Condition: Check if `{{$json.status}}` equals `"valid"`  
   - Connect Hunter node output to this node  

4. **Add NoOp node for invalid emails:**  
   - Type: `NoOp`  
   - Connect to the false branch of the IF node  

5. **Add Pipedrive node to search persons:**  
   - Type: `Pipedrive`  
   - Resource: `person`  
   - Operation: `search`  
   - Term: `{{ $json.email }}` (from Hunter node output)  
   - Set Pipedrive API credentials  
   - Connect true branch of IF node to this node  

6. **Add IF node to check if person exists:**  
   - Type: `IF`  
   - Condition: Check if `id` does not exist in search result (`number` operation `notExists`)  
   - Connect search person node to this IF  

7. **Add NoOp node for existing persons:**  
   - Type: `NoOp`  
   - Connect false branch of person existence IF node  

8. **Add Clearbit node to enrich person data:**  
   - Type: `Clearbit`  
   - Resource: `person`  
   - Email: `{{ $('Check if the email is valid').item.json.email }}`  
   - Set Clearbit API credentials  
   - Connect true branch of new person IF node  

9. **Add Pipedrive node to search organizations:**  
   - Type: `Pipedrive`  
   - Resource: `organization`  
   - Operation: `search`  
   - Term: `{{ $json.employment.name }}` (from Clearbit output)  
   - Connect Clearbit node to this node  

10. **Add IF node to check if organization exists:**  
    - Type: `IF`  
    - Condition: Check if `id` does not exist in org search result  
    - Connect org search node to this IF  

11. **Add Pipedrive node to create organization:**  
    - Type: `Pipedrive`  
    - Resource: `organization`  
    - Name: `{{ $('Clearbit').item.json.employment.name }}`  
    - Connect true branch of org existence IF node  

12. **Add Pipedrive node to create person:**  
    - Type: `Pipedrive`  
    - Resource: `person`  
    - Name: `{{ $('Clearbit').item.json.name.fullName }}`  
    - Email: `{{ $('Clearbit').item.json.email }}`  
    - Organization ID: `{{ $json.id }}` (from newly created or found org)  
    - Connect false branch of org existence IF node and output of create organization node to this node (merge or chain accordingly)  

13. **Add Pipedrive node to create lead:**  
    - Type: `Pipedrive`  
    - Resource: `lead`  
    - Title: `{{ $json.name }} from {{ $json.org_id.name }}`  
    - Person ID: `{{ $json.id }}` (from created person)  
    - Organization ID: `{{ $json.org_id.value }}`  
    - Connect create person node to this node  

14. **Add Sticky Notes for documentation:**  
    - One at the top with setup instructions ("Add your Hunter.io, Clearbit and Pipedrive credentials...")  
    - One near the form trigger node explaining alternative form options  

15. **Activate credentials for Hunter.io, Clearbit, and Pipedrive:**  
    - Configure API keys and OAuth2 as required in n8n credentials manager  

16. **Test the workflow:**  
    - Use the webhook URL from the form trigger node to submit test form data  
    - Verify logs and node outputs for successful execution  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Setup requires valid API credentials for Hunter.io, Clearbit, and Pipedrive.                              | Credentials section in n8n                                                                                |
| Workflow can be adapted to other form providers like Typeform, Google Forms, SurveyMonkey, etc.            | Sticky Note near form trigger node                                                                        |
| Consider adding filters or error handling for invalid emails or existing persons based on specific needs. | Workflow description and block analysis sections                                                         |
| Official n8n documentation and API docs for Hunter.io, Clearbit, and Pipedrive are helpful for customization | https://docs.n8n.io/, https://hunter.io/api, https://clearbit.com/docs, https://pipedrive.readme.io/       |

---

This detailed structured document enables developers and automation agents to fully understand, reproduce, and modify the workflow to suit their lead management needs.