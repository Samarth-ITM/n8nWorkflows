Discover HIDDEN Youtube Trends / Outlier Videos in Your Niche (Apify + Airtable)

https://n8nworkflows.xyz/workflows/discover-hidden-youtube-trends---outlier-videos-in-your-niche--apify---airtable--4187


# Discover HIDDEN Youtube Trends / Outlier Videos in Your Niche (Apify + Airtable)

### 1. Workflow Overview

This workflow, titled **"Discover HIDDEN Youtube Trends / Outlier Videos in Your Niche (Apify + Airtable)"**, automates weekly discovery and analysis of trending YouTube videos related to specific niche keywords. It leverages Apify’s YouTube scraper to collect video data, processes video transcripts with AI-driven summarization, and stores the distilled insights in an Airtable database for easy review and tracking.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Keyword Setup:** Weekly initiation and defining keywords to search.
- **1.2 Video Dataset Creation & Status Polling:** Requesting and waiting for Apify to prepare a dataset of relevant videos.
- **1.3 Video Data Retrieval & Iteration:** Fetching video metadata and iterating over each video.
- **1.4 Transcript Cleaning & AI Summarization:** Cleaning subtitles and generating structured summaries using AI.
- **1.5 Data Storage in Airtable:** Recording video and summary data into an Airtable base.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Keyword Setup

- **Overview:**  
  Initiates the workflow on a weekly schedule and sets the keywords to search for YouTube videos in the niche.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Setup Keywords

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow weekly on Mondays.  
    - Configuration: Interval set to trigger every week on Monday.  
    - Connections: Outputs to Setup Keywords.  
    - Edge Cases: Missed or delayed trigger if n8n instance is down; time zone considerations.

  - **Setup Keywords**  
    - Type: Set  
    - Role: Defines three niche keywords to be used for YouTube searches.  
    - Configuration: Assigns string values to `keyword1`, `keyword2`, and `keyword3`.  
    - Expressions: Keywords referenced later in HTTP request JSON body.  
    - Connections: Outputs to Create Videos Dataset node.  
    - Edge Cases: Changing number of keywords requires JSON update in dataset creation node.

#### 1.2 Video Dataset Creation & Status Polling

- **Overview:**  
  Sends a POST request to Apify to start scraping videos for the defined keywords, then repeatedly checks if the scraping run has completed.

- **Nodes Involved:**  
  - Create Videos Dataset  
  - Check IF Finished  
  - If  
  - Wait

- **Node Details:**  

  - **Create Videos Dataset**  
    - Type: HTTP Request (POST)  
    - Role: Initiates a YouTube scraping run on Apify with filters (e.g., videos from the past week, length filters).  
    - Key Config: JSON body includes keyword array, date filter set to “week”, and video properties filters.  
    - Expressions: Keywords injected from Setup Keywords node.  
    - Connections: Outputs to Check IF Finished.  
    - Edge Cases: API token must be valid; errors if token missing or quota exceeded; JSON must align with Apify API specs.

  - **Check IF Finished**  
    - Type: HTTP Request (GET)  
    - Role: Queries Apify API to get the status of the last scraping run.  
    - Configuration: URL uses the Apify API endpoint for last run, with API token.  
    - Connections: Outputs to If node.  
    - Edge Cases: Network timeouts, invalid token, API rate limits.

  - **If**  
    - Type: If Condition  
    - Role: Checks whether the scraping run status is “SUCCEEDED”.  
    - Configuration: Compares `$json.data.status` to string “SUCCEEDED”.  
    - Connections:  
      - True branch to Get Videos Data node  
      - False branch to Wait node (to delay and poll again).  
    - Edge Cases: Status could be other values (FAILED, RUNNING); logic depends on exact API response fields.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution before re-checking status to avoid rate limits.  
    - Configuration: Default wait duration (not specified, likely default seconds).  
    - Connections: Outputs back to Check IF Finished node.  
    - Edge Cases: Excessive waiting can delay entire workflow; too short delays may cause API throttling.

#### 1.3 Video Data Retrieval & Iteration

- **Overview:**  
  Once the dataset is ready, retrieve all video items and prepare for individual processing.

- **Nodes Involved:**  
  - Get Videos Data  
  - Loop Over Items  
  - No Operation, do nothing

- **Node Details:**  

  - **Get Videos Data**  
    - Type: HTTP Request (GET)  
    - Role: Fetches the dataset items (video metadata) from the last Apify scraping run.  
    - Configuration: Uses Apify API endpoint for dataset items, with API token.  
    - Connections: Outputs to Loop Over Items node.  
    - Edge Cases: Dataset may be empty; API errors or token issues.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Iterates over each video item individually for processing.  
    - Configuration: Uses default batch size (not specified).  
    - Connections: Outputs to two parallel branches: No Operation node and Text Cleaning node.  
    - Edge Cases: Large datasets may cause delays or memory issues.

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Placeholder node doing nothing, likely for workflow control or debugging.  
    - Connections: None (endpoint).  
    - Edge Cases: None.

#### 1.4 Transcript Cleaning & AI Summarization

- **Overview:**  
  Processes the YouTube video subtitles to clean raw transcript text and then summarizes the transcript using AI.

- **Nodes Involved:**  
  - Text Cleaning  
  - Text Synthesis

- **Node Details:**  

  - **Text Cleaning**  
    - Type: Code (JavaScript)  
    - Role: Cleans subtitle text by removing caption numbers, timing info, and formatting into a single paragraph string.  
    - Key Logic: Uses regex replacements on SRT subtitle format to produce clean transcript text.  
    - Connections: Outputs cleaned text to Text Synthesis node.  
    - Edge Cases: Subtitles array might be empty or missing; regex might fail if format changes.

  - **Text Synthesis**  
    - Type: Langchain Chain LLM Node  
    - Role: Uses an AI model to summarize the cleaned transcript into a structured format describing intro and video structure with bullet points.  
    - Configuration: Uses a prompt that instructs the AI not to invent but only summarize the transcript content, with specific output structure.  
    - Inputs: Accepts cleaned transcript text as input.  
    - Connections: Outputs summary text to Airtable node.  
    - Edge Cases: AI response failures; prompt interpretation variability; API quota or connectivity issues.

#### 1.5 Data Storage in Airtable

- **Overview:**  
  Stores all relevant video metadata and AI-generated summary into an Airtable base for tracking and analysis.

- **Nodes Involved:**  
  - Airtable

- **Node Details:**  

  - **Airtable**  
    - Type: Airtable  
    - Role: Creates new records in a specified Airtable base and table with video data and AI summary.  
    - Configuration:  
      - Maps fields such as video link, title, channel name, thumbnail URL (as an array for attachment), view counts, subscriber count, comments count, duration, and AI-generated structure summary.  
      - Calculates and stores a “VSC Ratio” field externally as a formula in Airtable (not computed here).  
    - Credentials: Requires Airtable Personal Access Token with appropriate permissions.  
    - Connections: Receives input from Text Synthesis node.  
    - Edge Cases: API authentication failures; schema mismatch; image upload issues if URL inaccessible.

---

### 3. Summary Table

| Node Name             | Node Type                       | Functional Role                            | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                |
|-----------------------|--------------------------------|--------------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                | Starts the workflow weekly                  | —                      | Setup Keywords           | SETUP: Scheduled Trigger frequency must align with dateFilter in Create Videos Dataset node.                               |
| Setup Keywords        | Set                            | Defines niche keywords for YouTube search  | Schedule Trigger        | Create Videos Dataset    | SETUP: Update keywords here and in Create Videos Dataset JSON accordingly.                                                  |
| Create Videos Dataset | HTTP Request (POST)             | Starts Apify YouTube scraping run           | Setup Keywords          | Check IF Finished        | SETUP: Replace [YOUR_API_TOKEN] with valid Apify token; see Apify docs https://docs.apify.com/api/v2/getting-started        |
| Check IF Finished     | HTTP Request (GET)              | Polls for scraping run status                | Create Videos Dataset, Wait | If                    |                                                                                                                            |
| If                    | If Condition                   | Checks if scraping run succeeded             | Check IF Finished       | Get Videos Data (true), Wait (false) |                                                                                                                            |
| Wait                  | Wait                           | Delays before re-checking scraping status   | If (false branch)       | Check IF Finished        |                                                                                                                            |
| Get Videos Data       | HTTP Request (GET)              | Retrieves scraped video metadata             | If (true branch)        | Loop Over Items          |                                                                                                                            |
| Loop Over Items       | Split In Batches               | Iterates over each video                      | Get Videos Data         | No Operation, Text Cleaning |                                                                                                                            |
| No Operation, do nothing | No Operation                 | Placeholder node                             | Loop Over Items         | —                       |                                                                                                                            |
| Text Cleaning         | Code (JavaScript)               | Cleans raw subtitle transcript               | Loop Over Items         | Text Synthesis           |                                                                                                                            |
| Text Synthesis        | Langchain Chain LLM             | Summarizes transcript into structured text  | Text Cleaning           | Airtable                 |                                                                                                                            |
| Airtable              | Airtable                        | Stores video data and summaries               | Text Synthesis          | Loop Over Items          | Airtable DB Fields explained, including how to upload thumbnails as URL array; VSC Ratio formula included.                  |
| Sticky Note           | Sticky Note                    | Describes overall workflow steps              | —                      | —                       | How this is Working? Stepwise explanation + YouTube tutorial link https://youtu.be/pH2hVaij3FY and help link https://tally.so/r/wayeqB |
| Sticky Note1          | Sticky Note                    | Airtable DB fields explanation                 | —                      | —                       | See Airtable DB Fields for storing video data and AI summaries.                                                             |
| Sticky Note2          | Sticky Note                    | Setup instructions and API token usage        | —                      | —                       | See SETUP instructions for scheduling, keyword setup, and API token replacement.                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Type: Schedule Trigger  
   - Set to trigger weekly on Monday.  
   - Connect to next node.

3. **Add a Set node named “Setup Keywords”:**
   - Assign three string variables: `keyword1`, `keyword2`, `keyword3`.  
   - Example values: `"Healthy food"`, `"Meal prep healthy"`, `"High Protein Snack"`.  
   - Connect Schedule Trigger node output to this node.

4. **Add an HTTP Request node named “Create Videos Dataset”:**
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/runs?token=[YOUR_API_TOKEN]` (replace token)  
   - Headers: Content-Type = application/json  
   - Body Type: JSON  
   - Body: Use JSON with parameters:
     ```json
     {
       "dateFilter": "week",
       "downloadSubtitles": true,
       "hasCC": false,
       "hasLocation": false,
       "hasSubtitles": false,
       "is360": false,
       "is3D": false,
       "is4K": false,
       "isBought": false,
       "isHD": false,
       "isHDR": false,
       "isLive": false,
       "isVR180": false,
       "lengthFilter": "between420",
       "maxResultStreams": 0,
       "maxResults": 3,
       "maxResultsShorts": 0,
       "preferAutoGeneratedSubtitles": false,
       "saveSubsToKVS": false,
       "searchQueries": [
         "{{ $json.keyword1 }}",
         "{{ $json.keyword2 }}",
         "{{ $json.keyword3 }}"
       ],
       "sortingOrder": "relevance",
       "subtitlesLanguage": "en"
     }
     ```
   - Connect Setup Keywords node output here.

5. **Add an HTTP Request node named “Check IF Finished”:**
   - Method: GET  
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/runs/last?token=[YOUR_API_TOKEN]`  
   - Connect output of Create Videos Dataset here.

6. **Add an If node named “If”:**
   - Condition: Check if `$json.data.status` equals `"SUCCEEDED"`.  
   - True branch to next node for data retrieval.  
   - False branch to Wait node.

7. **Add a Wait node:**
   - Default wait time (e.g. 30 seconds or adjust as needed).  
   - Connect False branch of If node to Wait node.  
   - Connect Wait node output back to Check IF Finished node to implement polling.

8. **Add an HTTP Request node named “Get Videos Data”:**
   - Method: GET  
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/runs/last/dataset/items?token=[YOUR_API_TOKEN]`  
   - Connect True branch of If node here.

9. **Add a Split In Batches node named “Loop Over Items”:**
   - Connect output of Get Videos Data here.  
   - Default batch size is fine.

10. **Add a No Operation node named “No Operation, do nothing”:**
    - Connect one output from Loop Over Items here (for workflow control).

11. **Add a Code node named “Text Cleaning”:**
    - JavaScript code to clean subtitles:
      ```js
      const input = $input.first().json.subtitles[0].srt;
      const cleaned = input
        .replace(/^\d+\s*$/gm, "")
        .replace(/^\d{2}:\d{2}:\d{1,2},\d{1,3} --> \d{2}:\d{2}:\d{1,2},\d{1,3}\s*$/gm, "")
        .replace(/\n{2,}/g, "\n")
        .replace(/\n/g, " ")
        .trim();
      return [{ cleanedText: cleaned }];
      ```
    - Connect second output from Loop Over Items to this node.

12. **Add Langchain Chain LLM node named “Text Synthesis”:**
    - Set prompt with instructions to summarize transcript into structured points.  
    - Input text: Use `{{ $json.cleanedText }}` from previous node.  
    - Connect output of Text Cleaning node here.  
    - Configure Langchain credentials as needed.

13. **Add Airtable node:**
    - Operation: Create  
    - Base and Table: Connect to your Airtable base and table (replace placeholders).  
    - Map fields:
      - Link: `={{ $('Loop Over Items').item.json.url }}`  
      - Title: `={{ $('Loop Over Items').item.json.title }}`  
      - Duration: `={{ $('Loop Over Items').item.json.duration }}`  
      - Structure: `={{ $json.text }}` (AI summary)  
      - SubsCount: `={{ $('Loop Over Items').item.json.numberOfSubscribers }}`  
      - Thumbnail: `={{ [{ "url": $('Loop Over Items').item.json.thumbnailUrl }] }}`  
      - ViewsCount: `={{ $('Loop Over Items').item.json.viewCount }}`  
      - ChannelName: `={{ $('Loop Over Items').item.json.channelName }}`  
      - CommentsCount: `={{ $('Loop Over Items').item.json.commentsCount }}`  
    - Connect output of Text Synthesis node here.  
    - Provide Airtable Personal Access Token credentials.

14. **Optional:** Add Sticky Notes for documentation reflecting setup instructions, Airtable schema explanations, and workflow logic.

15. **Replace all `[YOUR_API_TOKEN]` and Airtable placeholders with your actual API tokens and base URLs.**

16. **Activate the workflow and test with sample keywords.**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                               |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow scheduled trigger frequency must match the `dateFilter` parameter in the "Create Videos Dataset" node JSON. | Setup Instructions                                            |
| To upload images to Airtable attachment fields, URLs must be wrapped in an array of objects: `[{ "url": "..." }]`.    | Airtable node field mapping                                   |
| The "VSC Ratio" field in Airtable should be a formula: `(ViewsCount/SubsCount)*0.7 + (CommentsCount/ViewsCount)*0.3`. | Airtable field usage                                          |
| Apify YouTube Scraper API documentation for reference.                                                               | https://docs.apify.com/api/v2/getting-started                  |
| YouTube video tutorial explaining the workflow steps.                                                                | https://youtu.be/pH2hVaij3FY                                  |
| Support link for workflow help requests.                                                                              | https://tally.so/r/wayeqB                                     |

---

**Disclaimer:**  
The content provided is exclusively generated from an automated n8n workflow. It respects all applicable content policies and contains no illegal or offensive elements. All data processed is legal and publicly accessible.