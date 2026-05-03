Generate Tailored Resumes, Cover Letters & Interview Prep from LinkedIn Jobs with AI

https://n8nworkflows.xyz/workflows/generate-tailored-resumes--cover-letters---interview-prep-from-linkedin-jobs-with-ai-6748


# Generate Tailored Resumes, Cover Letters & Interview Prep from LinkedIn Jobs with AI

### 1. Workflow Overview

This workflow automates the generation of tailored resumes, cover letters, and interview preparation materials based on LinkedIn job postings using AI. It is designed for job seekers who want personalized job application documents and coaching, leveraging structured job and company data extracted from LinkedIn URLs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Data Extraction**  
  Receives job URL input via chat message, scrapes detailed structured job and company information from LinkedIn using BrightData.

- **1.2 Data Storage**  
  Appends extracted job and company details into Google Sheets for persistence and referencing by subsequent agents.

- **1.3 AI Orchestration**  
  The Master Agent orchestrates AI sub-agents:  
  - Resume Editor (edits resume based on job/company info and HR feedback)  
  - HR Reviewer (reviews the resume, returns comments and scores)  
  - Cover Letter Generator  
  - Interview Coach

- **1.4 Document Generation and Update**  
  Creates and updates Google Docs with generated resumes, cover letters, and interview coaching content.

- **1.5 Workflow Completion Notification**  
  Sends a final model message to confirm workflow completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Extraction

**Overview:**  
Starts with receiving a chat message containing a LinkedIn job URL. Uses BrightData to scrape detailed structured job posting and company information from LinkedIn.

**Nodes Involved:**  
- When chat message received  
- Extract structured data from a single URL  
- Extract structured data from a single URL1

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger (Webhook)  
  - Role: Entry point triggered by an incoming chat message (e.g., Telegram, Slack, custom UI) containing LinkedIn job URL.  
  - Config: Public webhook with basic HTTP authentication.  
  - Input: Chat message with job URL in `chatInput`.  
  - Output: Passes chat message JSON to next node.  
  - Potential failures: Authentication errors, invalid or missing URL in input.

- **Extract structured data from a single URL**  
  - Type: BrightData Web Scraper Node  
  - Role: Scrapes job posting details from the LinkedIn job URL received.  
  - Config: Uses BrightData dataset "Linkedin job listings information"; URL built dynamically from incoming message.  
  - Input: URL from chat message JSON.  
  - Output: Structured JSON with job posting info (title, summary, seniority, function, location, etc.).  
  - Failures: Scraper timeout, invalid URL, BrightData quota or auth errors.

- **Extract structured data from a single URL1**  
  - Type: BrightData Web Scraper Node  
  - Role: Scrapes company details from LinkedIn company page URL built using company ID extracted earlier.  
  - Config: Uses BrightData dataset "LinkedIn company information"; URL uses `company_id` from previous output.  
  - Input: Company LinkedIn URL derived from job data.  
  - Output: Structured company details (name, about, specialties, size, website, etc.).  
  - Failures: Same as above.

---

#### 2.2 Data Storage

**Overview:**  
Stores structured job and company data into Google Sheets for persistence and accessibility by AI agents.

**Nodes Involved:**  
- append job details  
- append company detail

**Node Details:**

- **append job details**  
  - Type: Google Sheets Append  
  - Role: Appends job posting structured data into a specified Google Sheet (sheet "Interested_list").  
  - Config: Auto-maps extracted job data fields (job title, location, summary, etc.) into sheet columns.  
  - Input: Output of company data extraction node.  
  - Output: Confirmation of append operation.  
  - Failures: Google Sheets API rate limits, invalid data mapping.

- **append company detail**  
  - Type: Google Sheets Append  
  - Role: Appends company structured data into a separate Google Sheet tab "company_detail".  
  - Config: Auto-maps company fields (name, followers, size, website, description, etc.). Uses "id" as matching column to avoid duplicates.  
  - Input: Output of job details append node.  
  - Output: Confirmation of append operation.  
  - Failures: Same as above.

---

#### 2.3 AI Orchestration

**Overview:**  
Central coordination by the Master Agent node, which uses LangChain agent architecture to call dedicated AI sub-agents for resume editing, HR reviewing, cover letter generation, and interview coaching. It manages iterative resume refinement based on HR feedback until quality threshold is met.

**Nodes Involved:**  
- Master Agent  
- Resume Editor (Tool Workflow)  
- HR Reviwer (Tool Workflow)  
- Generate Cover Letter (Execute Workflow)  
- Interview Coach (Execute Workflow)  
- GPT-4o (OpenAI LM)  

**Node Details:**

- **Master Agent**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates calls to Resume Editor and HR Reviewer agents; loops resume refinement until HR score > 75%.  
  - Config: System message defines orchestration logic and goals; calls sub-agents via tool interfaces with job & company info.  
  - Inputs: Job and company data from Google Sheets append nodes.  
  - Outputs: Final edited resume with HR comments and score.  
  - Failures: AI timeout, sub-agent workflow errors, infinite loop risk if scoring logic fails.

- **Resume Editor**  
  - Type: LangChain Tool Workflow  
  - Role: Edits resume based on job/company data and HR feedback comment (if any).  
  - Config: Sub-workflow ID linked, accepts query input (resume editing prompt).  
  - Inputs: Company/job data and optionally HR comments.  
  - Outputs: Edited resume text.  
  - Failures: Sub-workflow misconfiguration, input mapping errors.

- **HR Reviwer**  
  - Type: LangChain Tool Workflow  
  - Role: Reviews the edited resume, scores and provides comments/suggestions.  
  - Config: Sub-workflow ID linked, no direct inputs defined (expects data from Master Agent).  
  - Outputs: Comments and score on the resume quality.  
  - Failures: Sub-workflow errors, scoring logic issues.

- **Generate Cover Letter**  
  - Type: Execute Workflow  
  - Role: Generates a tailored cover letter based on the final resume output.  
  - Config: Calls Cover Letter Agent sub-workflow with query input (edited resume).  
  - Outputs: Cover letter text.  
  - Failures: Sub-workflow failure, input/output mismatch.

- **Interview Coach**  
  - Type: Execute Workflow  
  - Role: Provides interview coaching content based on HR feedback, company details, job description, and edited resume.  
  - Config: Calls Interview Coach sub-workflow with combined data input.  
  - Outputs: Interview coaching text.  
  - Failures: Sub-workflow failure, data mapping errors.

- **GPT-4o**  
  - Type: OpenAI Chat Model Node  
  - Role: Provides language model support for Master Agent orchestration and AI processing.  
  - Config: Uses GPT-4o-2024-05-13 model.  
  - Failures: API quota, timeout, rate limiting.

---

#### 2.4 Document Generation and Update

**Overview:**  
Saves AI-generated outputs (resume, cover letter, interview coaching) into Google Docs, creating and updating documents in a specified Drive folder.

**Nodes Involved:**  
- Create a document  
- Update a document  
- Create a document1  
- Update a document1  
- Create a document2  
- Update a document2

**Node Details:**

- **Create a document / Create a document1 / Create a document2**  
  - Type: Google Docs Create  
  - Role: Creates new Google Docs for resume, cover letter, and interview coaching respectively.  
  - Config: Title dynamically generated using company name and job title; saved to specific Drive folders.  
  - Input: Triggered after AI output is ready.  
  - Output: Document metadata including document URL.  
  - Failures: Google Drive permission issues, API limits.

- **Update a document / Update a document1 / Update a document2**  
  - Type: Google Docs Update  
  - Role: Inserts AI-generated text content into corresponding documents.  
  - Config: Text inserted at current cursor position; document URL dynamically passed from create nodes.  
  - Input: AI output text from Master Agent, Cover Letter Generator, and Interview Coach respectively.  
  - Failures: Document locked, permission denied, API errors.

---

#### 2.5 Workflow Completion Notification

**Overview:**  
Final node sends a simple prompt to OpenAI model to confirm workflow completion or status.

**Nodes Involved:**  
- Message a model

**Node Details:**

- **Message a model**  
  - Type: OpenAI Chat Model Node  
  - Role: Sends a concluding message "Is my workflow done?" to GPT-4o-mini model.  
  - Config: Minimal prompt, used to verify workflow end or trigger logging.  
  - Failures: API issues.

---

### 3. Summary Table

| Node Name                             | Node Type                                | Functional Role                                  | Input Node(s)                       | Output Node(s)                         | Sticky Note                                                                                         |
|-------------------------------------|----------------------------------------|-------------------------------------------------|-----------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received           | LangChain Chat Trigger                  | Entry trigger receives LinkedIn job URL          | -                                 | Extract structured data from a single URL | Trigger: When Chat Message Received. Starts workflow with message input.                           |
| Extract structured data from a single URL | BrightData Web Scraper                  | Scrapes job posting details from LinkedIn URL    | When chat message received         | Extract structured data from a single URL1 | Extract Structured Data from LinkedIn. Uses BrightData scraper.                                    |
| Extract structured data from a single URL1 | BrightData Web Scraper                  | Scrapes LinkedIn company details                   | Extract structured data from a single URL | append job details                    |                                                                                                   |
| append job details                   | Google Sheets Append                    | Stores job data into Google Sheet                  | Extract structured data from a single URL1 | append company detail                 | Append Job & Company Details saved for downstream agents.                                         |
| append company detail               | Google Sheets Append                    | Stores company data into Google Sheet              | append job details                 | Master Agent                         |                                                                                                   |
| Master Agent                       | LangChain Agent                         | Orchestrates resume editing and HR review agents  | append company detail              | Generate Cover Letter                 | Master Agent acts as central coordinator.                                                        |
| GPT-4o                            | OpenAI Chat Language Model              | Provides AI language model support                  | -                                 | Master Agent                         |                                                                                                   |
| Resume Editor                     | LangChain Tool Workflow                 | Edits resume using job and company info            | Master Agent                      | Master Agent                         | Sub-workflow agent for resume editing.                                                           |
| HR Reviwer                       | LangChain Tool Workflow                 | Reviews resume and provides comments and scores   | Master Agent                      | Master Agent                         | Sub-workflow agent for HR reviewing.                                                             |
| Generate Cover Letter             | Execute Workflow                        | Generates tailored cover letter                     | Master Agent                      | Create a document                    | Sub-workflow agent for cover letter generation.                                                  |
| Create a document                 | Google Docs Create                      | Creates Google Doc for resume                       | Generate Cover Letter             | Update a document                   | Output saved to Google Drive.                                                                     |
| Update a document                 | Google Docs Update                      | Inserts edited resume content                        | Create a document                 | Create a document1                  |                                                                                                   |
| Create a document1                | Google Docs Create                      | Creates Google Doc for cover letter                  | Update a document                 | Update a document1                  |                                                                                                   |
| Update a document1                | Google Docs Update                      | Inserts cover letter content                          | Create a document1                | Interview Coach                    |                                                                                                   |
| Interview Coach                  | Execute Workflow                        | Generates interview coaching content                 | Update a document1                | Create a document2                  | Sub-workflow agent for interview coaching.                                                       |
| Create a document2               | Google Docs Create                      | Creates Google Doc for interview coaching            | Interview Coach                  | Update a document2                  |                                                                                                   |
| Update a document2               | Google Docs Update                      | Inserts interview coaching content                    | Create a document2                | Message a model                   |                                                                                                   |
| Message a model                 | OpenAI Chat Language Model              | Sends final confirmation prompt                      | Update a document2                | -                                 |                                                                                                   |
| Sticky Note                    | Sticky Note                            | Notes on trigger and data extraction                 | -                                 | -                                 | Trigger: When Chat Message Received. Starts workflow when a message is received.                  |
| Sticky Note1                   | Sticky Note                            | Notes on Master Agent role                            | -                                 | -                                 | Master Agent acts as the central coordinator.                                                    |
| Sticky Note2                   | Sticky Note                            | Notes on sub-workflows such as Resume Editor, etc.  | -                                 | -                                 | Sub-workflows are modular AI agents automatically returning results to Master Agent.             |
| Sticky Note3                   | Sticky Note                            | Notes on Google Docs output                            | -                                 | -                                 | AI-generated documents are saved and updated in Google Drive (resume, cover letter, interview). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "When chat message received"**  
   - Type: LangChain Chat Trigger  
   - Configure webhook with Basic Auth credentials.  
   - Set public webhook to true.  
   - This node will receive chat messages containing LinkedIn job URLs.

2. **Add BrightData Node: "Extract structured data from a single URL"**  
   - Type: BrightData Web Scraper  
   - Credential: BrightData API account.  
   - Dataset: Use the LinkedIn job listings scraper dataset (ID: gd_lpfll7v5hcqtkxl6l).  
   - URL parameter: Set to `{{ $json.chatInput }}` from the trigger node.  
   - Set to execute once per input.

3. **Add BrightData Node: "Extract structured data from a single URL1"**  
   - Type: BrightData Web Scraper  
   - Credential: BrightData API account.  
   - Dataset: Use LinkedIn company information scraper (ID: gd_l1vikfnt1wgvvqz95w).  
   - URL parameter: Construct URL as `https://www.linkedin.com/company/{{ $json.company_id }}/`.  
   - Set to execute once.

4. **Add Google Sheets Node: "append job details"**  
   - Type: Google Sheets Append  
   - Credential: Google Sheets OAuth2.  
   - Spreadsheet: Specify document ID for "Interested_list" sheet.  
   - Map columns to job data fields from previous node output.  
   - Operation: Append new rows.

5. **Add Google Sheets Node: "append company detail"**  
   - Type: Google Sheets Append  
   - Credential: Google Sheets OAuth2.  
   - Spreadsheet: Specify document ID for "company_detail" sheet.  
   - Map columns to company data fields.  
   - Use "id" column to match existing records.  
   - Operation: Append or update rows.

6. **Add LangChain Agent Node: "Master Agent"**  
   - Credential: OpenAI API.  
   - System message defines the orchestration logic: calling Resume Editor and HR Reviewer sub-agents iteratively until score > 75%.  
   - Input: Data from appended company details.  
   - Output: Final edited resume with comments.

7. **Add LangChain Tool Workflow Node: "Resume Editor"**  
   - Link a sub-workflow designed to edit resumes based on job and company info and HR comments.  
   - Accepts input parameter `query` for resume editing instructions.

8. **Add LangChain Tool Workflow Node: "HR Reviwer"**  
   - Link sub-workflow that reviews resumes, returns comments and scores.  
   - Input from Master Agent.

9. **Add Execute Workflow Node: "Generate Cover Letter"**  
   - Link sub-workflow for cover letter generation.  
   - Input: Final edited resume from Master Agent.

10. **Add Google Docs Nodes to save outputs:**  
    - Create three Google Docs nodes: for Resume, Cover Letter, and Interview Coach documents.  
    - Each "Create a document" node: set title dynamically with company name and job title, specify Drive folder ID.  
    - Follow each create node with an "Update a document" node to insert AI-generated content.

11. **Add Execute Workflow Node: "Interview Coach"**  
    - Link sub-workflow for interview coaching using combined data (HR comments, company, job, resume).  
    - Input from Master Agent output.

12. **Add final OpenAI Node: "Message a model"**  
    - Sends a simple message to confirm workflow completion.

13. **Connect nodes in the following order:**  
    - When chat message received → Extract job data → Extract company data → Append job details → Append company details → Master Agent → Generate Cover Letter → Create & Update Resume Doc → Create & Update Cover Letter Doc → Interview Coach → Create & Update Interview Doc → Message a model.

14. **Credentials Setup:**  
    - BrightData API: For scraping LinkedIn data.  
    - Google Sheets OAuth2: For data appending.  
    - Google Docs OAuth2: For document creation and updates.  
    - OpenAI API: For all AI language model interactions.

15. **Sub-Workflows:**  
    - Resume Editor: Accepts job/company data and HR comments; outputs edited resume.  
    - HR Reviewer: Accepts resume and job/company data; outputs comments and score.  
    - Cover Letter Agent: Accepts final resume; outputs cover letter.  
    - Interview Coach: Accepts full context; outputs coaching content.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Trigger: When Chat Message Received starts the workflow upon incoming message with LinkedIn URL.     | Sticky Note on the initial trigger node.                                                      |
| Master Agent acts as the central coordinator, managing iterative AI sub-agents for resume tailoring. | Sticky Note on Master Agent node.                                                             |
| Sub-workflows (Resume Editor, Cover Letter Writer, Interview Coach) are modular agents that return results automatically to Master Agent. | Sticky Note explaining modular AI sub-workflows.                                              |
| AI-generated outputs are saved as Google Docs (resume, cover letter, interview coaching), created then updated with content. | Sticky Note on Google Docs output nodes.                                                      |
| Workflow uses BrightData for web scraping LinkedIn data, OpenAI GPT-4o models for AI processing, and Google Workspace for data persistence and document management. | Overall workflow integration details.                                                         |

---

**Disclaimer:**  
This document is generated exclusively from an automated n8n workflow. All data processed is legal and public, with strict adherence to content policies. No raw JSON or sensitive data is exposed here.