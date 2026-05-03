Build an IT Support Assistant Chatbot Leveraging Existing Support Portal

https://n8nworkflows.xyz/workflows/build-an-it-support-assistant-chatbot-leveraging-existing-support-portal-3498


# Build an IT Support Assistant Chatbot Leveraging Existing Support Portal

### 1. Workflow Overview

This workflow implements an IT Support Assistant Chatbot that leverages an existing support portal’s search API to provide accurate, up-to-date answers to user queries without duplicating or indexing the entire knowledge base. It exemplifies a Retrieval-Augmented Generation (RAG) approach by integrating live search results from AcuityScheduling’s help center into an AI-powered chatbot.

The workflow is logically divided into the following blocks:

- **1.1 Chat Input Reception:** Captures user chat messages to trigger the support assistant.
- **1.2 AI Agent Processing:** Uses an AI agent with OpenAI’s GPT-4o-mini model and simple memory to interpret queries and orchestrate tool usage.
- **1.3 Knowledgebase Search Tool:** Invokes a custom subworkflow tool that queries the AcuityScheduling support portal’s search API.
- **1.4 Search Results Processing:** Filters, extracts, and formats relevant fields from the search results.
- **1.5 Response Aggregation:** Aggregates the cleaned search results into a single response to send back to the AI agent.
- **1.6 Conditional Handling:** Checks if search results exist and handles empty responses gracefully.

---

### 2. Block-by-Block Analysis

#### 1.1 Chat Input Reception

- **Overview:**  
  Listens for incoming chat messages from users to initiate the support chatbot workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* LangChain Chat Trigger (root node)  
    - *Role:* Entry point for user chat messages, triggering the workflow on each new message.  
    - *Configuration:* Default options, webhook ID assigned for external chat integration.  
    - *Inputs:* External chat message webhook.  
    - *Outputs:* Passes chat message data downstream to the AI agent node.  
    - *Edge Cases:* Webhook connectivity issues, malformed chat messages, or unsupported message formats.

---

#### 1.2 AI Agent Processing

- **Overview:**  
  Processes user queries using an AI agent powered by OpenAI’s GPT-4o-mini model, maintaining conversational context via simple memory and orchestrating calls to the knowledgebase tool.

- **Nodes Involved:**  
  - AcuityScheduling Support Chatbot (LangChain Agent)  
  - OpenAI Chat Model  
  - Simple Memory  
  - Knowledgebase Tool

- **Node Details:**  
  - **AcuityScheduling Support Chatbot**  
    - *Type:* LangChain Agent  
    - *Role:* Core AI agent that interprets user queries, decides when to call tools, and formulates responses.  
    - *Configuration:*  
      - System message instructs the agent to act as a support assistant strictly for AcuityScheduling.com.  
      - Encourages factual answers referencing knowledgebase URLs.  
      - Restricts scope to AcuityScheduling service only.  
    - *Inputs:* Chat messages from "When chat message received" node.  
    - *Outputs:* Sends queries to OpenAI model, memory, and knowledgebase tool; outputs final response to chat.  
    - *Edge Cases:* Model API errors, rate limits, or unexpected user queries outside scope.  
    - *Version:* 1.8

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model  
    - *Role:* Provides language model completions using GPT-4o-mini.  
    - *Configuration:*  
      - Model set to "gpt-4o-mini" for cost-effective yet capable responses.  
      - Uses OpenAI API credentials.  
    - *Inputs:* Prompts from AI agent.  
    - *Outputs:* Model-generated text responses.  
    - *Edge Cases:* API authentication failures, network timeouts, or quota exceeded errors.  
    - *Version:* 1.2

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Maintains conversational context to enable coherent multi-turn dialogue.  
    - *Configuration:* Default buffer window, no custom parameters.  
    - *Inputs:* Conversation history from AI agent.  
    - *Outputs:* Provides memory context back to AI agent.  
    - *Edge Cases:* Memory overflow or loss of context if conversation is too long.  
    - *Version:* 1.3

  - **Knowledgebase Tool**  
    - *Type:* LangChain Tool Workflow  
    - *Role:* Custom tool that calls a subworkflow to query the AcuityScheduling support portal search API.  
    - *Configuration:*  
      - Tool named "acuity_support_search".  
      - Description clarifies its purpose.  
      - Passes user query as input parameter to the subworkflow.  
      - Uses the current workflow’s ID to invoke the subworkflow.  
    - *Inputs:* Query string from AI agent.  
    - *Outputs:* Search results passed back to AI agent.  
    - *Edge Cases:* Subworkflow invocation failures, input mapping errors.  
    - *Version:* 2.1

---

#### 1.3 Knowledgebase Search Tool (Subworkflow)

- **Overview:**  
  Executes the HTTP request to AcuityScheduling’s support portal search API with the user query and returns raw search results.

- **Nodes Involved:**  
  - KnowledgeBase Tool Subworkflow (Execute Workflow Trigger)  
  - Acuity Support Search API (HTTP Request)  
  - Has Results? (If node)  
  - Results to Items (Split Out)  
  - Extract Relevant Fields (Set)  
  - Aggregate Response (Aggregate)  
  - Empty Response (Set)

- **Node Details:**  
  - **KnowledgeBase Tool Subworkflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point for the subworkflow called by the Knowledgebase Tool.  
    - *Configuration:* Accepts "query" as input parameter.  
    - *Inputs:* Query string from main workflow.  
    - *Outputs:* Passes query to HTTP Request node.  
    - *Edge Cases:* Input parameter missing or malformed.  
    - *Version:* 1.1

  - **Acuity Support Search API**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to AcuityScheduling’s Algolia-powered search API.  
    - *Configuration:*  
      - URL includes Algolia application ID and API key in headers.  
      - POST body contains search query with parameters: hits per page = 5, page = 0, facets for locale, labels, and categories.  
      - Headers mimic browser request to avoid blocking.  
      - JSON body dynamically constructed using the input query.  
    - *Inputs:* Query string from subworkflow trigger.  
    - *Outputs:* Raw JSON search results.  
    - *Edge Cases:* API key expiration, rate limiting, network errors, malformed JSON responses.  
    - *Version:* 4.2

  - **Has Results?**  
    - *Type:* If Node  
    - *Role:* Checks if the search API returned any hits.  
    - *Configuration:*  
      - Condition: length of `results[0].hits` array > 0.  
    - *Inputs:* Search API response.  
    - *Outputs:* Routes to either processing nodes or empty response node.  
    - *Edge Cases:* Unexpected API response structure or empty results.  
    - *Version:* 2.2

  - **Results to Items**  
    - *Type:* Split Out  
    - *Role:* Splits the array of hits into individual items for processing.  
    - *Configuration:* Field to split out: `results[0].hits`.  
    - *Inputs:* Search API response (filtered by If node).  
    - *Outputs:* Individual hit items.  
    - *Edge Cases:* Empty array input or unexpected data format.  
    - *Version:* 1

  - **Extract Relevant Fields**  
    - *Type:* Set  
    - *Role:* Extracts and renames key fields from each hit to optimize token usage and clarity.  
    - *Configuration:*  
      - Extracts `title` from hit’s `title` field.  
      - Extracts `body` from hit’s `body_safe` field.  
      - Constructs `url` by concatenating base help center URL with article ID.  
    - *Inputs:* Individual hit items.  
    - *Outputs:* Cleaned and simplified search result items.  
    - *Edge Cases:* Missing fields in hits, malformed URLs.  
    - *Version:* 3.4

  - **Aggregate Response**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all cleaned items into a single array under the field `response`.  
    - *Configuration:* Aggregate all item data into one field.  
    - *Inputs:* Cleaned search result items.  
    - *Outputs:* Aggregated response array.  
    - *Edge Cases:* Large number of items causing payload size issues.  
    - *Version:* 1

  - **Empty Response**  
    - *Type:* Set  
    - *Role:* Provides an empty array response when no search results are found.  
    - *Configuration:* Sets `response` field to empty array `[]`.  
    - *Inputs:* If node output when no results.  
    - *Outputs:* Empty response to main workflow.  
    - *Edge Cases:* Ensures AI agent receives valid empty response instead of error.  
    - *Version:* 3.4

---

#### 1.4 Search Results Processing & Response Aggregation

- **Overview:**  
  Processes the raw search results to extract relevant information, optimize token usage, and aggregate them into a single response for the AI agent.

- **Nodes Involved:**  
  - Results to Items  
  - Extract Relevant Fields  
  - Aggregate Response  
  - Empty Response  
  - Has Results?

- **Node Details:**  
  - Covered in the subworkflow analysis above, these nodes collectively clean, filter, and prepare the search results for consumption by the AI agent.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                                  | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                                                            |
|-------------------------------|----------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received     | LangChain Chat Trigger            | Entry point for user chat messages               | —                           | AcuityScheduling Support Chatbot |                                                                                                                                                        |
| AcuityScheduling Support Chatbot | LangChain Agent                  | AI agent processing user queries                  | When chat message received, OpenAI Chat Model, Simple Memory, Knowledgebase Tool | —                           | ## 1. Simple Chatbot with Knowledgebase Tool [Learn more about AI agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| OpenAI Chat Model              | LangChain OpenAI Chat Model       | Provides GPT-4o-mini completions                  | AcuityScheduling Support Chatbot | AcuityScheduling Support Chatbot |                                                                                                                                                        |
| Simple Memory                 | LangChain Memory Buffer Window    | Maintains conversational context                  | AcuityScheduling Support Chatbot | AcuityScheduling Support Chatbot |                                                                                                                                                        |
| Knowledgebase Tool             | LangChain Tool Workflow           | Calls subworkflow to query support portal        | AcuityScheduling Support Chatbot | AcuityScheduling Support Chatbot | ## 2. Use your Existing Help Portal Search [Read more about the HTTP request tool](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| KnowledgeBase Tool Subworkflow | Execute Workflow Trigger          | Subworkflow entry point for knowledgebase tool   | Knowledgebase Tool          | Acuity Support Search API    |                                                                                                                                                        |
| Acuity Support Search API      | HTTP Request                     | Queries AcuityScheduling support portal search   | KnowledgeBase Tool Subworkflow | Has Results?                 |                                                                                                                                                        |
| Has Results?                  | If Node                         | Checks if search results exist                     | Acuity Support Search API   | Results to Items, Empty Response |                                                                                                                                                        |
| Results to Items              | Split Out                       | Splits search hits into individual items          | Has Results?                | Extract Relevant Fields      |                                                                                                                                                        |
| Extract Relevant Fields       | Set                             | Extracts and formats key fields from hits         | Results to Items            | Aggregate Response           | ## 3. Clean up the Results to Optimise Tokens [Read more about the aggregate node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.aggregate) |
| Aggregate Response            | Aggregate                       | Aggregates cleaned items into single response     | Extract Relevant Fields     | KnowledgeBase Tool Subworkflow (return) |                                                                                                                                                        |
| Empty Response               | Set                             | Returns empty array if no search results          | Has Results?                | KnowledgeBase Tool Subworkflow (return) |                                                                                                                                                        |
| Sticky Note                  | Sticky Note                     | Documentation and guidance                         | —                           | —                           | ## 1. Simple Chatbot with Knowledgebase Tool [Learn more about AI agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Sticky Note1                 | Sticky Note                     | Documentation on leveraging existing help portal  | —                           | —                           | ## 2. Use your Existing Help Portal Search [Read more about the HTTP request tool](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| Sticky Note2                 | Sticky Note                     | Documentation on cleaning results to optimize tokens | —                           | —                           | ## 3. Clean up the Results to Optimise Tokens [Read more about the aggregate node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.aggregate) |
| Sticky Note3                 | Sticky Note                     | General overview, usage instructions, and help links | —                           | —                           | See detailed description in Section 1. Workflow Overview. Includes links to Discord and community forum.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a **LangChain Chat Trigger** node named "When chat message received".  
   - Configure webhook settings as needed to receive chat messages from your chat platform.

2. **Add the AI Agent Node**  
   - Add a **LangChain Agent** node named "AcuityScheduling Support Chatbot".  
   - Set the system message to:  
     ```
     You are a support assistant for the SaaS company, AcuityScheduling.com. Your task is to openly help the user with any questions regarding the AcuityScheduling service however, you are restricted to only this service. If the user asks questions unrelated to AcuityScheduling, you may ask them for clarification, explain you are not able to help them out of scope or redirect them to support@acuityScheduling.com. Be factual in your answer, tap into the resources or tools available and do not rely on your training data (which might be out-of-date). When returning a response to the user, you are encouraged to share the URL of the knowledgebase page where the user can explore the documentation for themselves.
     ```  
   - Connect the output of "When chat message received" to this node’s main input.

3. **Add the OpenAI Chat Model Node**  
   - Add a **LangChain OpenAI Chat Model** node named "OpenAI Chat Model".  
   - Select the model "gpt-4o-mini".  
   - Configure OpenAI API credentials.  
   - Connect this node to the AI agent node’s language model input.

4. **Add the Simple Memory Node**  
   - Add a **LangChain Memory Buffer Window** node named "Simple Memory".  
   - Use default settings.  
   - Connect this node to the AI agent node’s memory input.

5. **Create the Knowledgebase Tool Node**  
   - Add a **LangChain Tool Workflow** node named "Knowledgebase Tool".  
   - Name the tool "acuity_support_search".  
   - Set the description: "Call this tool to query AcuityScheduling's Support Center Search API."  
   - Configure it to call a subworkflow (to be created next) with input parameter "query" (string).  
   - Connect this node to the AI agent node’s tool input.

6. **Create the KnowledgeBase Tool Subworkflow**  
   - Create a new workflow to serve as the subworkflow.  
   - Add an **Execute Workflow Trigger** node named "KnowledgeBase Tool Subworkflow".  
   - Configure it to accept an input parameter "query" (string).

7. **Add HTTP Request Node in Subworkflow**  
   - Add an **HTTP Request** node named "Acuity Support Search API".  
   - Set method to POST.  
   - URL:  
     ```
     https://2al21hjwoz-dsn.algolia.net/1/indexes/*/queries?x-algolia-agent=Algolia%20for%20JavaScript%20(3.35.1)%3B%20Browser%20(lite)%3B%20instantsearch.js%201.12.1%3B%20Zendesk%20Integration%20(2.32.0)%3B%20JS%20Helper%20(2.28.1)&x-algolia-application-id=2AL21HJWOZ&x-algolia-api-key=c3c07dd7fb575008575163085a62b92
     ```  
   - Headers:  
     - Accept-Language: en  
     - Cache-Control: no-cache  
     - Connection: keep-alive  
     - Origin: https://help.acuityscheduling.com  
     - Referer: https://help.acuityscheduling.com/  
     - User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36  
     - Accept: application/json  
   - Body (JSON):  
     ```json
     {
       "requests": [
         {
           "indexName": "Zendesk 4-25",
           "params": "query={{$json.query}}&hitsPerPage=5&page=0&facets=[\"locale.locale\",\"label_names\",\"category.title\"]&tagFilters=&facetFilters=[\"locale.locale:en-us\"]"
         }
       ]
     }
     ```  
   - Connect the subworkflow trigger node to this HTTP Request node.

8. **Add If Node to Check Results**  
   - Add an **If** node named "Has Results?".  
   - Condition: Check if length of `results[0].hits` array > 0.  
   - Connect HTTP Request node output to this node.

9. **Add Split Out Node**  
   - Add a **Split Out** node named "Results to Items".  
   - Field to split out: `results[0].hits`.  
   - Connect "Has Results?" true output to this node.

10. **Add Set Node to Extract Fields**  
    - Add a **Set** node named "Extract Relevant Fields".  
    - Assign fields:  
      - `title` = `{{$json.title}}`  
      - `body` = `{{$json.body_safe}}`  
      - `url` = `https://help.acuityscheduling.com/hc/en-us/articles/{{$json.id}}`  
    - Connect "Results to Items" output to this node.

11. **Add Aggregate Node**  
    - Add an **Aggregate** node named "Aggregate Response".  
    - Aggregate all item data into a single field named `response`.  
    - Connect "Extract Relevant Fields" output to this node.

12. **Add Set Node for Empty Response**  
    - Add a **Set** node named "Empty Response".  
    - Set field `response` to an empty array `[]`.  
    - Connect "Has Results?" false output to this node.

13. **Connect Outputs Back to Subworkflow Trigger**  
    - Connect "Aggregate Response" and "Empty Response" outputs back to the subworkflow trigger node’s output.

14. **Connect Subworkflow to Main Workflow**  
    - In the main workflow, configure the "Knowledgebase Tool" node to call this subworkflow by its workflow ID.  
    - Map the input parameter "query" from the AI agent’s tool input.

15. **Final Connections**  
    - Connect "When chat message received" → "AcuityScheduling Support Chatbot".  
    - Connect "OpenAI Chat Model" → "AcuityScheduling Support Chatbot" (language model input).  
    - Connect "Simple Memory" → "AcuityScheduling Support Chatbot" (memory input).  
    - Connect "Knowledgebase Tool" → "AcuityScheduling Support Chatbot" (tool input).

16. **Credential Setup**  
    - Configure OpenAI API credentials for the "OpenAI Chat Model" node.  
    - No authentication required for the AcuityScheduling search API as it uses a public Algolia API key embedded in the HTTP request headers.  
    - If your own support portal requires authentication, add appropriate credentials and update the HTTP Request node accordingly.

17. **Testing**  
    - Deploy the workflow.  
    - Send test chat messages such as:  
      - "How do I connect my icloud to acuityScheduling?"  
      - "How do I download past invoices for my Acuity account?"  
    - Verify the chatbot returns relevant answers with URLs to the knowledgebase articles.

---

This detailed reference enables both human developers and AI agents to fully understand, reproduce, and customize the IT Support Assistant Chatbot leveraging an existing support portal search API.