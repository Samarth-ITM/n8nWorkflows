Create Daily 4K Wallpapers from NASA's APOD with Google Drive & Slack Alerts

https://n8nworkflows.xyz/workflows/create-daily-4k-wallpapers-from-nasa-s-apod-with-google-drive---slack-alerts-10995


# Create Daily 4K Wallpapers from NASA's APOD with Google Drive & Slack Alerts

### 1. Workflow Overview

This workflow automates the daily retrieval, processing, and storage of NASA's Astronomy Picture of the Day (APOD). It is designed to:

- Fetch the APOD metadata for the previous day.
- Check if the APOD media is an image.
- Download the image and resize it to 4K resolution (3840x2160).
- Generate a standardized filename.
- Upload the processed 4K image to a configured Google Drive folder.
- If the APOD is not an image (e.g., a video), send a Slack notification alerting the user.

The workflow is structured into the following logical blocks:

- **1.1 Daily Trigger & Configuration Setup:** Initiates the workflow daily and sets key configuration parameters.
- **1.2 APOD Metadata Retrieval & Media Type Check:** Fetches the APOD metadata and determines if the media is an image or not.
- **1.3 Image Download & Processing:** Downloads the APOD image and resizes it to 4K resolution.
- **1.4 Filename Generation & Storage Decision:** Creates a filename and decides whether to upload based on storage provider.
- **1.5 Upload to Google Drive:** Uploads the resized image to Google Drive.
- **1.6 Slack Notification on Non-Image Media:** Sends Slack alerts if the APOD is not an image.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger & Configuration Setup

- **Overview:** This block triggers the workflow once daily and sets configuration parameters needed throughout the workflow.
- **Nodes Involved:** Daily Schedule, Workflow Configuration
- **Node Details:**

  - **Daily Schedule**
    - Type: Schedule Trigger
    - Role: Initiates the workflow execution daily (interval set to 1 day).
    - Configuration: Default daily interval without specific time customization.
    - Input: None
    - Output: Triggers the next node (Workflow Configuration).
    - Edge cases: Delay or failure in trigger execution could delay the workflow.
  
  - **Workflow Configuration**
    - Type: Set
    - Role: Stores workflow-wide configuration variables such as storage provider and Google Drive folder ID.
    - Configuration:
      - STORAGE_PROVIDER = "drive"
      - DRIVE_FOLDER_ID = "1uOPQCyF_w9VX7FcgvAmBAjD3v0jl8MSU" (user must replace with their folder ID)
    - Input: Trigger from Daily Schedule
    - Output: Passes configuration to Fetch Metadata node.
    - Edge cases: Misconfiguration or missing folder ID would cause upload failures.

#### 1.2 APOD Metadata Retrieval & Media Type Check

- **Overview:** Fetches metadata for the previous day's APOD and checks if the media type is an image.
- **Nodes Involved:** Fetch Metadata, If (media type check)
- **Node Details:**

  - **Fetch Metadata**
    - Type: NASA API node
    - Role: Retrieves APOD metadata for yesterday’s date.
    - Configuration: Date set dynamically to yesterday (`{{$now.minus({ days: 1 }).toISODate()}}`).
    - Input: Workflow Configuration output
    - Output: APOD metadata JSON to the If node.
    - Edge cases: API rate limits, network errors, or no data returned for that date.
  
  - **If (Media Type Check)**
    - Type: If node
    - Role: Branches workflow based on whether APOD media type is "image".
    - Configuration: Checks `media_type === "image"`.
    - Input: Output from Fetch Metadata
    - Output: True branch leads to image download; False branch leads to Slack notification.
    - Edge cases: Unexpected media_type values, missing media_type field.

#### 1.3 Image Download & Processing

- **Overview:** Downloads the APOD image and resizes it to 4K resolution.
- **Nodes Involved:** Download Image, Resize Image
- **Node Details:**

  - **Download Image**
    - Type: HTTP Request
    - Role: Downloads the APOD image file.
    - Configuration:
      - URL dynamically set to `hdurl` if available, else `url` from APOD metadata.
      - Response configured to return file content.
    - Input: True branch from If (media type check).
    - Output: Passes downloaded image to Resize Image node.
    - Edge cases: Broken URLs, timeouts, or unsupported media types.
  
  - **Resize Image**
    - Type: Edit Image
    - Role: Resizes downloaded image to 3840x2160 pixels (4K).
    - Configuration:
      - Operation: Resize
      - Width: 3840
      - Height: 2160
    - Input: Download Image output
    - Output: Passes resized image to Generate Filename node.
    - Edge cases: Corrupted images, unsupported formats.

#### 1.4 Filename Generation & Storage Decision

- **Overview:** Generates a consistent filename and checks which storage provider to use.
- **Nodes Involved:** Generate Filename, Check Storage Provider
- **Node Details:**

  - **Generate Filename**
    - Type: Set
    - Role: Creates a filename string based on APOD date.
    - Configuration:
      - Filename format: `apod_YYYY-MM-DD.jpg` using date from Fetch Metadata node.
    - Input: Resized image from Resize Image node.
    - Output: Passes filename and image data to Check Storage Provider node.
    - Edge cases: Date format inconsistencies.
  
  - **Check Storage Provider**
    - Type: If node
    - Role: Determines if the storage provider is Google Drive.
    - Configuration:
      - Checks if `STORAGE_PROVIDER` equals "drive".
    - Input: Generate Filename output
    - Output: True branch to upload node; False branch not used here (no alternative storage implemented).
    - Edge cases: Misconfigured STORAGE_PROVIDER value.

#### 1.5 Upload to Google Drive

- **Overview:** Uploads the resized and renamed image file to the specified Google Drive folder.
- **Nodes Involved:** Upload to Google Drive
- **Node Details:**

  - **Upload to Google Drive**
    - Type: Google Drive
    - Role: Uploads the image file with the generated filename to the configured Drive folder.
    - Configuration:
      - Filename set dynamically from previous node.
      - Folder ID set from Workflow Configuration.
      - Drive ID targeting "My Drive".
    - Input: True output from Check Storage Provider node.
    - Output: End of workflow for this branch.
    - Version-specific: Requires valid Google Drive OAuth2 credentials.
    - Edge cases: Permissions errors, folder not found, quota exceeded.

#### 1.6 Slack Notification on Non-Image Media

- **Overview:** Sends a Slack message notifying that the APOD is not an image and the wallpaper was skipped.
- **Nodes Involved:** Send a message (Slack)
- **Node Details:**

  - **Send a message**
    - Type: Slack
    - Role: Sends a message to a configured Slack channel.
    - Configuration:
      - Message text includes media type and APOD title.
      - Target channel ID is preconfigured.
      - Authentication via OAuth2.
    - Input: False branch from If (media type check).
    - Output: End of workflow for this branch.
    - Edge cases: Slack API rate limits, invalid channel ID, expired tokens.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                                          | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                     |
|----------------------|--------------------|----------------------------------------------------------|--------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| Daily Schedule       | Schedule Trigger    | Triggers workflow daily                                  | None                     | Workflow Configuration     | Starts the workflow automatically once every day.                                              |
| Workflow Configuration| Set                | Sets storage provider and Google Drive folder ID        | Daily Schedule           | Fetch Metadata             | Set your Google Drive Folder ID and Slack Channel ID for notifications.                         |
| Fetch Metadata       | NASA API           | Fetches APOD metadata for yesterday                      | Workflow Configuration   | If                        |                                                                                                |
| If                   | If                 | Checks if APOD media type is "image"                     | Fetch Metadata           | Download Image / Send a message | If media type is image, runs only if the daily APOD is an image.                                |
| Download Image       | HTTP Request       | Downloads APOD image file                                 | If (true branch)         | Resize Image               |                                                                                                |
| Resize Image         | Edit Image         | Resizes image to 4K (3840x2160)                          | Download Image           | Generate Filename          |                                                                                                |
| Generate Filename    | Set                | Generates standardized filename                           | Resize Image             | Check Storage Provider     |                                                                                                |
| Check Storage Provider| If                 | Checks if storage provider is Google Drive               | Generate Filename        | Upload to Google Drive     | Checks if a file with the same name already exists in your Google Drive folder.                 |
| Upload to Google Drive| Google Drive       | Uploads resized image to Google Drive                     | Check Storage Provider   | None                      |                                                                                                |
| Send a message       | Slack              | Sends Slack alert if APOD is not an image (e.g., video)  | If (false branch)        | None                      | If the APOD is a video, this node sends a notification to Slack.                               |
| Sticky Note3         | Sticky Note        | Explains the If node media type check                    | None                     | None                      | If media type is image, it runs only if the daily APOD is an image.                            |
| Sticky Note4         | Sticky Note        | Explains Workflow Configuration node                     | None                     | None                      | 1. Set your Google Drive Folder ID. 2. Set your Slack Channel ID for notifications.            |
| Sticky Note5         | Sticky Note        | General workflow overview and setup instructions         | None                     | None                      | This workflow downloads NASA's APOD daily, resizes to 4K, uploads to Drive, alerts Slack if video. |
| Sticky Note          | Sticky Note        | Explains Daily Schedule node                             | None                     | None                      | Starts the workflow automatically once every day.                                              |
| Sticky Note1         | Sticky Note        | Explains Check Storage Provider node                     | None                     | None                      | Checks if a file with the same name already exists in your Google Drive folder.                 |
| Sticky Note2         | Sticky Note        | Explains Slack notification node                         | None                     | None                      | If the APOD is a video, this node sends a notification to Slack.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Name: `Daily Schedule`
   - Configure to trigger every day (default interval).
   - Connect output to the next node.

3. **Add a Set node for configuration:**
   - Name: `Workflow Configuration`
   - Add two string fields:
     - `STORAGE_PROVIDER` = `"drive"`
     - `DRIVE_FOLDER_ID` = your Google Drive folder ID (e.g., `"1uOPQCyF_w9VX7FcgvAmBAjD3v0jl8MSU"`)
   - Connect input from `Daily Schedule` output.

4. **Add a NASA API node:**
   - Name: `Fetch Metadata`
   - Choose "Astronomy Picture of the Day" API.
   - Set "Download" to false.
   - Set `date` parameter to an expression: `{{$now.minus({ days: 1 }).toISODate()}}` (fetches yesterday's APOD).
   - Connect input from `Workflow Configuration`.

5. **Add an If node for media type check:**
   - Name: `If`
   - Condition: Check if `{{$json["media_type"]}} === "image"`.
   - Connect input from `Fetch Metadata`.

6. **Add an HTTP Request node to download the image:**
   - Name: `Download Image`
   - Method: GET
   - URL: Use expression `{{$json.hdurl || $json.url}}` to get the HD image URL or fallback.
   - Response Format: File
   - Connect from `If` true branch.

7. **Add an Edit Image node to resize the image:**
   - Name: `Resize Image`
   - Operation: Resize
   - Width: 3840
   - Height: 2160
   - Connect input from `Download Image`.

8. **Add a Set node to generate filename:**
   - Name: `Generate Filename`
   - Add a string field `fileName` with value: `={{ 'apod_' + $('Fetch Metadata').item.json.date + '.jpg' }}`
   - Include other fields (to preserve image data).
   - Connect input from `Resize Image`.

9. **Add an If node to check storage provider:**
   - Name: `Check Storage Provider`
   - Condition: Check if `{{$json.STORAGE_PROVIDER}} === "drive"`.
   - Connect input from `Generate Filename`.

10. **Add a Google Drive node to upload the image:**
    - Name: `Upload to Google Drive`
    - Operation: Upload File
    - Set file name dynamically from `fileName` field.
    - Folder ID: Use expression to get from workflow config: `{{$json.DRIVE_FOLDER_ID}}`
    - Drive ID: "My Drive"
    - Credentials: Configure Google Drive OAuth2 credentials before using.
    - Connect input from `Check Storage Provider` true branch.

11. **Add a Slack node for notifications:**
    - Name: `Send a message`
    - Message text: `"今日のNASA APODは画像ではなかったため、壁紙の取得をスキップしました。 メディアタイプ:{{ $('Fetch Metadata').item.json.media_type }} \nタイトル: {{ $('Fetch Metadata').item.json.title }}"`
    - Select Slack channel by ID (set your channel ID).
    - Authentication: Use OAuth2 Slack credentials.
    - Connect input from `If` false branch.

12. **Activate the workflow.**

13. **Credentials setup:**
    - NASA API node: configure API key if required.
    - Google Drive node: configure OAuth2 credentials with access to Drive folder.
    - Slack node: configure OAuth2 credentials with post message permissions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow automatically downloads NASA's APOD daily, resizes images to 4K resolution, uploads to Google Drive, and alerts via Slack if the media is a video. | Workflow purpose and general description.                                                                       |
| Ensure you have valid NASA API keys, Google Drive OAuth2 credentials with write permissions to your Drive folder, and Slack OAuth2 credentials with access to your notification channel. | Credential requirements.                                                                                        |
| Slack notification text is in Japanese, meaning: "Today's NASA APOD was not an image, so wallpaper retrieval was skipped. Media type: {{media_type}} Title: {{title}}" | Localization detail for Slack messages.                                                                         |
| Google Drive folder ID and Slack channel ID must be set correctly in the Workflow Configuration node for successful operation. | Critical manual configuration steps.                                                                             |
| NASA APOD API documentation: https://api.nasa.gov/                                                                            | Useful for understanding metadata fields and API usage.                                                        |
| Slack API documentation: https://api.slack.com/                                                                               | Useful for customizing Slack messages and troubleshooting.                                                     |

---

This comprehensive reference facilitates understanding, modification, and recreation of the "Create Daily 4K Wallpapers from NASA's APOD with Google Drive & Slack Alerts" workflow. It covers all nodes, their configurations, edge cases, and integration points with external services.