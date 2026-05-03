Bulk AI Video Generation with Freepik Minimax Hailuo & Google Suite Integration

https://n8nworkflows.xyz/workflows/bulk-ai-video-generation-with-freepik-minimax-hailuo---google-suite-integration-7335


# Bulk AI Video Generation with Freepik Minimax Hailuo & Google Suite Integration

### 1. Workflow Overview

This workflow automates bulk AI video generation using Freepik's Minimax Hailuo AI video generation API integrated with Google Suite services. It is designed for users who want to generate multiple video variations from prompts stored in a Google Sheet, monitor the generation process, download the completed videos, and upload them to a specific Google Drive folder seamlessly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preparation**: Fetch prompts from Google Sheets and create multiple variations for batch processing.
- **1.2 Video Generation Requests**: Submit AI video generation requests to Freepik’s API.
- **1.3 Video Generation Status Polling and Control Flow**: Monitor the status of video generation tasks and handle different states (completed, failed, in progress).
- **1.4 Video Download and Upload to Google Drive**: Download the completed video in Base64 format and upload it to a designated Google Drive folder.
- **1.5 Batch Processing and Loop Management**: Manage multiple video generation requests efficiently by processing items in batches.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

- **Overview:**  
This block fetches video generation prompts from a Google Sheet and duplicates each prompt to create multiple variations for batch processing.

- **Nodes Involved:**  
  - Get prompt from google sheet  
  - Duplicate Rows2

- **Node Details:**

  - **Get prompt from google sheet**  
    - Type: Google Sheets (Read Operation)  
    - Role: Retrieves prompts stored in a Google Sheet to be used as input for video generation.  
    - Configuration: Reads from a specific document ID and sheet (`Sheet1` or similar). Uses OAuth2 credentials for Google Sheets.  
    - Expressions: Not explicitly used beyond standard data retrieval.  
    - Inputs: None (starting point)  
    - Outputs: JSON array of prompts with metadata (e.g., name, prompt text).  
    - Potential Failures: OAuth token expiration, spreadsheet access permissions, empty or malformed sheet data.  

  - **Duplicate Rows2**  
    - Type: Code (JavaScript)  
    - Role: Creates two duplicates of each prompt with an added `run` identifier to allow multiple video variations per prompt.  
    - Configuration: JavaScript code duplicates the input item twice, appending `run:1` and `run:2` to each JSON object.  
    - Key Code:  
      ```javascript
      const original = items[0].json;
      return [
        { json: { ...original, run: 1 } },
        { json: { ...original, run: 2 } },
      ];
      ```  
    - Inputs: JSON from Google Sheets node  
    - Outputs: Two JSON objects per input prompt, each with unique `run` value.  
    - Potential Failures: Code syntax errors, empty input items.

---

#### 2.2 Video Generation Requests

- **Overview:**  
This block sends HTTP POST requests to Freepik’s AI video generation API to create videos based on each prompt.

- **Nodes Involved:**  
  - Loop Over Items  
  - Create Video

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Controls batch processing of multiple prompt variations to respect API rate limits and resource constraints.  
    - Configuration: Default batch size; no reset after batch completion.  
    - Inputs: Duplicated prompt variations.  
    - Outputs: Items batched for processing.  
    - Potential Failures: Large batch sizes causing timeouts or API rate limits.

  - **Create Video**  
    - Type: HTTP Request (POST)  
    - Role: Sends video generation requests to Freepik’s Minimax Hailuo API endpoint.  
    - Configuration:  
      - URL: `https://api.freepik.com/v1/ai/image-to-video/minimax-hailuo-02-768p`  
      - Method: POST  
      - Authentication: HTTP Header Auth with API key/secret  
      - Body Parameter: `prompt` set dynamically from input JSON (`={{ $json.Prompt }}`)  
    - Inputs: Batched prompt variations from Loop Over Items.  
    - Outputs: Response containing a task ID to track video generation.  
    - Potential Failures: Authentication errors, malformed prompts, API downtime, rate limits.

---

#### 2.3 Video Generation Status Polling and Control Flow

- **Overview:**  
After initiating video generation, this block polls the API to check the status of each video generation task, directing workflow logic based on the status.

- **Nodes Involved:**  
  - Get Video URL  
  - Switch  
  - Wait

- **Node Details:**

  - **Get Video URL**  
    - Type: HTTP Request (GET)  
    - Role: Queries Freepik API for the status and result URL of the video generation task using the task ID.  
    - Configuration:  
      - URL interpolates `task_id`: `https://api.freepik.com/v1/ai/image-to-video/minimax-hailuo-02-768p/{{ $json.data.task_id }}`  
      - Timeout: 120,000 ms (2 minutes)  
      - Authentication: HTTP Header Auth  
    - Inputs: Task ID from Create Video response or Wait node.  
    - Outputs: JSON with current status and possibly generated video URLs.  
    - Potential Failures: Timeout, authentication errors, invalid task IDs, API errors.

  - **Switch**  
    - Type: Switch (Conditional Routing)  
    - Role: Routes workflow based on the status of the video generation task.  
    - Configuration: Conditions on `{{ $json.data.status }}` with outputs for:  
      - Completed  
      - Failed  
      - Created  
      - In Progress  
    - Inputs: Status JSON from Get Video URL  
    - Outputs:  
      - Completed → proceeds to download video  
      - Failed → workflow ends or retries (not explicitly covered)  
      - Created / In Progress → triggers Wait node to delay before next status check  
    - Potential Failures: If status is missing or unexpected value, routing may fail.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses the workflow for 30 seconds before re-polling the video status.  
    - Configuration: Fixed 30 seconds delay, webhook-enabled for pause/resume.  
    - Inputs: From Switch node if status is Created or In Progress.  
    - Outputs: Loops back to Get Video URL node for re-check.  
    - Potential Failures: Workflow stuck if API never returns terminal status; webhook issues.

---

#### 2.4 Video Download and Upload to Google Drive

- **Overview:**  
Once video generation is complete, this block downloads the video file in Base64 format and uploads it to a specified Google Drive folder with a descriptive name.

- **Nodes Involved:**  
  - Download Video as Base64  
  - Upload to Google Drive1

- **Node Details:**

  - **Download Video as Base64**  
    - Type: HTTP Request (GET)  
    - Role: Downloads the completed video file from the generated URL provided by the API.  
    - Configuration:  
      - URL: dynamically set as `={{ $json.data.generated[0] }}` (first generated video URL)  
      - Method: GET  
    - Inputs: Video URL from Switch node’s Completed path.  
    - Outputs: Base64-encoded video data.  
    - Potential Failures: Invalid or expired video URL, network errors.

  - **Upload to Google Drive1**  
    - Type: Google Drive (Upload)  
    - Role: Uploads the downloaded video to Google Drive into a specific folder with a dynamic, descriptive filename.  
    - Configuration:  
      - File name: `video - {{ $('Get prompt from google sheet').item.json.Name }} - {{ $('Duplicate Rows2').item.json.run }}`  
      - Drive: My Drive  
      - Folder ID: Specific Google Drive folder (`1TnDibwPPPUm3VbmETiqWDVhtaUTLJ6mn`)  
      - Credentials: Google Drive OAuth2 account  
    - Inputs: Base64 video data from Download node.  
    - Outputs: Confirmation of file upload.  
    - Potential Failures: Authentication errors, insufficient permissions, quota limits on Drive.

---

#### 2.5 Batch Processing and Loop Management

- **Overview:**  
This block manages the overall batch processing and looping of multiple video generation items through the workflow.

- **Nodes Involved:**  
  - Loop Over Items (also part of 2.2)  
  - Connections among nodes ensure continuous flow from input to upload with batching.

- **Node Details:**

  - **Loop Over Items**  
    - Already described in 2.2, but also routes outputs differently:  
      - First output path is empty (no direct output)  
      - Second output path leads to Create Video node to start processing batches  
      - After Upload, the batch processing loop continues to process next items.  
    - Enables parallel or sequential processing without overwhelming API or system resources.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                            | Input Node(s)                | Output Node(s)              | Sticky Note                                                  |
|-------------------------|---------------------|------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------|
| Get prompt from google sheet | Google Sheets       | Fetch AI video prompts from Google Sheet | None                        | Duplicate Rows2             | Details on Google Sheets config and purpose                  |
| Duplicate Rows2          | Code                | Duplicate prompts to create variations   | Get prompt from google sheet | Loop Over Items             | JavaScript code snippet for duplication; link to sample sheet|
| Loop Over Items          | SplitInBatches      | Batch processing of prompt variations    | Duplicate Rows2              | Create Video, empty path    | Batch processing details; API rate limit management          |
| Create Video             | HTTP Request (POST) | Submit video generation requests to API  | Loop Over Items              | Get Video URL               | API endpoint, auth details, body params explained           |
| Get Video URL            | HTTP Request (GET)  | Poll status of video generation task     | Create Video, Wait           | Switch                     | Polling logic, long timeout, API status retrieval            |
| Switch                  | Switch              | Route based on video generation status   | Get Video URL                | Download Video, Wait        | Status-based routing rules and workflow control              |
| Wait                    | Wait                | Delay before rechecking status           | Switch                      | Get Video URL               | 30s wait with webhook ID for pause/resume                    |
| Download Video as Base64 | HTTP Request (GET)  | Download completed video as Base64       | Switch (Completed path)      | Upload to Google Drive1     | Downloads final video file                                    |
| Upload to Google Drive1  | Google Drive        | Upload video to Google Drive folder       | Download Video as Base64     | Loop Over Items             | Dynamic file naming and folder placement                      |
| Sticky Note17            | Sticky Note         | Contact info for help/customization      | None                        | None                       | Contact: robert@ynteractive.com; LinkedIn link               |
| Sticky Note2             | Sticky Note         | Details on Google Sheets and Code node   | None                        | None                       | Node config details including sample Google Sheet link       |
| Sticky Note3             | Sticky Note         | Details on Loop, Create Video, Get Video URL | None                    | None                       | API request details and batch processing notes               |
| Sticky Note4             | Sticky Note         | Details on Switch, Wait, Download, Upload | None                      | None                       | Status routing and upload process explained                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Operation: Read → Select your Google Sheet document and the worksheet (usually Sheet1).  
   - Credentials: Connect your Google Sheets OAuth2 account.  
   - Purpose: To fetch the list of video prompts with metadata.

2. **Add Code Node to Duplicate Rows**  
   - Type: Code (JavaScript)  
   - Paste the following code to duplicate each prompt twice with run identifiers:  
     ```javascript
     const original = items[0].json;
     return [
       { json: { ...original, run: 1 } },
       { json: { ...original, run: 2 } },
     ];
     ```  
   - Connect output of Google Sheets to this node.

3. **Add SplitInBatches Node**  
   - Type: SplitInBatches  
   - Use default batch size (adjust if needed for API limits).  
   - Connect Code node output to this node.

4. **Add HTTP Request Node for Video Creation**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.freepik.com/v1/ai/image-to-video/minimax-hailuo-02-768p`  
   - Authentication: HTTP Header Auth with your Freepik API credentials.  
   - Body Parameters: Add parameter named `prompt` with value set to the prompt field from input (`={{ $json.Prompt }}`).  
   - Connect SplitInBatches output (second output) to this node.

5. **Add HTTP Request Node to Get Video URL**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.freepik.com/v1/ai/image-to-video/minimax-hailuo-02-768p/{{ $json.data.task_id }}`  
   - Timeout: 120,000 ms  
   - Authentication: Same HTTP Header Auth credentials.  
   - Connect Create Video node output to this node.

6. **Add Switch Node to Route by Status**  
   - Type: Switch  
   - Add rules based on `{{$json.data.status}}`:  
     - Completed → output 1  
     - Failed → output 2 (optional handling)  
     - Created → output 3  
     - In Progress → output 4  
   - Connect Get Video URL node output to this node.

7. **Add Wait Node**  
   - Type: Wait  
   - Wait Time: 30 seconds  
   - Webhook Enabled: Yes (auto-generated)  
   - Connect Switch node outputs for Created and In Progress statuses to this node.

8. **Loop Back from Wait Node to Get Video URL Node**  
   - Connect Wait node output to Get Video URL node input to poll again.

9. **Add HTTP Request Node to Download Video as Base64**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $json.data.generated[0] }}` (first video URL)  
   - Connect Switch node Completed output to this node.

10. **Add Google Drive Node to Upload Video**  
    - Type: Google Drive (Upload)  
    - Operation: Upload  
    - Set file name dynamically: `video - {{ $('Get prompt from google sheet').item.json.Name }} - {{ $('Duplicate Rows2').item.json.run }}`  
    - Drive: My Drive  
    - Folder ID: Your desired Google Drive folder ID  
    - Credentials: Google Drive OAuth2 account  
    - Connect Download Video node output to this node.

11. **Loop Upload Output Back to SplitInBatches for Next Items**  
    - Connect Upload to Google Drive node output back to SplitInBatches node main input to continue processing next batch.

12. **Finalize Workflow**  
    - Add necessary sticky notes for documentation and contact info.  
    - Test with valid prompts and valid API credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Contact for help or customization: robert@ynteractive.com                                           | Email and LinkedIn: [LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/)                 |
| Sample Google Sheet for prompts available at: https://docs.google.com/spreadsheets/d/1_u9IxEZINcwKQB15Rfx7C1hM71zeDST58Fz3nRHTCUY/edit?usp=sharing | Used by "Get prompt from google sheet" node                                                        |
| Freepik Minimax Hailuo API documentation for AI video generation is needed for advanced customization | API endpoint: `https://api.freepik.com/v1/ai/image-to-video/minimax-hailuo-02-768p`                 |
| Wait node uses webhook ID for pause/resume to avoid long-running executions                          | Improves reliability in status polling loop                                                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.