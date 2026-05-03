Screen Job Applicants with Gemini AI: Jotform to Notion Hiring Pipeline

https://n8nworkflows.xyz/workflows/screen-job-applicants-with-gemini-ai--jotform-to-notion-hiring-pipeline-9955


# Screen Job Applicants with Gemini AI: Jotform to Notion Hiring Pipeline

### 1. Workflow Overview

This workflow automates the screening process of job applicants submitted via a JotForm form, leveraging Google Gemini AI to analyze candidate fit and storing results in Notion for hiring pipeline management. It targets HR teams and recruiters who want to streamline candidate evaluation by integrating form submissions, AI-driven resume analysis, and collaborative tools (Notion and Slack).

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and Initial Processing:** Trigger on JotForm submission, download and extract resume text, send confirmation email to candidate.
- **1.2 Job Data Retrieval:** Query Notion to find the job description and position details matching the applicant’s selected position.
- **1.3 AI Candidate Analysis:** Combine resume text, cover letter, and job description; send to Google Gemini AI for structured candidate evaluation.
- **1.4 Conditional Candidate Handling:** Based on AI "fit score," either create a candidate record in Notion and alert hiring team or ignore low-fit candidates.
- **1.5 Notification:** Send Slack notification to hiring team about high-quality candidates.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initial Processing

**Overview:**  
This block receives new job applications via JotForm, downloads the uploaded resume PDF, extracts its text, and sends a confirmation email to the applicant.

**Nodes Involved:**  
- JotForm Trigger  
- Download Resume PDF  
- Read Resume Text  
- Send Confirmation Email  

**Node Details:**

- **JotForm Trigger**  
  - *Type:* JotForm Trigger (Webhook)  
  - *Role:* Listens for form submissions on a specified JotForm.  
  - *Configuration:*  
    - Form ID connected to a job application form.  
    - "Resolve Data" is false to retain question IDs for file URLs and other fields.  
  - *Expressions:* Uses question IDs like `q3_fullName`, `q4_email`, `uploadYour` for form fields.  
  - *Input:* N/A (trigger node)  
  - *Output:* Emits form submission data with raw question IDs.  
  - *Edge Cases:* Form field ID mismatch, webhook misconfiguration, JotForm API credentials invalid or expired.

- **Download Resume PDF**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the candidate’s resume PDF file from the private URL provided by JotForm.  
  - *Configuration:*  
    - URL dynamically set from `{{$json.uploadYour[0]}}` (file upload field).  
    - Authentication via JotForm API Key (query parameter).  
    - Response format set to "file" to receive raw PDF data.  
  - *Input:* Resume file URL from JotForm Trigger.  
  - *Output:* Raw PDF file data.  
  - *Edge Cases:* Invalid or expired file URL, authentication failure, file download timeout or corruption.

- **Read Resume Text**  
  - *Type:* Extract From File  
  - *Role:* Extracts plain text from the downloaded PDF resume.  
  - *Configuration:* Operation set to "pdf" to parse PDF content.  
  - *Input:* Raw PDF file from Download Resume PDF.  
  - *Output:* Extracted plain text of resume content.  
  - *Edge Cases:* Corrupted PDF, unsupported PDF format, extraction errors.

- **Send Confirmation Email**  
  - *Type:* Gmail Node  
  - *Role:* Sends a personalized confirmation email to the applicant.  
  - *Configuration:*  
    - Recipient email dynamically taken from form submission (`q4_email`).  
    - Email subject and body use expressions to include candidate name and position applied for.  
    - Uses Gmail OAuth2 credentials.  
  - *Input:* JotForm Trigger data.  
  - *Output:* Confirmation email sent confirmation.  
  - *Edge Cases:* Invalid email address, Gmail API quota limits, OAuth token expiry.

---

#### 2.2 Job Data Retrieval

**Overview:**  
This block retrieves the job posting details from Notion to obtain the job description and the Notion page ID of the applied position for further processing.

**Nodes Involved:**  
- Find Job in Notion  

**Node Details:**

- **Find Job in Notion**  
  - *Type:* Notion Node (Database Page GetAll)  
  - *Role:* Searches "Open Positions" Notion database for the page matching the candidate’s applied position.  
  - *Configuration:*  
    - Database ID set to "Open Positions" database.  
    - Filter on the "Name" property equals the position name from JotForm (`q7_positionApplying`).  
    - Returns all matching pages (should be one).  
  - *Input:* JotForm Trigger data (position applied for).  
  - *Output:* Notion page data including full job description and page ID.  
  - *Edge Cases:* Position name mismatch, Notion API rate limits, invalid database ID, missing permissions.

---

#### 2.3 AI Candidate Analysis

**Overview:**  
This block combines the resume text, cover letter, and job description and sends them to Google Gemini AI to generate a structured analysis including a summary, fit score, and key skills.

**Nodes Involved:**  
- Combine Data  
- Google Gemini Chat Model  
- Structured Output Parser  
- AI Candidate Analysis  

**Node Details:**

- **Combine Data**  
  - *Type:* Merge Node  
  - *Role:* Waits until both the Notion job description and extracted resume text are available, then combines them to form the input for AI analysis.  
  - *Configuration:* Mode "combine" by position (merges inputs by their position in the input array).  
  - *Input:* Outputs from Find Job in Notion and Read Resume Text.  
  - *Output:* Combined JSON with job description and resume text.  
  - *Edge Cases:* One input missing or delayed, data format mismatch.

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini AI Node  
  - *Role:* Provides the underlying AI language model for candidate analysis.  
  - *Configuration:* Uses Google PaLM API credentials. No extra options configured.  
  - *Input:* Prompt text from AI Candidate Analysis node.  
  - *Output:* Raw AI response.  
  - *Edge Cases:* API quota exceeded, network timeouts, invalid credentials.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI raw output into a structured JSON format with keys: `ai_summary`, `ai_fit_score`, and `key_skills`.  
  - *Configuration:* JSON schema example provided for validation.  
  - *Input:* AI raw text from Google Gemini Chat Model.  
  - *Output:* Parsed JSON object with candidate analysis.  
  - *Edge Cases:* AI output malformed or non-JSON, parsing errors.

- **AI Candidate Analysis**  
  - *Type:* LangChain Agent Node  
  - *Role:* Core node orchestrating the AI prompt, sending combined candidate data and job description, and receiving parsed structured output.  
  - *Configuration:*  
    - Custom prompt defines instructions for AI to act as an expert HR screener, requesting JSON output with summary, score, and skills.  
    - Uses expressions to include Job Description, Cover Letter (from JotForm, note QID must be updated), and Resume Text.  
    - Output parser linked to Structured Output Parser node.  
  - *Input:* Combined data from Combine Data node.  
  - *Output:* Structured analysis JSON.  
  - *Edge Cases:* Incorrect prompt, invalid question IDs, AI service errors.

---

#### 2.4 Conditional Candidate Handling

**Overview:**  
Based on the AI fit score, this block decides whether to create a candidate entry in Notion and notify the hiring team or disregard the candidate.

**Nodes Involved:**  
- Score > 40? (If Node)  
- Create Candidate in Notion  
- Ignore (Score < 40)  

**Node Details:**

- **Score > 40?**  
  - *Type:* If Node  
  - *Role:* Evaluates if the AI fit score is greater than or equal to 40.  
  - *Configuration:* Checks if `ai_fit_score` >= 40 in the AI Candidate Analysis output JSON.  
  - *Input:* AI Candidate Analysis output.  
  - *Output:* Routes to Create Candidate or Ignore nodes based on condition.  
  - *Edge Cases:* Missing or malformed score value.

- **Create Candidate in Notion**  
  - *Type:* Notion Node (Database Page Create)  
  - *Role:* Creates a new candidate record in the "Candidates" Notion database with all relevant data.  
  - *Configuration:*  
    - Database ID set to "Candidates" database.  
    - Properties populated include: Candidate Name, Email, Phone, AI Summary, AI Fit Score, Key Skills, Position (relation to Notion job page), and Resume file URL.  
    - Values dynamically sourced from JotForm data, AI output, and Notion job page.  
  - *Input:* Passed from Score > 40? node.  
  - *Output:* Confirmation of candidate creation.  
  - *Edge Cases:* Notion API limits, invalid property mapping, permission errors.

- **Ignore (Score < 40)**  
  - *Type:* NoOp Node  
  - *Role:* Does nothing, effectively ignoring candidates with low AI fit score.  
  - *Configuration:* No parameters.  
  - *Input:* Passed from Score > 40? node.  
  - *Output:* None.  
  - *Edge Cases:* None.

---

#### 2.5 Notification

**Overview:**  
Once a candidate is created in Notion, this block sends an alert message to the hiring team on Slack with candidate summary and a link to the Notion record.

**Nodes Involved:**  
- Alert Hiring Team  

**Node Details:**

- **Alert Hiring Team**  
  - *Type:* Slack Node (Send Message)  
  - *Role:* Posts a formatted message in a Slack channel to notify hiring team members of a new high-quality candidate.  
  - *Configuration:*  
    - Channel ID set to a specific Slack channel (e.g., hiring team).  
    - Message dynamically constructed with candidate name, position, AI fit score, summary, and Notion URL.  
  - *Input:* Output of Create Candidate in Notion (includes Notion URL).  
  - *Output:* Confirmation message sent to Slack.  
  - *Edge Cases:* Slack API rate limits, invalid channel ID, authentication errors.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                          | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                        |
|-------------------------|-------------------------------------|----------------------------------------|----------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger         | JotForm Trigger                     | Initiate workflow on form submission   | N/A                        | Find Job in Notion, Download Resume PDF, Send Confirmation Email | ## 🚪 Configure Your Jotform Form — Details on required form fields and how to update question IDs               |
| Download Resume PDF     | HTTP Request                       | Download resume PDF file                | JotForm Trigger            | Read Resume Text               | ## Download and Read the Resume — Why and how to configure file download                                         |
| Read Resume Text        | Extract From File                   | Extract text from downloaded PDF       | Download Resume PDF         | Combine Data                  | ## Download and Read the Resume — See above                                                                        |
| Send Confirmation Email | Gmail                             | Send confirmation email to applicant  | JotForm Trigger            | N/A                           |                                                                                                                   |
| Find Job in Notion      | Notion (GetAll)                    | Retrieve job description and ID        | JotForm Trigger            | Combine Data                  | ## 🎯 Find Job in Notion — Instructions on database and filter setup                                             |
| Combine Data            | Merge                             | Combine job description and resume text| Find Job in Notion, Read Resume Text | AI Candidate Analysis          | ### Why — Technical note to explain data synchronization                                                          |
| Google Gemini Chat Model| LangChain LLM (Google Gemini)      | AI language model for candidate analysis| AI Candidate Analysis (ai_languageModel) | AI Candidate Analysis          | ## 🧠 The AI Brain (Candidate Analysis) — Explains AI analysis process and prompt customizations                   |
| Structured Output Parser| LangChain Output Parser             | Parse AI output into structured JSON  | AI Candidate Analysis (ai_outputParser) | AI Candidate Analysis          | ## 🧠 The AI Brain (Candidate Analysis) — See above                                                                 |
| AI Candidate Analysis   | LangChain Agent                    | Orchestrate AI screening prompt        | Combine Data               | Score > 40?                   | ## 🧠 The AI Brain (Candidate Analysis) — See above                                                                 |
| Score > 40?             | If                                | Check if candidate fit score meets threshold | AI Candidate Analysis       | Create Candidate in Notion, Ignore (Score < 40) |                                                                                                                   |
| Create Candidate in Notion | Notion (Create)                  | Add candidate to Notion database       | Score > 40?                | Alert Hiring Team             | ## Final Notion Output Preview — Includes screenshot of expected Notion output                                     |
| Ignore (Score < 40)     | NoOp                              | Ignore low-fit candidates               | Score > 40?                | N/A                           |                                                                                                                   |
| Alert Hiring Team       | Slack                            | Notify hiring team on Slack             | Create Candidate in Notion | N/A                           |                                                                                                                   |
| Sticky Note             | Sticky Note                      | Documentation and guidance notes       | N/A                        | N/A                           | Various detailed instructions and screenshots (multiple sticky notes throughout workflow)                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node:**  
   - Node type: JotForm Trigger  
   - Configure with your JotForm API credentials.  
   - Set Form ID to your job application form.  
   - Set "Resolve Data" to false to access raw question IDs.  
   - Note: Submit a test form to capture your unique field question IDs.

2. **Create Download Resume PDF Node:**  
   - Node type: HTTP Request  
   - URL expression: `={{ $json.<yourFileUploadFieldQID>[0] }}` (replace with your form’s file upload field ID).  
   - Authentication: Use HTTP Query Auth with your JotForm API key.  
   - Set response format to "file".  
   - Connect input from JotForm Trigger.

3. **Create Read Resume Text Node:**  
   - Node type: Extract From File  
   - Operation: PDF  
   - Connect input from Download Resume PDF.

4. **Create Send Confirmation Email Node:**  
   - Node type: Gmail  
   - Use OAuth2 credentials for Gmail.  
   - Configure recipient email as `={{ $('JotForm Trigger').item.json.<yourEmailFieldQID> }}`.  
   - Customize email subject and body with candidate’s name and position applied for using your field IDs.  
   - Connect input from JotForm Trigger.

5. **Create Find Job in Notion Node:**  
   - Node type: Notion (GetAll database pages)  
   - Select your "Open Positions" database.  
   - Add filter: Property "Name" equals `={{ $('JotForm Trigger').item.json.<yourPositionFieldQID> }}`.  
   - Connect input from JotForm Trigger.

6. **Create Combine Data Node:**  
   - Node type: Merge  
   - Mode: Combine by position  
   - Connect inputs from Find Job in Notion (first input) and Read Resume Text (second input).

7. **Create Google Gemini Chat Model Node:**  
   - Node type: LangChain Google Gemini  
   - Authenticate with Google PaLM API credentials.  
   - No additional parameters needed.  

8. **Create Structured Output Parser Node:**  
   - Node type: LangChain Structured Output Parser  
   - Provide JSON schema example with `ai_summary`, `ai_fit_score`, and `key_skills`.

9. **Create AI Candidate Analysis Node:**  
   - Node type: LangChain Agent  
   - Set prompt text to instruct Gemini AI to analyze job description, cover letter, and resume text; output JSON with summary, score, skills.  
   - Replace cover letter field ID with your form’s appropriate question ID.  
   - Connect AI language model to Google Gemini Chat Model node.  
   - Connect AI output parser to Structured Output Parser node.  
   - Connect input from Combine Data node.

10. **Create Score > 40? Node:**  
    - Node type: If  
    - Configure condition: `{{ $json.output.ai_fit_score }} >= 40` (number comparison).  
    - Connect input from AI Candidate Analysis.

11. **Create Create Candidate in Notion Node:**  
    - Node type: Notion (Create database page)  
    - Select your "Candidates" database.  
    - Map properties: Candidate Name, Email, Phone, AI Summary, AI Fit Score, Key Skills, Position (relation to job page), Resume file URL. Use expressions referencing JotForm Trigger fields, AI output, and Find Job in Notion output.  
    - Connect "true" branch from Score > 40?.

12. **Create Ignore (Score < 40) Node:**  
    - Node type: NoOp  
    - Connect "false" branch from Score > 40?.

13. **Create Alert Hiring Team Node:**  
    - Node type: Slack  
    - Authenticate with Slack API credentials.  
    - Configure channel ID for hiring notifications.  
    - Compose message including candidate name, position, AI fit score, summary, and Notion candidate page URL.  
    - Connect input from Create Candidate in Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow requires careful updating of JotForm question IDs after submitting a test form to capture your unique field IDs. Failure to update these will cause the workflow to malfunction.                                                                                                                                                                                                                                                                                                 | Sticky Note on JotForm Trigger node configuration                                               |
| Limit file upload field in JotForm to PDF format only to ensure compatibility with the resume text extraction node.                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note on Download Resume PDF and Read Resume Text                                         |
| The "Position Applying For" dropdown options in JotForm must exactly match the "Name" pages in your Notion "Open Positions" database to ensure accurate job lookup.                                                                                                                                                                                                                                                                                                                          | Sticky Note on Find Job in Notion                                                               |
| The AI prompt in the AI Candidate Analysis node is critical and must be customized with the correct cover letter question ID from your form. This node converts the workflow from data collection to decision making.                                                                                                                                                                                                                                                                        | Sticky Note on AI Candidate Analysis block                                                     |
| Screenshots of the expected JotForm and Notion outputs are included as sticky notes in the workflow for visual reference.                                                                                                                                                                                                                                                                                                                                                                      | Sticky Notes with preview images                                                                |
| To use Google Gemini AI, you need valid Google PaLM API credentials configured in n8n. Similarly, Gmail OAuth2 and Slack API credentials must be configured for email and notification nodes to work.                                                                                                                                                                                                                                                                                        | Credentials setup instructions                                                                 |
| The workflow uses a threshold fit score of 40 to filter candidates. You can adjust this value in the "Score > 40?" node if desired.                                                                                                                                                                                                                                                                                                                                                            | Configurable threshold in If node                                                              |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.