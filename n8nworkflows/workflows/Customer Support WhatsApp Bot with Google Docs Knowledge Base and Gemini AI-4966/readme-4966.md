Customer Support WhatsApp Bot with Google Docs Knowledge Base and Gemini AI

https://n8nworkflows.xyz/workflows/customer-support-whatsapp-bot-with-google-docs-knowledge-base-and-gemini-ai-4966


# Customer Support WhatsApp Bot with Google Docs Knowledge Base and Gemini AI

### 1. Workflow Overview

This workflow implements a **Customer Support WhatsApp Bot** integrated with a **Google Docs Knowledge Base** and powered by **Gemini AI** for natural language understanding and response generation. It is designed to answer customer inquiries received via WhatsApp by leveraging internal business knowledge stored in a Google Doc, without requiring explicit AI training. The bot can respond instantly with relevant, clean answers and log conversations into Google Sheets.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives WhatsApp messages via the WhatsApp Business Cloud API trigger.
- **1.2 Knowledge Retrieval:** Reads the company knowledge from a specified Google Doc.
- **1.3 Prompt Preparation:** Constructs a dynamic prompt incorporating today’s date, the Google Doc content, and the incoming WhatsApp message.
- **1.4 AI Processing:** Uses Gemini AI (via Langchain integration) with context memory to generate the answer.
- **1.5 Answer Cleaning:** Cleans and formats the AI-generated answer to remove markdown and unwanted text.
- **1.6 Logging & Validation:** Logs the interaction in Google Sheets and checks if the conversation is within a 24-hour window.
- **1.7 Response Delivery:** Sends the AI-generated answer or a pre-approved WhatsApp template message to reopen conversations outside the 24-hour window.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures incoming WhatsApp messages using the WhatsApp Business Cloud API trigger node.
- **Nodes Involved:** `when message received`
  
- **Node Details:**

  - **when message received**
    - Type: WhatsApp Trigger (Webhook)
    - Role: Listens for new WhatsApp messages (specifically message updates)
    - Configuration:
      - Webhook ID configured for WhatsApp Cloud API
      - Credentials: WhatsApp OAuth2 account connected
      - Listens to "messages" update events
    - Input: External WhatsApp message webhook
    - Output: JSON containing message details including sender ID, timestamp, and message body
    - Edge Cases:
      - Webhook connectivity issues
      - Invalid or malformed WhatsApp messages
      - OAuth token expiration or permission errors

#### 2.2 Knowledge Retrieval

- **Overview:** Fetches the latest content from a Google Doc that serves as the knowledge base for answering questions.
- **Nodes Involved:** `company's knowledge`
  
- **Node Details:**

  - **company's knowledge**
    - Type: Google Docs
    - Role: Retrieves full text content from specified Google Doc by ID
    - Configuration:
      - Operation: Get document content
      - Document URL/ID: Set to the business knowledge Google Doc
    - Input: Triggered after WhatsApp message reception
    - Output: Google Doc JSON response containing text content broken by paragraphs and text runs
    - Edge Cases:
      - Google API authentication failures
      - Document permission denied
      - Empty or malformed document content
    
#### 2.3 Prompt Preparation

- **Overview:** Constructs a unified prompt string combining today’s date, the Google Doc content (flattened into plain text), and the user’s WhatsApp question.
- **Nodes Involved:** `Prepare Prompt`
  
- **Node Details:**

  - **Prepare Prompt**
    - Type: AI Transform (JavaScript code)
    - Role: Extracts and formats data into a prompt for AI consumption
    - Configuration:
      - Uses JavaScript to:
        - Get current date formatted as "Month Day, Year"
        - Extract and join Google Doc text from content array into plain text
        - Extract user question from WhatsApp message body
        - Format finalPrompt string with date, doc text, and user question
      - Returns a single field `finalPrompt`
    - Input: Output from `company's knowledge` and `when message received`
    - Output: JSON with `finalPrompt` string
    - Edge Cases:
      - Missing or malformed input JSON fields
      - Date formatting errors
      - Google Doc content parsing issues

#### 2.4 AI Processing

- **Overview:** Sends the prepared prompt to Gemini AI agent (via Langchain) with session memory to generate a relevant answer.
- **Nodes Involved:** `Simple Memory`, `Google Gemini Chat Model`, `AI Agent`
  
- **Node Details:**

  - **Simple Memory**
    - Type: Langchain Memory Buffer Window
    - Role: Maintains conversation context keyed by WhatsApp user ID to provide session continuity
    - Configuration:
      - Session key extracted from WhatsApp sender ID (`wa_id`)
      - Uses custom key session ID type
    - Input: Connected to incoming WhatsApp messages
    - Output: Session memory context for AI Agent
    - Edge Cases:
      - Memory overflow or data corruption
      - Missing or invalid session key

  - **Google Gemini Chat Model**
    - Type: Langchain Google Gemini Chat Model
    - Role: Provides the AI language model interface for chat completion
    - Configuration:
      - Model: `models/gemini-2.5-flash-preview-04-17-thinking`
    - Input: Provides the language model to the AI Agent
    - Output: AI-generated chat completions
    - Edge Cases:
      - API rate limits or timeouts
      - Model unavailability or errors

  - **AI Agent**
    - Type: Langchain Agent Node
    - Role: Executes the language model prompt and returns the AI answer
    - Configuration:
      - Uses the `finalPrompt` field as input text
      - System message instructs no preamble and no document references in the answer for natural replies
      - Output parser enabled to cleanly extract the answer
      - Max retry attempts: 5
    - Input: Receives prompt from `Prepare Prompt`, AI model from `Google Gemini Chat Model`, and memory from `Simple Memory`
    - Output: AI-generated raw answer text
    - Edge Cases:
      - AI response failures or malformed output
      - Retry on AI service failures
      - Parsing errors on response

#### 2.5 Answer Cleaning

- **Overview:** Cleans AI-generated answers by removing markdown formatting, converting hyperlinks, and eliminating unwanted preamble text.
- **Nodes Involved:** `cleanAnswer`
  
- **Node Details:**

  - **cleanAnswer**
    - Type: Code Node (JavaScript)
    - Role: Post-processes AI output to improve readability before sending to user
    - Configuration:
      - Removes markdown bold/italic/strike markers (`*`, `_`, `~`)
      - Converts markdown links `[text](url)` into `text url`
      - Collapses multiple blank lines to two newlines
      - Removes phrases like "based on the document you provided" from start of text
    - Input: AI answer output from `AI Agent`
    - Output: Cleaned answer string in `answer` field
    - Edge Cases:
      - Unexpected markdown or formatting variants
      - Empty or missing AI answer string

#### 2.6 Logging & Validation

- **Overview:** Logs the interaction into Google Sheets and checks whether the user’s last message was sent within the past 24 hours to comply with WhatsApp messaging policies.
- **Nodes Involved:** `Date & Time`, `Google Sheets`, `24-hour window check`, `If`
  
- **Node Details:**

  - **Date & Time**
    - Type: DateTime Node
    - Role: Provides current timestamp to log with message
    - Configuration: Default
    - Input: From AI Agent
    - Output: Current date/time for logging

  - **Google Sheets**
    - Type: Google Sheets Node
    - Role: Appends or updates a row in a Google Sheet to record user message, AI response, user ID, and timestamp
    - Configuration:
      - Operation: Append or Update
      - Sheet: Configured with specific Google Sheet ID and sheet name
      - Columns mapped: User (WhatsApp sender), Message (WhatsApp text), Response (AI output), Timestamp (current date/time)
      - Matching column: Timestamp (to avoid duplicates)
    - Input: From `Date & Time` and AI Agent
    - Output: Confirmation of logging
    - Edge Cases:
      - Google Sheets API errors
      - Permission issues
      - Data type mismatches
    
  - **24-hour window check**
    - Type: Code Node (JavaScript)
    - Role: Checks if the last WhatsApp message timestamp is within the last 24 hours
    - Configuration:
      - Converts WhatsApp message timestamp (seconds) to milliseconds
      - Compares with current time to set boolean `withinWindow`
    - Input: Output from `Google Sheets`
    - Output: JSON with `withinWindow` boolean, answer and userId fields for subsequent logic
    - Edge Cases:
      - Missing or malformed timestamp
      - Timezone discrepancies

  - **If**
    - Type: If Node (Boolean condition)
    - Role: Routes workflow based on result of 24-hour window check
    - Configuration:
      - Condition: `withinWindow === true`
      - True branch: Continue to send AI answer
      - False branch: Send pre-approved WhatsApp template message to reopen conversation
    - Input: Output from `24-hour window check`
    - Output: Routes to two different WhatsApp send nodes
    - Edge Cases:
      - Condition evaluation errors
      - Missing input fields

#### 2.7 Response Delivery

- **Overview:** Sends either the AI-generated answer or a pre-approved WhatsApp template message based on the 24-hour window status.
- **Nodes Involved:** `Send AI Agent's Answer`, `Send Pre-approved Template Message to Reopen the Conversation`
  
- **Node Details:**

  - **Send AI Agent's Answer**
    - Type: WhatsApp Node (Send Message)
    - Role: Sends the cleaned AI answer back to the user on WhatsApp
    - Configuration:
      - Operation: Send text message
      - Phone Number ID: Configured WhatsApp Business Cloud phone number
      - Recipient Number: Extracted dynamically from incoming WhatsApp message contact `wa_id`
      - Text Body: Uses cleaned `answer` field from `cleanAnswer`
    - Input: From `cleanAnswer`
    - Output: Confirmation of message send
    - Edge Cases:
      - WhatsApp API rate limits or failures
      - Invalid recipient number
      - Message content formatting issues

  - **Send Pre-approved Template Message to Reopen the Conversation**
    - Type: WhatsApp Node (Send Template)
    - Role: Sends a pre-approved WhatsApp template message (e.g., "hello_world") to reopen conversations beyond 24-hour window
    - Configuration:
      - Template: `hello_world|en_US`
      - Phone Number ID: Configured WhatsApp Business Cloud phone number
      - Recipient Number: Extracted dynamically from incoming WhatsApp message contact `wa_id`
    - Input: From `If` node (false branch)
    - Output: Confirmation of template message send
    - Edge Cases:
      - Template approval required by WhatsApp
      - API errors
      - Incorrect template formatting

---

### 3. Summary Table

| Node Name                                     | Node Type                          | Functional Role                                   | Input Node(s)                  | Output Node(s)                                         | Sticky Note                                                                                                                                                                                                                                                                                                                                                   |
|-----------------------------------------------|----------------------------------|--------------------------------------------------|-------------------------------|--------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                                   | Sticky Note                      | Project overview and setup instructions           |                               |                                                        | **Smart WhatsApp AI Assistant Using Google Docs** — Answers WhatsApp questions instantly using internal document knowledge. No training needed — just update the doc. Setup steps and contact info included.                                                                                                                                                    |
| when message received                         | WhatsApp Trigger                 | Receives incoming WhatsApp messages                |                               | company's knowledge                                    |                                                                                                                                                                                                                                                                                                                                                               |
| company's knowledge                           | Google Docs                     | Retrieves Google Docs knowledge base content       | when message received          | Prepare Prompt                                        |                                                                                                                                                                                                                                                                                                                                                               |
| Prepare Prompt                               | AI Transform (Code)             | Builds AI prompt string from date, doc text, and message | company's knowledge           | AI Agent                                             |                                                                                                                                                                                                                                                                                                                                                               |
| Simple Memory                                | Langchain Memory Buffer Window  | Maintains conversational context per WhatsApp user |                               | AI Agent (ai_memory)                                  |                                                                                                                                                                                                                                                                                                                                                               |
| Google Gemini Chat Model                      | Langchain AI Model              | Language model interface for Gemini AI             |                               | AI Agent (ai_languageModel)                           |                                                                                                                                                                                                                                                                                                                                                               |
| AI Agent                                     | Langchain Agent                 | Processes AI prompt and generates answer           | Prepare Prompt, Simple Memory, Google Gemini Chat Model | Date & Time                                            |                                                                                                                                                                                                                                                                                                                                                               |
| Date & Time                                  | DateTime                       | Provides current timestamp for logging             | AI Agent                      | Google Sheets                                        |                                                                                                                                                                                                                                                                                                                                                               |
| Google Sheets                                | Google Sheets                   | Logs user message, AI response, and timestamp      | Date & Time                   | 24-hour window check                                 |                                                                                                                                                                                                                                                                                                                                                               |
| 24-hour window check                         | Code (JavaScript)               | Checks if last message is within 24 hours          | Google Sheets                 | If                                                   |                                                                                                                                                                                                                                                                                                                                                               |
| If                                           | If Node                        | Routes based on 24-hour window check                | 24-hour window check          | cleanAnswer (true branch), Send Pre-approved Template Message to Reopen the Conversation (false branch) |                                                                                                                                                                                                                                                                                                                                                               |
| cleanAnswer                                  | Code (JavaScript)               | Cleans AI answer text for sending                   | If (true branch)              | Send AI Agent's Answer                               |                                                                                                                                                                                                                                                                                                                                                               |
| Send AI Agent's Answer                       | WhatsApp Node (Send Message)    | Sends AI-generated answer back to user              | cleanAnswer                  |                                                        |                                                                                                                                                                                                                                                                                                                                                               |
| Send Pre-approved Template Message to Reopen the Conversation | WhatsApp Node (Send Template)   | Sends template message to reopen conversation outside 24h window | If (false branch)             |                                                        |                                                                                                                                                                                                                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the WhatsApp Trigger Node**
   - Type: WhatsApp Trigger
   - Configure webhook with WhatsApp Cloud API credentials (OAuth2)
   - Set to listen for "messages" update events
   - Position appropriately

2. **Add Google Docs Node for Knowledge Retrieval**
   - Type: Google Docs
   - Operation: Get document content
   - Enter your Google Doc ID or URL containing business knowledge
   - Connect output from WhatsApp Trigger node to this node

3. **Add AI Transform Node to Prepare Prompt**
   - Type: AI Transform (Code)
   - Paste JavaScript code to:
     - Get today’s date formatted as "Month Day, Year"
     - Extract Google Doc text content and join into plain text
     - Extract WhatsApp message body
     - Construct finalPrompt string combining above elements
   - Connect output from Google Docs node to this node (ensure also access to WhatsApp message data)

4. **Add Simple Memory Node**
   - Type: Langchain Memory Buffer Window
   - Set session key to WhatsApp sender ID: `{{$json["contacts"][0]["wa_id"]}}`
   - Use custom key session ID type for maintaining per-user context

5. **Add Google Gemini Chat Model Node**
   - Type: Langchain Google Gemini Chat Model
   - Set model to `models/gemini-2.5-flash-preview-04-17-thinking`

6. **Add AI Agent Node**
   - Type: Langchain Agent
   - Configure:
     - Text input: `={{ $json.finalPrompt }}`
     - System message to instruct no preamble, no document references, no date mentions (unless asked)
     - Enable output parser
     - Set max retry attempts to 5
   - Connect inputs:
     - AI prompt from `Prepare Prompt`
     - Memory from `Simple Memory` (ai_memory)
     - Language model from `Google Gemini Chat Model` (ai_languageModel)

7. **Add Date & Time Node**
   - Type: DateTime
   - Use default configuration
   - Connect output from AI Agent node

8. **Add Google Sheets Node for Logging**
   - Type: Google Sheets
   - Operation: Append or Update row
   - Configure:
     - Spreadsheet ID and sheet name for chat logs
     - Map columns:
       - User: WhatsApp sender ID (`messages[0].from`)
       - Message: WhatsApp message body (`messages[0].text.body`)
       - Response: AI Agent output
       - Timestamp: Current date/time from Date & Time node
     - Use Timestamp column as matching column to avoid duplicates
   - Connect output from Date & Time node

9. **Add 24-hour Window Check Node (Code)**
   - Type: Code Node
   - Paste JavaScript code to:
     - Extract WhatsApp message timestamp (convert seconds to ms)
     - Compare with current time to determine boolean `withinWindow`
     - Pass this boolean along with answer and userId
   - Connect output from Google Sheets node

10. **Add If Node for Routing**
    - Type: If
    - Condition: Check if `withinWindow` equals true (boolean)
    - Connect output from 24-hour window check node
    - True branch: route to answer sending flow
    - False branch: route to template message sending flow

11. **Add cleanAnswer Node (Code)**
    - Type: Code Node
    - Paste JavaScript code to:
      - Remove markdown formatting (`*`, `_`, `~`)
      - Convert markdown links to plain text with URL
      - Collapse multiple blank lines
      - Remove phrases like "based on the document you provided"
    - Connect true branch output from If node

12. **Add Send AI Agent's Answer Node**
    - Type: WhatsApp Node (Send Message)
    - Configure:
      - Operation: Send text message
      - Phone number ID: Your WhatsApp Business Cloud phone number
      - Recipient number: dynamically from incoming `wa_id`
      - Text body: cleaned answer text from `cleanAnswer`
    - Connect output from `cleanAnswer`

13. **Add Send Pre-approved Template Message Node**
    - Type: WhatsApp Node (Send Template)
    - Configure:
      - Template: `hello_world|en_US`
      - Phone number ID: Your WhatsApp Business Cloud phone number
      - Recipient number: dynamically from incoming `wa_id`
    - Connect false branch output from If node

14. **Add Sticky Note (Optional)**
    - Add a sticky note summarizing the workflow purpose, prerequisites, and contact info for assistance

15. **Set Credentials**
    - WhatsApp OAuth2 credentials for WhatsApp nodes and trigger
    - Google OAuth credentials for Google Docs and Google Sheets nodes
    - Langchain node credentials for Gemini AI or OpenAI access

16. **Test the Workflow**
    - Publish and test by sending a WhatsApp message to the connected number
    - Verify AI response, logging, and 24-hour window behaviors

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                                                                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Workflow answers WhatsApp questions instantly using a live Google Doc as knowledge base; no AI training needed beyond updating the doc.                                                                                                                                 | Project overview sticky note content                                                                                        |
| Setup requires WhatsApp Business Cloud API, Google OAuth with access to Google Docs and Sheets, and OpenAI or Gemini API keys.                                                                                                                                           | Project prerequisites                                                                                                       |
| Video guide for setting up PostgreSQL memory (optional enhancement): https://www.youtube.com/watch?v=JjBofKJnYIU                                                                                                                                                        | Memory setup reference                                                                                                      |
| Contact for setup, branding, customization services including WhatsApp API, Google OAuth, AI model tuning, and reporting:                                                                                                                                                | Email: tharwat.elsayed2000@gmail.com<br>WhatsApp: +201061803236                                                             |
| AI prompt system message removes references to documents or dates unless explicitly requested, ensuring natural human-like answers.                                                                                                                                      | AI Agent node configuration                                                                                                 |

---

This comprehensive documentation enables both advanced users and automation agents to fully understand, reproduce, or extend the Customer Support WhatsApp Bot workflow integrating Google Docs knowledge and Gemini AI.