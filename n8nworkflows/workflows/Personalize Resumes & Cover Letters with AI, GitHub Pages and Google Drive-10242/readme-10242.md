Personalize Resumes & Cover Letters with AI, GitHub Pages and Google Drive

https://n8nworkflows.xyz/workflows/personalize-resumes---cover-letters-with-ai--github-pages-and-google-drive-10242


# Personalize Resumes & Cover Letters with AI, GitHub Pages and Google Drive

### 1. Workflow Overview

This workflow automates the personalization of resumes and cover letters using AI, GitHub Pages, and Google Drive. It targets job seekers who want to automatically tailor their application materials based on job descriptions received via Telegram messages and manage their experience data for reuse. The workflow is structured into three main functional blocks:

- **1.1 Experience Input and Structuring:** Captures unstructured professional experience text, processes it via an AI agent to produce structured JSON data, and stores it in a data table for later use.
  
- **1.2 Job Description Analysis and Cover Letter Generation:** Receives job descriptions via Telegram, compares them with stored experience data, identifies matching skills and tools, and generates a tailored cover letter using AI.

- **1.3 Resume Generation, Hosting, and Tracking:** Converts the AI output into HTML resume format, commits it to a GitHub Pages repository, uploads the resume PDF to Google Drive, and logs generated materials and links into Google Sheets. It also sends a Telegram notification when the process completes.

Supporting components include GitHub workflow files for deployment notifications and styling assets for the resume HTML.

---

### 2. Block-by-Block Analysis

#### 2.1 Experience Input and Structuring

**Overview:**  
This block processes raw experience descriptions into structured professional data and stores it for retrieval and reuse.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Simple Memory  
- Structured Output Parser  
- Insert row (data table)

**Node Details:**  

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Entry point to receive experience text input  
  - Configuration: Default, listens for incoming chat messages to trigger the workflow  
  - Connections: Output → AI Agent  

- **AI Agent**  
  - Type: Langchain Agent (Experience Structuring)  
  - Role: Converts unstructured experience text into structured JSON  
  - Configuration: Uses a system prompt specifying strict formatting rules for JSON output to represent role, summary, tasks, skills, tools, and industry  
  - Inputs: Raw experience text from chat message  
  - Outputs: JSON object with structured experience data  
  - Connections: Output → Insert row  
  - Edge Cases: May output empty JSON if insufficient information; must handle invalid or partial inputs gracefully  
  - Version: 2.2  

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Language model used by AI Agent  
  - Configuration: Model set to “gpt-5” with no additional options  
  - Credentials: OpenAI API key provided  
  - Connections: AI Agent → OpenAI Chat Model (ai_languageModel)  

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Context memory for AI Agent to maintain conversation context over 10 tokens  
  - Connections: AI Agent (ai_memory)  

- **Structured Output Parser**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses the AI Agent's output ensuring it matches the expected JSON schema  
  - Configured with example JSON schema for experience data  
  - Connections: AI Agent (ai_outputParser)  

- **Insert row**  
  - Type: Data Table  
  - Role: Inserts structured experience data into a data table named "experience" for storage  
  - Configured to map JSON fields to corresponding columns  
  - Inputs: Output from AI Agent  
  - Outputs: None (data persistence)  

---

#### 2.2 Job Description Analysis and Cover Letter Generation

**Overview:**  
This block receives job descriptions via Telegram, compares them against stored experience data, identifies matching and additional skills, and generates a personalized application letter.

**Nodes Involved:**  
- Telegram Trigger  
- experience table (Data Table Tool)  
- AI Agent1  
- OpenAI Chat Model1  
- Simple Memory1  
- JSON Output  
- change JSON to HTML (Code node)  
- HTML code  

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages containing job descriptions  
  - Configuration: Monitors 'message' updates  
  - Credentials: Telegram API configured  
  - Connections: Output → AI Agent1  

- **experience table**  
  - Type: Data Table Tool (get operation)  
  - Role: Retrieves all stored experience records from the "experience" data table to provide context  
  - Connections: Output → AI Agent1 (ai_tool)  

- **AI Agent1**  
  - Type: Langchain Agent (Job Analysis and Letter Generation)  
  - Role: Processes received job description, compares with experience data, selects relevant skills and tools, and composes a tailored application letter  
  - Configuration: System prompt defines detailed rules including skill matching, constraints on adding new skills, output JSON format with fields for company, job_description, skills, tools_expertise, and application_letter  
  - Inputs: Telegram message text, experience data  
  - Outputs: Structured JSON with personalized cover letter and matching skills/tools  
  - Connections: Output → change JSON to HTML, JSON Output  
  - Edge Cases: Should handle missing or incomplete job descriptions; avoid including advanced skills not known; ensure valid JSON output  
  - Version: 2.2  

- **OpenAI Chat Model1**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Language model backing AI Agent1  
  - Configuration: Model set to “gpt-5-mini”  
  - Credentials: OpenAI API key configured  
  - Connections: AI Agent1 (ai_languageModel)  

- **Simple Memory1**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains session-based memory keyed by Telegram user id to preserve context per user  
  - Connections: AI Agent1 (ai_memory)  

- **JSON Output**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses AI Agent1 output to confirm JSON structure matches expected schema for job analysis  
  - Connections: AI Agent1 (ai_outputParser)  

- **change JSON to HTML**  
  - Type: Code (JavaScript)  
  - Role: Converts the JSON arrays of skills and tools into HTML formatted paragraphs suitable for embedding in the resume template  
  - Key Logic: Trims LinkedIn job URLs for cleaner display; maps arrays to HTML paragraphs with CSS classes  
  - Inputs: JSON output from AI Agent1, Telegram Trigger URL  
  - Outputs: JSON with HTML strings for skills and tools, plus trimmed URL  
  - Connections: Output → HTML code  

- **HTML code**  
  - Type: HTML Node  
  - Role: Holds the base HTML template for the resume, which will be updated with dynamic content  
  - Inputs: HTML strings from previous node  
  - Outputs: Connected to GitHub commit node  

---

#### 2.3 Resume Generation, Hosting, and Tracking

**Overview:**  
Converts the personalized resume to PDF, uploads it to Google Drive, commits HTML changes to GitHub Pages, appends application data to Google Sheets, and notifies the user via Telegram.

**Nodes Involved:**  
- HTTP Request  
- Upload file (Google Drive)  
- Append or update row in sheet (Google Sheets)  
- Send a text message (Telegram)  
- github (GitHub commit)  
- Append row in sheet (Google Sheets)  

**Node Details:**  

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Sends a POST request to a Chromium-based service to convert the resume URL to PDF  
  - Configuration: URL points to local or internal service at `http://000.00.00:3000/forms/chromium/convert/url` with form-data parameter "url" set to the GitHub Pages resume URL  
  - Inputs: Triggered via Webhook node  
  - Outputs: PDF content URL or file info  
  - Edge Cases: Possible network failure or service unavailability; ensure service endpoint is correct and accessible  
  - Version: 4.2  

- **Upload file (Google Drive)**  
  - Type: Google Drive Node  
  - Role: Uploads the generated PDF resume to a specified Google Drive folder  
  - Configuration: Filename dynamically generated using incoming message content; folder ID fixed for resume storage  
  - Credentials: Google Drive OAuth2 configured  
  - Inputs: PDF file link from HTTP Request  
  - Outputs: Google Drive file metadata including download link  

- **Append or update row in sheet (Google Sheets)**  
  - Type: Google Sheets Node  
  - Role: Records the resume upload link and company name into a Google Sheet, updating if the company already exists  
  - Configuration: Matches rows by "company" column to update or append new row  
  - Credentials: Google Sheets OAuth2 configured  
  - Inputs: Google Drive file link and job/company info  
  - Outputs: None  

- **Send a text message (Telegram)**  
  - Type: Telegram Node  
  - Role: Sends a completion notification to a specified Telegram chat ID  
  - Configuration: Text message "Done..." with attribution appended  
  - Credentials: Telegram API configured  
  - Inputs: Triggered after Google Sheets update  

- **github (GitHub commit)**  
  - Type: GitHub Node  
  - Role: Commits the updated HTML resume to the GitHub repository under `docs/index.html`  
  - Configuration: Repository, owner, and file path fixed; commit message is the company name from AI Agent1 output  
  - Credentials: GitHub API token configured  
  - Inputs: HTML content from "HTML code" node  
  - Outputs: Triggers Append row in sheet  

- **Append row in sheet (Google Sheets)**  
  - Type: Google Sheets Node  
  - Role: Logs the company, application letter, job description, and link from Telegram to a Google Sheet for tracking applications  
  - Configuration: Append-only operation, no row matching  
  - Credentials: Google Sheets OAuth2 configured  
  - Inputs: Parsed AI Agent1 output and Telegram data  
  - Outputs: None  

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                                       | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                                             |
|-------------------------------|---------------------------------------|------------------------------------------------------|------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| When chat message received     | Langchain Chat Trigger                 | Entry point for receiving experience input           |                              | AI Agent                        |                                                                                                                         |
| AI Agent                      | Langchain Agent                       | Converts experience text to structured JSON          | When chat message received    | Insert row                      |                                                                                                                         |
| OpenAI Chat Model             | Langchain LM Chat OpenAI              | Language model for AI Agent                            | AI Agent                     |                                 |                                                                                                                         |
| Simple Memory                 | Langchain Memory Buffer Window        | Context memory for AI Agent                            | AI Agent                     |                                 |                                                                                                                         |
| Structured Output Parser      | Langchain Output Parser Structured    | Parses AI Agent JSON output                            | AI Agent                     |                                 |                                                                                                                         |
| Insert row                   | Data Table                           | Stores structured experience in data table            | AI Agent                     |                                 |                                                                                                                         |
| Telegram Trigger             | Telegram Trigger                      | Receives job description messages                      |                              | AI Agent1                      |                                                                                                                         |
| experience table             | Data Table Tool                      | Retrieves stored experience data                       |                              | AI Agent1                      |                                                                                                                         |
| AI Agent1                   | Langchain Agent                      | Analyzes job description and generates cover letter   | Telegram Trigger, experience table | change JSON to HTML, JSON Output | Sticky note "Job Analyst Agent (Main)" - main agent for analysis and cover letter generation                            |
| OpenAI Chat Model1           | Langchain LM Chat OpenAI              | Language model for AI Agent1                           | AI Agent1                    |                                 |                                                                                                                         |
| Simple Memory1               | Langchain Memory Buffer Window        | Session memory for AI Agent1                           | AI Agent1                    |                                 |                                                                                                                         |
| JSON Output                 | Langchain Output Parser Structured    | Parses AI Agent1 output JSON                           | AI Agent1                    |                                 |                                                                                                                         |
| change JSON to HTML          | Code (JavaScript)                    | Converts JSON skills/tools to HTML for resume         | AI Agent1, Telegram Trigger  | HTML code                      |                                                                                                                         |
| HTML code                   | HTML Node                           | Base resume HTML template                              | change JSON to HTML          | github                        | Sticky Note3: "HTML template - editable with LLM assistance"                                                           |
| HTTP Request                | HTTP Request                       | Converts resume URL to PDF via Chromium service       | Webhook                     | Upload file                   | Sticky Note2: "Resume Download and Link Transfer" - downloads resume, uploads, and records links                         |
| Upload file                 | Google Drive                       | Uploads PDF resume to Google Drive                     | HTTP Request                | Append or update row in sheet |                                                                                                                         |
| Append or update row in sheet | Google Sheets                      | Updates sheet with resume link and company            | Upload file                 | Send a text message           |                                                                                                                         |
| Send a text message          | Telegram                           | Notifies user of workflow completion                   | Append or update row in sheet |                              |                                                                                                                         |
| github                      | GitHub                            | Commits updated HTML resume to GitHub Pages           | HTML code                   | Append row in sheet           |                                                                                                                         |
| Append row in sheet          | Google Sheets                      | Logs application details to Google Sheets              | github                      |                              |                                                                                                                         |
| When clicking ‘Execute workflow’ | Manual Trigger                    | Starts workflow to deploy CSS and GitHub workflows    |                            | css +yml code                 | Sticky Note1: "GITHUB SETUP - run once to add style.css and notify workflow"                                           |
| css +yml code               | Set Node                          | Sets CSS and YAML content for GitHub repo              | When clicking ‘Execute workflow’ | style.css1                   |                                                                                                                         |
| style.css1                   | GitHub                            | Commits style.css to GitHub                             | css +yml code               | index.html setup              |                                                                                                                         |
| index.html setup             | GitHub                            | Commits index.html to GitHub                            | style.css1                  | yml setup                    |                                                                                                                         |
| yml setup                   | GitHub                            | Commits GitHub Actions YAML workflow                   | index.html setup            |                              |                                                                                                                         |
| Sticky Note                 | Sticky Note                      | Experience Input Agent explanation                      |                            |                              | Sticky Note: "Experience Input Agent" explains data storage and RAG use                                                |
| Sticky Note2                | Sticky Note                      | Resume download and link transfer explanation          |                            |                              |                                                                                                                         |
| Sticky Note3                | Sticky Note                      | HTML template editing instructions                      |                            |                              |                                                                                                                         |
| style.css                   | Sticky Note                      | Job Analyst Agent main block explanation                |                            |                              |                                                                                                                         |
| Sticky Note1                | Sticky Note                      | GitHub setup instructions                               |                            |                              |                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Experience Input Trigger:**  
   - Add a **Langchain Chat Trigger** node named "When chat message received" to receive raw experience text.

2. **Add AI Agent for Experience Structuring:**  
   - Add a **Langchain Agent** node "AI Agent" connected to the chat trigger.  
   - Configure system prompt to convert unstructured experience into JSON with fields: role, summary, tasks, skills, tools, industry.  
   - Use **OpenAI Chat Model** node with "gpt-5" model credentials linked to the agent.  
   - Add a **Simple Memory** node with a 10-token window for context linked to the agent.  
   - Add a **Structured Output Parser** node with the JSON schema for experience data linked to the agent output parser.

3. **Store Experience Data:**  
   - Add a **Data Table** node "Insert row" configured to insert structured experience JSON into a table named "experience".

4. **Set Up Job Description Reception:**  
   - Add a **Telegram Trigger** configured to listen for new messages containing job descriptions.

5. **Retrieve Stored Experience Data:**  
   - Add a **Data Table Tool** node "experience table" set to `get` all records from the "experience" table.

6. **Add AI Agent for Job Analysis and Letter Generation:**  
   - Add **Langchain Agent** node "AI Agent1" connected to Telegram Trigger and experience table (the latter connected to agent’s ai_tool input).  
   - Configure system prompt to analyze job description vs. experience data, select relevant skills/tools, generate a personalized cover letter in JSON format with company, job_description, skills, tools_expertise, application_letter.  
   - Link **OpenAI Chat Model1** node with "gpt-5-mini" model credentials to AI Agent1.  
   - Add **Simple Memory1** node keyed by Telegram user ID for session memory.  
   - Add **JSON Output** parser node configured with the expected JSON schema connected to AI Agent1’s output parser.

7. **Convert JSON Skills and Tools to HTML:**  
   - Add a **Code** node "change JSON to HTML" connected to AI Agent1 output and Telegram Trigger, implementing JavaScript to output HTML paragraphs for skills and tools arrays and trim LinkedIn URLs.

8. **Prepare Resume HTML Template:**  
   - Add an **HTML** node "HTML code" containing the resume template HTML, connected to the Code node.

9. **Commit Resume HTML to GitHub:**  
   - Add a **GitHub** node "github" configured to commit changes to `docs/index.html` in the repository.  
   - Use company name from AI Agent1 output as commit message.  
   - Connect HTML node to GitHub node.

10. **Generate PDF Resume:**  
    - Add a **Webhook** node as a manual or event trigger for PDF generation.  
    - Add an **HTTP Request** node configured to POST to Chromium PDF conversion service with the GitHub Pages resume URL.  
    - Connect Webhook → HTTP Request.

11. **Upload PDF to Google Drive:**  
    - Add a **Google Drive** node "Upload file" configured to upload the PDF with a dynamic filename into a specific folder.  
    - Connect HTTP Request → Upload file.

12. **Update Google Sheets with Resume Link:**  
    - Add a **Google Sheets** node "Append or update row in sheet" to insert or update resume link and company info, keyed by company.  
    - Connect Upload file → Google Sheets.

13. **Log Application Data:**  
    - Add another **Google Sheets** node "Append row in sheet" to log company, cover letter, job description, and link from Telegram.  
    - Connect GitHub node to this node for tracking.

14. **Notify Completion via Telegram:**  
    - Add a **Telegram** node "Send a text message" to notify a predefined chat ID of workflow completion.  
    - Connect Google Sheets update node to Telegram node.

15. **Add Manual Trigger for GitHub Setup:**  
    - Add a **Manual Trigger** node "When clicking ‘Execute workflow’" for initial setup steps.

16. **Commit CSS and GitHub Workflow Files:**  
    - Add a **Set** node "css +yml code" containing CSS and GitHub Actions YAML content.  
    - Add **GitHub** nodes "style.css1", "index.html setup", and "yml setup" to commit CSS, HTML, and GitHub workflow files to the repository sequentially.  
    - Connect Manual Trigger → Set → style.css1 → index.html setup → yml setup.

17. **Add Sticky Notes:**  
    - Add sticky notes to document blocks and instructions as per the workflow (Experience Input Agent, Job Analyst Agent (Main), Resume Download and Link Transfer, HTML template, GitHub setup).

18. **Configure Credentials:**  
    - Ensure OpenAI API keys are set for OpenAI Chat Model nodes  
    - Configure Google Drive and Google Sheets OAuth2 credentials  
    - Set up Telegram API credentials for trigger and message nodes  
    - Configure GitHub API token with repository access for commit nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Experience Input Agent is a simple database tool for storing experience data, which can be upgraded to a vector database for efficient retrieval and token use.| Sticky Note in workflow describing experience data handling.                                                                                                               |
| Job Analyst Agent (Main) is the core AI agent that analyzes job descriptions, personalizes cover letters, and matches skills/tool expertise accordingly.       | Sticky Note labeling the main AI agent block.                                                                                                                               |
| Resume Download and Link Transfer downloads the generated resume, uploads it to Google Drive, and records the link in Google Sheets for tracking.             | Sticky Note describing the resume hosting and tracking process.                                                                                                             |
| GitHub Setup instructions include running once to add style.css and notify-n8n.yml GitHub Actions workflow with the correct webhook URL.                      | Sticky Note with setup instructions for GitHub repository and CI/CD workflow.                                                                                               |
| The CSS and HTML template can be edited to customize the resume appearance; LLM assistance can be used for edits.                                              | Sticky Note advising on customization of the resume template.                                                                                                              |
| GitHub Actions workflow `notify-n8n.yml` posts deployment notifications to an n8n webhook URL upon successful GitHub Pages deploy.                             | Workflow YAML content included in the "css +yml code" node for integration.                                                                                                  |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, respecting all applicable content policies and legal restrictions. All data manipulated is legal and publicly accessible.