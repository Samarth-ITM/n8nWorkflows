Daily Insight Email from Structured Web Data with Firecrawl

https://n8nworkflows.xyz/workflows/daily-insight-email-from-structured-web-data-with-firecrawl-4654


# Daily Insight Email from Structured Web Data with Firecrawl

### 1. Workflow Overview

This workflow automates a daily process of extracting, formatting, and emailing structured web data related to notable congressional stock trades. It is designed for users who want to receive insights on significant congressional trading activity without manually scraping or analyzing web data.

**Target Use Cases:**  
- Business analysts monitoring market or industry news related to congressional trading  
- Researchers and founders automating competitive intelligence gathering  
- Users seeking automated, no-code web data extraction and summarization  

**Logical Blocks:**  
- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specified hour.  
- **1.2 Data Extraction Request:** Sends a POST request to Firecrawl API to start extracting structured trade data from a congress trading website.  
- **1.3 Extraction Completion Polling:** Waits and polls with GET requests to Firecrawl to check if data extraction is completed, looping until data is ready.  
- **1.4 Data Validation and Processing:** Checks if notable trade data is present; if not, waits and retries the GET request.  
- **1.5 Data Preparation:** Extracts and sets the relevant data fields for formatting.  
- **1.6 AI Formatting:** Uses OpenAI (via LangChain node) to format the raw data into a human-readable email content.  
- **1.7 Email Dispatch:** Sends the formatted insight email using Gmail node.  

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Starts the workflow every day at 10:00 AM to ensure timely scraping and email delivery.  
- **Nodes Involved:**  
  - Schedule Trigger  
- **Node Details:**  
  - Type: Schedule Trigger  
  - Configuration: Set to trigger at hour 10 every day (daily at 10 AM)  
  - Inputs: None (workflow start node)  
  - Outputs: POST Request node  
  - Edge Cases: Workflow will not run outside this schedule; time zone should be confirmed in n8n instance settings to avoid timing issues.  
  - No sub-workflows invoked.  

#### 1.2 Data Extraction Request

- **Overview:**  
  Sends a POST request to Firecrawl API to initiate extraction of structured data from the congress trading webpage. The request includes a detailed prompt and a JSON schema describing the expected data structure.  
- **Nodes Involved:**  
  - POST Request  
- **Node Details:**  
  - Type: HTTP Request (POST)  
  - Configuration:  
    - URL: `https://api.firecrawl.dev/v1/extract`  
    - Body: JSON including target URL pattern (`https://www.quiverquant.com/congresstrading/*`), prompt requesting notable trades over $50,000 in the past month, and a schema defining `notable_trades` array with fields (congress_member_name, party, stock_or_asset, amount, transaction_date).  
    - Authentication: HTTP Header with API key (genericCredentialType)  
  - Inputs: Schedule Trigger  
  - Outputs: Wait 60 Seconds node  
  - Edge Cases:  
    - API key invalid or expired → authentication failure  
    - Network issues → request timeout or failure  
    - Schema mismatch → extraction errors or malformed data  
  - No sub-workflows invoked.  

#### 1.3 Extraction Completion Polling

- **Overview:**  
  After the initial POST request, the workflow waits 60 seconds and then sends repeated GET requests to Firecrawl using the extraction job ID to check if the extraction is complete. This loop continues until data is available.  
- **Nodes Involved:**  
  - Wait 60 Seconds  
  - HTTP GET Request  
  - If (condition check)  
  - Wait 30 seconds (retry delay)  
- **Node Details:**  
  - **Wait 60 Seconds:**  
    - Type: Wait node  
    - Configuration: Waits for 60 seconds before sending GET request  
    - Inputs: POST Request  
    - Outputs: HTTP GET Request  
  - **HTTP GET Request:**  
    - Type: HTTP Request (GET)  
    - Configuration:  
      - URL built dynamically using the extraction job ID from POST Request response (`https://api.firecrawl.dev/v1/extract/{{ $('POST Request').item.json.id }}`)  
      - Authentication: same as POST Request  
    - Inputs: Wait 60 Seconds or Wait 30 seconds (retry)  
    - Outputs: If node  
    - Edge Cases:  
      - Job ID missing or invalid → 404 or error  
      - Network issues → timeout or failure  
  - **If Node:**  
    - Type: If conditional node  
    - Condition: Checks if `data.notable_trades` array is empty  
    - Inputs: HTTP GET Request  
    - Outputs:  
      - If true (empty data): Wait 30 seconds node (retry delay)  
      - If false (data available): Edit Fields node (continue processing)  
  - **Wait 30 seconds:**  
    - Type: Wait node  
    - Configuration: Waits 30 seconds before retrying GET request  
    - Inputs: If node (true path)  
    - Outputs: HTTP GET Request (loop back)  
- **Edge Cases:**  
  - Infinite loop risk if extraction never completes → consider adding max retries or timeout  
  - Network or API errors during polling → may cause failures or false negatives  
  - Job ID expiration → extraction job may be invalid after some time  
- **No sub-workflows invoked.**  

#### 1.4 Data Preparation

- **Overview:**  
  Extracts the `notable_trades` array from the GET request response and assigns it to a simplified data field for further use.  
- **Nodes Involved:**  
  - Edit Fields (Set node)  
- **Node Details:**  
  - Type: Set node  
  - Configuration: Assigns `data.notable_trades` from the response JSON to a string field (for consistency and easier referencing)  
  - Inputs: If node (false path) — i.e., when data is present  
  - Outputs: Formatting AI node  
  - Edge Cases:  
    - If expected field missing or malformed → data may be empty or invalid  
- **No sub-workflows invoked.**  

#### 1.5 AI Formatting

- **Overview:**  
  Uses OpenAI (via LangChain node) to transform raw notable trades data into a human-readable email format based on a prompt.  
- **Nodes Involved:**  
  - Formatting AI  
- **Node Details:**  
  - Type: OpenAI (LangChain) node  
  - Configuration:  
    - Model: GPT-4o (GPT-4 optimized)  
    - Prompt: "Please format these trade date {{ $json.data.notable_trades }} into an easily readable email format."  
    - Input: The notable trades data from Edit Fields node  
  - Inputs: Edit Fields  
  - Outputs: Gmail node  
  - Credentials: OpenAI API key (configured in credentials)  
  - Edge Cases:  
    - API key invalid or quota exceeded → failure or delayed response  
    - Prompt or data formatting errors → malformed output  
    - Model response latency or timeout  
- **No sub-workflows invoked.**  

#### 1.6 Email Dispatch

- **Overview:**  
  Sends the AI-generated formatted content as a plain text email to a specified recipient using Gmail node.  
- **Nodes Involved:**  
  - Gmail  
- **Node Details:**  
  - Type: Gmail node  
  - Configuration:  
    - Recipient email address (configured in parameters)  
    - Subject: "Congress Stock Update"  
    - Email body: Content from AI Formatting node (`{{$json.message.content}}`)  
    - Email type: plain text  
  - Inputs: Formatting AI node  
  - Outputs: None (end of workflow)  
  - Credentials: Gmail OAuth2 credentials  
  - Edge Cases:  
    - Authentication failure or expired Gmail token  
    - Email send limits or quota exceeded  
    - Invalid recipient email format  
- **No sub-workflows invoked.**  

---

### 3. Summary Table

| Node Name          | Node Type                  | Functional Role                   | Input Node(s)       | Output Node(s)        | Sticky Note                                                                                   |
|--------------------|----------------------------|---------------------------------|---------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger   | Schedule Trigger            | Daily workflow start trigger     | None                | POST Request          | Scheduled Trigger                                                                            |
| POST Request       | HTTP Request (POST)         | Initiates Firecrawl extraction   | Schedule Trigger    | Wait 60 Seconds       | POST Request to Firecrawl                                                                    |
| Wait 60 Seconds    | Wait                       | Delay before polling extraction  | POST Request        | HTTP GET Request      | Wait for Extract Request to Complete                                                         |
| HTTP GET Request   | HTTP Request (GET)          | Polls Firecrawl for extraction completion | Wait 60 Seconds, Wait 30 seconds | If                  | GET Request Loop                                                                            |
| If                 | If                         | Checks if extracted data is ready| HTTP GET Request    | Wait 30 seconds (if empty), Edit Fields (if data present) | GET Request Loop                                                                            |
| Wait 30 seconds    | Wait                       | Retry delay before next poll     | If (empty data)     | HTTP GET Request      | GET Request Loop                                                                            |
| Edit Fields        | Set                        | Prepares extracted trade data    | If (data present)   | Formatting AI         | Formatting Agent                                                                            |
| Formatting AI      | OpenAI (LangChain)          | Formats raw data into email text | Edit Fields         | Gmail                 | Formatting Agent                                                                            |
| Gmail              | Gmail                      | Sends formatted email            | Formatting AI       | None                  | Send to Inbox                                                                              |
| Sticky Note        | Sticky Note                | Visual label for Schedule Trigger| None                | None                  | Scheduled Trigger                                                                            |
| Sticky Note1       | Sticky Note                | Visual label for POST Request    | None                | None                  | POST Request to Firecrawl                                                                    |
| Sticky Note2       | Sticky Note                | Visual label for Wait 60 Seconds | None                | None                  | Wait for Extract Request to Complete                                                         |
| Sticky Note3       | Sticky Note                | Visual label for GET Request loop| None                | None                  | GET Request Loop                                                                            |
| Sticky Note4       | Sticky Note                | Visual label for AI Formatting   | None                | None                  | Formatting Agent                                                                            |
| Sticky Note5       | Sticky Note                | Visual label for Gmail node      | None                | None                  | Send to Inbox                                                                              |
| Sticky Note6       | Sticky Note                | Workflow overview and instructions| None                | None                  | 🔥 Daily Web Scraper & AI Summary with Firecrawl + Email Automation ... Watch Full Video Step-by-step Tutorial Here: https://www.youtube.com/@Automatewithmarc |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 10:00 AM (hour 10)  
   - No credentials needed  

2. **Create POST Request Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v1/extract`  
   - Body Content-Type: JSON  
   - Body (JSON):  
     ```json
     {
       "urls": ["https://www.quiverquant.com/congresstrading/*"],
       "prompt": "Extract all notable congress member trades over $50,000 USD in the past 1 month. Include the Congress Member's name, party, the Stock/Asset purchased or sold, the Amount of the transaction, and the date of the transaction.",
       "schema": {
         "type": "object",
         "properties": {
           "notable_trades": {
             "type": "array",
             "items": {
               "type": "object",
               "properties": {
                 "congress_member_name": {"type": "string"},
                 "party": {"type": "string"},
                 "stock_or_asset": {"type": "string"},
                 "amount": {"type": "number"},
                 "transaction_date": {"type": "string"}
               },
               "required": ["congress_member_name", "stock_or_asset", "amount", "transaction_date"]
             }
           }
         },
         "required": ["notable_trades"]
       }
     }
     ```  
   - Enable sending body as JSON  
   - Authentication: HTTP Header Auth with Firecrawl API key credential  
   - Connect Schedule Trigger → POST Request  

3. **Create Wait 60 Seconds Node:**  
   - Type: Wait  
   - Set duration: 60 seconds  
   - Connect POST Request → Wait 60 Seconds  

4. **Create HTTP GET Request Node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Dynamic expression:  
     ```
     https://api.firecrawl.dev/v1/extract/{{ $json["id"] }}
     ```  
     where `id` is from POST Request response (use expression: `{{ $('POST Request').item.json.id }}`)  
   - Authentication: same Firecrawl API key credential  
   - Connect Wait 60 Seconds → HTTP GET Request  
   - Also connect Wait 30 seconds → HTTP GET Request (for retry loop)  

5. **Create If Node:**  
   - Type: If (Conditional)  
   - Condition: Check if `data.notable_trades` is empty array  
     - Expression: `{{ $json.data.notable_trades }}`  
     - Condition: Array is empty  
   - Connect HTTP GET Request → If  

6. **Create Wait 30 seconds Node:**  
   - Type: Wait  
   - Duration: 30 seconds  
   - Connect If (true path, i.e., empty data) → Wait 30 seconds  
   - Connect Wait 30 seconds → HTTP GET Request (loop)  

7. **Create Edit Fields (Set) Node:**  
   - Type: Set  
   - Assign a new field `data.notable_trades` with value: `{{ $json.data.notable_trades }}` to simplify access  
   - Connect If (false path, i.e., data available) → Edit Fields  

8. **Create Formatting AI Node:**  
   - Type: OpenAI (LangChain)  
   - Model: GPT-4o  
   - Prompt message:  
     ```
     Please format these trade date  {{ $json.data.notable_trades }} into an easily readable email format.
     ```  
   - Connect Edit Fields → Formatting AI  
   - Credentials: OpenAI API key  

9. **Create Gmail Node:**  
   - Type: Gmail  
   - Parameters:  
     - Send To: set recipient email address  
     - Subject: "Congress Stock Update"  
     - Message: `{{ $json.message.content }}` (content from AI formatting)  
     - Email Type: Text  
   - Connect Formatting AI → Gmail  
   - Credentials: Gmail OAuth2 credentials  

10. **Testing and Validation:**  
    - Test execution at scheduled time or trigger manually  
    - Confirm Firecrawl API key validity  
    - Confirm Gmail credentials and email delivery  
    - Check for errors in HTTP requests, API responses, and AI formatting  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| 🔥 Daily Web Scraper & AI Summary with Firecrawl + Email Automation  Need to extract and summarize web content from a site that doesn’t have an API? This workflow runs daily to scrape a web page using Firecrawl, summarize the content with OpenAI, and send it directly to your email — fully automated. Watch Full Video Step-by-step Tutorial Here: https://www.youtube.com/@Automatewithmarc 🔧 How It Works Daily Trigger – Starts the workflow every 24 hours. Firecrawl Node – Crawls and extracts structured data from any web page you specify. OpenAI Node (Optional) – Processes and summarizes the raw content using a prompt you control. Gmail Node – Sends the final summary or content snapshot to your email inbox. ✅ Perfect For Business analysts tracking daily market or industry news Researchers and founders automating competitive intelligence Anyone who wants web data delivered without coding or scraping scripts 🪜 Setup Instructions Firecrawl API Key – Sign up and insert your key in the credentials. Update Target URL – Edit the URL in the Firecrawl node to your desired site. Customize the Prompt – Tailor the OpenAI prompt to extract the insights you want. Connect Gmail – Add your Gmail credentials and set your recipient email. 🧰 Built With Firecrawl (Web scraping without code) OpenAI (For summarizing and insight extraction) Gmail (Automated notifications) n8n (Workflow automation engine) | Workflow sticky note with detailed overview and setup video link |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively with n8n automation tool, respecting all applicable content policies. The workflow contains no illegal or offensive content and uses only publicly available and legal data.