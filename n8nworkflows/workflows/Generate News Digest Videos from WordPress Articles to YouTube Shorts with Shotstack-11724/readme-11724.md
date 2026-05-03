Generate News Digest Videos from WordPress Articles to YouTube Shorts with Shotstack

https://n8nworkflows.xyz/workflows/generate-news-digest-videos-from-wordpress-articles-to-youtube-shorts-with-shotstack-11724


# Generate News Digest Videos from WordPress Articles to YouTube Shorts with Shotstack

### 1. Workflow Overview

This workflow automates the creation and publication of daily news digest videos from WordPress articles, optimized as YouTube Shorts. It runs once daily in the evenings, fetching all articles published that day on a WordPress site, extracting related media (videos and images), and generating a visually appealing video digest using the Shotstack API. Once rendered, the video is automatically uploaded to YouTube.

The workflow is logically structured into the following functional blocks:

- **1.1 Configuration and Scheduling:** Sets global variables and triggers the workflow daily in the evening.
- **1.2 Article Retrieval and Filtering:** Fetches published articles from WordPress for the current day and verifies a minimum count.
- **1.3 Media Extraction and Assignment:** Extracts embedded videos and featured images; assigns default images if none are found.
- **1.4 Video Payload Preparation:** Builds a detailed JSON payload for Shotstack to create the video digest.
- **1.5 Video Rendering and Monitoring:** Submits the video render request to Shotstack, waits for completion, and polls for status.
- **1.6 Video Download and YouTube Upload:** Downloads the rendered video and uploads it as a YouTube Short.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration and Scheduling

**Overview:**  
This block initializes configuration variables used throughout the workflow and schedules the workflow to run daily at 7 PM.

**Nodes Involved:**  
- Config Variables  
- Trice once a day - evenings

**Node Details:**

- **Config Variables**  
  - Type: Set  
  - Role: Defines constants such as logos, default image URL, button text and color, daily digest title, and background sound URL.  
  - Key Variables:  
    - `logo_big`: URL to main logo image used in video intro/outro.  
    - `logo_small`: URL to smaller logo for corner watermark.  
    - `default_photo_url`: Default image if no featured image exists.  
    - `daily_digest_text`: Title text for video intro/outro.  
    - `button_text`: Text for the call-to-action button.  
    - `button_bg_color`: Background color hex code for the button.  
    - `sound_url`: Background audio URL for the video.  
  - Inputs: None (start node)  
  - Outputs: Connects to "Get articles from today"  
  - Edge Cases: Missing or invalid URLs may cause rendering issues.

- **Trice once a day - evenings**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow every day at 19:00 (7 PM local time).  
  - Inputs: None  
  - Outputs: Connects to "Config Variables"  
  - Edge Cases: Misconfigured timezone could affect execution timing.

---

#### 2.2 Article Retrieval and Filtering

**Overview:**  
Retrieves all WordPress articles published on the current day and filters out runs if fewer than three articles are found.

**Nodes Involved:**  
- Get articles from today  
- Check if have enough articles  
- Check if have videos  
- Clean data and assign fields

**Node Details:**

- **Get articles from today**  
  - Type: WordPress  
  - Role: Fetches all published articles from WordPress published between midnight and current time of the day.  
  - Parameters:  
    - `status`: "publish"  
    - `after`: dynamically set to today's date at 00:00:00  
    - `before`: dynamically set to current time  
    - `returnAll`: true (get all articles)  
  - Credentials: WordPress API credentials required.  
  - Inputs: From Config Variables  
  - Outputs: Connects to "Check if have enough articles"  
  - Edge Cases: API rate limits or authentication failures; empty results if no articles.

- **Check if have enough articles**  
  - Type: If  
  - Role: Ensures there are more than two articles; otherwise, the workflow stops.  
  - Condition: Number of articles > 2  
  - Inputs: From "Get articles from today"  
  - Outputs: True branch connects to "Check if have videos"  
  - Edge Cases: Zero or one article stops workflow early.

- **Check if have videos**  
  - Type: Code  
  - Role: Parses each article’s content HTML to detect embedded video URLs.  
  - Logic:  
    - Looks for video URL in `<script class="wp-playlist-script">` JSON, `<video>` tags, or `<source>` tags.  
    - Adds a `video` field with `{src: url}` or `null`.  
  - Inputs: Output articles from "Check if have enough articles"  
  - Outputs: Connects to "Clean data and assign fields"  
  - Edge Cases: Malformed HTML or missing videos result in `null` video field.

- **Clean data and assign fields**  
  - Type: Set  
  - Role: Extracts and normalizes key article fields such as link, title (rendered), featured image ID (`photo_id`), and video object.  
  - Inputs: Output from "Check if have videos"  
  - Outputs: Connects to "Check for missing images"  
  - Edge Cases: Missing fields might cause undefined values downstream.

---

#### 2.3 Media Extraction and Assignment

**Overview:**  
Resolves the featured image URL for articles; assigns default image URL if none exists or if the image ID equals zero.

**Nodes Involved:**  
- Check for missing images  
- Assign Default Image  
- Get Image  
- Assign image URL  
- Merge image url with article details  
- Merge all articles

**Node Details:**

- **Check for missing images**  
  - Type: If  
  - Role: Checks if `photo_id` equals 0 (indicating missing featured image).  
  - Condition: `photo_id == 0`  
  - Inputs: From "Clean data and assign fields"  
  - Outputs:  
    - True branch: Connects to "Assign Default Image"  
    - False branch: Connects sequentially to "Get Image" and then "Merge image url with article details"  
  - Edge Cases: Articles without images get default assigned.

- **Assign Default Image**  
  - Type: Set  
  - Role: Assigns the default photo URL from Config Variables to `photo_url` field.  
  - Inputs: True branch from "Check for missing images"  
  - Outputs: Connects to "Merge all articles"  
  - Edge Cases: Default image must be valid URL to avoid rendering issues.

- **Get Image**  
  - Type: HTTP Request  
  - Role: Calls WordPress REST API to fetch media details for the featured image ID.  
  - URL Template: `https://your-site.com/wp-json/wp/v2/media/{{ photo_id }}`  
  - Inputs: False branch from "Check for missing images"  
  - Credentials: WordPress API credentials (same as article fetch)  
  - Outputs: Connects to "Assign image URL"  
  - Edge Cases: Invalid photo_id or API error returns failure.

- **Assign image URL**  
  - Type: Set  
  - Role: Extracts the image URL from media API response (`guid.rendered`) and assigns it to `photo_url`.  
  - Inputs: From "Get Image"  
  - Outputs: Connects to "Merge image url with article details"  
  - Edge Cases: Missing or malformed media data.

- **Merge image url with article details**  
  - Type: Merge  
  - Mode: Combine by position (index)  
  - Role: Merges the article data with the resolved image URL data, ensuring each article has associated image URL.  
  - Inputs:  
    - From "Assign image URL" (image data)  
    - From "Clean data and assign fields" (article data)  
  - Outputs: Connects to "Merge all articles"  
  - Edge Cases: Mismatched array lengths (should be avoided by design).

- **Merge all articles**  
  - Type: Merge  
  - Mode: Merge by default (combine all data per item)  
  - Role: Consolidates all article data with media URLs into a single unified list.  
  - Inputs:  
    - True branch from "Assign Default Image"  
    - From "Merge image url with article details"  
  - Outputs: Connects to "Prepare json for Shotstack"  
  - Edge Cases: None specific beyond upstream edge cases.

---

#### 2.4 Video Payload Preparation

**Overview:**  
Constructs the JSON payload required by Shotstack API to render the daily digest video, including intro/outro slides, article clips with backgrounds, overlays, logos, buttons, and looping audio.

**Nodes Involved:**  
- Prepare json for Shotstack

**Node Details:**

- **Prepare json for Shotstack**  
  - Type: Code  
  - Role: Transforms article data into a Shotstack-compatible timeline JSON structure for video rendering.  
  - Key Functions:  
    - `cleanText(text)`: Sanitizes HTML entities from article titles.  
    - `calculateReadingTime(text)`: Estimates clip duration based on word count (min 3s, max 7s).  
    - `getDateFormat()`: Returns current date string in DD-MM-YYYY format.  
  - Uses configuration variables (logos, colors, texts, sound URL).  
  - Builds several clip arrays: intro background, logos, titles, date, article clips with background media, overlays, buttons, pagination, outro slides, and looping audio track.  
  - Produces a final payload with timeline tracks and output video settings (1080x1920, mp4, 25fps).  
  - Inputs: From "Merge all articles"  
  - Outputs: Connects to "Shotstack - Submit Request"  
  - Edge Cases: Missing or invalid media URLs, empty article lists, or malformed config variables may cause rendering errors.

---

#### 2.5 Video Rendering and Monitoring

**Overview:**  
Submits the generated video payload to Shotstack for rendering, waits 30 seconds, then polls the render status until completion.

**Nodes Involved:**  
- Shotstack - Submit Request  
- Wait - Shotstack Works  
- Check Status  
- If video is ready

**Node Details:**

- **Shotstack - Submit Request**  
  - Type: HTTP Request  
  - Role: Sends POST request to Shotstack render API with JSON payload.  
  - URL: `https://api.shotstack.io/stage/render` (staging environment)  
  - Authentication: HTTP Header Auth (Shotstack API key)  
  - Headers: Content-Type: application/json  
  - Inputs: From "Prepare json for Shotstack"  
  - Outputs: Connects to "Wait - Shotstack Works"  
  - Edge Cases: API key invalid, request payload errors, network issues.

- **Wait - Shotstack Works**  
  - Type: Wait  
  - Role: Pauses workflow for 30 seconds to allow video rendering to progress.  
  - Inputs: From "Shotstack - Submit Request" or "If video is ready" false branch  
  - Outputs: Connects to "Check Status"  
  - Edge Cases: Fixed wait time might be insufficient or excessive depending on video length.

- **Check Status**  
  - Type: HTTP Request  
  - Role: GET request to Shotstack render status endpoint using the render job ID.  
  - URL Template: `https://api.shotstack.io/stage/render/{{render_id}}`  
  - Authentication: HTTP Header Auth  
  - Inputs: From "Wait - Shotstack Works"  
  - Outputs: Connects to "If video is ready"  
  - Edge Cases: API errors, job not found, or timeout if rendering fails.

- **If video is ready**  
  - Type: If  
  - Role: Checks if the Shotstack render status is "done".  
  - Condition: `$json.response.status == "done"`  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True branch: Connects to "Download video"  
    - False branch: Loops back to "Wait - Shotstack Works" to retry status check.  
  - Edge Cases: Infinite loop risk if render never completes.

---

#### 2.6 Video Download and YouTube Upload

**Overview:**  
Downloads the completed video from Shotstack and uploads it to YouTube as a Short with configured metadata.

**Nodes Involved:**  
- Download video  
- Upload video to youtube as

**Node Details:**

- **Download video**  
  - Type: HTTP Request  
  - Role: Downloads the rendered video file from the URL provided by Shotstack.  
  - URL: Taken dynamically from Shotstack render response `response.url`  
  - Inputs: From "If video is ready" true branch  
  - Outputs: Connects to "Upload video to youtube as"  
  - Edge Cases: URL invalid or inaccessible, network failures.

- **Upload video to youtube as**  
  - Type: YouTube  
  - Role: Uploads the video file as a YouTube Short.  
  - Parameters:  
    - Title: Composed from daily digest text and current date with day of week.  
    - Description: Same as title.  
    - Tags: "dailydigest"  
    - License: Creative Commons  
    - Embeddable: true  
    - Default language: Romanian ("ro")  
    - Notify subscribers: true  
    - Made for kids: true  
    - Category ID: 25 (News & Politics)  
    - Region Code: MD (Moldova)  
  - Credentials: YouTube OAuth2 API credentials  
  - Inputs: Video file from "Download video"  
  - Outputs: None (end node)  
  - Edge Cases: OAuth token expiry, upload failures, quota limits.

---

### 3. Summary Table

| Node Name                     | Node Type            | Functional Role                                | Input Node(s)                         | Output Node(s)                       | Sticky Note                                                                                                                  |
|-------------------------------|----------------------|-----------------------------------------------|-------------------------------------|------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Trice once a day - evenings    | Schedule Trigger     | Triggers workflow daily at 7 PM                |                                     | Config Variables                    |                                                                                                                              |
| Config Variables               | Set                  | Defines global variables and constants         | Trice once a day - evenings          | Get articles from today             | ## Configure workflow & retrieve articles<br>This section sets the default variables, fetches all articles published today, and checks whether any video is present. |
| Get articles from today        | WordPress            | Fetches today's published articles             | Config Variables                    | Check if have enough articles       |                                                                                                                              |
| Check if have enough articles  | If                   | Ensures minimum article count (>2)              | Get articles from today             | Check if have videos                |                                                                                                                              |
| Check if have videos           | Code                 | Extracts embedded video URLs from articles     | Check if have enough articles        | Clean data and assign fields        |                                                                                                                              |
| Clean data and assign fields   | Set                  | Extracts key fields from articles                | Check if have videos                | Check for missing images            |                                                                                                                              |
| Check for missing images       | If                   | Checks if featured image is missing (photo_id=0)| Clean data and assign fields        | Assign Default Image; Get Image     | ## Get background image URL<br>This section retrieves the image URL from WordPress, or assigns the default image if none is available. |
| Assign Default Image           | Set                  | Assigns default image URL for missing images    | Check for missing images (true)     | Merge all articles                 |                                                                                                                              |
| Get Image                     | HTTP Request          | Retrieves featured image metadata from WordPress | Check for missing images (false)    | Assign image URL                   |                                                                                                                              |
| Assign image URL               | Set                  | Extracts image URL from media metadata           | Get Image                         | Merge image url with article details|                                                                                                                              |
| Merge image url with article details | Merge           | Combines article data with image URLs            | Assign image URL, Clean data and assign fields | Merge all articles          |                                                                                                                              |
| Merge all articles             | Merge                 | Consolidates all article data with images        | Assign Default Image, Merge image url with article details | Prepare json for Shotstack  |                                                                                                                              |
| Prepare json for Shotstack     | Code                  | Builds Shotstack API payload for video rendering | Merge all articles                | Shotstack - Submit Request          | ## Generate video<br>This section prepares the JSON payload for Shotstack based on their API, submits the rendering request, and checks every 30 seconds until the video is ready, then retrieves the download URL. |
| Shotstack - Submit Request     | HTTP Request          | Submits video render request to Shotstack        | Prepare json for Shotstack          | Wait - Shotstack Works              |                                                                                                                              |
| Wait - Shotstack Works         | Wait                  | Waits 30 seconds between status checks           | Shotstack - Submit Request, If video is ready (false) | Check Status               |                                                                                                                              |
| Check Status                  | HTTP Request          | Polls Shotstack API for render job status         | Wait - Shotstack Works             | If video is ready                   |                                                                                                                              |
| If video is ready             | If                     | Checks if video render status is "done"           | Check Status                      | Download video; Wait - Shotstack Works (loop) |                                                                                                                              |
| Download video               | HTTP Request            | Downloads rendered video from Shotstack            | If video is ready (true)            | Upload video to youtube as          |                                                                                                                              |
| Upload video to youtube as    | YouTube                | Uploads the video as a YouTube Short               | Download video                     |                                  | ## Submits Short<br>This node uploads the generated Short to YouTube.                                                         |
| Sticky Note6                 | Sticky Note            | Workflow summary and setup instructions           |                                     |                                    | ## How it works<br>The workflow runs every evening. It scans all news articles published that day on your website, detects any embedded videos, and selects the featured image, or a default one if none is available. It then generates a Daily Digest video using Shotstack.com and automatically uploads it to your YouTube channel as a Short. The final video displays a list of your articles, each shown with its featured image or background video.<br><br>## Setup steps<br>We assume you are using a WordPress based news website.<br><br>- Add your credentials for WordPress, Shotstack, and YouTube.<br><br>- Configure the workflow by editing the Config Variables node, where you can set your main logo, corner logo, button color, title, default image URL, and sound URL. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger ("Trice once a day - evenings")**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 19:00 (7 PM).

2. **Create "Config Variables" (Set node)**  
   - Assign variables:  
     - `logo_big` = "https://your-domain.com/your-big-image.png"  
     - `logo_small` = "https://your-domain.com/your-small-image.png"  
     - `default_photo_url` = "https://your-domain.com/your-default-image.png"  
     - `daily_digest_text` = "Daily Digest"  
     - `button_text` = "Read more on your-site.com"  
     - `button_bg_color` = "#dd9933"  
     - `sound_url` = "https://getinnovation.dev/sounds/no-comment-minimal-background.wav"  
   - Connect Schedule Trigger output to this node.

3. **Create "Get articles from today" (WordPress node)**  
   - Operation: Get All Posts  
   - Filters:  
     - Status: Published  
     - After: Today's date at 00:00:00 (use expression to calculate dynamically)  
     - Before: Current date/time (use expression)  
   - Credentials: Set WordPress API credentials.  
   - Connect from "Config Variables".

4. **Create "Check if have enough articles" (If node)**  
   - Condition: Number of input items > 2  
   - Connect from "Get articles from today".

5. **Create "Check if have videos" (Code node)**  
   - Paste the provided JavaScript code to extract video URLs from article content HTML.  
   - Connect True branch of "Check if have enough articles" to this.

6. **Create "Clean data and assign fields" (Set node)**  
   - Assign:  
     - `link` = `{{$json["link"]}}`  
     - `title` = `{{$json["title"]["rendered"]}}`  
     - `photo_id` = `{{$json["featured_media"]}}`  
     - `video` = `{{$json["video"]}}`  
   - Connect from "Check if have videos".

7. **Create "Check for missing images" (If node)**  
   - Condition: `photo_id == 0`  
   - Connect from "Clean data and assign fields".

8. **Create "Assign Default Image" (Set node)**  
   - Assign `photo_url` = `{{ $('Config Variables').item.json.default_photo_url }}`  
   - Connect True branch from "Check for missing images".

9. **Create "Get Image" (HTTP Request node)**  
   - URL: `https://your-site.com/wp-json/wp/v2/media/{{ $json.photo_id }}`  
   - Credentials: WordPress API credentials  
   - Connect False branch from "Check for missing images".

10. **Create "Assign image URL" (Set node)**  
    - Assign `photo_url` = `{{$json.guid.rendered}}`  
    - Connect from "Get Image".

11. **Create "Merge image url with article details" (Merge node)**  
    - Mode: Combine by position  
    - Connect:  
      - Input 1: "Assign image URL"  
      - Input 2: "Clean data and assign fields"  
    - Connect from "Assign image URL" and "Clean data and assign fields".

12. **Create "Merge all articles" (Merge node)**  
    - Connect:  
      - Input 1: "Assign Default Image"  
      - Input 2: "Merge image url with article details"  
    - Connect from "Assign Default Image" and "Merge image url with article details".

13. **Create "Prepare json for Shotstack" (Code node)**  
    - Paste the provided JavaScript code that builds the Shotstack timeline payload.  
    - Connect from "Merge all articles".

14. **Create "Shotstack - Submit Request" (HTTP Request node)**  
    - Method: POST  
    - URL: `https://api.shotstack.io/stage/render`  
    - Headers: `Content-Type: application/json`  
    - Authentication: HTTP Header Auth with Shotstack API key  
    - Body: JSON from previous code node.  
    - Connect from "Prepare json for Shotstack".

15. **Create "Wait - Shotstack Works" (Wait node)**  
    - Wait time: 30 seconds  
    - Connect from "Shotstack - Submit Request".

16. **Create "Check Status" (HTTP Request node)**  
    - Method: GET  
    - URL: `https://api.shotstack.io/stage/render/{{ $json.response.id }}`  
    - Authentication: HTTP Header Auth with Shotstack API key  
    - Connect from "Wait - Shotstack Works".

17. **Create "If video is ready" (If node)**  
    - Condition: `$json.response.status == "done"`  
    - Connect from "Check Status".  
    - True branch connects to "Download video".  
    - False branch loops back to "Wait - Shotstack Works".

18. **Create "Download video" (HTTP Request node)**  
    - Method: GET  
    - URL: `{{$json.response.url}}` (dynamic)  
    - Connect from "If video is ready" True branch.

19. **Create "Upload video to youtube as" (YouTube node)**  
    - Operation: Upload Video  
    - Parameters:  
      - Title and Description: Template combining daily digest text and formatted current date with weekday.  
      - Tags: "dailydigest"  
      - License: Creative Commons  
      - Embeddable: true  
      - Default language: "ro"  
      - Notify subscribers: true  
      - Made for kids: true  
      - Category ID: 25 (News & Politics)  
      - Region code: MD  
    - Credentials: YouTube OAuth2 API credentials  
    - Connect from "Download video".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                             | Context or Link                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| The workflow runs every evening. It scans all news articles published that day on your website, detects any embedded videos, and selects the featured image, or a default one if none is available. It then generates a Daily Digest video using Shotstack.com and automatically uploads it to your YouTube channel as a Short. The final video displays a list of your articles, each shown with its featured image or background video. Setup requires WordPress, Shotstack, and YouTube credentials. Configure the workflow by editing the Config Variables node for branding, default images, and audio. | Workflow summary (Sticky Note6)             |
| Configure workflow & retrieve articles: This section sets the default variables, fetches all articles published today, and checks whether any video is present.                                                                                                                                                                                                          | Sticky Note1                                |
| Get background image URL: This section retrieves the image URL from WordPress, or assigns the default image if none is available.                                                                                                                                                                                                                                       | Sticky Note3                                |
| Generate video: This section prepares the JSON payload for Shotstack based on their API, submits the rendering request, and checks every 30 seconds until the video is ready, then retrieves the download URL.                                                                                                                                                             | Sticky Note5                                |
| Submits Short: The final node uploads the generated video to YouTube as a Short with configured metadata.                                                                                                                                                                                                                                                               | Sticky Note8                                |

---

**Disclaimer:** The content is exclusively derived from an automated n8n workflow. It complies fully with content policies and contains no illegal or protected materials. All data processed is legal and publicly accessible.