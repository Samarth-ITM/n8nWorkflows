Create a Context-Aware Slack Assistant with Google Sheets, Pinecone & GPT-4o

https://n8nworkflows.xyz/workflows/create-a-context-aware-slack-assistant-with-google-sheets--pinecone---gpt-4o-6020


# Create a Context-Aware Slack Assistant with Google Sheets, Pinecone & GPT-4o

### 1. Workflow Overview

This workflow, titled **"Create a Context-Aware Slack Assistant with Google Sheets, Pinecone & GPT-4o"**, is designed to enhance Slack interactions by integrating contextual knowledge retrieval and AI-powered responses. It listens to Slack messages, fetches user profiles, retrieves contextually relevant data from Google Sheets and Pinecone vector stores using semantic embeddings, and generates intelligent replies through Azure OpenAI GPT-4o models. The workflow is structured into three main logical blocks:

- **1.1 Input Reception and User Context Retrieval**  
  Listens for Slack messages and retrieves user profile information to personalize interactions.

- **1.2 Contextual Data Retrieval and Embedding Processing**  
  Uses Azure OpenAI embeddings to vectorize data, queries Pinecone vector stores, and fetches relevant rows from Google Sheets, combining multiple knowledge sources.

- **1.3 AI Agent Response Generation and Output Processing**  
  Utilizes LangChain AI agents with GPT-4o, Cohere rerankers, and structured output parsers to generate, rank, and format context-aware responses, finally updating vector stores and preparing outputs for Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and User Context Retrieval

**Overview:**  
This block initiates the workflow by triggering on incoming Slack messages, followed by fetching detailed user profile data to enrich subsequent AI processing.

**Nodes Involved:**  
- Slack Trigger  
- Get a user's profile  
- AI Agent1

**Node Details:**

- **Slack Trigger**  
  - *Type:* Slack Trigger  
  - *Role:* Listens for Slack message events to initiate the workflow.  
  - *Configuration:* Default Slack webhook without specific filters, triggers on any Slack message event.  
  - *Input:* Slack event webhook  
  - *Output:* Slack message metadata to next node  
  - *Edge Cases:* Slack authorization errors, webhook misconfiguration, or no message payload.  
  - *Version:* 1

- **Get a user's profile**  
  - *Type:* Slack API node  
  - *Role:* Retrieves detailed profile information of the Slack user who sent the message.  
  - *Configuration:* Uses Slack OAuth2 credentials; fetches user information based on Slack Trigger input user ID.  
  - *Input:* Slack Trigger output (message user ID)  
  - *Output:* User profile data (name, email, etc.) to AI Agent1  
  - *Edge Cases:* User not found, Slack API rate limits, invalid tokens.  
  - *Version:* 2.3

- **AI Agent1**  
  - *Type:* LangChain AI Agent  
  - *Role:* Acts as the first AI processing layer, receiving user profile and message data for further contextual operations.  
  - *Configuration:* Connected downstream to Pinecone Vector Store5 node for vector data retrieval.  
  - *Input:* User profile data from "Get a user's profile"  
  - *Output:* Feeds into Pinecone vector store query  
  - *Version:* 2  
  - *Edge Cases:* AI model invocation failure, missing input data.

---

#### 1.2 Contextual Data Retrieval and Embedding Processing

**Overview:**  
This block is responsible for embedding textual data, querying Pinecone vector stores for semantic similarity, fetching related Google Sheets data, and reranking retrieved results to ensure relevance.

**Nodes Involved:**  
- Pinecone Vector Store5  
- Embeddings Azure OpenAI5  
- Pinecone Vector Store2  
- Embeddings Azure OpenAI2  
- Pinecone Vector Store3  
- Embeddings Azure OpenAI3  
- Get row(s) in sheet in Google Sheets  
- Reranker Cohere1  
- Reranker Cohere3

**Node Details:**

- **Pinecone Vector Store5**  
  - *Type:* LangChain Pinecone Vector Store  
  - *Role:* Retrieves or updates vectors in the primary Pinecone index associated with user context.  
  - *Configuration:* Uses Pinecone credentials and index configured for Slack assistant context.  
  - *Input:* Embeddings from "Embeddings Azure OpenAI5" and outputs to "AI Agent3" and "Reranker Cohere3".  
  - *Output:* Vector search results to AI Agent3 and re-ranker.  
  - *Version:* 1.3  
  - *Edge Cases:* Network issues, index not found, invalid embeddings.

- **Embeddings Azure OpenAI5**  
  - *Type:* Azure OpenAI Embeddings Node  
  - *Role:* Converts input text into semantic vectors for Pinecone storage/search.  
  - *Configuration:* Uses Azure OpenAI API with embedding model credentials.  
  - *Input:* Text input from prior nodes or manual input for embedding.  
  - *Output:* Embeddings to Pinecone Vector Store5.  
  - *Version:* 1  
  - *Edge Cases:* API quota limits, malformed input.

- **Pinecone Vector Store2**  
  - *Type:* LangChain Pinecone Vector Store  
  - *Role:* Secondary Pinecone index used for additional context retrieval.  
  - *Configuration:* Separate Pinecone index or namespace for diverse context.  
  - *Input:* Embeddings from "Embeddings Azure OpenAI2" and outputs to AI Agent1 and reranker.  
  - *Output:* Vector search results.  
  - *Version:* 1.3  
  - *Edge Cases:* Similar to Pinecone Vector Store5.

- **Embeddings Azure OpenAI2**  
  - *Type:* Azure OpenAI Embeddings Node  
  - *Role:* Same as Embeddings Azure OpenAI5 but targeting Pinecone Vector Store2.  
  - *Version:* 1

- **Pinecone Vector Store3**  
  - *Type:* LangChain Pinecone Vector Store  
  - *Role:* Tertiary vector store for storing or retrieving embeddings after final AI processing.  
  - *Input:* From "Edit Fields1" node.  
  - *Output:* Continues the data pipeline or storage.  
  - *Version:* 1.3

- **Embeddings Azure OpenAI3**  
  - *Type:* Azure OpenAI Embeddings Node  
  - *Role:* Embeds text data for Pinecone Vector Store3.  
  - *Version:* 1

- **Get row(s) in sheet in Google Sheets**  
  - *Type:* Google Sheets Tool  
  - *Role:* Queries Google Sheets to return rows that provide additional context or knowledge base data.  
  - *Configuration:* Uses Google Sheets OAuth2 credentials, likely with filtering based on Slack input or AI agent needs.  
  - *Input:* From AI Agent1 as part of tool integration.  
  - *Output:* Rows sent back to AI Agent1 for contextual enrichment.  
  - *Version:* 4.6  
  - *Edge Cases:* Google API quota or permission errors, empty results.

- **Reranker Cohere1**  
  - *Type:* LangChain Cohere Reranker  
  - *Role:* Reorders or scores vector search results for improved relevance before AI consumption.  
  - *Input:* Receives results from Pinecone Vector Store2 and sends improved ranking to AI Agent1.  
  - *Version:* 1

- **Reranker Cohere3**  
  - *Type:* LangChain Cohere Reranker  
  - *Role:* Similar to Reranker Cohere1 but applied on Pinecone Vector Store5 results feeding AI Agent3.  
  - *Version:* 1

---

#### 1.3 AI Agent Response Generation and Output Processing

**Overview:**  
This block manages the generation of AI responses using Azure OpenAI GPT-4o chat models, applies output parsers for structured data, performs auto-fixing of outputs, and updates vector stores with new embeddings. It culminates in preparing the final data for Slack or further processing.

**Nodes Involved:**  
- AI Agent3  
- Azure OpenAI Chat Model3  
- Auto-fixing Output Parser  
- Structured Output Parser1  
- Structured Output Parser2  
- Edit Fields1  
- Pinecone Vector Store3

**Node Details:**

- **AI Agent3**  
  - *Type:* LangChain AI Agent  
  - *Role:* Main AI response generator combining reranked context and user input to produce answers.  
  - *Input:* From Pinecone Vector Store5 and Auto-fixing Output Parser outputs.  
  - *Output:* To "Edit Fields1".  
  - *Version:* 2  
  - *Edge Cases:* Model timeout, malformed input.

- **Azure OpenAI Chat Model3**  
  - *Type:* LangChain Azure OpenAI Chat Model  
  - *Role:* GPT-4o chat model used as the language backend for AI Agent3.  
  - *Input:* Connected as AI languageModel for AI Agent3 and Auto-fixing Output Parser.  
  - *Version:* 1

- **Auto-fixing Output Parser**  
  - *Type:* LangChain Auto-fixing Output Parser  
  - *Role:* Automatically corrects AI output structure to conform to expectations.  
  - *Input:* From Azure OpenAI Chat Model3, outputs to AI Agent3 and Structured Output Parser2.  
  - *Version:* 1  
  - *Edge Cases:* Parsing failure, unexpected output format.

- **Structured Output Parser1**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI Agent1 outputs into structured data formats.  
  - *Input:* From AI Agent1.  
  - *Output:* To AI Agent1 for downstream processing.  
  - *Version:* 1.3

- **Structured Output Parser2**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses auto-fixed outputs for AI Agent3.  
  - *Input:* From Auto-fixing Output Parser.  
  - *Output:* To Auto-fixing Output Parser for validation.  
  - *Version:* 1.3

- **Edit Fields1**  
  - *Type:* Set Node (Field Editor)  
  - *Role:* Modifies or adds fields in data payload before final storage or output.  
  - *Input:* From AI Agent3  
  - *Output:* To Pinecone Vector Store3  
  - *Version:* 3.4  
  - *Edge Cases:* Incorrect field mapping or missing data.

- **Pinecone Vector Store3**  
  - *Role:* Final storage or update of vectors after AI output processing.  
  - *Version:* 1.3

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                                | Input Node(s)            | Output Node(s)           | Sticky Note                                      |
|---------------------------|--------------------------------------|------------------------------------------------|--------------------------|--------------------------|-------------------------------------------------|
| Slack Trigger             | Slack Trigger                        | Initiates workflow on Slack message            | —                        | Get a user's profile     |                                                 |
| Get a user's profile       | Slack API Node                      | Fetches profile info of message sender         | Slack Trigger            | AI Agent1                |                                                 |
| AI Agent1                 | LangChain AI Agent                  | Processes initial AI logic with user context   | Get a user's profile     | Pinecone Vector Store5   |                                                 |
| Pinecone Vector Store5     | LangChain Pinecone Vector Store    | Vector retrieval/storage for context            | Embeddings Azure OpenAI5 | AI Agent3, Reranker Cohere3 |                                                 |
| Embeddings Azure OpenAI5   | Azure OpenAI Embeddings             | Embeds text for Pinecone Vector Store5         | —                        | Pinecone Vector Store5   |                                                 |
| Pinecone Vector Store2     | LangChain Pinecone Vector Store    | Secondary vector store query for AI Agent1     | Embeddings Azure OpenAI2 | AI Agent1, Reranker Cohere1 |                                                 |
| Embeddings Azure OpenAI2   | Azure OpenAI Embeddings             | Embeds text for Pinecone Vector Store2         | —                        | Pinecone Vector Store2   |                                                 |
| Pinecone Vector Store3     | LangChain Pinecone Vector Store    | Final vector storage after processing           | Edit Fields1             | —                        |                                                 |
| Embeddings Azure OpenAI3   | Azure OpenAI Embeddings             | Embeds text for Pinecone Vector Store3         | —                        | Pinecone Vector Store3   |                                                 |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool              | Retrieves relevant sheet rows for context       | AI Agent1                | AI Agent1                |                                                 |
| Reranker Cohere1           | LangChain Cohere Reranker          | Reranks Pinecone Vector Store2 results          | Pinecone Vector Store2   | AI Agent1                |                                                 |
| Reranker Cohere3           | LangChain Cohere Reranker          | Reranks Pinecone Vector Store5 results          | Pinecone Vector Store5   | AI Agent3                |                                                 |
| AI Agent3                 | LangChain AI Agent                  | Generates final AI response                       | Pinecone Vector Store5, Auto-fixing Output Parser | Edit Fields1             |                                                 |
| Azure OpenAI Chat Model1   | LangChain Azure OpenAI Model       | Language model for AI Agent1                      | —                        | AI Agent1                |                                                 |
| Azure OpenAI Chat Model3   | LangChain Azure OpenAI Model       | GPT-4o chat model for AI Agent3 and output parser | —                        | AI Agent3, Auto-fixing Output Parser |                                                 |
| Auto-fixing Output Parser  | LangChain Output Parser Autofixing | Corrects AI output structure                      | Azure OpenAI Chat Model3 | AI Agent3, Structured Output Parser2 |                                                 |
| Structured Output Parser1  | LangChain Structured Output Parser | Parses AI Agent1 output into structured format   | AI Agent1                | AI Agent1                |                                                 |
| Structured Output Parser2  | LangChain Structured Output Parser | Parses auto-fixed output for AI Agent3           | Auto-fixing Output Parser | Auto-fixing Output Parser |                                                 |
| Edit Fields1              | Set Node                          | Edits/sets fields before final vector storage    | AI Agent3                | Pinecone Vector Store3   |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger node**  
   - Type: Slack Trigger  
   - Configure with Slack credentials and event webhook to listen to Slack message events.

2. **Add "Get a user's profile" Slack node**  
   - Type: Slack API node  
   - Use Slack OAuth2 credentials.  
   - Configure to fetch user profile using user ID from Slack Trigger.

3. **Insert AI Agent1 node**  
   - Type: LangChain AI Agent (version 2)  
   - Connect input from "Get a user's profile".  
   - No specific parameter overrides; default agent setup.  
   - Connect output to Pinecone Vector Store5.

4. **Set up Pinecone Vector Store5 node**  
   - Type: LangChain Pinecone Vector Store (version 1.3)  
   - Provide Pinecone API credentials and index for main context.  
   - Connect input from Embeddings Azure OpenAI5.  
   - Connect output to AI Agent3 and Reranker Cohere3.

5. **Create Embeddings Azure OpenAI5 node**  
   - Type: Azure OpenAI Embeddings (version 1)  
   - Configure with Azure OpenAI API key, select embedding model (e.g., text-embedding-ada-002).  
   - Connect output to Pinecone Vector Store5.

6. **Create Pinecone Vector Store2 node**  
   - Same as Pinecone Vector Store5 but for secondary context index.  
   - Connect input from Embeddings Azure OpenAI2 and output to AI Agent1 and Reranker Cohere1.

7. **Create Embeddings Azure OpenAI2 node**  
   - Configure similarly to Embeddings Azure OpenAI5 but for Vector Store2.

8. **Add Google Sheets node "Get row(s) in sheet in Google Sheets"**  
   - Configure Google OAuth2 credentials.  
   - Specify spreadsheet and sheet to query.  
   - Connect input from AI Agent1 as a tool node.  
   - Output connects back to AI Agent1 for context enrichment.

9. **Add Reranker Cohere1 node**  
   - Type: LangChain Cohere Reranker (version 1)  
   - Configure with Cohere API key.  
   - Connect input from Pinecone Vector Store2 and output to AI Agent1.

10. **Add Reranker Cohere3 node**  
    - Configure similarly to Reranker Cohere1.  
    - Connect input from Pinecone Vector Store5 and output to AI Agent3.

11. **Add AI Agent3 node**  
    - LangChain AI Agent version 2.  
    - Connect inputs from Pinecone Vector Store5 and Auto-fixing Output Parser.  
    - Output to Edit Fields1 node.

12. **Add Azure OpenAI Chat Model3 node**  
    - Type: LangChain Azure OpenAI Chat Model (version 1)  
    - Configure with Azure OpenAI GPT-4o chat model credentials.  
    - Connect as languageModel input to AI Agent3 and Auto-fixing Output Parser.

13. **Add Auto-fixing Output Parser node**  
    - Type: LangChain Auto-fixing Output Parser (version 1)  
    - Connect input from Azure OpenAI Chat Model3.  
    - Output to AI Agent3 and Structured Output Parser2.

14. **Add Structured Output Parser1 node**  
    - Type: LangChain Structured Output Parser (version 1.3)  
    - Connect input from AI Agent1 and output back to AI Agent1.

15. **Add Structured Output Parser2 node**  
    - Type: LangChain Structured Output Parser (version 1.3)  
    - Connect input from Auto-fixing Output Parser and output back to Auto-fixing Output Parser.

16. **Add Edit Fields1 node (Set Node)**  
    - Type: Set Node (version 3.4)  
    - Configure to add or modify fields in the AI Agent3 output as required for vector storage or Slack output.  
    - Connect input from AI Agent3.  
    - Output connects to Pinecone Vector Store3.

17. **Add Pinecone Vector Store3 node**  
    - Configure with Pinecone API credentials for tertiary storage.  
    - Connect input from Edit Fields1.

18. **Set up all credentials:**  
    - Slack OAuth2 credentials for Slack nodes.  
    - Azure OpenAI API keys for embeddings and chat models.  
    - Pinecone API keys and indices for vector stores.  
    - Cohere API key for reranker nodes.  
    - Google OAuth2 credentials for Google Sheets node.

19. **Connect all nodes as per the input-output mapping:**  
    - Slack Trigger → Get a user's profile → AI Agent1 → Google Sheets, Pinecone Vector Store2 → Reranker Cohere1 → AI Agent1 (loop).  
    - Embeddings nodes feed into respective Pinecone vector stores.  
    - Pinecone Vector Store5 → Reranker Cohere3 → AI Agent3 → Edit Fields1 → Pinecone Vector Store3.  
    - Azure OpenAI Chat Model3 → Auto-fixing Output Parser → AI Agent3 and Structured Output Parser2.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The workflow integrates GPT-4o via Azure OpenAI for context-aware AI responses in Slack conversations.        | Azure OpenAI documentation: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/             |
| Pinecone is used as a vector database to store and query semantic embeddings for knowledge retrieval.         | Pinecone docs: https://www.pinecone.io/docs/                                                               |
| Cohere rerankers improve relevance of retrieved contexts before AI processing.                                | Cohere Reranker API: https://docs.cohere.ai/api-reference/rerank                                        |
| Google Sheets acts as an external knowledge base to ground AI responses with structured data.                 | Google Sheets API: https://developers.google.com/sheets/api                                                  |
| LangChain nodes enable chaining of AI models, embeddings, and vector stores for modular AI workflows.         | LangChain n8n integration guide: https://n8n.io/integrations/langchain                                      |
| Slack OAuth2 credentials and webhook setup must be done carefully to ensure smooth triggering and API access. | Slack API docs: https://api.slack.com/apis                                                                    |

---

**Disclaimer:** The provided description and analysis are based exclusively on the given n8n workflow JSON and do not contain any illegal, offensive, or protected content. All data processed within this workflow is legal and publicly accessible.