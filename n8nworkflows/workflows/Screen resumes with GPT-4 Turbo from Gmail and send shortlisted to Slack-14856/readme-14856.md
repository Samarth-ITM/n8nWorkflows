Screen resumes with GPT-4 Turbo from Gmail and send shortlisted to Slack

https://n8nworkflows.xyz/workflows/screen-resumes-with-gpt-4-turbo-from-gmail-and-send-shortlisted-to-slack-14856


# Screen resumes with GPT-4 Turbo from Gmail and send shortlisted to Slack

# Workflow Documentation: Resume Screener (HR Automation)

### 1. Workflow Overview

This workflow automates the recruitment screening process by integrating Gmail, OpenAI (GPT-4 Turbo), and Slack. Its primary purpose is to retrieve resumes from email drafts, extract text from PDF attachments, use AI to score candidates against specific job requirements, and notify the recruitment team via Slack when a high-quality candidate is found.

The logic is organized into the following functional blocks:
- **1.1 Input Reception:** Manually triggered retrieval of Gmail drafts and attachment formatting.
- **1.2 File Validation & Extraction:** Filtering for PDF files and extracting raw text.
- **1.3 AI Analysis:** Parsing the resume text and scoring the candidate using an LLM.
- **1.4 Decision Logic & Notification:** Filtering candidates based on a score threshold and routing the result to Slack or a rejection state.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
**Overview:** Retrieves email data and prepares binary attachments for individual processing.
- **Nodes Involved:** `Start Workflow Manually`, `Fetch Gmail`, `Format Attachments`, `Check Attachments Exist`.
- **Node Details:**
    - **Start Workflow Manually:** Manual trigger to initiate the process.
    - **Fetch Gmail:** Uses the Gmail API to `getAll` drafts.
    - **Format Attachments (Function):** A JavaScript node that transforms the `binary` object from Gmail into an array of individual items, allowing n8n to process each file separately.
    - **Check Attachments Exist (If):** A boolean check using `Object.keys($binary || {}).length > 0` to ensure the email actually contains files before proceeding.
    - **Potential Failures:** Gmail OAuth2 authentication errors; empty draft folders.

#### 2.2 File Validation & Extraction
**Overview:** Ensures only valid PDF resumes are processed and converts them into machine-readable text.
- **Nodes Involved:** `Process Each Attachment`, `Validate PDF File`, `Extract Resume Text`.
- **Node Details:**
    - **Process Each Attachment (Split In Batches):** Processes files one by one (batch size 1) to prevent memory overload during AI processing.
    - **Validate PDF File (If):** Checks if the `mimeType` is exactly `application/pdf`.
    - **Extract Resume Text (Extract From File):** Uses the PDF operation to convert the binary file into a plain text string.
    - **Potential Failures:** Corrupt PDF files; non-PDF files bypassing the check (though handled by the IF node).

#### 2.3 AI Analysis
**Overview:** Uses a Large Language Model to extract structured data and calculate a match score based on a specific job description.
- **Nodes Involved:** `AI Resume Analyzer`, `OpenAI Chat Model`, `Parse AI Response`.
- **Node Details:**
    - **OpenAI Chat Model:** Technical provider using `gpt-4-turbo`.
    - **AI Resume Analyzer (Agent):** The core logic. It uses a strict prompt requiring JSON output. It evaluates skills (React Native, JS, REST APIs, Git, Expo), experience (3+ years), and location (India).
    - **Parse AI Response (Code):** A JavaScript node that cleans the AI's output (removing `\n` characters) and executes `JSON.parse()` to turn the AI string into actual n8n JSON fields.
    - **Potential Failures:** AI hallucination; JSON formatting errors (handled by the try-catch block in the Parse node); OpenAI API timeouts.

#### 2.4 Decision Logic & Notification
**Overview:** Final routing based on the AI's match score.
- **Nodes Involved:** `Evaluate Candidate Score`, `Send Shortlisted to Slack`, `Mark as Rejected`.
- **Node Details:**
    - **Evaluate Candidate Score (If):** Checks if `match_score` is $\ge 70$.
    - **Send Shortlisted to Slack:** Sends a formatted message to a specific Slack channel (`C0ALTQD6L0H`) containing the name, email, experience, and the AI's reasoning.
    - **Mark as Rejected (Set):** A terminal node that simply labels the item as "Rejected candidate" for tracking.
    - **Potential Failures:** Slack API connection errors; incorrect channel IDs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start Workflow Manually | Manual Trigger | Entry Point | - | Fetch Gmail | - |
| Fetch Gmail | Gmail | Data Retrieval | Start Workflow Manually | Format Attachments | ## Fetch and Prepare Resume Files... |
| Format Attachments | Function | Data Transformation | Fetch Gmail | Check Attachments Exist | ## Fetch and Prepare Resume Files... |
| Check Attachments Exist | If | Validation | Format Attachments | Process Each Attachment | ## Fetch and Prepare Resume Files... |
| Process Each Attachment | Split In Batches | Iteration | Check Attachments Exist | Validate PDF File | ## Process and Validate Resume Files... |
| Validate PDF File | If | Format Filter | Process Each Attachment | Extract Resume Text | ## Process and Validate Resume Files... |
| Extract Resume Text | Extract From File | Text Conversion | Validate PDF File | AI Resume Analyzer | ## Extract Resume Content... |
| AI Resume Analyzer | AI Agent | Intelligence/Scoring | Extract Resume Text | Parse AI Response | ## Analyze Resume with AI... |
| OpenAI Chat Model | LLM Chat Model | AI Provider | - | AI Resume Analyzer | ## Analyze Resume with AI... |
| Parse AI Response | Code | Data Structuring | AI Resume Analyzer | Evaluate Candidate Score | ## Analyze Resume with AI... |
| Evaluate Candidate Score | If | Decision Gate | Parse AI Response | Slack / Mark Rejected | ## Evaluate Candidate Eligibility... |
| Send Shortlisted to Slack | Slack | Notification | Evaluate Candidate Score | - | ## Send Results and Final Status... |
| Mark as Rejected | Set | Status Update | Evaluate Candidate Score | - | ## Send Results and Final Status... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger & Gmail Setup:**
    - Create a **Manual Trigger**.
    - Add a **Gmail Node**: Set operation to `getAll` and filter for drafts. Connect your Gmail OAuth2 credentials.
2.  **Attachment Formatting:**
    - Add a **Function Node**. Use the code provided in the JSON to map binary keys to a list of items.
    - Add an **If Node**: Set a boolean condition to check if the binary object length is $> 0$.
3.  **PDF Processing:**
    - Add a **Split In Batches Node**: Set batch size to `1`.
    - Add an **If Node**: Check if `$binary.data.mimeType` is equal to `application/pdf`.
    - Add an **Extract From File Node**: Set operation to `pdf` to extract text.
4.  **AI Integration:**
    - Add an **AI Agent Node**.
    - Attach an **OpenAI Chat Model Node** to the agent. Configure it to use `gpt-4-turbo` and provide your OpenAI API key.
    - In the Agent's prompt, define the Job Requirements (Skills, Experience, Location) and the strict JSON output format (fields: `full_name`, `email`, `phone`, `skills`, `total_experience`, `current_company`, `match_score`, `decision`, `reason`).
5.  **JSON Parsing:**
    - Add a **Code Node**. Implement a `try-catch` block using `JSON.parse()` to convert the AI string output into a JSON object.
6.  **Filtering & Routing:**
    - Add an **If Node**: Check if `match_score` $\ge 70$.
    - **True Path:** Add a **Slack Node**. Configure the message template using expressions to include candidate details and send it to the target channel.
    - **False Path:** Add a **Set Node** to create a variable `status = "Rejected candidate"`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow Overview & Setup Guide | Detailed instructions on Gmail, OpenAI, and Slack credentials setup. |
| Job Requirements | Currently configured for: React Native, JS, REST APIs, Git, Expo (3+ years exp). |
| Output Requirement | AI must return ONLY JSON; no markdown or conversational text. |