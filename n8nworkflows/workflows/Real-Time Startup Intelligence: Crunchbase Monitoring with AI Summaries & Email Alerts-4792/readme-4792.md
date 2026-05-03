Real-Time Startup Intelligence: Crunchbase Monitoring with AI Summaries & Email Alerts

https://n8nworkflows.xyz/workflows/real-time-startup-intelligence--crunchbase-monitoring-with-ai-summaries---email-alerts-4792


# Real-Time Startup Intelligence: Crunchbase Monitoring with AI Summaries & Email Alerts

---

### 1. Workflow Overview

This workflow, titled **"Real-Time Startup Intelligence: Crunchbase Monitoring with AI Summaries & Email Alerts"**, is designed to automate the daily monitoring of startup company updates from Crunchbase. It retrieves the latest organization updates, processes the data, summarizes it using AI, and sends an email digest summarizing these updates.

**Target Use Cases:**
- Investors, startup founders, analysts, and marketers who want to stay informed about startup activities.
- Automating research and monitoring workflows to save time.
- Generating human-friendly summaries of complex Crunchbase data for easy consumption.

**Logical Blocks:**

- **1.1 Data Collection & Preprocessing**  
  Retrieval of updated company data from Crunchbase API and extracting relevant fields.

- **1.2 AI Processing and Summarization**  
  Feeding the simplified data into an AI agent (GPT-4) to generate a structured, readable summary.

- **1.3 Email Delivery**  
  Sending the generated summary via Gmail to specified recipients.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Data Collection & Preprocessing

**Overview:**  
This block fetches newly updated company data from Crunchbase and extracts key details into a simplified, structured format ready for AI summarization.

**Nodes Involved:**  
- Trigger Manual Test  
- Fetch Crunchbase Updates  
- Extract Company Details

---

**Node: Trigger Manual Test**  
- **Type & Role:** Manual Trigger node, used to start the workflow manually for testing or ad hoc runs.  
- **Configuration:** No parameters; simply triggers downstream nodes when clicked.  
- **Inputs/Outputs:** No inputs; outputs trigger the next node (Fetch Crunchbase Updates).  
- **Edge Cases:** In production, manual trigger should be replaced by a scheduled trigger (e.g., Cron node) for automation.

---

**Node: Fetch Crunchbase Updates**  
- **Type & Role:** HTTP Request node to call Crunchbase API for updated organizations.  
- **Configuration:**  
  - Method: GET  
  - URL: `https://api.crunchbase.com/api/v4/entities/organizations`  
  - Query Parameters:  
    - `user_key`: API key (placeholder `YOUR_API_KEY`)  
    - `updated_since`: fixed date `2025-06-05` (should be dynamic for live use)  
  - Headers: Accept JSON  
- **Inputs/Outputs:**  
  - Input: Trigger from manual node  
  - Output: JSON response with list of updated companies  
- **Edge Cases:**  
  - API key invalid or expired → authentication error  
  - Rate limiting by Crunchbase API  
  - Network timeouts or errors  
  - Empty response if no updates since given date  
- **Version:** HTTP Request node v4.2

---

**Node: Extract Company Details**  
- **Type & Role:** Set node used to simplify and extract essential company fields from the raw API response.  
- **Configuration:** Assigns fields from the first item in the response array:  
  - `name`  
  - `short_description`  
  - `categories`  
  - `city_name`  
  - `country_code`  
  - `updated_at`  
- **Key Expressions:** Uses expressions like `={{ $json.data.items[0].properties.name }}` to pick data from JSON path.  
- **Inputs/Outputs:**  
  - Input: JSON from Crunchbase API  
  - Output: Simplified JSON with key company details  
- **Edge Cases:**  
  - No items in API response (empty array) → expressions will fail or return undefined  
  - Missing fields or null values in API data  
- **Version:** Set node v3.4

---

#### 1.2 AI Processing and Summarization

**Overview:**  
This block takes extracted company details, sends them to an OpenAI GPT-4 based agent to generate a readable, professional summary formatted for email, and parses the structured JSON output.

**Nodes Involved:**  
- Summarizer Agent  
- OpenAI Chat Model  
- Structured Output Parser

---

**Node: OpenAI Chat Model**  
- **Type & Role:** Language model node (GPT-4o-mini) using OpenAI API to generate text output.  
- **Configuration:**  
  - Model: `gpt-4o-mini` (a GPT-4 variant)  
  - Credentials: OpenAI API key  
- **Inputs/Outputs:**  
  - Input: Prompt from Summarizer Agent node  
  - Output: Raw AI text response (expected JSON string)  
- **Edge Cases:**  
  - API authentication failure  
  - Rate limits or quota exceeded  
  - Model service downtime  
  - Invalid or malformed prompt causing poor or no output  
- **Version:** LangChain GPT node v1.2  

---

**Node: Summarizer Agent**  
- **Type & Role:** AI Agent node orchestrating prompt construction and invoking the language model.  
- **Configuration:**  
  - Input text template dynamically built from extracted company details (name, description, categories, location, updated date).  
  - System message instructs to produce an email-ready summary with specific formatting rules (bold company name, bullet points, etc.).  
  - Has output parser enabled to expect structured JSON.  
- **Inputs/Outputs:**  
  - Input: Simplified company data JSON from Extract Company Details  
  - Output: AI response to be parsed by Structured Output Parser  
- **Edge Cases:**  
  - Missing input data fields → incomplete summary  
  - AI returning text not matching expected JSON format  
  - Timeout or API errors when calling OpenAI  
- **Version:** LangChain Agent node v1.9

---

**Node: Structured Output Parser**  
- **Type & Role:** Ensures AI output is valid JSON and extracts `subject` and `body` fields for downstream use.  
- **Configuration:** Uses a JSON schema example to validate and parse the AI output.  
- **Inputs/Outputs:**  
  - Input: Raw AI output text from OpenAI Chat Model  
  - Output: Parsed JSON with `subject` and `body` strings  
- **Edge Cases:**  
  - AI output invalid JSON → parser error  
  - Missing keys in JSON object  
- **Version:** LangChain Output Parser v1.2

---

#### 1.3 Email Delivery

**Overview:**  
The final block sends the formatted summary email via Gmail using OAuth2 credentials.

**Nodes Involved:**  
- Send Email with Summary

---

**Node: Send Email with Summary**  
- **Type & Role:** Gmail node to send emails.  
- **Configuration:**  
  - Recipient: `shahkar.genai@gmail.com` (hardcoded)  
  - Subject: From `{{$json.output.subject}}` (parsed AI output)  
  - Message Body: From `{{$json.output.body}}` (parsed AI output)  
  - Credential: Gmail OAuth2 account configured  
- **Inputs/Outputs:**  
  - Input: Parsed JSON with subject and body from Structured Output Parser  
  - Output: Email send response status  
- **Edge Cases:**  
  - OAuth token expired or invalid  
  - Email send failures (invalid recipient, quota limits)  
  - Missing subject or body content  
- **Version:** Gmail node v2.1

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                          | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                              |
|-------------------------|---------------------------------|----------------------------------------|---------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Trigger Manual Test      | Manual Trigger                  | Manual workflow start                   | -                         | Fetch Crunchbase Updates   | In production, replace with Cron node for scheduled runs.                                              |
| Fetch Crunchbase Updates | HTTP Request                   | Fetch updated companies from Crunchbase| Trigger Manual Test       | Extract Company Details    | Calls Crunchbase API with user_key and updated_since parameters.                                       |
| Extract Company Details  | Set                            | Extract key company fields from API    | Fetch Crunchbase Updates  | Summarizer Agent           | Simplifies Crunchbase data to essential fields for AI summarization.                                   |
| OpenAI Chat Model        | LangChain OpenAI Chat Model    | Generate AI summary text                | Summarizer Agent          | Structured Output Parser   | Uses GPT-4o-mini model; outputs JSON string with email subject and body.                               |
| Summarizer Agent         | LangChain AI Agent             | Build prompt & orchestrate AI request  | Extract Company Details   | OpenAI Chat Model, Send Email with Summary | System prompt defines clear email summary format.                                           |
| Structured Output Parser | LangChain Output Parser        | Validate & parse AI JSON output         | OpenAI Chat Model         | Summarizer Agent           | Ensures AI response contains valid JSON with subject & body fields.                                   |
| Send Email with Summary  | Gmail                         | Send summary email                      | Summarizer Agent          | -                          | Sends email with AI-generated subject & body to specified Gmail address.                              |
| Sticky Note              | Sticky Note                   | Documentation and explanations          | -                         | -                          | Section 1: Data Collection & Preprocessing explanations.                                              |
| Sticky Note1             | Sticky Note                   | Documentation and explanations          | -                         | -                          | Section 2: AI Summary + Email Delivery explanations.                                                  |
| Sticky Note2             | Sticky Note                   | Documentation and explanations          | -                         | -                          | Section 3: Send email with summary.                                                                   |
| Sticky Note4             | Sticky Note                   | Workflow overview & instructions        | -                         | -                          | Comprehensive workflow overview and detailed notes, including usage tips.                             |
| Sticky Note9             | Sticky Note                   | Support and contact info                 | -                         | -                          | Workflow assistance contact and resource links.                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: `Trigger Manual Test`  
   - Purpose: Start the workflow manually for testing.  
   - No inputs or special parameters.

2. **Create an HTTP Request node:**  
   - Name: `Fetch Crunchbase Updates`  
   - Connect input from `Trigger Manual Test`.  
   - Method: GET  
   - URL: `https://api.crunchbase.com/api/v4/entities/organizations`  
   - Query Parameters:  
     - `user_key`: your Crunchbase API key (replace `"YOUR_API_KEY"`)  
     - `updated_since`: date string (for production, dynamically set to yesterday’s date)  
   - Headers: `Accept: application/json`  
   - Expected output: JSON with updated companies.

3. **Create a Set node:**  
   - Name: `Extract Company Details`  
   - Connect input from `Fetch Crunchbase Updates`.  
   - Configure assignments to extract fields from first company item:  
     - `name`: `={{ $json.data.items[0].properties.name }}`  
     - `short_description`: `={{ $json.data.items[0].properties.short_description }}`  
     - `categories`: `={{ $json.data.items[0].properties.categories }}`  
     - `city_name`: `={{ $json.data.items[0].properties.city_name }}`  
     - `country_code`: `={{ $json.data.items[0].properties.country_code }}`  
     - `updated_at`: `={{ $json.data.items[0].properties.updated_at }}`

4. **Create a LangChain AI Agent node:**  
   - Name: `Summarizer Agent`  
   - Connect input from `Extract Company Details`.  
   - Text input template:  
     ```
     Name: {{ $json.data.items[0].properties.name }}
     Description: {{ $json.data.items[0].properties.short_description }}
     categories: {{ $json.data.items[0].properties.categories }}
     city name: {{ $json.data.items[0].properties.city_name }}
     country name: {{ $json.data.items[0].properties.country_code }}
     updated at: {{ $json.data.items[0].properties.updated_at }}
     ```  
   - System message:  
     ```
     You are a helpful assistant that writes email-ready summaries of recent company updates from Crunchbase.

     The input contains data about recently updated organizations including their name, description, categories, location, and update time.

     Summarize the companies clearly in a professional tone. Include:
     - Company name (bold or highlighted)
     - One-line description
     - Industry/categories (comma-separated)
     - Location (City, Country)
     - Last updated time (formatted)

     Format it for a human reader in plain text or HTML for email use. Use line breaks or bullet points to separate companies.
     Avoid repeating keys or technical jargon.
     ```  
   - Enable output parser.

5. **Create a LangChain OpenAI Chat Model node:**  
   - Name: `OpenAI Chat Model`  
   - Connect as AI language model input for `Summarizer Agent`.  
   - Model: `gpt-4o-mini` (or GPT-4 / GPT-3.5 depending on availability)  
   - Credentials: Configure your OpenAI API credentials.

6. **Create a LangChain Structured Output Parser node:**  
   - Name: `Structured Output Parser`  
   - Connect AI output of `OpenAI Chat Model` to this node’s input.  
   - Configure JSON schema example:  
     ```json
     {
       "subject": "🚀 Crunchbase Company Updates - June 5, 2025",
       "body": "🚀 Here's a summary of recent company updates from Crunchbase:\n\n🔹 **OpenAI**\nAI research and deployment company.\nIndustry: Artificial Intelligence, Machine Learning\nLocation: San Francisco, USA\nLast Updated: June 5, 2025\n\n🔹 **DeepMind**\nSolving intelligence to benefit humanity.\nIndustry: Deep Learning, Healthcare\nLocation: London, GBR\nLast Updated: June 5, 2025"
     }
     ```
   - Output: Parsed JSON with subject & body.

7. **Connect `Structured Output Parser` output back to `Summarizer Agent` output parser input.**

8. **Create a Gmail node:**  
   - Name: `Send Email with Summary`  
   - Connect input from `Summarizer Agent` (main output after parsing).  
   - Recipient: `shahkar.genai@gmail.com` (replace with your intended recipient).  
   - Subject: `={{ $json.output.subject }}`  
   - Message: `={{ $json.output.body }}`  
   - Credentials: Configure Gmail OAuth2 credentials for sending email.

9. **(Optional) Replace Manual Trigger with a Cron node:**  
   - Schedule daily runs (e.g., every day at 8 AM).  
   - Connect to start of workflow, replacing manual trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                                                                                 |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow assistance and support contact: Yaron@nofluff.online                                                | Contact for questions or help                                                                                                                                  |
| Video tutorials and tips by Yaron Been                                                                       | YouTube: https://www.youtube.com/@YaronBeen/videos                                                                                                            |
| LinkedIn profile of Yaron Been                                                                                | https://www.linkedin.com/in/yaronbeen/                                                                                                                        |
| Crunchbase API documentation                                                                                  | https://data.crunchbase.com/docs                                                                                                                               |
| Use a Cron node in production to automate daily workflow execution                                            | Replace manual trigger with Cron for scheduled execution                                                                                                      |
| AI system prompt ensures email-ready, human-readable summaries with clear formatting                           | Critical for consistent output and parsing                                                                                                                    |
| Be aware of API rate limits and authentication expiry for Crunchbase and OpenAI APIs                           | Common integration pitfalls                                                                                                                                    |
| Ensure email credentials (Gmail OAuth2) are valid and have send permissions                                   | Prevents email sending failures                                                                                                                                |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---