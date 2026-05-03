Automated Resume Screening & Ranking with Llama 4 AI and Google Workspace

https://n8nworkflows.xyz/workflows/automated-resume-screening---ranking-with-llama-4-ai-and-google-workspace-3838


# Automated Resume Screening & Ranking with Llama 4 AI and Google Workspace

### 1. Workflow Overview

This workflow automates the initial screening, sorting, ranking, and tracking of candidate resumes (CVs) using AI (Groq Llama 4) integrated with Google Workspace services. It is designed for talent acquisition teams, recruitment agencies, HR professionals, and hiring managers who want to streamline bulk resume screening by eliminating manual evaluation. The workflow monitors a Google Drive folder for new resumes, extracts their content, compares them against a maintained job description, and uses AI to generate a suitability decision, score, and rationale. It then organizes resumes into status-based folders, updates a candidate tracking Google Sheet, and sends email notifications.

**Logical Blocks:**

- **1.1 Input Reception and File Handling**  
  Detects new resume files in a specific Google Drive folder and downloads them.

- **1.2 Data Extraction**  
  Extracts text content from the candidate resume PDF and the job description Google Doc.

- **1.3 AI Processing and Decision Making**  
  Uses Groq Llama 4 AI model via a LangChain AI Agent to analyze the candidate profile against the job description, producing a decision (Shortlisted/KIV/Rejected), a score, and a rationale.

- **1.4 Post-Processing Actions**  
  Moves the candidate resume file to the appropriate Google Drive folder based on AI decision, updates the candidate tracker Google Sheet, and sends notification emails.

- **1.5 Documentation and Setup Notes**  
  Sticky notes provide setup instructions, folder structure examples, and workflow explanations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and File Handling

**Overview:**  
This block listens for new resume files added to a designated Google Drive folder ("Unfiltered") and downloads the detected PDF files for processing.

**Nodes Involved:**  
- Google Drive - Resume CV File Created  
- Download Resume File From Gdrive

**Node Details:**

- **Google Drive - Resume CV File Created**  
  - Type: Google Drive Trigger  
  - Role: Watches a specific Google Drive folder for newly created files (resumes).  
  - Configuration: Trigger on file creation event, polling every hour, restricted to folder ID "17g2HGxLieONy6EWfsPADvA9IXDp5nJ8p" (Unfiltered folder).  
  - Inputs: None (trigger node)  
  - Outputs: Emits metadata of new file including file ID and name.  
  - Edge Cases: Possible failure if folder ID is incorrect or credentials expire; delay due to polling interval.  
  - Credentials: Google Drive OAuth2.

- **Download Resume File From Gdrive**  
  - Type: Google Drive (download operation)  
  - Role: Downloads the actual resume PDF file using the file ID from the trigger node.  
  - Configuration: Uses file ID and file name from the trigger node output.  
  - Inputs: File metadata from trigger node.  
  - Outputs: Binary data of the downloaded PDF file.  
  - Edge Cases: Download failure if file is deleted or access revoked; large file size may cause timeout.  
  - Credentials: Google Drive OAuth2.

---

#### 1.2 Data Extraction

**Overview:**  
Extracts textual content from the downloaded resume PDF and separately retrieves the job description text from a Google Doc.

**Nodes Involved:**  
- Extract from File  
- GDocs - Get Job Desc

**Node Details:**

- **Extract from File**  
  - Type: Extract From File (PDF)  
  - Role: Extracts text content from the downloaded PDF resume for AI analysis.  
  - Configuration: Operation set to PDF extraction, no additional options.  
  - Inputs: Binary PDF file from "Download Resume File From Gdrive".  
  - Outputs: JSON with extracted text under `text` property.  
  - Edge Cases: Extraction may fail if PDF is corrupted or scanned image-only; text quality depends on PDF format.  
  - Version: v1.

- **GDocs - Get Job Desc**  
  - Type: Google Docs  
  - Role: Retrieves the job description content from a specified Google Doc URL.  
  - Configuration: Operation "get" with a fixed document URL for the maintained job description.  
  - Inputs: None (manual retrieval).  
  - Outputs: JSON with document content under `content` property.  
  - Edge Cases: Access denied if credentials lack permission; document URL must be valid and accessible.  
  - Credentials: Google Docs OAuth2.

---

#### 1.3 AI Processing and Decision Making

**Overview:**  
The AI Agent compares the extracted candidate resume text with the job description, then produces a decision (Shortlisted, Rejected, KIV), a rationale, and a score. It orchestrates subsequent actions by invoking tools to move files, update trackers, and send notifications.

**Nodes Involved:**  
- Groq - llama 4 AI MODEL  
- AI Agent

**Node Details:**

- **Groq - llama 4 AI MODEL**  
  - Type: LangChain LM Chat (Groq)  
  - Role: Provides the underlying AI language model (Llama 4 Maverick 17b) used by the AI Agent for natural language understanding and generation.  
  - Configuration: Model set to "meta-llama/llama-4-maverick-17b-128e-instruct" with default options.  
  - Inputs: Prompt and context from AI Agent.  
  - Outputs: AI-generated text responses.  
  - Credentials: Groq API key.  
  - Edge Cases: API rate limits, model availability, or network errors.

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates the AI-driven evaluation workflow. Receives job description and candidate resume text, prompts the AI model to generate decision, reason, and score, then calls downstream tools to move files, update Google Sheets, and send emails.  
  - Configuration:  
    - System message: Expert backend principal engineer specializing in Python, tasked with comparing job description and candidate profile.  
    - Prompt includes job description and candidate resume text.  
    - Expected AI output: JSON with fields `decision`, `reason`, and `score`.  
    - Sequential tool invocation:  
      1. Move candidate resume file to appropriate folder (Reject/KIV/Shortlisted).  
      2. Update candidate tracker Google Sheet.  
      3. Send email notification with candidate name, role, decision, and reason.  
  - Inputs: Job description content (from GDocs node), candidate resume text (from Extract from File node).  
  - Outputs: AI decision and triggers downstream tools via ai_tool connections.  
  - Edge Cases: AI output parsing errors, unexpected AI responses, API failures, or tool invocation errors.  
  - Version: 1.9.

---

#### 1.4 Post-Processing Actions

**Overview:**  
Based on AI decision, this block moves the candidate resume file to the corresponding Google Drive folder, updates the candidate tracker spreadsheet with AI results, and sends an email notification.

**Nodes Involved:**  
- Gdrive:Move-To-Reject-Folder  
- Gdrive:Move-To-KIV-Folder  
- Gdrive:Move-To-Shortlisted-Folder  
- Gsheet: Update Candidate Tracker  
- Gmail:Notification

**Node Details:**

- **Gdrive:Move-To-Reject-Folder**  
  - Type: Google Drive Tool (Move file)  
  - Role: Moves rejected candidate resumes to the "REJECTED" folder.  
  - Configuration: Uses file ID from the original Google Drive trigger node; target folder ID "16BR7dhzd4-6i_kHYRStJd5UdqNWhpXKA".  
  - Inputs: Triggered by AI Agent tool call.  
  - Outputs: Confirmation of move operation.  
  - Credentials: Google Drive OAuth2.

- **Gdrive:Move-To-KIV-Folder**  
  - Type: Google Drive Tool (Move file)  
  - Role: Moves "Keep In View" (KIV) candidate resumes to the "KIV" folder.  
  - Configuration: Uses file ID from trigger; target folder ID "1KLfykacUhwtO0-wgYs6WsrcxbCHHKJ7o".  
  - Inputs: AI Agent tool call.  
  - Outputs: Confirmation of move operation.  
  - Credentials: Google Drive OAuth2.

- **Gdrive:Move-To-Shortlisted-Folder**  
  - Type: Google Drive Tool (Move file)  
  - Role: Moves shortlisted candidate resumes to the "SHORTLISTED" folder.  
  - Configuration: Uses file ID from trigger; target folder ID "1m8vrejmyPWpGsjJc6amnWfSXBRESlpfO".  
  - Inputs: AI Agent tool call.  
  - Outputs: Confirmation of move operation.  
  - Credentials: Google Drive OAuth2.

- **Gsheet: Update Candidate Tracker**  
  - Type: Google Sheets Tool (Append or Update)  
  - Role: Updates or appends candidate screening results (AI Score, Verdict, Reason) in a Google Sheet tracker.  
  - Configuration:  
    - Sheet ID: "1SwnbH_dnqPMho7SqX1LKAjFMc0YvLBGok4I1AdgrJjE"  
    - Sheet tab ID: 843593464  
    - Matching column: "Candidate Name" to update existing rows or append new.  
    - Columns updated: AI Score, AI Verdict, AI Reason, Candidate Name (from AI output).  
  - Inputs: AI Agent tool call with AI results.  
  - Outputs: Confirmation of update.  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Mismatched candidate names causing duplicate entries; permission errors.

- **Gmail:Notification**  
  - Type: Gmail Tool (Send email)  
  - Role: Sends email notifications to a fixed recipient with candidate name, job role, decision, and reason.  
  - Configuration:  
    - Recipient: aiix.space.noreply@gmail.com  
    - Subject and message dynamically generated from AI output.  
  - Inputs: AI Agent tool call.  
  - Outputs: Email sent confirmation.  
  - Credentials: Gmail OAuth2.  
  - Edge Cases: Email sending failures, invalid recipient, quota limits.

---

#### 1.5 Documentation and Setup Notes

**Overview:**  
Sticky Note nodes provide detailed instructions, folder and file setup examples, and workflow purpose explanations to assist users in configuring and understanding the workflow.

**Nodes Involved:**  
- Sticky Note (multiple: Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note8)

**Node Details:**

- Sticky notes contain:  
  - Workflow purpose and benefits overview.  
  - Folder and file structure examples with links to Google Drive folders and Google Docs/Sheets templates.  
  - Stepwise explanations of workflow blocks (e.g., downloading resumes, reading job description, moving files, updating tracker, sending notifications).  
  - Visual screenshots linked from GitHub.  
  - Notes on credential setup and prerequisites.  
  - Labels for folders like "UNFILTERED FOLDER".  
- These nodes have no inputs or outputs; purely informational.

---

### 3. Summary Table

| Node Name                      | Node Type                  | Functional Role                                  | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                      |
|--------------------------------|----------------------------|-------------------------------------------------|----------------------------------|--------------------------------------|-------------------------------------------------------------------------------------------------|
| Google Drive - Resume CV File Created | Google Drive Trigger        | Detect new resume files in "Unfiltered" folder   | None                             | Download Resume File From Gdrive      | ## Add candidate CV Resume into folder                                                          |
| Download Resume File From Gdrive | Google Drive               | Download resume PDF file                          | Google Drive - Resume CV File Created | Extract from File                     | ## Download and read candidate CV Resume                                                       |
| Extract from File               | Extract From File (PDF)    | Extract text from downloaded resume PDF          | Download Resume File From Gdrive | GDocs - Get Job Desc                  | ## Read Job Description                                                                         |
| GDocs - Get Job Desc            | Google Docs                | Retrieve job description text                     | Extract from File                | AI Agent                            | ## Read Job Description                                                                         |
| Groq - llama 4 AI MODEL         | LangChain LM Chat (Groq)  | Provide AI language model for analysis            | AI Agent                       | AI Agent                            |                                                                                                 |
| AI Agent                      | LangChain Agent            | Compare resume and job description, decide outcome, orchestrate file moves, tracker update, email notification | GDocs - Get Job Desc, Extract from File | Gdrive:Move-To-Reject-Folder, Gdrive:Move-To-KIV-Folder, Gdrive:Move-To-Shortlisted-Folder, Gsheet: Update Candidate Tracker, Gmail:Notification | ## 1. Move candidate cv to folder<br>## 2. Update tracker<br>## 3. Send email notification       |
| Gdrive:Move-To-Reject-Folder    | Google Drive Tool          | Move rejected resumes to "REJECTED" folder       | AI Agent (tool call)            | None                               | ## 1. Move candidate cv to folder                                                              |
| Gdrive:Move-To-KIV-Folder       | Google Drive Tool          | Move KIV resumes to "KIV" folder                   | AI Agent (tool call)            | None                               | ## 1. Move candidate cv to folder                                                              |
| Gdrive:Move-To-Shortlisted-Folder | Google Drive Tool          | Move shortlisted resumes to "SHORTLISTED" folder | AI Agent (tool call)            | None                               | ## 1. Move candidate cv to folder                                                              |
| Gsheet: Update Candidate Tracker | Google Sheets Tool         | Update candidate screening results in tracker     | AI Agent (tool call)            | None                               | ## 2. Update tracker                                                                           |
| Gmail:Notification              | Gmail Tool                 | Send email notification with screening results    | AI Agent (tool call)            | None                               | ## 3. Send email notification                                                                  |
| Sticky Note                    | Sticky Note                | Workflow purpose and instructions                  | None                          | None                               | # AI-Agent : Automated CV Resume Screening Evaluate Rating System ... (full content in node)    |
| Sticky Note1                   | Sticky Note                | Tracker update explanation                          | None                          | None                               | ## 2. Update tracker                                                                           |
| Sticky Note2                   | Sticky Note                | Email notification explanation                      | None                          | None                               | ## 3. Send email notification                                                                  |
| Sticky Note3                   | Sticky Note                | Download and read candidate CV explanation         | None                          | None                               | ## Download and read candidate CV Resume                                                       |
| Sticky Note4                   | Sticky Note                | Job description reading explanation                 | None                          | None                               | ## Read Job Description                                                                         |
| Sticky Note5                   | Sticky Note                | Workflow overview and setup instructions            | None                          | None                               | # AI-Agent : Automated CV Resume Screening Evaluate Rating System ... (full content in node)    |
| Sticky Note6                   | Sticky Note                | Unfiltered folder label                              | None                          | None                               | UNFILTERED FOLDER                                                                             |
| Sticky Note7                   | Sticky Note                | Folder and file setup instructions with examples   | None                          | None                               | Folder & File Setup with links to examples and screenshots                                     |
| Sticky Note8                   | Sticky Note                | Label for unfiltered folder                          | None                          | None                               | UNFILTERED FOLDER                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Folder Structure:**  
   - Create folders for "Unfiltered", "REJECTED", "KIV", and "SHORTLISTED" resumes.  
   - Note folder IDs for each.

2. **Create Job Description Google Doc:**  
   - Prepare a Google Doc with the job description text.  
   - Share it with the Google Docs OAuth2 account used in n8n.

3. **Create Candidate Tracker Google Sheet:**  
   - Create a Google Sheet with columns: Candidate Name, AI Score, AI Verdict, AI Reason.  
   - Share it with the Google Sheets OAuth2 account.

4. **Set Up Credentials in n8n:**  
   - Google Drive OAuth2 (with access to all relevant folders).  
   - Google Docs OAuth2 (access to job description doc).  
   - Google Sheets OAuth2 (access to tracker sheet).  
   - Gmail OAuth2 (for sending notifications).  
   - Groq API credentials (for Llama 4 model).

5. **Create Nodes in n8n:**

   - **Google Drive Trigger**  
     - Type: Google Drive Trigger  
     - Trigger on file creation in "Unfiltered" folder (use folder ID).  
     - Poll every hour or as preferred.

   - **Google Drive Download Node**  
     - Type: Google Drive  
     - Operation: Download  
     - File ID: Use expression to get file ID from trigger node.

   - **Extract From File Node**  
     - Type: Extract From File  
     - Operation: PDF  
     - Input: Binary data from download node.

   - **Google Docs Node**  
     - Type: Google Docs  
     - Operation: Get  
     - Document URL: Link to job description doc.

   - **Groq Llama 4 AI Model Node**  
     - Type: LangChain LM Chat (Groq)  
     - Model: meta-llama/llama-4-maverick-17b-128e-instruct  
     - Credentials: Groq API key.

   - **AI Agent Node**  
     - Type: LangChain Agent  
     - System Message: Expert backend engineer comparing job description and candidate profile.  
     - Prompt: Include job description content and candidate resume text extracted.  
     - Define expected output JSON with fields: decision, reason, score.  
     - Configure to sequentially call tools: Move file, update sheet, send email.

   - **Google Drive Move File Nodes (3 nodes)**  
     - For REJECTED, KIV, SHORTLISTED folders respectively.  
     - Operation: Move file to respective folder ID.  
     - File ID: Use original file ID from trigger node.

   - **Google Sheets Update Node**  
     - Operation: Append or Update  
     - Document ID: Candidate tracker sheet ID  
     - Sheet Name: Use correct sheet tab ID  
     - Matching Column: Candidate Name  
     - Columns to update: AI Score, AI Verdict, AI Reason, Candidate Name (from AI output).

   - **Gmail Send Email Node**  
     - Recipient: aiix.space.noreply@gmail.com (or your chosen email)  
     - Subject and Body: Use AI output variables for candidate name, decision, reason.

6. **Connect Nodes:**  
   - Trigger → Download → Extract → Google Docs → AI Agent  
   - AI Agent → Groq Llama 4 AI Model (ai_languageModel input)  
   - AI Agent → Google Drive Move File nodes (ai_tool inputs)  
   - AI Agent → Google Sheets Update node (ai_tool input)  
   - AI Agent → Gmail Notification node (ai_tool input)

7. **Test Workflow:**  
   - Upload a sample resume PDF to the "Unfiltered" folder.  
   - Verify extraction, AI decision, file movement, tracker update, and email notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Folder & File Setup examples with screenshots and Google Drive links for directory structure, job description document, and candidate tracker spreadsheet.                                                                           | [Folder Example](https://drive.google.com/drive/folders/1Uh7VdJORE03YBJkCmvr1TXg_esbiNnTV?dmr=1&ec=wgc-drive-hero-goto) <br> [Job Description Doc](https://docs.google.com/document/d/12dv1AXaotpJ3ST1nUI-QgCoi5SJjM52zeHmjhwZUtvs/edit?usp=drive_link) <br> [Candidate Tracker Sheet](https://docs.google.com/spreadsheets/d/1SwnbH_dnqPMho7SqX1LKAjFMc0YvLBGok4I1AdgrJjE/edit?gid=843593464#gid=843593464) |
| Official n8n documentation recommended for setting up Google Drive, Google Docs, Google Sheets, Gmail, and Groq API credentials.                                                                                                       | n8n official docs (https://docs.n8n.io/integrations/builtin/)                                                    |
| Workflow purpose: Automate resume screening to save time, increase accuracy, and maintain a scalable recruitment process.                                                                                                             | See Sticky Note5 content in workflow.                                                                           |
| Email notification recipient is fixed; customize as needed.                                                                                                                                                                          | Gmail node configuration.                                                                                        |
| AI Agent prompt and system message designed for backend engineer role to ensure consistent decision-making logic.                                                                                                                   | AI Agent node parameters.                                                                                        |

---

This document provides a detailed and structured reference for understanding, reproducing, and maintaining the "Automated Resume Screening & Ranking with Llama 4 AI and Google Workspace" workflow in n8n. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with the workflow.