🤖 AI-Powered WhatsApp Assistant for Restaurants & Delivery Automation

https://n8nworkflows.xyz/workflows/---ai-powered-whatsapp-assistant-for-restaurants---delivery-automation-3043


# 🤖 AI-Powered WhatsApp Assistant for Restaurants & Delivery Automation

### 1. Workflow Overview

This workflow, titled **"🤖 AI-Powered WhatsApp Assistant for Restaurants & Delivery Automation"**, is designed to automate and optimize restaurant order processing and delivery management via WhatsApp. It targets restaurants, burger joints, and delivery businesses aiming to streamline customer interactions, order handling, delivery fee calculation, and integration with CRM and POS systems.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Authentication**  
  Captures incoming WhatsApp messages via webhook, authenticates requests, and initiates processing.

- **1.2 Customer & Chat Data Management**  
  Retrieves, creates, and updates customer and chat records in the database (Supabase/Postgres).

- **1.3 AI-Powered Conversation & Order Processing**  
  Uses OpenAI language models and LangChain components to interpret customer messages, classify intents, and manage conversational flows.

- **1.4 Delivery Fee Calculation & Address Handling**  
  Calculates delivery distance and fees dynamically based on customer location data.

- **1.5 Order Finalization & CRM Integration**  
  Saves order details, forwards commands to the restaurant system, and updates order status to customers.

- **1.6 Document & Knowledge Base Management**  
  Handles document ingestion (PDF, Excel, text), embedding generation, and vector store updates for AI knowledge retrieval.

- **1.7 Voice Generation & Multimedia Handling**  
  Converts text responses to voice using ElevenLabs and Evolution APIs for enhanced customer interaction.

- **1.8 Scheduled Maintenance & Cleanup**  
  Periodically cleans up old data and manages session states.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Authentication

- **Overview:**  
  This block receives WhatsApp messages through a webhook, verifies the authenticity of incoming requests, and prepares data for processing.

- **Nodes Involved:**  
  - `input evolution` (Webhook)  
  - `Auth` (If)  
  - `dados_para_atendimento_humano` (Set)  
  - `Switch5` (Switch)  
  - `PARAR ISIS` (Redis)  
  - `Verifica Atendimento Humano` (Redis)  
  - `Switch Block1` (Switch)  
  - `Variáveis` (Set)  
  - `Date & Time1` (Date & Time)  
  - `Supabase4` (Supabase)  
  - `If5` (If)  
  - `N8N Labz YT1` (Supabase)  
  - `Gerar sessionID1` (Crypto)  
  - `Supabase5` (Supabase)  
  - `Code3` (Code)

- **Node Details:**

  - **`input evolution`**  
    - Type: Webhook  
    - Role: Entry point for WhatsApp messages, listens for incoming HTTP POST requests.  
    - Configuration: Receives JSON payloads containing WhatsApp message data.  
    - Outputs to: `Auth` node.  
    - Edge Cases: Invalid or malformed webhook requests, missing authentication keys.

  - **`Auth`**  
    - Type: If  
    - Role: Validates API key or other auth parameters in the webhook payload.  
    - Outputs to: `dados_para_atendimento_humano` if authorized.  
    - Edge Cases: Unauthorized access attempts.

  - **`dados_para_atendimento_humano`**  
    - Type: Set  
    - Role: Prepares data for human agent handoff if needed.  
    - Outputs to: `Switch5`.

  - **`Switch5`**  
    - Type: Switch  
    - Role: Decides between stopping automation (`PARAR ISIS`) or checking human attendance (`Verifica Atendimento Humano`).  
    - Outputs: `PARAR ISIS` or `Verifica Atendimento Humano`.

  - **`PARAR ISIS` & `Verifica Atendimento Humano`**  
    - Type: Redis  
    - Role: Manage flags in Redis cache for stopping or verifying human attendance.  
    - Outputs: `Switch Block1`.

  - **`Switch Block1`**  
    - Type: Switch  
    - Role: Further routing based on Redis flags or other conditions.  
    - Outputs: `Variáveis`.

  - **`Variáveis`**  
    - Type: Set  
    - Role: Sets workflow variables, possibly session or context data.  
    - Outputs: `Date & Time1`.

  - **`Date & Time1`**  
    - Type: Date & Time  
    - Role: Adds timestamp or date-related data.  
    - Outputs: `Supabase4`.

  - **`Supabase4`**  
    - Type: Supabase  
    - Role: Queries or updates Supabase database related to session or user data.  
    - Outputs: `If5`.

  - **`If5`**  
    - Type: If  
    - Role: Conditional logic based on Supabase query results.  
    - Outputs: `N8N Labz YT1` or `Gerar sessionID1`.

  - **`N8N Labz YT1`**  
    - Type: Supabase  
    - Role: Further database interaction, possibly retrieving user/session info.  
    - Outputs: `Code3`.

  - **`Gerar sessionID1`**  
    - Type: Crypto  
    - Role: Generates a session ID for new sessions.  
    - Outputs: `Supabase5`.

  - **`Supabase5`**  
    - Type: Supabase  
    - Role: Inserts new session record.  
    - Outputs: `Code3`.

  - **`Code3`**  
    - Type: Code  
    - Role: Custom JavaScript logic for session or flow control.  
    - Outputs: Next processing nodes (not directly connected here).

- **Version Requirements:**  
  - Webhook node requires n8n version supporting webhook v2.  
  - Redis nodes require Redis credentials configured.  
  - Supabase nodes require Supabase credentials.

- **Potential Failures:**  
  - Authentication failure.  
  - Redis connection issues.  
  - Supabase query or insert errors.  
  - Session ID generation errors.

---

#### 1.2 Customer & Chat Data Management

- **Overview:**  
  Manages customer and chat data storage and retrieval, ensuring customer details and chat history are saved and updated in the database.

- **Nodes Involved:**  
  - `FindPhone` (Supabase)  
  - `If3` (If)  
  - `AddChat-Supabase` (Supabase)  
  - `UpdateChat-Supabase` (Supabase)  
  - `Merge1` (Merge)  
  - `CreateMessage-Supabase` (Supabase)  
  - `ListChats-Supabase` (Supabase)  
  - `ListMessages-Supabase` (Supabase)  
  - `Aggregate` (Aggregate)  
  - `Code1` (Code)  
  - `DisableMessage-Supabase` (Supabase)  
  - `Loop Over Items1` (SplitInBatches)  
  - `Loop Over Items3` (SplitInBatches)  
  - `Wait1` (Wait)  
  - `UpdateChat-Supabase` (Supabase)

- **Node Details:**

  - **`FindPhone`**  
    - Type: Supabase  
    - Role: Searches for existing customer by phone number.  
    - Outputs: `If3`.

  - **`If3`**  
    - Type: If  
    - Role: Checks if customer exists.  
    - Outputs: `AddChat-Supabase` (new customer) or `UpdateChat-Supabase` (existing customer).

  - **`AddChat-Supabase`**  
    - Type: Supabase  
    - Role: Inserts new chat record for new customer.  
    - Outputs: `Merge1`.

  - **`UpdateChat-Supabase`**  
    - Type: Supabase  
    - Role: Updates chat record for existing customer.  
    - Outputs: `Merge1`.

  - **`Merge1`**  
    - Type: Merge  
    - Role: Combines outputs from add or update chat operations.  
    - Outputs: `CreateMessage-Supabase`.

  - **`CreateMessage-Supabase`**  
    - Type: Supabase  
    - Role: Inserts new chat message into database.  
    - Outputs: Next processing steps.

  - **`ListChats-Supabase` & `ListMessages-Supabase`**  
    - Type: Supabase  
    - Role: Retrieves chat and message history for processing or cleanup.  
    - Outputs: `Loop Over Items1` and `Aggregate`.

  - **`Aggregate`**  
    - Type: Aggregate  
    - Role: Aggregates message data for summarization or analysis.  
    - Outputs: `Code1`.

  - **`Code1`**  
    - Type: Code  
    - Role: Custom logic for processing aggregated data.  
    - Outputs: Further nodes.

  - **`DisableMessage-Supabase`**  
    - Type: Supabase  
    - Role: Marks messages as disabled or processed.  
    - Outputs: `Loop Over Items1`.

  - **`Loop Over Items1` & `Loop Over Items3`**  
    - Type: SplitInBatches  
    - Role: Processes large datasets in batches to avoid timeouts.  
    - Outputs: `ListMessages-Supabase` or `Evolution API2`.

  - **`Wait1`**  
    - Type: Wait  
    - Role: Delays processing to manage rate limits or timing.  
    - Outputs: `DisableMessage-Supabase`.

- **Version Requirements:**  
  - Supabase nodes require configured Supabase credentials.  
  - SplitInBatches nodes require n8n version supporting batching.

- **Potential Failures:**  
  - Database connection failures.  
  - Data consistency issues if updates fail.  
  - Batch processing timeouts.

---

#### 1.3 AI-Powered Conversation & Order Processing

- **Overview:**  
  Uses OpenAI and LangChain nodes to interpret customer messages, classify intents, manage conversation flows, and generate AI responses.

- **Nodes Involved:**  
  - `OpenAI Chat Model` (LangChain LLM Chat)  
  - `OpenAI Chat Model1`  
  - `OpenAI Chat Model2`  
  - `OpenAI Chat Model3`  
  - `Basic LLM Chain` (LangChain Chain LLM)  
  - `Text Classifier` (LangChain Text Classifier)  
  - `Delivery AI` (LangChain Agent)  
  - `OutputParser1` (LangChain Output Parser Structured)  
  - `Parser Chain` (LangChain Chain LLM)  
  - `busca_cliente` (LangChain Tool Workflow)  
  - `busca_endereco` (LangChain Tool Workflow)  
  - `envia_comanda` (LangChain Tool Workflow)  
  - `salvar_cliente` (LangChain Tool Workflow)  
  - `Postgres Chat Memory` (LangChain Memory Postgres Chat)

- **Node Details:**

  - **`OpenAI Chat Model` series**  
    - Type: LangChain LLM Chat nodes  
    - Role: Generate AI conversational responses based on input context.  
    - Configuration: Uses OpenAI API with configured model parameters.  
    - Inputs: Customer messages, chat history.  
    - Outputs: AI-generated text responses.

  - **`Basic LLM Chain`**  
    - Type: Chain LLM  
    - Role: Chains multiple LLM calls or logic steps.  
    - Outputs: Supabase node for data persistence.

  - **`Text Classifier`**  
    - Type: Text Classifier  
    - Role: Classifies incoming messages to determine intent or next action.  
    - Outputs: Branches workflow based on classification.

  - **`Delivery AI`**  
    - Type: LangChain Agent  
    - Role: Orchestrates AI tools and chains for delivery-specific tasks.  
    - Inputs: Config node sets parameters.  
    - Outputs: Switch3 node for further branching.

  - **`OutputParser1` & `Parser Chain`**  
    - Type: Output Parser & Chain LLM  
    - Role: Parses AI output into structured data and chains further processing.  
    - Outputs: `Segmentos1` node for batch processing.

  - **`busca_cliente`, `busca_endereco`, `envia_comanda`, `salvar_cliente`**  
    - Type: LangChain Tool Workflow nodes  
    - Role: Invoke sub-workflows for customer search, address retrieval, order sending, and customer saving.  
    - Inputs/Outputs: Integrated into Delivery AI agent flow.

  - **`Postgres Chat Memory`**  
    - Type: LangChain Memory Postgres Chat  
    - Role: Stores conversational memory in Postgres for context retention.

- **Version Requirements:**  
  - Requires LangChain nodes and OpenAI credentials.  
  - Sub-workflows must be properly configured and accessible.

- **Potential Failures:**  
  - OpenAI API rate limits or errors.  
  - Parsing errors if AI output is malformed.  
  - Sub-workflow invocation failures.  
  - Memory storage issues.

---

#### 1.4 Delivery Fee Calculation & Address Handling

- **Overview:**  
  Calculates delivery distance and fees dynamically using AI tools and workflows, based on customer address data.

- **Nodes Involved:**  
  - `Calculadora` (LangChain Tool Calculator)  
  - `busca_endereco` (LangChain Tool Workflow)  
  - `Delivery AI` (LangChain Agent) (shared with conversation block)

- **Node Details:**

  - **`Calculadora`**  
    - Type: LangChain Tool Calculator  
    - Role: Performs mathematical calculations such as delivery fee based on distance.  
    - Inputs: Distance data from address lookup.  
    - Outputs: Delivery fee data to AI agent.

  - **`busca_endereco`**  
    - Type: LangChain Tool Workflow  
    - Role: Retrieves or verifies customer address details.  
    - Outputs: Provides address data for calculation.

- **Version Requirements:**  
  - Requires LangChain nodes and configured sub-workflows.

- **Potential Failures:**  
  - Incorrect or missing address data.  
  - Calculation errors or invalid inputs.

---

#### 1.5 Order Finalization & CRM Integration

- **Overview:**  
  Saves finalized order details, forwards commands to restaurant systems, and updates customers on order status.

- **Nodes Involved:**  
  - `envia_comanda` (LangChain Tool Workflow)  
  - `salvar_cliente` (LangChain Tool Workflow)  
  - `AddChat-Supabase`, `UpdateChat-Supabase`, `CreateMessage-Supabase` (Supabase) (shared with data management)  
  - `Merge1` (Merge)

- **Node Details:**

  - **`envia_comanda`**  
    - Type: LangChain Tool Workflow  
    - Role: Sends order commands to restaurant or POS system.  
    - Inputs: Order details from AI processing.  
    - Outputs: Confirmation or status updates.

  - **`salvar_cliente`**  
    - Type: LangChain Tool Workflow  
    - Role: Saves or updates customer data in CRM or database.  
    - Outputs: Data persistence confirmation.

- **Version Requirements:**  
  - Sub-workflows must be configured for CRM/POS integration.

- **Potential Failures:**  
  - Integration failures with external systems.  
  - Data sync issues.

---

#### 1.6 Document & Knowledge Base Management

- **Overview:**  
  Handles ingestion of documents (PDF, Excel, text), extracts text, generates embeddings, and updates vector stores for AI knowledge retrieval.

- **Nodes Involved:**  
  - `File Created` & `File Updated` (Google Drive Trigger)  
  - `Download File` (Google Drive)  
  - `Switch` (Switch)  
  - `Extract PDF Text`, `Extract Document Text`, `Extract from Excel` (ExtractFromFile)  
  - `Aggregate1` (Aggregate)  
  - `Summarize` (Summarize)  
  - `Insert into Supabase Vectorstore` (LangChain Vector Store Supabase)  
  - `Embeddings OpenAI` & `Embeddings OpenAI2` (LangChain Embeddings OpenAI)  
  - `Character Text Splitter` (LangChain Text Splitter Character)  
  - `Default Data Loader1` (LangChain Document Default Data Loader)  
  - `Delete Old Doc Rows` (Supabase)  
  - `Code` (Code)  
  - `Loop Over Items` (SplitInBatches)

- **Node Details:**

  - **`File Created` & `File Updated`**  
    - Type: Google Drive Trigger  
    - Role: Detect new or updated files to process.  
    - Outputs: `Set File ID`.

  - **`Download File`**  
    - Type: Google Drive  
    - Role: Downloads the file for processing.  
    - Outputs: `Switch`.

  - **`Switch`**  
    - Type: Switch  
    - Role: Routes file processing based on file type (PDF, Excel, Document).  
    - Outputs: Corresponding extract nodes.

  - **`Extract PDF Text`, `Extract Document Text`, `Extract from Excel`**  
    - Type: ExtractFromFile  
    - Role: Extracts text content from files.  
    - Outputs: `Aggregate1` or `Summarize`.

  - **`Aggregate1`**  
    - Type: Aggregate  
    - Role: Aggregates extracted data.  
    - Outputs: `Summarize`.

  - **`Summarize`**  
    - Type: Summarize  
    - Role: Summarizes extracted content for embedding.  
    - Outputs: `Insert into Supabase Vectorstore`.

  - **`Insert into Supabase Vectorstore`**  
    - Type: LangChain Vector Store Supabase  
    - Role: Inserts embeddings into vector database for AI retrieval.  
    - Outputs: `Loop Over Items`.

  - **`Embeddings OpenAI` & `Embeddings OpenAI2`**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates vector embeddings from text.  
    - Outputs: Vector store insertion.

  - **`Character Text Splitter`**  
    - Type: LangChain Text Splitter Character  
    - Role: Splits text into manageable chunks for embedding.

  - **`Default Data Loader1`**  
    - Type: LangChain Document Default Data Loader  
    - Role: Loads documents for processing.

  - **`Delete Old Doc Rows`**  
    - Type: Supabase  
    - Role: Cleans old document data before inserting new.

  - **`Code`**  
    - Type: Code  
    - Role: Custom logic for document processing.

  - **`Loop Over Items`**  
    - Type: SplitInBatches  
    - Role: Processes documents in batches.

- **Version Requirements:**  
  - Google Drive nodes require OAuth2 credentials.  
  - LangChain nodes require OpenAI credentials.  
  - Supabase credentials required.

- **Potential Failures:**  
  - File download or access errors.  
  - Extraction failures due to unsupported file formats.  
  - API rate limits on OpenAI.  
  - Vector store insertion errors.

---

#### 1.7 Voice Generation & Multimedia Handling

- **Overview:**  
  Converts AI-generated text responses into voice audio files using ElevenLabs and Evolution APIs, enhancing customer interaction with voice messages.

- **Nodes Involved:**  
  - `Code2` (Code)  
  - `ElevenLabsGenerateVoice` (HTTP Request)  
  - `Audio-Base64-Extract from File` (ExtractFromFile)  
  - `Evolution API3` (Evolution API)  
  - `Evolution API2` (Evolution API)  
  - `Evolution API` (Evolution API)  
  - `OpenAI3` (LangChain LLM Chat)  
  - `If1` (If)  
  - `Parser Chain` (LangChain Chain LLM)  
  - `OutputParser1` (LangChain Output Parser Structured)

- **Node Details:**

  - **`Code2`**  
    - Type: Code  
    - Role: Prepares text for voice synthesis.  
    - Outputs: `ElevenLabsGenerateVoice`.

  - **`ElevenLabsGenerateVoice`**  
    - Type: HTTP Request  
    - Role: Calls ElevenLabs API to generate voice audio from text.  
    - Outputs: `Audio-Base64-Extract from File`.

  - **`Audio-Base64-Extract from File`**  
    - Type: ExtractFromFile  
    - Role: Extracts base64 audio data from response.  
    - Outputs: `Evolution API3`.

  - **`Evolution API3`, `Evolution API2`, `Evolution API`**  
    - Type: Evolution API nodes  
    - Role: Additional voice synthesis or audio processing steps.  
    - Outputs: Various downstream nodes.

  - **`OpenAI3`**  
    - Type: LangChain LLM Chat  
    - Role: Generates or processes text for voice synthesis.  
    - Outputs: `Parser Chain`.

  - **`If1`**  
    - Type: If  
    - Role: Conditional branching based on voice generation success or type.  
    - Outputs: `Evolution API` or `Parser Chain`.

  - **`Parser Chain` & `OutputParser1`**  
    - Type: LangChain nodes  
    - Role: Parse and structure AI outputs for voice generation.

- **Version Requirements:**  
  - HTTP Request node configured with ElevenLabs API key.  
  - Evolution API nodes require credentials.  
  - LangChain nodes require OpenAI credentials.

- **Potential Failures:**  
  - API authentication errors.  
  - Audio extraction failures.  
  - Network timeouts.

---

#### 1.8 Scheduled Maintenance & Cleanup

- **Overview:**  
  Periodically triggers cleanup tasks such as disabling old messages and managing session states.

- **Nodes Involved:**  
  - `Schedule Trigger` (Schedule Trigger)  
  - `ListChats-Supabase` (Supabase)  
  - `Loop Over Items1` (SplitInBatches)  
  - `ListMessages-Supabase` (Supabase)  
  - `Aggregate` (Aggregate)  
  - `Code1` (Code)  
  - `Delete Old Doc Rows` (Supabase)  
  - `Wait1` (Wait)  
  - `DisableMessage-Supabase` (Supabase)

- **Node Details:**

  - **`Schedule Trigger`**  
    - Type: Schedule Trigger  
    - Role: Initiates periodic workflow runs for maintenance.  
    - Outputs: `ListChats-Supabase`.

  - **`ListChats-Supabase` & `ListMessages-Supabase`**  
    - Type: Supabase  
    - Role: Retrieves chats and messages for cleanup.  
    - Outputs: `Loop Over Items1` and `Aggregate`.

  - **`Loop Over Items1`**  
    - Type: SplitInBatches  
    - Role: Processes data in batches.  
    - Outputs: `ListMessages-Supabase`.

  - **`Aggregate` & `Code1`**  
    - Type: Aggregate and Code  
    - Role: Summarizes and processes data for cleanup.  
    - Outputs: `Delete Old Doc Rows`.

  - **`Delete Old Doc Rows`**  
    - Type: Supabase  
    - Role: Deletes outdated document rows.  
    - Outputs: `Code`.

  - **`Wait1`**  
    - Type: Wait  
    - Role: Delays to manage timing.  
    - Outputs: `DisableMessage-Supabase`.

  - **`DisableMessage-Supabase`**  
    - Type: Supabase  
    - Role: Marks messages as disabled after processing.

- **Version Requirements:**  
  - Schedule Trigger requires n8n version supporting scheduling.  
  - Supabase credentials required.

- **Potential Failures:**  
  - Scheduling misconfiguration.  
  - Database cleanup failures.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                                  | Input Node(s)                 | Output Node(s)               | Sticky Note                          |
|-------------------------------|----------------------------------|-------------------------------------------------|------------------------------|-----------------------------|------------------------------------|
| input evolution               | Webhook                          | Entry point for WhatsApp messages                |                              | Auth                        |                                    |
| Auth                         | If                               | Authenticates incoming requests                  | input evolution              | dados_para_atendimento_humano |                                    |
| dados_para_atendimento_humano | Set                              | Prepares data for human attendance                | Auth                        | Switch5                     |                                    |
| Switch5                      | Switch                           | Routes to stop or verify human attendance         | dados_para_atendimento_humano | PARAR ISIS, Verifica Atendimento Humano |                                    |
| PARAR ISIS                   | Redis                            | Stops automation flag                             | Switch5                     | Switch Block1               |                                    |
| Verifica Atendimento Humano  | Redis                            | Checks human attendance flag                      | Switch5                     | Switch Block1               |                                    |
| Switch Block1                | Switch                           | Further routing based on flags                    | PARAR ISIS, Verifica Atendimento Humano | Variáveis                   |                                    |
| Variáveis                    | Set                              | Sets workflow variables                           | Switch Block1               | Date & Time1                |                                    |
| Date & Time1                 | Date & Time                      | Adds timestamp                                   | Variáveis                   | Supabase4                   |                                    |
| Supabase4                    | Supabase                         | Queries session/user data                         | Date & Time1                | If5                        |                                    |
| If5                          | If                               | Checks session existence                          | Supabase4                   | N8N Labz YT1, Gerar sessionID1 |                                    |
| N8N Labz YT1                 | Supabase                         | Retrieves session info                            | If5                         | Code3                      |                                    |
| Gerar sessionID1             | Crypto                           | Generates new session ID                          | If5                         | Supabase5                  |                                    |
| Supabase5                    | Supabase                         | Inserts new session                              | Gerar sessionID1            | Code3                      |                                    |
| Code3                        | Code                             | Custom session logic                             | N8N Labz YT1, Supabase5     |                              |                                    |
| FindPhone                   | Supabase                         | Finds customer by phone                           |                              | If3                        |                                    |
| If3                          | If                               | Checks if customer exists                         | FindPhone                   | AddChat-Supabase, UpdateChat-Supabase |                                    |
| AddChat-Supabase            | Supabase                         | Adds new chat record                             | If3                         | Merge1                     |                                    |
| UpdateChat-Supabase         | Supabase                         | Updates existing chat record                      | If3                         | Merge1                     |                                    |
| Merge1                       | Merge                            | Merges add/update chat outputs                    | AddChat-Supabase, UpdateChat-Supabase | CreateMessage-Supabase      |                                    |
| CreateMessage-Supabase      | Supabase                         | Inserts chat message                             | Merge1                      |                              |                                    |
| ListChats-Supabase          | Supabase                         | Lists chats for maintenance                       | Schedule Trigger            | Loop Over Items1            |                                    |
| ListMessages-Supabase       | Supabase                         | Lists messages for maintenance                    | Loop Over Items1            | Aggregate                  |                                    |
| Aggregate                   | Aggregate                        | Aggregates messages                              | ListMessages-Supabase       | Code1                      |                                    |
| Code1                       | Code                             | Processes aggregated data                         | Aggregate                   | Delete Old Doc Rows         |                                    |
| DisableMessage-Supabase     | Supabase                         | Disables processed messages                      | Wait1                       | Loop Over Items1            |                                    |
| Loop Over Items1            | SplitInBatches                  | Batch processing of messages                      | ListChats-Supabase          | ListMessages-Supabase       |                                    |
| Loop Over Items3            | SplitInBatches                  | Batch processing for Evolution API calls         | no.op                       | Evolution API2             |                                    |
| Wait1                       | Wait                             | Delay for rate limiting                           | Supabase                   | DisableMessage-Supabase     |                                    |
| OpenAI Chat Model           | LangChain LLM Chat              | AI conversation generation                        | Config                      | Delivery AI                |                                    |
| OpenAI Chat Model1          | LangChain LLM Chat              | AI document search                               | Embeddings OpenAI2           | busca_documentos           |                                    |
| OpenAI Chat Model2          | LangChain LLM Chat              | Basic LLM chain processing                        |                            | Basic LLM Chain            |                                    |
| OpenAI Chat Model3          | LangChain LLM Chat              | Text classification                              |                            | Text Classifier            |                                    |
| Basic LLM Chain             | LangChain Chain LLM             | Chains LLM calls                                 | OpenAI Chat Model2           | Supabase, SetConfig        |                                    |
| Text Classifier             | LangChain Text Classifier       | Classifies message intent                         | OpenAI Chat Model3           | Basic LLM Chain, Wait1     |                                    |
| Delivery AI                 | LangChain Agent                | Orchestrates AI tools and workflows               | Config                      | Switch3                    |                                    |
| OutputParser1               | LangChain Output Parser Structured | Parses AI output into structured data             | OpenAI3                     | Parser Chain               |                                    |
| Parser Chain                | LangChain Chain LLM             | Chains parsing and processing                     | OutputParser1               | Segmentos1                 |                                    |
| busca_cliente               | LangChain Tool Workflow         | Sub-workflow: customer search                     | Delivery AI                 | Delivery AI                |                                    |
| busca_endereco              | LangChain Tool Workflow         | Sub-workflow: address retrieval                    | Delivery AI                 | Delivery AI                |                                    |
| envia_comanda               | LangChain Tool Workflow         | Sub-workflow: sends order to restaurant           | Delivery AI                 | Delivery AI                |                                    |
| salvar_cliente              | LangChain Tool Workflow         | Sub-workflow: saves customer data                  | Delivery AI                 | Delivery AI                |                                    |
| Postgres Chat Memory       | LangChain Memory Postgres Chat | Stores chat memory for context                      | Delivery AI                 | Delivery AI                |                                    |
| Calculadora                 | LangChain Tool Calculator       | Calculates delivery fees                           | busca_endereco              | Delivery AI                |                                    |
| Schedule Trigger            | Schedule Trigger                | Triggers periodic maintenance                      |                              | ListChats-Supabase         |                                    |
| File Created                | Google Drive Trigger            | Detects new files                                  |                              | Set File ID                |                                    |
| File Updated                | Google Drive Trigger            | Detects updated files                              |                              | Set File ID                |                                    |
| Set File ID                 | Set                             | Stores file ID for processing                      | File Created, File Updated  | Delete Old Doc Rows        |                                    |
| Download File               | Google Drive                   | Downloads files for processing                     | Loop Over Items             | Switch                    |                                    |
| Switch                     | Switch                         | Routes file processing by type                     | Download File               | Extract PDF/Text/Excel     |                                    |
| Extract PDF Text            | ExtractFromFile                | Extracts text from PDFs                             | Switch                     | Insert into Supabase Vectorstore |                                    |
| Extract Document Text       | ExtractFromFile                | Extracts text from documents                        | Switch                     | Insert into Supabase Vectorstore |                                    |
| Extract from Excel          | ExtractFromFile                | Extracts data from Excel files                      | Switch                     | Aggregate1                 |                                    |
| Aggregate1                 | Aggregate                      | Aggregates extracted data                           | Extract from Excel          | Summarize                  |                                    |
| Summarize                  | Summarize                     | Summarizes extracted content                        | Aggregate1                  | Insert into Supabase Vectorstore |                                    |
| Insert into Supabase Vectorstore | LangChain Vector Store Supabase | Inserts embeddings into vector DB                   | Summarize, Extract PDF/Text | Loop Over Items            |                                    |
| Embeddings OpenAI           | LangChain Embeddings OpenAI    | Generates embeddings                                | Character Text Splitter     | Supabase Vector Store      |                                    |
| Character Text Splitter     | LangChain Text Splitter Character | Splits text into chunks                             | Default Data Loader1        | Embeddings OpenAI          |                                    |
| Default Data Loader1        | LangChain Document Default Data Loader | Loads documents for processing                      | Character Text Splitter     | Insert into Supabase Vectorstore |                                    |
| Delete Old Doc Rows         | Supabase                       | Deletes old document data                            | Set File ID                 | Code                       |                                    |
| Code                       | Code                           | Custom logic for document processing                | Delete Old Doc Rows         | Loop Over Items            |                                    |
| Code2                      | Code                           | Prepares text for voice synthesis                    |                            | ElevenLabsGenerateVoice    |                                    |
| ElevenLabsGenerateVoice     | HTTP Request                  | Calls ElevenLabs API for voice generation            | Code2                       | Audio-Base64-Extract from File |                                    |
| Audio-Base64-Extract from File | ExtractFromFile              | Extracts base64 audio data                            | ElevenLabsGenerateVoice     | Evolution API3             |                                    |
| Evolution API3             | Evolution API                 | Processes audio data                                  | Audio-Base64-Extract from File |                            |                                    |
| Evolution API2             | Evolution API                 | Additional audio processing                           | Loop Over Items3            | 1,2s1                      |                                    |
| Evolution API              | Evolution API                 | Audio processing                                      | If1                        | Parser Chain               |                                    |
| OpenAI3                    | LangChain LLM Chat            | AI text generation for voice                          |                            | OutputParser1              |                                    |
| If1                        | If                             | Branches voice generation flow                        | OpenAI3                    | Evolution API, Parser Chain |                                    |
| no.op                      | No Operation                  | Placeholder node                                      |                            | Loop Over Items3           |                                    |
| Loop Over Items            | SplitInBatches                | Batch processing for document insertion               | Insert into Supabase Vectorstore | Download File             |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Purpose: Receive WhatsApp messages.  
   - Configure to accept POST requests with JSON body.  
   - Connect output to `Auth` node.

2. **Create Auth Node**  
   - Type: If  
   - Purpose: Validate API key or authentication token in webhook payload.  
   - Configure condition to check API key correctness.  
   - On success, connect to `dados_para_atendimento_humano`.

3. **Create `dados_para_atendimento_humano` Node**  
   - Type: Set  
   - Purpose: Prepare data for human attendance flag check.  
   - Set necessary variables or flags.  
   - Connect to `Switch5`.

4. **Create `Switch5` Node**  
   - Type: Switch  
   - Purpose: Route to stop automation or verify human attendance.  
   - Configure two outputs: one to `PARAR ISIS`, one to `Verifica Atendimento Humano`.

5. **Create Redis Nodes `PARAR ISIS` and `Verifica Atendimento Humano`**  
   - Type: Redis  
   - Purpose: Manage flags in Redis cache.  
   - Configure Redis credentials.  
   - Connect both outputs to `Switch Block1`.

6. **Create `Switch Block1` Node**  
   - Type: Switch  
   - Purpose: Further routing based on Redis flags.  
   - Connect output to `Variáveis`.

7. **Create `Variáveis` Node**  
   - Type: Set  
   - Purpose: Set workflow variables such as session context.  
   - Connect to `Date & Time1`.

8. **Create `Date & Time1` Node**  
   - Type: Date & Time  
   - Purpose: Add timestamp.  
   - Connect to `Supabase4`.

9. **Create `Supabase4` Node**  
   - Type: Supabase  
   - Purpose: Query session or user data.  
   - Configure Supabase credentials.  
   - Connect to `If5`.

10. **Create `If5` Node**  
    - Type: If  
    - Purpose: Check if session exists.  
    - On true, connect to `N8N Labz YT1`; on false, connect to `Gerar sessionID1`.

11. **Create `N8N Labz YT1` Node**  
    - Type: Supabase  
    - Purpose: Retrieve session info.  
    - Connect to `Code3`.

12. **Create `Gerar sessionID1` Node**  
    - Type: Crypto  
    - Purpose: Generate new session ID.  
    - Connect to `Supabase5`.

13. **Create `Supabase5` Node**  
    - Type: Supabase  
    - Purpose: Insert new session record.  
    - Connect to `Code3`.

14. **Create `Code3` Node**  
    - Type: Code  
    - Purpose: Custom session logic.  
    - Connect to next processing nodes (e.g., AI processing).

15. **Create Customer & Chat Data Nodes**  
    - `FindPhone` (Supabase): Search customer by phone.  
    - `If3` (If): Check existence.  
    - `AddChat-Supabase` & `UpdateChat-Supabase` (Supabase): Add or update chat records.  
    - `Merge1` (Merge): Merge add/update outputs.  
    - `CreateMessage-Supabase` (Supabase): Insert chat message.

16. **Create AI Conversation Nodes**  
    - `OpenAI Chat Model` series: Configure OpenAI API credentials and model parameters.  
    - `Text Classifier`: Setup classification intents.  
    - `Delivery AI`: Configure LangChain agent with tools and workflows.  
    - `OutputParser1` & `Parser Chain`: Setup output parsing and chaining.

17. **Create Sub-Workflows for Tools**  
    - `busca_cliente`, `busca_endereco`, `envia_comanda`, `salvar_cliente`: Implement as separate workflows with expected inputs/outputs.

18. **Create Delivery Fee Calculation Nodes**  
    - `Calculadora`: Configure calculator tool for fee computation.  
    - Connect with address retrieval workflow.

19. **Create Document & Knowledge Base Nodes**  
    - Google Drive Triggers: Setup OAuth2 credentials.  
    - `Download File`, `Switch`, `Extract PDF/Text/Excel`: Configure file processing.  
    - `Aggregate1`, `Summarize`, `Insert into Supabase Vectorstore`: Setup summarization and embedding insertion.  
    - `Embeddings OpenAI`, `Character Text Splitter`, `Default Data Loader1`: Configure LangChain document processing.

20. **Create Voice Generation Nodes**  
    - `Code2`: Prepare text for voice.  
    - `ElevenLabsGenerateVoice`: Configure HTTP request with ElevenLabs API key.  
    - `Audio-Base64-Extract from File`: Extract audio data.  
    - `Evolution API` nodes: Configure with credentials for voice synthesis.

21. **Create Scheduled Maintenance Nodes**  
    - `Schedule Trigger`: Setup schedule (e.g., daily).  
    - `ListChats-Supabase`, `ListMessages-Supabase`: Query old data.  
    - `Aggregate`, `Code1`, `Delete Old Doc Rows`: Cleanup logic.  
    - `Wait1`, `DisableMessage-Supabase`: Manage timing and disable processed messages.

22. **Configure Credentials**  
    - OpenAI API key for LangChain nodes.  
    - Supabase credentials for database nodes.  
    - Redis credentials for caching nodes.  
    - Google Drive OAuth2 for file triggers and downloads.  
    - ElevenLabs and Evolution API keys for voice generation.

23. **Test Workflow**  
    - Send test WhatsApp messages to webhook.  
    - Verify customer data creation and AI responses.  
    - Confirm delivery fee calculation and order forwarding.  
    - Check document ingestion and voice message generation.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is compatible only with self-hosted n8n instances due to community node usage.     | Important for deployment environment.                                                           |
| Secure credential management is critical to prevent unauthorized API access.                     | Best practice for API keys and database credentials.                                            |
| For custom development or setup assistance, contact via WhatsApp: +55 17 99155-7874              | Support contact for users.                                                                       |
| Workflow integrates with Supabase as primary database and Redis for caching and flags.           | Database and cache backend details.                                                             |
| Uses LangChain nodes extensively for AI orchestration and OpenAI for language model processing. | AI technology stack.                                                                             |
| Google Drive integration requires OAuth2 credentials for file triggers and downloads.            | File ingestion setup.                                                                            |
| Voice generation uses ElevenLabs and Evolution APIs for audio synthesis.                         | Multimedia enhancement for customer interaction.                                                |
| The workflow includes multiple sub-workflows invoked as LangChain tools for modularity.          | Modular design for customer search, address lookup, order sending, and customer saving.         |
| Video and blog resources may be available from the workflow author (not included here).          | Users may seek additional learning materials externally.                                        |

---

This document provides a comprehensive reference for understanding, reproducing, and maintaining the AI-powered WhatsApp assistant workflow for restaurant delivery automation. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with this complex automation.