Convert Make.com blueprints to workflows with Azure OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/convert-make-com-blueprints-to-workflows-with-azure-openai-and-google-sheets-13794


# Convert Make.com blueprints to workflows with Azure OpenAI and Google Sheets

# Workflow Reference: AI-Powered Make.com to n8n Migration Engine

This document provides a technical breakdown of the n8n workflow designed to automate the conversion of Make.com blueprint JSON files into valid, importable n8n workflow exports using Azure OpenAI.

---

### 1. Workflow Overview

The purpose of this workflow is to bridge the gap between Make.com and n8n by programmatically remapping logic, connections, and node configurations. It eliminates manual migration effort by using an AI agent trained on n8n's schema requirements.

**Logical Blocks:**
*   **1.1 Trigger & Discovery:** Manual initiation followed by scanning Google Drive for target blueprint files.
*   **1.2 Data Preparation:** Downloading the binary file and extracting the JSON content.
*   **1.3 AI Intelligence:** An AI Agent (GPT-4o-mini) processes the Make.com JSON and outputs a structured n8n JSON.
*   **1.4 Post-Processing:** Refinement of the AI output to ensure strict schema compliance (fixing connections and conditions).
*   **1.5 Export:** Saving the finalized JSON to a Google Sheet for review or programmatic retrieval.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & File Discovery
*   **Overview:** Locates the source file within a cloud storage environment.
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Search Files and Folders`, `Filter Target Blueprint`.
*   **Node Details:**
    *   **Search Files and Folders (Google Drive):** Configured to list files. It requires Google Drive OAuth2 credentials.
    *   **Filter Target Blueprint (IF Node):** Checks if the filename contains the string "blueprint".
    *   **Edge Cases:** If multiple files contain "blueprint", the workflow proceeds with all of them, potentially triggering multiple AI runs.

#### 2.2 Download & Parse
*   **Overview:** Converts a cloud file reference into usable JSON data.
*   **Nodes Involved:** `Download Blueprint File`, `Extract JSON from File`.
*   **Node Details:**
    *   **Download Blueprint File (Google Drive):** Downloads the file using the `id` from the previous step. Output is a binary object.
    *   **Extract JSON from File:** Converts the binary file content into a standard n8n JSON object so the AI can read it.

#### 2.3 AI Conversion
*   **Overview:** The "brain" of the workflow that performs the technical mapping.
*   **Nodes Involved:** `Convert Blueprint with AI Agent`, `Azure OpenAI Chat Model`.
*   **Node Details:**
    *   **Convert Blueprint with AI Agent:** Uses a detailed system prompt to enforce n8n schema rules (e.g., forcing `main` connection types, mapping `google-sheets:watchRows` to `googleSheetsTrigger`).
    *   **Azure OpenAI Chat Model:** Configured with `gpt-4o-mini`. Requires a scoped Azure deployment and API key.
    *   **Key Expression:** `{{ JSON.stringify($json.data) }}` sends the raw Make.com data to the LLM.

#### 2.4 Cleanup & Transform
*   **Overview:** Programmatic "sanitization" of the AI's output to ensure 100% import compatibility.
*   **Nodes Involved:** `Normalize Connections and Conditions`, `Strip Make Fields and Finalize`.
*   **Node Details:**
    *   **Normalize Connections and Conditions (Code):** Maps Node IDs back to Node Names (as n8n requires names for connections) and fixes "IF" node structures that the AI might format incorrectly.
    *   **Strip Make Fields and Finalize (Code):** Removes Make-specific metadata (like `__IMTCONN__`) and strips `typeVersion` and `versionId` to prevent version mismatch errors during import.

#### 2.5 Export
*   **Overview:** Stores the result in an external database/sheet.
*   **Nodes Involved:** `Save Converted Workflow to Sheet`.
*   **Node Details:**
    *   **Google Sheets:** Appends the final JSON string into a designated column. 
    *   **Requirements:** Requires a pre-existing Spreadsheet ID and a sheet named "json".

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | manualTrigger | Workflow Entry | (None) | Search Files and Folders | Starts on manual execution... |
| Search Files and Folders | googleDrive | File Discovery | Manual Trigger | Filter Target Blueprint | Starts on manual execution... |
| Filter Target Blueprint | if | File Validation | Search Files | Download Blueprint File | Starts on manual execution... |
| Download Blueprint File | googleDrive | Data Retrieval | Filter | Extract JSON from File | Downloads the matched blueprint... |
| Extract JSON from File | extractFromFile | Data Parsing | Download | AI Agent | Downloads the matched blueprint... |
| Convert Blueprint with AI Agent | agent | AI Translation | Extract JSON | Normalize Connections | The Azure OpenAI-powered agent... |
| Azure OpenAI Chat Model | lmChatAzureOpenAi | LLM Provider | (None) | AI Agent | (None) |
| Normalize Connections and Conditions | code | Data Refinement | AI Agent | Strip Make Fields | Two code nodes clean up... |
| Strip Make Fields and Finalize | code | Data Finalization | Normalize | Save to Sheet | Two code nodes clean up... |
| Save Converted Workflow to Sheet | googleSheets | Data Storage | Strip Make Fields | (None) | (None) |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Set up **Google Drive & Sheets OAuth2** credentials.
    *   Set up **Azure OpenAI** credentials and deploy a `gpt-4o-mini` or `gpt-4o` model.
2.  **File Input Stage:**
    *   Add a **Manual Trigger**.
    *   Add a **Google Drive Node** (Search), set Resource to "File", and Operation to "List".
    *   Add an **IF Node** to filter files where the name contains "blueprint".
    *   Add another **Google Drive Node** (Download) using the `id` from the search.
    *   Add an **Extract from File Node**, setting the operation to "JSON".
3.  **AI Integration:**
    *   Add an **AI Agent Node**. Connect an **Azure OpenAI Chat Model** node to its tool/model input.
    *   **Prompt Configuration:** Paste a system message instructing the model to output *only* valid n8n JSON, mapping modules like `builtin:BasicRouter` to `n8n-nodes-base.if`.
4.  **The Logic Fix (Code Node 1):**
    *   Create a **Code Node** named "Normalize Connections".
    *   Write a script to iterate through the `connections` object, replacing ID keys with Name keys.
    *   Ensure "IF" node conditions use the `string` or `boolean` arrays correctly.
5.  **The Cleanup (Code Node 2):**
    *   Create a second **Code Node**.
    *   Write a script to delete `parameters.__IMTCONN__` from all nodes and remove top-level fields like `active` and `versionId`.
6.  **Final Export:**
    *   Add a **Google Sheets Node** (Append).
    *   Map the `json` column to the output of the final Code node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure the target Google Sheet has a header named "json" in the first row. | Setup Requirement |
| Use n8n version 1.0+ for compatibility with the AI Agent node. | Version Note |
| Azure OpenAI deployment name must match the model name used in the credentials. | Credential Setup |
| For large blueprints, increase the timeout in the AI Agent node settings. | Performance |