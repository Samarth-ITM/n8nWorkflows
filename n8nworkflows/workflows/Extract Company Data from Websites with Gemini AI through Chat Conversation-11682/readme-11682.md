Extract Company Data from Websites with Gemini AI through Chat Conversation

https://n8nworkflows.xyz/workflows/extract-company-data-from-websites-with-gemini-ai-through-chat-conversation-11682


# Extract Company Data from Websites with Gemini AI through Chat Conversation

---

### 1. Workflow Overview

This workflow automates the extraction of structured company data from official corporate websites by interacting with users through a chat interface powered by Google Gemini AI. It is designed for scenarios where users provide company website URLs in a conversational manner, and the workflow returns validated, localized company information such as CEO name, address, and a summary.

The workflow is logically organized into the following blocks:

**1.1 Chat Input Reception and URL Extraction**  
Receives chat messages from users, prompts for URLs if missing, and uses AI to parse and extract a valid URL from the chat input.

**1.2 URL Validation and Control Flow**  
Checks whether a valid URL was extracted and routes the flow accordingly—either proceeding to data extraction or requesting user input again.

**1.3 AI-Driven Company Data Extraction**  
Uses an AI agent with an embedded web scraper logic to iteratively explore the official corporate website, validate its legitimacy, extract requested fields, and return the results in CSV format.

**1.4 CSV Parsing and Formatting**  
Parses the AI-generated CSV data into structured JSON, formats the response message, and sends it back to the user within the chat.

**1.5 Conversation Continuation**  
Prompts the user for additional URLs to extract company data again, enabling a continuous conversational experience.

**Supporting Nodes**: Configuration settings, HTTP request tool used by AI for web scraping, and informative sticky notes for users and maintainers.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Input Reception and URL Extraction

**Overview:**  
This block manages incoming chat messages, initializes the conversation, and extracts a URL from the user's input using an AI agent.

**Nodes Involved:**  
- When chat message received  
- Config  
- AI Agent (Extract URL)  
- ExtractedChatInput  
- Request User Input  
- Google Gemini Chat Model  
- Sticky Note (Instructional)

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (Langchain)  
  - Role: Entry point; triggers workflow on chat message reception.  
  - Configuration: Public webhook; initial greeting message prompts user to enter a URL.  
  - Input: Incoming chat message  
  - Output: Passes chat input downstream  
  - Edge Cases: Missing or invalid chat input could cause URL extraction failure.

- **Config**  
  - Type: Set  
  - Role: Stores key parameters such as user chat input, target company fields (CEO, Address, Summary), and output language (English).  
  - Configuration: Assigns variables from incoming JSON or hardcoded defaults.  
  - Edge Cases: Incorrect or missing configuration values can affect later extraction.

- **AI Agent (Extract URL)**  
  - Type: Langchain Agent (Define prompt)  
  - Role: Uses AI to extract homepage URL from the user chat input; returns JSON with URL or null if none found.  
  - Configuration: Prompt instructs AI to strictly output valid JSON with either a URL or null.  
  - Inputs: `chatInput` from Config node  
  - Outputs: JSON object `{ "url": "..."} or { "url": null }`  
  - Edge Cases: No URL found returns null; must handle gracefully downstream.

- **ExtractedChatInput**  
  - Type: Code  
  - Role: Parses AI output to extract JSON URL string from possible code block formatting.  
  - Key Code: Regex to detect JSON code blocks and parse them safely.  
  - Input: Output from AI Agent (Extract URL)  
  - Output: JSON object with `url` field for flow control  
  - Edge Cases: Malformed JSON, missing code block, or empty output.

- **Check URL**  
  - Type: If  
  - Role: Verifies whether extracted URL is non-empty and valid to decide path forward.  
  - Configuration: Condition `url` is not empty string.  
  - Inputs: Parsed JSON with URL  
  - Outputs: True (valid URL) routes to extraction; False routes to prompt user input again.

- **Request User Input**  
  - Type: Langchain Chat  
  - Role: Prompts user to enter a company website URL when no valid URL is detected.  
  - Configuration: Message: "Please enter the company website URL you want to search."  
  - Edge Cases: User may still enter invalid input resulting in repeated prompts.

- **Google Gemini Chat Model**  
  - Type: Language Model (Google Gemini)  
  - Role: Provides AI language model backend for AI Agent (Extract URL).  
  - Credentials: Gemini API key from Google AI Studio.  
  - Edge Cases: API errors, rate limits, or credential misconfiguration.

- **Sticky Note**  
  - Provides detailed instructions on workflow usage, credential setup, and customization options.

---

#### 2.2 URL Validation and Control Flow

**Overview:**  
Controls the workflow path based on validity of the extracted URL, ensuring only valid input proceeds to data extraction.

**Nodes Involved:**  
- Check URL  
- Request User Input (fallback)

**Node Details:**  
Explained above in 2.1; no additional nodes.

---

#### 2.3 AI-Driven Company Data Extraction

**Overview:**  
This block uses an AI agent configured with explicit instructions to validate the official corporate website, navigate its pages, extract requested company fields, and produce a CSV-formatted output.

**Nodes Involved:**  
- AI Agent (Access URL)  
- Google Gemini Chat Model1  
- HttpRequestTool  
- Sticky Note2

**Node Details:**

- **AI Agent (Access URL)**  
  - Type: Langchain Agent (Define prompt)  
  - Role: Main AI-driven scraper agent that:  
    - Validates site legitimacy (rejects non-corporate, media, directory sites)  
    - Navigates to "Company Profile" or equivalent pages  
    - Extracts requested fields (CEO, Address, Summary) strictly from official domain  
    - Produces CSV output with status and reason fields  
  - Configuration:  
    - Max 15 iterations for navigation  
    - System message enforces strict validation rules and extraction process  
    - Uses HttpRequestTool internally for web content fetching  
    - Outputs CSV in a Markdown code block with mandatory quoting  
  - Inputs: Validated URL, target fields, language from Config  
  - Outputs: CSV text with extracted data and extraction status  
  - Edge Cases: Site rejection, HTTP timeouts, AI hallucination mitigated by strict prompt, API failures.

- **Google Gemini Chat Model1**  
  - Type: Language Model (Google Gemini)  
  - Role: Provides AI language model backend for AI Agent (Access URL).  
  - Credentials: Same Gemini API key.  
  - Edge Cases: Same as above.

- **HttpRequestTool**  
  - Type: HTTP Request (AI Tool)  
  - Role: Tool integrated for AI agent to fetch web pages; supports redirects, headers, timeout.  
  - Configuration:  
    - Custom User-Agent header to simulate browser  
    - 60 seconds timeout and up to 10 redirects allowed  
    - Accepts unauthorized certificates for flexibility  
  - Inputs: URLs supplied dynamically by AI agent  
  - Outputs: Full HTTP responses passed back to AI agent  
  - Edge Cases: Network errors, invalid SSL, redirect loops.

- **Sticky Note2**  
  - Explains AI Search block purpose: AI accesses URL to extract company info and returns results.

---

#### 2.4 CSV Parsing and Formatting

**Overview:**  
Parses the CSV output from the AI agent into structured JSON, formats a user-friendly chat message indicating success or failure, and sends it back to the user.

**Nodes Involved:**  
- ParsedCsv  
- ChatResponse  
- Respond to Chat

**Node Details:**

- **ParsedCsv**  
  - Type: Code  
  - Role: Parses Markdown CSV code block from AI output into JSON array of records.  
  - Key Code:  
    - Extracts CSV text from ```csv code block using regex  
    - Custom CSV parser handling quotes and commas  
    - Returns array of objects keyed by CSV headers  
  - Inputs: AI Agent (Access URL) output JSON with CSV markdown  
  - Outputs: Parsed JSON array or throws error if CSV not found  
  - Edge Cases: No CSV found, malformed CSV, empty data.

- **ChatResponse**  
  - Type: Code  
  - Role: Formats the extracted company data into a Markdown chat message.  
  - Logic:  
    - Checks `autoFetchStatus`; if not 'ok' returns warning with reason.  
    - If 'ok', lists all extracted fields (excluding technical keys) with labels and values.  
  - Inputs: Parsed CSV JSON record  
  - Outputs: JSON with formatted text for chat  
  - Edge Cases: Missing fields, partial data.

- **Respond to Chat**  
  - Type: Langchain Chat  
  - Role: Sends the formatted message back to the chat user.  
  - Configuration: Message content from ChatResponse node, no wait for user reply.

---

#### 2.5 Conversation Continuation

**Overview:**  
Prompts the user to enter another URL for extraction, enabling a conversational loop.

**Nodes Involved:**  
- Request Next URL  
- AI Agent (Extract URL)

**Node Details:**

- **Request Next URL**  
  - Type: Langchain Chat  
  - Role: Asks the user if they want to search for another company.  
  - Message: "Would you like to search for another company? Please enter the URL."  
  - Output: Loops back to AI Agent (Extract URL) to process new input.

- **AI Agent (Extract URL)**  
  - Reused to parse new URL from chat input and continue the cycle.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                       | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                                       |
|-------------------------|----------------------------------|-------------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger (Langchain)          | Entry point, receives user chat     |                            | Config                   | See Sticky Note - Usage instructions and setup details                                                                           |
| Config                  | Set                              | Store chat input and config params  | When chat message received | AI Agent (Extract URL)    |                                                                                                                                  |
| AI Agent (Extract URL)  | Langchain Agent                  | Extract URL JSON from chat input    | Config                     | ExtractedChatInput        | See Sticky Note1 - Extract URLs using AI                                                                                         |
| ExtractedChatInput      | Code                             | Parse AI JSON output for URL        | AI Agent (Extract URL)      | Check URL                 |                                                                                                                                  |
| Check URL               | If                               | Check if URL valid                   | ExtractedChatInput          | AI Agent (Access URL), Request User Input |                                                                                                                                  |
| Request User Input      | Langchain Chat                   | Prompt user for URL if missing      | Check URL                   | AI Agent (Extract URL)    |                                                                                                                                  |
| Google Gemini Chat Model | Language Model (Google Gemini)   | Backend for AI Agent (Extract URL)  |                            | AI Agent (Extract URL)    |                                                                                                                                  |
| AI Agent (Access URL)   | Langchain Agent                  | AI-driven website scraping & extract| Check URL                  | ParsedCsv                 | See Sticky Note2 - AI Search and extraction                                                                                      |
| Google Gemini Chat Model1| Language Model (Google Gemini)   | Backend for AI Agent (Access URL)   |                            | AI Agent (Access URL)     |                                                                                                                                  |
| HttpRequestTool         | HTTP Request (AI Tool)           | Fetch webpages for AI agent         | AI Agent (Access URL)       | AI Agent (Access URL)     |                                                                                                                                  |
| ParsedCsv               | Code                             | Parse CSV from AI output            | AI Agent (Access URL)       | ChatResponse              |                                                                                                                                  |
| ChatResponse            | Code                             | Format extracted data for chat      | ParsedCsv                   | Respond to Chat           |                                                                                                                                  |
| Respond to Chat         | Langchain Chat                   | Send formatted data back to chat   | ChatResponse                | Request Next URL          |                                                                                                                                  |
| Request Next URL        | Langchain Chat                   | Ask user if they want another search| Respond to Chat            | AI Agent (Extract URL)    |                                                                                                                                  |
| Sticky Note             | Sticky Note                     | Usage instructions and info         |                            |                          | Contains detailed instructions and credential setup link: https://aistudio.google.com/api-keys                                  |
| Sticky Note1            | Sticky Note                     | Explains URL extraction step        |                            |                          | ## Extract URLs using AI - This step uses AI to extract URLs from chat messages.                                                 |
| Sticky Note2            | Sticky Note                     | Explains AI Search step             |                            |                          | ## AI Search - The AI accesses the URL to extract company information and returns the results.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Node Type: `When chat message received` (Langchain chatTrigger)  
   - Configure webhook as public, enable response nodes mode  
   - Set initial message: "Hi! I can help you extract company information. Please enter the URL to get started."

2. **Add Config Node**  
   - Node Type: Set  
   - Assign variables:  
     - `chatInput` = Expression: `{{$json.chatInput}}`  
     - `targetCompanyFields` = `"CEO,Address,Summary"`  
     - `language` = `"English"`

3. **Add AI Agent Node (Extract URL)**  
   - Node Type: Langchain Agent  
   - Credential: Google Gemini API key (configured via Google AI Studio)  
   - Prompt Type: define  
   - Text Prompt:  
     ```
     Extract the homepage URL from the User Input and return it in the specified Output Format inside a code block.

     Important:
     Always return a valid JSON object.
     If no URL is found, you MUST return the object with null: { "url": null }

     [Output Format]
     { "url": "extracted_url_or_null" }

     [User Input]
     {{ $json.chatInput }}
     ```
   - Connect output of Config to this node.

4. **Add Code Node (ExtractedChatInput)**  
   - Node Type: Code  
   - JavaScript code to parse JSON from code block in AI output (see 2.1)  
   - Connect AI Agent (Extract URL) output here.

5. **Add If Node (Check URL)**  
   - Node Type: If  
   - Condition: Check if `url` field from ExtractedChatInput is not empty string.  
   - True branch: Proceed to AI Agent (Access URL)  
   - False branch: Request User Input.

6. **Add Langchain Chat Node (Request User Input)**  
   - Message: "Please enter the company website URL you want to search."  
   - Connect from False branch of Check URL.  
   - Connect output back to AI Agent (Extract URL) to loop.

7. **Add AI Agent Node (Access URL)**  
   - Type: Langchain Agent  
   - Credential: Same Google Gemini API key  
   - Prompt Type: define  
   - Prompt text includes detailed instructions to: validate site, navigate to company profile, extract fields into CSV, produce status and reason.  
   - Configure max iterations: 15  
   - System Message: As per node details in 2.3  
   - Connect True branch from Check URL here.

8. **Add HTTP Request Tool Node (HttpRequestTool)**  
   - URL: Dynamic, from AI Agent's tool input.  
   - Timeout: 60 seconds  
   - Max redirects: 10  
   - Send custom User-Agent header imitating Chrome browser on Mac  
   - Allow unauthorized certs enabled  
   - Connect as AI Tool to AI Agent (Access URL).

9. **Add Code Node (ParsedCsv)**  
   - JavaScript to extract CSV from markdown code block in AI Agent output and parse into JSON records.  
   - Connect AI Agent (Access URL) output here.

10. **Add Code Node (ChatResponse)**  
    - Format output message showing success or error and extracted fields in Markdown.  
    - Connect ParsedCsv output here.

11. **Add Langchain Chat Node (Respond to Chat)**  
    - Message: Use output from ChatResponse node.  
    - Wait for user reply: false  
    - Connect ChatResponse output here.

12. **Add Langchain Chat Node (Request Next URL)**  
    - Message: "Would you like to search for another company? Please enter the URL."  
    - Connect Respond to Chat output here.

13. **Loop back Request Next URL output to AI Agent (Extract URL)**  
    - This creates a conversational loop for multiple queries.

14. **Add Google Gemini Chat Model nodes**  
    - One connected as AI Language Model to AI Agent (Extract URL)  
    - Another connected as AI Language Model to AI Agent (Access URL)  
    - Assign the same Gemini API credentials.

15. **Add Sticky Notes**  
    - Include usage instructions, explanation of URL extraction step, and AI search step for maintainers/users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Please send a corporate website URL via chat. The AI will investigate the company website on your behalf and return the extracted company information. Retry or try another URL easily as this is conversational.            | Sticky Note in workflow                                            |
| To get started, set up the Credential in the Gemini node attached to the AI Agent node. API key can be obtained from Google AI Studio: https://aistudio.google.com/api-keys                                                    | Credential setup instruction included in Sticky Note              |
| You can customize the workflow by changing `targetCompanyFields` (default: CEO, Address, Summary) and `language` (default: English) in the Config node.                                                                     | Configuration node notes                                          |
| The AI enforces strict validation of the corporate website to avoid non-official, media, blog, or directory sites. The extraction only proceeds if the site is confirmed legitimate and contains the official company name.   | AI Agent (Access URL) prompt content                               |
| The HTTP Request Tool simulates a browser with a User-Agent header and handles redirects and SSL issues to maximize successful page fetches.                                                                               | HttpRequestTool configuration                                     |
| The CSV parser is robust to handle quoted fields and common CSV edge cases, throwing errors if CSV format is invalid or missing.                                                                                             | ParsedCsv code node notes                                         |

---

**Disclaimer:**  
The provided text derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly available.

---