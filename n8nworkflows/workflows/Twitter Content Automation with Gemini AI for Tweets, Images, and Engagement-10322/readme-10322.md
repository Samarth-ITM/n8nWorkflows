Twitter Content Automation with Gemini AI for Tweets, Images, and Engagement

https://n8nworkflows.xyz/workflows/twitter-content-automation-with-gemini-ai-for-tweets--images--and-engagement-10322


# Twitter Content Automation with Gemini AI for Tweets, Images, and Engagement

---

### 1. Workflow Overview

This workflow automates Twitter content creation and engagement using AI (Google Gemini and Langchain agents) combined with image generation via an external API. It targets social media managers, marketers, and content creators who want to generate tweets, images, direct messages, and engagement actions (likes) dynamically based on user input.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Configuration:** Receives user input via a web form specifying topic, tone, action type, and optional image instructions; sets configuration constants.
- **1.2 AI Content Generation:** Uses AI agents to generate tweet text and image prompts from the inputs.
- **1.3 Image Generation Pipeline:** Optionally creates images based on AI-generated prompts using an external image API.
- **1.4 Content Assembly & Posting:** Merges tweet text and images, then posts tweets or sends DMs; also supports engagement actions like liking tweets.
- **1.5 AI-Driven Twitter Tools:** Exposes Twitter post, DM, and engagement tools as callable AI tools for dynamic operations.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Configuration

**Overview:**  
Receives content preferences from the user through a web form and sets up key configuration constants used downstream.

**Nodes Involved:**  
- Twitter Content Form  
- Workflow Configuration

**Node Details:**

- **Twitter Content Form**  
  - *Type:* Form Trigger (Webhook)  
  - *Role:* Entry point; presents a web form with fields for Topic/Niche, Tone (dropdown), Action Type (dropdown), and Additional Instructions for images.  
  - *Configuration:*  
    - Public webhook path: `/twitter-content-form`  
    - Required fields: Topic/Niche, Tone  
    - Optional: Action Type defaults to "Post Tweet" if not selected  
    - Button labeled "Generate Content"  
  - *Input/Output:* Receives HTTP POST with form data; outputs JSON with input fields.  
  - *Failure Modes:* Missing required fields; webhook security risks if public (recommend IP whitelisting or secret paths in production).

- **Workflow Configuration**  
  - *Type:* Set Node  
  - *Role:* Defines constants shared across the workflow.  
  - *Configuration:*  
    - `imageGenerationChance`: 0.3 (30% chance to generate image)  
    - `maxTweetLength`: 280 (Twitter max character limit)  
    - `imageModelUrl`: URL for image generation API  
  - *Input/Output:* Takes form input JSON through connections; adds constants to JSON for downstream nodes.

---

#### 2.2 AI Content Generation

**Overview:**  
Generates tweet content and image prompt instructions using AI agents based on user input and configuration.

**Nodes Involved:**  
- Twitter AI Agent  
- Process AI Response

**Node Details:**

- **Twitter AI Agent**  
  - *Type:* Langchain AI Agent Node  
  - *Role:* Generates tweet text and structured output for Twitter tools.  
  - *Configuration:*  
    - Input prompt template includes user topic, tone, action, additional instructions, and max tweet length.  
    - System message defines capabilities: create tweets under character limit, include image attachments if available, manage interactions.  
    - AI model: Google Gemini (configured separately in Langchain model nodes connected internally)  
  - *Input/Output:*  
    - Input: JSON from Workflow Configuration and form data  
    - Output: JSON with AI-generated tweet text and parameters for tools (like tweet text, DM message, recipient username, tweet IDs for engagement)  
  - *Edge Cases:* AI may produce unexpected or malformed output; prompt tuning recommended.  
  - *Security:* Twitter tools exposed as AI tools; credentials must be secured.

- **Process AI Response**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Extracts and formats AI agent output, adding metadata like timestamps and status.  
  - *Key Code:*  
    - Maps each item, extracts `output` or `text` fields from AI response, wraps in JSON with timestamp and status.  
  - *Input/Output:* Input from AI agent node, output reformatted JSON for posting.

---

#### 2.3 Image Generation Pipeline

**Overview:**  
Optionally generates images based on AI-crafted prompts derived from user instructions, producing binary image data for tweet attachments.

**Nodes Involved:**  
- Fields - Set Values  
- AI Agent - Create Image From Prompt  
- Code - Clean Json  
- Code - Get Prompt  
- Code - Set Filename  
- HTTP Request - Create Image

**Node Details:**

- **Fields - Set Values**  
  - *Type:* Set Node  
  - *Role:* Sets image generation parameters: model type ("flux"), width (1080), height (1920).  
  - *Output:* JSON with image generation config.

- **AI Agent - Create Image From Prompt**  
  - *Type:* Langchain AI Agent  
  - *Role:* Converts user "Additional Instructions" into structured JSON prompts for image generation.  
  - *Configuration:*  
    - System message describes detailed guidelines for prompt creation, including scene, elements, text integration, lighting, technical specs, and style parameters.  
    - Output is a JSON formatted image prompt following strict rules for realism and quality.  
  - *Input/Output:* Receives additional instructions; outputs image prompt JSON.

- **Code - Clean Json**  
  - *Type:* Code Node  
  - *Role:* Parses AI text output to extract an array of image prompts.  
  - *Key Code:* Splits AI text by lines; extracts lines containing `"prompt":`; collects prompts into array.  
  - *Output:* JSON object containing `image_prompt` array.

- **Code - Get Prompt**  
  - *Type:* Code Node  
  - *Role:* Builds request body for image API for each prompt with parameters like image size, steps, guidance scale.  
  - *Output:* Items with JSON body for HTTP request.

- **Code - Set Filename**  
  - *Type:* Code Node  
  - *Role:* Assigns filenames to images in the format `images_001.png`, `images_002.png`, etc.  
  - *Output:* Adds `fileName` property to items.

- **HTTP Request - Create Image**  
  - *Type:* HTTP Request  
  - *Role:* Calls external image generation API (Pollinations) with prompt, retrieves binary image data (`imageData`).  
  - *Configuration:*  
    - URL templated with prompt text  
    - Sends JSON parameters for image size and model  
    - Headers specify JSON content type and acceptance  
    - Response expected as binary file stored in `imageData` property  
    - Retries on failure with 5 seconds wait  
  - *Edge Cases:* Network failures, API rate limits, malformed prompts, binary data handling.

---

#### 2.4 Content Assembly & Posting

**Overview:**  
Merges tweet text and optionally generated images, then posts tweets or sends direct messages. Also supports liking or engaging with tweets.

**Nodes Involved:**  
- Merge Tweet Text and Image  
- Create Tweet  
- Twitter Post Tool  
- Twitter DM Tool  
- Twitter Engagement Tool

**Node Details:**

- **Merge Tweet Text and Image**  
  - *Type:* Merge Node  
  - *Role:* Combines AI-generated tweet text with binary image data (if available) to prepare full content for posting.  
  - *Input:* Two streams - text and image data  
  - *Output:* Single combined item with text and attachments.

- **Create Tweet**  
  - *Type:* Twitter Node  
  - *Role:* Posts tweet with combined content (text and image attachment).  
  - *Configuration:*  
    - Text from AI agent output  
    - Additional fields allow image attachments  
  - *Input/Output:* Inputs merged content; outputs posted tweet info  
  - *Failure:* Auth errors, exceeding character limits, image upload failures.

- **Twitter Post Tool**  
  - *Type:* Twitter Tool Node  
  - *Role:* Posts tweets via AI-invoked tool calls for dynamic action.  
  - *Input:* Text from AI; image attachment binary  
  - *Use:* Invoked by AI agent for posting tweets programmatically.

- **Twitter DM Tool**  
  - *Type:* Twitter Tool Node  
  - *Role:* Sends direct messages using username and message content from AI.  
  - *Input:* DM text and recipient username from AI output  
  - *Failure:* Invalid usernames, DM permissions, rate limits.

- **Twitter Engagement Tool**  
  - *Type:* Twitter Tool Node  
  - *Role:* Likes tweets by tweet ID provided by AI.  
  - *Configuration:* Operation set to "like"  
  - *Input:* Tweet ID from AI  
  - *Failure:* Invalid tweet IDs, rate limits.

---

#### 2.5 AI-Driven Twitter Tools Integration

**Overview:**  
Exposes the Twitter Post, DM, and Engagement nodes as AI tools that the Twitter AI Agent can call dynamically.

**Nodes Involved:**  
- Twitter Post Tool  
- Twitter DM Tool  
- Twitter Engagement Tool

**Node Details:**

- These nodes are connected as AI tools to the Twitter AI Agent node, allowing the AI to invoke Twitter actions programmatically based on generated content or instructions.

- Important to ensure proper OAuth2 credentials for Twitter/X are configured.

- The AI agent uses tool connectors to call these nodes during execution for posting, messaging, or engagement.

---

### 3. Summary Table

| Node Name                    | Node Type                       | Functional Role                            | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                  |
|------------------------------|--------------------------------|-------------------------------------------|-----------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Twitter Content Form          | Form Trigger                   | Entry point; user input via web form      | —                           | Workflow Configuration          | Twitter Form & Config: Trigger on public webhook path `twitter-content-form`                 |
| Workflow Configuration       | Set                           | Sets constants (maxTweetLength, image API) | Twitter Content Form         | Twitter AI Agent, Fields - Set Values |                                                                                              |
| Twitter AI Agent             | Langchain AI Agent             | Generates tweet text and parameters       | Workflow Configuration       | Process AI Response, Merge Tweet Text and Image | Twitter Content AI: Generates tweet text and tool parameters                                |
| Process AI Response           | Code                          | Parses and wraps AI output with metadata  | Twitter AI Agent             | Create Tweet                   |                                                                                              |
| Fields - Set Values           | Set                           | Sets image generation parameters          | Workflow Configuration       | AI Agent - Create Image From Prompt | Optional Image Generation Path: sets model, width, height                                   |
| AI Agent - Create Image From Prompt | Langchain AI Agent       | Creates structured image prompts           | Fields - Set Values          | Code - Clean Json             | Detailed prompt guidelines for realistic image generation                                   |
| Code - Clean Json             | Code                          | Extracts prompts array from AI text output | AI Agent - Create Image From Prompt | Code - Get Prompt             |                                                                                              |
| Code - Get Prompt            | Code                          | Builds JSON body for image API requests   | Code - Clean Json            | Code - Set Filename           |                                                                                              |
| Code - Set Filename          | Code                          | Assigns image filenames                     | Code - Get Prompt            | HTTP Request - Create Image    |                                                                                              |
| HTTP Request - Create Image  | HTTP Request                  | Calls external API to generate images      | Code - Set Filename          | Merge Tweet Text and Image     | Image Generated Output: binary imageData returned, preview available                         |
| Merge Tweet Text and Image   | Merge                         | Combines tweet text and images              | Twitter AI Agent, HTTP Request - Create Image | Create Tweet                  | Merge & Publish (Twitter): combines text + image for posting                               |
| Create Tweet                | Twitter                       | Posts tweet with text and image             | Process AI Response, Merge Tweet Text and Image | —                             | Posting/DM/Engagement: posts tweets, must configure Twitter credentials                      |
| Twitter Post Tool           | Twitter Tool                  | AI-invoked tweet posting tool                | Twitter AI Agent (ai_tool)   | —                             | Posting/DM/Engagement: Twitter post tool exposed to AI                                     |
| Twitter DM Tool             | Twitter Tool                  | AI-invoked direct message tool                | Twitter AI Agent (ai_tool)   | —                             | Posting/DM/Engagement: DM tool exposed to AI                                              |
| Twitter Engagement Tool     | Twitter Tool                  | AI-invoked engagement tool (like tweets)     | Twitter AI Agent (ai_tool)   | —                             | Posting/DM/Engagement: engagement (like) tool exposed to AI                               |
| Google Gemini Chat Model2   | Langchain LM Chat (Google Gemini) | Underlying AI model for image prompt agent  |                             | AI Agent - Create Image From Prompt |                                                                                              |
| Google Gemini Chat Model3   | Langchain LM Chat (Google Gemini) | Underlying AI model for Twitter AI Agent     |                             | Twitter AI Agent              |                                                                                              |
| Sticky – Twitter Form        | Sticky Note                   | Explains form trigger and configuration     |                             |                               | Twitter Form & Config                                                                         |
| Sticky – Image Gen Flow      | Sticky Note                   | Explains image generation pipeline           |                             |                               | Optional Image Generation Path                                                               |
| Sticky – Twitter AI          | Sticky Note                   | Explains Twitter AI content generation       |                             |                               | Twitter Content AI                                                                           |
| Sticky – Merge & Tweet       | Sticky Note                   | Explains merge and posting flow               |                             |                               | Merge & Publish (Twitter)                                                                    |
| Sticky – Twitter Actions     | Sticky Note                   | Explains posting, DM, and engagement tools   |                             |                               | Posting/DM/Engagement                                                                        |
| Sticky – Credentials         | Sticky Note                   | Explains required credentials and environment |                             |                               | Credentials & Environment                                                                    |
| Sticky Note6                 | Sticky Note                   | Explains image output node and API notes      |                             |                               | Image Generated Output                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `Twitter Content Form`  
   - Configure webhook path: `twitter-content-form`  
   - Form title: "Twitter Content Generator"  
   - Fields:  
     - Text input: "Topic/Niche" (required)  
     - Dropdown: "Tone" with options: Casual, Professional, Normal, Humorous (required)  
     - Dropdown: "Action Type" with options: Post Tweet, Engage with Posts, Send Direct Message (optional)  
     - Text input: "Additional Instructions For Image(Prompt)" (optional)  
   - Button label: "Generate Content"

2. **Add a Set Node**  
   - Name: `Workflow Configuration`  
   - Set variables:  
     - `imageGenerationChance` = 0.3  
     - `maxTweetLength` = 280  
     - `imageModelUrl` = "https://openrouter.ai/api/v1/images/generations"  
   - Connect input from `Twitter Content Form`.

3. **Add Langchain AI Agent Node**  
   - Name: `Twitter AI Agent`  
   - Configure prompt input:  
     - Template includes form fields and constants: Topic, Tone, Action, Additional Instructions, Max Tweet Length  
   - System message explains AI role: create tweets under length limit, include images if provided, manage Twitter interactions  
   - Connect input from `Workflow Configuration`.

4. **Add Code Node**  
   - Name: `Process AI Response`  
   - JavaScript: extract AI text or output, wrap with timestamp and status  
   - Connect input from `Twitter AI Agent`.

5. **Add Set Node for Image Settings**  
   - Name: `Fields - Set Values`  
   - Assign:  
     - model = "flux"  
     - width = "1080"  
     - height = "1920"  
   - Connect input from `Workflow Configuration`.

6. **Add Langchain AI Agent Node**  
   - Name: `AI Agent - Create Image From Prompt`  
   - Input: additional instructions from form (`Workflow Configuration` or form data)  
   - System message includes detailed image prompt creation guidelines (scene, elements, style, technical parameters)  
   - Connect input from `Fields - Set Values`.

7. **Add Code Node**  
   - Name: `Code - Clean Json`  
   - JavaScript: parse AI text output to extract prompts array (check lines containing `"prompt":`)  
   - Connect input from `AI Agent - Create Image From Prompt`.

8. **Add Code Node**  
   - Name: `Code - Get Prompt`  
   - JavaScript: map prompts to HTTP request body for image API, including width, height, steps, scale, safety checker enabled  
   - Connect input from `Code - Clean Json`.

9. **Add Code Node**  
   - Name: `Code - Set Filename`  
   - JavaScript: assign filenames like `images_001.png` to each image item  
   - Connect input from `Code - Get Prompt`.

10. **Add HTTP Request Node**  
    - Name: `HTTP Request - Create Image`  
    - URL templated to image API endpoint (e.g. Pollinations or configurable URL)  
    - Method: POST with JSON body from previous node  
    - Headers: Content-Type and Accept as application/json  
    - Response format: file, output property named `imageData`  
    - Retry on failure with 5 seconds wait  
    - Connect input from `Code - Set Filename`.

11. **Add Merge Node**  
    - Name: `Merge Tweet Text and Image`  
    - Merge Mode: Combine inputs (e.g., Merge by index or by property)  
    - Input 1: from `Twitter AI Agent` output (tweet text)  
    - Input 2: from `HTTP Request - Create Image` (binary image data)

12. **Add Twitter Node**  
    - Name: `Create Tweet`  
    - Text: use field from AI agent output  
    - Attachments: set to `imageData` binary property from merge node  
    - Connect input from `Process AI Response` and `Merge Tweet Text and Image`.

13. **Add Twitter Tool Nodes** (Optional for AI tool invocation)  
    - `Twitter Post Tool`: posts tweets with text and image, input from AI agent tool call  
    - `Twitter DM Tool`: sends DMs using AI-supplied username and message  
    - `Twitter Engagement Tool`: likes tweets by ID supplied by AI  
    - Connect these as tools under AI agent node configuration.

14. **Configure Credentials**  
    - Twitter/X OAuth2 credentials for all Twitter nodes  
    - Google Gemini credentials for AI agent nodes  
    - Ensure outbound HTTP access for external image API  
    - Test all credentials to verify connectivity.

15. **Optional: Add Sticky Notes** as in original workflow for user guidance and documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Protect the webhook path `/twitter-content-form` in production with IP allow-listing or secret paths to prevent unauthorized access.                                                                                                     | Security recommendation for Form Trigger node                                                 |
| Image generation uses Pollinations API; verify service terms and compliance before production use. You may swap this with other image generation services by adjusting HTTP request configuration.                                           | Image generation pipeline                                                                    |
| Twitter AI Agent exposes Twitter Post, DM, and Engagement nodes as callable AI tools; ensure credentials and rate limits are respected to avoid failures or bans.                                                                         | AI-Driven Twitter Tools integration                                                          |
| Maximum tweet length is set to 280 characters as per Twitter's limitation. The AI prompt enforces this limit for generated content.                                                                                                       | Workflow Configuration and Twitter AI Agent prompt                                            |
| The AI image prompt agent uses a detailed system message to ensure high-quality, realistic, and professional images with text integration and composition rules.                                                                           | AI Agent - Create Image From Prompt node documentation                                        |
| For previewing generated images, click on the binary data property `imageData` in the HTTP Request node's execution output.                                                                                                              | Sticky Note6 content                                                                          |
| Credentials needed: Twitter OAuth2, Google Gemini AI, and outbound HTTP for external API calls. Test each before deployment.                                                                                                              | Sticky – Credentials note                                                                     |
| Consider adding conditional logic if image data is optional to skip merging or attaching images when none are generated.                                                                                                                | Suggested improvement based on sticky note "Merge & Publish (Twitter)"                        |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---