Download Instagram Reels & Videos with Telegram Bot

https://n8nworkflows.xyz/workflows/download-instagram-reels---videos-with-telegram-bot-7211


# Download Instagram Reels & Videos with Telegram Bot

### 1. Workflow Overview

This n8n workflow automates the process of downloading Instagram Reels and videos via a Telegram Bot. It is designed for users who want to send an Instagram video or Reel link to a Telegram bot and receive the highest quality video file back directly in their Telegram chat.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Captures Instagram video or Reel links sent by users to the Telegram bot.
- **1.2 Video Metadata Retrieval:** Sends the received Instagram URL to a third-party API to fetch video metadata including direct download URLs.
- **1.3 Processing Delay:** Introduces short delays to ensure the external API response is ready and to prevent connection errors during download.
- **1.4 URL Extraction:** Filters the API response to extract the direct video file URL.
- **1.5 Video Download:** Downloads the video file using a proxy API endpoint.
- **1.6 Output Delivery:** Sends the downloaded video file back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for messages sent to the Telegram bot. When a user sends an Instagram URL, this block triggers the workflow.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Entry point of the workflow; listens for Telegram messages (`updates: ["message"]`).  
    - Configuration: Uses Telegram API credentials linked to the Instagram Downloader bot.  
    - Key Expression: None, receives raw incoming messages.  
    - Input: Incoming Telegram messages.  
    - Output: Message data including chat ID and message text (expected to be Instagram URLs).  
    - Potential Failures: Authentication errors if Telegram API credentials are invalid; webhook connectivity issues.  
    - Version: 1.2  

---

#### 2.2 Video Metadata Retrieval

- **Overview:**  
  Sends the Instagram URL received from the Telegram Trigger node to a third-party API (`https://www.mediadl.app/api/download`). This API returns metadata including direct video URLs for the Instagram Reel or video.

- **Nodes Involved:**  
  - URL Download

- **Node Details:**  
  - **URL Download**  
    - Type: HTTP Request (POST)  
    - Role: Queries the external media download API with the Instagram URL to get video metadata.  
    - Configuration:  
      - URL: `https://www.mediadl.app/api/download`  
      - Method: POST  
      - Body Parameters:  
        - `url`: Instagram link from Telegram message (`{{$json.message.text}}`)  
        - `format`: `"bestvideo+bestaudio/best"` to get highest quality video+audio  
      - Headers: Set to mimic a browser request (User-Agent, Accept, Referer, etc.) to ensure API compatibility.  
      - Sends body and headers as configured.  
    - Input: Instagram URL from Telegram Trigger  
    - Output: JSON response with video metadata including an array `medias` with URLs.  
    - Potential Failures: API downtime, invalid or malformed Instagram URLs, rate limiting, unexpected JSON structure.  
    - Version: 4.2  

---

#### 2.3 Processing Delay (Post Metadata Request)

- **Overview:**  
  Introduces a 3-second wait to allow the API to fully process the request and prepare the video metadata.

- **Nodes Involved:**  
  - Delay 3S

- **Node Details:**  
  - **Delay 3S**  
    - Type: Wait/Delay  
    - Role: Pauses workflow execution for 3 seconds.  
    - Configuration: Amount = 3 seconds  
    - Input: Response from URL Download  
    - Output: Passes data unchanged after delay  
    - Potential Failures: None typical, except possible workflow timeout if delay is too long and n8n instance has execution limits.  
    - Version: 1.1  

---

#### 2.4 URL Extraction

- **Overview:**  
  Extracts the first direct video URL from the metadata array `medias` returned by the API.

- **Nodes Involved:**  
  - Filtering URL Only

- **Node Details:**  
  - **Filtering URL Only**  
    - Type: Set node  
    - Role: Filters and assigns only the direct video URL to be used for download.  
    - Configuration: Sets `medias[0].url` to the value extracted from the prior JSON response.  
    - Input: JSON containing `medias` array from Delay 3S  
    - Output: Simplified JSON with only the video URL in `medias[0].url`  
    - Potential Failures: If the `medias` array is empty or undefined, this node will fail or produce no URL, leading to errors downstream.  
    - Version: 3.4  

---

#### 2.5 Processing Delay (Pre Download)

- **Overview:**  
  Adds another 3-second delay to prevent connection errors and ensure readiness before downloading the video file.

- **Nodes Involved:**  
  - Delay 3S1

- **Node Details:**  
  - **Delay 3S1**  
    - Type: Wait/Delay  
    - Role: Pauses workflow execution for 3 seconds before download.  
    - Configuration: Amount = 3 seconds  
    - Input: Output of Filtering URL Only  
    - Output: Passes data unchanged after delay  
    - Potential Failures: Same as the previous delay node.  
    - Version: 1.1  

---

#### 2.6 Video Download

- **Overview:**  
  Downloads the Instagram video file using a proxy download API endpoint to avoid direct external download issues.

- **Nodes Involved:**  
  - Download

- **Node Details:**  
  - **Download**  
    - Type: HTTP Request (GET)  
    - Role: Downloads the video file from the direct URL extracted earlier using the proxy API `https://mediadl.app/api/proxy-download`.  
    - Configuration:  
      - URL: `https://mediadl.app/api/proxy-download`  
      - Sends video URL as query parameter `fileUrl` with value `{{$json.medias[0].url}}`  
      - Headers set to mimic a browser request (User-Agent, Accept, Referer, etc.)  
      - Response Format: File (binary data expected)  
    - Input: Contains direct video URL  
    - Output: Binary video file data ready to send back to Telegram  
    - Potential Failures: Proxy API downtime, corrupted video file, network issues, invalid URL, large file timeout, rate limiting.  
    - Version: 4.2  

---

#### 2.7 Output Delivery to Telegram

- **Overview:**  
  Sends the downloaded video file back to the Telegram user who initiated the request.

- **Nodes Involved:**  
  - Sent To Telegram Video

- **Node Details:**  
  - **Sent To Telegram Video**  
    - Type: Telegram node (sendVideo operation)  
    - Role: Sends the MP4 video as a Telegram video message to the user's chat ID.  
    - Configuration:  
      - `chatId`: Extracted dynamically from Telegram Trigger node message (`{{$node["Telegram Trigger"].item.json.message.chat.id}}`)  
      - Operation: sendVideo  
      - Binary Data: true, sends the video as binary file data  
      - Additional Fields: `fileName` set dynamically from previous delay node's JSON property `title` with `.mp4` extension  
      - Uses Telegram API credentials linked to the Instagram Downloader Bot  
    - Input: Binary video from Download node  
    - Output: Confirmation of message sent to Telegram  
    - Potential Failures: Telegram API errors, invalid chat ID, file size limits in Telegram, credential expiration.  
    - Version: 1.2  

---

### 3. Summary Table

| Node Name          | Node Type             | Functional Role               | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                              |
|--------------------|-----------------------|------------------------------|----------------------|--------------------------|---------------------------------------------------------------------------------------------------------|
| Telegram Trigger    | Telegram Trigger      | Receives Instagram links via Telegram Bot | —                    | URL Download             |                                                                                                         |
| URL Download       | HTTP Request          | Requests video metadata from API | Telegram Trigger      | Delay 3S                 |                                                                                                         |
| Delay 3S           | Wait/Delay            | Waits 3 seconds for API readiness | URL Download          | Filtering URL Only       |                                                                                                         |
| Filtering URL Only  | Set                   | Extracts direct video URL from API response | Delay 3S               | Delay 3S1                |                                                                                                         |
| Delay 3S1          | Wait/Delay            | Waits 3 seconds to avoid download errors | Filtering URL Only      | Download                 |                                                                                                         |
| Download           | HTTP Request          | Downloads video file via proxy API | Delay 3S1              | Sent To Telegram Video   |                                                                                                         |
| Sent To Telegram Video | Telegram             | Sends the downloaded video back to Telegram user | Download               | —                        |                                                                                                         |
| Sticky Note        | Sticky Note           | Workflow description          | —                    | —                        | "# Description\n\nThe Instagram Downloader workflow allows users to download Instagram videos or Reels directly through a Telegram Bot. Simply send an Instagram link to the bot, and it will process the link via a third-party API to fetch the highest quality video, then send it back to your Telegram chat." |
| Sticky Note1       | Sticky Note           | How It Works explanation      | —                    | —                        | "## How It Works\n\n1. Telegram Trigger\nThe workflow starts when the bot receives an Instagram link from a user.\n\n2. HTTP Request – URL Download\nThe link is sent to the API https://www.mediadl.app/api/download to retrieve video metadata.\n\n3. Delay\nWaits a few seconds to ensure the API response is ready.\n\n4. Filtering URL Only\nExtracts the direct video file URL from the API result.\n\n5. Delay\nAdds a short pause to prevent connection errors during download.\n\n6. HTTP Request – Proxy Download\nDownloads the MP4 video file directly from the filtered URL.\n\n7. Send Video to Telegram\nSends the downloaded video back to the user in Telegram." |
| Sticky Note2       | Sticky Note           | Workflow title               | —                    | —                        | "## Instagram Downloader"                                                                                |
| Sticky Note3       | Sticky Note           | Setup instructions           | —                    | —                        | "## How to Set Up\n\n1. Create & Configure a Telegram Bot\nOpen Telegram, search for BotFather.\nSend /newbot → choose a bot name & username.\nCopy the provided Bot Token.\n\n2. Prepare Your n8n Environment\nLog in to n8n (self-hosted or n8n Cloud).\nCreate Telegram API Credentials using your Bot Token.\n\n3. Import the Workflow\nIn n8n, click Import and select Instagram_Downloader.json.\n\n4. Configure Telegram Nodes\nConnect your Telegram API credentials in the Telegram Trigger and Send Video nodes.\n\n5. Configure HTTP Request Nodes\nEnsure the URL and headers in URL Download and Download nodes are correct (already pre-configured).\nSet responseFormat to file in the final download node.\n\n6. Activate & Test\nToggle Activate.\nSend an Instagram link to your bot to test." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram API Credentials:**  
   - Use your Telegram Bot Token from BotFather to create credentials in n8n under "Telegram API".

2. **Add Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure to listen to updates of type "message".  
   - Set credentials to the Telegram API created earlier.

3. **Add HTTP Request Node (URL Download):**  
   - Type: HTTP Request (POST)  
   - URL: `https://www.mediadl.app/api/download`  
   - Method: POST  
   - Body parameters:  
     - `url`: Set to `{{$json.message.text}}` (the Instagram link from Telegram)  
     - `format`: `"bestvideo+bestaudio/best"`  
   - Headers: Add standard browser headers to mimic user-agent and origin.  
   - Send body and headers as configured.

4. **Connect Telegram Trigger → URL Download**

5. **Add Delay Node (Delay 3S):**  
   - Type: Wait  
   - Amount: 3 seconds

6. **Connect URL Download → Delay 3S**

7. **Add Set Node (Filtering URL Only):**  
   - Type: Set  
   - Assign `medias[0].url` with the expression `{{$json.medias[0].url}}` to extract the first media URL.

8. **Connect Delay 3S → Filtering URL Only**

9. **Add Delay Node (Delay 3S1):**  
   - Type: Wait  
   - Amount: 3 seconds

10. **Connect Filtering URL Only → Delay 3S1**

11. **Add HTTP Request Node (Download):**  
    - Type: HTTP Request (GET)  
    - URL: `https://mediadl.app/api/proxy-download`  
    - Query Parameters:  
      - `fileUrl`: Set to `{{$json.medias[0].url}}`  
    - Headers: Same browser-like headers as URL Download node.  
    - Response Format: File (binary)  
    - This node downloads the actual video file.

12. **Connect Delay 3S1 → Download**

13. **Add Telegram Node (Send Video):**  
    - Type: Telegram  
    - Operation: sendVideo  
    - Chat ID: Set to `{{$node["Telegram Trigger"].item.json.message.chat.id}}`  
    - Binary Data: Enabled  
    - Additional Fields: Set `fileName` to `{{$node["Delay 3S"].item.json.title}}.mp4` or a default filename if unavailable.  
    - Use the same Telegram API credentials.

14. **Connect Download → Sent To Telegram Video**

15. **Test & Activate:**  
    - Activate the workflow.  
    - Send an Instagram video or Reel URL to your Telegram bot to verify it downloads and sends back the video correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| To create and configure your Telegram bot, use BotFather in Telegram and follow the steps: `/newbot` command to get your Bot Token.       | Telegram Bot Setup                                                                                  |
| The workflow depends on the external API `https://www.mediadl.app/api/download` and `https://mediadl.app/api/proxy-download`. API downtime or changes may affect functionality. | External API Documentation, mediadl.app                                                           |
| Ensure your n8n instance has sufficient execution time limits and permissions to handle video file downloads and uploads to Telegram.     | n8n Execution Environment                                                                          |
| Keep Telegram Bot API credentials secure; regenerate tokens if exposed.                                                                    | Telegram Security                                                                                   |
| For large video files, Telegram has upload limits (~50 MB for bots); large videos may fail to send or require splitting.                  | Telegram Bot File Size Limits                                                                      |

---

**Disclaimer:**  
The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.