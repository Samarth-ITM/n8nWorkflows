Discover Hidden Website API Endpoints Using Regex and AI

https://n8nworkflows.xyz/workflows/discover-hidden-website-api-endpoints-using-regex-and-ai-4627


# Discover Hidden Website API Endpoints Using Regex and AI

### 1. Workflow Overview

This workflow, titled **"Discover Hidden Website API Endpoints Using Regex and AI"**, is designed to automatically identify hidden or undocumented API endpoints embedded within JavaScript files of modern web applications. It is ideal for analyzing Single Page Applications (SPAs), Knockout.js, Next.js/Nuxt.js frameworks, and other sites where API endpoints are present as string literals in bundled or CDN-served JavaScript files.

The workflow is logically divided into several functional blocks:

- **1.1 API Endpoints Extraction with Predefined Regex**: Fetches the target website’s HTML, extracts JS file URLs, filters relevant JS files, downloads their content, and applies a predefined regex to extract potential API endpoints.

- **1.2 Initial AI Analysis of JS Files**: Sends each relevant JS file content to a Large Language Model (LLM) to perform a detailed analysis and description of API endpoints, including methods and parameters.

- **1.3 AI Agent Regex Generation and Validation**: Uses an AI agent to generate custom regex patterns based on the AI analysis, then validates these regex patterns against reference endpoint files through iterative refinement.

- **1.4 Final Regex Execution and Endpoint Aggregation**: Executes the validated regex to extract endpoints from AI analysis results, removes duplicates, and merges both predefined regex and AI-generated endpoints.

- **1.5 Export and Reporting**: Sorts and formats the combined endpoint data and exports the results to an Excel file for comparison and further use.

---

### 2. Block-by-Block Analysis

#### 2.1 API Endpoints Extraction with Predefined Regex

**Overview:**  
This block fetches the website HTML, extracts JS file URLs, filters for relevant JS files, retrieves their content, and applies a predefined regex to extract potential API endpoints.

**Nodes Involved:**  
- Configuration  
- Fetch Website HTML  
- Extract URLs of JS files  
- Split URLs of JS files  
- Keep Relevant JS Files  
- Fetch JS Content  
- Extract API Endpoints  
- Check Endpoints Count  
- Split Endpoints  
- Remove Duplicates  
- Reference for Source Metadata  
- Insufficient Endpoints (NoOp)

**Node Details:**  

- **Configuration**  
  - *Type:* Set  
  - *Role:* Stores user inputs: target website URL and User-Agent string for HTTP requests.  
  - *Key Parameters:*  
    - URL (string): Target site URL (e.g., https://example.com)  
    - User-Agent (string): Custom user-agent header to mimic a bot or browser.  
  - *Input:* Manual trigger  
  - *Output:* Provides URL and User-Agent for HTTP requests.

- **Fetch Website HTML**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the raw HTML content of the target website.  
  - *Configuration:*  
    - URL: Dynamic from Configuration node.  
    - Timeout: 30s, follow redirects, allow unauthorized SSL certificates.  
    - Headers: User-Agent from Configuration.  
    - Never error on response to allow downstream handling.  
  - *Input:* Configuration node  
  - *Output:* HTML content in response body.

- **Extract URLs of JS files**  
  - *Type:* HTML Extract  
  - *Role:* Parses HTML to extract `src` attributes of all `<script>` tags (JS file URLs).  
  - *Configuration:* CSS selector `script[src]`, extract `src` attribute as array.  
  - *Input:* Fetch Website HTML  
  - *Output:* List of JS file URLs under `JS URLs`.

- **Split URLs of JS files**  
  - *Type:* Split Out  
  - *Role:* Converts array of JS URLs into individual items for filtering.  
  - *Input:* Extract URLs of JS files  
  - *Output:* Single JS URL per item.

- **Keep Relevant JS Files**  
  - *Type:* Filter  
  - *Role:* Filters JS URLs based on heuristics to keep those likely containing API endpoints.  
  - *Key Filtering Logic:*  
    - Include URLs that start with `/` or `./` (relative paths).  
    - Include URLs containing domain of target site.  
    - Include URLs containing keywords like `yimg`, `/build/`, `/dist/`, `/frontend/`, `/packages/`, or `bundle`.  
    - Include URLs whose domain contains `cdn`.  
  - *Input:* Split URLs of JS files  
  - *Output:* Filtered JS URLs to fetch.

- **Fetch JS Content**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the content of each filtered JS file.  
  - *URL Logic:*  
    - Converts relative URLs to absolute by prefixing with target site URL.  
    - Converts protocol-relative URLs (`//`) to `https://`.  
  - *Batching:* Processes one JS file at a time to avoid overload.  
  - *Never error on response to allow partial results.  
  - *Input:* Keep Relevant JS Files  
  - *Output:* JS file content in `data` field.

- **Extract API Endpoints**  
  - *Type:* Set  
  - *Role:* Applies predefined regex to extract quoted URL-like strings from JS content.  
  - *Regex:* Matches strings enclosed in quotes that resemble URLs or paths, removes query parameters and trailing slashes, excludes static assets (`.js`, `.css`, `.png`, etc.), and filters short strings.  
  - *Output Fields:*  
    - `Endpoints`: Array of extracted endpoint strings.  
    - `JS`: Boolean flag true (indicates processed JS file).  
    - `Source`: Original JS file URL.  
  - *Input:* Fetch JS Content  
  - *Output:* List of candidate endpoints per JS file.

- **Check Endpoints Count**  
  - *Type:* If  
  - *Role:* Filters JS files with more than 4 extracted endpoints to focus on relevant files.  
  - *Input:* Extract API Endpoints  
  - *Output:*  
    - True branch: Continue processing (Split Endpoints, Reference for Source Metadata).  
    - False branch: NoOp node "Insufficient Endpoints" (discard).

- **Split Endpoints**  
  - *Type:* Split Out  
  - *Role:* Splits array of endpoints into individual items for further processing.  
  - *Input:* Check Endpoints Count (true branch)  
  - *Output:* Individual endpoint items.

- **Remove Duplicates**  
  - *Type:* Remove Duplicates  
  - *Role:* Removes duplicate endpoint entries from split endpoints.  
  - *Input:* Split Endpoints  
  - *Output:* Unique endpoints per JS file.

- **Reference for Source Metadata**  
  - *Type:* NoOp  
  - *Role:* Placeholder node to maintain source metadata across branches.  
  - *Input:* Check Endpoints Count (true branch)  
  - *Output:* Passes through data unchanged.

- **Insufficient Endpoints**  
  - *Type:* NoOp  
  - *Role:* Handles cases where too few endpoints are found; effectively discards these JS files.  
  - *Input:* Check Endpoints Count (false branch)

**Edge Cases & Failures:**  
- HTTP request failures (timeouts, SSL errors) handled gracefully by “never error” flags.  
- Filtering may exclude relevant JS files if URL heuristics are too strict.  
- Regex may miss dynamically built endpoints or heavily obfuscated strings.  
- JS files with too few matches are discarded which might exclude relevant endpoints.

---

#### 2.2 Initial AI Analysis of JS Files

**Overview:**  
This block sends the content of each filtered JS file to an AI model for comprehensive analysis of API endpoints, generating detailed descriptions, HTTP methods, and parameters.

**Nodes Involved:**  
- Reference for Source Metadata  
- AI Endpoints Analysis (HTTP Request to OpenRouter API)  
- Add Source Metadata (Set)  
- Process Each Analyzed File (Split In Batches)  
- Format AI Results (Set)  
- Prepare Endpoints File(s) (Convert To File)  
- Save Endpoints File(s) (Read/Write File)  
- Pass File Name, No Binary (Set)  
- Merge AI Analysis & File Name (Merge)

**Node Details:**  

- **AI Endpoints Analysis**  
  - *Type:* HTTP Request  
  - *Role:* Sends JS content to OpenRouter API LLM (default model: Gemini 2.5 Pro-Preview) for detailed endpoint extraction and description.  
  - *Request Body:* JSON with system prompt instructing the AI to identify API endpoints and summarize methods. User prompt contains JS file content.  
  - *Authentication:* Uses predefined OpenRouter API credentials.  
  - *Batching:* Processes one JS file at a time, with retries on failure and 5-second wait between tries.  
  - *Input:* Reference for Source Metadata  
  - *Output:* AI-generated analysis for each JS file.

- **Add Source Metadata**  
  - *Type:* Set  
  - *Role:* Adds original JS file URL (`Source`) metadata to the AI analysis results.  
  - *Input:* AI Endpoints Analysis  
  - *Output:* AI analysis results enriched with source info.

- **Process Each Analyzed File**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over AI analysis results to process files one by one for saving and formatting.  
  - *Input:* Add Source Metadata  
  - *Output:* Single file analyses in batches.

- **Format AI Results**  
  - *Type:* Set  
  - *Role:* Formats AI output, extracts message content or reasoning, assigns file names based on batch index (e.g., "file00.txt").  
  - *Input:* Process Each Analyzed File (batch index context)  
  - *Output:* Formatted AI analysis with filename and source.

- **Prepare Endpoints File(s)**  
  - *Type:* Convert To File  
  - *Role:* Converts AI analysis text into a binary file format for saving.  
  - *Input:* Format AI Results  
  - *Output:* Binary file data.

- **Save Endpoints File(s)**  
  - *Type:* Read/Write File  
  - *Role:* Saves binary files to the local n8n instance drive.  
  - *Input:* Prepare Endpoints File(s)  
  - *Output:* Confirmation of file saved.

- **Pass File Name, No Binary**  
  - *Type:* Set  
  - *Role:* Strips binary data for further workflow processing, passing only file metadata.  
  - *Input:* Process Each Analyzed File (continuation)  
  - *Output:* Metadata only.

- **Merge AI Analysis & File Name**  
  - *Type:* Merge  
  - *Role:* Combines formatted AI analysis and file name data streams to prepare for next steps.  
  - *Input:* Pass File Name, No Binary and Add Source Metadata  
  - *Output:* Merged enriched AI analysis results.

**Edge Cases & Failures:**  
- LLM API timeouts or errors handled with retries.  
- Potential for incomplete or inaccurate AI analysis depending on model.  
- File saving might fail if instance storage is full or permissions inadequate.

---

#### 2.3 AI Agent Regex Generation and Validation

**Overview:**  
Uses an AI agent to generate regex expressions based on AI analysis reports to extract API endpoints. The generated regexes are validated against reference files iteratively, with up to 4 attempts for optimization.

**Nodes Involved:**  
- Merge Initial AI Analysis & AI Agent Output (Merge)  
- Create Endpoints Regex With AI (Agent Node)  
- Regex Generation LLM (Language Model Node)  
- Auto-fixing Output Parser (Output Parser Node)  
- Parse AI Regex Output (Structured Output Parser)  
- Regex Validation Start (Execute Workflow Trigger)  
- Load Reference Endpoints (Read/Write File)  
- Extract Endpoints File Text (Extract From File)  
- Merge Regex, File Name & Reference Endpoints (Merge)  
- Evaluate LLM Regex (Set)  
- Validate LLM Regex (Tool Workflow Node)  
- Sticky Note3 (Sticky Note describing validation tool)

**Node Details:**  

- **Merge Initial AI Analysis & AI Agent Output**  
  - *Type:* Merge  
  - *Role:* Combines initial AI file analysis with regex generation results from the AI agent.  
  - *Input:* Add Source Metadata and output of Create Endpoints Regex With AI  
  - *Output:* Combined data for regex generation.

- **Create Endpoints Regex With AI**  
  - *Type:* LangChain Agent  
  - *Role:* AI agent receives analysis reports and generates regex expressions as n8n expressions to extract clean API endpoints.  
  - *System Prompt:* Detailed instructions to identify API patterns including `/api/`, versioned paths `/v1/`, dynamic constructions, template literals, and object properties.  
  - *Max Iterations:* 4  
  - *Input:* Merged AI analysis data and file name  
  - *Output:* Regex expressions for endpoint extraction.

- **Regex Generation LLM**  
  - *Type:* LangChain LM Chat (OpenRouter)  
  - *Role:* Language model used by agent to generate regex with temperature 0.2 for controlled creativity.  
  - *Model:* Claude 3.7 Sonnet  
  - *Input:* Passed from Create Endpoints Regex With AI as subnode.  
  - *Output:* Raw AI regex generation output.

- **Auto-fixing Output Parser**  
  - *Type:* LangChain Output Parser Autofixing  
  - *Role:* Attempts to repair and parse regex output from AI to avoid syntax errors in expressions.  
  - *Input:* Regex Generation LLM output  
  - *Output:* Cleaned and structured regex expression.

- **Parse AI Regex Output**  
  - *Type:* LangChain Output Parser Structured  
  - *Role:* Parses AI output into structured JSON containing the regex expression field `n8nexpression`.  
  - *Input:* Auto-fixing Output Parser output  
  - *Output:* Structured regex expression.

- **Regex Validation Start**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Initiates a sub-workflow to validate the generated regex against saved endpoint files.  
  - *Inputs:* `query.n8nexpression` (regex) and `Filename` (reference file).  
  - *Output:* Validation results.

- **Load Reference Endpoints**  
  - *Type:* Read/Write File  
  - *Role:* Loads saved AI analysis file to serve as ground truth for regex validation.  
  - *Input:* Regex Validation Start  
  - *Output:* Raw file content.

- **Extract Endpoints File Text**  
  - *Type:* Extract From File (text)  
  - *Role:* Extracts text content from loaded file for regex application.  
  - *Input:* Load Reference Endpoints  
  - *Output:* File text content.

- **Merge Regex, File Name & Reference Endpoints**  
  - *Type:* Merge  
  - *Role:* Combines regex expression, file name, and reference file content for evaluation.  
  - *Input:* Load Reference Endpoints and Regex Validation Start  
  - *Output:* Combined data.

- **Evaluate LLM Regex**  
  - *Type:* Set  
  - *Role:* Evaluates the generated regex expression against the reference file content to test accuracy.  
  - *Input:* Merge Regex, File Name & Reference Endpoints  
  - *Output:* Evaluation string result.

- **Validate LLM Regex**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Runs the validation sub-workflow and calls the agent for iterative regex refinement based on evaluation results.  
  - *Input:* Regex validation parameters.  
  - *Output:* Validated and potentially improved regex.

- **Sticky Note3**  
  - *Content:* Describes the validation tool as a self-evaluation and iterative improvement mechanism by the AI agent.

**Edge Cases & Failures:**  
- AI agent may fail to generate a regex or produce invalid regex expressions.  
- Validation may not converge within max iterations, resulting in fallback to predefined regex.  
- File loading errors or parsing failures could interrupt validation.  
- Regex evaluation expression errors if syntax is incorrect.

---

#### 2.4 Final Regex Execution and Endpoint Aggregation

**Overview:**  
Executes the validated AI-generated regex on the AI analysis outputs to extract endpoints, merges these with predefined regex results, removes duplicates, and prepares the final endpoint dataset.

**Nodes Involved:**  
- Prepare Data for Regex Extraction (Set)  
- Execute LLM Regex (Set)  
- Split LLM-extracted Endpoints (Split Out)  
- Remove LLM Endpoints Duplicates (Remove Duplicates)  
- Merge Original And LLM Endpoints (Merge)  
- Prepare Source & File Name for Merge (Remove Duplicates)  
- Final Merge with Enriched File Name (Merge)  
- Sort Merged Endpoints (Sort)  
- Reorder Output Fields (Set)  
- Export Comparison Results To Excel (Convert To File)

**Node Details:**  

- **Prepare Data for Regex Extraction**  
  - *Type:* Set  
  - *Role:* Prepares AI analysis data and regex expression output for execution.  
  - *Fields:*  
    - `data` containing AI analysis text.  
    - `output.n8nexpression` containing regex expression string.  
  - *Input:* Merge Initial AI Analysis & AI Agent Output  
  - *Output:* Ready for regex execution.

- **Execute LLM Regex**  
  - *Type:* Set  
  - *Role:* Evaluates the regex expression on AI analysis data to extract endpoints array.  
  - *Flags:* Sets boolean `LLM` to true to mark AI-extracted endpoints.  
  - *Input:* Prepare Data for Regex Extraction  
  - *Output:* Extracted endpoints.

- **Split LLM-extracted Endpoints**  
  - *Type:* Split Out  
  - *Role:* Splits extracted endpoints into individual items for duplicate removal.  
  - *Input:* Execute LLM Regex  
  - *Output:* Single endpoint items.

- **Remove LLM Endpoints Duplicates**  
  - *Type:* Remove Duplicates  
  - *Role:* Removes duplicates from AI-extracted endpoints.  
  - *Input:* Split LLM-extracted Endpoints  
  - *Output:* Unique AI endpoints.

- **Merge Original And LLM Endpoints**  
  - *Type:* Merge  
  - *Role:* Combines predefined regex extracted endpoints with AI-extracted endpoints, preserving all matches.  
  - *Join Mode:* Keep everything, matching on `Endpoints` and `Source`.  
  - *Input:* Remove Duplicates (predefined regex) and Remove LLM Endpoints Duplicates  
  - *Output:* Combined endpoint dataset.

- **Prepare Source & File Name for Merge**  
  - *Type:* Remove Duplicates  
  - *Role:* Cleans and deduplicates by `Source` and `fileName` fields in preparation for final merge.  
  - *Input:* Remove LLM Endpoints Duplicates  
  - *Output:* Unique source and filename entries.

- **Final Merge with Enriched File Name**  
  - *Type:* Merge  
  - *Role:* Enriches combined endpoint data by joining on `Source` to add `fileName` metadata.  
  - *Join Mode:* Enrich Input 1  
  - *Input:* Merge Original And LLM Endpoints and Prepare Source & File Name for Merge  
  - *Output:* Fully enriched endpoint data.

- **Sort Merged Endpoints**  
  - *Type:* Sort  
  - *Role:* Sorts final dataset by `fileName` and `Endpoints` for organized output.  
  - *Input:* Final Merge with Enriched File Name  
  - *Output:* Sorted endpoint list.

- **Reorder Output Fields**  
  - *Type:* Set  
  - *Role:* Sets field order and types for final output (Endpoints, JS flag, Source, fileName, LLM flag).  
  - *Input:* Sort Merged Endpoints  
  - *Output:* Cleaned, ordered output.

- **Export Comparison Results To Excel**  
  - *Type:* Convert To File (XLSX)  
  - *Role:* Exports the final combined and sorted endpoint data to an Excel file `api_comparison.xlsx` with headers.  
  - *Input:* Reorder Output Fields  
  - *Output:* Excel file saved or ready for download.

**Edge Cases & Failures:**  
- Regex evaluation errors if regex is malformed.  
- Duplicate removal relies on exact string matches; minor variants might persist.  
- Merge operations may lose unmatched endpoints if keys mismatch.  
- Excel export may fail if output size is very large or permissions incorrect.

---

#### 2.5 Manual Trigger and Workflow Entry

**Overview:**  
The workflow entry point that starts the entire process manually.

**Nodes Involved:**  
- Start API Discovery (Manual Trigger)

**Node Details:**  

- **Start API Discovery**  
  - *Type:* Manual Trigger  
  - *Role:* Entry trigger node to start the workflow manually.  
  - *Output:* Triggers the Configuration node to begin.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                                      | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                                                     |
|----------------------------------|----------------------------------|----------------------------------------------------|----------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Start API Discovery               | Manual Trigger                   | Entry point to start the workflow manually          | -                                | Configuration                      |                                                                                                                                |
| Configuration                    | Set                              | Stores target URL and User-Agent                     | Start API Discovery              | Fetch Website HTML                 |                                                                                                                                |
| Fetch Website HTML               | HTTP Request                    | Downloads website HTML content                       | Configuration                   | Extract URLs of JS files           |                                                                                                                                |
| Extract URLs of JS files         | HTML Extract                    | Extracts `src` attributes of `<script>` tags        | Fetch Website HTML              | Split URLs of JS files             |                                                                                                                                |
| Split URLs of JS files           | Split Out                      | Splits JS URLs array into individual items           | Extract URLs of JS files        | Keep Relevant JS Files             |                                                                                                                                |
| Keep Relevant JS Files           | Filter                         | Filters JS URLs likely containing API endpoints      | Split URLs of JS files          | Fetch JS Content                  |                                                                                                                                |
| Fetch JS Content                 | HTTP Request                   | Downloads content of each relevant JS file           | Keep Relevant JS Files          | Extract API Endpoints             |                                                                                                                                |
| Extract API Endpoints            | Set                             | Applies regex to extract API endpoints from JS       | Fetch JS Content               | Check Endpoints Count             |                                                                                                                                |
| Check Endpoints Count            | If                              | Checks JS files with more than 4 endpoints           | Extract API Endpoints          | Split Endpoints / Insufficient Endpoints |                                                                                                                                |
| Split Endpoints                 | Split Out                      | Splits endpoints array into individual items          | Check Endpoints Count (true)   | Remove Duplicates                 |                                                                                                                                |
| Remove Duplicates               | Remove Duplicates               | Removes duplicate endpoints                            | Split Endpoints               | Merge Original And LLM Endpoints  |                                                                                                                                |
| Reference for Source Metadata   | NoOp                            | Maintains source metadata                              | Check Endpoints Count (true)   | AI Endpoints Analysis             |                                                                                                                                |
| Insufficient Endpoints          | NoOp                            | Handles discarded JS files with too few endpoints    | Check Endpoints Count (false)  | -                              |                                                                                                                                |
| AI Endpoints Analysis           | HTTP Request                   | Sends JS content to LLM for detailed API analysis    | Reference for Source Metadata  | Add Source Metadata              |                                                                                                                                |
| Add Source Metadata             | Set                             | Adds JS file URL to AI analysis results               | AI Endpoints Analysis          | Process Each Analyzed File        |                                                                                                                                |
| Process Each Analyzed File      | Split In Batches               | Iterates over AI analyses for processing and saving  | Add Source Metadata            | Pass File Name, No Binary / Format AI Results |                                                                                                                                |
| Format AI Results               | Set                             | Formats AI output, assigns file names                 | Process Each Analyzed File     | Prepare Endpoints File(s)          |                                                                                                                                |
| Prepare Endpoints File(s)       | Convert To File                | Converts AI analysis text to binary files             | Format AI Results              | Save Endpoints File(s)             |                                                                                                                                |
| Save Endpoints File(s)          | Read/Write File               | Saves AI analysis files to n8n instance storage       | Prepare Endpoints File(s)      | Process Each Analyzed File (continuation) |                                                                                                                                |
| Pass File Name, No Binary       | Set                             | Strips binary data for further processing             | Process Each Analyzed File     | Merge AI Analysis & File Name     |                                                                                                                                |
| Merge AI Analysis & File Name   | Merge                           | Combines AI analysis and file name metadata           | Pass File Name, No Binary / Add Source Metadata | Merge Initial AI Analysis & AI Agent Output |                                                                                                                                |
| Merge Initial AI Analysis & AI Agent Output | Merge              | Combines AI analysis with regex generation output     | Merge AI Analysis & File Name / Create Endpoints Regex With AI | Prepare Data for Regex Extraction / Create Endpoints Regex With AI |                                                                                                                                |
| Create Endpoints Regex With AI  | LangChain Agent                | AI agent generates regex expressions for API endpoints | Merge Initial AI Analysis & AI Agent Output / Regex Generation LLM / Auto-fixing Output Parser | Merge Initial AI Analysis & AI Agent Output | Sticky Note3: Validation tool - AI self-evaluates and improves regex iteratively.                                              |
| Regex Generation LLM            | LangChain LM Chat (OpenRouter) | Language model used by AI agent to generate regex     | Create Endpoints Regex With AI | Auto-fixing Output Parser         | Sticky Note1: Stage 2 - LLM Regex Generation using Claude 3.7 Sonnet.                                                          |
| Auto-fixing Output Parser       | LangChain Output Parser Autofixing | Repairs and parses AI regex output                      | Regex Generation LLM           | Parse AI Regex Output             |                                                                                                                                |
| Parse AI Regex Output           | LangChain Output Parser Structured | Parses AI output into structured regex expression      | Auto-fixing Output Parser      | Regex Validation Start           |                                                                                                                                |
| Regex Validation Start          | Execute Workflow Trigger       | Initiates regex validation sub-workflow                | Parse AI Regex Output          | Load Reference Endpoints / Merge Regex, File Name & Reference Endpoints | Sticky Note: Stage 3 - LLM Regex Validation with iterative improvement.                                                        |
| Load Reference Endpoints        | Read/Write File               | Loads saved AI analysis file for validation             | Regex Validation Start         | Extract Endpoints File Text       |                                                                                                                                |
| Extract Endpoints File Text     | Extract From File (text)       | Extracts text content from loaded file                  | Load Reference Endpoints       | Merge Regex, File Name & Reference Endpoints |                                                                                                                                |
| Merge Regex, File Name & Reference Endpoints | Merge                 | Combines regex, filename, and reference file content   | Extract Endpoints File Text / Regex Validation Start | Evaluate LLM Regex              |                                                                                                                                |
| Evaluate LLM Regex             | Set                             | Evaluates regex expression against reference content   | Merge Regex, File Name & Reference Endpoints | Validate LLM Regex              |                                                                                                                                |
| Validate LLM Regex             | LangChain Tool Workflow        | Runs regex validation and triggers agent iterations    | Evaluate LLM Regex             | Create Endpoints Regex With AI    |                                                                                                                                |
| Prepare Data for Regex Extraction | Set                           | Prepares AI analysis data and regex for execution       | Merge Initial AI Analysis & AI Agent Output | Execute LLM Regex              |                                                                                                                                |
| Execute LLM Regex              | Set                             | Executes regex to extract endpoints from AI analysis   | Prepare Data for Regex Extraction | Split LLM-extracted Endpoints    |                                                                                                                                |
| Split LLM-extracted Endpoints  | Split Out                      | Splits AI-extracted endpoints into individual items    | Execute LLM Regex              | Remove LLM Endpoints Duplicates   |                                                                                                                                |
| Remove LLM Endpoints Duplicates | Remove Duplicates              | Removes duplicates among AI-extracted endpoints         | Split LLM-extracted Endpoints  | Merge Original And LLM Endpoints / Prepare Source & File Name for Merge |                                                                                                                                |
| Merge Original And LLM Endpoints | Merge                         | Combines predefined regex endpoints with AI-extracted   | Remove Duplicates / Remove LLM Endpoints Duplicates | Final Merge with Enriched File Name | Sticky Note12: Final comparison and merging of endpoints.                                                                      |
| Prepare Source & File Name for Merge | Remove Duplicates           | Deduplicates by source and filename prior to merge      | Remove LLM Endpoints Duplicates | Final Merge with Enriched File Name |                                                                                                                                |
| Final Merge with Enriched File Name | Merge                       | Enriches combined endpoints with filename metadata      | Merge Original And LLM Endpoints / Prepare Source & File Name for Merge | Sort Merged Endpoints          |                                                                                                                                |
| Sort Merged Endpoints          | Sort                           | Sorts final endpoint list by filename and endpoints     | Final Merge with Enriched File Name | Reorder Output Fields            |                                                                                                                                |
| Reorder Output Fields          | Set                            | Sets final fields order and cleans data                 | Sort Merged Endpoints          | Export Comparison Results To Excel |                                                                                                                                |
| Export Comparison Results To Excel | Convert To File (XLSX)        | Exports final data to Excel file                         | Reorder Output Fields          | -                              |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Name: `Start API Discovery`  
   - No parameters needed.

2. **Create Set node for Configuration:**  
   - Name: `Configuration`  
   - Add two string fields:  
     - `URL`: Input target website URL (e.g., https://example.com)  
     - `User-Agent`: Set to `"Mozilla/5.0 (compatible; API-Discovery-Bot/1.0)"`  
   - Connect `Start API Discovery` to `Configuration`.

3. **Create HTTP Request node to fetch HTML:**  
   - Name: `Fetch Website HTML`  
   - URL: `={{ $json.URL }}` (dynamic from Configuration)  
   - Method: GET  
   - Timeout: 30000 ms  
   - Allow unauthorized certs: true  
   - Follow redirects enabled  
   - Send header `User-Agent` with value: `={{ $json["User-Agent"] }}`  
   - Set `Never error` on response.  
   - Connect `Configuration` → `Fetch Website HTML`.

4. **Create HTML Extract node:**  
   - Name: `Extract URLs of JS files`  
   - Operation: Extract HTML content  
   - CSS Selector: `script[src]`  
   - Attribute: `src`  
   - Return array of values  
   - Connect `Fetch Website HTML` → `Extract URLs of JS files`.

5. **Create Split Out node:**  
   - Name: `Split URLs of JS files`  
   - Field to split out: `JS URLs`  
   - Connect `Extract URLs of JS files` → `Split URLs of JS files`.

6. **Create Filter node:**  
   - Name: `Keep Relevant JS Files`  
   - Add OR conditions to keep URLs that:  
     - Start with `/` or `./`  
     - Contain `yimg`  
     - Contain the domain of the target URL (use expression `{{ $('Configuration').first().json.URL.extractDomain() }}`)  
     - Contain `/build/`, `/dist/`, `/frontend/`, `/packages/`, or `bundle`  
     - Domain contains `cdn`  
     - Start with `/dist`  
   - Connect `Split URLs of JS files` → `Keep Relevant JS Files`.

7. **Create HTTP Request node to fetch JS content:**  
   - Name: `Fetch JS Content`  
   - URL: Use expression to convert relative URLs to absolute:  
     ```js
     ={{ !!$json['JS URLs'].extractDomain() ? 
         $json['JS URLs'].replace(/^\/\//, 'https://') : 
         ($('Configuration').first().json.URL + '/' + $json['JS URLs']).replaceAll(/(?<!:)\//g, '/') }}
     ```  
   - Method: GET  
   - Batch size: 1  
   - Set `Never error` on response  
   - Connect `Keep Relevant JS Files` → `Fetch JS Content`.

8. **Create Set node to extract endpoints with predefined regex:**  
   - Name: `Extract API Endpoints`  
   - Add fields:  
     - `Endpoints` (array): Use expression to extract quoted URL-like strings excluding static assets, clean duplicates and filter short strings  
     - `JS` (boolean): true  
     - `Source` (string): original JS URL from `Split URLs of JS files`  
   - Connect `Fetch JS Content` → `Extract API Endpoints`.

9. **Create If node to check endpoint count > 4:**  
   - Name: `Check Endpoints Count`  
   - Condition: `$json.Endpoints.length > 4`  
   - Connect `Extract API Endpoints` → `Check Endpoints Count`.

10. **Create Split Out node:**  
    - Name: `Split Endpoints`  
    - Field to split out: `Endpoints`  
    - Include fields: `JS`, `Source`  
    - Connect `Check Endpoints Count` (true branch) → `Split Endpoints`.

11. **Create Remove Duplicates node:**  
    - Name: `Remove Duplicates`  
    - Connect `Split Endpoints` → `Remove Duplicates`.

12. **Create NoOp node:**  
    - Name: `Reference for Source Metadata`  
    - Connect `Check Endpoints Count` (true branch) → `Reference for Source Metadata`.

13. **Create NoOp node:**  
    - Name: `Insufficient Endpoints`  
    - Connect `Check Endpoints Count` (false branch) → `Insufficient Endpoints`.

14. **Create HTTP Request node to send JS content to AI:**  
    - Name: `AI Endpoints Analysis`  
    - URL: `https://openrouter.ai/api/v1/chat/completions`  
    - Method: POST  
    - JSON Body: system user prompts with JS content as user message (default model Gemini 2.5 Pro-Preview)  
    - Authentication: OpenRouter API credentials  
    - Retry on fail enabled with 5-second interval  
    - Connect `Reference for Source Metadata` → `AI Endpoints Analysis`.

15. **Create Set node to add source metadata:**  
    - Name: `Add Source Metadata`  
    - Add field `Source` from `Reference for Source Metadata` node  
    - Connect `AI Endpoints Analysis` → `Add Source Metadata`.

16. **Create Split In Batches node:**  
    - Name: `Process Each Analyzed File`  
    - Connect `Add Source Metadata` → `Process Each Analyzed File`.

17. **Create Set node to format AI results:**  
    - Name: `Format AI Results`  
    - Assign `Filename` with padded batch index, `Source`, and AI message content fields  
    - Connect `Process Each Analyzed File` (batch output) → `Format AI Results`.

18. **Create Convert To File node:**  
    - Name: `Prepare Endpoints File(s)`  
    - Operation: toText  
    - Source Property: AI analysis message content  
    - FileName: use filename field  
    - Connect `Format AI Results` → `Prepare Endpoints File(s)`.

19. **Create Read/Write File node:**  
    - Name: `Save Endpoints File(s)`  
    - Operation: Write  
    - FileName: from binary data fileName  
    - Connect `Prepare Endpoints File(s)` → `Save Endpoints File(s)`.

20. **Create Set node to strip binary:**  
    - Name: `Pass File Name, No Binary`  
    - Strip binary data, keep metadata  
    - Connect `Process Each Analyzed File` (continuation) → `Pass File Name, No Binary`.

21. **Create Merge node:**  
    - Name: `Merge AI Analysis & File Name`  
    - Mode: Combine  
    - Connect `Pass File Name, No Binary` and `Add Source Metadata` → `Merge AI Analysis & File Name`.

22. **Create Merge node:**  
    - Name: `Merge Initial AI Analysis & AI Agent Output`  
    - Mode: Combine  
    - Connect `Merge AI Analysis & File Name` and `Create Endpoints Regex With AI` → `Merge Initial AI Analysis & AI Agent Output`.

23. **Create LangChain Agent node:**  
    - Name: `Create Endpoints Regex With AI`  
    - System message prompts AI to generate regex expressions based on AI analysis reports.  
    - Max iterations: 4  
    - Connect `Merge Initial AI Analysis & AI Agent Output` → `Create Endpoints Regex With AI`.

24. **Create LangChain LM Chat node:**  
    - Name: `Regex Generation LLM`  
    - Model: `anthropic/claude-3.7-sonnet`  
    - Temperature: 0.2  
    - Connect `Create Endpoints Regex With AI` → `Regex Generation LLM`.

25. **Create LangChain Output Parser Autofixing node:**  
    - Name: `Auto-fixing Output Parser`  
    - Connect `Regex Generation LLM` → `Auto-fixing Output Parser`.

26. **Create LangChain Output Parser Structured node:**  
    - Name: `Parse AI Regex Output`  
    - Schema expecting field `n8nexpression` containing regex expression string.  
    - Connect `Auto-fixing Output Parser` → `Parse AI Regex Output`.

27. **Create Execute Workflow Trigger node:**  
    - Name: `Regex Validation Start`  
    - Parameters: pass `query.n8nexpression` and `Filename` as inputs to sub-workflow.  
    - Connect `Parse AI Regex Output` → `Regex Validation Start`.

28. **Create Read/Write File node:**  
    - Name: `Load Reference Endpoints`  
    - File selector: use filename from inputs  
    - Connect `Regex Validation Start` → `Load Reference Endpoints`.

29. **Create Extract From File node:**  
    - Name: `Extract Endpoints File Text`  
    - Operation: text  
    - Connect `Load Reference Endpoints` → `Extract Endpoints File Text`.

30. **Create Merge node:**  
    - Name: `Merge Regex, File Name & Reference Endpoints`  
    - Mode: Combine  
    - Connect `Extract Endpoints File Text` and `Regex Validation Start` → `Merge Regex, File Name & Reference Endpoints`.

31. **Create Set node:**  
    - Name: `Evaluate LLM Regex`  
    - Evaluate expression in `query.n8nexpression` on file content to test regex.  
    - Connect `Merge Regex, File Name & Reference Endpoints` → `Evaluate LLM Regex`.

32. **Create LangChain Tool Workflow node:**  
    - Name: `Validate LLM Regex`  
    - Calls sub-workflow to validate regex, returns validation results.  
    - Connect `Evaluate LLM Regex` → `Validate LLM Regex`.

33. **Create Set node:**  
    - Name: `Prepare Data for Regex Extraction`  
    - Sets AI analysis text and regex expression for execution.  
    - Connect `Merge Initial AI Analysis & AI Agent Output` → `Prepare Data for Regex Extraction`.

34. **Create Set node:**  
    - Name: `Execute LLM Regex`  
    - Evaluates regex on AI analysis data, outputs array of endpoints, marks flag `LLM` true.  
    - Connect `Prepare Data for Regex Extraction` → `Execute LLM Regex`.

35. **Create Split Out node:**  
    - Name: `Split LLM-extracted Endpoints`  
    - Splits endpoints array into individual items.  
    - Connect `Execute LLM Regex` → `Split LLM-extracted Endpoints`.

36. **Create Remove Duplicates node:**  
    - Name: `Remove LLM Endpoints Duplicates`  
    - Removes duplicates from AI-extracted endpoints.  
    - Connect `Split LLM-extracted Endpoints` → `Remove LLM Endpoints Duplicates`.

37. **Create Merge node:**  
    - Name: `Merge Original And LLM Endpoints`  
    - Combines predefined regex endpoints with AI-extracted endpoints, matching on Endpoints and Source.  
    - Connect `Remove Duplicates` and `Remove LLM Endpoints Duplicates` → `Merge Original And LLM Endpoints`.

38. **Create Remove Duplicates node:**  
    - Name: `Prepare Source & File Name for Merge`  
    - Deduplicates on `Source` and `fileName`.  
    - Connect `Remove LLM Endpoints Duplicates` → `Prepare Source & File Name for Merge`.

39. **Create Merge node:**  
    - Name: `Final Merge with Enriched File Name`  
    - Enriches merged endpoints with file name metadata.  
    - Connect `Merge Original And LLM Endpoints` and `Prepare Source & File Name for Merge` → `Final Merge with Enriched File Name`.

40. **Create Sort node:**  
    - Name: `Sort Merged Endpoints`  
    - Sort by `fileName` and `Endpoints`.  
    - Connect `Final Merge with Enriched File Name` → `Sort Merged Endpoints`.

41. **Create Set node:**  
    - Name: `Reorder Output Fields`  
    - Orders output fields: `Endpoints`, `JS`, `Source`, `fileName`, `LLM`.  
    - Connect `Sort Merged Endpoints` → `Reorder Output Fields`.

42. **Create Convert To File node:**  
    - Name: `Export Comparison Results To Excel`  
    - Operation: XLSX with header row  
    - File name: `api_comparison.xlsx`  
    - Connect `Reorder Output Fields` → `Export Comparison Results To Excel`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| The workflow helps to automatically find hidden API endpoints in web platforms that don't provide public APIs by analyzing their JavaScript code. Modern web apps often embed internal API endpoints in JS files for frontend-backend communication.                              | Sticky Note2                                                                                                                    |
| This workflow is best suited for Knockout.js, Next.js/Nuxt.js, SPAs, and modern web apps with bundled JavaScript containing endpoint strings. It excludes dynamically generated endpoints, WebSocket-only communication, fully obfuscated code, or server-side only architectures. | Sticky Note6, Sticky Note7                                                                                                     |
| Regex extracts all URL-like strings with API-focused filtering, including relative and absolute URLs, excluding static assets and short strings.                                                                                                                                | Sticky Note5                                                                                                                   |
| Stage 1: Initial AI analysis with OpenRouter API (default Gemini 2.5 Pro-Preview) generates detailed endpoint descriptions.                                                                                                                                                       | Sticky Note10                                                                                                                  |
| Stage 2: LLM regex generation uses Claude 3.7 Sonnet model to create custom regex patterns from AI analysis reports.                                                                                                                                                              | Sticky Note1                                                                                                                   |
| Stage 3: LLM regex validation iteratively improves regex accuracy using a validation sub-workflow and agent tool.                                                                                                                                                                 | Sticky Note                                                                                                                    |
| Stage 4: LLM regex execution merges AI and predefined regex results, removes duplicates, and cleans data.                                                                                                                                                                         | Sticky Note13                                                                                                                  |
| Final comparison merges endpoints from both approaches, adds filename references, sorts and exports results to Excel.                                                                                                                                                             | Sticky Note12                                                                                                                  |
| Warning: The AI agent may fail to generate valid regex in some cases or stop due to max iterations. Predefined regex results and detailed AI endpoint descriptions serve as fallback.                                                                                            | Sticky Note11                                                                                                                  |
| Example target websites analyzed include sofaconcerts.org, bandcamp.com, otto.de, airbnb.com, jameda.de, and espn.co.uk with varying architectures and API endpoint discovery success.                                                                                            | Sticky Note15                                                                                                                  |
| GitHub gist with known ESPN API endpoints: https://gist.github.com/akeaswaran/b48b02f1c94f873c6655e7129910fc3b                                                                                                                                                                    | Sticky Note15                                                                                                                  |

---

**Disclaimer:** The text above is exclusively derived from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All data processed are legal and public.