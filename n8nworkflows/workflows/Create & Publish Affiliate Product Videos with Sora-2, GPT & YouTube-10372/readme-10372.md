Create & Publish Affiliate Product Videos with Sora-2, GPT & YouTube

https://n8nworkflows.xyz/workflows/create---publish-affiliate-product-videos-with-sora-2--gpt---youtube-10372


# Create & Publish Affiliate Product Videos with Sora-2, GPT & YouTube

### 1. Workflow Overview

This workflow automates the creation and publishing of affiliate product videos using a combination of Sora-2 AI agent, OpenAI GPT models, and YouTube integration. It targets affiliate marketers or digital content creators who want to streamline generating video content from product data and publish directly to YouTube channels with enriched metadata.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Data Retrieval**  
  Periodically triggers the workflow to fetch affiliate product rows from a Google Sheet, filtering active partnerships and unpublished products.

- **1.2 Product Data Acquisition and Validation**  
  Retrieves detailed product information from an external website API and validates the data, handling errors or missing content.

- **1.3 AI Content Generation for Video Creation**  
  Uses OpenAI GPT and a Sora-2 LangChain agent to generate prompts and scripts for video creation based on product details.

- **1.4 Video Generation and Processing**  
  Sends the prompt to a Text-to-Video API (Sora2), waits for video creation completion, downloads the video, and uploads it to YouTube.

- **1.5 YouTube Metadata Enhancement and Sheet Update**  
  Generates optimized YouTube video metadata with AI, updates the video info on YouTube, and marks the product as published in the Google Sheet.

- **1.6 User Interaction via Telegram**  
  Sends messages and waits for user responses on Telegram for possible manual input or confirmation during the metadata generation phase.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Scheduled Data Retrieval

**Overview:**  
Triggers the workflow periodically, reads product rows from a Google Sheet, and filters for active partnerships and products not yet published.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet (Google Sheets)  
- Partnership Active and Not Published (Filter)  
- Get Product Details (Set)  
- Limit

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow on a timer (no specific parameters shown)  
  - Inputs: None  
  - Outputs: To "Get row(s) in sheet"  
  - Edge cases: Missed triggers if n8n instance down, time zone mismatches

- **Get row(s) in sheet (Google Sheets)**  
  - Type: Google Sheets node  
  - Role: Retrieves rows containing affiliate product data  
  - Configuration: Spreadsheet and sheet identifiers expected (not specified)  
  - Inputs: From Schedule Trigger  
  - Outputs: To Partnership Active and Not Published filter  
  - Edge cases: API quota exceeded, authentication failure, empty sheet

- **Partnership Active and Not Published (Filter)**  
  - Type: Filter node  
  - Role: Filters rows where partnerships are active and videos not published  
  - Inputs: From Google Sheets node  
  - Outputs: To Get Product Details  
  - Edge cases: Expression failures if fields missing or malformed data

- **Get Product Details (Set)**  
  - Type: Set node  
  - Role: Prepares or enriches data for next steps (e.g., shaping data structure)  
  - Inputs: Filter output  
  - Outputs: To Limit node  
  - Edge cases: Data misalignment or missing required fields

- **Limit**  
  - Type: Limit node  
  - Role: Limits the number of items processed per run (throttling)  
  - Inputs: From Set node  
  - Outputs: To "Get Product Details from Website"  
  - Edge cases: If limit too low, may delay processing; if too high, overload downstream nodes

---

#### 1.2 Product Data Acquisition and Validation

**Overview:**  
Fetches detailed product data from an external website and validates it. Updates the Google Sheet if errors occur.

**Nodes Involved:**  
- Get Product Details from Website (HTTP Request)  
- If (Conditional)  
- Update about Error (Google Sheets)  
- Parse Product Data (Code)  
- Sora2 Prompt Generator (LangChain Agent)

**Node Details:**

- **Get Product Details from Website**  
  - Type: HTTP Request node  
  - Role: Fetch product details from an external API or website  
  - Configuration: URL and request parameters (not specified), "continue on error" enabled to avoid halting workflow  
  - Inputs: From Limit node  
  - Outputs: To If node (on success) and Update about Error node (on failure)  
  - Edge cases: Network timeouts, invalid JSON, HTTP errors, rate limiting

- **If**  
  - Type: If node  
  - Role: Checks if product data retrieval was successful and content valid  
  - Inputs: HTTP response data  
  - Outputs: To Parse Product Data (if true) or Update about Error (if false)  
  - Edge cases: Expression errors if data keys missing

- **Update about Error**  
  - Type: Google Sheets node  
  - Role: Logs errors or update status back in the sheet for tracking  
  - Inputs: From HTTP Request or If node (on failure branch)  
  - Outputs: None (ends branch)  
  - Edge cases: Google Sheets API issues, row update conflicts

- **Parse Product Data**  
  - Type: Code node  
  - Role: Parses and formats raw product data into structured format for AI input  
  - Inputs: From If node (success branch)  
  - Outputs: To Sora2 Prompt Generator  
  - Edge cases: Code errors, malformed JSON, unexpected data structure

- **Sora2 Prompt Generator**  
  - Type: LangChain Agent node (AI agent)  
  - Role: Generates detailed text prompts for video creation from parsed product data using AI  
  - Inputs: From Parse Product Data  
  - Outputs: To Text to Video2 node  
  - Edge cases: AI service timeouts, invalid prompt formatting

---

#### 1.3 AI Content Generation for Video Creation

**Overview:**  
Generates video content prompts and scripts via AI, preparing for video creation.

**Nodes Involved:**  
- OpenAI Chat Model1 (GPT Model)  
- Sora2 Prompt Generator (LangChain Agent)  
- Text to Video2 (HTTP Request)

**Node Details:**

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Provides AI language model for prompt refinement or additional content generation  
  - Inputs: (Unclear from connection, but linked to Sora2 Prompt Generator as language model)  
  - Outputs: Feeds into Sora2 Prompt Generator  
  - Edge cases: API key invalid, rate limits, prompt size limits

- **Sora2 Prompt Generator**  
  - Already described above; acts as AI agent using OpenAI model to generate final prompts

- **Text to Video2**  
  - Type: HTTP Request node  
  - Role: Sends generated prompt to Sora2 API or similar text-to-video service to initiate video creation  
  - Configuration: API endpoint, authentication, request body with prompt  
  - Inputs: From Sora2 Prompt Generator  
  - Outputs: To Wait4 node  
  - Edge cases: API failures, network issues, invalid response format

---

#### 1.4 Video Generation and Processing

**Overview:**  
Waits for video generation completion, retrieves video, downloads it, and uploads it to YouTube.

**Nodes Involved:**  
- Wait4 (Wait)  
- Get Video4 (HTTP Request)  
- If4 (Conditional)  
- Get Video (Set)  
- Download Video (HTTP Request)  
- Upload a video (YouTube)

**Node Details:**

- **Wait4**  
  - Type: Wait node  
  - Role: Delays execution to allow video processing time; webhook-enabled to resume on specific event/callback  
  - Inputs: From Text to Video2 or If4 (retry)  
  - Outputs: To Get Video4  
  - Edge cases: Timeout if video not ready, missed webhook callback

- **Get Video4**  
  - Type: HTTP Request node  
  - Role: Polls the video generation API to check video availability/status  
  - Inputs: From Wait4  
  - Outputs: To If4  
  - Edge cases: Network errors, incorrect status parsing

- **If4**  
  - Type: If node  
  - Role: Checks if video is ready; if yes, proceeds; if no, loops back to Wait4  
  - Inputs: From Get Video4  
  - Outputs: True → Get Video; False → Wait4 (retry loop)  
  - Edge cases: Infinite loop risk if video never ready

- **Get Video (Set)**  
  - Type: Set node  
  - Role: Prepares video metadata or URLs for download  
  - Inputs: From If4 (true path)  
  - Outputs: To Download Video  
  - Edge cases: Missing URLs or metadata

- **Download Video**  
  - Type: HTTP Request node  
  - Role: Downloads the generated video file for upload  
  - Inputs: From Get Video  
  - Outputs: To Upload a video  
  - Edge cases: Download failures, large file handling, timeouts

- **Upload a video (YouTube)**  
  - Type: YouTube node  
  - Role: Uploads the downloaded video to a configured YouTube channel  
  - Configuration: OAuth2 credentials required, video metadata parameters  
  - Inputs: From Download Video  
  - Outputs: To Wait node (next step)  
  - Edge cases: YouTube quota limits, authentication failure, upload errors

---

#### 1.5 YouTube Metadata Enhancement and Sheet Update

**Overview:**  
Generates optimized YouTube metadata with AI, updates YouTube video details, and marks product as published in Google Sheet.

**Nodes Involved:**  
- Wait (Wait)  
- Send message and wait for response (Telegram)  
- AI Agent for Meta Data of Youtube (LangChain Agent)  
- OpenAI Chat Model  
- Structured Output Parser1  
- Update Youtube Meta Data (YouTube)  
- Update row in sheet (Google Sheets)

**Node Details:**

- **Wait**  
  - Type: Wait node with webhook  
  - Role: Pauses workflow while awaiting Telegram user input  
  - Inputs: From Upload a video  
  - Outputs: To Send message and wait for response  
  - Edge cases: Timeout, missing webhook call

- **Send message and wait for response (Telegram)**  
  - Type: Telegram node  
  - Role: Sends message to user and waits for manual input or confirmation, improving metadata quality  
  - Inputs: From Wait  
  - Outputs: To AI Agent for Meta Data of Youtube  
  - Edge cases: Telegram API outages, user no response

- **AI Agent for Meta Data of Youtube**  
  - Type: LangChain Agent node  
  - Role: Uses OpenAI GPT to generate enriched YouTube video metadata based on user input and video context  
  - Inputs: From Telegram node and OpenAI Chat Model (via ai_languageModel link)  
  - Outputs: To Update Youtube Meta Data  
  - Edge cases: AI model latency, prompt failures

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Language model for AI Agent for Meta Data  
  - Inputs: Via ai_languageModel from Structured Output Parser and possibly Telegram data  
  - Outputs: To AI Agent for Meta Data  
  - Edge cases: Same as other OpenAI nodes

- **Structured Output Parser1**  
  - Type: Structured Output Parser (LangChain)  
  - Role: Parses AI-generated metadata into structured data for YouTube node  
  - Inputs: From OpenAI Chat Model  
  - Outputs: To AI Agent for Meta Data  
  - Edge cases: Parsing failures if AI output format changes

- **Update Youtube Meta Data (YouTube)**  
  - Type: YouTube node  
  - Role: Updates YouTube video metadata (title, description, tags) with AI-enhanced data  
  - Inputs: From AI Agent for Meta Data  
  - Outputs: To Update row in sheet  
  - Edge cases: API permission issues, quota limits

- **Update row in sheet (Google Sheets)**  
  - Type: Google Sheets node  
  - Role: Marks the product as published and logs metadata update status  
  - Inputs: From Update Youtube Meta Data  
  - Outputs: None (ends workflow)  
  - Edge cases: Sheet update conflicts, data mismatches

---

#### 1.6 User Interaction via Telegram

**Overview:**  
Integrates Telegram messaging to allow manual user interaction for metadata confirmation or editing.

**Nodes Involved:**  
- Send message and wait for response (Telegram)  
- Wait (Wait)  

**Node Details:**

- See details above in 1.5 block.  
- This block is crucial for incorporating human-in-the-loop validation for metadata quality before final publishing.

---

### 3. Summary Table

| Node Name                     | Node Type                             | Functional Role                               | Input Node(s)                   | Output Node(s)                           | Sticky Note                       |
|-------------------------------|-------------------------------------|----------------------------------------------|---------------------------------|-----------------------------------------|----------------------------------|
| Schedule Trigger              | Schedule Trigger                    | Starts workflow periodically                  | None                            | Get row(s) in sheet                     |                                  |
| Get row(s) in sheet           | Google Sheets                      | Reads affiliate product sheet rows            | Schedule Trigger                | Partnership Active and Not Published    |                                  |
| Partnership Active and Not Published | Filter                             | Filters rows for active partnerships and unpublished | Get row(s) in sheet           | Get Product Details                     |                                  |
| Get Product Details           | Set                               | Prepares data for next processing              | Partnership Active and Not Published | Limit                               |                                  |
| Limit                        | Limit                             | Limits items processed per run                  | Get Product Details             | Get Product Details from Website        |                                  |
| Get Product Details from Website | HTTP Request                    | Fetches detailed product data                   | Limit                          | If (success), Update about Error (failure) |                                  |
| If                           | If                                | Validates product data retrieval                | Get Product Details from Website | Parse Product Data (true), Update about Error (false) |                                  |
| Update about Error            | Google Sheets                     | Logs errors in sheet                            | Get Product Details from Website, If | None                                   |                                  |
| Parse Product Data            | Code                              | Parses raw product data                          | If                             | Sora2 Prompt Generator                  |                                  |
| OpenAI Chat Model1            | LangChain OpenAI Chat Model       | Provides AI model for prompt generation        | None (linked as languageModel) | Sora2 Prompt Generator                  |                                  |
| Sora2 Prompt Generator        | LangChain Agent                   | Generates video creation prompts                | Parse Product Data              | Text to Video2                         |                                  |
| Text to Video2               | HTTP Request                     | Sends prompt to text-to-video API                | Sora2 Prompt Generator          | Wait4                                  |                                  |
| Wait4                        | Wait                             | Pauses for video processing                      | Text to Video2, If4 (retry)     | Get Video4                            |                                  |
| Get Video4                   | HTTP Request                     | Checks video generation status                   | Wait4                          | If4                                   |                                  |
| If4                         | If                                | Checks if video is ready; loops or continues    | Get Video4                     | Get Video (true), Wait4 (false)       |                                  |
| Get Video                   | Set                               | Prepares data for video download                  | If4 (true)                    | Download Video                        |                                  |
| Download Video               | HTTP Request                     | Downloads generated video                         | Get Video                     | Upload a video                        |                                  |
| Upload a video               | YouTube                          | Uploads video to YouTube channel                  | Download Video                | Wait                                  |                                  |
| Wait                        | Wait                             | Waits for Telegram user input                     | Upload a video                | Send message and wait for response      |                                  |
| Send message and wait for response | Telegram                         | Sends message and waits for user input            | Wait                          | AI Agent for Meta Data of Youtube       |                                  |
| AI Agent for Meta Data of Youtube | LangChain Agent               | Generates YouTube video metadata                   | Send message and wait for response, OpenAI Chat Model | Update Youtube Meta Data           |                                  |
| OpenAI Chat Model            | LangChain OpenAI Chat Model       | Provides AI model for metadata generation          | Structured Output Parser1      | AI Agent for Meta Data of Youtube       |                                  |
| Structured Output Parser1    | LangChain Structured Output Parser| Parses AI output into structured metadata          | OpenAI Chat Model             | AI Agent for Meta Data of Youtube       |                                  |
| Update Youtube Meta Data     | YouTube                          | Updates video metadata on YouTube                  | AI Agent for Meta Data of Youtube | Update row in sheet                   |                                  |
| Update row in sheet          | Google Sheets                    | Marks product as published in sheet                | Update Youtube Meta Data       | None                                  |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run at desired intervals (e.g., daily)  
   - No credentials required

2. **Create Google Sheets Node ("Get row(s) in sheet")**  
   - Connect from Schedule Trigger  
   - Configure credentials for Google Sheets OAuth2  
   - Set spreadsheet ID and sheet name containing affiliate products  
   - Retrieve all rows or filtered range

3. **Create Filter Node ("Partnership Active and Not Published")**  
   - Connect from Google Sheets node  
   - Set filter conditions: partnership status = active AND published flag = false or empty

4. **Create Set Node ("Get Product Details")**  
   - Connect from Filter node  
   - Map or prepare fields as needed for downstream API calls

5. **Create Limit Node**  
   - Connect from Set node  
   - Limit the number of items processed per run (e.g., 1 or 5)

6. **Create HTTP Request Node ("Get Product Details from Website")**  
   - Connect from Limit node  
   - Configure URL and request parameters to fetch product details from external API  
   - Enable "Continue on Fail" to handle errors gracefully

7. **Create If Node ("If")**  
   - Connect from HTTP Request  
   - Set condition to verify successful response and required product data presence

8. **Create Google Sheets Node ("Update about Error")**  
   - Connect from HTTP Request (on failure) and If node (else branch)  
   - Configure to update error status in the product row

9. **Create Code Node ("Parse Product Data")**  
   - Connect from If node (true branch)  
   - Write JavaScript to parse and structure product data for AI prompt

10. **Create LangChain Agent Node ("Sora2 Prompt Generator")**  
    - Connect from Code node  
    - Select AI model (OpenAI GPT) for prompt generation  
    - Provide prompt template to create video scripts from product data

11. **Create OpenAI Chat Model Node ("OpenAI Chat Model1")**  
    - Select model parameters (e.g., GPT-4, temperature)  
    - Connect as AI language model to Sora2 Prompt Generator node

12. **Create HTTP Request Node ("Text to Video2")**  
    - Connect from Sora2 Prompt Generator  
    - Configure API endpoint for Sora2 text-to-video service  
    - Include authentication and pass generated prompt in request body

13. **Create Wait Node ("Wait4")**  
    - Connect from Text to Video2  
    - Set wait duration or enable webhook to pause until video ready

14. **Create HTTP Request Node ("Get Video4")**  
    - Connect from Wait4  
    - Poll Sora2 or video service to check video generation status

15. **Create If Node ("If4")**  
    - Connect from Get Video4  
    - Condition: video ready?  
    - True → Get Video  
    - False → Wait4 (loop)

16. **Create Set Node ("Get Video")**  
    - Connect from If4 (true)  
    - Extract video URL and metadata for download

17. **Create HTTP Request Node ("Download Video")**  
    - Connect from Get Video  
    - Download video file for upload to YouTube

18. **Create YouTube Node ("Upload a video")**  
    - Connect from Download Video  
    - Configure OAuth2 credentials for YouTube API  
    - Map video file and metadata fields

19. **Create Wait Node ("Wait")**  
    - Connect from Upload a video  
    - Configure webhook wait for Telegram response

20. **Create Telegram Node ("Send message and wait for response")**  
    - Connect from Wait  
    - Configure Telegram Bot API credentials  
    - Set message to prompt user for metadata input

21. **Create LangChain Agent Node ("AI Agent for Meta Data of Youtube")**  
    - Connect from Telegram node  
    - Use OpenAI Chat Model for generating metadata from user input

22. **Create OpenAI Chat Model Node ("OpenAI Chat Model")**  
    - Configure for metadata AI generation  
    - Connect as AI language model for AI Agent for Meta Data

23. **Create Structured Output Parser Node ("Structured Output Parser1")**  
    - Connect from OpenAI Chat Model  
    - Parse AI output into structured metadata fields

24. **Connect Structured Output Parser to AI Agent for Meta Data**

25. **Create YouTube Node ("Update Youtube Meta Data")**  
    - Connect from AI Agent for Meta Data  
    - Update video metadata on YouTube video

26. **Create Google Sheets Node ("Update row in sheet")**  
    - Connect from Update Youtube Meta Data  
    - Update sheet row marking video as published

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses Sora-2 AI assistant integrated via LangChain agents for advanced prompt generation. | Sora-2 LangChain agent nodes are key for AI-driven content generation and metadata optimization.  |
| Telegram integration allows seamless human-in-the-loop validation improving metadata quality.     | Telegram node requires Bot API token and webhook setup for two-way communication.                  |
| YouTube OAuth2 credentials must have permissions for upload and metadata update scopes.           | See https://developers.google.com/youtube/registering_an_application for setup details             |
| Google Sheets API must be enabled and credentials authorized for read/write access.               | https://developers.google.com/sheets/api/quickstart/nodejs                                         |
| Text-to-video API (Sora2) requires an accessible HTTP endpoint and authentication tokens.         | Check Sora2 API documentation for exact request and response formats                              |

---

This documentation provides a full reference on the structure, node roles, and setup required to reproduce and maintain the "Create & Publish Affiliate Product Videos with Sora-2, GPT & YouTube" workflow in n8n. It supports troubleshooting potential integration issues and aids developers or AI agents in understanding and modifying the workflow logic end-to-end.

---

*Disclaimer: The provided text is exclusively derived from an automated n8n workflow. All data handled are legal and public, and the workflow respects applicable content policies.*