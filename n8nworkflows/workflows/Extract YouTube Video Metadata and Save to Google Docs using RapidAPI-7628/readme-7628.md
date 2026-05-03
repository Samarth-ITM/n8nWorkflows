Extract YouTube Video Metadata and Save to Google Docs using RapidAPI

https://n8nworkflows.xyz/workflows/extract-youtube-video-metadata-and-save-to-google-docs-using-rapidapi-7628


# Extract YouTube Video Metadata and Save to Google Docs using RapidAPI

### 1. Workflow Overview

This workflow automates the extraction of detailed metadata from a YouTube video URL submitted via a form and appends the formatted metadata into a Google Docs document. It is designed for users who want to quickly gather and document video information for analysis, reporting, or record-keeping.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures user input of a YouTube video URL via a form submission trigger.
- **1.2 Metadata Retrieval:** Sends the submitted video URL to a third-party API (RapidAPI YouTube Metadata service) to fetch comprehensive video metadata.
- **1.3 Data Formatting:** Processes and reformats the raw metadata into a readable, structured text block.
- **1.4 Data Storage:** Appends the formatted text into a specified Google Docs document for documentation purposes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by waiting for a user to submit a YouTube video URL through a web form embedded in the n8n environment.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Starts the workflow upon user form submission.  
    - *Configuration:*  
      - Form title: "YouTube Metadata"  
      - One required input field labeled "url" (expects the YouTube video URL)  
    - *Expressions:* Accesses form field value via `$json.url` in downstream nodes.  
    - *Connections:* Outputs to the "YouTube Metadata" HTTP Request node.  
    - *Edge Cases / Failures:*  
      - Missing or malformed URL input (validation not explicitly handled here)  
      - Network issues affecting webhook trigger delivery  

#### 1.2 Metadata Retrieval

- **Overview:**  
  This block sends the submitted URL to the RapidAPI YouTube Metadata API to retrieve detailed metadata about the video.

- **Nodes Involved:**  
  - YouTube Metadata

- **Node Details:**  
  - **YouTube Metadata**  
    - *Type:* HTTP Request  
    - *Role:* Queries the YouTube Metadata API via RapidAPI using the submitted URL.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://youtube-metadata1.p.rapidapi.com/video_metadata.php`  
      - Content-Type: multipart/form-data  
      - Body parameter: `url` set dynamically as `={{ $json.url }}` (from form input)  
      - Headers:  
        - `x-rapidapi-host`: `youtube-metadata1.p.rapidapi.com`  
        - `x-rapidapi-key`: placeholder `"your key"` (must be replaced with a valid RapidAPI key)  
    - *Connections:* Outputs to the "Reformat" node.  
    - *Edge Cases / Failures:*  
      - Invalid or expired API key (authentication failure)  
      - API rate limiting or downtime  
      - Invalid or unsupported YouTube URL leading to API errors or empty responses  
      - Network timeouts or connectivity issues  

#### 1.3 Data Formatting

- **Overview:**  
  This block extracts relevant fields from the API response and formats them into a concise, human-readable markdown-style text string suitable for documentation.

- **Nodes Involved:**  
  - Reformat

- **Node Details:**  
  - **Reformat**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses API response JSON, extracts video metadata, and creates a formatted string.  
    - *Configuration:*  
      - Accesses the first item in the API response’s `items` array.  
      - Extracts video ID, snippet (title, description, channel, publish date, thumbnails, tags), content details (duration), and statistics (views, likes, comments).  
      - Joins tags array into a comma-separated string or defaults to "No tags available".  
      - Formats publish date into locale date string.  
      - Constructs a multi-line string with emojis and markdown-style bold formatting for clarity.  
    - *Key Expressions:* Uses `$input.first().json.items` to access API data.  
    - *Connections:* Outputs the formatted text as `docContent` to the "Append Data in Google Docs" node.  
    - *Edge Cases / Failures:*  
      - Empty or missing `items` array in API response causing access errors  
      - Missing metadata fields (e.g., no tags, no statistics) handled with defaults  
      - Unexpected data structure changes in API response  
      - Possible JavaScript runtime errors if data is malformed  

#### 1.4 Data Storage

- **Overview:**  
  This block inserts the formatted video metadata content into an existing Google Docs document for archival or further use.

- **Nodes Involved:**  
  - Append Data in Google Docs

- **Node Details:**  
  - **Append Data in Google Docs**  
    - *Type:* Google Docs node  
    - *Role:* Updates a Google Docs document by inserting the formatted metadata string.  
    - *Configuration:*  
      - Operation: `update`  
      - Action: `insert` with text content set via expression `={{ $json.docContent }}` from previous node.  
      - Authentication: Uses a Google Service Account credential (pre-configured) for access.  
    - *Connections:* Final node; no output connections.  
    - *Edge Cases / Failures:*  
      - Authentication failures if credentials are invalid or expired  
      - Document ID or permissions issues with the target Google Docs file  
      - API rate limits or Google Docs service downtime  
      - Large text content may require attention to Google Docs API limits  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                               | Input Node(s)      | Output Node(s)             | Sticky Note                                                                                                  |
|-------------------------|---------------------|-----------------------------------------------|--------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger        | Captures YouTube URL from user form submission | —                  | YouTube Metadata           | **On form submission:**  Triggers the workflow when a user submits a YouTube URL via the form.               |
| YouTube Metadata        | HTTP Request        | Calls RapidAPI to get YouTube video metadata  | On form submission | Reformat                   | **YouTube Metadata (HTTP Request):**  Sends the submitted URL to the RapidAPI YouTube Metadata service.      |
| Reformat                | Code (JavaScript)   | Parses and formats video metadata into a text | YouTube Metadata   | Append Data in Google Docs | **Reformat (Code):**  Extracts and formats key video details like title, description, stats, and thumbnails.  |
| Append Data in Google Docs | Google Docs       | Appends formatted metadata text into a Google Docs document | Reformat          | —                          | **Append Data In Google Sheet:**  Append Data in Google sheet for the future usages.                         |
| Sticky Note             | Sticky Note         | Documentation note                            | —                  | —                          | Automated YouTube Video Metadata Extraction and Documentation Workflow (overview and description)           |
| Sticky Note1            | Sticky Note         | Documentation note                            | —                  | —                          | **On form submission:**  Triggers the workflow when a user submits a YouTube URL via the form.               |
| Sticky Note2            | Sticky Note         | Documentation note                            | —                  | —                          | **YouTube Metadata (HTTP Request):**  Sends the submitted URL to the RapidAPI YouTube Metadata service.      |
| Sticky Note3            | Sticky Note         | Documentation note                            | —                  | —                          | **Reformat (Code):**  Extracts and formats key video details like title, description, stats, and thumbnails.  |
| Sticky Note4            | Sticky Note         | Documentation note                            | —                  | —                          | **Append Data In Google Sheet:**  Append Data in Google sheet for the future usages.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "On form submission" Node**  
   - Node Type: Form Trigger  
   - Configure:  
     - Form Title: `YouTube Metadata`  
     - Add one form field:  
       - Label: `url`  
       - Type: Text (default)  
       - Set as required  
   - This node will trigger the workflow when a user submits a YouTube URL.

2. **Create the "YouTube Metadata" Node**  
   - Node Type: HTTP Request  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://youtube-metadata1.p.rapidapi.com/video_metadata.php`  
     - Send Body: true  
     - Content Type: multipart/form-data  
     - Body Parameters: Add a parameter named `url` with value expression: `{{$json.url}}`  
     - Add Header Parameters:  
       - `x-rapidapi-host` : `youtube-metadata1.p.rapidapi.com`  
       - `x-rapidapi-key` : *Insert your valid RapidAPI key here*  
   - Connect "On form submission" output to this node's input.

3. **Create the "Reformat" Node**  
   - Node Type: Code (JavaScript)  
   - Configure with the following script:  
     ```javascript
     const itemsArray = $input.first().json.items;

     // Safely access the first video item
     const video = itemsArray[0];
     const {
       id: videoId,
       snippet,
       contentDetails,
       statistics,
     } = video;

     // Format tags as a comma-separated string
     const tags = snippet.tags && snippet.tags.length > 0
       ? snippet.tags.join(', ')
       : 'No tags available';

     // Format published date
     const publishedDate = new Date(snippet.publishedAt).toLocaleDateString();

     // Format duration (ISO 8601 format)
     const duration = contentDetails.duration;

     // Prepare formatted content
     const formatted = `
     🎬 **${snippet.title}**

     🧾 **Description:**
     ${snippet.description}

     📺 **Channel:** ${snippet.channelTitle}
     📅 **Published At:** ${publishedDate}

     📊 **Stats:**
     - Views: ${statistics.viewCount}
     - Likes: ${statistics.likeCount}
     - Comments: ${statistics.commentCount}

     🕒 **Duration:** ${duration}

     🏷️ **Tags:** ${tags}

     🔗 **Video URL:** https://www.youtube.com/watch?v=${videoId}
     🖼️ **Thumbnail:** ${snippet.thumbnails.high.url}
     `;

     // Return formatted string for use in Google Docs
     return [
       {
         json: {
           docContent: formatted.trim()
         }
       }
     ];
     ```  
   - Connect "YouTube Metadata" output to this node's input.

4. **Create the "Append Data in Google Docs" Node**  
   - Node Type: Google Docs  
   - Configure:  
     - Operation: `update`  
     - Actions: Add an action of type `insert` with text set to expression: `{{$json.docContent}}`  
     - Authentication: Use a Google Service Account credential configured with access to the target Google Docs document.  
   - Connect "Reformat" output to this node's input.

5. **Validate Credentials and Permissions**  
   - Ensure your RapidAPI key for the YouTube Metadata API is valid and not expired.  
   - Configure Google Service Account credentials in n8n with sufficient permissions to edit the target Google Docs file.  
   - Confirm the Google Docs document ID is properly set in the Google Docs node (usually selected during node setup).

6. **Test the Workflow**  
   - Deploy the workflow and submit a YouTube URL via the form trigger.  
   - Verify that the metadata is fetched, formatted, and appended to the Google Docs document correctly.  
   - Handle any errors such as invalid URLs or API key issues by monitoring node execution logs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Automated YouTube Video Metadata Extraction and Documentation Workflow. Easily extract detailed YouTube video metadata and automatically format and save it to Google Docs. | Workflow description from the main sticky note node.                                                     |
| Replace `"your key"` in the HTTP Request node with your valid RapidAPI key for the YouTube Metadata API.                        | Credential requirement for API access.                                                                   |
| Google Docs node uses Service Account authentication; ensure the service account has editor access to the target document.      | Authentication setup for Google Docs integration.                                                        |
| The video duration is returned in ISO 8601 format (e.g., PT4M16S) and not converted to a human-readable format.                 | Data formatting note from the Reformat node.                                                             |
| YouTube video tags may not always be available; the code defaults to "No tags available" if absent.                               | Edge case handled in the code node.                                                                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed are legal and publicly accessible.