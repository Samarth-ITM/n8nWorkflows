Copy Viral Reels with Gemini AI

https://n8nworkflows.xyz/workflows/copy-viral-reels-with-gemini-ai-2993


# Copy Viral Reels with Gemini AI

### 1. Workflow Overview

This workflow, titled **"Copy Viral Reels with Gemini AI"**, automates the process of fetching, analyzing, and recording insights from Instagram Reels videos. It is designed primarily for social media analysts, digital marketers, and content creators who want to leverage AI-driven analysis to understand and replicate viral video content efficiently.

The workflow is logically divided into two main scenarios:

- **Scenario 1: Video Analysis and AI Processing**  
  This block handles fetching a specific video from Airtable, uploading it to Gemini AI for content analysis, and saving the AI-generated insights back into Airtable.

- **Scenario 2: Data Retrieval and Video Record Creation**  
  This block automates the retrieval of Instagram Reels from creators listed in Airtable, processes the videos to extract key metadata, sorts and limits the results, and creates new video records in Airtable for further analysis.

The workflow also includes scheduling and manual triggers, integration with external services (Airtable, Apify Instagram Scraper, Gemini AI), and sub-workflow execution for detailed video analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Scenario 1: Video Analysis and AI Processing

**Overview:**  
This block is responsible for analyzing a single Instagram Reel video using Gemini AI. It fetches the video record from Airtable, generates an upload URL for Gemini, uploads the video file, sends a prompt with the video to Gemini for AI content generation, and finally updates the Airtable record with the AI-generated guideline.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Get Video  
- Gemini - Generate Upload URL  
- Download File  
- Gemini - Upload File  
- Save Values  
- Wait  
- Set Prompt  
- Gemini - Ask Questions  
- Set Guideline  

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for this scenario when called by another workflow or sub-workflow.  
  - Configuration: Passes input data through to the next node.  
  - Inputs: External workflow call  
  - Outputs: To "Get Video" node  
  - Edge Cases: Missing or malformed input data may cause failures.

- **Get Video**  
  - Type: Airtable (Read)  
  - Role: Retrieves a specific video record from Airtable by ID.  
  - Configuration: Uses the video record ID from input to fetch details from the "Videos" table.  
  - Inputs: Video record ID from trigger  
  - Outputs: Video metadata including file URL and size  
  - Edge Cases: Record not found, Airtable API errors, invalid ID.

- **Gemini - Generate Upload URL**  
  - Type: HTTP Request  
  - Role: Requests a resumable upload URL from Gemini AI for the video file.  
  - Configuration: POST to Gemini upload endpoint with file display name and headers for upload protocol.  
  - Inputs: Video file metadata from "Get Video"  
  - Outputs: Upload URL in response headers  
  - Edge Cases: Authentication errors (invalid token), network timeouts, malformed request.

- **Download File**  
  - Type: HTTP Request  
  - Role: Downloads the video file from the URL obtained in "Get Video".  
  - Configuration: GET request with response format set to file (binary).  
  - Inputs: Video URL from "Get Video"  
  - Outputs: Binary video data for upload  
  - Edge Cases: File not found, network errors, large file size causing timeouts.

- **Gemini - Upload File**  
  - Type: HTTP Request  
  - Role: Uploads the binary video file to Gemini using the resumable upload URL.  
  - Configuration: POST with binary data, headers for content length and upload command.  
  - Inputs: Binary video data from "Download File", upload URL from "Gemini - Generate Upload URL"  
  - Outputs: Confirmation of upload and file URI  
  - Edge Cases: Upload interruptions, token expiration, incorrect headers.

- **Save Values**  
  - Type: Set  
  - Role: Stores important values from upload response for later use.  
  - Configuration: Saves Gemini file URI, MIME type, and Airtable record ID.  
  - Inputs: Upload response JSON  
  - Outputs: Prepared data for next steps  
  - Edge Cases: Missing fields in upload response.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 60 seconds to ensure file processing readiness.  
  - Configuration: Fixed 60-second delay.  
  - Inputs: Triggered after "Save Values"  
  - Outputs: To "Set Prompt"  
  - Edge Cases: Delay may be insufficient if processing is slow.

- **Set Prompt**  
  - Type: Set  
  - Role: Defines the AI prompt text for Gemini analysis.  
  - Configuration: Static prompt string describing parameters to analyze in the video (background, pose, text, clothes, context, participants).  
  - Inputs: None (triggered after wait)  
  - Outputs: Prompt string for Gemini request  
  - Edge Cases: Prompt text must be clear and relevant for useful AI output.

- **Gemini - Ask Questions**  
  - Type: HTTP Request  
  - Role: Sends the prompt and uploaded video file reference to Gemini AI for content generation.  
  - Configuration: POST request to Gemini content generation endpoint with prompt and file data.  
  - Inputs: Prompt from "Set Prompt", file URI and MIME type from "Save Values"  
  - Outputs: AI-generated content response  
  - Edge Cases: API rate limits, token errors, malformed request.

- **Set Guideline**  
  - Type: Airtable (Update)  
  - Role: Updates the original video record in Airtable with the AI-generated guideline text.  
  - Configuration: Matches record by ID and updates the "Guideline" field with AI response text.  
  - Inputs: AI response from "Gemini - Ask Questions", record ID from "Get Video"  
  - Outputs: Confirmation of update  
  - Edge Cases: Airtable update failures, mismatched record IDs.

---

#### 2.2 Scenario 2: Data Retrieval and Video Record Creation

**Overview:**  
This block automates fetching Instagram Reels from creators listed in Airtable, processes and filters the videos by views, and creates new video records in Airtable for subsequent analysis.

**Nodes Involved:**  
- Schedule Trigger  
- Search Creators  
- Loop Over Items  
- Apify - Fetch Reels  
- Save Fields  
- Sort  
- Limit  
- Create Video  
- Execute Workflow1  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow on a monthly interval to refresh data.  
  - Configuration: Interval set to trigger every month.  
  - Inputs: None (time-based trigger)  
  - Outputs: To "Search Creators"  
  - Edge Cases: Time zone considerations, missed triggers if n8n is offline.

- **Search Creators**  
  - Type: Airtable (Search)  
  - Role: Retrieves a list of Instagram creators from the "Creators" table in Airtable.  
  - Configuration: Search operation without filters, returns limited results.  
  - Inputs: Trigger from schedule  
  - Outputs: List of creators with Instagram usernames  
  - Edge Cases: Airtable API limits, empty results.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes creators one by one to fetch their reels.  
  - Configuration: Default batch size (1 by default)  
  - Inputs: List of creators from "Search Creators" or from sub-workflow execution loop  
  - Outputs: To "Apify - Fetch Reels" for each creator  
  - Edge Cases: Large creator lists may slow processing.

- **Apify - Fetch Reels**  
  - Type: HTTP Request  
  - Role: Calls Apify Instagram Scraper API to fetch recent reels for a given creator.  
  - Configuration: POST request with JSON body specifying Instagram URL, date filters, and result limits (20 stories).  
  - Inputs: Instagram username from current batch item  
  - Outputs: List of reels with metadata (video URL, views, caption)  
  - Edge Cases: Invalid token, API rate limits, Instagram URL changes.

- **Save Fields**  
  - Type: Set  
  - Role: Extracts and formats key fields from fetched reels for further processing.  
  - Configuration: Creates JSON with video URL, view count, caption, and creator name.  
  - Inputs: Reels data from Apify  
  - Outputs: Structured data for sorting  
  - Edge Cases: Missing fields in API response.

- **Sort**  
  - Type: Sort  
  - Role: Sorts reels descending by view count to prioritize popular videos.  
  - Configuration: Sort by "views" field descending.  
  - Inputs: Structured reels data  
  - Outputs: Sorted list  
  - Edge Cases: Non-numeric or missing view counts.

- **Limit**  
  - Type: Limit  
  - Role: Restricts the number of reels processed further (default limit unspecified).  
  - Configuration: Default limit node (no explicit limit set in JSON).  
  - Inputs: Sorted reels list  
  - Outputs: Limited subset of reels  
  - Edge Cases: No limit set may process too many items.

- **Create Video**  
  - Type: Airtable (Create)  
  - Role: Creates new video records in Airtable "Videos" table with extracted metadata.  
  - Configuration: Maps video URL, views, caption, and creator to Airtable fields.  
  - Inputs: Limited reels data  
  - Outputs: Confirmation of record creation  
  - Edge Cases: Airtable API errors, duplicate records.

- **Execute Workflow1**  
  - Type: Execute Workflow  
  - Role: Calls a sub-workflow ("Gemini - Video Analysis 2") to analyze each newly created video.  
  - Configuration: Passes created video data to sub-workflow for processing.  
  - Inputs: Newly created video record data  
  - Outputs: Loops back to "Loop Over Items" to continue processing creators  
  - Edge Cases: Sub-workflow failures, data mismatch.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                              | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                   |
|----------------------------|-------------------------|----------------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger        | Initiates monthly data retrieval              |                              | Search Creators             |                                                                                              |
| Search Creators             | Airtable (Search)       | Fetches creators list from Airtable           | Schedule Trigger             | Loop Over Items             |                                                                                              |
| Loop Over Items            | Split In Batches        | Processes creators one by one                  | Search Creators, Execute Workflow1 | Apify - Fetch Reels, (second output to Apify - Fetch Reels) |                                                                                              |
| Apify - Fetch Reels         | HTTP Request            | Fetches Instagram Reels via Apify API          | Loop Over Items              | Save Fields                 | ### Replace Apify token                                                                       |
| Save Fields                | Set                     | Extracts key reel fields                        | Apify - Fetch Reels          | Sort                        |                                                                                              |
| Sort                       | Sort                    | Sorts reels by descending views                | Save Fields                  | Limit                       |                                                                                              |
| Limit                      | Limit                   | Limits number of reels processed                | Sort                        | Create Video                |                                                                                              |
| Create Video               | Airtable (Create)       | Creates new video records in Airtable          | Limit                       | Execute Workflow1           |                                                                                              |
| Execute Workflow1          | Execute Workflow        | Calls sub-workflow for video analysis          | Create Video                | Loop Over Items             |                                                                                              |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point for Scenario 1 video analysis      |                              | Get Video                  |                                                                                              |
| Get Video                  | Airtable (Read)         | Retrieves video record by ID                    | When Executed by Another Workflow | Gemini - Generate Upload URL |                                                                                              |
| Gemini - Generate Upload URL | HTTP Request            | Requests upload URL from Gemini AI              | Get Video                   | Download File               | ### Replace Google token                                                                     |
| Download File              | HTTP Request            | Downloads video file from URL                    | Gemini - Generate Upload URL | Gemini - Upload File        |                                                                                              |
| Gemini - Upload File       | HTTP Request            | Uploads video file to Gemini AI                  | Download File               | Save Values                 |                                                                                              |
| Save Values                | Set                     | Saves Gemini file URI and MIME type             | Gemini - Upload File         | Wait                        |                                                                                              |
| Wait                       | Wait                    | Waits 60 seconds for file processing             | Save Values                 | Set Prompt                  |                                                                                              |
| Set Prompt                 | Set                     | Sets AI prompt text for video analysis          | Wait                        | Gemini - Ask Questions      | ### Set own prompt for analysis                                                              |
| Gemini - Ask Questions     | HTTP Request            | Sends prompt and video to Gemini AI             | Set Prompt                  | Set Guideline               | ### Replace Google token                                                                     |
| Set Guideline              | Airtable (Update)       | Updates Airtable video record with AI guideline | Gemini - Ask Questions       |                             |                                                                                              |
| Sticky Note1               | Sticky Note             | Scenario 1 label                                |                              |                             |                                                                                              |
| Sticky Note                | Sticky Note             | Scenario 2 label                                |                              |                             |                                                                                              |
| Sticky Note2               | Sticky Note             | Reminder to replace Apify token                 |                              |                             | ### Replace Apify token                                                                       |
| Sticky Note3               | Sticky Note             | Reminder to replace Google token                |                              |                             | ### Replace Google token                                                                     |
| Sticky Note4               | Sticky Note             | Reminder to replace Google token                |                              |                             | ### Replace Google token                                                                     |
| Sticky Note5               | Sticky Note             | Reminder to replace Google token                |                              |                             | ### Replace Google token                                                                     |
| Sticky Note6               | Sticky Note             | Reminder to set own AI prompt                    |                              |                             | ### Set own prompt for analysis                                                              |
| Sticky Note7               | Sticky Note             | Link to setup video                               |                              |                             | ### ... or watch set up video [8 min] [https://youtu.be/SQPPM0KLsrM]                         |
| Sticky Note8               | Sticky Note             | Workflow branding and overview                    |                              |                             | ![5min Logo](https://www.5minai.com/logo.png) Made by Mark Shcherbakov from 5minAI community |
| Sticky Note9               | Sticky Note             | Setup instructions                                |                              |                             | 1. Create accounts 2. Replace credentials 3. Configure data flow                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger monthly (interval: months).  
   - Connect output to "Search Creators".

2. **Create an Airtable Search node ("Search Creators")**  
   - Connect to Schedule Trigger.  
   - Configure credentials with Airtable API key.  
   - Base: Your Airtable base containing creators.  
   - Table: "Creators" table.  
   - Operation: Search (no filters to fetch all creators).  
   - Output: List of creators with Instagram usernames.  
   - Connect output to "Loop Over Items".

3. **Create a Split In Batches node ("Loop Over Items")**  
   - Connect to "Search Creators".  
   - Default batch size (1).  
   - First output connects to "Apify - Fetch Reels".  
   - Second output connects back to "Apify - Fetch Reels" (for looping after sub-workflow).  

4. **Create an HTTP Request node ("Apify - Fetch Reels")**  
   - Connect to "Loop Over Items".  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/apify~instagram-scraper/run-sync-get-dataset-items`  
   - Body (JSON):  
     ```json
     {
       "addParentData": false,
       "directUrls": ["https://www.instagram.com/{{ $json['Instagram Username'] }}/"],
       "enhanceUserSearchWithFacebookPage": false,
       "isUserReelFeedURL": false,
       "isUserTaggedFeedURL": false,
       "onlyPostsNewerThan": "{{ new Date().toISOString().slice(0, 10).replace(/-\d+$/, '-01') }}",
       "resultsLimit": 20,
       "resultsType": "stories"
     }
     ```  
   - Query Parameter: `token` with your Apify API token.  
   - Connect output to "Save Fields".

5. **Create a Set node ("Save Fields")**  
   - Connect to "Apify - Fetch Reels".  
   - Mode: Raw JSON output.  
   - JSON Output:  
     ```json
     {
       "url": {{ JSON.stringify($json.videoUrl) }},
       "views": {{ $json.videoViewCount }},
       "caption": {{ JSON.stringify($json.caption) }},
       "creator": "{{ $('Loop Over Items').item.json.Name }}"
     }
     ```  
   - Connect output to "Sort".

6. **Create a Sort node ("Sort")**  
   - Connect to "Save Fields".  
   - Sort by "views" descending.  
   - Connect output to "Limit".

7. **Create a Limit node ("Limit")**  
   - Connect to "Sort".  
   - (Optional) Set max items to process per creator.  
   - Connect output to "Create Video".

8. **Create an Airtable Create node ("Create Video")**  
   - Connect to "Limit".  
   - Configure Airtable credentials.  
   - Base: Your Airtable base.  
   - Table: "Videos".  
   - Map fields:  
     - Video: Array with URL from "Save Fields".  
     - Views: Number from "views".  
     - Caption: Text from "caption".  
     - Creator: Array with creator name.  
   - Connect output to "Execute Workflow1".

9. **Create an Execute Workflow node ("Execute Workflow1")**  
   - Connect to "Create Video".  
   - Select sub-workflow ID for detailed video analysis (e.g., "Gemini - Video Analysis 2").  
   - Connect output back to "Loop Over Items" (second output) to continue processing creators.

10. **Create an Execute Workflow Trigger node ("When Executed by Another Workflow")**  
    - This node is the entry point for Scenario 1.  
    - Connect output to "Get Video".

11. **Create an Airtable Read node ("Get Video")**  
    - Connect to "When Executed by Another Workflow".  
    - Configure Airtable credentials.  
    - Base: Your Airtable base.  
    - Table: "Videos".  
    - Use input ID to fetch video record.  
    - Connect output to "Gemini - Generate Upload URL".

12. **Create an HTTP Request node ("Gemini - Generate Upload URL")**  
    - Connect to "Get Video".  
    - Method: POST  
    - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files`  
    - Headers:  
      - `X-Goog-Upload-Protocol`: `resumable`  
      - `X-Goog-Upload-Command`: `start`  
      - `X-Goog-Upload-Header-Content-Length`: video file size from "Get Video"  
      - `X-Goog-Upload-Header-Content-Type`: video MIME type from "Get Video"  
    - Query Parameter: `key` with your Google API token.  
    - Body JSON:  
      ```json
      {
        "file": {
          "display_name": "{{ $json.Video[0].filename }}"
        }
      }
      ```  
    - Connect output to "Download File".

13. **Create an HTTP Request node ("Download File")**  
    - Connect to "Gemini - Generate Upload URL".  
    - Method: GET  
    - URL: Video URL from "Get Video".  
    - Response Format: File (binary).  
    - Connect output to "Gemini - Upload File".

14. **Create an HTTP Request node ("Gemini - Upload File")**  
    - Connect to "Download File".  
    - Method: POST  
    - URL: Upload URL from "Gemini - Generate Upload URL" response headers.  
    - Headers:  
      - `Content-Length`: video file size  
      - `X-Goog-Upload-Offset`: 0  
      - `X-Goog-Upload-Command`: `upload, finalize`  
    - Query Parameter: `key` with your Google API token.  
    - Content Type: binaryData  
    - Input Data Field Name: `data` (binary from "Download File").  
    - Connect output to "Save Values".

15. **Create a Set node ("Save Values")**  
    - Connect to "Gemini - Upload File".  
    - Assign:  
      - `gemini_file_url`: file URI from upload response.  
      - `mimeType`: MIME type from upload response.  
      - `airtable_rec_id`: ID from "Get Video".  
    - Connect output to "Wait".

16. **Create a Wait node ("Wait")**  
    - Connect to "Save Values".  
    - Duration: 60 seconds.  
    - Connect output to "Set Prompt".

17. **Create a Set node ("Set Prompt")**  
    - Connect to "Wait".  
    - Assign prompt string describing video analysis parameters (background, pose, text, clothes, context, participants) as per example in original workflow.  
    - Connect output to "Gemini - Ask Questions".

18. **Create an HTTP Request node ("Gemini - Ask Questions")**  
    - Connect to "Set Prompt".  
    - Method: POST  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-pro:generateContent`  
    - Query Parameter: `key` with your Google API token.  
    - Body JSON:  
      ```json
      {
        "contents": [
          {
            "parts": [
              { "text": "{{ $json.prompt }}" },
              {
                "file_data": {
                  "mime_type": "{{ $('Save Values').item.json.mimeType }}",
                  "file_uri": "{{ $('Save Values').item.json.gemini_file_url }}"
                }
              }
            ]
          }
        ]
      }
      ```  
    - Connect output to "Set Guideline".

19. **Create an Airtable Update node ("Set Guideline")**  
    - Connect to "Gemini - Ask Questions".  
    - Configure Airtable credentials.  
    - Base: Your Airtable base.  
    - Table: "Videos".  
    - Match record by ID from "Get Video".  
    - Update "Guideline" field with AI-generated text from Gemini response.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow created by [Mark Shcherbakov](https://www.linkedin.com/in/marklowcoding/) from the community [5minAI](https://www.skool.com/5minai).                                                                                     | Branding and author credit                                                                       |
| Detailed video guide available: [YouTube Setup Video (8 min)](https://youtu.be/SQPPM0KLsrM) with thumbnail ![Youtube Thumbnail](https://res.cloudinary.com/de9jgixzm/image/upload/v1740390927/Youtube%20Thumbs/Video_21_-_Philipp_Instagram_Blur.png) | Setup instructions and walkthrough                                                              |
| Replace all placeholder tokens (`<your_token_here>`) with your actual API keys for Apify, Google Gemini, and Airtable credentials before running the workflow.                                                                     | Credential setup reminder                                                                        |
| The AI prompt in "Set Prompt" node can be customized to tailor the video analysis parameters according to user needs.                                                                                                             | Customization tip                                                                                |
| The workflow uses a sub-workflow ("Gemini - Video Analysis 2") for detailed video analysis; ensure this sub-workflow is imported and configured properly with expected input/output parameters.                                     | Sub-workflow dependency                                                                          |
| Airtable base and tables used: "Creators" for Instagram usernames, "Videos" for storing video metadata and AI analysis results. Ensure these tables have the required fields as mapped in the workflow nodes.                      | Data schema requirement                                                                          |
| Apify Instagram Scraper API is used to fetch Instagram Reels data; ensure the token is valid and the API endpoint is accessible.                                                                                                   | External API dependency                                                                          |
| Gemini AI API requires Google API token with appropriate permissions for generative language and file upload services.                                                                                                            | External API dependency                                                                          |

---

This document provides a complete and detailed reference for understanding, reproducing, and modifying the "Copy Viral Reels with Gemini AI" workflow in n8n. It covers all nodes, their configurations, data flows, and integration points, enabling advanced users and AI agents to work effectively with this automation.