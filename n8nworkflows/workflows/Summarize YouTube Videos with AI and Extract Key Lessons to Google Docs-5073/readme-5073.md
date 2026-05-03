Summarize YouTube Videos with AI and Extract Key Lessons to Google Docs

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-with-ai-and-extract-key-lessons-to-google-docs-5073


# Summarize YouTube Videos with AI and Extract Key Lessons to Google Docs

### 1. Workflow Overview

This workflow automates the process of summarizing YouTube video transcripts using AI and saving the key lessons and facts into a Google Doc. It is designed to take a YouTube video URL as input, fetch the full transcript via an external API, process and aggregate the transcript text, generate a structured summary using a large language model (LLM), and finally save the output in a Google Document on the user’s Drive.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the YouTube URL via a chat interface trigger.
- **1.2 API Transcript Retrieval:** Calls the Supadata API to fetch the YouTube transcript.
- **1.3 Transcript Processing:** Splits and aggregates the transcript text for AI processing.
- **1.4 AI Summary Generation:** Uses a language model to create a structured summary with lessons and interesting facts.
- **1.5 Output to Google Docs:** Creates a Google Document with the AI-generated summary.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming chat messages and expects a YouTube URL. It initiates the workflow by capturing user input.

**Nodes Involved:**  
- When chat message received  
- Set your supadata key  
- Sticky Note1 (instruction)

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Webhook entry point, listens for chat messages to start the workflow.  
  - Config: Default options, webhook ID assigned.  
  - Inputs: External chat message containing YouTube URL.  
  - Outputs: Passes chat input to next node.  
  - Edge cases: Invalid or missing URL input, non-YouTube URLs.  
  - Sticky Note1 instructs: "Only send a youtube URL in the chat."

- **Set your supadata key**  
  - Type: Set Node  
  - Role: Stores the Supadata API key as a workflow variable (`supadatakey`).  
  - Config: Key variable initialized but empty (requires user to set actual key).  
  - Inputs: Receives data from chat trigger.  
  - Outputs: Passes data with API key to HTTP request node.  
  - Edge cases: Missing or incorrect API key will cause API call failure.  
  - Sticky Note5 notes: "Supadata API key"

---

#### 1.2 API Transcript Retrieval

**Overview:**  
This block sends the YouTube URL to the Supadata API to retrieve the full transcript of the video.

**Nodes Involved:**  
- supadata API - YouTube endpoint  
- Sticky Note (instructions)

**Node Details:**

- **supadata API - YouTube endpoint**  
  - Type: HTTP Request  
  - Role: Calls external Supadata API endpoint to fetch YouTube transcript JSON.  
  - Config:  
    - URL dynamically constructed with the chat input YouTube URL:  
      `https://api.supadata.ai/v1/youtube/transcript?url={{ $('When chat message received').item.json.chatInput }}`  
    - Headers include `x-api-key` with the stored Supadata key.  
  - Inputs: Receives API key and chat input from the Set node.  
  - Outputs: Returns transcript data (likely multiple rows/segments).  
  - Edge cases: API key invalid, rate limits, network errors, invalid video URL, transcript unavailable.  
  - Sticky Note: "Communicate with Supadata.ai to send the Youtube link."

---

#### 1.3 Transcript Processing

**Overview:**  
Splits the returned transcript into individual text segments, then aggregates them into a single text blob for AI summarization.

**Nodes Involved:**  
- Split Out  
- Aggregate  
- Sticky Note2

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Role: Extracts the `content` field from each transcript segment into separate items.  
  - Config: Splitting on `content` field.  
  - Inputs: Transcript JSON from HTTP Request node.  
  - Outputs: Individual text segments for aggregation.  
  - Edge cases: Transcript data missing `content` field or empty transcript.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all text segments into a single aggregated text field.  
  - Config: Aggregates on `text` field (assumed to be the transcript text).  
  - Inputs: Output from Split Out node.  
  - Outputs: Single combined transcript text string.  
  - Edge cases: Empty segments causing empty aggregation.

---

#### 1.4 AI Summary Generation

**Overview:**  
This block sends the aggregated transcript to a large language model to create a structured summary consisting of a brief overview, lessons learned, and interesting facts in markdown format.

**Nodes Involved:**  
- Proces transcript to summary template  
- Gemini 2.0 flash  
- Sticky Note3

**Node Details:**

- **Proces transcript to summary template**  
  - Type: Langchain Chain LLM  
  - Role: Defines the prompt and sends text to the language model for processing.  
  - Config:  
    - Prompt instructs the LLM to:  
      1. Not change the tenor of the text  
      2. Extract best lessons as a teacher/trainer would  
      3. Output summary with three parts: brief summary, lessons in bullet points, interesting facts in bullet points  
      4. Output in markdown format  
    - Uses aggregated transcript text as input.  
  - Inputs: Aggregated transcript text from Aggregate node.  
  - Outputs: Summary text to next node.  
  - Edge cases: Model timeout, malformed prompt, token limits.  
  - Requires OpenRouter API credentials linked to the Gemini 2.0 flash node.

- **Gemini 2.0 flash**  
  - Type: Langchain LM Chat OpenRouter  
  - Role: Acts as the language model backend provider for the chain node.  
  - Config:  
    - Model set to `google/gemini-2.0-flash-001`.  
    - Uses OpenRouter API credentials.  
  - Inputs: Receives prompt from Chain LLM node.  
  - Outputs: Model’s summary response to Chain LLM node.  
  - Edge cases: API errors, credential expiry, network issues.

---

#### 1.5 Output to Google Docs

**Overview:**  
Creates a new Google Document in the user’s Drive and writes the AI-generated summary into it.

**Nodes Involved:**  
- Create new Google Doc with summary  
- Sticky Note4

**Node Details:**

- **Create new Google Doc with summary**  
  - Type: Google Drive  
  - Role: Creates a Google Document from the text content.  
  - Config:  
    - Document name formatted as `transcript {{ $now }}` to include timestamp.  
    - Content set to AI-generated summary text.  
    - Saves to “My Drive” root folder (folder ID left blank).  
  - Inputs: Summary text from Proces transcript to summary template node.  
  - Outputs: Google Drive document metadata.  
  - Edge cases: OAuth2 credential invalid/expired, Drive API quota exceeded, insufficient permissions.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                        | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                  |
|--------------------------------|----------------------------------|-------------------------------------|-----------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When chat message received      | Langchain Chat Trigger            | Entry point, receive YouTube URL    | -                           | Set your supadata key              | Only send a youtube URL in the chat.                                                        |
| Set your supadata key           | Set                              | Stores API key for Supadata          | When chat message received   | supadata API - YouTube endpoint    | Supadata API key                                                                            |
| supadata API - YouTube endpoint | HTTP Request                     | Fetch transcript from Supadata API  | Set your supadata key        | Split Out                         | Communicate with Supadata.ai to send the Youtube link                                      |
| Split Out                      | Split Out                        | Split transcript rows into segments | supadata API - YouTube endpoint | Aggregate                       | Split out all the rows from the transcript and aggregate to one text                        |
| Aggregate                     | Aggregate                       | Combine all transcript segments      | Split Out                   | Proces transcript to summary template | Split out all the rows from the transcript and aggregate to one text                        |
| Proces transcript to summary template | Langchain Chain LLM             | Generate summary, lessons, facts    | Aggregate                   | Create new Google Doc with summary | Send to your favorite LLM to transform to a summary, lessons and facts. (system instructions inside) |
| Gemini 2.0 flash                | Langchain LM Chat OpenRouter     | Language model backend               | Proces transcript to summary template (ai_languageModel input) | Proces transcript to summary template (ai_languageModel output) |                                                                                              |
| Create new Google Doc with summary | Google Drive                    | Save summary to Google Docs          | Proces transcript to summary template | -                                 | Write the output to a Google Doc on your Drive                                             |
| Sticky Note                    | Sticky Note                     | Instructional notes                  | -                           | -                                 | Communicate with Supadata.ai to send the Youtube link                                      |
| Sticky Note1                   | Sticky Note                     | Instructional notes                  | -                           | -                                 | Only send a youtube URL in the chat.                                                       |
| Sticky Note2                   | Sticky Note                     | Instructional notes                  | -                           | -                                 | Split out all the rows from the transcript and aggregate to one text                        |
| Sticky Note3                   | Sticky Note                     | Instructional notes                  | -                           | -                                 | Send to your favorite LLM to transform to a summary, lessons and facts. (system instructions inside) |
| Sticky Note4                   | Sticky Note                     | Instructional notes                  | -                           | -                                 | Write the output to a Google Doc on your Drive                                             |
| Sticky Note5                   | Sticky Note                     | Instructional notes                  | -                           | -                                 | Supadata API key                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add node: `When chat message received` (Langchain Chat Trigger)  
   - Default webhook settings, no special config required.  
   - This node receives the YouTube URL as chat input.

2. **Add a Set Node for API Key**  
   - Add node: `Set`  
   - Create a string variable named `supadatakey`. Leave the value blank initially; user to fill with actual Supadata API key.  
   - Connect output of "When chat message received" to this node.

3. **Add HTTP Request for Supadata API**  
   - Add node: `HTTP Request`  
   - Set method to GET.  
   - URL: `https://api.supadata.ai/v1/youtube/transcript?url={{ $('When chat message received').item.json.chatInput }}` (use expression to inject chat input).  
   - Headers: Add header `x-api-key` with value from `supadatakey` variable (`={{ $json.supadatakey }}`).  
   - Connect output of "Set your supadata key" to this node.

4. **Add Split Out Node**  
   - Add node: `Split Out`  
   - Configure `Field to split out` as `content`.  
   - Connect output of HTTP Request node to this.

5. **Add Aggregate Node**  
   - Add node: `Aggregate`  
   - Configure to aggregate on field `text` (the transcript text field).  
   - Connect output of Split Out node to this.

6. **Configure AI Chain Node for Summary**  
   - Add node: `Chain LLM` (from Langchain nodes)  
   - Configure prompt text with instructions:  
     ```
     Below is a transcript of a youtube video. 
     Please summarize this, following these rules for your output:
     1. NEVER change the tenor of the text
     2. Get the best lessons from this transcript, as if you are a teacher or trainer. 
     3. Summarize this with the following parts: Brief summary of the transcript, lessons learned in bullet points, interesting facts mentioned in bullet points. 

     ALWAYS put your output in markdown. 

     Transcript:
     {{ $json.text }}
     ```  
   - Set prompt type to `define`.  
   - Connect Aggregate node output to this node.

7. **Add Language Model Node**  
   - Add node: `LM Chat` (Langchain LM Chat OpenRouter)  
   - Select model: `google/gemini-2.0-flash-001`.  
   - Add OpenRouter API credentials (create and link before).  
   - Connect this node as the AI language model for the Chain LLM node (ai_languageModel input/output).

8. **Add Google Drive Node**  
   - Add node: `Google Drive`  
   - Operation: `createFromText`  
   - Name: `transcript {{ $now }}` (use expression to add timestamp)  
   - Content: Set to `{{ $json.text }}` from Chain LLM summary output.  
   - Drive folder: Select “My Drive” or specify desired folder.  
   - Configure with Google Drive OAuth2 credentials.  
   - Connect output of Chain LLM node to this node.

9. **Add Instructional Sticky Notes** (Optional)  
   - Add Sticky Note nodes near logical groups for documentation and clarity.

10. **Activate and Test**  
    - Activate workflow.  
    - Test by sending a YouTube URL in the chat interface.  
    - Verify transcript retrieval, summary generation, and Google Doc creation.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Supadata API requires a valid API key for transcript retrieval.                               | https://api.supadata.ai                                                                         |
| This workflow uses the google/gemini-2.0-flash-001 model via OpenRouter API for LLM processing.| OpenRouter: https://openrouter.ai                                                              |
| Google Drive OAuth2 credentials must have permissions to create documents in user’s Drive.    | Google Cloud Console setup for OAuth2 and Drive API                                             |
| Workflow expects only YouTube video URLs to be sent via chat to avoid errors in transcript fetch.| Sticky Note1: "Only send a youtube URL in the chat."                                           |
| Output summary is formatted in markdown with sections: summary, lessons learned, interesting facts.| Supports better readability and formatting in Google Docs                                       |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected content. All handled data is legal and publicly available.