Facebook Messenger Bot with GPT-4 for Text, Image & Voice Processing

https://n8nworkflows.xyz/workflows/facebook-messenger-bot-with-gpt-4-for-text--image---voice-processing-9347


# Facebook Messenger Bot with GPT-4 for Text, Image & Voice Processing

---

### 1. Workflow Overview

This workflow implements a **Facebook Messenger AI Bot** leveraging GPT-4 capabilities to process **text, images, and voice messages** received via Messenger. It supports dynamic message type detection, media downloading, transcription, AI-driven response generation with memory and tool integration, and replies back to users through the Facebook Graph API.

**Target Use Cases:**  
- Automating customer support or conversational agents on Facebook Messenger  
- Processing multimedia messages (text, image, audio) intelligently  
- Maintaining conversation context per user  
- Using external tools (Calculator, Wikipedia) to augment AI responses  

**Logical Blocks:**  
- **1.1 Webhook & Input Reception:** Receives Messenger events and extracts message content.  
- **1.2 Message Type Identification & Routing:** Detects message type and routes to appropriate processing paths.  
- **1.3 Media Download & Processing:** Downloads images or audio and converts them to text or description.  
- **1.4 AI Processing with Memory & Tools:** Uses GPT-4 chat model and Langchain agent with memory, Calculator, and Wikipedia tools.  
- **1.5 Post-processing & Formatting:** Enhances output text with Unicode bold formatting.  
- **1.6 Response Construction & Sending:** Builds Facebook Messenger response payload and sends it via Graph API.  

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook & Input Reception

- **Overview:**  
  Listens for incoming Facebook Messenger POST events, acknowledges them immediately, and extracts the raw message data for downstream processing.

- **Nodes Involved:**  
  - Webhook1  
  - Identify Message Type (Function)  
  - Sticky Note2 (comment)

- **Node Details:**  

  - **Webhook1**  
    - Type: Webhook (HTTP POST listener)  
    - Config: Path `/messenger`, HTTP method POST  
    - Role: Entry point for Messenger events, triggers workflow on message receipt  
    - Inputs: External webhook POST from Facebook Messenger  
    - Outputs: JSON payload containing Messenger event data  
    - Edge cases: Missing or malformed webhook calls, Facebook verification issues  
    - Sticky Note2 explains immediate acknowledgement and event reception  

  - **Identify Message Type (Function)**  
    - Type: Function node (JavaScript)  
    - Role: Parses Facebook payload to extract meaningful message data into a uniform format  
    - Config:  
      - Extracts message object from different possible JSON paths  
      - Detects if message contains attachments (image/audio) or text  
      - Returns an object with `type` (text, image, audio, unknown) and content or URL  
    - Inputs: Payload from Webhook1  
    - Outputs: Simplified message object for routing  
    - Edge cases: Missing `message` field triggers error, unsupported attachment types  
    - Sticky Note3 notes this standardization step for downstream predictability  

#### 1.2 Message Type Identification & Routing

- **Overview:**  
  Routes the simplified message object to one of three processing lanes based on detected type: image, audio, or text.

- **Nodes Involved:**  
  - Identify and ReRoute Message Types (Switch)  
  - Sticky Note (comment)

- **Node Details:**  

  - **Identify and ReRoute Message Types (Switch)**  
    - Type: Switch node  
    - Role: Directs message by checking `type` property in JSON (`image`, `audio`, `text`)  
    - Config: Three output routes named "Image Message", "Audio Message", and "Text Message"  
    - Inputs: Output from Identify Message Type node  
    - Outputs: One of three routes according to message type  
    - Edge cases: If message type does not match any route, no default route is defined (potential missing handling for unknown types)  
    - Sticky Note explains routing logic and suggests adding default route for unsupported formats  

#### 1.3 Media Download & Processing

- **Overview:**  
  Downloads media content (images or audio) from provided URLs, then processes them to extract descriptive or transcribed text.

- **Nodes Involved:**  
  - Download Image (HTTP Request)  
  - Analyze image (OpenAI Vision Model)  
  - Edit Fields3 (Set)  
  - Download Audio (HTTP Request)  
  - Audio Transcriber (OpenAI Audio Transcription)  
  - Edit Fields2 (Set)  
  - Sticky Notes 4, 5, 6, 7, 8, 9  

- **Node Details:**  

  - **Download Image**  
    - Type: HTTP Request  
    - Role: Downloads image binary data from Messenger URL  
    - Config: URL from message JSON; uses predefined Facebook Graph API credential for authentication  
    - Inputs: From "Image Message" route  
    - Outputs: Image file data as base64 or binary  
    - Edge cases: Facebook blocking media without proper token; token must be passed as query or header if needed  
    - Sticky Note4 provides download tips  

  - **Analyze image**  
    - Type: OpenAI Vision Model (Langchain OpenAI node)  
    - Role: Sends image content as base64 to GPT-4o model for descriptive text analysis  
    - Config: Model `gpt-4o`, operation = analyze image  
    - Inputs: Output of Download Image node  
    - Outputs: Text description of image  
    - Edge cases: Model API errors, large image size, rate limits  
    - Sticky Note5 highlights vision usage  

  - **Edit Fields3**  
    - Type: Set node  
    - Role: Assigns image description text to `user_prompt` field for AI processing  
    - Inputs: Output of Analyze Image  
    - Outputs: Enriched JSON with `user_prompt`  
    - Sticky Note6 notes preservation of other fields  

  - **Download Audio**  
    - Type: HTTP Request  
    - Role: Downloads audio file from Messenger URL  
    - Config: URL from message JSON; uses Facebook Graph API credential  
    - Inputs: From "Audio Message" route  
    - Outputs: Audio file data  
    - Edge cases: Token requirements similar to image download  
    - Sticky Note7 details download step  

  - **Audio Transcriber**  
    - Type: OpenAI Audio Transcription  
    - Role: Transcribes audio into text with automatic language detection  
    - Config: Operation = transcribe  
    - Inputs: Output of Download Audio  
    - Outputs: Transcribed text  
    - Edge cases: Audio quality, unsupported formats, API errors  
    - Sticky Note8 explains transcription  

  - **Edit Fields2**  
    - Type: Set node  
    - Role: Maps transcription text into `user_prompt` for AI processing  
    - Inputs: Output of Audio Transcriber  
    - Outputs: JSON with `user_prompt`  
    - Sticky Note9 suggests trimming whitespace or further cleaning  

#### 1.4 AI Processing with Memory & Tools

- **Overview:**  
  Uses the GPT-4 chat model with Langchain agent to generate AI responses based on user input. It integrates memory to maintain context per user and connects to external tools (Calculator, Wikipedia) for enriched answers.

- **Nodes Involved:**  
  - Edit Fields1 (Set)  
  - Simple Memory (Langchain Memory Buffer Window)  
  - OpenAI Chat Model (Langchain LM Chat OpenAI)  
  - Calculator (Langchain Tool)  
  - Wikipedia (Langchain Tool)  
  - AI Agent Messenger (Langchain Agent)  
  - Sticky Note14  

- **Node Details:**  

  - **Edit Fields1**  
    - Type: Set node  
    - Role: For text messages, assigns user text to `user_prompt`  
    - Inputs: From "Text Message" route  
    - Outputs: JSON with `user_prompt`  
    - Sticky Note10 indicates optional truncation  

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Stores conversation context per user using PSID as session key  
    - Config: Session key extracted dynamically from webhook sender ID  
    - Inputs: Connected as AI memory input to AI Agent Messenger  
    - Outputs: Maintains message history for context-aware replies  
    - Edge cases: Memory size limits, session key extraction failures  

  - **OpenAI Chat Model**  
    - Type: Langchain LM Chat OpenAI  
    - Role: GPT-4.1-mini chat model for conversation generation  
    - Config: Model version `gpt-4.1-mini`, temperature and other options default  
    - Inputs: AI Agent Messenger AI language model input  
    - Outputs: AI-generated text response  

  - **Calculator & Wikipedia**  
    - Type: Langchain Tools  
    - Role: Provide specialized computational and factual lookup capabilities  
    - Config: Default settings, called as needed by AI Agent rules  
    - Inputs: Connected as AI tools to AI Agent Messenger  
    - Outputs: Tool responses included in AI Agent output  
    - Sticky Note14 explains logic: Calculator for math/finance/stats, Wikipedia for general info  

  - **AI Agent Messenger**  
    - Type: Langchain Agent  
    - Role: Core orchestration node that takes `user_prompt`, runs system message, invokes chat model, memory, and tools, and produces final AI output  
    - Config:  
      - System Message specifies assistant role and tool usage rules  
      - Prompt type: Define with system message  
    - Inputs: Receives `user_prompt` from Edit Fields nodes, AI tools, AI memory, AI language model  
    - Outputs: AI response text  
    - Edge cases: Tool invocation failures, API rate limits, memory key mismatches, system message misconfigurations  

#### 1.5 Post-processing & Formatting

- **Overview:**  
  Converts markdown-style bold markers (`**bold**`, `*bold*`) in AI output to Unicode bold characters and removes leftover asterisks for improved Messenger text rendering.

- **Nodes Involved:**  
  - Code (JavaScript)  
  - Sticky Note11  

- **Node Details:**  

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Post-processing AI output text in field `output`  
    - Config:  
      - Regular expressions detect bold markers  
      - Converts ASCII letters and digits inside markers to Unicode bold characters  
      - Removes remaining asterisks  
    - Inputs: AI Agent Messenger output JSON  
    - Outputs: Formatted text with Unicode bold  
    - Edge cases: No `output` field, malformed markdown, Unicode rendering in Messenger clients  
    - Sticky Note11 explains rationale for usage  

#### 1.6 Response Construction & Sending

- **Overview:**  
  Builds the Facebook Messenger API response payload with proper recipient ID and text, then sends it via Graph API to reply to the user.

- **Nodes Involved:**  
  - Code1 (JavaScript)  
  - Send Response (HTTP Request)  
  - Sticky Notes12, 13  

- **Node Details:**  

  - **Code1**  
    - Type: Code node (JavaScript)  
    - Role: Constructs Messenger response JSON with `messaging_type: "RESPONSE"`, recipient PSID, and message text from processed output  
    - Config:  
      - Extracts PSID from Webhook1 payload dynamically  
      - Uses output text, fallback to default if empty  
      - Ensures recipient id is string for API compliance  
    - Inputs: Output from Code (formatted output)  
    - Outputs: Messenger API JSON payload  
    - Edge cases: Missing PSID, empty output text, message length limits  
    - Sticky Note12 describes payload structure  

  - **Send Response**  
    - Type: HTTP Request  
    - Role: Sends POST to `https://graph.facebook.com/v17.0/me/messages?access_token=PAGE_TOKEN`  
    - Config:  
      - Uses raw JSON body from Code1 output  
      - Sets Content-Type: application/json  
      - Authentication: Access token via URL query parameter (replace PAGE_TOKEN with credential)  
    - Inputs: JSON payload from Code1  
    - Outputs: Facebook API response  
    - Edge cases: HTTP errors 401/403/429 indicate token/permission issues or rate limiting; retry and logging recommended  
    - Sticky Note13 advises on error handling and token verification  

---

### 3. Summary Table

| Node Name                      | Node Type                       | Functional Role                          | Input Node(s)               | Output Node(s)                       | Sticky Note                                                 |
|--------------------------------|--------------------------------|----------------------------------------|-----------------------------|------------------------------------|-------------------------------------------------------------|
| Webhook1                      | Webhook                        | Entry point for Messenger events       | External webhook POST       | Identify Message Type              | Receives Messenger events and starts the processing. Reply “Immediately” to acknowledge fast, then let the flow continue. |
| Identify Message Type          | Function                       | Extracts and standardizes message data | Webhook1                   | Identify and ReRoute Message Types | Converts Messenger payload into simple shape: text, image, audio, or unknown. Makes flow predictable.                    |
| Identify and ReRoute Message Types | Switch                        | Routes message by type                  | Identify Message Type       | Download Image, Download Audio, Edit Fields1 | Routes each message down the right lane (image/audio/text). Add a default route for unsupported formats.                |
| Download Image                | HTTP Request                   | Downloads image media                   | Identify and ReRoute Message Types | Analyze image                     | Downloads image; add Page Token if Facebook blocks it.         |
| Analyze image                | OpenAI Vision Model            | Describes image content                 | Download Image              | Edit Fields3                      | Uses vision model to describe the image and produce text.       |
| Edit Fields3                | Set                           | Maps image description to user_prompt  | Analyze image              | AI Agent Messenger               | Copies image description into user_prompt; keeps other fields. |
| Download Audio                | HTTP Request                   | Downloads audio media                   | Identify and ReRoute Message Types | Audio Transcriber               | Downloads audio; add Page Token if Facebook blocks it.         |
| Audio Transcriber            | OpenAI Audio Transcription     | Transcribes audio to text               | Download Audio             | Edit Fields2                     | Transcribes voice note into clean text.                        |
| Edit Fields2                | Set                           | Maps transcription text to user_prompt | Audio Transcriber           | AI Agent Messenger               | Puts audio transcript into user_prompt; trimming possible.    |
| Edit Fields1                | Set                           | Maps user text to user_prompt           | Identify and ReRoute Message Types | AI Agent Messenger               | Maps user's text into user_prompt; optional truncation.       |
| Simple Memory                | Langchain Memory Buffer Window | Maintains per-user conversation memory | AI Agent Messenger          | AI Agent Messenger               | Enables context persistence using PSID as sessionId.          |
| OpenAI Chat Model            | Langchain LM Chat OpenAI       | GPT-4 chat model for response generation | AI Agent Messenger          | AI Agent Messenger               | Core chat model node.                                           |
| Calculator                  | Langchain Tool                | Supports math/financial/statistical calculations | AI Agent Messenger          | AI Agent Messenger               | Used for calculation tasks only.                              |
| Wikipedia                   | Langchain Tool                | Supports factual lookup and information retrieval | AI Agent Messenger          | AI Agent Messenger               | Used for general knowledge queries.                           |
| AI Agent Messenger           | Langchain Agent               | Orchestrates AI response with memory & tools | Edit Fields1,2,3 + Simple Memory + OpenAI Chat Model + Calculator + Wikipedia | Code                         | Applies system message, calls chat model, tools, and memory. |
| Code                       | Code                          | Converts markdown bold to Unicode bold | AI Agent Messenger          | Code1                          | Converts **...**/*...* to Unicode bold, strips leftover asterisks. |
| Code1                      | Code                          | Builds Messenger response JSON payload | Code                       | Send Response                  | Builds Graph API payload with recipient ID and message text.   |
| Send Response              | HTTP Request                  | Sends response via Facebook Graph API  | Code1                      | None                          | Posts JSON to Messenger API; handle errors and permissions.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `/messenger`  
   - HTTP Method: POST  
   - Purpose: Entry point for Facebook Messenger events  

2. **Add Function Node "Identify Message Type"**  
   - Code: Extract message from webhook payload, detect type (text, image, audio), output simplified object with `type` and content fields  

3. **Add Switch Node "Identify and ReRoute Message Types"**  
   - Setup three outputs:  
     - Condition 1: `$json.type === "image"` → "Image Message"  
     - Condition 2: `$json.type === "audio"` → "Audio Message"  
     - Condition 3: `$json.type === "text"` → "Text Message"  

4. **For Image Messages:**  
   - Add HTTP Request node "Download Image"  
     - URL: `{{$json.url}}`  
     - Authentication: Facebook Graph API credential (Page Access Token)  
   - Add OpenAI Vision node "Analyze image"  
     - Model: `gpt-4o`  
     - Operation: Analyze image (base64 input)  
   - Add Set node "Edit Fields3"  
     - Assign `user_prompt` = `{{$json.text}}` (image description)  
     - Include all other fields  

5. **For Audio Messages:**  
   - Add HTTP Request node "Download Audio"  
     - URL: `{{$json.url}}`  
     - Authentication: Facebook Graph API credential  
   - Add OpenAI Audio Transcription node "Audio Transcriber"  
     - Operation: transcribe audio  
   - Add Set node "Edit Fields2"  
     - Assign `user_prompt` = `{{$json.text}}` (transcribed text)  
     - Include other fields as needed  

6. **For Text Messages:**  
   - Add Set node "Edit Fields1"  
     - Assign `user_prompt` = `{{$json.text}}`  
     - Optionally truncate long texts  

7. **Add Langchain Memory Node "Simple Memory"**  
   - Session Key: `={{ $('Webhook1').item.json.body.entry[0].messaging[0].sender.id }}`  
   - Session Id Type: customKey  

8. **Add Langchain LM Chat OpenAI Node "OpenAI Chat Model"**  
   - Model: `gpt-4.1-mini`  
   - Credentials: OpenAI API key  

9. **Add Langchain Tool Nodes:**  
   - Calculator (default config)  
   - Wikipedia (default config)  

10. **Add Langchain Agent Node "AI Agent Messenger"**  
    - Text input: `{{$json.user_prompt}}`  
    - System Message:  
      ```
      Tu es un assistant spécialisé.
      - Si la demande nécessite uniquement des calculs mathématiques, financiers ou statistiques : utilise toujours l’outil Calculatrice.
      - Si la demande nécessite une recherche d’information externe (histoire, biographie, faits généraux) : utilise l’outil Wikipédia.
      - Ne mélange pas les deux outils sauf si c’est absolument nécessaire.
      - Lorsque la question contient un montant, un pourcentage, un taux ou une durée : c’est un calcul → utilise uniquement l’outil Calculatrice.
      ```  
    - Connect AI tools (Calculator, Wikipedia), AI memory (Simple Memory), AI language model (OpenAI Chat Model)  

11. **Add Code Node "Code" (Post-processing Bold)**  
    - JavaScript to convert markdown bold (`**bold**`, `*bold*`) to Unicode bold characters in field `output` and remove asterisks  

12. **Add Code Node "Code1" (Build Messenger Payload)**  
    - Extract PSID from Webhook1 payload  
    - Construct JSON with:  
      ```json
      {
        "messaging_type": "RESPONSE",
        "recipient": { "id": "PSID as string" },
        "message": { "text": "output text" }
      }
      ```  

13. **Add HTTP Request Node "Send Response"**  
    - URL: `https://graph.facebook.com/v17.0/me/messages?access_token=PAGE_TOKEN` (replace with credential)  
    - Method: POST  
    - Content-Type: application/json  
    - Body: Raw JSON from Code1 output  

14. **Connect Nodes According to Logic:**  
    - Webhook1 → Identify Message Type → Identify and ReRoute Message Types  
    - Image Route: → Download Image → Analyze image → Edit Fields3 → AI Agent Messenger  
    - Audio Route: → Download Audio → Audio Transcriber → Edit Fields2 → AI Agent Messenger  
    - Text Route: → Edit Fields1 → AI Agent Messenger  
    - AI Agent Messenger → Code → Code1 → Send Response  

15. **Credentials Setup:**  
    - Create Facebook Graph API credential with Page Access Token for media downloads and message sending  
    - Create OpenAI credential with API key  

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                     |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Automate Messenger in 6 Steps: Setup guidance, credential connections, webhook setup, personalization | Sticky Note1 (workflow intro)                                     |
| Video Tutorial for this workflow                                                                      | [YouTube Link](https://youtu.be/yHNPBDNgk88)                      |
| Contact for customization help                                                                        | [LinkedIn](https://www.linkedin.com/in/st%C3%A9phane-bordas-3439b4179/) |

---

**Disclaimer:**  
The provided content derives exclusively from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected material. All processed data is legal and public.

---