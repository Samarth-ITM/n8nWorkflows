Import workflows and map their credentials using a Multi-Form

https://n8nworkflows.xyz/workflows/import-workflows-and-map-their-credentials-using-a-multi-form-2506


# Import workflows and map their credentials using a Multi-Form

### 1. Workflow Overview

This workflow, titled **"Workflow Importer"**, facilitates importing an n8n workflow either from a local JSON file or from a remote n8n instance, with an interactive multi-form interface guiding users through the process. The workflow’s main goal is to enable seamless transfer and credential mapping of workflows between n8n instances, primarily for testing or development environments.

The workflow logic is organized into the following logical blocks:

- **1.1 Input Reception and Source Determination**  
  Accepts user input to select the workflow source (file upload or remote instance) and branches accordingly.

- **1.2 Remote Workflow Retrieval**  
  If the remote instance option is selected, it fetches the list of workflows from the chosen remote instance and allows the user to select one.

- **1.3 Workflow and Credentials Extraction**  
  Processes the selected or uploaded workflow JSON and concurrently exports all local credentials in decrypted form.

- **1.4 Credential Analysis and Mapping**  
  Extracts credentials used in the imported workflow, deduplicates them by name, and presents a form to map each to existing local credentials or create new ones.

- **1.5 Credential Creation and Workflow Update**  
  Creates new empty credentials for those mapped to “create new” options and updates the workflow credential IDs accordingly.

- **1.6 Workflow Creation and Completion Feedback**  
  Creates the updated workflow on the current instance via API and provides success or failure feedback to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Source Determination

- **Overview:**  
  This block initiates the process by presenting a form for the user to select the source of the workflow to import: either upload a JSON file or import from a remote n8n instance.

- **Nodes Involved:**  
  - On form submission  
  - Determine Workflow Source  
  - Upload File  
  - Settings  
  - Sticky Note7, Sticky Note8

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger (entry point)  
    - *Role:* Presents a form with a dropdown to select "File Upload" or "Remote Instance" as source.  
    - *Key Config:* Form field "Source" with options "File Upload" and "Remote Instance".  
    - *Input:* HTTP request (form submission)  
    - *Output:* JSON with user selection  
    - *Edge cases:* User submits invalid or no option; form validation ensures required field.

  - **Determine Workflow Source**  
    - *Type:* If node  
    - *Role:* Routes the workflow based on the "Source" value from the form.  
    - *Key Config:* Checks if Source equals "File Upload"  
    - *Input:* Output from On form submission  
    - *Output:* Two branches - true (File Upload) and false (Remote Instance)  
    - *Edge cases:* Unexpected values; strict case-sensitive check.

  - **Upload File**  
    - *Type:* Form node  
    - *Role:* Presents a form to upload an n8n workflow JSON file.  
    - *Key Config:* Single required file input accepting `.json`.  
    - *Input:* Triggered if file upload selected  
    - *Output:* Binary data of the uploaded file  
    - *Edge cases:* Uploading invalid JSON or wrong file type.

  - **Settings**  
    - *Type:* Set node  
    - *Role:* Holds predefined remote instances configured as an array of objects, each with `name`, `apiKey`, and `baseUrl`.  
    - *Key Config:* Static JSON array configured in the node parameters.  
    - *Input:* Triggered if remote instance selected  
    - *Output:* JSON object with `remoteInstances` array available for further steps  
    - *Edge cases:* Misconfigured or missing remote instances.

---

#### 1.2 Remote Workflow Retrieval

- **Overview:**  
  If the user selects "Remote Instance," this block lets the user choose which remote instance to use, fetches workflows from its API, and allows choosing the specific workflow to import.

- **Nodes Involved:**  
  - Generate Instance Options  
  - Choose Instance  
  - Prepare Request Data  
  - Get Workflows  
  - Split Out Workflows  
  - Sort by updatedAt DESC  
  - Get Workflow Names  
  - Choose Workflow  
  - Get Selected Workflow  
  - Sticky Note3, Sticky Note10, Sticky Note11, Sticky Note12, Sticky Note13, Sticky Note15, Sticky Note43

- **Node Details:**

  - **Generate Instance Options**  
    - *Type:* Code  
    - *Role:* Converts the remoteInstances array from Settings into dropdown options for the form.  
    - *Key Config:* Loops over each instance, extracting `name` for dropdown values.  
    - *Input:* Output from Settings node  
    - *Output:* JSON string with dropdown options  
    - *Edge cases:* Empty remoteInstances array results in empty options.

  - **Choose Instance**  
    - *Type:* Form node  
    - *Role:* Presents a dropdown form to select the remote instance from which to fetch workflows.  
    - *Key Config:* Dropdown populated by Generate Instance Options output  
    - *Input:* Output from Generate Instance Options  
    - *Output:* JSON with selected instance name  
    - *Edge cases:* User cancels or selects invalid instance.

  - **Prepare Request Data**  
    - *Type:* Code  
    - *Role:* Finds the selected instance's full config from the remoteInstances array for API requests.  
    - *Key Config:* Matches selected instance name against Settings.remoteInstances  
    - *Input:* Output from Choose Instance and Settings  
    - *Output:* JSON containing the selected `instance` object  
    - *Edge cases:* Selected instance not found.

  - **Get Workflows**  
    - *Type:* HTTP Request  
    - *Role:* Calls the remote instance API (`/workflows`) with API key header to retrieve workflows.  
    - *Key Config:* GET request with `X-N8N-API-KEY` header, limit=250 workflows  
    - *Input:* Output from Prepare Request Data  
    - *Output:* JSON list of workflows or error  
    - *Edge cases:* API key issues, network errors, API rate limits

  - **Split Out Workflows**  
    - *Type:* Split Out  
    - *Role:* Splits the workflows array into individual items for sorting  
    - *Input:* Output from Get Workflows  
    - *Output:* Individual workflow items  
    - *Edge cases:* Empty workflow list

  - **Sort by updatedAt DESC**  
    - *Type:* Sort  
    - *Role:* Sorts workflows descending by last update time  
    - *Input:* Split workflows  
    - *Output:* Sorted workflows  
    - *Edge cases:* Missing or malformed date fields

  - **Get Workflow Names**  
    - *Type:* Code  
    - *Role:* Maps sorted workflows to dropdown options by workflow name  
    - *Input:* Sorted workflows  
    - *Output:* JSON string with options for form dropdown  
    - *Edge cases:* Duplicate workflow names may confuse selection

  - **Choose Workflow**  
    - *Type:* Form  
    - *Role:* Presents dropdown to select one workflow from remote instance  
    - *Key Config:* Uses options generated by Get Workflow Names  
    - *Input:* Output from Get Workflow Names  
    - *Output:* JSON with selected workflow name  
    - *Edge cases:* User selects none or cancels

  - **Get Selected Workflow**  
    - *Type:* Code  
    - *Role:* Finds the full workflow object matching the selected workflow name  
    - *Input:* Output from Choose Workflow and Get Workflows  
    - *Output:* JSON of the selected workflow stored as `workflowData`  
    - *Edge cases:* Selected workflow not found (returns false)

---

#### 1.3 Workflow and Credentials Extraction

- **Overview:**  
  Processes the imported or uploaded workflow JSON and concurrently exports all credentials from the current instance (decrypted), parsing both for later mapping.

- **Nodes Involved:**  
  - Extract from File  
  - Export Credentials  
  - Get Credentials Data  
  - Binary to JSON  
  - Rename Keys  
  - Merge  
  - No Operation  
  - Sticky Note14, Sticky Note16

- **Node Details:**

  - **Extract from File**  
    - *Type:* Extract From File  
    - *Role:* Converts the uploaded binary workflow file to JSON  
    - *Key Config:* Reads binary property `Workflow_File` and outputs JSON under `workflowData`  
    - *Input:* Output from Upload File form  
    - *Output:* JSON workflow object  
    - *Edge cases:* Invalid JSON file causes failure

  - **Export Credentials**  
    - *Type:* Execute Command  
    - *Role:* Runs CLI command `n8n export:credentials` with parameters to export all decrypted credentials to `/tmp/cred`  
    - *Key Config:* Export all credentials, pretty format, decrypted  
    - *Input:* Triggered after Extract from File or Get Selected Workflow  
    - *Output:* Command output with credentials saved to file  
    - *Edge cases:* Permissions to write/read `/tmp/cred`, command failure

  - **Get Credentials Data**  
    - *Type:* Read Write File  
    - *Role:* Reads the credentials JSON file `/tmp/cred` for workflow use  
    - *Input:* Output from Export Credentials  
    - *Output:* Binary data of credentials JSON file  
    - *Edge cases:* Missing file or read errors

  - **Binary to JSON**  
    - *Type:* Extract From File  
    - *Role:* Converts credentials binary data to JSON  
    - *Input:* Output from Get Credentials Data  
    - *Output:* Credentials JSON object  
    - *Edge cases:* Malformed JSON

  - **Rename Keys**  
    - *Type:* Rename Keys  
    - *Role:* Renames the `data` key to `allCredentials` for consistency  
    - *Input:* Output from Binary to JSON  
    - *Output:* JSON with `allCredentials` key  
    - *Edge cases:* Missing `data` key

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Combines the workflow JSON (`workflowData`) and credentials JSON (`allCredentials`) into a single item for parallel processing  
    - *Key Config:* Combine mode with unpaired included, combined by position  
    - *Input:* Two inputs: one from Rename Keys (credentials), one from Extract from File or Get Selected Workflow (workflow)  
    - *Output:* Single merged JSON object with workflow and credentials  
    - *Edge cases:* Mismatched inputs, missing data

  - **No Operation**  
    - *Type:* NoOp  
    - *Role:* Placeholder to unify branches from file upload and remote retrieval before merge  
    - *Input:* From Extract from File or Get Selected Workflow  
    - *Output:* Pass-through  
    - *Edge cases:* None

---

#### 1.4 Credential Analysis and Mapping

- **Overview:**  
  Extracts credentials used by the imported workflow nodes, removes duplicates by name, generates dropdown options including existing credentials and an option to create new ones, and presents a multi-form for user mapping.

- **Nodes Involved:**  
  - Split Out Nodes  
  - Filter Out Nodes Having Credentials  
  - Extract Credentials  
  - Remove Duplicate Credentials by Name  
  - Generate Credential Options  
  - Map Credentials  
  - Get Missing Credentials  
  - Get Selected Credentials  
  - Sticky Note17, Sticky Note18, Sticky Note19, Sticky Note20, Sticky Note21

- **Node Details:**

  - **Split Out Nodes**  
    - *Type:* Split Out  
    - *Role:* Splits the workflow nodes array into individual items  
    - *Input:* Merged workflow data from Merge node  
    - *Output:* Each workflow node as separate item  
    - *Edge cases:* Empty nodes array

  - **Filter Out Nodes Having Credentials**  
    - *Type:* Filter  
    - *Role:* Filters nodes that have defined credentials  
    - *Key Config:* Checks if `credentials` property exists on node JSON  
    - *Input:* Output from Split Out Nodes  
    - *Output:* Nodes with credentials only  
    - *Edge cases:* Nodes without credentials filtered out

  - **Extract Credentials**  
    - *Type:* Set  
    - *Role:* Extracts the credential type, name, and ID from each node's credentials object  
    - *Key Config:* Uses the first key in the credentials object (usually credential type) to extract info  
    - *Input:* Filtered nodes with credentials  
    - *Output:* JSON with `type`, `name`, and `id` fields per credential  
    - *Edge cases:* Nodes with multiple credentials per node may not be fully captured if only first key is taken.

  - **Remove Duplicate Credentials by Name**  
    - *Type:* Remove Duplicates  
    - *Role:* Deduplicates the list of credentials by their name field  
    - *Key Config:* Compares `name` field only  
    - *Input:* Extracted credential list  
    - *Output:* Unique credentials by name  
    - *Edge cases:* Different credentials with same name could be merged erroneously.

  - **Generate Credential Options**  
    - *Type:* Code  
    - *Role:* Prepares dropdown options for each credential to be mapped by the user  
    - *Key Config:* For each credential type, lists all existing credentials of same type plus "[create new]" option  
    - *Input:* Unique credentials and all local credentials from Merge node  
    - *Output:* JSON string representing form fields for Map Credentials form  
    - *Edge cases:* No existing credentials of given type means only "[create new]" option.

  - **Map Credentials**  
    - *Type:* Form  
    - *Role:* Presents a multi-form for the user to map each original credential to an existing or new credential  
    - *Key Config:* Dynamic form fields from Generate Credential Options  
    - *Input:* Output from Generate Credential Options  
    - *Output:* User selections for each credential mapping  
    - *Edge cases:* User selects invalid or no mapping; form validation required.

  - **Get Missing Credentials**  
    - *Type:* Code  
    - *Role:* Identifies which credentials were set to create new and prepares data for creating empty credentials  
    - *Input:* Unique credential list and mapping selections from Map Credentials  
    - *Output:* Array of new credential objects with type, name (with trailing "⚠️"), and initial data (empty or OAuth placeholders)  
    - *Edge cases:* OAuth credentials require clientId/clientSecret placeholders.

  - **Get Selected Credentials**  
    - *Type:* Code  
    - *Role:* Maps the user selections for credentials that already exist to their credential IDs  
    - *Input:* Unique credentials, Map Credentials selections, and allCredentials from Merge  
    - *Output:* Array of credentials with oldName, new name, type, and credential ID for replacement  
    - *Edge cases:* Selected credential name might not be found in allCredentials.

---

#### 1.5 Credential Creation and Workflow Update

- **Overview:**  
  Creates new empty credentials for all mappings set to "[create new]", adds original old names for reference, then updates the workflow nodes to use the new credential IDs and names.

- **Nodes Involved:**  
  - Create Empty Credentials  
  - Add Old Names  
  - Collect Credentials to Replace  
  - Replace Credentials in Workflow  
  - Sticky Note21, Sticky Note22

- **Node Details:**

  - **Create Empty Credentials**  
    - *Type:* n8n API node  
    - *Role:* Creates new empty credential entries on the current instance for each new credential required  
    - *Key Config:* Uses the credential type and the original name as base, credentials created with empty data  
    - *Input:* Output from Get Missing Credentials  
    - *Output:* Created credential data including IDs  
    - *Credentials:* Uses saved n8n API credentials  
    - *Edge cases:* API failures, credential creation limits.

  - **Add Old Names**  
    - *Type:* Set  
    - *Role:* Adds the original credential name (without warning emoji) as `oldName` for reference in replacement  
    - *Input:* Output from Create Empty Credentials  
    - *Output:* Credential data with `oldName` field added  
    - *Edge cases:* Name normalization required.

  - **Collect Credentials to Replace**  
    - *Type:* Merge  
    - *Role:* Merges mapped existing credentials and newly created credentials into one list for workflow update  
    - *Input:* From Get Selected Credentials and Add Old Names  
    - *Output:* Combined list of credentials to replace  
    - *Edge cases:* Duplicate entries or missing data.

  - **Replace Credentials in Workflow**  
    - *Type:* Code  
    - *Role:* Iterates over workflow nodes and updates credential IDs and names based on mapping  
    - *Input:* Workflow data from Merge and credential list from Collect Credentials to Replace  
    - *Output:* Updated workflow JSON with new credential references  
    - *Edge cases:* Nodes with multiple credential types.

---

#### 1.6 Workflow Creation and Completion Feedback

- **Overview:**  
  Creates the updated workflow on the current instance using the n8n API and provides the user with success or error feedback via completion forms.

- **Nodes Involved:**  
  - Create Workflow  
  - Success  
  - Error  
  - Error1  
  - Sticky Note23, Sticky Note24

- **Node Details:**

  - **Create Workflow**  
    - *Type:* n8n API node  
    - *Role:* Creates the updated workflow on the current instance using the API  
    - *Key Config:* Sends the entire updated workflow JSON object  
    - *Input:* Output from Replace Credentials in Workflow  
    - *Output:* Success or error response  
    - *Credentials:* Uses saved n8n API credentials  
    - *Edge cases:* API errors, workflow conflicts, invalid data.

  - **Success**  
    - *Type:* Form (completion)  
    - *Role:* Displays success message with optional warning if some credentials need manual population (indicated by emoji)  
    - *Input:* Successful Create Workflow output  
    - *Output:* HTTP response to user  
    - *Edge cases:* None

  - **Error**  
    - *Type:* Form (completion)  
    - *Role:* Displays error message if workflow retrieval from remote instance failed  
    - *Input:* Error from Get Workflows node  
    - *Output:* HTTP response to user

  - **Error1**  
    - *Type:* Form (completion)  
    - *Role:* Displays error message if workflow creation failed  
    - *Input:* Error from Create Workflow node  
    - *Output:* HTTP response to user

---

### 3. Summary Table

| Node Name                       | Node Type               | Functional Role                                         | Input Node(s)                          | Output Node(s)                     | Sticky Note                                                                                       |
|--------------------------------|-------------------------|---------------------------------------------------------|--------------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger            | Entry point: Collect workflow source choice             | —                                    | Determine Workflow Source          | A form which collects the source option. *Consider securing the form using Basic Auth.*         |
| Determine Workflow Source       | If                      | Routes based on source: File Upload or Remote Instance   | On form submission                   | Upload File, Settings             | Switch between available options                                                                |
| Upload File                    | Form                    | Upload workflow JSON file                                | Determine Workflow Source (true)     | Extract from File                 | Choose an n8n workflow file                                                                     |
| Settings                      | Set                     | Defines remote instances configuration                   | Determine Workflow Source (false)    | Generate Instance Options         | Setup instances: each requires name, apiKey, baseURL                                           |
| Generate Instance Options      | Code                    | Prepares dropdown options for remote instances           | Settings                            | Choose Instance                  | Prepare a list of options for the next form                                                     |
| Choose Instance               | Form                    | Select a remote instance to import workflow from         | Generate Instance Options             | Prepare Request Data             | Select source instance                                                                          |
| Prepare Request Data          | Code                    | Maps selected instance config                             | Choose Instance                     | Get Workflows                   | Map Settings to selected instance and retrieve workflows                                        |
| Get Workflows                 | HTTP Request            | Retrieves workflows list from remote instance            | Prepare Request Data                | Split Out Workflows, Error       | Retrieve workflows list                                                                        |
| Split Out Workflows           | Split Out               | Splits workflows array into individual items             | Get Workflows                      | Sort by updatedAt DESC           |                                                                                                 |
| Sort by updatedAt DESC        | Sort                    | Sort workflows by last update descending                  | Split Out Workflows                | Get Workflow Names               |                                                                                                 |
| Get Workflow Names            | Code                    | Creates dropdown options from workflow names              | Sort by updatedAt DESC             | Choose Workflow                 | Prepare a list of options for the next form                                                     |
| Choose Workflow               | Form                    | Select specific workflow from remote list                 | Get Workflow Names                 | Get Selected Workflow           | Let the user choose a workflow from a list                                                     |
| Get Selected Workflow         | Code                    | Extracts selected workflow object                          | Choose Workflow                   | Export Credentials, No Operation | Convert the retrieved workflow into final JSON format                                          |
| Extract from File             | Extract From File       | Converts uploaded workflow file to JSON                    | Upload File                      | Export Credentials, No Operation |                                                                                                 |
| Export Credentials            | Execute Command         | Exports all decrypted local credentials to file           | Extract from File, Get Selected Workflow | Get Credentials Data            | Retrieve all credentials from this instance and convert data                                   |
| Get Credentials Data          | Read Write File         | Reads credentials JSON file                                | Export Credentials               | Binary to JSON                 |                                                                                                 |
| Binary to JSON               | Extract From File       | Converts credentials file binary to JSON                    | Get Credentials Data             | Rename Keys                   |                                                                                                 |
| Rename Keys                  | Rename Keys             | Renames 'data' key to 'allCredentials'                      | Binary to JSON                   | Merge                        |                                                                                                 |
| No Operation                 | NoOp                    | Placeholder to unify branches                              | Extract from File, Get Selected Workflow | Merge                        |                                                                                                 |
| Merge                       | Merge                   | Combines workflow and credentials JSON                     | Rename Keys, No Operation          | Split Out Nodes             | Combine the workflow and credential data to one item                                           |
| Split Out Nodes             | Split Out               | Splits workflow nodes array into individual nodes          | Merge                           | Filter Out Nodes Having Credentials |                                                                                                 |
| Filter Out Nodes Having Credentials | Filter             | Filters nodes that contain credentials                      | Split Out Nodes                 | Extract Credentials           |                                                                                                 |
| Extract Credentials          | Set                     | Extracts credential type, name, and ID from workflow nodes | Filter Out Nodes Having Credentials | Remove Duplicate Credentials by Name | Extract a list of all credentials from the workflow                                            |
| Remove Duplicate Credentials by Name | Remove Duplicates | Removes duplicate credentials by name                      | Extract Credentials             | Generate Credential Options    |                                                                                                 |
| Generate Credential Options  | Code                    | Generates dropdown options for credential mapping           | Remove Duplicate Credentials by Name | Map Credentials             | Prepare a list of options for the next form                                                    |
| Map Credentials             | Form                    | Presents form for user to map original credentials          | Generate Credential Options      | Get Missing Credentials, Get Selected Credentials | Let the user map every credential or create new ones                                           |
| Get Missing Credentials     | Code                    | Identifies credentials to create new empty entries          | Map Credentials, Remove Duplicate Credentials by Name | Create Empty Credentials      |                                                                                                 |
| Create Empty Credentials    | n8n API Node            | Creates new empty credentials on current instance           | Get Missing Credentials          | Add Old Names                | Create empty credentials where needed                                                         |
| Add Old Names              | Set                     | Adds original credential names for reference                | Create Empty Credentials         | Collect Credentials to Replace |                                                                                                 |
| Get Selected Credentials    | Code                    | Maps existing credential selections to IDs                   | Map Credentials, Remove Duplicate Credentials by Name, Merge | Collect Credentials to Replace |                                                                                                 |
| Collect Credentials to Replace | Merge                 | Combines new and existing credentials for workflow update    | Get Selected Credentials, Add Old Names | Replace Credentials in Workflow |                                                                                                 |
| Replace Credentials in Workflow | Code                 | Updates workflow nodes with new credential IDs and names      | Collect Credentials to Replace   | Create Workflow              |                                                                                                 |
| Create Workflow            | n8n API Node            | Creates the updated workflow on current instance             | Replace Credentials in Workflow  | Success, Error1              | Create the updated workflow on this instance                                                  |
| Success                   | Form (completion)        | Displays import success message                               | Create Workflow                 | —                            | Provide feedback to the user whether the process was successful                                |
| Error                     | Form (completion)        | Displays error if retrieving workflows from remote failed    | Get Workflows (error output)    | —                            |                                                                                                 |
| Error1                    | Form (completion)        | Displays error if workflow creation failed                     | Create Workflow (error output)  | —                            |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Form Node**  
   - Create a **Form Trigger** node named `On form submission`.  
   - Configure with a dropdown field labeled "Source" with options: "File Upload" and "Remote Instance".  
   - Set button label to "Continue".

2. **Add an If Node for Source Determination**  
   - Create an **If** node named `Determine Workflow Source`.  
   - Condition: Check if `{{$json.Source}} == "File Upload"`.  
   - Connect `On form submission` to this node.

3. **Setup File Upload Branch**  
   - Add a **Form** node named `Upload File` triggered if the source is "File Upload".  
   - Configure with a required file input field accepting `.json`.  
   - Connect `Determine Workflow Source` (true output) to `Upload File`.

4. **Process Uploaded File**  
   - Add an **Extract From File** node named `Extract from File` to convert uploaded file binary to JSON.  
   - Configure operation "fromJson", destination key `workflowData`, binary property name `Workflow_File`.  
   - Connect `Upload File` to `Extract from File`.

5. **Setup Remote Instance Branch**  
   - Add a **Set** node named `Settings` to store remote instances as an array with objects containing `name`, `apiKey`, and `baseUrl`.  
   - Connect `Determine Workflow Source` (false output) to `Settings`.

6. **Generate Dropdown for Instances**  
   - Add a **Code** node named `Generate Instance Options`.  
   - Code: Loop over `remoteInstances` from `Settings` and create dropdown options with their names.  
   - Connect `Settings` to `Generate Instance Options`.

7. **Form for Selecting Remote Instance**  
   - Add a **Form** node named `Choose Instance`.  
   - Configure dropdown field "Source" with options from `Generate Instance Options` output.  
   - Connect `Generate Instance Options` to `Choose Instance`.

8. **Prepare API Request Data**  
   - Add a **Code** node named `Prepare Request Data`.  
   - Code: Find the instance object in `remoteInstances` that matches the selected instance name.  
   - Connect `Choose Instance` to `Prepare Request Data`.

9. **Retrieve Workflows from Remote Instance**  
   - Add an **HTTP Request** node named `Get Workflows`.  
   - Configure URL to `={{ $json.instance.baseUrl }}/workflows`.  
   - Add header `X-N8N-API-KEY` with value `={{ $json.instance.apiKey }}`.  
   - Send body with parameter `limit=250`.  
   - Connect `Prepare Request Data` to `Get Workflows`.  
   - Set error output to connect to an **Error** form node (see below).

10. **Process Retrieved Workflows**  
    - Add a **Split Out** node `Split Out Workflows` on `data` field.  
    - Connect `Get Workflows` to `Split Out Workflows`.  
    - Add a **Sort** node `Sort by updatedAt DESC` to sort workflows descending by `updatedAt`.  
    - Connect `Split Out Workflows` to `Sort by updatedAt DESC`.  
    - Add a **Code** node `Get Workflow Names` to create dropdown options from sorted workflows.  
    - Connect `Sort by updatedAt DESC` to `Get Workflow Names`.  
    - Add a **Form** node `Choose Workflow` with dropdown field populated from `Get Workflow Names`.  
    - Connect `Get Workflow Names` to `Choose Workflow`.

11. **Extract Selected Workflow Object**  
    - Add a **Code** node `Get Selected Workflow`.  
    - Code: Find workflow object from all workflows matching selected workflow name.  
    - Connect `Choose Workflow` to `Get Selected Workflow`.

12. **Unify File Upload and Remote Workflow Branches**  
    - Add a **No Operation** node `No Operation`.  
    - Connect `Extract from File` and `Get Selected Workflow` outputs to `No Operation`.

13. **Export Credentials from Current Instance**  
    - Add an **Execute Command** node `Export Credentials` running `n8n export:credentials --all --pretty --decrypted --output=/tmp/cred`.  
    - Connect both `Extract from File` and `Get Selected Workflow` outputs to `Export Credentials`.

14. **Read and Parse Credentials JSON**  
    - Add a **Read Write File** node `Get Credentials Data` to read `/tmp/cred`.  
    - Connect `Export Credentials` to `Get Credentials Data`.  
    - Add an **Extract From File** node `Binary to JSON` to convert to JSON.  
    - Connect `Get Credentials Data` to `Binary to JSON`.  
    - Add a **Rename Keys** node `Rename Keys` to rename `data` key to `allCredentials`.  
    - Connect `Binary to JSON` to `Rename Keys`.

15. **Merge Workflow and Credentials Data**  
    - Add a **Merge** node `Merge` set to combine mode by position with unpaired included.  
    - Connect `Rename Keys` and `No Operation` to `Merge`.

16. **Extract Credentials Used in Workflow**  
    - Add a **Split Out** node `Split Out Nodes` on `workflowData.nodes`.  
    - Connect `Merge` to `Split Out Nodes`.  
    - Add a **Filter** node `Filter Out Nodes Having Credentials` checking if `credentials` exists on each node.  
    - Connect `Split Out Nodes` to `Filter Out Nodes Having Credentials`.  
    - Add a **Set** node `Extract Credentials` extracting first credential type, name, and id from each node's credentials object.  
    - Connect `Filter Out Nodes Having Credentials` to `Extract Credentials`.  
    - Add a **Remove Duplicates** node `Remove Duplicate Credentials by Name` comparing on `name`.  
    - Connect `Extract Credentials` to `Remove Duplicate Credentials by Name`.

17. **Generate Credential Mapping Options**  
    - Add a **Code** node `Generate Credential Options` that for each unique credential creates a dropdown with existing credentials of the same type plus "[create new]".  
    - Connect `Remove Duplicate Credentials by Name` and `Merge` to `Generate Credential Options`.  
    - Add a **Form** node `Map Credentials` using dynamic form fields from `Generate Credential Options`.  
    - Connect `Generate Credential Options` to `Map Credentials`.

18. **Identify Missing and Selected Credentials**  
    - Add a **Code** node `Get Missing Credentials` identifying credentials mapped to "[create new]" and preparing empty data templates.  
    - Connect `Map Credentials` and `Remove Duplicate Credentials by Name` to `Get Missing Credentials`.  
    - Add a **Code** node `Get Selected Credentials` mapping existing credentials selections to IDs.  
    - Connect `Map Credentials`, `Remove Duplicate Credentials by Name`, and `Merge` to `Get Selected Credentials`.

19. **Create New Empty Credentials**  
    - Add an **n8n API** node `Create Empty Credentials` to create credentials on instance with empty data for each new credential.  
    - Connect `Get Missing Credentials` to `Create Empty Credentials`.  
    - Use valid n8n API credentials.

20. **Add Old Names to New Credentials**  
    - Add a **Set** node `Add Old Names` to add original credential names (without emoji) for reference.  
    - Connect `Create Empty Credentials` to `Add Old Names`.

21. **Combine All Credentials for Replacement**  
    - Add a **Merge** node `Collect Credentials to Replace`.  
    - Connect `Get Selected Credentials` and `Add Old Names` to this merge node.

22. **Replace Credentials in Workflow**  
    - Add a **Code** node `Replace Credentials in Workflow`.  
    - Code: Loops over workflow nodes, replacing old credential names with new IDs and names.  
    - Connect `Collect Credentials to Replace` to `Replace Credentials in Workflow`.  
    - Connect `Merge` (workflow data) also as input.

23. **Create Updated Workflow on Current Instance**  
    - Add an **n8n API** node `Create Workflow`.  
    - Configure to create a workflow using updated JSON from `Replace Credentials in Workflow`.  
    - Use valid n8n API credentials.  
    - Connect `Replace Credentials in Workflow` to `Create Workflow`.

24. **Completion Forms for Feedback**  
    - Add a **Form (completion)** node `Success` showing success message and instructions to update new credential entries with emoji.  
    - Connect `Create Workflow` success output to `Success`.  
    - Add two **Form (completion)** nodes `Error` and `Error1` to handle errors from `Get Workflows` and `Create Workflow` respectively.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                               | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The workflow is primarily designed for testing/development environments. For production, n8n Enterprise edition is recommended for reliable workflow deployment between environments.                                                                                                       | Disclaimer in workflow description                                                                     |
| Security note: Sensitive credentials and API keys are decrypted and processed within this workflow. Exercise caution when using in shared or insecure environments.                                                                                                                          | Disclaimer in workflow description                                                                     |
| Configuration example for remote instances: an array of objects each with keys `name`, `apiKey`, and `baseUrl`.                                                                                                                                                                            | Sticky Note2 on the canvas                                                                              |
| Form URLs are exposed from the first node and can be shared for user interaction.                                                                                                                                                                                                          | How to use section in description                                                                       |
| The mapping UI allows creating new credentials on the fly; new credentials are named with a trailing "⚠️" emoji to indicate they need manual configuration after creation.                                                                                                               | Mapping credentials block, Get Missing Credentials node, and Success form completion message             |
| Remote API requests require the remote instance to have a valid API key and an accessible base URL with `/workflows` endpoint.                                                                                                                                                            | HTTP Request node configuration                                                                         |
| The workflow uses the `n8n export:credentials` CLI command to export decrypted credentials; ensure n8n user has appropriate permissions and `/tmp/cred` path is writable and readable.                                                                                                       | Export Credentials node                                                                                  |
| The workflow uses n8n API credentials stored in the node named `n8n account` for creating workflows and credentials on the current instance.                                                                                                                                              | Credential usage in Create Workflow and Create Empty Credentials nodes                                   |
| The multi-form approach guides the user step-by-step to prevent misconfiguration and to provide clear UI for mapping credentials, reducing errors in importing workflows between instances.                                                                                               | Overall workflow design                                                                                  |
| For extended automation, consider wrapping this workflow in secure environment or restricting access to the form URLs.                                                                                                                                                                     | Security consideration                                                                                   |
| Useful links and visuals provided in the source documentation: ![Image1](https://i.imgur.com/35UFNRa.png), ![Image2](https://i.imgur.com/MnWtCFJ.png)                                                                                                                                     | Workflow documentation images                                                                           |

---

This detailed documentation fully describes the **Workflow Importer** n8n workflow to allow both human users and automation agents to understand, reproduce, modify, and troubleshoot the process.