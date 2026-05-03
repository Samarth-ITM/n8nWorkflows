Summarize Nextcloud documents with IONOS AI Model Hub for sovereign AI

https://n8nworkflows.xyz/workflows/summarize-nextcloud-documents-with-ionos-ai-model-hub-for-sovereign-ai-14544


# Summarize Nextcloud documents with IONOS AI Model Hub for sovereign AI

# 1. Workflow Overview

This workflow automatically scans a Nextcloud folder on a schedule, identifies files that were added or modified within the last 24 hours, downloads each eligible file, summarizes its contents using the IONOS AI Model Hub through n8n LangChain nodes, and saves the generated summary back into Nextcloud as a note-like text file.

Its main use case is sovereign document summarization: organizations can keep file storage in Nextcloud and run AI summarization through European infrastructure via IONOS AI Model Hub. It is suitable for periodic review of recently updated internal documents, reports, notes, or uploaded files.

## 1.1 Scheduled Input Reception

The workflow starts automatically every 23 hours and requests the contents of a Nextcloud folder.

## 1.2 File Filtering

The listed folder items are filtered so that only non-folder entries modified within the last 24 hours continue through the workflow.

## 1.3 File Retrieval and AI Preparation

Each selected file is downloaded from Nextcloud, then passed into the LangChain summarization pipeline. A binary data loader reads the downloaded file, and a token splitter prepares large documents for chunked summarization.

## 1.4 AI Summarization

The summarization chain uses the IONOS Cloud Chat Model with the selected Llama model to produce a summary from the document content.

## 1.5 Result Storage

The resulting summary is uploaded back to Nextcloud into the `/Notes/` folder using a generated filename prefixed with `Summary_`.

---

# 2. Block-by-Block Analysis

## Block 1: Scheduled Retrieval of Nextcloud Folder Contents

### Overview

This block is responsible for triggering the workflow on a recurring basis and retrieving the contents of a target Nextcloud folder. It establishes the source dataset that all later processing depends on.

### Nodes Involved

- Schedule Trigger
- List a folder

### Node Details

#### 1. Schedule Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a fixed interval.
- **Configuration choices:**  
  Configured to run every 23 hours.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input
  - Output → `List a folder`
- **Version-specific requirements:**  
  Uses `typeVersion: 1.3`.
- **Edge cases or potential failure types:**  
  - Workflow must be activated for the schedule to run.
  - If the n8n instance is offline at trigger time, execution timing may drift depending on deployment setup.
- **Sub-workflow reference:**  
  None.

#### 2. List a folder

- **Type and technical role:** `n8n-nodes-base.nextCloud`  
  Reads the contents of a folder from Nextcloud.
- **Configuration choices:**  
  - Resource: `folder`
  - Operation: `list`
  - Authentication: OAuth2
  - Uses configured Nextcloud OAuth2 credentials
- **Key expressions or variables used:**  
  None shown in the exported configuration; the folder path is expected to be set directly in the node UI.
- **Input and output connections:**  
  - Input ← `Schedule Trigger`
  - Output → `New Files Only`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - OAuth2 credential failure or expired token
  - Folder path missing or misconfigured
  - Insufficient permissions to list the folder
  - Empty folder result
  - Unexpected metadata format from Nextcloud
- **Sub-workflow reference:**  
  None.

---

## Block 2: Filtering Recently Modified Files

### Overview

This block removes irrelevant items from the folder listing. Only files modified in the last 24 hours are preserved, and directories are explicitly excluded.

### Nodes Involved

- New Files Only

### Node Details

#### 1. New Files Only

- **Type and technical role:** `n8n-nodes-base.code`  
  Executes custom JavaScript to filter the incoming list of files.
- **Configuration choices:**  
  The code:
  - Calculates a cutoff timestamp of 24 hours before the current time
  - Iterates over all incoming items
  - Excludes any item where `type === 'directory'`
  - Parses `lastmod` or `lastModified`
  - Keeps only items modified on or after the cutoff
- **Key expressions or variables used:**  
  - `const cutoff = new Date(Date.now() - 24 * 60 * 60 * 1000);`
  - `item.json`
  - `f.type`
  - `f.lastmod || f.lastModified || 0`
- **Input and output connections:**  
  - Input ← `List a folder`
  - Output → `Download a file`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`, meaning the modern Code node behavior applies.
- **Edge cases or potential failure types:**  
  - If Nextcloud returns timestamps in an unexpected format, `new Date(...)` may become invalid
  - If both `lastmod` and `lastModified` are absent, the fallback value `0` causes the file to be excluded
  - Time zone interpretation may affect borderline cases near the 24-hour threshold
  - Large folders may create many items and increase downstream processing load
- **Sub-workflow reference:**  
  None.

---

## Block 3: File Download and Document Preparation

### Overview

This block downloads each selected file from Nextcloud and prepares it for document-based AI summarization. It uses a binary data loader and a token-based splitter to transform the file into chunks suitable for the summarization chain.

### Nodes Involved

- Download a file
- Default Data Loader
- Token Splitter

### Node Details

#### 1. Download a file

- **Type and technical role:** `n8n-nodes-base.nextCloud`  
  Downloads the file selected by the filtering step.
- **Configuration choices:**  
  - Operation: `download`
  - Authentication: OAuth2
  - Path expression reconstructs the file path from the listed item
- **Key expressions or variables used:**  
  - `={{ '/' + decodeURIComponent($json.path) }}`
- **Input and output connections:**  
  - Input ← `New Files Only`
  - Output → `Summarization Chain`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Invalid or improperly encoded path
  - File deleted or moved after folder listing but before download
  - OAuth2 permission problems
  - Large files may stress memory or execution limits
  - Some file types may download successfully but fail later in parsing
- **Sub-workflow reference:**  
  None.

#### 2. Default Data Loader

- **Type and technical role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
  Converts binary file content into a document object that the summarization chain can consume.
- **Configuration choices:**  
  - Data type: `binary`
  - Uses default loader options
- **Key expressions or variables used:**  
  None explicitly configured.
- **Input and output connections:**  
  - Input from `Token Splitter` through the AI text splitter connector
  - Output → `Summarization Chain` via `ai_document`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`. Requires LangChain nodes package enabled in n8n.
- **Edge cases or potential failure types:**  
  - Unsupported document format
  - Binary property not found if upstream data is malformed
  - Parsing issues for PDFs, Office files, or unusual encodings depending on installed capabilities
- **Sub-workflow reference:**  
  None.

#### 3. Token Splitter

- **Type and technical role:** `@n8n/n8n-nodes-langchain.textSplitterTokenSplitter`  
  Splits long text into token-based chunks so the summarization chain can process larger documents reliably.
- **Configuration choices:**  
  - Chunk size: `3000`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output → `Default Data Loader` via `ai_textSplitter`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`. Requires LangChain text-splitting support.
- **Edge cases or potential failure types:**  
  - Chunk size may still be too large depending on actual tokenizer behavior and model limits
  - If the file cannot be converted to text, splitting is irrelevant and downstream processing fails
- **Sub-workflow reference:**  
  None.

### Important structural note for this block

In this exported workflow, `Token Splitter` is connected to `Default Data Loader`, and `Default Data Loader` is connected to `Summarization Chain`, but there is no visible main connection from `Download a file` to `Default Data Loader`. In many LangChain node patterns, the document loader can still read the binary from the execution context when wired into the chain, but in practice this is the part most likely to require verification after import. If summaries do not generate, the first thing to validate is whether the loader is receiving the downloaded binary as expected.

---

## Block 4: AI Summarization with IONOS Model Hub

### Overview

This block performs the actual summarization by combining the downloaded document content with an LLM hosted through IONOS AI Model Hub. The summarization chain orchestrates the document ingestion and calls the model to generate the final summary text.

### Nodes Involved

- Summarization Chain
- IONOS Cloud Chat Model

### Node Details

#### 1. Summarization Chain

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainSummarization`  
  A LangChain summarization node that accepts a document source and an LLM, then produces a condensed summary.
- **Configuration choices:**  
  - Operation mode: `documentLoader`
  - Options: default
- **Key expressions or variables used:**  
  None explicitly configured in parameters.
- **Input and output connections:**  
  - Main input ← `Download a file`
  - AI document input ← `Default Data Loader`
  - AI language model input ← `IONOS Cloud Chat Model`
  - Output → `Upload a file`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`. Requires compatible LangChain package support in the installed n8n version.
- **Edge cases or potential failure types:**  
  - Missing document input or binary parsing failures
  - Model connectivity or credential issues
  - Token/context limitations on very large documents
  - Empty summary if the file content is blank or not extractable
  - Some imported workflows may need re-selection of model and document loader references in the UI
- **Sub-workflow reference:**  
  None.

#### 2. IONOS Cloud Chat Model

- **Type and technical role:** `@ionos-cloud/n8n-nodes-ionos-cloud.ionosCloudChatModel`  
  Supplies the LLM used by the summarization chain.
- **Configuration choices:**  
  - Model: `meta-llama/Llama-3.3-70B-Instruct`
  - Options: default
  - Uses IONOS Cloud API credentials
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output → `Summarization Chain` via `ai_languageModel`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`. Requires the `@ionos-cloud/n8n-nodes-ionos-cloud` package to be installed.
- **Edge cases or potential failure types:**  
  - Invalid API token
  - Region or service availability constraints
  - Rate limits or temporary provider-side failures
  - Model name changes or deprecation in the IONOS catalog
- **Sub-workflow reference:**  
  None.

---

## Block 5: Saving the Summary Back to Nextcloud

### Overview

This block takes the generated summary text and writes it into a file in Nextcloud. The output file is stored under `/Notes/` and includes basic metadata such as original filename and processing date.

### Nodes Involved

- Upload a file

### Node Details

#### 1. Upload a file

- **Type and technical role:** `n8n-nodes-base.nextCloud`  
  Uploads a generated text/markdown-like file to Nextcloud.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Destination path dynamically built from the original filename
  - File content dynamically composed using the summary output
- **Key expressions or variables used:**  
  - Path:  
    `=/Notes/Summary_{{ $node["Download a file"].json.path.split('/').pop() }}`
  - Content:
    - Original filename: `{{ $node["Download a file"].json.path.split('/').pop() }}`
    - Date: `{{ $today.format('yyyy-MM-dd') }}`
    - Summary text: `{{ $json.response.text }}`
- **Input and output connections:**  
  - Input ← `Summarization Chain`
  - No downstream output
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - `/Notes/` folder missing
  - Permission denied for writing
  - Filename collisions if the same source file is processed multiple times
  - `$json.response.text` may be empty or differently shaped if the summarization node output format changes
  - Invalid characters in original filenames may affect destination path creation
- **Sub-workflow reference:**  
  None.

---

## Block 6: Documentation and In-Canvas Notes

### Overview

These sticky notes do not execute logic, but they are important because they document requirements, expected flow, and the intended grouping of the workflow. They should be preserved when reproducing the workflow because they clarify operational prerequisites.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### 1. Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Provides overall workflow documentation and requirements.
- **Configuration choices:**  
  Large descriptive note placed on the canvas.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  None; non-executable.
- **Sub-workflow reference:**  
  None.

#### 2. Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the first processing section.
- **Configuration choices:**  
  Content: `## 1. Get new or modified files`
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 3. Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the processing and summarization section.
- **Configuration choices:**  
  Content: `## 2. Process & summarize`
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 4. Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels the output-saving section.
- **Configuration choices:**  
  Content: `## 3. Save the results and get a Note`
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Launches the workflow every 23 hours |  | List a folder | ## 1. Get new or modified files |
| List a folder | n8n-nodes-base.nextCloud | Lists files and folders from a target Nextcloud folder | Schedule Trigger | New Files Only | ## 1. Get new or modified files |
| New Files Only | n8n-nodes-base.code | Filters out directories and keeps only files modified in the last 24 hours | List a folder | Download a file | ## 1. Get new or modified files |
| Download a file | n8n-nodes-base.nextCloud | Downloads each eligible file from Nextcloud | New Files Only | Summarization Chain | ## 2. Process & summarize |
| Summarization Chain | @n8n/n8n-nodes-langchain.chainSummarization | Generates a summary from the loaded document using the configured LLM | Download a file; Default Data Loader; IONOS Cloud Chat Model | Upload a file | ## 2. Process & summarize |
| Default Data Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads downloaded binary content as a LangChain document | Token Splitter | Summarization Chain | ## 2. Process & summarize |
| Token Splitter | @n8n/n8n-nodes-langchain.textSplitterTokenSplitter | Splits long documents into 3000-token chunks |  | Default Data Loader | ## 2. Process & summarize |
| IONOS Cloud Chat Model | @ionos-cloud/n8n-nodes-ionos-cloud.ionosCloudChatModel | Provides the LLM used for summarization |  | Summarization Chain | ## 2. Process & summarize |
| Upload a file | n8n-nodes-base.nextCloud | Saves the summary into `/Notes/` in Nextcloud | Summarization Chain |  | ## 3. Save the results and get a Note |
| Sticky Note | n8n-nodes-base.stickyNote | Documents workflow purpose, requirements, and high-level flow |  |  | ## Sovereign AI Document Summarizer<br>### Nextcloud + IONOS AI Model Hub<br><br>This workflow automates document summarization while keeping full digital sovereignty. By combining Nextcloud for file storage and the IONOS AI Model Hub, your sensitive documents stay on European infrastructure.<br><br>**Requirements checklist:**<br>- [ ] Nextcloud OAuth2 credentials configured (read + write)<br>- [ ] IONOS Cloud API token added to credentials<br>- [ ] `@ionos-cloud/n8n-nodes-ionos-cloud` node package installed<br>- [ ] LangChain AI nodes enabled (`@n8n/n8n-nodes-langchain`)<br>- [ ] `/Notes/` folder created in your Nextcloud instance<br>- [ ] Folder path set in the **List a folder** node<br><br>**Data flow:**<br>Schedule → List Folder → Filter → Download → Summarize → Upload Summary |
| Sticky Note1 | n8n-nodes-base.stickyNote | Labels the retrieval/filtering section |  |  | ## 1. Get new or modified files |
| Sticky Note2 | n8n-nodes-base.stickyNote | Labels the AI processing section |  |  | ## 2. Process & summarize |
| Sticky Note3 | n8n-nodes-base.stickyNote | Labels the output section |  |  | ## 3. Save the results and get a Note |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Sovereign AI Document Summarizer Nextcloud + IONOS AI Model Hub`.

2. **Confirm prerequisites before adding nodes:**
   - Install the IONOS node package: `@ionos-cloud/n8n-nodes-ionos-cloud`
   - Ensure LangChain nodes are available: `@n8n/n8n-nodes-langchain`
   - Create a `/Notes/` folder in your Nextcloud instance
   - Prepare:
     - a Nextcloud OAuth2 credential with read and write access
     - an IONOS Cloud API credential/token

3. **Add a Schedule Trigger node.**
   - Node type: `Schedule Trigger`
   - Set interval mode to run every `23 hours`
   - This will be the workflow entry point.

4. **Add a Nextcloud node for folder listing.**
   - Name it `List a folder`
   - Node type: `Nextcloud`
   - Resource: `Folder`
   - Operation: `List`
   - Authentication: `OAuth2`
   - Select your Nextcloud OAuth2 credential
   - Set the folder path you want monitored
   - Connect `Schedule Trigger` → `List a folder`

5. **Add a Code node to keep only recently modified files.**
   - Name it `New Files Only`
   - Node type: `Code`
   - Paste this logic:
     ```javascript
     // Keep all files uploaded in the last 24 hours
     const cutoff = new Date(Date.now() - 24 * 60 * 60 * 1000);

     return $input.all().filter(item => {
       const f = item.json;
       // Ignore folders, only process files
       if (f.type === 'directory') return false;

       const modified = new Date(f.lastmod || f.lastModified || 0);
       return modified >= cutoff;
     });
     ```
   - Connect `List a folder` → `New Files Only`

6. **Add a Nextcloud download node.**
   - Name it `Download a file`
   - Node type: `Nextcloud`
   - Operation: `Download`
   - Authentication: `OAuth2`
   - Select the same Nextcloud credential
   - Set the path field to:
     ```javascript
     {{ '/' + decodeURIComponent($json.path) }}
     ```
   - Confirm the node outputs binary file data
   - Connect `New Files Only` → `Download a file`

7. **Add a Summarization Chain node.**
   - Name it `Summarization Chain`
   - Node type: `Summarization Chain`
   - Operation mode: `Document Loader`
   - Keep default options unless you need custom prompts or map-reduce behavior
   - Connect `Download a file` → `Summarization Chain` as the main connection

8. **Add an IONOS Cloud Chat Model node.**
   - Name it `IONOS Cloud Chat Model`
   - Node type: `IONOS Cloud Chat Model`
   - Select your IONOS API credential
   - Choose the model:
     `meta-llama/Llama-3.3-70B-Instruct`
   - Connect `IONOS Cloud Chat Model` → `Summarization Chain` using the `AI Language Model` connection

9. **Add a Default Data Loader node.**
   - Name it `Default Data Loader`
   - Node type: `Default Data Loader`
   - Set data type to `Binary`
   - Keep other options at default
   - Connect `Default Data Loader` → `Summarization Chain` using the `AI Document` connection

10. **Add a Token Splitter node.**
    - Name it `Token Splitter`
    - Node type: `Token Splitter`
    - Set chunk size to `3000`
    - Connect `Token Splitter` → `Default Data Loader` using the `AI Text Splitter` connection

11. **Verify the document loader receives the downloaded binary.**
    - This is important because exported AI workflows can require manual re-binding after recreation.
    - In the `Default Data Loader`, make sure it is configured to read the binary data produced by `Download a file`.
    - If needed, test with a sample PDF or text document and inspect whether the summarization chain receives actual document chunks.

12. **Add a Nextcloud upload node.**
    - Name it `Upload a file`
    - Node type: `Nextcloud`
    - Authentication: `OAuth2`
    - Select the same Nextcloud credential
    - Configure it to upload file content to a dynamic path
    - Use this path expression:
      ```javascript
      /Notes/Summary_{{ $node["Download a file"].json.path.split('/').pop() }}
      ```
    - Use this generated content:
      ```markdown
      ### 📂 New File Processed
      **File Name:** {{ $node["Download a file"].json.path.split('/').pop() }}
      **Processed on:** {{ $today.format('yyyy-MM-dd') }}

      ---
      #### 💡 AI Summary
      This file was added to your drive. Here is the summarization of your document:

      {{ $json.response.text }}
      ```
    - Connect `Summarization Chain` → `Upload a file`

13. **Add optional sticky notes to mirror the original canvas layout.**
    - One general note with requirements and flow
    - One note for section 1: `Get new or modified files`
    - One note for section 2: `Process & summarize`
    - One note for section 3: `Save the results and get a Note`

14. **Configure workflow settings if you want to match the source closely.**
    - Execution order: `v1`
    - Binary mode: `separate`

15. **Test the workflow manually before activation.**
    - Put one or more recently modified files in the watched Nextcloud folder
    - Run the workflow manually
    - Check:
      - folder listing returns files
      - code node keeps expected items
      - file download includes binary data
      - summarization chain returns a text summary
      - upload creates a file in `/Notes/`

16. **Activate the workflow** once testing succeeds.

### Required credentials

#### Nextcloud OAuth2
- Must support:
  - listing folders
  - downloading files
  - uploading files
- Common failure points:
  - wrong callback URL
  - insufficient scopes
  - expired refresh token
  - wrong base URL

#### IONOS Cloud API
- Must be valid for the AI Model Hub service
- Common failure points:
  - invalid token
  - missing service access
  - unavailable model identifier

### Expected input/output behavior

#### Input
- A Nextcloud folder containing files and possibly subfolders
- File metadata should include a path and modification timestamp

#### Intermediate
- Filtered file items
- Downloaded binary content
- Loaded document chunks from the data loader and token splitter
- AI-generated summary text

#### Output
- A summary file written to:
  `/Notes/Summary_<original filename>`

### Reproduction caution

Because the LangChain connections are hybrid rather than purely linear, the most important validation step is ensuring the `Default Data Loader` is correctly associated with the downloaded binary content. If you rebuild this and the chain produces empty output, inspect the binary payload and the loader configuration first.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates document summarization while keeping full digital sovereignty. By combining Nextcloud for file storage and the IONOS AI Model Hub, your sensitive documents stay on European infrastructure. | Workflow purpose |
| Requirements checklist: Nextcloud OAuth2 credentials configured (read + write); IONOS Cloud API token added to credentials; `@ionos-cloud/n8n-nodes-ionos-cloud` node package installed; LangChain AI nodes enabled (`@n8n/n8n-nodes-langchain`); `/Notes/` folder created in your Nextcloud instance; Folder path set in the **List a folder** node. | Operational prerequisites |
| Data flow: Schedule → List Folder → Filter → Download → Summarize → Upload Summary | High-level process |