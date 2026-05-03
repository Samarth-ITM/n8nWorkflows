End-to-End Blog Creation Automation with Gemini AI, Supabase and Nano-Banana

https://n8nworkflows.xyz/workflows/end-to-end-blog-creation-automation-with-gemini-ai--supabase-and-nano-banana-9079


# End-to-End Blog Creation Automation with Gemini AI, Supabase and Nano-Banana

### 1. Workflow Overview

This workflow automates the end-to-end creation of blog content by leveraging AI models (notably Google Gemini and Groq chat models), data management with Supabase, and image generation and handling with Nano-Banana and HTTP requests. It orchestrates content generation, image creation, metadata extraction, and storage, enabling a fully automated pipeline for blog post creation and publishing.

**Target Use Cases:**  
- Automated content generation for blogs or publications based on RSS feeds and web scraping  
- AI-assisted text and image generation with structured output and metadata extraction  
- Automated data persistence and asset management in Supabase  
- Scheduled workflows for periodic blog updates or new article creation  

**Logical Blocks:**

- **1.1 Scheduling and API Key Management:** Initiates workflow on schedule and retrieves required API keys.  
- **1.2 Data Ingestion:** Reads RSS feeds, performs web scraping, and searches to gather source content.  
- **1.3 AI Text Processing:** Uses LangChain agents with Google Gemini and Groq models for content generation, structured parsing, and metadata extraction.  
- **1.4 Content Processing and Looping:** Processes AI outputs, extracts images and other fields, loops over image generation steps.  
- **1.5 Image Generation and Upload:** Generates images, converts formats, uploads to storage, and generates presigned URLs.  
- **1.6 Final Content Assembly and Persistence:** Embeds images into HTML content and creates rows in Supabase for storage.  
- **1.7 Control Flow and Wait Handling:** Conditional routing and wait nodes to manage retries and pacing.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Scheduling and API Key Management

**Overview:**  
This block manages the scheduled trigger for workflow execution and fetches API keys from Supabase necessary for authentication with AI services and other integrations.

**Nodes Involved:**  
- Schedule Trigger  
- API Keys  
- Get many rows  
- Code2  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow based on defined schedule (default settings, no parameters specified)  
  - Inputs: None  
  - Outputs: API Keys  
  - Failures: None expected unless n8n environment issues  

- **API Keys**  
  - Type: Set  
  - Role: Prepares credentials or variables for API key retrieval  
  - Inputs: Schedule Trigger  
  - Outputs: Get many rows  
  - Failures: Misconfiguration if keys not set correctly  

- **Get many rows** (Supabase)  
  - Type: Supabase node  
  - Role: Fetches API keys or configuration data from a Supabase database table  
  - Inputs: API Keys  
  - Outputs: Code2  
  - Failures: Auth errors, network timeouts, empty result sets  

- **Code2**  
  - Type: Code (JavaScript)  
  - Role: Processes the retrieved API keys and prepares them for downstream nodes  
  - Inputs: Get many rows  
  - Outputs: AI Agent1  
  - Failures: Code errors if data structure unexpected  

---

#### 1.2 Data Ingestion

**Overview:**  
This block collects content from RSS feeds, performs web scraping, and executes web searches as input data for AI processing.

**Nodes Involved:**  
- RSS Read1  
- RSS Read2  
- RSS Read  
- URL Scraper  
- Google search  

**Node Details:**  

- **RSS Read, RSS Read1, RSS Read2**  
  - Type: RSS Feed Read  
  - Role: Reads multiple RSS feeds to gather new content items  
  - Inputs: Connected to AI Agents as ai_tool inputs  
  - Outputs: AI Agents  
  - Failures: Feed unavailability, malformed XML, empty feeds  

- **URL Scraper**  
  - Type: HTTP Request Tool  
  - Role: Scrapes content from URLs for enrichment  
  - Inputs: AI Agents as ai_tool inputs  
  - Outputs: AI Agents  
  - Failures: HTTP errors, rate limiting, dynamic content issues  

- **Google search**  
  - Type: HTTP Request Tool  
  - Role: Performs Google search requests to find relevant information  
  - Inputs: AI Agents as ai_tool inputs  
  - Outputs: AI Agents  
  - Failures: Captchas, API restrictions, network failures  

---

#### 1.3 AI Text Processing

**Overview:**  
This block uses AI agents configured with multiple language models (Google Gemini, Groq) to generate, parse, and extract structured content and metadata from the ingested data.

**Nodes Involved:**  
- AI Agent  
- AI Agent1  
- Groq Chat Model  
- Groq Chat Model1  
- Google Gemini Chat Model  
- Google Gemini Chat Model1  
- Google Gemini Chat Model2  
- Google Gemini Chat Model4  
- Structured Output Parser  
- Information Extractor  

**Node Details:**  

- **AI Agent, AI Agent1**  
  - Type: LangChain Agent  
  - Role: Central AI processing nodes that coordinate AI models and parse outputs  
  - Configuration: Retry enabled with max 2 tries, 5-second wait between tries, always outputs data  
  - Inputs: RSS Reads, URL Scraper, Google search, Code2  
  - Outputs: Conditional nodes (If, If1) or next processing nodes  
  - Failures: API limits, model failures, parsing errors  

- **Groq Chat Model, Groq Chat Model1**  
  - Type: LangChain AI Model (Groq)  
  - Role: Provide AI language model capabilities as part of agents  
  - Inputs: AI Agents as ai_languageModel  
  - Failures: Auth issues, model unavailability  

- **Google Gemini Chat Models (multiple instances)**  
  - Type: LangChain AI Model (Google Gemini)  
  - Role: Multiple instantiations for different AI tasks (chat, structured output, extraction)  
  - Inputs: AI Agents or parsers  
  - Failures: API errors, quota exceeded  

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI outputs into structured formats for further processing  
  - Inputs: AI Agent  
  - Outputs: AI Agent  
  - Failures: Parsing mismatches, unexpected output formats  

- **Information Extractor**  
  - Type: LangChain Information Extractor  
  - Role: Extracts key information from AI generated content (e.g., metadata, entities)  
  - Inputs: AI Agent1  
  - Outputs: Code node for next processing  
  - Failures: Extraction errors, incomplete data  

---

#### 1.4 Content Processing and Looping

**Overview:**  
Processes the structured AI output, extracts and prepares images, manages batch processing of images and prompts, and prepares data for image generation.

**Nodes Involved:**  
- Code  
- Loop Over Items  
- all_images  
- Message a model (image name writer)  
- Edit Fields (set image prompt and name)  
- nano banana  

**Node Details:**  

- **Code**  
  - Type: Code  
  - Role: Processes extracted information and prepares data for looping  
  - Inputs: Information Extractor  
  - Outputs: Loop Over Items  
  - Failures: Code exceptions  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Splits items for batch processing of images and prompts  
  - Inputs: Code  
  - Outputs: all_images, Message a model (image name writer)  
  - Failures: Batch size misconfiguration  

- **all_images**  
  - Type: Code  
  - Role: Aggregates or manipulates images into a final form for HTML embedding  
  - Inputs: Loop Over Items  
  - Outputs: html_content (set node)  
  - Failures: Data manipulation errors  

- **Message a model (image name writer)**  
  - Type: LangChain Google Gemini model  
  - Role: Generates image names or prompts for image generation based on text input  
  - Inputs: Loop Over Items  
  - Outputs: Edit Fields (set node)  
  - Failures: API errors, naming errors  

- **Edit Fields (set image prompt and name)**  
  - Type: Set  
  - Role: Sets key fields for image prompt and filename before image generation  
  - Inputs: Message a model (image name writer)  
  - Outputs: nano banana  
  - Failures: Missing field values  

- **nano banana**  
  - Type: HTTP Request  
  - Role: Calls Nano-Banana API for image generation  
  - Inputs: Edit Fields  
  - Outputs: download image node  
  - Failures: API failures, request timeouts  

---

#### 1.5 Image Generation and Upload

**Overview:**  
Handles image download, conversion, upload to storage, and presigned URL generation for access.

**Nodes Involved:**  
- download image  
- Edit Image (only for changing to png)  
- Upload object  
- Generate presigned URL  
- final image url  
- Wait1  

**Node Details:**  

- **download image**  
  - Type: HTTP Request  
  - Role: Downloads generated image from Nano-Banana or other sources  
  - Inputs: nano banana  
  - Outputs: Edit Image  
  - Failures: HTTP errors, file not found  

- **Edit Image (only for changing to png)**  
  - Type: Edit Image  
  - Role: Converts or edits image format to PNG as required for upload  
  - Inputs: download image  
  - Outputs: Upload object  
  - Failures: Image processing errors  

- **Upload object**  
  - Type: HTTP Request  
  - Role: Uploads image file to storage service (likely Supabase or other cloud storage)  
  - Inputs: Edit Image  
  - Outputs: Generate presigned URL  
  - Failures: Upload errors, auth failures  

- **Generate presigned URL**  
  - Type: HTTP Request  
  - Role: Generates a presigned URL for image access  
  - Inputs: Upload object  
  - Outputs: final image url  
  - Failures: API errors  

- **final image url**  
  - Type: Set  
  - Role: Stores or sets the final image URL to be used downstream  
  - Inputs: Generate presigned URL  
  - Outputs: Wait1  
  - Failures: Missing URL value  

- **Wait1**  
  - Type: Wait  
  - Role: Introduces delay before continuing to avoid rate limiting or pacing issues  
  - Inputs: final image url  
  - Outputs: Loop Over Items (for next batch)  
  - Failures: None expected  

---

#### 1.6 Final Content Assembly and Persistence

**Overview:**  
Assembles the final HTML content embedding images and creates a new row in Supabase for storing the blog content.

**Nodes Involved:**  
- html_content  
- imbed images in html  
- Create a row  

**Node Details:**  

- **html_content**  
  - Type: Set  
  - Role: Prepares or holds the HTML content with placeholder or base structure  
  - Inputs: all_images  
  - Outputs: imbed images in html  
  - Failures: Incorrect content formatting  

- **imbed images in html**  
  - Type: Code  
  - Role: Embeds all processed images into the HTML content dynamically  
  - Inputs: html_content  
  - Outputs: Create a row  
  - Failures: Code errors, malformed HTML  

- **Create a row**  
  - Type: Supabase  
  - Role: Inserts the completed blog post content with images and metadata into Supabase table  
  - Inputs: imbed images in html  
  - Outputs: None (end node)  
  - Failures: DB connection errors, data validation errors  

---

#### 1.7 Control Flow and Wait Handling

**Overview:**  
Handles conditional decision-making to route workflow paths and manages timing with wait nodes to control pacing and retries.

**Nodes Involved:**  
- If  
- If1  
- Wait  
- Wait1  
- Wait2  

**Node Details:**  

- **If, If1**  
  - Type: If  
  - Role: Conditional branching based on outputs or flags to choose between retry/wait or continue processing  
  - Inputs: AI Agent1, AI Agent  
  - Outputs: Wait nodes or next AI agent nodes  
  - Failures: Incorrect condition expressions  

- **Wait, Wait1, Wait2**  
  - Type: Wait  
  - Role: Pause execution for a defined duration or until webhook is triggered to manage rate limits or pacing  
  - Inputs: If, final image url, If1  
  - Outputs: AI Agents or Loop Over Items  
  - Failures: None expected  

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                                  | Input Node(s)                   | Output Node(s)                        | Sticky Note |
|----------------------------------|----------------------------------|-------------------------------------------------|--------------------------------|-------------------------------------|-------------|
| Schedule Trigger                 | Schedule Trigger                 | Starts workflow on schedule                      | None                           | API Keys                            |             |
| API Keys                        | Set                              | Prepares API key variables                       | Schedule Trigger               | Get many rows                      |             |
| Get many rows                   | Supabase                         | Retrieves API keys/configuration                 | API Keys                      | Code2                             |             |
| Code2                          | Code                             | Processes API keys for use                        | Get many rows                 | AI Agent1                        |             |
| RSS Read                       | RSS Feed Read                   | Reads RSS feed for content                        | AI Agent                      | AI Agent (ai_tool)                |             |
| RSS Read1                      | RSS Feed Read                   | Reads RSS feed for content                        | AI Agent1                     | AI Agent1 (ai_tool)               |             |
| RSS Read2                      | RSS Feed Read                   | Reads RSS feed for content                        | AI Agent1                     | AI Agent1 (ai_tool)               |             |
| URL Scraper                    | HTTP Request Tool               | Scrapes web content                               | AI Agent1, AI Agent (ai_tool) | AI Agent1, AI Agent (ai_tool)     |             |
| Google search                  | HTTP Request Tool               | Performs Google searches                          | AI Agent1, AI Agent (ai_tool) | AI Agent1, AI Agent (ai_tool)     |             |
| AI Agent                      | LangChain Agent                 | Central AI processing                             | RSS Read, Wait2, If1          | If1                              |             |
| AI Agent1                     | LangChain Agent                 | Central AI processing                             | Code2, Wait, If               | If                               |             |
| Groq Chat Model               | LangChain LM                   | Provides Groq language model                      | AI Agent                      | AI Agent (ai_languageModel)       |             |
| Groq Chat Model1              | LangChain LM                   | Provides Groq language model                      | AI Agent1                     | AI Agent1 (ai_languageModel)      |             |
| Google Gemini Chat Model      | LangChain LM                   | Provides Google Gemini language model             | AI Agent                      | AI Agent (ai_languageModel)       |             |
| Google Gemini Chat Model1     | LangChain LM                   | Provides Google Gemini language model             | AI Agent1                     | AI Agent1 (ai_languageModel)      |             |
| Google Gemini Chat Model2     | LangChain LM                   | Provides Google Gemini language model             | Structured Output Parser      | Structured Output Parser (ai_languageModel) |             |
| Google Gemini Chat Model4     | LangChain LM                   | Provides Google Gemini language model             | Information Extractor         | Information Extractor (ai_languageModel)    |             |
| Structured Output Parser      | LangChain Output Parser        | Parses structured AI output                        | AI Agent                     | AI Agent                         |             |
| Information Extractor         | LangChain Information Extractor | Extracts metadata and info from AI output         | AI Agent1                    | Code                            |             |
| Code                         | Code                           | Processes extracted info for batching             | Information Extractor        | Loop Over Items                 |             |
| Loop Over Items              | SplitInBatches                 | Processes images and prompts in batches           | Code                        | all_images, Message a model (image name writer) |             |
| all_images                  | Code                           | Aggregates images for HTML embedding               | Loop Over Items             | html_content                   |             |
| Message a model (image name writer) | LangChain Google Gemini LM | Generates image names/prompts                      | Loop Over Items             | Edit Fields (set image prompt and name) |             |
| Edit Fields (set image prompt and name) | Set                  | Sets image prompt and filename                      | Message a model              | nano banana                   |             |
| nano banana                | HTTP Request                   | Calls Nano-Banana API for image generation          | Edit Fields                 | download image                |             |
| download image             | HTTP Request                   | Downloads generated image                            | nano banana                 | Edit Image (only for changing to png) |             |
| Edit Image (only for changing to png) | Edit Image            | Converts images to PNG format                         | download image              | Upload object                |             |
| Upload object              | HTTP Request                   | Uploads image to storage                              | Edit Image                  | Generate presigned URL       |             |
| Generate presigned URL     | HTTP Request                   | Generates presigned URL for image access             | Upload object               | final image url              |             |
| final image url            | Set                            | Sets final image URL for embedding                    | Generate presigned URL      | Wait1                       |             |
| Wait1                     | Wait                           | Waits before continuing to avoid rate limits          | final image url             | Loop Over Items             |             |
| html_content              | Set                            | Holds HTML content base structure                      | all_images                  | imbed images in html       |             |
| imbed images in html      | Code                           | Embeds images into HTML content                         | html_content                | Create a row                |             |
| Create a row              | Supabase                       | Inserts blog content row into database                  | imbed images in html        | None                       |             |
| If                       | If                             | Conditional routing for AI Agent1 outputs              | AI Agent1                   | Wait, AI Agent1             |             |
| If1                      | If                             | Conditional routing for AI Agent outputs               | AI Agent                    | Wait2, Information Extractor |             |
| Wait                     | Wait                           | Waits to pace AI Agent1 retry or continuation           | If                         | AI Agent1                   |             |
| Wait2                    | Wait                           | Waits to pace AI Agent retry or continuation            | If1                        | AI Agent                    |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run at desired intervals (e.g., daily or hourly).

2. **Create Set Node "API Keys"**  
   - Prepare variables or placeholders for API keys.

3. **Create Supabase Node "Get many rows"**  
   - Connect to your Supabase instance.  
   - Configure to retrieve API keys or config data from a table.

4. **Create Code Node "Code2"**  
   - Process the retrieved data to extract and set API keys for downstream use.

5. **Create LangChain Agent Node "AI Agent1"**  
   - Connect credentials (Google Gemini, Groq models).  
   - Configure retry behavior (max 2 tries, 5s wait).  
   - Connect input from Code2.

6. **Create RSS Feed Read Nodes "RSS Read1" and "RSS Read2"**  
   - Configure each with appropriate RSS feed URLs.  
   - Connect as ai_tool inputs to AI Agent1.

7. **Create HTTP Request Tool Nodes "Google search" and "URL Scraper"**  
   - Configure for web search and scraping endpoints or APIs.  
   - Connect as ai_tool inputs to both AI Agent1 and AI Agent.

8. **Create LangChain Agent Node "AI Agent"**  
   - Setup like AI Agent1 with retry and credentials.  
   - Connect input from Wait2 and If1 nodes.

9. **Create Groq and Google Gemini Chat Model Nodes**  
   - Create nodes for Groq Chat Model, Groq Chat Model1, Google Gemini Chat Models (multiple).  
   - Connect these as ai_languageModel inputs to AI Agent and AI Agent1 as appropriate.

10. **Create Structured Output Parser Node**  
    - Connect output of AI Agent to this parser.  
    - Connect its output back to AI Agent for structured data processing.

11. **Create Information Extractor Node**  
    - Connect AI Agent1 output.  
    - Connect output to Code node.

12. **Create Code Node "Code"**  
    - Implement JavaScript logic to parse AI output and prepare image prompts and metadata.  
    - Output to SplitInBatches node.

13. **Create SplitInBatches Node "Loop Over Items"**  
    - Configure batch size (default or 1 for sequential processing).  
    - Connect input from Code.  
    - Outputs to Code node "all_images" and LangChain Google Gemini node "Message a model (image name writer)".

14. **Create Code Node "all_images"**  
    - Aggregate images and prepare HTML embedding data.  
    - Output to Set node "html_content".

15. **Create Set Node "html_content"**  
    - Holds the base HTML content or template.  
    - Connect output to Code node "imbed images in html".

16. **Create Code Node "imbed images in html"**  
    - Embed images into HTML dynamically.  
    - Output connected to Supabase "Create a row" node.

17. **Create Supabase Node "Create a row"**  
    - Configure to insert blog content into a Supabase table (content, metadata, images).  
    - Input from "imbed images in html".

18. **Create LangChain Google Gemini Node "Message a model (image name writer)"**  
    - Generates image prompts and names.  
    - Connect output to Set node "Edit Fields".

19. **Create Set Node "Edit Fields (set image prompt and name)"**  
    - Set image prompt and filename variables.  
    - Output to HTTP Request node "nano banana".

20. **Create HTTP Request Node "nano banana"**  
    - Configure for Nano-Banana API image generation endpoint.  
    - Input from "Edit Fields".  
    - Output to HTTP Request node "download image".

21. **Create HTTP Request Node "download image"**  
    - Download generated image file.  
    - Output to "Edit Image (only for changing to png)".

22. **Create Edit Image Node "Edit Image (only for changing to png)"**  
    - Converts images to PNG format.  
    - Output to HTTP Request node "Upload object".

23. **Create HTTP Request Node "Upload object"**  
    - Upload image to storage (e.g., Supabase storage bucket).  
    - Output to HTTP Request node "Generate presigned URL".

24. **Create HTTP Request Node "Generate presigned URL"**  
    - Generates presigned URL for image access.  
    - Output to Set node "final image url".

25. **Create Set Node "final image url"**  
    - Stores final image URL for embedding.  
    - Output to Wait node "Wait1".

26. **Create Wait Node "Wait1"**  
    - Introduce delay before looping back to "Loop Over Items" for next batch.

27. **Create Conditional If Nodes "If" and "If1"**  
    - Define conditions to route flow between retries (Wait nodes) or continue processing.

28. **Create Wait Nodes "Wait" and "Wait2"**  
    - Use for pacing retries or delays before AI agents restart.

29. **Connect all nodes according to dependencies and ensure credentials for all HTTP requests and AI models are correctly configured.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                        |
|-----------------------------------------------------------------------------------------------|-------------------------------------|
| This workflow uses Google Gemini AI models integrated via LangChain nodes in n8n.             | AI integration details               |
| Supabase is used for data persistence including API key storage and blog content management.  | https://supabase.com                 |
| Nano-Banana API is used for image generation tasks.                                           | https://nano-banana.io (example)    |
| Wait nodes are critical to avoid rate limiting when interacting with AI APIs.                 | n8n documentation on Wait node       |
| Structured Output Parser ensures AI responses are parsed into usable JSON structures.         | LangChain structured parsing docs    |

---

**Disclaimer:** The above documentation is produced from an automated n8n workflow export. It complies with content policies and contains no illegal or protected material. All data usages are legitimate and public.