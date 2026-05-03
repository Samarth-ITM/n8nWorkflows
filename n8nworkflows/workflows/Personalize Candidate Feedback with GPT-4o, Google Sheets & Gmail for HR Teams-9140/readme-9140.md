Personalize Candidate Feedback with GPT-4o, Google Sheets & Gmail for HR Teams

https://n8nworkflows.xyz/workflows/personalize-candidate-feedback-with-gpt-4o--google-sheets---gmail-for-hr-teams-9140


# Personalize Candidate Feedback with GPT-4o, Google Sheets & Gmail for HR Teams

### 1. Workflow Overview

This workflow automates personalized candidate feedback for HR teams by integrating Google Sheets, Google Drive, Azure OpenAI’s GPT-4o-mini model, and Gmail. It fetches candidate data and resumes, analyzes skill gaps relative to job requirements using AI, and sends tailored emails either congratulating shortlisted candidates with onboarding plans or politely rejecting others with constructive learning recommendations.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Data Fetch:** Trigger and retrieval of candidate data and resume files.
- **1.2 Resume Validation & Text Extraction:** Verification of resume file integrity and extraction of text from PDFs.
- **1.3 Candidate Data Enrichment:** Merging structured sheet data with unstructured resume text.
- **1.4 Status-based Branching:** Differentiating between shortlisted and rejected candidates.
- **1.5 AI Processing (LLM Backend):** Using GPT-4o-mini to generate personalized email content for each candidate type.
- **1.6 Email Dispatch:** Sending generated emails via Gmail integration.
- **1.7 Error Handling:** Logging failures related to resume download or extraction.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Fetch

**Overview:**  
Starts the workflow manually and fetches candidate details from a Google Sheet that serves as the source of truth for candidate metadata.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- 📑 Candidate Data Fetch (Google Sheets node)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates workflow execution manually  
  - Inputs: None  
  - Outputs: Triggers next node  
  - Failures: None expected  
  - Version: 1  

- **📑 Candidate Data Fetch**  
  - Type: Google Sheets (OAuth2)  
  - Role: Reads candidate data from a Google Sheet (columns include name, email, resume link, experience, job title, skills, status, etc.)  
  - Configuration:  
    - Document ID: Linked to specific Google Sheet (“Interviewer Brief Pack”)  
    - Sheet Name: “Sheet1” (gid=0)  
    - Credentials: OAuth2 for Google Sheets using automations@techdome.ai  
  - Inputs: Trigger from manual node  
  - Outputs: Passes candidate row data downstream  
  - Failure Modes: Authentication errors, API quota limits, spreadsheet access issues  
  - Version: 4  

---

#### 2.2 Resume Validation & Text Extraction

**Overview:**  
Downloads candidate resumes from Google Drive, validates successful download, and extracts raw text from PDF resumes for AI processing.

**Nodes Involved:**  
- 📥 Resume Downloader (Google Drive)  
- Resume File Check (If node)  
- PDF → Text Extractor (Extract from File node)  
- Error Logging (Google Sheets Append)

**Node Details:**  

- **📥 Resume Downloader**  
  - Type: Google Drive  
  - Role: Downloads PDF resume files using URL from candidate data  
  - Configuration: File ID dynamically set from “Resume Link” in Google Sheet row  
  - Credentials: OAuth2 for Google Drive (Techdome Account)  
  - Inputs: Candidate data from previous node  
  - Outputs: Binary file data or empty if download fails  
  - Failure Modes: Invalid/missing file link, permissions errors, file not found  
  - Version: 3  

- **Resume File Check**  
  - Type: If node  
  - Role: Checks if downloaded resume file contains data (non-empty)  
  - Configuration: Condition checks if JSON path `data` is not empty  
  - Inputs: Resume file binary data from downloader  
  - Outputs:  
    - True branch: Resume valid → proceeds to PDF text extraction  
    - False branch: Resume invalid → routes to error logging  
  - Failure Modes: Expression evaluation errors, empty or corrupt file  
  - Version: 2.2  

- **PDF → Text Extractor**  
  - Type: Extract from File  
  - Role: Extracts raw text content from the PDF resume  
  - Configuration: Operation set to PDF text extraction  
  - Inputs: Validated resume file  
  - Outputs: JSON containing extracted text  
  - Failure Modes: PDF parsing errors, file corruption  
  - Version: 1  

- **Error Logging**  
  - Type: Google Sheets Append  
  - Role: Logs failed or empty resume downloads for HR audit  
  - Configuration: Appends error details to “error log sheet” within same Google Sheet document  
  - Credentials: OAuth2 for Google Sheets (automations@techdome.ai)  
  - Inputs: Triggered when resume file check fails  
  - Outputs: None  
  - Failure Modes: Google Sheets write permission errors  
  - Version: 4.7  

---

#### 2.3 Candidate Data Enrichment

**Overview:**  
Combines structured candidate data from the sheet with unstructured resume text for a comprehensive data package used by AI.

**Nodes Involved:**  
- Candidate Data Builder (Code node)  

**Node Details:**  

- **Candidate Data Builder**  
  - Type: Code (JavaScript)  
  - Role: Merges Google Sheets candidate data with extracted resume text into a single JSON object  
  - Configuration: Custom JS code accesses PDF text and sheet row via `$json` and `$node` references, outputting enriched candidate data including name, email, experience, job title, skills, notes, status, and resume text  
  - Inputs: JSON from PDF → Text Extractor and original candidate data from sheet  
  - Outputs: Enriched JSON object forwarded for branching  
  - Failure Modes: Reference errors if upstream nodes fail or data missing  
  - Version: 2  

---

#### 2.4 Status-based Branching

**Overview:**  
Splits workflow based on candidate status (“Shortlisted” or other) to tailor communication.

**Nodes Involved:**  
- Shortlisted vs Rejected (If node)

**Node Details:**  

- **Shortlisted vs Rejected**  
  - Type: If node  
  - Role: Checks if candidate’s status equals “Shortlisted”  
  - Configuration: String equality condition on `$json.status`  
  - Inputs: Enriched candidate data from Candidate Data Builder  
  - Outputs:  
    - True branch: Shortlisted candidates → onboarding email generation  
    - False branch: Rejected candidates → polite rejection email generation  
  - Failure Modes: Case sensitivity or missing status field causing misrouting  
  - Version: 2.2  

---

#### 2.5 AI Processing (LLM Backend)

**Overview:**  
Uses Azure-hosted GPT-4o-mini to generate personalized HTML emails for both shortlisted and rejected candidates based on enriched data.

**Nodes Involved:**  
- LLM Backend (Azure OpenAI node)  
- LLM Backend1 (Azure OpenAI node)  
- Congrats + Onboarding Plan (LangChain chainLlm node)  
- Polite Rejection + Learning Plan (LangChain chainLlm node)

**Node Details:**  

- **LLM Backend & LLM Backend1**  
  - Type: Azure OpenAI Chat Language Model (GPT-4o-mini)  
  - Role: Provide AI processing backend for prompt chains generating emails  
  - Configuration: Model set to “gpt-4o-mini”  
  - Credentials: Azure OpenAI API account  
  - Inputs: Text prompts from chain nodes  
  - Outputs: AI-generated HTML email content  
  - Failure Modes: API limit, authentication, or latency errors  
  - Version: 1  

- **Congrats + Onboarding Plan**  
  - Type: LangChain chainLlm  
  - Role: Creates warm, congratulatory HTML email for shortlisted candidates including skill gaps, recommended courses, and onboarding steps  
  - Configuration:  
    - Input text template includes candidate details, resume text, job info, and notes  
    - Prompt instructs model to generate positive, professional email with sections on skill gaps, courses, next steps, formatted in HTML  
  - Inputs: Enriched candidate JSON, chained to LLM Backend  
  - Outputs: Clean HTML email content  
  - Failure Modes: Prompt errors, incomplete data causing incoherent emails  
  - Version: 1.7  

- **Polite Rejection + Learning Plan**  
  - Type: LangChain chainLlm  
  - Role: Generates polite rejection email with constructive skill gap analysis and learning recommendations (including course URLs)  
  - Configuration:  
    - Input text template similarly structured with candidate and job data  
    - Prompt instructs AI to identify skill gaps, recommend courses (including free options), and generate employer-brand-friendly HTML email  
  - Inputs: Enriched candidate JSON, chained to LLM Backend1  
  - Outputs: Polished HTML email content  
  - Failure Modes: Same as above; also depends on availability of relevant courses in AI knowledge base  
  - Version: 1.7  

---

#### 2.6 Email Dispatch

**Overview:**  
Sends the generated personalized emails via Gmail to candidates depending on their status.

**Nodes Involved:**  
- Candidate Mailer – Shortlisted (Gmail node)  
- Candidate Mailer – Rejected (Gmail node)

**Node Details:**  

- **Candidate Mailer – Shortlisted**  
  - Type: Gmail (OAuth2)  
  - Role: Sends congratulatory onboarding email to shortlisted candidates  
  - Configuration:  
    - Recipient email dynamically set from candidate data  
    - Email message body uses AI-generated HTML content  
    - Subject preset as “Your Personalized Learning Plan”  
    - Credentials: Gmail OAuth2 (Gmail account)  
  - Inputs: Output from Congrats + Onboarding Plan  
  - Outputs: Email sent confirmation  
  - Failure Modes: Authentication failure, email sending limits, invalid email addresses  
  - Version: 2.1  

- **Candidate Mailer – Rejected**  
  - Type: Gmail (OAuth2)  
  - Role: Sends polite rejection + training recommendation email to rejected candidates  
  - Configuration: Similar to shortlisted mailer but receives content from Polite Rejection + Learning Plan  
  - Inputs: Polite Rejection + Learning Plan output  
  - Outputs: Email sent confirmation  
  - Failure Modes: Same as above  
  - Version: 2.1  

---

#### 2.7 Error Handling

**Overview:**  
Logs any resume download or extraction failures to a dedicated Google Sheet for audit and manual follow-up.

**Nodes Involved:**  
- Error Logging (Google Sheets Append)

**Node Details:**  

- **Error Logging**  
  - See details in 2.2 Resume Validation & Text Extraction section  
  - Acts as final error catch branch when resume file is missing or empty

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                                  | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                                                                |
|---------------------------|----------------------------------|-------------------------------------------------|-------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Starts workflow manually                         | None                          | 📑 Candidate Data Fetch               |                                                                                                                                            |
| 📑 Candidate Data Fetch    | Google Sheets                    | Fetches candidate metadata from Google Sheet   | When clicking ‘Execute workflow’ | 📥 Resume Downloader                 | ## 📑 Candidate Data Fetch  \n- Retrieves candidate details from Google Sheet as source of truth.                                           |
| 📥 Resume Downloader       | Google Drive                    | Downloads candidate resume PDF                   | 📑 Candidate Data Fetch         | Resume File Check                    | ## 📥 Resume Downloader  \n- Downloads candidate’s resume PDF from Google Drive.                                                             |
| Resume File Check          | If                              | Checks if resume file is valid (non-empty)      | 📥 Resume Downloader           | PDF → Text Extractor (true branch) \nError Logging (false branch) | ## ✅/❌ Resume File Check  \n- Validates successful resume download; routes to error log if invalid.                                         |
| PDF → Text Extractor       | Extract from File                | Extracts raw text from PDF                        | Resume File Check (true branch) | Candidate Data Builder               | ## 📄 PDF → Text Extractor  \n- Extracts text from candidate’s resume PDF for AI analysis.                                                   |
| Candidate Data Builder     | Code                           | Merges sheet data and extracted resume text     | PDF → Text Extractor           | Shortlisted vs Rejected              | ## 🧩 Candidate Data Builder  \n- Combines structured and unstructured candidate data into one JSON object.                                  |
| Shortlisted vs Rejected    | If                              | Branches workflow based on candidate status     | Candidate Data Builder         | Congrats + Onboarding Plan (true) \nPolite Rejection + Learning Plan (false) | ## 🎯 Shortlisted vs Rejected  \n- Splits path based on candidate’s status field.                                                            |
| LLM Backend               | Azure OpenAI (LangChain)        | AI backend for shortlisted candidate emails     | Congrats + Onboarding Plan     | Congrats + Onboarding Plan           | ## 🤖 LLM Backend  \n- Provides GPT-4o-mini model for AI chains.                                                                             |
| LLM Backend1              | Azure OpenAI (LangChain)        | AI backend for rejected candidate emails         | Polite Rejection + Learning Plan | Polite Rejection + Learning Plan   | ## 🤖 LLM Backend  \n- Provides GPT-4o-mini model for AI chains.                                                                             |
| Congrats + Onboarding Plan | LangChain chainLlm              | Generates congratulatory onboarding HTML email  | Shortlisted vs Rejected (true) | Candidate Mailer – Shortlisted       | ## 🎉 Congrats + Onboarding Plan  \n- AI generates warm HTML email with skill gaps, courses, onboarding steps for shortlisted candidates.      |
| Polite Rejection + Learning Plan | LangChain chainLlm          | Generates polite rejection + learning plan email | Shortlisted vs Rejected (false) | Candidate Mailer – Rejected          | ## 🙏 Polite Rejection + Learning Plan  \n- AI generates professional rejection email with constructive feedback and training links.         |
| Candidate Mailer – Shortlisted | Gmail                        | Sends onboarding email to shortlisted candidate | Congrats + Onboarding Plan     | None                               | ## 📧 Candidate Mailer – Shortlisted  \n- Sends personalized onboarding email via Gmail.                                                    |
| Candidate Mailer – Rejected | Gmail                         | Sends rejection email with learning plan         | Polite Rejection + Learning Plan | None                             | ## 📧 Candidate Mailer – Rejected  \n- Sends polite rejection email with learning recommendations via Gmail.                                |
| Error Logging             | Google Sheets Append            | Logs failures in resume download or extraction  | Resume File Check (false branch) | None                               | ## ⚠️ Error Logging  \n- Logs failed or empty resume downloads into error log sheet for HR follow-up.                                         |
| Sticky Note1 to Sticky Note13 | Sticky Note                   | Provides documentation and explanations          | None                          | None                               | See detailed sticky notes content in Overview and block sections above.                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**  
   - Add a Manual Trigger node named “When clicking ‘Execute workflow’”.

2. **Add Google Sheets Node for Candidate Data Fetch**  
   - Create a Google Sheets node named “📑 Candidate Data Fetch”.  
   - Configure with OAuth2 credentials for Google Sheets (e.g., automations@techdome.ai).  
   - Set Document ID to the Google Sheet containing candidate data.  
   - Set Sheet Name to the sheet with candidates (e.g., “Sheet1”).  
   - Connect manual trigger output to this node input.

3. **Add Google Drive Node to Download Resume**  
   - Add a Google Drive node named “📥 Resume Downloader”.  
   - Configure with OAuth2 credentials for Google Drive (Techdome Account).  
   - Set operation to “download”.  
   - Set fileId parameter dynamically: `={{ $json["Resume Link"] }}`.  
   - Connect output of Candidate Data Fetch to this node.

4. **Add If Node to Check Resume File**  
   - Add an If node named “Resume File Check”.  
   - Configure condition to check if downloaded file data is not empty: expression on `$json.data` is not empty.  
   - Connect Resume Downloader output to this node.

5. **Add PDF Text Extractor Node**  
   - Add “Extract from File” node named “PDF → Text Extractor”.  
   - Configure operation to “pdf” text extraction.  
   - Connect true branch output of Resume File Check to this node.

6. **Add Code Node to Merge Candidate Data**  
   - Add a Code node named “Candidate Data Builder”.  
   - Use JavaScript to combine Google Sheets row and extracted resume text into one JSON object.  
   - Reference inputs from both the PDF extractor and initial Candidate Data Fetch node.  
   - Connect PDF → Text Extractor output to this node.

7. **Add If Node to Branch by Candidate Status**  
   - Add an If node named “Shortlisted vs Rejected”.  
   - Set condition to check if `{{$json.status}} == "Shortlisted"`.  
   - Connect Candidate Data Builder output to this node.

8. **Add Azure OpenAI Nodes for AI Backend**  
   - Add two Azure OpenAI Chat nodes (model “gpt-4o-mini”), named “LLM Backend” and “LLM Backend1”.  
   - Configure both with Azure OpenAI credentials.

9. **Add LangChain ChainLlm Nodes for Email Generation**  
   - Add “Congrats + Onboarding Plan” node (LangChain chainLlm) connected to “LLM Backend”.  
     - Use prompt template to create congratulatory onboarding email with skill gaps, recommended courses, and next steps, using enriched candidate data.  
   - Add “Polite Rejection + Learning Plan” node (LangChain chainLlm) connected to “LLM Backend1”.  
     - Use prompt template to generate polite rejection email with skill gaps and course recommendations.

10. **Connect Branching Outputs to Email Generators**  
    - Connect true branch of “Shortlisted vs Rejected” to “Congrats + Onboarding Plan”.  
    - Connect false branch to “Polite Rejection + Learning Plan”.

11. **Add Gmail Nodes to Send Emails**  
    - Add a Gmail node named “Candidate Mailer – Shortlisted”.  
      - Configure with Gmail OAuth2 credentials.  
      - Set recipient to `={{ $('📑 Candidate Data Fetch').item.json.Email }}`.  
      - Set subject to “Your Personalized Learning Plan”.  
      - Set message body to `={{ $json.text }}` from AI output.  
      - Connect output of “Congrats + Onboarding Plan” to this node.  
    - Add a Gmail node named “Candidate Mailer – Rejected” with similar configuration.  
      - Connect output of “Polite Rejection + Learning Plan” to this node.

12. **Add Google Sheets Append Node for Error Logging**  
    - Add Google Sheets node named “Error Logging”.  
    - Configure to append rows to “error log sheet” in the same Google Sheet document.  
    - Connect false branch of “Resume File Check” to this node.

13. **Verify and Test Workflow**  
    - Ensure all credentials are correctly configured and authorized.  
    - Test the workflow with sample candidate data and resume links.  
    - Monitor logs for errors and validate email content formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                            | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The GPT-4o-mini model is hosted on Azure OpenAI and used as the core AI engine for generating personalized candidate communications.                                                                                                                   | Azure OpenAI GPT-4o-mini                                                                                 |
| The Google Sheet named “Interviewer Brief Pack” is the authoritative source of candidate data, including resumes stored in Google Drive.                                                                                                                | Google Sheets document ID: 1Uldk_4BxWbdZTDZxFUeohIfeBmGHHqVEl9Ogb0l6R8Y                                  |
| Polite rejection emails include links to real online courses from Coursera, Udemy, and LinkedIn Learning, aiming to provide constructive feedback and improve candidate experience.                                                                      | See prompt instructions in “Polite Rejection + Learning Plan” node                                      |
| The workflow includes robust error handling to avoid silent failures in resume processing, improving reliability and auditability for HR teams.                                                                                                         | Error Logging node writes to a dedicated error log sheet                                               |
| Email sending uses OAuth2 Gmail integration ensuring emails are sent from the company’s official account for branding and professionalism.                                                                                                              | Gmail OAuth2 credentials used in mailer nodes                                                          |
| The workflow employs LangChain nodes for prompt chaining and structured AI interaction, facilitating complex multi-step instructions to the language model.                                                                                            | LangChain chainLlm nodes “Congrats + Onboarding Plan” and “Polite Rejection + Learning Plan”           |
| Sticky notes embedded within the workflow visually document the purpose and functionality of each block to assist maintenance and knowledge transfer.                                                                                                  | Sticky notes are positioned next to relevant nodes in the workflow editor                              |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. It complies fully with applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.