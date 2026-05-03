Generate Product Images & Videos with Gemini AI, DeepSeek, and GoAPI for E-commerce

https://n8nworkflows.xyz/workflows/generate-product-images---videos-with-gemini-ai--deepseek--and-goapi-for-e-commerce-8148


# Generate Product Images & Videos with Gemini AI, DeepSeek, and GoAPI for E-commerce

### 1. Workflow Overview

This workflow automates the generation of high-quality product images and videos for e-commerce using AI models and media hosting services. It is designed to intake detailed product descriptions and optional text, then produce aesthetic image prompts, generate images from those prompts, and further create cinematic video concepts and outputs featuring the product, optionally including a human model. The final media assets are uploaded to a media hosting service and optionally notified via Discord.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Data Preparation:** Receives product description and user inputs via a form, formats and sets initial variables.
- **1.2 AI Prompt Generation:** Uses AI agents to generate structured image prompts describing the product’s visual attributes.
- **1.3 Image Generation and Processing:** Converts AI-generated prompts into images via Google Gemini and OpenRouter, extracts base64 images, converts to binary files, and uploads to media hosting.
- **1.4 Image Analysis and Visualization:** Analyzes generated images for detailed visual descriptions and creates product-only descriptions.
- **1.5 Video Prompt Generation and Video Creation:** Transforms image descriptions into video prompts, generates videos using GoAPI, polls for completion, downloads, and uploads final videos.
- **1.6 Notifications and UI Rendering:** Sends notifications (Discord) with image links and generates an HTML page showcasing final images.
- **1.7 Error Handling:** Listens for errors and sends alerts to Discord for debugging.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Preparation

**Overview:**  
This block collects detailed product information and options from users via a form trigger and prepares the data for AI processing by formatting and consolidating relevant fields.

**Nodes Involved:**  
- Form 1 - Alpha  
- Get the Data  

**Node Details:**  

- **Form 1 - Alpha**  
  - Type: Form Trigger  
  - Role: Entry point for user input; collects product description, optional text on product, and multiple dropdown options for product photography styles.  
  - Config: Form fields include a required textarea for product description, optional text field, and multiple dropdowns with predefined style options tailored for different product categories.  
  - Output: JSON with user inputs.  
  - Edge Cases: Missing required description will prevent form submission. Dropdown selections may be empty/null if not chosen.

- **Get the Data**  
  - Type: Set  
  - Role: Extracts and consolidates form responses into structured variables: `description`, `text`, and concatenated `productOptions`.  
  - Key Expressions: Uses expressions to combine dropdown selections into a single string (removes quotes).  
  - Input: Output of Form 1 - Alpha  
  - Output: JSON with three keys prepared for AI prompt input.  
  - Edge Cases: Empty dropdown selections result in empty or malformed concatenation strings.

---

#### 1.2 AI Prompt Generation

**Overview:**  
Generates a detailed, structured prompt to describe the product visually, which will be used to generate images. Uses a LangChain agent with a manual JSON schema output parser to ensure structured responses.

**Nodes Involved:**  
- Product Prompt Agent  
- Structured Output Parser  

**Node Details:**  

- **Product Prompt Agent**  
  - Type: LangChain Agent  
  - Role: AI assistant persona acting as a product designer generating a JSON object with keys for product, aesthetics, composition, lighting, scene, text, lens, shot_type, camera_action.  
  - Configuration: System message instructs detailed prompt creation with specific fields; input combines product description, text, and product options.  
  - Output parser linked to Structured Output Parser.  
  - Input: JSON from "Get the Data" node  
  - Output: Structured JSON with the nine keys for image prompt generation.  
  - Edge Cases: AI may fail to produce valid JSON; fallback enabled.

- **Structured Output Parser**  
  - Type: LangChain Output Parser (structured)  
  - Role: Parses AI output according to the manually defined JSON schema, ensuring valid structured data.  
  - Input: Raw AI output from Product Prompt Agent  
  - Output: Validated JSON object with prompt elements  
  - Edge Cases: Parsing failures on malformed AI output.

---

#### 1.3 Image Generation and Processing

**Overview:**  
Generates product images from the structured prompt using AI models, processes base64 image data into files, uploads them to hosted media storage, and extracts final accessible URLs.

**Nodes Involved:**  
- Nano Banana Image  
- Get the Base64 String  
- Convert to File  
- Upload to MediaUpload  
- Extract Image Url  
- Model Prompt Generator  
- Get the Base64 String1  
- Convert to File1  
- Upload to MediaUpload1  
- Extract Image Url1  

**Node Details:**  

- **Nano Banana Image**  
  - Type: HTTP Request  
  - Role: Sends image generation request to OpenRouter’s Gemini 2.5 Flash Image Preview model using the structured prompt fields to generate initial product image.  
  - Configuration: POST with JSON body embedding prompt fields; replaces quotes with `**` to sanitize input strings.  
  - Output: AI response with image as base64-encoded string URL.  
  - Credential: OpenRouter API  
  - Edge Cases: API authentication failures, timeout, malformed JSON.  

- **Get the Base64 String**  
  - Type: Set  
  - Role: Extracts base64 image data from the AI response URL string by splitting at the comma.  
  - Output: Stores `image_string` for conversion.  
  - Input: Nano Banana Image output  
  - Edge Cases: Missing or malformed image URL.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts base64 string to binary file with randomized filename `nano-banana-xxxxx.png`.  
  - Input: `image_string` from Get the Base64 String  
  - Output: Binary file data for upload  
  - Edge Cases: Invalid base64 data, conversion failure.

- **Upload to MediaUpload**  
  - Type: HTTP Request  
  - Role: Uploads binary image file to a media hosting endpoint (custom homelab hosting).  
  - Configuration: Multipart form-data POST with HTTP header authentication.  
  - Output: JSON containing uploaded image URL.  
  - Credential: Mediaupload easypanel  
  - Edge Cases: Upload failure, authentication error.  

- **Extract Image Url**  
  - Type: Set  
  - Role: Extracts and normalizes the media hosting URL (enforces https).  
  - Input: Upload to MediaUpload response  
  - Output: Clean `url` string for further use.  

- **Model Prompt Generator**  
  - Type: HTTP Request  
  - Role: Uses AI (Gemini 2.5 flash image preview) to generate a refined high-resolution image prompt from the first image’s URL.  
  - Input: Prompt from Structured Output Parser1 (see below) and image URL from Extract Image Url.  
  - Credential: OpenRouter API  
  - Edge Cases: API errors.

- **Get the Base64 String1**  
  - Type: Set  
  - Role: Extracts base64 string from Model Prompt Generator response.  

- **Convert to File1**  
  - Type: Convert to File  
  - Role: Converts base64 string to binary file for the second image.  

- **Upload to MediaUpload1**  
  - Type: HTTP Request  
  - Role: Uploads second binary image file to media hosting.  

- **Extract Image Url1**  
  - Type: Set  
  - Role: Extracts final URL for the second uploaded image.

**Note:** Multiple sticky notes emphasize the ability to replace the media hosting endpoint with alternatives like vgy.me if needed.

---

#### 1.4 Image Analysis and Visualization

**Overview:**  
Analyzes generated images for detailed descriptions to guide video creation and marketing assets, and generates a concise product-only description for cataloging.

**Nodes Involved:**  
- Product Visualiser  
- Creative Visualiser  
- Model Prompt Generator  
- Google Gemini Chat Model  
- Google Gemini Chat Model1  
- Google Gemini Chat Model2  

**Node Details:**  

- **Product Visualiser**  
  - Type: Google Gemini Image Analysis  
  - Role: Produces a concise product-only textual description (80-100 words, lowercase, no punctuation) for cataloging and indexing.  
  - Input: Product image URL from Extract Image Url.  
  - Credential: Google Gemini API (Piyush's Work)  
  - Edge Cases: API rate limits or failures.

- **Creative Visualiser**  
  - Type: Google Gemini Image Analysis  
  - Role: Provides a rich, cinematic, visually precise descriptive passage of the product image to inspire video prompt generation.  
  - Input: Product image URL from Extract Image Url1.  
  - Credential: Google Gemini API (Korex)  

- **Model Prompt Generator** (reused in this block)  
  - Type: LangChain Agent  
  - Role: Generates an image prompt based on the concise product description.  

- **Google Gemini Chat Models (1,2)** and **Model (deepseek)** nodes are AI language model nodes configured to generate or assist with prompt generation and creative direction.

---

#### 1.5 Video Prompt Generation and Video Creation

**Overview:**  
Transforms image descriptions into dynamic video prompts, generates a short video using GoAPI, polls for task completion, downloads the video, uploads it to media hosting, and makes it accessible.

**Nodes Involved:**  
- Creative Director  
- Structured Output Parser2  
- Model2  
- Generate Video  
- HTTP Request  
- If  
- Wait  
- Download Video  
- Upload to MediaUpload2  

**Node Details:**  

- **Creative Director**  
  - Type: LangChain Agent  
  - Role: Converts the descriptive image prompt into a cinematic 4-second video ad shot direction prompt in JSON format with key `video_prompt`.  
  - Input: Text from Creative Visualiser output.  
  - Output parser: Structured Output Parser2.  
  - Edge Cases: Parsing failures, AI model timeouts.

- **Structured Output Parser2**  
  - Type: LangChain Output Parser (structured)  
  - Role: Parses the video prompt JSON output from Creative Director.  

- **Model2**  
  - Type: LangChain LM Chat OpenRouter  
  - Role: Supports Creative Director in AI processing with DeepSeek chat model.  

- **Generate Video**  
  - Type: HTTP Request  
  - Role: Submits video generation task to GoAPI with prompt from Creative Director and input image URL.  
  - Configuration: POST to GoAPI’s `/task` endpoint with task type `wan22-img2video-14b`, includes negative prompt constraints to exclude inappropriate content, aspect ratio 16:9.  
  - Credential: GoAPI HTTP Header Auth.  
  - Edge Cases: Task submission failures, invalid prompts.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Polls GoAPI task status endpoint to check if video generation is complete.  
  - Input: Dynamic task ID from Generate Video response.

- **If**  
  - Type: If conditional  
  - Role: Checks if task status is either `processing` or `pending`. If yes, triggers Wait node to delay and poll again; if no, proceeds to download video.  

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow execution before rechecking video task status to avoid excessive polling.  

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads the completed video from GoAPI using the video URL provided in the task output.  

- **Upload to MediaUpload2**  
  - Type: HTTP Request  
  - Role: Uploads the downloaded video file to the same media hosting endpoint for persistent access.  

---

#### 1.6 Notifications and UI Rendering

**Overview:**  
Sends generated image URLs to Discord for logging/notification and generates a user-facing HTML page to showcase the final product and UGC images with download links.

**Nodes Involved:**  
- Discord1  
- Generate Response  

**Node Details:**  

- **Discord1**  
  - Type: Discord Node  
  - Role: Sends a notification message to a Discord webhook with links to the product and user-generated content (UGC) images.  
  - Inputs: URLs from Extract Image Url and Extract Image Url1.  
  - Credential: Discord Webhook API.  

- **Generate Response**  
  - Type: HTML  
  - Role: Generates a responsive HTML page presenting the two final images side by side with download buttons.  
  - Content: Styled with Montserrat font, includes headings and descriptions, uses dynamic URL insertion from Discord1 outputs.  
  - Output: HTML content for web presentation.

---

#### 1.7 Error Handling

**Overview:**  
Catches any workflow errors and sends descriptive error messages to Discord for monitoring and debugging.

**Nodes Involved:**  
- Error Trigger  
- Send the Error  

**Node Details:**  

- **Error Trigger**  
  - Type: Error Trigger  
  - Role: Globally listens for any errors in the workflow execution.  

- **Send the Error**  
  - Type: Discord Node  
  - Role: Sends an error message to a Discord webhook channel, including the error message text.  
  - Credential: Discord Webhook API.  

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                                    | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                         |
|------------------------|----------------------------------|---------------------------------------------------|----------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|
| Form 1 - Alpha         | Form Trigger                     | Collect product description and options from user | -                                | Get the Data                   |                                                                                                   |
| Get the Data           | Set                              | Prepare structured input data for AI agents       | Form 1 - Alpha                   | Product Prompt Agent           |                                                                                                   |
| Product Prompt Agent   | LangChain Agent                  | Generate structured image prompt JSON              | Get the Data                    | Structured Output Parser       | ## 1. The Agent Generate the Image Prompt from the User Request.                                  |
| Structured Output Parser| LangChain Output Parser          | Parse JSON output from Product Prompt Agent        | Product Prompt Agent            | Nano Banana Image              |                                                                                                   |
| Nano Banana Image      | HTTP Request                    | Generate initial product image from prompt         | Structured Output Parser        | Get the Base64 String          | Replace this any image uploader like vgy.me if you dont have a hosted version of mediaupload      |
| Get the Base64 String  | Set                              | Extract base64 string from AI image response       | Nano Banana Image               | Convert to File                | ## 2. Converting Base64 String to Binary                                                          |
| Convert to File        | Convert to File                 | Convert base64 string to binary file                | Get the Base64 String           | Upload to MediaUpload          |                                                                                                   |
| Upload to MediaUpload  | HTTP Request                    | Upload binary image to media hosting                 | Convert to File                 | Extract Image Url              | Replace this any image uploader like vgy.me if you dont have a hosted version of mediaupload      |
| Extract Image Url      | Set                              | Extract and normalize uploaded image URL            | Upload to MediaUpload           | Product Visualiser             | ## 3. Visual description of the image for the product generating model.                           |
| Product Visualiser     | Google Gemini Image Analysis     | Generate concise product description                 | Extract Image Url               | Model Prompt Generator         |                                                                                                   |
| Model Prompt Generator | LangChain Agent                  | Generate refined high-res image prompt              | Product Visualiser              | Structured Output Parser1      | ## 4. Create and image of the product with a Model.                                              |
| Structured Output Parser1| LangChain Output Parser        | Parse JSON output from Model Prompt Generator        | Model Prompt Generator          | Generate Model with Product    |                                                                                                   |
| Generate Model with Product| HTTP Request                | Generate a high-res image from refined prompt        | Structured Output Parser1       | Get the Base64 String1         |                                                                                                   |
| Get the Base64 String1 | Set                              | Extract base64 string from second AI image response | Generate Model with Product     | Convert to File1               | ## 5. Converting Base64 String to Binary again.                                                  |
| Convert to File1       | Convert to File                 | Convert second base64 string to binary file          | Get the Base64 String1          | Upload to MediaUpload1         |                                                                                                   |
| Upload to MediaUpload1 | HTTP Request                    | Upload second binary image to media hosting           | Convert to File1                | Extract Image Url1             | Replace this any image uploader like vgy.me if you dont have a hosted version of mediaupload      |
| Extract Image Url1     | Set                              | Extract and normalize second uploaded image URL      | Upload to MediaUpload1          | Discord1, Creative Visualiser  | ## Send the Product and Model+Product IMage to Discord for logging (optional step)                |
| Discord1               | Discord                         | Send image URLs notification to Discord              | Extract Image Url1              | Generate Response             |                                                                                                   |
| Creative Visualiser    | Google Gemini Image Analysis     | Generate rich image description for video prompt    | Extract Image Url1              | Creative Director             | ## 6. Take Visual Description and Generate the Prompt for Video.                                 |
| Creative Director      | LangChain Agent                  | Generate cinematic video prompt JSON from image desc| Creative Visualiser            | Structured Output Parser2      | ## 7. Video Generation anf Getting the Final Output                                               |
| Structured Output Parser2| LangChain Output Parser        | Parse video prompt JSON output                         | Creative Director              | Generate Video                |                                                                                                   |
| Generate Video         | HTTP Request                    | Submit video generation task to GoAPI                 | Structured Output Parser2       | HTTP Request                 |                                                                                                   |
| HTTP Request           | HTTP Request                    | Poll GoAPI for video task status                       | Generate Video                 | If                           |                                                                                                   |
| If                     | If                              | Check video generation status and decide on wait     | HTTP Request                  | Wait, Download Video          |                                                                                                   |
| Wait                   | Wait                            | Pause workflow before re-polling GoAPI                | If                            | HTTP Request                 |                                                                                                   |
| Download Video         | HTTP Request                    | Download completed video from GoAPI                    | If                            | Upload to MediaUpload2         |                                                                                                   |
| Upload to MediaUpload2 | HTTP Request                    | Upload downloaded video to media hosting               | Download Video                | -                            | Replace this any image uploader like vgy.me if you dont have a hosted version of mediaupload      |
| Generate Response      | HTML                            | Generate HTML page showcasing images with download    | Discord1                      | -                            |                                                                                                   |
| Error Trigger          | Error Trigger                   | Trigger on any workflow error                           | -                            | Send the Error                | Get the Error on Discord or any preferred service for debugging.                                 |
| Send the Error         | Discord                         | Send error message notifications to Discord           | Error Trigger                 | -                            |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("Form 1 - Alpha")**  
   - Path: `/product-description`  
   - Button Label: "Submit a Product"  
   - Fields:  
     - Textarea (required): "A detailed description of the product."  
     - Text (optional): "Text to include in Product (optional)"  
     - Dropdowns for "E-commerce Product Options", "Jewelry Product Options", "Electronics Product Options", "Food Photography Options" with preset options as per workflow JSON.  
   - Append Attribution: False  
   - Save.

2. **Add a Set node ("Get the Data")**  
   - Extract from form data:  
     - `description` = `{{$json["A detailed description of the product."]}}`  
     - `text` = `{{$json["Text to inlucde in Product (optional)"]}}`  
     - `productOptions` = Concatenate dropdown selections into a single string, removing quotes.  
   - Connect output of Form 1 - Alpha to Get the Data.

3. **Add a LangChain Agent node ("Product Prompt Agent")**  
   - Text input:  
     ```
     About Product : {{ $json.description }}

     Text On Product: {{ $json.text }}

     Description: {{ $json.productOptions }}
     ```  
   - System Message: As per workflow, instructing generation of structured JSON with keys for product, aesthetics, composition, lighting, scene, text, lens, shot_type, camera_action.  
   - Enable output parser with manual JSON schema defining those keys.  
   - Set retry on fail.  
   - Connect Get the Data output to this node.

4. **Add a LangChain Output Parser node ("Structured Output Parser")**  
   - Use manual schema identical to agent’s expected output.  
   - Connect Product Prompt Agent output to this parser.

5. **Add HTTP Request node ("Nano Banana Image")**  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Authentication: OpenRouter API Credential  
   - Body (JSON): Use the template to construct AI prompt text dynamically with fields from Structured Output Parser output; sanitize quotes by replacing with `**`.  
   - Connect Structured Output Parser output to this node.

6. **Add Set node ("Get the Base64 String")**  
   - Extract base64 image data from the AI image URL in Nano Banana Image response (split by comma, take second part).  
   - Connect Nano Banana Image output.

7. **Add Convert to File node ("Convert to File")**  
   - Operation: toBinary  
   - Source Property: `image_string`  
   - File Name: template `nano-banana-{{ (Math.random() + 1).toString(36).substring(2,7) }}.png`  
   - Connect Get the Base64 String output.

8. **Add HTTP Request node ("Upload to MediaUpload")**  
   - Method: POST  
   - URL: `http://homelab-mediahosting.4m6m9w.easypanel.host/api/1/upload` (replace with your media uploader if needed)  
   - Authentication: HTTP Header Auth (Mediaupload easypanel credentials)  
   - Body: Multipart form-data with binary file input as `source`  
   - Connect Convert to File output.

9. **Add Set node ("Extract Image Url")**  
   - Extract uploaded image URL from Upload to MediaUpload response, replacing `http` with `https` for secure URLs.  
   - Connect Upload to MediaUpload output.

10. **Add Google Gemini Image Analysis node ("Product Visualiser")**  
    - Operation: Analyze  
    - Input: URL from Extract Image Url  
    - Model: Gemini 2.5 flash lite  
    - Purpose: generate concise product description for cataloging.  
    - Connect Extract Image Url output.

11. **Add LangChain Agent node ("Model Prompt Generator")**  
    - Text input: `Here is the Product : {{ $json.content.parts[0].text }}. Please generate an image prompt.`  
    - System message: Prompt to create a high-res image prompt.  
    - Output parser: manual JSON schema with key `prompt`.  
    - Connect Product Visualiser output.

12. **Add LangChain Output Parser node ("Structured Output Parser1")**  
    - Manual schema with single `prompt` string.  
    - Connect Model Prompt Generator output.

13. **Add HTTP Request node ("Generate Model with Product")**  
    - POST to OpenRouter API endpoint for Gemini 2.5 flash image preview.  
    - Body: JSON using prompt from Structured Output Parser1 and image URL from Extract Image Url.  
    - Connect Structured Output Parser1 output.

14. **Add Set node ("Get the Base64 String1")**  
    - Extract base64 image data from Generate Model with Product response.  
    - Connect Generate Model with Product output.

15. **Add Convert to File node ("Convert to File1")**  
    - Convert base64 to binary file as above.  
    - Connect Get the Base64 String1 output.

16. **Add HTTP Request node ("Upload to MediaUpload1")**  
    - Upload second image file similarly to step 8.  
    - Connect Convert to File1 output.

17. **Add Set node ("Extract Image Url1")**  
    - Extract second uploaded image URL.  
    - Connect Upload to MediaUpload1 output.

18. **Add Discord node ("Discord1")**  
    - Send message with product and UGC image URLs to Discord webhook.  
    - Connect Extract Image Url1 output.

19. **Add Google Gemini Image Analysis node ("Creative Visualiser")**  
    - Analyze second image URL for rich descriptive passage.  
    - Connect Extract Image Url1 output.

20. **Add LangChain Agent node ("Creative Director")**  
    - Input: creative visual description text.  
    - System message: detailed instructions to generate cinematic video shot prompt JSON with key `video_prompt`.  
    - Output parser: manual JSON schema with `video_prompt` key.  
    - Connect Creative Visualiser output.

21. **Add LangChain Output Parser node ("Structured Output Parser2")**  
    - Parse video prompt JSON.  
    - Connect Creative Director output.

22. **Add HTTP Request node ("Generate Video")**  
    - POST to GoAPI `/task` endpoint.  
    - Body: includes model `Qubico/wanx`, task type `wan22-img2video-14b`, with input prompt from Structured Output Parser2 and image URL from Extract Image Url1.  
    - Credentials: GoAPI HTTP Header Auth.  
    - Connect Structured Output Parser2 output.

23. **Add HTTP Request node ("HTTP Request")**  
    - GET request to GoAPI task status endpoint `/api/v1/task/{{ $json.data.task_id }}` to poll task status.  
    - Connect Generate Video output.

24. **Add If node ("If")**  
    - Condition: Check if status equals "processing" OR "pending".  
    - Connect HTTP Request output.

25. **Add Wait node ("Wait")**  
    - Delay before next status poll.  
    - Connect If node’s true output.

26. **Connect Wait output back to HTTP Request** for polling loop.

27. **Add HTTP Request node ("Download Video")**  
    - Downloads video from GoAPI output after completion.  
    - Connect If node’s false output.

28. **Add HTTP Request node ("Upload to MediaUpload2")**  
    - Upload downloaded video to media hosting.  
    - Connect Download Video output.

29. **Add HTML node ("Generate Response")**  
    - Create responsive HTML page showing product and UGC images with download links.  
    - Connect Discord1 output.

30. **Add Error Trigger node ("Error Trigger")**  
    - Global error catch.

31. **Add Discord node ("Send the Error")**  
    - Sends error alerts to Discord.  
    - Connect Error Trigger output.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                               |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Replace any media hosting HTTP upload endpoints with alternatives like [vgy.me](https://vgy.me) if needed. | Sticky notes on media upload nodes.                            |
| Error messages and debugging information are sent to Discord using a webhook for quick issue tracking.     | Error Trigger and Send the Error nodes.                        |
| The workflow uses Google Gemini models and OpenRouter API for AI-powered image and text generation.        | Requires proper credential setup for these APIs.              |
| Discord webhooks are used to send notifications of generated images for team or logging purposes.          | Credential: Discord Webhook API.                               |
| The HTML output node provides a simple product image showcase UI for end users.                            | Useful for integrating with web frontends or internal tools.  |
| The workflow includes multiple sticky notes grouping logical steps to assist understanding and maintenance.| Visible on canvas, e.g., prompt generation, base64 conversion. |

---

This documentation enables detailed understanding, customization, error anticipation, and full manual reproduction of the entire product image and video generation workflow using n8n.