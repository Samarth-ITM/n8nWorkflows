Generate & Auto-post AI Videos to Social Media with Veo3 and Blotato

https://n8nworkflows.xyz/workflows/generate---auto-post-ai-videos-to-social-media-with-veo3-and-blotato-5035


# Generate & Auto-post AI Videos to Social Media with Veo3 and Blotato

### 1. Workflow Overview

This workflow automates the generation, creation, and social media posting of AI-driven video content using Veo3 and Blotato platforms. It is designed to run daily, starting from AI-generated video concepts and prompts, proceeding through video creation, and culminating in automatic publication across multiple social media channels. The logical flow is divided into three main blocks:

- **1.1 Script & Prompt Generation:** Automatically generate a viral video idea, caption, and a Veo3-compatible video prompt using AI agents and language models.
- **1.2 Video Creation with Veo3:** Submit the generated prompt to Veo3’s API, wait for processing, and retrieve the final video URL.
- **1.3 Auto-Posting to Social Media via Blotato:** Upload the video to Blotato, then post the video with the associated caption to a wide range of social media platforms including Instagram, TikTok, YouTube, Facebook, Threads, Twitter, LinkedIn, Bluesky, and Pinterest.

Each block uses several specialized nodes including AI agents, HTTP requests, Google Sheets for data persistence, and automation triggers.

---

### 2. Block-by-Block Analysis

#### 2.1 Script & Prompt Generation

- **Overview:**  
  This block triggers daily to generate a concise, viral video concept and caption using AI. It then formats the idea into a detailed, Veo3-compatible prompt for video creation.

- **Nodes Involved:**  
  - Trigger: Run Daily Script Generator  
  - AI Agent: Generate Video Concept  
  - LLM: Generate Idea & Caption (GPT-4.1)  
  - Parser: Extract JSON from Idea  
  - Google Sheets: Save Script Idea  
  - AI Agent: Create Veo3-Compatible Prompt  
  - Tool: Inject Creativity  
  - Tool: Build Prompt Structure  
  - LLM: Format Prompt for Veo3 (GPT-4.1)

- **Node Details:**

  - **Trigger: Run Daily Script Generator**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow once daily  
    - Configuration: Runs every day (interval specified)  
    - Inputs: None (trigger)  
    - Outputs: Starts AI Agent: Generate Video Concept  
    - Edge cases: Trigger failure or misconfiguration could halt the entire workflow.

  - **AI Agent: Generate Video Concept**  
    - Type: Langchain AI Agent  
    - Role: Generates a single immersive, viral video idea, caption, environment description, and status in structured JSON format.  
    - Configuration: Uses a system message prompt enforcing strict formatting rules and output constraints (e.g., 1 idea, 13 words max, 12 hashtags).  
    - Inputs: Trigger output (invokes AI with fixed example topic "a Yeti speaking to a camera...")  
    - Outputs: JSON array with Caption, Idea, Environment, Status  
    - Edge cases: AI might return malformed JSON, empty results, or timeout.

  - **LLM: Generate Idea & Caption (GPT-4.1)**  
    - Type: OpenAI Chat Completion  
    - Role: Supports AI Agent with GPT-4.1 model for text generation  
    - Credentials: OpenAI API key required  
    - Inputs/Outputs: Connected to AI Agent node  
    - Edge cases: API rate limits, authentication failure, network issues.

  - **Parser: Extract JSON from Idea**  
    - Type: Langchain Output Parser  
    - Role: Parses AI Agent output to structured JSON as per example schema  
    - Inputs: Raw AI text output  
    - Outputs: Structured JSON for downstream nodes  
    - Edge cases: Parsing failures if AI output deviates from expected schema.

  - **Google Sheets: Save Script Idea**  
    - Type: Google Sheets node (append operation)  
    - Role: Persists generated idea, caption, environment prompt, and status into a Google Sheet for record-keeping.  
    - Configuration: Maps fields from AI output to specific columns, appends new rows  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Structured JSON from parser  
    - Outputs: Passes data to next AI agent  
    - Edge cases: Credential expiration, sheet access issues, API quota exceeded.

  - **AI Agent: Create Veo3-Compatible Prompt**  
    - Type: Langchain AI Agent  
    - Role: Converts the video idea and environment into a detailed Veo3 prompt following strict cinematic and stylistic rules.  
    - Configuration: Uses a detailed system message specifying format and technical requirements (e.g., time of day, lens, audio).  
    - Inputs: Data from Google Sheets (idea, environment)  
    - Outputs: Formatted prompt string  
    - Edge cases: Similar AI-related failures as prior AI agents.

  - **Tool: Inject Creativity**  
    - Type: Langchain Tool Think  
    - Role: Provides additional creativity injection for AI agent to generate viral video concepts  
    - Inputs: Connected internally to AI Agent: Generate Video Concept  
    - Outputs: Feeds back into AI Agent  
    - Edge cases: Tool invocation failure or misconfiguration.

  - **Tool: Build Prompt Structure**  
    - Type: Langchain Tool Think  
    - Role: Supports structuring the Veo3 prompt before final AI formatting  
    - Inputs: Connected to AI Agent for Veo3 prompt  
    - Outputs: Passed to LLM for final formatting  
    - Edge cases: Tool failure or processing delays.

  - **LLM: Format Prompt for Veo3 (GPT-4.1)**  
    - Type: OpenAI Chat Completion  
    - Role: Final formatting of prompt for Veo3 video generation using GPT-4.1  
    - Credentials: OpenAI API key required  
    - Inputs: Raw prompt structure  
    - Outputs: Final prompt string  
    - Edge cases: API or network failures.

---

#### 2.2 Video Creation with Veo3

- **Overview:**  
  This block submits the formatted prompt to the Veo3 API to generate the video, waits for processing, then retrieves the final video URL.

- **Nodes Involved:**  
  - Call Veo3 API to Generate Video  
  - Wait for Veo3 Processing (5 mins)  
  - Retrieve Final Video URL from Veo3  
  - Google Sheets: Log Final Video Output

- **Node Details:**

  - **Call Veo3 API to Generate Video**  
    - Type: HTTP Request  
    - Role: Sends POST request with JSON body containing the prompt to Veo3’s video generation API endpoint  
    - Configuration: Uses HTTP header authentication with credential; batch size 1 with 2-second interval  
    - Inputs: Prompt string from AI Agent: Create Veo3-Compatible Prompt  
    - Outputs: Returns a request ID for status polling  
    - Edge cases: API errors, authentication failure, malformed request body.

  - **Wait for Veo3 Processing (5 mins)**  
    - Type: Wait node  
    - Role: Pauses workflow execution for 5 minutes to allow video processing  
    - Inputs: Request ID from previous node  
    - Outputs: Triggers next node after delay  
    - Edge cases: Excessive delay if video processing is slow; fixed wait may not cover longer processing times.

  - **Retrieve Final Video URL from Veo3**  
    - Type: HTTP Request  
    - Role: GET request to Veo3 API to fetch the processed video URL using request ID  
    - Configuration: Uses same HTTP header authentication  
    - Inputs: Request ID from Call Veo3 API node  
    - Outputs: JSON containing video URL  
    - Edge cases: API timeout, video not ready, authentication failure.

  - **Google Sheets: Log Final Video Output**  
    - Type: Google Sheets Update  
    - Role: Updates the previously saved script idea row with production status "done" and logs the final video URL  
    - Configuration: Matches row by idea text; updates columns accordingly  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Video URL and original idea data  
    - Outputs: Triggers next block for video posting  
    - Edge cases: Sheet access issues, data mismatch, update conflicts.

---

#### 2.3 Auto-Posting to Social Media via Blotato

- **Overview:**  
  This block uploads the final video to Blotato and posts it to multiple social media accounts using their respective IDs stored in a set node.

- **Nodes Involved:**  
  - Get my video (Google Sheets read)  
  - Assign Social Media IDs (Set node)  
  - Upload Video to Blotato (HTTP Request)  
  - INSTAGRAM (HTTP Request)  
  - YOUTUBE (HTTP Request)  
  - TIKTOK (HTTP Request)  
  - FACEBOOK (HTTP Request)  
  - THREADS (HTTP Request)  
  - TWETTER [sic] (HTTP Request)  
  - LINKEDIN (HTTP Request)  
  - BLUESKY (HTTP Request)  
  - PINTEREST (HTTP Request)  
  - Google Sheets (Update post status)

- **Node Details:**

  - **Get my video**  
    - Type: Google Sheets Read  
    - Role: Reads the row containing the finalized video data (caption, description, video URL) to prepare for social posting  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Triggered by previous block’s completion  
    - Outputs: Passes video metadata forward  
    - Edge cases: Sheet access failure, no matching row found.

  - **Assign Social Media IDs**  
    - Type: Set  
    - Role: Holds hardcoded social media account IDs for all platforms targeted (Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Pinterest, Bluesky)  
    - Configuration: Static JSON with placeholder IDs ("1111") that must be replaced for production  
    - Inputs: Video metadata from Get my video  
    - Outputs: Provides social account IDs for posting nodes  
    - Edge cases: Missing or incorrect account IDs will cause post failures.

  - **Upload Video to Blotato**  
    - Type: HTTP Request  
    - Role: Uploads video URL to Blotato media endpoint using API key authentication  
    - Parameters: Sends video URL from Google Sheets final output field in POST body  
    - Authentication: API key in header ("blotato-api-key")  
    - Inputs: Video URL and social IDs  
    - Outputs: Returns media upload confirmation and URL for posting  
    - Edge cases: API key invalid, upload failure, network issues.

  - **Social Media Posting Nodes (INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWETTER, LINKEDIN, BLUESKY, PINTEREST)**  
    - Type: HTTP Request (POST)  
    - Role: Each sends a POST request to Blotato’s posting API endpoint to create a post on the respective platform.  
    - Configuration:  
      - Uses JSON body with accountId from Assign Social Media IDs  
      - Includes platform-specific target parameters (e.g., pageId for Facebook, boardId for Pinterest)  
      - Sets content text as the video description from Google Sheets  
      - Media URLs array includes the uploaded video URL from Blotato  
      - Authentication via Blotato API key header  
    - Inputs: Output from Upload Video to Blotato node  
    - Outputs: Confirmation of post creation (not directly captured)  
    - Edge cases:  
      - Invalid or expired API key  
      - Incorrect or missing account IDs  
      - Platform-specific posting errors (e.g., privacy settings)  
      - Rate limits or network errors

  - **Google Sheets (Update post status)**  
    - Type: Google Sheets Update  
    - Role: Updates the video’s status in the Google Sheet to "Publish" after successful posting  
    - Inputs: From Upload Video to Blotato and social posting nodes  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Sheet update conflicts or failures.

---

### 3. Summary Table

| Node Name                       | Node Type                        | Functional Role                                   | Input Node(s)                                   | Output Node(s)                                    | Sticky Note                                                                                       |
|--------------------------------|---------------------------------|-------------------------------------------------|------------------------------------------------|--------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Trigger: Run Daily Script Generator | Schedule Trigger                 | Initiates the workflow daily                     | None                                           | AI Agent: Generate Video Concept                  | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| AI Agent: Generate Video Concept | Langchain AI Agent              | Generates viral video idea & caption JSON       | Trigger: Run Daily Script Generator             | Google Sheets: Save Script Idea                    | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| LLM: Generate Idea & Caption (GPT-4.1) | OpenAI Chat Completion          | Supports idea & caption generation               | AI Agent: Generate Video Concept (ai_languageModel) | AI Agent: Generate Video Concept (ai_languageModel) | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| Parser: Extract JSON from Idea  | Langchain Output Parser          | Parses AI output into structured JSON            | AI Agent: Generate Video Concept (ai_outputParser) | AI Agent: Generate Video Concept                   | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| Google Sheets: Save Script Idea | Google Sheets (Append)           | Saves generated video idea and caption           | AI Agent: Generate Video Concept                 | AI Agent: Create Veo3-Compatible Prompt            | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| AI Agent: Create Veo3-Compatible Prompt | Langchain AI Agent              | Creates detailed Veo3 video prompt                | Google Sheets: Save Script Idea                   | Call Veo3 API to Generate Video                    | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| Tool: Inject Creativity          | Langchain Tool Think             | Injects creativity for AI ideas                   | AI Agent: Generate Video Concept (ai_tool)       | AI Agent: Generate Video Concept                    | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| Tool: Build Prompt Structure     | Langchain Tool Think             | Structures prompt before final AI formatting      | AI Agent: Create Veo3-Compatible Prompt (ai_tool) | LLM: Format Prompt for Veo3 (ai_languageModel)     | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| LLM: Format Prompt for Veo3 (GPT-4.1) | OpenAI Chat Completion          | Finalizes prompt format for Veo3                  | Tool: Build Prompt Structure                      | AI Agent: Create Veo3-Compatible Prompt (ai_languageModel) | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| Call Veo3 API to Generate Video | HTTP Request                    | Sends prompt to Veo3 API for video generation     | AI Agent: Create Veo3-Compatible Prompt           | Wait for Veo3 Processing (5 mins)                   | # ✅ STEP 2 — Create Video Using Veo3                                                            |
| Wait for Veo3 Processing (5 mins) | Wait                          | Pauses to allow Veo3 video processing             | Call Veo3 API to Generate Video                   | Retrieve Final Video URL from Veo3                   | # ✅ STEP 2 — Create Video Using Veo3                                                            |
| Retrieve Final Video URL from Veo3 | HTTP Request                    | Retrieves the final video URL                      | Wait for Veo3 Processing (5 mins)                  | Google Sheets: Log Final Video Output                | # ✅ STEP 2 — Create Video Using Veo3                                                            |
| Google Sheets: Log Final Video Output | Google Sheets (Update)          | Logs final video URL and sets production done     | Retrieve Final Video URL from Veo3                  | Get my video                                         | # ✅ STEP 2 — Create Video Using Veo3                                                            |
| Get my video                    | Google Sheets (Read)             | Reads video metadata for posting                   | Google Sheets: Log Final Video Output              | Assign Social Media IDs                              | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| Assign Social Media IDs          | Set                            | Sets all social media account IDs                  | Get my video                                        | Upload Video to Blotato                              | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| Upload Video to Blotato          | HTTP Request                    | Uploads video to Blotato platform                   | Assign Social Media IDs                             | INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWETTER, LINKEDIN, BLUESKY, PINTEREST, Google Sheets | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| INSTAGRAM                      | HTTP Request                    | Posts video to Instagram via Blotato                | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| YOUTUBE                        | HTTP Request                    | Posts video to YouTube via Blotato                  | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| TIKTOK                        | HTTP Request                    | Posts video to TikTok via Blotato                    | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| FACEBOOK                      | HTTP Request                    | Posts video to Facebook via Blotato                  | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| THREADS                       | HTTP Request                    | Posts video to Threads via Blotato                   | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| TWETTER                       | HTTP Request                    | Posts video to Twitter (typo in name) via Blotato   | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| LINKEDIN                      | HTTP Request                    | Posts video to LinkedIn via Blotato                   | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| BLUESKY                       | HTTP Request                    | Posts video to Bluesky via Blotato                    | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| PINTEREST                    | HTTP Request                    | Posts video to Pinterest via Blotato                  | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| Google Sheets                 | Google Sheets (Update)           | Updates video status to “Publish” after posting     | Upload Video to Blotato                             | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |
| Sticky Note                   | Sticky Note                     | Step 1 label for Script & Prompt generation          | None                                               | None                                               | # ✅ STEP 1 — Generate Script & Prompt with AI                                                   |
| Sticky Note1                  | Sticky Note                     | Step 2 label for Video Creation with Veo3             | None                                               | None                                               | # ✅ STEP 2 — Create Video Using Veo3                                                            |
| Sticky Note2                  | Sticky Note                     | Step 3 label for Publishing Video to Social Media     | None                                               | None                                               | # ✅ STEP 3 — Publish Video to Social Media                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: `Trigger: Run Daily Script Generator`  
   - Set interval to run once every day.

2. **Add an AI Agent Node for Video Concept Generation**  
   - Name: `AI Agent: Generate Video Concept`  
   - Use Langchain AI Agent node.  
   - Configure with system prompt to generate one viral video idea in JSON format with keys: Caption, Idea, Environment, Status.  
   - Set fixed input text describing the topic (e.g., a Yeti vlogging).  
   - Connect output from schedule trigger.

3. **Add OpenAI GPT-4.1 Node for Supporting Idea Generation**  
   - Name: `LLM: Generate Idea & Caption (GPT-4.1)`  
   - Connect as AI language model for the AI Agent node above.  
   - Configure with OpenAI credentials.

4. **Add Output Parser Node**  
   - Name: `Parser: Extract JSON from Idea`  
   - Configure with expected JSON schema example matching AI Agent output.  
   - Connect AI Agent output to this node.

5. **Add Google Sheets Append Node**  
   - Name: `Google Sheets: Save Script Idea`  
   - Configure to append a new row with columns: id, idea, caption, production status, environment_prompt.  
   - Use Google Sheets OAuth2 credentials.  
   - Connect parser output here.

6. **Add AI Agent Node to Create Veo3-Compatible Prompt**  
   - Name: `AI Agent: Create Veo3-Compatible Prompt`  
   - Langchain AI Agent node with detailed system message describing cinematic video prompt format, including rules and required technical elements.  
   - Input the idea and environment_prompt from Google Sheets saved data.

7. **Add Tool Nodes for Creativity Injection and Prompt Structuring**  
   - Name: `Tool: Inject Creativity` and `Tool: Build Prompt Structure`  
   - Connect appropriately to support AI Agent nodes for enhancing prompt generation.

8. **Add OpenAI GPT-4.1 Node for Final Prompt Formatting**  
   - Name: `LLM: Format Prompt for Veo3 (GPT-4.1)`  
   - Connect to Tool: Build Prompt Structure output.  
   - Configure with OpenAI credentials.

9. **Add HTTP Request Node to Call Veo3 API**  
   - Name: `Call Veo3 API to Generate Video`  
   - POST to `https://queue.fal.run/fal-ai/veo3` with JSON body containing the prompt string.  
   - Use HTTP Header authentication with Veo3 API key credential.  
   - Connect from AI Agent: Create Veo3-Compatible Prompt.

10. **Add Wait Node for 5 Minutes**  
    - Name: `Wait for Veo3 Processing (5 mins)`  
    - Connect from Veo3 API call node.

11. **Add HTTP Request Node to Retrieve Final Video URL**  
    - Name: `Retrieve Final Video URL from Veo3`  
    - GET request to `https://queue.fal.run/fal-ai/veo3/requests/{{request_id}}` where `request_id` is from previous response.  
    - Use same HTTP header authentication.  
    - Connect from wait node.

12. **Add Google Sheets Update Node to Log Final Video Output**  
    - Name: `Google Sheets: Log Final Video Output`  
    - Update row matched by idea, set production to "done", save final_output URL.  
    - Credentials: Google Sheets OAuth2.  
    - Connect from retrieve video URL node.

13. **Add Google Sheets Read Node to Get Final Video Metadata**  
    - Name: `Get my video`  
    - Read row containing final video data from Google Sheets.  
    - Credentials: Google Sheets OAuth2.  
    - Connect from Log Final Video Output node.

14. **Add Set Node for Social Media Account IDs**  
    - Name: `Assign Social Media IDs`  
    - Set JSON object with all social media platform account IDs (Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Pinterest, Bluesky).  
    - Replace placeholders with real account IDs.  
    - Connect from Google Sheets read node.

15. **Add HTTP Request Node to Upload Video to Blotato**  
    - Name: `Upload Video to Blotato`  
    - POST to `https://backend.blotato.com/v2/media` with video URL in body.  
    - Authenticate with Blotato API key header.  
    - Connect from Assign Social Media IDs node.

16. **Add HTTP Request Nodes for Each Social Media Platform**  
    - Names: `INSTAGRAM`, `YOUTUBE`, `TIKTOK`, `FACEBOOK`, `THREADS`, `TWETTER`, `LINKEDIN`, `BLUESKY`, `PINTEREST`  
    - POST to `https://backend.blotato.com/v2/posts` with JSON body specifying accountId, targetType, content text (video description), and mediaUrls (video URL).  
    - Use Blotato API key for authentication.  
    - Connect all from Upload Video to Blotato node.

17. **Add Google Sheets Update Node for Publishing Status**  
    - Name: `Google Sheets`  
    - Update the video row status to "Publish".  
    - Credentials: Google Sheets OAuth2.  
    - Connect from Upload Video to Blotato node or after social posts as needed.

18. **Add Sticky Note Nodes**  
    - Create Sticky Notes to label steps:  
      - Step 1: Generate Script & Prompt with AI  
      - Step 2: Create Video Using Veo3  
      - Step 3: Publish Video to Social Media  
    - Position them appropriately for workflow documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow requires valid API keys for OpenAI, Veo3 (via HTTP Header Auth), and Blotato (API key).   | Credentials setup                                    |
| Replace all placeholder account IDs in "Assign Social Media IDs" node with actual IDs from your social platforms. | Social media account integration                     |
| The workflow uses Google Sheets to store and track video ideas, prompts, and status updates.            | Data persistence and state management                |
| The Veo3 prompt must be highly detailed and follow strict cinematic rules to ensure quality video generation. | Video generation quality                              |
| Blotato platform serves as the media hosting and social media posting intermediary.                     | Video upload and multi-platform posting              |
| The workflow includes a fixed 5-minute wait which may need adjustment depending on Veo3 processing speed. | Timing and reliability consideration                  |
| Social media posting nodes have platform-specific parameters (e.g., privacyStatus on YouTube, boardId on Pinterest). | Platform-specific configuration                       |
| The node named "TWETTER" is likely a typo for Twitter; rename accordingly for clarity.                  | Naming convention                                    |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.