Upload Files to Dropbox and Generate Direct Download Links

https://n8nworkflows.xyz/workflows/upload-files-to-dropbox-and-generate-direct-download-links-10299


# Upload Files to Dropbox and Generate Direct Download Links

### 1. Workflow Overview

This workflow automates uploading files to Dropbox and generating direct download links for those files. It is designed to be triggered either by a form submission or another workflow and handles the entire process from receiving a file, uploading it to Dropbox, checking for existing shared links, creating new ones if necessary, and converting shared links into direct download URLs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Accepts file uploads via a form or triggers from other workflows.
- **1.2 Upload & Path Processing**: Uploads the received file to Dropbox and normalizes the file path for further API operations.
- **1.3 Sharing Link Management**: Checks for existing shared links on Dropbox for the uploaded file; creates a new shared link if none exists.
- **1.4 Direct Link Processing**: Converts the Dropbox shared link into a direct download link and prepares the final response.
- **1.5 Test & Support**: Includes a test workflow and explanatory notes for setup and usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the file input either from a user form or from another workflow execution, triggering the upload process.

- **Nodes Involved:**  
  - Upload File (Form Trigger)  
  - When Executed by Another Workflow (Execute Workflow Trigger)  
  - Call '[SUB] Dropbox upload links' (Execute Workflow)

- **Node Details:**

  - **Upload File**  
    - Type: Form Trigger  
    - Role: Receives file input from a user upload form with a required file field.  
    - Configuration: A single file field labeled "Upload file", required.  
    - Input: HTTP request with uploaded file.  
    - Output: File data passed downstream.  
    - Failure Types: File upload failures, HTTP errors, missing file field.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Starts the workflow when called by another workflow.  
    - Configuration: Pass-through input source to maintain incoming data.  
    - Input: Data from calling workflow.  
    - Output: Passes input to upload node.  
    - Failure Types: Workflow invocation errors, missing data.

  - **Call '[SUB] Dropbox upload links'**  
    - Type: Execute Workflow  
    - Role: Invokes the sub-workflow that handles upload and link generation.  
    - Configuration: References workflow by ID; no additional inputs mapped.  
    - Input: Data from Upload File node.  
    - Output: Completion marker node.  
    - Failure Types: Sub-workflow invocation errors.

---

#### 2.2 Upload & Path Processing

- **Overview:**  
  Uploads the file to Dropbox using OAuth2 credentials, then normalizes the file path for use with Dropbox API calls.

- **Nodes Involved:**  
  - Upload a file (Dropbox)  
  - Normalize Path1 (Set)  
  - Upload & Path Processing1 (Sticky Note)

- **Node Details:**

  - **Upload a file**  
    - Type: Dropbox node  
    - Role: Uploads binary file data to Dropbox at a defined path.  
    - Configuration:  
      - Path: Dynamic, under `/Automate/N8N/host/` plus the uploaded filename.  
      - Binary property: `"Upload_file"` (from form trigger).  
      - Authentication: OAuth2 via Dropbox OAuth2 credentials.  
    - Input: Binary file data from form or trigger.  
    - Output: JSON metadata about uploaded file including `path_lower`.  
    - Failure Types: OAuth token expiration, network errors, file conflicts if filename not unique.

  - **Normalize Path1**  
    - Type: Set  
    - Role: Prepares the normalized path (`path_lower`) from upload response for API calls.  
    - Configuration: No explicit fields set; passes through or sets `path_lower` as required.  
    - Input: Upload a file node output.  
    - Output: Normalized path for next steps.  
    - Failure Types: Missing path data due to upload failure.

  - **Upload & Path Processing1 (Sticky Note)**  
    - Notes the sequence: reading binary, uploading file, normalizing path.  
    - Provides conceptual grouping but no processing logic.

---

#### 2.3 Sharing Link Management

- **Overview:**  
  Determines whether a shared link already exists for the uploaded file. If not, retrieves an access token and creates a new shared link.

- **Nodes Involved:**  
  - Dropbox: List Shared Links (HTTP Request)  
  - Extract Existing Link (Set)  
  - If: Link Exists? (If)  
  - Get DropBox access token (Data Table)  
  - Dropbox: Create Shared Link (HTTP Request)  
  - Sharing Link Management1 (Sticky Note)

- **Node Details:**

  - **Dropbox: List Shared Links**  
    - Type: HTTP Request  
    - Role: Queries Dropbox API to list any existing shared links for the given path.  
    - Configuration:  
      - URL: Dropbox API endpoint `/2/sharing/list_shared_links`  
      - Method: POST  
      - Body: JSON with `path` from normalized path and `direct_only: true`  
      - Authentication: Dropbox OAuth2 credentials  
    - Input: Normalized path from previous block.  
    - Output: JSON array `links` with existing shared links or empty.  
    - Failure Types: API limits, invalid path, auth errors.

  - **Extract Existing Link**  
    - Type: Set  
    - Role: Extracts first shared link URL from list response for conditional checks.  
    - Configuration: Sets `links[0].url` for downstream conditionals.  
    - Input: List Shared Links output.  
    - Output: Prepared link or empty string.  
    - Failure Types: Empty list leading to missing URL.

  - **If: Link Exists?**  
    - Type: If  
    - Role: Branches workflow depending on whether a shared link already exists (`url` is empty or not).  
    - Configuration: Checks if `links[0].url` is empty string.  
    - Inputs: Extract Existing Link output.  
    - Outputs:  
      - True branch (no existing link): proceeds to get Dropbox token and create link.  
      - False branch (existing link): proceeds to rewrite host and build direct link.  
    - Failure Types: Expression evaluation issues.

  - **Get DropBox access token**  
    - Type: Data Table  
    - Role: Retrieves stored access token from an external data table called `cred-Dropbox`.  
    - Configuration: Retrieves row with ID 1 from the table.  
    - Input: True branch from If node.  
    - Output: Provides token JSON with `token` property.  
    - Failure Types: Missing or invalid token data.

  - **Dropbox: Create Shared Link**  
    - Type: HTTP Request  
    - Role: Creates a new shared link with public visibility for the uploaded file.  
    - Configuration:  
      - URL: Dropbox API endpoint `/2/sharing/create_shared_link_with_settings`  
      - Method: POST  
      - Body: JSON with `path` from normalized path and requested visibility `public`  
      - Authorization header: Bearer token from Data Table node  
    - Input: Access token from previous node.  
    - Output: New shared link data.  
    - Failure Types: Auth errors, rate limits, path errors.

  - **Sharing Link Management1 (Sticky Note)**  
    - Details the logical steps for managing shared links and token usage.  
    - Explains the need for external token storage and access token refresh workflows.

---

#### 2.4 Direct Link Processing

- **Overview:**  
  Converts the Dropbox shared link into a direct download link by rewriting the host part of the URL, then formats the final response for output.

- **Nodes Involved:**  
  - Rewrite Dropbox Host (Set)  
  - Build Direct Link (Set)  
  - Final Response (Set)  
  - Direct Link Processing1 (Sticky Note)

- **Node Details:**

  - **Rewrite Dropbox Host**  
    - Type: Set  
    - Role: Performs any necessary URL host rewrites (placeholder for URL normalization).  
    - Configuration: No explicit fields set; acts as a pass-through or minor modification step.  
    - Input: Existing link from If node false branch or newly created link.  
    - Output: Modified link for direct link building.  
    - Failure Types: Expression errors if input missing.

  - **Build Direct Link**  
    - Type: Set  
    - Role: Converts the Dropbox shared link URL into a direct download URL using regex replacement.  
    - Configuration:  
      - Creates a new string field `shared_link_direct` by replacing the Dropbox shared link domain (`https://www.dropbox.com/`) with `https://dl.dropboxusercontent.com/` for direct downloads.  
    - Input: URL from Rewrite Dropbox Host node.  
    - Output: Direct download URL string.  
    - Failure Types: Regex failures or empty input URL.

  - **Final Response**  
    - Type: Set  
    - Role: Prepares the final response JSON containing the direct download link under fields `response` and `url`.  
    - Configuration: Assigns the direct link string from previous node to output fields.  
    - Input: Direct link string from Build Direct Link node.  
    - Output: Final result ready for return or further use.  
    - Failure Types: Missing input data.

  - **Direct Link Processing1 (Sticky Note)**  
    - Describes the steps for rewriting hosts, building direct links, and sending success responses.

---

#### 2.5 Test & Support

- **Overview:**  
  Contains a test workflow to validate the upload and link generation process and includes detailed notes on setup, credentials, and operational tips.

- **Nodes Involved:**  
  - Upload File (Form Trigger)  
  - Call '[SUB] Dropbox upload links' (Execute Workflow)  
  - Completed (NoOp)  
  - Sticky Note (General instructions)  
  - Sticky Note2 (Test workflow instructions)

- **Node Details:**

  - **Completed**  
    - Type: NoOp  
    - Role: Marks successful completion of the called sub-workflow.  
    - Configuration: No parameters.  
    - Input: Final output of sub-workflow call.  
    - Output: None.  
    - Failure Types: None.

  - **Sticky Note (Large)**  
    - Contains detailed instructions on OAuth2 credential setup, Dropbox app creation, token refresh workflows, and advice on filename uniqueness to avoid link conflicts.  
    - Includes a link to Dropbox app creation page: https://www.dropbox.com/developers/apps  

  - **Sticky Note2**  
    - Advises copying the test workflow to isolate testing and to ensure proper sub-workflow linking.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                          | Input Node(s)                        | Output Node(s)                    | Sticky Note                                                                                              |
|-------------------------------|--------------------------------|----------------------------------------|------------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------|
| Upload File                   | Form Trigger                   | Captures file upload from user         | —                                  | Call '[SUB] Dropbox upload links' | Test workflow instructions: copy and link sub-workflows to run test.                                   |
| Call '[SUB] Dropbox upload links' | Execute Workflow              | Invokes sub-workflow for upload & link | Upload File                       | Completed                        |                                                                                                         |
| Completed                    | NoOp                          | Marks sub-workflow completion           | Call '[SUB] Dropbox upload links'  | —                                |                                                                                                         |
| When Executed by Another Workflow | Execute Workflow Trigger      | Starts workflow from another workflow   | —                                  | Upload a file                   |                                                                                                         |
| Upload a file                | Dropbox                       | Uploads file to Dropbox                  | When Executed by Another Workflow   | Normalize Path1                 | Upload & Path Processing block: reads file and uploads to Dropbox.                                      |
| Normalize Path1              | Set                          | Normalizes Dropbox file path             | Upload a file                     | Dropbox: List Shared Links       |                                                                                                         |
| Dropbox: List Shared Links    | HTTP Request                 | Lists existing shared links on Dropbox  | Normalize Path1                   | Extract Existing Link            | Sharing Link Management block: checks for existing shared links.                                        |
| Extract Existing Link         | Set                          | Extracts first shared link URL           | Dropbox: List Shared Links         | If: Link Exists?                |                                                                                                         |
| If: Link Exists?             | If                           | Branches based on link existence         | Extract Existing Link             | Get DropBox access token (true branch), Rewrite Dropbox Host (false branch) |                                                                                                         |
| Get DropBox access token      | Data Table                   | Retrieves stored Dropbox access token    | If: Link Exists? (true branch)     | Dropbox: Create Shared Link     |                                                                                                         |
| Dropbox: Create Shared Link   | HTTP Request                 | Creates a new shared link if needed      | Get DropBox access token          | Dropbox: List Shared Links       |                                                                                                         |
| Rewrite Dropbox Host          | Set                          | Rewrites Dropbox URL host for direct link | If: Link Exists? (false branch)   | Build Direct Link               | Direct Link Processing block: prepares URL for direct download.                                         |
| Build Direct Link             | Set                          | Converts shared link to direct download URL | Rewrite Dropbox Host             | Final Response                 |                                                                                                         |
| Final Response               | Set                          | Formats final response with direct link  | Build Direct Link                 | Call '[SUB] Dropbox upload links' (for sub-workflow) |                                                                                                         |
| Upload & Path Processing1     | Sticky Note                  | Notes on upload and path normalization   | —                                  | —                                | Describes steps: reading binary, uploading, normalizing path.                                          |
| Sharing Link Management1      | Sticky Note                  | Notes on managing shared links and tokens | —                                  | —                                | Describes flow for shared link management and token storage.                                           |
| Direct Link Processing1       | Sticky Note                  | Notes on rewriting and building direct links | —                                  | —                                | Describes URL rewriting and final response formatting.                                                 |
| Sticky Note                  | Sticky Note                  | Instructions for Dropbox app setup and OAuth2 credentials | —                                  | —                                | Provides setup instructions, token refresh info, and Dropbox app creation link: https://www.dropbox.com/developers/apps |
| Sticky Note2                 | Sticky Note                  | Test workflow instructions               | —                                  | —                                | Advises copying test workflow and linking sub-workflows for testing.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `Upload File`  
   - Set form title: "Upload File To DropBox"  
   - Add a required file field labeled "Upload file".  
   - This node will receive user file uploads.

2. **Create an Execute Workflow node**  
   - Name: `Call '[SUB] Dropbox upload links'`  
   - Configure to call the sub-workflow handling upload and link generation (workflow ID must be set accordingly).  
   - Connect `Upload File` output to this node.

3. **Create a NoOp node**  
   - Name: `Completed`  
   - Connect the output of `Call '[SUB] Dropbox upload links'` to this node to mark completion.

---

**Sub-Workflow: [SUB] Dropbox upload links**

4. **Create an Execute Workflow Trigger node**  
   - Name: `When Executed by Another Workflow`  
   - Set input source to "passthrough" to accept incoming data.

5. **Create a Dropbox node**  
   - Name: `Upload a file`  
   - Set operation to upload a file.  
   - Path: `/Automate/N8N/host/{{ $json['Upload file'][0].filename }}` (dynamic filename from input).  
   - Binary property name: `Upload_file`.  
   - Authenticate using Dropbox OAuth2 credentials (set up OAuth2 with Dropbox app credentials).  
   - Connect `When Executed by Another Workflow` to this node.

6. **Create a Set node**  
   - Name: `Normalize Path1`  
   - Pass through or explicitly assign `path_lower` from upload response for API usage.  
   - Connect from `Upload a file`.

7. **Create an HTTP Request node**  
   - Name: `Dropbox: List Shared Links`  
   - URL: `https://api.dropboxapi.com/2/sharing/list_shared_links`  
   - Method: POST  
   - Body: JSON with `"path": "{{ $json.path_lower }}"` and `"direct_only": true`  
   - Authenticate with Dropbox OAuth2 credentials.  
   - Connect from `Normalize Path1`.

8. **Create a Set node**  
   - Name: `Extract Existing Link`  
   - Extract the first shared link URL from the previous node’s response (`links[0].url`).  
   - Connect from `Dropbox: List Shared Links`.

9. **Create an If node**  
   - Name: `If: Link Exists?`  
   - Condition: Check if `links[0].url` is empty (empty string).  
   - Connect from `Extract Existing Link`.

10. **Create a Data Table node**  
    - Name: `Get DropBox access token`  
    - Operation: Get row from data table `cred-Dropbox` with ID 1 (must exist and contain Dropbox token).  
    - Connect from `If: Link Exists?` true branch (no existing link).

11. **Create an HTTP Request node**  
    - Name: `Dropbox: Create Shared Link`  
    - URL: `https://api.dropboxapi.com/2/sharing/create_shared_link_with_settings`  
    - Method: POST  
    - Body: JSON with path from normalized path, and settings requesting public visibility.  
    - Add Authorization header with Bearer token from the data table node.  
    - Connect from `Get DropBox access token`.

12. **Connect output of `Dropbox: Create Shared Link` back to `Dropbox: List Shared Links`**  
    - This allows refreshing the shared links list with the newly created link.

13. **Create a Set node**  
    - Name: `Rewrite Dropbox Host`  
    - Acts as a pass-through or modifies the existing shared link if needed.  
    - Connect from `If: Link Exists?` false branch (existing link).

14. **Create a Set node**  
    - Name: `Build Direct Link`  
    - Create a field `shared_link_direct` by replacing the shared link domain with `https://dl.dropboxusercontent.com/` to enable direct download.  
    - Expression example:  
      `={{ String($json.links[0].url || '').trim().replace(/^https:\/\/(?:www\.)?dropbox\.com\//, 'https://dl.dropboxusercontent.com/') }}`  
    - Connect from `Rewrite Dropbox Host`.

15. **Create a Set node**  
    - Name: `Final Response`  
    - Assign fields `response` and `url` with the value of `shared_link_direct`.  
    - Connect from `Build Direct Link`.

16. **Connect `Final Response` output to `When Executed by Another Workflow` output end or the calling node**  
    - This completes the sub-workflow returning the direct link.

---

### Credential Setup

- Create a Dropbox app in https://www.dropbox.com/developers/apps with full permissions for "Files and folders" and "Collaboration".
- Configure OAuth2 credentials in n8n with the App Key and Secret.
- Generate initial access and refresh tokens manually for testing uploads.
- Store the access token in a data table named `cred-Dropbox` with a single row (ID=1) for token retrieval.
- Implement or use a separate token refresher workflow to refresh tokens automatically to avoid manual renewal.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| This workflow works on both local and remote installs of n8n. The key to success is ensuring uploaded filenames are unique to avoid conflicts with existing Dropbox shared links.                                                                                                                                                                                                                                                                        | Sticky Note in Upload & Path Processing block                             |
| For Dropbox OAuth2 setup, create an app with full permissions under "Files and folders" and "Collaboration". Set redirect URIs to your n8n instance URL. Generate your first access token manually to test the upload workflow. Use a separate "Refresh Token Workflow" to automate token refreshes and avoid manual token renewal.                                                                                                                               | Dropbox app creation link: https://www.dropbox.com/developers/apps         |
| The sub-workflow is designed to be called either by a form trigger workflow or another workflow. Ensure to link workflows properly when testing or deploying.                                                                                                                                                                                                                                                                                               | Sticky Note2 in Test workflow block                                       |
| The data table `cred-Dropbox` must exist with at least one row containing the Dropbox access token with ID = 1. This token is used to create shared links and must be kept up to date by a token refresh workflow.                                                                                                                                                                                                                                            |                                                                                   |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.