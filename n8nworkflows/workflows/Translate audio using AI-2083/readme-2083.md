Translate audio using AI

https://n8nworkflows.xyz/workflows/translate-audio-using-ai-2083


# Translate audio using AI

### 1. Workflow Overview

This workflow is designed to translate French text into spoken audio, then transcribe that audio back into text, translate it into English, and finally generate an English audio file. It leverages ElevenLabs’ text-to-speech API for audio generation and OpenAI’s Whisper and GPT models for transcription and translation.

The workflow is logically divided into the following functional blocks:

- **1.1 Trigger and Input Setup**: Receives manual trigger and initializes voice ID and French text input.
- **1.2 French Text-to-Speech Generation**: Converts French text into spoken audio using ElevenLabs.
- **1.3 Audio File Preparation**: Adds filename metadata to the generated audio for downstream processing.
- **1.4 Audio Transcription to Text**: Sends the generated audio to OpenAI Whisper API to transcribe it back into French text.
- **1.5 Text Translation to English**: Translates the French transcription into English using OpenAI language models.
- **1.6 English Text-to-Speech Generation**: Converts the English translated text into spoken audio using ElevenLabs.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Setup

- **Overview:**  
  This block starts the workflow on manual execution and sets initial variables: the ElevenLabs voice ID and the source French text to be translated and processed.

- **Nodes Involved:**  
  - When clicking "Execute Workflow" (Manual Trigger)  
  - Set ElevenLabs voice ID and text

- **Node Details:**

  - **When clicking "Execute Workflow"**  
    - Type: Manual Trigger  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Triggers the next node on user execution.  
    - Failure Modes: None (manual trigger)  

  - **Set ElevenLabs voice ID and text**  
    - Type: Set  
    - Configuration:  
      - `voice_id`: A hardcoded string representing the ElevenLabs voice ID (example: "Xb7hH8MSUJpSbSDYk0k2")  
      - `text`: Long French text string to be synthesized into audio.  
    - Inputs: Manual trigger output  
    - Outputs: JSON object with `voice_id` and `text` fields to be used downstream.  
    - Failure Modes: Incorrect voice ID will cause failures when calling ElevenLabs API.

#### 2.2 French Text-to-Speech Generation

- **Overview:**  
  Converts the provided French text into an audio file via ElevenLabs Text-to-Speech API.

- **Nodes Involved:**  
  - Generate French Audio  
  - Add Filename

- **Node Details:**

  - **Generate French Audio**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.voice_id }}`  
      - Body: JSON containing the French text, model ID `eleven_multilingual_v2`, and voice settings (stability and similarity boost set to 0.5).  
      - Headers: Includes `accept: audio/mpeg` and authenticated with ElevenLabs API key via HTTP Header authentication (credential named "ElevenLabs API Key").  
      - Query Parameter: `optimize_streaming_latency=1` for faster response.  
      - Response: Expected as audio file (MPEG).  
    - Inputs: JSON with `voice_id` and `text` from previous node.  
    - Outputs: Binary audio data representing French speech.  
    - Failure Modes:  
      - Invalid voice ID or API key will cause authorization errors.  
      - Network timeouts or API rate limits can cause failures.  

  - **Add Filename**  
    - Type: Code (JavaScript)  
    - Configuration: Adds filename metadata `audio.mp3` to the binary audio data under the `data` field.  
    - Inputs: Binary audio data from previous HTTP Request node.  
    - Outputs: Same binary data with added filename.  
    - Failure Modes: If no binary data is present, the code will fail.

#### 2.3 Audio Transcription to Text

- **Overview:**  
  Transcribes the generated French audio back into text using OpenAI Whisper model.

- **Nodes Involved:**  
  - Transcribe audio

- **Node Details:**

  - **Transcribe audio**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.openai.com/v1/audio/transcriptions`  
      - Content-Type: multipart/form-data  
      - Body Parameters:  
        - `file`: Binary audio data (`data`) as form binary data  
        - `model`: `"whisper-1"`  
      - Headers: Content-Type multipart/form-data  
      - Authentication: OpenAI API key credential (named "OpenAi account")  
    - Inputs: Binary audio from the “Add Filename” node  
    - Outputs: JSON with transcribed text (in French)  
    - Failure Modes:  
      - Invalid or expired OpenAI API key  
      - Unsupported audio format or corrupted file  
      - Network failures or API rate limits

#### 2.4 Text Translation to English

- **Overview:**  
  Translates the French transcription text into English using OpenAI GPT language model.

- **Nodes Involved:**  
  - Translate Text to English  
  - OpenAI Chat Model1 (used as language model resource)

- **Node Details:**

  - **Translate Text to English**  
    - Type: LangChain LLM Chain node (chainLlm)  
    - Configuration:  
      - Prompt: Defines a simple translation prompt `"Translate to English:\n{{ $json.text }}"`  
      - Model: Uses the linked OpenAI Chat Model node.  
    - Inputs: JSON with French text from transcription node  
    - Outputs: JSON with English translated text  
    - Failure Modes:  
      - Incorrect prompt formatting could cause translation errors  
      - OpenAI API key issues  
      - Model unavailability or timeouts  

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model  
    - Configuration:  
      - Model: `gpt-4o-mini` (a lightweight GPT-4 variant)  
      - Credentials: OpenAI API key  
    - Role: Provides language model capability to the translation chain node.  
    - Edge Cases: Model response delays, quota limits  

#### 2.5 English Text-to-Speech Generation

- **Overview:**  
  Converts the translated English text back into spoken audio using ElevenLabs text-to-speech API.

- **Nodes Involved:**  
  - Translate English text to speech

- **Node Details:**

  - **Translate English text to speech**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $('Set ElevenLabs voice ID and text').first().json.voice_id }}` (reuses voice ID from initial Set node)  
      - Body: JSON with English text (escaped quotes and trimmed), model ID `eleven_multilingual_v2`, voice settings stability and similarity boost 0.5  
      - Headers: `accept: audio/mpeg`  
      - Authentication: ElevenLabs API key via HTTP Header Auth  
      - Query Parameter: `optimize_streaming_latency=1`  
    - Inputs: JSON with English translated text from previous node  
    - Outputs: Binary audio data of English speech  
    - Failure Modes:  
      - Invalid API key or voice ID  
      - Text formatting issues  
      - Network or API rate limiting issues

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                | Input Node(s)                       | Output Node(s)                       | Sticky Note                                                                                                      |
|------------------------------|-------------------------------|-------------------------------|-----------------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                | Workflow start trigger        | None                              | Set ElevenLabs voice ID and text   |                                                                                                                 |
| Set ElevenLabs voice ID and text | Set                          | Initialize voice ID and text  | When clicking "Execute Workflow"  | Generate French Audio               | 1] In ElevenLabs, add a voice to your [voice lab](https://elevenlabs.io/voice-lab) and copy its ID. Open this node and add the ID there |
| Generate French Audio          | HTTP Request                  | Generate French speech audio  | Set ElevenLabs voice ID and text  | Add Filename                       | 2] Get your ElevenLabs API key (click your name in ElevenLabs bottom-left and choose ‘profile’). In this node, create a new header auth cred named `xi-api-key` with your API key |
| Add Filename                  | Code                         | Add filename metadata to audio| Generate French Audio             | Transcribe audio                   |                                                                                                                 |
| Transcribe audio              | HTTP Request                  | Transcribe audio to text      | Add Filename                     | Translate Text to English          | 3] In the 'credential' field of this node, create a new OpenAI cred with your [OpenAI API key](https://platform.openai.com/api-keys) |
| Translate Text to English     | LangChain LLM Chain           | Translate French to English   | Transcribe audio                 | Translate English text to speech   |                                                                                                                 |
| OpenAI Chat Model1            | LangChain OpenAI Chat Model   | Language model for translation| None (used by Translate Text to English) | Translate Text to English          |                                                                                                                 |
| Translate English text to speech | HTTP Request                  | Generate English speech audio | Translate Text to English        | None                             |                                                                                                                 |
| Sticky Note1                  | Sticky Note                   | Setup steps overview          | None                             | None                             |                                                                                                                 |
| Sticky Note                   | Sticky Note                   | Workflow purpose description  | None                             | None                             |                                                                                                                 |
| Sticky Note2                  | Sticky Note                   | ElevenLabs voice ID instructions | None                             | None                             | 1] In ElevenLabs, add a voice to your [voice lab](https://elevenlabs.io/voice-lab) and copy its ID. Open this node and add the ID there |
| Sticky Note3                  | Sticky Note                   | ElevenLabs API key instructions | None                             | None                             | 2] Get your ElevenLabs API key (click your name in the bottom-left of [ElevenLabs](https://elevenlabs.io/voice-lab) and choose ‘profile’)\n\nIn this node, create a new header auth cred. Set the name to `xi-api-key` and the value to your API key |
| Sticky Note4                  | Sticky Note                   | OpenAI credential instructions | None                             | None                             | 3] In the 'credential' field of this node, create a new OpenAI cred with your [OpenAI API key](https://platform.openai.com/api-keys) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Leave default settings.

2. **Create Set Node to Define Voice ID and French Text**  
   - Node Type: Set  
   - Add two fields:  
     - `voice_id` (string): Paste your ElevenLabs voice ID (e.g., "Xb7hH8MSUJpSbSDYk0k2")  
     - `text` (string): Paste the French text to be converted to speech.  
   - Connect Manual Trigger node output to this Set node input.

3. **Create HTTP Request Node to Generate French Audio**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.voice_id }}`  
   - Authentication: HTTP Header Auth  
     - Create new credential: Name = `xi-api-key`, Value = Your ElevenLabs API key  
   - Headers:  
     - `accept`: `audio/mpeg`  
   - Query Parameters:  
     - `optimize_streaming_latency`: `1`  
   - Body (JSON):  
     ```json
     {
       "text": "{{ $json.text }}",
       "model_id": "eleven_multilingual_v2",
       "voice_settings": {
         "stability": 0.5,
         "similarity_boost": 0.5
       }
     }
     ```  
   - Set response format to file.  
   - Connect Set node output to this HTTP Request node input.

4. **Create Code Node to Add Filename to Audio Binary**  
   - Node Type: Code  
   - Language: JavaScript  
   - Code snippet:  
     ```js
     for (const item of $input.all()) {
       item.binary.data.fileName = "audio.mp3";
     }
     return $input.all();
     ```  
   - Connect HTTP Request (French Audio) output to this node input.

5. **Create HTTP Request Node to Transcribe Audio**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.openai.com/v1/audio/transcriptions`  
   - Authentication: OpenAI API (create credential with your OpenAI API key)  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - `file`: type `formBinaryData`, input field name `data` (from binary input)  
     - `model`: `whisper-1`  
   - Headers: `Content-Type: multipart/form-data`  
   - Connect Code node output (audio with filename) to this node input.

6. **Create LangChain OpenAI Chat Model Node**  
   - Node Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini` (or any preferred GPT-4 variant)  
   - Authentication: OpenAI API key credential  
   - No input connections (used by the chain node).

7. **Create LangChain Chain LLM Node to Translate Text**  
   - Node Type: LangChain LLM Chain  
   - Prompt type: Define prompt  
   - Text:  
     ```
     Translate to English:
     {{ $json.text }}
     ```  
   - Connect input from Transcribe Audio node output.  
   - Link the OpenAI Chat Model node in the language model parameter.

8. **Create HTTP Request Node to Generate English Audio**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $('Set ElevenLabs voice ID and text').first().json.voice_id }}`  
   - Authentication: HTTP Header Auth with ElevenLabs API key (same as French Audio node)  
   - Headers: `accept: audio/mpeg`  
   - Query Parameters: `optimize_streaming_latency=1`  
   - Body (JSON):  
     ```json
     {
       "text": "{{ $json[\"text\"].replaceAll('\"', '\\\\\"').trim() }}",
       "model_id": "eleven_multilingual_v2",
       "voice_settings": {
         "stability": 0.5,
         "similarity_boost": 0.5
       }
     }
     ```  
   - Connect output from Translate Text to English node to this node input.

9. **Connect all nodes in the following order:**  
   Manual Trigger → Set ElevenLabs voice ID and text → Generate French Audio → Add Filename → Transcribe audio → Translate Text to English → Translate English text to speech.

10. **Credential Setup:**  
    - ElevenLabs API Key: Setup as HTTP Header Auth credential with header name `xi-api-key`.  
    - OpenAI API Key: Setup as OpenAI API credential for transcription and language model nodes.

11. **Run the workflow by clicking the manual trigger "Execute Workflow" button.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| In ElevenLabs, add a voice to your [voice lab](https://elevenlabs.io/voice-lab) and copy its ID.     | ElevenLabs Voice Lab for voice configuration                                                        |
| Get your ElevenLabs API key from profile under your account in ElevenLabs.                           | https://elevenlabs.io/voice-lab                                                                    |
| Create an OpenAI API key at [OpenAI API Keys](https://platform.openai.com/api-keys)                  | OpenAI API key generation and management                                                           |
| ElevenLabs text-to-speech API supports `eleven_multilingual_v2` model with voice settings           | ElevenLabs API documentation                                                                        |
| OpenAI Whisper model `whisper-1` is used for audio transcription                                    | OpenAI Whisper API                                                                                  |
| LangChain nodes in n8n enable easy integration with OpenAI models including GPT-4 variants          | https://n8n.io/integrations/langchain                                                             |
| Ensure binary data includes filename metadata for multipart uploads to OpenAI Whisper API           | Failing to set a filename can cause upload errors                                                  |

---

This document provides a detailed, structured reference for understanding, reproducing, and extending the "Translate audio using AI" workflow in n8n. It covers all nodes, configurations, edge cases, and external dependencies.