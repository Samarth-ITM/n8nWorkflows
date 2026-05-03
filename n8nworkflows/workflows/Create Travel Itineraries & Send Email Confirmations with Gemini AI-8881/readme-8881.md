Create Travel Itineraries & Send Email Confirmations with Gemini AI

https://n8nworkflows.xyz/workflows/create-travel-itineraries---send-email-confirmations-with-gemini-ai-8881


# Create Travel Itineraries & Send Email Confirmations with Gemini AI

---
### 1. Workflow Overview

This workflow automates the process of creating personalized travel itineraries and sending email confirmations using Gemini AI models integrated with external data sources. It is designed for travel agents or services that handle user travel requests, extract key travel parameters, curate detailed travel plans including flights, activities, and accommodations, and deliver polished email confirmations to clients.

**Target Use Cases:**  
- Automated travel itinerary generation from unstructured user chat requests  
- Intelligent extraction and inference of travel details from natural language input  
- Integration with external APIs for live flight, activity, and accommodation data  
- Professional email formatting and delivery of travel plans  

**Logical Blocks:**

- **1.1 Input Reception & Extraction**  
  Triggered by incoming chat messages, this block uses AI to extract and structure user travel requests precisely.

- **1.2 Travel Planning & Data Aggregation**  
  Central planning agent that orchestrates API calls for flights, activities, and accommodations, compiles results into a comprehensive travel plan.

- **1.3 Email Generation & Sending**  
  Converts the structured travel plan into a visually appealing HTML email and sends it to the user.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Extraction

**Overview:**  
This block triggers on a new chat message, then uses a Google Gemini AI model combined with a structured output parser to analyze and extract detailed travel parameters from the user's natural language input. It ensures that all relevant data points and inferred assumptions are captured in a standardized JSON format.

**Nodes Involved:**  
- When chat message received  
- Google Gemini Chat Model  
- Extract User Request (LangChain Agent)  
- Structured Output Parser  

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger Node (LangChain)  
  - Role: Entry trigger listening for new chat messages from users  
  - Config: Basic webhook ID to receive messages, no extra options set  
  - Input: Incoming chat message  
  - Output: Passes raw user input to next node  
  - Failures: Possible webhook connectivity issues or malformed messages  

- **Google Gemini Chat Model**  
  - Type: Language Model Node (Google Gemini / PaLM API)  
  - Role: Processes user messages using Gemini AI for natural language understanding  
  - Config: Uses authenticated Google Palm API credentials  
  - Input: Raw chat message  
  - Output: AI-processed text for agent node  
  - Failures: API quota limits, auth errors, network timeouts  

- **Extract User Request**  
  - Type: LangChain Agent Node with system prompt and output parser  
  - Role: Expert information extraction from chat input, applying rules and inference  
  - Config: Detailed system prompt specifying travel domain, data schema, inference constraints, and output format  
  - Input: AI-processed message from Gemini model  
  - Output: Extracted structured travel parameters in JSON form  
  - Expressions: Uses dynamic current system time for context  
  - Failures: Parsing errors, incomplete data extraction, expression errors if data missing  
  - Version: LangChain v2 with output parser enabled  

- **Structured Output Parser**  
  - Type: LangChain Output Parser (Structured JSON)  
  - Role: Validates and formats the extracted data according to a strict JSON schema for travel parameters  
  - Config: Manual schema defining properties like origin, destination, dates, travelers, preferences, etc.  
  - Input: Raw output from Extract User Request agent  
  - Output: Clean, validated JSON object for downstream use  
  - Failures: Schema validation failures if output malformed  

---

#### 2.2 Travel Planning & Data Aggregation

**Overview:**  
This core block receives the structured travel request and uses a Google Gemini-powered planner agent to coordinate searches for flights, activities, and accommodations via external API calls. It collects and combines all data into a comprehensive travel plan JSON.

**Nodes Involved:**  
- Planner Agent (LangChain Agent)  
- Google Gemini Chat Model1  
- Flight booking (HTTP Request Tool)  
- Activities (HTTP Request Tool)  
- Accommodations (HTTP Request Tool)  
- Accommodation Details (HTTP Request Tool)  
- Structured Output Parser2  

**Node Details:**

- **Planner Agent**  
  - Type: LangChain Agent Node with multi-tool integration  
  - Role: Travel planning expert orchestrating API calls based on extracted parameters  
  - Config: System prompt with explicit instructions to use three planning tools (flights, activities, accommodations), output JSON structure, constraints on results count, and detail completeness  
  - Input: Structured travel request JSON  
  - Output: Combined travel plan JSON including flights, activities, accommodations, and request summary  
  - AI Model: Uses Google Gemini Chat Model1 for processing  
  - Failures: Tool API call failures, inconsistent data from tools, parsing errors  
  - Notes: Uses property_token values from accommodation results to fetch detailed info  

- **Flight booking**  
  - Type: HTTP Request Tool  
  - Role: Calls SerpAPI Google Flights endpoint to retrieve flight options  
  - Config: Uses HTTP query authentication, parameters include departure and arrival airport IDs extracted dynamically  
  - Input: Parameters from Planner Agent via $fromAI expressions  
  - Output: Flight search JSON data  
  - Failures: API limits, invalid query parameters, network issues  

- **Activities**  
  - Type: HTTP Request Tool  
  - Role: Calls SerpAPI Google Activities endpoint for destination activities  
  - Config: HTTP query authentication with dynamic query parameter for activity search  
  - Input: Destination and preferences from Planner Agent  
  - Output: JSON list of activity recommendations  
  - Failures: API errors, no results found  

- **Accommodations**  
  - Type: HTTP Request Tool  
  - Role: Calls SerpAPI Google Hotels endpoint for accommodation options  
  - Config: Authenticated query with parameters for check-in/out dates, adults count, and sorting  
  - Input: Parameters dynamically injected from Planner Agent  
  - Output: List of hotel search results  
  - Failures: Parameter mismatches, API response errors  

- **Accommodation Details**  
  - Type: HTTP Request Tool  
  - Role: Fetches detailed info for each accommodation using property_token from Accommodations node  
  - Config: Query parameter includes property_token and search query  
  - Input: Tokens from Accommodations output, passed from Planner Agent  
  - Output: Detailed property data including website URLs  
  - Failures: Missing or invalid tokens, API errors  

- **Structured Output Parser2**  
  - Type: LangChain Output Parser (Structured JSON)  
  - Role: Validates and formats the aggregated travel plan data according to a comprehensive schema covering flights, activities, accommodations, weather, and request summary  
  - Config: Manual JSON schema with required fields and nested objects for each travel category  
  - Input: Raw output from Planner Agent  
  - Output: Clean, validated travel plan JSON for email generation  
  - Failures: Schema validation errors  

---

#### 2.3 Email Generation & Sending

**Overview:**  
This block formats the finalized travel plan into a professional, responsive HTML email using Gemini AI, then sends the email via Gmail.

**Nodes Involved:**  
- Email Agent (LangChain Agent)  
- Google Gemini Chat Model2  
- Structured Output Parser1  
- Send a message (Gmail)  

**Node Details:**

- **Email Agent**  
  - Type: LangChain Agent Node with email design system prompt  
  - Role: Expert email marketing agent transforming travel plan JSON into a visually appealing, mobile-responsive HTML email with subject line and summary  
  - Config: Detailed system prompt specifying email design best practices, output JSON structure with subject, content, and summary, constraints on length and compatibility  
  - Input: Validated travel plan JSON from Structured Output Parser2  
  - Output: JSON containing email subject, HTML content, and summary  
  - AI Model: Uses Google Gemini Chat Model2  
  - Failures: Rendering issues, content formatting errors  

- **Structured Output Parser1**  
  - Type: LangChain Output Parser (Structured JSON)  
  - Role: Validates email output JSON schema ensuring subject, content, and summary are present and correctly formatted  
  - Config: Manual JSON schema with max length for subject and required HTML content  
  - Input: Raw output from Email Agent  
  - Output: Clean email data for sending  
  - Failures: Schema validation errors  

- **Send a message**  
  - Type: Gmail Node  
  - Role: Sends the generated email to the user email address  
  - Config: Uses OAuth2 credentials, dynamically sets recipient, subject, and message body from previous node outputs  
  - Input: Parsed email JSON content  
  - Output: Email sent confirmation  
  - Failures: Auth errors, quota limits, invalid email addresses  

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                               | Input Node(s)                     | Output Node(s)         | Sticky Note                                                                                       |
|-----------------------|----------------------------------|----------------------------------------------|----------------------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry webhook trigger on new user message    | -                                | Extract User Request    | ## Extract User Request When chat message received: This is the trigger that starts the workflow whenever a new message is received from a user. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Processes raw chat message with Gemini AI    | When chat message received        | Extract User Request    |                                                                                                 |
| Extract User Request   | @n8n/n8n-nodes-langchain.agent  | Extracts structured travel parameters        | Google Gemini Chat Model          | Structured Output Parser |                                                                                                 |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Validates and formats extracted travel data  | Extract User Request              | Planner Agent          |                                                                                                 |
| Planner Agent          | @n8n/n8n-nodes-langchain.agent  | Orchestrates planning, integrates data tools | Structured Output Parser          | Email Agent             | ## Planner Agent Planner Agent: This central node acts as a multi-step planner. It takes the parsed request from the first block and uses a Google Gemini Chat Model to orchestrate the travel planning process. |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI model used by Planner Agent                | Planner Agent                    | Planner Agent (internal) |                                                                                                 |
| Flight booking         | n8n-nodes-base.httpRequestTool  | Fetches flight options from external API     | Planner Agent                    | Planner Agent (internal) |                                                                                                 |
| Activities             | n8n-nodes-base.httpRequestTool  | Fetches activity options from external API   | Planner Agent                    | Planner Agent (internal) |                                                                                                 |
| Accommodations         | n8n-nodes-base.httpRequestTool  | Fetches accommodation options from external API | Planner Agent                | Planner Agent (internal) |                                                                                                 |
| Accommodation Details  | n8n-nodes-base.httpRequestTool  | Fetches detailed accommodation info          | Planner Agent                    | Planner Agent (internal) |                                                                                                 |
| Structured Output Parser2 | @n8n/n8n-nodes-langchain.outputParserStructured | Validates complete travel plan JSON           | Planner Agent                    | Email Agent             |                                                                                                 |
| Email Agent            | @n8n/n8n-nodes-langchain.agent  | Converts travel plan JSON into HTML email     | Structured Output Parser2        | Structured Output Parser1 | ## Email Agent Email Agent: This node is responsible for generating the final email. It takes the comprehensive travel plan from the Planner Agent and uses a second Google Gemini Chat Model to format it into a coherent, well-written email. |
| Google Gemini Chat Model2 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI model used by Email Agent                   | Email Agent                     | Email Agent (internal)  |                                                                                                 |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Validates final email JSON output              | Email Agent                     | Send a message          |                                                                                                 |
| Send a message         | n8n-nodes-base.gmail            | Sends the generated email to user             | Structured Output Parser1         | -                       |                                                                                                 |
| Sticky Note            | n8n-nodes-base.stickyNote       | Comment on Extract User Request block          | -                                | -                       | ## Extract User Request When chat message received: This is the trigger that starts the workflow whenever a new message is received from a user. Extract User Request: This node uses a Google Gemini Chat Model and a Structured Output Parser to analyze the user's message. Its purpose is to identify the user's intent and extract key information like the destination, travel dates, or specific requests. |
| Sticky Note1           | n8n-nodes-base.stickyNote       | Comment on Planner Agent and sub-nodes         | -                                | -                       | ## Planner Agent Planner Agent: This central node acts as a multi-step planner. It takes the parsed request from the first block and uses a Google Gemini Chat Model to orchestrate the travel planning process. Sub-nodes (Activities, Flight booking, Accommodation Details, Accommodations): The Planner Agent calls these sub-nodes, which are configured to make specific API calls (indicated by GET: https://usersapi.co/GET...) to gather real-time data on activities, flights, and accommodations. Structured Output Parser 2: This parser node likely combines the data from all the sub-nodes into a single, structured output that can be easily used by the next part of the workflow. |
| Sticky Note2           | n8n-nodes-base.stickyNote       | Comment on Email Agent block                     | -                                | -                       | ## Email Agent Email Agent: This node is responsible for generating the final email. It takes the comprehensive travel plan from the Planner Agent and uses a second Google Gemini Chat Model to format it into a coherent, well-written email. Structured Output Parser 1: This parser ensures the final email content is in a clean format, ready to be sent. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Chat Trigger** node named "When chat message received"  
   - Configure webhook to receive chat messages from users (no special options needed).  

2. **Create Language Model Node for Extraction:**  
   - Add **Google Gemini Chat Model** node (API credentials for Google PaLM required)  
   - Connect "When chat message received" → "Google Gemini Chat Model".  

3. **Create Extraction Agent:**  
   - Add **LangChain Agent** node named "Extract User Request"  
   - Set system prompt with detailed travel extraction instructions and JSON output schema (as specified in overview)  
   - Enable Output Parser with manual JSON schema matching travel data structure  
   - Connect "Google Gemini Chat Model" → "Extract User Request".  

4. **Add Structured Output Parser:**  
   - Add **LangChain Output Parser Structured** node named "Structured Output Parser"  
   - Paste the travel parameter JSON schema as manual input  
   - Connect "Extract User Request" → "Structured Output Parser".  

5. **Create Planner Agent:**  
   - Add **LangChain Agent** node named "Planner Agent"  
   - Provide detailed system prompt describing travel planning tasks and use of specific API tools (flights, activities, accommodations)  
   - Define the JSON output structure for the combined travel plan (including flights, activities, accommodations, weather, request info)  
   - Link a **Google Gemini Chat Model1** node as AI model for this agent with proper API credentials  
   - Connect "Structured Output Parser" → "Planner Agent".  

6. **Add API HTTP Request Nodes for Planning Tools:**  
   - Add "Flight booking" node: HTTP Request Tool to SerpAPI Google Flights endpoint, set authentication and query parameters dynamically from Planner Agent outputs  
   - Add "Activities" node: HTTP Request Tool to SerpAPI for activities search, authenticated with query parameters from Planner Agent  
   - Add "Accommodations" node: HTTP Request Tool to SerpAPI Google Hotels endpoint with check-in/out, adults, and sorting parameters from Planner Agent  
   - Add "Accommodation Details" node: HTTP Request Tool to SerpAPI with property_token parameter, to fetch detailed accommodation info  
   - Configure all nodes with the same HTTP Query Auth credentials  
   - Connect all these nodes as AI Tool integrations called by "Planner Agent" (via ai_tool connection).  

7. **Add Structured Output Parser for Planning Results:**  
   - Add **LangChain Output Parser Structured** node named "Structured Output Parser2"  
   - Use manual JSON schema defining flights, activities, accommodations, weather, and request_info  
   - Connect "Planner Agent" → "Structured Output Parser2".  

8. **Create Email Agent:**  
   - Add **LangChain Agent** node named "Email Agent"  
   - Provide system prompt for email marketing expert to generate responsive HTML email from travel plan JSON  
   - Define output JSON schema with subject, content (HTML), and summary  
   - Link **Google Gemini Chat Model2** node as AI model with credentials  
   - Connect "Structured Output Parser2" → "Email Agent".  

9. **Add Structured Output Parser for Email:**  
   - Add **LangChain Output Parser Structured** node named "Structured Output Parser1"  
   - Use manual schema requiring subject and content fields for email  
   - Connect "Email Agent" → "Structured Output Parser1".  

10. **Add Email Sending Node:**  
    - Add **Gmail** node named "Send a message"  
    - Configure OAuth2 credentials for Gmail account  
    - Set recipient email address (e.g., dynamically or static as per use case)  
    - Map subject and message body from "Structured Output Parser1" output fields  
    - Connect "Structured Output Parser1" → "Send a message".  

11. **Finalize Connections:**  
    - Ensure data flows from trigger → extraction → planning → email generation → sending  
    - Test API credentials and webhook connectivity  
    - Validate correct parameter mapping and JSON schema compliance at each step  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow uses Google Gemini (PaLM) AI models via n8n’s LangChain integration for advanced natural language processing and generation in travel domain tasks.                                                                                                       | Google Gemini (PaLM) API                      |
| External data is sourced via SerpAPI endpoints for flights, activities, and hotel searches, requiring valid HTTP Query Auth credentials and API keys.                                                                                                                  | https://serpapi.com/                          |
| Email generation follows responsive HTML email best practices with inline CSS and table layouts for broad email client compatibility, prioritizing user experience on desktop and mobile devices.                                                                       | Email design standards                        |
| The Planner Agent is the core orchestrator, leveraging multi-tool calls and combining data into a unified travel plan JSON, with strict schema validation to ensure robustness.                                                                                          | Workflow logical core                         |
| Current system time (`{{ $now }}`) is injected dynamically into AI prompts to provide time context for date inference and planning.                                                                                                                                      | Dynamic prompt variables                      |

---

**Disclaimer:** The text provided is derived exclusively from an automated n8n workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.