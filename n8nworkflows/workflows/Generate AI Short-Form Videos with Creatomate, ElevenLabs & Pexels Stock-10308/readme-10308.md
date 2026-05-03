Generate AI Short-Form Videos with Creatomate, ElevenLabs & Pexels Stock

https://n8nworkflows.xyz/workflows/generate-ai-short-form-videos-with-creatomate--elevenlabs---pexels-stock-10308


# Generate AI Short-Form Videos with Creatomate, ElevenLabs & Pexels Stock

### 1. Workflow Overview

This workflow automates the generation of AI-driven short-form videos by integrating multiple services: Creatomate for video creation, ElevenLabs for voice synthesis, and Pexels Stock for video assets. It is designed for content creators, marketers, and social media managers who want to turn text inputs into engaging short videos without manual editing. The workflow orchestrates text input reception, AI text processing, video and audio asset sourcing, voice generation, video assembly, and final publishing.

Logical blocks identified:

- **1.1 Input Reception:** Captures user input from a form trigger and initiates AI text processing.
- **1.2 AI Text Processing:** Uses OpenAI or Anthropic language models to create structured short video scripts and generate video search terms.
- **1.3 Video Asset Retrieval:** Searches Pexels Stock videos using AI-generated terms, retrieves video details, and selects appropriate video files.
- **1.4 Audio Generation and Processing:** Generates voice audio from the script with ElevenLabs, uploads and sets permissions on Google Drive.
- **1.5 Video Creation and Monitoring:** Submits video creation requests to Creatomate, waits for video processing, checks status, and optionally deletes temporary audio files.
- **1.6 Looping and Formatting:** Handles batching over multiple items and formats video links and music selection.
- **1.7 Finalization:** Completes the workflow after successful video creation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for user form submissions to begin the short video generation process.
- **Nodes Involved:** On form submission
- **Node Details:**
  - **On form submission**
    - Type: Form Trigger
    - Role: Entry point; listens for HTTP form submissions to trigger workflow.
    - Configuration: Webhook set up with generated webhook ID, no additional parameters.
    - Inputs: External HTTP POST with form data.
    - Outputs: Triggers "Short Text" node.
    - Edge cases: Failure if webhook unreachable or misconfigured; malformed form data.

#### 2.2 AI Text Processing

- **Overview:** Processes the input text via AI language models to generate the short video script and video search terms.
- **Nodes Involved:** Short Text, Structured Output Parser1, Video Search Terms, Structured Output Parser, Format For Loop Over Items
- **Node Details:**
  - **Short Text**
    - Type: Langchain AI Agent (OpenAI/Anthropic)
    - Role: Generates structured short video script from input text.
    - Configuration: Uses AI language model (Anthropic Chat Model disabled, so likely OpenAI Chat Model active).
    - Inputs: Data from "On form submission"
    - Outputs: "Video Search Terms"
    - Edge cases: API auth failure, rate limits, or malformed prompt responses.
  - **Structured Output Parser1**
    - Type: Langchain Structured Output Parser
    - Role: Parses AI text output into structured JSON.
    - Inputs: AI response from "Short Text"
    - Outputs: "Short Text" node (for further processing)
  - **Video Search Terms**
    - Type: Langchain AI Agent
    - Role: Generates keywords for video search based on the short video script.
    - Inputs: Output from "Short Text"
    - Outputs: "Format For Loop Over Items"
  - **Structured Output Parser**
    - Type: Langchain Structured Output Parser
    - Role: Parses AI-generated search terms into structured format.
    - Inputs: AI response from "Video Search Terms"
    - Outputs: "Video Search Terms"
  - **Format For Loop Over Items**
    - Type: Code
    - Role: Formats structured search terms into batchable items for looping.
    - Inputs: Parsed search terms
    - Outputs: "Loop Over Items"
    - Edge cases: Expression errors or empty input arrays.

#### 2.3 Video Asset Retrieval

- **Overview:** Searches Pexels Stock for videos matching AI-generated terms, retrieves detailed video info, finds the optimal video file size.
- **Nodes Involved:** Loop Over Items, Search Videos, Get The Video Details, Find Smaller Video File, Wait, Format Video Links
- **Node Details:**
  - **Loop Over Items**
    - Type: Split In Batches
    - Role: Iterates through batches of search terms for video search.
    - Inputs: Formatted search terms from "Format For Loop Over Items"
    - Outputs: "Format Video Links" and "Search Videos"
  - **Search Videos**
    - Type: HTTP Request
    - Role: Calls Pexels API to search videos by keyword.
    - Configuration: Authenticated with Pexels API key.
    - Inputs: Search term from "Loop Over Items"
    - Outputs: "Get The Video Details"
    - Edge cases: API limits, no results found.
  - **Get The Video Details**
    - Type: HTTP Request
    - Role: Fetches detailed metadata for each video found.
    - Inputs: Video IDs from "Search Videos"
    - Outputs: "Find Smaller Video File"
  - **Find Smaller Video File**
    - Type: Code
    - Role: Selects the smallest video file variant for bandwidth efficiency.
    - Inputs: Video details
    - Outputs: "Wait"
    - Edge cases: Missing file URLs, unexpected response formats.
  - **Wait**
    - Type: Wait
    - Role: Introduces delay to comply with API rate limits or processing requirements.
    - Inputs: From "Find Smaller Video File"
    - Outputs: "Loop Over Items"
  - **Format Video Links**
    - Type: Code
    - Role: Prepares video links for next workflow stages or output.
    - Inputs: From "Loop Over Items"
    - Outputs: "List Short Music Files"

#### 2.4 Audio Generation and Processing

- **Overview:** Generates voice narration from the script via ElevenLabs, uploads audio to Google Drive, and sets file permissions.
- **Nodes Involved:** Choose Music, Generate Voice, Google Drive, Set File Permissions, List Short Music Files
- **Node Details:**
  - **List Short Music Files**
    - Type: HTTP Request
    - Role: Retrieves a list of background music files available for the video.
    - Inputs: From "Format Video Links"
    - Outputs: "Choose Music"
  - **Choose Music**
    - Type: Code
    - Role: Selects appropriate background music based on criteria or randomness.
    - Inputs: Music list from "List Short Music Files"
    - Outputs: "Generate Voice"
  - **Generate Voice**
    - Type: HTTP Request
    - Role: Calls ElevenLabs API to synthesize speech from text.
    - Inputs: Script text, selected music
    - Outputs: "Google Drive"
    - Edge cases: API auth failure, synthesis errors.
  - **Google Drive**
    - Type: Google Drive Node
    - Role: Uploads generated audio file to Google Drive.
    - Inputs: Audio file from "Generate Voice"
    - Outputs: "Set File Permissions"
    - Edge cases: Upload failure, quota limits.
  - **Set File Permissions**
    - Type: HTTP Request
    - Role: Sets sharing permissions on the uploaded audio file.
    - Inputs: Uploaded file metadata
    - Outputs: "Create Short"

#### 2.5 Video Creation and Monitoring

- **Overview:** Creates the short video using Creatomate API, waits for processing, checks status, handles completion or retries, and deletes temporary audio.
- **Nodes Involved:** Create Short, Wait1, Check Video Status, If, Delete Audio, Finished
- **Node Details:**
  - **Create Short**
    - Type: HTTP Request
    - Role: Submits video creation request to Creatomate.
    - Inputs: Video and audio assets, script details.
    - Outputs: "Wait1"
    - Edge cases: API errors, malformed payloads.
  - **Wait1**
    - Type: Wait
    - Role: Waits before polling video creation status.
    - Inputs: From "Create Short" or "If" node
    - Outputs: "Check Video Status"
  - **Check Video Status**
    - Type: HTTP Request
    - Role: Polls Creatomate API to check if video processing is complete.
    - Inputs: Video job ID
    - Outputs: "If"
    - Edge cases: Network errors, job failure.
  - **If**
    - Type: Conditional node
    - Role: Branches workflow based on video status (finished or processing).
    - Inputs: Status from "Check Video Status"
    - Outputs: On success → "Delete Audio"; On pending → "Wait1"
  - **Delete Audio**
    - Type: Google Drive Node
    - Role: Deletes temporary audio file from Google Drive after video is done.
    - Inputs: Audio file metadata
    - Outputs: "Finished"
  - **Finished**
    - Type: Code
    - Role: Finalizes workflow, potentially logging or cleaning up.
    - Inputs: From "Delete Audio"
    - Outputs: None (end of workflow)

#### 2.6 Sticky Notes

- Multiple sticky notes are present but contain no content; they likely serve as placeholders or reminders for future documentation or enhancements.

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                                   | Input Node(s)             | Output Node(s)            | Sticky Note |
|----------------------------|-------------------------------------|-------------------------------------------------|---------------------------|---------------------------|-------------|
| On form submission          | Form Trigger                        | Entry point for receiving user form data         |                           | Short Text                |             |
| Short Text                 | Langchain Agent (AI Chat)            | Generate short video script from input text      | On form submission        | Video Search Terms        |             |
| Structured Output Parser1   | Langchain Structured Output Parser  | Parse AI script output into structured JSON      | Short Text                | Short Text                |             |
| Video Search Terms          | Langchain Agent                     | Generate keywords for video search                | Short Text                | Format For Loop Over Items|             |
| Structured Output Parser    | Langchain Structured Output Parser  | Parse AI search terms output                      | Video Search Terms        | Video Search Terms        |             |
| Format For Loop Over Items  | Code                               | Format search terms for batch processing          | Video Search Terms        | Loop Over Items           |             |
| Loop Over Items             | Split In Batches                   | Iterate over batches of search terms              | Format For Loop Over Items| Format Video Links, Search Videos |        |
| Format Video Links          | Code                               | Prepare video links for next stages                | Loop Over Items           | List Short Music Files    |             |
| Search Videos              | HTTP Request                       | Search Pexels videos using keywords                | Loop Over Items           | Get The Video Details     |             |
| Get The Video Details       | HTTP Request                       | Retrieve detailed video metadata                    | Search Videos             | Find Smaller Video File   |             |
| Find Smaller Video File     | Code                               | Select smallest video file variant                  | Get The Video Details     | Wait                      |             |
| Wait                       | Wait                               | Delay for API rate limits or processing             | Find Smaller Video File   | Loop Over Items           |             |
| List Short Music Files      | HTTP Request                       | Retrieve background music files                      | Format Video Links        | Choose Music              |             |
| Choose Music               | Code                               | Select suitable background music                     | List Short Music Files    | Generate Voice            |             |
| Generate Voice             | HTTP Request                       | Generate speech audio via ElevenLabs API             | Choose Music              | Google Drive              |             |
| Google Drive               | Google Drive                       | Upload audio file                                     | Generate Voice            | Set File Permissions      |             |
| Set File Permissions        | HTTP Request                       | Configure sharing permissions on audio file          | Google Drive              | Create Short              |             |
| Create Short               | HTTP Request                       | Submit video creation request to Creatomate          | Set File Permissions      | Wait1                     |             |
| Wait1                      | Wait                               | Wait before checking video processing status         | Create Short, If          | Check Video Status        |             |
| Check Video Status          | HTTP Request                       | Poll Creatomate API for video processing status      | Wait1                     | If                        |             |
| If                         | If                                 | Branch based on video processing completion          | Check Video Status        | Delete Audio, Wait1       |             |
| Delete Audio               | Google Drive                       | Delete temporary audio file from Google Drive         | If                        | Finished                  |             |
| Finished                   | Code                               | Finalize workflow                                      | Delete Audio              |                           |             |
| Anthropic Chat Model        | Langchain AI Model (Disabled)      | (Disabled) Alternative AI model                       |                           |                           |             |
| OpenAI Chat Model           | Langchain AI Model (Disabled)      | (Disabled) Alternative AI model                       |                           | Video Search Terms        |             |
| Structured Output Parser    | Langchain Structured Output Parser | (Duplicate or unused)                                  |                           | Video Search Terms        |             |
| Sticky Notes (multiple)     | Sticky Note                       | Visual notes (empty content)                           |                           |                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Form Trigger" node:**
   - Name: `On form submission`
   - Configure webhook with default settings.
   - No parameters needed.
   - This node will receive user input to trigger the workflow.

3. **Add a Langchain Agent node for short video script generation:**
   - Name: `Short Text`
   - Type: `@n8n/n8n-nodes-langchain.agent`
   - Connect input from `On form submission`.
   - Configure with your OpenAI credentials (Anthropic node is disabled).
   - Set prompt to generate a short video script from user input.
   - Output to `Structured Output Parser1`.

4. **Add a Langchain Structured Output Parser:**
   - Name: `Structured Output Parser1`
   - Connect input from `Short Text`.
   - Configure to parse AI output into structured JSON.
   - Output back to `Short Text` or next node.

5. **Add another Langchain Agent node for video search term generation:**
   - Name: `Video Search Terms`
   - Input from `Short Text`.
   - Configure to generate relevant keywords for video search.
   - Output to `Structured Output Parser`.

6. **Add Langchain Structured Output Parser for video search terms:**
   - Name: `Structured Output Parser`
   - Input from `Video Search Terms`.
   - Output to `Format For Loop Over Items`.

7. **Add a "Code" node to format the parsed search terms:**
   - Name: `Format For Loop Over Items`
   - Input from `Structured Output Parser`.
   - Write JavaScript to structure search terms into an array suitable for batching.
   - Output to `Loop Over Items`.

8. **Add a "Split In Batches" node:**
   - Name: `Loop Over Items`
   - Input from `Format For Loop Over Items`.
   - Output two connections:
     - To `Format Video Links`
     - To `Search Videos`

9. **Add a "Code" node to format video links:**
   - Name: `Format Video Links`
   - Input from `Loop Over Items`.
   - Transform video data for subsequent music retrieval.
   - Output to `List Short Music Files`.

10. **Add HTTP Request node to search Pexels videos:**
    - Name: `Search Videos`
    - Input from `Loop Over Items`.
    - Configure with Pexels API key authorization.
    - Use search term from batch item.
    - Output to `Get The Video Details`.

11. **Add HTTP Request node to get video details:**
    - Name: `Get The Video Details`
    - Input from `Search Videos`.
    - Retrieve detailed video metadata.
    - Output to `Find Smaller Video File`.

12. **Add a "Code" node to select the smallest video file:**
    - Name: `Find Smaller Video File`
    - Input from `Get The Video Details`.
    - Logic to pick smallest available video file URL.
    - Output to `Wait`.

13. **Add a "Wait" node:**
    - Name: `Wait`
    - Input from `Find Smaller Video File`.
    - Configure delay (e.g., seconds to respect API limits).
    - Output loops back to `Loop Over Items`.

14. **Add HTTP Request node to list short music files:**
    - Name: `List Short Music Files`
    - Input from `Format Video Links`.
    - Retrieve available music files for background.
    - Output to `Choose Music`.

15. **Add a "Code" node to choose the music file:**
    - Name: `Choose Music`
    - Input from `List Short Music Files`.
    - Select one music file based on logic or randomness.
    - Output to `Generate Voice`.

16. **Add HTTP Request node to generate voice audio:**
    - Name: `Generate Voice`
    - Input from `Choose Music`.
    - Configure ElevenLabs API with credentials.
    - Provide script text to synthesize speech.
    - Output to `Google Drive`.

17. **Add Google Drive node to upload audio:**
    - Name: `Google Drive`
    - Input from `Generate Voice`.
    - Configure Google Drive OAuth2 credentials.
    - Upload the generated audio file.
    - Output to `Set File Permissions`.

18. **Add HTTP Request node to set file permissions:**
    - Name: `Set File Permissions`
    - Input from `Google Drive`.
    - Set sharing permissions (e.g., public or shareable link).
    - Output to `Create Short`.

19. **Add HTTP Request node to create the short video:**
    - Name: `Create Short`
    - Input from `Set File Permissions`.
    - Configure Creatomate API credentials.
    - Submit video creation request with video & audio assets.
    - Output to `Wait1`.

20. **Add a "Wait" node:**
    - Name: `Wait1`
    - Input from `Create Short` and also from conditional retry branch.
    - Wait before polling video status.
    - Output to `Check Video Status`.

21. **Add HTTP Request node to check video status:**
    - Name: `Check Video Status`
    - Input from `Wait1`.
    - Poll Creatomate API for processing status.
    - Output to `If`.

22. **Add an "If" node to branch on video status:**
    - Name: `If`
    - Input from `Check Video Status`.
    - Condition: If video status is 'finished' → output 1, else output 2.
    - Output 1 to `Delete Audio`.
    - Output 2 to `Wait1` (retry loop).

23. **Add Google Drive node to delete temporary audio:**
    - Name: `Delete Audio`
    - Input from `If` (successful video).
    - Use Google Drive credentials.
    - Delete audio file used for narration.
    - Output to `Finished`.

24. **Add a "Code" node to finalize workflow:**
    - Name: `Finished`
    - Input from `Delete Audio`.
    - Optionally log completion or cleanup.
    - End of workflow.

25. **Add optional Sticky Note nodes at logical positions as placeholders or documentation aids.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                             |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| This workflow uses Creatomate API for video creation: https://creatomate.com/docs/api/                        | Video creation API documentation                                           |
| ElevenLabs API is used for voice synthesis: https://elevenlabs.io/docs/api                                   | Voice generation API documentation                                         |
| Pexels Stock videos are accessed via Pexels API: https://www.pexels.com/api/                                 | Stock video sourcing API documentation                                     |
| Google Drive OAuth2 credentials must be configured for file uploads and deletions                              | n8n Google Drive node documentation                                        |
| AI language models (OpenAI/Anthropic) require valid API keys and prompt tuning                                | OpenAI API docs: https://platform.openai.com/docs/                        |
| Use care with API rate limits and error handling for all HTTP Request nodes                                   | Implement retries and error catching in production                        |
| Empty Sticky Notes are present for future annotations or reminders                                            | Consider adding descriptive content for maintainability                   |

---

Disclaimer:  
The text provided originates exclusively from an automated workflow designed with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.