Convert Blog Posts to Audio Content with Eleven Labs & GPT-4

https://n8nworkflows.xyz/workflows/convert-blog-posts-to-audio-content-with-eleven-labs---gpt-4-6717


# Convert Blog Posts to Audio Content with Eleven Labs & GPT-4

### 1. Workflow Overview

This workflow automates the conversion of new blog posts into audio content, leveraging GPT-4 for metadata generation and Eleven Labs for text-to-speech synthesis. It targets content creators or marketers who want to repurpose their blog articles into audio formats for broader distribution, including email notifications or internal team alerts.

The workflow is logically organized into these main blocks:

- **1.1 Input Reception:** Detects new blog posts via an RSS feed trigger.
- **1.2 Article Processing:** Extracts and cleans article data from the RSS feed.
- **1.3 Audio Generation:** Converts cleaned article text into speech using Eleven Labs.
- **1.4 Audio Upload:** Uploads the generated audio file to Google Drive.
- **1.5 Metadata Generation:** Uses GPT-4 to create audio metadata (title, description, tags).
- **1.6 Metadata Combination:** Consolidates all metadata and relevant URLs.
- **1.7 Logging:** Logs audio production details to Google Sheets.
- **1.8 Conditional Notification:** Determines whether to send email notifications or Slack messages based on workflow conditions.
- **1.9 Notification Delivery:** Sends emails or Slack notifications accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Detects new blog posts as they appear in a specified RSS feed, triggering the workflow to start processing.

- **Nodes Involved:**  
  - RSS Feed Trigger

- **Node Details:**

  - **RSS Feed Trigger**  
    - Type: Trigger node (RSS Feed Read Trigger)  
    - Configuration: Default RSS feed polling to detect new articles. No custom parameters visible; likely configured to a specific blog RSS URL in credentials or external setup.  
    - Inputs: None (trigger node).  
    - Outputs: Emits new RSS feed items as JSON objects to downstream nodes.  
    - Edge Cases: RSS feed downtime, invalid RSS format, or no new entries could cause the trigger to fail or not activate.  
    - Version: 1  
    - Sub-workflow: None

#### 2.2 Article Processing

- **Overview:**  
  Extracts relevant article content from the RSS feed item and cleans it for further processing.

- **Nodes Involved:**  
  - Extract & Clean Article Data

- **Node Details:**

  - **Extract & Clean Article Data**  
    - Type: Set node (data transformation)  
    - Configuration: Likely extracts key fields such as article title, URL, and main text. Cleans or formats text to remove unwanted characters or HTML tags.  
    - Inputs: Connected from RSS Feed Trigger.  
    - Outputs: Passes cleaned article data forward.  
    - Expressions: Uses JSONPath or expressions to select and transform fields from the RSS item.  
    - Edge Cases: Missing article content, malformed data, or encoding issues could cause incomplete or incorrect extraction.  
    - Version: 3.4  
    - Sub-workflow: None

#### 2.3 Audio Generation

- **Overview:**  
  Converts the cleaned article text into audio using Eleven Labs’ text-to-speech service.

- **Nodes Involved:**  
  - Convert Text to Speech

- **Node Details:**

  - **Convert Text to Speech**  
    - Type: Eleven Labs node for TTS  
    - Configuration: Uses Eleven Labs API credentials. Input text is the cleaned article content from the previous node. Generates an audio file (likely MP3 or WAV).  
    - Inputs: From Extract & Clean Article Data.  
    - Outputs: Audio binary data sent to Upload Audio File node.  
    - Edge Cases: API authentication errors, text length limits, network timeouts, or service unavailability.  
    - Retry: Disabled (retryOnFail=false).  
    - Version: 1  
    - Sub-workflow: None

#### 2.4 Audio Upload

- **Overview:**  
  Uploads the generated audio file to Google Drive for storage and access.

- **Nodes Involved:**  
  - Upload Audio File

- **Node Details:**

  - **Upload Audio File**  
    - Type: Google Drive node (file upload)  
    - Configuration: Uses Google Drive OAuth2 credentials. Uploads the audio binary received from the TTS node. Sets filename and folder destination (not detailed in JSON).  
    - Inputs: Audio binary from Convert Text to Speech.  
    - Outputs: Returns file metadata including the Google Drive URL.  
    - Edge Cases: Authentication token expiry, quota limits, file size limits, and network errors.  
    - Version: 3  
    - Sub-workflow: None

#### 2.5 Metadata Generation

- **Overview:**  
  Generates descriptive metadata for the audio content (title, description, tags) using the OpenAI GPT-4 model.

- **Nodes Involved:**  
  - Generate Metadata

- **Node Details:**

  - **Generate Metadata**  
    - Type: LangChain OpenAI node  
    - Configuration: Calls GPT-4 model with prompts likely built from article content or audio file info to generate metadata. Uses OpenAI credentials.  
    - Inputs: Receives Google Drive upload metadata.  
    - Outputs: Structured metadata for the audio file.  
    - Edge Cases: API quota exceeded, authentication errors, prompt or response parsing errors, or timeout.  
    - Version: 1.8  
    - Sub-workflow: None

#### 2.6 Metadata Combination

- **Overview:**  
  Combines all metadata and audio file information into a single JSON object for logging and notifications.

- **Nodes Involved:**  
  - Combine All Metadata

- **Node Details:**

  - **Combine All Metadata**  
    - Type: Set node  
    - Configuration: Aggregates fields such as final article URL, title, audio file URL, generated audio title, description, tags, status, and timestamp. Possibly formats or renames fields.  
    - Inputs: From Generate Metadata.  
    - Outputs: Passes combined data to logging node.  
    - Expressions: Uses expressions to merge data fields.  
    - Edge Cases: Missing fields or inconsistent data could cause incomplete entries.  
    - Version: 3.4  
    - Sub-workflow: None

#### 2.7 Logging

- **Overview:**  
  Records the audio production process details into a Google Sheets spreadsheet for tracking and auditing.

- **Nodes Involved:**  
  - Log Audio Production

- **Node Details:**

  - **Log Audio Production**  
    - Type: Google Sheets node  
    - Configuration: Writes a row with the following columns and values (using expressions):  
      - Article URL: `={{ $json.finalArticleUrl }}`  
      - Article Title: `={{ $json.finalArticleTitle }}`  
      - Audio File URL: `={{ $json.audioFileUrl }}`  
      - Generated Audio Title: `={{ $json.generatedAudioTitle }}`  
      - Generated Audio Description: `={{ $json.generatedAudioDescription }}`  
      - Generated Audio Tags: `={{ $json.generatedAudioTags }}`  
      - Status: `={{ $json.workflowStatus }}`  
      - Timestamp: `={{ $json.timestamp }}`  
    - Inputs: Combined metadata from previous node.  
    - Outputs: Forwards data to conditional notification node.  
    - Edge Cases: API limits, sheet access permissions, or data format issues.  
    - Version: 4.6  
    - Sub-workflow: None

#### 2.8 Conditional Notification

- **Overview:**  
  Decides whether the audio content notification should be sent via email or Slack based on a condition.

- **Nodes Involved:**  
  - Is for Email Distribution?

- **Node Details:**

  - **Is for Email Distribution?**  
    - Type: If node (conditional branching)  
    - Configuration: Evaluates a condition (not explicitly shown) likely based on metadata or workflow parameters to determine notification channel.  
    - Inputs: From Log Audio Production.  
    - Outputs: Two branches:  
      - True branch → Send Audio Notification via Email  
      - False branch → Notify Internal Team via Slack  
    - Edge Cases: Condition expression failure or unexpected input data.  
    - Version: 2.2  
    - Sub-workflow: None

#### 2.9 Notification Delivery

- **Overview:**  
  Sends notifications about the new audio content either by email or Slack to appropriate recipients.

- **Nodes Involved:**  
  - Send Audio Notification via Email  
  - Notify Internal Team

- **Node Details:**

  - **Send Audio Notification via Email**  
    - Type: Gmail node  
    - Configuration: Sends an email notification with audio file details to external recipients. Uses Gmail OAuth2 credentials. Webhook ID present indicates it may be externally triggered or monitored.  
    - Inputs: From True branch of If node.  
    - Outputs: None (end of branch).  
    - Edge Cases: Email quota limits, auth token expiry, invalid email addresses, or message formatting issues.  
    - Version: 2.1  
    - Webhook ID: 41e96a51-49f9-494a-849c-d1302e137b18  
    - Sub-workflow: None

  - **Notify Internal Team**  
    - Type: Slack node  
    - Configuration: Sends a Slack message to internal team channel notifying about the audio content. Uses Slack webhook credentials.  
    - Inputs: From False branch of If node.  
    - Outputs: None (end of branch).  
    - Edge Cases: Webhook URL invalid, Slack API limits, network errors.  
    - Version: 2.3  
    - Webhook ID: e549edc5-add6-437f-9a8b-98ebb772aad7  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name                     | Node Type                             | Functional Role                      | Input Node(s)                | Output Node(s)                  | Sticky Note                                  |
|-------------------------------|-------------------------------------|------------------------------------|-----------------------------|--------------------------------|----------------------------------------------|
| RSS Feed Trigger              | RSS Feed Read Trigger                | Detect new blog posts via RSS feed | None                        | Extract & Clean Article Data    |                                              |
| Extract & Clean Article Data  | Set                                 | Extract and clean article content  | RSS Feed Trigger            | Convert Text to Speech          |                                              |
| Convert Text to Speech        | Eleven Labs TTS                     | Convert article text to audio      | Extract & Clean Article Data | Upload Audio File              |                                              |
| Upload Audio File             | Google Drive                        | Upload audio file to Google Drive  | Convert Text to Speech       | Generate Metadata              |                                              |
| Generate Metadata             | OpenAI (LangChain)                  | Generate audio metadata with GPT-4 | Upload Audio File           | Combine All Metadata           |                                              |
| Combine All Metadata          | Set                                 | Aggregate metadata and URLs        | Generate Metadata           | Log Audio Production           |                                              |
| Log Audio Production          | Google Sheets                      | Log production details             | Combine All Metadata        | Is for Email Distribution?      | Values mapped include article and audio info |
| Is for Email Distribution?   | If                                  | Conditional notification routing   | Log Audio Production        | Send Audio Notification via Email, Notify Internal Team |                                              |
| Send Audio Notification via Email | Gmail                             | Send email notification            | Is for Email Distribution?  | None                          |                                              |
| Notify Internal Team          | Slack                              | Notify internal team via Slack     | Is for Email Distribution?  | None                          |                                              |
| Sticky Note                  | Sticky Note                        | Non-functional annotation          | None                        | None                          |                                              |
| Sticky Note1                 | Sticky Note                        | Non-functional annotation          | None                        | None                          |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger Node**  
   - Type: RSS Feed Read Trigger  
   - Configure RSS URL to the blog’s RSS feed endpoint.  
   - Leave other settings default or customize polling intervals.  

2. **Create Extract & Clean Article Data Node**  
   - Type: Set node  
   - Connect input from RSS Feed Trigger.  
   - Configure fields to extract article title, URL, and main content from RSS item JSON.  
   - Add expressions to clean text content (e.g., strip HTML tags).  

3. **Create Convert Text to Speech Node (Eleven Labs)**  
   - Type: Eleven Labs TTS node  
   - Connect input from Extract & Clean Article Data.  
   - Add Eleven Labs credentials with API key.  
   - Map cleaned article text as input for TTS conversion.  
   - Configure voice settings as desired (voice selection, speed, etc.).  

4. **Create Upload Audio File Node (Google Drive)**  
   - Type: Google Drive node  
   - Connect input from Convert Text to Speech node.  
   - Configure Google Drive OAuth2 credentials.  
   - Set target folder and file name dynamically (e.g., based on article title).  
   - Enable upload of binary audio data.  

5. **Create Generate Metadata Node (OpenAI GPT-4)**  
   - Type: LangChain OpenAI node  
   - Connect input from Upload Audio File node.  
   - Configure OpenAI credentials with GPT-4 model.  
   - Set prompt to generate audio metadata (title, description, tags) from article and audio info.  
   - Map inputs accordingly.  

6. **Create Combine All Metadata Node**  
   - Type: Set node  
   - Connect input from Generate Metadata node.  
   - Aggregate fields: article URL, article title, audio file URL, generated audio title, description, tags, status, timestamp.  
   - Use expressions to merge data into one JSON object.  

7. **Create Log Audio Production Node (Google Sheets)**  
   - Type: Google Sheets node  
   - Connect input from Combine All Metadata node.  
   - Configure Google Sheets OAuth2 credentials.  
   - Specify spreadsheet and sheet name.  
   - Map columns to workflow data using expressions (as in the Summary Table).  

8. **Create Conditional Node: Is for Email Distribution?**  
   - Type: If node  
   - Connect input from Log Audio Production node.  
   - Set condition (e.g., a boolean flag or metadata field) to decide if email notification is needed.  

9. **Create Send Audio Notification via Email Node (Gmail)**  
   - Type: Gmail node  
   - Connect input from True branch of If node.  
   - Configure Gmail OAuth2 credentials.  
   - Compose email body using expressions referencing audio metadata and URLs.  
   - Set recipient addresses.  

10. **Create Notify Internal Team Node (Slack)**  
    - Type: Slack node  
    - Connect input from False branch of If node.  
    - Configure Slack webhook or OAuth2 credentials.  
    - Compose message referencing audio metadata and URLs.  

11. **(Optional) Add Sticky Note Nodes**  
    - For documentation or annotation within the workflow canvas.  

12. **Connect all nodes sequentially as described:**  
    - RSS Feed Trigger → Extract & Clean Article Data → Convert Text to Speech → Upload Audio File → Generate Metadata → Combine All Metadata → Log Audio Production → Is for Email Distribution? → (Send Audio Notification via Email / Notify Internal Team)  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                        |
|-----------------------------------------------------------------------------------------------------|-------------------------------------|
| Uses Eleven Labs for high-quality text-to-speech synthesis, enabling natural audio output.          | https://elevenlabs.io               |
| GPT-4 integration via LangChain for advanced metadata generation, improving content discoverability.| https://openai.com                  |
| Google Drive and Google Sheets nodes require OAuth2 credentials with appropriate scopes enabled.    | https://console.cloud.google.com    |
| Email notifications use Gmail node with OAuth2; ensure account has API access enabled.               | https://developers.google.com/gmail |
| Slack notifications require incoming webhook URL or OAuth2 credentials for message posting.         | https://api.slack.com/messaging/webhooks |
| Workflow designed for automation of blog to audio repurposing with scalable notification options.   |                                     |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.