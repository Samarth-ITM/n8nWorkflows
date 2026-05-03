WhatsApp Expense Tracker with Multi-Input (Text, Image & Audio)

https://n8nworkflows.xyz/workflows/whatsapp-expense-tracker-with-multi-input--text--image---audio--5201


# WhatsApp Expense Tracker with Multi-Input (Text, Image & Audio)

### 1. Workflow Overview

This workflow implements a WhatsApp-based multi-input expense tracking system that handles text messages, images (e.g., receipts), and audio messages (voice notes). Its main purpose is to enable users to log, query, and interact with their financial transactions through WhatsApp in natural language, supporting various input formats and providing detailed financial summaries or confirmations.

Logical blocks are organized as follows:

- **1.1 Input Reception & User Verification:** Captures incoming WhatsApp messages via webhook, identifies message type (text, image, audio), verifies user existence in the database or creates new users.

- **1.2 Intent Classification & Routing:** Uses AI to classify the intent of the message (e.g., transaction logging, report query, casual conversation) and routes the workflow accordingly.

- **1.3 Text-Based Transaction Processing:** Parses, validates, splits, and inserts structured transaction data from text messages.

- **1.4 Image-Based Transaction Processing:** Downloads images, converts them to base64, uses OCR to extract text, and processes extracted data for transaction logging.

- **1.5 Audio-Based Transaction Processing:** Downloads audio files, transcribes audio to text with Deepgram, then processes transcribed text for transaction logging.

- **1.6 Custom Report Query Handling:** Builds, transforms, executes, and formats database queries based on user financial report requests.

- **1.7 Financial Summary & Confirmation:** Constructs and sends financial summaries or confirmation messages back to the user via WhatsApp.

- **1.8 Supporting Tools & Utilities:** Includes date/time tools, calculators, output parsers, and converters used across blocks.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & User Verification

- **Overview:**  
Receives incoming WhatsApp messages through a webhook, determines message type, checks for user existence in PostgreSQL database, and creates new users if needed.

- **Nodes Involved:**  
  - Incoming WhatsApp Trigger  
  - Check Message Type (If)  
  - Fetch User Profile from Postgres  
  - Check user Exists or not (If)  
  - Create New User  

- **Node Details:**  

  - *Incoming WhatsApp Trigger*  
    - Type: WhatsApp Webhook Trigger  
    - Role: Entry point for all incoming WhatsApp messages  
    - Configuration: Configured with a webhook ID specific to WhatsApp integration  
    - Inputs: WhatsApp messages  
    - Outputs: Message payload forwarded for type checking  
    - Edge cases: Webhook failures or misconfigured webhook ID  

  - *Check Message Type*  
    - Type: If node  
    - Role: Logical branching based on message type (text, image, audio)  
    - Key variables: Checks media type from incoming message payload  
    - Inputs: Incoming WhatsApp Trigger  
    - Outputs: Routes to different branches depending on type  
    - Edge cases: Unknown or unsupported media types  

  - *Fetch User Profile from Postgres*  
    - Type: PostgreSQL node  
    - Role: Retrieves user record by WhatsApp sender ID  
    - Configuration: Uses PostgreSQL credentials, queries user table  
    - Inputs: Phone number or user ID from message payload  
    - Outputs: User data or empty if not found  
    - Edge cases: Database connectivity or query errors  

  - *Check user Exists or not*  
    - Type: If node  
    - Role: Checks if user data was returned from DB  
    - Inputs: Fetch User Profile output  
    - Outputs: Routes to Create New User if not exists, else to Intent Classifier  
    - Edge cases: False negatives if DB returns empty or error  

  - *Create New User*  
    - Type: PostgreSQL node  
    - Role: Inserts new user record into DB  
    - Configuration: Inserts phone number and metadata  
    - Inputs: User phone info from trigger  
    - Outputs: Confirmation and passes flow to Intent Classifier  
    - Edge cases: DB insertion errors, duplication  

---

#### 2.2 Intent Classification & Routing

- **Overview:**  
Classifies incoming messages’ intent using LangChain AI agent and routes messages to appropriate processing blocks.

- **Nodes Involved:**  
  - Message Intent Classifier  
  - Route by Intent (Switch)  

- **Node Details:**  

  - *Message Intent Classifier*  
    - Type: LangChain Agent  
    - Role: Uses AI to classify message intent (e.g., add transaction, query report, conversation)  
    - Configuration: Connected to OpenAI Chat Model5 for language understanding  
    - Inputs: User message text or extracted content  
    - Outputs: Intent classification result to Switch node  
    - Edge cases: Misclassification due to ambiguous input or AI model errors  

  - *Route by Intent*  
    - Type: Switch node  
    - Role: Routes flow into one of multiple paths based on intent value  
    - Outputs:  
      1. Build Custom Report Query (for report requests)  
      2. Parse & Validate Transaction (text transactions)  
      3. Create Image Download Link (image transactions)  
      4. Create Audio Download Link (audio transactions)  
      5. Simple Conversation (casual or unrelated chat)  
    - Edge cases: Unmatched intent leading to no route, routing conflicts  

---

#### 2.3 Text-Based Transaction Processing

- **Overview:**  
Processes text-based transaction messages by parsing, validating, splitting multiple transactions, and inserting into database.

- **Nodes Involved:**  
  - Parse & Validate Transaction  
  - Split Transaction Lines  
  - Format Transaction JSON1 (Code)  
  - Validate Transaction Fields1 (Code)  
  - Insert Transaction into DB1  
  - Build Confirmation Message1  
  - Send Financial Response1  

- **Node Details:**  

  - *Parse & Validate Transaction*  
    - Type: LangChain Agent  
    - Role: Parses transaction details from text using AI  
    - Connected to: OpenAI Chat Model7 and Structured Output Parser  
    - Inputs: Raw text message  
    - Outputs: Structured transaction data  
    - Edge cases: Parsing errors, incomplete data  

  - *Split Transaction Lines*  
    - Type: SplitOut  
    - Role: Splits multiple transactions if present in one message  
    - Inputs: Structured transaction array from parser  
    - Outputs: Individual transaction items for processing  

  - *Format Transaction JSON1*  
    - Type: Code  
    - Role: Formats each transaction into JSON suitable for DB insertion  
    - Inputs: Single transaction item  
    - Outputs: JSON object with required DB fields  
    - Edge cases: Formatting or missing fields  

  - *Validate Transaction Fields1*  
    - Type: Code  
    - Role: Validates required fields are present and correct (amount, date, category)  
    - Outputs: Passes valid transactions, errors on invalid  

  - *Insert Transaction into DB1*  
    - Type: PostgreSQL  
    - Role: Inserts validated transactions into transactions table  
    - Inputs: Formatted JSON  
    - Edge cases: DB errors, constraint violations  

  - *Build Confirmation Message1*  
    - Type: LangChain Agent  
    - Role: Builds a user-friendly confirmation message summarizing inserted transactions  
    - Connected to: OpenAI Chat Model1 and Date and time7 tool  
    - Outputs: Text message  

  - *Send Financial Response1*  
    - Type: WhatsApp node  
    - Role: Sends confirmation message back to user  
    - Inputs: Confirmation text  
    - Edge cases: WhatsApp sending errors  

---

#### 2.4 Image-Based Transaction Processing

- **Overview:**  
Manages receipt or bill images by downloading, converting to base64, performing OCR, and processing extracted text for transactions.

- **Nodes Involved:**  
  - Create Image Download link  
  - Download Image  
  - Transform image to base64  
  - Run OCR on Image (Gemini API)  
  - Parse & Validate Transaction (Receipts & Bills)  
  - Split Transaction Lines  
  - Format Transaction JSON1  
  - Validate Transaction Fields1  
  - Insert Transaction into DB1  
  - Build Confirmation Message1  
  - Send Financial Response1  

- **Node Details:**  

  - *Create Image Download link*  
    - Type: WhatsApp node  
    - Role: Generates a secure download URL for the image media  
    - Inputs: Incoming media metadata  
    - Outputs: URL for download  

  - *Download Image*  
    - Type: HTTP Request  
    - Role: Downloads the image file from WhatsApp media URL  
    - Edge cases: Download failures, invalid URLs  

  - *Transform image to base64*  
    - Type: ExtractFromFile  
    - Role: Converts binary image to base64 encoding for OCR API  

  - *Run OCR on Image (Gemini API)*  
    - Type: HTTP Request  
    - Role: Sends base64 image to Gemini OCR API and receives extracted text  
    - Configuration: API endpoint, authentication  
    - Edge cases: OCR failures, rate limits  

  - *Parse & Validate Transaction (Receipts & Bills)*  
    - Type: LangChain Agent  
    - Role: Parses transaction data from OCR text using OpenAI Chat Model2 and Structured Output Parser1  
    - Outputs: Structured transaction data  

  - Remaining nodes follow the same processing as text transaction block (split, format, validate, insert, confirm, send)  

---

#### 2.5 Audio-Based Transaction Processing

- **Overview:**  
Handles voice note messages by downloading audio, transcribing speech using Deepgram API, then processing transcription text for transactions.

- **Nodes Involved:**  
  - Create Audio Download link  
  - Download Audio  
  - Transcribe Audio by Deepgram  
  - Parse & Validate Transaction(Audio Transcribe)  
  - Split Transaction Lines  
  - Format Transaction JSON1  
  - Validate Transaction Fields1  
  - Insert Transaction into DB1  
  - Build Confirmation Message1  
  - Send Financial Response1  

- **Node Details:**  

  - *Create Audio Download link*  
    - Type: WhatsApp node  
    - Role: Generates download URL for audio message  

  - *Download Audio*  
    - Type: HTTP Request  
    - Role: Downloads audio file  

  - *Transcribe Audio by Deepgram*  
    - Type: HTTP Request  
    - Role: Sends audio to Deepgram transcription API, receives text transcript  
    - Configuration: API key, endpoint  
    - Edge cases: Transcription inaccuracies, API failures  

  - *Parse & Validate Transaction(Audio Transcribe)*  
    - Type: LangChain Agent  
    - Role: Parses transactions from transcribed text using OpenAI Chat Model3 and Structured Output Parser3  

  - Remaining steps same as text transaction processing  

---

#### 2.6 Custom Report Query Handling

- **Overview:**  
Builds and executes custom financial queries to generate reports per user request.

- **Nodes Involved:**  
  - Build Custom Report Query  
  - Transform Custom Query for Postgres  
  - Execute Custom Report Query  
  - Format Custom Query Results  
  - Convert to File  
  - Build Custom Financial Summary  
  - Send Custom Report  

- **Node Details:**  

  - *Build Custom Report Query*  
    - Type: LangChain Agent  
    - Role: Converts user query into a structured query request using OpenAI Chat Model6 and Date and time1 tool  
    - Outputs: Query parameters  

  - *Transform Custom Query for Postgres*  
    - Type: Code  
    - Role: Converts structured query into actual SQL query string for PostgreSQL  
    - Edge cases: SQL injection protection needed  

  - *Execute Custom Report Query*  
    - Type: PostgreSQL node  
    - Role: Runs SQL query and fetches results  

  - *Format Custom Query Results*  
    - Type: Code  
    - Role: Formats query results into user-friendly JSON or text  

  - *Convert to File*  
    - Type: ConvertToFile  
    - Role: Converts formatted data to a file attachment if needed  

  - *Build Custom Financial Summary*  
    - Type: LangChain Agent  
    - Role: Builds summary narrative or report text using OpenAI Chat Model and Calculator tool  

  - *Send Custom Report*  
    - Type: WhatsApp node  
    - Role: Sends report back to user  

---

#### 2.7 Financial Summary & Confirmation

- **Overview:**  
Constructs and sends user-facing confirmation messages and financial summaries.

- **Nodes Involved:**  
  - Build Confirmation Message1  
  - Send Financial Response1  
  - Build Custom Financial Summary  
  - Send Custom Report  

- **Node Details:**  
See details above in transaction blocks and report handling.

---

#### 2.8 Supporting Tools & Utilities

- **Overview:**  
Reusable LangChain tools and utility nodes supporting date/time handling, calculations, and output parsing.

- **Nodes Involved:**  
  - Date and time (multiple variants)  
  - Calculator  
  - Structured Output Parser (multiple)  
  - OpenAI Chat Models (multiple instances)  

- **Node Details:**  
Each Date and time tool provides timestamp or date context for AI agents. Calculator assists in financial calculations. Structured Output Parsers convert AI responses into structured JSON.

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                            | Input Node(s)                     | Output Node(s)                    | Sticky Note               |
|-------------------------------|----------------------------------|--------------------------------------------|----------------------------------|----------------------------------|---------------------------|
| Incoming WhatsApp Trigger      | WhatsApp Trigger                 | Entry point for incoming WhatsApp messages | —                                | Check Message Type               |                           |
| Check Message Type             | If                              | Branch by message type (text/image/audio)  | Incoming WhatsApp Trigger         | Fetch User Profile from Postgres |                           |
| Fetch User Profile from Postgres| PostgreSQL                      | Retrieve user profile by sender ID          | Check Message Type                | Check user Exists or not         |                           |
| Check user Exists or not       | If                              | Determines if user exists                    | Fetch User Profile from Postgres  | Create New User, Message Intent Classifier |                  |
| Create New User                | PostgreSQL                      | Insert new user record                       | Check user Exists or not          | Message Intent Classifier        |                           |
| Message Intent Classifier      | LangChain Agent                 | Classify message intent                      | Create New User / Check user Exists or not | Route by Intent           |                           |
| Route by Intent                | Switch                         | Route flow based on intent                   | Message Intent Classifier         | Multiple (see block 2.2)         |                           |
| Parse & Validate Transaction   | LangChain Agent                 | Parse & validate text transactions           | Route by Intent                   | Split Transaction Lines          |                           |
| Split Transaction Lines        | SplitOut                       | Split multiple transactions                  | Parse & Validate Transaction      | Format Transaction JSON1         |                           |
| Format Transaction JSON1       | Code                          | Format transaction JSON for DB               | Split Transaction Lines           | Validate Transaction Fields1     |                           |
| Validate Transaction Fields1   | Code                          | Validate transaction fields                   | Format Transaction JSON1          | Insert Transaction into DB1      |                           |
| Insert Transaction into DB1    | PostgreSQL                    | Insert transaction into DB                    | Validate Transaction Fields1      | Build Confirmation Message1      |                           |
| Build Confirmation Message1    | LangChain Agent               | Build confirmation message                    | Insert Transaction into DB1       | Send Financial Response1         |                           |
| Send Financial Response1       | WhatsApp                      | Send confirmation via WhatsApp                | Build Confirmation Message1       | —                               |                           |
| Create Image Download link     | WhatsApp                      | Generate image download link                   | Route by Intent                   | Download Image                  |                           |
| Download Image                | HTTP Request                 | Download image file                            | Create Image Download link        | Transform image to base64        |                           |
| Transform image to base64      | ExtractFromFile               | Convert image to base64 for OCR                | Download Image                   | Run OCR on Image (Gemini API)   |                           |
| Run OCR on Image (Gemini API) | HTTP Request                 | Perform OCR on image                            | Transform image to base64         | Parse & Validate Transaction (Receipts & Bills) |                |
| Parse & Validate Transaction (Receipts & Bills) | LangChain Agent | Parse & validate OCR text transactions         | Run OCR on Image (Gemini API)    | Split Transaction Lines          |                           |
| Create Audio Download link     | WhatsApp                      | Generate audio download link                   | Route by Intent                   | Download Audio                 |                           |
| Download Audio                | HTTP Request                 | Download audio file                            | Create Audio Download link        | Transcribe Audio by deepgram     |                           |
| Transcribe Audio by deepgram   | HTTP Request                 | Transcribe audio to text                        | Download Audio                   | Parse & Validate Transaction(Audio Transcribe) |               |
| Parse & Validate Transaction(Audio Transcribe) | LangChain Agent | Parse & validate transcribed text transactions | Transcribe Audio by deepgram      | Split Transaction Lines          |                           |
| Build Custom Report Query      | LangChain Agent               | Build user financial report query              | Route by Intent                   | Transform Custom Query for Postgres |                           |
| Transform Custom Query for Postgres | Code                      | Convert query into SQL                          | Build Custom Report Query         | Execute Custom Report Query      |                           |
| Execute Custom Report Query    | PostgreSQL                   | Execute SQL and fetch report                    | Transform Custom Query for Postgres | Format Custom Query Results      |                           |
| Format Custom Query Results    | Code                          | Format report results for user                   | Execute Custom Report Query       | Convert to File                 |                           |
| Convert to File                | ConvertToFile                | Convert formatted results to file                | Format Custom Query Results       | Build Custom Financial Summary   |                           |
| Build Custom Financial Summary | LangChain Agent               | Prepare financial summary narrative              | Convert to File, Calculator       | Send Custom Report              |                           |
| Send Custom Report             | WhatsApp                      | Send financial summary or report                 | Build Custom Financial Summary    | —                               |                           |
| Calculator                    | LangChain ToolCode           | Financial calculations utility                    | —                                | Build Custom Financial Summary   |                           |
| Date and time (multiple nodes) | LangChain ToolCode           | Date/time context for AI agents                   | —                                | Various connected AI agents      |                           |
| OpenAI Chat Model (multiple)   | LangChain LmChatOpenAi       | OpenAI GPT models for AI processing              | Connected to respective agents    | Agents                          |                           |
| Normal Conversation            | WhatsApp                      | Handles simple chat conversations                 | simple conversation               | —                               |                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Incoming WhatsApp Trigger Node:**  
   - Type: WhatsApp Trigger  
   - Configure webhook ID for WhatsApp integration  

2. **Add "Check Message Type" If Node:**  
   - Check incoming message media type (text, image, audio)  
   - Connect Incoming WhatsApp Trigger output to this node  

3. **Add PostgreSQL Node "Fetch User Profile from Postgres":**  
   - Query user table for sender ID/phone number  
   - Connect output of Check Message Type to this node  

4. **Add If Node "Check user Exists or not":**  
   - Check if user record exists (based on DB query result)  
   - Connect Fetch User Profile to this node  

5. **Create PostgreSQL Node "Create New User":**  
   - Insert new user if user does not exist  
   - Connect false branch of Check user Exists or not to this node  

6. **Create LangChain Agent Node "Message Intent Classifier":**  
   - Connect output of Create New User and true branch of Check user Exists or not to this node  
   - Configure with OpenAI credential and appropriate prompt/model  

7. **Add Switch Node "Route by Intent":**  
   - Connect output of Message Intent Classifier  
   - Setup multiple branches for intents: text transaction, image transaction, audio transaction, report, conversation  

8. **For Text Transaction Branch:**  
   - Add LangChain Agent "Parse & Validate Transaction" with OpenAI model and structured output parser  
   - Connect to SplitOut node "Split Transaction Lines"  
   - Add Code node "Format Transaction JSON1" to format each item  
   - Add Code node "Validate Transaction Fields1" for validation  
   - Add PostgreSQL node "Insert Transaction into DB1" to insert transactions  
   - Add LangChain Agent "Build Confirmation Message1" for confirmation text  
   - Add WhatsApp node "Send Financial Response1" to send confirmation  

9. **For Image Transaction Branch:**  
   - Add WhatsApp node "Create Image Download link"  
   - Connect to HTTP Request "Download Image"  
   - Connect to ExtractFromFile "Transform image to base64"  
   - Connect to HTTP Request "Run OCR on Image (Gemini API)" with correct API credentials  
   - Connect to LangChain Agent "Parse & Validate Transaction (Receipts & Bills)" with OpenAI and parser  
   - Then follow same flow as text transaction branch for splitting, formatting, validating, inserting, confirming, and sending  

10. **For Audio Transaction Branch:**  
    - Add WhatsApp node "Create Audio Download link"  
    - Connect to HTTP Request "Download Audio"  
    - Connect to HTTP Request "Transcribe Audio by Deepgram" with Deepgram API key  
    - Connect to LangChain Agent "Parse & Validate Transaction(Audio Transcribe)" with OpenAI and parser  
    - Then follow the same flow as text transaction branch  

11. **For Report Query Branch:**  
    - Add LangChain Agent "Build Custom Report Query" with OpenAI and date/time tool  
    - Add Code node "Transform Custom Query for Postgres" to build SQL query  
    - Add PostgreSQL node "Execute Custom Report Query"  
    - Add Code node "Format Custom Query Results"  
    - Add ConvertToFile node "Convert to File"  
    - Add LangChain Agent "Build Custom Financial Summary" with Calculator and date/time tool  
    - Add WhatsApp node "Send Custom Report"  

12. **For Simple Conversation Branch:**  
    - Add LangChain Agent "simple conversation" connected to WhatsApp "Normal Conversation" node  

13. **Add Supporting Nodes:**  
    - Multiple Date and time LangChain ToolCode nodes for timestamp context  
    - Calculator LangChain ToolCode node for calculations  
    - Multiple OpenAI Chat Model nodes configured with correct credentials and models for each agent node  

14. **Connect all nodes as per above described flow with proper input/output links.**  

15. **Set proper credentials:**  
    - WhatsApp API credentials for WhatsApp nodes  
    - PostgreSQL credentials for database nodes  
    - OpenAI API key for LangChain agents and chat models  
    - Gemini OCR API key for OCR node  
    - Deepgram API key for audio transcription  

16. **Set node parameters:**  
    - Configure SQL queries and insert statements in PostgreSQL nodes  
    - Configure LangChain agent prompts and output parsers  
    - HTTP request nodes with correct URLs and auth headers  

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow was designed to handle multi-modal inputs (text, images, audio) for financial expense tracking. | Workflow demonstrates integration of WhatsApp, OpenAI, PostgreSQL, Gemini OCR, and Deepgram APIs.  |
| For OCR, Gemini API is used for image text extraction.                                                        | Ensure Gemini API access and keys are configured correctly.                                         |
| For audio transcription, Deepgram API is used.                                                                 | Deepgram API key and endpoint must be set up.                                                      |
| LangChain nodes leverage OpenAI Chat models requiring OpenAI API key.                                          | Use GPT-4 or GPT-3.5 models depending on availability and cost.                                    |
| PostgreSQL database stores users and transactions; schema should include tables for users and financial records.| Database access and table schema must be predefined with proper indexes for performance.            |
| WhatsApp integration requires webhook and media handling setup per WhatsApp API documentation.                 | Webhook IDs and media URLs need to be correctly configured in WhatsApp Business API settings.       |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow. It respects all applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.