Generate Viral Instagram Scripts by Analyzing Trending Reels with Apify and GPT-4

https://n8nworkflows.xyz/workflows/generate-viral-instagram-scripts-by-analyzing-trending-reels-with-apify-and-gpt-4-10303


# Generate Viral Instagram Scripts by Analyzing Trending Reels with Apify and GPT-4

### 1. Workflow Overview

This n8n workflow automates the generation of viral Instagram video scripts by analyzing trending Instagram Reels through a multi-step process involving data scraping, transcription, AI-driven script rewriting, and reporting. It targets social media marketers, content creators, and automation enthusiasts aiming to create engaging short-form video content inspired by trending reels without copying their exact content.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Data Scraping:** Receives user input (hashtag and number of reels to scrape) via a form, then scrapes Instagram reels related to the given hashtag using Apify's API.

- **1.2 Data Filtering and Storage:** Filters reels to keep only trending ones based on engagement and recency, then appends the filtered data into a Google Sheet for tracking.

- **1.3 Video Transcription and AI Script Generation:** Transcribes each video’s audio to text, and uses an AI agent (GPT-4 via LangChain) to analyze the transcript and generate unique, structurally similar video scripts.

- **1.4 Summary and Notification:** Summarizes the generated scripts and sends a report email to the user.

Additional supporting nodes handle error checking, batch processing, and data updates.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Scraping

- **Overview:**  
  This block initiates the workflow by capturing user inputs for the hashtag and number of reels to scrape, then calls Apify’s Instagram scraping API to fetch reels matching those inputs.

- **Nodes Involved:**  
  - On form submission  
  - Scrape Hashtag

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Receives user input via a web form with fields for a hashtag (string) and number of reels to scrape (number).  
    - Configuration: Form titled "Viral Trending Reels Script Generator" with required fields "Hashtag" and "number of reels to scrape".  
    - Inputs: External HTTP form submission.  
    - Outputs: JSON containing the hashtag and number.  
    - Edge Cases: Missing required fields; form submission errors.  
    - Version: 2.3  

  - **Scrape Hashtag**  
    - Type: HTTP Request  
    - Role: Calls Apify API to scrape Instagram reels by hashtag.  
    - Configuration: POST request to Apify’s Instagram scraper act endpoint with JSON body specifying the hashtag (from form), number of reels, and type as "stories".  
    - Headers: Includes Authorization Bearer token (user must insert their Apify API key).  
    - Inputs: JSON with user form data.  
    - Outputs: JSON array of reels data.  
    - Edge Cases: API key invalid or missing, API rate limits, network timeouts.  
    - Version: 4.2  

---

#### 2.2 Data Filtering and Storage

- **Overview:**  
  Filters scraped reels to keep only those with more than 1000 likes and posted within the last 7 days. Then appends this filtered data to a predefined Google Sheet for tracking.

- **Nodes Involved:**  
  - Filter out from list  
  - Add to the sheet

- **Node Details:**

  - **Filter out from list**  
    - Type: Filter  
    - Role: Applies conditions to filter reels: likesCount > 1000 and timestamp within last 7 days.  
    - Configuration: Conditions use expressions evaluating JSON fields likesCount and timestamp against current date minus 7 days.  
    - Inputs: Full list of scraped reels.  
    - Outputs: Filtered list of reels.  
    - Edge Cases: Missing or malformed likesCount or timestamp fields; date parsing errors.  
    - Version: 2.2  

  - **Add to the sheet**  
    - Type: Google Sheets  
    - Role: Appends filtered reels data as rows in a Google Sheet.  
    - Configuration: Defines columns mapping reel properties (url, caption, songName, thumbnail, timestamp, likesCount, commentsCount, owner details, download URL, primary hashtag).  
    - Credentials: Uses OAuth2 Google Sheets credentials.  
    - Inputs: Filtered reels JSON items.  
    - Outputs: Confirmation of sheet append operation.  
    - Edge Cases: Google API authentication failure, sheet access denied, data format mismatch.  
    - Version: 4.7  

---

#### 2.3 Video Transcription and AI Script Generation

- **Overview:**  
  Processes each filtered reel in batches: transcribes the video audio using Apify’s transcription API, updates the Google Sheet with transcripts, then uses an AI language model (GPT-4 via LangChain agent) to analyze the transcript and generate a new video script with similar engagement structure but original content.

- **Nodes Involved:**  
  - Loop Over Items  
  - Transcribe Video  
  - If error  
  - Update sheet  
  - AI Agent  
  - Add AI script  
  - OpenAI Chat Model  
  - Wait (for retry on transcription error)

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes reels one by one to prevent API overload and enable iterative handling.  
    - Configuration: Default batch size, looping over filtered items from Google Sheet append.  
    - Inputs: List of reels.  
    - Outputs: Single reel item per iteration.  
    - Edge Cases: Large dataset causing long processing time.  
    - Version: 3  

  - **Transcribe Video**  
    - Type: HTTP Request  
    - Role: Calls Apify transcription act to transcribe video URL into text.  
    - Configuration: POST request with JSON body containing the reel’s video URL. Includes Apify API key in headers.  
    - Inputs: Single reel JSON with URL field.  
    - Outputs: JSON containing the transcript.  
    - Edge Cases: API failure, no speech detected, invalid video URL.  
    - Version: 4.2  

  - **If error**  
    - Type: If condition node  
    - Role: Detects transcription errors specifically when transcript contains "no speech found (unexpected error)".  
    - Configuration: Checks transcript string for error message.  
    - Inputs: Transcription output JSON.  
    - Outputs: On error, triggers Wait node; otherwise proceeds to Update sheet.  
    - Edge Cases: False positives or missing error string.  
    - Version: 2.2  

  - **Wait**  
    - Type: Wait  
    - Role: Delays processing before retrying transcription on error.  
    - Configuration: Default delay (unspecified).  
    - Inputs: Triggered by If error node on transcription error.  
    - Outputs: Retries Transcribe Video node.  
    - Edge Cases: Excessive retries causing workflow delays.  
    - Version: 1.1  

  - **Update sheet**  
    - Type: Google Sheets  
    - Role: Updates the Google Sheet row corresponding to the reel with its transcript.  
    - Configuration: Matches rows by URL, appends transcript field.  
    - Credentials: OAuth2 Google Sheets credentials.  
    - Inputs: Transcript and source URL.  
    - Outputs: Confirmation of update.  
    - Edge Cases: Row not found, API errors.  
    - Version: 4.7  

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Analyzes transcript and generates an original video script with similar engagement structure and tone.  
    - Configuration: Uses a system message prompting AI to rewrite scripts with specific stylistic guidelines (short lines, timestamps, conversational tone, Gen-Z style, etc.). Input is the transcript from the sheet.  
    - Inputs: Transcript string from Update sheet.  
    - Outputs: Generated AI script text.  
    - Edge Cases: AI model rate limits, prompt misinterpretation, incomplete scripts.  
    - Version: 2.2  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Backend language model (GPT-4.1-mini) used by AI Agent for text generation.  
    - Configuration: Uses OpenAI credentials with GPT-4.1-mini model.  
    - Inputs: Prompt from AI Agent.  
    - Outputs: AI-generated script text.  
    - Edge Cases: API quota exceeded, auth errors.  
    - Version: 1.2  

  - **Add AI script**  
    - Type: Google Sheets  
    - Role: Appends or updates the Google Sheet with the AI-generated script for the corresponding reel URL.  
    - Configuration: Matches rows by URL, updates "AI Generated Inspired Script" column.  
    - Credentials: OAuth2 Google Sheets credentials.  
    - Inputs: AI script output and reel URL.  
    - Outputs: Confirmation of update.  
    - Edge Cases: Sheet access errors, data mismatch.  
    - Version: 4.7  

---

#### 2.4 Summary and Notification

- **Overview:**  
  After processing all reels, summarizes the total count of AI-generated scripts and sends a styled HTML email report to the user’s Gmail inbox.

- **Nodes Involved:**  
  - Summarize  
  - Send a message  
  - Loop Over Items (continuation)

- **Node Details:**

  - **Summarize**  
    - Type: Summarize  
    - Role: Aggregates the "AI Generated Inspired Script" fields from all processed items to count how many scripts were generated.  
    - Configuration: Summarizes based on the AI script field in the batch.  
    - Inputs: Batch of processed reel items with generated scripts.  
    - Outputs: Count summary.  
    - Edge Cases: Empty or incomplete data sets.  
    - Version: 1.1  

  - **Send a message**  
    - Type: Gmail  
    - Role: Sends an HTML email to a predefined email address reporting the number of scripts generated.  
    - Configuration: Email includes styled HTML content with count placeholder, subject line "🚀 All Scripts Generated 👋".  
    - Credentials: Gmail OAuth2 credentials.  
    - Inputs: Summary count from Summarize node.  
    - Outputs: Email sent confirmation.  
    - Edge Cases: Gmail authentication failure, email delivery issues.  
    - Version: 2.1  

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                                 | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                              |
|---------------------|----------------------------------|------------------------------------------------|-----------------------|------------------------|--------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger                     | Receives hashtag and number input from user    | -                     | Scrape Hashtag          | ## Step 1: Scrape Hashtags<br>- Scrape given hashtag from n8n form<br>- Filters our trending ones<br>- Add to the google sheet |
| Scrape Hashtag       | HTTP Request                    | Scrapes Instagram reels by hashtag via Apify   | On form submission     | Filter out from list    | (See above)                                                                                                              |
| Filter out from list | Filter                         | Filters reels by likes and recency              | Scrape Hashtag         | Add to the sheet        | (See above)                                                                                                              |
| Add to the sheet     | Google Sheets                  | Appends filtered reels data to Google Sheet     | Filter out from list   | Loop Over Items         | (See above)                                                                                                              |
| Loop Over Items      | SplitInBatches                 | Processes reels one by one                       | Add to the sheet       | Summarize, Transcribe Video | (See below)                                                                                                              |
| Transcribe Video     | HTTP Request                  | Transcribes video audio to text using Apify API | Wait, Loop Over Items  | If error                | ## Step 2: Transcribe Video<br>- Transcribe all the videos one by one<br>- Add the scripts in the sheet                  |
| If error             | If                            | Checks transcription errors                      | Transcribe Video       | Wait (on error), Update sheet (success) | (See above)                                                                                                              |
| Wait                 | Wait                          | Waits before retrying transcription              | If error               | Transcribe Video        | (See above)                                                                                                              |
| Update sheet         | Google Sheets                  | Updates Google Sheet with transcript             | If error               | AI Agent                | (See above)                                                                                                              |
| AI Agent             | LangChain Agent               | Analyzes transcript and generates new script    | Update sheet           | Add AI script           | ## Step 3: Analyse and Rewrite<br>- AI analyses each script and rewrites in the same emotional tone and narrative structure while creating entirely new storylines<br>- New AI Script added to the sheet |
| OpenAI Chat Model    | LangChain OpenAI Chat Model  | GPT-4 model backend for AI Agent                 | AI Agent               | AI Agent                | (See above)                                                                                                              |
| Add AI script        | Google Sheets                  | Updates Google Sheet with AI-generated script    | AI Agent               | Loop Over Items         | (See above)                                                                                                              |
| Summarize            | Summarize                     | Counts total AI-generated scripts                 | Loop Over Items        | Send a message          | ## Step 4: Final Reporting<br>- Counts all the successful scripts<br>- Sends to the gmail inbox                           |
| Send a message       | Gmail                         | Sends email report of generated scripts          | Summarize              | -                       | (See above)                                                                                                              |
| Sticky Note          | Sticky Note                   | Workflow description and setup instructions      | -                      | -                       | ## AI Reel Writer<br>### How it works<br>1. Scrapes reels from a given hashtag<br>... (full content in node details)       |
| Sticky Note1         | Sticky Note                   | Step 1 summary note                               | -                      | -                       | ## Step 1: Scrape Hashtags<br>- Scrape given hashtag from n8n form<br>- Filters our trending ones<br>- Add to the google sheet |
| Sticky Note2         | Sticky Note                   | Step 4 summary note                               | -                      | -                       | ## Step 4: Final Reporting<br>- Counts all the successful scripts<br>- Sends to the gmail inbox                           |
| Sticky Note4         | Sticky Note                   | Step 2 summary note                               | -                      | -                       | ## Step 2: Transcribe Video<br>- Transcribe all the videos one by one<br>- Add the scripts in the sheet                  |
| Sticky Note5         | Sticky Note                   | Step 3 summary note                               | -                      | -                       | ## Step 3: Analyse and Rewrite<br>- AI analyses each script and rewrites in the same emotional tone and narrative structure while creating entirely new storylines<br>- New AI Script added to the sheet |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger (version 2.3)  
   - Configure form title: "Viral Trending Reels Script Generator"  
   - Add two required fields:  
     - "Hashtag" (string, placeholder: "aiautomation")  
     - "number of reels to scrape" (number, placeholder: "20")  
   - No special credentials needed.

2. **Create "Scrape Hashtag" node**  
   - Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/reGe1ST3OBgYZSsZJ/run-sync-get-dataset-items`  
   - Headers:  
     - Accept: application/json  
     - Authorization: Bearer [your Apify API key] (replace with your actual key)  
   - Body (JSON):  
     ```json
     {
       "hashtags": ["={{ $json.Hashtag }}"],
       "keywordSearch": false,
       "resultsLimit": {{ $json['number of reels to scrape'] }},
       "resultsType": "stories"
     }
     ```  
   - Connect output of "On form submission" to this node.

3. **Create "Filter out from list" node**  
   - Type: Filter (version 2.2)  
   - Conditions (AND):  
     - `likesCount` > 1000 (number comparison)  
     - `timestamp` before current time minus 7 days (dateTime comparison)  
   - Connect "Scrape Hashtag" output to this filter.

4. **Create "Add to the sheet" node**  
   - Type: Google Sheets (version 4.7)  
   - Operation: Append  
   - Document ID: Use your Google Sheet ID (copy from the provided sheet or your own)  
   - Sheet Name: Use the first sheet (e.g., "gid=0")  
   - Map columns as follows:  
     - url, caption, songName, thumbnail, timestamp (converted to DateTime), likesCount, download url (videoUrl), commentsCount, ownerFullName, ownerUsername (prepend "@" symbol), primaryHashtag (from form submission)  
   - Use Google Sheets OAuth2 credentials configured in your n8n instance.  
   - Connect "Filter out from list" output to this node.

5. **Create "Loop Over Items" node**  
   - Type: SplitInBatches (version 3)  
   - Default batch size (process one item at a time)  
   - Connect "Add to the sheet" output to this node.

6. **Create "Transcribe Video" node**  
   - Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/CVQmx5Se22zxPaWc1/run-sync-get-dataset-items`  
   - Headers:  
     - Accept: application/json  
     - Authorization: Bearer [your Apify API key] (same key as step 2)  
   - Body (JSON):  
     ```json
     {
       "start_urls": "{{ $json.url }}"
     }
     ```  
   - Connect "Loop Over Items" output to this node.

7. **Create "If error" node**  
   - Type: If (version 2.2)  
   - Condition: Check if transcript (field `transcript`) contains string "no speech found (unexpected error)"  
   - Connect "Transcribe Video" output to this node.

8. **Create "Wait" node**  
   - Type: Wait (version 1.1)  
   - Default wait time (optional, configure if needed to avoid API throttling)  
   - Connect "If error" true branch to this node.  
   - Connect "Wait" output back to "Transcribe Video" node for retry.

9. **Create "Update sheet" node**  
   - Type: Google Sheets (version 4.7)  
   - Operation: Append or Update  
   - Document ID and Sheet Name same as in step 4  
   - Matching column: `url`  
   - Map columns: `url` from item, `Transcript` from transcription result  
   - Connect "If error" false branch to this node.

10. **Create "AI Agent" node**  
    - Type: LangChain Agent (version 2.2)  
    - Parameter "text": `={{ $json.Transcript }}`  
    - System message: (copy the detailed prompt instructing AI to analyze transcript and generate original scripts with pacing, tone, timestamps, etc.)  
    - Connect "Update sheet" output to this node.

11. **Create "OpenAI Chat Model" node**  
    - Type: LangChain OpenAI Chat Model (version 1.2)  
    - Model: `gpt-4.1-mini`  
    - Credentials: OpenAI API key configured in n8n  
    - Connect as AI language model backend to "AI Agent".

12. **Create "Add AI script" node**  
    - Type: Google Sheets (version 4.7)  
    - Operation: Append or Update  
    - Document ID and Sheet Name same as above  
    - Matching column: `url`  
    - Map columns: `url` and `AI Generated Inspired Script` (output from AI Agent)  
    - Connect "AI Agent" output to this node.

13. **Connect "Add AI script" output back to "Loop Over Items" node**  
    - This enables processing all items sequentially.

14. **Create "Summarize" node**  
    - Type: Summarize (version 1.1)  
    - Summarize field: `AI Generated Inspired Script` count in batch  
    - Connect "Loop Over Items" output (after all items processed) to this node.

15. **Create "Send a message" node**  
    - Type: Gmail (version 2.1)  
    - Send to: Your email address (e.g., nitindixit18@gmail.com)  
    - Subject: "🚀 All Scripts Generated 👋"  
    - Message: Styled HTML including count placeholder from Summarize node  
    - Credentials: Gmail OAuth2 configured in n8n  
    - Connect "Summarize" output to this node.

16. **Add Sticky Notes**  
    - Add explanatory sticky notes near related nodes summarizing workflow steps, requirements, and setup instructions as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                          | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow requires Apify API key to access Instagram scraping and transcription services. You can create a free account at Apify: https://apify.com?fpr=eg224f                                                                                                          | Apify API Key Setup                                                                                     |
| Google Sheet template used for storing reels and scripts: [Make a copy](https://docs.google.com/spreadsheets/d/1XupKy-R6zPvCfeh5q9co5O-wPMI64D7xvavFkz0yQqk/edit?usp=sharing)                                                                                           | Google Sheet Template                                                                                   |
| Gmail OAuth2 credentials needed for sending email reports.                                                                                                                                                                                                             | Gmail API OAuth2 Setup                                                                                  |
| AI prompt crafted to guide GPT-4 in generating scripts that mimic the engagement style of the original video but with new content, conversational tone, short lines, pacing cues, and clear CTA.                                                                          | Prompt embedded in "AI Agent" node                                                                      |
| Workflow includes error handling with retries on transcription failures due to no speech detected, ensuring robustness.                                                                                                                                               | Error Handling in "If error" and "Wait" nodes                                                          |
| This design supports batch processing to handle multiple reels sequentially, avoiding API throttling and ensuring data consistency in Google Sheets.                                                                                                                  | Use of "Loop Over Items" node                                                                            |

---

_Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public._