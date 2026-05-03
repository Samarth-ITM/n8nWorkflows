Monitor APK uploads and run MobSF analysis with OpenAI and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-apk-uploads-and-run-mobsf-analysis-with-openai-and-slack-alerts-14543


# Monitor APK uploads and run MobSF analysis with OpenAI and Slack alerts

# 1. Workflow Overview

This workflow monitors a Google Drive folder for newly uploaded APK files, submits each APK to a MobSF instance for static analysis, extracts package usage information from the analysis output, classifies apparently unused libraries by removal risk, generates a human-readable summary with OpenAI, and posts that summary to Slack.

Its main use cases are:

- Continuous monitoring of APK drops in a shared Drive folder
- Automated Android dependency hygiene review
- Early identification of potentially removable or risky bundled libraries
- Automatic team notification without requiring manual MobSF review

## 1.1 Input Reception and APK Retrieval

The workflow starts with a Google Drive trigger configured to watch a specific folder. When a file is created there, the file is downloaded so it can be passed downstream as binary data.

## 1.2 MobSF Upload and Static Scan

The downloaded APK is uploaded to MobSF through its REST API. Once MobSF returns the uploaded file hash, a second HTTP request triggers static analysis for that APK.

## 1.3 Package Extraction and Unused Dependency Detection

The workflow processes the MobSF response with JavaScript code to derive two lists:

- packages inferred as used from code analysis findings
- packages inferred as detected from Android components and libraries

These lists are then compared to identify packages that appear present but not actively used.

## 1.4 Risk Classification and AI Summary Generation

A second code block classifies unused packages into safe, maybe-required, and risky categories using hardcoded prefix rules. The result is sent to an OpenAI node, which converts it into a concise Slack-ready developer summary.

## 1.5 Team Notification

The final generated message is posted to a designated Slack channel so the development team receives immediate visibility into the findings.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and APK Retrieval

### Overview

This block detects new files in a specific Google Drive folder and downloads the uploaded APK. It serves as the workflow entry point and provides the binary payload required for MobSF upload.

### Nodes Involved

- Start Analysis Trigger
- Download Apk

### Node Details

#### Start Analysis Trigger

- **Type and technical role:** `n8n-nodes-base.googleDriveTrigger`  
  Event-based polling trigger for Google Drive.
- **Configuration choices:**
  - Watches for the `fileCreated` event
  - Polling frequency is every minute
  - Trigger scope is a specific folder
  - Folder watched: `APK Uploads Folder`
- **Key expressions or variables used:**
  - No custom expressions in parameters
- **Input and output connections:**
  - Entry point node
  - Outputs to `Download Apk`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires valid Google Drive OAuth2 credentials
- **Edge cases or potential failure types:**
  - OAuth token expiration or insufficient Drive permissions
  - Folder ID invalid or inaccessible
  - Polling delay means execution is not truly instantaneous
  - If non-APK files are uploaded to the same folder, they may still trigger the workflow unless filtered elsewhere
- **Sub-workflow reference:** None

#### Download Apk

- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads the triggered file contents from Google Drive.
- **Configuration choices:**
  - Operation: `download`
  - File identifier is derived from the trigger payload using the file’s `webViewLink`
- **Key expressions or variables used:**
  - `={{ $json.webViewLink }}`
- **Input and output connections:**
  - Input from `Start Analysis Trigger`
  - Output to `MobSF – Upload APK`
- **Version-specific requirements:**
  - Uses `typeVersion: 3`
  - Requires the same or another valid Google Drive OAuth2 credential
- **Edge cases or potential failure types:**
  - Using `webViewLink` as the file identifier is somewhat fragile; if n8n expects a native file ID, this may fail depending on node behavior/version
  - Download may fail for unsupported file permissions or revoked access
  - Large APK files may increase execution time or memory usage
- **Sub-workflow reference:** None

---

## 2.2 MobSF Upload and Static Scan

### Overview

This block sends the APK binary to a MobSF server and initiates static analysis using the returned file hash. It is the core integration layer between Drive ingestion and security/package analysis.

### Nodes Involved

- MobSF – Upload APK
- MobSF – Start Static Analysis

### Node Details

#### MobSF – Upload APK

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Multipart HTTP POST request to MobSF’s upload endpoint.
- **Configuration choices:**
  - URL: `http://192.168.101.135:8000/api/v1/upload`
  - Method: `POST`
  - Content type: `multipart-form-data`
  - Binary field name sent to MobSF: `file`
  - Input binary property expected from upstream: `data`
  - Authorization header contains a static MobSF API key
- **Key expressions or variables used:**
  - Binary input field: `data`
- **Input and output connections:**
  - Input from `Download Apk`
  - Output to `MobSF – Start Static Analysis`
- **Version-specific requirements:**
  - Uses `typeVersion: 4.3`
  - Requires the upstream node to output binary data under the `data` property
- **Edge cases or potential failure types:**
  - MobSF server unreachable on local network
  - Invalid API key
  - MobSF upload endpoint changed or disabled
  - APK too large or malformed
  - HTTP non-200 responses if MobSF rejects the file
  - Hardcoded internal IP makes portability limited
- **Sub-workflow reference:** None

#### MobSF – Start Static Analysis

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Multipart POST request to trigger MobSF static scanning after upload.
- **Configuration choices:**
  - URL: `http://192.168.101.135:8000/api/v1/scan`
  - Method: `POST`
  - Content type: `multipart-form-data`
  - Body fields:
    - `hash` from previous node response
    - `re_scan` set to `1`
  - Authorization header contains the same static MobSF API key
- **Key expressions or variables used:**
  - `={{ $json.hash }}`
- **Input and output connections:**
  - Input from `MobSF – Upload APK`
  - Output to `Extract used and detected packages`
- **Version-specific requirements:**
  - Uses `typeVersion: 4.3`
- **Edge cases or potential failure types:**
  - Missing or malformed `hash` in upload response
  - Scan endpoint may return partial data, asynchronous status, or error if MobSF needs more time
  - `re_scan=1` forces rescanning, which may increase processing load
  - Network timeout if static analysis is slow
- **Sub-workflow reference:** None

---

## 2.3 Package Extraction and Unused Dependency Detection

### Overview

This block interprets MobSF scan output and derives package-level signals from it. It first extracts used and detected package lists, then computes packages that appear included but not referenced in the code analysis.

### Nodes Involved

- Extract used and detected packages
- Identify Unused Packages

### Node Details

#### Extract used and detected packages

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that parses MobSF scan data.
- **Configuration choices:**
  - Iterates over all incoming items
  - Reads the MobSF report from `item.json.json` if present, otherwise from `item.json`
  - Builds `usedPackages` from `code_analysis.findings[*].files`
  - Builds `detectedPackages` from:
    - `activities`
    - `receivers`
    - `providers`
    - `services`
    - `libraries`
  - Uses heuristic package shortening rules:
    - `com.google...` keeps first 4 segments
    - `androidx...` keeps first 2 segments
    - all others keep first 2 segments
  - Replaces the item JSON with:
    - `usedPackages`
    - `detectedPackages`
- **Key expressions or variables used:**
  - `$input.all()`
  - `item.json.json || item.json`
- **Input and output connections:**
  - Input from `MobSF – Start Static Analysis`
  - Output to `Identify Unused Packages`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Requires JavaScript Code node support as available in current n8n versions
- **Edge cases or potential failure types:**
  - If MobSF response structure differs from expected keys, the output may be empty or misleading
  - Replacing `item.json` discards any previous metadata unless intentionally preserved
  - Heuristic package extraction may overgeneralize or misclassify package ownership
  - If `code_analysis.findings` is absent, used packages will be empty, potentially inflating unused findings
- **Sub-workflow reference:** None

#### Identify Unused Packages

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that compares used and detected package lists.
- **Configuration choices:**
  - Reads the current item using `$input.item.json`
  - Converts `usedPackages` into a Set
  - Marks a detected package as unused if no used package starts with that detected package prefix
  - Returns a single JSON object containing:
    - `usedPackages`
    - `detectedPackages`
    - `unusedPackages`
- **Key expressions or variables used:**
  - `$input.item.json`
  - `Array.from(used).some(u => u.startsWith(pkg))`
- **Input and output connections:**
  - Input from `Extract used and detected packages`
  - Output to `Classify Unused Package Risk Levels`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Prefix matching may produce false positives or false negatives
  - If `usedPackages` is empty because MobSF omitted findings, nearly all detected packages may appear unused
  - Assumes one item context; if item batching changes later, logic may need adjustment
- **Sub-workflow reference:** None

---

## 2.4 Risk Classification and AI Summary Generation

### Overview

This block assigns risk levels to unused packages using simple prefix-based rules, then asks OpenAI to produce a short developer-oriented summary suitable for Slack delivery.

### Nodes Involved

- Classify Unused Package Risk Levels
- Generate Developer Summary

### Node Details

#### Classify Unused Package Risk Levels

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript rule engine for package risk categorization.
- **Configuration choices:**
  - Reads `unusedPackages`
  - Categorizes libraries into:
    - `riskyToRemove`
    - `maybeRequired`
    - `safeToRemove`
  - Current hardcoded rules:
    - Risky:
      - `com.google.android.gms`
      - `com.google.firebase`
      - `androidx.startup`
      - `com.google.android.datatransport`
    - Maybe:
      - `com.jakewharton`
      - `androidx.profileinstaller`
    - Everything else defaults to safe
- **Key expressions or variables used:**
  - `$json.unusedPackages || []`
- **Input and output connections:**
  - Input from `Identify Unused Packages`
  - Output to `Generate Developer Summary`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Rule set is static and incomplete; many libraries may be oversimplified
  - Defaulting unknown packages to safe may be risky in real production environments
  - No external package metadata or manifest-level dependency validation is performed
- **Sub-workflow reference:** None

#### Generate Developer Summary

- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  OpenAI chat/completion-style node used to produce a plain-text summary.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Prompt instructs the model to behave as an Android performance and security expert
  - Injects:
    - `unusedPackages`
    - `safeToRemove`
    - `maybeRequired`
    - `riskyToRemove`
  - Requires output to include:
    - number of unused packages
    - safe-to-remove packages
    - needs-review packages
    - high-risk packages
    - short recommendation
    - no markdown
  - Returns only the summary message
- **Key expressions or variables used:**
  - `{{JSON.stringify($json.unusedPackages, null, 2)}}`
  - `{{JSON.stringify($json.safeToRemove, null, 2)}}`
  - `{{JSON.stringify($json.maybeRequired, null, 2)}}`
  - `{{JSON.stringify($json.riskyToRemove, null, 2)}}`
- **Input and output connections:**
  - Input from `Classify Unused Package Risk Levels`
  - Output to `Notify Team on Slack`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Requires OpenAI credentials configured in n8n
  - Depends on LangChain/OpenAI node availability in the instance
- **Edge cases or potential failure types:**
  - Invalid or missing OpenAI API credentials
  - Model availability or quota issues
  - Output format may vary slightly despite prompt constraints
  - Token limits could matter if package lists become very large
- **Sub-workflow reference:** None

---

## 2.5 Team Notification

### Overview

This final block publishes the AI-generated summary into a Slack channel. It closes the loop by making the analysis actionable for developers.

### Nodes Involved

- Notify Team on Slack

### Node Details

#### Notify Team on Slack

- **Type and technical role:** `n8n-nodes-base.slack`  
  Slack message sender node.
- **Configuration choices:**
  - Sends a text message to a selected channel
  - Channel configured as `n8n`
  - Message body is taken from the OpenAI output structure
- **Key expressions or variables used:**
  - `={{ $json.output[0].content[0].text }}`
- **Input and output connections:**
  - Input from `Generate Developer Summary`
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
  - Requires Slack API credentials with permission to post to the selected channel
- **Edge cases or potential failure types:**
  - If the OpenAI node returns a different schema, the expression may fail
  - Missing channel access or Slack permission issues
  - Slack API rate limiting
  - Message length constraints if output becomes too long
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Analysis Trigger | googleDriveTrigger | Watches a Google Drive folder for newly created files |  | Download Apk | ## File Detection & APK Retrieval<br>This section monitors a specific Google Drive folder for new APK uploads. When a file appears, the workflow automatically grabs its details and downloads the APK so it can be processed further. This ensures the analysis begins instantly.) |
| Download Apk | googleDrive | Downloads the uploaded APK from Google Drive | Start Analysis Trigger | MobSF – Upload APK | ## File Detection & APK Retrieval<br>This section monitors a specific Google Drive folder for new APK uploads. When a file appears, the workflow automatically grabs its details and downloads the APK so it can be processed further. This ensures the analysis begins instantly.) |
| MobSF – Upload APK | httpRequest | Uploads the APK binary to MobSF | Download Apk | MobSF – Start Static Analysis | ## Upload & Trigger MobSF Static Scan<br>After downloading, the APK is uploaded to MobSF. Once uploaded successfully, the workflow immediately triggers a static analysis scan using the provided hash, ensuring a smooth handoff between file upload and security scanning steps. |
| MobSF – Start Static Analysis | httpRequest | Starts MobSF static analysis using the uploaded APK hash | MobSF – Upload APK | Extract used and detected packages | ## Upload & Trigger MobSF Static Scan<br>After downloading, the APK is uploaded to MobSF. Once uploaded successfully, the workflow immediately triggers a static analysis scan using the provided hash, ensuring a smooth handoff between file upload and security scanning steps. |
| Extract used and detected packages | code | Extracts package lists from the MobSF response | MobSF – Start Static Analysis | Identify Unused Packages | ## Extract, Compare & Classify Package<br>This section processes the MobSF report using JavaScript to extract all detected and used packages. It compares both lists to identify unused libraries and then classifies them into safe, maybe-required and high-risk categories. This creates a clear, structured understanding of dependency usage in the app. |
| Identify Unused Packages | code | Compares detected packages with used packages to find likely unused libraries | Extract used and detected packages | Classify Unused Package Risk Levels | ## Extract, Compare & Classify Package<br>This section processes the MobSF report using JavaScript to extract all detected and used packages. It compares both lists to identify unused libraries and then classifies them into safe, maybe-required and high-risk categories. This creates a clear, structured understanding of dependency usage in the app. |
| Classify Unused Package Risk Levels | code | Assigns unused packages into safe, maybe, and risky categories | Identify Unused Packages | Generate Developer Summary | ## Extract, Compare & Classify Package<br>This section processes the MobSF report using JavaScript to extract all detected and used packages. It compares both lists to identify unused libraries and then classifies them into safe, maybe-required and high-risk categories. This creates a clear, structured understanding of dependency usage in the app. |
| Generate Developer Summary | @n8n/n8n-nodes-langchain.openAi | Generates a plain-language analysis summary using OpenAI | Classify Unused Package Risk Levels | Notify Team on Slack | ## Generate & Deliver Developer Summary<br>This section creates a clean, developer-friendly summary using AI and instantly sends it to the team’s Slack channel. It ensures clear communication of unused package findings and helps developers quickly review risks and decide next steps without manual effort.) |
| Notify Team on Slack | slack | Posts the generated summary to Slack | Generate Developer Summary |  | ## Generate & Deliver Developer Summary<br>This section creates a clean, developer-friendly summary using AI and instantly sends it to the team’s Slack channel. It ensures clear communication of unused package findings and helps developers quickly review risks and decide next steps without manual effort.) |
| Sticky Note | stickyNote | Visual documentation for the Drive trigger and download section |  |  | ## File Detection & APK Retrieval<br>This section monitors a specific Google Drive folder for new APK uploads. When a file appears, the workflow automatically grabs its details and downloads the APK so it can be processed further. This ensures the analysis begins instantly.) |
| Sticky Note1 | stickyNote | Visual documentation for MobSF upload and scan section |  |  | ## Upload & Trigger MobSF Static Scan<br>After downloading, the APK is uploaded to MobSF. Once uploaded successfully, the workflow immediately triggers a static analysis scan using the provided hash, ensuring a smooth handoff between file upload and security scanning steps. |
| Sticky Note2 | stickyNote | Visual documentation for package extraction and classification section |  |  | ## Extract, Compare & Classify Package<br>This section processes the MobSF report using JavaScript to extract all detected and used packages. It compares both lists to identify unused libraries and then classifies them into safe, maybe-required and high-risk categories. This creates a clear, structured understanding of dependency usage in the app. |
| Sticky Note3 | stickyNote | Visual documentation for AI summary and Slack delivery |  |  | ## Generate & Deliver Developer Summary<br>This section creates a clean, developer-friendly summary using AI and instantly sends it to the team’s Slack channel. It ensures clear communication of unused package findings and helps developers quickly review risks and decide next steps without manual effort.) |
| Sticky Note4 | stickyNote | Global workflow documentation and setup guidance |  |  | ## How It Works<br>#### This workflow automatically analyzes newly uploaded APK files and generates a clean summary for the development team. It begins by monitoring a Google Drive folder for new APKs. When a file is added, the workflow downloads it and sends it to MobSF for static analysis. MobSF returns detailed findings, which are processed through JavaScript nodes to extract used, detected and unused packages. These packages are then evaluated and categorized into safe, maybe-required and risky groups. After processing, an AI model turns the results into a simple message that’s easy for developers to understand. Finally, the summary is automatically sent to a Slack channel, ensuring the team gets instant visibility into any unnecessary or risky dependencies.<br><br>## Setup Steps<br><br>### Connect Google Drive: Add your Google Drive credentials and specify the folder to watch for APK uploads.<br><br>### MobSF Configuration: Ensure your MobSF server URL and API key are correctly set in the HTTP Request nodes for upload and scan.<br><br>### JavaScript Nodes: The code is already included—no setup required unless you want to modify the logic.<br><br>### OpenAI Connection: Add your OpenAI API key to the “Generate Developer Summary” node.<br><br>### Slack Integration: Connect your Slack account and select the target channel.<br><br>### Enable the Workflow: Activate the workflow so it starts monitoring and processing files automatically. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `APK Upload Monitoring and Automated MobSF Analysis with Slack Reporting`.

2. **Add a Google Drive Trigger node**
   - Node type: `Google Drive Trigger`
   - Name: `Start Analysis Trigger`
   - Event: `File Created`
   - Trigger scope: `Specific Folder`
   - Select the folder to monitor, for example your APK uploads folder
   - Polling frequency: every minute
   - Attach Google Drive OAuth2 credentials with access to the target folder

3. **Add a Google Drive node for file download**
   - Node type: `Google Drive`
   - Name: `Download Apk`
   - Operation: `Download`
   - Set the file identifier from the trigger payload
   - In the source workflow, the value is `={{ $json.webViewLink }}`
   - Prefer validating whether your n8n version accepts this; in many setups the actual file ID is safer than a web link
   - Use the same Google Drive OAuth2 credentials
   - Connect `Start Analysis Trigger` → `Download Apk`

4. **Add an HTTP Request node to upload the APK to MobSF**
   - Node type: `HTTP Request`
   - Name: `MobSF – Upload APK`
   - Method: `POST`
   - URL: `http://<your-mobsf-host>:8000/api/v1/upload`
   - Enable headers
   - Add header:
     - `Authorization: <your_mobsf_api_key>`
   - Enable request body
   - Content type: `Multipart Form-Data`
   - Add body parameter:
     - Name: `file`
     - Type: binary/form binary
     - Input data field name: `data`
   - Ensure the incoming binary from `Download Apk` is available under `data`
   - Connect `Download Apk` → `MobSF – Upload APK`

5. **Add an HTTP Request node to start the static scan**
   - Node type: `HTTP Request`
   - Name: `MobSF – Start Static Analysis`
   - Method: `POST`
   - URL: `http://<your-mobsf-host>:8000/api/v1/scan`
   - Enable headers
   - Add header:
     - `Authorization: <your_mobsf_api_key>`
   - Enable request body
   - Content type: `Multipart Form-Data`
   - Add body parameters:
     - `hash` = `={{ $json.hash }}`
     - `re_scan` = `1`
   - Connect `MobSF – Upload APK` → `MobSF – Start Static Analysis`

6. **Add a Code node to extract used and detected packages**
   - Node type: `Code`
   - Name: `Extract used and detected packages`
   - Language: JavaScript
   - Paste the following logic conceptually:
     - Iterate all items
     - Read MobSF report from `item.json.json` or fallback to `item.json`
     - Extract used packages from `code_analysis.findings[*].files`
     - Extract detected packages from `activities`, `receivers`, `providers`, `services`, and `libraries`
     - Normalize package prefixes with heuristics
     - Output only:
       - `usedPackages`
       - `detectedPackages`
   - Use the exact script from the workflow if you want behavior parity
   - Connect `MobSF – Start Static Analysis` → `Extract used and detected packages`

7. **Add a Code node to identify unused packages**
   - Node type: `Code`
   - Name: `Identify Unused Packages`
   - Language: JavaScript
   - Logic:
     - Read `usedPackages` and `detectedPackages`
     - For each detected package, mark it unused when no entry in `usedPackages` starts with that package
     - Return:
       - `usedPackages`
       - `detectedPackages`
       - `unusedPackages`
   - Connect `Extract used and detected packages` → `Identify Unused Packages`

8. **Add a Code node to classify package risk**
   - Node type: `Code`
   - Name: `Classify Unused Package Risk Levels`
   - Language: JavaScript
   - Logic:
     - Input: `unusedPackages`
     - If package starts with:
       - `com.google.android.gms`
       - `com.google.firebase`
       - `androidx.startup`
       - `com.google.android.datatransport`
       then add to `riskyToRemove`
     - If package starts with:
       - `com.jakewharton`
       - `androidx.profileinstaller`
       then add to `maybeRequired`
     - Otherwise add to `safeToRemove`
     - Return:
       - `unusedPackages`
       - `safeToRemove`
       - `maybeRequired`
       - `riskyToRemove`
   - Connect `Identify Unused Packages` → `Classify Unused Package Risk Levels`

9. **Add an OpenAI node**
   - Node type: `OpenAI` via `@n8n/n8n-nodes-langchain.openAi`
   - Name: `Generate Developer Summary`
   - Configure OpenAI credentials with a valid API key
   - Model: `gpt-4o-mini`
   - Add a prompt that includes:
     - unused packages
     - safe-to-remove packages
     - maybe-required packages
     - risky-to-remove packages
   - In the source workflow, the prompt explicitly asks for:
     - number of unused packages
     - safe-to-remove list
     - needs-review list
     - high-risk list
     - 2–3 line recommendation
     - no markdown
     - only the final summary message
   - Use expressions such as:
     - `{{JSON.stringify($json.unusedPackages, null, 2)}}`
     - `{{JSON.stringify($json.safeToRemove, null, 2)}}`
     - `{{JSON.stringify($json.maybeRequired, null, 2)}}`
     - `{{JSON.stringify($json.riskyToRemove, null, 2)}}`
   - Connect `Classify Unused Package Risk Levels` → `Generate Developer Summary`

10. **Add a Slack node**
    - Node type: `Slack`
    - Name: `Notify Team on Slack`
    - Operation: send a message to a channel
    - Select the destination channel
    - Configure Slack credentials with permission to post
    - Set the message text to the OpenAI response
    - In the source workflow, the expression is:
      - `={{ $json.output[0].content[0].text }}`
    - Validate this path in your n8n version because LangChain/OpenAI output structures can differ
    - Connect `Generate Developer Summary` → `Notify Team on Slack`

11. **Optionally add sticky notes for maintainability**
    - Add one note for Drive monitoring and download
    - Add one note for MobSF upload and scanning
    - Add one note for extraction/comparison/classification
    - Add one note for OpenAI summary and Slack notification
    - Add one global note for setup instructions

12. **Configure credentials**
    - **Google Drive OAuth2**
      - Must allow reading the target folder and downloading files
    - **MobSF**
      - No dedicated credential object is used here; the API key is placed in the HTTP header
      - Consider moving this to a credential or environment variable for safer management
    - **OpenAI**
      - Add API key in n8n credentials
    - **Slack**
      - Use a bot or app token that can post to the chosen channel

13. **Test each stage**
    - Upload a small APK into the watched folder
    - Confirm trigger execution
    - Confirm `Download Apk` produces binary data
    - Confirm MobSF upload returns a `hash`
    - Confirm static scan response includes the expected JSON fields
    - Confirm the code nodes output package arrays
    - Confirm the OpenAI node returns a summary
    - Confirm Slack receives the message

14. **Activate the workflow**
    - Once validation succeeds, switch the workflow to active so polling begins automatically

## Reproduction Notes and Constraints

- The workflow has **one entry point**: `Start Analysis Trigger`.
- It uses **no sub-workflows**.
- The workflow assumes MobSF scan results are returned in a structure directly consumable by the code nodes. If your MobSF deployment returns only job metadata or delayed results, you may need to insert a polling/wait stage before parsing.
- For production use, consider adding:
  - file type validation before upload
  - error handling branches
  - retries for MobSF and OpenAI
  - timeout controls
  - logging or audit storage
  - deduplication if the same APK is reuploaded

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically analyzes newly uploaded APK files and generates a clean summary for the development team. It monitors a Google Drive folder, downloads APKs, submits them to MobSF, processes the findings with JavaScript, generates an AI summary, and posts the result to Slack. | Global workflow behavior |
| Connect Google Drive credentials and specify the folder to watch for APK uploads. | Setup guidance |
| Ensure the MobSF server URL and API key are configured correctly in the HTTP Request nodes for upload and scan. | Setup guidance |
| The JavaScript logic is already embedded in the Code nodes and only needs changes if you want to alter extraction or classification behavior. | Setup guidance |
| Add OpenAI credentials to the `Generate Developer Summary` node. | Setup guidance |
| Connect Slack credentials and choose the target channel. | Setup guidance |
| Activate the workflow after configuration so it starts monitoring and processing files automatically. | Operational guidance |

## Additional implementation observations

- The workflow title provided by the user and the internal workflow name are slightly different, but they describe the same process.
- The MobSF API key is hardcoded directly in the HTTP Request nodes; for security, this should be externalized.
- The Google Drive download step uses `webViewLink` as the file reference. In many environments, the file ID is a more reliable choice.
- The classification logic is heuristic, not authoritative dependency analysis. Results should be reviewed before removing libraries from a production Android build.