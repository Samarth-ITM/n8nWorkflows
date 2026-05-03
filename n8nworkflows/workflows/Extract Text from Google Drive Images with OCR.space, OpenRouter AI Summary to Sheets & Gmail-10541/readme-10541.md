Extract Text from Google Drive Images with OCR.space, OpenRouter AI Summary to Sheets & Gmail

https://n8nworkflows.xyz/workflows/extract-text-from-google-drive-images-with-ocr-space--openrouter-ai-summary-to-sheets---gmail-10541


# Extract Text from Google Drive Images with OCR.space, OpenRouter AI Summary to Sheets & Gmail

### 1. Workflow Overview

This workflow automates the extraction, summarization, and logging of text from newly uploaded image files in a specified Google Drive folder. It leverages OCR.space to perform optical character recognition (OCR) on images, OpenRouter AI to generate concise Japanese summaries, Google Sheets to log results, and Gmail to notify users upon completion.

The workflow is logically divided into the following blocks:

- **1.1 Image Detection and Download**: Monitors a designated Google Drive folder for new image files and downloads them.
- **1.2 OCR Text Extraction and Cleaning**: Sends the downloaded image to OCR.space for text extraction, then cleans and formats the extracted text.
- **1.3 AI Summarization**: Uses OpenRouter AI to summarize the cleaned OCR text in Japanese.
- **1.4 Logging to Google Sheets**: Appends a new row into a Google Sheets document with the filename, summary, and processing date.
- **1.5 Completion Notification via Gmail**: Sends an email notification including the OCR summary and links to Google Docs/Sheets, marking the process as complete.

---

### 2. Block-by-Block Analysis

#### 1.1 Image Detection and Download

- **Overview:**  
  Detects newly created image files in a specific Google Drive folder and downloads them for further processing.

- **Nodes Involved:**  
  - Google Drive New File Trigger1  
  - Download Image File1  
  - Sticky Note1 (provides descriptive context)

- **Node Details:**

  - **Google Drive New File Trigger1**  
    - *Type & Role:* Trigger node that monitors Google Drive for new files.  
    - *Configuration:* Watches a specific folder (ID: `1jTs0NNXMFTcChRq0C1KEoARVdIOOjQ1r`) and triggers on file creation events every minute.  
    - *Connections:* Output → Download Image File1  
    - *Potential Failures:* API rate limits, permission errors if folder is not accessible, network issues.

  - **Download Image File1**  
    - *Type & Role:* Downloads the detected image file binary data from Google Drive.  
    - *Configuration:* Uses file ID from trigger; downloads binary content under property `data`.  
    - *Connections:* Output → Extract Text with OCR.space1  
    - *Potential Failures:* File not found, permission denied, download timeout or corrupted binary data.

  - **Sticky Note1**  
    - *Content:* Describes this block as responsible for detecting and downloading new images.  
    - *No technical configuration.*

---

#### 1.2 OCR Text Extraction and Cleaning

- **Overview:**  
  Sends the downloaded image to the OCR.space API to extract Japanese text, then formats the raw OCR output by removing line breaks and excess spaces.

- **Nodes Involved:**  
  - Extract Text with OCR.space1  
  - Format OCR Result & Check for Empty1  
  - Sticky Note2

- **Node Details:**

  - **Extract Text with OCR.space1**  
    - *Type & Role:* HTTP Request node calling OCR.space API.  
    - *Configuration:*  
      - POST request to `https://api.ocr.space/parse/image`  
      - Multipart form-data with parameters:  
        - `apikey`: User’s OCR.space API key (masked in JSON)  
        - `language`: set to Japanese (`jpn`)  
        - `isOverlayRequired`: `false` (no overlay data)  
        - `file`: binary content passed from Download Image File1  
    - *Input:* Binary file data (`data`)  
    - *Output:* JSON response containing OCR results under `ParsedResults[0].ParsedText`  
    - *Potential Failures:* API key errors, rate limits, OCR service availability, malformed request, unsupported file types.

  - **Format OCR Result & Check for Empty1**  
    - *Type & Role:* Code node that cleans the OCR text output.  
    - *Configuration:* JavaScript code:  
      - Extracts raw OCR text or defaults to empty string.  
      - Removes carriage returns, line breaks, and collapses multiple spaces into one.  
      - Trims leading/trailing spaces.  
      - Extracts filename from the downloaded file node.  
      - Outputs JSON with `extractedText`, `fileName`, and boolean `hasText` indicating text presence.  
    - *Input:* JSON from OCR.space node, binary metadata from Download Image File1  
    - *Output:* Cleaned text and metadata for next step  
    - *Potential Failures:* Null or unexpected input format, missing file name, empty OCR results.

  - **Sticky Note2**  
    - *Content:* Explains this block extracts Japanese text and cleans formatting for readability.

---

#### 1.3 AI Summarization

- **Overview:**  
  Uses OpenRouter AI to generate a concise Japanese summary of the OCR-extracted text or a default message if no text was found.

- **Nodes Involved:**  
  - OpenRouter Chat Model1 (language model node linked to OpenRouter AI)  
  - Generate Summary with OpenRouter AI1 (agent node)  
  - Sticky Note3

- **Node Details:**

  - **OpenRouter Chat Model1**  
    - *Type & Role:* Language model node providing AI capabilities via OpenRouter API.  
    - *Configuration:* Uses connected OpenRouter API credentials. No custom options set.  
    - *Output:* AI-generated text response.  
    - *Connections:* Output → Generate Summary with OpenRouter AI1 (as language model input)  
    - *Potential Failures:* Authentication issues, API limits, service downtime.

  - **Generate Summary with OpenRouter AI1**  
    - *Type & Role:* Langchain agent node that sends prompt to OpenRouter Chat Model1.  
    - *Configuration:*  
      - Prompt text: Asks AI to summarize the extracted text concisely in Japanese.  
      - Uses expression to insert cleaned text from previous node or fallback message: "（テキストが見つかりませんでした）" (No text found).  
    - *Input:* Cleaned OCR text from Format OCR Result node  
    - *Output:* Summary text under `output` or `message.content`.  
    - *Potential Failures:* Expression evaluation errors, empty input, API or model response errors.

  - **Sticky Note3**  
    - *Content:* Describes this block’s role in summarizing extracted text efficiently.

---

#### 1.4 Logging to Google Sheets

- **Overview:**  
  Appends a new row to a Google Sheets document logging the filename, AI-generated summary, and processing date.

- **Nodes Involved:**  
  - Append row in sheet1  
  - Sticky Note4

- **Node Details:**

  - **Append row in sheet1**  
    - *Type & Role:* Google Sheets node for appending data to a spreadsheet.  
    - *Configuration:*  
      - Document ID: `11BmQnKfY4MmCD4iwlvOFUZPAM6kGMWdny8MefHzE2_0`  
      - Sheet name: sheet with GID `0` (named "要約")  
      - Columns mapped:  
        - `処理日` (Processing Date): current date in ISO format (YYYY-MM-DD)  
        - `要約結果` (Summary Result): AI summary from Generate Summary node  
        - `ファイル名` (Filename): extracted from Format OCR Result node  
      - Appends row operation  
    - *Input:* Summary and metadata from AI and formatting nodes  
    - *Output:* Confirmation of row append, triggers next node  
    - *Potential Failures:* Permission errors, sheet not found, invalid document ID, network errors.

  - **Sticky Note4**  
    - *Content:* Logs the summarized output with filename and date into a Google Sheet.

---

#### 1.5 Completion Notification via Gmail

- **Overview:**  
  Sends an email notification with the OCR file name, AI summary, and Google Docs link, then marks the process as completed.

- **Nodes Involved:**  
  - Send Completion Notification via Gmail1  
  - Process Completed1 (No Operation node)  
  - Sticky Note5

- **Node Details:**

  - **Send Completion Notification via Gmail1**  
    - *Type & Role:* Gmail node to send email notifications.  
    - *Configuration:*  
      - Recipient: `shukuwa@yubipass.tokyo` (modifiable)  
      - Subject: Includes filename with prefix "OCR処理完了:" (OCR Processing Complete)  
      - Message body: Includes  
        - Filename processed  
        - AI-generated summary content  
        - Google Docs document ID (from an unspecified Google Docs node referenced in message, likely external to this workflow)  
      - Uses Gmail OAuth2 credentials  
    - *Input:* Data from Format OCR Result, Generate Summary, and Google Docs node (external)  
    - *Output:* Triggers Process Completed1  
    - *Potential Failures:* Authentication issues, email quota exceeded, invalid email address, missing referenced document ID.

  - **Process Completed1**  
    - *Type & Role:* No Operation node marking workflow end.  
    - *Configuration:* None  
    - *Input:* From Gmail node  
    - *Output:* None  
    - *Potential Failures:* None (acts as endpoint)

  - **Sticky Note5**  
    - *Content:* Describes email notification and process completion.

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                           | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                  |
|----------------------------------|----------------------------------|-----------------------------------------|-----------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------|
| Google Drive New File Trigger1    | Google Drive Trigger              | Detect new image files in Drive folder  | -                                 | Download Image File1               | ### Section 1 ? Detect & Download Image                                                    |
| Download Image File1              | Google Drive                     | Download detected image file             | Google Drive New File Trigger1    | Extract Text with OCR.space1       | ### Section 1 ? Detect & Download Image                                                    |
| Extract Text with OCR.space1      | HTTP Request                     | Send image to OCR.space API              | Download Image File1              | Format OCR Result & Check for Empty1 | ### Section 2 ? Extract & Clean Text                                                     |
| Format OCR Result & Check for Empty1 | Code                          | Clean OCR text output, check empty text | Extract Text with OCR.space1      | Generate Summary with OpenRouter AI1 | ### Section 2 ? Extract & Clean Text                                                     |
| OpenRouter Chat Model1            | Langchain AI Language Model      | Provide OpenRouter AI language model     | -                                 | Generate Summary with OpenRouter AI1 | ### Section 3 ? Summarize with AI                                                      |
| Generate Summary with OpenRouter AI1 | Langchain Agent                | Generate concise Japanese summary via AI| Format OCR Result & Check for Empty1, OpenRouter Chat Model1 | Append row in sheet1             | ### Section 3 ? Summarize with AI                                                      |
| Append row in sheet1              | Google Sheets                   | Log filename, summary, and date          | Generate Summary with OpenRouter AI1 | Send Completion Notification via Gmail1 | ### Section 4 ? Save to Google Sheets                                                  |
| Send Completion Notification via Gmail1 | Gmail                      | Send email notification with summary info | Append row in sheet1             | Process Completed1                | ### Section 5 ? Notify via Gmail                                                       |
| Process Completed1                | No Operation                    | Marks end of workflow                     | Send Completion Notification via Gmail1 | -                               | ### Section 5 ? Notify via Gmail                                                       |
| Sticky Note                      | Sticky Note                     | Overview and instructions                 | -                                 | -                                 | ## Overview: Auto OCR → AI Summary → Google Sheets & Gmail                                |
| Sticky Note1                     | Sticky Note                     | Section 1 description                      | -                                 | -                                 | ### Section 1 ? Detect & Download Image                                                  |
| Sticky Note2                     | Sticky Note                     | Section 2 description                      | -                                 | -                                 | ### Section 2 ? Extract & Clean Text                                                    |
| Sticky Note3                     | Sticky Note                     | Section 3 description                      | -                                 | -                                 | ### Section 3 ? Summarize with AI                                                      |
| Sticky Note4                     | Sticky Note                     | Section 4 description                      | -                                 | -                                 | ### Section 4 ? Save to Google Sheets                                                  |
| Sticky Note5                     | Sticky Note                     | Section 5 description                      | -                                 | -                                 | ### Section 5 ? Notify via Gmail                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive New File Trigger node**  
   - Type: Google Drive Trigger  
   - Parameters:  
     - Event: `fileCreated`  
     - Trigger On: `specificFolder`  
     - Folder to Watch: Select or input folder ID (`1jTs0NNXMFTcChRq0C1KEoARVdIOOjQ1r`)  
     - Poll Interval: Every minute  
   - Credentials: Connect your Google Drive OAuth2 credentials  
   - Save node as `Google Drive New File Trigger1`

2. **Add Download Image File node**  
   - Type: Google Drive  
   - Operation: `download`  
   - File ID: Expression `{{$json["id"]}}` from trigger node  
   - Binary Property Name: `data`  
   - Connect input from `Google Drive New File Trigger1`  
   - Save as `Download Image File1`

3. **Add HTTP Request node for OCR.space**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.ocr.space/parse/image`  
   - Content Type: `multipart-form-data`  
   - Body Parameters:  
     - `apikey`: Your OCR.space API key (string)  
     - `language`: `jpn`  
     - `isOverlayRequired`: `false`  
     - `file`: Binary data from `Download Image File1` (property `data`)  
   - Connect input from `Download Image File1`  
   - Save as `Extract Text with OCR.space1`

4. **Add Code node to format OCR result and check for empty text**  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const raw = $input.first()?.json?.ParsedResults?.[0]?.ParsedText || '';
     const text = String(raw).replace(/\r/g, ' ').replace(/\n+/g, ' ').replace(/\s{2,}/g, ' ').trim();
     const fileName = $('Download Image File1').first()?.json?.name || 'unknown';
     return [{ json: { extractedText: text, fileName: fileName, hasText: text.length > 0 } }];
     ```  
   - Connect input from `Extract Text with OCR.space1`  
   - Save as `Format OCR Result & Check for Empty1`

5. **Add OpenRouter Chat Model node**  
   - Type: Langchain - LM Chat OpenRouter  
   - Credentials: Connect your OpenRouter API credentials  
   - Leave options as default unless customization needed  
   - Save as `OpenRouter Chat Model1`

6. **Add Langchain Agent node to generate summary**  
   - Type: Langchain Agent  
   - Prompt Type: Define  
   - Text parameter with expression:  
     ```
     以下の文章を簡潔に要約してください：
     {{ 
       $if(
         $('Format OCR Result & Check for Empty1').first()?.json?.extractedText, 
         $('Format OCR Result & Check for Empty1').first().json.extractedText, 
         "（テキストが見つかりませんでした）"
       ) 
     }}
     ```  
   - Connect AI language model input to `OpenRouter Chat Model1`  
   - Connect input from `Format OCR Result & Check for Empty1`  
   - Save as `Generate Summary with OpenRouter AI1`

7. **Add Google Sheets node to append data**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: `11BmQnKfY4MmCD4iwlvOFUZPAM6kGMWdny8MefHzE2_0` (replace with your own)  
   - Sheet Name: Use GID `0` or sheet named "要約"  
   - Columns: Define mapping for:  
     - `処理日`: expression `{{ $now.toISO().split('T')[0] }}` (current date)  
     - `要約結果`: expression `{{ $('Generate Summary with OpenRouter AI1').first().json.output }}`  
     - `ファイル名`: expression `{{ $('Format OCR Result & Check for Empty1').first().json.fileName }}`  
   - Connect input from `Generate Summary with OpenRouter AI1`  
   - Save as `Append row in sheet1`

8. **Add Gmail node to send notification email**  
   - Type: Gmail  
   - Send To: `shukuwa@yubipass.tokyo` (replace with your recipient)  
   - Subject: Expression `=OCR処理完了: {{ $('Format OCR Result & Check for Empty1').first().json.fileName }}`  
   - Message: Expression with multiline content including filename, summary, Google Docs document ID, e.g.:  
     ```
     画像ファイル「{{ $('Format OCR Result & Check for Empty1').first().json.fileName }}」のOCR処理が完了しました。

     【要約】
     {{ $('Generate Summary with OpenRouter AI1').first().json.message.content }}

     【Google Docs】
     新規ドキュメントを作成しました。
     ドキュメントID: {{ $('Google Docs へ要約を追記').first().json.documentId }}
     ```  
   - Connect input from `Append row in sheet1`  
   - Set Gmail OAuth2 credentials  
   - Save as `Send Completion Notification via Gmail1`

9. **Add No Operation node to mark completion**  
   - Type: No Operation  
   - Connect input from `Send Completion Notification via Gmail1`  
   - Save as `Process Completed1`

10. **Link all nodes sequentially as per connections:**  
    - `Google Drive New File Trigger1` → `Download Image File1` → `Extract Text with OCR.space1` → `Format OCR Result & Check for Empty1` → `Generate Summary with OpenRouter AI1` → `Append row in sheet1` → `Send Completion Notification via Gmail1` → `Process Completed1`  
    - Also link `OpenRouter Chat Model1` as AI language model input to `Generate Summary with OpenRouter AI1`

11. **Optional:**  
    - Add Sticky Notes for documentation in the workflow editor as per the original notes for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| This workflow automatically processes images uploaded to a specific Google Drive folder, extracts Japanese text via OCR.space, summarizes it using OpenRouter AI, logs the data into Google Sheets, and sends an email notification upon completion. It is ideal for digitizing books, lecture notes, or printed reports with minimal manual effort.                                                      | Overview sticky note included in workflow                                                                 |
| Setup requires connecting Google Drive, OpenRouter AI, Google Sheets, and Gmail credentials. The OCR.space API key must be inserted in the HTTP Request node. Update the Google Drive folder ID, Google Sheets document ID, and email recipient as needed.                                                                                                                                             | Setup instructions in Sticky Note                                                                       |
| OpenRouter AI is used via the Langchain integration in n8n, enabling flexible prompt management and AI summarization.                                                                                                                                                                                                                                                                           | OpenRouter API credentials required                                                                       |
| Google Sheets is used to store processed data with columns for filename, summary, and processing date, facilitating easy tracking and record keeping.                                                                                                                                                                                                                                            | Google Sheets document ID and sheet name must be configured                                               |
| Gmail notifications include the processed file name, AI summary, and a Google Docs document ID (referenced but creation node not included in this workflow JSON). Adjust recipient email accordingly.                                                                                                                                                                                           | Gmail OAuth2 credentials and recipient email address required                                            |
| For more information on OCR.space API: https://ocr.space/ocrapi                                                                                                                                                                                                                                                                                                                                   | OCR.space official API documentation                                                                      |
| For OpenRouter AI and Langchain integration details, refer to n8n Langchain node documentation.                                                                                                                                                                                                                                                                                                    | n8n Langchain node docs                                                                                   |

---

This structured reference fully discloses the workflow logic, node configurations, connections, and key considerations to facilitate understanding, reproduction, and modification.