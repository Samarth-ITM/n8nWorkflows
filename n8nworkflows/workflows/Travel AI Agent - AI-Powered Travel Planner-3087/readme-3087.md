Travel AI Agent - AI-Powered Travel Planner

https://n8nworkflows.xyz/workflows/travel-ai-agent---ai-powered-travel-planner-3087


# Travel AI Agent - AI-Powered Travel Planner

### 1. Workflow Overview

This workflow, titled **Business Travel AI Agent - AI-Powered Travel Assistant**, automates business travel planning through Telegram interaction. It leverages AI models and external APIs to transcribe user inputs, extract travel details, search flights and hotels, generate personalized travel recommendations, and deliver responses back to the user.

The workflow is logically divided into four main blocks:

- **1.1 User Input Processing:** Handles incoming Telegram messages (text or voice), transcribes voice messages, and extracts structured travel information using AI.
- **1.2 Travel Search:** Queries flight and hotel APIs based on extracted travel details to gather options.
- **1.3 AI Recommendations & Customization:** Uses AI agents to generate professional travel plans combining search results and user context, maintaining conversation memory.
- **1.4 Response Delivery:** Sends the formatted travel recommendations back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 User Input Processing

**Overview:**  
This block receives user messages from Telegram, determines the message type, transcribes voice messages if needed, and extracts structured travel details from the text.

**Nodes Involved:**  
- Telegram Trigger  
- Verify Message Type (Switch)  
- Telegram (File Retrieval)  
- OpenAI (Audio Transcription)  
- Edit Fields (Set)  
- Aggregate  
- OpenAI Chat Model  
- Information Extractor

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point; listens for incoming Telegram messages (text or voice).  
  - *Config:* Uses webhook with HTTPS; no parameters needed.  
  - *Input:* Telegram message event.  
  - *Output:* Raw message data including chat ID, message type, and content.  
  - *Failures:* Webhook misconfiguration, Telegram API downtime.

- **Verify Message Type**  
  - *Type:* Switch  
  - *Role:* Routes flow based on message type (voice, text, unsupported).  
  - *Config:* Conditions to detect voice messages, text messages, or unsupported types.  
  - *Input:* Telegram Trigger output.  
  - *Output:* Three branches: voice → Telegram node; text → Edit Fields; unsupported → Unsupported Message Type node.  
  - *Failures:* Incorrect message type detection could misroute data.

- **Telegram (File Retrieval)**  
  - *Type:* Telegram node  
  - *Role:* Downloads voice message file from Telegram servers.  
  - *Config:* Uses file ID from Telegram Trigger.  
  - *Input:* Voice message file ID.  
  - *Output:* Binary audio file data.  
  - *Failures:* File not found, Telegram API errors, network issues.  
  - *Notes:* Bypassed for text messages.

- **OpenAI (Audio Transcription)**  
  - *Type:* OpenAI audio transcription node  
  - *Role:* Converts voice audio to text using OpenAI speech-to-text.  
  - *Config:* Uses OpenAI API credentials; auto language detection.  
  - *Input:* Binary audio from Telegram node.  
  - *Output:* JSON with transcribed text.  
  - *Failures:* API key invalid, audio format unsupported, transcription errors.

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Prepares or normalizes text input for downstream processing.  
  - *Config:* Sets or modifies fields as needed (e.g., extracting text from message).  
  - *Input:* Text message data or transcribed text.  
  - *Output:* Cleaned text for aggregation.  
  - *Failures:* Expression errors if fields missing.

- **Aggregate**  
  - *Type:* Aggregate node  
  - *Role:* Combines multiple inputs into a single data object for consistent processing.  
  - *Config:* Aggregates text from Edit Fields or OpenAI transcription.  
  - *Input:* Text data.  
  - *Output:* Single combined JSON object.  
  - *Failures:* Data format inconsistencies.

- **OpenAI Chat Model**  
  - *Type:* AI language model (GPT-4o)  
  - *Role:* Supports the Information Extractor node by providing LLM capabilities.  
  - *Config:* GPT-4o model selected for accuracy and speed.  
  - *Input:* Text from Aggregate node.  
  - *Output:* AI-generated responses for information extraction.  
  - *Failures:* API limits, token overuse, latency.

- **Information Extractor**  
  - *Type:* AI information extraction node  
  - *Role:* Parses natural language input to extract structured travel details (origin, destination, dates, accommodation needs).  
  - *Config:* Uses OpenAI Chat Model as language model backend.  
  - *Input:* Aggregated user text.  
  - *Output:* JSON with fields: origin (IATA), destination (IATA), departure date, return date, flight type, accommodation boolean, nights.  
  - *Failures:* Misinterpretation of input, missing fields, date format errors.

---

#### 1.2 Travel Search

**Overview:**  
This block queries external APIs to find flight and hotel options based on extracted travel details.

**Nodes Involved:**  
- Search Flights (HTTP Request)  
- Search Hotels (HTTP Request)

**Node Details:**

- **Search Flights**  
  - *Type:* HTTP Request  
  - *Role:* Queries SerpAPI Google Flights engine for flight options.  
  - *Config:* Uses extracted origin, destination, departure/return dates, flight type, adults, stops, language, currency.  
  - *Input:* Structured travel info from Information Extractor.  
  - *Output:* JSON with flight options including airlines, prices, durations.  
  - *Failures:* API key invalid, rate limits, network errors, malformed requests.

- **Search Hotels**  
  - *Type:* HTTP Request  
  - *Role:* Queries SerpAPI Google Hotels engine for accommodations.  
  - *Config:* Uses destination, check-in/check-out dates (calculated dynamically from nights), adults, language, currency.  
  - *Input:* Flight search output and accommodation flag from Information Extractor.  
  - *Output:* JSON with hotel options, prices, ratings, amenities.  
  - *Conditional Execution:* Runs only if accommodation is requested.  
  - *Failures:* API errors, date calculation errors, no hotels found.

---

#### 1.3 AI Recommendations & Customization

**Overview:**  
Combines flight and hotel data with user context to generate a professional travel plan using AI, maintaining conversation memory for coherence.

**Nodes Involved:**  
- Window Buffer Memory  
- DeepSeek Chat Model  
- Business Travel Agent

**Node Details:**

- **Window Buffer Memory**  
  - *Type:* Memory Buffer (Langchain)  
  - *Role:* Stores recent conversation history to maintain context.  
  - *Config:* Fixed session key (demo); window size controls token retention.  
  - *Input:* Conversation data from previous nodes.  
  - *Output:* Contextual memory for AI agent.  
  - *Failures:* Memory overflow, privacy concerns if session keys not dynamic.

- **DeepSeek Chat Model**  
  - *Type:* AI language model (DeepSeek)  
  - *Role:* Provides AI capabilities for the Business Travel Agent node.  
  - *Config:* Uses DeepSeek API credentials.  
  - *Input:* Context and travel data.  
  - *Output:* AI-generated travel plan text.  
  - *Failures:* API downtime, token limits.

- **Business Travel Agent**  
  - *Type:* AI Agent (Langchain Agent)  
  - *Role:* Integrates flight and hotel data, user input, and memory to produce a structured, professional travel recommendation.  
  - *Config:* Uses Anthropic model; structured prompt template defines output format and agent role.  
  - *Input:* Flight and hotel search results, user original message, conversation memory.  
  - *Output:* Formatted travel plan with flights and hotels sections, including links.  
  - *Failures:* Prompt misconfiguration, API errors, incomplete data.

---

#### 1.4 Response Delivery

**Overview:**  
Sends the AI-generated travel recommendations back to the user on Telegram.

**Nodes Involved:**  
- Response (Telegram)

**Node Details:**

- **Response**  
  - *Type:* Telegram node  
  - *Role:* Sends message to user's Telegram chat with travel recommendations.  
  - *Config:* Uses original chat ID; supports Telegram message formatting; respects 4096 character limit.  
  - *Input:* Formatted travel plan from Business Travel Agent.  
  - *Output:* Message delivered to Telegram user.  
  - *Failures:* Message too long, Telegram API errors, invalid chat ID.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                              | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                   |
|-----------------------|----------------------------------|----------------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger                 | Entry point for Telegram messages             | —                       | Verify Message Type       |                                                                                              |
| Verify Message Type    | Switch                          | Routes flow based on message type             | Telegram Trigger         | Edit Fields, Telegram, Unsupported Message Type |                                                                                              |
| Telegram              | Telegram                        | Retrieves voice message file from Telegram    | Verify Message Type      | OpenAI                   | ## Telegram File Retrieval Node\nThis node retrieves the voice message file from Telegram servers for processing.\n\n**Input:** Voice message file ID from the Telegram trigger\n**Output:** Binary file data containing the voice recording\n\n**Note:** This node specifically handles voice messages. For text messages, this step would be bypassed. |
| OpenAI                | OpenAI Audio Transcription      | Transcribes voice audio to text                | Telegram                 | Aggregate                 | ## OpenAI Audio Transcription Node\nThis node uses OpenAI's speech-to-text capabilities to transcribe the voice message into text.\n\n**Input:** Binary voice file from Telegram\n**Output:** JSON object containing the transcribed text\n\n**Technical considerations:**\n- Requires OpenAI API credentials with access to audio transcription models\n- Handles various audio formats supported by OpenAI\n- Default language is auto-detected, but can be specified in options |
| Edit Fields           | Set                            | Prepares text input for processing             | Verify Message Type      | Aggregate                 |                                                                                              |
| Aggregate             | Aggregate                      | Combines text inputs into single object        | OpenAI, Edit Fields      | Information Extractor     |                                                                                              |
| OpenAI Chat Model     | AI Language Model (GPT-4o)       | Provides LLM for information extraction        | —                       | Information Extractor     | ## OpenAI Chat Model Node\nThis node provides the AI language model capabilities to the Information Extractor.\n\n**Model:** GPT-4o\n**Purpose:** Powers the extraction of structured travel information from natural language\n\n**Technical considerations:**\n- Model selection affects accuracy and processing speed\n- API usage is charged based on token consumption\n- Can be replaced with other compatible LLM providers |
| Information Extractor | AI Information Extractor         | Extracts structured travel details from text  | Aggregate, OpenAI Chat Model | Search Flights           | ## Information Extractor Node\nThis node uses AI to parse the user's message and extract structured travel information.\n\n**Input:** Transcribed text from the voice message\n**Output:** Structured JSON with travel details (origin, destination, dates, accommodation needs)\n\n**Key extracted fields:**\n- origin: IATA airport code for departure location\n- destination: IATA airport code for arrival location\n- departure: ISO 8601 date format (YYYY-MM-DD)\n- return: ISO 8601 date format (YYYY-MM-DD)\n- flight_type: \"1\" for round-trip, \"2\" for one-way\n- accommodation: Boolean indicating if lodging is needed\n- nights: Integer number of nights for accommodation |
| Search Flights        | HTTP Request                   | Queries flight options from SerpAPI            | Information Extractor    | Search Hotels             | ## Search Flights Node\nThis HTTP Request node queries the SerpAPI Google Flights integration to find available flights based on the extracted travel information.\n\n**Input:** Travel parameters from Information Extractor (origin, destination, dates)\n**Output:** Structured JSON containing flight options, prices, durations, and airlines\n\n**API parameters:**\n- engine: google_flights\n- departure_id/arrival_id: IATA airport codes\n- outbound_date/return_date: Travel dates\n- type: Flight type (1=round-trip, 2=one-way)\n- Additional parameters: adults, stops, language, currency |
| Search Hotels         | HTTP Request                   | Queries hotel options from SerpAPI             | Search Flights           | Business Travel Agent     | ## Search Hotels Node\nThis HTTP Request node queries the SerpAPI Google Hotels integration to find accommodations at the destination.\n\n**Input:** Destination and date information from the Information Extractor\n**Output:** Structured JSON containing hotel options, prices, ratings, and amenities\n\n**Technical details:**\n- Dynamic check-out date calculation based on number of nights\n- Filters for adults, language, and currency\n- Results include property types, nightly rates, and total costs\n\n**Note:** Only executes if accommodation is required (controlled by the Information Extractor output) |
| Window Buffer Memory  | Memory Buffer (Langchain)        | Maintains conversation context for AI agent   | —                       | Business Travel Agent     | ## Window Buffer Memory Node\nThis node maintains conversation context by storing recent interactions between the user and the AI assistant.\n\n**Purpose:** Enables the AI to reference previous messages for more coherent conversations\n**Configuration:** Uses a fixed session key for demonstration purposes\n\n**Technical considerations:**\n- Memory window size affects context retention and token usage\n- In production, should use dynamic session keys based on user ID\n- Memory persistence affects privacy and data retention policies |
| DeepSeek Chat Model   | AI Language Model (DeepSeek)     | Provides AI capabilities for travel plan generation | —                       | Business Travel Agent     |                                                                                              |
| Business Travel Agent | AI Agent (Langchain Agent)        | Generates structured travel recommendations    | Search Hotels, Window Buffer Memory, DeepSeek Chat Model | Response                  | ## Business Travel Agent Node\nThis agent node combines all the gathered travel information and generates a professional, well-structured travel recommendation.\n\n**Input:** Flight and hotel search results, user's original message\n**Output:** Formatted travel recommendation with flights and hotels sections\n\n**Technical details:**\n- Uses a structured prompt template to ensure consistent formatting\n- Leverages the Anthropic model for natural language generation\n- System message defines the agent's role, objectives, and output format\n\n**Optimization opportunities:**\n- Could be enhanced with price comparison and best value highlighting\n- Potential for adding transportation options between airport and hotels\n- Could incorporate user preferences from previous interactions |
| Response              | Telegram                       | Sends final travel recommendations to user    | Business Travel Agent    | —                        | ## Telegram Response Node\nThis node sends the final travel recommendations back to the user via Telegram.\n\n**Input:** Formatted travel plan from the Business Travel Agent\n**Output:** Message delivered to the user's Telegram chat\n\n**Technical details:**\n- Uses the original chat ID from the Telegram Trigger\n- Supports Telegram's message formatting options\n- Message size limitations apply (4096 characters max per message) |
| Unsupported Message Type | Telegram                    | Sends message for unsupported message types    | Verify Message Type      | —                        |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook URL with HTTPS and connect your Telegram Bot credentials.  
   - No additional parameters needed.

2. **Add Switch Node "Verify Message Type"**  
   - Type: Switch  
   - Configure conditions to detect:  
     - Voice messages (check message.voice exists)  
     - Text messages (check message.text exists)  
     - Else branch for unsupported types.

3. **Create Telegram Node "Telegram" for Voice File Retrieval**  
   - Type: Telegram  
   - Configure to download file using file_id from voice message.  
   - Connect from "Verify Message Type" voice branch.

4. **Add OpenAI Node for Audio Transcription**  
   - Type: OpenAI (Audio Transcription)  
   - Configure with OpenAI API credentials.  
   - Input: Binary audio file from Telegram node.  
   - Output: Transcribed text JSON.

5. **Add Set Node "Edit Fields"**  
   - Type: Set  
   - Configure to extract or normalize text from either transcribed audio or text messages.  
   - Connect from "Verify Message Type" text branch.

6. **Add Aggregate Node**  
   - Type: Aggregate  
   - Configure to combine text input from both transcription and text paths into a single JSON object.  
   - Connect from OpenAI transcription and Edit Fields nodes.

7. **Add OpenAI Chat Model Node**  
   - Type: AI Language Model (GPT-4o)  
   - Configure with OpenAI API credentials and select GPT-4o model.  
   - Connect as AI language model input to Information Extractor.

8. **Add Information Extractor Node**  
   - Type: AI Information Extractor  
   - Configure to parse natural language text into structured travel details: origin, destination, departure, return, flight_type, accommodation, nights.  
   - Connect input from Aggregate node and AI language model from OpenAI Chat Model.

9. **Add HTTP Request Node "Search Flights"**  
   - Type: HTTP Request  
   - Configure to call SerpAPI Google Flights engine with parameters from Information Extractor output (origin, destination, dates, flight_type, adults, stops, language, currency).  
   - Connect from Information Extractor.

10. **Add HTTP Request Node "Search Hotels"**  
    - Type: HTTP Request  
    - Configure to call SerpAPI Google Hotels engine with destination and check-in/out dates calculated from nights.  
    - Add conditional execution to run only if accommodation is true.  
    - Connect from Search Flights.

11. **Add Window Buffer Memory Node**  
    - Type: Memory Buffer (Langchain)  
    - Configure fixed session key for demo or dynamic session key for production.  
    - Connect to provide conversation context to Business Travel Agent.

12. **Add DeepSeek Chat Model Node**  
    - Type: AI Language Model (DeepSeek)  
    - Configure with DeepSeek API credentials.  
    - Connect as AI language model input to Business Travel Agent.

13. **Add Business Travel Agent Node**  
    - Type: AI Agent (Langchain Agent)  
    - Configure with Anthropic model and structured prompt template defining agent role and output format.  
    - Input: Flight and hotel search results, conversation memory, user original message.  
    - Connect from Search Hotels, Window Buffer Memory, and DeepSeek Chat Model.

14. **Add Telegram Node "Response"**  
    - Type: Telegram  
    - Configure to send message to user's chat ID with formatted travel plan.  
    - Connect from Business Travel Agent.

15. **Add Telegram Node "Unsupported Message Type"**  
    - Type: Telegram  
    - Configure to send a polite message indicating unsupported message type.  
    - Connect from Verify Message Type unsupported branch.

16. **Connect all nodes as per above flow.**

17. **Set up credentials:**  
    - Telegram Bot credentials for Telegram nodes.  
    - OpenAI API key for transcription and chat model nodes.  
    - SerpAPI key for HTTP Request nodes querying flights and hotels.  
    - DeepSeek API key for DeepSeek Chat Model node.

18. **Activate webhook and test by sending voice and text messages to the Telegram bot.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Telegram File Retrieval Node handles only voice messages; text messages bypass this step.          | Node: Telegram (File Retrieval)                                                                 |
| OpenAI Audio Transcription requires API access to speech-to-text models; supports multiple formats.| Node: OpenAI (Audio Transcription)                                                              |
| Information Extractor outputs key travel fields in ISO 8601 and IATA codes for interoperability.   | Node: Information Extractor                                                                     |
| Search Hotels node dynamically calculates check-out date based on nights for accurate booking.    | Node: Search Hotels                                                                             |
| Window Buffer Memory uses fixed session key for demo; production should use dynamic user-based keys.| Node: Window Buffer Memory                                                                      |
| Business Travel Agent uses Anthropic model with structured prompts for consistent output formatting.| Node: Business Travel Agent                                                                     |
| Telegram Response node respects Telegram message size limits (4096 chars).                         | Node: Response                                                                                  |
| Workflow designed for business travelers, executive assistants, and frequent travelers.           | Overview                                                                                       |
| Adjust prompts and memory settings to refine AI output and manage token usage.                     | Customization & Troubleshooting                                                                |
| Ensure all API keys are active and network connectivity is stable for smooth operation.            | Prerequisites                                                                                  |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the Business Travel AI Agent workflow in n8n.