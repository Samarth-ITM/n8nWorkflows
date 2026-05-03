Generate Job-Specific ATS Resumes with Perplexity AI and PDF Export

https://n8nworkflows.xyz/workflows/generate-job-specific-ats-resumes-with-perplexity-ai-and-pdf-export-7637


# Generate Job-Specific ATS Resumes with Perplexity AI and PDF Export

---

### 1. Workflow Overview

This workflow, titled **"Generate Job-Specific ATS Resumes with Perplexity AI and PDF Export"**, is designed to automate the creation of tailored, ATS (Applicant Tracking System)-friendly resumes by integrating candidate resume data with job descriptions (JDs). The final output is a polished PDF resume optimized for ATS parsing and keyword alignment.

**Target Use Cases:**  
- Job seekers wanting to customize their resumes for specific job descriptions quickly.  
- HR or recruiting services automating resume tailoring to improve applicant screening quality.  
- Resume writing services providing AI-enhanced resume customization.

**Logical Blocks:**  
- **1.1 Input Reception:** Captures user inputs (resume text or resume PDF + job description PDF) via a web form.  
- **1.2 Binary File Processing & Text Extraction:** Prepares and extracts text content from uploaded PDFs.  
- **1.3 Text Merging:** Combines extracted resume and JD texts into a single string for AI processing.  
- **1.4 AI-Powered Resume Customization:** Uses Perplexity AI to generate a clean, ATS-compliant HTML resume tailored to the JD.  
- **1.5 HTML Cleaning & Preview:** Processes the AI-generated HTML for formatting and preview purposes.  
- **1.6 PDF Generation & Upload:** Converts the HTML resume into PDF format and uploads it to Google Drive.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow by presenting a user-accessible web form to collect resume and job description inputs. It ensures the required files are received and passed for processing.

**Nodes Involved:**  
- Form Trigger

**Node Details:**  

- **Form Trigger**  
  - *Type & Role:* `formTrigger` node; acts as the entry point for user input.  
  - *Configuration:*  
    - Webhook path: `resume-builder`  
    - Form title: "resume builder"  
    - Fields:  
      - Optional plain text resume input (textarea).  
      - Required resume PDF upload (single file, `.pdf` only).  
      - Required job description PDF upload (single file, `.pdf` only).  
    - Response mode: `lastNode` (waits for workflow completion before responding).  
  - *Expressions/Variables:* None directly; outputs JSON and binary data from form submission.  
  - *Connections:* Outputs to `Process one binary file1`.  
  - *Version:* v1  
  - *Edge Cases / Failure:*  
    - Missing required PDFs causes form submission failure.  
    - Large file uploads may timeout or fail.  
    - Invalid file types rejected by form validation.  
  - *Sticky Note:* Explains purpose and function as user input point ([see Sticky Note content in Section 5]).

---

#### 1.2 Binary File Processing & Text Extraction

**Overview:**  
This block standardizes and extracts textual content from the uploaded PDF files for downstream text merging and AI processing.

**Nodes Involved:**  
- Process one binary file1  
- Extracting resume1

**Node Details:**  

- **Process one binary file1**  
  - *Type & Role:* `code` node; processes binary input files.  
  - *Configuration:*  
    - JavaScript loops through each input item’s binary data, renaming keys to `Upload_Resume_PDF` to standardize binary property keys.  
    - Outputs JSON with `fileName` and binary data with the new key.  
  - *Expressions/Variables:* Uses `items` array and binary keys manipulation.  
  - *Connections:* Outputs to `Extracting resume1`.  
  - *Version:* v2  
  - *Edge Cases / Failure:*  
    - No binary data found results in empty output.  
    - Unexpected binary key names may cause misnaming.  
  - *Sticky Note:* Explains role in file organization and naming.

- **Extracting resume1**  
  - *Type & Role:* `extractFromFile` node; extracts text from PDFs.  
  - *Configuration:*  
    - Operation: `pdf` extraction.  
    - Binary property name dynamically set to first binary key: `={{ Object.keys($binary)[0] }}`.  
  - *Expressions/Variables:* Dynamic binary key expression ensures correct file is processed.  
  - *Connections:* Outputs extracted text to `Merge Resume + JD1`.  
  - *Version:* v1  
  - *Edge Cases / Failure:*  
    - Corrupted PDFs may cause extraction failure.  
    - Large PDFs may cause timeouts.  
  - *Sticky Note:* Details purpose in text extraction from files.

---

#### 1.3 Text Merging

**Overview:**  
Combines the extracted textual content of the candidate’s resume and the job description into a single text field, preparing input for AI customization.

**Nodes Involved:**  
- Merge Resume + JD1

**Node Details:**  

- **Merge Resume + JD1**  
  - *Type & Role:* `code` node; merges texts from two separate inputs.  
  - *Configuration:*  
    - Concatenates `$json.text` from the first item (resume) and `$item(1).$json["text"]` (job description) separated by a space.  
  - *Expressions/Variables:* Uses `$json.text` and `$item(1).$json["text"]` for merging.  
  - *Connections:* Outputs merged text to `Customize resume1`.  
  - *Version:* v2  
  - *Edge Cases / Failure:*  
    - Missing text in either input results in incomplete merge (should be handled gracefully).  
  - *Sticky Note:* Explains merging role and output.

---

#### 1.4 AI-Powered Resume Customization

**Overview:**  
Uses Perplexity AI’s `sonar-reasoning` model to generate a single HTML ATS-optimized resume by tailoring the candidate’s resume to the job description content.

**Nodes Involved:**  
- Customize resume1

**Node Details:**  

- **Customize resume1**  
  - *Type & Role:* `perplexity` node; AI text generation and formatting.  
  - *Configuration:*  
    - Model: `sonar-reasoning`  
    - Messages:  
      - System prompt defines strict formatting and content rules for an ATS-friendly resume in HTML with inline CSS.  
      - User prompt includes merged resume + JD text with mandatory styling and layout instructions (white background, Arial font, bullet points, section titles, no placeholders or comments).  
    - Simplify output enabled.  
  - *Expressions/Variables:* Uses `{{ $json.merged }}` as input to AI.  
  - *Connections:* Outputs to `HTML format1`.  
  - *Version:* v1  
  - *Edge Cases / Failure:*  
    - AI response may fail or produce invalid HTML if input is malformed or API issues occur.  
    - Rate limits or API authentication errors possible.  
  - *Sticky Note:* Detailed explanation of AI node role and instructions.

---

#### 1.5 HTML Cleaning & Preview

**Overview:**  
Processes the AI-generated HTML to ensure it is clean, properly formatted, and ready for PDF conversion or preview.

**Nodes Involved:**  
- HTML format1  
- HTML Preview  
- HTML to PDF

**Node Details:**  

- **HTML format1**  
  - *Type & Role:* `code` node; cleans AI HTML output.  
  - *Configuration:*  
    - Extracts HTML content between `<html>` and `</html>` tags using regex.  
    - Removes newline characters to produce a compact string.  
    - Outputs cleaned and original response, with error flag if no HTML found.  
  - *Expressions/Variables:* Uses regex to parse HTML from AI response fields: `$json.message`, `$json.content`, `$json.text`.  
  - *Connections:* Outputs to `HTML Preview` and `HTML to PDF`.  
  - *Version:* v2  
  - *Edge Cases / Failure:*  
    - No HTML tags found leads to error message output.  
    - Malformed HTML may not be properly cleaned.  
  - *Sticky Note:* Explains cleaning and formatting role.

- **HTML Preview**  
  - *Type & Role:* `html` node; renders or processes cleaned HTML for preview or further handling.  
  - *Configuration:*  
    - HTML input: `{{ $json.cleanedResponse }}`  
  - *Connections:* No further outputs defined; used for preview or validation.  
  - *Version:* v1.2  
  - *Edge Cases / Failure:*  
    - Invalid HTML input may cause rendering issues.  
  - *Sticky Note:* Indicates use for visualization or debugging.

- **HTML to PDF**  
  - *Type & Role:* `@custom-js/n8n-nodes-pdf-toolkit.html2Pdf`; converts HTML to PDF.  
  - *Configuration:*  
    - HTML input: `{{ $json.cleanedResponse }}`  
  - *Credentials:* Requires `CustomJS account` credentials for PDF conversion service.  
  - *Connections:* Outputs PDF binary to `Upload file`.  
  - *Version:* v1  
  - *Edge Cases / Failure:*  
    - Conversion failures if HTML is invalid or service is unavailable.  
    - Credential errors or quota limits may block PDF generation.  
  - *Sticky Note:* Describes PDF generation purpose.

---

#### 1.6 PDF Upload

**Overview:**  
Uploads the generated PDF resume to a specified Google Drive folder, making the resume accessible and storable in the cloud.

**Nodes Involved:**  
- Upload file

**Node Details:**  

- **Upload file**  
  - *Type & Role:* `googleDrive` node; uploads files to Google Drive.  
  - *Configuration:*  
    - File name: `NEW_RESUNME` (note: typo, likely intended "NEW_RESUME")  
    - Drive: `My Drive`  
    - Folder ID: `1vzNpRjBe1ylcLmJ2TKl4TN40BAeri-HD` (named "Resume")  
  - *Credentials:* Uses `Google Drive account` OAuth2 credentials.  
  - *Connections:* Terminal node (no further outputs).  
  - *Version:* v3  
  - *Edge Cases / Failure:*  
    - Upload failure due to expired or missing credentials.  
    - Folder access permission denied.  
    - Name collisions or overwriting behavior depends on Google Drive settings.  
  - *Sticky Note:* Explains cloud storage finalization.

---

### 3. Summary Table

| Node Name           | Node Type                            | Functional Role                                | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                                        |
|---------------------|------------------------------------|------------------------------------------------|------------------------|-----------------------|--------------------------------------------------------------------------------------------------------------------|
| Form Trigger        | formTrigger                        | Collects user resume text/file and JD file     | None                   | Process one binary file1 | Explains form input role and fields.                                                                              |
| Process one binary file1 | code                            | Standardizes binary file keys for downstream use | Form Trigger           | Extracting resume1     | Details binary file renaming and standardization.                                                                  |
| Extracting resume1  | extractFromFile                    | Extracts text content from PDFs                 | Process one binary file1 | Merge Resume + JD1     | Describes PDF text extraction role.                                                                                |
| Merge Resume + JD1  | code                              | Merges resume and JD text into one string       | Extracting resume1       | Customize resume1      | Explains merging of extracted texts.                                                                               |
| Customize resume1   | perplexity                        | AI generates ATS-friendly tailored resume in HTML | Merge Resume + JD1       | HTML format1           | Details AI customization and strict HTML output guidelines.                                                      |
| HTML format1        | code                              | Cleans and formats AI-generated HTML            | Customize resume1        | HTML Preview, HTML to PDF | Explains HTML cleaning and extraction.                                                                             |
| HTML Preview        | html                              | Renders or previews cleaned HTML                 | HTML format1             | None                  | Describes usage for validation or visualization.                                                                   |
| HTML to PDF         | @custom-js/n8n-nodes-pdf-toolkit.html2Pdf | Converts HTML resume to PDF                      | HTML format1             | Upload file            | Converts cleaned HTML into PDF format using CustomJS service.                                                     |
| Upload file         | googleDrive                       | Uploads PDF resume to Google Drive               | HTML to PDF              | None                  | Uploads final PDF to specified Google Drive folder.                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node**  
   - Type: `formTrigger`  
   - Path: `resume-builder`  
   - Form Title: "resume builder"  
   - Fields:  
     - Text field: Label "Paste Resume text", placeholder "Resume plain text", optional  
     - File upload: Label "Upload Resume PDF", required, accept `.pdf`, single file  
     - File upload: Label "Upload JD PDF", required, accept `.pdf`, single file  
   - Response Mode: `lastNode`

2. **Create Code node named "Process one binary file1"**  
   - Use JavaScript to iterate input items binary data.  
   - Rename each binary key by replacing spaces with `Upload_Resume_PDF`.  
   - Output JSON with `fileName` and binary data under new key.  
   - Connect Form Trigger → Process one binary file1.

3. **Create ExtractFromFile node named "Extracting resume1"**  
   - Operation: `pdf`  
   - Binary Property Name: `={{ Object.keys($binary)[0] }}` (dynamic)  
   - Connect Process one binary file1 → Extracting resume1.

4. **Create Code node named "Merge Resume + JD1"**  
   - JavaScript: concatenate `$json.text` (resume) + space + `$item(1).$json["text"]` (JD) into `merged` field in JSON.  
   - Connect Extracting resume1 → Merge Resume + JD1.

5. **Create Perplexity node named "Customize resume1"**  
   - Model: `sonar-reasoning`  
   - Messages:  
     - System prompt: Provide exact instructions for ATS-friendly resume HTML generation (as detailed in Section 2.4).  
     - User prompt: Input merged resume + JD text (`{{ $json.merged }}`) with strict formatting rules.  
   - Simplify output: true  
   - Connect Merge Resume + JD1 → Customize resume1.  
   - Set up Perplexity API credentials.

6. **Create Code node named "HTML format1"**  
   - JavaScript: extract content between `<html>` and `</html>` tags from `$json.message` or `$json.content` or `$json.text`.  
   - Remove all newline characters.  
   - Output cleaned HTML as `cleanedResponse`.  
   - Connect Customize resume1 → HTML format1.

7. **Create HTML node named "HTML Preview"**  
   - HTML: `{{ $json.cleanedResponse }}`  
   - Connect HTML format1 → HTML Preview.

8. **Create CustomJS `html2Pdf` node named "HTML to PDF"**  
   - Input HTML: `{{ $json.cleanedResponse }}`  
   - Credentials: CustomJS API account (set up beforehand)  
   - Connect HTML format1 → HTML to PDF.

9. **Create Google Drive node named "Upload file"**  
   - Operation: Upload file  
   - File Name: `NEW_RESUNME` (consider correcting typo to `NEW_RESUME`)  
   - Drive: `My Drive`  
   - Folder ID: `1vzNpRjBe1ylcLmJ2TKl4TN40BAeri-HD` (target Resume folder)  
   - Credentials: Google Drive OAuth2 account.  
   - Connect HTML to PDF → Upload file.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| The Form Trigger node creates an interactive web form for users to submit resume and job description PDFs, initiating the workflow.                                                                                                     | See Sticky Note for Form Trigger                             |
| Extracting resume1 node dynamically targets the correct binary file for PDF text extraction, ensuring flexible input handling.                                                                                                           | See Sticky Note for Extracting resume1                      |
| The AI customization node employs strict guidelines for outputting a clean, ATS-friendly resume in HTML format with inline CSS, avoiding any placeholders or comments that could interfere with ATS parsing.                            | See Sticky Note for Customize resume1                       |
| HTML format1 node ensures that only valid, clean HTML is passed downstream, critical for PDF conversion and preview reliability.                                                                                                        | See Sticky Note for HTML format1                            |
| The workflow uses the `@custom-js/n8n-nodes-pdf-toolkit.html2Pdf` node for HTML to PDF conversion, requiring valid CustomJS credentials.                                                                                                | See Sticky Note for HTML to PDF                             |
| Final PDF upload uses Google Drive node to store the resume securely in a specific folder, allowing easy user access post-processing.                                                                                                    | See Sticky Note for Upload file                             |
| For further n8n node details and best practices, refer to the official documentation: [n8n Docs](https://docs.n8n.io/workflows/components/)                                                                                             | Official n8n Documentation                                  |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow, adhering to all applicable content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.