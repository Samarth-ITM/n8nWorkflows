Batch Resume Analysis with Google Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/batch-resume-analysis-with-google-gemini-ai-and-google-sheets-11667


# Batch Resume Analysis with Google Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow automates the batch analysis of candidate resumes submitted via a form, leveraging Google Gemini AI (PaLM) for natural language understanding and Google Sheets for output storage. It is designed for HR teams or recruiters who need to process multiple resumes quickly, extracting meaningful insights and summaries to identify the best candidates efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives form submissions containing a descriptive analysis prompt and multiple resume files.
- **1.2 File Preparation**: Separates each uploaded resume file individually and extracts text content from PDFs.
- **1.3 Data Aggregation**: Aggregates extracted texts from multiple resumes into a single textual input.
- **1.4 AI Processing**: Sends the aggregated text and user-provided analysis instructions to a Google Gemini AI-powered agent for summarization.
- **1.5 Output Storage**: Appends the AI-generated summary to a Google Sheets spreadsheet for record-keeping and further review.
- **1.6 User Guidance**: Includes sticky notes with instructions and setup details to assist users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures user input through a web form, including a textual description of the desired resume analysis and multiple uploaded resume files.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - **Type:** Form Trigger  
  - **Role:** Entry point; triggers workflow on form submissions.  
  - **Configuration:**  
    - Form titled "Resume Analysis"  
    - Two fields:  
      - Text field labeled "Describe the Analysis to be done" (required)  
      - File upload field allowing multiple resumes (PDFs)  
    - Webhook ID uniquely identifies the form trigger endpoint.  
  - **Key Variables:**  
    - User’s analysis description: `$json["Describe the Analysis to be done"]`  
    - Uploaded files accessible via binary data.  
  - **Connections:** Outputs to "map files independently" node.  
  - **Potential Failures:**  
    - Missing required field ("Describe the Analysis to be done")  
    - File upload errors or unsupported file types  
    - Webhook connectivity issues  

---

#### 2.2 File Preparation

**Overview:**  
This block separates each uploaded file into individual items for independent processing and extracts textual content from each PDF resume.

**Nodes Involved:**  
- map files independently  
- Extract from File

**Node Details:**  

- **map files independently**  
  - **Type:** Code (JavaScript)  
  - **Role:** Iterates over multiple uploaded files, creating one workflow item per file to enable parallel processing.  
  - **Configuration:**  
    - Extracts file names and MIME types for each file binary.  
    - Handles cases where file names or extensions might be missing, assigns default extensions based on MIME type.  
  - **Key Expressions:**  
    - Iterates over `items[0].binary` object entries.  
  - **Connections:** Outputs individual file items to "Extract from File".  
  - **Potential Failures:**  
    - Missing or malformed binary data  
    - Files without extensions or unknown MIME types  

- **Extract from File**  
  - **Type:** Extract From File (PDF)  
  - **Role:** Extracts text content from each uploaded PDF resume.  
  - **Configuration:**  
    - Operation set to "pdf" for PDF text extraction.  
  - **Connections:** Outputs extracted text to "Aggregate" node.  
  - **Potential Failures:**  
    - Corrupt or scanned PDFs with no extractable text  
    - Large files causing timeouts or memory issues  
    - Unsupported file formats if they slip through  

---

#### 2.3 Data Aggregation

**Overview:**  
Aggregates the extracted text from all resumes into a single concatenated string for analysis by the AI agent.

**Nodes Involved:**  
- Aggregate

**Node Details:**  

- **Aggregate**  
  - **Type:** Aggregate  
  - **Role:** Combines text fields from multiple input items into a single output.  
  - **Configuration:**  
    - Field to aggregate: "text" extracted from PDFs  
    - Default aggregation method (concatenate) assumed  
  - **Connections:** Outputs aggregated text to "Resume analysis Agent".  
  - **Potential Failures:**  
    - Empty or missing text fields  
    - Extremely large aggregated text causing AI model input limits  

---

#### 2.4 AI Processing

**Overview:**  
Uses Google Gemini AI to analyze the aggregated resume texts according to the user's descriptive prompt and generate a concise summary identifying key insights.

**Nodes Involved:**  
- Google Gemini Chat Model (for AI language model credentials)  
- Resume analysis Agent

**Node Details:**  

- **Google Gemini Chat Model**  
  - **Type:** LangChain LM Chat Node (Google Gemini)  
  - **Role:** Provides AI language model capabilities to the downstream agent node.  
  - **Configuration:**  
    - Uses Google Gemini (PaLM API) credentials configured externally.  
  - **Connections:** Feeds AI language model to "Resume analysis Agent".  
  - **Potential Failures:**  
    - Authentication errors with Google PaLM API  
    - API rate limits or quota exceeded  
    - Network or timeout errors  

- **Resume analysis Agent**  
  - **Type:** LangChain Agent Node  
  - **Role:** Runs AI agent with system instructions and input text to generate analysis summary.  
  - **Configuration:**  
    - Input text: Aggregated resume text (`{{$json.text}}`)  
    - System message: User’s "Describe the Analysis to be done" field, appended with instructions to summarize concisely in plain text, ≤200 words, no formatting, capturing identity details and key insights.  
    - Prompt type: "define" (custom prompt defined)  
  - **Connections:** Outputs AI-generated summary to "Append row in sheet".  
  - **Potential Failures:**  
    - AI generation errors or empty responses  
    - Input text exceeds token limits  
    - System message containing unsupported characters or formatting  

---

#### 2.5 Output Storage

**Overview:**  
Appends the AI-generated summary into a designated Google Sheets document for tracking and further analysis.

**Nodes Involved:**  
- Append row in sheet

**Node Details:**  

- **Append row in sheet**  
  - **Type:** Google Sheets  
  - **Role:** Adds a new row with the AI summary into a specified sheet.  
  - **Configuration:**  
    - Target Sheet ID and Sheet Name specified (named "AI Summary")  
    - Defines a single column "AI summary" mapped to the AI output (`{{$json.output}}`)  
    - Mapping mode set to define below with no type conversion.  
  - **Credentials:** Uses OAuth2 Google Sheets account credentials.  
  - **Connections:** Final output node.  
  - **Potential Failures:**  
    - Authorization errors with Google Sheets API  
    - Sheet or document ID mismatch  
    - Network or permission issues  
    - Data formatting errors  

---

#### 2.6 User Guidance

**Overview:**  
Provides workflow usage instructions, setup steps, and important notes to users and administrators.

**Nodes Involved:**  
- Sticky Note

**Node Details:**  

- **Sticky Note**  
  - **Type:** Sticky Note (UI only)  
  - **Role:** Contains detailed explanation of workflow purpose, usage instructions, setup tips, and links.  
  - **Content Summary:**  
    - Explains batch resume upload and AI-driven analysis benefits  
    - Notes on form setup for multiple files  
    - Instructions on system message configuration for job descriptions  
    - Link to a Google Sheets template for reference:  
      https://docs.google.com/spreadsheets/d/1XbMRGqX6N2xP92ZF73LFd09f-PVmGefaAlwOwR980Jk/edit?gid=0#gid=0  
  - **Connections:** No input/output; purely informational.  
  - **Potential Issues:** None  

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                          | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                        |
|-------------------------|-----------------------------------|----------------------------------------|-----------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger                      | Entry point; receives form data        | —                     | map files independently  | Explains batch resume upload workflow, setup steps, and links to Google Sheets template.                          |
| map files independently  | Code (JavaScript)                 | Splits multiple uploaded files into separate items | On form submission      | Extract from File         | Explains batch resume upload workflow, setup steps, and links to Google Sheets template.                          |
| Extract from File        | Extract From File (PDF)           | Extracts text from each PDF resume     | map files independently | Aggregate                | Explains batch resume upload workflow, setup steps, and links to Google Sheets template.                          |
| Aggregate               | Aggregate                        | Combines extracted texts into one text | Extract from File       | Resume analysis Agent    | Explains batch resume upload workflow, setup steps, and links to Google Sheets template.                          |
| Google Gemini Chat Model | LangChain LM Chat Node (Google Gemini) | Provides AI language model interface  | — (credential only)    | Resume analysis Agent (ai_languageModel input) |                                                                                                                    |
| Resume analysis Agent    | LangChain Agent Node             | Runs AI analysis and generates summary | Aggregate, Google Gemini Chat Model | Append row in sheet       |                                                                                                                    |
| Append row in sheet      | Google Sheets                    | Appends AI summary to Google Sheets   | Resume analysis Agent  | —                        |                                                                                                                    |
| Sticky Note             | Sticky Note                      | User instructions and workflow info   | —                     | —                        | Explains batch resume upload workflow, setup steps, and links to Google Sheets template.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Type: Form Trigger  
   - Configure:  
     - Form Title: "Resume Analysis"  
     - Form Description: "Upload the Candidates Resume/Cover letters"  
     - Fields:  
       - Text Field: Label "Describe the Analysis to be done", required  
       - File Upload Field: Label "Attach the resumes", allow multiple files  
   - Save and note the webhook URL generated.

2. **Add a Code Node to Map Files Independently**  
   - Type: Code (JavaScript)  
   - Paste the following logic to split each uploaded file into separate items:

```javascript
const output = [];

for (const [key, value] of Object.entries(items[0].binary || {})) {
  const originalName = value.fileName || 'unknown';
  const extension = originalName.includes('.') ? originalName.split('.').pop() : (value.mimeType?.split('/')[1] || 'bin');
  const fileName = originalName.includes('.') ? originalName : `unknown.${extension}`;

  output.push({
    binary: {
      data: value,
    },
    json: {
      originalKey: key,
      fileName: fileName,
      mimeType: value.mimeType || 'unknown',
    },
  });
}

return output;
```

3. **Add Extract From File Node**  
   - Type: Extract From File  
   - Operation: PDF  
   - Input: Connect from the Code node (map files independently)  
   - This extracts the text from each resume PDF.

4. **Add Aggregate Node**  
   - Type: Aggregate  
   - Configure to aggregate the "text" field from all extracted files into a single concatenated string.

5. **Add Google Gemini Chat Model Node**  
   - Type: LangChain LM Chat Node (Google Gemini)  
   - Credentials: Configure with your Google PaLM API credentials (OAuth or API key)  
   - No special parameters needed; this node serves as AI model provider.

6. **Add Resume Analysis Agent Node**  
   - Type: LangChain Agent Node  
   - Input Text: Use expression `{{$json.text}}` from Aggregate node  
   - System Message:  
     - Use expression to capture form input: `{{$node["On form submission"].json["Describe the Analysis to be done"]}}`  
     - Append instructions:  
       ```
       Summarize the analysis into brief solid insights. 
       Offer a general overall summary, based on all key aspects.
       Your output response should capture identity details and be clean.
       Your output must be plain text, no bold, no #, no symbols, less than 200 words, easy to read through.
       ```  
   - Prompt Type: Define (custom prompt)  
   - Connect AI language model input from Google Gemini Chat Model node.

7. **Add Append Row in Sheet Node**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your target Google Sheets document ID  
   - Sheet Name: The sheet tab name (e.g., "AI Summary")  
   - Columns Mapping: Map column "AI summary" to `{{$json.output}}` from Resume analysis Agent node  
   - Credentials: Use Google Sheets OAuth2 credentials with edit access

8. **Connect Nodes as Follows:**  
   - On form submission → map files independently → Extract from File → Aggregate → Resume analysis Agent → Append row in sheet  
   - Google Gemini Chat Model → Resume analysis Agent (as ai_languageModel input)

9. **Add Sticky Note (Optional)**  
   - Add a sticky note with the workflow description, usage instructions, and a link to your Google Sheets template for reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow allows uploading up to 20 PDF resumes at once, automating heavy lifting in resume screening with AI for high accuracy and speed.                                   | User instructions in Sticky Note                                                                                          |
| Adjust the system message in the Resume analysis Agent node to fit specific job descriptions and qualifications for tailored AI analysis.                                       | Sticky Note instructions                                                                                                  |
| Google Sheets template used for storing AI summaries: [Make a copy](https://docs.google.com/spreadsheets/d/1XbMRGqX6N2xP92ZF73LFd09f-PVmGefaAlwOwR980Jk/edit?gid=0#gid=0)       | Sticky Note link                                                                                                          |
| Google PaLM API credentials must be properly configured with required permissions for the AI nodes to function correctly.                                                       | Credentials setup in Google Gemini Chat Model node                                                                        |
| Ensure Google Sheets OAuth2 credentials have write access to the target spreadsheet to prevent authorization errors.                                                            | Credentials setup in Append row in sheet node                                                                             |
| Large files or scanned PDFs may cause text extraction failures or empty outputs; consider pre-processing resumes to ensure searchable text content.                             | Edge case consideration in Extract from File node                                                                         |
| The AI system message limits output to plain text, under 200 words, and no formatting to ensure clean insertion into Google Sheets and easy human reading.                      | Important prompt instruction in Resume analysis Agent node                                                                |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly follows current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.