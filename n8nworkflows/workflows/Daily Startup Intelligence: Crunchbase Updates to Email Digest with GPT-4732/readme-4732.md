Daily Startup Intelligence: Crunchbase Updates to Email Digest with GPT

https://n8nworkflows.xyz/workflows/daily-startup-intelligence--crunchbase-updates-to-email-digest-with-gpt-4732


# Daily Startup Intelligence: Crunchbase Updates to Email Digest with GPT

### 1. Workflow Overview

This workflow, titled **"Daily Startup Intelligence: Crunchbase Updates to Email Digest with GPT"**, automates the daily fetching, summarizing, and emailing of updated startup company data from Crunchbase. It is designed for startup founders, investors, analysts, and marketers who want to stay informed about recent company activity without manual research.

The workflow consists of three main logical blocks:

- **1.1 Data Collection & Preprocessing**  
  Automates retrieval of recently updated company data from Crunchbase API and simplifies the data structure for downstream processing.

- **1.2 AI Processing & Summary Generation**  
  Uses OpenAI GPT (GPT-4o-mini) via LangChain to generate a professional, human-readable summary of the company updates, structured as a JSON email draft.

- **1.3 Email Dispatch**  
  Sends the AI-generated summary via Gmail to a specified email address as a concise daily digest.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Data Collection & Preprocessing

**Overview:**  
This block fetches updated organization data from Crunchbase and extracts key company details into a simplified format ready for AI summarization.

**Nodes Involved:**  
- Trigger Manual Test  
- Fetch Crunchbase Updates  
- Extract Company Details

**Node Details:**

- **Trigger Manual Test**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow for testing or manual execution.  
  - *Configuration:* No parameters set; simply triggers downstream nodes when manually activated.  
  - *Connections:* Outputs to `Fetch Crunchbase Updates`.  
  - *Edge Cases:* None; manual trigger means no scheduled run. For production, should be replaced by a Cron node to automate.  

- **Fetch Crunchbase Updates**  
  - *Type:* HTTP Request  
  - *Role:* Calls Crunchbase API endpoint to retrieve organizations updated since a given date.  
  - *Configuration:*  
    - URL: `https://api.crunchbase.com/api/v4/entities/organizations`  
    - Method: GET  
    - Query Parameters:  
      - `user_key`: API key placeholder (`YOUR_API_KEY`) - must be replaced by a valid Crunchbase API key.  
      - `updated_since`: Fixed date `"2025-06-05"` (should be dynamic in production).  
    - Headers: Accept: application/json  
  - *Connections:* Inputs from Manual Trigger; outputs JSON response to `Extract Company Details`.  
  - *Edge Cases:*  
    - API key missing or invalid → authentication errors.  
    - Network timeout or API rate limiting.  
    - Empty or malformed API response.  

- **Extract Company Details**  
  - *Type:* Set (Data Transformation)  
  - *Role:* Simplifies Crunchbase API response, extracting only essential properties (name, description, categories, city, country, last updated).  
  - *Configuration:* Assignments use JSONPath expressions to copy specific nested properties from the API response to a flatter structure.  
  - *Key Expressions:*  
    - `data.items[0].properties.name`  
    - `data.items[0].properties.short_description`  
    - `data.items[0].properties.categories`  
    - `data.items[0].properties.city_name`  
    - `data.items[0].properties.country_code`  
    - `data.items[0].properties.updated_at`  
  - *Connections:* Receives data from `Fetch Crunchbase Updates`; outputs simplified data to `Summarizer Agent`.  
  - *Edge Cases:*  
    - If `data.items` is empty or missing, expressions will fail or return undefined.  
    - If properties are missing or null in API response, summary may lack information.

---

#### 2.2 Block 2: AI Processing & Summary Generation

**Overview:**  
This block passes cleaned company data to an AI language model to generate a professionally formatted email summary, ensuring the output is a valid JSON object.

**Nodes Involved:**  
- Summarizer Agent  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**

- **Summarizer Agent**  
  - *Type:* LangChain Agent (AI Agent)  
  - *Role:* Coordinates AI summarization by sending company data as prompt text, applying a system message to guide summary style and format.  
  - *Configuration:*  
    - Input Text Template: Constructs a multi-line string with company details from extracted data fields.  
    - System Message:  
      - Instructs AI to summarize company updates in clear, professional language.  
      - Specifies format including bold company names, one-line description, industry, location, last updated date.  
      - Output is plain text or HTML suitable for email.  
    - Output Parsing: Enabled to work with a downstream structured output parser.  
  - *Connections:*  
    - Input: Receives simplified company data from `Extract Company Details`.  
    - Output: Passes AI output to `Send Email with Summary`.  
    - AI Language Model: Connected downstream to `OpenAI Chat Model`.  
    - Output Parser: Connected downstream to `Structured Output Parser`.  
  - *Edge Cases:*  
    - AI model might fail to generate well-formed JSON without parser enforcement.  
    - API usage limits or credential issues with OpenAI.  
    - Prompt template errors or missing data cause unclear summaries.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat model node  
  - *Role:* Calls OpenAI GPT-4o-mini model to generate text based on prompt from `Summarizer Agent`.  
  - *Configuration:*  
    - Model: `gpt-4o-mini` selected, a variant of GPT-4 optimized for cost and speed.  
    - Credentials: Uses OpenAI API key stored in `OpenAi account 2`.  
  - *Connections:*  
    - Input: From `Summarizer Agent`.  
    - Output: To `Summarizer Agent` as the language model response.  
  - *Edge Cases:*  
    - API key invalid or quota exceeded.  
    - Timeout or network errors.  
    - Model returns unexpected or malformed output.

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser (Structured)  
  - *Role:* Parses AI-generated text into a structured JSON object with `subject` and `body` fields for email use.  
  - *Configuration:* Uses a JSON schema example to enforce output format:  
    ```json
    {
      "subject": "🚀 Crunchbase Company Updates - June 5, 2025",
      "body": "Formatted summary text here..."
    }
    ```  
  - *Connections:*  
    - Input: Receives AI raw output from `Summarizer Agent`.  
    - Output: Feeds parsed JSON summary back to `Summarizer Agent` node outputs.  
  - *Edge Cases:*  
    - AI output not valid JSON → parser errors.  
    - Malformed or incomplete summary JSON breaks downstream email formatting.

---

#### 2.3 Block 3: Email Dispatch

**Overview:**  
Final step sends the AI-generated company update summary as an email using Gmail.

**Nodes Involved:**  
- Send Email with Summary

**Node Details:**

- **Send Email with Summary**  
  - *Type:* Gmail node (OAuth2)  
  - *Role:* Sends an email with subject and body fields generated by the AI summary.  
  - *Configuration:*  
    - Recipient: Fixed to `shahkar.genai@gmail.com` (replace with desired recipient).  
    - Subject: Expression referencing parsed summary subject (`{{$json.output.subject}}`).  
    - Message Body: Expression referencing parsed summary body (`{{$json.output.body}}`).  
    - Options: Attribution disabled (no appended n8n footer).  
    - Credentials: Gmail OAuth2 credentials stored with name `Gmail account`.  
  - *Connections:*  
    - Input: From `Summarizer Agent` output containing parsed JSON summary.  
  - *Edge Cases:*  
    - Gmail OAuth token expired or invalid → authentication failure.  
    - Email sending limits or connectivity errors.  
    - Malformed subject or body causing email rendering issues.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                          | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                      |
|-------------------------|----------------------------------|----------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Trigger Manual Test      | Manual Trigger                   | Starts workflow manually for testing   | —                      | Fetch Crunchbase Updates | ✅ Manual trigger for testing; replace with Cron for automation                                 |
| Fetch Crunchbase Updates | HTTP Request                    | Fetches updated companies from Crunchbase API | Trigger Manual Test     | Extract Company Details  | 🌐 Fetches companies updated since a date; requires Crunchbase API key                         |
| Extract Company Details  | Set                             | Extracts and simplifies company data   | Fetch Crunchbase Updates | Summarizer Agent        | ✏️ Extracts essential company fields for AI summarization                                     |
| Summarizer Agent        | LangChain AI Agent               | Generates summary text from company data | Extract Company Details | Send Email with Summary  | 🤖 Coordinates AI summary generation and parsing                                              |
| OpenAI Chat Model        | LangChain OpenAI Chat           | Calls GPT-4o-mini to generate text     | Summarizer Agent        | Summarizer Agent         | 🤖 Uses GPT-4o-mini via OpenAI credentials                                                    |
| Structured Output Parser | LangChain Output Parser (JSON)  | Parses AI output into structured JSON  | Summarizer Agent        | Summarizer Agent         | 🔎 Ensures AI response is valid JSON with subject and body                                    |
| Send Email with Summary  | Gmail                           | Sends email with summary                | Summarizer Agent        | —                       | 📧 Sends email to configured recipient with AI-generated subject and body                      |
| Sticky Note              | Sticky Note                     | Documentation section 1                 | —                      | —                       | Section 1: Data Collection & Preprocessing explanation                                        |
| Sticky Note1             | Sticky Note                     | Documentation section 2                 | —                      | —                       | Section 2: AI Summary + Email Delivery explanation                                            |
| Sticky Note2             | Sticky Note                     | Documentation section 3                 | —                      | —                       | Section 3: Email sending explanation                                                          |
| Sticky Note4             | Sticky Note                     | Workflow overview and detailed notes   | —                      | —                       | Comprehensive workflow description with tips                                                  |
| Sticky Note9             | Sticky Note                     | Workflow assistance and contact info   | —                      | —                       | Contact info and resources for support                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing.  
   - Leave parameters default (no configuration).  

2. **Create HTTP Request Node (Fetch Crunchbase Updates)**  
   - Connect input from Manual Trigger.  
   - Set HTTP Method: `GET`  
   - URL: `https://api.crunchbase.com/api/v4/entities/organizations`  
   - Query Parameters:  
     - `user_key`: Your Crunchbase API key (replace `"YOUR_API_KEY"` with a valid key).  
     - `updated_since`: Use a dynamic date expression for the previous day or fixed date (e.g., `"2025-06-05"`).  
   - Headers:  
     - `Accept: application/json`  
   - Save credentials for Crunchbase API if needed (basic auth or API key in query).  

3. **Create Set Node (Extract Company Details)**  
   - Connect input from HTTP Request node.  
   - Add assignments to map:  
     - `data.items[0].properties.name` → as string  
     - `data.items[0].properties.short_description` → string  
     - `data.items[0].properties.categories` → string or array (depending on output)  
     - `data.items[0].properties.city_name` → string  
     - `data.items[0].properties.country_code` → string  
     - `data.items[0].properties.updated_at` → string  
   - Use expressions referencing `$json` to extract these fields from the HTTP response JSON.  

4. **Create LangChain Agent Node (Summarizer Agent)**  
   - Connect input from Set node.  
   - In "text" parameter, create a multi-line template:  
     ```
     Name: {{ $json.data.items[0].properties.name }}
     Description: {{ $json.data.items[0].properties.short_description }}
     categories: {{ $json.data.items[0].properties.categories }}
     city name: {{ $json.data.items[0].properties.city_name }}
     country name: {{ $json.data.items[0].properties.country_code }}
     updated at: {{ $json.data.items[0].properties.updated_at }}
     ```  
   - Set system message instructing AI to create a professional email-ready summary including bold company names, description, categories, location, and last update date with line breaks.  
   - Enable output parser.  

5. **Create OpenAI Chat Model Node**  
   - Connect as AI language model to `Summarizer Agent`.  
   - Select model `gpt-4o-mini`.  
   - Provide OpenAI API credentials (create or select existing OpenAI API credentials).  

6. **Create Structured Output Parser Node**  
   - Connect as AI output parser to `Summarizer Agent`.  
   - Set JSON schema example with `subject` and `body` fields matching expected email content format.  

7. **Connect Summarizer Agent output to Send Email Node**  
   - Create Gmail node.  
   - Connect input from Summarizer Agent node's main output.  
   - Configure email parameters:  
     - To: desired recipient email (e.g., `shahkar.genai@gmail.com`)  
     - Subject: Set expression `{{$json.output.subject}}`  
     - Message: Set expression `{{$json.output.body}}`  
   - Use Gmail OAuth2 credentials (set up OAuth2 credentials in n8n for Gmail).  
   - Disable appending attribution footer for cleaner emails.  

8. **(Optional) Replace Manual Trigger with Cron Node**  
   - To automate daily execution, create a Cron node configured to run at a preferred time (e.g., 8:00 AM daily).  
   - Connect Cron node output to `Fetch Crunchbase Updates`.  

9. **Test the workflow**  
   - Trigger manually or wait for scheduled Cron trigger.  
   - Verify Crunchbase API response, AI summary generation, and email delivery.  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| To automate daily runs, replace the manual trigger node with a Cron node scheduled for your desired time. | Section 1 manual trigger note; production best practice                                                |
| Crunchbase API documentation: https://data.crunchbase.com/docs                                            | Official Crunchbase API docs for parameter details and authentication                                  |
| OpenAI API key must have permissions and sufficient quota for GPT-4o-mini usage                            | Token management and billing for OpenAI API                                                             |
| Gmail OAuth2 credentials setup required to send emails securely                                          | Gmail API and OAuth2 setup in n8n credential manager                                                    |
| Workflow assistance and support contact: Yaron@nofluff.online                                            | Contact for workflow support                                                                            |
| Additional tutorials and videos available at: https://www.youtube.com/@YaronBeen/videos                   | Video resources for n8n workflows                                                                       |
| LinkedIn profile for further tips and professional networking: https://www.linkedin.com/in/yaronbeen/     | Professional contact and resources                                                                      |

---

**Disclaimer:**  
This document is based exclusively on an n8n workflow automation respecting all content policies. It processes only legal and publicly available data.