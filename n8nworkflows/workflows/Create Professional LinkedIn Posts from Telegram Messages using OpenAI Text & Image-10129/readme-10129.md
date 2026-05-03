Create Professional LinkedIn Posts from Telegram Messages using OpenAI Text & Image

https://n8nworkflows.xyz/workflows/create-professional-linkedin-posts-from-telegram-messages-using-openai-text---image-10129


# Create Professional LinkedIn Posts from Telegram Messages using OpenAI Text & Image

---

### 1. Workflow Overview

This n8n workflow automates the creation of professional LinkedIn posts based on Telegram messages, supporting both voice and text inputs. It is designed for content creators or social media managers who want to streamline the process of generating engaging LinkedIn posts enhanced with AI-generated images. The workflow integrates Telegram for message intake, OpenAI for transcription, text formatting, and image generation, and LinkedIn for publishing.

Logical blocks:

- **1.1 Input Reception:** Captures Telegram messages (voice or text) and routes them accordingly.
- **1.2 Voice Message Processing:** Downloads voice messages and transcribes them to text.
- **1.3 Text Preparation:** Extracts or sets text for AI processing.
- **1.4 LinkedIn Post Text Generation:** Uses OpenAI to format the message into a professional LinkedIn post.
- **1.5 Image Prompt Generation:** Creates an AI image prompt based on the LinkedIn post content.
- **1.6 AI Image Creation:** Generates an AI image corresponding to the prompt.
- **1.7 LinkedIn Posting:** Publishes the post text and image to LinkedIn.
- **1.8 Supporting Documentation:** Sticky notes for setup, customization, and usage instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming Telegram messages and differentiates them between voice and text messages to route them appropriately.

**Nodes Involved:**  
- Telegram Trigger  
- Switch

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger node  
  - *Role:* Listens to Telegram Bot messages (voice or text) triggered by the bot user.  
  - *Configuration:* Watches for "message" updates. Requires Telegram API credentials with the bot token.  
  - *Input/Output:* No input; outputs Telegram message JSON.  
  - *Edge Cases:* Unauthorized bot token, Telegram downtime, malformed message data.

- **Switch**  
  - *Type:* Switch node  
  - *Role:* Routes messages based on content type: voice messages with `voice.file_id` or text messages with `text`.  
  - *Configuration:* Two conditions, one checking existence of `message.voice.file_id` (voice), another for `message.text` (text).  
  - *Input/Output:* Input from Telegram Trigger; outputs to either "Voice" or "Text" branches.  
  - *Edge Cases:* Messages with neither voice nor text (unsupported message types), missing fields.

#### 1.2 Voice Message Processing

**Overview:**  
Handles voice messages by downloading the audio file and transcribing it to text via OpenAI.

**Nodes Involved:**  
- Download File  
- Transcribe Audio

**Node Details:**

- **Download File**  
  - *Type:* Telegram node  
  - *Role:* Downloads the voice message file from Telegram servers using the file ID.  
  - *Configuration:* Uses `message.voice.file_id` dynamically to fetch audio. Requires Telegram API credentials.  
  - *Input/Output:* Input from Switch (Voice output); outputs audio file data.  
  - *Edge Cases:* File not found, Telegram API errors, network timeouts.

- **Transcribe Audio**  
  - *Type:* OpenAI node (Langchain integration)  
  - *Role:* Transcribes the downloaded audio file into text.  
  - *Configuration:* Resource set to "audio", operation "transcribe". Uses OpenAI API credentials.  
  - *Input/Output:* Input audio file from Download File; outputs transcribed text.  
  - *Edge Cases:* Audio quality poor, API rate limits, transcription errors.

#### 1.3 Text Preparation

**Overview:**  
Extracts text from Telegram text messages directly and prepares it for AI processing.

**Nodes Involved:**  
- Set 'Text'

**Node Details:**

- **Set 'Text'**  
  - *Type:* Set node  
  - *Role:* Extracts the raw text from the Telegram message JSON and assigns it to a `text` field.  
  - *Configuration:* Expression `={{ $json.message.text }}` used to map Telegram message text.  
  - *Input/Output:* Input from Switch (Text output); outputs JSON with `text` property.  
  - *Edge Cases:* Empty or malformed text messages.

#### 1.4 LinkedIn Post Text Generation

**Overview:**  
Transforms the raw text (either transcribed or direct) into a polished LinkedIn post using OpenAI with detailed formatting and tone requirements.

**Nodes Involved:**  
- LinkedIn Post Text

**Node Details:**

- **LinkedIn Post Text**  
  - *Type:* OpenAI node (Langchain integration)  
  - *Role:* Formats input text into a professional LinkedIn post following strict stylistic and structural rules (hook, paragraphs, tone).  
  - *Configuration:* Uses GPT-4.1-mini model; prompt includes detailed instructions on tone, length, formatting, and content constraints; input text referenced by `{{ $json.text }}`.  
  - *Input/Output:* Input text from Set 'Text' or Transcribe Audio; outputs formatted post content in `message.content`.  
  - *Edge Cases:* OpenAI API errors, prompt misinterpretation, input text too short or empty.

#### 1.5 Image Prompt Generation

**Overview:**  
Creates a detailed and professional image description prompt based on the generated LinkedIn post content to guide AI image generation.

**Nodes Involved:**  
- Image Prompt

**Node Details:**

- **Image Prompt**  
  - *Type:* OpenAI node (Langchain integration)  
  - *Role:* Analyzes LinkedIn post text and generates a professional, business-appropriate image prompt with style and content specifications.  
  - *Configuration:* Uses GPT-4.1-mini model; prompt instructs to produce a clean, minimalist, metaphorical description without text or faces; input text referenced by `{{ $json.message.content }}`.  
  - *Input/Output:* Input from LinkedIn Post Text; outputs image prompt text.  
  - *Edge Cases:* Ambiguous post content, API errors.

#### 1.6 AI Image Creation

**Overview:**  
Generates a visual image based on the prompt created, suitable for LinkedIn posts.

**Nodes Involved:**  
- Create Image

**Node Details:**

- **Create Image**  
  - *Type:* OpenAI node (Langchain integration)  
  - *Role:* Produces an AI-generated image using the prompt provided.  
  - *Configuration:* Model set to "gpt-image-1"; prompt dynamically set from previous node; uses OpenAI credentials.  
  - *Input/Output:* Input from Image Prompt; outputs image data or URL.  
  - *Edge Cases:* Model availability, API rate limits, image generation failures.

#### 1.7 LinkedIn Posting

**Overview:**  
Publishes the final LinkedIn post with the generated text and image using LinkedIn OAuth authentication.

**Nodes Involved:**  
- Create a post

**Node Details:**

- **Create a post**  
  - *Type:* LinkedIn node  
  - *Role:* Posts the text and AI-generated image on LinkedIn under the authenticated user.  
  - *Configuration:* Text parameter set from `LinkedIn Post Text` node output; specifies share media category as IMAGE; requires LinkedIn OAuth2 credentials.  
  - *Input/Output:* Input from Create Image node; outputs post metadata.  
  - *Edge Cases:* OAuth token expiration, API limits, post rejection by LinkedIn.

#### 1.8 Supporting Documentation

**Overview:**  
Sticky notes provide users with detailed setup instructions, customization tips, and node flow explanations.

**Nodes Involved:**  
- Sticky Note8  
- Sticky Note4  
- Sticky Note1  
- Sticky Note

**Node Details:**

- **Sticky Note8**  
  - *Content:* Setup and customization instructions, credential requirements, usage overview, and extended integration ideas including multi-platform posting.  
  - *Context:* Helps users with initial setup and advanced customization.

- **Sticky Note4**  
  - *Content:* Detailed instructions on authenticating LinkedIn node and publishing posts.  
  - *Context:* Critical for LinkedIn OAuth setup.

- **Sticky Note1**  
  - *Content:* Explanation of message input handling and transcription flow.  
  - *Context:* Clarifies input block.

- **Sticky Note**  
  - *Content:* Describes AI content generation logic for post text and image prompt creation.  
  - *Context:* Clarifies AI processing block.

---

### 3. Summary Table

| Node Name          | Node Type                         | Functional Role                            | Input Node(s)    | Output Node(s)     | Sticky Note                                                                                 |
|--------------------|----------------------------------|-------------------------------------------|------------------|--------------------|---------------------------------------------------------------------------------------------|
| Telegram Trigger    | Telegram Trigger                 | Listens for Telegram messages (voice/text) | None             | Switch             | See Sticky Note1 for Input Reception details                                                |
| Switch             | Switch                          | Routes messages to voice or text paths     | Telegram Trigger | Download File (Voice), Set 'Text' (Text) | See Sticky Note1 for Input Reception details                                                |
| Download File      | Telegram                        | Downloads voice audio file                   | Switch (Voice)   | Transcribe Audio   | See Sticky Note1 for Input Reception details                                                |
| Transcribe Audio   | OpenAI (Langchain)              | Transcribes voice audio to text              | Download File    | LinkedIn Post Text | See Sticky Note1 for Input Reception details                                                |
| Set 'Text'         | Set                            | Extracts text from Telegram text message    | Switch (Text)    | LinkedIn Post Text | See Sticky Note1 for Input Reception details                                                |
| LinkedIn Post Text | OpenAI (Langchain)              | Formats raw text into professional LinkedIn post | Set 'Text', Transcribe Audio | Image Prompt        | See Sticky Note for AI Content Generation                                                   |
| Image Prompt       | OpenAI (Langchain)              | Generates AI image prompt based on post text | LinkedIn Post Text | Create Image       | See Sticky Note for AI Content Generation                                                   |
| Create Image       | OpenAI (Langchain)              | Creates AI image from prompt                  | Image Prompt     | Create a post      |                                                                                             |
| Create a post      | LinkedIn                       | Publishes post with text and image on LinkedIn | Create Image     | None               | See Sticky Note4 for LinkedIn OAuth setup and posting details                               |
| Sticky Note8       | Sticky Note                    | Setup and customization instructions         | None             | None               | Setup and customization overview with links                                                |
| Sticky Note4       | Sticky Note                    | LinkedIn OAuth and posting instructions       | None             | None               | LinkedIn OAuth and publishing notes                                                        |
| Sticky Note1       | Sticky Note                    | Input reception explanation                    | None             | None               | Input reception notes covering Telegram Trigger, Switch, Download File, Transcribe Audio   |
| Sticky Note        | Sticky Note                    | AI content generation explanation               | None             | None               | AI post text and image prompt generation overview                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**
   - Type: Telegram Trigger
   - Parameters: Listen for "message" updates
   - Credentials: Add Telegram API credentials (bot token)
   - Position: Left top area for input

2. **Add Switch node:**
   - Type: Switch
   - Purpose: Route messages by type
   - Rules:
     - Output "Voice" if `message.voice.file_id` exists
     - Output "Text" if `message.text` exists
   - Connect Telegram Trigger → Switch

3. **Voice message path:**

   3.1 **Add Download File node:**
   - Type: Telegram
   - Parameters: `fileId` set to `={{ $json.message.voice.file_id }}`
   - Credentials: Telegram API credentials
   - Connect Switch (Voice output) → Download File

   3.2 **Add Transcribe Audio node:**
   - Type: OpenAI (Langchain)
   - Parameters: Resource = "audio", Operation = "transcribe"
   - Credentials: OpenAI API key
   - Connect Download File → Transcribe Audio

   3.3 **Connect Transcribe Audio → LinkedIn Post Text node (see below)**

4. **Text message path:**

   4.1 **Add Set node named "Set 'Text'":**
   - Type: Set
   - Parameters: Assign `text` field with expression `={{ $json.message.text }}`
   - Connect Switch (Text output) → Set 'Text'

   4.2 **Connect Set 'Text' → LinkedIn Post Text node**

5. **Add LinkedIn Post Text node:**
   - Type: OpenAI (Langchain)
   - Parameters:
     - Model ID: GPT-4.1-mini
     - Messages: Prompt instructing to transform input text (`{{ $json.text }}`) into a professional LinkedIn post with specified structure and tone (hook, paragraphs, length, no hashtags, max 2 emojis, etc.)
   - Credentials: OpenAI API key

6. **Add Image Prompt node:**
   - Type: OpenAI (Langchain)
   - Parameters:
     - Model ID: GPT-4.1-mini
     - Messages: Prompt to generate an AI image prompt from the LinkedIn post text (`{{ $json.message.content }}`), specifying style, content, and technical specs suitable for LinkedIn.
   - Credentials: OpenAI API key
   - Connect LinkedIn Post Text → Image Prompt

7. **Add Create Image node:**
   - Type: OpenAI (Langchain)
   - Parameters:
     - Model: gpt-image-1
     - Prompt: `={{ $json.message.content }}` (from Image Prompt node output)
   - Credentials: OpenAI API key
   - Connect Image Prompt → Create Image

8. **Add Create a post node:**
   - Type: LinkedIn
   - Parameters:
     - Text: `={{ $('LinkedIn Post Text').item.json.message.content }}`
     - Share media category: IMAGE
   - Credentials: LinkedIn OAuth2 credentials (ensure OAuth authentication is completed)
   - Connect Create Image → Create a post

9. **Add Sticky Notes (optional but recommended):**
   - Add notes with setup instructions, customization tips, and explanations as per Sticky Note contents.

10. **Activate the workflow:**
    - Ensure all credentials are valid.
    - Test by sending text or voice messages to the Telegram bot.
    - Posts should be automatically created and published on LinkedIn with generated images.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| For more tutorials visit our [Youtube](https://www.youtube.com/@AIBIZElevate77)                                                                                                                                                                         | Setup and customization instructions in Sticky Note8                                                            |
| Create a Telegram bot via @BotFather and copy your Bot Token into Telegram nodes.                                                                                                                                                                        | Setup instructions in Sticky Note8                                                                                |
| Add credentials for Telegram, OpenAI, and LinkedIn (OAuth) in n8n credentials manager. For self-hosted LinkedIn OAuth, create an app at developer.linkedin.com to get Client ID and Secret.                                                             | Setup instructions in Sticky Note8                                                                                |
| LinkedIn Post Text prompt is highly customizable to change writing style or structure.                                                                                                                                                                  | Customization advice in Sticky Note8                                                                              |
| Image generation settings (size, style, model) can be adjusted in Create Image node.                                                                                                                                                                    | Customization advice in Sticky Note8                                                                              |
| Consider integrating Google Sheets, Airtable, or Notion for post approval workflows, scheduling, or analytics. Multi-platform posting to Twitter, Facebook, Instagram can be added for wider reach.                                                       | Workflow extension suggestions in Sticky Note8                                                                    |
| For further automation or integration help, [book an appointment](https://aibizelevate.com/) or contact via [LinkedIn](https://www.linkedin.com/in/barbora-svobodova-461b92285/)                                                                         | Support contact in Sticky Note8                                                                                    |
| LinkedIn OAuth requires an OAuth flow; double-click "Create a post" node to authenticate your LinkedIn account. Ensure tokens are refreshed to avoid posting failure.                                                                                   | Authentication note in Sticky Note4                                                                                |
| Voice messages require good audio quality for accurate transcription; transcription failures can impact post quality.                                                                                                                                   | Edge case note from Transcribe Audio node                                                                         |
| Telegram API limits and network issues can cause failures in receiving or downloading files.                                                                                                                                                            | Edge case notes from Telegram Trigger and Download File nodes                                                    |
| OpenAI API rate limits or errors can affect transcription, text formatting, image prompt generation, and image creation steps; consider implementing retry or error handling strategies.                                                                | Edge case notes from various OpenAI nodes                                                                         |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---