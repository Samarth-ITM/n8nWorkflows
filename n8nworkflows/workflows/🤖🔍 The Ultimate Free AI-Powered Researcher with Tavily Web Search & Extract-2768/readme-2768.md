🤖🔍 The Ultimate Free AI-Powered Researcher with Tavily Web Search & Extract

https://n8nworkflows.xyz/workflows/-----the-ultimate-free-ai-powered-researcher-with-tavily-web-search---extract-2768


# 🤖🔍 The Ultimate Free AI-Powered Researcher with Tavily Web Search & Extract

### 1. Workflow Overview

This n8n workflow, titled "🤖🔍 The Ultimate Free AI-Powered Researcher with Tavily Web Search & Extract," automates the process of conducting intelligent web research by integrating Tavily's Search and Extract APIs with AI-powered summarization. It is designed for users who want to input a research topic via chat, retrieve relevant web search results, extract detailed content from the top results, and receive a concise AI-generated summary in markdown format.

The workflow is logically divided into the following blocks:

- **1.1 User Input Reception:** Captures the user's research query through a chat interface.
- **1.2 API Key Management:** Sets and manages the Tavily API key for authentication.
- **1.3 Tavily Search Execution:** Performs a web search using Tavily's Search API based on the user's query.
- **1.4 Search Results Filtering:** Filters search results by relevance score to retain only high-quality results.
- **1.5 Content Extraction:** Extracts detailed content from the top filtered search result using Tavily's Extract API.
- **1.6 AI Summarization:** Uses OpenAI's language model to summarize the extracted web content into markdown format.
- **1.7 Documentation and Notes:** Provides embedded documentation and usage notes via sticky notes for user guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 User Input Reception

- **Overview:**  
  This block captures the user's research topic input through a chat trigger node, enabling interactive query submission.

- **Nodes Involved:**  
  - Provide search topic via Chat window

- **Node Details:**  
  - **Provide search topic via Chat window**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point for user input, listens for chat messages to start the workflow.  
    - Configuration: Default chat trigger with webhook ID set for external access.  
    - Inputs: None (trigger node)  
    - Outputs: Passes chat input as `chatInput` JSON property.  
    - Edge Cases: Webhook connectivity issues, malformed chat input, or empty queries.  
    - Sub-workflow: None

#### 1.2 API Key Management

- **Overview:**  
  This block sets the Tavily API key as a workflow variable for use in subsequent API calls.

- **Nodes Involved:**  
  - Tavily API Key

- **Node Details:**  
  - **Tavily API Key**  
    - Type: Set Node  
    - Role: Stores the Tavily API key string for authentication in API requests.  
    - Configuration: Assigns a string value `tvly-YOUR_API_KEY` to the variable `api_key`.  
    - Inputs: Receives data from chat trigger node.  
    - Outputs: Passes `api_key` along with incoming data.  
    - Edge Cases: Missing or invalid API key will cause authentication failures in API calls.  
    - Sub-workflow: None

#### 1.3 Tavily Search Execution

- **Overview:**  
  Executes a web search query against Tavily's Search API using the user's input and the stored API key.

- **Nodes Involved:**  
  - Tavily Search Topic

- **Node Details:**  
  - **Tavily Search Topic**  
    - Type: HTTP Request  
    - Role: Sends POST request to Tavily Search endpoint with query parameters.  
    - Configuration:  
      - URL: `https://api.tavily.com/search`  
      - Method: POST  
      - Body: JSON including `api_key`, `query` (from chat input), `search_depth` set to "basic", `include_images` true, `include_answer` false, `max_results` 5, and empty domain filters.  
    - Expressions: Uses `{{ $json.api_key }}` for API key and `{{ $('Provide search topic via Chat window').item.json.chatInput }}` for query.  
    - Inputs: Receives API key and chat input.  
    - Outputs: Returns search results JSON array under `results`.  
    - Edge Cases: API rate limits, invalid API key, network timeouts, empty or irrelevant search results.  
    - Sub-workflow: None

#### 1.4 Search Results Filtering

- **Overview:**  
  Filters the search results to retain only those with a relevance score above 0.80 (80%).

- **Nodes Involved:**  
  - Filter > 90%

- **Node Details:**  
  - **Filter > 90%**  
    - Type: Set Node  
    - Role: Filters the `results` array to include only items with `score > 0.80`.  
    - Configuration: Uses JavaScript expression `{{$json.results.filter(item => item.score > 0.80)}}`.  
    - Inputs: Receives search results from Tavily Search Topic node.  
    - Outputs: Passes filtered results array.  
    - Edge Cases: No results meeting the threshold will result in empty array, potentially causing downstream nodes to fail.  
    - Sub-workflow: None

#### 1.5 Content Extraction

- **Overview:**  
  Extracts detailed web page content from the top filtered search result URL using Tavily's Extract API.

- **Nodes Involved:**  
  - Get Top Result  
  - Tavily Extract Top Search

- **Node Details:**  
  - **Get Top Result**  
    - Type: Set Node  
    - Role: Selects the first item from the filtered results array as the top search result.  
    - Configuration: Assigns `results` to the first element of `$json.results`.  
    - Inputs: Receives filtered results.  
    - Outputs: Passes single top result object.  
    - Edge Cases: Empty input array leads to undefined output, causing failures downstream.  
    - Sub-workflow: None

  - **Tavily Extract Top Search**  
    - Type: HTTP Request  
    - Role: Sends POST request to Tavily Extract endpoint with the URL of the top search result.  
    - Configuration:  
      - URL: `https://api.tavily.com/extract`  
      - Method: POST  
      - Body: JSON including `api_key` (from Tavily API Key node) and `urls` array containing the top result URL.  
    - Expressions: Uses `{{ $('Tavily API Key').item.json.api_key }}` and `{{ $json.results.url }}`.  
    - Inputs: Receives top result object.  
    - Outputs: Returns extracted content JSON including raw HTML content and metadata.  
    - Edge Cases: Invalid or inaccessible URL, API errors, network timeouts, empty extraction results.  
    - Sub-workflow: None

#### 1.6 AI Summarization

- **Overview:**  
  Summarizes the extracted web page content into a concise markdown format using OpenAI's Chat Model.

- **Nodes Involved:**  
  - Summarize Web Page Content  
  - OpenAI Chat Model

- **Node Details:**  
  - **Summarize Web Page Content**  
    - Type: LangChain Chain LLM  
    - Role: Defines the prompt to summarize the raw content from the extracted web page.  
    - Configuration: Prompt text set to `"Summarize this web content and provide in Markdown format:  {{ $json.results[0].raw_content }}"`.  
    - Inputs: Receives extracted content JSON.  
    - Outputs: Passes prompt and content to OpenAI Chat Model.  
    - Edge Cases: Missing or malformed raw content causes prompt failure.  
    - Sub-workflow: None

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Processes the summarization prompt and generates markdown summary.  
    - Configuration: Uses connected OpenAI API credentials.  
    - Inputs: Receives prompt from Summarize Web Page Content node.  
    - Outputs: Returns AI-generated summary text.  
    - Edge Cases: API rate limits, invalid credentials, network issues, prompt length limits.  
    - Sub-workflow: None

#### 1.7 Documentation and Notes

- **Overview:**  
  Provides embedded documentation, API references, usage instructions, and setup notes via sticky notes for user guidance.

- **Nodes Involved:**  
  - Sticky Note1 (Tavily API Search Endpoint)  
  - Sticky Note (Tavily API Extract Endpoint)  
  - Sticky Note2 (Tavily API Documentation)  
  - Sticky Note3 (Tavily Use Cases)  
  - Sticky Note4 (Tavily API Overview)  
  - Sticky Note5 (Workflow Title and Summary)  
  - Sticky Note6 (Tavily API Key Setup)

- **Node Details:**  
  - All sticky notes are informational only, containing URLs, API parameter descriptions, example requests, and best practices.  
  - Positioned around the workflow for easy reference.  
  - No inputs or outputs.  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                        | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                         |
|-----------------------------|---------------------------------|-------------------------------------|----------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| Provide search topic via Chat window | LangChain Chat Trigger           | User input reception                 | None                             | Tavily API Key                 |                                                                                                   |
| Tavily API Key              | Set                             | Stores Tavily API key                | Provide search topic via Chat window | Tavily Search Topic            | "### Tavily API Key https://app.tavily.com/home"                                                  |
| Tavily Search Topic         | HTTP Request                    | Executes Tavily Search API call     | Tavily API Key                   | Filter > 90%                  |                                                                                                   |
| Filter > 90%               | Set                             | Filters search results by score     | Tavily Search Topic              | Get Top Result                |                                                                                                   |
| Get Top Result             | Set                             | Selects top search result           | Filter > 90%                    | Tavily Extract Top Search      |                                                                                                   |
| Tavily Extract Top Search  | HTTP Request                    | Extracts content from top URL       | Get Top Result                  | Summarize Web Page Content     |                                                                                                   |
| Summarize Web Page Content | LangChain Chain LLM             | Prepares summarization prompt       | Tavily Extract Top Search       | OpenAI Chat Model              |                                                                                                   |
| OpenAI Chat Model          | LangChain OpenAI Chat Model     | Generates AI summary                 | Summarize Web Page Content      | None                          |                                                                                                   |
| Sticky Note1               | Sticky Note                    | Tavily API Search Endpoint details  | None                           | None                          | "## Tavily API Search Endpoint ... Example Request ..."                                           |
| Sticky Note                | Sticky Note                    | Tavily API Extract Endpoint details | None                           | None                          | "## Tavily API Extract Endpoint ... Example Request ..."                                          |
| Sticky Note2               | Sticky Note                    | Tavily API Documentation overview   | None                           | None                          | "## Tavily API Documentation ... Error handling should check for failed results ..."              |
| Sticky Note3               | Sticky Note                    | Tavily Use Cases and references      | None                           | None                          | "## Tavily Use Cases ... GPT Researcher ..."                                                      |
| Sticky Note4               | Sticky Note                    | Tavily API Overview and benefits     | None                           | None                          | "## Tavily API Overview ... Implementation Benefits ..."                                         |
| Sticky Note5               | Sticky Note                    | Workflow title and summary           | None                           | None                          | "## Tavily Search and Extract with AI Summarization Example"                                     |
| Sticky Note6               | Sticky Note                    | Tavily API Key setup reminder        | None                           | None                          | "### Tavily API Key https://app.tavily.com/home"                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Add a LangChain Chat Trigger node named "Provide search topic via Chat window".  
   - Configure webhook ID (auto-generated or custom).  
   - This node will receive user input as `chatInput`.

2. **Create Set Node for API Key**  
   - Add a Set node named "Tavily API Key".  
   - Add a string field `api_key` with value `tvly-YOUR_API_KEY` (replace with your actual key).  
   - Connect output of chat trigger node to this node.

3. **Create HTTP Request Node for Tavily Search**  
   - Add an HTTP Request node named "Tavily Search Topic".  
   - Set method to POST.  
   - Set URL to `https://api.tavily.com/search`.  
   - Set content type to `application/json`.  
   - Set body to JSON with keys:  
     - `api_key`: `{{ $json.api_key }}`  
     - `query`: `{{ $('Provide search topic via Chat window').item.json.chatInput }}`  
     - `search_depth`: `"basic"`  
     - `include_answer`: `false`  
     - `include_images`: `true`  
     - `include_image_descriptions`: `true`  
     - `include_raw_content`: `false`  
     - `max_results`: `5`  
     - `include_domains`: `[]`  
     - `exclude_domains`: `[]`  
   - Connect output of "Tavily API Key" node to this node.

4. **Create Set Node to Filter Results**  
   - Add a Set node named "Filter > 90%".  
   - Add an assignment for `results` with expression:  
     `{{$json.results.filter(item => item.score > 0.80)}}`  
   - Enable "Include Other Fields" to true.  
   - Connect output of "Tavily Search Topic" node to this node.

5. **Create Set Node to Get Top Result**  
   - Add a Set node named "Get Top Result".  
   - Add an assignment for `results` with expression:  
     `{{$json.results.first()}}`  
   - Connect output of "Filter > 90%" node to this node.

6. **Create HTTP Request Node for Tavily Extract**  
   - Add an HTTP Request node named "Tavily Extract Top Search".  
   - Set method to POST.  
   - Set URL to `https://api.tavily.com/extract`.  
   - Set content type to `application/json`.  
   - Set body to JSON with keys:  
     - `api_key`: `{{ $('Tavily API Key').item.json.api_key }}`  
     - `urls`: `[ "{{ $json.results.url }}" ]`  
   - Connect output of "Get Top Result" node to this node.

7. **Create LangChain Chain LLM Node for Summarization Prompt**  
   - Add a LangChain Chain LLM node named "Summarize Web Page Content".  
   - Set prompt type to "define".  
   - Set text to:  
     `Summarize this web content and provide in Markdown format:  {{ $json.results[0].raw_content }}`  
   - Connect output of "Tavily Extract Top Search" node to this node.

8. **Create LangChain OpenAI Chat Model Node**  
   - Add a LangChain OpenAI Chat Model node named "OpenAI Chat Model".  
   - Connect your OpenAI API credentials to this node.  
   - Connect output of "Summarize Web Page Content" node to this node.

9. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add sticky notes with content from the original workflow to provide API references, usage instructions, and setup notes.  
   - Position them logically near related nodes for user guidance.

10. **Connect Nodes in Order:**  
    - "Provide search topic via Chat window" → "Tavily API Key" → "Tavily Search Topic" → "Filter > 90%" → "Get Top Result" → "Tavily Extract Top Search" → "Summarize Web Page Content" → "OpenAI Chat Model"

11. **Set Credentials:**  
    - Configure OpenAI credentials in "OpenAI Chat Model" node.  
    - No explicit credential node for Tavily API; the API key is set as a string in the Set node.

12. **Deploy and Test:**  
    - Deploy the workflow.  
    - Trigger via chat interface with a research topic.  
    - Verify that the workflow returns a markdown summary of relevant web content.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Tavily API Search Endpoint documentation and parameters with example requests.                            | https://docs.tavily.com/docs/rest-api/examples          |
| Tavily API Extract Endpoint documentation and example requests.                                          | https://docs.tavily.com/docs/rest-api/examples          |
| Tavily API general documentation and overview of features and benefits for AI integration.               | https://docs.tavily.com/docs/welcome                     |
| Use cases for Tavily API including data enrichment and company research.                                 | https://docs.tavily.com/docs/use-cases/data-enrichment  |
| GPT Researcher use case with Tavily API integration.                                                     | https://docs.tavily.com/docs/gpt-researcher/introduction|
| Reminder to obtain Tavily API key from https://app.tavily.com/home                                       | https://app.tavily.com/home                              |
| Workflow demonstrates practical integration of Tavily Search and Extract APIs with AI summarization.     | Workflow title and description                           |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the "🤖🔍 The Ultimate Free AI-Powered Researcher with Tavily Web Search & Extract" workflow in n8n. It covers all nodes, their configurations, data flow, and potential failure points to facilitate robust usage and customization.