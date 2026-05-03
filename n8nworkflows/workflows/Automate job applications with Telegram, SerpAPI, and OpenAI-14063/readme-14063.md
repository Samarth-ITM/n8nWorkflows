Automate job applications with Telegram, SerpAPI, and OpenAI

https://n8nworkflows.xyz/workflows/automate-job-applications-with-telegram--serpapi--and-openai-14063


# Automate job applications with Telegram, SerpAPI, and OpenAI

# 1. Workflow Overview

This workflow automates a job-application preparation pipeline driven from Telegram. A user sends either:

- a **job posting URL**, or
- a **job search keyword**

The workflow then follows one of two paths:

- **URL mode:** uses an OpenAI-powered AI Agent with web search to extract job data from a single posting.
- **Keyword mode:** uses SerpAPI Google Jobs search, filters unwanted roles, and asks the user in Telegram to approve individual jobs.

For approved or successfully extracted jobs, the workflow:

- records the job in Google Sheets,
- generates a cover letter PDF,
- generates a tailored CV PDF,
- uploads both to Google Drive,
- creates a per-application folder,
- stores the Drive folder link in Google Sheets,
- sends completion feedback in Telegram.

The workflow has **one entry point** but **two logical processing branches**.

## 1.1 Telegram Input Reception and Routing

The workflow starts with a Telegram Trigger. A routing IF node checks whether the incoming Telegram message contains `http`. If yes, it assumes the user sent a URL; otherwise it treats the message as a keyword search query.

## 1.2 URL Mode: AI Extraction from a Job Posting

If the input looks like a URL, an AI Agent uses OpenAI with web search to inspect the target page and return a structured JSON payload containing job metadata. A structured parser and auto-fixing parser enforce schema compliance. A second IF node checks whether extraction succeeded; if not, the user is informed about the failure.

## 1.3 URL Mode: Document Generation, Drive Upload, and Logging

If the AI extraction succeeds, the workflow immediately generates a cover letter PDF and a tailored CV PDF, uploads them to Google Drive, creates an application folder, moves both files into that folder, appends a row to Google Sheets, and notifies the user.

## 1.4 Keyword Mode: Google Jobs Search and Result Normalization

If the Telegram message is not a URL, the workflow queries Google Jobs through SerpAPI using the message text. A Code node flattens and normalizes the SerpAPI response into one item per job result.

## 1.5 Keyword Mode: Filtering and Telegram Approval Loop

The normalized job results are filtered to exclude unwanted categories such as student roles, internships, freelance, temporary, part-time, and a configurable exclusion keyword. Remaining jobs are processed one by one in a loop. Each job is sent to Telegram as a formatted message with approval buttons.

## 1.6 Approved Keyword Results: Logging, Document Generation, Drive Upload, and Final Update

Approved jobs are written into Google Sheets, then used to generate the same two PDF documents as in URL mode. The PDFs are uploaded to Google Drive, organized into a folder, and the corresponding Google Sheets row is updated with the Drive folder link. The user then receives a completion message.

---

# 2. Block-by-Block Analysis

## Block 1 — Telegram Input Reception and Routing

### Overview
This block receives user input from Telegram and decides whether the message should be handled as a direct job-posting URL or as a Google Jobs search query. It is the single entry point for the whole workflow.

### Nodes Involved
- Telegram Trigger
- If

### Node Details

#### 1. Telegram Trigger
- **Type and role:** `n8n-nodes-base.telegramTrigger`; event source for Telegram messages.
- **Configuration choices:**  
  - Listens to `message` updates only.
  - Restricts allowed sender(s) using `additionalFields.userIds = YOUR_TELEGRAM_USER_ID`.
- **Key expressions or variables used:**  
  - Downstream nodes use `$('Telegram Trigger').item.json.message.chat.id`
  - Input text comes from `$json.message.text`
- **Input and output connections:**  
  - No input; entry node.
  - Outputs to **If**.
- **Version-specific requirements:** `typeVersion 1.2`
- **Edge cases / failures:**  
  - Invalid Telegram credentials
  - Bot not started by user
  - Wrong user ID filter causing no executions
  - Non-text Telegram messages may not provide `message.text`
- **Sub-workflow reference:** None

#### 2. If
- **Type and role:** `n8n-nodes-base.if`; branch router.
- **Configuration choices:**  
  - Checks whether the incoming message text contains `http`.
  - True branch = URL mode
  - False branch = keyword search mode
- **Key expressions or variables used:**  
  - `={{ $json.message.text }}`
- **Input and output connections:**  
  - Input: **Telegram Trigger**
  - True output: **AI Agent**
  - False output: **Search google jobs listings with SERP-API**
- **Version-specific requirements:** `typeVersion 2.3`
- **Edge cases / failures:**  
  - False positives if a non-job message contains `http`
  - False negatives if the URL is malformed or omitted protocol
  - Empty text message may break routing intent
- **Sub-workflow reference:** None

---

## Block 2 — URL Mode: AI Extraction and Validation

### Overview
This block tries to extract job details from a single job posting URL using an AI Agent powered by OpenAI web search. The returned output is forced into a strict JSON schema and then validated for extraction success.

### Nodes Involved
- AI Agent
- OpenAI Chat Model
- Structured Output Parser
- Auto-fixing Output Parser
- OpenAI Chat Model1
- If1
- Notify user about error

### Node Details

#### 3. AI Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LLM agent orchestrating prompt execution and web search-based extraction.
- **Configuration choices:**  
  - Prompt is fully defined in the node.
  - Instructs the model to:
    - open the URL from Telegram,
    - detect inaccessible/expired/captcha/login-protected pages,
    - extract job posting fields,
    - classify language as `de`, `en`, or `other`,
    - return only raw JSON.
  - Has output parser enabled.
- **Key expressions or variables used:**  
  - Input URL: `{{ $('Telegram Trigger').item.json.message.text }}`
  - Downstream access pattern: `$('AI Agent').item.json.output.*`
- **Input and output connections:**  
  - Main input: **If** (URL branch)
  - AI language model input: **OpenAI Chat Model**
  - AI output parser input: **Auto-fixing Output Parser**
  - Main output: **If1**
- **Version-specific requirements:** `typeVersion 3.1`
- **Edge cases / failures:**  
  - Model cannot access the page meaningfully
  - Website blocking search/browser access
  - Unexpected page layout
  - Token/context pressure on very long pages
  - If model output diverges from schema, parser may still fail
- **Sub-workflow reference:** None

#### 4. OpenAI Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; primary LLM for the AI agent.
- **Configuration choices:**  
  - Model: `gpt-5.1`
  - Built-in tool: web search enabled
  - Search context size: `medium`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connected as AI language model to **AI Agent**
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases / failures:**  
  - Missing or invalid OpenAI credential
  - Model availability mismatch in account/region
  - Rate limits or quota exhaustion
  - Web search limitations on some domains
- **Sub-workflow reference:** None

#### 5. Structured Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; schema validator for AI output.
- **Configuration choices:**  
  - Manual JSON schema
  - Requires:
    - `extraction_successful`
    - `position`
    - `platform`
    - `link`
    - `location`
    - `posted_at`
    - `salary`
    - `share_link`
    - `description`
    - `company`
    - `language`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Output parser connection to **Auto-fixing Output Parser**
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases / failures:**  
  - Schema mismatch
  - Invalid enum for language
  - Model omits required fields
- **Sub-workflow reference:** None

#### 6. Auto-fixing Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserAutofixing`; attempts to repair non-compliant AI output.
- **Configuration choices:**  
  - Uses a secondary model to correct schema deviations automatically.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - AI language model input: **OpenAI Chat Model1**
  - AI output parser input: **Structured Output Parser**
  - Output parser connection to **AI Agent**
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases / failures:**  
  - If original output is too malformed, repair may fail
  - Added latency and token cost
- **Sub-workflow reference:** None

#### 7. OpenAI Chat Model1
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; repair model for parser auto-fixing.
- **Configuration choices:**  
  - Model: `gpt-5.1`
  - No built-in tools configured
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connected as AI language model to **Auto-fixing Output Parser**
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases / failures:**  
  - Same OpenAI credential/rate limit issues as main model
- **Sub-workflow reference:** None

#### 8. If1
- **Type and role:** `n8n-nodes-base.if`; validates whether extraction succeeded.
- **Configuration choices:**  
  - Condition checks if `$json.output.extraction_successful` is true.
  - True branch proceeds to document generation.
  - False branch notifies user about extraction failure.
- **Key expressions or variables used:**  
  - `={{ $json.output.extraction_successful }}`
- **Input and output connections:**  
  - Input: **AI Agent**
  - True output: **HTML to PDF**
  - False output: **Notify user about error**
- **Version-specific requirements:** `typeVersion 2.3`
- **Edge cases / failures:**  
  - If parser output shape changes unexpectedly, expression may fail
- **Sub-workflow reference:** None

#### 9. Notify user about error
- **Type and role:** `n8n-nodes-base.telegram`; sends fallback message to user.
- **Configuration choices:**  
  - Static text asks user to provide full description and application URL manually.
  - Chat ID derived from Telegram trigger.
- **Key expressions or variables used:**  
  - `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Input and output connections:**  
  - Input: **If1** false branch
  - No output
- **Version-specific requirements:** `typeVersion 1.2`
- **Edge cases / failures:**  
  - Telegram auth problems
  - Invalid or expired chat context
- **Sub-workflow reference:** None

---

## Block 3 — URL Mode: PDF Generation, Drive Upload, and Sheet Logging

### Overview
This block produces the application assets for a successfully extracted job URL. It generates a cover letter and CV as PDFs, uploads them to Google Drive, groups them in a folder, logs the job in Google Sheets, and confirms success to the user.

### Nodes Involved
- HTML to PDF
- Upload file
- Create_folder_for_attachments3
- Move file
- Create_personalized_cv3
- Upload_cv3
- Move_cv_to_attachments_folder3
- Append row in sheet1
- Notify user about successful process

### Node Details

#### 10. HTML to PDF
- **Type and role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf`; generates cover letter PDF from HTML/CSS.
- **Configuration choices:**  
  - Output format: file
  - Uses long embedded CSS/HTML template
  - Injects:
    - `$('AI Agent').item.json.output.company`
    - `$('AI Agent').item.json.output.position`
  - Includes placeholder personal details, links, photo URL, and motivation text
- **Key expressions or variables used:**  
  - `{{ $('AI Agent').item.json.output.company }}`
  - `{{ $('AI Agent').item.json.output.position }}`
- **Input and output connections:**  
  - Input: **If1** true branch
  - Output: **Upload file**
- **Version-specific requirements:** `typeVersion 1`
  - Requires community node `n8n-nodes-htmlcsstopdf`
  - Works on self-hosted n8n only, per note
- **Edge cases / failures:**  
  - Community node not installed
  - External image/font URLs unavailable
  - HTML syntax or expression issues break rendering
  - PDF engine/API auth failure
- **Sub-workflow reference:** None

#### 11. Upload file
- **Type and role:** `n8n-nodes-base.googleDrive`; uploads cover letter PDF to Drive.
- **Configuration choices:**  
  - Upload name: `YourName_CoverLetter_<company>`
  - Destination folder: `YOUR_GOOGLE_DRIVE_FOLDER_ID`
  - Drive: `My Drive`
- **Key expressions or variables used:**  
  - `=YourName_CoverLetter_{{ $('AI Agent').item.json.output.company }}`
- **Input and output connections:**  
  - Input: **HTML to PDF**
  - Output: **Create_folder_for_attachments3**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - Google Drive OAuth failure
  - Missing binary file from previous node
  - Invalid folder ID
  - Filename collisions are possible but usually tolerated
- **Sub-workflow reference:** None

#### 12. Create_folder_for_attachments3
- **Type and role:** `n8n-nodes-base.googleDrive`; creates per-job folder.
- **Configuration choices:**  
  - Resource: folder
  - Folder name: `<position>_<company>`
  - Parent folder: `YOUR_GOOGLE_DRIVE_FOLDER_ID`
- **Key expressions or variables used:**  
  - `={{ $('AI Agent').item.json.output.position }}_{{ $('AI Agent').item.json.output.company }}`
- **Input and output connections:**  
  - Input: **Upload file**
  - Output: **Move file**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - Duplicate folder names
  - Drive permissions issues
- **Sub-workflow reference:** None

#### 13. Move file
- **Type and role:** `n8n-nodes-base.googleDrive`; moves uploaded cover letter into new folder.
- **Configuration choices:**  
  - Operation: move
  - File ID from **Upload file**
  - Folder ID from current item (created folder)
- **Key expressions or variables used:**  
  - `={{ $('Upload file').item.json.id }}`
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input: **Create_folder_for_attachments3**
  - Output: **Create_personalized_cv3**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - File not found
  - Folder creation failed upstream
  - Permission issues
- **Sub-workflow reference:** None

#### 14. Create_personalized_cv3
- **Type and role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf`; generates tailored CV PDF.
- **Configuration choices:**  
  - Output format: file
  - Uses extensive embedded HTML/CSS
  - Dynamic header inserts target position and company
- **Key expressions or variables used:**  
  - `{{ $('AI Agent').item.json.output.position }}`
  - `{{ $('AI Agent').item.json.output.company }}`
- **Input and output connections:**  
  - Input: **Move file**
  - Output: **Upload_cv3**
- **Version-specific requirements:** `typeVersion 1`
  - Same community node requirement as other PDF nodes
- **Edge cases / failures:**  
  - Same rendering/dependency issues as cover letter PDF
- **Sub-workflow reference:** None

#### 15. Upload_cv3
- **Type and role:** `n8n-nodes-base.googleDrive`; uploads CV PDF.
- **Configuration choices:**  
  - Upload name: `YourName_CV_for_<company>`
  - Initial destination: same parent applications folder
- **Key expressions or variables used:**  
  - `=YourName_CV_for_{{ $('AI Agent').item.json.output.company }}`
- **Input and output connections:**  
  - Input: **Create_personalized_cv3**
  - Output: **Move_cv_to_attachments_folder3**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - Missing binary
  - OAuth or folder errors
- **Sub-workflow reference:** None

#### 16. Move_cv_to_attachments_folder3
- **Type and role:** `n8n-nodes-base.googleDrive`; moves CV into the created per-job folder.
- **Configuration choices:**  
  - Operation: move
  - File ID from **Upload_cv3**
  - Folder ID from **Create_folder_for_attachments3**
- **Key expressions or variables used:**  
  - `={{ $('Upload_cv3').item.json.id }}`
  - `={{ $('Create_folder_for_attachments3').item.json.id }}`
- **Input and output connections:**  
  - Input: **Upload_cv3**
  - Output: **Append row in sheet1**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - Folder/file ID mismatch
  - Drive permission issues
- **Sub-workflow reference:** None

#### 17. Append row in sheet1
- **Type and role:** `n8n-nodes-base.googleSheets`; logs extracted URL-mode job to the tracking sheet.
- **Configuration choices:**  
  - Operation: append
  - Target document: `YOUR_GOOGLE_SHEETS_ID`
  - Target sheet: `gid=0`
  - Writes fields such as title, company, location, salary, description, posting link, and Drive folder link
  - Generates manual `job_id` using timestamp
- **Key expressions or variables used:**  
  - `={{ $('AI Agent').item.json.output.platform }}`
  - `={{ $('AI Agent').item.json.output.link }}`
  - `=manually_added on {{ $now.toISO() }}`
  - `=https://drive.google.com/drive/u/0/folders/{{ $('Create_folder_for_attachments3').item.json.id }}`
- **Input and output connections:**  
  - Input: **Move_cv_to_attachments_folder3**
  - Output: **Notify user about successful process**
- **Version-specific requirements:** `typeVersion 4.7`
- **Edge cases / failures:**  
  - Google Sheets auth errors
  - Schema mismatch if sheet columns differ
  - Using append means duplicates are possible
- **Sub-workflow reference:** None

#### 18. Notify user about successful process
- **Type and role:** `n8n-nodes-base.telegram`; completion notification.
- **Configuration choices:**  
  - Sends `Done! :)` to the originating Telegram chat
- **Key expressions or variables used:**  
  - `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Input and output connections:**  
  - Inputs: **Append row in sheet1**, **Document link to attachments folder**
  - No output
- **Version-specific requirements:** `typeVersion 1.2`
- **Edge cases / failures:**  
  - Telegram auth/chat issues
- **Sub-workflow reference:** None

---

## Block 4 — Keyword Mode: Search, Extraction, and Filtering

### Overview
This block searches Google Jobs using the Telegram text, transforms the SerpAPI payload into normalized job items, and filters out jobs the user does not want to consider.

### Nodes Involved
- Search google jobs listings with SERP-API
- Extract Job results from SERP-API response
- Filter unwanted results based on your criteria
- No Operation, do nothing

### Node Details

#### 19. Search google jobs listings with SERP-API
- **Type and role:** `n8n-nodes-serpapi.serpApi`; queries SerpAPI Google Jobs endpoint.
- **Configuration choices:**  
  - Operation: `google_jobs`
  - Query comes from Telegram message text
  - Geographic parameters:
    - `gl = us`
    - `location = united states`
- **Key expressions or variables used:**  
  - `={{ $json.message.text }}`
- **Input and output connections:**  
  - Input: **If** false branch
  - Output: **Extract Job results from SERP-API response**
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases / failures:**  
  - Missing SerpAPI credentials
  - Rate limit or quota exceeded
  - Empty results
  - Query too vague
- **Sub-workflow reference:** None

#### 20. Extract Job results from SERP-API response
- **Type and role:** `n8n-nodes-base.code`; transforms SerpAPI result arrays into flat n8n items.
- **Configuration choices:**  
  - JavaScript iterates through `jobs_results`
  - Extracts:
    - core metadata
    - company/title/location/salary/type/description
    - up to 11 apply links and platform labels
  - Uses null defaults
- **Key expressions or variables used:**  
  - Reads `i.json.jobs_results`
  - Produces fields like `link1`, `application_platform_1`, etc.
- **Input and output connections:**  
  - Input: **Search google jobs listings with SERP-API**
  - Output: **Filter unwanted results based on your criteria**
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases / failures:**  
  - Unexpected SerpAPI response structure
  - `jobs_results` missing or empty
  - Only first 11 apply options are preserved
- **Sub-workflow reference:** None

#### 21. Filter unwanted results based on your criteria
- **Type and role:** `n8n-nodes-base.if`; exclusion filter.
- **Configuration choices:**  
  - OR-combined conditions
  - Ignores case
  - Excludes titles containing:
    - `student`
    - `intern`
    - `freelance`
    - `temporary`
    - `=internship` (likely a mistaken value but still configured)
    - `your_excluded_term`
  - Excludes types equal to:
    - `internship`
    - `part-time`
  - True branch = unwanted results
  - False branch = acceptable results
- **Key expressions or variables used:**  
  - `={{ $('Extract Job results from SERP-API response').item.json.title }}`
  - `={{ $('Extract Job results from SERP-API response').item.json.type }}`
- **Input and output connections:**  
  - Input: **Extract Job results from SERP-API response**
  - True output: **No Operation, do nothing**
  - False output: **Loop Over Items**
- **Version-specific requirements:** `typeVersion 2.3`
- **Edge cases / failures:**  
  - Title/type may be null, causing unexpected condition behavior
  - Exclusion logic may over-filter
  - The literal `=internship` condition looks accidental
- **Sub-workflow reference:** None

#### 22. No Operation, do nothing
- **Type and role:** `n8n-nodes-base.noOp`; sink for filtered-out jobs.
- **Configuration choices:** none
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input: **Filter unwanted results based on your criteria** true branch
  - No output
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases / failures:** none significant
- **Sub-workflow reference:** None

---

## Block 5 — Keyword Mode: Telegram Approval Loop

### Overview
This block iterates over acceptable search results, sends a formatted Telegram preview for each job, waits for user approval, and branches accordingly.

### Nodes Involved
- Loop Over Items
- Send Telegram message & approval card for each job
- Check if job was approved
- No Operation, do nothing1

### Node Details

#### 23. Loop Over Items
- **Type and role:** `n8n-nodes-base.splitInBatches`; item-by-item loop controller.
- **Configuration choices:**  
  - Default options; processes incoming jobs in batches/iterations
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input: **Filter unwanted results based on your criteria** false branch
  - First output to **Check if job was approved**
  - Second output to **Send Telegram message & approval card for each job**
  - Receives feedback from **Send Telegram message & approval card for each job** to continue loop
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - If no items remain after filtering, nothing proceeds
  - Misunderstanding loop wiring can make rebuilds fail
- **Sub-workflow reference:** None

#### 24. Send Telegram message & approval card for each job
- **Type and role:** `n8n-nodes-base.telegram`; sends message and waits for approval interaction.
- **Configuration choices:**  
  - Operation: `sendAndWait`
  - Approval type: `double`
  - `onError = continueRegularOutput`
  - Builds a Markdown-style message dynamically
  - Includes title, company, location, salary, description, and up to 10 apply links
  - Enforces a 4000-character limit:
    - tries full description first
    - removes description if too long
    - truncates if still too long
  - Uses `encodeURI` for safer link formatting
- **Key expressions or variables used:**  
  - `={{ $('Telegram Trigger').item.json.message.chat.id }}`
  - Large JavaScript expression in `message`
  - Uses `$json.title`, `$json.company`, `$json.location`, `$json.salary`, `$json.description`
- **Input and output connections:**  
  - Input: **Loop Over Items**
  - Output: back to **Loop Over Items**
- **Version-specific requirements:** `typeVersion 1.2`
- **Edge cases / failures:**  
  - Telegram formatting errors
  - Broken links
  - Approval interaction timeout or cancellation
  - Because `onError` continues, some failures may silently bypass ideal behavior
- **Sub-workflow reference:** None

#### 25. Check if job was approved
- **Type and role:** `n8n-nodes-base.if`; checks Telegram approval result.
- **Configuration choices:**  
  - Condition tests boolean `true` on:
    - `$('Send Telegram message & approval card for each job').item.json.data.approved`
  - True branch = approved
  - False branch = rejected/unapproved
- **Key expressions or variables used:**  
  - `={{ $('Send Telegram message & approval card for each job').item.json.data.approved }}`
- **Input and output connections:**  
  - Input: **Loop Over Items**
  - True output: **Document job results in Google sheet**
  - False output: **No Operation, do nothing1**
- **Version-specific requirements:** `typeVersion 2.2`
- **Edge cases / failures:**  
  - If approval payload shape changes or is absent, expression may fail
  - Timing issues if send-and-wait response not available as expected
- **Sub-workflow reference:** None

#### 26. No Operation, do nothing1
- **Type and role:** `n8n-nodes-base.noOp`; sink for rejected jobs.
- **Configuration choices:** none
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input: **Check if job was approved** false branch
  - No output
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases / failures:** none significant
- **Sub-workflow reference:** None

---

## Block 6 — Approved Keyword Results: Sheet Logging, PDF Generation, Drive Organization, and Completion

### Overview
This block handles approved jobs from the Telegram loop. It first writes or updates the Google Sheets tracker, then generates the cover letter and CV, uploads them to Drive, organizes them into a folder, updates the sheet with that folder link, and notifies the user.

### Nodes Involved
- Document job results in Google sheet
- Notify user about end of the loop
- Create customized cover letter
- Upload cover letter to Drive
- Create folder for attachments
- Move cover letter to attachments folder
- Create customized CV
- Upload customized CV
- Move CV to attachments folder
- Document link to attachments folder
- Notify user about successful process

### Node Details

#### 27. Document job results in Google sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; writes approved search result into tracking sheet.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Matching column: `job_id`
  - Sheet and document IDs are placeholders to replace
  - Maps extracted job fields from the Code node output
- **Key expressions or variables used:**  
  - Uses many expressions from `$('Extract Job results from SERP-API response').item.json.*`
  - Notably `job_id` expression contains a trailing newline in configuration
- **Input and output connections:**  
  - Input: **Check if job was approved** true branch
  - Outputs to:
    - **Create customized cover letter**
    - **Notify user about end of the loop**
- **Version-specific requirements:** `typeVersion 4.7`
- **Edge cases / failures:**  
  - Google auth or sheet mismatch
  - `appendOrUpdate` depends on stable `job_id`
  - The trailing newline in `job_id` mapping can cause matching inconsistencies
  - Parallel output means “end of the loop” message is not actually gated by completion of later document steps
- **Sub-workflow reference:** None

#### 28. Notify user about end of the loop
- **Type and role:** `n8n-nodes-base.telegram`; sends intermediate status update.
- **Configuration choices:**  
  - Text: `All done for now — your documents are being prepared!`
- **Key expressions or variables used:**  
  - `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Input and output connections:**  
  - Input: **Document job results in Google sheet**
  - No output
- **Version-specific requirements:** `typeVersion 1.2`
- **Edge cases / failures:**  
  - Message may be sent once per approved job, not once for whole loop
- **Sub-workflow reference:** None

#### 29. Create customized cover letter
- **Type and role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf`; generates cover letter PDF for approved keyword-mode job.
- **Configuration choices:**  
  - Output format: file
  - Uses HTML/CSS template referencing sheet data rather than AI Agent data
  - Inserts:
    - company from Google Sheets row
    - title from Google Sheets row
- **Key expressions or variables used:**  
  - `{{ $('Document job results in Google sheet').item.json.company }}`
  - `{{ $('Document job results in Google sheet').item.json.title }}`
- **Input and output connections:**  
  - Input: **Document job results in Google sheet**
  - Output: **Upload cover letter to Drive**
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases / failures:**  
  - Same HTML/PDF rendering issues as other PDF nodes
  - Depends on sheet node returning data in expected structure
- **Sub-workflow reference:** None

#### 30. Upload cover letter to Drive
- **Type and role:** `n8n-nodes-base.googleDrive`; uploads generated cover letter.
- **Configuration choices:**  
  - Name: `YourName_CoverLetter_<company>`
  - Initial parent folder: common applications folder
- **Key expressions or variables used:**  
  - `=YourName_CoverLetter_{{ $('Document job results in Google sheet').item.json.company}}`
- **Input and output connections:**  
  - Input: **Create customized cover letter**
  - Output: **Create folder for attachments**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - Missing binary
  - Invalid Drive folder ID
- **Sub-workflow reference:** None

#### 31. Create folder for attachments
- **Type and role:** `n8n-nodes-base.googleDrive`; creates target folder for approved application.
- **Configuration choices:**  
  - Resource: folder
  - Name: `<title>_<company>`
  - Parent folder: applications root
- **Key expressions or variables used:**  
  - `={{ $('Document job results in Google sheet').item.json.title }}_{{ $('Document job results in Google sheet').item.json.company }}`
- **Input and output connections:**  
  - Input: **Upload cover letter to Drive**
  - Output: **Move cover letter to attachments folder**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - Duplicate names
  - Permission issues
- **Sub-workflow reference:** None

#### 32. Move cover letter to attachments folder
- **Type and role:** `n8n-nodes-base.googleDrive`; moves cover letter into newly created folder.
- **Configuration choices:**  
  - File ID from uploaded cover letter
  - Folder ID from created folder
  - Operation: move
- **Key expressions or variables used:**  
  - `={{ $('Upload cover letter to Drive').item.json.id }}`
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input: **Create folder for attachments**
  - Output: **Create customized CV**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - Upstream ID missing
  - Drive permissions
- **Sub-workflow reference:** None

#### 33. Create customized CV
- **Type and role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf`; generates tailored CV PDF.
- **Configuration choices:**  
  - Output: file
  - Uses dynamic tag based on AI Agent values in template for one section, but node is in keyword mode chain
- **Key expressions or variables used:**  
  - Header references `{{ $('AI Agent').item.json.output.position }}`
  - Header references `{{ $('AI Agent').item.json.output.company }}`
  - File naming later uses sheet data
- **Input and output connections:**  
  - Input: **Move cover letter to attachments folder**
  - Output: **Upload customized CV**
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases / failures:**  
  - Important design flaw: in keyword mode there may be **no AI Agent execution**, so expressions referencing `AI Agent` can fail or resolve unexpectedly.
  - Same rendering issues as other PDF nodes
- **Sub-workflow reference:** None

#### 34. Upload customized CV
- **Type and role:** `n8n-nodes-base.googleDrive`; uploads CV.
- **Configuration choices:**  
  - Name: `YourName_CV_for_<company>`
  - Uploads first to root applications folder
- **Key expressions or variables used:**  
  - `=YourName_CV_for_{{ $('Document job results in Google sheet').item.json.company}}`
- **Input and output connections:**  
  - Input: **Create customized CV**
  - Output: **Move CV to attachments folder**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - Missing binary
  - OAuth/folder issues
- **Sub-workflow reference:** None

#### 35. Move CV to attachments folder
- **Type and role:** `n8n-nodes-base.googleDrive`; moves uploaded CV into application folder.
- **Configuration choices:**  
  - Operation: move
  - File ID from **Upload customized CV**
  - Folder ID from **Create folder for attachments**
- **Key expressions or variables used:**  
  - `={{ $('Upload customized CV').item.json.id }}`
  - `={{ $('Create folder for attachments').item.json.id }}`
- **Input and output connections:**  
  - Input: **Upload customized CV**
  - Output: **Document link to attachments folder**
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases / failures:**  
  - File/folder mismatch
- **Sub-workflow reference:** None

#### 36. Document link to attachments folder
- **Type and role:** `n8n-nodes-base.googleSheets`; updates existing sheet row with Drive folder link and repeats job fields.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Match by `job_id`
  - Writes `link_drive_ornder` with generated folder URL
- **Key expressions or variables used:**  
  - `=https://drive.google.com/drive/u/0/folders/{{ $('Create folder for attachments').item.json.id }}`
  - Most other values copied from `Document job results in Google sheet`
- **Input and output connections:**  
  - Input: **Move CV to attachments folder**
  - Output: **Notify user about successful process**
- **Version-specific requirements:** `typeVersion 4.7`
- **Edge cases / failures:**  
  - Sheet schema drift
  - Duplicate rows if match key is inconsistent
  - Column name typo `link_drive_ornder` is preserved and must match sheet schema exactly
- **Sub-workflow reference:** None

---

## Block 7 — Sticky Notes / Embedded Documentation

### Overview
These nodes are not executable logic but contain setup, usage, and block descriptions. They are important because they document prerequisites and intended behavior.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### 37. Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; high-level workflow documentation.
- **Configuration choices:**  
  - Explains both input modes, setup steps, and self-hosted PDF node requirement.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** None

#### 38. Sticky Note1
- **Type and role:** sticky note for Step 1
- **Configuration choices:** documents Telegram input routing
- **Connections:** none
- **Version:** `1`

#### 39. Sticky Note2
- **Type and role:** sticky note for URL mode AI extraction
- **Configuration choices:** documents Step 2a
- **Connections:** none
- **Version:** `1`

#### 40. Sticky Note3
- **Type and role:** sticky note for keyword mode SerpAPI path
- **Configuration choices:** documents Step 2b
- **Connections:** none
- **Version:** `1`

#### 41. Sticky Note4
- **Type and role:** sticky note for Telegram approval loop
- **Configuration choices:** documents Step 3
- **Connections:** none
- **Version:** `1`

#### 42. Sticky Note5
- **Type and role:** sticky note for document generation
- **Configuration choices:** documents Step 4
- **Connections:** none
- **Version:** `1`

#### 43. Sticky Note6
- **Type and role:** sticky note for Drive and Sheets update stage
- **Configuration choices:** documents Step 5
- **Connections:** none
- **Version:** `1`

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Entry point for Telegram messages |  | If | ### Step 1: Telegram Input & Routing<br>User sends either a **URL** or a **keyword** via Telegram.<br>The If node checks if the message contains `http` to route accordingly. |
| If | n8n-nodes-base.if | Route URL vs keyword mode | Telegram Trigger | AI Agent; Search google jobs listings with SERP-API | ### Step 1: Telegram Input & Routing<br>User sends either a **URL** or a **keyword** via Telegram.<br>The If node checks if the message contains `http` to route accordingly. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Extract job data from URL with AI | If | If1 | ### Step 2a: URL Mode -- AI Agent<br>OpenAI GPT with web search visits the URL, extracts structured job data (position, company, location, salary, description, language). Output is validated by a structured parser. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main LLM for AI Agent |  | AI Agent | ### Step 2a: URL Mode -- AI Agent<br>OpenAI GPT with web search visits the URL, extracts structured job data (position, company, location, salary, description, language). Output is validated by a structured parser. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for AI extraction |  | Auto-fixing Output Parser | ### Step 2a: URL Mode -- AI Agent<br>OpenAI GPT with web search visits the URL, extracts structured job data (position, company, location, salary, description, language). Output is validated by a structured parser. |
| Auto-fixing Output Parser | @n8n/n8n-nodes-langchain.outputParserAutofixing | Repair malformed AI output | Structured Output Parser; OpenAI Chat Model1 | AI Agent | ### Step 2a: URL Mode -- AI Agent<br>OpenAI GPT with web search visits the URL, extracts structured job data (position, company, location, salary, description, language). Output is validated by a structured parser. |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Repair model for output parser |  | Auto-fixing Output Parser | ### Step 2a: URL Mode -- AI Agent<br>OpenAI GPT with web search visits the URL, extracts structured job data (position, company, location, salary, description, language). Output is validated by a structured parser. |
| If1 | n8n-nodes-base.if | Check whether AI extraction succeeded | AI Agent | HTML to PDF; Notify user about error | ### Step 2a: URL Mode -- AI Agent<br>OpenAI GPT with web search visits the URL, extracts structured job data (position, company, location, salary, description, language). Output is validated by a structured parser. |
| Notify user about error | n8n-nodes-base.telegram | Inform user extraction failed | If1 |  | ### Step 2a: URL Mode -- AI Agent<br>OpenAI GPT with web search visits the URL, extracts structured job data (position, company, location, salary, description, language). Output is validated by a structured parser. |
| HTML to PDF | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Generate URL-mode cover letter PDF | If1 | Upload file | ### Step 4: Document Generation<br>Generates a styled **cover letter** and a **personalized CV** as PDF using HTML-to-PDF API. Both documents dynamically insert the target company and position. |
| Upload file | n8n-nodes-base.googleDrive | Upload URL-mode cover letter to Drive | HTML to PDF | Create_folder_for_attachments3 | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Create_folder_for_attachments3 | n8n-nodes-base.googleDrive | Create URL-mode application folder | Upload file | Move file | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Move file | n8n-nodes-base.googleDrive | Move URL-mode cover letter into folder | Create_folder_for_attachments3 | Create_personalized_cv3 | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Create_personalized_cv3 | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Generate URL-mode CV PDF | Move file | Upload_cv3 | ### Step 4: Document Generation<br>Generates a styled **cover letter** and a **personalized CV** as PDF using HTML-to-PDF API. Both documents dynamically insert the target company and position. |
| Upload_cv3 | n8n-nodes-base.googleDrive | Upload URL-mode CV to Drive | Create_personalized_cv3 | Move_cv_to_attachments_folder3 | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Move_cv_to_attachments_folder3 | n8n-nodes-base.googleDrive | Move URL-mode CV into folder | Upload_cv3 | Append row in sheet1 | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Append row in sheet1 | n8n-nodes-base.googleSheets | Append URL-mode job record to tracker | Move_cv_to_attachments_folder3 | Notify user about successful process | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Search google jobs listings with SERP-API | n8n-nodes-serpapi.serpApi | Search Google Jobs by keyword | If | Extract Job results from SERP-API response | ### Step 2b: Keyword Mode -- SerpAPI<br>Searches Google Jobs via SerpAPI, extracts results with a Code node, filters out unwanted job types (internships, freelance, part-time), then sends each result to Telegram for approval. |
| Extract Job results from SERP-API response | n8n-nodes-base.code | Flatten SerpAPI jobs into job items | Search google jobs listings with SERP-API | Filter unwanted results based on your criteria | ### Step 2b: Keyword Mode -- SerpAPI<br>Searches Google Jobs via SerpAPI, extracts results with a Code node, filters out unwanted job types (internships, freelance, part-time), then sends each result to Telegram for approval. |
| Filter unwanted results based on your criteria | n8n-nodes-base.if | Exclude unwanted job categories | Extract Job results from SERP-API response | No Operation, do nothing; Loop Over Items | ### Step 2b: Keyword Mode -- SerpAPI<br>Searches Google Jobs via SerpAPI, extracts results with a Code node, filters out unwanted job types (internships, freelance, part-time), then sends each result to Telegram for approval. |
| No Operation, do nothing | n8n-nodes-base.noOp | Sink for filtered-out jobs | Filter unwanted results based on your criteria |  | ### Step 2b: Keyword Mode -- SerpAPI<br>Searches Google Jobs via SerpAPI, extracts results with a Code node, filters out unwanted job types (internships, freelance, part-time), then sends each result to Telegram for approval. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterate through acceptable jobs | Filter unwanted results based on your criteria; Send Telegram message & approval card for each job | Check if job was approved; Send Telegram message & approval card for each job | ### Step 3: Telegram Approval Loop<br>Each job is sent to Telegram with a formatted preview. User approves or rejects each one via buttons. |
| Send Telegram message & approval card for each job | n8n-nodes-base.telegram | Send job preview and wait for approval | Loop Over Items | Loop Over Items | ### Step 3: Telegram Approval Loop<br>Each job is sent to Telegram with a formatted preview. User approves or rejects each one via buttons. |
| Check if job was approved | n8n-nodes-base.if | Branch approved vs rejected jobs | Loop Over Items | Document job results in Google sheet; No Operation, do nothing1 | ### Step 3: Telegram Approval Loop<br>Each job is sent to Telegram with a formatted preview. User approves or rejects each one via buttons. |
| No Operation, do nothing1 | n8n-nodes-base.noOp | Sink for rejected jobs | Check if job was approved |  | ### Step 3: Telegram Approval Loop<br>Each job is sent to Telegram with a formatted preview. User approves or rejects each one via buttons. |
| Document job results in Google sheet | n8n-nodes-base.googleSheets | Append or update approved keyword-mode job row | Check if job was approved | Create customized cover letter; Notify user about end of the loop | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Notify user about end of the loop | n8n-nodes-base.telegram | Interim status message after sheet logging | Document job results in Google sheet |  | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Create customized cover letter | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Generate keyword-mode cover letter PDF | Document job results in Google sheet | Upload cover letter to Drive | ### Step 4: Document Generation<br>Generates a styled **cover letter** and a **personalized CV** as PDF using HTML-to-PDF API. Both documents dynamically insert the target company and position. |
| Upload cover letter to Drive | n8n-nodes-base.googleDrive | Upload keyword-mode cover letter | Create customized cover letter | Create folder for attachments | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Create folder for attachments | n8n-nodes-base.googleDrive | Create keyword-mode application folder | Upload cover letter to Drive | Move cover letter to attachments folder | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Move cover letter to attachments folder | n8n-nodes-base.googleDrive | Move keyword-mode cover letter into folder | Create folder for attachments | Create customized CV | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Create customized CV | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Generate keyword-mode CV PDF | Move cover letter to attachments folder | Upload customized CV | ### Step 4: Document Generation<br>Generates a styled **cover letter** and a **personalized CV** as PDF using HTML-to-PDF API. Both documents dynamically insert the target company and position. |
| Upload customized CV | n8n-nodes-base.googleDrive | Upload keyword-mode CV | Create customized CV | Move CV to attachments folder | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Move CV to attachments folder | n8n-nodes-base.googleDrive | Move keyword-mode CV into folder | Upload customized CV | Document link to attachments folder | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Document link to attachments folder | n8n-nodes-base.googleSheets | Update approved keyword-mode row with Drive folder link | Move CV to attachments folder | Notify user about successful process | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Notify user about successful process | n8n-nodes-base.telegram | Final completion message | Append row in sheet1; Document link to attachments folder |  | ### Step 5: Google Drive & Sheets<br>Uploads cover letter + CV to Drive, creates a named folder per application, moves files into it, and updates the Google Sheets tracker with Drive folder link. |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow overview and setup notes |  |  | ## Automate job applications with Telegram, SerpAPI, and OpenAI<br><br>This workflow automates your job application pipeline via a **Telegram bot**. Send a job link or search keyword, approve the results, and get auto-generated cover letters and CVs uploaded to Google Drive.<br><br>## How it works<br><br>**Two input modes:**<br>1. **URL mode** -- Send a job posting link. An AI Agent (OpenAI with web search) visits the page and extracts structured job data (title, company, location, salary, description).<br>2. **Keyword mode** -- Send a search query. SerpAPI returns Google Jobs results, filters out unwanted types (internships, freelance, part-time), and sends each result to Telegram for approval.<br><br>**For each approved job:**<br>1. Logs the job details in a **Google Sheets** tracker<br>2. Generates a styled **cover letter** PDF (HTML-to-PDF)<br>3. Generates a tailored **CV** PDF with the target company and position<br>4. Uploads both documents to a **Google Drive** folder<br>5. Updates the Sheet with the Drive folder link<br>6. Confirms completion via **Telegram**<br><br>## Setup steps<br><br>1. **Telegram Bot** -- Create a bot via @BotFather. Add the token as a Telegram credential. Set your user ID in the Trigger node.<br>2. **OpenAI** -- Add your API key as an OpenAI credential.<br>3. **SerpAPI** -- Register at serpapi.com and add your API key.<br>4. **Google Sheets & Drive** -- Create OAuth2 credentials. Create a Sheet and update its ID in all Google Sheets nodes. Create a Drive folder and update its ID in all Google Drive nodes.<br>5. **HTML-to-PDF** -- Install the community node `n8n-nodes-htmlcsstopdf` (self-hosted only). Add the API credential.<br>6. **Customize templates** -- Edit the cover letter and CV HTML nodes with your personal details, photo, and work history.<br><br>> **Community node required:** `n8n-nodes-htmlcsstopdf` -- this template works on **self-hosted n8n only**. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for input routing |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for URL mode |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for keyword mode |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation for approval loop |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation for document generation |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation for Drive and Sheets |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence that mirrors the workflow logic.

## Prerequisites
1. Prepare a **Telegram bot** using `@BotFather`.
2. Obtain your **Telegram user ID**.
3. Prepare **OpenAI API credentials**.
4. Prepare **SerpAPI credentials**.
5. Prepare **Google OAuth2 credentials** for:
   - Google Drive
   - Google Sheets
6. Install the community node **`n8n-nodes-htmlcsstopdf`** on a self-hosted n8n instance.
7. Create:
   - one Google Drive parent folder for job applications,
   - one Google Sheets document with a sheet matching the columns used by the workflow.

## Suggested Google Sheets columns
Create a sheet with these columns at minimum:
- via
- title
- job_id
- location
- posted_at
- salary
- type
- share_link
- description
- company
- link1
- application_platform_1
- link2
- application_platform_2
- link3
- application_platform_3
- link4
- application_platform_4
- link5
- application_platform_5
- link6
- application_platform_6
- link7
- application_platform_7
- link8
- application_platform_8
- link9
- application_platform_9
- link10
- application_platform_10
- link_drive_ornder

## Build Steps

1. **Create a Telegram Trigger node**
   - Type: `Telegram Trigger`
   - Listen to `message` updates.
   - Restrict allowed users with your Telegram user ID.
   - Attach Telegram credentials.

2. **Create an IF node named `If`**
   - Condition: message text contains `http`
   - Left value: `{{$json.message.text}}`
   - Right value: `http`
   - Connect `Telegram Trigger -> If`

3. **Create the URL-mode AI branch**
   - Add `AI Agent`
   - Prompt type: define manually
   - Paste a prompt instructing the model to visit the URL and extract structured job data with `extraction_successful`
   - Use input URL: `{{ $('Telegram Trigger').item.json.message.text }}`

4. **Add the main OpenAI model**
   - Node type: `OpenAI Chat Model`
   - Model: `gpt-5.1`
   - Enable built-in `webSearch`
   - Search context size: medium
   - Connect as AI language model to `AI Agent`

5. **Add a Structured Output Parser**
   - Use manual schema
   - Include fields:
     - extraction_successful boolean
     - position
     - platform
     - link
     - location
     - posted_at
     - salary
     - share_link
     - description
     - company
     - language enum: de/en/other

6. **Add an Auto-fixing Output Parser**
   - Connect the structured parser into it.

7. **Add a second OpenAI Chat Model**
   - Model: `gpt-5.1`
   - No tool requirement
   - Connect it as the language model for the auto-fixing parser.

8. **Connect the parser chain**
   - `Structured Output Parser -> Auto-fixing Output Parser -> AI Agent`

9. **Connect URL routing**
   - `If` true output -> `AI Agent`

10. **Add IF node `If1`**
   - Condition: `{{$json.output.extraction_successful}}` is true
   - `AI Agent -> If1`

11. **Add Telegram node `Notify user about error`**
   - Send message:
     - “Sorry i was not able to extract url-content due to website restrictions. Please provide the full description and application url in this chat.”
   - Chat ID:
     - `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   - Connect `If1` false branch to it.

12. **Add HTML-to-PDF node for URL-mode cover letter**
   - Name it `HTML to PDF`
   - Output format: file
   - Paste your HTML and CSS template
   - Insert dynamic expressions for:
     - company: `{{ $('AI Agent').item.json.output.company }}`
     - position: `{{ $('AI Agent').item.json.output.position }}`
   - Replace placeholder personal data, image URLs, links, and text
   - Connect `If1` true branch -> `HTML to PDF`

13. **Add Google Drive upload node for cover letter**
   - Name: `Upload file`
   - Upload the incoming binary file
   - Set destination folder to your parent applications folder
   - File name example:
     - `YourName_CoverLetter_{{ $('AI Agent').item.json.output.company }}`
   - Connect from `HTML to PDF`

14. **Add Google Drive folder creation node**
   - Name: `Create_folder_for_attachments3`
   - Resource: folder
   - Parent folder: your applications folder
   - Name:
     - `{{ $('AI Agent').item.json.output.position }}_{{ $('AI Agent').item.json.output.company }}`
   - Connect from `Upload file`

15. **Add Google Drive move node for cover letter**
   - Name: `Move file`
   - Operation: move
   - File ID:
     - `{{ $('Upload file').item.json.id }}`
   - Folder ID:
     - `{{ $json.id }}`
   - Connect from `Create_folder_for_attachments3`

16. **Add HTML-to-PDF node for URL-mode CV**
   - Name: `Create_personalized_cv3`
   - Output format: file
   - Paste CV HTML/CSS template
   - Replace all placeholders with your real personal data
   - Keep target-specific dynamic text using:
     - `{{ $('AI Agent').item.json.output.position }}`
     - `{{ $('AI Agent').item.json.output.company }}`
   - Connect from `Move file`

17. **Add Drive upload node for CV**
   - Name: `Upload_cv3`
   - File name:
     - `YourName_CV_for_{{ $('AI Agent').item.json.output.company }}`
   - Upload to parent applications folder
   - Connect from `Create_personalized_cv3`

18. **Add Drive move node for CV**
   - Name: `Move_cv_to_attachments_folder3`
   - Operation: move
   - File ID:
     - `{{ $('Upload_cv3').item.json.id }}`
   - Folder ID:
     - `{{ $('Create_folder_for_attachments3').item.json.id }}`
   - Connect from `Upload_cv3`

19. **Add Google Sheets append node for URL mode**
   - Name: `Append row in sheet1`
   - Operation: append
   - Point to your spreadsheet and first sheet
   - Map fields from `AI Agent` output
   - Create a manual job ID such as:
     - `manually_added on {{ $now.toISO() }}`
   - Add Drive folder URL:
     - `https://drive.google.com/drive/u/0/folders/{{ $('Create_folder_for_attachments3').item.json.id }}`
   - Connect from `Move_cv_to_attachments_folder3`

20. **Add final Telegram success node**
   - Name: `Notify user about successful process`
   - Text: `Done! :)`
   - Chat ID:
     - `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   - Connect from `Append row in sheet1`

21. **Create the keyword-mode SerpAPI branch**
   - Add SerpAPI node
   - Name: `Search google jobs listings with SERP-API`
   - Operation: `google_jobs`
   - Query:
     - `{{ $json.message.text }}`
   - Additional fields:
     - `gl = us`
     - `location = united states`
   - Connect `If` false branch -> this node

22. **Add a Code node**
   - Name: `Extract Job results from SERP-API response`
   - Write JavaScript that:
     - loops through `jobs_results`
     - flattens one item per job
     - extracts title, company, description, type, salary, share link
     - extracts apply options into fields `link1..link11` and `application_platform_1..11`
   - Connect from SerpAPI node

23. **Add filtering IF node**
   - Name: `Filter unwanted results based on your criteria`
   - Use OR logic
   - Ignore case
   - Exclude terms like:
     - student
     - intern
     - freelance
     - temporary
     - your_excluded_term
   - Exclude exact type values:
     - internship
     - part-time
   - Connect from code node

24. **Add `No Operation, do nothing`**
   - Connect it to the true/output-for-unwanted branch of the filter node.

25. **Add `Loop Over Items`**
   - Node type: `Split In Batches`
   - Connect filter false branch -> loop

26. **Add Telegram send-and-wait approval node**
   - Name: `Send Telegram message & approval card for each job`
   - Operation: `sendAndWait`
   - Approval type: double
   - Chat ID:
     - `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   - Build a dynamic message showing title, company, location, salary, description, and apply links
   - Add length handling to stay under Telegram limits
   - Set `On Error` to continue regular output if desired
   - Connect `Loop Over Items -> Send Telegram message...`
   - Connect `Send Telegram message... -> Loop Over Items` to continue iteration

27. **Add approval check IF node**
   - Name: `Check if job was approved`
   - Condition:
     - `{{ $('Send Telegram message & approval card for each job').item.json.data.approved }}`
     - is true
   - Connect `Loop Over Items -> Check if job was approved`

28. **Add `No Operation, do nothing1`**
   - Connect it to the false branch of `Check if job was approved`

29. **Add Google Sheets append-or-update node for approved jobs**
   - Name: `Document job results in Google sheet`
   - Operation: `appendOrUpdate`
   - Match by `job_id`
   - Map all normalized fields from the Code node output
   - Connect from the true branch of `Check if job was approved`

30. **Add Telegram status node**
   - Name: `Notify user about end of the loop`
   - Text: `All done for now — your documents are being prepared!`
   - Chat ID from Telegram Trigger
   - Connect from `Document job results in Google sheet`

31. **Add keyword-mode cover letter PDF node**
   - Name: `Create customized cover letter`
   - Output format: file
   - HTML/CSS template should reference:
     - `{{ $('Document job results in Google sheet').item.json.company }}`
     - `{{ $('Document job results in Google sheet').item.json.title }}`
   - Connect from `Document job results in Google sheet`

32. **Add keyword-mode cover letter upload**
   - Name: `Upload cover letter to Drive`
   - File name:
     - `YourName_CoverLetter_{{ $('Document job results in Google sheet').item.json.company }}`
   - Parent folder: applications folder
   - Connect from `Create customized cover letter`

33. **Add keyword-mode folder creation**
   - Name: `Create folder for attachments`
   - Resource: folder
   - Name:
     - `{{ $('Document job results in Google sheet').item.json.title }}_{{ $('Document job results in Google sheet').item.json.company }}`
   - Connect from `Upload cover letter to Drive`

34. **Add keyword-mode move cover letter**
   - Name: `Move cover letter to attachments folder`
   - Move uploaded cover letter into created folder
   - Connect from `Create folder for attachments`

35. **Add keyword-mode CV PDF node**
   - Name: `Create customized CV`
   - Output format: file
   - Important: in a safer rebuild, use `Document job results in Google sheet` values everywhere instead of `AI Agent` values, because keyword mode does not run the AI Agent.
   - Connect from `Move cover letter to attachments folder`

36. **Add keyword-mode CV upload**
   - Name: `Upload customized CV`
   - File name:
     - `YourName_CV_for_{{ $('Document job results in Google sheet').item.json.company }}`
   - Connect from `Create customized CV`

37. **Add keyword-mode move CV**
   - Name: `Move CV to attachments folder`
   - Move uploaded CV into the created folder
   - Connect from `Upload customized CV`

38. **Add final Google Sheets update node**
   - Name: `Document link to attachments folder`
   - Operation: `appendOrUpdate`
   - Match by `job_id`
   - Write folder URL to `link_drive_ornder`
   - Connect from `Move CV to attachments folder`

39. **Connect keyword-mode completion**
   - Connect `Document link to attachments folder -> Notify user about successful process`

40. **Optional: add sticky notes**
   - Add notes for:
     - setup
     - routing
     - URL mode
     - keyword mode
     - approval loop
     - document generation
     - Drive/Sheets updates

## Credential Configuration Summary

### Telegram
- Use Telegram bot token credential
- Ensure the bot has been started by the target user
- Put your Telegram numeric user ID in the trigger restriction field

### OpenAI
- Use an OpenAI credential with access to `gpt-5.1`
- Ensure web search/tool usage is supported for your environment

### SerpAPI
- Add API key credential
- Confirm access to Google Jobs search

### Google Drive
- OAuth2 credential with Drive access
- Replace all `YOUR_GOOGLE_DRIVE_FOLDER_ID` placeholders

### Google Sheets
- OAuth2 credential with Sheets access
- Replace all `YOUR_GOOGLE_SHEETS_ID` placeholders

### HTML-to-PDF
- Install `n8n-nodes-htmlcsstopdf`
- Configure any required API/service credentials per the node package documentation
- Use self-hosted n8n

## Important Rebuild Improvements
When recreating this workflow, consider fixing these issues:
1. In keyword mode, change `Create customized CV` so it **does not reference `AI Agent` output**.
2. Remove the newline from the `job_id` mapping in `Document job results in Google sheet`.
3. Correct the likely typo `link_drive_ornder` only if your sheet schema is also updated consistently.
4. Consider sending “All done for now” only after the loop truly finishes, not per approved item.
5. Consider replacing the `http` heuristic with a URL validation regex.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Main workflow description: Automates job applications via Telegram bot, supporting URL mode and keyword mode, then generates cover letter and CV PDFs and stores them in Google Drive with Google Sheets tracking. | Embedded workflow note |
| Community node required: `n8n-nodes-htmlcsstopdf` | Self-hosted n8n only |
| Telegram bot must be created via `@BotFather` and restricted to your Telegram user ID in the trigger. | Telegram setup |
| SerpAPI account and API key are required for the Google Jobs search branch. | https://serpapi.com |
| Replace all placeholders such as `YOUR_GOOGLE_DRIVE_FOLDER_ID`, `YOUR_GOOGLE_SHEETS_ID`, profile image URLs, LinkedIn URL, creator URL, certificate URLs, and personal text. | Required customization |
| n8n Creator link placeholder used in templates | `https://n8n.io/creators/yourprofile/` |
| LinkedIn profile placeholder used in templates | `https://www.linkedin.com/in/yourprofile/` |
| Certificate placeholder link 1 used in templates | `https://example.com/your-certificate-1` |
| Certificate placeholder link 2 used in templates | `https://example.com/your-certificate-2` |
| Publication placeholder link 1 used in CV template | `https://example.com/your-publication-1` |
| Publication placeholder link 2 used in CV template | `https://example.com/your-publication-2` |
| Publication placeholder link 3 used in CV template | `https://example.com/your-publication-3` |
| Publication placeholder link 4 used in CV template | `https://example.com/your-publication-4` |