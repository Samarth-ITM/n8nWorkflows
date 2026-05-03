Convert PDF Documents to AI Podcasts with Google Gemini and Text-to-Speech

https://n8nworkflows.xyz/workflows/convert-pdf-documents-to-ai-podcasts-with-google-gemini-and-text-to-speech-4883


# Convert PDF Documents to AI Podcasts with Google Gemini and Text-to-Speech

---

### 1. Workflow Overview

This workflow automates the conversion of PDF documents into AI-generated podcasts using Google Gemini’s language model and text-to-speech (TTS) capabilities. It is designed to take an uploaded PDF file, extract its textual content, generate a conversational podcast script of approximately 5-7 minutes in length, synthesize the script into natural-sounding audio, and save the final podcast as a WAV audio file.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & PDF Text Extraction:** Receives a PDF file upload and extracts its text content.
- **1.2 AI Podcast Script Generation:** Uses Google Gemini’s language model to generate a podcast script based on the extracted text.
- **1.3 Text-to-Speech Preparation & Audio Generation:** Prepares and sends the podcast script to Google Gemini’s TTS API to generate speech audio.
- **1.4 Audio Processing & Storage:** Processes the received audio data, fixes encoding issues, packages it as a WAV file, and saves it to disk.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & PDF Text Extraction

**Overview:**  
This block initiates the workflow upon manual trigger, accepts a PDF file upload, and extracts the textual content from the PDF for further processing.

**Nodes Involved:**  
- 🎬 Start: Upload PDF File  
- 📄 Extract Text from PDF

**Node Details:**

- **🎬 Start: Upload PDF File**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for the workflow; waits for user interaction to upload a PDF file.  
  - *Configuration:* Default manual trigger with no parameters.  
  - *Input/Output:* Outputs the uploaded PDF file binary data.  
  - *Edge Cases:* No data input. User must upload a valid PDF file. Failure if no file or wrong format uploaded.  
  - *Version:* Compatible with n8n v1.x and above.

- **📄 Extract Text from PDF**  
  - *Type:* Extract From File  
  - *Role:* Extracts text from the uploaded PDF binary data.  
  - *Configuration:* Operation set to `pdf`. Reads binary property named `data0` from the previous node (the uploaded file).  
  - *Input:* Binary PDF file from the manual trigger.  
  - *Output:* Extracted text as JSON property `text`.  
  - *Edge Cases:* Failure if PDF is encrypted, corrupted, or text extraction is unsupported for the file structure. May produce incomplete text if PDF contains scanned images instead of text layers.  
  - *Version:* n8n standard node, no special version requirements.

---

#### 1.2 AI Podcast Script Generation

**Overview:**  
This block generates a podcast script in natural conversational language from the extracted PDF text. It uses Google Gemini’s language model to create a structured podcast script with introduction, discussion points, and conclusion.

**Nodes Involved:**  
- 🤖 Generate Podcast Script  
- Google Gemini Flash 2.0

**Node Details:**

- **🤖 Generate Podcast Script**  
  - *Type:* LangChain Chain LLM Node  
  - *Role:* Sends extracted text to the Google Gemini language model to generate a podcast script.  
  - *Configuration:*  
    - Input text expression: Uses extracted PDF text (`={{ $json.text }}`).  
    - Custom prompt instructs to create a 5-7 minute podcast script (~750-1000 words) with strict formatting and style guidelines: conversational, structured, with smooth transitions, avoiding complex punctuation for TTS compatibility.  
  - *Input:* JSON text from PDF extraction.  
  - *Output:* Podcast script text (`text` property).  
  - *Edge Cases:* Possible failures include invalid text input, API rate limits, or response formatting issues. The model may generate output not fully compliant with the prompt if input text is poor or too short.  
  - *Version:* Uses LangChain integration v1.7 features.  
  - *Sub-workflow:* Invokes the Google Gemini language model node below.

- **Google Gemini Flash 2.0**  
  - *Type:* LangChain LM Chat Google Gemini  
  - *Role:* Acts as the language model backend for script generation.  
  - *Configuration:*  
    - Model set to `models/gemini-2.0-flash-001`.  
    - Credentials: Uses Google PaLM API credentials configured as `Google Gemini(PaLM) Api account`.  
  - *Input:* Receives prompt from the LangChain node.  
  - *Output:* AI-generated podcast script text.  
  - *Edge Cases:* Authentication errors if credentials expire or are invalid; API timeout or quota exceeded; incomplete or nonsensical output if model fails.  
  - *Version:* Requires Google Gemini API access and n8n integration compatible with LangChain nodes.

---

#### 1.3 Text-to-Speech Preparation & Audio Generation

**Overview:**  
This block prepares the podcast script for TTS, constructs the request body for Google Gemini’s TTS API, and sends the request to generate audio content.

**Nodes Involved:**  
- ⚙️ Prepare TTS Request  
- 🎙️ Convert Text to Speech With Gemini

**Node Details:**

- **⚙️ Prepare TTS Request**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Constructs the JSON request body for the Gemini TTS API based on the generated podcast script.  
  - *Configuration:*  
    - Extracts `text` property from input JSON (podcast script).  
    - Selects voice name (`voiceName`) defaulting to `"Kore"` if not specified.  
    - Builds request body JSON with `contents` array and `generationConfig` specifying audio-only response with voice configuration.  
  - *Input:* Podcast script text from previous node.  
  - *Output:* JSON request body for TTS API.  
  - *Edge Cases:* Script text missing or empty causes invalid request; malformed JSON or missing voice name may cause API rejection.  
  - *Version:* Compatible with n8n Code node v2.

- **🎙️ Convert Text to Speech With Gemini**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to Google Gemini’s TTS API endpoint to generate audio from text.  
  - *Configuration:*  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent`  
    - Method: POST  
    - Headers: `Content-Type: application/json`  
    - Auth: Uses Google PaLM API credentials (`Google Gemini(PaLM) Api account`).  
    - Body: JSON from previous Code node.  
  - *Input:* JSON request body.  
  - *Output:* Raw TTS response with audio content encoded in base64.  
  - *Edge Cases:* Authentication failure, network timeout, malformed request errors, quota exceeded, or unsupported voice name.  
  - *Version:* Requires Google Gemini TTS API access and n8n HTTP Request node v4.2+.

---

#### 1.4 Audio Processing & Storage

**Overview:**  
Processes the TTS audio response by decoding base64 data, creating a proper WAV file header, concatenating audio data, and saving the resulting audio file to disk.

**Nodes Involved:**  
- 🔧 Process Audio Response  
- 💾 Save Podcast Audio

**Node Details:**

- **🔧 Process Audio Response**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Decodes base64 encoded audio from Gemini TTS response, creates a WAV PCM16 header, concatenates header and audio bytes, and prepares a binary audio file.  
  - *Configuration:*  
    - Checks if TTS response `finishReason` is `STOP` to confirm success.  
    - Extracts audio base64 data and MIME type.  
    - Decodes base64 to buffer, creates standard WAV header for 24kHz, mono, 16-bit PCM audio.  
    - Returns JSON metadata and binary audio data with proper file name (`tts_fixed_<timestamp>.wav`).  
  - *Input:* Raw JSON from TTS HTTP response.  
  - *Output:* JSON success flag and binary WAV file data.  
  - *Edge Cases:* Errors during base64 decoding, unexpected response format, or TTS failure indicated by other finish reasons. Logs debugging info to console.  
  - *Version:* Uses Node.js Buffer API; requires n8n Code node v2.

- **💾 Save Podcast Audio**  
  - *Type:* Write Binary File  
  - *Role:* Saves the processed WAV audio binary to disk using generated file name.  
  - *Configuration:*  
    - File name set dynamically from JSON `filename` property.  
    - Reads binary audio data from property `audio`.  
  - *Input:* Binary WAV file from previous node.  
  - *Output:* Writes file to n8n server file system.  
  - *Edge Cases:* File system permission errors, disk space limitations, invalid file name characters.  
  - *Version:* Standard n8n node v1.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                           | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                                                    |
|------------------------------|----------------------------------|-----------------------------------------|--------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| 🎬 Start: Upload PDF File     | Manual Trigger                   | Entry point; upload PDF file             | —                              | 📄 Extract Text from PDF        | ## 🎯 Main Template Overview<br>- Extracts text from uploaded PDF files<br>- Generates conversational podcast script using AI<br>- Converts script to natural-sounding audio<br>- Saves final podcast as WAV file<br><br>Setup required:<br>- Google Gemini API key (for AI script generation)<br>- Google PaLM API credentials (for TTS)<br>- File upload capability |
| 📄 Extract Text from PDF      | Extract From File                | Extracts text content from PDF           | 🎬 Start: Upload PDF File       | 🤖 Generate Podcast Script      |                                                                                                                                                |
| 🤖 Generate Podcast Script    | LangChain Chain LLM             | Generate podcast script from extracted text | 📄 Extract Text from PDF        | ⚙️ Prepare TTS Request          |                                                                                                                                                |
| Google Gemini Flash 2.0       | LangChain LM Chat Google Gemini | Language model backend for script generation | 🤖 Generate Podcast Script (ai_languageModel) | 🤖 Generate Podcast Script (ai_languageModel) |                                                                                                                                                |
| ⚙️ Prepare TTS Request         | Code                            | Build TTS request JSON for Gemini API   | 🤖 Generate Podcast Script      | 🎙️ Convert Text to Speech With Gemini |                                                                                                                                                |
| 🎙️ Convert Text to Speech With Gemini | HTTP Request                    | Send TTS request to Gemini API           | ⚙️ Prepare TTS Request          | 🔧 Process Audio Response       |                                                                                                                                                |
| 🔧 Process Audio Response      | Code                            | Decode and fix audio data, create WAV file | 🎙️ Convert Text to Speech With Gemini | 💾 Save Podcast Audio           |                                                                                                                                                |
| 💾 Save Podcast Audio          | Write Binary File               | Save WAV audio file to disk               | 🔧 Process Audio Response       | —                              |                                                                                                                                                |
| Sticky Note                  | Sticky Note                    | Overview and purpose comments             | —                              | —                              | ## 🎯 Main Template Overview<br>What this workflow does:<br>- Extracts text from uploaded PDF files<br>- Generates conversational podcast script using AI<br>- Converts script to natural-sounding audio<br>- Saves final podcast as WAV file<br><br>Setup required:<br>- Google Gemini API key (for AI script generation)<br>- Google PaLM API credentials (for TTS)<br>- File upload capability |
| Sticky Note1                 | Sticky Note                    | Setup instructions and test notes         | —                              | —                              | ## 🔧 Setup Instructions<br>Before You Start:<br><br>Get API Keys:<br>- Gemini API: https://aistudio.google.com/<br>- Add to n8n credentials as "Google Gemini(PaLM) Api account"<br><br>Test the workflow:<br>- Use the manual trigger<br>- Upload a PDF file<br>- Check that audio file is generated |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `🎬 Start: Upload PDF File`  
   - Type: Manual Trigger  
   - No parameters needed. This will be the workflow entry point.

2. **Add Extract From File Node**  
   - Name: `📄 Extract Text from PDF`  
   - Type: Extract From File  
   - Set operation to `pdf`.  
   - Set binary property name to `data0` (default for uploaded file binary).  
   - Connect output of manual trigger node to this node.

3. **Add LangChain Chain LLM Node**  
   - Name: `🤖 Generate Podcast Script`  
   - Type: LangChain Chain LLM (`@n8n/n8n-nodes-langchain.chainLlm`)  
   - Set `text` input to expression: `={{ $json.text }}` (the text extracted from PDF).  
   - In `messages`, define a system prompt that instructs creating a podcast script of 5-7 minutes (approx. 750-1000 words) with the specified style and structure as per the original prompt (introduction, discussion points, conclusion, natural conversational tone, TTS-friendly punctuation).  
   - Connect output of extract text node to this node.

4. **Add Google Gemini LM Chat Node**  
   - Name: `Google Gemini Flash 2.0`  
   - Type: LangChain LM Chat Google Gemini (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)  
   - Set model name to `models/gemini-2.0-flash-001`.  
   - Add credentials: Create and assign Google PaLM API credentials named `Google Gemini(PaLM) Api account` with valid API key from https://aistudio.google.com/.  
   - Connect the `ai_languageModel` output of this node back to `🤖 Generate Podcast Script` node’s language model input parameter.

5. **Add Code Node to Prepare TTS Request**  
   - Name: `⚙️ Prepare TTS Request`  
   - Type: Code (JavaScript)  
   - Use the following logic:  
     ```javascript
     const script = $input.first().json.text;
     const voiceName = $input.first().json.voiceName || "Kore";

     return {
       json: {
         requestBody: {
           contents: [{
             parts: [{
               text: script
             }]
           }],
           generationConfig: {
             responseModalities: ["AUDIO"],
             speechConfig: {
               voiceConfig: {
                 prebuiltVoiceConfig: {
                   voiceName: voiceName
                 }
               }
             }
           }
         }
       }
     };
     ```
   - Connect output of `🤖 Generate Podcast Script` node to this node.

6. **Add HTTP Request Node for Gemini TTS**  
   - Name: `🎙️ Convert Text to Speech With Gemini`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent`  
   - Headers: Add header `Content-Type: application/json`  
   - Authentication: Use the same `Google Gemini(PaLM) Api account` credentials.  
   - Body Content: Set to JSON and use the output from the previous Code node (`requestBody`).  
   - Connect output of `⚙️ Prepare TTS Request` to this node.

7. **Add Code Node to Process Audio Response**  
   - Name: `🔧 Process Audio Response`  
   - Type: Code (JavaScript)  
   - Use the provided JavaScript code that:  
     - Checks the TTS response success (`finishReason === "STOP"`).  
     - Decodes base64 inline audio data.  
     - Creates a WAV file header for PCM16 24kHz mono audio.  
     - Concatenates header and audio bytes.  
     - Returns JSON metadata and binary audio data named `audio`.  
   - Connect output of TTS HTTP Request node to this node.

8. **Add Write Binary File Node**  
   - Name: `💾 Save Podcast Audio`  
   - Type: Write Binary File  
   - File Name: Use expression `={{ $json.filename }}` from previous node.  
   - Data Property Name: `audio` (binary audio data property).  
   - Connect output of audio processing node to this node.

9. **Add Sticky Notes (Optional for clarity)**  
   - Add notes describing workflow purpose and setup instructions.  
   - Position them near relevant nodes for user reference.

10. **Test the Workflow**  
    - Run manual trigger, upload a PDF file.  
    - Monitor each node’s output to verify text extraction, script generation, audio synthesis, and file saving.  
    - Confirm WAV file is generated and playable.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| Google Gemini API key is required for both AI script generation and Text-to-Speech synthesis.                                                                  | https://aistudio.google.com/                   |
| The podcast script prompt emphasizes conversational style, natural flow, and TTS-friendly punctuation to improve speech clarity and engagement in audio output. | Prompt details in `🤖 Generate Podcast Script` node |
| Audio data returned by Gemini TTS API is base64 encoded and requires decoding and WAV header construction to produce a proper playable audio file.            | See code in `🔧 Process Audio Response` node   |
| This workflow assumes text-based PDFs; scanned image PDFs or encrypted files may not extract text correctly.                                                    | PDF extraction limitations                      |
| TTS voice default is set to "Kore", but can be customized by modifying the `voiceName` property before TTS request preparation.                                | `⚙️ Prepare TTS Request` node                   |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated process created with n8n, an integration and automation tool. This processing complies strictly with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---