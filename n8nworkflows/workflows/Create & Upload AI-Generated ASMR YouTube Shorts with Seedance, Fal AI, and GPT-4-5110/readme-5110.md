Create & Upload AI-Generated ASMR YouTube Shorts with Seedance, Fal AI, and GPT-4

https://n8nworkflows.xyz/workflows/create---upload-ai-generated-asmr-youtube-shorts-with-seedance--fal-ai--and-gpt-4-5110


# Create & Upload AI-Generated ASMR YouTube Shorts with Seedance, Fal AI, and GPT-4

---

### 1. Workflow Overview

This automated n8n workflow titled **"Automated AI YouTube Shorts Factory for ASMR (Seedance)"** is designed to autonomously create and publish ASMR-style YouTube Shorts videos leveraging multiple AI services. It targets creators or automation enthusiasts aiming to produce viral ASMR content without manual intervention, using a combination of AI-generated ideas, video clips, sound effects, and final video assembly.

The workflow logically divides into four main functional blocks:

**1.1 AI Ideation**  
- Begins with a scheduled trigger to periodically start the process  
- Generates a short, trendy ASMR video idea using AI (GPT-4)  
- Expands this idea into a detailed production plan with captions, environment descriptions, and sound prompts

**1.2 Asset Generation**  
- Parses detailed scene descriptions from the production plan  
- Uses Seedance AI (via Wavespeed API) to generate video clips for each scene  
- Calls Fal AI to produce corresponding ASMR sound effects  
- This block runs video and audio generation in parallel

**1.3 Final Assembly**  
- Sequences the multiple video clips into one continuous video using Fal AI’s video composition API  
- Waits for video processing and retrieves the final combined video file

**1.4 Distribution & Logging**  
- Downloads the final video file  
- Uploads it to YouTube with metadata derived from the AI-generated plan  
- Updates a Google Sheet to log the production status and YouTube URL  
- Sends notifications via Telegram and Gmail to alert about the new video publication

---

### 2. Block-by-Block Analysis

---

#### 1.1 AI Ideation

**Overview:**  
This block generates a viral ASMR video concept and expands it into a detailed production plan with captions, environment, and sound prompts using AI agents.

**Nodes Involved:**  
- Schedule Trigger  
- 1. Generate Trendy Idea  
- OpenAI Chat Model1  
- 2. Enrich Idea into Plan  
- Think (Langchain tool)  
- Parser  
- 3. Log New Idea to Sheet

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow every 30 minutes automatically  
  - Configuration: Interval set to 30 minutes  
  - Inputs: None  
  - Outputs: Triggers "1. Generate Trendy Idea"  
  - Edge cases: Missed triggers if n8n instance is down

- **1. Generate Trendy Idea**  
  - Type: Langchain Agent (AI prompt agent)  
  - Role: Produces a short, trendy ASMR video idea (under 10 words)  
  - Configuration: System message restricts output to a single line plain text idea; no extra words or formatting  
  - Inputs: Trigger from Schedule node  
  - Outputs: Single idea text, sent to "2. Enrich Idea into Plan"  
  - Credentials: OpenAI API (GPT-4)  
  - Edge cases: API rate limits, malformed prompt output

- **OpenAI Chat Model1**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Underlying language model for AI agents  
  - Configuration: GPT-4.1 model specified  
  - Inputs: Connected from "1. Generate Trendy Idea" node  
  - Outputs: AI-generated text to "2. Enrich Idea into Plan"  
  - Credentials: OpenAI API  
  - Edge cases: Authentication errors, API timeouts

- **2. Enrich Idea into Plan**  
  - Type: Langchain Agent  
  - Role: Transforms the short idea into a full production plan JSON including Idea, Caption, Environment, Sound, and Status fields  
  - Configuration: Detailed system message enforcing JSON array output with strict formatting rules and viral content criteria  
  - Inputs: Idea from previous node, uses Think tool for internal review  
  - Outputs: Structured JSON production plan to "3. Log New Idea to Sheet" and "Prompts AI Agent"  
  - Credentials: OpenAI API  
  - Edge cases: Parsing failures if AI output deviates from expected JSON format

- **Think**  
  - Type: Langchain Tool (Think)  
  - Role: Used internally by AI agents to reflect on and improve output quality  
  - Inputs/Outputs: Connected with "2. Enrich Idea into Plan" and "Prompts AI Agent"  
  - Edge cases: Tool failure could reduce AI output quality but not break workflow

- **Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI JSON output from "2. Enrich Idea into Plan" to a usable JSON format  
  - Configuration: Example JSON schema provided for validation  
  - Inputs: AI raw output JSON string  
  - Outputs: Parsed JSON object for downstream nodes  
  - Edge cases: Parsing errors if AI output is malformed

- **3. Log New Idea to Sheet**  
  - Type: Google Sheets Node  
  - Role: Appends the newly generated idea and metadata to a Google Sheet as the content management system (CMS)  
  - Configuration: Maps columns "idea," "caption," "production," "sound_prompt," "environment_prompt"  
  - Inputs: Parsed AI output JSON  
  - Outputs: Passes data to "Prompts AI Agent"  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Authentication errors, API quotas, sheet misconfiguration

---

#### 1.2 Asset Generation

**Overview:**  
Generates detailed scene prompts, creates video clips via Seedance AI, and produces ASMR sounds via Fal AI concurrently.

**Nodes Involved:**  
- Prompts AI Agent  
- Parser2  
- Unbundle Prompts  
- Create Clips (Seedance AI)  
- Wait for Clips  
- Get Clips  
- Create Sounds (Fal AI)  
- Wait for Sounds  
- Get Sounds  
- List Elements  
- Sequence Video (Fal AI)  
- Wait for Final Video  
- Get Final Video

**Node Details:**

- **Prompts AI Agent**  
  - Type: Langchain Agent  
  - Role: Generates detailed multi-scene video prompts based on the enriched production plan  
  - Configuration: System message defines strict style rules for ASMR scene descriptions including kinetic sand interactions  
  - Inputs: Output from "3. Log New Idea to Sheet"  
  - Outputs: Multi-scene JSON to "Unbundle Prompts"  
  - Credentials: OpenAI API  
  - Edge cases: AI output compliance with expected format

- **Parser2**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses the multi-scene prompt JSON from the AI agent  
  - Inputs: Raw AI output JSON string from "Prompts AI Agent"  
  - Outputs: Parsed JSON for video clip generation  
  - Edge cases: Parsing errors

- **Unbundle Prompts**  
  - Type: Code Node (JavaScript)  
  - Role: Extracts individual scene descriptions from nested JSON to create separate items for clip creation  
  - Configuration: Custom JS recursively finds keys starting with "scene" and outputs array of descriptions  
  - Inputs: Parsed AI multi-scene prompt  
  - Outputs: Each scene description item sent to "Create Clips"  
  - Edge cases: Missing scene keys throws error

- **Create Clips**  
  - Type: HTTP Request (POST)  
  - Role: Calls Wavespeed AI Seedance API to create 10-second video clips per scene prompt  
  - Configuration: Sends JSON body with aspect ratio 9:16, duration 10s, and prompt combining Idea, scene description, Environment  
  - Inputs: Scene description items from "Unbundle Prompts"  
  - Outputs: Request IDs for clip generation  
  - Authentication: Generic HTTP header auth (Wavespeed API key)  
  - Edge cases: API rate limits, invalid prompt causes no clip generated

- **Wait for Clips**  
  - Type: Wait Node  
  - Role: Delays further execution for 120 seconds to allow Seedance clip generation  
  - Inputs: After "Create Clips"  
  - Outputs: To "Get Clips"  
  - Edge cases: Insufficient wait time may cause premature fetching

- **Get Clips**  
  - Type: HTTP Request (GET)  
  - Role: Polls Wavespeed API for clip generation results using the request IDs  
  - Inputs: After "Wait for Clips"  
  - Outputs: Clip URLs to "Create Sounds"  
  - Edge cases: API errors, clip generation failures

- **Create Sounds**  
  - Type: HTTP Request (POST)  
  - Role: Calls Fal AI sound generation API to produce 10-second ASMR audio clips matching sound prompts  
  - Configuration: Sends prompt text combined with sound description from AI, includes video_url to sync sound  
  - Inputs: Clip URLs from "Get Clips"  
  - Outputs: Request IDs for sound generation  
  - Authentication: Generic HTTP header auth (Fal AI API key)  
  - Edge cases: API rate limits or malformed prompts

- **Wait for Sounds**  
  - Type: Wait Node  
  - Role: Waits 60 seconds for sound generation to complete  
  - Inputs: After "Create Sounds"  
  - Outputs: To "Get Sounds"  
  - Edge cases: Insufficient wait time causes premature polling

- **Get Sounds**  
  - Type: HTTP Request (GET)  
  - Role: Polls Fal AI sound API to retrieve completed sound effect URLs  
  - Inputs: After "Wait for Sounds"  
  - Outputs: Sound URLs to "List Elements"  
  - Edge cases: API errors, timeouts

- **List Elements**  
  - Type: Code Node  
  - Role: Aggregates all video clip URLs (from "Get Sounds") into a single array object to prepare for video sequencing  
  - Inputs: Sound URLs with video references  
  - Outputs: Video URLs array to "Sequence Video" node  
  - Edge cases: Empty input list causes failure

- **Sequence Video**  
  - Type: HTTP Request (POST)  
  - Role: Calls Fal AI video composition API to stitch multiple clips into one final video track  
  - Configuration: Sends JSON with ordered video track keyframes (timestamps, durations, URLs)  
  - Inputs: Video URLs array from "List Elements"  
  - Outputs: Request ID for final video processing  
  - Authentication: Generic HTTP header auth (Fal AI API key)  
  - Edge cases: API errors, invalid URLs, video format incompatibilities

- **Wait for Final Video**  
  - Type: Wait Node  
  - Role: Waits 60 seconds for final video to be assembled  
  - Inputs: After "Sequence Video"  
  - Outputs: To "Get Final Video"  
  - Edge cases: Premature polling

- **Get Final Video**  
  - Type: HTTP Request (GET)  
  - Role: Polls Fal AI API to retrieve completed final video URL  
  - Inputs: After "Wait for Final Video"  
  - Outputs: Video URL to "Update Final Video to Sheet"  
  - Edge cases: API failures or timeouts

---

#### 1.3 Final Assembly

**Overview:**  
Downloads the final video file and uploads it to YouTube with metadata, updating Google Sheet and sending notifications.

**Nodes Involved:**  
- Update Final Video to Sheet  
- Download Final Video  
- Upload to YouTube  
- Update Sheet with Youtube Link  
- Telegram Notification  
- Gmail Notification

**Node Details:**

- **Update Final Video to Sheet**  
  - Type: Google Sheets Node  
  - Role: Updates the same Google Sheet row marking production as "Done" and saves the final video URL  
  - Configuration: Uses "idea" as key to find row, updates columns "production" and "final_output"  
  - Inputs: Final video URL from "Get Final Video"  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Sheet access errors

- **Download Final Video**  
  - Type: HTTP Request (GET)  
  - Role: Downloads the final video file from the URL provided by "Get Final Video"  
  - Configuration: Response type set to file stream  
  - Inputs: Final video URL from "Update Final Video to Sheet"  
  - Outputs: Binary data to "Upload to YouTube"  
  - Edge cases: Download failures, network errors

- **Upload to YouTube**  
  - Type: YouTube Node (Upload Video)  
  - Role: Uploads the final video to YouTube with title, description, tags, privacy status, and subscriber notifications  
  - Configuration: Title and description dynamically set from idea and caption; tags are fixed ASMR-related hashtags  
  - Inputs: Video binary data from "Download Final Video"  
  - Credentials: OAuth2 YouTube API  
  - Edge cases: Authentication errors, quota exceeded, upload failures

- **Update Sheet with Youtube Link**  
  - Type: Google Sheets Node  
  - Role: Updates the same Google Sheet row with the YouTube video URL after upload  
  - Configuration: Uses "idea" as key, updates "youtube_url" column with full YouTube link  
  - Inputs: YouTube video ID from "Upload to YouTube"  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Sheet update failures

- **Telegram Notification**  
  - Type: Telegram Node  
  - Role: Sends a message to a specified Telegram chat notifying that the video is ready with a clickable YouTube link  
  - Configuration: Chat ID to be set by user, message formatted in HTML  
  - Inputs: YouTube video ID and idea from "Upload to YouTube" and sheet nodes  
  - Credentials: Telegram Bot API token  
  - Edge cases: Invalid chat ID, message sending failure

- **Gmail Notification**  
  - Type: Gmail Node  
  - Role: Sends an email notifying the user that the new video is live with a link  
  - Configuration: Recipient email, subject, and message text including YouTube link  
  - Inputs: YouTube video ID and idea from "Upload to YouTube" and sheet nodes  
  - Credentials: Gmail OAuth2  
  - Edge cases: SMTP errors, authentication failures

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                                          | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                      |
|----------------------------|---------------------------------|----------------------------------------------------------|----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger                | Triggers workflow every 30 minutes                        | —                          | 1. Generate Trendy Idea      | "Step 1: AI Brainstorms an Idea" - starts the ideation process                                                  |
| 1. Generate Trendy Idea     | Langchain Agent                | Generates viral ASMR video idea (short phrase)            | Schedule Trigger            | 2. Enrich Idea into Plan     | Same as above                                                                                                   |
| OpenAI Chat Model1          | Langchain OpenAI Model         | Supports AI language model for ideation                   | 1. Generate Trendy Idea     | 2. Enrich Idea into Plan     | Same as above                                                                                                   |
| 2. Enrich Idea into Plan    | Langchain Agent                | Expands idea to full production plan JSON                 | 1. Generate Trendy Idea     | 3. Log New Idea to Sheet, Prompts AI Agent | Same as above                                                                                      |
| Think                      | Langchain Tool (Think)          | AI self-reflection tool                                   | 2. Enrich Idea into Plan    | 2. Enrich Idea into Plan, Prompts AI Agent | "Step 2: Scene Generation & Video Creation" - improves prompt quality                                        |
| Parser                     | Langchain Output Parser         | Parses AI JSON output                                     | 2. Enrich Idea into Plan    | 2. Enrich Idea into Plan     | Same as above                                                                                                   |
| 3. Log New Idea to Sheet    | Google Sheets                  | Logs idea and metadata into Google Sheet                  | 2. Enrich Idea into Plan    | Prompts AI Agent             | Same as above                                                                                                   |
| Prompts AI Agent           | Langchain Agent                | Generates multi-scene detailed video prompts              | 3. Log New Idea to Sheet    | Unbundle Prompts             | "Step 2: Scene Generation & Video Creation" - generates detailed scene descriptions                            |
| Parser2                    | Langchain Output Parser         | Parses multi-scene AI JSON prompts                        | Prompts AI Agent            | Unbundle Prompts             | Same as above                                                                                                   |
| Unbundle Prompts           | Code Node                     | Extracts individual scene descriptions                    | Prompts AI Agent            | Create Clips                 | Same as above                                                                                                   |
| Create Clips               | HTTP Request                  | Calls Seedance API to create video clips                  | Unbundle Prompts            | Wait for Clips              | Same as above                                                                                                   |
| Wait for Clips             | Wait                         | Waits 120 seconds for clip generation                      | Create Clips                | Get Clips                   | Same as above                                                                                                   |
| Get Clips                  | HTTP Request                  | Retrieves Seedance generated clips                         | Wait for Clips              | Create Sounds               | Same as above                                                                                                   |
| Create Sounds              | HTTP Request                  | Calls Fal AI to create ASMR sound effects                  | Get Clips                  | Wait for Sounds             | Same as above                                                                                                   |
| Wait for Sounds            | Wait                         | Waits 60 seconds for sound generation                      | Create Sounds               | Get Sounds                  | Same as above                                                                                                   |
| Get Sounds                 | HTTP Request                  | Retrieves Fal AI generated sounds                          | Wait for Sounds             | List Elements               | Same as above                                                                                                   |
| List Elements              | Code Node                    | Aggregates video URLs for sequencing                       | Get Sounds                  | Sequence Video              | Same as above                                                                                                   |
| Sequence Video             | HTTP Request                  | Calls Fal AI to stitch clips into final video              | List Elements               | Wait for Final Video        | "Step 3: Final Assembly" - stitches clips into single video                                                   |
| Wait for Final Video       | Wait                         | Waits 60 seconds for final video assembly                  | Sequence Video              | Get Final Video             | Same as above                                                                                                   |
| Get Final Video            | HTTP Request                  | Retrieves final video URL                                  | Wait for Final Video        | Update Final Video to Sheet | Same as above                                                                                                   |
| Update Final Video to Sheet | Google Sheets                | Updates sheet with final video link and marks "Done"      | Get Final Video             | Download Final Video        | "Step 4: Distribution & Logging" - updates production status                                                  |
| Download Final Video       | HTTP Request                  | Downloads final video file                                 | Update Final Video to Sheet | Upload to YouTube           | Same as above                                                                                                   |
| Upload to YouTube          | YouTube Node                 | Uploads video to YouTube with metadata                     | Download Final Video        | Gmail Notification, Telegram Notification, Update Sheet with Youtube Link | Same as above                                                           |
| Update Sheet with Youtube Link | Google Sheets            | Adds YouTube URL to sheet row                              | Upload to YouTube           | —                           | Same as above                                                                                                   |
| Telegram Notification      | Telegram Node                | Sends Telegram message about new video                     | Upload to YouTube           | —                           | Same as above                                                                                                   |
| Gmail Notification         | Gmail Node                   | Sends email notification about new video                   | Upload to YouTube           | —                           | Same as above                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Set up Credentials**  
- Create and configure credentials for:  
  - OpenAI API (GPT-4)  
  - Wavespeed AI (Seedance) API key (generic HTTP header auth)  
  - Fal AI API key (generic HTTP header auth)  
  - Google OAuth2 (Google Sheets, YouTube Data API v3 enabled)  
  - Telegram Bot API token  
  - Gmail OAuth2 (optional for email notifications)  

**Step 2: Create Trigger and AI Ideation Nodes**  
1. Add **Schedule Trigger** node:  
   - Set interval to every 30 minutes (or desired frequency)  

2. Add **Langchain Agent** node named "1. Generate Trendy Idea":  
   - System message: AI to generate a single short viral ASMR video idea (under 10 words)  
   - Output parser enabled  
   - Connect Schedule Trigger -> "1. Generate Trendy Idea"  

3. Add **Langchain OpenAI Chat Model** node:  
   - Model: GPT-4.1  
   - Connect "1. Generate Trendy Idea" to this node as AI language model  

4. Add **Langchain Agent** node named "2. Enrich Idea into Plan":  
   - System message to expand the short idea into a JSON production plan with fields: Idea, Caption, Environment, Sound, Status  
   - Use Think tool enabled to review output  
   - Output parser enabled with JSON schema example  
   - Connect "1. Generate Trendy Idea" and OpenAI model outputs as inputs  

5. Add **Think Tool** node:  
   - Connect with "2. Enrich Idea into Plan"  

6. Add **Parser** node:  
   - Configure to parse the JSON output from "2. Enrich Idea into Plan"  

7. Add **Google Sheets** node named "3. Log New Idea to Sheet":  
   - Operation: Append  
   - Map columns: idea, caption, production ("In Progress"), sound_prompt, environment_prompt from parsed AI output  
   - Set your Google Sheet name and ID  
   - Connect the parser output to this node  

**Step 3: Asset Generation**  
8. Add **Langchain Agent** node named "Prompts AI Agent":  
   - System message instructs generating multi-scene, vivid ASMR scene descriptions in JSON format  
   - Input: output from "3. Log New Idea to Sheet"  
   - Output parser enabled with example JSON  

9. Add **Parser2** node:  
   - Parses multi-scene AI output from "Prompts AI Agent"  

10. Add **Code node** named "Unbundle Prompts":  
    - JavaScript to recursively extract scene descriptions from JSON  
    - Input: parsed multi-scene prompt  

11. Add **HTTP Request** node named "Create Clips":  
    - POST to Wavespeed AI Seedance API endpoint  
    - Body: JSON with aspect_ratio "9:16", duration 10, prompt combining Idea, scene description, Environment  
    - Authentication: Wavespeed API key via HTTP header  
    - Connect from "Unbundle Prompts"  

12. Add **Wait** node "Wait for Clips":  
    - Duration: 120 seconds  

13. Add **HTTP Request** node "Get Clips":  
    - GET request to Wavespeed API to retrieve clip results by request_id  
    - Connect from "Wait for Clips"  

14. Add **HTTP Request** node "Create Sounds":  
    - POST to Fal AI sound generation API  
    - Body: JSON with prompt combining sound description and video URL  
    - Authentication: Fal AI API key via HTTP header  
    - Connect from "Get Clips"  

15. Add **Wait** node "Wait for Sounds":  
    - Duration: 60 seconds  

16. Add **HTTP Request** node "Get Sounds":  
    - GET request to Fal AI to retrieve sound results by request_id  
    - Connect from "Wait for Sounds"  

17. Add **Code node** "List Elements":  
    - Aggregate all video URLs from sound info into a single array for sequencing  

18. Add **HTTP Request** node "Sequence Video":  
    - POST to Fal AI video composition API  
    - Body: JSON with video track keyframes (timestamp, duration, URL)  
    - Authentication: Fal AI API key  
    - Connect from "List Elements"  

19. Add **Wait** node "Wait for Final Video":  
    - Duration: 60 seconds  

20. Add **HTTP Request** node "Get Final Video":  
    - GET request to retrieve final video URL by request_id  
    - Connect from "Wait for Final Video"  

**Step 4: Final Assembly and Distribution**  
21. Add **Google Sheets** node "Update Final Video to Sheet":  
    - Operation: Update row keyed by "idea"  
    - Update columns: production = "Done", final_output = video_url  
    - Connect from "Get Final Video"  

22. Add **HTTP Request** node "Download Final Video":  
    - GET request to final video URL (response as file)  
    - Connect from "Update Final Video to Sheet"  

23. Add **YouTube** node "Upload to YouTube":  
    - Resource: video, Operation: upload  
    - Title: "AI ASMR : {{idea}}"  
    - Description and tags set as per workflow  
    - Privacy: public, notify subscribers true  
    - Connect from "Download Final Video"  

24. Add **Google Sheets** node "Update Sheet with Youtube Link":  
    - Update row keyed by "idea" with YouTube URL column  
    - Connect from "Upload to YouTube"  

25. Add **Telegram** node "Telegram Notification":  
    - Sends notification with video link to specified chat ID  
    - Connect from "Upload to YouTube"  

26. (Optional) Add **Gmail** node "Gmail Notification":  
    - Sends email notification with video link  
    - Connect from "Upload to YouTube"  

**Step 5: Connect Nodes According to Dependencies**  
- Ensure triggers flow from Schedule node through AI ideation, asset generation, final assembly, and distribution in sequence as described.  
- Use batching and wait nodes to handle asynchronous API processing and rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is a fully autonomous content factory creating and publishing ASMR YouTube Shorts without manual intervention. It chains multiple AI services to generate unique media, run on a schedule, and manages the entire content pipeline through a Google Sheet CMS.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Workflow description and setup provided by bilsimaging.com                                                                                                                                        |
| API Rate Limits: Frequent runs may cause API blocking. Adjust schedule intervals or batch intervals in HTTP Request nodes to mitigate.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow sticky note on pro-tips                                                                                                                                                                  |
| Cost Management: AI API calls incur costs; recommended to set budget alerts in OpenAI, Fal AI, and Wavespeed dashboards.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Workflow sticky note on pro-tips                                                                                                                                                                  |
| Customization: Modify the system messages in "Prompts AI Agent" and "2. Enrich Idea into Plan" nodes to change video style, tone, or type.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Workflow sticky note on customization                                                                                                                                                            |
| For help or tips on API handling, contact bilsimaging@gmail.com                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Contact info from workflow documentation                                                                                                                                                        |
| Video links: Final YouTube videos are automatically uploaded and URLs logged in Google Sheet, enabling easy tracking and management of created content.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Part of distribution & logging stage                                                                                                                                                             |
| Telegram and Gmail notifications provide real-time alerts to users immediately after video publication.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Useful for workflow monitoring                                                                                                                                                                  |
| Seedance via Wavespeed API is used for text-to-video generation with 9:16 aspect ratio and 10-second clips, suitable for YouTube Shorts format.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Asset generation stage information                                                                                                                                                              |
| Fal AI is used for both sound effect generation and video sequencing, illustrating multi-modal AI integration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Asset generation and assembly stages                                                                                                                                                           |

---

**Disclaimer:**  
The text provided originates exclusively from an n8n automated workflow. The process adheres strictly to current content policies and contains no illegal, offensive, or protected material. All manipulated data are legal and public.

---