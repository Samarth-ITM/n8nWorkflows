Generate AI Avatar Videos with HeyGen and Google Sheets Integration

https://n8nworkflows.xyz/workflows/generate-ai-avatar-videos-with-heygen-and-google-sheets-integration-5027


# Generate AI Avatar Videos with HeyGen and Google Sheets Integration

### 1. Workflow Overview

This workflow automates the generation of AI avatar videos using HeyGen's API and integrates with Google Sheets to manage input scripts and store resulting video URLs. It is designed for users who want to create personalized AI avatar videos based on text scripts and automatically log video output links for easy access and distribution.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Google Sheets Input Retrieval:** Fetch input text scripts from a Google Sheet.
- **1.3 AI Video Creation Request:** Send a POST request to HeyGen's API to generate a video with specified avatar and voice parameters.
- **1.4 Video Details Retrieval:** Query HeyGen's API to get the generated video details including the video URL.
- **1.5 Google Sheets Output Update:** Append the generated video URL back into the Google Sheet for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1.1: Input Reception

- **Overview:**  
  This block starts the workflow manually via a user-initiated trigger, enabling controlled execution.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution manually on demand.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connected to the Google Sheets node to start data retrieval  
    - Edge Cases: None significant; user must manually trigger workflow to run.

#### 2.2 Block 1.2: Google Sheets Input Retrieval

- **Overview:**  
  Retrieves script/voice text data from a specified Google Sheet to be used as input for AI video generation.

- **Nodes Involved:**  
  - Google Sheets (input node)

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets node  
    - Role: Reads data from a Google Spreadsheet where scripts are stored.  
    - Configuration:  
      - Document ID: `1xqIDGIYCCHlZyfWYMMHBisz4YggRKycygWzBfYaYI08`  
      - Sheet name: `gid=0` (first sheet)  
      - Operation: Default read/list operation (implied)  
      - Credentials: Google Sheets OAuth2 account configured  
    - Inputs: From Manual Trigger  
    - Outputs: Connected to HTTP Request node for video creation  
    - Edge Cases:  
      - OAuth token expiration or invalid credentials  
      - Empty or malformed sheet data  
      - API rate limits from Google Sheets  

#### 2.3 Block 1.3: AI Video Creation Request

- **Overview:**  
  Sends a POST request to HeyGen’s API to generate an AI video using preset avatar and voice parameters combined with input text.

- **Nodes Involved:**  
  - creating Video

- **Node Details:**

  - **creating Video**  
    - Type: HTTP Request node  
    - Role: Sends POST request to HeyGen API endpoint `/v2/video/generate`  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.heygen.com/v2/video/generate`  
      - Authentication: HTTP Header Auth with API key credential  
      - Headers: Accept: application/json  
      - Body (JSON):  
        - Specifies one video input with:  
          - Character as an avatar with fixed avatar_id `84c521ab0e4b4527a33423e1b813ea0c` and style `normal`  
          - Voice properties with fixed voice_id `2aa0fed15bbf424bb3a4f9c9537ab058`, speed 1.1  
          - Input text hardcoded as `"hey, this is sagar and it is my avatar"` (Note: This is static, does not reference input from Google Sheets)  
        - Dimension: 1280x720  
    - Inputs: From Google Sheets node (although static input text used)  
    - Outputs: Connected to Get Video node to fetch video details  
    - Edge Cases:  
      - API authentication failure (invalid/missing API key)  
      - Network timeout or API downtime  
      - Malformed JSON body or validation errors from API  
      - Static input text limits usefulness; ideally should use dynamic text from Google Sheets.

#### 2.4 Block 1.4: Video Details Retrieval

- **Overview:**  
  Retrieves detailed information about the generated video including the final video URL.

- **Nodes Involved:**  
  - Get Video

- **Node Details:**

  - **Get Video**  
    - Type: HTTP Request node  
    - Role: Sends GET request to HeyGen API endpoint `/v2/avatar/avatar_id/details`  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.heygen.com/v2/avatar/avatar_id/details` (avatar_id is static in URL)  
      - Authentication: HTTP Header Auth with same API key credential  
      - Header parameter: `video_id` is dynamically set from output of previous node (`={{ $json.data.video_id }}`)  
    - Inputs: From creating Video node (receives video_id)  
    - Outputs: Connected to Google Sheets1 node for output update  
    - Edge Cases:  
      - Missing or invalid video_id leading to 404 or error response  
      - API rate limits or authentication failure  
      - Network errors or response parsing failures  

#### 2.5 Block 1.5: Google Sheets Output Update

- **Overview:**  
  Appends the generated video URL to the Google Sheet, associating it with the input script.

- **Nodes Involved:**  
  - Google Sheets1

- **Node Details:**

  - **Google Sheets1**  
    - Type: Google Sheets node  
    - Role: Appends new rows to the spreadsheet with the final video URL linked to the script  
    - Configuration:  
      - Document ID: same Google Sheet as input  
      - Sheet name: same sheet (`gid=0`)  
      - Operation: Append row  
      - Mapping:  
        - Column "Final Video" is set to `={{ $json.data.video_url }}` from Get Video node output  
      - Matching Columns: "New Script" but set to append rather than update  
      - Credentials: Google Sheets OAuth2 account configured  
    - Inputs: From Get Video node  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - Append failures due to permission issues or invalid sheet ID  
      - Data type mismatches or schema changes in the sheet  
      - API rate limits or OAuth token expiration  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                       | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                                                            |
|-------------------------|---------------------|------------------------------------|---------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts workflow on demand           | -                         | Google Sheets            | 👈 Set up Manual Trigger Node. This trigger will be activated. Follow the steps (YouTube video): https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| Google Sheets           | Google Sheets        | Reads input scripts from spreadsheet| When clicking ‘Test workflow’ | creating Video           | 👈 Set up Google Sheets Node. Set up credentials. Create sheet with Script/voice text and Final Video Link columns. YouTube video: https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| creating Video          | HTTP Request         | Sends POST to generate AI video     | Google Sheets             | Get Video                | 👈 Set up HTTP Node. Method: POST URL: https://api.heygen.com/v2/video/generate. Set up authentication. YouTube video: https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| Get Video               | HTTP Request         | Fetches video details from HeyGen   | creating Video            | Google Sheets1           | 👈 Set up HTTP Node. Method: GET URL: https://api.heygen.com/v2/avatar/avatar_id/details. Use video_id param from prior node. YouTube video: https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| Google Sheets1          | Google Sheets        | Appends final video URL to sheet    | Get Video                 | -                       | 👈 Set up Google Sheets Node. Set up credentials. Append final video link to sheet. YouTube video: https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| Sticky Note1            | Sticky Note          | Instructional note on Manual Trigger| -                         | -                       | 👈 Set up Manual Trigger Node. This trigger will be activated. Follow the steps (YouTube video): https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| Sticky Note3            | Sticky Note          | Overview note about workflow        | -                         | -                       | Your AI Avatar Videos Automation Agent with Google Sheets, HTTP Nodes (Easy to set-up in 5 steps only)                                  |
| Sticky Note5            | Sticky Note          | Instructions for HTTP POST node setup| -                         | -                       | 👈 Set up HTTP Node. Method : Post URL : https://api.heygen.com/v2/video/generate. Setup authentication. YouTube video: https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| Sticky Note6            | Sticky Note          | Instructions for initial Google Sheets node setup| -                         | -                       | 👈 Set up Google Sheets Node. Set up credentials. Create sheet with Script/voice text, Final Video Link columns. YouTube video: https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| Sticky Note9            | Sticky Note          | Instructions for HTTP GET node setup| -                         | -                       | 👈 Set up HTTP Node. Method : Get URL : https://api.heygen.com/v2/avatar/avatar_id/details. Use video_id param. YouTube video: https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |
| Sticky Note10           | Sticky Note          | Instructions for Google Sheets1 setup| -                         | -                       | 👈 Set up Google Sheets Node. Set up credentials. Append final video link to sheet. YouTube video: https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow.  
   - No additional configuration needed.

2. **Create a Google Sheets Node (Input)**  
   - Type: Google Sheets  
   - Operation: Read/List rows  
   - Set Document ID to your Google Sheet (e.g., `1xqIDGIYCCHlZyfWYMMHBisz4YggRKycygWzBfYaYI08`)  
   - Set Sheet Name to the relevant sheet (e.g., `gid=0`)  
   - Authenticate with OAuth2 credentials for Google Sheets API  
   - Connect Manual Trigger node output to this node.

3. **Create an HTTP Request Node for Video Generation ("creating Video")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Authentication: HTTP Header Auth with HeyGen API key credentials  
   - Headers: Add header `Accept: application/json`  
   - Body: Raw JSON (application/json) with structure:
     ```json
     {
       "video_inputs": [
         {
           "character": {
             "type": "avatar",
             "avatar_id": "84c521ab0e4b4527a33423e1b813ea0c",
             "avatar_style": "normal"
           },
           "voice": {
             "type": "text",
             "input_text": "hey, this is sagar and it is my avatar",
             "voice_id": "2aa0fed15bbf424bb3a4f9c9537ab058",
             "speed": 1.1
           }
         }
       ],
       "dimension": {
         "width": 1280,
         "height": 720
       }
     }
     ```
   - Connect Google Sheets node output to this node.

4. **Create an HTTP Request Node to Get Video Details ("Get Video")**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.heygen.com/v2/avatar/avatar_id/details`  
   - Authentication: Same HTTP Header Auth credentials as above  
   - Add Header parameter:  
     - Name: `video_id`  
     - Value: Expression referencing previous node’s output: `{{$json["data"]["video_id"]}}`  
   - Connect "creating Video" node output to this node.

5. **Create a Google Sheets Node to Append Final Video URL ("Google Sheets1")**  
   - Type: Google Sheets  
   - Operation: Append row  
   - Document ID and Sheet Name same as input node  
   - Map column "Final Video" to the expression `{{$json["data"]["video_url"]}}` from "Get Video" node output  
   - Authenticate with Google Sheets OAuth2 credentials  
   - Connect "Get Video" node output to this node.

6. **Verify Credentials**  
   - HeyGen API key configured in HTTP Header Auth credential in n8n.  
   - Google Sheets OAuth2 credentials set up with proper access to the target spreadsheet.

7. **Run Manual Trigger to test the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Follow the detailed YouTube video walkthrough covering setup and configuration:                           | https://youtu.be/llm60n03x3c?si=vO4jU6EZms-ZpDvL                   |
| Workflow automates AI avatar video creation integrated with Google Sheets for easy management.             | -                                                                  |
| Make sure to create Google Sheet with at least two columns: one for input scripts and one for video URLs. | -                                                                  |
| Static text input in video creation node can be replaced with dynamic script text from Google Sheets for customization. | -                                                                  |
| HeyGen API requires valid API key for authentication in HTTP Header Auth credential in n8n.               | -                                                                  |

---

**Disclaimer:**  
The text provided is sourced exclusively from an automated workflow built with n8n, respecting all content policies. It contains no illegal, offensive, or protected material. All data handled is legal and public.