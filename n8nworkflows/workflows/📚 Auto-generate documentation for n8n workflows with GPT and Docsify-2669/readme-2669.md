📚 Auto-generate documentation for n8n workflows with GPT and Docsify

https://n8nworkflows.xyz/workflows/---auto-generate-documentation-for-n8n-workflows-with-gpt-and-docsify-2669


# 📚 Auto-generate documentation for n8n workflows with GPT and Docsify

### 1. Workflow Overview

This n8n workflow automatically generates and serves documentation for all workflows within an n8n instance using Docsify.js, Mermaid.js, and OpenAI GPT models. It provides a dynamic documentation website with interactive features such as workflow overviews, tag-based filtering, live Markdown editing with preview, and auto-generated workflow diagrams.

The workflow is logically divided into the following main blocks:

- **1.1 Configuration and Initialization**  
  Sets up essential environment variables and HTML templates used across the documentation pages.

- **1.2 Webhook Request Handling and Routing**  
  Listens for HTTP requests and routes them based on requested file types and URL parameters to serve different documentation pages or perform editing/saving actions.

- **1.3 Workflow Data Retrieval and Processing**  
  Fetches workflows and workflow tags from the n8n instance API, sorts workflows, and prepares Markdown tables displaying workflows information.

- **1.4 Documentation Generation**  
  Uses OpenAI GPT to auto-generate workflow textual documentation and Mermaid.js to create workflow visualization diagrams.

- **1.5 File Operations**  
  Reads existing Markdown files for workflows, creates new documentation files if missing, saves edited documentation files to disk.

- **1.6 HTML Page Serving**  
  Serves the main documentation portal HTML, workflow documentation pages, and the live editor HTML page with Markdown preview and Mermaid rendering.

- **1.7 Edit and Save Workflow Documentation**  
  Provides the live Markdown editor interface for users to modify workflow docs and saves changes back to the file system.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration and Initialization

- **Overview:**  
  This block defines global configuration variables and reusable HTML header, style, and script snippets for Docsify and Mermaid.js integration.

- **Nodes Involved:**  
  - CONFIG (Set node)  
  - Sticky Note2 (Documentation)  

- **Node Details:**  

  - **CONFIG**  
    - Type: Set node  
    - Role: Defines key environment and HTML template variables such as:  
      - `project_path`: Local directory for storing documentation Markdown files (`./.n8n/test_docs`)  
      - `instance_url`: Constructed from environment variables for base URL of n8n instance  
      - `HTML_headers`: Meta tags, Docsify CSS, Mermaid.js script includes  
      - `HTML_styles_editor`: CSS styles for the live Markdown editor UI  
      - `HTML_docsify_include`: Docsify JS script inclusion tag  
    - Inputs: None (initialization)  
    - Outputs: Supplies configuration JSON to downstream nodes  
    - Edge Cases: Environment variables `N8N_PROTOCOL` and `N8N_HOST` must be set correctly, otherwise URL construction may fail  

---

#### 1.2 Webhook Request Handling and Routing

- **Overview:**  
  This block listens for incoming HTTP requests to the `/docsify` webhook path and routes requests based on requested file names and query parameters (`action`). It directs traffic to different processing flows such as serving main page, workflow docs, editing interface, or saving content.

- **Nodes Involved:**  
  - docsify (Webhook node)  
  - single workflow (Webhook node)  
  - md files (Switch node)  
  - doc action (Switch node)  
  - file types (Switch node)  
  - Is Action Edit?1 (If node)  
  - Is Action Edit?2 (If node)  
  - Is Action Save? (If node)  
  - doc action downstream nodes (mkdir, Load Doc File, Empty Set, Edit Fields, Edit Page, etc.)  
  - Sticky Notes 1, 7, 11 (Documentation)  

- **Node Details:**  

  - **docsify**  
    - Type: Webhook  
    - Role: Entry point for Docsify documentation requests at path `/135bc21f-c7d0-4afe-be73-f984d444b43b`  
    - Inputs: HTTP GET or POST requests  
    - Outputs: Passes requests to CONFIG and then routing nodes  
    - Edge Cases: Must be uniquely identified webhook URL; no authentication implemented (security risk in production)  

  - **md files**  
    - Type: Switch  
    - Role: Routes based on requested file names: `README.md`, `docs_*.md`, `summary.md`, and tag files  
    - Output branches determine whether main page, docs, summaries, or tags are served  

  - **doc action**  
    - Type: Switch  
    - Role: Routes based on query parameter `action` values: `view`, `edit`, `recreate`, `save`  
    - Controls workflow documentation lifecycle (viewing, editing, generating, saving)  

  - **Is Action Edit?1 & Is Action Edit?2**  
    - Type: If  
    - Role: Detects if the request is for editing action, controls loading or generating edit content and displaying editor page  

  - **Is Action Save?**  
    - Type: If  
    - Role: Detects save action, triggers file write and responds with OK  

  - **mkdir**  
    - Type: Execute Command  
    - Role: Ensures the documentation directory exists before file operations  

  - **Sticky Notes**  
    - Provide inline documentation about webhook routing and action handling  

- **Edge Cases:**  
  - Missing or malformed `action` query parameter  
  - Requests for unknown file names routed to fallback  
  - Concurrent save operations may cause race conditions  
  - No authentication on editing/saving allows unauthorized changes  

---

#### 1.3 Workflow Data Retrieval and Processing

- **Overview:**  
  Fetches all workflows or filtered workflows by tag, sorts them by last update date, and constructs Markdown tables summarizing their details for display in Docsify.

- **Nodes Involved:**  
  - Get All Workflows (n8n API node)  
  - Get Workflow tags (n8n API node)  
  - Sort-workflows (Sort node)  
  - Fill Workflow Table (Set node)  
  - Instance overview (HTML node)  
  - Workflow Tags (HTML node)  
  - Sticky Notes 4, 5, 6 (Documentation)  

- **Node Details:**  

  - **Get All Workflows**  
    - Type: n8n API node  
    - Role: Retrieves workflows, optionally filtered by tag extracted from URL  
    - Credentials: Requires n8n API credentials with read access  
    - Edge Cases: API rate limits, empty results if no workflows match filter  

  - **Sort-workflows**  
    - Type: Sort  
    - Role: Orders workflows descending by `updatedAt` timestamp  

  - **Fill Workflow Table**  
    - Type: Set  
    - Role: Creates Markdown table rows with columns: Workflow name (linked), status (active/inactive), docs links (view/edit/recreate), created and updated timestamps, node count, trigger count  
    - Uses escaping to sanitize workflow names  

  - **Instance overview**  
    - Type: HTML  
    - Role: Combines all table rows into a Markdown table for Docsify rendering  

  - **Workflow Tags**  
    - Type: HTML  
    - Role: Generates Markdown list of unique workflow tags for sidebar navigation  

- **Edge Cases:**  
  - Workflows without tags or with unusual characters in names  
  - Large number of workflows may impact performance  

---

#### 1.4 Documentation Generation

- **Overview:**  
  Automatically generates detailed workflow documentation using OpenAI GPT-4 Turbo with a structured prompt, and creates Mermaid.js diagrams representing the workflow structure.

- **Nodes Involved:**  
  - Fetch Single Workflow1 (n8n API node)  
  - Generate Mermaid Chart (Code node)  
  - Basic LLM Chain (Langchain chainLlm node)  
  - OpenAI Chat Model (Langchain lmChatOpenAi node)  
  - Structured Output Parser (Langchain outputParserStructured node)  
  - Auto-fixing Output Parser (Langchain outputParserAutofixing node)  
  - Generated Doc (Set node)  
  - Sticky Note7, 11 (Documentation)  

- **Node Details:**  

  - **Fetch Single Workflow1**  
    - Type: n8n API node  
    - Role: Fetches a single workflow by ID parsed from file name, to provide detailed structure for documentation  
    - Credentials: Requires n8n API credentials  

  - **Generate Mermaid Chart**  
    - Type: Code  
    - Role: Creates Mermaid.js flowchart syntax representing workflow nodes and connections  
    - Logic:  
      - Excludes sticky notes  
      - Maps node types to Mermaid shapes (rhombus for if/switch, subroutine for code nodes, etc.)  
      - Handles disabled nodes by strike-through  
      - Converts special characters into HTML entities  
      - Generates connections with labels for main or other connection types  
    - Edge Cases: Complex workflows with nested or unusual node types may require updates to this logic  

  - **Basic LLM Chain**  
    - Type: Langchain chainLlm  
    - Role: Sends workflow JSON data to GPT with instructions to generate Markdown sections: workflow description and nodes settings  
    - Receives structured JSON output  

  - **OpenAI Chat Model**  
    - Type: Langchain lmChatOpenAi  
    - Role: GPT-4 Turbo model invocation, with timeout and temperature controls  

  - **Structured Output Parser** and **Auto-fixing Output Parser**  
    - Parse and validate GPT output, attempt automatic fixes if parsing fails  

  - **Generated Doc**  
    - Combines GPT-generated description and Mermaid diagram into a Markdown document string  

- **Edge Cases:**  
  - GPT model timeouts or API errors  
  - GPT output format deviations handled by Auto-fixing parser  
  - Mermaid diagram generation errors due to unexpected workflow structure  

---

#### 1.5 File Operations

- **Overview:**  
  Reads existing documentation Markdown files, creates a blank template if none exists, saves new or edited documentation files in the project directory.

- **Nodes Involved:**  
  - Load Doc File (ReadWriteFile node)  
  - Extract from File (Extract from File node)  
  - Save New Doc File (ReadWriteFile node)  
  - Convert to File (Convert to File node)  
  - Blank Doc File (Set node)  
  - Edit Fields (Set node)  
  - Sticky Note10, 9 (Documentation)  

- **Node Details:**  

  - **Load Doc File**  
    - Type: ReadWriteFile  
    - Role: Reads Markdown file for the requested workflow doc if it exists  

  - **Extract from File**  
    - Type: Extract from File  
    - Role: Extracts text content from file data into `workflowdata` field  

  - **Blank Doc File**  
    - Type: Set  
    - Role: Creates a basic Markdown template with placeholders for workflow name, description, Mermaid schematic, and metadata table, used when no existing file is found  

  - **Convert to File**  
    - Type: Convert to File  
    - Role: Converts Markdown text to file format for saving  

  - **Save New Doc File**  
    - Type: ReadWriteFile  
    - Role: Writes updated or newly generated Markdown content to file system  

  - **Edit Fields**  
    - Type: Set  
    - Role: Extracts edited content from HTTP POST request body  

- **Edge Cases:**  
  - File system permissions errors when reading or writing  
  - Concurrent edits causing conflicts  
  - Missing directories handled by `mkdir` command node  

---

#### 1.6 HTML Page Serving

- **Overview:**  
  Constructs and serves the main documentation portal page, individual workflow Markdown pages rendered by Docsify, and a live Markdown editor page with Mermaid preview.

- **Nodes Involved:**  
  - Main Page (HTML node)  
  - Instance overview (HTML node)  
  - Workflow md content (HTML node)  
  - Edit Page (HTML node)  
  - Respond with main page HTML (Respond node)  
  - Respond with HTML (Respond node)  
  - Respond with markdown (Respond node)  
  - Sticky Note3, 8 (Documentation)  

- **Node Details:**  

  - **Main Page**  
    - Type: HTML  
    - Role: Serves the main Docsify HTML page with Mermaid initialization and Docsify config, referencing the summary sidebar and workflow docs base path  
    - Uses HTML headers and Docsify includes from CONFIG node  

  - **Instance overview** and **Workflow md content**  
    - Type: HTML  
    - Role: Serve Markdown content embedded in the page for Docsify to render  

  - **Edit Page**  
    - Type: HTML  
    - Role: Provides a split-screen Markdown editor with live preview powered by Docsify and Mermaid, with Save and Cancel buttons triggering HTTP POST save requests  
    - Contains embedded CSS styles and JavaScript for live rendering and save/cancel actions  

  - **Respond nodes**  
    - Send back the constructed HTML or Markdown with appropriate content types  

- **Edge Cases:**  
  - Browser compatibility for Mermaid rendering and Docsify JS  
  - Large Markdown content rendering delays  
  - Edit page save failures handled with alerts  

---

#### 1.7 Edit and Save Workflow Documentation

- **Overview:**  
  Supports editing workflow documentation by providing a live Markdown editor and saving changes back to the filesystem.

- **Nodes Involved:**  
  - Edit Page (HTML node)  
  - Edit Fields (Set node)  
  - Convert to File (Convert to File node)  
  - Save New Doc File (ReadWriteFile node)  
  - Respond OK on Save (Respond node)  

- **Node Details:**  

  - **Edit Page**  
    - Serves the HTML editor UI to users on `action=edit` requests  

  - **Edit Fields**  
    - Receives POSTed edited Markdown content in request body  

  - **Convert to File**  
    - Converts edited Markdown string to file format  

  - **Save New Doc File**  
    - Saves the updated Markdown file to disk  

  - **Respond OK on Save**  
    - Returns HTTP 200 OK to confirm save success  

- **Edge Cases:**  
  - Unsaved changes if user cancels or closes browser  
  - No authentication risks unauthorized edits  
  - File write errors  

---

### 3. Summary Table

| Node Name               | Node Type                    | Functional Role                                   | Input Node(s)                     | Output Node(s)                             | Sticky Note                                                                                      |
|-------------------------|-----------------------------|-------------------------------------------------|----------------------------------|--------------------------------------------|-------------------------------------------------------------------------------------------------|
| CONFIG                  | Set                         | Holds global config and HTML templates           | None                             | Merge4, Merge5                             | ## EDIT THIS! * `project_path` must be writable * Set correct instance_url for your environment  |
| docsify                 | Webhook                     | Entry point webhook for Docsify requests         | None                             | CONFIG, Merge4                             | ## Main Docsify webhook serves main HTML page with Docsify JS library                          |
| single workflow         | Webhook                     | Webhook for single workflow documentation requests| None                             | CONFIG, Merge5                             | ## Single page requests: handle readme, docs, save POST                                         |
| md files                | Switch                      | Routes requests by requested file name            | Merge5                           | Get All Workflows, doc action, Get Workflow tags, No Operation |                                                                                                |
| doc action              | Switch                      | Routes requests by `action` query param           | md files                        | mkdir, Load Doc File, Edit Fields, Empty Set |                                                                                                |
| mkdir                   | Execute Command             | Creates documentation directory if missing       | doc action                      | Merge1                                     |                                                                                                |
| Load Doc File           | ReadWriteFile               | Reads existing Markdown documentation file        | mkdir                          | Extract from File, Merge1                   | ## Load existing doc file when View or Edit requested                                          |
| Extract from File       | Extract from File           | Extracts text content from loaded file            | Load Doc File                  | Merge3                                     |                                                                                                |
| HasFile?                | If                          | Checks if file content is non-empty                | Merge1                         | Extract from File (true), Fetch Single Workflow1 (false) |                                                                                                |
| Fetch Single Workflow1  | n8n API                     | Fetches full workflow JSON by ID                   | HasFile? (false)               | Generate Mermaid Chart, Merge              |                                                                                                |
| Generate Mermaid Chart  | Code                        | Generates Mermaid.js diagram from workflow JSON    | Fetch Single Workflow1          | Merge                                      |                                                                                                |
| Basic LLM Chain         | Langchain chainLlm          | Sends workflow JSON to GPT for auto-doc generation | Auto-fixing Output Parser       | Merge2                                     |                                                                                                |
| OpenAI Chat Model       | Langchain lmChatOpenAi      | GPT-4 Turbo model for workflow doc generation      | Basic LLM Chain                | Structured Output Parser                    |                                                                                                |
| Structured Output Parser| Langchain outputParserStructured | Parses GPT JSON output                             | OpenAI Chat Model              | Auto-fixing Output Parser                   |                                                                                                |
| Auto-fixing Output Parser| Langchain outputParserAutofixing | Fixes parsing errors from GPT output               | Structured Output Parser        | Basic LLM Chain                             |                                                                                                |
| Generated Doc           | Set                         | Combines GPT output and Mermaid diagram into Markdown | Merge2                      | Convert to File, Is Action Edit?2           |                                                                                                |
| Convert to File         | Convert to File             | Converts Markdown string to file format            | Generated Doc, Edit Fields      | Save New Doc File                           |                                                                                                |
| Save New Doc File       | ReadWriteFile               | Saves Markdown documentation file to disk          | Convert to File                | Merge6                                     | ## Save new file after generating or editing                                                  |
| Is Action Edit?1        | If                          | Checks if action is `edit` for initial doc loading | doc action                    | Blank Doc File (true), Basic LLM Chain (false) |                                                                                                |
| Blank Doc File          | Set                         | Creates a blank documentation template             | Is Action Edit?1 (true)        | Is Action Edit?2                            |                                                                                                |
| Is Action Edit?2        | If                          | Checks if action is `edit` to serve editing page   | Generated Doc, Blank Doc File  | Edit Page (true), Workflow md content (false) |                                                                                                |
| Edit Page               | HTML                        | Serves live Markdown editor HTML page              | Is Action Edit?2 (true)         | Respond with HTML                           | ## Custom markdown editor with Mermaid preview and Save/Cancel buttons                        |
| Workflow md content     | HTML                        | Serves Markdown content for workflow documentation | Is Action Edit?2 (false)        | Respond with markdown                       |                                                                                                |
| Respond with markdown   | Respond to Webhook          | Responds with Markdown content                       | Workflow md content, Instance overview, Workflow Tags, Fallback file name | None                                    |                                                                                                |
| Respond with HTML       | Respond to Webhook          | Responds with HTML content                           | Edit Page                     | None                                       |                                                                                                |
| Respond with main page HTML | Respond to Webhook       | Responds with main Docsify portal HTML               | Main Page                     | None                                       | ## Construct main HTML page using config variables                                           |
| Instance overview       | HTML                        | Creates Markdown table overview of all workflows    | Fill Workflow Table           | Respond with markdown                       | ## Serve main Markdown table with workflow overview                                           |
| Fill Workflow Table     | Set                         | Formats workflows into Markdown table rows           | Sort-workflows               | Instance overview                           |                                                                                                |
| Sort-workflows          | Sort                        | Sorts workflows descending by last updated date      | Get All Workflows             | Fill Workflow Table                         |                                                                                                |
| Get All Workflows       | n8n API                     | Fetches workflows from n8n instance API               | md files                     | Sort-workflows                             |                                                                                                |
| Get Workflow tags       | n8n API                     | Fetches all workflow tags                              | md files                     | Workflow Tags                              |                                                                                                |
| Workflow Tags           | HTML                        | Creates Markdown list of unique workflow tags          | Get Workflow tags            | Respond with markdown                       | ## Serve left pane content with tags list                                                    |
| Edit Fields             | Set                         | Extracts edited Markdown content from POST body        | doc action                   | Convert to File                            |                                                                                                |
| Is Action Save?         | If                          | Checks if action is `save` to trigger file save         | Merge6                       | Respond OK on Save                         |                                                                                                |
| Respond OK on Save      | Respond to Webhook          | Sends HTTP 200 OK after successful save                | Is Action Save?              | None                                       |                                                                                                |
| Empty Set               | Set                         | Provides empty data set for recreate action             | doc action                   | Merge1                                     |                                                                                                |
| Passthrough             | NoOp                        | Passes data through for further merges                  | doc action                   | Merge3                                     |                                                                                                |
| Merge                   | Merge                       | Combines multiple inputs                                | Passthrough, Is Action Edit?1, Generate Mermaid Chart | Is Action Edit?2                        |                                                                                                |
| Merge1                  | Merge                       | Combines mkdir, Load Doc File, Empty Set data          | mkdir, Load Doc File, Empty Set | HasFile?                                   |                                                                                                |
| Merge2                  | Merge                       | Combines GPT output with Mermaid diagram                | Basic LLM Chain, Generate Mermaid Chart | Generated Doc                         |                                                                                                |
| Merge3                  | Merge                       | Combines Extract from File and Passthrough              | Extract from File, Passthrough | Merge                                      |                                                                                                |
| Merge4                  | Merge                       | Combines CONFIG and docsify webhook outputs             | CONFIG, docsify              | Main Page                                  |                                                                                                |
| Merge5                  | Merge                       | Combines CONFIG and single workflow webhook outputs     | CONFIG, single workflow      | file types                                 |                                                                                                |
| Merge6                  | Merge                       | Combines Save New Doc File output with Edit Fields       | Save New Doc File, Edit Fields | Is Action Save?                            |                                                                                                |
| No Operation, do nothing| NoOp                        | Default fallback for unknown files                       | md files                     | Fallback file name                         |                                                                                                |
| Fallback file name      | HTML                        | Serves fallback Markdown content for unknown files       | No Operation                | Respond with markdown                       | ## Handle missing pages by serving requested file name content                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Set node "CONFIG":**  
   - Define variables:  
     - `project_path` = `./.n8n/test_docs` (adjust for your environment)  
     - `instance_url` = construct using environment vars: `={{$env["N8N_PROTOCOL"]}}://{{$env["N8N_HOST"]}}`  
     - `HTML_headers`: Include meta tags, Docsify CSS, Mermaid.js script tags  
     - `HTML_styles_editor`: CSS styles for the live Markdown editor UI  
     - `HTML_docsify_include`: Docsify JS script tag inclusion  
   - Purpose: Global config and reusable HTML snippets  

2. **Create Webhook node "docsify":**  
   - Path: choose unique path (e.g. `/135bc21f-c7d0-4afe-be73-f984d444b43b`)  
   - Methods: GET, POST  
   - Output: connect to CONFIG, then routing nodes  

3. **Create Webhook node "single workflow":**  
   - Path: `/:file` (captures file name from URL)  
   - Methods: GET, POST  
   - Output: connect to CONFIG, then routing nodes  

4. **Create Switch node "file types":**  
   - Input: from Merge5 (merging CONFIG and single workflow outputs)  
   - Define branches to check file extensions or names: `.md` files to "md files" node  

5. **Create Switch node "md files":**  
   - Input: from file types  
   - Define branches for:  
     - `README.md` (main docs page)  
     - Files starting with `docs_` (individual workflow docs)  
     - `summary.md` (navigation sidebar)  
     - `tag-` prefixed files (filtered by tag)  
   - Direct requests accordingly  

6. **Create Switch node "doc action":**  
   - Input from md files (docs branch)  
   - Branch by query param `action`: `view`, `edit`, `recreate`, `save`  

7. **Create Execute Command node "mkdir":**  
   - Command: `mkdir -p {{$('CONFIG').first().json.project_path}}`  
   - Ensures docs directory exists before read/write  

8. **Create ReadWriteFile node "Load Doc File":**  
   - Reads file at `={{ $('CONFIG').first().json.project_path }}/{{ $json.params.file }}`  
   - Always output data for empty file handling  

9. **Create Extract From File node "Extract from File":**  
   - Extracts text content to field `workflowdata`  

10. **Create If node "HasFile?":**  
    - Condition: Check if loaded file JSON keys length > 0 (file exists)  
    - True: Extract from File flow  
    - False: Fetch Single Workflow flow  

11. **Create n8n API node "Fetch Single Workflow1":**  
    - Operation: Get workflow by ID parsed from filename (strip `docs_` prefix and `.md` suffix)  
    - Credentials: n8n API credentials with read rights  

12. **Create Code node "Generate Mermaid Chart":**  
    - JS code generating Mermaid.js syntax from workflow JSON  
    - Excludes sticky notes  
    - Maps node types to shapes, marks disabled nodes with strike-through  
    - Formats connections with labels  

13. **Create Langchain nodes to generate GPT documentation:**  
    - OpenAI Chat Model node with GPT-4 Turbo, timeout 2 mins, temperature 0.2  
    - Basic LLM Chain node with prompt instructing GPT to generate structured Markdown doc (workflow description + nodes settings)  
    - Structured Output Parser node with JSON schema example  
    - Auto-fixing Output Parser node for error resilience  
    - Chain connections: OpenAI Chat → Structured Parser → Auto-Fix Parser → Basic LLM Chain  

14. **Create Set node "Generated Doc":**  
    - Combines GPT-generated description and Mermaid chart into Markdown string in `workflowdata`  

15. **Create Set node "Blank Doc File":**  
    - Template Markdown for new docs with placeholders for workflow name, description, Mermaid schematic, and metadata  

16. **Create ReadWriteFile node "Save New Doc File":**  
    - Writes Markdown file to `={{ $('CONFIG').first().json.project_path }}/{{ $('CONFIG').first().json.params.file }}`  

17. **Create Set node "Edit Fields":**  
    - Extracts edited Markdown content from POST request body (`$json.body.content`)  

18. **Create HTML nodes:**  
    - "Main Page": Serves Docsify main HTML page with Mermaid initialization and Docsify config  
    - "Instance overview": Markdown table overview of all workflows  
    - "Workflow md content": Serves Markdown content of a single workflow doc  
    - "Edit Page": Live Markdown editor page with split view and Save/Cancel buttons, including Mermaid rendering and Docsify preview  

19. **Create Respond To Webhook nodes:**  
    - Respond with main page HTML  
    - Respond with Markdown content (for Docsify rendering)  
    - Respond with HTML content (for editor page)  
    - Respond OK after save operation  

20. **Create If nodes "Is Action Edit?1", "Is Action Edit?2", "Is Action Save?":**  
    - For routing based on `action` query parameter to decide between viewing, editing, saving, or recreating docs  

21. **Create Merge nodes as needed:**  
    - To combine data flows from different branches appropriately  

22. **Create supporting nodes:**  
    - Empty Set (for recreate with empty content)  
    - Passthrough (NoOp) for flow control  
    - No Operation, do nothing (fallback for unknown files)  
    - Fallback file name (serves basic message for missing files)  

23. **Set up credentials:**  
    - n8n API credentials with at least read rights on workflows  
    - OpenAI API credentials for GPT usage  

24. **Set environment variables:**  
    - `N8N_PROTOCOL` and `N8N_HOST` for constructing instance URL in CONFIG node  

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| The workflow is part of the broader n8n Observability Series, including Workflow Dashboard and Mermaid workflow visualizer workflows | Series overview with links: https://n8n.io/workflows/2269 and https://n8n.io/workflows/2378                                     |
| Docsify.js library is used for dynamic documentation rendering with live Markdown support                                | https://docsify.js.org/                                                                                                       |
| Mermaid.js is used to visualize workflows as flowcharts dynamically                                                    | https://mermaid.js.org/                                                                                                       |
| OpenAI GPT-4 Turbo is used for generating natural language workflow documentation                                        | Requires OpenAI API credentials and configured Langchain nodes                                                                |
| ⚠️ Security Note: No authentication implemented for editing; consider adding authentication for production deployments | Important for production to prevent unauthorized document modifications                                                       |
| Demonstration video showcasing this workflow available on LinkedIn                                                      | https://www.linkedin.com/feed/update/urn:li:activity:7276671057992847361/                                                      |

---

This detailed documentation provides a comprehensive understanding of the workflow's structure, logic, and component interactions, enabling advanced users or AI agents to reproduce, modify, and troubleshoot the workflow effectively.