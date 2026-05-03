Create Knowledge Base from Jira Tickets with OpenAI Embeddings and Pinecone

https://n8nworkflows.xyz/workflows/create-knowledge-base-from-jira-tickets-with-openai-embeddings-and-pinecone-9620


# Create Knowledge Base from Jira Tickets with OpenAI Embeddings and Pinecone

---

### 1. Workflow Overview

This workflow automates the creation of a knowledge base by extracting information from recently completed Jira tickets, processing the ticket data into vector embeddings using OpenAI, and storing these vectors in a Pinecone vector database. It is designed for teams or organizations that want to maintain an up-to-date, searchable knowledge base of completed Jira issues enriched with ticket details and comments.

The workflow logically divides into three main blocks:

- **1.1 Scheduled Trigger and Jira Ticket Retrieval:** Initiates the workflow on a schedule and fetches completed Jira tickets updated within the last day.
- **1.2 Ticket Data Extraction and Formatting:** For each Jira ticket, retrieves detailed comments and formats the combined summary, description, and comments into a structured data object.
- **1.3 Embedding Generation and Pinecone Registration:** Converts the formatted ticket data into vector embeddings using OpenAI, then inserts these vectors into a Pinecone index for vector similarity search.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Jira Ticket Retrieval

- **Overview:**  
This block triggers the workflow periodically and retrieves a list of Jira issues that are marked as completed (statusCategory "Done") and updated within the last day.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Jira Issue List  
  - Loop Over Items  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `scheduleTrigger`  
    - Role: Initiates the workflow on a recurring schedule (default interval configured).  
    - Configuration: Runs at regular intervals (default unspecified here, typically every day or hour).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Jira Issue List".  
    - Potential Failures: Scheduling misconfiguration, n8n service downtime.  

  - **Jira Issue List**  
    - Type: `jira` (getAll operation)  
    - Role: Retrieves Jira issues with JQL filter "statusCategory IN ("Done") AND updated > -1d" to get tickets completed and updated in the last day.  
    - Configuration: Uses JQL to filter issues. No pagination or batch size explicitly set.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Sends list of issues to "Loop Over Items".  
    - Potential Failures: Jira API authentication errors, network timeouts, malformed JQL query errors.  

  - **Loop Over Items**  
    - Type: `splitInBatches`  
    - Role: Processes each Jira issue individually to handle downstream nodes one at a time, preventing rate limits or memory overload.  
    - Configuration: Default batch size (not explicitly defined, typically 1).  
    - Inputs: Receives all Jira issues from "Jira Issue List".  
    - Outputs: Two outputs; main output to "Code1" after batch processing, empty output to "Jira Issue Detail".  
    - Potential Failures: Batch size misconfiguration, data type errors in batches.

#### 2.2 Ticket Data Extraction and Formatting

- **Overview:**  
For each Jira issue, this block fetches detailed comments, extracts key fields (id, summary, description), and compiles all comments into a markdown-formatted string.

- **Nodes Involved:**  
  - Jira Issue Detail  
  - Code1  
  - Code2  

- **Node Details:**

  - **Jira Issue Detail**  
    - Type: `jira` (getAll operation on issueComment resource)  
    - Role: Retrieves up to 5 comments related to each Jira issue, identified by dynamic issueKey from input JSON `id`.  
    - Configuration: Limits to 5 comments per issue, fetches comment data structure.  
    - Inputs: Receives each issue from "Code1" node output (via "Loop Over Items" second output connected to "Code1").  
    - Outputs: Sends comments data to "Code2".  
    - Potential Failures: Jira API limits, missing issueKey, permission errors, empty comments.  

  - **Code1**  
    - Type: `code` (JavaScript)  
    - Role: Extracts from the first input JSON object the `id`, `fields.summary`, and `fields.description` of the Jira issue, producing a simplified object.  
    - Configuration: Executes once per batch, returns an object with keys `id`, `summary`, and `description`.  
    - Key expressions: Access `$input.first().json` to get the first input item.  
    - Inputs: Receives single Jira issue JSON from second output of "Loop Over Items".  
    - Outputs: Sends cleaned issue metadata to "Jira Issue Detail".  
    - Potential Failures: Missing fields in Jira JSON, undefined properties, code execution errors.

  - **Code2**  
    - Type: `code` (JavaScript)  
    - Role: Aggregates all comments fetched by "Jira Issue Detail" into a markdown string, including comment update date and text content. Returns an object combining the issue id, summary, description, and concatenated comments.  
    - Configuration: Iterates over all comments, concatenating text with markdown formatting (headers with dates).  
    - Key expressions: Uses `$input.all()` to access all comment items, and references "Code1" node for issue info.  
    - Inputs: Receives comments from "Jira Issue Detail".  
    - Outputs: Sends combined issue data to "Pinecone Vector Store".  
    - Potential Failures: Comments content structure variations, missing comment text, code errors.

#### 2.3 Embedding Generation and Pinecone Registration

- **Overview:**  
This block generates OpenAI embeddings from the prepared Jira ticket data and inserts these vectors into the Pinecone vector index. It also includes a default document loader node configured with metadata for Pinecone storage.

- **Nodes Involved:**  
  - Pinecone Vector Store  
  - Embeddings OpenAI  
  - Default Data Loader  

- **Node Details:**

  - **Pinecone Vector Store**  
    - Type: `vectorStorePinecone` (LangChain node)  
    - Role: Inserts vector embeddings and documents into a Pinecone index for vector similarity search and knowledge base construction.  
    - Configuration: Mode set to "insert", uses a Pinecone index ID placeholder `<Pinecone Index>`.  
    - Inputs: Receives input from "Code2" (main), from "Embeddings OpenAI" (embedding), and from "Default Data Loader" (document).  
    - Outputs: Connects back to "Loop Over Items" main output, likely to process next batch.  
    - Potential Failures: Pinecone authentication errors, index misconfiguration, network issues, data format mismatches.  

  - **Embeddings OpenAI**  
    - Type: `embeddingsOpenAi` (LangChain node)  
    - Role: Generates vector embeddings from the ticket data text using OpenAI embedding models.  
    - Configuration: Default options used, likely the OpenAI API key credential required.  
    - Inputs: Receives data from "Code2" or related nodes (connected via ai_embedding input).  
    - Outputs: Sends embeddings to "Pinecone Vector Store".  
    - Potential Failures: OpenAI API quota exceeded, invalid API key, network timeouts, invalid input format.  

  - **Default Data Loader**  
    - Type: `documentDefaultDataLoader` (LangChain node)  
    - Role: Loads document metadata for insertion into Pinecone; metadata includes JiraIssueID and JiraUrl (though JiraUrl value is empty here).  
    - Configuration: Metadata fields set as "JiraIssueID" with dynamic value `{{$json.id}}` and "JiraUrl" (empty).  
    - Inputs: Receives data from "Embeddings OpenAI" or "Code2" (via ai_document input).  
    - Outputs: Sends document metadata to "Pinecone Vector Store".  
    - Potential Failures: Missing metadata values, improper document structure.

---

### 3. Summary Table

| Node Name            | Node Type                | Functional Role                                       | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                         |
|----------------------|--------------------------|-----------------------------------------------------|-------------------------|-------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger      | scheduleTrigger          | Triggers workflow on a regular schedule             | None                    | Jira Issue List         | ## Step1 : Scheduled Trigger The workflow runs at regular intervals according to the schedule set in the Scheduled Trigger node. |
| Jira Issue List       | jira (getAll)            | Retrieves completed Jira tickets updated in last day| Schedule Trigger        | Loop Over Items         |                                                                                                   |
| Loop Over Items       | splitInBatches           | Processes Jira issues one by one                     | Jira Issue List          | Code1, Jira Issue Detail |                                                                                                   |
| Code1                | code                     | Extracts id, summary, and description from issue    | Loop Over Items          | Jira Issue Detail       | ## Step2 : Jira Trigger (Completed Tickets) Retrieves the summary, description, and comments of completed Jira tickets. The target tickets are fetched using the Jira Issue Detail node, and the data is formatted in the Code2 node. |
| Jira Issue Detail     | jira (getAll comments)   | Retrieves comments for each Jira issue               | Code1                    | Code2                   |                                                                                                   |
| Code2                | code                     | Aggregates comments and combines with issue info    | Jira Issue Detail        | Pinecone Vector Store   |                                                                                                   |
| Pinecone Vector Store | vectorStorePinecone      | Inserts vectors and documents into Pinecone index   | Code2, Embeddings OpenAI, Default Data Loader | Loop Over Items         | ## Step3 : Register to Pinecone Converts the retrieved ticket information into vectors and registers them in Pinecone. |
| Embeddings OpenAI     | embeddingsOpenAi         | Generates vector embeddings from ticket data         | Code2                    | Pinecone Vector Store   |                                                                                                   |
| Default Data Loader   | documentDefaultDataLoader| Loads document metadata for Pinecone index           | Embeddings OpenAI        | Pinecone Vector Store   |                                                                                                   |
| Sticky Note          | stickyNote               | Provides descriptive comments for workflow sections | Various                  | N/A                     | Multiple sticky notes with step explanations as noted above                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it**: `Jira Vector - Register Minimal`.

2. **Add a Schedule Trigger node**:  
   - Type: `scheduleTrigger`  
   - Position: top-left area  
   - Configure the interval (e.g., every day or as desired) to trigger the workflow regularly.  

3. **Add a Jira Issue List node**:  
   - Type: `jira` node with operation `getAll`  
   - Connect it from the Schedule Trigger node.  
   - Set the JQL parameter to: `statusCategory IN ("Done") AND updated > -1d` to fetch recently completed tickets.  
   - Use Jira credentials configured for your Jira instance.

4. **Add a Split In Batches node**:  
   - Type: `splitInBatches`  
   - Connect from the Jira Issue List node.  
   - Default batch size (1) to process issues one by one.

5. **Add a Code node named "Code1"**:  
   - Type: `code` (JavaScript)  
   - Connect from the second output of the Split In Batches node (to process the current batch).  
   - Use this JavaScript code to extract the issue id, summary, and description:  
     ```javascript
     return {
       id: $input.first().json.id,
       summary: $input.first().json.fields.summary,
       description: $input.first().json.fields.description,
     }
     ```
   - Set `Execute Once` to true.

6. **Add a Jira Issue Detail node**:  
   - Type: `jira` node with operation `getAll` on resource `issueComment`  
   - Connect from the output of "Code1".  
   - Set `issueKey` to `={{ $json.id }}` dynamically from the output of "Code1".  
   - Limit comments to 5.  
   - Use the same Jira credentials.

7. **Add a Code node named "Code2"**:  
   - Type: `code` (JavaScript)  
   - Connect from Jira Issue Detail node.  
   - Use this code to aggregate comments into markdown and combine with issue info:  
     ```javascript
     let comments = '';
     for (const item of $input.all()) {
       comments += '## ' + item.json.updated + '\n';
       comments += item.json.body.content[0].content[0].text + '\n\n';
     }
     return {
       id: $('Code1').first().json.id,
       summary: $('Code1').first().json.summary,
       description: $('Code1').first().json.description,
       comments: comments,
     };
     ```

8. **Add an Embeddings OpenAI node**:  
   - Type: `embeddingsOpenAi` (LangChain node)  
   - Connect from "Code2" node using the `ai_embedding` input.  
   - Configure with your OpenAI credentials.

9. **Add a Default Data Loader node**:  
   - Type: `documentDefaultDataLoader` (LangChain node)  
   - Connect from "Embeddings OpenAI" node using the `ai_document` input.  
   - Configure metadata fields:  
     - `JiraIssueID` with value: `={{ $json.id }}`  
     - `JiraUrl` (leave empty or set dynamically if URL available).

10. **Add a Pinecone Vector Store node**:  
    - Type: `vectorStorePinecone` (LangChain node)  
    - Connect from "Code2" main output, "Embeddings OpenAI" `ai_embedding` output, and "Default Data Loader" `ai_document` output.  
    - Set mode to "insert".  
    - Specify your Pinecone index ID in the configuration.  
    - Provide Pinecone API credentials.

11. **Connect the Pinecone Vector Store node back to the Split In Batches node main output** to continue processing remaining batches.

12. **Add Sticky Notes for documentation purposes** as follows:  
    - Step 1: Schedule Trigger description.  
    - Step 2: Jira Trigger and data retrieval details.  
    - Step 3: Pinecone registration explanation.

13. **Activate and test the workflow**, ensuring credentials are valid and nodes execute without errors.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow processes only Jira tickets with statusCategory "Done" updated in the last day.        | Important for ensuring only recent completed tickets are indexed.                                     |
| OpenAI embeddings require API key credentials with access to embedding models.                       | See OpenAI documentation for API key setup: https://platform.openai.com/docs/api-reference/authentication |
| Pinecone index must be pre-created and accessible via API key and environment configuration.         | Pinecone documentation: https://docs.pinecone.io/docs/quickstart                                        |
| Comments concatenation assumes a specific Jira comment body structure; adjustments may be required.| Variations in Jira comment JSON may require code changes in "Code2" node.                             |

---

**Disclaimer:**  
The text provided is exclusively generated from an automated n8n workflow. It strictly respects all current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.

---