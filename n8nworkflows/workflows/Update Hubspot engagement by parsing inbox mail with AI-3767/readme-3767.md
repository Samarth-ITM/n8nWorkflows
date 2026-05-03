Update Hubspot engagement by parsing inbox mail with AI

https://n8nworkflows.xyz/workflows/update-hubspot-engagement-by-parsing-inbox-mail-with-ai-3767


# Update Hubspot engagement by parsing inbox mail with AI

### 1. Workflow Overview

This workflow automates the process of updating HubSpot CRM engagements based on incoming emails. It is designed primarily for Customer Success Managers, sales, support, and marketing teams who want to eliminate manual data entry when logging email interactions with customers.

**Target Use Cases:**  
- Automatically logging email interactions as engagements in HubSpot  
- Creating new contacts in HubSpot if the email sender is not found  
- Streamlining CRM updates triggered by new incoming emails  

**Logical Blocks:**

- **1.1 Input Reception:** Trigger on new incoming emails via IMAP.  
- **1.2 AI Email Parsing:** Use OpenAI to extract structured contact and company data from the email content.  
- **1.3 HubSpot Contact Search:** Query HubSpot CRM to check if the sender already exists as a contact.  
- **1.4 Conditional HubSpot Update:**  
  - If contact exists, update engagement linked to that contact.  
  - If contact does not exist, create a new contact and then log the engagement.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new incoming emails via an IMAP account and outputs the raw email data for processing.

- **Nodes Involved:**  
  - When an email is received  
  - Sticky Note (Setup instructions)

- **Node Details:**

  - **When an email is received**  
    - Type: Email Read (IMAP)  
    - Role: Trigger node that detects new emails in the configured inbox.  
    - Configuration: Uses IMAP credentials; force reconnect set to 3 to ensure stable connection.  
    - Inputs: None (trigger)  
    - Outputs: Raw email data including subject, from, to, textHtml, textPlain, date, etc.  
    - Edge Cases: Connection drops, authentication failures, empty emails, malformed content.  
    - Notes: Replace default IMAP node with your email provider if needed.

  - **Sticky Note (Set receiving email account)**  
    - Provides instructions to configure the email trigger node.

---

#### 1.2 AI Email Parsing

- **Overview:**  
  Parses the incoming email content using OpenAI to extract structured contact information such as name, email, phone, company, and address.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Set the output Json  
  - Parse the mail with AI  
  - Sticky Notes (Prompt upgrade suggestion)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides the language model interface to GPT-4o-mini for parsing.  
    - Configuration: Model set to "gpt-4o-mini" with default options.  
    - Inputs: Email content and prompt messages from "Parse the mail with AI" node.  
    - Outputs: Raw AI response for further parsing.  
    - Credentials: OpenAI API key required.  
    - Edge Cases: API rate limits, malformed prompts, unexpected AI responses.

  - **Set the output Json**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses AI raw output into a structured JSON format based on a provided schema example.  
    - Configuration: JSON schema example includes fields like first name, last name, email, phone, company, address, etc.  
    - Inputs: AI raw response from OpenAI Chat Model.  
    - Outputs: Structured JSON with extracted contact info.  
    - Edge Cases: AI output not matching schema, parsing failures.

  - **Parse the mail with AI**  
    - Type: Langchain Chain LLM  
    - Role: Orchestrates the prompt sent to the AI model and applies the output parser.  
    - Configuration: Prompt requests extraction of key contact info from the email content, returning JSON.  
    - Inputs: Raw email content from "When an email is received" node.  
    - Outputs: Structured contact info JSON.  
    - Edge Cases: Missing email content, AI misinterpretation.

  - **Sticky Note (Upgrade the prompt!)**  
    - Suggests improving the AI prompt for better parsing accuracy.

---

#### 1.3 HubSpot Contact Search

- **Overview:**  
  Searches HubSpot CRM for a contact matching the parsed email address to determine if the contact exists.

- **Nodes Involved:**  
  - Search for the contact via email  
  - Sticky Note (HubSpot integration explanation)

- **Node Details:**

  - **Search for the contact via email**  
    - Type: HubSpot node (search operation)  
    - Role: Queries HubSpot contacts filtered by the parsed email address.  
    - Configuration: Uses OAuth2 authentication; filter on email property with value from parsed AI output.  
    - Inputs: Structured contact info JSON from AI parsing.  
    - Outputs: Search results containing contact details if found.  
    - Edge Cases: API limits, no contact found, authentication errors.

  - **Sticky Note (HubSpot integration)**  
    - Explains the logic: search contact, create engagement if found, else create contact then engagement.

---

#### 1.4 Conditional HubSpot Update

- **Overview:**  
  Based on the search result, decides whether to create a new contact or update an existing one, then logs the email as an engagement linked to the contact.

- **Nodes Involved:**  
  - contact exists? (IF node)  
  - Edit Fields (Set node)  
  - Creates contact (HubSpot node)  
  - Creates an email engagement (HubSpot node)  
  - Sticky Note (Contact me)

- **Node Details:**

  - **contact exists?**  
    - Type: IF node  
    - Role: Checks if the contact search returned a valid contact ID.  
    - Configuration: Condition tests if `$json.id` exists (contact found).  
    - Inputs: Output from HubSpot search node.  
    - Outputs: Two branches — true (contact exists), false (contact does not exist).  
    - Edge Cases: Null or malformed search results.

  - **Edit Fields**  
    - Type: Set node  
    - Role: Prepares data for engagement creation by setting the contact ID (`vid`) from the existing contact.  
    - Configuration: Sets `vid` to `$json.id`.  
    - Inputs: True branch from IF node.  
    - Outputs: Data for engagement creation.

  - **Creates contact**  
    - Type: HubSpot node (create contact)  
    - Role: Creates a new contact in HubSpot using parsed AI data.  
    - Configuration: Maps parsed fields (email, city, country, job title, last name, postal code, website, company, phone, street) to HubSpot contact properties.  
    - Inputs: False branch from IF node.  
    - Outputs: New contact data including contact ID.  
    - Credentials: OAuth2 HubSpot account.  
    - Edge Cases: API errors, missing required fields.

  - **Creates an email engagement**  
    - Type: HubSpot node (create engagement)  
    - Role: Logs the email as an engagement linked to the contact ID (`vid`).  
    - Configuration: Engagement type set to "email"; metadata includes email HTML or plain text, subject, toEmail, fromEmail; associations link to contact ID.  
    - Inputs: From both "Edit Fields" and "Creates contact" nodes.  
    - Credentials: OAuth2 HubSpot account.  
    - Edge Cases: Engagement creation failure, invalid contact ID.

  - **Sticky Note (Contact me)**  
    - Provides contact info for workflow modification or help.

---

### 3. Summary Table

| Node Name                    | Node Type                             | Functional Role                                   | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                      |
|------------------------------|-------------------------------------|--------------------------------------------------|------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| When an email is received     | Email Read (IMAP)                   | Trigger on new incoming email                     | None                         | Parse the mail with AI        | ## Set receiving email account - Ddefaults to an IMAP account node, but you can put a gmail account or any email trigger |
| Sticky Note                  | Sticky Note                         | Instruction for email trigger setup               | None                         | None                         | ## Set receiving email account - Ddefaults to an IMAP account node, but you can put a gmail account or any email trigger |
| OpenAI Chat Model            | Langchain OpenAI Chat Model         | AI model interface for parsing email              | Parse the mail with AI        | Parse the mail with AI        | ## Upgrade the prompt! This is a very simple prompt but oit does the job. Improve it and send it to me! |
| Set the output Json          | Langchain Output Parser Structured  | Parses AI raw output into structured JSON         | OpenAI Chat Model             | Parse the mail with AI        | ## Upgrade the prompt! This is a very simple prompt but oit does the job. Improve it and send it to me! |
| Parse the mail with AI       | Langchain Chain LLM                 | Orchestrates AI prompt and output parsing         | When an email is received     | Search for the contact via email | ## Upgrade the prompt! This is a very simple prompt but oit does the job. Improve it and send it to me! |
| Search for the contact via email | HubSpot (search)                  | Searches HubSpot contacts by email                 | Parse the mail with AI        | contact exists?               | ## Hubspot integration - Search for the contact in hubspot - If it is present creates an egagement - It it is not, creates it and adds an engagement |
| contact exists?              | IF Node                            | Checks if contact exists in HubSpot                | Search for the contact via email | Edit Fields (true branch), Creates contact (false branch) | ## Hubspot integration - Search for the contact in hubspot - If it is present creates an egagement - It it is not, creates it and adds an engagement |
| Edit Fields                 | Set Node                           | Sets contact ID for engagement creation            | contact exists? (true branch) | Creates an email engagement   | ## Hubspot integration - Search for the contact in hubspot - If it is present creates an egagement - It it is not, creates it and adds an engagement |
| Creates contact             | HubSpot (create contact)           | Creates new contact in HubSpot                      | contact exists? (false branch) | Creates an email engagement   | ## Hubspot integration - Search for the contact in hubspot - If it is present creates an egagement - It it is not, creates it and adds an engagement |
| Creates an email engagement | HubSpot (create engagement)        | Logs email as engagement linked to contact         | Edit Fields, Creates contact  | None                         | ## Hubspot integration - Search for the contact in hubspot - If it is present creates an egagement - It it is not, creates it and adds an engagement |
| Sticky Note1                | Sticky Note                        | Suggestion to improve AI prompt                     | None                         | None                         | ## Upgrade the prompt! This is a very simple prompt but oit does the job. Improve it and send it to me! |
| Sticky Note2                | Sticky Note                        | Explains HubSpot integration logic                  | None                         | None                         | ## Hubspot integration - Search for the contact in hubspot - If it is present creates an egagement - It it is not, creates it and adds an engagement |
| Sticky Note7                | Sticky Note                        | Contact info for workflow help or modifications     | None                         | None                         | ## Contact me - If you need any modification to this workflow - if you need some help with this workflow - Or if you need any workflow in n8n, Make, or Langchain / Langgraph Write to me: [thomas@pollup.net](mailto:thomas@pollup.net) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add **Email Read (IMAP)** node named "When an email is received".  
   - Configure with your IMAP credentials (host, port, username, password).  
   - Set "Force Reconnect" option to 3 for stability.

2. **Add AI Parsing Nodes:**  
   - Add **Langchain OpenAI Chat Model** node named "OpenAI Chat Model".  
     - Set model to "gpt-4o-mini".  
     - Configure with your OpenAI API credentials.  
   - Add **Langchain Output Parser Structured** node named "Set the output Json".  
     - Provide a JSON schema example for expected contact info fields (first name, last name, email, phone, company, address, etc.).  
   - Add **Langchain Chain LLM** node named "Parse the mail with AI".  
     - Set prompt to extract key contact info from the email content, including HTML/plain text, from, subject, date.  
     - Connect "OpenAI Chat Model" as the language model and "Set the output Json" as the output parser.  
     - Connect input from "When an email is received".

3. **Add HubSpot Contact Search:**  
   - Add **HubSpot** node named "Search for the contact via email".  
     - Set operation to "search".  
     - Configure OAuth2 credentials for HubSpot API access.  
     - Add filter group: filter by property "email" equal to `{{$json.output.contact_info.email}}`.  
     - Connect input from "Parse the mail with AI".

4. **Add Conditional Check for Contact Existence:**  
   - Add **IF** node named "contact exists?".  
     - Condition: Check if `$json.id` exists (string exists).  
     - Connect input from "Search for the contact via email".

5. **Add Branch for Existing Contact:**  
   - Add **Set** node named "Edit Fields".  
     - Set field `vid` to `{{$json.id}}`.  
     - Connect input from "contact exists?" true branch.

6. **Add Branch for New Contact Creation:**  
   - Add **HubSpot** node named "Creates contact".  
     - Operation: Create contact.  
     - Map fields from parsed AI output: email, city, country, job title, last name, postal code, website URL, company name, phone number, street address.  
     - Use OAuth2 credentials.  
     - Connect input from "contact exists?" false branch.

7. **Add Engagement Creation Node:**  
   - Add **HubSpot** node named "Creates an email engagement".  
     - Operation: Create engagement of type "email".  
     - Metadata:  
       - html: `{{$node["When an email is received"].json["textHtml"] || $node["When an email is received"].json["textPlain"]}}`  
       - subject: `{{$node["When an email is received"].json["subject"]}}`  
       - toEmail: `{{$node["When an email is received"].json["to"]}}` (as array)  
       - fromEmail: `{{$node["When an email is received"].json["from"]}}`  
     - Associations: contactIds set to `{{$json.vid}}` (from either "Edit Fields" or "Creates contact")  
     - Use OAuth2 credentials.  
     - Connect inputs from both "Edit Fields" and "Creates contact".

8. **Add Sticky Notes:**  
   - Add notes for setup instructions, prompt improvement suggestions, HubSpot integration explanation, and contact info for help.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                               |
|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow designed for Customer Success, Sales, Support, and Marketing teams using HubSpot CRM to automate email engagement logging. | Workflow purpose and target users                              |
| Contact for help or modifications: [thomas@pollup.net](mailto:thomas@pollup.net)                                                  | Support and customization contact                             |
| Discover other workflows by the author: [https://n8n.io/creators/zeerobug/](https://n8n.io/creators/zeerobug/)                   | Author's public workflow collection                            |
| Suggestion to improve AI prompt for better parsing accuracy.                                                                       | Sticky note near AI parsing nodes                              |
| HubSpot integration logic: search contact, create engagement if found, else create contact and engagement.                         | Sticky note near HubSpot nodes                                 |
| Replace default IMAP node with Gmail or other email provider as needed.                                                            | Sticky note near email trigger node                            |

---

This structured documentation enables both advanced users and AI agents to fully understand, reproduce, and modify the workflow, as well as anticipate potential issues such as API limits, authentication failures, and parsing errors.