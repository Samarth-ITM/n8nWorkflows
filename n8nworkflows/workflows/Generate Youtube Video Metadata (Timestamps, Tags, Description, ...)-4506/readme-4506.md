Generate Youtube Video Metadata (Timestamps, Tags, Description, ...)

https://n8nworkflows.xyz/workflows/generate-youtube-video-metadata--timestamps--tags--description-------4506


# Generate Youtube Video Metadata (Timestamps, Tags, Description, ...)

### 1. Workflow Overview

This workflow automates the generation of YouTube video metadata (timestamps, tags, and description) for newly posted videos on a specified YouTube channel. It is designed to detect new video uploads, scrape video data and subtitles via Apify, wait for the scraping process to complete, and then generate structured metadata using a Large Language Model (LLM). Finally, it formats the metadata and updates the corresponding YouTube video with the enriched description and tags.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception**: Detects new videos on a YouTube channel using an RSS feed trigger.
- **1.2 Video Scraping Setup**: Initiates scraping of video details and subtitles via the Apify YouTube Scraper.
- **1.3 Scraping Completion Handling**: Polls Apify to check and wait until the scraping job finishes, then fetches the dataset.
- **1.4 Metadata Generation**: Checks if metadata is already generated; if not, generates structured metadata (preview, timestamps, tags) using an LLM (Mistral Cloud model).
- **1.5 Metadata Formatting and Update**: Formats the generated metadata into a YouTube-friendly description and updates the YouTube video accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview**: Listens for new video uploads on a specified YouTube channel using the channel’s RSS feed, polling every 10 minutes.
- **Nodes Involved**:
  - Trigger New Video Posted
  - If Recent
  - No Operation, do nothing
- **Node Details**:
  - **Trigger New Video Posted**
    - Type: RSS Feed Read Trigger
    - Role: Entry point; triggers workflow on new video detection.
    - Configuration: Uses YouTube channel's RSS feed URL with the channel ID parameter; polling interval set to every 10 minutes.
    - Input/Output: No input; outputs feed items with video metadata.
    - Edge Cases: Network failures, invalid channel ID, feed format changes.
  - **If Recent**
    - Type: If node
    - Role: Filters videos published within the last 10 minutes to avoid processing old videos.
    - Configuration: Compares video publication timestamp against current time minus 10 minutes.
    - Input: Output from trigger node.
    - Output: If true, proceeds to scrape; else, no operation.
    - Edge Cases: Time zone discrepancies, malformed date strings.
  - **No Operation, do nothing**
    - Type: No-Op
    - Role: Terminates the path for videos not recent enough.
    - Configuration: Default (does nothing).
    - Input: False branch of If Recent.
    - Output: None.

#### 2.2 Video Scraping Setup

- **Overview**: Starts a scraping job on Apify to retrieve video details and subtitles for the detected video.
- **Nodes Involved**:
  - Scrape Video
  - Check IF Finished
  - If Finished
- **Node Details**:
  - **Scrape Video**
    - Type: HTTP Request (POST)
    - Role: Initiates Apify YouTube Scraper run with parameters:
      - Downloads subtitles in English.
      - Prefers manual over auto-generated subtitles.
      - Targets the detected video URL from the trigger.
    - Configuration: Sends JSON body with start URL and subtitle preferences.
    - Input: From If Recent true branch.
    - Output: Run ID and status from Apify.
    - Edge Cases: API token invalid/missing, request timeouts, malformed video URLs.
  - **Check IF Finished**
    - Type: HTTP Request (GET)
    - Role: Checks status of the last Apify run to determine if scraping is complete.
    - Configuration: Calls Apify API endpoint for the last run status.
    - Input: Output of Scrape Video.
    - Output: JSON with run status.
    - Edge Cases: API rate limits, network failures.
  - **If Finished**
    - Type: If node
    - Role: Branches depending on whether Apify run status equals "SUCCEEDED".
    - Configuration: Checks JSON field `data.status` equals "SUCCEEDED".
    - Input: Output from Check IF Finished.
    - Output: If true, proceeds to get dataset; if false, waits.
    - Edge Cases: Unexpected statuses, null responses.

#### 2.3 Scraping Completion Handling

- **Overview**: Waits if the scraping is not finished and retrieves the scraped video dataset once ready.
- **Nodes Involved**:
  - Wait
  - Get DataSet
  - If Not Generated
- **Node Details**:
  - **Wait**
    - Type: Wait
    - Role: Pauses execution before rechecking scraping completion.
    - Configuration: Default wait time (not explicitly set in workflow).
    - Input: False branch of If Finished.
    - Output: Loops back to Check IF Finished.
    - Edge Cases: Indefinite waiting if scraping fails.
  - **Get DataSet**
    - Type: HTTP Request (GET)
    - Role: Retrieves the dataset of scraped video information from Apify.
    - Configuration: Calls Apify dataset items endpoint for the last run.
    - Input: True branch of If Finished.
    - Output: JSON array with video metadata including subtitles.
    - Edge Cases: Empty dataset, API errors.
  - **If Not Generated**
    - Type: If node
    - Role: Checks if metadata (timestamps) has already been generated for the video.
    - Configuration: Checks if the text field from dataset does not contain "00:00" (common in timestamps).
    - Input: Output of Get DataSet.
    - Output: If true, triggers metadata generation; else no operation.
    - Edge Cases: False negatives due to text format variations.

#### 2.4 Metadata Generation

- **Overview**: Uses an LLM (Mistral Cloud) with LangChain integration to generate structured video metadata (preview, timestamps, tags) from the video subtitles.
- **Nodes Involved**:
  - Mistral Cloud Chat Model
  - Structured Output Parser
  - Generate Description
  - No Operation, do nothing1
- **Node Details**:
  - **Mistral Cloud Chat Model**
    - Type: LangChain LLM Chat Node
    - Role: Runs the Mistral large language model to process video subtitles and generate metadata.
    - Configuration: Model set to "mistral-large-latest"; credentials for Mistral Cloud API required.
    - Input: Subtitles from Get DataSet node.
    - Output: Raw LLM-generated text.
    - Edge Cases: API rate limits, parsing errors, model downtime.
  - **Structured Output Parser**
    - Type: LangChain Structured Output Parser
    - Role: Parses the LLM output into a JSON structure with fields: preview, timestamps, tags.
    - Configuration: JSON schema example defined for expected output structure.
    - Input: Output from Mistral Cloud Chat Model.
    - Output: Parsed JSON metadata.
    - Edge Cases: Malformed LLM output, schema mismatch.
  - **Generate Description**
    - Type: LangChain Chain LLM Node
    - Role: Defines the prompt to instruct the LLM to extract metadata in a specific format (preview, timestamps, tags).
    - Configuration: Includes detailed prompt instructions for preview length, timestamp formatting, and tag requirements.
    - Input: Parsed LLM output.
    - Output: Structured metadata.
    - Edge Cases: Prompt misinterpretation, incomplete responses.
  - **No Operation, do nothing1**
    - Type: No-Op
    - Role: Handles branch where metadata is already generated.
    - Input: False branch of If Not Generated.
    - Output: None.

#### 2.5 Metadata Formatting and Update

- **Overview**: Formats the generated metadata into a comprehensive YouTube description, including hardcoded links, and updates the video via the YouTube API.
- **Nodes Involved**:
  - Format 
  - Update YTB Video
- **Node Details**:
  - **Format**
    - Type: Code (JavaScript)
    - Role: Combines preview description, hardcoded external links, and timestamps into a formatted description string.
    - Configuration: Extracts preview and timestamps from LLM output; appends predefined links section.
    - Input: Output from Generate Description.
    - Output: JSON containing the final formatted description.
    - Edge Cases: Missing fields in input JSON, script errors.
  - **Update YTB Video**
    - Type: YouTube Node
    - Role: Updates the YouTube video’s description and tags with the newly generated metadata.
    - Configuration:
      - Title and videoId taken dynamically from Get DataSet.
      - Tags set from generated tags list.
      - Description set from formatted output.
      - Requires YouTube OAuth2 credentials.
    - Input: Output from Format node.
    - Output: API response from YouTube.
    - Edge Cases: OAuth token expiry, API quota limits, invalid video ID.

---

### 3. Summary Table

| Node Name                | Node Type                              | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                               |
|--------------------------|--------------------------------------|----------------------------------------|----------------------------|----------------------------|----------------------------------------------------------------------------------------------------------|
| Trigger New Video Posted  | RSS Feed Read Trigger                 | Detect new video uploads                | -                          | If Recent                  | ## 1- Input Enter the ID of the YTB channel to trigger the workflow when a new video is posted           |
| If Recent                | If                                   | Filter videos published in last 10 min | Trigger New Video Posted    | Scrape Video, No Operation | ## 1- Input Enter the ID of the YTB channel to trigger the workflow when a new video is posted           |
| No Operation, do nothing  | No-Op                                | End branch for old videos               | If Recent                  | -                          | ## 1- Input Enter the ID of the YTB channel to trigger the workflow when a new video is posted           |
| Scrape Video             | HTTP Request (POST)                   | Start Apify scraping job                | If Recent                  | Check IF Finished           | ## 2- Create DataSet Apify scrape the last YTB video of the channel                                      |
| Check IF Finished        | HTTP Request (GET)                    | Poll scraping job status                | Scrape Video               | If Finished                | ## 3- Wait for Completion & Get DataSet Wait until the dataset is completed in Apify and get it          |
| If Finished              | If                                   | Branch on scraping job completion       | Check IF Finished          | Get DataSet, Wait          | ## 3- Wait for Completion & Get DataSet Wait until the dataset is completed in Apify and get it          |
| Wait                     | Wait                                 | Pause before rechecking scraping status | If Finished                | Check IF Finished           | ## 3- Wait for Completion & Get DataSet Wait until the dataset is completed in Apify and get it          |
| Get DataSet              | HTTP Request (GET)                    | Retrieve scraped video dataset          | If Finished                | If Not Generated           | ## 3- Wait for Completion & Get DataSet Wait until the dataset is completed in Apify and get it          |
| If Not Generated         | If                                   | Check if metadata already generated     | Get DataSet                | Generate Description, No Operation1 | ## 4- Check and Generate Metadata Verify if Metadata are not already generated and generate them with LLM |
| Generate Description     | LangChain Chain LLM                   | Generate structured metadata            | If Not Generated           | Format                     | ## 4- Check and Generate Metadata Verify if Metadata are not already generated and generate them with LLM |
| No Operation, do nothing1 | No-Op                                | End branch if metadata already present | If Not Generated           | -                          | ## 4- Check and Generate Metadata Verify if Metadata are not already generated and generate them with LLM |
| Format                   | Code                                 | Format description and links            | Generate Description       | Update YTB Video            | ## 5- Output Format all the data created and update YTB Video                                            |
| Update YTB Video         | YouTube                              | Update video description and tags       | Format                     | -                          | ## 5- Output Format all the data created and update YTB Video                                            |
| Mistral Cloud Chat Model | LangChain LLM Chat                   | Process subtitles with LLM              | Generate Description       | Generate Description       | ## 4- Check and Generate Metadata Verify if Metadata are not already generated and generate them with LLM |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add **RSS Feed Read Trigger** node named *Trigger New Video Posted*.
   - Set feed URL to `https://www.youtube.com/feeds/videos.xml?channel_id=[YOUR_CHANNEL_ID]`.
   - Set polling interval to every 10 minutes.

2. **Add If Node to Filter Recent Videos:**
   - Add **If** node named *If Recent*.
   - Condition: Check if video `pubDate` timestamp is greater than current time minus 10 minutes.
   - Connect *Trigger New Video Posted* → *If Recent*.

3. **Add No-Op Node for Old Videos:**
   - Add **No Operation** node named *No Operation, do nothing*.
   - Connect false branch of *If Recent* → *No Operation, do nothing*.

4. **Add HTTP Request Node to Start Scraping:**
   - Add **HTTP Request (POST)** node named *Scrape Video*.
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/runs?token=[YOUR_API_TOKEN]`.
   - Method: POST
   - Headers: Content-Type application/json.
   - Body (JSON): 
     ```json
     {
       "downloadSubtitles": true,
       "preferAutoGeneratedSubtitles": false,
       "startUrls": [{"url": "={{ $json.link }}", "method": "GET"}],
       "subtitlesLanguage": "en"
     }
     ```
   - Connect true branch of *If Recent* → *Scrape Video*.

5. **Add HTTP Request Node to Check Scraping Status:**
   - Add **HTTP Request (GET)** node named *Check IF Finished*.
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/runs/last?token=[YOUR_API_TOKEN]`.
   - Connect *Scrape Video* → *Check IF Finished*.

6. **Add If Node to Branch on Scraping Completion:**
   - Add **If** node named *If Finished*.
   - Condition: Check if `data.status` equals "SUCCEEDED".
   - Connect *Check IF Finished* → *If Finished*.

7. **Add Wait Node to Pause if Not Finished:**
   - Add **Wait** node named *Wait* with default wait time.
   - Connect false branch of *If Finished* → *Wait*.
   - Connect *Wait* → back to *Check IF Finished* for polling.

8. **Add HTTP Request to Get Scraped Dataset:**
   - Add **HTTP Request (GET)** node named *Get DataSet*.
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/runs/last/dataset/items?token=[YOUR_API_TOKEN]`.
   - Connect true branch of *If Finished* → *Get DataSet*.

9. **Add If Node to Check if Metadata Exists:**
   - Add **If** node named *If Not Generated*.
   - Condition: Check if `text` field does not contain "00:00".
   - Connect *Get DataSet* → *If Not Generated*.

10. **Add No-Op Node for Already Generated Metadata:**
    - Add **No Operation** node named *No Operation, do nothing1*.
    - Connect false branch of *If Not Generated* → *No Operation, do nothing1*.

11. **Add LangChain Nodes for Metadata Generation:**
    - Add **LangChain LLM Chat Node** named *Mistral Cloud Chat Model*.
      - Set model to "mistral-large-latest".
      - Setup Mistral Cloud API credentials.
      - Connect *If Not Generated* true branch → *Mistral Cloud Chat Model*.
    - Add **LangChain Structured Output Parser** node named *Structured Output Parser*.
      - Define JSON schema for preview, timestamps, and tags.
      - Connect *Mistral Cloud Chat Model* → *Structured Output Parser*.
    - Add **LangChain Chain LLM** node named *Generate Description*.
      - Define prompt instructing LLM to format preview, timestamps, and tags strictly.
      - Enable output parser.
      - Connect *Structured Output Parser* → *Generate Description*.

12. **Add Code Node to Format Final Description:**
    - Add **Code** node named *Format*.
    - JavaScript code:
      ```js
      const preview = $input.first().json.output[0].description;
      const timestamps = $input.first().json.output[1].description;
      const tags = $input.first().json.output[2].description;

      const links = `
      Links:
      - Website: https://example.com
      - Twitter: https://twitter.com/example
      - GitHub: https://github.com/example
      `;

      const description = preview + "\n" + links + "\n" + timestamps;

      return [{
        json: {
          description: description.trim(),
          tags: tags.trim(),
        },
      }];
      ```
    - Connect *Generate Description* → *Format*.

13. **Add YouTube Node to Update Video:**
    - Add **YouTube** node named *Update YTB Video*.
    - Operation: Update video.
    - Set:
      - `title`: `={{ $('Get DataSet').item.json.title }}`
      - `videoId`: `={{ $('Get DataSet').item.json.id }}`
      - `description`: `={{ $json.description }}`
      - `tags`: `={{ $json.tags }}`
    - Connect *Format* → *Update YTB Video*.
    - Configure YouTube OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| How it works? 1- Enter the ID of the YTB channel to trigger the workflow when a new video is posted 2- Apify scrapes the last YTB video of the channel 3- Wait until dataset completion 4- Generate metadata with LLM 5- Format and update YTB Video | Workflow overview sticky note                                                                                       |
| 📺 Youtube Video Tutorial : https://youtu.be/HaQPAa6l5bU                                                                                                                                                                         | Workflow tutorial video link                                                                                        |
| 🛠️ Need Help with Your Workflows ? https://tally.so/r/wayeqB                                                                                                                                                                    | Support and help resource                                                                                           |
| 👨‍💻 More Workflows : https://n8n.io/creators/nasser/                                                                                                                                                                           | Additional workflows from the creator                                                                              |
| Setup Input YTB Channel : Retrieve the channel ID from the YouTube channel URL after "/channel/". Use free tools if needed.                                                                                                    | Setup instructions                                                                                                |
| Setup Output YTB Video Update : Connect your YouTube account via Google Cloud Console with OAuth2. Search for "youtube api OAuth" tutorials for setup guidance.                                                                | Setup instructions                                                                                                |
| APIs : Replace `[YOUR_API_TOKEN]` with your Apify API token or connect using client credentials. Apify API docs: https://docs.apify.com/api/v2/getting-started YouTube node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.youtube/ | API and credential setup references                                                                                 |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated workflow created with n8n, respecting all applicable content policies and handling only legal and public data.