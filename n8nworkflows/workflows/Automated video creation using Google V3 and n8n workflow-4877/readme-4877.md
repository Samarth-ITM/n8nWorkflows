Automated video creation using Google V3 and n8n workflow

https://n8nworkflows.xyz/workflows/automated-video-creation-using-google-v3-and-n8n-workflow-4877


# Automated video creation using Google V3 and n8n workflow

### 1. Workflow Overview

This workflow automates the creation of AI-generated videos using Google Sheets as the input source and the VEO3 video generation model from Fal AI as the output platform. It is designed for content creators, marketers, or production teams who want to transform simple video ideas into fully generated videos automatically.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Filtering:** Triggered manually, fetches video ideas marked as "ready" from a Google Sheet.
- **1.2 AI Prompt Engineering:** Uses a GPT-4.1-powered LangChain agent node to convert raw video ideas into richly detailed, structured prompts optimized for the VEO3 model.
- **1.3 Video Creation Submission:** Sends the generated prompt to Fal AI’s VEO3 API to create a video.
- **1.4 Video Generation Monitoring:** Polls the VEO3 API to check the video generation status, with a retry mechanism until completion.
- **1.5 Result Download and Storage:** Downloads the completed video URL and updates the corresponding row in the Google Sheet with the video link and status.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:**  
  This block triggers the workflow manually and fetches from Google Sheets the first video idea marked as "ready" for processing.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `📊 Fetch Ready Ideas from Sheet` (Google Sheets)

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point to start the workflow manually.  
     - Config: No parameters, simple manual trigger.  
     - Inputs: None  
     - Outputs: Trigger output to fetch data  
     - Edge cases: None expected; manual start  
     
  2. **📊 Fetch Ready Ideas from Sheet**  
     - Type: Google Sheets node (v4.6)  
     - Role: Reads the Google Sheet to find the first row where "Status" equals "ready".  
     - Config:  
       - Document ID and Sheet ID are set to a specific Google Sheet containing video ideas.  
       - Filters rows to only those where the "Status" column equals "ready".  
       - Returns only the first matching row to process one idea at a time.  
     - Inputs: Trigger node output  
     - Outputs: JSON data containing the video idea and its metadata, including row number for updates.  
     - Credentials: Uses Google Sheets OAuth2 credentials (named "Freelance Account").  
     - Edge cases:  
       - No rows with "ready" status → node returns empty, potentially halting workflow downstream.  
       - Google Sheets API quota or auth errors.  

---

#### 2.2 AI Prompt Engineering

- **Overview:**  
  Converts the simple idea text into a comprehensive, structured prompt for the VEO3 video generator using GPT-4.1 via LangChain agent.

- **Nodes Involved:**  
  - `🤖 VEO3 Prompt Generator (GPT-4.1)` (LangChain Agent)  
  - `🧠 OpenAI Chat Model` (OpenAI GPT-4.1-mini)  
  - `📝 Structured Output Parser` (LangChain Structured Output Parser)

- **Node Details:**

  1. **🤖 VEO3 Prompt Generator (GPT-4.1)**  
     - Type: LangChain Agent node (v2)  
     - Role: AI prompt engineering using a system message that instructs how to build a detailed VEO3 prompt from a raw idea.  
     - Config:  
       - Input text: "User Idea: {{ $json.Idea }}" (dynamic from fetched sheet row).  
       - System prompt includes detailed instructions on how to break down the prompt into categories such as scene description, visual style, camera movement, lighting, dialogue, subtitles, etc.  
       - Output parser expects a JSON with a "prompt" string field.  
     - Inputs: Output from Google Sheets node  
     - Outputs: JSON with a fully constructed VEO3 prompt string.  
     - Edge cases:  
       - GPT API failure or timeout  
       - Malformed output not matching schema → parsed by Structured Output Parser  
       - Empty or unclear input idea  
     - Requires: OpenAI API credentials configured in linked node.  
     
  2. **🧠 OpenAI Chat Model**  
     - Type: LangChain OpenAI Chat Model (v1.2)  
     - Role: Underlying language model for the LangChain agent node; uses GPT-4.1-mini model.  
     - Config: Model set to "gpt-4.1-mini", no additional options.  
     - Inputs: From LangChain agent node’s ai_languageModel port  
     - Outputs: To LangChain agent node, feeding generation results.  
     - Credentials: OpenAI API key required.  
     - Edge cases: API rate limits, network issues.  
     
  3. **📝 Structured Output Parser**  
     - Type: LangChain structured output parser (v1.2)  
     - Role: Parses the raw GPT output into a structured JSON object with a "prompt" field.  
     - Config: Manual schema expecting an object with a "prompt" string property.  
     - Inputs: Output from LangChain agent node’s ai_outputParser port  
     - Outputs: Clean JSON for next nodes.  
     - Edge cases: If output does not match schema, parsing fails.

---

#### 2.3 Video Creation Submission

- **Overview:**  
  Takes the generated prompt and submits it to Fal AI's VEO3 video generation API to start video creation.

- **Nodes Involved:**  
  - `🎬 Create Video with VEO3 (Fal AI)` (HTTP Request)

- **Node Details:**

  1. **🎬 Create Video with VEO3 (Fal AI)**  
     - Type: HTTP Request (v4.2)  
     - Role: POST request to Fal AI VEO3 endpoint to enqueue video generation.  
     - Config:  
       - URL: `https://queue.fal.run/fal-ai/veo3`  
       - Method: POST  
       - Body: JSON containing the prompt from the AI output (`{{ $json.output.prompt }}`)  
       - Authentication: HTTP Header Auth using Fal AI credentials (name: "Fal AI Accounts")  
       - Batching enabled with batch size 1 and 2 seconds interval to prevent overload  
     - Inputs: Structured prompt JSON  
     - Outputs: Response includes request ID and status URL for monitoring  
     - Edge cases:  
       - API errors or downtime  
       - Invalid prompt causing rejection  
       - Authentication failure  
       - Rate limiting from Fal AI  

---

#### 2.4 Video Generation Monitoring

- **Overview:**  
  Polls the Fal AI API repeatedly every 30 seconds to check if the video generation is completed, implementing a retry loop until completion.

- **Nodes Involved:**  
  - `🔍 Check Video Generation Status` (HTTP Request)  
  - `✅ Is Video Generation Complete?` (If Node)  
  - `⏰ Wait Before Retry` (Wait node)

- **Node Details:**

  1. **🔍 Check Video Generation Status**  
     - Type: HTTP Request (v4.2)  
     - Role: GET request polling the status of the video generation request.  
     - Config:  
       - URL dynamically constructed: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}/status`  
       - Authentication: Same Fal AI HTTP header auth credentials  
     - Inputs: Output from video creation node containing request_id  
     - Outputs: JSON with status field (e.g., "IN_QUEUE", "COMPLETED")  
     - Edge cases: API failures, network issues, invalid request_id  

  2. **✅ Is Video Generation Complete?**  
     - Type: If node (v2.2)  
     - Role: Checks if the status equals "COMPLETED".  
     - Config: Condition: `$json.status === "COMPLETED"`  
     - Inputs: Status JSON from previous node  
     - Outputs:  
       - True branch: video is ready → triggers download  
       - False branch: not ready → triggers wait and retry  
     - Edge cases: Status other than expected values, missing status field  

  3. **⏰ Wait Before Retry**  
     - Type: Wait node (v1.1)  
     - Role: Delays for 30 seconds before retrying status check.  
     - Config: Wait amount set to 30 seconds  
     - Inputs: False branch from If node  
     - Outputs: Loops back to status check node  
     - Edge cases: None intrinsic; workflow could potentially loop indefinitely if generation never completes  

---

#### 2.5 Result Download and Storage

- **Overview:**  
  Downloads the completed video metadata and updates the original Google Sheet row with the video URL and sets status to "completed".

- **Nodes Involved:**  
  - `📥 Download Generated Video` (HTTP Request)  
  - `📊 Update Sheet with Video URL` (Google Sheets)

- **Node Details:**

  1. **📥 Download Generated Video**  
     - Type: HTTP Request (v4.2)  
     - Role: Retrieves the final video info including download URL.  
     - Config:  
       - URL dynamically constructed: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}`  
       - Authentication: Fal AI credentials  
     - Inputs: True branch from "Is Video Generation Complete?" node  
     - Outputs: JSON containing video URL and metadata  
     - Edge cases: Failure to retrieve completed video, network errors  

  2. **📊 Update Sheet with Video URL**  
     - Type: Google Sheets node (v4.6)  
     - Role: Updates the Google Sheet row from which the idea was fetched: sets "Status" to "completed" and adds the "Video URL".  
     - Config:  
       - Document ID and Sheet same as fetch node  
       - Matching on row_number column to update the exact row  
       - Updates "Status" to "completed" and fills "Video URL" with the downloadable link from previous node  
     - Inputs: Video metadata JSON and original row number from initial fetch  
     - Credentials: Google Sheets OAuth2 ("Freelance Account")  
     - Edge cases: Sheet update conflicts, auth errors, incorrect row matching  

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                       | Input Node(s)                    | Output Node(s)                       | Sticky Note                                                                                                           |
|--------------------------------|---------------------------------|-------------------------------------|---------------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger                  | Workflow manual start                | None                            | 📊 Fetch Ready Ideas from Sheet     | ## 📊 Data Input & Trigger System - Fetches ready video ideas from Google Sheets - Set correct Doc ID and filter      |
| 📊 Fetch Ready Ideas from Sheet | Google Sheets                   | Fetches first "ready" idea           | When clicking ‘Execute workflow’ | 🤖 VEO3 Prompt Generator (GPT-4.1) | ## 📊 Data Input & Trigger System - Fetches ready video ideas from Google Sheets - Set correct Doc ID and filter      |
| 🤖 VEO3 Prompt Generator (GPT-4.1) | LangChain Agent               | Generates detailed VEO3 prompt       | 📊 Fetch Ready Ideas from Sheet  | 🎬 Create Video with VEO3 (Fal AI) | ## 🤖 AI-Powered Prompt Engineering - Transforms ideas into detailed prompts - Includes scene, style, lighting, etc.  |
| 🧠 OpenAI Chat Model             | LangChain LM Chat OpenAI        | Provides GPT-4.1-mini model          | 🤖 VEO3 Prompt Generator (GPT-4.1) ai_languageModel | 🤖 VEO3 Prompt Generator (GPT-4.1) ai_outputParser | ## 🤖 AI-Powered Prompt Engineering - Transforms ideas into detailed prompts - Includes scene, style, lighting, etc.  |
| 📝 Structured Output Parser      | LangChain Output Parser         | Parses GPT output to structured JSON | 🤖 VEO3 Prompt Generator (GPT-4.1) ai_outputParser | 🤖 VEO3 Prompt Generator (GPT-4.1) main | ## 🤖 AI-Powered Prompt Engineering - Transforms ideas into detailed prompts - Includes scene, style, lighting, etc.  |
| 🎬 Create Video with VEO3 (Fal AI) | HTTP Request                   | Submit prompt to Fal AI VEO3 model   | 🤖 VEO3 Prompt Generator (GPT-4.1) | 🔍 Check Video Generation Status   | ## 🎬 VEO3 Video Generation & Monitoring - Handles video creation with retry system - Aspect ratio 9:16, 5 sec video    |
| 🔍 Check Video Generation Status | HTTP Request                   | Polls video generation status        | 🎬 Create Video with VEO3 (Fal AI) | ✅ Is Video Generation Complete?    | ## 🎬 VEO3 Video Generation & Monitoring - Handles video creation with retry system - Aspect ratio 9:16, 5 sec video    |
| ✅ Is Video Generation Complete? | If Node                       | Checks if video is ready             | 🔍 Check Video Generation Status | 📥 Download Generated Video (true branch), ⏰ Wait Before Retry (false branch) | ## 🎬 VEO3 Video Generation & Monitoring - Handles video creation with retry system - Aspect ratio 9:16, 5 sec video    |
| ⏰ Wait Before Retry              | Wait Node                     | Delay 30 seconds before retrying     | ✅ Is Video Generation Complete? | 🔍 Check Video Generation Status   | ## 🎬 VEO3 Video Generation & Monitoring - Handles video creation with retry system - Aspect ratio 9:16, 5 sec video    |
| 📥 Download Generated Video      | HTTP Request                  | Downloads video info and URL         | ✅ Is Video Generation Complete? | 📊 Update Sheet with Video URL      | ## 📥 Results Processing & Storage - Saves video URL & updates status in Google Sheets - Video ready for sharing       |
| 📊 Update Sheet with Video URL   | Google Sheets                 | Updates status and video URL in sheet| 📥 Download Generated Video      | None                             | ## 📥 Results Processing & Storage - Saves video URL & updates status in Google Sheets - Video ready for sharing       |
| Sticky Note                     | Sticky Note                   | Overview and summary note             | None                            | None                              | 🎬 VEO3 AI Video Generation Pipeline - Automated video creation from ideas to final output - Link: https://fal.ai/models/fal-ai/veo3 |
| Sticky Note1                   | Sticky Note                   | Data input explanation                | None                            | None                              | 📊 Data Input & Trigger System - Fetches ready video ideas from Google Sheets                                          |
| Sticky Note2                   | Sticky Note                   | AI prompt engineering explanation    | None                            | None                              | 🤖 AI-Powered Prompt Engineering - Transforms simple ideas into detailed VEO3 prompts                                  |
| Sticky Note3                   | Sticky Note                   | Video generation & monitoring details| None                            | None                              | 🎬 VEO3 Video Generation & Monitoring - Handles video creation with smart retry system                                  |
| Sticky Note4                   | Sticky Note                   | Results processing explanation       | None                            | None                              | 📥 Results Processing & Storage - Saves generated video URLs back to Google Sheets                                      |
| Sticky Note5                   | Sticky Note                   | Creator info and external resources  | None                            | None                              | Creator: Lakshit Ukani - LinkedIn, YouTube, Community links, Documentation, Support                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - No parameters required.

2. **Create Google Sheets Node to Fetch Ideas**  
   - Type: Google Sheets  
   - Name: "📊 Fetch Ready Ideas from Sheet"  
   - Set operation to "Read" or "Lookup" (depending on version)  
   - Document ID: your Google Sheet ID containing ideas  
   - Sheet Name or GID: first sheet (gid=0)  
   - Filter rows where "Status" column equals "ready"  
   - Return only the first matching row (option: returnFirstMatch = true)  
   - Credentials: Connect your Google Sheets OAuth2 credentials.  
   - Connect "When clicking ‘Execute workflow’" output to this node's input.

3. **Create LangChain Agent Node for Prompt Generation**  
   - Type: LangChain Agent  
   - Name: "🤖 VEO3 Prompt Generator (GPT-4.1)"  
   - Input Text: `=User Idea: {{ $json.Idea }}`  
   - Add System Message with detailed instructions for prompt engineering as described in the workflow overview (scene description, visual style, etc.)  
   - Set prompt type to "define" with structured output parser expecting JSON with a "prompt" string.  
   - Connect "📊 Fetch Ready Ideas from Sheet" output to this node's input.

4. **Create LangChain OpenAI Chat Model Node**  
   - Type: LangChain LM Chat OpenAI  
   - Name: "🧠 OpenAI Chat Model"  
   - Model: Select "gpt-4.1-mini" or equivalent GPT-4.1 model  
   - Credentials: Provide OpenAI API key  
   - Connect as ai_languageModel input from "🤖 VEO3 Prompt Generator (GPT-4.1)".

5. **Create LangChain Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Name: "📝 Structured Output Parser"  
   - Input Schema: JSON object with property "prompt" of type string  
   - Connect as ai_outputParser input from "🤖 VEO3 Prompt Generator (GPT-4.1)".

6. **Create HTTP Request Node for Video Creation**  
   - Type: HTTP Request  
   - Name: "🎬 Create Video with VEO3 (Fal AI)"  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/veo3`  
   - Body (Raw JSON): `{ "prompt": "{{ $json.output.prompt }}" }`  
   - Authentication: HTTP Header Auth with Fal AI API key credentials  
   - Enable batching with batch size 1, interval 2000 ms  
   - Connect output of "🤖 VEO3 Prompt Generator (GPT-4.1)" to this node.

7. **Create HTTP Request Node for Status Check**  
   - Type: HTTP Request  
   - Name: "🔍 Check Video Generation Status"  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}/status`  
   - Authentication: Fal AI HTTP Header Auth  
   - Connect output of "🎬 Create Video with VEO3 (Fal AI)" to this node.

8. **Create If Node to Check Completion**  
   - Type: If  
   - Name: "✅ Is Video Generation Complete?"  
   - Condition: Check if `$json.status === "COMPLETED"`  
   - Connect output of "🔍 Check Video Generation Status" to this node.

9. **Create Wait Node for Retry Delay**  
   - Type: Wait  
   - Name: "⏰ Wait Before Retry"  
   - Set wait amount to 30 seconds  
   - Connect False branch of "✅ Is Video Generation Complete?" to this node, then connect "⏰ Wait Before Retry" back to "🔍 Check Video Generation Status" to form a loop.

10. **Create HTTP Request Node to Download Video Info**  
    - Type: HTTP Request  
    - Name: "📥 Download Generated Video"  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}`  
    - Authentication: Fal AI HTTP Header Auth  
    - Connect True branch of "✅ Is Video Generation Complete?" to this node.

11. **Create Google Sheets Node to Update Sheet**  
    - Type: Google Sheets  
    - Name: "📊 Update Sheet with Video URL"  
    - Operation: Update  
    - Document ID and Sheet same as fetch node  
    - Match rows on "row_number" column (passed from initial fetch)  
    - Columns to update:  
      - "Status" set to "completed"  
      - "Video URL" set to `{{ $json.video.url }}` (from download node)  
    - Credentials: Same Google Sheets OAuth2 credentials  
    - Connect output of "📥 Download Generated Video" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| 🎬 VEO3 AI Video Generation Pipeline overview with steps: input, prompt engineering, video creation, monitoring, and storage. Includes link to VEO3 model docs.                                                                                 | https://fal.ai/models/fal-ai/veo3                                    |
| 📊 Data Input & Trigger System: reminders to set Google Sheet ID, filter for "ready" status, and ensure "Idea" column contains video ideas.                                                                                                   | N/A                                                                  |
| 🤖 AI-Powered Prompt Engineering: detailed prompt elements generated by GPT-4.1 (scene, style, lighting, dialog, subtitles).                                                                                                                   | N/A                                                                  |
| 🎬 VEO3 Video Generation & Monitoring: settings including aspect ratio 9:16, 5-second duration, audio enabled, retry every 30 seconds until completion.                                                                                       | N/A                                                                  |
| 📥 Results Processing & Storage: updates Google Sheet with final video URL and changes status to "completed".                                                                                                                                   | N/A                                                                  |
| Creator Lakshit Ukani’s professional links, video tutorials, community support resources, and documentation references for troubleshooting and learning more about the workflow and VEO3 API.                                                    | LinkedIn: https://www.linkedin.com/in/lakshit-ukani/                 |
|                                                                                                                                                                                                                                               | YouTube: https://www.youtube.com/@lakshit-ukani?sub_confirmation=1  |
|                                                                                                                                                                                                                                               | Skool Community: https://www.skool.com/ai-automation-club-7843       |
|                                                                                                                                                                                                                                               | VEO3 API Docs: https://docs.fal.ai/models/kling-video                |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All handled data is legal and public.