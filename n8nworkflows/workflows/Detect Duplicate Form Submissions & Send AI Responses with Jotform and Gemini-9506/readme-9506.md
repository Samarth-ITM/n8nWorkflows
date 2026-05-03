Detect Duplicate Form Submissions & Send AI Responses with Jotform and Gemini

https://n8nworkflows.xyz/workflows/detect-duplicate-form-submissions---send-ai-responses-with-jotform-and-gemini-9506


# Detect Duplicate Form Submissions & Send AI Responses with Jotform and Gemini

### 1. Workflow Overview

This workflow automates the detection of duplicate business registration form submissions via Jotform and sends appropriate AI-generated email responses. It is designed to maintain clean records by preventing duplicate entries and ensuring submitters receive professional communication tailored to their submission status.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures new form submissions from Jotform via webhook.
- **1.2 Data Extraction & Retrieval:** Extracts key submission data and fetches all previous submissions for comparison.
- **1.3 Duplicate Detection:** Unpacks submissions, filters for active entries with matching emails, and determines if the new submission is a duplicate.
- **1.4 Duplicate Handling:** If duplicate, deletes the submission and sends a rejection notification email generated via AI.
- **1.5 New Submission Handling:** If new, generates a welcome email using AI and sends it to the submitter.
- **1.6 Email Format Validation and Parsing:** Uses AI to structure emails in JSON format, ensuring valid, professional output.
- **1.7 Auxiliary:** Includes sticky notes for setup guidance and workflow explanation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives real-time form submission data from Jotform to start the workflow.

**Nodes Involved:**  
- Form Submission Received

**Node Details:**  

- **Form Submission Received**  
  - Type: Jotform Trigger node  
  - Role: Listens for new submissions on a specified Jotform form via webhook.  
  - Configuration:  
    - Form ID set to the target Jotform form.  
    - `onlyAnswers` set to false to receive full submission data.  
  - Credentials: JotForm API account with valid API key.  
  - Inputs: External webhook trigger from Jotform.  
  - Outputs: JSON object containing full submission data, including `formID`, `submissionID`, and raw submission details.  
  - Failure points: Missing webhook setup, invalid form ID, or credential errors.  
  - Version: 1  

---

#### 1.2 Data Extraction & Retrieval

**Overview:**  
Extracts relevant submission identifiers and raw data, then retrieves all previous submissions for the form from Jotform API.

**Nodes Involved:**  
- Extract Submission Data  
- Fetch All Form Submissions

**Node Details:**  

- **Extract Submission Data**  
  - Type: Set node  
  - Role: Assigns key fields (`formID`, `submissionID`, `rawRequest`) from the submission JSON for easy reference downstream.  
  - Configuration: Uses expressions to map from incoming JSON fields.  
  - Inputs: Output from Form Submission Received node.  
  - Outputs: JSON with assigned fields for subsequent nodes.  
  - Version: 3.4  
  - Failure points: Missing expected JSON fields if form structure changes.

- **Fetch All Form Submissions**  
  - Type: HTTP Request node  
  - Role: Calls Jotform API to fetch all submissions for the given form.  
  - Configuration:  
    - URL dynamically constructed using `formID` from extracted data.  
    - Query parameter includes Jotform API key credential.  
  - Inputs: Output from Extract Submission Data node.  
  - Outputs: API response containing all form submissions in JSON.  
  - Failure points: API authentication errors, rate limits, or network failures.  
  - Version: 4.2  

---

#### 1.3 Duplicate Detection

**Overview:**  
Processes all submissions data to isolate active entries and identify duplicates based on the email address submitted.

**Nodes Involved:**  
- Unpack Submission Objects  
- Filter Active Submissions  
- Check Duplicate Count

**Node Details:**  

- **Unpack Submission Objects**  
  - Type: Split Out node  
  - Role: Splits the array of submissions into individual items for filtering.  
  - Configuration: Splits on `content` field, including `content.id` and `content.answers`.  
  - Inputs: Output from Fetch All Form Submissions node.  
  - Outputs: Each submission as a separate JSON object.  
  - Failure points: Unexpected API response structure.  
  - Version: 1  

- **Filter Active Submissions**  
  - Type: Filter node  
  - Role: Keeps only submissions with status "ACTIVE" and matching email address to the current submission.  
  - Configuration:  
    - Condition 1: `content.status` equals "ACTIVE".  
    - Condition 2: Email answer (field ID "5") equals the email from the incoming submission's raw request.  
  - Inputs: Output from Unpack Submission Objects node.  
  - Outputs: Filtered list of active submissions matching email.  
  - Failure points: Email field ID mismatch, missing fields, case sensitivity issues.  
  - Version: 2.2  

- **Check Duplicate Count**  
  - Type: If node  
  - Role: Determines if the current submission is a duplicate by checking if the count of matching submissions is two or more.  
  - Configuration:  
    - Condition 1: Email in each submission equals incoming submission email.  
    - Condition 2: The number of matching items (`$input.all().length`) is >= 2.  
  - Inputs: Output from Filter Active Submissions node.  
  - Outputs:  
    - True branch: Duplicate found.  
    - False branch: New submission.  
  - Failure points: Logic errors if input array is empty or missing.  
  - Version: 2.2  

---

#### 1.4 Duplicate Handling

**Overview:**  
Deletes duplicate submission in Jotform and sends an AI-generated rejection email to notify the submitter.

**Nodes Involved:**  
- Delete Duplicate Submission  
- Compose Rejection Email  
- Deliver Rejection Notice  
- Gemini LLM (for rejection email generation)  
- Validate Email Format (AI output parser)

**Node Details:**  

- **Delete Duplicate Submission**  
  - Type: HTTP Request node  
  - Role: Deletes the current submission from Jotform using API.  
  - Configuration:  
    - URL dynamically built using `submissionID` from Extract Submission Data node.  
    - Method: DELETE.  
    - Query parameter includes Jotform API key credential.  
  - Inputs: True branch from Check Duplicate Count node.  
  - Outputs: Confirmation of deletion.  
  - Failure points: API key invalid, submission already deleted, network issues.  
  - Version: 4.2  

- **Compose Rejection Email**  
  - Type: Langchain Agent node  
  - Role: Generates a professional, friendly rejection email for duplicate submissions using AI.  
  - Configuration:  
    - Prompt includes extracted submission data and instructions to produce JSON with email subject and HTML body.  
    - System message guides tone, content, and formatting.  
    - Output parser enabled for JSON.  
  - Inputs: Output from Delete Duplicate Submission node.  
  - Outputs: AI-generated JSON email content.  
  - Failure points: AI service errors, malformed outputs, missing data.  
  - Version: 2.2  

- **Gemini LLM**  
  - Type: Langchain Google Gemini Chat node  
  - Role: Underlying AI language model used by Compose Rejection Email node.  
  - Credentials: Google Gemini (PaLM) API.  
  - Inputs: Prompt from Compose Rejection Email node.  
  - Outputs: Raw AI response.  
  - Failure points: API quota, network, or auth failures.  
  - Version: 1  

- **Validate Email Format**  
  - Type: Langchain Output Parser Structured node  
  - Role: Parses and validates AI-generated JSON email content to ensure correctness.  
  - Inputs: Output from Gemini LLM node.  
  - Outputs: Structured JSON for email sending.  
  - Failure points: Parsing errors if AI output is invalid JSON.  
  - Version: 1.3  

- **Deliver Rejection Notice**  
  - Type: Gmail node  
  - Role: Sends the rejection email to the submitter using Gmail SMTP with OAuth2.  
  - Configuration:  
    - To: Email extracted from submission raw data.  
    - Subject and message body from AI-generated JSON output.  
  - Credentials: Gmail OAuth2.  
  - Inputs: Output from Compose Rejection Email node (through parser).  
  - Failure points: Email sending failures, credential expiration, quota limits.  
  - Version: 2.1  

---

#### 1.5 New Submission Handling

**Overview:**  
For new, unique submissions, generates a professional welcome email using AI and sends it to the submitter.

**Nodes Involved:**  
- Generate Welcome Email  
- Gemini LLM (Welcome)  
- Parse Welcome Email JSON  
- Send Welcome Email

**Node Details:**  

- **Generate Welcome Email**  
  - Type: Langchain Agent node  
  - Role: Uses AI to generate a welcome email based on submission details.  
  - Configuration:  
    - Prompt instructs AI to produce JSON with subject and HTML body including personalized content.  
    - System message defines tone, formatting, and content expectations.  
    - Output parser enabled.  
  - Inputs: False branch from Check Duplicate Count node.  
  - Outputs: AI-generated JSON email content.  
  - Failure points: AI service availability, malformed output.  
  - Version: 2.2  

- **Gemini LLM (Welcome)**  
  - Type: Langchain Google Gemini Chat node  
  - Role: AI model used by Generate Welcome Email node.  
  - Credentials: Google Gemini (PaLM) API.  
  - Inputs: Prompt from Generate Welcome Email node.  
  - Outputs: Raw AI response.  
  - Failure points: API or credential issues.  
  - Version: 1  

- **Parse Welcome Email JSON**  
  - Type: Langchain Output Parser Structured node  
  - Role: Parses AI output to ensure valid JSON email content for sending.  
  - Inputs: Output from Gemini LLM (Welcome) node.  
  - Outputs: Structured JSON with email subject and HTML body.  
  - Failure points: Parsing errors due to invalid JSON.  
  - Version: 1.3  

- **Send Welcome Email**  
  - Type: Gmail node  
  - Role: Sends the AI-generated welcome email to the submitter.  
  - Configuration:  
    - To: Email from submission raw data.  
    - Subject and message body from parsed AI output.  
  - Credentials: Gmail OAuth2.  
  - Inputs: Output from Parse Welcome Email JSON node.  
  - Failure points: Email sending failure, credential issues.  
  - Version: 2.1  

---

#### 1.6 Email Format Validation and Parsing (Supporting AI Outputs)

**Overview:**  
Ensures AI-generated email JSON responses conform to expected structure and content, preventing malformed emails.

**Nodes Involved:**  
- Validate Email Format  
- Parse Welcome Email JSON  

**Node Details:**  

- Both nodes use Langchain Output Parser Structured to validate and parse AI outputs into JSON with fields `email_subject` and `email_body_html`.  
- They serve as checkpoint nodes to catch AI output errors before email sending.  
- Failure points include invalid JSON from AI models that would cause downstream failures.

---

#### 1.7 Auxiliary: Sticky Notes

**Overview:**  
Provides detailed documentation, setup instructions, and explanations for the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**  

- **Sticky Note**  
  - Contains a comprehensive textual overview of workflow purpose, stepwise logic, features, and a Jotform link.  
  - Positioned off to the side for user reference.  

- **Sticky Note1**  
  - Provides stepwise setup guide including how to obtain API keys, configure Google Gemini API, set up Gmail OAuth2, and map form fields.  
  - Essential for initial deployment and credential setup.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                               | Input Node(s)                  | Output Node(s)                             | Sticky Note                                                                                                                    |
|-----------------------------|---------------------------------|----------------------------------------------|-------------------------------|--------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Form Submission Received     | Jotform Trigger                 | Receive new form submissions                  | (Webhook trigger)              | Extract Submission Data                     | See sticky note for full workflow overview and setup instructions                                                              |
| Extract Submission Data      | Set                            | Extract key submission fields                  | Form Submission Received       | Fetch All Form Submissions                  |                                                                                                                               |
| Fetch All Form Submissions   | HTTP Request                   | Retrieve all submissions from Jotform API     | Extract Submission Data        | Unpack Submission Objects                   |                                                                                                                               |
| Unpack Submission Objects    | Split Out                     | Split submissions array into individual items | Fetch All Form Submissions     | Filter Active Submissions                    |                                                                                                                               |
| Filter Active Submissions    | Filter                        | Filter active submissions matching email      | Unpack Submission Objects      | Check Duplicate Count                        |                                                                                                                               |
| Check Duplicate Count        | If                            | Determine if submission is duplicate          | Filter Active Submissions      | Delete Duplicate Submission, Generate Welcome Email |                                                                                                                               |
| Delete Duplicate Submission  | HTTP Request                  | Delete duplicate submission from Jotform     | Check Duplicate Count (true)   | Compose Rejection Email                      |                                                                                                                               |
| Compose Rejection Email      | Langchain Agent               | Generate AI rejection email                    | Delete Duplicate Submission    | Deliver Rejection Notice                     |                                                                                                                               |
| Deliver Rejection Notice     | Gmail                         | Send rejection email                           | Compose Rejection Email        |                                            |                                                                                                                               |
| Generate Welcome Email       | Langchain Agent               | Generate AI welcome email                      | Check Duplicate Count (false)  | Send Welcome Email                           |                                                                                                                               |
| Send Welcome Email           | Gmail                         | Send welcome email                             | Generate Welcome Email         |                                            |                                                                                                                               |
| Gemini LLM                  | Langchain Google Gemini Chat   | AI model for rejection email                   | Compose Rejection Email        | Validate Email Format                        |                                                                                                                               |
| Validate Email Format        | Langchain Output Parser        | Parse and validate AI rejection email JSON    | Gemini LLM                    | Compose Rejection Email                      |                                                                                                                               |
| Gemini LLM (Welcome)         | Langchain Google Gemini Chat   | AI model for welcome email                      | Generate Welcome Email         | Parse Welcome Email JSON                     |                                                                                                                               |
| Parse Welcome Email JSON     | Langchain Output Parser        | Parse and validate AI welcome email JSON      | Gemini LLM (Welcome)           | Generate Welcome Email                       |                                                                                                                               |
| Sticky Note                 | Sticky Note                   | Documentation of workflow overview             |                               |                                            | Contains detailed workflow overview and rationale                                                                             |
| Sticky Note1                | Sticky Note                   | Setup and configuration guide                   |                               |                                            | Contains step by step setup instructions with links                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Jotform Trigger node: "Form Submission Received"**  
   - Set the node to listen to your target form by specifying the Jotform Form ID.  
   - Configure webhook ID as needed.  
   - Attach JotForm API credentials.  
   - Verify webhook registration on Jotform side.

2. **Add a Set node: "Extract Submission Data"**  
   - Assign variables:  
     - `formID` = `{{$json.formID}}`  
     - `submissionID` = `{{$json.submissionID}}`  
     - `rawRequest` = `{{$json.rawRequest}}` (object)  
   - Connect output from "Form Submission Received".

3. **Add HTTP Request node: "Fetch All Form Submissions"**  
   - URL: `https://api.jotform.com/form/{{$json.formID}}/submissions`  
   - Method: GET  
   - Query parameter: `apiKey` with your Jotform API Key credential.  
   - Connect output from "Extract Submission Data".

4. **Add Split Out node: "Unpack Submission Objects"**  
   - Field to split out: `content`  
   - Include fields: `content.id`, `content.answers`  
   - Connect output from "Fetch All Form Submissions".

5. **Add Filter node: "Filter Active Submissions"**  
   - Filter conditions:  
     - `content.status` equals "ACTIVE"  
     - `content.answers["5"].answer` equals `{{$node["Form Submission Received"].json["rawRequest"]["E-mail"]}}` (adjust "5" to your email field ID)  
   - Connect output from "Unpack Submission Objects".

6. **Add If node: "Check Duplicate Count"**  
   - Conditions:  
     - Email in submission equals incoming email  
     - Number of filtered items (`$input.all().length`) >= 2  
   - Connect output from "Filter Active Submissions".

7. **On True branch (Duplicate):**

   a. **Add HTTP Request node: "Delete Duplicate Submission"**  
      - URL: `https://api.jotform.com/submission/{{$node["Extract Submission Data"].json["submissionID"]}}`  
      - Method: DELETE  
      - Query parameter: `apiKey` with your Jotform API Key.  
      - Connect from If node True branch.

   b. **Add Langchain Agent node: "Compose Rejection Email"**  
      - Prompt: Detailed instructions to generate a friendly duplicate notification email in JSON format with subject and HTML body.  
      - System message: Professional, courteous tone with clear next steps.  
      - Enable output parsing.  
      - Connect output from "Delete Duplicate Submission".

   c. **Add Langchain Google Gemini Chat node: "Gemini LLM"**  
      - Connect input from "Compose Rejection Email" prompt.  
      - Attach Google Gemini (PaLM) API credentials.  

   d. **Add Langchain Output Parser Structured node: "Validate Email Format"**  
      - JSON schema example with fields: `email_subject`, `email_body_html`.  
      - Connect output from "Gemini LLM".

   e. **Add Gmail node: "Deliver Rejection Notice"**  
      - Send To: Email from submission raw data.  
      - Subject and message body from parsed AI output.  
      - Use Gmail OAuth2 credentials.  
      - Connect output from "Validate Email Format".

8. **On False branch (New submission):**

   a. **Add Langchain Agent node: "Generate Welcome Email"**  
      - Prompt: Instructions to generate a welcome email in JSON format with subject and HTML body, personalized with submission data.  
      - System message: Professional, concise, mobile-responsive email tone.  
      - Enable output parsing.  
      - Connect output from If node False branch.

   b. **Add Langchain Google Gemini Chat node: "Gemini LLM (Welcome)"**  
      - Connect input from "Generate Welcome Email".  
      - Attach Google Gemini API credentials.

   c. **Add Langchain Output Parser Structured node: "Parse Welcome Email JSON"**  
      - JSON schema example with fields: `email_subject`, `email_body_html`.  
      - Connect output from "Gemini LLM (Welcome)".

   d. **Add Gmail node: "Send Welcome Email"**  
      - Send To: Email from submission raw data.  
      - Subject and message body from parsed AI output.  
      - Use Gmail OAuth2 credentials.  
      - Connect output from "Parse Welcome Email JSON".

9. **Add Sticky Note nodes for documentation**  
   - One for workflow overview and rationale.  
   - One for setup instructions including API key acquisition, OAuth2 setup, and field mappings.

10. **Credential setup:**  
    - Add Jotform API credentials with API key.  
    - Add Google Gemini (PaLM) API credentials with API key.  
    - Add Gmail OAuth2 credentials with Client ID and Client Secret, perform Google sign-in and authorization.

11. **Validate field mappings:**  
    - Confirm email field ID matches your form (e.g., answer ID "5" for email). Adjust in Filter Active Submissions and other nodes accordingly.

12. **Test workflow:**  
    - Submit test forms, verify duplicate detection, email generation, and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                             | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow automatically detects duplicate business registrations in Jotform and sends AI-generated emails accordingly. It prevents duplicate database entries and maintains clean records with professional communication.                                                         | Workflow purpose                                                                                             |
| Setup Guide details how to obtain Jotform API key, configure Google Gemini API, set up Gmail OAuth2 credentials, and map form fields. Essential before importing workflow.                                                                                                               | Setup instructions in Sticky Note1                                                                           |
| Jotform Form ID and email field ID must be correct and consistent across nodes for accurate duplicate detection.                                                                                                                                                                        | Input field mapping context                                                                                   |
| Google Gemini (PaLM) API is used as the AI language model for email generation. Ensure API key is active and has quota.                                                                                                                                                                  | AI integration note                                                                                           |
| Gmail OAuth2 credentials require proper OAuth consent and redirect URI configuration in Google Cloud Console.                                                                                                                                                                            | Email sending setup                                                                                           |
| Workflow includes inline CSS in email HTML for mobile responsiveness and professional appearance.                                                                                                                                                                                       | Email formatting best practices                                                                               |
| Jotform API rate limits and network latency may impact workflow execution times.                                                                                                                                                                                                        | API usage considerations                                                                                      |
| Workflow designed for business registration forms; adapt prompts and field mappings if used for other form types.                                                                                                                                                                      | Customization note                                                                                            |
| Link to Jotform platform for quick form creation: [https://www.jotform.com/?partner=roshanramanidev](https://www.jotform.com/?partner=roshanramanidev)                                                                                                                                   | Partner link from sticky note                                                                                  |
| Google Cloud Console for API & credential creation: [https://console.cloud.google.com](https://console.cloud.google.com)                                                                                                                                                                | Google API setup link                                                                                         |
| Gmail API enablement and OAuth2 client creation guide available in Google Cloud documentation.                                                                                                                                                                                          | Gmail OAuth setup resources                                                                                   |

---

**Disclaimer:** The provided workflow text originates exclusively from a workflow automated with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.