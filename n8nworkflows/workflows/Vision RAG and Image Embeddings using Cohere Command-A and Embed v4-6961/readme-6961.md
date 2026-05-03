Vision RAG and Image Embeddings using Cohere Command-A and Embed v4

https://n8nworkflows.xyz/workflows/vision-rag-and-image-embeddings-using-cohere-command-a-and-embed-v4-6961


# Vision RAG and Image Embeddings using Cohere Command-A and Embed v4

---

### 1. Workflow Overview

This n8n workflow demonstrates a Vision Retrieval-Augmented Generation (RAG) system combined with image embeddings using Cohere's Embed v4 and Command-A Vision model. It is designed primarily to handle scanned pages of the "Technology and Innovation Report 2025" (which contains charts, tables, and graphs) by embedding these images into a vector store (Qdrant) and enabling a conversational AI agent to answer user queries referencing these images.

The workflow logically separates into the following blocks:

- **1.1 Initialization and Data Preparation:** Loading URLs of report page images, downloading them, converting to Base64, and embedding images into vector representations.

- **1.2 Vector Storage Management:** Preparing embedding points, aggregating them, and inserting into the Qdrant vector database.

- **1.3 Vision RAG AI Agent Setup:** Handling user chat messages, running a language model agent with tool integration, deciding when to switch to the vision tool, and querying the vector store for relevant images.

- **1.4 Vision Model Query and Response:** Using the Cohere Command-A Vision model to interpret retrieved images along with user queries, and responding to the chat with enriched messages including image thumbnails.

- **1.5 Utility and Supporting Nodes:** Session memory management and intermediate message handling.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Data Preparation

- **Overview:** Starts with manual workflow execution to load a predefined list of image URLs representing report pages. It then downloads these images, converts them to Base64 format, and generates embeddings using Cohere’s Embed v4 model.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Technology and Innovation Report 2025  
  - Split Out Urls  
  - Batch 5  
  - Page Ref  
  - Download Page  
  - Convert Image to Base64  
  - Image Embeddings with Cohere Embed 4

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Configuration: No parameters; triggers downstream nodes.  
    - Input/Output: No input; outputs to "Technology and Innovation Report 2025".  
    - Edge cases: None; manual trigger.

  - **Technology and Innovation Report 2025**  
    - Type: Set node  
    - Role: Defines an array of URLs pointing to PNG images of report pages.  
    - Configuration: Raw JSON sets a key "url" with an array of 16 image URLs hosted on Cloudinary.  
    - Output: Passes the array to "Split Out Urls".  
    - Edge cases: URL list must be accurate; broken links cause download failures.

  - **Split Out Urls**  
    - Type: Split Out node  
    - Role: Splits the array of URLs into individual items for batch processing.  
    - Configuration: Splits on "url" field.  
    - Output: Each URL is output as a separate item.  
    - Edge cases: Empty array would result in no items; verify input.

  - **Batch 5**  
    - Type: Split In Batches  
    - Role: Processes URLs in batches of 5 to control concurrency and API usage.  
    - Configuration: Batch size set to 5.  
    - Edge cases: Final batch may contain fewer than 5 items.

  - **Page Ref**  
    - Type: No Operation (NoOp)  
    - Role: Pass-through node to facilitate referencing the current page URL downstream.  
    - Configuration: No parameters; serves as a placeholder.  
    - Edge cases: None.

  - **Download Page**  
    - Type: HTTP Request  
    - Role: Downloads the image at the current URL.  
    - Configuration: URL set dynamically from the current item's "url" field.  
    - Edge cases: HTTP errors (404, 500), slow responses, or timeouts.

  - **Convert Image to Base64**  
    - Type: Extract From File  
    - Role: Converts downloaded binary image file into Base64 string format.  
    - Configuration: Operation set to "binaryToProperty" to extract content as a property.  
    - Edge cases: Invalid file data, unsupported formats.

  - **Image Embeddings with Cohere Embed 4**  
    - Type: HTTP Request  
    - Role: Calls Cohere API to generate image embeddings from Base64 data.  
    - Configuration:  
      - POST to "https://api.cohere.com/v2/embed"  
      - Model: "embed-v4.0"  
      - Input type: "image"  
      - Embedding types: float embeddings  
      - Images array contains Base64 encoded image from previous node.  
    - Authentication: Uses stored "CohereApi" credentials.  
    - Edge cases: API limits, invalid image data, network errors.

---

#### 1.2 Vector Storage Management

- **Overview:** Prepares the embedding vectors with associated metadata, aggregates them into a single batch, and inserts them into a Qdrant vector store collection named "visionRagExample".

- **Nodes Involved:**  
  - Prepare Points  
  - Aggregate Points  
  - Insert Points  
  - Batch 5 (feedback connection)  
  - Create Collection

- **Node Details:**

  - **Prepare Points**  
    - Type: Set node  
    - Role: Formats the embedding output into Qdrant-compatible points with id, url, and embedding vector.  
    - Configuration:  
      - Sets "id" as the current item's id  
      - "url" from the "Page Ref" node’s URL  
      - "embedding" as the float vector extracted from the Cohere response  
    - Edge cases: Missing embedding data, mismatch of IDs or URLs.

  - **Aggregate Points**  
    - Type: Aggregate  
    - Role: Aggregates all point objects into one array under the field "points".  
    - Configuration: Aggregate all item data into "points".  
    - Edge cases: Empty input leading to empty points array.

  - **Insert Points**  
    - Type: Qdrant node (upsertPoints)  
    - Role: Inserts or updates the batch of embedding points into the Qdrant vector store.  
    - Configuration:  
      - Collection: "visionRagExample"  
      - Points JSON mapped from aggregated points: Each with id, payload (content=url), empty metadata, and vector embedding.  
    - Credentials: Uses local Qdrant REST API credentials.  
    - Edge cases: Network issues, Qdrant service unavailable, malformed points data.

  - **Create Collection**  
    - Type: Qdrant node (createCollection)  
    - Role: Creates the Qdrant collection "visionRagExample" with cosine distance and vector size 1536.  
    - Configuration:  
      - Distance metric: Cosine  
      - Vector size: 1536 (matching Cohere Embed v4 output)  
    - Credentials: Same as Insert Points.  
    - Edge cases: Collection already exists, permission issues.

---

#### 1.3 Vision RAG AI Agent Setup

- **Overview:** Handles incoming chat messages, processes them with a LangChain AI agent, checks if a specialized tool for the technology report is called, and branches accordingly between a vision-based Q&A or a regular text-based response.

- **Nodes Involved:**  
  - When chat message received  
  - Chat Model via Command-R  
  - AI Agent  
  - If has Tool Call?  
  - Technology Innovation Report Tool  
  - Get Query  
  - Quick Confirmation  
  - Get Relevant Images  
  - Aggregate Results  
  - Respond to Chat1  
  - Simple Memory1

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger (webhook)  
    - Role: Receives chat messages from users, public webhook enabled.  
    - Parameters: Response mode set to "responseNodes" to allow custom response handling.  
    - Edge cases: Webhook failures, malformed requests.

  - **Chat Model via Command-R**  
    - Type: LangChain Chat model (Cohere Command-R)  
    - Role: Provides a general-purpose language model for understanding user input and generating intermediate agent steps.  
    - Credentials: Cohere API.  
    - Edge cases: API limits, model unavailability.

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates the reasoning process, can invoke tools.  
    - Configuration: Returns intermediate steps to detect tool calls, system message sets assistant persona.  
    - Edge cases: Errors in tool execution, unexpected intermediate steps.

  - **Technology Innovation Report Tool**  
    - Type: LangChain Tool (code)  
    - Role: Represents a custom tool to query the technology report documents.  
    - Configuration: Manual input schema with a "query" string, dummy JS code returning "ok".  
    - Edge cases: Needs real implementation to query documents meaningfully.

  - **If has Tool Call?**  
    - Type: If node  
    - Role: Checks if any intermediate steps from AI Agent include a tool call to "Technology_Innovation_Report_Tool".  
    - Condition: Existence of such a tool call in the intermediate steps array.  
    - Branches: Yes -> Get Query; No -> Respond to Chat1 (normal agent response).  
    - Edge cases: False negatives if intermediateSteps format changes.

  - **Get Query**  
    - Type: Set node  
    - Role: Extracts the tool name and query text from the first intermediate step of the agent.  
    - Output: Passes extracted query to "Quick Confirmation".  
    - Edge cases: Intermediate steps array empty or malformed.

  - **Quick Confirmation**  
    - Type: LangChain Chat  
    - Role: Sends a "Please wait while I search the document..." message to user, indicating processing.  
    - Edge cases: Message delivery failures.

  - **Get Relevant Images**  
    - Type: LangChain Vector Store (Qdrant)  
    - Role: Queries the Qdrant vector store using the user’s query to retrieve relevant embedded image points.  
    - Parameters: Collection "visionRagExample"; prompt set to query string.  
    - Credentials: Qdrant API.  
    - Edge cases: Empty results, service unavailability.

  - **Aggregate Results**  
    - Type: Aggregate  
    - Role: Combines all retrieved vector store results into one array.  
    - Edge cases: Empty inputs.

  - **Respond to Chat1**  
    - Type: LangChain Chat  
    - Role: Sends a normal AI agent response when no tool call is detected.  
    - Memory connection enabled for conversation context.  
    - Edge cases: Message failures.

  - **Simple Memory1**  
    - Type: LangChain Memory Buffer (Window)  
    - Role: Maintains chat session memory based on user session ID for consistent conversation context.  
    - Edge cases: Memory overflow or session ID mismatches.

---

#### 1.4 Vision Model Query and Response

- **Overview:** After retrieving relevant images, this block runs the user query and the images through Cohere’s Command-A Vision model and sends back a chat response including image thumbnails.

- **Nodes Involved:**  
  - Image Understanding via Command-A-Vision  
  - Respond to Chat  
  - Simple Memory  
  - Embeddings

- **Node Details:**

  - **Image Understanding via Command-A-Vision**  
    - Type: HTTP Request  
    - Role: Sends a multimodal chat request to Cohere’s Command-A Vision LLM with user text query and images.  
    - Configuration:  
      - POST to "https://api.cohere.com/v2/chat"  
      - Model: "command-a-vision-07-2025"  
      - Message includes user query text and array of image URLs (with "auto" detail) from Qdrant results.  
    - Authentication: Cohere API.  
    - Edge cases: API limits, malformed input, large image array size.

  - **Respond to Chat**  
    - Type: LangChain Chat  
    - Role: Sends the vision model's answer back to the user with formatted message including image thumbnails.  
    - Message formatting uses Markdown tables with clickable thumbnails linking to original images.  
    - Memory connection enabled.  
    - Edge cases: Markdown rendering issues, message failures.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer (Window)  
    - Role: Session memory keyed by chat session ID to maintain conversation context.  
    - Edge cases: Same as Simple Memory1.

  - **Embeddings**  
    - Type: LangChain Embeddings (Cohere Embed v4)  
    - Role: Used internally for vector store querying to generate query embeddings matching the vector store format.  
    - Credentials: Cohere API.  
    - Edge cases: API rate limits.

---

#### 1.5 Utility and Supporting Nodes

- **Sticky Notes:** Multiple sticky notes provide context, instructions, and external resource links for users and developers, covering sections such as report download instructions, Cohere Embed v4 details, Qdrant usage, AI agent design, and response node usage.

- **Connection Notes:** The workflow uses some nodes as pass-through or placeholders (e.g., "Page Ref") to facilitate referencing data across branches.

---

### 3. Summary Table

| Node Name                          | Node Type                               | Functional Role                                    | Input Node(s)                      | Output Node(s)                             | Sticky Note                                      |
|-----------------------------------|---------------------------------------|---------------------------------------------------|----------------------------------|--------------------------------------------|-------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                        | Manual start trigger                               | -                                | Technology and Innovation Report 2025     |                                                 |
| Technology and Innovation Report 2025 | Set                                 | Provides list of report page image URLs            | When clicking ‘Execute workflow’ | Split Out Urls                            | Sticky Note about downloading report page scans |
| Split Out Urls                    | Split Out                             | Splits URL array into individual items             | Technology and Innovation Report 2025 | Batch 5                                |                                                 |
| Batch 5                          | Split In Batches                      | Processes URLs in batches of 5                      | Split Out Urls                   | Page Ref (via second output)                |                                                 |
| Page Ref                        | No Operation                          | Pass-through for referencing current URL           | Batch 5 (second output)          | Download Page                             |                                                 |
| Download Page                   | HTTP Request                         | Downloads image from URL                            | Page Ref                        | Convert Image to Base64                     |                                                 |
| Convert Image to Base64          | Extract From File                    | Converts binary images to Base64                    | Download Page                   | Image Embeddings with Cohere Embed 4       | Sticky Note about Embed v4 requiring base64      |
| Image Embeddings with Cohere Embed 4 | HTTP Request                         | Generates image embeddings using Cohere Embed v4   | Convert Image to Base64          | Prepare Points                            | Sticky Note about Cohere Embed v4 and image embeddings |
| Prepare Points                  | Set                                 | Formats embeddings for insertion into Qdrant       | Image Embeddings with Cohere Embed 4 | Aggregate Points                       | Sticky Note about storing vectors in Qdrant      |
| Aggregate Points                | Aggregate                           | Aggregates points into array for batch insertion    | Prepare Points                  | Insert Points                            |                                                 |
| Insert Points                  | Qdrant (upsertPoints)               | Inserts embedding points into Qdrant collection    | Aggregate Points                | Batch 5 (feedback)                       | Sticky Note about Qdrant collection creation      |
| Create Collection              | Qdrant (createCollection)           | Creates Qdrant collection "visionRagExample"        | -                              | -                                        | Sticky Note about creating Qdrant collection      |
| When chat message received      | LangChain Chat Trigger               | Receives chat messages                              | -                              | AI Agent                               |                                                 |
| Chat Model via Command-R        | LangChain Chat Model (Cohere)        | Language model for agent                            | When chat message received      | AI Agent                               |                                                 |
| AI Agent                      | LangChain Agent                      | Orchestrates AI reasoning and tool calls            | When chat message received, Chat Model via Command-R, Technology Innovation Report Tool | If has Tool Call?                    | Sticky Note about building Vision RAG agent       |
| Technology Innovation Report Tool | LangChain Tool Code                  | Custom tool placeholder for querying report         | AI Agent                       | AI Agent                               |                                                 |
| If has Tool Call?              | If                                  | Branches based on presence of tool call             | AI Agent                       | Get Query (yes), Respond to Chat1 (no)   | Sticky Note about vision model switch logic        |
| Get Query                    | Set                                 | Extracts query input from tool call                  | If has Tool Call? (yes)          | Quick Confirmation                       |                                                 |
| Quick Confirmation            | LangChain Chat                      | Sends "Please wait..." message                       | Get Query                     | Get Relevant Images                      |                                                 |
| Get Relevant Images           | LangChain Vector Store Qdrant       | Retrieves relevant images from vector store         | Quick Confirmation             | Aggregate Results                       |                                                 |
| Aggregate Results             | Aggregate                           | Aggregates retrieved images                          | Get Relevant Images            | Image Understanding via Command-A-Vision |                                                 |
| Image Understanding via Command-A-Vision | HTTP Request                         | Runs Command-A Vision model on query and images     | Aggregate Results              | Respond to Chat                         | Sticky Note about Command-A Vision model           |
| Respond to Chat               | LangChain Chat                      | Sends final vision model response to user           | Image Understanding via Command-A-Vision, Simple Memory | -                          | Sticky Note about Respond to Chat node usage       |
| Simple Memory                | LangChain Memory Buffer             | Maintains chat session memory                         | Respond to Chat                | Respond to Chat                         |                                                 |
| Respond to Chat1              | LangChain Chat                      | Sends normal agent response if no tool call          | If has Tool Call? (no)          | -                                        | Sticky Note about normal agent response             |
| Simple Memory1               | LangChain Memory Buffer             | Maintains chat session memory for normal responses   | Respond to Chat1               | Respond to Chat1                        |                                                 |
| Embeddings                  | LangChain Embeddings (Cohere)       | Generates embeddings for query vector search          | -                              | Get Relevant Images                     |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a "Manual Trigger" node named "When clicking ‘Execute workflow’". No parameters.

2. **Set Image URLs:**  
   - Add a "Set" node named "Technology and Innovation Report 2025".  
   - Assign a JSON field named "url" with an array of 16 PNG URLs from the Technology and Innovation Report 2025 scans.

3. **Split URLs into Items:**  
   - Add a "Split Out" node named "Split Out Urls".  
   - Configure it to split on the "url" field.

4. **Batch Processing:**  
   - Add "Split In Batches" node named "Batch 5".  
   - Set batch size to 5 for processing URLs in batches.

5. **Pass Through Current URL:**  
   - Add a "No Operation" node named "Page Ref" to hold the current URL reference.

6. **Download Images:**  
   - Add an "HTTP Request" node named "Download Page".  
   - Set URL to dynamic expression: `{{$json.url}}`.  
   - Method: GET.

7. **Convert Image to Base64:**  
   - Add "Extract From File" node named "Convert Image to Base64".  
   - Operation: binaryToProperty (extracts binary data to property).

8. **Generate Image Embeddings:**  
   - Add HTTP Request node named "Image Embeddings with Cohere Embed 4".  
   - POST to `https://api.cohere.com/v2/embed`.  
   - JSON Body:  
     ```json
     {
       "model": "embed-v4.0",
       "input_type": "image",
       "embedding_types": ["float"],
       "images": ["data:image/png;base64,{{ $json.data }}"]
     }
     ```  
   - Authentication: Use "CohereApi" credentials with proper API key.  
   - Headers: Accept and Content-Type as application/json.

9. **Prepare Points for Qdrant:**  
   - Add "Set" node named "Prepare Points".  
   - Assign fields:  
     - "id" = current item's id  
     - "url" = from "Page Ref" node's url  
     - "embedding" = first embedding float array from Cohere response.

10. **Aggregate Points:**  
    - Add "Aggregate" node named "Aggregate Points".  
    - Aggregate all item data into a single field "points".

11. **Insert Points into Qdrant:**  
    - Add "Qdrant" node named "Insert Points".  
    - Operation: upsertPoints  
    - Collection Name: "visionRagExample"  
    - Points JSON: map aggregated points to Qdrant point structure with id, payload (content=url), empty metadata, and vector embedding.  
    - Credentials: Qdrant REST API connected to your instance.

12. **(Optional) Create Qdrant Collection:**  
    - Add "Qdrant" node named "Create Collection".  
    - Operation: createCollection  
    - Collection Name: "visionRagExample"  
    - Vectors: distance="Cosine", size=1536  
    - Used if collection does not exist.

13. **Receive Chat Messages:**  
    - Add LangChain "Chat Trigger" node named "When chat message received".  
    - Public webhook enabled.

14. **Configure Chat Model:**  
    - Add LangChain "Chat Model" node named "Chat Model via Command-R".  
    - Model: "command-r" (Cohere).  
    - Link chat trigger node output to this node.

15. **Add AI Agent Node:**  
    - Add LangChain "Agent" node named "AI Agent".  
    - Enable "return Intermediate Steps".  
    - Set system message defining assistant persona.  
    - Connect chat trigger and chat model nodes as inputs.

16. **Add Technology Report Tool:**  
    - Add LangChain Tool Code node named "Technology Innovation Report Tool".  
    - Define manual JSON input schema requiring "query" string.  
    - Dummy JS code returning "ok" to simulate tool usage.

17. **Add Conditional Branch:**  
    - Add "If" node named "If has Tool Call?".  
    - Condition: Check if intermediate steps contain a tool call to "Technology_Innovation_Report_Tool".  
    - True branch connects to next steps for vision-based processing.  
    - False branch connects to normal agent response.

18. **Extract Query:**  
    - Add "Set" node named "Get Query".  
    - Extract "tool" and "query" from first intermediate step from AI Agent’s output.

19. **Send Quick Confirmation Message:**  
    - Add LangChain "Chat" node named "Quick Confirmation".  
    - Message: "Please wait while I search the document..."

20. **Query Vector Store:**  
    - Add LangChain "Vector Store Qdrant" node named "Get Relevant Images".  
    - Configure collection: "visionRagExample".  
    - Prompt: Set from extracted query.  
    - Credentials: Qdrant API account.

21. **Aggregate Retrieved Results:**  
    - Add "Aggregate" node named "Aggregate Results".  
    - Aggregate all retrieved items into one array.

22. **Call Vision Model:**  
    - Add HTTP Request node named "Image Understanding via Command-A-Vision".  
    - POST to `https://api.cohere.com/v2/chat`.  
    - Model: "command-a-vision-07-2025".  
    - JSON Body includes user query text and images URLs from aggregated results.  
    - Authentication: Cohere API credentials.

23. **Respond with Vision Model Output:**  
    - Add LangChain "Chat" node named "Respond to Chat".  
    - Format message with returned text plus Markdown table of clickable image thumbnails.  
    - Enable memory connection.

24. **Manage Session Memory:**  
    - Add LangChain Memory Buffer Window node named "Simple Memory".  
    - Session key based on chat session id.

25. **Normal Agent Response Branch:**  
    - Add LangChain "Chat" node named "Respond to Chat1" for default agent response.  
    - Add Memory Buffer Window node named "Simple Memory1" for this branch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow demonstrates how to embed and retrieve document scans with image embeddings and vision LLMs for improved RAG performance on graphical content. | Overall workflow purpose and design.                                                                         |
| [Technology and Innovation Report 2025 PDF](https://unctad.org/system/files/official-document/tir2025_en.pdf)                                            | Source report used for image data.                                                                            |
| Instructions on splitting PDFs to images available in [Transcribing Bank Statements To Markdown Using Gemini Vision AI](https://n8n.io/workflows/2421-transcribing-bank-statements-to-markdown-using-gemini-vision-ai/). | Related workflow for PDF image extraction.                                                                    |
| [Cohere Embed v4 Announcement](https://cohere.com/blog/embed-4)                                                                                           | Details on Cohere’s multimodal embedding capabilities.                                                        |
| [Qdrant Community Node](https://github.com/qdrant/n8n-nodes-qdrant) and [Qdrant Vector Store](https://qdrant.tech)                                        | Vector store integration and background.                                                                      |
| [AI Agents Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)                                  | Guidance on using LangChain AI agents in n8n.                                                                 |
| [Cohere Command-A Vision Model Blog](https://cohere.com/blog/command-a-vision)                                                                            | Vision model background and usage.                                                                             |
| [Respond to Chat Node Documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat)                                       | How to customize chat responses with embedded images and formatting.                                          |
| Join [n8n Discord](https://discord.com/invite/XPKeKXeB7d) or [n8n Forum](https://community.n8n.io/) for support                                            | Community support resources.                                                                                   |

---

**Disclaimer:**  
The text provided is sourced exclusively from an automated workflow created with n8n, an integration and automation tool. All processing respects prevailing content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.

---