Generate UGC videos from product images with Gemini and VEO3

https://n8nworkflows.xyz/workflows/generate-ugc-videos-from-product-images-with-gemini-and-veo3-8927


# Generate UGC videos from product images with Gemini and VEO3

### 1. Workflow Overview

This workflow automates the generation of User-Generated Content (UGC) style videos from product images sent via Telegram. It leverages advanced AI models (Google Gemini and Anthropic Claude) and the VEO3 video generation API to transform static images into photorealistic, scripted video content suitable for social media advertising.

The workflow is logically divided into these main blocks:

- **1.1 Initialization and Input Reception**  
  Handles Telegram bot integration to receive product images and user instructions.

- **1.2 Image Enhancement and AI Analysis**  
  Uses Google Gemini AI to analyze and enhance the received image, generating a photorealistic, square-format image optimized for video.

- **1.3 Script Generation**  
  Employs Claude AI to create a two-segment video script based on user input and detailed image analysis, with strict constraints to maintain realism.

- **1.4 Video Generation and Delivery**  
  Calls the VEO3 API to generate two video segments from the script and enhanced image, merges them into a single video, and sends the final video back to the user on Telegram.

- **1.5 Error Handling and Status Monitoring**  
  Implements waiting, polling, and switching logic to monitor video generation status and handle failures gracefully.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Input Reception

**Overview:**  
This block receives the product image uploaded by the user via Telegram, manages the Telegram bot token, and prompts the user for instructions on image enhancement.

**Nodes Involved:**  
- Telegram Trigger  
- Edit Fields  
- Send message and wait for response  
- Get a file  
- Send a photo message

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger node  
  - *Role:* Entry point that listens for incoming Telegram messages (images)  
  - *Configuration:* Listens to "message" updates; uses configured Telegram API credentials  
  - *Connections:* Output flows to "Edit Fields" node  
  - *Edge cases:* Missing or invalid Telegram token; user sends non-image message; Telegram API downtime

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Stores Telegram bot token as a workflow variable for reuse  
  - *Configuration:* Assigns a string variable "telegram_token" with the bot token value (placeholder "Your Telegram Token")  
  - *Connections:* Output to "Send message and wait for response"  
  - *Edge cases:* Telegram token missing or incorrect, causing API failures downstream

- **Send message and wait for response**  
  - *Type:* Telegram node (sendAndWait operation)  
  - *Role:* Sends a message confirming image receipt and asks user for enhancement instructions  
  - *Configuration:* Uses chat ID from Telegram Trigger; waits for free-text reply  
  - *Connections:* Output to "Get a file"  
  - *Edge cases:* User does not respond; Telegram API errors

- **Get a file**  
  - *Type:* Telegram node (get file operation)  
  - *Role:* Retrieves the image file from Telegram using the file ID extracted from the message  
  - *Configuration:* Targets the highest resolution photo variant (index 2); does not download binary data, only metadata  
  - *Connections:* Output to "HTTP Request" (Google Gemini API call)  
  - *Edge cases:* File ID missing; Telegram API errors; file path unavailable

- **Send a photo message**  
  - *Type:* Telegram node (sendPhoto operation)  
  - *Role:* Sends the enhanced image back to the user after conversion  
  - *Configuration:* Sends binary image data to the user's chat  
  - *Connections:* Input from "Convert to File" node  
  - *Edge cases:* Telegram API errors; binary data missing

---

#### 2.2 Image Enhancement and AI Analysis

**Overview:**  
This block sends the original image and user instructions to Google Gemini AI to generate a photorealistic, square-format enhanced image. The image is extracted from the AI response, converted to a file, and sent back to the user.

**Nodes Involved:**  
- HTTP Request (Google Gemini API for image generation)  
- Code  
- Convert to File  
- Send a photo message (from previous block)

**Node Details:**

- **HTTP Request (Google Gemini API)**  
  - *Type:* HTTP Request node  
  - *Role:* Queries OpenRouter API with Gemini 2.5 Flash Image Preview model, sending user instructions and image URL  
  - *Configuration:* POST request to `https://openrouter.ai/api/v1/chat/completions` with JSON body containing model, user messages including text instructions and image URL retrieved from Telegram file path  
  - *Authentication:* Predefined OpenRouter API credentials  
  - *Connections:* Output to "Code" node  
  - *Edge cases:* API rate limits; invalid or expired credentials; image URL inaccessible

- **Code**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Extracts the base64-encoded image URL from Gemini AI response and strips any data URI prefix  
  - *Key code:* Accesses `choices[0].message.images[0].image_url.url` and cleans the base64 string  
  - *Connections:* Output to "Convert to File"  
  - *Edge cases:* AI response format changes; missing or malformed image data

- **Convert to File**  
  - *Type:* ConvertToFile node  
  - *Role:* Converts base64 string into binary image file with PNG MIME type and file name "generated_image"  
  - *Connections:* Output to "Send a photo message" node  
  - *Edge cases:* Invalid base64 string; file conversion errors

---

#### 2.3 Script Generation

**Overview:**  
This block collects a dialogue idea from the user, analyzes the enhanced image using Gemini AI to describe image content, and then uses Claude AI to generate a realistic, two-segment UGC video script respecting strict constraints.

**Nodes Involved:**  
- Send message and wait for response1 (dialogue idea prompt)  
- Get a file1 (fetch enhanced image)  
- HTTP Request3 (Gemini AI image content description)  
- Anthropic Chat Model (Claude AI)  
- Basic LLM Chain (script generation with LangChain)  
- Structured Output Parser  
- Split Out

**Node Details:**

- **Send message and wait for response1**  
  - *Type:* Telegram node (sendAndWait)  
  - *Role:* Prompts user for an idea for video dialogue (e.g., testimonial or product benefits)  
  - *Connections:* Output to "Basic LLM Chain"  
  - *Edge cases:* No user reply; API errors

- **Get a file1**  
  - *Type:* Telegram node (get file)  
  - *Role:* Retrieves the enhanced image file from Telegram (highest resolution photo variant, index 3)  
  - *Connections:* Output to "HTTP Request3"  
  - *Edge cases:* File ID missing; Telegram API errors

- **HTTP Request3 (Gemini AI image description)**  
  - *Type:* HTTP Request node  
  - *Role:* Sends enhanced image to Gemini AI requesting a detailed description of image content (frame, subject, actions)  
  - *Configuration:* Similar to previous Gemini call but with a prompt for detailed image description  
  - *Connections:* Output to "Send message and wait for response1" and also feeds the description into "Basic LLM Chain"  
  - *Edge cases:* API errors; incorrect prompt response

- **Anthropic Chat Model (Claude AI)**  
  - *Type:* LangChain AI Chat node  
  - *Role:* Runs Claude 4 Sonnet model to generate a video script under strict constraints (no added objects, realistic actions)  
  - *Connections:* Output to "Basic LLM Chain" (LangChain chain node) as language model input  
  - *Edge cases:* API failures; timeout; output parsing errors

- **Basic LLM Chain**  
  - *Type:* LangChain chain node  
  - *Role:* Combines user dialogue idea and Gemini image description to generate a two-part script with detailed camera movements and natural dialogue  
  - *Key expressions:* Template prompt with constraints and format example; uses JSON output parser  
  - *Connections:* Output to "Split Out" for further processing  
  - *Edge cases:* Expression errors; incomplete output; malformed JSON

- **Structured Output Parser**  
  - *Type:* LangChain structured output parser node  
  - *Role:* Parses the two-segment JSON script output from the LLM chain to structured data with keys "segment-1" and "segment-2"  
  - *Connections:* Input to "Basic LLM Chain" (output parser)  
  - *Edge cases:* Parsing failures if output deviates from expected schema

- **Split Out**  
  - *Type:* SplitOut node  
  - *Role:* Splits the structured script output into individual segments for separate video generation  
  - *Connections:* Output to "HTTP Request1" to process each segment independently  
  - *Edge cases:* Missing or empty segments

---

#### 2.4 Video Generation and Delivery

**Overview:**  
This block calls the VEO3 API to generate video segments based on the scripts and enhanced image, waits for processing, polls for completion status, merges resulting videos into one, and sends the final video back to the user on Telegram.

**Nodes Involved:**  
- HTTP Request1 (VEO3 generate video)  
- Wait  
- HTTP Request2 (VEO3 check record-info status)  
- Switch  
- Aggregate  
- Merge video  
- Send a video  
- Send a text message (error message)

**Node Details:**

- **HTTP Request1 (VEO3 generate video)**  
  - *Type:* HTTP Request node  
  - *Role:* Sends POST request to `https://api.kie.ai/api/v1/veo/generate` with video prompt segment and image URL  
  - *Configuration:* Uses "veo3_fast" model, aspect ratio 16:9, fixed seed for reproducibility, disables fallback  
  - *Authentication:* Bearer token for KIE AI account  
  - *Connections:* Output to "Wait" node  
  - *Edge cases:* API limits; invalid prompt; network timeout

- **Wait**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow for 60 seconds to allow video generation processing  
  - *Connections:* Output to "HTTP Request2" (status check)  
  - *Edge cases:* Long wait times; premature timeout

- **HTTP Request2 (VEO3 check status)**  
  - *Type:* HTTP Request node  
  - *Role:* Queries video generation status using taskId from previous response  
  - *Configuration:* GET request with taskId query parameter; same Bearer token auth  
  - *Connections:* Output to "Switch" node  
  - *Edge cases:* API errors; taskId invalid or expired

- **Switch**  
  - *Type:* Switch node  
  - *Role:* Routes workflow based on status successFlag:  
    - Success (1): proceed to aggregation  
    - In Process (0): loop back to Wait node for another status check  
    - Else: trigger error message  
  - *Connections:* Outputs to "Aggregate", "Wait", or "Send a text message" respectively  
  - *Edge cases:* Unexpected status codes; infinite loops if status doesn't change

- **Aggregate**  
  - *Type:* Aggregate node  
  - *Role:* Collects all successful video URLs from multiple generated segments into a single array  
  - *Connections:* Output to "Merge video"  
  - *Edge cases:* Missing URLs; partial failures

- **Merge video**  
  - *Type:* mediaFX node  
  - *Role:* Merges video segments into one continuous video  
  - *Configuration:* Inputs are video URLs from aggregated results  
  - *Connections:* Output to "Send a video"  
  - *Edge cases:* Merge failures; incompatible video formats

- **Send a video**  
  - *Type:* Telegram node (sendVideo operation)  
  - *Role:* Sends the merged final video back to the user on Telegram  
  - *Connections:* No further outputs  
  - *Edge cases:* Telegram API errors; large video size limitations

- **Send a text message**  
  - *Type:* Telegram node  
  - *Role:* Sends an error message to user if video generation fails  
  - *Connections:* No further outputs  
  - *Edge cases:* Telegram API errors

---

### 3. Summary Table

| Node Name                     | Node Type                       | Functional Role                         | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                                                                                                                         |
|-------------------------------|--------------------------------|---------------------------------------|----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger              | telegramTrigger                | Receives images from user on Telegram | -                          | Edit Fields                 | # Phase 0: Initialization and Prerequisites: Config accounts and tokens for Telegram, OpenRouter, Anthropic, KIE.AI. Ensures automation ready.                                                                                      |
| Edit Fields                  | set                           | Holds Telegram bot token variable     | Telegram Trigger           | Send message and wait for response |                                                                                                                                                                                                                                     |
| Send message and wait for response | telegram                     | Prompts user for enhancement instructions | Edit Fields                | Get a file                  | # Phase 1: Image Reception and Enhancement: User sends image, system analyzes with Gemini, returns enhanced photorealistic image in square format.                                                                                   |
| Get a file                   | telegram                      | Retrieves uploaded image metadata     | Send message and wait for response | HTTP Request               |                                                                                                                                                                                                                                     |
| HTTP Request                 | httpRequest                   | Calls Gemini AI for image enhancement | Get a file                 | Code                       | ## Generate Image                                                                                                                                                                                                                    |
| Code                        | code                          | Extracts base64 image from AI response | HTTP Request               | Convert to File             |                                                                                                                                                                                                                                     |
| Convert to File              | convertToFile                 | Converts base64 string to binary file | Code                       | Send a photo message        |                                                                                                                                                                                                                                     |
| Send a photo message         | telegram                      | Sends enhanced image back to user     | Convert to File            | Get a file1                 |                                                                                                                                                                                                                                     |
| Get a file1                  | telegram                      | Retrieves enhanced image file          | Send a photo message       | HTTP Request3              |                                                                                                                                                                                                                                     |
| HTTP Request3                | httpRequest                   | Calls Gemini AI for detailed image description | Get a file1                | Send message and wait for response1 | ## Video Generation Prompts                                                                                                                                                                                                         |
| Send message and wait for response1 | telegram                     | Prompts user for video dialogue idea  | HTTP Request3              | Basic LLM Chain             | # Phase 2: Analysis and Script Creation: User gives dialogue idea; system analyzes image & user idea; generates 2-part video script with Claude AI respecting realism constraints.                                                    |
| Anthropic Chat Model         | langchain.lmChatAnthropic     | Runs Claude AI to generate video script | Basic LLM Chain            | Basic LLM Chain             |                                                                                                                                                                                                                                     |
| Basic LLM Chain              | langchain.chainLlm            | Generates two-segment UGC video script | Send message and wait for response1, HTTP Request3 | Split Out           |                                                                                                                                                                                                                                     |
| Structured Output Parser     | langchain.outputParserStructured | Parses JSON script output             | Basic LLM Chain            | Basic LLM Chain             |                                                                                                                                                                                                                                     |
| Split Out                   | splitOut                     | Splits script into segments            | Basic LLM Chain            | HTTP Request1              |                                                                                                                                                                                                                                     |
| HTTP Request1               | httpRequest                   | Sends video generation request to VEO3 | Split Out                 | Wait                       | # Phase 3: Video Generation: Calls VEO3 for video segments, waits, polls status, merges videos, sends final video back on Telegram.                                                                                                |
| Wait                       | wait                         | Pauses workflow for video generation  | HTTP Request1              | HTTP Request2              |                                                                                                                                                                                                                                     |
| HTTP Request2               | httpRequest                   | Polls VEO3 for video generation status | Wait                      | Switch                     |                                                                                                                                                                                                                                     |
| Switch                     | switch                       | Routes based on generation status      | HTTP Request2              | Aggregate, Wait, Send a text message |                                                                                                                                                                                                                                     |
| Aggregate                  | aggregate                    | Aggregates video segment URLs          | Switch (Success output)    | Merge video                |                                                                                                                                                                                                                                     |
| Merge video                | mediaFX                      | Merges video segments into one         | Aggregate                  | Send a video               |                                                                                                                                                                                                                                     |
| Send a video               | telegram                     | Sends final merged video to user       | Merge video                | -                          |                                                                                                                                                                                                                                     |
| Send a text message        | telegram                     | Sends error message if video generation fails | Switch (failure output)    | -                          |                                                                                                                                                                                                                                     |
| Sticky Notes (various)     | stickyNote                   | Document phases, instructions, and credits | -                          | -                          | See specific sticky note content below in General Notes & Resources                                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for 'message' updates  
   - Credentials: Telegram API with bot token  
   - Position: Entry node  

2. **Create Edit Fields node**  
   - Type: Set  
   - Assign string variable "telegram_token" with your Telegram bot token  
   - Connect from Telegram Trigger  

3. **Create Send message and wait for response node**  
   - Type: Telegram (sendAndWait operation)  
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Message: "Merci d'avoir téléchargé l'image. Veuillez fournir vos instructions."  
   - Credentials: Telegram API  
   - Connect from Edit Fields  

4. **Create Get a file node**  
   - Type: Telegram (get file)  
   - File ID: `={{ $('Telegram Trigger').item.json.message.photo[2].file_id }}` (highest res photo)  
   - Download: false  
   - Credentials: Telegram API  
   - Connect from Send message and wait for response  

5. **Create HTTP Request node for Gemini AI image enhancement**  
   - Type: HTTP Request (POST)  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Body (JSON): Model "google/gemini-2.5-flash-image-preview" with user instructions and image URL constructed from Telegram file path and telegram_token variable  
   - Authentication: OpenRouter API credentials  
   - Connect from Get a file  

6. **Create Code node**  
   - Extract base64 image URL from Gemini response (`choices[0].message.images[0].image_url.url`)  
   - Remove 'data:image/...' prefix if present  
   - Return JSON with property "base64_data"  
   - Connect from HTTP Request  

7. **Create Convert to File node**  
   - Operation: toBinary  
   - Source property: "base64_data"  
   - File Name: "generated_image.png"  
   - MIME Type: image/png  
   - Connect from Code  

8. **Create Send a photo message node**  
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Operation: sendPhoto  
   - Send binary image data from Convert to File  
   - Credentials: Telegram API  
   - Connect from Convert to File  

9. **Create Get a file1 node**  
   - Type: Telegram (get file)  
   - File ID: `={{ $json.result.photo[3].file_id }}` (highest res for enhanced image)  
   - Credentials: Telegram API  
   - Connect from Send a photo message  

10. **Create HTTP Request3 node for Gemini AI image description**  
    - Similar to earlier Gemini call but prompt for detailed image description  
    - Use enhanced image URL from Telegram file path and telegram_token  
    - Authentication: OpenRouter API  
    - Connect from Get a file1  

11. **Create Send message and wait for response1 node**  
    - Telegram sendAndWait node  
    - Message: "Veuillez fournir une idée de dialogue pour votre annonce."  
    - Chat ID: from Telegram Trigger  
    - Credentials: Telegram API  
    - Connect from HTTP Request3  

12. **Create Anthropic Chat Model node**  
    - Model: "claude-sonnet-4-20250514" (Claude 4 Sonnet)  
    - Credentials: Anthropic API  
    - Connect as AI language model input to Basic LLM Chain  

13. **Create Structured Output Parser node**  
    - JSON schema example with keys "segment-1" and "segment-2"  
    - Connect as output parser to Basic LLM Chain  

14. **Create Basic LLM Chain node**  
    - Prompt combines user dialogue idea and Gemini image description  
    - Constraints for realistic, natural UGC video script in two 7-8 second segments  
    - Connect from Send message and wait for response1 and Anthropic Chat Model  
    - Output to Split Out  

15. **Create Split Out node**  
    - Field to split: "output" (the parsed video script segments)  
    - Connect from Basic LLM Chain  

16. **Create HTTP Request1 node to call VEO3 video generation**  
    - POST to `https://api.kie.ai/api/v1/veo/generate`  
    - JSON body includes prompt segment, enhanced image URL, model "veo3_fast", aspect ratio "16:9", seed 12345, fallback disabled  
    - Authentication: Bearer token for KIE AI  
    - Connect from Split Out  

17. **Create Wait node**  
    - Wait for 60 seconds before status check  
    - Connect from HTTP Request1  

18. **Create HTTP Request2 node for VEO3 status check**  
    - GET to `https://api.kie.ai/api/v1/veo/record-info` with query param `taskId` from previous response  
    - Authentication: Bearer token for KIE AI  
    - Connect from Wait  

19. **Create Switch node**  
    - Condition on `data.successFlag`:  
      - 1 (Success) → Aggregate node  
      - 0 (In Process) → loop back to Wait node  
      - Else → Send a text message (error)  
    - Connect from HTTP Request2  

20. **Create Aggregate node**  
    - Aggregate `data.response.resultUrls` fields into an array  
    - Connect from Switch (Success output)  

21. **Create Merge video node**  
    - mediaFX node  
    - Input video URLs from Aggregate output  
    - Connect from Aggregate  

22. **Create Send a video node**  
    - Telegram sendVideo operation  
    - Chat ID from Telegram Trigger  
    - Send binary merged video  
    - Credentials: Telegram API  
    - Connect from Merge video  

23. **Create Send a text message node**  
    - Telegram node to send error message "La génération de la vidéo a échoué"  
    - Chat ID from Telegram Trigger  
    - Credentials: Telegram API  
    - Connect from Switch (failure output)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| # Phase 0: Initialization and Prerequisites - Configure Telegram Bot, OpenRouter, Anthropic, KIE.AI credentials and webhook URLs          | Sticky Note7, initial setup instructions                                                                                                |
| # Phase 1: Image Reception and Enhancement - User sends image, AI optimizes and returns photorealistic square image                       | Sticky Note4                                                                                                                            |
| # Phase 2: Analysis and Script Creation - Combines user dialogue input with AI image analysis to create realistic 2-part UGC video script | Sticky Note5                                                                                                                            |
| # Phase 3: Video Generation - Automatically generates, checks, merges, and delivers final video with error handling                       | Sticky Note6                                                                                                                            |
| Nano Banana 🍌 + VEO3 Fast UGC Generator by Growth Ai                                                                                    | Sticky Note3                                                                                                                            |
| Contact for advanced automation solutions at Growth-AI.fr                                                                                | https://www.linkedin.com/in/allanvaccarizi/, https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/ (Sticky Note13)           |
| Workflow uses Google Gemini 2.5 Flash Image Preview model and Anthropic Claude 4 Sonnet for AI processing                                  | API models referenced in HTTP Request nodes and Anthropic Chat Model node                                                                |
| Video generated is optimized for vertical social media formats (9:16 aspect ratio)                                                        | VEO3 API configuration in HTTP Request1 node                                                                                            |

---

**Disclaimer:** The source text is exclusively derived from an automated n8n workflow and complies strictly with content policies. It contains only legal and public data with no illegal, offensive, or protected elements.