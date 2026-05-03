Automatically Repurpose TikTok Videos to Multiple Platforms with RSS.app & Blotato

https://n8nworkflows.xyz/workflows/automatically-repurpose-tiktok-videos-to-multiple-platforms-with-rss-app---blotato-9780


# Automatically Repurpose TikTok Videos to Multiple Platforms with RSS.app & Blotato

### 1. Workflow Overview

This workflow automates the repurposing of TikTok videos by detecting new TikTok posts via an RSS feed, downloading the video without watermark, uploading it to Google Drive, and then reposting it automatically across multiple social media platforms using the Blotato API. The process enables content creators to extend their reach effortlessly across various channels without manual intervention.

Logical blocks:

- **1.1 RSS Feed Trigger and TikTok Video Retrieval:** Detects new TikTok videos via an RSS feed and extracts the direct video URL.
- **1.2 Video Download and Storage:** Downloads the video from TikTok and uploads the file to Google Drive.
- **1.3 Media Upload to Blotato:** Uploads the stored video file to Blotato as media for social posting.
- **1.4 Multi-Platform Social Posting via Blotato:** Posts the video with captions to multiple social media platforms using Blotato nodes.
- **1.5 Merging and Workflow Management:** Collects the outputs from social posting nodes and handles error continuation and retries.
- **1.6 Documentation and Setup Notes:** Sticky notes provide setup instructions, troubleshooting tips, and links to documentation.

---

### 2. Block-by-Block Analysis

#### 1.1 RSS Feed Trigger and TikTok Video Retrieval

**Overview:**  
This block detects newly posted TikTok videos through an RSS feed, fetches the TikTok page HTML, and extracts the direct video URL from embedded JSON data.

**Nodes Involved:**  
- RSS Feed Trigger  
- Get Tiktok Page  
- Get Video URL  

**Node Details:**

- **RSS Feed Trigger**  
  - Type: RSS Feed Read Trigger  
  - Configuration: Polls the specified TikTok RSS feed URL every minute for new items.  
  - Key expression: `feedUrl` set to RSS.app feed URL for TikTok profile  
  - Outputs: New TikTok video item JSON with metadata including link and title  
  - Failure Modes: RSS feed downtime, network errors, malformed feed data  

- **Get Tiktok Page**  
  - Type: HTTP Request  
  - Configuration: GET request to the TikTok video page URL from RSS feed item (`$('RSS Feed Trigger').item.json.link`)  
  - Headers: Sets User-Agent to mimic a modern Chrome browser for access  
  - Response: Full HTML response as text  
  - Outputs: HTML content of TikTok video page  
  - Failure Modes: HTTP request errors, 403 Forbidden (anti-bot measures), invalid URL  

- **Get Video URL**  
  - Type: Code (JavaScript)  
  - Configuration: Runs once per item, parses HTML to find JSON script tag `__UNIVERSAL_DATA_FOR_REHYDRATION__`  
  - Extracts: Direct video URL (`playAddr`) and cookies from response headers  
  - Outputs: JSON containing `videoUrl` and `cookies`  
  - Failure Modes: HTML structure changes, missing script tag, JSON parse errors  

#### 1.2 Video Download and Storage

**Overview:**  
Downloads the TikTok video using the extracted direct URL and uploads it to a specific Google Drive folder for storage and reuse.

**Nodes Involved:**  
- Get Video  
- Upload to Google Drive  

**Node Details:**

- **Get Video**  
  - Type: HTTP Request  
  - Configuration: GET request to direct video URL (`$('Get Video URL').item.json.videoUrl`), downloads as file  
  - Headers: User-Agent, Referer to TikTok, Accept headers for video formats, and cookies from previous step for session  
  - Output: Binary video file stream  
  - Failure Modes: Download failures, expired/invalid URL, connection timeout, unauthorized access  

- **Upload to Google Drive**  
  - Type: Google Drive Node  
  - Configuration: Uploads video file as `[TikTok video title].mp4` to designated Drive and folder IDs (configured with OAuth2 credentials)  
  - Output: JSON metadata including file ID and URL  
  - Failure Modes: Google Drive API quota limits, authentication failures, file size restrictions  

#### 1.3 Media Upload to Blotato

**Overview:**  
Uploads the Google Drive hosted video to Blotato media storage to prepare for posting on social platforms.

**Nodes Involved:**  
- Upload media  

**Node Details:**

- **Upload media**  
  - Type: Blotato API Node (Community Verified node)  
  - Configuration: Uses Google Drive public download URL constructed with file ID to upload media resource to Blotato  
  - Credentials: Blotato API key with appropriate permissions  
  - Retry: Retries twice on failure with 5 seconds wait  
  - Output: Media URL hosted on Blotato for use in posts  
  - Failure Modes: API errors, media upload size/type restrictions, network timeouts  

#### 1.4 Multi-Platform Social Posting via Blotato

**Overview:**  
Posts the uploaded video along with the TikTok title as caption to multiple social media platforms through Blotato’s unified API nodes.

**Nodes Involved:**  
- Instagram [BLOTATO]  
- Youtube [BLOTATO]  
- Facebook [BLOTATO]  
- Linkedin [BLOTATO]  
- Twitter [BLOTATO]  
- Threads [BLOTATO]  
- Pinterest [BLOTATO]  
- Bluesky [BLOTATO]  

**Node Details (Common for all):**

- Type: Blotato API Node  
- Configuration:  
  - Platform specified per node (e.g., facebook, twitter, youtube)  
  - Account ID selected from cached Blotato accounts  
  - Post content text uses TikTok video title, some sliced to platform character limits  
  - Media URLs use URL from "Upload media" node output  
  - Optional scheduling parameters (e.g., Bluesky scheduled time preset far in future)  
- Credentials: Shared Blotato API key  
- Error Handling: On error, continue workflow; retry on failure with 2 max tries and 5 seconds wait  
- Output: Post response data  
- Failure Modes: API rate limits, auth token expiry, media rejection by platform, content policy violations  

#### 1.5 Merging and Workflow Management

**Overview:**  
Collects results from all social posting nodes to a single merge node for consolidated output or further processing.

**Nodes Involved:**  
- Merge1  

**Node Details:**

- **Merge1**  
  - Type: Merge  
  - Configuration: Waits for all 8 social posting nodes to complete  
  - Inputs: Main outputs of Instagram, Youtube, Facebook, Linkedin, Twitter, Threads, Pinterest, Bluesky nodes  
  - Use: Can be extended to handle aggregated reporting or error handling  
  - Failure Modes: Unmatched input counts, deadlocks if upstream nodes fail without continue-on-error  

#### 1.6 Documentation and Setup Notes

**Overview:**  
Sticky notes provide comprehensive setup instructions, troubleshooting tips, and useful external links.

**Nodes Involved:**  
- Multiple Sticky Note nodes scattered spatially for step-by-step guidance

**Node Details:**

- Sticky Notes cover:  
  - Setup instructions for RSS.app, Google Drive, Blotato accounts and credentials  
  - Important notes on avoiding spam flags and disclosing AI content when relevant  
  - Links to official Blotato API documentation, troubleshooting dashboards, and media requirements  
  - Full tutorial link: https://help.blotato.com/api/templates/8-repurpose-tiktoks-on-autopilot  
- Context: Valuable for first-time workflow deployment and ongoing maintenance  

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                               | Input Node(s)               | Output Node(s)                                                                                         | Sticky Note                                                                                                  |
|-------------------------|----------------------------|-----------------------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| RSS Feed Trigger        | RSS Feed Read Trigger       | Detect new TikTok video via RSS feed          |                             | Get Tiktok Page                                                                                       | # Setup 1: Sign up for https://rss.app, create feed, set posts=1, paste URL in Feed URL node                  |
| Get Tiktok Page         | HTTP Request                | Fetch TikTok video page HTML                   | RSS Feed Trigger            | Get Video URL                                                                                        | # 1. RSS Feed Triggers on New Tiktok Video                                                                   |
| Get Video URL           | Code                       | Parse TikTok HTML to extract direct video URL | Get Tiktok Page             | Get Video                                                                                           | # 1. RSS Feed Triggers on New Tiktok Video                                                                   |
| Get Video               | HTTP Request                | Download TikTok video file                      | Get Video URL               | Upload to Google Drive                                                                               | # 1. RSS Feed Triggers on New Tiktok Video                                                                   |
| Upload to Google Drive  | Google Drive                | Save video file to Google Drive folder          | Get Video                   | Upload media                                                                                        | # Setup 2: Connect Google Drive, select drive and folder                                                     |
| Upload media            | Blotato API Node            | Upload video file from Google Drive to Blotato | Upload to Google Drive      | Instagram [BLOTATO], Youtube [BLOTATO], Facebook [BLOTATO], Linkedin [BLOTATO], Twitter [BLOTATO], Threads [BLOTATO], Pinterest [BLOTATO], Bluesky [BLOTATO] | # Setup 3: Connect Blotato and select social media accounts                                                  |
| Instagram [BLOTATO]     | Blotato API Node            | Post video to Instagram                        | Upload media                | Merge1                                                                                              | # 2. Repurpose Everywhere Using Blotato with important usage notes and links                                 |
| Youtube [BLOTATO]       | Blotato API Node            | Post video to YouTube                          | Upload media                | Merge1                                                                                              | # 2. Repurpose Everywhere Using Blotato with important usage notes and links                                 |
| Facebook [BLOTATO]      | Blotato API Node            | Post video to Facebook                         | Upload media                | Merge1                                                                                              | # 2. Repurpose Everywhere Using Blotato with important usage notes and links                                 |
| Linkedin [BLOTATO]      | Blotato API Node            | Post video to LinkedIn                         | Upload media                | Merge1                                                                                              | # 2. Repurpose Everywhere Using Blotato with important usage notes and links                                 |
| Twitter [BLOTATO]       | Blotato API Node            | Post video to Twitter                          | Upload media                | Merge1                                                                                              | # 2. Repurpose Everywhere Using Blotato with important usage notes and links                                 |
| Threads [BLOTATO]       | Blotato API Node            | Post video to Threads                          | Upload media                | Merge1                                                                                              | # 2. Repurpose Everywhere Using Blotato with important usage notes and links                                 |
| Pinterest [BLOTATO]     | Blotato API Node            | Post video to Pinterest                        | Upload media                | Merge1                                                                                              | # 2. Repurpose Everywhere Using Blotato with important usage notes and links                                 |
| Bluesky [BLOTATO]       | Blotato API Node            | Post video to Bluesky                          | Upload media                | Merge1                                                                                              | # 2. Repurpose Everywhere Using Blotato with important usage notes and links                                 |
| Merge1                  | Merge                      | Aggregate outputs from social posting nodes   | Instagram [BLOTATO], Youtube [BLOTATO], Facebook [BLOTATO], Linkedin [BLOTATO], Twitter [BLOTATO], Threads [BLOTATO], Pinterest [BLOTATO], Bluesky [BLOTATO] |                                                                                                          |                                                                                                              |
| Sticky Note             | Sticky Note                | Setup and usage instructions, troubleshooting  |                             |                                                                                                      | Multiple notes covering setup instructions, tips, tutorial links, and API documentation                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger node**:  
   - Type: RSS Feed Read Trigger  
   - Parameters:  
     - Feed URL: Your TikTok profile RSS feed from https://rss.app (set “Number of Posts” to 1)  
     - Poll interval: Every 1 minute  

2. **Create HTTP Request node “Get Tiktok Page”**:  
   - Connect from RSS Feed Trigger  
   - Method: GET  
   - URL: Set to `={{ $('RSS Feed Trigger').item.json.link }}`  
   - Headers: Add User-Agent header with value mimicking Chrome browser (e.g., `"Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/91.0.4472.124"`)  
   - Response format: Text, full response  

3. **Create Code node “Get Video URL”**:  
   - Connect from “Get Tiktok Page”  
   - Mode: Run once per item  
   - JavaScript code: Parse HTML, extract JSON script tag `__UNIVERSAL_DATA_FOR_REHYDRATION__`, parse JSON to get direct video URL `playAddr`, extract cookies from headers  
   - Output: JSON with `videoUrl` and `cookies`  

4. **Create HTTP Request node “Get Video”**:  
   - Connect from “Get Video URL”  
   - Method: GET  
   - URL: `={{ $('Get Video URL').item.json.videoUrl }}`  
   - Headers: Set User-Agent, Referer to `https://www.tiktok.com/`, Accept video types, Connection keep-alive, and Cookie with `={{ $json.cookies }}`  
   - Response: File (binary)  
   - Allow unauthorized certificates: true (to avoid SSL issues)  

5. **Create Google Drive node “Upload to Google Drive”**:  
   - Connect from “Get Video”  
   - OAuth2 credentials: Connect your Google Drive account  
   - Drive ID: Select your Drive (e.g., “Videos” folder)  
   - Folder ID: Select subfolder to store TikTok videos  
   - File name: `={{ $('RSS Feed Trigger').item.json.title }}.mp4`  
   - Content: Use binary data from “Get Video” node  

6. **Create Blotato API node “Upload media”**:  
   - Connect from “Upload to Google Drive”  
   - Credentials: Add Blotato API key credential  
   - Resource: media  
   - Media URL: Construct direct download link from Google Drive file ID: `=https://drive.google.com/uc?export=download&id={{ $('Upload to Google Drive').item.json.id }}`  
   - Set retry on fail with 2 attempts and 5 seconds interval  

7. **Create Blotato API nodes for each social platform:**  
   - Instagram, Youtube, Facebook, Linkedin, Twitter, Threads, Pinterest, Bluesky  
   - Connect each from “Upload media” node  
   - Credentials: Use same Blotato API key  
   - For each:  
     - Set platform parameter (where applicable)  
     - Set accountId to your Blotato connected account for that platform  
     - Set postContentText to `={{ $('RSS Feed Trigger').item.json.title }}` (slice to platform limits if needed)  
     - Set postContentMediaUrls to `={{ $('Upload media').item.json.url }}`  
     - Optional: Set scheduling options, e.g., Bluesky scheduledTime far in future for testing  
   - Set onError to continue, enable retry with 2 tries and 5s wait  

8. **Create Merge node “Merge1”**:  
   - Connect all social posting nodes’ main outputs to “Merge1” inputs (set numberInputs to 8)  
   - Purpose: Aggregate post results for further handling or logging  

9. **Add Sticky Note nodes as documentation:**  
   - Add notes for setup instructions: RSS.app feed setup, Google Drive credentials, Blotato account and API key creation, and social platform configurations  
   - Add notes for troubleshooting and API documentation links  

10. **Credential Setup:**  
    - Setup Google Drive OAuth2 credential with proper Drive and folder access  
    - Setup Blotato API credential using your API key from https://my.blotato.com/settings/api  
    - Ensure all connected social accounts are linked properly in Blotato dashboard  

11. **Testing and Validation:**  
    - Enable one social platform node at a time to test  
    - Use scheduling options to avoid immediate posting during tests  
    - Monitor logs and Blotato API dashboard for errors  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Full tutorial and detailed explanation of this workflow at https://help.blotato.com/api/templates/8-repurpose-tiktoks-on-autopilot                            | Official Blotato help center                                                                        |
| Blotato API documentation, media requirements, troubleshooting dashboard, and support contact links included in sticky notes                                | https://help.blotato.com/api, https://my.blotato.com/api-dashboard                                  |
| RSS.app setup instructions for creating TikTok RSS feeds with “Number of Posts” set to 1                                                                      | https://rss.app                                                                                     |
| Important usage note: Avoid posting the exact same video repeatedly to prevent spam flags; disclose AI-generated content when applicable                      | Sticky Note in Workflow                                                                             |
| Enable “Verified Community Nodes” in n8n admin panel to use the Blotato community node                                                                        | n8n settings                                                                                       |
| When testing, recommend enabling only one social platform node and/or using scheduled posting to control publishing timing                                    | Sticky Note in Workflow                                                                             |

---

This documentation enables advanced users or automation agents to understand, reproduce, and extend the workflow while anticipating common failure points and integration requirements.