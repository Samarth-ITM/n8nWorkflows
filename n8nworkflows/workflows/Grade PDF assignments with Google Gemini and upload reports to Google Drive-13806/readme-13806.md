Grade PDF assignments with Google Gemini and upload reports to Google Drive

https://n8nworkflows.xyz/workflows/grade-pdf-assignments-with-google-gemini-and-upload-reports-to-google-drive-13806


# Grade PDF assignments with Google Gemini and upload reports to Google Drive

# Workflow Reference: Grade PDF assignments with Google Gemini and upload reports to Google Drive

This workflow automates the grading process for engineering assignments. It receives a PDF via a webhook, extracts the text, uses Google Gemini to grade the content against a provided answer script, and finally generates and uploads a CSV report to Google Drive.

---

### 1. Workflow Overview

The workflow is designed for educators to automate repetitive grading tasks while maintaining high accuracy through AI analysis. It is organized into four main functional phases:

*   **1.1 Input Reception & Extraction:** Receives the student's assignment via a Webhook and converts the PDF binary data into searchable text.
*   **1.2 Data Preparation:** Organizes student metadata and loads the reference "Answer Script" (the "Gold Standard" for grading).
*   **1.3 AI Grading Engine:** A LangChain-powered agent uses Google Gemini to compare the student's work against the answer script, assigning marks and generating feedback.
*   **1.4 Report Generation & Storage:** Transforms AI results into structured CSV data and uploads the final report to a specific Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Data Extraction
This block handles the entry point and the transformation of raw files into usable text data.

*   **Nodes Involved:** `Webhook - Upload Test Paper`, `Extract Text from Test Paper`.
*   **Node Details:**
    *   **Webhook - Upload Test Paper:** Acts as the trigger. Configured to accept `POST` requests with a raw body (the PDF file).
    *   **Extract Text from Test Paper:** Uses the "PDF" operation to parse the binary content from the webhook into a text string stored in the JSON metadata.
    *   **Edge Cases:** Failure if the uploaded file is not a valid PDF or if the file size exceeds n8n/memory limits.

#### 2.2 Contextual Setup
This block prepares the environment for the AI by defining who the student is and what the correct answers are.

*   **Nodes Involved:** `Prepare Assignment Data`, `Load Answer Script`.
*   **Node Details:**
    *   **Prepare Assignment Data (Set Node):** Captures `studentName` and `assignmentTitle` from the webhook body. It also maps the text extracted in the previous step to a variable `testPaperText`.
    *   **Load Answer Script (Set Node):** A static configuration node where the instructor inputs the correct answers and the marking scheme (max marks per question).
    *   **Key Expressions:** `{{ $json.body.studentName || 'Unknown Student' }}`.

#### 2.3 Intelligent Grading
The core logic where the AI acts as an "Engineering Professor."

*   **Nodes Involved:** `AI Agent - Grade Assignment`, `Google Gemini Chat Model`, `Structured Output Parser`.
*   **Node Details:**
    *   **AI Agent - Grade Assignment:** A LangChain agent using a detailed prompt template. It receives the `answerScript` and `testPaperText`.
    *   **Google Gemini Chat Model:** Connects to the Gemini (PaLM) API to provide the reasoning capabilities.
    *   **Structured Output Parser:** Forces the AI to return data in a strict JSON format (questions, marks, feedback, total, grade) to ensure subsequent nodes don't fail.
    *   **Potential Failures:** LLM timeouts, API quota limits, or "hallucinations" if the assignment text is poorly extracted.

#### 2.4 Data Formatting & Storage
This block converts the AI's JSON response into a physical file and moves it to the cloud.

*   **Nodes Involved:** `Generate Results Table`, `Prepare CSV Data`, `Convert to CSV File`, `Upload file`.
*   **Node Details:**
    *   **Generate Results Table (Code Node):** Executes JavaScript to build an HTML representation (for internal logs) and a CSV string. It calculates the final percentage and formats the grade.
    *   **Prepare CSV Data (Set Node):** Isolates the CSV string for the file converter.
    *   **Convert to CSV File:** Transforms the text string into a binary file object. The filename is dynamically set using the student's name.
    *   **Upload file (Google Drive):** Connects via OAuth2 to upload the binary file to the root or a specific folder in Google Drive.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - Upload Test Paper | Webhook | Trigger/Input | None | Extract Text from Test Paper | ## Get question paper |
| Extract Text from Test Paper | Extract From File | PDF Parsing | Webhook | Prepare Assignment Data | ## Get question paper |
| Prepare Assignment Data | Set | Metadata Prep | Extract Text | Load Answer Script | ## Get question paper |
| Load Answer Script | Set | Reference Data | Prepare Assignment Data | AI Agent | ## check answers |
| AI Agent - Grade Assignment | AI Agent | Grading Logic | Load Answer Script | Generate Results Table | ## check answers |
| Google Gemini Chat Model | Gemini Chat Model | LLM Provider | None | AI Agent | ## check answers |
| Structured Output Parser | Output Parser | Schema Control | None | AI Agent | ## check answers |
| Generate Results Table | Code | Formatting | AI Agent | Prepare CSV Data | ## upload result to drive |
| Prepare CSV Data | Set | Data Selection | Generate Results Table | Convert to CSV File | ## upload result to drive |
| Convert to CSV File | Convert to File | Binary Creation | Prepare CSV Data | Upload file | ## upload result to drive |
| Upload file | Google Drive | Storage | Convert to CSV File | None | ## upload result to drive |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Webhook** node. Set HTTP Method to `POST`, Path to `grade-assignment`, and enable "Raw Body" in the options.
2.  **PDF Extraction:** Connect an **Extract From File** node. Set operation to "Read PDF".
3.  **Variable Initialization:** 
    *   Add a **Set** node (`Prepare Assignment Data`). Create fields for `studentName`, `assignmentTitle`, and `testPaperText` (referencing the extracted text).
    *   Add another **Set** node (`Load Answer Script`). Create a string variable `answerScript` containing your questions and correct answers.
4.  **AI Configuration:**
    *   Add an **AI Agent** node. Set the prompt to include the `answerScript` and the student's `testPaperText`.
    *   Attach a **Google Gemini Chat Model** node to the Agent (requires Google Cloud API Key).
    *   Attach a **Structured Output Parser** to the Agent to define the JSON schema (Question #, marks, feedback).
5.  **Transformation:**
    *   Add a **Code** node. Use JavaScript to map the AI's JSON output into a CSV string.
    *   Add a **Set** node to extract the `csvData` string from the code output.
6.  **File Generation:** Add a **Convert to File** node. Set "Source Property" to the CSV string and define the binary property name.
7.  **Cloud Storage:** Add a **Google Drive** node. Set the Action to "Upload". Configure credentials and select the destination folder. Map the "File to Upload" to the binary property created in the previous step.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **How it works:** The workflow triggers when a student's test paper is uploaded via a webhook. It extracts text, uses AI to grade, generates a report, and uploads to Drive. | General Overview (Sticky Note) |
| **Setup Step:** Connect your Google Gemini account. | [Google AI Studio](https://aistudio.google.com/) |
| **Setup Step:** Connect your Google Drive account via OAuth2. | n8n Credentials Setup |
| **Adjustment:** Adjust the 'Load Answer Script' node with your specific assignment's correct answers and marking scheme. | Customization Instruction |