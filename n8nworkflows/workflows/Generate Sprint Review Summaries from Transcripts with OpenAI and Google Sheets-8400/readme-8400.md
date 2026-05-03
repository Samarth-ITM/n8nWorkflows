Generate Sprint Review Summaries from Transcripts with OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/generate-sprint-review-summaries-from-transcripts-with-openai-and-google-sheets-8400


# Generate Sprint Review Summaries from Transcripts with OpenAI and Google Sheets

---

## 1. Workflow Overview

This workflow automates the process of generating structured Sprint Review summaries from transcript files using OpenAI’s language model and archives the results in Google Sheets. It is designed for Agile teams conducting Sprint Reviews, enabling them to quickly obtain concise, well-formatted meeting summaries with executive bullets, presentation recaps, and actionable items.

The workflow is logically divided into these main functional blocks:

- **1.1 Input Reception:** Collects the Sprint Review transcript file, sprint name, and domain/team via a web form.
- **1.2 Transcript Parsing:** Processes the uploaded transcript file (supports VTT and custom timestamped formats) and normalizes its content into a consistent `[HH:MM:SS] Speaker: text` format.
- **1.3 AI Summary Generation:** Uses an OpenAI-based agent to generate a detailed Markdown summary including a concise introduction, executive bullets, presentation recap table, and action items checklist.
- **1.4 Summary Preview:** Displays the AI-generated summary in a styled UI form for review.
- **1.5 Archiving:** Saves the summary along with the transcript, metadata, and file info into a Google Sheets spreadsheet for record-keeping.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block collects user input via a web form, including the transcript file, sprint name, and domain selection.

**Nodes Involved:**  
- Collect Sprint Review Input

**Node Details:**  

- **Collect Sprint Review Input**  
  - Type: Form Trigger  
  - Role: Entry point; receives user input via a web form.  
  - Configuration:  
    - Form titled "Sprint Review Summary" with fields:  
      - Transcript file (single file upload)  
      - Sprint name (text input)  
      - Domain (dropdown with options: Team 1, Team 2, Team 3)  
    - Submit button labeled "Create Summary"  
  - Key expressions: None at this stage; raw inputs captured.  
  - Input Connections: None (trigger node).  
  - Output Connections: Connected to "Parse Transcript".  
  - Edge Cases:  
    - No file uploaded → downstream code node will error out if file missing.  
    - Invalid file type (non-text or non-VTT) could cause parsing issues.  
  - Version: 2.2  

---

### 2.2 Transcript Parsing

**Overview:**  
This block processes the uploaded transcript file, normalizing various transcript formats into a standard `[timestamp] Speaker: text` line-by-line string suitable for AI consumption.

**Nodes Involved:**  
- Parse Transcript

**Node Details:**  

- **Parse Transcript**  
  - Type: Code node (JavaScript)  
  - Role: Reads the uploaded transcript binary file, converts base64 to UTF-8 text, and parses it.  
  - Configuration Highlights:  
    - Detects file content type: if it starts with "WEBVTT", it parses as VTT subtitle format extracting timestamps and text blocks.  
    - Otherwise, parses custom lines of format `[Speaker] HH:MM:SS text`.  
    - Outputs a single string `transcriptWithTimestamps` with each line formatted as `[HH:MM:SS] Speaker: text`.  
  - Key Expressions: Uses buffer conversion, regex matching for timestamps and speakers.  
  - Input Connections: From "Collect Sprint Review Input" node, receives uploaded file and metadata.  
  - Output Connections: To "Generate Summary" node.  
  - Edge Cases / Failure Modes:  
    - Missing binary file → throws error "No binary data found".  
    - Unexpected transcript format → may produce incomplete parsing or skip lines.  
    - Large files could cause performance issues or timeouts.  
  - Version: 2  

---

### 2.3 AI Summary Generation

**Overview:**  
This block generates a rich Markdown summary of the Sprint Review transcript using OpenAI language models, structured with executive bullets, presentation recap, and action items.

**Nodes Involved:**  
- Generate Summary  
- OpenAI LLM  

**Node Details:**  

- **Generate Summary**  
  - Type: LangChain Agent (AI agent node)  
  - Role: Defines the prompt and calls the AI language model to produce the summary.  
  - Configuration:  
    - Prompt instructs the AI as an expert Agile coach to summarize the transcript for the given domain and sprint.  
    - Requires output strictly in Markdown with sections: header title, concise summary, executive summary bullets, presentation recap table (Timestamp | Presenter | Topics), and action items checklist with owners.  
    - Uses dynamic expressions to inject:  
      - Sprint name (`{{ $json['Sprint name'] }}`)  
      - Domain (`{{ $json.Domain }}`)  
      - Parsed transcript (`{{ $json.transcriptWithTimestamps }}`)  
  - Input Connections: From "Parse Transcript".  
  - Output Connections: To "Preview Summary" and "Save to Google Sheets".  
  - Edge Cases:  
    - API rate limits or authentication errors with OpenAI.  
    - Model may not strictly follow formatting instructions without fine-tuning.  
    - Very large transcripts may be truncated or cause token limits exceeded.  
  - Version: 2.1  

- **OpenAI LLM**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Backend language model node used by "Generate Summary" agent.  
  - Configuration:  
    - Model selected: `gpt-5-mini-2025-08-07` (an advanced GPT-5 variant)  
    - Uses OpenAI API credentials configured separately.  
  - Input Connections: Connected internally by "Generate Summary".  
  - Output Connections: Returns AI-generated summary back to "Generate Summary".  
  - Edge Cases:  
    - Credential expiration or invalid API key.  
    - Network timeouts or API errors.  
  - Version: 1.2  

---

### 2.4 Summary Preview

**Overview:**  
Displays the AI-generated Markdown summary in a styled form for user review before saving.

**Nodes Involved:**  
- Preview Summary

**Node Details:**  

- **Preview Summary**  
  - Type: Form node  
  - Role: Presents the summary output with custom CSS styling for readability.  
  - Configuration:  
    - Form styled with monospace font, responsive design, pre-wrapped text for Markdown formatting.  
    - Completion title "Summary" and message displaying AI output (`={{ $json.output }}`).  
    - No user input expected; purely for display.  
  - Input Connections: From "Generate Summary".  
  - Output Connections: None (terminal node for UI).  
  - Edge Cases:  
    - If AI output is empty or malformed, display may be broken.  
    - Long summaries may need scroll or pagination (not implemented).  
  - Version: 1  

---

### 2.5 Archiving to Google Sheets

**Overview:**  
Appends or updates a Google Sheets document with the summary, transcript, metadata, and file details for archival.

**Nodes Involved:**  
- Save to Google Sheets

**Node Details:**  

- **Save to Google Sheets**  
  - Type: Google Sheets node  
  - Role: Saves processed data into a Google Sheet for traceability and historical record.  
  - Configuration:  
    - Uses OAuth2 credentials for Google Sheets API.  
    - Document ID and sheet defined explicitly.  
    - Columns mapped: Date (taken from `submittedAt`), Domain, Sprint name, Content (AI summary), VTT file name, Transcript (parsed), etc.  
    - Operation: Append or update by matching on Date column.  
  - Input Connections: From "Generate Summary".  
  - Output Connections: None (workflow end).  
  - Edge Cases:  
    - Invalid or expired Google OAuth credentials.  
    - Google API quota exceeded or network errors.  
    - Duplicate entries if date matching fails.  
  - Version: 4.6  

---

## 3. Summary Table

| Node Name               | Node Type                      | Functional Role                      | Input Node(s)             | Output Node(s)                      | Sticky Note                                                                                   |
|-------------------------|--------------------------------|------------------------------------|---------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------|
| Collect Sprint Review Input | Form Trigger                   | Input reception and form data capture | None                      | Parse Transcript                  | ## Input Form Collects transcript file + sprint + domain.                                    |
| Parse Transcript         | Code                           | Transcript file parsing and normalization | Collect Sprint Review Input | Generate Summary                  | ## Transcript Parser Normalizes to `[HH:MM:SS] Speaker: text`. Supports VTT and custom lines. |
| Generate Summary         | LangChain Agent                | AI summary generation using OpenAI | Parse Transcript           | Preview Summary, Save to Google Sheets | ## AI Summary Creates executive bullets, recap table, and action items in Markdown.          |
| OpenAI LLM              | LangChain OpenAI Chat Model     | Backend LLM for summary generation | Connected internally by Generate Summary | Generate Summary (returns output) | ## LLM Backend OpenAI Chat Model used by AI Agent.                                           |
| Preview Summary         | Form                           | UI preview of AI-generated Markdown summary | Generate Summary           | None                              | ## Preview Shows the generated Markdown with custom CSS.                                     |
| Save to Google Sheets    | Google Sheets                  | Archive summary and metadata to sheet | Generate Summary           | None                              | ## Archive to Sheets Saves summary + transcript + metadata (date/domain/sprint/file).         |
| Sticky: Overview1       | Sticky note                    | Workflow high-level overview       | None                      | None                              | ## Overview Sprint Review Transcript → AI Markdown Summary → Google Sheets Inputs: File (VTT/text), Sprint name, Domain. Outputs: Preview in UI + archived row in Sheet. |
| Sticky: Input Form1     | Sticky note                    | Comment on input form               | None                      | None                              | ## Input Form Collects transcript file + sprint + domain.                                    |
| Sticky: Parser1         | Sticky note                    | Comment on transcript parser       | None                      | None                              | ## Transcript Parser Normalizes to `[HH:MM:SS] Speaker: text`. Supports VTT and simple speaker/timestamp lines. |
| Sticky: AI Summary1     | Sticky note                    | Comment on AI summary generation   | None                      | None                              | ## AI Summary Creates executive bullets, recap table, and action items in Markdown.          |
| Sticky: Preview1        | Sticky note                    | Comment on preview form             | None                      | None                              | ## Preview Shows the generated Markdown with custom CSS.                                     |
| Sticky: Sheets1         | Sticky note                    | Comment on Google Sheets archiving | None                      | None                              | ## Archive to Sheets Saves summary + transcript + metadata (date/domain/sprint/file).         |
| Sticky: LLM1            | Sticky note                    | Comment on language model backend   | None                      | None                              | ## LLM Backend OpenAI Chat Model used by AI Agent.                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Collect Sprint Review Input"**  
   - Node Type: Form Trigger  
   - Configure form:  
     - Title: "Sprint Review Summary"  
     - Fields:  
       - File Upload (single file) labeled "Transcript file"  
       - Text input labeled "Sprint name"  
       - Dropdown labeled "Domain" with options: Team 1, Team 2, Team 3  
     - Button label: "Create Summary"  
   - Save and activate webhook.

2. **Create Code Node: "Parse Transcript"**  
   - Node Type: Code (JavaScript)  
   - Paste the provided script that:  
     - Extracts uploaded file from binary data  
     - Converts base64 to UTF-8 text  
     - Parses either WEBVTT or custom timestamped lines into normalized `[HH:MM:SS] Speaker: text` strings  
     - Outputs `transcriptWithTimestamps` as a joined string in JSON output  
   - Connect input from "Collect Sprint Review Input".

3. **Create LangChain Agent Node: "Generate Summary"**  
   - Node Type: LangChain Agent  
   - Prompt: Use the detailed prompt text instructing the AI to generate a Markdown summary with specified sections (concise summary, executive bullets, recap table, action items), injecting:  
     - Sprint name from input JSON  
     - Domain from input JSON  
     - Parsed transcript string  
   - No additional options needed.  
   - Connect input from "Parse Transcript".

4. **Create LangChain OpenAI Chat Model Node: "OpenAI LLM"**  
   - Node Type: LangChain OpenAI Chat Model  
   - Model: Select GPT-5 Mini or equivalent advanced GPT model  
   - Provide OpenAI API credentials (OAuth or API key) in node credentials.  
   - Connect as language model backend for "Generate Summary" node.

5. **Create Form Node: "Preview Summary"**  
   - Node Type: Form  
   - Configure to display output text from AI:  
     - Completion title: "Summary"  
     - Completion message expression: `={{ $json.output }}`  
     - Add custom CSS for monospace font, pre-wrapped text, responsive design (copy CSS from workflow).  
   - Connect input from "Generate Summary".

6. **Create Google Sheets Node: "Save to Google Sheets"**  
   - Node Type: Google Sheets  
   - Operation: Append or Update Rows  
   - Configure credentials with Google Sheets OAuth2 (ensure access to target spreadsheet).  
   - Set Document ID to target spreadsheet ID.  
   - Set Sheet name (gid=0 or sheet name).  
   - Map columns: Date (from transcript submission time), Domain, Sprint name, Content (AI summary), VTT file name, Transcript (parsed), etc.  
   - Connect input from "Generate Summary".

7. **Connect Nodes Sequentially:**  
   - "Collect Sprint Review Input" → "Parse Transcript" → "Generate Summary"  
   - "Generate Summary" → "Preview Summary"  
   - "Generate Summary" → "Save to Google Sheets"

8. **Test Workflow:**  
   - Deploy and activate webhook for form trigger.  
   - Submit test transcript file and metadata via form.  
   - Verify parsing, AI summary generation, preview display, and Google Sheets archival.

---

## 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow input form enables user-friendly upload and selection of sprint metadata.                                    | Input Form Node: "Collect Sprint Review Input".                                                  |
| Transcript parser supports common VTT format and a simple custom speaker/timestamp line format for flexibility.       | Code Node "Parse Transcript".                                                                    |
| AI prompt instructs strict Markdown output including structured sections for ease of reading and further processing.  | LangChain Agent Node "Generate Summary".                                                        |
| Preview form uses custom CSS for monospace font and responsive layout to display Markdown clearly in UI.              | Preview Summary Node styling.                                                                    |
| Google Sheets archival includes metadata and full transcript to allow audit and future reference.                     | Google Sheets Node "Save to Google Sheets".                                                     |
| OpenAI credentials must be configured with valid API key and sufficient quota for the GPT-5 Mini model.               | Credentials: OpenAI API.                                                                         |
| Google Sheets OAuth2 credentials require access to the chosen spreadsheet with appropriate permissions.               | Credentials: Google Sheets OAuth2 API.                                                          |

---

disclaimer The text provided derives exclusively from an automated workflow built with n8n, a tool for integration and automation. The processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.