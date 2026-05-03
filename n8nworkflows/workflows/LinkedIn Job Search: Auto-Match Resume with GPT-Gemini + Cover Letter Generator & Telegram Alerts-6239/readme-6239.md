LinkedIn Job Search: Auto-Match Resume with GPT/Gemini + Cover Letter Generator & Telegram Alerts

https://n8nworkflows.xyz/workflows/linkedin-job-search--auto-match-resume-with-gpt-gemini---cover-letter-generator---telegram-alerts-6239


# LinkedIn Job Search: Auto-Match Resume with GPT/Gemini + Cover Letter Generator & Telegram Alerts

### 1. Workflow Overview

This n8n workflow automates the process of searching for suitable jobs on LinkedIn by matching job postings against a user's resume using AI-driven scoring and generating personalized cover letters. It integrates with Google Drive to fetch the user's resume, Google Sheets to manage search filters and store results, LinkedIn job search via HTTP requests, AI services (OpenAI/GPT/Gemini) for semantic matching and cover letter generation, and Telegram for real-time notifications.

The workflow is logically divided into these blocks:

- **1.1 Initialization and Trigger:** Scheduled execution to start the workflow daily.
- **1.2 Resume Preparation:** Downloading and extracting text from the user's resume PDF stored in Google Drive.
- **1.3 Search Filter Acquisition:** Loading job search filter parameters from Google Sheets.
- **1.4 LinkedIn Job Search URL Construction:** Building the LinkedIn job search URL dynamically from filter criteria.
- **1.5 Job Search and Extraction:** Fetching LinkedIn search result pages and extracting individual job links.
- **1.6 Job Detail Processing Loop:** Iterating over each job link, fetching job details, extracting attributes, modifying them, and sending to AI for scoring and cover letter generation.
- **1.7 AI Processing:** Using an AI agent with the user's resume and job description to produce a matching score and a cover letter.
- **1.8 Results Handling:** Parsing AI output, storing results in Google Sheets, filtering jobs based on score, and sending Telegram alerts for high-scoring jobs.
- **1.9 Control Flow and Delay:** Managing batch processing and throttling to avoid overloading LinkedIn or API limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Trigger

**Overview:**  
This block initiates the workflow based on a scheduled trigger, set to execute daily at 5 PM (Europe/Berlin timezone). It ensures the workflow runs automatically without manual intervention.

**Nodes Involved:**  
- `Schedule Trigger`

**Node Details:**

- **Schedule Trigger**  
  - *Type:* `n8n-nodes-base.scheduleTrigger`  
  - *Purpose:* Starts the workflow on a daily schedule.  
  - *Configuration:* Set to trigger every day at 17:00 (5 PM).  
  - *Input:* None (trigger node).  
  - *Output:* Triggers the next node (`Download file`).  
  - *Edge Cases:* Ensure server time and timezone are correctly set to avoid mis-timed executions.  
  - *Sticky Note:* "Workflow executes daily at 5pm, you can change the interval and time of execution."

---

#### 2.2 Resume Preparation

**Overview:**  
Downloads the user's resume PDF from Google Drive and extracts its text content to be used for AI matching.

**Nodes Involved:**  
- `Download file`  
- `Extract from File`

**Node Details:**

- **Download file**  
  - *Type:* `n8n-nodes-base.googleDrive`  
  - *Purpose:* Downloads the resume PDF file from Google Drive.  
  - *Configuration:* Uses a list mode to select the file by ID (configured to a specific file ID).  
  - *Credentials:* Google Drive OAuth2 credentials required.  
  - *Input:* Trigger from `Schedule Trigger`.  
  - *Output:* File data passed to `Extract from File`.  
  - *Edge Cases:* File not found, access denied, or file ID changed.  
  - *Sticky Note:* "Read user resume from google drive, you need to upload your resume in pdf format and select in **From List** section."

- **Extract from File**  
  - *Type:* `n8n-nodes-base.extractFromFile`  
  - *Purpose:* Extracts textual content from the downloaded PDF resume.  
  - *Configuration:* Operation set to PDF text extraction.  
  - *Input:* File from `Download file`.  
  - *Output:* Text content of resume passed to `Get row(s) in sheet` node.  
  - *Edge Cases:* Corrupt PDF, extraction errors.  
  - *Sticky Note:* "Convert pdf file into text so AI can read content."

---

#### 2.3 Search Filter Acquisition

**Overview:**  
Fetches job search filter parameters from a Google Sheet, which includes keywords, location, experience level, remote options, job type, and "easy apply" preference.

**Nodes Involved:**  
- `Get row(s) in sheet`

**Node Details:**

- **Get row(s) in sheet**  
  - *Type:* `n8n-nodes-base.googleSheets`  
  - *Purpose:* Reads job search criteria from a specific sheet and document.  
  - *Configuration:* Reads from sheet "Filter" (sheet ID: 1038962310) within a Google Sheet document.  
  - *Credentials:* Google Sheets OAuth2 credentials required.  
  - *Input:* Output of `Extract from File`.  
  - *Output:* JSON containing search parameters sent to `Create search URL`.  
  - *Edge Cases:* Sheet or document ID changes, access permissions, empty or malformed data.  
  - *Sticky Note:* "Read search filter consisting of keywords, location, experience level, remote, job type and easy apply from google sheet. You can download [this Template](https://docs.google.com/spreadsheets/d/1mtKVxj_z_QCLGXMx0mJVihWSgS41SzHfU1Rv4r_mRY0) and copy in your personal space."

---

#### 2.4 LinkedIn Job Search URL Construction

**Overview:**  
Constructs a LinkedIn job search URL dynamically based on the filter parameters fetched from the sheet, transforming human-readable filter values into LinkedIn's URL query parameter format.

**Nodes Involved:**  
- `Create search URL`

**Node Details:**

- **Create search URL**  
  - *Type:* `n8n-nodes-base.code` (JavaScript)  
  - *Purpose:* Build a LinkedIn job search URL with parameters for keywords, location, experience level, remote work, job type, and easy apply.  
  - *Configuration:*  
    - Converts experience levels to LinkedIn's numeric codes.  
    - Converts remote options to LinkedIn's codes.  
    - Converts job types to abbreviations (e.g., Full-time → F).  
    - Adds easy apply flag if selected.  
  - *Input:* Search filter JSON from `Get row(s) in sheet`.  
  - *Output:* URL string sent to `Fetch Jobs from Linkedin`.  
  - *Edge Cases:* Empty filter values, unknown filter values, URL encoding issues.  
  - *Sticky Note:* "Create Linkedin search url from filter params."

---

#### 2.5 Job Search and Extraction

**Overview:**  
Fetches LinkedIn job search results page and extracts individual job posting links for further processing.

**Nodes Involved:**  
- `Fetch Jobs from