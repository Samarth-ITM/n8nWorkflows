Qualify lead lists and find professional emails with OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/qualify-lead-lists-and-find-professional-emails-with-openai-and-google-sheets-14198


# Qualify lead lists and find professional emails with OpenAI and Google Sheets

# 1. Workflow Overview

This workflow monitors a specific Google Drive folder for newly created Google Sheets lead files, reads the leads from the first sheet, normalizes each row into a consistent structure, asks OpenAI to identify a professional email address for each lead, and then separates the results into **Qualified** and **Unqualified** tabs in the same spreadsheet.

Typical use cases:
- Enriching imported lead lists with likely professional email addresses
- Filtering leads into usable and unusable outreach targets
- Creating a semi-automated outbound prospecting preparation flow

## 1.1 Input Reception and Lead Ingestion
The workflow starts when a new spreadsheet appears in a watched Google Drive folder. It reads rows from the default first sheet and reshapes the source columns into a standard schema used by the rest of the workflow.

## 1.2 Lead Processing and AI Email Discovery
Each normalized lead is processed in small batches. OpenAI is called for each item to return a structured result containing `email`, `source`, and `confidence`. A wait step is inserted to reduce rate-limit pressure.

## 1.3 Result Parsing and Routing
The AI response is parsed from text into fields. The workflow then checks whether the result appears to contain a valid email address and routes the lead into either the **Qualified** or **Unqualified** sheet tab.

## 1.4 Documentation and In-Canvas Guidance
Several sticky notes explain setup requirements, workflow purpose, and configuration expectations directly in the canvas.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Lead Ingestion

### Overview
This block detects new lead spreadsheets in Google Drive, reads the rows from the file, and maps the incoming columns into a normalized structure. It establishes the canonical field names used downstream.

### Nodes Involved
- Watch For New Leads
- Read Lead Sheet
- Normalize Lead Data

### Node Details

#### Watch For New Leads
- **Type and technical role:** `n8n-nodes-base.googleDriveTrigger`  
  Polling trigger that watches a specific Google Drive folder for newly created files.
- **Configuration choices:**
  - Event: `fileCreated`
  - Trigger scope: specific folder
  - Polling frequency: every minute
  - Folder is referenced by explicit Google Drive folder ID
- **Key expressions or variables used:**
  - Emits file metadata including `id`, which is later used as the Google Sheets document ID
- **Input and output connections:**
  - Entry point node
  - Outputs to: `Read Lead Sheet`
- **Version-specific requirements:**
  - Type version: `1`
  - Requires valid Google Drive OAuth2 credentials
- **Edge cases or potential failure types:**
  - Wrong folder ID
  - Missing permissions to the watched folder
  - Trigger polling delays
  - Non-spreadsheet files created in the folder may cause downstream Google Sheets read failures if not filtered elsewhere
- **Sub-workflow reference:** None

#### Read Lead Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the newly detected Google Sheets file.
- **Configuration choices:**
  - Document ID comes from the trigger file ID: `{{ $json.id }}`
  - Sheet selected: `gid=0` / first worksheet (`Sheet1` cached name)
  - No additional options configured
- **Key expressions or variables used:**
  - `={{ $json.id }}`
- **Input and output connections:**
  - Input from: `Watch For New Leads`
  - Output to: `Normalize Lead Data`
- **Version-specific requirements:**
  - Type version: `4.7`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Triggered file is not a Google Sheet
  - Target sheet is missing or first tab is not the intended input tab
  - Column headers do not match expected names
  - Permission mismatch between Drive and Sheets credentials
- **Sub-workflow reference:** None

#### Normalize Lead Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Transforms source row fields into standardized field names.
- **Configuration choices:**
  - Creates:
    - `fullName`
    - `companyName`
    - `companyWebsite`
    - `linkedinIndustry`
    - `linkedinUrl`
    - `jobTitle`
    - `location`
  - `fullName` is built by concatenating first and last name
- **Key expressions or variables used:**
  - `={{ ($json['first name'] ?? '') + ' ' + ($json['last name'] ?? '') }}`
  - `={{ $json.company }}`
  - `={{ $json['corporate website'] }}`
  - `={{ $json['linkedin industry'] }}`
  - `={{ $json['linkedin url'] }}`
  - `={{ $json['job title'] }}`
  - `={{ $json.location }}`
- **Input and output connections:**
  - Input from: `Read Lead Sheet`
  - Output to: `Batch Leads`
- **Version-specific requirements:**
  - Type version: `3.4`
- **Edge cases or potential failure types:**
  - Missing input headers produce blank strings or undefined-derived values
  - `fullName` may contain extra whitespace if one of the names is missing
  - Unexpected column capitalization or alternate header names will break mappings
- **Sub-workflow reference:** None

---

## 2.2 Lead Processing and AI Email Discovery

### Overview
This block iterates through leads in small batches, submits each lead to OpenAI, and introduces a delay to reduce API throttling risk. The design supports looping until all rows have been processed.

### Nodes Involved
- Batch Leads
- Find Email via OpenAI
- Rate Limit Delay

### Node Details

#### Batch Leads
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Controls iterative processing of the lead list in batches.
- **Configuration choices:**
  - Batch size: `3`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Normalize Lead Data`
  - Loop input from: `Parse Email Results`
  - Output 0 to: `Has Valid Email?`
  - Output 1 to: `Find Email via OpenAI`
- **Interpretation of wiring:**
  - The node is being used as a loop controller
  - After results are parsed, execution returns to this node to continue processing subsequent items
  - The presence of routing from output 0 to `Has Valid Email?` means the current batch/item state is also used to trigger qualification logic
- **Version-specific requirements:**
  - Type version: `3`
- **Edge cases or potential failure types:**
  - Confusion about branch semantics when manually rebuilding
  - If the loop is miswired, only the first batch may execute or routing may occur before parsed AI output is available
  - Batch size too large can increase OpenAI rate-limit pressure
- **Sub-workflow reference:** None

#### Find Email via OpenAI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Calls OpenAI to generate an email-finding result.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Retries enabled
  - Wait between retries: 5000 ms
  - No built-in tools configured
  - Response schema not strongly constrained in this export, so the node returns text
- **Key expressions or variables used:**
  - No explicit prompt is present in the exported node data
  - In practical use, this node must contain instructions telling the model to return JSON with at least:
    - `email`
    - `source`
    - `confidence`
- **Input and output connections:**
  - Input from: `Batch Leads`
  - Output to: `Rate Limit Delay`
- **Version-specific requirements:**
  - Type version: `2.1`
  - Requires OpenAI credentials
  - Depends on LangChain/OpenAI node availability in the n8n instance
- **Edge cases or potential failure types:**
  - Missing prompt or insufficiently strict prompt causes unparseable output
  - OpenAI authentication failure
  - Model unavailability
  - Rate limits
  - Hallucinated or low-confidence email addresses
  - Output enclosed in markdown fences, which is partially handled later
- **Sub-workflow reference:** None

#### Rate Limit Delay
- **Type and technical role:** `n8n-nodes-base.wait`  
  Inserts a fixed pause between AI calls / loop progression.
- **Configuration choices:**
  - Wait amount: `1.5`
  - `alwaysOutputData: true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Find Email via OpenAI`
  - Output to: `Parse Email Results`
- **Version-specific requirements:**
  - Type version: `1.1`
- **Edge cases or potential failure types:**
  - Misunderstanding of time unit in UI when recreating manually
  - Long-running executions if processing many leads
  - If self-hosted execution pruning is strict, waits may affect operational behavior
- **Sub-workflow reference:** None

---

## 2.3 Result Parsing and Routing

### Overview
This block parses the OpenAI response, merges it with the normalized lead fields, checks whether the returned email looks valid, and appends the record to the appropriate tab.

### Nodes Involved
- Parse Email Results
- Has Valid Email?
- Save Qualified Lead
- Save Unqualified Lead

### Node Details

#### Parse Email Results
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts AI-produced fields and rebuilds a final record containing both source lead data and enrichment output.
- **Configuration choices:**
  - Rehydrates original normalized fields by referencing `Normalize Lead Data`
  - Parses JSON out of `$json.text`
  - Strips markdown code fences before parsing
  - Extracts:
    - `email`
    - `source`
    - `confidence`
- **Key expressions or variables used:**
  - `={{ $('Normalize Lead Data').item.json.fullName }}`
  - `={{ $('Normalize Lead Data').item.json.companyName }}`
  - `={{ $('Normalize Lead Data').item.json.companyWebsite }}`
  - `={{ $('Normalize Lead Data').item.json.linkedinIndustry }}`
  - `={{ $('Normalize Lead Data').item.json.linkedinUrl }}`
  - `={{ $('Normalize Lead Data').item.json.jobTitle }}`
  - `={{ $('Normalize Lead Data').item.json.location }}`
  - `={{ JSON.parse($json.text.replace(/```json|```/g, '').trim()).email }}`
  - `={{ JSON.parse($json.text.replace(/```json|```/g, '').trim()).source }}`
  - `={{ JSON.parse($json.text.replace(/```json|```/g, '').trim()).confidence }}`
- **Input and output connections:**
  - Input from: `Rate Limit Delay`
  - Output to: `Batch Leads` for loop continuation
- **Version-specific requirements:**
  - Type version: `3.4`
  - Retries enabled
- **Edge cases or potential failure types:**
  - Invalid JSON from OpenAI causes `JSON.parse` failure
  - Missing `text` field depending on OpenAI node output mode
  - AI returns extra prose before or after JSON
  - Empty response
  - Item linking can fail if node execution order or pairing changes
- **Sub-workflow reference:** None

#### Has Valid Email?
- **Type and technical role:** `n8n-nodes-base.if`  
  Performs a simple email validity gate based on the presence of `@`.
- **Configuration choices:**
  - Condition checks whether `$('Parse Email Results').item.json.email` contains `@`
  - Strict type validation enabled
- **Key expressions or variables used:**
  - `={{ $('Parse Email Results').item.json.email }}`
- **Input and output connections:**
  - Input from: `Batch Leads`
  - Output 0 (true) to: `Save Qualified Lead`
  - Output 1 (false) to: `Save Unqualified Lead`
- **Version-specific requirements:**
  - Type version: `2.3`
- **Edge cases or potential failure types:**
  - This is only a superficial validation
  - Strings like `unknown@unknown` would pass
  - Empty or null values may fail strict evaluation depending on runtime content
  - Because it references another node rather than direct input fields, item-pairing consistency matters
- **Sub-workflow reference:** None

#### Save Qualified Lead
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends qualified leads to the `Qualified` tab of the same spreadsheet that triggered the workflow.
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `Qualified`
  - Document ID: from trigger file ID via `Watch For New Leads`
  - Explicit field mapping for:
    - `fullName`
    - `companyName`
    - `companyWebsite`
    - `linkedinUrl`
    - `jobTitle`
    - `location`
    - `email`
    - `source`
    - `confidence`
- **Key expressions or variables used:**
  - `={{ $('Parse Email Results').item.json.email }}`
  - `={{ $('Parse Email Results').item.json.source }}`
  - `={{ $('Parse Email Results').item.json.fullName }}`
  - `={{ $('Parse Email Results').item.json.jobTitle }}`
  - `={{ $('Parse Email Results').item.json.location }}`
  - `={{ $('Parse Email Results').item.json.confidence }}`
  - `={{ $('Parse Email Results').item.json.companyName }}`
  - `={{ $('Parse Email Results').item.json.linkedinUrl }}`
  - `={{ $('Parse Email Results').item.json.companyWebsite }}`
  - `={{ $('Watch For New Leads').item.json.id }}`
- **Input and output connections:**
  - Input from: `Has Valid Email?` true branch
  - No downstream connection
- **Version-specific requirements:**
  - Type version: `4.7`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Missing `Qualified` tab
  - Header mismatch in target sheet
  - Permission issue on destination spreadsheet
  - Append may still succeed with partial blanks if some parsed fields are empty
- **Sub-workflow reference:** None

#### Save Unqualified Lead
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends unqualified leads to the `Unqualified` tab in the same spreadsheet.
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `Unqualified`
  - Same mapped fields as the qualified branch
- **Key expressions or variables used:**
  - Same field references as `Save Qualified Lead`
  - Document ID: `={{ $('Watch For New Leads').item.json.id }}`
- **Input and output connections:**
  - Input from: `Has Valid Email?` false branch
  - No downstream connection
- **Version-specific requirements:**
  - Type version: `4.7`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Missing `Unqualified` tab
  - Header mismatch
  - Permission/authentication errors
- **Sub-workflow reference:** None

---

## 2.4 Documentation and In-Canvas Guidance

### Overview
These sticky notes provide operational guidance, setup instructions, and visual grouping of the workflow into functional sections.

### Nodes Involved
- Main Description
- Section - Ingest
- Section - Find Emails
- Section - Route Results
- Warning

### Node Details

#### Main Description
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  High-level description and setup checklist.
- **Configuration choices:**
  - Large descriptive note covering purpose, process, setup, and customization
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Section - Ingest
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Section label for ingestion nodes
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Section - Find Emails
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Section label for batching, AI lookup, and parsing nodes
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Section - Route Results
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Section label for validation and sheet routing nodes
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Warning
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Visual reminder to configure credentials and folder ID
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Watch For New Leads | googleDriveTrigger | Watches a Google Drive folder for newly created lead spreadsheets |  | Read Lead Sheet | ### Ingest leads<br>Watches a Google Drive folder for new spreadsheets, reads all rows, and normalizes fields into a consistent format.<br>**Dont forget to** Connect your Google Drive, Sheets, and OpenAI credentials and update the folder ID in the trigger node before activating this workflow. |
| Read Lead Sheet | googleSheets | Reads rows from the newly created spreadsheet | Watch For New Leads | Normalize Lead Data | ### Ingest leads<br>Watches a Google Drive folder for new spreadsheets, reads all rows, and normalizes fields into a consistent format. |
| Normalize Lead Data | set | Maps source sheet columns into normalized lead fields | Read Lead Sheet | Batch Leads | ### Ingest leads<br>Watches a Google Drive folder for new spreadsheets, reads all rows, and normalizes fields into a consistent format. |
| Batch Leads | splitInBatches | Iterates through leads in batches and controls the processing loop | Normalize Lead Data, Parse Email Results | Has Valid Email?, Find Email via OpenAI | ### Find & parse emails<br>Processes leads in batches of 3 with rate limiting. OpenAI searches for a verified professional email and returns email, source, and confidence score. |
| Find Email via OpenAI | @n8n/n8n-nodes-langchain.openAi | Calls OpenAI to identify a professional email and metadata | Batch Leads | Rate Limit Delay | ### Find & parse emails<br>Processes leads in batches of 3 with rate limiting. OpenAI searches for a verified professional email and returns email, source, and confidence score. |
| Rate Limit Delay | wait | Slows execution between AI calls to reduce rate-limit issues | Find Email via OpenAI | Parse Email Results | ### Find & parse emails<br>Processes leads in batches of 3 with rate limiting. OpenAI searches for a verified professional email and returns email, source, and confidence score. |
| Parse Email Results | set | Parses AI JSON output and rebuilds enriched lead objects | Rate Limit Delay | Batch Leads | ### Find & parse emails<br>Processes leads in batches of 3 with rate limiting. OpenAI searches for a verified professional email and returns email, source, and confidence score. |
| Has Valid Email? | if | Routes items based on whether the parsed email contains @ | Batch Leads | Save Qualified Lead, Save Unqualified Lead | ### Route results<br>Checks if a valid email was found. Qualified leads (with email) and unqualified leads (without) are written to separate tabs on the same sheet. |
| Save Qualified Lead | googleSheets | Appends qualified leads to the Qualified tab | Has Valid Email? |  | ### Route results<br>Checks if a valid email was found. Qualified leads (with email) and unqualified leads (without) are written to separate tabs on the same sheet. |
| Save Unqualified Lead | googleSheets | Appends unqualified leads to the Unqualified tab | Has Valid Email? |  | ### Route results<br>Checks if a valid email was found. Qualified leads (with email) and unqualified leads (without) are written to separate tabs on the same sheet. |
| Main Description | stickyNote | Canvas documentation for workflow purpose and setup |  |  | ## Qualify Lead Lists & Find Professional Emails<br><br>Drop a lead list into Google Drive. This workflow reads every row, uses OpenAI to find professional emails, and splits leads into Qualified and Unqualified tabs automatically.<br><br>### How it works<br><br>1. Google Drive trigger detects a new spreadsheet in your watched folder<br>2. Reads all rows and normalizes fields (name, company, title, LinkedIn, location)<br>3. Sends each lead to OpenAI in batches to find a verified professional email<br>4. Validates results -- leads with a valid email go to "Qualified", the rest to "Unqualified"<br><br>### Setup<br><br>- [ ] Connect Google Drive and Google Sheets OAuth2 credentials<br>- [ ] Set your watched folder ID in the trigger node<br>- [ ] Connect your OpenAI API key<br>- [ ] Input sheet columns: first name, last name, company, corporate website, linkedin industry, linkedin url, job title, location<br>- [ ] Create "Qualified" and "Unqualified" tabs with columns: fullName, companyName, companyWebsite, linkedinUrl, jobTitle, location, email, source, confidence<br><br>### Customization<br><br>- Adjust batch size (default 3) and delay (default 1.5s) for your API tier<br>- Swap GPT-4o for a cheaper model on high-volume lists |
| Section - Ingest | stickyNote | Section label for ingestion area |  |  | ### Ingest leads<br>Watches a Google Drive folder for new spreadsheets, reads all rows, and normalizes fields into a consistent format. |
| Section - Find Emails | stickyNote | Section label for AI enrichment area |  |  | ### Find & parse emails<br>Processes leads in batches of 3 with rate limiting. OpenAI searches for a verified professional email and returns email, source, and confidence score. |
| Section - Route Results | stickyNote | Section label for routing area |  |  | ### Route results<br>Checks if a valid email was found. Qualified leads (with email) and unqualified leads (without) are written to separate tabs on the same sheet. |
| Warning | stickyNote | Warning note for required credentials and folder setup |  |  | **Dont forget to** Connect your Google Drive, Sheets, and OpenAI credentials and update the folder ID in the trigger node before activating this workflow. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Qualify Lead Lists And Find Professional Emails With OpenAI and Google Sheets`.

2. **Add a Google Drive Trigger node**
   - Node type: `Google Drive Trigger`
   - Name: `Watch For New Leads`
   - Event: `File Created`
   - Trigger on: `Specific Folder`
   - Folder to watch: paste the target Google Drive folder ID
   - Polling: every minute
   - Credentials: connect a Google Drive OAuth2 account with access to that folder

3. **Add a Google Sheets node to read rows**
   - Node type: `Google Sheets`
   - Name: `Read Lead Sheet`
   - Connect `Watch For New Leads -> Read Lead Sheet`
   - Set the document ID to an expression using the trigger file ID:
     - `{{ $json.id }}`
   - Select the first worksheet, equivalent to `gid=0`
   - Operation should be the standard row read mode for fetching all rows
   - Credentials: connect a Google Sheets OAuth2 account with access to the spreadsheet

4. **Prepare the input spreadsheet format**
   - The source tab should contain these headers:
     - `first name`
     - `last name`
     - `company`
     - `corporate website`
     - `linkedin industry`
     - `linkedin url`
     - `job title`
     - `location`

5. **Ensure destination tabs exist in the same spreadsheet**
   - Create a tab named `Qualified`
   - Create a tab named `Unqualified`
   - Add these headers to both tabs:
     - `fullName`
     - `companyName`
     - `companyWebsite`
     - `linkedinUrl`
     - `jobTitle`
     - `location`
     - `email`
     - `source`
     - `confidence`

6. **Add a Set node to normalize lead fields**
   - Node type: `Set`
   - Name: `Normalize Lead Data`
   - Connect `Read Lead Sheet -> Normalize Lead Data`
   - Create these fields:
     - `fullName` = `{{ ($json['first name'] ?? '') + ' ' + ($json['last name'] ?? '') }}`
     - `companyName` = `{{ $json.company }}`
     - `companyWebsite` = `{{ $json['corporate website'] }}`
     - `linkedinIndustry` = `{{ $json['linkedin industry'] }}`
     - `linkedinUrl` = `{{ $json['linkedin url'] }}`
     - `jobTitle` = `{{ $json['job title'] }}`
     - `location` = `{{ $json.location }}`

7. **Add a Split In Batches node**
   - Node type: `Split In Batches`
   - Name: `Batch Leads`
   - Connect `Normalize Lead Data -> Batch Leads`
   - Set batch size to `3`

8. **Add the OpenAI node**
   - Node type: `OpenAI` from the LangChain/OpenAI integration
   - Name: `Find Email via OpenAI`
   - Connect the batch-processing output of `Batch Leads -> Find Email via OpenAI`
   - Credentials: add your OpenAI API key
   - Model: `gpt-4o`
   - Enable retry on fail
   - Set wait between retries to `5000 ms`

9. **Configure the OpenAI prompt carefully**
   - The exported JSON does not include a visible prompt, but the workflow depends on structured JSON output.
   - You should configure the node to send each lead’s details and require output in this shape:
     - `email`
     - `source`
     - `confidence`
   - Strongly instruct the model:
     - return only JSON
     - do not include prose
     - use empty string if unknown
   - Recommended payload content to include in the prompt:
     - full name
     - company name
     - company website
     - LinkedIn industry
     - LinkedIn URL
     - job title
     - location

10. **Example logic for the OpenAI instruction**
    - Tell the model to identify the most likely professional email for the person.
    - Ask it to return:
      - `email`: likely work email or empty string
      - `source`: basis or source type used
      - `confidence`: a confidence label or numeric score
    - Require strict JSON.

11. **Add a Wait node**
    - Node type: `Wait`
    - Name: `Rate Limit Delay`
    - Connect `Find Email via OpenAI -> Rate Limit Delay`
    - Set duration to `1.5`
    - Preserve output data after the wait
    - If your UI asks for unit, use the equivalent that matches 1.5 seconds

12. **Add a Set node for parsing results**
    - Node type: `Set`
    - Name: `Parse Email Results`
    - Connect `Rate Limit Delay -> Parse Email Results`
    - Recreate these fields:
      - `fullName` = `{{ $('Normalize Lead Data').item.json.fullName }}`
      - `companyName` = `{{ $('Normalize Lead Data').item.json.companyName }}`
      - `companyWebsite` = `{{ $('Normalize Lead Data').item.json.companyWebsite }}`
      - `linkedinIndustry` = `{{ $('Normalize Lead Data').item.json.linkedinIndustry }}`
      - `linkedinUrl` = `{{ $('Normalize Lead Data').item.json.linkedinUrl }}`
      - `jobTitle` = `{{ $('Normalize Lead Data').item.json.jobTitle }}`
      - `location` = `{{ $('Normalize Lead Data').item.json.location }}`
      - `email` = `{{ JSON.parse($json.text.replace(/```json|```/g, '').trim()).email }}`
      - `source` = `{{ JSON.parse($json.text.replace(/```json|```/g, '').trim()).source }}`
      - `confidence` = `{{ JSON.parse($json.text.replace(/```json|```/g, '').trim()).confidence }}`
    - Enable retry on fail if available

13. **Close the batch loop**
    - Connect `Parse Email Results -> Batch Leads`
   - This allows the workflow to request the next batch/item.

14. **Add an IF node for qualification**
   - Node type: `IF`
   - Name: `Has Valid Email?`
   - Connect the other output of `Batch Leads -> Has Valid Email?`
   - Create a string condition:
     - Left value: `{{ $('Parse Email Results').item.json.email }}`
     - Operation: `contains`
     - Right value: `@`

15. **Add the qualified output Google Sheets node**
   - Node type: `Google Sheets`
   - Name: `Save Qualified Lead`
   - Connect `Has Valid Email?` true branch to this node
   - Operation: `Append`
   - Document ID:
     - `{{ $('Watch For New Leads').item.json.id }}`
   - Sheet name: `Qualified`
   - Map these columns:
     - `fullName` = `{{ $('Parse Email Results').item.json.fullName }}`
     - `companyName` = `{{ $('Parse Email Results').item.json.companyName }}`
     - `companyWebsite` = `{{ $('Parse Email Results').item.json.companyWebsite }}`
     - `linkedinUrl` = `{{ $('Parse Email Results').item.json.linkedinUrl }}`
     - `jobTitle` = `{{ $('Parse Email Results').item.json.jobTitle }}`
     - `location` = `{{ $('Parse Email Results').item.json.location }}`
     - `email` = `{{ $('Parse Email Results').item.json.email }}`
     - `source` = `{{ $('Parse Email Results').item.json.source }}`
     - `confidence` = `{{ $('Parse Email Results').item.json.confidence }}`

16. **Add the unqualified output Google Sheets node**
   - Node type: `Google Sheets`
   - Name: `Save Unqualified Lead`
   - Connect `Has Valid Email?` false branch to this node
   - Operation: `Append`
   - Document ID:
     - `{{ $('Watch For New Leads').item.json.id }}`
   - Sheet name: `Unqualified`
   - Use the same column mappings as the qualified node

17. **Add optional sticky notes for documentation**
   - Add a large note describing the workflow purpose and setup checklist
   - Add a section note around the trigger/read/normalize nodes
   - Add a section note around the batching/OpenAI/parse nodes
   - Add a section note around the IF/save nodes
   - Add a warning note reminding users to configure credentials and folder ID

18. **Credential setup**
   - Google Drive OAuth2:
     - Must have permission to watch the folder
   - Google Sheets OAuth2:
     - Must have permission to read and append to the same spreadsheet
   - OpenAI:
     - Must support the selected model, here `gpt-4o`

19. **Recommended validation before activation**
   - Test with one spreadsheet in the watched folder
   - Confirm the trigger file ID resolves correctly in both read and write nodes
   - Confirm AI output is valid parseable JSON
   - Confirm both destination tabs exist
   - Confirm the source sheet headers exactly match expected names

20. **Recommended hardening improvements**
   - Add a file-type check before reading the sheet
   - Add JSON parse protection with a Code node or error branch
   - Replace the email validation rule with regex validation
   - Trim `fullName`
   - Explicitly define a prompt or structured output schema in the OpenAI node

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not contain sub-workflow trigger nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Qualify Lead Lists & Find Professional Emails: Drop a lead list into Google Drive. This workflow reads every row, uses OpenAI to find professional emails, and splits leads into Qualified and Unqualified tabs automatically. | Workflow purpose |
| Setup checklist: connect Google Drive and Google Sheets OAuth2 credentials, set the watched folder ID, connect OpenAI API key, prepare input columns, and create Qualified/Unqualified tabs with the expected headers. | Operational setup |
| Customization guidance: adjust batch size from 3 and delay from 1.5s depending on API tier; replace GPT-4o with a cheaper model for high-volume processing. | Performance and cost tuning |
| Warning: Connect your Google Drive, Sheets, and OpenAI credentials and update the folder ID in the trigger node before activating this workflow. | Activation prerequisite |

## Additional implementation notes
- The workflow assumes the uploaded file is a Google Sheets document, not merely an Excel file stored in Drive.
- The OpenAI node configuration shown in the export is incomplete for reliable parsing; structured prompting is essential.
- The routing logic uses cross-node expressions rather than only direct-input fields, so item pairing must remain stable.
- The email validation check is intentionally simple and should be strengthened if deliverability matters.