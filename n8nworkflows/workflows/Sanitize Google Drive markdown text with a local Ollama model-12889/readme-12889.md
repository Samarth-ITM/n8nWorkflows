Sanitize Google Drive markdown text with a local Ollama model

https://n8nworkflows.xyz/workflows/sanitize-google-drive-markdown-text-with-a-local-ollama-model-12889


# Sanitize Google Drive markdown text with a local Ollama model

## 1. Workflow Overview

**Purpose:**  
This workflow monitors a specific Google Drive folder for newly created subfolders. When a new subfolder appears, it retrieves all files in that subfolder, filters for Markdown files (`text/markdown`), downloads and extracts their text, then uses a **local Ollama LLM (llama3.1)** to **sanitize/redact PII** (Personally Identifiable Information). The sanitized content is recombined and saved back to Google Drive as a new Markdown file. Optionally, each processed chunk is logged to Google Sheets along with a `pii_found` flag.

**Target use cases:**
- Cleaning private/internal Markdown documents before sending them to public LLMs.
- Automated PII redaction pipeline for Drive-based document intake.

### 1.1 Trigger & File Discovery
Detects new subfolders and lists the files inside the created subfolder.

### 1.2 Filter, Download & Text Extraction
Keeps only Markdown files, downloads them, extracts text content, and aggregates all text.

### 1.3 Chunking & Local LLM Sanitization
Splits combined text into fixed-size chunks, sends each chunk to an Ollama-backed LLM chain, and parses strict JSON responses.

### 1.4 Output Assembly & Persistence
Aggregates sanitized chunks, joins them into a single text body, writes a cleaned Markdown file, and optionally logs chunk results to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & File Discovery
**Overview:**  
Watches a specific Google Drive folder for new subfolder creation. Once a subfolder is created, it queries the Drive API to list all files within that subfolder and splits the returned list into individual items.

**Nodes involved:**
- `on_subfolder_created`
- `return_files_in_folder`
- `split_files_item`

#### Node: on_subfolder_created
- **Type / role:** Google Drive Trigger (`googleDriveTrigger`) — entry point.
- **Config (interpreted):**
  - Event: **folderCreated**
  - Trigger scope: **specificFolder**
  - Polling: **every minute**
  - Folder to watch: configured by `folder_id` (list selector).
- **Inputs / outputs:**
  - **No input** (trigger).
  - Outputs a JSON object describing the created folder (notably includes `id`).
- **Edge cases / failures:**
  - OAuth scope issues or revoked token.
  - Polling delay/duplicates (common with polling triggers).
  - Folder created but later removed/permissions changed before processing starts.
- **Version notes:** Node typeVersion `1`.

#### Node: return_files_in_folder
- **Type / role:** HTTP Request — lists files in the created folder via Drive API.
- **Config (interpreted):**
  - Method: GET (implicit)
  - URL (expression):  
    `https://www.googleapis.com/drive/v3/files?q="{{ $json.id }}"+in+parents&fields=files(id,name,mimeType,modifiedTime,size)`
  - Auth: **predefinedCredentialType** using **Google Drive OAuth2**.
  - Output expected: JSON with `files: [...]`.
- **Key variables/expressions:** Uses trigger output: `{{ $json.id }}` (the new folder ID).
- **Inputs / outputs:**
  - Input: from `on_subfolder_created`.
  - Output: JSON with a `files` array.
- **Edge cases / failures:**
  - Insufficient permissions to list subfolder contents.
  - Folder ID not present (unexpected trigger payload) → malformed query.
  - Google API rate limits / transient HTTP errors.
- **Notes:** The node’s note says: `GET https://www.googleapis.com/drive/v3/files?q=folder_id` (conceptual hint).

#### Node: split_files_item
- **Type / role:** Split Out (`splitOut`) — converts a list into one item per element.
- **Config (interpreted):**
  - Field to split: `files`
- **Inputs / outputs:**
  - Input: single item containing `files[]`.
  - Output: multiple items, each representing one file object.
- **Edge cases / failures:**
  - If `files` is missing or empty → produces no items (downstream nodes won’t run).

---

### Block 2 — Filter, Download & Text Extraction
**Overview:**  
Filters the file list to keep only Markdown files, downloads them from Drive, extracts text content, and aggregates all extracted text into a single combined dataset.

**Nodes involved:**
- `select_markdown_files`
- `download_drive_files`
- `text_from_markdown`
- `combine_all_text`

#### Node: select_markdown_files
- **Type / role:** Filter — retains only Markdown files.
- **Config (interpreted):**
  - Condition: `{{$json.mimeType}}` **equals** `text/markdown`
  - Strict type validation enabled.
- **Inputs / outputs:**
  - Input: each file item from `split_files_item`.
  - Output: only matching items continue.
- **Edge cases / failures:**
  - Google Drive may report alternative MIME types for some Markdown uploads (or unknown types) → files may be skipped unintentionally.
  - Non-standard markdown could be `text/plain` → excluded unless condition updated.

#### Node: download_drive_files
- **Type / role:** Google Drive — downloads the file binary.
- **Config (interpreted):**
  - Operation: **download**
  - File ID: `{{$json.id}}`
  - Auth: Google Drive OAuth2.
- **Inputs / outputs:**
  - Input: filtered file metadata.
  - Output: binary data for each file (n8n binary property).
- **Edge cases / failures:**
  - File permission issues or file deleted between list and download.
  - Large files can cause timeouts/memory pressure depending on n8n limits.

#### Node: text_from_markdown
- **Type / role:** Extract From File — extracts plain text from downloaded file.
- **Config (interpreted):**
  - Operation: **text**
  - Uses incoming binary as source.
- **Inputs / outputs:**
  - Input: binary from `download_drive_files`.
  - Output: JSON containing extracted text (stored under `data` as used later).
- **Edge cases / failures:**
  - Unexpected encoding or binary missing → extraction fails.
  - If the extractor returns arrays/segments, later aggregation logic assumes iterable text segments.

#### Node: combine_all_text
- **Type / role:** Aggregate — collects extracted text from all markdown files.
- **Config (interpreted):**
  - Aggregates field: `data`
- **Inputs / outputs:**
  - Input: multiple items (one per markdown file’s extracted text).
  - Output: a single aggregated item where `data` is an array of texts/segments.
- **Edge cases / failures:**
  - If no markdown files passed filter → no input → downstream won’t execute.
  - Memory growth if many/large markdown files.

---

### Block 3 — Chunking & Local LLM Sanitization
**Overview:**  
Splits combined text into fixed-size chunks, sends each chunk to a local Ollama model using an LLM chain prompt that enforces strict JSON output, and parses the model’s response into structured fields.

**Nodes involved:**
- `chunk_text_for_local_llm`
- `Basic LLM Chain`
- `Ollama Model`
- `parse_json_text_with_flag`

#### Node: chunk_text_for_local_llm
- **Type / role:** Code node (Python) — chunks long text into fixed-size segments.
- **Config (interpreted):**
  - Language: Python
  - `text_chunk_size = 250` characters
  - For each incoming item:
    - `item_string = ''.join(item.json.data)`
    - Builds `[{ "data": chunk }, ...]` for chunk windows `[i:i+text_chunk_size]`
  - Returns `string_array` (multiple output items).
- **Inputs / outputs:**
  - Input: aggregated item from `combine_all_text` (`data` array).
  - Output: many items each with `json.data` (chunk string).
- **Edge cases / failures:**
  - If `item.json.data` is not an array of strings, `''.join(...)` may error or produce unexpected output.
  - Character-based chunking can split entities (emails, names, IDs) across boundaries → may reduce redaction quality.
  - Very small chunk size increases LLM calls and latency/cost.

#### Node: Basic LLM Chain
- **Type / role:** LangChain LLM Chain (`chainLlm`) — prompts LLM to redact PII with strict JSON output contract.
- **Config (interpreted):**
  - Prompt includes:
    - Instruction to remove PII without changing meaning.
    - Enumerated PII types.
    - Output must be **only** a single JSON object with:
      - `sanitized_text` (string)
      - `pii_found` (boolean)
  - Input text inserted via: `{{ $json.data }}`
  - Uses connected model via `ai_languageModel` relationship.
- **Inputs / outputs:**
  - Input: each chunk item from `chunk_text_for_local_llm`.
  - Output: LLM response text (typically exposed as a `text` field in n8n’s AI nodes).
- **Edge cases / failures:**
  - Model may return non-JSON (extra text, markdown, partial JSON) → parsing node may fail.
  - Hallucinated structure (extra keys) violates schema.
  - Timeouts if local Ollama is overloaded.
- **Version notes:** typeVersion `1.7`.

#### Node: Ollama Model
- **Type / role:** Ollama LLM connector (`lmOllama`) — provides local model inference.
- **Config (interpreted):**
  - Model: `llama3.1:latest`
  - Credentials: Ollama API (typically points to local Ollama base URL).
- **Inputs / outputs:**
  - Connected as the language model provider to `Basic LLM Chain`.
- **Edge cases / failures:**
  - Ollama not running / wrong host or port in credentials.
  - Model not pulled locally → request fails.
  - Concurrency limits and slow inference.
- **Version notes:** typeVersion `1`.

#### Node: parse_json_text_with_flag
- **Type / role:** Set node — cleans LLM output text and parses JSON into an `output` object.
- **Config (interpreted):**
  - `ignoreConversionErrors: true` (reduces hard failures on conversion, but expressions can still fail if parsing throws)
  - Creates field `output` as object from:
    - `$json.text.replaceAll("\r","").replaceAll("\n","").replaceAll("```","")`
    - then `.toJsonString().parseJson()`
- **Inputs / outputs:**
  - Input: LLM chain output (expects a `text` property containing JSON string).
  - Output: JSON with `output.sanitized_text` and `output.pii_found`.
- **Edge cases / failures:**
  - If `$json.text` is missing or not a string → expression fails.
  - If the model returns JSON containing unescaped quotes/newlines, naive newline removal may break validity.
  - Removing all newlines may damage legitimate JSON formatting if model returns multi-line strings.
  - If model outputs leading/trailing commentary, stripping backticks may be insufficient → parse fails.

---

### Block 4 — Output Assembly, Logging & File Creation
**Overview:**  
Aggregates sanitized text chunks back into one ordered list, joins them into a single text string, writes a cleaned Markdown file into Google Drive, and optionally appends chunk-level logs to a Google Sheet.

**Nodes involved:**
- `log_text_with_pii_flag (optional)`
- `combine_chunk_text`
- `join_text_chunks`
- `create_cleaned_text_file`

#### Node: log_text_with_pii_flag (optional)
- **Type / role:** Google Sheets — append chunk-level audit logs.
- **Config (interpreted):**
  - Operation: **append**
  - Document: `spreadsheet_id` (list selector)
  - Sheet tab: `gid=0` (named “logs” in cached label)
  - Columns mapped:
    - `cleaned text`: `{{$json.output.sanitized_text}}`
    - `pii found`: `{{$json.output.pii_found}}`
    - `created_at`: `{{$now.toISO()}}`
- **Inputs / outputs:**
  - Input: each parsed chunk result from `parse_json_text_with_flag`.
  - Output: append result (not used further).
- **Edge cases / failures:**
  - Sheet schema mismatch (renamed columns, wrong tab).
  - Rate limits if many chunks.
  - OAuth permission issues.

#### Node: combine_chunk_text
- **Type / role:** Aggregate — collects sanitized text from all chunk outputs.
- **Config (interpreted):**
  - Aggregates field: `output.sanitized_text`
  - Produces an array of sanitized chunk strings.
- **Inputs / outputs:**
  - Input: multiple items (one per chunk).
  - Output: a single item with aggregated array.
- **Edge cases / failures:**
  - If any chunk missing `output.sanitized_text`, aggregation may produce null/empty entries.

#### Node: join_text_chunks
- **Type / role:** Set — joins array of sanitized strings into one string.
- **Config (interpreted):**
  - Sets `sanitized_text[0]` to: `{{ $json.sanitized_text.join() }}`
  - Note: `.join()` with no separator uses a comma `,` by default in JS.
- **Inputs / outputs:**
  - Input: aggregated sanitized chunk array.
  - Output: JSON with `sanitized_text[0]` containing the joined text.
- **Edge cases / failures:**
  - Default comma separator will insert commas between chunks; for text reconstruction you typically want `.join('')` or `.join('\n')`.
  - Field naming `sanitized_text[0]` is unusual and can confuse downstream mapping.

#### Node: create_cleaned_text_file
- **Type / role:** Google Drive — creates a new markdown file from text.
- **Config (interpreted):**
  - Operation: **createFromText**
  - File name: `{{$now.toISO()}}_cleaned_text.md ` (note trailing space)
  - Content: `{{$json.sanitized_text[0]}}`
  - Target drive: “My Drive”
  - Folder ID: set to `folder_id` (static selection in node, not dynamically using the created subfolder)
- **Inputs / outputs:**
  - Input: joined sanitized text.
  - Output: created file metadata (not used further).
- **Edge cases / failures:**
  - Writes to a fixed folder (`folder_id`) rather than the newly created subfolder; if the intent is “same folder”, you must map folderId dynamically.
  - Trailing space in filename may be undesirable.
  - Drive permission errors or quota issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation header and requirements |  |  | ## Sanitize text with a local Ollama model… (includes requirements and Google OAuth link: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/#enable-apis) |
| on_subfolder_created | Google Drive Trigger | Watches for new subfolder creation | — | return_files_in_folder | ## Sanitize text… |
| return_files_in_folder | HTTP Request | Lists files in newly created folder via Drive API | on_subfolder_created | split_files_item | ### 1. Retrieves filenames from the google drive folder and filters for markdown (.md) files. |
| split_files_item | Split Out | Splits `files[]` into one item per file | return_files_in_folder | select_markdown_files | ### 1. Retrieves filenames… |
| select_markdown_files | Filter | Keeps only `text/markdown` | split_files_item | download_drive_files | ### 1. Retrieves filenames… |
| download_drive_files | Google Drive | Downloads each markdown file | select_markdown_files | text_from_markdown | ### 2. Downloads the markdown files and combines the extracted text. |
| text_from_markdown | Extract From File | Extracts text from downloaded markdown | download_drive_files | combine_all_text | ### 2. Downloads the markdown files and combines the extracted text. |
| combine_all_text | Aggregate | Aggregates extracted text from all files | text_from_markdown | chunk_text_for_local_llm | ### 2. Downloads the markdown files and combines the extracted text. |
| chunk_text_for_local_llm | Code (Python) | Splits combined text into chunks | combine_all_text | Basic LLM Chain | ### 3. Chunks the text, sends it to the Ollama model, and parses the JSON response. |
| Basic LLM Chain | LangChain Chain LLM | Prompts model to redact PII and output strict JSON | chunk_text_for_local_llm | parse_json_text_with_flag | ### 3. Chunks the text, sends it to the Ollama model, and parses the JSON response. |
| Ollama Model | Ollama LLM | Local LLM provider for chain | — (AI connection) | Basic LLM Chain (AI) | ### 3. Chunks the text, sends it to the Ollama model, and parses the JSON response. |
| parse_json_text_with_flag | Set | Parses model output JSON into `output` | Basic LLM Chain | log_text_with_pii_flag (optional); combine_chunk_text | ### 3. Chunks the text, sends it to the Ollama model, and parses the JSON response. |
| log_text_with_pii_flag (optional) | Google Sheets | Logs each chunk’s sanitized text + pii_found | parse_json_text_with_flag | — | ### 4b. Logs the model responses, including the flag indicating whether PII information was found. (optional). |
| combine_chunk_text | Aggregate | Aggregates `output.sanitized_text` across chunks | parse_json_text_with_flag | join_text_chunks | ### 4a. Combines the model responses and creates a new markdown file in the google drive folder. |
| join_text_chunks | Set | Joins chunk array into one text | combine_chunk_text | create_cleaned_text_file | ### 4a. Combines the model responses and creates a new markdown file in the google drive folder. |
| create_cleaned_text_file | Google Drive | Creates a cleaned markdown file from text | join_text_chunks | — | ### 4a. Combines the model responses and creates a new markdown file in the google drive folder. |
| Sticky Note1 | Sticky Note | Section label |  |  | ### 1. Retrieves filenames from the google drive folder and filters for markdown (.md) files. |
| Sticky Note2 | Sticky Note | Section label |  |  | ### 3. Chunks the text, sends it to the Ollama model, and parses the JSON response. |
| Sticky Note3 | Sticky Note | Section label |  |  | ### 2. Downloads the markdown files and combines the extracted text. |
| Sticky Note4 | Sticky Note | Section label |  |  | ### 4b. Logs the model responses… (optional). |
| Sticky Note5 | Sticky Note | Section label |  |  | ### 4a. Combines the model responses and creates a new markdown file in the google drive folder. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger node**
   - Add **Google Drive Trigger** named `on_subfolder_created`.
   - Set **Event** = `folderCreated`.
   - Set **Trigger On** = `specificFolder`.
   - Choose the parent folder to watch (this is your intake folder).
   - Set polling to **Every Minute**.
   - Attach **Google Drive OAuth2** credentials.

2. **List files in the created subfolder**
   - Add **HTTP Request** node `return_files_in_folder`.
   - Authentication: **Predefined Credential Type** → **Google Drive OAuth2**.
   - URL (expression):
     - `https://www.googleapis.com/drive/v3/files?q="{{ $json.id }}"+in+parents&fields=files(id,name,mimeType,modifiedTime,size)`
   - Connect: `on_subfolder_created` → `return_files_in_folder`.

3. **Split the returned files array**
   - Add **Split Out** node `split_files_item`.
   - Field to split out: `files`.
   - Connect: `return_files_in_folder` → `split_files_item`.

4. **Filter only Markdown files**
   - Add **Filter** node `select_markdown_files`.
   - Condition: `{{$json.mimeType}}` equals `text/markdown`.
   - Connect: `split_files_item` → `select_markdown_files`.

5. **Download each Markdown file**
   - Add **Google Drive** node `download_drive_files`.
   - Operation: **Download**.
   - File ID: expression `{{$json.id}}`.
   - Credentials: Google Drive OAuth2.
   - Connect: `select_markdown_files` → `download_drive_files`.

6. **Extract text from the downloaded file**
   - Add **Extract From File** node `text_from_markdown`.
   - Operation: **Text**.
   - Connect: `download_drive_files` → `text_from_markdown`.

7. **Aggregate all extracted text**
   - Add **Aggregate** node `combine_all_text`.
   - Aggregate field: `data`.
   - Connect: `text_from_markdown` → `combine_all_text`.

8. **Chunk the combined text**
   - Add **Code** node `chunk_text_for_local_llm`.
   - Language: **Python**.
   - Use chunking code (key parameter is `text_chunk_size = 250`).
   - Connect: `combine_all_text` → `chunk_text_for_local_llm`.

9. **Add Ollama model connector**
   - Add **Ollama Model** node named `Ollama Model`.
   - Model: `llama3.1:latest` (ensure it is pulled in Ollama).
   - Configure **Ollama API credentials** (base URL to your local Ollama, e.g. `http://localhost:11434` depending on your setup).

10. **Create the LLM chain**
   - Add **Basic LLM Chain** node.
   - Prompt: paste the sanitization prompt and include `{{ $json.data }}` as input text.
   - Connect the **AI Language Model** connection from `Ollama Model` → `Basic LLM Chain`.
   - Connect: `chunk_text_for_local_llm` → `Basic LLM Chain`.

11. **Parse strict JSON output**
   - Add **Set** node `parse_json_text_with_flag`.
   - Add a new field `output` (type: object) with expression:
     - `{{ $json.text.replaceAll("\r","").replaceAll("\n","").replaceAll("```","").toJsonString().parseJson() }}`
   - Enable “Ignore conversion errors” (as in workflow).
   - Connect: `Basic LLM Chain` → `parse_json_text_with_flag`.

12. **(Optional) Log each chunk to Google Sheets**
   - Add **Google Sheets** node `log_text_with_pii_flag (optional)`.
   - Operation: **Append**.
   - Select Spreadsheet and Sheet.
   - Map columns:
     - `cleaned text` → `{{$json.output.sanitized_text}}`
     - `pii found` → `{{$json.output.pii_found}}`
     - `created_at` → `{{$now.toISO()}}`
   - Connect: `parse_json_text_with_flag` → `log_text_with_pii_flag (optional)`.

13. **Aggregate sanitized chunks**
   - Add **Aggregate** node `combine_chunk_text`.
   - Aggregate field: `output.sanitized_text`.
   - Connect: `parse_json_text_with_flag` → `combine_chunk_text`.

14. **Join chunks into one text**
   - Add **Set** node `join_text_chunks`.
   - Set field `sanitized_text[0]` to: `{{$json.sanitized_text.join()}}`  
     (If you want no commas, use `join('')` or `join('\n')` instead.)
   - Connect: `combine_chunk_text` → `join_text_chunks`.

15. **Create cleaned Markdown file in Google Drive**
   - Add **Google Drive** node `create_cleaned_text_file`.
   - Operation: **Create From Text**.
   - Name: `{{$now.toISO()}}_cleaned_text.md` (recommended: remove trailing space).
   - Content: `{{$json.sanitized_text[0]}}`
   - Folder: select target folder.
     - If you truly want “same new subfolder”, set folderId dynamically from the trigger folder id (requires carrying that id through the flow).
   - Connect: `join_text_chunks` → `create_cleaned_text_file`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Drive OAuth requirements (enable APIs) | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/#enable-apis |
| Workflow intent: sanitize markdown text from Google Drive with local Ollama; redact PII; log chunks to Sheets | Sticky note description in the workflow |
| Customisation ideas: review logged chunks to expand PII definitions; adjust filter to support other MIME types; change `text_chunk_size` | Sticky note description + code node note (“Change `text_chunk_size`…”) |
| Important implementation detail: current file creation folder is hard-coded (`folder_id`), not automatically “the new subfolder” | Observed in `create_cleaned_text_file` configuration |
| Important implementation detail: `.join()` inserts commas by default when joining chunks | Observed in `join_text_chunks` expression |