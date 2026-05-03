Run a self-hosted multi-channel AI assistant with Claude, Gemini and Gmail

https://n8nworkflows.xyz/workflows/run-a-self-hosted-multi-channel-ai-assistant-with-claude--gemini-and-gmail-13717


# Run a self-hosted multi-channel AI assistant with Claude, Gemini and Gmail

# Run a self-hosted multi-channel AI assistant with Claude, Gemini and Gmail

### 1. Workflow Overview

This workflow implements **n8nClaw**, a sophisticated, self-hosted AI agent designed to act as a personal "Second Brain." It centralizes communications across Telegram, WhatsApp, and Gmail, providing a unified interface for task management, web research, and document handling.

The logic is structured into several functional layers:
- **1.1 Input Processing & Triggers:** Captures messages from Telegram, WhatsApp (via Evolution API), and Gmail. It includes a media processing pipeline to handle voice, images, and documents using Google Gemini.
- **1.2 Core Orchestration:** The central "n8nClaw" agent (Claude 3.5 Sonnet) manages conversation state, user profiles, and decides when to delegate tasks.
- **1.3 Toolset & Sub-Agents:** A hierarchical system of specialized agents for research, email management, document processing (Google Drive/Docs), and general task execution.
- **1.4 Long-Term Memory (LTM):** An asynchronous pipeline that periodicially summarizes chat histories and stores them in a Supabase vector database for Retrieval-Augmented Generation (RAG).
- **1.5 Task Management:** Persistent storage of tasks and subtasks using n8n Data Tables.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggers & Input Normalization
This block captures incoming data from various sources and formats it into a standard JSON object containing `user_message`, `system_prompt_details`, and `last_channel`.

- **Nodes Involved:** Telegram Trigger, Webhook (WhatsApp), Gmail Trigger, Hourly Heartbeat, Filter nodes, and various "Get row(s)" nodes.
- **Node Details:**
    - **Telegram Trigger:** Listens for "message" updates.
    - **Filter/Filter1:** Validates that incoming messages are from the authorized user/ID to prevent unauthorized access.
    - **Gmail Trigger:** Polls every minute for new emails.
    - **Hourly Heartbeat:** A Schedule Trigger that prompts the agent to check for pending tasks autonomously every hour.
    - **Get row(s) [1-4]:** Fetches the user's specific profile (soul, name, preferences) from the "ClawdBot Init" Data Table using a `username` filter.

#### 2.2 Media Handling (Telegram)
Converts non-text inputs from Telegram into text that the AI agent can understand.

- **Nodes Involved:** Switch1, Get a file [0-2], Transcribe a recording, Analyze an image, Analyze document.
- **Node Details:**
    - **Switch1:** Routes input based on the presence of `voice`, `photo`, or `document` fields.
    - **Gemini Nodes:** Uses `gemini-2.5-flash` for transcription and document analysis, and `nano-banana-pro-preview` for image analysis.
    - **Configuration:** Takes binary data from Telegram and outputs text into the `user_message` variable for the next block.

#### 2.3 Core AI Agent (n8nClaw)
The "brain" of the workflow. It processes the normalized input and interacts with the memory and toolset.

- **Nodes Involved:** n8nClaw (Agent), OpenRouter Chat Model (Claude 3.5 Sonnet), Postgres Chat Memory, Supabase Vector Store (Retrieval).
- **Node Details:**
    - **n8nClaw:** An AI Agent node configured with a comprehensive system message. It is instructed to use tools for task logging and to delegate work to sub-agents.
    - **Postgres Chat Memory:** Maintains a rolling window of the last 15 messages for short-term conversation context.
    - **Supabase Vector Store (Tool):** Configured as a tool to "get info about the user" by querying past summarized conversations.

#### 2.4 Specialized Sub-Agents & Tools
These nodes represent the capabilities n8nClaw can call upon.

- **Research Agent:** Uses Gemini 3 Flash to search Tavily and Wikipedia.
- **Email Manager:** Uses Claude Haiku to manage Gmail (Read, Reply, Send, Delete, Search, Mark as Read).
- **Document Manager:** Uses Claude Haiku to manage Google Docs and Google Drive (Create, Update, Move, Search).
- **Workers (1-3):** A tier of agents ranging from Claude Haiku (simple) to Claude Opus (complex reasoning) for parallel task processing.
- **Data Table Tools:** Four nodes for managing the "ClawdBot Tasks" and "ClawdBot Subtasks" tables (Get/Upsert).

#### 2.5 Long-Term Memory Pipeline
A separate flow that turns chat logs into searchable knowledge.

- **Nodes Involved:** Data Loader (Schedule), Execute a SQL query, Aggregate, Basic LLM Chain, Supabase Vector Store (Insert).
- **Node Details:**
    - **Execute SQL:** Queries the `n8n_chat_histories` table for IDs greater than the `last_vector_id`.
    - **Basic LLM Chain:** Summarizes the aggregated conversation logs into a concise format.
    - **Supabase Vector Store:** Generates OpenAI embeddings for the summary and stores it in the `documents` table.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram Trigger | Telegram Trigger | Entry Point | None | Filter | Triggers & Input Processing |
| Webhook | Webhook | WhatsApp Entry | None | Filter1 | Triggers & Input Processing |
| Gmail Trigger | Gmail Trigger | Email Entry | None | Get row(s)4 | Triggers & Input Processing |
| Hourly Heartbeat | Schedule Trigger | Auto-Tasking | None | Get row(s)3 | Triggers & Input Processing |
| n8nClaw | AI Agent | Orchestrator | Edit Fields [0-3] | Switch | Core AI Agent — n8nClaw |
| Switch1 | Switch | Media Routing | Filter | Get a file [0-2] | Media Handling (Telegram) |
| Research Agent | AI Agent Tool | Web Research | n8nClaw | None | Sub-Agents |
| Email Manager | AI Agent Tool | Gmail Ops | n8nClaw | None | Sub-Agents |
| Document Manager | AI Agent Tool | Doc Ops | n8nClaw, Workers | None | Sub-Agents |
| Upsert Task | Data Table Tool | Task Persistence | n8nClaw | None | Tools (Data Tables & Vector Store) |
| Supabase Vector Store | Vector Store | Memory Storage | Basic LLM Chain | Update row(s) | Long-Term Memory Pipeline |
| Switch | Switch | Output Routing | n8nClaw | Send a text / Enviar texto | Output Routing |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    - Create three n8n Data Tables: `ClawdBot Init` (columns: username, soul, user, last_channel, last_vector_id), `ClawdBot Tasks`, and `ClawdBot Subtasks`.
    - Set up a Postgres database for `n8n_chat_histories`.
    - Set up a Supabase project with a `documents` table and `match_documents` function for vector storage.

2.  **Triggers:**
    - Add a **Telegram Trigger** (set to `message`).
    - Add a **Webhook Node** for WhatsApp (requires Evolution API or similar).
    - Add a **Gmail Trigger** (polling).
    - Add a **Schedule Trigger** (1-hour interval).

3.  **Media Processing:**
    - Connect the Telegram Trigger to a **Switch** node. Create branches for voice, photo, and document.
    - Add **Telegram (File)** nodes to download media.
    - Add **Google Gemini** nodes for transcription and vision analysis.

4.  **The Agent Core:**
    - Add the **n8n AI Agent** node. Name it "n8nClaw".
    - Connect an **OpenRouter Chat Model** (Claude 3.5 Sonnet).
    - Connect **Postgres Chat Memory** (Session Key = `YOUR_USERNAME`).
    - Create the System Message: Detail the "soul" of the agent and instructions for tool usage.

5.  **Tool Integration:**
    - Attach **Data Table Tool** nodes for Tasks and Subtasks to the Agent's tool input.
    - Attach the **Agent Tool** nodes (Research, Email, Document Manager). Each sub-agent requires its own Model and Memory nodes.
    - For Gmail and Google Drive tools, configure **OAuth2 credentials** for Google.

6.  **Response Routing:**
    - Connect the Agent output to a **Switch** node.
    - Branch 1: `last_channel == "telegram"` -> **Telegram Send Text** node.
    - Branch 2: `last_channel == "whatsapp"` -> **Evolution API** node.

7.  **Memory Pipeline:**
    - Create a separate chain starting with a **Schedule Trigger**.
    - Use a **Postgres Node** to fetch recent chat history.
    - Use an **Aggregate Node** to bundle messages.
    - Use a **Basic LLM Chain** with **Claude Haiku** to summarize.
    - Insert the summary into **Supabase Vector Store**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video Walkthrough | [YouTube Link](https://www.youtube.com/watch?v=Yfo34yco5Oo) |
| Evolution API | Required for the WhatsApp Webhook integration |
| OpenRouter | Used to access Anthropic and Google models via a single API |
| Supabase Setup | Requires `pgvector` enabled and a specific `match_documents` RPC function |