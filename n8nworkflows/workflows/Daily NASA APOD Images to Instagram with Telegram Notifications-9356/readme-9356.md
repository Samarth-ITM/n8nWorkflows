Daily NASA APOD Images to Instagram with Telegram Notifications

https://n8nworkflows.xyz/workflows/daily-nasa-apod-images-to-instagram-with-telegram-notifications-9356


# Daily NASA APOD Images to Instagram with Telegram Notifications

### 1. Workflow Overview

This workflow automates the daily posting of NASA’s Astronomy Picture of the Day (APOD) images to Instagram, accompanied by Telegram notifications to inform about the success or failure of the posting process. It is designed for social media managers, educators, science communicators, and astronomy fans who want an effortless way to share inspiring space imagery daily.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger & Data Retrieval:** Initiates the workflow daily at a set time and fetches the APOD data from NASA’s API.
- **1.2 Caption Preparation:** Constructs the Instagram post caption by combining the image title and explanation.
- **1.3 Instagram Media Creation:** Sends the image and caption to Instagram to create a media container.
- **1.4 Post Processing & Status Checking:** Waits for Instagram to process the media, then checks the processing status until it is ready.
- **1.5 Publishing & Notification:** Publishes the media on Instagram and sends a Telegram notification about the post status (success or failure).
- **1.6 Error Handling:** Stops the workflow and sends a failure message if the Instagram media upload or publishing fails.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger & Data Retrieval

- **Overview:** This block triggers the workflow daily at 18:00 and retrieves the APOD image, title, and explanation from NASA’s API using an authenticated HTTP request.
- **Nodes Involved:** Schedule Trigger, Get APOD

**Node: Schedule Trigger**  
- Type: Schedule Trigger  
- Role: Starts workflow execution every day at 18:00 (configurable hour).  
- Configuration: Interval trigger set to trigger at hour 18 daily.  
- Inputs: None (start node)  
- Outputs: Initiates “Get APOD” node  
- Edge Cases: Workflow will not run outside the scheduled time; time zone considerations apply.  

**Node: Get APOD**  
- Type: HTTP Request  
- Role: Fetches the Astronomy Picture of the Day data from NASA’s API.  
- Configuration:  
  - URL: https://api.nasa.gov/planetary/apod  
  - Authentication via Query Parameter using NASA API key credential.  
- Inputs: Trigger from Schedule Trigger node  
- Outputs: JSON including image URL, title, and explanation sent to “Prepare Caption” node  
- Edge Cases:  
  - API rate limits or invalid API key can cause authentication errors.  
  - Network issues or API downtime may cause timeouts or failures.  
  - The API may occasionally return non-image media (e.g., video) which could affect later processing.

#### 2.2 Caption Preparation

- **Overview:** Extracts and formats the caption and image URL from the NASA API response for Instagram posting.  
- **Nodes Involved:** Prepare Caption

**Node: Prepare Caption**  
- Type: Set  
- Role: Creates workflow variables “caption” and “imageUrl” by concatenating the APOD title and explanation, and extracting the image URL.  
- Configuration:  
  - caption: `{{ $json.title }} - {{ $json.explanation }}`  
  - imageUrl: `{{ $json.url }}`  
- Inputs: Output from “Get APOD”  
- Outputs: Passes the formatted caption and imageUrl to “Prepare IG Post”  
- Edge Cases:  
  - If fields from NASA API are missing or empty, caption could be incomplete or malformed.

#### 2.3 Instagram Media Creation

- **Overview:** Sends a POST request to Instagram’s Graph API to create a media container with the APOD image and caption.  
- **Nodes Involved:** Prepare IG Post

**Node: Prepare IG Post**  
- Type: HTTP Request  
- Role: Calls Instagram Graph API endpoint to create an unpublished media object using the image URL and caption.  
- Configuration:  
  - URL: `https://graph.facebook.com/v23.0/{YOUR_APP_ID}/media` (replace `{YOUR_APP_ID}`)  
  - Method: POST  
  - Query Parameters:  
    - image_url = `={{ $json.url }}` (image URL from previous node)  
    - caption = `={{ $json.explanation }}` (caption text)  
  - Authentication: Facebook Graph API token with permissions `instagram_basic` and `pages_manage_posts`  
- Inputs: Prepared caption and image URL from “Prepare Caption”  
- Outputs: JSON containing media container ID for subsequent publishing  
- Edge Cases:  
  - Invalid or expired Facebook Graph API token causes authentication failure.  
  - Incorrect `{YOUR_APP_ID}` or permissions will cause API errors.  
  - Network issues or API rate limits may cause failures.

#### 2.4 Post Processing & Status Checking

- **Overview:** Waits for Instagram to finish processing the media container, then polls its status until it is ready to be published.  
- **Nodes Involved:** Wait for the preparation of the post, Get Status For Publish, If

**Node: Wait for the preparation of the post**  
- Type: Wait  
- Role: Pauses workflow to allow Instagram time to process the media container before checking status.  
- Configuration: Default wait (duration not explicitly set; typically a short delay)  
- Inputs: Media container creation response from “Prepare IG Post”  
- Outputs: Triggers “Get Status For Publish” node  
- Edge Cases:  
  - Insufficient wait time may cause status checks to be premature.  

**Node: Get Status For Publish**  
- Type: HTTP Request  
- Role: Queries Instagram Graph API for the processing status of the media container.  
- Configuration:  
  - URL: `https://graph.facebook.com/v23.0/{{ $('Prepare IG Post').item.json.id }}` (media container ID)  
  - Method: GET  
  - Query Parameter: fields=status_code,status  
  - Authentication: Facebook Graph API token  
- Inputs: Triggered after wait node  
- Outputs: Status JSON to “If” node  
- Edge Cases:  
  - API errors if token expired or invalid media container ID.  
  - Network timeouts or delays may affect status accuracy.

**Node: If**  
- Type: If  
- Role: Checks if the media container status_code equals “FINISHED” indicating readiness for publishing.  
- Configuration: Condition: `status_code == "FINISHED"`  
- Inputs: Status response from “Get Status For Publish”  
- Outputs:  
  - True branch: proceeds to “Publish Post”  
  - False branch: proceeds to “Send Fail Message”  
- Edge Cases:  
  - Status codes other than “FINISHED” may require additional handling or retries (not implemented).  

#### 2.5 Publishing & Notification

- **Overview:** Publishes the processed media on Instagram and sends a Telegram notification with the success status.  
- **Nodes Involved:** Publish Post, Send Finish Message

**Node: Publish Post**  
- Type: HTTP Request  
- Role: Publishes the media container to Instagram feed.  
- Configuration:  
  - URL: `https://graph.facebook.com/v23.0/{YOUR_APP_ID}/media_publish` (replace `{YOUR_APP_ID}`)  
  - Method: POST  
  - Query Parameter: `creation_id` = media container ID (`={{ $json.id }}`)  
  - Authentication: Facebook Graph API token  
- Inputs: True branch from “If” node indicating media ready  
- Outputs: Triggers “Send Finish Message” node  
- Edge Cases:  
  - Publishing may fail due to API errors or token issues.  

**Node: Send Finish Message**  
- Type: Telegram  
- Role: Sends a Telegram chat message with the publish status (e.g., “PUBLISH STATUS: FINISHED”).  
- Configuration:  
  - Text: `=PUBLISH STATUS: {{ $('If').item.json.status_code }}`  
  - Chat ID: `{YOUR_CHAT_ID}` (replace)  
  - Authentication: Telegram Bot token  
- Inputs: Output from “Publish Post”  
- Outputs: Workflow ends successfully  
- Edge Cases:  
  - Telegram API errors due to invalid token or chat ID.  

#### 2.6 Error Handling

- **Overview:** On failure of media preparation or publishing, sends a failure Telegram notification and stops the workflow with an error message.  
- **Nodes Involved:** Send Fail Message, Stop and Error

**Node: Send Fail Message**  
- Type: Telegram  
- Role: Sends a failure notification to Telegram chat indicating the upload failed.  
- Configuration:  
  - Text: “The upload of the publication has failed, check logs.”  
  - Chat ID: `{YOUR_CHAT_ID}`  
  - Authentication: Telegram Bot token  
- Inputs: False branch from “If” node  
- Outputs: Triggers “Stop and Error” node  
- Edge Cases:  
  - Telegram API connectivity or authentication issues.  

**Node: Stop and Error**  
- Type: Stop and Error  
- Role: Stops workflow execution and raises an error with a message.  
- Configuration:  
  - Error Message: “Status Code of Publish Api Call Isn't Finished”  
- Inputs: Output from “Send Fail Message”  
- Outputs: Workflow terminates with error  
- Edge Cases: None, this node intentionally stops workflow on error.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                         | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                            |
|-------------------------------|---------------------|---------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger    | Starts workflow daily at 18:00        | None                        | Get APOD                    | Content details and introduction to the workflow (see Sticky Note7)                                                    |
| Get APOD                      | HTTP Request       | Fetches NASA APOD data                 | Schedule Trigger            | Prepare Caption             | See Sticky Note7 for workflow purpose and setup instructions                                                          |
| Prepare Caption               | Set                | Builds caption and extracts image URL | Get APOD                    | Prepare IG Post             | See Sticky Note7 for caption building explanation                                                                      |
| Prepare IG Post               | HTTP Request       | Creates Instagram media container     | Prepare Caption             | Wait for the preparation... | See Sticky Note7 for Instagram API setup steps                                                                         |
| Wait for the preparation...   | Wait               | Pauses workflow for Instagram processing | Prepare IG Post           | Get Status For Publish      |                                                                                                                        |
| Get Status For Publish        | HTTP Request       | Checks media processing status        | Wait for the preparation... | If                         |                                                                                                                        |
| If                           | If                 | Checks if media is ready to publish   | Get Status For Publish      | Publish Post, Send Fail Msg |                                                                                                                        |
| Publish Post                 | HTTP Request       | Publishes media on Instagram           | If (true branch)            | Send Finish Message         |                                                                                                                        |
| Send Finish Message           | Telegram            | Sends success notification            | Publish Post                | None                       |                                                                                                                        |
| Send Fail Message             | Telegram            | Sends failure notification            | If (false branch)           | Stop and Error              |                                                                                                                        |
| Stop and Error               | Stop and Error     | Stops workflow with error              | Send Fail Message           | None                       |                                                                                                                        |
| Sticky Note1                  | Sticky Note        | Visual - NASA APOD image example       | None                        | None                       | ![](https://apod.nasa.gov/apod/image/2510/WitchBroom_Meyers_1080.jpg)                                                 |
| Sticky Note7                  | Sticky Note        | Workflow description, setup, notes    | None                        | None                       | Detailed workflow purpose, setup steps, and customization options                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 18:00 (adjust “triggerAtHour” to desired time).  
   - Connect output to “Get APOD” node.

2. **Create Get APOD Node**  
   - Type: HTTP Request  
   - Set URL: `https://api.nasa.gov/planetary/apod`  
   - Authentication: Use Query Auth with your NASA API key credential.  
   - Connect output to “Prepare Caption” node.

3. **Create Prepare Caption Node**  
   - Type: Set  
   - Add two string fields:  
     - `caption` with value `{{ $json.title }} - {{ $json.explanation }}`  
     - `imageUrl` with value `{{ $json.url }}`  
   - Connect output to “Prepare IG Post” node.

4. **Create Prepare IG Post Node**  
   - Type: HTTP Request  
   - URL: `https://graph.facebook.com/v23.0/{YOUR_APP_ID}/media` (replace `{YOUR_APP_ID}` with your Facebook app ID)  
   - Method: POST  
   - Query Parameters:  
     - `image_url` = `={{ $json.url }}`  
     - `caption` = `={{ $json.explanation }}`  
   - Authentication: Use Facebook Graph API credential with token that has permissions `instagram_basic` and `pages_manage_posts`.  
   - Connect output to “Wait for the preparation of the post” node.

5. **Create Wait for the preparation of the post Node**  
   - Type: Wait  
   - Use default wait or specify delay (e.g., 30 seconds or 1 minute) to allow Instagram media processing.  
   - Connect output to “Get Status For Publish” node.

6. **Create Get Status For Publish Node**  
   - Type: HTTP Request  
   - URL: `https://graph.facebook.com/v23.0/{{ $('Prepare IG Post').item.json.id }}`  
   - Method: GET  
   - Query Parameter: `fields` = `status_code,status`  
   - Authentication: Facebook Graph API credential as above.  
   - Connect output to “If” node.

7. **Create If Node**  
   - Type: If  
   - Condition: Check if `status_code` equals `"FINISHED"` (case sensitive)  
   - True branch connects to “Publish Post” node.  
   - False branch connects to “Send Fail Message” node.

8. **Create Publish Post Node**  
   - Type: HTTP Request  
   - URL: `https://graph.facebook.com/v23.0/{YOUR_APP_ID}/media_publish` (replace `{YOUR_APP_ID}`)  
   - Method: POST  
   - Query Parameter: `creation_id` = `={{ $json.id }}` (media container ID)  
   - Authentication: Facebook Graph API credential as above.  
   - Connect output to “Send Finish Message” node.

9. **Create Send Finish Message Node**  
   - Type: Telegram  
   - Text: `=PUBLISH STATUS: {{ $('If').item.json.status_code }}`  
   - Chat ID: Your Telegram chat ID (replace `{YOUR_CHAT_ID}`)  
   - Authentication: Telegram Bot token credential.  
   - No further connections (end node).

10. **Create Send Fail Message Node**  
    - Type: Telegram  
    - Text: “The upload of the publication has failed, check logs.”  
    - Chat ID: Your Telegram chat ID (replace `{YOUR_CHAT_ID}`)  
    - Authentication: Telegram Bot token credential.  
    - Connect output to “Stop and Error” node.

11. **Create Stop and Error Node**  
    - Type: Stop and Error  
    - Error Message: “Status Code of Publish Api Call Isn't Finished”  
    - No outputs (workflow terminates on error).

12. **Testing and Validation**  
    - Test the workflow end-to-end with valid credentials.  
    - Verify Instagram posts and Telegram notifications.  
    - Adjust wait time if media processing is slow.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow fetches NASA’s APOD daily image and posts it to Instagram, sending Telegram notifications on success or failure. It runs automatically every day at 18:00 (adjustable). The workflow requires NASA API key, Instagram linked to Facebook Page with correct Graph API token, and Telegram bot token with chat ID. Customize posting time, caption format, notifications, or add other platforms as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | See Sticky Note7 content embedded in the workflow for detailed description and setup instructions.     |
| NASA APOD image example shown here: ![](https://apod.nasa.gov/apod/image/2510/WitchBroom_Meyers_1080.jpg)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note1                                                                                           |
| Facebook Graph API documentation for media publishing: https://developers.facebook.com/docs/instagram-api/guides/content-publishing/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Official Facebook Graph API docs                                                                       |
| NASA API documentation: https://api.nasa.gov/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | NASA API portal                                                                                         |
| Telegram Bot API documentation: https://core.telegram.org/bots/api                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Telegram Bot API docs                                                                                   |

---

This reference document fully describes the workflow "Daily NASA APOD Images to Instagram with Telegram Notifications," enabling thorough understanding, modification, and reproduction.