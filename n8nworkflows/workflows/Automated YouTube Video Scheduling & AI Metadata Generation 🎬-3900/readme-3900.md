Automated YouTube Video Scheduling & AI Metadata Generation 🎬

https://n8nworkflows.xyz/workflows/workflow-3900-1747220148235.png


# Automated YouTube Video Scheduling & AI Metadata Generation 🎬

### 1. Workflow Overview

This workflow automates the scheduling and metadata optimization of YouTube videos for content creators, marketing teams, and channel managers. It addresses the manual effort and inconsistency in video publishing by extracting transcripts, generating SEO-optimized descriptions and tags, setting videos to private for scheduling, and publishing them at strategic times.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Video Retrieval:** Periodically triggers the workflow and fetches videos needing metadata updates or scheduling.
- **1.2 Video Filtering & Looping:** Filters unpublished or private/unlisted videos and loops over each video for individual processing.
- **1.3 Video Metadata Retrieval:** Retrieves detailed metadata for each video.
- **1.4 Transcript Extraction & Formatting:** Uses Apify to extract video transcripts and formats them for AI processing.
- **1.5 AI Metadata Generation:** Generates SEO-optimized descriptions, tags, and optionally titles using OpenAI and Google Gemini language models.
- **1.6 Video Metadata Update & Scheduling:** Updates the video metadata on YouTube, sets videos to private, and schedules publishing at calculated future dates.
- **1.7 Utility & Control Nodes:** Includes nodes for deduplication, waiting, and conditional branching to ensure smooth execution and avoid processing duplicates.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Video Retrieval

**Overview:**  
This block triggers the workflow on a schedule and fetches the latest videos from the YouTube channel to identify candidates for metadata enhancement and scheduling.

**Nodes Involved:**  
- Every Day (Schedule Trigger)  
- Get Videos to reschedule (YouTube node)  
- Get video Ids separated (Code node)  
- Loop over Video IDs (SplitInBatches node)  

**Node Details:**

- **Every Day**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution daily at a configured time (disabled by default).  
  - Configuration: Triggers at 14:22 daily (can be customized).  
  - Inputs: None  
  - Outputs: Connects to "Get Videos to reschedule"  
  - Edge Cases: Disabled by default; ensure enabled for automation.

- **Get Videos to reschedule**  
  - Type: YouTube API node (video resource, list operation)  
  - Role: Retrieves recent videos from the channel, limited to 2 by default, ordered by date.  
  - Credentials: Uses configured YouTube OAuth2 credentials with upload permissions.  
  - Inputs: Trigger from schedule node  
  - Outputs: Video list JSON objects  
  - Edge Cases: API quota limits, empty results if no videos uploaded recently.

- **Get video Ids separated**  
  - Type: Code (JavaScript)  
  - Role: Extracts video IDs from the YouTube API response and outputs each ID as a separate item for individual processing.  
  - Key Logic: Iterates over input items, extracts `id.videoId` property, returns array of items with `videoId`.  
  - Inputs: Video list JSON from previous node  
  - Outputs: Individual videoId items  
  - Edge Cases: Handles missing or malformed videoId gracefully by skipping.

- **Loop over Video IDs**  
  - Type: SplitInBatches  
  - Role: Processes video IDs in batches (default batch size) to avoid API rate limits and manage workflow load.  
  - Inputs: VideoId items from code node  
  - Outputs: Each batch passed downstream for further processing  
  - Edge Cases: Batch size can be adjusted to balance throughput and quota.

---

#### 2.2 Video Filtering & Looping

**Overview:**  
Filters videos to only those that are private or unlisted (eligible for scheduling) and loops over them for detailed processing.

**Nodes Involved:**  
- Get Video Data (YouTube node)  
- Return Private Videos (Code node)  
- Loop over All Videos not Published (SplitInBatches node)  

**Node Details:**

- **Get Video Data**  
  - Type: YouTube API node (video resource, get operation)  
  - Role: Retrieves full metadata for each videoId passed in.  
  - Configuration: Uses videoId from input item, fetches detailed video info.  
  - Inputs: VideoId from "Loop over Video IDs"  
  - Outputs: Full video metadata JSON  
  - Edge Cases: API errors if videoId invalid or access denied.

- **Return Private Videos**  
  - Type: Code (JavaScript)  
  - Role: Filters videos to only those with privacyStatus "private" or "unlisted" and sorts them by published date. Also calculates scheduled publish dates (next Fridays at 17:00 UTC, spaced weekly).  
  - Key Logic:  
    - Filters input items by privacyStatus  
    - Sorts by snippet.publishedAt ascending  
    - Assigns publishAt dates incrementally for scheduling  
  - Outputs: Array of video objects with videoId, publishAt, title, privacy, and selfDeclaredMadeForKids flag.  
  - Edge Cases: Returns empty if no videos match filter; ensures publishAt dates are in ISO 8601 format without milliseconds.

- **Loop over All Videos not Published**  
  - Type: SplitInBatches  
  - Role: Processes filtered videos in batches for update operations.  
  - Inputs: Filtered video list from "Return Private Videos"  
  - Outputs: Batches of videos for metadata update and scheduling  
  - Edge Cases: Batch size can be tuned.

---

#### 2.3 Video Metadata Retrieval (Supporting Node)

**Overview:**  
Retrieves the current title of each video for use in scheduling updates.

**Nodes Involved:**  
- gettitle (YouTube node)  

**Node Details:**

- **gettitle**  
  - Type: YouTube API node (video resource, get operation)  
  - Role: Fetches the current title of the video by videoId.  
  - Inputs: VideoId from "Loop over All Videos not Published"  
  - Outputs: Video metadata including title  
  - Edge Cases: API errors if videoId invalid.

---

#### 2.4 Transcript Extraction & Formatting

**Overview:**  
Extracts the transcript of each video using Apify's YouTube transcript scraper and formats the transcript text for AI processing.

**Nodes Involved:**  
- Get Transcript (HTTP Request node)  
- Adjust Transcript Format (Code node)  

**Node Details:**

- **Get Transcript**  
  - Type: HTTP Request  
  - Role: Calls Apify API to retrieve the transcript dataset for the video URL.  
  - Configuration:  
    - POST request to Apify actor endpoint with JSON body containing YouTube video URL  
    - Query parameter includes Apify API token (must be configured)  
  - Inputs: Video metadata with videoId  
  - Outputs: Transcript data array from Apify  
  - Edge Cases:  
    - Requires valid Apify API token  
    - Transcript only available if video is Unlisted or Published (not private)  
    - API rate limits or failures possible

- **Adjust Transcript Format**  
  - Type: Code (JavaScript)  
  - Role: Converts Apify transcript JSON array into a single concatenated string for AI input.  
  - Logic: Extracts `text` from each transcript segment, joins with spaces.  
  - Inputs: Transcript JSON array from Apify  
  - Outputs: Single string property `transcript`  
  - Edge Cases: Handles missing or malformed transcript segments gracefully.

---

#### 2.5 AI Metadata Generation

**Overview:**  
Generates SEO-optimized video descriptions, tags, and optionally titles using AI language models based on the formatted transcript.

**Nodes Involved:**  
- Create Description (OpenAI node)  
- YT Tags (LangChain Agent node)  
- YT Title (OpenAI node, disabled)  
- 2.5FlashPrev (Google Gemini node, linked to YT Tags)  

**Node Details:**

- **Create Description**  
  - Type: OpenAI (LangChain)  
  - Role: Generates a detailed but concise video description in German, written in first person, with confident language and emojis, ending with hashtags.  
  - Configuration:  
    - Model: GPT-4.1-NANO  
    - System prompt instructs style and content rules  
    - Input transcript injected dynamically  
  - Inputs: Formatted transcript from "Adjust Transcript Format"  
  - Outputs: AI-generated description text  
  - Edge Cases: Requires valid OpenAI API credentials; prompt failures or rate limits possible.

- **YT Tags**  
  - Type: LangChain Agent  
  - Role: Generates general, SEO-friendly YouTube tags in German, separated by commas, based on transcript.  
  - Configuration:  
    - System message defines tag generation rules and examples  
    - Input transcript dynamically injected  
  - Inputs: Formatted transcript  
  - Outputs: Comma-separated tags string  
  - Edge Cases: Requires Google Palm API credentials; language model availability.

- **YT Title** (Disabled)  
  - Type: OpenAI (LangChain)  
  - Role: Optionally generates a short SEO-optimized video title from transcript (max 100 characters).  
  - Disabled: Not active by default, but can be enabled to override titles.  
  - Inputs: Formatted transcript  
  - Outputs: AI-generated title  
  - Edge Cases: Same as Create Description.

- **2.5FlashPrev**  
  - Type: Google Gemini Language Model  
  - Role: Connected to YT Tags node as an AI language model provider (used internally by YT Tags).  
  - Inputs: From Create Description node  
  - Outputs: To YT Tags node  
  - Edge Cases: Requires Google Palm API credentials.

---

#### 2.6 Video Metadata Update & Scheduling

**Overview:**  
Updates YouTube videos with generated metadata, sets privacy to private, and schedules publishing at calculated future dates.

**Nodes Involved:**  
- Set Publish Date (YouTube node)  
- Update Video's Metadata (YouTube node)  
- 3s and 4s (Wait nodes)  

**Node Details:**

- **Set Publish Date**  
  - Type: YouTube API node (video resource, update operation)  
  - Role: Updates video metadata including title, privacyStatus (set to private), categoryId, regionCode, and sets the `publishAt` date for scheduled publishing.  
  - Inputs: Video metadata with calculated publishAt date from "Return Private Videos"  
  - Outputs: Updated video info passed downstream  
  - Edge Cases:  
    - Videos must be private for `publishAt` scheduling to work  
    - API quota and permission errors possible

- **Update Video's Metadata**  
  - Type: YouTube API node (video resource, update operation)  
  - Role: Applies AI-generated description and tags to the video, keeps title from latest video fetch.  
  - Inputs: AI-generated tags and description, videoId from latest video metadata  
  - Outputs: Passes to wait node for pacing  
  - Edge Cases: API errors, quota limits.

- **3s and 4s (Wait nodes)**  
  - Type: Wait  
  - Role: Introduce delays (3 and 4 seconds respectively) to avoid hitting API rate limits and ensure sequential processing.  
  - Inputs/Outputs: Control flow pacing between update nodes.  
  - Edge Cases: None significant.

---

#### 2.7 Utility & Control Nodes

**Overview:**  
Support nodes for deduplication, conditional branching, and manual triggering.

**Nodes Involved:**  
- Remove Duplicates from previous Runs (RemoveDuplicates node)  
- new video? (If node)  
- When clicking ‘Test workflow’ (Manual Trigger node)  

**Node Details:**

- **Remove Duplicates from previous Runs**  
  - Type: RemoveDuplicates  
  - Role: Prevents reprocessing of videos already handled in previous workflow executions by deduplicating on videoId.  
  - Inputs: VideoId items  
  - Outputs: Filtered unique videoId items  
  - Edge Cases: If database cleared, duplicates may reappear.

- **new video?**  
  - Type: If node  
  - Role: Checks if the current videoId is new (not seen before) based on deduplication results.  
  - Outputs: Routes new videos for processing, others bypass AI metadata generation.  
  - Edge Cases: Logic depends on RemoveDuplicates output.

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution of the workflow for testing purposes.  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name                         | Node Type                      | Functional Role                                  | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                          |
|----------------------------------|--------------------------------|-------------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Every Day                        | Schedule Trigger               | Triggers workflow daily                          | None                             | Get Videos to reschedule         | Disabled by default; configure schedule time                                                        |
| Get Videos to reschedule         | YouTube API (video:list)       | Fetches recent videos                            | Every Day                       | Get video Ids separated          |                                                                                                    |
| Get video Ids separated          | Code                          | Extracts video IDs from video list              | Get Videos to reschedule         | Loop over Video IDs              |                                                                                                    |
| Loop over Video IDs              | SplitInBatches                | Processes video IDs in batches                   | Get video Ids separated          | Get Video Data, Return Private Videos |                                                                                                    |
| Get Video Data                  | YouTube API (video:get)        | Retrieves detailed video metadata                | Loop over Video IDs              | Loop over Video IDs (second output) |                                                                                                    |
| Return Private Videos            | Code                          | Filters videos by privacy and schedules publish | Loop over Video IDs              | Loop over All Videos not Published | Sticky Note: "## Code only returns the videos that are not listed"                                 |
| Loop over All Videos not Published | SplitInBatches                | Processes filtered videos in batches             | Return Private Videos            | 4s, gettitle                    |                                                                                                    |
| gettitle                       | YouTube API (video:get)        | Retrieves current video title                     | Loop over All Videos not Published | Set Publish Date                |                                                                                                    |
| Set Publish Date                | YouTube API (video:update)     | Sets video to private and schedules publishing   | gettitle                       | Loop over All Videos not Published | Sticky Note: "## Video needs to be set to private TOGETHER with the PublishAt parameter in order for it to work." |
| Remove Duplicates from previous Runs | RemoveDuplicates             | Avoids reprocessing videos                        | get video id                   | new video?                     |                                                                                                    |
| new video?                     | If                            | Checks if video is new                            | Remove Duplicates from previous Runs | getLatestVideoID (if false)    |                                                                                                    |
| getLatestVideoID               | YouTube API (video:get)        | Retrieves latest video metadata                   | new video?                     | Get Transcript                 |                                                                                                    |
| Get Transcript                | HTTP Request                   | Calls Apify to get video transcript               | getLatestVideoID               | Adjust Transcript Format        | Sticky Note: "### Video needs to be Unlisted or Published in order for the scraper to be able to get the transcript" |
| Adjust Transcript Format       | Code                          | Formats transcript JSON to plain text             | Get Transcript                 | Create Description             |                                                                                                    |
| Create Description             | OpenAI (LangChain)             | Generates SEO-optimized video description         | Adjust Transcript Format       | YT Tags, YT Title              | Sticky Note: "Adjust the Prompts 👉🏻"                                                               |
| YT Tags                       | LangChain Agent                | Generates SEO-friendly YouTube tags                | Create Description             | Update Video's Metadata        |                                                                                                    |
| YT Title                      | OpenAI (LangChain)             | Generates SEO-optimized video title (disabled)    | Adjust Transcript Format       | None                         | Sticky Note: "🎥Title is kept from the upload, alternatively you can just add the YT Title module in the mix" |
| Update Video's Metadata       | YouTube API (video:update)     | Updates video description and tags                 | YT Tags                       | 3s                            |                                                                                                    |
| 3s                           | Wait                          | Waits 3 seconds to pace API calls                   | Update Video's Metadata        | Loop Over Items               |                                                                                                    |
| 4s                           | Wait                          | Waits 4 seconds to pace API calls                   | Loop over All Videos not Published | Fetch Latest Videos          |                                                                                                    |
| Fetch Latest Videos           | YouTube API (video:list)       | Fetches latest video for metadata update           | 4s                            | Loop Over Items               |                                                                                                    |
| Loop Over Items              | SplitInBatches                | Processes latest videos in batches                  | Fetch Latest Videos            | get video id                 |                                                                                                    |
| get video id                 | Set                           | Sets videoId property for deduplication             | Loop Over Items               | Remove Duplicates from previous Runs |                                                                                                    |
| When clicking ‘Test workflow’ | Manual Trigger                | Manual workflow trigger                             | None                         | Get Videos to reschedule      |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node ("Every Day")**  
   - Type: Schedule Trigger  
   - Configure to run daily at your preferred time (e.g., 14:22).  
   - Initially disabled for testing; enable when ready.

2. **Add YouTube node ("Get Videos to reschedule")**  
   - Resource: Video, Operation: List  
   - Limit: 2 (adjust as needed)  
   - Order: Date descending  
   - Credentials: Configure YouTube OAuth2 with upload permissions.

3. **Add Code node ("Get video Ids separated")**  
   - JavaScript code to extract `videoId` from each video item:  
     ```js
     const resultItems = [];
     for (const item of items) {
       if (item.json && item.json.id && item.json.id.videoId) {
         resultItems.push({ json: { videoId: item.json.id.videoId } });
       }
     }
     return resultItems;
     ```
   - Connect output of "Get Videos to reschedule" to this node.

4. **Add SplitInBatches node ("Loop over Video IDs")**  
   - Default batch size  
   - Connect from "Get video Ids separated".

5. **Add YouTube node ("Get Video Data")**  
   - Resource: Video, Operation: Get  
   - Video ID: `={{ $json.videoId }}`  
   - Credentials: Same YouTube OAuth2  
   - Connect from "Loop over Video IDs".

6. **Add Code node ("Return Private Videos")**  
   - JavaScript code to filter videos with privacyStatus "private" or "unlisted", sort by published date, and assign scheduled publish dates on upcoming Fridays at 17:00 UTC spaced weekly.  
   - Use the provided code from the workflow for exact logic.  
   - Connect from "Loop over Video IDs" (second output).

7. **Add SplitInBatches node ("Loop over All Videos not Published")**  
   - Default batch size  
   - Connect from "Return Private Videos".

8. **Add YouTube node ("gettitle")**  
   - Resource: Video, Operation: Get  
   - Video ID: `={{ $json.videoId }}`  
   - Connect from "Loop over All Videos not Published".

9. **Add YouTube node ("Set Publish Date")**  
   - Resource: Video, Operation: Update  
   - Video ID: `={{ $json.id }}`  
   - Title: `={{ $json.snippet.title }}`  
   - Update Fields:  
     - publishAt: `={{ $json.publishAt }}` (from scheduled dates)  
     - privacyStatus: "private"  
     - categoryId: "25" (or your preferred category)  
     - regionCode: "DE" (or your region)  
   - Connect from "gettitle".

10. **Add RemoveDuplicates node ("Remove Duplicates from previous Runs")**  
    - Operation: Remove items seen in previous executions  
    - Dedupe Value: `={{ $json.id.videoId }}`  
    - Connect from "Loop Over Items" (see step 16).

11. **Add If node ("new video?")**  
    - Condition: Check if current video is new (not in previous runs)  
    - Connect from "Remove Duplicates from previous Runs".

12. **Add YouTube node ("getLatestVideoID")**  
    - Resource: Video, Operation: Get  
    - Video ID: `={{ $json.id.videoId }}`  
    - Connect from "new video?" (false output).

13. **Add HTTP Request node ("Get Transcript")**  
    - Method: POST  
    - URL: `https://api.apify.com/v2/acts/pintostudio~youtube-transcript-scraper/run-sync-get-dataset-items`  
    - Query Parameter: token = Your Apify API token  
    - JSON Body: `{ "videoUrl": "https://www.youtube.com/watch?v={{ $json.id }}" }`  
    - Connect from "getLatestVideoID".

14. **Add Code node ("Adjust Transcript Format")**  
    - JavaScript to concatenate transcript text segments into one string.  
    - Connect from "Get Transcript".

15. **Add OpenAI node ("Create Description")**  
    - Model: GPT-4.1-NANO  
    - System prompt: Professional text writer instructions for German SEO-optimized description in first person with emojis and hashtags.  
    - Input: Formatted transcript from previous node.  
    - Credentials: OpenAI API key.  
    - Connect from "Adjust Transcript Format".

16. **Add LangChain Agent node ("YT Tags")**  
    - System message: Generate general German YouTube tags separated by commas based on transcript.  
    - Input: Formatted transcript or output from "Create Description".  
    - Credentials: Google Palm API key.  
    - Connect from "Create Description".

17. **Add YouTube node ("Update Video's Metadata")**  
    - Resource: Video, Operation: Update  
    - Video ID: From latest video metadata  
    - Update Fields:  
      - description: AI-generated description  
      - tags: AI-generated tags  
      - categoryId: "25"  
      - regionCode: "DE"  
    - Connect from "YT Tags".

18. **Add Wait nodes ("3s" and "4s")**  
    - Add 3-second and 4-second wait nodes to pace API calls.  
    - Connect "Update Video's Metadata" to "3s", then "Loop Over Items" to "Remove Duplicates", and "Loop over All Videos not Published" to "4s" and then "Fetch Latest Videos".

19. **Add YouTube node ("Fetch Latest Videos")**  
    - Resource: Video, Operation: List  
    - Limit: 1  
    - Order: Date descending  
    - Connect from "4s".

20. **Add SplitInBatches node ("Loop Over Items")**  
    - Connect from "Fetch Latest Videos".

21. **Add Set node ("get video id")**  
    - Assign `id.videoId` from input JSON to a new field for deduplication.  
    - Connect from "Loop Over Items".

22. **Connect "get video id" to "Remove Duplicates from previous Runs"** (step 10).

23. **Add Manual Trigger node ("When clicking ‘Test workflow’")**  
    - For manual testing.  
    - Connect to "Get Videos to reschedule".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Videos MUST be uploaded as private initially for the `publishAt` scheduling to work correctly.                                                                                                                                           | Important YouTube API requirement for scheduled publishing.                                        |
| Transcript extraction requires videos to be Unlisted or Published; private videos cannot have transcripts scraped.                                                                                                                     | See Sticky Note6 in workflow.                                                                      |
| Adjust prompt templates in AI nodes to match your brand voice and SEO strategy.                                                                                                                                                          | Customization advice for content creators.                                                         |
| API quotas for YouTube and Apify should be monitored to avoid failures during batch processing.                                                                                                                                          | Operational consideration.                                                                         |
| Workflow author and resources: GitHub [github.com/JimPresting](https://github.com/JimPresting), YouTube channel [youtube.com/@StardawnAI](https://www.youtube.com/@StardawnAI)                                                           | Source and support links.                                                                           |
| To avoid processing duplicates, the Remove Duplicates node uses a persistent database; clearing it resets the history.                                                                                                                  | Workflow state management.                                                                         |
| Scheduling logic sets publish dates on upcoming Fridays at 17:00 UTC, spaced weekly per video.                                                                                                                                           | Scheduling strategy detail.                                                                         |
| The workflow includes disabled nodes (e.g., YT Title) for optional enhancements.                                                                                                                                                         | Flexibility for users to enable as needed.                                                        |
| API credentials (YouTube OAuth2, OpenAI, Google Palm, Apify) must be securely configured in n8n credentials manager before running the workflow.                                                                                        | Security best practice.                                                                             |
| Webhook URLs (for wait nodes with webhookId) should be protected to prevent unauthorized triggering.                                                                                                                                     | Security best practice.                                                                             |

---

This comprehensive documentation enables advanced users and AI agents to understand, reproduce, and customize the YouTube video scheduling and metadata generation workflow effectively.