Plan Travel Itineraries with Gemini AI, Live Amadeus Flights, and Airbnb Stays

https://n8nworkflows.xyz/workflows/plan-travel-itineraries-with-gemini-ai--live-amadeus-flights--and-airbnb-stays-6652


# Plan Travel Itineraries with Gemini AI, Live Amadeus Flights, and Airbnb Stays

### 1. Workflow Overview

This workflow automates the planning of travel itineraries by integrating AI-driven itinerary generation with live flight data from Amadeus and accommodation options from Airbnb. It is designed for travel agencies, itinerary planners, or travel-focused applications seeking to generate personalized travel plans dynamically based on user input.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Captures user requests via a webhook.
- **1.2 AI Agent Processing:** Uses an AI agent (LangChain-based) with Google Vertex Chat and MCP tools to interpret user input and generate initial itinerary concepts.
- **1.3 Data Extraction and Cleaning:** Extracts and cleans relevant flight and accommodation parameters from AI output.
- **1.4 Live Flight Search:** Queries live flight data with pricing using the Amadeus API.
- **1.5 Integration with Airbnb Listings:** Merges flight data with Airbnb stay options to create a comprehensive itinerary.
- **1.6 Response Preparation:** Formats and sends the final itinerary back to the user via the webhook response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures incoming user requests containing travel preferences or queries through an HTTP webhook endpoint.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook Trigger  
    - Role: Entry point for user input; awaits HTTP POST or GET requests with travel itinerary queries.  
    - Configuration: Default webhook, no custom parameters specified. Uses webhook ID to identify the endpoint.  
    - Inputs: External HTTP request  
    - Outputs: Passes incoming data to the AI Agent node.  
    - Edge cases: Missing or malformed requests, timeout if no request received.

---

#### 2.2 AI Agent Processing

- **Overview:**  
  Processes incoming user input using an AI agent that orchestrates language models and client tools to interpret the travel request and generate a structured itinerary plan.

- **Nodes Involved:**  
  - AI Agent  
  - Google Vertex Chat Model  
  - MCP Client List Tool  
  - MCP Execute Tool

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Core AI orchestrator that manages the conversation and planning logic. Accepts input from the webhook and integrates multiple AI tools.  
    - Configuration: Linked to Google Vertex Chat Model for language understanding; connected to MCP Client List Tool and MCP Execute Tool as AI tools for specialized processing.  
    - Inputs: Webhook outputs; AI tools and language model outputs.  
    - Outputs: Generates AI-processed itinerary data, forwarded to data cleaning node.  
    - Edge cases: AI processing errors, tool unavailability, API rate limits.

  - **Google Vertex Chat Model**  
    - Type: Language Model Node (Google Vertex AI)  
    - Role: Provides natural language processing and understanding capabilities for the AI Agent.  
    - Configuration: Default parameters; integrated via ai_languageModel connection to AI Agent.  
    - Inputs: AI Agent's prompt or query.  
    - Outputs: Language model responses to AI Agent.  
    - Edge cases: Authentication failures, quota limits, model downtime.

  - **MCP Client List Tool**  
    - Type: Specialized AI Tool  
    - Role: Provides a list or database of client preferences or parameters that assist AI Agent in itinerary planning.  
    - Configuration: No specific parameters shown; integrated as AI tool.  
    - Inputs: AI Agent requests.  
    - Outputs: Data or preferences to AI Agent.  
    - Edge cases: Data fetching errors, unavailability.

  - **MCP Execute Tool**  
    - Type: Specialized AI Tool  
    - Role: Executes specific client-related commands or queries to support itinerary generation.  
    - Configuration: No specific parameters; connected as AI tool.  
    - Inputs: AI Agent requests.  
    - Outputs: Execution results to AI Agent.  
    - Edge cases: Execution errors, command failures.

---

#### 2.3 Data Extraction and Cleaning

- **Overview:**  
  Extracts structured flight and accommodation information from the AI Agent’s output, ensuring data is clean and formatted for subsequent API queries.

- **Nodes Involved:**  
  - Grabbing Clean Data

- **Node Details:**

  - **Grabbing Clean Data**  
    - Type: Code Node (JavaScript)  
    - Role: Parses AI Agent output to extract clear parameters such as dates, locations, traveler count, and preferences.  
    - Configuration: Custom JavaScript code that processes input JSON or text, normalizes data formats.  
    - Inputs: Output from AI Agent.  
    - Outputs: Cleaned data forwarded to live flight search.  
    - Edge cases: Parsing errors, unexpected data formats, missing fields.

---

#### 2.4 Live Flight Search

- **Overview:**  
  Performs real-time flight searches using the Amadeus API, retrieving available flights and fare information based on cleaned input data.

- **Nodes Involved:**  
  - Flight Search with fare

- **Node Details:**

  - **Flight Search with fare**  
    - Type: HTTP Request Node  
    - Role: Queries Amadeus flight API for live availability and pricing.  
    - Configuration: HTTP method and endpoint configured to Amadeus flight search; parameters set dynamically from cleaned data; authentication credentials required (Amadeus API key/secret).  
    - Inputs: Cleaned flight parameters.  
    - Outputs: Flight data JSON to next node.  
    - Edge cases: API authentication failures, network timeouts, empty flight results.

---

#### 2.5 Integration with Airbnb Listings

- **Overview:**  
  Merges flight data with Airbnb accommodation listings to produce a comprehensive travel itinerary including both flights and stays.

- **Nodes Involved:**  
  - Flight Data + Airbnb Listings

- **Node Details:**

  - **Flight Data + Airbnb Listings**  
    - Type: Code Node (JavaScript)  
    - Role: Combines flight search results with Airbnb stay options; likely calls Airbnb APIs or uses stored data to find relevant listings matching travel dates and destination.  
    - Configuration: Custom JavaScript code merging datasets, applying filters, formatting combined itinerary.  
    - Inputs: Flight search results.  
    - Outputs: Enriched itinerary data to final formatting node.  
    - Edge cases: Missing Airbnb data, API errors, mismatched dates or locations.

---

#### 2.6 Response Preparation

- **Overview:**  
  Formats the combined itinerary data into a suitable response structure and sends it back to the requester through the webhook response node.

- **Nodes Involved:**  
  - Edit Fields  
  - Respond to Webhook

- **Node Details:**

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Adjusts, formats, or filters the final response data fields before sending.  
    - Configuration: Likely sets key-value pairs, removes unnecessary fields, or renames response attributes.  
    - Inputs: Combined itinerary data.  
    - Outputs: Prepared data for webhook response.  
    - Edge cases: Misconfiguration leading to missing fields.

  - **Respond to Webhook**  
    - Type: Respond to Webhook Node  
    - Role: Sends HTTP response back to the original webhook caller with the final itinerary data.  
    - Configuration: Default setup, sends data received from Edit Fields node.  
    - Inputs: Formatted itinerary data.  
    - Outputs: HTTP response to user.  
    - Edge cases: Network issues, response timeouts.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                  | Input Node(s)              | Output Node(s)                 | Sticky Note                                       |
|-------------------------|----------------------------------|-------------------------------------------------|----------------------------|-------------------------------|--------------------------------------------------|
| Webhook                 | HTTP Webhook Trigger             | Receive user travel itinerary requests          | External HTTP request       | AI Agent                      |                                                  |
| AI Agent                | LangChain Agent Node             | AI orchestration and itinerary generation        | Webhook                    | Grabbing Clean Data            |                                                  |
| Google Vertex Chat Model| Language Model (Google Vertex AI) | NLP for AI Agent                                 | AI Agent                   | AI Agent                      |                                                  |
| MCP Client List Tool    | Specialized AI Tool              | Provides client-related data                      | AI Agent                   | AI Agent                      |                                                  |
| MCP Execute Tool        | Specialized AI Tool              | Executes client commands                          | AI Agent                   | AI Agent                      |                                                  |
| Grabbing Clean Data     | Code Node                       | Parse and clean AI output                         | AI Agent                   | Flight Search with fare        |                                                  |
| Flight Search with fare | HTTP Request                    | Query live flight data and fares from Amadeus API | Grabbing Clean Data         | Flight Data + Airbnb Listings  |                                                  |
| Flight Data + Airbnb Listings | Code Node                   | Combine flight data with Airbnb listings         | Flight Search with fare     | Edit Fields                   |                                                  |
| Edit Fields             | Set Node                       | Format final response                             | Flight Data + Airbnb Listings | Respond to Webhook             |                                                  |
| Respond to Webhook      | Respond to Webhook Node          | Return itinerary response to user                | Edit Fields                | External HTTP response         |                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Configure to receive HTTP POST requests (default).  
   - No additional parameters needed.  
   - Position as the entry point.

2. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Connect input from Webhook node.  
   - Configure to use:  
     - Google Vertex Chat Model (set as AI languageModel).  
     - MCP Client List Tool and MCP Execute Tool (set as AI tools).  
   - No additional parameters shown; link credentials for Google Vertex AI and MCP if needed.

3. **Create Google Vertex Chat Model Node**  
   - Type: Language Model (Google Vertex AI)  
   - No special configuration beyond authentication credentials.  
   - Connect output to AI Agent as language model input.

4. **Create MCP Client List Tool Node**  
   - Type: MCP Client Tool  
   - No parameters shown.  
   - Connect output as AI tool input to AI Agent.

5. **Create MCP Execute Tool Node**  
   - Type: MCP Client Tool  
   - No parameters shown.  
   - Connect output as AI tool input to AI Agent.

6. **Create Grabbing Clean Data Node**  
   - Type: Code Node (JavaScript)  
   - Connect input from AI Agent output.  
   - Implement code to parse AI output and extract structured travel parameters (dates, locations, travelers).  
   - Ensure error handling for malformed input.

7. **Create Flight Search with fare Node**  
   - Type: HTTP Request  
   - Configure to call Amadeus API flight search endpoint.  
   - Use parameters from cleaned data node dynamically (e.g., origin, destination, dates).  
   - Configure authentication with Amadeus API credentials.  
   - Connect input from Grabbing Clean Data node.

8. **Create Flight Data + Airbnb Listings Node**  
   - Type: Code Node (JavaScript)  
   - Connect input from Flight Search with fare output.  
   - Implement code to fetch or merge Airbnb listings matching destination and travel dates.  
   - Combine flight and accommodation data into a single JSON object.

9. **Create Edit Fields Node**  
   - Type: Set Node  
   - Connect input from Flight Data + Airbnb Listings node.  
   - Configure to format or filter response fields (e.g., rename keys, remove unnecessary data).

10. **Create Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Connect input from Edit Fields node.  
    - No additional parameters; ensures response is sent back to webhook caller.

11. **Connect all nodes as per the flow:**  
    Webhook → AI Agent → Grabbing Clean Data → Flight Search with fare → Flight Data + Airbnb Listings → Edit Fields → Respond to Webhook  
    Additionally, Google Vertex Chat Model, MCP Client List Tool, and MCP Execute Tool connect as AI tools/language models into AI Agent.

12. **Set up Credentials:**  
    - Google Vertex AI credentials for Google Vertex Chat Model.  
    - Amadeus API credentials for Flight Search with fare HTTP Request node.  
    - MCP credentials if required for MCP Client List Tool and MCP Execute Tool.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                       |
|-------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| This workflow relies on the integration of Amadeus flight data and Airbnb listings to provide comprehensive travel plans. | Overview of Amadeus API: https://developers.amadeus.com/ |
| Google Vertex AI is used to leverage advanced language models for itinerary understanding and generation.         | Google Vertex AI Docs: https://cloud.google.com/vertex-ai |
| MCP Tools are custom or third-party tools integrated as AI tools; ensure proper credentials and service availability. | Check MCP Tool documentation for setup and usage.    |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It fully complies with applicable content policies and contains no illegal or protected elements. All data handled is legal and publicly accessible.