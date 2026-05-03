Obsidian Notes Read Aloud using AI: Available as a Podcast Feed

https://n8nworkflows.xyz/workflows/obsidian-notes-read-aloud-using-ai--available-as-a-podcast-feed-2699


# Obsidian Notes Read Aloud using AI: Available as a Podcast Feed

### 1. Workflow Overview

This workflow automates the conversion of Obsidian notes into natural-sounding audio episodes, which are then published as a professional podcast feed compatible with major platforms like Apple Podcasts, Spotify, and Google Podcasts. It is designed for users who want to transform their written notes into audio content and distribute it seamlessly via podcast channels.

The workflow is logically divided into two main blocks:

- **1.1 Input Reception and Audio Generation**  
  Receives note content from Obsidian via webhook, converts the text to audio using OpenAI’s text-to-speech capabilities, generates episode descriptions, uploads audio to Cloudinary, and prepares metadata for storage.

- **1.2 Podcast Feed Generation**  
  Retrieves episode metadata from Google Sheets, combines it with static podcast metadata, generates a compliant RSS podcast feed XML, and serves it via webhook for consumption by podcast platforms.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Audio Generation

**Overview:**  
This block handles incoming note data from Obsidian, converts the text to audio, generates a concise episode description, uploads the audio file to Cloudinary, and prepares the episode metadata for storage in Google Sheets.

**Nodes Involved:**  
- Webhook GET Note  
- OpenAI1 (Audio generation)  
- OpenAI (Description generation)  
- Give Audio Unique Name  
- Upload Audio to Cloudinary  
- Send Audio to Obsidian  
- Merge  
- Aggregate  
- Rename Fields  
- Append Item to Google Sheet  
- Sticky Notes: Sticky Note, Sticky Note2, Sticky Note7, Sticky Note5

**Node Details:**

- **Webhook GET Note**  
  - Type: Webhook (POST)  
  - Role: Entry point receiving note content from Obsidian’s Post Webhook plugin.  
  - Configuration: Listens on a unique webhook path; expects JSON body with note content and metadata such as timestamp and filename.  
  - Inputs: External HTTP POST from Obsidian.  
  - Outputs: Passes note content downstream.  
  - Edge Cases: Missing or malformed payload; webhook downtime.

- **OpenAI1**  
  - Type: OpenAI (Langchain node)  
  - Role: Converts note text to natural-sounding audio in MP3 format using OpenAI’s audio resource.  
  - Configuration: Input is the note content from webhook; response format set to "mp3".  
  - Credentials: Requires OpenAI API key.  
  - Inputs: Note content JSON.  
  - Outputs: Binary audio data.  
  - Edge Cases: API rate limits, network errors, unsupported content.

- **OpenAI**  
  - Type: OpenAI (Langchain node)  
  - Role: Generates a concise, engaging episode description (50–150 characters) based on note content.  
  - Configuration: Uses GPT-4o-mini model; system prompt instructs to summarize key ideas without repetition.  
  - Inputs: Note content JSON.  
  - Outputs: Text description.  
  - Edge Cases: API errors, ambiguous input text.

- **Give Audio Unique Name**  
  - Type: Set node  
  - Role: Assigns a unique filename to the audio file based on the note’s timestamp (e.g., "timestamp.mp3").  
  - Configuration: Extracts timestamp from webhook payload and appends ".mp3".  
  - Inputs: Audio data and metadata.  
  - Outputs: Adds `fileName` field for downstream use.  
  - Edge Cases: Missing timestamp field.

- **Upload Audio to Cloudinary**  
  - Type: HTTP Request  
  - Role: Uploads the generated MP3 audio file to Cloudinary cloud storage.  
  - Configuration: POST multipart/form-data to Cloudinary upload API; uses custom HTTP header authentication with base64-encoded API key and secret; preset "rb_preset" used; resource type auto-detected.  
  - Credentials: Custom Auth with Cloudinary API key/secret.  
  - Inputs: Binary audio data with assigned filename.  
  - Outputs: Cloudinary response including URL and duration metadata.  
  - Edge Cases: Authentication failures, upload errors, network timeouts.

- **Send Audio to Obsidian**  
  - Type: Respond to Webhook  
  - Role: Sends the generated audio file back to the Obsidian client as an HTTP response with content-type "audio/mpeg".  
  - Inputs: Binary audio data.  
  - Outputs: HTTP response to webhook caller.  
  - Edge Cases: Large file size causing timeout, client disconnect.

- **Merge**  
  - Type: Merge node  
  - Role: Combines Cloudinary upload response and OpenAI description outputs into a single data stream for further processing.  
  - Inputs: From Upload Audio to Cloudinary and OpenAI description nodes.  
  - Outputs: Merged JSON data.  
  - Edge Cases: Mismatched input counts or timing issues.

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Aggregates all item data into one object for easier field manipulation.  
  - Inputs: Merged data.  
  - Outputs: Single aggregated JSON object.  
  - Edge Cases: Empty input data.

- **Rename Fields**  
  - Type: Set node  
  - Role: Maps and renames fields to standard podcast metadata keys: title, link (Cloudinary URL), description, date (upload date), duration.  
  - Configuration: Extracts filename without extension as title; uses Cloudinary URL and metadata; description from OpenAI; date and duration from Cloudinary response.  
  - Inputs: Aggregated data.  
  - Outputs: Standardized metadata JSON.  
  - Edge Cases: Missing or malformed Cloudinary response fields.

- **Append Item to Google Sheet**  
  - Type: Google Sheets (Append operation)  
  - Role: Stores episode metadata (title, link, description, date, duration) in a Google Sheet for podcast feed generation.  
  - Configuration: Uses OAuth2 credentials; appends to sheet "gid=0" in specified spreadsheet.  
  - Inputs: Standardized metadata JSON.  
  - Outputs: Confirmation of append operation.  
  - Edge Cases: Google API quota limits, authentication errors.

- **Sticky Notes**  
  - Provide setup instructions, usage tips, and explanations for the above nodes and their roles.

---

#### 2.2 Podcast Feed Generation

**Overview:**  
This block generates a professional podcast RSS feed XML by combining static podcast metadata with dynamic episode data retrieved from Google Sheets. The feed is served via webhook for podcast platforms to consume.

**Nodes Involved:**  
- Webhook GET Podcast Feed  
- Get Items from Google Sheets  
- Manually Enter Other Data for Podcast Feed  
- Write RSS Feed (Code node)  
- Return Podcast Feed to Webhook  
- Sticky Notes: Sticky Note1, Sticky Note4, Sticky Note6

**Node Details:**

- **Webhook GET Podcast Feed**  
  - Type: Webhook (GET)  
  - Role: Entry point to serve the generated podcast RSS feed XML.  
  - Configuration: Listens on a unique webhook path; responds with XML content.  
  - Inputs: HTTP GET request from podcast platforms or browsers.  
  - Outputs: Triggers feed generation nodes.  
  - Edge Cases: High traffic load, invalid requests.

- **Get Items from Google Sheets**  
  - Type: Google Sheets (Read operation)  
  - Role: Retrieves all episode metadata rows from the configured Google Sheet.  
  - Configuration: Uses OAuth2 credentials; reads from sheet "gid=0" in specified spreadsheet.  
  - Inputs: Triggered by webhook.  
  - Outputs: Array of episode metadata JSON objects.  
  - Edge Cases: Empty sheet, API errors.

- **Manually Enter Other Data for Podcast Feed**  
  - Type: Set node  
  - Role: Holds static podcast metadata such as base URL, podcast title, description, author, owner info, cover image URL, language, explicit content flag, and iTunes category.  
  - Configuration: User-editable fields with default values.  
  - Inputs: Episode metadata from Google Sheets.  
  - Outputs: Combined data for feed generation.  
  - Edge Cases: Missing or incorrect static metadata.

- **Write RSS Feed**  
  - Type: Code (JavaScript)  
  - Role: Generates the full RSS feed XML string using episode data and static metadata.  
  - Configuration:  
    - Formats dates to RFC 822 format.  
    - Converts duration seconds to HH:MM:SS format.  
    - Sanitizes text fields to escape XML entities.  
    - Dynamically assigns episode numbers based on row number; season number fixed to 1.  
    - Constructs RSS channel and item elements compliant with iTunes podcast specs.  
  - Inputs: Combined static and dynamic metadata.  
  - Outputs: JSON object containing the RSS feed XML string.  
  - Edge Cases: Malformed data causing XML errors, missing required fields.

- **Return Podcast Feed to Webhook**  
  - Type: Respond to Webhook  
  - Role: Returns the generated RSS feed XML as the HTTP response with content-type "application/xml".  
  - Inputs: RSS feed XML string.  
  - Outputs: HTTP response to podcast platform or client.  
  - Edge Cases: Large feed size, network errors.

- **Sticky Notes**  
  - Explain the purpose of the podcast feed module, configuration instructions, and usage notes.

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                                   | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                          |
|--------------------------------|--------------------------------|-------------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook GET Note               | Webhook (POST)                 | Receives note content from Obsidian             | External HTTP POST             | OpenAI1, OpenAI                | Send Notes to Webhook: Setup and usage instructions with Post Webhook Plugin link                   |
| OpenAI1                       | OpenAI (Langchain)             | Converts note text to MP3 audio                   | Webhook GET Note              | Give Audio Unique Name         | Create Audio and Write Description: OpenAI TTS and description generation explanation              |
| OpenAI                        | OpenAI (Langchain)             | Generates concise episode description             | Webhook GET Note              | Merge                         | Create Audio and Write Description                                                                 |
| Give Audio Unique Name         | Set                           | Assigns unique filename to audio file             | OpenAI1                       | Upload Audio to Cloudinary, Send Audio to Obsidian | Audio to Cloudinary and Obsidian: Cloudinary setup instructions                                     |
| Upload Audio to Cloudinary     | HTTP Request                  | Uploads audio file to Cloudinary                   | Give Audio Unique Name        | Merge                         | Audio to Cloudinary and Obsidian                                                                   |
| Send Audio to Obsidian         | Respond to Webhook             | Sends audio file back to Obsidian client          | Give Audio Unique Name        | —                            | Audio to Cloudinary and Obsidian                                                                   |
| Merge                         | Merge                         | Combines Cloudinary upload response and description | Upload Audio to Cloudinary, OpenAI | Aggregate                    | Prepare Relevant Data: Consolidates data for Google Sheets                                         |
| Aggregate                    | Aggregate                    | Aggregates merged data into single object          | Merge                        | Rename Fields                 | Prepare Relevant Data                                                                              |
| Rename Fields                 | Set                           | Maps and renames fields for podcast metadata       | Aggregate                    | Append Item to Google Sheet   | Prepare Relevant Data                                                                              |
| Append Item to Google Sheet    | Google Sheets (Append)         | Stores episode metadata in Google Sheets           | Rename Fields                | —                            | Append Row to Google Sheets: Saves podcast parameters                                             |
| Webhook GET Podcast Feed       | Webhook (GET)                 | Serves podcast RSS feed XML                         | External HTTP GET             | Get Items from Google Sheets  | Generic Podcast Feed Module: Reusable podcast feed generation module                              |
| Get Items from Google Sheets   | Google Sheets (Read)           | Retrieves episode metadata from Google Sheets      | Webhook GET Podcast Feed     | Manually Enter Other Data for Podcast Feed | Podcast Feed Configuration: Static and dynamic metadata instructions                              |
| Manually Enter Other Data for Podcast Feed | Set                           | Holds static podcast metadata                       | Get Items from Google Sheets | Write RSS Feed               | Podcast Feed Configuration                                                                       |
| Write RSS Feed                | Code (JavaScript)              | Generates RSS feed XML from metadata                | Manually Enter Other Data for Podcast Feed | Return Podcast Feed to Webhook | Write Podcast Feed: Generates RSS feed XML from collected data                                    |
| Return Podcast Feed to Webhook | Respond to Webhook             | Returns RSS feed XML as HTTP response               | Write RSS Feed               | —                            | Write Podcast Feed                                                                               |
| Sticky Note                   | Sticky Note                   | Setup instructions for sending notes to webhook    | —                           | —                            | Send Notes to Webhook: Setup and usage instructions with Post Webhook Plugin link                   |
| Sticky Note1                  | Sticky Note                   | Explains generic podcast feed module                | —                           | —                            | Generic Podcast Feed Module: Reusable podcast feed generation module                              |
| Sticky Note2                  | Sticky Note                   | Explains audio creation and description generation | —                           | —                            | Create Audio and Write Description                                                               |
| Sticky Note3                  | Sticky Note                   | Explains Google Sheets append operation             | —                           | —                            | Append Row to Google Sheets                                                                       |
| Sticky Note4                  | Sticky Note                   | Explains podcast feed configuration                 | —                           | —                            | Podcast Feed Configuration                                                                       |
| Sticky Note5                  | Sticky Note                   | Explains data preparation for Google Sheets         | —                           | —                            | Prepare Relevant Data                                                                            |
| Sticky Note6                  | Sticky Note                   | Explains podcast feed writing                        | —                           | —                            | Write Podcast Feed                                                                              |
| Sticky Note7                  | Sticky Note                   | Explains Cloudinary audio upload and setup          | —                           | —                            | Audio to Cloudinary and Obsidian                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook GET Note Node**  
   - Type: Webhook (POST)  
   - Path: Unique identifier (e.g., "64fac784-9b98-4bbc-aaf2-dd45763d3362")  
   - Response Mode: Response Node  
   - Purpose: Receive note content JSON from Obsidian Post Webhook plugin.

2. **Create OpenAI1 Node (Audio Generation)**  
   - Type: OpenAI (Langchain)  
   - Resource: Audio  
   - Options: Response format set to "mp3"  
   - Input: Expression `={{ $json.body.content }}` from Webhook GET Note  
   - Credentials: OpenAI API key  
   - Purpose: Convert text note to MP3 audio.

3. **Create OpenAI Node (Description Generation)**  
   - Type: OpenAI (Langchain)  
   - Model: GPT-4o-mini  
   - Messages:  
     - User content: `={{ $json.body.content }}`  
     - System prompt: "Based on the user input text, write a concise and engaging description of 50–150 characters..."  
   - Credentials: OpenAI API key  
   - Input: Webhook GET Note output  
   - Purpose: Generate episode description.

4. **Create Give Audio Unique Name Node**  
   - Type: Set  
   - Assign `fileName` field: `={{ $('Webhook GET Note').item.json.body.timestamp }}.mp3`  
   - Include other fields  
   - Input: OpenAI1 output  
   - Purpose: Assign unique filename for audio.

5. **Create Upload Audio to Cloudinary Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.cloudinary.com/v1_1/CLOUDINARY_ENV/upload` (replace `CLOUDINARY_ENV` with your Cloudinary cloud name)  
   - Authentication: Custom HTTP Header Auth  
   - Headers: Content-Type: multipart/form-data  
   - Body Parameters:  
     - file: binary data from previous node  
     - upload_preset: "rb_preset"  
     - resource_type: "auto"  
   - Credentials: Custom Auth with base64-encoded API key and secret as Authorization header  
   - Input: Give Audio Unique Name output  
   - Purpose: Upload audio file to Cloudinary.

6. **Create Send Audio to Obsidian Node**  
   - Type: Respond to Webhook  
   - Respond With: Binary  
   - Response Headers: content-type = audio/mpeg  
   - Input: Give Audio Unique Name output  
   - Purpose: Return audio file to Obsidian client.

7. **Create Merge Node**  
   - Type: Merge  
   - Inputs: Upload Audio to Cloudinary and OpenAI (description) outputs  
   - Purpose: Combine audio upload response and description.

8. **Create Aggregate Node**  
   - Type: Aggregate  
   - Aggregate: All item data into one object  
   - Input: Merge output  
   - Purpose: Consolidate data for field mapping.

9. **Create Rename Fields Node**  
   - Type: Set  
   - Assign fields:  
     - title: filename without ".md" extension from webhook payload  
     - link: Cloudinary URL from upload response  
     - description: OpenAI description content  
     - date: Cloudinary created_at timestamp  
     - duration: Cloudinary audio duration  
   - Input: Aggregate output  
   - Purpose: Standardize metadata fields.

10. **Create Append Item to Google Sheet Node**  
    - Type: Google Sheets (Append)  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: "gid=0" or your sheet tab  
    - Columns: title, link, description, date, duration  
    - Credentials: Google Sheets OAuth2  
    - Input: Rename Fields output  
    - Purpose: Store episode metadata for feed generation.

11. **Create Webhook GET Podcast Feed Node**  
    - Type: Webhook (GET)  
    - Path: Unique identifier (e.g., "2f0a6706-54da-4b89-91f4-5e147b393bd8h")  
    - Response Mode: Response Node  
    - Purpose: Serve podcast RSS feed XML.

12. **Create Get Items from Google Sheets Node**  
    - Type: Google Sheets (Read)  
    - Document ID and Sheet Name: Same as Append node  
    - Credentials: Google Sheets OAuth2  
    - Input: Webhook GET Podcast Feed output  
    - Purpose: Retrieve all episode metadata.

13. **Create Manually Enter Other Data for Podcast Feed Node**  
    - Type: Set  
    - Assign static podcast metadata fields:  
      - baseUrl, podcastTitle, podcastDescription, authorName, ownerName, ownerEmail, coverImageUrl, language, explicitContent, itunesCategory  
    - Input: Get Items from Google Sheets output  
    - Purpose: Provide static metadata for feed.

14. **Create Write RSS Feed Node**  
    - Type: Code (JavaScript)  
    - Input: Manually Enter Other Data for Podcast Feed output (static + dynamic data)  
    - Logic:  
      - Sanitize text fields for XML  
      - Format dates to RFC 822  
      - Format duration to HH:MM:SS  
      - Generate RSS XML with iTunes podcast tags  
      - Assign episode numbers dynamically based on row number  
    - Output: JSON with `rssFeed` string  
    - Purpose: Generate podcast RSS feed XML.

15. **Create Return Podcast Feed to Webhook Node**  
    - Type: Respond to Webhook  
    - Respond With: Text  
    - Response Headers: Content-Type = application/xml  
    - Input: Write RSS Feed output  
    - Purpose: Return RSS feed XML to caller.

16. **Add Sticky Notes**  
    - Add explanatory sticky notes at relevant points for setup instructions, usage tips, and clarifications.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Install and configure the [Post Webhook Plugin](https://github.com/Masterb1234/obsidian-post-webhook/) in Obsidian to send notes to n8n. | Obsidian integration setup.                                                                             |
| Create Custom Auth credentials in n8n for Cloudinary using base64-encoded API key and secret.    | Cloudinary authentication setup.                                                                        |
| The podcast feed module is reusable for any '[...]-to-Podcast' workflow, generating standard RSS from Google Sheets data. | Podcast feed generation modularity.                                                                     |
| Podcast feed metadata (title, author, cover image, etc.) must be configured in the "Manually Enter Other Data for Podcast Feed" node. | Static podcast metadata configuration.                                                                  |
| The workflow supports audio file storage in Cloudinary, which also provides duration metadata for podcast feed compliance. | Audio hosting and metadata extraction.                                                                  |
| The RSS feed generated complies with iTunes podcast specifications, including episode and season numbering, explicit content flag, and categories. | Podcast feed compliance and compatibility.                                                             |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Obsidian Notes Read Aloud: Available as a Podcast Feed" workflow in n8n. It covers all nodes, their configurations, data flow, and integration points, enabling advanced users and AI agents to work effectively with this automation.