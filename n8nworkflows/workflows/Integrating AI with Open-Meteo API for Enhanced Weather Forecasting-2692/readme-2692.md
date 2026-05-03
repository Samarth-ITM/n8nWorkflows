Integrating AI with Open-Meteo API for Enhanced Weather Forecasting

https://n8nworkflows.xyz/workflows/integrating-ai-with-open-meteo-api-for-enhanced-weather-forecasting-2692


# Integrating AI with Open-Meteo API for Enhanced Weather Forecasting

### 1. Workflow Overview

This workflow demonstrates how to integrate AI-driven decision-making with external APIs to provide enhanced weather forecasting. It is designed primarily as a workshop example to teach how AI agents can orchestrate multiple tools (functions) in a single conversational flow. The workflow uses an AI agent to dynamically decide the sequence of API calls: first to obtain geolocation data for a city, then to fetch the weather forecast for that location.

**Target Use Cases:**
- Educational workshops on AI tool integration and chaining
- Interactive weather forecasting assistants that respond to user chat inputs
- Travel planning tools that provide upcoming weather based on city names

**Logical Blocks:**

- **1.1 Input Reception:** Receives chat messages from users via a hosted web chat interface.
- **1.2 Chat Memory Management:** Maintains conversational context across interactions.
- **1.3 AI Processing:** Uses an AI agent (OpenAI GPT-based) to interpret user input and decide which tools to invoke.
- **1.4 Geolocation Tool:** Calls Open-Meteo’s geocoding API to convert city names into geographic coordinates.
- **1.5 Weather Forecast Tool:** Calls Open-Meteo’s weather forecast API using coordinates to retrieve upcoming weather data.
- **1.6 User Interaction and Setup Notes:** Sticky notes provide documentation, setup instructions, and API specifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures user chat messages from a hosted web chat interface, triggering the workflow to start processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: Chat Trigger (LangChain)  
    - Role: Entry point for user messages via webhook  
    - Configuration:  
      - Public webhook enabled  
      - Chat interface titled "N8N 👋" with subtitle "Weather Assistant: Example of Tools Using ChatGPT"  
      - File uploads disabled  
      - Previous session memory loaded from "memory"  
      - Initial prompt example: "Weather Forecast for the Next 7 Days in São Paulo"  
    - Input: Incoming HTTP webhook chat message  
    - Output: Chat message data forwarded to AI agent  
    - Edge cases: Webhook downtime, malformed chat messages, session memory errors  
    - Sticky Note1 nearby explains how to create hosted web chat and webhook setup.

#### 1.2 Chat Memory Management

- **Overview:**  
  Maintains a sliding window memory buffer to preserve conversational context for the AI agent.

- **Nodes Involved:**  
  - Chat Memory Buffer

- **Node Details:**  
  - **Chat Memory Buffer**  
    - Type: Memory Buffer Window (LangChain)  
    - Role: Stores recent chat messages to maintain context  
    - Configuration: Default parameters (window size unspecified)  
    - Input: Receives chat messages from trigger and AI agent nodes  
    - Output: Provides memory context to AI agent node  
    - Edge cases: Memory overflow, context loss, synchronization issues  

#### 1.3 AI Processing

- **Overview:**  
  Uses an AI agent to interpret user input, decide which API tools to call, and orchestrate the calls in the correct order.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Generic AI Tool Agent

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: Language Model Chat (OpenAI)  
    - Role: Provides GPT-based chat completions for the AI agent  
    - Configuration: Uses OpenAI API credentials (OpenAI key required)  
    - Input: Receives chat context and prompts from memory buffer and agent  
    - Output: Chat completions passed to AI agent  
    - Edge cases: API rate limits, authentication errors, network timeouts  
  - **Generic AI Tool Agent**  
    - Type: Agent (LangChain)  
    - Role: Acts as the orchestrator AI that decides which tools to invoke based on user input  
    - Configuration: Default options; linked to OpenAI Chat Model for language understanding  
    - Input: Chat messages from trigger and memory buffer; tool outputs  
    - Output: Calls to tools (HTTP request nodes) and final responses  
    - Edge cases: Tool invocation failures, incorrect tool selection, unexpected API responses  

#### 1.4 Geolocation Tool

- **Overview:**  
  Queries Open-Meteo’s geocoding API to retrieve latitude and longitude for a given city name.

- **Nodes Involved:**  
  - A tool for inputting the city and obtaining geolocation

- **Node Details:**  
  - **A tool for inputting the city and obtaining geolocation**  
    - Type: HTTP Request Tool (LangChain)  
    - Role: Calls Open-Meteo geocoding API to get city coordinates  
    - Configuration:  
      - URL: https://geocoding-api.open-meteo.com/v1/search  
      - Query parameters:  
        - name (city name, provided by AI model)  
        - count=1 (limit to first result)  
        - format=json  
      - Tool description: "Input the City and get geolocation, geocode or coordinates from Requested City"  
      - Placeholders: name (string, city requested by user)  
    - Input: City name from AI agent  
    - Output: JSON with geolocation data (latitude, longitude)  
    - Edge cases: City not found, API downtime, malformed responses  
    - Sticky Note5 explains API usage and parameters.

#### 1.5 Weather Forecast Tool

- **Overview:**  
  Uses the coordinates from the geolocation tool to fetch weather forecast data for the upcoming days.

- **Nodes Involved:**  
  - A tool to get the weather forecast based on geolocation

- **Node Details:**  
  - **A tool to get the weather forecast based on geolocation**  
    - Type: HTTP Request Tool (LangChain)  
    - Role: Calls Open-Meteo forecast API to retrieve weather data  
    - Configuration:  
      - URL: https://api.open-meteo.com/v1/forecast  
      - Query parameters:  
        - latitude (from geolocation tool)  
        - longitude (from geolocation tool)  
        - daily=temperature_2m_max,precipitation_sum  
        - timezone=GMT  
        - forecast_days (number of days ahead, provided by AI)  
      - Tool description: "To get forecast of next [forecast_days] input the geolocation of a City"  
      - Placeholders: latitude (number), longitude (number), forecast_days (number)  
    - Input: Coordinates and forecast days from AI agent  
    - Output: JSON weather forecast data  
    - Edge cases: Invalid coordinates, API rate limits, incomplete data  
    - Sticky Note6 explains API usage and parameters.

#### 1.6 User Interaction and Setup Notes

- **Overview:**  
  Provides documentation and setup instructions via sticky notes for users and maintainers.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note4  
  - Sticky Note5  
  - Sticky Note6

- **Node Details:**  
  - Sticky notes contain detailed explanations of the workflow purpose, API specifications, setup instructions, and usage tips.  
  - They do not affect workflow execution but are essential for understanding and maintaining the workflow.  
  - Sticky Note includes author credit and LinkedIn profile link.

---

### 3. Summary Table

| Node Name                                         | Node Type                          | Functional Role                         | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                      |
|--------------------------------------------------|----------------------------------|---------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received                        | Chat Trigger (LangChain)          | Entry point for user chat messages    | -                             | Generic AI Tool Agent          | Sticky Note1: "Create an Hosted Web Chat and setup the trigger! Example: https://website/webhook/4a4..../chat" |
| Chat Memory Buffer                                | Memory Buffer Window (LangChain) | Maintains conversational context     | When chat message received     | When chat message received, Generic AI Tool Agent | Sticky Note2: "Setup OpenAI Key"                                                                   |
| OpenAI Chat Model                                | Language Model Chat (OpenAI)      | Provides GPT chat completions          | Chat Memory Buffer             | Generic AI Tool Agent          |                                                                                                |
| Generic AI Tool Agent                             | Agent (LangChain)                 | Orchestrates tool calls based on input| When chat message received, Chat Memory Buffer, OpenAI Chat Model, Geolocation Tool, Weather Forecast Tool | -                             |                                                                                                |
| A tool for inputting the city and obtaining geolocation | HTTP Request Tool (LangChain)     | Calls Open-Meteo geocoding API         | Generic AI Tool Agent          | Generic AI Tool Agent          | Sticky Note5: "Open Meteo SPEC - City Geolocation: API URL and parameters explained"             |
| A tool to get the weather forecast based on geolocation | HTTP Request Tool (LangChain)     | Calls Open-Meteo weather forecast API | Generic AI Tool Agent          | Generic AI Tool Agent          | Sticky Note6: "Open Meteo SPEC - Weather Forecast: API URL and parameters explained"             |
| Sticky Note                                      | Sticky Note                      | Documentation and credits             | -                             | -                             | Workflow overview, use case, author credit, LinkedIn link                                       |
| Sticky Note1                                     | Sticky Note                      | Documentation                        | -                             | -                             | Hosted web chat creation and webhook setup                                                     |
| Sticky Note2                                     | Sticky Note                      | Documentation                        | -                             | -                             | OpenAI Key setup                                                                               |
| Sticky Note4                                     | Sticky Note                      | Documentation                        | -                             | -                             | Chat button testing instructions                                                              |
| Sticky Note5                                     | Sticky Note                      | Documentation                        | -                             | -                             | Open Meteo geolocation API specification                                                      |
| Sticky Note6                                     | Sticky Note                      | Documentation                        | -                             | -                             | Open Meteo weather forecast API specification                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add node: **When chat message received** (LangChain Chat Trigger)  
   - Configure:  
     - Public webhook enabled  
     - Title: "N8N 👋"  
     - Subtitle: "Weather Assistant: Example of Tools Using ChatGPT"  
     - Disable file uploads  
     - Load previous session from memory  
     - Initial message prompt: "Type like this: Weather Forecast for the Next 7 Days in São Paulo"  
   - Position: Top-left area (for clarity)

2. **Add Chat Memory Buffer Node**  
   - Add node: **Chat Memory Buffer** (LangChain Memory Buffer Window)  
   - Use default parameters  
   - Connect input from "When chat message received" node (ai_memory output)  
   - Connect output to "When chat message received" and "Generic AI Tool Agent" nodes (ai_memory input)

3. **Add OpenAI Chat Model Node**  
   - Add node: **OpenAI Chat Model** (LangChain LM Chat OpenAI)  
   - Configure credentials: select or create OpenAI API key credential  
   - Use default options  
   - Connect input from "Chat Memory Buffer" node (ai_languageModel input)  
   - Connect output to "Generic AI Tool Agent" node (ai_languageModel output)

4. **Add Generic AI Tool Agent Node**  
   - Add node: **Generic AI Tool Agent** (LangChain Agent)  
   - Use default options  
   - Connect input from:  
     - "When chat message received" (ai_memory)  
     - "Chat Memory Buffer" (ai_memory)  
     - "OpenAI Chat Model" (ai_languageModel)  
     - "A tool for inputting the city and obtaining geolocation" (ai_tool)  
     - "A tool to get the weather forecast based on geolocation" (ai_tool)  
   - Connect output to:  
     - "A tool for inputting the city and obtaining geolocation" (ai_tool)  
     - "A tool to get the weather forecast based on geolocation" (ai_tool)

5. **Add Geolocation HTTP Request Tool Node**  
   - Add node: **A tool for inputting the city and obtaining geolocation** (LangChain Tool HTTP Request)  
   - Configure:  
     - URL: `https://geocoding-api.open-meteo.com/v1/search`  
     - Send query parameters:  
       - name (string, placeholder, to be filled by AI)  
       - count = 1 (fixed)  
       - format = json (fixed)  
     - Tool description: "Input the City and get geolocation, geocode or coordinates from Requested City"  
     - Placeholder definitions: name (string, "Requested City")  
   - Connect input from "Generic AI Tool Agent" (ai_tool)  
   - Connect output to "Generic AI Tool Agent" (ai_tool)

6. **Add Weather Forecast HTTP Request Tool Node**  
   - Add node: **A tool to get the weather forecast based on geolocation** (LangChain Tool HTTP Request)  
   - Configure:  
     - URL: `https://api.open-meteo.com/v1/forecast`  
     - Send query parameters:  
       - latitude (number, placeholder)  
       - longitude (number, placeholder)  
       - daily = temperature_2m_max,precipitation_sum (fixed)  
       - timezone = GMT (fixed)  
       - forecast_days (number, placeholder)  
     - Tool description: "To get forecast of next [forecast_days] input the geolocation of a City"  
     - Placeholder definitions: latitude (number), longitude (number), forecast_days (number)  
   - Connect input from "Generic AI Tool Agent" (ai_tool)  
   - Connect output to "Generic AI Tool Agent" (ai_tool)

7. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add sticky notes with the following content for maintainers and users:  
     - Workflow overview and use case (author credit and LinkedIn link)  
     - Hosted web chat creation and webhook setup instructions  
     - OpenAI API key setup reminder  
     - Chat button testing instructions  
     - Open-Meteo geolocation API specification  
     - Open-Meteo weather forecast API specification

8. **Activate the Workflow**  
   - Insert valid OpenAI API key credentials  
   - Save and activate the workflow  
   - Test by sending chat messages like: "Weather Forecast for the Next 7 Days in São Paulo" via the webhook URL or hosted web chat

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow created by Davi Saranszky Mesquita                                                                   | https://www.linkedin.com/in/mesquitadavi/                       |
| Workshop focus: teaching AI tool chaining with LangChain and OpenAI models                                    | -                                                               |
| Hosted web chat example URL pattern: `https://website/webhook/4a4..../chat`                                   | Sticky Note1                                                    |
| OpenAI API key required for AI language model nodes                                                           | Sticky Note2                                                    |
| Open-Meteo Geocoding API: https://geocoding-api.open-meteo.com/v1/search                                      | Sticky Note5                                                    |
| Open-Meteo Weather Forecast API: https://api.open-meteo.com/v1/forecast                                       | Sticky Note6                                                    |

---

This document provides a complete, structured reference for understanding, reproducing, and maintaining the "Integrating AI with Open-Meteo API for Enhanced Weather Forecasting" workflow in n8n. It covers all nodes, their roles, configurations, and interconnections, enabling both human developers and AI agents to work effectively with this workflow.