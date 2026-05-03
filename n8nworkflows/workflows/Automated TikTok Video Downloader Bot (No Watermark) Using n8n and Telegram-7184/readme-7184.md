Automated TikTok Video Downloader Bot (No Watermark) Using n8n and Telegram

https://n8nworkflows.xyz/workflows/automated-tiktok-video-downloader-bot--no-watermark--using-n8n-and-telegram-7184


# Automated TikTok Video Downloader Bot (No Watermark) Using n8n and Telegram

### 1. Workflow Overview

This workflow is an automated TikTok video downloader bot integrated with Telegram and n8n. It enables users to send a TikTok or Instagram Reels video link via a Telegram bot, which then retrieves the highest quality video without a watermark and delivers it back to the user through Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Listens for incoming Telegram messages containing TikTok or Reels video links.
- **1.2 Video Link Processing & API Request:** Sends the received link to a third-party API (mediadl.app) to fetch video metadata and download URLs.
- **1.3 Delays for Reliable Processing:** Introduces controlled wait times to ensure API responses are ready and to avoid connection errors during download.
- **1.4 URL Extraction & Video Download:** Extracts the direct video URL from the API response and downloads the video file.
- **1.5 Delivery to User:** Sends the downloaded video file back to the user’s Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow by triggering on any new message received via Telegram, extracting the TikTok or Reels link sent by the user.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Listens for incoming Telegram messages containing user input.  
    - Configuration:  
      - Listens for "message" updates only.  
      - Uses Telegram API credentials configured with the bot token.  
    - Key Expressions: None; raw message text is extracted later.  
    - Input Connections: None (starting node).  
    - Output Connections: Passes message data to "URL Download" node.  
    - Potential Failures:  
      - Authentication errors due to invalid bot token.  
      - Network or webhook issues preventing message receipt.  
    - Version-specific: Uses Telegram Trigger v1.2.

---

#### 2.2 Video Link Processing & API Request

- **Overview:**  
  This block takes the received video link from Telegram and sends it as a POST request to the mediadl.app API to retrieve video metadata, including download URLs.

- **Nodes Involved:**  
  - URL Download

- **Node Details:**

  - **URL Download**  
    - Type: HTTP Request  
    - Role: Sends the user’s TikTok/Reels URL to the third-party API to get video info.  
    - Configuration:  
      - POST method to `https://mediadl.app/api/download`.  
      - Body includes "url" parameter set dynamically from Telegram message text (`{{$json.message.text}}`).  
      - Format parameter set to `"bestvideo+bestaudio/best"` requesting highest quality.  
      - Headers set to mimic a browser request with Accept, Origin, Referer, User-Agent, and other CORS-related headers to avoid blocking or detection.  
    - Key Expressions:  
      - `{{$json.message.text}}` extracts the video link from the Telegram message.  
    - Input Connections: From Telegram Trigger.  
    - Output Connections: To Delay 3S node.  
    - Potential Failures:  
      - API rate limiting or downtime.  
      - Invalid or malformed video URLs.  
      - Network timeouts.  
    - Version-specific: HTTP Request v4.2.

---

#### 2.3 Delays for Reliable Processing

- **Overview:**  
  Introduces wait times after receiving API response and before downloading the file to ensure data availability and avoid connection errors.

- **Nodes Involved:**  
  - Delay 3S  
  - Delay 3S1

- **Node Details:**

  - **Delay 3S**  
    - Type: Wait  
    - Role: Waits 3 seconds after API response before processing further.  
    - Configuration: 3 seconds delay.  
    - Input Connections: From URL Download.  
    - Output Connections: To Filtering URL Only.  
    - Potential Failures: None significant; only workflow pause.

  - **Delay 3S1**  
    - Type: Wait  
    - Role: Waits 3 seconds after extracting video URL before downloading video to avoid connection errors.  
    - Configuration: 3 seconds delay.  
    - Input Connections: From Filtering URL Only.  
    - Output Connections: To Download node.  
    - Potential Failures: None significant; only workflow pause.

---

#### 2.4 URL Extraction & Video Download

- **Overview:**  
  Extracts the direct URL of the video file from the API response and downloads the MP4 video file.

- **Nodes Involved:**  
  - Filtering URL Only  
  - Download

- **Node Details:**

  - **Filtering URL Only**  
    - Type: Set  
    - Role: Filters and assigns the direct video URL from the API response JSON to a variable for downloading.  
    - Configuration:  
      - Sets `medias[1].url` to the value extracted from `{{$json.medias[1].url}}`.  
    - Input Connections: From Delay 3S.  
    - Output Connections: To Delay 3S1.  
    - Potential Failures:  
      - Missing or unexpected API response structure causing expression errors.  
      - Empty or invalid URL fields.

  - **Download**  
    - Type: HTTP Request  
    - Role: Downloads the video file directly using the filtered URL via a proxy endpoint.  
    - Configuration:  
      - GET request to `https://www.mediadl.app/api/proxy-download` with query parameter `fileUrl` set to `{{$json.medias[1].url}}`.  
      - Includes headers to simulate a browser request for successful download.  
      - Response expected as file binary format (implied by downstream usage).  
    - Input Connections: From Delay 3S1.  
    - Output Connections: To Sent To Telegram Video node.  
    - Potential Failures:  
      - Connection errors or timeouts downloading the video.  
      - Invalid or expired URLs.  
      - Proxy API downtime.

---

#### 2.5 Delivery to User

- **Overview:**  
  Sends the downloaded video file back to the Telegram user in the chat where the original link was received.

- **Nodes Involved:**  
  - Sent To Telegram Video

- **Node Details:**

  - **Sent To Telegram Video**  
    - Type: Telegram node  
    - Role: Sends the downloaded video file as a Telegram video message to the user.  
    - Configuration:  
      - Chat ID dynamically set to the original sender’s chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`.  
      - Operation: "sendVideo" with binary data enabled.  
      - Filename dynamically set to the author field from the downloaded video metadata: `{{$('URL Download').item.json.author}}.mp4`.  
      - Uses the same Telegram API credentials as the trigger node.  
    - Input Connections: From Download node.  
    - Output Connections: None (end node).  
    - Potential Failures:  
      - Telegram API rate limits or downtime.  
      - File size exceeding Telegram limits.  
      - Missing or corrupted binary data.  
    - Version-specific: Telegram node v1.2.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                         | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                                   |
|-----------------------|---------------------|---------------------------------------|---------------------|------------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger    | Receives video link messages from user| None                | URL Download           |                                                                                                              |
| URL Download          | HTTP Request       | Sends video link to API for metadata  | Telegram Trigger    | Delay 3S               | # Description: This n8n automation workflow allows users to download TikTok videos without watermark...       |
| Delay 3S              | Wait               | Waits 3 seconds after API response    | URL Download        | Filtering URL Only      | **How It Works** block explaining workflow steps.                                                            |
| Filtering URL Only    | Set                 | Extracts direct video URL from API response | Delay 3S           | Delay 3S1              |                                                                                                              |
| Delay 3S1             | Wait               | Waits 3 seconds before video download | Filtering URL Only  | Download               |                                                                                                              |
| Download              | HTTP Request       | Downloads video file from filtered URL | Delay 3S1           | Sent To Telegram Video |                                                                                                              |
| Sent To Telegram Video| Telegram            | Sends downloaded video back to user   | Download            | None                   |                                                                                                              |
| Sticky Note           | Sticky Note        | Description of workflow                | None                | None                   | This n8n automation workflow allows users to download TikTok videos without watermarks...                    |
| Sticky Note1          | Sticky Note        | How the workflow operates             | None                | None                   | Explains step-by-step process: Telegram Trigger, API request, delays, download, send video.                  |
| Sticky Note2          | Sticky Note        | Workflow title                        | None                | None                   | ## Tiktok Downloader                                                                                          |
| Sticky Note3          | Sticky Note        | Setup instructions                    | None                | None                   | Detailed setup guide for Telegram Bot creation, credentials, workflow import, and activation.                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Use BotFather in Telegram to create a new bot and obtain the Bot Token.  
   - In n8n, go to Credentials and create new Telegram API credentials using this token.

2. **Add Telegram Trigger Node**  
   - Add a "Telegram Trigger" node.  
   - Set "Updates" to listen for "message".  
   - Connect the Telegram API credentials created earlier.

3. **Add HTTP Request Node ("URL Download")**  
   - Add an "HTTP Request" node.  
   - Method: POST  
   - URL: `https://mediadl.app/api/download`  
   - Body Parameters:  
     - `url`: Expression `{{$json.message.text}}` (to dynamically get Telegram message text)  
     - `format`: `"bestvideo+bestaudio/best"`  
   - Headers: Set to mimic browser headers including Accept, Origin, Referer, User-Agent, etc.  
   - Connect Telegram Trigger output to this node.

4. **Add Wait Node ("Delay 3S")**  
   - Add a "Wait" node with a duration of 3 seconds.  
   - Connect from "URL Download".

5. **Add Set Node ("Filtering URL Only")**  
   - Add a "Set" node.  
   - Create a new field `medias[1].url` and set its value to the expression `{{$json.medias[1].url}}` (extract direct video URL from API response).  
   - Connect from "Delay 3S".

6. **Add Wait Node ("Delay 3S1")**  
   - Add another "Wait" node with a duration of 3 seconds.  
   - Connect from "Filtering URL Only".

7. **Add HTTP Request Node ("Download")**  
   - Add an "HTTP Request" node.  
   - Method: GET (default)  
   - URL: `https://www.mediadl.app/api/proxy-download`  
   - Query Parameter:  
     - `fileUrl`: Set to expression `{{$json.medias[1].url}}` (direct video URL)  
   - Headers: Same browser-mimicking headers as before.  
   - Connect from "Delay 3S1".

8. **Add Telegram Node ("Sent To Telegram Video")**  
   - Add a "Telegram" node.  
   - Operation: "sendVideo"  
   - Set `chatId` to expression `{{$('Telegram Trigger').item.json.message.chat.id}}` (to send to original sender).  
   - Enable binary data sending (for video file).  
   - Set file name to expression `{{$('URL Download').item.json.author}}.mp4`.  
   - Connect Telegram API credentials.  
   - Connect from "Download".

9. **Activate Workflow**  
   - Save and activate the workflow.  
   - Test by sending a TikTok or Reels video link to the Telegram bot.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This n8n automation workflow allows users to download TikTok videos without watermarks by sending links through a Telegram bot. | General workflow description and purpose                                                        |
| How It Works: 1) Telegram Trigger listens for links 2) Sends to mediadl.app API 3) Wait delays 4) Extract direct video URL 5) Wait delay 6) Download video 7) Send video to user | Workflow operational logic summary                                                               |
| **How to Set Up:** Create Telegram bot via BotFather, configure credentials in n8n, import workflow JSON, configure HTTP nodes, activate workflow, and test. | Setup instructions for users deploying this automation                                          |
| Workflow title: Automated TikTok Video Downloader Bot (No Watermark) Using n8n and Telegram          | Workflow identification and naming                                                              |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.