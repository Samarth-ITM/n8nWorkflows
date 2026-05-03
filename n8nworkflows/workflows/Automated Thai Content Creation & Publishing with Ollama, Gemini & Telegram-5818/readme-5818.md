Automated Thai Content Creation & Publishing with Ollama, Gemini & Telegram

https://n8nworkflows.xyz/workflows/automated-thai-content-creation---publishing-with-ollama--gemini---telegram-5818


# Automated Thai Content Creation & Publishing with Ollama, Gemini & Telegram

---

### 1. Workflow Overview

This workflow automates the process of creating, enriching, and publishing Thai-language content leveraging AI models and various APIs. It is designed for content marketers, SEO specialists, and social media managers who want to generate articles, enhance them with SEO metadata, create appropriate images, and publish to a website and Telegram automatically.

The workflow comprises these main logical blocks:

- **1.1 Input Reception:** Fetches new news items from Google Sheets to trigger the content creation cycle.
- **1.2 Image Type Analysis:** Uses AI to decide if the image for the article should be a real photo or AI-generated, then fetches or generates the image accordingly.
- **1.3 Article Composition:** Generates a high-quality Thai article using AI, selecting the article format adaptively.
- **1.4 SEO & Marketing Metadata Generation:** Creates SEO-friendly titles, excerpts, and tags to accompany the article.
- **1.5 Post Preparation and Publishing:** Prepares the combined post data, uploads the content and images to a website via HTTP API, and sends a Telegram notification about the new article.
- **1.6 Image Conversion and Handling:** Converts image data between different formats and attaches it to the post payload.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow whenever a new news item is added or updated in a Google Sheets spreadsheet. It serves as the entry point, feeding fresh content for processing.

- **Nodes Involved:**  
  - Google Sheets: Read News  
  - LLM: Image Type Analyzer

- **Node Details:**

  - **Google Sheets: Read News**  
    - *Type:* Google Sheets Trigger  
    - *Role:* Watches a specified Google Sheet for new or changed rows to start the workflow.  
    - *Configuration:* Polls every minute; sheet and document IDs configured via credentials.  
    - *Inputs:* None (trigger)  
    - *Outputs:* JSON containing news description and metadata.  
    - *Potential failures:* OAuth2 auth errors, API rate limits, sheet unavailability.

  - **LLM: Image Type Analyzer (LangChain chainLlm node)**  
    - *Type:* AI language model chain node  
    - *Role:* Decides if the article should use a real image or AI-generated image based on news content, returning a type and keywords if real image.  
    - *Configuration:* Uses a strict instruction prompt to output only "ภาพจริง" or "ภาพ AI" plus keywords.  
    - *Inputs:* News description text from Google Sheets.  
    - *Outputs:* JSON with `"type"` ("ภาพจริง" or "ภาพ AI") and `"keywords"` array.  
    - *Failures:* AI model timeouts, malformed response, empty input text.

#### 2.2 Image Type Analysis & Image Retrieval

- **Overview:**  
  Based on the image type determined, this block fetches either a real image from a local server or generates an AI image using Google Gemini. It also converts and prepares image data for downstream usage.

- **Nodes Involved:**  
  - If: Is AI Image?  
  - Merge Image Data  
  - If: Is Real Image?  
  - HTTP Request: Fetch Real Image  
  - Read/Write Files from Disk (Local)  
  - Extract Image Binary Data  
  - Convert Extracted Image to File  
  - Function: Attach Image Binary  
  - Google Gemini: Generate Photo  
  - Convert AI Image to File  
  - Google Gemini: Generate Risoprint Image  
  - Convert AI Risoprint Image to File

- **Node Details:**

  - **If: Is AI Image?**  
    - *Type:* Conditional branch node  
    - *Role:* Checks if image type is "ภาพ AI".  
    - *Input:* Output of LLM Image Type Analyzer.  
    - *Outputs:* True branch proceeds to AI image generation; false branch leads to real image acquisition.

  - **Merge Image Data**  
    - *Type:* Merge  
    - *Role:* Combines multiple data streams for image and article processing.  
    - *Inputs:* Receives from AI image generation and real image processing branches.  
    - *Outputs:* Feeds merged data to article writing node.

  - **If: Is Real Image?**  
    - *Type:* Conditional branch node  
    - *Role:* Checks if image type is "ภาพจริง".  
    - *Input:* Output from Function: Prepare Post Data.  
    - *Outputs:* Determines next image handling step.

  - **HTTP Request: Fetch Real Image**  
    - *Type:* HTTP request  
    - *Role:* Posts keywords to local image server API to fetch a real image.  
    - *Configuration:* Uses JSON body containing keywords extracted by analyzer.  
    - *Inputs:* Keywords from Function: Image Type Analyzer.  
    - *Outputs:* Image file data on disk.

  - **Read/Write Files from Disk (Local)**  
    - *Type:* File system access  
    - *Role:* Reads image files from a local directory path matching image extensions.  
    - *Inputs:* None (triggered after HTTP request).  
    - *Outputs:* Binary image data.

  - **Extract Image Binary Data**  
    - *Type:* Extract from file  
    - *Role:* Converts file binary data to JSON property for further processing.  
    - *Inputs:* Binary image file from disk.  
    - *Outputs:* JSON with base64 or binary image data.

  - **Convert Extracted Image to File**  
    - *Type:* Convert to file  
    - *Role:* Converts extracted binary data back to a file-type binary property for upload.  
    - *Inputs:* JSON with binary image data.  
    - *Outputs:* Binary file named "generated-image.png".

  - **Function: Attach Image Binary**  
    - *Type:* Function (code)  
    - *Role:* Attaches the binary image data to the JSON payload for downstream merging.  
    - *Inputs:* Items with binary "generated-image.png".  
    - *Outputs:* Items with image binary added as JSON property `imageBinaryData`.

  - **Google Gemini: Generate Photo**  
    - *Type:* HTTP Request to Google Gemini API  
    - *Role:* Generates photorealistic image based on AI image prompt.  
    - *Configuration:* Sends JSON body with detailed prompt including "thai-face" theme, HDR, cinematic style.  
    - *Inputs:* Image prompt from parsed LLM output.  
    - *Outputs:* JSON response with base64 image data.

  - **Convert AI Image to File**  
    - *Type:* Convert to file  
    - *Role:* Converts Gemini API base64 image data into binary file for upload.  
    - *Inputs:* Gemini API response.  
    - *Outputs:* Binary file "generated-image.png".

  - **Google Gemini: Generate Risoprint Image**  
    - *Type:* HTTP Request to Google Gemini API  
    - *Role:* Applies risoprint effect to an existing image (binary data sent inline).  
    - *Inputs:* Image binary attached from previous nodes.  
    - *Outputs:* JSON with transformed image data.

  - **Convert AI Risoprint Image to File**  
    - *Type:* Convert to file  
    - *Role:* Converts risoprint style image data to binary file.  
    - *Inputs:* Gemini risoprint API response.  
    - *Outputs:* Binary file for publishing.

#### 2.3 Article Composition

- **Overview:**  
  Generates a structured, high-quality Thai article based on input description, automatically deciding on article format (list or analytical).

- **Nodes Involved:**  
  - Ollama: Article Writer Model  
  - LLM: Article Writer

- **Node Details:**

  - **Ollama: Article Writer Model**  
    - *Type:* LangChain Ollama AI model node  
    - *Role:* Provides base language model for article writing.  
    - *Configuration:* Uses latest llama3 model.  
    - *Inputs:* None directly (used by LLM chain node).  
    - *Outputs:* Model interface for text generation.

  - **LLM: Article Writer**  
    - *Type:* LangChain chain LLM node  
    - *Role:* Executes AI prompt to write a long-form article in Thai, adapting format based on topic analysis.  
    - *Configuration:*  
      - Instruction includes strict rules for flawless Thai, no titles or labels in output, minimum 800 words.  
      - Input uses news description from Google Sheets.  
    - *Inputs:* News description and AI language model node.  
    - *Outputs:* JSON with generated article text.  
    - *Failures:* Model timeouts, malformed or incomplete text, language errors.

#### 2.4 SEO & Marketing Metadata Generation

- **Overview:**  
  Generates SEO-friendly title, excerpt, tags, and image prompt metadata to complement the article.

- **Nodes Involved:**  
  - Ollama: SEO & Marketing Model  
  - LLM: SEO & Marketing  
  - Function: Parse LLM Output

- **Node Details:**

  - **Ollama: SEO & Marketing Model**  
    - *Type:* LangChain Ollama AI model node  
    - *Role:* Provides base model for SEO & marketing metadata generation.  
    - *Configuration:* Uses llama3 latest model.  
    - *Inputs:* None directly (used by LLM chain node).  
    - *Outputs:* Model interface.

  - **LLM: SEO & Marketing**  
    - *Type:* LangChain chain LLM node  
    - *Role:* Generates structured data including title, excerpt, tags (two sets), and image prompt in strict output format.  
    - *Configuration:*  
      - Prompts instruct only the structured output between defined markers.  
      - Input is the article text from LLM Article Writer.  
    - *Inputs:* Article text.  
    - *Outputs:* Raw structured text with metadata fields.

  - **Function: Parse LLM Output**  
    - *Type:* Code Function  
    - *Role:* Parses the structured text output from the SEO & Marketing LLM to extract fields: TITLE, EXCERPT, TAGS, TAGS2, IMAGE_PROMPT.  
    - *Key logic:* Uses regex to extract fields between markers; cleans and formats tags arrays; ensures tags2 have leading ‘#’.  
    - *Inputs:* Raw text from LLM SEO & Marketing node.  
    - *Outputs:* JSON with clean fields: postTitle, postContent, tags1, tags2, imagePrompt.  
    - *Failures:* Malformed LLM output, missing markers, regex mismatch.

#### 2.5 Post Preparation and Publishing

- **Overview:**  
  Prepares the final post payload combining article, metadata, and image, then uploads it to the website and sends a Telegram notification.

- **Nodes Involved:**  
  - Function: Prepare Post Data  
  - If: Is Real Image? (second usage)  
  - HTTP Request: Post to Website  
  - Telegram: Send Notification

- **Node Details:**

  - **Function: Prepare Post Data**  
    - *Type:* Code Function  
    - *Role:* Combines parsed LLM output, article text, and image binary into a JSON payload for the website API. Applies default values for user ID, username, background color, affiliate links, post type, embedded content if not present.  
    - *Inputs:* Parsed LLM output, image binary data, article text.  
    - *Outputs:* JSON with all necessary fields and binary image data.  
    - *Failures:* Missing upstream data, missing binary image data.

  - **If: Is Real Image? (second instance)**  
    - *Type:* Conditional node  
    - *Role:* Determines image processing route for final post.  
    - *Inputs:* Output from Prepare Post Data.  
    - *Outputs:* Branches to either direct AI image upload or risoprint style image generation.

  - **HTTP Request: Post to Website**  
    - *Type:* HTTP Request  
    - *Role:* Sends multipart/form-data POST request to website API with all post data and image binary.  
    - *Configuration:*  
      - URL and API key from credentials/connections.  
      - Body includes user info, post title, content with appended tags, background color, affiliate links, embedded content, and image file binary.  
    - *Inputs:* Prepared post data with attached image binary file.  
    - *Outputs:* API response.  
    - *Failures:* Network errors, authentication failure, malformed request, server errors.

  - **Telegram: Send Notification**  
    - *Type:* Telegram node  
    - *Role:* Sends a chat message notifying about the new article publication, including title, summary, and tags.  
    - *Inputs:* Parsed LLM output fields.  
    - *Configuration:* Chat ID and credentials from connections.  
    - *Failures:* Telegram API rate limits, invalid chatId.

---

### 3. Summary Table

| Node Name                      | Node Type                                       | Functional Role                              | Input Node(s)                      | Output Node(s)                        | Sticky Note                           |
|--------------------------------|------------------------------------------------|----------------------------------------------|-----------------------------------|-------------------------------------|-------------------------------------|
| Google Sheets: Read News        | Google Sheets Trigger                           | Trigger workflow with new news data           | None                              | LLM: Image Type Analyzer             |                                     |
| LLM: Image Type Analyzer        | LangChain chain LLM                             | Analyze news to determine image type          | Google Sheets: Read News           | If: Is AI Image?                    |                                     |
| If: Is AI Image?                | If Node                                        | Branch based on image type being AI            | LLM: Image Type Analyzer           | Merge Image Data / Function: Image Type Analyzer |                                     |
| Merge Image Data                | Merge Node                                     | Combine image and article data                  | Google Gemini: Generate Risoprint Image, Function: Attach Image Binary | LLM: Article Writer                 |                                     |
| Function: Image Type Analyzer   | Code Function                                  | Parse LLM output to detect image type & keywords | LLM: Image Type Analyzer (second) | HTTP Request: Fetch Real Image, Merge Image Data |                                     |
| HTTP Request: Fetch Real Image  | HTTP Request                                   | Fetch real image from local image server       | Function: Image Type Analyzer       | Read/Write Files from Disk (Local)  |                                     |
| Read/Write Files from Disk (Local) | Read/Write Files                            | Read image files from disk                       | HTTP Request: Fetch Real Image     | Extract Image Binary Data            |                                     |
| Extract Image Binary Data       | Extract From File                              | Extract binary from image file                   | Read/Write Files from Disk (Local) | Convert Extracted Image to File      |                                     |
| Convert Extracted Image to File | Convert To File                                | Convert extracted image data to file binary     | Extract Image Binary Data          | Function: Attach Image Binary        |                                     |
| Function: Attach Image Binary   | Code Function                                  | Attach binary image data to JSON payload        | Convert Extracted Image to File    | Merge Image Data                    |                                     |
| Google Gemini: Generate Photo   | HTTP Request                                   | Generate photorealistic AI image via Gemini    | If: Is Real Image? (true branch)  | Convert AI Image to File             |                                     |
| Convert AI Image to File        | Convert To File                                | Convert Gemini image response to file binary    | Google Gemini: Generate Photo      | HTTP Request: Post to Website        |                                     |
| Google Gemini: Generate Risoprint Image | HTTP Request                             | Generate risoprint style image via Gemini       | If: Is Real Image? (false branch) | Convert AI Risoprint Image to File   |                                     |
| Convert AI Risoprint Image to File | Convert To File                            | Convert risoprint image data to file binary     | Google Gemini: Generate Risoprint Image | HTTP Request: Post to Website    |                                     |
| Ollama: Article Writer Model    | LangChain Ollama Model                         | Base AI model for article writing               | None                             | LLM: Article Writer                 |                                     |
| LLM: Article Writer             | LangChain chain LLM                            | Write long-form Thai article                      | Ollama: Article Writer Model      | LLM: SEO & Marketing                |                                     |
| Ollama: SEO & Marketing Model   | LangChain Ollama Model                         | Base AI model for SEO metadata generation        | None                             | LLM: SEO & Marketing               |                                     |
| LLM: SEO & Marketing            | LangChain chain LLM                            | Generate SEO title, excerpt, tags, image prompt | LLM: Article Writer               | Function: Parse LLM Output          |                                     |
| Function: Parse LLM Output      | Code Function                                  | Parse structured SEO & marketing data from text | LLM: SEO & Marketing              | Function: Prepare Post Data          |                                     |
| Function: Prepare Post Data     | Code Function                                  | Combine article, SEO data, image binary for posting | Function: Parse LLM Output, Merge Image Data | If: Is Real Image? (second)     |                                     |
| If: Is Real Image? (second)     | If Node                                        | Branch for final image processing before posting | Function: Prepare Post Data        | Google Gemini: Generate Photo, Google Gemini: Generate Risoprint Image |                                     |
| HTTP Request: Post to Website   | HTTP Request                                   | Upload final post data and image to website API | Convert AI Image to File, Convert AI Risoprint Image to File | Telegram: Send Notification     |                                     |
| Telegram: Send Notification     | Telegram                                       | Notify Telegram channel of newly published article | HTTP Request: Post to Website     | None                               |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node:**  
   - Type: Google Sheets Trigger  
   - Poll every minute  
   - Configure with Google Sheets OAuth2 credentials  
   - Specify Sheet ID and Sheet name to monitor for new rows.

2. **Add Ollama (or LangChain) LLM Node for Image Type Analysis:**  
   - Model: llama3:latest  
   - Prompt: Instruction to decide image type ("ภาพจริง" or "ภาพ AI") based on news content, outputting keywords for real images.  
   - Input: News description from Google Sheets node.

3. **Add If Node "Is AI Image?" to branch based on `"type"` output from the previous node:**  
   - Condition: Check if output type equals "ภาพ AI".  
   - True branch: Proceed to Merge node for AI image.  
   - False branch: Proceed to real image fetching.

4. **Create Function Node "Function: Image Type Analyzer":**  
   - Code to parse multi-line LLM output to extract image type and keywords.  
   - Input: Raw LLM output text.  
   - Output: JSON with `"type"` and `"keywords"` array.

5. **Add HTTP Request Node "HTTP Request: Fetch Real Image":**  
   - POST to local image server URL (configured via credentials).  
   - JSON body includes keywords from previous function node.  
   - Content-Type: application/json.

6. **Add Read/Write Files from Disk Node:**  
   - Reads image files from local directory with specified image extensions.

7. **Add Extract Image Binary Data Node:**  
   - Extracts binary data from read files.

8. **Add Convert Extracted Image to File Node:**  
   - Converts JSON binary data back to file-type binary property named "generated-image.png".

9. **Add Function Node "Function: Attach Image Binary":**  
   - Attaches binary image file to JSON payload as `imageBinaryData`.

10. **Add Merge Node "Merge Image Data":**  
    - Combine image binary data with article data.

11. **Add Ollama LLM Node "Ollama: Article Writer Model":**  
    - Use llama3:latest model.

12. **Add LangChain Chain LLM Node "LLM: Article Writer":**  
    - Prompt for writing high-quality Thai article of min 800 words, adaptive format based on topic (list or analytical).  
    - Input: News description from Google Sheets.

13. **Connect Merge Image Data to LLM Article Writer Node.**

14. **Add Ollama LLM Node "Ollama: SEO & Marketing Model":**  
    - llama3:latest model for metadata generation.

15. **Add LangChain Chain LLM Node "LLM: SEO & Marketing":**  
    - Prompt to generate structured SEO data: title, excerpt, tags, image prompt.  
    - Input: Article text from LLM Article Writer.

16. **Add Function Node "Function: Parse LLM Output":**  
    - Parse structured text output to extract fields: TITLE, EXCERPT, TAGS, TAGS2, IMAGE_PROMPT.  
    - Clean tags arrays and format properly.

17. **Add Function Node "Function: Prepare Post Data":**  
    - Combine parsed SEO data, article text, and image binary data into final JSON payload for posting.  
    - Use default values for user_id, username, background color, linkAff, postType, and embeddedContent if missing.

18. **Add If Node "If: Is Real Image?" (second usage):**  
    - Checks if image type is "ภาพจริง".  
    - True branch: Call Google Gemini to generate photorealistic image.  
    - False branch: Call Google Gemini to generate risoprint style image.

19. **Add HTTP Request Node "Google Gemini: Generate Photo":**  
    - POST to Gemini API for photo generation.  
    - Use image prompt from parsed LLM output.

20. **Add Convert To File Node "Convert AI Image to File":**  
    - Convert Gemini response image base64 data to binary file.

21. **Add HTTP Request Node "Google Gemini: Generate Risoprint Image":**  
    - POST to Gemini API for risoprint effect, sending binary image inline.

22. **Add Convert To File Node "Convert AI Risoprint Image to File":**  
    - Convert risoprint image base64 data to file binary.

23. **Add HTTP Request Node "HTTP Request: Post to Website":**  
    - POST multipart/form-data to website API endpoint with final post data and image binary file.  
    - Use API key from credentials.

24. **Add Telegram Node "Telegram: Send Notification":**  
    - Send message to Telegram chat notifying about new article.  
    - Use chat ID and credentials.

25. **Connect all nodes following described data flow connections to ensure proper data propagation.**

26. **Configure all credentials:**  
    - Ollama API account  
    - Google Gemini API key  
    - Google Sheets OAuth2  
    - Website API key and URL with user info  
    - Telegram OAuth2 and chat ID  
    - Local image server URL for real images

27. **Set default parameters where applicable, e.g., default user_id, username, background color (#FFFFFF), post type ('General').**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow relies on the latest llama3 model on Ollama and Google Gemini APIs for AI generation tasks.                                                        | Ollama & Google Gemini API documentation                                                                         |
| The AI prompts are strictly designed to output structured data with clear markers to enable robust parsing downstream.                                           | Strict prompt engineering improves parsing reliability                                                           |
| Telegram notifications include article title, summary, and tags to provide instant updates to the audience.                                                      | Telegram Bot API                                                                                                |
| Local image server integration allows fetching real images based on keywords, enabling hybrid AI-real image workflows.                                           | Requires accessible local image server API                                                                       |
| The risoprint image effect adds artistic style to images to diversify content presentation.                                                                       | Google Gemini's image style generation                                                                            |
| Default values for user ID, username, and post metadata can be customized via credentials or workflow parameters to adapt to different deployment environments. | Workflow flexibility                                                                                               |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a no-code automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---