Scan Single URLs for Security Vulnerabilities with GPT-4 (JS, PHP, Python)

https://n8nworkflows.xyz/workflows/scan-single-urls-for-security-vulnerabilities-with-gpt-4--js--php--python--10801


# Scan Single URLs for Security Vulnerabilities with GPT-4 (JS, PHP, Python)

### 1. Workflow Overview

This workflow is designed to analyze the security of single source code files accessed via a raw URL, focusing on JavaScript, PHP, or Python files. Leveraging AI agents specialized in each language, it scans for exploitable vulnerabilities in the code and produces structured JSON vulnerability reports. The results are then formatted into styled HTML reports and uploaded to Google Drive for easy access.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Collects user input via a web form, specifying a single file URL and selecting one or more AI language analyzers.

- **1.2 Language Selection & Routing:** Splits the user-selected AI analyzers and routes the workflow to the corresponding language-specific analysis branches (JavaScript, PHP, Python).

- **1.3 Language-Specific Analysis:** For each supported language, this block performs:
  - HTTP GET request to fetch the source code.
  - AI agent invocation to analyze the code using a specialized prompt.
  - Parsing and prettifying the AI JSON output.
  - Splitting and filtering results to remove empty findings.
  - Generating an HTML table and wrapping it in a styled HTML template.
  - Uploading the final HTML report to Google Drive.

- **1.4 Utility and Documentation:** Sticky notes provide setup instructions, usage guidelines, and contact information for support.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block gathers the user's input via a web form, allowing entry of a single file URL and selection of one or more AI language analyzers to run.

- **Nodes Involved:**  
  - `Form`

- **Node Details:**

  - **Form**  
    - Type: `formTrigger` — initiates the workflow upon user form submission.  
    - Configuration:  
      - Form Title: "AI-Powered Code Analyzer"  
      - Form Fields:  
        - "Single File URL" (text input with placeholder URL example)  
        - "AI-Powered Code Analyzer" (checkbox with options: AI JavaScript Expert, AI PHP Expert, AI Python Expert) — mandatory field, user must select at least one.  
      - Description: Clarifies only one option can be selected among three types (GitHub repository, domain crawl, single file URL).  
    - Input: User fills form on webhook URL.  
    - Output: JSON containing the URL and selected analyzers.  
    - Edge cases: No selection or invalid URL input; empty or malformed form submission.

---

#### 2.2 Language Selection & Routing

- **Overview:**  
  Splits the user-selected analyzer options and routes the workflow execution to the corresponding language-specific branches.

- **Nodes Involved:**  
  - `Split AI-Powered Code Analyzer (Single URL)`  
  - `Python Expert (Single URL)` (Switch)  
  - `PHP Expert (Single URL)` (Switch)  
  - `JavaScript Expert (Single URL)` (Switch)

- **Node Details:**

  - **Split AI-Powered Code Analyzer (Single URL)**  
    - Type: `splitOut`  
    - Configuration: Splits the array field `["AI-Powered Code Analyzer"]` from the form output into separate executions for each selected option.  
    - Input: Output from `Form` node.  
    - Output: Multiple executions, each with one selected analyzer.

  - **Python Expert (Single URL)**  
    - Type: `switch`  
    - Configuration: Routes if the item equals "AI Python Expert".  
    - Input: Output from `Split AI-Powered Code Analyzer` for each item.  
    - Output: Executes Python analysis branch if condition matches.  
    - Edge cases: Mismatched or missing selection fields.

  - **PHP Expert (Single URL)**  
    - Type: `switch`  
    - Configuration: Routes if the item equals "AI PHP Expert".  
    - Input/Output similar to Python switch.

  - **JavaScript Expert (Single URL)**  
    - Type: `switch`  
    - Configuration: Routes if the item equals "AI JavaScript Expert".  
    - Input/Output similar to Python switch.

---

#### 2.3 JavaScript Analysis Branch

- **Overview:**  
  Retrieves the JavaScript file via HTTP, analyzes it for vulnerabilities using a specialized AI agent, processes the results, and generates a styled HTML report uploaded to Google Drive.

- **Nodes Involved:**  
  - `HTTP Request JavaScript (Single URL)`  
  - `OpenAI JavaScript (Single URL)`  
  - `JavaScript Expert Agent (Single URL)`  
  - `Prettify JavaScript Results (Single URL)`  
  - `Split JavaScript Expert Results (Single URL)`  
  - `Remove JavaScript Empty Results (Single URL)`  
  - `Create HTML Table JavaScript (Single URL)`  
  - `Create HTML Template JavaScript (Single URL)`  
  - `Upload HTML JavaScript Report (Single URL)`

- **Node Details:**

  - **HTTP Request JavaScript (Single URL)**  
    - Type: `httpRequestTool`  
    - Configuration: Performs HTTP GET on the URL from the form input (`Single File URL`), with 20s timeout.  
    - Input: Form URL.  
    - Output: HTTP response body (JavaScript file content).  
    - Edge cases: HTTP errors, timeouts, empty response.

  - **OpenAI JavaScript (Single URL)**  
    - Type: `lmChatOpenAi` (OpenAI GPT-4.1-mini model)  
    - Credentials: OpenAI API key.  
    - Configuration: Sends the file URL text to the AI agent node.  
    - Input: URL string.  
    - Output: AI-generated JSON with vulnerability analysis.  
    - Edge cases: API rate limits, auth failure, empty or invalid AI response.

  - **JavaScript Expert Agent (Single URL)**  
    - Type: `langchain.agent`  
    - Configuration:  
      - System message describes a strict vulnerability detection protocol for JavaScript files, forbidding actions outside the URL list, truncating code bodies at 20,000 bytes, parsing with AST and regex heuristics, and returning only JSON-formatted vulnerability findings with no explanations or payloads.  
      - Limits on iterations, rate, timeouts, and output format strictly enforced.  
      - Vulnerabilities detected include DOM XSS, reflected/stored XSS, RCE via eval/new/Function, CSRF, info leaks, and others.  
    - Input: HTTP response body from previous node.  
    - Output: JSON string with vulnerability results.  
    - Edge cases: Invalid JSON from AI, empty results, AI rejection of illegal/offensive inputs.

  - **Prettify JavaScript Results (Single URL)**  
    - Type: `code`  
    - Configuration: Parses AI JSON output string, trims and cleans quotes, returns parsed JSON as structured data.  
    - Input: AI raw output string.  
    - Output: Parsed JSON object or error if invalid JSON.  
    - Edge cases: Parsing errors, empty outputs.

  - **Split JavaScript Expert Results (Single URL)**  
    - Type: `splitOut`  
    - Configuration: Splits the `results` array in the AI output JSON into individual items for further processing.  
    - Input: Parsed JSON object.  
    - Output: Individual vulnerability objects.

  - **Remove JavaScript Empty Results (Single URL)**  
    - Type: `filter`  
    - Configuration: Removes items where the `url` field does not exist or is empty (filters out empty results).  
    - Input: Individual results.  
    - Output: Only valid vulnerability findings.

  - **Create HTML Table JavaScript (Single URL)**  
    - Type: `html`  
    - Configuration: Converts filtered JSON results into an HTML table with capitalized headers.  
    - Input: Filtered results.  
    - Output: Raw HTML table as string.

  - **Create HTML Template JavaScript (Single URL)**  
    - Type: `html`  
    - Configuration: Wraps the HTML table in a styled HTML template with CSS for formatting and responsiveness.  
    - Input: HTML table string.  
    - Output: Complete styled HTML report.

  - **Upload HTML JavaScript Report (Single URL)**  
    - Type: `googleDrive`  
    - Credentials: Google Drive OAuth2  
    - Configuration: Uploads the HTML report as a text file to “My Drive” root folder with a filename pattern including current date and time.  
    - Input: HTML content.  
    - Output: Google Drive file metadata.  
    - Edge cases: Authentication failure, API quota, upload errors.

---

#### 2.4 PHP Analysis Branch

- **Overview:**  
  Mirrors the JavaScript branch but specialized for PHP code analysis with a dedicated AI agent and prompt.

- **Nodes Involved:**  
  - `HTTP Request PHP (Single URL)`  
  - `OpenAI PHP (Single URL)`  
  - `PHP Expert Agent (Single URL)`  
  - `Prettify PHP Results (Single URL)`  
  - `Split PHP Expert Results (Single URL)`  
  - `Remove PHP Empty Results (Single URL)`  
  - `Create HTML Table PHP (Single URL)`  
  - `Create HTML Template PHP (Single URL)`  
  - `Upload HTML PHP Report (Single URL)`

- **Node Details:**  
  Similar to JavaScript branch, with changes:  
  - AI prompt tailored to PHP vulnerabilities (SQLi, RCE, LFI/RFI, command injection, CSRF, insecure eval, insecure file operations, header injection, session management, etc.).  
  - HTTP GET fetches PHP file.  
  - Same output formatting and upload strategy.

---

#### 2.5 Python Analysis Branch

- **Overview:**  
  Similar to the JavaScript and PHP branches but focused on Python code vulnerabilities, with a specialized AI prompt.

- **Nodes Involved:**  
  - `HTTP Request Python (Single URL)`  
  - `OpenAI Python (Single URL)`  
  - `Python Expert Agent (Single URL)`  
  - `Prettify Python Results (Single URL)`  
  - `Split Python Expert Results (Single URL)`  
  - `Remove Python Empty Results (Single URL)`  
  - `Create HTML Table Python (Single URL)`  
  - `Create HTML Template Python (Single URL)`  
  - `Upload HTML Python Report (Single URL)`

- **Node Details:**  
  - AI prompt targets Remote Code Execution, command injection, insecure deserialization, SQL injection, path traversal, server-side template injection, CSRF, SSRF, insecure crypto, JWT/session handling, and other server-side vulnerabilities.  
  - HTTP GET fetches Python file.  
  - Same formatting and upload approach.

---

#### 2.6 Utility and Documentation

- **Overview:**  
  Sticky notes provide guidance on setup, usage, and support.

- **Nodes Involved:**  
  - `Sticky Note1` (Setup instructions for OpenAI and Google Drive)  
  - `Sticky Note2` (Usage instructions on input and AI analyzer selection)  
  - `Sticky Note6` (Label: "Form Input")  
  - `Sticky Note8` (Label: "JavaScript Analyzer")  
  - `Sticky Note10` (Label: "PHP Analyzer")  
  - `Sticky Note11` (Label: "Python Analyzer")  
  - `Sticky Note9` (Contact info for consulting)  
  - `Sticky Note3`, `Sticky Note5`, `Sticky Note7`, `Sticky Note13`, `Sticky Note14` (Empty or structural notes)

- **Node Details:**  
  - Provide important context for users and maintainers.  
  - Link to OpenAI API keys, Google Cloud console, and contact channels.

---

### 3. Summary Table

| Node Name                              | Node Type                  | Functional Role                            | Input Node(s)                        | Output Node(s)                             | Sticky Note                               |
|--------------------------------------|----------------------------|--------------------------------------------|------------------------------------|--------------------------------------------|-------------------------------------------|
| Form                                 | formTrigger                | Collect user URL and selected analyzers   | -                                  | Split AI-Powered Code Analyzer (Single URL) | Sticky Note6 ("Form Input")               |
| Split AI-Powered Code Analyzer (Single URL) | splitOut                   | Split selected languages into branches    | Form                               | Python Expert (Single URL), PHP Expert (Single URL), JavaScript Expert (Single URL) |                                           |
| Python Expert (Single URL)            | switch                    | Route to Python analysis branch            | Split AI-Powered Code Analyzer      | Python Expert Agent (Single URL)            | Sticky Note11 ("Python Analyzer")         |
| PHP Expert (Single URL)               | switch                    | Route to PHP analysis branch               | Split AI-Powered Code Analyzer      | PHP Expert Agent (Single URL)               | Sticky Note10 ("PHP Analyzer")            |
| JavaScript Expert (Single URL)        | switch                    | Route to JavaScript analysis branch       | Split AI-Powered Code Analyzer      | JavaScript Expert Agent (Single URL)       | Sticky Note8 ("JavaScript Analyzer")     |
| HTTP Request Python (Single URL)      | httpRequestTool           | Fetch Python source code                    | Python Expert (Single URL)           | Python Expert Agent (Single URL)            |                                           |
| OpenAI Python (Single URL)            | lmChatOpenAi              | AI model invocation for Python              | Python Expert (Single URL)           | Python Expert Agent (Single URL)            |                                           |
| Python Expert Agent (Single URL)      | langchain.agent           | Analyze Python code for vulnerabilities    | HTTP Request Python, OpenAI Python  | Prettify Python Results (Single URL)        |                                           |
| Prettify Python Results (Single URL) | code                      | Parse and clean AI JSON output              | Python Expert Agent                 | Split Python Expert Results (Single URL)   |                                           |
| Split Python Expert Results (Single URL) | splitOut                   | Split vulnerability results array           | Prettify Python Results             | Remove Python Empty Results (Single URL)   |                                           |
| Remove Python Empty Results (Single URL) | filter                    | Filter out empty or invalid results          | Split Python Expert Results          | Create HTML Table Python (Single URL)      |                                           |
| Create HTML Table Python (Single URL) | html                      | Convert JSON results to HTML table           | Remove Python Empty Results          | Create HTML Template Python (Single URL)   |                                           |
| Create HTML Template Python (Single URL) | html                      | Wrap table in styled HTML template           | Create HTML Table Python             | Upload HTML Python Report (Single URL)     |                                           |
| Upload HTML Python Report (Single URL) | googleDrive               | Upload final HTML report to Google Drive     | Create HTML Template Python          | -                                          |                                           |
| HTTP Request PHP (Single URL)         | httpRequestTool           | Fetch PHP source code                       | PHP Expert (Single URL)              | PHP Expert Agent (Single URL)               |                                           |
| OpenAI PHP (Single URL)               | lmChatOpenAi              | AI model invocation for PHP                 | PHP Expert (Single URL)              | PHP Expert Agent (Single URL)               |                                           |
| PHP Expert Agent (Single URL)         | langchain.agent           | Analyze PHP code for vulnerabilities       | HTTP Request PHP, OpenAI PHP         | Prettify PHP Results (Single URL)           |                                           |
| Prettify PHP Results (Single URL)    | code                      | Parse and clean AI JSON output              | PHP Expert Agent                    | Split PHP Expert Results (Single URL)      |                                           |
| Split PHP Expert Results (Single URL) | splitOut                   | Split vulnerability results array           | Prettify PHP Results                | Remove PHP Empty Results (Single URL)      |                                           |
| Remove PHP Empty Results (Single URL) | filter                    | Filter out empty or invalid results          | Split PHP Expert Results             | Create HTML Table PHP (Single URL)          |                                           |
| Create HTML Table PHP (Single URL)    | html                      | Convert JSON results to HTML table           | Remove PHP Empty Results             | Create HTML Template PHP (Single URL)       |                                           |
| Create HTML Template PHP (Single URL) | html                      | Wrap table in styled HTML template           | Create HTML Table PHP                | Upload HTML PHP Report (Single URL)          |                                           |
| Upload HTML PHP Report (Single URL)   | googleDrive               | Upload final HTML report to Google Drive     | Create HTML Template PHP             | -                                            |                                           |
| HTTP Request JavaScript (Single URL) | httpRequestTool           | Fetch JavaScript source code                | JavaScript Expert (Single URL)       | JavaScript Expert Agent (Single URL)        |                                           |
| OpenAI JavaScript (Single URL)        | lmChatOpenAi              | AI model invocation for JavaScript          | JavaScript Expert (Single URL)       | JavaScript Expert Agent (Single URL)        |                                           |
| JavaScript Expert Agent (Single URL)  | langchain.agent           | Analyze JavaScript code for vulnerabilities | HTTP Request JavaScript, OpenAI JS   | Prettify JavaScript Results (Single URL)    |                                           |
| Prettify JavaScript Results (Single URL) | code                      | Parse and clean AI JSON output              | JavaScript Expert Agent             | Split JavaScript Expert Results (Single URL) |                                           |
| Split JavaScript Expert Results (Single URL) | splitOut                   | Split vulnerability results array           | Prettify JavaScript Results          | Remove JavaScript Empty Results (Single URL) |                                           |
| Remove JavaScript Empty Results (Single URL) | filter                    | Filter out empty or invalid results          | Split JavaScript Expert Results      | Create HTML Table JavaScript (Single URL)  |                                           |
| Create HTML Table JavaScript (Single URL) | html                      | Convert JSON results to HTML table           | Remove JavaScript Empty Results      | Create HTML Template JavaScript (Single URL) |                                           |
| Create HTML Template JavaScript (Single URL) | html                      | Wrap table in styled HTML template           | Create HTML Table JavaScript         | Upload HTML JavaScript Report (Single URL)  |                                           |
| Upload HTML JavaScript Report (Single URL) | googleDrive               | Upload final HTML report to Google Drive     | Create HTML Template JavaScript      | -                                            |                                           |
| Sticky Note1                         | stickyNote                 | Setup instructions for OpenAI & Google Drive | -                                  | -                                            | Setup instructions with links to OpenAI API and Google Drive |
| Sticky Note2                         | stickyNote                 | Usage instructions for form and analyzers  | -                                  | -                                            | Input guidelines and mandatory AI analyzer selection          |
| Sticky Note6                         | stickyNote                 | Label for Form Input                         | -                                  | -                                            | "Form Input"                                                 |
| Sticky Note8                         | stickyNote                 | Label for JavaScript Analyzer                | -                                  | -                                            | "JavaScript Analyzer"                                        |
| Sticky Note10                        | stickyNote                 | Label for PHP Analyzer                        | -                                  | -                                            | "PHP Analyzer"                                              |
| Sticky Note11                        | stickyNote                 | Label for Python Analyzer                     | -                                  | -                                            | "Python Analyzer"                                           |
| Sticky Note9                         | stickyNote                 | Contact info for consulting and support     | -                                  | -                                            | Contact via LinkedIn and Email                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Node:**  
   - Type: `formTrigger`  
   - Configure form with title "AI-Powered Code Analyzer"  
   - Add two fields:  
     - Text input labeled "Single File URL" with placeholder example URL  
     - Checkbox labeled "AI-Powered Code Analyzer" with options: "AI JavaScript Expert", "AI PHP Expert", "AI Python Expert" (mandatory field)  
   - Save and expose webhook.

2. **Add Split Node for AI Analyzer Selection:**  
   - Type: `splitOut`  
   - Configure to split the field `["AI-Powered Code Analyzer"]` from form output.

3. **Add Switch Nodes for Language Routing:**  
   - Create three switch nodes named "Python Expert", "PHP Expert", "JavaScript Expert" respectively.  
   - Each switch routes if the current item equals the corresponding option string, e.g., "AI Python Expert".

4. **Build Python Analysis Branch:**  
   - Add `httpRequestTool` node labeled "HTTP Request Python (Single URL)". Set URL to `={{ $('Form').item.json['Single File URL'] }}` and timeout 20000 ms.  
   - Add OpenAI `lmChatOpenAi` node labeled "OpenAI Python (Single URL)", select model "gpt-4.1-mini", link OpenAI credentials.  
   - Connect output of "HTTP Request Python" and "OpenAI Python" nodes as inputs to a `langchain.agent` node labeled "Python Expert Agent (Single URL)".  
   - Configure the agent with the detailed system prompt specialized for Python vulnerability auditing (copy the provided prompt logic).  
   - Add a `code` node "Prettify Python Results", with JavaScript code that parses and cleans the JSON output from the AI agent.  
   - Add a `splitOut` node "Split Python Expert Results" to split the `results` array.  
   - Add a `filter` node "Remove Python Empty Results" to exclude items without a valid `url` field.  
   - Add an `html` node "Create HTML Table Python" to convert JSON to an HTML table with capitalize headers.  
   - Add another `html` node "Create HTML Template Python" with the provided CSS style and template wrapping the HTML table.  
   - Add a `googleDrive` node "Upload HTML Python Report", configure to save with a dynamic filename containing date/time, content from previous node, and link Google Drive OAuth2 credentials.  
   - Connect nodes in the order described.

5. **Build PHP Analysis Branch:**  
   - Repeat steps from Python branch substituting nodes for PHP:  
     - "HTTP Request PHP (Single URL)"  
     - "OpenAI PHP (Single URL)"  
     - "PHP Expert Agent (Single URL)" with PHP-specific system prompt  
     - "Prettify PHP Results (Single URL)"  
     - "Split PHP Expert Results (Single URL)"  
     - "Remove PHP Empty Results (Single URL)"  
     - "Create HTML Table PHP (Single URL)"  
     - "Create HTML Template PHP (Single URL)"  
     - "Upload HTML PHP Report (Single URL)"

6. **Build JavaScript Analysis Branch:**  
   - Repeat steps with JavaScript nodes and prompt:  
     - "HTTP Request JavaScript (Single URL)"  
     - "OpenAI JavaScript (Single URL)"  
     - "JavaScript Expert Agent (Single URL)" with JS-specific prompt  
     - "Prettify JavaScript Results (Single URL)"  
     - "Split JavaScript Expert Results (Single URL)"  
     - "Remove JavaScript Empty Results (Single URL)"  
     - "Create HTML Table JavaScript (Single URL)"  
     - "Create HTML Template JavaScript (Single URL)"  
     - "Upload HTML JavaScript Report (Single URL)"

7. **Connect Workflow:**  
   - Connect `Form` node output to `Split AI-Powered Code Analyzer`  
   - Connect split outputs to three switch nodes  
   - Connect each switch's matching output to corresponding HTTP Request and OpenAI nodes as inputs for the respective AI agent nodes.

8. **Credential Setup:**  
   - OpenAI API key credential configured and linked to OpenAI nodes.  
   - Google Drive OAuth2 credential configured and linked to Google Drive upload nodes.

9. **Validation and Testing:**  
   - Test with valid URLs for each language type (.js, .php, .py).  
   - Verify AI outputs are parsed correctly and HTML reports upload successfully.  
   - Handle exceptions and errors per node's error handling settings.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Setup instructions to obtain OpenAI API key and set up Google Drive OAuth credentials are included in Sticky Note1.                    | See [OpenAI API Keys](https://platform.openai.com/api-keys), [Google Cloud Console](https://console.cloud.google.com/) |
| Usage instructions emphasize that a user must provide a valid single raw file URL and select at least one AI language analyzer.       | Sticky Note2                                                                                        |
| Contact information for consulting and support is provided via LinkedIn and email.                                                     | LinkedIn: https://www.linkedin.com/in/javier-rieiro-2900b5354/ Email: pyus3r@gmail.com               |
| The AI agents strictly enforce not executing or fetching URLs outside the user-provided list, truncating large files, and returning only JSON with exploitable vulnerabilities. | Included in each language agent system prompt                                                       |
| HTML styling for reports uses responsive CSS and a clean table design for readability.                                                  | Templates in Create HTML Template nodes                                                             |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.