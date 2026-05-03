Monitor Google Maps Reviews with GPT-4o Sentiment Analysis & Telegram RAG Agent using Pinecone

https://n8nworkflows.xyz/workflows/monitor-google-maps-reviews-with-gpt-4o-sentiment-analysis---telegram-rag-agent-using-pinecone-11381


# Monitor Google Maps Reviews with GPT-4o Sentiment Analysis & Telegram RAG Agent using Pinecone

### 1. Workflow Overview

This workflow automates the monitoring of Google Maps reviews for a specified location, performing sentiment analysis using GPT-4o, and interacts with users via Telegram through a Retrieval-Augmented Generation (RAG) agent leveraging Pinecone vector search. It is designed to:

- Periodically scrape new Google Maps reviews.
- Process and format review data.
- Generate embeddings for semantic search and knowledge management in Pinecone.
- Analyze reviews with AI models for sentiment and insights.
- Notify users via Telegram with alerts and respond to queries interactively.
- Support manual execution and scheduled triggers.

The workflow is logically grouped into the following blocks:

- **1.1 Scheduled and Manual Triggering:** Initiates the workflow automatically at midnight or weekly, or manually.
- **1.2 Google Maps Scraper Control:** Starts scraping reviews, waits for completion, and fetches results.
- **1.3 Review Data Processing:** Formats scraped review data and logs it to Google Sheets.
- **1.4 Pinecone Vector Store Ingestion and Search:** Embeds review data, ingests into Pinecone, and performs vector searches.
- **1.5 AI Analysis and Agent Interaction:** Uses OpenAI GPT models for sentiment analysis, review interpretation, and handles Telegram-triggered queries.
- **1.6 Telegram Messaging and Alerts:** Sends messages and alerts to Telegram users and confirms actions.
- **1.7 Configuration and Setup:** Handles user-specific configuration such as chat IDs, map links, and namespaces.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled and Manual Triggering

**Overview:**  
This block initiates the workflow either automatically at defined times (midnight daily and every Sunday at 11 pm) or manually via a trigger node.

**Nodes Involved:**  
- Trigger at midnight  
- Every Sunday at 11pm  
- When clicking "Execute"

**Node Details:**

- **Trigger at midnight**  
  - Type: Schedule Trigger  
  - Role: Starts workflow automatically every midnight.  
  - Connections: Outputs to "⚠️ CONFIGURATION".  
  - Edge Cases: Timezone configuration issues could cause missed triggers.

- **Every Sunday at 11pm**  
  - Type: Schedule Trigger  
  - Role: Weekly workflow trigger for secondary configuration.  
  - Connections: Outputs to "⚠️ CONFIGURATION2".  
  - Edge Cases: Same as above.

- **When clicking "Execute"**  
  - Type: Manual Trigger  
  - Role: Allows user to manually start the workflow for testing or immediate execution.  
  - Connections: Outputs to "⚠️ CONFIGURATION".  
  - Edge Cases: None significant; manual use only.

---

#### 1.2 Google Maps Scraper Control

**Overview:**  
Manages scraping of Google Maps reviews, including starting the scraper, waiting for completion, and fetching the scraped data.

**Nodes Involved:**  
- ⚠️ CONFIGURATION  
- Start Google Maps Scraper  
- Wait for Scraper  
- Get Dataset Results

**Node Details:**

- **⚠️ CONFIGURATION**  
  - Type: Set  
  - Role: Holds user-specific parameters like Chat ID, Map Link, and Namespace for scraping.  
  - Notes: "SETUP: Enter your Chat ID, Map Link, and Namespace here."  
  - Connections: Outputs to "Start Google Maps Scraper".  
  - Edge Cases: Misconfiguration leads to invalid scraping or no data returned.

- **Start Google Maps Scraper**  
  - Type: HTTP Request  
  - Role: Sends request to trigger Google Maps scraping process.  
  - Input: Configuration node parameters.  
  - Output: Triggers "Wait for Scraper".  
  - Edge Cases: Network failures, API rate limits, or scraping service downtime.

- **Wait for Scraper**  
  - Type: Wait  
  - Role: Pauses workflow until scraper completes.  
  - Output: Proceeds to "Get Dataset Results".  
  - Edge Cases: Timeout if scraper takes too long; no data returned.

- **Get Dataset Results**  
  - Type: HTTP Request  
  - Role: Retrieves the scraped reviews data.  
  - Output: Passes data to "Format Data (Reviews)".  
  - Edge Cases: Empty or malformed data responses.

---

#### 1.3 Review Data Processing

**Overview:**  
Formats the raw review data and logs it into Google Sheets for record-keeping or further analysis.

**Nodes Involved:**  
- Format Data (Reviews)  
- Log data

**Node Details:**

- **Format Data (Reviews)**  
  - Type: Set  
  - Role: Structures raw review data into a clean, consistent format.  
  - Input: Data from "Get Dataset Results".  
  - Output: To "Log data".  
  - Edge Cases: Data inconsistencies causing formatting errors.

- **Log data**  
  - Type: Google Sheets  
  - Role: Inserts processed review data into a Google Sheets document.  
  - Input: Formatted review data.  
  - Edge Cases: API quota limits, authentication errors.

---

#### 1.4 Pinecone Vector Store Ingestion and Search

**Overview:**  
Generates embeddings from review data, ingests them into Pinecone vector database, and performs semantic search to support AI agent queries.

**Nodes Involved:**  
- Default Data Loader  
- Embeddings OpenAI  
- Pinecone Ingest  
- Wait 2s  
- Embeddings OpenAI1  
- Pinecone Search Tool

**Node Details:**

- **Default Data Loader**  
  - Type: Document Default Data Loader  
  - Role: Loads documents for embedding ingestion.  
  - Output: To "Pinecone Ingest".  
  - Edge Cases: Document loading failures or empty inputs.

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings  
  - Role: Generates vector embeddings from the review text using OpenAI API.  
  - Input: Review text or documents.  
  - Output: To "Pinecone Ingest".  
  - Edge Cases: API errors, rate limits.

- **Pinecone Ingest**  
  - Type: Pinecone Vector Store  
  - Role: Inserts embeddings into Pinecone namespace.  
  - Input: Embeddings from OpenAI node.  
  - Output: To "Wait 2s".  
  - Edge Cases: Connection failures, authentication errors.

- **Wait 2s**  
  - Type: Wait  
  - Role: Pauses workflow briefly to avoid rate limits or ensure ingestion completes.  
  - Output: Loops back to "Loop Over Items".  
  - Edge Cases: None significant.

- **Embeddings OpenAI1**  
  - Type: OpenAI Embeddings  
  - Role: Generates embeddings for input queries to perform vector search.  
  - Output: To "Pinecone Search Tool".  
  - Edge Cases: Same as Embeddings OpenAI.

- **Pinecone Search Tool**  
  - Type: Pinecone Vector Store Search  
  - Role: Performs similarity search in Pinecone to retrieve relevant documents for AI agent.  
  - Output: To AI agent node.  
  - Edge Cases: Search failures, empty results.

---

#### 1.5 AI Analysis and Agent Interaction

**Overview:**  
Applies GPT models to analyze reviews for sentiment and insights, and uses an AI agent to answer user queries from Telegram with context from Pinecone.

**Nodes Involved:**  
- AI Review Analyst  
- Structured Output Parser  
- GPT 5 mini  
- Aggregate Data  
- Loop Over Items  
- AI Data Analyst  
- OpenAI Chat Model

**Node Details:**

- **AI Review Analyst**  
  - Type: Langchain Agent  
  - Role: Processes review data and produces structured sentiment analysis.  
  - Input: Output of "Structured Output Parser".  
  - Output: To "Send Alert".  
  - Edge Cases: Model latency, interpretation errors.

- **Structured Output Parser**  
  - Type: Structured Output Parser  
  - Role: Parses AI model output into structured format for agent.  
  - Output: To "AI Review Analyst".  
  - Edge Cases: Parsing issues if AI output is malformed.

- **GPT 5 mini**  
  - Type: OpenAI Chat Model  
  - Role: Provides language model for "AI Review Analyst" agent.  
  - Edge Cases: API limits, model version dependencies.

- **Aggregate Data**  
  - Type: Aggregate  
  - Role: Aggregates batch data items during processing.  
  - Output: To "AI Review Analyst".  
  - Edge Cases: Data aggregation errors with large batches.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Splits large datasets into manageable batches for processing and ingestion.  
  - Output: To "Aggregate Data" and "Pinecone Ingest".  
  - Edge Cases: Batch size misconfiguration causing performance issues.

- **AI Data Analyst**  
  - Type: Langchain Agent  
  - Role: Handles Telegram-triggered user queries, uses OpenAI chat and Pinecone search for RAG.  
  - Input: From "⚠️ CONFIGURATION1" and "OpenAI Chat Model", "Pinecone Search Tool".  
  - Output: Sends processed text to "Send a text message".  
  - Edge Cases: Authentication issues, query understanding failures.

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model  
  - Role: Language model used for AI Data Analyst agent.  
  - Edge Cases: Same as GPT 5 mini.

---

#### 1.6 Telegram Messaging and Alerts

**Overview:**  
Sends alerts and responses to Telegram users, and confirms action completions.

**Nodes Involved:**  
- Send Alert  
- Send a text message  
- Confirmation text  
- Telegram Trigger

**Node Details:**

- **Send Alert**  
  - Type: Telegram  
  - Role: Sends alert messages based on AI review analysis.  
  - Input: From "AI Review Analyst".  
  - Edge Cases: Telegram API rate limits, chat ID errors.

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends AI agent responses to user queries.  
  - Input: From "AI Data Analyst".  
  - Edge Cases: Same as above.

- **Confirmation text**  
  - Type: Telegram  
  - Role: Sends confirmation messages after namespace clearing or other operations.  
  - Input: From "Empty Namespace (Keep DB)".  
  - Edge Cases: Chat ID misconfiguration.

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages to initiate AI Data Analyst processing.  
  - Output: To "⚠️ CONFIGURATION1".  
  - Edge Cases: Webhook setup errors, missing permissions.

---

#### 1.7 Configuration and Setup

**Overview:**  
Contains nodes for user configuration and auxiliary operations such as namespace clearing.

**Nodes Involved:**  
- ⚠️ CONFIGURATION1  
- ⚠️ CONFIGURATION2  
- Empty Namespace (Keep DB)  
- Sticky Notes (multiple)

**Node Details:**

- **⚠️ CONFIGURATION1**  
  - Type: Set  
  - Role: Holds chat ID, map link, and namespace for Telegram-triggered AI queries.  
  - Output: To "AI Data Analyst".  
  - Edge Cases: Incorrect values break Telegram interaction.

- **⚠️ CONFIGURATION2**  
  - Type: Set  
  - Role: Holds chat ID, map link, and namespace for weekly cleanup or other scheduled tasks.  
  - Output: To "Empty Namespace (Keep DB)".  
  - Edge Cases: Misconfiguration may cause data loss or no cleanup.

- **Empty Namespace (Keep DB)**  
  - Type: HTTP Request  
  - Role: Sends request to clear or keep Pinecone namespace database.  
  - Output: To "Confirmation text".  
  - Edge Cases: Request failures, unintended data deletion.

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Provide setup instructions and contextual info for users.  
  - Notes: Several sticky notes remind to enter Chat ID, Map Link, Namespace, and other setup steps.

---

### 3. Summary Table

| Node Name                   | Node Type                            | Functional Role                           | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                        |
|-----------------------------|------------------------------------|-----------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------|
| Trigger at midnight          | Schedule Trigger                   | Daily automatic workflow initiation     |                              | ⚠️ CONFIGURATION              |                                                                   |
| Every Sunday at 11pm         | Schedule Trigger                   | Weekly automatic workflow initiation    |                              | ⚠️ CONFIGURATION2             |                                                                   |
| When clicking "Execute"      | Manual Trigger                    | Manual workflow start                    |                              | ⚠️ CONFIGURATION              |                                                                   |
| ⚠️ CONFIGURATION            | Set                              | User parameters for scraping             | Trigger at midnight, Manual   | Start Google Maps Scraper      | SETUP: Enter your Chat ID, Map Link, and Namespace here.          |
| Start Google Maps Scraper    | HTTP Request                     | Initiates Google Maps review scraping   | ⚠️ CONFIGURATION              | Wait for Scraper               |                                                                   |
| Wait for Scraper             | Wait                             | Waits for scraper to finish              | Start Google Maps Scraper     | Get Dataset Results            |                                                                   |
| Get Dataset Results          | HTTP Request                     | Retrieves scraped review data             | Wait for Scraper              | Format Data (Reviews)          |                                                                   |
| Format Data (Reviews)        | Set                              | Formats raw review data                   | Get Dataset Results           | Log data                      |                                                                   |
| Log data                    | Google Sheets                    | Logs formatted review data                | Format Data (Reviews)         | Wait 5s                       |                                                                   |
| Wait 5s                     | Wait                             | Pauses for rate limit or processing delay| Log data                     | Loop Over Items               |                                                                   |
| Loop Over Items              | Split In Batches                 | Splits data into batches for processing  | Wait 5s                      | Aggregate Data, Pinecone Ingest|                                                                   |
| Aggregate Data              | Aggregate                       | Aggregates batch data                      | Loop Over Items              | AI Review Analyst             |                                                                   |
| AI Review Analyst            | Langchain Agent                 | Performs sentiment analysis on reviews   | Aggregate Data, Structured Output Parser | Send Alert            |                                                                   |
| Structured Output Parser     | Structured Output Parser         | Parses AI output into structured format  | GPT 5 mini                   | AI Review Analyst             |                                                                   |
| GPT 5 mini                  | OpenAI Chat Model                | Language model for review analysis        |                             | AI Review Analyst             |                                                                   |
| Pinecone Ingest             | Pinecone Vector Store            | Inserts embeddings into Pinecone           | Embeddings OpenAI, Default Data Loader | Wait 2s                 |                                                                   |
| Wait 2s                     | Wait                             | Pauses post ingestion                      | Pinecone Ingest              | Loop Over Items               |                                                                   |
| Default Data Loader          | Document Default Data Loader     | Loads documents for embedding              |                             | Pinecone Ingest              |                                                                   |
| Embeddings OpenAI            | OpenAI Embeddings               | Generates embeddings from reviews          |                             | Pinecone Ingest              |                                                                   |
| Embeddings OpenAI1           | OpenAI Embeddings               | Embeds queries for vector search           |                             | Pinecone Search Tool         |                                                                   |
| Pinecone Search Tool         | Pinecone Vector Store Search    | Retrieves relevant documents via search   | Embeddings OpenAI1           | AI Data Analyst              |                                                                   |
| AI Data Analyst              | Langchain Agent                 | Handles Telegram queries with RAG          | Pinecone Search Tool, OpenAI Chat Model, ⚠️ CONFIGURATION1 | Send a text message |                                                                   |
| OpenAI Chat Model            | OpenAI Chat Model               | Language model for AI Data Analyst         |                             | AI Data Analyst              |                                                                   |
| Telegram Trigger             | Telegram Trigger                | Listens for Telegram user input            |                             | ⚠️ CONFIGURATION1            |                                                                   |
| ⚠️ CONFIGURATION1           | Set                              | User parameters for Telegram interaction  | Telegram Trigger             | AI Data Analyst              | SETUP: Enter your Chat ID, Map Link, and Namespace here.          |
| Send Alert                  | Telegram                       | Sends AI analysis alerts to Telegram       | AI Review Analyst            |                             |                                                                   |
| Send a text message          | Telegram                       | Sends AI agent responses to Telegram       | AI Data Analyst              |                             |                                                                   |
| ⚠️ CONFIGURATION2           | Set                              | User parameters for weekly tasks           | Every Sunday at 11pm         | Empty Namespace (Keep DB)     | SETUP: Enter your Chat ID, Map Link, and Namespace here.          |
| Empty Namespace (Keep DB)    | HTTP Request                   | Clears or maintains Pinecone namespace DB  | ⚠️ CONFIGURATION2            | Confirmation text            |                                                                   |
| Confirmation text            | Telegram                       | Confirms namespace clearing to user        | Empty Namespace (Keep DB)    |                             |                                                                   |
| Sticky Note (multiple)       | Sticky Note                    | Setup instructions and notes                |                             |                             | Multiple sticky notes remind setup of Chat ID, Map Link, Namespace |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node named "Trigger at midnight" for daily execution at 00:00.  
   - Add a **Schedule Trigger** node named "Every Sunday at 11pm" for weekly execution at 23:00 Sunday.  
   - Add a **Manual Trigger** node named "When clicking 'Execute'".

2. **Add Configuration Nodes:**  
   - Add a **Set** node "⚠️ CONFIGURATION" linked from "Trigger at midnight" and "When clicking 'Execute'". Configure parameters: Chat ID, Google Maps Link, Pinecone Namespace.  
   - Add a **Set** node "⚠️ CONFIGURATION2" linked from "Every Sunday at 11pm" with similar parameters for scheduled cleanup/maintenance.  
   - Add a **Set** node "⚠️ CONFIGURATION1" linked from "Telegram Trigger" (step 9) with parameters for Telegram chat interaction.

3. **Google Maps Scraper Control:**  
   - Add an **HTTP Request** node "Start Google Maps Scraper" connected from "⚠️ CONFIGURATION" to start review scraping. Configure with API endpoint and authentication as needed.  
   - Add a **Wait** node "Wait for Scraper" connected from scraper start node, configured with reasonable delay or webhook wait.  
   - Add an **HTTP Request** node "Get Dataset Results" connected after waiting, to retrieve scraped review data.

4. **Review Data Processing:**  
   - Add a **Set** node "Format Data (Reviews)" connected from "Get Dataset Results" to normalize and format the review data.  
   - Add a **Google Sheets** node "Log data" connected from formatted data to store reviews into a Google Sheet.

5. **Batch Processing and Ingestion:**  
   - Add a **Wait** node "Wait 5s" connected from "Log data" to throttle processing.  
   - Add a **Split In Batches** node "Loop Over Items" connected after wait, set batch size as needed.  
   - Add an **Aggregate** node "Aggregate Data" connected from "Loop Over Items" for data aggregation.  
   - Add a **Document Default Data Loader** "Default Data Loader" to load data for embeddings (connect as required).  
   - Add an **OpenAI Embeddings** node "Embeddings OpenAI" connected to "Default Data Loader" output to create vector embeddings.  
   - Add a **Pinecone Vector Store** node "Pinecone Ingest" connected from embeddings to insert vectors into Pinecone.  
   - Add a **Wait** node "Wait 2s" connected after ingestion to avoid rate limits, looping back to "Loop Over Items".

6. **AI Sentiment Analysis:**  
   - Add an **OpenAI Chat Model** node "GPT 5 mini" configured with GPT-4o or appropriate model.  
   - Add a **Structured Output Parser** node "Structured Output Parser" connected from the chat model to parse AI outputs.  
   - Add a **Langchain Agent** node "AI Review Analyst" connected from parser and aggregate node to analyze reviews.  
   - Add a **Telegram** node "Send Alert" connected from AI Review Analyst to send alert messages.

7. **Telegram Interaction and RAG Agent:**  
   - Add a **Telegram Trigger** node "Telegram Trigger" to listen for user messages.  
   - Connect "Telegram Trigger" to "⚠️ CONFIGURATION1" for Telegram-specific parameters.  
   - Add **OpenAI Chat Model** node "OpenAI Chat Model" for language modeling in RAG agent.  
   - Add an **OpenAI Embeddings** node "Embeddings OpenAI1" for query embeddings.  
   - Add a **Pinecone Vector Store** node "Pinecone Search Tool" connected from embeddings for vector search.  
   - Add a **Langchain Agent** node "AI Data Analyst" connected from configuration, chat model, and Pinecone search to handle queries.  
   - Add a **Telegram** node "Send a text message" connected from AI Data Analyst to reply on Telegram.

8. **Namespace Maintenance:**  
   - Connect "Every Sunday at 11pm" → "⚠️ CONFIGURATION2" → **HTTP Request** node "Empty Namespace (Keep DB)" to clear or maintain Pinecone namespace.  
   - Connect to **Telegram** node "Confirmation text" to confirm operation.

9. **Add Sticky Notes:**  
   - Add sticky notes near configuration nodes reminding users to input Chat ID, Map Link, and Namespace.  
   - Add any other notes for setup instructions as per original workflow.

10. **Credentials Setup:**  
    - Configure credentials for OpenAI API (API key with GPT-4o access).  
    - Configure credentials for Pinecone (API key, environment, index).  
    - Configure credentials for Telegram Bot (Bot token).  
    - Configure Google Sheets credentials (OAuth2 or API key).

11. **Connect Nodes as per described connections above** and verify the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Setup reminders: Enter your Chat ID, Map Link, and Namespace in configuration nodes ⚠️ CONFIGURATION | Inline sticky notes in workflow configuration nodes                |
| Uses GPT-4o for advanced sentiment analysis and AI chat capabilities                                | Requires OpenAI API key with GPT-4o access                         |
| Pinecone used as vector store for semantic search and RAG agent                                     | Requires Pinecone account and index setup                          |
| Telegram integration requires bot credentials and webhook setup                                    | Telegram Bot API documentation: https://core.telegram.org/bots/api|
| Google Sheets used for logging review data                                                          | Google Sheets API via OAuth2 or API key                            |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.