Automated Resume Screening & Candidate Routing with GPT-4 and LlamaParse via JotForm

https://n8nworkflows.xyz/workflows/automated-resume-screening---candidate-routing-with-gpt-4-and-llamaparse-via-jotform-9399


# Automated Resume Screening & Candidate Routing with GPT-4 and LlamaParse via JotForm

### 1. Workflow Overview

This workflow automates the screening of job applications submitted via a JotForm job application form, leveraging AI-powered resume parsing and analysis to categorize candidates and send personalized email responses. It targets HR and recruitment teams handling large volumes of applicants, aiming to save time and improve candidate communication quality.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception**: Triggered by a new JotForm submission; retrieves full submission data.
- **1.2 Resume Processing and Parsing**: Downloads the candidate’s uploaded resume PDF, uploads it to LlamaCloud for parsing, and polls for parsing completion.
- **1.3 AI Resume Analysis**: Sends parsed resume text and candidate info to an AI agent (via LangChain using GPT-4) for detailed candidate evaluation.
- **1.4 Data Cleaning and Categorization**: Cleans the AI output JSON, extracts key assessment fields, and categorizes candidate strength (Strong, Moderate, Weak).
- **1.5 Candidate Email Routing**: Based on AI categorization, sends personalized emails to candidates (interview invite, hold message, or rejection).
- **1.6 HR Notification**: Sends a comprehensive interview brief email to HR for strong candidates including candidate details, AI assessment, and suggested interview questions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for new job application submissions on JotForm and retrieves complete submission data for processing.

- **Nodes Involved:**  
  - JotForm Trigger  
  - Get Form Data

- **Node Details:**

  - **JotForm Trigger**  
    - *Type:* JotForm Trigger node  
    - *Role:* Listens for new submissions to the specified JotForm (Job Application form).  
    - *Configuration:* Linked to JotForm form ID `252802628500451`.  
    - *Credentials:* Uses JotForm API key stored in "JotForm account".  
    - *Inputs:* Webhook trigger on form submission.  
    - *Outputs:* Emits submission metadata including submission ID.  
    - *Edge Cases:* Network failures, invalid webhook setup, or form ID mismatch.

  - **Get Form Data**  
    - *Type:* HTTP Request node  
    - *Role:* Retrieves full submission data using submission ID from the trigger.  
    - *Configuration:* GET request to JotForm API endpoint with submission ID.  
    - *Credentials:* Uses HTTP Header Auth with JotForm API key.  
    - *Inputs:* Submission ID from JotForm Trigger.  
    - *Outputs:* Complete form answers including candidate info and uploaded resume URL.  
    - *Edge Cases:* API rate limits, invalid submission ID, auth errors.

---

#### 2.2 Resume Processing and Parsing

- **Overview:**  
  Downloads the candidate’s resume PDF from the URL in the form data, uploads it to LlamaCloud for parsing, and polls until parsing is complete.

- **Nodes Involved:**  
  - Download Resume  
  - Upload to LlamaCloud  
  - Check Parsing  
  - Wait 5 Seconds  
  - Retrieve Result

- **Node Details:**

  - **Download Resume**  
    - *Type:* HTTP Request  
    - *Role:* Downloads resume file using URL from form answers.  
    - *Configuration:* URL taken dynamically from form answer field `answers['8'].answer[0]`.  
    - *Credentials:* Uses HTTP header auth for JotForm.  
    - *Inputs:* Form data from Get Form Data.  
    - *Outputs:* Binary data of the resume PDF.  
    - *Edge Cases:* File not found, expired URL, download timeout.

  - **Upload to LlamaCloud**  
    - *Type:* HTTP Request  
    - *Role:* Uploads downloaded resume PDF to LlamaCloud parsing API.  
    - *Configuration:* POST multipart/form-data with file binary data.  
    - *Credentials:* HTTP header auth with Bearer token for LlamaCloud.  
    - *Inputs:* Binary resume file from Download Resume.  
    - *Outputs:* Parsing job ID and status.  
    - *Edge Cases:* API errors, auth failures, file size limits.

  - **Check Parsing**  
    - *Type:* HTTP Request  
    - *Role:* Polls LlamaCloud API for parsing job status by job ID.  
    - *Configuration:* GET request to job status endpoint with job ID.  
    - *Credentials:* LlamaCloud HTTP header auth.  
    - *Inputs:* Job ID from Upload to LlamaCloud response.  
    - *Outputs:* Parsing job status JSON.  
    - *Edge Cases:* Polling too fast, job failure, network issues.

  - **Wait 5 Seconds**  
    - *Type:* Wait node  
    - *Role:* Delays workflow for 5 seconds before the next polling attempt.  
    - *Inputs:* From Check Parsing if job status is not success.  
    - *Outputs:* Triggers another Check Parsing cycle.  
    - *Edge Cases:* Potential infinite loops if job never completes (should be safeguarded externally).

  - **Retrieve Result**  
    - *Type:* HTTP Request  
    - *Role:* Fetches parsed resume result in markdown format once parsing is successful.  
    - *Configuration:* GET request to result endpoint using job ID.  
    - *Credentials:* LlamaCloud HTTP header auth.  
    - *Outputs:* Parsed resume text in markdown.  
    - *Edge Cases:* Result not ready, API errors.

---

#### 2.3 AI Resume Analysis

- **Overview:**  
  Sends candidate details and the parsed resume markdown text to an AI agent for advanced analysis to score and categorize the candidate.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Composes a detailed prompt combining form data and resume markdown; sends to AI for JSON-formatted candidate assessment.  
    - *Configuration:* Prompt template includes candidate name, contact info, applied position, cover letter, and resume markdown. Asks for JSON structured response with compatibility score, recommendation, experience, skills, strengths, concerns, interview focus areas, and summary.  
    - *Inputs:* Parsed resume markdown from Retrieve Result, plus form data.  
    - *Outputs:* Raw AI-generated JSON assessment text.  
    - *Edge Cases:* AI response formatting errors, timeout, rate limits.  
    - *Version:* Uses LangChain agent with GPT-4 model.

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat node  
    - *Role:* Supports AI Agent node as underlying language model using GPT-4o-mini.  
    - *Credentials:* OpenAI API key.  
    - *Edge Cases:* API quota exhaustion, network failures.

---

#### 2.4 Data Cleaning and Categorization

- **Overview:**  
  Parses the AI agent's JSON text output, extracts key metrics and flags, and classifies candidate strength for routing.

- **Nodes Involved:**  
  - Clean Data  
  - Switch

- **Node Details:**

  - **Clean Data**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Cleans up AI JSON output by removing markdown code blocks, parsing JSON safely, and extracting key properties such as compatibility score, recommendation, skills arrays, strengths, concerns, relevant experience, interview focus areas, and overall assessment. Also sets boolean flags `is_strong_candidate`, `is_moderate_candidate`, `is_weak_candidate` based on score thresholds.  
    - *Inputs:* Raw AI JSON string from AI Agent.  
    - *Outputs:* Cleaned structured JSON object with flattened fields and flags.  
    - *Edge Cases:* Malformed AI output, JSON parse errors, missing fields.

  - **Switch**  
    - *Type:* Switch node  
    - *Role:* Routes flow based on candidate classification flags set in Clean Data node.  
    - *Configuration:* Three outputs for Strong, Moderate, and Weak candidates determined by boolean flags.  
    - *Inputs:* Cleaned candidate data.  
    - *Outputs:* Different email sending nodes per candidate category.  
    - *Edge Cases:* Multiple flags true (allMatchingOutputs enabled), no flags true (no match).

---

#### 2.5 Candidate Email Routing

- **Overview:**  
  Sends personalized email messages to candidates based on their AI-assessed category (Strong, Moderate, Weak).

- **Nodes Involved:**  
  - Send Email_1 (Strong Candidate)  
  - Send Email_2 (Moderate Candidate)  
  - Send Email_3 (Weak Candidate)

- **Node Details:**

  - **Send Email_1** (Strong Candidate)  
    - *Type:* Gmail node  
    - *Role:* Sends an enthusiastic interview invitation email, highlighting strengths and next steps.  
    - *Parameters:* Uses candidate’s email from form data; HTML email with styled content and dynamic fields (candidate name, applied position, interview time).  
    - *Credentials:* Gmail OAuth2.  
    - *Outputs:* Triggers Send Email to HR node.  
    - *Edge Cases:* Email delivery failures, invalid email addresses.

  - **Send Email_2** (Moderate Candidate)  
    - *Type:* Gmail node  
    - *Role:* Sends a polite hold message indicating the application is under consideration, with encouragement to share updates.  
    - *Credentials:* Gmail OAuth2.  
    - *Edge Cases:* Same as above.

  - **Send Email_3** (Weak Candidate)  
    - *Type:* Gmail node  
    - *Role:* Sends a considerate rejection email, encouraging future applications and providing resources.  
    - *Credentials:* Gmail OAuth2.  
    - *Edge Cases:* Same as above.

---

#### 2.6 HR Notification

- **Overview:**  
  Sends a detailed interview briefing email to HR with candidate details, AI assessment summary, strengths, concerns, interview focus areas, and suggested questions.

- **Nodes Involved:**  
  - Send Email to HR

- **Node Details:**

  - **Send Email to HR**  
    - *Type:* Gmail node  
    - *Role:* Sends a text email to HR email address (`hr@company.com`) summarizing candidate info and AI evaluation for interview prep.  
    - *Parameters:* Uses dynamic fields for candidate name, email, phone, scores, strengths, concerns, experience, and suggested questions.  
    - *Credentials:* Gmail OAuth2.  
    - *Edge Cases:* Email delivery issues, formatting of complex dynamic content.

---

### 3. Summary Table

| Node Name          | Node Type                      | Functional Role                      | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                          |
|--------------------|--------------------------------|------------------------------------|----------------------|----------------------|----------------------------------------------------------------------------------------------------|
| JotForm Trigger    | JotForm Trigger                | Trigger on new form submission     | -                    | Get Form Data        | AI-powered resume screening and candidate email automation overview and instructions.              |
| Get Form Data      | HTTP Request                  | Retrieve full form submission data | JotForm Trigger      | Download Resume       | Instructions for JotForm credentials and usage.                                                    |
| Download Resume    | HTTP Request                  | Download candidate resume PDF       | Get Form Data        | Upload to LlamaCloud  | JotForm API key usage instructions.                                                                |
| Upload to LlamaCloud| HTTP Request                  | Upload resume for parsing           | Download Resume      | Check Parsing         | LlamaCloud API key usage instructions.                                                             |
| Check Parsing      | HTTP Request                  | Poll for resume parsing status      | Upload to LlamaCloud | If / Wait 5 Seconds   | LlamaCloud API key usage instructions.                                                             |
| Wait 5 Seconds     | Wait                         | Pause between polling attempts      | Check Parsing        | Check Parsing         | -                                                                                                  |
| Retrieve Result    | HTTP Request                  | Retrieve parsed resume result       | If                   | AI Agent              | LlamaCloud API key usage instructions.                                                             |
| AI Agent           | LangChain Agent               | Analyze resume & candidate profile  | Retrieve Result      | Clean Data            | OpenAI GPT-4 usage instructions.                                                                   |
| Clean Data         | Code                         | Parse & flatten AI JSON output      | AI Agent             | Switch                | -                                                                                                  |
| Switch             | Switch                       | Route by candidate category         | Clean Data           | Send Email_1,2,3      | -                                                                                                  |
| Send Email_1       | Gmail                        | Email strong candidates             | Switch               | Send Email to HR      | Gmail OAuth2 setup instructions and email template info.                                          |
| Send Email_2       | Gmail                        | Email moderate candidates           | Switch               | -                    | Gmail OAuth2 setup instructions and email template info.                                          |
| Send Email_3       | Gmail                        | Email weak candidates               | Switch               | -                    | Gmail OAuth2 setup instructions and email template info.                                          |
| Send Email to HR   | Gmail                        | Send interview brief to HR          | Send Email_1         | -                    | Gmail OAuth2 setup instructions, includes detailed interview brief email template.                 |
| Sticky Note        | Sticky Note                  | General documentation and reminders | -                    | -                    | Various sticky notes cover API setup, credential instructions, and workflow overview.              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials (create JotForm API key and add to n8n credentials as "JotForm account").  
   - Set Form ID to your job application form (e.g., `252802628500451`).  
   - Position it as the workflow start node.

2. **Add Get Form Data Node**  
   - Type: HTTP Request  
   - Configure URL: `https://api.jotform.com/submission/{{ $json.submissionID }}`  
   - Use HTTP Header Auth credentials with header: `APIKEY = <your JotForm API key>`  
   - Connect output of JotForm Trigger to input of Get Form Data.

3. **Add Download Resume Node**  
   - Type: HTTP Request  
   - Set URL to: `={{ $json.content.answers['8'].answer[0] }}` (dynamic resume file URL).  
   - Use HTTP Header Auth with JotForm API key (same as Get Form Data).  
   - Connect Get Form Data output to Download Resume input.

4. **Add Upload to LlamaCloud Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`  
   - Set Content-Type to `multipart/form-data`.  
   - Attach file binary data from Download Resume as form binary data parameter named `file`.  
   - Use HTTP Header Auth credentials with Bearer token from LlamaCloud API key (add as "LlamaCloud").  
   - Connect Download Resume output to Upload to LlamaCloud.

5. **Add Check Parsing Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}` (use parsing job ID).  
   - Use LlamaCloud HTTP Header Auth credentials.  
   - Connect Upload to LlamaCloud output to Check Parsing input.

6. **Add If Node to Check Parsing Completion**  
   - Type: If  
   - Condition: `$json.status` equals `SUCCESS`  
   - Connect Check Parsing output to If node.

7. **Add Wait 5 Seconds Node**  
   - Type: Wait  
   - Default 5 seconds delay.  
   - Connect If node "False" output to Wait node, then connect Wait node back to Check Parsing node (poll until success).

8. **Add Retrieve Result Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`  
   - Use LlamaCloud credentials.  
   - Connect If node "True" output to Retrieve Result.

9. **Add AI Agent Node**  
   - Type: LangChain Agent  
   - Configure prompt with candidate info from form data and resume markdown from Retrieve Result.  
   - Ask for JSON output with detailed candidate assessment including scoring, skills, strengths, concerns, and recommendations.  
   - Connect Retrieve Result output to AI Agent input.

10. **Add OpenAI Chat Model Node**  
    - Type: LangChain OpenAI Chat  
    - Set model to `gpt-4o-mini` or similar GPT-4 variant.  
    - Use OpenAI API credentials.  
    - Connect AI Agent node's languageModel input to this node.

11. **Add Clean Data Node**  
    - Type: Code node with JavaScript code to parse AI Agent raw JSON output, extract fields and set candidate category flags.  
    - Connect AI Agent output to Clean Data input.

12. **Add Switch Node**  
    - Type: Switch  
    - Configure three outputs for `is_strong_candidate`, `is_moderate_candidate`, and `is_weak_candidate` flags from Clean Data.  
    - Connect Clean Data output to Switch input.

13. **Add Send Email Nodes for Candidates**  
    - Add three Gmail nodes: Send Email_1 (Strong), Send Email_2 (Moderate), Send Email_3 (Weak).  
    - Set recipient to candidate email from form data.  
    - Use Gmail OAuth2 credentials (named e.g., "Gmail account").  
    - Customize email subjects and HTML bodies per candidate category using dynamic expressions from form data and AI analysis.  
    - Connect Switch outputs accordingly.

14. **Add Send Email to HR Node**  
    - Type: Gmail  
    - Recipient: HR email (e.g., `hr@company.com`).  
    - Compose detailed text email with candidate info, AI scores, strengths, concerns, interview questions, and recommendations.  
    - Use Gmail OAuth2 credentials.  
    - Connect Send Email_1 output (Strong candidate path) to this node.

15. **Add Sticky Notes**  
    - Add notes for documenting API key setups, credential instructions, and workflow overview as per best practices.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| AI-powered resume screening and candidate email automation workflow helps HR teams save time by automating application review and communication.                                                                       | Workflow overview sticky note.                                                                          |
| Create a JotForm account and use Job Application Template: https://www.jotform.com/form-templates/job-application                                                                                                    | Sticky note with JotForm setup instructions.                                                           |
| LlamaCloud API for PDF parsing: https://cloud.llamaindex.ai/                                                                                                                                                          | Sticky note with LlamaCloud setup instructions.                                                        |
| OpenAI API key is required for GPT-4 usage: https://platform.openai.com/                                                                                                                                              | Sticky note on OpenAI setup.                                                                            |
| Gmail OAuth2 credentials must be created and configured in n8n for email sending nodes. Replace placeholders like `[Company Name]` and `[HR Manager Name]` in email templates before deployment.                       | Sticky note for Gmail setup and email customization.                                                  |
| The workflow expects the JotForm to have specific fields for resume upload, name, email, phone, applied position, cover letter, interview preferences, etc. Adapt expressions if your form structure differs.         | General note on form field dependencies and customizations.                                            |
| Polling interval and retries for LlamaCloud parsing should be tuned to avoid infinite loops or excessive API calls. Consider adding maximum retry limits or timeouts if needed.                                        | Best practice note on polling and error handling.                                                      |
| AI agent prompt is designed for HR recruiter perspective and expects JSON output. Validate AI outputs and handle parse errors gracefully to avoid workflow failures.                                                   | Note on AI prompt design and robustness.                                                               |

---

This documentation fully describes the structure, logic, and configuration of the Automated Resume Screening & Candidate Routing workflow, enabling reproduction, customization, and troubleshooting.