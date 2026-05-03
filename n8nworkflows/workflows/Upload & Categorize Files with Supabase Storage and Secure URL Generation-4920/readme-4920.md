Upload & Categorize Files with Supabase Storage and Secure URL Generation

https://n8nworkflows.xyz/workflows/upload---categorize-files-with-supabase-storage-and-secure-url-generation-4920


# Upload & Categorize Files with Supabase Storage and Secure URL Generation

### 1. Workflow Overview

This workflow automates the process of uploading files to Supabase Storage, categorizing them into buckets based on their MIME type, and generating secure signed URLs for access. It is primarily designed to be invoked by other workflows but also includes a temporary form for quick testing. The workflow handles conversion of base64 encoded file data into binary format, performs the upload via Supabase’s HTTP API with authentication, and returns either a success response with a signed URL or an error response if any step fails.

**Logical Blocks:**

- **1.1 Input Reception & Testing:** Receives input either from another workflow or via a temporary test form.
- **1.2 Data Preparation:** Determines the appropriate storage bucket based on MIME type and prepares the file data.
- **1.3 File Conversion:** Converts base64 encoded file data into binary format for upload.
- **1.4 Upload Process:** Uploads the binary file to Supabase Storage using authenticated HTTP requests.
- **1.5 Signed URL Generation:** Generates a signed URL for secure access to the uploaded file.
- **1.6 Response Handling:** Sends success or error responses depending on upload and signing outcomes.
- **1.7 Documentation & Notes:** Includes multiple sticky notes explaining design decisions, usage instructions, and warnings.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Testing

**Overview:**  
This block handles the initial input of file data either from an external workflow trigger or via a built-in temporary form for testing purposes. It ensures that necessary data fields are available for subsequent processing.

**Nodes Involved:**  
- When Executed by Another Workflow  
- temp form to test workflow  
- Sticky Note1 (contextual documentation)

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point when called by another workflow, receives inputs ‘mime_type’, ‘original_filename’, and ‘binary_data_base64’.  
  - Configuration: Defines expected input parameters for downstream processing.  
  - Connections: Outputs to ‘Prepare Upload Data’.  
  - Edge Cases: Missing or malformed input parameters could cause failures downstream.

- **temp form to test workflow**  
  - Type: Form Trigger  
  - Role: Provides a temporary web form to manually input test data (filename, base64 encoded file, mime type).  
  - Configuration: Webhook path is `action-workflows-testform`; form fields marked required as appropriate.  
  - Connections: Outputs to ‘Prepare Upload Data’.  
  - Edge Cases: Large base64 data may cause performance issues; form intended only for temporary use.

- **Sticky Note1**  
  - Role: Documents the purpose of the test form and usage instructions.  
  - Content: Warns about removing the test form before production and describes rationale for base64 input.

---

#### 1.2 Data Preparation

**Overview:**  
Determines the target storage bucket based on the MIME type of the input file and passes through all other data fields unchanged. This categorization supports organizing files into logical buckets in Supabase Storage.

**Nodes Involved:**  
- Prepare Upload Data  
- Sticky Note3

**Node Details:**

- **Prepare Upload Data**  
  - Type: Set Node  
  - Role: Assigns a `bucket_name` field derived from the input MIME type using JavaScript expressions.  
  - Configuration:  
    - Buckets:  
      - `image-files` for image/* MIME types  
      - `audio-files` for audio/* MIME types  
      - `video-files` for video/* MIME types  
      - `document-files` for all others (default)  
    - Includes all other input fields unchanged.  
  - Expression: Inline JavaScript for MIME type bucket mapping.  
  - Connections: Outputs to ‘Convert to File’.  
  - Edge Cases: Missing or unknown MIME types default to ‘document-files’.

- **Sticky Note3**  
  - Role: Explains the rationale behind data preparation and file conversion steps.

---

#### 1.3 File Conversion

**Overview:**  
Converts the base64 encoded string representing the file into binary data suitable for uploading through the HTTP request.

**Nodes Involved:**  
- Convert to File  
- Sticky Note2

**Node Details:**

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts base64 `binary_data_base64` to binary file data with correct filename and MIME type.  
  - Configuration: File name is set dynamically from `original_filename`, MIME type from `mime_type`.  
  - Source Property: `binary_data_base64` (base64 string).  
  - Connections: Outputs to ‘Upload to Supabase Storage’.  
  - Edge Cases: Incorrect base64 encoding or missing data will cause conversion failure.

- **Sticky Note2**  
  - Role: Discusses pros and cons of using base64 encoding, including increased data size and potential performance impact for large files.

---

#### 1.4 Upload Process

**Overview:**  
Uploads the binary file to Supabase Storage by sending an authenticated HTTP POST request to the appropriate bucket and filename path.

**Nodes Involved:**  
- Upload to Supabase Storage  
- Sticky Note4  
- Upload Error Response

**Node Details:**

- **Upload to Supabase Storage**  
  - Type: HTTP Request  
  - Role: Sends file binary data to Supabase Storage API endpoint for the appropriate bucket and filename.  
  - URL: Dynamic, constructed as `https://api-sb.janwillemaltink.com/storage/v1/object/{bucket_name}/{original_filename}`.  
  - Method: POST  
  - Authentication: Uses stored Supabase API credential (Bearer token).  
  - Content Type: binaryData  
  - Input Data Field: `data` (the binary file data).  
  - Connections:  
    - On success: to ‘Generate Signed Url’  
    - On error: to ‘Upload Error Response’ (continuing workflow despite error).  
  - Edge Cases: Network issues, authentication failures, oversized files, or API errors.  
  - Notes: On error, workflow continues with error response node.

- **Upload Error Response**  
  - Type: Set Node  
  - Role: Sets error response data including action type and status when upload fails.  
  - Connections: Terminal (no outputs).  
  - Edge Cases: Used only if upload returns an error.

- **Sticky Note4**  
  - Role: Explains Supabase Storage authentication model (Bearer tokens, anon/service keys), and importance of signed URLs.

---

#### 1.5 Signed URL Generation

**Overview:**  
Generates a time-limited signed URL for secure access to the uploaded file, avoiding the need to expose credentials on the client side.

**Nodes Involved:**  
- Generate Signed Url  
- add domain  
- Sign Error Response

**Node Details:**

- **Generate Signed Url**  
  - Type: HTTP Request  
  - Role: Requests a signed URL from Supabase Storage API for the uploaded object key.  
  - URL: Dynamic, constructed as `https://api-sb.janwillemaltink.com/storage/v1/object/sign/{Key}` where `Key` is the object path.  
  - Method: POST  
  - Body: JSON with `"expiresIn": 2592000` (30 days).  
  - Authentication: Same Supabase API credentials as upload node.  
  - Connections:  
    - On success: to ‘add domain’  
    - On error: to ‘Sign Error Response’ (continuing workflow despite error).  
  - Edge Cases: Authentication failure, invalid key, or API errors.

- **add domain**  
  - Type: Set Node  
  - Role: Prefixes the relative signed URL path with the domain to produce a full signed URL.  
  - Value: Concatenates base URL `https://api-sb.janwillemaltink.com/storage/v1` with the `signedURL` field from the previous node.  
  - Connections: Outputs to ‘Success Response’.  
  - Edge Cases: Missing or malformed signed URL.

- **Sign Error Response**  
  - Type: Set Node  
  - Role: Sets error response data for failed signed URL generation attempts.  
  - Connections: Terminal (no outputs).

---

#### 1.6 Response Handling

**Overview:**  
Prepares the final success or error response payloads that downstream workflows or clients can consume, indicating upload status and any relevant URLs.

**Nodes Involved:**  
- Success Response  
- Upload Error Response  
- Sign Error Response

**Node Details:**

- **Success Response**  
  - Type: Set Node  
  - Role: Sets action type as ‘s3_upload’ and status as ‘success’, passing through all other fields including the full signed URL.  
  - Connections: Terminal node (no outputs).  
  - Edge Cases: Should only execute on fully successful upload and signing.

- **Upload Error Response** and **Sign Error Response**  
  - Detailed above; set error status and action type accordingly.

---

#### 1.7 Documentation & Notes

**Overview:**  
Various sticky notes are included to provide context, instructions, and warnings about the workflow’s design choices, testing methods, and limitations.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

**Node Details:**  
- These nodes do not affect workflow execution but serve as inline documentation and reminders for users.

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                                   | Input Node(s)                  | Output Node(s)                        | Sticky Note                                                                                                   |
|------------------------------|---------------------------|-------------------------------------------------|-------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger  | Entry point for external workflow invocation     |                               | Prepare Upload Data                 |                                                                                                               |
| temp form to test workflow    | Form Trigger              | Temporary form for manual testing                 |                               | Prepare Upload Data                 | ## 🧪  Quick Test Form during building ... ⚠️ Remove before putting flow live.                                |
| Prepare Upload Data           | Set                       | Determines bucket based on MIME type              | When Executed by Another Workflow, temp form to test workflow | Convert to File                    | ## 🔧Highlevel dataprep explanation ...                                                                       |
| Convert to File               | Convert to File           | Converts base64 string to binary file             | Prepare Upload Data            | Upload to Supabase Storage          | Instead of the code note i would prefer to use the n8n dedicated note for this ...                            |
| Upload to Supabase Storage    | HTTP Request              | Uploads binary file to Supabase Storage            | Convert to File               | Generate Signed Url, Upload Error Response | ## ⚖️ Pro's and Con's of base64 encoding here ... ## 🔑 About Supabase Storage ...                           |
| Generate Signed Url           | HTTP Request              | Requests signed URL for uploaded object            | Upload to Supabase Storage    | add domain, Sign Error Response     | ## ⚖️ Pro's and Con's of base64 encoding here ... ## 🔑 About Supabase Storage ...                           |
| add domain                   | Set                       | Constructs full signed URL with domain             | Generate Signed Url           | Success Response                   |                                                                                                               |
| Success Response             | Set                       | Sets success response payload                       | add domain                   |                                     |                                                                                                               |
| Upload Error Response         | Set                       | Sets error response if upload fails                 | Upload to Supabase Storage    |                                     |                                                                                                               |
| Sign Error Response           | Set                       | Sets error response if signed URL generation fails | Generate Signed Url           |                                     |                                                                                                               |
| Sticky Note                  | Sticky Note               | Documentation and context                           |                               |                                     | ## 📦 Upload files to Supabase Storage ... [Supabase docs](https://supabase.com/docs/guides/storage)          |
| Sticky Note1                 | Sticky Note               | Explains the test form                              |                               |                                     | ## 🧪  Quick Test Form during building ... ⚠️ Remove before putting flow live.                                |
| Sticky Note2                 | Sticky Note               | Discusses base64 encoding trade-offs               |                               |                                     | ## ⚖️ Pro's and Con's of base64 encoding here ...                                                             |
| Sticky Note3                 | Sticky Note               | Explains data preparation and conversion           |                               |                                     | ## 🔧Highlevel dataprep explanation ...                                                                       |
| Sticky Note4                 | Sticky Note               | Explains Supabase Storage auth and usage           |                               |                                     | ## 🔑 About Supabase Storage ...                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create ‘When Executed by Another Workflow’ node**  
   - Type: Execute Workflow Trigger (version 1.1)  
   - Parameters: Define workflow inputs: `mime_type`, `original_filename`, `binary_data_base64`  
   - Position: Start on canvas left side.  

2. **Create ‘temp form to test workflow’ node** (optional, for testing)  
   - Type: Form Trigger (version 2.2)  
   - Configure form path as `action-workflows-testform`  
   - Add form fields:  
     - `original_filename` (required string)  
     - `binary_data_base64` (required textarea)  
     - `mime_type` (optional string)  
   - Position near ‘When Executed by Another Workflow’.

3. **Create ‘Prepare Upload Data’ node**  
   - Type: Set (version 3.4)  
   - Add assignment field `bucket_name` with expression:  
     ```js
     (() => {
       const mimeType = $json.mime_type || 'application/octet-stream';
       if (mimeType.startsWith('image/')) return 'image-files';
       if (mimeType.startsWith('audio/')) return 'audio-files';
       if (mimeType.startsWith('video/')) return 'video-files';
       return 'document-files';
     })()
     ```  
   - Include all other fields (`includeOtherFields` = true)  
   - Connect inputs from both ‘When Executed by Another Workflow’ and ‘temp form to test workflow’.

4. **Create ‘Convert to File’ node**  
   - Type: Convert to File (version 1.1)  
   - Operation: toBinary  
   - Source Property: `binary_data_base64`  
   - Configure file name: `={{ $json.original_filename }}`  
   - Configure MIME type: `={{ $json.mime_type }}`  
   - Connect output from ‘Prepare Upload Data’.

5. **Create ‘Upload to Supabase Storage’ node**  
   - Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL:  
     ```text
     =https://api-sb.janwillemaltink.com/storage/v1/object/{{ $('Prepare Upload Data').item.json.bucket_name }}/{{ $('Prepare Upload Data').item.json.original_filename }}
     ```  
   - Authentication: Predefined credential, select or create ‘Supabase account’ with Bearer token for Supabase Storage  
   - Content Type: binaryData  
   - Input Data Field Name: `data`  
   - On Error: Continue with error output  
   - Connect input from ‘Convert to File’.

6. **Create ‘Generate Signed Url’ node**  
   - Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL:  
     ```text
     =https://api-sb.janwillemaltink.com/storage/v1/object/sign/{{ $json.Key }}
     ```  
   - Authentication: Use same Supabase API credential as upload node  
   - Body Type: JSON  
   - JSON Body: `{ "expiresIn": 2592000 }` (30 days)  
   - On Error: Continue with error output  
   - Connect input from ‘Upload to Supabase Storage’.

7. **Create ‘add domain’ node**  
   - Type: Set (version 3.4)  
   - Add field `full_signedURL` with expression:  
     ```text
     =https://api-sb.janwillemaltink.com/storage/v1{{ $json.signedURL }}
     ```  
   - Connect input from ‘Generate Signed Url’.

8. **Create ‘Success Response’ node**  
   - Type: Set (version 3.4)  
   - Set fields:  
     - `action` = ‘s3_upload’  
     - `status` = ‘success’  
   - Include all other fields  
   - Connect input from ‘add domain’.

9. **Create ‘Upload Error Response’ node**  
   - Type: Set (version 3.4)  
   - Set fields:  
     - `action` = ‘supabase_upload’  
     - `status` = ‘error’  
   - Include all other fields  
   - Connect error output from ‘Upload to Supabase Storage’.

10. **Create ‘Sign Error Response’ node**  
    - Type: Set (version 3.4)  
    - Set fields:  
      - `action` = ‘supabase_sign’  
      - `status` = ‘error’  
    - Include all other fields  
    - Connect error output from ‘Generate Signed Url’.

11. **Connect all nodes appropriately**:  
    - ‘When Executed by Another Workflow’ → ‘Prepare Upload Data’  
    - ‘temp form to test workflow’ → ‘Prepare Upload Data’  
    - ‘Prepare Upload Data’ → ‘Convert to File’ → ‘Upload to Supabase Storage’ → ‘Generate Signed Url’ → ‘add domain’ → ‘Success Response’  
    - Error outputs from ‘Upload to Supabase Storage’ → ‘Upload Error Response’  
    - Error outputs from ‘Generate Signed Url’ → ‘Sign Error Response’

12. **Create and add Sticky Notes** (optional but recommended for documentation):  
    - Add notes with content from the original workflow to explain usage, trade-offs, and setup instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| 📦 Upload files to Supabase Storage - Demonstrates uploading files categorized by MIME type into Supabase buckets; outputs signed URLs.                                          | [Supabase Storage Docs](https://supabase.com/docs/guides/storage)                                   |
| 🧪 Quick Test Form during building - Temporary form for manual testing of workflow without external triggers; warns to remove before production use.                            |                                                                                                     |
| ⚖️ Pro's and Con's of base64 encoding here - Notes increased size of base64 encoded files (~33%) and large execution data; caution for large files.                            |                                                                                                     |
| 🔧 High-level data preparation explanation - Describes bucket mapping by MIME type and file conversion process.                                                                  |                                                                                                     |
| 🔑 About Supabase Storage - Explains Supabase HTTP API authentication with Bearer tokens, use of anon/service keys, and rationale for generating signed URLs instead of sharing keys. |                                                                                                     |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.