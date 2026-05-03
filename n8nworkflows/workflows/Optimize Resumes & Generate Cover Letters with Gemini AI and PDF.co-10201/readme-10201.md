Optimize Resumes & Generate Cover Letters with Gemini AI and PDF.co

https://n8nworkflows.xyz/workflows/optimize-resumes---generate-cover-letters-with-gemini-ai-and-pdf-co-10201


# Optimize Resumes & Generate Cover Letters with Gemini AI and PDF.co

---

### 1. Workflow Overview

This workflow automates the optimization of resumes and the generation of tailored cover letters using Google Gemini AI models and PDF.co services. It targets users submitting their resume and job application data via an input form and produces ATS-optimized resumes and personalized cover letters delivered by email.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception & File Retrieval:** Handles form submission triggers, fetches resume files from Google Drive, and downloads them.
- **1.2 Resume Data Extraction & Cleaning:** Extracts text data from PDF resume files and cleans/parses it for AI processing.
- **1.3 Resume Optimization AI Processing:** Sends cleaned resume data to an AI agent powered by Google Gemini to optimize it for ATS (Applicant Tracking Systems).
- **1.4 Cover Letter Generation AI Processing:** Based on a condition, generates a personalized cover letter using a separate AI agent with Google Gemini.
- **1.5 Document Conversion & Merging:** Converts AI output to PDF format, merges resume and cover letter documents.
- **1.6 Email Delivery:** Sends the final optimized resume and cover letter PDF via Gmail.
- **1.7 Control Flow & Error Handling:** Implements conditional branching and switches to manage workflow paths based on form inputs or file presence.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & File Retrieval

- **Overview:**  
  This block waits for a form submission to start the workflow, fetches the resume files from Google Drive, and downloads them for further processing.

- **Nodes Involved:**  
  - On form submission  
  - Get PDF Files/File (Google Drive)  
  - Download Retrieval Files/File (Google Drive)

- **Node Details:**  
  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point, triggers workflow on user form submission  
    - *Config:* Default webhook, no additional parameters  
    - *Inputs:* HTTP form submission  
    - *Outputs:* Passes form data to Google Drive file retrieval  
    - *Edge Cases:* Missing or corrupted form data, webhook failures

  - **Get PDF Files/File**  
    - *Type:* Google Drive node  
    - *Role:* Retrieves file metadata or file list from Google Drive based on prior input  
    - *Config:* Expected to receive file identifiers or queries from form data  
    - *Inputs:* Trigger from form submission  
    - *Outputs:* File metadata for download  
    - *Edge Cases:* Authentication errors, file not found, permission denied

  - **Download Retrieval Files/File**  
    - *Type:* Google Drive node  
    - *Role:* Downloads actual file content for processing  
    - *Config:* Uses file metadata from prior node  
    - *Inputs:* File metadata  
    - *Outputs:* Binary file data for extraction  
    - *Edge Cases:* Download failures, network issues, file format errors

---

#### 1.2 Resume Data Extraction & Cleaning

- **Overview:**  
  Extracts textual data from the downloaded resume PDF and cleans it to prepare for AI optimization.

- **Nodes Involved:**  
  - Extract Files/File's Data (Extract from File)  
  - Get PDF Data Only (Set)  
  - Data Parser & Cleaner (Code)

- **Node Details:**  
  - **Extract Files/File's Data**  
    - *Type:* Extract from File  
    - *Role:* Extracts text content from the binary PDF file  
    - *Config:* Default extraction settings (likely text extraction from PDF)  
    - *Inputs:* Binary file data from Google Drive download  
    - *Outputs:* Extracted text content  
    - *Edge Cases:* Extraction failures with malformed PDFs, unsupported file types

  - **Get PDF Data Only**  
    - *Type:* Set node  
    - *Role:* Filters or sets specific relevant data from extracted content, possibly isolating text only  
    - *Config:* Empty parameters indicate default or placeholder settings, possibly used for structuring data  
    - *Inputs:* Extracted text  
    - *Outputs:* Cleaned or structured text data  
    - *Edge Cases:* Empty or incomplete extraction output

  - **Data Parser & Cleaner**  
    - *Type:* Code (JavaScript)  
    - *Role:* Executes custom parsing and cleaning logic to normalize resume text for the AI agent  
    - *Config:* Custom code to handle text anomalies, remove noise, format data for AI input  
    - *Inputs:* PDF text data  
    - *Outputs:* Cleaned resume data string or object  
    - *Edge Cases:* Code errors, unexpected input format, missing fields

---

#### 1.3 Resume Optimization AI Processing

- **Overview:**  
  Processes cleaned resume data using an AI agent powered by Google Gemini to optimize the resume for ATS.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - Google Gemini Chat Model

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - *Type:* LangChain LM Chat Google Gemini  
    - *Role:* Provides the language model backend for AI processing  
    - *Config:* Uses Google Gemini chat model with default or preset parameters  
    - *Inputs:* Passed from AI Agent node as language model provider  
    - *Outputs:* AI response to agent  
    - *Edge Cases:* Model API limits, invalid input, latency

  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Orchestrates prompt and AI interactions to optimize resume content  
    - *Config:* Uses Google Gemini Chat Model as LM, no explicit parameters visible (likely complex prompt)  
    - *Inputs:* Cleaned resume data from parser  
    - *Outputs:* Optimized resume text or structured output  
    - *Edge Cases:* Agent logic errors, prompt failures, AI rate limiting

---

#### 1.4 Cover Letter Generation AI Processing

- **Overview:**  
  Conditionally generates a customized cover letter using a separate AI agent if the workflow path requires it.

- **Nodes Involved:**  
  - Switch (conditional branching)  
  - If  
  - Cover Letter Agent (LangChain Agent)  
  - Google Gemini Chat Model1

- **Node Details:**  
  - **Switch**  
    - *Type:* Switch node  
    - *Role:* Routes workflow based on form or metadata conditions (e.g., whether cover letter generation is requested)  
    - *Config:* Conditional expressions (not explicitly detailed)  
    - *Inputs:* Form submission data or prior node output  
    - *Outputs:* Two paths — one leads to cover letter generation (If node), the other bypasses it  
    - *Edge Cases:* Invalid or missing condition data

  - **If**  
    - *Type:* If node  
    - *Role:* Further filters or confirms condition to run cover letter generation  
    - *Config:* Boolean expression or condition  
    - *Inputs:* From Switch node  
    - *Outputs:*  
      - True: Passes to Cover Letter Agent  
      - False: Passes to Merge node to continue workflow

  - **Google Gemini Chat Model1**  
    - *Type:* LangChain LM Chat Google Gemini  
    - *Role:* Language model for cover letter generation agent  
    - *Config:* Similar to Gemini Chat Model, but separate instance for cover letter context  
    - *Inputs:* Connected to Cover Letter Agent as LM provider  
    - *Outputs:* AI-generated cover letter text

  - **Cover Letter Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Uses AI to generate a personalized cover letter based on provided data and prompts  
    - *Config:* Uses Google Gemini Chat Model1 as language model  
    - *Inputs:* Data from prior nodes and conditions  
    - *Outputs:* Generated cover letter content  
    - *Edge Cases:* Same AI-related errors as AI Agent; prompt failures

---

#### 1.5 Document Conversion & Merging

- **Overview:**  
  Converts AI-generated content to PDF format and merges resume and cover letter PDFs into a single document.

- **Nodes Involved:**  
  - HTML to PDF (HTTP Request)  
  - Merge

- **Node Details:**  
  - **HTML to PDF**  
    - *Type:* HTTP Request (likely calling PDF.co or similar API)  
    - *Role:* Converts HTML or text-based AI output into PDF format  
    - *Config:* HTTP method, API endpoint, authentication setup (not detailed)  
    - *Inputs:* AI Agent optimized resume content  
    - *Outputs:* PDF binary data  
    - *Edge Cases:* API failures, invalid HTML input, network issues

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines multiple inputs (optimized resume PDF and optionally cover letter PDF) into one data stream for sending  
    - *Config:* Default merge mode (likely 'combine' or 'append')  
    - *Inputs:*  
      - From HTML to PDF (optimized resume)  
      - From Cover Letter Agent (cover letter PDF or content)  
    - *Outputs:* Merged document for email sending  
    - *Edge Cases:* Mismatched data formats, empty inputs

---

#### 1.6 Email Delivery

- **Overview:**  
  Sends the final merged PDF document to the user via Gmail.

- **Nodes Involved:**  
  - Send a message (Gmail)

- **Node Details:**  
  - **Send a message**  
    - *Type:* Gmail node  
    - *Role:* Sends email with attachments containing the optimized resume and cover letter  
    - *Config:* OAuth2 credentials for Gmail, email recipient sourced from form submission, subject/body templates (not detailed)  
    - *Inputs:* Merged PDF document  
    - *Outputs:* None (end of workflow)  
    - *Edge Cases:* Authentication errors, invalid email addresses, attachment size limits

---

#### 1.7 Control Flow & Error Handling

- **Overview:**  
  Implements branching logic to handle optional cover letter generation and manages workflow paths.

- **Nodes Involved:**  
  - Switch  
  - If  
  - Merge (used twice)

- **Node Details:**  
  - **Switch**  
    - *See 1.4 details.*

  - **If**  
    - *See 1.4 details.*

  - **Merge** (two occurrences)  
    - One merges cover letter output into the main flow  
    - The other merges resume PDF conversion and cover letter outputs before sending email  
    - Ensures flexible combination of documents regardless of presence of cover letter

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                                 | Input Node(s)                     | Output Node(s)                    | Sticky Note                                  |
|-------------------------|--------------------------------|------------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------|
| On form submission      | Form Trigger                   | Entry point: triggers on form submit            | —                                | Get PDF Files/File, Switch        |                                              |
| Get PDF Files/File      | Google Drive                   | Retrieves resume file metadata                   | On form submission               | Download Retrieval Files/File     |                                              |
| Download Retrieval Files/File | Google Drive             | Downloads resume file                             | Get PDF Files/File               | Extract Files/File's Data         |                                              |
| Extract Files/File's Data | Extract from File             | Extracts text from PDF                            | Download Retrieval Files/File    | Get PDF Data Only                 |                                              |
| Get PDF Data Only       | Set                           | Prepares/filters extracted data                  | Extract Files/File's Data        | Data Parser & Cleaner             |                                              |
| Data Parser & Cleaner   | Code (JavaScript)              | Cleans and parses extracted resume text          | Get PDF Data Only                | AI Agent                        |                                              |
| AI Agent                | LangChain Agent               | Optimizes resume text with Google Gemini AI      | Data Parser & Cleaner            | HTML to PDF                      |                                              |
| Google Gemini Chat Model| LM Chat Google Gemini          | Language model for AI Agent                       | AI Agent (ai_languageModel)      | AI Agent                        |                                              |
| Switch                  | Switch                        | Routes flow based on cover letter generation flag| On form submission               | If                             |                                              |
| If                      | If                           | Conditional check for cover letter generation    | Switch                         | Cover Letter Agent, Merge        |                                              |
| Cover Letter Agent      | LangChain Agent               | Generates personalized cover letter               | If                             | Merge                          |                                              |
| Google Gemini Chat Model1| LM Chat Google Gemini          | Language model for Cover Letter Agent             | Cover Letter Agent (ai_languageModel) | Cover Letter Agent          |                                              |
| HTML to PDF             | HTTP Request                  | Converts optimized resume text to PDF             | AI Agent                       | Merge                         |                                              |
| Merge                   | Merge                         | Combines resume and cover letter PDFs             | HTML to PDF, Cover Letter Agent (conditional) | Send a message               |                                              |
| Send a message          | Gmail                         | Sends final PDF via email                          | Merge                         | —                              |                                              |
| Sticky Note             | Sticky Note                   | Comments/notes                                     | —                                | —                              |                                              |
| Sticky Note1            | Sticky Note                   | Comments/notes                                     | —                                | —                              |                                              |
| Sticky Note2            | Sticky Note                   | Comments/notes                                     | —                                | —                              |                                              |
| Sticky Note3            | Sticky Note                   | Comments/notes                                     | —                                | —                              |                                              |
| Sticky Note4            | Sticky Note                   | Comments/notes                                     | —                                | —                              |                                              |
| Sticky Note5            | Sticky Note                   | Comments/notes                                     | —                                | —                              |                                              |
| Sticky Note6            | Sticky Note                   | Comments/notes                                     | —                                | —                              |                                              |
| Sticky Note7            | Sticky Note                   | Comments/notes                                     | —                                | —                              |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Add a "Form Trigger" node named "On form submission" to start the workflow on user input. Configure webhook as default.

2. **Add Google Drive 'Get Files' Node:**  
   - Add "Google Drive" node named "Get PDF Files/File". Configure it to retrieve file metadata based on input from form submission (e.g., file ID or query).

3. **Add Google Drive 'Download File' Node:**  
   - Add "Google Drive" node named "Download Retrieval Files/File". Connect from "Get PDF Files/File". Configure to download the file content.

4. **Add 'Extract from File' Node:**  
   - Add "Extract from File" node named "Extract Files/File's Data". Connect from "Download Retrieval Files/File". Configure for text extraction from PDF.

5. **Add 'Set' Node to Prepare Data:**  
   - Add "Set" node named "Get PDF Data Only". Connect from "Extract Files/File's Data". Configure to isolate or format extracted text as needed.

6. **Add 'Code' Node for Parsing:**  
   - Add "Code" node named "Data Parser & Cleaner". Connect from "Get PDF Data Only". Implement JavaScript for cleaning and structuring resume text.

7. **Add LangChain Agent for Resume Optimization:**  
   - Add LangChain Agent node named "AI Agent". Connect from "Data Parser & Cleaner". Configure it to use the "Google Gemini Chat Model" node as its language model.

8. **Add Google Gemini Chat Model Node:**  
   - Add LangChain LM Chat Google Gemini node named "Google Gemini Chat Model". Connect as AI language model to "AI Agent". Configure credentials/API keys.

9. **Add 'Switch' Node for Cover Letter Generation:**  
   - Add "Switch" node named "Switch". Connect from "On form submission". Define condition(s) to check if cover letter generation is needed.

10. **Add 'If' Node:**  
    - Add "If" node named "If". Connect from "Switch". Configure condition to confirm cover letter generation flag.

11. **Add LangChain Agent for Cover Letter:**  
    - Add LangChain Agent node named "Cover Letter Agent". Connect 'true' output from "If". Configure to use "Google Gemini Chat Model1" as language model.

12. **Add Google Gemini Chat Model Node for Cover Letter:**  
    - Add LangChain LM Chat Google Gemini node named "Google Gemini Chat Model1". Connect as AI language model to "Cover Letter Agent". Configure credentials.

13. **Add HTTP Request Node for PDF Conversion:**  
    - Add "HTTP Request" node named "HTML to PDF". Connect from "AI Agent". Configure to call a PDF conversion API (e.g., PDF.co) to convert AI output to PDF.

14. **Add Merge Node to Combine PDFs:**  
    - Add "Merge" node named "Merge". Connect inputs from "HTML to PDF" and from "Cover Letter Agent" (cover letter output). Configure to merge documents appropriately.

15. **Add Final Merge Node if Needed:**  
    - If workflow branches, ensure a second "Merge" node is present to combine all documents before sending.

16. **Add Gmail Node to Send Email:**  
    - Add "Gmail" node named "Send a message". Connect from "Merge". Configure OAuth2 credentials. Set recipient from form data, include merged PDF as attachment.

17. **Connect all nodes respecting data flow:**  
    - Ensure "On form submission" connects to "Get PDF Files/File" and "Switch".  
    - "Switch" connects to "If".  
    - "If" true to "Cover Letter Agent", false to "Merge".  
    - "Cover Letter Agent" to "Merge".  
    - "AI Agent" to "HTML to PDF" to "Merge".  
    - "Merge" to "Send a message".

18. **Configure Credentials:**  
    - Setup Google Drive OAuth2 credentials.  
    - Setup Google Gemini API credentials for both LM Chat nodes.  
    - Setup Gmail OAuth2 credentials.

19. **Set Default Values & Parameters:**  
    - Configure file paths or IDs for Google Drive nodes based on form inputs.  
    - Define prompt templates or parameters for LangChain Agents.  
    - Configure PDF conversion API URL and authentication.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                            |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer for workflow content                             |
| For PDF conversion, PDF.co or similar API services are recommended to convert HTML/text to PDF.    | PDF.co: https://pdf.co/                                    |
| Google Gemini AI integration requires appropriate API credentials and quota management.            | Google Gemini API documentation                             |
| Gmail node requires OAuth2 credentials with mail sending permissions.                               | Gmail API documentation                                    |

---

This structured document fully describes the workflow’s design, node-level details, and instructions for manual reconstruction without dependency on the original JSON. It anticipates common failure modes and integration specifics for a robust deployment.