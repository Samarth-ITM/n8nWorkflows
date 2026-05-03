Generate Song Lyrics and Music from Text Prompts using OpenAI and Fal.ai Minimax

https://n8nworkflows.xyz/workflows/generate-song-lyrics-and-music-from-text-prompts-using-openai-and-fal-ai-minimax-10005


# Generate Song Lyrics and Music from Text Prompts using OpenAI and Fal.ai Minimax

### 1. Workflow Overview

This workflow enables automated generation of original song lyrics and corresponding music tracks based on user-provided text prompts via a chat interface. It targets content creators, educators, artists, and anyone needing instant, AI-generated songs with matching music.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by an incoming chat message containing the user's song prompt.
- **1.2 AI Songwriting:** Uses OpenAI models to generate detailed song lyrics in JSON format with style and lyrics fields.
- **1.3 Variable Extraction:** Extracts and sets variables for lyrical style and lyrics to feed into the music generation step.
- **1.4 Music Generation Request:** Sends the lyrics and style to Fal.ai’s Minimax music generation API to queue music creation.
- **1.5 Polling Loop for Music Generation:** Waits and repeatedly checks the status of the music generation until completion.
- **1.6 Final Result Fetching:** On completion, retrieves the generated music track and prepares it for output.

This modular design ensures clear separation of input handling, AI processing, music generation, and asynchronous polling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Listens for incoming chat messages to initiate the songwriting process.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point; triggers workflow on chat message reception.  
    - Config: Default options; webhook ID set for chat integration.  
    - Inputs: External chat platform (n8n chat).  
    - Outputs: Passes chat message text to AI Songwriting Agent.  
    - Edge cases: Trigger failures if webhook misconfigured or chat integration offline.

- **Sticky Note:**  
  - Purpose: Triggers on incoming chat prompts for instant song requests.  
  - Note: Integrates with n8n chat; passes message to agent.

#### 2.2 AI Songwriting

- **Overview:**  
Generates original song lyrics and a lyrical style description from the user's input using OpenAI’s GPT-5 chat model, enforcing a strict JSON output schema.

- **Nodes Involved:**  
  - AI Songwriting Agent  
  - OpenAI Chat Model  
  - Parse Output1

- **Node Details:**  
  - **AI Songwriting Agent**  
    - Type: LangChain Agent  
    - Role: Core AI logic to generate lyrics and style as JSON.  
    - Config: Uses a detailed system prompt instructing to produce >600 character lyrics with labeled sections and a lyrical_style field minimum 10 characters; no conversation, immediate JSON output only.  
    - Inputs: Incoming chat message text.  
    - Outputs: Raw AI response JSON string.  
    - Version: 2.2  
    - Edge cases: Invalid JSON output (mitigated by output parser), model downtime, rate limits.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Language model provider for AI Songwriting Agent.  
    - Config: Uses model "gpt-5-chat-latest".  
    - Credentials: Requires valid OpenAI API key.  
    - Edge cases: API errors, quota exceeded, network issues.

  - **Parse Output1**  
    - Type: LangChain Output Parser (Structured)  
    - Role: Parses AI JSON output strictly according to schema (lyrical_style:string, lyrics:string).  
    - Config: JSON schema validation ensures output correctness.  
    - Inputs: AI Songwriting Agent output.  
    - Outputs: Parsed object with extracted fields.  
    - Edge cases: Parsing failures if AI output malformed.

- **Sticky Note:**  
  - Purpose: Generates 600+ char lyrics/genre via OpenAI, parses JSON output.  
  - Note: Uses gpt-5-chat-latest; enforces schema for style & lyrics fields.

#### 2.3 Variable Extraction

- **Overview:**  
Extracts parsed JSON fields into workflow variables to prepare for music generation.

- **Nodes Involved:**  
  - Set Script Variables

- **Node Details:**  
  - **Set Script Variables**  
    - Type: Set Node  
    - Role: Maps parsed lyrical_style and lyrics into variables named `Lyrical_style` and `Lyrics`.  
    - Config: Uses expressions to assign variables from parsed output JSON.  
    - Inputs: Parsed JSON from Parse Output1.  
    - Outputs: Variables used in subsequent HTTP request.  
    - Edge cases: Missing or empty fields cause downstream failures.

- **Sticky Note:**  
  - Purpose: Extracts parsed lyrics/style into variables for music gen.  
  - Note: Maps output.lyrical_style & output.lyrics via expressions.

#### 2.4 Music Generation Request

- **Overview:**  
Submits lyrics and lyrical style to Fal.ai’s Minimax music generation API to queue a music track creation.

- **Nodes Involved:**  
  - Generate Music Track

- **Node Details:**  
  - **Generate Music Track**  
    - Type: HTTP Request  
    - Role: POST request to Fal.ai API endpoint `/fal-ai/minimax-music/v1.5`.  
    - Config:  
      - Body parameters: `prompt` with lyrics text, `lyrics_prompt` with lyrical style.  
      - Headers: `Content-type: application/json`.  
      - Auth: HTTP Header Authentication with Fal.ai API key.  
    - Inputs: Variables from Set Script Variables.  
    - Outputs: Fal.ai returns a `request_id` to track generation.  
    - Credentials: Fal.ai HTTP Header Auth required.  
    - Edge cases: Authentication failure (401), invalid request body, API downtime.

- **Sticky Note:**  
  - Purpose: POSTs lyrics/style to Fal.ai minimax-music queue.

#### 2.5 Polling Loop for Music Generation

- **Overview:**  
Waits a fixed interval, then polls the Fal.ai API for the status of the music generation job, looping until it's complete.

- **Nodes Involved:**  
  - Wait for Generation  
  - Check Generation Status  
  - Route on Status

- **Node Details:**  
  - **Wait for Generation**  
    - Type: Wait Node  
    - Role: Pauses workflow for 30 seconds between status checks.  
    - Config: 30-second delay.  
    - Inputs: After music generation request or polling loop.  
    - Outputs: Triggers status check.  
    - Edge cases: Extended wait times may slow user feedback.

  - **Check Generation Status**  
    - Type: HTTP Request  
    - Role: GET request to `/fal-ai/minimax-music/requests/{request_id}/status` endpoint.  
    - Config: Uses dynamic URL with `request_id` from previous response.  
    - Auth: Fal.ai HTTP Header Auth.  
    - Outputs: Returns current status in JSON (`status` field).  
    - Edge cases: Network errors, invalid request_id, auth errors.

  - **Route on Status**  
    - Type: Switch Node  
    - Role: Routes workflow based on `status` field from status check.  
    - Config:  
      - If status = "COMPLETED": proceed to fetch final result.  
      - Else: loop back to Wait for Generation.  
    - Edge cases: Case sensitivity in status string; unexpected status values.

- **Sticky Note:**  
  - Wait for Generation + Check Generation Status: Waits 30s, polls until 'COMPLETED'.  
  - Route on Status + Fetch Final Result: Routes to fetch on 'COMPLETED', else loops.

#### 2.6 Final Result Fetching

- **Overview:**  
On completion, fetches the final generated music track data including audio URL.

- **Nodes Involved:**  
  - Fetch Final Result

- **Node Details:**  
  - **Fetch Final Result**  
    - Type: HTTP Request  
    - Role: GET request to `/fal-ai/minimax-music/requests/{request_id}` to retrieve final music track data.  
    - Config: Uses dynamic `request_id`; authenticated with Fal.ai API key.  
    - Outputs: Contains audio URL and metadata for final output.  
    - Edge cases: API downtime, invalid request_id, auth failure.

- **Sticky Note:**  
  - Outputs link for chat response after successful generation.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                         | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                 |
|---------------------------|--------------------------------------|---------------------------------------|----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger | Input trigger from chat platform       | -                          | AI Songwriting Agent         | Triggers on incoming chat prompts for instant song requests. Integrates with n8n chat; passes message.      |
| AI Songwriting Agent       | @n8n/n8n-nodes-langchain.agent        | Generates original lyrics & style JSON | When chat message received | Set Script Variables         | Generates 600+ char lyrics/genre via OpenAI, parses JSON output. Uses gpt-5-chat-latest; enforces schema.     |
| OpenAI Chat Model          | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides language model for songwriting| AI Songwriting Agent        | AI Songwriting Agent         |                                                                                                             |
| Parse Output1              | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI JSON output into structured data | AI Songwriting Agent        | Set Script Variables         |                                                                                                             |
| Set Script Variables       | n8n-nodes-base.set                    | Extracts variables for music generation | Parse Output1               | Generate Music Track         | Extracts parsed lyrics/style into variables for music gen. Maps output.lyrical_style & output.lyrics.        |
| Generate Music Track       | n8n-nodes-base.httpRequest            | Sends lyrics/style to Fal.ai music API | Set Script Variables        | Wait for Generation          | POSTs lyrics/style to Fal.ai minimax-music queue.                                                           |
| Wait for Generation        | n8n-nodes-base.wait                   | Waits fixed interval between polls     | Generate Music Track, Route on Status (Progress) | Check Generation Status     | Waits 30s, then polls status endpoint until 'COMPLETED'.                                                    |
| Check Generation Status    | n8n-nodes-base.httpRequest            | Checks music generation status          | Wait for Generation         | Route on Status              |                                                                                                             |
| Route on Status            | n8n-nodes-base.switch                 | Routes based on generation status       | Check Generation Status     | Fetch Final Result (Done), Wait for Generation (Progress) | Routes to fetch on 'COMPLETED', else loops; outputs link for chat response.                                  |
| Fetch Final Result         | n8n-nodes-base.httpRequest            | Retrieves final music track data        | Route on Status (Done)      | -                           |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: `When chat message received`  
   - Configuration: Default options, ensure webhook ID is set for chat integration.

2. **Add OpenAI Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: `OpenAI Chat Model`  
   - Set model to `gpt-5-chat-latest`.  
   - Attach OpenAI API credentials with valid API key.

3. **Add AI Songwriting Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `AI Songwriting Agent`  
   - Configure system prompt as per the detailed instructions:  
     - Generate original song lyrics JSON with `lyrical_style` and `lyrics` fields.  
     - Lyrics minimum 600 characters including section labels.  
     - Style minimum 10 characters.  
     - Enforce immediate JSON output, no conversation.  
   - Assign `OpenAI Chat Model` as language model.  
   - Set node version to 2.2.  
   - Connect input from `When chat message received`.  
   - Enable Output Parser.

4. **Add Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Name: `Parse Output1`  
   - Configure JSON schema with required fields: `lyrical_style` (string), `lyrics` (string).  
   - Connect input from `AI Songwriting Agent` output parser port.

5. **Add Set Node for Variables**  
   - Type: `n8n-nodes-base.set`  
   - Name: `Set Script Variables`  
   - Add two string variables:  
     - `Lyrical_style` = expression: `{{$json.output.lyrical_style}}`  
     - `Lyrics` = expression: `{{$json.output.lyrics}}`  
   - Connect input from `Parse Output1`.

6. **Add HTTP Request Node for Music Generation**  
   - Type: `n8n-nodes-base.httpRequest`  
   - Name: `Generate Music Track`  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/minimax-music/v1.5`  
   - Headers: `Content-type: application/json`  
   - Body (JSON):  
     - `prompt`: `={{$json.Lyrics}}`  
     - `lyrics_prompt`: `={{$json.Lyrical_style}}`  
   - Authentication: HTTP Header Auth with Fal.ai API Key credential.  
   - Connect input from `Set Script Variables`.

7. **Add Wait Node**  
   - Type: `n8n-nodes-base.wait`  
   - Name: `Wait for Generation`  
   - Configure wait time: 30 seconds.  
   - Connect input from `Generate Music Track` and from the polling switch loop.

8. **Add HTTP Request Node for Status Check**  
   - Type: `n8n-nodes-base.httpRequest`  
   - Name: `Check Generation Status`  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/minimax-music/requests/{{$json.request_id}}/status`  
   - Authentication: HTTP Header Auth with Fal.ai API Key credential.  
   - Connect input from `Wait for Generation`.

9. **Add Switch Node for Status Routing**  
   - Type: `n8n-nodes-base.switch`  
   - Name: `Route on Status`  
   - Condition:  
     - If `status` equals `COMPLETED` (case sensitive), output `Done`.  
     - Else output `Progress`.  
   - Connect input from `Check Generation Status`.

10. **Add HTTP Request Node for Final Result**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Name: `Fetch Final Result`  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/minimax-music/requests/{{$json.request_id}}`  
    - Authentication: HTTP Header Auth with Fal.ai API Key credential.  
    - Connect input from `Route on Status` output `Done`.

11. **Connect Polling Loop**  
    - From `Route on Status` output `Progress` connect back to `Wait for Generation` to repeat status checks.

12. **Ensure all credentials set:**  
    - OpenAI API key configured in `OpenAI Chat Model`.  
    - Fal.ai API key configured in all HTTP Request nodes interacting with Fal.ai service.

13. **Activate the workflow and test with a chat prompt**, e.g., "Upbeat pop road trip song".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| [AI Songwriter Chat: Lyrics to Full Music Tracks] - This template uses chat triggers to generate detailed song lyrics and matching music tracks, polling until ready, returning lyrics and audio to chat. Prerequisites include OpenAI API and Fal.ai account. Setup instructions for credentials and API keys included. Use cases: content creation, education, gifting, prototyping. | Overview sticky note in workflow titled "Overview Note6".                                                         |
| Troubleshooting tips: Invalid JSON outputs can be mitigated by stress-testing the AI prompt and validating parser schema; Fal.ai auth or quota errors manifest as HTTP 401; polling loops can be adjusted by increasing wait time; ensure lyrics meet 600+ char minimum to avoid generation errors.                                                                | Included in "Overview Note6" sticky note.                                                                          |
| Fal.ai API documentation and account setup required to obtain API keys and understand rate limits: https://fal.ai/                                                                                                                                                                                                                                                | Referenced credential setup in workflow notes.                                                                     |
| OpenAI API key creation and usage guide: https://platform.openai.com/docs/api-reference                                                                                                                                                                                                                                                                             | Referenced credential setup in workflow notes.                                                                     |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It complies fully with content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.