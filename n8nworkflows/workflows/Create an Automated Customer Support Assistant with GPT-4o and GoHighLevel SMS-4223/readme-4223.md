Create an Automated Customer Support Assistant with GPT-4o and GoHighLevel SMS

https://n8nworkflows.xyz/workflows/create-an-automated-customer-support-assistant-with-gpt-4o-and-gohighlevel-sms-4223


# Create an Automated Customer Support Assistant with GPT-4o and GoHighLevel SMS

### 1. Workflow Overview

This workflow automates a customer support assistant that integrates GPT-4o (an OpenAI large language model) with GoHighLevel (GHL) SMS platform. Its primary purpose is to scrape a website’s content, build a searchable vector knowledge base from that content, and use an AI agent to respond intelligently to incoming SMS inquiries by referencing the website knowledge. The responses are sent back via GoHighLevel SMS.

The workflow logically divides into these blocks:

- **1.1 Website Content Scraping & Data Preparation:** Scrapes the website homepage and sitemap, extracts textual content and links, normalizes and merges links, and loads the data for embedding.
- **1.2 Vector Store Construction:** Generates OpenAI embeddings from the website text chunks and stores them in an in-memory vector store to create a searchable knowledge base.
- **1.3 Incoming SMS Webhook & Contact Lookup:** Receives inbound SMS messages via a GHL webhook, verifies message type, and looks up contact details.
- **1.4 AI Processing & Contextual Chat Memory:** Uses Redis for chat memory per contact, invokes the AI agent configured with access to the knowledge base, and generates a contextual response.
- **1.5 SMS Response:** Sends the AI-generated response back to the contact via GoHighLevel SMS API.
- **1.6 Scheduling & Batch Processing:** Scheduled trigger to periodically update the knowledge base and batch scrape website pages.
- **1.7 Utilities & Error Handling:** Includes filtering, duplicate removal, link editing, and error continuation for robustness.

---

### 2. Block-by-Block Analysis

#### 1.1 Website Content Scraping & Data Preparation

- **Overview:**  
  This block fetches the website content from the homepage and sitemap, extracts text and links, merges them, and prepares the data for embedding.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set Website URL  
  - BrightData (web scraping proxy)  
  - HTML (extract HTML content)  
  - Get sitemap (HTTP request for sitemap XML)  
  - Get XML file  
  - Split out links  
  - Edit Links1  
  - Split Out (split links array)  
  - Filter (links starting with "/" or base URL)  
  - Edit Links  
  - Merge  
  - Remove Duplicates1  
  - Loop Over Items  
  - BrightData1 (scrape each link)  
  - HTML1 (extract HTML from each scraped page)  
  - Simple Vector Store (insert mode)  
  - Default Data Loader  
  - Recursive Character Text Splitter

- **Node Details:**  
  - **Schedule Trigger:** Runs every 4 days at 12:00 to initiate data scraping.  
  - **Set Website URL:** Sets a static variable `website_url` (e.g., https://yourwebsite.com) used downstream.  
  - **BrightData:** Uses Bright Data proxy to fetch the homepage content securely, bypassing anti-bot measures.  
  - **HTML:** Extracts raw text from the homepage HTML body and all `<a>` links' href attributes.  
  - **Get sitemap:** Attempts to fetch the sitemap XML (e.g., https://yourwebsite.com/post-sitemap.xml). Continues on error if sitemap unavailable.  
  - **Get XML file:** Parses the sitemap XML.  
  - **Split out links:** Extracts individual URLs from the sitemap XML nodes.  
  - **Edit Links1:** Maps sitemap URLs into a normalized `link` field.  
  - **Split Out:** Splits the array of links extracted from the homepage.  
  - **Filter:** Filters links to only those starting with “/” (relative paths) or the base website URL to avoid external links.  
  - **Edit Links:** Normalizes filtered links by prepending base URL if relative.  
  - **Merge:** Combines sitemap and homepage extracted links arrays.  
  - **Remove Duplicates1:** Removes duplicate links from the merged list.  
  - **Loop Over Items:** Iterates over each unique link in batches.  
  - **BrightData1:** Scrapes each link’s page content via Bright Data proxy.  
  - **HTML1:** Extracts raw text from each scraped page HTML body.  
  - **Simple Vector Store:** Inserts all extracted text chunks into an in-memory vector store (`vector_db`), clearing previous content on each run.  
  - **Default Data Loader:** Loads raw text into Langchain document format for embedding.  
  - **Recursive Character Text Splitter:** Splits large text into chunks of 300 characters to optimize embedding.

- **Edge Cases & Potential Failures:**  
  - Sitemap may not exist or be blocked; workflow continues with homepage links only.  
  - Bright Data API failures or rate limits may cause missing pages.  
  - Link normalization must carefully handle relative vs absolute URLs.  
  - Duplicate removal critical to avoid redundant scraping.  
  - Large websites may exceed batch size and memory limits in in-memory vector store.  
  - Recursive splitter may fail on malformed or empty text.

---

#### 1.2 Vector Store Construction

- **Overview:**  
  Converts the prepared website text chunks into vector embeddings using OpenAI embeddings and stores them in an in-memory Langchain vector store for retrieval.

- **Nodes Involved:**  
  - Embeddings OpenAI  
  - Simple Vector Store (insert mode)  
  - Simple Vector Store2 (insert mode, parallel path)  
  - Default Data Loader  
  - Recursive Character Text Splitter

- **Node Details:**  
  - **Embeddings OpenAI:** Calls OpenAI API to generate embeddings for each text chunk.  
  - **Simple Vector Store:** Stores the embeddings in memory under key `vector_db`.  
  - The vector store is cleared before insertion each run to refresh data.  
  - Note: This in-memory store is suitable for testing; production should use persistent vector DBs like Pinecone or Supabase.

- **Edge Cases & Potential Failures:**  
  - OpenAI API rate limits or auth failures.  
  - Large text volumes leading to high token usage or timeout.  
  - In-memory store data loss on workflow restart.

---

#### 1.3 Incoming SMS Webhook & Contact Lookup

- **Overview:**  
  Listens for inbound SMS messages from GoHighLevel, filters inbound message types, and fetches contact details for personalized responses.

- **Nodes Involved:**  
  - Webhook from GHL - SMS Reply Trigger  
  - If (filter inbound messages)  
  - Look Up GHL Contact by ID  
  - Set Website URL1

- **Node Details:**  
  - **Webhook from GHL - SMS Reply Trigger:** Receives POST webhook from GHL with message payload including contact ID and message body.  
  - **If:** Checks that the webhook event type equals "InboundMessage" to proceed.  
  - **Look Up GHL Contact by ID:** Uses GHL API to fetch contact phone number and details by contact ID from webhook. Requires OAuth2 credentials.  
  - **Set Website URL1:** Re-sets the `website_url` variable for AI context.

- **Edge Cases & Potential Failures:**  
  - Webhook misconfiguration or missing events.  
  - Non-inbound messages or unsupported event types ignored.  
  - GHL API auth errors or contact not found.  
  - Missing contact phone number breaks SMS response.

---

#### 1.4 AI Processing & Contextual Chat Memory

- **Overview:**  
  Uses GPT-4o to generate an AI response to the SMS inquiry, enriched by chat memory and the website knowledgebase as a vector tool.

- **Nodes Involved:**  
  - Redis Chat Memory  
  - Simple Vector Store1 (retrieve tool mode)  
  - Embeddings OpenAI1  
  - AI Agent  
  - OpenAI Chat Model  
  - Set Website URL1

- **Node Details:**  
  - **Redis Chat Memory:** Maintains conversational context per contact using Redis, keyed by contact ID from webhook. Limits context window to 2 previous messages.  
  - **Simple Vector Store1:** Acts as a Langchain tool to retrieve relevant knowledgebase info from `vector_db` during agent execution.  
  - **Embeddings OpenAI1:** Provides embeddings for similarity search during retrieval.  
  - **AI Agent:** The core Langchain agent that processes the inbound SMS text, uses the chatbot system message to identify itself as a helpful assistant for the website, and calls the vector store tool as needed. Uses the OpenAI chat model.  
  - **OpenAI Chat Model:** Executes the GPT-4o model to generate the conversational response.  
  - The system prompt instructs the agent to always use the knowledgebase tool for queries about the company.

- **Edge Cases & Potential Failures:**  
  - Redis connectivity or data loss impacts chat context.  
  - OpenAI API timeouts or errors.  
  - Agent tool invocation errors or empty retrieval results.  
  - Unexpected input formats in webhook body.

---

#### 1.5 SMS Response

- **Overview:**  
  Sends the AI-generated reply back to the user via GoHighLevel SMS API.

- **Nodes Involved:**  
  - Send SMS via GHL

- **Node Details:**  
  - **Send SMS via GHL:** HTTP POST request to GHL conversations/messages endpoint with OAuth2 credentials. Includes message text, contact ID, and recipient phone number from lookup. Retries on failure enabled.

- **Edge Cases & Potential Failures:**  
  - GHL API auth errors or rate limits.  
  - Missing or invalid phone numbers.  
  - Network errors or timeouts.

---

#### 1.6 Scheduling & Batch Processing

- **Overview:**  
  Automates periodic knowledgebase refreshing and manages batch scraping of multiple pages.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Loop Over Items  
  - Wait

- **Node Details:**  
  - **Schedule Trigger:** Initiates scraping every 4 days.  
  - **Loop Over Items:** Processes scraped links in batches to avoid rate limits and large payloads.  
  - **Wait:** Introduces delay if required between batches (configured but no parameters set).  

- **Edge Cases & Potential Failures:**  
  - Large websites may exceed batch sizes.  
  - Timing conflicts if schedule triggers before previous run completes.

---

#### 1.7 Utilities & Error Handling

- **Overview:**  
  Supports other blocks with filtering, duplicate removal, merging, and error continuation.

- **Nodes Involved:**  
  - Filter  
  - Remove Duplicates1  
  - Merge  
  - If (with continue on error for sitemap fetch)  

- **Node Details:**  
  - **Filter:** Ensures only relevant links are processed.  
  - **Remove Duplicates1:** Avoids redundant processing.  
  - **Merge:** Combines multiple input streams.  
  - **If:** Checks event types.  
  - **Get sitemap:** Configured with `continueErrorOutput` to allow graceful failure if sitemap unavailable.

- **Edge Cases & Potential Failures:**  
  - Filter misconfiguration may exclude valid links or include unwanted ones.  
  - Merge node input count must be managed carefully if sitemap is missing.

---

### 3. Summary Table

| Node Name                         | Node Type                               | Functional Role                                   | Input Node(s)                             | Output Node(s)                               | Sticky Note                                                                                                                                                    |
|----------------------------------|---------------------------------------|-------------------------------------------------|------------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | scheduleTrigger                       | Initiates periodic scraping                      |                                          | Set Website URL                               |                                                                                                                                                                |
| Set Website URL                 | set                                  | Sets base website URL variable                   | Schedule Trigger                          | BrightData                                    | ## Set the website URL                                                                                                                                          |
| BrightData                      | brightData                           | Scrapes homepage content via proxy               | Set Website URL                           | HTML                                          | ## Scrape the home page<br>If you are using N8N cloud version, you may replace Bright Data node with the HTTP request node and rewire it.                      |
| HTML                           | html                                 | Extracts homepage raw text and links             | BrightData                               | Simple Vector Store, Get sitemap, Split Out  |                                                                                                                                                                |
| Get sitemap                    | httpRequest                          | Fetches sitemap XML                               | HTML                                     | Get XML file, Merge                           | ## Get the website's sitemap<br>Sitemap may not work on all of the website. The alternative is to scrape all the links in a website as shown below.               |
| Get XML file                   | xml                                  | Parses sitemap XML                                | Get sitemap                              | Split out links                               |                                                                                                                                                                |
| Split out links               | splitOut                             | Extracts individual URLs from sitemap             | Get XML file                             | Edit Links1                                   |                                                                                                                                                                |
| Edit Links1                   | set                                  | Normalizes sitemap URLs into `link` field        | Split out links                          | Merge                                         |                                                                                                                                                                |
| Split Out                    | splitOut                             | Splits homepage extracted links array             | HTML                                     | Filter                                        |                                                                                                                                                                |
| Filter                       | filter                               | Filters links starting with "/" or base URL       | Split Out                               | Edit Links                                    |                                                                                                                                                                |
| Edit Links                   | set                                  | Normalizes filtered links with base URL           | Filter                                  | Merge                                         |                                                                                                                                                                |
| Merge                        | merge                                | Merges sitemap and homepage links                 | Edit Links1, Edit Links, Get sitemap (error path) | Remove Duplicates1                            | ## Merge the links from the sitemap and the extracted links from the webpage<br>If the sitemap is not working, remove the merge node or set the node's input into 1. |
| Remove Duplicates1           | removeDuplicates                     | Removes duplicate links                            | Merge                                   | Loop Over Items                                |                                                                                                                                                                |
| Loop Over Items              | splitInBatches                      | Processes links in batches                         | Remove Duplicates1                      | BrightData1 (batch scrape)                     | ## Scrape each links by batch                                                                                                                                   |
| BrightData1                  | brightData                          | Scrapes each page content via proxy               | Loop Over Items                        | HTML1                                         |                                                                                                                                                                |
| HTML1                       | html                                | Extracts raw text from each scraped page          | BrightData1                           | Simple Vector Store2                           |                                                                                                                                                                |
| Simple Vector Store          | vectorStoreInMemory                 | Inserts text chunks into in-memory vector DB      | Default Data Loader, Embeddings OpenAI |                                           | ## Store into a vector database<br>Warning: This will only save into N8N's memory and it will not be a good in production use. Please consider moving the a dedicated vector database such as Pinecone, Supabase, etc. |
| Default Data Loader          | documentDefaultDataLoader           | Loads raw text into Langchain documents            | HTML                                  | Recursive Character Text Splitter              |                                                                                                                                                                |
| Recursive Character Text Splitter | textSplitterRecursiveCharacterTextSplitter | Splits large text into chunks for embedding       | Default Data Loader                   | Embeddings OpenAI                              |                                                                                                                                                                |
| Embeddings OpenAI            | embeddingsOpenAi                   | Generates embeddings from text chunks              | Recursive Character Text Splitter      | Simple Vector Store                            |                                                                                                                                                                |
| Simple Vector Store2         | vectorStoreInMemory                 | Inserts text chunks into in-memory vector DB (parallel path) | HTML1, Embeddings OpenAI               | Wait                                          |                                                                                                                                                                |
| Wait                        | wait                               | Introduces delay after batch processing             | Simple Vector Store2                   | Loop Over Items (continue batch)               |                                                                                                                                                                |
| Webhook from GHL - SMS Reply Trigger | webhook                           | Receives inbound SMS webhook from GHL              |                                          | If                                            | ## Webhook from GoHighlevel <br>Set the webhook events from your GoHighLevel Marketplace app, copy the webhook the node, then paste it in app's setting in the GHL app. To learn more about setting up GHL Marketplace app, please refer to this Loom video: https://www.loom.com/share/f32384758de74a4dbb647e0b7962c4ea |
| If                          | if                                  | Filters inbound message type "InboundMessage"      | Webhook from GHL - SMS Reply Trigger  | Look Up GHL Contact by ID                      |                                                                                                                                                                |
| Look Up GHL Contact by ID    | highLevel                          | Fetches contact info by ID                          | If                                    | Set Website URL1                               |                                                                                                                                                                |
| Set Website URL1             | set                                | Sets website URL variable for AI context           | Look Up GHL Contact by ID              | AI Agent                                      |                                                                                                                                                                |
| Redis Chat Memory           | memoryRedisChat                   | Maintains chat context per contact via Redis       | Webhook from GHL - SMS Reply Trigger  | AI Agent                                      |                                                                                                                                                                |
| Simple Vector Store1         | vectorStoreInMemory               | Vector store retrieval tool for AI agent            | Embeddings OpenAI1                    | AI Agent                                      |                                                                                                                                                                |
| Embeddings OpenAI1           | embeddingsOpenAi                 | Embeddings for retrieval during AI agent run       |                                    | Simple Vector Store1                           |                                                                                                                                                                |
| AI Agent                    | agent                             | Processes SMS text with GPT-4o and knowledge base  | Redis Chat Memory, OpenAI Chat Model, Simple Vector Store1 | Send SMS via GHL                              | ## AI Agent then send the output via SMS in GHL                                                                                                              |
| OpenAI Chat Model            | lmChatOpenAi                     | GPT-4o language model for generating responses      |                                    | AI Agent                                      |                                                                                                                                                                |
| Send SMS via GHL             | httpRequest                      | Sends AI-generated SMS reply to contact             | AI Agent                             |                                              |                                                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**  
   - Type: Schedule Trigger  
   - Configure: Set interval to every 4 days at 12:00.

2. **Create `Set Website URL` node:**  
   - Type: Set  
   - Add string variable `website_url` with your website URL (e.g., https://yourwebsite.com).  
   - Connect from Schedule Trigger.

3. **Create `BrightData` node:**  
   - Type: BrightData  
   - Credentials: Configure with your Bright Data API key.  
   - URL: Set to `={{ $json.website_url }}`  
   - Zone: Select appropriate proxy zone.  
   - Connect from Set Website URL.

4. **Create `HTML` node:**  
   - Type: HTML  
   - Operation: Extract HTML content.  
   - Extraction values: Extract raw text from `body` and all link hrefs (`a` tags).  
   - Connect from BrightData.

5. **Create `Get sitemap` node:**  
   - Type: HTTP Request  
   - URL: `={{ $json.website_url }}/post-sitemap.xml`  
   - Method: GET  
   - Enable "Continue on Error" to allow workflow to proceed if sitemap missing.  
   - Connect from HTML.

6. **Create `Get XML file` node:**  
   - Type: XML  
   - Connect from Get sitemap.

7. **Create `Split out links` node:**  
   - Type: SplitOut  
   - Field to split: `urlset.url` (from sitemap XML).  
   - Connect from Get XML file.

8. **Create `Edit Links1` node:**  
   - Type: Set  
   - Assignment: Create `link` field set to `={{ $json.loc }}`.  
   - Connect from Split out links.

9. **Create `Split Out` node:**  
   - Type: SplitOut  
   - Field to split: `links` (from homepage extraction).  
   - Connect from HTML.

10. **Create `Filter` node:**  
    - Type: Filter  
    - Condition: Keep items where `link` starts with "/" OR starts with `website_url` variable.  
    - Connect from Split Out.

11. **Create `Edit Links` node:**  
    - Type: Set  
    - Assignment: Set `link` to `={{ $json.link.startsWith("/") ? $json.website_url + $json.link : $json.link }}`.  
    - Connect from Filter.

12. **Create `Merge` node:**  
    - Type: Merge  
    - Connect inputs from Edit Links1, Edit Links, and Get sitemap error output (for fallback).  
    - Configure to merge arrays.

13. **Create `Remove Duplicates1` node:**  
    - Type: RemoveDuplicates  
    - Connect from Merge.

14. **Create `Loop Over Items` node:**  
    - Type: SplitInBatches  
    - Connect from Remove Duplicates1.

15. **Create `BrightData1` node:**  
    - Type: BrightData (with same API credentials)  
    - URL: `={{ $json.link }}`  
    - Connect from Loop Over Items.

16. **Create `HTML1` node:**  
    - Type: HTML  
    - Operation: Extract raw text from `body`.  
    - Connect from BrightData1.

17. **Create `Simple Vector Store2` node:**  
    - Type: vectorStoreInMemory (insert mode)  
    - Memory Key: `vector_db`  
    - Clear store: true  
    - Connect from HTML1.

18. **Create `Wait` node (optional):**  
    - Type: Wait  
    - Connect from Simple Vector Store2.  
    - Connect output back to Loop Over Items to continue batches.

19. **Create `Default Data Loader` node:**  
    - Type: documentDefaultDataLoader  
    - Input JSON: `={{ $json.raw_text }}` (from HTML)  
    - Connect from HTML.

20. **Create `Recursive Character Text Splitter` node:**  
    - Type: textSplitterRecursiveCharacterTextSplitter  
    - Chunk size: 300  
    - Connect from Default Data Loader.

21. **Create `Embeddings OpenAI` node:**  
    - Type: embeddingsOpenAi  
    - Credentials: OpenAI API key with GPT-4o access  
    - Connect from Recursive Character Text Splitter.

22. **Connect `Embeddings OpenAI` output to `Simple Vector Store` (insert mode):**  
    - Memory Key: `vector_db`  
    - Clear store: true

23. **Create `Webhook from GHL - SMS Reply Trigger` node:**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Path: unique webhook path  
    - Configure in GHL app Marketplace webhook settings.

24. **Create `If` node:**  
    - Condition: `$json.body.type` equals "InboundMessage"  
    - Connect from webhook.

25. **Create `Look Up GHL Contact by ID` node:**  
    - Type: highLevel (GHL API)  
    - Operation: get contact by ID  
    - Contact ID: `={{ $json.body.contactId }}`  
    - Connect from If node.

26. **Create `Set Website URL1` node:**  
    - Type: Set  
    - Assign `website_url` string variable same as before  
    - Connect from Look Up GHL Contact by ID.

27. **Create `Redis Chat Memory` node:**  
    - Type: memoryRedisChat  
    - Session Key: `={{ $json.body.contactId }}` (from webhook)  
    - Context window length: 2  
    - Connect from Webhook.

28. **Create `Embeddings OpenAI1` node:**  
    - Same OpenAI credentials  
    - Connect as AI embedding input for Simple Vector Store1.

29. **Create `Simple Vector Store1` node:**  
    - Type: vectorStoreInMemory (retrieve-as-tool mode)  
    - Tool name: `vector_db`  
    - Memory key: `english_center_nl`  
    - Tool description: "Call this tool to get information about the company and its products."  
    - Connect from Embeddings OpenAI1.

30. **Create `OpenAI Chat Model` node:**  
    - Type: lmChatOpenAi  
    - Model: GPT-4o  
    - Connect to AI Agent.

31. **Create `AI Agent` node:**  
    - Type: agent  
    - Text input: `={{ $('Webhook from GHL - SMS Reply Trigger').item.body.body }}`  
    - System message: "You are a helpful chatbot for {{ $json.website_url }}. Your goal is to assist website visitors by answering their questions based on the information available on the website. Knowledgebase: Always run the 'website_knowledgebase' tool when the user inquires anything about the company."  
    - Prompt type: define  
    - Connect inputs: Redis Chat Memory (ai_memory), OpenAI Chat Model (ai_languageModel), Simple Vector Store1 (ai_tool)  
    - Connect output to Send SMS via GHL.

32. **Create `Send SMS via GHL` node:**  
    - Type: HTTP Request  
    - URL: `https://services.leadconnectorhq.com/conversations/messages`  
    - Method: POST  
    - Authentication: predefined OAuth2 credential for GHL  
    - Body parameters:  
      - type: SMS  
      - contactId: `={{ $('Webhook from GHL - SMS Reply Trigger').item.json.body.contactId }}`  
      - message: `={{ $json.output }}` (AI Agent output)  
      - toNumber: `={{ $('Look Up GHL Contact by ID').item.json.phone }}`  
    - Headers: Accept application/json, Version 2021-04-15  
    - Enable retry on failure.  
    - Connect from AI Agent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook from GoHighlevel: Set the webhook events from your GoHighLevel Marketplace app, copy the webhook URL, then paste it in the app's settings in the GHL app.               | Loom video: https://www.loom.com/share/f32384758de74a4dbb647e0b7962c4ea                              |
| Scrape the home page: If you are using N8N cloud version, you may replace Bright Data node with the HTTP request node and rewire it.                                          |                                                                                                    |
| Store into a vector database: Warning: This workflow uses an in-memory vector store for demo purposes. In production, use dedicated vector databases like Pinecone or Supabase. |                                                                                                    |
| Get the website's sitemap: Sitemap may not work on all websites. If sitemap is unavailable, fallback is to scrape all links from the homepage and subpages.                    |                                                                                                    |
| Merge the links from the sitemap and the extracted links from the webpage. If the sitemap is not working, remove the merge node or set the node's input into 1.                |                                                                                                    |
| AI Agent then sends the output via SMS in GHL.                                                                                                                                 |                                                                                                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.