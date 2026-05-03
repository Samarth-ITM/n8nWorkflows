Convert YouTube Videos to MP4 & MP3 with RapidAPI and Google Sheets Logging

https://n8nworkflows.xyz/workflows/convert-youtube-videos-to-mp4---mp3-with-rapidapi-and-google-sheets-logging-6924


# Convert YouTube Videos to MP4 & MP3 with RapidAPI and Google Sheets Logging

### 1. Workflow Overview

This workflow automates the process of converting YouTube videos into downloadable MP4 (multiple resolutions) and MP3 formats, leveraging the **YouTube Video Downloader API** via RapidAPI. It captures user input through a form, calls the API to retrieve download links, verifies the response success, and logs the resulting video URLs into a Google Sheets document for persistent tracking.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the YouTube video URL submitted by a user via a web form.
- **1.2 API Request & Data Retrieval:** Sends the received URL to the YouTube Video Downloader API and obtains video download links.
- **1.3 Validation:** Checks if the API returned a successful response before proceeding.
- **1.4 Data Logging:** Stores the original video URL and downloadable links for various qualities into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by providing a form interface where users submit the YouTube video URL they wish to convert.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **On form submission**  
  - **Type:** Form Trigger (Webhook-based)  
  - **Technical Role:** Initiates the workflow upon receiving form input.  
  - **Configuration:**  
    - Form Title: "YouTube To Mp4"  
    - Single required field: "URL" (expects a YouTube video URL)  
    - Webhook ID enabled for external form reception  
  - **Expressions/Variables:**  
    - Accesses the submitted URL as `{{$json.URL}}` in subsequent nodes.  
  - **Input/Output:**  
    - No input (trigger node)  
    - Output connects to "HTTP Request" node  
  - **Version Requirements:** Version 2.2 supports form triggers.  
  - **Potential Failures:**  
    - Missing or invalid URL input  
    - Webhook connectivity or permission issues  
  - **Sub-workflow:** None

---

#### 2.2 API Request & Data Retrieval

**Overview:**  
This block sends the user-provided YouTube URL to the YouTube Video Downloader API hosted on RapidAPI and retrieves downloadable video and audio links.

**Nodes Involved:**  
- HTTP Request

**Node Details:**

- **HTTP Request**  
  - **Type:** HTTP Request node  
  - **Technical Role:** Performs a POST request to the API endpoint to fetch download URLs.  
  - **Configuration:**  
    - URL: `https://youtube-video-downloader-fast.p.rapidapi.com/download.php`  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Body parameter: "url" set dynamically from the form input `={{ $json.URL }}`  
    - Headers:  
      - `x-rapidapi-host`: `"youtube-video-downloader-fast.p.rapidapi.com"`  
      - `x-rapidapi-key`: `"your key"` (replace with valid RapidAPI key)  
  - **Expressions/Variables:** Uses dynamic expression `{{$json.URL}}` from the form node.  
  - **Input/Output:**  
    - Input from "On form submission"  
    - Output to "If" node  
  - **Version Requirements:** Version 4.2 supports multipart/form-data POST requests with dynamic headers.  
  - **Potential Failures:**  
    - API authentication failure due to invalid or missing API key  
    - Network timeouts or connectivity issues  
    - API rate limits or quota exceeded  
    - Malformed or invalid URL input causing API errors  
  - **Sub-workflow:** None

---

#### 2.3 Validation

**Overview:**  
This conditional block verifies if the API response indicates a successful retrieval of download links before logging.

**Nodes Involved:**  
- If

**Node Details:**

- **If**  
  - **Type:** Conditional (If) node  
  - **Technical Role:** Checks the boolean field `success` in the API response to confirm successful data retrieval.  
  - **Configuration:**  
    - Condition: Checks if `{{$json.success}}` is true (boolean true)  
    - Uses strict type validation and case sensitivity  
  - **Expressions/Variables:** Evaluates `{{$json.success}}` from the HTTP Request node’s JSON output.  
  - **Input/Output:**  
    - Input from "HTTP Request"  
    - Output (true branch) to "Google Sheets" node  
    - No false branch configured (workflow ends silently if false)  
  - **Version Requirements:** Version 2.2 supports advanced conditional logic.  
  - **Potential Failures:**  
    - Missing or malformed `success` property in response JSON  
    - Expression evaluation errors if response schema changes  
  - **Sub-workflow:** None

---

#### 2.4 Data Logging

**Overview:**  
This block appends the video data to a Google Sheets spreadsheet to maintain a persistent log of converted videos and their download URLs.

**Nodes Involved:**  
- Google Sheets

**Node Details:**

- **Google Sheets**  
  - **Type:** Google Sheets node  
  - **Technical Role:** Appends a new row to a specified Google Sheets document with video download URLs.  
  - **Configuration:**  
    - Document ID: `1BFR7Ce8FYarGF7kvfaQf6fBsu7hLxDbzzp_RL8DDlas` (Google Sheet "YT to Mp4")  
    - Sheet Name: `gid=0` (Sheet1)  
    - Operation: Append  
    - Authentication: Service Account (via configured Google Docs credentials)  
    - Columns mapped:  
      - Url → Original video URL (`{{$json.url}}`)  
      - Mp4 360p → `{{$json.medias[0].url}}`  
      - Mp4 720 → `{{$json.medias[3].url}}`  
      - Mp4 1080 → `{{$json.medias[1].url}}`  
      - Mp3 → `{{$json.medias[18].url}}`  
    - Mapping mode: Defined below, no type conversion  
  - **Expressions/Variables:** Uses multiple JSON path expressions to extract URLs from the `medias` array in the API response.  
  - **Input/Output:**  
    - Input from "If" node (true branch)  
    - Output: None (terminal node)  
  - **Version Requirements:** Version 4.6 supports advanced Google Sheets operations with service account authentication.  
  - **Potential Failures:**  
    - Authentication errors with Google API (invalid/missing service account credentials)  
    - Incorrect document ID or sheet name causing append failure  
    - Missing or differently structured `medias` data leading to undefined URL extraction  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name          | Node Type          | Functional Role                           | Input Node(s)         | Output Node(s)     | Sticky Note                                                                                                                   |
|--------------------|--------------------|-----------------------------------------|-----------------------|--------------------|------------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger       | Captures user input YouTube video URL   | None (Trigger)        | HTTP Request       | Displays a form where the user enters the YouTube video URL. Acts as the starting trigger for the workflow.                   |
| HTTP Request       | HTTP Request       | Calls YouTube Video Downloader API      | On form submission    | If                 | Sends the provided URL to YouTube Video Downloader API via RapidAPI. Retrieves different resolution MP4 and MP3 download links.|
| If                 | If Node            | Validates API response success           | HTTP Request          | Google Sheets      | Checks if the API request was successful. Ensures only valid results are processed for logging.                                |
| Google Sheets       | Google Sheets      | Logs video URLs and metadata in spreadsheet | If                  | None               | Appends original URL, MP4 (360p, 720p, 1080p) and MP3 links into Google Sheets. Provides structured download history.         |
| Sticky Note        | Sticky Note        | Documentation and overview               | None                  | None               | # 🎥 YouTube to MP4 Downloader - Describes workflow purpose, node-by-node explanation, and customization options.             |
| Sticky Note1       | Sticky Note        | Documentation for On form submission     | None                  | None               | ### 1️⃣ **On Form Submission** - Displays form for user input and triggers workflow.                                           |
| Sticky Note2       | Sticky Note        | Documentation for HTTP Request           | None                  | None               | ### 2️⃣ **HTTP Request** - Sends URL to API and fetches video/audio links.                                                    |
| Sticky Note3       | Sticky Note        | Documentation for If Node                | None                  | None               | ### 3️⃣ **If Node** - Checks API call success before proceeding.                                                              |
| Sticky Note4       | Sticky Note        | Documentation for Google Sheets          | None                  | None               | ### 4️⃣ **Google Sheets** - Logs all download links and original URL into spreadsheet.                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Name: On form submission  
   - Configuration:  
     - Form Title: "YouTube To Mp4"  
     - Add one required field labeled "URL"  
     - Enable webhook, note the webhook URL for external usage  
   - No input connections (trigger node)

2. **Add an HTTP Request Node**  
   - Type: HTTP Request  
   - Name: HTTP Request  
   - Connect input from "On form submission" node’s main output  
   - Configuration:  
     - HTTP Method: POST  
     - URL: `https://youtube-video-downloader-fast.p.rapidapi.com/download.php`  
     - Content Type: multipart/form-data  
     - Body Parameters: Add parameter "url" with value expression `={{ $json.URL }}`  
     - Headers:  
       - `x-rapidapi-host`: `youtube-video-downloader-fast.p.rapidapi.com`  
       - `x-rapidapi-key`: your RapidAPI key (replace `"your key"`)  
     - Ensure "Send Body" and "Send Headers" are enabled

3. **Add an If Node to Validate API Response**  
   - Type: If  
   - Name: If  
   - Connect input from HTTP Request node’s main output  
   - Configuration:  
     - Condition Type: Boolean  
     - Expression: Check if `{{$json.success}}` equals `true` (boolean) with strict type validation

4. **Add a Google Sheets Node**  
   - Type: Google Sheets  
   - Name: Google Sheets  
   - Connect input from the If node’s "true" output branch  
   - Configuration:  
     - Operation: Append  
     - Document ID: Use your target Google Sheet ID or the one provided (`1BFR7Ce8FYarGF7kvfaQf6fBsu7hLxDbzzp_RL8DDlas`)  
     - Sheet Name: `gid=0` or your specific sheet tab name  
     - Authentication: Service Account (set up Google API credentials in n8n)  
     - Columns to append:  
       - Url: `={{ $json.url }}`  
       - Mp4 360p: `={{ $json.medias[0].url }}`  
       - Mp4 720: `={{ $json.medias[3].url }}`  
       - Mp4 1080: `={{ $json.medias[1].url }}`  
       - Mp3: `={{ $json.medias[18].url }}`

5. **Set Credentials**  
   - For HTTP Request: Insert the valid RapidAPI key in the header parameter  
   - For Google Sheets: Configure Google API Service Account credentials with appropriate access to the target spreadsheet

6. **Test the Workflow**  
   - Trigger the form submission with a valid YouTube video URL  
   - Confirm API returns success and links are appended in Google Sheets

7. **Optional Enhancements**  
   - Add error handling for API or Google Sheets failures  
   - Extend form to select formats or qualities  
   - Implement notifications or file downloads

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                                                |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses the **YouTube Video Downloader API** via RapidAPI to fetch download links.                                    | https://rapidapi.com/skdeveloper/api/youtube-video-downloader-fast                                                             |
| It supports multiple MP4 resolutions (360p, 720p, 1080p) and MP3 audio extraction.                                               |                                                                                                                               |
| Google Sheets logging provides a persistent history of all video conversions with direct download links.                        |                                                                                                                               |
| You can customize the workflow to add more video qualities, metadata fields, or integrate with storage and delivery services. |                                                                                                                               |
| Form Trigger node enables easy web form embedding or webhook-based external integration.                                         |                                                                                                                               |

---

**Disclaimer:** The provided content originates exclusively from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.