Viral ASMR Video Factory: Automatically generate viral videos on autopilot.

https://n8nworkflows.xyz/workflows/viral-asmr-video-factory--automatically-generate-viral-videos-on-autopilot--5324


# Viral ASMR Video Factory: Automatically generate viral videos on autopilot.

### 1. Workflow Overview

This workflow, titled **"Viral ASMR Video Factory: Automatically generate viral videos on autopilot,"** automates the process of generating, uploading, and managing viral ASMR video content across multiple social media platforms. It leverages AI agents for ideation and prompt generation, integrates with Google Sheets for data persistence, and uses HTTP requests to interact with external video generation and social media upload services.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger & Past Data Retrieval:** Triggering the workflow manually and fetching past video data from Google Sheets.
- **1.2 Data Aggregation & Preparation:** Aggregating past data and formatting it for AI processing.
- **1.3 AI Processing (Idea and Prompt Generation):** Using Langchain AI agents and OpenAI models to generate video ideas and prompts.
- **1.4 Video Generation & Result Polling:** Requesting video generation via HTTP API and polling results with wait intervals.
- **1.5 Upload to Social Platforms:** Uploading generated videos to YouTube, Instagram, and TikTok via HTTP API calls.
- **1.6 Data Maintenance in Google Sheets:** Managing Google Sheets rows by deleting old entries and appending new video metadata.
- **1.7 Wait and Retry Logic:** Incorporating wait nodes to handle asynchronous external API processing times.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger & Past Data Retrieval

**Overview:**  
This block initiates the workflow upon manual trigger and retrieves previously generated video data from Google Sheets to provide context for AI agents.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get Past Objects (Google Sheets)  
- Aggregate (Aggregate)  
- Set Object List (Set)  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Configuration: Default manual trigger settings  
  - Inputs: None  
  - Outputs: To "Get Past Objects"  
  - Edge Cases: User must trigger manually; no automated scheduling  
  - Version: v1  

- **Get Past Objects**  
  - Type: Google Sheets  
  - Role: Fetches existing video data rows from a specified Google Sheet  
  - Configuration: Connects to Google Sheets credential; retrieves relevant data range (likely all rows to provide history)  
  - Inputs: From manual trigger  
  - Outputs: To "Aggregate" node  
  - Edge Cases: API quota limits, authorization errors, empty sheet data  
  - Version: v4.6  

- **Aggregate**  
  - Type: Aggregate  
  - Role: Processes and aggregates the retrieved sheet data into a consolidated format for AI processing  
  - Configuration: Groups or concatenates rows into a list or summary object  
  - Inputs: From "Get Past Objects"  
  - Outputs: To "Set Object List"  
  - Edge Cases: Empty input data, aggregation failures  
  - Version: v1  

- **Set Object List**  
  - Type: Set  
  - Role: Formats aggregated data into a structured list for downstream AI agents  
  - Configuration: Uses expressions to set fields, e.g., list of past video objects  
  - Inputs: From "Aggregate"  
  - Outputs: To "Idea Agent"  
  - Edge Cases: Expression or data formatting errors  
  - Version: v3.4  

---

#### 2.2 AI Processing (Idea and Prompt Generation)

**Overview:**  
This block uses AI models to generate new video ideas based on past video data and then creates prompts for video generation.

**Nodes Involved:**  
- Idea Agent (Langchain Agent)  
- Prompt Agent (Langchain Agent)  
- OpenAI Chat Model (Langchain LM Chat OpenAI)  
- Object & Caption (Langchain Output Parser Structured)  

**Node Details:**

- **Idea Agent**  
  - Type: Langchain Agent  
  - Role: Generates new video concepts/ideas using input data from "Set Object List"  
  - Configuration: Connected to "OpenAI Chat Model" as language backend  
  - Inputs: From "Set Object List" and "Object & Caption" (output parser)  
  - Outputs: To "Prompt Agent"  
  - Edge Cases: AI model latency, token limits, API errors  
  - Version: v2  

- **Prompt Agent**  
  - Type: Langchain Agent  
  - Role: Converts video ideas into detailed prompts suitable for video generation API  
  - Configuration: Uses "OpenAI Chat Model" for language model support  
  - Inputs: From "Idea Agent"  
  - Outputs: To "Generate Video" (HTTP request)  
  - Edge Cases: API call failures, malformed prompts  
  - Version: v2  

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Provides language model capabilities to AI agents  
  - Configuration: Uses OpenAI credentials (likely GPT-4 or GPT-3.5)  
  - Inputs: Feeds both "Prompt Agent" and "Idea Agent"  
  - Outputs: N/A (used internally by agents)  
  - Edge Cases: API rate limits, auth errors  
  - Version: v1.2  

- **Object & Caption**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses AI output into structured JSON with defined fields (e.g., video object and caption)  
  - Configuration: Custom parsing rules for expected AI output format  
  - Inputs: From AI agents (likely after "Idea Agent")  
  - Outputs: To "Idea Agent" for iterative processing  
  - Edge Cases: Parsing errors if AI output format changes or is malformed  
  - Version: v1.3  

---

#### 2.3 Video Generation & Result Polling

**Overview:**  
This block sends a video generation request via HTTP, then waits and polls until the processing is complete.

**Nodes Involved:**  
- Generate Video (HTTP Request)  
- 5 Minutes (Wait)  
- Get Result (HTTP Request)  
- Result (Set)  
- 30 Seconds (Wait)  

**Node Details:**

- **Generate Video**  
  - Type: HTTP Request  
  - Role: Sends video generation request to external API/service using data from AI prompt  
  - Configuration: POST request with payload containing prompt and parameters  
  - Inputs: From "Prompt Agent"  
  - Outputs: To "5 Minutes" wait node  
  - Edge Cases: API timeout, request failure, malformed request  
  - Version: v4.2  

- **5 Minutes**  
  - Type: Wait  
  - Role: Waits 5 minutes before polling results to allow video processing  
  - Configuration: Fixed delay of 5 minutes  
  - Inputs: From "Generate Video"  
  - Outputs: To "Get Result"  
  - Edge Cases: Long wait may cause workflow delays; no timeout or cancellation configured  
  - Version: v1.1  

- **Get Result**  
  - Type: HTTP Request  
  - Role: Polls the video generation API for processing status and retrieves the result  
  - Configuration: GET or POST request with video ID or token  
  - Inputs: From "5 Minutes" and "30 Seconds" (retry loop)  
  - Outputs: To "Result" and "30 Seconds" wait node for retry  
  - OnError: Continue on error to avoid failure on transient issues  
  - Edge Cases: API errors, incomplete results, retries needed  
  - Version: v4.2  

- **Result**  
  - Type: Set  
  - Role: Stores or formats the final video generation result for next steps  
  - Configuration: Sets fields like video URL, metadata, status  
  - Inputs: From "Get Result"  
  - Outputs: To "Delete First Row" (Google Sheets)  
  - Edge Cases: Missing or malformed result data  
  - Version: v3.4  

- **30 Seconds**  
  - Type: Wait  
  - Role: Waits 30 seconds before retrying "Get Result" if the video is not ready  
  - Configuration: Fixed delay of 30 seconds  
  - Inputs: From "Get Result" (for retry)  
  - Outputs: To "Get Result"  
  - Edge Cases: Potential for infinite loop if video never completes; no max retry count visible  
  - Version: v1.1  

---

#### 2.4 Upload to Social Platforms

**Overview:**  
This block uploads the finalized video content to YouTube, Instagram, and TikTok via HTTP requests.

**Nodes Involved:**  
- Upload (HTTP Request)  
- YouTube (HTTP Request)  
- Instagram (HTTP Request)  
- TikTok (HTTP Request)  

**Node Details:**

- **Upload**  
  - Type: HTTP Request  
  - Role: Uploads the generated video to a centralized service or directly manages file upload payloads  
  - Configuration: Likely POST with video file or URL  
  - Inputs: From upstream nodes (not explicitly shown, likely "Result")  
  - Outputs: To "YouTube", "Instagram", and "TikTok" nodes in parallel  
  - Edge Cases: Upload failures, large file timeouts  
  - Version: v4.2  

- **YouTube**  
  - Type: HTTP Request  
  - Role: Sends video upload request to YouTube API or a proxy service  
  - Configuration: Authenticated HTTP request; on error, continues without aborting workflow  
  - Inputs: From "Upload"  
  - Outputs: None  
  - Edge Cases: Auth errors, quota limits, video format restrictions  
  - Version: v4.2  

- **Instagram**  
  - Type: HTTP Request  
  - Role: Sends video upload request to Instagram API or proxy  
  - Configuration: Authenticated HTTP request with error continuation  
  - Inputs: From "Upload"  
  - Outputs: None  
  - Edge Cases: API changes, session expiration, format errors  
  - Version: v4.2  

- **TikTok**  
  - Type: HTTP Request  
  - Role: Sends video upload request to TikTok API or proxy  
  - Configuration: Authenticated HTTP request with error continuation  
  - Inputs: From "Upload"  
  - Outputs: None  
  - Edge Cases: API rate limits, token expiration  
  - Version: v4.2  

---

#### 2.5 Data Maintenance in Google Sheets

**Overview:**  
Maintains the Google Sheets data by deleting the oldest row and appending the new video metadata after successful upload.

**Nodes Involved:**  
- Delete First Row (Google Sheets)  
- Append Object (Google Sheets)  

**Node Details:**

- **Delete First Row**  
  - Type: Google Sheets  
  - Role: Deletes the top (oldest) row to keep sheet size manageable  
  - Configuration: Uses Google Sheets API to delete a specific row (likely row 2 to preserve headers)  
  - Inputs: From "Result" node  
  - Outputs: To "Append Object"  
  - Edge Cases: Permissions errors, concurrent edits causing conflicts  
  - Version: v4.6  

- **Append Object**  
  - Type: Google Sheets  
  - Role: Appends newly generated video metadata as a new row  
  - Configuration: Adds data like video title, URL, caption, date, etc.  
  - Inputs: From "Delete First Row"  
  - Outputs: None (end of data chain)  
  - Edge Cases: Append failures, data format issues  
  - Version: v4.6  

---

#### 2.6 Wait and Retry Logic

**Overview:**  
Wait nodes are used to handle asynchronous operations and API response delays gracefully.

**Nodes Involved:**  
- 5 Minutes (Wait)  
- 30 Seconds (Wait)  

**Node Details:**

- **5 Minutes**  
  - Type: Wait  
  - Role: Initial wait after sending video generation request  
  - Configuration: 5-minute fixed wait  
  - Inputs: From "Generate Video"  
  - Outputs: To "Get Result"  
  - Edge Cases: Workflow delay, no cancellation logic  
  - Version: v1.1  

- **30 Seconds**  
  - Type: Wait  
  - Role: Retry delay between "Get Result" polling requests  
  - Configuration: 30-second fixed wait  
  - Inputs: From "Get Result" on error or incomplete result  
  - Outputs: To "Get Result" (loop)  
  - Edge Cases: Possible infinite retry loop if video never completes  
  - Version: v1.1  

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                                     | Input Node(s)                  | Output Node(s)                                    | Sticky Note |
|-----------------------------|-----------------------------------|----------------------------------------------------|-------------------------------|--------------------------------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger                    | Workflow entry point                               | -                             | Get Past Objects                                  |             |
| Get Past Objects             | Google Sheets                     | Retrieves past video data                           | When clicking ‘Execute workflow’ | Aggregate                                        |             |
| Aggregate                   | Aggregate                        | Aggregates past data                               | Get Past Objects              | Set Object List                                  |             |
| Set Object List             | Set                              | Formats aggregated data for AI                      | Aggregate                     | Idea Agent                                       |             |
| Idea Agent                 | Langchain Agent                  | Generates video ideas                               | Set Object List, Object & Caption | Prompt Agent                                    |             |
| Object & Caption            | Langchain Output Parser Structured | Parses AI output into structured format             | Idea Agent                    | Idea Agent                                       |             |
| Prompt Agent               | Langchain Agent                  | Creates video generation prompts                    | Idea Agent                   | Generate Video                                   |             |
| OpenAI Chat Model           | Langchain LM Chat OpenAI         | Provides AI language model                           | -                             | Idea Agent, Prompt Agent                         |             |
| Generate Video              | HTTP Request                    | Sends video generation request                      | Prompt Agent                  | 5 Minutes                                       |             |
| 5 Minutes                  | Wait                            | Waits 5 minutes before polling                      | Generate Video                | Get Result                                      |             |
| Get Result                 | HTTP Request                    | Polls video generation status                        | 5 Minutes, 30 Seconds         | Result, 30 Seconds                               |             |
| Result                     | Set                             | Stores video generation result                       | Get Result                   | Delete First Row                                 |             |
| Delete First Row           | Google Sheets                   | Deletes oldest data row                              | Result                       | Append Object                                    |             |
| Append Object              | Google Sheets                   | Appends new video metadata                           | Delete First Row             | -                                               |             |
| Upload                     | HTTP Request                    | Uploads video for social platforms                   | Result                       | YouTube, Instagram, TikTok                       |             |
| YouTube                    | HTTP Request                    | Uploads video to YouTube                             | Upload                       | -                                               |             |
| Instagram                  | HTTP Request                    | Uploads video to Instagram                           | Upload                       | -                                               |             |
| TikTok                     | HTTP Request                    | Uploads video to TikTok                              | Upload                       | -                                               |             |
| 30 Seconds                 | Wait                            | Waits 30 seconds between polling retries            | Get Result                   | Get Result                                      |             |
| Sticky Note, Sticky Note1, Sticky Note2, Sticky Note4, Sticky Note7 | Sticky Note                     | Notes and comments (empty content)                  | -                             | -                                               |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No additional parameters  

2. **Add Google Sheets Node to Get Past Data**  
   - Name: `Get Past Objects`  
   - Type: Google Sheets  
   - Configure credentials for Google Sheets API  
   - Set operation to "Read Rows" specifying sheet and range containing past video data  

3. **Add Aggregate Node**  
   - Name: `Aggregate`  
   - Type: Aggregate  
   - Configure to group or concatenate data rows into a list for AI input  

4. **Add Set Node**  
   - Name: `Set Object List`  
   - Type: Set  
   - Use expressions to format aggregated data into structured input for AI agents  

5. **Add Langchain OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: Langchain LM Chat OpenAI  
   - Configure with OpenAI API credentials (GPT-3.5/4)  

6. **Add Langchain AI Agent Node for Idea Generation**  
   - Name: `Idea Agent`  
   - Type: Langchain Agent  
   - Configure to use `OpenAI Chat Model` as language model backend  
   - Input: `Set Object List`  
   - Output: Connect to `Object & Caption` and `Prompt Agent`  

7. **Add Langchain Output Parser Node**  
   - Name: `Object & Caption`  
   - Type: Langchain Output Parser Structured  
   - Configure structured parsing rules to extract video idea and caption from AI output  
   - Input: From `Idea Agent`  
   - Output: Back to `Idea Agent` (for iterative refinement)  

8. **Add Langchain AI Agent Node for Prompt Generation**  
   - Name: `Prompt Agent`  
   - Type: Langchain Agent  
   - Connect input from `Idea Agent`  
   - Connect language model backend to `OpenAI Chat Model`  
   - Output: Connect to `Generate Video` HTTP Request  

9. **Add HTTP Request Node to Generate Video**  
   - Name: `Generate Video`  
   - Type: HTTP Request  
   - Configure POST request to video generation API with prompt payload from `Prompt Agent`  
   - Output: Connect to `5 Minutes` Wait node  

10. **Add Wait Node for Initial Delay**  
    - Name: `5 Minutes`  
    - Type: Wait  
    - Set wait time to 5 minutes  
    - Output: Connect to `Get Result` HTTP Request  

11. **Add HTTP Request Node to Poll Result**  
    - Name: `Get Result`  
    - Type: HTTP Request  
    - Configure to poll video generation API for status/results  
    - On error: Set to continue on error without aborting  
    - Output: Connect to `Result` Set node and `30 Seconds` Wait node  

12. **Add Set Node to Store Result**  
    - Name: `Result`  
    - Type: Set  
    - Configure fields to capture video URL, metadata, status  
    - Output: Connect to `Delete First Row` Google Sheets node and `Upload` HTTP Request node  

13. **Add Wait Node for Retry Delay**  
    - Name: `30 Seconds`  
    - Type: Wait  
    - Set to 30 seconds  
    - Output: Connect back to `Get Result` for retry polling  

14. **Add Google Sheets Node to Delete Old Row**  
    - Name: `Delete First Row`  
    - Type: Google Sheets  
    - Configure to delete the first data row (e.g., row 2) to maintain sheet size  
    - Output: Connect to `Append Object`  

15. **Add Google Sheets Node to Append New Data**  
    - Name: `Append Object`  
    - Type: Google Sheets  
    - Configure to append the new video metadata row  
    - Output: None (end of chain)  

16. **Add HTTP Request Node to Upload Video**  
    - Name: `Upload`  
    - Type: HTTP Request  
    - Configure to upload video file or URL to a centralized upload API or service  
    - Output: Connect in parallel to `YouTube`, `Instagram`, and `TikTok` nodes  

17. **Add HTTP Request Nodes for Social Platform Uploads**  
    - Names: `YouTube`, `Instagram`, `TikTok`  
    - Type: HTTP Request  
    - Configure each with respective platform API endpoints and credentials  
    - Set error handling to continue on error (do not fail entire workflow)  
    - Outputs: None  

18. **Add Sticky Note Nodes (Optional)**  
    - Add any notes or comments for workflow documentation and explainers  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                  |
|------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow automates viral ASMR video creation across YouTube, Instagram, TikTok | Workflow purpose                                |
| Uses Langchain AI agents with OpenAI GPT models for idea and prompt generation | AI integration details                          |
| Google Sheets used for persistent history and data management                 | Data persistence                                |
| Wait nodes handle asynchronous API delays during video generation             | Asynchronous processing logic                   |
| HTTP Requests with error continuation prevent workflow failure on upload errors | Robust error handling                           |
| Manual trigger initiates workflow; no automated scheduling configured          | Workflow activation                             |

---

**Disclaimer:** The content provided is exclusively derived from an n8n automated workflow. It complies fully with content policies, contains no illegal or offensive elements, and handles only legal and public data.