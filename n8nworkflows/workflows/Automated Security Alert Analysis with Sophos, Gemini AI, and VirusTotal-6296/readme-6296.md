Automated Security Alert Analysis with Sophos, Gemini AI, and VirusTotal

https://n8nworkflows.xyz/workflows/automated-security-alert-analysis-with-sophos--gemini-ai--and-virustotal-6296


# Automated Security Alert Analysis with Sophos, Gemini AI, and VirusTotal

### 1. Workflow Overview

This workflow, titled **"Automated Security Alert Analysis with Sophos, Gemini AI, and VirusTotal"**, is designed to automate the ingestion, analysis, and reporting of security events detected by Sophos endpoint protection. It integrates multiple security intelligence sources and AI-driven analysis to triage alerts based on severity and type, enrich the alert with threat intelligence from VirusTotal, and generate actionable mitigation recommendations using Google Gemini AI. The output is sent as a formatted Telegram message to designated security personnel.

Logical blocks:

- **1.1 Input Reception & Filtering:** Receives incoming webhook POST requests from Sophos, applies filtering logic to focus on high-severity or critical endpoint threat events.
- **1.2 Indicator Extraction & VirusTotal Enrichment:** Extracts relevant indicators (file hash, domain, or IP) from the event, constructs VirusTotal API queries, and processes reputation data.
- **1.3 AI Analysis & Recommendation Generation:** Uses LangChain agents with embedded AI models (Google Gemini) to analyze the enriched data and produce a structured JSON report including risk level, summary, and mitigation steps.
- **1.4 Notification Dispatch:** Formats and sends a detailed alert message via Telegram to notify SOC analysts.
- **1.5 Contextual Memory Management:** Maintains session-based conversational memory keyed by customer ID for AI context persistence.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:**  
  Listens for incoming security event data via HTTP webhook, then filters events to process only those with high or critical severity or specific endpoint threat types.

- **Nodes Involved:**  
  - Webhook  
  - If

- **Node Details:**

  - **Webhook**  
    - *Type & Role:* HTTP Webhook node, entry point receiving POST requests from Sophos alerts.  
    - *Configuration:* HTTP method POST; path parameter placeholder "replace-with-your-webhook-path" (must be customized).  
    - *Expressions/Variables:* Entire payload accessed downstream via `$json.body`.  
    - *Connections:* Output connected to the If node.  
    - *Edge Cases:*  
      - Missing or malformed POST data could cause failures.  
      - The webhook path must be publicly accessible and secure.  
    - *Version:* 2  

  - **If**  
    - *Type & Role:* Conditional node that filters incoming events based on severity and event type.  
    - *Configuration:*  
      - Checks if `event.severity` matches regex `high|critical`.  
      - Or if `event.type` contains any of these substrings:  
        - "Event::Endpoint::Threat"  
        - "Event::Endpoint::WebControlViolation"  
        - "Event::Endpoint::WebFilteringBlocked"  
    - *Expressions:* Uses expressions like `={{ $json.body.event.severity }}` and `={{ $json.body.event.type }}` for conditions.  
    - *Connections:* Output connected to Code node on "true" branch (main path).  
    - *Edge Cases:*  
      - If event JSON structure changes, conditions may fail or misclassify.  
      - No handling for "false" branch; events not matching filters are dropped.  
    - *Version:* 2.2  

---

#### 2.2 Indicator Extraction & VirusTotal Enrichment

- **Overview:**  
  Extracts relevant threat indicators from the Sophos event (file hash, domain, or IP), constructs VirusTotal API endpoint URLs accordingly, performs VirusTotal reputation lookups, and formats the results for AI consumption.

- **Nodes Involved:**  
  - Code (indicator extraction & URL construction)  
  - Virus_Total (HTTP request to VirusTotal API)  
  - For_Gemini_Prompt (formatting VirusTotal results for AI prompt)

- **Node Details:**

  - **Code**  
    - *Type & Role:* JavaScript code node responsible for parsing the event and extracting the most relevant indicator for VirusTotal lookup.  
    - *Configuration Highlights:*  
      - Extracts `sha256` hash from `event.data.sha256`.  
      - Parses domain from `event.name` using a custom split by quotes and heuristic check for a dot.  
      - Falls back to `source_ip` if no hash or domain found.  
      - Constructs VirusTotal API URL based on indicator type (file, domain, or IP).  
      - Throws error if no indicator found.  
    - *Expressions:* Uses `$input.item.json.body.event` for event data.  
    - *Connections:* Output connects to Virus_Total node.  
    - *Edge Cases:*  
      - Input event might lack expected fields, causing errors.  
      - Domain extraction heuristic may fail for unexpected formats.  
      - VirusTotal API URLs must match indicator type strictly.  
    - *Version:* 2  

  - **Virus_Total**  
    - *Type & Role:* HTTP Request node querying VirusTotal API for indicator reputation.  
    - *Configuration:*  
      - URL set dynamically from `Code` node output (`url_to_check`).  
      - Uses HTTP Header Authentication with generic credential (likely API key).  
    - *Expressions:* URL uses expression `={{ $json.url_to_check }}`.  
    - *Connections:* Output connects to For_Gemini_Prompt node.  
    - *Edge Cases:*  
      - API key authentication failure.  
      - Rate limits or API downtime.  
      - Unexpected or empty response JSON.  
    - *Version:* 4.2  

  - **For_Gemini_Prompt**  
    - *Type & Role:* Code node that parses VirusTotal response and summarizes vendor detections for AI prompt.  
    - *Configuration:*  
      - Iterates over `last_analysis_results` to count vendors marking indicator as malicious/suspicious.  
      - Constructs formatted string listing vendors and their verdicts.  
      - Creates summary sentence for AI consumption.  
    - *Expressions:* Accesses Virus_Total response JSON.  
    - *Connections:* Output connects to AI Agent node.  
    - *Edge Cases:*  
      - Missing or malformed `last_analysis_results`.  
      - No vendors marking indicator as malicious (handles gracefully).  
    - *Version:* 2  

---

#### 2.3 AI Analysis & Recommendation Generation

- **Overview:**  
  Uses LangChain AI Agent (with Google Gemini model) to analyze event and VirusTotal data, generate a structured JSON output including risk level, summary, reputation description, affected internal IP, and mitigation steps.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Simple Memory (context memory for AI sessions)

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* LangChain AI Agent node that processes the entire alert with context and generates structured JSON output.  
    - *Configuration:*  
      - Prompt instructs to behave as a Senior Network Security Analyst.  
      - Emphasizes strict use of event data without assumptions or additions.  
      - Inputs include Sophos event JSON, VirusTotal stats, and formatted reputation text from previous nodes.  
      - Output keys: event, summary, risk_level, reputation description, affected internal IP with user and host info, and mitigation steps (array of 3 actionable recommendations).  
    - *Expressions:* Uses expressions to reference outputs from Webhook, If, Virus_Total, and For_Gemini_Prompt nodes.  
    - *Connections:* Output connects to Send a text message node.  
    - *Edge Cases:*  
      - AI model output may be malformed JSON or incomplete.  
      - Long prompts or response timeouts.  
    - *Version:* 2  

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model node providing AI inference capabilities used by AI Agent for natural language understanding and generation.  
    - *Configuration:* Default, no special parameters overridden.  
    - *Connections:* Feeds into AI Agent node as its language model.  
    - *Edge Cases:*  
      - API key or service outage issues.  
    - *Version:* 1  

  - **Simple Memory**  
    - *Type & Role:* Memory buffer node that maintains conversational context per customer ID for AI Agent.  
    - *Configuration:*  
      - Session key dynamically set to `customer_id` from webhook event data.  
      - Stores context to allow incremental or contextual AI reasoning across multiple events.  
    - *Connections:* Feeds memory into AI Agent node.  
    - *Edge Cases:*  
      - Session key missing or malformed.  
      - Memory size limits or persistence issues.  
    - *Version:* 1.3  

---

#### 2.4 Notification Dispatch

- **Overview:**  
  Formats the AI-generated analysis and VirusTotal data into a Telegram message alert and sends it to a configured chat ID for SOC personnel notification.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**

  - **Send a text message**  
    - *Type & Role:* Telegram node that posts a formatted text message to a specified chat.  
    - *Configuration:*  
      - Message text includes:  
        - Alert banner with risk level in uppercase.  
        - Threat name from webhook event.  
        - AI-generated summary.  
        - VirusTotal report including indicator value, number of flagged engines, and list of vendors marking it malicious/suspicious.  
        - Mitigation recommendations enumerated from AI output.  
      - Chat ID must be configured (placeholder `"YOUR_CHAT_ID"`).  
    - *Expressions:* Utilizes multiple expression interpolations to combine AI Agent output and VirusTotal summary data.  
    - *Connections:* Terminal node; no outputs.  
    - *Edge Cases:*  
      - Telegram API authentication failure or invalid chat ID.  
      - Message length or formatting issues.  
    - *Version:* 1.2  

---

### 3. Summary Table

| Node Name             | Node Type                             | Functional Role                            | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                           |
|-----------------------|-------------------------------------|--------------------------------------------|-------------------------|------------------------|-----------------------------------------------------------------------------------------------------|
| Webhook               | n8n-nodes-base.webhook               | Receives Sophos alert via HTTP POST         | -                       | If                     |                                                                                                     |
| If                    | n8n-nodes-base.if                    | Filters events by severity and type         | Webhook                 | Code                   |                                                                                                     |
| Code                  | n8n-nodes-base.code                  | Extracts indicator (hash/domain/IP) and builds VT URL | If                      | Virus_Total             |                                                                                                     |
| Virus_Total           | n8n-nodes-base.httpRequest           | Queries VirusTotal API for indicator reputation | Code                    | For_Gemini_Prompt      |                                                                                                     |
| For_Gemini_Prompt     | n8n-nodes-base.code                  | Processes VT response to list malicious vendors and summary | Virus_Total              | AI Agent               |                                                                                                     |
| AI Agent              | @n8n/n8n-nodes-langchain.agent       | AI-based analysis and mitigation recommendation generation | For_Gemini_Prompt, Simple Memory, Google Gemini Chat Model | Send a text message      |                                                                                                     |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides language model for AI Agent         | -                       | AI Agent               |                                                                                                     |
| Simple Memory         | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains AI session memory by customer ID   | -                       | AI Agent               |                                                                                                     |
| Send a text message    | n8n-nodes-base.telegram              | Sends formatted alert message to Telegram  | AI Agent                | -                      |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Webhook`  
   - Type: HTTP Webhook  
   - HTTP Method: POST  
   - Path: Set to a unique endpoint path (e.g., `sophos-alert`)  
   - No authentication  
   - Connect output to the `If` node  

2. **Create If Node**  
   - Name: `If`  
   - Type: If  
   - Add multiple OR conditions:  
     - `event.severity` matches regex `high|critical`  
     - `event.type` contains `Event::Endpoint::Threat`  
     - `event.type` contains `Event::Endpoint::WebControlViolation`  
     - `event.type` contains `Event::Endpoint::WebFilteringBlocked`  
   - Use expressions like `={{ $json.body.event.severity }}` for left values  
   - Connect output true branch to `Code` node  

3. **Create Code Node (Indicator Extraction)**  
   - Name: `Code`  
   - Type: Code (JavaScript)  
   - Paste the provided code that:  
     - Extracts `sha256` hash from `event.data.sha256`  
     - Extracts domain from quoted substring in `event.name`  
     - Uses `source_ip` as fallback  
     - Constructs VirusTotal URL accordingly (file, domain, or IP endpoint)  
     - Throws error if no indicator found  
   - Connect output to `Virus_Total` node  

4. **Create HTTP Request Node (Virus_Total)**  
   - Name: `Virus_Total`  
   - Type: HTTP Request  
   - URL: Use expression `={{ $json.url_to_check }}`  
   - Authentication: HTTP Header Auth with VirusTotal API key (create credential)  
   - Method: GET  
   - Connect output to `For_Gemini_Prompt` node  

5. **Create Code Node (For_Gemini_Prompt)**  
   - Name: `For_Gemini_Prompt`  
   - Type: Code (JavaScript)  
   - Paste code that:  
     - Parses `last_analysis_results` from VirusTotal response  
     - Counts vendors flagging as malicious/suspicious  
     - Builds formatted vendor list string and summary sentence  
   - Connect output to `AI Agent` node  

6. **Create LangChain Nodes: Google Gemini Chat Model and Simple Memory**  
   - Name: `Google Gemini Chat Model`  
     - Type: LangChain LM Chat Google Gemini  
     - Default settings  
   - Name: `Simple Memory`  
     - Type: LangChain Memory Buffer Window  
     - Session key expression: `={{ $('Webhook').item.json.body.event.customer_id }}`  
   - Connect both as inputs to `AI Agent` node, respectively under `ai_languageModel` and `ai_memory` inputs  

7. **Create AI Agent Node**  
   - Name: `AI Agent`  
   - Type: LangChain Agent  
   - Prompt: Use the detailed prompt instructing AI to analyze Sophos event and VirusTotal data, strictly using event data only, outputting structured JSON with keys: event, summary, risk_level, reputation, affected_internal_ip, mitigation_steps (array).  
   - Connect output to `Send a text message` node  

8. **Create Telegram Node (Send a text message)**  
   - Name: `Send a text message`  
   - Type: Telegram  
   - Text: Use the formatted message including:  
     - Risk level uppercase banner from AI output  
     - Threat name from webhook event  
     - AI summary  
     - VirusTotal report with indicator value, total flags, and vendor list  
     - Mitigation steps enumerated  
   - Set Chat ID to recipient group or user chat  
   - Connect input from `AI Agent` node  

9. **Connect Nodes in Sequence:**  
   Webhook → If → Code → Virus_Total → For_Gemini_Prompt → AI Agent → Send a text message  
   Google Gemini Chat Model & Simple Memory → (inputs to) AI Agent  

10. **Configure Credentials:**  
    - VirusTotal API key in HTTP Request node credentials  
    - Telegram Bot credentials in Telegram node  
    - Google Gemini API credentials as required by LangChain nodes  

11. **Test the Workflow:**  
    - Send sample Sophos alert POST data matching filter criteria to webhook URL  
    - Verify correct indicator extraction, VirusTotal query, AI analysis, and Telegram notification delivery  

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The webhook path must be secured and publicly accessible to receive Sophos alerts.                                                      | Webhook Node Configuration                                                                       |
| VirusTotal API usage requires valid API key with access to v3 endpoints. Rate limits apply.                                              | Virus_Total Node HTTP Request                                                                    |
| AI Agent prompt enforces strict usage of input data without assumptions to maintain factual integrity in security analysis.            | AI Agent Node                                                                                   |
| Telegram chat ID must be replaced with actual recipient chat or group ID for notification delivery.                                     | Send a text message Node                                                                         |
| The domain extraction logic relies on domain being enclosed in quotes in `event.name` field and containing a dot ('.') character.       | Code Node (Indicator Extraction)                                                                 |
| For detailed LangChain AI node setup and credential management, refer to n8n LangChain integration documentation.                      | n8n Docs: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                          |
| This workflow exemplifies combining automated threat intelligence enrichment with generative AI for enhanced SOC alert triage.         | General project context                                                                           |

---

**Disclaimer:** The provided text is generated exclusively from an n8n automated workflow. It complies strictly with all content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.