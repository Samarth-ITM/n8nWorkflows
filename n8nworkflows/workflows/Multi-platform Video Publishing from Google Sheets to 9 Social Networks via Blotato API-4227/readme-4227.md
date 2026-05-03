Multi-platform Video Publishing from Google Sheets to 9 Social Networks via Blotato API

https://n8nworkflows.xyz/workflows/multi-platform-video-publishing-from-google-sheets-to-9-social-networks-via-blotato-api-4227


# Multi-platform Video Publishing from Google Sheets to 9 Social Networks via Blotato API

### 1. Workflow Overview

This workflow automates multi-platform video publishing using metadata and video references stored in Google Sheets. It schedules periodic triggers to fetch new content details from Google Sheets, processes and uploads videos via the Blotato API, and publishes them across nine different social networks including YouTube, Instagram, TikTok, Facebook, LinkedIn, Twitter, Threads, Bluesky, and Pinterest. The workflow also integrates OpenAI for generating or enhancing content metadata (likely descriptions or captions) before publishing.

Logical blocks:

- **1.1 Schedule and Data Retrieval:** Periodic trigger initiates data fetch from Google Sheets and Google Drive.
- **1.2 Social Account Setup:** Prepares social media account details and credentials.
- **1.3 Video Upload:** Uploads video files to Blotato platform.
- **1.4 AI Content Processing:** Uses OpenAI node to generate or refine content metadata.
- **1.5 Multi-Platform Publishing:** Publishes the uploaded videos to various social networks through the Blotato API.
- **1.6 Image Upload and Pinterest Publishing:** Handles image upload separately for Pinterest publishing.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Data Retrieval

**Overview:**  
Triggers execution on schedule, then fetches video publishing data from Google Sheets and associated Google Drive IDs.

**Nodes Involved:**  
- Schedule Trigger  
- Google Sheets  
- Get Google Drive ID  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow on a timed schedule (default periodic trigger, no parameters shown)  
  - Connections: Output connected to Google Sheets  
  - Potential Failures: Scheduling misconfigurations, time zone issues

- **Google Sheets**  
  - Type: Google Sheets node  
  - Role: Reads video publishing data (e.g., video URLs, titles, descriptions) from a specified Google Sheet  
  - Configuration: Not detailed, but must include spreadsheet ID, sheet name, and range  
  - Connections: Output connected to Get Google Drive ID  
  - Failure Modes: Authentication errors, sheet not found, empty data

- **Get Google Drive ID**  
  - Type: Set node  
  - Role: Sets or extracts Google Drive file IDs corresponding to videos for upload  
  - Configuration: Likely uses expressions to extract IDs from Google Sheets data  
  - Connections: Output connected to Setup Social Accounts  
  - Edge Cases: Missing or malformed IDs, expression evaluation errors

---

#### 2.2 Social Account Setup

**Overview:**  
Prepares credentials and data necessary to interact with multiple social media platforms via Blotato API.

**Nodes Involved:**  
- Setup Social Accounts  

**Node Details:**

- **Setup Social Accounts**  
  - Type: Set node  
  - Role: Defines and sets social account parameters, such as access tokens or platform-specific IDs required for Blotato API calls  
  - Configuration: Likely sets JSON or variables for each platform  
  - Connections: Output connected to Upload to Blotato  
  - Failure Modes: Missing or invalid credentials, improperly formatted data

---

#### 2.3 Video Upload

**Overview:**  
Uploads the video files from Google Drive to the Blotato platform, preparing them for publishing.

**Nodes Involved:**  
- Upload to Blotato  

**Node Details:**

- **Upload to Blotato**  
  - Type: HTTP Request node  
  - Role: Sends videos to Blotato via API for hosting and further distribution  
  - Configuration: Uses HTTP POST with authentication headers, sends video data or URLs  
  - Connections: Outputs to all publishing nodes plus OpenAI node  
  - Failure Modes: Network errors, API authentication failure, invalid file references, timeout

---

#### 2.4 AI Content Processing

**Overview:**  
Uses OpenAI to generate or enhance video descriptions or captions before publishing.

**Nodes Involved:**  
- OpenAI  

**Node Details:**

- **OpenAI**  
  - Type: OpenAI node (Langchain integration)  
  - Role: Generates or refines content metadata such as video descriptions or hashtags  
  - Configuration: Model and prompt configuration not detailed but essential  
  - Connections: Output connected to Upload to Blotato - Image (for Pinterest image upload)  
  - Failure Modes: API key issues, rate limits, prompt errors, model response errors

---

#### 2.5 Multi-Platform Publishing

**Overview:**  
Publishes videos across multiple social media platforms via HTTP requests to the Blotato API.

**Nodes Involved:**  
- [Instagram] Publish via Blotato (disabled)  
- [Facebook] Publish via Blotato (disabled)  
- [Linkedin] Publish via Blotato  
- [Tiktok] Publish via Blotato  
- [Youtube] Publish via Blotato  
- [Threads] Publish via Blotato  
- [Twitter] Publish via Blotato  
- [Bluesky] Publish via Blotato  

**Node Details:**

Each node is an HTTP Request node configured to call the Blotato API endpoint corresponding to a social platform.

- **Configuration Highlights:**
  - HTTP Method: POST  
  - Authentication: Likely API key or OAuth token in headers  
  - Payload: JSON containing video ID, description, scheduling info, etc.  
  - Disabled nodes: Instagram and Facebook publishing nodes are disabled, possibly due to API changes or testing  
  - Connected from Upload to Blotato node output

- **Potential Failures:**
  - API authentication errors  
  - Platform-specific rate limits or restrictions  
  - Payload validation errors  
  - Network failures

---

#### 2.6 Image Upload and Pinterest Publishing

**Overview:**  
Handles image upload (possibly thumbnails or promotional images) and publishes to Pinterest via Blotato API.

**Nodes Involved:**  
- Upload to Blotato - Image  
- [Pinterest] Publish via Blotato  

**Node Details:**

- **Upload to Blotato - Image**  
  - Type: HTTP Request node  
  - Role: Uploads an image file to Blotato for Pinterest publishing  
  - Connections: Output connected to Pinterest publish node  
  - Failure Modes: File not found, API errors

- **[Pinterest] Publish via Blotato**  
  - Type: HTTP Request node  
  - Role: Publishes uploaded image/video to Pinterest via Blotato API  
  - Failure Modes: API limits, authentication, payload errors

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                           | Input Node(s)           | Output Node(s)                                                                                 | Sticky Note                         |
|-----------------------------|----------------------------|------------------------------------------|------------------------|------------------------------------------------------------------------------------------------|-----------------------------------|
| Schedule Trigger            | Schedule Trigger           | Initiates periodic workflow execution    | —                      | Google Sheets                                                                                  |                                   |
| Google Sheets              | Google Sheets              | Reads video publishing data               | Schedule Trigger        | Get Google Drive ID                                                                            |                                   |
| Get Google Drive ID        | Set                        | Extracts Google Drive file IDs            | Google Sheets           | Setup Social Accounts                                                                          |                                   |
| Setup Social Accounts      | Set                        | Prepares social media account parameters  | Get Google Drive ID     | Upload to Blotato                                                                             |                                   |
| Upload to Blotato          | HTTP Request               | Uploads videos to Blotato                  | Setup Social Accounts   | Multiple publishing nodes, OpenAI                                                             |                                   |
| [Instagram] Publish via Blotato | HTTP Request (disabled) | Publishes to Instagram                     | Upload to Blotato       | —                                                                                              |                                   |
| [Facebook] Publish via Blotato  | HTTP Request (disabled) | Publishes to Facebook                      | Upload to Blotato       | —                                                                                              |                                   |
| [Linkedin] Publish via Blotato  | HTTP Request            | Publishes to LinkedIn                      | Upload to Blotato       | —                                                                                              |                                   |
| [Tiktok] Publish via Blotato    | HTTP Request            | Publishes to TikTok                        | Upload to Blotato       | —                                                                                              |                                   |
| OpenAI                     | OpenAI (Langchain)         | Generates/refines video metadata           | Upload to Blotato       | Upload to Blotato - Image                                                                     |                                   |
| Upload to Blotato - Image  | HTTP Request               | Uploads images for Pinterest               | OpenAI                  | [Pinterest] Publish via Blotato                                                               |                                   |
| [Pinterest] Publish via Blotato | HTTP Request            | Publishes to Pinterest                     | Upload to Blotato - Image| —                                                                                              |                                   |
| [Youtube] Publish via Blotato    | HTTP Request            | Publishes to YouTube                       | Upload to Blotato       | —                                                                                              |                                   |
| [Threads] Publish via Blotato    | HTTP Request            | Publishes to Threads                       | Upload to Blotato       | —                                                                                              |                                   |
| [Twitter] Publish via Blotato    | HTTP Request            | Publishes to Twitter                       | Upload to Blotato       | —                                                                                              |                                   |
| [Bluesky] Publish via Blotato    | HTTP Request            | Publishes to Bluesky                       | Upload to Blotato       | —                                                                                              |                                   |
| Sticky Note                 | Sticky Note                | —                                          | —                      | —                                                                                              |                                   |
| Sticky Note1                | Sticky Note                | —                                          | —                      | —                                                                                              |                                   |
| Sticky Note2                | Sticky Note                | —                                          | —                      | —                                                                                              |                                   |
| Sticky Note4                | Sticky Note                | —                                          | —                      | —                                                                                              |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set desired periodic interval (e.g., daily, hourly) for workflow execution  
   - Connect output to Google Sheets node

2. **Create Google Sheets node**  
   - Type: Google Sheets  
   - Configure credentials for Google API access  
   - Set Spreadsheet ID and Sheet Name/range to fetch video publishing data (e.g., columns with video URLs, titles)  
   - Connect output to "Get Google Drive ID" node

3. **Create Get Google Drive ID node**  
   - Type: Set node  
   - Configure expressions to extract Google Drive file IDs from Google Sheets data (e.g., parse URLs to get file IDs)  
   - Connect output to "Setup Social Accounts" node

4. **Create Setup Social Accounts node**  
   - Type: Set node  
   - Define variables or JSON objects containing authentication tokens, API keys, or platform-specific account IDs for Blotato API calls  
   - Connect output to "Upload to Blotato" node

5. **Create Upload to Blotato node**  
   - Type: HTTP Request  
   - Configure HTTP Method: POST  
   - Set API endpoint URL for video upload (Blotato API)  
   - Configure headers with necessary authentication (API keys or OAuth tokens)  
   - In the body, include video file references or URLs and metadata as required by Blotato  
   - Connect outputs to:  
     - All social publishing HTTP Request nodes (LinkedIn, TikTok, YouTube, Threads, Twitter, Bluesky)  
     - OpenAI node

6. **Create OpenAI node**  
   - Type: OpenAI (Langchain)  
   - Configure OpenAI credentials (API key)  
   - Set prompt template to generate or enhance video metadata (captions, descriptions) based on input data  
   - Connect output to "Upload to Blotato - Image" node

7. **Create Upload to Blotato - Image node**  
   - Type: HTTP Request  
   - Configure POST method to upload images (e.g., thumbnails) to Blotato  
   - Set headers for authentication  
   - Connect output to "[Pinterest] Publish via Blotato" node

8. **Create social publishing nodes for each platform**  
   For each platform (LinkedIn, TikTok, YouTube, Threads, Twitter, Bluesky, Pinterest):  
   - Type: HTTP Request  
   - Configure HTTP Method: POST  
   - Set API endpoint URLs for each social platform via Blotato API  
   - Include required headers for authentication  
   - Configure payload with video/image IDs, descriptions, scheduling info as per Blotato API documentation  
   - Connect input from "Upload to Blotato" output (Pinterest from image upload node)  
   - Note: Instagram and Facebook nodes are optional and currently disabled in the original workflow

9. **Verify all connections**  
   - Schedule Trigger → Google Sheets → Get Google Drive ID → Setup Social Accounts → Upload to Blotato  
   - Upload to Blotato → All social publishing nodes + OpenAI  
   - OpenAI → Upload to Blotato - Image → Pinterest Publish

10. **Credential setup**  
    - Google Sheets and Google Drive: OAuth2 credentials with read access  
    - Blotato API: API key or OAuth token for HTTP Request nodes  
    - OpenAI: API key for OpenAI node

11. **Test workflow stepwise**  
    - Trigger schedule manually to verify Google Sheets data retrieval  
    - Confirm video ID extraction and social account setup  
    - Test video upload to Blotato with sample data  
    - Validate OpenAI content generation separately  
    - Test multi-platform publishing calls independently to confirm API access and payload correctness

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                        |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Workflow automates video publishing to 9 social platforms via Blotato API and Google Sheets data.   | Workflow description                                   |
| Instagram and Facebook publishing nodes are disabled, possibly due to API restrictions or testing.  | Node status in workflow                                |
| Uses OpenAI Langchain node for AI content processing, enhancing video metadata before publishing.   | Node function                                         |
| Blotato API endpoints require specific authentication and payload structure, verify documentation. | https://blotato.com/api-docs (example placeholder)    |
| Google Sheets and Drive credentials must have appropriate scopes for reading and file access.       | Google API OAuth2 scopes                               |

---

*Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*