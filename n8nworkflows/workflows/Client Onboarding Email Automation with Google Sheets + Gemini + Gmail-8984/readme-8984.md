Client Onboarding Email Automation with Google Sheets + Gemini + Gmail

https://n8nworkflows.xyz/workflows/client-onboarding-email-automation-with-google-sheets---gemini---gmail-8984


# Client Onboarding Email Automation with Google Sheets + Gemini + Gmail

### 1. Workflow Overview

This workflow automates client onboarding email communication by integrating Google Sheets form submissions, Google Gemini AI for personalized email generation, and Gmail for message delivery. It is designed for service providers who collect client data via Google Forms/Sheets and want to send customized onboarding emails automatically.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Triggered by new rows added to a Google Sheet (client onboarding form submissions).  
- **1.2 Data Extraction & Preparation:** Extracts and structures client details from the new form entry for later use.  
- **1.3 AI-Driven Personalization:** Uses a preset onboarding checklist and client data to generate a personalized onboarding email body via Google Gemini AI.  
- **1.4 Email Delivery & Execution Monitoring:** Sends the generated email via Gmail and handles execution success or failure states with error handling.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new client onboarding form submissions in a specified Google Sheet, triggering the workflow each time a new row is added.

**Nodes Involved:**  
- Trigger on New Client Form Submission  
- Sticky Note (STEP 1 · Trigger & Intake)

**Node Details:**  

- **Trigger on New Client Form Submission**  
  - Type: Google Sheets Trigger  
  - Role: Starts the workflow when a new row is added to the specified sheet ("Form Responses 1") within the "Onboarding" Google Sheets document.  
  - Configuration: Polls the sheet every minute for new rows. Uses OAuth2 credentials for Google Sheets access.  
  - Key expressions: Accesses form submission columns with exact keys (noting spaces in column headers such as `" email "` and `"  Company Name  "`).  
  - Inputs: None (trigger node)  
  - Outputs: Passes newly submitted row data downstream for processing.  
  - Edge Cases:  
    - OAuth token expiration or permission errors.  
    - Missing or renamed sheet or document.  
    - Data format inconsistencies due to column spaces.  
  - Sticky Note: Explains importance of exact column key matching.

- **Sticky Note (STEP 1 · Trigger & Intake)**  
  - Provides clarifications on the trigger configuration and column key sensitivity.

---

#### 1.2 Data Extraction & Preparation

**Overview:**  
This block consolidates client form fields into a structured string, preparing the data needed to personalize the onboarding email.

**Nodes Involved:**  
- Extract and Structure Client Data  
- Client Checklist (Set)  
- Sticky Note1 (STEP 2 · Checklist Context)

**Node Details:**  

- **Extract and Structure Client Data**  
  - Type: Set  
  - Role: Maps multiple client data fields from the Google Sheet submission into a single formatted string named `fields`.  
  - Configuration: Uses expressions to access specific columns from the trigger node's JSON, including client name, email, company, services needed, and other notes.  
  - Input: Data from "Trigger on New Client Form Submission"  
  - Output: JSON with a `fields` property containing the structured client info string.  
  - Edge Cases:  
    - Missing or blank form fields.  
    - Column name variations due to spaces or typos.  
  - Sticky Note: None directly attached but related to STEP 1 note.

- **Client Checklist**  
  - Type: Set  
  - Role: Defines a static onboarding checklist as a string to guide email personalization.  
  - Configuration: Hard-coded checklist steps numbered 1 to 6 describing onboarding stages.  
  - Input: Data from "Extract and Structure Client Data"  
  - Output: JSON with `Checklist` string property.  
  - Edge Cases: None critical; checklist must be manually updated to match actual onboarding process requirements.  
  - Sticky Note1: Explains this checklist is editable and used to enrich the AI prompt.

- **Sticky Note1 (STEP 2 · Checklist Context)**  
  - Explains the checklist role and advises on editing it to fit the specific service.

---

#### 1.3 AI-Driven Personalization

**Overview:**  
This block generates a customized onboarding email body using Google Gemini AI, combining the checklist and client data.

**Nodes Involved:**  
- Google Gemini Chat Model (AI language model node)  
- Personalize Using Gemini (Chain LLM node)  
- Sticky Note2 (STEP 3 · Personalize with Gemini)

**Node Details:**  

- **Google Gemini Chat Model**  
  - Type: Google Gemini Chat Model (Langchain LLM node)  
  - Role: Provides the AI model interface for generating text.  
  - Configuration: Uses the "models/gemini-2.0-flash" model with Google PaLM API credentials.  
  - Input: Connected as AI language model source for the downstream chain node.  
  - Output: Supplies AI-generated text to "Personalize Using Gemini".  
  - Edge Cases:  
    - API quota limits or authentication errors.  
    - Model response delays or failures.  
    - Prompt misinterpretation causing irrelevant outputs.

- **Personalize Using Gemini**  
  - Type: Chain LLM (Langchain chain node)  
  - Role: Constructs the prompt and invokes the AI to generate the email body specifically tailored with client name, company, and checklist context.  
  - Configuration:  
    - Prompt is carefully defined to produce only the email body (no extra wrapper text).  
    - Starts greeting with client name and ends with company team sign-off.  
    - Uses expressions to inject values from the trigger and checklist nodes.  
  - Input: From "Client Checklist" and AI model node.  
  - Output: JSON containing the generated email text in `text` property.  
  - Edge Cases:  
    - Expression failures if referenced nodes have no data.  
    - AI generating unexpected formatting or content.  
  - Sticky Note2: Emphasizes prompt design to limit output strictly to email body.

- **Sticky Note2 (STEP 3 · Personalize with Gemini)**  
  - Highlights the AI model use and prompt constraints to ensure clean email output.

---

#### 1.4 Email Delivery & Execution Monitoring

**Overview:**  
Sends the personalized email to the client and manages workflow completion or error events.

**Nodes Involved:**  
- Send Email to Client (Gmail node)  
- Execution Completed (NoOp)  
- Execution Failure (NoOp)  
- Error Handler (Error Trigger)  
- Sticky Note3 (STEP 4 · Send & Run State)

**Node Details:**  

- **Send Email to Client**  
  - Type: Gmail  
  - Role: Sends the onboarding email to the client’s email address collected from the form.  
  - Configuration:  
    - `sendTo`: Uses client email from trigger node.  
    - `subject`: Personalized welcome with client name.  
    - `message`: Uses AI-generated email body from the previous node.  
    - Uses OAuth2 Gmail credentials.  
    - Supports HTML if needed (manual adjustment required if email body includes HTML).  
  - Input: AI-generated text from "Personalize Using Gemini"  
  - Output: Triggers either "Execution Completed" or error flow.  
  - Edge Cases:  
    - Gmail API authentication or quota errors.  
    - Invalid email addresses.  
    - Message formatting issues.

- **Execution Completed**  
  - Type: NoOp  
  - Role: Marks successful workflow completion with no side effects.  
  - Input: From "Send Email to Client" on success.  
  - Output: None.  
  - Edge Cases: None.

- **Execution Failure**  
  - Type: NoOp  
  - Role: Marks failure state when error handling triggers.  
  - Input: From "Error Handler" on error.  
  - Output: None.

- **Error Handler**  
  - Type: Error Trigger  
  - Role: Catches any unhandled workflow errors and routes to "Execution Failure".  
  - Input: Workflow errors.  
  - Output: "Execution Failure" node.  
  - Edge Cases: None.

- **Sticky Note3 (STEP 4 · Send & Run State)**  
  - Describes email sending, error, and completion nodes usage and advice on HTML mode for Gmail if needed.

---

### 3. Summary Table

| Node Name                     | Node Type                       | Functional Role                        | Input Node(s)                      | Output Node(s)                       | Sticky Note                                                                                                  |
|-------------------------------|--------------------------------|-------------------------------------|----------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Trigger on New Client Form Submission | Google Sheets Trigger           | Start workflow on new form entry     | None                             | Extract and Structure Client Data  | Explains trigger and column key sensitivity (STEP 1 · Trigger & Intake)                                      |
| Extract and Structure Client Data | Set                            | Consolidate client data into string  | Trigger on New Client Form Submission | Client Checklist                   |                                                                                                             |
| Client Checklist               | Set                            | Provides onboarding checklist string | Extract and Structure Client Data | Personalize Using Gemini           | Advises editing checklist to fit service (STEP 2 · Checklist Context)                                        |
| Google Gemini Chat Model       | Langchain LLM (Google Gemini)  | AI model interface                   | None (used as AI model)          | Personalize Using Gemini           |                                                                                                             |
| Personalize Using Gemini       | Chain LLM                      | Generate personalized email body    | Client Checklist, Google Gemini Chat Model | Send Email to Client, Execution Completed | Emphasizes prompt design to generate only email body (STEP 3 · Personalize with Gemini)                        |
| Send Email to Client           | Gmail                         | Send personalized onboarding email  | Personalize Using Gemini          | Execution Completed                | Notes email sending setup and error handling (STEP 4 · Send & Run State)                                     |
| Execution Completed            | NoOp                          | Marks successful workflow completion | Send Email to Client              | None                             |                                                                                                             |
| Execution Failure             | NoOp                          | Marks failed workflow completion     | Error Handler                    | None                             |                                                                                                             |
| Error Handler                 | Error Trigger                 | Catch unhandled errors                | Workflow errors                  | Execution Failure                 |                                                                                                             |
| Sticky Note                   | Sticky Note                   | Explains trigger and data extraction | None                            | None                            | Explains Column keys and trigger (STEP 1 · Trigger & Intake)                                                |
| Sticky Note1                  | Sticky Note                   | Explains checklist context            | None                            | None                            | Advises on checklist editing (STEP 2 · Checklist Context)                                                  |
| Sticky Note2                  | Sticky Note                   | Explains AI personalization step     | None                            | None                            | Describes Gemini AI prompt constraints (STEP 3 · Personalize with Gemini)                                    |
| Sticky Note3                  | Sticky Note                   | Explains email sending and execution | None                            | None                            | Describes email sending and error/completion handling (STEP 4 · Send & Run State)                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger node**  
   - Type: Google Sheets Trigger  
   - Configure to watch a specific spreadsheet (Onboarding) and sheet ("Form Responses 1").  
   - Set event to "rowAdded".  
   - Poll frequency: every minute.  
   - Use OAuth2 credentials for Google Sheets with read access.

2. **Add a Set node named "Extract and Structure Client Data"**  
   - Connect from the trigger node.  
   - Add one field assignment:  
     - Name: `fields`  
     - Type: String  
     - Value:  
       ```
       Name:  {{ $json['Client name'] }} 
       Email:  {{ $json[' email '] }}
       Company: {{ $json['  Company Name  '] }}
       Service Needed: {{ $json['  Services Needed  '] }}
       Other info: {{ $json['  Any other onboarding info  '] }}
       ```  
   - Note: Pay attention to exact column names including spaces.

3. **Add another Set node named "Client Checklist"**  
   - Connect from "Extract and Structure Client Data".  
   - Assign a string field named `Checklist` with the multiline checklist text:  
     ```
     "Checklist": "
     1. Account setup
     2. Welcome call scheduled
     3. Document collection
     4. Service configuration
     5. Onboarding session
     6. First milestone review"
     ```  
   - This checklist is editable to match your onboarding process.

4. **Add a Google Gemini Chat Model node**  
   - Select Google Gemini 2.0 Flash model.  
   - Use Google PaLM API credentials with proper scopes and billing enabled.  
   - No direct input connections; this node serves as the AI model backend.

5. **Add a Chain LLM node named "Personalize Using Gemini"**  
   - Connect the output of "Client Checklist" to this node's main input.  
   - Configure the AI model to use the Google Gemini Chat Model node as its language model.  
   - Set prompt type to "define".  
   - Define prompt text exactly as:  
     ```
     Give me an onboarding check list for an email to the client, give me only email body and don't generate extra text like "Okay, here's an email template ..." and start and end on new lines
     start with:
     Hi {{ $('Trigger on New Client Form Submission').item.json['Client name'] }}, 
     and end with 
     Best regards,
     Your {{ $('Trigger on New Client Form Submission').item.json['  Company Name  '] }} Team

     :
     Also use information from checklist and Fields below
     {{ $json.Checklist }}

     Fields: {{ $('Extract and Structure Client Data').item.json.fields }}
     ```  
   - This prompt instructs the model to generate a clean email body only.

6. **Add a Gmail node named "Send Email to Client"**  
   - Connect from "Personalize Using Gemini".  
   - Configure to send email to `={{ $('Trigger on New Client Form Submission').item.json[' email '] }}`.  
   - Subject: `=Welcome to Our Service,  {{ $('Trigger on New Client Form Submission').item.json['Client name'] }}`  
   - Message body set to `= {{ $json.text }}` (the AI generated email).  
   - Use Gmail OAuth2 credentials with send mail permissions.  
   - If the message contains HTML, enable HTML mode manually.

7. **Add a NoOp node named "Execution Completed"**  
   - Connect from "Send Email to Client" on successful execution.

8. **Add an Error Trigger node named "Error Handler"**  
   - No input connections; triggers on workflow errors.

9. **Add a NoOp node named "Execution Failure"**  
   - Connect from "Error Handler" to mark failure state.

10. **Connect the error trigger flow:**  
    - From "Error Handler" to "Execution Failure".

11. **Add sticky notes for documentation (optional) to reflect:**  
    - Step 1: Trigger & Intake  
    - Step 2: Checklist Context  
    - Step 3: Personalize with Gemini  
    - Step 4: Send & Run State

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Column keys in Google Sheets include spaces (e.g., `" email "`, `"  Company Name  "`) and must be matched exactly.       | Important for correct data extraction from form submissions.                                                      |
| The onboarding checklist string can be edited to reflect your actual client onboarding stages and processes.             | Located in the "Client Checklist" Set node.                                                                       |
| The Gemini AI prompt is crafted to output only the email body with no extraneous text; adjust carefully if modifying.    | Ensures clean email content for direct sending.                                                                   |
| Gmail node requires OAuth2 credentials with send email permission; enable HTML mode if the email body includes HTML.    | Gmail API setup notes.                                                                                             |
| Google Gemini (PaLM) API requires billing enabled and appropriate access scopes; watch for quota or auth errors.         | See Google Cloud documentation for API setup.                                                                      |
| Error handling implemented with Error Trigger and NoOp nodes to distinguish success and failure states cleanly.          | Allows monitoring and troubleshooting of workflow runs.                                                           |
| More details on integrating Google Gemini with n8n via Langchain nodes: https://n8n.io/integrations/n8n-nodes-langchain   | Official n8n documentation.                                                                                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.