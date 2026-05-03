HR Job Posting and Evaluation with AI

https://n8nworkflows.xyz/workflows/hr-job-posting-and-evaluation-with-ai-2773


# HR Job Posting and Evaluation with AI

### 1. Workflow Overview

The **HR Job Posting and Evaluation with AI** workflow automates the recruitment process for technical roles, specifically targeting Automation Specialists. It streamlines candidate application collection, resume management, AI-driven candidate evaluation, questionnaire generation, interview scheduling, and personalized communication. The workflow integrates multiple services such as Airtable, Google Drive, OpenAI, and Google Calendar to provide an end-to-end hiring solution.

The workflow is logically divided into the following blocks:

- **1.1 Candidate Application Intake and Storage**: Captures candidate submissions via a form, uploads CVs to Google Drive, and stores applicant data in Airtable.
- **1.2 AI-Powered Candidate Evaluation**: Downloads CVs, extracts text, compares resumes against job descriptions using AI, and scores candidates.
- **1.3 Candidate Shortlisting and Airtable Update**: Based on AI scores, candidates are either shortlisted or rejected, with Airtable records updated accordingly.
- **1.4 Questionnaire Generation and Response Collection**: Generates tailored interview questions using AI, presents them in a form for candidate responses, and updates Airtable with answers.
- **1.5 Personalized Email Communication and Interview Scheduling**: Crafts personalized emails inviting candidates for phone interviews, schedules meetings via Google Calendar, and updates Airtable with interview times.
- **1.6 Screening Questions Generation and Airtable Update**: Creates screening questions based on candidate data and job description, then updates Airtable for further interview stages.

---

### 2. Block-by-Block Analysis

#### 1.1 Candidate Application Intake and Storage

- **Overview:**  
  This block collects candidate information through a web form, uploads the candidate’s CV to Google Drive, and stores all relevant details in Airtable for centralized tracking.

- **Nodes Involved:**  
  - On form submission  
  - Upload CV to google drive  
  - applicant details (Set node)  
  - Airtable  

- **Node Details:**  

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point capturing candidate application data via a custom form.  
    - Configuration: Form fields include First Name, Last Name, Email, Phone, Years of Experience, and CV upload (PDF only). The form description details the job role and requirements.  
    - Inputs: Web form submission  
    - Outputs: Candidate data and uploaded CV file  
    - Edge Cases: Form validation failures, file upload errors, bot submissions (bot filtering disabled)  

  - **Upload CV to google drive**  
    - Type: Google Drive node  
    - Role: Uploads the candidate’s CV file to a specified Google Drive folder ("HR Test").  
    - Configuration: Uses OAuth2 credentials, uploads the file with original filename, targets a specific folder by ID.  
    - Inputs: Binary file from form submission  
    - Outputs: Metadata including webViewLink for the uploaded file  
    - Edge Cases: Google Drive API errors, permission issues, file size limits  

  - **applicant details**  
    - Type: Set node  
    - Role: Prepares structured candidate data for Airtable insertion.  
    - Configuration: Combines first and last name, extracts phone, email, experience, submission timestamp, and CV link from previous nodes.  
    - Inputs: Data from Google Drive upload node  
    - Outputs: JSON with normalized candidate details  
    - Edge Cases: Missing or malformed data from previous nodes  

  - **Airtable**  
    - Type: Airtable node  
    - Role: Creates a new record in the "Applicants" table of Airtable with candidate data.  
    - Configuration: Uses Airtable API token credentials, maps fields including Name, Phone, CV Link, Applying for (fixed to "Automation Specialist"), and Email address.  
    - Inputs: Structured data from Set node  
    - Outputs: Confirmation of record creation  
    - Edge Cases: Airtable API rate limits, authentication errors, schema mismatches  

---

#### 1.2 AI-Powered Candidate Evaluation

- **Overview:**  
  This block downloads the candidate’s CV from Google Drive, extracts text content, retrieves the job description from Airtable, and uses an AI agent to score the candidate’s suitability.

- **Nodes Involved:**  
  - Airtable (search Applicants)  
  - download CV  
  - Extract from File  
  - Airtable1 (search Positions)  
  - AI Agent  
  - Structured Output Parser  

- **Node Details:**  

  - **download CV**  
    - Type: Google Drive node (download operation)  
    - Role: Downloads the candidate’s CV file using the stored CV link from Airtable.  
    - Configuration: Uses OAuth2 credentials, fileId dynamically set from Airtable record.  
    - Inputs: CV Link from Airtable record  
    - Outputs: Binary file data  
    - Edge Cases: File not found, permission denied, invalid fileId  

  - **Extract from File**  
    - Type: Extract from File node  
    - Role: Extracts text content from the downloaded PDF CV.  
    - Configuration: Operation set to PDF extraction.  
    - Inputs: Binary PDF file  
    - Outputs: Extracted text content in JSON  
    - Edge Cases: Corrupted PDF, extraction failures  

  - **Airtable1**  
    - Type: Airtable Tool node (search operation)  
    - Role: Retrieves the job description and requirements from the "Positions" table in Airtable.  
    - Configuration: Uses Airtable API token credentials, searches for relevant job posting.  
    - Inputs: None (search criteria implicit or default)  
    - Outputs: Job description data  
    - Edge Cases: No matching job posting, API errors  

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Compares candidate CV text against the job description, assigns a qualification score (0 to 1) and a brief reason.  
    - Configuration: Prompt instructs to provide only score and reason in less than 20 words.  
    - Inputs: Extracted CV text and job description from Airtable1  
    - Outputs: AI-generated score and reason  
    - Edge Cases: AI API rate limits, prompt failures, unexpected output format  

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI Agent’s JSON output to extract score and reason fields reliably.  
    - Configuration: JSON schema example provided for validation.  
    - Inputs: Raw AI Agent output  
    - Outputs: Parsed structured data  
    - Edge Cases: Parsing errors if AI output deviates from schema  

---

#### 1.3 Candidate Shortlisting and Airtable Update

- **Overview:**  
  This block evaluates the AI score to determine if the candidate should be shortlisted or rejected, then updates the Airtable record accordingly.

- **Nodes Involved:**  
  - shortlisted? (If node)  
  - Potential Hire (Airtable update)  
  - Rejected (Airtable update)  

- **Node Details:**  

  - **shortlisted?**  
    - Type: If node  
    - Role: Checks if AI score is >= 0.7 to decide candidate status.  
    - Configuration: Numeric comparison on AI score field.  
    - Inputs: Parsed AI score from Structured Output Parser  
    - Outputs: Two branches - true (score >= 0.7), false (score < 0.7)  
    - Edge Cases: Missing score, non-numeric values  

  - **Potential Hire**  
    - Type: Airtable node (update operation)  
    - Role: Updates candidate record setting Stage to "Interviewing" and records AI score and notes.  
    - Configuration: Uses Airtable API token, updates by record ID.  
    - Inputs: Candidate record ID and AI evaluation data  
    - Outputs: Confirmation of update  
    - Edge Cases: Airtable update conflicts, invalid record ID  

  - **Rejected**  
    - Type: Airtable node (update operation)  
    - Role: Updates candidate record setting Stage to "No hire" with AI score and notes.  
    - Configuration: Similar to Potential Hire node but different Stage value.  
    - Inputs: Candidate record ID and AI evaluation data  
    - Outputs: Confirmation of update  
    - Edge Cases: Same as Potential Hire  

---

#### 1.4 Questionnaire Generation and Response Collection

- **Overview:**  
  For shortlisted candidates, this block generates customized interview questions using AI, presents them in a form for candidate responses, and updates Airtable with the answers.

- **Nodes Involved:**  
  - generate questionnaires (OpenAI node)  
  - questionnaires (Form node)  
  - update questionnaires (Airtable update)  

- **Node Details:**  

  - **generate questionnaires**  
    - Type: OpenAI node  
    - Role: Creates 5 interview questions based on job description and candidate CV.  
    - Configuration: Uses GPT-4O-MINI model, prompt instructs focus on projects, responsibilities, skills, problem-solving, and alignment with company values.  
    - Inputs: Job description from Airtable, candidate CV text  
    - Outputs: JSON with interview questions  
    - Edge Cases: AI API errors, prompt misinterpretation  

  - **questionnaires**  
    - Type: Form node  
    - Role: Presents generated questions to candidate for answers via a web form.  
    - Configuration: Dynamic form fields populated from AI-generated questions, required fields enforced.  
    - Inputs: AI-generated questions  
    - Outputs: Candidate responses  
    - Edge Cases: Form submission errors, incomplete responses  

  - **update questionnaires**  
    - Type: Airtable node (update operation)  
    - Role: Updates candidate record with questionnaire responses in Airtable.  
    - Configuration: Maps each question and response into a single string field for tracking.  
    - Inputs: Candidate record ID, form responses  
    - Outputs: Confirmation of update  
    - Edge Cases: Airtable update failures, data mapping errors  

---

#### 1.5 Personalized Email Communication and Interview Scheduling

- **Overview:**  
  This block crafts a personalized email inviting the candidate for a phone interview, sends the email, schedules the interview meeting using Google Calendar, and updates Airtable with the scheduled time.

- **Nodes Involved:**  
  - Personalize email (OpenAI node)  
  - Edit Fields (Set node)  
  - Send Email  
  - Book Meeting (OpenAI node)  
  - Google Calendar  
  - update phone meeting time (Airtable update)  

- **Node Details:**  

  - **Personalize email**  
    - Type: OpenAI node  
    - Role: Generates a professional and warm email tailored to the candidate’s strengths and questionnaire responses.  
    - Configuration: Uses GPT-4O model, prompt includes candidate CV, job description, and questionnaire data from Airtable.  
    - Inputs: Candidate data and questionnaire responses  
    - Outputs: Email content with To, Subject, and body fields  
    - Edge Cases: AI generation errors, incomplete data  

  - **Edit Fields**  
    - Type: Set node  
    - Role: Extracts and formats email fields (To, Subject, Content) from AI output for sending.  
    - Inputs: AI-generated email JSON  
    - Outputs: Structured email fields  
    - Edge Cases: Missing fields in AI output  

  - **Send Email**  
    - Type: Email Send node  
    - Role: Sends the personalized email to the candidate via SMTP.  
    - Configuration: Uses configured SMTP credentials, sends plain text email.  
    - Inputs: Email fields from Set node  
    - Outputs: Email send confirmation  
    - Edge Cases: SMTP authentication failures, email delivery issues  

  - **Book Meeting**  
    - Type: OpenAI node  
    - Role: Checks interviewer calendar for available 30-minute slots next day, schedules meeting, and confirms time.  
    - Configuration: Uses GPT-4O model, prompt includes interviewer calendar and current date, instructs to use calendar tool for booking.  
    - Inputs: Interviewer calendar data, current date  
    - Outputs: Meeting start and end times  
    - Edge Cases: Calendar conflicts, AI scheduling errors  

  - **Google Calendar**  
    - Type: Google Calendar Tool node  
    - Role: Creates the scheduled meeting event in Google Calendar with online location.  
    - Configuration: Uses OAuth2 credentials, sets start and end times from AI output.  
    - Inputs: Meeting times from Book Meeting node  
    - Outputs: Event creation confirmation  
    - Edge Cases: Calendar API errors, invalid time formats  

  - **update phone meeting time**  
    - Type: Airtable node (update operation)  
    - Role: Updates candidate record with scheduled phone interview time.  
    - Configuration: Maps meeting start time to "Phone interview" field in Airtable.  
    - Inputs: Candidate record ID, meeting start time  
    - Outputs: Confirmation of update  
    - Edge Cases: Airtable update failures  

---

#### 1.6 Screening Questions Generation and Airtable Update

- **Overview:**  
  Generates screening questions based on the job description, candidate CV, and questionnaire responses, then updates Airtable with these questions for further interview stages.

- **Nodes Involved:**  
  - Screening Questions (OpenAI node)  
  - job_posting1 (Airtable search)  
  - candidate_insights1 (Airtable tool)  
  - Edit Fields1 (Set node)  
  - screening questions (Airtable update)  

- **Node Details:**  

  - **Screening Questions**  
    - Type: OpenAI node  
    - Role: Creates a list of at least 5 screening questions focusing on experience, skills, and cultural fit.  
    - Configuration: Uses GPT-4O model, prompt includes job description, candidate CV, and questionnaire responses.  
    - Inputs: Extracted CV text, Airtable data for job description and questionnaire responses  
    - Outputs: Screening questions as a paragraph with each question on a new line  
    - Edge Cases: AI output formatting issues, API errors  

  - **job_posting1**  
    - Type: Airtable Tool node (search operation)  
    - Role: Retrieves job posting data for use in AI prompt.  
    - Inputs: None (default search)  
    - Outputs: Job description data  
    - Edge Cases: No data found  

  - **candidate_insights1**  
    - Type: Airtable Tool node  
    - Role: Retrieves candidate questionnaire responses for AI prompt.  
    - Inputs: Candidate record ID  
    - Outputs: Questionnaire responses  
    - Edge Cases: Missing data  

  - **Edit Fields1**  
    - Type: Set node  
    - Role: Extracts screening questions from AI output for Airtable update.  
    - Inputs: AI-generated screening questions  
    - Outputs: Structured field for Airtable  
    - Edge Cases: Missing or malformed AI output  

  - **screening questions**  
    - Type: Airtable node (update operation)  
    - Role: Updates candidate record with generated screening questions.  
    - Inputs: Candidate record ID, screening questions text  
    - Outputs: Confirmation of update  
    - Edge Cases: Airtable update errors  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                                      | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------|--------------------------------|-----------------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                   | Capture candidate application data                   | -                             | Upload CV to google drive       |                                                                                              |
| Upload CV to google drive| Google Drive                  | Upload candidate CV to Google Drive                   | On form submission             | applicant details               |                                                                                              |
| applicant details       | Set                           | Prepare structured candidate data for Airtable       | Upload CV to google drive      | Airtable                       | ## Grab User Details and Update in Airtable                                                  |
| Airtable               | Airtable                      | Store candidate data in Airtable                      | applicant details              | download CV                    |                                                                                              |
| download CV            | Google Drive (download)        | Download candidate CV from Google Drive               | Airtable                      | Extract from File              | ## Download the CV and get the job description and requirements. Send the details to ChatGPT to score the viability of the candidate |
| Extract from File       | Extract from File              | Extract text from PDF CV                               | download CV                   | AI Agent                      |                                                                                              |
| Airtable1              | Airtable Tool                 | Retrieve job description from Airtable                | -                             | AI Agent                      |                                                                                              |
| AI Agent               | Langchain Agent               | AI evaluation of candidate CV vs job description      | Extract from File, Airtable1  | Structured Output Parser       |                                                                                              |
| Structured Output Parser| Langchain Output Parser       | Parse AI output into structured JSON                  | AI Agent                     | shortlisted?                  |                                                                                              |
| shortlisted?            | If                            | Decide candidate status based on AI score             | Structured Output Parser       | Potential Hire, Rejected       | ## Update Airtable with score and reason for the score. If score > 0.7, shortlist and continue flow. Get questionnaires based on JD and CV. Update responses in Airtable |
| Potential Hire          | Airtable                      | Update Airtable record for shortlisted candidates     | shortlisted? (true branch)     | generate questionnaires        |                                                                                              |
| Rejected                | Airtable                      | Update Airtable record for rejected candidates        | shortlisted? (false branch)    | -                            |                                                                                              |
| generate questionnaires | OpenAI                        | Generate interview questions based on JD and CV       | Potential Hire                | questionnaires                |                                                                                              |
| questionnaires          | Form                          | Present interview questions to candidate              | generate questionnaires        | update questionnaires          |                                                                                              |
| update questionnaires   | Airtable                      | Update Airtable with candidate questionnaire responses| questionnaires                | Personalize email             |                                                                                              |
| Personalize email       | OpenAI                        | Generate personalized interview invitation email      | update questionnaires          | Edit Fields                   | ## Personalize email and send. Schedule Meeting and update meeting time in Airtable           |
| Edit Fields             | Set                           | Format email fields for sending                        | Personalize email              | Send Email                   |                                                                                              |
| Send Email              | Email Send                    | Send personalized email to candidate                   | Edit Fields                   | Book Meeting                 |                                                                                              |
| Book Meeting            | OpenAI                        | Schedule phone interview meeting                        | Send Email                   | Google Calendar              |                                                                                              |
| Google Calendar         | Google Calendar Tool          | Create calendar event for interview                    | Book Meeting                 | update phone meeting time    |                                                                                              |
| update phone meeting time| Airtable                     | Update Airtable with scheduled interview time          | Google Calendar              | Screening Questions          |                                                                                              |
| Screening Questions     | OpenAI                        | Generate screening questions for further interviews    | update phone meeting time     | Edit Fields1                 | ## Generate Screening Questions and post to Airtable                                         |
| job_posting             | Airtable Tool                 | Retrieve job posting data                               | -                             | Personalize email             |                                                                                              |
| candidate_insights      | Airtable Tool                 | Retrieve candidate insights                             | update questionnaires         | Personalize email             |                                                                                              |
| job_posting1            | Airtable Tool                 | Retrieve job posting data                               | -                             | Screening Questions          |                                                                                              |
| candidate_insights1     | Airtable Tool                 | Retrieve candidate insights                             | update questionnaires         | Screening Questions          |                                                                                              |
| Edit Fields1            | Set                           | Prepare screening questions for Airtable update        | Screening Questions           | screening questions          |                                                                                              |
| screening questions     | Airtable                      | Update Airtable with screening questions                | Edit Fields1                 | -                            |                                                                                              |
| Sticky Note             | Sticky Note                   | Notes on user details and Airtable update              | -                             | -                            | ## Grab User Details and Update in Airtable                                                  |
| Sticky Note1            | Sticky Note                   | Notes on CV download and AI scoring                     | -                             | -                            | ## Download the CV and get the job description and requirements. Send the details to ChatGPT to score the viability of the candidate |
| Sticky Note2            | Sticky Note                   | Notes on Airtable update and questionnaire generation   | -                             | -                            | ## Update Airtable with score and reason for the score. If score is above 0.7, shortlist and continue flow. Get questionnaires based on the JD and CV. Update the responses in Airtable |
| Sticky Note3            | Sticky Note                   | Notes on email personalization and meeting scheduling  | -                             | -                            | ## Personalize email and send. Schedule Meeting and update meeting time in Airtable           |
| Sticky Note4            | Sticky Note                   | Notes on screening questions generation and Airtable update | -                             | -                            | ## Generate Screening Questions and post to Airtable                                         |
| Sticky Note5            | Sticky Note                   | Setup instructions and customization notes              | -                             | -                            | ## Actions: Change the Form Description with the job description you are hiring for. Check and change prompts if needed. Use Simple Applicant Tracker template on Airtable. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form path: `automation-specialist-application`  
   - Add fields: First Name (required), Last Name (required), Email (email, required), Phone (number, required), Years of experience (number, required), Upload your CV (file, PDF only, required)  
   - Set form description with detailed job description and requirements  
   - Enable webhook  

2. **Add Google Drive Node ("Upload CV to google drive")**  
   - Operation: Upload  
   - Folder: Select or enter folder ID for CV storage (e.g., "HR Test" folder)  
   - File name: Use original file name from form submission  
   - Input data field: `Upload_your_CV` (binary)  
   - Connect credentials for Google Drive OAuth2  

3. **Add Set Node ("applicant details")**  
   - Assign fields:  
     - Name: Concatenate First Name + " " + Last Name from form data  
     - Phone: Phone number from form data  
     - Email: Email from form data  
     - Experience: Years of experience from form data  
     - Applied On: Submission timestamp  
     - CV link: Use `webViewLink` from Google Drive upload output  

4. **Add Airtable Node ("Airtable")**  
   - Operation: Create  
   - Base: Select Airtable base for applicant tracking  
   - Table: "Applicants"  
   - Map fields: Name, Phone, CV Link, Applying for (set to ["Automation Specialist"]), Email address  
   - Use Airtable API token credentials  

5. **Add Google Drive Node ("download CV")**  
   - Operation: Download  
   - File ID: Use CV Link from Airtable record  
   - Connect Google Drive OAuth2 credentials  

6. **Add Extract from File Node ("Extract from File")**  
   - Operation: PDF extraction  
   - Input: Binary file from download CV node  

7. **Add Airtable Tool Node ("Airtable1")**  
   - Operation: Search  
   - Base: Same as above  
   - Table: "Positions"  
   - Use Airtable API token credentials  

8. **Add Langchain Agent Node ("AI Agent")**  
   - Prompt: Compare job description and resume, assign score 0-1 with reason (max 20 words)  
   - Inputs: Extracted CV text and job description from Airtable1  
   - Connect OpenAI API credentials  

9. **Add Structured Output Parser Node ("Structured Output Parser")**  
   - JSON schema example: `{ "score": 0.8, "reason": "Does not meet required number of experience in years" }`  
   - Input: AI Agent output  

10. **Add If Node ("shortlisted?")**  
    - Condition: Check if score >= 0.7  
    - Input: Parsed score from Structured Output Parser  

11. **Add Airtable Node ("Potential Hire")**  
    - Operation: Update  
    - Table: "Applicants"  
    - Update fields: Stage = "Interviewing", JD CV score, CV Score Notes  
    - Input: Candidate record ID from Airtable  

12. **Add Airtable Node ("Rejected")**  
    - Operation: Update  
    - Table: "Applicants"  
    - Update fields: Stage = "No hire", JD CV score, CV Score Notes  
    - Input: Candidate record ID from Airtable  

13. **Add OpenAI Node ("generate questionnaires")**  
    - Model: GPT-4O-MINI  
    - Prompt: Generate 5 interview questions based on job description and candidate CV  
    - Input: Job description and extracted CV text  
    - Connect OpenAI API credentials  

14. **Add Form Node ("questionnaires")**  
    - Dynamic form fields: Populate with generated questions  
    - Button label: Submit  
    - Form description: "Kindly fill in the following questions to proceed."  

15. **Add Airtable Node ("update questionnaires")**  
    - Operation: Update  
    - Table: "Applicants"  
    - Update field: Store questionnaire questions and candidate responses as a formatted string  
    - Input: Candidate record ID and form responses  

16. **Add OpenAI Node ("Personalize email")**  
    - Model: GPT-4O  
    - Prompt: Generate personalized email inviting candidate for phone interview, referencing CV and questionnaire responses  
    - Inputs: Candidate CV, job description, questionnaire responses from Airtable  
    - Connect OpenAI API credentials  

17. **Add Set Node ("Edit Fields")**  
    - Extract To, Subject, and Email Content from AI output for email sending  

18. **Add Email Send Node ("Send Email")**  
    - Configure SMTP credentials  
    - From: Your email address  
    - To, Subject, Text: From Set node  
    - Email format: Text  

19. **Add OpenAI Node ("Book Meeting")**  
    - Model: GPT-4O  
    - Prompt: Check interviewer calendar for 30-minute slots next day, schedule meeting, confirm time  
    - Inputs: Interviewer calendar, current date  
    - Connect OpenAI API credentials  

20. **Add Google Calendar Node ("Google Calendar")**  
    - Operation: Create event  
    - Calendar: Select interviewer calendar  
    - Start and End times: From AI output  
    - Location: Online  
    - Connect Google Calendar OAuth2 credentials  

21. **Add Airtable Node ("update phone meeting time")**  
    - Operation: Update  
    - Table: "Applicants"  
    - Update field: Phone interview time with meeting start time  
    - Input: Candidate record ID  

22. **Add OpenAI Node ("Screening Questions")**  
    - Model: GPT-4O  
    - Prompt: Generate screening questions based on job description, candidate CV, and questionnaire responses  
    - Inputs: Job description, CV text, questionnaire responses  
    - Connect OpenAI API credentials  

23. **Add Airtable Tool Nodes ("job_posting1", "candidate_insights1")**  
    - Retrieve job posting and candidate questionnaire data for AI prompt  

24. **Add Set Node ("Edit Fields1")**  
    - Extract screening questions from AI output  

25. **Add Airtable Node ("screening questions")**  
    - Operation: Update  
    - Table: "Applicants"  
    - Update field: Screening questions  
    - Input: Candidate record ID  

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Change the `Form Description` with the job description you are hiring for.                                                    | Sticky Note5                                                                                            |
| Make sure to check and modify AI prompts to suit your specific recruitment needs.                                            | Sticky Note5                                                                                            |
| Use the "Simple Applicant Tracker" Airtable template to set up required tables and fields.                                   | Sticky Note5, Airtable base: https://airtable.com/appublMkWVQfHkZ09                                       |
| The workflow uses OpenAI GPT-4O and GPT-4O-MINI models for candidate evaluation, questionnaire generation, and email crafting. | Requires valid OpenAI API key configured in n8n.                                                        |
| Google Drive folder for CV storage must have appropriate write permissions.                                                  | Folder ID example: `1u_YBpqSU5TjNsu72sQKFMIesb62JKHXz`                                                   |
| SMTP email account is required for sending candidate communications.                                                         | Configure SMTP credentials in n8n for email sending node.                                                |
| Interview scheduling respects working hours (8 AM - 5 PM) and next-day availability.                                          | Managed by AI prompt in "Book Meeting" node.                                                            |

---

This documentation provides a comprehensive understanding of the HR Job Posting and Evaluation with AI workflow, enabling users and automation agents to reproduce, modify, and troubleshoot the workflow effectively.