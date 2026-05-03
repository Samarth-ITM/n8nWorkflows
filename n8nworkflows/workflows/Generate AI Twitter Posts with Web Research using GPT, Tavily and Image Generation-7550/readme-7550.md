Generate AI Twitter Posts with Web Research using GPT, Tavily and Image Generation

https://n8nworkflows.xyz/workflows/generate-ai-twitter-posts-with-web-research-using-gpt--tavily-and-image-generation-7550


# Generate AI Twitter Posts with Web Research using GPT, Tavily and Image Generation

### 1. Workflow Overview

This workflow, titled **"Generate AI Twitter Posts with Web Research using GPT, Tavily and Image Generation"**, automates the creation of engaging Twitter content by combining AI language models, web research, and image generation. It is designed for content creators, social media managers, and marketers who want to produce timely, viral Twitter posts enhanced with custom AI-generated images.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives Twitter post ideas or chat messages as triggers.
- **1.2 Web Research Integration**: Uses Tavily to perform current web searches based on input.
- **1.3 AI Twitter Content Generation**: Employs OpenAI GPT-4 based agents to create engaging tweet text grounded in recent research.
- **1.4 AI Image Prompt Generation**: Converts tweet text into detailed prompts for image generation.
- **1.5 AI Image Generation**: Generates images from prompts using OpenAI’s image generation API.
- **1.6 Email Notification**: Sends the generated tweet and image via email.
- **1.7 Supportive Elements**: Sticky notes providing setup instructions and credential reminders.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming chat messages that serve as the seed input for Twitter content generation.

**Nodes Involved:**  
- When chat message received

**Node Details:**

- **When chat message received**  
  - *Type & Role:* LangChain chat trigger node; entry point webhook that activates workflow on chat message reception.  
  - *Configuration:* Default options; no custom parameters.  
  - *Expressions/Variables:* Outputs `chatInput` from incoming message JSON.  
  - *Connections:* Outputs to "Twitter Content" node.  
  - *Version-Specific Requirements:* Uses version 1.3 of this node type.  
  - *Edge Cases:* Webhook failures, message format errors, or missing payloads could cause workflow halt.  
  - *Sub-workflows:* None.

---

#### 2.2 Web Research Integration

**Overview:**  
This block performs live web research using Tavily’s API to gather current and trending information related to the input query.

**Nodes Involved:**  
- Search in Tavily

**Node Details:**

- **Search in Tavily**  
  - *Type & Role:* Tavily search node; queries Tavily with the chat input to retrieve recent information for tweet context.  
  - *Configuration:* The search query is dynamically set from `chatInput` coming from the trigger node.  
  - *Expressions:* `={{ $json.chatInput }}` for query parameter.  
  - *Connections:* Connected as an AI tool input to the "Twitter Content" node.  
  - *Credentials:* Requires Tavily API credentials.  
  - *Edge Cases:* API rate limits, no search results, or connection issues.  
  - *Sub-workflows:* None.

---

#### 2.3 AI Twitter Content Generation

**Overview:**  
Generates tweet text using an AI agent that integrates Tavily research results and applies a sophisticated content creation strategy.

**Nodes Involved:**  
- Twitter Content  
- OpenAI Chat Model1

**Node Details:**

- **Twitter Content**  
  - *Type & Role:* LangChain agent node; main AI agent generating Twitter content based on input text and web research.  
  - *Configuration:*  
    - Input text taken from `chatInput`.  
    - System prompt defines role as expert Twitter content creator with detailed instructions on research protocol, tweet creation framework, content categories, optimization rules, and quality checks.  
    - Prompt emphasizes using Tavily search results to produce viral, engaging tweets under 280 characters.  
  - *Expressions:* `={{ $json.chatInput }}` for input text.  
  - *Connections:*  
    - Receives AI language model input from "OpenAI Chat Model1".  
    - Receives AI tool input from "Search in Tavily".  
    - Outputs tweet text to "Twitter Image Prompt" node.  
  - *Credentials:* None directly; uses linked OpenAI Chat Model.  
  - *Edge Cases:* AI model response errors, malformed output, or insufficient input data.  
  - *Sub-workflows:* None.

- **OpenAI Chat Model1**  
  - *Type & Role:* LangChain OpenAI Chat model node; provides GPT-4.1-mini model for the Twitter Content agent.  
  - *Configuration:* GPT-4.1-mini model selected; no extra options.  
  - *Credentials:* Requires OpenAI API key.  
  - *Connections:* Outputs to "Twitter Content" node as AI language model.  
  - *Edge Cases:* API key errors, rate limits, timeout, or model unavailability.  
  - *Sub-workflows:* None.

---

#### 2.4 AI Image Prompt Generation

**Overview:**  
Transforms the generated tweet text into a descriptive image generation prompt to create compelling visuals aligned with the tweet’s message.

**Nodes Involved:**  
- Twitter Image Prompt  
- OpenAI Chat Model

**Node Details:**

- **Twitter Image Prompt**  
  - *Type & Role:* LangChain agent node; expert visual strategist that analyzes tweet text and produces an optimized prompt for AI image generation.  
  - *Configuration:*  
    - Input text is the output tweet text from "Twitter Content".  
    - System message guides prompt creation with detailed instructions on visual strategy, style, composition, color palette, and platform optimization for Twitter.  
  - *Expressions:* `={{ $json.output }}` for tweet text input.  
  - *Connections:*  
    - Receives AI language model input from "OpenAI Chat Model".  
    - Outputs detailed image prompt to "gpt-image-1" HTTP request node.  
  - *Credentials:* None directly; uses linked OpenAI Chat Model.  
  - *Edge Cases:* Model generation errors, ambiguous tweet text, or prompt formatting issues.  
  - *Sub-workflows:* None.

- **OpenAI Chat Model**  
  - *Type & Role:* LangChain OpenAI Chat model node; GPT-4.1-mini model supporting image prompt generation.  
  - *Configuration:* GPT-4.1-mini selected; no additional options.  
  - *Credentials:* Uses OpenAI API key.  
  - *Connections:* Outputs to "Twitter Image Prompt".  
  - *Edge Cases:* API errors or timeouts.  
  - *Sub-workflows:* None.

---

#### 2.5 AI Image Generation

**Overview:**  
Generates an image based on the AI-generated image prompt using OpenAI’s image generation API.

**Nodes Involved:**  
- gpt-image-1  
- Convert to File

**Node Details:**

- **gpt-image-1**  
  - *Type & Role:* HTTP Request node; sends prompt to OpenAI images generation endpoint to create a 1024x1024 image.  
  - *Configuration:*  
    - POST request to `https://api.openai.com/v1/images/generations`.  
    - Body includes model "gpt-image-1", the prompt from "Twitter Image Prompt" output, and image size 1024x1024.  
    - Uses Bearer token authentication with OpenAI API key.  
  - *Expressions:* `={{ $json.output }}` for prompt.  
  - *Connections:* Outputs to "Convert to File".  
  - *Credentials:* OpenAI API key configured as HTTP Bearer Auth.  
  - *Edge Cases:* API limits, invalid prompt, or network issues.  
  - *Sub-workflows:* None.

- **Convert to File**  
  - *Type & Role:* ConvertToFile node; converts base64 image data into binary file format for downstream usage.  
  - *Configuration:* Converts `"data[0].b64_json"` JSON property to binary.  
  - *Connections:* Outputs to "Send a message" node.  
  - *Edge Cases:* Missing or malformed image data causing conversion failure.  
  - *Sub-workflows:* None.

---

#### 2.6 Email Notification

**Overview:**  
Sends the generated tweet content and attached image file by email for review or posting.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message**  
  - *Type & Role:* Gmail node; sends an email with generated tweet text and attached AI image.  
  - *Configuration:*  
    - Recipient email address hardcoded as `"emailhere"`.  
    - Subject set to "New Tweet Generated".  
    - Email body contains the tweet text from "Twitter Content" output JSON.  
    - Sends plain text email without appended attribution.  
    - Attachments included from binary data (converted AI image).  
  - *Credentials:* Gmail OAuth2 credentials configured; requires Google Cloud Console client ID and secret.  
  - *Edge Cases:* Authentication failure, email sending limits, invalid recipient address.  
  - *Sub-workflows:* None.

---

#### 2.7 Supportive Elements

**Overview:**  
Sticky notes provide important setup instructions related to API keys and credentials for OpenAI, Tavily, and Gmail.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**

- **Sticky Note**  
  - *Content:* Reminder to use OpenAI API key for chat model credentials and Tavily API key for corresponding credentials setup.  
  - *Position:* Placed near research and AI model nodes.

- **Sticky Note1**  
  - *Content:* Gmail credentials require client ID and secret from Google Cloud Console (https://console.cloud.google.com).  
  - *Position:* Near the email sending node.

- **Sticky Note2**  
  - *Content:* For generic credential type, choose Bearer Auth and provide the API key.  
  - *Position:* Near HTTP request node for image generation.

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                        | Input Node(s)                 | Output Node(s)         | Sticky Note                                                                                      |
|-------------------------|--------------------------------------------|-------------------------------------|------------------------------|------------------------|------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger                    | Input reception webhook             | -                            | Twitter Content         |                                                                                                |
| Search in Tavily          | Tavily Search Node                        | Web research integration            | -                            | Twitter Content (ai_tool) | Use your tavily api key to setup the credentials (Sticky Note)                                 |
| Twitter Content           | LangChain Agent                          | AI Twitter content generation       | When chat message received, Search in Tavily, OpenAI Chat Model1 | Twitter Image Prompt    | Use your openai api key to setup the chat model credentials (Sticky Note)                      |
| OpenAI Chat Model1        | LangChain OpenAI Chat Model              | Provides GPT model for content gen  | -                            | Twitter Content (ai_languageModel) | Use your openai api key to setup the chat model credentials (Sticky Note)                      |
| Twitter Image Prompt      | LangChain Agent                          | Generates image prompt from tweet   | Twitter Content, OpenAI Chat Model | gpt-image-1             |                                                                                                |
| OpenAI Chat Model         | LangChain OpenAI Chat Model              | Provides GPT model for image prompt | -                            | Twitter Image Prompt (ai_languageModel) | Use your openai api key to setup the chat model credentials (Sticky Note)                      |
| gpt-image-1               | HTTP Request                            | AI image generation API call        | Twitter Image Prompt          | Convert to File          | Choose Bearer Auth and use your api key (Sticky Note2)                                        |
| Convert to File           | ConvertToFile                            | Converts base64 image to binary     | gpt-image-1                  | Send a message           |                                                                                                |
| Send a message            | Gmail Node                              | Sends generated tweet + image email | Convert to File              | -                        | Use the client id and secret id you got from https://console.cloud.google.com (Sticky Note1)  |
| Sticky Note               | Sticky Note                             | Setup instructions                  | -                            | -                        | Use your openai api key to setup the chat model credentials; use your tavily api key to setup the credentials |
| Sticky Note1              | Sticky Note                             | Gmail credentials instructions      | -                            | -                        | Use the client id and secret id you got from https://console.cloud.google.com                  |
| Sticky Note2              | Sticky Note                             | Generic credential (Bearer Auth)    | -                            | -                        | Choose Bearer Auth and use your api key                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **LangChain Chat Trigger** node named "When chat message received".  
   - Use default webhook settings. This node will receive the initial Twitter topic or chat input.

2. **Add Tavily Search Node:**  
   - Add a **Tavily Search** node named "Search in Tavily".  
   - Set the query parameter to: `={{ $json.chatInput }}` to dynamically use the input from the chat trigger.  
   - Configure Tavily API credentials with your Tavily API key.

3. **Configure OpenAI Chat Model for Twitter Content:**  
   - Add a **LangChain OpenAI Chat Model** node named "OpenAI Chat Model1".  
   - Select model: `gpt-4.1-mini`.  
   - Configure with your OpenAI API key credentials.

4. **Create Twitter Content Agent Node:**  
   - Add a **LangChain Agent** node named "Twitter Content".  
   - Set the input text parameter to: `={{ $json.chatInput }}`.  
   - Paste the detailed system prompt (provided in workflow overview section 2.3) that instructs the agent on Twitter content creation, research usage (Tavily), optimization, and output structure.  
   - Connect "When chat message received" to "Twitter Content" main input.  
   - Connect "Search in Tavily" to "Twitter Content" AI tool input.  
   - Connect "OpenAI Chat Model1" to "Twitter Content" AI language model input.

5. **Configure OpenAI Chat Model for Image Prompt:**  
   - Add another **LangChain OpenAI Chat Model** node named "OpenAI Chat Model".  
   - Select model: `gpt-4.1-mini`.  
   - Configure with the same OpenAI API key credential.

6. **Create Twitter Image Prompt Agent:**  
   - Add a **LangChain Agent** node named "Twitter Image Prompt".  
   - Set input text: `={{ $json.output }}` (output from "Twitter Content").  
   - Paste the detailed system message describing how to convert tweets into image generation prompts, including visual strategy and Twitter platform optimization (see section 2.4).  
   - Connect "Twitter Content" to "Twitter Image Prompt" main input.  
   - Connect "OpenAI Chat Model" to "Twitter Image Prompt" AI language model input.

7. **Add HTTP Request Node for Image Generation:**  
   - Add an **HTTP Request** node named "gpt-image-1".  
   - Configure for POST request to `https://api.openai.com/v1/images/generations`.  
   - Body parameters:  
     - model: `gpt-image-1`  
     - prompt: `={{ $json.output }}` (from "Twitter Image Prompt")  
     - size: `1024x1024`  
   - Set authentication as Bearer Auth with your OpenAI API key.  
   - Connect "Twitter Image Prompt" to "gpt-image-1".

8. **Add ConvertToFile Node:**  
   - Add a **ConvertToFile** node named "Convert to File".  
   - Operation: "toBinary".  
   - Source Property: `"data[0].b64_json"` (this extracts base64 image data from OpenAI response).  
   - Connect "gpt-image-1" to "Convert to File".

9. **Add Gmail Node for Email Sending:**  
   - Add a **Gmail** node named "Send a message".  
   - Configure with Gmail OAuth2 credentials, created via Google Cloud Console (client ID and secret).  
   - Set recipient email to your desired address (replace `"emailhere"`).  
   - Email subject: "New Tweet Generated".  
   - Email body: `={{ $('Twitter Content').item.json.output }}` (tweet text).  
   - Attach the binary image file from "Convert to File".  
   - Connect "Convert to File" to "Send a message".

10. **Add Sticky Notes for Documentation:**  
    - Add three sticky notes with instructions for setting up OpenAI, Tavily API keys, and Gmail credentials as described in section 2.7.

11. **Connect Nodes:**  
    - Connect "When chat message received" → "Twitter Content".  
    - Connect "Search in Tavily" → "Twitter Content" (AI tool input).  
    - Connect "OpenAI Chat Model1" → "Twitter Content" (AI language model input).  
    - Connect "Twitter Content" → "Twitter Image Prompt".  
    - Connect "OpenAI Chat Model" → "Twitter Image Prompt" (AI language model input).  
    - Connect "Twitter Image Prompt" → "gpt-image-1".  
    - Connect "gpt-image-1" → "Convert to File".  
    - Connect "Convert to File" → "Send a message".

12. **Test the Workflow:**  
    - Trigger a chat message with a Twitter post idea.  
    - Verify each step produces expected outputs: researched data, tweet text, image prompt, AI-generated image, and email sent successfully.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                    |
|--------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Use your OpenAI API key to set up chat model credentials.                                                    | Sticky Note near AI model nodes                    |
| Use your Tavily API key to set up credentials.                                                              | Sticky Note near Tavily search node                |
| Gmail credentials require client ID and secret from Google Cloud Console: https://console.cloud.google.com | Sticky Note near Gmail node                         |
| For generic credential type nodes (HTTP request), use Bearer Auth and provide API key.                       | Sticky Note near HTTP Request node for image gen  |

---

**Disclaimer:** The text and data processed in this workflow are fully compliant with current content policies and only handle legal and public data.