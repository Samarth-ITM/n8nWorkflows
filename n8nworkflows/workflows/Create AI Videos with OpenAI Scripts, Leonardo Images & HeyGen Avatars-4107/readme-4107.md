Create AI Videos with OpenAI Scripts, Leonardo Images & HeyGen Avatars

https://n8nworkflows.xyz/workflows/create-ai-videos-with-openai-scripts--leonardo-images---heygen-avatars-4107


# Create AI Videos with OpenAI Scripts, Leonardo Images & HeyGen Avatars

---
### 1. Workflow Overview

This n8n workflow, titled **"Create AI Videos with OpenAI Scripts, Leonardo Images & HeyGen Avatars"**, is a comprehensive automation system designed to generate short videos through a fully AI-driven process. It targets creators, marketers, and businesses seeking to rapidly produce professional-quality video content without manual intervention. The workflow orchestrates multiple AI tools and APIs to handle script generation, image creation, avatar rendering, video assembly, captioning, and progress tracking.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preprocessing**: Receives video requests from a Baserow database webhook, validates and prepares data for processing.
- **1.2 Script Generation**: Generates video scripts using OpenAI models either automatically (AI generation) or via manual input.
- **1.3 Visual Asset Generation**:
  - **Leonardo AI Image Generation**: Creates background images based on prompts.
  - **RunwayML Scene Video Creation**: Produces acting scenes when images are not used.
- **1.4 Avatar Generation and Integration**: Uses HeyGen or Captions.ai for avatar video or caption overlays.
- **1.5 Video Assembly and Rendering**: Uses JSON2Video API to assemble video components into final output.
- **1.6 Progress Tracking and Error Handling**: Updates Baserow with statuses, handles retries and errors via sub-workflows and conditional logic.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preprocessing

**Overview:**  
This block triggers the workflow upon receiving new rows in Baserow, extracts input parameters, checks if processing should proceed, and routes the workflow depending on script type.

**Nodes Involved:**  
- Webhook  
- Body (Set)  
- Should Process? (If)  
- Switch ScriptType  
- Baserow Processing

**Node Details:**

- **Webhook**  
  - *Type:* Webhook (HTTP trigger)  
  - *Role:* Receives incoming video creation requests triggered by new rows in Baserow database.  
  - *Configuration:* Listens for POST requests; input includes fields like Topic, ScriptType, AvatarId, etc.  
  - *Connections:* Output to Body node.  
  - *Edge Cases:* Missing or malformed input data, unauthorized webhook calls.

- **Body**  
  - *Type:* Set node  
  - *Role:* Prepares and structures incoming data for further processing.  
  - *Parameters:* Sets variables extracted from webhook payload.  
  - *Connections:* Outputs to Should Process? node.

- **Should Process?**  
  - *Type:* If node  
  - *Role:* Checks if the incoming request meets criteria for processing (e.g., status or flags).  
  - *Connections:* If true, passes to Switch ScriptType; else, can branch to error handling or stop.

- **Switch ScriptType**  
  - *Type:* Switch node  
  - *Role:* Routes workflow based on script type (e.g., AI-generated or manual).  
  - *Connections:* Routes to AI script generation or manual script chain.

- **Baserow Processing**  
  - *Type:* Baserow node  
  - *Role:* Reads or updates records in Baserow, used for managing workflow state.  
  - *Connections:* Parallel to other nodes; manages database state.

---

#### 2.2 Script Generation

**Overview:**  
Generates or processes the video script text either through AI models or manual input, then updates the Baserow record with the generated script.

**Nodes Involved:**  
- Basic LLM Chain (AI script generation)  
- Basic LLM Chain Manual (Manual script processing)  
- OpenAI Chat Model / OpenAI Chat Model1 (Language models)  
- Structured Output Parser  
- Update Script (Baserow)

**Node Details:**

- **Basic LLM Chain / Basic LLM Chain Manual**  
  - *Type:* LangChain LLM Chain node  
  - *Role:* Connects with OpenAI models to generate or process scripts.  
  - *Configuration:* Uses OpenAI Chat models; manual chain handles manual input scripts.  
  - *Connections:* Output connects to Scenes Mapping and Update Script nodes.  
  - *Error Handling:* On error, continues workflow but triggers error sub-workflows downstream.

- **OpenAI Chat Model / OpenAI Chat Model1**  
  - *Type:* LangChain Chat Model node  
  - *Role:* Provides the underlying AI model for text generation (e.g., GPT-4).  
  - *Connections:* Feeds into LLM Chain nodes.

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser  
  - *Role:* Parses AI model output into structured JSON to extract scenes and script details.  
  - *Connections:* Outputs parsed data to LLM Chain nodes for further use.

- **Update Script**  
  - *Type:* Baserow node  
  - *Role:* Updates the Baserow record with the generated script and scenes metadata.  
  - *Connections:* Downstream to scene processing nodes.

---

#### 2.3 Visual Asset Generation

**Overview:**  
Generates visual backgrounds or scenes either via Leonardo AI image generation or RunwayML video creation, depending on the background type selected.

**Nodes Involved:**  
- Scenes Mapping  
- loop_over_scenes (SplitInBatches)  
- If_with_avatar (If)  
- Leo - Improve Prompt  
- Leo - Generate Image  
- Leo - Get imageId  
- Wait1  
- BackgroundType (If)  
- Runway - Create Video  
- Wait2  
- Runway - Get Video

**Node Details:**

- **Scenes Mapping**  
  - *Type:* Set node  
  - *Role:* Maps scenes parsed from script to individual processing units.  
  - *Connections:* Leads to loop_over_scenes.

- **loop_over_scenes**  
  - *Type:* SplitInBatches  
  - *Role:* Processes each scene individually, enabling batch processing.  
  - *Connections:* Routes to If_with_avatar and Leo - Improve Prompt nodes.

- **If_with_avatar**  
  - *Type:* If node  
  - *Role:* Decides whether to generate avatar or proceed with background visuals.  
  - *Connections:* True path leads to avatar generation block; false path continues with Leonardo image generation.

- **Leo - Improve Prompt**  
  - *Type:* HTTP Request  
  - *Role:* Calls Leonardo API to enhance image generation prompt based on scene context.  
  - *Connections:* Outputs to Leo - Generate Image.

- **Leo - Generate Image**  
  - *Type:* HTTP Request  
  - *Role:* Calls Leonardo API to generate an image for scene background.  
  - *Connections:* On success, triggers Wait1; on error, continues error output.

- **Wait1**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow briefly to wait for Leonardo image generation completion.  
  - *Connections:* Proceeds to Leo - Get imageId.

- **Leo - Get imageId**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the generated image ID or URL from Leonardo API.  
  - *Connections:* Routes to BackgroundType node.

- **BackgroundType**  
  - *Type:* If node  
  - *Role:* Decides between background image or Runway scene video creation based on input.  
  - *Connections:* True path leads to Runway - Create Video, false to output image handling.

- **Runway - Create Video**  
  - *Type:* HTTP Request  
  - *Role:* Requests creation of a video scene via RunwayML API.  
  - *Connections:* On success, triggers Wait2.

- **Wait2**  
  - *Type:* Wait node  
  - *Role:* Waits for Runway video creation to complete.  
  - *Connections:* Proceeds to Runway - Get Video.

- **Runway - Get Video**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the completed video scene from RunwayML API.  
  - *Connections:* Outputs to main output node and error handling workflows.

---

#### 2.4 Avatar Generation and Integration

**Overview:**  
Depending on avatar preference, this block generates avatar videos using HeyGen or overlays captions using Captions.ai, then integrates them into the video flow.

**Nodes Involved:**  
- If_with_heygen (If)  
- HeyGen  
- HeyGen : Check Video  
- heygen_response (Switch)  
- Code Heygen (Code)  
- Wait4  
- heygen Execute ERROR / heygen Execute ERROR2 (ExecuteWorkflow)  
- CaptionsAI1  
- CaptionsAI : Check Poll1  
- cap_response (Switch)  
- Code Add Sub  
- CAPTIONS Execute ERROR / CAPTIONS Execute ERROR1 (ExecuteWorkflow)

**Node Details:**

- **If_with_heygen**  
  - *Type:* If node  
  - *Role:* Decides whether to use HeyGen avatars or fallback to other methods.  
  - *Connections:* True path to HeyGen, false path to json2video rendering.

- **HeyGen**  
  - *Type:* HTTP Request  
  - *Role:* Triggers HeyGen API to generate avatar video with specified parameters (avatar ID, voice, position).  
  - *Connections:* On success, proceeds to Wait4; on error, triggers heygen Execute ERROR workflow.

- **HeyGen : Check Video**  
  - *Type:* HTTP Request  
  - *Role:* Polls HeyGen API to check avatar video generation status.  
  - *Connections:* Routes output to heygen_response switch for decision.

- **heygen_response**  
  - *Type:* Switch node  
  - *Role:* Routes based on HeyGen video status (completed, pending, error).  
  - *Connections:* Completed triggers Code Heygen; pending loops back to Wait4; error triggers error workflows.

- **Code Heygen**  
  - *Type:* Code node  
  - *Role:* Processes HeyGen response data and prepares video segments for assembly.  
  - *Connections:* Leads to json2video rendering node.

- **Wait4**  
  - *Type:* Wait node  
  - *Role:* Delays next check to HeyGen video to avoid excessive polling.  
  - *Connections:* Loops back to HeyGen : Check Video.

- **heygen Execute ERROR / heygen Execute ERROR2**  
  - *Type:* ExecuteWorkflow nodes  
  - *Role:* Sub-workflows designated to handle HeyGen-specific errors and logging.

- **CaptionsAI1**  
  - *Type:* HTTP Request  
  - *Role:* Sends video data to Captions.ai API for caption overlay generation.  
  - *Connections:* Routes to Wait6 or error workflows.

- **CaptionsAI : Check Poll1**  
  - *Type:* HTTP Request  
  - *Role:* Polls Captions.ai API for captioning job status.  
  - *Connections:* Routes to cap_response.

- **cap_response**  
  - *Type:* Switch node  
  - *Role:* Routes based on Captions.ai job status (completed, pending, error).  
  - *Connections:* Completed triggers Code Add Sub; pending loops back to Wait6; error triggers error workflows.

- **Code Add Sub**  
  - *Type:* Code node  
  - *Role:* Processes captioning results and prepares data for final video assembly.  
  - *Connections:* Leads to json2video rendering node.

- **CAPTIONS Execute ERROR / CAPTIONS Execute ERROR1**  
  - *Type:* ExecuteWorkflow nodes  
  - *Role:* Sub-workflows for handling captioning errors and logging.

- **Wait6**  
  - *Type:* Wait node  
  - *Role:* Delays polling to Captions.ai to prevent API spamming.

---

#### 2.5 Video Assembly and Rendering

**Overview:**  
This block assembles all generated assets (scripts, images, avatars, captions) into a final video using JSON2Video API, monitors rendering status, and updates Baserow with results.

**Nodes Involved:**  
- output (Set)  
- json2video : Video Rendering  
- Wait  
- json2video : Check Video Rendering  
- j2v_response (Switch)  
- Baserow  
- json2video Execute ERROR / json2video Execute ERROR1 (ExecuteWorkflow)

**Node Details:**

- **output**  
  - *Type:* Set node  
  - *Role:* Prepares the final JSON payload for JSON2Video API including script, images, avatars, captions, and settings.  
  - *Connections:* Feeds into json2video : Video Rendering.

- **json2video : Video Rendering**  
  - *Type:* HTTP Request  
  - *Role:* Sends video assembly request to JSON2Video API.  
  - *Connections:* Wait node to monitor rendering progress or error workflow on failure.

- **Wait**  
  - *Type:* Wait node  
  - *Role:* Pauses to allow video rendering to complete before status check.  
  - *Connections:* Leads to json2video : Check Video Rendering.

- **json2video : Check Video Rendering**  
  - *Type:* HTTP Request  
  - *Role:* Polls JSON2Video API to check rendering status and retrieve final video URL.  
  - *Connections:* Routes to j2v_response switch.

- **j2v_response**  
  - *Type:* Switch node  
  - *Role:* Routes workflow depending on rendering status (completed, pending, error).  
  - *Connections:* Completed triggers Baserow update and Wait node; pending loops back to Wait; error triggers error workflows.

- **Baserow**  
  - *Type:* Baserow node  
  - *Role:* Updates Baserow record with final video URL, status, and logs.  
  - *Connections:* Terminal node for successful rendering.

- **json2video Execute ERROR / json2video Execute ERROR1**  
  - *Type:* ExecuteWorkflow nodes  
  - *Role:* Sub-workflows for handling rendering errors and retries.

---

#### 2.6 Progress Tracking and Error Handling

**Overview:**  
Monitors errors or failures in any block, logs them to Baserow, and stops workflow execution gracefully to allow manual intervention or retries.

**Nodes Involved:**  
- Baserow Error  
- Stop and Error  
- When Executed by Another Workflow (ExecuteWorkflowTrigger)

**Node Details:**

- **Baserow Error**  
  - *Type:* Baserow node  
  - *Role:* Logs error details into Baserow for monitoring.  
  - *Connections:* Leads to Stop and Error node.

- **Stop and Error**  
  - *Type:* Stop and Error node  
  - *Role:* Stops workflow execution and throws an error, signaling failure.  
  - *Connections:* Terminal node.

- **When Executed by Another Workflow**  
  - *Type:* ExecuteWorkflowTrigger  
  - *Role:* Allows this workflow to be triggered as a sub-workflow from other workflows, preserving error handling behavior.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                            | Input Node(s)               | Output Node(s)                 | Sticky Note                     |
|-------------------------------|----------------------------------|--------------------------------------------|-----------------------------|-------------------------------|--------------------------------|
| Webhook                       | Webhook                          | Entry point: receives Baserow webhook events | -                           | Body                          |                                |
| Body                         | Set                              | Prepares input data                         | Webhook                     | Should Process?               |                                |
| Should Process?               | If                               | Checks if processing should continue       | Body                        | Switch ScriptType, Baserow Processing |                                |
| Switch ScriptType             | Switch                           | Routes based on script type (AI/manual)    | Should Process?             | If, Basic LLM Chain           |                                |
| If                           | If                               | Conditions for AI script generation         | Switch ScriptType            | Basic LLM Chain Manual, Execute Workflow |                                |
| Basic LLM Chain              | LangChain LLM Chain              | AI script generation                        | If                          | Scenes Mapping, Update Script, Execute Workflow4 |                                |
| Basic LLM Chain Manual       | LangChain LLM Chain              | Manual script processing                    | If                          | Update Script, Scenes Mapping, Execute Workflow4 |                                |
| OpenAI Chat Model            | LangChain Chat Model             | AI language model for script generation     | -                           | Basic LLM Chain               |                                |
| OpenAI Chat Model1           | LangChain Chat Model             | AI language model for manual script          | -                           | Basic LLM Chain Manual        |                                |
| Structured Output Parser     | LangChain Output Parser          | Parses AI output into structured scenes     | Basic LLM Chain, Basic LLM Chain Manual | Basic LLM Chain, Basic LLM Chain Manual |                                |
| Update Script                | Baserow                         | Updates Baserow record with script and scenes | Basic LLM Chain, Basic LLM Chain Manual | -                           |                                |
| Scenes Mapping               | Set                              | Maps scenes data for processing             | Basic LLM Chain, Basic LLM Chain Manual | Split Out                    |                                |
| loop_over_scenes             | SplitInBatches                  | Processes each scene one by one              | Scenes Mapping, output image | If_with_avatar, Leo - Improve Prompt |                                |
| If_with_avatar              | If                               | Decides avatar presence, routes accordingly | loop_over_scenes            | Aggregate, Code               |                                |
| Aggregate                   | Aggregate                       | Aggregates avatar data for processing       | If_with_avatar              | CaptionsAI1                  |                                |
| Code                        | Code                            | Custom processing logic for avatar data     | If_with_avatar              | If_with_heygen               |                                |
| If_with_heygen              | If                               | Chooses between HeyGen avatar or json2video | Code                        | HeyGen, json2video : Video Rendering |                                |
| HeyGen                      | HTTP Request                    | Triggers HeyGen avatar video generation      | If_with_heygen              | Wait4, heygen Execute ERROR  |                                |
| Wait4                       | Wait                            | Waits for HeyGen video to be ready           | HeyGen                      | HeyGen : Check Video         |                                |
| HeyGen : Check Video        | HTTP Request                    | Polls HeyGen for avatar video status          | Wait4                       | heygen_response              |                                |
| heygen_response             | Switch                         | Routes HeyGen status responses                | HeyGen : Check Video        | Code Heygen, Wait4, heygen Execute ERROR2 |                                |
| Code Heygen                 | Code                            | Processes HeyGen video response                | heygen_response             | json2video : Video Rendering |                                |
| heygen Execute ERROR        | ExecuteWorkflow                 | Handles HeyGen errors                         | HeyGen, heygen_response     | -                           |                                |
| heygen Execute ERROR2       | ExecuteWorkflow                 | Handles HeyGen errors                         | heygen_response             | -                           |                                |
| CaptionsAI1                | HTTP Request                    | Sends video to Captions.ai for captions       | Aggregate                   | Wait6, CAPTIONS Execute ERROR1 |                                |
| Wait6                      | Wait                            | Waits for Captions.ai processing               | CaptionsAI1, cap_response   | CaptionsAI : Check Poll1     |                                |
| CaptionsAI : Check Poll1   | HTTP Request                    | Polls Captions.ai for captioning status        | Wait6                       | cap_response                 |                                |
| cap_response               | Switch                         | Routes Captions.ai responses                    | CaptionsAI : Check Poll1    | Code Add Sub, Wait6, CAPTIONS Execute ERROR |                                |
| Code Add Sub               | Code                            | Processes captioning results                     | cap_response                | json2video : Video Rendering |                                |
| CAPTIONS Execute ERROR     | ExecuteWorkflow                 | Handles Captions.ai errors                       | cap_response                | -                           |                                |
| CAPTIONS Execute ERROR1    | ExecuteWorkflow                 | Handles Captions.ai errors                       | CaptionsAI1                 | -                           |                                |
| output                     | Set                              | Prepares JSON payload for video rendering       | Runway - Get Video, j2v_response | loop_over_scenes           |                                |
| output image               | Set                              | Prepares output data for image-based scenes      | BackgroundType              | loop_over_scenes             |                                |
| Leo - Improve Prompt       | HTTP Request                    | Enhances Leonardo image generation prompt        | loop_over_scenes            | Leo - Generate Image         |                                |
| Leo - Generate Image       | HTTP Request                    | Requests Leonardo AI to generate image             | Leo - Improve Prompt        | Wait1                       |                                |
| Wait1                      | Wait                            | Waits for Leonardo image generation                | Leo - Generate Image        | Leo - Get imageId            |                                |
| Leo - Get imageId          | HTTP Request                    | Retrieves generated Leonardo image ID               | Wait1                       | BackgroundType               |                                |
| BackgroundType             | If                               | Chooses background type: image or Runway video       | Leo - Get imageId           | Runway - Create Video, output image |                                |
| Runway - Create Video      | HTTP Request                    | Requests scene video creation via RunwayML API       | BackgroundType              | Wait2                       |                                |
| Wait2                      | Wait                            | Waits for RunwayML video creation completion          | Runway - Create Video       | Runway - Get Video           |                                |
| Runway - Get Video         | HTTP Request                    | Retrieves completed RunwayML video                     | Wait2                       | output, Execute Workflow6    |                                |
| json2video : Video Rendering | HTTP Request                  | Sends final video composition request to JSON2Video API | Code Heygen, Code Add Sub  | Wait, json2video Execute ERROR |                                |
| Wait                       | Wait                            | Waits for final video rendering completion             | json2video : Video Rendering | json2video : Check Video Rendering |                                |
| json2video : Check Video Rendering | HTTP Request            | Polls JSON2Video API for rendering status               | Wait                        | j2v_response                |                                |
| j2v_response               | Switch                         | Routes JSON2Video rendering responses                   | json2video : Check Video Rendering | Baserow, Wait, json2video Execute ERROR1 |                                |
| Baserow                    | Baserow                        | Updates Baserow record with final video URL and status   | j2v_response                | -                           |                                |
| json2video Execute ERROR   | ExecuteWorkflow                 | Handles JSON2Video rendering errors                      | json2video : Video Rendering | -                           |                                |
| json2video Execute ERROR1  | ExecuteWorkflow                 | Handles JSON2Video rendering errors                      | j2v_response                | -                           |                                |
| Baserow Error              | Baserow                        | Logs workflow errors in Baserow                            | When Executed by Another Workflow | Stop and Error             |                                |
| Stop and Error             | Stop and Error                 | Stops workflow execution on critical error                | Baserow Error               | -                           |                                |
| When Executed by Another Workflow | ExecuteWorkflowTrigger     | Allows this workflow to be invoked as a sub-workflow        | -                           | Baserow Error               |                                |
| Execute Workflow           | ExecuteWorkflow                | Sub-workflow calls at various points                        | If, Basic LLM Chain         | -                           |                                |
| Execute Workflow2          | ExecuteWorkflow                | Sub-workflow call after Runway - Create Video              | Runway - Create Video       | -                           |                                |
| Execute Workflow3          | ExecuteWorkflow                | Sub-workflow call after Leonardo image generation           | Leo - Generate Image        | -                           |                                |
| Execute Workflow4          | ExecuteWorkflow                | Sub-workflow call after script generation                    | Basic LLM Chain, Basic LLM Chain Manual | -                     |                                |
| Execute Workflow5          | ExecuteWorkflow                | Sub-workflow call after Leonardo prompt improvement          | Leo - Improve Prompt        | -                           |                                |
| Execute Workflow6          | ExecuteWorkflow                | Sub-workflow call after Runway - Get Video                   | Runway - Get Video          | -                           |                                |
| CAPTIONS Execute ERROR     | ExecuteWorkflow                | Error handling for Captions.ai                                | cap_response                | -                           |                                |
| CAPTIONS Execute ERROR1    | ExecuteWorkflow                | Error handling for Captions.ai                                | CaptionsAI1                 | -                           |                                |
| heygen Execute ERROR       | ExecuteWorkflow                | HeyGen error handling                                         | HeyGen                      | -                           |                                |
| heygen Execute ERROR2      | ExecuteWorkflow                | HeyGen error handling                                         | heygen_response             | -                           |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook  
   - Configure to receive POST requests triggered from Baserow on new video row creation.  
   - No authentication or set up expected headers.  
   - Connect output to a Set node.

2. **Create a Set Node (Body)**  
   - Extract and set all incoming parameters such as Topic, ScriptType, AvatarId, ScenesNumber, BackgroundType, etc.  
   - Connect to an If node to determine if processing should proceed.

3. **Add If Node (Should Process?)**  
   - Configure conditions to check if the row is ready for processing (e.g., Status is empty or flagged for processing).  
   - True path connects to a Switch node for ScriptType; false path can connect to error handling or terminate.

4. **Add Switch Node (Switch ScriptType)**  
   - Setup cases for script generation method: ‘ai gen’ or ‘manual’.  
   - ‘ai gen’ path leads to AI script generation chain; ‘manual’ path leads to manual script processing chain.

5. **Add LangChain Nodes for Script Generation**  
   - Add OpenAI Chat Model node configured with your OpenAI credentials.  
   - Create Basic LLM Chain node connected to the Chat Model for AI script generation; similarly, Basic LLM Chain Manual for manual input scripts.  
   - Connect outputs to a Structured Output Parser node to parse scene info.

6. **Add Structured Output Parser Node**  
   - Configure to parse AI output into JSON with scenes, timing, captions, etc.  
   - Connect output to Update Script node and Scenes Mapping node.

7. **Add Baserow Node (Update Script)**  
   - Configure to update the row with the generated script and scene metadata.  
   - Connect output to Scenes Mapping node.

8. **Add Set Node (Scenes Mapping)**  
   - Map scenes data for batch processing.  
   - Connect to SplitInBatches node.

9. **Add SplitInBatches Node (loop_over_scenes)**  
   - Configure batch size = 1 to process scenes sequentially.  
   - Connect to If node (If_with_avatar).

10. **Add If Node (If_with_avatar)**  
    - Check if avatar generation is needed based on input (AvatarId presence).  
    - True path connects to Aggregate node; false path connects to Code node for avatar data processing.

11. **Add Aggregate Node**  
    - Aggregate avatar data if needed.  
    - Connect to CaptionsAI1 HTTP Request node.

12. **Add HTTP Request Node (CaptionsAI1)**  
    - Configure to send video data to Captions.ai API for caption overlay generation.  
    - Connect output to Wait node (Wait6).

13. **Add Wait Node (Wait6)**  
    - Set delay (e.g., 10-30 seconds) for caption processing.  
    - Connect to HTTP Request node (CaptionsAI : Check Poll1).

14. **Add HTTP Request Node (CaptionsAI : Check Poll1)**  
    - Poll Captions.ai API for job status.  
    - Connect to Switch node (cap_response).

15. **Add Switch Node (cap_response)**  
    - Routes based on caption job status: completed → Code Add Sub, pending → Wait6, error → CAPTIONS Execute ERROR workflows.

16. **Add Code Node (Code Add Sub)**  
    - Process captioning results into video assembly format.  
    - Connect output to JSON2Video : Video Rendering node.

17. **Add HTTP Request Nodes for Leonardo AI Image Generation**  
    - Leo - Improve Prompt: Enhance prompts for images.  
    - Leo - Generate Image: Request image generation.  
    - Leo - Get imageId: Retrieve image ID post generation.  
    - Connect these sequentially with a Wait node (Wait1) after Generate Image for processing delay.

18. **Add If Node (BackgroundType)**  
    - Determines whether to use Leonardo images or RunwayML videos as backgrounds.  
    - True path: Runway - Create Video, False path: output image Set node.

19. **Add HTTP Request Nodes for RunwayML Video Scenes**  
    - Runway - Create Video: Requests scene creation.  
    - Wait2 node to allow processing.  
    - Runway - Get Video: Retrieves finished video.  
    - Connect these sequentially.

20. **Add If Node (If_with_heygen)**  
    - Decides whether to generate avatar videos using HeyGen or proceed directly to video rendering.

21. **Add HTTP Request Nodes for HeyGen**  
    - HeyGen: Submits avatar video generation request.  
    - Wait4: Waits before polling status.  
    - HeyGen : Check Video: Polls status.  
    - heygen_response Switch node to handle success, pending, or error.  
    - Code Heygen: Processes HeyGen result.  
    - Connect error paths to heygen Execute ERROR workflows.

22. **Add HTTP Request Node (json2video : Video Rendering)**  
    - Sends final video assembly request to JSON2Video API with all components.  
    - Connect to Wait node.

23. **Add Wait Node (Wait)**  
    - Waits for video rendering to complete.

24. **Add HTTP Request Node (json2video : Check Video Rendering)**  
    - Polls JSON2Video API for rendering status.

25. **Add Switch Node (j2v_response)**  
    - Routes based on rendering status: completed → Baserow update, pending → Wait, error → json2video Execute ERROR workflows.

26. **Add Baserow Node**  
    - Updates record with final video URL and status.

27. **Add Error Handling Nodes**  
    - Baserow Error: Logs errors.  
    - Stop and Error: Terminates workflow on critical failures.

28. **Add ExecuteWorkflow Nodes for Sub-Workflows**  
    - For error handling and specialized processing (HeyGen errors, Captions.ai errors, JSON2Video errors, etc.).  
    - Configure each sub-workflow with appropriate inputs and outputs.

29. **Configure Credentials**  
    - OpenAI: API key for Chat models.  
    - Leonardo AI: API credentials for image generation.  
    - RunwayML: API access tokens.  
    - HeyGen: API key and authentication.  
    - Captions.ai: API key.  
    - Baserow: API token and workspace details.

30. **Connect all nodes following the logical flow described.**  
    - Validate all expressions, variable mappings, and ensure error paths are correctly connected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Best Short Content AI System - Watch Video                                                                                       | https://www.loom.com/share/4e8af8c4cc6b4214a245b0f2c2458577                                            |
| Explore more workflows and buy custom automations at adamcrafts.cloudysoftwares.com                                               | https://adamcrafts.cloudysoftwares.com                                                                 |
| For custom workflow requests contact adamcrafts@cloudysoftwares.com                                                              | mailto:adamcrafts@cloudysoftwares.com                                                                  |
| This workflow requires API credentials for OpenAI, Leonardo AI, RunwayML, HeyGen, Captions.ai, and Baserow (free tier available) | Setup instructions included in the workflow description and video guides                               |
| Designed to generate professional short videos with zero manual editing, ideal for social media and marketing content at scale   | Use with caution on API rate limits and manage error handling for smooth automation                     |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated n8n workflow. It fully complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.