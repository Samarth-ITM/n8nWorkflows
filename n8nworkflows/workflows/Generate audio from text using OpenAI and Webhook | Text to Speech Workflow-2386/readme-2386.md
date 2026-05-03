Generate audio from text using OpenAI and Webhook | Text to Speech Workflow

https://n8nworkflows.xyz/workflows/generate-audio-from-text-using-openai-and-webhook---text-to-speech-workflow-2386


# Generate audio from text using OpenAI and Webhook | Text to Speech Workflow

### 1. Workflow Overview

This workflow automates the conversion of text input into speech audio using OpenAI's text-to-speech capabilities. It is designed for developers, content creators, or customer support teams who want to streamline the process of generating audio from text. The workflow is triggered by an external POST request to a webhook endpoint, processes the input text to generate speech audio via OpenAI, and then responds with the audio data.

Logical blocks:

- **1.1 Input Reception:** Receives incoming text data via a webhook POST request.
- **1.2 Audio Generation:** Uses the OpenAI API to convert the input text into speech audio.
- **1.3 Output Delivery:** Sends the generated audio back to the requester through a webhook response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP POST requests containing text to be converted. It acts as the entry point for the workflow.

- **Nodes Involved:**  
  - Webhook  
  - Sticky Note (documentation)

- **Node Details:**

  - **Webhook**  
    - *Type & Role:* HTTP webhook node; triggers workflow upon receiving POST requests.  
    - *Configuration:*  
      - Path: `/generate_audio`  
      - HTTP Method: POST  
      - Response Mode: `responseNode` (defers response to a later node)  
    - *Key Expressions/Variables:* Accesses incoming JSON body via `$json.body`  
    - *Input/Output Connections:* No input; output connects to OpenAI node.  
    - *Version Requirements:* Uses version 2 of the webhook node for enhanced features.  
    - *Edge Cases / Failures:*  
      - Invalid or missing POST data.  
      - Unauthorized or malformed requests.  
      - Webhook URL not activated in production mode.  
    - *Sticky Note Content:*  
      - Describes testing modes (test webhook URL vs production URL).  
      - Explains the need to activate workflow for production use.

  - **Sticky Note**  
    - *Purpose:* Provides instructions and reminders about webhook testing and activation modes.  
    - *Content Highlights:*  
      - Test mode is single-use per activation.  
      - Production mode requires workflow activation.  
      - Production calls do not show on the canvas but appear in executions list.

---

#### 1.2 Audio Generation

- **Overview:**  
  Converts the received text input into speech audio using OpenAI's audio resource feature.

- **Nodes Involved:**  
  - OpenAI  
  - Sticky Note (configuration instructions)

- **Node Details:**

  - **OpenAI**  
    - *Type & Role:* OpenAI API node specialized for language and audio generation.  
    - *Configuration:*  
      - Input: Expression `={{ $json.body.text_to_convert }}` extracts the text field from webhook payload.  
      - Voice: Set to `"fable"` (indicates the voice model for speech synthesis).  
      - Resource: Set to `"audio"` (enables text-to-speech functionality).  
      - Options: Default/empty.  
      - Credentials: Uses stored OpenAI API credentials (`n8n OpenAI`).  
    - *Input/Output Connections:*  
      - Input from Webhook node.  
      - Output connects to Respond to Webhook node.  
    - *Version Requirements:* Version 1.3 of OpenAI node, supporting audio resource.  
    - *Edge Cases / Failures:*  
      - Invalid or expired API key (authentication failure).  
      - Network timeout or API limits reached.  
      - Text missing or empty in input JSON, causing errors or empty audio.  
      - Unsupported voice or resource parameters.  

  - **Sticky Note1**  
    - *Purpose:* Guides the user to create and configure OpenAI credentials.  
    - *Content Highlights:*  
      - Instructions for obtaining API key from OpenAI.  
      - Steps to create new credentials in n8n OpenAI node.

---

#### 1.3 Output Delivery

- **Overview:**  
  Sends the generated audio binary data back as the response to the initial webhook request.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - *Type & Role:* Responds to the webhook call with data.  
    - *Configuration:*  
      - Respond with: `binary` (returns binary audio data to the client).  
      - Options: default/empty.  
    - *Input/Output Connections:*  
      - Input from OpenAI node.  
      - No outputs (terminates workflow).  
    - *Version Requirements:* Version 1.1.  
    - *Edge Cases / Failures:*  
      - No or invalid binary data to respond with.  
      - Client disconnects before response.  
      - Response formatting errors may cause client failures.

---

### 3. Summary Table

| Node Name         | Node Type                      | Functional Role              | Input Node(s) | Output Node(s)    | Sticky Note                                                                                                          |
|-------------------|--------------------------------|-----------------------------|---------------|-------------------|----------------------------------------------------------------------------------------------------------------------|
| Webhook           | n8n-nodes-base.webhook          | Receives POST text inputs   | None          | OpenAI            | This `Webhook` node triggers the workflow when it receives a POST request. Test and production mode instructions given. |
| OpenAI            | @n8n/n8n-nodes-langchain.openAi| Converts text to speech     | Webhook       | Respond to Webhook| Configure OpenAI API key as credentials; instructions provided for setup.                                           |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Sends audio response        | OpenAI        | None              |                                                                                                                      |
| Sticky Note       | n8n-nodes-base.stickyNote       | Documentation               | None          | None              | Explains webhook testing and production modes.                                                                      |
| Sticky Note1      | n8n-nodes-base.stickyNote       | Documentation               | None          | None              | Guides OpenAI API key credential setup.                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Webhook`  
   - Type: `Webhook` (version 2)  
   - HTTP Method: `POST`  
   - Path: `generate_audio`  
   - Response Mode: `responseNode` (to defer response)  
   - No authentication or special options unless required.

2. **Create OpenAI Node:**  
   - Name: `OpenAI`  
   - Type: `@n8n/n8n-nodes-langchain.openAi` (version 1.3)  
   - Resource: `audio` (enables text-to-speech)  
   - Input: Set expression to `={{ $json.body.text_to_convert }}` to extract text from webhook payload  
   - Voice: Select `"fable"` voice preset  
   - Credentials: Create new OpenAI API credentials with your API key from OpenAI website and assign here.

3. **Connect Webhook to OpenAI:**  
   - Connect `Webhook` node output to `OpenAI` node input.

4. **Create Respond to Webhook Node:**  
   - Name: `Respond to Webhook`  
   - Type: `Respond to Webhook` (version 1.1)  
   - Respond With: Set to `binary` (to send back audio binary data)  
   - No extra options needed.

5. **Connect OpenAI to Respond to Webhook:**  
   - Connect `OpenAI` node output to `Respond to Webhook` node input.

6. **Activate Workflow:**  
   - Save and activate the workflow to enable production webhook URL.  
   - Test workflow via test webhook URL or by sending a POST request with JSON body containing the key `text_to_convert` and the text string value.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Use the test webhook URL for single-call testing; the workflow must be active for production.                                        | Sticky Note on Webhook node                                                                        |
| Obtain OpenAI API key at https://platform.openai.com/account/api-keys                                                                | Sticky Note1 on OpenAI node                                                                        |
| The voice parameter `"fable"` specifies a particular voice model for audio generation; ensure it is supported by your OpenAI plan. | OpenAI node configuration detail                                                                  |
| Webhook response mode set to `responseNode` enables asynchronous response handling, required to send binary audio back.              | Webhook node parameter explanation                                                                |