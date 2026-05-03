Generate LinkedIn Posts from Books using OpenAI, LangChain & Pinecone Vector Search

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-from-books-using-openai--langchain---pinecone-vector-search-6198


# Generate LinkedIn Posts from Books using OpenAI, LangChain & Pinecone Vector Search

### 1. Workflow Overview

This n8n workflow automates the generation of LinkedIn posts inspired by the content of books uploaded as PDF files to a specific Google Drive folder. It leverages AI technologies including OpenAI language models, LangChain agents, and Pinecone vector search to extract, process, and creatively repurpose book content into engaging LinkedIn post ideas and finalized posts.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and Input Reception:** Detects new or updated PDF files in a designated Google Drive folder and downloads them.
- **1.2 Content Extraction and Vector Database Generation:** Extracts text from the PDF, splits it into chunks, generates embeddings using OpenAI, and inserts these into a Pinecone vector database for semantic search.
- **1.3 LinkedIn Post Idea Generation:** Uses a LangChain agent connected to Pinecone to retrieve relevant book content and generate multiple creative LinkedIn post ideas.
- **1.4 Post Content Generation and Management:** Converts each post idea into a polished LinkedIn post using OpenAI, stores the posts in Google Sheets, and manages posting schedules.
- **1.5 Scheduled Posting and Publishing:** Periodically checks for unpublished posts, posts them to LinkedIn, and updates their status in the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Reception

**Overview:**  
This block monitors a specific Google Drive folder for new or updated PDF files (books). Upon detection, it downloads the PDF for further processing.

**Nodes Involved:**  
- Google Drive Trigger  
- DownLoadPdf

**Node Details:**

- **Google Drive Trigger**  
  - *Type:* Trigger node for Google Drive events  
  - *Configuration:* Watches for `fileUpdated` events every minute on a specific folder (ID: `1x24Xr6wl3INqGv68UbFIVVqkKybnZeDf`) named "LinkedinPosts"  
  - *Credentials:* Google Drive OAuth2  
  - *Input:* None (trigger)  
  - *Output:* Metadata of updated files including file ID and original filename  
  - *Potential Failures:* Authentication errors, folder access permissions, API rate limits

- **DownLoadPdf**  
  - *Type:* Google Drive node (download operation)  
  - *Configuration:* Downloads the file using the ID from the trigger node  
  - *Credentials:* Google Drive OAuth2  
  - *Input:* File ID from Google Drive Trigger  
  - *Output:* Binary PDF file content for extraction  
  - *Potential Failures:* File not found, download errors, permission issues

---

#### 2.2 Content Extraction and Vector Database Generation

**Overview:**  
Extracts text from the downloaded PDF, processes it into manageable chunks, generates embeddings, and stores them in a Pinecone vector index for semantic search.

**Nodes Involved:**  
- Extract from File  
- Pinecone Vector Store  
- Embeddings OpenAI  
- Default Data Loader  
- Recursive Character Text Splitter  
- Aggregate

**Node Details:**

- **Extract from File**  
  - *Type:* File extraction node  
  - *Configuration:* Extracts text from PDF binary data  
  - *Input:* PDF binary from DownLoadPdf  
  - *Output:* Extracted raw text content  
  - *Potential Failures:* Corrupted PDF, unsupported format, extraction errors

- **Default Data Loader**  
  - *Type:* LangChain document loader  
  - *Configuration:* Loads extracted text for further processing  
  - *Input:* Extracted text from previous node  
  - *Output:* Document object for LangChain processing

- **Recursive Character Text Splitter**  
  - *Type:* LangChain text splitter  
  - *Configuration:* Splits text into chunks with 10-character overlap for context preservation  
  - *Input:* Document from Default Data Loader  
  - *Output:* Array of text chunks suitable for embedding

- **Embeddings OpenAI**  
  - *Type:* LangChain embeddings generator  
  - *Configuration:* Uses OpenAI API to generate vector embeddings for text chunks  
  - *Credentials:* OpenAI API key  
  - *Input:* Text chunks from splitter  
  - *Output:* Embeddings vectors

- **Pinecone Vector Store**  
  - *Type:* LangChain Pinecone vector store node  
  - *Configuration:* Inserts embeddings into Pinecone index named `linkdenpost-new` with namespace set dynamically to the PDF filename  
  - *Credentials:* Pinecone API key  
  - *Input:* Embeddings from OpenAI node  
  - *Output:* Confirmation of insertion  
  - *Potential Failures:* API errors, namespace conflicts, rate limits

- **Aggregate**  
  - *Type:* Aggregate node  
  - *Configuration:* Aggregates metadata fields for downstream processing  
  - *Input:* Output from Pinecone Vector Store  
  - *Output:* Aggregated data

---

#### 2.3 LinkedIn Post Idea Generation

**Overview:**  
Generates creative LinkedIn post ideas by querying the Pinecone vector database with a LangChain agent enhanced by OpenAI chat models.

**Nodes Involved:**  
- OpenAI Chat Model  
- Pinecorn Vector Store-book (Retrieve Tool)  
- Simple Memory  
- Structured Output Parser  
- LinkedIn Post Idea Generation (LangChain Agent)  
- Split Out

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* LangChain chat language model node  
  - *Configuration:* Uses GPT-4o model for conversational AI tasks  
  - *Credentials:* OpenAI API key  
  - *Input:* Queries from LangChain agent  
  - *Output:* AI-generated text responses

- **Pinecorn Vector Store-book**  
  - *Type:* LangChain Pinecone vector store retrieval tool  
  - *Configuration:* Retrieves relevant book content from Pinecone index `linkdenpost-new` using the PDF filename as namespace  
  - *Credentials:* Pinecone API key  
  - *Input:* Query from LangChain agent  
  - *Output:* Relevant text chunks for AI processing

- **Simple Memory**  
  - *Type:* LangChain memory buffer window  
  - *Configuration:* Maintains session context with a fixed session key (`111`)  
  - *Input/Output:* Stores conversation history for agent continuity

- **Structured Output Parser**  
  - *Type:* LangChain output parser  
  - *Configuration:* Parses AI output into a JSON array of objects with fields: Hook, Insight, CTA  
  - *Input:* Raw AI text output  
  - *Output:* Structured JSON for downstream nodes

- **LinkedIn Post Idea Generation**  
  - *Type:* LangChain agent node  
  - *Configuration:*  
    - Uses a detailed system prompt defining the role as a creative content strategist.  
    - Tasked to search Pinecone vector DB and generate exactly 5 unique LinkedIn post ideas with hook, insight, and CTA.  
    - Output is JSON formatted.  
    - Max 3 iterations for refinement.  
  - *Input:* Book name from Google Drive Trigger  
  - *Output:* JSON array of post ideas  
  - *Potential Failures:* API timeouts, parsing errors, insufficient vector data

- **Split Out**  
  - *Type:* Split Out node  
  - *Configuration:* Splits the JSON array of post ideas into individual items for separate processing  
  - *Input:* JSON array from output parser  
  - *Output:* Individual post idea objects

---

#### 2.4 Post Content Generation and Management

**Overview:**  
Transforms each LinkedIn post idea into a polished LinkedIn post, stores the posts in a Google Sheet, and prepares them for publishing.

**Nodes Involved:**  
- GeneratePostContent (OpenAI)  
- linkedInPostsContent (Google Sheets)  
- linkedInPostsContent1 (Google Sheets)  
- linkedInPostsContent2 (Google Sheets)  
- Limit  
- If  
- LinkedIn

**Node Details:**

- **GeneratePostContent**  
  - *Type:* LangChain OpenAI node  
  - *Configuration:*  
    - Uses GPT-4.1-mini model to generate a professional LinkedIn post (max 600 characters) based on Hook, Insight, and CTA.  
    - Includes constraints for tone, emojis, hashtags, and LinkedIn appropriateness.  
  - *Credentials:* OpenAI API key  
  - *Input:* Individual post idea (Hook, Insight, CTA) from Split Out  
  - *Output:* Final LinkedIn post content string

- **linkedInPostsContent**  
  - *Type:* Google Sheets node  
  - *Configuration:* Appends or updates rows in a Google Sheet (ID: `1ZqO8N0nrI529TimXaH7RZHJmQgdi86feyolN1tTlQmc`, sheet gid=0) with post data including book name, hook, insight, CTA, post content, and published status ("no")  
  - *Credentials:* Google Sheets OAuth2  
  - *Input:* Post content and metadata from GeneratePostContent and Split Out  
  - *Output:* Confirmation of sheet update

- **linkedInPostsContent1**  
  - *Type:* Google Sheets node  
  - *Configuration:* Reads rows from the same Google Sheet filtering for posts where `published` = "no" (unpublished posts)  
  - *Credentials:* Google Sheets OAuth2  
  - *Input:* Scheduled trigger (see next block)  
  - *Output:* List of unpublished posts for processing

- **Limit**  
  - *Type:* Limit node  
  - *Configuration:* Limits the number of posts processed per run (default limit, no parameters specified)  
  - *Input:* Unpublished posts from linkedInPostsContent1  
  - *Output:* Limited subset of posts

- **LinkedIn**  
  - *Type:* LinkedIn node  
  - *Configuration:* Posts the content to LinkedIn on behalf of a specific user (person ID: `nSBpG_FGY1`)  
  - *Credentials:* LinkedIn OAuth2  
  - *Input:* Post content from Limit node  
  - *Output:* Posting result including URN

- **If**  
  - *Type:* Conditional node  
  - *Configuration:* Checks if the LinkedIn post URN exists (indicating successful post)  
  - *Input:* LinkedIn node output  
  - *Output:* Routes to update Google Sheet if post succeeded

- **linkedInPostsContent2**  
  - *Type:* Google Sheets node  
  - *Configuration:* Updates the corresponding row in the Google Sheet to mark the post as `published = "yes"` and adds the current date  
  - *Credentials:* Google Sheets OAuth2  
  - *Input:* Post metadata and row number from linkedInPostsContent1 and If node  
  - *Output:* Confirmation of update

---

#### 2.5 Scheduled Posting and Publishing

**Overview:**  
Periodically triggers the workflow to check for unpublished LinkedIn posts and initiates the posting process.

**Nodes Involved:**  
- Schedule Trigger  
- linkedInPostsContent1  
- Limit  
- LinkedIn  
- If  
- linkedInPostsContent2

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule trigger node  
  - *Configuration:* Runs at a configured interval (default unspecified, but example timestamp shows July 20, 2025, 09:07 UTC)  
  - *Input:* None (trigger)  
  - *Output:* Triggers downstream nodes to process unpublished posts

- The rest of the nodes in this block are as described in 2.4, forming the scheduled posting pipeline.

---

### 3. Summary Table

| Node Name                    | Node Type                                   | Functional Role                                   | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                     |
|-----------------------------|---------------------------------------------|-------------------------------------------------|-----------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Google Drive Trigger         | n8n-nodes-base.googleDriveTrigger            | Detect new/updated PDFs in Google Drive folder  | None                        | DownLoadPdf                    | ## Trigger when book uploaded                                                                  |
| DownLoadPdf                 | n8n-nodes-base.googleDrive                   | Download PDF file from Google Drive              | Google Drive Trigger         | Extract from File               | ## Trigger when book uploaded                                                                  |
| Extract from File           | n8n-nodes-base.extractFromFile               | Extract text content from PDF                     | DownLoadPdf                 | Pinecone Vector Store           | ## Generate the vector database of the book                                                    |
| Default Data Loader         | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Load extracted text as document for processing   | Extract from File           | Recursive Character Text Splitter | ## Generate the vector database of the book                                                    |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Split text into chunks with overlap               | Default Data Loader         | Embeddings OpenAI              | ## Generate the vector database of the book                                                    |
| Embeddings OpenAI           | @n8n/n8n-nodes-langchain.embeddingsOpenAi   | Generate embeddings for text chunks               | Recursive Character Text Splitter | Pinecone Vector Store           | ## Generate the vector database of the book                                                    |
| Pinecone Vector Store       | @n8n/n8n-nodes-langchain.vectorStorePinecone | Insert embeddings into Pinecone vector index      | Embeddings OpenAI, Extract from File | Aggregate                     | ## Generate the vector database of the book                                                    |
| Aggregate                  | n8n-nodes-base.aggregate                      | Aggregate metadata for downstream processing      | Pinecone Vector Store       | LinkedIn Post Idea Generation   | ## Generate post Ideas                                                                         |
| OpenAI Chat Model           | @n8n/n8n-nodes-langchain.lmChatOpenAi       | AI language model for chat-based tasks            | LinkedIn Post Idea Generation (agent) | LinkedIn Post Idea Generation (agent) | ## Generate post Ideas                                                                         |
| Pinecorn Vector Store-book  | @n8n/n8n-nodes-langchain.vectorStorePinecone | Retrieve relevant book content from Pinecone      | LinkedIn Post Idea Generation (agent) | LinkedIn Post Idea Generation (agent) | ## Generate post Ideas                                                                         |
| Simple Memory               | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintain conversational context                    | LinkedIn Post Idea Generation (agent) | LinkedIn Post Idea Generation (agent) | ## Generate post Ideas                                                                         |
| Structured Output Parser    | @n8n/n8n-nodes-langchain.outputParserStructured | Parse AI output into structured JSON               | LinkedIn Post Idea Generation (agent) | LinkedIn Post Idea Generation   | ## Generate post Ideas                                                                         |
| LinkedIn Post Idea Generation | @n8n/n8n-nodes-langchain.agent              | Generate 5 LinkedIn post ideas from vector DB     | Aggregate                   | Split Out                      | ## Generate post Ideas                                                                         |
| Split Out                  | n8n-nodes-base.splitOut                       | Split array of post ideas into individual items   | LinkedIn Post Idea Generation | GeneratePostContent            | ## Generate post content and update sheet                                                     |
| GeneratePostContent         | @n8n/n8n-nodes-langchain.openAi              | Generate final LinkedIn post content from ideas   | Split Out                   | linkedInPostsContent           | ## Generate post content and update sheet                                                     |
| linkedInPostsContent        | n8n-nodes-base.googleSheets                   | Append or update LinkedIn post data in Google Sheets | GeneratePostContent          | None                          | ## Generate post content and update sheet                                                     |
| Schedule Trigger            | n8n-nodes-base.scheduleTrigger                | Periodic trigger to check for unpublished posts   | None                        | linkedInPostsContent1          | # Post linkedin Daily                                                                         |
| linkedInPostsContent1       | n8n-nodes-base.googleSheets                   | Read unpublished posts from Google Sheets        | Schedule Trigger            | Limit                         | # Post linkedin Daily                                                                         |
| Limit                      | n8n-nodes-base.limit                          | Limit number of posts to process per run          | linkedInPostsContent1       | LinkedIn                      | # Post linkedin Daily                                                                         |
| LinkedIn                   | n8n-nodes-base.linkedIn                        | Publish post content to LinkedIn                   | Limit                       | If                            | # Post linkedin Daily                                                                         |
| If                         | n8n-nodes-base.if                             | Check if LinkedIn post was successful             | LinkedIn                   | linkedInPostsContent2          | # Post linkedin Daily                                                                         |
| linkedInPostsContent2       | n8n-nodes-base.googleSheets                   | Update published status and date in Google Sheets | If                         | None                          | # Post linkedin Daily                                                                         |
| Sticky Note                | n8n-nodes-base.stickyNote                      | Visual note: Post linkedin Daily                   | None                        | None                         | # Post linkedin Daily                                                                         |
| Sticky Note1               | n8n-nodes-base.stickyNote                      | Visual note: Trigger when book uploaded            | None                        | None                         | ## Trigger when book uploaded                                                                 |
| Sticky Note2               | n8n-nodes-base.stickyNote                      | Visual note: Generate the vector database of the book | None                        | None                         | ## Generate the vector database of the book                                                   |
| Sticky Note3               | n8n-nodes-base.stickyNote                      | Visual note: Generate post Ideas                    | None                        | None                         | ## Generate post Ideas                                                                        |
| Sticky Note4               | n8n-nodes-base.stickyNote                      | Visual note: Generate post content and update sheet | None                        | None                         | ## Generate post content<br>## Update the linkedin post sheet                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Event: `fileUpdated`  
   - Poll every minute  
   - Watch specific folder by ID: `1x24Xr6wl3INqGv68UbFIVVqkKybnZeDf` ("LinkedinPosts")  
   - Credentials: Google Drive OAuth2 account

2. **Create DownLoadPdf Node**  
   - Type: Google Drive (download operation)  
   - File ID: Use expression `{{$json["id"]}}` from Google Drive Trigger  
   - Credentials: Same Google Drive OAuth2 account  
   - Connect Google Drive Trigger → DownLoadPdf

3. **Create Extract from File Node**  
   - Type: Extract from File  
   - Operation: PDF text extraction  
   - Input: Binary data from DownLoadPdf  
   - Connect DownLoadPdf → Extract from File

4. **Create Default Data Loader Node**  
   - Type: LangChain Document Default Data Loader  
   - Input: Text output from Extract from File  
   - Connect Extract from File → Default Data Loader

5. **Create Recursive Character Text Splitter Node**  
   - Type: LangChain Recursive Character Text Splitter  
   - Chunk Overlap: 10 characters  
   - Input: Document from Default Data Loader  
   - Connect Default Data Loader → Recursive Character Text Splitter

6. **Create Embeddings OpenAI Node**  
   - Type: LangChain Embeddings OpenAI  
   - Credentials: OpenAI API key  
   - Input: Text chunks from Recursive Character Text Splitter  
   - Connect Recursive Character Text Splitter → Embeddings OpenAI

7. **Create Pinecone Vector Store Node**  
   - Type: LangChain Vector Store Pinecone (Insert mode)  
   - Pinecone Index: `linkdenpost-new`  
   - Namespace: Use expression `{{$node["DownLoadPdf"].item.json.name}}` (PDF filename)  
   - Credentials: Pinecone API key  
   - Input: Embeddings from Embeddings OpenAI  
   - Connect Embeddings OpenAI → Pinecone Vector Store  
   - Also connect Extract from File → Pinecone Vector Store (for metadata)

8. **Create Aggregate Node**  
   - Type: Aggregate  
   - Aggregate metadata field for downstream use  
   - Connect Pinecone Vector Store → Aggregate

9. **Create LinkedIn Post Idea Generation Node (LangChain Agent)**  
   - Type: LangChain Agent  
   - Text Input: `Name of the book: {{$node["Google Drive Trigger"].item.json.originalFilename}}`  
   - System Prompt: Use detailed prompt for generating 5 LinkedIn post ideas based on Pinecone vector DB content  
   - Max Iterations: 3  
   - Output Format: JSON array with Hook, Insight, CTA  
   - Connect Aggregate → LinkedIn Post Idea Generation

10. **Create OpenAI Chat Model Node**  
    - Type: LangChain Chat OpenAI  
    - Model: GPT-4o  
    - Credentials: OpenAI API key  
    - Connect as AI language model for LinkedIn Post Idea Generation node

11. **Create Pinecone Vector Store-book Node (Retrieve Tool)**  
    - Type: LangChain Vector Store Pinecone (Retrieve mode)  
    - Pinecone Index: `linkdenpost-new`  
    - Namespace: Set to a sample book PDF filename or dynamic as needed  
    - Tool Name: `linkedinpostnew`  
    - Credentials: Pinecone API key  
    - Connect as AI tool for LinkedIn Post Idea Generation node

12. **Create Simple Memory Node**  
    - Type: LangChain Memory Buffer Window  
    - Session Key: `111` (custom key)  
    - Connect as AI memory for LinkedIn Post Idea Generation node

13. **Create Structured Output Parser Node**  
    - Type: LangChain Output Parser Structured  
    - Schema: JSON array of objects with string properties Hook, Insight, CTA  
    - Connect as AI output parser for LinkedIn Post Idea Generation node

14. **Create Split Out Node**  
    - Type: Split Out  
    - Field to split: `output` (the array of post ideas)  
    - Connect LinkedIn Post Idea Generation → Split Out

15. **Create GeneratePostContent Node (OpenAI)**  
    - Type: LangChain OpenAI  
    - Model: GPT-4.1-mini  
    - Credentials: OpenAI API key  
    - Messages:  
      - First message: Template injecting Hook, Insight, CTA from Split Out  
      - Second message: Prompt template for generating professional LinkedIn post (max 600 chars, emojis, hashtags)  
    - Connect Split Out → GeneratePostContent

16. **Create linkedInPostsContent Node (Google Sheets)**  
    - Type: Google Sheets (append or update)  
    - Document ID: `1ZqO8N0nrI529TimXaH7RZHJmQgdi86feyolN1tTlQmc`  
    - Sheet GID: `0`  
    - Columns: bookname, hook, insight, cta, postContent, published ("no")  
    - Credentials: Google Sheets OAuth2  
    - Connect GeneratePostContent → linkedInPostsContent

17. **Create Schedule Trigger Node**  
    - Type: Schedule Trigger  
    - Configure interval as desired (e.g., daily at 9:00 UTC)  
    - Connect to linkedInPostsContent1

18. **Create linkedInPostsContent1 Node (Google Sheets)**  
    - Type: Google Sheets (read)  
    - Document ID and Sheet GID same as above  
    - Filter: `published` = "no" (unpublished posts)  
    - Credentials: Google Sheets OAuth2  
    - Connect Schedule Trigger → linkedInPostsContent1

19. **Create Limit Node**  
    - Type: Limit  
    - Default limit or set as needed (e.g., 1 post per run)  
    - Connect linkedInPostsContent1 → Limit

20. **Create LinkedIn Node**  
    - Type: LinkedIn  
    - Person ID: `nSBpG_FGY1` (LinkedIn user to post as)  
    - Text: Use post content from Limit node  
    - Credentials: LinkedIn OAuth2  
    - Connect Limit → LinkedIn

21. **Create If Node**  
    - Type: If  
    - Condition: Check if LinkedIn post URN exists (post success)  
    - Connect LinkedIn → If

22. **Create linkedInPostsContent2 Node (Google Sheets)**  
    - Type: Google Sheets (update)  
    - Document ID and Sheet GID same as above  
    - Update row: set `published` = "yes", `date` = current timestamp, matched by row_number from linkedInPostsContent1  
    - Credentials: Google Sheets OAuth2  
    - Connect If (true branch) → linkedInPostsContent2

23. **Add Sticky Notes for Documentation**  
    - Add visual sticky notes in the editor to mark logical blocks: Trigger, Vector DB generation, Post Idea Generation, Post Content Generation, and Scheduled Posting.

---

This detailed breakdown and stepwise reconstruction guide enable developers and AI agents to understand, reproduce, and modify the workflow effectively, while anticipating potential failure points such as API authentication issues, data parsing errors, or rate limits on external services.