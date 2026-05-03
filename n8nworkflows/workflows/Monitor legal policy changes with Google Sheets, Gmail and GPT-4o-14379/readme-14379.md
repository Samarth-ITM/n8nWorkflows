Monitor legal policy changes with Google Sheets, Gmail and GPT-4o

https://n8nworkflows.xyz/workflows/monitor-legal-policy-changes-with-google-sheets--gmail-and-gpt-4o-14379


# Monitor legal policy changes with Google Sheets, Gmail and GPT-4o

# 1. Workflow Overview

This workflow monitors legal-policy-style web pages such as Terms of Service, Privacy Policies, or related compliance pages, detects textual changes, uses GPT-4o to summarize verified differences, sends an alert by email, and updates a Google Sheets-based tracking register.

Its main use cases are:
- monitoring public legal documents for changes,
- maintaining a historical change log,
- receiving concise AI-generated alerts only when meaningful differences are detected,
- keeping a sheet of monitored pages updated with the latest fetched content.

The workflow is organized into the following logical blocks.

## 1.1 Scheduled Start
A daily schedule triggers the workflow every day at 8:00.

## 1.2 Page List Retrieval and Fetching
The workflow reads monitored page records from a Google Sheets tab named `Pages`, then fetches each URL over HTTP.

## 1.3 Content Extraction and Change Detection
Fetched HTML is cleaned into plain text, compared against previously stored content, and classified as first run, changed, unchanged, or failed.

## 1.4 AI Change Summarization
If a page has changed, the workflow sends a constrained comparison prompt to GPT-4o to produce a structured summary of verifiable differences.

## 1.5 Alerting and Change Logging
For changed pages, the workflow builds an HTML email, sends it via Gmail, logs the event in a `Change Log` sheet, and updates the corresponding page record in `Pages`.

## 1.6 Record Maintenance for Unchanged or Non-alert Paths
If no change is detected, the workflow still updates the `Pages` sheet with the latest check date and latest fetched content.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Start

### Overview
This block is the workflow entry point. It launches the monitoring cycle once per day at a configured hour.

### Nodes Involved
- `Daily Check`

### Node Details

#### Daily Check
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger node that starts the workflow automatically.
- **Configuration choices:**  
  Configured with an interval rule that triggers at hour `8`. This means the workflow runs daily at 08:00 according to the instance timezone.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none, entry point
  - Output: `Get Pages to Monitor`
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`. Behavior depends on n8n scheduler support being enabled on the instance.
- **Edge cases or potential failure types:**  
  - Workflow inactive: no execution occurs
  - Server timezone mismatch: runs at an unexpected local time
  - Self-hosted scheduler issues or worker misconfiguration
- **Sub-workflow reference:**  
  None.

---

## 2.2 Page List Retrieval and Fetching

### Overview
This block loads the list of monitored pages from Google Sheets and fetches the current version of each URL. It is the acquisition layer for all monitored content.

### Nodes Involved
- `Get Pages to Monitor`
- `Fetch Page`

### Node Details

#### Get Pages to Monitor
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheets workbook tab containing the pages to monitor.
- **Configuration choices:**  
  - Google Sheet document ID is set explicitly as `YOUR_GOOGLE_SHEET_ID`
  - Sheet name is `Pages`
  - No special options are enabled
  - Default operation here is effectively row retrieval
- **Key expressions or variables used:**  
  This node provides row-level fields later used downstream, especially:
  - `url`
  - `page_name`
  - `last_content`
  - likely `last_checked` if present in the sheet
- **Input and output connections:**  
  - Input: `Daily Check`
  - Output: `Fetch Page`
- **Version-specific requirements:**  
  Uses `typeVersion 4.7`, which supports the newer Google Sheets parameter model and schema-aware mappings in other sheet nodes.
- **Edge cases or potential failure types:**  
  - OAuth credential invalid or expired
  - Incorrect Google Sheet ID
  - Missing `Pages` tab
  - Rows missing a `url`
  - Header names in the sheet not matching expected field names
- **Sub-workflow reference:**  
  None.

#### Fetch Page
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs an HTTP GET request to retrieve the current content of each monitored page.
- **Configuration choices:**  
  - URL is dynamic: `{{ $json.url }}`
  - Sends a custom `User-Agent`: `Mozilla/5.0 (compatible; TOSWatcher/1.0)`
  - `onError` is set to `continueRegularOutput`, so failures are passed onward instead of stopping execution
- **Key expressions or variables used:**  
  - `={{ $json.url }}`
- **Input and output connections:**  
  - Input: `Get Pages to Monitor`
  - Output: `Extract & Compare`
- **Version-specific requirements:**  
  Uses `typeVersion 4.4`.
- **Edge cases or potential failure types:**  
  - Invalid URL in the sheet
  - Timeout or DNS failure
  - SSL/TLS issues
  - Access denied / anti-bot response
  - Redirect chains
  - Non-HTML responses
  - Since errors continue downstream, later code must correctly interpret error payloads
- **Sub-workflow reference:**  
  None.

---

## 2.3 Content Extraction and Change Detection

### Overview
This block normalizes fetched responses into plain text, checks for fetch errors or anti-bot protection, compares new text with stored text, and prepares a compact diff-focused context for AI analysis.

### Nodes Involved
- `Extract & Compare`
- `Content Changed?`

### Node Details

#### Extract & Compare
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes custom JavaScript once per item to transform HTML into comparable text and compute change metadata.
- **Configuration choices:**  
  - Mode: `runOnceForEachItem`
  - Reads current HTTP result from `$input.item.json`
  - Also reads original sheet row via `$('Get Pages to Monitor').item.json`
  - Creates a normalized `checked_at` date in `YYYY-MM-DD`
  - Detects HTTP errors and returns structured error objects
  - Detects Cloudflare/bot-challenge patterns
  - Rejects very short responses under 200 characters
  - Strips `<script>`, `<style>`, all HTML tags, selected entities, repeated whitespace
  - Truncates final plain text to 40,000 characters
  - Computes:
    - `is_first_run`
    - `changed`
    - `old_content_for_ai`
    - `new_content_for_ai`
  - For changed pages, it attempts to find the first differing character and sends a focused excerpt plus tail content to AI
- **Key expressions or variables used:**  
  Important output fields:
  - `url`
  - `page_name`
  - `new_content`
  - `old_content`
  - `old_content_for_ai`
  - `new_content_for_ai`
  - `changed`
  - `is_first_run`
  - `error`
  - `checked_at`
- **Input and output connections:**  
  - Input: `Fetch Page`
  - Output: `Content Changed?`
- **Version-specific requirements:**  
  Uses `typeVersion 2`. Requires Code node JavaScript support.
- **Edge cases or potential failure types:**  
  - Expression lookup failure if upstream item linking changes unexpectedly
  - Unexpected HTTP response structure
  - HTML cleaning may merge text in a way that creates false positives
  - Entity decoding is partial, not exhaustive
  - Dynamic or personalized page content may trigger constant changes
  - The simplistic diff-start logic finds first divergence, but not semantic diff blocks
  - First run is intentionally not treated as a “changed” alert
  - Errors are returned as data, but not branched explicitly by `error`; they simply produce `changed: false`
- **Sub-workflow reference:**  
  None.

#### Content Changed?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches execution based on whether a real content change was detected.
- **Configuration choices:**  
  - Checks `{{ $json.changed }}` is boolean true
  - Uses condition version 2
- **Key expressions or variables used:**  
  - `={{ $json.changed }}`
- **Input and output connections:**  
  - Input: `Extract & Compare`
  - True output: `Summarize Changes`
  - False output: `Update Last Checked`
- **Version-specific requirements:**  
  Uses `typeVersion 2.3`.
- **Edge cases or potential failure types:**  
  - If `changed` is missing or malformed, loose validation may still evaluate unexpectedly
  - Errors from prior fetches are not separated from normal “unchanged” states
- **Sub-workflow reference:**  
  None.

---

## 2.4 AI Change Summarization

### Overview
This block uses GPT-4o through the LangChain integration to compare previous and current content snippets and produce a structured legal-style change summary.

### Nodes Involved
- `Summarize Changes`
- `OpenAI Chat Model`

### Node Details

#### Summarize Changes
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Sends a prompt to a connected chat language model and returns text output.
- **Configuration choices:**  
  - Prompt type: defined text prompt
  - Prompt strongly instructs the model to:
    - compare old and new versions,
    - list only verifiable changes,
    - avoid guessing,
    - quote exact changes,
    - assign impact level,
    - explain practical user impact
  - Includes page metadata:
    - `page_name`
    - `url`
  - Uses:
    - `old_content_for_ai`
    - `new_content_for_ai`
- **Key expressions or variables used:**  
  - `{{ $json.page_name }}`
  - `{{ $json.url }}`
  - `{{ $json.old_content_for_ai }}`
  - `{{ $json.new_content_for_ai }}`
- **Input and output connections:**  
  - Main input: `Content Changed?` true branch
  - AI language model input: `OpenAI Chat Model`
  - Main output: `Build Email Body`
- **Version-specific requirements:**  
  Uses `typeVersion 1.9`. Requires compatible LangChain nodes installed in the n8n instance.
- **Edge cases or potential failure types:**  
  - Missing OpenAI credentials
  - Model access denied for `gpt-4o`
  - Token/context limits if excerpts grow too large
  - Prompt output format may vary slightly despite instructions
  - If page text is noisy or badly extracted, summary quality declines
- **Sub-workflow reference:**  
  None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the GPT-4o chat model instance used by the LLM chain.
- **Configuration choices:**  
  - Model selected: `gpt-4o`
  - No extra options set
  - Built-in tools disabled / unused
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to: `Summarize Changes`
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`. Requires OpenAI API credentials and account access to the selected model.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Quota exceeded
  - Rate limiting
  - Selected model unavailable in region/account
- **Sub-workflow reference:**  
  None.

---

## 2.5 Alerting and Change Logging

### Overview
When changes are confirmed and summarized, this block creates an HTML notification email, sends it through Gmail, records the event in a log sheet, and updates the monitored page’s stored content.

### Nodes Involved
- `Build Email Body`
- `Send Change Alert`
- `Log Change`
- `Update Page Record`

### Node Details

#### Build Email Body
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts AI output into a formatted HTML email payload and passes forward metadata needed for logging and record updates.
- **Configuration choices:**  
  - Reads all incoming items with `$input.all()`
  - For each item:
    - Gets AI summary from `item.json.text` or `item.json.output`
    - Pulls original comparison data from `$('Extract & Compare').item.json`
    - Builds HTML body with page name, URL, date, AI summary, and footer
    - Sets:
      - `email_subject`
      - `email_body`
      - `url`
      - `page_name`
      - `summary`
      - `checked_at`
      - `new_content`
- **Key expressions or variables used:**  
  - `$('Extract & Compare').item.json`
  - `item.json.text`
  - `item.json.output`
- **Input and output connections:**  
  - Input: `Summarize Changes`
  - Output: `Send Change Alert`
- **Version-specific requirements:**  
  Uses `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - Cross-item linking may become unreliable if item order changes or batching behavior differs
  - HTML email body is assembled manually, so malformed summary formatting may affect display
  - Summary fallback text may hide upstream AI failures
- **Sub-workflow reference:**  
  None.

#### Send Change Alert
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated alert email using Gmail OAuth2.
- **Configuration choices:**  
  - Recipient: `user@example.com`
  - Subject: `{{ $json.email_subject }}`
  - Message body: `{{ $json.email_body }}`
  - `appendAttribution` disabled
- **Key expressions or variables used:**  
  - `={{ $json.email_subject }}`
  - `={{ $json.email_body }}`
- **Input and output connections:**  
  - Input: `Build Email Body`
  - Output: `Log Change`
- **Version-specific requirements:**  
  Uses `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth expired
  - Send quota exceeded
  - HTML rendering differences in mail clients
  - Hardcoded recipient not updated for production use
- **Sub-workflow reference:**  
  None.

#### Log Change
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row to a historical log of detected changes.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet name: `Change Log`
  - Writes columns:
    - `url`
    - `date`
    - `summary`
    - `page_name`
  - Uses expressions that explicitly reference `Build Email Body`
- **Key expressions or variables used:**  
  - `{{ $('Build Email Body').item.json.url }}`
  - `{{ $('Build Email Body').item.json.checked_at }}`
  - `{{ $('Build Email Body').item.json.summary }}`
  - `{{ $('Build Email Body').item.json.page_name }}`
- **Input and output connections:**  
  - Input: `Send Change Alert`
  - Output: `Update Page Record`
- **Version-specific requirements:**  
  Uses `typeVersion 4.7`.
- **Edge cases or potential failure types:**  
  - Missing `Change Log` tab
  - Column headers mismatch
  - Google API write quota or auth failure
  - Long summaries may exceed practical spreadsheet usability
- **Sub-workflow reference:**  
  None.

#### Update Page Record
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates or appends the corresponding monitored page row after a detected change.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Sheet: `Pages`
  - Matching column: `url`
  - Updates:
    - `url`
    - `last_checked`
    - `last_content`
  - Uses `Build Email Body` outputs
- **Key expressions or variables used:**  
  - `{{ $('Build Email Body').item.json.url }}`
  - `{{ $('Build Email Body').item.json.checked_at }}`
  - `{{ $('Build Email Body').item.json.new_content }}`
- **Input and output connections:**  
  - Input: `Log Change`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion 4.7`.
- **Edge cases or potential failure types:**  
  - No matching `url` header in the `Pages` sheet
  - Duplicate URLs in the sheet can create ambiguous updates
  - Very large `last_content` values may make the sheet heavy and harder to manage
- **Sub-workflow reference:**  
  None.

---

## 2.6 Record Maintenance for Unchanged or Non-alert Paths

### Overview
This block handles the false branch of the change detector. It updates the page’s last checked timestamp and stores the latest content even when no email or AI summary is generated.

### Nodes Involved
- `Update Last Checked`

### Node Details

#### Update Last Checked
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Performs an `appendOrUpdate` into the `Pages` sheet for pages that did not produce a change alert.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Sheet name: `Pages`
  - Matching column: `url`
  - Writes:
    - `url`
    - `last_checked`
    - `last_content`
  - Source values come directly from the current item
- **Key expressions or variables used:**  
  - `={{ $json.url }}`
  - `={{ $json.checked_at }}`
  - `={{ $json.new_content }}`
- **Input and output connections:**  
  - Input: `Content Changed?` false branch
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion 4.7`.
- **Edge cases or potential failure types:**  
  - This branch also handles first runs and fetch-error cases because they result in `changed: false`
  - If fetch failed, `new_content` may be empty, which can overwrite stored content with blank data
  - Duplicate URLs in the sheet may lead to inconsistent updates
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Check | n8n-nodes-base.scheduleTrigger | Starts the workflow every day at 08:00 |  | Get Pages to Monitor | ## Daily scheduling\nInitiates the workflow on a daily schedule to start the monitoring process. |
| Get Pages to Monitor | n8n-nodes-base.googleSheets | Reads monitored page records from the `Pages` sheet | Daily Check | Fetch Page | ## Fetch pages for monitoring\nRetrieves list of pages to monitor from Google Sheets and fetches each page's content. |
| Fetch Page | n8n-nodes-base.httpRequest | Downloads each monitored page over HTTP | Get Pages to Monitor | Extract & Compare | ## Fetch pages for monitoring\nRetrieves list of pages to monitor from Google Sheets and fetches each page's content. |
| Extract & Compare | n8n-nodes-base.code | Normalizes fetched HTML, compares with stored content, and prepares AI diff context | Fetch Page | Content Changed? | ## Content comparison\nCompares the fetched content with the stored version to detect changes. |
| Content Changed? | n8n-nodes-base.if | Branches based on whether a content change was detected | Extract & Compare | Summarize Changes; Update Last Checked | ## Content comparison\nCompares the fetched content with the stored version to detect changes. |
| Summarize Changes | @n8n/n8n-nodes-langchain.chainLlm | Uses an LLM prompt to summarize verified changes | Content Changed?; OpenAI Chat Model | Build Email Body | ## AI-driven change summarization\nIf changes are detected, uses AI to summarize the differences between the current and previous content. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the GPT-4o model to the LLM chain |  | Summarize Changes | ## AI-driven change summarization\nIf changes are detected, uses AI to summarize the differences between the current and previous content. |
| Build Email Body | n8n-nodes-base.code | Builds the HTML alert email and carries summary metadata | Summarize Changes | Send Change Alert | ## Email alert and logging\nBuilds the email content, sends alerts via Gmail, and logs the change in Google Sheets. |
| Send Change Alert | n8n-nodes-base.gmail | Sends the alert email through Gmail | Build Email Body | Log Change | ## Email alert and logging\nBuilds the email content, sends alerts via Gmail, and logs the change in Google Sheets. |
| Log Change | n8n-nodes-base.googleSheets | Appends a record to the `Change Log` sheet | Send Change Alert | Update Page Record | ## Email alert and logging\nBuilds the email content, sends alerts via Gmail, and logs the change in Google Sheets. |
| Update Page Record | n8n-nodes-base.googleSheets | Updates the `Pages` sheet after a detected change | Log Change |  | ## Email alert and logging\nBuilds the email content, sends alerts via Gmail, and logs the change in Google Sheets. |
| Update Last Checked | n8n-nodes-base.googleSheets | Updates the `Pages` sheet when no alert path is taken | Content Changed? |  | ## Update monitoring records\nUpdates the Google Sheets record to reflect the latest check time, whether changes were detected or not. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation / workflow description |  |  | ## Terms of Service Change Watcher with AI Summaries\n\n### How it works\n\n1. Schedules a daily check using Daily Check. 2. Retrieves URLs from a Google Sheets workbook. 3. Fetches the specified pages and compares content. 4. Uses an AI model to summarize changes if detected. 5. Sends a change alert email and logs the change.\n\n### Setup steps\n\n- [ ] Set up Google Sheets credentials for retrieving and storing page data\n- [ ] Configure the OpenAI Chat Model for generating AI summaries\n- [ ] Set up Gmail credentials to send email alerts\n\n### Customization\n\nTo monitor different URLs, update the Google Sheets file with new entries. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for scheduling section |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for sheet read and fetch section |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for comparison section |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note for AI section |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas note for email and logging section |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas note for record update section |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `Terms of Service Change Watcher with AI Summaries`.

2. **Add the schedule trigger**
   - Create a `Schedule Trigger` node.
   - Name it `Daily Check`.
   - Configure it to run daily at hour `8`.
   - Ensure your n8n instance timezone matches your intended operating timezone.

3. **Prepare the Google Sheet**
   - Create a Google Sheets file.
   - Add a tab named `Pages`.
   - Add at least these columns:
     - `url`
     - `page_name`
     - `last_content`
     - `last_checked`
   - Add another tab named `Change Log`.
   - Add these columns:
     - `date`
     - `page_name`
     - `url`
     - `summary`

4. **Add the Google Sheets reader**
   - Create a `Google Sheets` node.
   - Name it `Get Pages to Monitor`.
   - Connect `Daily Check -> Get Pages to Monitor`.
   - Choose Google Sheets OAuth2 credentials.
   - Set the spreadsheet document ID.
   - Set sheet name to `Pages`.
   - Configure it to read rows from the sheet.

5. **Add the HTTP fetch node**
   - Create an `HTTP Request` node.
   - Name it `Fetch Page`.
   - Connect `Get Pages to Monitor -> Fetch Page`.
   - Set the URL to the expression `{{ $json.url }}`.
   - Enable headers and add:
     - `User-Agent: Mozilla/5.0 (compatible; TOSWatcher/1.0)`
   - Set error behavior to continue output instead of failing the whole run.

6. **Add the comparison code node**
   - Create a `Code` node.
   - Name it `Extract & Compare`.
   - Connect `Fetch Page -> Extract & Compare`.
   - Set mode to `Run Once for Each Item`.
   - Paste logic equivalent to the workflow’s script:
     - read the HTTP response,
     - get original sheet row from `Get Pages to Monitor`,
     - compute `checked_at`,
     - detect HTTP failures,
     - reject Cloudflare challenge pages,
     - reject tiny responses,
     - strip scripts, styles, HTML tags, basic entities, and extra whitespace,
     - truncate to about 40,000 characters,
     - compare with `last_content`,
     - mark `is_first_run`,
     - mark `changed`,
     - generate focused `old_content_for_ai` and `new_content_for_ai`.
   - Ensure the node returns fields:
     - `url`
     - `page_name`
     - `new_content`
     - `old_content`
     - `old_content_for_ai`
     - `new_content_for_ai`
     - `changed`
     - `is_first_run`
     - `error`
     - `checked_at`

7. **Add the IF node**
   - Create an `If` node.
   - Name it `Content Changed?`.
   - Connect `Extract & Compare -> Content Changed?`.
   - Add a boolean condition checking whether `{{ $json.changed }}` is true.

8. **Add the OpenAI model node**
   - Create an `OpenAI Chat Model` node from the LangChain set.
   - Name it `OpenAI Chat Model`.
   - Select model `gpt-4o`.
   - Add OpenAI credentials with a valid API key.
   - Leave optional model settings at default unless you need temperature or token tuning.

9. **Add the LLM chain node**
   - Create a `Basic LLM Chain` / `Chain LLM` node from the LangChain nodes.
   - Name it `Summarize Changes`.
   - Connect the **true** branch of `Content Changed?` to this node.
   - Connect `OpenAI Chat Model` to the LLM node using the AI language model connector.
   - Set prompt type to manual/defined text.
   - Use a prompt that:
     - tells the model to compare old and new content,
     - requires quoting only visible, verifiable differences,
     - forbids guessing,
     - requires sections for overview, changes, impact level, and user impact.
   - Insert these expressions in the prompt:
     - `{{ $json.page_name }}`
     - `{{ $json.url }}`
     - `{{ $json.old_content_for_ai }}`
     - `{{ $json.new_content_for_ai }}`

10. **Add the email body builder**
    - Create a `Code` node.
    - Name it `Build Email Body`.
    - Connect `Summarize Changes -> Build Email Body`.
    - Build a script that:
      - loops through incoming LLM results,
      - extracts the summary from `text` or `output`,
      - retrieves original page metadata from `Extract & Compare`,
      - creates HTML email content,
      - outputs:
        - `email_subject`
        - `email_body`
        - `url`
        - `page_name`
        - `summary`
        - `checked_at`
        - `new_content`

11. **Add the Gmail sender**
    - Create a `Gmail` node.
    - Name it `Send Change Alert`.
    - Connect `Build Email Body -> Send Change Alert`.
    - Configure Gmail OAuth2 credentials.
    - Set recipient email, for example replacing `user@example.com` with the real destination.
    - Subject expression: `{{ $json.email_subject }}`
    - Message expression: `{{ $json.email_body }}`
    - Disable appended attribution if desired.

12. **Add the change log writer**
    - Create another `Google Sheets` node.
    - Name it `Log Change`.
    - Connect `Send Change Alert -> Log Change`.
    - Choose the same spreadsheet.
    - Set operation to `Append`.
    - Set sheet name to `Change Log`.
    - Map columns:
      - `url` = `{{ $('Build Email Body').item.json.url }}`
      - `date` = `{{ $('Build Email Body').item.json.checked_at }}`
      - `summary` = `{{ $('Build Email Body').item.json.summary }}`
      - `page_name` = `{{ $('Build Email Body').item.json.page_name }}`

13. **Add the page record updater for changed pages**
    - Create another `Google Sheets` node.
    - Name it `Update Page Record`.
    - Connect `Log Change -> Update Page Record`.
    - Set operation to `Append or Update`.
    - Sheet name: `Pages`
    - Matching column: `url`
    - Map:
      - `url` = `{{ $('Build Email Body').item.json.url }}`
      - `last_checked` = `{{ $('Build Email Body').item.json.checked_at }}`
      - `last_content` = `{{ $('Build Email Body').item.json.new_content }}`

14. **Add the no-change updater**
    - Create another `Google Sheets` node.
    - Name it `Update Last Checked`.
    - Connect the **false** branch of `Content Changed?` to `Update Last Checked`.
    - Set operation to `Append or Update`.
    - Sheet name: `Pages`
    - Matching column: `url`
    - Map:
      - `url` = `{{ $json.url }}`
      - `last_checked` = `{{ $json.checked_at }}`
      - `last_content` = `{{ $json.new_content }}`

15. **Configure credentials**
    - **Google Sheets OAuth2**
      - Must have read/write access to the target spreadsheet.
    - **OpenAI API**
      - Must allow `gpt-4o`.
    - **Gmail OAuth2**
      - Must permit sending emails from the chosen account.

16. **Seed the `Pages` sheet**
    - Add one row per monitored page.
    - Example values:
      - `url`: public legal-policy URL
      - `page_name`: human-readable label
      - `last_content`: leave empty initially
      - `last_checked`: leave empty initially

17. **Test the workflow**
    - Run manually once.
    - On first run:
      - pages should be fetched,
      - `last_content` and `last_checked` should be stored,
      - no alert should be sent because first run is not treated as a change.
    - Modify one monitored page or simulate a changed target URL.
    - Run again and verify:
      - AI summary is generated,
      - Gmail alert is sent,
      - `Change Log` receives a new row,
      - `Pages` gets updated.

18. **Activate the workflow**
    - Turn the workflow on once all credentials, sheet names, and recipient addresses are correct.

### Expected Inputs and Outputs
- **Input source:** scheduled trigger only
- **Primary data source:** rows in `Pages`
- **Expected fields in `Pages`:**
  - `url` as unique key
  - `page_name`
  - `last_content`
  - `last_checked`
- **Expected fields in `Change Log`:**
  - `date`
  - `page_name`
  - `url`
  - `summary`

### Important implementation constraints
- Use unique URLs in the `Pages` sheet.
- Prefer stable, public, non-personalized web pages.
- Avoid sites behind anti-bot or JavaScript-heavy rendering unless you swap the fetch method.
- Be aware that storing full content in Sheets can become large over time.

### Reproduction note for AI agents
There are **no sub-workflow nodes** and **no separate called workflows** in this workflow. All logic is contained in one workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Terms of Service Change Watcher with AI Summaries | Workflow title / branding |
| How it works: 1. Schedules a daily check using Daily Check. 2. Retrieves URLs from a Google Sheets workbook. 3. Fetches the specified pages and compares content. 4. Uses an AI model to summarize changes if detected. 5. Sends a change alert email and logs the change. | General workflow description |
| Setup steps: Set up Google Sheets credentials for retrieving and storing page data; Configure the OpenAI Chat Model for generating AI summaries; Set up Gmail credentials to send email alerts. | Deployment notes |
| Customization: To monitor different URLs, update the Google Sheets file with new entries. | Operational note |

## Additional technical observations
- The false branch currently updates the sheet even when fetch errors occur. This can overwrite previous stored content with empty content if a page fails to load.
- First-run pages do not generate alerts by design.
- The HTML-to-text extraction is intentionally simple and may produce noise on complex sites.
- The LLM prompt is careful and restrictive, but it does not guarantee perfect literal diffing; for high-stakes legal monitoring, consider supplementing with deterministic diff logic before AI summarization.
- The workflow uses hardcoded placeholder values for:
  - Google Sheet ID
  - email recipient
  - credential references  
  These must be replaced before production use.