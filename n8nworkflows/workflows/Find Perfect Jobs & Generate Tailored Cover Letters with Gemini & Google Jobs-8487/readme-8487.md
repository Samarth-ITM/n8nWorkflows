Find Perfect Jobs & Generate Tailored Cover Letters with Gemini & Google Jobs

https://n8nworkflows.xyz/workflows/find-perfect-jobs---generate-tailored-cover-letters-with-gemini---google-jobs-8487


# Find Perfect Jobs & Generate Tailored Cover Letters with Gemini & Google Jobs

### 1. Workflow Overview

This workflow, titled **"Find Perfect Jobs & Generate Tailored Cover Letters with Gemini & Google Jobs"**, automates the process of job hunting and application letter generation using AI and job search APIs. It targets job seekers who want personalized job matches based on their CV and preferences, and automatically receives the results via email with a professional HTML template.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Captures user inputs including a CV PDF upload and job preferences via a form.
- **1.2 CV Extraction:** Extracts raw text content from the uploaded PDF CV.
- **1.3 Data Preparation:** Normalizes and organizes user inputs and extracted CV text into structured JSON variables.
- **1.4 Job Hunter Agent (AI Processing):** Uses Google Gemini LLM combined with SerpAPI to analyze the CV, search for the best job match on Google Jobs, and generate a structured JSON with job details and a tailored cover letter.
- **1.5 Output Parsing:** Cleans and parses the raw AI JSON output to ensure validity.
- **1.6 Email Generation:** Constructs a styled HTML email embedding the job match, cover letter, and application tips.
- **1.7 Email Sending:** Sends the generated email to the user’s specified email address via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block triggers the workflow when a user submits a form containing their CV (PDF) and job preferences.
- **Nodes Involved:**  
  - `On form submission`

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Starts workflow upon form submission  
    - *Configuration:*  
      - Form titled "🎯 Smart Job Hunter + Cover Letter Generator"  
      - Form fields: CV/Resume (PDF, required), Preferred Location (required), Job Type (dropdown, required), Work Setting (dropdown, required), Minimum Salary Expectation (optional), Additional Preferences (optional), Email (required)  
      - Webhook ID: `smart-job-hunter-v2`  
    - *Inputs:* External user submission  
    - *Outputs:* Data including file (CV), text fields  
    - *Edge cases:* Missing required fields, invalid email format, file upload issues  
    - *Version:* 2.3

#### 2.2 CV Extraction

- **Overview:** Extracts textual content from the uploaded PDF CV to be used by the AI agent.
- **Nodes Involved:**  
  - `Extract CV from PDF`

- **Node Details:**

  - **Extract CV from PDF**  
    - *Type:* Extract From File  
    - *Role:* Extract plain text from the PDF file contained in binary property `CV_Resume__PDF_`  
    - *Configuration:* Operation set to `pdf`, target binary property is the uploaded CV  
    - *Inputs:* Binary PDF from form submission  
    - *Outputs:* Extracted text in JSON `text` property  
    - *Edge cases:* Corrupted or scanned PDFs may fail extraction, empty or very short text output  
    - *Version:* 1

#### 2.3 Data Preparation

- **Overview:** Normalizes form fields and extracted CV text into JSON variables, preparing input for the AI agent.
- **Nodes Involved:**  
  - `Prepare Data`

- **Node Details:**

  - **Prepare Data**  
    - *Type:* Set Node  
    - *Role:* Assigns extracted CV text and form inputs to named JSON fields for clarity  
    - *Configuration:*  
      - `cvData` ← extracted text  
      - `location`, `workSetting`, `jobType`, `salaryExpectation`, `additionalPreferences`, `userEmail` ← corresponding form fields  
      - Uses expressions referencing previous nodes  
      - Default values for optional fields: "Not specified" or "None"  
    - *Inputs:* Extracted CV text, form submission JSON  
    - *Outputs:* Structured JSON variables for AI use  
    - *Edge cases:* Missing optional fields handled gracefully with defaults  
    - *Version:* 3.4

#### 2.4 Job Hunter Agent (AI Processing)

- **Overview:** Core AI logic that analyzes the CV, performs a job search, and generates a personalized cover letter and job info structured as JSON.
- **Nodes Involved:**  
  - `Job Hunter Agent`  
  - `SerpAPI` (as AI tool)  
  - `Gemini 2.5 Pro` (as AI language model)

- **Node Details:**

  - **Job Hunter Agent**  
    - *Type:* Langchain Agent Node  
    - *Role:* Orchestrates AI tasks combining LLM and external tools (SerpAPI)  
    - *Configuration:*  
      - Prompt includes CV content and user preferences  
      - System message instructs: find exactly 1 best matching job, output strict JSON with candidate info, job details, cover letter, and application tips  
      - Uses `Gemini 2.5 Pro` as language model  
      - Uses `SerpAPI` for live Google Jobs search  
    - *Inputs:* Normalized data from `Prepare Data`  
    - *Outputs:* Raw AI JSON output string in `output` field  
    - *Version:* 2.2  
    - *Edge cases:*  
      - AI may fail to return valid JSON → fallback handled downstream  
      - API key limits or downtime from SerpAPI or Gemini  
      - Timeout or latency in AI response  
    - *Sub-workflow:* Integrates SerpAPI and Gemini nodes internally

  - **SerpAPI**  
    - *Type:* Langchain Tool Node for SerpAPI  
    - *Role:* Performs Google Jobs search as part of the agent’s toolset  
    - *Configuration:* Requires SerpAPI credential  
    - *Inputs:* Query from agent node  
    - *Outputs:* Search results JSON passed back to agent  
    - *Edge cases:* API quota exceeded, invalid API key, network errors

  - **Gemini 2.5 Pro**  
    - *Type:* Langchain Language Model Node (Google Gemini)  
    - *Role:* Processes prompts and generates AI text responses  
    - *Configuration:* Model set to `models/gemini-2.5-pro`  
    - *Inputs:* Prompt from agent node  
    - *Outputs:* Raw AI text response  
    - *Credential:* Google Palm / Gemini API key required  
    - *Edge cases:* Authentication errors, model unavailability, API throttling

#### 2.5 Output Parsing

- **Overview:** Cleans the raw AI output string and parses it into JSON for subsequent processing.
- **Nodes Involved:**  
  - `Parse Agent Output`

- **Node Details:**

  - **Parse Agent Output**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Removes markdown code fences and parses JSON safely  
    - *Logic:*  
      - Removes ```json and ``` markers  
      - Tries JSON.parse; on failure returns an error JSON with raw output  
    - *Inputs:* AI raw output string from `Job Hunter Agent`  
    - *Outputs:* Parsed JSON object or error object  
    - *Edge cases:* Malformed AI output, missing fields handled by error object  
    - *Version:* 2

#### 2.6 Email Generation

- **Overview:** Builds a polished HTML email embedding candidate profile, job match, cover letter, and application tips.
- **Nodes Involved:**  
  - `Generate HTML Email`

- **Node Details:**

  - **Generate HTML Email**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Generates responsive and branded HTML email content using parsed AI JSON data  
    - *Logic:*  
      - Extracts candidate and job info from JSON input  
      - Constructs HTML with inline styles and semantic structure  
      - Includes header, profile summary, job match details, cover letter, application tips, and footer with branding  
      - Handles missing job gracefully with "No job found." message  
    - *Inputs:* Parsed JSON from `Parse Agent Output` and user email from `Prepare Data`  
    - *Outputs:* Object containing `htmlEmail` (HTML string), `userEmail`, and full job data JSON  
    - *Edge cases:* Missing or incomplete job data results in fallback message  
    - *Version:* 2

#### 2.7 Email Sending

- **Overview:** Sends the generated email to the user’s provided email address via Gmail OAuth2.
- **Nodes Involved:**  
  - `Send a message`

- **Node Details:**

  - **Send a message**  
    - *Type:* Gmail Node  
    - *Role:* Sends the crafted email message  
    - *Configuration:*  
      - Recipient: email from form submission (`$('On form submission').item.json.Email`)  
      - Subject: "Your Job MATCH!"  
      - Message body: HTML email from previous node (`$json.htmlEmail`)  
      - Option to disable Gmail attribution appended to email  
      - Uses OAuth2 credential configured as `contactmuhtadin`  
    - *Inputs:* HTML email content and recipient email  
    - *Outputs:* Gmail send response  
    - *Edge cases:* OAuth token expiration, sending limits, invalid email address  
    - *Version:* 2.1

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                         | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                                                                                                                                                                               |
|-----------------------|----------------------------------|---------------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger                     | User input reception                   | -                        | Extract CV from PDF      | ## ⚙️ SETUP GUIDE (Updated) <br> 1. Google Gemini API key setup <br> 2. SerpAPI key setup <br> 3. Form webhook auto-generated                                                                                                                                                              |
| Extract CV from PDF    | Extract From File                | Extract text from uploaded CV PDF     | On form submission       | Prepare Data            | ## 🎯 SMART JOB HUNTER + COVER LETTER GENERATOR <br> Extract CV text to `cvData`                                                                                                                                                                                                          |
| Prepare Data           | Set Node                        | Normalize and structure user inputs   | Extract CV from PDF       | Job Hunter Agent        | ## 🎯 SMART JOB HUNTER + COVER LETTER GENERATOR <br> Normalize form fields and extracted CV text                                                                                                                                                                                         |
| SerpAPI                | Langchain Tool (SerpAPI)         | Search Google Jobs                    | Job Hunter Agent (ai_tool) | Job Hunter Agent        | ## ⚙️ SETUP GUIDE (Updated) <br> Requires SerpAPI API key credential                                                                                                                                                                                                                      |
| Gemini 2.5 Pro         | Langchain LLM (Google Gemini)    | AI language model for text generation | Job Hunter Agent (ai_languageModel) | Job Hunter Agent        | ## ⚙️ SETUP GUIDE (Updated) <br> Requires Google Gemini API key credential                                                                                                                                                                                                                |
| Job Hunter Agent       | Langchain Agent Node             | Analyze CV, find job, generate letter | Prepare Data, SerpAPI, Gemini 2.5 Pro | Parse Agent Output      | ## 🎯 SMART JOB HUNTER + COVER LETTER GENERATOR <br> Enforces exactly 1 job result, outputs structured JSON                                                                                                                                                                               |
| Parse Agent Output     | Code Node (JavaScript)           | Clean and parse AI JSON output        | Job Hunter Agent          | Generate HTML Email     | ## ⚠️ Known Limitations <br> If AI output invalid JSON, fallback error object shown <br> Email will show "No job found"                                                                                                                                                                    |
| Generate HTML Email    | Code Node (JavaScript)           | Build styled HTML email                | Parse Agent Output        | Send a message          | ## 🎯 SMART JOB HUNTER + COVER LETTER GENERATOR <br> Embeds candidate info, job match, cover letter, and tips in email                                                                                                                                                                     |
| Send a message         | Gmail Node                      | Send email to user                     | Generate HTML Email       | -                       | ## ⚠️ Known Limitations <br> Requires Gmail OAuth2 credential <br> Ensure token refresh and valid email address                                                                                                                                                                           |
| Sticky Note - How It Works | Sticky Note                    | Overview and workflow explanation      | -                        | -                       | Describes the end-to-end workflow steps and logic                                                                                                                                                                                                                                        |
| Sticky Note - Setup Guide | Sticky Note                    | Setup instructions for API keys and form | -                        | -                       | Provides API and form configuration instructions with links: [Google AI Studio](https://aistudio.google.com/), [SerpAPI Dashboard](https://serpapi.com/)                                                                                                                                  |
| Sticky Note - Pro Tips | Sticky Note                     | Customization tips                     | -                        | -                       | Suggestions for prompt tuning, email branding, job filters, LLM model alternatives, and optional attachments                                                                                                                                                                             |
| Sticky Note - How It Works1 | Sticky Note                 | Known limitations                      | -                        | -                       | Notes on job source limitations, single result count, JSON parsing fallback, and Gmail API requirements                                                                                                                                                                                  |
| Sticky Note            | Sticky Note                     | Visual reference image 1               | -                        | -                       | ![Inbox 1](https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/Website/%20JOB%20Hunter/inbox1.PNG)                                                                                                                                                                             |
| Sticky Note1           | Sticky Note                     | Visual reference image 2               | -                        | -                       | ![Inbox 2](https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/Website/%20JOB%20Hunter/inbox%202.PNG)                                                                                                                                                                           |
| Sticky Note2           | Sticky Note                     | Visual reference image 3               | -                        | -                       | ![Inbox 3](https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/Website/%20JOB%20Hunter/inbox3.PNG)                                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `On form submission`  
   - Type: Form Trigger  
   - Configure form fields:  
     - CV/Resume (PDF, required, accept `.pdf`)  
     - Preferred Location (text, required)  
     - Job Type (dropdown: Full Time, Part Time, Freelance, Contract, Internship)  
     - Work Setting (dropdown: On Site, Remote, Hybrid)  
     - Minimum Salary Expectation (optional text)  
     - Additional Preferences (optional textarea)  
     - Email (email field, required)  
   - Save and note the webhook URL for form submission.

2. **Add Extract From File Node**  
   - Name: `Extract CV from PDF`  
   - Type: Extract From File  
   - Operation: `pdf`  
   - Binary Property Name: The binary property containing the uploaded PDF file from the form (e.g., `CV_Resume__PDF_`)  
   - Connect output of `On form submission` to input of this node.

3. **Add Set Node for Data Preparation**  
   - Name: `Prepare Data`  
   - Type: Set Node  
   - Assign the following fields using expressions:  
     - `cvData` = extracted text from previous node (`{{$json["text"]}}`)  
     - `location` = `$('On form submission').item.json['Preferred Location']`  
     - `workSetting` = `$('On form submission').item.json['Work Setting']`  
     - `jobType` = `$('On form submission').item.json['Job Type']`  
     - `salaryExpectation` = `$('On form submission').item.json['Minimum Salary Expectation (Optional)'] || 'Not specified'`  
     - `additionalPreferences` = `$('On form submission').item.json['Additional Preferences (Optional)'] || 'None'`  
     - `userEmail` = `$('On form submission').item.json['Email']`  
   - Connect output of `Extract CV from PDF` to this node.

4. **Setup API Credentials**  
   - Create and add credentials for:  
     - Google Gemini / Google Palm API with access to Gemini 2.5 Pro model  
     - SerpAPI with key for Google Jobs search  
     - Gmail OAuth2 for sending emails (refresh tokens and scopes configured)

5. **Add Langchain Tool Node for SerpAPI**  
   - Name: `SerpAPI`  
   - Type: Langchain Tool (SerpAPI)  
   - Assign SerpAPI credentials  
   - Leave options default

6. **Add Langchain Language Model Node for Google Gemini**  
   - Name: `Gemini 2.5 Pro`  
   - Type: Langchain LLM (Google Gemini)  
   - Model: `models/gemini-2.5-pro`  
   - Assign Google Palm API credentials

7. **Add Langchain Agent Node**  
   - Name: `Job Hunter Agent`  
   - Type: Langchain Agent Node  
   - Configure prompt text combining CV content and user preferences with tasks:  
     - Extract CV info  
     - Search for exactly 1 best matching job using SerpAPI  
     - Generate personalized cover letter (250–350 words)  
     - Output structured JSON as per documented schema  
   - Set system message instructing strict single job output and JSON format  
   - Link AI language model input to `Gemini 2.5 Pro` node  
   - Link AI tool input to `SerpAPI` node  
   - Connect input from `Prepare Data` node

8. **Add Code Node to Parse Agent Output**  
   - Name: `Parse Agent Output`  
   - JavaScript code to:  
     - Remove markdown code fences (```json and ```) from raw AI output string (`$json.output`)  
     - Safely parse JSON; if fails, output error object with raw text  
   - Connect output from `Job Hunter Agent` to this node

9. **Add Code Node for Email Generation**  
   - Name: `Generate HTML Email`  
   - JavaScript code to build full HTML email template embedding:  
     - Candidate profile summary from AI JSON  
     - Job match details with styling and "Apply Now" button  
     - Personalized cover letter text  
     - Application tips section  
     - Footer with branding and current date  
   - Use expressions to get userEmail from `Prepare Data` node and job data from parsed JSON  
   - Connect output from `Parse Agent Output` to this node

10. **Add Gmail Node to Send Email**  
    - Name: `Send a message`  
    - Type: Gmail  
    - Credentials: Gmail OAuth2 credentials created earlier  
    - Set "Send To" as user’s email from form submission (`$('On form submission').item.json.Email`)  
    - Subject: "Your Job MATCH!"  
    - Message: HTML content from `Generate HTML Email` node (`$json.htmlEmail`)  
    - Disable Gmail attribution append  
    - Connect output from `Generate HTML Email` node

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow integrates Google Gemini 2.5 Pro LLM with SerpAPI for live Google Jobs search and cover letter generation.                       | Setup Guide sticky note with detailed API key creation and credential setup                        |
| Single job result per run enforced by system prompt; modify prompt to allow multiple job matches if desired.                              | Known Limitations sticky note                                                                       |
| Email template includes branding: "Generated by khmuhtadin.com" with current date footer.                                                  | Customize footer text in `Generate HTML Email` node                                                |
| Prompt and email template are customizable to adjust tone, length, and branding for user needs.                                           | Pro Tips sticky note                                                                               |
| Workflow requires valid Gmail OAuth2 credentials with refresh token for email sending; ensure token validity to avoid failures.          | Known Limitations sticky note                                                                       |
| Visual references for email output are available online: Inbox 1, 2, and 3 screenshots linked in sticky notes for design inspiration.     | Sticky Note images with URLs provided                                                              |
| Form webhook URL is auto-generated by n8n; share with users or embed in websites for data collection.                                    | Setup Guide sticky note                                                                            |

---

This detailed reference enables full understanding, modification, and reproduction of the "JOB Hunter Agent" workflow designed to smartly match jobs and generate tailored cover letters automatically.