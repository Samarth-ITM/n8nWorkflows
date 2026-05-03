YouTube Channel Monitor with Gemini AI Transcription & Summarization to Google Sheets

https://n8nworkflows.xyz/workflows/youtube-channel-monitor-with-gemini-ai-transcription---summarization-to-google-sheets-9268


# YouTube Channel Monitor with Gemini AI Transcription & Summarization to Google Sheets

### 1. Workflow Overview

This workflow automates the monitoring of YouTube channels by fetching recent videos, transcribing their audio using Google Gemini AI, generating summaries, and logging structured data into Google Sheets. It is designed for content creators, marketers, or analysts who want automated, AI-powered insights from YouTube video content.

The workflow is logically divided into the following functional blocks:

- **1.1 Scheduled Trigger & Input Retrieval**: Periodically triggers the workflow, reads YouTube channel URLs from Google Sheets, and prepares these for processing.
- **1.2 Channel & Video Data Acquisition**: For each channel, fetches channel details, reads recent video RSS feeds, and filters videos posted within a defined timeframe.
- **1.3 Video Processing Loop**: Iterates over videos, checks for existing records, and proceeds to transcribe and summarize new videos.
- **1.4 AI Transcription & Summarization**: Uses Google Gemini AI to transcribe video audio and produce a structured summary.
- **1.5 Data Update & Output**: Updates or appends video data and summaries into Google Sheets, managing pacing with wait nodes to respect API limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Input Retrieval

**Overview:**  
This block initiates the workflow on a schedule, retrieves YouTube channel URLs from a Google Sheets document, and prepares the list for processing.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet  
- Split into individual links  
- Check if YouTube link  
- Loop Over channels  

**Node Details:**

- **Schedule Trigger**  
  - **Type:** Trigger node that runs the workflow on a defined schedule.  
  - **Configuration:** Uses default scheduling (likely periodic).  
  - **Inputs:** None (trigger only).  
  - **Outputs:** Starts the workflow chain to get channel URLs.  
  - **Edge cases:** Misconfiguration may cause missing triggers; system downtime can affect schedule.  

- **Get row(s) in sheet**  
  - **Type:** Google Sheets node to fetch data rows.  
  - **Configuration:** Retrieves rows containing YouTube channel URLs from a specified sheet.  
  - **Inputs:** Trigger output.  
  - **Outputs:** List of channel URL strings.  
  - **Edge cases:** Authentication errors, empty or corrupted sheet data.  

- **Split into individual links**  
  - **Type:** Code node.  
  - **Configuration:** Parses the retrieved sheet data to split multiple channel URLs into individual items for processing.  
  - **Inputs:** Rows from Google Sheets.  
  - **Outputs:** Array of single URL items.  
  - **Edge cases:** Malformed input data causing parsing errors; empty or null values.  

- **Check if YouTube link**  
  - **Type:** If node.  
  - **Configuration:** Validates if each item is a valid YouTube channel URL.  
  - **Inputs:** Individual URLs.  
  - **Outputs:** Valid URLs routed forward; invalid filtered out or handled.  
  - **Edge cases:** Unexpected URL formats, non-YouTube URLs.  

- **Loop Over channels**  
  - **Type:** SplitInBatches node.  
  - **Configuration:** Processes channels one batch at a time (batch size default or configured).  
  - **Inputs:** Valid channel URLs.  
  - **Outputs:** Each channel URL sequentially processed.  
  - **Edge cases:** Large number of channels may cause long processing times or API rate limits.

---

#### 1.2 Channel & Video Data Acquisition

**Overview:**  
For each YouTube channel, this block retrieves channel metadata, fetches recent video RSS feeds, filters videos posted within a configurable recent period, and prepares video metadata for processing.

**Nodes Involved:**  
- Set channel  
- Get Channel details  
- RSS Read (youtube channel videos)  
- Videos Posted in Last X days (change on line 6)  
- Video details  
- If title?  
- All videos  

**Node Details:**

- **Set channel**  
  - **Type:** Set node.  
  - **Configuration:** Prepares channel URL or ID for API requests.  
  - **Inputs:** Channel URL from previous block.  
  - **Outputs:** Structured data with channel identifier.  
  - **Edge cases:** Incorrect parsing or input format.  

- **Get Channel details**  
  - **Type:** HTTP Request node.  
  - **Configuration:** Calls YouTube API or related endpoint to fetch channel metadata (e.g., title, description).  
  - **Inputs:** Channel identifier.  
  - **Outputs:** Channel metadata JSON.  
  - **Edge cases:** API authentication issues, rate limits, network errors.  

- **RSS Read (youtube channel videos)**  
  - **Type:** RSS Feed Read node.  
  - **Configuration:** Reads the RSS feed for the channel’s uploaded videos.  
  - **Inputs:** Channel metadata or feed URL.  
  - **Outputs:** List of videos in RSS format.  
  - **Edge cases:** Invalid or outdated RSS feed, empty feed.  

- **Videos Posted in Last X days (change on line 6)**  
  - **Type:** Code node.  
  - **Configuration:** Filters videos to only those posted in the last X days (configurable inside the code, line 6).  
  - **Inputs:** Video list from RSS feed.  
  - **Outputs:** Filtered video list.  
  - **Edge cases:** Date parsing errors, empty filtered list.  

- **Video details**  
  - **Type:** Set node.  
  - **Configuration:** Extracts and structures relevant video metadata fields (title, ID, date, link).  
  - **Inputs:** Filtered videos.  
  - **Outputs:** Structured video data.  
  - **Edge cases:** Missing metadata fields.  

- **If title?**  
  - **Type:** If node.  
  - **Configuration:** Checks if the video title exists to ensure valid video data.  
  - **Inputs:** Video details.  
  - **Outputs:** Videos with titles continue; others discarded.  
  - **Edge cases:** Videos without titles or malformed metadata.  

- **All videos**  
  - **Type:** Set node.  
  - **Configuration:** Aggregates videos passing filters.  
  - **Inputs:** Valid videos.  
  - **Outputs:** Video list for next processing block.  
  - **Edge cases:** Empty video list.  

---

#### 1.3 Video Processing Loop

**Overview:**  
Processes each video individually: checks if the video is already recorded, fetches or skips transcription, and manages pacing with wait nodes to handle API limits.

**Nodes Involved:**  
- Loop Over Videos  
- Video Data and criteria  
- Get row(s) in sheet1  
- If not exist in sheet?  
- Youtube Transcriber Gemini  
- If transcript exists  
- Edit Fields1  
- Wait 85 sec on fail  
- Wait 70 sec for next video  
- End  

**Node Details:**

- **Loop Over Videos**  
  - **Type:** SplitInBatches node.  
  - **Configuration:** Iterates over each video from the previous block one by one.  
  - **Inputs:** Video list.  
  - **Outputs:** Single video data per iteration.  
  - **Edge cases:** Large video count may cause long processing times.  

- **Video Data and criteria**  
  - **Type:** Set node.  
  - **Configuration:** Prepares video metadata and search criteria for sheet lookup.  
  - **Inputs:** Single video.  
  - **Outputs:** Structured data for lookup.  
  - **Edge cases:** Incorrect or missing metadata fields.  

- **Get row(s) in sheet1**  
  - **Type:** Google Sheets node.  
  - **Configuration:** Queries Google Sheets to check if the video is already logged.  
  - **Inputs:** Video criteria data.  
  - **Outputs:** Matching sheet rows or empty result.  
  - **Edge cases:** Sheet access issues, empty or corrupted data.  

- **If not exist in sheet?**  
  - **Type:** If node.  
  - **Configuration:** Checks if the video record does not exist in sheet to proceed with transcription.  
  - **Inputs:** Sheet query result.  
  - **Outputs:** Routes to transcription if new; else wait before next video.  
  - **Edge cases:** False negatives due to sheet sync lag.  

- **Youtube Transcriber Gemini**  
  - **Type:** HTTP Request node.  
  - **Configuration:** Sends video URL or ID to Google Gemini AI transcription service.  
  - **Inputs:** Video data.  
  - **Outputs:** Transcription text or error.  
  - **Edge cases:** API timeouts, transcription failures, rate limits.  
  - **OnError:** configured to continue workflow despite errors.  

- **If transcript exists**  
  - **Type:** If node.  
  - **Configuration:** Checks if transcription output is non-empty.  
  - **Inputs:** Transcription results.  
  - **Outputs:** Routes to data editing or wait on failure.  
  - **Edge cases:** Empty or partial transcription.  

- **Edit Fields1**  
  - **Type:** Set node.  
  - **Configuration:** Prepares transcription data and metadata for AI summarization.  
  - **Inputs:** Transcription and video data.  
  - **Outputs:** Structured input for AI summarizer.  
  - **Edge cases:** Missing transcription or malformed data.  

- **Wait 85 sec on fail**  
  - **Type:** Wait node.  
  - **Configuration:** Pauses workflow 85 seconds to handle API rate limits or failures before retrying.  
  - **Inputs:** Failure route from transcription or AI summarization.  
  - **Outputs:** Loop back to video processing.  
  - **Edge cases:** Excessive delays if many failures.  

- **Wait 70 sec for next video**  
  - **Type:** Wait node.  
  - **Configuration:** Pauses 70 seconds between processing videos to manage pacing and API limits.  
  - **Inputs:** Successful video processing path.  
  - **Outputs:** Continues workflow for next video.  

- **End**  
  - **Type:** Set node.  
  - **Configuration:** Marks processing completion for a batch or loop.  
  - **Inputs:** Loop completion.  
  - **Outputs:** Workflow end or next iteration.  

---

#### 1.4 AI Transcription & Summarization

**Overview:**  
Utilizes Google Gemini AI to transcribe video audio and generate structured video summaries for enhanced insights.

**Nodes Involved:**  
- Edit Fields1  
- Google Gemini Chat Model13  
- Structured Output Parser1  
- youtube video summariser  
- Append or update row in sheet  

**Node Details:**

- **Google Gemini Chat Model13**  
  - **Type:** LangChain Google Gemini Chat Model node.  
  - **Configuration:** Configured to run chat-based AI summarization on transcription data.  
  - **Inputs:** Structured transcription data from Edit Fields1.  
  - **Outputs:** AI-generated summary in chat format.  
  - **Edge cases:** API quota exceeded, malformed input, timeout errors.  

- **Structured Output Parser1**  
  - **Type:** LangChain Structured Output Parser.  
  - **Configuration:** Parses AI chat output into structured data fields suitable for sheet storage.  
  - **Inputs:** Text output from Google Gemini Chat Model13.  
  - **Outputs:** Parsed JSON or structured data.  
  - **Edge cases:** Parsing failures if AI output format changes.  

- **youtube video summariser**  
  - **Type:** LangChain Chain LLM node.  
  - **Configuration:** Chains AI model and parser to generate and structure video summaries.  
  - **Inputs:** Parsed AI output.  
  - **Outputs:** Final structured summary data.  
  - **Edge cases:** Failures if AI inputs are invalid or parser crashes.  

- **Append or update row in sheet**  
  - **Type:** Google Sheets node.  
  - **Configuration:** Appends new video summaries or updates existing rows with transcription and summaries.  
  - **Inputs:** Structured summary data.  
  - **Outputs:** Confirmation of sheet update.  
  - **Edge cases:** Sheet permission errors, API limits, data format mismatches.  

---

### 3. Summary Table

| Node Name                        | Node Type                                  | Functional Role                                  | Input Node(s)                      | Output Node(s)                     | Sticky Note                                |
|---------------------------------|--------------------------------------------|-------------------------------------------------|----------------------------------|----------------------------------|--------------------------------------------|
| Schedule Trigger                | Schedule Trigger                           | Starts workflow on schedule                      | None                             | Get row(s) in sheet               |                                            |
| Get row(s) in sheet             | Google Sheets                             | Retrieves YouTube channel URLs                   | Schedule Trigger                 | Split into individual links       |                                            |
| Split into individual links     | Code                                      | Splits sheet rows into individual URLs          | Get row(s) in sheet              | Check if YouTube link             |                                            |
| Check if YouTube link           | If                                        | Validates if input is a YouTube URL              | Split into individual links      | Loop Over channels                |                                            |
| Loop Over channels              | SplitInBatches                            | Iterates over each YouTube channel               | Check if YouTube link            | If4, set channel                 |                                            |
| If4                            | If                                        | Conditional branching for channels               | Loop Over channels               | Loop Over Videos, set channel    |                                            |
| set channel                    | Set                                       | Prepares channel ID/URL for API calls            | Loop Over channels               | Get Channel details              |                                            |
| Get Channel details            | HTTP Request                              | Fetches YouTube channel metadata                  | set channel                    | RSS Read (youtube channel videos)|                                            |
| RSS Read (youtube channel videos)| RSS Feed Read                            | Reads RSS feed of channel videos                  | Get Channel details             | Videos Posted in Last X days      |                                            |
| Videos Posted in Last X days (change on line 6) | Code                          | Filters videos posted in last X days              | RSS Read (youtube channel videos)| video details                   |                                            |
| video details                  | Set                                       | Extracts and structures video metadata           | Videos Posted in Last X days     | If title?                       |                                            |
| If title?                     | If                                        | Checks if video has a title                        | video details                   | all videos                      |                                            |
| all videos                    | Set                                       | Aggregates valid videos                            | If title?                      | Wait 2 sec                     |                                            |
| Wait 2 sec                   | Wait                                      | Waits 2 seconds between channel batches           | all videos                     | Loop Over channels              |                                            |
| Loop Over Videos             | SplitInBatches                            | Iterates over videos for processing               | If4                            | End, Video Data and criteria    |                                            |
| Video Data and criteria      | Set                                       | Prepares video metadata for sheet lookup          | Loop Over Videos               | Get row(s) in sheet1            |                                            |
| Get row(s) in sheet1         | Google Sheets                             | Checks if video already logged in sheet           | Video Data and criteria        | if not exist in sheet?          |                                            |
| if not exist in sheet?       | If                                        | Checks if video record is missing                  | Get row(s) in sheet1           | Youtube Transcriber Gemini, Wait 70 sec for next video |                                            |
| Youtube Transcriber Gemini   | HTTP Request                              | Sends video for Google Gemini transcription        | if not exist in sheet?         | If transcript exists            |                                            |
| If transcript exists          | If                                        | Checks if transcription exists                      | Youtube Transcriber Gemini     | Edit Fields1, Wait 85 sec on fail |                                            |
| Edit Fields1                 | Set                                       | Prepares transcription data for AI summarization | If transcript exists           | Google Gemini Chat Model13      |                                            |
| Google Gemini Chat Model13   | LangChain Google Gemini Chat Model        | AI summarization of transcription                   | Edit Fields1                  | Structured Output Parser1       |                                            |
| Structured Output Parser1    | LangChain Structured Output Parser        | Parses AI output into structured data               | Google Gemini Chat Model13      | youtube video summariser        |                                            |
| youtube video summariser     | LangChain Chain LLM                        | Chains AI model and parser to produce summary       | Structured Output Parser1       | Append or update row in sheet   |                                            |
| Append or update row in sheet| Google Sheets                             | Updates or appends video data and summaries         | youtube video summariser        | Wait 70 sec for next video      |                                            |
| Wait 85 sec on fail          | Wait                                      | Waits 85 seconds after failure                      | If transcript exists (fail)     | Video Data and criteria         |                                            |
| Wait 70 sec for next video   | Wait                                      | Waits 70 seconds between processing videos          | Append or update row in sheet, if not exist in sheet? (skip) | Loop Over Videos           |                                            |
| End                         | Set                                       | Marks end of video batch processing                  | Loop Over Videos               | None                          |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run at desired intervals (e.g., every hour or daily).

2. **Add a Google Sheets node to Get row(s) in sheet**  
   - Connect from Schedule Trigger.  
   - Configure with credentials to your Google Sheets.  
   - Set spreadsheet and sheet name containing YouTube channel URLs.  
   - Retrieve all rows or a specific range.

3. **Add a Code node Split into individual links**  
   - Connect from Get row(s) in sheet.  
   - Code should parse sheet rows to extract and split multiple YouTube channel URLs into individual items.

4. **Add an If node Check if YouTube link**  
   - Connect from Split into individual links.  
   - Configure condition to validate if each item is a valid YouTube URL (pattern matching on URL).

5. **Add SplitInBatches node Loop Over channels**  
   - Connect from Check if YouTube link (true branch).  
   - Configure batch size (e.g., 1 or more depending on API limits).

6. **Add an If node If4**  
   - Connect from Loop Over channels.  
   - Use for conditional logic if needed (e.g., to branch processing or handle empty batches).

7. **Add a Set node set channel**  
   - Connect from Loop Over channels (or from If4 if used).  
   - Extract or prepare channel ID or URL for API requests.

8. **Add HTTP Request node Get Channel details**  
   - Connect from set channel.  
   - Configure to call YouTube API or other endpoint to retrieve channel info.  
   - Set authentication (API key or OAuth2).

9. **Add RSS Feed Read node RSS Read (youtube channel videos)**  
   - Connect from Get Channel details.  
   - Configure to read the RSS feed URL for channel videos.

10. **Add Code node Videos Posted in Last X days**  
    - Connect from RSS Read.  
    - Write code to filter videos by published date within last X days (configurable).

11. **Add Set node video details**  
    - Connect from code node.  
    - Extract fields like video title, ID, URL, publish date.

12. **Add If node If title?**  
    - Connect from video details.  
    - Check if video title exists.

13. **Add Set node all videos**  
    - Connect from If title? (true branch).  
    - Aggregate valid videos.

14. **Add Wait node Wait 2 sec**  
    - Connect from all videos.  
    - Wait 2 seconds before looping back.

15. **Connect Wait 2 sec back to Loop Over channels** to continue batch processing.

16. **Add SplitInBatches node Loop Over Videos**  
    - Connect from If4 (true branch).  
    - Configure to iterate over videos one by one.

17. **Add Set node Video Data and criteria**  
    - Connect from Loop Over Videos.  
    - Prepare video metadata and lookup criteria for Google Sheets.

18. **Add Google Sheets node Get row(s) in sheet1**  
    - Connect from Video Data and criteria.  
    - Query sheet to check if video is already logged.

19. **Add If node if not exist in sheet?**  
    - Connect from Google Sheets node.  
    - Check if query result is empty (video not logged).

20. **Add HTTP Request node Youtube Transcriber Gemini**  
    - Connect from if not exist in sheet? (true branch).  
    - Configure to send video URL or ID to Google Gemini transcription API.  
    - Set authentication credentials as required.  
    - Set onError to continue workflow without stopping.

21. **Add If node If transcript exists**  
    - Connect from Youtube Transcriber Gemini.  
    - Check if transcription output is not empty.

22. **Add Set node Edit Fields1**  
    - Connect from If transcript exists (true branch).  
    - Prepare transcription text and metadata for AI summarization input.

23. **Add LangChain Google Gemini Chat Model node Google Gemini Chat Model13**  
    - Connect from Edit Fields1.  
    - Configure model parameters for summarization.

24. **Add LangChain Structured Output Parser node Structured Output Parser1**  
    - Connect from Google Gemini Chat Model13.  
    - Configure to parse AI chat output into structured JSON.

25. **Add LangChain Chain LLM node youtube video summariser**  
    - Connect from Structured Output Parser1.  
    - Chain AI model and parser output for final summary.

26. **Add Google Sheets node Append or update row in sheet**  
    - Connect from youtube video summariser.  
    - Append or update video data and AI summary in target Google Sheet.

27. **Add Wait node Wait 70 sec for next video**  
    - Connect from Append or update row in sheet.  
    - Wait 70 seconds before processing next video.

28. **Connect Wait 70 sec for next video back to Loop Over Videos** to continue processing.

29. **Add Wait node Wait 85 sec on fail**  
    - Connect from If transcript exists (false branch).  
    - Wait 85 seconds before retrying or continuing to next video.

30. **Connect Wait 85 sec on fail back to Video Data and criteria** to retry or proceed.

31. **Add Set node End**  
    - Connect from Loop Over Videos (second output).  
    - Marks end of batch or workflow completion.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                              |
|------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| The code node "Videos Posted in Last X days" contains a configurable line (#6) to set the timeframe for filtering recent videos. Adjust as needed. | Code node configuration for date filtering.                                                |
| The workflow uses Google Gemini AI for both transcription and summarization—requires valid Google AI credentials. | Credential setup instructions for Google Gemini API.                                       |
| Wait nodes are used strategically to handle API rate limits from YouTube, Google Sheets, and Google AI services. | Ensures smooth operation respecting service limits.                                       |
| LangChain nodes are used for AI interaction, chaining model and parser nodes to structure AI output cleanly.     | See LangChain documentation for advanced customization.                                   |
| No sticky notes with content were present in this workflow to document additional insights or instructions.      | -                                                                                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an automation and integration tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.