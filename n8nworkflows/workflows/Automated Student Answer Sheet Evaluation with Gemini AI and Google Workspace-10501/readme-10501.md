Automated Student Answer Sheet Evaluation with Gemini AI and Google Workspace

https://n8nworkflows.xyz/workflows/automated-student-answer-sheet-evaluation-with-gemini-ai-and-google-workspace-10501


# Automated Student Answer Sheet Evaluation with Gemini AI and Google Workspace

---

### 1. Workflow Overview

This workflow automates the evaluation of student answer sheets using AI and Google Workspace tools. It is designed for teachers who want to streamline grading by uploading scanned answer sheets and receiving detailed evaluation reports without manual marking.

**Target Use Cases:**
- Educational institutions automating student exam evaluation.
- Teachers needing to assess scanned or photographed answer sheets.
- Integration of AI-based text/image analysis with Google Docs and Google Sheets for record keeping.

**Logical Blocks:**

- **1.1 Input Reception:** Collect teacher input and student answer sheet image via a web form.
- **1.2 AI Processing:** Use Google Gemini AI to analyze the image and extract answers.
- **1.3 AI Evaluation:** Compare extracted answers against Question Paper and Correct Answer Paper stored in Google Docs, calculate marks, and generate structured JSON results.
- **1.4 Data Parsing and Preparation:** Parse AI output into structured data and merge question-wise details into JSON objects.
- **1.5 Results Storage:** Append summary and detailed evaluation results into Google Sheets for record management.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow by collecting the examiner's name and the student's answer sheet image via a form.

**Nodes Involved:**  
- Answer Sheet Uploader

**Node Details:**

- **Answer Sheet Uploader**  
  - *Type:* Form Trigger  
  - *Role:* Collects examiner name and uploads the scanned student answer sheet (image file).  
  - *Configuration:*  
    - Form titled "Examiner" with required "Examiner Name" text field.  
    - File upload field accepting `.png` and `.jpg`.  
  - *Input/Output:*  
    - Input: HTTP request via webhook.  
    - Output: Data including examiner name and uploaded file binary data.  
  - *Failure cases:*  
    - Missing required fields (examiner name).  
    - Unsupported file types or upload failures.  
  - *Sticky Note:* "Form Trigger – Collects Teacher Name and uploads the Student’s scanned Answer Sheet."

---

#### 1.2 AI Processing

**Overview:**  
This block analyzes the uploaded answer sheet image using Google Gemini AI to extract student answers, section names, question numbers, and student name.

**Nodes Involved:**  
- Analyze an image

**Node Details:**

- **Analyze an image**  
  - *Type:* Google Gemini AI Image Analysis node (`@n8n/n8n-nodes-langchain.googleGemini`)  
  - *Role:* Processes the uploaded image binary to extract structured textual data about the student's answers.  
  - *Configuration:*  
    - Model: `models/gemini-2.5-flash`  
    - Input type: binary property named "Upload_Answer_Sheet" (the uploaded image).  
    - Prompt: instructs AI to extract answers with section name, question number, and student name from the image.  
  - *Credentials:* Google Gemini (PaLM) API credentials required.  
  - *Input/Output:*  
    - Input: Binary image file from "Answer Sheet Uploader" node.  
    - Output: Text content parsing the student's answers.  
  - *Failure cases:*  
    - API authentication errors.  
    - Image quality too poor for extraction.  
    - AI response timeout or incomplete data.  
  - *Sticky Note:* "Gemini image Analysis Node – Extracts the student’s answers from the scanned image."

---

#### 1.3 AI Evaluation

**Overview:**  
This block compares the extracted student answers with the official question paper and correct answers, calculates marks, and generates a detailed evaluation summary in JSON.

**Nodes Involved:**  
- AI Agent  
- Question Paper 5Th Class  
- Answer Paper 5th Class  
- Google Gemini Chat Model  
- Structured Output Parser1

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent Node (`@n8n/n8n-nodes-langchain.agent`)  
  - *Role:* Core AI evaluation engine that compares student answers with question and answer papers, calculates marks, and produces final JSON output.  
  - *Configuration:*  
    - System message guides the AI through a multi-step evaluation process: reading question paper, correct answers, student answers, comparing and scoring, and formatting output JSON.  
    - Uses dynamic expressions to pass extracted text and examiner name.  
    - Utilizes tools connected to Google Docs nodes (for question and answer papers).  
  - *Input:*  
    - Text from "Analyze an image" node (student answers).  
    - Google Docs content from "Question Paper 5Th Class" and "Answer Paper 5th Class" via ai_tool inputs.  
    - Google Gemini Chat model as language model backend.  
    - Structured Output Parser1 for output parsing.  
  - *Output:*  
    - JSON with student evaluation including total correct/incorrect answers, marks, and detailed question-wise data.  
  - *Failure cases:*  
    - AI model API errors or rate limits.  
    - Mismatch or missing Google Docs content.  
    - Parsing errors if AI output is malformed.  
  - *Sticky Note:*  
    "AI Evaluation Agent Node – Uses attached Google Docs (Question Paper and Correct Answer Sheet) to compare answers and generate a structured JSON result with correct/incorrect count and marks obtained."

- **Question Paper 5Th Class**  
  - *Type:* Google Docs Tool node  
  - *Role:* Retrieves the question paper document content from Google Docs via URL to supply to AI Agent.  
  - *Configuration:*  
    - Document URL set to a specific Google Docs link.  
    - OAuth2 credentials for Google Docs access.  
  - *Input/Output:*  
    - Output: Document text passed to AI Agent.  
  - *Failure cases:*  
    - Document access permissions or expired link.  
    - OAuth token expiration.  

- **Answer Paper 5th Class**  
  - *Type:* Google Docs Tool node  
  - *Role:* Retrieves the correct answer sheet document content for AI comparison.  
  - *Configuration:*  
    - Document URL set to a specific Google Docs link.  
    - OAuth2 credentials for Google Docs access.  
  - *Failure cases:* Same as Question Paper node.

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini Chat node  
  - *Role:* Language model backend for AI Agent, providing conversational AI capabilities.  
  - *Configuration:*  
    - Model: `models/gemini-2.5-pro`  
    - Credentials: Google Gemini (PaLM) API credentials.

- **Structured Output Parser1**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses the AI Agent's textual output into structured JSON format based on an example schema.  
  - *Configuration:*  
    - JSON schema provided for expected output to ensure consistent parsing.

---

#### 1.4 Data Parsing and Preparation

**Overview:**  
This block transforms the AI-generated JSON output into individual question entries formatted for appending to Google Sheets.

**Nodes Involved:**  
- Code to merge multiple items into one JSON

**Node Details:**

- **Code to merge multiple items into one JSON**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Extracts relevant fields from the AI output JSON and generates an array of JSON objects for each question. Each object includes student and examiner names, question number, correct and student answers, status, and overall marks summary.  
  - *Key code logic:*  
    - Reads AI output from input JSON field "output".  
    - Throws error if no valid data is found.  
    - Maps over the questions array to produce individual JSON items.  
  - *Input/Output:*  
    - Input: AI Agent output JSON.  
    - Output: Array of JSON objects, each representing a question evaluation entry.  
  - *Failure cases:*  
    - Missing or malformed AI output data.  
    - Runtime errors within code node.  
  - *Sticky Note:*  
    "This code merges all the questions into one JSON to append into google sheet."

---

#### 1.5 Results Storage

**Overview:**  
This block appends the summarized and detailed evaluation results into Google Sheets for record keeping.

**Nodes Involved:**  
- Append Summary  
- Append Scorecard

**Node Details:**

- **Append Summary**  
  - *Type:* Google Sheets node  
  - *Role:* Appends a summary row containing student name, examiner name, total questions, counts of correct and incorrect answers, total marks, and marks obtained into a Google Sheet.  
  - *Configuration:*  
    - Spreadsheet ID and sheet "gid=0" specified.  
    - Mapping fields from AI output JSON to sheet columns.  
    - OAuth2 credentials for Google Sheets access.  
  - *Failure cases:*  
    - Google Sheets API permission issues.  
    - Rate limits or quota exceeded.  
  - *Sticky Note:*  
    "Google Sheets (Summary Update) – Appends an entry with Student Name, Teacher Name, Total Questions, Correct Count, Incorrect Count, Total Marks, and Marks Obtained."

- **Append Scorecard**  
  - *Type:* Google Sheets node  
  - *Role:* Appends detailed question-wise evaluation data into a separate Google Sheet tab, including question number, correct answer, student answer, status, and student name.  
  - *Configuration:*  
    - Spreadsheet ID and sheet tab with ID `760946435`.  
    - Field mapping from code node output JSON.  
    - OAuth2 credentials for Google Sheets.  
  - *Failure cases:* Same as Append Summary.  
  - *Sticky Note:*  
    "Google Sheets (Detailed Report Update) – Appends question-wise details with Question, Correct Answer, Student Answer, and Evaluation Status."

---

### 3. Summary Table

| Node Name                      | Node Type                             | Functional Role                                    | Input Node(s)             | Output Node(s)                     | Sticky Note                                                                                   |
|--------------------------------|-------------------------------------|---------------------------------------------------|---------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Answer Sheet Uploader           | Form Trigger                        | Collects Examiner Name and student answer sheet   | -                         | Analyze an image                  | Form Trigger – Collects Teacher Name and uploads the Student’s scanned Answer Sheet.         |
| Analyze an image                | Google Gemini AI Image Analysis     | Extracts student answers from scanned image       | Answer Sheet Uploader      | AI Agent                        | Gemini image Analysis Node – Extracts the student’s answers from the scanned image.          |
| AI Agent                       | LangChain Agent                    | Evaluates answers vs question and answer papers   | Analyze an image, Question Paper 5Th Class, Answer Paper 5th Class, Google Gemini Chat Model, Structured Output Parser1 | Append Summary, Code to merge multiple items into one JSON | AI Evaluation Agent Node – Uses attached Google Docs (Question Paper and Correct Answer Sheet) to compare answers and generate a structured JSON result with correct/incorrect count and marks obtained. |
| Question Paper 5Th Class        | Google Docs Tool                   | Provides official question paper text              | -                         | AI Agent (ai_tool)               |                                                                                              |
| Answer Paper 5th Class          | Google Docs Tool                   | Provides official correct answers                   | -                         | AI Agent (ai_tool)               |                                                                                              |
| Google Gemini Chat Model        | LangChain Google Gemini Chat Model | Language model backend for AI Agent                 | -                         | AI Agent (ai_languageModel)      |                                                                                              |
| Structured Output Parser1       | LangChain Structured Output Parser | Parses AI output into structured JSON               | -                         | AI Agent (ai_outputParser)       |                                                                                              |
| Append Summary                 | Google Sheets                     | Appends summary of evaluation results               | AI Agent                  | Code to merge multiple items into one JSON | Google Sheets (Summary Update) – Appends an entry with Student Name, Teacher Name, Total Questions, Correct Count, Incorrect Count, Total Marks, and Marks Obtained |
| Code to merge multiple items into one JSON | Code (JavaScript)               | Converts AI JSON output into individual question entries | Append Summary            | Append Scorecard                 | This code merges all the questions into one JSON to append into google sheet                  |
| Append Scorecard               | Google Sheets                     | Appends detailed question-wise evaluation data     | Code to merge multiple items into one JSON | -                              | Google Sheets (Detailed Report Update) – Appends question-wise details with Question, Correct Answer, Student Answer, and Evaluation Status. |
| Sticky Note                   | Sticky Note                      | Documentation notes                                | -                         | -                                |                                                                                              |
| Sticky Note1                  | Sticky Note                      | Documentation notes                                | -                         | -                                |                                                                                              |
| Sticky Note2                  | Sticky Note                      | Documentation notes                                | -                         | -                                |                                                                                              |
| Sticky Note3                  | Sticky Note                      | Documentation notes                                | -                         | -                                |                                                                                              |
| Sticky Note4                  | Sticky Note                      | Documentation notes                                | -                         | -                                |                                                                                              |
| Sticky Note5                  | Sticky Note                      | Documentation notes                                | -                         | -                                |                                                                                              |
| Sticky Note6                  | Sticky Note                      | Documentation notes                                | -                         | -                                |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form title as "Examiner"  
   - Add two fields:  
     - "Examiner Name" (text, required)  
     - "Upload Answer Sheet" (file upload, accept `.png`, `.jpg`)  
   - Save and note the webhook URL for triggering.

2. **Add Google Gemini Image Analysis Node**  
   - Type: LangChain Google Gemini (`googleGemini`)  
   - Set operation to "analyze" on resource "image"  
   - Input type: binary, property name matching the file upload field ("Upload_Answer_Sheet")  
   - Model: select `models/gemini-2.5-flash`  
   - Set prompt text instructing to extract answers, section names, question numbers, and student name from the image.  
   - Add Google Gemini API credentials.

3. **Add Google Docs Nodes for Question and Answer Papers**  
   - Type: Google Docs Tool (operation: get)  
   - For Question Paper node, set the document URL to the 5th class question paper Google Docs link  
   - For Answer Paper node, set the document URL to the 5th class correct answer Google Docs link  
   - Add Google Docs OAuth2 credentials to both.

4. **Add Google Gemini Chat Model Node**  
   - Type: LangChain Google Gemini Chat  
   - Select model `models/gemini-2.5-pro`  
   - Add Google Gemini API credentials.

5. **Add LangChain Agent Node**  
   - Configure system message to guide AI through:  
     - Reading question and answer papers, extracting questions and correct answers  
     - Reading student answers from the image analysis result  
     - Comparing answers with tolerance for punctuation and currency symbol variations  
     - Calculating marks (e.g., 2 marks per correct answer)  
     - Generating a comprehensive JSON report including counts and detailed data per question  
   - Pass inputs:  
     - Text from the image analysis node  
     - Google Docs nodes as AI tools for question and answer papers  
     - Google Gemini Chat Model as the language model  
     - Structured Output Parser node for JSON parsing  
   - Use expressions to inject examiner name and extracted answers dynamically.

6. **Add Structured Output Parser Node**  
   - Configure with a JSON schema example that matches the desired output format (student name, examiner name, total correct/incorrect, total marks, marks obtained, and data array of questions).

7. **Add Google Sheets Append Node for Summary**  
   - Configure to append to a Google Sheet (e.g., Score Card spreadsheet, sheet `gid=0`)  
   - Map columns: Student Name, Examiner Name, Correct Answer (count), Incorrect Answers (count), Total Marks, Total Marks Obtain (marks obtained)  
   - Use OAuth2 credentials for Google Sheets.

8. **Add Code Node to Merge AI Output**  
   - Use JavaScript code to parse AI output JSON and create an array of JSON objects, each representing a question's evaluation with fields: question number, correct answer, student answer, status, and overall marks summary.  
   - Handle errors if AI output is missing or malformed.

9. **Add Google Sheets Append Node for Detailed Scorecard**  
   - Configure to append question-wise data into a Google Sheet tab (e.g., sheet ID `760946435`)  
   - Map columns: Student Name, Question Number, Actual Answer, Student Answer, Status  
   - Use OAuth2 credentials for Google Sheets.

10. **Connect Nodes**  
    - Connect Form Trigger → Analyze an image → AI Agent  
    - Connect Question Paper and Answer Paper nodes as AI tools to AI Agent  
    - Connect Google Gemini Chat Model as language model to AI Agent  
    - Connect Structured Output Parser as output parser to AI Agent  
    - Connect AI Agent output to Append Summary and Code Node in parallel  
    - Connect Code Node output to Append Scorecard  

11. **Test the Workflow**  
    - Upload a sample answer sheet and examiner name via the form.  
    - Verify AI extracts answers correctly.  
    - Verify evaluation JSON output correctness and parsing.  
    - Confirm data appends properly to Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow automates student answer evaluation to reduce manual grading workload for teachers by leveraging AI and Google Workspace integration. Future enhancements could include multi-page answer sheet handling and customizable grading formats.                                                                                                                                                                                                                                                                                                      | Sticky Note6 content                                                                                      |
| The AI Agent uses Google Docs as tools to fetch question and answer papers, ensuring the evaluation logic dynamically references official documents.                                                                                                                                                                                                                                                                                                                                                                                                         | AI Agent node system message                                                                              |
| Google Sheets integration is used for both summary and detailed scorecard updates, enabling easy data review and archival in spreadsheet format.                                                                                                                                                                                                                                                                                                                                                                                                           | Append Summary and Append Scorecard nodes                                                                 |
| Google Gemini API credentials and OAuth2 credentials for Google Docs and Sheets must be correctly set up and maintained for uninterrupted workflow operation.                                                                                                                                                                                                                                                                                                                                                                                               | Credential notes across nodes                                                                              |
| The workflow expects image uploads in PNG or JPG formats with sufficient clarity for AI to extract text; image quality issues may reduce accuracy.                                                                                                                                                                                                                                                                                                                                                                                                          | Analyze an image node considerations                                                                       |
| For more details on Google Gemini (PaLM) API and LangChain integration, refer to official n8n documentation and Google Cloud AI resources.                                                                                                                                                                                                                                                                                                                                                                                                                | Official docs (external)                                                                                   |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.

---