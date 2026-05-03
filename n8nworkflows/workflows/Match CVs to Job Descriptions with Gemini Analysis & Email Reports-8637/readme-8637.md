Match CVs to Job Descriptions with Gemini Analysis & Email Reports

https://n8nworkflows.xyz/workflows/match-cvs-to-job-descriptions-with-gemini-analysis---email-reports-8637


# Match CVs to Job Descriptions with Gemini Analysis & Email Reports

### 1. Workflow Overview

This workflow, titled **"Match CVs to Job Descriptions with Gemini Analysis & Email Reports"**, automates the process of analyzing a candidate’s CV against a specified job posting URL using AI (Google Gemini/PaLM). It extracts relevant skills, assesses fit, identifies gaps, and generates personalized recommendations and optimization tips. The candidate receives a structured email report summarizing the analysis, helping them improve their CV and better target job applications.

**Target Use Cases:**  
- Job seekers wanting objective, AI-powered feedback on how well their CV matches a job posting.  
- Recruiters or career advisors automating CV-job matching and personalized feedback.  
- Platforms or services offering AI-based career guidance and CV optimization.  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives candidate's CV upload, job posting URL, and email via a webhook and serves an HTML form.  
- **1.2 CV Text Extraction & Preparation:** Extracts plain text from the uploaded PDF CV and prepares it for analysis.  
- **1.3 Job Posting Retrieval and Cleaning:** Fetches the job posting webpage content and cleans it to plain text.  
- **1.4 Data Merging:** Combines CV text and cleaned job posting text for AI input.  
- **1.5 AI Analysis:** Uses Google Gemini/PaLM AI to analyze the CV against the job posting, producing a structured JSON output with fit scores, recommendations, and advice.  
- **1.6 Email Reporting:** Sends a personalized email report to the candidate summarizing the AI analysis results.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block handles HTTP requests from candidates submitting their CV, job posting URL, and email. It provides an interactive HTML form for uploads and returns an immediate response.

**Nodes Involved:**  
- Webhook - CV Optimizer Form  
- Webhook Response - HTML Form

**Node Details:**  

- **Webhook - CV Optimizer Form**  
  - *Type:* Webhook  
  - *Role:* Entry point to receive HTTP requests (POST with multipart form data) containing CV PDF, job posting URL, and email.  
  - *Configuration:*  
    - Path: `cv-optimizer`  
    - Accepts multiple HTTP methods (GET, POST)  
    - Responds with a node (Webhook Response) to serve the HTML form.  
  - *Expressions:* None critical here; uses default webhook functionality.  
  - *Connections:* Outputs to both "Webhook Response - HTML Form" and "Extract CV Text (PDF)".  
  - *Edge Cases:* Invalid HTTP methods, malformed form data, missing required fields, file upload errors.  

- **Webhook Response - HTML Form**  
  - *Type:* Respond to Webhook  
  - *Role:* Serves a styled HTML form to the user for uploading CV, entering job URL and email.  
  - *Configuration:*  
    - Responds with HTML content embedded directly in the node parameters.  
    - Form uses client-side JavaScript to POST data back to the webhook URL.  
  - *Key Features:*  
    - Accepts PDF files only for CV upload.  
    - Validates required fields on client side.  
    - Displays status messages based on submission success or failure.  
  - *Connections:* Input from Webhook node, no outputs.  
  - *Edge Cases:* Browser incompatibility, JavaScript disabled, network errors during submission.  

---

#### 1.2 CV Text Extraction & Preparation

**Overview:**  
Extracts raw text content from the uploaded PDF CV and prepares it as a plain text string for AI analysis.

**Nodes Involved:**  
- Extract CV Text (PDF)  
- Prepare CV Text

**Node Details:**  

- **Extract CV Text (PDF)**  
  - *Type:* Extract from File  
  - *Role:* Converts the binary PDF upload to raw text.  
  - *Configuration:*  
    - Operation: PDF extraction  
    - Binary Property Name: references the uploaded file property named `cv` from the webhook.  
  - *Connections:* Input from webhook, output to "Prepare CV Text".  
  - *Edge Cases:* Corrupted or encrypted PDFs, empty files, extraction failures.  

- **Prepare CV Text**  
  - *Type:* Set  
  - *Role:* Extracts the text from the previous node's output JSON and assigns it to a new field `cv_text`.  
  - *Configuration:*  
    - Assigns `cv_text` string with the extracted text content from the PDF.  
  - *Connections:* Input from "Extract CV Text (PDF)", outputs to "Fetch Job Posting".  
  - *Edge Cases:* Empty or null text extraction results, data format mismatch.  

---

#### 1.3 Job Posting Retrieval and Cleaning

**Overview:**  
Fetches the job posting's HTML page using the provided URL and cleans the HTML to extract readable plain text, removing scripts and styles.

**Nodes Involved:**  
- Fetch Job Posting  
- Job Text Cleaner

**Node Details:**  

- **Fetch Job Posting**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the raw HTML content of the job posting page.  
  - *Configuration:*  
    - URL is dynamically taken from the webhook form field `job_url`.  
    - Default HTTP GET method, no special headers.  
  - *Connections:* Input from "Prepare CV Text" node, output to "Job Text Cleaner".  
  - *Edge Cases:* Invalid URL, 404 or 5xx HTTP errors, slow responses/timeouts, redirects, blocked by CORS or anti-bot measures.  

- **Job Text Cleaner**  
  - *Type:* Code (JavaScript)  
  - *Role:* Cleans the fetched HTML to remove scripts, styles, and tags, resulting in plain readable text.  
  - *Configuration:*  
    - Uses regular expressions to remove `<script>`, `<style>`, and all HTML tags, then normalizes whitespace.  
    - Outputs a single JSON field `job_text`.  
  - *Connections:* Input from "Fetch Job Posting", output to "Merge CV + Job Data".  
  - *Edge Cases:* Extremely large pages causing performance issues, incomplete cleaning if HTML is malformed.  

---

#### 1.4 Data Merging

**Overview:**  
Combines the prepared CV text and the cleaned job posting text into a single data structure to feed the AI analysis node.

**Nodes Involved:**  
- Merge CV + Job Data

**Node Details:**  

- **Merge CV + Job Data**  
  - *Type:* Merge  
  - *Role:* Combines two inputs into one output by position, merging CV text and job text into a single JSON object.  
  - *Configuration:*  
    - Mode: Combine (by position)  
  - *Connections:*  
    - Inputs: From "Prepare CV Text" (CV side) and "Job Text Cleaner" (job description side)  
    - Output: To "AI CV Analyzer" node  
  - *Edge Cases:* Mismatched input counts (e.g., one input missing), empty texts.  

---

#### 1.5 AI Analysis

**Overview:**  
The core AI processing block that compares CV and job posting texts using Google Gemini/PaLM models to generate a detailed structured analysis, including fit scores, skill matches, missing critical skills, recommendations, and personalized advice.

**Nodes Involved:**  
- AI CV Analyzer  
- Gemini Model - Primary (fallback or secondary)  
- Gemini Model  
- Parse AI JSON Output

**Node Details:**  

- **AI CV Analyzer**  
  - *Type:* Langchain Agent  
  - *Role:* Runs a complex AI prompt to analyze CV vs job description and produce a structured JSON output.  
  - *Configuration:*  
    - Prompt includes instructions to: extract job title, location, matched and missing skills, advice, score, recommendations, and email body.  
    - Enforces strict JSON output schema to avoid parsing errors.  
    - Limits text inputs to 18,000 characters each for performance.  
    - System message instructs it to behave as a professional career assistant.  
  - *Connections:* Input from "Merge CV + Job Data", output to "Send report".  
  - *Credentials:* Google Gemini (PaLM) API account.  
  - *Edge Cases:* API rate limits, invalid JSON output, timeout, model hallucinations or misinterpretations, malformed input texts.  

- **Gemini Model - Primary & Gemini Model**  
  - *Type:* Langchain LM Chat (Google Gemini)  
  - *Role:* These nodes represent the language model calls to Google Gemini. "Gemini Model - Primary" seems connected for AI invocation within the AI CV Analyzer or as fallback.  
  - *Configuration:* No additional parameters; uses credentials.  
  - *Connections:* Feed into the AI CV Analyzer, or used internally by it.  
  - *Credentials:* Google Gemini (PaLM) API account.  
  - *Edge Cases:* Authentication failures, quota exceeded, network issues.  

- **Parse AI JSON Output**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses and auto-corrects the AI JSON output from the AI CV Analyzer to ensure valid structured data.  
  - *Configuration:*  
    - Uses a JSON schema example for validation and auto-fixing.  
  - *Connections:* Input from "Gemini Model", output to "AI CV Analyzer" node (for further processing).  
  - *Edge Cases:* Parsing errors if AI output is malformed or incomplete, schema mismatches.  

---

#### 1.6 Email Reporting

**Overview:**  
Sends a personalized email to the candidate containing the AI analysis results, including fit summary, gaps, score, recommendations, and tips for CV improvement.

**Nodes Involved:**  
- Send report

**Node Details:**  

- **Send report**  
  - *Type:* Gmail (OAuth2)  
  - *Role:* Sends an email report to the candidate’s submitted email address.  
  - *Configuration:*  
    - Sends to email from webhook form field.  
    - Email subject and message body dynamically constructed using the AI analysis output fields (job title, location, fit summary, missing skills, fit score, recommendation, CV optimization tips, final advice).  
  - *Credentials:* Gmail OAuth2 account (required).  
  - *Connections:* Input from "AI CV Analyzer".  
  - *Edge Cases:* Email sending failures, invalid email addresses, OAuth token expiration, quota limits.  

---

### 3. Summary Table

| Node Name                  | Node Type                                    | Functional Role                     | Input Node(s)                        | Output Node(s)                  | Sticky Note                                                                                              |
|----------------------------|----------------------------------------------|-----------------------------------|------------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook - CV Optimizer Form| Webhook                                      | Receive CV, job URL, email input  | None                               | Webhook Response - HTML Form, Extract CV Text (PDF) |                                                                                                        |
| Webhook Response - HTML Form| Respond to Webhook                          | Serve HTML form to users           | Webhook - CV Optimizer Form         | None                           |                                                                                                        |
| Extract CV Text (PDF)       | Extract from File                            | Extract text from uploaded CV PDF | Webhook - CV Optimizer Form         | Prepare CV Text                |                                                                                                        |
| Prepare CV Text             | Set                                          | Prepare extracted CV text for AI  | Extract CV Text (PDF)               | Fetch Job Posting              |                                                                                                        |
| Fetch Job Posting           | HTTP Request                                | Download job posting HTML page    | Prepare CV Text                    | Job Text Cleaner              |                                                                                                        |
| Job Text Cleaner            | Code                                         | Clean job posting HTML to text    | Fetch Job Posting                  | Merge CV + Job Data            |                                                                                                        |
| Merge CV + Job Data         | Merge                                        | Combine CV text and job text      | Prepare CV Text, Job Text Cleaner  | AI CV Analyzer                |                                                                                                        |
| AI CV Analyzer              | Langchain Agent                              | Analyze CV vs job posting via AI  | Merge CV + Job Data                | Send report                   | Sticky Note1: "### AI - Compare CV with Job\nYou can adjust the AI Agent prompt for output schema, scoring, or language." |
| Gemini Model - Primary      | Langchain LM Chat (Google Gemini)            | AI language model call             | (Internal to AI CV Analyzer)       | Parse AI JSON Output          | Sticky Note2: "###  Gemini / AI \nCredentials:  \nUse **Google Gemini/PaLM** credential."               |
| Gemini Model               | Langchain LM Chat (Google Gemini)            | AI language model call             | Parse AI JSON Output               | AI CV Analyzer                | Sticky Note2 (same as above)                                                                           |
| Parse AI JSON Output        | Langchain Output Parser (Structured JSON)    | Parse AI JSON output               | Gemini Model                      | AI CV Analyzer                |                                                                                                        |
| Send report                | Gmail (OAuth2)                               | Send email report to candidate    | AI CV Analyzer                    | None                          | Sticky Note5: "### Send report \nSend Email\nCredentials: Use **Gmail OAuth2** credential. (required)." |
| Sticky Note                | Sticky Note                                  | Documentation and overview        | None                             | None                          | "## AI CV Optimizer: Match Your CV to Job Descriptions with AI\n\nThis workflow uses AI to automatically analyze a candidate’s CV against any job posting. It extracts key skills, requirements, and gaps, then generates a clear fit summary, recommendations, and optimization tips. Candidates also receive a structured email report, helping them improve their CV and focus on the right roles.\n\nNo more guesswork, the workflow delivers objective. \n### AI-powered career insights in minutes." |
| Sticky Note1               | Sticky Note                                  | AI prompt and comparison note     | None                             | None                          | "### AI - Compare CV with Job\n\nYou can adjust the AI Agent prompt for output schema, scoring, or language." |
| Sticky Note2               | Sticky Note                                  | Gemini/AI credentials reminder    | None                             | None                          | "###  Gemini / AI \nCredentials:  \nUse **Google Gemini/PaLM** credential."                             |
| Sticky Note3               | Sticky Note                                  | Customization checklist           | None                             | None                          | "## Customization checklist\n✅ Update Webhook URL\n✅ Configure Google Gemini credentials\n✅ Set Gmail OAuth2 credentials\n✅ Adjust AI prompt if schema changes" |
| Sticky Note5               | Sticky Note                                  | Email send credentials note       | None                             | None                          | "### Send report \nSend Email\nCredentials: Use **Gmail OAuth2** credential. (required)."               |
| Sticky Note11              | Sticky Note                                  | Form submission instructions      | None                             | None                          | "### Submission form\n**POST (required)**\n- `Upload CV`\n- `Job Link` \n- `email` "                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node**  
   - Type: Webhook  
   - Name: "Webhook - CV Optimizer Form"  
   - Path: `cv-optimizer`  
   - Accept multiple methods (GET, POST)  
   - Response Mode: "Response Node" (to serve HTML form)  

2. **Create Respond to Webhook node**  
   - Type: Respond to Webhook  
   - Name: "Webhook Response - HTML Form"  
   - Connect input from Webhook node  
   - Set response content type to `text/html`  
   - Paste the provided HTML form code with embedded CSS and JavaScript for UI and submission handling  

3. **Create Extract from File node**  
   - Type: Extract from File  
   - Name: "Extract CV Text (PDF)"  
   - Operation: PDF  
   - Binary Property Name: `cv` (matches the file upload form field)  
   - Connect input from the Webhook node (POST output)  

4. **Create Set node**  
   - Type: Set  
   - Name: "Prepare CV Text"  
   - Add a string field `cv_text`  
   - Assign value: `={{ $json.text }}` (extracts text from previous node)  
   - Connect input from "Extract CV Text (PDF)"  

5. **Create HTTP Request node**  
   - Type: HTTP Request  
   - Name: "Fetch Job Posting"  
   - URL: `={{ $(' Webhook - CV Optimizer Form').item.json.body.job_url }}` (dynamic from webhook form input)  
   - Method: GET (default)  
   - Connect input from "Prepare CV Text"  

6. **Create Code node**  
   - Type: Code (JavaScript)  
   - Name: "Job Text Cleaner"  
   - Paste code snippet that removes `<script>`, `<style>`, and HTML tags, normalizes spaces, outputs field `job_text`  
   - Connect input from "Fetch Job Posting"  

7. **Create Merge node**  
   - Type: Merge  
   - Name: "Merge CV + Job Data"  
   - Mode: Combine  
   - Combine By: Position  
   - Connect inputs from "Prepare CV Text" (CV text) and "Job Text Cleaner" (job text)  

8. **Create Langchain Agent node**  
   - Type: Langchain Agent  
   - Name: "AI CV Analyzer"  
   - Prompt: Use the provided multi-step prompt instructing AI to analyze CV and job, produce JSON with fields: job_title, location, fit_score, recommendation, matched_skills, missing_critical, advice, cv_optimization, email_body, final_recommendation  
   - Enable output parser with JSON schema example  
   - Limit CV and job text inputs to 18,000 characters each  
   - Connect input from "Merge CV + Job Data"  

9. **Create Langchain LM Chat nodes** (optional, if using separate model nodes)  
   - Name: "Gemini Model - Primary" and "Gemini Model"  
   - Type: Langchain LM Chat (Google Gemini)  
   - Assign Google Gemini/PaLM API credential  
   - Connect model nodes as per AI CV Analyzer internal usage or fallback  

10. **Create Langchain Output Parser node**  
    - Type: Structured Output Parser  
    - Name: "Parse AI JSON Output"  
    - Provide JSON schema example for validation and auto-fix  
    - Connect input from "Gemini Model" node  
    - Connect output back to "AI CV Analyzer" if required by architecture  

11. **Create Gmail node**  
    - Type: Gmail (OAuth2)  
    - Name: "Send report"  
    - Credentials: Configure Gmail OAuth2 credentials  
    - Send To: `={{ $(' Webhook - CV Optimizer Form').item.json.body.email }}` (dynamic)  
    - Subject: `=Your CV Review: {{ $json.output.job_title }} in {{ $json.output.location }}`  
    - Message: Compose HTML email body using fields from AI output (fit summary, gaps, score, recommendations, tips, final advice)  
    - Connect input from "AI CV Analyzer"  

12. **Sticky Notes (Optional for documentation):**  
    - Add sticky notes to document AI prompt details, credential requirements, form submission instructions, and customization checklist as per original workflow.  

13. **Credential Setup:**  
    - Google Gemini/PaLM API credential for AI nodes.  
    - Gmail OAuth2 credentials for sending emails.  

14. **Activate and Test:**  
    - Deploy the workflow.  
    - Access the webhook URL to verify the form loads.  
    - Submit a test CV PDF, job URL, and email.  
    - Confirm AI analysis runs and email is received with the report.  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| AI CV Optimizer provides objective career insights in minutes by automatically matching CVs with job postings using Google Gemini AI. | Overview sticky note in workflow.                                                              |
| Ensure to update the webhook URL when deploying on a new instance.                                           | Sticky Note3 - Customization checklist.                                                        |
| Google Gemini/PaLM API credentials are mandatory for AI nodes.                                               | Sticky Note2 - Credential reminder.                                                           |
| Gmail OAuth2 credentials are required for the email sending node.                                            | Sticky Note5 - Email send credentials reminder.                                                |
| The HTML form served is styled and uses JavaScript to provide user feedback during submission.               | Webhook Response - HTML Form node content.                                                    |
| AI prompt and JSON schema are customizable to adjust output language, scoring, or additional fields.        | Sticky Note1 - AI prompt customization advice.                                                |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.