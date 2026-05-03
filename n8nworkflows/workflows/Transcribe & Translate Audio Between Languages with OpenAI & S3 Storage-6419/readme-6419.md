Transcribe & Translate Audio Between Languages with OpenAI & S3 Storage

https://n8nworkflows.xyz/workflows/transcribe---translate-audio-between-languages-with-openai---s3-storage-6419


# Transcribe & Translate Audio Between Languages with OpenAI & S3 Storage

### 1. Workflow Overview

This workflow automates the process of transcribing audio files, translating the resulting text between two specified languages, generating audio from the translated text, and storing the output audio file on AWS S3. It is designed for multilingual content creators, businesses requiring audio translation services, educational platforms, and anyone needing efficient audio-to-audio translation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives audio files and language parameters via a webhook.
- **1.2 Audio Transcription:** Uses OpenAI Whisper to transcribe audio into text.
- **1.3 Translation and Structuring:** Uses GPT-4 to clean, structure, and translate the transcribed text.
- **1.4 Audio Generation:** Converts the translated text into speech using OpenAI's text-to-speech capabilities.
- **1.5 Storage and Delivery:** Uploads the generated audio file to AWS S3 with public access and prepares the response with text and audio URL.
- **1.6 Response Delivery:** Sends the final structured and translated text along with the audio URL back to the client.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Receives the audio file and language parameters via an HTTP POST webhook, serving as the entry point of the workflow.

- **Nodes Involved:**  
  - Receive Audio File

- **Node Details:**

  - **Receive Audio File:**  
    - *Type & Role:* Webhook node that listens for incoming POST requests at the path `/audio-translator`.  
    - *Configuration:* HTTP method POST, response mode set to respond with the node output.  
    - *Key Expressions/Variables:* Extracts binary audio file and a JSON body containing `languages`.  
    - *Input/Output:* No input nodes; outputs to "Transcribe Audio to Text".  
    - *Edge Cases:* Missing audio file or malformed request, invalid languages parameter, webhook authentication not configured (security note advises setting this).  
    - *Sub-workflow:* None.

#### 1.2 Audio Transcription

- **Overview:**  
  Converts the received audio file into text using OpenAI Whisper, supporting multiple languages and accents.

- **Nodes Involved:**  
  - Transcribe Audio to Text  
  - Step 1 - Transcription (sticky note)

- **Node Details:**

  - **Transcribe Audio to Text:**  
    - *Type & Role:* OpenAI audio resource node performing transcription operation.  
    - *Configuration:* Uses binary property `audiofile` from the webhook; configured with OpenAI API credentials. Retry enabled on failure for reliability.  
    - *Key Expressions:* Accesses binary audio from webhook node.  
    - *Input/Output:* Input from "Receive Audio File"; outputs to "Translate and Structure Text".  
    - *Edge Cases:* API authentication failure, large audio files causing timeouts, unsupported audio formats.  
    - *Sub-workflow:* None.

  - **Step 1 - Transcription (Sticky Note):**  
    - Provides context and usage tips for transcription block.

#### 1.3 Translation and Structuring

- **Overview:**  
  Processes the transcribed text to remove repetitions, structure it, and translate into the target language using GPT-4.

- **Nodes Involved:**  
  - Translate and Structure Text  
  - Step 2 - Translation (sticky note)

- **Node Details:**

  - **Translate and Structure Text:**  
    - *Type & Role:* OpenAI GPT-4 language model node for text processing.  
    - *Configuration:* Model `gpt-4.1`; message instructs it to output a JSON with two fields: `structuringMessage` (cleaned original text) and `translateMessage` (translated text).  
    - *Key Expressions:*  
      - Input text: `{{ $json.text }}` from transcription node output.  
      - Languages: `{{ $('Receive Audio File').item.json.body.languages }}` from webhook input.  
    - *Input/Output:* Input from "Transcribe Audio to Text"; outputs to "Prepare Response Data".  
    - *Edge Cases:* GPT API errors, invalid or missing language parameters, malformed inputs causing JSON parsing issues.  
    - *Sub-workflow:* None.

  - **Step 2 - Translation (Sticky Note):**  
    - Explains the translation and structuring process.

#### 1.4 Audio Generation

- **Overview:**  
  Converts the translated text into an audio file using OpenAI’s text-to-speech capabilities.

- **Nodes Involved:**  
  - Prepare Response Data  
  - Generate Translated Audio  
  - Step 3 - Audio Generation (sticky note)

- **Node Details:**

  - **Prepare Response Data:**  
    - *Type & Role:* Set node that extracts and reshapes data for the next steps.  
    - *Configuration:* Assigns:  
      - `structuringMessage` and `translateMessage` from the GPT JSON output fields.  
      - `audiofilename` generated dynamically using the current timestamp sanitized to alphanumeric and appended `.mp3`.  
    - *Input/Output:* Input from "Translate and Structure Text"; outputs to "Generate Translated Audio".  
    - *Edge Cases:* Data extraction errors if GPT output format changes; timestamp formatting errors unlikely.  

  - **Generate Translated Audio:**  
    - *Type & Role:* OpenAI audio resource node for text-to-speech generation.  
    - *Configuration:* Uses the prepared `translateMessage` string as input text; configured with OpenAI API credentials.  
    - *Input/Output:* Input from "Prepare Response Data"; outputs to "Upload Audio to S3".  
    - *Edge Cases:* API limits or authentication failure, unsupported text content, audio generation errors.

  - **Step 3 - Audio Generation (Sticky Note):**  
    - Describes audio generation and customization options.

#### 1.5 Storage and Delivery

- **Overview:**  
  Uploads the generated audio file to an AWS S3 bucket with public read access for easy sharing.

- **Nodes Involved:**  
  - Upload Audio to S3  
  - Step 4 - Storage (sticky note)  
  - Security Note (sticky note)

- **Node Details:**

  - **Upload Audio to S3:**  
    - *Type & Role:* AWS S3 node to upload files.  
    - *Configuration:*  
      - Upload operation using the filename from "Prepare Response Data".  
      - Bucket name and ACL set to `publicRead` (requires replacing placeholder with actual bucket name).  
    - *Input/Output:* Input from "Generate Translated Audio"; outputs to "Send Translation Results".  
    - *Edge Cases:* Invalid or missing AWS credentials, incorrect bucket name, insufficient permissions, network issues.

  - **Step 4 - Storage (Sticky Note):**  
    - Highlights importance of configuring bucket name and region properly.

  - **Security Note (Sticky Note):**  
    - Details security requirements: replacing placeholders, webhook authentication, CORS configuration.

#### 1.6 Response Delivery

- **Overview:**  
  Sends the final response to the client, including structured and translated texts and a public URL to the generated audio file.

- **Nodes Involved:**  
  - Send Translation Results

- **Node Details:**

  - **Send Translation Results:**  
    - *Type & Role:* Respond to webhook node that returns a JSON response.  
    - *Configuration:* Constructs a JSON response with:  
      - `StructuringMessage`: structured original text  
      - `translateMessage`: translated text  
      - `audiofile`: HTTPS URL to the uploaded audio on S3 (bucket name and region placeholders to be replaced).  
    - *Input/Output:* Input from "Upload Audio to S3"; outputs HTTP response to webhook caller.  
    - *Edge Cases:* Incorrect URLs due to placeholders not replaced, response formatting errors.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                  | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                        |
|-------------------------|-------------------------------------|--------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------|
| Receive Audio File       | Webhook                             | Entry point, receives audio     | -                            | Transcribe Audio to Text     |                                                                                                                    |
| Transcribe Audio to Text | OpenAI Audio Resource (Transcribe)  | Transcribe audio to text        | Receive Audio File            | Translate and Structure Text |                                                                                                                    |
| Translate and Structure Text | OpenAI GPT-4 Language Model       | Structure and translate text    | Transcribe Audio to Text      | Prepare Response Data        |                                                                                                                    |
| Prepare Response Data    | Set                                 | Prepare data for TTS generation | Translate and Structure Text  | Generate Translated Audio    |                                                                                                                    |
| Generate Translated Audio| OpenAI Audio Resource (TTS)          | Generate audio from translated text | Prepare Response Data      | Upload Audio to S3           |                                                                                                                    |
| Upload Audio to S3       | AWS S3                              | Upload audio file to S3         | Generate Translated Audio     | Send Translation Results     | ⚠️ **Security Configuration Required** Replace bucket name, region; configure webhook auth and CORS.               |
| Send Translation Results | Respond to Webhook                  | Return final JSON response      | Upload Audio to S3            | -                           |                                                                                                                    |
| Workflow Overview       | Sticky Note                        | Describes overall workflow      | -                            | -                           |                                                                                                                    |
| Step 1 - Transcription  | Sticky Note                        | Explains transcription step    | -                            | -                           |                                                                                                                    |
| Step 2 - Translation    | Sticky Note                        | Explains translation step      | -                            | -                           |                                                                                                                    |
| Step 3 - Audio Generation| Sticky Note                        | Explains TTS step              | -                            | -                           |                                                                                                                    |
| Step 4 - Storage        | Sticky Note                        | Explains storage & delivery    | -                            | -                           |                                                                                                                    |
| Security Note           | Sticky Note                        | Security configuration warnings| -                            | -                           | ⚠️ **Security Configuration Required** Replace bucket name, region; configure webhook auth and CORS.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Receive Audio File`  
   - Type: Webhook  
   - Set HTTP Method to POST  
   - Path: `audio-translator`  
   - Response Mode: `Response Node`  
   - Expect binary audio file in the request and a JSON body parameter `languages`.

2. **Add OpenAI Audio Node for Transcription**  
   - Name: `Transcribe Audio to Text`  
   - Type: OpenAI Audio resource (Transcribe operation)  
   - Configure to use binary property from webhook containing the audio file (e.g., `audiofile`).  
   - Attach OpenAI API credentials.  
   - Enable "Retry on Fail" for robustness.  
   - Connect output of `Receive Audio File` to this node.

3. **Add OpenAI GPT-4 Node for Translation and Structuring**  
   - Name: `Translate and Structure Text`  
   - Type: OpenAI GPT-4 language model  
   - Model ID: `gpt-4.1`  
   - Set message content with instructions to:  
     - Receive input text and two languages (source and target).  
     - Clean and structure the original text removing repetitions.  
     - Translate the text into the target language.  
     - Return JSON with fields `structuringMessage` and `translateMessage`.  
   - Use expressions to:  
     - Input text: `{{ $json.text }}` from transcription output.  
     - Languages: `{{ $('Receive Audio File').item.json.body.languages }}` from webhook input.  
   - Attach OpenAI API credentials.  
   - Connect output of `Transcribe Audio to Text` to this node.

4. **Add Set Node to Prepare Data**  
   - Name: `Prepare Response Data`  
   - Type: Set  
   - Assign fields:  
     - `structuringMessage`: `={{ $json.message.content.structuringMessage }}`  
     - `translateMessage`: `={{ $json.message.content.translateMessage }}`  
     - `audiofilename`: `={{ $now.toString().replace(/[^a-zA-Z0-9]/g, '') }}.mp3` (dynamic unique filename)  
   - Connect output of `Translate and Structure Text` to this node.

5. **Add OpenAI Audio Node for Text-to-Speech**  
   - Name: `Generate Translated Audio`  
   - Type: OpenAI Audio resource (Text-to-Speech)  
   - Input text: `={{ $json.translateMessage }}` from `Prepare Response Data`  
   - Attach OpenAI API credentials.  
   - Connect output of `Prepare Response Data` to this node.

6. **Add AWS S3 Node for Uploading Audio**  
   - Name: `Upload Audio to S3`  
   - Type: AWS S3  
   - Operation: Upload  
   - Bucket Name: Replace `"YOUR-BUCKET-NAME"` with your actual bucket name  
   - File Name: `={{ $('Prepare Response Data').item.json.audiofilename }}` (dynamic filename)  
   - Additional Fields: Set ACL to `publicRead` for public access  
   - Configure AWS credentials with appropriate permissions  
   - Connect output of `Generate Translated Audio` to this node.

7. **Add Respond to Webhook Node**  
   - Name: `Send Translation Results`  
   - Type: Respond to Webhook  
   - Set response type to JSON  
   - Response body:  
     ```json
     {
       "StructuringMessage": "{{ $('Prepare Response Data').item.json.structuringMessage }}",
       "translateMessage": "{{ $('Prepare Response Data').item.json.translateMessage }}",
       "audiofile": "https://YOUR-BUCKET-NAME.s3.YOUR-REGION.amazonaws.com/{{ $('Prepare Response Data').item.json.audiofilename }}"
     }
     ```  
   - Replace placeholders with actual S3 bucket name and AWS region.  
   - Connect output of `Upload Audio to S3` to this node.

8. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Document each major step with sticky notes containing:  
     - Overview of transcription, translation, audio generation, storage, and security notes.  
     - Security note instructing to replace bucket name, region, set webhook authentication, and configure CORS.

9. **Configure Credentials**  
   - Set up OpenAI API credentials for transcription, translation, and TTS nodes.  
   - Set up AWS credentials with permissions to upload to the specified S3 bucket.

10. **Test Workflow**  
    - Send a POST request to the webhook URL `/audio-translator` with:  
      - Binary audio file in the request body.  
      - JSON `languages` parameter specifying source and target languages (e.g., `"English, Spanish"`).  
    - Verify transcription, translation, audio generation, upload, and JSON response with audio URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Replace `"YOUR-BUCKET-NAME"` and `"YOUR-REGION"` placeholders with actual S3 bucket name and AWS region in nodes and URLs.  | Critical for correct S3 upload and public URL formation.                                        |
| Ensure webhook authentication is configured to prevent unauthorized access to the audio translation service.               | Security best practice for public-facing webhooks.                                              |
| Configure CORS on the S3 bucket if the audio files will be accessed directly from browsers.                                | Prevents cross-origin access errors in web apps.                                                |
| Retry on fail is enabled for the transcription node to improve reliability with larger files or unstable API connections. | Improves robustness of transcription.                                                           |
| The workflow uses OpenAI Whisper for transcription and GPT-4 for translation and text structuring.                         | Requires OpenAI API key with appropriate access.                                               |
| Audio generation uses OpenAI's text-to-speech capabilities, which may require specific account access or API plan.         | Verify your OpenAI account supports this feature.                                              |
| For more customization, consider adding language detection, file size limits, and voice settings adjustments.             | Enhances flexibility and control over the translation and audio output.                         |
| Workflow designed for multilingual audio translation use cases including content creation, education, and business.        | Use case guidance for potential adopters.                                                      |

---

**Disclaimer:**  
This document is generated from an n8n workflow JSON export. The workflow strictly complies with content policies and manipulates only legal and public data. It contains no illegal, offensive, or protected elements.