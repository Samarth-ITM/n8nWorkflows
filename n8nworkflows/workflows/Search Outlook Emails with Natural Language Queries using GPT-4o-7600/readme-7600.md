Search Outlook Emails with Natural Language Queries using GPT-4o

https://n8nworkflows.xyz/workflows/search-outlook-emails-with-natural-language-queries-using-gpt-4o-7600


# Search Outlook Emails with Natural Language Queries using GPT-4o

### 1. Workflow Overview

This workflow enables users to search their Microsoft Outlook mailbox using natural language queries, powered by GPT-4o. It interprets user input as a search query, fetches relevant emails from Outlook, ranks them by relevance to the query, and outputs a ranked table of results with scores and email links.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Receives the natural language query from a chat interface.
- **1.2 Search Term Generation**: Uses GPT-4o to convert the natural language input into a structured Outlook search term.
- **1.3 Outlook Email Retrieval**: Executes the search query on Outlook to retrieve matching emails.
- **1.4 Email Relevance Scoring**: Uses GPT-4o to score each retrieved email on its relevance to the original user query.
- **1.5 Results Aggregation and Formatting**: Aggregates the scored emails and formats the output into a user-friendly table.

Supporting these are memory and parsing nodes to maintain context and handle structured data, plus a comprehensive credential setup for OpenAI and Outlook integrations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for incoming chat messages containing natural language search requests.

- **Nodes Involved:**  
  - `When chat message received`

- **Node Details:**  
  - **Type:** Chat Trigger (Langchain)  
  - **Role:** Entry point receiving user queries via a chat interface webhook.  
  - **Configuration:** Default options; triggered automatically on incoming chat messages.  
  - **Expressions/Variables:** Captures chat input as `chatInput` JSON property.  
  - **Input Connections:** None (start node).  
  - **Output Connections:** Sends data to `Generate Search Term` and `Merge` nodes.  
  - **Edge Cases:** Webhook connectivity issues, malformed input, missing chat message.  
  - **Version:** 1.3

---

#### 1.2 Search Term Generation

- **Overview:**  
  Converts the user's natural language query into a structured Outlook search term using GPT-4o.

- **Nodes Involved:**  
  - `Generate Search Term`  
  - `Structured Output Parser3`  
  - `Simple Memory1`  
  - `OpenAI Chat Model6`

- **Node Details:**  
  - **Generate Search Term**  
    - Type: Langchain Agent  
    - Role: Transforms user chat input into a JSON with a `"search"` key containing the Outlook search query string.  
    - Configuration: Uses system message prompt explaining the expected output format (`{"search": "search term"}`).  
    - Expressions: Input derived from chat message JSON property.  
    - Input: From `When chat message received` (main output).  
    - Output: Produces parsed JSON with the search term.  
    - Edge Cases: Parsing errors if GPT output deviates from schema, API timeouts, incomplete prompts.  
    - Version: 2.2  

  - **Structured Output Parser3**  
    - Type: Output Parser (Langchain)  
    - Role: Parses GPT response into structured JSON (expects `{"search": "string"}`).  
    - Input: From `OpenAI Chat Model6`.  
    - Output: To `Generate Search Term`.  
    - Edge Cases: Parsing failure if output malformed.  
    - Version: 1.3  

  - **Simple Memory1**  
    - Type: Langchain Memory Buffer (windowed)  
    - Role: Provides short-term memory context to the agent.  
    - Input: Feeds memory to `Generate Search Term`.  
    - Edge Cases: Memory overflow or context loss if conversation too long.  
    - Version: 1.3  

  - **OpenAI Chat Model6**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Executes GPT-4o-mini model for generating search term.  
    - Configuration: Uses GPT-4o-mini model with OpenAI credentials configured.  
    - Input: Connected from `Generate Search Term` AI language model slot.  
    - Output: To `Structured Output Parser3`.  
    - Edge Cases: API limits, authentication errors, latency.  
    - Version: 1.2  

---

#### 1.3 Outlook Email Retrieval

- **Overview:**  
  Performs the search query against the Outlook mailbox and collects emails matching the generated search term.

- **Nodes Involved:**  
  - `Search Outlook`  
  - `Sticky Note24` (instructional only)  
  - `OpenAI Chat Model5` (connected downstream for scoring)  
  - `Merge`

- **Node Details:**  
  - **Search Outlook**  
    - Type: Microsoft Outlook node  
    - Role: Fetches emails using Microsoft Outlook OAuth2 API with a limit of 5 results by default.  
    - Configuration: Performs `getAll` operation filtered by search term from `Generate Search Term` output, excludes emails from `rbreen@ynteractive.com` to avoid self-results.  
    - Input: From `Generate Search Term` main output.  
    - Output: Raw email data to `Merge`.  
    - Credentials: Requires OAuth2 credentials for Outlook account.  
    - Edge Cases: OAuth token expiration, API throttling, search syntax errors, empty result sets.  
    - Version: 2  

  - **Merge**  
    - Type: Merge node  
    - Role: Combines two inputs: the original chat message and the Outlook search results for further processing.  
    - Configuration: Combine mode “combineAll”.  
    - Input: From `When chat message received` and `Search Outlook`.  
    - Output: To `Score if email is relevent to search`.  
    - Edge Cases: Input mismatch or missing data.  
    - Version: 3.2  

  - **Sticky Note24**  
    - Content with setup instructions for Outlook OAuth2 API credentials and usage tips.  
    - No functional role.  

---

#### 1.4 Email Relevance Scoring

- **Overview:**  
  For each retrieved email, calculates a relevance score (1-100) indicating how likely it matches the user's original query, using GPT-4o.

- **Nodes Involved:**  
  - `Score if email is relevent to search`  
  - `Structured Output Parser5`  
  - `OpenAI Chat Model5`  
  - `Convert to one Output`

- **Node Details:**  
  - **Score if email is relevent to search**  
    - Type: Langchain Agent  
    - Role: Scores each email's relevance based on the email content, sender, URL, and user query.  
    - Configuration: System message requests a numeric score output, structured JSON expected with `"score"` and `"url"`.  
    - Expressions: Constructs a prompt combining user question and email fields (`body.content`, `sender.emailAddress.address`, `webLink`).  
    - Input: Combined data from `Merge`.  
    - Output: Parsed JSON scores.  
    - Edge Cases: Parsing failures, inconsistent scoring format, API failures.  
    - Version: 2.2  

  - **Structured Output Parser5**  
    - Type: Output Parser (Langchain)  
    - Role: Parses the scoring agent's output into JSON with `score` and `url`.  
    - Input: From `OpenAI Chat Model5`.  
    - Output: To `Score if email is relevent to search`.  
    - Edge Cases: Parsing errors if GPT output malformed.  
    - Version: 1.3  

  - **OpenAI Chat Model5**  
    - Type: Langchain OpenAI Chat Model  
    - Role: GPT-4o-mini for scoring emails.  
    - Credentials: Uses configured OpenAI API credentials.  
    - Input: Connected to `Score if email is relevent to search`.  
    - Output: To `Structured Output Parser5`.  
    - Edge Cases: API rate limits, authentication errors.  
    - Version: 1.2  

  - **Convert to one Output**  
    - Type: Aggregate node  
    - Role: Aggregates all individual email scores into a single output data set.  
    - Input: From `Score if email is relevent to search`.  
    - Output: To `Output as table for user`.  
    - Edge Cases: Empty inputs, aggregation failures.  
    - Version: 1  

---

#### 1.5 Results Aggregation and Formatting

- **Overview:**  
  Formats the aggregated email scores and URLs into a clear, tabular output for the user.

- **Nodes Involved:**  
  - `Output as table for user`  
  - `OpenAI Chat Model7`

- **Node Details:**  
  - **Output as table for user**  
    - Type: Langchain Agent  
    - Role: Takes aggregated data and produces a formatted table with columns “score” and “url”.  
    - Configuration: System prompt instructs to write data into a two-column table output.  
    - Input: From `Convert to one Output`.  
    - Output: Final output to chat interface or downstream consumption.  
    - Edge Cases: Formatting errors, large data sets that break formatting.  
    - Version: 2.2  

  - **OpenAI Chat Model7**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Uses GPT-4o-mini to generate the final table format.  
    - Credentials: Uses OpenAI API credentials.  
    - Input: From `Output as table for user`.  
    - Output: To chat interface or workflow endpoint.  
    - Edge Cases: API errors, formatting issues.  
    - Version: 1.2  

---

### 3. Summary Table

| Node Name                    | Node Type                             | Functional Role                          | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                                                  |
|------------------------------|-------------------------------------|----------------------------------------|-------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When chat message received    | Chat Trigger (Langchain)             | Receive natural language search query  | None                          | Generate Search Term, Merge           |                                                                                                                              |
| Sticky Note21                | Sticky Note                         | Setup instructions (OpenAI & Outlook) | None                          | None                                 | ⚙️ Setup Instructions: OpenAI and Outlook credential setup details + contact info                                              |
| Sticky Note22                | Sticky Note                         | Workflow description                   | None                          | None                                 | 📧 Outlook Email Search Assistant overview                                                                                   |
| OpenAI Chat Model5            | Langchain OpenAI Chat Model         | Email relevance scoring (GPT-4o-mini) | Score if email is relevent to search | Structured Output Parser5             |                                                                                                                              |
| Sticky Note23                | Sticky Note                         | OpenAI connection setup instructions   | None                          | None                                 | 1️⃣ Set Up OpenAI Connection instructions                                                                                    |
| OpenAI Chat Model6            | Langchain OpenAI Chat Model         | Search term generation (GPT-4o-mini)   | Generate Search Term           | Structured Output Parser3             |                                                                                                                              |
| Simple Memory1               | Langchain Memory Buffer             | Provide memory context                  | None                          | Generate Search Term                  |                                                                                                                              |
| Structured Output Parser3     | Langchain Output Parser             | Parse GPT output for search term        | OpenAI Chat Model6             | Generate Search Term                  |                                                                                                                              |
| Merge                        | Merge Node                         | Combine user input with search results | When chat message received, Search Outlook | Score if email is relevent to search |                                                                                                                              |
| Structured Output Parser5     | Langchain Output Parser             | Parse GPT output for email scoring      | OpenAI Chat Model5             | Score if email is relevent to search |                                                                                                                              |
| Search Outlook               | Microsoft Outlook Node              | Fetch emails matching search term       | Generate Search Term           | Merge                               | 2️⃣ Set Up Outlook Connection instructions                                                                                   |
| Score if email is relevent to search | Langchain Agent                    | Score emails by relevance                | Merge                         | Convert to one Output                 |                                                                                                                              |
| Output as table for user      | Langchain Agent                    | Format final results into table          | Convert to one Output          | None (final output)                   |                                                                                                                              |
| Convert to one Output         | Aggregate Node                    | Aggregate scored emails                   | Score if email is relevent to search | Output as table for user              |                                                                                                                              |
| OpenAI Chat Model7            | Langchain OpenAI Chat Model         | Generate table output                      | Output as table for user       | None (final output)                   |                                                                                                                              |
| Sticky Note24                | Sticky Note                         | Outlook OAuth2 API credential instructions | None                          | None                                 | 2️⃣ Set Up Outlook Connection details                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Purpose: Receive user natural language queries.  
   - Configure webhook to receive chat messages.  
   - No special parameters needed.  

2. **Create a Langchain Agent Node "Generate Search Term"**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Purpose: Convert user input into a structured Outlook search string.  
   - Configure `systemMessage` with instructions:  
     ```
     Take in a search that someone wants to search their outlook for a specific mail. Once they tell you what they want, output a search term.  
     Output like this:  
     {  
       "search": "search term"  
     }
     ```  
   - Enable `hasOutputParser`.  
   - Connect the input from the Chat Trigger node main output.  

3. **Create a Structured Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Purpose: Parse GPT output into JSON with a `"search"` field.  
   - Configure example JSON schema:  
     ```json
     {
       "search": "search term"
     }
     ```  
   - Connect input from `Generate Search Term` agent's AI language model output.  
   - Connect output to the `Generate Search Term` agent input for structured output.  

4. **Create a Simple Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Purpose: Provide conversational memory context to the agent.  
   - Connect output to the `Generate Search Term` agent memory input.  

5. **Create an OpenAI Chat Model Node for Search Term Generation**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `gpt-4o-mini`  
   - Connect to `Generate Search Term` agent AI language model input.  
   - Attach OpenAI API credentials (configured in n8n).  

6. **Create Microsoft Outlook Node "Search Outlook"**  
   - Type: `n8n-nodes-base.microsoftOutlook`  
   - Operation: `getAll`  
   - Filter: Use the expression `={{ $json.output.search }} -from:rbreen@ynteractive.com` to exclude specific sender.  
   - Limit: Default to 5 emails (modifiable).  
   - Attach Microsoft Outlook OAuth2 credentials.  
   - Connect input from `Generate Search Term` main output.  

7. **Create a Merge Node**  
   - Type: `n8n-nodes-base.merge`  
   - Mode: `Combine` → `combineAll`  
   - Connect inputs:  
     - From `When chat message received` (main output).  
     - From `Search Outlook` (main output).  

8. **Create Langchain Agent "Score if email is relevent to search"**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - System Message:  
     ```
     take in the user question and give a number 1-100 on how likely this is the email they are looking for. output like this:
     ```  
   - Text Parameter: Use expression:  
     ```
     =User question: {{ $json.chatInput }} Email content: {{ $json.body.content }}  sender: {{ $json.sender.emailAddress.address }}  url: {{ $json.webLink }}
     ```  
   - Enable output parser.  
   - Connect input from `Merge` node.  

9. **Create Structured Output Parser for Scoring**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - JSON schema example:  
     ```json
     {
       "score": "score",
       "url": "url"
     }
     ```  
   - Connect input from `OpenAI Chat Model5`.  
   - Connect output to the scoring agent.  

10. **Create OpenAI Chat Model Node for Scoring**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Model: `gpt-4o-mini`  
    - Connect to `Score if email is relevent to search` agent AI language model input.  
    - Attach OpenAI API credentials.  

11. **Create Aggregate Node "Convert to one Output"**  
    - Type: `n8n-nodes-base.aggregate`  
    - Aggregate: `aggregateAllItemData`  
    - Connect input from `Score if email is relevent to search`.  

12. **Create Langchain Agent "Output as table for user"**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - System Message:  
      ```
      write this data into a table output with two columns. score, then url.
      ```  
    - Text Parameter: `={{ $json.data }}` (aggregated scored data)  
    - Connect input from `Convert to one Output`.  

13. **Create OpenAI Chat Model Node for Final Output**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Model: `gpt-4o-mini`  
    - Connect to `Output as table for user` agent AI language model input.  
    - Attach OpenAI API credentials.  

14. **Connect the workflow outputs accordingly**  
    - Final output arises from the last Langchain Agent node (`Output as table for user`).  

15. **Credential Setup**  
    - **OpenAI:** Obtain API key from [OpenAI Platform](https://platform.openai.com/api-keys), add billing funds, create credentials in n8n under OpenAI API.  
    - **Microsoft Outlook:** Create OAuth2 credential in n8n for Microsoft Outlook, authorize with user account, attach credentials to `Search Outlook` node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow enables natural language searching of Outlook emails using GPT-4o to generate queries and score relevance.                                                    | Workflow Description                                                                               |
| Setup instructions for OpenAI API key and billing, plus Outlook OAuth2 credential creation, are embedded as sticky notes in the workflow.                              | Sticky Note21, Sticky Note23, Sticky Note24                                                      |
| For help customizing or building similar workflows, contact Robert Breen: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/, website: https://ynteractive.com | Contact info in Sticky Note21                                                                     |
| Excludes emails from "rbreen@ynteractive.com" in search results by default to avoid self-references; this can be modified in the `Search Outlook` node filter expression. | Search Outlook node filter configuration                                                          |
| Recommended to adjust the `limit` parameter of the `Search Outlook` node if more or fewer search results are desired.                                                  | Sticky Note24                                                                                    |

---

*Disclaimer: The provided text exclusively originates from an automated n8n workflow. All data processed complies with current content policies and contains no illegal or offensive elements. All data handled is legal and public.*