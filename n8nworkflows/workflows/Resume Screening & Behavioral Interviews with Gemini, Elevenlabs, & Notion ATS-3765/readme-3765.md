Resume Screening & Behavioral Interviews with Gemini, Elevenlabs, & Notion ATS

https://n8nworkflows.xyz/workflows/resume-screening---behavioral-interviews-with-gemini--elevenlabs----notion-ats-3765


# Resume Screening & Behavioral Interviews with Gemini, Elevenlabs, & Notion ATS

### 1. Workflow Overview

This workflow automates the recruitment process from candidate application through resume screening to AI-led behavioral interviews and applicant tracking in Notion. It targets HR teams, recruiters, and talent acquisition professionals aiming to streamline early-stage hiring by leveraging AI for resume assessment, voice interviews, and centralized candidate insights.

The workflow is logically divided into two main parts:

- **1.1 Candidate Application & Resume Screening**  
  Handles receiving applications via embedded n8n forms, extracting and parsing resumes, summarizing applicant qualifications, retrieving job descriptions from Notion, scoring candidates against job requirements using an AI model, and creating/updating applicant records in Notion and Google Sheets for compliance tracking.

- **1.2 AI Voice Behavioral Interview & Assessment**  
  Manages the AI voice interview conducted by Elevenlabs, triggered by Notion automations. It receives interview data via webhook, extracts audio, scores the interview using an AI agent with evaluation criteria from Notion, uploads audio to Google Drive, and updates the applicant’s Notion record with detailed AI insights and audio links.

---

### 2. Block-by-Block Analysis

#### 2.1 Candidate Application & Resume Screening

**Overview:**  
This block captures candidate applications from three separate n8n embedded forms (for three job roles), uploads resumes to Google Drive, extracts resume content, summarizes qualifications, fetches the relevant job description from Notion, scores the candidate against the job description using an AI model (Google Gemini), and creates/updates applicant records in Notion and Google Sheets.

**Nodes Involved:**  
- Application Form 1 of 3  
- Application Form 2 of 3  
- Application form 3 of 3  
- Upload CV  
- Resume URL  
- Wait  
- Extract Resume  
- Applicant Personal Data  
- Applicant Qualifications  
- Merge  
- Applicant Summary  
- Get Job Description  
- Job Description Summary  
- Job Description Mapping  
- HR Expert  
- Structured Output Parser  
- Applicant Data Backup  
- Create Applicant Record  
- Get Applicant Record  
- Embed Resume in Notion  

**Node Details:**

- **Application Form 1 of 3 / 2 of 3 / 3 of 3**  
  - *Type:* Form Trigger  
  - *Role:* Entry points for candidate applications for three distinct job roles.  
  - *Config:* Each form has a hidden job code matching the Notion job description; collects Name, Email, and Resume PDF.  
  - *Outputs:* Triggers Upload CV and Application Data nodes.  
  - *Edge Cases:* Form submission failures, invalid file types, missing required fields.

- **Upload CV**  
  - *Type:* Google Drive node  
  - *Role:* Uploads the candidate's resume PDF to a designated Google Drive folder.  
  - *Config:* Authenticated Google Drive account; folder ID set to "[CV]" folder.  
  - *Input:* Resume binary data from form.  
  - *Output:* Provides Google Drive file ID for URL construction.  
  - *Failures:* Auth errors, upload timeouts, file size limits.

- **Resume URL**  
  - *Type:* Set node  
  - *Role:* Constructs a preview URL for the uploaded resume file on Google Drive.  
  - *Input:* File ID from Upload CV.  
  - *Output:* URL string for embedding in Notion.  

- **Wait**  
  - *Type:* Wait node  
  - *Role:* Ensures Google Drive file is fully available before proceeding.  
  - *Input:* Resume URL.  
  - *Output:* Triggers Get Applicant Record.

- **Extract Resume**  
  - *Type:* Extract From File  
  - *Role:* Extracts text content from the uploaded PDF resume.  
  - *Input:* Resume binary data.  
  - *Output:* Plain text for further processing.  
  - *Failures:* PDF parsing errors, corrupted files.

- **Applicant Personal Data**  
  - *Type:* LangChain Information Extractor  
  - *Role:* Extracts personal info (full name, email, phone, city) from resume text.  
  - *Config:* Uses a strict schema; returns N/A if data missing.  
  - *Input:* Resume text.  
  - *Output:* Structured personal data.  
  - *Failures:* Extraction inaccuracies if resume format varies.

- **Applicant Qualifications**  
  - *Type:* LangChain Information Extractor  
  - *Role:* Extracts and summarizes education, job history, skills, total years experience, and recent job title/employer from resume text.  
  - *Config:* Extracts multiple attributes with max word limits.  
  - *Input:* Resume text.  
  - *Output:* Structured qualifications data.  
  - *Edge Cases:* Incomplete resumes, ambiguous skill listings.

- **Merge**  
  - *Type:* Merge node (combine mode)  
  - *Role:* Combines outputs from Applicant Personal Data and Applicant Qualifications into a single JSON object.  
  - *Input:* Two separate extraction outputs.  
  - *Output:* Unified applicant data object.

- **Applicant Summary**  
  - *Type:* LangChain Summarization Chain  
  - *Role:* Generates a concise, conversational summary of education, job history, and skills (max 300 words).  
  - *Input:* Extracted qualification fields from Merge node.  
  - *Output:* Text summary.

- **Get Job Description**  
  - *Type:* Notion node (databasePage getAll)  
  - *Role:* Retrieves the job description matching the applicant’s job code from Notion ATS database.  
  - *Input:* Job code from Application Data.  
  - *Output:* Job description page(s).  
  - *Failures:* Notion API errors, missing job code matches.

- **Job Description Summary**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Summarizes the job description text into less than 250 words.  
  - *Input:* Job description text.  
  - *Output:* Summarized job description string.

- **Job Description Mapping**  
  - *Type:* Set node  
  - *Role:* Maps summarized job description into a variable for downstream AI scoring.  
  - *Input:* Summarized job description.  
  - *Output:* job_description string.

- **HR Expert**  
  - *Type:* LangChain Chain LLM (Google Gemini)  
  - *Role:* Compares applicant summary against job description and scores candidate 1-10 with rationale.  
  - *Config:* Custom prompt instructing scoring and explanation.  
  - *Input:* job_description and applicant summary text.  
  - *Output:* JSON with resume_score and resume_evaluation.  
  - *Edge Cases:* Model misinterpretation, scoring inconsistencies.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses HR Expert output into structured JSON fields (resume_score, resume_evaluation).  
  - *Input:* Raw text from HR Expert.  
  - *Output:* Parsed JSON.

- **Applicant Data Backup**  
  - *Type:* Google Sheets Append  
  - *Role:* Logs applicant data and AI resume assessment to Google Sheets for compliance and reporting.  
  - *Input:* Merged applicant data plus HR Expert output and job description info.  
  - *Output:* Appended row in Google Sheet.  
  - *Failures:* Google Sheets API limits, auth errors.

- **Create Applicant Record**  
  - *Type:* Notion Create Database Page  
  - *Role:* Creates a new applicant record in Notion ATS with all extracted and AI-assessed data including resume score, qualifications, and summary.  
  - *Input:* Combined applicant data and AI scoring.  
  - *Output:* New Notion page ID.  
  - *Failures:* Notion API rate limits, data mapping errors.

- **Get Applicant Record**  
  - *Type:* Notion Get Database Pages  
  - *Role:* Retrieves applicant records missing resume files to update them.  
  - *Output:* List of applicant pages.

- **Embed Resume in Notion**  
  - *Type:* Notion Update Database Page  
  - *Role:* Updates applicant record with Google Drive resume preview URL.  
  - *Input:* Applicant page ID and resume URL.  
  - *Output:* Updated Notion page.

---

#### 2.2 AI Voice Behavioral Interview & Assessment

**Overview:**  
This block handles the AI voice interview conducted by Elevenlabs, triggered by Notion automation. It receives interview data via webhook, extracts and uploads audio, scores the interview transcript using an AI agent with evaluation criteria from Notion, and updates the applicant’s Notion record with detailed AI insights and audio links.

**Nodes Involved:**  
- ElevenLabs Web Hook  
- ai_convo_items  
- AI Agent  
- Evaluation Criteria  
- Google Gemini Model  
- Applicant Record  
- Filter_Notion_db  
- Update_Applicant_Record  
- Extract_Audio  
- Upload Audio to Drive  
- Link Audio in Notion  

**Node Details:**

- **ElevenLabs Web Hook**  
  - *Type:* Webhook (POST)  
  - *Role:* Receives post-call data from Elevenlabs AI Conversation Agent after interview completion.  
  - *Input:* JSON payload containing conversation transcript, evaluation criteria results, and metadata.  
  - *Output:* Triggers ai_convo_items node.  
  - *Failures:* Webhook connectivity, payload format errors.

- **ai_convo_items**  
  - *Type:* Set node  
  - *Role:* Extracts and maps key data elements from webhook payload, including evaluation criteria results, phone number, full name, conversation ID, and full transcript.  
  - *Input:* Webhook JSON body.  
  - *Output:* Structured data for AI Agent and database updates.

- **Evaluation Criteria**  
  - *Type:* Notion Tool node  
  - *Role:* Provides evaluation criteria from Notion database to AI Agent for interview assessment.  
  - *Input:* Filters for non-empty evaluation criteria.  
  - *Output:* Supplies criteria as AI tool input.  
  - *Failures:* Notion API errors, missing criteria.

- **Google Gemini Model**  
  - *Type:* LangChain LLM (Google Gemini)  
  - *Role:* Supports AI Agent with language model capabilities.  
  - *Input:* Full transcript and evaluation criteria.  
  - *Output:* AI assessment text.

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Reviews full interview transcript and evaluation criteria, provides overall interview assessment and score (1-5), confirming use of criteria.  
  - *Config:* Custom system prompt as Talent Acquisition Specialist.  
  - *Input:* Full transcript, evaluation criteria from Notion.  
  - *Output:* Text summary with score.  
  - *Edge Cases:* Transcript quality issues, model misinterpretation.

- **Applicant Record**  
  - *Type:* Notion Get Database Pages  
  - *Role:* Retrieves applicant records to match with interview data.  
  - *Output:* List of applicant pages.

- **Filter_Notion_db**  
  - *Type:* Filter node  
  - *Role:* Matches applicant record by phone number captured during interview.  
  - *Input:* Applicant records and phone number from interview data.  
  - *Output:* Single matching applicant record.

- **Update_Applicant_Record**  
  - *Type:* Notion Update Database Page  
  - *Role:* Updates matched applicant record with interview evaluation criteria results and AI Agent overall interview assessment.  
  - *Input:* Applicant page ID, AI evaluation fields.  
  - *Output:* Updated Notion page.

- **Extract_Audio**  
  - *Type:* HTTP Request  
  - *Role:* Downloads interview audio from Elevenlabs API using conversation ID and API key.  
  - *Input:* Conversation ID from ai_convo_items.  
  - *Output:* Audio binary data.  
  - *Failures:* API key invalid, network errors.

- **Upload Audio to Drive**  
  - *Type:* Google Drive node  
  - *Role:* Uploads interview audio file to designated Google Drive folder.  
  - *Input:* Audio binary data.  
  - *Output:* Google Drive file ID.

- **Link Audio in Notion**  
  - *Type:* Notion Update Database Page  
  - *Role:* Embeds Google Drive audio preview URL into applicant’s Notion record for easy playback by hiring manager.  
  - *Input:* Applicant page ID, Google Drive file URL.  
  - *Output:* Updated Notion page.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                    | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                   |
|-------------------------|----------------------------------|---------------------------------------------------|-------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Application Form 1 of 3  | Form Trigger                     | Candidate application entry for Job 1             | —                             | Upload CV, Application Data  | # Application Initiation; matches job code for role 1                                        |
| Application Form 2 of 3  | Form Trigger                     | Candidate application entry for Job 2             | —                             | Upload CV, Application Data  | # Application Initiation; matches job code for role 2                                        |
| Application form 3 of 3  | Form Trigger                     | Candidate application entry for Job 3             | —                             | Upload CV, Application Data  | # Application Initiation; matches job code for role 3                                        |
| Upload CV                | Google Drive                    | Upload candidate resume PDF                        | Application Form nodes         | Resume URL                  | Stores resumes in Google Drive folder [CV]                                                   |
| Resume URL               | Set                             | Constructs Google Drive preview URL for resume    | Upload CV                     | Wait                        |                                                                                               |
| Wait                    | Wait                            | Waits for file availability                        | Resume URL                    | Get Applicant Record         |                                                                                               |
| Get Applicant Record     | Notion                          | Retrieves applicant records missing resume files  | Wait                         | Embed Resume in Notion       |                                                                                               |
| Embed Resume in Notion   | Notion                          | Updates applicant record with resume file URL     | Get Applicant Record          | —                           |                                                                                               |
| Extract Resume           | Extract From File               | Extracts text from resume PDF                       | Application Data              | Applicant Personal Data, Applicant Qualifications |                                                                                               |
| Applicant Personal Data  | LangChain Information Extractor | Extracts personal info from resume text            | Extract Resume                | Merge                       |                                                                                               |
| Applicant Qualifications | LangChain Information Extractor | Extracts education, job history, skills, experience| Extract Resume                | Merge                       | ## Applicant Qualifications: summary of education, job history, skills, total years experience|
| Merge                   | Merge                           | Combines personal data and qualifications          | Applicant Personal Data, Applicant Qualifications | Applicant Summary           |                                                                                               |
| Applicant Summary        | LangChain Summarization Chain   | Summarizes applicant qualifications concisely     | Merge                        | Get Job Description          | ## Applicant Summary: concise summary of education, job history, skills                       |
| Get Job Description      | Notion                          | Retrieves job description matching job code        | Applicant Summary             | Job Description Summary      | ## Gets Job Description: pulls description matching job code from Notion ATS                  |
| Job Description Summary  | LangChain LLM Chain             | Summarizes job description text                     | Get Job Description           | Job Description Mapping      | ## Job Description Summary: summarizes job description in 250 words or less                   |
| Job Description Mapping  | Set                             | Maps summarized job description for AI scoring     | Job Description Summary       | HR Expert                   |                                                                                               |
| HR Expert                | LangChain Chain LLM             | Scores candidate vs job description with rationale | Job Description Mapping, Applicant Summary | Applicant Data Backup        | ## HR Expert Evaluation: scores candidate 1-10 with rationale                                |
| Structured Output Parser | LangChain Output Parser         | Parses HR Expert output into structured JSON       | HR Expert                    | Applicant Data Backup        |                                                                                               |
| Applicant Data Backup    | Google Sheets Append            | Logs applicant data and AI resume assessment       | Structured Output Parser, Merge, Application Data, Get Job Description | Create Applicant Record      | ## Creates G-Sheets Record: compliance data tracking                                         |
| Create Applicant Record  | Notion Create Database Page     | Creates applicant record in Notion ATS              | Applicant Data Backup         | —                           | ## Creates ATS Record: updates Notion ATS with applicant info and AI assessment               |
| ElevenLabs Web Hook      | Webhook (POST)                  | Receives AI voice interview data from Elevenlabs   | —                            | ai_convo_items              | ## Elevenlabs Trigger: receives post-call data including AI evaluation                       |
| ai_convo_items           | Set                             | Extracts key interview data and evaluation results | ElevenLabs Web Hook           | AI Agent                    | ## Data Mapping: maps conversation data and evaluation criteria                              |
| Evaluation Criteria      | Notion Tool                    | Provides evaluation criteria from Notion to AI Agent | —                            | AI Agent                    |                                                                                               |
| Google Gemini Model      | LangChain LLM                  | Supports AI Agent with language model capabilities | —                            | AI Agent                    |                                                                                               |
| AI Agent                 | LangChain Agent                | Reviews transcript and evaluation criteria, scores interview | ai_convo_items, Evaluation Criteria, Google Gemini Model | Applicant Record             | ## AI Agent Interview Assessment: scores interview 1-5 with summary                         |
| Applicant Record         | Notion Get Database Pages       | Retrieves applicant records for matching            | —                            | Filter_Notion_db            | ## Applicant Tracker: pulls applicant record from Notion db                                 |
| Filter_Notion_db         | Filter                         | Matches applicant by phone number from interview    | Applicant Record, ai_convo_items | Update_Applicant_Record      | ## Applicant ID: matches interview with candidate record                                    |
| Update_Applicant_Record  | Notion Update Database Page     | Updates applicant record with interview scores and rationale | Filter_Notion_db             | Extract_Audio               | ## Update Notion DB: updates applicant record with AI interview assessment                   |
| Extract_Audio            | HTTP Request                   | Downloads interview audio from Elevenlabs API       | Update_Applicant_Record       | Upload Audio to Drive       | ## Conversation Audio: downloads audio for storage                                          |
| Upload Audio to Drive    | Google Drive                   | Uploads interview audio to Google Drive              | Extract_Audio                | Link Audio in Notion        |                                                                                               |
| Link Audio in Notion     | Notion Update Database Page     | Embeds audio file link in applicant Notion record   | Upload Audio to Drive         | —                           | ## Embed audio transcript in Notion: provides easy access to audio playback                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create three Form Trigger nodes** for candidate applications, one per job role:  
   - Configure form fields: Name (required), Email (required, email type), Resume (required, accept .pdf), and a hidden Job Code field matching the Notion job code for each role.  
   - Set form titles and submission confirmation messages accordingly.

2. **Add a Google Drive node ("Upload CV")**:  
   - Authenticate Google Drive OAuth2.  
   - Set target folder to your "[CV]" folder.  
   - Input: Resume file from form trigger.  
   - Output: File ID for URL construction.

3. **Add a Set node ("Resume URL")**:  
   - Construct Google Drive preview URL using file ID from Upload CV node.  
   - Example: `https://drive.google.com/file/d/{{ $json.id }}/preview`

4. **Add a Wait node ("Wait")**:  
   - To ensure file availability before next steps.

5. **Add a Notion node ("Get Applicant Record")**:  
   - Authenticate Notion API.  
   - Retrieve applicant records missing resume files to update.

6. **Add a Notion node ("Embed Resume in Notion")**:  
   - Update applicant record with resume preview URL.

7. **Add an Extract From File node ("Extract Resume")**:  
   - Extract text content from uploaded PDF resume.

8. **Add two LangChain Information Extractor nodes:**  
   - "Applicant Personal Data": extract full name, email, phone, city (strict schema).  
   - "Applicant Qualifications": extract education, job history, skills, experience, recent title/employer.

9. **Add a Merge node ("Merge")**:  
   - Combine outputs of the two extractors.

10. **Add a LangChain Summarization Chain node ("Applicant Summary")**:  
    - Summarize education, job history, and skills in 300 words or less.

11. **Add a Notion node ("Get Job Description")**:  
    - Retrieve job description from Notion ATS database filtered by job code.

12. **Add a LangChain LLM Chain node ("Job Description Summary")**:  
    - Summarize job description text to less than 250 words.

13. **Add a Set node ("Job Description Mapping")**:  
    - Map summarized job description to a variable.

14. **Add a LangChain Chain LLM node ("HR Expert")**:  
    - Use Google Gemini model.  
    - Input: job description and applicant summary.  
    - Prompt: score candidate 1-10 with rationale.  
    - Enable structured output parser.

15. **Add a LangChain Structured Output Parser node ("Structured Output Parser")**:  
    - Parse HR Expert output into JSON fields.

16. **Add a Google Sheets Append node ("Applicant Data Backup")**:  
    - Authenticate Google Sheets OAuth2.  
    - Append applicant data and AI resume assessment for compliance.

17. **Add a Notion Create Database Page node ("Create Applicant Record")**:  
    - Create applicant record with all data and AI scores.

18. **Configure connections:**  
    - Each form triggers Upload CV and Application Data nodes.  
    - Upload CV triggers Resume URL → Wait → Get Applicant Record → Embed Resume in Notion.  
    - Application Data triggers Extract Resume → Applicant Personal Data & Applicant Qualifications → Merge → Applicant Summary → Get Job Description → Job Description Summary → Job Description Mapping → HR Expert → Structured Output Parser → Applicant Data Backup → Create Applicant Record.

---

19. **For AI Voice Behavioral Interview:**

20. **Add a Webhook node ("ElevenLabs Web Hook")**:  
    - HTTP POST method.  
    - Copy production URL for Elevenlabs post-call webhook configuration.

21. **Add a Set node ("ai_convo_items")**:  
    - Map evaluation criteria results, phone number, full name, conversation ID, and full transcript from webhook payload.

22. **Add a Notion Tool node ("Evaluation Criteria")**:  
    - Retrieve evaluation criteria from Notion database.

23. **Add LangChain LLM nodes ("Google Gemini Model")**:  
    - Use Google Gemini model for AI processing.

24. **Add LangChain Agent node ("AI Agent")**:  
    - Input: full transcript and evaluation criteria.  
    - Prompt: act as Talent Acquisition Specialist, score interview 1-5 with summary.

25. **Add Notion Get Database Pages node ("Applicant Record")**:  
    - Retrieve applicant records.

26. **Add Filter node ("Filter_Notion_db")**:  
    - Match applicant record by phone number from interview.

27. **Add Notion Update Database Page node ("Update_Applicant_Record")**:  
    - Update applicant record with interview evaluation and AI assessment.

28. **Add HTTP Request node ("Extract_Audio")**:  
    - Download audio from Elevenlabs API using conversation ID and API key.

29. **Add Google Drive node ("Upload Audio to Drive")**:  
    - Upload audio file to Google Drive "[CV]" folder.

30. **Add Notion Update Database Page node ("Link Audio in Notion")**:  
    - Embed Google Drive audio preview URL in applicant record.

31. **Configure connections:**  
    - ElevenLabs Web Hook → ai_convo_items → AI Agent (with Evaluation Criteria and Google Gemini Model) → Applicant Record → Filter_Notion_db → Update_Applicant_Record → Extract_Audio → Upload Audio to Drive → Link Audio in Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| AI Powered Resume Screening & Voice AI that interviews like a Recruiter!                                                                                                                                                                   | [Video Demo](https://drive.google.com/uc?export=view&id=197WXAPUe0256SJniNEbBQ8A9QNByaLCM)                       |
| AI Insights in Notion dashboard                                                                                                                                                                                                            | [Dashboard Demo](https://drive.google.com/uc?export=view&id=19A7qjEjN0hCUNVwxxsUyh3TUUqYziPg1)                   |
| Click to hear AI Voice Agent in action                                                                                                                                                                                                     | [Audio Example](https://drive.google.com/uc?export=download&id=16YgEnDt4RWU1U7i_la5b3jGNDUcMpvdx)                |
| Notion Template for Career Page and ATS                                                                                                                                                                                                    | [Notion Template](https://www.notion.so/marketplace/templates/ai-recruiter?cr=cre%3Aculturedatasolutions)         |
| Important: Consult State and Country regulations regarding AI Compliance, AI Bias Audits, AI Risk Assessment, and disclosure requirements before deploying AI in recruitment.                                                              | Compliance advisory                                                                                              |
| Workflow supports parallel processing of 3 requisitions; can be extended by duplicating form triggers and adjusting job codes accordingly.                                                                                                | Scalability note                                                                                                |
| Elevenlabs Conversation AI Agent requires configuration of system prompt, interview questions, evaluation criteria, and post-call webhook URL (n8n webhook URL).                                                                           | Elevenlabs setup instructions                                                                                   |
| Google Drive and Google Sheets require OAuth2 credentials with appropriate scopes for file upload and sheet editing.                                                                                                                      | Credential setup                                                                                                |
| Notion API credentials require access to ATS databases for job descriptions, applicant tracker, and evaluation criteria.                                                                                                                  | Credential setup                                                                                                |
| Gemini (Google PaLM) API credentials required for AI language model nodes.                                                                                                                                                                 | Credential setup                                                                                                |
| Gmail and Calendly integrations are referenced for interview scheduling but not included in this workflow JSON.                                                                                                                            | Scheduling integrations                                                                                          |

---

This reference document fully describes the workflow structure, node roles, configurations, and data flows to enable reproduction, modification, and error anticipation for advanced users and AI agents.