Extract Organizations & Summarize Documents with Foxit and Diffbot

https://n8nworkflows.xyz/workflows/extract-organizations---summarize-documents-with-foxit-and-diffbot-6286


# Extract Organizations & Summarize Documents with Foxit and Diffbot

### 1. Workflow Overview

This n8n workflow automates the process of extracting organizational entities and summarizing documents uploaded to a specific Google Drive folder. It combines document ingestion, text extraction through Foxit PDF services, and content analysis via Diffbot’s AI to create an informative email report.

**Target Use Cases:**  
- Automated document ingestion and analysis for business intelligence.  
- Extracting key organizational mentions and summaries from PDFs or other documents.  
- Sending summarized insights via email upon document upload.

**Logical Blocks:**  
- **1.1 Ingestion:** Detect new files in a Google Drive folder and download them.  
- **1.2 Uploading and Extraction:** Upload the file to Foxit, trigger asynchronous text extraction, and poll until extraction completes.  
- **1.3 Analyzing:** Fetch the extracted text and call Diffbot to extract entities and summarize content.  
- **1.4 Shaping and Emailing:** Process Diffbot results into an HTML report and send it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Ingestion

**Overview:**  
This block monitors a specific Google Drive folder and triggers the workflow when a new file is created. It downloads the detected file for further processing.

**Nodes Involved:**  
- Fire on New File in Google Drive Folder  
- Download File  
- Sticky Note (Ingestion explanation)

**Node Details:**

- **Fire on New File in Google Drive Folder**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches a specified Google Drive folder for new files (polls every minute).  
  - *Configuration:* Trigger on `fileCreated` event for folder URL `https://drive.google.com/drive/folders/1yHMr9A68V4QLtd6fIKuM7BE1U4mpN6cD`.  
  - *Credentials:* Uses Google Drive OAuth2.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Triggers on new file metadata.  
  - *Failures:* Possible API rate limits, folder access permission errors.  
  - *Sticky Note:* Explains workflow starts on file upload.

- **Download File**  
  - *Type:* Google Drive  
  - *Role:* Downloads the file identified by the trigger node.  
  - *Configuration:* Downloads file by ID using expression `={{ $json.id }}` from trigger output.  
  - *Credentials:* Google Drive OAuth2.  
  - *Inputs:* File metadata from trigger.  
  - *Outputs:* Binary file data for upload.  
  - *Failures:* File access errors, download timeouts.

- **Sticky Note (Ingestion)**  
  - *Content:* Explains the ingestion block functionality.

---

#### 1.2 Uploading and Extraction

**Overview:**  
Uploads the downloaded file to Foxit's PDF services, initiates an asynchronous text extraction job, then polls Foxit’s API until the extraction task completes.

**Nodes Involved:**  
- Upload to Foxit  
- Kick off Foxit Extract  
- Check Task  
- Is the job done? (IF node)  
- Wait  
- Sticky Note (Uploading and Extracting)

**Node Details:**

- **Upload to Foxit**  
  - *Type:* HTTP Request  
  - *Role:* Uploads the binary file to Foxit PDF Services for processing.  
  - *Configuration:* POST to `https://na1.fusion.foxit.com/pdf-services/api/documents/upload` with multipart form-data including the binary file under "file".  
  - *Authentication:* Custom HTTP authentication via stored credentials.  
  - *Inputs:* Binary file data from "Download File".  
  - *Outputs:* JSON response including `documentId`.  
  - *Failures:* Network issues, auth errors, file size limits.

- **Kick off Foxit Extract**  
  - *Type:* HTTP Request  
  - *Role:* Starts an asynchronous text extraction job on the uploaded document.  
  - *Configuration:* POST to Foxit’s `/pdf-extract` API endpoint with JSON body referencing `documentId` and specifying extraction type as `"TEXT"`.  
  - *Authentication:* Same custom HTTP credentials.  
  - *Inputs:* JSON from "Upload to Foxit" node.  
  - *Outputs:* JSON with extraction task information including `taskId`.  
  - *Failures:* API rate limits, invalid document ID.

- **Check Task**  
  - *Type:* HTTP Request  
  - *Role:* Polls Foxit API to check the status of the extraction task.  
  - *Configuration:* GET request to `https://na1.fusion.foxit.com/pdf-services/api/tasks/{{$json.taskId}}`.  
  - *Authentication:* Same custom HTTP credentials.  
  - *Inputs:* Extraction task info from "Kick off Foxit Extract".  
  - *Outputs:* JSON status of the task (`status` field).  
  - *Failures:* Network timeouts, task not found errors.

- **Is the job done?**  
  - *Type:* IF  
  - *Role:* Evaluates whether the extraction task status equals `"COMPLETED"`.  
  - *Conditions:* `$json.status === "COMPLETED"`.  
  - *Inputs:* From "Check Task".  
  - *Outputs:*  
    - True branch: proceed to download extracted text.  
    - False branch: loop back to Wait node.  
  - *Failures:* Expression parsing errors.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses the workflow before rechecking task status (implements polling delay).  
  - *Inputs:* False branch from "Is the job done?".  
  - *Outputs:* Loops back to "Check Task".  
  - *Failures:* Potentially long wait times or infinite loop if task never completes.

- **Sticky Note (Uploading and Extracting)**  
  - *Content:* Describes this asynchronous extraction process.

---

#### 1.3 Analyzing

**Overview:**  
After text extraction, this block downloads the extracted text from Foxit and sends it to Diffbot’s Natural Language API to extract entities and generate a summary.

**Nodes Involved:**  
- Download Extracted Text  
- Get Diffbot Entities  
- Sticky Note (Analyzing)

**Node Details:**

- **Download Extracted Text**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the extracted text document from Foxit.  
  - *Configuration:* GET request to `https://na1.fusion.foxit.com/pdf-services/api/documents/{{ $('Check Task').item.json.resultDocumentId }}/download`.  
  - *Authentication:* Same custom HTTP credentials.  
  - *Inputs:* From "Is the job done?" node (True branch).  
  - *Outputs:* JSON containing extracted text data.  
  - *Failures:* Download errors, invalid `resultDocumentId`.

- **Get Diffbot Entities**  
  - *Type:* HTTP Request  
  - *Role:* Sends extracted text to Diffbot API to obtain entities and summary.  
  - *Configuration:* POST request to `https://nl.diffbot.com/v1/?fields=entities,summary` with JSON body containing text content, language `"en"`, and format `"plain text"`.  
  - *Authentication:* Query parameter authentication with stored credentials.  
  - *Inputs:* Extracted text JSON from "Download Extracted Text".  
  - *Outputs:* JSON with entities array and summary string.  
  - *Failures:* API auth errors, rate limits, invalid text content.

- **Sticky Note (Analyzing)**  
  - *Content:* Summarizes this analysis step.

---

#### 1.4 Shaping and Emailing

**Overview:**  
Processes Diffbot’s entities to filter for high-confidence organizations, constructs an HTML report, and sends it via Gmail.

**Nodes Involved:**  
- Shape Data (Code node)  
- Make Email Contents (Code node)  
- Gmail  
- Sticky Note (Shaping and Emailing)

**Node Details:**

- **Shape Data**  
  - *Type:* Code  
  - *Role:* Filters Diffbot entities to extract organizations with confidence ≥ 0.85 and prepares a simplified JSON with organization names and summary.  
  - *Code Highlights:*  
    - Filters entities where `allTypes` includes `"organization"` and `confidence` is ≥ 0.85.  
    - Extracts organization names into an array.  
    - Returns an object with `organizations` array and `summary` string.  
  - *Inputs:* Diffbot JSON from "Get Diffbot Entities".  
  - *Outputs:* Structured JSON with relevant organizations and summary.  
  - *Failures:* JS errors if input is malformed or missing fields.

- **Make Email Contents**  
  - *Type:* Code  
  - *Role:* Builds an HTML email body listing organizations and document summary, linking back to the original Google Drive file.  
  - *Code Highlights:*  
    - Retrieves document name and link from the original Google Drive trigger node.  
    - Constructs HTML with document title as link, organization list, and summary paragraph.  
  - *Inputs:* Output JSON from "Shape Data".  
  - *Outputs:* JSON with `html` field containing the email body.  
  - *Failures:* JS errors, missing fields from upstream nodes.

- **Gmail**  
  - *Type:* Gmail  
  - *Role:* Sends the constructed HTML email to a fixed recipient (`raymondcamden@gmail.com`).  
  - *Configuration:*  
    - Recipient email hardcoded.  
    - Subject line: "intelligent Document Report on Upload".  
    - Message body uses the HTML from previous node.  
  - *Credentials:* Gmail OAuth2.  
  - *Inputs:* HTML email from "Make Email Contents".  
  - *Outputs:* Email send status.  
  - *Failures:* Gmail auth errors, quota limits.

- **Sticky Note (Shaping and Emailing)**  
  - *Content:* Explains the shaping and emailing process.

---

### 3. Summary Table

| Node Name                          | Node Type                 | Functional Role                      | Input Node(s)                     | Output Node(s)                  | Sticky Note                                              |
|----------------------------------|---------------------------|------------------------------------|---------------------------------|--------------------------------|----------------------------------------------------------|
| Fire on New File in Google Drive Folder | Google Drive Trigger       | Trigger on new file in folder       | None                            | Download File                  | ## Ingestion The workflow kicks off when a new file is added to Google Drive in a particular folder. The data is downloaded to n8n. |
| Download File                    | Google Drive              | Downloads new file                  | Fire on New File in Google Drive Folder | Upload to Foxit               |                                                          |
| Upload to Foxit                 | HTTP Request              | Upload file to Foxit PDF service   | Download File                   | Kick off Foxit Extract         | ## Uploading and Extracting Next, the document is uploaded to Foxit and the Extraction API is fired to get the text. This is an async job so we check the status in a loop. |
| Kick off Foxit Extract          | HTTP Request              | Start text extraction job           | Upload to Foxit                | Check Task                    |                                                          |
| Check Task                     | HTTP Request              | Poll extraction task status         | Kick off Foxit Extract          | Is the job done?              |                                                          |
| Is the job done?                | IF                        | Check if extraction is complete     | Check Task                     | Download Extracted Text, Wait |                                                          |
| Wait                           | Wait                      | Pause before polling again          | Is the job done?               | Check Task                    |                                                          |
| Download Extracted Text         | HTTP Request              | Download extracted text             | Is the job done? (True branch) | Get Diffbot Entities          |                                                          |
| Get Diffbot Entities            | HTTP Request              | Analyze text with Diffbot API       | Download Extracted Text         | Shape Data                    | ## Analyzing After the Extract result is downloaded, Diffbot is called and asked to analyze and summarize the contents. |
| Shape Data                     | Code                      | Filter organizations and prepare summary | Get Diffbot Entities          | Make Email Contents           | ## Shaping and Emailing Finally, we shape the result from Diffbot and create an HTML string that can be used in the email stage. |
| Make Email Contents             | Code                      | Build HTML email body               | Shape Data                    | Gmail                        |                                                          |
| Gmail                         | Gmail                     | Send HTML email                     | Make Email Contents            | None                         |                                                          |
| Sticky Note                    | Sticky Note               | Ingestion explanation               | None                          | None                         | ## Ingestion The workflow kicks off when a new file is added to Google Drive in a particular folder. The data is downloaded to n8n. |
| Sticky Note1                   | Sticky Note               | Uploading and extraction explanation | None                          | None                         | ## Uploading and Extracting Next, the document is uploaded to Foxit and the Extraction API is fired to get the text. This is an async job so we check the status in a loop. |
| Sticky Note2                   | Sticky Note               | Analyzing explanation               | None                          | None                         | ## Analyzing After the Extract result is downloaded, Diffbot is called and asked to analyze and summarize the contents. |
| Sticky Note3                   | Sticky Note               | Shaping and emailing explanation    | None                          | None                         | ## Shaping and Emailing Finally, we shape the result from Diffbot and create an HTML string that can be used in the email stage. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**  
   - Type: Google Drive Trigger  
   - Event: `fileCreated`  
   - Folder to watch: Use folder URL `https://drive.google.com/drive/folders/1yHMr9A68V4QLtd6fIKuM7BE1U4mpN6cD`  
   - Poll interval: Every minute  
   - Credentials: Configure Google Drive OAuth2 API

2. **Add Google Drive node to Download File:**  
   - Operation: Download  
   - File ID: Expression `={{ $json.id }}` from trigger output  
   - Credentials: Google Drive OAuth2 API  
   - Connect output of trigger to this node.

3. **Add HTTP Request node "Upload to Foxit":**  
   - Method: POST  
   - URL: `https://na1.fusion.foxit.com/pdf-services/api/documents/upload`  
   - Content-Type: multipart/form-data  
   - Body Parameters: Add parameter named `file` as form binary data, input field: `data` (binary from previous node)  
   - Authentication: Custom HTTP Auth (configure with Foxit API credentials)  
   - Connect "Download File" output to this node.

4. **Add HTTP Request node "Kick off Foxit Extract":**  
   - Method: POST  
   - URL: `https://na1.fusion.foxit.com/pdf-services/api/documents/modify/pdf-extract`  
   - Body Type: JSON  
   - Body content:  
     ```json
     {
       "documentId": "{{ $json.documentId }}",
       "extractType": "TEXT"
     }
     ```  
   - Authentication: Same Custom HTTP Auth  
   - Connect output of "Upload to Foxit" to this node.

5. **Add HTTP Request node "Check Task":**  
   - Method: GET  
   - URL: `https://na1.fusion.foxit.com/pdf-services/api/tasks/{{$json.taskId}}`  
   - Authentication: Custom HTTP Auth  
   - Connect output of "Kick off Foxit Extract" to this node.

6. **Add IF node "Is the job done?":**  
   - Condition: Check if `$json.status` equals `"COMPLETED"` (strict, case sensitive)  
   - Connect "Check Task" output to this node.

7. **Add Wait node:**  
   - No special parameters (default wait time can be configured if desired)  
   - Connect False output of IF node to this Wait node.  
   - Connect Wait node output back to "Check Task" node for polling loop.

8. **Add HTTP Request node "Download Extracted Text":**  
   - Method: GET  
   - URL: `https://na1.fusion.foxit.com/pdf-services/api/documents/{{ $('Check Task').item.json.resultDocumentId }}/download`  
   - Authentication: Custom HTTP Auth  
   - Connect True output of IF node to this node.

9. **Add HTTP Request node "Get Diffbot Entities":**  
   - Method: POST  
   - URL: `https://nl.diffbot.com/v1/?fields=entities,summary`  
   - Body Type: JSON  
   - Body content:  
     ```json
     [{
       "content": {{ JSON.stringify($json.data) }},
       "lang": "en",
       "format": "plain text"
     }]
     ```  
   - Authentication: Query Parameter Auth (Diffbot API key)  
   - Connect output of "Download Extracted Text" to this node.

10. **Add Code node "Shape Data":**  
    - JavaScript code:  
      ```javascript
      let organizations = $input.all()[0].json.entities.filter(d => {
        let hasOrg = d.allTypes.some(t => t.name === 'organization');
        let isConfident = d.confidence >= 0.85;
        return hasOrg && isConfident;
      });
      let orgNames = organizations.map(o => o.name);
      return { organizations: orgNames, summary: $input.all()[0].json.summary };
      ```  
    - Connect output of "Get Diffbot Entities" to this node.

11. **Add Code node "Make Email Contents":**  
    - JavaScript code:  
      ```javascript
      let input = $input.all()[0].json;
      let doc = $('Fire on New File in Google Drive Folder').first().json.name;
      let link = $('Fire on New File in Google Drive Folder').first().json.webViewLink;

      let html = `
      <h2>Document Report for <a href="${link}">${doc}</a></h2>
      <p>This document discussed the following organizations:</p>
      <ul>
      `;
      html += input.organizations.map(o => `<li>${o}</li>`).join('');
      html += `</ul>
      <p>Summary for this document:</p>
      <p>${input.summary}</p>
      `;

      return { html };
      ```  
    - Connect output of "Shape Data" to this node.

12. **Add Gmail node:**  
    - Send To: `raymondcamden@gmail.com` (or desired recipient)  
    - Subject: `intelligent Document Report on Upload`  
    - Message: Use expression to pass `html` from previous node (`={{ $json.html }}`)  
    - Credentials: Gmail OAuth2 account  
    - Connect output of "Make Email Contents" to this node.

13. **Add Sticky Note nodes** (optional for documentation):  
    - Add descriptive sticky notes for each block as per the content in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                    |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow uses Foxit PDF Services API for document upload and asynchronous text extraction.     | Foxit API docs: https://developer.foxit.com/pdf-services/       |
| Diffbot Natural Language API is called to extract entities and summarize text content.             | Diffbot NLP API: https://www.diffbot.com/dev/docs/natural-lang/ |
| Gmail node requires OAuth2 credentials with send mail permission.                                  | OAuth2 setup required in n8n credentials                         |
| The polling loop uses a Wait node and an IF node to check task completion asynchronously.          | Avoids infinite loops by waiting between polls                   |
| The workflow is designed to handle English language documents (`lang: "en"`).                      | Modify code and API calls for other languages as needed          |
| The email recipient is hardcoded; adjust the "Gmail" node parameter to customize.                  |                                                                  |

---

This document fully describes the workflow "Extract Organizations & Summarize Documents with Foxit and Diffbot," enabling reproduction, modification, and error anticipation for effective maintenance and extension.