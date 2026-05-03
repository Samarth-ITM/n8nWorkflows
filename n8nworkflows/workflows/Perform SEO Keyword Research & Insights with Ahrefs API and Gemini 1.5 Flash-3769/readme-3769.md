Perform SEO Keyword Research & Insights with Ahrefs API and Gemini 1.5 Flash

https://n8nworkflows.xyz/workflows/perform-seo-keyword-research---insights-with-ahrefs-api-and-gemini-1-5-flash-3769


# Perform SEO Keyword Research & Insights with Ahrefs API and Gemini 1.5 Flash

### 1. Workflow Overview

This n8n workflow automates **SEO keyword research** by integrating the **Ahrefs Keyword Tool API** with AI-powered language models (Google Gemini 1.5 Flash) to deliver enriched keyword insights and recommendations. It is designed for SEO specialists, content marketers, and digital agencies aiming to quickly analyze keywords, retrieve related keyword data, and receive formatted, actionable SEO advice.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the user’s keyword query via a chat trigger.
- **1.2 AI Input Verification & Cleaning:** Uses an AI agent to validate and correct the keyword input.
- **1.3 Ahrefs API Data Retrieval:** Queries the Ahrefs API with the cleaned keyword to obtain detailed keyword metrics.
- **1.4 Data Extraction & Aggregation:** Processes the raw API response to extract main keyword data and up to 10 related keywords.
- **1.5 AI Formatting & Response Generation:** Uses an AI agent to format the aggregated data into a clean, user-friendly text output.
- **1.6 Final Output Delivery:** Sends the formatted SEO insights back to the user.

The workflow also includes robust error handling and retry mechanisms to ensure reliability when interacting with external APIs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the initial keyword query from the user via a chat interface, serving as the workflow’s entry point.

- **Nodes Involved:**  
  - When chat message received  
  - Sticky Note (Trigger Node)

- **Node Details:**  

  - **When chat message received**  
    - *Type:* Chat Trigger (LangChain)  
    - *Role:* Listens for incoming chat messages containing the keyword query.  
    - *Configuration:* Default webhook setup to receive chat input.  
    - *Input:* External chat message from user.  
    - *Output:* Passes raw user message downstream.  
    - *Edge Cases:* Missing or empty input, unsupported message formats.  
    - *Sticky Note:* Explains this node is a sample trigger and can be replaced by Telegram, WhatsApp, or webhook nodes.

  - **Sticky Note (Trigger Node)**  
    - *Type:* Sticky Note  
    - *Role:* Documentation for users explaining the trigger node purpose.

---

#### 2.2 AI Input Verification & Cleaning

- **Overview:**  
  Validates and cleans the user’s keyword input using an AI agent to ensure only a single, grammatically correct SEO keyword is passed to the API.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Keyword Query Extraction & Cleaning Agent  
  - Sticky Note (Keyword Query Extraction)

- **Node Details:**  

  - **Google Gemini Chat Model**  
    - *Type:* AI Language Model (Google Gemini 1.5 Flash)  
    - *Role:* Provides natural language understanding capabilities to process user input.  
    - *Configuration:* Model set to "models/gemini-1.5-flash" with default options.  
    - *Credentials:* Uses Google PaLM API credentials.  
    - *Input:* Raw user message from chat trigger.  
    - *Output:* Processed text passed to the next AI agent.  
    - *Edge Cases:* API rate limits, network errors, malformed input.

  - **Keyword Query Extraction & Cleaning Agent**  
    - *Type:* AI Agent (LangChain Agent)  
    - *Role:* Extracts exactly one SEO keyword from the input, correcting grammar without rephrasing or commentary.  
    - *Configuration:* System message instructs to output only one corrected keyword phrase.  
    - *Input:* Output from Google Gemini Chat Model.  
    - *Output:* Cleaned keyword string for API query.  
    - *Edge Cases:* Ambiguous input, multiple keywords, or no keyword detected.  
    - *Sticky Note:* Emphasizes importance of this node to ensure only the keyword phrase is sent to the API and correct misspellings.

---

#### 2.3 Ahrefs API Data Retrieval

- **Overview:**  
  Sends the cleaned keyword to the Ahrefs Keyword Tool API to retrieve detailed keyword metrics including search volume, competition, and CPC data.

- **Nodes Involved:**  
  - Ahrefs Keyword API Request  
  - Sticky Note (API Request)

- **Node Details:**  

  - **Ahrefs Keyword API Request**  
    - *Type:* HTTP Request  
    - *Role:* Queries the Ahrefs API endpoint `/global-volume` with the cleaned keyword.  
    - *Configuration:*  
      - URL: `https://ahrefs-keyword-tool.p.rapidapi.com/global-volume`  
      - Query parameter: `keyword` set dynamically from cleaned keyword.  
      - Headers: Includes `x-rapidapi-host` and `x-rapidapi-key` (user must replace `"your_rapid_api_key_here"` with actual key).  
      - Retry: Enabled with max 2 tries on failure.  
    - *Input:* Cleaned keyword string.  
    - *Output:* Raw JSON response from Ahrefs API containing keyword data.  
    - *Edge Cases:* Authentication errors, API rate limits, network timeouts, invalid keyword input.  
    - *Sticky Note:* Provides link to Ahrefs API docs and notes on tweaking API endpoints.

---

#### 2.4 Data Extraction & Aggregation

- **Overview:**  
  Extracts relevant fields from the large API response, focusing on the main keyword data and up to 10 related keywords, then aggregates the data for formatting.

- **Nodes Involved:**  
  - Extract Main Keyword & 10 related Keyword data (Code node)  
  - Aggregate Keyword Data  
  - Sticky Note (Extract Keyword Data)

- **Node Details:**  

  - **Extract Main Keyword & 10 related Keyword data**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses the API response JSON to extract key metrics for the main keyword and related keywords.  
    - *Configuration:*  
      - Extracts fields: keyword, avg_monthly_searches, competition_index, competition_value, high_cpc, low_cpc.  
      - Limits related keywords to 10 entries.  
    - *Input:* Raw API response JSON.  
    - *Output:* Array of objects with cleaned keyword data.  
    - *Edge Cases:* Missing fields in API response, empty related keywords array.  
    - *Sticky Note:* Explains the purpose of this node and encourages customization.

  - **Aggregate Keyword Data**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all extracted keyword data items into a single data structure for downstream processing.  
    - *Configuration:* Uses "aggregateAllItemData" to combine all items.  
    - *Input:* Output array from the code node.  
    - *Output:* Single aggregated JSON object.  
    - *Edge Cases:* Empty input array.

---

#### 2.5 AI Formatting & Response Generation

- **Overview:**  
  Formats the aggregated keyword data into a clean, readable text format using an AI agent, preparing the final SEO insights for user consumption.

- **Nodes Involved:**  
  - Google Gemini Chat Model1  
  - Keyword Data Response Formatter  
  - Sticky Note (Response Formatter)

- **Node Details:**  

  - **Keyword Data Response Formatter**  
    - *Type:* AI Agent (LangChain Agent)  
    - *Role:* Formats the main keyword and related keywords data into a user-friendly text output.  
    - *Configuration:*  
      - System message defines the formatting template, listing keyword metrics clearly for the main keyword and each related keyword (up to 10).  
      - Uses n8n expressions to dynamically insert data fields into the message.  
    - *Input:* Aggregated keyword data JSON.  
    - *Output:* Formatted text response.  
    - *Edge Cases:* Missing data fields, formatting errors, AI model failures.  
    - *Sticky Note:* Notes flexibility to customize the system message for different output styles.

  - **Google Gemini Chat Model1**  
    - *Type:* AI Language Model (Google Gemini 1.5 Flash)  
    - *Role:* Supports the formatting agent with language model capabilities.  
    - *Configuration:* Same model as earlier, with Google PaLM API credentials.  
    - *Input:* Aggregated keyword data.  
    - *Output:* Formatted text passed to response formatter.  
    - *Edge Cases:* API errors, rate limits.

---

#### 2.6 Final Output Delivery

- **Overview:**  
  The final formatted SEO keyword insights are delivered back to the user via the chat interface (implicit in the workflow’s trigger-response design).

- **Nodes Involved:**  
  - (No explicit output node shown; assumed response sent via chat trigger node’s webhook response or integration.)

- **Node Details:**  
  - The workflow’s last node outputs the formatted text which can be sent back to the user through the chat platform connected to the initial trigger.  
  - Edge cases include message delivery failures or connection issues with the chat platform.

---

### 3. Summary Table

| Node Name                           | Node Type                          | Functional Role                          | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                                                   |
|-----------------------------------|----------------------------------|----------------------------------------|----------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When chat message received         | Chat Trigger (LangChain)          | Captures user keyword input             | -                                | Keyword Query Extraction & Cleaning Agent | Explains this is a sample trigger node; can be replaced by Telegram, WhatsApp, webhook, etc.                                  |
| Google Gemini Chat Model            | AI Language Model (Google Gemini) | Processes raw user input                 | When chat message received        | Keyword Query Extraction & Cleaning Agent |                                                                                                                              |
| Keyword Query Extraction & Cleaning Agent | AI Agent (LangChain Agent)        | Extracts and cleans single SEO keyword  | Google Gemini Chat Model          | Ahrefs Keyword API Request            | Important to ensure only the keyword phrase is sent to API; corrects misspellings.                                             |
| Ahrefs Keyword API Request          | HTTP Request                     | Queries Ahrefs API for keyword data     | Keyword Query Extraction & Cleaning Agent | Extract Main Keyword & 10 related Keyword data | Link to API docs; can tweak endpoint for different keyword data types.                                                        |
| Extract Main Keyword & 10 related Keyword data | Code (JavaScript)                | Extracts main and related keyword metrics | Ahrefs Keyword API Request        | Aggregate Keyword Data                | Extracts key fields from large API response; customizable to extract more/different data.                                      |
| Aggregate Keyword Data              | Aggregate                       | Aggregates extracted keyword data       | Extract Main Keyword & 10 related Keyword data | Keyword Data Response Formatter       |                                                                                                                              |
| Google Gemini Chat Model1           | AI Language Model (Google Gemini) | Supports formatting of keyword data     | Aggregate Keyword Data            | Keyword Data Response Formatter       |                                                                                                                              |
| Keyword Data Response Formatter     | AI Agent (LangChain Agent)        | Formats keyword data into readable text | Google Gemini Chat Model1         | (Final output to user)                | Allows customization of output format; formats main and related keywords clearly.                                             |
| Sticky Note                       | Sticky Note                     | Documentation                          | -                                | -                                    | Explains trigger node usage.                                                                                                  |
| Sticky Note1                      | Sticky Note                     | Documentation                          | -                                | -                                    | Notes on API request node and link to Ahrefs API docs.                                                                        |
| Sticky Note2                      | Sticky Note                     | Documentation                          | -                                | -                                    | Explains purpose of keyword data extraction code node.                                                                        |
| Sticky Note3                      | Sticky Note                     | Documentation                          | -                                | -                                    | Explains trigger node options.                                                                                                |
| Sticky Note4                      | Sticky Note                     | Documentation                          | -                                | -                                    | Notes on response formatter AI agent and customization options.                                                               |
| Simple Memory                    | Memory Buffer Window             | (Unused or placeholder in this workflow) | -                                | -                                    |                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Chat Trigger** node (LangChain chatTrigger).  
   - Configure webhook to receive chat messages containing the keyword query.

2. **Add AI Language Model Node (Input Processing):**  
   - Add **Google Gemini Chat Model** node.  
   - Set model to `"models/gemini-1.5-flash"`.  
   - Connect credentials for Google PaLM API.  
   - Connect input from the Chat Trigger node.

3. **Add AI Agent Node (Keyword Extraction & Cleaning):**  
   - Add **LangChain Agent** node.  
   - Configure system message to instruct extracting exactly one SEO keyword, correcting grammar but no commentary.  
   - Connect input from Google Gemini Chat Model node.

4. **Add HTTP Request Node (Ahrefs API):**  
   - Add **HTTP Request** node.  
   - Set method to GET.  
   - URL: `https://ahrefs-keyword-tool.p.rapidapi.com/global-volume`.  
   - Add query parameter `keyword` with value from previous node’s cleaned keyword output.  
   - Add headers:  
     - `x-rapidapi-host`: `ahrefs-keyword-tool.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key (replace placeholder).  
   - Enable retry on failure with max 2 tries.  
   - Connect input from Keyword Extraction & Cleaning Agent node.

5. **Add Code Node (Extract Keyword Data):**  
   - Add **Code** node (JavaScript).  
   - Paste the provided script to extract main keyword data and up to 10 related keywords with selected fields.  
   - Connect input from Ahrefs API Request node.

6. **Add Aggregate Node:**  
   - Add **Aggregate** node.  
   - Configure to aggregate all item data into one JSON object.  
   - Connect input from Code node.

7. **Add AI Language Model Node (Formatting Support):**  
   - Add another **Google Gemini Chat Model** node with same model and credentials.  
   - Connect input from Aggregate node.

8. **Add AI Agent Node (Response Formatter):**  
   - Add **LangChain Agent** node.  
   - Configure system message with the detailed formatting template to output main and related keyword data clearly.  
   - Use n8n expressions to map data fields dynamically.  
   - Connect input from second Google Gemini Chat Model node.

9. **Connect Final Output:**  
   - Configure the workflow or chat platform integration to send the formatted text response back to the user.

10. **Add Sticky Notes:**  
    - Add sticky notes at appropriate places to document trigger usage, API request details, data extraction purpose, and response formatting customization.

11. **Credentials Setup:**  
    - Set up Google PaLM API credentials for Gemini nodes.  
    - Set up RapidAPI key for Ahrefs API HTTP Request node.

12. **Test Workflow:**  
    - Trigger the workflow with sample keywords.  
    - Verify keyword extraction, API data retrieval, data formatting, and final output.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| To use the Ahrefs API, sign up and get your API key at:                                                          | [Ahrefs Keyword Tool API on RapidAPI](https://rapidapi.com/environmentn1t21r5/api/ahrefs-keyword-tool)    |
| The API request node can be customized to query different endpoints or parameters as per Ahrefs API documentation | [Ahrefs API Playground & Docs](https://rapidapi.com/environmentn1t21r5/api/ahrefs-keyword-tool/playground/apiendpoint_d2790246-c8ef-437f-b928-c0eb6f6ffff4) |
| The AI agent system messages are fully customizable to adjust keyword extraction and response formatting          | Modify system messages in LangChain Agent nodes to tailor behavior and output style                       |
| This workflow includes retry logic on API failures to improve robustness                                          | HTTP Request node configured with retryOnFail enabled and maxTries set to 2                              |
| The JavaScript extraction code can be modified to extract additional metrics or change the number of related keywords | Located in the "Extract Main Keyword & 10 related Keyword data" node                                     |
| The workflow is designed to be extensible with other chat platforms or triggers replacing the initial chat trigger | Replace "When chat message received" node with Telegram, WhatsApp, or webhook nodes as needed            |

---

This documentation provides a complete, structured reference to understand, reproduce, and customize the SEO keyword research workflow integrating Ahrefs API and Google Gemini AI models within n8n.