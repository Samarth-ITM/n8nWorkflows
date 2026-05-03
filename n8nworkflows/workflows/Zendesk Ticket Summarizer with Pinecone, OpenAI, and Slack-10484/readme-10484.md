Zendesk Ticket Summarizer with Pinecone, OpenAI, and Slack

https://n8nworkflows.xyz/workflows/zendesk-ticket-summarizer-with-pinecone--openai--and-slack-10484


# Zendesk Ticket Summarizer with Pinecone, OpenAI, and Slack

### 1. Workflow Overview

This workflow automates the daily summarization of Zendesk customer support tickets using vector search and AI summarization, then posts the results to a Slack channel. It is designed for support teams needing timely insights into recurring customer issues without manually reviewing every ticket.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Input Retrieval:** Daily trigger initiates the workflow and fetches Zendesk tickets created in the last 24 hours, with optional brand filtering.
- **1.2 Data Preparation and Vector Storage:** Tickets are transformed into structured documents with metadata, embedded using OpenAI embeddings, and stored in a Pinecone vector database with a specified namespace.
- **1.3 AI Processing and Summarization:** After a short wait, an AI agent queries the Pinecone vector store to generate a summary highlighting common complaints and their frequency across tickets.
- **1.4 Notification Output:** The generated summary is sent to a designated Slack channel for team review.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Input Retrieval

- **Overview:**  
  This block triggers the workflow daily at 10 AM and fetches all Zendesk tickets created in the previous 24 hours, optionally filtered by brand ID for targeted analysis.

- **Nodes Involved:**  
  - Daily trigger  
  - Get tickets  
  - Sticky Note (instructions)

- **Node Details:**

  - **Daily trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates workflow daily at 10 AM.  
    - *Configuration:* Set to trigger once a day, hour 10.  
    - *Connections:* Outputs to Get tickets node.  
    - *Failure Modes:* Trigger timing issues unlikely; ensure server time is correct.  
    - *Sticky Note:* "Trigger once a day at 10am."

  - **Get tickets**  
    - *Type:* Zendesk node  
    - *Role:* Retrieves tickets created within last 24 hours.  
    - *Configuration:* Query string uses expression to filter tickets by creation time and brand ID, sorting tickets descending by creation date, returns all tickets.  
    - *Key Expression:* `=created>{{$now.minus({hours: 24})}} brand:<brand_id_here>` (replace `<brand_id_here>` as needed)  
    - *Connections:* Outputs to Pinecone Vector Store - Tickets node.  
    - *Failure Modes:* API auth errors, rate limits, invalid query syntax, empty results if no tickets matched.  
    - *Sticky Note:* "Fetch tickets created in the last 24 hours from Zendesk. Make sure the query string contains the id of the brand if querying tickets for a specific brand."

---

#### 2.2 Data Preparation and Vector Storage

- **Overview:**  
  Converts raw Zendesk tickets into structured documents with metadata, generates embeddings via OpenAI, and inserts them into a Pinecone vector index for efficient retrieval.

- **Nodes Involved:**  
  - Pinecone Vector Store - Tickets  
  - Default Data Loader1  
  - Embeddings OpenAI  
  - Sticky Notes (mapping guidance, storage details)

- **Node Details:**

  - **Pinecone Vector Store - Tickets**  
    - *Type:* Pinecone Vector Store node (Insert mode)  
    - *Role:* Inserts ticket embeddings into Pinecone index and clears namespace before insertion.  
    - *Configuration:*  
      - Mode: Insert  
      - Clear namespace: True (to remove old data)  
      - Index: "n8n-zendesk-YOUR_BRAND_NAME_HERE" (replace with your actual index)  
      - Namespace: `<namespace_name_here>` (replace accordingly)  
    - *Connections:* Receives embeddings and documents from Embeddings OpenAI and Default Data Loader1 nodes; outputs to Wait1 node.  
    - *Failure Modes:* API connectivity, auth failures, namespace misconfiguration causing data loss or incorrect data insertion.  
    - *Sticky Note:* "Store tickets in a Pinecone index and namespace. Select the index and define a namespace in the node. Namespace gets created automatically. Clear namespace before storing the new tickets."

  - **Default Data Loader1**  
    - *Type:* Document Default Data Loader  
    - *Role:* Formats Zendesk ticket fields into a textual document with metadata for embedding.  
    - *Configuration:*  
      - JSON data expression consolidates fields: subject, description, device, OS from ticket JSON.  
      - Metadata includes URL and ticket creation date.  
    - *Key Expression:*  
      ```
      subject: {{ $json.raw_subject }} | description: {{ $json.description }} | device: {{ $json.fields[3].value }} | OS: {{ $json.fields[5].value }}
      ```  
    - *Connections:* Outputs formatted documents to Pinecone Vector Store - Tickets node as ai_document.  
    - *Failure Modes:* Expression errors if fields missing or incorrectly indexed; incomplete metadata if ticket data is inconsistent.  
    - *Sticky Note:* "Match Zendesk ticket fields to data and metadata in Pinecone"

  - **Embeddings OpenAI**  
    - *Type:* OpenAI Embeddings node  
    - *Role:* Converts documents into vector embeddings suitable for Pinecone insertion.  
    - *Configuration:* Default embedding options used; connected to OpenAI account credentials.  
    - *Connections:* Outputs embeddings to Pinecone Vector Store - Tickets node as ai_embedding.  
    - *Failure Modes:* API quota limits, network errors, invalid API key.  
    - *Sticky Note:* None directly.

---

#### 2.3 AI Processing and Summarization

- **Overview:**  
  After a short delay to ensure vector data consistency, an AI agent uses GPT-4 to query Pinecone vector store and generate a concise summary of common complaints, including counts of tickets mentioning each complaint.

- **Nodes Involved:**  
  - Wait1  
  - AI Agent - Summary generator  
  - Pinecone Vector Store2  
  - Embeddings OpenAI2  
  - OpenAI Chat Model1  
  - Sticky Notes (wait explanation, indexing consistency, AI agent description)

- **Node Details:**

  - **Wait1**  
    - *Type:* Wait node  
    - *Role:* Pauses workflow 15 seconds before AI summarization to ensure data is indexed in Pinecone.  
    - *Configuration:* Wait 15 seconds.  
    - *Connections:* Receives from Pinecone Vector Store - Tickets, outputs to AI Agent - Summary generator.  
    - *Failure Modes:* Delay too short causing incomplete data in vector store; too long increases workflow runtime unnecessarily.  
    - *Sticky Note:* "A short wait is needed before AI agent starts summarising the tickets stored in Pinecone."

  - **AI Agent - Summary generator**  
    - *Type:* Langchain Agent node  
    - *Role:* Uses AI to generate a summary based on tickets retrieved from Pinecone vector store.  
    - *Configuration:*  
      - Prompt instructs AI to summarize tickets highlighting main complaints and count frequencies.  
      - System message clarifies AI role: assist users by summarizing without reading tickets one-by-one.  
      - Connected to OpenAI Chat Model1 for language model and Pinecone Vector Store2 as tool.  
      - On error: continue with error output (non-blocking).  
    - *Key Expression:*  
      ```
      Create a summary of tickets available in the vector store. Highlight main complaints that are mentioned in multiple tickets. For each complaint, mention the number of different tickets that the complaint was raised in.
      ```  
    - *Connections:* Outputs summary text to Send to Slack channel node.  
    - *Failure Modes:* API errors, vector store query failures, prompt misconfiguration leading to poor summaries.  
    - *Sticky Note:* "AI agent to summarise tickets."

  - **Pinecone Vector Store2**  
    - *Type:* Pinecone Vector Store node (Retrieve-as-tool mode)  
    - *Role:* Serves as a query tool for the AI agent to retrieve relevant ticket data by similarity search.  
    - *Configuration:*  
      - Mode: Retrieve-as-tool  
      - topK: 200 (max number of results)  
      - Index and namespace same as Pinecone Vector Store - Tickets node (must match for consistency).  
      - Tool description explains purpose as vector store for ticket data.  
    - *Connections:* Connected as ai_tool input to AI Agent - Summary generator node.  
    - *Failure Modes:* Mismatched namespaces/indexes cause empty or wrong search results.  
    - *Sticky Note:* "Select the same index and namespace used in \"Pinecone Vector Store - Tickets\" node"

  - **Embeddings OpenAI2**  
    - *Type:* OpenAI Embeddings node  
    - *Role:* Provides embeddings for AI agent tools if needed.  
    - *Connections:* Connected as ai_embedding input to Pinecone Vector Store2 node.  
    - *Failure Modes:* Similar to Embeddings OpenAI.  
    - *Sticky Note:* None directly.

  - **OpenAI Chat Model1**  
    - *Type:* OpenAI Chat Model node (GPT-4)  
    - *Role:* Language model backend for AI Agent to generate summaries.  
    - *Configuration:* Uses GPT-4.1-mini model variant.  
    - *Connections:* Connected as ai_languageModel input to AI Agent - Summary generator.  
    - *Failure Modes:* API limits, model availability, credential issues.  
    - *Sticky Note:* "Select the same index and namespace used in \"Pinecone Vector Store - Tickets\" node" (this note also applies to model selection contextually)

---

#### 2.4 Notification Output

- **Overview:**  
  The AI-generated summary of Zendesk tickets is sent as a message to a configured Slack channel to inform the support team.

- **Nodes Involved:**  
  - Send to Slack channel

- **Node Details:**

  - **Send to Slack channel**  
    - *Type:* Slack node  
    - *Role:* Posts the summary message to a specific Slack channel.  
    - *Configuration:*  
      - Text message is dynamically set to the output from "AI Agent - Summary generator".  
      - Channel selected by ID (replace `"C1111111"` with your target channel ID).  
      - Uses Slack OAuth2 credentials.  
    - *Connections:* Receives input from AI Agent - Summary generator.  
    - *Failure Modes:* Slack API auth errors, invalid channel IDs, message formatting issues.  
    - *Sticky Note:* "Send summary to the Slack channel"

---

### 3. Summary Table

| Node Name                     | Node Type                                  | Functional Role                          | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                              |
|-------------------------------|--------------------------------------------|----------------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------|
| Daily trigger                 | Schedule Trigger                           | Initiate workflow daily at 10am        | -                                | Get tickets                      | Trigger once a day at 10am                                                                              |
| Get tickets                  | Zendesk                                    | Fetch tickets created last 24 hours    | Daily trigger                   | Pinecone Vector Store - Tickets  | Fetch tickets created in the last 24 hours from Zendesk. Make sure the query string contains the id of the brand if querying tickets for a specific brand. |
| Pinecone Vector Store - Tickets | Pinecone Vector Store (Insert mode)        | Store ticket embeddings in Pinecone    | Get tickets, Default Data Loader1, Embeddings OpenAI | Wait1                           | Store tickets in a Pinecone index and namespace. Select the index and define a namespace in the node. Namespace gets created automatically. Clear namespace before storing the new tickets. |
| Default Data Loader1          | Document Default Data Loader               | Format ticket data and metadata         | Get tickets                    | Pinecone Vector Store - Tickets  | Match Zendesk ticket fields to data and metadata in Pinecone                                           |
| Embeddings OpenAI            | OpenAI Embeddings                          | Generate embeddings for tickets         | Default Data Loader1           | Pinecone Vector Store - Tickets  |                                                                                                         |
| Wait1                       | Wait                                       | Pause 15 seconds before AI summarization | Pinecone Vector Store - Tickets | AI Agent - Summary generator    | A short wait is needed before AI agent starts summarising the tickets stored in Pinecone.                |
| AI Agent - Summary generator | Langchain Agent                            | Generate summary from tickets           | Wait1, OpenAI Chat Model1, Pinecone Vector Store2 | Send to Slack channel            | AI agent to summarise tickets.                                                                          |
| Pinecone Vector Store2       | Pinecone Vector Store (Retrieve-as-tool)  | Query tickets for AI agent retrieval    | Embeddings OpenAI2             | AI Agent - Summary generator    | Select the same index and namespace used in "Pinecone Vector Store - Tickets" node                       |
| Embeddings OpenAI2           | OpenAI Embeddings                          | Provide embeddings for Pinecone query   | -                              | Pinecone Vector Store2          |                                                                                                         |
| OpenAI Chat Model1           | OpenAI Chat Model (GPT-4)                  | Language model for AI agent              | -                              | AI Agent - Summary generator    | Select the same index and namespace used in "Pinecone Vector Store - Tickets" node                       |
| Send to Slack channel        | Slack                                     | Post summary message to Slack channel   | AI Agent - Summary generator   | -                              | Send summary to the Slack channel                                                                       |
| Sticky Note                  | Sticky Note                               | Instructional comments                   | -                              | -                              | Various notes as detailed in analysis sections                                                          |
| Sticky Note1                 | Sticky Note                               | Instructional comments                   | -                              | -                              | Various notes as detailed in analysis sections                                                          |
| Sticky Note2                 | Sticky Note                               | Instructional comments                   | -                              | -                              | Various notes as detailed in analysis sections                                                          |
| Sticky Note3                 | Sticky Note                               | Instructional comments                   | -                              | -                              | Various notes as detailed in analysis sections                                                          |
| Sticky Note4                 | Sticky Note                               | Instructional comments                   | -                              | -                              | Various notes as detailed in analysis sections                                                          |
| Sticky Note5                 | Sticky Note                               | Instructional comments                   | -                              | -                              | Various notes as detailed in analysis sections                                                          |
| Sticky Note6                 | Sticky Note                               | Project overview and setup instructions | -                              | -                              | See detailed content in section 5                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Daily trigger" node**  
   - Type: Schedule Trigger  
   - Set to trigger once daily at 10:00 AM.

2. **Create the "Get tickets" Zendesk node**  
   - Connect input from "Daily trigger" node.  
   - Operation: Get All tickets  
   - Query: Use expression to fetch tickets created in last 24 hours and filter by brand:  
     `=created>{{$now.minus({hours: 24})}} brand:<brand_id_here>` (replace `<brand_id_here>` with your Zendesk brand ID)  
   - Sort by `created_at` descending  
   - Return all results: true  
   - Configure Zendesk credentials.

3. **Create "Default Data Loader1" node**  
   - Connect input from "Get tickets" node.  
   - Type: Document Default Data Loader  
   - JSON Data (expression):  
     ```
     subject: {{ $json.raw_subject }} | description: {{ $json.description }} | device: {{ $json.fields[3].value }} | OS: {{ $json.fields[5].value }}
     ```  
   - Metadata fields: Add `url` from `{{$json.url}}` and `created_at` from `{{$json.created_at}}`.

4. **Create "Embeddings OpenAI" node**  
   - Connect input from "Default Data Loader1" node.  
   - Type: OpenAI Embeddings  
   - Use your OpenAI API credentials.

5. **Create "Pinecone Vector Store - Tickets" node**  
   - Connect main input from "Get tickets" (for raw data) and ai_document input from "Default Data Loader1", ai_embedding input from "Embeddings OpenAI".  
   - Mode: Insert  
   - Clear namespace: true  
   - Pinecone index: `"n8n-zendesk-YOUR_BRAND_NAME_HERE"` (replace with your index)  
   - Namespace: `<namespace_name_here>` (replace accordingly)  
   - Configure Pinecone API credentials.

6. **Create "Wait1" node**  
   - Connect input from "Pinecone Vector Store - Tickets" node.  
   - Set wait duration: 15 seconds.

7. **Create "OpenAI Chat Model1" node**  
   - Type: OpenAI Chat Model  
   - Model: gpt-4.1-mini or equivalent GPT-4 variant  
   - Use OpenAI API credentials.

8. **Create "Embeddings OpenAI2" node**  
   - No input connection; used by Pinecone vector store for AI agent tool.  
   - Use OpenAI API credentials.

9. **Create "Pinecone Vector Store2" node**  
   - Mode: Retrieve-as-tool  
   - topK: 200  
   - Pinecone index and namespace must match "Pinecone Vector Store - Tickets" node.  
   - Connect ai_embedding input from "Embeddings OpenAI2".  
   - Use Pinecone API credentials.

10. **Create "AI Agent - Summary generator" node**  
    - Type: Langchain Agent  
    - Connect input from "Wait1" node.  
    - Connect ai_languageModel input from "OpenAI Chat Model1".  
    - Connect ai_tool input from "Pinecone Vector Store2".  
    - Prompt:  
      ```
      Create a summary of tickets available in the vector store. Highlight main complaints that are mentioned in multiple tickets. For each complaint, mention the number of different tickets that the complaint was raised in.
      ```  
    - System message:  
      ```
      Your job is to respond to prompts using the information you gather from the customer support tickets. Your main role is to help your users to gain insights from tickets without having to read tickets one by one.
      ```  
    - On error: Continue with error output.

11. **Create "Send to Slack channel" node**  
    - Connect input from "AI Agent - Summary generator" node.  
    - Text: Use expression to get summary output:  
      `={{ $('AI Agent - Summary generator').item.json.output }}`  
    - Select channel by ID; replace `"C1111111"` with your Slack channel ID.  
    - Configure Slack OAuth2 credentials.

12. **Validate all credentials (Zendesk, Pinecone, OpenAI, Slack).**

13. **Finalize connections:**  
    - Daily trigger → Get tickets → Pinecone Vector Store - Tickets → Wait1 → AI Agent - Summary generator → Send to Slack channel  
    - Default Data Loader1 and Embeddings OpenAI feed into Pinecone Vector Store - Tickets  
    - Embeddings OpenAI2 feeds Pinecone Vector Store2 → AI Agent  
    - OpenAI Chat Model1 feeds AI Agent

14. **Test workflow manually, monitor logs for errors, and adjust parameters as needed.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| This workflow automates daily summarization of Zendesk tickets with Pinecone vector search and OpenAI GPT-4 AI, posting summaries to Slack for support team insights.                                                                                                                                                                                                                                                                                                      | Sticky Note6 content in the workflow                                                                               |
| Setup requires configuring Zendesk, Pinecone, OpenAI, and Slack credentials properly. Replace placeholder values for Pinecone index/namespace, Zendesk brand ID, and Slack channel ID.                                                                                                                                                                                                                                                                                       | Sticky Note6 content in the workflow                                                                               |
| Pinecone namespaces get created automatically; clearing the namespace before inserting new tickets ensures fresh data.                                                                                                                                                                                                                                                                                                                                                   | Sticky Note1                                                                                                      |
| Wait node ensures Pinecone index updates before AI summarization; insufficient wait time may cause incomplete summaries.                                                                                                                                                                                                                                                                                                                                                   | Sticky Note3                                                                                                      |
| For best results, ensure ticket fields used in data loader exist and are correctly indexed; missing fields may lead to incomplete or erroneous embeddings.                                                                                                                                                                                                                                                                                                                  | Sticky Note2                                                                                                      |
| Slack node uses OAuth2 credentials; ensure token has permissions to post in the target channel.                                                                                                                                                                                                                                                                                                                                                                             | Slack API documentation                                                                                           |
| For detailed instructions and community support, visit n8n forums and official documentation on integrating Langchain, Pinecone, and OpenAI.                                                                                                                                                                                                                                                                                                                               | https://docs.n8n.io/, https://www.pinecone.io/, https://openai.com/docs/                                         |

---

**Disclaimer:** The text provided is generated exclusively from an automated n8n workflow. It complies fully with content policies and does not contain illegal, offensive, or protected material. All processed data is legal and public.