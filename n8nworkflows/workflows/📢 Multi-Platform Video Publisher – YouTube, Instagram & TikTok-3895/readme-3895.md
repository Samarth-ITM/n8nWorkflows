📢 Multi-Platform Video Publisher – YouTube, Instagram & TikTok

https://n8nworkflows.xyz/workflows/---multi-platform-video-publisher---youtube--instagram---tiktok-3895


# 📢 Multi-Platform Video Publisher – YouTube, Instagram & TikTok

### 1. Workflow Overview

This workflow, titled **"📢 Multi-Platform Video Publisher – YouTube, Instagram & TikTok"**, is designed for content creators, marketing teams, and agencies who want to **publish a single video simultaneously across YouTube, Instagram Reels, and TikTok**. It automates the process of downloading a video from a URL, then uploading and publishing it on three major social media platforms using their respective APIs and integrations.

The workflow is logically divided into the following blocks:

- **1.1 Input & Credentials Setup:** Receives user input such as video URL, titles, descriptions, and API tokens.
- **1.2 Video Download:** Downloads the video from the provided URL.
- **1.3 YouTube Upload:** Uploads the video to YouTube using the official YouTube node.
- **1.4 Instagram Reels Upload:** Uses Meta Graph API to create a container, upload the video, check processing status, and publish the reel.
- **1.5 TikTok Upload:** Searches for TikTok data and publishes the video via TikTok’s API.
- **1.6 Status Control & Retry Logic:** Uses switches and wait nodes to check video processing status on Instagram and retry publishing once ready.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Credentials Setup

- **Overview:**  
  This block initializes the workflow by setting all necessary user inputs and credentials for the three platforms. It acts as the starting point for the workflow execution.

- **Nodes Involved:**  
  - `Clicking ‘Test workflow’` (Manual Trigger)  
  - `Credentials` (Set)

- **Node Details:**

  - **Clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or manual execution.  
    - Configuration: No parameters; simply triggers downstream nodes.  
    - Inputs: None  
    - Outputs: Connects to `Credentials` node.  
    - Edge Cases: None; manual trigger.

  - **Credentials**  
    - Type: Set  
    - Role: Holds user inputs such as tokens, video URL, titles, descriptions, and Instagram account ID.  
    - Configuration: User must fill in:  
      - Instagram Token  
      - TikTok Token  
      - YouTube Title  
      - YouTube Description  
      - Video URL  
      - Instagram Account ID  
    - Inputs: From manual trigger  
    - Outputs: Feeds `Search Data Tiktok`, `Create Container` (Instagram), and `Download Vídeo` nodes.  
    - Edge Cases: Missing or invalid tokens/IDs will cause API failures downstream.

---

#### 2.2 Video Download

- **Overview:**  
  Downloads the video file from the URL provided in the `Credentials` node to be used for all platform uploads.

- **Nodes Involved:**  
  - `Download Vídeo` (HTTP Request)

- **Node Details:**

  - **Download Vídeo**  
    - Type: HTTP Request  
    - Role: Fetches the video binary data from the provided URL.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: Expression referencing the video URL from `Credentials` node.  
      - Response Format: File/Binary  
    - Inputs: From `Credentials` node  
    - Outputs: Connects to `YouTube` node for upload.  
    - Edge Cases:  
      - Invalid or inaccessible URL causes download failure.  
      - Large video files may cause timeout or memory issues.  
    - Version Requirements: HTTP Request node v4.2 or higher recommended for stable binary handling.

---

#### 2.3 YouTube Upload

- **Overview:**  
  Uploads the downloaded video to YouTube with the specified title and description.

- **Nodes Involved:**  
  - `YouTube` (YouTube node)

- **Node Details:**

  - **YouTube**  
    - Type: YouTube node (official integration)  
    - Role: Uploads video to YouTube channel.  
    - Configuration:  
      - OAuth2 credentials connected for YouTube API access.  
      - Video file input from `Download Vídeo` node binary data.  
      - Title and description set via expressions referencing `Credentials` node.  
    - Inputs: Video binary from `Download Vídeo` node  
    - Outputs: None connected further (end of YouTube branch)  
    - Edge Cases:  
      - OAuth token expiration or invalid credentials cause upload failure.  
      - API quota limits may restrict uploads.  
    - Version Requirements: YouTube node v1 or higher.

---

#### 2.4 Instagram Reels Upload & Publishing

- **Overview:**  
  This block handles Instagram Reels publishing via the Meta Graph API. It creates a media container, uploads the video, polls for processing status, and publishes the reel once ready.

- **Nodes Involved:**  
  - `Create Container` (HTTP Request)  
  - `ID Mapping` (Set)  
  - `Check Video ready` (HTTP Request)  
  - `Current Status` (Switch)  
  - `Publish Container` (HTTP Request)  
  - `Please wait 30 sec.` (Wait)

- **Node Details:**

  - **Create Container**  
    - Type: HTTP Request  
    - Role: Calls Instagram Graph API to create a media container for the video upload.  
    - Configuration:  
      - POST method to Instagram endpoint with video URL and Instagram account ID.  
      - Uses Instagram token from `Credentials`.  
    - Inputs: From `Credentials` node  
    - Outputs: Connects to `ID Mapping` node.  
    - Edge Cases:  
      - Invalid token or account ID causes API failure.  
      - API rate limits or network errors.

  - **ID Mapping**  
    - Type: Set  
    - Role: Extracts and maps container ID from the API response for use in status checking and publishing.  
    - Configuration: Uses expressions to parse JSON response.  
    - Inputs: From `Create Container` and from `Please wait 30 sec.` (loop)  
    - Outputs: Connects to `Check Video ready` node.  
    - Edge Cases: Malformed API response or missing IDs.

  - **Check Video ready**  
    - Type: HTTP Request  
    - Role: Polls Instagram API to check if the video container processing status is finished.  
    - Configuration:  
      - GET request to Instagram media container status endpoint.  
      - Uses container ID from `ID Mapping`.  
      - Uses Instagram token.  
    - Inputs: From `ID Mapping` node  
    - Outputs: Connects to `Current Status` switch node.  
    - Edge Cases: API errors, network timeouts.

  - **Current Status**  
    - Type: Switch  
    - Role: Branches logic based on video processing status.  
    - Configuration:  
      - Checks if status equals "finished" or not.  
    - Inputs: From `Check Video ready` node  
    - Outputs:  
      - If finished: to `Publish Container` node.  
      - If not finished: to `Please wait 30 sec.` node (retry).  
    - Edge Cases: Unexpected status values.

  - **Publish Container**  
    - Type: HTTP Request  
    - Role: Calls Instagram API to publish the video container as a Reel.  
    - Configuration:  
      - POST request to publish endpoint with container ID.  
      - Uses Instagram token.  
    - Inputs: From `Current Status` node (finished branch)  
    - Outputs: None (end of Instagram branch)  
    - Edge Cases: API failures, token expiration.

  - **Please wait 30 sec.**  
    - Type: Wait  
    - Role: Delays workflow for 30 seconds before retrying status check.  
    - Configuration: Fixed 30-second wait.  
    - Inputs: From `Current Status` node (not finished branch)  
    - Outputs: Loops back to `ID Mapping` node to re-check status.  
    - Edge Cases: None; ensures API rate limits respected.

---

#### 2.5 TikTok Upload

- **Overview:**  
  Publishes the video to TikTok using their official API by searching for necessary data and then uploading the video.

- **Nodes Involved:**  
  - `Search Data Tiktok` (HTTP Request)  
  - `Publish Video` (HTTP Request)

- **Node Details:**

  - **Search Data Tiktok**  
    - Type: HTTP Request  
    - Role: Queries TikTok API or internal data to prepare for video upload (e.g., getting upload URLs or video IDs).  
    - Configuration:  
      - Uses TikTok token from `Credentials`.  
      - May include video metadata or search parameters.  
    - Inputs: From `Credentials` node  
    - Outputs: Connects to `Publish Video` node.  
    - Edge Cases: API authentication errors, invalid parameters.

  - **Publish Video**  
    - Type: HTTP Request  
    - Role: Uploads and publishes the video on TikTok using data from previous node.  
    - Configuration:  
      - POST request with video binary or upload URL.  
      - Uses TikTok token.  
    - Inputs: From `Search Data Tiktok` node  
    - Outputs: None (end of TikTok branch)  
    - Edge Cases: Upload failures, token expiration, API limits.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                     | Input Node(s)              | Output Node(s)               | Sticky Note |
|-----------------------|---------------------|-----------------------------------|----------------------------|-----------------------------|-------------|
| Clicking ‘Test workflow’ | Manual Trigger      | Starts workflow manually           | -                          | Credentials                  |             |
| Credentials           | Set                 | Holds user tokens, video URL, titles | Clicking ‘Test workflow’    | Search Data Tiktok, Create Container, Download Vídeo |             |
| Download Vídeo        | HTTP Request        | Downloads video from URL           | Credentials                | YouTube                     |             |
| YouTube               | YouTube             | Uploads video to YouTube           | Download Vídeo             | -                           |             |
| Create Container      | HTTP Request        | Creates Instagram media container  | Credentials                | ID Mapping                  |             |
| ID Mapping            | Set                 | Extracts container ID for Instagram | Create Container, Please wait 30 sec. | Check Video ready           |             |
| Check Video ready     | HTTP Request        | Checks Instagram video processing status | ID Mapping                 | Current Status              |             |
| Current Status        | Switch              | Branches based on Instagram video status | Check Video ready           | Publish Container, Please wait 30 sec. |             |
| Publish Container     | HTTP Request        | Publishes Instagram Reel           | Current Status             | -                           |             |
| Please wait 30 sec.   | Wait                | Waits 30 seconds before retrying   | Current Status             | ID Mapping                  |             |
| Search Data Tiktok    | HTTP Request        | Prepares TikTok upload data        | Credentials                | Publish Video               |             |
| Publish Video         | HTTP Request        | Uploads and publishes video on TikTok | Search Data Tiktok          | -                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `Clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Create Set Node for Credentials:**  
   - Name: `Credentials`  
   - Add fields:  
     - `Token Instagram` (string)  
     - `Token TikTok` (string)  
     - `YouTube title` (string)  
     - `YouTube description` (string)  
     - `Video URL` (string)  
     - `Instagram Account ID` (string)  
   - Connect output of `Clicking ‘Test workflow’` to this node.

3. **Create HTTP Request Node to Download Video:**  
   - Name: `Download Vídeo`  
   - HTTP Method: GET  
   - URL: Use expression referencing `Video URL` from `Credentials` node (e.g., `{{$node["Credentials"].json["Video URL"]}}`)  
   - Response Format: File/Binary  
   - Connect output of `Credentials` to this node.

4. **Create YouTube Node:**  
   - Name: `YouTube`  
   - Operation: Upload Video  
   - Credentials: Connect your YouTube OAuth2 credentials  
   - Video File: Use binary data from `Download Vídeo` node  
   - Title: Expression from `Credentials` node (e.g., `{{$node["Credentials"].json["YouTube title"]}}`)  
   - Description: Expression from `Credentials` node (e.g., `{{$node["Credentials"].json["YouTube description"]}}`)  
   - Connect output of `Download Vídeo` to this node.

5. **Create HTTP Request Node for Instagram Create Container:**  
   - Name: `Create Container`  
   - HTTP Method: POST  
   - URL: Instagram Graph API endpoint for media container creation (e.g., `https://graph.facebook.com/v15.0/{{InstagramAccountID}}/media`)  
   - Authentication: Bearer token from `Token Instagram` in `Credentials`  
   - Body Parameters: video URL, media type, caption if needed  
   - Connect output of `Credentials` to this node.

6. **Create Set Node for ID Mapping:**  
   - Name: `ID Mapping`  
   - Purpose: Extract container ID from `Create Container` response JSON (e.g., `{{$json["id"]}}`)  
   - Connect output of `Create Container` to this node.

7. **Create HTTP Request Node to Check Video Ready:**  
   - Name: `Check Video ready`  
   - HTTP Method: GET  
   - URL: Instagram Graph API endpoint to check media container status using container ID from `ID Mapping` (e.g., `https://graph.facebook.com/v15.0/{{$node["ID Mapping"].json["id"]}}?fields=status_code`)  
   - Authentication: Bearer token from `Token Instagram`  
   - Connect output of `ID Mapping` to this node.

8. **Create Switch Node for Current Status:**  
   - Name: `Current Status`  
   - Condition: Check if `status_code` equals `FINISHED` (or equivalent success status)  
   - Connect output of `Check Video ready` to this node.

9. **Create HTTP Request Node to Publish Container:**  
   - Name: `Publish Container`  
   - HTTP Method: POST  
   - URL: Instagram Graph API endpoint to publish media container (e.g., `https://graph.facebook.com/v15.0/{{InstagramAccountID}}/media_publish`)  
   - Body Parameters: container ID from `ID Mapping`  
   - Authentication: Bearer token from `Token Instagram`  
   - Connect "finished" branch of `Current Status` to this node.

10. **Create Wait Node:**  
    - Name: `Please wait 30 sec.`  
    - Wait Time: 30 seconds  
    - Connect "not finished" branch of `Current Status` to this node.

11. **Loop Wait Back to ID Mapping:**  
    - Connect output of `Please wait 30 sec.` back to `ID Mapping` node to retry status check.

12. **Create HTTP Request Node for TikTok Search Data:**  
    - Name: `Search Data Tiktok`  
    - HTTP Method and URL: According to TikTok API for preparing upload (e.g., getting upload URL or video ID)  
    - Authentication: Bearer token from `Token TikTok`  
    - Connect output of `Credentials` to this node.

13. **Create HTTP Request Node for TikTok Publish Video:**  
    - Name: `Publish Video`  
    - HTTP Method: POST  
    - URL: TikTok API endpoint for video upload and publishing  
    - Body: Video binary or upload URL from `Search Data Tiktok` node  
    - Authentication: Bearer token from `Token TikTok`  
    - Connect output of `Search Data Tiktok` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                  |
|----------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow supports manual trigger and can be extended to webhook triggers for automation. | Setup instructions section                       |
| Instagram API requires valid Instagram Business or Creator account and proper permissions.   | Instagram Graph API documentation                |
| YouTube node requires OAuth2 credentials with YouTube Data API enabled.                      | YouTube API documentation                        |
| TikTok API usage requires developer account and access tokens.                              | TikTok for Developers portal                     |
| For large videos, consider API rate limits and timeout settings in HTTP Request nodes.       | n8n HTTP Request node documentation              |
| Workflow image and branding by Amanda, available at: https://i.ibb.co/zVv3psvZ/n8n-workflow-midia-publish.png | Workflow image link                              |
| More workflows and support available at https://iloveflows.com and https://n8n.partnerlinks.io/amanda | External resources and credits                    |

---

This documentation provides a detailed, structured reference to understand, reproduce, and maintain the multi-platform video publishing workflow in n8n. It covers all nodes, their configurations, connections, and potential failure points to ensure robust automation.