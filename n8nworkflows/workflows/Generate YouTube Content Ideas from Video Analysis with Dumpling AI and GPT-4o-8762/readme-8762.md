Generate YouTube Content Ideas from Video Analysis with Dumpling AI and GPT-4o

https://n8nworkflows.xyz/workflows/generate-youtube-content-ideas-from-video-analysis-with-dumpling-ai-and-gpt-4o-8762


# Generate YouTube Content Ideas from Video Analysis with Dumpling AI and GPT-4o

---

## 1. Workflow Overview

This workflow automates the generation of YouTube video content ideas by analyzing newly added YouTube video data in a Google Sheet. It leverages Dumpling AI’s services for extracting transcripts and comments from YouTube videos and then uses OpenAI’s GPT-4o model to generate actionable video ideas based on this data. The generated ideas are saved back to Google Sheets and emailed as a summary.

The workflow is logically divided into two main blocks:

- **1.1 Data Collection Block**: Watches for new YouTube video entries in a Google Sheet, processes each video individually by retrieving its transcript and comments via Dumpling AI, then extracts and aggregates comment content.

- **1.2 Idea Generation and Output Block**: Uses GPT-4o to analyze the collected data (transcript, search topic, comments) and generate 3-5 new video ideas. These ideas are split into individual entries, saved to a Google Sheet, and a summary email is sent.

---

## 2. Block-by-Block Analysis

### 2.1 Data Collection Block

**Overview:**  
This block triggers on new rows added to a specified Google Sheet containing YouTube video data. It processes each video entry individually by calling Dumpling AI APIs to retrieve the video’s transcript and comments. The comments are then parsed to extract content and aggregated into a single field for analysis.

**Nodes Involved:**  
- Trigger on New YouTube Video Row  
- Loop Over Videos  
- Wait Between Requests  
- Get Transcript from Dumpling AI  
- Get Comments from Dumpling AI  
- Extract Comment Content  
- Merge Comments into Single Field  

**Node Details:**

- **Trigger on New YouTube Video Row**  
  - Type: Google Sheets Trigger  
  - Role: Initiates the workflow when a new row is added to the "YouTube finds" sheet in the configured Google Sheets document.  
  - Configuration: Polls every minute for new rows. Uses OAuth2 credentials for Google Sheets.  
  - Inputs: None (trigger node).  
  - Outputs: Emits new row JSON with video metadata including "Video Link" and "search topic".  
  - Edge Cases: API rate limits, OAuth token expiration, sheet access permissions issues.  

- **Loop Over Videos**  
  - Type: SplitInBatches  
  - Role: Processes each new video row one at a time to avoid API rate limits or concurrency issues.  
  - Configuration: Default batch size (1).  
  - Inputs: Trigger node outputs.  
  - Outputs: Individual video row for downstream processing.  
  - Edge Cases: Large batch sizes may cause throttling; ensure batch size is 1 for serialized processing.  

- **Wait Between Requests**  
  - Type: Wait  
  - Role: Adds a delay between API calls to Dumpling AI to avoid hitting rate limits.  
  - Configuration: Default (no explicit delay set in JSON, but positioned for throttling).  
  - Inputs: From Loop Over Videos.  
  - Outputs: Passes data to transcript retrieval node.  
  - Edge Cases: Network timeouts; ensure wait time configured adequately to respect API rate limits.  

- **Get Transcript from Dumpling AI**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Dumpling AI API to get the transcript of the YouTube video URL.  
  - Configuration:  
    - URL: https://app.dumplingai.com/api/v1/get-youtube-transcript  
    - Body: JSON with "videoUrl" set dynamically from current video row's "Video Link"  
    - Authentication: HTTP Header Auth using stored Dumpling AI API key credentials  
  - Inputs: Receives current video data from Wait node.  
  - Outputs: JSON with transcript text in `.transcript`.  
  - Edge Cases: API errors (invalid URL, rate limits), missing transcript, network issues.  

- **Get Comments from Dumpling AI**  
  - Type: HTTP Request  
  - Role: Retrieves comments for the video URL from Dumpling AI API.  
  - Configuration:  
    - URL: https://app.dumplingai.com/api/v1/youtube/video/comments  
    - Body: JSON with "url" set dynamically from current video row’s “Video Link”  
    - Authentication: Same Dumpling AI credentials as transcript node.  
  - Inputs: Output from transcript node.  
  - Outputs: JSON array `.comments` containing comment objects with `content`.  
  - Edge Cases: No comments, API rate limiting, malformed response.  

- **Extract Comment Content**  
  - Type: Code (Function)  
  - Role: Parses the `.comments` array from Dumpling AI response and outputs one item per comment with only the `content` field.  
  - Configuration: JavaScript code iterates over all comments and extracts `.content`.  
  - Inputs: Comment JSON from previous node.  
  - Outputs: Array of items each containing `{ content: "comment text" }`.  
  - Edge Cases: Empty or missing comments array; code assumes `.comments` is an array.  

- **Merge Comments into Single Field**  
  - Type: Aggregate  
  - Role: Combines all extracted comment contents into a single field named `comment`, concatenating all comment texts.  
  - Configuration: Aggregates all items’ `content` fields into one string under `comment`.  
  - Inputs: Multiple comment content items from Extract Comment Content node.  
  - Outputs: Single JSON item with merged comments string.  
  - Edge Cases: Very large comment sets may cause payload size issues downstream.  

---

### 2.2 Idea Generation and Output Block

**Overview:**  
This block takes the aggregated transcript, search topic, and merged comments as input to generate structured YouTube video content ideas using GPT-4o. The ideas are split into individual entries, appended to a Google Sheet for record-keeping, and a summary email is sent to notify stakeholders.

**Nodes Involved:**  
- Generate Video Ideas with GPT-4o  
- Split Content Ideas  
- Save Video Ideas to Google Sheets  
- Email Content Ideas  

**Node Details:**

- **Generate Video Ideas with GPT-4o**  
  - Type: OpenAI (LangChain)  
  - Role: Sends a prompt to GPT-4o to analyze transcript, search topic, and comments, and generate 3 to 5 actionable YouTube video ideas as structured JSON.  
  - Configuration:  
    - Model: chatgpt-4o-latest  
    - System prompt: Defines the AI as a YouTube content strategist analyzing transcript, search topic, comments to generate ideas.  
    - User message: Injects merged comments, transcript text, and search topic dynamically from previous nodes.  
    - Output: JSON parsed from AI response, forced JSON output enabled.  
  - Inputs: Aggregated comments, transcript, search topic from Trigger node.  
  - Outputs: JSON containing array `contentIdeas` with fields: title, whyGoodIdea, engagementPotential.  
  - Edge Cases: AI response not valid JSON, timeout, API quota exceeded, malformed input data.  

- **Split Content Ideas**  
  - Type: SplitOut  
  - Role: Splits the array of video ideas from GPT into individual items for separate processing.  
  - Configuration: Splits on field `message.content.contentIdeas`.  
  - Inputs: Output from GPT node.  
  - Outputs: Individual content idea items.  
  - Edge Cases: Empty or missing `contentIdeas` array.  

- **Save Video Ideas to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends each generated video idea as a new row into the "Youtube Content Idea" sheet within the configured Google Sheets document.  
  - Configuration:  
    - Operation: Append  
    - Columns mapped: `title`, `whyGoodIdea`, `engagementPotential` from current JSON item.  
    - Uses OAuth2 credentials for Google Sheets.  
  - Inputs: Individual video idea items from Split Content Ideas.  
  - Outputs: Confirmation of appended row.  
  - Edge Cases: Sheet access permissions, API rate limits, malformed data causing append failure.  

- **Email Content Ideas**  
  - Type: Gmail (Send Email)  
  - Role: Sends a summary email with a link to the Google Sheet containing the newly generated video ideas.  
  - Configuration:  
    - To: example@gmail.com (placeholder, should be replaced with actual recipient)  
    - Subject: "New YouTube Content Ideas Based on Video Analysis"  
    - Message: Text email explaining the source of ideas and providing a link to the sheet  
    - Credentials: Gmail OAuth2 configured.  
  - Inputs: Triggered from Loop Over Videos node (parallel branch) to send once per batch.  
  - Outputs: Email sent confirmation.  
  - Edge Cases: Email sending failure, incorrect recipient address, OAuth token expiration.  

---

## 3. Summary Table

| Node Name                          | Node Type                     | Functional Role                                   | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                                                                                                                                         |
|-----------------------------------|-------------------------------|--------------------------------------------------|---------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger on New YouTube Video Row  | Google Sheets Trigger          | Watches for new YouTube video rows in sheet      | —                               | Loop Over Videos                  | Triggers when a new YouTube video is added to a Google Sheet.                                                                                                                                                                     |
| Loop Over Videos                  | SplitInBatches                 | Processes each video row one at a time            | Trigger on New YouTube Video Row | Wait Between Requests, Email Content Ideas | Loops over video entries serially to handle API rate limits.                                                                                                                                                                     |
| Wait Between Requests             | Wait                          | Adds delay between API calls to Dumpling AI       | Loop Over Videos                | Get Transcript from Dumpling AI    | Adds pauses to avoid API rate limits.                                                                                                                                                                                            |
| Get Transcript from Dumpling AI  | HTTP Request                  | Retrieves YouTube video transcript from Dumpling AI | Wait Between Requests           | Get Comments from Dumpling AI      | Calls Dumpling AI API to get video transcript.                                                                                                                                                                                   |
| Get Comments from Dumpling AI    | HTTP Request                  | Retrieves YouTube video comments from Dumpling AI | Get Transcript from Dumpling AI | Extract Comment Content          | Calls Dumpling AI API to get video comments.                                                                                                                                                                                     |
| Extract Comment Content          | Code (Function)               | Extracts comment text from Dumpling AI comments   | Get Comments from Dumpling AI   | Merge Comments into Single Field   | Parses comments array to extract only the text content.                                                                                                                                                                         |
| Merge Comments into Single Field | Aggregate                    | Aggregates all comment texts into one field       | Extract Comment Content         | Generate Video Ideas with GPT-4o   | Merges all comments into a single string for analysis.                                                                                                                                                                          |
| Generate Video Ideas with GPT-4o | OpenAI (LangChain)           | Generates YouTube video ideas from transcript, comments, and topic | Merge Comments into Single Field | Split Content Ideas               | Generates 3–5 video ideas using GPT-4o analyzing transcript, search topic, and comments.                                                                                                                                          |
| Split Content Ideas              | SplitOut                     | Splits generated ideas array into individual items | Generate Video Ideas with GPT-4o | Save Video Ideas to Google Sheets  | Splits the AI response array into separate rows.                                                                                                                                                                                |
| Save Video Ideas to Google Sheets | Google Sheets                | Appends each video idea to Google Sheets          | Split Content Ideas             | Loop Over Videos                  | Saves generated video ideas back to the "Youtube Content Idea" sheet.                                                                                                                                                            |
| Email Content Ideas             | Gmail                        | Sends email notification with the link to ideas  | Loop Over Videos                | —                               | Sends summary email with link to the content ideas sheet.                                                                                                                                                                       |
| Sticky Note                    | Sticky Note                   | Documentation note                                | —                               | —                               | ## 📌 YouTube Ideas from Dumpling AI + GPT-4o\n\nThis workflow triggers when a new YouTube video is added to a Google Sheet.\n\n**Branch 1 – Collect Data**\n1. 📄 Trigger on new video in sheet  \n2. 🔁 Loop over rows (one at a time)  \n3. 📝 Get transcript (Dumpling AI)  \n4. 💬 Get comments (Dumpling AI)  \n5. 🔃 Extract and merge comment text |
| Sticky Note1                  | Sticky Note                   | Documentation note                                | —                               | —                               | ## 🤖 Generate & Save Video Ideas\n\n**Branch 2 – Process & Output**\n6. 🧠 GPT-4o generates 3–5 video ideas  \n7. 🧩 Split each idea  \n8. 📊 Save to “YouTube Content Idea” sheet  \n9. 📧 Email summary link\n\n✅ Make sure:\n- Dumpling + OpenAI credentials are stored securely\n- Sheet columns: `Video Link`, `search topic`, `title`, `whyGoodIdea`, `engagementPotential` |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets document** with two sheets:
   - Sheet 1 named "YouTube finds" with columns at least including: `Video Link`, `search topic`.
   - Sheet 2 named "Youtube Content Idea" with columns: `title`, `whyGoodIdea`, `engagementPotential`.

2. **Create the trigger node:**
   - Add a **Google Sheets Trigger** node named **"Trigger on New YouTube Video Row"**.
   - Configure it to watch the "YouTube finds" sheet in your Google Sheets document.
   - Set event to `rowAdded`.
   - Set polling interval to every 1 minute.
   - Configure OAuth2 credentials for Google Sheets access.

3. **Add a SplitInBatches node:**
   - Add **"Loop Over Videos"** (SplitInBatches).
   - Connect Trigger node's output to this node's input.
   - Leave batch size at 1 (default) to process videos one-by-one.

4. **Add a Wait node:**
   - Add **"Wait Between Requests"** node.
   - Connect Loop Over Videos node’s output to this node.
   - Configure a delay if needed (e.g., 1–2 seconds) to avoid API rate limits.

5. **Add HTTP Request node to get transcript:**
   - Add **"Get Transcript from Dumpling AI"** node.
   - Configure as POST request to `https://app.dumplingai.com/api/v1/get-youtube-transcript`.
   - Set JSON body to `{ "videoUrl": "{{ $json['Video Link'] }}" }`.
   - Use HTTP Header Authentication with Dumpling AI API Key credentials.
   - Connect Wait node output to this node.

6. **Add HTTP Request node to get comments:**
   - Add **"Get Comments from Dumpling AI"** node.
   - Configure as POST request to `https://app.dumplingai.com/api/v1/youtube/video/comments`.
   - Set JSON body to `{ "url": "{{ $json['Video Link'] }}" }`.
   - Use same Dumpling AI credentials.
   - Connect output of Get Transcript node to this node.

7. **Add a Code (Function) node to extract comment content:**
   - Add **"Extract Comment Content"** node.
   - Use JavaScript code to iterate `.comments` array and output items with only `content` field.
   - Connect output of Get Comments node to this node.

8. **Add Aggregate node to merge comments:**
   - Add **"Merge Comments into Single Field"** node.
   - Configure to aggregate all `content` fields into one string field named `comment`.
   - Connect Extract Comment Content node to this node.

9. **Add OpenAI LangChain node to generate video ideas:**
   - Add **"Generate Video Ideas with GPT-4o"** node.
   - Select `chatgpt-4o-latest` as model.
   - Configure system prompt to instruct AI as YouTube content strategist analyzing transcript, search topic, and comments.
   - Configure user message to dynamically insert:  
     - `comment: {{ JSON.stringify($json.comment) }}`  
     - `transcript:{{ $('Get Transcript from Dumpling AI').item.json.transcript }}`  
     - `Search Topic:{{ $('Trigger on New YouTube Video Row').item.json['search topic'] }}`
   - Enable JSON output parsing.
   - Connect Merge Comments node to this node.

10. **Add SplitOut node to split generated ideas:**
    - Add **"Split Content Ideas"** node.
    - Configure to split array at `message.content.contentIdeas`.
    - Connect Generate Video Ideas node output to this node.

11. **Add Google Sheets node to save video ideas:**
    - Add **"Save Video Ideas to Google Sheets"** node.
    - Configure to append rows to "Youtube Content Idea" sheet.
    - Map columns:  
      - `title` → `={{ $json.title }}`  
      - `whyGoodIdea` → `={{ $json.whyGoodIdea }}`  
      - `engagementPotential` → `={{ $json.engagementPotential }}`
    - Connect Split Content Ideas node to this node.
    - Configure OAuth2 credentials for Google Sheets.

12. **Add Gmail node to email content ideas summary:**
    - Add **"Email Content Ideas"** node.
    - Configure recipient email address (replace example@gmail.com with actual).  
    - Subject: "New YouTube Content Ideas Based on Video Analysis"  
    - Message body: Include link to Google Sheets document and explanation of analysis.  
    - Connect Loop Over Videos node to this node (separate branch) to trigger email per batch.  
    - Configure Gmail OAuth2 credentials.

13. **Connect nodes as per workflow logic:**
    - Trigger → Loop Over Videos  
    - Loop Over Videos → Wait Between Requests  
    - Wait Between Requests → Get Transcript from Dumpling AI  
    - Get Transcript → Get Comments  
    - Get Comments → Extract Comment Content  
    - Extract Comment Content → Merge Comments  
    - Merge Comments → Generate Video Ideas  
    - Generate Video Ideas → Split Content Ideas  
    - Split Content Ideas → Save Video Ideas to Google Sheets  
    - Loop Over Videos → Email Content Ideas (parallel branch)

14. **Activate the workflow and test with new rows:**
    - Add new YouTube video links and search topics to the "YouTube finds" sheet.
    - Monitor executions for errors or rate limits.
    - Verify generated ideas saved to "Youtube Content Idea" sheet and email receipt.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses Dumpling AI to extract YouTube video transcripts and comments, which provides rich context for GPT-4o to analyze and generate targeted video content ideas.                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Dumpling AI API documentation (https://app.dumplingai.com/docs)                                                                        |
| The GPT-4o prompt is designed to produce strictly valid JSON output describing content ideas with structured fields, facilitating automated downstream processing without manual parsing.                                                                                                                                                                                                                                                                                                                                                                                                                                        | GPT-4o OpenAI API                                                                                                                      |
| Google Sheets OAuth2 credentials must be properly configured with edit permissions on the target sheets to allow appending rows and triggering. Gmail node requires OAuth2 with send mail permissions for email dispatch.                                                                                                                                                                                                                                                                                                                                                                                                          | Google Cloud Console - OAuth2 setup for Gmail and Sheets                                                                                 |
| The workflow includes wait nodes and batch splitting to prevent API rate limit errors and ensure sequential processing of videos. This is important when scaling to many entries.                                                                                                                                                                                                                                                                                                                                                                                                                                               | Rate limiting best practices                                                                                                           |
| Sticky notes in the workflow explain the branching into data collection and processing/output phases, and recommend secure storage of API credentials and proper sheet column setup.                                                                                                                                                                                                                                                                                                                                                                                                                                           | Workflow internal documentation                                                                                                       |
| The email node sends a static link to the Google Sheets document. Update the URL in the message body to your actual sheet for correct access.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Edit email message parameter                                                                                                          |
| The workflow can be extended by adding error handling nodes, retries for HTTP requests, or integrating additional AI analysis steps like sentiment analysis or keyword extraction.                                                                                                                                                                                                                                                                                                                                                                                                                                              | Recommended workflow enhancements                                                                                                    |
| This automation is useful for content creators, social media managers, and marketing teams aiming to generate data-driven video ideas based on real audience feedback and video content analysis.                                                                                                                                                                                                                                                                                                                                                                                                                              | Use case description                                                                                                                  |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.