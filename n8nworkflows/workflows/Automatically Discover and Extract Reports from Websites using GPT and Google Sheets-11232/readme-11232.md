Automatically Discover and Extract Reports from Websites using GPT and Google Sheets

https://n8nworkflows.xyz/workflows/automatically-discover-and-extract-reports-from-websites-using-gpt-and-google-sheets-11232


# Automatically Discover and Extract Reports from Websites using GPT and Google Sheets

### 1. Workflow Overview

The **AI-Powered Report Discovery Agent** workflow automatically discovers and extracts the latest relevant downloadable reports from publication websites by leveraging AI and Google Sheets integration. It is designed for users who maintain a list of report source URLs and want to automate the detection and logging of newly published reports such as PDFs, DOCX, XLSX, and PPTX files.

This workflow is structured into the following logical blocks:

- **1.1 Trigger Sources:** Initiates workflow execution via manual trigger, scheduled daily run, or invocation from another workflow.
- **1.2 Source Reading:** Reads active report sources from a configured Google Sheets document.
- **1.3 Iteration & Fetching:** Loops over the list of sources, fetches each publication webpage’s HTML content, and converts it to Markdown.
- **1.4 AI-Based Report Extraction:** Uses a language model agent to analyze the Markdown content and extract metadata about the most recent downloadable report.
- **1.5 Output Validation:** Validates and normalizes the AI output to ensure correctness and completeness.
- **1.6 Result Handling:** Depending on validation, either saves valid reports to a “Discovered Reports” sheet or logs failures to a “Discovery Log” sheet.
- **1.7 Completion Summary:** Records metadata such as the number of sources processed and workflow completion timestamp.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Sources

**Overview:**  
Starts the workflow execution by any of three possible triggers: manual initiation, scheduled daily run, or external workflow call.

**Nodes Involved:**  
- Manual Trigger  
- Schedule (Daily)  
- Called by Another Workflow

**Node Details:**

- **Manual Trigger**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow for on-demand report discovery.  
  - Configuration: Default, no parameters needed.  
  - Inputs: None  
  - Outputs: Connects to "Read Active Sources" node.  
  - Edge Cases: None specific; manual operation depends on user initiation.

- **Schedule (Daily)**  
  - Type: Schedule Trigger  
  - Role: Runs the workflow automatically once every day.  
  - Configuration: Interval set to daily (default).  
  - Inputs: None  
  - Outputs: Connects to "Read Active Sources" node.  
  - Edge Cases: Network or API downtime may cause missed runs.

- **Called by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be triggered and receive input data from another workflow.  
  - Configuration: Passes input data through without modification.  
  - Inputs: External workflow payload  
  - Outputs: Connects to "Read Active Sources" node.  
  - Edge Cases: Input format from external workflow must match expected schema; otherwise may cause errors.

---

#### 2.2 Source Reading

**Overview:**  
Reads the list of active report sources from a Google Sheets document, specifically from a sheet named "Report Sources".

**Nodes Involved:**  
- Read Active Sources

**Node Details:**

- **Read Active Sources**  
  - Type: Google Sheets (Read operation)  
  - Role: Retrieves the list of report source URLs and metadata from the configured Google Sheets document.  
  - Configuration: Reads from sheet named "Report Sources" in user’s Google Sheets document (document ID parameterized).  
  - Credentials: Google Sheets OAuth2 connected to user account.  
  - Input: Trigger nodes (Manual, Schedule, or External Workflow)  
  - Output: Connects to "Loop Over Sources".  
  - Edge Cases:  
    - Missing or incorrect document ID or sheet name causes failure.  
    - API quota limits or authentication errors may occur.

---

#### 2.3 Iteration & Fetching

**Overview:**  
Processes each source individually by looping over the list, fetching the publication page HTML, and converting it to Markdown for AI processing.

**Nodes Involved:**  
- Loop Over Sources  
- Fetch Publication Page  
- Convert HTML to Markdown  
- Completion Summary

**Node Details:**

- **Loop Over Sources**  
  - Type: SplitInBatches  
  - Role: Iterates over each row (source) from the Google Sheet one at a time.  
  - Configuration: Default batch size 1, no reset between batches.  
  - Input: Output from "Read Active Sources".  
  - Outputs: Two outputs: first to "Completion Summary", second to "Fetch Publication Page".  
  - Edge Cases: Large source lists may slow processing. No reset can cause issues if re-run in same session.

- **Fetch Publication Page**  
  - Type: HTTP Request  
  - Role: Retrieves the HTML content of the source URL for analysis.  
  - Configuration:  
    - URL dynamically set from current source's `Source_URL` field.  
    - Request headers include realistic User-Agent and accept headers to mimic a browser.  
    - Timeout set to 30 seconds.  
    - Error handling set to continue workflow even on failure (onError: continueRegularOutput).  
  - Input: Output from "Loop Over Sources".  
  - Output: Connects to "Convert HTML to Markdown".  
  - Edge Cases:  
    - Timeout or unreachable URLs handled gracefully but result in empty or failed data.  
    - Some sites may block bots or require cookies/authentication, causing empty or error responses.

- **Convert HTML to Markdown**  
  - Type: Markdown node  
  - Role: Converts fetched HTML content to Markdown to facilitate better AI text processing.  
  - Configuration: Input HTML is set from the `data` property of the HTTP response.  
  - Error handling: continue on error, avoids workflow failure if conversion fails.  
  - Input: Output from "Fetch Publication Page".  
  - Output: Connects to "AI Report Discovery Agent".  
  - Edge Cases: HTML content could be malformed or missing, causing conversion issues.

- **Completion Summary**  
  - Type: Set node  
  - Role: Keeps track of the total number of sources processed and records the completion timestamp.  
  - Configuration:  
    - Assigns two variables: `sourcesChecked` = total items processed, `completedAt` = current ISO timestamp.  
  - Input: Main output from "Loop Over Sources".  
  - Output: No further connections (used for logging or status purposes).  
  - Edge Cases: None specific.

---

#### 2.4 AI-Based Report Extraction

**Overview:**  
Analyzes the Markdown content of the publication page using a custom AI language model agent to identify and extract metadata about the latest downloadable report.

**Nodes Involved:**  
- AI Report Discovery Agent  
- OpenAI GPT-5.1 (Language Model)  
- Structured Output Parser

**Node Details:**

- **AI Report Discovery Agent**  
  - Type: LangChain Agent (Custom AI agent)  
  - Role: Uses prompt engineering to instruct the AI to identify the single most recent and relevant downloadable report from the page content.  
  - Configuration:  
    - Input text is the Markdown content from previous node.  
    - System message includes detailed extraction rules, objectives, constraints, and output format requirements (JSON schema).  
    - Enforces returning exactly one report or a "No report found" JSON object.  
  - Input: Markdown text from "Convert HTML to Markdown".  
  - Output: Connects to "Validate & Normalize Output".  
  - Edge Cases:  
    - If AI returns invalid JSON or incomplete data, downstream validation catches it.  
    - Potential API errors or timeouts from language model.  
  - Sub-workflow: Uses OpenAI GPT-5.1 as the underlying language model.

- **OpenAI GPT-5.1**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides the underlying language model (GPT-5.1) for the AI agent.  
  - Configuration:  
    - Model specified as "gpt-5.1".  
    - Temperature set low (0.1) for deterministic output.  
    - Credentials: OpenAI API key connected.  
  - Input: From "AI Report Discovery Agent" node internally.  
  - Output: Passed back to "AI Report Discovery Agent".  
  - Edge Cases: API quota limits, network errors, or invalid API key.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI-generated text output into structured JSON based on example schema.  
  - Configuration: Example JSON schema provided for expected fields: source, title, link, file_type, description.  
  - Input: AI text output from "AI Report Discovery Agent".  
  - Output: Routed internally to "AI Report Discovery Agent" node’s output parser interface.  
  - Edge Cases: Malformed output, parse failures cause fallback or empty data.

---

#### 2.5 Output Validation

**Overview:**  
Validates and normalizes the AI output ensuring that required fields are present, links are valid URLs, and determines the report discovery status.

**Nodes Involved:**  
- Validate & Normalize Output  
- Valid Report Found? (If node)

**Node Details:**

- **Validate & Normalize Output**  
  - Type: Code (JavaScript)  
  - Role: Processes AI output JSON to:  
    - Extract fields with fallbacks  
    - Validate link format (must start with http)  
    - Check if title is meaningful (not "No report found")  
    - Detect if link is a direct download (checking file extensions)  
    - Assign status flags ("Discovered", "No Report Found", "Link May Not Be Direct Download")  
    - Append metadata (source URL, category, timestamp).  
  - Input: Output JSON from "AI Report Discovery Agent".  
  - Output: Connects to "Valid Report Found?" node.  
  - Edge Cases: Malformed AI output or missing fields handled gracefully with defaults.

- **Valid Report Found?**  
  - Type: If node  
  - Role: Branches flow based on whether a valid report was found (`isValid` boolean).  
  - Configuration: Checks if `isValid` equals true.  
  - Input: Validated output from previous code node.  
  - Outputs:  
    - True branch to "Save Discovered Report"  
    - False branch to "Log No Report Found"  
  - Edge Cases: Logic depends on accurate validation results.

---

#### 2.6 Result Handling

**Overview:**  
Logs the discovery results into Google Sheets. Valid reports are saved to "Discovered Reports" sheet, whereas failures or no reports found are logged in "Discovery Log."

**Nodes Involved:**  
- Save Discovered Report  
- Log No Report Found

**Node Details:**

- **Save Discovered Report**  
  - Type: Google Sheets (Append or Update)  
  - Role: Saves validated discovered report metadata into the "Discovered Reports" sheet.  
  - Configuration:  
    - Operation: appendOrUpdate to avoid duplicates.  
    - Sheet and document ID parameterized.  
    - Uses connected Google Sheets OAuth2 credentials.  
  - Input: True branch from "Valid Report Found?"  
  - Output: Loops back to "Loop Over Sources" to continue processing next source.  
  - Edge Cases: Google Sheets API errors, permission issues.

- **Log No Report Found**  
  - Type: Google Sheets (Append or Update)  
  - Role: Logs cases where no valid report was found for a source into the "Discovery Log" sheet.  
  - Configuration: Same as above but different sheet name ("Discovery Log").  
  - Input: False branch from "Valid Report Found?"  
  - Output: Loops back to "Loop Over Sources" for next iteration.  
  - Edge Cases: Same as above.

---

#### 2.7 Additional Nodes (Sticky Notes)

**Overview:**  
Sticky notes provide detailed documentation and guidance for workflow users and developers.

**Nodes Involved:**  
- Sticky Note - Introduction  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

**Node Details:**  
- These nodes contain explanatory text describing the workflow purpose, setup instructions, and block descriptions.  
- They are not connected to the workflow execution but serve an important documentation role.  
- Sticky notes highlight sections like "Read the Source URLs", "Fetch the Publication and Convert to Markdown", "Process the Publication using the LLM", and "Validate the AI Output" etc.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                                       | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                              |
|----------------------------|--------------------------------------|------------------------------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------|
| Manual Trigger             | Manual Trigger                       | Start workflow on demand                              | None                         | Read Active Sources            |                                                                                                         |
| Schedule (Daily)           | Schedule Trigger                    | Start workflow daily                                  | None                         | Read Active Sources            |                                                                                                         |
| Called by Another Workflow | Execute Workflow Trigger            | Start workflow from external workflow                 | None                         | Read Active Sources            |                                                                                                         |
| Read Active Sources        | Google Sheets (Read)                | Read report source URLs and metadata                  | Manual Trigger, Schedule, Called by Another Workflow | Loop Over Sources             | Sticky Note: "## Read the Source URLs"                                                                  |
| Loop Over Sources          | SplitInBatches                     | Iterate over each report source                        | Read Active Sources           | Completion Summary, Fetch Publication Page |                                                                                                         |
| Completion Summary         | Set                               | Record count and completion timestamp                 | Loop Over Sources             | None                          |                                                                                                         |
| Fetch Publication Page     | HTTP Request                      | Download HTML content of source URL                    | Loop Over Sources             | Convert HTML to Markdown       | Sticky Note1: "## Fetch the Publication and Convert to Markdown"                                        |
| Convert HTML to Markdown   | Markdown                         | Convert HTML to Markdown for AI processing            | Fetch Publication Page        | AI Report Discovery Agent      |                                                                                                         |
| AI Report Discovery Agent  | LangChain Agent                  | AI analyzes page content to extract report metadata   | Convert HTML to Markdown      | Validate & Normalize Output    | Sticky Note2: "## Process the Publication using the LLM"                                                |
| OpenAI GPT-5.1            | LangChain LM Chat OpenAI          | Language model backend for AI agent                    | AI Report Discovery Agent     | AI Report Discovery Agent      |                                                                                                         |
| Structured Output Parser   | LangChain Structured Output Parser| Parse AI output into JSON                              | AI Report Discovery Agent     | AI Report Discovery Agent      |                                                                                                         |
| Validate & Normalize Output| Code                             | Validate and normalize AI output                       | AI Report Discovery Agent     | Valid Report Found?            | Sticky Note3: "## Validate the AI Output"                                                               |
| Valid Report Found?        | If                               | Branch depending on whether report is valid           | Validate & Normalize Output   | Save Discovered Report, Log No Report Found |                                                                                                         |
| Save Discovered Report     | Google Sheets (AppendOrUpdate)    | Save valid report data to "Discovered Reports" sheet  | Valid Report Found? (true)    | Loop Over Sources              | Sticky Note4: "## Log the Results in Google Sheets"                                                     |
| Log No Report Found        | Google Sheets (AppendOrUpdate)    | Log no report found cases to "Discovery Log" sheet    | Valid Report Found? (false)   | Loop Over Sources              |                                                                                                         |
| Sticky Note - Introduction | Sticky Note                      | Workflow overview and setup instructions               | None                         | None                          | Full detailed introduction and setup instructions                                                      |
| Sticky Note               | Sticky Note                      | Section documentation: Reading source URLs             | None                         | None                          | "## Read the Source URLs"                                                                                |
| Sticky Note1              | Sticky Note                      | Section documentation: Fetching & Markdown conversion  | None                         | None                          | "## Fetch the Publication and Convert to Markdown"                                                     |
| Sticky Note2              | Sticky Note                      | Section documentation: AI processing                    | None                         | None                          | "## Process the Publication using the LLM"                                                             |
| Sticky Note3              | Sticky Note                      | Section documentation: Validation                        | None                         | None                          | "## Validate the AI Output"                                                                              |
| Sticky Note4              | Sticky Note                      | Section documentation: Logging results                   | None                         | None                          | "## Log the Results in Google Sheets"                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add **Manual Trigger** node with default settings.  
   - Add **Schedule Trigger** node configured to run daily at your preferred time (default interval daily).  
   - Add **Execute Workflow Trigger** node configured for passthrough input.  

2. **Add Google Sheets Read Node:**  
   - Add a **Google Sheets** node named "Read Active Sources".  
   - Operation: Read rows from sheet.  
   - Configure document ID and sheet name as "Report Sources".  
   - Attach Google Sheets OAuth2 credentials.  
   - Connect outputs of the three trigger nodes (Manual, Schedule, Execute Workflow) to this node.

3. **Add Batch Split Node:**  
   - Add a **SplitInBatches** node named "Loop Over Sources".  
   - Default batch size 1, no reset option enabled.  
   - Connect output of "Read Active Sources" to input of this node.

4. **Add Completion Summary Node:**  
   - Add a **Set** node named "Completion Summary".  
   - Create variables: `sourcesChecked` = number of items processed (`$items().length`), `completedAt` = current timestamp (`$now.toISO()`).  
   - Connect first output of "Loop Over Sources" to this node.

5. **Fetch Publication Page Node:**  
   - Add an **HTTP Request** node named "Fetch Publication Page".  
   - Set method to GET.  
   - URL: Set dynamically from current item source URL: `={{ $json.Source_URL }}`.  
   - Add headers: User-Agent (modern browser string), Accept (HTML/XML), Accept-Language (en-US).  
   - Set timeout to 30000ms (30 seconds).  
   - On error: continue workflow.  
   - Connect second output of "Loop Over Sources" to this node.

6. **Convert HTML to Markdown Node:**  
   - Add a **Markdown** node named "Convert HTML to Markdown".  
   - Input HTML: `={{ $json.data }}` from HTTP Request node.  
   - On error: continue workflow.  
   - Connect "Fetch Publication Page" output to this node.

7. **Add AI Report Discovery Agent Node:**  
   - Add a **LangChain Agent** node named "AI Report Discovery Agent".  
   - Text input: `={{ $json.data }}` (Markdown content).  
   - Paste the system message prompt that includes detailed instructions for identifying the latest downloadable report, extraction rules, validation, and output format.  
   - Connect "Convert HTML to Markdown" output to this node.

8. **Add OpenAI GPT-5.1 Node:**  
   - Add a **LangChain LM Chat OpenAI** node named "OpenAI GPT-5.1".  
   - Model: Set to "gpt-5.1".  
   - Temperature: 0.1 (low randomness).  
   - Connect this node as the language model for the "AI Report Discovery Agent" node.  
   - Provide OpenAI API credentials.

9. **Add Structured Output Parser Node:**  
   - Add a **LangChain Structured Output Parser** node named "Structured Output Parser".  
   - Provide example JSON schema to parse AI output (fields: source, title, link, file_type, description).  
   - Connect as output parser to "AI Report Discovery Agent".

10. **Validate & Normalize Output Node:**  
    - Add a **Code** node named "Validate & Normalize Output".  
    - Use JavaScript code to:  
      - Extract fields from AI output with defaults.  
      - Check if `link` starts with "http".  
      - Check if `title` is not "No report found".  
      - Determine `status` ("Discovered" or "No Report Found" or "Link May Not Be Direct Download").  
      - Add metadata fields (sourceUrl, category, discoveredAt timestamp).  
    - Connect output of "AI Report Discovery Agent" to this node.

11. **Add If Node for Validation:**  
    - Add an **If** node named "Valid Report Found?".  
    - Condition: Check if boolean field `isValid` is true.  
    - Connect output of "Validate & Normalize Output" node to this node.

12. **Add Google Sheets Nodes to Log Results:**  
    - Add **Google Sheets (AppendOrUpdate)** node named "Save Discovered Report".  
      - Configure to append or update rows in sheet "Discovered Reports".  
      - Connect true output of "Valid Report Found?" to this node.  
      - Use same Google Sheets credentials.  
    - Add **Google Sheets (AppendOrUpdate)** node named "Log No Report Found".  
      - Configure to append or update rows in sheet "Discovery Log".  
      - Connect false output of "Valid Report Found?" to this node.  
      - Use same Google Sheets credentials.

13. **Loop Back Connections:**  
    - Connect output of both "Save Discovered Report" and "Log No Report Found" back to the input of "Loop Over Sources" node to continue processing remaining sources.

14. **Add Sticky Notes:**  
    - Add descriptive sticky notes at relevant workflow sections to document purpose and instructions for users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                          | Context or Link                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| This workflow uses advanced AI prompt engineering to direct GPT-5.1 to extract structured report metadata from publication web pages. Ensure the OpenAI API key has sufficient quota and permissions.                                                                                                                                  | AI integration notes                                                       |
| Google Sheets document must have three sheets: "Report Sources" (list of URLs), "Discovered Reports" (to save results), and "Discovery Log" (to log failures). Proper OAuth2 credentials with edit access are required.                                                                                                                    | Google Sheets setup                                                        |
| The HTTP Request node uses realistic browser headers to avoid blocks; however, some sites with anti-bot measures may still prevent scraping. Consider adding proxy or authentication if needed.                                                                                                                                          | HTTP Request considerations                                               |
| The AI agent returns exactly one report or a "No report found" JSON. Validation code ensures data integrity before saving.                                                                                                                                                                                                             | AI output validation                                                       |
| Sticky notes in the workflow provide detailed explanations for each block and setup instructions. Users should review these for better understanding and troubleshooting.                                                                                                                                                              | Workflow documentation                                                    |
| For further customization, consider enhancing the AI prompt with domain-specific keywords or additional metadata extraction fields.                                                                                                                                                                                                  | Customization advice                                                      |
| The workflow is designed to be modular and extensible; additional triggers or output destinations can be added as required.                                                                                                                                                                                                             | Workflow extensibility                                                    |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.