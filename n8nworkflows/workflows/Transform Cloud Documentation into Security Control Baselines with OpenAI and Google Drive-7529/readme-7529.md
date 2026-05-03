Transform Cloud Documentation into Security Control Baselines with OpenAI and Google Drive

https://n8nworkflows.xyz/workflows/transform-cloud-documentation-into-security-control-baselines-with-openai-and-google-drive-7529


# Transform Cloud Documentation into Security Control Baselines with OpenAI and Google Drive

### 1. Workflow Overview

This workflow automates transforming cloud provider documentation URLs into structured, auditable security control baselines using AI and Google Drive integration. The key use case is to ingest multiple documentation web pages, extract security-related controls via OpenAI assistants, compose them coherently, and produce consolidated baseline documents stored and managed on Google Drive.

The workflow logic is divided into the following blocks:

- **1.1 Input Reception & Validation:** Accepts POST requests with cloud provider, technology, and URLs; validates mandatory fields and generates a unique identifier.

- **1.2 Google Drive Folder Resolution & Assistant Identification:** Locates or creates a target folder in Google Drive and detects OpenAI assistants by name to use in the AI pipeline.

- **1.3 URL Processing & Content Sanitization:** Splits the URLs, downloads their content, sanitizes the HTML to remove scripts/styles/headers, and prepares clean textual data for AI.

- **1.4 AI Extraction & Composition Pipeline:** Sequentially runs OpenAI assistants to extract controls from sanitized text, compose them, and build a final security baseline.

- **1.5 File Handling in Google Drive:** Searches for existing baseline files, decides whether to append or create new files, merges data if necessary, uploads or updates files in Google Drive.

- **1.6 Response Handling & Error Management:** Checks for cases when no controls are found, and responds with appropriate JSON messages or downloadable baseline files.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block receives incoming POST requests at the `/create` webhook endpoint, authenticates via Basic Auth, validates mandatory fields (`cloudProvider`, `technology`, `urls`), and generates a short unique identifier (UUID).

- **Nodes Involved:**  
  - `create` (Webhook)  
  - `check_mandatory_fields` (IF)  
  - `generate_uuid` (Code)  
  - `No Operation, do nothing` (NoOp)

- **Node Details:**

  - **create**  
    - Type: Webhook (HTTP POST)  
    - Config: HTTP POST on `/create` path, Basic Auth enabled with stored credential.  
    - Inputs: External HTTP request  
    - Outputs: JSON body forwarded downstream  
    - Potential Failures: Auth errors if credentials invalid; missing request body  
    - Notes: Entry point of the workflow.

  - **check_mandatory_fields**  
    - Type: IF node (conditional check)  
    - Config: Checks non-empty values for `cloudProvider`, `technology`, and `urls` in request body.  
    - Inputs: From `create` node  
    - Outputs: Passes to `generate_uuid` if valid; else `No Operation`.  
    - Edge Cases: Missing or empty fields cause no further processing.

  - **generate_uuid**  
    - Type: Code node (JavaScript)  
    - Config: Generates a 12-character random alphanumeric string used as UUID.  
    - Inputs: From `check_mandatory_fields` (valid branch)  
    - Outputs: JSON with `uuid` key  
    - Edge Cases: Random generation failure unlikely; no external dependencies.

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Config: Does nothing, used as a sink for invalid input branch.  
    - Inputs: From `check_mandatory_fields` (invalid branch)  
    - Outputs: None; effectively stops processing.

---

#### 2.2 Google Drive Folder Resolution & Assistant Identification

- **Overview:**  
  Using the generated UUID, this block resolves the target Google Drive folder (default or overridden), lists OpenAI assistants, and selects specific assistants by heuristics for extraction, composition, and baseline building.

- **Nodes Involved:**  
  - `get_gdrive_id` (Google Drive)  
  - `OpenAI_Assistants_List` (OpenAI List Assistants)  
  - `resolve_assistants` (Code)  
  - `settings` (Set)

- **Node Details:**

  - **get_gdrive_id**  
    - Type: Google Drive (search files/folders)  
    - Config: Searches for folder named `n8n_defysec` in root drive.  
    - Inputs: From `generate_uuid`  
    - Outputs: Folder metadata including ID  
    - Edge Cases: Folder not found; should be created or error handled externally.

  - **OpenAI_Assistants_List**  
    - Type: OpenAI node (list assistants)  
    - Config: Retrieves AI assistants list from the OpenAI account using stored credentials.  
    - Inputs: From `get_gdrive_id`  
    - Outputs: List of assistant objects with IDs and names.

  - **resolve_assistants**  
    - Type: Code node  
    - Config: Parses the assistants list, heuristically selects three assistants by name patterns:  
      - Extractor assistant (e.g., `1_DefySec_Extractor`)  
      - Composer assistant (e.g., `2_DefySec_Control_Composer`)  
      - Baseline builder (e.g., `3_DefySec Baseline Builder`)  
    - Outputs: JSON with IDs for each assistant  
    - Edge Cases: Missing assistants result in empty IDs; may affect downstream AI calls.

  - **settings**  
    - Type: Set node  
    - Config: Consolidates key workflow variables: UUID, cloud provider, technology, URLs array, Google Drive folder ID, assistant IDs (from previous node or POST overrides).  
    - Inputs: From `resolve_assistants` and `create` node (for POST body)  
    - Outputs: JSON with all context variables for downstream nodes.

---

#### 2.3 URL Processing & Content Sanitization

- **Overview:**  
  This block explodes the array of URLs into individual items, fetches each URL's HTML content, sanitizes it by removing scripts, styles, comments, headers, and then prepares clean text for AI processing.

- **Nodes Involved:**  
  - `explode_urls` (Code)  
  - `process_url` (Split In Batches)  
  - `http_get_url` (HTTP Request)  
  - `html_sanitizer` (Code)  
  - `1_DefySec_Extractor` (OpenAI)

- **Node Details:**

  - **explode_urls**  
    - Type: Code node  
    - Config: Reads `urls` array from `settings` and emits one item per URL, preserving UUID, cloud provider, and technology.  
    - Inputs: From `settings`  
    - Outputs: Items with single URL each.

  - **process_url**  
    - Type: SplitInBatches  
    - Config: Processes URLs in batches (default size unspecified, likely 1) to avoid overloading requests.  
    - Inputs: From `explode_urls`  
    - Outputs: Sequential URL items for HTTP fetching.

  - **http_get_url**  
    - Type: HTTP Request  
    - Config: GET request to the URL with 10-second timeout, allows unauthorized SSL certs, sends `User-Agent` header mimicking Chrome browser.  
    - Inputs: From `process_url`  
    - Outputs: HTML content in `data` or `body` field.  
    - Edge Cases: Timeout, connection errors, invalid SSL, non-HTML responses.

  - **html_sanitizer**  
    - Type: Code node  
    - Config: Removes `<script>`, `<style>`, comments, and non-content tags, strips all HTML tags, normalizes whitespace and newlines, returns sanitized text plus URL metadata.  
    - Inputs: From `http_get_url` and `process_url` (for metadata)  
    - Outputs: JSON with sanitized text and metadata.

  - **1_DefySec_Extractor**  
    - Type: OpenAI Assistant  
    - Config: Uses assistant ID from `settings` to extract relevant security controls from sanitized text. Input prompt includes cloud provider, technology, URL, and sanitized text.  
    - Inputs: From `html_sanitizer`  
    - Outputs: Extracted controls text in assistant response.  
    - Edge Cases: API errors, rate limits, malformed input.

---

#### 2.4 AI Extraction & Composition Pipeline

- **Overview:**  
  This block composes and consolidates extracted controls. It verifies extraction success, routes accordingly, uses a second OpenAI assistant to compose controls, and a third assistant to build the final baseline.

- **Nodes Involved:**  
  - `ec_controls_check` (IF)  
  - `ec_search_files` (Google Drive)  
  - `ec_extract_file_info` (Code)  
  - `ec_append_create_filter` (IF)  
  - `ec_download_existing_file` (Google Drive)  
  - `ec_upload_new_file` (Google Drive)  
  - `ec_update_existing_file` (Google Drive)  
  - `ec_merge_data` (Code)  
  - `2_DefySec_Control_Composer` (OpenAI)  
  - `cc_search_files` (Google Drive)  
  - `cc_extract_file_info` (Code)  
  - `cc_controls_router` (Switch)  
  - `cc_controls_check` (IF)  
  - `3_DefySec Baseline Builder` (OpenAI)  
  - `cc_no_controls_answer` (RespondToWebhook)  
  - `cc_controls_check` (Code)  

- **Node Details:**

  - **ec_controls_check**  
    - Type: IF node  
    - Config: Checks if extractor output is different from `"NO_CONTROLS_FOUND"`.  
    - Inputs: From `1_DefySec_Extractor`  
    - Outputs: If controls found, continues to search for existing files; else retries URLs.  
    - Edge Cases: Missing or malformed output.

  - **ec_search_files**  
    - Type: Google Drive (list files)  
    - Config: Search for files matching `${uuid}_extractedControls_` prefix in target folder.  
    - Inputs: From `ec_controls_check` true branch  
    - Outputs: List of matching files.

  - **ec_extract_file_info**  
    - Type: Code node  
    - Config: Decides whether to append or create new extracted controls file based on search results. Prepares metadata and optionally encodes new control text in base64 for upload.  
    - Inputs: From `ec_search_files`  
    - Outputs: JSON with `action` (`append` or `create`), fileId if existing, fileName, folderId, and newText.

  - **ec_append_create_filter**  
    - Type: IF node  
    - Config: Checks if action is `"append"` (update existing file) or else `"create"` (upload new file).  
    - Inputs: From `ec_extract_file_info`  
    - Outputs: Append branch to download existing file; create branch to upload new file.

  - **ec_download_existing_file**  
    - Type: Google Drive (download file)  
    - Config: Downloads existing file content for appending.  
    - Inputs: From `ec_append_create_filter` append branch  
    - Outputs: Binary file content.

  - **ec_upload_new_file**  
    - Type: Google Drive (upload file)  
    - Config: Uploads new extracted controls file to target folder.  
    - Inputs: From `ec_append_create_filter` create branch  
    - Outputs: Metadata of uploaded file.

  - **ec_merge_data**  
    - Type: Code node  
    - Config: Reads existing file content (from download or item), appends new extracted text with spacing, encodes merged content to base64 for update.  
    - Inputs: From `ec_download_existing_file`  
    - Outputs: Prepared binary data to update file.

  - **ec_update_existing_file**  
    - Type: Google Drive (update file)  
    - Config: Updates existing file in Drive with merged content binary.  
    - Inputs: From `ec_merge_data`  
    - Outputs: Metadata of updated file.

  - **2_DefySec_Control_Composer**  
    - Type: OpenAI Assistant  
    - Config: Uses composer assistant ID to synthesize extracted controls, input includes cloud provider, technology, and current file content.  
    - Inputs: From `cc_extract_file_info`  
    - Outputs: Composed control text or `"NO_CONTROLS_FOUND"`.

  - **cc_search_files**  
    - Type: Google Drive (search files)  
    - Config: Searches for `${uuid}_extractedControls.txt` file in target folder as input for composition.  
    - Inputs: From `process_url` (batch iteration)  
    - Outputs: Files matching naming pattern.

  - **cc_extract_file_info**  
    - Type: Code node  
    - Config: Reads file content from Google Drive download or current item, normalizes text, prepares metadata for AI composer.  
    - Inputs: From `cc_search_files`  
    - Outputs: JSON with file content and metadata.

  - **cc_controls_router**  
    - Type: Switch node  
    - Config: Routes based on composer output: if `"NO_CONTROLS_FOUND"`, sends to no-controls response; else proceeds to baseline builder.  
    - Inputs: From `2_DefySec_Control_Composer`  
    - Outputs: Two branches: no controls or continue.

  - **cc_controls_check**  
    - Type: Code node  
    - Config: Detects if the output from baseline builder is `"NO_CONTROLS_TO_BE_CONSOLIDATED"` and replaces payload with extracted file info for fallback.  
    - Inputs: From `3_DefySec Baseline Builder`  
    - Outputs: Normalized items for next steps.

  - **3_DefySec Baseline Builder**  
    - Type: OpenAI Assistant  
    - Config: Uses baseline builder assistant ID to create final baseline from composed controls, input includes cloud provider, technology, and composed controls content.  
    - Inputs: From `cc_controls_check`  
    - Outputs: Final baseline text or special `"NO_CONTROLS_TO_BE_CONSOLIDATED"`.

  - **cc_no_controls_answer**  
    - Type: RespondToWebhook  
    - Config: Returns JSON response indicating no valid controls found with guidance on input formatting.  
    - Inputs: From `cc_controls_router` no-controls branch  
    - Outputs: HTTP JSON response.

---

#### 2.5 File Handling in Google Drive & Final Response

- **Overview:**  
  Processes final baseline data into downloadable text files, checks for valid output, and sends responses either as downloadable files or error JSON.

- **Nodes Involved:**  
  - `bb_controls_check` (IF)  
  - `bb_data_prep` (Code)  
  - `bb_no_controls_answer` (RespondToWebhook)  
  - `bb_data_respond` (RespondToWebhook)

- **Node Details:**

  - **bb_controls_check**  
    - Type: IF node  
    - Config: Checks if baseline builder output is not `"NO_CONTROLS_FOUND"`.  
    - Inputs: From `3_DefySec Baseline Builder`  
    - Outputs: Passes valid output to data prep or no-controls response.

  - **bb_data_prep**  
    - Type: Code node  
    - Config: Consolidates baseline outputs, strips code fences, detects technology for filename, encodes text to base64, prepares binary data for response with appropriate headers.  
    - Inputs: From `bb_controls_check` true branch  
    - Outputs: Binary file data and metadata for download.

  - **bb_no_controls_answer**  
    - Type: RespondToWebhook  
    - Config: Returns JSON error message with instructions if no controls found.  
    - Inputs: From `bb_controls_check` false branch  
    - Outputs: HTTP JSON response.

  - **bb_data_respond**  
    - Type: RespondToWebhook  
    - Config: Responds with binary data as file attachment with `Content-Disposition` header for download.  
    - Inputs: From `bb_data_prep`  
    - Outputs: HTTP response with downloadable `.txt` file.

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                          | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                                                   |
|---------------------------|----------------------------|----------------------------------------|----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| create                    | Webhook                    | Receive POST request with Basic Auth   | —                                | check_mandatory_fields             |                                                                                                                              |
| check_mandatory_fields    | IF                         | Validate mandatory input fields         | create                           | generate_uuid, No Operation, do nothing |                                                                                                                              |
| generate_uuid             | Code                       | Generate short UUID for request         | check_mandatory_fields           | get_gdrive_id                     |                                                                                                                              |
| No Operation, do nothing  | NoOp                       | Halt processing on invalid input        | check_mandatory_fields           | —                                 |                                                                                                                              |
| get_gdrive_id             | Google Drive               | Resolve target folder in Drive           | generate_uuid                   | OpenAI_Assistants_List            |                                                                                                                              |
| OpenAI_Assistants_List    | OpenAI (List Assistants)   | Retrieve available OpenAI assistants     | get_gdrive_id                   | resolve_assistants                |                                                                                                                              |
| resolve_assistants        | Code                       | Select assistants by name heuristics     | OpenAI_Assistants_List          | settings                         |                                                                                                                              |
| settings                  | Set                        | Consolidate config variables              | resolve_assistants, create      | explode_urls                     |                                                                                                                              |
| explode_urls              | Code                       | Split URLs array into individual items   | settings                       | process_url                     |                                                                                                                              |
| process_url               | SplitInBatches             | Process URLs one by one                   | explode_urls                   | cc_search_files (branch1), http_get_url (branch2) |                                                                                                                              |
| http_get_url              | HTTP Request               | Download URL content                      | process_url                    | html_sanitizer                   |                                                                                                                              |
| html_sanitizer            | Code                       | Sanitize HTML to plain text               | http_get_url                   | 1_DefySec_Extractor             |                                                                                                                              |
| 1_DefySec_Extractor       | OpenAI Assistant           | Extract security controls from text       | html_sanitizer                 | ec_controls_check               |                                                                                                                              |
| ec_controls_check         | IF                         | Check if extraction found controls        | 1_DefySec_Extractor            | ec_search_files (true), process_url (false) |                                                                                                                              |
| ec_search_files           | Google Drive               | Search for existing extracted controls file | ec_controls_check (true)       | ec_extract_file_info            |                                                                                                                              |
| ec_extract_file_info      | Code                       | Decide append or create new file           | ec_search_files               | ec_append_create_filter         |                                                                                                                              |
| ec_append_create_filter   | IF                         | Branch on append vs create action          | ec_extract_file_info           | ec_download_existing_file (append), ec_upload_new_file (create) |                                                                                                                              |
| ec_download_existing_file | Google Drive               | Download existing controls file for append | ec_append_create_filter (append) | ec_merge_data                  |                                                                                                                              |
| ec_upload_new_file        | Google Drive               | Upload new extracted controls file         | ec_append_create_filter (create) | process_url                   |                                                                                                                              |
| ec_merge_data             | Code                       | Merge existing file content with new extract | ec_download_existing_file       | ec_update_existing_file         |                                                                                                                              |
| ec_update_existing_file   | Google Drive               | Update existing controls file in Drive      | ec_merge_data                 | process_url                    |                                                                                                                              |
| cc_search_files           | Google Drive               | Search for extracted controls text file     | process_url                   | cc_extract_file_info           |                                                                                                                              |
| cc_extract_file_info      | Code                       | Read and normalize extracted controls file content | cc_search_files               | 2_DefySec_Control_Composer     |                                                                                                                              |
| 2_DefySec_Control_Composer| OpenAI Assistant           | Compose extracted controls into structured content | cc_extract_file_info          | cc_controls_router             |                                                                                                                              |
| cc_controls_router        | Switch                     | Route on control composition result          | 2_DefySec_Control_Composer    | cc_no_controls_answer (no controls), cc_controls_check (controls found) |                                                                                                                              |
| cc_no_controls_answer     | RespondToWebhook           | Respond with JSON if no controls found        | cc_controls_router            | —                             |                                                                                                                              |
| cc_controls_check         | Code                       | Detect special no consolidation output         | 3_DefySec Baseline Builder    | 3_DefySec Baseline Builder (altered) |                                                                                                                              |
| 3_DefySec Baseline Builder| OpenAI Assistant           | Build final security baseline from composed controls | cc_controls_check             | bb_controls_check              |                                                                                                                              |
| bb_controls_check         | IF                         | Check if baseline has valid controls           | 3_DefySec Baseline Builder    | bb_data_prep (valid), bb_no_controls_answer (invalid) |                                                                                                                              |
| bb_data_prep              | Code                       | Prepare binary file for response download       | bb_controls_check            | bb_data_respond               |                                                                                                                              |
| bb_no_controls_answer     | RespondToWebhook           | Respond with JSON if no valid baseline            | bb_controls_check            | —                             |                                                                                                                              |
| bb_data_respond           | RespondToWebhook           | Respond with downloadable `.txt` file            | bb_data_prep                | —                             |                                                                                                                              |
| Sticky Note               | Sticky Note                | Overview and workflow summary                    | —                            | —                             | This template turns provider docs (URLs) into an **auditable security baseline**: POST /create, Google Drive integration, AI pipeline. |
| Sticky Note1              | Sticky Note                | Setup & Credentials instructions                 | —                            | —                             | OpenAI, Google Drive OAuth2, Basic Auth details, assistant names, folder resolution explained.                                 |
| Sticky Note2              | Sticky Note                | Run & Troubleshooting tips                         | —                            | —                             | Test instructions, Drive search notes, security recommendations.                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("create")**  
   - HTTP Method: POST  
   - Path: `create`  
   - Authentication: Basic Auth (configure credential with username/password)  
   - Response Mode: Response Node

2. **Add IF Node ("check_mandatory_fields")**  
   - Condition: Check that `body.cloudProvider`, `body.technology`, and `body.urls` are not empty strings  
   - True output: Continue; False output: Stop (No Operation node)

3. **Add Code Node ("generate_uuid")**  
   - JavaScript: Generate 12-character random alphanumeric string as `uuid`  
   - Connect from True branch of IF

4. **Add Google Drive Node ("get_gdrive_id")**  
   - Operation: Search for folder named `n8n_defysec` in Drive root  
   - Connect from `generate_uuid`

5. **Add OpenAI Node ("OpenAI_Assistants_List")**  
   - Operation: List assistants  
   - Credential: OpenAI API key  
   - Connect from `get_gdrive_id`

6. **Add Code Node ("resolve_assistants")**  
   - Parse assistant list; select IDs for extractor, composer, baseline assistants by name regex  
   - Connect from `OpenAI_Assistants_List`

7. **Add Set Node ("settings")**  
   - Assign variables: `uuid` from `generate_uuid`, `cloudprovider`, `technology`, `urls` from webhook body, `gdrive_target` from `get_gdrive_id` output, assistant IDs from `resolve_assistants` or webhook overrides  
   - Connect from both `resolve_assistants` and `create` (merge inputs if needed)

8. **Add Code Node ("explode_urls")**  
   - Split `urls` array into individual items, each with metadata (`uuid`, `cloudProvider`, `technology`)  
   - Connect from `settings`

9. **Add SplitInBatches Node ("process_url")**  
   - Batch size default (e.g., 1) to process URLs serially  
   - Connect from `explode_urls`

10. **Add HTTP Request Node ("http_get_url")**  
    - GET request to URL from input item  
    - Timeout: 10s, Allow unauthorized SSL certs, Header User-Agent set as Chrome  
    - Connect from `process_url`

11. **Add Code Node ("html_sanitizer")**  
    - Remove scripts, styles, comments, headers, and HTML tags from response body, normalize whitespace  
    - Connect from `http_get_url`

12. **Add OpenAI Node ("1_DefySec_Extractor")**  
    - Use assistant ID (extracted from `settings`)  
    - Prompt includes cloudProvider, technology, URL, sanitized text  
    - Connect from `html_sanitizer`

13. **Add IF Node ("ec_controls_check")**  
    - Check that extractor output != `"NO_CONTROLS_FOUND"`  
    - True branch: continue; False branch: retry or halt

14. **Add Google Drive Node ("ec_search_files")**  
    - Search files with prefix `${uuid}_extractedControls_` in target folder  
    - Connect from `ec_controls_check` true branch

15. **Add Code Node ("ec_extract_file_info")**  
    - Decide append or create file based on search results  
    - Prepare metadata and base64-encoded new text if creating new  
    - Connect from `ec_search_files`

16. **Add IF Node ("ec_append_create_filter")**  
    - Branch on `action` = `"append"` or `"create"`  
    - Append branch: download existing file  
    - Create branch: upload new file

17. **Add Google Drive Node ("ec_download_existing_file")**  
    - Download existing file content  
    - Connect from append branch

18. **Add Google Drive Node ("ec_upload_new_file")**  
    - Upload new extracted controls file to Drive folder  
    - Connect from create branch

19. **Add Code Node ("ec_merge_data")**  
    - Merge existing and new extracted controls text  
    - Encode merged content for update  
    - Connect from `ec_download_existing_file`

20. **Add Google Drive Node ("ec_update_existing_file")**  
    - Update existing file with merged content  
    - Connect from `ec_merge_data`

21. **Add Google Drive Node ("cc_search_files")**  
    - Search for `${uuid}_extractedControls.txt` file in target folder  
    - Connect from `process_url` (batch continuation)

22. **Add Code Node ("cc_extract_file_info")**  
    - Read and normalize extracted controls content from Drive or current item  
    - Connect from `cc_search_files`

23. **Add OpenAI Node ("2_DefySec_Control_Composer")**  
    - Use composer assistant ID  
    - Input: cloud provider, technology, extracted controls content  
    - Connect from `cc_extract_file_info`

24. **Add Switch Node ("cc_controls_router")**  
    - Route on composer output: if `"NO_CONTROLS_FOUND"` → respond no controls; else → continue

25. **Add RespondToWebhook Node ("cc_no_controls_answer")**  
    - Respond JSON indicating no controls found with instructions  
    - Connect from `cc_controls_router` no controls branch

26. **Add Code Node ("cc_controls_check")**  
    - Detect `"NO_CONTROLS_TO_BE_CONSOLIDATED"` output from baseline builder; replace payload with extracted file info if needed  
    - Connect from `cc_controls_router` controls branch

27. **Add OpenAI Node ("3_DefySec Baseline Builder")**  
    - Use baseline assistant ID  
    - Input: cloud provider, technology, composed controls text  
    - Connect from `cc_controls_check`

28. **Add IF Node ("bb_controls_check")**  
    - Check baseline builder output not `"NO_CONTROLS_FOUND"`  
    - True → data prep; False → no controls response

29. **Add Code Node ("bb_data_prep")**  
    - Prepare binary `.txt` file from baseline output, strip fences, detect tech for filename, encode base64  
    - Connect from `bb_controls_check` true branch

30. **Add RespondToWebhook Node ("bb_data_respond")**  
    - Respond with binary file attachment with proper `Content-Disposition` header for download  
    - Connect from `bb_data_prep`

31. **Add RespondToWebhook Node ("bb_no_controls_answer")**  
    - Respond JSON error message if no valid baseline  
    - Connect from `bb_controls_check` false branch

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This template turns provider docs (URLs) into an **auditable security baseline** with AI and Google Drive.     | Overview sticky note in workflow                                                                                                    |
| Setup requires OpenAI API key, Google Drive OAuth2 for file read/write, and Basic Auth credential for `/create`. | Credentials sticky note                                                                                                              |
| Google Drive folder `n8n_defysec` is auto-resolved in root; can be overridden via POST with `gdriveTargetId`.  | Folder resolution detail in sticky note                                                                                              |
| Three OpenAI assistants are resolved dynamically by name heuristics: Extractor, Composer, Baseline Builder.    | Assistants resolution sticky note                                                                                                   |
| Test the workflow by POSTing JSON with `cloudProvider`, `technology`, and `urls` array to `/create` endpoint.  | Run & Troubleshooting sticky note                                                                                                    |
| If "NO_CONTROLS_FOUND" is returned, ensure input pages follow expected content structure and format.           | Troubleshooting instructions                                                                                                        |
| All credentials are managed securely within n8n's credential manager; no API keys are hardcoded in HTTP headers. | Security best practices note                                                                                                        |

---

This document fully describes the workflow, enabling reproduction, modification, and troubleshooting. The workflow is designed to integrate AI-powered semantic extraction with cloud documentation ingestion and Google Drive file management, delivering structured security baselines efficiently.