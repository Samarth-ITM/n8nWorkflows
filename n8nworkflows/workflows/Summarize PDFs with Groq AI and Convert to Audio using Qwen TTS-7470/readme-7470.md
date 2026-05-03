Summarize PDFs with Groq AI and Convert to Audio using Qwen TTS

https://n8nworkflows.xyz/workflows/summarize-pdfs-with-groq-ai-and-convert-to-audio-using-qwen-tts-7470


# Summarize PDFs with Groq AI and Convert to Audio using Qwen TTS

### 1. Workflow Overview

This workflow automates the process of summarizing PDF documents and converting the summary into audio. It is designed to receive PDF files via a webhook, extract text content, generate a concise summary using an AI language model (Groq AI), convert the summary to speech via Qwen TTS, and return both the textual summary and audio URL to the requester.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and PDF Extraction:** Receives a PDF file through a webhook and extracts the text content from it.
- **1.2 AI Summarization:** Processes the extracted text with Groq AI via Langchain, using memory buffering to speed up repeated requests.
- **1.3 Text-to-Speech Conversion:** Sends the AI-generated summary to Qwen TTS to convert text to audio, then polls for the audio URL.
- **1.4 Response Assembly and Delivery:** Collects the summary and audio URL and responds back via the webhook.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and PDF Extraction

**Overview:**  
This block handles receiving the PDF file upload and extracting raw text from it for further processing.

**Nodes Involved:**  
- Webhook  
- Extract from File  
- Code2 (Python)  

**Node Details:**  

- **Webhook**  
  - Type: HTTP Webhook Trigger  
  - Role: Entry point; listens for POST requests at path `/summarize-pdf`  
  - Configuration: Response mode is set to respond with the last node's output  
  - Inputs: External HTTP POST request carrying the PDF file in binary form  
  - Outputs: Binary PDF file data forwarded to extraction node  
  - Edge Cases: Invalid file uploads, unsupported file types, large file timeouts

- **Extract from File**  
  - Type: File Extraction (PDF)  
  - Role: Extract text content from the uploaded PDF binary  
  - Configuration: Operation set to `pdf`, binary input property named `file` (expression)  
  - Inputs: Binary PDF from Webhook  
  - Outputs: Extracted text content as JSON under `text` property  
  - Edge Cases: Corrupted PDF, extraction failures

- **Code2 (Python)**  
  - Type: Code (Python)  
  - Role: Converts extracted text into a standard JSON data structure for AI processing  
  - Configuration: Reads `text` property from input JSON and returns it as `data` in output JSON  
  - Inputs: JSON with extracted text  
  - Outputs: JSON with `data` property containing the extracted text string  
  - Edge Cases: Missing `text` property, empty text

---

#### 1.2 AI Summarization

**Overview:**  
This block summarizes the extracted PDF text into a concise summary using Groq AI with Langchain’s AI agent and a simple memory buffer to optimize performance.

**Nodes Involved:**  
- Groq Chat Model1  
- Simple Memory  
- AI Agent1  
- summary (Set node)  

**Node Details:**  

- **Groq Chat Model1**  
  - Type: Langchain AI Chat Model (Groq GPT-OSS 20B)  
  - Role: Provides the language model for AI summarization  
  - Configuration: Model set to `openai/gpt-oss-20b`; uses Groq API credentials  
  - Inputs: Connected to AI Agent1 as language model provider  
  - Outputs: AI-generated text responses  
  - Edge Cases: API authentication failures, rate limits, model unavailability

- **Simple Memory**  
  - Type: Langchain Memory Buffer (Window)  
  - Role: Stores recent AI interactions to avoid redundant processing and speed up repeated queries  
  - Configuration: Default window memory buffer  
  - Inputs: Connected as AI memory for AI Agent1  
  - Outputs: Supplies memory context to AI Agent1  
  - Edge Cases: Memory overflow, persistence issues

- **AI Agent1**  
  - Type: Langchain AI Agent  
  - Role: Executes prompt to summarize the extracted text  
  - Configuration: Prompt defined as `Summarize the data into 4-5 lines or less : {{ $json.data }}`; output parser enabled  
  - Inputs: Receives input JSON containing extracted text as `data` property; uses memory and language model from connected nodes  
  - Outputs: JSON summary string under `output` property  
  - Edge Cases: Parsing errors, empty input data, API errors

- **summary (Set node)**  
  - Type: Set  
  - Role: Formats the AI agent's output into a JSON object with a `summary` field for downstream use  
  - Configuration: Raw JSON output set to `{"summary": "<AI output>"}` using expression on AI Agent1's output property  
  - Inputs: AI Agent1 output JSON  
  - Outputs: JSON object with `summary` property  
  - Edge Cases: Empty or malformed AI output

---

#### 1.3 Text-to-Speech Conversion

**Overview:**  
This block sends the summary text to Qwen TTS API to generate speech audio and polls the service until an audio URL is available.

**Nodes Involved:**  
- TTS Request  
- TTS Poll  
- Extract Audio URL (Code)  

**Node Details:**  

- **TTS Request**  
  - Type: HTTP Request  
  - Role: Sends POST request to Hugging Face Qwen TTS demo API with summary text and voice parameter  
  - Configuration: URL set to `https://qwen-qwen-tts-demo.hf.space/gradio_api/call/predict`; JSON body contains `[summary, "Dylan"]`  
  - Inputs: JSON with `summary` property from previous block  
  - Outputs: JSON response containing an `event_id` for polling  
  - Edge Cases: Network failures, API rate limits, invalid summary data

- **TTS Poll**  
  - Type: HTTP Request  
  - Role: Polls the Qwen TTS API using the received `event_id` to check if the audio generation is complete  
  - Configuration: URL dynamically constructed as `https://qwen-qwen-tts-demo.hf.space/gradio_api/call/predict/{{ $json.event_id }}`  
  - Inputs: JSON with `event_id` from TTS Request  
  - Outputs: JSON or SSE data containing audio URL or status  
  - Edge Cases: Polling timeouts, invalid event_id, API errors

- **Extract Audio URL (Code)**  
  - Type: Code (JavaScript)  
  - Role: Parses the polling response to extract the final audio URL  
  - Configuration: JavaScript logic to handle multiple response formats, including raw SSE text and parsed JSON arrays, returning `audioUrl` or error info  
  - Inputs: Polling response JSON or raw text  
  - Outputs: JSON with `audioUrl` or error message  
  - Edge Cases: JSON parse errors, missing URL, unexpected response formats

---

#### 1.4 Response Assembly and Delivery

**Overview:**  
This block assembles the summary and audio URL into a single JSON response and sends it back through the webhook response.

**Nodes Involved:**  
- Edit Fields (Set)  
- Respond with Both  

**Node Details:**  

- **Edit Fields (Set)**  
  - Type: Set  
  - Role: Combines the extracted audio URL and summary into one JSON object for response  
  - Configuration: Assigns `audioUrl` from the previous node and `summary` from the `summary` node’s JSON output using expressions  
  - Inputs: JSON with `audioUrl` from Extract Audio URL and JSON with `summary` from summary node  
  - Outputs: JSON containing both `summary` and `audioUrl` fields  
  - Edge Cases: Missing audio URL or summary, expression evaluation errors

- **Respond with Both**  
  - Type: Respond to Webhook  
  - Role: Sends the combined JSON response back to the original webhook caller  
  - Configuration: Responds with JSON containing `summary` and `audioUrl` fields using expressions  
  - Inputs: JSON from Edit Fields  
  - Outputs: HTTP response to requestor  
  - Edge Cases: Response formatting errors, webhook connection issues

---

### 3. Summary Table

| Node Name           | Node Type                        | Functional Role                                  | Input Node(s)         | Output Node(s)     | Sticky Note                                                                                  |
|---------------------|---------------------------------|-------------------------------------------------|-----------------------|--------------------|----------------------------------------------------------------------------------------------|
| Webhook             | HTTP Webhook Trigger             | Entry point for PDF upload                       | —                     | Extract from File   | - This will extract the pdf file from the frontend.                                         |
| Extract from File    | Extract from File (PDF)          | Extracts text from uploaded PDF                  | Webhook               | Code2              |                                                                                              |
| Code2               | Code (Python)                   | Converts extracted text to JSON property `data` | Extract from File      | AI Agent1          | - Use code node to get the data into the json output.                                       |
| Groq Chat Model1     | Langchain AI Chat Model (Groq)  | Provides Groq AI model for summarization         | —                     | AI Agent1 (lmChat) | - Use AI Agent with groq model (Open AI API) using free tokens and integrating simple memory |
| Simple Memory       | Langchain Memory Buffer          | Stores recent AI interactions to optimize calls | —                     | AI Agent1 (memory) |                                                                                              |
| AI Agent1            | Langchain AI Agent               | Performs text summarization                       | Code2, Groq Chat Model1, Simple Memory | summary (Set) |                                                                                              |
| summary              | Set                             | Formats AI output as JSON with `summary` field  | AI Agent1              | TTS Request        |                                                                                              |
| TTS Request          | HTTP Request                    | Sends summary to Qwen TTS API                     | summary                | TTS Poll           | - Use Hugging Face Qwen TTS demo for converting text to speech by using the summary & generate the audio. |
| TTS Poll             | HTTP Request                    | Polls Qwen TTS API for audio generation status   | TTS Request            | Extract Audio URL   |                                                                                              |
| Extract Audio URL    | Code (JavaScript)               | Extracts audio URL from polling response          | TTS Poll               | Edit Fields        | - Use code node to extract the audio url and send both the response into the webhook to generate the summary and audio on the frontend. |
| Edit Fields          | Set                             | Combines summary and audio URL for response       | Extract Audio URL, summary | Respond with Both |                                                                                              |
| Respond with Both    | Respond to Webhook               | Sends JSON response with summary and audio URL   | Edit Fields            | —                  |                                                                                              |
| Sticky Note          | Sticky Note                    | Project title and description                      | —                     | —                  | ## SMART PDF SUMMARIZER WITH AUDIO PLAYBACK                                                  |
| Sticky Note1         | Sticky Note                    | Comments on PDF extraction step                    | —                     | —                  | - This will extract the pdf file from the frontend.                                         |
| Sticky Note2         | Sticky Note                    | Comments on AI summarization step                  | —                     | —                  | - Use code node to get the data into the json output. Use AI Agent with groq model (Open AI API) using free tokens and integrating simple memory to store the response (getting same results faster) |
| Sticky Note3         | Sticky Note                    | Comments on TTS conversion step                     | —                     | —                  | - Use Hugging Face Qwen TTS demo for converting text to speech by using the summary & generate the audio. |
| Sticky Note4         | Sticky Note                    | Comments on response assembly step                  | —                     | —                  | - Use code node to extract the audio url and send both the response into the webhook to generate the summary and audio on the frontend. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - HTTP Method: POST  
   - Path: `summarize-pdf`  
   - Response Mode: Last Node  
   - No authentication required (adjust as needed)  

2. **Add Extract from File Node**  
   - Type: Extract from File  
   - Operation: PDF  
   - Binary Property Name: `=file` (expression referencing binary file from webhook)  
   - Connect Webhook output to this node input  

3. **Add Code Node (Python)**  
   - Type: Code (Python)  
   - Code:  
     ```python
     data = _input.first().json.text
     return {"data": data}
     ```  
   - Connect Extract from File output to this node input  

4. **Add Groq Chat Model Node**  
   - Type: Langchain AI Chat Model (Groq)  
   - Model: `openai/gpt-oss-20b`  
   - Credentials: Configure Groq API credentials with your account information  
   - No inputs; will connect to AI Agent  

5. **Add Simple Memory Node**  
   - Type: Langchain Memory Buffer Window  
   - Default settings  
   - No inputs; will connect to AI Agent  

6. **Add AI Agent Node**  
   - Type: Langchain AI Agent  
   - Text Prompt: `Summarize the data into 4-5 lines or less : {{ $json.data }}`  
   - Prompt Type: Define  
   - Enable Output Parser  
   - Connect inputs:  
     - Input data from Code (Python) node  
     - AI Language Model: Groq Chat Model node  
     - AI Memory: Simple Memory node  

7. **Add Set Node (summary)**  
   - Type: Set  
   - Mode: Raw  
   - JSON Output: `={{ '{ "summary": "' + $json.output + '" }' }}`  
   - Connect AI Agent output to this node  

8. **Add TTS Request Node**  
   - Type: HTTP Request  
   - URL: `https://qwen-qwen-tts-demo.hf.space/gradio_api/call/predict`  
   - Method: POST  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "data": [
         "{{ $json.summary }}",
         "Dylan"
       ]
     }
     ```  
   - Connect summary node output to this node  

9. **Add TTS Poll Node**  
   - Type: HTTP Request  
   - URL: `=https://qwen-qwen-tts-demo.hf.space/gradio_api/call/predict/{{ $json.event_id }}` (dynamic URL)  
   - Method: GET (default)  
   - Connect TTS Request output to this node  

10. **Add Extract Audio URL Node (Code)**  
    - Type: Code (JavaScript)  
    - Code:  
      ```js
      const raw = $json.data || $json || "";

      if (Array.isArray($json.data)) {
        return {
          json: {
            audioUrl: $json.data[0]?.url || null,
            rawJson: $json.data
          }
        };
      }

      const match = typeof raw === "string" ? raw.match(/data:\\s*(\\[[^\\]]+\\])/s) : null;

      if (match) {
        try {
          const parsed = JSON.parse(match[1]);
          return {
            json: {
              audioUrl: parsed?.[0]?.url || null,
              rawJson: parsed
            }
          };
        } catch (e) {
          return {
            json: {
              error: "Failed to parse SSE JSON",
              raw
            }
          };
        }
      }

      return {
        json: {
          error: "No audio URL found",
          raw
        }
      };
      ```  
    - Connect TTS Poll output to this node  

11. **Add Edit Fields Node (Set)**  
    - Type: Set  
    - Assignments:  
      - `audioUrl` = `={{ $json.audioUrl }}` (from Extract Audio URL)  
      - `summary` = `={{ $('summary').item.json.summary }}` (from summary node)  
    - Connect Extract Audio URL output to this node  

12. **Add Respond with Both Node**  
    - Type: Respond to Webhook  
    - Respond With: JSON  
    - Response Body:  
      ```json
      {
        "summary": "{{ $json.summary }}",
        "audioUrl": "{{ $json.audioUrl }}"
      }
      ```  
    - Connect Edit Fields output to this node  

13. **Connect Workflow**  
    - Webhook → Extract from File → Code2 (Python) → AI Agent1  
    - AI Agent1 → summary (Set) → TTS Request → TTS Poll → Extract Audio URL → Edit Fields → Respond with Both  
    - Groq Chat Model1 and Simple Memory connect as AI language model and memory input to AI Agent1 respectively

14. **Credentials Setup**  
    - Groq API credentials must be configured and assigned to Groq Chat Model1 node  
    - No authentication needed for Qwen TTS demo (public demo), but monitor usage limits  

15. **Testing**  
    - Test the webhook by POSTing a PDF file to `/summarize-pdf`  
    - Expect JSON response with `summary` and `audioUrl` fields  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow title: **SMART PDF SUMMARIZER WITH AUDIO PLAYBACK**                                        | Displayed in Sticky Note for project branding                                                           |
| Uses Groq AI OpenAI-compatible model for summarization with free tokens and memory buffering        | Node: Groq Chat Model1, Simple Memory                                                                   |
| Uses Hugging Face demo API for Qwen TTS audio generation                                            | API URL: `https://qwen-qwen-tts-demo.hf.space/gradio_api/call/predict`                                  |
| The workflow uses code nodes to handle JSON parsing and data extraction for flexible response parsing | Code2 (Python) and Extract Audio URL (JavaScript)                                                      |
| The poll mechanism is required due to asynchronous audio generation by Qwen TTS                     | TTS Poll node repeatedly requests audio status using event_id                                          |
| Ensure the Groq API credentials are valid and have sufficient quota for production usage            | Credential setup required in n8n credentials manager                                                     |
| This workflow can be integrated with any frontend that supports file uploads and webhook interaction | Webhook path `/summarize-pdf`                                                                           |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with prevailing content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.