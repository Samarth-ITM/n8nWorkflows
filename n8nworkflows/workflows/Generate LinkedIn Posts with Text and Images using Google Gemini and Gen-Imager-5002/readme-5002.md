Generate LinkedIn Posts with Text and Images using Google Gemini and Gen-Imager

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-with-text-and-images-using-google-gemini-and-gen-imager-5002


# Generate LinkedIn Posts with Text and Images using Google Gemini and Gen-Imager

---

### 1. Workflow Overview

This workflow automates the creation and publication of professional LinkedIn posts enhanced by AI-generated images. It is designed for digital marketers, content creators, and professionals seeking to streamline their LinkedIn content production with AI assistance.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures user-submitted topics via a web form.
- **1.2 Prompt Preparation:** Maps and prepares the input for AI processing.
- **1.3 AI Content Generation:** Utilizes Google Gemini AI to generate LinkedIn post content and a corresponding image prompt.
- **1.4 Output Normalization:** Cleans and formats the AI response to distinct text and image prompt components.
- **1.5 Image Generation:** Uses the gen-imager API to create a professional image based on the AI-generated prompt.
- **1.6 Image Decoding:** Decodes the received base64 image data into a binary buffer suitable for LinkedIn upload.
- **1.7 Post Publishing:** Publishes the generated text and image as a LinkedIn post.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block initiates the workflow by listening for form submissions where users provide a topic for a LinkedIn post.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Captures form input to start the workflow  
    - *Configuration:*  
      - Form Title: "LinkedIn Post Generator"  
      - Single required field labeled "Topic" with placeholder "Enter prompt here...."  
    - *Input/Output:* No input; outputs JSON containing the submitted topic  
    - *Edge Cases:* Missing or empty topic input will prevent workflow continuation; form validation enforces required field  
    - *Notes:* Triggers workflow on HTTP webhook call upon submission

#### 2.2 Prompt Preparation

- **Overview:**  
Maps the user’s topic input into a variable for downstream nodes to consume in AI prompt construction.

- **Nodes Involved:**  
  - Mapper

- **Node Details:**  
  - **Mapper**  
    - *Type:* Set node  
    - *Role:* Assigns the submitted topic to a variable named `chatInput`  
    - *Configuration:* Assigns `chatInput` = value of `Topic` from form submission JSON  
    - *Input/Output:* Input from form submission; output includes `chatInput` string  
    - *Edge Cases:* Expression failure if `Topic` field is missing or malformed, but form validation mitigates this

#### 2.3 AI Content Generation

- **Overview:**  
Generates professional LinkedIn post content and an image prompt using Google Gemini AI, based on the provided topic.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model

- **Node Details:**  
  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Acts as the orchestrator for AI content generation  
    - *Configuration:*  
      - System message template includes instructions to generate a LinkedIn post with:  
        - Engaging text with hook, insights, CTA, and relevant hashtags  
        - An image prompt focusing on professional LinkedIn aesthetics and corporate themes (e.g., modern workspaces, AI elements)  
      - Output expected in structured JSON with `post_content` and `image_prompt` keys  
    - *Input/Output:* Receives `chatInput` from Mapper; outputs AI-generated JSON string  
    - *Edge Cases:* AI service downtime, malformed JSON output, generation timeout, or prompt interpretation errors  
  - **Google Gemini Chat Model**  
    - *Type:* LangChain Google Gemini LM node  
    - *Role:* Provides underlying AI language model for the AI Agent  
    - *Configuration:* Uses model "models/gemini-2.0-flash" with Google PaLM API credentials  
    - *Input/Output:* Receives prompt from AI Agent; outputs raw AI response JSON string  
    - *Edge Cases:* API authentication failure, rate limiting, network errors

#### 2.4 Output Normalization

- **Overview:**  
Cleans and parses the AI-generated JSON string, extracting the LinkedIn post text and image prompt description for further use.

- **Nodes Involved:**  
  - Normalizer

- **Node Details:**  
  - **Normalizer**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Strips code block markdown, parses JSON, extracts `post_content.text` and `image_prompt.description`  
    - *Configuration:* Custom JS script that handles string cleanup and JSON parsing  
    - *Input/Output:* Input raw AI output string; outputs structured JSON with two properties: `post_content` (text) and `image_prompt` (string)  
    - *Edge Cases:* JSON parsing errors if AI output format changes or contains syntax errors

#### 2.5 Image Generation

- **Overview:**  
Sends the image prompt to the gen-imager API to generate a professional image aligned with LinkedIn’s corporate style.

- **Nodes Involved:**  
  - Text to image

- **Node Details:**  
  - **Text to image**  
    - *Type:* HTTP Request node  
    - *Role:* Calls gen-imager API to generate image from prompt  
    - *Configuration:*  
      - POST request with multipart-form-data containing the `Prompt` parameter set to the extracted image prompt  
      - Headers include `x-rapidapi-host` and `x-rapidapi-key` for authentication  
    - *Input/Output:* Input is image prompt string; outputs API response JSON containing base64 image data  
    - *Edge Cases:* API key invalid or missing, rate limits, network timeouts, malformed prompt causing API errors

#### 2.6 Image Decoding

- **Overview:**  
Decodes the base64-encoded image data received from the gen-imager API into a binary buffer suitable for LinkedIn upload.

- **Nodes Involved:**  
  - Decoder

- **Node Details:**  
  - **Decoder**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Parses API response, extracts base64 image string, decodes it into a buffer, and returns base64 string again for compatibility  
    - *Configuration:* Custom JS code that safely parses JSON, extracts `base64_img`, converts to buffer, and returns base64 string and length  
    - *Input/Output:* Input raw API response JSON string; output JSON with `decoded_image_buffer` (base64 string), and image size info  
    - *Edge Cases:* JSON parse errors, missing base64_img field, corrupted image data

#### 2.7 Post Publishing

- **Overview:**  
Publishes the generated LinkedIn post including the text and the decoded image directly to the user's LinkedIn profile.

- **Nodes Involved:**  
  - LinkedIn

- **Node Details:**  
  - **LinkedIn**  
    - *Type:* LinkedIn node  
    - *Role:* Uploads and shares the post content with associated image  
    - *Configuration:*  
      - Text parameter set dynamically from Normalizer node’s `post_content` output  
      - Binary property name set to decoded image buffer for image upload  
      - Media category set to "IMAGE"  
    - *Input/Output:* Input post text and image buffer; outputs LinkedIn API response  
    - *Edge Cases:* Authentication failure, API rate limits, image upload errors, text formatting issues

---

### 3. Summary Table

| Node Name              | Node Type                         | Functional Role                                    | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                                              |
|------------------------|----------------------------------|---------------------------------------------------|-----------------------|----------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger                     | Captures user topic input via form submission     | None                  | Mapper               | **On Form Submission**  \n> Triggered when a user submits a topic through the form. This starts the workflow and captures the topic to generate a LinkedIn post. |
| Mapper                 | Set                             | Maps input topic to `chatInput` variable          | On form submission    | AI Agent             | > **Mapper**  \n> Maps the user-submitted topic from the form and prepares it for the next step by assigning it to a variable (`chatInput`). |
| AI Agent               | LangChain Agent                 | Orchestrates AI text and image prompt generation  | Mapper                | Normalizer           | **AI Agent**  \n> Uses the **Google Gemini** model to generate professional content for the LinkedIn post, including text and an image prompt based on the given topic. |
| Google Gemini Chat Model | LangChain Google Gemini LM Node | Provides AI language model for content generation | AI Agent (ai_languageModel) | AI Agent           |                                                                                                                                          |
| Normalizer             | Code                            | Cleans and parses AI output into structured data  | AI Agent              | Text to image        |  **Normalizer**  \n> Cleans and formats the AI-generated output into a readable structure for the next steps, extracting both the post text and image prompt. |
| Text to image          | HTTP Request                    | Sends prompt to gen-imager API for image creation | Normalizer            | Decoder              | > **Text to Image**  \n> Sends the image prompt to the **[gen-imager API](https://rapidapi.com/PrineshPatel/api/gen-imager)** to generate a professional image for the LinkedIn post based on the given topic. |
| Decoder                | Code                            | Decodes base64 image data into binary buffer      | Text to image         | LinkedIn             | **Decoder**  \n> Decodes the image from its base64 format into a usable binary buffer that can be uploaded to LinkedIn.                  |
| LinkedIn               | LinkedIn API                   | Publishes text and image as LinkedIn post         | Decoder               | None                 | > **LinkedIn**  \n> Publishes the generated LinkedIn post, including the text and the newly created image, directly to the user's LinkedIn profile. |
| Sticky Note            | Sticky Note                    | Documentation and comments                         | None                  | None                 | > **AI-Powered LinkedIn Post Automation**  \n> This workflow automates the process of creating LinkedIn posts based on user-submitted topics. It generates both **content** and a **professional image** using AI, and automatically publishes the post to LinkedIn.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure Form Title: "LinkedIn Post Generator"  
   - Add one required field: Label "Topic", placeholder "Enter prompt here...."  
   - This node triggers the workflow on HTTP webhook form submission.

2. **Create "Mapper" node**  
   - Type: Set  
   - Add assignment: Variable name `chatInput` assigned from expression `{{$json.Topic}}` (the form input)  
   - Connect "On form submission" output → "Mapper" input.

3. **Create "AI Agent" node**  
   - Type: LangChain Agent  
   - Configure system message with instructions to generate LinkedIn post content and image prompt using `{{ $json.chatInput }}`  
   - Use the detailed prompt template from the original workflow to ensure output JSON with `post_content.text` and `image_prompt.description`.  
   - Connect "Mapper" output → "AI Agent" input.

4. **Create "Google Gemini Chat Model" node**  
   - Type: LangChain Google Gemini LM Node  
   - Model Name: `models/gemini-2.0-flash`  
   - Assign Google PaLM API credentials (setup required in n8n)  
   - Connect "Google Gemini Chat Model" output to "AI Agent" language model input.

5. **Create "Normalizer" node**  
   - Type: Code (JavaScript)  
   - Insert JS code to strip markdown, parse JSON, extract `post_content.text` and `image_prompt.description` as output JSON keys.  
   - Connect "AI Agent" output → "Normalizer" input.

6. **Create "Text to image" node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://gen-imager.p.rapidapi.com/genimager/index.php`  
   - Content-Type: multipart/form-data  
   - Body Parameters: One with name `Prompt` and value expression `{{$json.image_prompt}}`  
   - Header Parameters:  
     - `x-rapidapi-host`: `gen-imager.p.rapidapi.com`  
     - `x-rapidapi-key`: Your gen-imager RapidAPI key (credential needed)  
   - Connect "Normalizer" output → "Text to image" input.

7. **Create "Decoder" node**  
   - Type: Code (JavaScript)  
   - JS code to parse the API JSON response, extract `base64_img`, decode to buffer, and return base64 string and length.  
   - Connect "Text to image" output → "Decoder" input.

8. **Create "LinkedIn" node**  
   - Type: LinkedIn API node  
   - Text: Set value to expression `{{$node["Normalizer"].json.post_content}}`  
   - Binary Property Name: Set to `decoded_image_buffer` (the decoded base64 image buffer from Decoder node)  
   - Media Category: "IMAGE"  
   - Configure LinkedIn OAuth2 credentials in n8n  
   - Connect "Decoder" output → "LinkedIn" input.

9. **Verify and test the full flow:**  
   - Submit a test topic via the form webhook URL  
   - Confirm AI generates text and image prompt correctly  
   - Confirm image is generated via gen-imager API  
   - Confirm LinkedIn post is published with text and image.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow automates LinkedIn post creation with AI-generated text and images, streamlining social media marketing efforts.   | General project description                                                                              |
| gen-imager API is used for professional image generation based on AI prompts.                                                     | [gen-imager API on RapidAPI](https://rapidapi.com/PrineshPatel/api/gen-imager)                           |
| Google Gemini (PaLM) is the AI model used for content generation, requiring Google PaLM API credentials.                         | Google PaLM API documentation and credential setup in n8n                                                |
| Suggested fonts and visual style for images: clean, modern sans-serif fonts (Helvetica, Arial, Roboto), corporate color palettes. | Included in AI prompt system message for consistent branding and style                                   |
| Ensure LinkedIn OAuth2 credentials are properly configured for API access and posting permissions.                               | LinkedIn Developer portal for OAuth2 setup                                                               |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.