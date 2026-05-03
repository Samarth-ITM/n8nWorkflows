Screen CVs and score candidates with Gmail, Google Drive, OpenAI, and Sheets

https://n8nworkflows.xyz/workflows/screen-cvs-and-score-candidates-with-gmail--google-drive--openai--and-sheets-13970


# Screen CVs and score candidates with Gmail, Google Drive, OpenAI, and Sheets

# 1. Workflow Overview

This workflow automates candidate intake and screening from incoming email applications. It watches a Gmail inbox for new emails with CV attachments, stores the CV in Google Drive, detects the file type, extracts resume text, evaluates the candidate with an OpenAI-powered recruiter agent against a predefined job description, extracts core candidate identity fields, and appends the final results into Google Sheets.

Typical use cases:
- HR teams screening inbound applications automatically
- Recruiters centralizing resumes and scoring candidates consistently
- Companies building a lightweight AI-assisted candidate review pipeline without a full ATS

## 1.1 Application Intake and File Storage
The workflow starts when a new email arrives in Gmail. It downloads the attachment and uploads the CV into Google Drive so the file can be processed in a consistent location.

## 1.2 File Type Detection and Extraction Routing
Once the CV is stored in Drive, the workflow inspects the file MIME type and routes it to the correct extraction path for PDF, DOCX, or TXT files.

## 1.3 Resume Text Extraction and Standardization
Each file type is processed differently to extract usable text. The extracted text is normalized into a common `text` field.

## 1.4 Job Description Retrieval
In parallel with resume normalization, the workflow downloads a Google Docs file representing the target job description, converts it to PDF, and extracts its text.

## 1.5 AI-Based Candidate Evaluation
The standardized resume text and extracted job description text are sent into an AI recruiter agent. The agent uses an OpenAI chat model and a structured output parser to produce a consistent screening report.

## 1.6 Candidate Identity Extraction and Result Storage
After the AI evaluation, a second AI node extracts the candidate’s first name, last name, and email from the resume text. These identity fields plus the screening results are written to Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Application Intake and CV Storage

**Overview:**  
This block listens for new candidate emails in Gmail and uploads the first attachment into Google Drive. It establishes the source event and persists the CV for downstream processing.

**Nodes Involved:**  
- On New Candidate Email
- New CV

### Node: On New Candidate Email
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Event trigger that polls Gmail for new incoming emails.
- **Configuration choices:**  
  - Uses Gmail OAuth2 credentials
  - Polling interval set to every minute
  - Attachment download enabled
  - `simple` mode disabled, so the workflow receives richer Gmail message data
- **Key expressions or variables used:**  
  None in node parameters
- **Input and output connections:**  
  - Input: none, this is an entry point
  - Output: `New CV`
- **Version-specific requirements:**  
  Type version `1.2`
- **Edge cases or potential failure types:**  
  - Gmail OAuth expiration or revocation
  - Emails without attachments
  - Multiple attachments: this workflow only uses `attachment_0`
  - Gmail polling delays or quota/rate limits
  - Unexpected attachment type or malformed email payload
- **Sub-workflow reference:**  
  None

### Node: New CV
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the email attachment into Google Drive as a file.
- **Configuration choices:**  
  - File name is generated as `{{ $json.subject }} CV`
  - Upload target is a specific folder in “My Drive”
  - Reads binary input from `attachment_0`
- **Key expressions or variables used:**  
  - `={{ $json.subject }} CV`
- **Input and output connections:**  
  - Input: `On New Candidate Email`
  - Output: `Route by File Type`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - Missing `attachment_0`
  - Google Drive OAuth/auth scope issues
  - Folder access denied
  - Duplicate or ambiguous file names
  - Large attachment upload failures
- **Sub-workflow reference:**  
  None

---

## 2.2 File Type Detection and Extraction Routing

**Overview:**  
This block determines whether the uploaded CV is a DOCX, PDF, or TXT file and sends it through the correct extraction branch. This is necessary because each format requires a different parsing approach.

**Nodes Involved:**  
- Route by File Type
- Extract Text from Docx
- Download Word File
- Download PDF File
- Download TXT File

### Node: Route by File Type
- **Type and technical role:** `n8n-nodes-base.switch`  
  Branching node based on the uploaded Drive file’s MIME type.
- **Configuration choices:**  
  - Three named outputs:
    - `Word` for DOCX
    - `PDF` for PDF
    - `TXT` for plain text
  - Strict string equality matching on `$json.mimeType`
- **Key expressions or variables used:**  
  - `={{ $json.mimeType }}`
- **Input and output connections:**  
  - Input: `New CV`
  - Outputs:
    - `Extract Text from Docx`
    - `Download PDF File`
    - `Download TXT File`
- **Version-specific requirements:**  
  Type version `3.2`
- **Edge cases or potential failure types:**  
  - Unsupported MIME types (no default branch exists)
  - Files uploaded with unexpected or generic MIME types
  - Empty or missing `mimeType`
- **Sub-workflow reference:**  
  None

### Node: Extract Text from Docx
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Google Drive API directly to copy and convert a DOCX file into a Google-native format.
- **Configuration choices:**  
  - POST request to Google Drive v2 API
  - Uses predefined Google Drive OAuth2 credential type
  - URL copies the file by ID with `convert=True`
- **Key expressions or variables used:**  
  - `=https://www.googleapis.com/drive/v2/files/{{ $json.id }}/copy?convert=True`
- **Input and output connections:**  
  - Input: `Route by File Type` (Word branch)
  - Output: `Download Word File`
- **Version-specific requirements:**  
  Type version `4.2`
  - Uses Drive API v2 behavior; this can be sensitive to API deprecations or tenant policy restrictions
- **Edge cases or potential failure types:**  
  - Google Drive API permissions insufficient for copy/convert
  - API version behavior changes
  - Non-DOCX documents routed incorrectly
  - HTTP 403/404/429/5xx
- **Sub-workflow reference:**  
  None

### Node: Download Word File
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads the converted Google Doc as a PDF so it can be parsed using the same PDF extraction approach.
- **Configuration choices:**  
  - File ID taken from the previous node output
  - Download operation
  - Google file conversion enabled: Docs to PDF
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input: `Extract Text from Docx`
  - Output: `Parse Word Content`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - The copied file not created successfully
  - Conversion failures from Google Doc to PDF
  - Permission issues on copied file
- **Sub-workflow reference:**  
  None

### Node: Download PDF File
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads a PDF CV from Drive for parsing.
- **Configuration choices:**  
  - File ID comes from the Drive upload result
  - Standard download operation with no conversion
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input: `Route by File Type` (PDF branch)
  - Output: `Parse PDF Content`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - Missing file ID
  - Download permission errors
  - Corrupt PDF file
- **Sub-workflow reference:**  
  None

### Node: Download TXT File
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads a plain text CV file from Drive.
- **Configuration choices:**  
  - File ID comes from the Drive upload result
  - Standard download operation
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input: `Route by File Type` (TXT branch)
  - Output: `Parse Text Content`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - Missing file ID
  - Permission issues
  - Text file encoding issues
- **Sub-workflow reference:**  
  None

---

## 2.3 Resume Text Extraction and Standardization

**Overview:**  
This block parses the actual content of the CV regardless of original format and normalizes it into a single `text` property. It provides a stable input shape for the later AI steps.

**Nodes Involved:**  
- Parse Word Content
- Parse PDF Content
- Parse Text Content
- Standardize

### Node: Parse Word Content
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts text from the PDF version of the original DOCX.
- **Configuration choices:**  
  - Operation set to `pdf`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Download Word File`
  - Output: `Standardize`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - OCR-like limitations if the PDF is image-based
  - Bad conversion resulting in unreadable PDF
  - Empty extraction on complex layouts
- **Sub-workflow reference:**  
  None

### Node: Parse PDF Content
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts text directly from PDF CV files.
- **Configuration choices:**  
  - Operation set to `pdf`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Download PDF File`
  - Output: `Standardize`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Scanned PDFs with no embedded text
  - Multi-column resumes producing poorly ordered text
  - Corrupt binary data
- **Sub-workflow reference:**  
  None

### Node: Parse Text Content
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Reads text from a plain text resume.
- **Configuration choices:**  
  - Operation set to `text`
  - Destination key explicitly set to `text`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Download TXT File`
  - Output: `Standardize`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Character encoding issues
  - Empty file
  - Strange delimiters or formatting artifacts
- **Sub-workflow reference:**  
  None

### Node: Standardize
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes upstream extraction output into a consistent schema containing only `text`.
- **Configuration choices:**  
  - Creates a string field `text`
  - Value copied from current item’s `text`
- **Key expressions or variables used:**  
  - `={{ $json.text }}`
- **Input and output connections:**  
  - Inputs: `Parse Word Content`, `Parse PDF Content`, `Parse Text Content`
  - Output: `saves CV in folder`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Missing `text` from prior parser
  - Blank extraction resulting in empty AI input
- **Sub-workflow reference:**  
  None

---

## 2.4 Job Description Retrieval

**Overview:**  
This block fetches a predefined Google Docs job description, converts it to PDF, and extracts its text. That extracted text becomes the job description context used by the recruiter agent.

**Nodes Involved:**  
- saves CV in folder
- Extract Final Text

### Node: saves CV in folder
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Despite its name, this node does not save the candidate CV. It downloads a specific Google Docs file and converts it to PDF.
- **Configuration choices:**  
  - File ID is hardcoded to a specific Google Docs document named “Job Title: AI Solutions Architect”
  - Operation is `download`
  - Google file conversion: Docs to PDF
- **Key expressions or variables used:**  
  None in parameters
- **Input and output connections:**  
  - Input: `Standardize`
  - Output: `Extract Final Text`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - Misleading node name may confuse maintenance
  - Access denied to job description file
  - Wrong or outdated file ID
  - Job description unavailable or deleted
- **Sub-workflow reference:**  
  None

### Node: Extract Final Text
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts the text of the job description from the downloaded PDF.
- **Configuration choices:**  
  - Operation set to `pdf`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `saves CV in folder`
  - Output: `Recruiter Agent`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Empty or malformed job description PDF
  - Poor text extraction from formatted job description documents
- **Sub-workflow reference:**  
  None

---

## 2.5 AI Recruiter Evaluation

**Overview:**  
This block compares the candidate CV text against the job description using an LLM-backed agent. It enforces a structured schema so the result can be stored reliably downstream.

**Nodes Involved:**  
- Recruiter Agent
- OpenAI Chat Model
- Structured Output Parser

### Node: Recruiter Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent node that receives the candidate resume as primary text and the job description inside the system prompt.
- **Configuration choices:**  
  - Prompt type is `define`
  - Main text input is the candidate resume:
    - `Candidates Resume:\n\n{{ $('Standardize').item.json.text }}`
  - System message defines detailed recruiter behavior and output expectations
  - The job description is inserted dynamically from current input item:
    - `{{ $json.text }}`
  - Structured output parser attached
- **Key expressions or variables used:**  
  - `{{ $('Standardize').item.json.text }}`
  - `{{ $json.text }}`
- **Input and output connections:**  
  - Main input: `Extract Final Text`
  - AI language model input: `OpenAI Chat Model`
  - AI output parser input: `Structured Output Parser`
  - Main output: `Information Extractor`
- **Version-specific requirements:**  
  Type version `2`
  - Depends on n8n LangChain integration compatibility
- **Edge cases or potential failure types:**  
  - Model output not matching parser schema
  - Long CV + long job description causing token pressure
  - Hallucinated or overly generic reasoning
  - Empty candidate or job text
  - LLM/API timeout or quota exhaustion
- **Sub-workflow reference:**  
  None

### Node: OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM used by both the recruiter agent and information extractor.
- **Configuration choices:**  
  - Model selected: `o4-mini`
  - OpenAI API credential configured
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output to:
    - `Recruiter Agent`
    - `Information Extractor`
- **Version-specific requirements:**  
  Type version `1.2`
- **Edge cases or potential failure types:**  
  - Invalid OpenAI API key
  - Model availability changes
  - API quotas, rate limits, or latency spikes
- **Sub-workflow reference:**  
  None

### Node: Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a strict JSON schema for the recruiter agent output.
- **Configuration choices:**  
  - Manual JSON schema with required fields:
    - `candidate_strengths`
    - `candidate_weaknesses`
    - `risk_factor`
    - `reward_factor`
    - `overall_fit_rating`
    - `justification_for_rating`
  - Risk and reward scores restricted to `Low`, `Medium`, `High`
  - Overall fit rating restricted to integer 0–10
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI output parser connected to `Recruiter Agent`
- **Version-specific requirements:**  
  Type version `1.2`
- **Edge cases or potential failure types:**  
  - Schema mismatch if the LLM returns malformed structure
  - Arrays/objects serialized unexpectedly
  - Integer range violations
- **Sub-workflow reference:**  
  None

---

## 2.6 Candidate Identity Extraction and Result Storage

**Overview:**  
This block extracts core identity fields from the candidate CV text and appends a final record into Google Sheets. It combines outputs from multiple earlier nodes into one row.

**Nodes Involved:**  
- Information Extractor
- new candidate created

### Node: Information Extractor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.informationExtractor`  
  AI extraction node that pulls specific structured candidate identity fields from the CV text.
- **Configuration choices:**  
  - Input text is the standardized resume text:
    - `={{ $('Standardize').item.json.text }}`
  - Required attributes:
    - First name
    - Last name
    - email
  - Uses the same OpenAI chat model as the recruiter agent
- **Key expressions or variables used:**  
  - `={{ $('Standardize').item.json.text }}`
- **Input and output connections:**  
  - Main input: `Recruiter Agent`
  - AI language model input: `OpenAI Chat Model`
  - Main output: `new candidate created`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - Candidate CV missing a visible email
  - Ambiguous or multi-part names
  - LLM extraction errors or partial compliance
  - Dependence on `Recruiter Agent` means identity extraction only happens if evaluation succeeds
- **Sub-workflow reference:**  
  None

### Node: new candidate created
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a candidate screening record into a Google Sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Uses mapping mode “define below”
  - Appends into a specific spreadsheet and sheet tab
  - Mapped fields include:
    - `CV` from `New CV.webViewLink`
    - `Day` from `$now`
    - `Name` from extracted first name
    - `Last name` from extracted last name
    - `Email` from extracted email
    - `Strength` from recruiter output strengths
    - `Weaknesses` from recruiter output weaknesses
    - `Risk factor` from recruiter output risk object
    - `Reward factor` from recruiter output reward object
    - `Justification` from recruiter output justification
- **Key expressions or variables used:**  
  - `={{ $('New CV').item.json.webViewLink }}`
  - `={{ $now }}`
  - `={{ $json.output['First name'] }}`
  - `={{ $json.output['Last name'] }}`
  - `={{ $json.output.email }}`
  - `={{ $('Recruiter Agent').item.json.output.candidate_strengths }}`
  - `={{ $('Recruiter Agent').item.json.output.candidate_weaknesses }}`
  - `={{ $('Recruiter Agent').item.json.output.risk_factor }}`
  - `={{ $('Recruiter Agent').item.json.output.reward_factor }}`
  - `={{ $('Recruiter Agent').item.json.output.justification_for_rating }}`
- **Input and output connections:**  
  - Input: `Information Extractor`
  - Output: none
- **Version-specific requirements:**  
  Type version `4.6`
- **Edge cases or potential failure types:**  
  - Google Sheets auth or sharing issues
  - Object/array values may be written as serialized strings rather than human-friendly formatting
  - Column name mismatches if the sheet schema changes
  - Missing extracted name/email causing blank cells
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On New Candidate Email | Gmail Trigger | Entry point; monitors Gmail for new candidate emails and downloads attachments |  | New CV | ## A. Application intake & CV storage<br>This section monitors incoming job applications through Gmail.<br>When a candidate sends an email containing a CV attachment, the workflow triggers automatically and stores the file in Google Drive. This ensures all resumes are centralized and accessible for processing. |
| New CV | Google Drive | Uploads the incoming email attachment to Google Drive | On New Candidate Email | Route by File Type | ## A. Application intake & CV storage<br>This section monitors incoming job applications through Gmail.<br>When a candidate sends an email containing a CV attachment, the workflow triggers automatically and stores the file in Google Drive. This ensures all resumes are centralized and accessible for processing. |
| Route by File Type | Switch | Routes uploaded CVs by MIME type | New CV | Extract Text from Docx; Download PDF File; Download TXT File | ## B. File type detection<br>This step identifies the type of CV file received.<br>Depending on whether the resume is a PDF, Word document, or text file, the workflow routes the file to the appropriate extraction method so the content can be processed correctly. |
| Extract Text from Docx | HTTP Request | Converts DOCX in Drive into a Google-native file via Drive API copy/convert | Route by File Type | Download Word File | ## B. File type detection<br>This step identifies the type of CV file received.<br>Depending on whether the resume is a PDF, Word document, or text file, the workflow routes the file to the appropriate extraction method so the content can be processed correctly. |
| Download Word File | Google Drive | Downloads converted Google Doc as PDF | Extract Text from Docx | Parse Word Content | ## B. File type detection<br>This step identifies the type of CV file received.<br>Depending on whether the resume is a PDF, Word document, or text file, the workflow routes the file to the appropriate extraction method so the content can be processed correctly. |
| Download PDF File | Google Drive | Downloads uploaded PDF CV | Route by File Type | Parse PDF Content | ## B. File type detection<br>This step identifies the type of CV file received.<br>Depending on whether the resume is a PDF, Word document, or text file, the workflow routes the file to the appropriate extraction method so the content can be processed correctly. |
| Download TXT File | Google Drive | Downloads uploaded TXT CV | Route by File Type | Parse Text Content | ## B. File type detection<br>This step identifies the type of CV file received.<br>Depending on whether the resume is a PDF, Word document, or text file, the workflow routes the file to the appropriate extraction method so the content can be processed correctly. |
| Parse Word Content | Extract From File | Extracts text from PDF-converted DOCX | Download Word File | Standardize | ## C. CV text extraction & standardization<br>In this section, the workflow extracts the full textual content of the resume and standardizes the format.<br>This ensures the AI agent receives a clean, structured input regardless of the original CV format. |
| Parse PDF Content | Extract From File | Extracts text from PDF CV | Download PDF File | Standardize | ## C. CV text extraction & standardization<br>In this section, the workflow extracts the full textual content of the resume and standardizes the format.<br>This ensures the AI agent receives a clean, structured input regardless of the original CV format. |
| Parse Text Content | Extract From File | Extracts text from TXT CV | Download TXT File | Standardize | ## C. CV text extraction & standardization<br>In this section, the workflow extracts the full textual content of the resume and standardizes the format.<br>This ensures the AI agent receives a clean, structured input regardless of the original CV format. |
| Standardize | Set | Normalizes extracted CV content into a common `text` field | Parse Word Content; Parse PDF Content; Parse Text Content | saves CV in folder | ## C. CV text extraction & standardization<br>In this section, the workflow extracts the full textual content of the resume and standardizes the format.<br>This ensures the AI agent receives a clean, structured input regardless of the original CV format. |
| saves CV in folder | Google Drive | Downloads the fixed job description Google Doc as PDF | Standardize | Extract Final Text | ## C. CV text extraction & standardization<br>In this section, the workflow extracts the full textual content of the resume and standardizes the format.<br>This ensures the AI agent receives a clean, structured input regardless of the original CV format. |
| Extract Final Text | Extract From File | Extracts text from the job description PDF | saves CV in folder | Recruiter Agent | ## D. AI recruiter evaluation<br>This is the core of the workflow.<br>The AI recruiter agent analyzes the candidate profile and evaluates:<br>- Professional strengths<br>- Weaknesses<br>- Skills and experience<br>- Potential contribution to the company<br><br>The candidate is also assigned a score from 1 to 10 indicating how well they match the role. |
| Recruiter Agent | LangChain Agent | Compares CV text against the job description and outputs a structured screening report | Extract Final Text; OpenAI Chat Model; Structured Output Parser | Information Extractor | ## D. AI recruiter evaluation<br>This is the core of the workflow.<br>The AI recruiter agent analyzes the candidate profile and evaluates:<br>- Professional strengths<br>- Weaknesses<br>- Skills and experience<br>- Potential contribution to the company<br><br>The candidate is also assigned a score from 1 to 10 indicating how well they match the role. |
| OpenAI Chat Model | OpenAI Chat Model | Supplies the LLM for AI evaluation and data extraction |  | Recruiter Agent; Information Extractor | ## D. AI recruiter evaluation<br>This is the core of the workflow.<br>The AI recruiter agent analyzes the candidate profile and evaluates:<br>- Professional strengths<br>- Weaknesses<br>- Skills and experience<br>- Potential contribution to the company<br><br>The candidate is also assigned a score from 1 to 10 indicating how well they match the role. |
| Structured Output Parser | Structured Output Parser | Enforces the AI screening response schema |  | Recruiter Agent | ## D. AI recruiter evaluation<br>This is the core of the workflow.<br>The AI recruiter agent analyzes the candidate profile and evaluates:<br>- Professional strengths<br>- Weaknesses<br>- Skills and experience<br>- Potential contribution to the company<br><br>The candidate is also assigned a score from 1 to 10 indicating how well they match the role. |
| Information Extractor | Information Extractor | Extracts candidate first name, last name, and email from resume text | Recruiter Agent; OpenAI Chat Model | new candidate created | ## D. AI recruiter evaluation<br>This is the core of the workflow.<br>The AI recruiter agent analyzes the candidate profile and evaluates:<br>- Professional strengths<br>- Weaknesses<br>- Skills and experience<br>- Potential contribution to the company<br><br>The candidate is also assigned a score from 1 to 10 indicating how well they match the role. |
| new candidate created | Google Sheets | Appends candidate details and AI evaluation to Google Sheets | Information Extractor |  | ## E. Candidate data storage<br>Finally, the structured evaluation results are saved to Google Sheets.<br>This creates a searchable database of candidate profiles and AI evaluations that recruiters can easily review and compare. |
| Sticky Note | Sticky Note | Documentation/comment block for workflow purpose |  |  | ## Who this template is for<br><br>This workflow is designed for HR teams, recruiters, and companies that receive job applications via email and want to automatically analyze and evaluate candidates using AI.<br><br>It is ideal for organizations that process a high volume of resumes and want to standardize candidate evaluation, reduce manual screening, and quickly identify the best applicants.<br><br>No advanced n8n knowledge is required, and the workflow can easily be extended with additional evaluation logic, ATS integrations, or automated interview scheduling.<br><br>What this workflow does<br><br>This workflow automatically processes incoming job applications, extracts information from the candidate's CV, and evaluates the candidate using an AI recruiter agent.<br><br>When a candidate sends their CV to the company email:<br><br>1. The workflow detects the new email.<br><br>2. The attached CV is uploaded and stored in Google Drive.<br><br>3. The system detects the file type (PDF, DOCX, TXT).<br><br>4. The text content of the CV is extracted.<br><br>5. The extracted information is sent to an AI recruiter agent.<br><br>6. The AI analyzes the candidate and evaluates:<br>- Strengths<br>- Weaknesses<br>- Skills and knowledge<br>- Potential value for the company<br><br>7. The candidate receives a score from 1 to 10 based on their suitability for the role.<br><br>8. The results are saved automatically in Google Sheets for easy review by the hiring team.<br><br>How it works<br><br>The workflow listens for incoming emails using a Gmail trigger. When a new email containing a CV is received, the attachment is uploaded to Google Drive for processing.<br><br>A routing step detects the file format of the resume and selects the appropriate extraction method depending on whether the CV is a PDF, Word document, or text file.<br><br>The system then extracts the full textual content of the resume and standardizes it into a consistent format so it can be analyzed reliably.<br><br>An AI recruiter agent processes the candidate profile using an OpenAI chat model. The agent evaluates the candidate’s background, identifies strengths and weaknesses, and estimates how well the candidate fits the role.<br><br>The evaluation is then structured using a Structured Output Parser, extracting key attributes such as candidate insights and suitability score.<br><br>Finally, the results are stored in Google Sheets, creating a structured database of candidate evaluations that recruiters can easily review and filter. |
| Sticky Note1 | Sticky Note | Documentation for intake and storage block |  |  | ## A. Application intake & CV storage<br><br>This section monitors incoming job applications through Gmail.<br><br>When a candidate sends an email containing a CV attachment, the workflow triggers automatically and stores the file in Google Drive. This ensures all resumes are centralized and accessible for processing. |
| Sticky Note2 | Sticky Note | Documentation for file type routing block |  |  | ## B. File type detection<br><br>This step identifies the type of CV file received.<br><br>Depending on whether the resume is a PDF, Word document, or text file, the workflow routes the file to the appropriate extraction method so the content can be processed correctly. |
| Sticky Note3 | Sticky Note | Documentation for extraction/standardization block |  |  | ## C. CV text extraction & standardization<br><br>In this section, the workflow extracts the full textual content of the resume and standardizes the format.<br><br>This ensures the AI agent receives a clean, structured input regardless of the original CV format. |
| Sticky Note4 | Sticky Note | Documentation for AI evaluation block |  |  | ## D. AI recruiter evaluation<br><br>This is the core of the workflow.<br><br>The AI recruiter agent analyzes the candidate profile and evaluates:<br>- Professional strengths<br>- Weaknesses<br>- Skills and experience<br>- Potential contribution to the company<br><br>The candidate is also assigned a score from 1 to 10 indicating how well they match the role. |
| Sticky Note5 | Sticky Note | Documentation for storage block |  |  | ## E. Candidate data storage<br><br>Finally, the structured evaluation results are saved to Google Sheets.<br><br>This creates a searchable database of candidate profiles and AI evaluations that recruiters can easily review and compare. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Building CV automation`.

2. **Add a Gmail Trigger node**
   - Node type: **Gmail Trigger**
   - Name: `On New Candidate Email`
   - Configure Gmail OAuth2 credentials.
   - Set:
     - `Download Attachments` = enabled
     - Poll interval = every minute
     - `Simple` = false
   - This node will be the entry point.

3. **Add a Google Drive node to upload the attachment**
   - Node type: **Google Drive**
   - Name: `New CV`
   - Configure Google Drive OAuth2 credentials.
   - Operation: upload/create file
   - Set:
     - File/folder destination to your target Drive folder
     - File name = `{{ $json.subject }} CV`
     - Binary property = `attachment_0`
     - Drive = `My Drive`
   - Connect `On New Candidate Email -> New CV`

4. **Add a Switch node for MIME-type routing**
   - Node type: **Switch**
   - Name: `Route by File Type`
   - Create 3 outputs with renamed output labels:
     1. `Word`
     2. `PDF`
     3. `TXT`
   - Add conditions:
     - Word: `$json.mimeType` equals `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
     - PDF: `$json.mimeType` equals `application/pdf`
     - TXT: `$json.mimeType` equals `text/plain`
   - Connect `New CV -> Route by File Type`

5. **Create the DOCX conversion branch**
   - Add an **HTTP Request** node
   - Name: `Extract Text from Docx`
   - Method: `POST`
   - URL:
     `https://www.googleapis.com/drive/v2/files/{{ $json.id }}/copy?convert=True`
   - Authentication:
     - Predefined credential type
     - Credential type = Google Drive OAuth2 API
   - Connect `Route by File Type (Word) -> Extract Text from Docx`

6. **Add a Google Drive download node for converted Word files**
   - Node type: **Google Drive**
   - Name: `Download Word File`
   - Operation: `download`
   - File ID = `{{ $json.id }}`
   - Enable Google file conversion:
     - Docs to format = `application/pdf`
   - Connect `Extract Text from Docx -> Download Word File`

7. **Add a parser for Word-derived PDF content**
   - Node type: **Extract From File**
   - Name: `Parse Word Content`
   - Operation: `pdf`
   - Connect `Download Word File -> Parse Word Content`

8. **Create the PDF branch**
   - Add a **Google Drive** node
   - Name: `Download PDF File`
   - Operation: `download`
   - File ID = `{{ $json.id }}`
   - Connect `Route by File Type (PDF) -> Download PDF File`

9. **Add a parser for PDF resumes**
   - Node type: **Extract From File**
   - Name: `Parse PDF Content`
   - Operation: `pdf`
   - Connect `Download PDF File -> Parse PDF Content`

10. **Create the TXT branch**
    - Add a **Google Drive** node
    - Name: `Download TXT File`
    - Operation: `download`
    - File ID = `{{ $json.id }}`
    - Connect `Route by File Type (TXT) -> Download TXT File`

11. **Add a parser for TXT resumes**
    - Node type: **Extract From File**
    - Name: `Parse Text Content`
    - Operation: `text`
    - Destination key = `text`
    - Connect `Download TXT File -> Parse Text Content`

12. **Add a Set node to standardize resume text**
    - Node type: **Set**
    - Name: `Standardize`
    - Keep a field:
      - `text` (string) = `{{ $json.text }}`
    - Connect:
      - `Parse Word Content -> Standardize`
      - `Parse PDF Content -> Standardize`
      - `Parse Text Content -> Standardize`

13. **Add a Google Drive node to fetch the job description**
    - Node type: **Google Drive**
    - Name: `saves CV in folder`
    - Important: this node is really used to download the job description, so you may want to rename it in your own version.
    - Operation: `download`
    - Select the Google Docs file containing the job description
    - Enable Google file conversion:
      - Docs to format = `application/pdf`
    - Connect `Standardize -> saves CV in folder`

14. **Add a file extraction node for the job description**
    - Node type: **Extract From File**
    - Name: `Extract Final Text`
    - Operation: `pdf`
    - Connect `saves CV in folder -> Extract Final Text`

15. **Add the OpenAI chat model**
    - Node type: **OpenAI Chat Model**
    - Name: `OpenAI Chat Model`
    - Configure OpenAI credentials
    - Select model: `o4-mini`

16. **Add the Structured Output Parser**
    - Node type: **Structured Output Parser**
    - Name: `Structured Output Parser`
    - Schema type: manual
    - Paste a schema with:
      - `candidate_strengths` as array of strings
      - `candidate_weaknesses` as array of strings
      - `risk_factor` object with `score` and `explanation`
      - `reward_factor` object with `score` and `explanation`
      - `overall_fit_rating` integer from 0 to 10
      - `justification_for_rating` string
    - Mark all of them required

17. **Add the recruiter agent**
    - Node type: **AI Agent**
    - Name: `Recruiter Agent`
    - Prompt type: `define`
    - Main text input:
      `Candidates Resume:\n\n{{ $('Standardize').item.json.text }}`
    - System message should instruct the model to:
      - act as an expert technical recruiter
      - compare candidate resume to the job description
      - identify strengths and weaknesses
      - assign risk and reward factors
      - determine short-term or long-term fit
      - output a 0–10 overall fit rating
      - justify the rating using evidence from the resume and job description
    - Include the job description in the system message as:
      `{{ $json.text }}`
    - Enable structured output parsing
    - Connect:
      - `Extract Final Text -> Recruiter Agent`
      - `OpenAI Chat Model -> Recruiter Agent` using AI language model connection
      - `Structured Output Parser -> Recruiter Agent` using AI output parser connection

18. **Add the information extraction node**
    - Node type: **Information Extractor**
    - Name: `Information Extractor`
    - Input text:
      `{{ $('Standardize').item.json.text }}`
    - Add required attributes:
      - `First name`
      - `Last name`
      - `email`
    - Connect:
      - `Recruiter Agent -> Information Extractor`
      - `OpenAI Chat Model -> Information Extractor` using AI language model connection

19. **Add a Google Sheets append node**
    - Node type: **Google Sheets**
    - Name: `new candidate created`
    - Configure Google Sheets OAuth2 credentials
    - Operation: `append`
    - Choose your spreadsheet and target sheet tab
    - Mapping mode: define below
    - Map the columns as follows:
      - `CV` = `{{ $('New CV').item.json.webViewLink }}`
      - `Day` = `{{ $now }}`
      - `Name` = `{{ $json.output['First name'] }}`
      - `Last name` = `{{ $json.output['Last name'] }}`
      - `Email` = `{{ $json.output.email }}`
      - `Strength` = `{{ $('Recruiter Agent').item.json.output.candidate_strengths }}`
      - `Weaknesses` = `{{ $('Recruiter Agent').item.json.output.candidate_weaknesses }}`
      - `Risk factor` = `{{ $('Recruiter Agent').item.json.output.risk_factor }}`
      - `Reward factor` = `{{ $('Recruiter Agent').item.json.output.reward_factor }}`
      - `Justification` = `{{ $('Recruiter Agent').item.json.output.justification_for_rating }}`
    - Enable append behavior
    - Connect `Information Extractor -> new candidate created`

20. **Optionally add sticky notes**
    - Add notes for:
      - intake/storage
      - file routing
      - extraction/standardization
      - AI evaluation
      - Sheets storage

21. **Test with one PDF CV first**
    - Send an email with one PDF attachment to the monitored Gmail inbox.
    - Confirm:
      - Gmail trigger fires
      - File uploads to Drive
      - PDF branch is selected
      - Text is extracted
      - Job description text is fetched
      - AI output is structured
      - Candidate row appears in Sheets

22. **Test DOCX and TXT variants**
    - Validate the alternate extraction paths.
    - Pay special attention to the DOCX conversion via Drive API.

23. **Harden the workflow before production**
    - Add an IF node before upload if you want to reject emails without attachments.
    - Add fallback handling for unsupported MIME types.
    - Consider serializing arrays/objects into readable text before writing to Sheets.
    - Consider renaming `saves CV in folder` to something clearer like `Download Job Description`.

## Credential configuration required
- **Gmail OAuth2**
  - Must allow reading inbox and attachments
- **Google Drive OAuth2**
  - Must allow file upload, file copy/convert, and file download
- **Google Sheets OAuth2**
  - Must allow append/write access to the target spreadsheet
- **OpenAI API**
  - Must allow access to the selected chat model

## Input/output expectations
- **Workflow input:** new Gmail message, ideally with one CV attachment
- **Intermediate outputs:** uploaded Drive file, extracted resume text, extracted job description text, AI evaluation object, identity extraction object
- **Final output:** one appended Google Sheets row per processed candidate

## No sub-workflow usage
This workflow does not call any sub-workflow and does not expose itself as a sub-workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is designed for HR teams, recruiters, and companies that receive job applications via email and want to automatically analyze and evaluate candidates using AI. | General purpose |
| It is ideal for organizations that process a high volume of resumes and want to standardize candidate evaluation, reduce manual screening, and quickly identify the best applicants. | General purpose |
| No advanced n8n knowledge is required, and the workflow can easily be extended with additional evaluation logic, ATS integrations, or automated interview scheduling. | Extension guidance |
| The workflow automatically processes incoming job applications, extracts information from the candidate's CV, and evaluates the candidate using an AI recruiter agent. | General behavior |
| Processing flow described in the notes: detect email, upload CV, detect file type, extract text, send to AI recruiter, analyze strengths/weaknesses/skills/value, assign score, save to Google Sheets. | Overall process |
| Important maintenance note: the node named `saves CV in folder` actually downloads the job description file, not the candidate CV. Renaming it would improve maintainability. | Implementation detail |
| Important data-model note: the recruiter output fields `candidate_strengths`, `candidate_weaknesses`, `risk_factor`, and `reward_factor` are arrays/objects and may not render cleanly in Google Sheets without formatting or string conversion. | Storage consideration |
| Important logic note: the recruiter agent receives the CV text from `Standardize` and the job description text from `Extract Final Text`. This dependency is implemented through expressions rather than by merging data in a dedicated node. | Architecture note |