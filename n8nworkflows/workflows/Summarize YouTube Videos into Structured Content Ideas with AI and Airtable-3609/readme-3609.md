Summarize YouTube Videos into Structured Content Ideas with AI and Airtable

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-into-structured-content-ideas-with-ai-and-airtable-3609


# Summarize YouTube Videos into Structured Content Ideas with AI and Airtable

### 1. Workflow Overview

This workflow automates the process of transforming YouTube videos into structured content ideas stored in Airtable. It is designed for content creators, marketers, and knowledge managers who want to maintain a fresh content pipeline by extracting and summarizing video content efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Airtable Query:** Periodically scans Airtable for new YouTube video links that have not been processed.
- **1.2 Video ID Extraction:** Parses the YouTube URL to extract the video ID required for transcript retrieval.
- **1.3 Transcript Retrieval:** Calls a third-party API via RapidAPI to fetch the video transcript.
- **1.4 Transcript Processing:** Combines and formats the transcript data into a single text string.
- **1.5 AI Summarization:** Uses a Large Language Model (LLM) via LangChain to generate a main idea and key takeaways from the transcript.
- **1.6 Airtable Update:** Writes the summarized insights back to Airtable and marks the video as processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Airtable Query

- **Overview:**  
  This block triggers the workflow every 5 minutes and queries Airtable for new YouTube video entries that have not yet been processed.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Airtable (Search)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow on a recurring schedule (every 5 minutes).  
    - Configuration: Interval set to minutes with a default frequency of 5 minutes.  
    - Inputs: None (trigger node).  
    - Outputs: Triggers the Airtable node.  
    - Edge Cases: If the schedule is misconfigured, the workflow may not run as expected.

  - **Airtable (Search)**  
    - Type: Airtable node  
    - Role: Searches the Airtable base "Content Hub" in the "Ideas" table for records where `Status` is empty and `Type` is "Youtube Video".  
    - Configuration:  
      - Base: Content Hub  
      - Table: Ideas  
      - Filter formula: `AND({Status} = "", {Type} = "Youtube Video")`  
      - Limit: 1 record per run to process videos sequentially.  
    - Credentials: Airtable Personal Access Token.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Passes matched records to the next node.  
    - Edge Cases: If no matching records exist, workflow stops gracefully. Authentication errors or API limits may cause failure.

#### 2.2 Video ID Extraction

- **Overview:**  
  Extracts the YouTube video ID from the URL stored in Airtable to use it for transcript retrieval.

- **Nodes Involved:**  
  - Get Video ID (Code node)

- **Node Details:**

  - **Get Video ID**  
    - Type: Code (JavaScript)  
    - Role: Parses the `Source` field (YouTube URL) to extract the 11-character video ID using regex.  
    - Configuration: Uses regex `/(?:v=|\/)([a-zA-Z0-9_-]{11})/` to find the video ID.  
    - Inputs: Airtable search results.  
    - Outputs: Adds `videoId` field to each item JSON.  
    - Edge Cases: If URL is malformed or missing video ID, `videoId` is set to null, potentially causing downstream errors.

#### 2.3 Transcript Retrieval

- **Overview:**  
  Calls the RapidAPI YouTube Video Summarizer API to fetch the transcript of the video using the extracted video ID.

- **Nodes Involved:**  
  - Get video transcript (HTTP Request)

- **Node Details:**

  - **Get video transcript**  
    - Type: HTTP Request  
    - Role: Sends GET request to `https://youtube-video-summarizer-gpt-ai.p.rapidapi.com/api/v1/get-transcript-v2` with query parameters: `video_id` and `platform=youtube`.  
    - Headers: Includes `X-Rapidapi-Key` (user must configure) and `X-Rapidapi-Host`.  
    - Inputs: Receives `videoId` from previous node.  
    - Outputs: Returns transcript data in JSON format.  
    - Edge Cases: API key missing or invalid, rate limits, network errors, or invalid video ID may cause failures.

#### 2.4 Transcript Processing

- **Overview:**  
  Processes the nested transcript JSON structure to concatenate all transcript text segments into a single string.

- **Nodes Involved:**  
  - Get All Transcripts (Set)  
  - Combine Transcripts (Code)  
  - Get Full Transcript (Set)

- **Node Details:**

  - **Get All Transcripts**  
    - Type: Set  
    - Role: Extracts `data.transcripts` from the API response and sets it explicitly for processing.  
    - Inputs: Output from HTTP Request node.  
    - Outputs: Passes transcript data to Combine Transcripts node.

  - **Combine Transcripts**  
    - Type: Code (JavaScript)  
    - Role: Iterates over all transcript segments, concatenates their `text` fields into one string with spaces.  
    - Inputs: Receives `data.transcripts` object.  
    - Outputs: Returns a single string field `Transcript`.  
    - Edge Cases: Missing or malformed transcript data results in empty transcript string.

  - **Get Full Transcript**  
    - Type: Set  
    - Role: Assigns the combined transcript string to a field named `Transcript` for downstream use.  
    - Inputs: Output from Combine Transcripts.  
    - Outputs: Passes transcript text to AI summarization node.

#### 2.5 AI Summarization

- **Overview:**  
  Uses an AI-powered LangChain node to generate a structured summary including a main idea and key takeaways from the transcript.

- **Nodes Involved:**  
  - Extract detailed summary (LangChain Information Extractor)  
  - Get Main Idea & Key Takeaways (Set)

- **Node Details:**

  - **Extract detailed summary**  
    - Type: LangChain Information Extractor  
    - Role: Sends the transcript text to an LLM (OpenAI, Claude, Gemini) to extract a detailed summary.  
    - Configuration:  
      - Prompt instructs the model to output:  
        - Main Idea  
        - Takeaways (as a list)  
      - JSON schema example provided to enforce output structure.  
    - Inputs: Receives `Transcript` string.  
    - Outputs: JSON with `MainIdea` and `Key Takeaways` fields.  
    - Credentials: Requires LLM credentials configured in n8n.  
    - Edge Cases: Model output may be malformed or incomplete; network or API errors possible.

  - **Get Main Idea & Key Takeaways**  
    - Type: Set  
    - Role: Maps the AI output fields `MainIdea` and `Key Takeaways` to workflow variables `Main Idea` and `Key Takeaways`.  
    - Inputs: Output from LangChain node.  
    - Outputs: Prepares data for Airtable update.

#### 2.6 Airtable Update

- **Overview:**  
  Updates the original Airtable record with the generated main idea, key takeaways, and marks the status checkbox as completed.

- **Nodes Involved:**  
  - Update Airtable (Airtable node)

- **Node Details:**

  - **Update Airtable**  
    - Type: Airtable node  
    - Role: Updates the matched record in the "Ideas" table with:  
      - `Status` set to true (checked)  
      - `Main Idea` field updated with summary text  
      - `Takeaways` field updated with key takeaways list  
    - Configuration:  
      - Uses record ID from the original Airtable search node to identify the record.  
      - Typecast enabled to ensure correct field types.  
    - Credentials: Airtable Personal Access Token.  
    - Inputs: Receives summary data and record ID.  
    - Outputs: Final step; no further nodes.  
    - Edge Cases: Airtable API errors, permission issues, or data type mismatches.

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                          |
|-------------------------|-------------------------------|----------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger               | Starts workflow every 5 minutes        | None                    | Airtable                 |                                                                                                    |
| Airtable               | Airtable                      | Searches Airtable for new YouTube links| Schedule Trigger        | Get Video ID             |                                                                                                    |
| Get Video ID           | Code                          | Extracts YouTube video ID from URL     | Airtable                | Get video transcript     |                                                                                                    |
| Get video transcript   | HTTP Request                  | Fetches transcript from RapidAPI       | Get Video ID            | Get All Transcripts      |                                                                                                    |
| Get All Transcripts    | Set                           | Extracts transcripts object            | Get video transcript    | Combine Transcripts      |                                                                                                    |
| Combine Transcripts    | Code                          | Concatenates transcript text           | Get All Transcripts     | Get Full Transcript      |                                                                                                    |
| Get Full Transcript    | Set                           | Prepares full transcript string        | Combine Transcripts     | Extract detailed summary |                                                                                                    |
| Extract detailed summary| LangChain Information Extractor| Summarizes transcript with AI          | Get Full Transcript     | Get Main Idea & Key Takeaways |                                                                                                    |
| Get Main Idea & Key Takeaways | Set                     | Maps AI output to variables             | Extract detailed summary| Update Airtable          |                                                                                                    |
| Update Airtable        | Airtable                      | Updates Airtable record with summary   | Get Main Idea & Key Takeaways | None                  |                                                                                                    |
| Sticky Note            | Sticky Note                   | Workflow description note               | None                    | None                     | ## 📝 Description Automatically turn YouTube videos into clear, structured content ideas stored in Airtable. This workflow pulls new video links from Airtable, extracts transcripts using a RapidAPI service, summarizes them with your favourite LLM, and logs the main idea and key takeaways—keeping your content pipeline fresh with minimal effort. |
| Sticky Note1           | Sticky Note                   | Workflow functionality note             | None                    | None                     | ## ⚙️ What It Does - **Scans** Airtable for new YouTube video links every 5 minutes.. - **Extracts** the transcript of the video using a third-party API via RapidAPI. - **Summarizes** the content to generate a main idea and takeaways. - **Updates** the original Airtable entry with the insights and marks it as completed. |
| Sticky Note2           | Sticky Note                   | Setup instructions note                 | None                    | None                     | ## 🧰 Setup Instructions 1. Clone this template into your n8n workspace. 2. Open the Get YouTube Sources node and configure your Airtable credentials. 3. In the Get video transcript node: - Enter your X-RapidAPI-Key under headers. - The API endpoint is pre-configured. 4. Connect your LLM credentials to the Extract detailed summary node. 5. (Optional) Adjust the summarization prompt in the LangChain node to better suit your tone. 6. Set your preferred schedule in the Trigger node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to every 5 minutes (or preferred frequency).

2. **Add an Airtable node (Search operation):**  
   - Connect Schedule Trigger → Airtable node.  
   - Configure credentials with your Airtable Personal Access Token.  
   - Set Base to your Airtable base (e.g., "Content Hub").  
   - Set Table to "Ideas".  
   - Operation: Search.  
   - Filter formula: `AND({Status} = "", {Type} = "Youtube Video")`  
   - Limit: 1 (to process one video at a time).

3. **Add a Code node named "Get Video ID":**  
   - Connect Airtable → Get Video ID.  
   - Paste the following JavaScript code to extract video ID from the `Source` field:
     ```javascript
     for (const item of $input.all()) {
         const Source = item.json.Source;
         const videoIdMatch = Source.match(/(?:v=|\/)([a-zA-Z0-9_-]{11})/);
         const videoId = videoIdMatch ? videoIdMatch[1] : null;
         item.json.videoId = videoId;
     }
     return $input.all();
     ```
4. **Add an HTTP Request node named "Get video transcript":**  
   - Connect Get Video ID → Get video transcript.  
   - Method: GET  
   - URL: `https://youtube-video-summarizer-gpt-ai.p.rapidapi.com/api/v1/get-transcript-v2`  
   - Query Parameters:  
     - `video_id` = `={{ $json.videoId }}`  
     - `platform` = `youtube`  
   - Headers:  
     - `X-Rapidapi-Key` = your RapidAPI key  
     - `X-Rapidapi-Host` = `youtube-video-summarizer-gpt-ai.p.rapidapi.com`  
   - Ensure credentials and API key are correctly set.

5. **Add a Set node named "Get All Transcripts":**  
   - Connect Get video transcript → Get All Transcripts.  
   - Set field `data.transcripts` to `={{ $json.data.transcripts }}`.

6. **Add a Code node named "Combine Transcripts":**  
   - Connect Get All Transcripts → Combine Transcripts.  
   - Use this JavaScript code to concatenate transcript text:
     ```javascript
     let Transcript = "";
     if ($json.data && $json.data.transcripts) {
         for (const key in $json.data.transcripts) {
             if ($json.data.transcripts[key].custom) {
                 const customArray = $json.data.transcripts[key].custom;
                 for (const item of customArray) {
                     if (item.text) {
                         Transcript += item.text + " ";
                     }
                 }
             }
         }
     }
     return [{ json: { Transcript: Transcript.trim() } }];
     ```
7. **Add a Set node named "Get Full Transcript":**  
   - Connect Combine Transcripts → Get Full Transcript.  
   - Assign field `Transcript` to `={{ $json.Transcript }}`.

8. **Add a LangChain Information Extractor node named "Extract detailed summary":**  
   - Connect Get Full Transcript → Extract detailed summary.  
   - Configure LLM credentials (OpenAI, Claude, Gemini).  
   - Set prompt text:
     ```
     Your job is to generate detailed summary of "{{ $json.Transcript }}".

     Always output your answer in the following format:

     - Main Idea
     - Takeaways
     ```
   - Provide JSON schema example to enforce output format (optional but recommended).

9. **Add a Set node named "Get Main Idea & Key Takeaways":**  
   - Connect Extract detailed summary → Get Main Idea & Key Takeaways.  
   - Map fields:  
     - `Main Idea` = `={{ $json.output.MainIdea }}`  
     - `Key Takeaways` = `={{ $json.output['Key Takeaways'] }}`

10. **Add an Airtable node named "Update Airtable":**  
    - Connect Get Main Idea & Key Takeaways → Update Airtable.  
    - Configure Airtable credentials.  
    - Set Base and Table to match the original Airtable source.  
    - Operation: Update.  
    - Map record ID from the original Airtable search node (`={{ $('Airtable').item.json.id }}`).  
    - Update fields:  
      - `Status` = true (checked)  
      - `Main Idea` = `={{ $json['Main Idea'] }}`  
      - `Takeaways` = `={{ $json['Key Takeaways'] }}`  
    - Enable typecast to ensure proper field types.

11. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Automatically turn YouTube videos into clear, structured content ideas stored in Airtable. This workflow pulls new video links from Airtable, extracts transcripts using a RapidAPI service, summarizes them with your favourite LLM, and logs the main idea and key takeaways. | Workflow Description Sticky Note                                                                         |
| - Scans Airtable for new YouTube video links every 5 minutes. - Extracts the transcript of the video using a third-party API via RapidAPI. - Summarizes the content to generate a main idea and takeaways. - Updates the original Airtable entry with the insights and marks it as completed. | Workflow Functionality Sticky Note                                                                        |
| Setup Instructions: 1. Clone this template into your n8n workspace. 2. Configure Airtable credentials in the Airtable node. 3. Enter your X-RapidAPI-Key in the HTTP Request node headers. 4. Connect your LLM credentials. 5. Optionally adjust the summarization prompt. 6. Set your preferred schedule. | Workflow Setup Sticky Note                                                                                |
| Airtable base "Content Hub" with table "Ideas" must have columns: Type (Single select, must be "Youtube Video"), Source (URL), Status (Checkbox), MainIdea (Single line text), Key Takeaways (Long text).                       | Airtable Setup Instructions                                                                              |
| Requires valid RapidAPI key for youtube-video-summarizer-gpt-ai API and LLM credentials (OpenAI, Claude, Gemini) configured in n8n.                                                                                           | Prerequisites                                                                                           |

---

This document provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling advanced users and AI agents to reproduce, modify, and troubleshoot it effectively.