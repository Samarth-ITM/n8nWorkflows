Analyze, classify, and summarize Gmail with OpenAI RAG and Google Sheets

https://n8nworkflows.xyz/workflows/analyze--classify--and-summarize-gmail-with-openai-rag-and-google-sheets-13568


# Analyze, classify, and summarize Gmail with OpenAI RAG and Google Sheets

# Workflow Reference: Gmail Analysis, Classification, and RAG Summarization

This document provides a technical breakdown of an n8n workflow designed to automate the processing of Gmail messages. It uses Retrieval-Augmented Generation (RAG) to classify emails against a dynamic taxonomy, stores results in Google Sheets, and provides audio notifications via Telegram.

---

### 1. Workflow Overview

The workflow is an intelligent email management system that goes beyond simple filtering. It uses an AI Agent to interpret email content based on a "source of truth" taxonomy stored in Google Sheets.

**Logical Blocks:**
*   **1.1 Email Ingestion & Preprocessing:** Polls Gmail for new messages and extracts clean text content.
*   **1.2 Vector Database Initialization:** A manual trigger path that loads existing classification examples into an in-memory vector store.
*   **1.3 AI Analysis (RAG):** An OpenAI-powered agent retrieves similar classification examples to categorize, summarize, and extract keywords from the email.
*   **1.4 Data Persistence:** Saves the enriched data into a master tracking spreadsheet.
*   **1.5 Adaptive Taxonomy Learning:** Detects if the AI created a new category/subcategory and automatically updates the reference spreadsheet.
*   **1.6 Audio Notification:** Converts the summary into speech and sends it to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Ingestion and Preprocessing
This block handles the entry point for live data.
*   **Nodes Involved:** `Gmail Trigger`, `Fetch full Gmail message`, `Normalize email fields`.
*   **Node Details:**
    *   **Gmail Trigger:** Polls every minute for new emails.
    *   **Fetch full Gmail message:** Uses the Message ID to retrieve the complete body.
    *   **Normalize email fields:** Uses a `Set` node to map sender, date, subject, and strips HTML tags from the content using `.removeTags()`.

#### 2.2 Vector Database Initialization
Required for the RAG system to function; it populates the AI's "memory."
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Get row(s) in sheet`, `Prepare taxonomy documents`, `Create embeddings (OpenAI)`, `Vector store — taxonomy index`.
*   **Node Details:**
    *   **Get row(s) in sheet:** Pulls classification samples from a specific Google Sheet ([Link to Spreadsheet](https://docs.google.com/spreadsheets/d/16kSdJJ0imPSq2A7gIhueuEvjwdcvjRaCMYGg-u2QdFs/edit#gid=67988996)).
    *   **Prepare taxonomy documents:** Combines `subject` and `email_text` into a single document for the vector store, attaching category/subcategory as metadata.
    *   **Vector store — taxonomy index:** Stores the embeddings in memory under the key `email_tagging`.

#### 2.3 AI Analysis using RAG
The core intelligence of the workflow.
*   **Nodes Involved:** `Analyze email with RAG agent`, `OpenAI Chat Model`, `Vector store retriever (RAG tool)`.
*   **Node Details:**
    *   **Analyze email with RAG agent:** A Tool-using agent. The **System Prompt** defines strict rules: it must return valid JSON, prefer retrieved categories over general knowledge, and set boolean flags for new categories.
    *   **Vector store retriever:** Acts as the "knowledge base." It provides the Agent with the most relevant example from the spreadsheet to ensure consistent classification.
    *   **OpenAI Chat Model:** Configured with `gpt-4.1-mini` (or similar) at temperature 0.7.

#### 2.4 Storage and Logic
Processing the AI's output and branching based on results.
*   **Nodes Involved:** `Parse AI analysis result`, `Store processed email`, `Check for new taxonomy items`.
*   **Node Details:**
    *   **Parse AI analysis result:** Extracts the JSON string from the AI output into individual n8n fields using `.parseJson()`.
    *   **Store processed email:** Appends the final analyzed data (summary, category, keywords) to the tracking sheet.
    *   **Check for new taxonomy items:** An `IF` node checking if `new_category` or `new_subcategory` is `true`.

#### 2.5 Automatic Taxonomy Learning
*   **Nodes Involved:** `Load taxonomy dataset`, `Get next taxonomy ID`, `Append new taxonomy entry`, `Notify admin about taxonomy update`.
*   **Node Details:**
    *   **Get next taxonomy ID:** A Python Code node that calculates the next incremental ID from the existing spreadsheet rows.
    *   **Append new taxonomy entry:** Adds the new classification to the reference sheet so the AI can use it for future emails.

#### 2.6 Audio Summary Notification
*   **Nodes Involved:** `Generate audio summary`, `Send audio summary to Telegram`.
*   **Node Details:**
    *   **Generate audio summary:** Uses OpenAI’s Text-to-Speech (TTS) to convert the AI-generated summary into an audio file.
    *   **Send audio summary to Telegram:** Dispatches the audio file to a specific chat with a formatted caption containing sender, date, and subject.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Gmail Trigger | gmailTrigger | Trigger | (None) | Fetch full Gmail message | Email ingestion and preprocessing |
| Fetch full Gmail message | gmail | Data Retrieval | Gmail Trigger | Normalize email fields | Email ingestion and preprocessing |
| Normalize email fields | set | Transformation | Fetch full Gmail message | Analyze email with RAG agent | Email ingestion and preprocessing |
| Analyze email with RAG agent | agent | AI Processing | Normalize email fields | Parse AI analysis result | AI analysis using RAG classification |
| OpenAI Chat Model | lmChatOpenAi | AI Model | (None) | Analyze email with RAG agent | AI analysis using RAG classification |
| Vector store retriever | vectorStoreInMemory | AI Tool | Create embeddings | Analyze email with RAG agent | AI analysis using RAG classification |
| Parse AI analysis result | set | Transformation | Analyze email with RAG agent | Store processed email | Store processed emails |
| Store processed email | googleSheets | Data Storage | Parse AI analysis result | Check for new taxonomy items | Store processed emails |
| Check for new taxonomy items | if | Logic | Store processed email | Load taxonomy dataset, Generate audio summary | New Category/Subcategory? |
| Load taxonomy dataset | googleSheets | Data Retrieval | Check for new taxonomy items | Get next taxonomy ID | Automatic taxonomy learning |
| Get next taxonomy ID | code | Logic (Python) | Load taxonomy dataset | Append new taxonomy entry | Automatic taxonomy learning |
| Append new taxonomy entry | googleSheets | Data Storage | Get next taxonomy ID | Notify admin... | Automatic taxonomy learning |
| Notify admin... | telegram | Notification | Append new taxonomy entry | Generate audio summary, Get row(s) in sheet | Automatic taxonomy learning |
| Generate audio summary | openAi | AI Audio | Check for new taxonomy items, Notify admin... | Send audio summary to Telegram | Audio summary notification |
| Send audio summary to Telegram | telegram | Notification | Generate audio summary | (None) | Audio summary notification |
| Get row(s) in sheet | googleSheets | Data Retrieval | Manual Trigger, Notify admin... | Vector store — taxonomy index | Vector database initialization (manual first run) |
| Vector store — taxonomy index | vectorStoreInMemory | Data Storage | Get row(s) in sheet, Prepare taxonomy documents | (None) | Vector database initialization (manual first run) |
| Prepare taxonomy documents | documentDefaultDataLoader | Transformation | Get row(s) in sheet | Vector store — taxonomy index | Create embeddings for taxonomy knowledge base |
| Create embeddings (OpenAI) | embeddingsOpenAi | AI Model | (None) | Vector store, Vector retriever | Create embeddings for taxonomy knowledge base |
| When clicking ‘Execute workflow’ | manualTrigger | Trigger | (None) | Get row(s) in sheet | Vector database initialization (manual first run) |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with two tabs: `email` (for logs) and `tagging_sample` (for taxonomy).
    *   Ensure the `tagging_sample` tab has columns: `id`, `subject`, `email_text`, `category`, `subcategory`, `keywords`.

2.  **Initialize the Knowledge Base:**
    *   Add a **Manual Trigger** connected to a **Google Sheets (Get Rows)** node.
    *   Connect a **Document Loader** (set to "Expression Data") using the sheet rows as input.
    *   Connect the loader to an **In-Memory Vector Store** node (Action: Insert).
    *   Attach an **OpenAI Embeddings** node to the Vector Store.
    *   *Critical:* Set a unique `Memory Key` (e.g., `email_tagging`) in the Vector Store.

3.  **Setup Ingestion:**
    *   Create a **Gmail Trigger** (Polling).
    *   Add a **Gmail Node** (Operation: Get) to fetch the message by ID.
    *   Add a **Set Node** to clean the body text (e.g., `{{ $json.html.removeTags() }}`).

4.  **Setup AI Agent:**
    *   Add an **AI Agent** node. Set the prompt to receive the normalized email fields.
    *   In the Agent's "Tools," add the **In-Memory Vector Store** (Action: Retrieve as Tool).
    *   Set the `Top K` to 1. Use the same `Memory Key` as step 2.
    *   Connect an **OpenAI Chat Model** (GPT-4o or GPT-4o-mini).

5.  **Setup Post-Processing:**
    *   Add a **Set Node** using `.parseJson()` to turn the Agent's text output back into structured data fields.
    *   Add a **Google Sheets (Append)** node to log the processed email.

6.  **Setup Learning Logic:**
    *   Add an **IF Node** to check if `new_category == "true"` or `new_subcategory == "true"`.
    *   If **True**: 
        *   Fetch the current taxonomy sheet.
        *   Use a **Code Node** (Python) to calculate `max(ids) + 1`.
        *   Append the new category details to the taxonomy sheet.
        *   Send a **Telegram Message** informing the admin of the new category.

7.  **Setup Notification:**
    *   Add an **OpenAI Node** (Resource: Audio, Operation: Generate).
    *   Pass the summarized text into the audio generator.
    *   Add a **Telegram Node** (Operation: Send Audio) and toggle "Binary Data" to `true`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Taxonomy Source Spreadsheet | [Google Sheets Link](https://docs.google.com/spreadsheets/d/16kSdJJ0imPSq2A7gIhueuEvjwdcvjRaCMYGg-u2QdFs/edit#gid=67988996) |
| Initialization Requirement | You **must** run the manual trigger once before activating the workflow to populate the vector store. |
| Credentials Required | Gmail (OAuth2), OpenAI (API Key), Google Sheets (OAuth2), Telegram (Bot Token). |
| Data Privacy | The workflow uses an In-Memory vector store; data persists in Google Sheets, not the vector store itself across service restarts. |