Score Resumes Against JDs with Google Drive, Google Sheets, and GPT-4o

https://n8nworkflows.xyz/workflows/score-resumes-against-jds-with-google-drive--google-sheets--and-gpt-4o-8531


# Score Resumes Against JDs with Google Drive, Google Sheets, and GPT-4o

### 1. Workflow Overview

This workflow automates the evaluation of candidate resumes against job descriptions (JDs) stored in Google Drive, using AI (GPT-4o via Azure OpenAI) to score and analyze the fit. It is designed for HR, recruitment teams, or talent acquisition systems aiming to objectively and efficiently assess candidate suitability for specific roles.

The workflow is logically divided into two main parallel pipelines that converge:

- **1.1 Job Description Retrieval and Preparation**  
  Searches for JDs in a dedicated Google Drive folder, downloads PDF files, and extracts text content for AI analysis.

- **1.2 Resume Retrieval and Preparation**  
  Searches for candidate resumes in a separate Google Drive folder, downloads PDF files, and extracts text content for AI analysis.

- **1.3 AI Evaluation and Scoring**  
  Uses the extracted JD and resume texts as input to an AI agent that compares both documents, produces a compatibility score (0–100), identifies missing or bonus qualifications, and summarizes candidate fit in structured JSON.

- **1.4 Results Processing and Storage**  
  Parses the AI JSON output, saves a detailed evaluation report as a text file in Google Drive, and appends or updates the candidate’s evaluation record in a Google Sheets database.

The workflow supports manual triggering and batch processing of multiple JDs and resumes, producing structured, auditable outputs with scoring and qualitative assessments.

---

### 2. Block-by-Block Analysis

#### 2.1 Job Description Retrieval and Preparation

- **Overview:**  
  Locates job description PDF files in a designated Google Drive folder, downloads them, and extracts textual content for AI processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Search JD (Google Drive Search)  
  - Download file (Google Drive Download)  
  - Extract from File (PDF Text Extraction)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; triggers the chain.  
    - Inputs: None  
    - Outputs: Triggers "Search JD" node.  
    - Edge Cases: Manual start only; no automatic scheduling.

  - **Search JD**  
    - Type: Google Drive (File/Folder Search)  
    - Role: Finds all JD files in a specified folder "JD store".  
    - Configuration: Folder ID targeting JD storage folder; returns all files.  
    - Inputs: Trigger from manual node.  
    - Outputs: List of JD files with metadata including file IDs.  
    - Edge Cases: Empty folder returns no files; permissions errors if folder inaccessible.

  - **Download file**  
    - Type: Google Drive (File Download)  
    - Role: Downloads the JD PDF file using its file ID.  
    - Configuration: Takes file ID dynamically from previous search node output.  
    - Credentials: Google Drive OAuth2 required.  
    - Inputs: File ID from "Search JD" node.  
    - Outputs: Binary PDF content.  
    - Edge Cases: File not found, permission denial, download failure.

  - **Extract from File**  
    - Type: PDF Text Extraction  
    - Role: Converts JD PDF binary into plain text.  
    - Configuration: PDF operation, no special options.  
    - Inputs: Binary PDF from "Download file".  
    - Outputs: Extracted text content under `json.text`.  
    - Edge Cases: Corrupt PDFs, extraction errors, poor text quality.

---

#### 2.2 Resume Retrieval and Preparation

- **Overview:**  
  Searches for candidate resume PDFs in a separate Google Drive folder, downloads them, and extracts text for AI analysis.

- **Nodes Involved:**  
  - Search Resume (Google Drive Search)  
  - Download file1 (Google Drive Download)  
  - Extract from File1 (PDF Text Extraction)  

- **Node Details:**

  - **Search Resume**  
    - Type: Google Drive (File/Folder Search)  
    - Role: Retrieves all resume files from "Resume_store" folder.  
    - Configuration: Folder ID for resumes, returns all files.  
    - Inputs: Triggered after JD extraction completes.  
    - Outputs: List of resume files with IDs and metadata.  
    - Edge Cases: Empty folder, permission issues.

  - **Download file1**  
    - Type: Google Drive (File Download)  
    - Role: Downloads each candidate resume PDF.  
    - Configuration: Uses file ID from "Search Resume".  
    - Credentials: Google Drive OAuth2.  
    - Inputs: File ID from "Search Resume".  
    - Outputs: Binary PDF data.  
    - Edge Cases: File missing, download failure.

  - **Extract from File1**  
    - Type: PDF Text Extraction  
    - Role: Converts resume PDF binary data into text.  
    - Configuration: PDF extraction, no options.  
    - Inputs: Binary from "Download file1".  
    - Outputs: Extracted resume text under `json.text`.  
    - Edge Cases: Extraction errors, poor OCR results.

---

#### 2.3 AI Evaluation and Scoring

- **Overview:**  
  Compares extracted JD and resume texts using an AI agent, scoring the candidate’s fit, identifying gaps and bonuses, and generating a summary.

- **Nodes Involved:**  
  - AI Agent (Langchain Agent Node)  
  - Azure OpenAI Chat Model (GPT-4o-mini)  
  - Code (JSON Parsing)  

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain AI Agent (Custom AI node)  
    - Role: Orchestrates prompt construction and sends request to AI model.  
    - Configuration:  
      - Prompt includes job description and candidate resume texts embedded via expressions.  
      - System message instructs the AI to evaluate fit with a structured JSON response.  
      - Scoring scale from 0 to 100 with defined thresholds.  
      - Requires output strictly in JSON format without extra text.  
    - Inputs: Resume text from "Extract from File1" and JD text from "Extract from File".  
    - Outputs: AI raw text output with evaluation.  
    - Edge Cases: API rate limits, malformed prompt, incomplete responses.

  - **Azure OpenAI Chat Model**  
    - Type: Azure OpenAI GPT Model  
    - Role: Executes the AI prompt using "gpt-4o-mini" model.  
    - Configuration: Model set to GPT-4o-mini for cost-effective and capable evaluation.  
    - Credentials: Azure OpenAI API key required.  
    - Inputs: Prompt from AI Agent node.  
    - Outputs: AI response text.  
    - Edge Cases: API errors, timeout, authentication failures.

  - **Code**  
    - Type: Code Node (JavaScript)  
    - Role: Parses AI text response into structured JSON.  
    - Configuration:  
      - Removes markdown code fences (```json ... ```) from AI output.  
      - Attempts JSON.parse, returns error object if parsing fails.  
    - Inputs: AI raw text from AI Agent node.  
    - Outputs: Clean JSON object with fields: score, must_have_gaps, nice_to_have_bonus, summary.  
    - Edge Cases: Parsing failures, invalid JSON output.

---

#### 2.4 Results Processing and Storage

- **Overview:**  
  Saves evaluation summaries as text files in Google Drive and updates a Google Sheets candidate database with scores and summaries.

- **Nodes Involved:**  
  - Create file from text (Google Drive Create File)  
  - Append or update row in sheet (Google Sheets)  

- **Node Details:**

  - **Create file from text**  
    - Type: Google Drive Create File from Text  
    - Role: Saves a textual summary of the evaluation for each candidate.  
    - Configuration:  
      - Filename based on candidate resume file name plus suffix "_result-summary".  
      - Content includes score, must-have gaps, nice-to-have bonuses, and summary.  
      - Stored in "Resume_store" Google Drive folder.  
    - Inputs: JSON evaluation data from Code node.  
    - Credentials: Google Drive OAuth2.  
    - Outputs: Confirmation of file creation.  
    - Edge Cases: Write permission denied, naming collisions.

  - **Append or update row in sheet**  
    - Type: Google Sheets Append or Update Row  
    - Role: Updates candidate database with evaluation results in a structured spreadsheet.  
    - Configuration:  
      - Matches rows by candidate Name to append or update.  
      - Stores fields "Name", "Score", "Summary".  
      - Target spreadsheet and sheet specified by ID and sheet name.  
    - Inputs: Evaluation JSON data from "Create file from text".  
    - Credentials: Google Sheets OAuth2.  
    - Outputs: Updated spreadsheet row confirmation.  
    - Edge Cases: Sheet access denied, schema mismatch, concurrency issues.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                            | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                       |
|----------------------------|------------------------------------|--------------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                     | Initiates the workflow manually             | None                         | Search JD                     | **Start Resume Evaluation Process** - Manual start of workflow                                  |
| Search JD                  | Google Drive Search                 | Finds job description files                  | When clicking ‘Execute workflow’ | Download file                 | **Find Job Descriptions in Drive** - Locates JD PDFs in designated folder                       |
| Download file              | Google Drive Download               | Downloads JD PDF                             | Search JD                    | Extract from File             | **Download Job Description PDF** - Downloads PDF from Drive                                    |
| Extract from File          | Extract from File (PDF Text Extract) | Extracts text content from JD PDF           | Download file                | Search Resume                 | **Extract JD Text Content** - Converts JD PDF to text                                          |
| Search Resume              | Google Drive Search                 | Finds candidate resumes                      | Extract from File            | Download file1                | **Find Candidate Resumes in Drive** - Searches resumes folder                                  |
| Download file1             | Google Drive Download               | Downloads candidate resume PDF               | Search Resume                | Extract from File1            | **Download Candidate Resume PDF** - Downloads resume PDF                                      |
| Extract from File1         | Extract from File (PDF Text Extract) | Extracts text from resume PDF                | Download file1               | AI Agent                     | **Extract Resume Text Content** - Parses resume PDF text                                      |
| AI Agent                  | Langchain AI Agent                  | Compares JD and resume, generates JSON eval | Extract from File1, Extract from File | Code                        | **AI Resume Evaluator Agent** - Evaluates fit and scores candidate                            |
| Azure OpenAI Chat Model    | Azure OpenAI GPT Model              | Provides GPT-4o-mini AI model                | AI Agent (ai_languageModel)  | AI Agent                    | **Azure OpenAI GPT-4 Model** - GPT-4o-mini model for evaluation                               |
| Code                      | Code Node (JS)                     | Parses AI text output into JSON              | AI Agent                    | Create file from text         | **Parse AI Response to JSON** - Cleans and parses AI output                                  |
| Create file from text      | Google Drive Create File from Text | Saves evaluation summary as text file        | Code                        | Append or update row in sheet | **Save Evaluation Report to Drive** - Stores summary text file in Drive                        |
| Append or update row in sheet | Google Sheets Append or Update Row | Updates candidate evaluation database         | Create file from text        | None                         | **Update Candidate Database** - Updates Google Sheets with scores and summaries               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually. No parameters needed.  

2. **Create Google Drive Search Node for JDs**  
   - Type: Google Drive (File/Folder Search)  
   - Parameters:  
     - Folder ID: Set to the JD storage folder ID (e.g., "1KnopVQ7CIdUl5j9ICwijk1RG5w7050nW")  
     - Return all files: True  
   - Credentials: Connect Google Drive OAuth2 API.  
   - Connect manual trigger output to this node.

3. **Create Google Drive Download Node for JD**  
   - Type: Google Drive Download  
   - Parameters:  
     - File ID: Set dynamically from Search JD output, expression `{{$json["id"]}}`  
     - Operation: Download  
   - Credentials: Google Drive OAuth2  
   - Connect Search JD output to this node.

4. **Create Extract from File Node for JD**  
   - Type: Extract from File (PDF)  
   - Parameters: Operation set to "pdf"  
   - Connect Download file output to this node.

5. **Create Google Drive Search Node for Resumes**  
   - Type: Google Drive (File/Folder Search)  
   - Parameters:  
     - Folder ID: Resume storage folder ID (e.g., "1MIvpHU_ZqG76Vov2-D5WlS5dD3UhOMSz")  
     - Return all files: True  
   - Credentials: Google Drive OAuth2  
   - Connect Extract from File output to this node.

6. **Create Google Drive Download Node for Resume**  
   - Type: Google Drive Download  
   - Parameters:  
     - File ID: Dynamic from Search Resume output (`{{$json["id"]}}`)  
     - Operation: Download  
   - Credentials: Google Drive OAuth2  
   - Connect Search Resume output to this node.

7. **Create Extract from File Node for Resume**  
   - Type: Extract from File (PDF)  
   - Parameters: Operation set to "pdf"  
   - Connect Download file1 output to this node.

8. **Create Langchain AI Agent Node**  
   - Type: Langchain Agent (AI evaluation)  
   - Parameters:  
     - Text prompt: Use expression incorporating extracted JD and resume texts, e.g.:  
       ```
       Compare the following job description and resume, then score and evaluate according to the instructions.

       Job Description (JD):
       {{ $('Extract from File').item.json.text }}

       Candidate Resume:
       {{ $json.text }}

       Return the evaluation in the required JSON format.
       ```  
     - System message: Provide instructions for scoring (0-100), identifying must-have gaps, nice-to-have bonuses, and summary.  
   - Connect Extract from File1 output to this node.

9. **Create Azure OpenAI GPT-4 Model Node**  
   - Type: Azure OpenAI Chat Model  
   - Parameters:  
     - Model: gpt-4o-mini  
     - Options: default  
   - Credentials: Azure OpenAI API  
   - Connect as language model for the Langchain AI Agent node.

10. **Create Code Node for JSON Parsing**  
    - Type: Code (JavaScript)  
    - Parameters (example JS code):  
      ```js
      return items.map(item => {
        let text = item.json.output;
        text = text.replace(/```json|```/g, "").trim();
        try {
          return { json: JSON.parse(text) };
        } catch (error) {
          return { json: { error: "Failed to parse JSON", raw: text } };
        }
      });
      ```  
    - Connect AI Agent output to this node.

11. **Create Google Drive Create File from Text Node**  
    - Type: Google Drive Create File from Text  
    - Parameters:  
      - Name: Expression using candidate resume name + "_result-summary"  
      - Content: Template with score, must-have gaps, nice-to-have bonuses, summary fields from JSON output  
      - Folder ID: Same as resume folder  
    - Credentials: Google Drive OAuth2  
    - Connect Code node output to this node.

12. **Create Google Sheets Append or Update Row Node**  
    - Type: Google Sheets Append or Update  
    - Parameters:  
      - Document ID: Target Google Sheets document  
      - Sheet Name: Target sheet within the document  
      - Columns: Map Name, Score, Summary from JSON output  
      - Matching Columns: Name (to update existing rows)  
    - Credentials: Google Sheets OAuth2  
    - Connect Create file from text output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses GPT-4o-mini on Azure OpenAI for cost-effective yet capable natural language evaluation of resumes against job descriptions.                  | Azure OpenAI GPT model: gpt-4o-mini                                                                 |
| Folder IDs for JD and Resume storage must be correctly set in Google Drive and accessible via OAuth2 credentials.                                              | Google Drive folder links provided in node parameters.                                              |
| The scoring system is designed to provide objective, structured evaluation to support HR decision-making, with clear thresholds for fit quality.               | Scoring: 90-100 Excellent, 70-89 Good, 50-69 Moderate, <50 Poor                                     |
| The workflow can be triggered manually, allowing batch or single candidate evaluations as needed.                                                              | Manual Trigger node starts the process.                                                             |
| JSON parsing includes error handling to catch malformed AI output; however, AI prompt design should minimize this risk.                                        | Code node includes JSON parse with fallback error object.                                          |
| Evaluation reports are saved with candidate-specific file names for traceability and audit purposes.                                                           | Reports stored in the same Google Drive folder as resumes.                                         |
| Candidate database updates rely on matching by candidate name; consistency in naming is critical for accurate data updates.                                   | Google Sheets Append or Update Row node uses "Name" as key.                                        |
| The workflow separates JD and resume processing but integrates them for AI comparison, ensuring modularity and clarity.                                        | Parallel pipelines for JD and resumes converge at AI evaluation stage.                              |
| All Google Drive and Sheets nodes require OAuth2 credentials configured in n8n with adequate permissions.                                                     | Credentials must be set up prior to running workflow.                                              |

---

**Disclaimer:**  
The text provided is derived exclusively from a workflow automated with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.