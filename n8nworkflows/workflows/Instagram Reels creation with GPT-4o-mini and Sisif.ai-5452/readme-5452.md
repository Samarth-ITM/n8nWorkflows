Instagram Reels creation with GPT-4o-mini and Sisif.ai

https://n8nworkflows.xyz/workflows/instagram-reels-creation-with-gpt-4o-mini-and-sisif-ai-5452


# Instagram Reels creation with GPT-4o-mini and Sisif.ai

### 1. Workflow Overview

This workflow automates the creation of Instagram Reels videos using AI-generated content and an external video rendering service. It is designed for creators, marketers, and agencies who want to produce Instagram Reels videos hands-free with minimal manual intervention.

**Target Use Cases:**  
- Automated generation of Instagram Reels content ideas based on a configurable topic and style  
- Automated rendering of short vertical videos (5 seconds, 360x640 resolution) using Sisif.ai  
- Periodic execution (every 6 hours by default) to continuously produce fresh Instagram Reels videos  
- Polling Sisif.ai until the video rendering is complete and collecting the final video URL and metadata

**Logical Blocks:**

- **1.1 Scheduling and Configuration Setup:** Sets the trigger and video configuration parameters.  
- **1.2 Idea Generation with GPT-4o-mini:** Uses OpenAI’s GPT-4o-mini to generate creative, trend-aware Instagram Reels ideas based on the configured topic and style.  
- **1.3 Sisif.ai Video Creation and Polling:** Sends the AI-generated idea to Sisif.ai to create a video, then polls the video status until it is ready.  
- **1.4 Final Data Preparation:** Once the video is ready, formats and outputs the final video metadata and URLs.  
- **1.5 Documentation and User Guidance:** Sticky notes provide workflow explanation, usage instructions, and configuration hints.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Configuration Setup

**Overview:**  
This block triggers the workflow every 6 hours by default and sets essential parameters for the Reels creation such as topic, style, duration, and resolution.

**Nodes Involved:**  
- Schedule Trigger  
- Set Reels Config  
- ⬜️ Variables to edit (Sticky Note)

**Node Details:**

- *Schedule Trigger*  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow on a timed interval (every 6 hours)  
  - Configuration: Interval set to 6 hours  
  - Inputs: None (trigger node)  
  - Outputs: Connects to “Set Reels Config” node  
  - Edge Cases: Workflow won’t run if the schedule is disabled; time zone considerations may affect trigger timing  

- *Set Reels Config*  
  - Type: Set  
  - Role: Defines key input variables for the AI prompt and video rendering parameters  
  - Configuration: Sets static values for `reels_topic` (travel-inspired beach vibes), `style` (honeymoon), `duration` (5 seconds), and `resolution` (360x640)  
  - Inputs: From Schedule Trigger  
  - Outputs: To Idea creator  
  - Edge Cases: Changing values here directly impacts all downstream nodes; ensure correct formatting  

- *⬜️ Variables to edit* (Sticky Note)  
  - Purpose: User guidance on which variables to edit in “Set Reels Config”  
  - Content: Lists variables `reels_topic`, `style`, `duration`, `resolution` for customization  

---

#### 1.2 Idea Generation with GPT-4o-mini

**Overview:**  
This block generates a concise, trend-aware Instagram Reels idea using OpenAI’s GPT-4o-mini model, tailored to the specified topic and style.

**Nodes Involved:**  
- Idea creator  
- Structured Output Parser  
- OpenAI Chat Model

**Node Details:**

- *OpenAI Chat Model*  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides the GPT-4o-mini language model for generating text  
  - Configuration: Model set to “gpt-4o-mini” with default options  
  - Credentials: Uses OpenAI API credentials  
  - Inputs: Connected as AI language model input for “Idea creator”  
  - Outputs: Supplies model output to “Structured Output Parser”  
  - Edge Cases: API quota limits, authentication errors, model downtime  

- *Idea creator*  
  - Type: LangChain Chain LLM  
  - Role: Sends prompt to model to generate a JSON-formatted creative idea for Instagram Reels  
  - Configuration: Prompt includes dynamic expressions for `reels_topic` and `style` from input JSON; instructs output JSON with key `idea` and limits length to 30 words  
  - Inputs: From Set Reels Config (variables) and OpenAI Chat Model (model)  
  - Outputs: To “Create Sisif Video” node (after parsing)  
  - Edge Cases: Prompt failure, malformed JSON output, missing input variables  

- *Structured Output Parser*  
  - Type: LangChain Structured Output Parser  
  - Role: Parses JSON output from GPT-4o-mini to extract the `idea` field cleanly  
  - Configuration: JSON schema example defines expected output structure with `idea` string  
  - Inputs: AI output from “OpenAI Chat Model”  
  - Outputs: Parsed JSON to “Idea creator” main output  
  - Edge Cases: Parsing errors if model output is malformed or unexpected  

---

#### 1.3 Sisif.ai Video Creation and Polling

**Overview:**  
This block sends the AI-generated idea to Sisif.ai to create a vertical video, then repeatedly polls the Sisif.ai API until the video is ready.

**Nodes Involved:**  
- Create Sisif Video  
- Wait 10s  
- Check Video Status  
- Video Ready? (IF node)

**Node Details:**

- *Create Sisif Video*  
  - Type: HTTP Request  
  - Role: POST request to Sisif.ai API to initiate video generation with prompt, duration, and resolution  
  - Configuration:  
    - URL: https://sisif.ai/api/videos/generate/  
    - Method: POST  
    - Body parameters: `prompt` from `idea` in JSON, fixed `duration` (5 seconds), fixed `resolution` (360x640)  
    - Authentication: HTTP Bearer token with configured credentials (Sisif.ai API keys)  
  - Inputs: From “Idea creator”  
  - Outputs: To “Wait 10s”  
  - Edge Cases: Authentication failures, API rate limits, malformed prompt causing errors, network timeouts  

- *Wait 10s*  
  - Type: Wait  
  - Role: Pauses workflow for 10 seconds before polling video status  
  - Configuration: Wait for 10 seconds  
  - Inputs: From “Create Sisif Video” and “Video Ready?” (retry loop)  
  - Outputs: To “Check Video Status”  
  - Edge Cases: Workflow delays if Sisif.ai responds slowly or longer processing times  

- *Check Video Status*  
  - Type: HTTP Request  
  - Role: GET request to Sisif.ai API to check current status of the video rendering  
  - Configuration:  
    - URL dynamically composed with video `id` from previous response  
    - Authentication: Same as “Create Sisif Video”  
  - Inputs: From “Wait 10s”  
  - Outputs: To “Video Ready?” IF node  
  - Edge Cases: API errors, missing video ID, network issues  

- *Video Ready?* (IF node)  
  - Type: If (Conditional)  
  - Role: Checks if the video status returned is “READY”  
  - Configuration: Compares `$json.status` to string “READY”  
  - Inputs: From “Check Video Status”  
  - Outputs:  
    - True: To “Prepare Final Data”  
    - False: Loops back to “Wait 10s” for retry  
  - Edge Cases: Status never becomes READY (infinite loop risk), unexpected status values  

---

#### 1.4 Final Data Preparation

**Overview:**  
Once the video is ready, this block formats the final output with detailed video metadata, including direct and watch URLs.

**Nodes Involved:**  
- Prepare Final Data

**Node Details:**

- *Prepare Final Data*  
  - Type: Function  
  - Role: Extracts and formats final video details into a clean JSON output  
  - Configuration:  
    - JavaScript code extracts fields such as prompt, duration, resolution, Sisif.ai ID, video URL, status  
    - Adds a direct watch URL constructed from Sisif.ai ID  
    - Adds a timestamp for completion  
  - Inputs: From “Video Ready?” (True path)  
  - Outputs: Final structured JSON object for downstream use or storage  
  - Edge Cases: Missing fields in input JSON, malformed video URLs  

---

#### 1.5 Documentation and User Guidance

**Overview:**  
Sticky notes provide detailed instructions, explanations, and configuration guidance to the user.

**Nodes Involved:**  
- 🟡 How this workflow works (Sticky Note)  
- ⬜️ Variables to edit (Sticky Note) (covered above)

**Node Details:**

- *🟡 How this workflow works*  
  - Type: Sticky Note  
  - Role: Explains the workflow’s purpose, intended users, setup steps, and customization tips  
  - Content Highlights:  
    - Step-by-step setup instructions for API credentials and workflow activation  
    - Explanation of the 3 main steps: idea generation, video rendering, polling  
    - Customization guidance for frequency, prompt, retry policy  
  - Position: Top-left of workflow for visibility  

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                       | Input Node(s)        | Output Node(s)          | Sticky Note                                                                                   |
|-------------------------|-------------------------------------|------------------------------------|----------------------|-------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                    | Triggers workflow every 6 hours     | None                 | Set Reels Config         |                                                                                               |
| Set Reels Config        | Set                                | Sets input variables for reels      | Schedule Trigger      | Idea creator             | **Edit these in ➜ Set Reels Config** • `reels_topic`, `style`, `duration`, `resolution`      |
| ⬜️ Variables to edit    | Sticky Note                        | User guidance on variables to edit  | None                 | None                    | **Edit these in ➜ Set Reels Config** • `reels_topic`, `style`, `duration`, `resolution`      |
| Idea creator            | LangChain Chain LLM                | Generates Instagram Reels idea      | Set Reels Config      | Create Sisif Video       |                                                                                               |
| Structured Output Parser| LangChain Structured Output Parser| Parses AI output to JSON             | OpenAI Chat Model     | Idea creator             |                                                                                               |
| OpenAI Chat Model       | LangChain OpenAI Chat Model        | Provides GPT-4o-mini model          | Idea creator (AI LM)  | Structured Output Parser |                                                                                               |
| Create Sisif Video      | HTTP Request                      | Sends prompt to Sisif.ai API        | Idea creator          | Wait 10s                 |                                                                                               |
| Wait 10s                | Wait                              | Waits 10 seconds before polling     | Create Sisif Video, Video Ready? (False) | Check Video Status          |                                                                                               |
| Check Video Status      | HTTP Request                      | Polls Sisif.ai for video status     | Wait 10s              | Video Ready?             |                                                                                               |
| Video Ready?            | If                                | Checks if video status is READY     | Check Video Status    | Prepare Final Data, Wait 10s |                                                                                               |
| Prepare Final Data      | Function                         | Prepares final video metadata output| Video Ready? (True)    | None                    |                                                                                               |
| 🟡 How this workflow works | Sticky Note                     | Workflow explanation and instructions| None                 | None                    | Automate Instagram Reels video creation with OpenAI & Sisif.ai. Setup and customization steps.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set interval to every 6 hours  
   - No credentials required  
   - Position: Start of the workflow  

2. **Create Set Node “Set Reels Config”:**  
   - Type: Set  
   - Add string fields:  
     - `reels_topic` = "travel-inspired beach vibes"  
     - `style` = "honeymoon"  
     - `duration` = "5" (seconds)  
     - `resolution` = "360x640"  
   - Connect input from Schedule Trigger  

3. **Create LangChain OpenAI Chat Model Node:**  
   - Type: LangChain LM Chat OpenAI  
   - Model: gpt-4o-mini  
   - Set OpenAI API credentials (OpenAI API key)  
   - No input connections here; connected as AI LM for “Idea creator”  

4. **Create LangChain Chain LLM Node “Idea creator”:**  
   - Type: LangChain Chain LLM  
   - Prompt (define type):  
     ```
     You are a creative strategist for Instagram Reels. Using the topic {{$json.reels_topic}}, generate a concise, trend-aware idea. Mention the following style {{$json.style}}. Short, vivid wording. Max 30 words.

     Output JSON only: { "idea": "..." }
     ```  
   - Configure it to use the OpenAI Chat Model node as AI language model  
   - Enable output parser usage  

5. **Create LangChain Structured Output Parser Node:**  
   - Type: LangChain Structured Output Parser  
   - Set JSON schema example:  
     ```json
     {
       "idea": "Generate video ..."
     }
     ```  
   - Connect OpenAI Chat Model output to this node  
   - Then connect this parser output to the “Idea creator” main output  

6. **Create HTTP Request Node “Create Sisif Video”:**  
   - Method: POST  
   - URL: https://sisif.ai/api/videos/generate/  
   - Authentication: HTTP Bearer Auth with Sisif.ai API Key credentials  
   - Body parameters (JSON):  
     - `prompt`: `={{ $json.output.idea }}` (expression referencing the parsed idea)  
     - `duration`: `5` (seconds)  
     - `resolution`: `360x640`  
   - Connect input from “Idea creator” node  

7. **Create Wait Node “Wait 10s”:**  
   - Wait for 10 seconds  
   - Connect input from “Create Sisif Video” (initial call) and from “Video Ready?” (retry when video not ready)  

8. **Create HTTP Request Node “Check Video Status”:**  
   - Method: GET  
   - URL: `=https://sisif.ai/api/videos/{{ $json.id }}/` (dynamic video ID from previous response)  
   - Authentication: Same Sisif.ai API credentials  
   - Connect input from “Wait 10s”  

9. **Create If Node “Video Ready?”:**  
   - Condition: Check if `$json.status` equals “READY”  
   - True output: Connect to “Prepare Final Data”  
   - False output: Connect back to “Wait 10s” for retry  

10. **Create Function Node “Prepare Final Data”:**  
    - JavaScript code:  
      ```javascript
      const videoData = items[0].json;

      return [{
        json: {
          prompt: videoData.prompt,
          duration: videoData.duration,
          resolution: videoData.resolution,
          sisif_id: videoData.sisif_id,
          video_url: videoData.video_url,
          status: videoData.status,
          watch_url: `https://sisif.ai/app/watch/${videoData.sisif_id}/`,
          completed_at: new Date().toISOString()
        }
      }];
      ```  
    - Connect input from “Video Ready?” True output  

11. **Add Sticky Notes for Documentation:**  
    - Create a large sticky note with workflow explanation, setup instructions, and customization tips (content from “🟡 How this workflow works”)  
    - Create a smaller sticky note near “Set Reels Config” listing editable variables  

12. **Credential Setup:**  
    - Add OpenAI API credentials (OpenAI API Key) under n8n credentials  
    - Add Sisif.ai API credentials (HTTP Bearer token) under n8n credentials  

13. **Activate Workflow:**  
    - Enable Schedule Trigger to run every 6 hours or adjust frequency as desired  
    - Test run the workflow once to verify correct operation  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow automates Instagram Reels video creation using OpenAI GPT-4o-mini and Sisif.ai video rendering API.  | Provided in sticky note “🟡 How this workflow works”                                                        |
| OpenAI API key creation: https://platform.openai.com/api-keys                                                  | Required for GPT-4o-mini model access                                                                       |
| Sisif.ai API key creation: https://sisif.ai/users/api-keys/                                                    | Required for video rendering API authentication                                                             |
| Customization tips include adjusting Cron frequency, modifying prompt tone or content, and retry policies.    | Included in sticky note “🟡 How this workflow works”                                                        |
| Sisif.ai video watch URL format: https://sisif.ai/app/watch/{sisif_id}/                                        | Enables direct video preview                                                                                 |
| Use short video durations and vertical resolutions suitable for Instagram Reels (e.g., 5 seconds, 360x640 px). | Default configuration optimized for Instagram Reels                                                         |

---

**Disclaimer:** The provided content is extracted exclusively from an n8n automated workflow. It complies fully with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.