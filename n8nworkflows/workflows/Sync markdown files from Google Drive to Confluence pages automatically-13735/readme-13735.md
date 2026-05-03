Sync markdown files from Google Drive to Confluence pages automatically

https://n8nworkflows.xyz/workflows/sync-markdown-files-from-google-drive-to-confluence-pages-automatically-13735


# Sync markdown files from Google Drive to Confluence pages automatically

# Markdown to Confluence Sync Workflow

This reference document describes an automated n8n workflow designed to synchronize Markdown files from a specific Google Drive folder to Atlassian Confluence. The workflow monitors for both new file creations and updates to existing files, converting Markdown content into Confluence-compatible HTML (Storage Format) automatically.

---

### 1. Workflow Overview

The workflow serves as a bridge between Google Drive and Confluence, ensuring that documentation managed in Markdown remains up-to-date in a corporate wiki environment. It operates on a polling basis (every hour) and handles the lifecycle of a document from initial creation to versioned updates.

**Logical Blocks:**
- **1.1 Change Detection:** Monitors Google Drive for new or modified files.
- **1.2 Validation & Extraction:** Filters for `.md` extensions and downloads the binary content.
- **1.3 Content Transformation:** Converts the raw file into a text string and then transforms Markdown syntax into HTML.
- **1.4 Routing & Integration:** Determines if the action is a "Create" or "Update" and performs the corresponding REST API call to Confluence.

---

### 2. Block-by-Block Analysis

#### 2.1 Change Detection & Validation
Detects changes in Google Drive and ensures only Markdown files proceed.
- **Nodes Involved:** `File Created`, `File Updated`, `If file ends in .md`.
- **Node Details:**
    - **Google Drive Triggers:** Configured to poll a specific folder every hour.
    - **Validation (If Node):** Uses a string comparison (`endsWith`) on `originalFilename` or `name` to ensure the file is a `.md` file.
    - **Failure cases:** Incorrect folder ID, expired Google OAuth credentials, or non-markdown files (which are correctly ignored by the `False` output).

#### 2.2 Content Transformation
Fetches the file content and prepares it for Confluence.
- **Nodes Involved:** `Download file`, `Convert file to markdown string`, `Markdown`.
- **Node Details:**
    - **Download file (Google Drive):** Downloads the file using the ID from the trigger. It includes a conversion option for Google Docs to Markdown if applicable.
    - **Convert file to markdown string (Extract from File):** Converts the binary buffer into a readable UTF-8 string.
    - **Markdown:** Uses the `markdownToHtml` mode to transform the file content into the HTML format required by the Confluence "Storage" representation.

#### 2.3 Routing Logic
Directs the data based on whether the file is new or updated.
- **Nodes Involved:** `If Trigger was File Created`.
- **Node Details:**
    - **Logic:** Checks the execution status of the `File Created` node. If `true`, it routes to the creation path; otherwise, it defaults to the update path (triggered by `File Updated`).

#### 2.4 Confluence Integration
Executes the final API requests to Atlassian.
- **Nodes Involved:** `Create Confluence Page`, `Get Page ID`, `Update Confluence Page`.
- **Node Details:**
    - **Get Page ID (HTTP Request):** For updates, it searches for the existing page ID using the filename as the title.
    - **Create/Update (HTTP Request):** 
        - Uses the `PUT` method (for create/update).
        - **Body:** Sends a JSON payload containing the `spaceId`, `title`, and `body` (type: storage).
        - **Versioning:** For updates, it increments the version number based on the current version fetched in the previous step.
    - **Authentication:** Requires Header Auth (Generic Credential) with a Confluence API Token.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| File Created | Google Drive Trigger | Entry Point (New) | None | If file ends in .md | This workflow monitors a folder in a Google Drive for changes every hour... |
| File Updated | Google Drive Trigger | Entry Point (Edit) | None | If file ends in .md | This workflow monitors a folder in a Google Drive for changes every hour... |
| If file ends in .md | If | Validation | File Created, File Updated | Download file | This workflow monitors a folder in a Google Drive for changes every hour... |
| Download file | Google Drive | Data Fetching | If file ends in .md | Convert file to markdown string | ... |
| Convert file to markdown string | Extract From File | File Processing | Download file | Markdown | ... |
| Markdown | Markdown | Conversion | Convert file to markdown string | If Trigger was File Created | ... |
| If Trigger was File Created | If | Logic Routing | Markdown | Create Confluence Page, Get Page ID | ... |
| Get Page ID | HTTP Request | Metadata Lookup | If Trigger was File Created | Update Confluence Page | Setup: Add your Confluence Cloud ID to the URL... |
| Create Confluence Page | HTTP Request | Integration (Write) | If Trigger was File Created | None | Setup: Create a scoped API token for Confluence... |
| Update Confluence Page | HTTP Request | Integration (Write) | Get Page ID | None | Setup: Add your Confluence Cloud ID to the URL... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Google Drive Setup:**
    *   Create two **Google Drive Trigger** nodes.
    *   Set Trigger 1 to `File Created` and Trigger 2 to `File Updated`.
    *   Select "Specific Folder" and point to your desired Markdown repository. Set polling to "Every Hour".
2.  **Validation:**
    *   Add an **If** node. Check if `name` or `originalFilename` ends with `.md`.
3.  **Data Retrieval:**
    *   Add a **Google Drive** node, set to `Download`. Map the File ID from the trigger.
    *   Add an **Extract From File** node. Set the operation to `text` to get the Markdown string.
4.  **Conversion:**
    *   Add a **Markdown** node. Set the mode to `Markdown to HTML`. Map the input from the previous extraction.
5.  **Routing:**
    *   Add an **If** node. Use the expression `{{ $('File Created').isExecuted }}` to check the source.
6.  **Confluence Integration (Create):**
    *   On the **True** path, add an **HTTP Request** node.
    *   **Method:** `PUT`. **URL:** `https://api.atlassian.com/ex/confluence/<CLOUD_ID>/wiki/api/v2/pages/`.
    *   **Body:** Provide a JSON structure including `spaceId`, `title`, and `body` (value: `{{ JSON.stringify($json.data) }}`).
7.  **Confluence Integration (Update):**
    *   On the **False** path, add an **HTTP Request** node ("Get Page ID").
    *   **URL:** Search by title: `.../wiki/api/v2/pages?title={{ $node["File Updated"].json["name"] }}`.
    *   Follow this with another **HTTP Request** node ("Update Confluence Page").
    *   **Method:** `PUT`. **URL:** Use the ID from the search result.
    *   **Body:** Increment the version number: `{{ $json.results[0].version.number + 1 }}`.
8.  **Credentials:**
    *   Configure **Google OAuth2** for the triggers/download.
    *   Configure **Header Auth** for Confluence (Key: `Authorization`, Value: `Bearer <YOUR_TOKEN>`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Confluence Scopes Required** | Needs `write:confluence-content` and `write:content:confluence`. |
| **Cloud ID Requirement** | Users must replace `<CLOUD_ID>` in all HTTP URLs with their actual Atlassian Cloud ID. |
| **Storage Format** | Confluence uses a specific HTML "Storage Format"; the Markdown node output is compatible with this. |