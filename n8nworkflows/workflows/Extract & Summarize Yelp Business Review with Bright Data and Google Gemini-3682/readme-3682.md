Extract & Summarize Yelp Business Review with Bright Data and Google Gemini

https://n8nworkflows.xyz/workflows/extract---summarize-yelp-business-review-with-bright-data-and-google-gemini-3682


# Extract & Summarize Yelp Business Review with Bright Data and Google Gemini

### 1. Workflow Overview

This workflow automates the extraction, structuring, summarization, and forwarding of Yelp business reviews. It is designed to scrape Yelp review data using Bright Data’s Web Unlocker, transform raw review data into structured JSON, generate concise summaries via Google Gemini’s large language model (LLM), and send the combined structured data and summary to a specified webhook endpoint.

**Target Use Cases:**

- Local SEO specialists analyzing Yelp reviews for optimization.
- Business owners seeking quick insights from customer feedback.
- Reputation managers monitoring brand sentiment.
- Data analysts extracting Yelp review patterns.
- AI developers requiring clean Yelp review data for models.

**Logical Blocks:**

- **1.1 Input Reception & URL Setup:** Manual trigger and URL preparation for scraping.
- **1.2 Data Extraction via Bright Data:** HTTP request to Bright Data’s Web Unlocker API to scrape Yelp data.
- **1.3 Structured Data Extraction:** Using Google Gemini LLM to parse raw Yelp data into structured JSON.
- **1.4 Summarization:** Summarizing the structured Yelp reviews with Google Gemini.
- **1.5 Output Merging & Delivery:** Combining structured data and summary, then sending to a webhook endpoint.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & URL Setup

**Overview:**  
Starts the workflow manually and sets the Yelp URL with Bright Data zone parameters for scraping.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set Yelp URL with the Bright Data Zone

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters; triggers workflow execution on demand.  
  - Inputs: None  
  - Outputs: Connects to “Set Yelp URL with the Bright Data Zone”  
  - Edge cases: None typical; manual trigger ensures controlled execution.

- **Set Yelp URL with the Bright Data Zone**  
  - Type: Set  
  - Role: Assigns the target Yelp URL and Bright Data zone name for scraping.  
  - Configuration:  
    - `url` set to a Yelp search page filtered by category and location (default: Restaurants in San Francisco, CA, sorted by rating).  
    - `zone` set to “web_unlocker1” (Bright Data zone identifier).  
  - Inputs: From manual trigger  
  - Outputs: Connects to HTTP Request node for scraping  
  - Edge cases: URL must be updated to target desired Yelp business or category; incorrect URL or zone may cause scraping failure.

---

#### 1.2 Data Extraction via Bright Data

**Overview:**  
Sends a POST request to Bright Data’s Web Unlocker API to scrape Yelp business review data.

**Nodes Involved:**  
- HTTP Request to fetch the Yelp Business Reviews

**Node Details:**

- **HTTP Request to fetch the Yelp Business Reviews**  
  - Type: HTTP Request  
  - Role: Calls Bright Data API to scrape raw Yelp review HTML or data.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.brightdata.com/request`  
    - Body Parameters: `zone`, `url` (from previous node), `format` set to `raw` for raw response.  
    - Authentication: Generic HTTP Header Authentication using configured credentials.  
  - Inputs: From “Set Yelp URL with the Bright Data Zone”  
  - Outputs: Raw scraped data JSON to next node  
  - Edge cases:  
    - Authentication failure if credentials invalid.  
    - Rate limiting or quota exceeded on Bright Data side.  
    - Network timeouts or API errors.  
    - Invalid zone or URL causing no data or errors.

---

#### 1.3 Structured Data Extraction

**Overview:**  
Processes raw Yelp data using Google Gemini LLM to extract and format it into structured JSON with fields like restaurant name, location, ratings, and individual reviews.

**Nodes Involved:**  
- Structured Data Extractor  
- Google Gemini Chat Model  
- Structured Output Parser

**Node Details:**

- **Structured Data Extractor**  
  - Type: LangChain LLM Chain  
  - Role: Defines the prompt to summarize and analyze Yelp reviews and requests structured output.  
  - Configuration:  
    - Prompt includes dynamic input: `Summarize and analyze Yelp reviews {{ $json.data }}`  
    - Output parser enabled to enforce JSON schema.  
  - Inputs: Raw data from HTTP Request node  
  - Outputs: Sends prompt to Google Gemini Chat Model and receives structured response parsed by Structured Output Parser  
  - Edge cases:  
    - Expression failure if input data missing or malformed.  
    - LLM response may not match schema if prompt unclear or input noisy.

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini LLM node  
  - Role: Executes the LLM prompt for structured data extraction.  
  - Configuration:  
    - Model: `models/gemini-2.0-flash-exp` (flash experimental version)  
    - Credentials: Google Gemini API key configured.  
  - Inputs: Prompt from Structured Data Extractor  
  - Outputs: LLM response to Structured Output Parser  
  - Edge cases:  
    - API key invalid or expired.  
    - Rate limits or quota exceeded.  
    - Model errors or timeouts.

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses LLM output into predefined JSON schema for Yelp reviews.  
  - Configuration: JSON schema example includes fields like `restaurant_name`, `location`, `average_rating`, `review_count`, and nested `reviews` with reviewer info.  
  - Inputs: LLM response from Google Gemini Chat Model  
  - Outputs: Structured JSON data to next node  
  - Edge cases:  
    - Parsing failure if LLM output deviates from schema.  
    - Malformed JSON causing errors.

---

#### 1.4 Summarization

**Overview:**  
Generates a concise summary of the structured Yelp reviews using Google Gemini LLM with a summarization chain.

**Nodes Involved:**  
- Summarization Chain  
- Google Gemini Chat Model for Summarization

**Node Details:**

- **Summarization Chain**  
  - Type: LangChain Summarization Chain  
  - Role: Uses a prompt to create a concise summary of the review text.  
  - Configuration:  
    - Prompts:  
      - Map prompt: `"Write a concise summary of the following:\n\n\"{text}\"\n\n"`  
      - Combine prompt: `"Write a concise summary of the following:\n\n\n\n\n\nCONCISE SUMMARY: {{ $json.output }}"`  
  - Inputs: Structured data from Structured Data Extractor  
  - Outputs: Summary text and structured data merged downstream  
  - Edge cases:  
    - Empty or incomplete input causing poor summaries.  
    - LLM errors or timeouts.

- **Google Gemini Chat Model for Summarization**  
  - Type: LangChain Google Gemini LLM node  
  - Role: Runs the summarization prompt on Google Gemini.  
  - Configuration:  
    - Model: `models/gemini-2.0-flash-exp`  
    - Credentials: Google Gemini API key  
  - Inputs: Summarization Chain prompt  
  - Outputs: Summary text to Summarization Chain  
  - Edge cases: Same as previous Google Gemini node.

---

#### 1.5 Output Merging & Delivery

**Overview:**  
Merges the structured Yelp review data and the generated summary, then sends the combined result to a configured webhook endpoint.

**Nodes Involved:**  
- Merge  
- Webhook Notifier for the merged response

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from Structured Data Extractor (structured data) and Summarization Chain (summary) into one data stream.  
  - Configuration: Default merge mode (likely “Wait” or “Merge by index”).  
  - Inputs:  
    - Main input: Structured data from Structured Data Extractor  
    - Secondary input: Summary from Summarization Chain  
  - Outputs: Combined JSON to webhook notifier  
  - Edge cases:  
    - Synchronization issues if one input delayed or missing.

- **Webhook Notifier for the merged response**  
  - Type: HTTP Request  
  - Role: Sends the final combined JSON with reviews and summary to an external webhook endpoint.  
  - Configuration:  
    - Method: POST (default)  
    - URL: Configured webhook URL (default example: https://webhook.site/daf9d591-a130-4010-b1d3-0c66f8fcf467)  
    - Body Parameters:  
      - `reviews`: structured JSON output  
      - `summary`: summarized text response  
  - Inputs: Merged data from Merge node  
  - Outputs: None (terminal node)  
  - Edge cases:  
    - Webhook URL incorrect or unreachable.  
    - Network errors or timeouts.  
    - Payload size limits on webhook endpoint.

---

### 3. Summary Table

| Node Name                             | Node Type                               | Functional Role                              | Input Node(s)                          | Output Node(s)                        | Sticky Note                                                                                                  |
|-------------------------------------|---------------------------------------|----------------------------------------------|--------------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’        | Manual Trigger                        | Workflow entry point                          | None                                 | Set Yelp URL with the Bright Data Zone |                                                                                                              |
| Set Yelp URL with the Bright Data Zone | Set                                  | Sets Yelp URL and Bright Data zone for scraping | When clicking ‘Test workflow’         | HTTP Request to fetch the Yelp Business Reviews | Set the URL which you are interested to scrap the data                                                        |
| HTTP Request to fetch the Yelp Business Reviews | HTTP Request                        | Calls Bright Data API to scrape Yelp data    | Set Yelp URL with the Bright Data Zone | Structured Data Extractor            |                                                                                                              |
| Structured Data Extractor            | LangChain LLM Chain                   | Extracts and structures Yelp review data     | HTTP Request to fetch the Yelp Business Reviews | Summarization Chain, Merge          |                                                                                                              |
| Google Gemini Chat Model             | LangChain Google Gemini LLM           | Runs LLM prompt for structured data extraction | Structured Data Extractor (ai_languageModel) | Structured Output Parser            | ## LLM Usages Google Gemini Flash Exp model is being used. Basic LLM Chain with the Output parser for building the structured data. Summarization Chain to summarize the structured response. |
| Structured Output Parser             | LangChain Output Parser Structured    | Parses LLM output into JSON schema            | Google Gemini Chat Model             | Structured Data Extractor           |                                                                                                              |
| Summarization Chain                 | LangChain Summarization Chain         | Summarizes the structured Yelp reviews       | Structured Data Extractor            | Merge                              |                                                                                                              |
| Google Gemini Chat Model for Summarization | LangChain Google Gemini LLM           | Runs LLM prompt for summarization             | Summarization Chain (ai_languageModel) | Summarization Chain                |                                                                                                              |
| Merge                              | Merge                                | Combines structured data and summary          | Structured Data Extractor, Summarization Chain | Webhook Notifier for the merged response |                                                                                                              |
| Webhook Notifier for the merged response | HTTP Request                        | Sends combined data and summary to webhook   | Merge                               | None                              | **Please make sure to update the "Set Yelp URL with the Bright Data Zone" and the Webhook Notification URL** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for Yelp URL and Bright Data Zone**  
   - Name: `Set Yelp URL with the Bright Data Zone`  
   - Type: Set  
   - Add two string fields:  
     - `url`: Set default to `https://www.yelp.com/search?find_desc=Restaurants&find_loc=San+Francisco%2C+CA&sortby=rating?product=unlocker&method=api` (update as needed)  
     - `zone`: Set to `web_unlocker1` (Bright Data zone name)  
   - Connect Manual Trigger output to this node.

3. **Create HTTP Request Node to Bright Data API**  
   - Name: `HTTP Request to fetch the Yelp Business Reviews`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: Generic HTTP Header Authentication (configure credentials with Bright Data API key)  
   - Body Parameters (send as form or JSON):  
     - `zone`: `={{ $json.zone }}`  
     - `url`: `={{ $json.url }}`  
     - `format`: `raw`  
   - Connect Set node output to this node.

4. **Create LangChain LLM Chain Node for Structured Data Extraction**  
   - Name: `Structured Data Extractor`  
   - Type: LangChain Chain LLM  
   - Prompt: `Summarize and analyze Yelp reviews {{ $json.data }}` (dynamic input)  
   - Enable Output Parser.  
   - Connect HTTP Request node output to this node.

5. **Create Google Gemini Chat Model Node for Structured Data Extraction**  
   - Name: `Google Gemini Chat Model`  
   - Type: LangChain Google Gemini LLM  
   - Model Name: `models/gemini-2.0-flash-exp`  
   - Credentials: Configure Google Gemini API key credentials.  
   - Connect as `ai_languageModel` input from Structured Data Extractor.

6. **Create Structured Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Type: LangChain Output Parser Structured  
   - JSON Schema Example:  
     ```json
     [
       {
         "restaurant_name": "string",
         "location": "string",
         "average_rating": "float",
         "review_count": "int",
         "reviews": [
           {
             "reviewer": "string",
             "rating": "float",
             "date": "YYYY-MM-DD",
             "text": "string"
           }
         ]
       }
     ]
     ```  
   - Connect as `ai_outputParser` input from Google Gemini Chat Model.

7. **Create LangChain Summarization Chain Node**  
   - Name: `Summarization Chain`  
   - Type: LangChain Summarization Chain  
   - Configure prompts:  
     - Map prompt:  
       ```
       Write a concise summary of the following:

       "{text}"
       ```  
     - Combine prompt:  
       ```
       Write a concise summary of the following:

       CONCISE SUMMARY: {{ $json.output }}
       ```  
   - Connect main output from Structured Data Extractor to this node.

8. **Create Google Gemini Chat Model Node for Summarization**  
   - Name: `Google Gemini Chat Model for Summarization`  
   - Type: LangChain Google Gemini LLM  
   - Model Name: `models/gemini-2.0-flash-exp`  
   - Credentials: Use same Google Gemini API key.  
   - Connect as `ai_languageModel` input from Summarization Chain.

9. **Create Merge Node**  
   - Name: `Merge`  
   - Type: Merge  
   - Connect main output from Structured Data Extractor to input 1.  
   - Connect main output from Summarization Chain to input 2.

10. **Create HTTP Request Node for Webhook Notification**  
    - Name: `Webhook Notifier for the merged response`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Set to your webhook endpoint (default example: `https://webhook.site/daf9d591-a130-4010-b1d3-0c66f8fcf467`)  
    - Body Parameters:  
      - `reviews`: `={{ $json.output }}` (structured data)  
      - `summary`: `={{ $json.response.text }}` (summary text)  
    - Connect Merge node output to this node.

11. **Verify Connections:**  
    - Manual Trigger → Set Yelp URL  
    - Set Yelp URL → HTTP Request to Bright Data  
    - HTTP Request → Structured Data Extractor  
    - Structured Data Extractor → Google Gemini Chat Model (ai_languageModel)  
    - Google Gemini Chat Model → Structured Output Parser (ai_outputParser)  
    - Structured Data Extractor → Summarization Chain  
    - Summarization Chain → Google Gemini Chat Model for Summarization (ai_languageModel)  
    - Structured Data Extractor and Summarization Chain → Merge  
    - Merge → Webhook Notifier

12. **Credential Setup:**  
    - Configure Bright Data API credentials as HTTP Header Auth with your API key.  
    - Configure Google Gemini API credentials with your Google Gemini (PaLM) API key.

13. **Update URLs:**  
    - Modify the Yelp URL in the Set node to target desired business or category.  
    - Update the webhook URL in the final HTTP Request node to your desired endpoint.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses Google Gemini Flash Experimental model for LLM tasks including structured data extraction and summarization. | LLM model reference: `models/gemini-2.0-flash-exp`                                             |
| Ensure to update the Yelp URL in the “Set Yelp URL with the Bright Data Zone” node and the webhook URL in the final notifier node before running. | Critical setup instruction for correct data scraping and output delivery.                      |
| Bright Data Web Unlocker setup is required: create a Web Unlocker zone and configure credentials in n8n accordingly. | https://brightdata.com/                                                                         |
| Google Gemini API key or access via Vertex AI or proxy must be configured in n8n credentials for LLM nodes.     | Google Cloud Platform or Vertex AI documentation for API key setup.                             |
| This workflow is designed for flexibility: you can customize Yelp URLs, limit reviews, adjust output schema, and redirect output to other destinations like Google Sheets, Airtable, Slack, or custom APIs. | Customization guidance as per user needs.                                                      |
| Sticky Note1 content: "## LLM Usages\n\nGoogle Gemini Flash Exp model is being used.\n\nBasic LLM Chain with the Output parser for building the structured data.\n\nSummarization Chain to summarize the structured response." | Explains LLM usage and chaining strategy in the workflow.                                      |
| Sticky Note content: "## Note\n\nDeals with the Yelp Business Review data extraction using the Bright Data and LLM for structured data extraction and summarization.\n\n**Please make sure to update the \"Set Yelp URL with the Bright Data Zone\" and the Webhook Notification URL**" | Important reminder for workflow configuration.                                                 |

---

This documentation provides a comprehensive, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.