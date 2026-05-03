Goodreads Quote Extraction with Bright Data and Gemini

https://n8nworkflows.xyz/workflows/goodreads-quote-extraction-with-bright-data-and-gemini-4438


# Goodreads Quote Extraction with Bright Data and Gemini

### 1. Workflow Overview

This workflow automates the extraction of quotes from Goodreads pages tagged with a specific theme, leveraging the Bright Data proxy service for web scraping and Google's Gemini AI model for natural language understanding and extraction. It targets users who want to programmatically gather thematic quotes from Goodreads, process the raw scraped content with AI to cleanly extract quotes, and structure the data for further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger and setting initial parameters such as the target URL and proxy zone.
- **1.2 Web Scraping via Bright Data:** Using Bright Data's API to perform a proxied web request to Goodreads, returning raw page content.
- **1.3 AI Processing with Google Gemini:** Applying an AI chat model to interpret the scraped content.
- **1.4 Quote Extraction:** Using an AI-powered information extractor node to parse and extract quotes from the AI-processed data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block initiates the workflow manually and sets the static fields required for the web scraping request, specifically the Goodreads URL to scrape and the Bright Data proxy zone.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set the fields

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution of the workflow.  
    - Configuration: No parameters; simply triggers workflow start.  
    - Inputs: None  
    - Outputs: Connected to "Set the fields".  
    - Edge cases: None typical; manual trigger ensures controlled start.

  - **Set the fields**  
    - Type: Set Node  
    - Role: Defines fixed input parameters for the scraping operation.  
    - Configuration:  
      - Sets `"url"` to `https://www.goodreads.com/quotes/tag/free` (target Goodreads page tagged with "free")  
      - Sets `"zone"` to `"web_unlocker1"` (Bright Data proxy zone identifier)  
    - Inputs: From manual trigger  
    - Outputs: To "Perform Bright Data Web Request"  
    - Edge cases: Hardcoded URL limits flexibility; changing tags requires manual update. Misconfigured zone could cause proxy failures.

#### 2.2 Web Scraping via Bright Data

- **Overview:**  
  This block sends a POST request to Bright Data’s API to fetch the webpage content from Goodreads through a proxy, avoiding IP blocks or captchas.

- **Nodes Involved:**  
  - Perform Bright Data Web Request

- **Node Details:**

  - **Perform Bright Data Web Request**  
    - Type: HTTP Request  
    - Role: Executes the proxied web scraping via Bright Data API.  
    - Configuration:  
      - POST to `https://api.brightdata.com/request`  
      - Body parameters include `zone` (proxy zone), `url` (Goodreads page), `format` set to `"raw"`, and `data_format` set to `"markdown"` for the response.  
      - Authentication via HTTP Header Auth credential.  
    - Inputs: From "Set the fields" with dynamic expressions for `zone` and `url`.  
    - Outputs: To "Quotes Extractor"  
    - Edge cases: Possible proxy auth failures, network timeouts, or rate limiting by Bright Data. Incorrect zone or URL may cause invalid responses.

#### 2.3 AI Processing with Google Gemini

- **Overview:**  
  This block prepares the scraped data for quote extraction by passing it through Google Gemini’s chat model for language understanding and formatting.

- **Nodes Involved:**  
  - Google Gemini Chat Model

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: AI Chat Model (Google Gemini via LangChain)  
    - Role: AI natural language processing to interpret the raw scraped data.  
    - Configuration:  
      - Model selected: `models/gemini-2.0-flash-exp`  
      - No additional options configured.  
      - Credential: Google PaLM API (Google Gemini)  
    - Inputs: From "Perform Bright Data Web Request" (though note workflow connections show it linked after "Quotes Extractor" - see connections section for details)  
    - Outputs: To "Quotes Extractor" (via AI language model connection)  
    - Edge cases: API quota limits, auth failures, or malformed inputs can cause errors. Model unavailability or version mismatch may affect results.

#### 2.4 Quote Extraction

- **Overview:**  
  This block uses an AI-powered information extractor to parse the AI-processed text and extract clean, structured quotes.

- **Nodes Involved:**  
  - Quotes Extractor

- **Node Details:**

  - **Quotes Extractor**  
    - Type: Information Extractor (LangChain node)  
    - Role: Extracts structured quote data from the processed text.  
    - Configuration:  
      - Input text expression: `"=Extract the quotes from  {{ $json.data }}"` — applies an instruction to the AI extractor to find quotes in the given data field.  
      - Schema defined manually: expects an array of objects with a `quote` string property.  
    - Inputs:  
      - From "Perform Bright Data Web Request" (main connection)  
      - From "Google Gemini Chat Model" (ai_languageModel connection)  
    - Outputs: None connected in this workflow (presumably final output).  
    - Edge cases: Incorrect schema or unexpected data format can cause extraction failures. If AI output is noisy or incomplete, extraction accuracy may degrade.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                          | Input Node(s)           | Output Node(s)                | Sticky Note                          |
|----------------------------|--------------------------------------|----------------------------------------|------------------------|------------------------------|------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                      | Manual start of workflow                | None                   | Set the fields               |                                    |
| Set the fields             | Set                                  | Initialize URL and proxy zone parameters | When clicking ‘Test workflow’ | Perform Bright Data Web Request |                                    |
| Perform Bright Data Web Request | HTTP Request                       | Scrape Goodreads page via Bright Data proxy | Set the fields          | Quotes Extractor             |                                    |
| Google Gemini Chat Model   | AI Chat Model (Google Gemini)          | Process scraped data with AI chat model | (No direct main input; connected via ai_languageModel) | Quotes Extractor (ai_languageModel) |                                    |
| Quotes Extractor           | Information Extractor (AI)             | Extract structured quotes from AI data | Perform Bright Data Web Request (main), Google Gemini Chat Model (ai_languageModel) | None                         |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`. No parameters needed.

2. **Create Set Node (`Set the fields`)**  
   - Add a **Set** node named `Set the fields`.  
   - Configure two fields:  
     - `url` (string): `https://www.goodreads.com/quotes/tag/free`  
     - `zone` (string): `web_unlocker1`  
   - Connect output of `When clicking ‘Test workflow’` to this node.

3. **Create HTTP Request Node (`Perform Bright Data Web Request`)**  
   - Add an **HTTP Request** node named `Perform Bright Data Web Request`.  
   - Set method to `POST`.  
   - Set URL to `https://api.brightdata.com/request`.  
   - Under "Body Parameters", add:  
     - `zone`: Expression `{{$json["zone"]}}`  
     - `url`: Expression `{{$json["url"]}}`  
     - `format`: `raw`  
     - `data_format`: `markdown`  
   - Enable “Send Body” and “Send Headers”.  
   - Authentication: Use HTTP Header Auth credentials for Bright Data proxy (`Header Auth account`).  
   - Connect output of `Set the fields` node to this node.

4. **Create Google Gemini Chat Model Node (`Google Gemini Chat Model`)**  
   - Add an **AI Chat Model** node of type `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`.  
   - Name it `Google Gemini Chat Model`.  
   - Set model name to `models/gemini-2.0-flash-exp`.  
   - Configure Google PaLM API credentials (`Google Gemini(PaLM) Api account`).  
   - No additional options needed.  
   - Connect as an AI language model node to the next node (Quotes Extractor).

5. **Create Information Extractor Node (`Quotes Extractor`)**  
   - Add an **Information Extractor** node named `Quotes Extractor`.  
   - Set input text parameter to expression: `=Extract the quotes from  {{ $json.data }}`  
   - Define manual schema as:  
     ```json
     {
       "type": "array",
       "properties": {
         "quote": {
           "type": "string"
         }
       }
     }
     ```  
   - Connect main input from `Perform Bright Data Web Request`.  
   - Connect AI language model input from `Google Gemini Chat Model`.

6. **Set Node Connections**  
   - Connect `When clicking ‘Test workflow’` → `Set the fields`.  
   - Connect `Set the fields` → `Perform Bright Data Web Request`.  
   - Connect `Perform Bright Data Web Request` → `Quotes Extractor` (main input).  
   - Connect `Google Gemini Chat Model` → `Quotes Extractor` (ai_languageModel input).

7. **Save and Activate**  
   - Ensure all credentials are valid and properly configured for Bright Data and Google Gemini API.  
   - Test workflow manually by triggering `When clicking ‘Test workflow’`.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                              |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| The workflow uses Bright Data proxy to avoid IP blocking and captchas while scraping Goodreads. | Bright Data API documentation: https://brightdata.com/api    |
| Google Gemini model `models/gemini-2.0-flash-exp` requires valid Google PaLM API credentials.  | Google PaLM API docs: https://developers.google.com/palm     |
| The extraction relies on AI language understanding; output quality depends on model responses. | Consider API rate limits and error handling in production.   |
| The schema for quotes extraction expects an array of objects with a `quote` string property.   | Adjust schema if output format changes.                       |

---

**Disclaimer:** The provided content is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.