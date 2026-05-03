🤖 AI content generation for Auto Service 🚘 Automate your social media📲!

https://n8nworkflows.xyz/workflows/---ai-content-generation-for-auto-service----automate-your-social-media----4600


# 🤖 AI content generation for Auto Service 🚘 Automate your social media📲!

### 1. Workflow Overview

This n8n workflow automates AI-powered content generation and social media posting for an Auto Service business. It is designed to produce engaging, professional Telegram posts daily, generate matching AI-driven images, and distribute content across multiple social media platforms (Telegram, Facebook, LinkedIn, Twitter/X). The logic can be adapted to any niche by customizing prompts and AI models.

The workflow is structurally divided into the following logical blocks:

- **1.1 Trigger Inputs:** Multiple entry points that initiate the content generation process based on schedule, manual execution, or new Google Sheets rows.
- **1.2 AI Content Generation:** Uses Langchain agents and OpenAI models to research, draft, and finalize the Telegram post text leveraging real-time internet research with Tavily.
- **1.3 AI Image Generation:** Converts Telegram posts into detailed image prompts and generates photorealistic images using OpenAI’s GPT-image model and other supported image generation APIs.
- **1.4 Content Distribution:** Splits the generated content and posts it simultaneously to Telegram, Facebook, LinkedIn, and Twitter (X).
- **1.5 Optional AI Model Selection:** Multiple alternative AI chat model nodes are present for customization but are not connected by default.
- **1.6 Auxiliary Notes and Branding:** Several sticky notes provide guidance, branding, and visual references.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger Inputs  
**Overview:**  
This block initiates the workflow via three possible triggers: scheduled daily execution, manual trigger (for testing or ad hoc runs), and new rows added in a Google Sheets document.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking Execute workflow (Manual Trigger)  
- Google Sheets Trigger  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configured to trigger daily at 09:00 hours (9 AM).  
  - Input: None (time-based)  
  - Output: Starts content generation flow.  
  - Failure Modes: Cron misconfiguration, timezone issues.  

- **When clicking Execute workflow**  
  - Type: Manual Trigger  
  - Used for manual initiation of the workflow.  
  - Input: None (manual user action)  
  - Output: Starts content generation flow.  
  - Failure Modes: User not triggering node.  

- **Google Sheets Trigger**  
  - Type: Google Sheets Trigger  
  - Watches for new rows added in column "Links for articles to refer" (range A2:A10) in a specified Google Sheet.  
  - Polls every minute.  
  - Input: New row data containing article links.  
  - Output: Triggers content generation with article reference.  
  - Failure Modes: Google OAuth expiration, API rate limits, incorrect sheet/range configuration.  

---

#### 2.2 AI Content Generation  
**Overview:**  
This block performs real-time internet research using Tavily, then generates a professional Telegram post text about auto repair topics, incorporating up-to-date facts.

**Nodes Involved:**  
- GENERATE TEXT (Langchain Agent)  
- Tavily Internet Search (AI Tool)  

**Node Details:**  

- **GENERATE TEXT**  
  - Type: Langchain Agent (AI language model orchestrator)  
  - Purpose: Create engaging, educational Telegram posts under 1024 characters for Auto Service.  
  - Configuration:  
    - Custom prompt instructs the agent to conduct live research with Tavily, write professional and engaging posts including hashtags, minimal emojis, source attribution, and calls to action.  
    - References article links from trigger input.  
    - Output is the Telegram post text only (no explanations).  
  - Input: Trigger data + Tavily research results (via AI tool connection).  
  - Output: Telegram post text in JSON field `output`.  
  - Failure Modes: AI API errors, Tavily API errors, prompt misinterpretation, internet downtime.  

- **Tavily Internet Search**  
  - Type: External AI research tool node  
  - Purpose: Provide up-to-date internet research based on the post topic.  
  - Configured to search recent content (last 7 days), advanced depth, returning 1 result with answer included.  
  - Input: Query built from trigger data (article link or topic).  
  - Output: Research result fed into GENERATE TEXT agent for informed content generation.  
  - Failure Modes: API key expiration, rate limits, network failures.  

---

#### 2.3 AI Image Generation  
**Overview:**  
This block transforms the generated Telegram post text into a detailed image generation prompt, then creates a photorealistic image using OpenAI’s GPT-image model.

**Nodes Involved:**  
- GENERATE PROMPT (Langchain Agent)  
- OPENAI WRITES PROMPTS (OpenAI Chat)  
- OPENAI GENERATES IMAGE (OpenAI Image generation)  
- Split Out  

**Node Details:**  

- **GENERATE PROMPT**  
  - Type: Langchain Agent  
  - Purpose: Convert Telegram post text into a detailed, hyper-realistic image prompt with instructions for 16K photographic resolution and AI super-resolution guidance.  
  - Input: Telegram post text from GENERATE TEXT node.  
  - Output: Detailed image prompt text.  
  - Failure Modes: AI chat API errors, prompt generation failures.  

- **OPENAI WRITES PROMPTS**  
  - Type: OpenAI Chat model (GPT-4.1)  
  - Purpose: (Present but not directly connected) Alternative or backup prompt writing node to refine image prompts.  
  - Failure Modes: API limits, model unavailability.  

- **OPENAI GENERATES IMAGE**  
  - Type: OpenAI Image generation (GPT-image-1)  
  - Purpose: Generate photorealistic image from prompt created by GENERATE PROMPT.  
  - Input: Image prompt text.  
  - Output: Image binary data (photo).  
  - Failure Modes: API quota exceeded, network errors, prompt rejection.  

- **Split Out**  
  - Type: Split Out (splitting JSON array or field)  
  - Purpose: Extract image data from OpenAI image generation response for distribution.  
  - Input: Image generation output.  
  - Output: Individual image content for posting nodes.  
  - Failure Modes: JSON parsing errors, missing fields.  

---

#### 2.4 Content Distribution  
**Overview:**  
This block posts the generated content and images to multiple social media platforms simultaneously.

**Nodes Involved:**  
- Telegram  
- Facebook  
- LinkedIn  
- X (Twitter)  
- Split Out (input node for distribution)  

**Node Details:**  

- **Telegram**  
  - Type: Telegram node (sendPhoto operation)  
  - Sends the generated image as a photo with the generated Telegram post text as caption.  
  - Requires Telegram API credentials.  
  - Inputs: Image binary data and caption text.  
  - Failure Modes: API token expiration, chat ID errors, network issues.  

- **Facebook**  
  - Type: Facebook Graph API (POST method)  
  - Posts content to Facebook page or profile.  
  - Requires Facebook OAuth2 credentials.  
  - Input: Text content (from Split Out node).  
  - Failure Modes: OAuth token expiration, permissions errors, API changes.  

- **LinkedIn**  
  - Type: LinkedIn node (community management)  
  - Publishes post text on LinkedIn under a configured person ID.  
  - Requires LinkedIn OAuth2 credentials.  
  - Input: Text content.  
  - Failure Modes: Permission issues, token expiration.  

- **X (Twitter)**  
  - Type: Twitter node (X)  
  - Posts generated content on Twitter/X.  
  - Requires Twitter OAuth credentials.  
  - Input: Text content.  
  - Failure Modes: Rate limits, token issues.  

---

#### 2.5 Optional AI Model Selection (Not connected by default)  
**Overview:**  
Several nodes representing alternative AI chat models (Mistral Cloud, OpenRouter, Anthropic Claude, Google Gemini, xAI Grok, DeepSeek, HuggingFace, Ollama, Azure OpenAI) are included for customization or future enhancements.

**Nodes Involved:**  
- Mistral Cloud Chat Model  
- OpenRouter Chat Model  
- Anthropic Chat Model  
- Google Gemini Chat Model  
- xAI Grok Chat Model  
- DeepSeek Chat Model  
- Hugging Face Inference Model  
- Ollama Chat Model  
- Azure OpenAI Chat Model  

**Node Details:**  
- Each node is an AI chat model with specific credentials and model selections.  
- Can be swapped or added in the AI content generation block to diversify or optimize text generation.  
- Failure modes depend on each API’s uptime, auth tokens, and usage limits.  

---

#### 2.6 Auxiliary Nodes: Image Generation APIs (Not connected by default)  
**Overview:**  
Various HTTP request nodes allow integration with external image generation services for alternative or additional visual content.

**Nodes Involved:**  
- Freepik API  
- Runware API  
- Clipdrop API  
- Ideogram API  
- Replicate API  
- Imagen Google API  
- Runway Images  
- Minimax Images  
- Kling Images  
- Leonardo Images  
- APITemplate.io  

**Node Details:**  
- Each node is configured for its respective service with typical POST JSON or multipart form-data requests.  
- Credentials and API keys are required for operation.  
- These nodes provide options for enhanced, varied image generation beyond OpenAI.  
- Currently, no direct connections; can be integrated or used for experimentation.  
- Failure modes include API limits, key expiry, or request formatting errors.  

---

#### 2.7 Sticky Notes and Branding  
**Overview:**  
Multiple sticky notes provide visual guidance, branding, example images, and workflow usage instructions.

**Notes Include:**  
- Source example image link.  
- Branding credit with clickable link to author profile.  
- Step-by-step user guide references with images.  
- Description of workflow’s universal logic applicability and encouragement to customize.  
- Visual design references for prompt editing and AI model setup.  

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                  | Input Node(s)             | Output Node(s)                                | Sticky Note                                                                                                  |
|---------------------------|----------------------------------|---------------------------------|---------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger                 | Scheduled daily workflow start  | None                      | GENERATE TEXT                                 | # START - Choose a Trigger![Guide](https://i.ibb.co/d41JsL8q/Screenshot-2025-05-30-122423-1.jpg#full-width#full-width) |
| When clicking Execute workflow | Manual Trigger                 | Manual workflow start           | None                      | GENERATE TEXT                                 | # START - Choose a Trigger![Guide](https://i.ibb.co/d41JsL8q/Screenshot-2025-05-30-122423-1.jpg#full-width#full-width) |
| Google Sheets Trigger     | Google Sheets Trigger            | Trigger on new Google Sheets row| None                      | GENERATE TEXT                                 | # START - Choose a Trigger![Guide](https://i.ibb.co/d41JsL8q/Screenshot-2025-05-30-122423-1.jpg#full-width#full-width) |
| Tavily Internet Search    | Tavily AI Tool                   | Internet research for content   | GENERATE TEXT (as AI_tool)| GENERATE TEXT                                 |                                                                                                              |
| GENERATE TEXT            | Langchain Agent                  | Generates Telegram post text    | Schedule Trigger, Manual Trigger, Google Sheets Trigger, Tavily Internet Search | GENERATE PROMPT                                |                                                                                                              |
| GENERATE PROMPT          | Langchain Agent                  | Creates detailed image prompts  | GENERATE TEXT             | OPENAI GENERATES IMAGE                         | ### Edit prompt and system message up for you, customize llm and search links, add your own prompts database ![](https://i.ibb.co/TxQrh405/erasebg-transformed-removebg-preview.png#full-width) |
| OPENAI GENERATES IMAGE    | OpenAI Image Generation          | Generates photorealistic images | GENERATE PROMPT           | Split Out                                      | ### Set up Ai model for generating images, customize prompt up for you ![](https://i.ibb.co/TxQrh405/erasebg-transformed-removebg-preview.png#full-width) |
| Split Out                | Split Out                       | Extracts and splits image data  | OPENAI GENERATES IMAGE    | Telegram, Facebook, LinkedIn, X (Twitter)     | # Finish - Upload to Platforms![Guide](https://i.ibb.co/d41JsL8q/Screenshot-2025-05-30-122423-1.jpg#full-width#full-width) |
| Telegram                 | Telegram Node                   | Posts image and text to Telegram| Split Out                 | None                                          |                                                                                                              |
| Facebook                 | Facebook Graph API              | Posts content to Facebook       | Split Out                 | None                                          |                                                                                                              |
| LinkedIn                 | LinkedIn Node                  | Posts content to LinkedIn       | Split Out                 | None                                          |                                                                                                              |
| X                        | Twitter Node                   | Posts content to Twitter/X      | Split Out                 | None                                          |                                                                                                              |
| OPENAI WRITES POSTS       | OpenAI Chat Model               | Alternative AI content generation| GENERATE TEXT (ai_languageModel) | GENERATE TEXT                                |                                                                                                              |
| OPENAI WRITES PROMPTS     | OpenAI Chat Model               | Alternative prompt writing      | GENERATE PROMPT (ai_languageModel) | GENERATE PROMPT                            |                                                                                                              |
| Mistral Cloud Chat Model  | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| OpenRouter Chat Model     | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| Anthropic Chat Model      | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| Google Gemini Chat Model  | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| xAI Grok Chat Model       | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| DeepSeek Chat Model       | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| Hugging Face Inference Model | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| Ollama Chat Model         | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| Azure OpenAI Chat Model   | Langchain Chat Model            | Optional AI model for text      | None                      | None                                          |                                                                                                              |
| Freepik API               | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Runware API               | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Clipdrop API              | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Ideogram API              | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Replicate API             | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Imagen Google API         | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Runway Images             | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Minimax Images            | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Kling Images              | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Leonardo Images           | HTTP Request                   | Alternative image generation    | None                      | None                                          |                                                                                                              |
| APITemplate.io            | APITemplate.io Node            | Alternative image generation    | None                      | None                                          |                                                                                                              |
| Sticky Note / Sticky Note2 - 15 | Sticky Note                  | Notes, branding, visual guides | None                      | None                                          | Various sticky notes with branding, instructions, visual examples, and encouragement to customize the workflow |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node: Set to trigger daily at 09:00.  
   - Add a **Manual Trigger** node for manual runs.  
   - Add a **Google Sheets Trigger** node: Configure with OAuth2 credentials, select target spreadsheet, watch for new rows added in the range A2:A10 in the "Links for articles to refer" column, polling every 1 minute.  

2. **Add Tavily Internet Search Node:**  
   - Add the Tavily AI Tool node.  
   - Configure API credentials for Tavily.  
   - Set query parameters to search the trigger’s article link or topic with advanced depth, 7-day time range, 1 result, and include answer only.  

3. **Add GENERATE TEXT Node (Langchain Agent):**  
   - Set as Langchain Agent node.  
   - Configure with OpenAI GPT-4.1 or preferred LLM credentials.  
   - Use a prompt instructing to conduct real-time research via Tavily, write short (max 1024 characters) professional Telegram posts for Auto Service content with hooks, educational tone, minimal emojis, source attribution, hashtags, and call to action.  
   - Link Tavily node output as AI tool input.  
   - Connect all triggers (Schedule, Manual, Google Sheets) as input to this node.  

4. **Add GENERATE PROMPT Node (Langchain Agent):**  
   - Another Langchain Agent node.  
   - Configure prompt to transform Telegram post text into an ultra-high-resolution, hyper-realistic image generation prompt with detailed instructions for 16K photographic quality and AI super-resolution upscaling.  
   - Connect GENERATE TEXT’s output as input.  

5. **Add OPENAI GENERATES IMAGE Node:**  
   - Use OpenAI Image generation node with GPT-image-1 model.  
   - Configure OpenAI API credentials.  
   - Set prompt input from GENERATE PROMPT node output.  
   - Output image as binary data.  

6. **Add Split Out Node:**  
   - Splits the generated image data field to prepare for posting.  
   - Connect OPENAI GENERATES IMAGE output to this node.  

7. **Add Social Media Posting Nodes:**  
   - **Telegram:** Configure with Telegram API credentials, set operation to sendPhoto, attach image from Split Out node binary data, and caption from GENERATE TEXT output.  
   - **Facebook:** Configure Facebook Graph API node with OAuth2 credentials to post text content.  
   - **LinkedIn:** Configure LinkedIn node with OAuth2 credentials and person ID to post content text.  
   - **X (Twitter):** Configure Twitter node with OAuth credentials to post content text.  
   - Connect Split Out node output to all these posting nodes in parallel.  

8. **Optional:** Add alternative AI chat model nodes (Mistral, Anthropic, etc.) or alternative image generation APIs (Freepik, Runware, Replicate, etc.) for customization or experimentation.  

9. **Add Sticky Notes:**  
   - Add sticky notes with visual instructions, branding, and references as per the original workflow to guide users.  

10. **Test Each Trigger:**  
    - Run manual trigger to validate end-to-end content generation and posting.  
    - Verify scheduled and Google Sheets triggers function correctly.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow template is designed for daily Auto Service content uploads but is universal; easily customizable for any niche.     | Sticky Note1 with image: https://i.ibb.co/qLxMHbd5/customize-ride1.jpg#full-width                            |
| Made with ❤️ by N8ner – Community profile and badges for support and feedback.                                                    | https://community.n8n.io/u/n8ner/badges                                                                     |
| Visual guides for workflow start and finish steps, including trigger selection and content upload to platforms.                  | https://i.ibb.co/d41JsL8q/Screenshot-2025-05-30-122423-1.jpg#full-width                                      |
| Example source image to inspire prompt and content style.                                                                          | https://i.ibb.co/PZF4szJr/photo-2025-05-30-13-24-04.jpg#full-width                                           |
| Customize AI model selection and prompt content to improve content quality and style.                                              | Sticky Note13, Sticky Note14, Sticky Note15 with graphics https://i.ibb.co/TxQrh405/erasebg-transformed-removebg-preview.png#full-width |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, adhering strictly to content policies without illegal or offensive material. All data handled is legal and public.