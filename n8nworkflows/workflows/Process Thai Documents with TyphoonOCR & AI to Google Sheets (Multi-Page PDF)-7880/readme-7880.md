Process Thai Documents with TyphoonOCR & AI to Google Sheets (Multi-Page PDF)

https://n8nworkflows.xyz/workflows/process-thai-documents-with-typhoonocr---ai-to-google-sheets--multi-page-pdf--7880


# Process Thai Documents with TyphoonOCR & AI to Google Sheets (Multi-Page PDF)

### 1. Workflow Overview

This workflow automates the processing of multiple Thai-language PDF documents containing official or bureaucratic text. It is designed to handle multi-file and multi-page PDFs stored in a designated folder, extracting structured data and saving it to Google Sheets. The process involves Optical Character Recognition (OCR) via Typhoon OCR, AI-powered text structuring through a Large Language Model (LLM), and subsequent data formatting to fit spreadsheet columns.

Logical blocks of the workflow:

- **1.1 Input Reception**  
  Manual trigger to start the workflow and loading multi-page PDF files from a specified folder.

- **1.2 File Preparation and Splitting**  
  Writing input files to disk, calculating page counts, and splitting each PDF into individual pages.

- **1.3 OCR Extraction**  
  Running Typhoon OCR on each PDF page to extract raw text from images.

- **1.4 Text Aggregation and AI Structuring**  
  Aggregating page-wise OCR texts into a single document, then using an LLM to convert raw OCR text into structured JSON data.

- **1.5 JSON Parsing and Formatting**  
  Parsing the AI-generated JSON and formatting it into rows and columns suitable for Google Sheets.

- **1.6 Data Saving and Cleanup**  
  Appending the processed data to a Google Sheet, cleaning temporary files, and moving processed PDFs to a completed folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and loads all multi-page PDF files from the specified input directory.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Load PDFs from doc Folder (File Reader)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Config: No parameters.  
    - Connections: Outputs to Load PDFs from doc Folder.  
    - Edge Cases: None expected; user must manually trigger.

  - **Load PDFs from doc Folder**  
    - Type: Read/Write File (Read operation)  
    - Role: Reads all PDF files matching `/doc/multipage/*` path.  
    - Config: File selector set to `/doc/multipage/*`, always outputs data.  
    - Connections: Outputs to Loop Over Items.  
    - Edge Cases: Empty folder returns no items; file permission errors possible.

---

#### 2.2 File Preparation and Splitting

- **Overview:**  
  Each PDF file is processed one at a time, written temporarily to disk, and split into individual pages using system commands.

- **Nodes Involved:**  
  - Loop Over Items (Batch processor)  
  - Read/Write Files from Disk (Write operation)  
  - Set_Input_Path (Code)  
  - Split PDF page (Execute Command)  
  - Read Splite PDF Page (Read operation)

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes each PDF file individually.  
    - Config: Default batch size 1 (process one file at a time).  
    - Connections: Outputs to Read/Write Files from Disk and secondary output to end loop.  
    - Edge Cases: Empty input stops workflow; batch misconfiguration may cause concurrency issues.

  - **Read/Write Files from Disk**  
    - Type: Read/Write File (Write operation)  
    - Role: Writes the current PDF file to `/doc/tmp/in.pdf` for processing.  
    - Config: Operation set to "write", filename fixed to `/doc/tmp/in.pdf`.  
    - Input: File binary or path from Loop Over Items.  
    - Connections: Outputs to Set_Input_Path.  
    - Edge Cases: Disk space issues; file lock conflicts.

  - **Set_Input_Path**  
    - Type: Code (JavaScript)  
    - Role: Calculates and sets an `inputPath` property for the PDF file, using existing JSON fields or composing from directory and filename.  
    - Key Code: Uses filePath if available; else constructs from directory + fileName.  
    - Connections: Outputs to Split PDF page.  
    - Edge Cases: Missing fields may result in malformed paths.

  - **Split PDF page**  
    - Type: Execute Command  
    - Role: Uses shell commands (`pdfinfo` and `pdfseparate`) to count pages and split the PDF into separate page files stored in `/doc/tmp/pages`.  
    - Key Commands:  
      - Counts pages with `pdfinfo`.  
      - Splits with `pdfseparate`.  
      - Lists output files.  
    - Connections: Outputs to Read Splite PDF Page.  
    - Edge Cases: Missing `pdfinfo` or `pdfseparate` utilities; unreadable PDF file; zero pages detected.

  - **Read Splite PDF Page**  
    - Type: Read/Write File (Read operation)  
    - Role: Reads all the individual PDF page files from `/doc/tmp/pages/*.pdf` for OCR processing.  
    - Connections: Outputs to Extract Text with Typhoon OCR.  
    - Edge Cases: No pages found; read permission errors.

---

#### 2.3 OCR Extraction

- **Overview:**  
  Runs Typhoon OCR on each individual PDF page file to extract raw text content.

- **Nodes Involved:**  
  - Extract Text with Typhoon OCR (Execute Command)  
  - Aggregate (Aggregate)

- **Node Details:**

  - **Extract Text with Typhoon OCR**  
    - Type: Execute Command  
    - Role: Executes Python code that calls Typhoon OCR API to extract text from the given page PDF file.  
    - Command: Inline Python script setting `TYPHOON_OCR_API_KEY` environment variable (placeholder to be updated by user).  
    - Input: Path to PDF page file (from JSON field `fileName`).  
    - Output: OCR text as standard output.  
    - Connections: Outputs to Aggregate.  
    - Edge Cases: API key missing or invalid; network issues; empty OCR result; Python environment issues.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates OCR text (`stdout` field) from all pages of the current file into one combined output.  
    - Config: Aggregates the `stdout` field across all items.  
    - Connections: Outputs combined text to Structure Text to JSON with LLM.  
    - Edge Cases: No OCR text to aggregate; partial failures.

---

#### 2.4 Text Aggregation and AI Structuring

- **Overview:**  
  Sends the aggregated OCR raw text to an AI language model to parse and extract structured data in JSON format.

- **Nodes Involved:**  
  - Structure Text to JSON with LLM (Langchain Chain LLM)  
  - OpenRouter Chat Model (LM Chat Node)

- **Node Details:**

  - **Structure Text to JSON with LLM**  
    - Type: Langchain Chain LLM  
    - Role: Takes OCR raw text and applies a prompt to extract key document fields as JSON.  
    - Prompt: Defines expected JSON keys with Thai labels and instructions, embedding the OCR text dynamically.  
    - Connections: Sends prompt output to OpenRouter Chat Model.  
    - Edge Cases: LLM API limits; poor OCR quality; prompt failures.

  - **OpenRouter Chat Model**  
    - Type: Langchain LM Chat (OpenRouter)  
    - Role: Uses OpenRouter API with GPT-4o-mini model to generate AI response for text structuring.  
    - Credentials: Uses OpenRouter API key stored in credentials.  
    - Connections: Outputs AI response back to Structure Text to JSON with LLM for parsing.  
    - Edge Cases: API key invalid or expired; network timeouts; AI model errors.

---

#### 2.5 JSON Parsing and Formatting

- **Overview:**  
  Parses the AI-generated JSON text, cleans formatting artifacts, and prepares data fields for Google Sheets insertion.

- **Nodes Involved:**  
  - Parse JSON to Sheet Format (Code)  
  - Save to Google Sheet (Google Sheets)

- **Node Details:**

  - **Parse JSON to Sheet Format**  
    - Type: Code (JavaScript)  
    - Role: Cleans raw LLM output by removing markdown code block markers, parses JSON, extracts fields including nested contact info, and returns a flattened JSON object.  
    - Key Expressions:  
      - Removes ```json and ``` markdown.  
      - Uses try-catch for JSON.parse with error throwing on failure.  
      - Extracts contact subfields from nested object.  
    - Connections: Outputs parsed JSON to Save to Google Sheet.  
    - Edge Cases: Malformed JSON from AI; missing expected fields; parsing errors.

  - **Save to Google Sheet**  
    - Type: Google Sheets  
    - Role: Appends a new row per processed PDF file with structured data fields mapped to columns.  
    - Config:  
      - Spreadsheet ID: `1h70cJyLj5i2j0Ag5kqp93ccZjjhJnqpLmz-ee5r4brU`  
      - Sheet GID: `0`  
      - Mapping defined for fields including book_id, date, subject, to, attach, detail, signed_by, signed_by2, contact_phone, contact_email, contact_fax, download_url.  
      - Matching column: book_id (for duplicate detection).  
    - Credentials: Google Sheets OAuth2 API credentials required.  
    - Connections: Outputs to Clear tmp files.  
    - Edge Cases: Google API rate limits; authentication errors; schema mismatches.

---

#### 2.6 Data Saving and Cleanup

- **Overview:**  
  Cleans up temporary files used during processing and moves processed input PDFs to a "Completed" folder.

- **Nodes Involved:**  
  - Clear tmp files (Execute Command)  
  - Loop Over Items (Cycle continuation)

- **Node Details:**

  - **Clear tmp files**  
    - Type: Execute Command  
    - Role: Executes shell script to delete `/doc/tmp/pages` temporary folder and move original PDFs from `/doc/multipage/` to `/doc/multipage/Completed/`.  
    - Script Details:  
      - Removes temp pages directory.  
      - Creates Completed folder if missing.  
      - Moves processed file using filename from original Load PDFs node.  
    - Connections: Outputs to Loop Over Items to process next file or end.  
    - Edge Cases: File permission errors; missing files; incomplete cleanup.

  - **Loop Over Items** (second output)  
    - Controls the batch processing loop, triggering writing of next file or ending the workflow.

---

### 3. Summary Table

| Node Name                      | Node Type                     | Functional Role                         | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                                                 |
|--------------------------------|-------------------------------|---------------------------------------|----------------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                | Workflow start trigger                 | –                                | Load PDFs from doc Folder           |                                                                                                                             |
| Load PDFs from doc Folder       | Read/Write File (Read)        | Loads all PDFs from input folder      | When clicking ‘Test workflow’    | Loop Over Items                    |                                                                                                                             |
| Loop Over Items                | SplitInBatches                | Processes each PDF file individually  | Load PDFs from doc Folder         | Read/Write Files from Disk, secondary output to end loop |                                                                                                                             |
| Read/Write Files from Disk      | Read/Write File (Write)       | Writes PDF file temporarily to disk   | Loop Over Items                  | Set_Input_Path                     |                                                                                                                             |
| Set_Input_Path                | Code                         | Generates inputPath for PDF file      | Read/Write Files from Disk        | Split PDF page                    |                                                                                                                             |
| Split PDF page                 | Execute Command               | Splits PDF into single pages          | Set_Input_Path                   | Read Splite PDF Page              |                                                                                                                             |
| Read Splite PDF Page           | Read/Write File (Read)        | Reads individual PDF page files       | Split PDF page                  | Extract Text with Typhoon OCR      |                                                                                                                             |
| Extract Text with Typhoon OCR   | Execute Command               | Runs OCR on each page using Typhoon OCR | Read Splite PDF Page           | Aggregate                        |                                                                                                                             |
| Aggregate                     | Aggregate                    | Combines OCR text from all pages      | Extract Text with Typhoon OCR     | Structure Text to JSON with LLM    |                                                                                                                             |
| Structure Text to JSON with LLM | Langchain Chain LLM           | Uses AI to parse text to JSON         | Aggregate                       | OpenRouter Chat Model             |                                                                                                                             |
| OpenRouter Chat Model           | Langchain LM Chat             | Calls OpenRouter GPT-4o-mini model    | Structure Text to JSON with LLM  | Structure Text to JSON with LLM (ai_languageModel output) |                                                                                                                             |
| Parse JSON to Sheet Format      | Code                         | Cleans and parses JSON for Sheets     | Structure Text to JSON with LLM  | Save to Google Sheet              |                                                                                                                             |
| Save to Google Sheet            | Google Sheets                 | Saves structured data to spreadsheet  | Parse JSON to Sheet Format        | Clear tmp files                  |                                                                                                                             |
| Clear tmp files                | Execute Command               | Cleans temp files and archives PDFs   | Save to Google Sheet             | Loop Over Items (to continue)     |                                                                                                                             |
| Sticky Note                   | Sticky Note                  | Documentation and workflow description | –                                | –                                | ## Thai OCR to Sheet (V2.2 – Multi-File & Multi-Page with AI)\n\nWorkflow นี้ออกแบบมาเพื่อประมวลผลเอกสาร PDF หลายไฟล์ที่อยู่ในโฟลเดอร์เดียวกัน... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters. This node starts the workflow manually.

2. **Create Read/Write File node to load PDFs**  
   - Name: `Load PDFs from doc Folder`  
   - Operation: Read  
   - File selector: `/doc/multipage/*`  
   - Always output data enabled.  
   - Connect output of Manual Trigger to this node.

3. **Create SplitInBatches node**  
   - Name: `Loop Over Items`  
   - Default batch size (1) to process files one at a time.  
   - Connect output of `Load PDFs from doc Folder` to this node.

4. **Create Read/Write File node to write current PDF**  
   - Name: `Read/Write Files from Disk`  
   - Operation: Write  
   - File name: `/doc/tmp/in.pdf` (fixed path)  
   - Connect first output of `Loop Over Items` to this node.

5. **Create Code node to set input path**  
   - Name: `Set_Input_Path`  
   - Language: JavaScript  
   - Code:  
     ```js
     return items.map(({ json, binary }) => {
       const filePath =
         json.filePath
           ? json.filePath
           : (json.directory
               ? `${json.directory.replace(/\/$/, '')}/${json.fileName}`
               : json.fileName);

       return {
         json: {
           ...json,
           inputPath: filePath,
         },
         binary,
       };
     });
     ```  
   - Connect output of `Read/Write Files from Disk` to this node.

6. **Create Execute Command node to split PDF pages**  
   - Name: `Split PDF page`  
   - Command:  
     ```sh
     sh -lc '
     set -e
     IN="{{ $json.inputPath }}"
     OUT="/doc/tmp/pages"
     rm -rf "$OUT" && mkdir -p "$OUT"

     PAGES=$(pdfinfo "$IN" 2>/dev/null | awk -F": *" "/^Pages/{print \$2}")
     [ -n "$PAGES" ] || { echo "Cannot detect page count for: $IN" >&2; exit 1; }

     pdfseparate -f 1 -l "$PAGES" "$IN" "$OUT/page_%d.pdf"
     ls -1 "$OUT"/page_*.pdf
     '
     ```  
   - Connect output of `Set_Input_Path` to this node.

7. **Create Read/Write File node to read split pages**  
   - Name: `Read Splite PDF Page`  
   - Operation: Read  
   - File selector: `/doc/tmp/pages/*.pdf`  
   - Connect output of `Split PDF page` to this node.

8. **Create Execute Command node for Typhoon OCR**  
   - Name: `Extract Text with Typhoon OCR`  
   - Command:  
     ```python
     =python -c "import sys, os; os.environ['TYPHOON_OCR_API_KEY'] = '<Please update your Typhoon OCR>'; from typhoon_ocr import ocr_document; sys.stdout.reconfigure(encoding='utf-8'); input_path = sys.argv[1]; text = ocr_document(input_path); print(text)" "/doc/tmp/pages/{{$json[\"fileName\"]}}"
     ```  
   - Replace `<Please update your Typhoon OCR>` with your valid Typhoon OCR API key.  
   - Connect output of `Read Splite PDF Page` to this node.

9. **Create Aggregate node**  
   - Name: `Aggregate`  
   - Aggregate field: `stdout` (the OCR text output)  
   - Connect output of `Extract Text with Typhoon OCR` to this node.

10. **Create Langchain Chain LLM node**  
    - Name: `Structure Text to JSON with LLM`  
    - Prompt Type: Define  
    - Text:  
      ```
      =ข้อความด้านล่างนี้เป็นเนื้อหา OCR จากหนังสือราชการ กรุณาแยกหัวข้อสำคัญออกมาในรูปแบบ JSON:

      1. book_id: เลขที่หนังสือ
      2. date: วันที่ในเอกสาร
      3. subject: หัวเรื่อง
      4. to: เรียน
      5. attach: สิ่งที่ส่งมาด้วย
      6. detail: เนื้อความในหนังสือ
      7. signed_by: ผู้ลงนาม
      8. signed_by2: ตำแหน่งผู้ลงนาม
      9. contact: ช่องทางติดต่อ (เช่น เบอร์โทร อีเมล)
      10. download_url: ลิงก์สำหรับดาวน์โหลด (ถ้ามี)

      OCR_TEXT:
      """
      {{ $json["stdout"] }}
      """
      ```
    - Connect output of `Aggregate` to this node.

11. **Create Langchain LM Chat node (OpenRouter)**  
    - Name: `OpenRouter Chat Model`  
    - Model: `openai/gpt-4o-mini`  
    - Credentials: Configure OpenRouter API credentials.  
    - Connect AI languageModel output of `Structure Text to JSON with LLM` to this node.

12. **Connect AI output of OpenRouter Chat Model back to Structure Text to JSON with LLM**  
    - This completes the AI generation cycle.

13. **Create Code node to parse JSON and prepare for Sheets**  
    - Name: `Parse JSON to Sheet Format`  
    - Run once for each item.  
    - Code:  
      ```js
      const raw = $json["text"];

      // Remove ```json and ``` markers
      const cleaned = raw.replace(/```json\n?|```/g, "").trim();

      let parsed;
      try {
        parsed = JSON.parse(cleaned);
      } catch (err) {
        throw new Error("JSON parsing failed: " + err.message + "\n\nRaw text:\n" + cleaned);
      }

      const contact = parsed.contact || {};

      return {
        book_id: parsed.book_id || "",
        date: parsed.date || "",
        subject: parsed.subject || "",
        to: parsed.to || "",
        attach: parsed.attach || "",
        detail: parsed.detail || "",
        signed_by: parsed.signed_by || "",
        signed_by2: parsed.signed_by2 || "",
        contact_phone: contact.phone || "",
        contact_email: contact.email || "",
        contact_fax: contact.fax || "",
        download_url: parsed.download_url || ""
      };
      ```
    - Connect output of `Structure Text to JSON with LLM` (main output) to this node.

14. **Create Google Sheets node to append data**  
    - Name: `Save to Google Sheet`  
    - Operation: Append  
    - Spreadsheet ID: `1h70cJyLj5i2j0Ag5kqp93ccZjjhJnqpLmz-ee5r4brU`  
    - Sheet: GID `0` (Sheet1)  
    - Mapping: Map the parsed JSON fields to columns as per the schema (book_id, date, subject, to, attach, detail, signed_by, signed_by2, contact_phone, contact_email, contact_fax, download_url)  
    - Matching column: `book_id` (to avoid duplicates)  
    - Credentials: Google Sheets OAuth2 credentials required.  
    - Connect output of `Parse JSON to Sheet Format` to this node.

15. **Create Execute Command node for cleanup**  
    - Name: `Clear tmp files`  
    - Command:  
      ```sh
      sh -lc '
      set -e

      # Remove temp pages folder
      rm -rf /doc/tmp/pages

      # Create Completed folder if missing
      mkdir -p /doc/multipage/Completed

      # Move original PDF to Completed
      src="/doc/multipage/{{ $('Load PDFs from doc Folder').item.json.fileName }}"
      dst="/doc/multipage/Completed/{{ $('Load PDFs from doc Folder').item.json.fileName }}"

      mv "$src" "$dst"
      echo "Moved $src → $dst"
      '
      ```
    - Connect output of `Save to Google Sheet` to this node.

16. **Connect output of `Clear tmp files` to Loop Over Items second output**  
    - This restarts the batch loop for the next PDF file until all are processed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow processes multi-file, multi-page PDFs with Thai OCR and AI to structured Google Sheets rows.                        | Description in Sticky Note node inside workflow.                                                 |
| AI prompt instructs extracting specific fields from official Thai documents OCR text.                                        | See `Structure Text to JSON with LLM` node prompt.                                              |
| Typhoon OCR requires user to replace placeholder API key with a valid key (`TYPHOON_OCR_API_KEY`).                           | Update in `Extract Text with Typhoon OCR` node command.                                         |
| Requires poppler-utils (pdfinfo, pdfseparate) installed on host system for PDF splitting.                                    | `Split PDF page` node shell commands.                                                           |
| Google Sheets node requires OAuth2 credentials with write access to target spreadsheet.                                      | Spreadsheet ID and Sheet GID hardcoded, ensure user has access.                                 |
| OpenRouter API key required for calling GPT-4o-mini model for text structuring.                                              | Credential setup needed for `OpenRouter Chat Model` node.                                       |
| Workflow designed for batch processing, one file at a time, with cleanup to avoid disk clutter.                              | Temporary files in `/doc/tmp/pages` deleted after each file.                                    |
| The final result is one row per PDF file in Google Sheets, with parsed fields as columns.                                    | Suitable for archival and data extraction from government or official documents in Thai.        |

---

*Disclaimer: The provided content is extracted exclusively from an n8n workflow automation. It complies strictly with content policies and contains no illegal, offensive, or protected materials. All processed data are legal and public.*