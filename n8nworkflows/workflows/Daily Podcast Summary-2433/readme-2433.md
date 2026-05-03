Daily Podcast Summary

https://n8nworkflows.xyz/workflows/daily-podcast-summary-2433


# Daily Podcast Summary

### 1. Workflow Overview

This workflow, titled **Daily Podcast Summary**, automates the process of retrieving the daily top podcasts from a specified genre, transcribing and summarizing their content, and emailing the results in a formatted summary. It targets users interested in receiving daily podcast highlights without manually searching or listening to episodes, useful for content curators, podcast enthusiasts, or media analysts.

The workflow logic is organized into these functional blocks:

- **1.1 Scheduled Trigger & Genre Setup**: Defines the schedule for daily execution and sets the podcast genre for which top charts are retrieved.
- **1.2 Fetch Top Podcasts from Taddy API**: Queries the Taddy API to obtain the top podcast episodes' metadata for the chosen genre.
- **1.3 Podcast Audio Download and Cropping**: Downloads each podcast audio file and crops a specific segment for transcription.
- **1.4 Audio Transcription and Summarization**: Uses OpenAI's Whisper API to transcribe the audio segment, then summarizes the transcription using GPT.
- **1.5 Data Aggregation and HTML Formatting**: Collects all summaries, formats them into an HTML email body.
- **1.6 Email Dispatch**: Sends the composed email with podcast summaries via Gmail.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger & Genre Setup

- **Overview:** This block triggers the workflow daily at a specified hour and sets the podcast genre filter for the API query.
- **Nodes Involved:**  
  - `Schedule`  
  - `Genre`

- **Node Details:**

1. **Schedule**  
   - Type: Schedule Trigger  
   - Role: Initiates workflow execution once daily at 8:00 AM (configurable).  
   - Key configuration: Trigger at hour 8.  
   - Inputs: None (trigger node).  
   - Outputs: Connects to `Genre`.  
   - Potential failure: Misconfiguration in schedule may cause the workflow not to trigger.

2. **Genre**  
   - Type: Set  
   - Role: Defines the genre of podcasts to fetch (e.g., TECHNOLOGY).  
   - Configuration: Sets a string variable `genre` = "TECHNOLOGY" (modifiable).  
   - Inputs: From `Schedule`.  
   - Outputs: Connects to `TaddyTopDaily`.  
   - Edge cases: Invalid genre values may cause empty or failed API responses.

---

#### 2.2 Fetch Top Podcasts from Taddy API

- **Overview:** Queries the Taddy API to retrieve top podcast episodes metadata filtered by genre and country.
- **Nodes Involved:**  
  - `TaddyTopDaily`  
  - `Split Out`

- **Node Details:**

1. **TaddyTopDaily**  
   - Type: HTTP Request  
   - Role: Posts a GraphQL query to Taddy API to fetch top 10 podcast episodes for the selected genre and country (USA).  
   - Configuration:  
     - HTTP POST to `https://api.taddy.org/`  
     - GraphQL query filtered by genre (`PODCASTSERIES_{{ $json.genre }}`) and country.  
     - Headers require `X-USER-ID` and `X-API-KEY` (set by user credentials).  
   - Inputs: From `Genre`.  
   - Outputs: Connects to `Split Out`.  
   - Edge cases:  
     - API key or user ID missing or invalid → auth errors.  
     - Network or timeout errors.  
     - Invalid genre filter → empty results.

2. **Split Out**  
   - Type: Split Out  
   - Role: Splits the array of podcast episodes into individual items for parallel processing.  
   - Configuration: Splits on `data.getTopChartsByGenres.podcastEpisodes` field.  
   - Inputs: From `TaddyTopDaily`.  
   - Outputs: Connects to `Download Podcast`.  
   - Potential failure: If the field is missing or empty, downstream nodes receive no data.

---

#### 2.3 Podcast Audio Download and Cropping

- **Overview:** Downloads each podcast audio file, sends it to a cropping service to extract an 8-24 minute segment, then waits for cropping to complete.
- **Nodes Involved:**  
  - `Download Podcast`  
  - `Request Audio Crop`  
  - `Get Download Link`  
  - `If Downloads Ready`  
  - `Wait`  
  - `Download Cut MP3`

- **Node Details:**

1. **Download Podcast**  
   - Type: HTTP Request  
   - Role: Downloads the full podcast audio file from the episode's `audioUrl`.  
   - Configuration: URL taken dynamically from current item JSON `audioUrl`.  
   - Inputs: From `Split Out`.  
   - Outputs: Connects to `Request Audio Crop`.  
   - Edge cases:  
     - Invalid or missing URL → download failure.  
     - Large files may cause timeout or memory issues.

2. **Request Audio Crop**  
   - Type: HTTP Request  
   - Role: Sends the downloaded audio to Aspose audio cutter API to crop from 08:00 to 24:00 minutes, output in mp3 format.  
   - Configuration:  
     - POST multipart/form-data with binary audio data.  
     - Body parameter `convertOption` specifies startTime/endTime/audioFormat.  
     - Required headers for CORS and origin set.  
   - Inputs: From `Download Podcast`.  
   - Outputs: Connects to `Get Download Link`.  
   - Edge cases:  
     - API errors, rate limits, or invalid audio format.  
     - Network timeouts.

3. **Get Download Link**  
   - Type: HTTP Request  
   - Role: Polls Aspose API for the status of the cropped audio file and retrieves the download URL.  
   - Configuration: URL dynamically built using `fileRequestId` from previous node output.  
   - Inputs: From `Request Audio Crop` and from `Wait` (loop).  
   - Outputs: Connects to `If Downloads Ready`.  
   - Edge cases:  
     - Polling too fast or too slow may affect performance.  
     - API errors or missing download link.

4. **If Downloads Ready**  
   - Type: If  
   - Role: Checks if all cropped audio download links are ready and valid (all not empty).  
   - Configuration: Boolean condition evaluating all items' `Data.DownloadLink` fields.  
   - Inputs: From `Get Download Link`.  
   - Outputs:  
     - True path connects to `Download Cut MP3` (proceed).  
     - False path connects to `Wait` (retry delay).  
   - Edge cases:  
     - Partial readiness may cause indefinite waiting.  
     - Missing or malformed fields.

5. **Wait**  
   - Type: Wait  
   - Role: Pauses execution (default delay) before retrying download link check.  
   - Inputs: From `If Downloads Ready` false path.  
   - Outputs: Connects back to `Get Download Link` (polling loop).  
   - Edge cases: Infinite loop if download never completes.

6. **Download Cut MP3**  
   - Type: HTTP Request  
   - Role: Downloads the cropped mp3 segment using the obtained direct download URL.  
   - Configuration: URL dynamically set from `Data.DownloadLink`.  
   - Inputs: From `If Downloads Ready` true path.  
   - Outputs: Connects to `Whisper Transcribe Audio`.  
   - Edge cases: Download errors, invalid or expired URLs.

---

#### 2.4 Audio Transcription and Summarization

- **Overview:** Transcribes the cropped audio segment via OpenAI Whisper API, then summarizes the transcription with GPT-4o-mini.
- **Nodes Involved:**  
  - `Whisper Transcribe Audio`  
  - `Summarize Podcast`  
  - `Final Data`

- **Node Details:**

1. **Whisper Transcribe Audio**  
   - Type: HTTP Request  
   - Role: Sends audio file to OpenAI Whisper API for transcription.  
   - Configuration:  
     - POST to `https://api.openai.com/v1/audio/transcriptions`  
     - Model set to `whisper-1`.  
     - Authentication via OpenAI API credentials.  
     - Multipart form-data with binary audio data.  
   - Inputs: From `Download Cut MP3`.  
   - Outputs: Connects to `Summarize Podcast`.  
   - Edge cases:  
     - API rate limits, auth errors.  
     - Audio format incompatibility.  
     - Transcription failure or empty result.

2. **Summarize Podcast**  
   - Type: OpenAI (Chat Completion)  
   - Role: Summarizes the transcribed text into 3-4 paragraphs focusing on key points.  
   - Configuration:  
     - Model: `gpt-4o-mini` (chat resource).  
     - Prompt: Instruction to summarize the transcription starting with "This episode focuses on...".  
     - Max tokens: 500.  
     - Inputs: Uses transcribed text from previous node as variable `$json.text`.  
   - Inputs: From `Whisper Transcribe Audio`.  
   - Outputs: Connects to `Final Data`.  
   - Edge cases:  
     - API quota exceeded.  
     - Prompt formatting errors.  
     - Empty or irrelevant summary output.

3. **Final Data**  
   - Type: Set  
   - Role: Constructs a cleaned JSON object containing podcast name, episode title, direct audio URL, and summary with HTML-safe formatting.  
   - Configuration:  
     - Extracts podcast series name and episode name from original Taddy data using item index.  
     - Sanitizes summary text for HTML.  
   - Inputs: From `Summarize Podcast`.  
   - Outputs: Connects to `Merge Results`.  
   - Edge cases:  
     - Index mismatch if data ordering changes.  
     - Encoding issues in summary text.

---

#### 2.5 Data Aggregation and HTML Formatting

- **Overview:** Aggregates all podcast summaries into a single dataset and formats it into an HTML table for email delivery.
- **Nodes Involved:**  
  - `Merge Results`  
  - `HTML`

- **Node Details:**

1. **Merge Results**  
   - Type: Code  
   - Role: Combines all individual podcast summary items into one array under `fields`.  
   - Configuration: JavaScript code returns an object with `fields` property containing all items' JSON data.  
   - Inputs: From `Final Data`.  
   - Outputs: Connects to `HTML`.  
   - Edge cases:  
     - Empty input array → empty email content.

2. **HTML**  
   - Type: HTML  
   - Role: Generates an HTML email body rendering a table of podcasts, episodes, and summaries.  
   - Configuration:  
     - HTML template with a table header and rows dynamically created from `fields`.  
     - Styles for table borders, padding, and text colors.  
     - Executes once to produce a single HTML output.  
   - Inputs: From `Merge Results`.  
   - Outputs: Connects to `Gmail`.  
   - Edge cases:  
     - Malformed HTML if input data contains unexpected characters.  
     - Large summaries may affect email formatting.

---

#### 2.6 Email Dispatch

- **Overview:** Sends the compiled HTML summary via email using Gmail credentials.
- **Nodes Involved:**  
  - `Gmail`

- **Node Details:**

1. **Gmail**  
   - Type: Gmail node  
   - Role: Sends an email with the subject "Podcast Review" and the HTML content as the message body.  
   - Configuration:  
     - Message body set dynamically from the HTML node's output (`{{ $json.html }}`).  
     - Subject fixed as "Podcast Review".  
     - Uses OAuth2 credentials for Gmail (configured via Google Cloud Console).  
   - Inputs: From `HTML`.  
   - Outputs: None (final node).  
   - Edge cases:  
     - OAuth token expiry or invalid credentials.  
     - Email delivery failures due to recipient address errors or Gmail limits.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                        | Input Node(s)                | Output Node(s)           | Sticky Note                                                                                                                             |
|-----------------------|---------------------|-------------------------------------|-----------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule              | Schedule Trigger    | Daily trigger for workflow start     | None                        | Genre                    | Change the schedule time for sending email from `Schedule` to desired time.                                                             |
| Genre                 | Set                 | Sets podcast genre filter             | Schedule                    | TaddyTopDaily             | Set genre of podcasts you want summaries for. Valid values: TECHNOLOGY, NEWS, ARTS, COMEDY, SPORTS, FICTION, etc.                       |
| TaddyTopDaily         | HTTP Request        | Fetches top podcasts metadata         | Genre                       | Split Out                 | Input your user number and API key into `X-USER-ID` and `X-API-KEY` headers in this node. Setup instructions at https://taddy.org/signup/developers |
| Split Out             | Split Out           | Splits podcast episodes for processing | TaddyTopDaily               | Download Podcast          | Get daily list of top podcasts (according to Apple charts) and download audio, then crop for OpenAI                                      |
| Download Podcast      | HTTP Request        | Downloads full podcast audio file     | Split Out                   | Request Audio Crop        | Crop the podcast down before analysis                                                                                                   |
| Request Audio Crop    | HTTP Request        | Sends audio to cropping API           | Download Podcast            | Get Download Link         | Crop the podcast down before analysis                                                                                                   |
| Get Download Link     | HTTP Request        | Polls for cropped audio download URL | Request Audio Crop, Wait    | If Downloads Ready        | Crop the podcast down before analysis                                                                                                   |
| If Downloads Ready    | If                  | Checks readiness of cropped audio     | Get Download Link           | Download Cut MP3, Wait    | Crop the podcast down before analysis                                                                                                   |
| Wait                  | Wait                | Pauses before re-polling download URL | If Downloads Ready (false)  | Get Download Link         | Crop the podcast down before analysis                                                                                                   |
| Download Cut MP3      | HTTP Request        | Downloads cropped audio segment       | If Downloads Ready (true)   | Whisper Transcribe Audio  | Whisper transcribes and Open AI summarizes the podcast                                                                                   |
| Whisper Transcribe Audio | HTTP Request      | Sends cropped audio to Whisper API    | Download Cut MP3             | Summarize Podcast         | Whisper transcribes and Open AI summarizes the podcast                                                                                   |
| Summarize Podcast     | OpenAI (Chat)       | Summarizes transcribed text           | Whisper Transcribe Audio    | Final Data                | Whisper transcribes and Open AI summarizes the podcast                                                                                   |
| Final Data            | Set                 | Constructs final JSON summary object  | Summarize Podcast           | Merge Results             |                                                                                                                                        |
| Merge Results         | Code                | Aggregates all summaries into array   | Final Data                  | HTML                     |                                                                                                                                        |
| HTML                  | HTML                | Formats podcast summaries as HTML     | Merge Results               | Gmail                    | Finally, send the email!                                                                                                                 |
| Gmail                 | Gmail               | Sends summary email via Gmail         | HTML                        | None                     | Enter your email address in this node. Setup Gmail OAuth2 credentials as per https://developers.google.com/workspace/guides/create-credentials |

Sticky Notes applying to multiple nodes:

- Nodes `Download Podcast`, `Request Audio Crop`, `Get Download Link`, `If Downloads Ready`, `Wait`, `Download Cut MP3`:  
  "### Crop the podcast down before analysis"

- Nodes `Whisper Transcribe Audio`, `Summarize Podcast`:  
  "### Whisper transcribes and Open AI summarizes the podcast"

- Nodes `TaddyTopDaily`, `Split Out`:  
  "### Get daily list of top podcasts (according to Apple charts) and download audio, then crop for OpenAI"

- Node `Gmail`:  
  "### Finally, send the email!"

- General Sticky Note attached in the workflow header area:  
  Detailed setup instructions and testing notes as in the given description, including links to Taddy API signup and Gmail credential setup.

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Schedule`  
   - Set to trigger daily at 8:00 AM (adjustable).  
   - No credentials required.

2. **Create a Set node**  
   - Name: `Genre`  
   - Add field `genre` as string.  
   - Set value to desired podcast genre (e.g., `TECHNOLOGY`).  
   - Connect input from `Schedule`.

3. **Create an HTTP Request node**  
   - Name: `TaddyTopDaily`  
   - Method: POST  
   - URL: `https://api.taddy.org/`  
   - Body: GraphQL query with variables for genre and country:  
     ```graphql
     query {
       getTopChartsByGenres(
         limitPerPage: 10,
         filterByCountry: UNITED_STATES_OF_AMERICA,
         taddyType: PODCASTEPISODE,
         genres: PODCASTSERIES_{{ $json.genre }}
       ) {
         topChartsId
         podcastEpisodes {
           uuid
           name
           audioUrl
           podcastSeries {
             uuid
             name
           }
         }
       }
     }
     ```  
   - Headers: Add `X-USER-ID` and `X-API-KEY` with your Taddy API credentials.  
   - Connect input from `Genre`.

4. **Create a Split Out node**  
   - Name: `Split Out`  
   - Set field to split to `data.getTopChartsByGenres.podcastEpisodes`.  
   - Connect input from `TaddyTopDaily`.

5. **Create an HTTP Request node**  
   - Name: `Download Podcast`  
   - Method: GET  
   - URL: Set dynamically to `{{ $json.audioUrl }}`.  
   - Connect input from `Split Out`.

6. **Create an HTTP Request node**  
   - Name: `Request Audio Crop`  
   - Method: POST  
   - URL: `https://api.products.aspose.app/audio/cutter/api/cutter`  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - Binary data from `Download Podcast` output.  
     - Field `convertOption` with JSON string: `{"startTime":"00:08:00","endTime":"00:24:00","audioFormat":"mp3"}`  
   - Headers: Set as per Aspose API requirements (Accept, Connection, Origin, Referer, Sec-Fetch headers).  
   - Connect input from `Download Podcast`.

7. **Create an HTTP Request node**  
   - Name: `Get Download Link`  
   - Method: GET  
   - URL: Dynamic: `https://api.products.aspose.app/audio/cutter/api/cutter/HandleStatus?fileRequestId={{ $('Request Audio Crop').item.json.Data.FileRequestId }}`  
   - Headers: Same as in `Request Audio Crop`.  
   - Connect input from `Request Audio Crop` and later from `Wait`.

8. **Create an If node**  
   - Name: `If Downloads Ready`  
   - Condition: Check if all items' `Data.DownloadLink` fields are truthy (non-empty).  
   - True output: Connect to `Download Cut MP3`.  
   - False output: Connect to `Wait`.

9. **Create a Wait node**  
   - Name: `Wait`  
   - Default wait time (e.g., 10 seconds or as preferred).  
   - Connect input from `If Downloads Ready` false output.  
   - Connect output back to `Get Download Link` for polling.

10. **Create an HTTP Request node**  
    - Name: `Download Cut MP3`  
    - Method: GET  
    - URL: Dynamic: `{{ $json.Data.DownloadLink }}`  
    - Connect input from `If Downloads Ready` true output.

11. **Create an HTTP Request node**  
    - Name: `Whisper Transcribe Audio`  
    - Method: POST  
    - URL: `https://api.openai.com/v1/audio/transcriptions`  
    - Authentication: Use OpenAI API credentials with OAuth or API key.  
    - Content-Type: multipart/form-data  
    - Body Parameters:  
      - `model`: `whisper-1`  
      - `file`: Binary data from `Download Cut MP3` output  
    - Connect input from `Download Cut MP3`.

12. **Create an OpenAI Chat node**  
    - Name: `Summarize Podcast`  
    - Model: `gpt-4o-mini` (or equivalent GPT-4 variant)  
    - Prompt:  
      ```
      Summarize the major points of the following podcast: {{ $json.text }}. Start your answer by saying 'This episode focuses on', 'This episode is about', etc. Contain your answer to 3-4 paragraphs max, and focus on only key information.
      ```  
    - Max tokens: 500  
    - Connect input from `Whisper Transcribe Audio`.

13. **Create a Set node**  
    - Name: `Final Data`  
    - Mode: Raw JSON  
    - Fields:  
      - `podcast`: `{{ $('TaddyTopDaily').item.json.data.getTopChartsByGenres.podcastEpisodes[$itemIndex].podcastSeries.name }}`  
      - `name`: `{{ $('TaddyTopDaily').item.json.data.getTopChartsByGenres.podcastEpisodes[$itemIndex].name.replace(/\"/g,'\"') }}`  
      - `url`: `{{ $('TaddyTopDaily').item.json.data.getTopChartsByGenres.podcastEpisodes[$itemIndex].audioUrl.replace(/"/g,'') }}`  
      - `summary`: Sanitized output of summary text with HTML escapes  
    - Connect input from `Summarize Podcast`.

14. **Create a Code node**  
    - Name: `Merge Results`  
    - JavaScript code:  
      ```js
      return [{ fields: $input.all().map(x => x.json) }];
      ```  
    - Connect input from `Final Data`.

15. **Create an HTML node**  
    - Name: `HTML`  
    - Configure HTML template for an email body with a table of Podcast, Episode, and Summary columns. Use dynamic iteration over `fields`.  
    - Add CSS styling for table borders and text appearance.  
    - Execute once.  
    - Connect input from `Merge Results`.

16. **Create a Gmail node**  
    - Name: `Gmail`  
    - Set subject: "Podcast Review"  
    - Message: Use `{{ $json.html }}` from `HTML` node output.  
    - Configure Gmail OAuth2 credentials (client_secret.json from Google Cloud).  
    - Connect input from `HTML`.

17. **Connect all nodes according to the flow:**  
    Schedule → Genre → TaddyTopDaily → Split Out → Download Podcast → Request Audio Crop → Get Download Link → If Downloads Ready → (True) Download Cut MP3 → Whisper Transcribe Audio → Summarize Podcast → Final Data → Merge Results → HTML → Gmail  
    If Downloads Ready → (False) Wait → Get Download Link (loop)

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Setup Taddy API key and user number in `TaddyTopDaily` node header parameters `X-USER-ID` and `X-API-KEY`.                     | https://taddy.org/signup/developers                                                            |
| Create Gmail OAuth2 credentials as per official Google guidelines and upload client_secret.json for `Gmail` node.              | https://developers.google.com/workspace/guides/create-credentials                               |
| Valid podcast genres include TECHNOLOGY, NEWS, ARTS, COMEDY, SPORTS, FICTION, etc. See full list on https://api.taddy.org.    | API Documentation and genre list                                                                |
| The workflow is designed to run daily and send a summarized email of the top 10 podcasts for the selected genre.               | Intended use-case described in workflow header and sticky notes                                 |
| Testing instructions: Replace `Schedule` node with a `Test Workflow` node for manual triggering, then check email inbox.      | Workflow header sticky note                                                                     |
| Audio cropping uses Aspose audio cutter API to reduce audio length to 8-24 minutes for faster transcription and summarization. | https://products.aspose.app/audio/cutter/                                                      |
| Transcription uses OpenAI Whisper API; summarization uses GPT-4o-mini model via OpenAI chat completions.                       | Requires OpenAI API key and configured credentials                                              |

---

This document provides a detailed, node-level understanding and stepwise reconstruction guide for the **Daily Podcast Summary** workflow, ensuring robust reproduction, modification, and integration planning.