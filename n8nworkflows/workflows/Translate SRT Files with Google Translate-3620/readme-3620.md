Translate SRT Files with Google Translate

https://n8nworkflows.xyz/workflows/translate-srt-files-with-google-translate-3620


# Translate SRT Files with Google Translate

### 1. Workflow Overview

This workflow automates the translation of subtitle files in SRT format using Google Translate API. It is designed for users who need to convert subtitles from one language to another efficiently while preserving the SRT structure, including timestamps and numbering.

**Target Use Cases:**  
- Content creators preparing multilingual videos  
- Video editors localizing subtitles  
- Translators working with subtitle files  
- Educational content distributors requiring translated subtitles  

**Logical Blocks:**  
- **1.1 Input Reception:** Receive the SRT file and target language from the user via a web form.  
- **1.2 File Extraction and Parsing:** Extract text content from the uploaded binary SRT file and parse it into subtitle groups.  
- **1.3 Text Preparation for Translation:** Split subtitle entries into parts suitable for translation and prepare language parameters.  
- **1.4 Translation Processing:** Use Google Translate API to translate subtitle text segments.  
- **1.5 Post-Processing and Reassembly:** Clean translated text, format it properly into subtitle lines, and aggregate all translated segments.  
- **1.6 Output Generation:** Convert the translated text back into a binary file and respond to the user with the translated SRT file.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block handles user input by presenting a form to upload an SRT file and select the target translation language.

**Nodes Involved:**  
- Receive SRT File to Translate

**Node Details:**  
- **Receive SRT File to Translate**  
  - Type: Form Trigger  
  - Role: Entry point for the workflow; collects user input via a web form.  
  - Configuration:  
    - Form fields include a dropdown for "Translate to Language" with options "EN" and "JP" (English and Japanese).  
    - File upload field accepts only `.srt` files.  
    - Response mode set to return data from the last node executed.  
  - Inputs: Webhook trigger from user submission  
  - Outputs: JSON containing uploaded file metadata and selected language  
  - Edge Cases:  
    - User uploads unsupported file type (not `.srt`) — form restricts this but validation should be considered.  
    - User submits without selecting language or file — form requires these fields.  
  - Notes: Language options can be extended by modifying the dropdown field.

---

#### 1.2 File Extraction and Parsing

**Overview:**  
Extracts the text content from the uploaded binary SRT file and parses it into logical subtitle groups for translation.

**Nodes Involved:**  
- Expose Binary  
- Extract text from Binary File  
- Split SRT Lines

**Node Details:**  
- **Expose Binary**  
  - Type: Code  
  - Role: Transfers the binary data of the uploaded file into JSON for further processing.  
  - Configuration: Copies `$binary` property into JSON field `binary`.  
  - Inputs: Output from form trigger node  
  - Outputs: JSON with binary data exposed  
  - Edge Cases: Binary data missing or corrupted could cause failures.

- **Extract text from Binary File**  
  - Type: Extract From File  
  - Role: Extracts plain text from the binary SRT file.  
  - Configuration: Operation set to "text" on binary property `Upload_SRT_file`.  
  - Inputs: JSON with binary data  
  - Outputs: JSON with extracted text content  
  - Edge Cases: File encoding issues or corrupted files may cause extraction errors.

- **Split SRT Lines**  
  - Type: Code  
  - Role: Parses the extracted text into grouped subtitle entries by splitting on blank lines.  
  - Configuration:  
    - Splits text by newlines, groups lines into subtitle blocks separated by empty lines.  
    - Cleans quotes from first and last subtitle groups.  
    - Stores grouped subtitles in JSON field `txt`.  
  - Inputs: Extracted text JSON  
  - Outputs: JSON with array of subtitle groups (`txt`)  
  - Edge Cases: Malformed SRT files with irregular spacing or missing blank lines may cause incorrect grouping.

---

#### 1.3 Text Preparation for Translation

**Overview:**  
Prepares subtitle text segments for translation by splitting each subtitle entry into two parts: the numbering/timestamp and the actual subtitle text. Also retrieves the target language.

**Nodes Involved:**  
- Split Out  
- Prep Parts for Translate

**Node Details:**  
- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of subtitle groups (`txt`) into individual items for processing.  
  - Configuration: Field to split out is `txt`.  
  - Inputs: Parsed subtitle groups  
  - Outputs: Individual subtitle entries as separate items  
  - Edge Cases: Empty or missing `txt` field would cause no output.

- **Prep Parts for Translate**  
  - Type: Code  
  - Role: Splits each subtitle entry into two parts: the first two lines (number and timestamp) and the remaining subtitle text. Also attaches the target language from the form input.  
  - Configuration:  
    - Splits subtitle string at second newline.  
    - Returns an object with `firstPart` (number and timestamp), `secondPart` (text to translate), and `language`.  
  - Inputs: Individual subtitle entries  
  - Outputs: JSON with `parts` object and language code  
  - Edge Cases: Subtitle entries with fewer than two newlines will have empty `secondPart`, potentially causing empty translations.

---

#### 1.4 Translation Processing

**Overview:**  
Translates the subtitle text segments using Google Translate API.

**Nodes Involved:**  
- Google Translate

**Node Details:**  
- **Google Translate**  
  - Type: Google Translate  
  - Role: Translates the subtitle text (`secondPart`) into the target language.  
  - Configuration:  
    - Text input is JSON stringified `parts.secondPart`.  
    - Target language is dynamically set from `$json.language`.  
    - Uses OAuth2 credentials configured for Google Translate API.  
  - Inputs: Prepared subtitle parts with text to translate and language  
  - Outputs: Translated text in JSON field `translatedText`  
  - Version Specifics: Uses version 2 of the Google Translate node.  
  - Edge Cases:  
    - API authentication failure (invalid or expired OAuth2 token).  
    - Rate limiting or quota exceeded on Google Translate API.  
    - Empty or malformed text input causing translation errors.

---

#### 1.5 Post-Processing and Reassembly

**Overview:**  
Cleans the translated text, formats it properly into subtitle lines, and aggregates all translated segments back into a complete SRT content.

**Nodes Involved:**  
- Clean Translations & Group Titles  
- Aggregate  
- Join completed text with double new line  
- Edit Fields  
- Generate Binary

**Node Details:**  
- **Clean Translations & Group Titles**  
  - Type: Code  
  - Role: Cleans translation artifacts (e.g., HTML entities), formats long subtitle lines into two lines if needed, and recombines with numbering and timestamp.  
  - Configuration:  
    - Replaces escaped newline characters and HTML entities.  
    - Splits long lines (>40 characters) at last space before limit.  
    - Constructs a new field `complete` combining `firstPart` and cleaned, formatted translation.  
  - Inputs: Translated text per subtitle entry  
  - Outputs: JSON with `complete` subtitle entry  
  - Edge Cases: Translations containing unexpected characters or formatting may require additional cleaning.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Collects all individual translated subtitle entries into an array.  
  - Configuration: Aggregates all item data into one array.  
  - Inputs: Cleaned and formatted subtitle entries  
  - Outputs: Single item with array of translated subtitles in `data` field  
  - Edge Cases: Empty input array results in empty aggregation.

- **Join completed text with double new line**  
  - Type: Code  
  - Role: Joins the array of translated subtitle entries into a single string separated by double newlines, restoring the SRT format.  
  - Configuration: Joins `data` array items by `\n\n` into `complete_text`.  
  - Inputs: Aggregated subtitle entries  
  - Outputs: JSON with full translated SRT text in `complete_text`  
  - Edge Cases: Missing or empty `data` array results in empty output.

- **Edit Fields**  
  - Type: Set  
  - Role: Assigns the final translated text to a field and attaches original file metadata for output.  
  - Configuration:  
    - Sets `complete_text` field with joined translated text.  
    - Copies original file metadata from the initial form node.  
  - Inputs: Joined translated text  
  - Outputs: JSON with final text and file metadata  
  - Edge Cases: Missing original file metadata may affect output file naming.

- **Generate Binary**  
  - Type: Code  
  - Role: Encodes the translated text into Base64 and prepares the binary file object for output.  
  - Configuration:  
    - Encodes `complete_text` to Base64 using environment-appropriate methods.  
    - Calculates file size from Base64 length.  
    - Stores encoded data and size in file object.  
  - Inputs: Final translated text and file metadata  
  - Outputs: Binary file object ready for conversion  
  - Edge Cases: Encoding errors, environment incompatibility, or corrupted text data.

---

#### 1.6 Output Generation

**Overview:**  
Converts the Base64 encoded text into a binary file and responds to the user with the translated SRT file.

**Nodes Involved:**  
- Convert to File  
- Respond with file

**Node Details:**  
- **Convert to File**  
  - Type: Convert To File  
  - Role: Converts Base64 encoded text into a binary file with appropriate filename and MIME type.  
  - Configuration:  
    - Filename derived from original filename, replacing `.srt` with ` <language>.srt`.  
    - MIME type preserved from original upload.  
    - Source property is Base64 encoded `data`.  
    - Output stored in binary property `file`.  
  - Inputs: Base64 encoded file object  
  - Outputs: Binary file object with proper filename and MIME type  
  - Edge Cases: Filename or MIME type missing or malformed could affect file integrity.

- **Respond with file**  
  - Type: Form  
  - Role: Sends the translated file back to the user as a downloadable response.  
  - Configuration:  
    - Operation set to "completion".  
    - Responds with binary data from `file` property.  
    - Completion title set to "Done".  
  - Inputs: Binary file object  
  - Outputs: HTTP response with translated SRT file download  
  - Edge Cases: Network issues or large file sizes may affect response delivery.

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                         | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                  |
|--------------------------------|-----------------------|---------------------------------------|-------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Receive SRT File to Translate   | Form Trigger          | Accept user file upload and language  | —                             | Expose Binary                  | Required Credentials: https://docs.n8n.io/integrations/builtin/credentials/google/ <br> Update language dropdown as needed.   |
| Expose Binary                  | Code                  | Expose binary data for processing     | Receive SRT File to Translate  | Extract text from Binary File   |                                                                                                                              |
| Extract text from Binary File  | Extract From File     | Extract text content from binary file | Expose Binary                 | Split SRT Lines                |                                                                                                                              |
| Split SRT Lines               | Code                  | Parse SRT text into subtitle groups   | Extract text from Binary File | Split Out                     |                                                                                                                              |
| Split Out                    | Split Out             | Split subtitle groups into items      | Split SRT Lines               | Prep Parts for Translate       |                                                                                                                              |
| Prep Parts for Translate       | Code                  | Split subtitle entry and set language | Split Out                    | Google Translate               |                                                                                                                              |
| Google Translate              | Google Translate      | Translate subtitle text segments       | Prep Parts for Translate      | Clean Translations & Group Titles |                                                                                                                              |
| Clean Translations & Group Titles | Code               | Clean and format translated text       | Google Translate             | Aggregate                     |                                                                                                                              |
| Aggregate                    | Aggregate             | Aggregate translated subtitle entries  | Clean Translations & Group Titles | Join completed text with double new line |                                                                                                                              |
| Join completed text with double new line | Code          | Join all translated entries into SRT text | Aggregate                   | Edit Fields                   |                                                                                                                              |
| Edit Fields                  | Set                   | Assign final text and file metadata    | Join completed text with double new line | Generate Binary              |                                                                                                                              |
| Generate Binary              | Code                  | Encode text to Base64 and prepare file | Edit Fields                  | Convert to File               |                                                                                                                              |
| Convert to File              | Convert To File       | Convert Base64 text to binary file     | Generate Binary              | Respond with file             |                                                                                                                              |
| Respond with file            | Form                  | Return translated file to user         | Convert to File              | —                            |                                                                                                                              |
| Sticky Note                 | Sticky Note           | Notes on credentials and language setup | —                            | —                            | Required Credentials: https://docs.n8n.io/integrations/builtin/credentials/google/ <br> Update language dropdown as needed.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: "Receive SRT File to Translate"  
   - Configure webhook with form fields:  
     - Dropdown "Translate to Language" with options "EN" and "JP" (add more as needed).  
     - File upload field accepting `.srt` files only.  
   - Set response mode to "lastNode".

2. **Add a Code Node "Expose Binary"**  
   - Purpose: Copy the binary file data into JSON for extraction.  
   - JavaScript:  
     ```js
     $input.item.json.binary = $binary;
     return $input.item;
     ```  
   - Connect from "Receive SRT File to Translate".

3. **Add an Extract From File Node "Extract text from Binary File"**  
   - Operation: Text  
   - Binary Property Name: `Upload_SRT_file` (match the uploaded file property name).  
   - Connect from "Expose Binary".

4. **Add a Code Node "Split SRT Lines"**  
   - Purpose: Parse text into subtitle groups separated by blank lines.  
   - JavaScript:  
     ```js
     let text = $json.data;
     delete $json.base64;
     delete $json.binary;
     const lines = text.split('\n');
     let subtitleGroups = [];
     let currentGroup = [];
     for (let i = 0; i < lines.length; i++) {
       const line = lines[i].trim();
       if (line === '' && currentGroup.length > 0) {
         subtitleGroups.push(currentGroup.join('\n'));
         currentGroup = [];
       } else if (line !== '') {
         currentGroup.push(line);
       }
     }
     if (currentGroup.length > 0) {
       subtitleGroups.push(currentGroup.join('\n'));
     }
     if (subtitleGroups.length > 0) {
       subtitleGroups[0] = subtitleGroups[0].replace(/^\"/, '');
       subtitleGroups[subtitleGroups.length - 1] = subtitleGroups[subtitleGroups.length - 1].replace(/\"$/, '');
     }
     $input.item.json.txt = subtitleGroups;
     return $input.item;
     ```  
   - Connect from "Extract text from Binary File".

5. **Add a Split Out Node "Split Out"**  
   - Field to Split Out: `txt`  
   - Connect from "Split SRT Lines".

6. **Add a Code Node "Prep Parts for Translate"**  
   - Purpose: Split subtitle into number/timestamp and text, attach language.  
   - JavaScript:  
     ```js
     function splitBySecondNewline(text) {
       const firstNewlinePos = text.indexOf('\n');
       if (firstNewlinePos === -1) return { firstPart: text, secondPart: '' };
       const secondNewlinePos = text.indexOf('\n', firstNewlinePos + 1);
       if (secondNewlinePos === -1) return { firstPart: text, secondPart: '' };
       return {
         firstPart: text.substring(0, secondNewlinePos),
         secondPart: text.substring(secondNewlinePos + 1)
       };
     }
     let lang = $('Receive SRT File to Translate').first().json['Translate to Language'];
     return {
       parts: splitBySecondNewline($json.txt),
       language: lang
     };
     ```  
   - Connect from "Split Out".

7. **Add a Google Translate Node "Google Translate"**  
   - Text to Translate: `={{ JSON.stringify($json.parts.secondPart) }}`  
   - Translate To: `={{ $json.language }}`  
   - Credentials: Configure Google Translate OAuth2 credentials.  
   - Connect from "Prep Parts for Translate".

8. **Add a Code Node "Clean Translations & Group Titles"**  
   - Purpose: Clean translation artifacts, format long lines, recombine subtitle parts.  
   - JavaScript:  
     ```js
     let translated = $json.translatedText.replaceAll("\\n","\n").replaceAll('&quot;',"").replaceAll('&#39;',"'");
     function splitIntoTwoLines(text, maxLength = 40) {
       if (text.includes('\n') || text.length <= maxLength) return text;
       let splitIndex = text.lastIndexOf(' ', maxLength);
       if (splitIndex === -1) splitIndex = maxLength;
       const firstLine = text.substring(0, splitIndex);
       const secondLine = text.substring(splitIndex + 1);
       return firstLine + '\n' + secondLine;
     }
     $input.item.json.complete = `${$('Prep Parts for Translate').item.json.parts.firstPart}\n` + splitIntoTwoLines(translated);
     return $input.item;
     ```  
   - Connect from "Google Translate".

9. **Add an Aggregate Node "Aggregate"**  
   - Aggregate: Aggregate all item data  
   - Connect from "Clean Translations & Group Titles".

10. **Add a Code Node "Join completed text with double new line"**  
    - Purpose: Join all translated subtitle entries into one SRT text string.  
    - JavaScript:  
      ```js
      let texts = $json.data.map(item => item.complete);
      $input.item.json.complete_text = texts.join('\n\n');
      return $input.item;
      ```  
    - Connect from "Aggregate".

11. **Add a Set Node "Edit Fields"**  
    - Assign fields:  
      - `complete_text` = `={{ $json.complete_text }}`  
      - `file` = `={{$('Receive SRT File to Translate').first().json}}` (copies original file metadata)  
    - Connect from "Join completed text with double new line".

12. **Add a Code Node "Generate Binary"**  
    - Purpose: Encode translated text to Base64 and prepare file object.  
    - JavaScript:  
      ```js
      function encodeBase64(text) {
        try {
          if (typeof window !== 'undefined') {
            const utf8String = encodeURIComponent(text).replace(/%([0-9A-F]{2})/g, (_, hex) => String.fromCharCode(parseInt(hex, 16)));
            return btoa(utf8String);
          } else if (typeof Buffer !== 'undefined') {
            return Buffer.from(text).toString('base64');
          }
          throw new Error('Environment not supported for Base64 encoding');
        } catch (error) {
          console.error('Error encoding to Base64:', error);
          return null;
        }
      }
      let data = encodeBase64($json.complete_text);
      let file = $json.file;
      file.data = data;
      let paddingCount = 0;
      if (data.endsWith('==')) paddingCount = 2;
      else if (data.endsWith('=')) paddingCount = 1;
      file.size = Math.floor(data.length * 3 / 4) - paddingCount;
      return file;
      ```  
    - Connect from "Edit Fields".

13. **Add a Convert To File Node "Convert to File"**  
    - Operation: To Binary  
    - Source Property: `data`  
    - Binary Property Name: `file`  
    - File Name: `={{ $json['Upload SRT file'].filename.replaceAll('.srt',` ${$('Prep Parts for Translate').first().json.language}.srt`)}}`  
    - MIME Type: `={{ $json['Upload SRT file'].mimetype }}`  
    - Connect from "Generate Binary".

14. **Add a Form Node "Respond with file"**  
    - Operation: Completion  
    - Respond With: Return Binary  
    - Completion Title: "Done"  
    - Input Data Field Name: `file`  
    - Connect from "Convert to File".

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Required Credentials: Google Cloud project with Translate API enabled and OAuth2 credentials configured. | https://docs.n8n.io/integrations/builtin/credentials/google/                                    |
| Language dropdown can be extended by editing the form trigger node's dropdown options.                    | Workflow configuration step                                                                     |
| Google Translate node can be set to fixed language instead of dynamic form input to simplify setup.       | Workflow configuration note                                                                     |

---

This documentation fully describes the workflow structure, node configurations, data flow, and setup instructions to enable reproduction, modification, and troubleshooting.