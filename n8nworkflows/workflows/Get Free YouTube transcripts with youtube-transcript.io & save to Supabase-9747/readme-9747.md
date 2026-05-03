Get Free YouTube transcripts with youtube-transcript.io & save to Supabase

https://n8nworkflows.xyz/workflows/get-free-youtube-transcripts-with-youtube-transcript-io---save-to-supabase-9747


# Get Free YouTube transcripts with youtube-transcript.io & save to Supabase

---

### 1. Workflow Overview

This workflow automates the process of fetching recent videos from specified YouTube channels, retrieving their transcripts via the external service youtube-transcript.io, and saving these transcripts into a Supabase database. It is designed for content creators, researchers, or automation enthusiasts who want to systematically archive and analyze YouTube video transcripts.

The workflow is logically divided into three main blocks:

- **1.1 Fetch Recent Videos**: Receive a list of YouTube channel IDs, validate and convert them into RSS feed URLs, fetch recent videos, and loop over each video.
- **1.2 Video Transcription**: Extract and validate the YouTube video ID from the video URL, query the youtube-transcript.io API for the transcript, parse the transcript response, and handle failures.
- **1.3 Save Transcripts**: Combine video metadata with the transcript and save the resulting data into a Supabase table, incorporating a controlled wait to manage API limits and rate constraints.

---

### 2. Block-by-Block Analysis

#### 2.1 Block: Fetch Recent Videos

- **Overview:**  
  This block initializes the process by defining which YouTube channels to track, validating channel IDs, constructing RSS feed URLs, and retrieving the list of recent videos for each channel. It splits the input channels and videos into batches for iterative processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Channels To Track (Set)  
  - Split Out (Split Out)  
  - Channel Info (Set)  
  - Verify Channel ID + Create RSS Link (Code)  
  - Channel Info + Channel ID (Set)  
  - Loop Over Each Channel (Split In Batches)  
  - Find Channel's Videos (RSS Feed Read)  
  - Loop Over New Videos (Split In Batches)  
  - Filter Out YouTube Shorts (If)  

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution  
    - Configuration: Default  
    - Inputs/Outputs: No input; outputs trigger downstream nodes  
    - Edge Cases: None  
    - Notes: Replaceable by a Schedule node for automation

  - **Channels To Track**  
    - Type: Set  
    - Role: Defines YouTube channel IDs to monitor  
    - Configuration: Assigns an array of channel IDs to the field `source_identifier`  
    - Inputs: Trigger node  
    - Outputs: One output with channel array  
    - Edge Cases: Empty or invalid channel list results in no processing  
    - Sticky Note: Advises how to obtain channel IDs (e.g., tunepocket.com)

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits channel IDs array into individual items for looping  
    - Configuration: Splits on `source_identifier` field  
    - Inputs: Channels To Track  
    - Outputs: One item per channel ID  
    - Edge Cases: Empty arrays produce no output; must handle gracefully  
    - Sticky Note: Explains purpose of splitting for loop processing

  - **Channel Info**  
    - Type: Set  
    - Role: Assigns the individual channel ID to a field `youtubeChannels`  
    - Configuration: Sets `youtubeChannels` to the current item’s `source_identifier`  
    - Inputs: Split Out  
    - Outputs: Single item with channel ID field  
    - Edge Cases: If input missing, downstream logic may fail

  - **Verify Channel ID + Create RSS Link**  
    - Type: Code (JavaScript)  
    - Role: Validates channel ID format and constructs RSS feed URL  
    - Configuration:  
      - Checks if `youtubeChannels` exists and matches YouTube channel ID pattern (starts with “UC” and 24 characters)  
      - Builds RSS URL: `https://www.youtube.com/feeds/videos.xml?channel_id=CHANNEL_ID`  
      - Returns null for invalid IDs to skip processing  
    - Inputs: Channel Info  
    - Outputs: Validated channel ID and RSS URL or null  
    - Edge Cases: Invalid or missing channel IDs are skipped  
    - Retry on Fail: Enabled  
    - Sticky Note: Marks it as the step to convert to valid RSS feed links

  - **Channel Info + Channel ID**  
    - Type: Set  
    - Role: Passes RSS URL downstream for fetching videos  
    - Configuration: Assigns the generated `rssUrl` from previous code node  
    - Inputs: Verify Channel ID + Create RSS Link  
    - Outputs: Items with `rssUrl` field  
    - Edge Cases: If input null, no output produced

  - **Loop Over Each Channel**  
    - Type: Split In Batches  
    - Role: Processes channels one by one to avoid overload  
    - Configuration: Default batch size  
    - Inputs: Channel Info + Channel ID  
    - Outputs: Single channel per batch  
    - Edge Cases: Large channel lists can slow processing

  - **Find Channel's Videos**  
    - Type: RSS Feed Read  
    - Role: Fetches the RSS feed of videos for the current channel  
    - Configuration: Uses `rssUrl` field dynamically  
    - Inputs: Loop Over Each Channel  
    - Outputs: Items representing videos from the feed  
    - Retry on Fail: Enabled  
    - Edge Cases: RSS feed unavailable or malformed, network errors

  - **Loop Over New Videos**  
    - Type: Split In Batches  
    - Role: Processes each video individually  
    - Configuration: Default batch size  
    - Inputs: Loop Over Each Channel (also from Wait node later)  
    - Outputs: Single video per batch  
    - Edge Cases: Large video lists increase run time

  - **Filter Out YouTube Shorts**  
    - Type: If  
    - Role: Filters out YouTube Shorts videos by checking if the video link contains “youtube.com/shorts”  
    - Configuration:  
      - Checks if `link` field exists and does not contain “youtube.com/shorts”  
      - Left branch: passes non-shorts videos  
      - Right branch: redirects shorts back to Loop Over New Videos if user wants to include shorts  
    - Inputs: Loop Over New Videos  
    - Outputs: Filtered videos for further processing  
    - Edge Cases: Shorts videos excluded by default; can be included by removing this filter  
    - Sticky Note: Explains how to include Shorts by deleting the second filter condition

---

#### 2.2 Block: Video Transcription

- **Overview:**  
  This block extracts a clean YouTube video ID from the video URL, validates it, sends a request to youtube-transcript.io API to retrieve the transcript, parses the transcript JSON, and handles failure scenarios.

- **Nodes Involved:**  
  - Rename Original URL (Set)  
  - Find Video ID (Set)  
  - Is Video ID valid? (Set)  
  - Clean Up URL (Set)  
  - Was Video ID Found? (If)  
  - Rename URL (Set)  
  - try to get video_id again (Code)  
  - Merge Video ID With Video Data (Merge)  
  - Get Transcript From API (HTTP Request)  
  - Transcript Worked? (If)  
  - Parse Transcript from API Response (Code)  
  - Transcript Failed (Stop and Error)  
  - Add Transcript to Video Data (Merge)  

- **Node Details:**  

  - **Rename Original URL**  
    - Type: Set  
    - Role: Renames the video `link` to `original_url` for consistency  
    - Configuration: Sets `original_url` field to current `link`  
    - Inputs: Filter Out YouTube Shorts (left branch)  
    - Outputs: Items with `original_url` field added  
    - Edge Cases: Missing `link` can cause null values downstream

  - **Find Video ID**  
    - Type: Set  
    - Role: Extracts the YouTube video ID from `original_url` with a regex  
    - Configuration: Uses regex to match 11-character video ID from URLs  
    - Assigns extracted ID to `video_id` field  
    - Inputs: Rename Original URL  
    - Outputs: Items with `video_id` field  
    - Edge Cases: Invalid URLs yield null `video_id`

  - **Is Video ID valid?**  
    - Type: Set  
    - Role: Validates whether `video_id` is present and valid (non-null)  
    - Configuration: Boolean field `is_valid` set based on regex match success  
    - Inputs: Find Video ID  
    - Outputs: Items with `is_valid` boolean  
    - Edge Cases: False for invalid or null IDs

  - **Clean Up URL**  
    - Type: Set  
    - Role: Constructs a clean YouTube watch URL from `video_id`  
    - Configuration: If valid `video_id` exists, sets `clean_url` to standard YouTube watch URL  
    - Inputs: Is Video ID valid?  
    - Outputs: Items with `clean_url`  
    - Edge Cases: Null if no valid `video_id`

  - **Was Video ID Found?**  
    - Type: If  
    - Role: Branches workflow based on validity of `is_valid` flag  
    - Configuration: Checks if `is_valid` is true  
    - True branch: proceeds to merge video ID with video data  
    - False branch: attempts to retry extraction via try to get video_id again  
    - Inputs: Clean Up URL  
    - Outputs: Two branches for valid and invalid IDs

  - **Rename URL**  
    - Type: Set  
    - Role: Renames `original_url` to `url` for consistency in retry branch  
    - Configuration: Sets `url` field from `original_url`  
    - Inputs: Was Video ID Found? (False branch)  
    - Outputs: Items for retry extraction

  - **try to get video_id again**  
    - Type: Code (JavaScript)  
    - Role: Alternative method to extract `video_id` from multiple possible URL fields  
    - Configuration: Uses a robust regex and loops over all input URLs to extract video IDs  
    - Inputs: Rename URL  
    - Outputs: Items with `video_id`, `is_valid`, `clean_url` fields  
    - Retry on Fail: Enabled  
    - Edge Cases: May still fail if URLs are malformed

  - **Merge Video ID With Video Data**  
    - Type: Merge  
    - Role: Combines video metadata and video ID data for downstream processing  
    - Configuration: Combines by position (aligned indices)  
    - Inputs: Was Video ID Found? (True branch) and try to get video_id again  
    - Outputs: Combined video data with ID fields

  - **Get Transcript From API**  
    - Type: HTTP Request  
    - Role: Sends POST request to youtube-transcript.io API with video ID to fetch transcript  
    - Configuration:  
      - URL: `https://www.youtube-transcript.io/api/transcripts`  
      - Method: POST  
      - Headers: `Content-Type: application/json`, API key via HTTP Header Auth credential  
      - Body: JSON with array containing one video ID  
      - Response: Text with full response, never error to allow graceful handling  
    - Credentials: Uses externally configured HTTP Header Auth credential with API key  
    - Retry on Fail: Enabled with 5-second wait between tries  
    - Edge Cases: API limits (25 free per month), service overload (503 errors), videos without captions  
    - Sticky Note: Provides link for API key registration and notes on captions requirement

  - **Transcript Worked?**  
    - Type: If  
    - Role: Checks if transcript API call succeeded by verifying `statusMessage` equals “OK”  
    - Configuration: Condition on `statusMessage` field  
    - True branch: parse transcript response  
    - False branch: stops workflow with error  
    - Inputs: Get Transcript From API  
    - Outputs: Two branches for success or failure

  - **Parse Transcript from API Response**  
    - Type: Code (JavaScript)  
    - Role: Parses JSON transcript data from API response, concatenates transcript text from tracks  
    - Configuration:  
      - Parses `data` property (string or array)  
      - Extracts transcript text from first track  
      - Outputs objects with `id` and `transcript` fields  
    - Inputs: Transcript Worked? (True branch)  
    - Outputs: Parsed transcript data

  - **Transcript Failed**  
    - Type: Stop and Error  
    - Role: Stops workflow execution on failure with a custom error message “Transcript Failed”  
    - Inputs: Transcript Worked? (False branch)  
    - Outputs: None (terminates execution)

  - **Add Transcript to Video Data**  
    - Type: Merge  
    - Role: Combines parsed transcript data with video metadata for final output  
    - Configuration: Combines by position  
    - Inputs: Merge Video ID With Video Data, Parse Transcript from API Response  
    - Outputs: Items enriched with transcript text

---

#### 2.3 Block: Save Transcripts

- **Overview:**  
  This block saves the enriched video transcript data into a Supabase database table and inserts a wait node to throttle processing and avoid hitting rate limits or API restrictions.

- **Nodes Involved:**  
  - Save Data to Supabase (Supabase node)  
  - Wait (Wait node)  

- **Node Details:**  

  - **Save Data to Supabase**  
    - Type: Supabase  
    - Role: Inserts video transcript and metadata into a Supabase table named `content_queue_1`  
    - Configuration:  
      - Table: `content_queue_1`  
      - Fields mapped:  
        - `content_type` = "youtube" (static)  
        - `title` = video title  
        - `source_url` = original video link  
        - `content_snippet` = transcript text  
        - `published_date` = video publish date  
        - `creator` = video author  
      - On error: continue regular output to avoid full workflow failure  
    - Credentials: Supabase API credentials preconfigured  
    - Retry on Fail: Enabled  
    - Edge Cases: Insert errors, connection issues, malformed data

  - **Wait**  
    - Type: Wait  
    - Role: Introduces a delay of 30 seconds between processing items to manage API rate limits  
    - Configuration: Wait amount set to 30 seconds  
    - Inputs: Save Data to Supabase output  
    - Outputs: Loops back to Loop Over New Videos for continued processing  
    - Edge Cases: None significant; ensures pacing of workflow runs  
    - Sticky Note: Mentions “Wait for a sec” to manage pacing

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                                  | Input Node(s)                        | Output Node(s)                             | Sticky Note                                                                                                  |
|-------------------------------|-----------------------|-------------------------------------------------|------------------------------------|--------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Entry point for manual start                      | -                                  | Channels To Track                          |                                                                                                              |
| Channels To Track              | Set                   | Defines YouTube channel IDs to track             | When clicking ‘Execute workflow’    | Split Out                                 | Stores the Channel IDs of the youtube channels you are tracking. Find channel IDs at https://www.tunepocket.com/youtube-channel-id-finder |
| Split Out                     | Split Out             | Splits array of channel IDs into individual items | Channels To Track                   | Channel Info                              | Splits all of the websites into their own items so that they go into the loop one at a time                   |
| Channel Info                  | Set                   | Assigns individual channel ID field               | Split Out                         | Verify Channel ID + Create RSS Link       |                                                                                                              |
| Verify Channel ID + Create RSS Link | Code (JavaScript)    | Validates channel ID and creates RSS URL          | Channel Info                      | Channel Info + Channel ID                  | Convert to a valid RSS feed link                                                                              |
| Channel Info + Channel ID      | Set                   | Passes RSS feed URL downstream                     | Verify Channel ID + Create RSS Link | Loop Over Each Channel                     |                                                                                                              |
| Loop Over Each Channel         | Split In Batches      | Processes each channel in batches                  | Channel Info + Channel ID           | Find Channel's Videos / Loop Over New Videos |                                                                                                              |
| Find Channel's Videos          | RSS Feed Read         | Fetches videos for a channel from RSS feed        | Loop Over Each Channel             | Loop Over Each Channel                     |                                                                                                              |
| Loop Over New Videos           | Split In Batches      | Processes individual videos in batches            | Loop Over Each Channel / Wait       | Filter Out YouTube Shorts                  |                                                                                                              |
| Filter Out YouTube Shorts      | If                    | Filters out YouTube Shorts videos                  | Loop Over New Videos               | Rename Original URL / Loop Over New Videos | Filter excludes Shorts by default; delete second filter condition to include shorts                          |
| Rename Original URL            | Set                   | Renames `link` to `original_url` for consistency  | Filter Out YouTube Shorts (true)  | Find Video ID                             |                                                                                                              |
| Find Video ID                 | Set                   | Extracts YouTube video ID from URL                 | Rename Original URL               | Is Video ID valid?                        |                                                                                                              |
| Is Video ID valid?             | Set                   | Validates extracted video ID                        | Find Video ID                    | Clean Up URL                             |                                                                                                              |
| Clean Up URL                  | Set                   | Builds clean YouTube watch URL from video ID       | Is Video ID valid?                | Was Video ID Found?                      |                                                                                                              |
| Was Video ID Found?            | If                    | Checks if video ID extraction succeeded            | Clean Up URL                    | Merge Video ID With Video Data / Rename URL |                                                                                                              |
| Rename URL                   | Set                   | Renames `original_url` to `url` for retry          | Was Video ID Found? (false branch) | try to get video_id again                  |                                                                                                              |
| try to get video_id again      | Code (JavaScript)      | Alternative robust extraction of video ID          | Rename URL                      | Merge Video ID With Video Data            |                                                                                                              |
| Merge Video ID With Video Data | Merge                 | Combines video metadata and ID data                 | Was Video ID Found? (true), try to get video_id again | Get Transcript From API / Add Transcript to Video Data |                                                                                                              |
| Get Transcript From API        | HTTP Request          | Calls youtube-transcript.io API to get transcript  | Merge Video ID With Video Data    | Transcript Worked?                       | API key required: https://www.youtube-transcript.io/. Only works on videos with captions                     |
| Transcript Worked?             | If                    | Checks if transcript retrieval was successful      | Get Transcript From API          | Parse Transcript from API Response / Transcript Failed |                                                                                                              |
| Parse Transcript from API Response | Code (JavaScript)      | Parses and concatenates transcript text             | Transcript Worked? (true)          | Add Transcript to Video Data              |                                                                                                              |
| Transcript Failed             | Stop and Error        | Stops workflow with error message on failure        | Transcript Worked? (false)         | -                                        |                                                                                                              |
| Add Transcript to Video Data   | Merge                 | Combines transcript with video data                  | Merge Video ID With Video Data, Parse Transcript from API Response | Save Data to Supabase                       |                                                                                                              |
| Save Data to Supabase          | Supabase              | Saves video metadata and transcript into database   | Add Transcript to Video Data      | Wait                                     |                                                                                                              |
| Wait                         | Wait                  | Delays workflow to manage API limits                 | Save Data to Supabase            | Loop Over New Videos                      | Wait for a sec                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger (default settings)  

2. **Add Set Node for Channels To Track:**  
   - Name: `Channels To Track`  
   - Type: Set  
   - Add field `source_identifier` (type: array)  
   - Set value to array of YouTube Channel IDs to track, e.g., `["UCaEkuhQejDMyindRnUbISIg", "UCIPPMRA040LQr5QPyJEbmXA"]`  

3. **Add Split Out Node:**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Field to split: `source_identifier`  
   - Connect `Channels To Track` output to this node  

4. **Add Set Node for Channel Info:**  
   - Name: `Channel Info`  
   - Type: Set  
   - Assign field `youtubeChannels` to current item value (`={{ $json.source_identifier }}`)  

5. **Add Code Node to Verify Channel ID and Create RSS Link:**  
   - Name: `Verify Channel ID + Create RSS Link`  
   - Type: Code (JavaScript)  
   - Code: Validate if `youtubeChannels` starts with "UC" and length is 24; if valid, create field `rssUrl` as `https://www.youtube.com/feeds/videos.xml?channel_id=CHANNEL_ID`  
   - If invalid, return null to skip processing  
   - Enable "Retry on Fail" and "Always Output Data"  

6. **Add Set Node to Pass RSS URL:**  
   - Name: `Channel Info + Channel ID`  
   - Type: Set  
   - Assign `rssUrl` from previous node output (`={{ $json.rssUrl }}`)  

7. **Add Split In Batches Node to Loop Over Each Channel:**  
   - Name: `Loop Over Each Channel`  
   - Type: Split In Batches  
   - Default batch size (e.g., 1)  

8. **Add RSS Feed Read Node to Find Channel's Videos:**  
   - Name: `Find Channel's Videos`  
   - Type: RSS Feed Read  
   - URL: dynamic `={{ $json.rssUrl }}`  
   - Enable retry on fail  

9. **Add Split In Batches Node to Loop Over New Videos:**  
   - Name: `Loop Over New Videos`  
   - Type: Split In Batches  

10. **Add If Node to Filter Out YouTube Shorts:**  
    - Name: `Filter Out YouTube Shorts`  
    - Type: If  
    - Conditions:  
      - Check if `link` field exists  
      - Check `link` does NOT contain `"youtube.com/shorts"` (to exclude shorts)  
    - True branch continues processing; False branch loops back to `Loop Over New Videos`  

11. **Add Set Node to Rename Original URL:**  
    - Name: `Rename Original URL`  
    - Type: Set  
    - Assign `original_url` = `{{ $json.link }}`  
    - Include other fields  

12. **Add Set Node to Extract Video ID:**  
    - Name: `Find Video ID`  
    - Type: Set  
    - Use regex to extract video ID from `original_url` and assign to `video_id`:  
      ```js
      {{$json.original_url.match(/(?:youtube\.com\/(?:[^\/\n\s]+\/\S+\/|(?:v|e(?:mbed)?|shorts)\/|.*[?&]v=)|youtu\.be\/)([a-zA-Z0-9_-]{11})/i)?.[1] || null}}
      ```  
    - Include other fields  

13. **Add Set Node to Validate Video ID:**  
    - Name: `Is Video ID valid?`  
    - Type: Set  
    - Assign boolean `is_valid` to true if video ID extracted successfully, false otherwise  

14. **Add Set Node to Clean Up URL:**  
    - Name: `Clean Up URL`  
    - Type: Set  
    - Assign `clean_url` = `https://www.youtube.com/watch?v={{$json.video_id}}` if valid, else null  

15. **Add If Node for Video ID Found Check:**  
    - Name: `Was Video ID Found?`  
    - Type: If  
    - Condition: `is_valid` equals true  
    - True branch connects forward  
    - False branch connects to a retry extraction path  

16. **Add Set Node to Rename URL for Retry:**  
    - Name: `Rename URL`  
    - Type: Set  
    - Assign `url` = `{{ $json.original_url }}`  

17. **Add Code Node for Retry Video ID Extraction:**  
    - Name: `try to get video_id again`  
    - Type: Code (JavaScript)  
    - Code: Robust regex extraction from multiple possible URL fields; returns video_id, is_valid, clean_url  
    - Enable retry on fail  

18. **Add Merge Node to Combine Video ID With Video Data:**  
    - Name: `Merge Video ID With Video Data`  
    - Mode: Combine by position to join outputs from both video ID extraction branches  

19. **Add HTTP Request Node to Call youtube-transcript.io API:**  
    - Name: `Get Transcript From API`  
    - HTTP Method: POST  
    - URL: `https://www.youtube-transcript.io/api/transcripts`  
    - Headers: `Content-Type: application/json` plus API key via HTTP Header Auth credential  
    - Body: JSON with array `ids` containing `video_id`  
    - Authentication: HTTP Header Auth with API key credential created for youtube-transcript.io  
    - Retry on fail with 5 seconds wait  

20. **Add If Node to Check Transcript Success:**  
    - Name: `Transcript Worked?`  
    - Condition: `statusMessage` equals `"OK"`  
    - True branch proceeds to parse transcript  
    - False branch stops workflow with error  

21. **Add Code Node to Parse Transcript from API Response:**  
    - Name: `Parse Transcript from API Response`  
    - Code: Parse JSON `data` field, extract transcript text by concatenating track texts  

22. **Add Stop and Error Node for Transcript Failure:**  
    - Name: `Transcript Failed`  
    - Error message: “Transcript Failed”  

23. **Add Merge Node to Add Transcript to Video Data:**  
    - Name: `Add Transcript to Video Data`  
    - Mode: Combine by position to merge parsed transcript with video metadata  

24. **Add Supabase Node to Save Data:**  
    - Name: `Save Data to Supabase`  
    - Table name: `content_queue_1`  
    - Map fields:  
      - `content_type` = "youtube" (static)  
      - `title` = video title  
      - `source_url` = original video link  
      - `content_snippet` = transcript text  
      - `published_date` = publish date  
      - `creator` = video author  
    - Credentials: Set up Supabase API credentials  
    - Error handling: continue regular output on error  
    - Retry on fail enabled  

25. **Add Wait Node for Rate Limiting:**  
    - Name: `Wait`  
    - Wait time: 30 seconds  
    - Connect output back to `Loop Over New Videos` to pace execution  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Stores the Channel IDs of the youtube channels you are tracking. Find channel IDs for free by using a website such as https://www.tunepocket.com/youtube-channel-id-finder | Sticky Note on `Channels To Track` node                                                                |
| Filter excludes Shorts by default; delete second filter condition to include Shorts                                                                            | Sticky Note on `Filter Out YouTube Shorts` node                                                       |
| API key required: https://www.youtube-transcript.io/. Only works on videos with captions                                                                       | Sticky Note on `Get Transcript From API` node                                                         |
| Replace manual trigger with a Schedule node to automate workflow runs                                                                                          | Sticky Note near `When clicking ‘Execute workflow’` node                                              |
| Wait node to pace workflow execution and avoid hitting API rate limits                                                                                         | Sticky Note near `Wait` node                                                                           |
| Quick Setup Guide: Add Channel IDs, set API key, connect database, optionally replace manual trigger with schedule to automate.                               | Sticky Note at workflow start explaining usage and customization                                      |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---