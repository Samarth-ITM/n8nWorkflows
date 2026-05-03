Narrative Chaining: AI-Generated Video Scene Extensions with Veo3

https://n8nworkflows.xyz/workflows/narrative-chaining--ai-generated-video-scene-extensions-with-veo3-9145


# Narrative Chaining: AI-Generated Video Scene Extensions with Veo3

---

### 1. Workflow Overview

This workflow, titled **"Narrative Chaining: AI-Generated Video Scene Extensions with Veo3"**, automates the process of extending a given video by generating additional scenes using AI-based video generation and video analysis tools. Its primary use case is to create a seamless narrative continuation of a starting video clip by analyzing its content, generating prompts for new scenes, synthesizing video clips for these scenes, and merging all clips into a final extended video output. It is tailored for creative video content creators and AI developers who want to automate narrative video expansions with contextual consistency.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**  
  Fetches user inputs and initializes variables controlling the scene generation loop.

- **1.2 Loop Control and Iteration Setup**  
  Manages the looping logic to generate multiple scenes sequentially.

- **1.3 Video Analysis and Frame Extraction**  
  Analyzes the current video clip to extract descriptive metadata and captures the last frame image.

- **1.4 AI Prompt Generation for Next Scene**  
  Uses an AI agent to generate a detailed video prompt for the next scene based on analysis and user parameters.

- **1.5 Scene Generation and Video Production**  
  Calls video generation APIs to create the new scene video clip and waits for completion.

- **1.6 Scene Storage and Step Increment**  
  Logs generated scene data into Google Sheets, increments progression counters, and determines continuation.

- **1.7 Final Video Assembly and Output Update**  
  Aggregates all generated scenes, merges them with the original clip, and updates the input sheet with the final video URL and status.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
  This block fetches control parameters from a Google Sheet, clears the scene storage sheet, and initializes loop variables including the starting clip URL, the total number of scenes to generate, and the current step.

- **Nodes Involved:**  
  - Execute (Manual Trigger)  
  - Get input (Google Sheets)  
  - Clear scenes (Google Sheets)  
  - Initial Values (Set)  
  - Looper (Set)

- **Node Details:**

  - **Execute**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually.  
    - Connections: Outputs to "Get input".  
    - Edge Cases: None; user-triggered.

  - **Get input**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves the next row with status "for production" that contains user parameters like video URL, number of scenes, aspect ratio, model, narrative theme, and special requests.  
    - Configuration: Filters rows where `status = "for production"`.  
    - Credentials: Google Sheets OAuth2.  
    - Output: JSON with input fields.  
    - Edge Cases: Empty or missing rows, authentication failure.  

  - **Clear scenes**  
    - Type: Google Sheets (Clear Range)  
    - Role: Clears the range B2:C20 on the scene storage sheet to reset previous generated scene data.  
    - Configuration: Specifies clear range on Sheet2 for scenes.  
    - Credentials: Same as above.  
    - Edge Cases: Sheet access errors.

  - **Initial Values**  
    - Type: Set  
    - Role: Initializes key workflow variables:  
      - step = 1 (starting scene)  
      - complete = total number of clips to add (from input)  
      - videoURL = starting clip URL  
    - Connections: Outputs to "Looper".  
    - Edge Cases: Missing or invalid input data.

  - **Looper**  
    - Type: Set  
    - Role: Holds loop state variables propagated through iterations:  
      - step (current scene number)  
      - complete (total scenes requested)  
      - mediaURL (video URL for current iteration)  
    - Connections: Feeds into "Analyze Video".  
    - Edge Cases: Incorrect variable propagation.

---

#### 1.2 Loop Control and Iteration Setup

- **Overview:**  
  Controls the iterative process of generating scenes until all requested clips are created. Uses conditional checks to proceed or end the loop.

- **Nodes Involved:**  
  - If complete (If)  
  - Looper (Set) (repeated)  
  - Get scenes (Google Sheets)  
  - Aggregate (Aggregate)  
  - Combine Clips (HTTP Request)  
  - Wait 3 (Wait)  
  - Get Final Video (HTTP Request)  
  - Update log (Google Sheets)

- **Node Details:**

  - **If complete**  
    - Type: If  
    - Role: Checks if `step` exceeds `complete`. If yes, proceeds to final assembly; else loops back for next scene generation.  
    - Condition: `step < complete` (strictly less)  
    - Connections: True → Get scenes; False → Looper (next iteration).  
    - Edge Cases: Off-by-one errors, type mismatches.

  - **Get scenes**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves all stored scene URLs from the scene storage sheet column C.  
    - Configuration: Reads column C (sceneURL).  
    - Credentials: Google Sheets OAuth2.  
    - Output: List of generated scene URLs.  
    - Edge Cases: Empty sheet, access errors.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates scene URLs into a single array for merging.  
    - Configuration: Aggregates values from field "sceneURL".  
    - Connections: Outputs to "Combine Clips".  
    - Edge Cases: Empty inputs.

  - **Combine Clips**  
    - Type: HTTP Request  
    - Role: Calls the File.AI ffmpeg API to merge the original clip with all generated scenes into one video.  
    - Request: POST with JSON body containing array of video URLs (original + scenes).  
    - Authentication: HTTP Header Auth (fal.ai).  
    - Output: Response with a URL to the merged video.  
    - Edge Cases: API failures, invalid URLs, network timeout.

  - **Wait 3**  
    - Type: Wait  
    - Role: Pauses workflow 60 seconds to allow video merging processing.  
    - Edge Cases: Workflow pause duration may cause delays.

  - **Get Final Video**  
    - Type: HTTP Request  
    - Role: Polls the response URL from "Combine Clips" to get the merged video file URL.  
    - Authentication: HTTP Header Auth (fal.ai).  
    - Output: Final video metadata including downloadable URL.  
    - Edge Cases: Polling failures, incomplete processing.

  - **Update log**  
    - Type: Google Sheets (Update)  
    - Role: Updates the original input row to mark status as "done" and stores the final output video URL.  
    - Configuration: Matches row by `id` from input sheet.  
    - Credentials: Google Sheets OAuth2.  
    - Edge Cases: Write conflicts, access errors.

---

#### 1.3 Video Analysis and Frame Extraction

- **Overview:**  
  Analyzes the current video clip to understand its content and extracts the last frame image for use in the next scene generation, ensuring visual continuity.

- **Nodes Involved:**  
  - Analyze Video (HTTP Request)  
  - Get Analysis (HTTP Request)  
  - Wait (Wait)  
  - Request last (HTTP Request)  
  - Get last (HTTP Request)  
  - Wait 1 (Wait)

- **Node Details:**

  - **Analyze Video**  
    - Type: HTTP Request  
    - Role: Sends current video URL to the Fal.ai video understanding API to get detailed description including last frame details, music, camera, dialogue, accents, and characters.  
    - Request: POST JSON with "video_url" and a detailed prompt.  
    - Authentication: HTTP Header Auth (fal.ai).  
    - Output: Response URL to fetch analysis result.  
    - Edge Cases: API rate limits, malformed input.

  - **Get Analysis**  
    - Type: HTTP Request  
    - Role: Polls the response URL from "Analyze Video" to retrieve completed analysis text.  
    - Authentication: HTTP Header Auth (fal.ai).  
    - Output: JSON with detailed video analysis.  
    - Edge Cases: Polling timeouts, incomplete analysis.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses 60 seconds to allow analysis processing time.

  - **Request last**  
    - Type: HTTP Request  
    - Role: Calls Fal.ai API to extract the last frame (image) from the current video clip.  
    - Request: POST with video URL and frame_type set to "last".  
    - Authentication: HTTP Header Auth (fal.ai).  
    - Output: Response URL for the extracted frame image.  
    - Edge Cases: API failures, missing video.

  - **Get last**  
    - Type: HTTP Request  
    - Role: Polls the response URL from "Request last" to get the actual last frame image URL.  
    - Authentication: HTTP Header Auth (fal.ai).  
    - Output: JSON with URL to image.  
    - Edge Cases: Polling failure.

  - **Wait 1**  
    - Type: Wait  
    - Role: Pauses 30 seconds waiting for frame extraction to complete.

---

#### 1.4 AI Prompt Generation for Next Scene

- **Overview:**  
  Uses a LangChain AI agent (GPT-4 via OpenAI) and a Think tool to generate a video prompt for the next scene. This prompt is carefully crafted to maintain narrative and visual consistency with the previous scene, respecting user preferences.

- **Nodes Involved:**  
  - ExtendRobo AI Agent (LangChain Agent)  
  - GPT (LangChain Chat with OpenAI)  
  - Think (LangChain Think Tool)  
  - Structured Output (LangChain Structured Output Parser)

- **Node Details:**

  - **ExtendRobo AI Agent**  
    - Type: LangChain Agent  
    - Role: Generates video generation instructions for the next scene based on previous video analysis, user preferences (narrative theme, aspect ratio, model, special requests), and system prompt guidelines.  
    - Configuration:  
      - System prompt emphasizes scene continuity, video-only instructions, consistency in camera, music, dialogue, and format output as JSON with stringified YAML video prompts.  
      - Uses the Think tool to verify the output.  
      - Input includes the detailed video analysis text from "Get Analysis".  
    - Output: JSON with fields: `video_prompt` (YAML string), `aspect_ratio_video`, and `model`.  
    - Edge Cases: AI output errors, timeout, content policy filtering.

  - **GPT**  
    - Type: LangChain Chat OpenAI (GPT-4.1)  
    - Role: Provides language model capabilities to the AI agent.  
    - Credentials: OpenAI API key.  
    - Edge Cases: API rate limits, model unavailability.

  - **Think**  
    - Type: LangChain Tool (Think)  
    - Role: Evaluates AI agent output for consistency and correctness before finalizing prompt.  
    - Edge Cases: Evaluation errors.

  - **Structured Output**  
    - Type: LangChain Output Parser  
    - Role: Parses AI agent’s text output into structured JSON ensuring prompt format correctness.  
    - Configuration: Auto-fix enabled, expects JSON with video prompt details.  
    - Edge Cases: Parsing failures if AI output is malformed.

---

#### 1.5 Scene Generation and Video Production

- **Overview:**  
  Sends the AI-generated prompt, aspect ratio, model, and last frame image URL to the Kie.ai Veo3 video generation API to create the next video scene clip. Waits for processing to complete.

- **Nodes Involved:**  
  - Create Video (HTTP Request)  
  - Wait 2 (Wait)  
  - Get Video (HTTP Request)  
  - If (If)  
  - Sticky Note15 and Sticky Note16 (documentation aids)

- **Node Details:**

  - **Create Video**  
    - Type: HTTP Request  
    - Role: Submits a video generation request to Kie.ai Veo3 API with:  
      - Prompt JSON from AI Agent  
      - Model (veo3_fast or veo3)  
      - Aspect ratio (9:16 or 16:9)  
      - Image URL of last frame for seamless chaining  
    - Authentication: HTTP Header Auth (Kie.ai).  
    - Output: Task ID for video generation.  
    - Edge Cases: API failures, invalid parameters.

  - **Wait 2**  
    - Type: Wait  
    - Role: Waits 60 seconds for video generation to proceed.

  - **Get Video**  
    - Type: HTTP Request  
    - Role: Polls the generation task status and retrieves generated video URL once ready.  
    - Authentication: HTTP Header Auth (Kie.ai).  
    - Output: JSON with video URL and task details.  
    - Edge Cases: Task failure, timeout.

  - **If**  
    - Type: If  
    - Role: Checks if video generation succeeded by verifying `successFlag == 1`.  
    - True: Proceeds to store scene.  
    - False: Waits and retries.  
    - Edge Cases: Handling failed generation gracefully.

---

#### 1.6 Scene Storage and Step Increment

- **Overview:**  
  Logs the newly generated scene’s prompt and video URL into the Google Sheets scene storage. Increments the step counter and updates the current video URL to the newly generated clip for next iteration.

- **Nodes Involved:**  
  - Add scene (Google Sheets Update)  
  - Increment step (Set)  
  - If complete (If) (loops back or ends)

- **Node Details:**

  - **Add scene**  
    - Type: Google Sheets (Update)  
    - Role: Updates the scene storage sheet (Sheet2) with the scene number, prompt, and scene video URL. Matches rows by scene number.  
    - Credentials: Google Sheets OAuth2.  
    - Edge Cases: Write conflicts, missing rows.

  - **Increment step**  
    - Type: Set  
    - Role: Increments `step` by 1, carries forward `complete` and sets `videoURL` to the newly generated clip URL for next iteration.  
    - Edge Cases: Incorrect increment or variable overwrite.

  - **If complete**  
    - Reused from above. Decides whether to loop or finalize.

---

#### 1.7 Final Video Assembly and Output Update

- **Overview:**  
  Once all scenes are generated, collects all scene clips, merges them with the initial clip, and updates the original input row with the final video URL and marks status as done.

- **Nodes Involved:**  
  - Get scenes (Google Sheets)  
  - Aggregate (Aggregate)  
  - Combine Clips (HTTP Request)  
  - Wait 3 (Wait)  
  - Get Final Video (HTTP Request)  
  - Update log (Google Sheets)

- **Node Details:**

  - **Get scenes**  
    - See above.

  - **Aggregate**  
    - See above.

  - **Combine Clips**  
    - See above.

  - **Wait 3**  
    - See above.

  - **Get Final Video**  
    - See above.

  - **Update log**  
    - See above.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                                      | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                      |
|---------------------|--------------------------------|-----------------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------|
| Execute             | Manual Trigger                 | Starts the workflow manually                         | -                                | Get input                       |                                                                                                                |
| Get input           | Google Sheets                  | Fetches user parameters for video extension         | Execute                         | Clear scenes                   |                                                                                                                |
| Clear scenes        | Google Sheets                  | Clears scene storage sheet to reset previous data   | Get input                      | Initial Values                 |                                                                                                                |
| Initial Values      | Set                           | Initializes loop variables (step, complete, videoURL) | Clear scenes                   | Looper                        |                                                                                                                |
| Looper              | Set                           | Holds loop state variables for iteration             | Initial Values / Increment step | Analyze Video                 |                                                                                                                |
| Analyze Video       | HTTP Request                  | Calls video analysis API to get detailed description | Looper                         | Wait                         | ## Step 1: Analyze preceding vid                                                                               |
| Wait                | Wait                          | Waits 60 seconds for analysis processing              | Analyze Video                  | Get Analysis                 |                                                                                                                |
| Get Analysis        | HTTP Request                  | Polls video analysis results                          | Wait                          | ExtendRobo AI Agent          |                                                                                                                |
| ExtendRobo AI Agent | LangChain Agent               | Generates next scene video prompt based on analysis  | Get Analysis / GPT / Think     | Request last                 | ## Step 2: Make next prompt                                                                                    |
| GPT                 | LangChain Chat OpenAI          | Provides GPT-4 model support for AI agent            | ExtendRobo AI Agent            | Structured Output            |                                                                                                                |
| Think               | LangChain Tool (Think)        | Verifies AI agent output consistency                  | GPT                           | ExtendRobo AI Agent          |                                                                                                                |
| Structured Output   | LangChain Output Parser       | Parses AI output into structured JSON                 | GPT                           | ExtendRobo AI Agent          |                                                                                                                |
| Request last        | HTTP Request                  | Requests last video frame image extraction            | ExtendRobo AI Agent            | Wait 1                      | ## Step 3: Get last frame                                                                                      |
| Wait 1              | Wait                          | Waits 30 seconds for frame extraction                 | Request last                  | Get last                    |                                                                                                                |
| Get last            | HTTP Request                  | Polls last frame image URL                            | Wait 1                        | Create Video                |                                                                                                                |
| Create Video        | HTTP Request                  | Submits video generation request with prompt and image | Get last                     | Wait 2                      | ## Step 4: Generate next scene                                                                                 |
| Wait 2              | Wait                          | Waits 60 seconds for video generation                  | Create Video                  | Get Video                   |                                                                                                                |
| Get Video           | HTTP Request                  | Polls video generation status and retrieves video URL | Wait 2                       | If                          |                                                                                                                |
| If                  | If                            | Checks if video generation succeeded                   | Get Video                    | Add scene / Wait 2           |                                                                                                                |
| Add scene           | Google Sheets (Update)        | Logs generated scene data to sheet                      | If (true)                    | Increment step              | ## Step 5: Add scene & increment step                                                                          |
| Increment step      | Set                           | Increments step counter and updates videoURL           | Add scene                    | If complete                 |                                                                                                                |
| If complete         | If                            | Decides to loop or finalize workflow                   | Increment step               | Get scenes / Looper          |                                                                                                                |
| Get scenes          | Google Sheets (Read)          | Retrieves all generated scene URLs                      | If complete (true)            | Aggregate                   |                                                                                                                |
| Aggregate           | Aggregate                     | Aggregates scene URLs into array for merging            | Get scenes                   | Combine Clips               |                                                                                                                |
| Combine Clips       | HTTP Request                  | Merges original video and generated scenes             | Aggregate                    | Wait 3                      |                                                                                                                |
| Wait 3              | Wait                          | Waits 60 seconds for merging process                    | Combine Clips                | Get Final Video             |                                                                                                                |
| Get Final Video     | HTTP Request                  | Retrieves merged final video URL                         | Wait 3                      | Update log                  |                                                                                                                |
| Update log          | Google Sheets (Update)        | Updates input sheet row with final video URL and status | Get Final Video              | -                           |                                                                                                                |
| Sticky Note11       | Sticky Note                   | Label for Step 3: Get last frame                        | -                            | -                           | ## Step 3: Get last frame                                                                                      |
| Sticky Note12       | Sticky Note                   | Label for Step 1: Analyze preceding vid                 | -                            | -                           | ## Step 1: Analyze preceding vid                                                                               |
| Sticky Note14       | Sticky Note                   | Label for Step 2: Make next prompt                       | -                            | -                           | ## Step 2: Make next prompt                                                                                    |
| Sticky Note15       | Sticky Note                   | Label for Step 4: Generate next scene                    | -                            | -                           | ## Step 4: Generate next scene                                                                                 |
| Sticky Note16       | Sticky Note                   | Label for Step 5: Add scene & increment step             | -                            | -                           | ## Step 5: Add scene & increment step                                                                          |
| Sticky Note1        | Sticky Note                   | Structured Setup Guide and workflow explanation          | -                            | -                           | Structured Setup Guide: Narrative Chaining with N8N + AI (detailed multi-step instructions and costs)          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: Execute  
   - Purpose: Start workflow manually.

2. **Add Google Sheets Node to Read Input**  
   - Name: Get input  
   - Operation: Read rows from main control sheet.  
   - Filter: `status` equals `"for production"`.  
   - Credentials: Set up Google Sheets OAuth2.  
   - Output: User parameters including starting video URL, clips to add, aspect ratio, model, narrative theme, special requests.

3. **Add Google Sheets Node to Clear Scenes Sheet Range**  
   - Name: Clear scenes  
   - Operation: Clear specific range B2:C20 on the scenes sheet.  
   - Credentials: Same as above.

4. **Add Set Node to Initialize Variables**  
   - Name: Initial Values  
   - Variables:  
     - `step` = 1  
     - `complete` = number of clips to add (from Get input)  
     - `videoURL` = starting clip URL (from Get input)

5. **Add Set Node for Loop State**  
   - Name: Looper  
   - Variables carried forward each iteration:  
     - `step`  
     - `complete`  
     - `mediaURL` (holds current video URL)

6. **Add HTTP Request Node for Video Analysis**  
   - Name: Analyze Video  
   - URL: `https://queue.fal.run/fal-ai/video-understanding`  
   - Method: POST  
   - Body (JSON):  
     - `video_url`: from `mediaURL` (Looper)  
     - `prompt`: detailed text asking for last frame description, music, camera, dialogue, accents, characters  
     - `detailed_analysis`: true  
   - Authentication: HTTP Header Auth (fal.ai API key)  
   - Output: Response URL for polling.

7. **Add Wait Node**  
   - Name: Wait  
   - Duration: 60 seconds (for analysis processing).

8. **Add HTTP Request Node to Poll Analysis Result**  
   - Name: Get Analysis  
   - URL: Response URL from Analyze Video  
   - Authentication: Same as above.

9. **Add LangChain Agent Node**  
   - Name: ExtendRobo AI Agent  
   - System Prompt: Scene continuation AI agent instructions including narrative chaining rules, format requirements, and user preferences.  
   - Input: Uses detailed analysis from Get Analysis and user inputs from Get input.  
   - Tools: Enabled Think tool for output verification.  
   - Output: JSON with `video_prompt` (stringified YAML), `aspect_ratio_video`, `model`.

10. **Add LangChain Chat OpenAI Node**  
    - Name: GPT  
    - Model: GPT-4.1 or latest available.  
    - Credentials: OpenAI API key.

11. **Connect GPT as AI Language Model to ExtendRobo AI Agent**  
    - Link GPT node for language model backend.

12. **Add LangChain Think Tool Node**  
    - Name: Think  
    - Used by AI Agent for output checking.

13. **Add LangChain Structured Output Parser Node**  
    - Name: Structured Output  
    - AutoFix: enabled  
    - JSON Schema Example: Expect JSON with `video_prompt`, `aspect_ratio_video`, `model`.

14. **Add HTTP Request Node to Request Last Frame Extraction**  
    - Name: Request last  
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/extract-frame`  
    - Method: POST  
    - Body: JSON with `video_url` from Looper `mediaURL` and `frame_type` set to `"last"`  
    - Authentication: fal.ai HTTP Header Auth.

15. **Add Wait Node**  
    - Name: Wait 1  
    - Duration: 30 seconds (wait for frame extraction).

16. **Add HTTP Request Node to Poll Last Frame Result**  
    - Name: Get last  
    - URL: Response URL from Request last  
    - Authentication: fal.ai.

17. **Add HTTP Request Node to Create Video Scene**  
    - Name: Create Video  
    - URL: `https://api.kie.ai/api/v1/veo/generate`  
    - Method: POST  
    - Body: JSON containing:  
      - `prompt` = `video_prompt` from AI Agent output (escaped)  
      - `model` = AI Agent output model field  
      - `aspectRatio` = AI Agent output aspect ratio  
      - `imageUrls` = URL of last frame image from Get last  
    - Authentication: Kie.ai HTTP Header Auth.

18. **Add Wait Node**  
    - Name: Wait 2  
    - Duration: 60 seconds (video generation wait).

19. **Add HTTP Request Node to Poll Video Generation Status**  
    - Name: Get Video  
    - URL: Kie.ai API endpoint for task status with `taskId` from Create Video response.  
    - Authentication: Kie.ai.

20. **Add If Node to Check SuccessFlag**  
    - Name: If  
    - Condition: `data.successFlag == 1` (video generated successfully).  
    - True: Proceed to Add scene.  
    - False: Re-wait or retry.

21. **Add Google Sheets Node to Update Scene Storage**  
    - Name: Add scene  
    - Operation: Update sheet 2 row matching `scene` number with:  
      - `prompt` = AI Agent video prompt  
      - `sceneURL` = generated video URL from Get Video  
    - Credentials: Google Sheets OAuth2.

22. **Add Set Node to Increment Step and Update Video URL**  
    - Name: Increment step  
    - Increment `step` by 1  
    - Copy `complete` unchanged  
    - Update `videoURL` with newly generated clip URL.

23. **Add If Node to Check if Complete**  
    - Name: If complete  
    - Condition: `step < complete`  
    - True: Loop back to Looper for next iteration.  
    - False: Proceed to final assembly.

24. **Add Google Sheets Node to Get All Scenes**  
    - Name: Get scenes  
    - Reads column C (scene URLs) from scenes sheet.

25. **Add Aggregate Node**  
    - Name: Aggregate  
    - Aggregates all scene URLs into array.

26. **Add HTTP Request Node to Combine Clips**  
    - Name: Combine Clips  
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`  
    - Method: POST  
    - Body: JSON array of original video URL + all scene URLs.  
    - Authentication: fal.ai.

27. **Add Wait Node**  
    - Name: Wait 3  
    - Duration: 60 seconds (wait for merging).

28. **Add HTTP Request Node to Get Final Video URL**  
    - Name: Get Final Video  
    - Polls merge response URL for final video URL.  
    - Authentication: fal.ai.

29. **Add Google Sheets Node to Update Original Input Row**  
    - Name: Update log  
    - Updates row matching input `id` with:  
      - `status` = "done"  
      - `outputURL` = final merged video URL.

30. **Add Sticky Notes at appropriate points**  
    - For clarity and documentation within the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Structured Setup Guide embedded as Sticky Note1 provides detailed instructions for input sheet setup, workflow logic, and cost estimates. | Workflow Sticky Note1 |
| Fal.ai APIs are used for video understanding, frame extraction, and video merging. Authentication via HTTP Header is required. | https://queue.fal.run/ |
| Kie.ai Veo3 API is used for AI-driven video generation with model options `veo3` and `veo3_fast`. | https://api.kie.ai/ |
| AI agent system prompt and generation rules emphasize narrative consistency, format constraints, and content policy adherence. | ExtendRobo AI Agent node configuration |
| User input Google Sheet must contain fields: `videoURL`, `clips to add`, `aspect ratio`, `model`, `narrative theme`, `any special requests`, and `status`. | Get input node parameters |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly follows current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---