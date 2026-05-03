Build a WhatsApp Assistant with Memory, Google Suite & Multi-AI Research and Imaging

https://n8nworkflows.xyz/workflows/build-a-whatsapp-assistant-with-memory--google-suite---multi-ai-research-and-imaging-7663


# Build a WhatsApp Assistant with Memory, Google Suite & Multi-AI Research and Imaging

### 1. Workflow Overview

This workflow transforms a WhatsApp chat interface into a versatile AI assistant with memory and multi-AI capabilities. It handles inputs via WhatsApp messages (text or images), processes them with AI agents, and integrates with various productivity tools and external APIs for comprehensive user assistance. Core capabilities include understanding natural conversation, image analysis and generation, web research, calendar and task management, and persistent memory through vector databases.

The workflow is logically divided into these main blocks:

- **1.1 WhatsApp Input Reception & Preprocessing**  
  Captures incoming WhatsApp messages, distinguishes between text and media, downloads and analyzes images if present, and prepares a unified input for AI processing.

- **1.2 Core AI Agent Processing & Tool Orchestration**  
  Uses an advanced AI agent with embedded memory to interpret user messages, decide which external tools or sub-agents to invoke, and generate responses.

- **1.3 Productivity Tools Integration**  
  Contains sub-agents and nodes for Google Calendar, Google Tasks, and Gmail to manage scheduling, task lists, and email retrieval.

- **1.4 Research & Web Search Agent**  
  Handles factual, news, and in-depth research queries using multiple search engines, including Brave Search, Wikipedia, Perplexcia, and Tavily.

- **1.5 Image Generation & Analysis**  
  Processes user requests for images, generating prompts, calling an external image generation API, and returning images to the user.

- **1.6 Memory Management & Persistence**  
  Extracts important conversational information to store in a MongoDB Atlas vector store for long-term memory and context retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 WhatsApp Input Reception & Preprocessing

**Overview:**  
Listens for incoming WhatsApp messages (text or media), routes based on content type, downloads and analyzes images, and prepares a standardized input for AI processing.

**Nodes Involved:**  
- WhatsApp Trigger  
- Switch  
- Download media  
- HTTP Request5  
- Analyze image  
- Code1  
- Typing....

**Node Details:**

- **WhatsApp Trigger**  
  - Type: Event trigger for WhatsApp messages  
  - Configuration: Listens for message updates, connected via WhatsApp OAuth credentials  
  - Output: Raw WhatsApp message JSON  
  - Failure modes: Auth errors, webhook failures, message format variations

- **Switch**  
  - Type: Conditional routing  
  - Configuration: Checks if message contains an image or text using existence conditions  
  - Routes: "image" path if image present, "chat" path if text present  
  - Failure modes: Unexpected message formats may not match any condition

- **Download media**  
  - Type: WhatsApp media download  
  - Configuration: Downloads media by media ID from incoming message  
  - On error: continue without stopping workflow  
  - Failure modes: Network errors, invalid media ID

- **HTTP Request5**  
  - Type: HTTP request to WhatsApp API (for media retrieval)  
  - Configuration: Uses WhatsApp API credentials and Bearer Auth to download media content from URL  
  - Failure modes: Auth errors, network timeouts

- **Analyze image**  
  - Type: Google Gemini image analysis  
  - Configuration: Uses Google Palm API with Gemini-2.0-flash model to analyze image binary data  
  - Input: Binary image data from HTTP Request5  
  - Failure modes: API quota, analysis errors

- **Code1**  
  - Type: JavaScript code node  
  - Role: Consolidates user input from either image analysis result or text message into a single field called `input`, sets a source flag  
  - Throws error if no valid input found  
  - Failure modes: Expression errors, missing data fields

- **Typing....**  
  - Type: HTTP Request  
  - Role: Sends 'typing' indicator back to user on WhatsApp for UX  
  - Failure modes: API errors, rate limits

---

#### 1.2 Core AI Agent Processing & Tool Orchestration

**Overview:**  
Processes the unified user input to generate AI responses. Maintains a simple short-term memory buffer and orchestrates calls to external tools and sub-agents based on user intent.

**Nodes Involved:**  
- AI Agent1  
- Simple Memory  
- MongoDB Atlas Vector Store2  
- Gmail tool (agent tool)  
- Calendar Agent (agent tool)  
- Task agent (agent tool)  
- WeatherMap (OpenWeatherMap tool)  
- Research agent (agent tool)  
- image creation (HTTP request tool)  
- Send message

**Node Details:**

- **AI Agent1**  
  - Type: LangChain agent node with GPT and tool integration  
  - Configuration: Custom system prompt to behave as a friendly assistant, strict formatting rules, retries on fail enabled  
  - Uses embedded Simple Memory and vector store memory (MongoDB Atlas Vector Store2) as context  
  - Decides which sub-agent or tool to invoke (Calendar, Task, Gmail, Research, WeatherMap, image creation) based on user input  
  - Failure modes: AI model errors, tool failures, memory retrieval failures

- **Simple Memory**  
  - Type: LangChain memory buffer node  
  - Configuration: Uses session key "whatsapp" for session-based short-term memory  
  - Provides context to AI Agent1  
  - Failure modes: Session key misconfiguration, memory overflow

- **MongoDB Atlas Vector Store2**  
  - Type: Vector store retrieval tool  
  - Configuration: Retrieves top 5 most relevant memory documents for context  
  - Metadata fields: info, text, summary  
  - Failure modes: DB connectivity, query errors

- **Gmail tool**  
  - Type: LangChain agent tool for Gmail  
  - Configuration: Retrieves recent emails on demand  
  - Credentials: OAuth2 Gmail  
  - Failure modes: Auth issues, API limits

- **Calendar Agent**  
  - Type: LangChain agent tool for Google Calendar  
  - Configuration: Supports create, get, update, delete events with detailed instructions in system message  
  - Credentials: Google Calendar OAuth2  
  - Failure modes: Permission errors, calendar ID issues

- **Task agent**  
  - Type: LangChain agent tool for Google Tasks  
  - Configuration: Manages tasks and task lists with CRUD operations  
  - Failure modes: OAuth issues, task ID retrieval failures

- **WeatherMap**  
  - Type: OpenWeatherMap tool  
  - Configuration: Retrieves 5-day weather forecast for Bengaluru (default city)  
  - Failure modes: API key issues, network errors

- **Research agent**  
  - Type: LangChain agent tool for research  
  - Configuration: Uses multiple search tools (Brave Search, Wikipedia, Perplexcia, Tavily) with detailed system instructions on usage, query delays, and execution order  
  - Failure modes: API call frequency limits, tool outages

- **image creation**  
  - Type: HTTP Request tool  
  - Configuration: Sends user message to external image generation webhook  
  - Failure modes: API errors, invalid prompt issues

- **Send message**  
  - Type: WhatsApp send message node  
  - Configuration: Sends the AI Agent1's generated response back to the user  
  - Failure modes: WhatsApp API errors, number formatting issues

---

#### 1.3 Productivity Tools Integration

**Overview:**  
This block connects AI agents with Google Calendar, Google Tasks, and Gmail nodes to enable scheduling, task management, and email retrieval seamlessly within the assistant.

**Nodes Involved:**  
- Calendar Agent  
- Get many events  
- Create an event  
- Update an event  
- Delete an event  
- Get an event  
- Task agent  
- Create a task in Google Tasks  
- Delete a task in Google Tasks  
- Get many tasks in Google Tasks  
- Gmail tool  
- Get many messages  
- Get a message

**Node Details:**

- **Calendar Agent**  
  - See Block 1.2 for details

- **Get many events**  
  - Type: Google Calendar tool (get all events)  
  - Configuration: Retrieves events from a specific Google calendar (Reminders calendar)  
  - Parameters: TimeMax configurable via AI prompt  
  - Failure modes: Calendar access issues

- **Create an event**  
  - Type: Google Calendar tool (create event)  
  - Configuration: Creates event with start, end, and summary fields from AI prompt data  
  - Failure modes: Invalid time formats

- **Update an event**  
  - Type: Google Calendar tool (update event)  
  - Configuration: Updates event by event ID with fields from AI prompt  
  - Failure modes: Event ID missing or invalid

- **Delete an event**  
  - Type: Google Calendar tool (delete event)  
  - Configuration: Deletes event by event ID  
  - Failure modes: Event ID missing

- **Get an event**  
  - Type: Google Calendar tool (get single event)  
  - Configuration: Retrieves event by event ID  
  - Failure modes: Event ID missing

- **Task agent**  
  - See Block 1.2 for details

- **Create a task in Google Tasks**  
  - Type: Google Tasks tool (create task)  
  - Configuration: Creates a task in a fixed task list with title from AI prompt  
  - Failure modes: Task list ID errors

- **Delete a task in Google Tasks**  
  - Type: Google Tasks tool (delete task)  
  - Configuration: Deletes task by task ID  
  - Failure modes: Task ID missing

- **Get many tasks in Google Tasks**  
  - Type: Google Tasks tool (get all tasks)  
  - Configuration: Retrieves tasks from fixed task list with optional due date filters  
  - Failure modes: API limits

- **Gmail tool**  
  - See Block 1.2 for details

- **Get many messages**  
  - Type: Gmail tool (get all emails)  
  - Configuration: Retrieves recent emails with optional filters  
  - Failure modes: Gmail API limits

- **Get a message**  
  - Type: Gmail tool (get single email)  
  - Configuration: Gets a specific email by message ID  
  - Failure modes: Message ID missing

---

#### 1.4 Research & Web Search Agent

**Overview:**  
Handles research-related queries by querying multiple search engines and knowledge bases, merging results for comprehensive answers.

**Nodes Involved:**  
- Research agent  
- Brave Search  
- Brave Search1 (news)  
- perprlexcia (Perplexcia)  
- Wikipedia  
- Tavily web search  
- gpt-4.1-nanoChat Model1  
- openai/gpt-oss-120b  
- gemini-2.5-flash  
- gpt-5-nano

**Node Details:**

- **Research agent**  
  - Coordinates search across Brave Search (web and news), Wikipedia, Perplexcia, and Tavily  
  - Enforces 1-second delay between Brave web and news queries  
  - Uses system message to define execution order and tool strengths  
  - Failure modes: API rate limits, network errors

- **Brave Search & Brave Search1**  
  - Type: Brave Search API nodes  
  - Configuration: One for general web search, one for news  
  - Failure modes: API key issues

- **perprlexcia**  
  - Type: HTTP Request tool to Perplexcia API  
  - Configuration: POST request with query and focus mode "webSearch"  
  - Failure modes: Host availability, auth errors

- **Wikipedia**  
  - Type: LangChain Wikipedia tool  
  - Configuration: Used for quick factual lookups  
  - Failure modes: API errors

- **Tavily web search**  
  - Type: HTTP Request tool  
  - Configuration: POST to Tavily search API with Authorization header (requires API key)  
  - Failure modes: Missing API key, network errors

- **gpt-4.1-nanoChat Model1, openai/gpt-oss-120b, gemini-2.5-flash, gpt-5-nano**  
  - Various AI language model nodes used as sub-models for research agent or Gmail tool  
  - Failure modes: Model availability, quota

---

#### 1.5 Image Generation & Analysis

**Overview:**  
Processes user requests involving images: downloads and analyzes images sent by user, generates AI prompts for image creation, calls external image APIs, and sends images back via WhatsApp.

**Nodes Involved:**  
- Download media (also in 1.1)  
- HTTP Request5 (media fetch)  
- Analyze image  
- Code1 (input consolidation)  
- AI Agent (image creation prompt generator)  
- Webhook3  
- Clean Prompt Text1  
- HTTP Request (image generation API call)  
- Edit Fields5  
- Convert to File  
- Send message3 (WhatsApp send image)  
- Summarize Chat

**Node Details:**

- **AI Agent (image creation prompt generator)**  
  - Specialized agent that uses memory retrieval and generates detailed prompts for image creation models  
  - Uses MongoDB Atlas Vector Store1 for memory context  
  - Failure modes: No relevant memory found, API errors

- **Webhook3**  
  - Receives image creation requests triggered by user  
  - Failure modes: Webhook not reachable

- **Clean Prompt Text1**  
  - Code node that sanitizes AI-generated prompt text by removing quotes and excess whitespace before API call  
  - Failure modes: Unexpected input format

- **HTTP Request (image generation API call)**  
  - Calls external image generation API (Together.xyz) with prompt and generation parameters  
  - Requires header auth credentials  
  - Failure modes: API quota, network issues

- **Edit Fields5**  
  - Extracts base64 encoded image from response payload for conversion  
  - Failure modes: Missing or malformed response data

- **Convert to File**  
  - Converts base64 image string to binary file for sending  
  - Failure modes: Conversion errors

- **Send message3**  
  - Sends generated image back to WhatsApp user  
  - Failure modes: WhatsApp API errors

- **Summarize Chat**  
  - Generates a brief summary of the image generation interaction for memory storage  
  - Failure modes: AI model errors

---

#### 1.6 Memory Management & Persistence

**Overview:**  
Extracts critical new information from conversations, decides if it is worth remembering, and stores it in a MongoDB Atlas vector store for retrieval during future interactions, ensuring long-term context.

**Nodes Involved:**  
- Webhook2  
- Extract Memory Info  
- Check If Worth Remembering  
- Edit Fields1  
- MongoDB Atlas Vector Store  
- Google Gemini Chat Model  
- Default Data Loader  
- MongoDB Atlas Vector Store1

**Node Details:**

- **Webhook2**  
  - Receives conversation data (user messages and assistant responses) for memory processing  
  - Failure modes: Webhook downtime

- **Extract Memory Info**  
  - LangChain chain LLM node that analyzes conversation turns to extract new, important info for memory  
  - Returns concise third-person summary or "nothing" if no info to remember  
  - Failure modes: AI model failures

- **Check If Worth Remembering**  
  - If node that proceeds only if extracted info is not "nothing"  
  - Failure modes: Incorrect string matching

- **Edit Fields1**  
  - Prepares metadata and fields for inserting into MongoDB vector store, including timestamp and user ID  
  - Failure modes: Data format issues

- **MongoDB Atlas Vector Store**  
  - Inserts new memory documents into MongoDB vector store collection "document"  
  - Failure modes: DB connectivity, write permission

- **Google Gemini Chat Model**  
  - Provides language model capabilities for memory extraction and processing

- **Default Data Loader**  
  - Used for loading default documents into the vector store, possibly for initialization or testing

- **MongoDB Atlas Vector Store1**  
  - Vector store retrieval tool used by AI Agent for memory context during image creation prompt generation

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                                    | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                           |
|-----------------------------|--------------------------------------------|---------------------------------------------------|------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| WhatsApp Trigger             | WhatsApp Trigger                           | Entry point: receives WhatsApp messages           |                              | Switch                          | Entry point for WhatsApp messages and starts the workflow                                           |
| Switch                      | Switch                                    | Routes messages by type (image or text)           | WhatsApp Trigger             | Download media, Typing....       | Routes incoming messages to image or text processing                                                |
| Download media              | WhatsApp (media download)                  | Downloads media from WhatsApp messages             | Switch                      | HTTP Request5                   | Downloads images sent by user                                                                       |
| HTTP Request5               | HTTP Request                              | Retrieves media URL content                         | Download media               | Analyze image                  | Fetches media content for analysis                                                                  |
| Analyze image               | Google Gemini (image analysis)             | Analyzes downloaded image                           | HTTP Request5               | Code1                         | Performs image content analysis                                                                     |
| Code1                      | Code                                      | Consolidates input from image or text              | Analyze image, Typing....    | AI Agent1                     | Prepares unified input for AI agent                                                                |
| Typing....                 | HTTP Request                              | Sends typing indicator on WhatsApp                 | Switch                      | Code1                         | Improves UX with typing feedback                                                                    |
| AI Agent1                   | LangChain Agent                           | Core AI processing and tool orchestration          | Code1                       | Send message                  | Main AI brain that chooses tools and generates responses                                           |
| Simple Memory               | LangChain Memory                          | Maintains short-term session memory                |                              | AI Agent1                     | Provides conversation memory for AI agent                                                          |
| MongoDB Atlas Vector Store2 | Vector Store MongoDB Atlas                | Retrieves relevant memory for AI context           |                              | AI Agent1                     | Provides long-term memory retrieval                                                                 |
| Gmail tool                  | LangChain AgentTool                       | Retrieves user emails                               |                              | AI Agent1                     | Accesses Gmail for email-related queries                                                           |
| Calendar Agent              | LangChain AgentTool                       | Manages Google Calendar events                      |                              | AI Agent1                     | Calendar scheduling and event management                                                           |
| Task agent                  | LangChain AgentTool                       | Manages Google Tasks                                |                              | Calendar Agent                | Task list management                                                                                |
| WeatherMap                  | OpenWeatherMap Tool                       | Provides weather forecast                           |                              | AI Agent1                     | Weather data retrieval                                                                              |
| Research agent              | LangChain AgentTool                       | Handles multi-source web research                   |                              | AI Agent1                     | Research assistant with web and news search                                                        |
| image creation              | HTTP Request Tool                         | Sends user prompts to external image generator     |                              | AI Agent1                     | Triggers image generation                                                                           |
| Send message                | WhatsApp Send Message                     | Sends AI responses to WhatsApp users                | AI Agent1                   | send chat history to MongoDB    | Sends final chat replies                                                                            |
| send chat history to MongoDB| HTTP Request                             | Sends conversation logs to external server          | Send message                |                              | Logs conversations for further processing                                                          |
| Get many events             | Google Calendar Tool                      | Retrieves multiple calendar events                   | Calendar Agent              |                              | Fetches calendar events                                                                             |
| Create an event             | Google Calendar Tool                      | Creates new calendar event                           | Calendar Agent              |                              | Adds events to calendar                                                                             |
| Update an event             | Google Calendar Tool                      | Updates existing calendar event                      | Calendar Agent              |                              | Modifies calendar events                                                                            |
| Delete an event             | Google Calendar Tool                      | Deletes calendar event                               | Calendar Agent              |                              | Removes events from calendar                                                                        |
| Get an event                | Google Calendar Tool                      | Retrieves a specific calendar event                  | Calendar Agent              |                              | Fetches event details                                                                               |
| Create a task in Google Tasks| Google Tasks Tool                       | Creates a new task                                  | Task agent                  |                              | Adds tasks                                                                                        |
| Delete a task in Google Tasks| Google Tasks Tool                       | Deletes a task                                      | Task agent                  |                              | Removes tasks                                                                                      |
| Get many tasks in Google Tasks| Google Tasks Tool                      | Retrieves multiple tasks                            | Task agent                  |                              | Fetches task lists                                                                                |
| Get many messages           | Gmail Tool                              | Retrieves multiple emails                            | Gmail tool                  |                              | Fetches emails                                                                                     |
| Get a message               | Gmail Tool                              | Retrieves a specific email                           | Gmail tool                  |                              | Fetches single email                                                                               |
| Research agent              | LangChain AgentTool                     | Coordinates multiple research tools                  |                              | AI Agent1                     | Handles research queries                                                                           |
| Brave Search                | Brave Search Tool                       | Web search tool                                     | Research agent              |                              | Provides independent web search                                                                   |
| Brave Search1               | Brave Search Tool (news)                | News search tool                                    | Research agent              |                              | Provides news search                                                                              |
| perprlexcia                 | HTTP Request Tool                      | Calls Perplexcia API for web search                  | Research agent              |                              | Deep research tool                                                                               |
| Wikipedia                  | LangChain Wikipedia Tool                | Wikipedia knowledge base                            | Research agent              |                              | Factual lookup                                                                                   |
| Tavily web search           | HTTP Request Tool                      | Calls Tavily search API                              | Research agent              |                              | Structured API-driven search                                                                     |
| gpt-4.1-nanoChat Model1     | AI Language Model                      | Language model for research agent                    | Research agent              |                              | AI model used in research                                                                         |
| gemini-2.5-flash            | AI Language Model                      | Language model for Gmail tool                         | Gmail tool                  |                              | AI model for email processing                                                                    |
| gpt-5-nano                  | AI Language Model                      | Language model for Gmail tool                         | Gmail tool                  |                              | AI model for email processing                                                                    |
| Webhook2                    | Webhook                               | Receives conversation data for memory processing     |                              | Extract Memory Info           | Memory input webhook                                                                             |
| Extract Memory Info         | LangChain Chain LLM                   | Extracts important info from conversation            | Webhook2                   | Check If Worth Remembering    | Processes conversation for memory extraction                                                    |
| Check If Worth Remembering  | If                                   | Decides if extracted info is worth remembering       | Extract Memory Info         | Edit Fields1                 | Filters relevant memory info                                                                    |
| Edit Fields1                | Set                                  | Prepares data fields for memory database insertion    | Check If Worth Remembering  | MongoDB Atlas Vector Store   | Formats info for MongoDB storage                                                                |
| MongoDB Atlas Vector Store  | Vector Store MongoDB Atlas            | Inserts extracted memory info into vector store       | Edit Fields1               |                              | Stores long-term memory                                                                         |
| Google Gemini Chat Model    | LangChain LLM                        | Language model used for memory extraction             |                              | Extract Memory Info           | Provides AI processing for memory node                                                         |
| Default Data Loader         | LangChain Document Loader            | Loads default documents into vector store             |                              | MongoDB Atlas Vector Store   | Initializes vector store                                                                        |
| MongoDB Atlas Vector Store1 | Vector Store MongoDB Atlas            | Retrieves memory for image prompt generation           |                              | AI Agent                     | Memory retrieval for image agent                                                               |
| AI Agent                   | LangChain Agent                      | Specialized AI agent for image prompt generation       | Webhook3                   | Clean Prompt Text1            | Generates detailed image generation prompts                                                    |
| Webhook3                   | Webhook                             | Receives image generation requests                     |                              | AI Agent                     | Entry point for image generation workflow                                                     |
| Clean Prompt Text1          | Code                                | Cleans AI-generated prompt text                         | AI Agent                   | HTTP Request                 | Prepares prompt for image generation API                                                     |
| HTTP Request               | HTTP Request                        | Calls external image generation API                     | Clean Prompt Text1          | Edit Fields5                 | Sends prompt to image generation service                                                     |
| Edit Fields5               | Set                                 | Extracts base64 image string from API response          | HTTP Request               | Convert to File              | Prepares image data for sending                                                             |
| Convert to File            | Convert To File                     | Converts base64 image string to binary file              | Edit Fields5               | Send message3                | Prepares image for WhatsApp transmission                                                  |
| Send message3              | WhatsApp Send Message               | Sends generated image back to WhatsApp user               | Convert to File            | Summarize Chat              | Sends created image to user                                                                 |
| Summarize Chat             | LangChain Chain LLM                 | Summarizes conversation after image generation           | Send message3              | MongoDB Atlas Vector Store   | Creates short summaries for memory storage                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node**  
   - Type: WhatsApp Trigger  
   - Configure to listen for message updates  
   - Connect WhatsApp OAuth credentials

2. **Add Switch node**  
   - Condition 1: Check if incoming message JSON contains an image field (exists)  
   - Condition 2: Check if incoming message JSON contains text body (exists)  
   - Connect from WhatsApp Trigger

3. **For image path:**  
   - Add Download media node  
     - Operation: mediaUrlGet  
     - Media ID from `{{$json.messages[0].image.id}}`  
     - Connect from Switch "image" output  
     - Configure WhatsApp API credentials

   - Add HTTP Request5 node  
     - Method: GET  
     - URL from `{{$json.url}}` (output from Download media)  
     - Use WhatsApp API and Bearer Auth credentials  
     - Connect from Download media

   - Add Analyze image node  
     - Google Gemini image analysis  
     - Model: `models/gemini-2.0-flash`  
     - Input type: binary  
     - Connect from HTTP Request5

4. **For text path:**  
   - Add Typing.... node  
     - HTTP POST to Facebook Graph API to send typing indicator  
     - Use WhatsApp API credentials  
     - Connect from Switch "chat" output

5. **Add Code1 node**  
   - JavaScript code to extract and unify input from either image analysis or text message  
   - Connect from Analyze image and Typing.... nodes

6. **Add AI Agent1 node**  
   - LangChain Agent node  
   - Configure system message with detailed instructions (see overview)  
   - Enable retry on fail  
   - Connect from Code1  
   - Attach Simple Memory and MongoDB Vector Store2 as context memory  
   - Connect external tool agents (Calendar, Task, Gmail, WeatherMap, Research, image creation)

7. **Add Simple Memory node**  
   - Session key: "whatsapp"  
   - Connect to AI Agent1 memory input

8. **Add MongoDB Atlas Vector Store2 node**  
   - Mode: retrieve-as-tool  
   - Top K: 5  
   - Configure MongoDB credentials and collection "document"  
   - Connect to AI Agent1 tool inputs

9. **Add external productivity tools:**  
   - Calendar Agent and Google Calendar nodes (Get many events, Create event, Update event, Delete event, Get event)  
   - Task agent and Google Tasks nodes (Create, Delete, Get many tasks)  
   - Gmail tool with Get many messages and Get a message nodes  
   - Connect each agent to AI Agent1 as tools

10. **Add Research agent node**  
    - Connect Brave Search, Brave Search1, perprlexcia, Wikipedia, Tavily web search nodes as tools  
    - Configure system message with search and execution rules  
    - Connect to AI Agent1

11. **Add image creation webhook (Webhook3)**  
    - Triggered by image creation requests  
    - Connect to AI Agent node specialized for image prompt generation  
    - Add Clean Prompt Text1 (code node) to sanitize prompt  
    - Add HTTP Request node to call external image generation API (Together.xyz)  
    - Add Edit Fields5 to extract base64 image  
    - Add Convert to File node to create binary file  
    - Add Send message3 node to send image back on WhatsApp  
    - Add Summarize Chat node for conversation summary  
    - Store summaries in MongoDB Atlas Vector Store

12. **Add memory management workflow:**  
    - Webhook2 to receive conversation turns  
    - Extract Memory Info node (chain LLM) to analyze conversation  
    - Check If Worth Remembering (If node) to filter info  
    - Edit Fields1 to prepare data  
    - MongoDB Atlas Vector Store node to insert memory  
    - Connect Google Gemini Chat Model for LLM processing

13. **Connect final Send message node**  
    - Sends AI Agent1's response back to WhatsApp user  
    - Connect from AI Agent1 main output

14. **Add send chat history to MongoDB HTTP Request**  
    - Logs conversation transcript externally  
    - Connect from Send message

15. **Configure all necessary credentials:**  
    - WhatsApp OAuth and API credentials  
    - Google OAuth2 for Calendar, Tasks, Gmail  
    - MongoDB Atlas credentials  
    - Google Palm API credentials for Gemini models  
    - OpenRouter, Groq, Vercel AI Gateway credentials for various LLMs  
    - API keys for Brave Search, Tavily, Perplexcia, Together.xyz image API

16. **Set default values and parameters:**  
    - Calendar and Task list IDs fixed as per setup  
    - City name in WeatherMap node as "bengaluru" or configurable  
    - Ensure all webhook paths are unique and registered in n8n

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow converts WhatsApp into a powerful multi-modal AI assistant with memory, productivity suite integration, and multi-agent orchestration. It supports Tamil, English, and mixed language inputs (Thunglish).                                                                                                                                                                                                                         | Sticky Note: Main Workflow                                                                                       |
| The WhatsApp trigger and media download section improves user experience with typing indicators and processes images with Google Gemini analysis.                                                                                                                                                                                                                                                                                             | Sticky Note: WhatsApp Trigger & Download media                                                                  |
| The core AI agent orchestrates multiple sub-agents and tools, using memory and external APIs for a rich conversational experience.                                                                                                                                                                                                                                                                                                          | Sticky Note: Core AI Agent                                                                                        |
| Productivity tools agents connect to Google Calendar, Tasks, and Gmail for scheduling, task management, and email reading.                                                                                                                                                                                                                                                                                                                  | Sticky Note: Productivity Tools Agen                                                                              |
| The memory subsystem extracts and stores meaningful conversation data for long-term context using MongoDB vector stores.                                                                                                                                                                                                                                                                                                                   | Sticky Note: Memory Webhook                                                                                       |
| Image generation is handled in a dedicated webhook workflow, generating prompts, calling external APIs, and sending images back to WhatsApp.                                                                                                                                                                                                                                                                                               | Sticky Note: Image Generation Webhook                                                                             |
| Research agent uses Brave Search, Wikipedia, Perplexcia, and Tavily with detailed rules for best results and rate limiting.                                                                                                                                                                                                                                                                                                                | Sticky Note: Research Tool Agent                                                                                  |
| The workflow uses multiple AI models from providers like Google Gemini (PaLM), OpenAI, OpenRouter, Groq, and Vercel AI Gateway for diversified AI capabilities.                                                                                                                                                                                                                                                                            | Credential notes                                                                                                |
| For detailed understanding of n8n LangChain nodes and vector stores, see n8n official docs and LangChain integrations.                                                                                                                                                                                                                                                                                                                   | n8n Docs: https://docs.n8n.io/nodes/                                                               |
| For WhatsApp API setup and credential configuration, refer to WhatsApp Business API documentation and n8n credential guides.                                                                                                                                                                                                                                                                                                              | WhatsApp API Docs: https://developers.facebook.com/docs/whatsapp/                                                |

---

*Disclaimer: The provided text is generated exclusively from an automated workflow created with n8n, respecting content policies. All data handled is legal and public.*