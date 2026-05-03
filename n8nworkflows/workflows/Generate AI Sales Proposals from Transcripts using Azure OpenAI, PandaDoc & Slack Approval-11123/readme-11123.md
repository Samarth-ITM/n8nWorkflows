Generate AI Sales Proposals from Transcripts using Azure OpenAI, PandaDoc & Slack Approval

https://n8nworkflows.xyz/workflows/generate-ai-sales-proposals-from-transcripts-using-azure-openai--pandadoc---slack-approval-11123


# Generate AI Sales Proposals from Transcripts using Azure OpenAI, PandaDoc & Slack Approval

---

### 1. Workflow Overview

This workflow automates the generation, approval, and sending of AI-generated sales proposals from client intake forms and sales call transcripts. It is designed for B2B service providers, agencies, and consultants to streamline proposal creation, approval, and follow-up, improving sales efficiency and reducing manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Data Gathering:** Collects client data via a submission form and retrieves associated sales call transcripts from Fireflies.ai.
- **1.2 AI Proposal Generation:** Uses Azure OpenAI through LangChain to generate a structured sales proposal draft based on client data and call summary.
- **1.3 Proposal Document Creation:** Creates a proposal document in PandaDoc by populating a predefined template with AI-generated content and client details.
- **1.4 Human Approval Loop via Slack:** Sends the proposal link to a Slack user for approval. If disapproved, collects feedback, updates variables, and regenerates the proposal until approved.
- **1.5 Sending Proposal and CRM Update:** Once approved, sends the proposal document via PandaDoc and updates the client’s stage in Airtable CRM.
- **1.6 Follow-Up System and Audit Trail:** Monitors the proposal status using PandaDoc audit trail. If unsigned after 48 hours, sends a reminder to the client and updates the CRM stage. Upon signing, updates CRM to “Document Signed”.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Gathering

**Overview:**  
Receives client proposal input via a web form and retrieves the latest sales call transcript for context.

**Nodes Involved:**  
- On form submission  
- Get a list of transcripts  
- Get a transcript  
- Set Initial Variables  

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; collects detailed client info including name, email, company, industry, budget, scope, problem, solution, timeline, and deliverables.  
  - Config: Custom form titled "Client Proposal" with multiple required fields.  
  - Connections: Outputs to "Get a list of transcripts".  
  - Edge cases: Missing or invalid form data; webhook failures.

- **Get a list of transcripts**  
  - Type: Fireflies API node  
  - Role: Fetches a list of transcripts matching client email.  
  - Config: Filters transcripts by participant email from form input.  
  - Connections: Passes transcript list to "Get a transcript".  
  - Edge cases: No transcripts found; API rate limits; auth errors.

- **Get a transcript**  
  - Type: Fireflies API node  
  - Role: Retrieves full transcript details for the first transcript in the list.  
  - Config: Uses transcript ID from previous node output.  
  - Connections: Outputs to "Set Initial Variables".  
  - Edge cases: Invalid transcript ID; API failures.

- **Set Initial Variables**  
  - Type: Set node  
  - Role: Initializes variables `draftText` and `lastFeedback` as empty strings to store proposal draft and feedback for iterative improvements.  
  - Connections: Passes forward to "Basic LLM Chain".  
  - Edge cases: Variable overwrite or missing initialization.

---

#### 1.2 AI Proposal Generation

**Overview:**  
Generates a structured proposal draft using Azure OpenAI via LangChain, incorporating client form data and sales call summary.

**Nodes Involved:**  
- Basic LLM Chain  
- Azure OpenAI Chat Model  
- Structured Output Parser  

**Node Details:**  

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain node  
  - Role: Defines prompt and manages AI call to generate structured proposal content. Accepts prior draft and feedback for iterative refinement.  
  - Config: Dynamic prompt includes sales call summary and all client form inputs. Requests breakdown of scope, deliverables, and timeline into multiple structured parts. Output expected to be parsed JSON.  
  - Connections: Input from "Set Initial Variables", output to "Create Document".  
  - Expressions: Uses multiple expressions to pull data from form and transcript nodes.  
  - Edge cases: Prompt formatting errors, AI model timeouts, incomplete response, or parsing failures.

- **Azure OpenAI Chat Model**  
  - Type: LangChain Azure OpenAI model node  
  - Role: Provides the AI language model engine for the LLM Chain.  
  - Config: Uses Azure OpenAI credentials; model set to "Open AI".  
  - Connections: Connected as AI model input to "Basic LLM Chain".  
  - Edge cases: Authentication failure; API rate limits; network errors.

- **Structured Output Parser**  
  - Type: LangChain structured output parser  
  - Role: Parses AI response into structured JSON fields such as introduction, problem, solution, scope, deliverables, timeline, investment, next steps.  
  - Config: Uses JSON schema example for expected output structure.  
  - Connections: Connected as output parser in "Basic LLM Chain".  
  - Edge cases: Output not matching schema; parsing errors.

---

#### 1.3 Proposal Document Creation

**Overview:**  
Creates a PandaDoc document from the AI-generated proposal content and client details, using a predefined PandaDoc template.

**Nodes Involved:**  
- Create Document  

**Node Details:**  

- **Create Document**  
  - Type: HTTP Request node  
  - Role: Sends POST request to PandaDoc API to create a new document.  
  - Config:  
    - URL: PandaDoc documents endpoint.  
    - Body: JSON object with document name, template UUID, recipient info (client email, name), and tokens populated by both form data and AI-generated content fields (introduction, problem, solution, scope, deliverables, timeline, investment, next steps).  
    - Pricing table included with budget from form input.  
    - Authorization header with API Key required (placeholder in workflow).  
  - Connections: Output to "Send a message and wait for response" (Slack approval step).  
  - Edge cases: API key missing/invalid, template ID mismatch, invalid token substitution, request timeouts.

---

#### 1.4 Human Approval Loop via Slack

**Overview:**  
Sends proposal link to a Slack user for approval. If disapproved, collects feedback, updates variables, regenerates proposal draft, and loops until approval.

**Nodes Involved:**  
- Send a message and wait for response (Slack)  
- If (approval check)  
- Update Variables  
- Basic LLM Chain (loop back)  

**Node Details:**  

- **Send a message and wait for response**  
  - Type: Slack node (send and wait)  
  - Role: Sends Slack message to a specified user with proposal link and collects approval or disapproval with feedback.  
  - Config:  
    - User: specific Slack user ID (e.g., "sparsh").  
    - Message includes PandaDoc document link.  
    - Custom form with radio buttons for "Approve" or "Disapprove" and a textarea for feedback if disapproved.  
  - Connections: Output to "If" node.  
  - Edge cases: Slack API rate limits; user unresponsive; webhook failures.

- **If**  
  - Type: If node  
  - Role: Evaluates if the Slack response is "Approve" or "Disapprove".  
  - Config: Checks if "Do you want to approve?" equals "Approve".  
  - Connections:  
    - If approved: proceeds to "Wait" (for sending document).  
    - If disapproved: proceeds to "Update Variables" for feedback incorporation.  
  - Edge cases: Unexpected or missing Slack responses.

- **Update Variables**  
  - Type: Set node  
  - Role: Updates `draftText` with latest AI proposal output and `lastFeedback` with disapproval feedback to influence next AI generation.  
  - Connections: Loops back to "Basic LLM Chain" for proposal regeneration.  
  - Edge cases: Data overwrite; missing feedback.

---

#### 1.5 Sending Proposal and CRM Update

**Overview:**  
After approval, sends the proposal via PandaDoc, notifies Slack, and updates the client’s proposal stage in Airtable CRM.

**Nodes Involved:**  
- Wait  
- Send Document  
- Send a message1 (Slack notification)  
- Update Contact (Airtable)  

**Node Details:**  

- **Wait**  
  - Type: Wait node  
  - Role: Delays workflow for 10 seconds before sending document (likely to ensure PandaDoc document is ready).  
  - Config: Fixed 10-second wait.  
  - Connections: Output to "Send Document".  
  - Edge cases: Delay too short or long; workflow timeout.

- **Send Document**  
  - Type: HTTP Request  
  - Role: Sends POST request to PandaDoc API to send the created document to the client.  
  - Config: Uses document ID from "Create Document" node, sends with Authorization header.  
  - Connections: Outputs to "Send a message1" (Slack notification).  
  - Edge cases: API call failure; invalid document ID.

- **Send a message1**  
  - Type: Slack node (send message)  
  - Role: Notifies Slack user that the proposal has been sent, including client and company name.  
  - Config: Sends a simple text message to specified Slack user.  
  - Connections: Outputs to "Update Contact".  
  - Edge cases: Slack API issues.

- **Update Contact**  
  - Type: Airtable node (upsert)  
  - Role: Updates client record in Airtable CRM to stage "Proposal Sent" based on client email.  
  - Config: Airtable base and table specified; matches on Email column; updates Stage field.  
  - Connections: Outputs to "Wait1" (follow-up block).  
  - Edge cases: Airtable API errors, no matching record, permission issues.

---

#### 1.6 Follow-Up System and Audit Trail

**Overview:**  
Monitors proposal signing status through PandaDoc audit trail. If unsigned after 48 hours, sends reminder and updates CRM. If signed, updates CRM to "Document Signed".

**Nodes Involved:**  
- Wait1  
- Get Audit Trail  
- If1 (check audit trail count)  
- Get Recepient ID  
- Send Reminder1  
- Send a message (Slack reminder notification)  
- Update Contact1 (CRM stage "Reminder Sent")  
- Update Contact2 (CRM stage "Document Signed")  

**Node Details:**  

- **Wait1**  
  - Type: Wait node  
  - Role: Waits 10 seconds before checking audit trail (likely triggered by upstream logic, could be adjusted for timing).  
  - Connections: Outputs to "Get Audit Trail".  
  - Edge cases: Timing sensitivity.

- **Get Audit Trail**  
  - Type: HTTP Request  
  - Role: Retrieves PandaDoc audit trail for the created document to determine signing events.  
  - Config: Uses document ID; Authorization header.  
  - Connections: Outputs to "If1".  
  - Edge cases: API failures; incorrect document ID.

- **If1**  
  - Type: If node  
  - Role: Checks if the audit trail count is less than 9 (threshold for determining unsigned or signed status).  
  - Connections:  
    - True (less than 9): proceeds to "Get Recipient ID" to send reminder.  
    - False: proceeds to "Update Contact2" to mark document signed.  
  - Edge cases: Threshold logic accuracy.

- **Get Recepient ID**  
  - Type: HTTP Request  
  - Role: Fetches recipient ID from PandaDoc document details, needed to send reminders.  
  - Config: Uses document ID; Authorization header.  
  - Connections: Outputs to "Send Reminder1".  
  - Edge cases: API errors; recipient not found.

- **Send Reminder1**  
  - Type: HTTP Request  
  - Role: Sends a reminder to the client via PandaDoc.  
  - Config: Sends JSON with recipient ID and delivery methods (email only), Authorization header.  
  - Connections: Outputs to "Send a message".  
  - Edge cases: API failures.

- **Send a message**  
  - Type: Slack node (send message)  
  - Role: Notifies Slack user that a reminder has been sent due to unsigned document after 48 hours.  
  - Config: Message includes PandaDoc document link.  
  - Connections: Outputs to "Update Contact1".  
  - Edge cases: Slack API failures.

- **Update Contact1**  
  - Type: Airtable node (upsert)  
  - Role: Updates client stage in CRM to "Reminder Sent".  
  - Config: Matches on Email; updates Stage.  
  - Connections: End of reminder flow.  
  - Edge cases: Airtable API errors.

- **Update Contact2**  
  - Type: Airtable node (upsert)  
  - Role: Updates client stage in CRM to "Document Signed" upon successful signing detection.  
  - Config: Matches on Email; updates Stage.  
  - Connections: End of signed flow.  
  - Edge cases: Airtable API errors.

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                                      | Input Node(s)                 | Output Node(s)                       | Sticky Note                                             |
|-----------------------------|----------------------------------|-----------------------------------------------------|------------------------------|------------------------------------|---------------------------------------------------------|
| On form submission           | Form Trigger                     | Collect client proposal data                         |                              | Get a list of transcripts           | ## Form Trigger                                         |
| Get a list of transcripts    | Fireflies API                    | Retrieve transcripts list for client email          | On form submission            | Get a transcript                    | ## Sales Call Intelligence                              |
| Get a transcript             | Fireflies API                    | Get detailed sales call transcript                   | Get a list of transcripts     | Set Initial Variables               |                                                         |
| Set Initial Variables        | Set                             | Initialize draftText and lastFeedback variables     | Get a transcript             | Basic LLM Chain                    |                                                         |
| Basic LLM Chain              | LangChain LLM Chain             | Generate structured AI proposal draft                | Set Initial Variables         | Create Document                    | ## AI Content Generator for Proposal                    |
| Azure OpenAI Chat Model      | LangChain Azure OpenAI Model    | AI language model for proposal generation            |                              | Basic LLM Chain (model input)      |                                                         |
| Structured Output Parser     | LangChain Output Parser         | Parse AI response into structured JSON               |                              | Basic LLM Chain (output parser)    |                                                         |
| Create Document             | HTTP Request (PandaDoc API)      | Create proposal document in PandaDoc                  | Basic LLM Chain              | Send a message and wait for response | ## Proposal Creation using PandaDoc                     |
| Send a message and wait for response | Slack Send & Wait            | Send proposal approval request and collect feedback  | Create Document              | If                                | ## Human Approval using Slack                           |
| If                          | If                              | Branch based on approval/disapproval                  | Send a message and wait for response | Wait / Update Variables          |                                                         |
| Update Variables            | Set                             | Update draftText and lastFeedback on disapproval     | If (Disapprove)              | Basic LLM Chain                    |                                                         |
| Wait                        | Wait                            | Delay before sending document                          | If (Approve)                 | Send Document                     |                                                         |
| Send Document               | HTTP Request (PandaDoc API)      | Send proposal document to client                       | Wait                        | Send a message1                   | ## Sending Proposal using PandaDoc                      |
| Send a message1             | Slack Send Message               | Notify Slack user proposal has been sent              | Send Document               | Update Contact                   |                                                         |
| Update Contact              | Airtable Upsert                 | Update CRM stage to "Proposal Sent"                   | Send a message1             | Wait1                           | ## Update Contact Stage in CRM                          |
| Wait1                       | Wait                            | Delay before audit trail check                         | Update Contact              | Get Audit Trail                 |                                                         |
| Get Audit Trail             | HTTP Request (PandaDoc API)      | Retrieve document audit trail                          | Wait1                       | If1                            |                                                         |
| If1                         | If                              | Check audit trail count to determine signing status  | Get Audit Trail             | Get Recepient ID / Update Contact2 |                                                         |
| Get Recepient ID            | HTTP Request (PandaDoc API)      | Get recipient ID to send reminder                      | If1 (true branch)           | Send Reminder1                 |                                                         |
| Send Reminder1              | HTTP Request (PandaDoc API)      | Send reminder to client for unsigned document         | Get Recepient ID            | Send a message                 |                                                         |
| Send a message              | Slack Send Message               | Notify Slack user reminder sent                        | Send Reminder1              | Update Contact1               |                                                         |
| Update Contact1             | Airtable Upsert                 | Update CRM stage to "Reminder Sent"                    | Send a message              |                                |                                                         |
| Update Contact2             | Airtable Upsert                 | Update CRM stage to "Document Signed"                  | If1 (false branch)          |                                |                                                         |
| Sticky Note                 | Sticky Note                    | Workflow annotations and explanations                  |                              |                                    | Multiple sticky notes as per node groupings             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form titled "Client Proposal" with fields: Client First Name, Client Last Name, Client Email (email type), Client Company Name, Client Industry (textarea), Client Website, Scope (textarea), Budget (number), Problem (textarea), Solution (textarea), Timeline, Deliverables (textarea).  
   - Set webhook ID (auto-generated).  

2. **Add Fireflies node ("Get a list of transcripts")**  
   - Operation: getTranscriptsList  
   - Filter: participantEmail = `{{$json["Client Email"]}}` (expression from form submission)  
   - Configure Fireflies API credentials.  
   - Connect output from form trigger node.

3. **Add Fireflies node ("Get a transcript")**  
   - Operation: getTranscript  
   - TranscriptId: `{{$json.data.id}}` (first transcript from list)  
   - Connect output from "Get a list of transcripts".

4. **Add Set node ("Set Initial Variables")**  
   - Assign variables: draftText = `""` (empty string), lastFeedback = `""` (empty string).  
   - Connect output from "Get a transcript".

5. **Add LangChain Azure OpenAI Chat Model node ("Azure OpenAI Chat Model")**  
   - Configure to use Azure OpenAI credentials.  
   - Model: Open AI.

6. **Add LangChain Structured Output Parser node ("Structured Output Parser")**  
   - Provide JSON schema example for expected proposal output (introduction, client_problem, proposed_solution, scope_of_work array, deliverables array, timeline_breakdown array, investment, next_steps).

7. **Add LangChain Chain LLM node ("Basic LLM Chain")**  
   - Configure prompt with dynamic expressions including sales call summary, all form fields, previous draftText, and lastFeedback.  
   - Set AI model to "Azure OpenAI Chat Model".  
   - Set output parser to "Structured Output Parser".  
   - Connect input from "Set Initial Variables".  

8. **Add HTTP Request node ("Create Document")**  
   - Method: POST  
   - URL: `https://api.pandadoc.com/public/v1/documents`  
   - Headers: Authorization: `API-Key <Your API Key>`, Accept: application/json  
   - Body: JSON with document name (using client first and last name), template UUID (your PandaDoc template), recipients (client email and name), tokens with form data and AI output fields, pricing table with budget.  
   - Connect output from "Basic LLM Chain".

9. **Add Slack node ("Send a message and wait for response")**  
   - Operation: sendAndWait  
   - User: select Slack user (e.g., sales approver)  
   - Message: includes PandaDoc document link with document ID from "Create Document"  
   - Form fields: radio buttons for "Do you want to approve?" (Approve/Disapprove), textarea for feedback if disapproved  
   - Connect output from "Create Document".

10. **Add If node ("If")**  
    - Condition: check if Slack response "Do you want to approve?" equals "Approve"  
    - Connect output from Slack node.

11. **Add Set node ("Update Variables")**  
    - Update variables: draftText = AI proposal output from "Basic LLM Chain", lastFeedback = Slack feedback text  
    - Connect from If node (Disapprove branch).

12. **Connect "Update Variables" back to "Basic LLM Chain"** to loop for proposal regeneration.

13. **Add Wait node ("Wait")**  
    - Wait 10 seconds  
    - Connect from If node (Approve branch).

14. **Add HTTP Request node ("Send Document")**  
    - Method: POST  
    - URL: `https://api.pandadoc.com/public/v1/documents/{{documentId}}/send`  
    - Headers: Authorization and Accept  
    - Connect output from "Wait".

15. **Add Slack node ("Send a message1")**  
    - Send notification to Slack user that proposal has been sent, including client and company name.  
    - Connect output from "Send Document".

16. **Add Airtable node ("Update Contact")**  
    - Operation: upsert  
    - Base and Table: your Airtable CRM base and table  
    - Match on Email from form submission  
    - Update field "Stage" to "Proposal Sent"  
    - Connect output from "Send a message1".

17. **Add Wait node ("Wait1")**  
    - Wait 10 seconds (or longer for real use)  
    - Connect output from "Update Contact".

18. **Add HTTP Request node ("Get Audit Trail")**  
    - Method: GET or POST to PandaDoc audit trail endpoint with document ID  
    - Headers with Authorization  
    - Connect output from "Wait1".

19. **Add If node ("If1")**  
    - Condition: check if audit trail count < 9 (unsigned).  
    - Connect output from "Get Audit Trail".

20. **Add HTTP Request node ("Get Recepient ID")**  
    - Get recipient ID from PandaDoc document details for reminder  
    - Connect If1 true branch.

21. **Add HTTP Request node ("Send Reminder1")**  
    - Send reminder to client using recipient ID  
    - Connect from "Get Recepient ID".

22. **Add Slack node ("Send a message")**  
    - Notify Slack user reminder sent including PandaDoc link  
    - Connect from "Send Reminder1".

23. **Add Airtable node ("Update Contact1")**  
    - Update CRM Stage to "Reminder Sent"  
    - Connect from "Send a message".

24. **Add Airtable node ("Update Contact2")**  
    - Update CRM Stage to "Document Signed"  
    - Connect If1 false branch.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                   |
|----------------------------------------------------------------------------------------------|--------------------------------------------------|
| AI Proposal Workflow Overview by Sparsh from Automation Jinn www.automationjinn.com          | Summary note describing workflow purpose & steps |
| Workflow designed for fast, structured, reusable proposal generation and approval process    | Ideal for Agencies, Consultants, B2B SaaS teams  |
| Core integrations: Fireflies.ai (transcripts), Azure OpenAI (AI generation), PandaDoc (docs), Slack (approval), Airtable (CRM) | Modular integration; components can be swapped   |
| Slack approval loop ensures human in the loop for quality control                            | Enables iterative draft improvements              |
| PandaDoc audit trail monitoring automates follow-up reminders and CRM stage updates          | Reduces manual sales follow-up                     |
| API Keys placeholders must be replaced with valid keys in HTTP Request nodes                 | Critical for PandaDoc and Azure OpenAI API calls  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.

---