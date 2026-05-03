Classify & Process IT Requests with Jotform, Gemini AI and Google Workspace

https://n8nworkflows.xyz/workflows/classify---process-it-requests-with-jotform--gemini-ai-and-google-workspace-10127


# Classify & Process IT Requests with Jotform, Gemini AI and Google Workspace

### 1. Workflow Overview

This workflow automates the intake, classification, and processing of IT service requests submitted via a JotForm form. It targets IT support teams needing efficient handling of incoming tickets by leveraging AI-powered summarization and priority classification, centralized data storage in Google Sheets, automated notifications, and user acknowledgments.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception & Data Structuring:** Captures new IT service requests from JotForm, structures key fields for downstream processing.
- **1.2 AI Summarization & Priority Classification:** Uses Google Gemini AI and LangChain nodes to summarize the problem description and classify request priority levels (P0, P1, P2).
- **1.3 Data Storage & Routing:** Stores the processed request into a specific Google Sheets tab based on priority and sends urgent notifications.
- **1.4 User Acknowledgment:** Sends an email reply to the requester summarizing the ticket and confirming receipt.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Structuring

- **Overview:**  
  This block starts the workflow upon submission of a new IT service request via JotForm. It captures form data fields and standardizes them into structured variables for AI processing and storage.

- **Nodes Involved:**  
  - JotForm Trigger  
  - Downstream Fields  
  - Sticky Note (Form description)

- **Node Details:**

  - **JotForm Trigger**  
    - Type: Trigger node (JotForm Trigger)  
    - Role: Listens for new form submissions on a configured JotForm form.  
    - Configuration: Connected to a specific JotForm form via API credentials; triggers workflow on submission.  
    - Key Expressions: Captures all form fields including Full Name, Department, Email, Building Number, Problem Category, Comments.  
    - Inputs: None (trigger node)  
    - Outputs: JSON payload of submitted form data.  
    - Possible Failures: API authentication errors, JotForm downtime, webhook misconfiguration.  

  - **Downstream Fields**  
    - Type: Set node  
    - Role: Maps and renames incoming JSON fields to standardized variables for consistent downstream use.  
    - Configuration: Assigns fields such as Full Name, Department, E-mail, Building Number, Problem Category (first entry), Please Specify, and Comment and Questions.  
    - Key Expressions: Uses expressions like `{{$json['Full Name']}}`, `{{$json.Department}}`, etc.  
    - Inputs: Output from JotForm Trigger  
    - Outputs: Structured JSON object with renamed and grouped fields.  
    - Edge Cases: Missing fields or unexpected data formats could cause null values or errors.  

  - **Sticky Note (Form)**  
    - Purpose: Documents the form input block, listing captured fields and workflow start.  
    - No functional role.  

#### 2.2 AI Summarization & Priority Classification

- **Overview:**  
  This block uses AI to generate a concise summary of the IT request and classifies its priority based on problem description and comments, enabling automated routing.

- **Nodes Involved:**  
  - Summarization Chain (LangChain)  
  - Google Gemini Chat Model (PaLM)  
  - Priority Classifier (LangChain Text Classifier)  
  - Sticky Note (Summarization & Classifier)

- **Node Details:**

  - **Summarization Chain**  
    - Type: LangChain Summarization Chain  
    - Role: Generates a short summary text of the problem description for reporting and notification.  
    - Configuration: Default summarization options with AI backend integration.  
    - Inputs: Structured data from Downstream Fields node.  
    - Outputs: JSON with summarized text in `output.text`.  
    - Edge Cases: AI response timeout, incomplete or ambiguous input data.  

  - **Google Gemini Chat Model**  
    - Type: Language Model node (Google Gemini via PaLM API)  
    - Role: Provides AI language model capabilities to Summarization Chain and Priority Classifier.  
    - Configuration: Uses authenticated Google PaLM API account.  
    - Inputs/Outputs: Connected as AI language model backend to Summarization Chain and Priority Classifier nodes.  
    - Edge Cases: API rate limits, authentication failure, model unavailability.  

  - **Priority Classifier**  
    - Type: LangChain Text Classifier  
    - Role: Classifies the ticket priority into three categories: Low (P2), Medium (P1), High (P0).  
    - Configuration:  
      - Input text composed from "Please Specify" and "Comment and Questions" fields.  
      - Categories defined with descriptions to guide classification:  
        - Low: Non-critical, routine tasks  
        - Medium: Affects individual productivity  
        - High: Major impact, requires immediate attention  
    - Inputs: Summarized output text and structured fields.  
    - Outputs: Classification result controlling downstream routing.  
    - Edge Cases: Misclassification due to ambiguous text, classification failures or AI errors.  

  - **Sticky Note (Summarization & Classifier)**  
    - Purpose: Explains AI summarization and classification logic, priority definitions, and routing targets.  

#### 2.3 Data Storage & Routing

- **Overview:**  
  Stores the summarized and classified ticket data into the appropriate Google Sheets tab based on priority, and for P0 (High priority) requests, sends urgent notifications via Telegram.

- **Nodes Involved:**  
  - Store to Sheets P0 (Google Sheets Append)  
  - Store to Sheets P1 (Google Sheets Append)  
  - Store to Sheets P2 (Google Sheets Append)  
  - Send to Group (Telegram)  
  - Sticky Note (Automation overview)

- **Node Details:**

  - **Store to Sheets P0**  
    - Type: Google Sheets node (Append operation)  
    - Role: Appends P0 (High priority) tickets into the corresponding Google Sheets tab.  
    - Configuration:  
      - Maps fields such as Date (formatted current date/time), Name, Email, Status ("TODO"), Problem, Summary, Category, Priority ("P0"), Department, Building Number.  
      - Uses credentials for authorized access.  
    - Inputs: From Priority Classifier node upon P0 classification.  
    - Outputs: Passes data downstream to Reply User node.  
    - Edge Cases: Sheet access permissions, network errors, schema mismatch.  

  - **Store to Sheets P1**  
    - Same as above but for P1 (Medium priority) tickets, priority field set to "P1".  

  - **Store to Sheets P2**  
    - Same as above but for P2 (Low priority) tickets, priority field set to "P2".  

  - **Send to Group**  
    - Type: Telegram node  
    - Role: Sends urgent Telegram notification messages for P0 tickets to a configured group/channel.  
    - Configuration:  
      - Message text includes alert, summary, request details, comments, requester name, department, category, building number.  
      - Uses Telegram API credentials.  
    - Inputs: Connected only to P0 path from Priority Classifier.  
    - Outputs: Feeds into Store to Sheets P0 node (ensuring notification before storage).  
    - Edge Cases: Telegram API limits, connectivity, malformed message.  

  - **Sticky Note (Automation overview)**  
    - Explains overall automation goals: faster response, centralized tracking, AI-driven summarization and routing.  

#### 2.4 User Acknowledgment

- **Overview:**  
  Sends an email confirming receipt of the IT service request, including a summary and request details, to the requester.

- **Nodes Involved:**  
  - Reply User (Gmail)  

- **Node Details:**

  - **Reply User**  
    - Type: Gmail node (send email)  
    - Role: Sends acknowledgment emails to users who submitted IT requests.  
    - Configuration:  
      - Email recipient: dynamically set from requester's Email field.  
      - Subject: includes requester’s name, department, category, and date.  
      - Message body: polite confirmation including summary and full problem description.  
      - Uses OAuth2 Gmail credentials for authorized sending.  
    - Inputs: Connected from all three Google Sheets storage nodes (P0, P1, P2), ensuring acknowledgment regardless of priority.  
    - Outputs: None (end node).  
    - Edge Cases: Email sending failures, invalid email addresses, OAuth token expiry.  

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                         | Input Node(s)           | Output Node(s)                      | Sticky Note                                                                                             |
|---------------------|----------------------------------|---------------------------------------|------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------|
| JotForm Trigger     | JotForm Trigger                  | Workflow start, captures form input   | None                   | Downstream Fields                  | ## Form\n\nStarts workflow when a new IT Service Request is submitted on Jotform.\n\nCaptures: Full Name, Department, Email, Building Number, Problem Category, and Comments.\n\nConnects to “Set Fields” node for structured data mapping. |
| Downstream Fields   | Set                             | Structures incoming data fields        | JotForm Trigger        | Summarization Chain               |                                                                                                       |
| Summarization Chain | LangChain Summarization Chain   | Summarizes problem description         | Downstream Fields      | Priority Classifier               | ## Summarizaion & Classifer\n\n- Uses AI to summarize problem details for reports and alerts.\n- Works with Google Gemini model for summarization and classification.\n- Produces a short “Summary” used in Sheets, emails, and Telegram messages.\n- Analyzes request details (problem + comments).\n\n### Classifies into:\nP0 (High) – Immediate action\nP1 (Medium) – Affects single user\nP2 (Low) – Non-critical maintenance\n\nRoutes to correct Google Sheet tab and notification channel. |
| Google Gemini Chat Model | Google Gemini AI Model        | AI backend for summarization/classification | Connected as AI backend to Summarization Chain and Priority Classifier | Priority Classifier, Summarization Chain |                                                                                                       |
| Priority Classifier | LangChain Text Classifier       | Classifies priority (P0, P1, P2)      | Summarization Chain    | Store to Sheets P2, Store to Sheets P1, Send to Group |                                                                                                       |
| Store to Sheets P2  | Google Sheets Append            | Stores low priority requests (P2)      | Priority Classifier    | Reply User                       |                                                                                                       |
| Store to Sheets P1  | Google Sheets Append            | Stores medium priority requests (P1)   | Priority Classifier    | Reply User                       |                                                                                                       |
| Store to Sheets P0  | Google Sheets Append            | Stores high priority requests (P0)     | Send to Group          | Reply User                       |                                                                                                       |
| Send to Group       | Telegram                       | Sends urgent notification for P0       | Priority Classifier    | Store to Sheets P0               |                                                                                                       |
| Reply User          | Gmail (OAuth2)                  | Sends acknowledgement email to user   | Store to Sheets P0, Store to Sheets P1, Store to Sheets P2 | None                            |                                                                                                       |
| Sticky Note         | Sticky Note                    | Documentation: Form input block        | None                   | None                            | ## Form\n\nStarts workflow when a new IT Service Request is submitted on Jotform.\n\nCaptures: Full Name, Department, Email, Building Number, Problem Category, and Comments.\n\nConnects to “Set Fields” node for structured data mapping. |
| Sticky Note1        | Sticky Note                    | Documentation: AI summarization/classifier | None                   | None                            | ## Summarizaion & Classifer\n\n- Uses AI to summarize problem details for reports and alerts.\n- Works with Google Gemini model for summarization and classification.\n- Produces a short “Summary” used in Sheets, emails, and Telegram messages.\n- Analyzes request details (problem + comments).\n\n### Classifies into:\nP0 (High) – Immediate action\nP1 (Medium) – Affects single user\nP2 (Low) – Non-critical maintenance\n\nRoutes to correct Google Sheet tab and notification channel. |
| Sticky Note2        | Sticky Note                    | Documentation: Overall automation goals | None                   | None                            | ## Automate IT Service Request intake → classification → storage → notification → acknowledgment.\n\n- Faster IT response time\n- Centralized data in Google Sheets\n- AI-driven ticket summarization\n- Automatic priority routing and communication |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow named** "JOTFORM IT Service Request".

2. **Add JotForm Trigger node:**  
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials.  
   - Select the specific IT service request form to listen for submissions.  
   - Position: Left side, as start node.

3. **Add Set node (Downstream Fields):**  
   - Connect input from JotForm Trigger.  
   - Add assignments for the following variables with expressions referencing incoming JSON:  
     - Full Name (object) = `{{$json['Full Name']}}`  
     - Department (string) = `{{$json.Department}}`  
     - E-mail (string) = `{{$json['E-mail']}}`  
     - Building Number (string) = `{{$json['Building Number']}}`  
     - Problem Category (string) = `{{$json['Problem Category'][0]}}`  
     - Please Specify (string) = `{{$json['Please Specify']}}`  
     - Comment and Questions (string) = `{{$json['Comment and Questions']}}`

4. **Add LangChain Summarization Chain node:**  
   - Connect input from Downstream Fields node.  
   - Use default summarization settings to produce a short summary of the problem description.

5. **Add Google Gemini Chat Model node:**  
   - Configure with Google PaLM API credentials.  
   - Set as AI backend for both Summarization Chain and Priority Classifier nodes.

6. **Add LangChain Text Classifier node (Priority Classifier):**  
   - Connect input from Summarization Chain output.  
   - Configure input text with:  
     ```
     Description:
     {{$json['Please Specify']}}

     Comments:
     {{$json['Comment and Questions']}}
     ```  
   - Define three categories with descriptions:  
     - Low: Non-critical, routine maintenance (P2)  
     - Medium: Affects individual user, moderate priority (P1)  
     - High: Major impact, immediate attention (P0)

7. **Add three Google Sheets Append nodes:**  
   - **Store to Sheets P0:** For P0 priority tickets  
     - Connect input from Send to Group node (see step 8).  
     - Map columns: Date (current date/time), Name, Email, Status ("TODO"), Problem, Summary, Category, Priority ("P0"), Department, Building Number.  
     - Configure Google Sheets credentials and target spreadsheet and sheet tab for P0 tickets.  
   - **Store to Sheets P1:** For P1 priority tickets  
     - Connect input from Priority Classifier node's medium priority output.  
     - Same column mappings, set Priority to "P1".  
     - Configure accordingly.  
   - **Store to Sheets P2:** For P2 priority tickets  
     - Connect input from Priority Classifier node's low priority output.  
     - Same column mappings, set Priority to "P2".  
     - Configure accordingly.

8. **Add Telegram node (Send to Group):**  
   - Connect input from Priority Classifier node's high priority output (P0).  
   - Configure Telegram API credentials.  
   - Compose message text including alert prefix, summary, problem, comments, requester name, department, category, building number.  
   - Connect output to Store to Sheets P0 node to ensure storage after notification.

9. **Add Gmail node (Reply User):**  
   - Connect inputs from all three Store to Sheets nodes outputs (P0, P1, P2).  
   - Configure Gmail OAuth2 credentials for sending email.  
   - Set recipient to requester's email (`{{$json.Email}}`).  
   - Subject: `"IT Service Request: {{$json.Name}} - {{$json.Department}} - {{$json.Category}} - {{$json.Date}}"`  
   - Message body: Include greeting, acknowledgment, summary, full problem description, closing signature.

10. **Add Sticky Notes as needed:**  
    - To document form input fields, AI summarization/classification logic, and overall workflow goals for maintainability.

11. **Test the workflow:**  
    - Submit sample form entries to verify correct AI summarization, priority classification, data storage, notification, and email acknowledgment.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow improves IT support efficiency by integrating JotForm submissions with AI-powered summarization and priority classification using Google Gemini (PaLM) and LangChain.                                           | Project description                                 |
| Google Gemini model requires valid PaLM API credentials with access to the chat model. Ensure API quotas and billing are set up.                                                                                              | Google Cloud Platform documentation                  |
| Gmail node uses OAuth2 credentials; token refresh must be configured to avoid email sending interruptions.                                                                                                                    | Gmail OAuth2 n8n documentation                        |
| Telegram notifications are sent only for P0 (high priority) requests to alert the IT support group immediately.                                                                                                              | Telegram Bot API docs                                 |
| The workflow uses dynamic expressions extensively; malformed or missing data in form submissions can cause failures or incorrect processing. Input validation on JotForm is recommended.                                      | JotForm field validation                             |
| Google Sheets append operations require correct sheet and tab IDs; ensure the document schema matches the mapped fields to avoid data corruption.                                                                            | Google Sheets API docs                               |
| The workflow is designed for n8n version supporting LangChain nodes and Google Gemini model integrations (v2.1+ for Summarization Chain, v1.1+ for Priority Classifier).                                                       | n8n node version compatibility                       |
| For improved reliability, add error handling nodes or workflow-level retry mechanisms for API calls to JotForm, Google APIs, and Telegram.                                                                                    | n8n error handling best practices                     |
| Project credits: Inspired by integration use cases combining JotForm, AI language models, Google Workspace, and messaging platforms for IT ticket automation.                                                                | n8n community forums and official blog                |
| Video tutorials on integrating LangChain with n8n and Google Gemini can be found at https://n8n.io/blog and https://developers.google.com/palm for advanced customization and troubleshooting.                                 | Official n8n and Google developers resources          |

---

This structured documentation enables comprehensive understanding, modification, and reproduction of the "JOTFORM IT Service Request" workflow, supporting both advanced users and automation agents.