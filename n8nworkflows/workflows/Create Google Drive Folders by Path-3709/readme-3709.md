Create Google Drive Folders by Path

https://n8nworkflows.xyz/workflows/create-google-drive-folders-by-path-3709


# Create Google Drive Folders by Path

### 1. Workflow Overview

This workflow automates the creation of nested folders in Google Drive based on a provided path string (e.g., `Projects/Clients/Reports`). It is designed for use cases where structured folder hierarchies need to be programmatically ensured before uploading or managing files within Google Drive. The workflow accepts a root folder ID and a slash-separated path, then iteratively checks for the existence of each folder in the path, creating any missing folders along the way. Finally, it outputs the ID of the deepest folder created or found, enabling seamless integration with subsequent Google Drive operations.

**Logical Blocks:**

- **1.1 Input Reception & Initialization:** Receives input parameters and prepares the path for processing.
- **1.2 Path Parsing & Preparation:** Splits the path string into folder name segments.
- **1.3 Folder Existence Check & Creation Loop:** Iteratively checks if each folder exists under the current parent folder; creates it if missing.
- **1.4 Completion & Output:** Determines when the entire path has been processed and outputs the final folder ID.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

**Overview:**  
This block handles the initial input reception, either manually triggered or invoked from another workflow, and sets up the initial parameters for the folder creation process.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Dummy input data  
- Execute Workflow Trigger

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution of the workflow for testing.  
  - *Configuration:* Default manual trigger with no parameters.  
  - *Inputs:* None  
  - *Outputs:* Triggers downstream nodes.  
  - *Edge Cases:* None specific, but manual trigger requires user interaction.

- **Dummy input data**  
  - *Type:* Set  
  - *Role:* Provides test input data for manual runs.  
  - *Configuration:* Sets two string fields:  
    - `google_drive_folder_id` = `"root"` (indicating Google Drive root folder)  
    - `desired_path` = `"testXavier/2024/Q4/03 Documenten"` (example nested path)  
  - *Inputs:* Triggered by manual trigger  
  - *Outputs:* Passes the input data downstream.  
  - *Edge Cases:* Hardcoded test data; for production, inputs should come externally.

- **Execute Workflow Trigger**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Allows this workflow to be triggered from other workflows.  
  - *Configuration:* Default, no parameters.  
  - *Inputs:* External trigger  
  - *Outputs:* Passes input data downstream.  
  - *Edge Cases:* Requires proper input data format when triggered externally.

---

#### 1.2 Path Parsing & Preparation

**Overview:**  
This block splits the input path string into an array of folder names and prepares the data structure for iterative processing.

**Nodes Involved:**  
- Split the desired path  
- Create desired path

**Node Details:**

- **Split the desired path**  
  - *Type:* Code  
  - *Role:* Parses the `desired_path` string into an array by splitting on `/`.  
  - *Configuration:* JavaScript code:  
    ```js
    $input.item.json.desired_path = $input.item.json.desired_path.split('/');
    return $input.item;
    ```  
  - *Inputs:* Receives JSON with `desired_path` as string  
  - *Outputs:* JSON with `desired_path` as array of folder names  
  - *Edge Cases:* Empty or malformed path strings could cause unexpected results; no explicit validation.

- **Create desired path**  
  - *Type:* Code  
  - *Role:* Passes through the current Google Drive folder ID and the array of folder names for further processing.  
  - *Configuration:* JavaScript code:  
    ```js
    return {
      google_drive_folder_id: $json.google_drive_folder_id,
      desired_path: $json.desired_path,
    };
    ```  
  - *Inputs:* JSON with `google_drive_folder_id` and `desired_path` array  
  - *Outputs:* Same JSON structure forwarded  
  - *Edge Cases:* None significant; simple passthrough.

---

#### 1.3 Folder Existence Check & Creation Loop

**Overview:**  
This core block iteratively processes each folder name in the path array. For each folder, it checks if the folder exists under the current parent folder. If it exists, it updates the parent folder ID for the next iteration; if not, it creates the folder and updates the parent folder ID accordingly.

**Nodes Involved:**  
- Check if top folder exists  
- If top folder doesn't exist  
- Create new subfolder  
- Set parameters for next run  
- If path has been completely created

**Node Details:**

- **Check if top folder exists**  
  - *Type:* Google Drive (File/Folder Search)  
  - *Role:* Searches for a folder with the current folder name inside the current parent folder.  
  - *Configuration:*  
    - Resource: `fileFolder`  
    - Query: Folder name equals the first element of `desired_path` array (`{{$json.desired_path[0]}}`)  
    - Folder ID filter: Current `google_drive_folder_id`  
    - Search restricted to folders only  
    - Credentials: Google Drive OAuth2 (configured externally)  
  - *Inputs:* JSON with `google_drive_folder_id` and `desired_path` array  
  - *Outputs:* Search results (empty if folder not found)  
  - *Edge Cases:*  
    - API rate limits or auth errors  
    - Folder name collisions (multiple folders with same name) — only first found is used  
    - Empty or invalid folder ID inputs

- **If top folder doesn't exist**  
  - *Type:* If  
  - *Role:* Checks if the search result from previous node is empty (folder not found).  
  - *Configuration:* Condition tests if the JSON result is empty.  
  - *Inputs:* Output of "Check if top folder exists"  
  - *Outputs:*  
    - True branch: Folder does not exist  
    - False branch: Folder exists  
  - *Edge Cases:* Expression failures if input data structure changes.

- **Create new subfolder**  
  - *Type:* Google Drive (Folder Create)  
  - *Role:* Creates a new folder with the current folder name inside the current parent folder.  
  - *Configuration:*  
    - Name: Current folder name (`{{$('Create desired path').item.json.desired_path[0]}}`)  
    - Parent folder ID: Current `google_drive_folder_id`  
    - Drive ID: "My Drive" (default Google Drive root)  
    - Credentials: Google Drive OAuth2  
  - *Inputs:* Triggered only if folder does not exist  
  - *Outputs:* Newly created folder metadata including ID  
  - *Edge Cases:*  
    - API errors, quota limits  
    - Folder name conflicts (Google Drive allows duplicate folder names)  
    - Permissions errors

- **Set parameters for next run**  
  - *Type:* Code  
  - *Role:* Updates the `desired_path` array by removing the first element (folder just processed) and sets the `google_drive_folder_id` to the ID of the current folder (existing or newly created) for the next iteration.  
  - *Configuration:* JavaScript code:  
    ```js
    const desired_path = $('Create desired path').item.json.desired_path;
    desired_path.shift();

    return {
      desired_path: desired_path,
      google_drive_folder_id: $json.id,
    }
    ```  
  - *Inputs:* Receives the current folder ID from either existing folder or newly created folder  
  - *Outputs:* Updated JSON with shortened `desired_path` and updated `google_drive_folder_id`  
  - *Edge Cases:*  
    - If `desired_path` is empty, subsequent logic handles completion  
    - Expression failures if referenced nodes or fields change

- **If path has been completely created**  
  - *Type:* If  
  - *Role:* Checks if the `desired_path` array is now empty, indicating all folders have been processed.  
  - *Configuration:* Condition tests if `desired_path` array is empty.  
  - *Inputs:* Output of "Set parameters for next run"  
  - *Outputs:*  
    - True branch: Path creation complete  
    - False branch: Continue processing next folder  
  - *Edge Cases:* Expression failures if data structure changes.

---

#### 1.4 Completion & Output

**Overview:**  
This block outputs the final folder ID after the entire path has been processed, making it available for downstream nodes or workflows.

**Nodes Involved:**  
- Return the ID of the last folder

**Node Details:**

- **Return the ID of the last folder**  
  - *Type:* Set  
  - *Role:* Sets the output JSON with the final `google_drive_folder_id` representing the deepest folder created or found.  
  - *Configuration:* Sets `google_drive_folder_id` to the value from "Set parameters for next run" node.  
  - *Inputs:* Triggered when path creation is complete  
  - *Outputs:* JSON with final folder ID for downstream use  
  - *Edge Cases:* None significant; output depends on prior successful processing.

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                        | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                          |
|------------------------------|----------------------------|-------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger             | Manual start of workflow             | None                          | Dummy input data               | ## Test data for the workflow Use this in case you want to test the workflow.                      |
| Dummy input data              | Set                        | Provides test input parameters       | When clicking ‘Test workflow’  | Split the desired path         | ## Test data for the workflow Use this in case you want to test the workflow.                      |
| Execute Workflow Trigger      | Execute Workflow Trigger   | Allows external workflow invocation  | None                          | Split the desired path         | ## Triggered from another workflow This workflow is intended to be triggered by other workflows. Don't copy/paste this workflow as it will be more difficult to maintain and keep up-to-date. |
| Split the desired path        | Code                       | Splits path string into array        | Dummy input data / Execute Workflow Trigger | Create desired path            | ## Main loop Take the desired_path and split it into parts. Eg: `Projects/Clients/Reports` will turn into 3 parts: Projects, Clients, Reports. We then check if the top folder exists and create it if not. We repeat this process until all subfolders have been created and correctly nested. |
| Create desired path           | Code                       | Passes folder ID and path array      | Split the desired path         | Check if top folder exists     | ## Main loop Take the desired_path and split it into parts. Eg: `Projects/Clients/Reports` will turn into 3 parts: Projects, Clients, Reports. We then check if the top folder exists and create it if not. We repeat this process until all subfolders have been created and correctly nested. |
| Check if top folder exists    | Google Drive               | Searches for current folder          | Create desired path            | If top folder doesn't exist    | ## Main loop Take the desired_path and split it into parts. Eg: `Projects/Clients/Reports` will turn into 3 parts: Projects, Clients, Reports. We then check if the top folder exists and create it if not. We repeat this process until all subfolders have been created and correctly nested. |
| If top folder doesn't exist   | If                         | Branches based on folder existence   | Check if top folder exists     | Create new subfolder / Set parameters for next run | ## Main loop Take the desired_path and split it into parts. Eg: `Projects/Clients/Reports` will turn into 3 parts: Projects, Clients, Reports. We then check if the top folder exists and create it if not. We repeat this process until all subfolders have been created and correctly nested. |
| Create new subfolder          | Google Drive               | Creates folder if missing             | If top folder doesn't exist    | Set parameters for next run    | ## Main loop Take the desired_path and split it into parts. Eg: `Projects/Clients/Reports` will turn into 3 parts: Projects, Clients, Reports. We then check if the top folder exists and create it if not. We repeat this process until all subfolders have been created and correctly nested. |
| Set parameters for next run   | Code                       | Updates path array and parent folder | Create new subfolder / If top folder doesn't exist (false branch) | If path has been completely created | ## Main loop Take the desired_path and split it into parts. Eg: `Projects/Clients/Reports` will turn into 3 parts: Projects, Clients, Reports. We then check if the top folder exists and create it if not. We repeat this process until all subfolders have been created and correctly nested. |
| If path has been completely created | If                  | Checks if all folders processed      | Set parameters for next run    | Return the ID of the last folder / Create desired path | ## Rerturn data Here we return the ID of the last folder in the path, so you can start uploading new files to it. |
| Return the ID of the last folder | Set                    | Outputs final folder ID               | If path has been completely created (true branch) | None                          | ## Rerturn data Here we return the ID of the last folder in the path, so you can start uploading new files to it. |
| Sticky Note                  | Sticky Note                | Overview and usage instructions      | None                          | None                          | # Create Google Drive Folders by Path This workflow created nested Google Drive folder from a path string and returns the ID of the final folder for immediate use. Use this workflow in your other flows by calling it directly with the following data: - `google_drive_folder_id` -> The ID of the folder where you want to create additional folders in. You can use "root" if you want to begin at root level of your Drive. - `desired_path` -> The folder structure you'd like to create in Google Drive. Each folder is separated by a slash, eg: `Projects/Clients/Reports` |
| Sticky Note1                 | Sticky Note                | Test data description                | None                          | None                          | ## Test data for the workflow Use this in case you want to test the workflow.                      |
| Sticky Note2                 | Sticky Note                | Trigger usage guidance               | None                          | None                          | ## Triggered from another workflow This workflow is intended to be triggered by other workflows. Don't copy/paste this workflow as it will be more difficult to maintain and keep up-to-date. |
| Sticky Note3                 | Sticky Note                | Main loop explanation                | None                          | None                          | ## Main loop Take the desired_path and split it into parts. Eg: `Projects/Clients/Reports` will turn into 3 parts: Projects, Clients, Reports. We then check if the top folder exists and create it if not. We repeat this process until all subfolders have been created and correctly nested. |
| Sticky Note4                 | Sticky Note                | Output explanation                   | None                          | None                          | ## Rerturn data Here we return the ID of the last folder in the path, so you can start uploading new files to it. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing.

2. **Create Set Node ("Dummy input data")**  
   - Type: Set  
   - Purpose: Provide test inputs.  
   - Parameters:  
     - `google_drive_folder_id` = `"root"` (string)  
     - `desired_path` = `"testXavier/2024/Q4/03 Documenten"` (string)  
   - Connect Manual Trigger → Dummy input data.

3. **Create Execute Workflow Trigger Node**  
   - Type: Execute Workflow Trigger  
   - Purpose: Allow external workflows to trigger this workflow.  
   - Connect no input; output connects to next node.

4. **Create Code Node ("Split the desired path")**  
   - Type: Code  
   - Purpose: Split the `desired_path` string into an array.  
   - Parameters (JavaScript):  
     ```js
     $input.item.json.desired_path = $input.item.json.desired_path.split('/');
     return $input.item;
     ```  
   - Connect Dummy input data → Split the desired path  
   - Connect Execute Workflow Trigger → Split the desired path

5. **Create Code Node ("Create desired path")**  
   - Type: Code  
   - Purpose: Pass through `google_drive_folder_id` and `desired_path` array.  
   - Parameters (JavaScript):  
     ```js
     return {
       google_drive_folder_id: $json.google_drive_folder_id,
       desired_path: $json.desired_path,
     };
     ```  
   - Connect Split the desired path → Create desired path

6. **Create Google Drive Node ("Check if top folder exists")**  
   - Type: Google Drive (File/Folder Search)  
   - Purpose: Search for folder named as first element of `desired_path` inside current parent folder.  
   - Parameters:  
     - Resource: `fileFolder`  
     - Query: `={{ $json.desired_path[0] }}`  
     - Folder ID filter: `={{ $json.google_drive_folder_id }}`  
     - Filter to folders only  
   - Credentials: Configure with your Google Drive OAuth2 credentials.  
   - Connect Create desired path → Check if top folder exists

7. **Create If Node ("If top folder doesn't exist")**  
   - Type: If  
   - Purpose: Check if search result is empty (folder not found).  
   - Condition: Check if JSON output from previous node is empty.  
   - Connect Check if top folder exists → If top folder doesn't exist

8. **Create Google Drive Node ("Create new subfolder")**  
   - Type: Google Drive (Folder Create)  
   - Purpose: Create folder if missing.  
   - Parameters:  
     - Name: `={{ $('Create desired path').item.json.desired_path[0] }}`  
     - Parent folder ID: `={{ $('Create desired path').item.json.google_drive_folder_id }}`  
     - Drive ID: "My Drive"  
   - Credentials: Same Google Drive OAuth2 credentials.  
   - Connect If top folder doesn't exist (true branch) → Create new subfolder

9. **Create Code Node ("Set parameters for next run")**  
   - Type: Code  
   - Purpose: Remove first folder from path array and update parent folder ID for next iteration.  
   - Parameters (JavaScript):  
     ```js
     const desired_path = $('Create desired path').item.json.desired_path;
     desired_path.shift();

     return {
       desired_path: desired_path,
       google_drive_folder_id: $json.id,
     }
     ```  
   - Connect Create new subfolder → Set parameters for next run  
   - Connect If top folder doesn't exist (false branch) → Set parameters for next run  
     (This handles the case where folder exists; the existing folder ID is used.)

10. **Create If Node ("If path has been completely created")**  
    - Type: If  
    - Purpose: Check if `desired_path` array is empty, meaning all folders processed.  
    - Condition: Check if `desired_path` array is empty.  
    - Connect Set parameters for next run → If path has been completely created

11. **Create Set Node ("Return the ID of the last folder")**  
    - Type: Set  
    - Purpose: Output the final folder ID for downstream use.  
    - Parameters:  
      - Set `google_drive_folder_id` to `={{ $('Set parameters for next run').item.json.google_drive_folder_id }}`  
    - Connect If path has been completely created (true branch) → Return the ID of the last folder

12. **Loop Back for Iteration**  
    - Connect If path has been completely created (false branch) → Create desired path  
    - This loop continues until all folders are processed.

13. **Sticky Notes (Optional)**  
    - Add sticky notes to document workflow purpose, test data, trigger usage, main loop explanation, and output explanation as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is intended to be triggered by other workflows rather than copied and pasted. This ensures maintainability and easier updates.                                                                                                  | Sticky Note2                                                                                        |
| To use the root of Google Drive as starting folder, set `google_drive_folder_id` to `"root"`. For specific folders, use the folder ID from the URL.                                                                                             | Workflow Description                                                                               |
| Google Drive API credentials must be configured in n8n and linked to the Google Drive nodes for folder search and creation.                                                                                                                  | Setup Steps                                                                                       |
| The workflow handles folder name collisions by using the first folder found with the matching name. Google Drive allows multiple folders with the same name under one parent, which can cause ambiguity.                                        | Edge Cases in Folder Existence Check node                                                        |
| The iterative loop removes the first element of the path array after each folder is processed, ensuring eventual completion when the array is empty.                                                                                          | Main loop explanation (Sticky Note3)                                                             |
| The final output is the ID of the last folder created or found, allowing immediate use in subsequent Google Drive operations like file uploads or document creation.                                                                           | Output explanation (Sticky Note4)                                                                |
| Example usage in other workflows involves calling this workflow with parameters `google_drive_folder_id` and `desired_path` to dynamically create nested folders before further processing.                                                     | Workflow Description and Sticky Note                                                             |
| For more information about Google Drive folder IDs and API usage, consult Google Drive API documentation: https://developers.google.com/drive/api/v3/reference/files | External resource for understanding folder IDs and API behavior                                  |

---

This completes the comprehensive reference documentation for the "Create Google Drive Folders by Path" n8n workflow.