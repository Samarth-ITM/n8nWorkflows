Screen Applicants With AI, notify HR and save them in a Google Sheet

https://n8nworkflows.xyz/workflows/screen-applicants-with-ai--notify-hr-and-save-them-in-a-google-sheet-2632


# Screen Applicants With AI, notify HR and save them in a Google Sheet

### 1. Workflow Overview

This workflow automates the process of screening job applicants’ CVs using AI, storing their compatibility ratings in Google Sheets, and notifying both HR and candidates via email. It is designed for HR teams to streamline recruitment by integrating form submissions, AI-powered analysis, data storage, and communication.

Logical blocks:

- **1.1 Input Reception:** Candidate submits details and CV through a web form, triggering the workflow.
- **1.2 PDF Extraction:** Extract text content from the uploaded PDF CV.
- **1.3 AI Processing & Rating:** Use AI to analyze the extracted CV text against a job description and generate a compatibility rating and recommendation.
- **1.4 Data Storage:** Append candidate details and AI rating to a Google Sheet.
- **1.5 Notifications:** Send confirmation email to candidate and notify HR about the new CV and rating.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures candidate application data and CV file submission via a web form, serving as the workflow’s entry point.

**Nodes Involved:**  
- Application Form

**Node Details:**

- **Application Form**  
  - Type: Form Trigger  
  - Role: Captures candidate submitted data and PDF CV file from a custom web form titled "Application for Software Engineer Position".  
  - Configuration: Requires fields Full Name, E-mail, Expectation (salary), Linkedin, and a PDF upload for "Your Resume/CV".  
  - Expressions: Outputs form data as JSON, including the binary file for CV.  
  - Input: Web form submission triggers the workflow.  
  - Output: Passes captured form JSON and binary CV file to next node.  
  - Edge Cases: Form validation ensures required fields; file upload must be PDF; failure to upload or malformed file may cause errors.

#### 1.2 PDF Extraction

**Overview:**  
Extracts raw text content from the candidate’s uploaded PDF CV to prepare for AI analysis.

**Nodes Involved:**  
- Convert Binary to Json

**Node Details:**

- **Convert Binary to Json**  
  - Type: Extract From File (PDF extraction)  
  - Role: Extract text content from the binary PDF file uploaded in the form.  
  - Configuration: Operates on binary property "Your_Resume_CV" which holds the uploaded PDF file.  
  - Input: Receives binary PDF from Application Form node.  
  - Output: Produces extracted text in JSON format for AI processing.  
  - Edge Cases: PDF extraction may fail due to corrupted files or unsupported formats; no retry enabled.

#### 1.3 AI Processing & Rating

**Overview:**  
Uses Google Gemini AI to analyze the extracted CV text against the job description "Software Engineer" and generate a compatibility rating and recommendation.

**Nodes Involved:**  
- Google Gemini Chat Model (Google PaLM API)  
- Using AI Analysis & Rating

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Language Model Chat (Google Langchain integration)  
  - Role: Provides the AI engine to generate candidate compatibility analysis.  
  - Credentials: Uses Google PaLM API account.  
  - Input: Receives prompt and extracted CV text.  
  - Output: AI-generated analysis text.  
  - Edge Cases: API quota limits, authentication errors, or response timeouts possible.

- **Using AI Analysis & Rating**  
  - Type: Chain LLM Node (Langchain)  
  - Role: Constructs the prompt sent to Google Gemini model with detailed instructions to rate candidate compatibility on a 1-10 scale and provide a recommendation, strictly in a specified output language.  
  - Configuration: Text prompt dynamically includes the extracted CV text, job description ("Software Engineer"), and output language variable.  
  - Input: Receives extracted text from PDF extraction node.  
  - Output: AI text analysis with rating and recommendation.  
  - Edge Cases: Expression evaluation errors if variables undefined; AI output may be inconsistent if prompt misunderstood.

#### 1.4 Data Storage

**Overview:**  
Appends the candidate’s submitted details and AI compatibility rating to a Google Sheet for record-keeping and HR review.

**Nodes Involved:**  
- Candidate Lists

**Node Details:**

- **Candidate Lists**  
  - Type: Google Sheets (append data)  
  - Role: Stores candidate data including CV filename, contact info, LinkedIn, salary expectation, and AI rating into a designated Google Sheet.  
  - Configuration: Maps form fields and AI analysis text to columns in the sheet titled "CV of Software Engineers" (sheet gid=0).  
  - Credentials: Uses Google Sheets OAuth2 account.  
  - Input: Receives AI rating and candidate data from AI node and form node.  
  - Output: Passes data forward to notification node.  
  - Edge Cases: Google Sheets API errors, permission issues, or rate limiting.

#### 1.5 Notifications

**Overview:**  
Sends automated emails confirming CV receipt to candidates and notifying HR with candidate details and AI rating.

**Nodes Involved:**  
- Inform HR New CV Received  
- Confirmation of CV Submission

**Node Details:**

- **Inform HR New CV Received**  
  - Type: Gmail Send  
  - Role: Sends an email to HR with candidate details including AI rating and LinkedIn profile.  
  - Configuration: Sends to fixed HR email address; email subject "New Candidate CV Awaiting Review"; message body uses expressions to insert candidate info and AI rating.  
  - Credentials: Gmail OAuth2 account.  
  - Input: Comes after Google Sheets append node, includes candidate and rating info.  
  - Output: Triggers candidate confirmation email.  
  - Edge Cases: Email sending failures, invalid email address, or service outages.

- **Confirmation of CV Submission**  
  - Type: Gmail Send  
  - Role: Sends a confirmation email to the candidate acknowledging receipt of their CV.  
  - Configuration: Sends to candidate email captured from form; subject "We Have Received Your CV"; body personalized with candidate name.  
  - Credentials: Gmail OAuth2 account.  
  - Input: Triggered by HR notification node completion.  
  - Output: Ends workflow.  
  - Edge Cases: Email delivery failures, invalid candidate email.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                    | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                      |
|-----------------------------|----------------------------------|----------------------------------|----------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Application Form            | Form Trigger                     | Capture candidate data and CV     | (Trigger)                  | Convert Binary to Json           |                                                                                                 |
| Convert Binary to Json      | Extract From File (PDF)          | Extract text from PDF CV          | Application Form           | Using AI Analysis & Rating       |                                                                                                 |
| Google Gemini Chat Model    | Langchain Google PaLM API        | AI language model engine          | (AI integration)           | Using AI Analysis & Rating (ai_languageModel) |                                                                                                 |
| Using AI Analysis & Rating  | Chain LLM Node                  | Generate AI rating and recommendation | Convert Binary to Json, Google Gemini Chat Model | Candidate Lists                   |                                                                                                 |
| Candidate Lists            | Google Sheets Append             | Store candidate data and AI rating | Using AI Analysis & Rating | Inform HR New CV Received        |                                                                                                 |
| Inform HR New CV Received   | Gmail Send                      | Notify HR of new CV and rating    | Candidate Lists            | Confirmation of CV Submission    |                                                                                                 |
| Confirmation of CV Submission | Gmail Send                    | Confirm CV receipt to candidate   | Inform HR New CV Received  | (End)                          |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: Application Form  
   - Configure form with fields:  
     - Full Name (required)  
     - E-mail (required)  
     - Expectation (required, placeholder "2000-3000$")  
     - Linkedin (required)  
     - Your Resume/CV (required, file upload, accept only `.pdf`)  
   - Save and note webhook URL to embed in your form.

2. **Add Extract From File node**  
   - Name: Convert Binary to Json  
   - Operation: PDF extraction  
   - Binary Property: "Your_Resume_CV" (matches uploaded file from form)  
   - Connect Application Form → Convert Binary to Json.

3. **Add Google Gemini Chat Model node**  
   - Name: Google Gemini Chat Model  
   - Credentials: Link Google PaLM API account  
   - Set no options, will be called by next node.

4. **Add Chain LLM node**  
   - Name: Using AI Analysis & Rating  
   - Type: Langchain Chain LLM  
   - Configure prompt to analyze CV text (`{{$json.text}}`) against job description "Software Engineer" with instructions:  
     - Output compatibility rating 1-10  
     - Provide recommendation  
     - Output must be in a specified language variable `${output_language}`  
     - No markdown formatting  
   - Link Google Gemini Chat Model node as language model provider in node config.  
   - Connect Convert Binary to Json → Using AI Analysis & Rating.

5. **Add Google Sheets node**  
   - Name: Candidate Lists  
   - Operation: Append  
   - Document ID: Your Google Sheet ID for storing candidates  
   - Sheet Name: Select your sheet (gid=0 or equivalent)  
   - Map columns:  
     - CV: filename of uploaded CV from form  
     - Full Name, E-mail, Expectation, Linkedin: from form fields  
     - AI Rating: from Using AI Analysis & Rating node output (`$json.text`)  
   - Credentials: Link Google Sheets OAuth2 account  
   - Connect Using AI Analysis & Rating → Candidate Lists.

6. **Add Gmail Send node**  
   - Name: Inform HR New CV Received  
   - Credentials: Gmail OAuth2 account  
   - Send To: HR email address (e.g., sarfaraz@mediusaware.com)  
   - Subject: "New Candidate CV Awaiting Review"  
   - Message body: Include candidate full name, email, LinkedIn, expectation, and AI rating using expressions from form and AI nodes.  
   - Connect Candidate Lists → Inform HR New CV Received.

7. **Add Gmail Send node**  
   - Name: Confirmation of CV Submission  
   - Credentials: Gmail OAuth2 account  
   - Send To: Candidate’s email from form  
   - Subject: "We Have Received Your CV"  
   - Message: Personalized thank you message including candidate's full name.  
   - Connect Inform HR New CV Received → Confirmation of CV Submission.

8. **Set workflow activation**  
   - Ensure all credentials are properly configured and tested.  
   - Activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Modify the email templates to reflect your organization’s branding.                                            | Email nodes (Inform HR New CV Received, Confirmation of CV Submission)                            |
| Adjust AI compatibility rating thresholds and language settings based on your recruitment policies.           | Using AI Analysis & Rating node prompt configuration                                             |
| Ensure Google Sheets and Gmail OAuth2 credentials are correctly set up with necessary permissions.             | Google Sheets and Gmail nodes                                                                      |
| Use the provided form template and webhook URL to embed the candidate submission form.                         | Application Form node                                                                              |
| Workflow designed to handle PDF CV uploads only; other formats require workflow adjustment.                    | Application Form and Convert Binary to Json nodes                                                 |
| For AI analysis, the output language must be defined as `${output_language}` to avoid workflow failure.        | Using AI Analysis & Rating prompt instructions                                                   |