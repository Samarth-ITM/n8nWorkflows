Connect AI to any chats in Kommo

https://n8nworkflows.xyz/workflows/connect-ai-to-any-chats-in-kommo-2841


# Connect AI to any chats in Kommo

### 1. Workflow Overview

This workflow, titled **"Connect AI to Kommo"**, enables automated AI-driven customer service responses within Kommo chat channels (Telegram, WhatsApp, Facebook, etc.) using n8n and OpenAI. It intercepts incoming messages from Kommo via webhook, processes them (including voice transcription if needed), applies AI language understanding and response generation, and sends replies back to Kommo chats.

**Target Use Cases:**  
- Customer support automation  
- Client qualification  
- Invoice-related communication  

**Logical Blocks:**

- **1.1 Input Reception:** Receives incoming messages from Kommo via webhook and extracts entities.  
- **1.2 Stop Condition Check:** Determines if the conversation should be handled by AI or stopped for manual intervention.  
- **1.3 Voice Message Handling:** Detects voice messages, retrieves and transcribes them to text.  
- **1.4 AI Processing:** Uses LangChain nodes with OpenAI to generate AI responses based on conversation memory.  
- **1.5 Kommo API Interaction:** Obtains authentication tokens and sends AI-generated replies back to Kommo chats.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives incoming messages from Kommo via a webhook and extracts relevant entities for further processing.

- **Nodes Involved:**  
  - `new_message` (Webhook)  
  - `get_entity` (HTTP Request)  

- **Node Details:**  

  - **new_message**  
    - Type: Webhook  
    - Role: Entry point for incoming Kommo messages. Listens for HTTP POST requests from Kommo.  
    - Configuration: Uses a specific webhook ID to receive messages.  
    - Inputs: External HTTP request from Kommo.  
    - Outputs: Passes message data to `get_entity`.  
    - Edge Cases: Missing or malformed webhook payloads; webhook authentication issues.  

  - **get_entity**  
    - Type: HTTP Request  
    - Role: Extracts entities or relevant data from the incoming message payload.  
    - Configuration: Likely calls Kommo or internal API to parse message content.  
    - Inputs: Data from `new_message`.  
    - Outputs: Passes extracted data to `hasStopTag`.  
    - Edge Cases: API failures, unexpected response formats, timeouts.

---

#### 2.2 Stop Condition Check

- **Overview:**  
  This block checks if the conversation or message contains a stop tag, indicating that AI should not respond and a human should intervene.

- **Nodes Involved:**  
  - `hasStopTag` (Switch)  

- **Node Details:**  

  - **hasStopTag**  
    - Type: Switch  
    - Role: Routes workflow based on presence of stop tag in message or entity data.  
    - Configuration: Evaluates conditions to decide if AI processing continues or stops.  
    - Inputs: Data from `get_entity`.  
    - Outputs:  
      - If stop tag present: likely terminates or routes to manual handling (not explicitly shown).  
      - If no stop tag: continues to voice detection (`isVoice`).  
    - Edge Cases: Incorrect tag detection, false positives/negatives.

---

#### 2.3 Voice Message Handling

- **Overview:**  
  Detects if the incoming message is a voice message, retrieves the voice file, and transcribes it to text for AI processing.

- **Nodes Involved:**  
  - `isVoice` (If)  
  - `get voice` (HTTP Request)  
  - `transcribe voice` (OpenAI node)  
  - `setText` (Set)  

- **Node Details:**  

  - **isVoice**  
    - Type: If  
    - Role: Checks if the message type is voice.  
    - Configuration: Condition based on message metadata.  
    - Inputs: From `hasStopTag`.  
    - Outputs:  
      - True: to `get voice`.  
      - False: to `setText` (directly sets text for AI).  
    - Edge Cases: Incorrect message type detection.  

  - **get voice**  
    - Type: HTTP Request  
    - Role: Downloads the voice message file from Kommo or related storage.  
    - Configuration: URL and headers configured to fetch voice data.  
    - Inputs: From `isVoice` (true branch).  
    - Outputs: Passes audio data to `transcribe voice`.  
    - Edge Cases: Download failures, invalid URLs, network errors.  

  - **transcribe voice**  
    - Type: OpenAI (LangChain)  
    - Role: Uses OpenAI's speech-to-text capabilities to transcribe voice audio to text.  
    - Configuration: OpenAI credentials required; configured for transcription.  
    - Inputs: Audio data from `get voice`.  
    - Outputs: Transcribed text to `setText`.  
    - Edge Cases: Transcription errors, audio format issues, API rate limits.  

  - **setText**  
    - Type: Set  
    - Role: Sets the transcribed or original text message into a standardized field for AI processing.  
    - Configuration: Maps transcription or text message to a variable used by AI nodes.  
    - Inputs: From `transcribe voice` or `isVoice` (false branch).  
    - Outputs: Passes text to `ai` node.  
    - Edge Cases: Missing or empty text.

---

#### 2.4 AI Processing

- **Overview:**  
  This block generates AI responses using LangChain integration with OpenAI, maintaining conversation memory and language model configuration.

- **Nodes Involved:**  
  - `ai` (LangChain Agent)  
  - `model` (LangChain OpenAI LM)  
  - `memory` (LangChain Memory Buffer Window)  

- **Node Details:**  

  - **model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Configures the OpenAI language model used for generating responses.  
    - Configuration: OpenAI API key credential; model parameters (temperature, max tokens, etc.) set here.  
    - Inputs: None (used as a resource node).  
    - Outputs: Connected to `ai` as language model.  
    - Edge Cases: API key issues, rate limits, model unavailability.  

  - **memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains a sliding window of conversation history to provide context to AI.  
    - Configuration: Window size and memory parameters configured.  
    - Inputs: None (resource node).  
    - Outputs: Connected to `ai` as memory provider.  
    - Edge Cases: Memory overflow, context loss.  

  - **ai**  
    - Type: LangChain Agent  
    - Role: Core AI processing node that generates responses using the language model and memory.  
    - Configuration: References `model` and `memory` nodes; input text from `setText`.  
    - Inputs: Text message from `setText`.  
    - Outputs: AI-generated reply to `Get token`.  
    - Edge Cases: AI generation errors, timeouts, invalid inputs.

---

#### 2.5 Kommo API Interaction

- **Overview:**  
  This block handles Kommo API authentication and sends the AI-generated reply back to the appropriate Kommo chat.

- **Nodes Involved:**  
  - `Get token` (HTTP Request)  
  - `Recieve message` (HTTP Request)  

- **Node Details:**  

  - **Get token**  
    - Type: HTTP Request  
    - Role: Obtains an authentication token from Kommo API to authorize message sending.  
    - Configuration: Uses Kommo credentials; configured with client ID/secret or API key.  
    - Inputs: AI response from `ai`.  
    - Outputs: Passes token and message data to `Recieve message`.  
    - Edge Cases: Auth failures, expired tokens, API errors.  

  - **Recieve message**  
    - Type: HTTP Request  
    - Role: Sends the AI-generated reply message back to Kommo chat via API.  
    - Configuration: Uses token from `Get token`; posts message to Kommo endpoint.  
    - Inputs: Auth token and message data from `Get token`.  
    - Outputs: Terminal node (no further output).  
    - Edge Cases: Message send failures, network errors, invalid chat IDs.

---

### 3. Summary Table

| Node Name       | Node Type                      | Functional Role                          | Input Node(s)       | Output Node(s)     | Sticky Note                          |
|-----------------|--------------------------------|----------------------------------------|---------------------|--------------------|------------------------------------|
| new_message     | Webhook                       | Receives incoming Kommo messages       | External HTTP       | get_entity         |                                    |
| get_entity      | HTTP Request                  | Extracts entities from message         | new_message         | hasStopTag         |                                    |
| hasStopTag      | Switch                       | Checks for stop tag to halt AI         | get_entity          | isVoice            |                                    |
| isVoice         | If                           | Detects if message is voice             | hasStopTag          | get voice, setText |                                    |
| get voice       | HTTP Request                  | Downloads voice message audio           | isVoice (true)      | transcribe voice   |                                    |
| transcribe voice| OpenAI (LangChain)            | Transcribes voice audio to text         | get voice           | setText            |                                    |
| setText         | Set                          | Sets text message for AI processing     | transcribe voice, isVoice (false) | ai          |                                    |
| model           | LangChain LM Chat OpenAI      | Configures OpenAI language model        | None                | ai                 |                                    |
| memory          | LangChain Memory Buffer Window| Maintains conversation context          | None                | ai                 |                                    |
| ai              | LangChain Agent               | Generates AI response                    | setText, model, memory | Get token        |                                    |
| Get token       | HTTP Request                  | Gets Kommo API auth token                | ai                  | Recieve message    |                                    |
| Recieve message | HTTP Request                  | Sends AI reply to Kommo chat             | Get token           | None               |                                    |
| Sticky Note1    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note2    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note3    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note4    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note5    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note6    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note7    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note8    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note9    | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note10   | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note11   | Sticky Note                  |                                        |                     |                    |                                    |
| Sticky Note12   | Sticky Note                  |                                        |                     |                    |                                    |

*Note: Sticky notes are present but empty in this workflow export.*

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: `new_message`**  
   - Type: Webhook  
   - Configure webhook to receive POST requests from Kommo chat messages.  
   - Save webhook URL for Kommo integration.

2. **Add HTTP Request Node: `get_entity`**  
   - Connect input from `new_message`.  
   - Configure to extract entities or parse incoming message content (e.g., call Kommo API or internal parsing endpoint).  
   - Set method, URL, headers as per Kommo API or your parsing service.

3. **Add Switch Node: `hasStopTag`**  
   - Connect input from `get_entity`.  
   - Configure condition to detect a stop tag in message or entity data (e.g., check if message contains a specific keyword or flag).  
   - Define two outputs: one for stop tag present (manual handling), one for no stop tag (continue AI).

4. **Add If Node: `isVoice`**  
   - Connect input from `hasStopTag` (no stop tag branch).  
   - Configure condition to check if message type is voice (e.g., check message metadata field).  
   - Define two outputs: true (voice message), false (text message).

5. **Add HTTP Request Node: `get voice`**  
   - Connect input from `isVoice` (true branch).  
   - Configure to download voice message audio file from Kommo or storage URL.  
   - Set method to GET, URL dynamically from message data.

6. **Add OpenAI Node: `transcribe voice`**  
   - Connect input from `get voice`.  
   - Configure with OpenAI credentials for speech-to-text transcription.  
   - Set model and parameters for transcription.

7. **Add Set Node: `setText`**  
   - Connect inputs from `transcribe voice` (voice branch) and `isVoice` (false branch).  
   - Configure to set a unified text field with either transcribed text or original text message for AI input.

8. **Add LangChain LM Chat OpenAI Node: `model`**  
   - Configure with OpenAI credentials.  
   - Set model parameters (e.g., GPT-4 or GPT-3.5, temperature, max tokens).

9. **Add LangChain Memory Buffer Window Node: `memory`**  
   - Configure window size and memory parameters to maintain conversation context.

10. **Add LangChain Agent Node: `ai`**  
    - Connect inputs: text from `setText`, language model from `model`, memory from `memory`.  
    - Configure agent parameters as needed.

11. **Add HTTP Request Node: `Get token`**  
    - Connect input from `ai`.  
    - Configure to authenticate with Kommo API and obtain access token.  
    - Set method, URL, headers, and credentials (OAuth2 or API key).

12. **Add HTTP Request Node: `Recieve message`**  
    - Connect input from `Get token`.  
    - Configure to send AI-generated reply back to Kommo chat via API.  
    - Set method POST, URL, headers including auth token, and body with message content.

13. **Activate the workflow** and test by sending messages to Kommo chat.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                  |
|------------------------------------------------------------------------------|-------------------------------------------------|
| Entrust customer service to AI using n8n and Kommo!                          | Workflow description                            |
| Works with any message channel connected to Kommo (Telegram, WhatsApp, Facebook) | Advantages of integration                        |
| Understands voice and text messages                                          | Advantages of integration                        |
| You can stop AI for specific transactions or contacts for manual help       | Advantages of integration                        |
| Possible to supplement AI with additional tools                             | Advantages of integration                        |
| Useful for customer support, client qualification, invoicing                | Use cases                                       |
| Workflow operation: webhook receives message → n8n processes → reply sent   | How it works                                    |
| Installation steps: install workflow, connect Kommo, set OpenAI credentials, activate, test | Installation instructions                        |
| Video demonstration available: https://youtu.be/yFqkp-HrCeY                 | Video link                                      |

---

This document fully describes the "Connect AI to Kommo" workflow, enabling users and AI agents to understand, reproduce, and extend the integration confidently.