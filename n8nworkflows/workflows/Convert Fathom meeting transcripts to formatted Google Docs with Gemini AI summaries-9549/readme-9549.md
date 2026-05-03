Convert Fathom meeting transcripts to formatted Google Docs with Gemini AI summaries

https://n8nworkflows.xyz/workflows/convert-fathom-meeting-transcripts-to-formatted-google-docs-with-gemini-ai-summaries-9549


# Convert Fathom meeting transcripts to formatted Google Docs with Gemini AI summaries

### 1. Workflow Overview

This workflow automates the transformation of Fathom meeting transcripts into well-structured, AI-analyzed meeting notes saved as formatted Google Docs. It is designed for users who want to leverage AI-driven summaries and structured outputs from meeting transcripts without manual effort, integrating Fathom, Google Gemini AI, and Google Drive/Docs services.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Validation**  
  Receive Fathom meeting data via webhook, parse and format transcript data, and validate the presence of a meaningful transcript before further processing.

- **1.2 AI-Based Meeting Analysis**  
  Send the formatted transcript to Google Gemini AI for structured meeting analysis, parsing the output into categorized meeting insights.

- **1.3 Data Formatting & Google Docs Conversion**  
  Format the AI-generated analysis and original transcript into styled HTML, upload this HTML to Google Drive, convert it to a native Google Doc, improve document layout, and clean up temporary files.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Validation

**Overview:**  
This block ingests raw Fathom meeting data via webhook, extracts and normalizes key transcript and meeting metadata, then checks if the transcript contains at least 3 conversation turns to ensure meaningful content before proceeding.

**Nodes Involved:**  
- Get Fathom Meeting (Webhook)  
- Format Key Parts (Code)  
- Transcript Present? (If)  
- Sticky Note (Instructional)

**Node Details:**

- **Get Fathom Meeting**  
  - *Type:* Webhook  
  - *Role:* Entry point receiving POST requests directly from Fathom API webhook when a meeting ends.  
  - *Configuration:* Listens on a unique webhook path, expects raw body including binary data property named "data."  
  - *Input/Output:* No input; outputs raw JSON meeting data.  
  - *Failures:* Network errors, invalid payloads, webhook security not detailed here.  
  - *Notes:* This node triggers the entire workflow.

- **Format Key Parts**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses raw Fathom payload; extracts meeting metadata like title, attendees, timings; processes the transcript by merging consecutive lines from the same speaker, sorting by timestamp, and creating a normalized transcript structure with per-line timestamps.  
  - *Key expressions:*  
    - Converts timestamps (HH:MM:SS) to seconds for sorting  
    - Deduplicates attendees by email or name  
    - Merges transcript lines by speaker  
  - *Output:* JSON with structured fields like meeting_title, attendees array, scheduled/recording times, merged transcript array, and concatenated transcript text.  
  - *Failures:* Malformed timestamps, missing fields, transcript array missing or empty could cause incorrect output.  
  - *Version:* Uses n8n Code node v2.

- **Transcript Present?**  
  - *Type:* If Condition  
  - *Role:* Validates that the merged transcript exists and contains at least 3 conversation turns before continuing to AI analysis.  
  - *Conditions:*  
    - `transcript_merged` must not be empty string  
    - `transcript_merged.length >= 3`  
  - *Failures:* If conditions fail, workflow halts or could branch differently (not shown).  
  - *Purpose:* Avoid unnecessary AI calls and costs for insignificant or empty meetings.

- **Sticky Note (Capture Meeting and Validate)**  
  - *Type:* Sticky Note (Documentation)  
  - *Content:* Explains the purpose of this block: to capture meeting data, extract and format content, and validate transcript presence before triggering AI processing.

---

#### 2.2 AI-Based Meeting Analysis

**Overview:**  
This block sends the structured transcript data to Google Gemini Chat Model for detailed meeting analysis, using a custom prompt to generate a structured summary including key points, decisions, action items, and other meeting insights. It parses the AI output into usable fields.

**Nodes Involved:**  
- Google Gemini Chat Model (Google PaLM API Node)  
- Structured Output Parser  
- AI Meeting Analysis (LangChain LLM Chain)  
- Set Fields (Data Assignment)  
- Sticky Note1 (Documentation)

**Node Details:**

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini Chat Model Node  
  - *Role:* Interface to Google Gemini (PaLM) API, providing AI language model capabilities.  
  - *Configuration:* Model set to "models/gemini-2.5-pro" with default options.  
  - *Credentials:* Uses Google Palm API credentials (OAuth/API key).  
  - *Input:* Receives prompt and transcript data prepared by AI Meeting Analysis node via LangChain integration.  
  - *Failures:* API quota exceeded, invalid credentials, network timeout, malformed prompt.  
  - *Version:* TypeVersion 1.

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser (Structured)  
  - *Role:* Parses AI output JSON according to predefined schema example covering meeting title, URL, times, attendees, executive summary, key points, actions, decisions, risks, questions, and entity extraction.  
  - *Configuration:* Uses a detailed JSON schema example as template.  
  - *Failures:* Parsing errors if AI output does not match schema, malformed JSON, missing fields.  
  - *Version:* 1.3.

- **AI Meeting Analysis**  
  - *Type:* LangChain Chain LLM node  
  - *Role:* Constructs a detailed prompt that instructs the AI to analyze the meeting transcript and generate a structured report with multiple meeting sections (summary, key points, action items, decisions, risks, questions, entities).  
  - *Prompt Details:*  
    - Includes meeting metadata (title, URLs, times, attendees)  
    - Instructs AI to produce concise, non-fluffy output  
    - Specifies output format and fallback strings when data is missing ("None identified in the transcript")  
    - Embeds entire merged transcript as JSON string for AI analysis  
  - *Input:* Receives normalized transcript and meeting metadata from Format Key Parts node.  
  - *Output:* Parsed structured meeting report JSON.  
  - *Failures:* AI model processing errors, prompt formatting issues, incomplete data leading to incomplete summaries.  
  - *Version:* 1.7.

- **Set Fields**  
  - *Type:* Set Node (Data Transformation)  
  - *Role:* Maps and formats AI output fields into a flat output object, including combined meeting title with timestamp, and passes merged transcript data for downstream use.  
  - *Expressions:* Uses templated expressions to concatenate strings, replace colons in timestamps, and pass arrays.  
  - *Failures:* Expression evaluation errors if input missing or malformed.  
  - *Version:* 3.4.

- **Sticky Note1 (Perform Meeting Analysis AI)**  
  - *Type:* Sticky Note  
  - *Content:* Describes that this block performs full AI analysis of meeting transcript and generates notes including summary and actions.

---

#### 2.3 Data Formatting & Google Docs Conversion

**Overview:**  
This block formats the AI-generated meeting analysis and transcript into HTML, uploads the HTML file to Google Drive, converts it into a native Google Doc, improves page margins for readability, and deletes the temporary HTML file to keep storage clean.

**Nodes Involved:**  
- Create as HTML (Code)  
- Upload File as HTML (Google Drive)  
- Convert to Google Doc (HTTP Request)  
- Improve Page Layout (HTTP Request)  
- Delete HTML File (Google Drive)  
- Sticky Note2 (Documentation)

**Node Details:**

- **Create as HTML**  
  - *Type:* Code (JavaScript)  
  - *Role:* Generates a well-structured HTML document from the AI output object and merged transcript.  
  - *Key Logic:*  
    - Escapes HTML entities for safety  
    - Formats meeting metadata into headings and paragraphs  
    - Converts multiline text into unordered lists  
    - Splits transcript into chunks per speaker and groups every 3 sentences  
    - Includes full transcript below summary sections  
    - Prepares binary data buffer for uploading with MIME type 'text/html'  
  - *Output:* JSON with filename and binary data property for file upload node.  
  - *Failures:* HTML injection if escaping fails, empty fields cause empty sections, encoding errors.  
  - *Version:* 2.

- **Upload File as HTML**  
  - *Type:* Google Drive Node  
  - *Role:* Uploads the generated HTML file to the user's Google Drive root folder or specified folder.  
  - *Configuration:* Uses OAuth2 credentials for Google Drive, allows folder selection.  
  - *Input:* Binary file data from Create as HTML node.  
  - *Output:* File metadata including file ID needed for next steps.  
  - *Failures:* Auth errors, quota exceeded, network issues.  
  - *Version:* 3.

- **Convert to Google Doc**  
  - *Type:* HTTP Request  
  - *Role:* Uses Google Drive API to copy the uploaded HTML file with a new MIME type set to Google Docs format, effectively converting the HTML file into a Google Doc.  
  - *Endpoint:* POST to `https://www.googleapis.com/drive/v3/files/{{fileId}}/copy`  
  - *Body:* Sets new file name (removes .html extension) and mimeType to `application/vnd.google-apps.document`.  
  - *Headers:* Content-Type application/json; OAuth2 credentials for Drive.  
  - *Failures:* Permission denied, invalid file ID, API limits.  
  - *Version:* 4.2.

- **Improve Page Layout**  
  - *Type:* HTTP Request  
  - *Role:* Calls Google Docs API to batch update document style, specifically setting uniform page margins (36pt = 0.5 inch) on all sides for better readability.  
  - *Endpoint:* POST to `https://docs.googleapis.com/v1/documents/{{docId}}:batchUpdate`  
  - *Body:* JSON with margin size requests.  
  - *Failures:* API quota, document locked, invalid document ID.  
  - *Version:* 4.2.

- **Delete HTML File**  
  - *Type:* Google Drive Node  
  - *Role:* Deletes the temporary HTML file after conversion to keep Drive storage clean.  
  - *Configuration:* Uses fileId from Upload File as HTML node.  
  - *Options:* Permanently delete file (no trash).  
  - *Failures:* File not found, permission issues.  
  - *Version:* 3.

- **Sticky Note2 (Convert to HTML Output and Upload to Google Drive)**  
  - *Type:* Sticky Note  
  - *Content:* Describes that this block creates HTML from meeting data and AI notes, uploads and converts to Google Doc, then deletes the HTML file.

---

### 3. Summary Table

| Node Name                | Node Type                              | Functional Role                                | Input Node(s)           | Output Node(s)           | Sticky Note                                                           |
|--------------------------|--------------------------------------|-----------------------------------------------|-------------------------|--------------------------|-----------------------------------------------------------------------|
| Get Fathom Meeting       | Webhook                              | Receive raw Fathom meeting data                | —                       | Format Key Parts          | Capture Meeting and Validate: Get meeting data, format and validate. |
| Format Key Parts         | Code                                 | Extract and normalize meeting & transcript data | Get Fathom Meeting      | Transcript Present?       | Capture Meeting and Validate: Format transcript and attendees.       |
| Transcript Present?      | If Condition                         | Validate transcript presence and length        | Format Key Parts         | AI Meeting Analysis       | Capture Meeting and Validate: Ensure meaningful transcript.          |
| Google Gemini Chat Model | LangChain Google Gemini AI           | AI language model for meeting analysis         | AI Meeting Analysis (via LangChain) | AI Meeting Analysis (LLM Chain) | Perform Meeting Analysis (AI): Use Google Gemini for summary.        |
| Structured Output Parser | LangChain Output Parser (Structured) | Parse AI output into structured JSON            | Google Gemini Chat Model | AI Meeting Analysis       | Perform Meeting Analysis (AI): Parse structured meeting output.      |
| AI Meeting Analysis      | LangChain Chain LLM                  | Prompt AI to analyze meeting and produce notes | Transcript Present?, Google Gemini Chat Model, Structured Output Parser | Set Fields                 | Perform Meeting Analysis (AI): Generate detailed meeting report.     |
| Set Fields               | Set                                  | Map AI output fields and merge with transcript | AI Meeting Analysis      | Create as HTML            | Perform Meeting Analysis (AI): Format output fields for HTML creation.|
| Create as HTML           | Code                                 | Generate formatted HTML document from data     | Set Fields               | Upload File as HTML       | Convert to HTML Output and Upload: Create HTML for Google Docs.      |
| Upload File as HTML      | Google Drive                         | Upload HTML file to Google Drive                | Create as HTML           | Convert to Google Doc     | Convert to HTML Output and Upload: Upload HTML to Drive.             |
| Convert to Google Doc    | HTTP Request                        | Convert uploaded HTML file to Google Docs format | Upload File as HTML      | Delete HTML File, Improve Page Layout | Convert to HTML Output and Upload: Convert HTML to native Doc.       |
| Improve Page Layout      | HTTP Request                        | Adjust Google Doc margins for readability       | Convert to Google Doc    | —                        | Convert to HTML Output and Upload: Improve document layout.          |
| Delete HTML File         | Google Drive                         | Delete temporary HTML file post-conversion      | Convert to Google Doc    | —                        | Convert to HTML Output and Upload: Cleanup temporary file.           |
| Sticky Note              | Sticky Note                         | Documentation                                   | —                       | —                        | Capture Meeting and Validate block description.                      |
| Sticky Note1             | Sticky Note                         | Documentation                                   | —                       | —                        | Perform Meeting Analysis (AI) block description.                     |
| Sticky Note2             | Sticky Note                         | Documentation                                   | —                       | —                        | Convert to HTML Output and Upload block description.                 |
| Sticky Note3             | Sticky Note                         | Documentation                                   | —                       | —                        | Full workflow overview, setup and customization notes.               |
| Sticky Note4             | Sticky Note                         | Documentation                                   | —                       | —                        | Sample Google Doc and Google Drive output links and screenshots.     |
| Sticky Note7             | Sticky Note                         | Documentation                                   | —                       | —                        | Quick troubleshooting tips for common issues.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Get Fathom Meeting"**  
   - Type: Webhook (HTTP POST)  
   - Configure unique webhook path (e.g., “2fab6c8f-ade4-49ba-b160-7cf6aa11cb15”)  
   - Enable raw body and binary property "data"  
   - No credentials needed  

2. **Create Code Node: "Format Key Parts"**  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that:  
     - Extracts meeting metadata (title, attendees, times, URLs)  
     - Parses and merges transcript lines by speaker, sorting by timestamp  
     - Outputs structured JSON with `meeting_title`, `attendees` array, `transcript_merged` array, and other fields  
   - Connect "Get Fathom Meeting" output to this node's input  

3. **Create If Node: "Transcript Present?"**  
   - Type: If Condition (Version 2)  
   - Condition 1: `transcript_merged` (stringified) is not empty  
   - Condition 2: Length of `transcript_merged` array >= 3  
   - Connect "Format Key Parts" output to this node's input  

4. **Create LangChain Google Gemini Chat Model Node: "Google Gemini Chat Model"**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Model: "models/gemini-2.5-pro"  
   - Credentials: Google Palm API key (OAuth2 or API key)  
   - No direct connection yet; will connect via LangChain chain node  

5. **Create LangChain Structured Output Parser Node: "Structured Output Parser"**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Paste the provided JSON schema example defining expected AI output structure  
   - No direct connection yet; will connect via LangChain chain node  

6. **Create LangChain Chain LLM Node: "AI Meeting Analysis"**  
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
   - Prompt: Paste the detailed prompt instructing the AI to analyze transcript and produce structured meeting notes, embedding meeting metadata and full transcript JSON string  
   - Configure to use "Google Gemini Chat Model" as language model node and "Structured Output Parser" as output parser node  
   - Connect "Transcript Present?" node’s true output to this node's input  

7. **Create Set Node: "Set Fields"**  
   - Type: Set  
   - Assign fields using expressions combining AI output properties and data from "Format Key Parts", e.g.:  
     - `"Meeting Title"` = AI output ‘Meeting Title’ + formatted scheduled date/time with colons replaced by dashes  
     - Pass other AI output fields like Recording URL, Executive Summary, Action Items, etc.  
     - Pass `transcript_merged` from "Format Key Parts" node  
   - Connect "AI Meeting Analysis" output to this node  

8. **Create Code Node: "Create as HTML"**  
   - Type: Code (JavaScript)  
   - Paste the provided code that:  
     - Converts meeting metadata and AI output fields into styled HTML with headings and lists  
     - Formats full transcript with timestamps grouped by speaker and every 3 sentences  
     - Escapes HTML entities  
     - Prepares binary data buffer with MIME type 'text/html' and a filename  
   - Connect "Set Fields" output to this node  

9. **Create Google Drive Node: "Upload File as HTML"**  
   - Type: Google Drive (Upload)  
   - Configure OAuth2 credentials for Google Drive  
   - Upload to desired folder (default is root)  
   - Use binary data from "Create as HTML" node  
   - Connect "Create as HTML" to this node  

10. **Create HTTP Request Node: "Convert to Google Doc"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://www.googleapis.com/drive/v3/files/{{ $json.id }}/copy` (use file ID from upload node output)  
    - Body (JSON): `{ "name": "{{ $json.name.replace('.html', '') }}", "mimeType": "application/vnd.google-apps.document" }`  
    - Headers: Content-Type: application/json  
    - Authentication: Use Google Drive OAuth2 credentials  
    - Connect "Upload File as HTML" to this node  

11. **Create HTTP Request Node: "Improve Page Layout"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://docs.googleapis.com/v1/documents/{{$node["Convert to Google Doc"].json.id}}:batchUpdate`  
    - Body (JSON):  
      ```json
      {
        "requests": [
          {
            "updateDocumentStyle": {
              "documentStyle": {
                "marginTop": {"magnitude": 36, "unit": "PT"},
                "marginBottom": {"magnitude": 36, "unit": "PT"},
                "marginLeft": {"magnitude": 36, "unit": "PT"},
                "marginRight": {"magnitude": 36, "unit": "PT"}
              },
              "fields": "marginTop,marginBottom,marginLeft,marginRight"
            }
          }
        ]
      }
      ```  
    - Authentication: Google Drive OAuth2  
    - Connect "Convert to Google Doc" to this node  

12. **Create Google Drive Node: "Delete HTML File"**  
    - Type: Google Drive (Delete File)  
    - File ID: Use the file ID from "Upload File as HTML" node  
    - Operation: Delete permanently  
    - Credentials: Google Drive OAuth2  
    - Connect "Convert to Google Doc" to this node  

13. **Connect Flow:**  
    - "Get Fathom Meeting" → "Format Key Parts" → "Transcript Present?" (true branch) → "AI Meeting Analysis" → "Set Fields" → "Create as HTML" → "Upload File as HTML" → "Convert to Google Doc" → parallel to "Delete HTML File" and "Improve Page Layout"

14. **Add Sticky Notes:**  
    - Add sticky notes at strategic points to describe each block (optional but recommended for clarity and maintenance).

15. **Credentials Setup:**  
    - Google Palm API credentials for Gemini Chat Model node  
    - Google Drive OAuth2 credentials for Drive upload, delete, and Docs API calls  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow works fully with the Fathom free account and uses Google Gemini API, which has a generous free tier and rate limits to minimize costs. Check the current pricing at [Google AI Pricing](https://ai.google.dev/pricing).                                                                                                                                                                                                                                                                                            | Workflow summary; Google AI Pricing link                                                                        |
| The workflow uses a webhook to immediately respond to Fathom meeting end events, preventing timeout or duplicate triggers. It validates that the transcript contains at least 3 conversation turns before proceeding to AI analysis to reduce costs and avoid empty results.                                                                                                                                                                                                                                                 | Workflow design note                                                                                            |
| Temporary HTML files are uploaded to Google Drive and then converted to Google Docs, after which the HTML files are deleted to keep storage clean. Page margins are adjusted via Google Docs API for improved readability.                                                                                                                                                                                                                                                                                                    | Document creation and cleanup best practices                                                                    |
| Sample output Google Doc meeting notes and Google Drive folder structure are linked and illustrated in sticky notes for reference.                                                                                                                                                                                                                                                                                                                                                                                        | Sample outputs and UI reference in sticky notes                                                                 |
| Quick troubleshooting tips include: ensuring transcript presence (3+ turns), verifying the HTTP POST to the Drive copy endpoint for conversion, re-authorizing Google credentials if 401/403 errors occur, and customizing AI prompts if meeting notes are inadequate.                                                                                                                                                                                                                                                  | Troubleshooting guidance included in sticky notes                                                               |
| The Google Gemini API key can be obtained free at [Google Makersuite API Keys](https://makersuite.google.com/app/apikey).                                                                                                                                                                                                                                                                                                                                                                                                 | Credential acquisition resource                                                                                  |
| The workflow is suitable for individuals or teams wanting automated, flexible AI meeting summaries and formatted documentation, ideal for syncs, client meetings, and interviews.                                                                                                                                                                                                                                                                                                                                          | Target audience and use cases                                                                                     |

---

**Disclaimer:** The provided text is exclusively from an automated n8n workflow. It fully complies with content policies and handles only legal and public data.