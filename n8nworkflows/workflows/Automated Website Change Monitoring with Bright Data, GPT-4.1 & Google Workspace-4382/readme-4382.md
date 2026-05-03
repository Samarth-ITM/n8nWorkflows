Automated Website Change Monitoring with Bright Data, GPT-4.1 & Google Workspace

https://n8nworkflows.xyz/workflows/automated-website-change-monitoring-with-bright-data--gpt-4-1---google-workspace-4382


# Automated Website Change Monitoring with Bright Data, GPT-4.1 & Google Workspace

### 1. Workflow Overview

This workflow automates weekly monitoring of website changes using Bright Data scraping, GPT-4.1 AI processing, and Google Workspace integration. Its primary use case is to track changes on multiple web pages, extract structured content weekly, compare new data against the previous week, generate detailed change reports, and notify stakeholders via email.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Initialization**: Sets up the workflow variables and reads comparison URLs from a Google Sheet.
- **1.2 Web Scraping and AI Data Extraction**: Uses Bright Data MCP to scrape each URL as Markdown and GPT-4.1 agent to parse and structure the webpage data into JSON.
- **1.3 Current Week Data Storage and Sheet Update**: Converts the AI output to JSON files, uploads them to Google Drive, and updates the comparison spreadsheet with current week file metadata.
- **1.4 Previous Week Data Retrieval and Setup**: Checks for previous week data, downloads and converts it to JSON, prepares both weeks' data for comparison.
- **1.5 Test Mode Mocking (Optional)**: If enabled, mocks previous week changes for testing.
- **1.6 Detecting Changes Between Weeks**: Compares previous and current week JSON data, detects granular changes organized by website section.
- **1.7 Generating and Storing Change Reports**: Converts detected changes into markdown and HTML, creates/updates Google Docs comparison documents, and updates the spreadsheet with comparison file links.
- **1.8 Sending Email Notifications**: Sends an email with the HTML comparison report to the configured recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Initialization

- **Overview**: This block triggers the workflow weekly and initializes key variables by reading from a Google Sheet and merging with hardcoded workflow variables.
- **Nodes Involved**: 
  - Schedule Trigger
  - Set workflow variables
  - Read from comparison spreadsheets
  - Merge workflow variables with Google Sheet data
  - Loop over each comparison URL
- **Node Details**:

1. **Schedule Trigger**  
   - *Type*: Schedule Trigger  
   - *Role*: Initiates workflow execution weekly every Wednesday at 12:00 PM.  
   - *Config*: Interval set for weekly trigger on Wednesdays at noon.  
   - *Connections*: Outputs to "Set workflow variables".  
   - *Edge Cases*: Missed trigger if n8n instance offline; no retries by default.

2. **Set workflow variables**  
   - *Type*: Set  
   - *Role*: Defines workflow-level variables like Google Drive folder ID, spreadsheet ID, sheet name, email recipient, and a test mode flag.  
   - *Config*: Variables initially empty or false; user must configure.  
   - *Connections*: Outputs to "Read from comparison spreadsheets".  
   - *Edge Cases*: Misconfigured or empty IDs cause downstream failures.

3. **Read from comparison spreadsheets**  
   - *Type*: Google Sheets  
   - *Role*: Reads rows (URLs and file metadata) from the configured comparison spreadsheet.  
   - *Config*: Uses "ComparisonSpreadsheetID" and "ComparisonSpreadsheetSheetName" from variables.  
   - *Credentials*: Google Sheets OAuth2 required.  
   - *Connections*: Outputs to "Merge workflow variables with Google Sheet data".  
   - *Failure*: Auth errors, sheet not found, permission denied.

4. **Merge workflow variables with Google Sheet data**  
   - *Type*: Set  
   - *Role*: Merges static workflow variables and dynamic spreadsheet data to unify per-item context for each URL.  
   - *Config*: Assigns spreadsheet columns to workflow variables using expressions.  
   - *Connections*: Outputs to "Loop over each comparison URL".  
   - *Edge Cases*: Data mismatches or missing fields cause incomplete merges.

5. **Loop over each comparison URL**  
   - *Type*: SplitInBatches  
   - *Role*: Iterates over each URL row to process them individually downstream.  
   - *Config*: Default batch size 1; each URL is processed sequentially.  
   - *Connections*: Main output to "Web scraping and data extraction agent".  
   - *Edge Cases*: Empty input halts processing; no URLs means no scraping.

---

#### 2.2 Web Scraping and AI Data Extraction

- **Overview**: For each URL, scrape the full webpage content as Markdown using Bright Data MCP, then extract structured data using a GPT-4.1 agent.
- **Nodes Involved**:
  - Web scraping and data extraction agent (LangChain agent)
  - scrape_as_markdown (Bright Data MCP client tool)
  - GPT-4.1 (LM Chat OpenAI)
  - Structured Output Parser
  - Auto-fixing Output Parser
- **Node Details**:

1. **scrape_as_markdown**  
   - *Type*: MCP Client Tool  
   - *Role*: Uses Bright Data MCP to scrape the webpage content as Markdown, bypassing bot/CAPTCHA protections.  
   - *Config*: Tool name "scrape_as_markdown" with URL as input parameter.  
   - *Credentials*: Bright Data MCP credentials required.  
   - *Connections*: Outputs scraped markdown content to "Web scraping and data extraction agent".  
   - *Failure*: Network timeouts, scraping errors, CAPTCHA failures.

2. **GPT-4.1**  
   - *Type*: LangChain LM Chat OpenAI  
   - *Role*: Processes content using GPT-4.1 model for content understanding (used internally by the agent node).  
   - *Config*: Model set to gpt-4.1.  
   - *Credentials*: OpenAI API key required.  
   - *Connections*: Feeds AI output to output parsers.  
   - *Failure*: API rate limits, connectivity, or prompt errors.

3. **Web scraping and data extraction agent**  
   - *Type*: LangChain agent node  
   - *Role*: Coordinates scraping and GPT-based parsing, instructing the AI to extract structured JSON with fields like filename, metadata, headings, pricing, navigation, CTAs, contact info, banners, FAQ, etc.  
   - *Config*: Uses a detailed system prompt defining extraction rules and expected JSON schema; input is URL.  
   - *Connections*: Outputs parsed JSON to "Convert current week JSON response to file".  
   - *Failure*: Parsing errors, invalid JSON output, missing data fields.

4. **Structured Output Parser** & **Auto-fixing Output Parser**  
   - *Type*: LangChain output parsers  
   - *Role*: Validate and auto-correct the AI output to ensure well-formed JSON matching the schema.  
   - *Config*: The auto-fixing parser retries with instructions on errors.  
   - *Connections*: Chain from Structured Output Parser to Auto-fixing Output Parser, then to agent node input parsing.  
   - *Failure*: Persistent parsing failures may cause workflow errors.

---

#### 2.3 Current Week Data Storage and Sheet Update

- **Overview**: Converts AI JSON output into a file, uploads it to Google Drive, and updates the comparison spreadsheet with this week's file metadata.
- **Nodes Involved**:
  - Convert current week JSON response to file
  - Upload current week JSON file
  - Update comparison sheet with current week file data
  - Check presence of previous week's file
- **Node Details**:

1. **Convert current week JSON response to file**  
   - *Type*: ConvertToFile  
   - *Role*: Converts the JSON output from the agent into a JSON file object suitable for upload.  
   - *Config*: Operation set to convert JSON to a file blob.  
   - *Connections*: Outputs file to "Upload current week JSON file".  
   - *Edge Cases*: Data too large or malformed JSON causes failure.

2. **Upload current week JSON file**  
   - *Type*: Google Drive  
   - *Role*: Uploads the JSON file to the specified Google Drive folder.  
   - *Config*: Uses the filename generated by the agent; folder ID from workflow variables.  
   - *Credentials*: Google Drive OAuth2.  
   - *Connections*: Outputs file metadata to "Update comparison sheet with current week file data".  
   - *Edge Cases*: Permissions errors, quota limits.

3. **Update comparison sheet with current week file data**  
   - *Type*: Google Sheets  
   - *Role*: Updates the row in the comparison spreadsheet for the URL to store current week file ID and link, also retaining previous week metadata.  
   - *Config*: Matches on URL column; updates current and previous week file IDs and links accordingly.  
   - *Credentials*: Google Sheets OAuth2.  
   - *Edge Cases*: Row not found, permission denied.

4. **Check presence of previous week's file**  
   - *Type*: If node  
   - *Role*: Checks if there is a previous week file ID to decide whether to continue with comparison or skip.  
   - *Config*: Condition tests if "Previous Week ID" is not empty.  
   - *Connections*: True branch to download previous week file; False branch loops to next URL.  
   - *Edge Cases*: Missing previous week data means no comparison.

---

#### 2.4 Previous Week Data Retrieval and Setup

- **Overview**: Downloads and converts the previous week's JSON data file from Google Drive, then prepares both previous and current week data for comparison.
- **Nodes Involved**:
  - Download previous week's file
  - Convert previous week's file to JSON
  - Set previous week and current week
- **Node Details**:

1. **Download previous week's file**  
   - *Type*: Google Drive  
   - *Role*: Downloads the previous week's JSON file by ID from Google Drive.  
   - *Config*: File ID from the spreadsheet row's "Previous Week ID".  
   - *Credentials*: Google Drive OAuth2.  
   - *Connections*: Outputs binary file to "Convert previous week's file to JSON".  
   - *Edge Cases*: File missing, permission errors.

2. **Convert previous week's file to JSON**  
   - *Type*: ExtractFromFile  
   - *Role*: Converts the downloaded JSON file binary data into a JSON object for comparison.  
   - *Config*: Operation set to "fromJson".  
   - *Connections*: Outputs JSON to "Set previous week and current week".  
   - *Edge Cases*: Malformed JSON files cause parsing errors.

3. **Set previous week and current week**  
   - *Type*: Set  
   - *Role*: Prepares the context by assigning previous and current week JSON objects into variables for comparison.  
   - *Config*: Uses expressions to extract previous data from downloaded file and current data from AI agent output.  
   - *Connections*: Outputs to "Check if test mode".  
   - *Edge Cases*: Missing fields cause incomplete comparisons.

---

#### 2.5 Test Mode Mocking (Optional)

- **Overview**: If enabled, simulates changes in the previous week data to test change detection and reporting.
- **Nodes Involved**:
  - Check if test mode
  - Mock previous week changes
- **Node Details**:

1. **Check if test mode**  
   - *Type*: If node  
   - *Role*: Checks the boolean "IsTest" flag to decide whether to mock changes or proceed normally.  
   - *Config*: Condition tests if "IsTest" is true.  
   - *Connections*: True branch to "Mock previous week changes"; False branch to "Detect changes between weeks".  
   - *Edge Cases*: Misconfigured flag might cause unwanted mocks.

2. **Mock previous week changes**  
   - *Type*: Code  
   - *Role*: Modifies previous week data by increasing prices, changing features, and appending FAQ answers to simulate differences.  
   - *Config*: JavaScript code modifying nested JSON data.  
   - *Connections*: Outputs modified data to "Detect changes between weeks".  
   - *Edge Cases*: Unexpected data structures cause code errors.

---

#### 2.6 Detecting Changes Between Weeks

- **Overview**: Compares previous and current JSON data deeply to detect additions, removals, and changes, organizing results by website section.
- **Nodes Involved**:
  - Detect changes between weeks
- **Node Details**:

1. **Detect changes between weeks**  
   - *Type*: Code  
   - *Role*: Implements recursive comparison of previous and current JSON objects, detecting changes in primitives, arrays (with special handling for pricing plans and FAQs), and objects. Collects and structures change metadata with display paths and change types.  
   - *Config*: Detailed JavaScript logic for structured diff generation.  
   - *Connections*: Outputs changes object to "Generate Markdown from detected changes".  
   - *Edge Cases*: Large JSON objects may cause performance issues; malformed inputs cause exceptions.

---

#### 2.7 Generating and Storing Change Reports

- **Overview**: Converts detected changes into a human-readable Markdown changelog, converts to HTML, creates a Google Docs document, updates the spreadsheet with comparison document link.
- **Nodes Involved**:
  - Generate Markdown from detected changes
  - Convert Markdown to HTML
  - Create comparison document
  - Update comparison document with results
  - Update comparison spreadsheet with comparison file
- **Node Details**:

1. **Generate Markdown from detected changes**  
   - *Type*: Code  
   - *Role*: Transforms the structured changes object into a Markdown formatted changelog including section headers, bullet points for changes, and a summary with total changes and date.  
   - *Config*: JavaScript markdown generation with formatting helpers.  
   - *Connections*: Outputs markdown to "Convert Markdown to HTML".  
   - *Edge Cases*: No changes detected outputs a no-change message.

2. **Convert Markdown to HTML**  
   - *Type*: Markdown  
   - *Role*: Converts Markdown changelog into HTML for email rendering.  
   - *Config*: Mode set to markdownToHtml.  
   - *Connections*: Outputs HTML to "Create comparison document".  
   - *Edge Cases*: Markdown parsing errors unlikely.

3. **Create comparison document**  
   - *Type*: Google Docs  
   - *Role*: Creates a new Google Docs document named after the URL and date for storing the changelog report.  
   - *Config*: Title dynamically constructed from filename; default folder used.  
   - *Credentials*: Google Docs OAuth2.  
   - *Connections*: Outputs document metadata to "Update comparison document with results".  
   - *Edge Cases*: Permissions or quota issues.

4. **Update comparison document with results**  
   - *Type*: Google Docs  
   - *Role*: Inserts the HTML changelog content into the newly created Google Doc.  
   - *Config*: Uses "insert" action with HTML content.  
   - *Credentials*: Google Docs OAuth2.  
   - *Connections*: Outputs updated document data to "Update comparison spreadsheet with comparison file".  
   - *Edge Cases*: Document locking or API rate limits.

5. **Update comparison spreadsheet with comparison file**  
   - *Type*: Google Sheets  
   - *Role*: Updates the spreadsheet row for the URL with a link to the comparison Google Docs file.  
   - *Config*: Matches on URL column; updates "Comparison File" column with document link.  
   - *Credentials*: Google Sheets OAuth2.  
   - *Edge Cases*: Sheet access errors.

---

#### 2.8 Sending Email Notifications

- **Overview**: Sends an email with the HTML changelog report to a configured recipient.
- **Nodes Involved**:
  - Send email of comparison results
- **Node Details**:

1. **Send email of comparison results**  
   - *Type*: Gmail (OAuth2)  
   - *Role*: Sends an email containing the HTML changelog for each URL comparison weekly.  
   - *Config*: Recipient email from workflow variables; subject includes date and URL; message body is HTML from markdown conversion.  
   - *Credentials*: Gmail OAuth2.  
   - *Connections*: Loops back to "Loop over each comparison URL" to process next.  
   - *Edge Cases*: Email send failures, quota limits, invalid email.

---

### 3. Summary Table

| Node Name                             | Node Type                         | Functional Role                           | Input Node(s)                           | Output Node(s)                                     | Sticky Note                                                                                                                           |
|-------------------------------------|----------------------------------|-----------------------------------------|---------------------------------------|---------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                    | Schedule Trigger                 | Weekly trigger                          | -                                     | Set workflow variables                            | ## Scheduled Weekly                                                                                                                  |
| Set workflow variables             | Set                             | Defines main workflow variables         | Schedule Trigger                      | Read from comparison spreadsheets                  | ## Initialization Initialises workflow variables and merges sheet data                                                               |
| Read from comparison spreadsheets  | Google Sheets                   | Reads URLs and metadata from spreadsheet| Set workflow variables               | Merge workflow variables with Google Sheet data   | ## Initialization                                                                                                                    |
| Merge workflow variables with Google Sheet data | Set                             | Combines workflow and sheet variables   | Read from comparison spreadsheets    | Loop over each comparison URL                      | ## Initialization                                                                                                                    |
| Loop over each comparison URL       | SplitInBatches                  | Iterates URLs for processing             | Merge workflow variables with Google Sheet data | Web scraping and data extraction agent, Loop over... |                                                                                                                                      |
| scrape_as_markdown                  | MCP Client Tool                 | Scrapes webpage as Markdown              | Web scraping and data extraction agent | Web scraping and data extraction agent             | ## AI Scraping For given URL, scrape page content as Markdown                                                                        |
| GPT-4.1                            | LangChain LM Chat OpenAI        | AI language model for parsing            | Web scraping and data extraction agent | Auto-fixing Output Parser                          | ## AI Scraping                                                                                                                       |
| Web scraping and data extraction agent | LangChain agent                | Coordinates scraping and structured extraction | GPT-4.1, scrape_as_markdown, Auto-fixing Output Parser | Convert current week JSON response to file          | ## AI Scraping                                                                                                                       |
| Structured Output Parser           | LangChain output parser         | Validates AI JSON output                 | GPT-4.1                               | Auto-fixing Output Parser                          | ## AI Scraping                                                                                                                       |
| Auto-fixing Output Parser          | LangChain output parser         | Auto-corrects AI JSON output             | Structured Output Parser              | Web scraping and data extraction agent             | ## AI Scraping                                                                                                                       |
| Convert current week JSON response to file | ConvertToFile                 | Converts JSON to file object              | Web scraping and data extraction agent | Upload current week JSON file                      | ## Process current week results                                                                                                     |
| Upload current week JSON file      | Google Drive                   | Uploads JSON file to Drive                | Convert current week JSON response to file | Update comparison sheet with current week file data | ## Process current week results                                                                                                     |
| Update comparison sheet with current week file data | Google Sheets                 | Updates spreadsheet with current week file metadata | Upload current week JSON file         | Check presence of previous week's file             | ## Process current week results                                                                                                     |
| Check presence of previous week's file | If                            | Checks if previous week file exists       | Update comparison sheet with current week file data | Download previous week's file, Loop over each comparison URL | ## Process current week results                                                                                                     |
| Download previous week's file      | Google Drive                   | Downloads previous week JSON file         | Check presence of previous week's file | Convert previous week's file to JSON               | ## Process previous week results                                                                                                   |
| Convert previous week's file to JSON | ExtractFromFile                | Converts binary file to JSON              | Download previous week's file         | Set previous week and current week                  | ## Process previous week results                                                                                                   |
| Set previous week and current week | Set                             | Prepares previous and current data        | Convert previous week's file to JSON | Check if test mode                                  | ## Process previous week results                                                                                                   |
| Check if test mode                 | If                              | Determines test mode path                  | Set previous week and current week   | Mock previous week changes, Detect changes between weeks | ## Mocking If test mode enabled, mock example changes                                                                              |
| Mock previous week changes        | Code                            | Modifies previous data for testing         | Check if test mode                   | Detect changes between weeks                        | ## Mocking                                                                                                                         |
| Detect changes between weeks      | Code                            | Compares previous and current JSON data   | Mock previous week changes, Check if test mode (false branch) | Generate Markdown from detected changes             | ## Current week vs previous week comparison                                                                                        |
| Generate Markdown from detected changes | Code                            | Generates Markdown changelog from changes | Detect changes between weeks          | Convert Markdown to HTML                            | ## Current week vs previous week comparison                                                                                        |
| Convert Markdown to HTML          | Markdown                        | Converts Markdown to HTML                  | Generate Markdown from detected changes | Create comparison document                          | ## Current week vs previous week comparison                                                                                        |
| Create comparison document        | Google Docs                    | Creates Google Docs for comparison report  | Convert Markdown to HTML             | Update comparison document with results             | ## Current week vs previous week comparison                                                                                        |
| Update comparison document with results | Google Docs                    | Inserts HTML changelog into Docs           | Create comparison document            | Update comparison spreadsheet with comparison file  | ## Current week vs previous week comparison                                                                                        |
| Update comparison spreadsheet with comparison file | Google Sheets                 | Updates sheet with comparison Doc link     | Update comparison document with results | Send email of comparison results                    | ## Current week vs previous week comparison                                                                                        |
| Send email of comparison results | Gmail                          | Sends email with comparison report         | Update comparison spreadsheet with comparison file | Loop over each comparison URL                      | ## Send the comparison email                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**
   - Type: Schedule Trigger
   - Configure to trigger weekly on Wednesdays at 12:00 PM.

2. **Add Set node ("Set workflow variables"):**
   - Define variables:
     - DriveFolderID (string) — your Google Drive folder ID for storing files.
     - ComparisonSpreadsheetID (string) — Google Sheets file ID containing URL list.
     - ComparisonSpreadsheetSheetName (string) — usually "Sheet1".
     - Email (string) — recipient email address for notifications.
     - IsTest (boolean) — false for production, true for test mode.

3. **Add Google Sheets node ("Read from comparison spreadsheets"):**
   - Operation: Read rows
   - Document ID: set to ComparisonSpreadsheetID variable.
   - Sheet Name: ComparisonSpreadsheetSheetName variable.
   - Credential: Google Sheets OAuth2.

4. **Add Set node ("Merge workflow variables with Google Sheet data"):**
   - Merge static workflow variables with each spreadsheet row's data using expressions.
   - For example, assign Email = `{{$json.Email}}`, DriveFolderID, etc.

5. **Add SplitInBatches node ("Loop over each comparison URL"):**
   - Batch size: 1.
   - Connect after merging variables.

6. **Add MCP Client Tool node ("scrape_as_markdown"):**
   - Tool name: "scrape_as_markdown"
   - Input parameter: URL from current batch item.
   - Credential: Bright Data MCP client credentials.

7. **Add LangChain GPT-4.1 node ("GPT-4.1"):**
   - Model: gpt-4.1
   - Credential: OpenAI API key for GPT-4.1.

8. **Add LangChain Output Parsers:**
   - Structured Output Parser: define expected JSON schema for webpage content.
   - Auto-fixing Output Parser: configured to retry on parse errors.

9. **Add LangChain Agent node ("Web scraping and data extraction agent"):**
   - Prompt: Detailed system prompt to scrape and extract structured JSON from the Markdown content.
   - Link scrape_as_markdown and GPT-4.1 outputs.
   - Connect output parsers in sequence.

10. **Add ConvertToFile node ("Convert current week JSON response to file"):**
    - Operation: Convert JSON to file blob.

11. **Add Google Drive node ("Upload current week JSON file"):**
    - Upload file named by AI-generated filename to DriveFolderID.
    - Credential: Google Drive OAuth2.

12. **Add Google Sheets node ("Update comparison sheet with current week file data"):**
    - Operation: Update
    - Match rows on URL.
    - Update columns for Current Week File ID and Link, Previous Week File ID and Link.
    - Credential: Google Sheets OAuth2.

13. **Add If node ("Check presence of previous week's file"):**
    - Condition: Previous Week ID is not empty.
    - True: continue to download previous week file.
    - False: skip to next URL batch.

14. **Add Google Drive node ("Download previous week's file"):**
    - Download file by Previous Week ID.
    - Credential: Google Drive OAuth2.

15. **Add ExtractFromFile node ("Convert previous week's file to JSON"):**
    - Operation: fromJson.

16. **Add Set node ("Set previous week and current week"):**
    - Assign previous week JSON from downloaded file.
    - Assign current week JSON from AI output.

17. **Add If node ("Check if test mode"):**
    - Condition: IsTest == true.
    - True: route to mocking changes.
    - False: route to detecting changes.

18. **Add Code node ("Mock previous week changes"):**
    - JavaScript logic to modify previous week data to simulate changes.

19. **Add Code node ("Detect changes between weeks"):**
    - JavaScript recursive comparison of previous and current JSON data.
    - Outputs a structured changes object.

20. **Add Code node ("Generate Markdown from detected changes"):**
    - Converts changes object into a Markdown changelog.

21. **Add Markdown node ("Convert Markdown to HTML"):**
    - Converts Markdown changelog to HTML.

22. **Add Google Docs node ("Create comparison document"):**
    - Creates a new Google Doc titled with URL and date.
    - Credential: Google Docs OAuth2.

23. **Add Google Docs node ("Update comparison document with results"):**
    - Inserts the HTML changelog into the created doc.

24. **Add Google Sheets node ("Update comparison spreadsheet with comparison file"):**
    - Updates spreadsheet row with link to Google Docs comparison document.

25. **Add Gmail node ("Send email of comparison results"):**
    - Sends email to configured recipient.
    - Subject includes date and URL.
    - Body contains the HTML changelog.
    - Credential: Gmail OAuth2.

26. **Connect "Send email of comparison results" back to "Loop over each comparison URL" to process next URL.**

27. **Add Sticky Notes at appropriate places to document setup instructions, variable definitions, and block purposes.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| BrightData Weekly Comparison - This workflow tracks webpage changes weekly, comparing data, generating reports, and sending email notifications. | Overview sticky note in the workflow covers setup steps and prerequisites.                                |
| Prerequisites include setting up Bright Data MCP server and installing the n8n MCP client community node (https://www.npmjs.com/package/n8n-nodes-mcp). | Setup sticky note in workflow.                                                                            |
| Copy the Google Sheet template from [Sheet to Copy](https://docs.google.com/spreadsheets/d/1oPyAaTS8GMqlaBcyCO7G7MRtzMUUaOnA45JfWCzcCa8/edit?usp=sharing). | Setup instructions in sticky note.                                                                        |
| Configure all required OAuth2 credentials for Google Sheets, Drive, Docs, Gmail, OpenAI, and Bright Data MCP before running the workflow.       | Setup instructions in sticky note.                                                                        |
| The workflow is designed to run fully automated on a weekly schedule after initial manual testing and configuration.                            | Workflow overview note.                                                                                   |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.