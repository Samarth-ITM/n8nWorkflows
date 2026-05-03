Automate job applications with PredictLeads, ScrapegraphAI, OpenAI and Gmail

https://n8nworkflows.xyz/workflows/automate-job-applications-with-predictleads--scrapegraphai--openai-and-gmail-13812


# Automate job applications with PredictLeads, ScrapegraphAI, OpenAI and Gmail

# Reference Document: Automate Job Applications with PredictLeads, ScrapeGraphAI, OpenAI, and Gmail

This document provides a technical breakdown of an advanced n8n workflow designed to automate the discovery of job opportunities, extract data from job postings, and generate/draft personalized application emails.

---

### 1. Workflow Overview

The workflow acts as an intelligent career assistant. It allows a user to interact via chat to find companies or provide job links. The logic is divided into five functional phases:

*   **1.1 Lead Discovery & Intent Analysis:** Uses a specialized AI agent to research companies via PredictLeads and Context7 or interpret raw user input.
*   **1.2 Link Extraction:** If the input contains job listings, an LLM chain extracts the specific URLs for processing.
*   **1.3 Intelligent Scraping:** For every extracted link, the workflow uses ScrapeGraphAI to pinpoint the job title, description, and contact email.
*   **1.4 Personalized Content Generation:** A Google Gemini-powered agent crafts a professional email tailored to the specific job description and the candidate's profile.
*   **1.5 Email Drafting & Attachment:** The workflow fetches a CV from a remote URL and creates a draft in Gmail with the personalized text and the resume attached.

---

### 2. Block-by-Block Analysis

#### 2.1 PredictLeads Agent (Lead Discovery)
**Overview:** This block handles the initial user interaction. It determines if the user is asking for company research or providing information, using external tools to gather firmographics.
- **Nodes Involved:** `When chat message received`, `PredictLeads Agent`, `OpenAI Chat Model1`, `Context7`, `PredictLeads`, `Simple Memory`.
- **Node Details:**
    - **PredictLeads Agent:** A LangChain agent with a strict system message to prioritize Context7 for research. It must output valid JSON.
    - **Context7 / PredictLeads:** MCP (Model Context Protocol) tools used for deep company data and job opening discovery.
    - **Parser (Code Node):** Cleans the AI output, stripping markdown formatting to ensure valid JSON for downstream logic.
    - **If list?:** A conditional branch that checks a boolean `list` flag. If true, it proceeds to link extraction; if false, it sends a direct chat response.

#### 2.2 Links Extraction
**Overview:** When a list of jobs or a text containing links is identified, this block isolates the URLs.
- **Nodes Involved:** `Links Extractor`, `Structured Output Parser`, `Split Out`, `Limit`.
- **Node Details:**
    - **Links Extractor:** Uses an LLM chain to focus solely on URL identification.
    - **Structured Output Parser:** Forces the LLM to return an array of strings (the links).
    - **Split Out:** Converts the array into individual n8n items for batch processing.

#### 2.3 Scrape Job
**Overview:** Navigates to each job URL to extract structured data required for an application.
- **Nodes Involved:** `Loop Over Items`, `Scrape Job` (ScrapeGraphAI), `Contain email?`.
- **Node Details:**
    - **Scrape Job:** Utilizes ScrapeGraphAI to intelligently extract `email`, `position`, and `text` from the webpage without manual selector mapping.
    - **Contain email?:** An "If" node checking if the extracted data contains an "@" symbol. If no email is found, it routes to a "Chat2" node to inform the user.

#### 2.4 Job Application Agent
**Overview:** Generates the actual email content based on the scraped job details and hardcoded candidate info.
- **Nodes Involved:** `Job Application Agent`, `Google Gemini Chat Model1`, `Create email` (HITL Tool), `Send email` (Workflow Tool).
- **Node Details:**
    - **Job Application Agent:** Instructed to be a "professional job application expert." It uses the candidate's personal data (Name, Resident, Skills) to personalize the draft.
    - **Create email:** A Human-in-the-loop (HITL) tool that requests "double approval" before proceeding, allowing the user to review the AI-generated email.
    - **Send email:** A Tool Workflow node that triggers the secondary part of the workflow (or a sub-workflow) with the final text.

#### 2.5 Gmail Drafting
**Overview:** The final technical execution—gathering the resume and preparing the email draft.
- **Nodes Involved:** `When Executed by Another Workflow`, `Get CV`, `Create a draft`.
- **Node Details:**
    - **Get CV:** An HTTP Request node that downloads the candidate's resume (PDF) from a provided URL.
    - **Create a draft:** Authenticates with Gmail OAuth2 to create a draft in the user's "Drafts" folder, attaching the CV and inserting the generated subject/body.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When chat message received | Chat Trigger | Entry Point | - | PredictLeads Agent | |
| PredictLeads Agent | AI Agent | Research & Intent | When chat message received | Parser | STEP 1 - PredictLeads Agent processes the request |
| Context7 | MCP Tool | Company Research | - | PredictLeads Agent | STEP 1 - If the request involves company research... |
| PredictLeads | MCP Tool | Job Discovery | - | PredictLeads Agent | STEP 1 - ...then optionally PredictLeads for deeper data. |
| Parser | Code | Data Formatting | PredictLeads Agent | If list? | |
| If list? | If | Routing | Parser | Links Extractor, Chat | |
| Links Extractor | LLM Chain | URL Extraction | If list? | Split Out | STEP 2 - Extract Links |
| Split Out | Split Out | Data Normalization | Links Extractor | Limit | |
| Loop Over Items | Loop | Batch Processing | Split Out, Chat1, Chat2 | Scrape Job | |
| Scrape Job | ScrapeGraphAI | Data Extraction | Loop Over Items | Contain email? | STEP 3 - Scrape Job extracts: Email, Position, Description |
| Contain email? | If | Logic Filter | Scrape Job | Job application Agent, Chat2 | |
| Job application Agent | AI Agent | Email Copywriting | Contain email? | Chat1 | STEP 4 - Job Application Agent generates a professional email |
| Create email | HITL Tool | Human Approval | Job application Agent | Send email | STEP 4 - A tool (Create email) to format the subject and body |
| Send email | Tool Workflow | Sub-workflow Trigger | Create email | - | STEP 5 - The agent triggers the Send email workflow |
| Get CV | HTTP Request | File Retrieval | Executed by Another Workflow | Create a draft | STEP 5 - Fetches the CV from a public URL |
| Create a draft | Gmail | Email Execution | Get CV | - | STEP 5 - Creates a draft in Gmail with the CV attached |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Create a new workflow in n8n. Ensure you have credentials for OpenAI (GPT-4), Google Gemini, ScrapeGraphAI, Gmail (OAuth2), Context7, and PredictLeads.
2.  **Initial Agent (Phase 1):** 
    *   Add a **Chat Trigger** node.
    *   Connect an **AI Agent** node (named "PredictLeads Agent") using **OpenAI Chat Model**.
    *   Connect **Context7** and **PredictLeads** MCP tools to the agent.
    *   Set the System Prompt to enforce JSON output: `{"list": boolean, "output": string}`.
3.  **Data Cleaning:** 
    *   Add a **Code Node** ("Parser") to clean the output of the agent using `JSON.parse` and regex to remove markdown backticks.
4.  **Routing:**
    *   Add an **If Node** ("If list?") to check the `list` property from the Parser.
5.  **Link Extraction (Phase 2):**
    *   Add a **Chain LLM** node ("Links Extractor") and connect a **Structured Output Parser** configured to an array of links.
    *   Follow this with a **Split Out** node to handle multiple links.
6.  **Scraping (Phase 3):**
    *   Add a **Split in Batches** (Loop Over Items) node.
    *   Inside the loop, add the **ScrapeGraphAI** node. Configure the prompt to extract "email", "position", and "text".
    *   Use an **If Node** to verify if an email address was actually found.
7.  **Email Generation (Phase 4):**
    *   Add an **AI Agent** node ("Job Application Agent") using **Google Gemini**.
    *   Add a **Chat HITL Tool** ("Create email") to allow for manual approval of the text.
    *   Connect a **Workflow Tool** ("Send email") to the agent. *Note: Set the Workflow ID to the ID of this current workflow if using self-triggering.*
8.  **Execution (Phase 5):**
    *   Add a **Execute Workflow Trigger**.
    *   Add an **HTTP Request** node to download your CV from a static URL (e.g., `https://your-site.com/cv.pdf`).
    *   Add a **Gmail Node** set to "Create a draft". Map the `text`, `subject`, and `emailTo` from the trigger and the binary file from the HTTP Request.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **YouTube Channel** | Practical tutorials and free templates: [Subscribe to n3witalia](https://youtube.com/@n3witalia) |
| **ScrapeGraphAI** | Advanced scraping capabilities: [Dashboard](https://dashboard.scrapegraphai.com/?via=n3witalia) |
| **PredictLeads** | Business data source: [PredictLeads Website](https://predictleads.com/) |
| **Setup Requirement** | Ensure the "Send email" node uses the current Workflow ID or points to a dedicated child workflow. |