Generate job descriptions from briefing notes with OpenAI and Google Docs

https://n8nworkflows.xyz/workflows/generate-job-descriptions-from-briefing-notes-with-openai-and-google-docs-14065


# Generate job descriptions from briefing notes with OpenAI and Google Docs

# 1. Workflow Overview

This workflow automatically converts a newly added Google Docs briefing transcript into a polished job description, routes it through a human approval loop in Microsoft Teams, and—once approved—exports the final result as a PDF to Google Drive.

Its main use case is recruitment operations: a Hiring Manager / Recruiter conversation is captured as a Google Doc, then AI extracts structured hiring data, drafts a professional English job description, and supports iterative revision based on reviewer feedback.

## 1.1 Input Reception and File Organization
The workflow starts when a new Google Docs file is created in a specific Google Drive folder. It immediately creates a timestamped subfolder and moves the source briefing document into it so that all downstream artifacts can be stored together.

## 1.2 Transcript Reading and AI-Based Data Extraction
The moved Google Doc is read, and its textual content is sent to an AI Agent backed by an OpenAI chat model. The agent extracts structured job-related information from the German transcript and returns it in JSON format.

## 1.3 Parsing and Logging Structured Job Data
The raw AI output is parsed into actual JSON using a Code node, then appended to a Google Sheets tracker. This creates a structured record of the extracted hiring data.

## 1.4 Job Description Prompt Assembly and Draft Generation
A second Code node assembles a prompt for a JD-writing AI agent. It supports both first-pass generation and revision mode, depending on whether reviewer feedback exists. The JD-Writer agent then produces HTML output for the job description.

## 1.5 Human Approval Loop in Microsoft Teams
The generated HTML is sent to Microsoft Teams using a send-and-wait interaction with a custom form. The reviewer can approve or reject the draft and optionally provide feedback. Rejections loop back into the prompt-building stage to generate a revised version.

## 1.6 PDF Export and Final Archiving
Once approved, the HTML is converted to PDF via a community HTML-to-PDF node, then uploaded to Google Drive into the folder created for that briefing document.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Trigger and File Organization

### Overview
This block detects newly created Google Docs files in a watched folder, creates a dedicated timestamped subfolder, and moves the briefing document into it. It ensures organized storage for the source document and final deliverables.

### Nodes Involved
- Google Drive Trigger
- Create timestamped subfolder
- Move briefing doc to subfolder

### Node Details

#### Google Drive Trigger
- **Type and technical role:** `n8n-nodes-base.googleDriveTrigger`  
  Polling trigger that watches a Google Drive folder for new files.
- **Configuration choices:**
  - Event: `fileCreated`
  - Trigger only on a specific folder
  - Restricts file type to Google Docs: `application/vnd.google-apps.document`
  - Polling frequency: every minute
- **Key expressions or variables used:**
  - Uses a configured folder ID: `YOUR_GOOGLE_DRIVE_FOLDER_ID`
- **Input and output connections:**
  - Entry point of the workflow
  - Outputs to: `Create timestamped subfolder`
- **Version-specific requirements:**
  - Type version: 1
  - Requires valid Google Drive OAuth credentials
- **Edge cases or potential failure types:**
  - Invalid or inaccessible folder ID
  - OAuth token expiration or missing permissions
  - Polling delays or duplicate trigger behavior if files are recreated/copied
  - Non-Google-Docs files are ignored due to MIME filter
- **Sub-workflow reference:** None

#### Create timestamped subfolder
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Creates a folder in Google Drive to hold all assets related to the detected briefing.
- **Configuration choices:**
  - Resource: `folder`
  - Parent folder: same watched folder
  - Folder name pattern: original file name + creation timestamp
- **Key expressions or variables used:**
  - `={{ $json.name }}_{{ $json.createdTime }}`
- **Input and output connections:**
  - Input from: `Google Drive Trigger`
  - Output to: `Move briefing doc to subfolder`
- **Version-specific requirements:**
  - Type version: 3
- **Edge cases or potential failure types:**
  - Invalid folder naming if `createdTime` contains characters problematic for downstream use
  - Insufficient permissions to create folders
  - Parent folder not found
- **Sub-workflow reference:** None

#### Move briefing doc to subfolder
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Moves the just-triggered Google Doc into the newly created subfolder.
- **Configuration choices:**
  - Operation: `move`
  - File ID pulled from the trigger node
  - Destination folder ID pulled from the previous folder-creation step
- **Key expressions or variables used:**
  - File ID: `={{ $('Google Drive Trigger').item.json.id }}`
  - Folder ID: `={{ $json.id }}`
- **Input and output connections:**
  - Input from: `Create timestamped subfolder`
  - Output to: `Read Google Doc content`
- **Version-specific requirements:**
  - Type version: 3
- **Edge cases or potential failure types:**
  - File not found if it was deleted or moved externally before this step
  - Insufficient Drive permissions
  - Expression resolution failure if prior node output is missing
- **Sub-workflow reference:** None

---

## 2.2 Block: Transcript Reading and Structured AI Extraction

### Overview
This block reads the contents of the Google Doc and sends the transcript to an AI extraction agent. The agent is instructed to extract only information present in the transcript and return a strict JSON structure in professional business English.

### Nodes Involved
- Read Google Doc content
- Extract job data from transcript
- OpenAI Chat Model

### Node Details

#### Read Google Doc content
- **Type and technical role:** `n8n-nodes-base.googleDocs`  
  Fetches the content of the Google Doc after it has been moved.
- **Configuration choices:**
  - Operation: `get`
  - Document reference taken from the incoming item’s `id`
- **Key expressions or variables used:**
  - `documentURL: ={{ $json.id }}`
- **Input and output connections:**
  - Input from: `Move briefing doc to subfolder`
  - Output to: `Extract job data from transcript`
- **Version-specific requirements:**
  - Type version: 2
  - Requires Google Docs access via Google credentials
- **Edge cases or potential failure types:**
  - Misleading parameter name: `documentURL` is supplied with an ID expression; this must still be accepted by the node/runtime
  - Document inaccessible after move
  - Empty or malformed document content
- **Sub-workflow reference:** None

#### Extract job data from transcript
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent that performs information extraction from a German transcript into a structured English JSON object.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Strong extraction instructions:
    - only use transcript facts
    - translate into professional Business English
    - no invention of missing values
    - use `null` or `[]` when data is absent
  - Expects structured JSON-only output
  - Output parser enabled
- **Key expressions or variables used:**
  - Transcript injection: `{{ $json.content }}`
- **Input and output connections:**
  - Main input from: `Read Google Doc content`
  - AI language model input from: `OpenAI Chat Model`
  - Main output to: `Parse AI JSON output`
- **Version-specific requirements:**
  - Type version: 3.1
  - Depends on LangChain-compatible AI nodes in n8n
- **Edge cases or potential failure types:**
  - Hallucinated values despite prompt constraints
  - Invalid JSON returned by the model
  - Transcript content too long for the model context window
  - Empty transcript causing underfilled output
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the OpenAI model used by the extraction agent.
- **Configuration choices:**
  - Model: `gpt-5.1`
  - Output text format constrained via JSON Schema
  - Extensive schema enforces nested objects, required fields, allowed nullability, and prohibits additional properties
- **Key expressions or variables used:**
  - None in expressions; schema is hard-coded in node configuration
- **Input and output connections:**
  - Connected as AI language model to: `Extract job data from transcript`
- **Version-specific requirements:**
  - Type version: 1.3
  - Requires OpenAI credentials
  - JSON-schema response formatting support must be available in the installed node/model combination
- **Edge cases or potential failure types:**
  - Unsupported model name in a given OpenAI account/region
  - Schema enforcement incompatibility
  - Rate limits, token limits, invalid API key, or billing issues
- **Sub-workflow reference:** None

---

## 2.3 Block: Parsing and Logging Structured Job Data

### Overview
This block converts the AI agent’s output string into usable JSON and appends the structured record to Google Sheets. It acts as the persistence layer for extracted recruitment data.

### Nodes Involved
- Parse AI JSON output
- Log job data in Google Sheets

### Node Details

#### Parse AI JSON output
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the `output` field from the extraction agent into native JSON items.
- **Configuration choices:**
  - JavaScript code loops through all input items
  - Attempts `JSON.parse(output.output)`
  - Logs failed parses and filters out invalid items
- **Key expressions or variables used:**
  - Reads `item.json.output`
- **Input and output connections:**
  - Input from: `Extract job data from transcript`
  - Output to: `Log job data in Google Sheets`
- **Version-specific requirements:**
  - Type version: 2
- **Edge cases or potential failure types:**
  - If the agent already emits structured JSON instead of a string, the parser logic may fail
  - Invalid JSON causes item loss due to filtering
  - Silent partial failures because malformed outputs are only logged, not raised as errors
- **Sub-workflow reference:** None

#### Log job data in Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends extracted job data to a spreadsheet row.
- **Configuration choices:**
  - Operation: `append`
  - Mapping mode: define below
  - Explicit column mapping for all major schema fields
  - No automatic type conversion
  - Writes to `Sheet1` in a specified spreadsheet
- **Key expressions or variables used:**
  - Examples:
    - `={{ $json.meta_data.job_title }}`
    - `={{ $json.requirements_profile.languages }}`
    - `={{ $json.contact_info }}`
    - `={{ $json.location_details.is_remote_friendly }}`
- **Input and output connections:**
  - Input from: `Parse AI JSON output`
  - Output to: `Build JD-Writer prompt`
- **Version-specific requirements:**
  - Type version: 4.7
  - Requires Google Sheets credentials
- **Edge cases or potential failure types:**
  - Spreadsheet column mismatch with configured schema
  - Arrays/objects may be written as stringified or implementation-dependent values
  - `convertFieldsToString` is disabled, so complex values may not serialize as expected
  - Missing sheet or permission issues
- **Sub-workflow reference:** None

---

## 2.4 Block: Prompt Assembly and Job Description Generation

### Overview
This block creates the prompt for the job-description-writing agent. It supports two modes: initial creation and revision after reviewer rejection. It also attempts to recover the previous HTML draft so the AI can make minimal edits instead of rewriting from scratch.

### Nodes Involved
- Build JD-Writer prompt
- JD-Writer
- OpenAI Chat Model (JD-Writer)

### Node Details

#### Build JD-Writer prompt
- **Type and technical role:** `n8n-nodes-base.code`  
  Central orchestration logic that builds the prompt and determines whether to create a new draft or revise an existing one.
- **Configuration choices:**
  - Reads current structured job data from input
  - Looks up prior node outputs using `$items(...)`
  - Determines revision mode based on output branch of `Check if JD is approved`
  - Extracts prior HTML from the `JD-Writer` output using direct path checks and deep recursive scan
  - Builds:
    - `system` message
    - `user` message
    - `mode`
    - debugging metadata
  - Uses a large template for create mode
  - Uses a constrained revise prompt for feedback-based edits
- **Key expressions or variables used:**
  - Internal constants:
    - `AGENT_NODE_NAME = 'JD-Writer'`
    - `IF_NODE_NAME = 'Check if JD is approved'`
    - `TEAMS_NODE_NAME = 'Send JD to Teams for approval'`
  - Reads incoming structured fields and `data.feedback`
  - Uses helper functions:
    - `getByPath`
    - `getLastItemFromNode`
    - `findHtmlString`
- **Input and output connections:**
  - Input from:
    - `Log job data in Google Sheets` on first pass
    - `Check if JD is approved` false branch on revision pass
  - Output to: `JD-Writer`
- **Version-specific requirements:**
  - Type version: 2
  - Depends on n8n Code node support for `$items()` and cross-node item inspection
- **Edge cases or potential failure types:**
  - Node-name dependency is strict; renaming referenced nodes breaks logic
  - Branch detection may behave unexpectedly if execution history contains multiple items
  - Previous HTML detection may fail if output structure changes
  - If revision is requested but previous HTML is not found, logic falls back to create mode
  - Assumes Teams feedback field naming stays stable
- **Sub-workflow reference:** None

#### JD-Writer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent that creates or revises a full HTML job description.
- **Configuration choices:**
  - Prompt text comes from `={{ $json.user }}`
  - System message comes from `={{ $json.system }}`
  - Output parser enabled
- **Key expressions or variables used:**
  - User prompt: `={{ $json.user }}`
  - System prompt: `={{ $json.system }}`
- **Input and output connections:**
  - Main input from: `Build JD-Writer prompt`
  - AI language model input from: `OpenAI Chat Model (JD-Writer)`
  - Main output to: `Send JD to Teams for approval`
- **Version-specific requirements:**
  - Type version: 3.1
- **Edge cases or potential failure types:**
  - Model may output non-HTML despite instructions
  - Large prompts may exceed token limits
  - Output parser behavior depends on model consistency
- **Sub-workflow reference:** None

#### OpenAI Chat Model (JD-Writer)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used by the JD-writing agent.
- **Configuration choices:**
  - Model: `gpt-5.1`
  - No extra formatting schema enforced here
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as AI language model to: `JD-Writer`
- **Version-specific requirements:**
  - Type version: 1.3
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - Same OpenAI issues as above: auth, rate limits, billing, model availability
  - Since no HTML schema is enforced, output structure depends entirely on prompting quality
- **Sub-workflow reference:** None

---

## 2.5 Block: Human Approval and Revision Loop

### Overview
This block sends the generated job description to Microsoft Teams, waits for reviewer input, and either approves it for export or loops feedback back into the prompt builder for revision.

### Nodes Involved
- Send JD to Teams for approval
- Check if JD is approved

### Node Details

#### Send JD to Teams for approval
- **Type and technical role:** `n8n-nodes-base.microsoftTeams`  
  Sends the generated HTML into a Teams chat and pauses execution until a reviewer submits a form response.
- **Configuration choices:**
  - Resource: `chatMessage`
  - Operation: `sendAndWait`
  - Sends to a configured chat ID
  - Message body is the generated HTML
  - Custom form fields:
    - Dropdown `Approved?` with `yes`/`no`
    - Optional text field `feedback`
- **Key expressions or variables used:**
  - Message: `={{ $json.output }}`
- **Input and output connections:**
  - Input from: `JD-Writer`
  - Output to: `Check if JD is approved`
- **Version-specific requirements:**
  - Type version: 2
  - Requires Microsoft Teams credentials and webhook/send-and-wait support
- **Edge cases or potential failure types:**
  - Invalid Teams chat ID
  - OAuth permission issues
  - HTML may render poorly or be stripped in Teams
  - Workflow can remain waiting indefinitely until user responds
  - Response payload shape must remain compatible with downstream condition logic
- **Sub-workflow reference:** None

#### Check if JD is approved
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates the approval form response and routes execution accordingly.
- **Configuration choices:**
  - Checks whether `data['Approved?']` equals `yes`
  - True branch goes to PDF conversion
  - False branch goes back to prompt building
- **Key expressions or variables used:**
  - `={{ $json.data['Approved?'] }}`
- **Input and output connections:**
  - Input from: `Send JD to Teams for approval`
  - True output to: `Convert approved JD to PDF`
  - False output to: `Build JD-Writer prompt`
- **Version-specific requirements:**
  - Type version: 2.3
- **Edge cases or potential failure types:**
  - Case sensitivity: only exact `yes` matches
  - Missing response field breaks routing logic
  - Any value other than exact `yes` goes to revision path
- **Sub-workflow reference:** None

---

## 2.6 Block: PDF Export and Final Upload

### Overview
After approval, the final HTML job description is converted to a PDF file and uploaded to Google Drive. The target is intended to be the same subfolder created at the beginning.

### Nodes Involved
- Convert approved JD to PDF
- Upload PDF to Google Drive

### Node Details

#### Convert approved JD to PDF
- **Type and technical role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf`  
  Community node that converts HTML content into a PDF file.
- **Configuration choices:**
  - HTML source comes from the JD-Writer node output
  - Output format: file
- **Key expressions or variables used:**
  - `={{ $('JD-Writer').item.json.output }}`
- **Input and output connections:**
  - Input from: `Check if JD is approved` true branch
  - Output to: `Upload PDF to Google Drive`
- **Version-specific requirements:**
  - Type version: 1
  - Requires community package `n8n-nodes-htmlcsstopdf`
  - Marked in notes as self-hosted only
- **Edge cases or potential failure types:**
  - Node unavailable on n8n Cloud if community nodes are not supported
  - Invalid HTML causes broken rendering
  - CSS support may be limited depending on implementation
  - External asset references may fail during rendering
- **Sub-workflow reference:** None

#### Upload PDF to Google Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the generated PDF file to Google Drive.
- **Configuration choices:**
  - Uploads binary file output from previous node
  - Intended folder is the created subfolder
  - File name is set from job title
- **Key expressions or variables used:**
  - Name: `={{ $('Append row in sheet').item.json.job_title }}`
  - Folder ID: `={{ $('Create folder').item.json.id }}`
- **Input and output connections:**
  - Input from: `Convert approved JD to PDF`
  - No downstream nodes
- **Version-specific requirements:**
  - Type version: 3
  - Requires Google Drive credentials
- **Edge cases or potential failure types:**
  - **Important configuration issue:** the expressions reference nodes named `Append row in sheet` and `Create folder`, but those names do not exist in this workflow JSON. The actual nodes appear to be `Log job data in Google Sheets` and `Create timestamped subfolder`. As configured, this node will fail unless corrected.
  - Missing binary property if PDF generation fails
  - Invalid folder ID or insufficient permission
- **Sub-workflow reference:** None

---

## 2.7 Documentation Nodes

These are non-executable annotation nodes used to describe the workflow visually.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Large introductory note describing the workflow, setup steps, and community node requirement.
- **Connections:** None
- **Edge cases:** None functionally; informational only.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents Step 1: Trigger & File Organization.
- **Connections:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents Step 2: AI Data Extraction.
- **Connections:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents Step 3: JD Generation & Approval Loop.
- **Connections:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents Step 4: PDF Export & Upload.
- **Connections:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Drive Trigger | n8n-nodes-base.googleDriveTrigger | Watches a Google Drive folder for newly created Google Docs |  | Create timestamped subfolder | ### Step 1: Trigger & File Organization<br>Google Drive Trigger watches a folder for new Google Docs. When detected, a timestamped subfolder is created and the document is moved into it. |
| Create timestamped subfolder | n8n-nodes-base.googleDrive | Creates a dedicated subfolder for the incoming briefing file | Google Drive Trigger | Move briefing doc to subfolder | ### Step 1: Trigger & File Organization<br>Google Drive Trigger watches a folder for new Google Docs. When detected, a timestamped subfolder is created and the document is moved into it. |
| Move briefing doc to subfolder | n8n-nodes-base.googleDrive | Moves the newly detected Google Doc into the created subfolder | Create timestamped subfolder | Read Google Doc content | ### Step 1: Trigger & File Organization<br>Google Drive Trigger watches a folder for new Google Docs. When detected, a timestamped subfolder is created and the document is moved into it. |
| Read Google Doc content | n8n-nodes-base.googleDocs | Reads the content of the Google Doc transcript | Move briefing doc to subfolder | Extract job data from transcript | ### Step 2: AI Data Extraction<br>The Google Doc content is read and passed to an OpenAI AI Agent that extracts structured job data (title, department, responsibilities, skills, benefits, etc.) from the German transcript into a JSON schema. |
| Extract job data from transcript | @n8n/n8n-nodes-langchain.agent | Extracts structured hiring data from the transcript using AI | Read Google Doc content; OpenAI Chat Model | Parse AI JSON output | ### Step 2: AI Data Extraction<br>The Google Doc content is read and passed to an OpenAI AI Agent that extracts structured job data (title, department, responsibilities, skills, benefits, etc.) from the German transcript into a JSON schema. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies the extraction LLM with JSON schema enforcement |  | Extract job data from transcript | ### Step 2: AI Data Extraction<br>The Google Doc content is read and passed to an OpenAI AI Agent that extracts structured job data (title, department, responsibilities, skills, benefits, etc.) from the German transcript into a JSON schema. |
| Parse AI JSON output | n8n-nodes-base.code | Parses the extraction agent’s JSON string output into objects | Extract job data from transcript | Log job data in Google Sheets |  |
| Log job data in Google Sheets | n8n-nodes-base.googleSheets | Appends extracted job data to a tracking spreadsheet | Parse AI JSON output | Build JD-Writer prompt |  |
| Build JD-Writer prompt | n8n-nodes-base.code | Builds create/revise prompts and reconstructs prior HTML when needed | Log job data in Google Sheets; Check if JD is approved (false branch) | JD-Writer | ### Step 3: JD Generation & Approval Loop<br>Extracted data is logged in Google Sheets, then an Input Builder prepares a prompt for the JD-Writer AI Agent. The generated HTML job description is sent to Microsoft Teams for review. If rejected, feedback is looped back for revision. |
| JD-Writer | @n8n/n8n-nodes-langchain.agent | Generates or revises the HTML job description | Build JD-Writer prompt; OpenAI Chat Model (JD-Writer) | Send JD to Teams for approval | ### Step 3: JD Generation & Approval Loop<br>Extracted data is logged in Google Sheets, then an Input Builder prepares a prompt for the JD-Writer AI Agent. The generated HTML job description is sent to Microsoft Teams for review. If rejected, feedback is looped back for revision. |
| OpenAI Chat Model (JD-Writer) | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies the LLM for job description generation |  | JD-Writer | ### Step 3: JD Generation & Approval Loop<br>Extracted data is logged in Google Sheets, then an Input Builder prepares a prompt for the JD-Writer AI Agent. The generated HTML job description is sent to Microsoft Teams for review. If rejected, feedback is looped back for revision. |
| Send JD to Teams for approval | n8n-nodes-base.microsoftTeams | Sends the HTML JD to Teams and waits for approval/revision feedback | JD-Writer | Check if JD is approved | ### Step 3: JD Generation & Approval Loop<br>Extracted data is logged in Google Sheets, then an Input Builder prepares a prompt for the JD-Writer AI Agent. The generated HTML job description is sent to Microsoft Teams for review. If rejected, feedback is looped back for revision. |
| Check if JD is approved | n8n-nodes-base.if | Routes approved drafts to export and rejected drafts back to revision | Send JD to Teams for approval | Convert approved JD to PDF; Build JD-Writer prompt | ### Step 3: JD Generation & Approval Loop<br>Extracted data is logged in Google Sheets, then an Input Builder prepares a prompt for the JD-Writer AI Agent. The generated HTML job description is sent to Microsoft Teams for review. If rejected, feedback is looped back for revision. |
| Convert approved JD to PDF | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Converts approved HTML into a PDF file | Check if JD is approved (true branch) | Upload PDF to Google Drive | ### Step 4: PDF Export & Upload<br>Once approved, the HTML job description is converted to PDF via HTML-to-PDF API and uploaded to the Google Drive subfolder alongside the original briefing document. |
| Upload PDF to Google Drive | n8n-nodes-base.googleDrive | Uploads the approved PDF to Google Drive | Convert approved JD to PDF |  | ### Step 4: PDF Export & Upload<br>Once approved, the HTML job description is converted to PDF via HTML-to-PDF API and uploaded to the Google Drive subfolder alongside the original briefing document. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for the full workflow and setup prerequisites |  |  | ## Auto-generate job descriptions from briefing notes<br><br>Turn a **Google Docs role briefing transcript** (e.g. from a Hiring Manager / Recruiter conversation) into a polished, structured **job description** in HTML and PDF -- fully automated with AI and a human-in-the-loop approval via Microsoft Teams.<br><br>## How it works<br><br>1. A new Google Doc appears in a watched Drive folder (the briefing transcript).<br>2. The doc is moved into a timestamped subfolder for organization.<br>3. An AI Agent (OpenAI) reads the transcript and extracts structured job data (title, responsibilities, skills, benefits, etc.) into JSON.<br>4. The extracted data is logged in a Google Sheets tracker.<br>5. A second AI Agent (JD-Writer) generates a full HTML job description from the structured data.<br>6. The draft is sent to **Microsoft Teams** for review with an approve/reject form.<br>7. If approved: the HTML is converted to PDF and uploaded to Google Drive.<br>8. If rejected: feedback is looped back and the JD is revised automatically.<br><br>## Setup steps<br><br>1. **Google Drive & Docs** -- Create OAuth2 credentials. Set the watched folder ID in the Google Drive Trigger node.<br>2. **Google Sheets** -- Create a spreadsheet with columns matching the job data schema (job_title, department, responsibilities, etc.). Update the Sheet ID.<br>3. **OpenAI** -- Add your API key. Used for both the data extraction agent and the JD-Writer agent.<br>4. **Microsoft Teams** -- Create OAuth2 credentials. Set the Teams chat ID in the approval node.<br>5. **HTML-to-PDF** -- Install the community node `n8n-nodes-htmlcsstopdf` (self-hosted only). Add the API credential.<br><br>> **Community node required:** `n8n-nodes-htmlcsstopdf` -- this template works on **self-hosted n8n only**. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual annotation for trigger/file organization stage |  |  | ### Step 1: Trigger & File Organization<br>Google Drive Trigger watches a folder for new Google Docs. When detected, a timestamped subfolder is created and the document is moved into it. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual annotation for AI extraction stage |  |  | ### Step 2: AI Data Extraction<br>The Google Doc content is read and passed to an OpenAI AI Agent that extracts structured job data (title, department, responsibilities, skills, benefits, etc.) from the German transcript into a JSON schema. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual annotation for JD generation and approval loop |  |  | ### Step 3: JD Generation & Approval Loop<br>Extracted data is logged in Google Sheets, then an Input Builder prepares a prompt for the JD-Writer AI Agent. The generated HTML job description is sent to Microsoft Teams for review. If rejected, feedback is looped back for revision. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual annotation for PDF export and upload stage |  |  | ### Step 4: PDF Export & Upload<br>Once approved, the HTML job description is converted to PDF via HTML-to-PDF API and uploaded to the Google Drive subfolder alongside the original briefing document. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Auto-generate job descriptions from briefing notes with OpenAI and Google Docs`.

2. **Add a Google Drive Trigger node**
   - Node type: `Google Drive Trigger`
   - Event: `File Created`
   - Trigger mode: specific folder
   - Folder to watch: your Google Drive folder containing briefing docs
   - Polling interval: every minute
   - File type filter: `application/vnd.google-apps.document`
   - Configure Google OAuth2 credentials with Drive access.

3. **Add a Google Drive node to create a subfolder**
   - Node type: `Google Drive`
   - Name it: `Create timestamped subfolder`
   - Resource: `Folder`
   - Parent folder: the same watched folder
   - Folder name expression:
     - `{{ $json.name }}_{{ $json.createdTime }}`
   - Connect: `Google Drive Trigger -> Create timestamped subfolder`

4. **Add a Google Drive node to move the original briefing doc**
   - Node type: `Google Drive`
   - Name it: `Move briefing doc to subfolder`
   - Operation: `Move`
   - File ID:
     - `{{ $('Google Drive Trigger').item.json.id }}`
   - Destination folder ID:
     - `{{ $json.id }}`
   - Connect: `Create timestamped subfolder -> Move briefing doc to subfolder`

5. **Add a Google Docs node to read the document**
   - Node type: `Google Docs`
   - Name it: `Read Google Doc content`
   - Operation: `Get`
   - Document reference:
     - use the incoming document ID from the moved file item
     - configured in the source workflow as `{{ $json.id }}`
   - Connect: `Move briefing doc to subfolder -> Read Google Doc content`

6. **Add an OpenAI Chat Model node for extraction**
   - Node type: `OpenAI Chat Model`
   - Name it: `OpenAI Chat Model`
   - Model: `gpt-5.1`
   - Enable structured response formatting with a JSON Schema
   - Use a schema matching the extracted structure:
     - `meta_data`
     - `location_details`
     - `content_core`
     - `requirements_profile`
     - `benefits_and_culture`
     - `contact_info`
   - Ensure nullability is allowed where the extraction prompt expects missing values.

7. **Add an AI Agent node for extraction**
   - Node type: `AI Agent`
   - Name it: `Extract job data from transcript`
   - Prompt source: define directly
   - Enable output parser
   - Paste an extraction prompt that:
     - states the input is a German transcript
     - instructs the model to extract facts only
     - translate to professional Business English
     - output only one JSON object
     - set missing strings to `null`, arrays to `[]`, unclear booleans to `null`
   - Inject transcript content with:
     - `{{ $json.content }}`
   - Connect:
     - Main: `Read Google Doc content -> Extract job data from transcript`
     - AI model: `OpenAI Chat Model -> Extract job data from transcript`

8. **Add a Code node to parse the extraction output**
   - Node type: `Code`
   - Name it: `Parse AI JSON output`
   - Use JavaScript that:
     - loops through all incoming items
     - parses `item.json.output`
     - filters out invalid JSON
   - Equivalent behavior:
     - `JSON.parse(output.output)` inside try/catch
   - Connect: `Extract job data from transcript -> Parse AI JSON output`

9. **Create a Google Sheets spreadsheet for logging**
   - Add columns for:
     - `job_title`
     - `job_subtitle`
     - `department`
     - `employment_type`
     - `seniority_level`
     - `locations`
     - `is_remote_friendly`
     - `location_policy_text`
     - `mission_statement`
     - `responsibilities`
     - `tech_stack`
     - `hard_skills`
     - `soft_skills`
     - `languages`
     - `benefits_list`
     - `culture_statement`
     - `contact_info`

10. **Add a Google Sheets node to append extracted data**
    - Node type: `Google Sheets`
    - Name it: `Log job data in Google Sheets`
    - Operation: `Append`
    - Select your spreadsheet and target sheet
    - Use manual field mapping
    - Map fields from the parsed JSON, for example:
      - `job_title -> {{ $json.meta_data.job_title }}`
      - `department -> {{ $json.meta_data.department }}`
      - `locations -> {{ $json.location_details.locations }}`
      - `responsibilities -> {{ $json.content_core.responsibilities }}`
      - etc.
    - Connect: `Parse AI JSON output -> Log job data in Google Sheets`

11. **Add a second OpenAI Chat Model node for JD generation**
    - Node type: `OpenAI Chat Model`
    - Name it: `OpenAI Chat Model (JD-Writer)`
    - Model: `gpt-5.1`
    - No schema enforcement is strictly required in this workflow, since the output is HTML.
    - Configure OpenAI credentials.

12. **Add a Code node to build the JD prompt**
    - Node type: `Code`
    - Name it: `Build JD-Writer prompt`
    - Paste logic that:
      - reads structured fields from current input
      - checks whether the execution came from the approval-rejection branch
      - reads Teams feedback if present
      - retrieves previous JD HTML from `JD-Writer`
      - chooses between `create` and `revise` mode
      - outputs:
        - `mode`
        - `system`
        - `user`
        - `feedback`
        - `previous_html`
        - flattened job fields
    - Important:
      - Keep downstream-referenced node names exactly consistent if using `$items('Node Name')`
      - The original script depends on:
        - `JD-Writer`
        - `Check if JD is approved`
        - `Send JD to Teams for approval`

13. **Add the JD-writing AI Agent**
    - Node type: `AI Agent`
    - Name it: `JD-Writer`
    - Prompt text:
      - `{{ $json.user }}`
    - System message:
      - `{{ $json.system }}`
    - Enable output parser
    - Connect:
      - Main: `Build JD-Writer prompt -> JD-Writer`
      - AI model: `OpenAI Chat Model (JD-Writer) -> JD-Writer`

14. **Add a Microsoft Teams node for human approval**
    - Node type: `Microsoft Teams`
    - Name it: `Send JD to Teams for approval`
    - Resource: `Chat Message`
    - Operation: `Send and Wait`
    - Choose the target Teams chat
    - Message body:
      - `{{ $json.output }}`
    - Response type: custom form
    - Add form fields:
      1. Dropdown field labeled `Approved?`
         - options: `yes`, `no`
         - required
         - default: `yes`
      2. Text field labeled `feedback`
         - placeholder: `what should i change or improve?`
    - Connect: `JD-Writer -> Send JD to Teams for approval`
    - Configure Microsoft Teams OAuth2 credentials.

15. **Add an IF node for approval routing**
    - Node type: `IF`
    - Name it: `Check if JD is approved`
    - Condition:
      - left value: `{{ $json.data['Approved?'] }}`
      - operator: equals
      - right value: `yes`
    - Connect: `Send JD to Teams for approval -> Check if JD is approved`

16. **Connect the rejection path back into prompt building**
    - Connect the **false** output of `Check if JD is approved` to `Build JD-Writer prompt`
    - This creates the revision loop.

17. **Install the HTML-to-PDF community node**
    - Package: `n8n-nodes-htmlcsstopdf`
    - This is required for the export stage.
    - The source notes indicate this setup is intended for self-hosted n8n.

18. **Add the HTML-to-PDF node**
    - Node type: `HTML to PDF` from `n8n-nodes-htmlcsstopdf`
    - Name it: `Convert approved JD to PDF`
    - HTML content:
      - `{{ $('JD-Writer').item.json.output }}`
    - Output format: file
    - Connect the **true** output of `Check if JD is approved` to this node.

19. **Add a Google Drive upload node**
    - Node type: `Google Drive`
    - Name it: `Upload PDF to Google Drive`
    - Configure it to upload the binary file from the PDF node
    - Folder ID should reference the subfolder created earlier:
      - use `{{ $('Create timestamped subfolder').item.json.id }}`
    - File name should ideally reference a valid available field, for example:
      - `{{ $('Log job data in Google Sheets').item.json.meta_data?.job_title || $json.job_title || 'job-description' }}`
      - or a simpler stable expression you know exists in scope
    - Connect: `Convert approved JD to PDF -> Upload PDF to Google Drive`

20. **Correct the broken node references from the source workflow**
    - In the provided JSON, the upload node references:
      - `Append row in sheet`
      - `Create folder`
    - These node names do not exist.
    - Replace them with actual node names:
      - `Log job data in Google Sheets`
      - `Create timestamped subfolder`

21. **Optionally add sticky notes**
    - Add visual notes for:
      - Trigger & File Organization
      - AI Data Extraction
      - JD Generation & Approval Loop
      - PDF Export & Upload
      - General setup prerequisites

22. **Test the full workflow**
    - Drop a new Google Doc transcript into the watched folder
    - Confirm:
      - trigger fires
      - subfolder is created
      - source doc is moved
      - doc content is readable
      - extraction returns valid JSON
      - sheet row is appended
      - JD HTML is generated
      - Teams approval form appears
      - rejection loops correctly
      - approval produces PDF
      - PDF uploads to Drive

23. **Validate operational constraints**
    - Ensure the transcript length fits model context limits
    - Ensure Teams reviewers understand they must use exact approval flow
    - Consider serializing arrays/objects before Sheets insertion if spreadsheet formatting matters
    - Consider adding explicit error handling branches for parse failures or missing AI outputs

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Auto-generate job descriptions from briefing notes | Workflow branding/title |
| Turn a Google Docs role briefing transcript into a polished, structured job description in HTML and PDF with AI and a human approval loop in Microsoft Teams | General workflow purpose |
| Google Drive & Docs — Create OAuth2 credentials and set the watched folder ID in the Google Drive Trigger node | Setup prerequisite |
| Google Sheets — Create a spreadsheet with columns matching the job data schema and update the Sheet ID | Setup prerequisite |
| OpenAI — Add your API key for both extraction and JD-writing agents | Setup prerequisite |
| Microsoft Teams — Create OAuth2 credentials and set the Teams chat ID in the approval node | Setup prerequisite |
| HTML-to-PDF — Install the community node `n8n-nodes-htmlcsstopdf` and add the API credential | Setup prerequisite |
| Community node required: `n8n-nodes-htmlcsstopdf` — this template works on self-hosted n8n only | Important deployment constraint |

## Additional implementation note
The workflow contains a real configuration inconsistency in the final Google Drive upload node: it references non-existent nodes (`Append row in sheet` and `Create folder`). This must be fixed before the approval-to-upload path will work successfully.