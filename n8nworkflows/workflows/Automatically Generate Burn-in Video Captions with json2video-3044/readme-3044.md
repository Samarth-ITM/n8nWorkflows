Automatically Generate Burn-in Video Captions with json2video

https://n8nworkflows.xyz/workflows/automatically-generate-burn-in-video-captions-with-json2video-3044


# Automatically Generate Burn-in Video Captions with json2video

### 1. Workflow Overview

This workflow automates the process of adding accurate, synchronized captions to videos by leveraging the Json2Video API. It targets content creators, marketers, educators, and businesses who want to enhance video accessibility, viewer engagement, and SEO without manual subtitle creation.

The workflow is logically divided into two main blocks:

- **1.1 Input Configuration and Caption Request Submission:**  
  Accepts video details (URL, width, height) and sends a POST request to Json2Video to generate captions burned into the video.

- **1.2 Caption Processing Status Monitoring:**  
  Periodically polls the Json2Video API every 10 seconds to check the processing status of the caption generation job, handling errors and completion states accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Configuration and Caption Request Submission

**Overview:**  
This block initializes the workflow with video input parameters and submits a request to the Json2Video API to generate a captioned video.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Config (Set)  
- json2video - Add Captions (HTTP Request)  
- Wait (Wait)  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow for testing or execution.  
  - *Configuration:* No parameters; triggers workflow on manual activation.  
  - *Inputs:* None  
  - *Outputs:* Connects to Config node.  
  - *Edge Cases:* None specific; user must trigger manually.

- **Config**  
  - *Type:* Set  
  - *Role:* Defines and assigns input parameters for the video: URL, width, and height.  
  - *Configuration:*  
    - `video_url`: URL of the publicly accessible video (default example provided).  
    - `width`: Video width in pixels (default "1080").  
    - `height`: Video height in pixels (default "1920").  
  - *Inputs:* From Manual Trigger  
  - *Outputs:* Connects to json2video - Add Captions node.  
  - *Edge Cases:* Input values must be valid; invalid URLs or dimensions may cause API errors.

- **json2video - Add Captions**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to Json2Video API to initiate caption generation on the provided video.  
  - *Configuration:*  
    - URL: `https://api.json2video.com/v2/movies`  
    - Method: POST  
    - Body: JSON payload including video source URL, subtitle style settings, language ("en"), and video dimensions dynamically injected from previous node variables.  
    - Authentication: Custom HTTP header with API key (credential named "json2video").  
  - *Key Expressions:*  
    - `{{ $json.video_url }}` for video source URL  
    - `{{ $json.width }}` and `{{ $json.height }}` for video dimensions  
  - *Inputs:* From Config node  
  - *Outputs:* Connects to Wait node  
  - *Edge Cases:*  
    - API authentication failure (invalid or missing API key).  
    - Invalid video URL or unsupported video format.  
    - Network timeouts or API rate limits.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses workflow execution for 10 seconds before polling status.  
  - *Configuration:* Wait time set to 10 seconds.  
  - *Inputs:* From json2video - Add Captions node  
  - *Outputs:* Connects to json2video - Get Status node (in next block)  
  - *Edge Cases:* None significant; ensures API is not polled too frequently.

---

#### 2.2 Caption Processing Status Monitoring

**Overview:**  
This block repeatedly checks the status of the caption generation job by querying the Json2Video API, handling errors, completion, and looping with delay until the job is done.

**Nodes Involved:**  
- json2video - Get Status (HTTP Request)  
- Is Error (If)  
- Handle Error (NoOp)  
- is Completed (If)  
- Output (Set)  
- Wait (Wait) [reused from previous block]  
- Sticky Note1 (Sticky Note)  

**Node Details:**

- **json2video - Get Status**  
  - *Type:* HTTP Request  
  - *Role:* Sends a GET request to Json2Video API to retrieve the current processing status of the video caption job.  
  - *Configuration:*  
    - URL dynamically constructed using the project ID returned from the Add Captions node:  
      `https://api.json2video.com/v2/movies?id={{ $('json2video - Add Captions').first().json.project }}`  
    - Authentication: Same custom HTTP header credentials as Add Captions node.  
  - *Inputs:* From Wait node  
  - *Outputs:* Connects to Is Error node  
  - *Edge Cases:*  
    - API authentication failure.  
    - Invalid or missing project ID.  
    - Network issues or API downtime.

- **Is Error**  
  - *Type:* If  
  - *Role:* Checks if the returned status from the API is "error".  
  - *Configuration:* Condition: `{{$json.movie.status}} == "error"`  
  - *Inputs:* From json2video - Get Status node  
  - *Outputs:*  
    - True branch: Handle Error node  
    - False branch: is Completed node  
  - *Edge Cases:* Correct JSON path must exist; if API response structure changes, expression may fail.

- **Handle Error**  
  - *Type:* NoOp (No Operation)  
  - *Role:* Placeholder node to handle error cases (could be extended to send notifications or logs).  
  - *Inputs:* From Is Error (true branch)  
  - *Outputs:* None (workflow ends or could be extended)  
  - *Edge Cases:* Currently no error handling logic; user may want to add alerts.

- **is Completed**  
  - *Type:* If  
  - *Role:* Checks if the video processing status is "done".  
  - *Configuration:* Condition: `{{$json.movie.status}} == "done"`  
  - *Inputs:* From Is Error (false branch)  
  - *Outputs:*  
    - True branch: Output node  
    - False branch: Wait node (loop back to poll again)  
  - *Edge Cases:* Same as Is Error node regarding JSON path validity.

- **Output**  
  - *Type:* Set  
  - *Role:* Prepares the final output data including the URL of the subtitled video and metadata such as duration, size, dimensions, rendering time, project ID, and remaining quota.  
  - *Configuration:* Assigns multiple fields from the API response JSON:  
    - `url` (final video URL)  
    - `duration`, `size`, `width`, `height`, `rendering_time`, `project`  
    - `remaining_quota.time` (quota info)  
  - *Inputs:* From is Completed (true branch)  
  - *Outputs:* None (end of workflow)  
  - *Edge Cases:* Assumes all fields exist in API response; missing fields may cause errors.

- **Wait** (reused)  
  - *Role:* Delays 10 seconds before next status check if processing not complete.  
  - *Inputs:* From is Completed (false branch)  
  - *Outputs:* Loops back to json2video - Get Status node  
  - *Edge Cases:* None significant.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Documentation block labeled "Check video status" to clarify the purpose of this monitoring section.  
  - *Inputs/Outputs:* None  

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                                  | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                       |
|---------------------------|--------------------|-------------------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Entry point to start workflow manually           | None                         | Config                      |                                                                                                 |
| Config                    | Set                | Defines video URL and dimensions                  | When clicking ‘Test workflow’ | json2video - Add Captions    | # ☝️ Provide Video Details: URL, Width & Height                                                  |
| json2video - Add Captions | HTTP Request       | Sends video and caption request to Json2Video API | Config                       | Wait                        |                                                                                                 |
| Wait                      | Wait               | Pauses 10 seconds before status polling           | json2video - Add Captions / is Completed (false branch) | json2video - Get Status     |                                                                                                 |
| json2video - Get Status   | HTTP Request       | Polls Json2Video API for caption generation status | Wait                         | Is Error                    | ## Check video status                                                                           |
| Is Error                  | If                 | Checks if API returned an error status             | json2video - Get Status       | Handle Error (true), is Completed (false) |                                                                                                 |
| Handle Error              | NoOp               | Placeholder for error handling                      | Is Error (true)               | None                        |                                                                                                 |
| is Completed              | If                 | Checks if caption generation is complete           | Is Error (false)              | Output (true), Wait (false) |                                                                                                 |
| Output                    | Set                | Outputs final video URL and metadata                | is Completed (true)           | None                        |                                                                                                 |
| Sticky Note1              | Sticky Note        | Documentation for status checking block             | None                         | None                        | ## Check video status                                                                           |
| Sticky Note2              | Sticky Note        | Workflow overview, setup instructions, and links   | None                         | None                        | # Automatically Generate Captions for Your Videos with json2video ... [Try json2video for free](https://json2video.com/?afco=manu) |
| Sticky Note3              | Sticky Note        | Instructions to provide video details               | None                         | None                        | # ☝️ Provide Video Details: URL, Width & Height                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow.

2. **Create Set Node (Config)**  
   - Type: Set  
   - Purpose: Define input parameters for the video.  
   - Parameters to set:  
     - `video_url` (string): URL of the publicly accessible video (e.g., `https://aiatelier.s3.eu-west-1.amazonaws.com/workflows-material/json2video/captions-sample.mp4`)  
     - `width` (string): Video width in pixels (e.g., "1080")  
     - `height` (string): Video height in pixels (e.g., "1920")  
   - Connect Manual Trigger output to this node.

3. **Create HTTP Request Node (json2video - Add Captions)**  
   - Type: HTTP Request  
   - Purpose: Submit video and caption request to Json2Video API.  
   - Settings:  
     - HTTP Method: POST  
     - URL: `https://api.json2video.com/v2/movies`  
     - Authentication: Use Custom HTTP Header Auth credentials with your Json2Video API key.  
     - Body Content Type: JSON  
     - Body (JSON):  
       ```json
       {
         "id": "qbaasr7s",
         "resolution": "custom",
         "quality": "high",
         "scenes": [
           {
             "id": "qyjh9lwj",
             "comment": "Scene 1",
             "elements": []
           }
         ],
         "elements": [
           {
             "id": "q6dznzcv",
             "type": "video",
             "src": "{{ $json.video_url }}"
           },
           {
             "id": "q41n9kxp",
             "type": "subtitles",
             "settings": {
               "style": "classic-progressive",
               "font-family": "Oswald",
               "font-size": 140,
               "word-color": "#FCF5C9",
               "shadow-color": "#260B1B",
               "line-color": "#F1E7F4",
               "shadow-offset": 2,
               "box-color": "#260B1B"
             },
             "language": "en"
           }
         ],
         "width": {{ $json.width }},
         "height": {{ $json.height }}
       }
       ```  
   - Connect Config node output to this node.

4. **Create Wait Node (Wait 10 seconds)**  
   - Type: Wait  
   - Purpose: Delay 10 seconds before polling status.  
   - Parameters: Wait time = 10 seconds  
   - Connect json2video - Add Captions output to this node.

5. **Create HTTP Request Node (json2video - Get Status)**  
   - Type: HTTP Request  
   - Purpose: Poll Json2Video API for processing status.  
   - Settings:  
     - HTTP Method: GET  
     - URL: `https://api.json2video.com/v2/movies?id={{ $('json2video - Add Captions').first().json.project }}`  
     - Authentication: Use same Custom HTTP Header Auth credentials as Add Captions node.  
   - Connect Wait node output to this node.

6. **Create If Node (Is Error)**  
   - Type: If  
   - Purpose: Check if API returned an error status.  
   - Condition: `{{$json.movie.status}} == "error"`  
   - Connect json2video - Get Status output to this node.

7. **Create NoOp Node (Handle Error)**  
   - Type: No Operation  
   - Purpose: Placeholder for error handling (can be extended).  
   - Connect Is Error node "true" output to this node.

8. **Create If Node (is Completed)**  
   - Type: If  
   - Purpose: Check if caption generation is complete.  
   - Condition: `{{$json.movie.status}} == "done"`  
   - Connect Is Error node "false" output to this node.

9. **Create Set Node (Output)**  
   - Type: Set  
   - Purpose: Prepare final output data.  
   - Assign the following fields from API response:  
     - `url` = `{{$json.movie.url}}`  
     - `duration` = `{{$json.movie.duration}}`  
     - `size` = `{{$json.movie.size}}`  
     - `width` = `{{$json.movie.width}}`  
     - `height` = `{{$json.movie.height}}`  
     - `rendering_time` = `{{$json.movie.rendering_time}}`  
     - `project` = `{{$json.movie.project}}`  
     - `remaining_quota` = `{{$json.remaining_quota.time}}`  
   - Connect is Completed node "true" output to this node.

10. **Connect is Completed node "false" output back to Wait node**  
    - This creates a loop that polls every 10 seconds until the job is done.

11. **Add Sticky Notes (Optional for Documentation)**  
    - Add a sticky note near the status polling nodes with content "## Check video status".  
    - Add a sticky note near Config node with instructions to provide video URL, width, and height.  
    - Add a sticky note with workflow overview and setup instructions referencing https://json2video.com/?afco=manu.

12. **Create Credentials in n8n**  
    - Create a new HTTP Custom Auth credential named "json2video".  
    - Configure with header:  
      ```json
      {
        "headers": {
          "x-api-key": "your-json2video-api-key"
        }
      }
      ```  
    - Assign this credential to both HTTP Request nodes interacting with Json2Video.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Automatically add captions to videos using Json2Video API for improved accessibility, engagement, and SEO.                       | Workflow purpose                                                                                 |
| Json2Video API requires a valid API key; sign up at [json2video.com](https://json2video.com/?afco=manu) to obtain one.             | API signup and credentials                                                                       |
| The workflow polls the API every 10 seconds to check processing status, balancing responsiveness and API rate limits.             | Polling strategy                                                                                |
| Caption style is customizable in the JSON payload; current style is "classic-progressive" with font "Oswald" and specific colors.| Caption styling                                                                                  |
| For best results, ensure the input video URL is publicly accessible and the dimensions are accurate.                              | Input preconditions                                                                             |
| This workflow can be extended to include error notifications or retries in the Handle Error node.                                 | Potential workflow enhancements                                                                 |
| [Try Json2Video for Free](https://json2video.com/?afco=manu)                                                                       | Promotional link included in sticky notes                                                       |

---

This documentation provides a complete, detailed reference for understanding, reproducing, and modifying the "Fully automated Video Captions generation with json2video" workflow in n8n.