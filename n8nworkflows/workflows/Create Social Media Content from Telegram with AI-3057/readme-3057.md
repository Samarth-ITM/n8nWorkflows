Create Social Media Content from Telegram with AI

https://n8nworkflows.xyz/workflows/create-social-media-content-from-telegram-with-ai-3057


# Create Social Media Content from Telegram with AI

### 1. Workflow Overview

This workflow automates the creation of social media content and detailed image prompts from user requests sent via Telegram messages (voice or text). It leverages AI technologies including OpenAI’s Whisper for voice transcription, OpenAI Chat Models combined with SerpAPI for research and content generation, and optionally Hugging Face’s Stable Diffusion model for image generation.

**Target Use Cases:**  
- Social media managers and content creators seeking rapid, AI-assisted content and image prompt generation.  
- Marketing teams automating research and drafting of social media posts.  
- Users who prefer voice or text input via Telegram for content requests.

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving Telegram messages and determining message type (voice or text).  
- **1.2 Voice Processing:** Fetching and transcribing voice messages to text.  
- **1.3 Text Preparation:** Preparing the text input for AI processing.  
- **1.4 AI Content Generation:** Using an AI agent with OpenAI Chat Model and SerpAPI to research and generate social media content and image prompts.  
- **1.5 Optional Image Generation:** Using the generated image prompt to create an image via Hugging Face’s Stable Diffusion API.  
- **1.6 Final Output Preparation:** Extracting and formatting the final content and image data for output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming Telegram messages and routes them based on whether they contain voice or text content.

**Nodes Involved:**  
- Receive Telegram Messages  
- Voice or Text? (Switch)  
- Sticky Note2 (comment)

**Node Details:**  

- **Receive Telegram Messages**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (voice or text) from users.  
  - Configuration: Listens to "message" updates; uses Telegram bot credentials.  
  - Inputs: Telegram webhook trigger.  
  - Outputs: Message JSON containing voice or text data.  
  - Edge Cases: Telegram API downtime, invalid bot token, webhook misconfiguration.

- **Voice or Text?**  
  - Type: Switch  
  - Role: Determines if the incoming message is a voice message, text message, or error.  
  - Configuration:  
    - Checks if `message.voice.file_id` exists → routes to "Audio" output.  
    - Checks if `message.text` exists → routes to "Text" output.  
    - Otherwise routes to "Error".  
  - Inputs: Output from Telegram Trigger.  
  - Outputs: Branches to voice processing or text preparation.  
  - Edge Cases: Messages without voice or text, malformed JSON, unexpected message types.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Content: "This workflow listens for incoming voice or text messages from Telegram users."  
  - Role: Documentation aid.

---

#### 2.2 Voice Processing

**Overview:**  
Fetches the voice message file from Telegram and transcribes it into text using OpenAI’s Whisper API.

**Nodes Involved:**  
- Fetch Voice Message  
- Transcribe Voice to Text  
- Sticky Note3 (comment)

**Node Details:**  

- **Fetch Voice Message**  
  - Type: Telegram  
  - Role: Downloads the voice message file from Telegram servers using the voice file ID.  
  - Configuration: Uses `message.voice.file_id` from the incoming message JSON.  
  - Inputs: "Audio" output from Switch node.  
  - Outputs: Binary audio file data.  
  - Edge Cases: File not found, Telegram API rate limits, network errors.

- **Transcribe Voice to Text**  
  - Type: OpenAI (Whisper)  
  - Role: Translates/transcribes the audio file into text.  
  - Configuration: Uses OpenAI Whisper API with audio translation operation.  
  - Inputs: Binary audio file from Fetch Voice Message.  
  - Outputs: JSON with transcribed text.  
  - Edge Cases: Audio format unsupported, API errors, transcription inaccuracies.

- **Sticky Note3**  
  - Content: "Voice messages are fetched from Telegram and transcribed into text using OpenAI's Whisper API."  
  - Role: Documentation aid.

---

#### 2.3 Text Preparation

**Overview:**  
Prepares the text input (either transcribed voice or direct text) for the AI agent by setting it into a uniform variable.

**Nodes Involved:**  
- Prepare for LLM  
- Sticky Note (general AI agent comment)

**Node Details:**  

- **Prepare for LLM**  
  - Type: Set  
  - Role: Assigns the text content to a variable named `text` for downstream AI processing.  
  - Configuration: Sets `text` to `{{$json.message.text}}` for text messages or to transcribed text for voice messages (via connection).  
  - Inputs: "Text" output from Switch or output from Transcribe Voice to Text.  
  - Outputs: JSON with `text` property.  
  - Edge Cases: Empty or null text, expression evaluation errors.

- **Sticky Note (id:9a273f0b-bcb0-4ed8-93f5-6161d192e3ef)**  
  - Content: "The AI agent uses the OpenAI Chat Model and SerpAPI tool to conduct research and generate social media content and an image prompt based on the user request."  
  - Role: Documentation aid.

---

#### 2.4 AI Content Generation

**Overview:**  
An AI agent uses the input text to research the topic via SerpAPI, then generates engaging social media content and a detailed image prompt using OpenAI’s Chat Model.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- SerpAPI  
- Structured Output Parser  
- Sticky Note1 (AI agent explanation)

**Node Details:**  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates the AI workflow: research with SerpAPI, content and image prompt generation with OpenAI Chat Model.  
  - Configuration:  
    - Input text: `{{$json.text}}` from previous node.  
    - System message defines goals: research topic, create SEO-optimized social media content (800-1000 chars), generate detailed photorealistic image prompt.  
    - Uses SerpAPI tool for research phase.  
    - Uses OpenAI Chat Model (gpt-4o-mini) for content generation.  
    - Outputs JSON with `content` and `image_prompt` fields.  
  - Inputs: Text from Prepare for LLM node.  
  - Outputs: JSON with generated content and image prompt.  
  - Edge Cases: API rate limits, malformed input, incomplete research data, AI hallucinations.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides the language model for content and prompt generation.  
  - Configuration: Model set to "gpt-4o-mini".  
  - Inputs: From AI Agent as language model.  
  - Outputs: Text completions for AI Agent.  
  - Edge Cases: API errors, model unavailability.

- **SerpAPI**  
  - Type: LangChain SerpAPI Tool  
  - Role: Provides web search results for the AI agent to research the topic.  
  - Configuration: Uses SerpAPI credentials.  
  - Inputs: From AI Agent as research tool.  
  - Outputs: Search result summaries.  
  - Edge Cases: API quota exceeded, network errors.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI Agent output into structured JSON with `content` and `image_prompt`.  
  - Configuration: JSON schema example provided.  
  - Inputs: AI Agent output.  
  - Outputs: Parsed JSON.  
  - Edge Cases: Parsing errors if AI output deviates from schema.

- **Sticky Note1**  
  - Content: "The AI agent uses the OpenAI Chat Model and SerpAPI tool to conduct research and generate social media content and an image prompt based on the user request."  
  - Role: Documentation aid.

---

#### 2.5 Optional Image Generation

**Overview:**  
Uses the detailed image prompt generated by the AI agent to create a photorealistic image via Hugging Face’s Stable Diffusion API.

**Nodes Involved:**  
- Generate Image  
- Extract from File  
- Sticky Note1 (image generation comment)

**Node Details:**  

- **Generate Image**  
  - Type: HTTP Request  
  - Role: Sends the image prompt to Hugging Face’s Stable Diffusion API to generate an image.  
  - Configuration:  
    - POST request to `https://router.huggingface.co/hf-inference/models/stabilityai/stable-diffusion-3.5-large`.  
    - Body parameter `inputs` set to `{{$json.output.image_prompt}}`.  
    - Uses Hugging Face API credentials with header authentication.  
  - Inputs: AI Agent output containing `image_prompt`.  
  - Outputs: Binary image data or image URL in response.  
  - Edge Cases: API rate limits, invalid prompt, network errors.

- **Extract from File**  
  - Type: Extract from File  
  - Role: Converts binary image data to a usable property (e.g., base64 or URL).  
  - Configuration: Binary to property operation.  
  - Inputs: Binary image data from Generate Image.  
  - Outputs: JSON with image data property.  
  - Edge Cases: Binary data corruption, extraction failures.

- **Sticky Note1**  
  - Content: "An image is generated using the image prompt."  
  - Role: Documentation aid.

---

#### 2.6 Final Output Preparation

**Overview:**  
Prepares the final JSON output containing the generated social media content and the image data for downstream use or delivery.

**Nodes Involved:**  
- Prepare Final Output

**Node Details:**  

- **Prepare Final Output**  
  - Type: Set  
  - Role: Assigns the final output fields: `content` (social media text) and `image` (image data or URL).  
  - Configuration:  
    - `content` set from AI Agent’s JSON output `output.content`.  
    - `image` set from extracted image data property.  
  - Inputs: Output from Extract from File node.  
  - Outputs: Final JSON object with content and image.  
  - Edge Cases: Missing or malformed AI output, missing image data.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                         | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                  |
|-------------------------|----------------------------------|---------------------------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Receive Telegram Messages | Telegram Trigger                 | Listens for incoming Telegram messages                  | -                           | Voice or Text?             | This workflow listens for incoming voice or text messages from Telegram users.                              |
| Voice or Text?           | Switch                          | Routes messages based on voice or text content          | Receive Telegram Messages    | Fetch Voice Message, Prepare for LLM |                                                                                                              |
| Fetch Voice Message      | Telegram                        | Downloads voice message file from Telegram               | Voice or Text? (Audio)       | Transcribe Voice to Text    | Voice messages are fetched from Telegram and transcribed into text using OpenAI's Whisper API.             |
| Transcribe Voice to Text | OpenAI Whisper                  | Transcribes audio to text                                 | Fetch Voice Message          | AI Agent                   |                                                                                                              |
| Prepare for LLM          | Set                            | Prepares text input for AI processing                    | Voice or Text? (Text)        | AI Agent                   |                                                                                                              |
| AI Agent                 | LangChain Agent                | Conducts research and generates social media content and image prompt | Prepare for LLM, Transcribe Voice to Text | Generate Image             | The AI agent uses the OpenAI Chat Model and SerpAPI tool to conduct research and generate social media content and an image prompt based on the user request. |
| OpenAI Chat Model        | LangChain OpenAI Chat Model    | Provides language model for content generation           | AI Agent (ai_languageModel) | AI Agent                   |                                                                                                              |
| SerpAPI                  | LangChain SerpAPI Tool         | Provides web search results for AI research              | AI Agent (ai_tool)           | AI Agent                   |                                                                                                              |
| Structured Output Parser | LangChain Output Parser        | Parses AI output into structured JSON                     | AI Agent (ai_outputParser)   | AI Agent                   |                                                                                                              |
| Generate Image           | HTTP Request                   | Generates image from prompt using Hugging Face API       | AI Agent                    | Extract from File          | An image is generated using the image prompt.                                                               |
| Extract from File        | Extract from File              | Converts binary image data to usable property             | Generate Image               | Prepare Final Output       |                                                                                                              |
| Prepare Final Output     | Set                            | Prepares final JSON output with content and image data   | Extract from File            | -                          |                                                                                                              |
| Sticky Note2             | Sticky Note                   | Documentation                                            | -                           | -                          | This workflow listens for incoming voice or text messages from Telegram users.                              |
| Sticky Note3             | Sticky Note                   | Documentation                                            | -                           | -                          | Voice messages are fetched from Telegram and transcribed into text using OpenAI's Whisper API.             |
| Sticky Note              | Sticky Note                   | Documentation                                            | -                           | -                          | The AI agent uses the OpenAI Chat Model and SerpAPI tool to conduct research and generate social media content and an image prompt based on the user request. |
| Sticky Note1             | Sticky Note                   | Documentation                                            | -                           | -                          | An image is generated using the image prompt.                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials.  
   - Set to listen for "message" updates.  
   - Position: Start of workflow.

2. **Add Switch Node "Voice or Text?"**  
   - Type: Switch  
   - Configure three outputs:  
     - "Audio" if `message.voice.file_id` exists.  
     - "Text" if `message.text` exists.  
     - "Error" for others.  
   - Connect Telegram Trigger output to this node.

3. **Add Telegram Node "Fetch Voice Message"**  
   - Type: Telegram  
   - Configure to fetch file using `message.voice.file_id`.  
   - Use Telegram credentials.  
   - Connect "Audio" output of Switch node here.

4. **Add OpenAI Node "Transcribe Voice to Text"**  
   - Type: OpenAI (Whisper)  
   - Set resource to "audio" and operation to "translate".  
   - Use OpenAI API credentials.  
   - Connect output of Fetch Voice Message node here.

5. **Add Set Node "Prepare for LLM"**  
   - Type: Set  
   - Assign variable `text` with expression:  
     - For text messages: `{{$json.message.text}}`  
     - For voice messages: connect from Transcribe Voice to Text node output, assign `text` from transcribed text.  
   - Connect "Text" output of Switch node and Transcribe Voice to Text node output to this node.

6. **Add LangChain Agent Node "AI Agent"**  
   - Type: LangChain Agent  
   - Input text: `{{$json.text}}`  
   - Configure system prompt with detailed instructions for:  
     - Research using SerpAPI.  
     - Generate SEO-optimized social media content (800-1000 chars).  
     - Generate detailed photorealistic image prompt.  
     - Output JSON with `content` and `image_prompt`.  
   - Connect Prepare for LLM node output here.

7. **Add LangChain OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: "gpt-4o-mini" or preferred GPT-4 variant.  
   - Use OpenAI API credentials.  
   - Connect as language model input to AI Agent node.

8. **Add LangChain SerpAPI Tool Node**  
   - Type: LangChain SerpAPI Tool  
   - Use SerpAPI credentials.  
   - Connect as AI tool input to AI Agent node.

9. **Add LangChain Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Provide JSON schema example for output:  
     ```json
     {
       "content": "[SOCIAL_MEDIA_CONTENT]",
       "image_prompt": "[IMAGE_PROMPT]"
     }
     ```  
   - Connect as output parser input to AI Agent node.

10. **Add HTTP Request Node "Generate Image" (Optional)**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://router.huggingface.co/hf-inference/models/stabilityai/stable-diffusion-3.5-large`  
    - Body parameter `inputs`: set to `{{$json.output.image_prompt}}`  
    - Use Hugging Face API credentials with header authentication.  
    - Connect AI Agent output here.

11. **Add Extract from File Node**  
    - Type: Extract from File  
    - Operation: Binary to property (convert image binary to usable property).  
    - Connect output of Generate Image node here.

12. **Add Set Node "Prepare Final Output"**  
    - Type: Set  
    - Assign:  
      - `content` = `{{$node["AI Agent"].json["output"]["content"]}}`  
      - `image` = `{{$json.data}}` (from Extract from File)  
    - Connect output of Extract from File node here.

13. **Add Sticky Notes** (Optional for documentation)  
    - Add notes describing each block’s purpose for clarity.

14. **Test the Workflow**  
    - Send a voice or text message to your Telegram bot.  
    - Verify transcription, AI content generation, and optional image generation.  
    - Adjust prompts and parameters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow combines Telegram messaging with AI-powered content and image prompt generation for social media. | Workflow description and use cases provided in the initial documentation.                        |
| OpenAI Whisper API is used for voice transcription.                                                            | Requires OpenAI API key with Whisper access.                                                    |
| SerpAPI is used for web research to ensure factual accuracy and topical relevance.                             | Requires SerpAPI account and API key.                                                           |
| Hugging Face Stable Diffusion API is used for optional image generation.                                        | Requires Hugging Face API key and proper authentication setup.                                  |
| The AI agent prompt emphasizes photorealistic, detailed image prompts suitable for DALL-E or Stable Diffusion. | See AI Agent system message for detailed prompt instructions.                                   |
| For more information on n8n Telegram integration, see: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.telegramTrigger/ | Official n8n Telegram Trigger node documentation.                                               |
| For OpenAI API usage and Whisper, see: https://platform.openai.com/docs/api-reference/audio/create | Official OpenAI API documentation for audio transcription.                                      |
| For SerpAPI details, see: https://serpapi.com/                                                                 | Official SerpAPI documentation.                                                                 |
| For Hugging Face inference API, see: https://huggingface.co/docs/api-inference/index                         | Official Hugging Face API documentation.                                                        |

---

This structured documentation provides a comprehensive understanding of the workflow’s design, node configurations, and operational logic, enabling advanced users and AI agents to reproduce, modify, or extend the workflow confidently.