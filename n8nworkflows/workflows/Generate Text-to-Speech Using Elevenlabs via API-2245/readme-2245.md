Generate Text-to-Speech Using Elevenlabs via API

https://n8nworkflows.xyz/workflows/generate-text-to-speech-using-elevenlabs-via-api-2245


# Generate Text-to-Speech Using Elevenlabs via API

### 1. Workflow Overview

This workflow provides a simple API endpoint to convert text into speech audio using the Elevenlabs text-to-speech service via its HTTP API. It is designed primarily for video producers or content creators who want to automate voice generation from text inputs.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Validation:**  
  Accepts POST requests via a webhook with parameters `voice_id` and `text`. Validates the presence of these parameters before proceeding.

- **1.2 Voice Generation via Elevenlabs API:**  
  Calls the Elevenlabs text-to-speech API with validated inputs to generate speech audio.

- **1.3 Response Handling:**  
  Returns the generated speech audio as a binary response if successful, or a JSON error message if inputs are invalid.


---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block listens for incoming POST requests and validates that the required parameters `voice_id` and `text` are present before allowing the workflow to continue.

**Nodes Involved:**  
- Webhook  
- If params correct  

**Node Details:**  

- **Webhook**  
  - Type: Webhook (HTTP API entry point)  
  - Configuration:  
    - Method: POST  
    - Path: `/generate-voice`  
    - Response Mode: Uses a separate response node (delayed response)  
  - Inputs: External HTTP POST request  
  - Outputs: Passes request data to "If params correct" node  
  - Version: 2  
  - Edge Cases:  
    - Malformed HTTP requests  
    - Missing or incorrect HTTP method  
    - Timeout if client disconnects before response  

- **If params correct**  
  - Type: If (conditional branching)  
  - Configuration: Checks existence of both `voice_id` and `text` in request body  
    - Conditions combined with AND:  
      - `voice_id` exists and is a string  
      - `text` exists and is a string  
  - Inputs: Output from Webhook  
  - Outputs:  
    - True branch: valid inputs —> "Generate voice" node  
    - False branch: invalid inputs —> "Error" node  
  - Version: 2  
  - Edge Cases:  
    - Parameters present but empty strings (still passes existence check)  
    - Unexpected data types could cause expression errors  
    - Case sensitivity enforced (strict)  
  - Notes: Ensures workflow only proceeds with valid API parameters  


#### 2.2 Voice Generation via Elevenlabs API

**Overview:**  
Sends a POST request to Elevenlabs API to generate speech audio from provided text and voice ID, using a custom HTTP credential containing the API key.

**Nodes Involved:**  
- Generate voice  

**Node Details:**  

- **Generate voice**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.body.voice_id }}` (dynamic voice ID from request)  
    - Method: POST  
    - Headers:  
      - `Content-Type: application/json`  
      - Authentication header `xi-api-key` injected via custom HTTP credentials  
    - Body: JSON containing the text to synthesize: `{ "text": "{{ $json.body.text }}" }`  
    - Authentication: Custom HTTP header authentication using user-provided Elevenlabs API key  
  - Inputs: Validated parameters from "If params correct"  
  - Outputs: Binary audio data to "Respond to Webhook"  
  - Version: 4.2  
  - Edge Cases:  
    - Invalid or expired API key causing 401 Unauthorized  
    - Invalid voice ID causing 404 or 400 errors  
    - Text exceeding API limits causing request rejection  
    - Network timeouts or API downtime  
    - Malformed JSON body or header errors  
  - Credentials: Custom HTTP Authentication with header `"xi-api-key": "<api-key>"`  
  - Notes: Central API integration node enabling text-to-speech functionality  


#### 2.3 Response Handling

**Overview:**  
Returns the resultant speech audio back to the API caller as a binary response or returns an error JSON when inputs are invalid.

**Nodes Involved:**  
- Respond to Webhook  
- Error  

**Node Details:**  

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Configuration:  
    - Responds with binary data (audio)  
  - Inputs: Output from "Generate voice"  
  - Outputs: HTTP response to client with audio file  
  - Version: 1.1  
  - Edge Cases:  
    - Large audio payloads causing delays or timeouts  
    - Client disconnects before response completes  

- **Error**  
  - Type: Respond to Webhook  
  - Configuration:  
    - Responds with JSON error message: `{ "error": "Invalid inputs." }`  
  - Inputs: False branch from "If params correct"  
  - Outputs: HTTP response to client with error message  
  - Version: 1.1  
  - Edge Cases:  
    - Malformed JSON responses (unlikely)  


---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                         | Input Node(s)       | Output Node(s)          | Sticky Note                                                                                                                               |
|---------------------|-----------------------|---------------------------------------|---------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook             | Webhook               | API entry point, accepts POST requests| (external)          | If params correct       |                                                                                                                                           |
| If params correct    | If                    | Validates presence of `voice_id` & `text` | Webhook             | Generate voice (true), Error (false) |                                                                                                                                           |
| Generate voice       | HTTP Request          | Calls Elevenlabs API to generate speech| If params correct (true) | Respond to Webhook       |                                                                                                                                           |
| Respond to Webhook   | Respond to Webhook    | Returns binary audio response          | Generate voice       | (external HTTP response) |                                                                                                                                           |
| Error               | Respond to Webhook    | Returns JSON error for invalid inputs  | If params correct (false) | (external HTTP response) |                                                                                                                                           |
| Sticky Note         | Sticky Note           | Documentation and instructions         | -                   | -                       | ## Generate Text-to-Speech Using Elevenlabs via API<br>This workflow provides an API endpoint to generate speech from text using Elevenlabs.io.<br>Step 1: Configure Custom Credentials in n8n with JSON header containing your API key.<br>Step 2: Send POST to webhook with `voice_id` and `text`.<br>This workflow has been a significant time-saver in my video production tasks. Happy automating! The n8Ninja |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `generate-voice`  
   - Response Mode: `Response Node` (delayed response)  
   - Position: Anywhere convenient (e.g., [300,200])  

2. **Create the If node to validate inputs:**  
   - Type: If  
   - Conditions:  
     - Check that `voice_id` exists (string) in incoming request body  
     - Check that `text` exists (string) in incoming request body  
   - Combine conditions with AND  
   - Position: Connected from Webhook output (e.g., [500,200])  

3. **Create the HTTP Request node to call Elevenlabs API:**  
   - Type: HTTP Request  
   - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.body.voice_id }}` (use expression to pull voice_id)  
   - Method: POST  
   - Headers: Add `Content-Type: application/json`  
   - Body: JSON mode, content: `{ "text": "{{ $json.body.text }}" }` (expression to use input text)  
   - Authentication: Custom HTTP Authentication configured with header: `"xi-api-key": "<your-elevenlabs-api-key>"`  
   - Position: Connected from If node's true output (e.g., [740,180])  

4. **Create Respond to Webhook node for success:**  
   - Type: Respond to Webhook  
   - Response Mode: Binary  
   - Connect from HTTP Request node output  
   - Position: (e.g., [960,180])  

5. **Create Respond to Webhook node for errors:**  
   - Type: Respond to Webhook  
   - Response Mode: JSON  
   - Response Body: `{ "error": "Invalid inputs." }`  
   - Connect from If node's false output  
   - Position: (e.g., [740,360])  

6. **Connect nodes accordingly:**  
   - Webhook → If params correct  
   - If params correct (true) → Generate voice  
   - Generate voice → Respond to Webhook (success)  
   - If params correct (false) → Respond to Webhook (error)  

7. **Configure Custom HTTP Credentials:**  
   - In n8n Credentials section, create new Custom HTTP Auth credentials  
   - Add header `"xi-api-key"` with your Elevenlabs API key as value  

8. **Optional: Add a Sticky Note with instructions:**  
   - Content: Workflow purpose, credential setup JSON snippet, input parameters explanation, and usage tips  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                           | Context or Link                                |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Workflow uses a custom HTTP credential to handle Elevenlabs API key securely via header injection.                                                                                                                                                                                                                     | Credential configuration in n8n               |
| Elevenlabs API docs: https://elevenlabs.io/docs/ (for advanced customization of voice, more parameters, error handling)                                                                                                                                                                                                | Elevenlabs API documentation                   |
| The workflow is designed for POST requests with JSON body containing `voice_id` and `text` keys; no other parameters are currently supported.                                                                                                                                                                         | Input format specification                      |
| The creator offers a custom node implementing Elevenlabs features in beta: [Try it here](http://go.n8n.ninja/811)                                                                                                                                                                                                      | Custom Elevenlabs node link                     |
| This workflow is intended as a time-saving tool primarily for video production voiceovers and similar content creation tasks.                                                                                                                                                                                           | Use case description                            |
| Sticky note inside the workflow contains detailed setup instructions and usage notes.                                                                                                                                                                                                                                   | Internal documentation within workflow UI      |

---

This completes the detailed analysis and documentation of the "Generate Text-to-Speech Using Elevenlabs via API" n8n workflow.