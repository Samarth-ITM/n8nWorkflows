Check Suspicious Links via Telegram with GPT-4 Analysis of VirusTotal & urlscan.io Results

https://n8nworkflows.xyz/workflows/check-suspicious-links-via-telegram-with-gpt-4-analysis-of-virustotal---urlscan-io-results-7926


# Check Suspicious Links via Telegram with GPT-4 Analysis of VirusTotal & urlscan.io Results

### 1. Workflow Overview

This workflow enables the rapid analysis of potentially malicious URLs submitted via Telegram messages by leveraging dual URL scanning services (VirusTotal and urlscan.io) combined with a GPT-4 powered AI summarization. Its primary use case is to allow users, especially those on mobile devices, to quickly check suspicious links without manual investigation, receiving a clear, professional report directly in Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures URL inputs from Telegram messages.
- **1.2 URL Scanning:** Sends the URL to both VirusTotal and urlscan.io services for threat analysis.
- **1.3 Results Aggregation:** Merges and prepares combined scan data for AI analysis.
- **1.4 AI Processing & Summarization:** Uses a GPT-4 powered LangChain agent to generate a readable, plain-language summary of scan results.
- **1.5 Output & Logging:** Sends the summary back to the Telegram user and logs the results into Google Sheets for auditing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming Telegram messages containing URLs to be scanned.
- **Nodes Involved:**  
  - Telegram Trigger
- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Entry point listening for new Telegram messages (`updates: ["message"]`).  
    - Configuration: Uses a Telegram Bot credential named "Malicious URL Bot".  
    - Key expressions: Reads URL from `{{$json.message.text}}`.  
    - Inputs: None (trigger node)  
    - Outputs: Emits the Telegram message data to downstream nodes.  
    - Edge cases: May receive irrelevant messages without URLs; no explicit URL validation here.  
    - Failure modes: Telegram API downtime, webhook misconfiguration.

#### 2.2 URL Scanning

- **Overview:** Sends the captured URL to two distinct URL scanning APIs, VirusTotal and urlscan.io, to gather threat intelligence.
- **Nodes Involved:**  
  - VirusTotal HTTP Request  
  - urlscan Perform Scan
- **Node Details:**  
  - **VirusTotal HTTP Request**  
    - Type: HTTP Request node  
    - Role: Submits the URL to VirusTotal API for scanning.  
    - Configuration: POST request to `https://www.virustotal.com/api/v3/urls` with query parameter `url={{$json.message.text}}`. Uses stored VirusTotal credentials.  
    - Inputs: Receives Telegram Trigger output.  
    - Outputs: VirusTotal scan results JSON.  
    - Error handling: Continues workflow on error (`onError: "continueRegularOutput"`).  
    - Edge cases: API rate limits, invalid URLs, network errors, or API key expiration.  
  - **urlscan Perform Scan**  
    - Type: urlscan.io node  
    - Role: Performs a URL scan with urlscan.io service.  
    - Configuration: URL set to `{{$json.message.text}}`, uses urlscan.io credentials.  
    - Inputs: Receives Telegram Trigger output.  
    - Outputs: urlscan.io scan results.  
    - Error handling: Continues workflow on error.  
    - Edge cases: Scan blocked by urlscan.io, API quota exceeded, invalid URL format.

#### 2.3 Results Aggregation

- **Overview:** Combines the two scan results into a single data structure and prepares a summarized text for AI analysis.
- **Nodes Involved:**  
  - Merge Scans  
  - Prepare Summary Data
- **Node Details:**  
  - **Merge Scans**  
    - Type: Merge node  
    - Role: Combines outputs from VirusTotal and urlscan.io scans.  
    - Configuration: Default merge settings with two inputs.  
    - Inputs: Receives data from `VirusTotal HTTP Request` and `urlscan Perform Scan`.  
    - Outputs: Merged item with partial scan data.  
  - **Prepare Summary Data**  
    - Type: Code node  
    - Role: Reads merged scan results and constructs a combined summary string, distinguishing success or failure of urlscan.io scan.  
    - Configuration: JavaScript code that:  
      - Checks if urlscan.io data is available or failed.  
      - Constructs a summary field combining urlscan.io and VirusTotal JSON results or fallback message.  
    - Inputs: Receives merged scan data.  
    - Outputs: JSON with added `summary` field for AI.  
    - Edge cases: Malformed or missing scan results.

#### 2.4 AI Processing & Summarization

- **Overview:** Uses a GPT-4 powered LangChain agent to analyze the scan results and generate a user-friendly summary report.
- **Nodes Involved:**  
  - Malicious URL Summary Agent (LangChain Agent)  
  - OpenAI Model (GPT-4)  
  - Malicious URL Memory (Buffer Memory)  
  - Limit 1 Summary
- **Node Details:**  
  - **Malicious URL Summary Agent**  
    - Type: LangChain Agent node  
    - Role: Orchestrates the summarization process using AI with memory support.  
    - Configuration:  
      - Prompt defines role as cybersecurity assistant summarizing VirusTotal and urlscan.io data.  
      - Instructions include steps for threat level classification, detection quantification, and recommendations.  
      - Output format: bullet points plus a 4-8 sentence summary.  
    - Inputs: Receives `summary` field from `Prepare Summary Data`.  
    - Outputs: AI-generated readable summary.  
  - **OpenAI Model**  
    - Type: LangChain LM Chat OpenAI node  
    - Role: Provides GPT-4 model inference for the agent.  
    - Configuration: Uses GPT-4 model (credential "OpenAi account 2").  
    - Inputs: From Malicious URL Summary Agent.  
    - Outputs: AI text response.  
  - **Malicious URL Memory**  
    - Type: Memory Buffer Window node  
    - Role: Stores conversational context per analyzed URL (sessionKey "summary") for continuity.  
    - Inputs: From OpenAI Model (AI output).  
    - Outputs: Back to Malicious URL Summary Agent.  
  - **Limit 1 Summary**  
    - Type: Limit node  
    - Role: Ensures only one summary message is passed on per URL scan to avoid duplicates.  
    - Inputs: From Malicious URL Summary Agent.  
    - Outputs: Sends summary downstream.

- **Edge cases:** OpenAI API rate limits or errors, malformed prompt variables, memory overflow.

#### 2.5 Output & Logging

- **Overview:** Sends the AI-generated summary back to the user on Telegram and logs the scan results with timestamp in Google Sheets.
- **Nodes Involved:**  
  - Send a text message (Telegram)  
  - URL Logging (Google Sheets)
- **Node Details:**  
  - **Send a text message**  
    - Type: Telegram node  
    - Role: Sends the summary text back to the Telegram chat from which the URL originated.  
    - Configuration: Message text set to `{{$json.output}}`; chatId dynamically set from Telegram Trigger.  
    - Inputs: From Limit 1 Summary node.  
    - Outputs: None downstream (terminal).  
    - Credential: Telegram Bot "Malicious URL Bot".  
    - Edge cases: Telegram API failures, chatId resolution errors.  
  - **URL Logging**  
    - Type: Google Sheets node  
    - Role: Append or update a row in a Google Sheet with URL, summary report, and current timestamp for auditing.  
    - Configuration:  
      - Document ID and Sheet gid for a specific spreadsheet ("URL Scanner").  
      - Columns: URL, Report (summary), Date/Time.  
      - Append or update operation matching on URL.  
    - Inputs: From Limit 1 Summary node.  
    - Credential: Google OAuth2 account.  
    - Edge cases: Google API quota or permission errors, document or sheet not found.

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                                 | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                  |
|--------------------------|---------------------------|------------------------------------------------|-----------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note              | Sticky Note               | Visual marker: "Malicious URL Scanner Through Telegram" | None                        | None                         | Malicious URL Scanner Through Telegram                                                                       |
| Telegram Trigger         | Telegram Trigger          | Capture incoming Telegram messages with URLs  | None                        | VirusTotal HTTP Request, urlscan Perform Scan |                                                                                                              |
| VirusTotal HTTP Request  | HTTP Request              | Submit URL to VirusTotal API                    | Telegram Trigger            | Merge Scans                  |                                                                                                              |
| urlscan Perform Scan     | urlscan.io                | Submit URL to urlscan.io API                    | Telegram Trigger            | Merge Scans                  |                                                                                                              |
| Merge Scans              | Merge                     | Combine VirusTotal and urlscan.io results      | VirusTotal HTTP Request, urlscan Perform Scan | Prepare Summary Data         |                                                                                                              |
| Prepare Summary Data     | Code                      | Build combined summary text for AI processing  | Merge Scans                 | Malicious URL Summary Agent  |                                                                                                              |
| Malicious URL Summary Agent | LangChain Agent          | Analyze scan results and generate summary      | Prepare Summary Data, Malicious URL Memory | Limit 1 Summary              |                                                                                                              |
| OpenAI Model             | LangChain LM Chat OpenAI  | GPT-4 model inference for summarization        | Malicious URL Summary Agent | Malicious URL Memory         |                                                                                                              |
| Malicious URL Memory     | LangChain Memory Buffer   | Maintain session memory for AI agent            | OpenAI Model                | Malicious URL Summary Agent  |                                                                                                              |
| Limit 1 Summary          | Limit                     | Limit output to one summary per URL             | Malicious URL Summary Agent | Send a text message, URL Logging |                                                                                                              |
| Send a text message      | Telegram                  | Send summary message back to Telegram user      | Limit 1 Summary             | None                         |                                                                                                              |
| URL Logging              | Google Sheets             | Log URL scan summary with timestamp             | Limit 1 Summary             | None                         |                                                                                                              |
| Sticky Note1             | Sticky Note               | Workflow goal description and node overview     | None                        | None                         | Goal:\nThis workflow allows users to quickly analyze potentially malicious URLs received via text or email, directly from their mobile device using Telegram.\nWhen users don't have time to manually investigate links on a computer or search engine, they can simply paste the URL into the Telegram chat and receive a concise summary of the scan results—powered by two open-source URL scanning services.\n\nDisclaimer:\nWhile this tool helps identify threats, it does not guarantee full protection. Use caution—Telegram itself could be a potential point of compromise. Always follow safe browsing practices.\n\nWorkflow Nodes:\nTelegram Trigger – Listens for incoming messages with URLs.\n\nurlscan.io – Performs the first malicious URL scan.\n\nVirusTotal (HTTP Request) – Executes a second URL scan using VirusTotal.\n\nMerge Node – Combines the results from both scanners.\n\nAI Agent (ChatGPT with simple memory) – Analyzes the scan results and generates a readable summary.\n\nLimit Node – Ensures only one summary is sent per URL.\n\nTelegram Send Message Node – Sends the summary back to the user.\n\nGoogle Sheets (Logging) – Records scan results for auditing or historical reference.\n |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Credential: Configure with your Telegram Bot API credentials ("Malicious URL Bot").  
   - Parameters: Listen for update type "message".  
   - Purpose: Receive incoming Telegram messages.

2. **Create VirusTotal HTTP Request node:**  
   - Type: HTTP Request  
   - Credential: Configure with VirusTotal API key.  
   - Parameters:  
     - Method: POST  
     - URL: `https://www.virustotal.com/api/v3/urls`  
     - Query Parameter: `url={{ $json.message.text }}` (captures URL from Telegram message)  
     - Error Handling: Continue on error  
   - Connect input from Telegram Trigger output.

3. **Create urlscan Perform Scan node:**  
   - Type: urlscan.io  
   - Credential: Configure with urlscan.io API key.  
   - Parameters:  
     - URL: `{{$json.message.text}}`  
     - Error Handling: Continue on error  
   - Connect input from Telegram Trigger output.

4. **Create Merge Scans node:**  
   - Type: Merge  
   - Parameters: Default (merge two inputs)  
   - Connect inputs from VirusTotal HTTP Request and urlscan Perform Scan nodes.

5. **Create Prepare Summary Data node:**  
   - Type: Code node (JavaScript)  
   - Parameters: Use code to access merged inputs and build a combined summary string, e.g.:  
     ```javascript
     const items = $input.all();
     return items.map(item => {
       const urlscan = item.json.urlscan || {};
       const virustotal = item.json.virustotal || {};
       let summary = "";
       if (urlscan.message) {
         summary = `✅ urlscan.io result:\n${JSON.stringify(urlscan)}\n\n✅ VirusTotal result:\n${JSON.stringify(virustotal)}`;
       } else {
         summary = `⚠️ urlscan.io scan failed. Falling back to VirusTotal only:\n${JSON.stringify(virustotal)}`;
       }
       return { json: { ...item.json, summary }, binary: item.binary ?? undefined };
     });
     ```  
   - Connect input from Merge Scans output.

6. **Create Malicious URL Summary Agent (LangChain Agent) node:**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text prompt: Role definition and detailed instructions for cybersecurity assistant analyzing VirusTotal and urlscan.io results.  
     - Output format specifying bullet points and summary text with recommendations.  
   - Connect input from Prepare Summary Data output.

7. **Create OpenAI Model node:**  
   - Type: LangChain LM Chat OpenAI  
   - Credential: Configure OpenAI API key ("OpenAi account 2").  
   - Parameters: Select model "gpt-4".  
   - Connect input from Malicious URL Summary Agent node.

8. **Create Malicious URL Memory node:**  
   - Type: LangChain Memory Buffer Window  
   - Parameters:  
     - Session Key: "summary"  
     - Session ID Type: customKey  
   - Connect input from OpenAI Model output.  
   - Connect output back to Malicious URL Summary Agent (to provide memory context).

9. **Create Limit 1 Summary node:**  
   - Type: Limit  
   - Parameters: Default (limit to 1 item)  
   - Connect input from Malicious URL Summary Agent output.

10. **Create Send a text message node:**  
    - Type: Telegram  
    - Credential: Use Telegram Bot credentials ("Malicious URL Bot").  
    - Parameters:  
      - `text`: `={{ $json.output }}` (the AI-generated summary)  
      - `chatId`: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Connect input from Limit 1 Summary output.

11. **Create URL Logging node:**  
    - Type: Google Sheets  
    - Credential: Configure Google OAuth2 credentials.  
    - Parameters:  
      - Document ID: Google Sheet ID for logging.  
      - Sheet name/gid: e.g. "gid=0"  
      - Operation: Append or update  
      - Columns mapping:  
        - URL: `={{ $('Telegram Trigger').item.json.message.text }}`  
        - Report: `={{ $json.output }}`  
        - Date/Time: `={{ $now }}`  
    - Connect input from Limit 1 Summary output.

12. **Create Sticky Notes (optional):**  
    - Add descriptive sticky notes for workflow overview and goal as visual aids.

13. **Connect nodes according to the flow:**  
    - Telegram Trigger → VirusTotal HTTP Request & urlscan Perform Scan  
    - VirusTotal HTTP Request & urlscan Perform Scan → Merge Scans  
    - Merge Scans → Prepare Summary Data  
    - Prepare Summary Data → Malicious URL Summary Agent  
    - Malicious URL Summary Agent → OpenAI Model  
    - OpenAI Model → Malicious URL Memory → Malicious URL Summary Agent  
    - Malicious URL Summary Agent → Limit 1 Summary  
    - Limit 1 Summary → Send a text message & URL Logging

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                      | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow allows quick threat assessment of suspicious URLs via Telegram, integrating VirusTotal and urlscan.io scans with AI summarization to aid users without technical expertise.                                                                                                             | Workflow description and user context                                                                          |
| Disclaimer: This tool does not guarantee complete protection. Users should continue safe browsing practices; Telegram itself can be a potential attack vector.                                                                                                                                      | Sticky Note1 content                                                                                          |
| Uses GPT-4 via LangChain integration in n8n for advanced natural language processing and summarization of JSON scan data.                                                                                                                                                                         | Node: Malicious URL Summary Agent                                                                             |
| VirusTotal API and urlscan.io API credentials must be provisioned and maintained with sufficient quota to avoid rate limiting or scan failures.                                                                                                                                                  | Credential requirements                                                                                        |
| Google Sheets logging enables audit trail and historical analysis of URL scans, useful for security teams or compliance.                                                                                                                                                                         | Node: URL Logging                                                                                              |
| Telegram Bot token must have proper webhook setup and permissions to receive and send messages in the target chat(s).                                                                                                                                                                            | Node: Telegram Trigger and Send a text message                                                                |
| For advanced users: The memory buffer can be extended or replaced with other LangChain memory types to enable richer conversational context or multi-turn analysis.                                                                                                                             | Node: Malicious URL Memory                                                                                      |
| OpenAI API usage may incur costs; monitor usage and optimize prompts accordingly.                                                                                                                                                                                                                 | Node: OpenAI Model                                                                                              |
| Useful URL scanning references: VirusTotal (https://www.virustotal.com/), urlscan.io (https://urlscan.io/)                                                                                                                                                                                        | External resources                                                                                            |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation platform. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.