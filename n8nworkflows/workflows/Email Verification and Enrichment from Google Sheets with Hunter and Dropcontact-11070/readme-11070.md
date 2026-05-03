Email Verification and Enrichment from Google Sheets with Hunter and Dropcontact

https://n8nworkflows.xyz/workflows/email-verification-and-enrichment-from-google-sheets-with-hunter-and-dropcontact-11070


# Email Verification and Enrichment from Google Sheets with Hunter and Dropcontact

---

### 1. Workflow Overview

This workflow automates the lead capture, AI-assisted email drafting, manual approval, email sending, and CRM logging process for inbound inquiries submitted via an n8n form. It is designed for small sales teams or businesses that want to combine AI-generated personalized follow-up emails with human oversight before sending, while keeping detailed records in Airtable.

**Target use cases:**

- Automatically processing new leads from a form submission.
- Generating personalized follow-up emails using AI.
- Routing email drafts to sales for approval via email links.
- Sending approved emails automatically.
- Logging all leads and email statuses in Airtable for CRM tracking.

**Logical blocks:**

- **1.1 Lead Intake & Preparation:** Receive and normalize form data for consistent AI input.
- **1.2 AI Email Draft Generation:** Use LangChain with OpenAI GPT-4 to create a personalized email draft.
- **1.3 Human Approval Workflow:** Send the draft to sales for review and wait on their approval or rejection.
- **1.4 Final Actions:** Depending on approval, send the email to the lead and log the lead and email status into Airtable.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Intake & Preparation

**Overview:**  
This block captures lead data from the n8n form submission, applies normalization, and sets up configuration variables for company branding and CRM integration.

**Nodes Involved:**  
- On form submission  
- Workflow Configuration  
- Extract Lead Data  

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; triggers when a user submits the "Leads" form.  
  - Config: Form fields for Name, Email, Company, Phone, Message; all required.  
  - Input/Output: No input; outputs form data JSON.  
  - Edge Cases: Missing required fields prevented by form validation; no direct validation in node.  
  - Notes: Webhook ID used to receive submissions.

- **Workflow Configuration**  
  - Type: Set node  
  - Role: Holds static/company configuration variables such as Slack channel, Airtable base/table IDs, company contact info, and URLs.  
  - Config: Assigns placeholders for company and integration info as string variables.  
  - Input: Output from form submission node.  
  - Output: JSON enriched with configuration data.  
  - Edge Cases: Must be properly configured before workflow runs; placeholders must be replaced.

- **Extract Lead Data**  
  - Type: Set node  
  - Role: Normalize and extract lead data fields (Name, Email, Company, Phone, Message) from the form submission JSON for consistent downstream use.  
  - Config: Assigns leadName, leadEmail, leadCompany, leadPhone, leadMessage from JSON body or defaults.  
  - Input: Configuration-enriched form data.  
  - Output: JSON with normalized lead data.  
  - Edge Cases: Missing fields default to empty strings or "Unknown" for name.

---

#### 1.2 AI Email Draft Generation

**Overview:**  
Generates a personalized follow-up email draft using LangChain’s OpenAI GPT-4 model based on normalized lead data.

**Nodes Involved:**  
- Generate Email Draft  
- OpenAI Chat Model  

**Node Details:**

- **Generate Email Draft**  
  - Type: LangChain Agent node (AI prompt manager)  
  - Role: Constructs a prompt for a professional sales assistant to write a warm, personalized follow-up email.  
  - Config:  
    - Prompt uses lead details (Name, Email, Company, Phone, Message) via expressions like `{{ $json.Name }}`.  
    - Instructions specify tone, structure, and content elements (thank you, inquiry address, call to action).  
  - Input: Normalized lead data from Extract Lead Data node.  
  - Output: AI-generated email body text only (no subject).  
  - Edge Cases: API failures, rate limits, or malformed expressions may cause failure; ensure proper credentials and input completeness.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Language Model  
  - Role: Provides GPT-4.1-mini model access to Generate Email Draft node.  
  - Config: Model set to "gpt-4.1-mini".  
  - Credentials: OpenAI API key required.  
  - Input/Output: AI language model interface; no direct input/output in workflow, used internally by Generate Email Draft.  
  - Edge Cases: API quota exceeded, auth errors, or network issues.

---

#### 1.3 Human Approval Workflow

**Overview:**  
Sends the AI-generated email draft to the sales team for manual approval via email with approve/reject links, then waits for the team’s response.

**Nodes Involved:**  
- Send Approval Request Email  
- Wait for Approval  
- Check Approval Status  

**Node Details:**

- **Send Approval Request Email**  
  - Type: Gmail node  
  - Role: Sends an HTML email to the configured sales approver containing lead info and the AI email draft.  
  - Config:  
    - Recipient hardcoded (example: calvinc4@me.com).  
    - Email body includes lead details and AI draft with styled HTML.  
    - Two links for approval and rejection: each calls the Wait for Approval webhook with `action=approve` or `action=reject` query parameters.  
  - Credentials: Gmail OAuth2 required.  
  - Input: AI email draft and lead data.  
  - Output: None.  
  - Edge Cases: Gmail quota limits, authentication, or connectivity issues.

- **Wait for Approval**  
  - Type: Wait node with resume webhook  
  - Role: Pauses workflow execution until approval webhook is triggered via the email links.  
  - Config:  
    - Waits max 24 hours (or configured duration).  
    - Resumes via webhook with query param `action`.  
  - Input: None (starts wait after sending approval email).  
  - Output: Resumes with webhook data when approver acts.  
  - Edge Cases: Link not clicked, timeout expires — workflow continues with timeout or may require manual intervention.

- **Check Approval Status**  
  - Type: If node  
  - Role: Checks if approval action equals "approve".  
  - Config: Checks `$('Wait for Approval').item.json.query.action == 'approve'`.  
  - Input: Wait for Approval resume data.  
  - Output:  
    - True branch: approved path.  
    - False branch: rejected path.  
  - Edge Cases: Unexpected or missing query params; safe fallback to rejection branch recommended.

---

#### 1.4 Final Actions: Send Email & CRM Logging

**Overview:**  
Depending on the approval result, sends the finalized email to the lead and logs or updates lead records in Airtable accordingly.

**Nodes Involved:**  
- Send Email to Lead  
- Create Lead in Airtable  
- Update Lead Status (Rejected)  

**Node Details:**

- **Send Email to Lead**  
  - Type: Gmail node  
  - Role: Sends the approved AI-generated email to the lead’s email address.  
  - Config:  
    - Recipient: lead’s email from form submission.  
    - Message: Styled HTML email including AI-generated email body and company signature/branding from workflow configuration.  
    - Subject: Fixed "Thank you for your interest!"  
  - Credentials: Gmail OAuth2 required.  
  - Input: Approval check node true branch.  
  - Output: None.  
  - Edge Cases: Gmail limits, invalid email addresses, or network issues.

- **Create Lead in Airtable**  
  - Type: Airtable node  
  - Role: Creates a new lead record in Airtable with lead info, email draft, and status "Email Sent - Approved".  
  - Config:  
    - Base and Table IDs configured from Workflow Configuration.  
    - Columns mapped: Email, Notes, Phone, Status, First Name, Email Draft, Company Name.  
    - Operation: Create record.  
  - Credentials: Airtable OAuth2 required.  
  - Input: After successful email send.  
  - Edge Cases: Airtable API limits, misconfigured base/table IDs, or schema mismatches.

- **Update Lead Status (Rejected)**  
  - Type: Airtable node  
  - Role: Creates a lead record with status "Email Rejected - Needs Review" when the email draft is rejected.  
  - Config: Similar to Create Lead but status reflects rejection.  
  - Credentials: Airtable OAuth2 required.  
  - Input: Approval check node false branch.  
  - Edge Cases: Same as Create Lead node.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                        | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                                            |
|----------------------------|--------------------------------|--------------------------------------|----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                   | Entry point, receive lead data       | None                       | Workflow Configuration       |                                                                                                                        |
| Workflow Configuration      | Set                           | Set static company & integration vars| On form submission         | Extract Lead Data            |                                                                                                                        |
| Extract Lead Data           | Set                           | Normalize lead fields for AI         | Workflow Configuration     | Generate Email Draft         | 📝 Lead Intake & Preparation: normalize fields for consistency                                                        |
| Generate Email Draft        | LangChain Agent               | AI generates personalized email draft| Extract Lead Data, OpenAI  | Send Approval Request Email  | 🤖 Draft Generation & Human Approval: AI draft then human review                                                        |
| OpenAI Chat Model           | LangChain OpenAI Chat Model   | Provides GPT-4 model for AI node     | Used internally by Generate Email Draft | Generate Email Draft |                                                                                                                        |
| Send Approval Request Email | Gmail                         | Send AI draft to sales for approval  | Generate Email Draft       | Wait for Approval            |                                                                                                                        |
| Wait for Approval           | Wait (resume webhook)          | Pause workflow waiting for approval  | Send Approval Request Email | Check Approval Status        |                                                                                                                        |
| Check Approval Status       | If                            | Decide path based on approval status | Wait for Approval          | Send Email to Lead, Update Lead Status (Rejected) |                                                                                                                        |
| Send Email to Lead          | Gmail                         | Send approved email to lead          | Check Approval Status      | Create Lead in Airtable      | 📊 Log Lead in Airtable & Send Final Email: send & log approved leads                                                  |
| Create Lead in Airtable     | Airtable                      | Log approved lead & email status     | Send Email to Lead         | None                        |                                                                                                                        |
| Update Lead Status (Rejected) | Airtable                    | Log rejected lead & status           | Check Approval Status      | None                        |                                                                                                                        |
| Sticky Note                 | Sticky Note                   | Documentation notes                   | None                       | None                        | 📌 Purpose: Automate AI-assisted lead response with human approval and CRM logging                                     |
| Sticky Note1                | Sticky Note                   | Setup instructions                   | None                       | None                        | 🔧 Setup Instructions: credentials, Airtable setup, company details, approval email, webhook, form publishing           |
| Sticky Note2                | Sticky Note                   | Troubleshooting                      | None                       | None                        | 🛠️ Troubleshooting: approval link, email sending, Airtable updates, draft fields, waiting for approval                 |
| Sticky Note3                | Sticky Note                   | Lead Intake & Preparation explanation| None                      | None                        | 📝 Lead Intake & Preparation: normalize form input for AI consistency                                                 |
| Sticky Note4                | Sticky Note                   | Draft generation & human approval    | None                       | None                        | 🤖 Draft Generation & Human Approval: AI writes draft, sales review                                                   |
| Sticky Note5                | Sticky Note                   | Logging & sending final email        | None                       | None                        | 📊 Log Lead in Airtable & Send Final Email: clean CRM records, saved drafts, status tracking                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form fields: Name (required), Email (email, required), Company (required), Phone (required), Message (required)  
   - Set form description as desired.  
   - Save and note the webhook URL.

2. **Create Set node ("Workflow Configuration")**  
   - Type: Set  
   - Add string variables for:  
     - slackChannel (Slack channel ID)  
     - airtableBaseId (Airtable base ID)  
     - airtableTableId (Airtable table ID)  
     - companyName, senderName, senderTitle, companyEmail, companyPhone, companyAddress  
     - linkedinUrl, twitterUrl, facebookUrl, websiteUrl  
   - Use placeholders initially; replace with actual values before running.

3. **Connect "On form submission" → "Workflow Configuration".**

4. **Create Set node ("Extract Lead Data")**  
   - Type: Set  
   - Assign normalized fields:  
     - leadName = `{{$json.body.name || 'Unknown'}}`  
     - leadEmail = `{{$json.body.email || ''}}`  
     - leadCompany = `{{$json.body.company || ''}}`  
     - leadPhone = `{{$json.body.phone || ''}}`  
     - leadMessage = `{{$json.body.message || ''}}`  
   - Include all other incoming fields.

5. **Connect "Workflow Configuration" → "Extract Lead Data".**

6. **Create LangChain AI Agent node ("Generate Email Draft")**  
   - Type: LangChain Agent  
   - Configure prompt with instructions to generate a professional sales email using lead details from JSON fields (`{{ $json.Name }}`, etc.)  
   - Set to return only the email body text.  
   - Set LangChain model to OpenAI GPT-4.1-mini.

7. **Create LangChain OpenAI Chat Model node ("OpenAI Chat Model")**  
   - Type: LangChain OpenAI Chat Model  
   - Set model to "gpt-4.1-mini".  
   - Connect credentials for OpenAI API.

8. **Link "Extract Lead Data" → "Generate Email Draft".**  
   - Link "OpenAI Chat Model" as AI language model for "Generate Email Draft".

9. **Create Gmail node ("Send Approval Request Email")**  
   - Type: Gmail  
   - Recipient: sales team email (e.g., calvinc4@me.com)  
   - Subject: "New Lead Email Draft - Approval Required"  
   - Message: HTML email with lead details and AI email draft embedded, plus two styled links to approve or reject using the Wait node webhook URL with query parameters `action=approve` and `action=reject`.  
   - Use credentials for Gmail OAuth2.

10. **Connect "Generate Email Draft" → "Send Approval Request Email".**

11. **Create Wait node ("Wait for Approval")**  
    - Type: Wait node with webhook resume  
    - Configure webhook to resume workflow.  
    - Set max wait time to 24 hours or desired duration.

12. **Connect "Send Approval Request Email" → "Wait for Approval".**

13. **Create If node ("Check Approval Status")**  
    - Type: If  
    - Condition: Check if `$('Wait for Approval').item.json.query.action` equals "approve".

14. **Connect "Wait for Approval" → "Check Approval Status".**

15. **Create Gmail node ("Send Email to Lead")**  
    - Type: Gmail  
    - Recipient: Lead email from form submission JSON.  
    - Subject: "Thank you for your interest!"  
    - Message: Styled HTML email embedding the AI-generated email body and company branding from Workflow Configuration node.  
    - Use Gmail OAuth2 credentials.

16. **Connect If node true output (approved) → "Send Email to Lead".**

17. **Create Airtable node ("Create Lead in Airtable")**  
    - Type: Airtable  
    - Operation: Create record  
    - Configure base and table using Workflow Configuration variables.  
    - Map fields: Email, Notes, Phone, Status ("Email Sent - Approved"), First Name, Email Draft, Company Name.  
    - Use Airtable OAuth2 credentials.

18. **Connect "Send Email to Lead" → "Create Lead in Airtable".**

19. **Create Airtable node ("Update Lead Status (Rejected)")**  
    - Type: Airtable  
    - Operation: Create record (or update as per use case)  
    - Configure base and table as above.  
    - Map fields similarly but set Status to "Email Rejected - Needs Review".  
    - Use Airtable OAuth2 credentials.

20. **Connect If node false output (rejected) → "Update Lead Status (Rejected)".**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow blends AI automation with human approval to ensure quality and personalization in lead follow-up emails.                                                                                                          | Workflow purpose                                                                                             |
| Setup requires valid credentials for Gmail (OAuth2), Airtable (OAuth2 or Personal Access Token), and OpenAI API keys.                                                                                                           | Setup instructions                                                                                           |
| Approval links rely on the Wait node’s webhook resume URL; ensure webhooks use production URLs to avoid broken links.                                                                                                            | Troubleshooting notes                                                                                        |
| Airtable table must include fields: Name, Email, Phone, Company Name, Message, Status, Email Draft, Created On for correct mapping.                                                                                              | Setup instructions                                                                                           |
| The AI prompt in the LangChain agent is customizable to adjust tone, content, or language of the email drafts.                                                                                                                  | Customization opportunity                                                                                     |
| If approval links are not clicked within the Wait node timeout, the workflow will stop waiting; consider alerting or retry mechanisms as needed.                                                                                | Troubleshooting notes                                                                                        |
| Use the sticky notes embedded in the workflow for quick reference on purpose, setup, troubleshooting, and block explanations.                                                                                                   | Embedded workflow documentation nodes                                                                        |
| For best results, ensure the form field names match exactly the expressions used in the AI prompt and set nodes to prevent missing data.                                                                                       | Troubleshooting notes                                                                                        |
| The workflow can be adapted to other CRMs or email providers by replacing Airtable and Gmail nodes with respective integrations.                                                                                                | Extension possibilities                                                                                       |
| The Gmail nodes use OAuth2 credentials, which require prior setup of API access with Google and user consent for sending emails.                                                                                                | Credential setup                                                                                              |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---