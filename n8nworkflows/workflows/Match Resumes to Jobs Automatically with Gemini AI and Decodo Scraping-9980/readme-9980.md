Match Resumes to Jobs Automatically with Gemini AI and Decodo Scraping

https://n8nworkflows.xyz/workflows/match-resumes-to-jobs-automatically-with-gemini-ai-and-decodo-scraping-9980


# Match Resumes to Jobs Automatically with Gemini AI and Decodo Scraping

### 1. Workflow Overview

This workflow automates the process of matching candidate resumes to relevant job opportunities by leveraging AI-powered analysis and web scraping tools. It is designed primarily for recruiters, hiring managers, and talent teams who need to efficiently shortlist candidates against available jobs. The workflow includes the following logical blocks:

- **1.1 Input Reception:** Accepts candidate data from a web intake form including email, short summary, resume URL, and LinkedIn profile URL.
- **1.2 Data Parsing & Enrichment:** Extracts and summarizes candidate information from the resume and LinkedIn profile using Google Gemini AI and Decodo scraping.
- **1.3 Profile Assembly:** Merges insights from multiple sources into a comprehensive candidate profile.
- **1.4 Job Scraping & Matching:** Uses Decodo to scrape job boards and Gemini AI to analyze and score job matches based on candidate profile.
- **1.5 Reporting & Notification:** Generates an HTML report of top job matches and emails it to the candidate.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives candidate submission via a webhook, extracts form fields, and routes workflow based on the presence of a LinkedIn URL.

**Nodes Involved:**  
- Webhook: Receive Intake Form (POST)  
- Set: Capture Form Fields  
- Switch: Has LinkedIn URL?

**Node Details:**

- **Webhook: Receive Intake Form (POST)**
  - Type & Role: HTTP webhook node to receive POST requests from the intake form.
  - Config: Listens on a unique webhook path; expects form data with fields.
  - Inputs: External HTTP POST request.
  - Outputs: JSON object containing form submission.
  - Edge Cases: Missing or malformed data; webhook path misconfiguration.
  
- **Set: Capture Form Fields**
  - Type & Role: Extracts and assigns specific form fields from webhook data to named variables.
  - Config: Maps email, short summary, resume URL, LinkedIn URL from received JSON fields.
  - Expressions: Uses expressions like `{{$json.body.data.fields[n].value}}` to extract data.
  - Inputs: Webhook output.
  - Outputs: JSON with user_email, short_summary, resume_url_link, user_linkedin.
  - Edge Cases: Form structure changes; missing fields; empty LinkedIn URL.
  
- **Switch: Has LinkedIn URL?**
  - Type & Role: Conditional routing based on whether a LinkedIn URL is provided.
  - Config: Checks if `user_linkedin` is empty or not.
  - Outputs: Two branches - “If LinkedIn User is Exist” and “If LinkedIn User is not Exist.”
  - Inputs: Set node output.
  - Outputs: Routes to LinkedIn scraping or directly to resume parsing.
  - Edge Cases: Invalid LinkedIn URL formats, empty strings, missing fields.

---

#### 2.2 Data Parsing & Enrichment

**Overview:**  
Extracts detailed candidate information either by scraping LinkedIn profile with Decodo or directly parsing the resume with Gemini AI. Further extracts structured LinkedIn details using a language model.

**Nodes Involved:**  
- Decodo: Scrape LinkedIn Profile  
- Gemini: Parse Resume  
- LLM: Extract LinkedIn Details  
- Gemini Model: Profile Extraction

**Node Details:**

- **Decodo: Scrape LinkedIn Profile**
  - Type & Role: Community Decodo node for scraping LinkedIn profile data.
  - Config: Uses candidate LinkedIn URL; geo parameter set to "=" (default).
  - Credentials: Requires Decodo API credentials.
  - Inputs: Switch node (LinkedIn exists branch).
  - Outputs: Raw LinkedIn profile data.
  - Edge Cases: API limits, invalid URLs, Decodo downtime, scraping errors.
  
- **Gemini: Parse Resume**
  - Type & Role: Google Gemini AI node for document parsing.
  - Config: Processes resume URL to extract textual content and summary.
  - Resource: Document type; uses Gemini 2.5 flash model.
  - Credentials: Google Palm API credential required.
  - Inputs: Switch node (LinkedIn missing branch).
  - Outputs: Parsed resume content.
  - Edge Cases: Unreachable resume URL, unsupported file formats, parsing errors.
  
- **LLM: Extract LinkedIn Details**
  - Type & Role: Langchain information extractor to parse structured LinkedIn attributes.
  - Config: Extracts multiple attributes like name, headline, location, experience, education, certifications, volunteer work from Decodo output.
  - Inputs: Decodo node output.
  - Outputs: Structured JSON with LinkedIn fields.
  - Edge Cases: Missing or incomplete data, extraction failures.
  
- **Gemini Model: Profile Extraction**
  - Type & Role: Google Gemini chat model to extract profile summary.
  - Config: No specific options; used as AI language model to support LinkedIn parsing.
  - Inputs: Decodo output (via LLM extraction).
  - Outputs: Enhanced profile extraction.
  - Credentials: Google Palm API.
  - Edge Cases: Rate limits, API errors.

---

#### 2.3 Profile Assembly

**Overview:**  
Combines parsed resume data and LinkedIn profile details into a single candidate profile JSON object for further processing.

**Nodes Involved:**  
- Merge: Assemble Candidate Profile

**Node Details:**

- **Merge: Assemble Candidate Profile**
  - Type & Role: Merges data from two sources by position.
  - Config: Combine mode with “combineByPosition”.
  - Inputs:  
    - Resume parsing output (Gemini: Parse Resume)  
    - LinkedIn details extraction output (LLM: Extract LinkedIn Details)  
  - Outputs: Combined candidate profile JSON.
  - Edge Cases: Mismatched inputs, empty data from one source.

---

#### 2.4 Job Scraping & Matching

**Overview:**  
Scrapes job boards for openings, then uses Gemini AI to analyze, rank, and score jobs based on the candidate profile.

**Nodes Involved:**  
- Decodo Tool: Scrape Job Boards  
- Gemini Model: Match & Score Jobs  
- Gemini Model: Job Matching

**Node Details:**

- **Decodo Tool: Scrape Job Boards**
  - Type & Role: Decodo community node to scrape multiple job boards like LinkedIn Jobs, JobStreet.
  - Config: No specific parameters; relies on integrated scraping logic.
  - Credentials: Requires Decodo API credentials.
  - Inputs: Triggered by Gemini Model: Match & Score Jobs node.
  - Outputs: Raw job listings data.
  - Edge Cases: API limits, scraping failures, dynamic page structures.
  
- **Gemini Model: Match & Score Jobs**
  - Type & Role: Google Gemini AI model to analyze job listings and score matches.
  - Config: No parameters set; used as AI language model.
  - Credentials: Google Palm API.
  - Inputs: Job listings data from Decodo Tool.
  - Outputs: Processed job match data.
  - Edge Cases: API errors, scoring inconsistencies.
  
- **Gemini Model: Job Matching**
  - Type & Role: Langchain Agent model acting as a Smart Job Match Finder Agent.
  - Config:  
    - System message defines the agent’s role to find best job matches using Decodo Tool for scraping.  
    - Input prompt combines short summary, resume content, and LinkedIn user data.  
    - Output is clean HTML for email.
  - Inputs:  
    - Merged candidate profile.  
    - Job listings data via linked AI tool nodes.  
  - Outputs: HTML-formatted job match report.
  - Edge Cases: No matching jobs found, short resume text, API limits, prompt failures.

---

#### 2.5 Reporting & Notification

**Overview:**  
Sends the final HTML formatted job match report to the candidate’s email via Gmail.

**Nodes Involved:**  
- Gmail: Send Resume Summary and Top Matches Job

**Node Details:**

- **Gmail: Send Resume Summary and Top Matches Job**
  - Type & Role: Gmail node for sending email.
  - Config:  
    - Recipient email from user_email field.  
    - Subject fixed: “Top Job Matches for you”.  
    - Message body uses the HTML output from Gemini Model: Job Matching.  
    - “Send as HTML” enabled for clean formatting.  
    - Attribution disabled.
  - Credentials: Gmail OAuth2 credentials.
  - Inputs: Gemini Model: Job Matching output.
  - Outputs: Email sent confirmation.
  - Edge Cases: Authentication errors, email delivery failures, invalid email addresses.

---

### 3. Summary Table

| Node Name                          | Node Type                               | Functional Role                   | Input Node(s)                        | Output Node(s)                           | Sticky Note                                                                                          |
|-----------------------------------|---------------------------------------|---------------------------------|------------------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook: Receive Intake Form (POST)| n8n-nodes-base.webhook                 | Intake form reception            | -                                  | Set: Capture Form Fields                 | Copy the Webhook URL to your Form. You can test this workflow using form below https://tally.so/r/3jOA51 |
| Set: Capture Form Fields            | n8n-nodes-base.set                     | Extract form fields              | Webhook: Receive Intake Form        | Switch: Has LinkedIn URL?                | Make sure to adjust the form fields capture intake from the webhook                                 |
| Switch: Has LinkedIn URL?           | n8n-nodes-base.switch                  | Conditional routing             | Set: Capture Form Fields             | Decodo: Scrape LinkedIn Profile; Gemini: Parse Resume |                                                                                                    |
| Decodo: Scrape LinkedIn Profile     | @decodo/n8n-nodes-decodo.decodo        | LinkedIn profile scraping       | Switch: Has LinkedIn URL? (true)    | LLM: Extract LinkedIn Details            | Community node; works on self-hosted n8n. Make sure to add your Decodo API credentials.              |
| Gemini: Parse Resume                | @n8n/n8n-nodes-langchain.googleGemini | Resume parsing                  | Switch: Has LinkedIn URL? (false)   | Merge: Assemble Candidate Profile (second input) |                                                                                                    |
| LLM: Extract LinkedIn Details       | @n8n/n8n-nodes-langchain.informationExtractor | Extract structured LinkedIn details | Decodo: Scrape LinkedIn Profile      | Merge: Assemble Candidate Profile (first input)   | Community node; works on self-hosted n8n. Make sure to add your Decodo API credentials.              |
| Gemini Model: Profile Extraction    | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI-based profile extraction     | LLM: Extract LinkedIn Details       | -                                       | Community node; works on self-hosted n8n. Make sure to add your Decodo API credentials.              |
| Merge: Assemble Candidate Profile   | n8n-nodes-base.merge                   | Combine profile data            | Gemini: Parse Resume; LLM: Extract LinkedIn Details | Gemini Model: Job Matching                |                                                                                                    |
| Decodo Tool: Scrape Job Boards      | @decodo/n8n-nodes-decodo.decodoTool    | Scrape job listings             | Gemini Model: Match & Score Jobs (ai_tool) | Gemini Model: Job Matching (ai_tool)      | Community node; works on self-hosted n8n. Make sure to add your Decodo API credentials.              |
| Gemini Model: Match & Score Jobs    | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI scoring of jobs             | Decodo Tool: Scrape Job Boards      | Gemini Model: Job Matching (ai_languageModel) |                                                                                                    |
| Gemini Model: Job Matching          | @n8n/n8n-nodes-langchain.agent         | Job matching & HTML generation | Merge: Assemble Candidate Profile; Gemini Model: Match & Score Jobs | Gmail: Send Resume Summary and Top Matches Job | You can adjust the email message by editing the prompt from the agent, or make your email message yourself but ensure HTML format |
| Gmail: Send Resume Summary and Top Matches Job | n8n-nodes-base.gmail                 | Email final report              | Gemini Model: Job Matching           | -                                       |                                                                                                    |
| Sticky Note (Title & Instructions) | n8n-nodes-base.stickyNote               | Documentation & instructions   | -                                  | -                                       | Contains workflow description, target users, setup instructions, and disclaimers                    |
| Sticky Note1                       | n8n-nodes-base.stickyNote               | Empty / layout                 | -                                  | -                                       |                                                                                                    |
| Sticky Note3                       | n8n-nodes-base.stickyNote               | Reminder about form field capture | -                                  | -                                       | Make sure to adjust the form fields capture intake from the webhook                                 |
| Sticky Note4                       | n8n-nodes-base.stickyNote               | Webhook testing reference      | -                                  | -                                       | Copy the Webhook URL to your Form. You can test this workflow using form below https://tally.so/r/3jOA51 |
| Sticky Note5                       | n8n-nodes-base.stickyNote               | Decodo node credential reminder | -                                  | -                                       | Community node; works on self-hosted n8n. Make sure to add your Decodo API credentials.              |
| Sticky Note6                       | n8n-nodes-base.stickyNote               | Decodo node credential reminder | -                                  | -                                       | Community node; works on self-hosted n8n. Make sure to add your Decodo API credentials.              |
| Sticky Note7                       | n8n-nodes-base.stickyNote               | Email message customization    | -                                  | -                                       | You can adjust the email message by editing the prompt from the agent, or make your email message yourself but ensure HTML format |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Receive Intake Form (POST)**
   - Type: Webhook (HTTP POST)
   - Path: Unique path (e.g., "ca18f47b-399e-4689-9795-80ac1d23829b")
   - HTTP Method: POST
   - Purpose: Receive candidate form submissions.

2. **Create Set Node: Capture Form Fields**
   - Type: Set
   - Assign fields from webhook JSON:  
     - `user_email` = `{{$json.body.data.fields[0].value}}`  
     - `short_summary` = `{{$json.body.data.fields[1].value}}`  
     - `resume_url_link` = `{{$json.body.data.fields[2].value[0].url}}`  
     - `user_linkedin` = `{{$json.body.data.fields[3].value}}`  
   - Connect: Webhook → Set node.

3. **Create Switch Node: Has LinkedIn URL?**
   - Type: Switch
   - Two outputs:  
     - If `user_linkedin` is not empty (non-empty string)  
     - If `user_linkedin` is empty  
   - Connect: Set node → Switch node.

4. **Create Decodo Node: Scrape LinkedIn Profile**
   - Type: Decodo node (community)
   - Parameter: `url` = `{{$json.user_linkedin}}`
   - Credentials: Configure Decodo API credentials.
   - Connect: Switch node (if LinkedIn exists) → Decodo node.

5. **Create LLM Node: Extract LinkedIn Details**
   - Type: Langchain information extractor
   - Input text: `{{$json.data.results[0].content}}` (from Decodo output)
   - Attributes: Name, Headline, Location, About summary, Experience, Education, Licenses & Certifications, Volunteer Experience (all required)
   - Connect: Decodo node → LLM node.

6. **Create Gemini Model: Profile Extraction**
   - Type: Google Gemini chat model
   - Credentials: Google Palm API
   - Connect: LLM node → Gemini Model (optional continuation).

7. **Create Gemini: Parse Resume**
   - Type: Google Gemini document parsing node
   - Parameters:  
     - Text prompt: “What’s in this document or picture?”  
     - Resource: Document  
     - Document URL: `{{$json.resume_url_link}}`
   - Credentials: Google Palm API
   - Connect: Switch node (if no LinkedIn URL) → Gemini Parse Resume node.

8. **Create Merge Node: Assemble Candidate Profile**
   - Type: Merge
   - Mode: Combine, combineByPosition
   - Connect: LLM node output → Merge (first input)  
             Gemini Parse Resume node output → Merge (second input).

9. **Create Gemini Model: Match & Score Jobs**
   - Type: Google Gemini chat model (lmChatGoogleGemini)
   - Credentials: Google Palm API
   - Connect: Decodo Tool node → Gemini Model Match & Score Jobs (ai_languageModel input).

10. **Create Decodo Tool Node: Scrape Job Boards**
    - Type: Decodo Tool (community node)
    - Credentials: Decodo API
    - Connect: Gemini Model: Match & Score Jobs (ai_tool input).

11. **Create Gemini Model: Job Matching**
    - Type: Langchain Agent
    - Prompt:  
      - Includes short summary from captured fields, resume content, LinkedIn user details  
      - System message instructs to scrape and score jobs using Decodo, output clean HTML report
    - Connect: Merge node output → Gemini Model Job Matching (main input)  
              Decodo Tool and Gemini Match & Score Jobs nodes → Gemini Model Job Matching (ai_tool and ai_languageModel inputs).

12. **Create Gmail Node: Send Resume Summary and Top Matches Job**
    - Type: Gmail node
    - Send To: `{{$json.user_email}}`
    - Subject: “Top Job Matches for you”
    - Message: `{{$json.output}}` (HTML content from Gemini Model Job Matching)
    - Options: Send as HTML enabled, disable attribution.
    - Credentials: Gmail OAuth2.
    - Connect: Gemini Model Job Matching → Gmail node.

13. **Workflow Testing and Configuration**
    - Update webhook URL in your candidate forms.
    - Ensure all API credentials (Decodo, Google Palm API, Gmail OAuth2) are configured.
    - Adjust field mappings if intake form structure changes.
    - Enable self-hosted n8n if using community Decodo nodes.
    - Test workflow end-to-end with sample data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Sign up for Decodo [HERE](https://visit.decodo.com/discount) for Discount                                                               | Decodo API signup and discount                                    |
| Copy the Webhook URL to your Form. You can test this workflow using form below https://tally.so/r/3jOA51                                | Webhook testing and form example                                  |
| This workflow relies on a community Decodo node and requires self-hosted n8n. Install the node from the community marketplace first.    | Community node prerequisite                                      |
| Customize the email message by editing the prompt in the Gemini Model node or craft your own HTML message ensuring proper formatting.   | Email message customization advice                                |
| Scoring criteria for job matching: 50% skill alignment, 25% experience match, 15% location/work type, 10% keyword similarity.            | Job match scoring logic                                           |
| Fallback behaviors: informs user politely if no jobs found, requests more experience if resume is too short, never outputs raw HTML.    | AI fallback and error handling guidelines                         |

---

**Disclaimer:** This workflow was created exclusively using n8n automation platform and respects all applicable content policies. All processed data is legal and public. The workflow requires self-hosted n8n due to community nodes used.