Download Instagram & Facebook Videos/Reels with Telegram Bot

https://n8nworkflows.xyz/workflows/download-instagram---facebook-videos-reels-with-telegram-bot-9352


# Download Instagram & Facebook Videos/Reels with Telegram Bot

### 1. Workflow Overview

This workflow automates the process of downloading Instagram and Facebook videos or reels upon receiving a Telegram message containing a link. It is designed as a Telegram bot interaction that accepts user input, validates the link (Facebook or Instagram), fetches the video source via appropriate APIs or scraping techniques, downloads the video, and sends it back to the user on Telegram.

Logical blocks:

- **1.1 Input Reception & Link Extraction:** Captures user Telegram messages, extracts, and identifies whether the link is Facebook or Instagram.
- **1.2 Link Validation:** Checks if the received link is valid and corresponds to Facebook or Instagram.
- **1.3 Facebook Video Processing:** Uses a Facebook-specific API to extract the downloadable video URL, downloads the video, and sends it to the user.
- **1.4 Instagram Video Processing:** Uses an Instagram-specific scraping endpoint to find video URLs, cleans and downloads the video, then sends it to the user.
- **1.5 Error Handling:** Sends a Telegram message if the input link is invalid or not recognized as Facebook or Instagram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Link Extraction

- **Overview:**  
  This block listens for Telegram messages and extracts potential Facebook or Instagram links from the text.

- **Nodes Involved:**  
  - Telegram Trigger  
  - FB IG LINK REGEX  

- **Node Details:**  

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Entry point, listens for incoming messages from Telegram users.  
    - Configuration: Listens for "message" updates, uses stored Telegram API credentials ("JOY AI").  
    - Input/Output: Output contains the message text and metadata.  
    - Edge Cases: Telegram API downtime, webhook misconfiguration, or missing credentials cause trigger failure.

  - **FB IG LINK REGEX**  
    - Type: Code node (JavaScript)  
    - Role: Parses the Telegram message text to detect and extract Facebook or Instagram URLs.  
    - Configuration: Uses regex patterns to match Facebook and Instagram URLs, outputs boolean flags and URLs.  
    - Key expressions:  
      ```js
      const fbRegex = /https?:\/\/([a-zA-Z0-9-]+\.)?facebook\.com\/[^\s]+/i;
      const igRegex = /https?:\/\/([a-zA-Z0-9-]+\.)?instagram\.com\/[^\s]+/i;
      ```
    - Input: Telegram message text  
    - Output: JSON with `isFacebookLink`, `facebookUrl`, `isInstagramLink`, `instagramUrl`.  
    - Edge Cases: Messages without URLs, malformed URLs, or URLs not matching regex will result in false flags.

---

#### 1.2 Link Validation

- **Overview:**  
  This block validates the extracted URLs to decide whether to proceed with Facebook or Instagram download flows or reject invalid links.

- **Nodes Involved:**  
  - If  
  - Link checking  
  - INVALID URL  

- **Node Details:**  

  - **If**  
    - Type: If Condition node  
    - Role: Checks if the input contains a valid Instagram link (`isInstagramLink` is true).  
    - Configuration: Condition on boolean `isInstagramLink`.  
    - Input: Output of FB IG LINK REGEX  
    - Output: True branch (Instagram) or False branch (to Link checking).  
    - Edge Cases: Expression evaluation errors if input lacks fields.

  - **Link checking**  
    - Type: If Condition node  
    - Role: Checks if the input contains a valid Facebook link (`isFacebookLink` is true).  
    - Configuration: Condition on boolean `isFacebookLink`.  
    - Input: False branch from If node  
    - Output: True branch (Facebook flow) or False branch (invalid link).  
    - Edge Cases: Similar expression errors possible.

  - **INVALID URL**  
    - Type: Telegram node (send message)  
    - Role: Sends an error message back to the user if neither Facebook nor Instagram links are detected.  
    - Configuration: Static text: "Please paste only facebook/Instagram link ⚠️", sends to user's chat ID from Telegram Trigger.  
    - Input: False branch of Link checking node  
    - Edge Cases: Telegram API errors, chat ID missing.

---

#### 1.3 Facebook Video Processing

- **Overview:**  
  This block handles Facebook video URLs. It calls a third-party API to retrieve a downloadable video URL, downloads the video content, and sends it to the Telegram user.

- **Nodes Involved:**  
  - FB API FETCHING  
  - video scraping  
  - Genrate download link  
  - DOWNLOAD FB VID  
  - SEND FB VIDEO  
  - Send a text message1  

- **Node Details:**  

  - **FB API FETCHING**  
    - Type: HTTP Request (POST)  
    - Role: Sends the user Facebook URL to the API endpoint `https://fbdownloader.to/api/ajaxSearch` to fetch video meta data.  
    - Configuration: POST with form-data parameter `q` set to Telegram message text.  
    - Input: True branch of Link checking node  
    - Output: Raw response with HTML/video data.  
    - Edge Cases: API downtime, rate limits, invalid responses.

  - **video scraping**  
    - Type: Set node  
    - Role: Extracts `data` field from API response JSON for further processing.  
    - Configuration: Assigns `data = {{$json.data}}`  
    - Input: FB API FETCHING  
    - Output: Cleaned data string with video page information.  
    - Edge Cases: Missing or malformed data field.

  - **Genrate download link**  
    - Type: Code node  
    - Role: Parses the HTML data to extract a direct download link via regex matching URLs starting with `https://dl.snapcdn.app/download?token=...`.  
    - Configuration: JavaScript regex  
      ```js
      const match = html.match(/https:\/\/dl\.snapcdn\.app\/download\?token=[^"]+/);
      ```
    - Input: video scraping  
    - Output: JSON with `downloadUrl` or null.  
    - Edge Cases: No matches, broken regex, missing data.

  - **DOWNLOAD FB VID**  
    - Type: HTTP Request  
    - Role: Downloads the actual video file from `downloadUrl`.  
    - Configuration: URL set to `{{$json.downloadUrl}}`, GET request.  
    - Input: Genrate download link  
    - Output: Binary video data.  
    - Edge Cases: Download failure, invalid URL, timeouts.

  - **SEND FB VIDEO**  
    - Type: Telegram node (sendVideo)  
    - Role: Sends the downloaded Facebook video back to the user's chat.  
    - Configuration: Uses Telegram API credentials, sends binary video, reply options like forceReply.  
    - Input: DOWNLOAD FB VID  
    - Output: None (end node)  
    - Edge Cases: Telegram API errors, file size limits.

  - **Send a text message1**  
    - Type: Telegram node (send message)  
    - Role: Sends a "Downloading... Please wait" message before the video is sent.  
    - Input: After DOWNLOAD FB VID (parallel to SEND FB VIDEO)  
    - Edge Cases: Telegram API errors.

---

#### 1.4 Instagram Video Processing

- **Overview:**  
  This block processes Instagram video links by scraping a third-party Instagram downloader page, extracting the video URL, cleaning it, downloading the video, and sending it to the user.

- **Nodes Involved:**  
  - INSTA API FETCHING  
  - URL FINDER  
  - URL DECODER  
  - DOWNLOAD INSTA VID  
  - SEND INSTA VIDEO  
  - Send a text message  

- **Node Details:**  

  - **INSTA API FETCHING**  
    - Type: HTTP Request (GET)  
    - Role: Fetches Instagram video page HTML from `https://snapdownloader.com/tools/instagram-downloader/download?url={{instagramUrl}}`.  
    - Configuration: GET request, URL dynamically built from Instagram URL detected.  
    - Input: True branch of If node (Instagram link)  
    - Output: HTML page content.  
    - Edge Cases: Rate limits, endpoint changes, invalid URLs.

  - **URL FINDER**  
    - Type: Code node  
    - Role: Parses the HTML content to extract the first `.mp4` video URL using regex.  
    - Configuration: Regex to find video URLs ending with `.mp4`.  
    - Input: INSTA API FETCHING  
    - Output: JSON with `videoUrl` or null.  
    - Edge Cases: No video found, regex failures.

  - **URL DECODER**  
    - Type: Code node  
    - Role: Cleans the extracted video URL, replacing HTML entities like `&amp;` with `&`.  
    - Configuration: String replace operation.  
    - Input: URL FINDER  
    - Output: JSON with `videoUrlClean`.  
    - Edge Cases: Missing URL, malformed URL.

  - **DOWNLOAD INSTA VID**  
    - Type: HTTP Request  
    - Role: Downloads the video file from cleaned video URL.  
    - Configuration: URL set to `{{$json.videoUrlClean}}`, GET request.  
    - Input: URL DECODER  
    - Output: Binary video data.  
    - Edge Cases: Download failures, invalid URLs.

  - **SEND INSTA VIDEO**  
    - Type: Telegram node (sendVideo)  
    - Role: Sends the downloaded Instagram video back to the user.  
    - Configuration: Uses Telegram API credentials, sends binary video with forceReply.  
    - Input: DOWNLOAD INSTA VID  
    - Output: None (terminal node)  
    - Edge Cases: Telegram API errors, file size constraints.

  - **Send a text message**  
    - Type: Telegram node (send message)  
    - Role: Sends a "Downloading... Please wait" message before video delivery.  
    - Input: Parallel to SEND INSTA VIDEO  
    - Edge Cases: Telegram API failures.

---

#### 1.5 Error Handling

- **Overview:**  
  Sends an error message to the user when the input does not contain a valid Facebook or Instagram link.

- **Nodes Involved:**  
  - INVALID URL  

- **Node Details:**  

  - **INVALID URL**  
    - Type: Telegram node (send message)  
    - Role: Notifies the user to paste only Facebook or Instagram links.  
    - Configuration: Static text message, sends to the chat ID from Telegram trigger.  
    - Input: False branch of Link checking node  
    - Edge Cases: Telegram API errors, missing chat ID.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                          | Input Node(s)        | Output Node(s)                        | Sticky Note                                    |
|---------------------|---------------------------|----------------------------------------|----------------------|-------------------------------------|------------------------------------------------|
| Telegram Trigger     | Telegram Trigger           | Entry point, receives Telegram message | -                    | FB IG LINK REGEX                    | ## Getting link from usr                        |
| FB IG LINK REGEX     | Code                      | Extract FB/IG URLs from message text   | Telegram Trigger     | If                                 | ## Checking FB or IG                           |
| If                  | If Condition              | Check if Instagram link present        | FB IG LINK REGEX      | INSTA API FETCHING, Link checking  | ## CHECKING VALID LINK                         |
| Link checking       | If Condition              | Check if Facebook link present          | If                   | FB API FETCHING, INVALID URL        | ## CHECKING VALID LINK                         |
| INVALID URL          | Telegram (send message)   | Send error message for invalid links   | Link checking        | -                                   | ## SENDING ERROR MSG IF INVALID LINK          |
| INSTA API FETCHING   | HTTP Request              | Fetch Instagram downloader page        | If (true branch)      | URL FINDER                         | ## INSTAGRAM NODE                              |
| URL FINDER           | Code                      | Extract Instagram video URL             | INSTA API FETCHING    | URL DECODER                       | ## INSTAGRAM NODE                              |
| URL DECODER          | Code                      | Clean video URL (decode entities)       | URL FINDER            | DOWNLOAD INSTA VID                 | ## INSTAGRAM NODE                              |
| DOWNLOAD INSTA VID   | HTTP Request              | Download Instagram video file           | URL DECODER          | SEND INSTA VIDEO, Send a text message | ## INSTAGRAM NODE                              |
| SEND INSTA VIDEO     | Telegram (sendVideo)      | Send Instagram video to Telegram user  | DOWNLOAD INSTA VID   | -                                   | ## INSTAGRAM NODE                              |
| Send a text message  | Telegram (send message)   | Send "Downloading..." message for Insta| DOWNLOAD INSTA VID   | -                                   | ## INSTAGRAM NODE                              |
| FB API FETCHING      | HTTP Request              | Call Facebook video API with user link | Link checking (true) | video scraping                    | ## FACEBOOK NODE                              |
| video scraping       | Set                       | Extract HTML data from API response     | FB API FETCHING       | Genrate download link             | ## FACEBOOK NODE                              |
| Genrate download link| Code                      | Extract Facebook direct download URL    | video scraping        | DOWNLOAD FB VID                   | ## FACEBOOK NODE                              |
| DOWNLOAD FB VID      | HTTP Request              | Download Facebook video file             | Genrate download link | SEND FB VIDEO, Send a text message1| ## FACEBOOK NODE                              |
| SEND FB VIDEO        | Telegram (sendVideo)      | Send Facebook video to Telegram user   | DOWNLOAD FB VID       | -                                   | ## FACEBOOK NODE                              |
| Send a text message1 | Telegram (send message)   | Send "Downloading..." message for Facebook | DOWNLOAD FB VID   | -                                   | ## FACEBOOK NODE                              |
| Sticky Note          | Sticky Note               | Visual note for "Getting link from usr" | -                    | -                                   | ## Getting link from usr                        |
| Sticky Note1         | Sticky Note               | Visual note for "Checking FB or IG"     | -                    | -                                   | ## Checking FB or IG                           |
| Sticky Note2         | Sticky Note               | Visual note for "FACEBOOK NODE"          | -                    | -                                   | ## FACEBOOK NODE                              |
| Sticky Note3         | Sticky Note               | Visual note for "INSTAGRAM NODE"         | -                    | -                                   | ## INSTAGRAM NODE                              |
| Sticky Note4         | Sticky Note               | Visual note for "CHECKING VALID LINK"   | -                    | -                                   | ## CHECKING VALID LINK                         |
| Sticky Note5         | Sticky Note               | Visual note for "SENDING ERROR MSG IF INVALID LINK" | -         | -                                   | ## SENDING ERROR MSG IF INVALID LINK          |
| Sticky Note6         | Sticky Note               | Large visual note, no content            | -                    | -                                   |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Node Type: Telegram Trigger  
   - Configure to listen to "message" updates.  
   - Connect Telegram API credentials (OAuth2 or token) named "JOY AI".  
   - Position: Entry point.

2. **Create FB IG LINK REGEX Code Node:**  
   - Node Type: Code  
   - Input: Output of Telegram Trigger  
   - Code: Use regex to detect Facebook and Instagram URLs and output flags and URLs.  
   - Connect Telegram Trigger → FB IG LINK REGEX.

3. **Create If Node (Instagram check):**  
   - Node Type: If  
   - Condition: `$json["isInstagramLink"] === true`  
   - Input: FB IG LINK REGEX  
   - True branch → Instagram processing nodes  
   - False branch → Link checking node.

4. **Create Link checking If Node (Facebook check):**  
   - Node Type: If  
   - Condition: `$json["isFacebookLink"] === true`  
   - Input: False branch from If node  
   - True branch → Facebook processing nodes  
   - False branch → INVALID URL node.

5. **Create INVALID URL Telegram Node:**  
   - Node Type: Telegram (send message)  
   - Text: "Please paste only facebook/Instagram link ⚠️"  
   - Chat ID: `{{$json["message"]["chat"]["id"]}}` from Telegram Trigger  
   - Input: False branch of Link checking.

6. **Facebook Processing Flow:**  

   a. **FB API FETCHING HTTP Request:**  
      - POST `https://fbdownloader.to/api/ajaxSearch`  
      - Content-Type: multipart-form-data  
      - Body param `q`: Telegram message text (`{{$json["message"]["text"]}}`)  
      - Input: True branch of Link checking.

   b. **video scraping Set Node:**  
      - Assign `data` field from FB API FETCHING response: `{{ $json.data }}`  
      - Input: FB API FETCHING.

   c. **Genrate download link Code Node:**  
      - Extract download URL using regex on `data` field.  
      - Output: `{ downloadUrl: string|null }`  
      - Input: video scraping.

   d. **DOWNLOAD FB VID HTTP Request:**  
      - GET request to `{{ $json.downloadUrl }}`  
      - Input: Genrate download link.

   e. **SEND FB VIDEO Telegram Node:**  
      - Send video operation using binary data from DOWNLOAD FB VID  
      - Chat ID: from Telegram Trigger  
      - Input: DOWNLOAD FB VID.

   f. **Send a text message1 Telegram Node:**  
      - Text: "⏬Downloading.... Please wait ⌛"  
      - Chat ID: from Telegram Trigger  
      - Input: DOWNLOAD FB VID (parallel to SEND FB VIDEO).

7. **Instagram Processing Flow:**  

   a. **INSTA API FETCHING HTTP Request:**  
      - GET request to `https://snapdownloader.com/tools/instagram-downloader/download?url={{ $json.instagramUrl }}`  
      - Input: True branch of If node.

   b. **URL FINDER Code Node:**  
      - Parse HTML from INSTA API FETCHING to extract first `.mp4` URL via regex.  
      - Output: `{ videoUrl: string|null }`  
      - Input: INSTA API FETCHING.

   c. **URL DECODER Code Node:**  
      - Replace HTML entities `&amp;` with `&` in videoUrl  
      - Output: `{ videoUrlClean: string }`  
      - Input: URL FINDER.

   d. **DOWNLOAD INSTA VID HTTP Request:**  
      - GET request to `{{ $json.videoUrlClean }}`  
      - Input: URL DECODER.

   e. **SEND INSTA VIDEO Telegram Node:**  
      - Send video operation with binary data from DOWNLOAD INSTA VID  
      - Chat ID: from Telegram Trigger  
      - Input: DOWNLOAD INSTA VID.

   f. **Send a text message Telegram Node:**  
      - Text: "⏬Downloading.... Please wait ⌛"  
      - Chat ID: from Telegram Trigger  
      - Input: DOWNLOAD INSTA VID (parallel to SEND INSTA VIDEO).

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow uses third-party scraping and APIs (`fbdownloader.to`, `snapdownloader.com`)        | These endpoints are external and may change or have rate limits affecting workflow stability.     |
| Telegram API credential named "JOY AI" used across all Telegram nodes                        | Requires proper Telegram Bot token and webhook configuration for message receipt and sending.     |
| Video file size limits on Telegram may restrict large video files from being sent            | Consider Telegram max file size constraints when deploying.                                       |
| Regex patterns in code nodes are designed for current URL formats and may require updating   | Monitor Facebook and Instagram URL structure changes to maintain functionality.                    |
| Sticky notes in workflow provide useful visual grouping and explanations for each logic part | They are for visualization only; no operational effect.                                           |