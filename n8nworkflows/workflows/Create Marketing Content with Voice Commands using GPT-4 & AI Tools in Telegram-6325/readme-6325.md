Create Marketing Content with Voice Commands using GPT-4 & AI Tools in Telegram

https://n8nworkflows.xyz/workflows/create-marketing-content-with-voice-commands-using-gpt-4---ai-tools-in-telegram-6325


# Create Marketing Content with Voice Commands using GPT-4 & AI Tools in Telegram

### 1. Workflow Overview

This workflow, titled **"Create Marketing Content with Voice Commands using GPT-4 & AI Tools in Telegram"**, is designed to enable users to generate and edit marketing content through Telegram messages, using advanced AI capabilities. It supports both voice and text inputs from users and leverages GPT-4 and specialized sub-workflows to create diverse marketing assets such as blog posts, LinkedIn posts, images, edited images, videos, and image database searches.

The workflow is logically organized into the following functional blocks:

- **1.1 Input Reception & Preprocessing:** Captures user messages from Telegram, distinguishes between voice and text inputs, and transcribes voice messages.
- **1.2 Conversational Memory & AI Brain:** Maintains context with session memory and uses GPT-4 along with an AI agent to interpret user commands and decide which content creation tool to invoke.
- **1.3 Content Creation Tools:** Calls specific sub-workflows for generating or editing images, blog posts, LinkedIn posts, videos, or searching an image database.
- **1.4 Response Delivery:** Sends the generated content or responses back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

**Overview:**  
This block handles incoming Telegram messages, distinguishes whether the input is voice or text, downloads voice files if needed, and transcribes voice audio into text.

**Nodes Involved:**  
- Telegram Trigger  
- Switch  
- Download Voice File  
- Transcribe Audio  
- Set 'Text'  

**Node Details:**  

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point listening for new Telegram messages ("message" updates).  
  - *Config:* Uses Telegram API credentials; webhook set up to receive messages.  
  - *Inputs:* External Telegram messages.  
  - *Outputs:* Passes message JSON to Switch node.  
  - *Failures:* Network/webhook issues; permission or credential errors in Telegram API.

- **Switch**  
  - *Type:* Switch node  
  - *Role:* Determines if the incoming message contains a voice message or plain text.  
  - *Config:* Uses expression checks on message JSON for presence of `voice.file_id` or `text`.  
  - *Inputs:* Telegram Trigger output.  
  - *Outputs:* Routes to either "Voice" path (Download Voice File) or "Text" path (Set 'Text').  
  - *Failures:* Expression failure if message JSON structure changes or missing fields.

- **Download Voice File**  
  - *Type:* Telegram node  
  - *Role:* Downloads the voice message file from Telegram servers using `file_id`.  
  - *Config:* Uses Telegram API credentials, accesses `voice.file_id` dynamically.  
  - *Inputs:* Voice path from Switch node.  
  - *Outputs:* Binary audio data to Transcribe Audio.  
  - *Failures:* Telegram API errors, file not found, download timeout.

- **Transcribe Audio**  
  - *Type:* LangChain OpenAI Node (Audio Transcription)  
  - *Role:* Transcribes downloaded voice audio to text using OpenAI API.  
  - *Config:* OpenAI API credentials; operation set to "transcribe".  
  - *Inputs:* Audio data from Download Voice File.  
  - *Outputs:* Text transcription for further processing.  
  - *Failures:* API rate limits, transcription errors, unsupported audio formats.

- **Set 'Text'**  
  - *Type:* Set node  
  - *Role:* Extracts text message from Telegram JSON for downstream processing.  
  - *Config:* Sets variable `text` to the message's text content.  
  - *Inputs:* Text path from Switch node.  
  - *Outputs:* Text assigned for AI processing.  
  - *Failures:* Missing or malformed text field in the message.

---

#### 2.2 Conversational Memory & AI Brain

**Overview:**  
This block manages session memory to maintain context per user chat and uses GPT-4 and a custom AI marketing agent to interpret user commands, decide which tool to invoke, and formulate responses.

**Nodes Involved:**  
- Simple Memory  
- GPT 4.1  
- Marketing Team Agent  
- Think  

**Node Details:**  

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Stores conversational context per Telegram chat using chat ID as session key.  
  - *Config:* Session key dynamically set via Telegram chat ID.  
  - *Inputs:* Incoming text or transcribed text.  
  - *Outputs:* Provides context to the Marketing Team Agent.  
  - *Failures:* Memory overflow or limits, key mismatch.

- **GPT 4.1**  
  - *Type:* LangChain OpenRouter Chat Model  
  - *Role:* Provides advanced language understanding and generation capabilities using GPT-4.1 model.  
  - *Config:* Uses OpenRouter API credentials; no additional options set.  
  - *Inputs:* Text and context from Simple Memory or Marketing Team Agent.  
  - *Outputs:* Language model responses for agent decisions.  
  - *Failures:* API errors, rate limits, model unavailability.

- **Marketing Team Agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Core AI agent that receives user input text, maintains system instructions, and delegates tasks to specialized sub-workflows (tools) based on user intent.  
  - *Config:*  
    - System message defines the agent’s role as a marketing team assistant.  
    - Lists tools: createImage, editImage, Image Database, blogPost, linkedinPost, video, and Think.  
    - Provides instructions for interpreting commands like "edit that image" and output formatting.  
  - *Inputs:* Text from Set 'Text' or Transcribe Audio and memory context.  
  - *Outputs:* Calls appropriate sub-workflows as AI tools and sends final response text to Telegram node.  
  - *Failures:* Misinterpretation of user intent, tool invocation errors, API limitations.

- **Think**  
  - *Type:* LangChain Tool Think Node  
  - *Role:* Helper tool invoked by the agent for internal decision-making or complex reasoning steps.  
  - *Config:* No parameters; used internally by the Marketing Team Agent.  
  - *Inputs:* Called by the agent when needed.  
  - *Outputs:* Returns reasoning to the agent.  
  - *Failures:* AI reasoning errors or timeouts.

---

#### 2.3 Content Creation Tools

**Overview:**  
This block contains calls to multiple sub-workflows specialized in creating or editing marketing content in various formats. The Marketing Team Agent invokes these tools based on user requests.

**Nodes Involved:**  
- Create Image  
- Edit Image  
- Search Images  
- Blog Post  
- LinkedIn Post  
- Video  

**Node Details:**  

- **Create Image**  
  - *Type:* LangChain Tool Workflow (Sub-workflow call)  
  - *Role:* Generates images based on an image title and prompt provided by the user.  
  - *Config:* Passes chatID, imageTitle, and imagePrompt dynamically from AI variables.  
  - *Inputs:* Called by Marketing Team Agent.  
  - *Outputs:* Returns image link and metadata to agent.  
  - *Failures:* Sub-workflow errors, image generation API limits.

- **Edit Image**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Edits an existing image identified by pictureID based on user’s edit request.  
  - *Config:* Inputs include image, chatID, request (edit description), and pictureID.  
  - *Inputs:* Called by Marketing Team Agent.  
  - *Outputs:* Edited image data to agent.  
  - *Failures:* Invalid pictureID, edit request interpretation errors.

- **Search Images**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Searches the image database for images matching a title and intent ("Get" or "Edit").  
  - *Config:* Inputs include image title, chatID, and intent.  
  - *Inputs:* Called by Marketing Team Agent.  
  - *Outputs:* Search results sent back to agent.  
  - *Failures:* Database access errors, ambiguous queries.

- **Blog Post**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Creates a blog post given a topic and target audience.  
  - *Config:* Inputs are blogTopic, targetAudience, and chatID.  
  - *Inputs:* Called by Marketing Team Agent.  
  - *Outputs:* Blog post content to agent.  
  - *Failures:* Content generation errors, topic ambiguity.

- **LinkedIn Post**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Creates a LinkedIn post based on a topic and target audience.  
  - *Config:* Inputs are postTopic, targetAudience, and chatID.  
  - *Inputs:* Called by Marketing Team Agent.  
  - *Outputs:* LinkedIn post content to agent.  
  - *Failures:* Similar to Blog Post.

- **Video**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Creates faceless videos on a given topic.  
  - *Config:* Inputs are videoTopic and chatID.  
  - *Inputs:* Called by Marketing Team Agent.  
  - *Outputs:* Video link or metadata to agent.  
  - *Failures:* Video generation API failures or delays.

---

#### 2.4 Response Delivery

**Overview:**  
This block sends the final generated content or response text back to the user on Telegram.

**Nodes Involved:**  
- Telegram  

**Node Details:**  

- **Telegram**  
  - *Type:* Telegram node  
  - *Role:* Sends messages to the user chat on Telegram.  
  - *Config:*  
    - Uses `chatId` dynamically from the Telegram Trigger node.  
    - Sends the AI agent’s output text as the message.  
    - Attribution disabled for clean message output.  
  - *Inputs:* Output from Marketing Team Agent.  
  - *Outputs:* Message delivered to Telegram user.  
  - *Failures:* Telegram API errors, network issues, chat restrictions.

---

### 3. Summary Table

| Node Name           | Node Type                             | Functional Role                      | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                   |
|---------------------|-------------------------------------|------------------------------------|------------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger                    | Entry point for Telegram messages  | —                      | Switch                | # Trigger                                                                                   |
| Switch              | Switch                             | Routes voice or text input          | Telegram Trigger        | Download Voice File, Set 'Text' | # Voice or Text                                                                            |
| Download Voice File  | Telegram node                      | Downloads voice audio file          | Switch (Voice path)     | Transcribe Audio       | # Voice or Text                                                                            |
| Transcribe Audio     | OpenAI audio transcription node   | Converts voice to text              | Download Voice File     | Marketing Team Agent   | # Voice or Text                                                                            |
| Set 'Text'           | Set node                          | Extracts text from Telegram message | Switch (Text path)      | Marketing Team Agent   | # Voice or Text                                                                            |
| Simple Memory        | LangChain Memory Buffer Window    | Stores conversational context      | Marketing Team Agent    | Marketing Team Agent   | # Brain                                                                                     |
| GPT 4.1              | LangChain Chat Model              | Provides GPT-4 language capabilities | Marketing Team Agent    | Marketing Team Agent   | # Brain                                                                                     |
| Marketing Team Agent | LangChain Agent                   | Core AI agent deciding tasks       | Set 'Text', Transcribe Audio, Simple Memory, GPT 4.1 | Telegram, Various Tool Workflows | # Marketing Team                                                                        |
| Think                | LangChain Tool Think              | Internal reasoning helper           | Marketing Team Agent    | Marketing Team Agent   | # Brain                                                                                     |
| Create Image         | LangChain Tool Workflow           | Creates marketing images            | Marketing Team Agent    | Marketing Team Agent   | # Image Creation                                                                           |
| Edit Image           | LangChain Tool Workflow           | Edits existing images               | Marketing Team Agent    | Marketing Team Agent   | # Image Creation                                                                           |
| Search Images        | LangChain Tool Workflow           | Searches image database             | Marketing Team Agent    | Marketing Team Agent   | # Image Database                                                                           |
| Blog Post            | LangChain Tool Workflow           | Creates blog posts                  | Marketing Team Agent    | Marketing Team Agent   | # Content Creation                                                                         |
| LinkedIn Post        | LangChain Tool Workflow           | Creates LinkedIn posts              | Marketing Team Agent    | Marketing Team Agent   | # Content Creation                                                                         |
| Video                | LangChain Tool Workflow           | Creates faceless videos             | Marketing Team Agent    | Marketing Team Agent   | # Content Creation                                                                         |
| Telegram             | Telegram node                    | Sends final responses to users     | Marketing Team Agent    | —                     | # Response                                                                                |
| Sticky Note          | Sticky Note                      | Visual annotation                  | —                      | —                     | # Marketing Team                                                                          |
| Sticky Note1         | Sticky Note                      | Visual annotation                  | —                      | —                     | # Trigger                                                                                 |
| Sticky Note2         | Sticky Note                      | Visual annotation                  | —                      | —                     | # Content Creation                                                                       |
| Sticky Note3         | Sticky Note                      | Visual annotation                  | —                      | —                     | # Image Creation                                                                         |
| Sticky Note4         | Sticky Note                      | Visual annotation                  | —                      | —                     | # Image Database                                                                         |
| Sticky Note5         | Sticky Note                      | Visual annotation                  | —                      | —                     | # Response                                                                              |
| Sticky Note6         | Sticky Note                      | Visual annotation                  | —                      | —                     | # Brain                                                                                |
| Sticky Note7         | Sticky Note                      | Visual annotation                  | —                      | —                     | # Voice or Text                                                                        |
| Sticky Note8         | Sticky Note                      | Setup guide and instructions       | —                      | —                     | # 📋 Setup Guide  Author: [Muhammad Ashar Ishfaq](https://www.linkedin.com/in/muhammad-ashar-ishfaq/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook to listen for "message" updates.  
   - Set Telegram API credentials.  

2. **Add Switch Node**  
   - Type: Switch  
   - Set two rules:  
     - Check if `message.voice.file_id` exists → output "Voice"  
     - Check if `message.text` exists → output "Text"  
   - Connect Telegram Trigger output to Switch input.

3. **Add Download Voice File Node** (for Voice path)  
   - Type: Telegram node  
   - Configure to download file using `{{$json.message.voice.file_id}}`.  
   - Use Telegram API credentials.  
   - Connect Switch "Voice" output to this node.

4. **Add Transcribe Audio Node**  
   - Type: LangChain OpenAI node (resource: audio, operation: transcribe)  
   - Configure with OpenAI API credentials.  
   - Connect Download Voice File output to this node.

5. **Add Set 'Text' Node** (for Text path)  
   - Type: Set node  
   - Create assignment: `text = {{$json.message.text}}`  
   - Connect Switch "Text" output to this node.

6. **Add Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Set `sessionKey` to `{{$json.message.chat.id}}` (from Telegram Trigger).  
   - Connect outputs from Set 'Text' and Transcribe Audio nodes to this node, or integrate memory with Marketing Team Agent as per design.

7. **Add GPT 4.1 Node**  
   - Type: LangChain Chat Model (OpenRouter)  
   - Select model `openai/gpt-4.1`.  
   - Setup OpenRouter API credentials.  
   - Connect appropriate inputs for language model usage in the agent.

8. **Add Marketing Team Agent Node**  
   - Type: LangChain Agent  
   - Set input text to `{{$json.text}}` (from Set 'Text' or transcribed text).  
   - Provide system message defining the marketing assistant role, tool list (createImage, editImage, Image Database, blogPost, linkedinPost, video, Think), and instructions.  
   - Connect Simple Memory node as AI memory input and GPT 4.1 as language model.  
   - Connect outputs to various tool workflows and to Telegram node.

9. **Create Sub-workflows for Tools:**  
   - **Create Image:** Inputs `chatID`, `imageTitle`, `imagePrompt`. Outputs image link.  
   - **Edit Image:** Inputs `chatID`, `image`, `request`, `pictureID`. Outputs edited image.  
   - **Search Images:** Inputs `chatID`, `image`, `intent`. Outputs search results.  
   - **Blog Post:** Inputs `chatID`, `blogTopic`, `targetAudience`. Outputs blog text.  
   - **LinkedIn Post:** Inputs `chatID`, `postTopic`, `targetAudience`. Outputs LinkedIn post text.  
   - **Video:** Inputs `chatID`, `videoTopic`. Outputs video link or data.  
   - Implement these workflows with appropriate API integrations and n8n nodes.

10. **Add Think Node**  
    - Type: LangChain Tool Think  
    - Leave default config for AI reasoning helper.  
    - Connect to Marketing Team Agent as auxiliary tool.

11. **Add Telegram Node for Response**  
    - Type: Telegram node  
    - Configure to send messages with `text` from Marketing Team Agent output.  
    - Use Telegram API credentials.  
    - Connect Marketing Team Agent output to this node.

12. **Add Sticky Notes for Clarity (Optional)**  
    - Add labeled sticky notes near logical blocks for documentation and visual clarity.

13. **Configure Credentials:**  
    - Telegram API credential for inbound/outbound messaging.  
    - OpenRouter API for GPT-4.1 model access.  
    - OpenAI API for audio transcription.  
    - API keys for sub-workflows (PiAPI, Runway, ElevenLabs for video; Tavily for blog and LinkedIn posts).  
    - Ensure all sub-workflows have correct API keys and are imported and linked correctly.

14. **Activate and Test Workflow**  
    - Test with text commands in Telegram.  
    - Test voice commands by sending voice messages.  
    - Verify content creation and editing features respond correctly.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Author and Setup Guide by Muhammad Ashar Ishfaq with detailed instructions on importing sub-workflows, linking tools, and configuring credentials and APIs. | [LinkedIn Profile](https://www.linkedin.com/in/muhammad-ashar-ishfaq/) |
| Important credentials include Telegram, OpenRouter (for GPT-4), Tavily (for research in blog/LinkedIn posts), PiAPI, Runway, ElevenLabs (for video generation). | See Sticky Note8 content in workflow. |
| Creatomate image templates and Google Sheets log templates are recommended for image creation and content tracking. | Creatomate: https://creatomate.com/ ; Sheets Template: https://docs.google.com/spreadsheets/d/1wQxM9cAwewCigPH22KDidMu_i9j_dx4MHEa5rmJiw5I/edit?usp=sharing |
| System instructions in the Marketing Team Agent ensure user commands like “edit that image” refer to the most recent image, and outputs include clickable image links. | Part of Marketing Team Agent system message. |

---

**Disclaimer:**  
The content above is derived exclusively from an n8n workflow automation. It adheres strictly to content policies and contains no illegal or offensive material. All data handled is legal and public.