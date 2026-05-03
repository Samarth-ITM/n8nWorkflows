Automated Weekly Tech Stack Reports with BuiltWith, GPT-4o & Gmail

https://n8nworkflows.xyz/workflows/automated-weekly-tech-stack-reports-with-builtwith--gpt-4o---gmail-4788


# Automated Weekly Tech Stack Reports with BuiltWith, GPT-4o & Gmail

### 1. Workflow Overview

This workflow, titled **"Automated Weekly Tech Stack Reports with BuiltWith, GPT-4o & Gmail"**, is designed to automate the process of collecting, analyzing, and summarizing the technology stacks used by a list of ecommerce domains on a weekly basis. It leverages BuiltWith’s API for tech stack data, OpenAI’s GPT-4o model for natural-language summarization, and Gmail for delivering the final report via email.

The workflow is structured into three major logical blocks reflecting the data flow and functional roles:

- **1.1 Data Collection Stage**  
  Automates the weekly trigger and fetches a dynamic list of ecommerce website domains from Google Sheets.

- **1.2 Tech Stack Intelligence Stage**  
  Queries BuiltWith API for each domain to retrieve raw technology stack data, then processes and extracts the most relevant tech details.

- **1.3 Smart Summary & Delivery Stage**  
  Uses OpenAI GPT-4o to generate human-friendly summaries of the tech stacks and emails the consolidated report automatically.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Collection Stage

**Overview:**  
This block triggers the workflow every Monday at 9 AM and pulls a list of ecommerce domains from a Google Sheet. It sets the foundation for the entire process by providing input domains dynamically.

**Nodes Involved:**  
- Weekly Trigger  
- Fetch Domain List  

**Node Details:**

- **Weekly Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow automatically every Monday at 9:00 AM.  
  - Configuration: Weekly interval, trigger day = Monday, trigger hour = 9.  
  - Input: None (trigger node)  
  - Output: Connects to Fetch Domain List node  
  - Failures: Workflow won’t run if schedule misconfigured or n8n instance offline.  
  - Notes: Critical for automation, no manual intervention required.

- **Fetch Domain List**  
  - Type: Google Sheets  
  - Role: Reads ecommerce domains from a specific Google Sheet and sheet tab.  
  - Configuration:  
    - Document ID: linked to the Google Sheet containing ecommerce domains list.  
    - Sheet Name: “gid=0” (default first sheet)  
  - Credentials: Google Sheets OAuth2 (ensure valid and authorized).  
  - Input: Connected from Weekly Trigger  
  - Output: Passes domain data to BuiltWith API request node.  
  - Failures: Authentication errors, API quota limits, or sheet permission issues.  
  - Edge Case: Empty or malformed sheet data may stop downstream processing.

---

#### 1.2 Tech Stack Intelligence Stage

**Overview:**  
This block sends each domain from the sheet to BuiltWith API to retrieve detailed tech stack information and then extracts the key tech insights from the API’s complex JSON response.

**Nodes Involved:**  
- Get Tech Stack (BuiltWith API)  
- Extract Tech Stack Info  

**Node Details:**

- **Get Tech Stack (BuiltWith API)**  
  - Type: HTTP Request  
  - Role: Queries BuiltWith’s API to get technology data for each domain.  
  - Configuration:  
    - Method: GET  
    - URL: https://api.builtwith.com/v21/api.json  
    - Query Parameters:  
      - KEY: API key (replace `"YOUR_API_KEY"` with valid key)  
      - LOOKUP: domain name from the Google Sheet, expression `={{ $json['Domain '] }}` (note trailing space in key)  
  - Input: Domains list from Fetch Domain List  
  - Output: Raw JSON response with tech stack data to parsing node  
  - Failures: Invalid API key, rate limit exceeded, network errors, or missing domain parameter.  
  - Edge Cases: Domains with no data or unexpected API response format.

- **Extract Tech Stack Info**  
  - Type: Code (JavaScript)  
  - Role: Parses BuiltWith API JSON response to extract the first relevant technology group and its details.  
  - Configuration: Custom JS code that:  
    - Takes the first result item  
    - Extracts domain and URL info  
    - Iterates through Groups to find the first technology entry  
    - Returns simplified object with fields: Technology, Category, First Detected, Last Detected, Domain, URL  
  - Input: Raw API response from BuiltWith node  
  - Output: Cleaned and simplified tech stack data for AI summarization  
  - Failures: Null or undefined properties, JSON structure changes, empty group arrays.  
  - Edge Cases: Domains with no tech groups or incomplete data.

---

#### 1.3 Smart Summary & Delivery Stage

**Overview:**  
This block generates a natural language summary of each domain’s technology stack using an AI language model and then sends the compiled summary via Gmail.

**Nodes Involved:**  
- OpenAI Chat Model  
- Generate Stack Summary (AI)  
- Send Summary Email  

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Provides GPT-4o-mini model to process prompt data for text generation.  
  - Configuration:  
    - Model: `gpt-4o-mini` (lightweight GPT-4 variant)  
    - No additional options configured  
  - Credentials: OpenAI API key required and configured  
  - Input: Connected internally to Generate Stack Summary (AI) node via ai_languageModel input  
  - Output: Generated text to Generate Stack Summary (AI) node  
  - Failures: API key errors, rate limits, network timeouts, or model unavailability.

- **Generate Stack Summary (AI)**  
  - Type: Langchain Agent  
  - Role: Constructs prompt with extracted tech data and calls AI model to generate the summary.  
  - Configuration:  
    - Prompt uses template text incorporating domain data fields: Domain, Technology, Category, First Detected, Last Detected, URL.  
    - Prompt example:  
      ```
      Provide summary of the data that is scraped from BuiltWith

      Domain: {{ $json.Domain }}
      Technology: {{ $json.Technology }}
      Category: {{ $json.Category }}
      First Detected: {{ $json['First Detected'] }}
      Last Detected: {{ $json['Last Detected'] }}
      URL: {{ $json.URL }}
      ```  
  - Inputs: Simplified tech stack JSON from Extract Tech Stack Info node  
  - Outputs: AI-generated summary text to Send Summary Email node  
  - Failures: Expression errors, AI API errors, malformed inputs.

- **Send Summary Email**  
  - Type: Gmail  
  - Role: Sends the AI-generated summary email to specified recipients.  
  - Configuration:  
    - Recipient: `shahkar.genai@gmail.com` (hardcoded)  
    - Subject: “Weekly BuitlWith Ecommerce Platform Summary” (note typo “BuitlWith”)  
    - Message: Set to the AI-generated summary text expression `={{ $json.output }}`  
    - Email Type: Text (plain text)  
    - Append Attribution: Disabled  
  - Credentials: Gmail OAuth2 account with send email permission  
  - Input: AI-generated summary from Generate Stack Summary (AI) node  
  - Output: None (final node)  
  - Failures: Authentication errors, quota limits, invalid email address, network issues.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                  | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                     |
|-----------------------------|---------------------------------|--------------------------------|-----------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Weekly Trigger              | Schedule Trigger                | Automates weekly start          | None                        | Fetch Domain List             | ## 📦 **Section 1: Data Collection Stage** — Triggers every Monday 9 AM to start workflow       |
| Fetch Domain List           | Google Sheets                   | Fetches ecommerce domains       | Weekly Trigger              | Get Tech Stack (BuiltWith API) | ## 📦 **Section 1: Data Collection Stage** — Pulls domains from Google Sheets                    |
| Get Tech Stack (BuiltWith API) | HTTP Request                  | Calls BuiltWith API             | Fetch Domain List           | Extract Tech Stack Info       | # 🧠 **Section 2: Tech Stack Intelligence Stage** — Fetches tech stack info from BuiltWith API |
| Extract Tech Stack Info     | Code (JavaScript)               | Parses and extracts tech info   | Get Tech Stack (BuiltWith API) | Generate Stack Summary (AI)   | # 🧠 **Section 2: Tech Stack Intelligence Stage** — Cleans and extracts relevant tech data      |
| OpenAI Chat Model           | Langchain LM Chat OpenAI        | Provides GPT-4o mini model      | Connected internally to Generate Stack Summary (AI) | Generate Stack Summary (AI) | # 💌 **Section 3: Smart Summary & Delivery** — AI model for natural language summarization      |
| Generate Stack Summary (AI) | Langchain Agent                 | Generates natural language summary | Extract Tech Stack Info, OpenAI Chat Model | Send Summary Email            | # 💌 **Section 3: Smart Summary & Delivery** — Summarizes tech stack using AI                   |
| Send Summary Email          | Gmail                          | Sends summary email             | Generate Stack Summary (AI) | None                         | # 💌 **Section 3: Smart Summary & Delivery** — Emails the AI-generated summary                  |
| Sticky Note                 | Sticky Note                    | Documentation and explanation   | None                        | None                         | Various notes covering sections 1, 2, and 3, including workflow assistance and links           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: `Weekly Trigger`  
   - Configure to trigger every week on Monday at 9:00 AM.

2. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Name: `Fetch Domain List`  
   - Connect input from `Weekly Trigger` node.  
   - Configure:  
     - OAuth2 credentials for Google Sheets.  
     - Set Document ID to your Google Sheet containing ecommerce domains.  
     - Set Sheet Name or GID to the relevant sheet tab (e.g., `gid=0`).  
   - This node reads the domains from the sheet to feed into the next step.

3. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Name: `Get Tech Stack (BuiltWith API)`  
   - Connect input from `Fetch Domain List` node.  
   - Configure as GET request to `https://api.builtwith.com/v21/api.json`.  
   - Under Query Parameters, add:  
     - `KEY` with your BuiltWith API key.  
     - `LOOKUP` with an expression referencing domain field from Google Sheets (watch for exact field name, e.g., `={{ $json['Domain '] }}` including any trailing spaces).  
   - Set appropriate timeout and retry settings to handle API rate limits.

4. **Create Code Node**  
   - Type: Code (JavaScript)  
   - Name: `Extract Tech Stack Info`  
   - Connect input from `Get Tech Stack (BuiltWith API)` node.  
   - Paste the JavaScript code that:  
     - Extracts first available tech group and first tech entry.  
     - Returns object with Technology, Category, First Detected, Last Detected, Domain, URL.  
   - Validate the code for your specific API response structure.

5. **Create Langchain LM Chat OpenAI Node**  
   - Type: Langchain LM Chat OpenAI  
   - Name: `OpenAI Chat Model`  
   - Configure with OpenAI API credentials.  
   - Select model `gpt-4o-mini` or preferred GPT-4 variant.  
   - No special options needed.

6. **Create Langchain Agent Node**  
   - Type: Langchain Agent  
   - Name: `Generate Stack Summary (AI)`  
   - Connect inputs from both `Extract Tech Stack Info` (main input) and `OpenAI Chat Model` (ai_languageModel input).  
   - Configure prompt with the template text incorporating extracted fields:  
     ```
     Provide summary of the data that is scraped from BuiltWith

     Domain: {{ $json.Domain }}
     Technology: {{ $json.Technology }}
     Category: {{ $json.Category }}
     First Detected: {{ $json['First Detected'] }}
     Last Detected: {{ $json['Last Detected'] }}
     URL: {{ $json.URL }}
     ```
   - Set prompt type to "define" or equivalent.

7. **Create Gmail Node**  
   - Type: Gmail  
   - Name: `Send Summary Email`  
   - Connect input from `Generate Stack Summary (AI)` node.  
   - Configure with Gmail OAuth2 credentials with send email permissions.  
   - Set recipient email (e.g., `shahkar.genai@gmail.com`).  
   - Set subject: `"Weekly BuiltWith Ecommerce Platform Summary"` (correct typo if desired).  
   - Set message body to the expression referencing AI output text, e.g. `={{ $json.output }}`.  
   - Choose email type as plain text.  
   - Disable append attribution if not desired.

8. **Link All Nodes in Sequence**  
   - `Weekly Trigger` → `Fetch Domain List` → `Get Tech Stack (BuiltWith API)` → `Extract Tech Stack Info` → `Generate Stack Summary (AI)` → `Send Summary Email`  
   - Connect `OpenAI Chat Model` node as the AI language model input to `Generate Stack Summary (AI)`.

9. **Test and Validate**  
   - Run the workflow manually to check data flows and outputs.  
   - Verify API keys and credentials are correct.  
   - Confirm the Google Sheets document contains valid domain list data.  
   - Monitor for API limits or errors and adjust retry/timeouts as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow assistance and support contact: Yaron@nofluff.online. Explore additional tips and tutorials on YouTube and LinkedIn: https://www.youtube.com/@YaronBeen/videos and https://www.linkedin.com/in/yaronbeen/                  | Workflow assistance and creator contact                                                      |
| This workflow is a zero-touch, AI-powered automation specifically designed for ecommerce tech stack analysis, but it can be adapted for other industries by updating the domain list and adjusting parsing logic accordingly.        | General workflow purpose and adaptability                                                   |
| The BuiltWith API documentation: https://builtwith.com/api                                                                                                                                                                       | For API reference and advanced usage                                                        |
| OpenAI API documentation: https://platform.openai.com/docs/api-reference                                                                                                                                                         | For understanding AI model parameters and error handling                                   |
| Important: Ensure Google Sheets credentials have read permission on the target Sheet. Gmail credentials require send mail permission. OpenAI and BuiltWith API keys must be valid and not exceed usage quotas.                      | Credentials and permissions reminder                                                        |
| The domain field in Google Sheets contains a trailing space in the key (`"Domain "`). Confirm exact field names in your sheet or adjust expressions accordingly to avoid errors.                                                  | Common pitfall with field names                                                             |

---

This documentation enables advanced users and automation agents to fully understand, reproduce, and maintain the **Automated Weekly Tech Stack Reports with BuiltWith, GPT-4o & Gmail** workflow. It details each node’s purpose, configuration, and integration points while highlighting potential failure scenarios and best practices.