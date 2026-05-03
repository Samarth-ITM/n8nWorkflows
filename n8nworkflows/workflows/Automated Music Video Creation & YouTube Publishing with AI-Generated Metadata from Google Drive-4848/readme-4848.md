Automated Music Video Creation & YouTube Publishing with AI-Generated Metadata from Google Drive

https://n8nworkflows.xyz/workflows/automated-music-video-creation---youtube-publishing-with-ai-generated-metadata-from-google-drive-4848


# Automated Music Video Creation & YouTube Publishing with AI-Generated Metadata from Google Drive

### 1. Workflow Overview

This workflow automates the creation and publishing of music videos on YouTube using audio files uploaded to Google Drive. It integrates AI-generated metadata (description, tags, image prompts) and manages media assets throughout the process. The workflow primarily targets musicians, content creators, or agencies looking to streamline video creation and publishing with AI enhancements.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Initial Processing**  
  Detects new song uploads on Google Drive and extracts keywords and cleans up titles.

- **1.2 Audio Processing & AI Transcription**  
  Downloads audio, transcribes it using OpenAI, and generates video descriptions and tags.

- **1.3 Art Generation & Management**  
  Searches for existing cover art or generates prompts for AI image creation, uploads cover art.

- **1.4 Video Composition & Upload**  
  Combines audio and generated image into a video, uploads video to Google Drive and then to YouTube.

- **1.5 Metadata Scheduling & Playlist Management**  
  Retrieves publishing schedule from Google Sheets, decides playlist based on genre, and updates records.

- **1.6 Notifications & Logging**  
  Sends status messages to Discord channels and logs execution data for debugging and monitoring.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initial Processing

**Overview:**  
This block listens for new audio files uploaded to Google Drive and extracts initial metadata to prepare for downstream AI processing.

**Nodes Involved:**  
- Watch New Song in Drive  
- Keywords  
- Clean Title  
- Extract Genre  
- Download Audio  

**Node Details:**  

- **Watch New Song in Drive**  
  - *Type:* Google Drive Trigger  
  - *Role:* Starts workflow when a new song file is detected in Google Drive.  
  - *Config:* Default trigger on new files in configured folder.  
  - *Connections:* Outputs to Keywords and Log8.  
  - *Failures:* Connectivity issues, permission errors on Drive.  

- **Keywords**  
  - *Type:* Set  
  - *Role:* Extract or set keywords related to the song from trigger data.  
  - *Config:* Sets variables based on incoming data.  
  - *Connections:* Outputs to Clean Title.  
  - *Failures:* Expression errors if input data format unexpected.  

- **Clean Title**  
  - *Type:* Code  
  - *Role:* Cleans and formats the song title for consistent processing.  
  - *Config:* Custom JavaScript code to sanitize title strings.  
  - *Connections:* Outputs to Extract Genre and Date & Time nodes.  
  - *Failures:* Code execution errors if input malformed.  

- **Extract Genre**  
  - *Type:* Code  
  - *Role:* Determines the genre of the song based on title or metadata for styling decisions.  
  - *Config:* Custom JavaScript logic.  
  - *Connections:* Outputs to Download Audio.  
  - *Failures:* Logic errors or missing metadata.  

- **Download Audio**  
  - *Type:* Google Drive  
  - *Role:* Downloads the audio file for further transcription and processing.  
  - *Config:* Uses file ID from trigger.  
  - *Connections:* Outputs to Transcribe node.  
  - *Failures:* Download failures, file not found, permission issues.

---

#### 1.2 Audio Processing & AI Transcription

**Overview:**  
Transcribes the downloaded audio using OpenAI, then uses AI to generate a video description and tags.

**Nodes Involved:**  
- Transcribe  
- Description  
- Tags  
- Truncate  
- Message6 (Discord notification)  

**Node Details:**  

- **Transcribe**  
  - *Type:* OpenAI (LangChain)  
  - *Role:* Transcribes audio content to text.  
  - *Config:* Uses OpenAI audio transcription model.  
  - *Connections:* Outputs to Description, Log, and Truncate.  
  - *Failures:* API rate limits, transcription errors, audio format issues.  

- **Description**  
  - *Type:* OpenAI (LangChain)  
  - *Role:* Generates a YouTube video description from transcription.  
  - *Config:* Prompted with transcription text. Retries on failure with 3s intervals.  
  - *Connections:* Outputs to Message2 (Discord), Log2, Genre Styles switch.  
  - *Failures:* API failures, prompt formatting errors.  

- **Tags**  
  - *Type:* OpenAI (LangChain)  
  - *Role:* Generates tags for the YouTube video.  
  - *Config:* Retries on failure with waiting periods.  
  - *Connections:* Outputs to Get Schedule, Message1 (Discord), Log6.  
  - *Failures:* Same as Description.  

- **Truncate**  
  - *Type:* Code  
  - *Role:* Truncates transcription or description text to fit limits.  
  - *Config:* Custom JS code.  
  - *Connections:* Outputs to Message6 (Discord).  
  - *Failures:* Code errors if input data missing or malformed.  

- **Message6**  
  - *Type:* Discord  
  - *Role:* Sends truncated transcription or status to Discord channel.  
  - *Config:* Uses webhooks for notifications.  
  - *Connections:* None.  
  - *Failures:* Webhook failures or Discord API limits.

---

#### 1.3 Art Generation & Management

**Overview:**  
Searches for existing cover art or generates AI prompts for new cover art images, uploads and manages them in Google Drive.

**Nodes Involved:**  
- Search Art  
- If (decision node)  
- Delete Art  
- Cover Art  
- Upload Art  
- Message3 (Discord)  
- Search Video  

**Node Details:**  

- **Search Art**  
  - *Type:* Google Drive  
  - *Role:* Looks for existing cover art files in Drive.  
  - *Config:* Search by naming convention or metadata.  
  - *Connections:* Outputs to If node.  
  - *Failures:* Search errors, permission issues.  

- **If**  
  - *Type:* If (boolean condition)  
  - *Role:* Checks if cover art exists; if yes deletes old art, else proceeds to generate.  
  - *Config:* Condition based on Search Art output count.  
  - *Connections:* True path to Delete Art, False path to Cover Art.  
  - *Failures:* Condition evaluation errors.  

- **Delete Art**  
  - *Type:* Google Drive  
  - *Role:* Deletes outdated or unwanted cover art from Drive.  
  - *Config:* Uses file ID from Search Art.  
  - *Connections:* Outputs to Cover Art after deletion.  
  - *Failures:* Permission denied, file locked.  

- **Cover Art**  
  - *Type:* OpenAI (LangChain)  
  - *Role:* Generates cover art prompts or metadata for art creation.  
  - *Config:* AI-powered prompt generation.  
  - *Connections:* Outputs to Upload Art node.  
  - *Failures:* API errors, prompt failures.  

- **Upload Art**  
  - *Type:* Google Drive  
  - *Role:* Uploads newly generated cover art image to Drive.  
  - *Config:* Retry enabled with 3s wait.  
  - *Connections:* Outputs to Message3 (Discord), Log1 and Search Video.  
  - *Failures:* Upload errors, quota limits.  

- **Message3**  
  - *Type:* Discord  
  - *Role:* Notifies about cover art upload status.  
  - *Config:* Discord webhook.  
  - *Connections:* None.  
  - *Failures:* Discord API issues.  

- **Search Video**  
  - *Type:* Google Drive  
  - *Role:* Searches for existing video files to avoid duplicates.  
  - *Config:* Uses naming conventions.  
  - *Connections:* Outputs to If1.  
  - *Failures:* Search problems, permission issues.

---

#### 1.4 Video Composition & Upload

**Overview:**  
Combines audio and image to create a video, uploads it to Google Drive and then publishes on YouTube with the generated metadata.

**Nodes Involved:**  
- If1  
- Delete Video  
- Combine Audio + Image  
- Upload Video  
- Upload  
- Determine Playlist  
- Switch  
- Update Row  
- Various YouTube nodes (Pop, EDM, Disco, Reggae, Country)  

**Node Details:**  

- **If1**  
  - *Type:* If  
  - *Role:* Checks if video already exists; if yes, deletes old video, else proceeds to combine.  
  - *Connections:* True to Delete Video, False to Combine Audio + Image and Log7.  
  - *Failures:* Condition evaluation issues.  

- **Delete Video**  
  - *Type:* Google Drive  
  - *Role:* Removes old video file from Drive.  
  - *Connections:* Outputs to Combine Audio + Image.  
  - *Failures:* Permission errors, locked files.  

- **Combine Audio + Image**  
  - *Type:* HTTP Request  
  - *Role:* Calls external service to create video from audio and image.  
  - *Config:* Retry enabled, 5s wait.  
  - *Connections:* Outputs to Upload Video.  
  - *Failures:* HTTP errors, service downtime.  

- **Upload Video**  
  - *Type:* Google Drive  
  - *Role:* Uploads the created video file to Drive.  
  - *Connections:* Outputs to Message4 (Discord), Log3, and Tags node.  
  - *Failures:* Upload limits, quota exhaustion.  

- **Upload**  
  - *Type:* YouTube  
  - *Role:* Uploads video from Google Drive to YouTube with metadata.  
  - *Connections:* Outputs to Determine Playlist and Log5.  
  - *Failures:* OAuth token expiry, quota issues, invalid metadata.  

- **Determine Playlist**  
  - *Type:* Code  
  - *Role:* Determines which YouTube playlist to use based on genre.  
  - *Connections:* Outputs to Switch node.  
  - *Failures:* Logic errors, undefined genre.  

- **Switch**  
  - *Type:* Switch  
  - *Role:* Routes upload to appropriate YouTube playlist node based on genre.  
  - *Connections:* Routes to Reggae, EDM, Country, Disco, Pop, or default Update Row.  
  - *Failures:* Routing errors if genre not recognized.  

- **Update Row**  
  - *Type:* Google Sheets  
  - *Role:* Updates publishing schedule or status in Google Sheets.  
  - *Connections:* Outputs to Discord Message and Log nodes.  
  - *Failures:* API limits, sheet access errors.  

- **YouTube Nodes (Pop, EDM, Disco, Reggae, Country)**  
  - *Type:* YouTube  
  - *Role:* Uploads video to specified playlist according to genre.  
  - *Failures:* OAuth issues, quota limits, playlist errors.

---

#### 1.5 Metadata Scheduling & Playlist Management

**Overview:**  
Retrieves scheduling data from Google Sheets, sorts and limits results, formats dates, and controls the timing of publishing.

**Nodes Involved:**  
- Tags  
- Get Schedule  
- Sort  
- Limit  
- Add  
- Format Date For Youtube  
- Format  
- If2  

**Node Details:**  

- **Tags**  
  - See above (AI-generated tags) outputs to Get Schedule.  

- **Get Schedule**  
  - *Type:* Google Sheets  
  - *Role:* Pulls publishing schedule for upcoming videos.  
  - *Connections:* Outputs to Sort.  
  - *Failures:* Sheet access, data format issues.  

- **Sort**  
  - *Type:* Sort  
  - *Role:* Sorts schedule rows by date/time or priority.  
  - *Connections:* Outputs to Limit.  
  - *Failures:* Sorting errors if data is malformed.  

- **Limit**  
  - *Type:* Limit  
  - *Role:* Limits number of videos to publish per run.  
  - *Connections:* Outputs to Add and If2.  
  - *Failures:* Misconfiguration of limits.  

- **Add**  
  - *Type:* DateTime  
  - *Role:* Adds time offset to dates for scheduling.  
  - *Connections:* Outputs to Format Date For Youtube and Format nodes.  
  - *Failures:* Date format errors.  

- **Format Date For Youtube**  
  - *Type:* Code  
  - *Role:* Formats date strings to YouTube API compatible format.  
  - *Connections:* Outputs to Download Video.  
  - *Failures:* Date parsing errors.  

- **Format**  
  - *Type:* DateTime  
  - *Role:* Formats dates for logging or display.  
  - *Connections:* Outputs to Log9.  
  - *Failures:* Formatting issues.  

- **If2**  
  - *Type:* If  
  - *Role:* Decision node controlling conditional workflow branching based on schedule.  
  - *Failures:* Condition misconfiguration.

---

#### 1.6 Notifications & Logging

**Overview:**  
Sends progress and status messages to Discord channels and records execution data for monitoring.

**Nodes Involved:**  
- Message (Discord) nodes: Message, Message1, Message2, Message3, Message4, Message6  
- Log nodes: Log, Log1, Log2, Log3, Log4, Log5, Log6, Log7, Log8, Log9  
- Execution Data nodes  

**Node Details:**  

- **Message Nodes**  
  - *Type:* Discord  
  - *Role:* Inform users of workflow progress, errors, or success notifications via Discord webhooks.  
  - *Config:* Webhook URLs per message node.  
  - *Failures:* Webhook failure, rate limits.  

- **Log Nodes**  
  - *Type:* Execution Data  
  - *Role:* Capture and store execution details for debugging.  
  - *Config:* Default.  
  - *Failures:* None critical; just diagnostic.

---

### 3. Summary Table

| Node Name            | Node Type                  | Functional Role                            | Input Node(s)                          | Output Node(s)                         | Sticky Note                      |
|----------------------|----------------------------|------------------------------------------|--------------------------------------|--------------------------------------|---------------------------------|
| Watch New Song in Drive | Google Drive Trigger       | Detect new song upload                    | -                                    | Keywords, Log8                       |                                 |
| Keywords             | Set                        | Extract keywords from upload              | Watch New Song in Drive               | Clean Title                         |                                 |
| Clean Title          | Code                       | Clean and format song title               | Keywords                             | Extract Genre, Date & Time           |                                 |
| Extract Genre        | Code                       | Determine song genre                       | Clean Title                         | Download Audio                      |                                 |
| Download Audio       | Google Drive               | Download audio file                        | Extract Genre                       | Transcribe                         |                                 |
| Transcribe           | OpenAI (LangChain)         | Transcribe audio to text                   | Download Audio                      | Description, Log, Truncate          |                                 |
| Description          | OpenAI (LangChain)         | Generate video description                 | Transcribe                         | Message2, Log2, Genre Styles        | Attempt failed so retrying      |
| Tags                 | OpenAI (LangChain)         | Generate YouTube tags                      | Upload Video                       | Get Schedule, Message1, Log6        | Attempt failed so retrying      |
| Truncate             | Code                       | Truncate long text                         | Transcribe                         | Message6                          |                                 |
| Message6             | Discord                    | Notify truncated transcription             | Truncate                           | -                                  |                                 |
| Search Art           | Google Drive               | Find existing cover art                     | Image Prompt                      | If                               |                                 |
| If                   | If                         | Decide to delete or create art             | Search Art                        | Delete Art, Cover Art               |                                 |
| Delete Art           | Google Drive               | Delete old cover art                        | If                               | Cover Art                         |                                 |
| Cover Art            | OpenAI (LangChain)         | Generate cover art prompt                   | If                               | Upload Art                        | Attempt failed so retrying      |
| Upload Art           | Google Drive               | Upload new cover art                        | Cover Art                        | Message3, Log1, Search Video        |                                 |
| Message3             | Discord                    | Notify art upload status                    | Upload Art                      | -                                  |                                 |
| Search Video         | Google Drive               | Check for existing video                    | Upload Art                      | If1                              |                                 |
| If1                  | If                         | Decide to delete old video or combine      | Search Video                    | Delete Video, Combine Audio + Image |                                 |
| Delete Video         | Google Drive               | Delete old video                            | If1                              | Combine Audio + Image              |                                 |
| Combine Audio + Image | HTTP Request               | Create video from audio and image           | Delete Video, If1 (false path)    | Upload Video                      |                                 |
| Upload Video         | Google Drive               | Upload created video                        | Combine Audio + Image            | Message4, Log3, Tags              |                                 |
| Upload               | YouTube                    | Publish video to YouTube                     | Download Video                   | Determine Playlist, Log5          |                                 |
| Determine Playlist   | Code                       | Choose playlist based on genre               | Upload                         | Switch                          |                                 |
| Switch               | Switch                     | Route to playlist-specific YouTube node     | Determine Playlist               | Reggae, EDM, Country, Disco, Pop, Update Row |                                 |
| Reggae               | YouTube                    | Upload to Reggae playlist                    | Switch                         | Update Row                      |                                 |
| EDM                  | YouTube                    | Upload to EDM playlist                       | Switch                         | Update Row                      |                                 |
| Country              | YouTube                    | Upload to Country playlist                   | Switch                         | Update Row                      |                                 |
| Disco                | YouTube                    | Upload to Disco playlist                     | Switch                         | Update Row                      |                                 |
| Pop                  | YouTube                    | Upload to Pop playlist                       | Switch                         | Update Row                      |                                 |
| Update Row           | Google Sheets              | Update publishing schedule/status            | Various YouTube nodes           | Log4, Message                    |                                 |
| Get Schedule         | Google Sheets              | Retrieve publishing schedule                  | Tags                           | Sort                           |                                 |
| Sort                 | Sort                       | Sort schedule data                            | Get Schedule                  | Limit                          |                                 |
| Limit                | Limit                      | Limit number of publishes per run             | Sort                          | Add, If2                       |                                 |
| Add                  | DateTime                   | Add offset time for scheduling                 | Limit                         | Format Date For Youtube, Format |                                 |
| Format Date For Youtube | Code                     | Format date for YouTube API                    | Add                           | Download Video                 |                                 |
| Format               | DateTime                   | Format date for logging or display              | Add                           | Log9                          |                                 |
| If2                  | If                         | Conditional branching on scheduling             | Limit                         | -                             |                                 |
| Message              | Discord                    | Notify publishing status                        | Update Row                    | -                             |                                 |
| Message1             | Discord                    | Notify tags generation                           | Tags                         | -                             |                                 |
| Message2             | Discord                    | Notify description generation                    | Description                  | -                             |                                 |
| Message4             | Discord                    | Notify video upload                              | Upload Video                 | -                             |                                 |
| Log                  | Execution Data             | Log transcription data                           | Transcribe                   | -                             |                                 |
| Log1                 | Execution Data             | Log cover art upload                              | Upload Art                   | -                             |                                 |
| Log2                 | Execution Data             | Log description generation                         | Description                  | -                             |                                 |
| Log3                 | Execution Data             | Log video upload                                   | Upload Video                 | -                             |                                 |
| Log4                 | Execution Data             | Log update row operation                           | Update Row                   | -                             |                                 |
| Log5                 | Execution Data             | Log YouTube upload                                 | Upload                       | -                             |                                 |
| Log6                 | Execution Data             | Log tags generation                                 | Tags                         | -                             |                                 |
| Log7                 | Execution Data             | Log video combining process                         | If1                          | -                             |                                 |
| Log8                 | Execution Data             | Log initial trigger data                            | Watch New Song in Drive      | -                             |                                 |
| Log9                 | Execution Data             | Log formatted date                                  | Format                       | -                             |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node** named "Watch New Song in Drive"  
   - Configure to trigger on new audio files in a specific folder.

2. **Add Set Node "Keywords"**  
   - Extract keywords or initial metadata from the trigger data (e.g., file name).

3. **Add Code Node "Clean Title"**  
   - Implement JavaScript code to sanitize and standardize the song title.

4. **Add Code Node "Extract Genre"**  
   - Use custom logic to infer genre from title or metadata.

5. **Add Google Drive Node "Download Audio"**  
   - Configure to download the audio file based on file ID from trigger.

6. **Add OpenAI Node "Transcribe"**  
   - Use OpenAI audio transcription model on downloaded audio.  
   - Connect "Download Audio" output here.

7. **Add OpenAI Node "Description"**  
   - Configure prompt to generate YouTube description from transcription.  
   - Enable retry on failure with 3s wait and max retries.

8. **Add OpenAI Node "Tags"**  
   - Generate YouTube video tags from transcription or description.  
   - Enable retry on failure similarly.

9. **Add Code Node "Truncate"**  
   - Implement text truncation logic for long transcriptions/descriptions.

10. **Add Discord Node "Message6"**  
    - Send truncated transcription notification to Discord webhook.

11. **Add Google Drive Node "Search Art"**  
    - Search for existing cover art files by naming convention.

12. **Add If Node "If"**  
    - Condition: If cover art exists (search results > 0), route to "Delete Art" else to "Cover Art".

13. **Add Google Drive Node "Delete Art"**  
    - Delete existing cover art files.

14. **Add OpenAI Node "Cover Art"**  
    - Generate new cover art prompt with retry on failure.

15. **Add Google Drive Node "Upload Art"**  
    - Upload generated cover art image to Drive with retry.

16. **Add Discord Node "Message3"**  
    - Notify about cover art upload status.

17. **Add Google Drive Node "Search Video"**  
    - Search for existing videos in Drive by naming.

18. **Add If Node "If1"**  
    - If video exists, route to "Delete Video" else to "Combine Audio + Image".

19. **Add Google Drive Node "Delete Video"**  
    - Delete old video files.

20. **Add HTTP Request Node "Combine Audio + Image"**  
    - Call external video creation API or service with audio and cover art as inputs.  
    - Enable retry with 5s wait.

21. **Add Google Drive Node "Upload Video"**  
    - Upload created video file to Drive.

22. **Add Discord Node "Message4"**  
    - Notify video upload status.

23. **Add OpenAI Node "Tags"**  
    - (Re-use or connect here for metadata generation).

24. **Add YouTube Node "Upload"**  
    - Upload video to YouTube from Google Drive file.  
    - Configure OAuth2 credentials.  
    - Connect output to "Determine Playlist".

25. **Add Code Node "Determine Playlist"**  
    - Implement logic to select playlist based on genre.

26. **Add Switch Node "Switch"**  
    - Route to playlist-specific YouTube nodes: Reggae, EDM, Country, Disco, Pop, or default.

27. **Add YouTube Nodes for each playlist**  
    - Configure each with respective playlist IDs and credentials.

28. **Add Google Sheets Node "Get Schedule"**  
    - Retrieve publishing schedule data.

29. **Add Sort Node**  
    - Sort schedule data by date.

30. **Add Limit Node**  
    - Limit the number of videos to publish.

31. **Add DateTime Node "Add"**  
    - Add offset time for scheduling.

32. **Add Code Node "Format Date For Youtube"**  
    - Format dates to YouTube API compatible strings.

33. **Add Google Drive Node "Download Video"**  
    - Download video file for upload.

34. **Add Google Sheets Node "Update Row"**  
    - Update schedule with publishing status.

35. **Add Multiple Discord Message Nodes**  
    - Send notifications at various stages.

36. **Add Execution Data Nodes** as needed for logging and debugging.

37. **Set up all necessary credentials**:  
    - Google Drive (OAuth2)  
    - Google Sheets (OAuth2)  
    - YouTube (OAuth2) with playlist access  
    - OpenAI API key  
    - Discord webhook URLs

38. **Test each step individually** to validate data flow and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow heavily relies on OpenAI LangChain nodes for transcription and metadata generation.    | OpenAI API documentation: https://openai.com/api |
| Google Drive and YouTube OAuth2 credentials must be authorized with appropriate scopes.             | Google Cloud Console for credential setup         |
| Discord webhooks are used for real-time notifications; ensure webhooks are properly configured.     | Discord webhook setup guide                        |
| Retry mechanisms with wait times are implemented on critical nodes to improve resilience.            | n8n retry configuration documentation              |
| External HTTP service for combining audio and image must be operational and accessible.              | Custom or third-party video creation API           |

---

This documentation should empower users and automation agents to fully understand, reproduce, and maintain the "Automated Music Video Creation & YouTube Publishing with AI-Generated Metadata from Google Drive" workflow in n8n.