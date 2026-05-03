Resume Screening & Candidate Routing with GPT-4o-mini, Jotform, and Google Sheets

https://n8nworkflows.xyz/workflows/resume-screening---candidate-routing-with-gpt-4o-mini--jotform--and-google-sheets-9435


# Resume Screening & Candidate Routing with GPT-4o-mini, Jotform, and Google Sheets

### 1. Workflow Overview

This workflow automates the hiring pipeline by screening job applications submitted via Jotform, parsing resumes with AI (OpenAI GPT-4o-mini), and routing candidates based on AI-driven evaluation scores. It targets HR teams and recruiters seeking to streamline candidate intake, AI-powered resume analysis, intelligent candidate assessment, and decision-based routing for interview invitations, manager review, or polite rejection, while logging all data to Google Sheets for analytics.

The workflow is divided into four main logical blocks:

- **1.1 Application Intake:** Captures job applications from Jotform, extracts candidate data, and downloads the resume file.
- **1.2 AI Resume Parsing:** Converts the resume into a structured AI-readable format and extracts detailed candidate profile information using GPT-4o-mini.
- **1.3 AI Candidate Screening:** Evaluates the parsed candidate data against job requirements using an AI agent, producing structured scoring and recommendations.
- **1.4 Candidate Routing & Actions:** Routes candidates into three paths (Strong Yes, Yes/Maybe, No/Strong No) for sending interview invitations, requesting manager review, or sending rejection emails, respectively, with all results logged into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Application Intake

**Overview:**  
This block triggers upon receiving a job application from Jotform, extracts relevant candidate information, generates a unique application ID, and downloads the candidate's resume for further processing.

**Nodes Involved:**  
- JotForm Trigger  
- Extract Application Data (Set node)  
- Download Resume (HTTP Request node)  
- Sticky Note - Intake (documentation)

**Node Details:**

- **JotForm Trigger**  
  - *Type:* Trigger node for Jotform submissions  
  - *Configuration:* Connected to a specific Jotform form (ID: 252815294070456) with credentials for API access  
  - *Input/Output:* Input is the form submission webhook; output is raw submission JSON  
  - *Failures:* Possible webhook or API auth failures  
- **Extract Application Data (Set node)**  
  - *Type:* Data transformation node  
  - *Configuration:* Extracts fields like candidate name, email, phone, position applied, years of experience, LinkedIn URL, portfolio URL, resume URL, cover letter, generates application ID, and current timestamp  
  - *Expressions:* Uses JavaScript expressions to safely extract nested JSON values with fallback defaults  
  - *Input:* Raw JSON from JotForm Trigger  
  - *Output:* Structured candidate data JSON  
  - *Failures:* Expression evaluation errors if expected fields are missing or malformed  
- **Download Resume (HTTP Request node)**  
  - *Type:* File download node  
  - *Configuration:* Downloads the resume from the extracted URL in the candidate data  
  - *Input:* Resume URL from previous node  
  - *Output:* Binary data of the resume file  
  - *Failures:* URL invalid, network errors, or file not found  
- **Sticky Note - Intake**  
  - *Role:* Documentation block explaining this intake process and key data captured

---

#### 2.2 AI Resume Parsing

**Overview:**  
Processes the downloaded resume file by converting it to base64 for AI consumption and uses GPT-4o-mini to parse and summarize resume contents into structured markdown.

**Nodes Involved:**  
- Process Resume (Code node)  
- AI Resume Parser (OpenAI GPT-4o-mini)  
- Sticky Note - Parsing (documentation)

**Node Details:**

- **Process Resume (Code node)**  
  - *Type:* JavaScript code node for handling binary data  
  - *Configuration:* Checks if resume binary data exists; converts it to base64 string; flags if unavailable or errors occur  
  - *Input:* Binary resume data from Download Resume node  
  - *Output:* JSON with base64 resume and flags indicating availability  
  - *Failures:* Binary data missing or decoding errors  
- **AI Resume Parser (OpenAI node)**  
  - *Type:* OpenAI GPT-4o-mini language model node  
  - *Configuration:* Sends a prompt including structured candidate data and resume content to extract professional summary, skills, experience, education, projects, red flags, and standout qualities formatted in markdown  
  - *Input:* JSON with base64 resume and candidate data  
  - *Output:* AI-generated markdown summary of resume  
  - *Credentials:* OpenAI API key required  
  - *Failures:* API rate limits, auth errors, prompt formatting issues  
- **Sticky Note - Parsing**  
  - *Role:* Documentation about AI parsing logic and expected output format

---

#### 2.3 AI Candidate Screening

**Overview:**  
This block evaluates the AI-parsed candidate profile against specific job requirements using an AI agent. It produces a detailed structured assessment with scores, recommendations, and interview focus areas.

**Nodes Involved:**  
- AI Candidate Screener (Langchain Agent)  
- Structured Output Parser  
- OpenAI Chat Model (supporting node)  
- Sticky Note - Screening (documentation)

**Node Details:**

- **AI Candidate Screener (Langchain Agent node)**  
  - *Type:* AI agent node with a defined prompt for candidate evaluation  
  - *Configuration:* Detailed prompt with job requirements, candidate info, and resume summary; requests structured evaluation including overall score, skills match, experience, cultural fit, red flags, standout qualities, interview focus, salary range, and markdown assessment  
  - *Input:* Candidate data and AI Resume Parser output  
  - *Output:* JSON with structured evaluation fields and detailed markdown text  
  - *Failures:* AI response parsing errors, API failures  
- **Structured Output Parser**  
  - *Type:* Output parser node with JSON schema validation  
  - *Configuration:* Validates AI response against a strict JSON schema defining scores, arrays, strings, and markdown fields  
  - *Input:* Raw AI text output  
  - *Output:* Parsed, strongly typed JSON suitable for downstream logic  
  - *Failures:* Schema mismatch or incomplete AI output  
- **OpenAI Chat Model**  
  - *Type:* Language model node used internally by AI Candidate Screener as the underlying LLM  
  - *Configuration:* Uses GPT-4o-mini  
  - *Credentials:* OpenAI API key  
- **Sticky Note - Screening**  
  - *Role:* Explains evaluation criteria, scoring system, and output structure

---

#### 2.4 Candidate Routing & Actions

**Overview:**  
Routes candidates based on AI recommendation into three paths: Strong Yes (direct interview invite), Yes/Maybe (manager review required), and No/Strong No (send rejection). Each route triggers appropriate emails and logs the candidate data in Google Sheets.

**Nodes Involved:**  
- Strong Yes? (If node)  
- Maybe or Yes? (If node)  
- Send a message (Slack notification node)  
- Send Interview Invitation (Gmail node)  
- Request Manager Review (Gmail node with sendAndWait)  
- Send Rejection Email (Gmail node)  
- Log to Hiring Database (Google Sheets node)  
- Sticky Note - Routing (documentation)  
- Sticky Note - Analytics (documentation)

**Node Details:**

- **Strong Yes? (If node)**  
  - *Type:* Conditional node testing if recommendation == "strong_yes"  
  - *Input:* AI Candidate Screener output  
  - *Output:* True branch leads to Slack alert and interview invitation  
  - *Failures:* Missing or malformed recommendation field  
- **Maybe or Yes? (If node)**  
  - *Type:* Conditional node testing if recommendation == "yes" or "maybe"  
  - *Input:* AI Candidate Screener output  
  - *Output:* True branch leads to manager review email and waits for approval; false branch leads to rejection email  
- **Send a message (Slack node)**  
  - *Type:* Slack message node  
  - *Configuration:* Sends Slack alert to specified channel (channel ID to be configured) about hot candidate  
  - *Failures:* Slack API auth or channel misconfiguration  
- **Send Interview Invitation (Gmail node)**  
  - *Type:* Email node sending personalized interview invitation  
  - *Configuration:* Uses candidate email and name, HTML template with next steps and links  
  - *Failures:* Email sending errors, invalid email address  
- **Request Manager Review (Gmail node with sendAndWait)**  
  - *Type:* Email node that sends detailed candidate assessment to hiring manager and waits for response (approval/rejection)  
  - *Configuration:* Includes candidate info, scores, strengths, concerns, salary estimate, and actionable decision request  
  - *Failures:* Manager non-response, email delivery failures  
- **Send Rejection Email (Gmail node)**  
  - *Type:* Email node sending polite rejection with encouragement and links for other opportunities  
  - *Failures:* Email sending errors  
- **Log to Hiring Database (Google Sheets node)**  
  - *Type:* Data append node  
  - *Configuration:* Appends candidate data, scores, and status to Google Sheets for analytics  
  - *Failures:* Google API auth, sheet ID or permissions errors  
- **Sticky Notes - Routing & Analytics**  
  - *Role:* Document routing logic, actions per path, and analytics metrics logged

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                          | Input Node(s)                | Output Node(s)                     | Sticky Note                                         |
|-------------------------|--------------------------------|----------------------------------------|-----------------------------|----------------------------------|-----------------------------------------------------|
| JotForm Trigger         | Jotform Trigger                | Triggers on job application submission | (Webhook trigger)            | Extract Application Data          | Intake block: Application intake explanation        |
| Extract Application Data| Set                            | Extracts candidate fields from form    | JotForm Trigger             | Download Resume                   | Intake block: Application intake explanation        |
| Download Resume         | HTTP Request                   | Downloads resume file from URL          | Extract Application Data    | Process Resume                   | Intake block: Application intake explanation        |
| Process Resume          | Code                           | Converts resume binary to base64        | Download Resume             | AI Resume Parser                  | Parsing block: AI resume analysis explanation       |
| AI Resume Parser        | OpenAI GPT-4o-mini             | Parses resume content into structured info | Process Resume           | AI Candidate Screener            | Parsing block: AI resume analysis explanation       |
| AI Candidate Screener   | Langchain AI Agent             | Evaluates candidate vs job requirements | AI Resume Parser            | Strong Yes?, Maybe or Yes?        | Screening block: Candidate evaluation explanation   |
| Structured Output Parser| Output Parser (JSON Schema)    | Validates and structures AI output      | AI Candidate Screener       | AI Candidate Screener (parsed output) | Screening block: Candidate evaluation explanation   |
| OpenAI Chat Model       | OpenAI GPT-4o-mini             | LLM for candidate screening agent       | (Internal to AI Candidate Screener) | AI Candidate Screener          | Screening block: Candidate evaluation explanation   |
| Strong Yes?             | If                             | Checks if candidate is strong yes       | AI Candidate Screener       | Send a message                   | Routing block: Smart routing explanation             |
| Send a message          | Slack                         | Sends Slack alert for hot candidate     | Strong Yes? (true branch)   | Send Interview Invitation         | Routing block: Smart routing explanation             |
| Send Interview Invitation| Gmail                         | Sends interview invitation email        | Send a message              | Log to Hiring Database           | Routing block: Smart routing explanation             |
| Maybe or Yes?           | If                             | Checks if candidate is yes or maybe      | AI Candidate Screener       | Request Manager Review, Send Rejection Email | Routing block: Smart routing explanation             |
| Request Manager Review  | Gmail (sendAndWait)            | Sends candidate review email to manager | Maybe or Yes? (true branch) | Log to Hiring Database           | Routing block: Smart routing explanation             |
| Send Rejection Email    | Gmail                         | Sends rejection email to candidate       | Maybe or Yes? (false branch)| Log to Hiring Database           | Routing block: Smart routing explanation             |
| Log to Hiring Database  | Google Sheets                  | Logs candidate and evaluation data       | Send Interview Invitation, Request Manager Review, Send Rejection Email | (end)  | Analytics block: Hiring analytics explanation       |
| Sticky Note - Intake    | Sticky Note                   | Documentation of Application Intake      | -                           | -                                | Application intake explanation                        |
| Sticky Note - Parsing   | Sticky Note                   | Documentation of AI Resume Parsing       | -                           | -                                | AI resume parsing explanation                         |
| Sticky Note - Screening | Sticky Note                   | Documentation of AI Candidate Screening  | -                           | -                                | Candidate screening explanation                       |
| Sticky Note - Routing   | Sticky Note                   | Documentation of Candidate Routing       | -                           | -                                | Smart routing explanation                             |
| Sticky Note - Analytics | Sticky Note                   | Documentation of Hiring Analytics        | -                           | -                                | Hiring analytics explanation                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a JotForm Trigger node:**  
   - Type: Jotform Trigger  
   - Configure with your Jotform API credentials  
   - Select the form ID corresponding to your job application form  
   - This node triggers the workflow upon new form submission  

2. **Add a Set node named "Extract Application Data":**  
   - Extract candidate fields from JSON form input:  
     - candidate_name: combine first and last name fields safely  
     - candidate_email, candidate_phone, position_applied, years_experience, LinkedIn URL, portfolio URL, resume URL, cover letter  
     - application_id: generate unique ID using submission ID and timestamp  
     - application_date: current ISO timestamp  
   - Use expressions to handle missing or optional fields with defaults  

3. **Add an HTTP Request node "Download Resume":**  
   - Configure URL parameter dynamically from extracted resume_url field  
   - No special authentication needed unless files are protected  
   - Set output to binary data for subsequent processing  

4. **Add a Code node "Process Resume":**  
   - Write JavaScript to check for binary resume data presence  
   - Convert binary to base64 string for AI consumption  
   - Provide flags if resume is missing or errors occur  
   - Output JSON with base64 string and flags  

5. **Add an OpenAI node "AI Resume Parser":**  
   - Model: GPT-4o-mini  
   - Credentials: configure OpenAI API key  
   - Message prompt: include candidate extracted data and base64 resume or cover letter  
   - Ask to extract structured markdown summary with professional summary, skills, experience, education, projects, red flags, standout qualities  

6. **Add Langchain Agent node "AI Candidate Screener":**  
   - Model: GPT-4o-mini (via OpenAI Chat Model node)  
   - Use defined prompt with job requirements, candidate info, and parsed resume summary  
   - Request a structured JSON evaluation including scores, recommendation, skills lists, concerns, interview focus, salary estimate, markdown assessment  
   - Credentials: OpenAI API key  

7. **Add Structured Output Parser node:**  
   - Define JSON schema matching expected AI output fields  
   - Parse and validate AI Candidate Screener output JSON  

8. **Add two If nodes for routing:**  
   - "Strong Yes?" node checks if recommendation equals "strong_yes"  
   - "Maybe or Yes?" node checks if recommendation equals "yes" or "maybe"  

9. **Add Slack node "Send a message":**  
   - Configure Slack credentials and target channel  
   - Send alert for strong yes candidates  

10. **Add Gmail node "Send Interview Invitation":**  
    - Configure Gmail OAuth2 credentials  
    - Compose personalized HTML email with candidate name and position, next steps, and links  
    - Connect from Slack message node and Strong Yes? node  

11. **Add Gmail node "Request Manager Review":**  
    - Configure Gmail OAuth2 credentials  
    - Compose detailed email with candidate data, AI scores, strengths, concerns, salary estimate  
    - Use "send and wait" operation to pause workflow for manager decision  
    - Connect from Maybe or Yes? node (true branch)  

12. **Add Gmail node "Send Rejection Email":**  
    - Configure Gmail OAuth2 credentials  
    - Compose polite rejection email with encouragement and useful links  
    - Connect from Maybe or Yes? node (false branch)  

13. **Add Google Sheets node "Log to Hiring Database":**  
    - Configure Google Sheets OAuth2 credentials  
    - Set spreadsheet ID and sheet name  
    - Map columns for candidate data, AI scores, status, recommendation, application date, etc.  
    - Connect from Send Interview Invitation, Request Manager Review, Send Rejection Email nodes  

14. **Link all nodes according to logic:**  
    - JotForm Trigger → Extract Application Data → Download Resume → Process Resume → AI Resume Parser → AI Candidate Screener → Structured Output Parser  
    - Structured Output Parser → Strong Yes?, Maybe or Yes?  
    - Strong Yes? (true) → Send a message (Slack) → Send Interview Invitation → Log to Hiring Database  
    - Maybe or Yes? (true) → Request Manager Review → Log to Hiring Database  
    - Maybe or Yes? (false) → Send Rejection Email → Log to Hiring Database  

15. **Add sticky notes for documentation at each logical block for clarity.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| JotForm used for application intake; create forms for free at https://www.jotform.com/?partner=mediajade | Application intake block documentation                                                                 |
| AI resume parsing uses OpenAI GPT-4o-mini for cost-effective yet detailed extraction                | Parsing block documentation                                                                              |
| Candidate evaluation uses detailed job requirements prompt to ensure comprehensive screening        | Screening block documentation                                                                            |
| Smart routing ensures fast processing of strong candidates and human review for borderline cases    | Routing block documentation                                                                              |
| Google Sheets logs enable building dashboards for application volume, score distributions, and time-to-hire | Analytics block documentation                                                                            |
| Slack channel alerts for immediate attention on hot candidates                                       | Requires Slack channel ID configuration                                                                 |
| Gmail nodes require OAuth2 credentials with sending permissions                                      | Gmail OAuth2 credential setup needed                                                                     |

---

This structured document provides a complete technical reference for the "Resume Screening & Candidate Routing with GPT-4o-mini, Jotform, and Google Sheets" workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.