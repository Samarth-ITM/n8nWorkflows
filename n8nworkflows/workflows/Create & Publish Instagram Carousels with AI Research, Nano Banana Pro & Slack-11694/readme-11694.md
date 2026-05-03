Create & Publish Instagram Carousels with AI Research, Nano Banana Pro & Slack

https://n8nworkflows.xyz/workflows/create---publish-instagram-carousels-with-ai-research--nano-banana-pro---slack-11694


# Create & Publish Instagram Carousels with AI Research, Nano Banana Pro & Slack

### 1. Workflow Overview

This workflow automates the creation and publishing of Instagram carousel posts from a simple topic, leveraging AI research, content drafting, image generation, and Slack for human approvals. It is designed for social media managers, content creators, and marketing teams aiming to produce high-quality, data-rich Instagram carousels with minimal manual effort while retaining control via approvals.

**Logical Blocks:**

- **1.1 Input Reception and Theme Setup**  
  Receives post theme input from Slack and sets it as a variable.

- **1.2 AI Research and Drafting**  
  Uses AI agents to gather detailed research on the theme and draft a structured Instagram carousel post in JSON format.

- **1.3 Draft Approval via Slack**  
  Sends the draft to Slack for human double approval before proceeding.

- **1.4 Image Prompt Generation and Parsing**  
  Converts the approved draft into detailed image generation prompts for Nano Banana Pro AI and parses JSON responses.

- **1.5 Image Generation with Kie.ai**  
  Sends prompts to Kie.ai for image creation, waits for completion, and retrieves image URLs.

- **1.6 Final Approval and Instagram Publishing**  
  Sends final caption and image preview to Slack for final approval, then posts the carousel to Instagram via Facebook Graph API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Theme Setup

**Overview:**  
This block initiates the workflow on a scheduled trigger, requests a theme input from Slack, and sets the theme variable for subsequent use.

**Nodes Involved:**  
- Schedule Trigger  
- Theme Input (Slack)  
- Set Theme  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow every defined interval (hourly)  
  - Configuration: Interval set to every hour  
  - Input: None  
  - Output: Triggers next node  

- **Theme Input (Slack)**  
  - Type: Slack Node (sendAndWait)  
  - Role: Requests theme input from Slack channel (#example) via a custom form  
  - Configuration: Message prompts user to input theme, input is required, response type is customForm with one field named "theme"  
  - Input: Trigger from Schedule Trigger  
  - Output: Passes Slack response JSON with theme data  

- **Set Theme**  
  - Type: Set Node  
  - Role: Extracts theme string from Slack response JSON and assigns it to variable `thema`  
  - Configuration: Assigns `thema` = `{{$json.data.theme}}`  
  - Input: Slack theme input  
  - Output: Passes theme variable for AI research  

**Edge Cases / Failure Modes:**  
- Slack API authorization issues or channel misconfiguration  
- User doesn’t respond or provides invalid theme input  
- Schedule trigger misfires or is disabled  

---

#### 1.2 AI Research and Drafting

**Overview:**  
Uses two AI agents sequentially: first to research the theme online and summarize data, second to draft a detailed Instagram carousel post structure in JSON.

**Nodes Involved:**  
- Research Agent (LangChain Chain LLM)  
- Drafting Agent (LangChain Chain LLM)  
- JSON Parse (Content)  
- OpenAI Model A (GPT-5-mini)  
- OpenAI Model B (GPT-4o-mini)  

**Node Details:**

- **Research Agent**  
  - Type: LangChain Chain LLM  
  - Role: Performs web search and summarizes deep, valuable content on theme  
  - Configuration: Prompt instructs AI to research theme, output in Markdown  
  - Uses OpenAI Model A as language model  
  - Input: Theme variable `thema`  
  - Output: Research summary text  

- **Drafting Agent**  
  - Type: LangChain Chain LLM  
  - Role: Creates structured draft of Instagram carousel including title, caption, slides with headlines, points, and visual concepts in JSON  
  - Configuration: Prompt includes research content, strict style and tone guidelines, outputs JSON only  
  - Uses OpenAI Model B as language model  
  - Input: Research Agent output  
  - Output: Draft JSON text  

- **JSON Parse (Content)**  
  - Type: Code Node  
  - Role: Parses JSON string output from Drafting Agent, handles failures gracefully by returning raw content with error if parse fails  
  - Input: Drafting Agent JSON text  
  - Output: Parsed JSON object for downstream use  

- **OpenAI Model A & B**  
  - Type: LangChain OpenAI Model Nodes  
  - Role: Provide AI language model backends with specific model versions and web search tool enabled  
  - Credentials: OpenAI API connected  
  - Input/Output: Connects to Research and Drafting Agents respectively  

**Edge Cases / Failure Modes:**  
- AI model quota exceeded or API errors  
- Unexpected AI output format breaking JSON parse  
- Timeout or slow AI responses  
- Network or auth failures with OpenAI API  

---

#### 1.3 Draft Approval via Slack

**Overview:**  
Sends the draft content formatted as a Slack message for double approval; only proceeds if approved.

**Nodes Involved:**  
- Slack Approval (Draft)  
- Approval Loop (If Node)  

**Node Details:**

- **Slack Approval (Draft)**  
  - Type: Slack Node (sendAndWait)  
  - Role: Sends detailed draft message to Slack channel (#example) with formatted list of slides and requests double approval  
  - Configuration: Approval options set to require double approval (two users must approve)  
  - Input: Parsed draft JSON  
  - Output: Approval response JSON  

- **Approval Loop**  
  - Type: If Node  
  - Role: Checks if the Slack approval response contains `approved: true`  
  - Configuration: Condition checks `$json.data.approved === true`  
  - Input: Slack Approval (Draft) approval response  
  - Output: True branch proceeds to Image Prompt Generation; false branch loops back or halts  

**Edge Cases / Failure Modes:**  
- Slack API errors or bot permission issues  
- Users reject or do not respond, causing workflow to stall  
- Misconfiguration of approval options  

---

#### 1.4 Image Prompt Generation and Parsing

**Overview:**  
Transforms the approved draft slides into detailed image generation prompts and parses the AI-generated prompt JSON.

**Nodes Involved:**  
- Image Prompt Gen (LangChain Chain LLM)  
- JSON Parse (Prompts)  
- Split Slides (Item Lists)  

**Node Details:**

- **Image Prompt Gen**  
  - Type: LangChain Chain LLM  
  - Role: Creates detailed prompts optimized for Nano Banana Pro image generation, one per slide  
  - Configuration: Prompt includes constraints for layout, style (minimalist, tech, dark mode), text inclusion, and output JSON format  
  - Uses OpenAI Model B  
  - Input: Parsed draft JSON slides (first slide content example)  
  - Output: JSON with array `visual_prompts` containing slide number and prompt text  

- **JSON Parse (Prompts)**  
  - Type: Code Node  
  - Role: Parses JSON text from Image Prompt Gen, similar error handling as previous JSON Parse node  
  - Input: Image Prompt Gen text output  
  - Output: Parsed JSON object with visual prompts  

- **Split Slides**  
  - Type: Item Lists Node  
  - Role: Splits `visual_prompts` array into individual items for parallel processing (image generation per slide)  
  - Input: Parsed prompts JSON  
  - Output: Individual prompt items  

**Edge Cases / Failure Modes:**  
- AI output malformed JSON or missing fields  
- Parsing failures  
- Large number of slides causing performance issues  
- API quota or timeout issues for AI model  

---

#### 1.5 Image Generation with Kie.ai

**Overview:**  
Sends each image prompt to Kie.ai Nano Banana Pro model for image generation, waits for completion, then fetches image URLs.

**Nodes Involved:**  
- Image Generation (Kie.ai) (HTTP Request)  
- Wait (Generation) (Wait Node)  
- Get Image (Kie.ai) (HTTP Request)  

**Node Details:**

- **Image Generation (Kie.ai)**  
  - Type: HTTP Request  
  - Role: Sends POST request to Kie.ai API to create an image generation task with prompt, aspect ratio 1:1, resolution 1K, output PNG  
  - Configuration: Uses generic HTTP header authentication with Kie.ai API key  
  - Input: Individual prompt item from Split Slides  
  - Output: Task ID for image generation  

- **Wait (Generation)**  
  - Type: Wait Node  
  - Role: Pauses workflow for 100 seconds to allow image generation to complete  
  - Input: After image generation request  
  - Output: Triggers image retrieval  

- **Get Image (Kie.ai)**  
  - Type: HTTP Request  
  - Role: Queries Kie.ai API with task ID to retrieve image generation results, extracts image URLs  
  - Configuration: Uses same Kie.ai API key credential  
  - Input: Task ID from Image Generation node  
  - Output: JSON including image URLs  

**Edge Cases / Failure Modes:**  
- Kie.ai API downtime or authentication failure  
- Image generation task failure or timeout  
- Network delays causing long wait times  
- Missing or invalid image URLs returned  

---

#### 1.6 Final Approval and Instagram Publishing

**Overview:**  
Sends final caption and image preview to Slack for user check, then publishes the carousel post to Instagram via Facebook Graph API after final confirmation.

**Nodes Involved:**  
- Slack Approval (Final)  
- IG Create Container (Facebook Graph API)  
- Wait (Upload) (Wait Node)  
- IG Publish (Facebook Graph API)  

**Node Details:**

- **Slack Approval (Final)**  
  - Type: Slack Node (sendAndWait)  
  - Role: Sends the final post caption and image URL to Slack channel (#example), requests free-text approval/feedback from user  
  - Input: Parsed draft content and image URLs from Get Image node  
  - Output: User response with final approval or edits  

- **IG Create Container**  
  - Type: Facebook Graph API Node  
  - Role: Creates Instagram media container with caption text and image URL  
  - Configuration: Uses Instagram Business Account ID, posts to `/media` edge, HTTP POST  
  - Input: Final caption text and image URL (parsed from Kie.ai response)  
  - Output: Media container ID  

- **Wait (Upload)**  
  - Type: Wait Node  
  - Role: Waits 30 seconds to allow media container processing before publishing  
  - Input: IG Create Container output  
  - Output: Triggers publish step  

- **IG Publish**  
  - Type: Facebook Graph API Node  
  - Role: Publishes the created media container to Instagram feed using container ID  
  - Configuration: Posts to `/media_publish` edge with creation_id parameter  
  - Input: Media container ID from IG Create Container  
  - Output: Instagram post published confirmation  

**Edge Cases / Failure Modes:**  
- Slack API errors or user non-response  
- Facebook Graph API auth errors or permissions issues  
- Instagram API rate limits or failures  
- Delay or failure in media container processing  
- Image URL or caption formatting errors  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                            | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                     |
|-------------------------|--------------------------------|-------------------------------------------|-------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                | Starts workflow on schedule                | None                          | Theme Input (Slack)           | ## 🚀 Automate Instagram Carousel Creation This workflow turns a simple topic into a full Instagram Carousel post using AI agents. ... |
| Theme Input (Slack)     | Slack Node                     | Receives theme input from Slack channel    | Schedule Trigger              | Set Theme                    | ## 🚀 Automate Instagram Carousel Creation This workflow turns a simple topic into a full Instagram Carousel post using AI agents. ... |
| Set Theme               | Set Node                      | Sets theme variable for AI agents          | Theme Input (Slack)            | Research Agent               | ## 🚀 Automate Instagram Carousel Creation This workflow turns a simple topic into a full Instagram Carousel post using AI agents. ... |
| Research Agent          | LangChain Chain LLM            | Researches theme, gathers web info         | Set Theme                     | Drafting Agent               | ## 1. Research & Draft AI Agents research the topic and create a structured draft.              |
| OpenAI Model A          | LangChain OpenAI Model         | Provides GPT-5-mini for research           | Research Agent (ai_languageModel) | Research Agent           |                                                                                                |
| Drafting Agent          | LangChain Chain LLM            | Drafts Instagram carousel post JSON        | Research Agent                | JSON Parse (Content)          | ## 1. Research & Draft AI Agents research the topic and create a structured draft.              |
| OpenAI Model B          | LangChain OpenAI Model         | Provides GPT-4o-mini for drafting and prompts | Drafting Agent (ai_languageModel), Image Prompt Gen (ai_languageModel) | Drafting Agent, Image Prompt Gen |                                                                                                |
| JSON Parse (Content)    | Code Node                     | Parses JSON draft output                    | Drafting Agent                | Slack Approval (Draft)        | ## 2. Human Approval Review the draft before generating images to save credits.                 |
| Slack Approval (Draft)  | Slack Node                     | Sends draft to Slack for double approval   | JSON Parse (Content)          | Approval Loop                | ## 2. Human Approval Review the draft before generating images to save credits.                 |
| Approval Loop           | If Node                       | Checks approval status                      | Slack Approval (Draft)        | Image Prompt Gen (true branch), Drafting Agent (false branch) | ## 2. Human Approval Review the draft before generating images to save credits.                 |
| Image Prompt Gen        | LangChain Chain LLM            | Creates image generation prompts JSON      | Approval Loop                 | JSON Parse (Prompts)          | ## 3. Image Generation Converts the draft into visual prompts and generates images using Kie.ai.|
| JSON Parse (Prompts)    | Code Node                     | Parses image prompt JSON                     | Image Prompt Gen              | Split Slides                 | ## 3. Image Generation Converts the draft into visual prompts and generates images using Kie.ai.|
| Split Slides            | Item Lists Node                | Splits prompts array for parallel image generation | JSON Parse (Prompts)          | Image Generation (Kie.ai)    | ## 3. Image Generation Converts the draft into visual prompts and generates images using Kie.ai.|
| Image Generation (Kie.ai) | HTTP Request                  | Sends image generation requests to Kie.ai | Split Slides                 | Wait (Generation)            | ## 3. Image Generation Converts the draft into visual prompts and generates images using Kie.ai.|
| Wait (Generation)       | Wait Node                     | Waits for image generation to complete      | Image Generation (Kie.ai)     | Get Image (Kie.ai)            | ## 3. Image Generation Converts the draft into visual prompts and generates images using Kie.ai.|
| Get Image (Kie.ai)      | HTTP Request                  | Retrieves generated image URLs              | Wait (Generation)             | Slack Approval (Final)        | ## 3. Image Generation Converts the draft into visual prompts and generates images using Kie.ai.|
| Slack Approval (Final)  | Slack Node                    | Sends final caption and image for approval | Get Image (Kie.ai)            | IG Create Container           |                                                                                                |
| IG Create Container     | Facebook Graph API Node       | Creates Instagram media container           | Slack Approval (Final)        | Wait (Upload)                 |                                                                                                |
| Wait (Upload)           | Wait Node                    | Waits for media container processing        | IG Create Container           | IG Publish                   |                                                                                                |
| IG Publish              | Facebook Graph API Node       | Publishes carousel post to Instagram        | Wait (Upload)                 | None                         |                                                                                                |
| Sticky Note             | Sticky Note                   | Workflow overview and instructions          | None                         | None                         | ## 🚀 Automate Instagram Carousel Creation This workflow turns a simple topic into a full Instagram Carousel post using AI agents. ... |
| Sticky Note Research    | Sticky Note                   | Notes on research and draft block            | None                         | None                         | ## 1. Research & Draft AI Agents research the topic and create a structured draft.              |
| Sticky Note Approval    | Sticky Note                   | Notes on approval block                       | None                         | None                         | ## 2. Human Approval Review the draft before generating images to save credits.                 |
| Sticky Note Image       | Sticky Note                   | Notes on image generation block               | None                         | None                         | ## 3. Image Generation Converts the draft into visual prompts and generates images using Kie.ai.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval to 1 hour  
   - Connect to next node  

2. **Create Slack Node "Theme Input (Slack)"**  
   - Operation: sendAndWait  
   - Channel: `#example` (or your Slack channel)  
   - Message: "【Request】Input the theme for the post."  
   - Form field: One required text input named "theme" with placeholder "e.g. Latest AI Trends"  
   - Connect from Schedule Trigger  

3. **Create Set Node "Set Theme"**  
   - Assign variable `thema` to expression: `{{$json.data.theme}}`  
   - Connect from Slack Input node  

4. **Create LangChain Chain LLM Node "Research Agent"**  
   - Prompt: "You are an excellent Instagram Researcher. ... Theme: {{ $json.thema }} ..." (use full prompt from workflow)  
   - Set Prompt Type: define  
   - Connect OpenAI Model A (GPT-5-mini) as AI language model  
   - Connect from Set Theme node  

5. **Create OpenAI Model Node "OpenAI Model A"**  
   - Model: GPT-5-mini  
   - Enable built-in Web Search tool, medium context size  
   - Connect to Research Agent's AI language model input  

6. **Create LangChain Chain LLM Node "Drafting Agent"**  
   - Prompt: Professional editor creates structured JSON draft from research results (copy full prompt)  
   - Set Prompt Type: define, enable output parser for JSON  
   - Connect OpenAI Model B (GPT-4o-mini) as AI language model  
   - Connect from Research Agent output  

7. **Create OpenAI Model Node "OpenAI Model B"**  
   - Model: GPT-4o-mini  
   - Enable built-in Web Search tool, medium context size  
   - Connect to Drafting Agent and Image Prompt Gen AI language model inputs  

8. **Create Code Node "JSON Parse (Content)"**  
   - Paste JavaScript code to parse JSON safely from Drafting Agent output (as in workflow)  
   - Connect from Drafting Agent output  

9. **Create Slack Node "Slack Approval (Draft)"**  
   - Operation: sendAndWait with double approval option  
   - Channel: `#example`  
   - Message: formatted text listing slide numbers, headlines, and points from parsed draft JSON  
   - Connect from JSON Parse (Content)  

10. **Create If Node "Approval Loop"**  
    - Condition: `$json.data.approved === true` (boolean true)  
    - True path connects to Image Prompt Gen  
    - False path loops back or halts as desired  
    - Connect from Slack Approval (Draft)  

11. **Create LangChain Chain LLM Node "Image Prompt Gen"**  
    - Prompt: Create detailed image generation prompts from draft slides JSON (copy full prompt)  
    - Set Prompt Type: define, enable output parser for JSON  
    - Connect OpenAI Model B AI language model  
    - Connect from Approval Loop true branch  

12. **Create Code Node "JSON Parse (Prompts)"**  
    - Paste JavaScript code to parse image prompt JSON (similar to content parse)  
    - Connect from Image Prompt Gen output  

13. **Create Item Lists Node "Split Slides"**  
    - Field to split out: `visual_prompts`  
    - Connect from JSON Parse (Prompts)  

14. **Create HTTP Request Node "Image Generation (Kie.ai)"**  
    - Method: POST to `https://api.kie.ai/api/v1/jobs/createTask`  
    - Body JSON: model `nano-banana-pro`, prompt from current item, 1:1 aspect ratio, 1K resolution, PNG output  
    - Authentication: HTTP Header Auth with Kie.ai API key  
    - Connect from Split Slides  

15. **Create Wait Node "Wait (Generation)"**  
    - Wait 100 seconds  
    - Connect from Image Generation (Kie.ai)  

16. **Create HTTP Request Node "Get Image (Kie.ai)"**  
    - Method: GET `https://api.kie.ai/api/v1/jobs/recordInfo` with query param taskId from previous node  
    - Authentication: Kie.ai API key  
    - Connect from Wait (Generation)  

17. **Create Slack Node "Slack Approval (Final)"**  
    - Operation: sendAndWait, free text response allowed  
    - Channel: `#example`  
    - Message: shows final caption and image URL for review  
    - Connect from Get Image (Kie.ai)  

18. **Create Facebook Graph API Node "IG Create Container"**  
    - Edge: `media`, Node: your Instagram Business Account ID  
    - HTTP Method: POST  
    - Query Parameters: caption text and image URL (parsed from Slack Approval Final output)  
    - Credentials: Facebook Graph API OAuth  
    - Connect from Slack Approval (Final)  

19. **Create Wait Node "Wait (Upload)"**  
    - Wait 30 seconds  
    - Connect from IG Create Container  

20. **Create Facebook Graph API Node "IG Publish"**  
    - Edge: `media_publish`, Node: Instagram Business Account ID  
    - HTTP Method: POST  
    - Query Parameter: `creation_id` from IG Create Container output  
    - Credentials: Facebook Graph API OAuth  
    - Connect from Wait (Upload)  

21. **Add Sticky Notes** at suitable places to document workflow blocks and instructions for users.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates Instagram carousel creation using AI and Slack approvals, integrating OpenAI, Kie.ai and Facebook APIs. | Workflow Overview and usage instructions.                                                      |
| Setup requires connecting OpenAI API credentials, Slack Bot OAuth with channel access, Facebook Graph API with Instagram Business account, and Kie.ai API key. | Credentials configuration.                                                                     |
| Slack channel `#example` used for input and approvals; replace with your own channel as needed.                        | Slack channel configuration.                                                                   |
| Instagram Business Account ID must be set in Facebook Graph API nodes for media creation and publishing.                 | Instagram API integration.                                                                     |
| Refer to Kie.ai Nano Banana Pro documentation for image generation API details: https://docs.kie.ai/                   | External API resource.                                                                          |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It complies fully with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.