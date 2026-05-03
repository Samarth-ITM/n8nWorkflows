Automated Content Migration from ClickUp Docs to Airtable Records

https://n8nworkflows.xyz/workflows/automated-content-migration-from-clickup-docs-to-airtable-records-9135


# Automated Content Migration from ClickUp Docs to Airtable Records

---

### 1. Workflow Overview

This workflow automates the migration of content from ClickUp Docs into Airtable records. It is triggered by the creation of a new task in ClickUp that includes a URL linking to a ClickUp Doc. The workflow extracts the Doc’s content, parses it into structured data segments, and then creates corresponding records in a specified Airtable base and table, linking them to related "Vertical" records.

**Target Use Cases:**
- Content creators or managers who maintain knowledge bases in ClickUp Docs but want structured tracking in Airtable.
- Teams needing automated synchronization from unstructured task/doc content into a database format.
- Operations teams managing content workflows between ClickUp and Airtable.

**Logical Blocks:**

- **1.1 Trigger and Task Retrieval:** Detect new ClickUp tasks and retrieve detailed task info.
- **1.2 Doc URL Extraction and Content Fetching:** Extract Doc and Workspace IDs from task names, then retrieve full Doc details and pages.
- **1.3 Content Parsing:** Process raw page content into structured segments with fields like content and notes.
- **1.4 Airtable Base and Table Identification:** Match parsed data to the correct Airtable base and tables by name.
- **1.5 Airtable Record Search and Creation:** Find related "Vertical" records and create new Airtable records for each parsed content segment.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Task Retrieval

**Overview:**  
This block initiates the workflow when a new task is created in ClickUp within a specified team. It then fetches detailed information about that task.

**Nodes Involved:**  
- Trigger on New Clickup Task  
- Get Task Details

**Node Details:**

- **Trigger on New Clickup Task**  
  - *Type:* ClickUp Trigger  
  - *Role:* Starts workflow on "taskCreated" event in a specific ClickUp team  
  - *Configuration:* Listens for new task creation events in team ID "9014329600" using OAuth2 authentication  
  - *Input/Output:* No input; outputs the new task’s basic info including task_id  
  - *Edge Cases:* Missed events if webhook fails; auth token expiration; event filter misconfiguration  

- **Get Task Details**  
  - *Type:* ClickUp Node  
  - *Role:* Retrieves full task details using the task_id from the trigger  
  - *Configuration:* Operation "get" with id taken from trigger output expression `{{$json.task_id}}`  
  - *Input/Output:* Input from trigger; outputs detailed task JSON including task name  
  - *Edge Cases:* API rate limits; task deletion between trigger and fetch; OAuth token issues  

---

#### 2.2 Doc URL Extraction and Content Fetching

**Overview:**  
Extracts ClickUp Doc and Workspace IDs from the task name URL and fetches the Doc’s metadata and all its pages.

**Nodes Involved:**  
- Extract Doc & Workspace IDs from URL  
- Get ClickUp Doc Details  
- Fetch All Pages in Doc

**Node Details:**

- **Extract Doc & Workspace IDs from URL**  
  - *Type:* Code  
  - *Role:* Parses the task name to extract workspaceId and docId via regex matching the ClickUp Doc URL format  
  - *Key Expression:* Regex `/clickup\.com\/(\d+)\/v\/dc\/([^/]+)/` applied to task name string  
  - *Input/Output:* Input from Get Task Details; outputs JSON with extracted workspaceId and docId  
  - *Edge Cases:* Task name missing URL or malformed; multiple URLs in name; no match found returns empty  

- **Get ClickUp Doc Details**  
  - *Type:* HTTP Request  
  - *Role:* Fetches metadata for the ClickUp Doc using workspaceId and docId  
  - *Configuration:* GET request to `https://api.clickup.com/api/v3/workspaces/{workspaceId}/docs/{docId}`  
  - *Auth:* Uses OAuth2 ClickUp credential  
  - *Input/Output:* Input from code node; outputs Doc metadata JSON  
  - *Edge Cases:* API errors; invalid IDs; network timeouts; token expiration  

- **Fetch All Pages in Doc**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves all pages within the specified Doc, including nested pages  
  - *Configuration:* GET request to `https://api.clickup.com/api/v3/workspaces/{workspace_id}/docs/{id}/pages` with query param `max_page_depth = -2` (all levels)  
  - *Auth:* OAuth2 ClickUp credential  
  - *Input/Output:* Input from Doc details node; outputs array of pages with content fields  
  - *Edge Cases:* Large docs may cause timeouts; paging issues; API limits; missing page content fields  

---

#### 2.3 Content Parsing

**Overview:**  
Processes the raw content of each Doc page to separate it into logical content pieces and notes, splitting by the delimiter `* * *` and the keyword "notes:" to structure data for Airtable.

**Nodes Involved:**  
- Parse Content from Doc Pages  
- Loop Over Pages

**Node Details:**

- **Parse Content from Doc Pages**  
  - *Type:* Code  
  - *Role:* Iterates over pages, splits content on `* * *`, separates notes by detecting "notes:" keyword, removes asterisks, and prepares JSON objects with fields `base`, `table`, `name`, `content`, and `notes`  
  - *Key Logic:*  
    - Split content by delimiter `* * *` if present  
    - Separate main content and notes by regex on "notes:" ignoring case  
    - Trim and clean up asterisks in content  
  - *Input/Output:* Input from Fetch All Pages; outputs array of structured content pieces  
  - *Edge Cases:* Pages with empty content; missing "notes:" keyword; inconsistent delimiters; malformed text  

- **Loop Over Pages**  
  - *Type:* SplitInBatches  
  - *Role:* Processes each parsed content piece individually to enable sequential downstream processing  
  - *Input/Output:* Input from Parse Content; outputs single items for Airtable processing  
  - *Edge Cases:* Batch size too large causing memory issues; empty input array  

---

#### 2.4 Airtable Base and Table Identification

**Overview:**  
Matches the base and table names extracted from the Doc metadata against existing Airtable bases and tables to identify the correct destination for record creation.

**Nodes Involved:**  
- Get All Base  
- Match Airtable Base by Name  
- Get All Tables in Selected Base  
- Match Airtable Table by Name

**Node Details:**

- **Get All Base**  
  - *Type:* Airtable  
  - *Role:* Fetches all Airtable bases available to the connected account  
  - *Credentials:* Airtable Personal Access Token  
  - *Input/Output:* No input; outputs list of bases  
  - *Edge Cases:* API rate limits; auth errors; empty base list  

- **Match Airtable Base by Name**  
  - *Type:* Code  
  - *Role:* Filters the list of Airtable bases to find one matching the base name parsed from ClickUp Doc (case insensitive, trims " (new)" suffix)  
  - *Input/Output:* Input from Get All Base and Loop Over Pages (for base name); outputs matching base JSON  
  - *Edge Cases:* No matching base found; multiple matches  

- **Get All Tables in Selected Base**  
  - *Type:* Airtable  
  - *Role:* Retrieves schema (tables) for the matched Airtable base  
  - *Credentials:* Airtable Personal Access Token  
  - *Input/Output:* Input from Match Airtable Base; outputs list of tables  
  - *Edge Cases:* API errors; empty tables array; base permission issues  

- **Match Airtable Table by Name**  
  - *Type:* Code  
  - *Role:* Filters tables to find one matching the table name parsed from ClickUp Doc (case insensitive)  
  - *Input/Output:* Input from Get All Tables and Loop Over Pages (table name); outputs matching table JSON  
  - *Edge Cases:* No matching table; multiple matches; name case inconsistencies  

---

#### 2.5 Airtable Record Search and Creation

**Overview:**  
Locates a related "Vertical" record in Airtable matching the name from the parsed content, then creates a new record in the target table with fields populated from the parsed content.

**Nodes Involved:**  
- Find Corresponding "Vertical" Record  
- Create New Record in Airtable

**Node Details:**

- **Find Corresponding "Vertical" Record**  
  - *Type:* Airtable  
  - *Role:* Searches a specified table in the matched base for a record whose `Name` field matches the current content name  
  - *Filter Formula:* `={Name} = '{{ Loop Over Pages item.json.name }}'`  
  - *Credentials:* Airtable Personal Access Token  
  - *Input/Output:* Input from Match Airtable Table; outputs found record(s)  
  - *Edge Cases:* No matching record found; multiple matches; formula syntax errors  

- **Create New Record in Airtable**  
  - *Type:* Airtable  
  - *Role:* Creates a new record in the matched Airtable base and table with fields mapped from parsed content and related "Vertical" record  
  - *Fields Mapped:*  
    - Text: content from ClickUp page  
    - Notes: notes extracted from content  
    - Status: hardcoded to "Testing"  
    - Vertical: linked record(s) from found "Vertical" record’s Name field  
    - Created Date: current timestamp at execution  
  - *Options:* Typecast enabled to convert field types automatically  
  - *Credentials:* Airtable Personal Access Token  
  - *Input/Output:* Input from Find "Vertical" record; outputs created record details  
  - *Edge Cases:* Invalid field mappings; missing required Airtable fields; API rate limits; typecast failures  

---

### 3. Summary Table

| Node Name                        | Node Type              | Functional Role                                  | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                                                                                                                                                                                             |
|---------------------------------|------------------------|-------------------------------------------------|------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger on New Clickup Task      | ClickUp Trigger        | Starts workflow on new task creation             |                              | Get Task Details                 | ## Fetch Clickup Task, Doc, and Page Content                                                                                                                                                                                                                            |
| Get Task Details                | ClickUp Node           | Retrieves full task details                        | Trigger on New Clickup Task   | Extract Doc & Workspace IDs from URL | ## Fetch Clickup Task, Doc, and Page Content                                                                                                                                                                                                                            |
| Extract Doc & Workspace IDs from URL | Code                 | Extracts workspaceId and docId from task name URL | Get Task Details             | Get ClickUp Doc Details          | ## Fetch Clickup Task, Doc, and Page Content                                                                                                                                                                                                                            |
| Get ClickUp Doc Details          | HTTP Request           | Fetches Doc metadata                              | Extract Doc & Workspace IDs  | Fetch All Pages in Doc           | ## Fetch Clickup Task, Doc, and Page Content                                                                                                                                                                                                                            |
| Fetch All Pages in Doc           | HTTP Request           | Retrieves all pages in the Doc                    | Get ClickUp Doc Details       | Parse Content from Doc Pages     | ## Fetch Clickup Task, Doc, and Page Content                                                                                                                                                                                                                            |
| Parse Content from Doc Pages     | Code                   | Parses page content into structured segments      | Fetch All Pages in Doc        | Loop Over Pages                 | ## Move Everything to Airtable                                                                                                                                                                                                                                          |
| Loop Over Pages                 | SplitInBatches         | Processes content segments one at a time          | Parse Content from Doc Pages  | Get All Base; Create New Record in Airtable | ## Move Everything to Airtable                                                                                                                                                                                                                                          |
| Get All Base                    | Airtable               | Retrieves all Airtable bases                       | Loop Over Pages (2nd output)  | Match Airtable Base by Name      | ## Move Everything to Airtable                                                                                                                                                                                                                                          |
| Match Airtable Base by Name      | Code                   | Filters Airtable bases by name                     | Get All Base                 | Get All Tables in Selected Base  | ## Move Everything to Airtable                                                                                                                                                                                                                                          |
| Get All Tables in Selected Base  | Airtable               | Retrieves tables for matched Airtable base        | Match Airtable Base by Name  | Match Airtable Table by Name     | ## Move Everything to Airtable                                                                                                                                                                                                                                          |
| Match Airtable Table by Name     | Code                   | Filters tables by name                             | Get All Tables in Selected Base | Find Corresponding "Vertical" Record | ## Move Everything to Airtable                                                                                                                                                                                                                                          |
| Find Corresponding "Vertical" Record | Airtable           | Searches for related "Vertical" record             | Match Airtable Table by Name | Create New Record in Airtable    | ## Move Everything to Airtable                                                                                                                                                                                                                                          |
| Create New Record in Airtable    | Airtable               | Creates new record with parsed content             | Find Corresponding "Vertical" Record | Loop Over Pages (2nd output)     | ## Move Everything to Airtable                                                                                                                                                                                                                                          |
| Sticky Note                    | Sticky Note            | Workflow overview and detailed instructions       |                              |                                 | ## Create Airtable records from new ClickUp Doc pages  This workflow automates the process of turning content from ClickUp Docs into structured data in Airtable. When a new task is created in ClickUp with a link to a ClickUp Doc in its name, this workflow triggers, fetches the entire content of that Doc, parses it into individual records, and then creates a new record for each item in a specified Airtable base and table.  ## Who's it for  This template is perfect for content creators, project managers, and operations teams who use ClickUp Docs for drafting or knowledge management and Airtable for tracking and organizing data. It helps bridge the gap between unstructured text and a structured database.  ## How it works  1.  **Trigger:** The workflow starts when a new task is created in a specific ClickUp Team.  2.  **Fetch & Parse URL:** It gets the new task's details and extracts the ClickUp Doc URL from the task name.  3.  **Get Doc Content:** It uses the URL to fetch the main Doc and all its sub-pages from the ClickUp API.  4.  **Process Content:** A Code node parses the text from each page. It's designed to split content by `* * *` and separate notes by looking for the "notes:" keyword.  5.  **Find Airtable Destination:** The workflow finds the correct Airtable Base and Table IDs by matching the names you provide.  6.  **Create Records:** It loops through each parsed content piece and creates a new record in your specified Airtable table.  ## How to set up  1.  **Configure the `Set` Node:** Open the "Configure Variables" node and set the following values:  * `clickupTeamId`: Your ClickUp Team ID. Find it in your ClickUp URL (e.g., `app.clickup.com/9014329600/...`).  * `airtableBaseName`: The exact name of your target Airtable Base.  * `airtableTableName`: The exact name of your target Airtable Table.  * `airtableVerticalsTableName`: The name of the table in your base that holds "Vertical" records, which are linked in the main table.  2.  **Set Up Credentials:** Add your ClickUp (OAuth2) and Airtable (Personal Access Token) credentials to the respective nodes.  3.  **Airtable Fields:** Ensure your Airtable table has fields corresponding to the ones in the `Create New Record in Airtable` node (e.g., `Text`, `Status`, `Vertical`, `Notes`). You can customize the mapping in this node.  4.  **Activate Workflow:** Save and activate the workflow.  5.  **Test:** Create a new task in your designated ClickUp team. In the task name, include the full URL of the ClickUp Doc you want to process.  ## How to customize the workflow  * **Parsing Logic:** You can change how the content is parsed by modifying the JavaScript in the `Parse Content from Doc Pages` Code node. For example, you could change the delimiter from `* * *` to something else.  * **Field Mapping:** Adjust the `Create New Record in Airtable` node to map data to different fields or add more fields from the source data.  * **Trigger Events:** Modify the `Trigger on New ClickUp Task` node to respond to different events, such as `taskUpdated` or `taskCommentPosted`. |
| Sticky Note1                   | Sticky Note            | Section label for ClickUp fetch and parse steps   |                              |                                 | ## Fetch Clickup Task, Doc, and Page Content                                                                                                                                                                                                                            |
| Sticky Note2                   | Sticky Note            | Section label for Airtable data insertion steps   |                              |                                 | ## Move Everything to Airtable                                                                                                                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "Trigger on New Clickup Task"**  
   - Type: ClickUp Trigger  
   - Event: `taskCreated`  
   - Team ID: `9014329600` (replace with your own)  
   - Authentication: OAuth2 (configure ClickUp OAuth2 credentials)  
   - Save and activate webhook  

2. **Add Node: "Get Task Details"**  
   - Type: ClickUp  
   - Operation: Get task  
   - Task ID: Use expression `{{$json.task_id}}` from trigger output  
   - Authentication: OAuth2 (same as above)  

3. **Add Node: "Extract Doc & Workspace IDs from URL"**  
   - Type: Code  
   - Language: JavaScript  
   - Paste code to parse `workspaceId` and `docId` from task name URL using regex `/clickup\.com\/(\d+)\/v\/dc\/([^/]+)/`  
   - Input: from "Get Task Details" node  

4. **Add Node: "Get ClickUp Doc Details"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.clickup.com/api/v3/workspaces/{{$json.workspaceId}}/docs/{{$json.docId}}`  
   - Query Parameters: `max_page_depth=-2`  
   - Headers: `accept: application/json`  
   - Authentication: Predefined credential type with ClickUp OAuth2 credentials  

5. **Add Node: "Fetch All Pages in Doc"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.clickup.com/api/v3/workspaces/{{$json.workspace_id}}/docs/{{$json.id}}/pages`  
   - Query Parameters: `max_page_depth=-2`  
   - Headers: `accept: application/json`  
   - Authentication: Predefined ClickUp OAuth2 credentials  

6. **Add Node: "Parse Content from Doc Pages"**  
   - Type: Code  
   - JavaScript to iterate pages: split content by `* * *`, separate notes by "notes:" keyword, remove asterisks, trim results  
   - Input: from "Fetch All Pages in Doc" node  

7. **Add Node: "Loop Over Pages"**  
   - Type: SplitInBatches  
   - Configuration: default batch size (can be adjusted for performance)  
   - Input: from "Parse Content from Doc Pages"  

8. **Add Node: "Get All Base"**  
   - Type: Airtable  
   - Operation: List bases  
   - Credentials: Airtable Personal Access Token  
   - Input: second output from "Loop Over Pages" (parallel branch)  

9. **Add Node: "Match Airtable Base by Name"**  
   - Type: Code  
   - Filter Airtable bases by comparing base name (case insensitive, trimming " (new)") against the base name from parsed content  
   - Input: from "Get All Base" and "Loop Over Pages"  

10. **Add Node: "Get All Tables in Selected Base"**  
    - Type: Airtable  
    - Operation: Get schema (tables) for matched base  
    - Credentials: Airtable Personal Access Token  
    - Input: from "Match Airtable Base by Name"  

11. **Add Node: "Match Airtable Table by Name"**  
    - Type: Code  
    - Filter tables by name matching the parsed table name (case insensitive)  
    - Input: from "Get All Tables in Selected Base" and "Loop Over Pages"  

12. **Add Node: "Find Corresponding \"Vertical\" Record"**  
    - Type: Airtable  
    - Operation: Search records in the matched base and table  
    - Filter formula: `={Name} = '{{ Loop Over Pages item.json.name }}'`  
    - Credentials: Airtable Personal Access Token  
    - Input: from "Match Airtable Table by Name"  

13. **Add Node: "Create New Record in Airtable"**  
    - Type: Airtable  
    - Operation: Create record  
    - Base and Table: Use matched base and table IDs from previous nodes  
    - Fields:  
      - Text: from content field  
      - Notes: from notes field  
      - Status: static "Testing"  
      - Vertical: linked record name from found "Vertical" record  
      - Created Date: current datetime `{{$now}}`  
    - Options: Enable typecast  
    - Credentials: Airtable Personal Access Token  
    - Input: from "Find Corresponding \"Vertical\" Record"  

14. **Connect nodes as per flow:**  
    - Trigger → Get Task Details → Extract Doc & Workspace IDs → Get ClickUp Doc Details → Fetch All Pages in Doc → Parse Content → Loop Over Pages  
    - Loop Over Pages (secondary output) → Get All Base → Match Airtable Base → Get All Tables → Match Airtable Table → Find Corresponding "Vertical" Record → Create New Record  
    - Loop Over Pages (main output) → Create New Record  

15. **Add Sticky Notes** (optional but recommended for documentation):  
    - Add notes summarizing each section of the workflow to aid future maintenance.  

16. **Configure Credentials:**  
    - Set up and test ClickUp OAuth2 credentials  
    - Set up Airtable Personal Access Token credentials with appropriate API scopes  

17. **Activate Workflow and Test:**  
    - Activate the workflow  
    - Create a new ClickUp task in the specified team with a name containing a valid ClickUp Doc URL  
    - Monitor workflow execution and Airtable record creation  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow bridges unstructured ClickUp Docs content with structured Airtable records by automating content parsing and record creation, reducing manual copy-pasting and potential errors. It leverages ClickUp API and Airtable API with OAuth2 and Personal Access Token authentication respectively. The parsing logic is customizable to adapt to different content delimiters or note markers. | Workflow purpose and integration context                                                                 |
| For ClickUp OAuth2 credential setup, refer to ClickUp API documentation: https://clickup.com/api | ClickUp OAuth2 API documentation                                                                         |
| For Airtable API token generation and scopes, see: https://airtable.com/api | Airtable API documentation                                                                                |
| The JavaScript code nodes rely heavily on consistent formatting of ClickUp Doc URLs and content structure; inconsistent or malformed input can cause parsing failures or empty outputs. | Important note on input data quality                                                                      |
| The workflow includes two sticky notes visually dividing the workflow into two functional sections: "Fetch Clickup Task, Doc, and Page Content" and "Move Everything to Airtable" for easier navigation and understanding. | Workflow documentation best practice                                                                     |
| To customize parsing logic or field mappings, edit the respective Code and Airtable nodes respectively. This allows adaptation for different content structures or database schemas without changing the overall workflow structure. | Customization guideline                                                                                   |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---