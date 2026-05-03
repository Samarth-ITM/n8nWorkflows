Automated Customer Support System with Gmail, GPT-4 & Vector Knowledge Base

https://n8nworkflows.xyz/workflows/automated-customer-support-system-with-gmail--gpt-4---vector-knowledge-base-8196


# Automated Customer Support System with Gmail, GPT-4 & Vector Knowledge Base

### 1. Workflow Overview

This workflow implements an **Automated Customer Support System** integrating Gmail, GPT-4 powered AI agents via LangChain, and a vector-based knowledge base using Qdrant. It is designed to automate the processing of incoming customer emails, analyze their content using advanced AI models, generate appropriate replies, escalate issues requiring human intervention, and log all interactions for tracking and further analysis. Additionally, it supports knowledge base updates from form submissions.

The workflow is organized into the following logical blocks:

- **1.1 Email Reception & Data Extraction:** Trigger on new Gmail messages, filter new customer emails, and extract relevant data.
- **1.2 Email Thread Processing:** Retrieve full email thread history for context aggregation.
- **1.3 AI Analysis & Response Generation:** Use GPT-4 based LangChain agents and vector search to analyze inquiries and generate automated replies.
- **1.4 Escalation Decision & Labeling:** Determine if escalation to human support is necessary, label Gmail threads accordingly.
- **1.5 Automated Reply & Logging:** Send AI-generated replies to customers and log tickets into Google Sheets.
- **1.6 Status Updates:** Notify via Telegram about ticket processing status.
- **1.7 Knowledge Base Management:** Handle form submissions to update or create vector embeddings and collections in Qdrant.

---

### 2. Block-by-Block Analysis

#### 1.1 Email Reception & Data Extraction

- **Overview:**  
Receives new emails via a Gmail trigger, extracts key email data, and filters to process only new customer emails.

- **Nodes Involved:**  
  - Gmail Trigger1  
  - Extract Email Data  
  - Filter New Customer Emails  

- **Node Details:**

  - **Gmail Trigger1**  
    - *Type:* Gmail Trigger  
    - *Role:* Initiates workflow upon new incoming email in Gmail account.  
    - *Config:* Default Gmail trigger settings, likely monitoring inbox or specific label.  
    - *Connections:* Outputs to Extract Email Data.  
    - *Edge cases:* Authentication failures, Gmail API quota limits, trigger firing delays.

  - **Extract Email Data**  
    - *Type:* Set Node  
    - *Role:* Extracts and formats relevant email fields (sender, subject, body, date) for further processing.  
    - *Config:* Sets variables with expressions referencing Gmail trigger data (e.g., `{{$json["from"]}}`, `{{$json["subject"]}}`).  
    - *Connections:* Main output to Filter New Customer Emails.  
    - *Edge cases:* Missing fields in email, malformed data.

  - **Filter New Customer Emails**  
    - *Type:* If Node  
    - *Role:* Determines if the email is from a new customer or existing thread to prevent duplicate processing.  
    - *Config:* Conditional expressions checking email metadata or labels.  
    - *Connections:*  
      - True branch: proceeds to Edit Fields1.  
      - False branch: proceeds to Get Thread History (to process existing conversations).  
    - *Edge cases:* Incorrect filtering logic causing missed or duplicate processing.

---

#### 1.2 Email Thread Processing

- **Overview:**  
For emails identified as belonging to existing threads, retrieves full thread history and aggregates context for AI analysis.

- **Nodes Involved:**  
  - Get Thread History  
  - Process Thread History  
  - Aggregate  

- **Node Details:**

  - **Get Thread History**  
    - *Type:* Gmail Node  
    - *Role:* Fetches all messages within the email thread for context building.  
    - *Config:* Uses thread ID from incoming email to fetch full conversation.  
    - *Connections:* Outputs to Process Thread History.  
    - *Edge cases:* API rate limits, incomplete thread data.

  - **Process Thread History**  
    - *Type:* Code Node  
    - *Role:* Parses and processes thread messages into a structured format suitable for AI input (e.g., concatenates messages, removes duplicates).  
    - *Config:* Custom JavaScript code handling message extraction and formatting.  
    - *Connections:* Outputs to Aggregate.  
    - *Edge cases:* Script errors, unexpected message formats.

  - **Aggregate**  
    - *Type:* Aggregate Node  
    - *Role:* Aggregates processed messages, possibly grouping or summarizing for AI input.  
    - *Config:* Aggregation method (e.g., concatenation, grouping by sender).  
    - *Connections:* Outputs to Edit Fields1.  
    - *Edge cases:* Large threads causing performance issues.

---

#### 1.3 AI Analysis & Response Generation

- **Overview:**  
Uses GPT-4 powered LangChain agents combined with vector search on a Qdrant knowledge base to analyze the email content and generate responses.

- **Nodes Involved:**  
  - Edit Fields1  
  - AI Analysis Agent  
  - OpenAI Chat Model1  
  - Qdrant Vector Store  
  - Embeddings Mistral Cloud  
  - Process AI Analysis  
  - Edit Fields  
  - Needs Escalation?  

- **Node Details:**

  - **Edit Fields1**  
    - *Type:* Set Node  
    - *Role:* Prepares and formats input data for AI agent, e.g., combines email text with context.  
    - *Config:* Sets variables for AI prompt input.  
    - *Connections:* Outputs to AI Analysis Agent.  
    - *Edge cases:* Missing or malformed input data.

  - **AI Analysis Agent**  
    - *Type:* LangChain Agent Node  
    - *Role:* Coordinates AI processing, invoking the chat model and vector store tools for analysis.  
    - *Config:* Integrates OpenAI chat and Qdrant vector store tools, configured for GPT-4.  
    - *Connections:*  
      - ai_languageModel input from OpenAI Chat Model1  
      - ai_tool input from Qdrant Vector Store  
      - Outputs to Process AI Analysis.  
    - *Edge cases:* AI service downtime, API limits, model errors.

  - **OpenAI Chat Model1**  
    - *Type:* LangChain OpenAI Chat Model Node  
    - *Role:* Provides GPT-4 based natural language generation and understanding.  
    - *Config:* Model set to GPT-4, with parameters like temperature, max tokens possibly tuned.  
    - *Connections:* Linked as language model to AI Analysis Agent.  
    - *Edge cases:* Authentication errors, token limits.

  - **Qdrant Vector Store**  
    - *Type:* LangChain Vector Store Node  
    - *Role:* Provides relevant knowledge base documents through vector similarity search.  
    - *Config:* Connected to Qdrant instance, embedding model is Mistral Cloud.  
    - *Connections:*  
      - ai_embedding input from Embeddings Mistral Cloud  
      - ai_tool input to AI Analysis Agent.  
    - *Edge cases:* Connection failures, empty vector collections.

  - **Embeddings Mistral Cloud**  
    - *Type:* LangChain Embeddings Node  
    - *Role:* Generates vector embeddings for input text to query Qdrant.  
    - *Config:* Uses Mistral Cloud embeddings API.  
    - *Connections:* Outputs to Qdrant Vector Store's embedding input.  
    - *Edge cases:* API failures, rate limits.

  - **Process AI Analysis**  
    - *Type:* Set Node  
    - *Role:* Processes AI agent output to extract and format analysis results.  
    - *Config:* Sets fields such as response text, escalation flags.  
    - *Connections:* Outputs to Edit Fields.  
    - *Edge cases:* Missing or malformed AI output.

  - **Edit Fields**  
    - *Type:* Set Node  
    - *Role:* Finalizes data formatting before escalation decision.  
    - *Config:* Prepares variables required by next decision node.  
    - *Connections:* Outputs to Needs Escalation? node.

  - **Needs Escalation?**  
    - *Type:* If Node  
    - *Role:* Determines if the AI analysis indicates escalation is necessary.  
    - *Config:* Boolean condition checking AI output flags or confidence scores.  
    - *Connections:*  
      - True branch to Needs Human Review Label.  
      - False branch to Send Automated Reply and Auto Resolved Label.  
    - *Edge cases:* Incorrect escalation decisions may cause workflow errors.

---

#### 1.4 Escalation Decision & Labeling

- **Overview:**  
Assigns Gmail labels based on escalation decision to either flag for human review or mark as auto-resolved.

- **Nodes Involved:**  
  - Needs Human Review Label  
  - Auto Resolved Label  

- **Node Details:**

  - **Needs Human Review Label**  
    - *Type:* Gmail Node  
    - *Role:* Adds a Gmail label to flag the email/thread for human support intervention.  
    - *Config:* Sets label "Needs Human Review" on the thread.  
    - *Connections:* Outputs to Log Ticket to Google Sheets.  
    - *Edge cases:* Label not found or permission issues.

  - **Auto Resolved Label**  
    - *Type:* Gmail Node  
    - *Role:* Adds Gmail label indicating the issue was resolved automatically.  
    - *Config:* Sets label "Auto Resolved".  
    - *Connections:* Outputs to Log Ticket to Google Sheets.  
    - *Edge cases:* Same as above.

---

#### 1.5 Automated Reply & Logging

- **Overview:**  
Sends AI-generated email replies to customers and logs all ticket data to Google Sheets for tracking. Sends status updates via Telegram.

- **Nodes Involved:**  
  - Send Automated Reply  
  - Log Ticket to Google Sheets  
  - Send Status Update  

- **Node Details:**

  - **Send Automated Reply**  
    - *Type:* Gmail Node  
    - *Role:* Sends the AI-generated reply email back to the customer.  
    - *Config:* Uses processed reply content, sets recipient and reply-to fields appropriately.  
    - *Connections:* Outputs to Log Ticket to Google Sheets.  
    - *Edge cases:* Sending failures, quota limits.

  - **Log Ticket to Google Sheets**  
    - *Type:* Google Sheets Node  
    - *Role:* Logs ticket details including email metadata, AI analysis results, and status labels.  
    - *Config:* Spreadsheet and worksheet configured for support ticket logging.  
    - *Connections:* Outputs to Send Status Update.  
    - *Edge cases:* Sheet permission issues, API limits.

  - **Send Status Update**  
    - *Type:* Telegram Node  
    - *Role:* Sends a Telegram message notifying about the ticket processing status.  
    - *Config:* Uses Telegram Bot credentials, sends to predefined chat/channel.  
    - *Connections:* Terminal node of the workflow branch.  
    - *Edge cases:* Telegram API downtime or invalid chat IDs.

---

#### 1.6 Knowledge Base Management

- **Overview:**  
Handles form submissions that update the knowledge base, processing documents, creating vector embeddings, and storing them in Qdrant.

- **Nodes Involved:**  
  - On form submission  
  - create collection1 (disabled)  
  - Qdrant Vector Store2  
  - Embeddings Mistral Cloud2  
  - Default Data Loader1  
  - Recursive Character Text Splitter  

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Starts workflow upon external form submissions (e.g., knowledge base updates).  
    - *Config:* Webhook URL exposed for form submissions.  
    - *Connections:* Parallel outputs to create collection1 (disabled) and Qdrant Vector Store2.  
    - *Edge cases:* Missing form data, webhook security.

  - **create collection1** (disabled)  
    - *Type:* HTTP Request  
    - *Role:* Intended to create a new collection in Qdrant but currently disabled.  
    - *Config:* HTTP call to Qdrant API.  
    - *Connections:* None (disabled).  
    - *Edge cases:* N/A due to disabled status.

  - **Qdrant Vector Store2**  
    - *Type:* LangChain Vector Store Node  
    - *Role:* Stores vector embeddings of new documents into Qdrant.  
    - *Config:* Connected to Qdrant instance.  
    - *Connections:* ai_embedding input from Embeddings Mistral Cloud2.  
    - *Edge cases:* Connection issues, data consistency.

  - **Embeddings Mistral Cloud2**  
    - *Type:* LangChain Embeddings Node  
    - *Role:* Converts new document text into vector embeddings.  
    - *Config:* Uses Mistral Cloud API.  
    - *Connections:* Outputs to Qdrant Vector Store2.  
    - *Edge cases:* API failures.

  - **Default Data Loader1**  
    - *Type:* LangChain Document Default Data Loader  
    - *Role:* Loads documents or content submitted via form for processing.  
    - *Config:* Handles input text or files.  
    - *Connections:* Outputs to Qdrant Vector Store2 as documents.  
    - *Edge cases:* Unsupported file types, corrupted data.

  - **Recursive Character Text Splitter**  
    - *Type:* LangChain Text Splitter Node  
    - *Role:* Splits large documents into chunks to optimize embedding and vector search.  
    - *Config:* Uses recursive character splitting strategy.  
    - *Connections:* Outputs to Default Data Loader1.  
    - *Edge cases:* Extremely large documents causing delays.

---

### 3. Summary Table

| Node Name                   | Node Type                                 | Functional Role                          | Input Node(s)              | Output Node(s)                    | Sticky Note                                                                                     |
|-----------------------------|-------------------------------------------|----------------------------------------|----------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Gmail Trigger1              | Gmail Trigger                            | Email reception trigger                 |                            | Extract Email Data               |                                                                                                 |
| Extract Email Data          | Set                                      | Extract key email fields                | Gmail Trigger1             | Filter New Customer Emails       |                                                                                                 |
| Filter New Customer Emails  | If                                       | Filter new vs existing customer emails | Extract Email Data          | Edit Fields1, Get Thread History |                                                                                                 |
| Get Thread History          | Gmail                                    | Fetch email thread history              | Filter New Customer Emails | Process Thread History           |                                                                                                 |
| Process Thread History      | Code                                     | Process and format thread messages      | Get Thread History         | Aggregate                       |                                                                                                 |
| Aggregate                  | Aggregate                               | Aggregate processed messages            | Process Thread History     | Edit Fields1                   |                                                                                                 |
| Edit Fields1               | Set                                      | Prepare AI input data                   | Aggregate                  | AI Analysis Agent               |                                                                                                 |
| AI Analysis Agent          | LangChain Agent                          | Orchestrate AI analysis                 | Edit Fields1               | Process AI Analysis             |                                                                                                 |
| OpenAI Chat Model1         | LangChain OpenAI Chat Model               | GPT-4 language model                    |                            | AI Analysis Agent (ai_languageModel) |                                                                                                 |
| Qdrant Vector Store        | LangChain Vector Store                    | Knowledge base vector search            | Embeddings Mistral Cloud   | AI Analysis Agent (ai_tool)     |                                                                                                 |
| Embeddings Mistral Cloud   | LangChain Embeddings                      | Generate text embeddings                 |                            | Qdrant Vector Store (ai_embedding) |                                                                                                 |
| Process AI Analysis        | Set                                      | Process AI output                       | AI Analysis Agent          | Edit Fields                    |                                                                                                 |
| Edit Fields               | Set                                      | Final formatting before escalation     | Process AI Analysis        | Needs Escalation?               |                                                                                                 |
| Needs Escalation?          | If                                       | Decide if issue needs escalation       | Edit Fields                | Needs Human Review Label, Send Automated Reply + Auto Resolved Label |                                                                                                 |
| Needs Human Review Label   | Gmail                                    | Label thread for human review          | Needs Escalation? (true)   | Log Ticket to Google Sheets     |                                                                                                 |
| Send Automated Reply       | Gmail                                    | Send AI-generated reply email          | Needs Escalation? (false)  | Log Ticket to Google Sheets     |                                                                                                 |
| Auto Resolved Label        | Gmail                                    | Label thread as auto-resolved          | Needs Escalation? (false)  | Log Ticket to Google Sheets     |                                                                                                 |
| Log Ticket to Google Sheets| Google Sheets                            | Log ticket data                        | Needs Human Review Label, Send Automated Reply, Auto Resolved Label | Send Status Update             |                                                                                                 |
| Send Status Update         | Telegram                                 | Notify ticket status                    | Log Ticket to Google Sheets|                                |                                                                                                 |
| On form submission         | Form Trigger                            | Trigger on external form submission    |                            | create collection1 (disabled), Qdrant Vector Store2 |                                                                                                 |
| create collection1         | HTTP Request (disabled)                  | Intended to create Qdrant collection   | On form submission         | None                          | Disabled node                                                                                   |
| Qdrant Vector Store2       | LangChain Vector Store                    | Store new embeddings in knowledge base | Embeddings Mistral Cloud2, Default Data Loader1 |                                |                                                                                                 |
| Embeddings Mistral Cloud2  | LangChain Embeddings                      | Generate embeddings for new documents  |                            | Qdrant Vector Store2 (ai_embedding) |                                                                                                 |
| Default Data Loader1       | LangChain Document Default Data Loader   | Load form-submitted documents           | Recursive Character Text Splitter | Qdrant Vector Store2 (ai_document) |                                                                                                 |
| Recursive Character Text Splitter | LangChain Text Splitter              | Split large documents into chunks       |                            | Default Data Loader1           |                                                                                                 |
| Process AI Analysis1       | Set                                      | Process AI output for form submissions | AI Analysis Agent1         |                              |                                                                                                 |
| AI Analysis Agent1         | LangChain Agent                          | AI analysis for form submissions        |                            | Process AI Analysis1           |                                                                                                 |
| OpenAI Chat Model2         | LangChain OpenAI Chat Model               | GPT-4 model for form submissions        |                            | AI Analysis Agent1 (ai_languageModel) |                                                                                                 |
| Qdrant Vector Store1       | LangChain Vector Store                    | Vector store for form submissions        | Embeddings Mistral Cloud1  | AI Analysis Agent1 (ai_tool)   |                                                                                                 |
| Embeddings Mistral Cloud1  | LangChain Embeddings                      | Embeddings for form submissions          |                            | Qdrant Vector Store1 (ai_embedding) |                                                                                                 |
| Structured Output Parser1  | LangChain Output Parser Structured        | Parses AI output into structured format |                            | AI Analysis Agent1 (ai_outputParser) |                                                                                                 |
| Sticky Note               | Sticky Note                              | (Empty content)                         |                            |                              |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Purpose: Start workflow on new incoming emails.  
   - Configure credentials for Gmail OAuth2.  
   - Set filters if needed (e.g., inbox only).

2. **Add a Set node "Extract Email Data"**  
   - Extract key fields: sender, subject, body, date from Gmail trigger data.  
   - Use expressions referencing incoming email JSON.

3. **Add an If node "Filter New Customer Emails"**  
   - Condition: Determine if email is from a new customer or existing thread.  
   - True branch proceeds to Edit Fields1, False branch to Get Thread History.

4. **Create Gmail node "Get Thread History"**  
   - Fetch all messages in thread using thread ID.  
   - Connect from False branch of filter.

5. **Add a Code node "Process Thread History"**  
   - JS code to parse messages, concatenate or clean data for AI processing.

6. **Add an Aggregate node**  
   - Aggregate messages for context preparation.

7. **Add a Set node "Edit Fields1"**  
   - Format and combine all context and email data into AI prompt inputs.

8. **Add LangChain OpenAI Chat Model node "OpenAI Chat Model1"**  
   - Configure GPT-4 model with required parameters.  
   - Set OpenAI credentials.

9. **Add LangChain Embeddings node "Embeddings Mistral Cloud"**  
   - Configure with Mistral Cloud API credentials.

10. **Add LangChain Vector Store node "Qdrant Vector Store"**  
    - Connect embedding input from Embeddings node.  
    - Configure Qdrant credentials and collection.

11. **Add LangChain Agent node "AI Analysis Agent"**  
    - Connect language model input from OpenAI Chat Model1.  
    - Connect tool input from Qdrant Vector Store.  
    - Configure for GPT-4 usage.

12. **Add a Set node "Process AI Analysis"**  
    - Extract AI response and flags (e.g., escalation needed) from agent output.

13. **Add a Set node "Edit Fields"**  
    - Prepare final data for escalation decision.

14. **Add If node "Needs Escalation?"**  
    - Condition: Check AI output flag for escalation.  
    - True branch to "Needs Human Review Label", False branch to "Send Automated Reply" and "Auto Resolved Label".

15. **Add Gmail nodes "Needs Human Review Label" and "Auto Resolved Label"**  
    - Configure to add respective labels to Gmail threads.

16. **Add Gmail node "Send Automated Reply"**  
    - Configure to send reply email using AI-generated content.

17. **Add Google Sheets node "Log Ticket to Google Sheets"**  
    - Connect outputs from label nodes and reply node.  
    - Configure spreadsheet and worksheet for logging.

18. **Add Telegram node "Send Status Update"**  
    - Configure with Telegram Bot credentials.  
    - Notify about ticket processing status.

19. **(Optional) Knowledge Base Management Block:**  
    - Add Form Trigger node "On form submission".  
    - Add LangChain Text Splitter node "Recursive Character Text Splitter".  
    - Add LangChain Document Default Data Loader node.  
    - Add LangChain Embeddings node "Embeddings Mistral Cloud2".  
    - Add LangChain Vector Store node "Qdrant Vector Store2".  
    - Configure each node with appropriate API keys and parameters.  
    - Connect flow from form submission through text splitter, data loader, embeddings, to vector store.

20. **Configure error handling and retries** on all nodes interacting with external services to handle API errors, timeouts, and authentication issues.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow uses GPT-4 via LangChain agents for AI analysis and reply generation.                   | Requires OpenAI API key with GPT-4 access.                                                         |
| Qdrant vector store integration is used for knowledge base search and update.                        | Requires Qdrant instance with appropriate API credentials.                                         |
| Mistral Cloud embeddings are used to generate vector representations of text.                        | Requires Mistral Cloud API credentials.                                                            |
| Google Sheets is used for ticket logging to maintain support history and reporting.                  | Requires Google Sheets API credentials with write access.                                          |
| Telegram bot is used for status update notifications to support team or admin channel.              | Requires Telegram Bot token and chat ID.                                                           |
| Gmail labels “Needs Human Review” and “Auto Resolved” must exist in the Gmail account labels.       | Setup in Gmail needed prior to usage.                                                              |
| The disabled "create collection1" HTTP Request node indicates potential future support for dynamic collection creation in Qdrant, currently inactive. | Can be enabled and configured if needed.                                                           |

---

**Disclaimer:**  
This documentation is derived exclusively from an n8n workflow configuration and respects all applicable content and usage policies. All processed data is legal and public.