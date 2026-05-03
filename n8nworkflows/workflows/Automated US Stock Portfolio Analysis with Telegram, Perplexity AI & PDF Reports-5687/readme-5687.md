Automated US Stock Portfolio Analysis with Telegram, Perplexity AI & PDF Reports

https://n8nworkflows.xyz/workflows/automated-us-stock-portfolio-analysis-with-telegram--perplexity-ai---pdf-reports-5687


# Automated US Stock Portfolio Analysis with Telegram, Perplexity AI & PDF Reports

### 1. Workflow Overview

This workflow automates the analysis of US stock portfolios by integrating Telegram messaging, AI research and analysis tools (OpenAI and Perplexity AI), and automated PDF report generation. It targets financial advisors or portfolio managers who want to receive up-to-date, AI-enhanced insights on their clients' stock portfolios and send personalized analysis reports via Telegram.

The workflow is logically organized into the following blocks:

- **1.1 Telegram Interaction & User Management**: Handles incoming Telegram messages, verifies user existence in Supabase database, creates new users if needed, and manages conversation states and memory.
- **1.2 Portfolio Data Retrieval & Client Looping**: Scheduled trigger that queries clients from a Postgres database and iterates over their portfolios for batch processing.
- **1.3 AI Research & Analysis Agents**: Uses Perplexity AI and OpenAI Chat models in multi-step agents to generate research insights and deep portfolio analysis.
- **1.4 Output Parsing & Formatting**: Parses AI outputs through structured and auto-fixing parsers, reformats responses, and prepares HTML reports.
- **1.5 PDF Report Generation & Delivery**: Converts HTML reports to PDFs, uploads them, and sends final analysis reports back to clients via Telegram.
- **1.6 Wallet & Status Updates**: Updates Supabase records about wallet info and tracks report delivery status.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Interaction & User Management

**Overview:**  
Handles incoming Telegram messages via webhook, verifies if the user exists in Supabase, creates new users if not, and manages the conversational flow with AI memory and agents.

**Nodes Involved:**  
- Telegram Trigger  
- GET Supabase User  
- IF User Exists  
- Create Supabase User  
- IF You Have PDF  
- Download PDF Telegram  
- Supabase (user update)  
- Code (for custom logic)  
- Conversation Agent  
- OpenAI Conversation  
- Postgres Chat Memory  
- Send Reply Conversation  
- update_wallet (Supabase tool node)  

**Node Details:**  
- **Telegram Trigger**: Entry webhook node that listens for Telegram messages; configured with Telegram Bot credentials.  
- **GET Supabase User**: Queries Supabase for user data based on Telegram user ID; always outputs data to handle user presence logic.  
- **IF User Exists**: Conditional node branching flow if user is found or not.  
- **Create Supabase User**: Inserts new user data into Supabase if user does not exist.  
- **IF You Have PDF**: Checks if the user already has a PDF report available; branches accordingly to download or update user data.  
- **Download PDF Telegram**: Downloads the existing PDF from storage to send via Telegram.  
- **Supabase (user update)**: Updates user record in Supabase after PDF generation or other changes; configured with Supabase credentials.  
- **Code**: Contains JavaScript for custom data transformations or conditional checks.  
- **Conversation Agent**: LangChain agent node managing AI conversational logic.  
- **OpenAI Conversation**: OpenAI Chat model node integrated as AI engine for conversational responses.  
- **Postgres Chat Memory**: Maintains chat history in Postgres for context-aware AI conversations.  
- **Send Reply Conversation**: Sends AI-generated reply back to Telegram user via Telegram node.  
- **update_wallet**: Uses Supabase tool node to update wallet-related info as part of user management.  

**Edge Cases / Failures:**  
- Telegram webhook failure or timeout  
- Supabase connectivity/auth issues  
- User data missing or malformed  
- AI conversation memory retrieval errors  
- PDF download failures due to missing files or permissions  

---

#### 2.2 Portfolio Data Retrieval & Client Looping

**Overview:**  
Triggered on schedule, queries clients from Postgres, loops over each client’s portfolio batch-wise for processing.

**Nodes Involved:**  
- Schedule Trigger  
- Search Clients (Postgres query)  
- Loop Over Items (splitInBatches)  

**Node Details:**  
- **Schedule Trigger**: Runs periodically to start portfolio analysis automatically.  
- **Search Clients**: Postgres node querying client portfolio data; uses SQL to fetch relevant client info.  
- **Loop Over Items**: Splits client list into batches for sequential processing, preventing overload and managing rate limits.  

**Edge Cases / Failures:**  
- Database query timeouts or errors  
- Large datasets causing memory or timeout issues  
- Batch size misconfiguration  

---

#### 2.3 AI Research & Analysis Agents

**Overview:**  
Performs research on portfolio stocks using Perplexity AI and OpenAI chat models orchestrated by LangChain agents; parses and refines AI output.

**Nodes Involved:**  
- RESEARCH REQUEST SPECIALIST AGENT  
- Message a model (Perplexity)  
- Perplexity Response Formatter (Code)  
- OpenAI Chat Model2  
- OpenAI Chat Model1  
- Auto-fixing Output Parser  
- Structured Output Parser  
- HTML FORMATTER AGENT  
- HTML Report Generator  

**Node Details:**  
- **RESEARCH REQUEST SPECIALIST AGENT**: Agent node that coordinates AI research requests on stock data.  
- **Message a model**: Perplexity AI node sending research queries and receiving responses.  
- **Perplexity Response Formatter**: Code node that cleans and formats Perplexity AI raw output.  
- **OpenAI Chat Model2 and OpenAI Chat Model1**: OpenAI Chat models configured with conversational prompts for research and analysis.  
- **Auto-fixing Output Parser**: LangChain node that attempts to fix parsing errors in AI output automatically.  
- **Structured Output Parser**: Parses AI output into structured JSON for easier downstream processing.  
- **HTML FORMATTER AGENT**: Agent that transforms AI analysis into HTML-formatted content.  
- **HTML Report Generator**: Code node that generates the final HTML report from formatted AI content.  

**Edge Cases / Failures:**  
- AI model rate limiting or timeouts  
- Parsing failures due to unexpected AI output format  
- API authentication errors  
- Formatting code exceptions  

---

#### 2.4 PDF Report Generation & Delivery

**Overview:**  
Converts HTML reports to PDF files, uploads to external storage, and sends files via Telegram to clients.

**Nodes Involved:**  
- HTML  
- PDF Generator (PDF.co API)  
- HTTP Request (upload or download related)  
- Telegram  
- Update Sent (Supabase)  

**Node Details:**  
- **HTML**: Converts HTML content into a format suitable for PDF generation.  
- **PDF Generator**: Uses PDF.co API to convert HTML to PDF; configured with API keys and conversion parameters.  
- **HTTP Request**: Handles file uploads/downloads for report storage or retrieval.  
- **Telegram**: Sends the generated PDF report to users via Telegram bot.  
- **Update Sent**: Updates Supabase to mark report as sent for client tracking.  

**Edge Cases / Failures:**  
- PDF.co API quota exceeded or errors  
- File upload/download network issues  
- Telegram file size limits or message sending failures  
- Supabase update failures  

---

#### 2.5 Wallet & Status Updates

**Overview:**  
Manages the updating or creation of wallet records in Supabase and triggers confirmation messages after report delivery.

**Nodes Involved:**  
- If (conditional on wallet existence)  
- Update wallet (Supabase)  
- Created Supabase Wallet  
- Send Analysis Confirmation (Telegram)  

**Node Details:**  
- **If**: Checks if a wallet exists for the client to branch update vs. create logic.  
- **Update wallet**: Updates wallet information in Supabase.  
- **Created Supabase Wallet**: Creates a new wallet record if none exists.  
- **Send Analysis Confirmation**: Telegram node sends confirmation message that analysis was completed and report sent.  

**Edge Cases / Failures:**  
- Supabase record conflicts or permission issues  
- Telegram confirmation message not delivered  
- Conditional logic misbranches due to unexpected data  

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                  | Input Node(s)                      | Output Node(s)                     | Sticky Note                            |
|-------------------------------|--------------------------------------|--------------------------------|----------------------------------|-----------------------------------|--------------------------------------|
| Telegram Trigger               | telegramTrigger                      | Entry point for Telegram msgs  |                                  | GET Supabase User                 |                                      |
| GET Supabase User             | supabase                            | Retrieve user data             | Telegram Trigger                 | IF User Exists                   |                                      |
| IF User Exists                | if                                  | Branch if user exists          | GET Supabase User                | IF You Have PDF, Create Supabase User |                                      |
| Create Supabase User          | supabase                            | Add new user                  | IF User Exists                  | IF You Have PDF                  |                                      |
| IF You Have PDF               | if                                  | Check if user has PDF          | IF User Exists, Create Supabase User | Download PDF Telegram, Supabase   |                                      |
| Download PDF Telegram         | telegram                            | Download PDF for sending       | IF You Have PDF                 | HTTP Request4                   |                                      |
| Supabase (user update)        | supabase                            | Update user info               | IF You Have PDF                 | Code                           |                                      |
| Code                         | code                                | Custom logic                  | Supabase                       | Conversation Agent              |                                      |
| Conversation Agent            | langchain.agent                     | AI conversational agent       | Code                          | Send Reply Conversation         |                                      |
| OpenAI Conversation           | langchain.lmChatOpenAi              | OpenAI chat model             | Conversation Agent             | Conversation Agent              |                                      |
| Postgres Chat Memory          | langchain.memoryPostgresChat        | Chat memory                   |                              | Conversation Agent              |                                      |
| Send Reply Conversation       | telegram                            | Send reply to user            | Conversation Agent             |                               |                                      |
| update_wallet                 | supabaseTool                       | Update wallet info            |                              | Conversation Agent              |                                      |
| Schedule Trigger             | scheduleTrigger                    | Scheduled portfolio processing |                                | Search Clients                 |                                      |
| Search Clients               | postgres                           | Query clients                 | Schedule Trigger               | Loop Over Items                |                                      |
| Loop Over Items              | splitInBatches                    | Batch processing clients      | Search Clients                | RESEARCH REQUEST SPECIALIST AGENT |                                      |
| RESEARCH REQUEST SPECIALIST AGENT | langchain.agent                  | AI research coordination      | Loop Over Items               | PARSE RESEARCH                |                                      |
| Message a model              | perplexity                         | Perplexity AI query           | PARSE RESEARCH                | Perplexity Response Formatter |                                      |
| Perplexity Response Formatter | code                              | Format Perplexity output      | Message a model               | HTML FORMATTER AGENT          |                                      |
| OpenAI Chat Model1           | langchain.lmChatOpenAi              | AI chat model for formatting  | Auto-fixing Output Parser      | HTML FORMATTER AGENT          |                                      |
| Auto-fixing Output Parser    | langchain.outputParserAutofixing    | Fix AI output parsing errors  | Structured Output Parser       | HTML FORMATTER AGENT          |                                      |
| Structured Output Parser     | langchain.outputParserStructured    | Parse AI output structured    | OpenAI Chat Model              | Auto-fixing Output Parser      |                                      |
| HTML FORMATTER AGENT         | langchain.agent                     | Format AI content to HTML     | Auto-fixing Output Parser      | HTML Report Generator          |                                      |
| HTML Report Generator        | code                              | Generate HTML report          | HTML FORMATTER AGENT          | HTML                          |                                      |
| HTML                        | html                              | Prepare HTML for PDF          | HTML Report Generator          | PDF Generator                 |                                      |
| PDF Generator               | pdfco.Api                         | Convert HTML to PDF           | HTML                         | HTTP Request                  |                                      |
| HTTP Request                | httpRequest                       | Upload or download files      | PDF Generator                 | Telegram                     |                                      |
| Telegram                   | telegram                          | Send PDF report               | HTTP Request                  | Update Sent                  |                                      |
| Update Sent                | supabase                         | Mark report sent              | Telegram                     | Loop Over Items              |                                      |
| IF                            | if                                  | Check wallet existence        | GET Supabase User1             | Update wallet, Created Supabase Wallet |                                      |
| Update wallet               | supabase                         | Update wallet record          | IF                          | Send Analysis Confirmation |                                      |
| Created Supabase Wallet     | supabase                         | Create wallet record          | IF                          | Send Analysis Confirmation |                                      |
| Send Analysis Confirmation | telegram                          | Notify user of completion     | Update wallet, Created Supabase Wallet | RESEARCH REQUEST SPECIALIST AGENT1 |                                      |
| RESEARCH REQUEST SPECIALIST AGENT1 | langchain.agent                  | Follow-up AI research         | Send Analysis Confirmation    | PARSE RESEARCH1              |                                      |
| Message a model1            | perplexity                         | Perplexity AI query           | PARSE RESEARCH1               | Perplexity Response Formatter1 |                                      |
| Perplexity Response Formatter1 | code                              | Format Perplexity output      | Message a model1              | HTML FORMATTER AGENT1         |                                      |
| HTML FORMATTER AGENT1       | langchain.agent                     | Format AI content to HTML     | Auto-fixing Output Parser1    | HTML Report Generator1        |                                      |
| HTML Report Generator1      | code                              | Generate HTML report          | HTML FORMATTER AGENT1         | HTML1                        |                                      |
| HTML1                      | html                              | Prepare HTML for PDF          | HTML Report Generator1        | PDF Generator1               |                                      |
| PDF Generator1             | pdfco.Api                         | Convert HTML to PDF           | HTML1                        | HTTP Request3                |                                      |
| HTTP Request3              | httpRequest                       | Upload/download files         | PDF Generator1               | Telegram1                   |                                      |
| Telegram1                  | telegram                          | Send PDF report               | HTTP Request3                | Supabase1                   |                                      |
| Supabase1                  | supabase                         | Update user data              | Telegram1                   |                             |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Configure Webhook Nodes**  
   - Create `Telegram Trigger` node with your Telegram Bot credentials. Set it as the entry point for incoming messages.  
   - Create `Telegram` nodes for sending messages (`Send Reply Conversation`, `Telegram`, `Telegram1`, `Send Analysis Confirmation`, `Download PDF Telegram`) configured with the same bot credentials.

2. **Setup Supabase User Management**  
   - Add `GET Supabase User` node to query user data by Telegram user ID. Configure credentials for your Supabase instance.  
   - Add `IF User Exists` node to branch the flow based on user presence.  
   - Create `Create Supabase User` node to insert new users.  
   - Add `IF You Have PDF` node to check if user already has a generated PDF report.  
   - Add `Download PDF Telegram` node to retrieve existing PDFs for sending.  
   - Add `Supabase` update nodes (`Supabase`, `Supabase1`) to update user and report status.  
   - Add `Code` node for any custom logic needed to handle user data or conditions.

3. **Implement AI Conversational Flow**  
   - Add `Postgres Chat Memory` node configured with your Postgres instance to store chat history.  
   - Add `OpenAI Conversation` node configured with OpenAI credentials, linked to `Conversation Agent` (LangChain agent node).  
   - Configure `Conversation Agent` to manage prompts and integrate with memory.  
   - Connect `Conversation Agent` output to `Send Reply Conversation` to send AI responses back to Telegram.

4. **Scheduled Portfolio Processing**  
   - Add `Schedule Trigger` node to run periodic portfolio analysis.  
   - Add `Search Clients` node (Postgres) that queries client portfolios.  
   - Add `Loop Over Items` node to batch process clients to avoid overload.

5. **AI Research & Analysis Agents**  
   - Add `RESEARCH REQUEST SPECIALIST AGENT` LangChain agent configured with OpenAI Chat and Perplexity AI credentials to generate research data.  
   - Add `Message a model` node (Perplexity) for external research queries.  
   - Add `Perplexity Response Formatter` (Code node) to clean Perplexity output.  
   - Add OpenAI Chat Model nodes (`OpenAI Chat Model1`, `OpenAI Chat Model2`) configured for analysis tasks.  
   - Add `Structured Output Parser` and `Auto-fixing Output Parser` to ensure AI responses are correctly formatted and parsed.  
   - Add `HTML FORMATTER AGENT` to convert parsed AI data into HTML format.  
   - Add `HTML Report Generator` (Code node) to create final HTML report content.

6. **PDF Generation and Delivery**  
   - Add `HTML` node to prepare HTML content for PDF conversion.  
   - Add `PDF Generator` node configured with PDF.co API credentials to convert HTML to PDF.  
   - Add `HTTP Request` node(s) to handle PDF file upload or download as needed.  
   - Add `Telegram` nodes to send PDF files to clients.  
   - Add `Update Sent` Supabase node to mark reports as sent.

7. **Wallet Management & Confirmation**  
   - Add `GET Supabase User1` node to fetch wallet info.  
   - Add `IF` node to check wallet existence.  
   - Add `Update wallet` and `Created Supabase Wallet` nodes to update or create wallet records accordingly.  
   - Add `Send Analysis Confirmation` Telegram node to notify user of report delivery.  
   - Link confirmation node to a follow-up AI research agent (`RESEARCH REQUEST SPECIALIST AGENT1`) for continuous analysis if desired.

8. **Additional Utilities**  
   - Add `Wait` node to manage timing between HTTP requests or message sending for rate limiting.  
   - Add any additional `Code` nodes as needed for customized transformations or error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow integrates both OpenAI and Perplexity AI for complementary research and analysis.     | Combines advanced AI models for richer portfolio insights.                                                  |
| PDF generation uses PDF.co API — ensure API keys and quotas are configured correctly.                | https://pdf.co/                                                                                              |
| Telegram messaging nodes require bot token and webhook setup with proper permissions.                | n8n Telegram integration docs: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.telegram/        |
| Supabase nodes require service key with appropriate read/write permissions for users and wallets.   | Supabase docs: https://supabase.com/docs                                                                       |
| Postgres Chat Memory node stores conversational context for AI agents to maintain dialogue history. | Useful for context-aware AI conversations in Telegram chat flows.                                            |

---

This documentation covers all nodes and their connections, allowing developers or AI agents to understand, rebuild, or modify the workflow confidently. It anticipates integration points and potential error sources, facilitating robust automation of US stock portfolio analysis with Telegram and AI.