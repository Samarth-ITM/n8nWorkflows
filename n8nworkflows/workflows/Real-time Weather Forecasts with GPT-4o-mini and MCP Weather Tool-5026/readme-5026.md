Real-time Weather Forecasts with GPT-4o-mini and MCP Weather Tool

https://n8nworkflows.xyz/workflows/real-time-weather-forecasts-with-gpt-4o-mini-and-mcp-weather-tool-5026


# Real-time Weather Forecasts with GPT-4o-mini and MCP Weather Tool

### 1. Workflow Overview

This workflow, titled **"Real-time Weather Forecasts with GPT-4o-mini and MCP Weather Tool"**, is designed to provide real-time weather information and forecasts in response to user text queries. It leverages a combination of AI language understanding (via GPT-4o-mini) and a specialized weather data API (MCP WeatherTrax) to parse user requests and fetch accurate weather data for specified locations and durations.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Capture and set the user’s weather query.
- **1.2 AI Processing and Intent Extraction:** Use GPT-4o-mini and a custom AI Agent to interpret the user’s request and determine the required weather data parameters.
- **1.3 Weather Data Retrieval:** Call the MCP WeatherTrax HTTP API with structured parameters extracted by the AI Agent.
- **1.4 Results Presentation:** Format and display the results for review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block is responsible for initiating the workflow and providing the user's weather-related query as input.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `🧪 Enter weather request`

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type & Role:* Manual Trigger node; initiates the workflow manually.  
    - *Configuration:* No parameters; user manually triggers execution.  
    - *Input/Output:* No input; outputs to `🧪 Enter weather request`.  
    - *Edge Cases:* None typical; manual trigger avoids automatic runtime errors.  

  - **🧪 Enter weather request**  
    - *Type & Role:* Set node; sets a fixed string representing the user's weather query.  
    - *Configuration:* Sets string variable `Weather Request` with value `"What is the weather in New York City?"` (default example).  
    - *Input/Output:* Input from manual trigger; output feeds into `AI Agent`.  
    - *Edge Cases:* The static input could be replaced with dynamic inputs in production; no validation on string content here.  

#### 1.2 AI Processing and Intent Extraction

- **Overview:**  
  This block uses OpenAI's GPT-4o-mini model combined with a custom AI Agent node to interpret the user's natural language weather query. The AI Agent is configured to extract parameters needed for the weather API call (location, query type, number of forecast days).

- **Nodes Involved:**  
  - `OpenAI Chat Model`  
  - `AI Agent`

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type & Role:* Language Model node (GPT-4o-mini); provides the underlying LLM for the AI Agent node.  
    - *Configuration:* Uses the `gpt-4o-mini` model via OpenAI credentials. No additional options set; defaults used.  
    - *Input/Output:* Connected as the language model for `AI Agent` via `ai_languageModel` connection.  
    - *Edge Cases:* Possible API authentication errors, rate limits, or timeouts. Model must support conversation context.  

  - **AI Agent**  
    - *Type & Role:* LangChain Agent node; orchestrates AI calls with a system message defining strict rules for weather queries.  
    - *Configuration:*  
      - Reads the variable `{{$json['Weather Request']}}` as input text.  
      - System message instructs the AI to always call the connected `weatherTool` function with strict parameters (`location`, `query_type`, `num_days`).  
      - Provides examples of how to parse user requests into function calls.  
      - `promptType` set to `define` to ensure fixed instruction-based prompt.  
    - *Input/Output:*  
      - Input from `🧪 Enter weather request`.  
      - Outputs to `🧪 View Results Here` (main output) and `MCP - WeatherTrax` (tool call output).  
    - *Edge Cases:*  
      - Failure to correctly parse location or query type.  
      - Model may return malformed function calls.  
      - Network or API issues with OpenAI.  
      - Requires LangChain v2 node compatibility.  

#### 1.3 Weather Data Retrieval

- **Overview:**  
  This block calls the MCP WeatherTrax API using the structured parameters extracted by the AI Agent to get real-time weather or forecasts.

- **Nodes Involved:**  
  - `MCP - WeatherTrax`

- **Node Details:**

  - **MCP - WeatherTrax**  
    - *Type & Role:* HTTP Request Tool node; sends POST request to MCP WeatherTrax API.  
    - *Configuration:*  
      - URL: `https://mcp-weathertrax.jaredco.com`  
      - Method: POST  
      - Sends JSON body dynamically injected from AI Agent’s tool call output (using a placeholder for AI override: `$fromAI('JSON', ``, 'json')`).  
      - Tool description clarifies it supports current weather, forecasts, and multi-day predictions.  
    - *Input/Output:*  
      - Input from `AI Agent` via the `ai_tool` connection.  
      - Output not connected further but implicitly feeds back to AI Agent for possible use or to `🧪 View Results Here`.  
    - *Edge Cases:*  
      - HTTP errors (timeouts, 4xx/5xx responses).  
      - Malformed JSON from AI Agent causing API call failure.  
      - Network connectivity issues.  
      - API quota or authentication issues if applicable.  

#### 1.4 Results Presentation

- **Overview:**  
  This block formats and displays the AI Agent’s output and the parameters used for the weather API call for human review.

- **Nodes Involved:**  
  - `🧪 View Results Here`

- **Node Details:**

  - **🧪 View Results Here**  
    - *Type & Role:* Code node; extracts and formats key pieces of information for visualization.  
    - *Configuration:*  
      - JavaScript code returns an object containing:  
        - `fullAgentOutput`: entire JSON output from previous node (`AI Agent`).  
        - `toolCall`: the function call object issued by the AI.  
        - `toolParams`: inputs parsed for the weather tool call.  
        - `message`: a static message prompting users to click this node for details.  
    - *Input/Output:*  
      - Input from `AI Agent` main output.  
      - Output for viewing in the n8n editor UI.  
    - *Edge Cases:*  
      - Failures if expected JSON structure is missing or malformed.  
      - No fallback formatting if properties are undefined.  

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                   | Input Node(s)               | Output Node(s)            | Sticky Note                              |
|----------------------------|----------------------------------|---------------------------------|-----------------------------|---------------------------|-----------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                    | Initiates workflow manually      | —                           | 🧪 Enter weather request   |                                         |
| 🧪 Enter weather request    | Set                              | Sets the user weather query      | When clicking ‘Execute workflow’ | AI Agent                  |                                         |
| OpenAI Chat Model           | LangChain Chat LLM (GPT-4o-mini) | Provides language model for AI   | — (credential-based)         | AI Agent (ai_languageModel) |                                         |
| AI Agent                   | LangChain Agent                  | Parses user query and calls tool | 🧪 Enter weather request; OpenAI Chat Model | MCP - WeatherTrax (ai_tool), 🧪 View Results Here (main) | System message enforces strict weatherTool calls with parameters. |
| MCP - WeatherTrax           | HTTP Request Tool                | Calls external weather API       | AI Agent (ai_tool)           | AI Agent                   | Sends JSON body extracted dynamically from AI output.            |
| 🧪 View Results Here        | Code                             | Formats and displays results     | AI Agent (main)              | —                          | Shows detailed tool call and parsed input for debugging.         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.  
   - No special configuration required.  

2. **Create Set Node for Weather Request:**  
   - Add a **Set** node named `🧪 Enter weather request`.  
   - Configure to set a string field:  
     - Key: `Weather Request`  
     - Value: `"What is the weather in New York City?"` (default example)  
   - Connect output of `When clicking ‘Execute workflow’` to this node.  

3. **Add OpenAI Chat Model Node:**  
   - Add a **LangChain Chat OpenAI** node named `OpenAI Chat Model`.  
   - Choose model `gpt-4o-mini` from the model list.  
   - Set credentials for OpenAI API access (must create and link OpenAI API credential node).  
   - No additional options needed.  

4. **Add AI Agent Node:**  
   - Add a **LangChain Agent** node named `AI Agent`.  
   - Set the input text expression as: `={{ $json["Weather Request"] }}`.  
   - Set `promptType` to `define`.  
   - In the system message, paste:  
     ```
     When users ask about weather, you MUST call the connected weatherTool function with these exact parameters:
     - location: (string) The city or location name
     - query_type: (string) Either "current" for current weather or "multi_day" for forecasts
     - num_days: (integer) Number of days for forecast (only when using "multi_day")

     IMPORTANT: Always use the function call format for weather requests.

     Examples:
     - "Weather in Boca Raton" → Call weatherTool with location="Boca Raton", query_type="current"
     - "5 day forecast for Miami" → Call weatherTool with location="Miami", query_type="multi_day", num_days=5

     Always extract the location from the user's request and call the weatherTool function.
     ```  
   - Connect `🧪 Enter weather request` to `AI Agent` main input.  
   - Connect `OpenAI Chat Model` to `AI Agent` via the `ai_languageModel` connection.  

5. **Add HTTP Request Tool Node for Weather API:**  
   - Add an **HTTP Request Tool** node named `MCP - WeatherTrax`.  
   - Set method to POST.  
   - Set URL to `https://mcp-weathertrax.jaredco.com`.  
   - Configure to send a JSON body.  
   - For the JSON body, use the dynamic expression to pull from the AI Agent’s tool call output:  
     ```
     {{$fromAI('JSON', '', 'json')}}
     ```  
     (This is a placeholder for AI override to inject the JSON generated by AI Agent).  
   - Connect `AI Agent` to `MCP - WeatherTrax` via the `ai_tool` connection (special connection for tool calls).  

6. **Add Code Node to View Results:**  
   - Add a **Code** node named `🧪 View Results Here`.  
   - Set language to JavaScript.  
   - Paste the following code:  
     ```javascript
     return [{
       fullAgentOutput: $json,
       toolCall: $json.tool_call,
       toolParams: $json.tool_call?.input,
       message: '✅ Click this node to view the tool call and parsed input!'
     }];
     ```  
   - Connect `AI Agent` main output to this node.  

7. **Verify Connections:**  
   - `When clicking ‘Execute workflow’` → `🧪 Enter weather request` → `AI Agent`  
   - `OpenAI Chat Model` → `AI Agent` (`ai_languageModel` input)  
   - `AI Agent` (`ai_tool` output) → `MCP - WeatherTrax`  
   - `AI Agent` (main output) → `🧪 View Results Here`  

8. **Set Credentials:**  
   - Configure OpenAI API credentials for `OpenAI Chat Model` node.  
   - No credentials required for HTTP Request Tool unless the MCP API requires (not indicated).  

9. **Save and Test:**  
   - Activate the workflow (optional).  
   - Trigger manually using `When clicking ‘Execute workflow’`.  
   - Observe the output in the `🧪 View Results Here` node to verify extraction and API call parameters.  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                              |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| The AI Agent enforces strict function call format to ensure standardized weather queries.       | System message in AI Agent node.                                            |
| The MCP WeatherTrax API supports current weather and multi-day forecasts via JSON parameters.   | API URL: https://mcp-weathertrax.jaredco.com                                |
| For best results, customize the initial weather query input to accept dynamic user inputs.      | See `🧪 Enter weather request` node configuration.                           |
| LangChain nodes require version 2 or higher for compatibility with this workflow’s features.    | Relevant for `AI Agent` and `OpenAI Chat Model` nodes.                      |
| Use the `🧪 View Results Here` node to debug and verify AI parsing and API call structure.       | Critical for troubleshooting parsing and integration issues.               |

---

**Disclaimer:**  
This text is generated exclusively from an automated n8n workflow analysis respecting content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and public.