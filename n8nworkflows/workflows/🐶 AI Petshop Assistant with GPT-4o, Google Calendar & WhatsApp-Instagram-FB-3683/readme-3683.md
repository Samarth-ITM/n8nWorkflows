🐶 AI Petshop Assistant with GPT-4o, Google Calendar & WhatsApp/Instagram/FB

https://n8nworkflows.xyz/workflows/---ai-petshop-assistant-with-gpt-4o--google-calendar---whatsapp-instagram-fb-3683


# 🐶 AI Petshop Assistant with GPT-4o, Google Calendar & WhatsApp/Instagram/FB

### 1. Workflow Overview

This workflow, titled **🐶 AI Petshop Assistant with GPT-4o, Google Calendar & WhatsApp/Instagram/FB**, is a sophisticated multichannel AI assistant designed specifically for petshops. It automates customer interactions across WhatsApp, Instagram Direct, Facebook Messenger, and other platforms, providing fast, friendly, and context-aware support. The assistant handles natural language conversations, voice notes, images, PDFs, booking appointments in real-time via Google Calendar, generating invoices/payments through Asaas, checking product stock via Supabase, and maintaining detailed customer histories.

The workflow is logically divided into the following main blocks:

- **1.1 Input Reception & Channel Routing**: Receives incoming messages from multiple channels, identifies message types, and routes accordingly.
- **1.2 Media Processing**: Handles voice notes, images, and PDFs by converting and extracting relevant content.
- **1.3 Customer Data Management**: Searches, registers, and updates customer data in Supabase.
- **1.4 AI Conversation & Context Handling**: Uses GPT-4o and LangChain agents to process messages, maintain conversation context, and generate responses.
- **1.5 Booking & Calendar Management**: Integrates with Google Calendar to check availability, create, update, or delete appointments.
- **1.6 Payment Processing**: Creates payment requests and invoices via Asaas API and sends payment links.
- **1.7 Product Stock Checking**: Queries Supabase inventory to answer product availability questions.
- **1.8 Follow-up & Timeout Management**: Manages follow-up messages and timeouts to ensure timely responses.
- **1.9 Outgoing Message Dispatch**: Sends responses back through the appropriate channel (WhatsApp, Instagram, Facebook, Evolution API).
- **1.10 File & Document Processing**: Handles document uploads, extraction, embedding, and updates vector stores for knowledge retrieval.
- **1.11 Validation & Utility Functions**: Validates CPF (Brazilian tax ID), CEP (postal code), and calculates delivery distances.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Channel Routing

**Overview:**  
This block receives incoming messages from various platforms via webhooks, determines the message source and type, and routes the flow accordingly.

**Nodes Involved:**  
- START (Webhook)  
- origem (Code)  
- Fixed Credentials (Set)  
- Data&Hora (Code)  
- MsgType (Switch)  
- Channel (Switch)  
- Filter WhatsApp (Filter)  
- Filter Insta&Face (Filter)  
- Get Reply Insta (HTTP Request)  
- Get Reply Facebook (HTTP Request)  
- Reply Message Insta (If)  
- Reply Message Wp (If)  
- Message Markup Insta (Set)  
- Message Markup Face (Set)  
- Message Markup Wp (Set)  
- No Marking (Set)  

**Node Details:**  
- **START**: Webhook node that listens for incoming messages from all integrated channels.  
- **origem**: Code node that identifies the origin platform of the message (WhatsApp, Instagram, Facebook, Evolution API).  
- **Fixed Credentials**: Sets API credentials and environment variables for downstream nodes.  
- **Data&Hora**: Generates or formats the current date and time for logging and scheduling.  
- **MsgType**: Switch node that branches logic based on the message type (text, image, audio, PDF, etc.).  
- **Channel**: Switch node that routes messages to platform-specific handlers.  
- **Filter WhatsApp / Filter Insta&Face**: Filters messages to ensure they come from the correct platform and are not from the business itself (to avoid loops).  
- **Get Reply Insta / Get Reply Facebook**: HTTP Request nodes that fetch message details or replies from Instagram and Facebook APIs.  
- **Reply Message Insta / Reply Message Wp**: Conditional nodes that decide whether to reply or not based on message content or sender.  
- **Message Markup Insta / Face / Wp**: Set nodes that prepare the message payload with appropriate markup for each platform.  
- **No Marking**: Default set node for messages that do not require special markup.

**Edge Cases / Failures:**  
- Webhook failures or misconfigurations.  
- API authentication errors for social media platforms.  
- Unexpected message types not handled by MsgType switch.  
- Messages from the business itself causing loops (filtered out).  

---

#### 1.2 Media Processing

**Overview:**  
Processes incoming media such as images, voice notes, and PDFs by downloading, converting, transcribing, or extracting text.

**Nodes Involved:**  
- Converter Img (ConvertToFile)  
- Img WhatsApp (OpenAI)  
- Img Instagram (OpenAI)  
- Down Img Insta&Face (HTTP Request)  
- Get Audio WpAPI (HTTP Request)  
- Down Audio WpAPI (HTTP Request)  
- Down Audio Insta&Face (HTTP Request)  
- Convert File Wp (ConvertToFile)  
- TranscAudio (OpenAI)  
- Converte PDF (ConvertToFile)  
- Extract PDF (ExtractFromFile)  
- Extract PDF Text (ExtractFromFile)  
- Extract from Text File (ExtractFromFile)  
- Extract from Excel (ExtractFromFile)  
- Convert to Google Doc (HTTP Request)  

**Node Details:**  
- **Converter Img**: Converts incoming image URLs or attachments into file format for processing.  
- **Img WhatsApp / Img Instagram**: Uses OpenAI to analyze or generate captions/responses based on images.  
- **Down Img Insta&Face**: Downloads images from Instagram and Facebook messages.  
- **Get Audio WpAPI / Down Audio WpAPI / Down Audio Insta&Face**: Downloads voice notes from WhatsApp and social platforms.  
- **Convert File Wp**: Converts downloaded audio files into a format suitable for transcription.  
- **TranscAudio**: Transcribes audio files into text using OpenAI.  
- **Converte PDF / Extract PDF / Extract PDF Text**: Converts PDFs to files and extracts text content for AI processing.  
- **Extract from Text File / Extract from Excel**: Extracts content from text and Excel files attached by users.  
- **Convert to Google Doc**: Converts documents to Google Docs format for further processing or storage.

**Edge Cases / Failures:**  
- File download failures due to broken URLs or API limits.  
- Unsupported file formats.  
- Transcription errors or timeouts.  
- Large files causing performance issues.  

---

#### 1.3 Customer Data Management

**Overview:**  
Manages customer data by searching existing records, registering new leads, updating information, and saving conversation history in Supabase.

**Nodes Involved:**  
- Lead Search (Supabase)  
- Lead Found (If)  
- Register New Lead (Supabase)  
- SalvaHistorico / SalvaHistorico1 / SalvaHistorico2 (Supabase)  
- up_consumer (SupabaseTool)  
- update_name (SupabaseTool)  
- Desbloquear IA / Bloquear IA (Supabase)  
- Block IA (If)  
- JSONhistorico / JSONhistorico1 / JSONhistorico2 / JSONhistorico3 (Code)  
- PhoneID (PhoneNumberParser)  

**Node Details:**  
- **Lead Search**: Queries Supabase to find if the customer already exists by phone or ID.  
- **Lead Found**: Conditional node branching logic based on customer existence.  
- **Register New Lead**: Inserts new customer data into Supabase if not found.  
- **SalvaHistorico** nodes: Save conversation and interaction history for context and records.  
- **up_consumer / update_name**: Update customer details in Supabase.  
- **Bloquear IA / Desbloquear IA**: Manage AI interaction blocking flags in the database to control conversation flow.  
- **Block IA**: Checks if AI interaction is currently blocked for a user.  
- **JSONhistorico** nodes: Prepare or parse JSON data for history saving.  
- **PhoneID**: Parses and normalizes phone numbers for consistent storage and lookup.

**Edge Cases / Failures:**  
- Database connection or query failures.  
- Duplicate records or race conditions on lead registration.  
- Incorrect phone number parsing.  
- AI blocking logic causing unintended conversation halts.

---

#### 1.4 AI Conversation & Context Handling

**Overview:**  
Processes user messages using GPT-4o and LangChain agents, maintains conversation context with memory, and generates intelligent, friendly responses.

**Nodes Involved:**  
- Modelo IA (lmChatOpenAi)  
- Best Agent AI (agent)  
- AI Agent FollowUP (agent)  
- AI Agent Products (agent)  
- Text Wrap (chainLlm)  
- OutputParser (outputParserStructured)  
- Memory50 (memoryPostgresChat)  
- n8n_chat_histories (toolVectorStore)  
- instrucoes (toolVectorStore)  
- Supabase instrucoes (vectorStoreSupabase)  
- SB n8n_chat_histories (vectorStoreSupabase)  
- Embed AI / Embed02 / Embed03 (embeddingsOpenAi)  
- Recursive Character (textSplitterRecursiveCharacterTextSplitter)  
- checar_cpf / checar_cep / create_payment / reagir_waapi / reagir_evo / stock_query / calendarAgent (toolHttpRequest)  
- Calculator / Calculator1 / Calc201 / Calc6301 (toolCalculator)  
- Model20 / ModelB2 / ModelB3 / ChatModel9 (lmChatOpenAi)  

**Node Details:**  
- **Modelo IA / Best Agent AI / AI Agent FollowUP / AI Agent Products**: Different LangChain agents configured for general conversation, follow-up, product inquiries, and calendar management.  
- **Text Wrap**: Chains and formats AI responses for clarity and structure.  
- **OutputParser**: Parses structured AI output for further processing.  
- **Memory50 / n8n_chat_histories / instrucoes**: Manage conversation memory and knowledge bases stored in Postgres or Supabase vector stores.  
- **Embed AI / Embed02 / Embed03**: Generate embeddings for documents or instructions to enhance AI understanding.  
- **Recursive Character**: Splits large texts recursively for embedding or summarization.  
- **checar_cpf / checar_cep / create_payment / reagir_waapi / reagir_evo / stock_query / calendarAgent**: Custom HTTP tool nodes invoking APIs for CPF validation, postal code lookup, payment creation, reaction handling, stock queries, and calendar operations.  
- **Calculator nodes**: Perform arithmetic or date/time calculations as needed.  
- **Model20 / ModelB2 / ModelB3 / ChatModel9**: Specialized OpenAI chat models for different AI tasks.

**Edge Cases / Failures:**  
- API rate limits or authentication errors with OpenAI or external APIs.  
- Parsing errors if AI output is malformed.  
- Memory store connection issues.  
- Timeout or slow responses from external APIs.  
- Unexpected user inputs causing AI confusion.

---

#### 1.5 Booking & Calendar Management

**Overview:**  
Manages real-time booking, event creation, updates, and deletions using Google Calendar integration.

**Nodes Involved:**  
- CalendarAgentAI (Webhook)  
- CalendarAgentAI1 (lmChatOpenAi)  
- Create Event / Create Event with Attendee / Update Event / Delete Event / Get Events (GoogleCalendarTool)  
- update_meet_link (SupabaseTool)  
- ChatModel9 (lmChatOpenAi)  
- Agendado (RespondToWebhook)  
- TryAgain (RespondToWebhook)  

**Node Details:**  
- **CalendarAgentAI**: Webhook entry point for calendar-related requests.  
- **CalendarAgentAI1**: AI model specialized in calendar management tasks.  
- **Create Event / Create Event with Attendee / Update Event / Delete Event / Get Events**: Google Calendar nodes to manipulate events.  
- **update_meet_link**: Updates meeting links or event metadata in Supabase.  
- **ChatModel9**: Chat model for calendar-related conversations.  
- **Agendado / TryAgain**: Webhook responses indicating success or failure in booking.

**Edge Cases / Failures:**  
- Google Calendar API authentication or quota issues.  
- Conflicting bookings or unavailable time slots.  
- Network errors causing event creation failures.  
- Missing or malformed event data.

---

#### 1.6 Payment Processing

**Overview:**  
Creates payment requests and invoices via Asaas API and sends payment links back to customers.

**Nodes Involved:**  
- Criar Cliente (HTTP Request)  
- Criar Cobrança (HTTP Request)  
- Add Link (Supabase)  
- WH Asaas (Webhook)  
- Response Asaas (RespondToWebhook)  
- Asaas Erro / Asaas Erro1 / Asaas Erro2 (RespondToWebhook)  

**Node Details:**  
- **Criar Cliente**: Creates a customer profile in Asaas.  
- **Criar Cobrança**: Creates a payment charge or invoice.  
- **Add Link**: Stores payment link in Supabase for record-keeping.  
- **WH Asaas**: Webhook to receive payment status updates from Asaas.  
- **Response Asaas**: Sends confirmation back to the chat.  
- **Asaas Erro nodes**: Handle and respond to errors during payment creation or processing.

**Edge Cases / Failures:**  
- API authentication or rate limits with Asaas.  
- Payment creation failures due to invalid data.  
- Webhook delivery failures.  
- Customer cancellation or payment timeouts.

---

#### 1.7 Product Stock Checking

**Overview:**  
Checks product availability in real-time by querying Supabase inventory and responds to customer inquiries.

**Nodes Involved:**  
- Check Stock (Webhook)  
- stock_query (toolHttpRequest)  
- Respond stock (RespondToWebhook)  
- AI Agent Products (agent)  

**Node Details:**  
- **Check Stock**: Webhook entry point for stock queries.  
- **stock_query**: HTTP request tool node querying Supabase or external stock API.  
- **Respond stock**: Sends stock availability response back to the customer.  
- **AI Agent Products**: AI agent specialized in product-related conversations.

**Edge Cases / Failures:**  
- Inventory data stale or inconsistent.  
- API failures or timeouts.  
- Incorrect product queries or ambiguous requests.

---

#### 1.8 Follow-up & Timeout Management

**Overview:**  
Manages follow-up messages, timeouts, and scheduling to ensure customers receive timely responses and reminders.

**Nodes Involved:**  
- Follow Up Search (Supabase)  
- Individual Followup (Supabase)  
- Waiting (Wait)  
- Aguarda (Wait)  
- Add Timeout (Supabase)  
- Busca Timeout (Supabase)  
- CalculoTime (Set)  
- Calculate Time (Code)  
- Filters: 10 Min, 24 Horas, 2 Horas, 72 Horas (Filter)  
- AI Agent FollowUP (agent)  

**Node Details:**  
- **Follow Up Search / Individual Followup**: Query and manage follow-up records.  
- **Waiting / Aguarda**: Wait nodes to pause execution for scheduled follow-ups.  
- **Add Timeout / Busca Timeout**: Manage timeout records in Supabase.  
- **CalculoTime / Calculate Time**: Calculate intervals and check timing rules.  
- **Filters**: Branch logic based on elapsed time since last interaction.  
- **AI Agent FollowUP**: AI agent to generate follow-up messages.

**Edge Cases / Failures:**  
- Timing miscalculations causing missed or premature follow-ups.  
- Database query failures.  
- Overlapping follow-up schedules.

---

#### 1.9 Outgoing Message Dispatch

**Overview:**  
Sends AI-generated responses and media back to customers through the appropriate communication channel.

**Nodes Involved:**  
- Send Instagram / Send Instagram1 (HTTP Request)  
- Send Facebook / Send Facebook1 (HTTP Request)  
- Send WhatsApp / Send WhatsApp1 (HTTP Request)  
- Send Evolution / Send Evolution1 (HTTP Request)  
- Send via Instagram / Send via Instagram1 (Filter)  
- Send via Face / Send via Face1 (Filter)  
- Send via WAAPI / Send via WAAPI1 (Filter)  
- Send via Evolution / Send via Evolution1 (Filter)  
- Set Field Evo (Set)  
- Evolution Imagem (Evolution API)  
- Response Img / Response Img Erro (RespondToWebhook)  
- Wait (Wait)  

**Node Details:**  
- **Send Instagram / Facebook / WhatsApp / Evolution**: HTTP requests to respective APIs to send messages or media.  
- **Send via ...**: Filters to route messages to the correct sending node based on channel.  
- **Set Field Evo**: Prepares message fields for Evolution API.  
- **Evolution Imagem**: Sends images via Evolution API.  
- **Response Img / Response Img Erro**: Webhook responses confirming success or failure of image sending.  
- **Wait**: Pauses to handle asynchronous image sending or API rate limits.

**Edge Cases / Failures:**  
- API authentication or quota limits.  
- Message formatting errors causing rejection.  
- Network timeouts.  
- Channel-specific restrictions or outages.

---

#### 1.10 File & Document Processing

**Overview:**  
Handles uploaded documents, extracts content, embeds data for AI knowledge bases, and manages file lifecycle.

**Nodes Involved:**  
- File Updated (GoogleDriveTrigger)  
- Definir ID (Set)  
- If (If)  
- Delete Old Doc Rows (Supabase)  
- Limit (Limit)  
- Set File Version (OpenAI)  
- Loop Over Items / Loop Over Items1 (SplitInBatches)  
- Aggregate (Aggregate)  
- Summarize (Summarize)  
- Delete File1 (GoogleDrive)  
- Enhanced Default (documentDefaultDataLoader)  
- Update Data RAG (vectorStoreSupabase)  
- Recursive Character (textSplitterRecursiveCharacterTextSplitter)  
- Embed AI / Embed02 / Embed03 (embeddingsOpenAi)  

**Node Details:**  
- **File Updated**: Triggered when a file is updated in Google Drive.  
- **Definir ID / If**: Logic to determine if file processing is needed.  
- **Delete Old Doc Rows**: Cleans old document records from Supabase.  
- **Limit**: Limits batch processing size.  
- **Set File Version**: Sets version metadata for documents.  
- **Loop Over Items**: Processes files in batches.  
- **Aggregate / Summarize**: Aggregates and summarizes document content.  
- **Delete File1**: Deletes processed files from Google Drive.  
- **Enhanced Default**: Loads documents for AI processing.  
- **Update Data RAG**: Updates vector store with new embeddings.  
- **Recursive Character**: Splits text for embedding.  
- **Embed AI / Embed02 / Embed03**: Generates embeddings for documents.

**Edge Cases / Failures:**  
- File access or permission errors.  
- Large files causing timeouts.  
- Embedding API failures.  
- Data consistency issues in vector store.

---

#### 1.11 Validation & Utility Functions

**Overview:**  
Validates Brazilian CPF and CEP codes, calculates delivery distances, and performs utility functions.

**Nodes Involved:**  
- CPF (Webhook)  
- CPF exist (If)  
- Validate CPF (Function)  
- CPF Válido (If)  
- RespondCPF2 / RespondCPF3 (RespondToWebhook)  
- CEP (Webhook)  
- CEP exist (If)  
- Checa CEP (Function)  
- CEP Válido (If)  
- RespondCEP / RespondCEP Erro (RespondToWebhook)  
- Mapbox Cliente (HTTP Request)  
- Mapbox Distancia (HTTP Request)  
- Apontamentos (Set)  
- TriggerMapbox (Webhook)  
- CalculateDelivery (toolHttpRequest)  
- Code4 (Code)  

**Node Details:**  
- **CPF / CEP**: Webhook entry points for validation requests.  
- **CPF exist / CEP exist**: Check if CPF or CEP exists in database or is valid.  
- **Validate CPF / Checa CEP**: Functions implementing validation logic.  
- **CPF Válido / CEP Válido**: Branch based on validation result.  
- **RespondCPF2 / RespondCPF3 / RespondCEP / RespondCEP Erro**: Send validation results back to requester.  
- **Mapbox Cliente / Mapbox Distancia**: HTTP requests to Mapbox API for geolocation and distance calculations.  
- **Apontamentos**: Sets data for delivery calculation.  
- **TriggerMapbox**: Webhook to trigger Mapbox-related processing.  
- **CalculateDelivery**: Tool node to calculate delivery cost or time.  
- **Code4**: Additional code logic for processing Mapbox results.

**Edge Cases / Failures:**  
- Invalid CPF or CEP input formats.  
- API failures or rate limits with Mapbox.  
- Network errors.  
- Incorrect geolocation data.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                      | Input Node(s)                      | Output Node(s)                     | Sticky Note                         |
|-------------------------|----------------------------------|------------------------------------|----------------------------------|----------------------------------|-----------------------------------|
| START                   | Webhook                          | Entry point for incoming messages  |                                  | origem                           |                                   |
| origem                  | Code                            | Identify message origin platform   | START                            | Fixed Credentials                |                                   |
| Fixed Credentials       | Set                             | Set API credentials and env vars   | origem                           | Data&Hora                       |                                   |
| Data&Hora               | Code                            | Generate current date/time          | Fixed Credentials                | MsgType                        |                                   |
| MsgType                 | Switch                         | Route by message type               | Data&Hora                       | Maps, Converter Img, Down Img Insta&Face, etc. |                                   |
| Channel                 | Switch                         | Route by platform                   | Message Markup Insta, Face, Wp  | Get Reply Insta, Get Reply Facebook |                                   |
| Filter WhatsApp         | Filter                         | Filter WhatsApp messages            | Add Response                    | Reply Message Wp                |                                   |
| Filter Insta&Face       | Filter                         | Filter Instagram/Facebook messages | Add Response                    | Reply Message Insta             |                                   |
| Get Reply Insta         | HTTP Request                   | Fetch Instagram message details    | Channel                        | Message Markup Insta            |                                   |
| Get Reply Facebook      | HTTP Request                   | Fetch Facebook message details     | Channel                        | Message Markup Face             |                                   |
| Reply Message Insta     | If                             | Decide to reply on Instagram       | Filter Insta&Face              | Channel                        |                                   |
| Reply Message Wp        | If                             | Decide to reply on WhatsApp         | Filter WhatsApp                | Message Markup Wp               |                                   |
| Message Markup Insta    | Set                            | Prepare Instagram message payload  | Get Reply Insta                | Response Refined               |                                   |
| Message Markup Face     | Set                            | Prepare Facebook message payload   | Get Reply Facebook             | Response Refined               |                                   |
| Message Markup Wp       | Set                            | Prepare WhatsApp message payload   | Reply Message Wp               | Response Refined               |                                   |
| No Marking              | Set                            | Default message payload             | Reply Message Wp               | Response Refined               |                                   |
| Converter Img           | ConvertToFile                  | Convert images to file format       | MsgType                       | Img WhatsApp                   |                                   |
| Img WhatsApp            | OpenAI                        | Analyze/process WhatsApp images     | Converter Img                 | Maps2                         |                                   |
| Img Instagram           | OpenAI                        | Analyze/process Instagram images    | Down Img Insta&Face           | Maps6                         |                                   |
| Down Img Insta&Face     | HTTP Request                  | Download images from social media   | MsgType                       | Img Instagram                 |                                   |
| Get Audio WpAPI         | HTTP Request                  | Download WhatsApp audio             | MsgType                       | Down Audio WpAPI              |                                   |
| Down Audio WpAPI        | HTTP Request                  | Download WhatsApp audio file        | Get Audio WpAPI               | TranscAudio                  |                                   |
| Down Audio Insta&Face   | HTTP Request                  | Download Instagram/Facebook audio   | MsgType                       | TranscAudio                  |                                   |
| Convert File Wp         | ConvertToFile                 | Convert audio files for transcription | MapAudioEvo                  | TranscAudio                  |                                   |
| TranscAudio             | OpenAI                        | Transcribe audio to text            | Down Audio WpAPI, Down Audio Insta&Face, Convert File Wp | Maps1                         |                                   |
| Converte PDF            | ConvertToFile                 | Convert PDF attachments             | MsgType                       | Extract PDF                  |                                   |
| Extract PDF             | ExtractFromFile               | Extract text from PDF               | Converte PDF                 | Maps5                         |                                   |
| Extract PDF Text        | ExtractFromFile               | Extract text from PDF               | Switch Type                  | Update Data RAG              |                                   |
| Extract from Text File  | ExtractFromFile               | Extract text from text files        | Switch Type                  | Update Data RAG              |                                   |
| Extract from Excel      | ExtractFromFile               | Extract data from Excel files        | Switch Type                  | Aggregate                   |                                   |
| Convert to Google Doc   | HTTP Request                  | Convert files to Google Docs format | Switch Type                  | Delete File1                |                                   |
| Lead Search             | Supabase                      | Search for existing customer        | Variaveis                   | Lead Found                  |                                   |
| Lead Found              | If                            | Branch if customer exists            | Lead Search                 | FromMe Insta&Face, FromMe WhatsApp, Register New Lead |                                   |
| Register New Lead       | Supabase                      | Register new customer                | Lead Found (false)           | Waiting                     |                                   |
| SalvaHistorico / 1 / 2  | Supabase                      | Save conversation history           | JSONhistorico nodes          |                             |                                   |
| up_consumer             | SupabaseTool                  | Update customer data                 |                             |                             |                                   |
| update_name             | SupabaseTool                  | Update customer name                 |                             |                             |                                   |
| Bloquear IA             | Supabase                      | Block AI interaction                 | Block IA                    | Desbloqueia IA              |                                   |
| Desbloqueia IA          | Supabase                      | Unblock AI interaction               | Block IA                    | MapTextHuman                |                                   |
| Block IA                | If                            | Check if AI interaction is blocked  |                             | Bloquear IA / Desbloqueia IA |                                   |
| JSONhistorico nodes     | Code                          | Prepare JSON for history saving     |                             | SalvaHistorico nodes        |                                   |
| PhoneID                 | PhoneNumberParser             | Parse and normalize phone numbers   | Sem Dados                   | Criar Cliente               |                                   |
| Modelo IA               | lmChatOpenAi                  | Main AI language model               | Prompt                      | Best Agent AI               |                                   |
| Best Agent AI           | Agent                         | Main AI agent for conversation      | Modelo IA                   | Text Wrap                   |                                   |
| AI Agent FollowUP       | Agent                         | AI agent for follow-up messages     | Model                      | Filter                      |                                   |
| AI Agent Products       | Agent                         | AI agent for product inquiries      | Model20                    | Respond stock               |                                   |
| Text Wrap               | chainLlm                      | Format AI response                   | Best Agent AI               | Segmentos, Filtro Reset     |                                   |
| OutputParser            | outputParserStructured        | Parse AI output structure           | Modelo IA                   | Text Wrap                   |                                   |
| Memory50                | memoryPostgresChat            | Conversation memory storage          |                             | AI Agent FollowUP           |                                   |
| n8n_chat_histories      | toolVectorStore               | Vector store for chat history        |                             | Best Agent AI               |                                   |
| instrucoes              | toolVectorStore               | Vector store for instructions        |                             | Best Agent AI               |                                   |
| Supabase instrucoes     | vectorStoreSupabase           | Supabase vector store for instructions | Embed02                   | instrucoes                  |                                   |
| SB n8n_chat_histories   | vectorStoreSupabase           | Supabase vector store for chat history | Embed03                   | n8n_chat_histories          |                                   |
| Embed AI / 02 / 03      | embeddingsOpenAi              | Generate embeddings for documents    |                             | Supabase instrucoes, SB n8n_chat_histories |                                   |
| Recursive Character     | textSplitterRecursiveCharacterTextSplitter | Split text recursively for embedding | Enhanced Default          | Embed AI                    |                                   |
| checar_cpf / checar_cep | toolHttpRequest               | Validate CPF and CEP via API         |                             | Best Agent AI               |                                   |
| create_payment          | toolHttpRequest               | Create payment via Asaas API         |                             | Best Agent AI               |                                   |
| reagir_waapi / reagir_evo | toolHttpRequest             | Handle reactions on WhatsApp/Evolution |                             | Best Agent AI               |                                   |
| stock_query             | toolHttpRequest               | Query product stock                  |                             | Best Agent AI               |                                   |
| calendarAgent           | toolHttpRequest               | Calendar API interaction             |                             | Best Agent AI               |                                   |
| Calculator nodes        | toolCalculator                | Perform calculations                  |                             | AI Agent FollowUP, AI Agent Products |                                   |
| CalendarAgentAI         | Webhook                      | Entry point for calendar requests    |                             | CalendarAgentAI1            |                                   |
| CalendarAgentAI1        | lmChatOpenAi                  | AI model for calendar management     | CalendarAgentAI             | Create/Update/Delete Event  |                                   |
| Create / Update / Delete Event | GoogleCalendarTool       | Manage Google Calendar events        | CalendarAgentAI1            | CalendarAgentAI1            |                                   |
| update_meet_link        | SupabaseTool                  | Update meeting links in database     |                             |                             |                                   |
| Agendado / TryAgain     | RespondToWebhook              | Send booking success or retry response | CalendarAgentAI1          |                             |                                   |
| Criar Cliente           | HTTP Request                 | Create customer in Asaas              | PhoneID                     | Criar Cobrança, Asaas Erro1 |                                   |
| Criar Cobrança          | HTTP Request                 | Create payment charge in Asaas       | Criar Cliente               | Add Link, Asaas Erro2       |                                   |
| Add Link                | Supabase                      | Store payment link                    | Criar Cobrança              | Response Asaas              |                                   |
| WH Asaas                | Webhook                      | Receive payment updates from Asaas   |                             | Sem Dados                   |                                   |
| Response Asaas          | RespondToWebhook              | Confirm payment status to user       | Add Link                    |                             |                                   |
| Asaas Erro nodes        | RespondToWebhook              | Handle payment errors                 | Criar Cliente, Criar Cobrança |                             |                                   |
| Check Stock             | Webhook                      | Entry point for stock queries         |                             | AI Agent Products           |                                   |
| Respond stock           | RespondToWebhook              | Send stock availability response     | AI Agent Products           |                             |                                   |
| Follow Up Search        | Supabase                      | Search for follow-up records          | Start Time                  | Loop                        |                                   |
| Individual Followup     | Supabase                      | Manage individual follow-ups          | Loop                       | FollowUP Eu                 |                                   |
| Waiting / Aguarda       | Wait                         | Pause for scheduled follow-ups        | Register New Lead, Add Timeout | Lead Search, Busca Timeout |                                   |
| Add Timeout / Busca Timeout | Supabase                  | Manage timeout records                 | JSONhistorico2              | Aguarda                     |                                   |
| CalculoTime / Calculate Time | Set / Code                | Calculate time intervals               | Busca Timeout               | Filters (10 Min, 24 Horas, etc.) |                                   |
| Filters (10 Min, 24 Horas, 2 Horas, 72 Horas) | Filter | Branch follow-up logic based on elapsed time | Calculate Time            | AI Agent FollowUP           |                                   |
| Send Instagram / Facebook / WhatsApp / Evolution | HTTP Request | Send messages/media to customers      | Send via ... filters        | 1,2s, NoOperation           |                                   |
| Send via ... filters    | Filter                       | Route outgoing messages to correct sender | Fixed Credentials1         | Send Instagram1, Send Facebook1, etc. |                                   |
| Set Field Evo           | Set                          | Prepare Evolution API message fields  | Send Images                 | Evolution Imagem            |                                   |
| Evolution Imagem        | Evolution API                | Send images via Evolution API          | Set Field Evo               | Wait, Response Img Erro     |                                   |
| Response Img / Response Img Erro | RespondToWebhook        | Confirm image send success or failure | Wait, Evolution Imagem      |                             |                                   |
| Wait                    | Wait                         | Pause for asynchronous operations     | Evolution Imagem            | Response Img                |                                   |
| File Updated            | GoogleDriveTrigger           | Trigger on file update in Google Drive |                             | Definir ID                  |                                   |
| Definir ID              | Set                          | Set file processing ID                 | File Updated                | If                         |                                   |
| If                      | If                           | Decide if file should be processed     | Definir ID                  | Delete Old Doc Rows, etc.   |                                   |
| Delete Old Doc Rows     | Supabase                      | Remove old document records            | If                         | Limit                      |                                   |
| Limit                   | Limit                        | Limit batch size                       | Delete Old Doc Rows          | Set File Version            |                                   |
| Set File Version        | OpenAI                       | Set document version metadata          | Limit                      | Loop Over Items             |                                   |
| Loop Over Items / Loop Over Items1 | SplitInBatches          | Batch process files                     | Set File Version, Segmentos | Switch Type, Send via ...   |                                   |
| Aggregate               | Aggregate                    | Aggregate extracted data                | Extract from Excel          | Summarize                  |                                   |
| Summarize               | Summarize                    | Summarize aggregated data               | Aggregate                   | Update Data RAG             |                                   |
| Delete File1            | GoogleDrive                  | Delete processed files                  | Convert to Google Doc       |                             |                                   |
| Enhanced Default        | documentDefaultDataLoader    | Load documents for AI processing        | Recursive Character         | Update Data RAG             |                                   |
| Update Data RAG         | vectorStoreSupabase          | Update vector store with embeddings     | Summarize, Enhanced Default |                             |                                   |
| CPF                     | Webhook                      | Entry point for CPF validation          |                             | CPF exist                  |                                   |
| CPF exist               | If                           | Check if CPF exists                      | CPF                        | Validate CPF, RespondCPF3   |                                   |
| Validate CPF            | Function                     | Validate CPF format and checksum        | CPF exist                  | CPF Válido                 |                                   |
| CPF Válido              | If                           | Branch on CPF validity                    | Validate CPF               | RespondCPF2, RespondCPF3    |                                   |
| RespondCPF2 / RespondCPF3 | RespondToWebhook            | Respond with CPF validation result       | CPF Válido, CPF exist      |                             |                                   |
| CEP                     | Webhook                      | Entry point for CEP validation          |                             | CEP exist                  |                                   |
| CEP exist               | If                           | Check if CEP exists                      | CEP                        | Checa CEP, RespondCEP Erro  |                                   |
| Checa CEP               | Function                     | Validate CEP format and lookup           | CEP exist                  | CEP Válido                 |                                   |
| CEP Válido              | If                           | Branch on CEP validity                    | Checa CEP                  | RespondCEP, RespondCEP Erro |                                   |
| RespondCEP / RespondCEP Erro | RespondToWebhook          | Respond with CEP validation result       | CEP Válido, CEP exist      |                             |                                   |
| Mapbox Cliente          | HTTP Request                 | Query Mapbox for client location          | Apontamentos               | Code4                      |                                   |
| Mapbox Distancia        | HTTP Request                 | Calculate distance via Mapbox API         | Code4                      | Code Resposta              |                                   |
| Apontamentos            | Set                          | Prepare data for Mapbox queries            | TriggerMapbox              | Mapbox Cliente             |                                   |
| TriggerMapbox           | Webhook                      | Trigger Mapbox distance calculation        |                             | Apontamentos               |                                   |
| CalculateDelivery       | toolHttpRequest              | Calculate delivery cost/time               |                             | Best Agent AI              |                                   |
| Code4                   | Code                         | Process Mapbox API results                  | Mapbox Cliente             | Mapbox Distancia           |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes for Incoming Messages**  
   - Create a webhook node named `START` to receive messages from WhatsApp, Instagram, Facebook, and Evolution API.  
   - Configure webhook URLs accordingly.

2. **Identify Message Origin**  
   - Add a `Code` node `origem` to parse incoming data and identify the platform source.  
   - Connect `START` → `origem`.

3. **Set Credentials and Environment Variables**  
   - Add a `Set` node `Fixed Credentials` to define API keys and environment variables for OpenAI, Supabase, Google Calendar, Asaas, and messaging APIs.  
   - Connect `origem` → `Fixed Credentials`.

4. **Generate Current Date & Time**  
   - Add a `Code` node `Data&Hora` to generate timestamps for logging and scheduling.  
   - Connect `Fixed Credentials` → `Data&Hora`.

5. **Determine Message Type**  
   - Add a `Switch` node `MsgType` to branch based on message content type (text, image, audio, PDF, etc.).  
   - Connect `Data&Hora` → `MsgType`.

6. **Route by Platform**  
   - Add a `Switch` node `Channel` to route messages to platform-specific handlers.  
   - Connect `MsgType` branches to appropriate nodes, then to `Channel`.

7. **Add Filters for Platform Messages**  
   - Add `Filter` nodes `Filter WhatsApp` and `Filter Insta&Face` to exclude messages from the business itself.  
   - Connect `Add Response` node to these filters.

8. **Fetch Message Details**  
   - Add HTTP Request nodes `Get Reply Insta` and `Get Reply Facebook` to fetch message details from APIs.  
   - Connect `Channel` outputs accordingly.

9. **Prepare Message Markup**  
   - Add `Set` nodes `Message Markup Insta`, `Message Markup Face`, and `Message Markup Wp` to format outgoing messages.  
   - Connect from respective reply nodes.

10. **Process Media**  
    - Add `ConvertToFile` nodes for images and PDFs (`Converter Img`, `Converte PDF`).  
    - Add HTTP Request nodes to download images and audio (`Down Img Insta&Face`, `Get Audio WpAPI`, etc.).  
    - Add OpenAI nodes to transcribe audio (`TranscAudio`) and analyze images (`Img WhatsApp`, `Img Instagram`).  
    - Connect `MsgType` branches to these nodes.

11. **Manage Customer Data**  
    - Add Supabase nodes `Lead Search`, `Register New Lead`, `SalvaHistorico` nodes to search, register, and save customer data.  
    - Add `If` nodes `Lead Found`, `Block IA` to branch logic.  
    - Add `PhoneID` node to parse phone numbers.  
    - Connect media processing and message markup outputs to customer data nodes.

12. **AI Conversation Handling**  
    - Add LangChain nodes: `Modelo IA`, `Best Agent AI`, `AI Agent FollowUP`, `AI Agent Products`.  
    - Add `Text Wrap` and `OutputParser` nodes to format and parse AI responses.  
    - Add memory nodes `Memory50`, vector stores `n8n_chat_histories`, `instrucoes`, and embedding nodes.  
    - Connect customer data outputs to AI nodes.

13. **Booking & Calendar Integration**  
    - Add webhook `CalendarAgentAI` for calendar requests.  
    - Add LangChain AI node `CalendarAgentAI1` for calendar conversation.  
    - Add Google Calendar nodes for event management (`Create Event`, `Update Event`, `Delete Event`, `Get Events`).  
    - Add SupabaseTool `update_meet_link` to update meeting info.  
    - Connect AI calendar nodes to Google Calendar nodes.

14. **Payment Processing**  
    - Add HTTP Request nodes `Criar Cliente`, `Criar Cobrança` for Asaas API calls.  
    - Add Supabase node `Add Link` to save payment links.  
    - Add webhook `WH Asaas` and response nodes for payment status.  
    - Connect customer data and AI nodes to payment nodes.

15. **Product Stock Checking**  
    - Add webhook `Check Stock` and HTTP Request `stock_query` to query inventory.  
    - Add response node `Respond stock`.  
    - Connect AI product agent to stock query nodes.

16. **Follow-up & Timeout Management**  
    - Add Supabase nodes `Follow Up Search`, `Individual Followup`, `Add Timeout`, `Busca Timeout`.  
    - Add `Wait` nodes `Waiting`, `Aguarda`.  
    - Add `Code` and `Filter` nodes to calculate and branch on time intervals.  
    - Connect AI follow-up agent to these nodes.

17. **Outgoing Message Dispatch**  
    - Add HTTP Request nodes to send messages on Instagram, Facebook, WhatsApp, Evolution API.  
    - Add filter nodes `Send via ...` to route messages.  
    - Add `Set Field Evo` and `Evolution Imagem` nodes for Evolution API image sending.  
    - Add `Wait` and response nodes for asynchronous handling.

18. **File & Document Processing**  
    - Add Google Drive Trigger `File Updated`.  
    - Add logic nodes `Definir ID`, `If`, `Delete Old Doc Rows`, `Limit`, `Set File Version`.  
    - Add batch processing nodes `Loop Over Items`.  
    - Add extraction nodes for PDFs, text, Excel.  
    - Add summarization and embedding nodes.  
    - Add vector store update nodes.

19. **Validation & Utility Functions**  
    - Add webhooks `CPF`, `CEP`.  
    - Add validation logic nodes `Validate CPF`, `Checa CEP`.  
    - Add response nodes for validation results.  
    - Add Mapbox API HTTP Request nodes for geolocation and distance.  
    - Add delivery calculation tool node.

20. **Final Testing and Deployment**  
    - Test each webhook and API integration individually.  
    - Verify AI responses and conversation flow.  
    - Confirm booking and payment flows work end-to-end.  
    - Monitor logs and handle edge cases.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow designed with love to provide smart, friendly AI assistance for petshops.              | Workflow description                                |
| Supports WhatsApp (official and Evolution API), Instagram Direct, Facebook Messenger.           | Multichannel support                                |
| Powered by GPT-4o, LangChain, Google Calendar, Asaas, Supabase, and more.                        | Technology stack                                    |
| Fully customizable tone, prompt, and flow to match brand style.                                 | Customization                                       |
| Saves all customer history securely in Supabase.                                               | Data storage                                        |
| Respects business hours and holidays for scheduling.                                           | Scheduling rules                                    |
| Buy Workflows and support available at https://iloveflows.gumroad.com                          | Purchase and support                                |
| Video and documentation links embedded in sticky notes within the workflow (not reproduced here) | Internal workflow notes                             |

---

This document provides a comprehensive, structured reference to understand, reproduce, and modify the AI Petshop Assistant workflow in n8n. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and AI agents to work effectively with the workflow.