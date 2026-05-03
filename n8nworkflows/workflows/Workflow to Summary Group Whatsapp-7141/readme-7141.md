Workflow to Summary Group Whatsapp

https://n8nworkflows.xyz/workflows/workflow-to-summary-group-whatsapp-7141


# Workflow to Summary Group Whatsapp

---

### 1. Workflow Overview

This workflow, titled **"Daily WhatsApp Summary with Group-Level Control"**, automates the process of capturing WhatsApp group messages, storing them in a Google Sheet, and generating daily summaries sent back to each group. It is designed for organizations or communities that want to maintain organized, thematic recaps of WhatsApp group conversations with controlled group-level permissions.

The workflow is logically divided into two main parts:

- **1.1 Message Capture and Storage:** Receives incoming WhatsApp messages via webhook, filters for group messages, extracts and processes message content (text or audio), and saves messages into a Google Sheet log.

- **1.2 Summary Generation and Delivery:** Triggered daily, this part reads the previous day's stored messages, groups messages by WhatsApp group, validates group permissions for receiving summaries, uses AI to generate concise thematic summaries, and sends these summaries as WhatsApp messages back to the respective groups.

---

### 2. Block-by-Block Analysis

#### 2.1 Part 1 – Message Capture and Storage

**Overview:**  
This block listens for incoming WhatsApp webhook events, filters messages to only those from groups, extracts relevant fields, distinguishes message types (text or audio), transcribes audio messages when necessary, and saves all processed messages to a Google Sheet for persistent storage.

**Nodes Involved:**  
- Webhook  
- If  
- Edit Fields  
- Switch  
- Audio - HTTP Request  
- Convert to File  
- Transcrever Áudio (OpenAI audio transcription)  
- Edit Fields 2  
- Merge  
- Salvar mensagens na planilha (Google Sheets)  
- Buscar Grupos (HTTP Request to external API)  
- Verifica Aba Filtro de Grupos (Google Sheets)  
- If1  
- Salva grupo para Envio (Google Sheets)  
- Finaliza Fluxo  
- Finaliza Fluxo2  
- Sticky Notes (contextual comments)

**Node Details:**

- **Webhook**  
  - *Type:* HTTP Webhook (POST)  
  - *Role:* Entry point receiving WhatsApp message events.  
  - *Config:* Path: `/resumir_grupos`  
  - *Input:* External WhatsApp webhook payload.  
  - *Output:* Passes message data downstream.  
  - *Edge Cases:* Malformed payloads, webhook misconfiguration.

- **If**  
  - *Type:* Conditional node  
  - *Role:* Checks if incoming message is from a WhatsApp group by verifying if `remoteJid` contains `@g.us`.  
  - *Config:* Condition: `body.data.key.remoteJid` contains `@g.us`  
  - *Branches:*  
    - True → Continue processing  
    - False → Finaliza Fluxo (NoOp node to terminate flow)  
  - *Edge Cases:* Unexpected message structure, missing fields.

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Extracts and renames multiple relevant fields from the webhook payload such as group ID, sender info, message content, message type, timestamp, message ID, etc.  
  - *Key expressions:* Uses JSON path expressions like `{{$json.body.data.key.remoteJid}}`  
  - *Output:* Structured JSON with normalized fields for downstream processing.  
  - *Edge Cases:* Missing or undefined message properties.

- **Switch**  
  - *Type:* Switch node  
  - *Role:* Routes messages based on their type (`conversation` for text, `audioMessage` for audio).  
  - *Branches:*  
    - `texto` → text messages  
    - `audio` → audio messages  
  - *Edge Cases:* Unsupported message types.

- **Audio - HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Calls external API to convert audio message to base64 format.  
  - *Config:* POST to `https://evolution.modexflow.com/chat/getBase64FromMediaMessage/{{ instance }}` with message ID in body.  
  - *Auth:* Header Authentication with EVOAPI token.  
  - *Output:* Base64 audio data.  
  - *Edge Cases:* API failures, auth errors, network timeouts.

- **Convert to File**  
  - *Type:* Convert to File  
  - *Role:* Converts base64 audio data from HTTP Request into a binary file format suitable for transcription.  
  - *Config:* Uses MIME type from message metadata.  
  - *Output:* Binary data for transcription.  
  - *Edge Cases:* Invalid base64 data or MIME type.

- **Transcrever Áudio**  
  - *Type:* OpenAI Audio Transcription (LangChain node)  
  - *Role:* Transcribes binary audio data into Portuguese text.  
  - *Credentials:* OpenAI API key configured.  
  - *Output:* Transcribed text.  
  - *Edge Cases:* OpenAI API rate limits, transcription errors, unsupported audio formats.

- **Edit Fields 2**  
  - *Type:* Set node  
  - *Role:* Normalizes the transcribed text into the `mensagem` field for consistency with text messages.  
  - *Output:* Unified field for message text.

- **Merge**  
  - *Type:* Merge node (combine mode)  
  - *Role:* Combines outputs of text and audio transcription branches back into a single stream for uniform processing.  
  - *Edge Cases:* Mismatched data keys or empty outputs.

- **Salvar mensagens na planilha**  
  - *Type:* Google Sheets Append or Update  
  - *Role:* Inserts or updates messages into a Google Sheets document logging WhatsApp messages.  
  - *Fields:* Date, message content, message ID, webhook event, group ID, WhatsApp number, sender name.  
  - *Credentials:* Google Sheets OAuth2 configured.  
  - *Edge Cases:* API quota limits, permission errors.

- **Buscar Grupos**  
  - *Type:* HTTP Request  
  - *Role:* Queries external API for detailed group information based on group ID.  
  - *Auth:* EVOAPI header authentication.  
  - *Output:* Group metadata such as group name and ID.  
  - *Edge Cases:* API unavailability, auth failure.

- **Verifica Aba Filtro de Grupos**  
  - *Type:* Google Sheets Lookup  
  - *Role:* Checks if the group exists in a filter sheet and retrieves its details.  
  - *Input:* Group ID from previous node.  
  - *Output:* Group filter record.  
  - *Edge Cases:* Missing group entries.

- **If1**  
  - *Type:* Conditional node  
  - *Role:* Checks existence of the group in the filter sheet.  
  - *Branches:*  
    - If group does not exist → append new group info with "Mandar Resumo" set to FALSE  
    - Else → end flow with Finaliza Fluxo2  
  - *Edge Cases:* False negatives if sheet is outdated.

- **Salva grupo para Envio**  
  - *Type:* Google Sheets Append  
  - *Role:* Adds new groups into the filter sheet with default "Mandar Resumo" = FALSE to control summary sending permissions.  
  - *Edge Cases:* Sheet write failures.

- **Finaliza Fluxo / Finaliza Fluxo2**  
  - *Type:* NoOp (End flow)  
  - *Role:* Terminates processing for non-group messages or groups not enabled for summaries.

- **Sticky Notes:** Provide context for the above nodes, grouped as:  
  - Message reception and group filtering  
  - Message formatting and type distinction  
  - Audio transcription  
  - Message saving  
  - Group information fetching and permission setup

---

#### 2.2 Part 2 – Summary Generation and Delivery

**Overview:**  
This block runs daily at 08:00 AM, retrieves messages from the previous day, groups them by WhatsApp group, verifies group permissions for summary sending, generates summaries using an AI model with strict formatting instructions, and sends the summary messages back to the corresponding WhatsApp groups.

**Nodes Involved:**  
- Schedule Trigger  
- Acessar conversas na Planilha (Google Sheets)  
- Separar mensagens por grupos (Code node)  
- Validar Grupo (Google Sheets)  
- Loop Over Items (SplitInBatches)  
- AI Agent (LangChain agent using OpenAI)  
- OpenAI Chat Model (GPT-4o-mini)  
- Enviar mensagem (HTTP Request)  
- Replace Me (NoOp)  
- Sticky Notes (contextual comments)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Scheduled trigger  
  - *Role:* Triggers workflow daily at 08:00 (America/Sao_Paulo timezone).  
  - *Edge Cases:* Timezone misconfigurations.

- **Acessar conversas na Planilha**  
  - *Type:* Google Sheets Lookup  
  - *Role:* Retrieves messages logged in the Google Sheet from the previous day (filter by date).  
  - *Input:* Date filter set to yesterday's date in `dd/MM/yyyy` format.  
  - *Output:* List of messages for processing.  
  - *Edge Cases:* Missing data, sheet access errors.

- **Separar mensagens por grupos**  
  - *Type:* Code (JavaScript)  
  - *Role:* Groups messages by `RemoteJid Grupo` and concatenates messages text into a single string per group separated by new lines.  
  - *Output:* Array of objects `{ grupo: <groupId>, mensagens: <concatenated messages> }`.  
  - *Edge Cases:* Empty message sets.

- **Validar Grupo**  
  - *Type:* Google Sheets Lookup  
  - *Role:* Filters groups to those where "Mandar Resumo" is set to true in the group filter sheet, allowing only authorized groups to receive summaries.  
  - *Input:* Group ID from previous step.  
  - *Output:* Validated groups for summary sending.  
  - *Edge Cases:* Sheet sync issues.

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates over each validated group to process summaries individually.  
  - *Edge Cases:* Large batch sizes might cause rate limits.

- **AI Agent**  
  - *Type:* LangChain Agent with OpenAI  
  - *Role:* Generates a concise, thematic summary of the previous day's messages per group using GPT-4o-mini model.  
  - *Prompt Highlights:*  
    - Summarize faithfully in Portuguese (Brazil)  
    - Max 1500 characters  
    - Use specific formatting and structure (intro with date, thematic blocks, bullet points)  
    - Preserve real names exactly, no generic placeholders or hallucinations  
    - No emojis or closing remarks  
    - Links enclosed in quotes to suppress thumbnails  
  - *Input:* Concatenated messages per group.  
  - *Credentials:* OpenAI API.  
  - *Edge Cases:* API limits, prompt failures, incomplete summaries.

- **OpenAI Chat Model**  
  - *Type:* Language Model node, integrated with AI Agent  
  - *Role:* Provides underlying language model for AI Agent.  
  - *Model:* GPT-4o-mini  
  - *Edge Cases:* API downtime, auth issues.

- **Enviar mensagem**  
  - *Type:* HTTP Request  
  - *Role:* Sends the generated summary as a WhatsApp message to the group via external API.  
  - *Config:* POST to `https://evolution.modexflow.com/message/sendText/Sabrina - Business` with group ID and summary text in body.  
  - *Auth:* EVOAPI header authentication.  
  - *Edge Cases:* Message delivery failures, API errors.

- **Replace Me**  
  - *Type:* NoOp  
  - *Role:* Placeholder node connected after message sending, potentially for future expansion or logging.  
  - *Edge Cases:* None.

- **Sticky Notes:** Provide context for the scheduled summary process, message grouping, validation, AI summarization, and sending.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                           | Input Node(s)                         | Output Node(s)                          | Sticky Note                                                                                       |
|----------------------------|----------------------------------|-----------------------------------------|-------------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------|
| Webhook                    | HTTP Webhook                     | Entry point receiving WhatsApp messages | -                                   | If                                    | ## Recebe mensagem\n### Verifica se é de Grupo                                                  |
| If                         | If                              | Filters for group messages               | Webhook                             | Edit Fields (true), Finaliza Fluxo (false) |                                                                                                 |
| Edit Fields                | Set                             | Extracts relevant fields from webhook   | If (true)                          | Switch                                | ### Organiza as mensagens e define o formato (texto, audio)                                     |
| Switch                     | Switch                          | Routes messages by type (text/audio)    | Edit Fields                       | Merge (text), Audio - HTTP Request (audio) |                                                                                                 |
| Audio - HTTP Request       | HTTP Request                    | Retrieves base64 audio from external API| Switch (audio)                    | Convert to File                      | ### Transcreve o áudio                                                                           |
| Convert to File            | Convert to File                 | Converts base64 to binary audio file    | Audio - HTTP Request              | Transcrever Áudio                     |                                                                                                 |
| Transcrever Áudio          | OpenAI Audio Transcription      | Transcribes audio to text                | Convert to File                   | Edit Fields 2                        |                                                                                                 |
| Edit Fields 2              | Set                             | Normalizes transcribed text              | Transcrever Áudio                 | Merge (audio branch)                  |                                                                                                 |
| Merge                      | Merge (Combine)                 | Combines text and audio branches         | Switch (text), Edit Fields 2      | Salvar mensagens na planilha          |                                                                                                 |
| Salvar mensagens na planilha| Google Sheets                  | Saves message log entries                 | Merge                            | Buscar Grupos                       | ### Salva Mensagens na Planilha                                                                |
| Buscar Grupos              | HTTP Request                    | Fetches group info from external API     | Salvar mensagens na planilha      | Verifica Aba Filtro de Grupos          | ### Busca Grupo                                                                                |
| Verifica Aba Filtro de Grupos| Google Sheets                | Checks if group exists in filter sheet   | Buscar Grupos                    | If1                                  | ### Vê se exista o Grupo na planilha                                                          |
| If1                        | If                              | Conditional on group presence             | Verifica Aba Filtro de Grupos    | Salva grupo para Envio (false), Finaliza Fluxo2 (true) | ### Ativa opção de Mandar Resumo\n### Se sim, finaliza o fluxo                                |
| Salva grupo para Envio     | Google Sheets                  | Adds new group to filter sheet            | If1 (false)                     | Finaliza Fluxo2                      |                                                                                                 |
| Finaliza Fluxo             | NoOp                           | Ends flow for non-group messages          | If (false)                     | -                                    | ### Se não for, finaliza o fluxo                                                               |
| Finaliza Fluxo2            | NoOp                           | Ends flow when group already in filter   | If1 (true), Salva grupo para Envio | -                                  | ### Se sim, finaliza o fluxo                                                                   |
| Schedule Trigger           | Schedule Trigger               | Triggers daily summary generation         | -                               | Acessar conversas na Planilha          | ### Roda todo dia\n#### 08am                                                                   |
| Acessar conversas na Planilha | Google Sheets               | Retrieves previous day messages           | Schedule Trigger               | Separar mensagens por grupos            | ### Acessa a data do último dia\n### Valida se o grupo está habilitado para mandar o resumo     |
| Separar mensagens por grupos| Code (JavaScript)             | Groups messages by group                   | Acessar conversas na Planilha  | Validar Grupo                        |                                                                                                 |
| Validar Grupo              | Google Sheets                  | Filters groups enabled to receive summaries| Separar mensagens por grupos  | Loop Over Items                    |                                                                                                 |
| Loop Over Items            | SplitInBatches                | Iterates over groups for summary generation| Validar Grupo                | AI Agent (true branch), (false branch empty) |                                                                                                 |
| AI Agent                   | LangChain Agent (OpenAI)      | Generates thematic summaries               | Loop Over Items               | Enviar mensagem                     | ### IA faz o resumo das mensagens do dia anterior e envia no grupo em questão.                |
| OpenAI Chat Model          | Language Model (OpenAI)        | Underlying AI model for AI Agent           | AI Agent                     | AI Agent (language model interface) |                                                                                                 |
| Enviar mensagem            | HTTP Request                  | Sends summary message to WhatsApp group    | AI Agent                     | Replace Me                        |                                                                                                 |
| Replace Me                | NoOp                         | Placeholder / flow continuation             | Enviar mensagem             | Loop Over Items                    |                                                                                                 |
| Sticky Note (various)      | Sticky Note                   | Contextual comments                         | -                             | -                                 | Covers relevant nodes with explanations as detailed in node groups above                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook node:**  
   - Type: HTTP Webhook  
   - Path: `resumir_grupos`  
   - Method: POST  
   - Purpose: Receive WhatsApp webhook messages.

2. **Add an If node to filter group messages:**  
   - Condition: Check if `{{$json.body.data.key.remoteJid}}` contains `@g.us`  
   - True branch continues; false branch connects to a NoOp node ("Finaliza Fluxo") to end flow.

3. **Add a Set node ("Edit Fields"):**  
   - Extract fields from webhook JSON into normalized fields:  
     - `remoteJid_grupo` = `body.data.key.remoteJid`  
     - `fromMe` = `body.data.key.fromMe`  
     - `remoteJid_enviou_mensagem` = `body.data.key.participant`  
     - `pessoa_enviou_mensagem` = `body.data.pushName`  
     - `mensagem` = `body.data.message.conversation`  
     - `messageType` = `body.data.messageType`  
     - `hora_recebida_mensagem` = `body.date_time`  
     - `evento_webhook` = `body.event`  
     - `instancia` = `body.instance`  
     - `id_msg` = `body.data.key.id`  
     - `resposta_mensagem_origem` = `body.data.contextInfo.quotedMessage.conversation`

4. **Add a Switch node:**  
   - Route based on `messageType`:  
     - `conversation` → text messages  
     - `audioMessage` → audio messages

5. **For text messages branch:**  
   - Connect directly to a Merge node (combining with audio branch later).

6. **For audio messages branch:**  
   - Add HTTP Request node ("Audio - HTTP Request"):  
     - POST to `https://evolution.modexflow.com/chat/getBase64FromMediaMessage/{{instance}}`  
     - Body: JSON with message ID for media retrieval  
     - Use header authentication credentials (EVOAPI).  
   - Add Convert to File node:  
     - Source: binary from previous node base64 property  
     - File name: "audio"  
     - MIME type from message metadata  
   - Add OpenAI node ("Transcrever Áudio"):  
     - Resource: Audio, Operation: Transcribe  
     - Language: Portuguese (pt)  
     - Credential: OpenAI API key  
   - Add Set node ("Edit Fields 2"):  
     - Assign `mensagem` field from transcription result (`text` property)  
   - Connect to the Merge node (audio branch).

7. **Merge node:**  
   - Combine mode, join by message content field to unify text and audio messages.

8. **Google Sheets node ("Salvar mensagens na planilha"):**  
   - Operation: Append or Update  
   - Sheet: "Página1" (gid=0) in specified Google Sheets document  
   - Map fields: Date (current date), message, message ID, webhook event, group ID, WhatsApp number (extracted), sender name  
   - Credential: Google Sheets OAuth2 configured

9. **HTTP Request node ("Buscar Grupos"):**  
   - GET group info from `https://evolution.modexflow.com/group/findGroupInfos/Sabrina - Business?groupJid={{ RemoteJid Grupo }}`  
   - Auth: EVOAPI header auth

10. **Google Sheets node ("Verifica Aba Filtro de Grupos"):**  
    - Lookup group ID in filter sheet (gid=1244658672)  
    - Credential: Google Sheets OAuth2

11. **If node ("If1"):**  
    - Condition: Group exists in filter sheet?  
    - If not: append group info with "Mandar Resumo" = FALSE to filter sheet via "Salva grupo para Envio" node  
    - If yes: end flow with "Finaliza Fluxo2" NoOp node

12. **Schedule Trigger node:**  
    - Runs daily at 08:00 (America/Sao_Paulo timezone)

13. **Google Sheets node ("Acessar conversas na Planilha"):**  
    - Read all messages from log sheet for previous day (filter by date)

14. **Code node ("Separar mensagens por grupos"):**  
    - Group messages by `RemoteJid Grupo` and concatenate messages with newlines

15. **Google Sheets node ("Validar Grupo"):**  
    - Filter groups where "Mandar Resumo" = true in filter sheet

16. **SplitInBatches node ("Loop Over Items"):**  
    - Iterate over each group to process individual summaries

17. **LangChain AI Agent node ("AI Agent"):**  
    - Use GPT-4o-mini via OpenAI credentials  
    - Input prompt: messages text grouped per group with strict instructions for faithful, thematic summary in Portuguese (max 1500 chars) including date intro, thematic blocks, bullet points, no hallucinations or generic terms  
    - Connect with underlying OpenAI Chat Model node

18. **HTTP Request node ("Enviar mensagem"):**  
    - POST summary back to WhatsApp group via external API  
    - Body params: group ID number, summary text  
    - Auth: EVOAPI header auth

19. **NoOp node ("Replace Me"):**  
    - Placeholder after sending message

20. **NoOp nodes ("Finaliza Fluxo", "Finaliza Fluxo2"):**  
    - To terminate flows where conditions do not permit further processing

21. **Add Sticky Notes throughout:**  
    - Provide context and documentation as per node groups above for clarity and maintenance.

22. **Credential Setup:**  
    - Configure Google Sheets OAuth2 with access to required spreadsheets  
    - Configure OpenAI API key with transcription and chat models enabled  
    - Configure EVOAPI header authentication for external API calls

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The AI Agent prompt enforces strict preservation of real names, thematic grouping, and Brazilian Portuguese. | Ensures fidelity and prevents hallucinations in summaries.                                                  |
| Workflow uses an external API "https://evolution.modexflow.com" for media conversion and WhatsApp message sending. | Custom API integration requiring header authentication.                                                     |
| Google Sheets are used both for message logging and group permission management ("Filtro de Grupos" sheet).  | Manage group-level control over summary sending permissions.                                                |
| Scheduled trigger is set for 8 AM Sao Paulo time zone to generate summaries daily for the previous day.      | Aligns summary timing with daily routines.                                                                   |
| Audio transcription uses OpenAI's audio transcription service (LangChain node).                              | Ensure OpenAI API key is enabled for audio transcription and chat models.                                   |
| Sticky Notes included in workflow provide essential developer context and should be preserved for maintenance. | Useful for onboarding and troubleshooting.                                                                   |

---

**Disclaimer:**  
The provided text exclusively originates from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---