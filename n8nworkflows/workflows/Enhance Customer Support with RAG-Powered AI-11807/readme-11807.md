Enhance Customer Support with RAG-Powered AI

https://n8nworkflows.xyz/workflows/enhance-customer-support-with-rag-powered-ai-11807


# Enhance Customer Support with RAG-Powered AI

---

## 1. Workflow Overview

This workflow, titled **"Enhance Customer Support with RAG-Powered AI"**, is a comprehensive multi-channel customer support automation system leveraging Retrieval-Augmented Generation (RAG) and AI models to deliver intelligent responses. It targets use cases where customer inquiries arrive through diverse channels (email, live chat, WhatsApp, Slack, Discord), requiring consistent normalization, context-aware AI response generation, sentiment and confidence evaluation, and conditional escalation to human agents when necessary.

The workflow is logically segmented into the following blocks:

- **1.1 Inbound Message Reception and Normalization:** Captures incoming messages from multiple channels and normalizes disparate payloads into a unified schema.

- **1.2 Conversation Context Management:** Maintains and enriches conversation history per user session to provide context to AI models.

- **1.3 Query Preprocessing and RAG Response Generation:** Cleans and structures user queries, then invokes a RAG-powered AI agent to generate grounded responses based on product documentation.

- **1.4 Sentiment and Confidence Analysis:** Classifies the sentiment of the message and calculates a confidence score for the AI-generated response.

- **1.5 Decision Logic for Response vs. Escalation:** Applies gating logic based on confidence and sentiment, checks for human takeover requests and scope, and routes accordingly.

- **1.6 Response Formatting and Delivery:** Formats AI responses per communication channel and delivers them, including sending emails or posting messages in Slack, Discord, WhatsApp, or webhook responses.

- **1.7 Escalation Handling:** When escalation is required, generates an escalation reason, prepares ticket data, checks business hours, creates or updates Zendesk tickets, and notifies the support team via Slack.

- **1.8 Conversation Logging and Categorization:** Automatically categorizes conversations and logs interactions to Google Sheets and Zendesk for audit and analytics.

- **1.9 Weekly Maintenance and Knowledge Base Update:** Scheduled weekly job to fetch metrics, prepare content, embed it, update the Supabase vector store, and post a summary report in Slack to continuously improve AI retrieval quality.

---

## 2. Block-by-Block Analysis

### 2.1 Inbound Message Reception and Normalization

#### Overview  
This block collects inbound customer messages from multiple channels (Email, Live Chat webhook, WhatsApp, Slack mentions, Discord webhook) and normalizes them into a consistent JSON schema for downstream processing.

#### Nodes Involved  
- Email Trigger (IMAP)  
- Webhook Trigger (Live Chat)  
- WhatsApp Trigger  
- Slack Trigger (Mentions)  
- Discord Webhook Trigger  
- Workflow Configuration (sets environment/config variables)  
- Normalize Payload

#### Node Details

- **Email Trigger (IMAP)**  
  - *Type:* Email Read IMAP Trigger  
  - *Role:* Watches an IMAP email inbox for incoming support emails.  
  - *Config:* Uses IMAP credentials; no custom filters specified.  
  - *Inputs:* External (email server)  
  - *Outputs:* Passes raw email JSON to Workflow Configuration node.  
  - *Failures:* Possible auth errors, connection timeouts, malformed emails.

- **Webhook Trigger (Live Chat)**  
  - *Type:* Webhook Trigger  
  - *Role:* Receives live chat messages via webhook POST requests.  
  - *Config:* Static webhook path set; no specific filters.  
  - *Inputs:* Incoming HTTP requests.  
  - *Outputs:* Passes webhook data to Workflow Configuration.  
  - *Failures:* Network issues, invalid payloads.

- **WhatsApp Trigger**  
  - *Type:* WhatsApp Trigger  
  - *Role:* Listens for WhatsApp messages or updates via WhatsApp API.  
  - *Config:* OAuth2 credentials for WhatsApp API; listens for account review updates.  
  - *Inputs:* WhatsApp API events.  
  - *Outputs:* Forwards normalized WhatsApp message data.  
  - *Failures:* OAuth token expiry, API rate limits.

- **Slack Trigger (Mentions)**  
  - *Type:* Slack Trigger (App Mention)  
  - *Role:* Captures Slack app mentions in a configured Slack channel.  
  - *Config:* Slack OAuth credentials; channel ID from environment.  
  - *Inputs:* Slack app_mention events.  
  - *Outputs:* Structured Slack event JSON.  
  - *Failures:* Slack API rate limits, credential expiration.

- **Discord Webhook Trigger**  
  - *Type:* Webhook Trigger  
  - *Role:* Receives Discord webhook messages.  
  - *Config:* Static webhook path.  
  - *Inputs:* Incoming HTTP requests from Discord.  
  - *Outputs:* Raw Discord payload.  
  - *Failures:* Network issues, webhook misconfiguration.

- **Workflow Configuration**  
  - *Type:* Set node  
  - *Role:* Loads environment variables and sets global workflow parameters including confidence threshold, model names, table/sheet IDs, Slack channel IDs, and support email.  
  - *Inputs:* From inbound triggers.  
  - *Outputs:* Passes configuration alongside incoming message JSON.  
  - *Notes:* Uses environment variables with fallbacks.

- **Normalize Payload**  
  - *Type:* Set node  
  - *Role:* Transforms heterogeneous inbound messages into a normalized schema with fields: `channel`, `userId`, `message`, and `timestamp`.  
  - *Config:* Uses conditional expressions to detect channel source and extract userId/message from variable field names.  
  - *Inputs:* Workflow Configuration output.  
  - *Outputs:* Normalized JSON for conversation state management.  
  - *Failures:* Missing fields or unexpected message formats may cause incomplete normalization.

---

### 2.2 Conversation Context Management

#### Overview  
Manages per-user conversation history to provide context for AI reasoning. Identifies whether a conversation is new or ongoing.

#### Nodes Involved  
- Conversation History Manager  
- New vs Existing Conversation  
- Fetch Conversation History  
- Merge History with Query

#### Node Details

- **Conversation History Manager**  
  - *Type:* Code node  
  - *Role:* Extracts or initializes conversation history and sessionId from normalized input. Marks if the conversation is new.  
  - *Config:* Checks if conversationHistory array exists; if not, initializes empty.  
  - *Inputs:* Normalized Payload output.  
  - *Outputs:* JSON with `sessionId`, `conversationHistory`, and `isNewConversation` flag.  
  - *Failures:* Invalid or missing conversationHistory data.

- **New vs Existing Conversation**  
  - *Type:* If node  
  - *Role:* Routes based on whether the conversation is new.  
  - *Config:* Checks boolean `isNewConversation` equals true.  
  - *Inputs:* Conversation History Manager.  
  - *Outputs:*  
    - If new: proceeds to Preprocess Query  
    - If existing: fetches conversation history and merges with current query.  
  - *Failures:* Misclassification could lead to improper context.

- **Fetch Conversation History**  
  - *Type:* Code node  
  - *Role:* Simulates or retrieves recent conversation history (last 10 messages), appending current message.  
  - *Config:* Maintains bounded message history for context.  
  - *Inputs:* From New vs Existing Conversation when existing.  
  - *Outputs:* Updated conversationHistory array.  
  - *Failures:* Real-world integration may require external memory store; simulation here could limit accuracy.

- **Merge History with Query**  
  - *Type:* Merge node  
  - *Role:* Combines conversation history with current query data for unified downstream processing.  
  - *Inputs:* Output from Fetch Conversation History and the original input.  
  - *Outputs:* Combined JSON including enriched conversation context.  
  - *Failures:* Data mismatches could cause merge errors.

---

### 2.3 Query Preprocessing and RAG Response Generation

#### Overview  
Prepares the user query by cleaning and extracting keywords, then uses a RAG-powered AI agent with knowledge base documents and conversation context to generate an answer.

#### Nodes Involved  
- Preprocess Query  
- OpenAI Chat Model (RAG)  
- RAG Support Agent

#### Node Details

- **Preprocess Query**  
  - *Type:* Code node  
  - *Role:* Cleans input message (trim, lowercase), extracts keywords longer than 3 characters, and prepares chat input.  
  - *Inputs:* Merged conversation history + query.  
  - *Outputs:* JSON with cleanedQuery, keywords array, and chatInput fields.  
  - *Failures:* Unexpected message formats could impair keyword extraction.

- **OpenAI Chat Model (RAG)**  
  - *Type:* Langchain OpenAI Chat model node  
  - *Role:* Configured with model from workflow config (default GPT-4o-mini), temperature 0.7 for balanced creativity.  
  - *Inputs:* Query data from Preprocess Query node.  
  - *Outputs:* AI-generated text response.  
  - *Credentials:* OpenAI API credentials required.  
  - *Failures:* API rate limits, auth errors, or model unavailability.

- **RAG Support Agent**  
  - *Type:* Langchain Agent node  
  - *Role:* Uses RAG approach combining knowledge base documents and conversation context to answer the query. System message instructs to provide professional, empathetic, concise answers or escalate if unsure.  
  - *Inputs:* OpenAI Chat Model (RAG) output and conversation history memory.  
  - *Outputs:* AI response text.  
  - *Failures:* Knowledge base retrieval failures, incomplete document embeddings.

---

### 2.4 Sentiment and Confidence Analysis

#### Overview  
Analyzes sentiment of the incoming message and calculates confidence in the AI-generated response to support gating logic.

#### Nodes Involved  
- Sentiment Analysis  
- OpenAI Chat Model (Sentiment)  
- Confidence Scoring  
- Check Confidence Threshold  
- Check Sentiment

#### Node Details

- **Sentiment Analysis**  
  - *Type:* Langchain Chain LLM  
  - *Role:* Classifies sentiment as positive, neutral, or negative based on tone, urgency, and emotional content. Uses OpenAI chat model with low temperature (0.3) for stable output.  
  - *Inputs:* AI response message.  
  - *Outputs:* Sentiment label.  
  - *Failures:* Ambiguous messages may lead to misclassification.

- **Confidence Scoring**  
  - *Type:* Code node  
  - *Role:* Calculates confidence score (0-0.95) based on response length, presence of references (URLs, docs), and specificity (keywords like step, guide).  
  - *Inputs:* AI response text and sentiment.  
  - *Outputs:* Numeric confidence score and sentiment.  
  - *Failures:* Over-reliance on simple heuristics may misjudge confidence.

- **Check Confidence Threshold**  
  - *Type:* If node  
  - *Role:* Checks if confidence is greater or equal to threshold (default 0.7 from config).  
  - *Inputs:* Confidence Scoring output.  
  - *Outputs:*  
    - True: continue to sentiment check.  
    - False: route to escalation needed.  
  - *Failures:* Threshold tuning is critical for optimal gating.

- **Check Sentiment**  
  - *Type:* If node  
  - *Role:* Checks if sentiment is not negative.  
  - *Inputs:* Confidence threshold pass branch.  
  - *Outputs:*  
    - True (not negative): route to scope check.  
    - False (negative): route to escalation needed.  
  - *Failures:* Misclassification could cause improper escalation.

---

### 2.5 Decision Logic for Response vs. Escalation

#### Overview  
Applies rules to determine if the AI can respond automatically or if human escalation is necessary, including detecting explicit human takeover requests and business hour validations.

#### Nodes Involved  
- Check Message Scope  
- Human Takeover Detection  
- Escalation Needed?  
- Business Hours Check

#### Node Details

- **Check Message Scope**  
  - *Type:* If node  
  - *Role:* Detects if user message contains phrases indicating desire for human intervention (e.g., "speak to human").  
  - *Inputs:* Check Sentiment true branch.  
  - *Outputs:*  
    - If no human takeover intent, proceed to response delivery.  
    - Otherwise, escalate.  
  - *Failures:* Phrase detection may miss variants or synonyms.

- **Human Takeover Detection**  
  - *Type:* If node  
  - *Role:* Double-checks message content for absence of "agent" keyword before routing to escalation or response.  
  - *Inputs:* Check Message Scope outputs.  
  - *Outputs:* Routes to Channel Router for auto-response or Escalation Needed.  
  - *Failures:* False negatives/positives possible.

- **Escalation Needed?**  
  - *Type:* If node  
  - *Role:* Combines conditions of low confidence, negative sentiment, or explicit human takeover request to decide escalation.  
  - *Inputs:* Multiple upstream nodes for fallback.  
  - *Outputs:*  
    - True: escalate.  
    - False: allow auto-response.  
  - *Failures:* Complex logic needs testing for edge cases.

- **Business Hours Check**  
  - *Type:* If node  
  - *Role:* Only allows ticket creation/update during business hours (9:00 to 17:00 local time).  
  - *Inputs:* Escalation flow.  
  - *Outputs:*  
    - True: create new Zendesk ticket.  
    - False: update existing ticket (or alternative handling).  
  - *Failures:* Time zone assumptions may cause errors.

---

### 2.6 Response Formatting and Delivery

#### Overview  
Formats the AI response suitably per communication channel and delivers it. Email responses are sent via SMTP; other channels end after formatting as replies are typically handled externally.

#### Nodes Involved  
- Channel Router  
- Format Email Response  
- Send Email Reply  
- Format Webhook Response  
- Format WhatsApp Response  
- Format Discord Response  
- Format Slack Response  
- End Workflow

#### Node Details

- **Channel Router**  
  - *Type:* Switch node  
  - *Role:* Routes normalized messages by channel (email, webhook, WhatsApp, Discord, Slack) to channel-specific formatting nodes.  
  - *Inputs:* Human Takeover Detection true branch or auto-response path.  
  - *Outputs:* To formatting nodes.  
  - *Failures:* Unknown channels route to no output.

- **Format Email Response**  
  - *Type:* Set node  
  - *Role:* Prepares email subject and body from AI response for sending.  
  - *Inputs:* Channel Router Email branch.  
  - *Outputs:* Passes formatted email to Send Email Reply node.  
  - *Failures:* Missing email addresses cause send failures.

- **Send Email Reply**  
  - *Type:* Email Send node  
  - *Role:* Sends email reply to user using SMTP credentials.  
  - *Inputs:* Formatted email data.  
  - *Outputs:* Triggers Auto-Categorization.  
  - *Failures:* SMTP auth errors, network issues.

- **Format Webhook Response**  
  - *Type:* Set node  
  - *Role:* Prepares JSON response with AI answer and status for webhook replies.  
  - *Inputs:* Channel Router Webhook branch.  
  - *Outputs:* Ends workflow after sending response.  
  - *Failures:* None specific; webhook client must handle response.

- **Format WhatsApp Response**  
  - *Type:* Set node  
  - *Role:* Formats WhatsApp reply message and target user ID.  
  - *Inputs:* Channel Router WhatsApp branch.  
  - *Outputs:* Ends workflow; actual WhatsApp sending handled externally or by other means.  
  - *Failures:* Missing user ID or message.

- **Format Discord Response**  
  - *Type:* Set node  
  - *Role:* Prepares Discord message content and channel ID.  
  - *Inputs:* Channel Router Discord branch.  
  - *Outputs:* Ends workflow.  
  - *Failures:* Missing channel ID or content.

- **Format Slack Response**  
  - *Type:* Set node  
  - *Role:* Prepares Slack message text, channel, and thread timestamp for reply.  
  - *Inputs:* Channel Router Slack branch.  
  - *Outputs:* Ends workflow.  
  - *Failures:* Missing event data.

- **End Workflow**  
  - *Type:* NoOp node  
  - *Role:* Termination point for response flows.  
  - *Inputs:* Formatting nodes.  
  - *Outputs:* None.  
  - *Failures:* None.

---

### 2.7 Escalation Handling

#### Overview  
Handles cases requiring human intervention by generating an escalation justification, preparing ticket data, checking business hours, creating or updating Zendesk tickets, and notifying support via Slack.

#### Nodes Involved  
- Escalation Reasoning  
- Prepare Ticket Object  
- Business Hours Check  
- Create Zendesk Ticket  
- Update Zendesk Ticket  
- Escalation Alert to Support Team

#### Node Details

- **Escalation Reasoning**  
  - *Type:* Langchain Chain LLM  
  - *Role:* Generates concise explanation of why escalation is needed, based on message, AI response, confidence, and sentiment.  
  - *Inputs:* Escalation Needed? true branch data.  
  - *Outputs:* Escalation reason text for ticket description.  
  - *Failures:* Ambiguous input may produce vague reasons.

- **Prepare Ticket Object**  
  - *Type:* Set node  
  - *Role:* Constructs Zendesk ticket subject, description, and priority (high for negative sentiment, normal otherwise).  
  - *Inputs:* Escalation Reasoning output.  
  - *Outputs:* Ticket payload for Zendesk API.  
  - *Failures:* Truncation or malformed data may cause API errors.

- **Business Hours Check**  
  - *Type:* If node (as described earlier)  
  - *Role:* Routes to ticket creation or update based on current time.  
  - *Inputs:* Prepare Ticket Object output.  
  - *Outputs:* Create or update ticket nodes.

- **Create Zendesk Ticket**  
  - *Type:* Zendesk node  
  - *Role:* Creates a new ticket with provided description and subject.  
  - *Inputs:* Business Hours Check true branch.  
  - *Outputs:* Alerts support team after ticket creation.  
  - *Credentials:* Zendesk API.  
  - *Failures:* API quota, auth errors.

- **Update Zendesk Ticket**  
  - *Type:* Zendesk node  
  - *Role:* Updates an existing ticket (by ticketId) with internal note describing escalation.  
  - *Inputs:* Business Hours Check false branch.  
  - *Outputs:* Alerts support team.  
  - *Failures:* Missing ticket ID, API errors.

- **Escalation Alert to Support Team**  
  - *Type:* Slack node  
  - *Role:* Sends alert message to configured Slack support channel with escalation details including user, priority, reason, and message.  
  - *Inputs:* From ticket creation or update.  
  - *Outputs:* None.  
  - *Credentials:* Slack API.  
  - *Failures:* Slack API rate limits or credential issues.

---

### 2.8 Conversation Logging and Categorization

#### Overview  
Automatically categorizes conversations by topic and logs conversation metadata and AI responses to Google Sheets and Zendesk for auditing and analytics.

#### Nodes Involved  
- Auto-Categorization  
- Log Conversation to Sheets  
- Log Conversation to Zendesk

#### Node Details

- **Auto-Categorization**  
  - *Type:* Code node  
  - *Role:* Assigns category tags (billing, technical, account, integration, general) based on keyword matching in the message text.  
  - *Inputs:* After Send Email Reply or other final response nodes.  
  - *Outputs:* Adds `category` field to JSON.  
  - *Failures:* Simple keyword matching can miscategorize.

- **Log Conversation to Sheets**  
  - *Type:* Google Sheets node  
  - *Role:* Appends conversation logs to a configured Google Sheets document with columns for userId, channel, message, category, sentiment, timestamp, AI response, and confidence.  
  - *Inputs:* Auto-Categorization output.  
  - *Credentials:* Google Sheets service account.  
  - *Failures:* Sheet access issues, quota limits.

- **Log Conversation to Zendesk**  
  - *Type:* Zendesk node  
  - *Role:* Creates a log entry ticket (or comment) in Zendesk with conversation summary as subject and description.  
  - *Inputs:* Log Conversation to Sheets output.  
  - *Credentials:* Zendesk API.  
  - *Failures:* API errors, rate limits.

---

### 2.9 Weekly Maintenance and Knowledge Base Update

#### Overview  
A scheduled weekly job fetches support metrics from Google Sheets, prepares a summary document, splits and embeds the text, inserts it into the Supabase vector store, and posts a summary in Slack to improve AI model retrieval over time.

#### Nodes Involved  
- Weekly Maintenance Schedule  
- Fetch Weekly Metrics  
- Prepare Metrics for Vector Insert  
- Document Loader  
- Text Splitter  
- OpenAI Embeddings (Insert)  
- Supabase Vector Insert  
- Send Weekly Summary

#### Node Details

- **Weekly Maintenance Schedule**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers the weekly maintenance pipeline on a recurring interval (daily or weekly as configured).  
  - *Inputs:* None.  
  - *Outputs:* Starts metrics fetch.  
  - *Failures:* Scheduler misconfiguration.

- **Fetch Weekly Metrics**  
  - *Type:* Google Sheets node  
  - *Role:* Reads support metrics from specified Google Sheets document and sheet.  
  - *Inputs:* Schedule trigger.  
  - *Outputs:* Metrics JSON for embedding.  
  - *Failures:* Sheet access issues.

- **Prepare Metrics for Vector Insert**  
  - *Type:* Set node  
  - *Role:* Formats metrics data into a textual summary and metadata object for the vector store.  
  - *Inputs:* Fetch Weekly Metrics output.  
  - *Outputs:* Prepares document text and metadata for embedding.  
  - *Failures:* Stringify errors.

- **Document Loader**  
  - *Type:* Langchain Document Loader  
  - *Role:* Loads the prepared metrics summary as a document for processing.  
  - *Inputs:* Prepare Metrics for Vector Insert output.  
  - *Outputs:* Document object.  
  - *Failures:* Document format issues.

- **Text Splitter**  
  - *Type:* Langchain Text Splitter  
  - *Role:* Splits the document text into manageable chunks for embedding.  
  - *Inputs:* Document Loader output.  
  - *Outputs:* Split document chunks.  
  - *Failures:* Improper chunking may affect embeddings.

- **OpenAI Embeddings (Insert)**  
  - *Type:* Langchain Embeddings node  
  - *Role:* Generates vector embeddings for document chunks using configured embedding model (default text-embedding-3-small).  
  - *Inputs:* Text Splitter output.  
  - *Outputs:* Embeddings data.  
  - *Credentials:* OpenAI API.  
  - *Failures:* API errors.

- **Supabase Vector Insert**  
  - *Type:* Langchain Vector Store Supabase  
  - *Role:* Inserts embeddings into Supabase vector database for improved RAG retrieval.  
  - *Inputs:* Embeddings data and Document Loader output.  
  - *Credentials:* Supabase API.  
  - *Failures:* Database connection or auth errors.

- **Send Weekly Summary**  
  - *Type:* Slack node  
  - *Role:* Posts a summary message in configured Slack channel with metrics like total conversations, average confidence, and escalation count.  
  - *Inputs:* Supabase Vector Insert output.  
  - *Credentials:* Slack API.  
  - *Failures:* Slack API limits.

---

## 3. Summary Table

| Node Name                   | Node Type                                     | Functional Role                              | Input Node(s)                   | Output Node(s)                             | Sticky Note                                                                                                                                                                                                                                             |
|-----------------------------|-----------------------------------------------|----------------------------------------------|---------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Configuration       | Set                                           | Global configuration and env var setup       | Email Trigger (IMAP), Webhook Trigger, WhatsApp Trigger, Slack Trigger, Discord Trigger | Normalize Payload                         | Defines system-wide settings: confidence threshold, model choices, vector store table, logging destinations, and Slack/support routing identifiers.                                                                                                     |
| Email Trigger (IMAP)         | Email Read IMAP Trigger                        | Inbound email support messages                | External email server            | Workflow Configuration                      | Starts the workflow from multiple sources (email, live chat webhook, WhatsApp, Slack mention, Discord webhook).                                                                                                                                          |
| Webhook Trigger (Live Chat)  | Webhook Trigger                               | Inbound live chat webhook messages            | External webhook calls           | Workflow Configuration                      |                                                                                                                                                                                                                                                         |
| WhatsApp Trigger             | WhatsApp Trigger                              | Inbound WhatsApp messages                      | WhatsApp API                    | Workflow Configuration                      |                                                                                                                                                                                                                                                         |
| Slack Trigger (Mentions)     | Slack Trigger                                 | Inbound Slack app mention events               | Slack API                      | Workflow Configuration                      |                                                                                                                                                                                                                                                         |
| Discord Webhook Trigger      | Webhook Trigger                               | Inbound Discord webhook messages               | External webhook calls           | Workflow Configuration                      |                                                                                                                                                                                                                                                         |
| Normalize Payload            | Set                                           | Normalizes all inbound messages into unified schema | Workflow Configuration          | Conversation History Manager                 | Normalizes payloads to a common schema (channel, userId, message, timestamp) and determines whether the message belongs to a new or existing conversation.                                                                                               |
| Conversation History Manager | Code                                          | Extracts or initializes conversation history  | Normalize Payload               | New vs Existing Conversation                 |                                                                                                                                                                                                                                                         |
| New vs Existing Conversation | If                                             | Routes based on new or existing conversation  | Conversation History Manager    | Preprocess Query, Fetch Conversation History |                                                                                                                                                                                                                                                         |
| Fetch Conversation History   | Code                                          | Retrieves last 10 messages for context        | New vs Existing Conversation    | Merge History with Query                      | Fetches recent conversation history for the user and merges it with the current query to provide context for downstream reasoning.                                                                                                                     |
| Merge History with Query     | Merge                                          | Combines conversation history with current query | Fetch Conversation History + Normalize Payload | Preprocess Query                              |                                                                                                                                                                                                                                                         |
| Preprocess Query             | Code                                          | Cleans and extracts keywords from user message | Merge History with Query        | OpenAI Chat Model (RAG)                       | Cleans and structures the customer message (cleanedQuery, keywords, chatInput) to improve retrieval and response quality.                                                                                                                              |
| OpenAI Chat Model (RAG)      | Langchain OpenAI Chat Model                    | AI generation of grounded response with RAG  | Preprocess Query                | RAG Support Agent                            | Generates a grounded support answer using the knowledge base + conversation context (RAG). Uses the configured OpenAI chat model.                                                                                                                      |
| RAG Support Agent            | Langchain Agent                                | RAG-powered AI response agent                  | OpenAI Chat Model (RAG), Chat Memory | Sentiment Analysis                            |                                                                                                                                                                                                                                                         |
| Chat Memory                 | Langchain Memory Buffer                         | Maintains conversation memory for context     | RAG Support Agent              | -                                            |                                                                                                                                                                                                                                                         |
| Sentiment Analysis           | Langchain Chain LLM                            | Classifies message sentiment                    | RAG Support Agent              | Confidence Scoring                           | Classifies sentiment and computes a confidence score for the AI response. Gates the flow based on confidence threshold and sentiment polarity.                                                                                                       |
| OpenAI Chat Model (Sentiment)| Langchain OpenAI Chat Model                    | AI model for sentiment classification          | Sentiment Analysis             | Confidence Scoring                           |                                                                                                                                                                                                                                                         |
| Confidence Scoring           | Code                                          | Calculates confidence score based on response | Sentiment Analysis             | Check Confidence Threshold                    |                                                                                                                                                                                                                                                         |
| Check Confidence Threshold   | If                                             | Gating based on confidence threshold           | Confidence Scoring             | Check Sentiment, Escalation Needed?          |                                                                                                                                                                                                                                                         |
| Check Sentiment             | If                                             | Checks if sentiment is not negative             | Check Confidence Threshold     | Check Message Scope, Escalation Needed?       |                                                                                                                                                                                                                                                         |
| Check Message Scope          | If                                             | Detects if user requests human takeover         | Check Sentiment               | Human Takeover Detection, Escalation Needed? | Checks if the request is within scope and detects explicit human takeover intent. Routes either to response delivery or escalation.                                                                                                                  |
| Human Takeover Detection     | If                                             | Guards against explicit human takeover phrase  | Check Message Scope            | Channel Router, Escalation Needed?            |                                                                                                                                                                                                                                                         |
| Escalation Needed?           | If                                             | Decides if escalation to human support is needed | Check Confidence Threshold, Check Sentiment, Check Message Scope, Human Takeover Detection | Escalation Reasoning, Escalation Needed false branch | Explains why escalation is required, prepares a ticket payload, checks business hours, creates/updates a Zendesk ticket, alerts support team in Slack.                                                                                                |
| Escalation Reasoning         | Langchain Chain LLM                            | AI generates reason for escalation              | Escalation Needed?             | Prepare Ticket Object                        |                                                                                                                                                                                                                                                         |
| Prepare Ticket Object        | Set                                           | Builds Zendesk ticket subject, description, priority | Escalation Reasoning          | Business Hours Check                         |                                                                                                                                                                                                                                                         |
| Business Hours Check         | If                                             | Routes escalation actions based on business hours | Prepare Ticket Object          | Create Zendesk Ticket, Update Zendesk Ticket |                                                                                                                                                                                                                                                         |
| Create Zendesk Ticket        | Zendesk                                        | Creates new support ticket                       | Business Hours Check (true)    | Escalation Alert to Support Team              |                                                                                                                                                                                                                                                         |
| Update Zendesk Ticket        | Zendesk                                        | Updates existing support ticket                   | Business Hours Check (false)   | Escalation Alert to Support Team              |                                                                                                                                                                                                                                                         |
| Escalation Alert to Support Team | Slack                                      | Notifies support team in Slack channel          | Create Zendesk Ticket, Update Zendesk Ticket | -                                            |                                                                                                                                                                                                                                                         |
| Channel Router              | Switch                                         | Routes messages to channel-specific response formatting | Human Takeover Detection       | Format Email Response, Format Webhook Response, Format WhatsApp Response, Format Discord Response, Format Slack Response |                                                                                                                                                                                                                                                         |
| Format Email Response        | Set                                           | Prepares email subject and body                   | Channel Router (Email)         | Send Email Reply                            | Routes the AI response back to the original channel and applies channel-specific formatting. Email replies are sent; other channels end after formatting.                                                                                             |
| Send Email Reply             | Email Send                                     | Sends email reply to user                          | Format Email Response          | Auto-Categorization                          |                                                                                                                                                                                                                                                         |
| Format Webhook Response      | Set                                           | Formats JSON response for webhook replies          | Channel Router (Webhook)       | End Workflow                                |                                                                                                                                                                                                                                                         |
| Format WhatsApp Response     | Set                                           | Formats WhatsApp reply message                      | Channel Router (WhatsApp)      | End Workflow                                |                                                                                                                                                                                                                                                         |
| Format Discord Response      | Set                                           | Formats Discord reply message                       | Channel Router (Discord)       | End Workflow                                |                                                                                                                                                                                                                                                         |
| Format Slack Response        | Set                                           | Prepares Slack reply message                        | Channel Router (Slack)         | End Workflow                                |                                                                                                                                                                                                                                                         |
| End Workflow                | NoOp                                           | Terminates response flows                           | Format Webhook/WhatsApp/Discord/Slack Response | -                                            |                                                                                                                                                                                                                                                         |
| Auto-Categorization          | Code                                          | Assigns conversation category based on keywords    | Send Email Reply               | Log Conversation to Sheets                   | Automatically categorizes conversations and logs data to Sheets and Zendesk for audit and analytics.                                                                                                                                                    |
| Log Conversation to Sheets   | Google Sheets                                  | Appends conversation logs into Google Sheets        | Auto-Categorization            | Log Conversation to Zendesk                   |                                                                                                                                                                                                                                                         |
| Log Conversation to Zendesk  | Zendesk                                        | Logs conversation summary in Zendesk                | Log Conversation to Sheets     | -                                            |                                                                                                                                                                                                                                                         |
| Weekly Maintenance Schedule  | Schedule Trigger                              | Triggers weekly maintenance and vector updates      | None                          | Fetch Weekly Metrics                         | Runs weekly: fetches metrics from Sheets, prepares content, splits + embeds documents, inserts into Supabase vector store, and posts a Slack weekly summary.                                                                                         |
| Fetch Weekly Metrics         | Google Sheets                                  | Reads metrics data from Google Sheets                 | Weekly Maintenance Schedule    | Supabase Vector Insert, Prepare Metrics for Vector Insert |                                                                                                                                                                                                                                                         |
| Prepare Metrics for Vector Insert | Set                                     | Formats metrics summary and metadata for embedding   | Fetch Weekly Metrics           | Supabase Vector Insert                       |                                                                                                                                                                                                                                                         |
| Document Loader             | Langchain Document Loader                       | Loads text documents for embedding                    | Prepare Metrics for Vector Insert | Supabase Vector Insert                       |                                                                                                                                                                                                                                                         |
| Text Splitter               | Langchain Text Splitter                         | Splits documents into chunks for embeddings          | Document Loader               | OpenAI Embeddings (Insert)                   |                                                                                                                                                                                                                                                         |
| OpenAI Embeddings (Insert)  | Langchain Embeddings                            | Generates vector embeddings from text chunks          | Text Splitter                 | Supabase Vector Insert                       |                                                                                                                                                                                                                                                         |
| Supabase Vector Insert      | Langchain Vector Store Supabase                 | Inserts vector embeddings into Supabase vector database | OpenAI Embeddings, Document Loader, Fetch Weekly Metrics | Send Weekly Summary                          |                                                                                                                                                                                                                                                         |
| Send Weekly Summary         | Slack                                          | Posts weekly summary message to Slack channel          | Supabase Vector Insert         | -                                            |                                                                                                                                                                                                                                                         |
| Sticky Note (multiple)      | Sticky Note                                    | Documentation and explanations for blocks             | None                          | None                                         | See individual sticky notes content throughout the workflow.                                                                                                                                                                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Inbound Triggers for Multiple Channels:**  
   - Add Email Read IMAP node configured with IMAP credentials to monitor support inbox.  
   - Add Webhook Trigger node with a unique path for live chat messages.  
   - Add WhatsApp Trigger node configured with WhatsApp OAuth2 credentials.  
   - Add Slack Trigger node set to "app_mention" events in the support Slack channel, using Slack OAuth.  
   - Add Discord Webhook Trigger node with a static webhook path.

2. **Add a Set Node “Workflow Configuration”:**  
   - Define environment variables or static values for:  
     - `confidenceThreshold` (default 0.7)  
     - `supabaseTable`, `slackChannelId`, `slackSupportChannelId`, `slackWeeklySummaryChannelId`  
     - `supportEmail`  
     - `openaiModel` (default "gpt-4o-mini")  
     - `embeddingModel` (default "text-embedding-3-small")  
     - `googleSheetsDocId`, `googleSheetsLogSheet`, `googleSheetsMetricsSheet`  
   - Connect all inbound triggers to this node.

3. **Normalize Payload Node:**  
   - Add a Set node to standardize incoming messages into fields: `channel`, `userId`, `message`, `timestamp`.  
   - Use expressions to detect message origin and extract relevant fields.

4. **Conversation History Management:**  
   - Add a Code node “Conversation History Manager” to initialize or extract conversation history and sessionId.  
   - Add an If node “New vs Existing Conversation” checking if conversation is new.  
   - For new conversations, proceed to query preprocessing.  
   - For existing, add a Code node “Fetch Conversation History” simulating retrieval of last 10 messages and merge with current query using a Merge node.

5. **Query Preprocessing:**  
   - Add a Code node “Preprocess Query” to clean and extract keywords from the message.  
   - Connect this to an OpenAI Chat Model node configured with the workflow’s OpenAI model (temperature 0.7).

6. **RAG Support Agent:**  
   - Add a Langchain Agent node “RAG Support Agent” configured with a system prompt to answer using knowledge base or escalate.  
   - Connect output from OpenAI Chat Model (RAG) and link conversation history via Chat Memory node.

7. **Sentiment and Confidence Analysis:**  
   - Add a Langchain Chain LLM node “Sentiment Analysis” using the same OpenAI model at low temperature (0.3).  
   - Follow with a Code node “Confidence Scoring” that calculates confidence based on response content and sentiment.  
   - Add an If node “Check Confidence Threshold” to gate based on configured threshold.  
   - Add an If node “Check Sentiment” to route if sentiment is not negative.

8. **Scope & Human Takeover Detection:**  
   - Add If nodes “Check Message Scope” and “Human Takeover Detection” to detect explicit human intervention requests or keywords.  
   - Route accordingly between escalation or auto-response.

9. **Escalation Handling:**  
   - Add an If node “Escalation Needed?” to combine conditions (low confidence, negative sentiment, human takeover).  
   - If true, add Langchain Chain LLM “Escalation Reasoning” to generate escalation justification.  
   - Add Set node “Prepare Ticket Object” to build ticket payload including subject, description, and priority.  
   - Add If node “Business Hours Check” to route ticket creation or update.  
   - Add Zendesk nodes “Create Zendesk Ticket” and “Update Zendesk Ticket” with Zendesk API credentials.  
   - Add Slack node “Escalation Alert to Support Team” to notify support channel.

10. **Response Formatting and Delivery:**  
    - Add a Switch node “Channel Router” to route messages based on normalized `channel` field.  
    - Add Set nodes to format replies for each channel: Email, Webhook, WhatsApp, Discord, Slack.  
    - Connect Email formatting to Email Send node configured with SMTP credentials for sending.  
    - Other channels terminate workflow after formatting.  
    - Add a NoOp node “End Workflow” to mark flow termination points.

11. **Conversation Logging and Categorization:**  
    - Add a Code node “Auto-Categorization” to assign categories by keyword matching.  
    - Add Google Sheets node “Log Conversation to Sheets” appending logs to configured sheet using service account credentials.  
    - Add Zendesk node “Log Conversation to Zendesk” for audit logging.

12. **Weekly Maintenance & Vector Updates:**  
    - Add Schedule Trigger node “Weekly Maintenance Schedule” with weekly/daily interval.  
    - Add Google Sheets node “Fetch Weekly Metrics” to read metrics from Sheets.  
    - Add Set node “Prepare Metrics for Vector Insert” to format metrics summary and metadata.  
    - Add Langchain Document Loader node “Document Loader” to load summary text.  
    - Add Langchain Text Splitter node “Text Splitter” to chunk document.  
    - Add Langchain Embeddings node “OpenAI Embeddings (Insert)” to create embeddings with embedding model.  
    - Add Langchain Vector Store node “Supabase Vector Insert” to insert embeddings into Supabase.  
    - Add Slack node “Send Weekly Summary” to post summary in Slack channel.

13. **Credentials Setup:**  
    - Configure credentials for:  
      - IMAP and SMTP email accounts  
      - WhatsApp OAuth2 account  
      - Slack API OAuth token  
      - Zendesk API  
      - Google Sheets service account  
      - OpenAI API  
      - Supabase API

14. **Testing and Validation:**  
    - Test inbound messages on each channel for normalization correctness.  
    - Validate AI responses for both auto-response and escalation flows.  
    - Confirm ticket creation and Slack alerts work as expected.  
    - Verify logging to Google Sheets and Zendesk.  
    - Schedule and validate weekly maintenance flow.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                               | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow is a multi-channel customer support AI system powered by Retrieval-Augmented Generation (RAG), ingesting inbound messages from multiple sources and normalizing them into a single schema for unified processing and context-aware AI responses.                             | Sticky Note: How it works                                                                                   |
| Setup requires environment variables for Slack channel IDs, Google Sheets document IDs, Supabase table, OpenAI and embedding model names, and support email addresses. Credential connections must be established for all external services used.                                         | Sticky Note: Setup steps                                                                                    |
| The workflow maintains conversation continuity per user by tracking recent message history (last 10) to provide richer AI context.                                                                                                                                                        | Sticky Note: Context Enrichment (History)                                                                  |
| Sentiment classification and confidence scoring gate response delivery, with escalation to human support if confidence is low or sentiment negative.                                                                                                                                     | Sticky Note: Quality & Risk Gating                                                                           |
| Escalation reasons are generated by AI to provide concise, specific human-readable explanations for support tickets.                                                                                                                                                                      | Sticky Note: Escalation to Human Support                                                                    |
| Weekly maintenance updates the knowledge base embedding in Supabase vector store to improve AI retrieval over time, posting summaries to Slack for team visibility.                                                                                                                     | Sticky Note: Weekly Maintenance & Vector Updates                                                            |
| For more about RAG and Langchain integration in n8n, see: https://tuguidragos.com/blog/rag-support-agent-n8n/                                                                                                                                                                            | External blog link                                                                                           |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It fully complies with applicable content policies and contains no illegal, offensive, or protected materials. All processed data is legal and public.

---