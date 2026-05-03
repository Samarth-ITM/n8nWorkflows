Generate LinkedIn posts using Google Gemini, MongoDB Atlas, Google Drive and Sheets

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-using-google-gemini--mongodb-atlas--google-drive-and-sheets-13782


# Generate LinkedIn posts using Google Gemini, MongoDB Atlas, Google Drive and Sheets

# Workflow Reference: LinkedIn Post Generator (Gemini, MongoDB & Google Workspace)

This document provides a technical analysis and reproduction guide for the n8n workflow designed to automate the creation of LinkedIn content. The system leverages Retrieval-Augmented Generation (RAG) by combining Google Gemini’s AI capabilities with a MongoDB Atlas vector database and Google Sheets for structured data access.

---

### 1. Workflow Overview

The workflow is designed to streamline LinkedIn content creation by using existing successful posts as a reference for tone, style, and structure. It operates through two primary functional paths:

*   **1.1 Data Ingestion & Embedding (Asynchronous):** Monitors a Google Drive folder for new reference files (CSV/Text), generates vector embeddings using Gemini, and stores them in MongoDB Atlas.
*   **1.2 AI Content Generation (On-Demand):** Triggered via a chat interface, it uses two distinct methods to generate content:
    *   **Vector Search Method:** Semantic retrieval of "thematically similar" posts to match a specific vibe.
    *   **Direct Sheet Tool Method:** Precise retrieval of data from Google Sheets for factual accuracy.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion Phase
This block handles the long-term memory of the AI by converting static documents into searchable vectors.

*   **Nodes Involved:** `Google Drive Trigger`, `Download file`, `MongoDB Vector Store Inserter`, `Embeddings Google Gemini`, `Default Data Loader`.
*   **Node Details:**
    *   **Google Drive Trigger:** Polls a specific folder (`1Yabi9e0BkBarkD945Yk_3WQQl-c7BgKX`) every minute for file updates.
    *   **Download file:** Downloads the detected file content as binary data.
    *   **MongoDB Vector Store Inserter:** Inserts the processed data into the `n8n_rag_data` collection using the `data_index`.
    *   **Embeddings Google Gemini:** Uses `models/gemini-embedding-001` to transform text into numerical vectors.
    *   **Default Data Loader:** Configured in `binaryMode` to parse the file downloaded from Drive.
*   **Edge Cases:** Large files may hit timeout limits during embedding; missing folder permissions will cause the trigger to fail.

#### 2.2 Method A: Vector Search (Semantic Retrieval)
This block enables the AI to "understand" the style and tone of previous posts.

*   **Nodes Involved:** `Knowledge Base Agent`, `MongoDB Vector Search`, `Google Gemini Chat Model`, `Embeddings Google Gemini1`, `Simple Memory`.
*   **Node Details:**
    *   **Knowledge Base Agent:** Acts as a "Senior LinkedIn Strategist." It uses the Vector Search tool to find context.
    *   **MongoDB Vector Search:** Functions as a tool (`n8n_rag`) that the agent calls to retrieve documentation based on semantic similarity.
    *   **Google Gemini Chat Model:** The core LLM processing the prompt and retrieved context.
    *   **Simple Memory:** A buffer that stores the current conversation history to maintain context during the chat.

#### 2.3 Method B: Direct Sheet Tool
This block provides the AI with a "hard" link to structured data, ideal for specific facts or templates.

*   **Nodes Involved:** `Knowledge Base Agent1`, `When chat message received`, `Google Sheets Tool`, `Google Gemini Chat Model1`, `Simple Memory1`.
*   **Node Details:**
    *   **When chat message received:** The entry point for the user to interact with the ghostwriter agent.
    *   **Knowledge Base Agent1:** An "Expert LinkedIn Ghostwriter" persona.
    *   **Google Sheets Tool:** Provides direct read access to a specific Spreadsheet ID. It allows the agent to pull exact rows for high-accuracy requirements.
    *   **Simple Memory1:** Maintains a window of past interactions for the ghostwriter agent.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Google Drive Trigger** | Google Drive Trigger | Entry point (Ingestion) | None | Download file | 1. DATA INGESTION PHASE: Watches folder for new post examples. |
| **Download file** | Google Drive | Data Retrieval | Google Drive Trigger | MongoDB Inserter | 1. DATA INGESTION PHASE: Watches folder for new post examples. |
| **MongoDB Vector Store Inserter** | MongoDB Vector Store | Data Storage | Download file | None | 1. DATA INGESTION PHASE: Stores text in MongoDB Atlas. |
| **Embeddings Google Gemini** | Gemini Embeddings | AI Processing | None | MongoDB Inserter | 1. DATA INGESTION PHASE: Embeds text for long-term memory. |
| **Default Data Loader** | Document Loader | Data Formatting | None | MongoDB Inserter | 1. DATA INGESTION PHASE: Handles binary to text conversion. |
| **Knowledge Base Agent** | AI Agent | Style Matching | None | None | 2. METHOD A: VECTOR SEARCH. Finds thematically similar posts. |
| **MongoDB Vector Search** | MongoDB Vector Store | Semantic Tool | None | Knowledge Base Agent | 2. METHOD A: VECTOR SEARCH. Best for matching 'Vibe' and 'Tone'. |
| **Google Gemini Chat Model** | Gemini Chat | AI Logic | None | Knowledge Base Agent | 2. METHOD A: VECTOR SEARCH. |
| **Simple Memory** | Memory Buffer | Chat Context | None | Knowledge Base Agent | 2. METHOD A: VECTOR SEARCH. |
| **When chat message received** | Chat Trigger | Entry point (Chat) | None | Agent 1 | 3. METHOD B: DIRECT SHEET TOOL. |
| **Knowledge Base Agent1** | AI Agent | Content Creation | Chat Trigger | None | 3. METHOD B: DIRECT SHEET TOOL. Pulls specific rows. |
| **Google Sheets Tool** | Google Sheets Tool | Data Tool | None | Agent 1 | 3. METHOD B: DIRECT SHEET TOOL. Ensuring 100% accuracy. |
| **Google Gemini Chat Model1** | Gemini Chat | AI Logic | None | Agent 1 | 3. METHOD B: DIRECT SHEET TOOL. |
| **Simple Memory1** | Memory Buffer | Chat Context | None | Agent 1 | 3. METHOD B: DIRECT SHEET TOOL. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Storage Setup:**
    *   Create a **MongoDB Atlas** cluster. Create a database and a collection named `n8n_rag_data`.
    *   In Atlas, create a Vector Search Index named `data_index` on the collection.
2.  **Ingestion Logic:**
    *   Add a **Google Drive Trigger** node. Set to "File Updated" and select your reference folder.
    *   Connect a **Google Drive** node (Download operation) using the File ID from the trigger.
    *   Connect a **MongoDB Vector Store** node. Set Action to "Insert".
    *   Attach the **Embeddings Google Gemini** node (model: `gemini-embedding-001`) and **Default Data Loader** (set to Binary Mode) to the MongoDB Inserter.
3.  **Agent A (Vector-Based):**
    *   Create an **AI Agent** node. Set the System Prompt to "LinkedIn Content Strategist".
    *   Add a **MongoDB Vector Store** node to the agent's Tool input. Set Action to "Retrieve as Tool". Name the tool `n8n_rag`.
    *   Attach **Google Gemini Chat Model**, **Embeddings Google Gemini**, and **Window Buffer Memory** to the agent.
4.  **Agent B (Sheet-Based):**
    *   Create a **Chat Trigger** node.
    *   Create a second **AI Agent** node. Connect the Chat Trigger to it.
    *   Add a **Google Sheets Tool** node to the agent's Tool input. Provide the Spreadsheet ID of your content repository.
    *   Attach **Google Gemini Chat Model** and **Window Buffer Memory** to this agent.
5.  **Credentials:**
    *   Configure OAuth2 for Google Drive and Google Sheets.
    *   Configure API Key for Google Gemini (PaLM).
    *   Configure Connection String for MongoDB.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Method A vs Method B** | Use Method A for creative "vibe" matching and Method B for factual data extraction from sheets. |
| **Vector Indexing** | Ensure the `vectorIndexName` in n8n exactly matches the index name in MongoDB Atlas UI. |
| **Google Gemini Model** | The workflow uses `gemini-embedding-001`. Ensure your API key has access to the Gemini Pro and Embedding models. |