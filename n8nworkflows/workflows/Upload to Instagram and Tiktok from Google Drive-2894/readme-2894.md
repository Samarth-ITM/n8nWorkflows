Upload to Instagram and Tiktok from Google Drive

https://n8nworkflows.xyz/workflows/upload-to-instagram-and-tiktok-from-google-drive-2894


# Upload to Instagram and Tiktok from Google Drive

### 1. Workflow Overview

This workflow automates the process of uploading videos from a specific Google Drive folder to multiple social media platforms—Instagram, TikTok, and YouTube—while generating engaging video descriptions using OpenAI. It is designed for content creators, digital marketers, and social media managers who want to streamline video posting and maintain consistent multi-platform presence with minimal manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Download:** Detects new video files in a designated Google Drive folder and downloads them.
- **1.2 Audio Extraction and Description Generation:** Extracts audio from the downloaded video and uses OpenAI to transcribe and generate a social media description.
- **1.3 Video Upload to Social Platforms:** Uploads the video along with the generated description to TikTok, Instagram, and YouTube via the upload-post.com API.
- **1.4 Error Handling and Notifications:** Monitors for errors during execution and sends Telegram notifications if issues occur.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Download

**Overview:**  
This block monitors a specific Google Drive folder for new video files, then downloads the detected video file for further processing.

**Nodes Involved:**  
- Google Drive Trigger  
- Google Drive  
- Read video from Google Drive (write binary file)

**Node Details:**

- **Google Drive Trigger**  
  - *Type:* Trigger node for Google Drive  
  - *Role:* Watches a specific Google Drive folder (`folderToWatch` ID: `18m0i341QLQuyWuHv_FBdz8-r-QDtofYm`) for new files created.  
  - *Configuration:* Polls every minute; triggers on new file creation only in the specified folder.  
  - *Input/Output:* No input; outputs metadata of new file detected.  
  - *Edge Cases:* Possible OAuth token expiration; folder permission issues; API rate limits.

- **Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the video file identified by the trigger.  
  - *Configuration:* Downloads file using OAuth2 credentials; file ID dynamically taken from trigger output (`{{$json.id || $json.data[0].id}}`).  
  - *Input/Output:* Input from Google Drive Trigger; outputs binary video data and metadata.  
  - *Edge Cases:* Download failures, file not found, permission errors; retry enabled with 5-second wait.

- **Read video from Google Drive**  
  - *Type:* Write Binary File node  
  - *Role:* Saves the downloaded video file locally with filename sanitized (spaces replaced by underscores).  
  - *Configuration:* Filename derived from original filename metadata.  
  - *Input/Output:* Input from Google Drive node; outputs file path for subsequent nodes.  
  - *Edge Cases:* Disk write permission issues; filename conflicts.

---

#### 2.2 Audio Extraction and Description Generation

**Overview:**  
Extracts audio from the downloaded video and uses OpenAI to transcribe the audio and generate a social media description tailored for Instagram and TikTok.

**Nodes Involved:**  
- Get Audio from Video (OpenAI Langchain node)  
- Generate Description for Videos in Tiktok and Instagram (OpenAI Langchain node)

**Node Details:**

- **Get Audio from Video**  
  - *Type:* OpenAI Langchain node (audio resource)  
  - *Role:* Transcribes audio from the video binary data to text.  
  - *Configuration:* Uses OpenAI API key; operation set to "transcribe".  
  - *Input/Output:* Input from "Read video from Google Drive" node (binary video); outputs transcribed text.  
  - *Edge Cases:* API rate limits; transcription errors; unsupported audio formats; retry enabled with 5-second wait.

- **Generate Description for Videos in Tiktok and Instagram**  
  - *Type:* OpenAI Langchain node (chat completion)  
  - *Role:* Generates an engaging social media description based on the transcribed audio text.  
  - *Configuration:* Uses GPT-4o model; system prompt sets role as expert assistant for social media titles; user prompt includes examples and inserts transcribed audio dynamically (`{{ $('Get Audio from Video').item.json.text }}`).  
  - *Input/Output:* Input from "Get Audio from Video"; outputs generated description text.  
  - *Edge Cases:* API errors; prompt formatting issues; empty or inaccurate transcription input; retry enabled with 5-second wait.

---

#### 2.3 Video Upload to Social Platforms

**Overview:**  
Uploads the video and generated description to TikTok, Instagram, and YouTube using the upload-post.com API.

**Nodes Involved:**  
- Read Video from Google Drive (three instances)  
- Upload Video and Description to Tiktok (HTTP Request)  
- Upload Video and Description to Instagram (HTTP Request)  
- Upload Video and Description to Youtube (HTTP Request)

**Node Details:**

- **Read Video from Google Drive (3 nodes)**  
  - *Type:* Read Binary File nodes  
  - *Role:* Reads the locally saved video file three times independently to provide binary data for each platform upload.  
  - *Configuration:* File path derived from sanitized filename; data property named `datavideo`.  
  - *Input/Output:* Input from "Generate Description for Videos in Tiktok and Instagram" node; outputs binary video data for upload.  
  - *Edge Cases:* File not found; read permission errors.

- **Upload Video and Description to Tiktok**  
  - *Type:* HTTP Request node  
  - *Role:* Uploads video and description to TikTok via upload-post.com API.  
  - *Configuration:*  
    - URL: `https://api.upload-post.com/api/upload`  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Authentication: HTTP Header with API key from upload-post.com  
    - Body parameters:  
      - `title`: Description text from OpenAI output, sanitized to remove quotes  
      - `platform[]`: "tiktok"  
      - `video`: Binary video data from "Read Video from Google Drive"  
      - `user`: User identifier for upload-post.com (to be customized)  
  - *Input/Output:* Input from "Read Video from Google Drive" node; outputs API response.  
  - *Edge Cases:* API authentication errors; upload failures; network timeouts.

- **Upload Video and Description to Instagram**  
  - *Type:* HTTP Request node  
  - *Role:* Uploads video and description to Instagram via upload-post.com API.  
  - *Configuration:* Similar to TikTok upload node, with `platform[]` set to "instagram".  
  - *Input/Output:* Input from second "Read Video from Google Drive" node; outputs API response.  
  - *Edge Cases:* Same as TikTok upload node.

- **Upload Video and Description to Youtube**  
  - *Type:* HTTP Request node  
  - *Role:* Uploads video and description to YouTube via upload-post.com API.  
  - *Configuration:* Similar to other upload nodes, with `platform[]` set to "youtube".  
    - The description is truncated to 70 characters for YouTube title.  
    - User parameter set to "test2" (to be customized).  
  - *Input/Output:* Input from third "Read Video from Google Drive" node; outputs API response.  
  - *Edge Cases:* Same as other upload nodes.

---

#### 2.4 Error Handling and Notifications

**Overview:**  
Monitors for workflow errors and sends Telegram notifications if an error occurs, excluding DNS server offline errors.

**Nodes Involved:**  
- Error Trigger  
- If (conditional)  
- Telegram

**Node Details:**

- **Error Trigger**  
  - *Type:* Error Trigger node  
  - *Role:* Captures any error occurring in the workflow execution.  
  - *Input/Output:* No input; triggers on error events.  
  - *Edge Cases:* May not catch errors outside n8n environment.

- **If**  
  - *Type:* Conditional node  
  - *Role:* Filters errors to exclude those containing the message "The DNS server returned an error, perhaps the server is offline".  
  - *Configuration:* Checks if error message does NOT contain the DNS error string.  
  - *Input/Output:* Input from Error Trigger; outputs to Telegram node if condition met.  
  - *Edge Cases:* Expression evaluation errors.

- **Telegram**  
  - *Type:* Telegram node  
  - *Role:* Sends an alert message "🔔 ERROR SUBIENDO VIDEOS" to a configured Telegram chat.  
  - *Configuration:* Uses Telegram bot credentials; disables attribution; retries on failure with 5-second wait.  
  - *Input/Output:* Input from If node; no output.  
  - *Edge Cases:* Telegram API errors; bot token invalid; network issues.

---

### 3. Summary Table

| Node Name                                 | Node Type                     | Functional Role                              | Input Node(s)                      | Output Node(s)                                         | Sticky Note                                                                                                  |
|-------------------------------------------|-------------------------------|----------------------------------------------|----------------------------------|-------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger                      | Google Drive Trigger           | Detect new video files in Google Drive folder | None                             | Google Drive                                          |                                                                                                              |
| Google Drive                             | Google Drive                  | Download detected video file                   | Google Drive Trigger             | Read video from Google Drive                           |                                                                                                              |
| Read video from Google Drive              | Write Binary File             | Save downloaded video locally                   | Google Drive                    | Get Audio from Video                                  |                                                                                                              |
| Get Audio from Video                      | OpenAI Langchain (audio)      | Transcribe audio from video                     | Read video from Google Drive    | Generate Description for Videos in Tiktok and Instagram | Extract the audio from video for generate the description                                                    |
| Generate Description for Videos in Tiktok and Instagram | OpenAI Langchain (chat)       | Generate social media description from transcription | Get Audio from Video            | Read Video from Google Drive (3 nodes)                | Request to OpenAi for generate description with the audio extracted from the video                            |
| Read Video from Google Drive              | Read Binary File              | Read saved video file for TikTok upload         | Generate Description for Videos | Upload Video and Description to Tiktok                |                                                                                                              |
| Read Video from Google Drive2             | Read Binary File              | Read saved video file for Instagram upload      | Generate Description for Videos | Upload Video and Description to Instagram             |                                                                                                              |
| Read Video from Google Drive3             | Read Binary File              | Read saved video file for YouTube upload        | Generate Description for Videos | Upload Video and Description to Youtube               |                                                                                                              |
| Upload Video and Description to Tiktok    | HTTP Request                 | Upload video and description to TikTok          | Read Video from Google Drive    | None                                                  | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here) |
| Upload Video and Description to Instagram | HTTP Request                 | Upload video and description to Instagram       | Read Video from Google Drive2   | None                                                  | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here) |
| Upload Video and Description to Youtube   | HTTP Request                 | Upload video and description to YouTube         | Read Video from Google Drive3   | None                                                  | Generate in upload-post.com the token and add to the credentials in the header-> Authorization: Apikey (token here) |
| Error Trigger                            | Error Trigger                | Capture workflow errors                          | None                           | If                                                    |                                                                                                              |
| If                                       | If                          | Filter errors to exclude DNS server offline errors | Error Trigger                  | Telegram                                              |                                                                                                              |
| Telegram                                 | Telegram                    | Send error notification to Telegram chat        | If                             | None                                                  |                                                                                                              |
| Sticky Note                              | Sticky Note                 | Workflow description and setup instructions     | None                           | None                                                  | ## Description This automation allows you to upload a video to a configured Google Drive folder, and it will automatically create descriptions and upload it to Instagram and TikTok. ## How to Use 1. Generate an API token at upload-post.com and add to Upload to Tiktok and Upload to Instagram nodes 2. Configure your Google Drive folder 3. Customize the OpenAI prompt for your specific use case 4. Optional: Configure Telegram for error notifications ## Requirements - upload-post.com account - Google Drive account - OpenAI API key |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Set event to `fileCreated`  
   - Set polling interval to every minute  
   - Configure to watch a specific folder by folder ID (e.g., `18m0i341QLQuyWuHv_FBdz8-r-QDtofYm`)  
   - Connect Google Drive OAuth2 credentials

2. **Create Google Drive node**  
   - Type: Google Drive  
   - Operation: `download`  
   - File ID: Expression `={{ $json.id || $json.data[0].id }}` from trigger output  
   - Connect Google Drive OAuth2 credentials  
   - Connect output of Google Drive Trigger to this node

3. **Create Write Binary File node ("Read video from Google Drive")**  
   - Type: Write Binary File  
   - Filename: Expression `={{ $json.originalFilename.replaceAll(" ", "_") }}`  
   - Connect output of Google Drive node to this node

4. **Create OpenAI Langchain node ("Get Audio from Video")**  
   - Type: OpenAI Langchain (audio resource)  
   - Operation: `transcribe`  
   - Connect OpenAI API credentials  
   - Connect output of Write Binary File node to this node

5. **Create OpenAI Langchain node ("Generate Description for Videos in Tiktok and Instagram")**  
   - Type: OpenAI Langchain (chat completion)  
   - Model: GPT-4o (or equivalent)  
   - Messages:  
     - System role: "You are an expert assistant in creating engaging social media video titles."  
     - User content: Provide examples of Instagram descriptions and include dynamic insertion of transcribed audio text with expression: `{{ $('Get Audio from Video').item.json.text }}`  
     - Instruction: Reply only with the description text  
   - Connect OpenAI API credentials  
   - Connect output of "Get Audio from Video" node to this node

6. **Create three Read Binary File nodes for video reuse**  
   - Type: Read Binary File  
   - Filename: Expression `={{ $('Read video from Google Drive').item.json.originalFilename.replaceAll(" ", "_") }}`  
   - Data property name: `datavideo`  
   - Connect output of "Generate Description for Videos in Tiktok and Instagram" node to each of these three nodes separately

7. **Create HTTP Request node ("Upload Video and Description to Tiktok")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.upload-post.com/api/upload`  
   - Content-Type: multipart/form-data  
   - Authentication: HTTP Header Auth with API key from upload-post.com  
   - Body parameters:  
     - `title`: Expression `={{ $('Generate Description for Videos  in Tiktok and Instagram').item.json.message.content.replaceAll("\"", "") }}`  
     - `platform[]`: "tiktok"  
     - `video`: Binary data from corresponding Read Binary File node (`datavideo`)  
     - `user`: Your upload-post.com user identifier  
   - Connect output of first Read Binary File node to this node

8. **Create HTTP Request node ("Upload Video and Description to Instagram")**  
   - Same as TikTok node but with `platform[]` set to "instagram"  
   - Connect output of second Read Binary File node to this node

9. **Create HTTP Request node ("Upload Video and Description to Youtube")**  
   - Same as above but with `platform[]` set to "youtube"  
   - `title` parameter truncated to first 70 characters of description:  
     `={{ $('Generate Description for Videos  in Tiktok and Instagram').item.json.message.content.replaceAll("\"", "").substring(0, 70) }}`  
   - Connect output of third Read Binary File node to this node

10. **Create Error Trigger node**  
    - Type: Error Trigger  
    - No configuration needed

11. **Create If node**  
    - Type: If  
    - Condition: Check if error message does NOT contain `"The DNS server returned an error, perhaps the server is offline"`  
    - Connect Error Trigger output to this node

12. **Create Telegram node**  
    - Type: Telegram  
    - Text: `"🔔 ERROR SUBIENDO VIDEOS"`  
    - Disable attribution  
    - Connect Telegram credentials (bot token)  
    - Connect output of If node to this node

13. **Add Sticky Note node** (optional)  
    - Add workflow description, setup instructions, and requirements as content for user reference

14. **Connect all nodes as per the described flow**  
    - Google Drive Trigger → Google Drive → Write Binary File → Get Audio from Video → Generate Description → (three Read Binary File nodes) → (three HTTP Request upload nodes)  
    - Error Trigger → If → Telegram

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Generate API token at [upload-post.com](https://upload-post.com) and configure it in Upload to TikTok and Instagram nodes | Upload-post.com API token setup                                                                 |
| Create Google Cloud Project, enable Google Drive API, and generate OAuth credentials for Google Drive node           | Google Cloud Platform setup for Google Drive OAuth2                                             |
| Customize OpenAI prompt in "Generate Description for Videos in Tiktok and Instagram" node to match your brand tone    | OpenAI prompt customization                                                                     |
| Optional: Configure Telegram bot token for error notifications                                                      | Telegram bot setup for workflow error alerts                                                    |
| This workflow supports uploading to YouTube in addition to TikTok and Instagram                                     | YouTube upload included via upload-post.com API                                                |
| Retry mechanisms enabled on critical nodes (Google Drive download, OpenAI calls, Telegram) with 5-second intervals  | Improves robustness against transient errors                                                   |

---

This documentation provides a complete understanding of the workflow structure, node configurations, and operational logic, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.