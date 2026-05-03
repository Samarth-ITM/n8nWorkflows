Free YouTube Video Analyzer with AI-Powered Summaries & Email Alerts

https://n8nworkflows.xyz/workflows/free-youtube-video-analyzer-with-ai-powered-summaries---email-alerts-2964


# Free YouTube Video Analyzer with AI-Powered Summaries & Email Alerts

### 1. Workflow Overview

This workflow, titled **Free YouTube Video Analyzer with AI-Powered Summaries & Email Alerts**, automates the process of extracting transcripts from YouTube videos, analyzing the content using AI language models, and sending a structured summary via email. It is tailored for content creators, marketers, educators, and researchers who need quick, automated insights from video content.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Video ID Extraction**  
  Starts with a manual trigger and sets or receives a YouTube video URL, then extracts the video ID using a custom JavaScript function.

- **1.2 Transcript Retrieval and Validation**  
  Uses the extracted video ID to request the transcript from a free third-party API, then verifies if a transcript exists.

- **1.3 Transcript Processing**  
  Combines the transcript segments into a single full-text string for analysis.

- **1.4 AI-Powered Analysis**  
  Sends the full transcript text to an AI language model (DeepSeek, OpenAI, or OpenRouter) to generate a structured summary with a title and key points.

- **1.5 Email Notification**  
  Sends the AI-generated summary via email using configured SMTP credentials.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Video ID Extraction

- **Overview:**  
  This block initiates the workflow manually and sets the YouTube video URL. It then extracts the video ID from the URL, which is essential for fetching the transcript.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set YouTube URL (Set Node)  
  - YouTube Video ID (Code Node)  
  - Sticky Note1 (Instructional)  
  - Sticky Note7 (Instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on manual user action.  
    - Configuration: No parameters; triggers workflow execution.  
    - Inputs: None  
    - Outputs: Connects to "Set YouTube URL"  
    - Edge Cases: None; manual trigger is straightforward.

  - **Set YouTube URL**  
    - Type: Set Node  
    - Role: Defines the YouTube video URL to analyze.  
    - Configuration: Sets a string variable `youtubeUrl` with a sample URL `"https://youtu.be/VIDEOID"`. This should be replaced with the actual video URL or dynamically set.  
    - Inputs: From manual trigger  
    - Outputs: Connects to "YouTube Video ID"  
    - Edge Cases: If URL is malformed or empty, subsequent extraction will fail.

  - **YouTube Video ID**  
    - Type: Code Node (JavaScript)  
    - Role: Extracts the 11-character YouTube video ID from the provided URL using regex.  
    - Configuration: Uses a regex pattern matching both `youtu.be` and `youtube.com` URL formats. Returns `null` if no match found.  
    - Key Expression:  
      ```js
      const pattern = /(?:youtube\.com\/(?:[^\/]+\/.+\/|(?:v|e(?:mbed)?)\/|.*[?&]v=)|youtu\.be\/)([^"&?\/\s]{11})/;
      const match = url.match(pattern);
      return match ? match[1] : null;
      ```  
    - Inputs: Receives `youtubeUrl` from "Set YouTube URL"  
    - Outputs: JSON with `videoId`  
    - Edge Cases: Invalid or non-standard URLs yield `null` videoId, causing downstream failures.

  - **Sticky Note1 & Sticky Note7**  
    - Type: Sticky Note  
    - Role: Provide user instructions: "Get the Youtube video ID from the URL" and "Set Youtube video URL manually" respectively.  
    - Inputs/Outputs: None

---

#### 1.2 Transcript Retrieval and Validation

- **Overview:**  
  This block sends the extracted video ID to the YouTube Transcript API to fetch the transcript, then checks if a transcript exists before proceeding.

- **Nodes Involved:**  
  - Generate transcript (HTTP Request)  
  - Get transcript (Set Node)  
  - Exist? (If Node)  
  - Sticky Note (Instructional) nodes: Sticky Note (API key reminder), Sticky Note2, Sticky Note3

- **Node Details:**

  - **Generate transcript**  
    - Type: HTTP Request  
    - Role: Sends a POST request to `https://www.youtube-transcript.io/api/transcripts` with the video ID to retrieve transcript data.  
    - Configuration:  
      - Method: POST  
      - Body: JSON containing `ids` array with the video ID  
      - Headers: Content-Type: application/json  
      - Authentication: HTTP Header Auth with API key (credential named "Youtube Transcript Extractor API")  
    - Inputs: Receives `videoId` from "YouTube Video ID"  
    - Outputs: JSON response with transcript data or empty if none available  
    - Edge Cases:  
      - API key missing or invalid → authentication error  
      - Video without transcript → empty or missing transcript field  
      - Network timeout or API downtime

  - **Get transcript**  
    - Type: Set Node  
    - Role: Extracts the transcript array and language from the API response for easier access downstream.  
    - Configuration:  
      - Sets `transcript` to `tracks[0].transcript` array  
      - Sets `language` to `tracks[0].language` string  
    - Inputs: From "Generate transcript"  
    - Outputs: To "Exist?" node  
    - Edge Cases: If `tracks` array is empty or missing, values may be undefined.

  - **Exist?**  
    - Type: If Node  
    - Role: Checks if the `transcript` array is not empty to decide whether to continue processing.  
    - Configuration: Condition checks that `transcript` is a non-empty array.  
    - Inputs: From "Get transcript"  
    - Outputs:  
      - True branch: to "Get Fulltext"  
      - False branch: workflow stops (no further nodes connected)  
    - Edge Cases: If transcript is empty or null, workflow halts gracefully.

  - **Sticky Notes:**  
    - Sticky Note: "Get a FREE API on youtube-transcript.io and insert the Authentication" (reminder to set API key)  
    - Sticky Note2: "Get the Youtube video transcript"  
    - Sticky Note3: "Not all videos have text translations of the video" (warning about missing transcripts)

---

#### 1.3 Transcript Processing

- **Overview:**  
  This block concatenates all transcript segments into a single string variable for AI analysis.

- **Nodes Involved:**  
  - Get Fulltext (Code Node)  
  - Sticky Note4 (Instructional)

- **Node Details:**

  - **Get Fulltext**  
    - Type: Code Node (JavaScript)  
    - Role: Iterates over the transcript array and concatenates all `text` fields into one string `fulltext`.  
    - Configuration:  
      ```js
      let fulltext = "";
      for (const item of $input.all()[0].json.transcript) {
        fulltext += item.text + " ";
      }
      fulltext = fulltext.trim();
      return { fulltext };
      ```  
    - Inputs: From "Exist?" (true branch)  
    - Outputs: JSON with `fulltext` string  
    - Edge Cases: Empty transcript array would produce empty string; no error but downstream AI analysis may be meaningless.

  - **Sticky Note4:** "Get the full video transcript in a single variable"

---

#### 1.4 AI-Powered Analysis

- **Overview:**  
  This block sends the full transcript text to an AI language model chain that generates a structured summary with a title and key points formatted in markdown.

- **Nodes Involved:**  
  - Analyze LLM Chain (LangChain LLM Chain Node)  
  - DeepSeek Chat Model (AI Model)  
  - OpenAI Chat Model (AI Model)  
  - OpenRouter Chat Model (AI Model)  
  - Structured Output Parser (Output Parser)  
  - Sticky Note5 (Instructional)

- **Node Details:**

  - **Analyze LLM Chain**  
    - Type: LangChain LLM Chain Node  
    - Role: Sends the full transcript text to the AI model(s) for analysis and summary generation.  
    - Configuration:  
      - Input text: `={{ $json.fulltext }}`  
      - Prompt instructs AI to create a JSON with `title` and `text` fields, summarizing key points in markdown with bullet points, bold terms, tables, and structured sections (definition, characteristics, implementation, pros/cons).  
      - Output parser enabled to parse AI response into structured JSON.  
    - Inputs: From "Get Fulltext"  
    - Outputs: To "Send Email"  
    - Edge Cases:  
      - AI model timeout or API errors  
      - Unexpected AI output format causing parser failure  
      - Empty input text leads to poor summaries

  - **DeepSeek Chat Model**  
    - Type: LangChain AI Model Node  
    - Role: Provides DeepSeek AI model for analysis.  
    - Configuration: Model `deepseek-reasoner` with API credentials.  
    - Inputs: Connected as AI model for "Analyze LLM Chain"  
    - Edge Cases: API key invalid, rate limits.

  - **OpenAI Chat Model**  
    - Type: LangChain AI Model Node  
    - Role: Provides OpenAI GPT-4o-mini model for analysis.  
    - Configuration: Uses OpenAI API credentials.  
    - Inputs: Available but not connected in main flow (optional alternative).  
    - Edge Cases: API key invalid, quota exceeded.

  - **OpenRouter Chat Model**  
    - Type: LangChain AI Model Node  
    - Role: Provides OpenRouter model `deepseek-r1:free`.  
    - Configuration: Uses OpenRouter API credentials.  
    - Inputs: Available but not connected in main flow (optional alternative).  
    - Edge Cases: API key invalid, rate limits.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser Node  
    - Role: Parses AI response into JSON with `title` and `text` fields.  
    - Configuration: JSON schema manually defined with two string properties: `title` and `text`.  
    - Inputs: Connected as output parser for "Analyze LLM Chain"  
    - Edge Cases: Parsing errors if AI output deviates from schema.

  - **Sticky Note5:** "Generate detailed video analysis and create a title"

---

#### 1.5 Email Notification

- **Overview:**  
  Sends the AI-generated summary via email to a configured recipient.

- **Nodes Involved:**  
  - Send Email (Email Send Node)

- **Node Details:**

  - **Send Email**  
    - Type: Email Send Node  
    - Role: Sends an email containing the summary title as subject and the summary text as email body.  
    - Configuration:  
      - Subject: `={{ $json.output.title }}` (from AI output)  
      - Text: `={{ $json.output.text }}` (from AI output)  
      - Email format: Plain text  
      - SMTP credentials configured (e.g., Gmail, Outlook)  
    - Inputs: From "Analyze LLM Chain" (parsed output)  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - SMTP authentication failure  
      - Invalid recipient email (not set here, so likely default sender)  
      - Network issues causing send failure

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                                | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                   |
|---------------------------|--------------------------------|-----------------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Starts workflow manually                       | None                        | Set YouTube URL            |                                                                                              |
| Set YouTube URL           | Set Node                       | Sets YouTube video URL                         | When clicking ‘Test workflow’ | YouTube Video ID           | Set Youtube video URL manually                                                               |
| YouTube Video ID          | Code Node                     | Extracts YouTube video ID from URL            | Set YouTube URL             | Generate transcript         | Get the Youtube video ID from the URL                                                       |
| Generate transcript       | HTTP Request                  | Requests transcript from YouTube Transcript API | YouTube Video ID            | Get transcript              | Get a FREE API on youtube-transcript.io and insert the Authentication                        |
| Get transcript            | Set Node                     | Extracts transcript array and language        | Generate transcript          | Exist?                     | Get the Youtube video transcript                                                             |
| Exist?                   | If Node                      | Checks if transcript exists                    | Get transcript              | Get Fulltext (true branch) | Not all videos have text translations of the video                                           |
| Get Fulltext              | Code Node                    | Concatenates transcript segments into full text | Exist?                      | Analyze LLM Chain           | Get the full video transcript in a single variable                                           |
| Analyze LLM Chain         | LangChain LLM Chain Node     | Sends transcript to AI for structured summary | Get Fulltext                | Send Email                 | Generate detailed video analysis and create a title                                         |
| DeepSeek Chat Model       | LangChain AI Model Node      | Provides DeepSeek AI model                     | Connected to Analyze LLM Chain | Analyze LLM Chain (AI model) |                                                                                              |
| OpenAI Chat Model         | LangChain AI Model Node      | Provides OpenAI GPT model                      | Not connected in main flow  |                            |                                                                                              |
| OpenRouter Chat Model     | LangChain AI Model Node      | Provides OpenRouter AI model                   | Not connected in main flow  |                            |                                                                                              |
| Structured Output Parser  | LangChain Output Parser Node | Parses AI response into JSON                    | Analyze LLM Chain           | Analyze LLM Chain (parser) |                                                                                              |
| Send Email                | Email Send Node              | Sends summary email                            | Analyze LLM Chain           | None                       |                                                                                              |
| Sticky Note               | Sticky Note                  | Instructional notes                            | None                        | None                       | Get a FREE API on youtube-transcript.io and insert the Authentication                        |
| Sticky Note1              | Sticky Note                  | Instructional notes                            | None                        | None                       | Get the Youtube video ID from the URL                                                       |
| Sticky Note2              | Sticky Note                  | Instructional notes                            | None                        | None                       | Get the Youtube video transcript                                                             |
| Sticky Note3              | Sticky Note                  | Instructional notes                            | None                        | None                       | Not all videos have text translations of the video                                           |
| Sticky Note4              | Sticky Note                  | Instructional notes                            | None                        | None                       | Get the full video transcript in a single variable                                           |
| Sticky Note5              | Sticky Note                  | Instructional notes                            | None                        | None                       | Generate detailed video analysis and create a title                                         |
| Sticky Note7              | Sticky Note                  | Instructional notes                            | None                        | None                       | Set Youtube video URL manually                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`. No parameters needed.

2. **Create Set Node to Define YouTube URL**  
   - Add a **Set** node named `Set YouTube URL`.  
   - Add a string field `youtubeUrl` with value `"https://youtu.be/VIDEOID"` (replace `VIDEOID` with actual video ID or set dynamically).  
   - Connect output of Manual Trigger to this node.

3. **Create Code Node to Extract YouTube Video ID**  
   - Add a **Code** node named `YouTube Video ID`.  
   - Use JavaScript code to extract video ID from `youtubeUrl` input:  
     ```js
     const extractYoutubeId = (url) => {
       const pattern = /(?:youtube\.com\/(?:[^\/]+\/.+\/|(?:v|e(?:mbed)?)\/|.*[?&]v=)|youtu\.be\/)([^"&?\/\s]{11})/;
       const match = url.match(pattern);
       return match ? match[1] : null;
     };
     const youtubeUrl = items[0].json.youtubeUrl;
     return [{ json: { videoId: extractYoutubeId(youtubeUrl) } }];
     ```  
   - Connect output of `Set YouTube URL` to this node.

4. **Create HTTP Request Node to Generate Transcript**  
   - Add an **HTTP Request** node named `Generate transcript`.  
   - Configure:  
     - Method: POST  
     - URL: `https://www.youtube-transcript.io/api/transcripts`  
     - Body Content Type: JSON  
     - Body: `{ "ids": ["{{ $json.videoId }}"] }`  
     - Headers: `Content-Type: application/json`  
     - Authentication: HTTP Header Auth with API key credential (create credential with your free API key from youtube-transcript.io).  
   - Connect output of `YouTube Video ID` to this node.

5. **Create Set Node to Extract Transcript and Language**  
   - Add a **Set** node named `Get transcript`.  
   - Set fields:  
     - `transcript` = `={{ $json.tracks[0].transcript }}` (array)  
     - `language` = `={{ $json.tracks[0].language }}` (string)  
   - Connect output of `Generate transcript` to this node.

6. **Create If Node to Check Transcript Existence**  
   - Add an **If** node named `Exist?`.  
   - Condition: Check if `transcript` is a non-empty array (use condition type "Array not empty" on `transcript`).  
   - Connect output of `Get transcript` to this node.

7. **Create Code Node to Concatenate Transcript Text**  
   - Add a **Code** node named `Get Fulltext`.  
   - JavaScript code:  
     ```js
     let fulltext = "";
     for (const item of $input.all()[0].json.transcript) {
       fulltext += item.text + " ";
     }
     fulltext = fulltext.trim();
     return { fulltext };
     ```  
   - Connect the **true** output of `Exist?` to this node.

8. **Create LangChain LLM Chain Node for AI Analysis**  
   - Add a **LangChain LLM Chain** node named `Analyze LLM Chain`.  
   - Set input text to `={{ $json.fulltext }}`.  
   - Configure prompt to instruct AI to generate a JSON summary with `title` and `text` fields, using markdown formatting and structured sections as described.  
   - Enable output parser with a JSON schema:  
     ```json
     {
       "type": "object",
       "properties": {
         "title": { "type": "string" },
         "text": { "type": "string" }
       }
     }
     ```  
   - Connect output of `Get Fulltext` to this node.

9. **Create LangChain AI Model Node(s)**  
   - Add one or more AI model nodes (choose one or multiple):  
     - `DeepSeek Chat Model` (model: `deepseek-reasoner`) with DeepSeek API credentials  
     - `OpenAI Chat Model` (model: `gpt-4o-mini`) with OpenAI API credentials  
     - `OpenRouter Chat Model` (model: `deepseek-r1:free`) with OpenRouter API credentials  
   - Connect the chosen AI model node(s) as the AI model input to `Analyze LLM Chain`.

10. **Create LangChain Output Parser Node**  
    - Add a **Structured Output Parser** node with the same JSON schema as above.  
    - Connect it as the output parser for `Analyze LLM Chain`.

11. **Create Email Send Node**  
    - Add an **Email Send** node named `Send Email`.  
    - Configure:  
      - Subject: `={{ $json.output.title }}`  
      - Text: `={{ $json.output.text }}`  
      - Email format: Plain text  
      - SMTP credentials: Configure with your SMTP server (Gmail, Outlook, etc.)  
    - Connect output of `Analyze LLM Chain` to this node.

12. **Add Sticky Notes (Optional)**  
    - Add sticky notes with instructions at relevant points for clarity, e.g., API key setup, URL setting, transcript availability.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Obtain a free API key from **youtube-transcript.io** to enable transcript generation.                          | https://youtube-transcript.io                    |
| Configure AI model credentials for DeepSeek, OpenAI, or OpenRouter depending on your preference.               | DeepSeek, OpenAI, OpenRouter official docs       |
| SMTP credentials must be valid and allow sending emails from your chosen email provider.                        | Gmail, Outlook, or any SMTP service               |
| The workflow supports multiple AI models; you can enable or disable models by connecting or disconnecting nodes.| Flexibility in AI model choice                     |
| Not all YouTube videos have transcripts available; the workflow stops gracefully if no transcript is found.    | See "Exist?" node and Sticky Note3                |
| The AI prompt is designed to produce a structured JSON summary with markdown formatting for clarity and readability.| Prompt details in "Analyze LLM Chain" node        |

---

This documentation provides a complete, detailed reference to understand, reproduce, and modify the YouTube Video Analyzer workflow, including error handling considerations and integration points.