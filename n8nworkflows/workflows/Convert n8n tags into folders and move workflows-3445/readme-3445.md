Convert n8n tags into folders and move workflows

https://n8nworkflows.xyz/workflows/convert-n8n-tags-into-folders-and-move-workflows-3445


# Convert n8n tags into folders and move workflows

### 1. Workflow Overview

This workflow automates the migration of existing n8n workflows from tag-based organization to the newer folder-based system introduced in n8n. It converts each tag into a folder and moves all workflows associated with that tag into the corresponding folder. For workflows tagged with multiple tags, the workflow is assigned to the folder corresponding to the last processed tag.

The workflow is structured into these logical blocks:

- **1.1 User Input & Authentication:** Receives user credentials and authenticates with the n8n instance.
- **1.2 Project & Tag Retrieval:** Fetches the user's personal projects and associated tags.
- **1.3 Tag Selection & Processing:** Presents tags to the user for selection and processes the selected tags.
- **1.4 Folder Management:** Checks for existing folders matching tag names, creates folders if missing, and normalizes folder names.
- **1.5 Workflow Retrieval & Movement:** Retrieves workflows by tag and moves them into the corresponding folders.
- **1.6 Completion & Feedback:** Provides a success message summarizing the import.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input & Authentication

**Overview:**  
This block collects the n8n instance URL and user credentials, then authenticates the user to obtain session cookies for subsequent API calls.

**Nodes Involved:**  
- On form submission  
- set credentials  
- login n8n

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; triggers workflow on user form submission.  
  - *Config:* Webhook listens for form submission with a "Submit" button and a form titled "Tags to Folders".  
  - *Inputs:* None  
  - *Outputs:* Passes form data to `set credentials`.  
  - *Edge Cases:* Failure if webhook URL is not accessible or form not submitted.

- **set credentials**  
  - *Type:* Set  
  - *Role:* Stores n8n instance URL, username, and password from user input.  
  - *Config:* Assigns three string fields: `n8n` (instance URL without trailing slash), `username`, and `password`.  
  - *Inputs:* From `On form submission`  
  - *Outputs:* Provides credentials to `login n8n`.  
  - *Edge Cases:* User must input valid URL and credentials; no validation here.

- **login n8n**  
  - *Type:* HTTP Request  
  - *Role:* Authenticates user by sending POST to `/rest/login` endpoint.  
  - *Config:*  
    - URL: `={{ $json.n8n }}/rest/login`  
    - Method: POST  
    - Body: Includes `emailOrLdapLoginId` and `password` from `set credentials`.  
    - Headers: Accept JSON, user-agent spoofed.  
    - Response: Full response including headers (to extract cookies).  
  - *Inputs:* From `set credentials`  
  - *Outputs:* Provides session cookie for authenticated requests.  
  - *Edge Cases:* Authentication failure returns error; cookie extraction failure breaks subsequent calls.

---

#### 2.2 Project & Tag Retrieval

**Overview:**  
Fetches the user's personal projects and the tags associated with those projects.

**Nodes Involved:**  
- my-projects  
- Split Out  
- filter owned projects  
- get tags  
- tags to string  
- Code

**Node Details:**

- **my-projects**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves all projects accessible to the user.  
  - *Config:* GET `/rest/projects/my-projects` with session cookie header.  
  - *Inputs:* From `login n8n` (uses cookie)  
  - *Outputs:* JSON list of projects.  
  - *Edge Cases:* API failure, unauthorized access.

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits project list into individual items for filtering.  
  - *Config:* Splits on `data` field.  
  - *Inputs:* From `my-projects`  
  - *Outputs:* Single project per item.  
  - *Edge Cases:* Empty projects list.

- **filter owned projects**  
  - *Type:* Filter  
  - *Role:* Filters projects to those personally owned by the authenticated user.  
  - *Config:*  
    - Checks if project name contains user's email (extracted from username).  
    - Checks if role equals `project:personalOwner`.  
  - *Inputs:* From `Split Out`  
  - *Outputs:* Projects owned by user.  
  - *Edge Cases:* No owned projects found.

- **get tags**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves tags for the filtered owned projects.  
  - *Config:* GET `/rest/tags` with session cookie.  
  - *Inputs:* From `filter owned projects`  
  - *Outputs:* List of tags.  
  - *Edge Cases:* No tags found, API failure.

- **tags to string**  
  - *Type:* Set  
  - *Role:* Extracts tag names into an array for further processing.  
  - *Config:* Maps `body.data` to extract tag `name` fields into `name` array.  
  - *Inputs:* From `get tags`  
  - *Outputs:* Array of tag names.  
  - *Edge Cases:* Empty tag list.

- **Code**  
  - *Type:* Code (JavaScript)  
  - *Role:* Formats tag names into dropdown options for user selection.  
  - *Config:*  
    - Splits comma-separated tag string.  
    - Cleans and capitalizes each tag name.  
    - Adds a "[create new]" option.  
    - Returns JSON for form dropdown options.  
  - *Inputs:* From `tags to string`  
  - *Outputs:* Dropdown options JSON.  
  - *Edge Cases:* Tags with junk tokens filtered out.

---

#### 2.3 Tag Selection & Processing

**Overview:**  
Presents a form to the user to select tags to convert to folders, then processes the selected tags one by one.

**Nodes Involved:**  
- select tags to move  
- Split Out the tags  
- extract name from form  
- Loop Over Items  
- pass all items

**Node Details:**

- **select tags to move**  
  - *Type:* Form  
  - *Role:* Displays dropdown with tag options for user to select multiple tags to process.  
  - *Config:*  
    - Multiselect dropdown using JSON from `Code` node.  
    - Webhook listens for form submission.  
  - *Inputs:* From `Code`  
  - *Outputs:* Selected tags array.  
  - *Edge Cases:* User must select at least one tag.

- **Split Out the tags**  
  - *Type:* Split Out  
  - *Role:* Splits selected tags array into individual tag items for processing.  
  - *Config:* Splits on `['Dropdown Options']` field.  
  - *Inputs:* From `select tags to move`  
  - *Outputs:* Single tag per item.  
  - *Edge Cases:* Empty selection.

- **extract name from form**  
  - *Type:* Set  
  - *Role:* Extracts the tag name string from the form data.  
  - *Config:* Sets `name` field from `Dropdown Options`.  
  - *Inputs:* From `Split Out the tags`  
  - *Outputs:* Tag name string.  
  - *Edge Cases:* Missing field in form data.

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes each tag sequentially for folder creation and workflow movement.  
  - *Config:* Default batch size (processes one tag at a time).  
  - *Inputs:* From `extract name from form`  
  - *Outputs:* Passes each tag item downstream.  
  - *Edge Cases:* Large number of tags may slow processing.

- **pass all items**  
  - *Type:* Set  
  - *Role:* Collects processed items for final summary.  
  - *Config:* Passes all fields through.  
  - *Inputs:* From `Loop Over Items` (after folder and workflow processing)  
  - *Outputs:* All processed tag items.  
  - *Edge Cases:* None.

---

#### 2.4 Folder Management

**Overview:**  
For each selected tag, checks if a folder with the same name exists in the user's personal project. If no folder exists, creates one. Normalizes folder names to title case.

**Nodes Involved:**  
- Get folders  
- If no folder  
- Split Out2  
- Normalize names  
- Remove Duplicates  
- Filter  
- Limit1  
- If folder exists  
- set folder name + id  
- set folder name + id1  
- set name  
- Create folders  
- Edit Fields  
- set global

**Node Details:**

- **Get folders**  
  - *Type:* HTTP Request  
  - *Role:* Queries folders in the user's project filtered by folder name (tag name).  
  - *Config:* GET `/rest/projects/{projectId}/folders` with filter by folder name and sorting by updated date descending.  
  - *Inputs:* From `Loop Over Items` (tag name and project ID)  
  - *Outputs:* Folder data matching tag name.  
  - *Edge Cases:* API failure, no folders found.

- **If no folder**  
  - *Type:* If  
  - *Role:* Checks if any folder was found (`count > 0`).  
  - *Config:* Condition on folder count.  
  - *Inputs:* From `Get folders`  
  - *Outputs:*  
    - True branch: Folder(s) exist  
    - False branch: No folder found  
  - *Edge Cases:* Folder count field missing.

- **Split Out2**  
  - *Type:* Split Out  
  - *Role:* Splits folder data array for further processing.  
  - *Inputs:* True branch of `If no folder`  
  - *Outputs:* Individual folder items.  
  - *Edge Cases:* Empty folder list.

- **Normalize names**  
  - *Type:* Set  
  - *Role:* Converts folder names to title case for consistency.  
  - *Config:* Lowercases and capitalizes first letter of each word in folder name.  
  - *Inputs:* From `Split Out2`  
  - *Outputs:* Normalized folder names.  
  - *Edge Cases:* Empty or malformed folder names.

- **Remove Duplicates**  
  - *Type:* Remove Duplicates  
  - *Role:* Removes duplicate folder names to ensure uniqueness.  
  - *Config:* Compares on `name` field.  
  - *Inputs:* From `Normalize names`  
  - *Outputs:* Unique folder names.  
  - *Edge Cases:* None.

- **Filter**  
  - *Type:* Filter  
  - *Role:* Filters folders to match the current tag name exactly.  
  - *Config:* Case-insensitive equality check on `name`.  
  - *Inputs:* From `Remove Duplicates`  
  - *Outputs:* Filtered folder(s).  
  - *Edge Cases:* No exact match found.

- **Limit1**  
  - *Type:* Limit  
  - *Role:* Limits output to one folder to avoid ambiguity.  
  - *Inputs:* From `Filter`  
  - *Outputs:* Single folder item.  
  - *Edge Cases:* Multiple folders found.

- **If folder exists**  
  - *Type:* If  
  - *Role:* Checks if folder name exists in the limited result.  
  - *Config:* Checks existence of `name` field.  
  - *Inputs:* From `Limit1`  
  - *Outputs:*  
    - True: Folder exists  
    - False: Folder does not exist  
  - *Edge Cases:* Missing fields.

- **set folder name + id**  
  - *Type:* Set  
  - *Role:* Sets folder name, ID, and tag for downstream use when folder was created.  
  - *Inputs:* From `Create folders`  
  - *Outputs:* Folder info.  
  - *Edge Cases:* Missing folder data.

- **set folder name + id1**  
  - *Type:* Set  
  - *Role:* Sets folder name, ID, and tag for downstream use when folder already exists.  
  - *Inputs:* From `If folder exists` (true branch)  
  - *Outputs:* Folder info.  
  - *Edge Cases:* Missing folder data.

- **set name**  
  - *Type:* Set  
  - *Role:* Normalizes tag name to title case for folder creation.  
  - *Inputs:* From `If folder exists` (false branch)  
  - *Outputs:* Normalized tag name.  
  - *Edge Cases:* None.

- **Create folders**  
  - *Type:* HTTP Request  
  - *Role:* Creates a new folder in the user's project with the normalized tag name.  
  - *Config:* POST `/rest/projects/{projectId}/folders` with folder `name` in body.  
  - *Inputs:* From `set name`  
  - *Outputs:* Folder creation response.  
  - *Edge Cases:* API failure, name conflicts.

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Sets folder name and tag fields for folder creation path.  
  - *Inputs:* From `If no folder` (false branch)  
  - *Outputs:* Folder info.  
  - *Edge Cases:* None.

- **set global**  
  - *Type:* Set  
  - *Role:* Stores folder ID and name globally for use in workflow movement.  
  - *Inputs:* From both `set folder name + id` and `set folder name + id1`  
  - *Outputs:* Folder info for next block.  
  - *Edge Cases:* None.

---

#### 2.5 Workflow Retrieval & Movement

**Overview:**  
Retrieves all workflows tagged with the current tag and moves them into the corresponding folder.

**Nodes Involved:**  
- get workflows  
- Move workflow to folder

**Node Details:**

- **get workflows**  
  - *Type:* n8n Node (n8n API)  
  - *Role:* Retrieves workflows filtered by the current tag.  
  - *Config:* Filter by `tags` field equal to current tag.  
  - *Credentials:* Uses n8n API credentials (OAuth2 or API key).  
  - *Inputs:* From `set global` (folder info and tag)  
  - *Outputs:* List of workflows with the tag.  
  - *Edge Cases:* API failure, no workflows found.

- **Move workflow to folder**  
  - *Type:* HTTP Request  
  - *Role:* Updates each workflow's `parentFolderId` to move it into the folder.  
  - *Config:* PATCH `/rest/workflows/{workflowId}` with body containing:  
    - `parentFolderId` set to folder ID from `set global`  
    - `versionId` from workflow data  
  - *Inputs:* From `get workflows`  
  - *Outputs:* Confirmation of workflow update.  
  - *Edge Cases:* API failure, permission issues, workflow locked.

---

#### 2.6 Completion & Feedback

**Overview:**  
After processing all tags and workflows, the workflow responds with a success message summarizing the number of workflows moved.

**Nodes Involved:**  
- end import

**Node Details:**

- **end import**  
  - *Type:* Form (completion)  
  - *Role:* Displays a completion message to the user.  
  - *Config:*  
    - Title: "Import complete"  
    - Message: "Successfully imported X workflows to the folders Y" where X is the count of workflows processed and Y is the list of tags selected.  
  - *Inputs:* From `pass all items`  
  - *Outputs:* None (end of workflow)  
  - *Edge Cases:* None.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                          | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                       |
|------------------------|----------------------|----------------------------------------|--------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger         | Entry point, triggers workflow          | None                           | set credentials                |                                                                                                 |
| set credentials         | Set                  | Stores n8n URL, username, password      | On form submission             | login n8n                     |                                                                                                 |
| login n8n              | HTTP Request         | Authenticates user, obtains session     | set credentials                | my-projects                   | Step 1: Login to n8n, and get the tags we have for our personal owned projects                   |
| my-projects             | HTTP Request         | Retrieves user's projects                | login n8n                     | Split Out                    |                                                                                                 |
| Split Out              | Split Out            | Splits projects into individual items   | my-projects                   | filter owned projects         |                                                                                                 |
| filter owned projects   | Filter               | Filters projects owned by user           | Split Out                    | get tags                     |                                                                                                 |
| get tags               | HTTP Request         | Retrieves tags for owned projects        | filter owned projects          | tags to string               |                                                                                                 |
| tags to string          | Set                  | Extracts tag names array                  | get tags                     | Code                        | Step 2: Extract the tags as a json string, and format this into a suitable format for form input |
| Code                   | Code                 | Formats tags into dropdown options       | tags to string                | select tags to move          |                                                                                                 |
| select tags to move     | Form                 | User selects tags to process              | Code                         | Split Out the tags           |                                                                                                 |
| Split Out the tags      | Split Out            | Splits selected tags into individual items | select tags to move           | extract name from form       |                                                                                                 |
| extract name from form  | Set                  | Extracts tag name string                   | Split Out the tags            | Loop Over Items              |                                                                                                 |
| Loop Over Items         | Split In Batches     | Processes each tag sequentially           | extract name from form        | pass all items, Get folders  | Step 3: Extract the form details and loop over each tag to process                              |
| Get folders             | HTTP Request         | Checks for existing folders by tag name  | Loop Over Items              | If no folder                | Step 3: We search for the folders and filter based on the number of folders found               |
| If no folder            | If                   | Branches based on folder existence        | Get folders                  | Split Out2, Edit Fields      |                                                                                                 |
| Split Out2              | Split Out            | Splits folder data array                   | If no folder (true)           | Normalize names             | Step 4 a): If more than 1 folder is found, dedupe and limit to one folder                       |
| Normalize names         | Set                  | Normalizes folder names to title case     | Split Out2                   | Remove Duplicates           |                                                                                                 |
| Remove Duplicates       | Remove Duplicates    | Removes duplicate folder names             | Normalize names              | Filter                      |                                                                                                 |
| Filter                 | Filter               | Filters folders matching tag name          | Remove Duplicates            | Limit1                      |                                                                                                 |
| Limit1                 | Limit                | Limits to one folder                        | Filter                      | If folder exists            |                                                                                                 |
| If folder exists        | If                   | Checks if folder exists                     | Limit1                      | set folder name + id1, set name |                                                                                                 |
| set folder name + id1   | Set                  | Sets folder info for existing folder       | If folder exists (true)       | set global                  |                                                                                                 |
| set name               | Set                  | Normalizes tag name for folder creation    | If folder exists (false)      | Create folders              |                                                                                                 |
| Create folders          | HTTP Request         | Creates new folder in project               | set name                    | set folder name + id        | Step 4 b): If no folder is found, create a new folder                                          |
| set folder name + id    | Set                  | Sets folder info for newly created folder  | Create folders              | set global                  |                                                                                                 |
| Edit Fields             | Set                  | Sets folder name and tag for folder creation | If no folder (false)          | set name                   |                                                                                                 |
| set global              | Set                  | Stores folder ID and name globally          | set folder name + id, set folder name + id1 | get workflows              | Step 5: Merge the paths so we use one workflow                                                  |
| get workflows           | n8n Node (n8n API)   | Retrieves workflows tagged with current tag | set global                  | Move workflow to folder     | Step 6: Get the workflows and move them to the respective folders                              |
| Move workflow to folder | HTTP Request         | Moves workflows into corresponding folder   | get workflows               | Loop Over Items             |                                                                                                 |
| pass all items          | Set                  | Collects processed items for summary         | Loop Over Items             | end import                  |                                                                                                 |
| end import              | Form (completion)    | Displays completion message to user           | pass all items              | None                       | Step 7: Respond with a success message                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named `On form submission` with webhook ID set. Configure form with title "Tags to Folders", a "Submit" button, and a description "Convert all tags into folders".

2. **Add a Set node** named `set credentials` connected from `On form submission`. Configure three string fields:
   - `n8n`: Your n8n instance URL without trailing slash.
   - `username`: Your n8n username.
   - `password`: Your n8n password.

3. **Add an HTTP Request node** named `login n8n` connected from `set credentials`. Configure:
   - Method: POST
   - URL: `={{ $json.n8n }}/rest/login`
   - Body parameters: `emailOrLdapLoginId` = `={{ $json.username }}`, `password` = `={{ $json.password }}`
   - Headers: Accept JSON, user-agent spoofed.
   - Response: Full response including headers.

4. **Add an HTTP Request node** named `my-projects` connected from `login n8n`. Configure:
   - Method: GET
   - URL: `={{ $('set credentials').item.json.n8n }}/rest/projects/my-projects`
   - Headers: Include cookie from `login n8n` (`set-cookie` header).
   - Response: JSON.

5. **Add a Split Out node** named `Split Out` connected from `my-projects`. Configure to split on `data`.

6. **Add a Filter node** named `filter owned projects` connected from `Split Out`. Configure:
   - Conditions:
     - Project name contains the email extracted from username.
     - Role equals `project:personalOwner`.

7. **Add an HTTP Request node** named `get tags` connected from `filter owned projects`. Configure:
   - Method: GET
   - URL: `={{ $('set credentials').item.json.n8n }}/rest/tags`
   - Headers: Include cookie from `login n8n`.
   - Query parameter: `withUsageCount=false`
   - Response: JSON, never error.

8. **Add a Set node** named `tags to string` connected from `get tags`. Configure to map `body.data` array to extract `name` fields into a new field `name` (array).

9. **Add a Code node** named `Code` connected from `tags to string`. Paste the provided JavaScript code that:
   - Splits tag names,
   - Capitalizes them,
   - Filters junk tokens,
   - Adds a "[create new]" option,
   - Returns dropdown options JSON.

10. **Add a Form node** named `select tags to move` connected from `Code`. Configure:
    - Dropdown multiselect field using JSON from `Code`.
    - Webhook ID set.

11. **Add a Split Out node** named `Split Out the tags` connected from `select tags to move`. Configure to split on `['Dropdown Options']`.

12. **Add a Set node** named `extract name from form` connected from `Split Out the tags`. Configure to set `name` field from the dropdown selection.

13. **Add a Split In Batches node** named `Loop Over Items` connected from `extract name from form`. Default batch size.

14. **Add an HTTP Request node** named `Get folders` connected from `Loop Over Items`. Configure:
    - Method: GET
    - URL: `/rest/projects/{{ projectId }}/folders` with query filter for folder name equal to current tag name.
    - Headers: Include cookie.

15. **Add an If node** named `If no folder` connected from `Get folders`. Configure condition: `count > 0`.

16. **From true branch of `If no folder`:**  
    - Add a Split Out node `Split Out2`.  
    - Add a Set node `Normalize names` to convert folder names to title case.  
    - Add a Remove Duplicates node `Remove Duplicates` comparing on `name`.  
    - Add a Filter node `Filter` to match folder name exactly.  
    - Add a Limit node `Limit1` limiting to 1 item.  
    - Add an If node `If folder exists` checking if `name` exists.  
    - From true branch, add Set node `set folder name + id1` to set folder info.  
    - From false branch, add Set node `set name` to normalize tag name for folder creation, then HTTP Request node `Create folders` to create folder, then Set node `set folder name + id` to set folder info.

17. **From false branch of `If no folder`:**  
    - Add Set node `Edit Fields` to set folder name and tag.  
    - Connect to `set name` node in folder creation path.

18. **Merge both folder info paths into a Set node `set global`** to store folder ID and name globally.

19. **Add an n8n node `get workflows`** connected from `set global`. Configure to filter workflows by current tag. Use n8n API credentials.

20. **Add an HTTP Request node `Move workflow to folder`** connected from `get workflows`. Configure to PATCH workflow endpoint with `parentFolderId` from `set global` and `versionId` from workflow data. Include cookie header.

21. **Connect `Move workflow to folder` back to `Loop Over Items`** to continue processing next tag.

22. **Add a Set node `pass all items`** connected from `Loop Over Items` to collect all processed items.

23. **Add a Form node `end import`** connected from `pass all items`. Configure completion form with title "Import complete" and message showing number of workflows imported and tags processed.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow built by Zacharia Kimotho - Imperol                                                  | [LinkedIn](https://www.linkedin.com/in/zacharia-kimotho/)                                       |
| Read more about this workflow and its usage                                                   | [funautomations.io article](https://funautomations.io/workflows/how-to-convert-n8n-tags-into-folders/) |
| n8n introduced folders as an improvement over tags for workflow management                    | Workflow assumes tag names are valid folder names; workflows with multiple tags assigned to last processed tag's folder |
| User must have existing workflows and tags before running this workflow                       |                                                                                                 |
| Credentials needed: n8n login details and n8n API credentials                                 |                                                                                                 |
| Potential failure points include authentication errors, API permission issues, and empty data |                                                                                                 |

---

This detailed reference document provides a comprehensive understanding of the workflow structure, logic, and configuration, enabling advanced users and automation agents to reproduce, modify, and troubleshoot the workflow effectively.