Automated Upload & Send a Large File with TransferNow

https://n8nworkflows.xyz/workflows/automated-upload---send-a-large-file-with-transfernow-8742


# Automated Upload & Send a Large File with TransferNow

### 1. Workflow Overview

This workflow automates the process of uploading a single large file (up to 5GB) through a custom web form and sending it via TransferNow, a file transfer service that supports multi-part uploads for large files. It handles all the complexity of TransferNow’s API, including creating transfers, uploading file parts, and finalizing the transfer, while providing the user with a friendly upload interface and confirmation.

Logical blocks:

- **1.1 Input Reception:** Web form to collect file and metadata from the user.
- **1.2 File Size Calculation:** Determine exact file size and prepare file metadata for TransferNow.
- **1.3 Transfer Creation:** Call TransferNow API to create a transfer and prepare for upload.
- **1.4 Multi-Part Upload Handling:** Retrieve upload URLs for parts, upload file parts, and confirm upload completion.
- **1.5 Transfer Finalization and Confirmation:** Verify transfer completion, extract download URL and recipient info, and present confirmation to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures user input via a web form, including title, message, recipient email, and the file to upload.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Configuration:  
    - Form titled "Upload Form"  
    - Fields: Title (text), Message (textarea), To (email), data (single file upload)  
    - Description: "Upload PDF files to send via TransferNow"  
  - Input: HTTP webhook triggered on form submission  
  - Output: JSON including form fields and the uploaded file’s binary data  
  - Edge cases: Validate required fields; file upload failures or unsupported file types may cause issues  
  - Version: 2.3  

---

#### 1.2 File Size Calculation

**Overview:**  
Calculates the exact size of the uploaded file and prepares the file metadata object required by TransferNow’s API.

**Nodes Involved:**  
- Calculate size  
- Set Json  
- Merge

**Node Details:**  
- **Calculate size**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Retrieves binary buffer of uploaded file  
    - Extracts file name from binary metadata  
    - Returns JSON containing `name` and `size` of the file  
  - Input: Output from "On form submission" (binary file data)  
  - Output: JSON with file metadata  
  - Edge cases: Failures in accessing binary data or file metadata may occur  
  - Version: 2  

- **Set Json**  
  - Type: Set  
  - Configuration:  
    - Constructs JSON object with properties: `name` (from previous node), `size` (from previous node)  
  - Input: Output of "Calculate size"  
  - Output: JSON formatted specifically for TransferNow API  
  - Version: 3.4  

- **Merge**  
  - Type: Merge  
  - Configuration:  
    - Combines two inputs by position: the original form submission data and the JSON file metadata  
  - Input: Two branches —  
    - Input 1: From "On form submission" (form data)  
    - Input 2: From "Calculate size" (file metadata)  
  - Output: Combined JSON with all form fields plus file metadata  
  - Edge cases: Mismatch in input ordering or data shape could cause merges to fail  
  - Version: 3.1  

---

#### 1.3 Transfer Creation

**Overview:**  
Creates a new transfer on TransferNow using the provided metadata and initiates the multi-part upload structure.

**Nodes Involved:**  
- Set Transfer  
- Get Upload Url

**Node Details:**  
- **Set Transfer**  
  - Type: HTTP Request  
  - Configuration:  
    - POST to `https://api.transfernow.net/v1/transfers`  
    - JSON body includes:  
      - `langCode`: "it" (Italian language)  
      - `toEmails`: array with recipient email from form  
      - `files`: array with file info JSON stringified from previous node  
      - `message`, `subject`: from form fields  
      - `validityStart`: current timestamp  
      - `validityEnd`: one week from now  
      - `allowPreview`: true  
      - `maxDownloads`: 7  
    - Authentication: HTTP header with `x-api-key` (TransferNow API key credential)  
    - Headers: Content-Type application/json  
  - Input: Output from "Set Json"  
  - Output: JSON response from TransferNow including transfer and file IDs, multipart upload info  
  - Edge cases: API key errors, invalid email, exceeding size limits, or malformed requests  
  - Version: 4.2  

- **Get Upload Url**  
  - Type: HTTP Request  
  - Configuration:  
    - GET request to obtain upload URL for the first part of the first file in the multipart upload  
    - URL is dynamically constructed using transfer ID, file ID, part number, and upload ID from previous node’s JSON  
    - Authentication: same HTTP header auth  
    - Headers: Content-Type application/json  
  - Input: Output from "Set Transfer"  
  - Output: JSON containing upload URL for part 1  
  - Edge cases: Incorrect IDs or expired transfer may cause 404 or auth errors  
  - Version: 4.2  

---

#### 1.4 Multi-Part Upload Handling

**Overview:**  
Uploads the file part to the provided URL and signals TransferNow that the upload is complete.

**Nodes Involved:**  
- Send UploadUrl  
- Send Transfer  
- Is complete?  
- Upload done

**Node Details:**  
- **Send UploadUrl**  
  - Type: HTTP Request  
  - Configuration:  
    - PUT request to the upload URL received from "Get Upload Url"  
    - Sends raw binary data of the file as request body (`data` field)  
    - Headers: Content-Type `application/octet-stream`  
    - Response: Full HTTP response enabled to track success status  
  - Input: Output from "Merge" (contains binary data and file metadata), connected via "Get Upload Url" output  
  - Output: Response from file upload endpoint  
  - Edge cases: Network timeouts, partial uploads, or HTTP errors must be handled  
  - Version: 4.2  

- **Send Transfer**  
  - Type: HTTP Request  
  - Configuration:  
    - PUT request to TransferNow endpoint signaling upload completion for the file part  
    - URL dynamically constructed using transferId, fileId, and uploadId from "Set Transfer" node  
    - Authentication: HTTP header auth  
  - Input: Output from "Send UploadUrl"  
  - Output: JSON response indicating upload status  
  - Edge cases: Upload incomplete or API rejection  
  - Version: 4.2  

- **Is complete?**  
  - Type: If  
  - Configuration:  
    - Checks if the message returned by "Send Transfer" equals "OK" to confirm successful upload completion  
  - Input: Output from "Send Transfer"  
  - Outputs:  
    - True branch: proceeds to "Upload done" and "Get transfer data"  
    - False branch: no further action (could be enhanced with error handling)  
  - Version: 2.2  

- **Upload done**  
  - Type: HTTP Request  
  - Configuration:  
    - PUT request to `upload-done` endpoint on TransferNow to finalize the entire upload process for the transfer ID  
    - Authentication: HTTP header auth  
  - Input: True branch from "Is complete?"  
  - Output: Response confirming upload finalization  
  - Edge cases: Failures in finalizing upload, network errors  
  - Version: 4.2  

---

#### 1.5 Transfer Finalization and Confirmation

**Overview:**  
Retrieves final transfer details, prepares user-friendly download URL and email, and displays a completion message.

**Nodes Involved:**  
- Get transfer data  
- Get parameters  
- Form

**Node Details:**  
- **Get transfer data**  
  - Type: HTTP Request  
  - Configuration:  
    - GET request to retrieve full transfer info by transferId from "Set Transfer" node  
    - Authentication: HTTP header auth  
  - Input: Output from "Upload done"  
  - Output: Detailed JSON including domain, transfer ID, recipients, and file info  
  - Edge cases: API errors or invalid transfer ID  
  - Version: 4.2  

- **Get parameters**  
  - Type: Set  
  - Configuration:  
    - Extracts and constructs:  
      - `url_transfer`: download URL constructed as `https://{domain}/dl/{id}/{recipient_secret}`  
      - `email`: recipient email from transfer data  
      - `name_transfer`: transfer name/title  
  - Input: Output from "Get transfer data"  
  - Output: JSON with user-friendly parameters for confirmation  
  - Version: 3.4  

- **Form**  
  - Type: Form (Completion)  
  - Configuration:  
    - Completion form displayed after workflow finishes  
    - Title: "Upload complete"  
    - Message:  
      - Includes the recipient email and the generated download URL dynamically rendered from previous node’s output  
  - Input: Output from "Get parameters"  
  - Output: Display to user confirming successful upload and providing download link  
  - Version: 2.3  

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                   | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                  |
|-------------------|---------------------|---------------------------------|---------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note       | Sticky Note         | Instruction / Setup Info         | -                         | -                       | ## STEP 1 Create a FREE account on [TransferNow](https://developers.transfernow.net/). FREE TRIAL 14 DAYS Set Header Auth in 'HTTP Nodes': - NAME: x-api-key - VALUE: YOUR_API_KEY |
| Sticky Note1      | Sticky Note         | Workflow overview description    | -                         | -                       | #  Automated Send a Large File with TransferNow This workflow creates a user-friendly web form to upload a file, which allows users to upload a **single large file** (*up to 5Gb*) through a custom web form and automatically send it via **TransferNow**, handling the complex multi-part upload process required for **large files**. |
| Sticky Note2      | Sticky Note         | File size calculation explanation| -                         | -                       | ## Size Calculate the exact size of the file and put it into an array that will be sent to Transfernow |
| Sticky Note3      | Sticky Note         | Upload block explanation         | -                         | -                       | ## Upload Upload the file to the TransferNow server and create the transfer                   |
| Sticky Note4      | Sticky Note         | Transfer creation explanation    | -                         | -                       | ## Transfer Create file transfer                                                            |
| Sticky Note5      | Sticky Note         | Download URL explanation         | -                         | -                       | ## Download Provides the URL for downloading the file                                       |
| On form submission| Form Trigger        | Input reception via web form     | -                         | Merge, Calculate size    |                                                                                              |
| Calculate size    | Code                | Calculate file size & metadata   | On form submission         | Set Json                 |                                                                                              |
| Set Json          | Set                 | Format JSON for TransferNow API | Calculate size            | Set Transfer             |                                                                                              |
| Merge             | Merge               | Combine form data & file info    | On form submission, Get Upload Url | Send UploadUrl       |                                                                                              |
| Set Transfer      | HTTP Request        | Create transfer on TransferNow   | Set Json                  | Get Upload Url           |                                                                                              |
| Get Upload Url    | HTTP Request        | Get upload URL for file part     | Set Transfer              | Merge                    |                                                                                              |
| Send UploadUrl    | HTTP Request        | Upload file part binary data     | Merge                     | Send Transfer            |                                                                                              |
| Send Transfer     | HTTP Request        | Signal upload completion         | Send UploadUrl            | Is complete?             |                                                                                              |
| Is complete?      | If                  | Check upload completion message  | Send Transfer             | Upload done, Get transfer data |                                                                                              |
| Upload done       | HTTP Request        | Finalize upload on TransferNow   | Is complete? (true branch)| Get transfer data        |                                                                                              |
| Get transfer data | HTTP Request        | Retrieve transfer details        | Upload done               | Get parameters           |                                                                                              |
| Get parameters    | Set                 | Extract download URL & email     | Get transfer data         | Form                     |                                                                                              |
| Form              | Form                | Show confirmation message        | Get parameters            | -                        |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Web Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Configure form with fields:  
     - Title (text, required)  
     - Message (textarea, required)  
     - To (email, required)  
     - data (file, single file, required)  
   - Set webhook ID or leave autogenerated  

2. **Add Code Node to Calculate File Size**  
   - Type: Code (JavaScript)  
   - Name: "Calculate size"  
   - Code:  
     ```js
     const buffer = await this.helpers.getBinaryDataBuffer(0, 'data');
     const name = $input.first().binary.data.fileName;
     return [{ json: { name, size: buffer.length } }];
     ```  
   - Connect input from "On form submission" node  

3. **Add Set Node to Format JSON File Metadata**  
   - Type: Set  
   - Name: "Set Json"  
   - Mode: Raw JSON  
   - JSON Output:  
     ```json
     {
       "name": "={{ $json.name }}",
       "size": {{ $json.size }}
     }
     ```  
   - Input from "Calculate size" node  

4. **Add HTTP Request Node to Create Transfer**  
   - Type: HTTP Request  
   - Name: "Set Transfer"  
   - Method: POST  
   - URL: `https://api.transfernow.net/v1/transfers`  
   - Authentication: HTTP Header Auth with credential containing `x-api-key` header (set in n8n Credentials)  
   - Headers: Content-Type application/json  
   - JSON Body:  
     ```json
     {
       "langCode": "it",
       "toEmails": ["={{ $('On form submission').item.json.To }}"],
       "files": [={{ JSON.stringify($json) }}],
       "message": "={{ $('On form submission').item.json.Message }}",
       "subject": "={{ $('On form submission').item.json.Title }}",
       "validityStart": "={{ $now }}",
       "validityEnd": "={{ $now.plus({week:1}) }}",
       "allowPreview": true,
       "maxDownloads": 7
     }
     ```  
   - Input from "Set Json" node  

5. **Add HTTP Request Node to Get Upload URL**  
   - Type: HTTP Request  
   - Name: "Get Upload Url"  
   - Method: GET  
   - URL:  
     ```text
     https://api.transfernow.net/v1/transfers/{{ $json.transferId }}/files/{{ $json.files[0].id }}/parts/{{ $json.files[0].multipartUpload.parts[0].partNumber }}?uploadId={{ $json.files[0].multipartUpload.uploadId }}
     ```  
   - Authentication: HTTP Header Auth same as above  
   - Headers: Content-Type application/json  
   - Input from "Set Transfer" node  

6. **Add Merge Node to Combine Form Data and Upload URL**  
   - Type: Merge  
   - Mode: Combine by Position  
   - Inputs:  
     - Input 1: Output of "On form submission"  
     - Input 2: Output of "Get Upload Url"  

7. **Add HTTP Request Node to Upload File Part**  
   - Type: HTTP Request  
   - Name: "Send UploadUrl"  
   - Method: PUT  
   - URL: `={{ $json.uploadUrl }}` (from "Get Upload Url")  
   - Authentication: None  
   - Headers: Content-Type application/octet-stream  
   - Send Body: Binary Data  
   - Binary Property: "data" (file uploaded in form)  
   - Input from "Merge" node  

8. **Add HTTP Request Node to Signal Upload Completion**  
   - Type: HTTP Request  
   - Name: "Send Transfer"  
   - Method: PUT  
   - URL:  
     ```text
     https://api.transfernow.net/v1/transfers/{{ $('Set Transfer').item.json.transferId }}/files/{{ $('Set Transfer').item.json.files[0].id }}/upload-done?uploadId={{ $('Set Transfer').item.json.files[0].multipartUpload.uploadId }}
     ```  
   - Authentication: HTTP Header Auth  
   - Input from "Send UploadUrl"  

9. **Add If Node to Check Upload Completion**  
   - Type: If  
   - Name: "Is complete?"  
   - Condition: `{{$json.message}}` equals `"OK"`  
   - Input from "Send Transfer"  

10. **Add HTTP Request Node to Finalize Upload**  
    - Type: HTTP Request  
    - Name: "Upload done"  
    - Method: PUT  
    - URL:  
      ```text
      https://api.transfernow.net/v1/transfers/{{ $('Set Transfer').item.json.transferId }}/upload-done
      ```  
    - Authentication: HTTP Header Auth  
    - Input from True branch of "Is complete?"  

11. **Add HTTP Request Node to Get Transfer Details**  
    - Type: HTTP Request  
    - Name: "Get transfer data"  
    - Method: GET  
    - URL: `https://api.transfernow.net/v1/transfers/{{ $('Set Transfer').item.json.transferId }}`  
    - Authentication: HTTP Header Auth  
    - Input from "Upload done"  

12. **Add Set Node to Extract Parameters for Confirmation**  
    - Type: Set  
    - Name: "Get parameters"  
    - Assignments:  
      - url_transfer = `https://{{ $json.domain }}/dl/{{ $json.id }}/{{ $json.recipients[0].secret }}`  
      - email = `{{ $json.recipients[0].email }}`  
      - name_transfer = `{{ $json.name }}`  
    - Input from "Get transfer data"  

13. **Add Form Node to Display Completion Message**  
    - Type: Form (Completion mode)  
    - Name: "Form"  
    - Completion Title: "Upload complete"  
    - Completion Message:  
      ```
      The files have been sent to the following email address: {{ $json.email }}
      Download URL: {{ $json.url_transfer }}
      ```  
    - Input from "Get parameters"  

14. **Connect all nodes as per dependencies above.**  

15. **Configure Credentials:**  
    - Create HTTP Header Auth Credential in n8n with header name `x-api-key` and your TransferNow API key as the value.  
    - Assign this credential to all HTTP Request nodes that interact with TransferNow API.  

16. **Test workflow starting from form submission to final confirmation.**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Create a FREE account on TransferNow to get an API key. Free trial available for 14 days.                                                                    | https://developers.transfernow.net/                    |
| Set Header Auth in n8n HTTP credential with header name `x-api-key` and your API key value before running the workflow.                                     | Setup instruction from Sticky Note                      |
| This workflow supports uploading large files up to 5GB by handling TransferNow’s multipart upload API automatically.                                        | Workflow description                                    |
| Transfer validity is set for 1 week with a maximum of 7 downloads allowed per file.                                                                           | Parameters in "Set Transfer" HTTP node                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.