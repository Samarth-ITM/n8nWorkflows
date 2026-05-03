Automated Multilingual Gmail Draft Reply with OpenAI GPT-4o

https://n8nworkflows.xyz/workflows/automated-multilingual-gmail-draft-reply-with-openai-gpt-4o-4870


# Automated Multilingual Gmail Draft Reply with OpenAI GPT-4o

### 1. Workflow Overview

This workflow automates the drafting of multilingual Gmail replies using OpenAI GPT-4o. It is primarily designed for users who receive high volumes of emails in multiple languages, such as customer support teams or multilingual communication managers. The workflow listens for incoming Gmail messages with a specific label, assesses whether a reply is needed, detects the language, and then generates an appropriate draft reply using AI. The reply is formatted in HTML and saved as a draft in the original Gmail thread for review.

The workflow’s logic is divided into the following blocks:

- **1.1 Input Reception:** Listens for new Gmail messages with a specific label.
- **1.2 Message Assessment:** Uses OpenAI GPT-4o to determine if a reply is needed and detects the message language.
- **1.3 Language-Based Routing:** Routes the workflow based on detected language (English or Japanese).
- **1.4 AI Draft Generation:** Generates a context-aware draft reply tailored to the recipient's language using specialized AI prompts.
- **1.5 Draft Creation in Gmail:** Converts the AI-generated text to HTML and saves it as a draft reply in the original email thread.
- **1.6 Workflow Meta and Documentation:** Sticky notes provide guidance and explanations about each stage.


---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block monitors the user’s Gmail inbox for new incoming messages filtered by a specific label (e.g., "Inquiry"). It triggers the workflow every minute to check for new emails.

**Nodes Involved:**  
- Gmail trigger  
- Sticky Note (Gmail Trigger)  

**Node Details:**

- **Gmail trigger**  
  - Type: Gmail Trigger  
  - Role: Watches Gmail inbox for new emails with a filter.  
  - Configuration:  
    - Polling interval: Every minute  
    - Filter query: Empty (but intended to be configured with a Gmail label such as "Inquiry")  
    - Credentials: OAuth2 for Gmail  
  - Input: None (trigger node)  
  - Output: Emits new email data including headers, subject, text, and thread ID  
  - Potential failures: Authentication errors, rate limiting by Gmail API, empty or incorrect label filters causing missed emails  
  - Version: v1  

- **Sticky Note (Gmail Trigger)**  
  - Provides user instructions on setting up Gmail label filters and adjusting polling intervals.

---

#### 2.2 Message Assessment

**Overview:**  
This block assesses the incoming email’s content to determine whether a reply is necessary and identifies the email’s language (English, Japanese, or Other). It uses OpenAI GPT-4o for natural language understanding and outputs structured JSON data.

**Nodes Involved:**  
- Assess if a message needs a reply and identify the language  
- OpenAI GPT-4o  
- Parser Result  
- If Needs Reply  
- Sticky Note1  

**Node Details:**

- **Assess if a message needs a reply and identify the language**  
  - Type: Chain LLM (LangChain)  
  - Role: Sends the email subject and HTML body text to OpenAI GPT-4o with a prompt instructing it to detect language and reply necessity.  
  - Configuration:  
    - Input text includes subject and HTML email body  
    - Prompt asks for JSON output with fields "message_language" and "needsReply"  
    - Instructions to ignore marketing emails (no reply needed)  
  - Input: Email data from Gmail trigger  
  - Output: JSON string with language and reply flag  
  - Potential failures: GPT response format errors, incorrect or ambiguous language detection, API latency/timeouts  
  - Version: v1.3  

- **OpenAI GPT-4o**  
  - Type: Language Model Chat (OpenAI GPT-4o)  
  - Role: Provides the underlying AI model for the Chain LLM node above  
  - Configuration:  
    - Model: gpt-4o  
    - Temperature: 0 (deterministic output)  
    - Response format: JSON object for parsing  
  - Credentials: OpenAI API key  
  - Potential failures: Auth errors, rate limits, model unavailability  
  - Version: v1  

- **Parser Result**  
  - Type: Output Parser Structured (LangChain)  
  - Role: Parses the JSON response from the assessment LLM node to extract "needsReply" (boolean) and "message_language" (string) fields in a structured format.  
  - Configuration: JSON schema requiring "needsReply" and "message_language"  
  - Input: AI JSON response  
  - Output: Parsed JSON object for downstream nodes  
  - Potential failures: JSON parse errors if AI response malformed  
  - Version: v1  

- **If Needs Reply**  
  - Type: If node  
  - Role: Checks if the parsed field "needsReply" is true to decide whether to continue generating a draft reply or end workflow.  
  - Configuration: Condition evaluates $json.needsReply == true  
  - Input: Parsed result from previous node  
  - Output: True branch leads to language-based routing; false branch ends or does nothing  
  - Potential failures: Expression evaluation errors if field missing  

- **Sticky Note1**  
  - Explains the use of OpenAI GPT-4o and JSON parsing to determine reply necessity and language detection.

---

#### 2.3 Language-Based Routing

**Overview:**  
This block routes the workflow to different reply generation branches depending on the detected language of the email (Japanese or English). It uses a switch node with condition-based outputs.

**Nodes Involved:**  
- Switch based on email language  
- Sticky Note3  

**Node Details:**

- **Switch based on email language**  
  - Type: Switch node  
  - Role: Routes execution based on the value of "message_language" from parsed JSON  
  - Configuration:  
    - Output "Japanese version" if message_language equals "Japanese"  
    - Output "English version" if message_language equals "English"  
    - Fallback output for any other language (not explicitly configured for generation, so likely no reply)  
  - Input: Parsed JSON from "If Needs Reply" node  
  - Output: Two branches leading to respective AI draft generation nodes  
  - Potential failures: Missing or unexpected language values; fallback branch may cause silent skips  

- **Sticky Note3**  
  - Describes the generation of replies based on language-specific rules and prompts, including tone, greetings, and conventions.

---

#### 2.4 AI Draft Generation

**Overview:**  
This block generates the draft reply email text in the appropriate language using OpenAI GPT-4o with customized prompts to suit Japanese or English professional email etiquette.

**Nodes Involved:**  
- Generate email for a Japanese client  
- Generate email reply for an English client  
- OpenAI GPT-4o - for Japanese  
- OpenAI GPT 4o - For English  

**Node Details:**

- **Generate email for a Japanese client**  
  - Type: LangChain Chain LLM  
  - Role: Generates a Japanese draft reply using a prompt with specific instructions for polite business email style (desu-masu), including structured greetings and closings.  
  - Configuration:  
    - Input includes original subject and HTML message  
    - Prompt demands concise, professional Japanese response with placeholders as needed  
    - Includes instructions for dual responses for yes-no questions separated by a delimiter  
  - Input: Email data from Gmail trigger (via switch path)  
  - Output: Draft reply text  
  - Version: v1.4  

- **Generate email reply for an English client**  
  - Type: LangChain Chain LLM  
  - Role: Generates an English draft reply with professional but business casual tone, starting with “Dear,” and ending with “Best regards,”.  
  - Configuration:  
    - Similar input and dual response format as Japanese, but tailored to English email conventions  
  - Input: Email data from Gmail trigger (via switch path)  
  - Output: Draft reply text  
  - Version: v1.4  

- **OpenAI GPT-4o - for Japanese**  
  - Type: LLM Chat OpenAI GPT-4o  
  - Role: Provides the AI model for the Japanese draft generation node  
  - Credentials: OpenAI API  
  - Version: v1  

- **OpenAI GPT 4o - For English**  
  - Type: LLM Chat OpenAI GPT-4o  
  - Role: Provides the AI model for the English draft generation node  
  - Credentials: OpenAI API  
  - Version: v1  

- Potential failures (all nodes): API errors, prompt misinterpretation, incorrect or incomplete draft outputs, API rate limits or timeouts.

---

#### 2.5 Draft Creation in Gmail

**Overview:**  
This block converts the AI-generated reply text to HTML format and creates a draft reply in the original Gmail thread, linking it to the initial email for context.

**Nodes Involved:**  
- Gmail - Create Draft  
- Sticky Note4  

**Node Details:**

- **Gmail - Create Draft**  
  - Type: Gmail node  
  - Role: Creates an HTML draft reply in Gmail within the original thread.  
  - Configuration:  
    - Converts plain text reply to HTML by replacing line breaks with `<br />`  
    - Sets draft recipient as the sender of the original email  
    - Uses original thread ID to keep reply in the conversation  
    - Subject prefixed with “Re:” and original subject included  
    - Uses Gmail OAuth2 credentials  
  - Input: Draft reply text from either Japanese or English generation node  
  - Output: Confirmation of draft creation  
  - Potential failures: Gmail API quota limits, OAuth token expiry, malformed HTML, thread ID missing or invalid  

- **Sticky Note4**  
  - Explains that draft integration converts text to HTML and places the draft as a reply in the Gmail thread.

---

#### 2.6 Workflow Meta and Documentation

**Nodes Involved:**  
- Sticky Note5  

- **Sticky Note5**  
  - Provides a comprehensive overview of the workflow’s purpose, target users, problem addressed, detailed steps, setup instructions, and customization tips.

---

### 3. Summary Table

| Node Name                            | Node Type                   | Functional Role                           | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                                   |
|------------------------------------|-----------------------------|-----------------------------------------|---------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Gmail trigger                      | Gmail Trigger               | Input reception: monitor incoming emails| None                            | Assess if a message needs a reply and identify the language | Sticky Note: Gmail Trigger - explains label filtering and polling interval                                                     |
| Assess if a message needs a reply and identify the language | Chain LLM (LangChain)       | Message assessment: reply necessity and language detection | Gmail trigger                  | If Needs Reply                   | Sticky Note1: Explains use of OpenAI GPT-4o and JSON parser for assessment                                                     |
| OpenAI GPT-4o                     | LLM Chat OpenAI GPT-4o      | AI model backend for assessment          | Assess if a message needs a reply and identify the language | Assess if a message needs a reply and identify the language |                                                                                                                               |
| Parser Result                     | Output Parser Structured    | Parses AI JSON output                     | Assess if a message needs a reply and identify the language | If Needs Reply                   |                                                                                                                               |
| If Needs Reply                   | If                          | Checks if reply is needed                 | Parser Result                  | Switch based on email language   |                                                                                                                               |
| Switch based on email language    | Switch                      | Routes based on detected language        | If Needs Reply                 | Generate email for a Japanese client, Generate email reply for an English client | Sticky Note3: Explains language-based reply generation rules and conventions                                                   |
| Generate email for a Japanese client | Chain LLM (LangChain)       | Generates Japanese draft reply            | Switch based on email language | Gmail - Create Draft            |                                                                                                                               |
| OpenAI GPT-4o - for Japanese      | LLM Chat OpenAI GPT-4o      | AI model backend for Japanese generation | Generate email for a Japanese client | Generate email for a Japanese client |                                                                                                                               |
| Generate email reply for an English client | Chain LLM (LangChain)       | Generates English draft reply             | Switch based on email language | Gmail - Create Draft            |                                                                                                                               |
| OpenAI GPT 4o - For English       | LLM Chat OpenAI GPT-4o      | AI model backend for English generation   | Generate email reply for an English client | Generate email reply for an English client |                                                                                                                               |
| Gmail - Create Draft             | Gmail                       | Creates draft reply in Gmail thread       | Generate email for a Japanese client, Generate email reply for an English client | None                            | Sticky Note4: Explains draft creation and HTML formatting                                                                      |
| Sticky Note                       | Sticky Note                 | Documentation                            | None                            | None                            | Sticky Note: Gmail Trigger - label filtering and polling instructions                                                          |
| Sticky Note1                      | Sticky Note                 | Documentation                            | None                            | None                            | Sticky Note1: Message assessment explanation                                                                                  |
| Sticky Note3                      | Sticky Note                 | Documentation                            | None                            | None                            | Sticky Note3: Draft generation instructions per language                                                                       |
| Sticky Note4                      | Sticky Note                 | Documentation                            | None                            | None                            | Sticky Note4: Draft creation explanation                                                                                      |
| Sticky Note5                      | Sticky Note                 | Documentation                            | None                            | None                            | Sticky Note5: Detailed workflow overview, target audience, problem statement, setup and customization notes                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Credentials: Connect Gmail account via OAuth2  
   - Configure polling interval: Every minute (adjust as needed)  
   - Set Gmail label filter (e.g., "Inquiry") in query to watch specific emails  

2. **Add Chain LLM node "Assess if a message needs a reply and identify the language":**  
   - Type: LangChain Chain LLM  
   - Connect input from Gmail Trigger node  
   - Configure prompt:  
     ```
     Subject: {{ $json.subject }}
     Message:
     {{ $json.textAsHtml }}
     
     Your task is to:
     - identify the language of the message and set the value to field "message_language": "English", "Japanese", or "Other"
     - assess if the message requires a response and set the value to field "needsReply": true if yes, false otherwise.
     Return in JSON format.
     Marketing emails don't require a response.
     ```  
   - Version: 1.3  

3. **Add OpenAI GPT-4o node:**  
   - Type: LLM Chat OpenAI GPT-4o  
   - Connect as AI model for the Chain LLM above  
   - Credentials: Use OpenAI API key  
   - Set model to "gpt-4o"  
   - Set temperature to 0 and response format as JSON object  

4. **Add Output Parser Structured node "Parser Result":**  
   - Configure JSON schema requiring:  
     ```json
     {
       "type": "array",
       "items": {
         "type": "object",
         "properties": {
           "needsReply": { "type": "boolean" },
           "message_language": { "type": "string" }
         },
         "required": ["needsReply", "message_language"]
       }
     }
     ```  
   - Connect input from Chain LLM output  

5. **Add If node "If Needs Reply":**  
   - Condition: `$json.needsReply == true`  
   - Connect input from Parser Result node  
   - True output leads to language-based routing, false output ends workflow or stops further processing  

6. **Add Switch node "Switch based on email language":**  
   - Connect input from If Needs Reply node (true branch)  
   - Configure output rules:  
     - Output "Japanese version" if `$json.message_language == "Japanese"`  
     - Output "English version" if `$json.message_language == "English"`  
     - Optional fallback output (no action)  

7. **Create Chain LLM node "Generate email for a Japanese client":**  
   - Connect to "Japanese version" output of Switch node  
   - Prompt example:  
     ```
     Subject: {{ $('Gmail trigger').item.json.subject }}
     Message: {{ $('Gmail trigger').item.json.textAsHtml }}
     
     You're a helpful personal assistant and your task is to draft replies on my behalf to my incoming emails. Whenever I provide some text from an email, return an appropriate draft reply for it and nothing else.
     Ensure that the reply is suitable for a professional email setting and addresses the topic in a clear, structured, and detailed manner.
     Do not make things up.
     
     Detailed instructions:
     - Be concise and maintain a business tone (desu-masu).
     - Follow the following content 
     [INSERT CLIENT NAME] 様
     お世話になっております。
     株式会社 [INSERT YOUR COMPANY NAME] の[INSERT YOUR NAME]です。
     
     BODY (main content)
     
     引き続き、何卒どうぞ宜しくお願いいたします。
     
     - When replying to yes-no questions, draft 2 responses: one affirmative and one negative separated by " - - - - - - - OR - - - - - - - "
     - If you don't know an answer, you can leave placeholders like "[YOUR_ANSWER_HERE]".
     - Don't use any special formatting, only plain text.
     - Reply in the same language as the inbound email.
     ```  
   - Connect AI model node (OpenAI GPT-4o - for Japanese) with correct credentials  

8. **Create Chain LLM node "Generate email reply for an English client":**  
   - Connect to "English version" output of Switch node  
   - Prompt example:  
     ```
     Subject: {{ $('Gmail trigger').item.json.subject }}
     Message: {{ $('Gmail trigger').item.json.textAsHtml }}
     
     You're a helpful personal assistant and your task is to draft replies on my behalf to my incoming emails. Whenever I provide some text from an email, return an appropriate draft reply for it and nothing else.
     Ensure that the reply is suitable for a professional email setting and addresses the topic in a clear, structured, and detailed manner.
     Do not make things up.
     
     Detailed instructions:
     - Be concise and maintain a business casual tone.
     - Start with "Dear,", and end with "Best regards,"
     - When replying to yes-no questions, draft 2 responses: one affirmative and one negative separated by " - - - - - - - OR - - - - - - - "
     - If you don't know an answer, you can leave placeholders like "[YOUR_ANSWER_HERE]".
     - Don't use any special formatting, only plain text.
     - Reply in the same language as the inbound email.
     ```  
   - Connect AI model node (OpenAI GPT 4o - For English) with credentials  

9. **Add Gmail node "Gmail - Create Draft":**  
   - Connect inputs from both Japanese and English draft generation nodes  
   - Configure:  
     - Message content: Replace line breaks `\n` with `<br />\n` for HTML formatting  
     - Send To: Email address of original sender (`{{ $('Gmail trigger').item.json.headers.from }}`)  
     - Thread ID: Use original email’s thread ID (`{{ $('Gmail trigger').item.json.threadId }}`)  
     - Subject: Prefix with "Re: " and original subject (`={{ 'Re: ' + $('Gmail trigger').item.json.headers.subject }}`)  
     - Email type: HTML  
   - Credentials: Gmail OAuth2  

10. **Add Sticky Notes throughout the workflow:**  
    - Include explanatory notes for Gmail trigger setup, message assessment, language-based switching, draft generation, and draft creation for user orientation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is ideal for anyone who receives a high volume of Gmail inquiries, especially those providing multilingual customer support or handling diverse client communications.                                                                                                                                                                                                                      | Sticky Note5 overview content                                                                              |
| Managing frequent emails in multiple languages can be overwhelming. This workflow reduces manual drafting by automatically generating context-aware replies using OpenAI GPT-4o, letting users focus on personalization and quality assurance.                                                                                                                                                         | Sticky Note5 overview content                                                                              |
| Setup instructions: Connect Gmail and OpenAI accounts in n8n, configure Gmail labels correctly, and customize AI prompt instructions and supported languages to suit your brand’s tone.                                                                                                                                                                                                                 | Sticky Note5 overview content                                                                              |
| Gmail Trigger node polling interval can be adjusted to balance timeliness and API rate limits. Use Gmail filters or labels to precisely target incoming messages.                                                                                                                                                                                                                                      | Sticky Note (Gmail Trigger)                                                                                 |
| The AI assessment step uses a JSON parser to ensure the AI output is structured and reliable for downstream logic.                                                                                                                                                                                                                                                                                    | Sticky Note1                                                                                               |
| Language-based reply generation includes culturally appropriate greetings, tone, and formatting instructions to maintain professionalism.                                                                                                                                                                                                                                                             | Sticky Note3                                                                                               |
| Draft replies are converted to HTML and saved as drafts in the original Gmail conversation thread for user review before sending.                                                                                                                                                                                                                                                                      | Sticky Note4                                                                                               |

---

**Disclaimer:**  
The text and data provided are exclusively from an automated n8n workflow. All processing complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.