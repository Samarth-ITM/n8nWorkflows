Process Voice Notes to AI Responses with Claude Sonnet, Nuclino & Slack

https://n8nworkflows.xyz/workflows/process-voice-notes-to-ai-responses-with-claude-sonnet--nuclino---slack-6657


# Process Voice Notes to AI Responses with Claude Sonnet, Nuclino & Slack

### 1. Workflow Overview

This workflow automates the processing of voice notes by converting speech transcripts into AI-generated responses using the Claude Sonnet AI model. It then archives the original prompts and AI outputs in NocoDB, creates a structured document in Nuclino for collaborative knowledge management, and notifies a Slack channel with the prompt, response, and a link to the Nuclino note. The workflow is designed for teams or individuals who want to streamline the handling of voice note queries and integrate AI insights into their documentation and communication channels.

Logical blocks:

- **1.1 Input Reception:** Ingests voice note data via a webhook.
- **1.2 Data Preparation:** Extracts and formats key information from the webhook payload.
- **1.3 AI Processing:** Sends the transcript as a prompt to the Claude Sonnet AI model for a detailed, structured response.
- **1.4 Data Aggregation & Formatting:** Combines prompt and AI response, prepares metadata.
- **1.5 Archival:** Saves both the prompt and AI output to NocoDB databases.
- **1.6 Documentation Creation:** Posts a new item in Nuclino with the prompt and AI response.
- **1.7 Notification:** Sends a formatted message with AI results and Nuclino link to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives incoming voice note data via HTTP POST, acting as the workflow trigger.
- **Nodes Involved:**  
  - Webhook  
  - Sticky Note (Voicenotes voice note tagged as prompt)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook Trigger  
    - Role: Entry point, listens for POST requests at a specific path.  
    - Configuration: HTTP method POST, path set to unique ID.  
    - Key Variables: Incoming JSON body with voice note data including `title` and `transcript`.  
    - Inputs: External HTTP POST request.  
    - Outputs: JSON payload to next node.  
    - Failure Modes: Invalid/malformed JSON, unauthorized requests if security not enforced.  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Documentation/comment for workflow users — marks voice note reception.  
    - No inputs or outputs.

#### 1.2 Data Preparation

- **Overview:** Extracts the voice note title and transcript from the webhook payload and maps them into new fields for further processing.
- **Nodes Involved:**  
  - Edit Fields  
  - Sticky Note (Gets sent to an AI agent)

- **Node Details:**

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Creates two new string fields: `note_title` (voice note title) and `prompt` (the transcript text).  
    - Configuration: Uses expressions to map:  
      - `note_title` = `{{$json.body.data.title}}`  
      - `prompt` = `{{$json.body.data.transcript}}`  
    - Inputs: JSON from Webhook node.  
    - Outputs: JSON with added `note_title` and `prompt`.  
    - Failure Modes: Missing expected data fields in webhook JSON, expression errors.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Indicates data is ready to be sent to AI agent.

#### 1.3 AI Processing

- **Overview:** Sends the extracted prompt to a configured AI agent using the Claude Sonnet model for a comprehensive AI-generated response.
- **Nodes Involved:**  
  - AI Agent (LangChain Agent node)  
  - OpenRouter Chat Model  
  - Sticky Note2 (Concatenating prompt + response)  
  - Merge

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: AI Language Model Node (Anthropic Claude Sonnet via OpenRouter)  
    - Role: Hosts the language model used by the AI Agent.  
    - Configuration: Model set to `anthropic/claude-sonnet-4`. Authenticated with OpenRouter API credentials.  
    - Inputs: Receives prompt text from AI Agent node internally.  
    - Outputs: AI-generated text response to AI Agent.  
    - Failure Modes: API auth errors, rate limiting, network timeouts, malformed prompts.

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Orchestrates prompt sending and response retrieval, applying system message instructions to format AI output strictly in Markdown with SUMMARY, PROMPT, RESPONSE sections.  
    - Configuration:  
      - Takes `text` input from `prompt` field.  
      - System message enforces detailed formatting and style guidelines.  
    - Inputs: JSON with `prompt` string from previous node.  
    - Outputs: JSON with `output` field containing AI response.  
    - Failure Modes: Expression errors in prompt extraction, AI model failures, response parsing issues.

  - **Merge**  
    - Type: Merge Node  
    - Role: Combines outputs from AI Agent and Edit Fields nodes by position to produce a single enriched JSON object containing prompt and response.  
    - Configuration: Mode set to 'combine' by position.  
    - Inputs: Two inputs: AI Agent output and Edit Fields output.  
    - Outputs: Combined JSON object with `prompt` and `output`.  
    - Failure Modes: Mismatched input array lengths, missing inputs.

#### 1.4 Data Aggregation & Formatting

- **Overview:** Prepares a final data object with all relevant fields for archiving, documentation, and notification.
- **Nodes Involved:**  
  - Edit Fields1  
  - Sticky Note3 (Saving to NocoDB for archive)

- **Node Details:**

  - **Edit Fields1**  
    - Type: Set Node  
    - Role: Assigns and consolidates fields for database storage and downstream use. Fields include:  
      - `note_title` (copied from Edit Fields)  
      - `prompt` (copied from Edit Fields)  
      - `output` (from AI Agent output)  
      - `created` timestamp (from Webhook payload)  
    - Inputs: From Merge node and Webhook node (timestamp)  
    - Outputs: JSON prepared for database insertion and documentation.  
    - Failure Modes: Missing data fields, expression syntax errors.

#### 1.5 Archival

- **Overview:** Saves the original prompt and the AI-generated response into two separate tables in NocoDB for record-keeping and future reference.
- **Nodes Involved:**  
  - Save Prompt (NocoDB node)  
  - Save Output (NocoDB node)  
  - Sticky Note4 (Create a doc in Nuclino) [contextually near archival but functionally for documentation]

- **Node Details:**

  - **Save Prompt**  
    - Type: NocoDB Create Node  
    - Role: Inserts the prompt text into a NocoDB table `mg36i2thdctcg5b`.  
    - Configuration:  
      - Field `c76za9u4d0lhiyx` set to `prompt`.  
      - Uses NocoDB API token credentials.  
    - Inputs: Edit Fields1 output.  
    - Outputs: Confirmation response, triggers Save Output.  
    - Failure Modes: API auth failure, network errors, table schema mismatch.

  - **Save Output**  
    - Type: NocoDB Create Node  
    - Role: Inserts the AI response into another NocoDB table `mn2ttgi7atggy1t`.  
    - Configuration:  
      - Fields: AI output, prompt, and a fixed string identifying the automation source.  
      - Uses same NocoDB credentials.  
    - Inputs: Output of Save Prompt node.  
    - Outputs: Confirmation response.  
    - Failure Modes: Same as above.

#### 1.6 Documentation Creation

- **Overview:** Creates a new Nuclino document (item) containing the prompt, AI response, and metadata for collaborative knowledge management.
- **Nodes Involved:**  
  - HTTP Request (Nuclino API POST)  
  - Sticky Note5 (Notify Via Slack)

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request Node  
    - Role: Sends POST request to Nuclino API to create a new item with structured content.  
    - Configuration:  
      - URL: `https://api.nuclino.com/v0/items`  
      - Method: POST  
      - Authentication: HTTP Header Auth with Nuclino API token.  
      - Body Parameters include:  
        - `workspaceId` (string, placeholder `your-workspace-id`)  
        - `title` (from `note_title`)  
        - `collectionID` (string, placeholder `your-collection-id`)  
        - `content` formatted in Markdown including title, creation date, prompt, and AI response.  
    - Inputs: From Edit Fields1 node.  
    - Outputs: JSON response with new Nuclino item data, including item URL.  
    - Failure Modes: API auth failure, invalid workspace or collection IDs, network issues.

#### 1.7 Notification

- **Overview:** Posts a detailed Slack message to a specified channel with the prompt, AI response, and Nuclino note URL.
- **Nodes Involved:**  
  - Send a message (Slack node)

- **Node Details:**

  - **Send a message**  
    - Type: Slack Node  
    - Role: Sends a formatted message to Slack channel `C097VFQFB1U`.  
    - Configuration:  
      - Message text uses Markdown formatting including:  
        - Bold title with note title  
        - Prompt text  
        - AI response text  
        - Nuclino note URL  
      - Slack channel selected from list, using channel ID.  
      - Options: Markdown enabled, unfurl links and media enabled.  
      - Authentication: OAuth2 credential for Slack.  
    - Inputs: Nuclino HTTP Request node output (for URL), and merged prompt/output data.  
    - Outputs: Confirmation of message sent.  
    - Failure Modes: Slack API token invalid, channel not found, rate limits.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                          | Input Node(s)              | Output Node(s)          | Sticky Note                                      |
|---------------------|--------------------------------|----------------------------------------|----------------------------|-------------------------|--------------------------------------------------|
| Webhook             | Webhook Trigger                | Receives voice note data via HTTP POST | External HTTP request       | Edit Fields              | ## Voicenotes voice note tagged as prompt        |
| Edit Fields         | Set Node                      | Extracts title and transcript fields   | Webhook                    | AI Agent, Merge          |                                                  |
| AI Agent            | LangChain Agent (AI)          | Sends prompt to Claude Sonnet AI model | Edit Fields                | Merge                    | ## Gets sent to an AI agent                       |
| OpenRouter Chat Model| AI Language Model             | Provides Claude Sonnet model interface  | AI Agent (internal)         | AI Agent                 |                                                  |
| Merge               | Merge Node                   | Combines prompt and AI response data   | AI Agent, Edit Fields       | Edit Fields1, HTTP Request| ## Concatenating prompt + response                |
| Edit Fields1        | Set Node                      | Consolidates fields for storage & docs | Merge                      | Save Prompt              | ## Saving to NocoDB for archive                    |
| Save Prompt          | NocoDB Create Node            | Saves prompt text to NocoDB             | Edit Fields1               | Save Output              |                                                  |
| Save Output          | NocoDB Create Node            | Saves AI output text to NocoDB          | Save Prompt                |                         |                                                  |
| HTTP Request         | HTTP Request (Nuclino API)    | Creates new document in Nuclino         | Edit Fields1, Merge        | Send a message           | ## Create a doc in Nuclino                         |
| Send a message       | Slack Node                    | Sends message with prompt, response & URL | HTTP Request               |                         | ## Notify Via Slack                                |
| Sticky Note          | Sticky Note                   | Comments/Documentation                   | None                      | None                    | See respective notes per node                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique webhook ID (e.g., `23eefb69-ae58-46e6-9fcc-a53f6175845e`)  
   - No authentication configured (or configure if desired for security).

2. **Create Set Node ("Edit Fields")**  
   - Create two new fields:  
     - `note_title` = Expression: `{{$json.body.data.title}}`  
     - `prompt` = Expression: `{{$json.body.data.transcript}}`  
   - Connect Webhook output to this node.

3. **Create OpenRouter Chat Model Node**  
   - Type: AI Language Model  
   - Select model: `anthropic/claude-sonnet-4`  
   - Set credentials: OpenRouter API token

4. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Input field `text`: Expression: `={{ $json.prompt }}`  
   - System message: (Copy provided detailed instructions that enforce the Markdown format and style)  
   - Connect OpenRouter Chat Model node as AI language model for this agent  
   - Connect "Edit Fields" node output to AI Agent input.

5. **Create Merge Node**  
   - Mode: Combine by position  
   - Connect AI Agent output to input 1  
   - Connect Edit Fields output to input 2

6. **Create Set Node ("Edit Fields1")**  
   - Assign fields:  
     - `note_title` = `={{$('Edit Fields').item.json.note_title}}`  
     - `prompt` = `={{$('Edit Fields').item.json.prompt}}`  
     - `output` = `={{$json.output}}` (from AI Agent output merged)  
     - `created` = `={{$('Webhook').item.json.body.timestamp}}`  
   - Connect Merge node output to this node.

7. **Create NocoDB Node ("Save Prompt")**  
   - Operation: Create  
   - Table: `mg36i2thdctcg5b` (your prompt table)  
   - Field mapping: Map your prompt field to `c76za9u4d0lhiyx`  
   - Credentials: Set NocoDB API token  
   - Connect Edit Fields1 output to Save Prompt input.

8. **Create NocoDB Node ("Save Output")**  
   - Operation: Create  
   - Table: `mn2ttgi7atggy1t` (your output table)  
   - Map fields:  
     - AI output → `cafb7x2p0a3cstt`  
     - Prompt → `cu62cluoip9v7v5`  
     - Fixed string "Voicenotes.com Automation (N8N)" → `c4megtoozksi8kk`  
   - Credentials: Same NocoDB API token  
   - Connect Save Prompt output to Save Output input.

9. **Create HTTP Request Node (Nuclino Document Creation)**  
   - Method: POST  
   - URL: `https://api.nuclino.com/v0/items`  
   - Authentication: HTTP Header Auth with Nuclino API token  
   - Body Parameters (JSON or form-data):  
     - `workspaceId`: your workspace ID  
     - `title`: `={{ $json.note_title }}`  
     - `collectionID`: your collection ID  
     - `content`: Markdown text combining title, created date, prompt, and AI response as shown:  
       ```
       ## {{$json.note_title}}

       ### Created

       {{$json.created}}

       ## Prompt

       {{$json.prompt}}

       ## AI Response

       {{$json.output}}
       ```  
   - Connect Edit Fields1 output to HTTP Request input.

10. **Create Slack Node ("Send a message")**  
    - Set channel to your target Slack channel ID (e.g., `C097VFQFB1U`)  
    - Text: Construct a Markdown message including:  
      ```
      =*New AI Response - {{$json.data.title}}*

      *PROMPT*

      {{$('Merge').item.json.prompt}}

      *RESPONSE:*

      {{$('Merge').item.json.output}}

      *NUCLINO NOTE URL:*

      {{$json.data.url}}
      ```  
    - Enable Markdown, unfurl links and media  
    - Authentication: Slack OAuth2 credentials  
    - Connect HTTP Request output to Slack node input.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                           |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The AI Agent node uses a detailed system message to enforce output in Markdown with SUMMARY, PROMPT, and RESPONSE sections. | Important for consistent AI output formatting.            |
| Nuclino API requires valid workspace and collection IDs; placeholders must be replaced.      | https://docs.nuclino.com/api-reference                    |
| Slack OAuth2 credentials must have permission to post messages in the target channel.        | Slack API documentation                                   |
| NocoDB tables use specific field IDs—adjust if using different schema or tables.             | https://www.nocodb.com/docs/api                           |
| Workflow designed for single-turn AI prompts, suitable for processing individual voice notes.| Use caution for multi-turn or conversational inputs.      |

---

**Disclaimer:** The text provided is exclusively generated from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.