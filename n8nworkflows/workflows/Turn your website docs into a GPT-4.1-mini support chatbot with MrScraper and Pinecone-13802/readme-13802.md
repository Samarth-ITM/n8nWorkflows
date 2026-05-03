Turn your website docs into a GPT-4.1-mini support chatbot with MrScraper and Pinecone

https://n8nworkflows.xyz/workflows/turn-your-website-docs-into-a-gpt-4-1-mini-support-chatbot-with-mrscraper-and-pinecone-13802


# Turn your website docs into a GPT-4.1-mini support chatbot with MrScraper and Pinecone

# Reference Document: Website Docs to GPT-4.1-mini Support Chatbot

This document provides a technical breakdown of the n8n workflow designed to transform website documentation into a smart AI support chatbot using MrScraper for web crawling, OpenAI for embeddings, and Pinecone as a vector database.

---

### 1. Workflow Overview

The workflow is a complete pipeline for **Retrieval-Augmented Generation (RAG)**. It is divided into two primary functional phases:
1.  **The Indexing Pipeline:** Automatically crawls a website, extracts content, chunks the text, generates vector embeddings, and stores them in Pinecone.
2.  **The Chat Interface:** A real-time conversational agent that uses the stored documentation to answer user queries via a web-compatible chat trigger.

#### Logical Blocks:
*   **1.1 URL Discovery:** Identification of all relevant documentation links using a site map agent.
*   **1.2 Content Extraction:** Batch processing of URLs to extract clean text/markdown content.
*   **1.3 Data Preparation:** Chunking long texts into manageable pieces and attaching metadata (URL, Title).
*   **1.4 Vector Indexing:** Converting text chunks into vectors and upserting them into Pinecone.
*   **1.5 Support Chat Agent:** A conversational interface that retrieves context from Pinecone to provide factual answers.

---

### 2. Block-by-Block Analysis

#### 2.1 URL Discovery (Phase 1)
*   **Overview:** Identifies the structure of the target site and collects specific URLs to be indexed.
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `MrScraper - Discover URLs (Map Agent)`, `Extract Url`.
*   **Node Details:**
    *   **MrScraper (Map Agent):** Uses a `scraperId` to crawl the `siteRoot`. Configured with include/exclude patterns (e.g., include `/docs/`, exclude `/assets/`).
    *   **Extract Url (Code):** Parses the JSON response from MrScraper to isolate the `data.urls` array and returns each URL as an individual n8n item.
*   **Potential Failures:** Invalid `scraperId`, site blocking the crawler, or empty discovery results if patterns are too restrictive.

#### 2.2 Page Extraction (Phase 2)
*   **Overview:** Visits each discovered URL to pull the actual text content while managing rate limits.
*   **Nodes Involved:** `Batch URLs (Controlled Crawl)`, `MrScraper - Extract Page Content (General Agent)`, `Pick Content Field`.
*   **Node Details:**
    *   **Batch URLs:** A `SplitInBatches` node set to a size of 10 (adjustable) to prevent overloading the scraper or the target site.
    *   **MrScraper (General Agent):** Extracts structured data (title, body text) from a single URL.
    *   **Pick Content Field (Code):** Normalizes the output. It selects the best available text field (e.g., `content` or `markdown`) and preserves the source URL.
*   **Potential Failures:** Timeouts on large pages, 404 errors on discovered links, or extraction of "empty" pages (navigation-only).

#### 2.3 Chunking & Embedding (Phase 3)
*   **Overview:** Breaks down large documents into smaller, overlapping segments to ensure the AI can find specific information efficiently.
*   **Nodes Involved:** `Chunk Text for Embeddings`, `Docs Loader (Chunks → Documents)`, `OpenAI Embeddings (Indexing)`, `Pinecone Vector Store (Upsert)`.
*   **Node Details:**
    *   **Chunk Text (Code):** Implements a sliding window strategy. `chunkSize`: 1100 chars, `overlap`: 180 chars. It filters out chunks shorter than 80 characters.
    *   **Docs Loader:** Formats the JavaScript objects into LangChain Document objects and maps metadata (URL, Title, index).
    *   **OpenAI Embeddings:** Converts text into high-dimensional vectors. Requires consistent model selection (e.g., `text-embedding-3-small`).
    *   **Pinecone Vector Store:** Performs the "Upsert" operation into a specific namespace.
*   **Edge Cases:** Very long pages creating hundreds of chunks; metadata exceeding Pinecone's size limits.

#### 2.4 Support Chat Interface (Phase 4)
*   **Overview:** The "live" part of the workflow that handles user interaction.
*   **Nodes Involved:** `Chat` (Trigger), `Support Chat Agent`, `OpenAI Chat`, `Chat Memory (Short)`, `Pinecone Retriever Tool`, `OpenAI Embeddings (Chat)`.
*   **Node Details:**
    *   **Support Chat Agent:** A LangChain Agent (Type: Tool Agent) with a system message identifying it as "Nova".
    *   **Pinecone Retriever Tool:** Configured to fetch the `topK` (8) most relevant chunks from the database.
    *   **Chat Memory:** A `Buffer Window Memory` set to 8 interactions to maintain conversation context.
    *   **OpenAI Chat:** Set to `gpt-4.1-mini` for cost-efficient and fast reasoning.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **When clicking...** | Manual Trigger | Manual Entry Point | None | MrScraper Map | - |
| **MrScraper Map** | mrscraper | URL Discovery | Manual Trigger | Extract Url | Phase 1 — URL Discovery |
| **Extract Url** | Code | Data Parsing | MrScraper Map | Batch URLs | Phase 1 — URL Discovery |
| **Batch URLs** | splitInBatches | Rate Limiting | Extract Url, Pinecone Upsert | MrScraper General | Phase 2 — Page Extraction |
| **MrScraper General**| mrscraper | Content Extraction | Batch URLs | Pick Content Field | Phase 2 — Page Extraction |
| **Pick Content Field**| Code | Data Normalization | MrScraper General | Chunk Text | Phase 2 — Page Extraction |
| **Chunk Text** | Code | Text Segmentation | Pick Content Field | Pinecone Upsert | Phase 3 — Chunking + Embedding |
| **OpenAI (Index)** | embeddingsOpenAi | Vectorization | Pinecone Upsert | Pinecone Upsert | Phase 3 — Chunking + Embedding |
| **Docs Loader** | documentDefault | Metadata Mapping | None | Pinecone Upsert | - |
| **Pinecone Upsert** | vectorStorePinecone | Database Storage | Chunk Text, OpenAI, Docs Loader | Batch URLs | Phase 4 — Vector Store |
| **Chat** | chat | Chat Webhook | None | Support Agent | Phase 4 — Chat Endpoint |
| **Support Agent** | agent | Core Logic | Chat, OpenAI Chat, Memory, Tool | None | Phase 4 — Answer rules |
| **OpenAI Chat** | lmChatOpenAi | LLM Model | Support Agent | Support Agent | Phase 4 — Answer rules |
| **Chat Memory** | memoryBuffer | Context History | Support Agent | Support Agent | Phase 4 — Chat Endpoint |
| **Pinecone Tool** | vectorStorePinecone | Context Retrieval | Support Agent | Support Agent | Phase 4 — Set up retrieval |
| **OpenAI (Chat)** | embeddingsOpenAi | Query Vectorization| Pinecone Tool | Pinecone Tool | Phase 4 — Vector Store |

---

### 4. Reproducing the Workflow from Scratch

1.  **Setup Credentials:**
    *   **OpenAI:** API Key for GPT-4.1-mini and Embeddings.
    *   **MrScraper:** API Key and two specific Scraper IDs (Map Agent and General Agent).
    *   **Pinecone:** API Key, Environment, and Index Name (Dimensions must match OpenAI embeddings).

2.  **Build the Indexing Pipeline:**
    *   Add a **Manual Trigger** connected to the **MrScraper Map Agent**.
    *   Add a **Code Node** ("Extract Url") using a `map()` function to split the `data.urls` array into individual items.
    *   Connect to **Split In Batches** (Size 10).
    *   Inside the loop, add **MrScraper General Agent** (Pass the URL from the loop).
    *   Add a **Code Node** ("Pick Content Field") to isolate the text and title.
    *   Add a **Code Node** ("Chunk Text") to split content into 1100-character segments with 180-character overlap.
    *   Add **Pinecone Vector Store (Insert)**. Connect an **OpenAI Embeddings** node and a **Docs Loader** (to map metadata like `url` and `title`).
    *   Connect the output of Pinecone back to the **Split In Batches** node to continue the loop.

3.  **Build the Chat Pipeline:**
    *   Add a **Chat Trigger** node.
    *   Add an **AI Agent** node.
    *   Connect **OpenAI Chat** (Model: `gpt-4.1-mini`) to the Agent.
    *   Connect **Window Buffer Memory** to the Agent.
    *   Add a **Pinecone Vector Store (Retrieve as Tool)** node.
    *   Connect a separate **OpenAI Embeddings** node to this tool.
    *   **Configure Agent System Message:** Instruct the agent to only use retrieved context and provide source URLs from metadata.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **MrScraper Dashboard** | Required to create the Map and General scrapers before running. |
| **Pinecone Indexing** | Dimensions: 1536 for `text-embedding-3-small` or 3072 for `text-embedding-3-large`. |
| **Anti-Hallucination** | System prompt: "Never invent pricing, policies, or guarantees." |
| **Efficiency** | The "Batch URLs" step is critical to avoid 429 rate limit errors from MrScraper or your own website. |