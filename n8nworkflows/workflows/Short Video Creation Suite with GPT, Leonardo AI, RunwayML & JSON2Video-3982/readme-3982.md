Short Video Creation Suite with GPT, Leonardo AI, RunwayML & JSON2Video

https://n8nworkflows.xyz/workflows/short-video-creation-suite-with-gpt--leonardo-ai--runwayml---json2video-3982


# Short Video Creation Suite with GPT, Leonardo AI, RunwayML & JSON2Video

### 1. Workflow Overview

The **Short Video Creation Suite with GPT, Leonardo AI, RunwayML & JSON2Video** is a comprehensive, AI-driven automation designed to generate short videos seamlessly from concept to final output with minimal manual intervention. It targets creators, marketers, and businesses aiming to produce high-quality, engaging short videos efficiently for social media or promotional use.

The workflow logically divides into the following functional blocks:

- **1.1 Input Reception & Preprocessing**  
  Captures video requirements via a Baserow form webhook and initiates processing. Includes initial decision logic to determine if processing should proceed and which script generation path (AI-generated or manual) to use.

- **1.2 Script Generation (AI or Manual)**  
  Generates or accepts video scripts using OpenAI GPT-4 or manual input. Parses and structures the script into scenes.

- **1.3 Scene Handling & Visual Generation**  
  Maps scenes, then for each scene, generates visuals either through Leonardo AI (image generation) or RunwayML (acting scenes/video). Manages asynchronous waits while awaiting external service responses.

- **1.4 Video Assembly and Editing**  
  Combines generated visuals and captions into a final video using JSON2Video or HeyGen APIs. Handles polling for video rendering completion and error workflows.

- **1.5 Captions & Post-Processing**  
  Generates captions via CaptionsAI and manages their integration and error handling.

- **1.6 Data Management and Tracking**  
  Updates and tracks progress, errors, and outputs in Baserow databases to enable monitoring, retries, and iteration.

These blocks interconnect with conditional routing and batch processing to support single or bulk video generation modes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

- **Overview:**  
  This block receives the initial video request data from a Baserow webhook. It then filters whether to process the request and chooses the script generation method.

- **Nodes Involved:**  
  - Webhook  
  - Body (Set node)  
  - Should Process? (If node)  
  - Switch ScriptType (Switch node)  
  - Baserow Processing (Baserow node)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (entry point)  
    - Role: Receives JSON payload from Baserow form submissions containing video parameters.  
    - Key variables: Incoming JSON with fields like Topic, ScriptType, AvatarId, ScenesNumber, etc.  
    - Output: Passes data to the Body node.  
    - Failures: Missing or malformed webhook payloads, connectivity issues.

  - **Body**  
    - Type: Set node  
    - Role: Prepares and normalizes the webhook data for downstream nodes.  
    - Inputs: Webhook output.  
    - Outputs: Structured data for conditional checks.

  - **Should Process?**  
    - Type: If node  
    - Role: Determines if the workflow should proceed based on input parameters or status fields.  
    - Inputs: Body node data.  
    - Outputs: True branch routes to Switch ScriptType, False branch to Baserow Processing to handle non-processing cases or errors.

  - **Switch ScriptType**  
    - Type: Switch node  
    - Role: Routes the workflow depending on the script generation type (e.g., AI-generated script or manual script input).  
    - Inputs: Should Process? true output.  
    - Outputs: Routes to Basic LLM Chain (AI) or Basic LLM Chain Manual (manual script).

  - **Baserow Processing**  
    - Type: Baserow node  
    - Role: Updates or manages records when processing is skipped or for error logging.  
    - Inputs: Should Process? false output.  
    - Failures: API authentication or record update errors.

---

#### 2.2 Script Generation (AI or Manual)

- **Overview:**  
  Generates video scripts either by calling OpenAI GPT-4 or using manually provided scripts. The output is parsed into structured scenes for further processing.

- **Nodes Involved:**  
  - Basic LLM Chain (LangChain LLM Chain)  
  - Basic LLM Chain Manual (LangChain LLM Chain)  
  - OpenAI Chat Model  
  - OpenAI Chat Model1  
  - Structured Output Parser  
  - Execute Workflow4 (sub-workflow for script refinement or additional processing)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain Chain LLM  
    - Role: Generates script text using OpenAI Chat Model based on input parameters.  
    - Config: Uses OpenAI Chat Model as language model; on error, continues with error output.  
    - Input: Data from Switch ScriptType node when AI generation is selected.  
    - Output: Raw script text.

  - **Basic LLM Chain Manual**  
    - Type: LangChain Chain LLM  
    - Role: Processes manual script input with minimal AI intervention or formatting.  
    - Similar configuration and error handling as Basic LLM Chain.  
    - Input: Manual script data branch.

  - **OpenAI Chat Model & OpenAI Chat Model1**  
    - Type: Language Model (OpenAI GPT)  
    - Role: Provide GPT-4 or relevant OpenAI LLM API interface to generate or process text.  
    - Key parameters: API key credential, model selection, temperature, max tokens, etc.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser (Structured)  
    - Role: Parses LLM output into structured JSON separating scenes, captions, and other metadata.  
    - Inputs: Output from Basic LLM Chain / Basic LLM Chain Manual.  
    - Outputs: Structured scene data for next steps.

  - **Execute Workflow4**  
    - Type: Execute Workflow (Sub-workflow)  
    - Role: Further processing/refinement of the script or scene data (e.g., formatting, validation).  
    - Input/Output: Receives structured scenes, returns enriched data.

- **Failure Modes:**  
  - LLM API quota exceeded or auth errors.  
  - Parsing errors if output is malformed.  
  - Sub-workflow failures or misconfiguration.

---

#### 2.3 Scene Handling & Visual Generation

- **Overview:**  
  Processes each scene individually, generating visuals by calling Leonardo AI for images or RunwayML for video scenes, based on the background type (images or acting scenes).

- **Nodes Involved:**  
  - Scenes Mapping (Set node)  
  - Split Out (Split Out node)  
  - loop_over_scenes (SplitInBatches node)  
  - Leo - Improve Prompt (HTTP Request)  
  - Leo - Generate Image (HTTP Request)  
  - Leo - Get imageId (HTTP Request)  
  - Runway - Create Video (HTTP Request)  
  - Runway - Get Video (HTTP Request)  
  - Wait1, Wait2 (Wait nodes)  
  - Execute Workflow2, Execute Workflow3, Execute Workflow5, Execute Workflow6 (sub-workflows)  
  - BackgroundType (If node)  
  - output image (Set node)  
  - output (Set node)

- **Node Details:**

  - **Scenes Mapping**  
    - Type: Set node  
    - Role: Organizes scripts/scenes into an array for batch processing.  
    - Input: Parsed script scenes.  
    - Output: Array of scenes.

  - **Split Out**  
    - Type: Split Out node  
    - Role: Divides the scenes array into individual scene items for iteration.  
    - Output: Single scene per iteration.

  - **loop_over_scenes**  
    - Type: SplitInBatches node  
    - Role: Controls batch size to process scenes sequentially or in small groups.  
    - Input: Individual scenes from Split Out.  
    - Outputs: Routes scene data for visual generation.

  - **BackgroundType**  
    - Type: If node  
    - Role: Decides visual generation method based on background type (Images -> Leonardo AI, else RunwayML).  
    - Input: Scene data.  
    - Outputs: Branches to Leo - Improve Prompt or Runway - Create Video accordingly.

  - **Leo - Improve Prompt**  
    - Type: HTTP Request node  
    - Role: Refines or enhances the prompt for Leonardo AI image generation.  
    - Output: Updated prompt for image generation.

  - **Leo - Generate Image**  
    - Type: HTTP Request node  
    - Role: Sends refined prompt to Leonardo AI to create an image.  
    - Error handling: Continues on error to allow partial results.

  - **Wait1**  
    - Type: Wait node  
    - Role: Waits for Leonardo AI image generation to complete or become available.

  - **Leo - Get imageId**  
    - Type: HTTP Request node  
    - Role: Polls Leonardo AI API to retrieve the generated image ID.

  - **Runway - Create Video**  
    - Type: HTTP Request node  
    - Role: Requests RunwayML to create acting scene videos based on scene data.  
    - On error: Continues to allow downstream error handling.

  - **Wait2**  
    - Type: Wait node  
    - Role: Waits while RunwayML processes video creation.

  - **Runway - Get Video**  
    - Type: HTTP Request node  
    - Role: Polls RunwayML for the status and URL of the generated video.

  - **Execute Workflow2, Execute Workflow3, Execute Workflow5, Execute Workflow6**  
    - Type: Execute Workflow nodes (sub-workflows)  
    - Role: Handle specialized tasks such as post-image generation processing, video finalization, or error handling.

  - **output image & output**  
    - Type: Set nodes  
    - Role: Package and prepare scene asset data (images/videos) for final assembly.

- **Failures & Edge Cases:**  
  - Leonardo AI or RunwayML API timeouts or rate limits.  
  - Image/video generation failure or incomplete data.  
  - Wait nodes timing out or webhook-based waits not triggered.  
  - Sub-workflow dependency failures.

---

#### 2.4 Video Assembly and Editing

- **Overview:**  
  Assembles the generated visuals, captions, and other metadata into the final video using JSON2Video or HeyGen services. Handles polling for rendering completion and error workflows.

- **Nodes Involved:**  
  - json2video : Video Rendering (HTTP Request)  
  - json2video : Check Video Rendering (HTTP Request)  
  - HeyGen (HTTP Request)  
  - HeyGen : Check Video (HTTP Request)  
  - j2v_response (Switch)  
  - heygen_response (Switch)  
  - Wait (Wait node)  
  - Wait4 (Wait node)  
  - Execute Workflow (Error handling workflows)  
  - If_with_heygen (If node)  
  - Code Heygen (Code node)  
  - Code Add Sub (Code node)

- **Node Details:**

  - **json2video : Video Rendering**  
    - Type: HTTP Request node  
    - Role: Sends final video assembly request to JSON2Video API with assets and captions.  
    - Error handling: Continues on error for fallback.

  - **json2video : Check Video Rendering**  
    - Type: HTTP Request node  
    - Role: Polls JSON2Video for rendering status and video URL.  
    - Output: Routes based on rendering success or failure.

  - **HeyGen**  
    - Type: HTTP Request node  
    - Role: Requests HeyGen UGC-style video generation based on avatar and script.  
    - Error handling: Continues on error.

  - **HeyGen : Check Video**  
    - Type: HTTP Request node  
    - Role: Polls HeyGen API to check video generation status.

  - **j2v_response & heygen_response**  
    - Type: Switch nodes  
    - Role: Branch workflow based on JSON2Video or HeyGen API responses for success or failure.

  - **Wait & Wait4**  
    - Type: Wait nodes  
    - Role: Pauses workflow to allow asynchronous video rendering completion.

  - **If_with_heygen**  
    - Type: If node  
    - Role: Determines if HeyGen or JSON2Video path is followed based on input parameters.

  - **Code Heygen & Code Add Sub**  
    - Type: Code nodes  
    - Role: Custom JavaScript code to manipulate API responses, prepare payloads, or integrate captions.

  - **Execute Workflow (Error Handling)**  
    - Type: Execute Workflow nodes  
    - Role: Dedicated error workflows for retry, logging, or alerting on video assembly failures.

- **Potential Failures:**  
  - API authentication or quota exceeded.  
  - Video rendering timeouts or failures.  
  - Incorrect payloads causing API errors.  
  - Delays in asynchronous video availability.

---

#### 2.5 Captions & Post-Processing

- **Overview:**  
  Generates captions for videos using CaptionsAI, polls for caption creation status, and handles errors related to captions integration.

- **Nodes Involved:**  
  - CaptionsAI1 (HTTP Request)  
  - CaptionsAI : Check Poll1 (HTTP Request)  
  - cap_response (Switch)  
  - CAPTIONS Execute ERROR & CAPTIONS Execute ERROR1 (Execute Workflow nodes)  
  - Aggregate (Aggregate node)  
  - Wait6 (Wait node)  
  - Code Add Sub (Code node)

- **Node Details:**

  - **CaptionsAI1**  
    - Type: HTTP Request node  
    - Role: Initiates caption generation request to CaptionsAI service.  
    - Error handling: Continues on error.

  - **CaptionsAI : Check Poll1**  
    - Type: HTTP Request node  
    - Role: Polls CaptionsAI for status of captions generation.

  - **cap_response**  
    - Type: Switch node  
    - Role: Routes workflow based on captions generation success or failure.

  - **CAPTIONS Execute ERROR & CAPTIONS Execute ERROR1**  
    - Type: Execute Workflow nodes  
    - Role: Handle caption generation errors or retries.

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Collects and consolidates caption data for integration.

  - **Wait6**  
    - Type: Wait node  
    - Role: Delays polling intervals or synchronizes caption integration timing.

  - **Code Add Sub**  
    - Type: Code node  
    - Role: Adds or modifies subtitle data into video assembly payload.

- **Failure Modes:**  
  - Caption generation API errors or delays.  
  - Missing or malformed subtitles causing video assembly issues.

---

#### 2.6 Data Management and Tracking

- **Overview:**  
  Manages workflow state, updates Baserow database records with progress, errors, and final video URLs to enable monitoring and iterative management.

- **Nodes Involved:**  
  - Baserow (multiple nodes, including Baserow Processing, Update Script)  
  - Output (Set node)  
  - Various Execute Workflow nodes handling errors and updates.

- **Node Details:**

  - **Baserow Processing**  
    - Type: Baserow node  
    - Role: Updates record status, logs errors or notes at various stages.  
    - Inputs: Data from conditional branches and error handlers.

  - **Update Script**  
    - Type: Baserow node  
    - Role: Updates script fields or scene data after generation.

  - **Baserow (final)**  
    - Type: Baserow node  
    - Role: Writes final video URLs and metadata back to database.

  - **Output**  
    - Type: Set node  
    - Role: Prepares final output data structure before database update.

  - **Execute Workflow (Error Handlers)**  
    - Type: Execute Workflow nodes  
    - Role: Specialized error handling for JSON2Video, HeyGen, captions, etc.

- **Failures:**  
  - API authentication, connectivity, or permission issues with Baserow.  
  - Data consistency errors.

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                            | Input Node(s)                     | Output Node(s)                        | Sticky Note                              |
|-------------------------------|-----------------------------------|--------------------------------------------|----------------------------------|-------------------------------------|------------------------------------------|
| Webhook                       | Webhook                           | Entry point: receives video requests       | -                                | Body                                |                                          |
| Body                         | Set                               | Prepares/normalizes input data              | Webhook                          | Should Process?                     |                                          |
| Should Process?               | If                                | Decides if video should be processed       | Body                            | Switch ScriptType, Baserow Processing |                                          |
| Switch ScriptType             | Switch                            | Routes based on script generation type     | Should Process?                  | Basic LLM Chain, Basic LLM Chain Manual |                                          |
| Basic LLM Chain               | LangChain Chain LLM               | AI-generated script creation                | Switch ScriptType                | Scenes Mapping, Update Script, Execute Workflow4 |                                          |
| Basic LLM Chain Manual        | LangChain Chain LLM               | Manual script processing                     | Switch ScriptType                | Update Script, Scenes Mapping, Execute Workflow4 |                                          |
| OpenAI Chat Model             | LangChain LLM Chat Model          | GPT-4 model for AI script generation       | -                              | Basic LLM Chain                    |                                          |
| OpenAI Chat Model1            | LangChain LLM Chat Model          | GPT-4 model for manual script processing    | -                              | Basic LLM Chain Manual              |                                          |
| Structured Output Parser      | LangChain Output Parser            | Parses AI script output into scenes         | Basic LLM Chain, Basic LLM Chain Manual | Basic LLM Chain, Basic LLM Chain Manual |                                          |
| Scenes Mapping               | Set                               | Converts script to scenes array              | Basic LLM Chain                 | Split Out, Update Script            |                                          |
| Split Out                    | Split Out                        | Splits scenes array into individual scenes  | Scenes Mapping                 | loop_over_scenes                   |                                          |
| loop_over_scenes             | SplitInBatches                   | Processes scenes in batches                   | Split Out                     | If_with_avatar, Leo - Improve Prompt |                                          |
| If_with_avatar               | If                                | Detects avatar usage for captioning          | loop_over_scenes               | Aggregate, Code                    |                                          |
| Aggregate                   | Aggregate                        | Collects caption data                         | If_with_avatar                 | CaptionsAI1                      |                                          |
| Code                        | Code                             | Custom logic for scene or caption processing | If_with_avatar                 | If_with_heygen                   |                                          |
| If_with_heygen              | If                                | Chooses HeyGen or JSON2Video video path     | Code                          | HeyGen, json2video : Video Rendering |                                          |
| Leo - Improve Prompt        | HTTP Request                    | Enhances Leonardo AI prompt                   | loop_over_scenes             | Leo - Generate Image              |                                          |
| Leo - Generate Image        | HTTP Request                    | Requests Leonardo AI to generate image       | Leo - Improve Prompt          | Wait1                           |                                          |
| Wait1                       | Wait                             | Wait for Leonardo AI image generation        | Leo - Generate Image          | Leo - Get imageId                |                                          |
| Leo - Get imageId           | HTTP Request                    | Retrieves Leonardo AI image ID                | Wait1                        | BackgroundType                  |                                          |
| BackgroundType              | If                                | Route to RunwayML or Leonardo AI based on background type | Leo - Get imageId             | Runway - Create Video, output image |                                          |
| Runway - Create Video       | HTTP Request                    | Requests RunwayML video generation            | BackgroundType               | Wait2                          |                                          |
| Wait2                       | Wait                             | Wait for RunwayML video processing            | Runway - Create Video         | Runway - Get Video              |                                          |
| Runway - Get Video          | HTTP Request                    | Polls RunwayML for video URL                   | Wait2                        | output, Execute Workflow6       |                                          |
| output                      | Set                               | Packages video scene output                     | Runway - Get Video            | loop_over_scenes               |                                          |
| Execute Workflow            | Execute Workflow                 | Sub-workflow for script processing             | Switch ScriptType             | -                             |                                          |
| Execute Workflow2           | Execute Workflow                 | Sub-workflow for Runway video processing       | Runway - Create Video         | -                             |                                          |
| Execute Workflow3           | Execute Workflow                 | Sub-workflow after Leonardo image generation  | Leo - Generate Image          | -                             |                                          |
| Execute Workflow4           | Execute Workflow                 | Script refinement and validation                 | Basic LLM Chain, Basic LLM Chain Manual | -                             |                                          |
| Execute Workflow5           | Execute Workflow                 | Post image prompt improvement tasks              | Leo - Improve Prompt          | -                             |                                          |
| Execute Workflow6           | Execute Workflow                 | Post Runway video retrieval tasks                 | Runway - Get Video            | -                             |                                          |
| json2video : Video Rendering | HTTP Request                    | Sends video assembly request to JSON2Video      | If_with_heygen               | Wait                           |                                          |
| json2video : Check Video Rendering | HTTP Request             | Polls JSON2Video rendering status                | Wait                        | j2v_response                   |                                          |
| j2v_response                | Switch                            | Routes based on JSON2Video response success/failure | json2video : Check Video Rendering | Baserow, Wait, Error workflows |                                          |
| HeyGen                      | HTTP Request                    | Requests HeyGen video generation                 | If_with_heygen               | Wait4, heygen Execute ERROR    |                                          |
| HeyGen : Check Video        | HTTP Request                    | Polls HeyGen for video generation status           | Wait4                       | heygen_response                |                                          |
| heygen_response             | Switch                            | Routes based on HeyGen API response success/failure | HeyGen : Check Video         | Code Heygen, Wait4             |                                          |
| Code Heygen                 | Code                             | Processes HeyGen response and prepares payloads   | heygen_response              | json2video : Video Rendering   |                                          |
| Wait                       | Wait                             | Waits for JSON2Video video rendering to finish   | json2video : Video Rendering | json2video : Check Video Rendering |                                          |
| Wait4                      | Wait                             | Waits for HeyGen video rendering to finish       | HeyGen                      | HeyGen : Check Video           |                                          |
| CaptionsAI1                | HTTP Request                    | Initiates captions generation                      | Aggregate                   | Wait6, CAPTIONS Execute ERROR1 |                                          |
| CaptionsAI : Check Poll1   | HTTP Request                    | Polls captions generation status                   | Wait6                       | cap_response                  |                                          |
| cap_response               | Switch                            | Routes based on captions generation success/failure | CaptionsAI : Check Poll1    | Code Add Sub, Wait6, CAPTIONS Execute ERROR |                                          |
| Code Add Sub               | Code                             | Integrates captions into payload                     | cap_response                | json2video : Video Rendering   |                                          |
| Wait6                      | Wait                             | Waits between captions polls                         | CaptionsAI1, cap_response   | CaptionsAI : Check Poll1       |                                          |
| Baserow                    | Baserow                          | Updates records with final video URLs and statuses | j2v_response                | Wait                         |                                          |
| Baserow Processing         | Baserow                          | Handles skipped processing or error logging         | Should Process? false output | -                            |                                          |
| Update Script              | Baserow                          | Updates script or scene data after generation        | Basic LLM Chain, Basic LLM Chain Manual | Scenes Mapping                |                                          |
| json2video Execute ERROR   | Execute Workflow                 | Error handling for JSON2Video video rendering        | json2video : Video Rendering | -                            |                                          |
| json2video Execute ERROR1  | Execute Workflow                 | Additional error handling for JSON2Video              | j2v_response                | -                            |                                          |
| CAPTIONS Execute ERROR     | Execute Workflow                 | Caption generation error handling                     | cap_response                | -                            |                                          |
| CAPTIONS Execute ERROR1    | Execute Workflow                 | Additional caption error handling                      | CaptionsAI1                 | -                            |                                          |
| heygen Execute ERROR       | Execute Workflow                 | HeyGen video generation error handling                | HeyGen                      | -                            |                                          |
| heygen Execute ERROR2      | Execute Workflow                 | Additional HeyGen error handling                       | HeyGen : Check Video        | -                            |                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook  
   - Purpose: Receive video creation requests from Baserow form submissions.  
   - Configure path and method as POST.  
   - Save output JSON structure for downstream processing.

2. **Add a Set Node ("Body")**  
   - Normalize and prepare incoming webhook data (e.g., flatten nested fields).  
   - Pass data onwards.

3. **Add an If Node ("Should Process?")**  
   - Condition: Check if processing should occur (e.g., skip if Status field is set).  
   - True branch proceeds to script generation, False branch routes to error handling.

4. **Add a Switch Node ("Switch ScriptType")**  
   - Route based on script type field (e.g., "ai gen" vs "manual").  
   - True branch: AI script generation.  
   - False branch: Manual script processing.

5. **Configure AI Script Generation Branch:**  
   - Add OpenAI Chat Model Node(s): Configure with OpenAI API key, select GPT-4 or desired model.  
   - Add Basic LLM Chain Node: Connect to OpenAI Chat Model, configure prompt templates for script generation. Set onError to continue error output.  
   - Add Structured Output Parser Node: Parse generated text into structured scene JSON.  
   - Add Execute Workflow Node (sub-workflow #4): For script refinement/validation.

6. **Configure Manual Script Branch:**  
   - Add OpenAI Chat Model Node1: For any minimal processing if needed.  
   - Add Basic LLM Chain Manual Node: Use manual script input, process similarly to AI branch but with minimal generation.  
   - Connect to Update Script Baserow node and Scenes Mapping.

7. **Add Scenes Mapping Set Node:**  
   - Convert script to an array of scenes for iteration.

8. **Add Split Out Node:**  
   - Split scenes array into individual scene items.

9. **Add SplitInBatches Node ("loop_over_scenes"):**  
   - Process scenes sequentially or in batches.

10. **Add If Node ("BackgroundType"):**  
    - Based on background type field, route scenes to either Leonardo AI or RunwayML.

11. **Leonardo AI Path:**  
    - Add HTTP Request Node ("Leo - Improve Prompt"): Modify prompt for image generation.  
    - Add HTTP Request Node ("Leo - Generate Image"): Call Leonardo AI API.  
    - Add Wait Node ("Wait1"): Pause for image generation.  
    - Add HTTP Request Node ("Leo - Get imageId"): Poll for image ID.  
    - Connect to output image Set Node.

12. **RunwayML Path:**  
    - Add HTTP Request Node ("Runway - Create Video"): Request video scene generation.  
    - Add Wait Node ("Wait2"): Pause while waiting for video.  
    - Add HTTP Request Node ("Runway - Get Video"): Poll video URL.  
    - Connect to output Set Node.

13. **Add Set Nodes ("output", "output image"):**  
    - Prepare scene data for final assembly.

14. **Video Assembly:**  
    - Add If Node ("If_with_heygen"): Decide whether to use HeyGen or JSON2Video for final video rendering.  
    - For JSON2Video:  
      - HTTP Request Node ("json2video : Video Rendering"): Submit video assembly request.  
      - Wait Node ("Wait"): Wait for rendering.  
      - HTTP Request Node ("json2video : Check Video Rendering"): Poll rendering status.  
      - Switch Node ("j2v_response"): Handle success or error; connect to Baserow update or error workflows.

    - For HeyGen:  
      - HTTP Request Node ("HeyGen"): Submit UGC-style video request.  
      - Wait Node ("Wait4"): Wait for video generation.  
      - HTTP Request Node ("HeyGen : Check Video"): Poll status.  
      - Switch Node ("heygen_response"): Handle success or error; connect accordingly.

15. **Captions Generation:**  
    - HTTP Request Node ("CaptionsAI1"): Submit captions request.  
    - Wait Node ("Wait6"): Polling delay.  
    - HTTP Request Node ("CaptionsAI : Check Poll1"): Poll captions status.  
    - Switch Node ("cap_response"): Route success or error.  
    - Code Node ("Code Add Sub"): Integrate captions into video payload.  
    - Connect back to video rendering.

16. **Baserow Integration:**  
    - Add Baserow Nodes:  
      - "Baserow Processing" for skipped/error cases.  
      - "Update Script" for script updates.  
      - "Baserow" for final video metadata and status updates.  
    - Configure API credentials and database/table IDs.

17. **Error Handling Sub-Workflows:**  
    - Create sub-workflows for JSON2Video errors, HeyGen errors, and captions errors with Execute Workflow nodes.

18. **Set Up Credentials:**  
    - OpenAI API key for GPT-4.  
    - Leonardo AI API credentials.  
    - RunwayML API credentials.  
    - Baserow API credentials.  
    - HeyGen/Caption AI credentials.

19. **Configure Webhook Trigger:**  
    - Connect webhook to external Baserow form or manual trigger for video requests.

20. **Testing & Deployment:**  
    - Run test inputs to validate each block.  
    - Monitor Baserow for data updates and error logs.  
    - Adjust parameters such as video duration, avatar selection, and styles.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| This workflow leverages multiple AI services integrated through HTTP Requests and LangChain nodes within n8n.           | General architecture note                                                                                                       |
| Baserow is used as a centralized platform for input collection, status tracking, and error logging.                     | https://baserow.io                                                                                                               |
| Leonardo AI generates high-quality images based on prompts refined during the workflow.                                  | Leonardo AI API documentation (private/internal)                                                                                |
| RunwayML is used for dynamic video generation involving avatars or acting scenes.                                        | https://runwayml.com                                                                                                             |
| JSON2Video assembles final video with captions and effects; polling mechanisms handle asynchronous rendering completion.| JSON2Video API docs (internal/private)                                                                                           |
| HeyGen provides UGC-style avatar video generation with voice synthesis.                                                  | HeyGen API documentation                                                                                                        |
| Workflow includes robust error handling with Execute Workflow nodes dedicated to retry and logging mechanisms.           | Error handling best practices                                                                                                   |
| Setup video and Baserow template are provided alongside the workflow JSON for easier onboarding.                        | Provided with workflow package                                                                                                  |
| Recommended to monitor API rate limits and implement proper credential management for all integrated services.          | Best practice note                                                                                                              |

---

This structured documentation should enable advanced users and AI agents to understand, reproduce, and maintain the Short Video Creation Suite workflow efficiently.