Generate AI Music with the ElevenLabs API and upload to Google Drive

https://n8nworkflows.xyz/workflows/generate-ai-music-with-the-elevenlabs-api-and-upload-to-google-drive-7793


# Generate AI Music with the ElevenLabs API and upload to Google Drive

### 1. Workflow Overview

This workflow enables users to generate AI-composed music tracks via a web form and the ElevenLabs Music API, then automatically uploads the resulting MP3 file to Google Drive and provides a styled web page with a link to listen to the generated music. It is designed for content creators, musicians, or developers who want to quickly produce custom music tracks based on textual descriptions without manual audio production.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Capture user input describing desired music style via a web form.
- **1.2 API Key Injection:** Inject the ElevenLabs API key needed for authentication.
- **1.3 AI Music Composition:** Call the ElevenLabs API to generate the music based on user input.
- **1.4 Upload to Google Drive:** Upload the generated MP3 to the authenticated Google Drive.
- **1.5 Response Preparation:** Generate a responsive HTML page with a link to the uploaded music file.
- **1.6 User Response Delivery:** Return the prepared HTML page to the user via form response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block presents a web form to the user to collect a textual description of the music they want to create. It initiates the workflow when the form is submitted.

- **Nodes Involved:**  
  - AI Music Generator

- **Node Details:**  
  - **AI Music Generator**  
    - Type: Form Trigger (n8n-nodes-base.formTrigger)  
    - Role: Webhook entry point capturing user input via an HTML form  
    - Configuration:  
      - Webhook path: `/music-generator`  
      - Form title: "AI Music Generator"  
      - Form description: "What kind of music do you want to create?"  
      - Field: "Music" (required), placeholder example "epic cinematic music, instrumental, fantasy style"  
      - Submit button labeled "Submit"  
      - Appends attribution automatically  
    - Inputs: HTTP form submission  
    - Outputs: JSON with `Music` field containing user input  
    - Version: 2.2  
    - Edge cases:  
      - Missing or empty input (form requires field, so unlikely unless bypassed)  
      - Network or webhook errors  

#### 2.2 API Key Injection

- **Overview:**  
  This block provides the ElevenLabs API key as a variable for downstream API authentication.

- **Nodes Involved:**  
  - API Key

- **Node Details:**  
  - **API Key**  
    - Type: Set (n8n-nodes-base.set)  
    - Role: Inserts the ElevenLabs API key into the workflow data  
    - Configuration:  
      - Sets a string variable `ELEVENLABS_API_KEY` with user-supplied API key (to be replaced by user)  
    - Inputs: Receives data from form trigger node  
    - Outputs: Passes data enriched with `ELEVENLABS_API_KEY`  
    - Version: 3.4  
    - Edge cases:  
      - Missing or invalid API key will cause downstream API request failures  
      - User must replace placeholder "your key" with a valid key  
    - Notes: Link to ElevenLabs API key signup: https://try.elevenlabs.io/api-music

#### 2.3 AI Music Composition

- **Overview:**  
  This block sends a POST request to the ElevenLabs Music API to generate an MP3 music file based on the user's textual description.

- **Nodes Involved:**  
  - elevenlabs_api

- **Node Details:**  
  - **elevenlabs_api**  
    - Type: HTTP Request (n8n-nodes-base.httpRequest)  
    - Role: Calls external ElevenLabs API to generate music  
    - Configuration:  
      - URL: `https://api.elevenlabs.io/v1/music/compose`  
      - Method: POST  
      - Headers: Includes `xi-api-key` set to `ELEVENLABS_API_KEY` from workflow data  
      - Body parameters:  
        - `prompt`: set dynamically from form input `Music`  
        - `output_format`: fixed to `mp3_44100_128` (MP3 format, 44.1kHz, 128 kbps)  
      - Sends both headers and body encoded as parameters  
    - Inputs: Receives data enriched with API key and user prompt  
    - Outputs: Receives API response including a downloadable music file (likely as binary or URL, depending on API)  
    - Version: 4.2  
    - Edge cases:  
      - Authentication failures due to invalid or missing API key  
      - API rate limits or quota exceeded  
      - Network or timeout errors  
      - Invalid prompt causing API errors  
    - Notes: Official API docs: https://try.elevenlabs.io/api-music

#### 2.4 Upload to Google Drive

- **Overview:**  
  This block uploads the generated MP3 music file to the user's Google Drive root folder, making it accessible for sharing.

- **Nodes Involved:**  
  - Upload mp3

- **Node Details:**  
  - **Upload mp3**  
    - Type: Google Drive (n8n-nodes-base.googleDrive)  
    - Role: Uploads a file to Google Drive  
    - Configuration:  
      - Uploads file named `music.mp3`  
      - Drive: "My Drive" (default user drive)  
      - Folder: root folder (`/ (Root folder)`)  
      - File content source: output from ElevenLabs API node (expects binary data or file URL converted to binary)  
    - Inputs: Receives generated music file from API node  
    - Outputs: Metadata about the uploaded file including links  
    - Credentials: Uses OAuth2 credentials configured as "Google Drive inforeole"  
    - Version: 3  
    - Edge cases:  
      - Credential expiration or invalid authorization  
      - Upload failures due to quota limits or network issues  
      - File size restrictions  
    - Notes: User must authenticate Google Drive OAuth2 for this node

#### 2.5 Response Preparation

- **Overview:**  
  This block generates an HTML response page that visually informs the user their music is ready and provides a clickable link to listen to it.

- **Nodes Involved:**  
  - prepare reponse

- **Node Details:**  
  - **prepare reponse**  
    - Type: HTML (n8n-nodes-base.html)  
    - Role: Creates a styled HTML page with music title and playback link  
    - Configuration:  
      - Static HTML with CSS styling for dark theme and gradient text  
      - Displays a music note emoji, heading "Your Music is Ready!", and a subtext  
      - Shows track title from JSON field `trackTitle` or defaults to "Cosmic Melody"  
      - Provides a link (`webViewLink`) to the uploaded file for playback  
      - Uses expressions `{{ $json.trackTitle || "Cosmic Melody" }}` and `{{ $json.webViewLink }}` to inject data dynamically  
    - Inputs: Receives metadata from Google Drive upload node, expecting `webViewLink` to the uploaded file  
    - Outputs: HTML content as string to be sent to user  
    - Version: 1.2  
    - Edge cases:  
      - Missing or invalid link data may cause broken or empty links  
      - Expression evaluation errors if expected fields absent

#### 2.6 User Response Delivery

- **Overview:**  
  This final block returns the prepared HTML page back to the submitting user through the original form trigger response.

- **Nodes Involved:**  
  - display mp3

- **Node Details:**  
  - **display mp3**  
    - Type: Form (n8n-nodes-base.form)  
    - Role: Sends a response to the form trigger HTTP request with the prepared HTML embedded  
    - Configuration:  
      - Operation: Completion (respond after workflow finishes)  
      - Respond With: Show text (renders the HTML content)  
      - Response Text: injected dynamically from previous node's HTML output (`{{ $json.html }}`)  
      - Webhook ID linked to the form trigger node for seamless response  
    - Inputs: Receives HTML content from prepare response node  
    - Outputs: HTTP response to user  
    - Version: 1  
    - Edge cases:  
      - Failure to render or send response (network issues)  
      - Malformed HTML causing rendering issues in browser

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                         | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                                                |
|--------------------|---------------------|---------------------------------------|----------------------|----------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| AI Music Generator  | Form Trigger        | Capture user input via web form       | (Trigger)            | API Key              |                                                                                                                                             |
| API Key            | Set                 | Inject ElevenLabs API key             | AI Music Generator   | elevenlabs_api       | Get your ElevenLabs Music API key : https://try.elevenlabs.io/api-music                                                                    |
| elevenlabs_api     | HTTP Request        | Call ElevenLabs API to generate music| API Key              | Upload mp3           | https://try.elevenlabs.io/api-music                                                                                                        |
| Upload mp3         | Google Drive        | Upload generated MP3 to Google Drive | elevenlabs_api       | prepare reponse      |                                                                                                                                             |
| prepare reponse    | HTML                | Prepare styled HTML response page     | Upload mp3           | display mp3          |                                                                                                                                             |
| display mp3        | Form                | Return HTML response to user          | prepare reponse      | (Response)           |                                                                                                                                             |
| Sticky Note        | Sticky Note         | Overview and setup instructions       |                      |                      | ## Generate music with the ElevenLabs API and upload to Google Drive<br>### What it does<br>This workflow provides a simple web form to generate unique, AI-powered music tracks using the ElevenLabs API. Enter a text description of the music you envision, and the workflow will compose it, save the MP3 file to your Google Drive, and instantly provide a link to listen to your creation.<br>### How to set up<br>1.  **Set API Key**: Get your API key from your [ElevenLabs account](https://try.elevenlabs.io/api-music) and paste it into the "API Key" node.<br>2.  **Connect Google Drive**: Authenticate your Google Drive account in the "Upload mp3" node.<br>3.  **Activate**: Save and activate the workflow to enable the Form Trigger URL. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node "AI Music Generator"**  
   - Type: Form Trigger  
   - Parameters:  
     - Webhook path: `music-generator`  
     - Form title: "AI Music Generator"  
     - Form description: "What kind of music do you want to create?"  
     - Add one required form field:  
       - Field label: "Music"  
       - Placeholder: "Describe your music (e.g., epic cinematic music, instrumental, fantasy style)"  
     - Submit button label: "Submit"  
     - Enable append attribution  
   - Position: Left side (e.g., X: -272, Y: 0)

2. **Create Set Node "API Key"**  
   - Type: Set  
   - Add assignment:  
     - Variable name: `ELEVENLABS_API_KEY`  
     - Type: String  
     - Value: Your ElevenLabs API key (replace `"your key"`)  
   - Connect input from "AI Music Generator"  
   - Position: Near form node (e.g., X: -64, Y: 0)

3. **Create HTTP Request Node "elevenlabs_api"**  
   - Type: HTTP Request  
   - Configure:  
     - URL: `https://api.elevenlabs.io/v1/music/compose`  
     - Method: POST  
     - Headers: Add header `xi-api-key` with value `={{ $json.ELEVENLABS_API_KEY }}`  
     - Body parameters (as parameters):  
       - `prompt`: `={{ $('AI Music Generator').item.json.Music }}`  
       - `output_format`: `mp3_44100_128`  
     - Send body and headers as form parameters  
   - Connect input from "API Key" node  
   - Position: Right of API Key node (e.g., X: 160, Y: 0)

4. **Create Google Drive Node "Upload mp3"**  
   - Type: Google Drive  
   - Configure:  
     - Operation: Upload file  
     - File name: `music.mp3`  
     - Drive: My Drive (default)  
     - Folder: Root folder (`/ (Root folder)`)  
     - File content: Use binary data from previous node (map as needed)  
   - Authenticate Google Drive OAuth2 credentials (create or select existing)  
   - Connect input from "elevenlabs_api" node  
   - Position: Right of HTTP Request node (e.g., X: 368, Y: 0)

5. **Create HTML Node "prepare reponse"**  
   - Type: HTML  
   - Configure HTML content as provided:  
     - Dark-themed styled page with heading, track title, and a link to the uploaded music using expressions:  
       - `{{ $json.trackTitle || "Cosmic Melody" }}` for music title  
       - `{{ $json.webViewLink }}` for link URL  
   - Connect input from "Upload mp3" node  
   - Position: Right of Google Drive node (e.g., X: 576, Y: 0)

6. **Create Form Node "display mp3"**  
   - Type: Form  
   - Parameters:  
     - Operation: Completion  
     - Respond with: Show Text  
     - Response text: `={{ $json.html }}` (use expression to inject HTML)  
     - Link this form node to the webhook of "AI Music Generator" for response  
   - Connect input from "prepare reponse" node  
   - Position: Rightmost (e.g., X: 784, Y: 0)

7. **Optional: Add Sticky Note**  
   - Add a sticky note with instructions and overview as in the original workflow's note content  
   - Position suitably (e.g., top-left)

8. **Final Steps:**  
   - Save workflow  
   - Activate workflow  
   - Share or use the webhook URL exposed by "AI Music Generator" node for users to submit music descriptions

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Get your ElevenLabs Music API key from your ElevenLabs account to enable music generation.                                                                                                                                                     | https://try.elevenlabs.io/api-music                 |
| Google Drive OAuth2 credentials must be configured and authorized with sufficient scopes to upload files.                                                                                                                                     | Google Drive API documentation                      |
| The workflow returns a visually styled dark-themed HTML page with gradient text and a playback link for user convenience.                                                                                                                    | —                                                  |
| The ElevenLabs API supports various output formats; this workflow uses MP3 at 44.1kHz and 128 kbps bitrate.                                                                                                                                   | https://try.elevenlabs.io/api-music                 |
| Ensure your API key is kept secure and not exposed publicly.                                                                                                                                                                                   | Security best practices                             |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This treatment strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.