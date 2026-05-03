Convert GIF to MP4 with FFmpegAPI

https://n8nworkflows.xyz/workflows/convert-gif-to-mp4-with-ffmpegapi-8321


# Convert GIF to MP4 with FFmpegAPI

### 1. Workflow Overview

This workflow automates the conversion of a GIF file into an MP4 video using the FFmpegAPI service. It is designed for users who want to upload a GIF and receive a downloadable MP4 version through an easy-to-use web form interface.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives a GIF file upload from a user through a form trigger.
- **1.2 Upload Preparation**: Requests an upload URL and method from FFmpegAPI to securely upload the GIF.
- **1.3 File Upload**: Uploads the GIF file to the FFmpegAPI server.
- **1.4 Processing Request**: Sends a conversion task to FFmpegAPI to transform the uploaded GIF into an MP4 file.
- **1.5 Result Delivery**: Presents the download URL for the converted MP4 file back to the user via a form response.

Supporting the workflow are sticky notes providing setup instructions and user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving a GIF file upload via a web form trigger node. It collects the user input and prepares it for processing.

- **Nodes Involved:**  
  - Attach file

- **Node Details:**

  - **Attach file**  
    - Type: Form Trigger (Webhook trigger with form interface)  
    - Role: Entry point to the workflow, receives GIF file from user  
    - Configuration:  
      - Form Title: "Convert GIF"  
      - Form Description: "Upload a GIF to convert into MP4 using FFmpegAPI"  
      - One file input field labeled "file", accepts `.gif` files only, single file required  
      - Button label: "Convert"  
      - Responds with confirmation text "Done" after submission  
    - Inputs: None (trigger node)  
    - Outputs: Passes file JSON data downstream  
    - Edge cases:  
      - File format validation relies on frontend restriction, backend validation may be absent  
      - If no file is uploaded or an unsupported file type is submitted, workflow may fail or behave unexpectedly  
    - Version: 2.3

#### 2.2 Upload Preparation

- **Overview:**  
  Requests a secure upload URL and method from FFmpegAPI. This URL is used to upload the actual GIF file securely.

- **Nodes Involved:**  
  - Get Upload URL  
  - Merge

- **Node Details:**

  - **Get Upload URL**  
    - Type: HTTP Request  
    - Role: Sends POST request to FFmpegAPI to obtain an upload URL for the GIF file  
    - Configuration:  
      - URL: `https://api.ffmpeg-api.com/file`  
      - Method: POST  
      - Body: JSON containing `"file_name"` dynamically set to the uploaded GIF filename from the "Attach file" node  
      - Headers: Authorization header using Basic Auth with API key stored in variable `FFMPEG_API_KEY`  
    - Expressions:  
      - `{{$item("0").$node["Attach file"].json["file"]["filename"]}}` to access filename of uploaded file  
      - Authorization header: `"=Basic {{ $vars.FFMPEG_API_KEY }}"`  
    - Inputs: Output from "Attach file" (through "Merge")  
    - Outputs: Provides upload URL and method for file upload  
    - Edge cases:  
      - API key missing or invalid → Authorization error  
      - Network timeout or API service unavailability  
      - Response format changes or unexpected errors from FFmpegAPI  
    - Version: 4.2

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from "Attach file" and "Get Upload URL" to synchronize data for next steps  
    - Configuration:  
      - Mode: Combine, combines all incoming items into one array  
    - Inputs:  
      - From "Attach file" node (main output)  
      - From "Get Upload URL" node (main output)  
    - Outputs: One combined dataset containing both file info and upload URL details  
    - Edge cases:  
      - Data mismatch if one input fails or is delayed  
      - Potential for empty input arrays causing downstream errors  
    - Version: 3.2

#### 2.3 File Upload

- **Overview:**  
  Uploads the GIF file to the URL provided by FFmpegAPI using the specified HTTP method.

- **Nodes Involved:**  
  - Upload File

- **Node Details:**

  - **Upload File**  
    - Type: HTTP Request  
    - Role: Performs the actual file upload to FFmpegAPI's server  
    - Configuration:  
      - URL: dynamically read from `Get Upload URL` node output at `upload.url`  
      - Method: dynamically read from `Get Upload URL` node output at `upload.method` (usually PUT)  
      - Content type: binaryData  
      - Input data field name: "file" (binary input)  
      - Full HTTP response requested for verification  
    - Inputs: Output from "Merge" node (which includes file binary and upload URL info)  
    - Outputs: HTTP response from the upload endpoint  
    - Edge cases:  
      - Network issues, upload failures or timeouts  
      - Incorrect URL or method causing 4xx/5xx errors  
      - Binary data not properly passed or corrupted  
    - Version: 4.2

#### 2.4 Processing Request

- **Overview:**  
  Sends a task request to FFmpegAPI to convert the uploaded GIF file to MP4 format.

- **Nodes Involved:**  
  - Process File

- **Node Details:**

  - **Process File**  
    - Type: HTTP Request  
    - Role: Submits the conversion task to FFmpegAPI's processing endpoint  
    - Configuration:  
      - URL: `https://api.ffmpeg-api.com/ffmpeg/process`  
      - Method: POST  
      - Headers: Authorization with API key (`FFMPEG_API_KEY`) using Basic Auth  
      - Body: JSON specifying the input file path (from `Get Upload URL` node response) and output specification for MP4 file named "output.mp4"  
      - Requests full JSON response and full HTTP response  
    - Expressions:  
      - Input file path: `{{$node["Get Upload URL"].item.json.file.file_path}}`  
      - Authorization: `"=Basic {{ $vars.FFMPEG_API_KEY }}"`  
    - Inputs: Output from "Upload File" node (ensures upload is complete before processing)  
    - Outputs: JSON response containing result details including download URL  
    - Edge cases:  
      - Invalid or expired input file path → processing error  
      - API key issues or rate limiting  
      - Conversion failures or unsupported file types  
      - Network or API downtime  
    - Version: 4.2

#### 2.5 Result Delivery

- **Overview:**  
  Presents the user with a download link for the converted MP4 file in the form response.

- **Nodes Involved:**  
  - Download URL

- **Node Details:**

  - **Download URL**  
    - Type: Form (Completion response)  
    - Role: Sends a completion message with a clickable download link to the user after conversion is finished  
    - Configuration:  
      - Operation: completion mode  
      - Completion Title: "Done!"  
      - Completion Message: Includes markdown link `[Download](URL)` where URL is dynamically injected from the conversion result's download URL (`{{$json.body.result[0].download_url}}`)  
    - Inputs: Output from "Process File" node  
    - Outputs: Final user-facing response via webhook/form interface  
    - Edge cases:  
      - If download URL is missing or malformed, link will be broken  
      - User experience depends on timely completion of previous steps  
    - Version: 2.3

---

### 3. Summary Table

| Node Name       | Node Type           | Functional Role             | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                         |
|-----------------|---------------------|----------------------------|-----------------------|------------------------|---------------------------------------------------------------------------------------------------|
| Attach file     | Form Trigger        | Receive GIF file upload     | -                     | Merge                  | ## Intro  This is an example workflow that demos the [FFmpegAPI](https://ffmpeg-api.com/) for file conversion. It only shows GIF to MP4 conversion, but you're free to modify it for your use case. |
| Merge           | Merge               | Combine file data and upload URL | Attach file, Get Upload URL | Upload File            |                                                                                                   |
| Get Upload URL  | HTTP Request        | Get upload URL and method   | Merge                  | Merge                  | ## Step 1  Create an account on [FFmpegAPI](https://ffmpeg-api.com/) and copy your API Key from the dashboard. Go to the **Variables** section from the left sidebar of this page. and use **FFMPEG_API_KEY** name to store that key you copied. |
| Upload File     | HTTP Request        | Upload GIF file to FFmpegAPI | Merge                  | Process File            |                                                                                                   |
| Process File    | HTTP Request        | Request conversion of GIF to MP4 | Upload File            | Download URL            | ## Step 2  Click the 'Execute workflow' button at the bottom. A pop-up window will open where you can select a GIF file from your PC. Once you click convert, the workflow will: a. get an upload file path b. upload that gif c. convert that to mp4 You'll get the final download URL when it's done. |
| Download URL    | Form                | Provide download link       | Process File            | -                      |                                                                                                   |
| Sticky Note     | Sticky Note         | Instructions Step 1         | -                      | -                      | ## Step 1  Create an account on [FFmpegAPI](https://ffmpeg-api.com/) and copy your API Key from the dashboard. Go to the **Variables** section from the left sidebar of this page. and use **FFMPEG_API_KEY** name to store that key you copied. |
| Sticky Note1    | Sticky Note         | Instructions Step 2         | -                      | -                      | ## Step 2  Click the 'Execute workflow' button at the bottom. A pop-up window will open where you can select a GIF file from your PC. Once you click convert, the workflow will: a. get an upload file path b. upload that gif c. convert that to mp4 You'll get the final download URL when it's done. |
| Sticky Note2    | Sticky Note         | Introductory note           | -                      | -                      | ## Intro  This is an example workflow that demos the [FFmpegAPI](https://ffmpeg-api.com/) for file conversion. It only shows GIF to MP4 conversion, but you're free to modify it for your use case. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node ("Attach file"):**  
   - Set type to *Form Trigger* (Webhook).  
   - Configure form title: "Convert GIF".  
   - Add a single file upload field:  
     - Label: "file"  
     - Accept only `.gif` files  
     - Required field, single file only  
   - Set button label to "Convert".  
   - Configure response mode to "lastNode" with confirmation text "Done".  
   - Save webhook ID for endpoint reference.

3. **Add an HTTP Request node ("Get Upload URL"):**  
   - Method: POST  
   - URL: `https://api.ffmpeg-api.com/file`  
   - Headers:  
     - Authorization: `Basic {{ $vars.FFMPEG_API_KEY }}` (store your API key in n8n Variables as `FFMPEG_API_KEY`)  
   - Body: JSON with property `"file_name"` set dynamically to uploaded file's filename from "Attach file":  
     ```json
     {
       "file_name": "{{ $item(\"0\").$node[\"Attach file\"].json[\"file\"][\"filename\"] }}"
     }
     ```  
   - Set body type to JSON.  
   - Set response format to JSON.

4. **Add a Merge node ("Merge"):**  
   - Mode: Combine  
   - Combine all inputs  
   - Connect "Attach file" output to first input of "Merge".  
   - Connect "Get Upload URL" output to second input of "Merge".

5. **Add an HTTP Request node ("Upload File"):**  
   - URL: dynamically set from "Get Upload URL" output `upload.url`  
   - Method: dynamically set from "Get Upload URL" output `upload.method` (usually PUT)  
   - Content type: binaryData  
   - Input data field name: "file"  
   - Send the actual GIF file binary from "Attach file" (via "Merge")  
   - Request full HTTP response.

6. **Add an HTTP Request node ("Process File"):**  
   - Method: POST  
   - URL: `https://api.ffmpeg-api.com/ffmpeg/process`  
   - Headers: Authorization: `Basic {{ $vars.FFMPEG_API_KEY }}`  
   - Body (JSON):  
     ```json
     {
       "task": {
         "inputs": [{ "file_path": "{{ $('Get Upload URL').item.json.file.file_path }}" }],
         "outputs": [{ "file": "output.mp4" }]
       }
     }
     ```  
   - Set body type to JSON.  
   - Request full JSON response.

7. **Add a Form node ("Download URL"):**  
   - Operation: completion  
   - Completion Title: "Done!"  
   - Completion Message:  
     ```
     Please download the file: [Download]({{ $json.body.result[0].download_url }})
     ```  
   - Connect output of "Process File" to "Download URL".

8. **Connect nodes in this order:**  
   - "Attach file" → "Merge" (input 1)  
   - "Get Upload URL" → "Merge" (input 2)  
   - "Merge" → "Upload File"  
   - "Upload File" → "Process File"  
   - "Process File" → "Download URL"

9. **Credentials:**  
   - Store your FFmpegAPI API key in n8n Variables under the name `FFMPEG_API_KEY`.  
   - No other credentials are required unless you add authentication.

10. **Save and activate the workflow.**

11. **Test by executing the workflow:**  
    - Trigger the webhook or use the form interface to upload a `.gif`.  
    - The workflow will process and return a download link for the MP4 file.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                  |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Create an account on [FFmpegAPI](https://ffmpeg-api.com/) and get your API key for authentication.   | https://ffmpeg-api.com/                          |
| Store the API key in n8n Variables under the key name `FFMPEG_API_KEY`.                               | n8n Variables section                            |
| This workflow demonstrates only GIF to MP4 conversion, but FFmpegAPI supports many other formats.    | FFmpegAPI documentation and API capabilities    |
| Execution instructions: use the 'Execute workflow' button and upload a GIF to start conversion.      | User interface instructions from sticky notes   |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.