Automated Video Translation & Distribution with DubLab to Multiple Platforms

https://n8nworkflows.xyz/workflows/automated-video-translation---distribution-with-dublab-to-multiple-platforms-4751


# Automated Video Translation & Distribution with DubLab to Multiple Platforms

### 1. Workflow Overview

This workflow automates video translation and dubbing using the DubLab API, then distributes the dubbed videos to multiple platforms including Telegram, Box, Dropbox, YouTube, and Postiz. It is designed for content creators or social media managers who want to easily convert videos into different languages and share them across various social channels without manual intervention.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Preparation:** Receive videos and selected target languages from a form input, then prepare combinations of videos and languages for dubbing.
- **1.2 DubLab Dubbing Initialization and Upload:** Initialize a dubbing project on DubLab, upload the source video, and start the dubbing process.
- **1.3 Monitoring and Fetching Dubbed Videos:** After dubbing starts, monitor progress and fetch the dubbed video once ready.
- **1.4 Distribution to Platforms:** Distribute the dubbed video to Telegram, Box, Dropbox, YouTube, and Postiz platforms.
- **1.5 Webhook and Original Video Handling:** Receive webhook callbacks from DubLab for dubbed video availability and optionally send original video to Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

- **Overview:**  
Receives user input via a form for uploading video files and selecting multiple destination languages. Then it combines each video with each selected language into individual processing items with proper language codes.

- **Nodes Involved:**  
  - Start (Form Trigger)  
  - Combine Videos and Languages (Code)  
  - Proxy Videos (Merge)  

- **Node Details:**  

  - **Start (Form Trigger)**  
    - *Type:* Form Trigger  
    - *Role:* Entry point; collects video files and destination languages via a form.  
    - *Config:* Accepts mp4 files and multi-select dropdown for languages (English, German, Russian, Polish, Spanish, Portuguese, Italian, Arabic, French, Turkish, Dutch).  
    - *Connections:* Outputs to "Combine Videos and Languages".  
    - *Failures:* User submits invalid file type or no languages selected.  
    - *Notes:* Form-based input enables easy user-driven processing.

  - **Combine Videos and Languages (Code)**  
    - *Type:* Function/Code Node  
    - *Role:* Creates distinct processing items for each video-language pair, mapping language names to ISO codes.  
    - *Config:*  
      - Maps language names to codes (`en`, `de`, `ru`, etc.).  
      - Iterates over all input items, combines videos and languages for parallel processing.  
      - Assigns `binary_order` to track video binaries.  
    - *Expressions:* Uses `$input.all()` and accesses both JSON and binary data.  
    - *Connections:* Outputs to "Init Dubbing" and "Proxy Videos".  
    - *Failures:* Undefined languages not in map fallback gracefully; missing binaries could cause errors.  

  - **Proxy Videos (Merge)**  
    - *Type:* Merge Node  
    - *Role:* Combines outputs from different nodes for aligned processing.  
    - *Config:* Combines inputs by position (array index).  
    - *Connections:* Inputs from "Combine Videos and Languages", outputs to "Upload Video" and "Proxy Ids".  
    - *Failures:* Misalignment of inputs if counts differ.

#### 2.2 DubLab Dubbing Initialization and Upload

- **Overview:**  
Initializes a dubbing project via DubLab API, uploads the raw video to a pre-signed URL, then triggers the dubbing process.

- **Nodes Involved:**  
  - Init Dubbing (HTTP Request)  
  - Upload Video (HTTP Request)  
  - Proxy Ids (Merge)  
  - Start Dubbing (HTTP Request)  

- **Node Details:**  

  - **Init Dubbing (HTTP Request)**  
    - *Type:* HTTP Request  
    - *Role:* Calls DubLab `/v1/init-dub` endpoint to initiate dubbing session.  
    - *Config:*  
      - POST request with JSON body containing destination language, video duration (fixed 30s), file type, filename, and source language ("auto").  
      - Uses `ApiKey` header for authentication (environment variable `DUBLAB_API_KEY`).  
      - Dynamically sets language and filename from input JSON.  
    - *Connections:* Outputs to "Proxy Videos".  
    - *Failures:* API key errors, invalid parameters, network timeouts.  

  - **Upload Video (HTTP Request)**  
    - *Type:* HTTP Request  
    - *Role:* Uploads the video binary to DubLab's pre-signed URL.  
    - *Config:*  
      - PUT request to URL from `upload_url` field in previous response.  
      - Sends binary video data using dynamic binary field name (`VIDEO_x`).  
      - Sets `Content-Type` header to `video/mp4`.  
    - *Connections:* Outputs to "Proxy Ids".  
    - *Failures:* Incorrect binary reference, upload failures, URL expiration.  

  - **Proxy Ids (Merge)**  
    - *Type:* Merge Node  
    - *Role:* Combines outputs from "Init Dubbing" and "Upload Video" for synchronized processing.  
    - *Connections:* Outputs to "Start Dubbing".  
    - *Failures:* If counts mismatch or data missing, downstream nodes may fail.  

  - **Start Dubbing (HTTP Request)**  
    - *Type:* HTTP Request  
    - *Role:* Sends request to DubLab `/v1/start-dub` to start the dubbing process using the dubbing project ID.  
    - *Config:*  
      - POST with JSON body containing `id` received from previous steps.  
      - Uses `ApiKey` header for authentication.  
    - *Connections:* Outputs to "Fetch Projects".  
    - *Failures:* Invalid ID, API failures, network issues.

#### 2.3 Monitoring and Fetching Dubbed Videos

- **Overview:**  
Polls DubLab for the status of dubbing projects and fetches the dubbed video and original video URLs when ready.

- **Nodes Involved:**  
  - Fetch Projects (HTTP Request)  
  - Webhook (Webhook)  
  - Dubbed Video (HTTP Request)  
  - Original Video (HTTP Request)  

- **Node Details:**  

  - **Fetch Projects (HTTP Request)**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves list of dubbing projects from DubLab `/v1/dubs`.  
    - *Config:* GET request with `ApiKey` header.  
    - *Connections:* No downstream connections (end of polling chain).  
    - *Failures:* API key errors, network timeouts.  

  - **Webhook (Webhook)**  
    - *Type:* Webhook Node  
    - *Role:* Receives asynchronous callback notifications from DubLab when dubbing finishes.  
    - *Config:* POST webhook listening on path `e41c5f62-f2de-4df0-ac29-851837a67282`.  
    - *Connections:* Triggers "Dubbed Video" and "Original Video" HTTP requests.  
    - *Failures:* Missing or malformed webhook payload; security considerations for webhook URL exposure.  

  - **Dubbed Video (HTTP Request)**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the dubbed video binary from the URL specified in webhook payload.  
    - *Config:* GET request to `dubbed_download_url`.  
    - *Connections:* Outputs to all distribution nodes (Telegram, Box, Dropbox, YouTube, Postiz).  
    - *Failures:* 404 if file not yet available; network issues.  

  - **Original Video (HTTP Request)**  
    - *Type:* HTTP Request  
    - *Role:* Optionally downloads the original video binary from the webhook payload URL.  
    - *Config:* GET request to `original_download_url`.  
    - *Connections:* Outputs to Telegram node (for possible sharing).  
    - *Failures:* Similar to dubbed video.

#### 2.4 Distribution to Platforms

- **Overview:**  
Uploads or sends the dubbed video to multiple platforms for sharing and distribution.

- **Nodes Involved:**  
  - Telegram (Telegram Node)  
  - Box (Box Node)  
  - Dropbox (Dropbox Node)  
  - YouTube (YouTube Node)  
  - Postiz (HTTP Request)  

- **Node Details:**  

  - **Telegram (Telegram Node)**  
    - *Type:* Telegram Node  
    - *Role:* Sends the dubbed video as a video message to a Telegram chat.  
    - *Config:*  
      - Uses chat ID `145187085`.  
      - Sends video as binary data with filename "Dubbed Video".  
      - Requires Telegram API credentials configured with OAuth token.  
    - *Failures:* Invalid chat ID, token expiration, Telegram API limits.  

  - **Box (Box Node)**  
    - *Type:* Box Node  
    - *Role:* Uploads the dubbed video binary to Box cloud storage.  
    - *Config:*  
      - Filename generated randomly with destination language and original filename suffix.  
      - OAuth2 credentials required.  
    - *Failures:* OAuth token expiry, quota limits.  

  - **Dropbox (Dropbox Node)**  
    - *Type:* Dropbox Node  
    - *Role:* Uploads dubbed video to Dropbox folder `/dublab-files/` with a randomized filename.  
    - *Config:* OAuth2 authentication.  
    - *Failures:* Token issues, API rate limits.  

  - **YouTube (YouTube Node)**  
    - *Type:* YouTube Node  
    - *Role:* Uploads dubbed video to YouTube as a new video.  
    - *Config:*  
      - Title set to original filename.  
      - Region code set to "PL" (Poland).  
      - OAuth2 credentials for YouTube API.  
    - *Failures:* Quota exceeded, upload failures.  

  - **Postiz (HTTP Request)**  
    - *Type:* HTTP Request  
    - *Role:* Uploads the video to Postiz platform supporting multiple social media distributions.  
    - *Config:*  
      - POST multipart/form-data with binary video as `file` field.  
      - Headers include `Authorization` and `Proxy-Authorization` keys (environment variables).  
    - *Failures:* Authentication failures, proxy issues, file size limits.  

#### 2.5 Sticky Notes and Documentation Nodes

- **Overview:**  
Sticky notes provide workflow documentation, instructions, and external resource links.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  

- **Node Details:**  

  - **Sticky Note**  
    - Summarizes workflow purpose and technology stacks used including API requirements (DubLab, Telegram, Box, Dropbox, YouTube, Postiz).  

  - **Sticky Note1**  
    - Instructions on usage: obtaining API keys, webhook setup, and upload providers configuration.  

  - **Sticky Note2**  
    - Describes Postiz platform features and supported social media integrations with link: https://postiz.com/.

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                          | Input Node(s)                        | Output Node(s)                                                | Sticky Note                                                                                              |
|---------------------------|-------------------|----------------------------------------|------------------------------------|---------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Start                     | Form Trigger      | Receives user video and language input | None                               | Combine Videos and Languages                                   |                                                                                                        |
| Combine Videos and Languages | Code              | Creates video-language pairs            | Start                              | Init Dubbing, Proxy Videos                                     |                                                                                                        |
| Proxy Videos              | Merge             | Combines video data for upload          | Combine Videos and Languages       | Upload Video, Proxy Ids                                        |                                                                                                        |
| Init Dubbing              | HTTP Request      | Initializes dubbing project in DubLab   | Combine Videos and Languages       | Proxy Videos                                                  |                                                                                                        |
| Upload Video              | HTTP Request      | Uploads video binary to DubLab           | Proxy Videos                      | Proxy Ids                                                    |                                                                                                        |
| Proxy Ids                 | Merge             | Combines IDs for dubbing start           | Upload Video, Init Dubbing         | Start Dubbing                                                |                                                                                                        |
| Start Dubbing             | HTTP Request      | Starts the dubbing process                | Proxy Ids                        | Fetch Projects                                               |                                                                                                        |
| Fetch Projects            | HTTP Request      | Polls DubLab for dubbing projects status | Start Dubbing                    | None                                                        |                                                                                                        |
| Webhook                   | Webhook           | Receives DubLab webhook callbacks        | None                            | Dubbed Video, Original Video                                  |                                                                                                        |
| Dubbed Video              | HTTP Request      | Downloads dubbed video from DubLab        | Webhook                         | Telegram, Box, Dropbox, YouTube, Postiz                      |                                                                                                        |
| Original Video            | HTTP Request      | Downloads original video from DubLab      | Webhook                         | Telegram                                                    |                                                                                                        |
| Telegram                  | Telegram          | Sends dubbed video to Telegram chat       | Dubbed Video, Original Video     | None                                                        |                                                                                                        |
| Box                       | Box               | Uploads dubbed video to Box cloud          | Dubbed Video                    | None                                                        |                                                                                                        |
| Dropbox                   | Dropbox           | Uploads dubbed video to Dropbox folder     | Dubbed Video                    | None                                                        |                                                                                                        |
| YouTube                   | YouTube           | Uploads dubbed video to YouTube channel    | Dubbed Video                    | None                                                        |                                                                                                        |
| Postiz                    | HTTP Request      | Uploads dubbed video to Postiz platform    | Dubbed Video                    | None                                                        |                                                                                                        |
| Sticky Note               | Sticky Note       | Workflow overview and technology stacks   | None                            | None                                                        | ## Dub Videos And Share on Social Media\n\n### Workflows\n1. Via n8n form select files to dub for desired languages.\n2. Listen webhook and whenever dubbing finishes upload to desired platforms\n\n### Used Stacks\n- DubLab App (ApiKey, Webhook Setup Required)\n- Telegram (Token Required)\n- Box (Oauth2 Required)\n- Dropbox (Oauth2 Required)\n- Youtube (Oauth2 Required)\n- Postiz (ApiKey Required) |
| Sticky Note1              | Sticky Note       | Usage instructions and setup notes        | None                            | None                                                        | ## How to Use\n- Obtain Api Key from https://dublab.app, set as variable or just update on nodes.\n- Grab Webhook Url and add to DubLab App\n- Setup upload providers (Telegram, Box, Dropbox, YouTube, Postiz etc.) |
| Sticky Note2              | Sticky Note       | Postiz platform description and link      | None                            | None                                                        | ### Postiz\n- https://postiz.com/\n- Pretty neat tool which supports many social media platforms\n- Supports cloud and self-hosted version\n- Via API post can be shared over below platforms (Facebook, Instagram, Threads, Tiktok, Reddit etc.) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "Start"  
   - Configure form with two fields:  
     - File upload (accept `.mp4`, required)  
     - Destination Language dropdown (multi-select, options: English, German, Russian, Polish, Spanish, Portuguese, Italian, Arabic, French, Turkish, Dutch, required)  

2. **Add Code Node**  
   - Type: Function/Code  
   - Name: "Combine Videos and Languages"  
   - Paste JavaScript code to:  
     - Map language names to ISO codes  
     - Iterate over all input items  
     - For each video and language, create a new item with language code and binary data reference (`VIDEO_n`)  
   - Connect "Start" output to this node’s input.  

3. **Add Merge Node**  
   - Type: Merge  
   - Name: "Proxy Videos"  
   - Set mode to "Combine" by position.  
   - Connect "Combine Videos and Languages" node output to input 1 of this node.  

4. **Add HTTP Request Node**  
   - Type: HTTP Request  
   - Name: "Init Dubbing"  
   - Method: POST  
   - URL: `https://api.dublab.app/v1/init-dub`  
   - Body type: JSON  
   - Body:  
     ```json
     {
       "dest_lang": "{{ $json.language }}",
       "duration": 30,
       "fileType": "{{ $json.mimetype }}",
       "name": "{{ $json.filename }}",
       "source_lang": "auto"
     }
     ```  
   - Headers: Add `ApiKey` header with value from environment variable `DUBLAB_API_KEY`  
   - Connect "Combine Videos and Languages" output to this node’s input (as second input to "Proxy Videos").  

5. **Connect "Init Dubbing" output to input 2 of "Proxy Videos" node**  

6. **Add HTTP Request Node**  
   - Type: HTTP Request  
   - Name: "Upload Video"  
   - Method: PUT  
   - URL: `={{ $json.upload_url }}`  
   - Content Type: binary data  
   - Binary data field name: `=VIDEO_{{ $json.binary_order }}`  
   - Header: `Content-Type: video/mp4`  
   - Connect "Proxy Videos" output to this node’s input.  

7. **Add Merge Node**  
   - Type: Merge  
   - Name: "Proxy Ids"  
   - Mode: Combine by position  
   - Connect "Upload Video" output to input 1  
   - Connect "Init Dubbing" output to input 2 (via "Proxy Videos")  

8. **Add HTTP Request Node**  
   - Type: HTTP Request  
   - Name: "Start Dubbing"  
   - Method: POST  
   - URL: `https://api.dublab.app/v1/start-dub`  
   - Body type: JSON  
   - Body: `{ "id": "{{ $json.id }}" }`  
   - Header: `ApiKey` with `DUBLAB_API_KEY`  
   - Connect "Proxy Ids" output to this node’s input.  

9. **Add HTTP Request Node**  
   - Type: HTTP Request  
   - Name: "Fetch Projects"  
   - Method: GET  
   - URL: `https://api.dublab.app/v1/dubs`  
   - Header: `ApiKey` with `DUBLAB_API_KEY`  
   - Connect "Start Dubbing" output to this node’s input.  

10. **Add Webhook Node**  
    - Type: Webhook  
    - Name: "Webhook"  
    - HTTP Method: POST  
    - Path: set to unique webhook path (e.g., `e41c5f62-f2de-4df0-ac29-851837a67282`)  
    - This node receives DubLab's callbacks when dubbing is done.  

11. **Add HTTP Request Node**  
    - Type: HTTP Request  
    - Name: "Dubbed Video"  
    - Method: GET  
    - URL: `={{ $json.body.dubbed_download_url }}`  
    - Connect "Webhook" output to this node’s input.  

12. **Add HTTP Request Node**  
    - Type: HTTP Request  
    - Name: "Original Video"  
    - Method: GET  
    - URL: `={{ $json.body.original_download_url }}`  
    - Connect "Webhook" output to this node’s input.  

13. **Add Telegram Node**  
    - Type: Telegram  
    - Name: "Telegram"  
    - Operation: sendVideo  
    - Chat ID: `145187085`  
    - Send binary data with filename "Dubbed Video"  
    - Connect outputs of "Dubbed Video" and "Original Video" to this node.  
    - Configure Telegram credentials (token).  

14. **Add Box Node**  
    - Type: Box  
    - Name: "Box"  
    - Upload file with randomized filename pattern including destination language and original filename  
    - Use binary data from "Dubbed Video"  
    - Connect "Dubbed Video" output to this node.  
    - Configure OAuth2 credentials.  

15. **Add Dropbox Node**  
    - Type: Dropbox  
    - Name: "Dropbox"  
    - Path: `/dublab-files/{{ random-string }}-dubbed-{{ $json.body.dest_lang }}-{{ $json.body.name }}`  
    - Upload binary data from "Dubbed Video"  
    - Connect "Dubbed Video" output to this node.  
    - Configure OAuth2 credentials.  

16. **Add YouTube Node**  
    - Type: YouTube  
    - Name: "YouTube"  
    - Resource: Video  
    - Operation: Upload  
    - Title: `{{ $json.body.name }}`  
    - Region Code: PL  
    - Connect "Dubbed Video" output to this node.  
    - Configure OAuth2 credentials.  

17. **Add HTTP Request Node**  
    - Type: HTTP Request  
    - Name: "Postiz"  
    - Method: POST  
    - URL: `https://api.postiz.com/api/public/v1/upload`  
    - Content Type: multipart/form-data  
    - Body Parameter: file (formBinaryData), inputDataFieldName: `data` (binary video)  
    - Headers:  
      - `Authorization`: `POSTIZ_API_KEY` environment variable  
      - `Proxy-Authorization`: `PROXY_AUTH` environment variable  
    - Connect "Dubbed Video" output to this node.  

18. **Add Sticky Notes** for documentation and instructions as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                              |
|--------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| Obtain API key from https://dublab.app and set as environment variable or directly in nodes for DubLab API authentication. | DubLab API setup                             |
| Add the generated webhook URL to the DubLab App dashboard to receive callbacks on dubbing completion.                     | DubLab webhook configuration                 |
| Configure OAuth2 credentials for Telegram, Box, Dropbox, YouTube according to respective platform guidelines.              | OAuth2 credential setup                       |
| Postiz is a social media management platform supporting multiple networks including Facebook, Instagram, Threads, TikTok, Reddit. | https://postiz.com/                           |
| Workflow supports multiple languages with ISO code mappings; unmapped languages default to original input.                 | Language mapping in code node                 |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It complies fully with content policies and contains no illegal or protected content. All data processed is legal and publicly accessible.