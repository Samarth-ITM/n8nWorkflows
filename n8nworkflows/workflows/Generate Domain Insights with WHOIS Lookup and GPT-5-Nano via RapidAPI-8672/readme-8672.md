Generate Domain Insights with WHOIS Lookup and GPT-5-Nano via RapidAPI

https://n8nworkflows.xyz/workflows/generate-domain-insights-with-whois-lookup-and-gpt-5-nano-via-rapidapi-8672


# Generate Domain Insights with WHOIS Lookup and GPT-5-Nano via RapidAPI

### 1. Workflow Overview

This workflow automates the process of retrieving WHOIS domain information, analyzing it with an AI language model (GPT-5-Nano), and returning a well-formatted HTML summary card. It is designed for use cases where users want quick, clear insights into domain registration data enriched with an AI-generated description.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives an external HTTP request via a webhook to trigger the workflow.
- **1.2 Configuration Setup:** Defines key parameters such as target domain, language, API keys, and search options.
- **1.3 WHOIS Data Retrieval:** Queries a WHOIS API via RapidAPI to fetch detailed domain registration data.
- **1.4 AI Processing:** Sends the WHOIS data to GPT-5-Nano to generate a concise analysis; optionally uses a Bing search tool.
- **1.5 Response Formatting and Delivery:** Constructs a styled HTML card with WHOIS details and AI insights, then responds to the initial webhook request.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  The entry point of the workflow, this block captures incoming HTTP requests and triggers the automation.

- **Nodes Involved:**  
  - Webhook  
  - OPTIONS (Configuration Node, triggered immediately after webhook)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook trigger node  
    - Configuration: Path set to a unique UUID string; uses response mode "responseNode" (defers response until Respond to Webhook node)  
    - Inputs: External HTTP requests  
    - Outputs: Triggers OPTIONS node downstream  
    - Potential Failures: Invalid webhook path, unauthorized requests, network issues

  - **OPTIONS**  
    - Type: Set node for variable initialization  
    - Configuration: Assigns static values for `lang` ("spanish"), `search_web` (true), `domain` ("ergates.net"), and RapidAPI key  
    - Inputs: Triggered by Webhook node  
    - Outputs: Provides configuration JSON for downstream nodes  
    - Edge Cases: Hardcoded domain and keys; if parameters need to be dynamic, this node requires modification

---

#### 2.2 WHOIS Data Retrieval

- **Overview:**  
  Queries the RapidAPI WHOIS service to fetch domain registration data in a split format for easy parsing.

- **Nodes Involved:**  
  - Request API

- **Node Details:**

  - **Request API**  
    - Type: HTTP Request node  
    - Configuration:  
      - URL: `https://pointsdb-bulk-whois-v1.p.rapidapi.com/whois`  
      - Query Parameters: `domains` set dynamically from OPTIONS domain; `format` set to "split"  
      - Headers: `x-rapidapi-host` and `x-rapidapi-key` from OPTIONS  
    - Inputs: OPTIONS node output  
    - Outputs: JSON array with WHOIS fields indexed by line number for templating  
    - Potential Failures: Invalid API key, rate limiting, network errors, unexpected response format

---

#### 2.3 AI Processing

- **Overview:**  
  Passes the WHOIS data and context to GPT-5-Nano for summarization. Optionally uses an HTTP Request tool node as a search tool triggered by the model for web queries.

- **Nodes Involved:**  
  - Message a model (OpenAI GPT-5-Nano)  
  - HTTP Request1 (AI tool for web search)

- **Node Details:**

  - **Message a model**  
    - Type: LangChain OpenAI node  
    - Configuration:  
      - Model: GPT-5-Nano  
      - System message: Instructions to analyze the WHOIS, produce a brief description, optionally search web if `search_web` is true, respond in configured language  
      - User message: Includes WHOIS data lines formatted as HTML list items  
    - Inputs: WHOIS data from Request API node; AI tool input from HTTP Request1  
    - Outputs: AI response with domain insight summary  
    - Credentials: OpenAI API credentials required  
    - Potential Failures: API quota exceeded, malformed input data, expression evaluation errors

  - **HTTP Request1**  
    - Type: HTTP Request (AI Tool)  
    - Configuration: Provides Bing search capability to AI when requested, URL parameter overridden dynamically by AI prompt  
    - Inputs: None directly, invoked as AI tool by Message a model node  
    - Outputs: Search results passed back to AI node  
    - Potential Failures: HTTP errors, invalid query parameter, API limits on Bing search

---

#### 2.4 Response Formatting and Delivery

- **Overview:**  
  Formats the WHOIS data and AI-generated descriptions into a responsive, styled HTML card and returns it as the webhook response.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - Type: Respond to Webhook node  
    - Configuration:  
      - Response Body: Custom HTML template embedding WHOIS fields and AI message content  
      - Style: CSS included for light/dark mode, grid layout, typography, and interactive hover effects  
    - Inputs: AI message output containing the domain analysis and WHOIS data from Request API node  
    - Outputs: HTTP response to original webhook request with complete HTML content  
    - Potential Failures: Expression errors extracting WHOIS fields, malformed HTML, large payloads exceeding size limits

---

### 3. Summary Table

| Node Name          | Node Type                        | Functional Role                              | Input Node(s)        | Output Node(s)     | Sticky Note                                                                                                                   |
|--------------------|---------------------------------|----------------------------------------------|----------------------|--------------------|------------------------------------------------------------------------------------------------------------------------------|
| Webhook            | Webhook                         | Entry point: receives external HTTP requests| -                    | OPTIONS            | The Webhook node receives external requests (HTTP) and acts as an entry point to the flow, starting execution when a request arrives at the generated URL. |
| OPTIONS            | Set                             | Configuration: sets domain, language, API key, search options | Webhook              | Request API        | The OPTIONS node defines and assigns initial variables (language, domain, API key, web search, etc.) that will be used in subsequent nodes in the flow to maintain centralized configuration. |
| Request API        | HTTP Request                    | Query WHOIS data from RapidAPI service       | OPTIONS              | Message a model     | The Request API node makes the call to the external WHOIS service (RapidAPI), sending the configured domain and returning the registration information in a split format so that the rest of the flow can process it. |
| Message a model     | LangChain OpenAI                | AI analysis of WHOIS data and optional web search | Request API, HTTP Request1 (ai_tool) | Respond to Webhook | Message to model: Sends the WHOIS (and context) to GPT-5-Nano to generate a short parse in the language defined in OPTIONS, and can invoke a web search if search_web is true. HTTP Request1 (AI Tool): Provides the AI ​​with a tool to launch HTTP searches (e.g., Bing) when prompted, returning the data to the model node. |
| HTTP Request1      | HTTP Request (AI Tool)          | AI tool for performing Bing web searches     | - (invoked by AI)    | Message a model (ai_tool) |                                                                                                                              |
| Respond to Webhook | Respond to Webhook              | Send final HTML response with WHOIS + AI data| Message a model       | -                  | The Respond to Webhook node sends the final output of the flow back to the requesting client. In this case, it returns a stylized HTML card with the processed WHOIS data and the AI-generated description, providing a clear, browser-ready presentation. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Set Path to a unique identifier (e.g., "76654401-5166-4a5a-94df-fa3a2d986142")  
   - Set Response Mode to "responseNode" to defer response  
   - Position: Start of the workflow

2. **Create OPTIONS Node**  
   - Type: Set  
   - Add variables with Assignments:  
     - `lang`: string, e.g., "spanish"  
     - `search_web`: boolean, true or false  
     - `domain`: string, e.g., "ergates.net" (can be dynamic)  
     - `rapidapi`: string, your RapidAPI key for WHOIS service  
   - Connect Webhook → OPTIONS

3. **Create Request API Node**  
   - Type: HTTP Request  
   - URL: `https://pointsdb-bulk-whois-v1.p.rapidapi.com/whois`  
   - Query Parameters:  
     - `domains`: set to `{{$json.domain}}` from OPTIONS  
     - `format`: "split"  
   - Headers:  
     - `x-rapidapi-host`: "pointsdb-bulk-whois-v1.p.rapidapi.com"  
     - `x-rapidapi-key`: `{{$json.rapidapi}}` from OPTIONS  
   - Connect OPTIONS → Request API

4. **Create HTTP Request1 Node for AI Tool**  
   - Type: HTTP Request  
   - Leave URL blank (will be overridden dynamically by AI node)  
   - This node acts as an AI tool to perform web searches when AI prompts it  
   - No direct input connection; AI node will invoke it as a tool

5. **Create Message a model Node**  
   - Type: LangChain OpenAI node  
   - Model: Select "gpt-5-nano"  
   - Credentials: Set OpenAI API credentials  
   - Messages Setup:  
     - System message:  
       ```
       Your task is to analyze the whois shared by the user and draw a conclusion or analysis based on your knowledge.

       Important: make a short description

       Forced to search the web: {{ $('OPTIONS').item.json.search_web }} (if this value is true you can use the tool to search the web, search for bing (https://www.bing.com/search?q=example)

       Respond in this language: {{ $('OPTIONS').item.json.lang }}
       ```  
     - User message:  
       ```
       Whois recivido:

       {{ $json["ergates.net"].map(e => `<li>${Object.values(e)[0]}</li>`).join("") }}
       ```  
   - Link AI tool: Add HTTP Request1 as AI tool node  
   - Connect Request API → Message a model  
   - AI tool connection: HTTP Request1 → Message a model (ai_tool)

6. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Respond With: "text"  
   - Response Body: Paste the provided multi-line HTML + CSS template that extracts WHOIS fields from Request API output and AI description from Message a model output. Use expressions referencing the "Request API" node and AI message content as in the original workflow.  
   - Connect Message a model → Respond to Webhook

7. **Validation and Testing**  
   - Save workflow and activate webhook  
   - Send test HTTP request to webhook URL  
   - Verify WHOIS data retrieval, AI analysis generation, and clean HTML response delivery

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed for quick domain insight generation combining WHOIS data with AI summarization and optional web search, providing a modern, visual, and concise result.                                                | Workflow purpose and design note                                                                 |
| Project by OXSR: Find more professional and business workflows at https://n8n.io/creators/oxsr11/ and source code at https://github.com/oxsr                                                                                   | Author and source                                                                                 |
| License: This project is open for personal, educational, and commercial use with modification rights, but resale or commercial distribution of copies is prohibited.                                                            | Custom open license details                                                                      |
| Discount offer: Send a screenshot of use to oriolrotllant3@gmail.com to receive 25% off on products (one use per person).                                                                                                        | Promotional offer                                                                                |
| The response HTML includes automatic dark mode support and responsive design for mobile devices, enhancing user experience across platforms.                                                                                   | UI/UX design note                                                                                |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. It adheres strictly to content policies and includes no illegal, offensive, or protected materials. All data handled is lawful and public.