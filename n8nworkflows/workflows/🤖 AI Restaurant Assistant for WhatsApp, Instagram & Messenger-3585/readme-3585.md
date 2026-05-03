🤖 AI Restaurant Assistant for WhatsApp, Instagram & Messenger

https://n8nworkflows.xyz/workflows/---ai-restaurant-assistant-for-whatsapp--instagram---messenger-3585


# 🤖 AI Restaurant Assistant for WhatsApp, Instagram & Messenger

### 1. Workflow Overview

This workflow, titled **"🤖 AI Restaurant Assistant for WhatsApp, Instagram & Messenger"**, is designed as a comprehensive AI-powered customer interaction assistant tailored for restaurants, burger joints, and delivery businesses. It automates customer communication across multiple messaging platforms, manages orders, validates delivery areas, generates payment links, and notifies the kitchen, all while maintaining a natural and kind conversational style using OpenAI’s GPT-4o.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception & Channel Detection**  
  Receives incoming messages from WhatsApp, Instagram Direct, Messenger, and Evolution API via webhooks, identifies the message type and source channel, and routes accordingly.

- **1.2 Message Preprocessing & Media Handling**  
  Processes incoming media such as images, audio, and PDFs; converts and extracts content as needed for further processing.

- **1.3 AI Conversational Processing**  
  Uses OpenAI GPT-4o and LangChain agents to generate natural language responses, handle upsells, and manage order dialogues.

- **1.4 Delivery Area Validation**  
  Checks customer ZIP codes using Google Maps and Mapbox APIs to confirm delivery eligibility within a configurable radius.

- **1.5 Payment Link Creation & Management**  
  Integrates with Asaas API to create payment links, handle payment confirmations, and manage customer billing.

- **1.6 Order Management & Kitchen Notification**  
  Sends confirmed orders to the kitchen via Evolution API or other configured channels.

- **1.7 Data Persistence & Session Management**  
  Stores all message histories, session data, and follow-ups securely in Supabase, including timeout and reset logic.

- **1.8 Scheduled Follow-ups & Timeout Handling**  
  Manages follow-up messages and session timeouts to maintain customer engagement and workflow continuity.

- **1.9 Media & Document Processing for Menu & Orders**  
  Handles menu images and document uploads, converting and embedding them for AI reference and customer display.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Channel Detection

- **Overview:**  
  This block receives incoming messages from multiple platforms via webhooks, determines the message type (text, image, audio, PDF, etc.), and identifies the source channel (WhatsApp, Instagram, Messenger, Evolution API). It routes the flow accordingly for specialized processing.

- **Nodes Involved:**  
  - `START` (Webhook)  
  - `origem` (Code)  
  - `Fixed Credentials` (Set)  
  - `Data&Hora` (Code)  
  - `MsgType` (Switch)  
  - `Channel` (Switch)  
  - `Filter Insta&Face` (Filter)  
  - `Filter WhatsApp` (Filter)  
  - `Filter` (Filter)  
  - `Reply Message Insta` (If)  
  - `Reply Message Wp` (If)  
  - `Get Reply Insta` (HTTP Request)  
  - `Get Reply Facebook` (HTTP Request)  
  - `Message Markup Insta` (Set)  
  - `Message Markup Face` (Set)  
  - `Message Markup Wp` (Set)  
  - `No Marking` (Set)  

- **Node Details:**  
  - `START`: Webhook node configured to receive incoming messages from all supported platforms.  
  - `origem`: Determines message origin and sets initial variables.  
  - `Fixed Credentials`: Sets API keys or credentials needed downstream.  
  - `Data&Hora`: Captures current timestamp for logging and session management.  
  - `MsgType`: Switch node that classifies message content type (text, image, audio, PDF, etc.).  
  - `Channel`: Switch node that routes messages based on platform (WhatsApp, Instagram, Facebook Messenger).  
  - `Filter Insta&Face`, `Filter WhatsApp`, `Filter`: Filters messages to ensure they are relevant and not from the business itself (avoids loops).  
  - `Reply Message Insta`, `Reply Message Wp`: Conditional nodes to handle replies per platform.  
  - `Get Reply Insta`, `Get Reply Facebook`: HTTP requests to fetch message details or metadata from Instagram and Facebook APIs.  
  - `Message Markup Insta`, `Message Markup Face`, `Message Markup Wp`: Prepare message formatting for replies on each platform.  
  - `No Marking`: Default message formatting fallback.  

- **Edge Cases / Failures:**  
  - Webhook failures or misconfiguration causing missed messages.  
  - API rate limits or authentication errors when fetching message details.  
  - Unexpected message types or unsupported media.  
  - Messages sent by the business itself causing loops (filtered).  

---

#### 1.2 Message Preprocessing & Media Handling

- **Overview:**  
  Processes incoming media messages, converting images, audio, and PDFs to usable formats, extracting text where applicable, and preparing content for AI processing.

- **Nodes Involved:**  
  - `Converter Img` (ConvertToFile)  
  - `Down Img Insta&Face` (HTTP Request)  
  - `Img Instagram` (OpenAI)  
  - `Img WhatsApp` (OpenAI)  
  - `Down Audio WpAPI` (HTTP Request)  
  - `Down Audio Insta&Face` (HTTP Request)  
  - `TranscAudio` (OpenAI)  
  - `MapAudioEvo` (Set)  
  - `Convert File Wp` (ConvertToFile)  
  - `Converte PDF` (ConvertToFile)  
  - `Extract PDF` (ExtractFromFile)  
  - `Maps`, `Maps1`, `Maps2`, `Maps5`, `Maps6` (Set)  

- **Node Details:**  
  - Media download nodes fetch media files from platform URLs.  
  - Conversion nodes transform media into files usable by AI or other nodes.  
  - `TranscAudio` uses OpenAI to transcribe audio messages.  
  - PDF extraction nodes extract text content for AI understanding.  
  - `Maps` nodes set location or address data extracted from messages or media.  

- **Edge Cases / Failures:**  
  - Media download failures due to expired URLs or API errors.  
  - Unsupported media formats.  
  - Transcription errors or timeouts.  
  - Large file sizes causing performance issues.  

---

#### 1.3 AI Conversational Processing

- **Overview:**  
  Uses OpenAI GPT-4o and LangChain agents to generate natural, kind, and context-aware responses, manage upsells, and handle order dialogues.

- **Nodes Involved:**  
  - `Modelo IA` (LM Chat OpenAI)  
  - `Best Agent AI` (LangChain Agent)  
  - `AI Agent FollowUP` (LangChain Agent)  
  - `AI Agent Products` (LangChain Agent)  
  - `Prompt` (Set)  
  - `OutputParser` (Structured Output Parser)  
  - `Text Wrap` (Chain LLM)  
  - `Model`, `Model20`, `ModelB2`, `ModelB3`, `OpenAI2` (LM Chat OpenAI)  
  - `Calculator`, `Calculator1`, `Calc201`, `Calc6301` (Tool Calculator)  
  - `Embed AI`, `Embed02`, `Embed03` (OpenAI Embeddings)  
  - `Postgres Memory`, `Memory50` (Postgres Chat Memory)  
  - `n8n_chat_histories`, `instrucoes`, `SB n8n_chat_histories`, `Supabase instrucoes` (Vector Store Supabase)  

- **Node Details:**  
  - Language model nodes configured with GPT-4o for conversational AI.  
  - Agents orchestrate multiple tools and memory for context-aware responses.  
  - Output parsers structure AI responses for downstream processing.  
  - Calculator tools handle numeric computations or order calculations.  
  - Embedding nodes create vector representations for semantic search and context retrieval.  
  - Memory nodes maintain conversation history in Postgres or Supabase vector stores.  
  - Prompt nodes set customized AI prompts reflecting brand voice and style.  

- **Edge Cases / Failures:**  
  - API rate limits or authentication errors with OpenAI.  
  - Parsing errors if AI response format is unexpected.  
  - Memory store connection issues.  
  - Timeout or latency in AI responses.  

---

#### 1.4 Delivery Area Validation

- **Overview:**  
  Validates customer ZIP codes and calculates delivery eligibility using Google Maps and Mapbox APIs, ensuring orders are only accepted within the configured delivery radius.

- **Nodes Involved:**  
  - `checar_cep` (HTTP Request tool)  
  - `TriggerMapbox` (Webhook)  
  - `Mapbox Cliente` (HTTP Request)  
  - `Mapbox Distancia` (HTTP Request)  
  - `Apontamentos` (Set)  
  - `Code4` (Code)  
  - `Se Erro` (If)  
  - `Response Address Sucess` (RespondToWebhook)  
  - `Response Address Error` (RespondToWebhook)  
  - `CEP`, `CEP exist`, `Checa CEP`, `CEP Válido`, `CEP exist` (If, Supabase, Function nodes)  
  - `RespondCEP`, `RespondCEP Erro` (RespondToWebhook)  

- **Node Details:**  
  - `checar_cep` calls external API to validate ZIP code.  
  - Mapbox nodes calculate distance between restaurant and customer address.  
  - Conditional nodes handle success or failure responses.  
  - Responses sent back to customer indicating delivery availability.  

- **Edge Cases / Failures:**  
  - Invalid or malformed ZIP codes.  
  - API errors or rate limits from Mapbox or Google Maps.  
  - Network timeouts.  
  - Customer outside delivery radius.  

---

#### 1.5 Payment Link Creation & Management

- **Overview:**  
  Creates payment links via Asaas API, manages customer creation, billing, and handles payment confirmation callbacks.

- **Nodes Involved:**  
  - `WH Asaas` (Webhook)  
  - `PhoneID` (Phone Number Parser)  
  - `Criar Cliente` (HTTP Request)  
  - `Criar Cobrança` (HTTP Request)  
  - `Add Link` (Supabase)  
  - `Response Asaas` (RespondToWebhook)  
  - `Asaas Erro`, `Asaas Erro1`, `Asaas Erro2` (RespondToWebhook)  
  - `Sem Dados` (If)  

- **Node Details:**  
  - `WH Asaas` receives payment status callbacks.  
  - `PhoneID` parses customer phone numbers for billing.  
  - `Criar Cliente` and `Criar Cobrança` create customer and payment records in Asaas.  
  - `Add Link` stores payment link in Supabase.  
  - Error handling nodes respond with appropriate messages on failure.  

- **Edge Cases / Failures:**  
  - Payment creation failures due to invalid data or API errors.  
  - Missing customer data causing errors.  
  - Payment confirmation delays or failures.  

---

#### 1.6 Order Management & Kitchen Notification

- **Overview:**  
  Sends confirmed orders to the kitchen system via Evolution API or other configured channels, ensuring smooth order processing.

- **Nodes Involved:**  
  - `send_pedido` (HTTP Request tool)  
  - `Send Pedido` (Webhook)  
  - `Set Field Pedido` (Set)  
  - `Wait1` (Wait)  
  - `Send Evolution` (HTTP Request)  
  - `Send via Evolution` (Filter)  
  - `reagir_evo` (HTTP Request tool)  

- **Node Details:**  
  - `send_pedido` formats and sends order data to kitchen API.  
  - `Send Pedido` receives order submission webhook.  
  - `Wait1` manages timing for order confirmation.  
  - Filters ensure orders are sent only via configured channels.  

- **Edge Cases / Failures:**  
  - Kitchen API downtime or errors.  
  - Order data formatting issues.  
  - Network timeouts.  

---

#### 1.7 Data Persistence & Session Management

- **Overview:**  
  Stores all messages, session data, and follow-up information in Supabase, including logic for session timeouts, resets, and blocking/unblocking AI interactions.

- **Nodes Involved:**  
  - `SalvaHistorico`, `SalvaHistorico1`, `SalvaHistorico2` (Supabase)  
  - `Add Timeout`, `Busca Timeout` (Supabase)  
  - `Bloquear IA`, `Desbloqueia IA` (Supabase)  
  - `Block IA` (If)  
  - `Bloqueio de IA` (Filter)  
  - `Follow Up Search`, `Individual Followup`, `FollowUP Eu` (Supabase, If)  
  - `Filtro Reset` (Filter)  
  - `JSONhistorico`, `JSONhistorico1`, `JSONhistorico2`, `JSONhistorico3` (Code)  

- **Node Details:**  
  - Supabase nodes insert or update session and message records.  
  - Timeout nodes manage session expiration and trigger waits.  
  - Blocking nodes prevent AI responses under certain conditions.  
  - Follow-up nodes manage scheduled customer engagement.  

- **Edge Cases / Failures:**  
  - Database connection issues.  
  - Data consistency errors.  
  - Timeout misconfigurations causing premature session ends.  

---

#### 1.8 Scheduled Follow-ups & Timeout Handling

- **Overview:**  
  Uses schedule triggers and wait nodes to manage follow-up messages, session timeouts, and customer re-engagement.

- **Nodes Involved:**  
  - `Start Time` (Schedule Trigger)  
  - `Start Deletion` (Schedule Trigger)  
  - `Aguarda` (Wait)  
  - `Waiting` (Wait)  
  - `Wait` (Wait)  
  - `Wait1` (Wait)  
  - `10 Min`, `24 Horas`, `2 Horas`, `72 Horas` (Filter)  
  - `Calculate Time`, `CalculoTime` (Code)  
  - `FollowUP Eu` (If)  
  - `Loop` (SplitInBatches)  

- **Node Details:**  
  - Schedule triggers initiate periodic checks or cleanup.  
  - Wait nodes pause workflow to allow asynchronous events.  
  - Filters determine appropriate follow-up timing.  
  - Code nodes calculate elapsed times for session management.  

- **Edge Cases / Failures:**  
  - Schedule trigger misfires or downtime.  
  - Wait node timeouts or premature continuation.  
  - Incorrect time calculations causing missed follow-ups.  

---

#### 1.9 Media & Document Processing for Menu & Orders

- **Overview:**  
  Handles uploading, converting, and embedding menu images and documents for AI reference and customer display.

- **Nodes Involved:**  
  - `Send Images` (Webhook)  
  - `Set Field Evo` (Set)  
  - `Evolution Imagem` (Evolution API node)  
  - `Response Img`, `Response Img Erro` (RespondToWebhook)  
  - `Wait` (Wait)  
  - `Download File` (Google Drive)  
  - `File Updated` (Google Drive Trigger)  
  - `Switch Type` (Switch)  
  - `Extract PDF Text`, `Extract from Text File`, `Extract from Excel` (ExtractFromFile)  
  - `Convert to Google Doc` (HTTP Request)  
  - `Delete File1` (Google Drive)  
  - `Aggregate`, `Summarize` (Aggregate, Summarize)  
  - `Recursive Character` (Text Splitter)  
  - `Update Data RAG` (Vector Store Supabase)  
  - `Enhanced Default` (Document Default Data Loader)  
  - `Set File Version` (OpenAI)  
  - `Calc201`, `Calc6301` (Calculator)  

- **Node Details:**  
  - Handles file uploads via webhook and Google Drive triggers.  
  - Extracts text from various document formats for AI embedding.  
  - Converts documents to Google Docs format for processing.  
  - Deletes temporary files after processing.  
  - Aggregates and summarizes document content.  
  - Splits text recursively for embedding.  
  - Updates vector store for retrieval-augmented generation (RAG).  

- **Edge Cases / Failures:**  
  - File format incompatibilities.  
  - Google Drive API errors or permission issues.  
  - Large documents causing performance degradation.  
  - Embedding or summarization failures.  

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                          | Input Node(s)                          | Output Node(s)                         | Sticky Note                              |
|------------------------|----------------------------------|----------------------------------------|--------------------------------------|--------------------------------------|-----------------------------------------|
| START                  | Webhook                          | Entry point for incoming messages      | -                                    | origem                               |                                         |
| origem                 | Code                             | Determines message origin               | START                                | Fixed Credentials                    |                                         |
| Fixed Credentials      | Set                              | Sets API credentials                    | origem                               | Data&Hora                           |                                         |
| Data&Hora              | Code                             | Captures current timestamp              | Fixed Credentials                    | MsgType                            |                                         |
| MsgType                | Switch                           | Classifies message type                 | Data&Hora                           | Maps, Converter Img, Down Img Insta&Face, etc. |                                         |
| Channel                | Switch                           | Routes messages by platform             | Reply Message Insta, Reply Message Wp | Get Reply Insta, Get Reply Facebook |                                         |
| Filter Insta&Face      | Filter                           | Filters Instagram & Facebook messages   | Add Response                        | Reply Message Insta                 |                                         |
| Filter WhatsApp        | Filter                           | Filters WhatsApp messages                | Add Response                        | Reply Message Wp                   |                                         |
| Reply Message Insta    | If                               | Conditional reply for Instagram         | Filter Insta&Face                   | Channel                           |                                         |
| Reply Message Wp       | If                               | Conditional reply for WhatsApp          | Filter WhatsApp                    | Channel                           |                                         |
| Get Reply Insta        | HTTP Request                    | Fetches Instagram message details       | Channel                           | Message Markup Insta               |                                         |
| Get Reply Facebook     | HTTP Request                    | Fetches Facebook message details        | Channel                           | Message Markup Face                |                                         |
| Message Markup Insta   | Set                              | Prepares Instagram reply formatting     | Get Reply Insta                   | Response Refined                  |                                         |
| Message Markup Face    | Set                              | Prepares Facebook reply formatting      | Get Reply Facebook                | Response Refined                  |                                         |
| Message Markup Wp      | Set                              | Prepares WhatsApp reply formatting      | Reply Message Wp                 | Response Refined                  |                                         |
| No Marking             | Set                              | Default message formatting               | Reply Message Wp                 | Response Refined                  |                                         |
| Converter Img          | ConvertToFile                   | Converts images to file format           | MsgType                         | Img WhatsApp                     |                                         |
| Down Img Insta&Face    | HTTP Request                    | Downloads Instagram/Facebook images      | MsgType                         | Img Instagram                   |                                         |
| Img Instagram          | OpenAI                         | Processes Instagram images with AI      | Down Img Insta&Face             | Maps6                           |                                         |
| Img WhatsApp           | OpenAI                         | Processes WhatsApp images with AI       | Converter Img                  | Maps2                           |                                         |
| Down Audio WpAPI       | HTTP Request                    | Downloads WhatsApp audio                 | MsgType                         | TranscAudio                    |                                         |
| Down Audio Insta&Face  | HTTP Request                    | Downloads Instagram/Facebook audio       | MsgType                         | TranscAudio                    |                                         |
| TranscAudio            | OpenAI                         | Transcribes audio messages               | Down Audio WpAPI, Down Audio Insta&Face | Maps1                       |                                         |
| MapAudioEvo            | Set                              | Sets audio metadata for Evolution API   | MsgType                         | Convert File Wp                |                                         |
| Convert File Wp        | ConvertToFile                   | Converts audio file for processing       | MapAudioEvo                    | TranscAudio                    |                                         |
| Modelo IA              | LM Chat OpenAI                 | Main AI conversational model             | Prompt                         | Best Agent AI                 |                                         |
| Best Agent AI          | LangChain Agent                | Orchestrates AI tools and memory        | Modelo IA                     | Text Wrap                     |                                         |
| AI Agent FollowUP      | LangChain Agent                | Handles follow-up conversations          | Model                        | Filter                        |                                         |
| AI Agent Products      | LangChain Agent                | Manages product upsells and orders       | Model20                      | Respond Bebida                |                                         |
| Prompt                 | Set                              | Sets AI prompt with brand voice          | Interferencia                 | Modelo IA                    |                                         |
| OutputParser           | Structured Output Parser       | Parses AI responses into structured data | Best Agent AI                 | Text Wrap                    |                                         |
| Text Wrap              | Chain LLM                     | Wraps AI output for batch processing     | Best Agent AI                 | Segmentos, Filtro Reset       |                                         |
| Segmentos              | SplitOut                      | Splits text into segments for processing | Text Wrap                    | Loop Over Items1             |                                         |
| Loop Over Items1       | SplitInBatches                | Processes segmented text in batches      | Segmentos                    | Send via Evolution, Instagram, Face, WAAPI |                                         |
| checar_cep             | HTTP Request tool             | Validates ZIP code via external API      | Best Agent AI                | CEP exist                   |                                         |
| CEP exist              | If                               | Checks if ZIP code exists in DB           | CEP                         | Checa CEP, RespondCEP Erro  |                                         |
| Checa CEP              | Function                      | Processes ZIP code validation logic       | CEP exist                   | CEP Válido                  |                                         |
| CEP Válido             | If                               | Validates ZIP code delivery eligibility   | Checa CEP                   | RespondCEP, RespondCEP Erro |                                         |
| RespondCEP             | RespondToWebhook              | Sends success response for ZIP validation | CEP Válido                  | -                           |                                         |
| RespondCEP Erro        | RespondToWebhook              | Sends error response for ZIP validation   | CEP exist, CEP Válido       | -                           |                                         |
| WH Asaas               | Webhook                      | Receives payment status callbacks         | -                           | Sem Dados                   |                                         |
| PhoneID                | Phone Number Parser          | Parses customer phone numbers              | Sem Dados                   | Criar Cliente               |                                         |
| Criar Cliente          | HTTP Request                 | Creates customer in Asaas API              | PhoneID                     | Criar Cobrança, Asaas Erro1 |                                         |
| Criar Cobrança         | HTTP Request                 | Creates payment charge in Asaas API        | Criar Cliente               | Add Link, Asaas Erro2       |                                         |
| Add Link               | Supabase                     | Stores payment link in database            | Criar Cobrança              | Response Asaas              |                                         |
| Response Asaas         | RespondToWebhook             | Responds to payment webhook                 | Add Link                    | -                           |                                         |
| Asaas Erro             | RespondToWebhook             | Handles errors in payment creation          | Sem Dados                   | -                           |                                         |
| Asaas Erro1            | RespondToWebhook             | Handles errors in customer creation         | Criar Cliente               | -                           |                                         |
| Asaas Erro2            | RespondToWebhook             | Handles errors in charge creation           | Criar Cobrança              | -                           |                                         |
| send_pedido            | HTTP Request tool             | Sends order to kitchen API                   | Best Agent AI               | -                           |                                         |
| Send Pedido            | Webhook                      | Receives order submission                     | -                           | Set Field Pedido            |                                         |
| Set Field Pedido       | Set                          | Prepares order data for sending               | Send Pedido                 | Wait1                      |                                         |
| Wait1                  | Wait                         | Waits before confirming order sent            | Set Field Pedido            | Response4                   |                                         |
| Send Evolution         | HTTP Request                 | Sends order to Evolution API                   | Send via Evolution          | 1,2s                       |                                         |
| Send via Evolution     | Filter                       | Filters orders to send via Evolution API       | Loop Over Items1            | Send Evolution             |                                         |
| SalvaHistorico         | Supabase                     | Saves message history                          | JSONhistorico               | -                           |                                         |
| SalvaHistorico1        | Supabase                     | Saves message history                          | JSONhistorico1              | -                           |                                         |
| SalvaHistorico2        | Supabase                     | Saves message history                          | JSONhistorico3              | -                           |                                         |
| Add Timeout            | Supabase                     | Adds session timeout record                     | JSONhistorico2              | Aguarda                    |                                         |
| Busca Timeout          | Supabase                     | Checks for session timeout                       | Aguarda                    | CalculoTime                |                                         |
| Bloquear IA            | Supabase                     | Blocks AI interaction                            | Block IA                   | -                           |                                         |
| Desbloqueia IA         | Supabase                     | Unblocks AI interaction                          | Block IA                   | MapTextHuman               |                                         |
| Block IA               | If                           | Determines if AI should be blocked                | Desbloqueia IA             | Bloquear IA, Desbloqueia IA |                                         |
| Follow Up Search       | Supabase                     | Searches for follow-up messages                   | Start Time                 | Lead Found                 |                                         |
| Individual Followup    | Supabase                     | Handles individual follow-up                      | Loop                       | FollowUP Eu                |                                         |
| FollowUP Eu            | If                           | Checks if follow-up is for current user            | Individual Followup        | Data&Hora1, Loop           |                                         |
| Loop                   | SplitInBatches               | Processes follow-ups in batches                     | FollowUP Eu                | Individual Followup, NoOperation1 |                                         |
| Calculate Time         | Code                         | Calculates elapsed time for session management      | Map FollowUP               | 10 Min, 24 Horas, 2 Horas |                                         |
| 10 Min                 | Filter                       | Filters follow-ups for 10-minute timeout             | Calculate Time             | AI Agent FollowUP          |                                         |
| 24 Horas               | Filter                       | Filters follow-ups for 24-hour timeout               | Calculate Time             | AI Agent FollowUP          |                                         |
| 2 Horas                | Filter                       | Filters follow-ups for 2-hour timeout                | Calculate Time             | AI Agent FollowUP          |                                         |
| AI Agent FollowUP      | LangChain Agent              | Handles follow-up AI conversations                    | 10 Min, 24 Horas, 2 Horas | Filter                    |                                         |
| Start Time             | Schedule Trigger             | Triggers follow-up search periodically                 | -                         | Follow Up Search           |                                         |
| Start Deletion         | Schedule Trigger             | Triggers scheduled deletion of old data                | -                         | n8n                       |                                         |
| n8n                    | n8n                          | Deletes old document rows                               | Start Deletion             | -                         |                                         |
| Download File          | Google Drive                 | Downloads files for processing                          | Send Pedido                | Loop Over Items            |                                         |
| File Updated           | Google Drive Trigger         | Triggers on file update                                 | -                         | Definir ID                 |                                         |
| Definir ID             | Set                          | Defines file ID for processing                           | File Updated               | If                        |                                         |
| If                     | If                           | Conditional branching for file processing                | Definir ID                 | Delete Old Doc Rows        |                                         |
| Delete Old Doc Rows    | Supabase                     | Deletes old document rows                                 | If                        | Limit                      |                                         |
| Limit                  | Limit                        | Limits number of processed items                          | Delete Old Doc Rows        | Set File Version           |                                         |
| Set File Version       | OpenAI                       | Sets file version metadata                                | Limit                      | Loop Over Items            |                                         |
| Loop Over Items        | SplitInBatches               | Processes files in batches                                 | Download File              | Switch Type, Download File |                                         |
| Switch Type            | Switch                       | Routes files based on type (PDF, Text, Excel, etc.)       | Loop Over Items            | Extract PDF Text, Extract from Text File, Extract from Excel, Convert to Google Doc |                                         |
| Extract PDF Text       | ExtractFromFile              | Extracts text from PDF                                     | Switch Type                | Update Data RAG            |                                         |
| Extract from Text File | ExtractFromFile              | Extracts text from text files                              | Switch Type                | Update Data RAG            |                                         |
| Extract from Excel     | ExtractFromFile              | Extracts data from Excel files                             | Switch Type                | Aggregate                  |                                         |
| Convert to Google Doc  | HTTP Request                 | Converts files to Google Docs format                       | Switch Type                | Delete File1               |                                         |
| Delete File1           | Google Drive                 | Deletes temporary files                                    | Convert to Google Doc      | -                         |                                         |
| Aggregate              | Aggregate                   | Aggregates extracted data                                  | Extract from Excel         | Summarize                  |                                         |
| Summarize              | Summarize                   | Summarizes aggregated data                                 | Aggregate                  | Update Data RAG            |                                         |
| Update Data RAG        | Vector Store Supabase        | Updates vector store with embedded document data          | Extract PDF Text, Extract from Text File, Summarize, Enhanced Default | -                         |                                         |
| Enhanced Default       | Document Default Data Loader | Loads document data for embedding                           | Recursive Character        | Update Data RAG            |                                         |
| Recursive Character    | Text Splitter               | Splits text recursively for embedding                      | -                         | Enhanced Default           |                                         |
| Set Field Evo          | Set                          | Sets fields for Evolution API image sending                | Send Images                | Evolution Imagem           |                                         |
| Evolution Imagem       | Evolution API                | Sends images to Evolution API                              | Set Field Evo              | Wait, Response Img Erro    |                                         |
| Wait                   | Wait                         | Waits for Evolution API response                            | Evolution Imagem           | Response Img               |                                         |
| Response Img           | RespondToWebhook             | Responds to image send webhook                              | Wait                      | -                         |                                         |
| Response Img Erro      | RespondToWebhook             | Responds with error if image sending fails                  | Evolution Imagem           | -                         |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes for Input Reception:**  
   - Create a webhook node named `START` to receive messages from WhatsApp, Instagram, Messenger, and Evolution API. Configure it with appropriate webhook URLs.  

2. **Determine Message Origin and Set Credentials:**  
   - Add a `Code` node `origem` to parse incoming data and identify the source platform.  
   - Add a `Set` node `Fixed Credentials` to assign API keys and credentials for OpenAI, Supabase, Google Maps, Asaas, etc.  

3. **Capture Timestamp:**  
   - Add a `Code` node `Data&Hora` to capture the current date and time for session tracking.  

4. **Classify Message Type:**  
   - Add a `Switch` node `MsgType` to classify incoming messages by type (text, image, audio, PDF, etc.).  

5. **Route by Channel:**  
   - Add a `Switch` node `Channel` to route messages based on platform (WhatsApp, Instagram, Facebook Messenger).  

6. **Filter Messages:**  
   - Add `Filter` nodes `Filter Insta&Face`, `Filter WhatsApp`, and `Filter` to exclude messages sent by the business or irrelevant content.  

7. **Fetch Message Details:**  
   - Add HTTP Request nodes `Get Reply Insta` and `Get Reply Facebook` to fetch message metadata from respective APIs.  

8. **Prepare Message Markup:**  
   - Add `Set` nodes `Message Markup Insta`, `Message Markup Face`, and `Message Markup Wp` to format replies per platform.  
   - Add a fallback `Set` node `No Marking`.  

9. **Handle Media Processing:**  
   - Add `ConvertToFile` nodes `Converter Img`, `Converte PDF`, and `Convert File Wp` to convert images, PDFs, and audio files.  
   - Add HTTP Request nodes `Down Img Insta&Face`, `Down Audio WpAPI`, `Down Audio Insta&Face` to download media.  
   - Add OpenAI nodes `Img Instagram`, `Img WhatsApp`, and `TranscAudio` for image processing and audio transcription.  
   - Add `ExtractFromFile` nodes for PDF and text extraction.  
   - Add `Set` nodes `Maps`, `Maps1`, `Maps2`, `Maps5`, `Maps6`, and `MapAudioEvo` to set metadata from media.  

10. **Configure AI Conversational Nodes:**  
    - Add OpenAI GPT-4o nodes `Modelo IA`, `Model`, `Model20`, `ModelB2`, `ModelB3`, `OpenAI2` for different AI tasks.  
    - Add LangChain agent nodes `Best Agent AI`, `AI Agent FollowUP`, `AI Agent Products` to orchestrate AI tools.  
    - Add `Set` node `Prompt` to define AI prompt with brand voice.  
    - Add `OutputParser` node to parse AI responses.  
    - Add `Chain LLM` node `Text Wrap` and `SplitOut` node `Segmentos` for response processing.  
    - Add calculator nodes `Calculator`, `Calculator1`, `Calc201`, `Calc6301` for numeric computations.  
    - Add embedding nodes `Embed AI`, `Embed02`, `Embed03` and vector store nodes `Supabase instrucoes`, `SB n8n_chat_histories`, `n8n_chat_histories`, `instrucoes` for memory and context.  
    - Add Postgres memory nodes `Postgres Memory`, `Memory50`.  

11. **Implement Delivery Validation:**  
    - Add HTTP Request nodes `checar_cep`, `Mapbox Cliente`, `Mapbox Distancia` to validate ZIP codes and calculate distances.  
    - Add webhook `TriggerMapbox` to receive delivery validation requests.  
    - Add `If` nodes `CEP exist`, `CEP Válido`, `Se Erro` to handle validation logic.  
    - Add `RespondToWebhook` nodes `Response Address Sucess`, `Response Address Error`, `RespondCEP`, `RespondCEP Erro` to respond to customers.  
    - Add function node `Checa CEP` for ZIP code processing.  

12. **Set Up Payment Processing:**  
    - Add webhook `WH Asaas` to receive payment callbacks.  
    - Add phone number parser `PhoneID`.  
    - Add HTTP Request nodes `Criar Cliente`, `Criar Cobrança` to create customers and charges in Asaas.  
    - Add Supabase node `Add Link` to store payment links.  
    - Add `RespondToWebhook` nodes `Response Asaas`, `Asaas Erro`, `Asaas Erro1`, `Asaas Erro2` for payment responses and error handling.  
    - Add `If` node `Sem Dados` to check for missing data.  

13. **Order Management and Kitchen Notification:**  
    - Add webhook `Send Pedido` to receive order submissions.  
    - Add `Set` node `Set Field Pedido` to prepare order data.  
    - Add `Wait` node `Wait1` to manage timing.  
    - Add HTTP Request node `send_pedido` to send orders to kitchen API.  
    - Add HTTP Request node `Send Evolution` and filter `Send via Evolution` for sending orders via Evolution API.  

14. **Data Persistence and Session Management:**  
    - Add Supabase nodes `SalvaHistorico`, `SalvaHistorico1`, `SalvaHistorico2` to save message and session data.  
    - Add Supabase nodes `Add Timeout`, `Busca Timeout` to manage session timeouts.  
    - Add Supabase nodes `Bloquear IA`, `Desbloqueia IA` to block/unblock AI responses.  
    - Add `If` node `Block IA` and filter `Bloqueio de IA` for AI blocking logic.  
    - Add Supabase nodes `Follow Up Search`, `Individual Followup`, `FollowUP Eu` for follow-up management.  
    - Add `Filter` node `Filtro Reset` and code nodes `JSONhistorico`, `JSONhistorico1`, `JSONhistorico2`, `JSONhistorico3` for data processing.  

15. **Scheduled Follow-ups and Timeout Handling:**  
    - Add schedule triggers `Start Time` and `Start Deletion` for periodic tasks.  
    - Add wait nodes `Aguarda`, `Waiting`, `Wait`, `Wait1` for asynchronous delays.  
    - Add filter nodes `10 Min`, `24 Horas`, `2 Horas`, `72 Horas` for timing conditions.  
    - Add code nodes `Calculate Time`, `CalculoTime` for time calculations.  
    - Add split batches node `Loop` for processing follow-ups.  

16. **Media & Document Processing for Menu & Orders:**  
    - Add webhook `Send Images` for receiving menu images.  
    - Add `Set` node `Set Field Evo` and Evolution API node `Evolution Imagem` for image sending.  
    - Add `Wait` node and `RespondToWebhook` nodes `Response Img`, `Response Img Erro` for image response handling.  
    - Add Google Drive nodes `Download File`, `File Updated`, `Delete File1` for file management.  
    - Add switch node `Switch Type` to route file types.  
    - Add extract nodes `Extract PDF Text`, `Extract from Text File`, `Extract from Excel`.  
    - Add HTTP Request node `Convert to Google Doc` for document conversion.  
    - Add aggregate and summarize nodes `Aggregate`, `Summarize` for data processing.  
    - Add text splitter `Recursive Character` and vector store update `Update Data RAG`.  
    - Add document loader `Enhanced Default` and OpenAI node `Set File Version`.  

17. **Connect all nodes according to the logical flow described, ensuring proper error handling and retry mechanisms where applicable.**

18. **Configure all credentials securely in n8n:**  
    - OpenAI API key  
    - Supabase credentials  
    - Google Maps and Mapbox API keys  
    - Asaas API key  
    - Evolution API credentials  
    - Facebook and Instagram API credentials  

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow created with care for restaurant and delivery businesses to automate customer service.  | Workflow description                                                                                      |
| Supabase table template for session and message storage: [Supabase Sheet Template](https://drive.google.com/file/d/1KUGb0ujyfwy1yUroxZTqpUp9c5-FTXf6/view?usp=sharing) | Setup instructions                                                                                         |
| Contact for customization and support: Chat via WhatsApp [https://wa.me/5517991557874](https://wa.me/5517991557874) (+55 17 99155-7874) | Support and customization                                                                                  |
| Compatible with n8n Cloud and self-hosted n8n; all credentials stored securely in n8n credential manager. | Security and deployment notes                                                                              |
| Uses OpenAI GPT-4o for natural language processing and LangChain agents for orchestration.       | AI technology details                                                                                      |
| Integrates with WhatsApp API, Instagram Direct, Messenger, Evolution API, Google Maps, Asaas API, Supabase. | Integration overview                                                                                       |
| Delivery radius default is 10km but configurable.                                                | Business logic parameter                                                                                   |
| Menu images can be uploaded from Google Drive, website, or CDN links.                            | Media management                                                                                           |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the AI Restaurant Assistant workflow in n8n. It covers all nodes, logic blocks, and integration points to anticipate potential errors and ensure smooth operation.