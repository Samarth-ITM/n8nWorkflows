Supabase Storage Tutorial: Upload, Fetch, Sign & List Files

https://n8nworkflows.xyz/workflows/supabase-storage-tutorial--upload--fetch--sign---list-files-8228


# Supabase Storage Tutorial: Upload, Fetch, Sign & List Files

### 1. Workflow Overview

This workflow, titled **"Supabase Storage Tutorial: Upload, Fetch, Sign & List Files"**, demonstrates fundamental operations with Supabase Storage integrated in n8n. It serves as a practical learning tool for developers and teams to automate file storage tasks using Supabase's storage API.

The workflow logically divides into four functional blocks representing core Supabase Storage operations:

- **1.1 Upload File**: Receives a file via an n8n form and uploads it to a Supabase storage bucket.
- **1.2 Fetch File**: Retrieves a stored file from Supabase by filename via a user form submission.
- **1.3 Generate Temporary Signed URL**: Creates a temporary signed URL to provide secure time-limited access to a file.
- **1.4 List Files**: Lists files currently stored in the Supabase bucket, triggered manually.

Each block uses HTTP Request nodes configured with Supabase API credentials to interact with the Supabase Storage REST API endpoints. The workflow also includes form triggers to initiate user interactions and sticky notes for instructional context.

---

### 2. Block-by-Block Analysis

#### 1.1 Upload File to Supabase Storage

- **Overview:**  
  This block allows users to upload a file through an n8n form. The file is then uploaded to the configured Supabase storage bucket using a POST HTTP request with binary data.

- **Nodes Involved:**  
  - On form submission (formTrigger)  
  - upload_to_supabase_storage (httpRequest)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Receives user file input via an n8n web form titled "Supabase Storage File Upload".  
    - Config: Single file upload field labeled "File". The form button labeled "Upload to Supabase Storage".  
    - Inputs: Trigger node, no inputs.  
    - Outputs: Connected to `upload_to_supabase_storage`.  
    - Edge Cases: File size limits or unsupported formats depend on Supabase bucket policies; no explicit validation here.  
    - Credentials: None required.  

  - **upload_to_supabase_storage**  
    - Type: HTTP Request  
    - Role: Uploads received binary file to Supabase storage bucket `test-n8n`.  
    - Config:  
      - Method: POST  
      - URL: Constructed dynamically using the bucket name and filename `https://uptttiuxuaacxrdofqgm.supabase.co/storage/v1/object/test-n8n/{{ $binary.File.fileName }}`  
      - Body: Binary data from the uploaded file (`inputDataFieldName` set to "File").  
      - Content-Type: binaryData (appropriate for file upload)  
      - Authentication: Supabase API credential (using anon key and project URL).  
    - Inputs: Receives binary file from form trigger.  
    - Outputs: Response from Supabase API (upload success or failure).  
    - Edge Cases: Network errors, authentication failures, invalid bucket or filename, large file uploads timing out.  
    - Version: Type version 4.2.

---

#### 1.2 Fetch File from Supabase Storage

- **Overview:**  
  Retrieves a file from Supabase storage by filename provided via a form. The file is fetched via a GET HTTP request to the Supabase storage object URL.

- **Nodes Involved:**  
  - On form submission1 (formTrigger)  
  - fetch_file_to_review1 (httpRequest)

- **Node Details:**

  - **On form submission1**  
    - Type: Form Trigger  
    - Role: Accepts a filename input from the user to fetch that file from storage.  
    - Config: Single text input labeled "File Name".  
    - Inputs: Trigger node, no inputs.  
    - Outputs: Connected to `fetch_file_to_review1`.  
    - Edge Cases: Empty or invalid filename input.  
    - Credentials: None.  

  - **fetch_file_to_review1**  
    - Type: HTTP Request  
    - Role: Fetches the file from Supabase storage bucket `test-n8n` using HTTP GET.  
    - Config:  
      - Method: GET (default)  
      - URL: `https://uptttiuxuaacxrdofqgm.supabase.co/storage/v1/object/test-n8n/{{ $json['File Name'] }}` dynamically using the form input.  
      - Authentication: Supabase API credential.  
    - Inputs: Filename JSON from form trigger.  
    - Outputs: File content or error response.  
    - Edge Cases: File not found, invalid filename, authentication issues, network timeouts.  
    - Version: 4.2.

---

#### 1.3 Generate Temporary Signed URL for File Access

- **Overview:**  
  Generates a signed URL that provides temporary access to a file in Supabase storage, valid for a specified expiry time (3600 seconds).

- **Nodes Involved:**  
  - On form submission2 (formTrigger)  
  - get_sign_file_for_temp_access (httpRequest)

- **Node Details:**

  - **On form submission2**  
    - Type: Form Trigger  
    - Role: Receives filename input to create a signed URL for that file.  
    - Config: Single text input labeled "File Name".  
    - Inputs: Trigger node, no inputs.  
    - Outputs: Connected to `get_sign_file_for_temp_access`.  
    - Edge Cases: Empty filename, invalid names.  
    - Credentials: None.  

  - **get_sign_file_for_temp_access**  
    - Type: HTTP Request  
    - Role: Calls Supabase API to generate a signed URL for temporary access to the specified file.  
    - Config:  
      - Method: POST  
      - URL: `https://uptttiuxuaacxrdofqgm.supabase.co/storage/v1/object/sign/test-n8n/{{ $json['File Name'] }}`  
      - Body: JSON `{ "expiresIn": 3600 }` specifying 1-hour expiry.  
      - Authentication: Supabase API credential.  
    - Inputs: Filename from form trigger JSON.  
    - Outputs: JSON response containing signed URL or error details.  
    - Edge Cases: File not found, permission issues, exceeding maximum expiry limits, auth errors.  
    - Version: 4.2.

---

#### 1.4 List All Files in Supabase Storage Bucket

- **Overview:**  
  Provides a list of files stored in the Supabase bucket `test-n8n`, sorted by filename ascending, limited to 20 entries.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (manualTrigger)  
  - list_all_the_object (httpRequest)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the listing process on manual execution.  
    - Inputs: None.  
    - Outputs: Connected to `list_all_the_object`.  
    - Edge Cases: None specific; manual trigger.  

  - **list_all_the_object**  
    - Type: HTTP Request  
    - Role: Calls Supabase storage API to list objects in the bucket.  
    - Config:  
      - Method: POST  
      - URL: `https://uptttiuxuaacxrdofqgm.supabase.co/storage/v1/object/list/test-n8n`  
      - Body: JSON specifying:  
        ```json
        {
          "prefix": "",
          "limit": 20,
          "offset": 0,
          "sortBy": {
            "column": "name",
            "order": "asc"
          }
        }
        ```  
      - Authentication: Supabase API credential.  
    - Inputs: Manual trigger.  
    - Outputs: JSON list of objects in bucket.  
    - Edge Cases: Empty bucket, network errors, API limits.  
    - Version: 4.2.

---

#### Sticky Notes (Instructional and Contextual)

Multiple sticky notes provide instructional content, setup guidance, and visual references for:

- Creating Supabase bucket and policies  
- Getting API keys and project URL  
- Setting up Supabase API credentials in n8n  
- Overview of lessons included in the workflow  
- General project introduction and prerequisites

These are not directly connected to nodes but essential for proper configuration and understanding.

---

### 3. Summary Table

| Node Name                  | Node Type         | Functional Role                               | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                                                |
|----------------------------|-------------------|-----------------------------------------------|------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger      | Receive file upload input                      | None                   | upload_to_supabase_storage   | Lesson 1  - Upload file to storage                                                                                                         |
| upload_to_supabase_storage | HTTP Request      | Upload binary file to Supabase storage bucket | On form submission     | None                        | Lesson 1  - Upload file to storage                                                                                                         |
| On form submission1        | Form Trigger      | Receive filename input to fetch file           | None                   | fetch_file_to_review1        | Lesson 2  - Fetch file from storage                                                                                                        |
| fetch_file_to_review1       | HTTP Request      | Fetch file content from Supabase storage       | On form submission1    | None                        | Lesson 2  - Fetch file from storage                                                                                                        |
| On form submission2        | Form Trigger      | Receive filename input for signed URL           | None                   | get_sign_file_for_temp_access | Lesson 3  - Create Temp document with expire time                                                                                          |
| get_sign_file_for_temp_access| HTTP Request    | Generate signed URL with expiry for file access | On form submission2    | None                        | Lesson 3  - Create Temp document with expire time                                                                                          |
| When clicking ‘Execute workflow’ | Manual Trigger | Trigger listing of stored files                 | None                   | list_all_the_object          | Lesson 4  - Fetch list all the items in storage                                                                                           |
| list_all_the_object        | HTTP Request      | List files in Supabase bucket                   | When clicking ‘Execute workflow’ | None                  | Lesson 4  - Fetch list all the items in storage                                                                                           |
| Sticky Note                | Sticky Note       | Instructional context on access policy          | None                   | None                        | Update/create policy to access that. (note based on key we have create policy) [image link in note]                                        |
| Sticky Note1               | Sticky Note       | Lesson 1 header                                  | None                   | None                        | Lesson 1  - Upload file to storage                                                                                                         |
| Sticky Note2               | Sticky Note       | How to get project URL                           | None                   | None                        | Get project URL [image link in note]                                                                                                       |
| Sticky Note3               | Sticky Note       | Create Supabase API credential in n8n           | None                   | None                        | Create a Credential of Supabase API Credential Type [image link in note]                                                                   |
| Sticky Note4               | Sticky Note       | How to get API (Anon) key                        | None                   | None                        | Get API key (Anon) key. [image link in note]                                                                                               |
| Sticky Note5               | Sticky Note       | Create a bucket in Supabase                      | None                   | None                        | Create a bucket in supabase [image link in note]                                                                                           |
| Sticky Note6               | Sticky Note       | Lesson 2 header                                  | None                   | None                        | Lesson 2  - Fetch file from storage                                                                                                        |
| Sticky Note7               | Sticky Note       | Lesson 3 header                                  | None                   | None                        | Lesson 3  - Create Temp document with expire time                                                                                          |
| Sticky Note8               | Sticky Note       | Lesson 4 header                                  | None                   | None                        | Lesson 4  - Fetch list all the items in storage                                                                                            |
| Sticky Note9               | Sticky Note       | Full workflow overview and setup instructions   | None                   | None                        | Learn Supabase Storage Fundamentals with n8n workflow detailed description including setup, requirements, and lessons [full text included] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Supabase API Credential in n8n**  
   - Navigate to Credentials.  
   - Create new credential of type "Supabase API".  
   - Enter your Supabase project URL and Anon API key.  
   - Save credential.

2. **Create Upload File Form Trigger**  
   - Add a `Form Trigger` node named `On form submission`.  
   - Configure:  
     - Form Title: "Supabase Storage File Upload"  
     - Button Label: "Upload to Supabase Storage"  
     - Add a single field: Type `File`, Label `File`, single file only.  
   - No credentials needed.

3. **Create Upload HTTP Request Node**  
   - Add `HTTP Request` node named `upload_to_supabase_storage`.  
   - Method: POST  
   - URL: Use expression:  
     `https://<your-project>.supabase.co/storage/v1/object/test-n8n/{{ $binary.File.fileName }}`  
   - Send Body: true  
   - Content Type: binaryData  
   - Input Data Field Name: `File`  
   - Authentication: Use Supabase API Credential created in step 1.  
   - Connect output of `On form submission` to input of this node.

4. **Create Fetch File Form Trigger**  
   - Add a `Form Trigger` node named `On form submission1`.  
   - Configure:  
     - Form Title: "Get File from Storage"  
     - Add a single text field: Label `File Name`.  
   - No credentials needed.

5. **Create Fetch HTTP Request Node**  
   - Add `HTTP Request` node named `fetch_file_to_review1`.  
   - Method: GET (default)  
   - URL: Expression:  
     `https://<your-project>.supabase.co/storage/v1/object/test-n8n/{{ $json["File Name"] }}`  
   - Authentication: Use Supabase API Credential.  
   - Connect output of `On form submission1` to input of this node.

6. **Create Signed URL Form Trigger**  
   - Add a `Form Trigger` node named `On form submission2`.  
   - Configure:  
     - Form Title: "Get File from Storage"  
     - Add a single text field: Label `File Name`.  
   - No credentials needed.

7. **Create Signed URL Generation HTTP Request Node**  
   - Add `HTTP Request` node named `get_sign_file_for_temp_access`.  
   - Method: POST  
   - URL: Expression:  
     `https://<your-project>.supabase.co/storage/v1/object/sign/test-n8n/{{ $json["File Name"] }}`  
   - Body Content: JSON `{ "expiresIn": 3600 }` (1 hour expiry)  
   - Send Body: true, Specify body as JSON  
   - Authentication: Use Supabase API Credential.  
   - Connect output of `On form submission2` to input of this node.

8. **Create Manual Trigger for Listing Files**  
   - Add `Manual Trigger` node named `When clicking ‘Execute workflow’`.

9. **Create List Files HTTP Request Node**  
   - Add `HTTP Request` node named `list_all_the_object`.  
   - Method: POST  
   - URL:  
     `https://<your-project>.supabase.co/storage/v1/object/list/test-n8n`  
   - Body Content: JSON specifying:  
     ```json
     {
       "prefix": "",
       "limit": 20,
       "offset": 0,
       "sortBy": {
         "column": "name",
         "order": "asc"
       }
     }
     ```  
   - Send Body: true, Specify body as JSON  
   - Authentication: Use Supabase API Credential.  
   - Connect output of `When clicking ‘Execute workflow’` to input of this node.

10. **Configure Bucket and Policies in Supabase**  
    - In the Supabase project, create a storage bucket named `test-n8n`.  
    - Create appropriate policies to allow read/write access via API key (Anon key).  
    - Refer to sticky notes images and Supabase documentation for policy syntax.

11. **Save and Activate the Workflow**  
    - Test each form with appropriate input and verify upload, fetch, signed URL generation, and listing functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Full workflow introduction and setup instructions, including prerequisites and lesson descriptions.                                                                                                                                                                  | [Workflow overview content from Sticky Note9]                                                                                      |
| To access Supabase Storage via API, update/create bucket access policies based on your anon key permissions.                                                                                                                               | ![Policy Image](https://github.com/nextwebspark/n8n-templates/blob/main/supabase-n8n-fundamental/policy.JPG?raw=true)               |
| How to get your Supabase project URL for API requests.                                                                                                                                                                                 | ![Project URL Image](https://github.com/nextwebspark/n8n-templates/blob/main/supabase-n8n-fundamental/api-key.JPG?raw=true)         |
| Create Supabase API Credential in n8n with your Project URL and Anon key.                                                                                                                                                              | ![Credential Setup Image](https://github.com/nextwebspark/n8n-templates/blob/main/supabase-n8n-fundamental/n8n-supabase-key.JPG?raw=true) |
| How to get your Supabase anon API key.                                                                                                                                                                                               | ![Anon Key Image](https://github.com/nextwebspark/n8n-templates/blob/main/supabase-n8n-fundamental/supdabase-url.JPG?raw=true)        |
| Instructions to create a storage bucket in Supabase.                                                                                                                                                                                  | ![Bucket Creation Image](https://github.com/nextwebspark/n8n-templates/blob/main/supabase-n8n-fundamental/create-bucket.JPG?raw=true) |

---

This structured documentation allows developers or AI agents to understand the workflow’s logic, reproduce it entirely in n8n, anticipate error cases, and integrate Supabase Storage effectively.