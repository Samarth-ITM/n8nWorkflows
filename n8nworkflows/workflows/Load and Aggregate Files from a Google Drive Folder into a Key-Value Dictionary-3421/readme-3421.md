Load and Aggregate Files from a Google Drive Folder into a Key-Value Dictionary

https://n8nworkflows.xyz/workflows/load-and-aggregate-files-from-a-google-drive-folder-into-a-key-value-dictionary-3421


# Load and Aggregate Files from a Google Drive Folder into a Key-Value Dictionary

### 1. Workflow Overview

This workflow automates the retrieval and aggregation of Google Docs files from a specified Google Drive folder into a single key-value dictionary. The dictionary maps each file name to its normalized content, facilitating use cases such as knowledge ingestion, prompt management, or feeding data into Retrieval-Augmented Generation (RAG) systems.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Input Reception:** Initiates the workflow when executed by another workflow.
- **1.2 File Retrieval:** Fetches all files from a specified Google Drive folder.
- **1.3 Content Download:** Downloads the full content of each Google Docs file.
- **1.4 Mapping:** Maps each file name to its corresponding content.
- **1.5 Aggregation:** Aggregates all individual mappings into a single dictionary with normalized newlines.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Reception

- **Overview:**  
  This block starts the workflow when triggered externally by another workflow, allowing integration into larger automation pipelines.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point; triggers this workflow upon external execution.  
    - Configuration: Uses a JSON example input `{}` as a placeholder; no specific input data required.  
    - Inputs: None (trigger node)  
    - Outputs: Passes empty JSON to the next node.  
    - Edge Cases: If the triggering workflow fails to call this node properly, this workflow will not start.  
    - Version: 1.1  

---

#### 1.2 File Retrieval

- **Overview:**  
  Retrieves all files located in a specified Google Drive folder. By default, it targets the root folder but can be customized to any folder ID.

- **Nodes Involved:**  
  - Get files from folder  
  - Sticky Note (Step 1)

- **Node Details:**  

  - **Get files from folder**  
    - Type: Google Drive node  
    - Role: Lists all files in the target folder.  
    - Configuration:  
      - Resource: fileFolder  
      - Filter: folderId set to `"root"` (default root folder).  
      - Credentials: Uses OAuth2 credentials for Google Drive.  
    - Inputs: Receives trigger from previous node.  
    - Outputs: Emits a list of files metadata (including file IDs and names).  
    - Edge Cases:  
      - Folder ID invalid or inaccessible → authentication or permission errors.  
      - Empty folder → no files output, downstream nodes receive no data.  
    - Version: 3  

  - **Sticky Note (Step 1)**  
    - Content: "## Step1\nDefine folder you want to search all files in."  
    - Role: Instructional note for users to customize folder selection.  

---

#### 1.3 Content Download

- **Overview:**  
  Downloads the full content of each Google Docs file identified in the previous step.

- **Nodes Involved:**  
  - Download Google Docs  
  - Sticky Note1 (Step 2)

- **Node Details:**  

  - **Download Google Docs**  
    - Type: Google Docs node  
    - Role: Retrieves the content of each Google Docs file by document URL (file ID).  
    - Configuration:  
      - Operation: get  
      - Document URL: Set dynamically to the file ID from the previous node (`={{ $json.id }}`).  
      - Credentials: Uses OAuth2 credentials for Google Docs.  
    - Inputs: Receives file metadata from "Get files from folder".  
    - Outputs: Emits the content of each Google Docs file in the `content` property.  
    - Edge Cases:  
      - File is not a Google Doc → node may fail or return empty content.  
      - Permission denied or expired token → authentication errors.  
      - Large documents → possible timeout or partial content.  
    - Version: 2  

  - **Sticky Note1 (Step 2)**  
    - Content: "## Step2\nIf you have files other than google docs change node here."  
    - Role: Instructional note indicating where to modify the workflow to support other file types.  

---

#### 1.4 Mapping

- **Overview:**  
  Maps each file name to its corresponding content, creating individual key-value pairs for each document.

- **Nodes Involved:**  
  - Mapping  
  - Sticky Note2 (Mapping)

- **Node Details:**  

  - **Mapping**  
    - Type: Set node  
    - Role: Creates an object with the file name as the key and the document content as the value.  
    - Configuration:  
      - Assignments:  
        - Key: dynamically set to the file name from "Get files from folder" (`={{ $('Get files from folder').item.json.name }}`)  
        - Value: set to the content from "Download Google Docs" (`={{ $json.content }}`)  
    - Inputs: Receives document content from "Download Google Docs".  
    - Outputs: Emits a JSON object with a single key-value pair per item.  
    - Edge Cases:  
      - Duplicate file names → keys will overwrite in aggregation step.  
      - Missing content → value may be empty or undefined.  
    - Version: 3.4  

  - **Sticky Note2 (Mapping)**  
    - Content: "## Mapping\nThis mapping part will output a dictionary with key:value where key if file name and value is file content"  
    - Role: Clarifies the purpose of the mapping node.  

---

#### 1.5 Aggregation

- **Overview:**  
  Aggregates all individual key-value pairs into a single dictionary object, normalizing newlines in the content.

- **Nodes Involved:**  
  - Code

- **Node Details:**  

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Combines all incoming items into one JSON object where each key is a file name and each value is the normalized content string.  
    - Configuration:  
      - JavaScript code loops through all input items, validates each item’s JSON, and merges key-value pairs into a single object.  
      - Normalizes newlines by replacing multiple consecutive newlines with a single newline (`replaceAll(/\n+/g, "\n")`).  
      - Returns an array with one item containing the aggregated dictionary.  
    - Inputs: Receives multiple items from the Mapping node.  
    - Outputs: Single item with aggregated dictionary in `json` property.  
    - Edge Cases:  
      - Input items with invalid JSON structure are skipped with a warning logged.  
      - Duplicate keys overwrite previous values (last one wins).  
      - Large number of files → potential performance impact.  
    - Version: 2  

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                              | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                   |
|----------------------------|------------------------------|----------------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger      | Entry trigger to start workflow               | None                         | Get files from folder        |                                                                                              |
| Get files from folder       | Google Drive                 | Retrieves all files from specified folder     | When Executed by Another Workflow | Download Google Docs         | ## Step1<br>Define folder you want to search all files in.                                  |
| Download Google Docs        | Google Docs                  | Downloads content of each Google Docs file    | Get files from folder         | Mapping                     | ## Step2<br>If you have files other than google docs change node here.                      |
| Mapping                    | Set                         | Maps file name to document content            | Download Google Docs          | Code                        | ## Mapping<br>This mapping part will output a dictionary with key:value where key is file name and value is file content |
| Code                       | Code                        | Aggregates all mappings into a single dictionary with normalized newlines | Mapping                      | None                        |                                                                                              |
| Sticky Note                | Sticky Note                 | Instructional note for folder selection        | None                         | None                        | ## Step1<br>Define folder you want to search all files in.                                  |
| Sticky Note1               | Sticky Note                 | Instructional note for file type customization | None                         | None                        | ## Step2<br>If you have files other than google docs change node here.                      |
| Sticky Note2               | Sticky Note                 | Instructional note explaining mapping purpose  | None                         | None                        | ## Mapping<br>This mapping part will output a dictionary with key:value where key is file name and value is file content |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - No special configuration needed; leave default JSON example input `{}`.

2. **Create Google Drive Node to List Files**  
   - Add a **Google Drive** node named `Get files from folder`.  
   - Set **Resource** to `fileFolder`.  
   - Under **Filter**, set `folderId` mode to `list` and value to the target folder ID (default is `"root"`).  
   - Configure Google Drive OAuth2 credentials with access to the target folder.  
   - Connect output of `When Executed by Another Workflow` to this node.

3. **Create Google Docs Node to Download Content**  
   - Add a **Google Docs** node named `Download Google Docs`.  
   - Set **Operation** to `get`.  
   - Set **Document URL** to expression: `={{ $json.id }}` to dynamically use file IDs from previous node.  
   - Configure Google Docs OAuth2 credentials with access to the documents.  
   - Connect output of `Get files from folder` to this node.

4. **Create Set Node for Mapping**  
   - Add a **Set** node named `Mapping`.  
   - Add an assignment:  
     - Name (key): Expression `={{ $('Get files from folder').item.json.name }}` to get file name.  
     - Value: Expression `={{ $json.content }}` to get document content.  
   - Connect output of `Download Google Docs` to this node.

5. **Create Code Node for Aggregation**  
   - Add a **Code** node named `Code`.  
   - Paste the following JavaScript code:

   ```javascript
   const aggregatedDict = {};
   const inputItems = $input.all();

   for (const item of inputItems) {
     const itemJson = item.json;
     if (itemJson && typeof itemJson === 'object' && !Array.isArray(itemJson)) {
       for (const key of Object.keys(itemJson)) {
         aggregatedDict[key] = itemJson[key].replaceAll(/\n+/g, "\n");
       }
     } else {
       console.warn(`Skipping item - 'json' property is not a valid object:`, itemJson);
     }
   }

   return [{ json: aggregatedDict }];
   ```

   - Connect output of `Mapping` to this node.

6. **Add Sticky Notes (Optional for Documentation)**  
   - Add three **Sticky Note** nodes with the following content and place them near relevant nodes for clarity:  
     - Near `Get files from folder`: "## Step1\nDefine folder you want to search all files in."  
     - Near `Download Google Docs`: "## Step2\nIf you have files other than google docs change node here."  
     - Near `Mapping`: "## Mapping\nThis mapping part will output a dictionary with key:value where key is file name and value is file content."

7. **Save and Test**  
   - Save the workflow.  
   - Trigger the workflow via an external workflow or manual trigger to verify it correctly outputs a JSON object mapping file names to normalized content.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is ideal for dynamically loading prompt templates or knowledge bases from Google Docs. | Use case described in the workflow overview.                                                       |
| To support other file types (e.g., PDFs or text files), replace the Google Docs node with appropriate nodes. | See Sticky Note1 near the Google Docs node.                                                        |
| OAuth2 credentials for both Google Drive and Google Docs must be configured and authorized beforehand. | Pre-requisites section in the description.                                                         |
| Normalization of newlines in the Code node ensures consistent formatting for downstream AI or database ingestion. | Code node JavaScript comments.                                                                     |
| For more advanced use, consider integrating this workflow with AI nodes like OpenAI or Hugging Face for prompt management or RAG systems. | Additional use cases section in the description.                                                   |
| n8n documentation on Google Drive and Google Docs nodes: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googledrive/ and https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googledocs/ | Official n8n docs for node configuration details.                                                  |

---

This structured reference document provides a detailed understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot it effectively.