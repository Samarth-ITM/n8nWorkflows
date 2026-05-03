Generate AI Viral Videos with Seedance and Upload to TikTok, YouTube & Instagram

https://n8nworkflows.xyz/workflows/generate-ai-viral-videos-with-seedance-and-upload-to-tiktok--youtube---instagram-5338


# Generate AI Viral Videos with Seedance and Upload to TikTok, YouTube & Instagram

---

### 1. Workflow Overview

This workflow automates the creation of viral AI-generated videos starting from a creative concept, producing video clips and sounds, stitching them into a final video, and then publishing the video across multiple social media platforms including TikTok, YouTube, Instagram, Facebook, Twitter, LinkedIn, Pinterest, Threads, and Bluesky.

The logical structure is organized into the following blocks:

- **1.1 Input Reception and Idea Generation:**  
  Initiates daily content creation triggered by a schedule; generates a creative video idea based on a defined prompt.

- **1.2 Idea Processing and Prompt Generation:**  
  Parses AI output, saves metadata, and generates detailed multi-scene video prompts for cinematic-style ASMR videos.

- **1.3 Video Clip Generation (Wavespeed AI):**  
  Extracts detailed video scenes and requests Wavespeed AI to generate corresponding video clips.

- **1.4 Sound Generation (Fal AI):**  
  Generates ASMR soundtracks aligned with the video clips.

- **1.5 Video Stitching and Finalization (Fal AI):**  
  Merges the generated clips into a single video and retrieves the final output.

- **1.6 Social Media Publishing via Blotato:**  
  Uploads the final video to Blotato, assigns target social media account IDs, and publishes the video to multiple platforms with appropriate metadata.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Idea Generation

**Overview:**  
This block triggers the workflow at scheduled intervals and uses AI to generate a creative video idea involving a hard material being sliced, formatted as a structured JSON output.

**Nodes Involved:**  
- Trigger: Start Daily Content Generation  
- AI Agent: Generate Creative Video Idea  
- Tool: Inject Creative Perspective (Idea)  
- LLM: Generate Raw Idea (GPT-4.1)  
- Parse AI Output (Idea, Environment, Sound)  
- Save Idea & Metadata to Google Sheets  

**Node Details:**

- **Trigger: Start Daily Content Generation**  
  - Type: Schedule Trigger  
  - Configuration: Runs on an interval schedule (default daily)  
  - Inputs: None  
  - Outputs: Starts the workflow chain to generate video ideas  
  - Potential Failures: Missed trigger due to workflow downtime or scheduler misconfiguration  

- **AI Agent: Generate Creative Video Idea**  
  - Type: LangChain Agent (AI)  
  - Configuration: Generates one immersive video idea based on a specific prompt about slicing hard materials; uses a Think tool to validate output  
  - Inputs: Trigger output  
  - Outputs: JSON array with Caption, Idea, Environment, Sound, Status  
  - Key Expressions: Uses a system message enforcing strict output format and content rules  
  - Potential Failures: AI model errors, malformed output, rate limiting  

- **Tool: Inject Creative Perspective (Idea)**  
  - Type: LangChain Tool (Think Tool)  
  - Role: Assists AI Agent by injecting creative thinking or validations  
  - Inputs: AI Agent node output  
  - Outputs: Refined AI responses  
  - Potential Failures: Expression errors or tool unavailability  

- **LLM: Generate Raw Idea (GPT-4.1)**  
  - Type: OpenAI Chat Completion (GPT-4.1)  
  - Configuration: Used as a fallback or enhancement to refine ideas  
  - Credentials: OpenAI API key  
  - Inputs: AI Agent prompt  
  - Outputs: Refined idea text  
  - Potential Failures: API quota limits, network issues, invalid credentials  

- **Parse AI Output (Idea, Environment, Sound)**  
  - Type: Structured Output Parser (LangChain)  
  - Role: Parses JSON output from AI to extract relevant fields  
  - Inputs: Raw AI output  
  - Outputs: Structured JSON for downstream use  
  - Potential Failures: Parsing errors if AI output format deviates  

- **Save Idea & Metadata to Google Sheets**  
  - Type: Google Sheets Append  
  - Configuration: Saves idea, caption, environment, sound prompt, and production status to a spreadsheet  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Parsed AI output  
  - Outputs: Confirmation of data saved  
  - Potential Failures: Authentication errors, sheet access denied, quota exceeded  

---

#### 1.2 Idea Processing and Prompt Generation

**Overview:**  
Generates detailed multi-scene cinematic video prompts from the initial idea, parses outputs, and extracts individual scene descriptions for clip generation.

**Nodes Involved:**  
- AI Agent: Generate Detailed Video Prompts  
- LLM: Draft Video Prompt Details (GPT-4.1)  
- Tool: Refine and Validate Prompts1  
- Parse Structured Video Prompt Output  
- Extract Individual Scene Descriptions  

**Node Details:**

- **AI Agent: Generate Detailed Video Prompts**  
  - Type: LangChain Agent (AI)  
  - Configuration: Generates 3+ detailed video prompts embodying cinematic, macro-level cutting scenes; uses Think tool for validation  
  - Inputs: Saved idea and related environment and sound prompts  
  - Outputs: JSON with detailed scenes descriptions  
  - Potential Failures: AI generation errors, format issues  

- **LLM: Draft Video Prompt Details (GPT-4.1)**  
  - Type: OpenAI Chat Completion  
  - Role: Supports AI Agent with detailed prompt drafting  
  - Credentials: OpenAI API key  
  - Potential Failures: API errors, network issues  

- **Tool: Refine and Validate Prompts1**  
  - Type: LangChain Tool (Think Tool)  
  - Role: Validates and refines prompt quality before use  
  - Potential Failures: Tool or expression failures  

- **Parse Structured Video Prompt Output**  
  - Type: Structured Output Parser  
  - Role: Parses detailed video prompt JSON, extracting scenes with descriptions  
  - Potential Failures: Parsing errors on malformed AI output  

- **Extract Individual Scene Descriptions**  
  - Type: Code Node (JavaScript)  
  - Role: Extracts all scene description strings from the parsed JSON to produce an array of scenes for clip generation  
  - Edge Cases: No scene keys found results in error; ensures robust extraction from nested objects  

---

#### 1.3 Video Clip Generation (Wavespeed AI)

**Overview:**  
Requests Wavespeed AI to generate short video clips for each detailed scene, waits for generation, then retrieves the clips.

**Nodes Involved:**  
- Generate Video Clips (Wavespeed AI)  
- Wait for Clip Generation (Wavespeed AI)  
- Retrieve Video Clips  

**Node Details:**

- **Generate Video Clips (Wavespeed AI)**  
  - Type: HTTP Request  
  - Configuration: POST request to Wavespeed API with video theme, scene description, and environment as prompt; aspect ratio 9:16, duration 10 seconds per clip; uses API key authentication via HTTP Header  
  - Inputs: Scene descriptions from code node, idea, environment  
  - Outputs: Job ID for clip generation  
  - Edge Cases: API rate limits, authentication failure, invalid prompt data  

- **Wait for Clip Generation (Wavespeed AI)**  
  - Type: Wait Node  
  - Configuration: Waits 240 seconds (4 minutes) for clip generation to complete  
  - Inputs: Job ID from previous node  
  - Outputs: Triggers next node after delay  
  - Edge Cases: Insufficient wait time may cause retrieval failures  

- **Retrieve Video Clips**  
  - Type: HTTP Request  
  - Configuration: GET request to Wavespeed API to fetch generated clip URLs using job ID  
  - Outputs: JSON containing video clip URLs  
  - Edge Cases: Job not ready yet, network errors, invalid job ID  

---

#### 1.4 Sound Generation (Fal AI)

**Overview:**  
Generates ASMR sound effects matching the video clips and waits for completion.

**Nodes Involved:**  
- Generate ASMR Sound (Fal AI)  
- Wait for Sound Generation (Fal AI)  
- Retrieve Final Sound Output  

**Node Details:**

- **Generate ASMR Sound (Fal AI)**  
  - Type: HTTP Request  
  - Configuration: POST to Fal AI audio generation endpoint with prompt describing the sound, duration 10 seconds, and URL of the first video clip for reference  
  - Inputs: Video clip URL and sound prompt from AI  
  - Outputs: Request ID for sound generation  
  - Edge Cases: API errors, authentication issues, malformed prompt  

- **Wait for Sound Generation (Fal AI)**  
  - Type: Wait Node  
  - Configuration: Waits 60 seconds for sound generation to complete  
  - Inputs: Request ID from previous node  
  - Outputs: Triggers next node  
  - Edge Cases: Insufficient wait time causing premature retrieval  

- **Retrieve Final Sound Output**  
  - Type: HTTP Request  
  - Configuration: GET request to Fal AI API to fetch generated sound using request ID  
  - Outputs: Sound file URLs or references  
  - Edge Cases: Resource not ready, network failures  

---

#### 1.5 Video Stitching and Finalization (Fal AI)

**Overview:**  
Compiles the generated clips into a single final video, waits for rendering, retrieves the merged video, and updates Google Sheets with the final video URL.

**Nodes Involved:**  
- List Clip URLs for Stitching  
- Merge Clips into Final Video (Fal AI)  
- Wait for Video Rendering (Fal AI)  
- Retrieve Final Merged Video  
- URL Final Video (Google Sheets update)  

**Node Details:**

- **List Clip URLs for Stitching**  
  - Type: Code Node  
  - Role: Collects all video clip URLs into a single array for stitching  
  - Inputs: Retrieved clips JSON  
  - Outputs: JSON with array of clip URLs  

- **Merge Clips into Final Video (Fal AI)**  
  - Type: HTTP Request  
  - Configuration: POST to Fal AI ffmpeg API to compose clips sequentially into one video track, each clip 10 seconds, providing URLs of each clip  
  - Inputs: Array of video URLs  
  - Outputs: Request ID for video rendering  
  - Edge Cases: Invalid URLs, API limits  

- **Wait for Video Rendering (Fal AI)**  
  - Type: Wait Node  
  - Configuration: Waits 60 seconds for video rendering to complete  
  - Inputs: Request ID  
  - Outputs: Triggers retrieval node  

- **Retrieve Final Merged Video**  
  - Type: HTTP Request  
  - Configuration: GET request to Fal AI to fetch the final merged video URL using request ID  
  - Outputs: Final video URL  

- **URL Final Video**  
  - Type: Google Sheets Update  
  - Configuration: Updates the Google Sheet row corresponding to the idea with the final video URL and marks production as done  
  - Inputs: Final video URL and idea identifier  
  - Outputs: Confirmation of update  
  - Edge Cases: Authentication failures, row not found  

---

#### 1.6 Social Media Publishing via Blotato

**Overview:**  
Assigns social media account IDs, uploads final video to Blotato, then posts the video content to multiple social media platforms via Blotato API.

**Nodes Involved:**  
- Assign Social Media IDs  
- Get my video (Google Sheets read)  
- Upload Video to Blotato  
- INSTAGRAM  
- YOUTUBE  
- TIKTOK  
- FACEBOOK  
- THREADS  
- TWITTER  
- LINKEDIN  
- BLUESKY  
- PINTEREST  
- Update Production (Google Sheets)  

**Node Details:**

- **Assign Social Media IDs**  
  - Type: Set Node  
  - Configuration: Hardcoded JSON object assigning placeholder IDs for various social media accounts (Instagram, YouTube, TikTok, Facebook, Twitter, LinkedIn, Pinterest, Bluesky, Threads)  
  - Outputs: Social media IDs for downstream nodes  

- **Get my video**  
  - Type: Google Sheets Read  
  - Configuration: Reads video metadata from Google Sheets including final video URL and descriptions  
  - Inputs: N/A  
  - Outputs: Video details for posting  

- **Upload Video to Blotato**  
  - Type: HTTP Request  
  - Configuration: POST to Blotato media endpoint uploading the video using the URL from Google Sheets; requires API key in header  
  - Inputs: Final video URL  
  - Outputs: Confirmation and media ID from Blotato  
  - Edge Cases: API key invalid, upload failure  

- **Social Media Posting Nodes (INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, TWITTER, LINKEDIN, BLUESKY, PINTEREST)**  
  - Type: HTTP Request  
  - Configuration: POST requests to Blotato posts endpoint with respective platform account IDs, content text, media URLs, and platform-specific metadata (privacy, notification settings, etc.)  
  - Inputs: Media URL from upload, video description from Google Sheets, social media IDs  
  - Outputs: Posting confirmation  
  - Edge Cases: Invalid credentials, API errors, rate limits, platform-specific restrictions  

- **Update Production**  
  - Type: Google Sheets Update  
  - Configuration: Updates the production status in the sheet to "Publish" for the corresponding video row  
  - Inputs: Row number from Google Sheets read  
  - Outputs: Confirmation of update  

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                                  | Input Node(s)                   | Output Node(s)                                  | Sticky Note                                                             |
|----------------------------------|----------------------------------|-------------------------------------------------|---------------------------------|-------------------------------------------------|-------------------------------------------------------------------------|
| Trigger: Start Daily Content Generation | Schedule Trigger                 | Initiate workflow on schedule                    | None                            | AI Agent: Generate Creative Video Idea          | ## | INPUT: Starting Idea Section                                          |
| AI Agent: Generate Creative Video Idea  | LangChain Agent (AI)              | Generate initial creative video idea             | Trigger: Start Daily Content Generation | Save Idea & Metadata to Google Sheets             |                                                                       |
| Tool: Inject Creative Perspective (Idea) | LangChain Tool (Think)            | Inject creative perspective into idea generation | AI Agent: Generate Creative Video Idea | AI Agent: Generate Creative Video Idea             |                                                                       |
| LLM: Generate Raw Idea (GPT-4.1)          | OpenAI LLM Chat                  | Enhance idea generation                           | Tool: Inject Creative Perspective (Idea) | AI Agent: Generate Creative Video Idea             |                                                                       |
| Parse AI Output (Idea, Environment, Sound) | Output Parser (LangChain)        | Parse AI idea output into structured data        | AI Agent: Generate Creative Video Idea | Save Idea & Metadata to Google Sheets             |                                                                       |
| Save Idea & Metadata to Google Sheets      | Google Sheets Append             | Store generated idea and metadata                 | Parse AI Output (Idea, Environment, Sound) | AI Agent: Generate Detailed Video Prompts         |                                                                       |
| AI Agent: Generate Detailed Video Prompts  | LangChain Agent (AI)              | Generate detailed multi-scene video prompts       | Save Idea & Metadata to Google Sheets | Extract Individual Scene Descriptions               | ## | Step 3: Stitch Video (Fal AI)                                       |
| LLM: Draft Video Prompt Details (GPT-4.1) | OpenAI LLM Chat                  | Support detailed video prompt generation          | AI Agent: Generate Detailed Video Prompts | AI Agent: Generate Detailed Video Prompts         |                                                                       |
| Tool: Refine and Validate Prompts1         | LangChain Tool (Think)            | Refine and validate prompt quality                 | AI Agent: Generate Detailed Video Prompts | AI Agent: Generate Detailed Video Prompts         |                                                                       |
| Parse Structured Video Prompt Output        | Output Parser (LangChain)        | Parse detailed video prompt JSON                   | AI Agent: Generate Detailed Video Prompts | AI Agent: Generate Detailed Video Prompts         |                                                                       |
| Extract Individual Scene Descriptions       | Code (JavaScript)                | Extract individual scene descriptions              | AI Agent: Generate Detailed Video Prompts | Generate Video Clips (Wavespeed AI)               | ## | Step 1: Generate Clips (Wavespeed AI)                              |
| Generate Video Clips (Wavespeed AI)         | HTTP Request                    | Request AI to generate video clips                  | Extract Individual Scene Descriptions | Wait for Clip Generation (Wavespeed AI)           |                                                                       |
| Wait for Clip Generation (Wavespeed AI)    | Wait Node                      | Wait for clip generation to complete                | Generate Video Clips (Wavespeed AI) | Retrieve Video Clips                              |                                                                       |
| Retrieve Video Clips                        | HTTP Request                    | Retrieve generated video clips                       | Wait for Clip Generation (Wavespeed AI) | Generate ASMR Sound (Fal AI)                      | ## | Step 2: Generate Sounds (Fal AI)                                  |
| Generate ASMR Sound (Fal AI)                 | HTTP Request                    | Request ASMR sound generation                        | Retrieve Video Clips             | Wait for Sound Generation (Fal AI)                |                                                                       |
| Wait for Sound Generation (Fal AI)          | Wait Node                      | Wait for sound generation to complete                | Generate ASMR Sound (Fal AI)       | Retrieve Final Sound Output                       |                                                                       |
| Retrieve Final Sound Output                   | HTTP Request                    | Retrieve generated ASMR sound                        | Wait for Sound Generation (Fal AI) | List Clip URLs for Stitching                      |                                                                       |
| List Clip URLs for Stitching                  | Code (JavaScript)               | Consolidate clip URLs for stitching                  | Retrieve Final Sound Output       | Merge Clips into Final Video (Fal AI)             |                                                                       |
| Merge Clips into Final Video (Fal AI)        | HTTP Request                    | Request video stitch/merge operation                 | List Clip URLs for Stitching      | Wait for Video Rendering (Fal AI)                 |                                                                       |
| Wait for Video Rendering (Fal AI)            | Wait Node                      | Wait for final video rendering                        | Merge Clips into Final Video (Fal AI) | Retrieve Final Merged Video                       |                                                                       |
| Retrieve Final Merged Video                    | HTTP Request                    | Retrieve final stitched video URL                     | Wait for Video Rendering (Fal AI) | URL Final Video                                   |                                                                       |
| URL Final Video                              | Google Sheets Update            | Update Google Sheet with final video URL and status | Retrieve Final Merged Video       | Get my video                                      |                                                                       |
| Get my video                                | Google Sheets Read              | Read video metadata and URLs for publishing           | URL Final Video                  | Assign Social Media IDs                           |                                                                       |
| Assign Social Media IDs                      | Set Node                       | Assign social media account IDs for posting            | Get my video                    | Upload Video to Blotato                           |                                                                       |
| Upload Video to Blotato                      | HTTP Request                   | Upload final video to Blotato media library             | Assign Social Media IDs           | INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS... |                                                                       |
| INSTAGRAM                                  | HTTP Request                   | Publish video post to Instagram via Blotato             | Upload Video to Blotato           | None                                            |                                                                       |
| YOUTUBE                                    | HTTP Request                   | Publish video post to YouTube via Blotato               | Upload Video to Blotato           | None                                            |                                                                       |
| TIKTOK                                     | HTTP Request                   | Publish video post to TikTok via Blotato                | Upload Video to Blotato           | None                                            |                                                                       |
| FACEBOOK                                   | HTTP Request                   | Publish video post to Facebook via Blotato              | Upload Video to Blotato           | None                                            |                                                                       |
| THREADS                                    | HTTP Request                   | Publish video post to Threads via Blotato               | Upload Video to Blotato           | None                                            |                                                                       |
| TWITTER                                    | HTTP Request                   | Publish video post to Twitter via Blotato               | Upload Video to Blotato           | None                                            |                                                                       |
| LINKEDIN                                   | HTTP Request                   | Publish video post to LinkedIn via Blotato              | Upload Video to Blotato           | None                                            |                                                                       |
| BLUESKY                                    | HTTP Request                   | Publish video post to Bluesky via Blotato               | Upload Video to Blotato           | None                                            |                                                                       |
| PINTEREST                                  | HTTP Request                   | Publish video post to Pinterest via Blotato             | Upload Video to Blotato           | None                                            |                                                                       |
| Update Production                           | Google Sheets Update           | Mark video production as "Publish" in Google Sheets      | Upload Video to Blotato           | None                                            |                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure trigger to run daily (or desired interval).

2. **Create AI Agent: Generate Creative Video Idea (LangChain Agent):**  
   - Add LangChain Agent node.  
   - Set prompt to generate a single creative video idea involving a hard material being sliced, output as a JSON array with caption, idea, environment, sound, and status.  
   - Enable Think tool for output validation.

3. **Add Tool: Inject Creative Perspective (LangChain Tool - Think):**  
   - Connect to AI Agent node to refine idea generation.

4. **Add LLM Node (OpenAI GPT-4.1):**  
   - Connect to the tool node as fallback or enhancement.  
   - Set model to GPT-4.1.  
   - Use OpenAI credentials.

5. **Add Parse AI Output Node (LangChain Structured Output Parser):**  
   - Parse AI Agent output JSON into structured fields (Caption, Idea, Environment, Sound, Status).

6. **Add Google Sheets Append Node:**  
   - Connect to parser output.  
   - Configure document and sheet with columns: idea, caption, production status, environment_prompt, sound_prompt.  
   - Use Google Sheets OAuth2 credentials.

7. **Add AI Agent: Generate Detailed Video Prompts:**  
   - Input fields: idea, environment, sound from sheet or previous node.  
   - Prompt to produce 3+ cinematic, macro-level cutting scenes with detailed descriptions.  
   - Use Think tool for validation.

8. **Add LLM Node (GPT-4.1) to assist prompt drafting:**  
   - Connect to AI Agent node.

9. **Add Tool: Refine and Validate Prompts (Think Tool):**  
   - Connect to AI Agent node.

10. **Add Parse Structured Video Prompt Output Node:**  
    - Parse detailed multi-scene video prompt JSON.

11. **Add Code Node to Extract Individual Scene Descriptions:**  
    - Use provided JavaScript to recursively find all "scene" keys and output an array of scene descriptions.

12. **Add HTTP Request Node to Generate Video Clips (Wavespeed AI):**  
    - POST to Wavespeed API with prompt combining idea and individual scene descriptions.  
    - Set aspect ratio 9:16, duration 10 seconds.  
    - Use HTTP Header authentication with Wavespeed API key.

13. **Add Wait Node:**  
    - Wait 240 seconds for video generation.

14. **Add HTTP Request Node to Retrieve Video Clips:**  
    - GET request using job ID from generation node.

15. **Add HTTP Request Node to Generate ASMR Sound (Fal AI):**  
    - POST with prompt referencing AI sound description and video clip URL.  
    - Use HTTP Header authentication with Fal AI API key.

16. **Add Wait Node:**  
    - Wait 60 seconds for sound generation.

17. **Add HTTP Request Node to Retrieve Final Sound Output:**  
    - GET request using sound generation request ID.

18. **Add Code Node to List Clip URLs for Stitching:**  
    - Aggregate all clip URLs into an array.

19. **Add HTTP Request Node to Merge Clips (Fal AI):**  
    - POST to Fal AI ffmpeg API to stitch clips sequentially.

20. **Add Wait Node:**  
    - Wait 60 seconds for video rendering.

21. **Add HTTP Request Node to Retrieve Final Merged Video:**  
    - GET request for final stitched video URL.

22. **Add Google Sheets Update Node:**  
    - Update the row with final video URL and production status "done".

23. **Add Google Sheets Read Node to Get Video Metadata:**  
    - Retrieve video URLs and descriptions for publishing.

24. **Add Set Node to Assign Social Media Account IDs:**  
    - Populate JSON with all target platform IDs (Instagram, YouTube, TikTok, etc.).

25. **Add HTTP Request Node to Upload Video to Blotato:**  
    - POST video URL to Blotato media endpoint with API key header.

26. **For each social media platform (Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Bluesky, Pinterest):**  
    - Add HTTP Request nodes with POST to Blotato posts endpoint.  
    - Use assigned account IDs, video URL, description, and platform-specific parameters.  
    - Set API key header.

27. **Add Google Sheets Update Node to mark production status as "Publish".**

28. **Ensure all nodes are connected in the correct order as per the workflow connections.**

29. **Configure all credentials (OpenAI API, Google Sheets OAuth2, Wavespeed API key, Fal AI API key, Blotato API key).**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow integrates AI-generated concepts with Wavespeed and Fal AI for video and sound generation.              | Workflow purpose                                                                                 |
| Blotato API is used to upload and distribute videos across multiple social media platforms programmatically.         | API documentation: https://backend.blotato.com/docs                                           |
| The workflow uses LangChain AI agents with Think tool to validate and refine AI outputs before proceeding.           | LangChain docs: https://docs.langchain.com                                                     |
| Google Sheets is used as a metadata store to track ideas, prompts, production status, and final video URLs.          | Google Sheets API and OAuth2 credentials required                                             |
| Video clips are generated in 9:16 aspect ratio, suitable for vertical video platforms like TikTok and Instagram.     |                                                                                                 |
| Wait nodes with generous timeouts are essential to handle asynchronous AI generation latency for clips and sounds. | Adjust wait durations based on actual API performance and quota limits                           |
| Hardcoded social media IDs in the Set node must be replaced with real account identifiers before publishing.         | Replace placeholder IDs before deployment                                                      |
| API keys for Wavespeed, Fal AI, OpenAI, and Blotato must be securely stored and configured in n8n credentials.       |                                                                                                 |
| The workflow is inactive by default; activate it to enable scheduling.                                               |                                                                                                 |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---