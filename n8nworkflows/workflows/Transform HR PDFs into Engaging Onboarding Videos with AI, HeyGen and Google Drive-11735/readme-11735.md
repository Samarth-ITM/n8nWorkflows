Transform HR PDFs into Engaging Onboarding Videos with AI, HeyGen and Google Drive

https://n8nworkflows.xyz/workflows/transform-hr-pdfs-into-engaging-onboarding-videos-with-ai--heygen-and-google-drive-11735


# Transform HR PDFs into Engaging Onboarding Videos with AI, HeyGen and Google Drive

### 1. Workflow Overview

This workflow automates the transformation of HR policy PDFs into engaging onboarding videos using AI services, HeyGen video generation, and Google Drive storage. It is designed for HR teams or companies wanting to streamline the creation of clear, friendly onboarding content from complex policy documents.

The workflow is logically divided into the following blocks:

- **1.1 PDF Intake & Extraction:** Receives the HR policy PDF via webhook, downloads it, and extracts text content for AI processing.
- **1.2 Policy Simplification AI:** Uses an AI agent to simplify and restructure the raw policy text into clear, structured onboarding instructions in JSON format.
- **1.3 Script Generation AI:** Transforms the simplified onboarding content into a natural, conversational video narration script suitable for HeyGen avatars.
- **1.4 Video Creation Task:** Sends the generated script to HeyGen to create a video, then polls the video status until completion.
- **1.5 Video Download & Delivery:** Downloads the completed video from HeyGen, uploads it to Google Drive, and responds to the webhook with the video file for client use.
- **1.6 Credentials & Security:** Ensures secure connections to Google Drive, HeyGen, and OpenAI APIs with OAuth2 and API keys.

---

### 2. Block-by-Block Analysis

#### 2.1 PDF Intake & Extraction

**Overview:**  
This block handles receiving the HR policy PDF from an external source via webhook, downloading the file, and extracting its text content to prepare for AI-based processing.

**Nodes Involved:**  
- Webhook - Receive Policy PDF  
- Dowload file  
- Extract from File  

**Node Details:**

- **Webhook - Receive Policy PDF**  
  - Type: Webhook  
  - Role: Entry point; receives HTTP POST with PDF file or link.  
  - Config: Path set to `hr-onboarding-upload`; accepts POST; binary property name is `data`.  
  - Inputs: External HTTP request.  
  - Outputs: File URL or binary data forwarded.  
  - Edge cases: Incorrect file format or missing data; HTTP errors.  

- **Dowload file**  
  - Type: HTTP Request  
  - Role: Downloads the PDF file from URL provided in webhook data.  
  - Config: URL taken from incoming JSON body field `file_url`; response expected as file.  
  - Inputs: Webhook output JSON with `file_url`.  
  - Outputs: Binary PDF file to next node.  
  - Edge cases: Network errors, invalid URLs, file inaccessible.  

- **Extract from File**  
  - Type: ExtractFromFile (PDF operation)  
  - Role: Extracts text content from the downloaded PDF file.  
  - Config: Operation set to `pdf`.  
  - Inputs: Binary PDF file from previous node.  
  - Outputs: JSON with extracted text under `$json.text`.  
  - Edge cases: Corrupted PDF, extraction failure, unsupported content formats.

---

#### 2.2 Policy Simplification AI

**Overview:**  
Simplifies complex HR policy text into a structured, employee-friendly onboarding JSON format using a LangChain AI agent and an OpenAI GPT-4 model.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Structured Output Parser  
- Simple Memory1  

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Processes extracted policy text to simplify and structure it.  
  - Config: Prompt instructs simplification into clear onboarding instructions, friendly tone, accurate and compliant, structured output in JSON only.  
  - Inputs: Extracted text (`$json.text`) from "Extract from File".  
  - Outputs: Simplified policy JSON with sections like summary, key points, onboarding instructions, compliance requirements.  
  - Edge cases: AI response format errors, prompt misinterpretation, API failures.  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4.1-mini language model for AI Agent processing.  
  - Config: Model set to `gpt-4.1-mini`; connected to OpenAI API credential.  
  - Inputs: AI Agent calls language model.  
  - Outputs: AI text completion.  
  - Edge cases: API quota exceeded, network issues, version compatibility.  

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates and enforces AI output JSON schema for policy simplification.  
  - Config: JSON schema example contains summary, key_points, onboarding_instructions, compliance_requirements arrays.  
  - Inputs: AI Agent raw output.  
  - Outputs: Parsed, validated JSON data for next steps.  
  - Edge cases: AI output not matching schema; parsing errors.  

- **Simple Memory1**  
  - Type: LangChain Memory Buffer Window  
  - Role: Stores AI conversation history with custom session key `"description writer"` to maintain context.  
  - Inputs/Outputs: Connects as AI memory for the AI Agent node.  
  - Edge cases: Memory overflow or context drift.

---

#### 2.3 Script Generation AI

**Overview:**  
Generates a professional, conversational onboarding video script from the simplified policy JSON, formatted for HeyGen video avatars.

**Nodes Involved:**  
- Generate Script With AI  
- OpenAI Chat Model1  
- Structured Output Parser1  
- Simple Memory  

**Node Details:**

- **Generate Script With AI**  
  - Type: LangChain Agent  
  - Role: Converts structured onboarding content into a natural narration script with sections and full script text.  
  - Config: Prompt instructs conversational, friendly tone, warm introduction, clear sections, subtle pause markers, no bullet points; returns JSON only with video_title, intro, script_sections, closing, full_script.  
  - Inputs: Simplified policy JSON from previous block.  
  - Outputs: Video script JSON for HeyGen.  
  - Edge cases: AI output formatting errors, missing sections, API failures.  

- **OpenAI Chat Model1**  
  - Type: LangChain OpenAI Chat Model  
  - Role: GPT-4.1-mini model used for script generation.  
  - Config: Same as prior OpenAI node; connected to OpenAI API.  
  - Inputs: Agent prompts.  
  - Outputs: Text completions.  
  - Edge cases: Rate limits, connectivity.  

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates script JSON schema with fields for video_title, intro, script_sections, closing, full_script.  
  - Inputs: Raw AI script output.  
  - Outputs: Parsed JSON for video creation.  
  - Edge cases: Parsing failures, schema mismatches.  

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Stores conversation memory for the script generation agent with same session key `"description writer"`.  
  - Inputs/Outputs: Connects as AI memory for Generate Script With AI.  
  - Edge cases: Context drift.

---

#### 2.4 Video Creation Task

**Overview:**  
Submits the generated video script to HeyGen API, starts video generation, then periodically checks video status with timed wait loops until the video is completed.

**Nodes Involved:**  
- Create HeyGen Video Task1  
- Wait 10 Seconds  
- Check HeyGen Video Status  
- Wait  
- Check If Video Completed  

**Node Details:**

- **Create HeyGen Video Task1**  
  - Type: HTTP Request  
  - Role: Sends POST request to HeyGen API to start video generation with avatar, voice, background, and script.  
  - Config: Uses API endpoint `https://api.heygen.com/v2/video/generate`; includes headers with API key; body includes video title and full_script from previous node.  
  - Inputs: Script JSON from Generate Script With AI.  
  - Outputs: JSON response containing video ID.  
  - Edge cases: API key errors, malformed request, rate limiting.  

- **Wait 10 Seconds**  
  - Type: Wait Node  
  - Role: Pauses workflow execution for 60 seconds before checking video status again.  
  - Config: Wait time set to 60 seconds.  
  - Inputs: From Create HeyGen Video Task1 or Check If Video Completed (loop).  
  - Outputs: Triggers next status check.  

- **Check HeyGen Video Status**  
  - Type: HTTP Request  
  - Role: Queries HeyGen API for current video generation status using video ID.  
  - Config: GET request to `https://api.heygen.com/v1/video_status.get` with query parameter `video_id`; includes API key in headers.  
  - Inputs: Video ID from Create HeyGen Video Task1 or loop.  
  - Outputs: JSON with video status (e.g., "completed", "processing").  
  - Edge cases: API errors, invalid video ID.  

- **Wait**  
  - Type: Wait Node  
  - Role: Additional 60 seconds delay after status check (used after Check HeyGen Video Status).  
  - Config: Wait time 60 seconds.  
  - Inputs: From Check HeyGen Video Status.  
  - Outputs: Loops back to Check If Video Completed.  

- **Check If Video Completed**  
  - Type: If Node  
  - Role: Evaluates if video status equals "completed"; routes accordingly.  
  - Config: Condition checks if `$json.data.status === "completed"`.  
  - Inputs: Status JSON from Check HeyGen Video Status.  
  - Outputs:  
    - True: proceeds to download video.  
    - False: loops back to Wait 10 Seconds to re-check.  
  - Edge cases: Unexpected status values, logic loops.

---

#### 2.5 Video Download & Delivery

**Overview:**  
Downloads the completed video file from HeyGen, uploads it to a specified Google Drive folder, and sends the video file back via webhook response for client consumption.

**Nodes Involved:**  
- Download Completed Video  
- Upload Video to Google Drive  
- Download file1  
- Respond to Webhook  

**Node Details:**

- **Download Completed Video**  
  - Type: HTTP Request  
  - Role: Downloads the generated video file from HeyGen using the video URL returned in status.  
  - Config: URL taken from `$json.data.video_url`; response expected as a file (binary).  
  - Inputs: Video status JSON from Check If Video Completed node (True branch).  
  - Outputs: Binary video file.  
  - Edge cases: Network issues, URL expiration.  

- **Upload Video to Google Drive**  
  - Type: Google Drive Node  
  - Role: Uploads downloaded video file to a Google Drive folder.  
  - Config:  
    - File name pattern: `HR_Onboarding_Video_{{ $now.format('yyyy-MM-dd_HHmmss') }}.mp4`  
    - Drive: "My Drive"  
    - Folder ID: preset folder `"1zm4CTaf9mcX34-Bsk9dzQ7r02BROObOO"`  
    - Credential: OAuth2 Google Drive connection (Techdome Account)  
  - Inputs: Binary video file from Download Completed Video.  
  - Outputs: Confirmation and file metadata.  
  - Edge cases: OAuth token expiration, upload failures.  

- **Download file1**  
  - Type: Google Drive Node  
  - Role: Downloads the uploaded video file by ID (passed from Upload Video to Google Drive) to prepare for webhook response.  
  - Config: Operation "download" by file ID from previous node.  
  - Inputs: File ID from Upload Video to Google Drive output.  
  - Outputs: Binary video file.  
  - Edge cases: File not found, permission denied.  

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends the final video file binary back to the original webhook caller.  
  - Config: Respond with binary data source set from Download file1.  
  - Inputs: Binary file from Download file1.  
  - Outputs: HTTP response to client.  
  - Edge cases: Large file size, client timeout, response errors.

---

#### 2.6 Credentials & Security

**Overview:**  
Manages secure API connections and credentials for Google Drive, HeyGen, and OpenAI.

**Nodes Involved:**  
- None directly for credentials; credentials referenced in respective nodes.

**Notes:**  
- Google Drive OAuth2 credential named "Techdome Account" is used for Drive operations.  
- HeyGen API key placeholders must be replaced with valid keys.  
- OpenAI API keys configured in LangChain OpenAI Chat Model nodes.  
- Avoid storing API keys directly in workflow exports; use environment variables or n8n credential management.  
- OAuth2 token refresh handled by n8n Google Drive node.  
- Secure webhook endpoint configured with path `hr-onboarding-upload`.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                           | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                                                                           |
|---------------------------|----------------------------------|-----------------------------------------|-------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook - Receive Policy PDF | Webhook                          | Receives HR policy PDF upload            | —                             | Dowload file                  | ## PDF Intake & Extraction<br>Handles the upload of the employee policy PDF and extracts readable text for AI processing. Ensures the document is correctly retrieved. |
| Dowload file              | HTTP Request                     | Downloads PDF from URL                    | Webhook - Receive Policy PDF   | Extract from File             | ## PDF Intake & Extraction (see above)                                                                                                                             |
| Extract from File         | ExtractFromFile (PDF operation)  | Extracts text from PDF                    | Dowload file                  | AI Agent                     | ## PDF Intake & Extraction (see above)                                                                                                                             |
| AI Agent                 | LangChain AI Agent               | Simplifies policy text into onboarding JSON | Extract from File             | Structured Output Parser       | ## Policy Simplification AI<br>Converts raw policy text into clear, structured onboarding content. Uses JSON schema for downstream use.                              |
| OpenAI Chat Model         | LangChain OpenAI Chat Model      | GPT-4.1-mini language model for AI Agent | AI Agent (ai_languageModel)   | AI Agent                     | ## Policy Simplification AI (see above)                                                                                                                           |
| Structured Output Parser  | LangChain Structured Output Parser | Validates simplified policy JSON         | AI Agent                     | AI Agent                     | ## Policy Simplification AI (see above)                                                                                                                           |
| Simple Memory1            | LangChain Memory Buffer Window   | Maintains AI conversation context        | AI Agent (ai_memory)          | AI Agent                     | ## Policy Simplification AI (see above)                                                                                                                           |
| Generate Script With AI    | LangChain AI Agent               | Creates natural onboarding video script  | Structured Output Parser       | Create HeyGen Video Task1      | ## Script Generation AI<br>Transforms simplified policy into natural video narration script optimized for HeyGen.                                                  |
| OpenAI Chat Model1         | LangChain OpenAI Chat Model      | GPT-4.1-mini model for script generation | Generate Script With AI (ai_languageModel) | Generate Script With AI       | ## Script Generation AI (see above)                                                                                                                               |
| Structured Output Parser1  | LangChain Structured Output Parser | Validates video script JSON               | Generate Script With AI       | Generate Script With AI       | ## Script Generation AI (see above)                                                                                                                               |
| Simple Memory             | LangChain Memory Buffer Window   | Context memory for script generation      | Generate Script With AI (ai_memory) | Generate Script With AI       | ## Script Generation AI (see above)                                                                                                                               |
| Create HeyGen Video Task1 | HTTP Request                     | Starts HeyGen video generation task       | Generate Script With AI        | Wait 10 Seconds              | ## Video Creation Task<br>Submits script to HeyGen and monitors generation status.                                                                                  |
| Wait 10 Seconds           | Wait Node                       | Delays 60 seconds before status check     | Create HeyGen Video Task1 / Check If Video Completed (False branch) | Check HeyGen Video Status    | ## Video Creation Task (see above)                                                                                                                                |
| Check HeyGen Video Status | HTTP Request                     | Checks HeyGen video generation status     | Wait 10 Seconds               | Wait / Check If Video Completed | ## Video Creation Task (see above)                                                                                                                                |
| Wait                      | Wait Node                       | Additional 60 seconds delay after status check | Check HeyGen Video Status     | Check If Video Completed      | ## Video Creation Task (see above)                                                                                                                                |
| Check If Video Completed  | If Node                        | Evaluates if video status is "completed" | Check HeyGen Video Status     | Download Completed Video / Wait 10 Seconds | ## Video Creation Task (see above)                                                                                                                                |
| Download Completed Video  | HTTP Request                     | Downloads finished video file from HeyGen | Check If Video Completed (True) | Upload Video to Google Drive | ## Video Download & Delivery<br>Downloads HeyGen video for storage and response.                                                                                    |
| Upload Video to Google Drive | Google Drive Node                | Uploads video file to Google Drive folder | Download Completed Video      | Download file1               | ## Video Download & Delivery (see above)                                                                                                                          |
| Download file1            | Google Drive Node                | Downloads uploaded video for webhook response | Upload Video to Google Drive  | Respond to Webhook           | ## Video Download & Delivery (see above)                                                                                                                          |
| Respond to Webhook        | Respond to Webhook Node          | Sends video file back to webhook caller   | Download file1                | —                             | ## Video Download & Delivery (see above)                                                                                                                          |
| Sticky Note               | Sticky Note                     | Workflow overview and setup instructions  | —                             | —                             | ## HR Onboarding Video Generator<br>Workflow description, setup steps, and testing instructions.                                                                   |
| Sticky Note1              | Sticky Note                     | PDF Intake & Extraction description       | —                             | —                             | ## PDF Intake & Extraction (see above)                                                                                                                            |
| Sticky Note2              | Sticky Note                     | Policy Simplification AI description       | —                             | —                             | ## Policy Simplification AI (see above)                                                                                                                           |
| Sticky Note3              | Sticky Note                     | Script Generation AI description            | —                             | —                             | ## Script Generation AI (see above)                                                                                                                               |
| Sticky Note4              | Sticky Note                     | Video Creation Task description             | —                             | —                             | ## Video Creation Task (see above)                                                                                                                                |
| Sticky Note5              | Sticky Note                     | Video Download & Delivery description       | —                             | —                             | ## Video Download & Delivery (see above)                                                                                                                          |
| Sticky Note6              | Sticky Note                     | Credentials & Security notes                 | —                             | —                             | ## Credentials & Security<br>Use OAuth2 and API keys securely. Avoid embedding secrets directly.                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Configure path: `hr-onboarding-upload`  
   - HTTP Method: POST  
   - Accept binary data property named `data` (for file upload)  
   - Output: Pass JSON body containing `file_url` or binary data onward.  

2. **Add HTTP Request Node to Download File**  
   - Type: HTTP Request  
   - URL: `={{ $json.body.file_url }}` (from webhook JSON)  
   - Response format: File (binary)  
   - Connect input from Webhook node.  

3. **Add Extract From File Node**  
   - Type: ExtractFromFile  
   - Operation: pdf  
   - Input: Binary file from previous node  
   - Output: Extracted text in JSON field `$json.text`  

4. **Add LangChain AI Agent Node (Policy Simplification)**  
   - Type: LangChain Agent  
   - Prompt: Simplify company policy into clear onboarding instructions (friendly tone, accurate, structured JSON output)  
   - Input: `$json.text` from Extract From File  
   - Configure system message and output parser schema as per policy simplification requirements  
   - Connect AI memory node with session key `"description writer"`  
   - Connect OpenAI Chat Model (GPT-4.1-mini) node as language model  

5. **Add LangChain Structured Output Parser Node**  
   - Type: Output Parser Structured  
   - Use JSON schema example with summary, key_points, onboarding_instructions, compliance_requirements  
   - Input: AI Agent raw output  

6. **Add LangChain AI Agent Node (Script Generation)**  
   - Type: LangChain Agent  
   - Prompt: Generate a professional, friendly onboarding video script with sections and full narration, formatted for HeyGen  
   - Input: Parsed JSON from previous node  
   - Configure system message and output parser schema to produce video_title, intro, script_sections, closing, full_script  
   - Connect AI memory node with same session key `"description writer"`  
   - Connect OpenAI Chat Model node for language model  

7. **Add LangChain Structured Output Parser Node for Script**  
   - Type: Output Parser Structured  
   - Schema: video_title, intro, script_sections, closing, full_script  
   - Input: Generate Script With AI output  

8. **Add HTTP Request Node to Create HeyGen Video Task**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Headers: Accept JSON, `x-api-key` with HeyGen API key  
   - Body (JSON): Includes avatar_id, voice_id, background color, video title, full script text, and video dimension (1280x720)  
   - Input: Script JSON from script generation parser  

9. **Add Wait Node (Wait 10 Seconds)**  
   - Type: Wait  
   - Duration: 60 seconds  
   - Input: From Create HeyGen Video Task and loop back from Check If Video Completed (False branch)  

10. **Add HTTP Request Node to Check HeyGen Video Status**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.heygen.com/v1/video_status.get`  
    - Query parameter: video_id from previous response  
    - Headers: Accept JSON, `x-api-key` with HeyGen API key  
    - Input: From Wait 10 Seconds node  

11. **Add Wait Node (Wait)**  
    - Type: Wait  
    - Duration: 60 seconds  
    - Input: From Check HeyGen Video Status  

12. **Add If Node (Check If Video Completed)**  
    - Type: If  
    - Condition: `$json.data.status === "completed"` (string equals)  
    - True branch: proceeds to download video  
    - False branch: loops to Wait 10 Seconds node  

13. **Add HTTP Request Node to Download Completed Video**  
    - Type: HTTP Request  
    - URL: `$json.data.video_url` from completed status response  
    - Response format: File (binary)  
    - Input: From If node (True branch)  

14. **Add Google Drive Node to Upload Video**  
    - Type: Google Drive (upload)  
    - File name: `HR_Onboarding_Video_{{ $now.format('yyyy-MM-dd_HHmmss') }}.mp4`  
    - Drive: My Drive  
    - Folder ID: your target folder ID for onboarding videos  
    - Credential: Google Drive OAuth2 (configured with your account)  
    - Input: Binary file from Download Completed Video  

15. **Add Google Drive Node to Download Uploaded File**  
    - Type: Google Drive (download)  
    - File ID: from Upload Video to Google Drive output  
    - Credential: same as above  
    - Input: From upload node  

16. **Add Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Respond with: Binary data from Download file1 node  
    - Input: From Download file1 node  

17. **Set Up Credentials**  
    - Google Drive OAuth2 with required API scopes for file upload and download  
    - HeyGen API key (replace placeholder)  
    - OpenAI API key configured for LangChain nodes  

18. **Test Workflow**  
    - Upload a sample HR policy PDF via webhook URL  
    - Monitor extraction, AI simplification, script generation  
    - Confirm HeyGen video task creation and completion polling  
    - Verify video upload to Google Drive and response delivery  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow transforms HR policy PDFs into onboarding videos with AI simplification, script generation, HeyGen.  | Workflow use case summary.                                                                                                                                           |
| Replace placeholder values for HeyGen API key, avatar ID, voice ID with your actual credentials.               | Critical for successful video generation.                                                                                                                           |
| Use OAuth2 credential for Google Drive integration to ensure secure, token-managed access.                     | OAuth2 recommended for Drive nodes; token refresh handled by n8n.                                                                                                   |
| Webhook path `hr-onboarding-upload` must be exposed and secured for PDF upload.                               | Secure endpoint recommended to avoid unauthorized access.                                                                                                          |
| AI prompts are designed for compliance, clarity, and friendly tone, maintaining policy accuracy.               | Ensures professional and compliant output.                                                                                                                          |
| Workflow includes loop with wait nodes to poll HeyGen video generation status every 60 seconds until completion. | Prevents excessive API calls and respects rate limits.                                                                                                             |
| Output JSON schemas enforce structured AI outputs for reliable downstream processing.                          | Reduces parsing errors and enforces contract between AI and workflow.                                                                                              |
| For advanced customization, update AI prompts or JSON schemas as needed to fit company-specific policies.     | Allows tailoring tone, structure, or onboarding content.                                                                                                           |
| Workflow tested with OpenAI GPT-4.1-mini model; API version compatibility required for LangChain nodes.       | Verify n8n and LangChain package versions support these nodes.                                                                                                     |
| [HeyGen API Documentation](https://docs.heygen.com) for avatar and voice IDs, video generation parameters.    | Reference for video creation API parameters and capabilities.                                                                                                       |
| [n8n Google Drive Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/) | Reference to configure Google Drive file upload and download nodes.                                                                                                |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.