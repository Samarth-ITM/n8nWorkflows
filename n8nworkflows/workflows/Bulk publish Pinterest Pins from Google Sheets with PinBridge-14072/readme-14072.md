Bulk publish Pinterest Pins from Google Sheets with PinBridge

https://n8nworkflows.xyz/workflows/bulk-publish-pinterest-pins-from-google-sheets-with-pinbridge-14072


# Bulk publish Pinterest Pins from Google Sheets with PinBridge

# 1. Workflow Overview

This workflow bulk-submits Pinterest Pins using rows stored in a Google Sheet. It uses Google Sheets as the queue and source of truth, n8n as the controller, and PinBridge as the publishing service.

Its main use case is batch processing Pinterest content prepared in spreadsheet form. Each row represents one Pin candidate. The workflow reads rows, skips already published ones, validates required fields, downloads the source image, uploads that image to PinBridge, submits a Pinterest publish job, and then writes the outcome back to the same row.

The workflow is intentionally limited to **job submission tracking**, not final Pinterest delivery confirmation. It records successful submissions as `submitted`, and invalid rows as `invalid`.

## 1.1 Input Reception and Queue Read
The workflow starts manually and reads all rows from the `Pins` tab in Google Sheets.

## 1.2 Reprocessing Guard
Rows already marked as `published` are filtered out to avoid duplicate processing during reruns.

## 1.3 Validation and Routing
Each remaining row is checked for required fields:
- `title`
- `description`
- `link_url`
- `image_url`
- `board_id`

Valid rows continue to the publishing path. Invalid rows are marked in the sheet with an error.

## 1.4 Asset Preparation and Publish Submission
For valid rows, the workflow downloads the image from `image_url`, uploads it to PinBridge as an asset, and then submits a Pinterest publish job through PinBridge.

## 1.5 Writeback to Google Sheets
After submission, the workflow updates the original row using `row_id` as the matching key. It writes either:
- a success payload: `submitted`, `job_id`, `published_at`, blank `error_message`
- or an invalid payload: `invalid`, blank `job_id`, blank `published_at`, descriptive `error_message`

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Sheet Read

**Overview:**  
This block starts the workflow manually and loads all candidate Pin rows from Google Sheets. It establishes the spreadsheet as the operational queue for the entire process.

**Nodes Involved:**  
- Manual Trigger
- Read Sheet Rows

### Manual Trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual execution entry point.
- **Configuration choices:** No parameters are configured. The workflow must be started manually from the editor or via manual execution context.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - No input
  - Output → `Read Sheet Rows`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at node level; it simply starts execution.
- **Sub-workflow reference:** None.

### Read Sheet Rows
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from a specific Google Sheet tab.
- **Configuration choices:**  
  - Uses Google Sheets OAuth2 credentials
  - Document ID is set to `YOUR_GOOGLE_SHEET_ID`
  - Sheet name is `Pins`
  - No special options enabled
- **Key expressions or variables used:** Static sheet identifier and tab name.
- **Input and output connections:**  
  - Input ← `Manual Trigger`
  - Output → `Skip Published Rows`
- **Version-specific requirements:** Type version `4.5`; requires compatible Google Sheets credentials and access to the target spreadsheet.
- **Edge cases or potential failure types:**  
  - Invalid or unreplaced spreadsheet ID
  - OAuth permission failure
  - Sheet tab `Pins` missing or renamed
  - Empty sheet returning no items
  - Header mismatches causing downstream field lookups to fail
- **Sub-workflow reference:** None.

---

## 2.2 Published Row Filtering

**Overview:**  
This block prevents already published rows from re-entering the processing path. It is a basic idempotency safeguard for reruns.

**Nodes Involved:**  
- Skip Published Rows

### Skip Published Rows
- **Type and technical role:** `n8n-nodes-base.if`; conditional filter.
- **Configuration choices:**  
  - Evaluates whether `status` is not equal to `published`
  - Converts the incoming status to lowercase string to make the comparison case-insensitive in practice
- **Key expressions or variables used:**  
  - `={{ ($json.status || '').toString().toLowerCase() }}`
  - Compared against `published`
- **Input and output connections:**  
  - Input ← `Read Sheet Rows`
  - True output → `Validate Required Fields`
  - False output is unused, so rows already marked `published` simply stop here
- **Version-specific requirements:** Type version `2.2`, condition options version `2`.
- **Edge cases or potential failure types:**  
  - If status contains extra whitespace, it will not be trimmed and may pass unexpectedly
  - Rows marked as `submitted`, `invalid`, or any custom state will continue
  - Empty status also continues
- **Sub-workflow reference:** None.

---

## 2.3 Validation and Branching

**Overview:**  
This block checks whether the row has all required business fields before any external API call occurs. It separates valid publishing candidates from incomplete spreadsheet rows.

**Nodes Involved:**  
- Validate Required Fields

### Validate Required Fields
- **Type and technical role:** `n8n-nodes-base.if`; gatekeeper for required field completeness.
- **Configuration choices:**  
  - Uses an `and` combinator across five boolean checks
  - Each field is converted into a truthy/falsy boolean using JavaScript double negation
  - Required fields checked: `title`, `description`, `link_url`, `image_url`, `board_id`
- **Key expressions or variables used:**  
  - `={{ !!$json.title }}`
  - `={{ !!$json.description }}`
  - `={{ !!$json.link_url }}`
  - `={{ !!$json.image_url }}`
  - `={{ !!$json.board_id }}`
- **Input and output connections:**  
  - Input ← `Skip Published Rows`
  - True output → `Download Image`
  - False output → `Build Invalid Result`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Strings containing only spaces will still count as valid because `!!'   '` is `true`
  - No URL format validation is performed for `link_url` or `image_url`
  - No board ID format validation is performed
  - No check exists for `row_id`, although later sheet updates depend on it
- **Sub-workflow reference:** None.

---

## 2.4 Valid Row Processing: Download, Upload, Submit

**Overview:**  
This is the main operational path. It fetches the image, uploads it to PinBridge, and submits the Pinterest publish job using the row’s content metadata.

**Nodes Involved:**  
- Download Image
- Upload Image to PinBridge
- Publish to Pinterest

### Download Image
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the source image from the provided URL.
- **Configuration choices:**  
  - URL is dynamically set from the row field `image_url`
  - No additional options are explicitly configured
- **Key expressions or variables used:**  
  - `={{ $json.image_url }}`
- **Input and output connections:**  
  - Input ← `Validate Required Fields` (true branch)
  - Output → `Upload Image to PinBridge`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Non-public image URL
  - Redirects or hotlink protections
  - Timeout or DNS failure
  - 404/403 response
  - Binary handling assumptions may depend on the HTTP Request node defaults in the n8n version used
  - Unsupported or malformed image content
- **Sub-workflow reference:** None.

### Upload Image to PinBridge
- **Type and technical role:** `n8n-nodes-pinbridge.pinBridge`; creates or uploads an asset to PinBridge.
- **Configuration choices:**  
  - Resource is set to `assets`
  - Uses PinBridge API credentials named `PinBridge Key Sandbox`
- **Key expressions or variables used:** None explicitly shown in parameters; the node likely consumes incoming binary/image data from the previous node.
- **Input and output connections:**  
  - Input ← `Download Image`
  - Output → `Publish to Pinterest`
- **Version-specific requirements:** Type version `1`; requires the PinBridge community node/package to be installed in the n8n instance and matching credentials configured.
- **Edge cases or potential failure types:**  
  - Missing PinBridge credential
  - PinBridge node package not installed
  - Incoming image not available in expected binary format
  - File type or size rejected by PinBridge
  - API rate limits or sandbox limitations
- **Sub-workflow reference:** None.

### Publish to Pinterest
- **Type and technical role:** `n8n-nodes-pinbridge.pinBridge`; submits the Pinterest publication job through PinBridge.
- **Configuration choices:**  
  - `accountId` is hardcoded as `YOUR_PINTEREST_ACCOUNT_ID`
  - `title` from `title`
  - `description` from `description`
  - `linkUrl` from `link_url`
  - `boardId` from `board_id`
  - `altText` from `alt_text` or empty string
  - `dominantColor` from `dominant_color` or `#ffffff`
- **Key expressions or variables used:**  
  - `={{ $json.title }}`
  - `={{ $json.alt_text || '' }}`
  - `={{ $json.board_id }}`
  - `={{ $json.link_url }}`
  - `={{ $json.description }}`
  - `={{ $json.dominant_color || '#ffffff' }}`
- **Input and output connections:**  
  - Input ← `Upload Image to PinBridge`
  - Output → `Build Success Result`
- **Version-specific requirements:** Type version `1`; requires the same PinBridge extension and valid API access.
- **Edge cases or potential failure types:**  
  - Placeholder `YOUR_PINTEREST_ACCOUNT_ID` not replaced
  - Invalid board ID
  - Invalid account-board relationship
  - Metadata rejected by downstream API
  - Publish node may depend on asset information emitted by the previous PinBridge node; field compatibility must be preserved
- **Sub-workflow reference:** None.

---

## 2.5 Success Result Construction and Sheet Update

**Overview:**  
This block normalizes a successful submit response into spreadsheet fields and writes the update back to the original row. It does not confirm final Pinterest publication, only successful job submission.

**Nodes Involved:**  
- Build Success Result
- Update Sheet Success

### Build Success Result
- **Type and technical role:** `n8n-nodes-base.set`; builds a clean success payload for sheet writeback.
- **Configuration choices:**  
  - Sets `status` to `submitted`
  - Sets `job_id` from the current item’s `id`
  - Sets `published_at` to the execution timestamp `$now`
  - Clears `error_message`
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
  - `={{ $now }}`
- **Input and output connections:**  
  - Input ← `Publish to Pinterest`
  - Output → `Update Sheet Success`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If PinBridge returns a different identifier field than `id`, `job_id` will be blank or wrong
  - Depending on Set node behavior, fields not explicitly assigned may or may not be preserved; the downstream update relies on `row_id` still being present
- **Sub-workflow reference:** None.

### Update Sheet Success
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates the matching row in the `Pins` sheet after successful submission.
- **Configuration choices:**  
  - Operation: `update`
  - Sheet name: `Pins`
  - Document ID: `YOUR_GOOGLE_SHEET_ID`
  - Matching column: `row_id`
  - Writes `job_id`, `status`, `published_at`, `error_message`
  - Mapping mode is explicitly defined
- **Key expressions or variables used:**  
  - `={{ $json.job_id }}`
  - `={{ $json.status }}`
  - `={{ $json.published_at }}`
  - `={{ $json.error_message }}`
- **Input and output connections:**  
  - Input ← `Build Success Result`
  - No downstream node
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases or potential failure types:**  
  - If `row_id` is absent or not preserved from prior nodes, the update may fail or affect no row
  - If `row_id` is not unique, update behavior may be ambiguous
  - Google Sheets permissions or tab mismatch
  - Data type mismatches if the sheet schema is changed
- **Sub-workflow reference:** None.

---

## 2.6 Invalid Row Handling and Sheet Update

**Overview:**  
This block handles rows that fail validation. Instead of calling external services, it writes a clear invalid state and error message back to the source row.

**Nodes Involved:**  
- Build Invalid Result
- Update Sheet Invalid Row

### Build Invalid Result
- **Type and technical role:** `n8n-nodes-base.set`; creates the payload used to mark a row as invalid.
- **Configuration choices:**  
  - Sets `status` to `invalid`
  - Sets `error_message` to a fixed explanation listing required fields
  - Clears `job_id`
  - Clears `published_at`
- **Key expressions or variables used:** None beyond static values.
- **Input and output connections:**  
  - Input ← `Validate Required Fields` (false branch)
  - Output → `Update Sheet Invalid Row`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Same `row_id` preservation concern as in the success path
  - The message is generic and does not identify which specific field is missing
- **Sub-workflow reference:** None.

### Update Sheet Invalid Row
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates the original row with invalid-state fields.
- **Configuration choices:**  
  - Operation: `update`
  - Sheet name: `Pins`
  - Document ID: `YOUR_GOOGLE_SHEET_ID`
  - Matching column: `row_id`
  - Writes `job_id`, `status`, `published_at`, `error_message`
- **Key expressions or variables used:**  
  - `={{ $json.job_id }}`
  - `={{ $json.status }}`
  - `={{ $json.published_at }}`
  - `={{ $json.error_message }}`
- **Input and output connections:**  
  - Input ← `Build Invalid Result`
  - No downstream node
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases or potential failure types:**  
  - Missing or non-unique `row_id`
  - Spreadsheet access errors
  - Sheet schema changes that remove expected columns
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Documentation note describing the overall workflow purpose and process |  |  | # Bulk publish Pinterest Pins from Google Sheets  This template uses **Google Sheets as the input queue**, **n8n as the orchestration layer**, and **PinBridge as the Pinterest publishing layer**.  ### How it works 1. Read rows from the `Pins` sheet 2. Skip rows already marked as published 3. Validate required row fields 4. Download the image 5. Upload the image to PinBridge 6. Submit the Pinterest publish job 7. Write the result back to the same sheet |
| Sticky Note - Setup | n8n-nodes-base.stickyNote | Documentation note listing prerequisite configuration |  |  | ## Setup checklist  Before running this workflow, replace the placeholders and connect credentials:  - Add a **Google Sheets** credential - Add a **PinBridge** credential - Replace `YOUR_GOOGLE_SHEET_ID` - Replace `YOUR_PINTEREST_ACCOUNT_ID` - Confirm the sheet tab name is `Pins` - Confirm each row has a unique `row_id` - Confirm `image_url` points to a **publicly reachable image** - Confirm `board_id` is the **real Pinterest board ID**  ### Expected columns `row_id`, `title`, `description`, `link_url`, `image_url`, `board_id`, `alt_text`, `dominant_color`, `status`, `job_id`, `published_at`, `error_message` |
| Sticky Note - Read and filter | n8n-nodes-base.stickyNote | Documentation note for the intake and filtering section |  |  | ## Read and filter rows  The workflow starts manually, then reads all rows from the `Pins` tab.  `Skip Published Rows` prevents rows already marked as `published` from being processed again.  This keeps re-runs safer and gives you a clear place to define future reprocessing logic. |
| Sticky Note - Valid rows | n8n-nodes-base.stickyNote | Documentation note for the valid-row publish path |  |  | ## Validate and submit valid rows  `Validate Required Fields` checks that the row contains: `title` | `description` | `link_url` | `image_url` | `board_id`  If valid, the workflow downloads the image, uploads it to PinBridge, then submits the Pinterest publish job.  On success, the workflow writes: - `status = submitted` \| `job_id` \| `published_at` \| empty `error_message` |
| Sticky Note - Invalid rows | n8n-nodes-base.stickyNote | Documentation note for invalid-row handling |  |  | ## Handle invalid rows  Rows missing one or more required fields do **not** continue to external steps.  Instead, the workflow marks the row as:  - `status = invalid` - `error_message = Missing one or more required fields...`  This makes the sheet self-correcting and easier to review after each run. |
| Sticky Note - Scope | n8n-nodes-base.stickyNote | Documentation note clarifying workflow scope |  |  | ## What this template covers  This template records **successful job submission** only.  A separate workflow should handle: - webhook callbacks - final publish confirmation - retries for downstream failures - notifications and reporting  That keeps this template easier to import, test, and understand. |
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual workflow start |  | Read Sheet Rows | ## Read and filter rows  The workflow starts manually, then reads all rows from the `Pins` tab.  `Skip Published Rows` prevents rows already marked as `published` from being processed again.  This keeps re-runs safer and gives you a clear place to define future reprocessing logic. |
| Read Sheet Rows | n8n-nodes-base.googleSheets | Read all queue rows from the `Pins` sheet | Manual Trigger | Skip Published Rows | ## Read and filter rows  The workflow starts manually, then reads all rows from the `Pins` tab.  `Skip Published Rows` prevents rows already marked as `published` from being processed again.  This keeps re-runs safer and gives you a clear place to define future reprocessing logic. |
| Skip Published Rows | n8n-nodes-base.if | Filter out rows already marked `published` | Read Sheet Rows | Validate Required Fields | ## Read and filter rows  The workflow starts manually, then reads all rows from the `Pins` tab.  `Skip Published Rows` prevents rows already marked as `published` from being processed again.  This keeps re-runs safer and gives you a clear place to define future reprocessing logic. |
| Validate Required Fields | n8n-nodes-base.if | Check that mandatory publishing fields exist | Skip Published Rows | Download Image; Build Invalid Result | ## Validate and submit valid rows  `Validate Required Fields` checks that the row contains: `title` | `description` | `link_url` | `image_url` | `board_id`  If valid, the workflow downloads the image, uploads it to PinBridge, then submits the Pinterest publish job.  On success, the workflow writes: - `status = submitted` \| `job_id` \| `published_at` \| empty `error_message` |
| Download Image | n8n-nodes-base.httpRequest | Fetch the image from the row’s `image_url` | Validate Required Fields | Upload Image to PinBridge | ## Validate and submit valid rows  `Validate Required Fields` checks that the row contains: `title` | `description` | `link_url` | `image_url` | `board_id`  If valid, the workflow downloads the image, uploads it to PinBridge, then submits the Pinterest publish job.  On success, the workflow writes: - `status = submitted` \| `job_id` \| `published_at` \| empty `error_message` |
| Upload Image to PinBridge | n8n-nodes-pinbridge.pinBridge | Upload the downloaded image as a PinBridge asset | Download Image | Publish to Pinterest | ## Validate and submit valid rows  `Validate Required Fields` checks that the row contains: `title` | `description` | `link_url` | `image_url` | `board_id`  If valid, the workflow downloads the image, uploads it to PinBridge, then submits the Pinterest publish job.  On success, the workflow writes: - `status = submitted` \| `job_id` \| `published_at` \| empty `error_message` |
| Publish to Pinterest | n8n-nodes-pinbridge.pinBridge | Submit the Pinterest publish job through PinBridge | Upload Image to PinBridge | Build Success Result | ## Validate and submit valid rows  `Validate Required Fields` checks that the row contains: `title` | `description` | `link_url` | `image_url` | `board_id`  If valid, the workflow downloads the image, uploads it to PinBridge, then submits the Pinterest publish job.  On success, the workflow writes: - `status = submitted` \| `job_id` \| `published_at` \| empty `error_message` |
| Build Success Result | n8n-nodes-base.set | Prepare sheet update payload for successful submission | Publish to Pinterest | Update Sheet Success | ## Validate and submit valid rows  `Validate Required Fields` checks that the row contains: `title` | `description` | `link_url` | `image_url` | `board_id`  If valid, the workflow downloads the image, uploads it to PinBridge, then submits the Pinterest publish job.  On success, the workflow writes: - `status = submitted` \| `job_id` \| `published_at` \| empty `error_message` |
| Update Sheet Success | n8n-nodes-base.googleSheets | Update the row in Google Sheets after successful submission | Build Success Result |  | ## Validate and submit valid rows  `Validate Required Fields` checks that the row contains: `title` | `description` | `link_url` | `image_url` | `board_id`  If valid, the workflow downloads the image, uploads it to PinBridge, then submits the Pinterest publish job.  On success, the workflow writes: - `status = submitted` \| `job_id` \| `published_at` \| empty `error_message` |
| Build Invalid Result | n8n-nodes-base.set | Prepare sheet update payload for invalid rows | Validate Required Fields | Update Sheet Invalid Row | ## Handle invalid rows  Rows missing one or more required fields do **not** continue to external steps.  Instead, the workflow marks the row as:  - `status = invalid` - `error_message = Missing one or more required fields...`  This makes the sheet self-correcting and easier to review after each run. |
| Update Sheet Invalid Row | n8n-nodes-base.googleSheets | Update the row in Google Sheets for invalid data | Build Invalid Result |  | ## Handle invalid rows  Rows missing one or more required fields do **not** continue to external steps.  Instead, the workflow marks the row as:  - `status = invalid` - `error_message = Missing one or more required fields...`  This makes the sheet self-correcting and easier to review after each run. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Bulk publish Pinterest Pins from Google Sheets with PinBridge`.

2. **Prepare the Google Sheet** before building nodes.  
   Create a spreadsheet with a tab named `Pins`. Add these columns exactly:
   - `row_id`
   - `title`
   - `description`
   - `link_url`
   - `image_url`
   - `board_id`
   - `alt_text`
   - `dominant_color`
   - `status`
   - `job_id`
   - `published_at`
   - `error_message`

3. **Ensure row identity is stable.**  
   Every row must have a unique `row_id` because both update nodes use `row_id` as the matching key.

4. **Create credentials for Google Sheets.**
   - Add a `Google Sheets OAuth2` credential in n8n.
   - Authorize it to access the spreadsheet.
   - Verify the account can read and update the target file.

5. **Install and configure PinBridge support.**
   - Ensure the PinBridge node package is available in your n8n instance.
   - Create a `PinBridge` credential using your API key or service access details.
   - If using sandbox credentials, verify the sandbox supports the target operations.

6. **Add a `Manual Trigger` node.**
   - No extra configuration is required.
   - This will be the workflow’s only entry point.

7. **Add a `Google Sheets` node** and name it `Read Sheet Rows`.
   - Connect `Manual Trigger` → `Read Sheet Rows`.
   - Choose the Google Sheets credential created earlier.
   - Set the document ID to your spreadsheet ID, replacing `YOUR_GOOGLE_SHEET_ID`.
   - Set the sheet/tab name to `Pins`.
   - Configure it to read rows from the sheet.

8. **Add an `If` node** and name it `Skip Published Rows`.
   - Connect `Read Sheet Rows` → `Skip Published Rows`.
   - Build one condition:
     - Left value: `={{ ($json.status || '').toString().toLowerCase() }}`
     - Operator: string `not equals`
     - Right value: `published`
   - This allows all rows except those already marked `published`.

9. **Add another `If` node** and name it `Validate Required Fields`.
   - Connect the **true** output of `Skip Published Rows` to this node.
   - Use `AND` logic across five conditions:
     - `={{ !!$json.title }}` equals `true`
     - `={{ !!$json.description }}` equals `true`
     - `={{ !!$json.link_url }}` equals `true`
     - `={{ !!$json.image_url }}` equals `true`
     - `={{ !!$json.board_id }}` equals `true`

10. **Add an `HTTP Request` node** and name it `Download Image`.
    - Connect the **true** output of `Validate Required Fields` → `Download Image`.
    - Set the URL to:
      - `={{ $json.image_url }}`
    - Use GET.
    - Make sure the node is configured in a way that preserves downloaded image data for the next node. In many n8n setups this means enabling binary response handling if needed.

11. **Add a `PinBridge` node** and name it `Upload Image to PinBridge`.
    - Connect `Download Image` → `Upload Image to PinBridge`.
    - Select your PinBridge credential.
    - Set the resource to `assets`.
    - Confirm the node consumes the incoming image/binary content from the previous step as expected by your PinBridge node version.

12. **Add a second `PinBridge` node** and name it `Publish to Pinterest`.
    - Connect `Upload Image to PinBridge` → `Publish to Pinterest`.
    - Select the same PinBridge credential.
    - Configure these fields:
      - `accountId`: replace `YOUR_PINTEREST_ACCOUNT_ID`
      - `title`: `={{ $json.title }}`
      - `description`: `={{ $json.description }}`
      - `boardId`: `={{ $json.board_id }}`
      - `linkUrl`: `={{ $json.link_url }}`
      - `altText`: `={{ $json.alt_text || '' }}`
      - `dominantColor`: `={{ $json.dominant_color || '#ffffff' }}`
    - Verify your PinBridge node version automatically uses the uploaded asset from the prior node.

13. **Add a `Set` node** and name it `Build Success Result`.
    - Connect `Publish to Pinterest` → `Build Success Result`.
    - Add these assignments:
      - `status` = `submitted`
      - `job_id` = `={{ $json.id }}`
      - `published_at` = `={{ $now }}`
      - `error_message` = empty string
    - Important: ensure the original row context, especially `row_id`, is still available to the next Google Sheets update. If your n8n configuration drops unassigned fields, enable field retention or re-add `row_id`.

14. **Add a `Google Sheets` node** and name it `Update Sheet Success`.
    - Connect `Build Success Result` → `Update Sheet Success`.
    - Use the same Google Sheets credential.
    - Set operation to `update`.
    - Set spreadsheet ID to the same file.
    - Set sheet name to `Pins`.
    - Set matching column to `row_id`.
    - Map these fields:
      - `status` ← `={{ $json.status }}`
      - `job_id` ← `={{ $json.job_id }}`
      - `published_at` ← `={{ $json.published_at }}`
      - `error_message` ← `={{ $json.error_message }}`

15. **Add a `Set` node** and name it `Build Invalid Result`.
    - Connect the **false** output of `Validate Required Fields` → `Build Invalid Result`.
    - Add these assignments:
      - `status` = `invalid`
      - `error_message` = `Missing one or more required fields: title, description, link_url, image_url, board_id`
      - `job_id` = empty string
      - `published_at` = empty string
    - As with the success path, ensure `row_id` remains present for the update node.

16. **Add another `Google Sheets` node** and name it `Update Sheet Invalid Row`.
    - Connect `Build Invalid Result` → `Update Sheet Invalid Row`.
    - Use the same Google Sheets credential.
    - Set operation to `update`.
    - Set spreadsheet ID to the same file.
    - Set sheet name to `Pins`.
    - Set matching column to `row_id`.
    - Map:
      - `status` ← `={{ $json.status }}`
      - `job_id` ← `={{ $json.job_id }}`
      - `published_at` ← `={{ $json.published_at }}`
      - `error_message` ← `={{ $json.error_message }}`

17. **Optionally add sticky notes** to document the workflow in the canvas.
    Recommended notes:
    - Overview of the full flow
    - Setup checklist
    - Read/filter explanation
    - Valid rows explanation
    - Invalid rows explanation
    - Scope clarification

18. **Replace all placeholders.**
    - Replace `YOUR_GOOGLE_SHEET_ID`
    - Replace `YOUR_PINTEREST_ACCOUNT_ID`

19. **Test with a small batch first.**
    - Add a few rows:
      - one fully valid row
      - one invalid row missing a required field
      - one row with `status` set to `published`
    - Run the workflow manually.

20. **Verify expected outcomes.**
    - Published-status rows should be skipped.
    - Invalid rows should be marked `invalid` with the configured error text.
    - Valid rows should be submitted and updated with:
      - `status = submitted`
      - `job_id = returned PinBridge job id`
      - `published_at = execution timestamp`
      - `error_message = ''`

21. **Validate image accessibility.**
    - Ensure `image_url` points to a publicly reachable image file.
    - If image download fails, the workflow will stop on that item unless you add custom error handling.

22. **Validate Pinterest metadata.**
    - Ensure each `board_id` is the real Pinterest board ID for the target account.
    - Ensure the account ID belongs to the same Pinterest environment or workspace expected by PinBridge.

23. **Be aware of current scope constraints.**
    - This workflow does not process webhooks.
    - It does not confirm final publication on Pinterest.
    - It does not implement retries, reporting, or notifications.

24. **If needed, extend the workflow for production use.**
    - Add explicit error branches using error triggers or continue-on-fail patterns.
    - Add URL validation and trimming.
    - Add a webhook-based follow-up workflow for final publish confirmation.
    - Add status values like `processing`, `failed`, or `retry_pending`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow uses Google Sheets as the input queue, n8n as the orchestration layer, and PinBridge as the Pinterest publishing layer. | Workflow design context |
| Expected sheet columns: `row_id`, `title`, `description`, `link_url`, `image_url`, `board_id`, `alt_text`, `dominant_color`, `status`, `job_id`, `published_at`, `error_message` | Data model |
| Replace placeholders before use: `YOUR_GOOGLE_SHEET_ID`, `YOUR_PINTEREST_ACCOUNT_ID` | Setup requirement |
| `image_url` must be publicly reachable. | External dependency requirement |
| `row_id` should be unique for reliable row updates. | Data integrity requirement |
| This workflow records successful job submission only. | Scope note |
| A separate workflow should handle webhook callbacks, final publish confirmation, downstream retries, notifications, and reporting. | Recommended architecture |