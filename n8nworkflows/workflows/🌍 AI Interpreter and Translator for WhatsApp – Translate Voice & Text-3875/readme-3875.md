🌍 AI Interpreter and Translator for WhatsApp – Translate Voice & Text

https://n8nworkflows.xyz/workflows/---ai-interpreter-and-translator-for-whatsapp---translate-voice---text-3875


# 🌍 AI Interpreter and Translator for WhatsApp – Translate Voice & Text

### 1. Workflow Overview

This workflow, titled **"🌍 AI Interpreter and Translator for WhatsApp – Translate Voice & Text"**, is designed to automate multilingual communication over WhatsApp by receiving incoming messages (text or voice), detecting the sender’s language based on their phone prefix, transcribing audio messages, translating the content using advanced AI models (GPT-4o via LangChain), and replying instantly with culturally fluent and native-tone responses.

**Target Use Cases:**  
- International customer support  
- Global community engagement  
- Multilingual chatbot interactions  
- Agencies managing diverse client bases

**Logical Blocks:**

- **1.1 Input Reception and Sender Identification**  
  Receives WhatsApp messages via webhook, identifies the sender’s phone number and extracts country/language information.

- **1.2 Media Handling and Transcription**  
  Processes incoming audio messages by downloading media, converting it to a file, and transcribing speech to text using OpenAI Whisper.

- **1.3 Language Detection and Routing**  
  Maps sender phone prefix to language, filters messages by country, and routes text or transcribed audio for translation.

- **1.4 AI Translation and Response Generation**  
  Uses LangChain agents with GPT-4o and other AI tools to translate and generate culturally appropriate replies.

- **1.5 Response Formatting and Sending**  
  Structures the AI-generated reply into WhatsApp-compatible JSON and sends it back via WhatsApp API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Sender Identification

- **Overview:**  
  This block captures incoming WhatsApp messages via a webhook, identifies the sender, and prepares the data for further processing.

- **Nodes Involved:**  
  - Webhook  
  - Who Sent? (If)  
  - Your Number (If)  
  - Format Phone (Code)  
  - PrefixMap (Code)  
  - Filter the Country (Filter)  
  - International (Filter)

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook  
    - Role: Entry point receiving incoming WhatsApp messages (text or audio).  
    - Configuration: Listens for POST requests with message payloads.  
    - Inputs: External WhatsApp API calls.  
    - Outputs: Routes to "Who Sent?" node.  
    - Edge Cases: Missing or malformed payload, webhook authentication failure.

  - **Who Sent? (If)**  
    - Type: Conditional (If)  
    - Role: Checks if the sender is the user’s own number or an external number.  
    - Configuration: Compares incoming phone number with stored user number.  
    - Inputs: Webhook output.  
    - Outputs: Branches to "Your Number" or "Format Phone".  
    - Edge Cases: Incorrect number format, missing sender info.

  - **Your Number (If)**  
    - Type: Conditional (If)  
    - Role: Further filters messages from the user’s own number.  
    - Inputs: From "Who Sent?" node.  
    - Outputs: Routes to "Audio2" (for audio handling) or "PrefixMap".  
    - Edge Cases: Misclassification of sender.

  - **Format Phone (Code)**  
    - Type: Code (JavaScript)  
    - Role: Normalizes and formats the sender’s phone number for prefix extraction.  
    - Inputs: From "Who Sent?" node.  
    - Outputs: "International" filter.  
    - Edge Cases: Unexpected phone number formats, missing prefixes.

  - **PrefixMap (Code)**  
    - Type: Code (JavaScript)  
    - Role: Maps phone number prefixes to country and language codes.  
    - Inputs: From "Your Number" node.  
    - Outputs: "Filter the Country".  
    - Edge Cases: Unknown or unsupported prefixes.

  - **Filter the Country (Filter)**  
    - Type: Filter  
    - Role: Filters messages based on country/language mapping.  
    - Inputs: From "PrefixMap".  
    - Outputs: Routes to audio or text processing.  
    - Edge Cases: No matching country found.

  - **International (Filter)**  
    - Type: Filter  
    - Role: Determines if the message is from an international sender for special handling.  
    - Inputs: From "Format Phone".  
    - Outputs: Routes to "Audio" node for audio check.  
    - Edge Cases: Ambiguous country codes.

---

#### 1.2 Media Handling and Transcription

- **Overview:**  
  This block processes incoming audio messages by downloading media files, converting them to appropriate formats, and transcribing speech to text using OpenAI Whisper.

- **Nodes Involved:**  
  - Audio (If)  
  - Audio1 (If)  
  - Audio2 (If)  
  - GET Media  
  - GET Media1  
  - GET Media2  
  - Convert to File  
  - Convert to File1  
  - Convert to File2  
  - OpenAI Whisper (OpenAI nodes with transcription role)

- **Node Details:**

  - **Audio, Audio1, Audio2 (If)**  
    - Type: Conditional (If)  
    - Role: Checks if the incoming message contains audio media.  
    - Inputs: From "International" or "Your Number".  
    - Outputs: Routes to media download or text mapping.  
    - Edge Cases: Missing media URL, unsupported media types.

  - **GET Media, GET Media1, GET Media2 (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Downloads the audio media from WhatsApp or Evolution API.  
    - Configuration: Uses media URLs from incoming payload, authenticated with API credentials.  
    - Inputs: From respective Audio If nodes.  
    - Outputs: Passes media binary data to "Convert to File" nodes.  
    - Edge Cases: Network errors, expired media URLs, auth failures.

  - **Convert to File, Convert to File1, Convert to File2 (ConvertToFile)**  
    - Type: Convert to File  
    - Role: Converts downloaded binary media into file format suitable for transcription.  
    - Inputs: From GET Media nodes.  
    - Outputs: Passes file to OpenAI transcription nodes.  
    - Edge Cases: Unsupported file formats, conversion failures.

  - **OpenAI Whisper (OpenAI nodes)**  
    - Type: OpenAI (LangChain)  
    - Role: Transcribes audio files into text using OpenAI Whisper model.  
    - Inputs: From Convert to File nodes.  
    - Outputs: Transcribed text forwarded to text mapping nodes.  
    - Edge Cases: API rate limits, transcription inaccuracies, audio quality issues.

---

#### 1.3 Language Detection and Routing

- **Overview:**  
  This block detects the language of the message, classifies text content, and routes the message to appropriate AI translation agents.

- **Nodes Involved:**  
  - Text Mapping, Text Mapping1, Text Mapping2 (Set)  
  - Text Classifier, Text Classifier1 (LangChain Text Classifier)  
  - Filter  
  - AI Agent, AI Agent1, AI Agent2 (LangChain Agents)  
  - Calculator, Calculator1, Calculator2 (LangChain Calculator Tool)  
  - Think, Think1, Think2 (LangChain Think Tool)  
  - OpenAI Chat Model, OpenAI Chat Model1, OpenAI Chat Model2, OpenAI Chat Model3 (LangChain Chat Models)  
  - Groq Chat Model (LangChain Chat Model)

- **Node Details:**

  - **Text Mapping, Text Mapping1, Text Mapping2 (Set)**  
    - Type: Set  
    - Role: Prepares and structures text data for classification and AI processing.  
    - Inputs: From transcription or direct text input.  
    - Outputs: To Text Classifier nodes.  
    - Edge Cases: Missing or malformed text data.

  - **Text Classifier, Text Classifier1 (LangChain Text Classifier)**  
    - Type: AI Text Classifier  
    - Role: Classifies text to determine language, intent, or translation needs.  
    - Inputs: From Text Mapping nodes.  
    - Outputs: To AI Agent nodes or further classification.  
    - Edge Cases: Misclassification, ambiguous text.

  - **Filter**  
    - Type: Filter  
    - Role: Filters messages based on classification results for routing.  
    - Inputs: From AI Agent or Text Classifier.  
    - Outputs: Routes to Code or further AI processing.  
    - Edge Cases: Incorrect routing due to classification errors.

  - **AI Agent, AI Agent1, AI Agent2 (LangChain Agents)**  
    - Type: AI Agent  
    - Role: Core AI processing nodes that use GPT-4o and LangChain to translate and generate replies.  
    - Inputs: From Text Classifier or Calculator nodes.  
    - Outputs: To subsequent nodes for formatting or sending.  
    - Edge Cases: API errors, timeout, unexpected input formats.

  - **Calculator, Calculator1, Calculator2 (LangChain Calculator Tool)**  
    - Type: AI Calculator Tool  
    - Role: Performs auxiliary computations or data processing to support AI agents.  
    - Inputs: From AI Agent or Chat Model nodes.  
    - Outputs: To AI Agent nodes.  
    - Edge Cases: Calculation errors, invalid inputs.

  - **Think, Think1, Think2 (LangChain Think Tool)**  
    - Type: AI Think Tool  
    - Role: Provides reasoning or intermediate processing steps for AI agents.  
    - Inputs: From Chat Models or AI Agents.  
    - Outputs: To AI Agent nodes.  
    - Edge Cases: Logic errors, incomplete reasoning.

  - **OpenAI Chat Model, OpenAI Chat Model1, OpenAI Chat Model2, OpenAI Chat Model3 (LangChain Chat Models)**  
    - Type: AI Chat Model  
    - Role: Provides conversational AI capabilities using OpenAI GPT models.  
    - Inputs: From various nodes including Calculator and Think.  
    - Outputs: To AI Agent or Text Classifier nodes.  
    - Edge Cases: API rate limits, malformed prompts.

  - **Groq Chat Model**  
    - Type: AI Chat Model (Groq)  
    - Role: Alternative AI conversational model for classification or translation.  
    - Inputs: From Text Classifier.  
    - Outputs: To Text Classifier or AI Agent.  
    - Edge Cases: Model availability, API errors.

---

#### 1.4 AI Translation and Response Generation

- **Overview:**  
  This block uses AI agents to translate the message content into the target language with native tone and cultural fluency, generating a reply.

- **Nodes Involved:**  
  - AI Agent, AI Agent1, AI Agent2  
  - OpenAI Chat Model, OpenAI Chat Model1, OpenAI Chat Model2, OpenAI Chat Model3  
  - Calculator, Calculator1, Calculator2  
  - Think, Think1, Think2

- **Node Details:**  
  (Details largely overlap with 1.3 as AI Agents and related tools perform translation and response generation.)

  - AI Agents use GPT-4o via LangChain to translate and generate replies.  
  - Calculator and Think nodes assist in complex reasoning or calculations if needed.  
  - Chat Models provide conversational context and tone adaptation.

  - Edge Cases:  
    - API failures or timeouts  
    - Unexpected input formats causing translation errors  
    - Rate limits on OpenAI or Groq APIs

---

#### 1.5 Response Formatting and Sending

- **Overview:**  
  Formats the AI-generated reply into WhatsApp-compatible JSON and sends it back to the sender using WhatsApp API.

- **Nodes Involved:**  
  - Code, Code1 (Code nodes)  
  - WhatsApp, WhatsApp1, WhatsApp2 (HTTP Request)

- **Node Details:**

  - **Code, Code1 (Code)**  
    - Type: Code (JavaScript)  
    - Role: Formats AI response into JSON structure expected by WhatsApp API.  
    - Inputs: From AI Agent or Filter nodes.  
    - Outputs: To WhatsApp HTTP Request nodes.  
    - Edge Cases: JSON formatting errors, missing fields.

  - **WhatsApp, WhatsApp1, WhatsApp2 (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Sends the formatted reply back to WhatsApp API for delivery.  
    - Configuration: Uses WhatsApp API credentials and endpoints.  
    - Inputs: From Code nodes.  
    - Outputs: Final step; no further nodes.  
    - Edge Cases: API authentication errors, message send failures, network issues.

---

### 3. Summary Table

| Node Name          | Node Type                      | Functional Role                          | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                  |
|--------------------|--------------------------------|----------------------------------------|---------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Webhook            | HTTP Webhook                   | Entry point for incoming WhatsApp msgs | -                         | Who Sent?                |                                                                                              |
| Who Sent?           | If                            | Checks sender identity                  | Webhook                   | Your Number, Format Phone |                                                                                              |
| Your Number         | If                            | Filters messages from own number       | Who Sent?                 | Audio2, PrefixMap         |                                                                                              |
| Format Phone        | Code                          | Normalizes sender phone number         | Who Sent?                 | International            |                                                                                              |
| PrefixMap           | Code                          | Maps phone prefix to country/language  | Your Number               | Filter the Country       |                                                                                              |
| Filter the Country  | Filter                        | Filters messages by country             | PrefixMap                 | Audio1                   |                                                                                              |
| International       | Filter                        | Determines international messages       | Format Phone              | Audio                    |                                                                                              |
| Audio               | If                            | Checks if message contains audio        | International             | GET Media2, Text Mapping |                                                                                              |
| Audio1              | If                            | Checks if message contains audio        | Filter the Country        | GET Media1, Text Mapping1|                                                                                              |
| Audio2              | If                            | Checks if message contains audio        | Your Number               | GET Media, Text Mapping2 |                                                                                              |
| GET Media           | HTTP Request                  | Downloads audio media                   | Audio2                    | Convert to File2         |                                                                                              |
| GET Media1          | HTTP Request                  | Downloads audio media                   | Audio1                    | Convert to File          |                                                                                              |
| GET Media2          | HTTP Request                  | Downloads audio media                   | Audio                     | Convert to File1         |                                                                                              |
| Convert to File     | ConvertToFile                 | Converts media to file format           | GET Media1                 | OpenAI1                  |                                                                                              |
| Convert to File1    | ConvertToFile                 | Converts media to file format           | GET Media2                 | OpenAI                   |                                                                                              |
| Convert to File2    | ConvertToFile                 | Converts media to file format           | GET Media                  | OpenAI2                  |                                                                                              |
| OpenAI              | OpenAI (LangChain)            | Transcribes audio or translates text   | Convert to File1           | Text Mapping1            |                                                                                              |
| OpenAI1             | OpenAI (LangChain)            | Transcribes audio or translates text   | Convert to File            | Text Mapping             |                                                                                              |
| OpenAI2             | OpenAI (LangChain)            | Transcribes audio or translates text   | Convert to File2           | Text Mapping2            |                                                                                              |
| Text Mapping        | Set                           | Prepares text for classification       | OpenAI1                   | Text Classifier          |                                                                                              |
| Text Mapping1       | Set                           | Prepares text for classification       | OpenAI                    | AI Agent1                |                                                                                              |
| Text Mapping2       | Set                           | Prepares text for classification       | OpenAI2                   | AI Agent2                |                                                                                              |
| Text Classifier     | LangChain Text Classifier     | Classifies text language/intent         | Text Mapping              | AI Agent, Text Classifier1|                                                                                              |
| Text Classifier1    | LangChain Text Classifier     | Further classification                  | Text Classifier           | AI Agent                 |                                                                                              |
| AI Agent            | LangChain Agent               | Core AI translation and reply generation| Text Classifier1           | Filter                   |                                                                                              |
| AI Agent1           | LangChain Agent               | AI translation and reply generation    | Text Mapping1             | Code1                    |                                                                                              |
| AI Agent2           | LangChain Agent               | AI translation and reply generation    | Text Mapping2             | WhatsApp                 |                                                                                              |
| Filter              | Filter                        | Routes AI response for formatting      | AI Agent                  | Code                     |                                                                                              |
| Code                | Code                          | Formats reply JSON for WhatsApp         | Filter                    | WhatsApp1                |                                                                                              |
| Code1               | Code                          | Formats reply JSON for WhatsApp         | AI Agent1                 | WhatsApp2                |                                                                                              |
| WhatsApp            | HTTP Request                  | Sends reply via WhatsApp API             | AI Agent2, Code1          | -                        |                                                                                              |
| WhatsApp1           | HTTP Request                  | Sends reply via WhatsApp API             | Code                      | -                        |                                                                                              |
| WhatsApp2           | HTTP Request                  | Sends reply via WhatsApp API             | Code1                     | -                        |                                                                                              |
| Calculator          | LangChain Calculator Tool     | Supports AI agents with calculations    | AI Agent                  | AI Agent1                |                                                                                              |
| Calculator1         | LangChain Calculator Tool     | Supports AI agents with calculations    | AI Agent                  | AI Agent                 |                                                                                              |
| Calculator2         | LangChain Calculator Tool     | Supports AI agents with calculations    | AI Agent2                 | AI Agent2                |                                                                                              |
| Think               | LangChain Think Tool          | Provides reasoning for AI agents        | OpenAI Chat Model         | AI Agent                 |                                                                                              |
| Think1              | LangChain Think Tool          | Provides reasoning for AI agents        | OpenAI Chat Model         | AI Agent1                |                                                                                              |
| Think2              | LangChain Think Tool          | Provides reasoning for AI agents        | OpenAI Chat Model3        | AI Agent2                |                                                                                              |
| OpenAI Chat Model   | LangChain Chat Model          | Conversational AI model                  | Calculator                | AI Agent1                |                                                                                              |
| OpenAI Chat Model1  | LangChain Chat Model          | Conversational AI model                  | Text Classifier1          | AI Agent                 |                                                                                              |
| OpenAI Chat Model2  | LangChain Chat Model          | Conversational AI model                  | AI Agent                  | AI Agent                 |                                                                                              |
| OpenAI Chat Model3  | LangChain Chat Model          | Conversational AI model                  | AI Agent2                 | Think2                   |                                                                                              |
| Groq Chat Model     | LangChain Chat Model (Groq)  | Alternative AI chat model for classification | Text Classifier          | Text Classifier          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - Purpose: Receive incoming WhatsApp messages (text or audio).  
   - Configure webhook URL and HTTP method POST.

2. **Add "Who Sent?" Node (If)**  
   - Type: If  
   - Condition: Check if incoming phone number equals your own number.  
   - Connect Webhook → Who Sent?

3. **Add "Your Number" Node (If)**  
   - Type: If  
   - Condition: Further filter messages from your own number.  
   - Connect Who Sent? → Your Number

4. **Add "Format Phone" Node (Code)**  
   - Type: Code  
   - Purpose: Normalize sender phone number for prefix extraction.  
   - Connect Who Sent? → Format Phone

5. **Add "PrefixMap" Node (Code)**  
   - Type: Code  
   - Purpose: Map phone prefix to country/language code.  
   - Connect Your Number → PrefixMap

6. **Add "Filter the Country" Node (Filter)**  
   - Type: Filter  
   - Purpose: Filter messages by country from prefix mapping.  
   - Connect PrefixMap → Filter the Country

7. **Add "International" Node (Filter)**  
   - Type: Filter  
   - Purpose: Detect if message is from international sender.  
   - Connect Format Phone → International

8. **Add "Audio", "Audio1", "Audio2" Nodes (If)**  
   - Type: If  
   - Purpose: Check if message contains audio media.  
   - Connect International → Audio  
   - Connect Filter the Country → Audio1  
   - Connect Your Number → Audio2

9. **Add "GET Media", "GET Media1", "GET Media2" Nodes (HTTP Request)**  
   - Type: HTTP Request  
   - Purpose: Download audio media files using media URLs.  
   - Connect Audio2 → GET Media  
   - Connect Audio1 → GET Media1  
   - Connect Audio → GET Media2  
   - Configure with WhatsApp API credentials and media URLs.

10. **Add "Convert to File", "Convert to File1", "Convert to File2" Nodes (ConvertToFile)**  
    - Type: ConvertToFile  
    - Purpose: Convert downloaded media to file format for transcription.  
    - Connect GET Media1 → Convert to File  
    - Connect GET Media2 → Convert to File1  
    - Connect GET Media → Convert to File2

11. **Add OpenAI Whisper Nodes (OpenAI)**  
    - Type: OpenAI (LangChain)  
    - Purpose: Transcribe audio to text using Whisper model.  
    - Connect Convert to File → OpenAI1  
    - Connect Convert to File1 → OpenAI  
    - Connect Convert to File2 → OpenAI2  
    - Configure with OpenAI API key and Whisper model.

12. **Add Text Mapping Nodes (Set)**  
    - Type: Set  
    - Purpose: Prepare transcribed or text data for classification.  
    - Connect OpenAI1 → Text Mapping  
    - Connect OpenAI → Text Mapping1  
    - Connect OpenAI2 → Text Mapping2

13. **Add Text Classifier Nodes (LangChain Text Classifier)**  
    - Type: Text Classifier  
    - Purpose: Detect language and intent of text.  
    - Connect Text Mapping → Text Classifier  
    - Connect Text Classifier → Text Classifier1

14. **Add AI Agent Nodes (LangChain Agent)**  
    - Type: AI Agent  
    - Purpose: Translate and generate reply using GPT-4o via LangChain.  
    - Connect Text Classifier1 → AI Agent  
    - Connect Text Mapping1 → AI Agent1  
    - Connect Text Mapping2 → AI Agent2  
    - Configure with OpenAI API key and GPT-4o model.

15. **Add Calculator and Think Nodes (LangChain Tools)**  
    - Type: Calculator and Think  
    - Purpose: Support AI agents with calculations and reasoning.  
    - Connect Calculator → AI Agent1  
    - Connect Calculator1 → AI Agent  
    - Connect Calculator2 → AI Agent2  
    - Connect Think → AI Agent  
    - Connect Think1 → AI Agent1  
    - Connect Think2 → AI Agent2

16. **Add OpenAI Chat Model Nodes (LangChain Chat Model)**  
    - Type: Chat Model  
    - Purpose: Provide conversational context and tone adaptation.  
    - Connect Calculator → OpenAI Chat Model  
    - Connect Text Classifier1 → OpenAI Chat Model1  
    - Connect AI Agent → OpenAI Chat Model2  
    - Connect AI Agent2 → OpenAI Chat Model3

17. **Add Filter Node**  
    - Type: Filter  
    - Purpose: Route AI agent output for formatting.  
    - Connect AI Agent → Filter

18. **Add Code Nodes for Formatting**  
    - Type: Code  
    - Purpose: Format AI response into WhatsApp JSON structure.  
    - Connect Filter → Code  
    - Connect AI Agent1 → Code1

19. **Add WhatsApp HTTP Request Nodes**  
    - Type: HTTP Request  
    - Purpose: Send formatted reply back via WhatsApp API.  
    - Connect Code → WhatsApp1  
    - Connect Code1 → WhatsApp2  
    - Connect AI Agent2 → WhatsApp  
    - Configure with WhatsApp API credentials and endpoints.

20. **Set Credentials**  
    - Configure OpenAI API credentials for all OpenAI nodes.  
    - Configure WhatsApp API or Evolution API credentials for HTTP Request nodes.

21. **Test Workflow**  
    - Send test WhatsApp messages (text and voice) to webhook URL.  
    - Verify transcription, translation, and reply delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Workflow image illustrating architecture and flow: ![AI Translation N8N](https://i.ibb.co/JWJWP03y/AI-Translator-for-Whats-App-N8-N.png) | Workflow description                                       |
| Developed by Bruno, expert in AI-powered workflows for n8n and Make for 2+ years                     | Author credentials                                        |
| Purchase workflows and support: [https://iloveflows.com](https://iloveflows.com)                     | Purchase link                                             |
| Try n8n Cloud for easy deployment: [https://n8n.partnerlinks.io/amanda](https://n8n.partnerlinks.io/amanda) | Deployment option                                         |
| Requires OpenAI API key and WhatsApp or Evolution API access                                         | Prerequisites                                             |
| Optional integrations with Typebot, Airtable, CRM possible                                          | Extension possibilities                                   |

---

This documentation provides a detailed, structured reference for the entire workflow, enabling advanced users and AI agents to understand, reproduce, and modify the multilingual WhatsApp translation automation confidently.