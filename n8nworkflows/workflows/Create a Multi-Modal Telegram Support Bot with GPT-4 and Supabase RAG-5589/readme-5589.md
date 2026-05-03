Create a Multi-Modal Telegram Support Bot with GPT-4 and Supabase RAG

https://n8nworkflows.xyz/workflows/create-a-multi-modal-telegram-support-bot-with-gpt-4-and-supabase-rag-5589


# Create a Multi-Modal Telegram Support Bot with GPT-4 and Supabase RAG

### 1. Workflow Overview

This workflow implements a multi-modal Telegram AI support chatbot that integrates GPT-4 with a Supabase-based Retrieval-Augmented Generation (RAG) system. It handles diverse input types received via Telegram—including text, voice messages, images, and multiple document formats—and processes them to provide intelligent, context-aware responses grounded in uploaded knowledge base documents.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Routing:** Captures Telegram user messages and routes them based on input type (text, audio, photo, document).
- **1.2 Media Download and Conversion:** Downloads media files from Telegram, converts voice to text, images to textual descriptions, and documents to text.
- **1.3 Document Validation and Categorization:** Validates file extensions, groups documents by type, and routes them accordingly.
- **1.4 Text Extraction from Documents:** Extracts text from various document types (PDF, Word, Spreadsheet, JSON, XML).
- **1.5 Knowledge Base Data Loading:** Downloads and processes knowledge base documents from Google Drive into Supabase vector store embeddings.
- **1.6 AI Processing and Memory:** Embeds input text, queries Supabase for relevant content, manages conversational memory, and generates AI responses.
- **1.7 Response Delivery:** Sends generated responses back to the Telegram user.
- **1.8 Utility and Control Nodes:** Includes manual triggers, typing indicators, and sticky notes for documentation and usage guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Routing

- **Overview:**  
  Captures incoming Telegram messages and routes them according to their content type (text, voice, photo, document).

- **Nodes Involved:**  
  - Telegram Trigger  
  - Typing…  
  - Input Message Router (Switch)  

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Listens for new Telegram messages (only "message" updates)  
    - Config: Uses Telegram API credentials; webhook enabled  
    - Input: Telegram message updates  
    - Output: JSON message object with message data  
    - Edge cases: Telegram API auth failure, webhook misconfiguration  
  - **Typing…**  
    - Type: Telegram node  
    - Role: Sends "typing" chat action to indicate bot is processing  
    - Config: Uses chat ID from incoming message  
    - Input: Chat ID  
    - Output: None (side-effect node)  
    - Edge cases: Telegram API rate limits or chat action failures  
  - **Input Message Router**  
    - Type: Switch node  
    - Role: Routes message based on whether it contains text, voice message, photo array, or document  
    - Conditions: Checks existence of fields `message.text`, `message.voice`, `message.photo`, `message.document`  
    - Input: Telegram Trigger output  
    - Output: Branches for Text, Audio, Photo, Document  
    - Edge cases: Unexpected or unsupported message types, missing fields  

#### 1.2 Media Download and Conversion

- **Overview:**  
  Downloads media files from Telegram servers, converts voice messages to text, and analyzes images to produce textual descriptions.

- **Nodes Involved:**  
  - Download Audio  
  - Translate to Text  
  - Download Photo  
  - Fix mimeType1  
  - Photo to text  
  - Download Image  
  - Fix mimeType  
  - Photo to text1  

- **Node Details:**  
  - **Download Audio**  
    - Type: Telegram node  
    - Role: Downloads voice message audio file via file_id  
    - Config: Uses Telegram credentials, obtains `message.voice.file_id`  
    - Output: Binary audio file  
    - Edge cases: File not found, API limits, unsupported audio formats  
  - **Translate to Text**  
    - Type: OpenAI Audio Translation node  
    - Role: Translates audio file to text (speech-to-text)  
    - Config: Uses OpenAI API, operation "translate" on audio resource  
    - Input: Binary audio from Download Audio  
    - Output: Transcribed text  
    - Edge cases: Audio quality issues, OpenAI API errors  
  - **Download Photo**  
    - Type: Telegram node  
    - Role: Downloads photo file (first photo in array)  
    - Config: Uses file_id from `message.photo[0].file_id`  
    - Output: Binary image file  
    - Edge cases: Missing photo data, API errors  
  - **Fix mimeType1**  
    - Type: Code node  
    - Role: Sets correct mime types on downloaded images based on file extension  
    - Logic: Checks fileName suffix (.png, .jpg, .jpeg, .webp, .gif) and sets mimeType accordingly  
    - Input: Binary data from Download Photo  
    - Output: Binary data with corrected mimeType  
    - Edge cases: Missing fileName, unexpected extensions  
  - **Photo to text**  
    - Type: OpenAI Image Analysis node  
    - Role: Uses GPT-4o-mini model to generate textual description of image content  
    - Input: Base64-encoded image from Fix mimeType1  
    - Output: Text description of image content  
    - Edge cases: OpenAI image analysis API errors, large image sizes  
  - **Download Image**  
    - Type: Telegram node  
    - Role: Downloads image file from document attachment (not photo)  
    - Config: Uses `message.document.file_id`  
    - Output: Binary image data  
    - Edge cases: Unsupported document as image, file not found  
  - **Fix mimeType**  
    - Type: Code node  
    - Role: Fixes mimeType for downloaded images from documents  
    - Logic: Checks file extension and sets mimeType (png, jpg/jpeg, webp)  
    - Input: Binary from Download Image  
    - Output: Binary with corrected mimeType  
  - **Photo to text1**  
    - Type: OpenAI Image Analysis node  
    - Role: Similar to Photo to text, analyzes image and generates content description  
    - Input: Base64 image from Fix mimeType  
    - Output: Text description  
    - Edge cases: Same as Photo to text  

#### 1.3 Document Validation and Categorization

- **Overview:**  
  Validates uploaded document file extensions against supported list and categorizes documents by type for further routing.

- **Nodes Involved:**  
  - Supported Document File Types  
  - If  
  - Group Similar Documents  
  - Document Router  

- **Node Details:**  
  - **Supported Document File Types**  
    - Type: Code node  
    - Role: Checks if document file extension is in supported list (jpg, jpeg, png, webp, pdf, doc, docx, xls, xlsx, json, xml)  
    - Output: Adds `is_supported` boolean and `reason` string to JSON  
    - Edge cases: Missing fileName, unsupported extensions  
  - **If**  
    - Type: If node  
    - Role: Branches flow based on file support status (`is_supported` and `reason`)  
    - True: Continue processing supported files  
    - False: Exit early with error message about unsupported formats  
  - **Group Similar Documents**  
    - Type: Code node  
    - Role: Parses file extension, assigns categorical type (image, pdf, word document, spreadsheet, json, xml file)  
    - Output: Adds `fileTypeCategory` field to JSON  
  - **Document Router**  
    - Type: Switch node  
    - Role: Routes documents by `fileTypeCategory` to dedicated download and extraction flows  
    - Branches: Image, PDF, Word Document, Spreadsheet, JSON, XML File  
    - Edge cases: Unknown or misclassified document types  

#### 1.4 Text Extraction from Documents

- **Overview:**  
  Extracts textual content from various document types using dedicated nodes and external services.

- **Nodes Involved:**  
  - Download PDF  
  - Extract from PDF  
  - Download Word Document  
  - File to Base64  
  - Convert to text (convertapi.com)  
  - Download Text  
  - Extract Text5  
  - Download Spreadsheet  
  - Extract from Spreadsheet  
  - Download JSON  
  - Extract from JSON  
  - Download XML  
  - Extract from XML  
  - Extract Text1, Extract Text2, Extract Text3, Extract Text4  

- **Node Details:**  
  - **Download PDF / Download Word Document / Download Spreadsheet / Download JSON / Download XML**  
    - Type: Telegram nodes  
    - Role: Download respective file types from Telegram using `file_id`  
    - Output: Binary file data  
    - Edge cases: File not found, API limits  
  - **Extract from PDF / Extract from Spreadsheet / Extract from JSON / Extract from XML**  
    - Type: ExtractFromFile nodes  
    - Role: Extract text or structured data from binary files  
    - Config: Operation set to respective file type (pdf, xlsx, fromJson, xml)  
    - Output: Extracted text or JSON data  
  - **File to Base64**  
    - Type: ExtractFromFile node  
    - Role: Converts binary Word document to Base64 string for API consumption  
  - **Convert to text (convertapi.com)**  
    - Type: HTTP Request node  
    - Role: Sends Base64 Word document to ConvertAPI to convert docx to plain text  
    - Config: Requires ConvertAPI key in header, posts JSON with file info  
    - Output: JSON with URL to converted text file  
    - Edge cases: API key missing, request failure  
  - **Download Text**  
    - Type: HTTP Request node  
    - Role: Downloads converted text file from ConvertAPI URL  
  - **Extract Text1 to Extract Text5**  
    - Type: Set nodes  
    - Role: Assign extracted or converted text to `text` property for downstream AI processing  
    - Input: Various extracted or API response JSONs  
    - Output: JSON containing plain text  
    - Edge cases: Missing or malformed extraction data  

#### 1.5 Knowledge Base Data Loading

- **Overview:**  
  Imports knowledge base documents from Google Drive, processes them into vector embeddings, and stores them in Supabase for retrieval.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Download file (Google Drive)  
  - Default Data Loader1  
  - Recursive Character Text Splitter1  
  - Embeddings OpenAI  
  - Add to Supabase Vector DB  

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the import process manually  
  - **Download file**  
    - Type: Google Drive node  
    - Role: Downloads knowledge base file using fileId (Google Drive document)  
    - Credentials: Google Drive OAuth2 API  
    - Edge cases: Access permissions, expired tokens  
  - **Default Data Loader1**  
    - Type: Document Default Data Loader (Langchain)  
    - Role: Loads and prepares document data (from binary) for splitting  
  - **Recursive Character Text Splitter1**  
    - Type: Text Splitter  
    - Role: Splits loaded document text recursively for embedding chunks  
  - **Embeddings OpenAI**  
    - Type: OpenAI embeddings node  
    - Role: Converts text chunks to vector embeddings  
    - Credentials: OpenAI API  
  - **Add to Supabase Vector DB**  
    - Type: Supabase Vector Store node  
    - Role: Inserts vector embeddings into Supabase table for retrieval  
    - Credentials: Supabase API  
    - Config: Table `documents`  

#### 1.6 AI Processing and Memory

- **Overview:**  
  Processes user input text through an AI agent enhanced with conversational memory and knowledge base context retrieved from Supabase.

- **Nodes Involved:**  
  - Extract Text (and Extract Text1-5, Extract Text & Content, Extract Error Message)  
  - Knowledge Base AI Agent  
  - Supabase Vector Store Search  
  - Reranker Cohere  
  - Embeddings OpenAI1  
  - Postgres Chat Memory  
  - OpenAI Chat Model1  

- **Node Details:**  
  - **Extract Text** (and similar nodes)  
    - Type: Set nodes  
    - Role: Normalize and assign text extracted from various inputs to `text` field  
  - **Knowledge Base AI Agent**  
    - Type: Langchain Agent node  
    - Role: Main AI chatbot logic; takes input text, context, memory, and tools (vector search) to produce response  
    - Config: GPT-4.1-mini model, no special system message set  
    - Inputs: Text, vector search tool, memory  
    - Output: Response text  
    - Edge cases: API limits, malformed prompt, memory inconsistencies  
  - **Supabase Vector Store Search**  
    - Type: Langchain Vector Store node  
    - Role: Queries Supabase vector DB for top-k semantically similar document chunks relevant to input  
    - Config: topK=10, reranker enabled using Cohere  
    - Credentials: Supabase API  
  - **Reranker Cohere**  
    - Type: Reranker node  
    - Role: Refines retrieved vector search results to improve relevance  
    - Credentials: Cohere API  
  - **Embeddings OpenAI1**  
    - Role: Produces embeddings for query text as part of search tool input  
  - **Postgres Chat Memory**  
    - Type: Langchain memory node using Postgres  
    - Role: Stores recent chat history (contextWindowLength=3) for conversational context  
    - Credentials: Postgres  
  - **OpenAI Chat Model1**  
    - Type: Langchain OpenAI Chat model  
    - Role: Provides language generation for AI agent internally  
    - Credentials: OpenAI API  

#### 1.7 Response Delivery

- **Overview:**  
  Sends the AI-generated response back to the user on Telegram.

- **Nodes Involved:**  
  - Telegram  

- **Node Details:**  
  - **Telegram**  
    - Type: Telegram node  
    - Role: Sends message text to Telegram chat ID extracted from original message  
    - Config: Uses output `output` from AI Agent node as message text  
    - Credentials: Telegram API  
    - Edge cases: Rate limits, chat ID invalidation  

#### 1.8 Utility and Control Nodes

- **Overview:**  
  Includes workflow control and documentation nodes to facilitate deployment and understanding.

- **Nodes Involved:**  
  - Sticky Note1, Sticky Note4, Sticky Note5  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Typing… (also part of 1.1)  

- **Node Details:**  
  - Sticky notes provide detailed usage instructions, credential requirements, workflow explanation, and operational notes.  
  - Manual trigger enables manual import of knowledge base documents.

---

### 3. Summary Table

| Node Name                    | Node Type                      | Functional Role                          | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                                     |
|------------------------------|--------------------------------|----------------------------------------|-----------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger               | Captures incoming Telegram messages    |                             | Input Message Router, Typing…   |                                                                                                                                |
| Typing…                     | Telegram                      | Sends "typing" indicator to Telegram   | Telegram Trigger             |                                 |                                                                                                                                |
| Input Message Router        | Switch                       | Routes message by input type            | Telegram Trigger             | Extract Text, Download Audio, Download Photo, Supported Document File Types |                                                                                                                                |
| Download Audio              | Telegram                      | Downloads voice message audio           | Input Message Router (Audio) | Translate to Text               |                                                                                                                                |
| Translate to Text           | OpenAI Audio Translation      | Converts voice audio to text            | Download Audio               | Knowledge Base AI Agent         |                                                                                                                                |
| Download Photo              | Telegram                      | Downloads photo file                    | Input Message Router (Photo) | Fix mimeType1                  |                                                                                                                                |
| Fix mimeType1               | Code                         | Fixes mimeType for photo binary         | Download Photo               | Photo to text                  |                                                                                                                                |
| Photo to text               | OpenAI Image Analysis         | Analyzes image content to text          | Fix mimeType1                | Extract Text & Content          |                                                                                                                                |
| Download Image              | Telegram                      | Downloads image file from document      | Document Router (Image)      | Fix mimeType                   |                                                                                                                                |
| Fix mimeType                | Code                         | Fixes mimeType for image binary         | Download Image               | Photo to text1                 |                                                                                                                                |
| Photo to text1              | OpenAI Image Analysis         | Analyzes image content to text          | Fix mimeType                 | Extract Text & Content          |                                                                                                                                |
| Supported Document File Types| Code                         | Validates supported file extensions     | Input Message Router (Document) | If                          |                                                                                                                                |
| If                         | If                           | Branches on file support status          | Supported Document File Types | Group Similar Documents, Extract Error Message |                                                                                                                                |
| Group Similar Documents     | Code                         | Categorizes documents by type            | If (True)                   | Document Router                |                                                                                                                                |
| Document Router             | Switch                       | Routes documents by category             | Group Similar Documents      | Download Image, Download PDF, Download Word Document, Download Spreadsheet, Download JSON, Download XML |                                                                                                                                |
| Download PDF                | Telegram                      | Downloads PDF file                      | Document Router (PDF)        | Extract from PDF              |                                                                                                                                |
| Extract from PDF            | ExtractFromFile              | Extracts text from PDF                   | Download PDF                 | Extract Text4                 |                                                                                                                                |
| Extract Text4               | Set                          | Assigns extracted PDF text to `text`    | Extract from PDF             | Knowledge Base AI Agent        |                                                                                                                                |
| Download Word Document      | Telegram                      | Downloads Word document                  | Document Router (Word Document) | File to Base64             |                                                                                                                                |
| File to Base64              | ExtractFromFile              | Converts Word doc binary to Base64       | Download Word Document       | Convert to text (convertapi.com) |                                                                                                                                |
| Convert to text (convertapi.com) | HTTP Request               | Converts Word docx to text via API      | File to Base64               | Download Text                 |                                                                                                                                |
| Download Text              | HTTP Request                 | Downloads converted text file            | Convert to text              | Extract Text5                 |                                                                                                                                |
| Extract Text5              | Set                          | Assigns converted Word text to `text`   | Download Text                | Knowledge Base AI Agent        |                                                                                                                                |
| Download Spreadsheet        | Telegram                      | Downloads spreadsheet file               | Document Router (Spreadsheet) | Extract from Spreadsheet      |                                                                                                                                |
| Extract from Spreadsheet    | ExtractFromFile              | Extracts text from spreadsheet           | Download Spreadsheet         | Extract Text1                 |                                                                                                                                |
| Extract Text1              | Set                          | Assigns spreadsheet text to `text`      | Extract from Spreadsheet     | Knowledge Base AI Agent        |                                                                                                                                |
| Download JSON              | Telegram                      | Downloads JSON file                     | Document Router (JSON)       | Extract from JSON             |                                                                                                                                |
| Extract from JSON          | ExtractFromFile              | Parses JSON file                        | Download JSON                | Extract Text2                 |                                                                                                                                |
| Extract Text2              | Set                          | Assigns JSON text to `text`             | Extract from JSON            | Knowledge Base AI Agent        |                                                                                                                                |
| Download XML               | Telegram                      | Downloads XML file                      | Document Router (XML File)   | Extract from XML              |                                                                                                                                |
| Extract from XML           | ExtractFromFile              | Parses XML file                        | Download XML                 | Extract Text3                 |                                                                                                                                |
| Extract Text3              | Set                          | Assigns XML text to `text`              | Extract from XML             | Knowledge Base AI Agent        |                                                                                                                                |
| Extract Error Message      | Set                          | Assigns error message text               | If (False)                  | Knowledge Base AI Agent        |                                                                                                                                |
| Extract Text               | Set                          | Assigns text from text messages          | Input Message Router (Text)  | Knowledge Base AI Agent        |                                                                                                                                |
| Extract Text & Content     | Set                          | Combines photo analysis and caption      | Photo to text, Photo to text1 | Knowledge Base AI Agent        |                                                                                                                                |
| Knowledge Base AI Agent    | Langchain Agent              | Main AI processing and response generation | Multiple Extract Text nodes, Supabase Vector Store Search, Postgres Chat Memory, OpenAI Chat Model1 | Telegram                  |                                                                                                                                |
| Supabase Vector Store Search| Langchain Vector Store      | Retrieves relevant document chunks       | Reranker Cohere, Embeddings OpenAI1 | Knowledge Base AI Agent        |                                                                                                                                |
| Reranker Cohere            | Langchain Reranker          | Improves semantic search relevance       | Supabase Vector Store Search | Supabase Vector Store Search   |                                                                                                                                |
| Embeddings OpenAI1         | Langchain Embeddings        | Embeds query text for vector search      |                             | Supabase Vector Store Search   |                                                                                                                                |
| Postgres Chat Memory       | Langchain Memory            | Maintains conversational context         |                             | Knowledge Base AI Agent        |                                                                                                                                |
| OpenAI Chat Model1         | Langchain Language Model    | Provides GPT-4 based language generation |                             | Knowledge Base AI Agent        |                                                                                                                                |
| Telegram                   | Telegram                    | Sends AI response message to user        | Knowledge Base AI Agent      |                             |                                                                                                                                |
| When clicking ‘Execute workflow’ | Manual Trigger            | Starts knowledge base import workflow     |                             | Download file                 |                                                                                                                                |
| Download file              | Google Drive                | Downloads knowledge base document         | When clicking ‘Execute workflow’ | Default Data Loader1       |                                                                                                                                |
| Default Data Loader1       | Langchain Data Loader       | Loads knowledge base document binary      | Download file                | Recursive Character Text Splitter1 |                                                                                                                                |
| Recursive Character Text Splitter1 | Langchain Text Splitter    | Splits document text into chunks          | Default Data Loader1         | Embeddings OpenAI             |                                                                                                                                |
| Embeddings OpenAI          | Langchain Embeddings        | Embeds document chunks for vector DB      | Recursive Character Text Splitter1 | Add to Supabase Vector DB   |                                                                                                                                |
| Add to Supabase Vector DB  | Langchain Vector Store      | Inserts embeddings into Supabase vector DB | Embeddings OpenAI, Download file |                             |                                                                                                                                |
| Fix mimeType               | Code                        | Fixes mimeType for images from documents  | Download Image               | Photo to text1                |                                                                                                                                |
| Fix mimeType1              | Code                        | Fixes mimeType for photos                   | Download Photo               | Photo to text                 |                                                                                                                                |
| Sticky Note1               | Sticky Note                 | Instructions for manual knowledge base import |                             |                             | "Before using the Telegram chatbot: Manually run this workflow to import your knowledge base from Google Drive into Supabase with vector embeddings." |
| Sticky Note4               | Sticky Note                 | Lists required credentials                  |                             |                             | "1. Google Drive API Key; 2. Telegram Bot Token; 3. OpenAI API Key; 4. Supabase API Key + Environment; 5. ConvertAPI Key; 6. Cohere API Key; 7. Postgres API Key or n8n memory node." |
| Sticky Note5               | Sticky Note                 | Detailed workflow explanation and routing |                             |                             | "How it works: Message routing, document type routing, supported file types, RAG via Supabase, response delivery." Includes supported formats and link references. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure credentials with Telegram Bot API token  
   - Set to receive updates on "message" only  

2. **Add Typing… node**  
   - Type: Telegram  
   - Set chatId to `{{$json.message.chat.id}}` from Telegram Trigger  
   - Operation: sendChatAction  

3. **Add Input Message Router (Switch) node**  
   - Route based on message content existence:  
     - Text: `$json.message.text` exists  
     - Audio: `$json.message.voice` exists  
     - Photo: `$json.message.photo` exists  
     - Document: `$json.message.document` exists  

4. **For Text branch:**  
   - Add a Set node to assign `text` = `$json.message.text`  
   - Connect to Knowledge Base AI Agent (step 17)  

5. **For Audio branch:**  
   - Add Telegram Download Audio node  
     - fileId: `$json.message.voice.file_id`  
   - Add OpenAI Audio Translation node  
     - Operation: translate (speech to text)  
     - Use OpenAI API credentials  
   - Connect to Knowledge Base AI Agent  

6. **For Photo branch:**  
   - Add Telegram Download Photo node  
     - fileId: `$json.message.photo[0].file_id`  
   - Add Code node "Fix mimeType1" to set mime type based on file extension (png, jpg, jpeg, webp, gif)  
   - Add OpenAI Image Analysis node ("Photo to text")  
     - Model: GPT-4o-mini  
     - Operation: analyze  
     - InputType: base64 from previous node  
   - Add Set node "Extract Text & Content" to format description and caption into `text`  
   - Connect to Knowledge Base AI Agent  

7. **For Document branch:**  
   - Add Code node "Supported Document File Types"  
     - Check file extension against supported list (jpg, jpeg, png, webp, pdf, doc, docx, xls, xlsx, json, xml)  
   - Add If node to branch on support status  
     - If unsupported, set error message and connect to AI Agent for error reply  
     - If supported, continue  
   - Add Code node "Group Similar Documents" to assign `fileTypeCategory` based on extension  

8. **Add Document Router (Switch)**  
   - Route by `fileTypeCategory`: image, pdf, word document, spreadsheet, json, xml file  

9. **For Image documents:**  
   - Download file via Telegram node  
   - Fix mimeType via Code node  
   - Analyze image with OpenAI Image Analysis node  
   - Extract text & content with Set node  
   - Connect to AI Agent  

10. **For PDF documents:**  
    - Download PDF via Telegram node  
    - Extract text with ExtractFromFile node (operation: pdf)  
    - Assign text with Set node  
    - Connect to AI Agent  

11. **For Word documents:**  
    - Download Word document via Telegram node  
    - Convert binary to Base64  
    - Use HTTP Request node to ConvertAPI for docx to txt conversion (include API key in headers)  
    - Download converted text  
    - Assign text with Set node  
    - Connect to AI Agent  

12. **For Spreadsheet documents:**  
    - Download spreadsheet via Telegram node  
    - Extract text with ExtractFromFile node (operation: xlsx)  
    - Assign text with Set node  
    - Connect to AI Agent  

13. **For JSON documents:**  
    - Download JSON file  
    - Extract JSON content with ExtractFromFile (operation: fromJson)  
    - Assign text with Set node  
    - Connect to AI Agent  

14. **For XML documents:**  
    - Download XML file  
    - Extract XML content with ExtractFromFile (operation: xml)  
    - Assign text with Set node  
    - Connect to AI Agent  

15. **Knowledge Base Import Workflow:**  
    - Create Manual Trigger node  
    - Add Google Drive node to download knowledge base document using fileId and Google Drive OAuth2 credentials  
    - Add Langchain Document Default Data Loader node to load document binary  
    - Add Recursive Character Text Splitter node to split text  
    - Add OpenAI Embeddings node to create vector embeddings from text chunks  
    - Add Supabase Vector Store node to insert embeddings into Supabase table "documents"  
    - Configure Supabase API credentials  

16. **AI Processing:**  
    - Add Langchain Supabase Vector Store Search node  
      - Configure to retrieve top 10 relevant chunks with reranker enabled using Cohere  
      - Provide Supabase credentials  
    - Add Langchain Reranker Cohere node with Cohere API credentials  
    - Add Langchain Postgres Chat Memory node for session-based memory (session key from Telegram chat id)  
    - Add Langchain OpenAI Chat Model node (GPT-4.1-mini) using OpenAI credentials  
    - Add Langchain Agent node  
      - Inputs: extracted text, vector store search as tool, chat memory, language model  
      - Outputs: AI response text  

17. **Response Delivery:**  
    - Add Telegram node to send message back to user  
    - Use chatId from Telegram Trigger and response text from AI Agent  

18. **Add Typing… node after Telegram Trigger to show typing during processing**  

19. **Add Sticky Notes for documentation:**  
    - Credential requirements  
    - Workflow explanation and routing  
    - Instructions for manual knowledge base import  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Telegram Bot Token setup guide: [Telegram Credential Setup](https://docs.n8n.io/integrations/builtin/credentials/telegram/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal) | Telegram API credential configuration for the bot                                                                |
| Workflow supports multi-modal input: text, voice (speech-to-text), image (image analysis), and document parsing for enhanced chatbot interactions.                             | Functional scope                                                                                                |
| RAG (Retrieval-Augmented Generation) approach uses Supabase vector DB with OpenAI embeddings and Cohere reranker for grounded AI responses.                                    | AI knowledge base integration                                                                                     |
| ConvertAPI is used to convert Word documents to text, requiring an API key.                                                                                                     | External service dependency                                                                                        |
| Postgres is used for chat memory storage to maintain conversational context across user sessions.                                                                              | Memory persistence                                                                                                 |
| Google Drive API key is needed to manually import knowledge base documents into Supabase vector store.                                                                          | Knowledge base management                                                                                          |
| Supports a broad range of document types: images (.jpg, .png, .webp), PDFs, Word docs, spreadsheets, JSON, XML.                                                                 | Input flexibility                                                                                                |
| Recommended to manually import knowledge base documents before using the chatbot to ensure data is embedded and searchable.                                                    | Operational recommendation                                                                                        |

---

This structured analysis and reproduction guide should enable advanced users and AI agents to fully understand, replicate, and maintain the multi-modal Telegram AI support chatbot workflow integrating GPT-4 and Supabase RAG.