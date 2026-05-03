Extract Amazon Best Seller Electronic Info with Bright Data and Google Gemini

https://n8nworkflows.xyz/workflows/extract-amazon-best-seller-electronic-info-with-bright-data-and-google-gemini-3681


# Extract Amazon Best Seller Electronic Info with Bright Data and Google Gemini

### 1. Workflow Overview

This workflow automates the extraction of Amazon Best Seller product information specifically from the Electronics category. It leverages Bright Data’s Web Unlocker API to scrape real-time product data from Amazon, then uses Google Gemini’s large language model (LLM) to parse and transform the raw HTML/text data into structured JSON. Finally, it forwards the structured data to a configurable webhook endpoint for further processing or integration.

**Target Use Cases:**

- E-commerce analysts tracking Amazon best-seller trends.
- Product intelligence teams gathering competitor insights.
- AI chatbot developers requiring fresh, structured product data.
- Growth hackers and marketers automating competitive research.
- Data aggregators and price trackers needing reliable Amazon data extraction.

**Logical Blocks:**

- **1.1 Input Reception and Initialization:** Manual trigger and setting the Amazon URL with Bright Data zone parameters.
- **1.2 Data Extraction:** HTTP request to Bright Data’s Web Unlocker API to fetch raw Amazon best seller page content.
- **1.3 AI Processing:** Using Google Gemini LLM to extract structured product information from raw data.
- **1.4 Output Dispatch:** Sending the structured JSON data to a webhook endpoint.
- **1.5 Documentation and Notes:** Sticky notes providing context and instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
  This block initiates the workflow manually and sets the target Amazon URL and Bright Data zone parameters for scraping.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Amazon URL with the Bright Data Zone (Set Node)  
  - Sticky Note (Instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually for testing or on-demand runs.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connected to “Set Amazon URL with the Bright Data Zone”  
    - Edge Cases: None typical; manual trigger requires user interaction.

  - **Set Amazon URL with the Bright Data Zone**  
    - Type: Set  
    - Role: Defines the Amazon best seller Electronics URL and Bright Data zone name for the scraping request.  
    - Configuration:  
      - `url`: "https://www.amazon.in/gp/bestsellers/electronics/1389432031?product=unlocker&method=api"  
      - `zone`: "web_unlocker1"  
    - Inputs: From manual trigger  
    - Outputs: To HTTP Request node for data fetching  
    - Notes: This URL and zone must be updated by the user to match their Bright Data setup and desired Amazon category.  
    - Edge Cases: Incorrect URL or zone will cause scraping failure or invalid data.

  - **Sticky Note (near input block)**  
    - Content: Reminder to update the Amazon URL and webhook notification URL before running.  
    - Role: Documentation and user guidance.

---

#### 1.2 Data Extraction

- **Overview:**  
  This block sends a POST request to Bright Data’s Web Unlocker API to scrape the Amazon best seller page content using the configured URL and zone.

- **Nodes Involved:**  
  - HTTP Request to fetch the Amazon Best Seller Products

- **Node Details:**

  - **HTTP Request to fetch the Amazon Best Seller Products**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Bright Data API to retrieve raw Amazon page data.  
    - Configuration:  
      - URL: "https://api.brightdata.com/request"  
      - Method: POST  
      - Authentication: Generic HTTP Header Authentication (configured with Bright Data API key)  
      - Body Parameters:  
        - `zone`: from previous node (`{{$json.zone}}`)  
        - `url`: from previous node (`{{$json.url}}`)  
        - `format`: "raw" (requesting raw HTML/text)  
      - Headers: Set via credential (Bright Data header auth)  
    - Inputs: From “Set Amazon URL with the Bright Data Zone”  
    - Outputs: Raw data passed to the AI processing node  
    - Edge Cases:  
      - Authentication failure (invalid API key)  
      - Network timeout or API rate limits  
      - Invalid zone or URL causing empty or error responses  
      - Unexpected HTML structure changes on Amazon side

---

#### 1.3 AI Processing

- **Overview:**  
  This block uses Google Gemini’s Flash Exp model to parse the raw Amazon page content and extract structured product information according to a defined JSON schema.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Structured Data Extractor

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat LLM Node  
    - Role: Provides the language model interface to Google Gemini’s Flash Exp model for natural language processing.  
    - Configuration:  
      - Model: "models/gemini-2.0-flash-exp"  
      - Credentials: Google Gemini (PaLM) API key configured in n8n credentials  
    - Inputs: None directly; connected as AI model provider to the extractor node  
    - Outputs: AI model responses to the extractor node  
    - Edge Cases:  
      - API key invalid or expired  
      - Model unavailability or quota exceeded  
      - Latency or timeout issues

  - **Structured Data Extractor**  
    - Type: LangChain Information Extractor Node  
    - Role: Uses the Google Gemini LLM to extract structured JSON data from raw text input.  
    - Configuration:  
      - Input Text: `{{$json.data}}` (raw HTML/text from Bright Data)  
      - System Prompt: Instructs the model to extract only relevant information and omit unknown attributes.  
      - Schema: Manual JSON schema defining expected output structure for Amazon best sellers in Electronics, including category, description, page info, and an array of bestsellers with rank, title, image URL, rating (stars and total ratings), offer, and product URL.  
    - Inputs: Raw data from HTTP Request node  
    - Outputs: Structured JSON passed to webhook notifier  
    - Edge Cases:  
      - Model misinterpretation or incomplete extraction if HTML structure changes  
      - Missing attributes if data is not present or unclear  
      - Schema validation errors if output does not conform

  - **Sticky Note1 (near AI nodes)**  
    - Content: Notes that Google Gemini Flash Exp model is used for information extraction to build structured data.  
    - Role: Documentation of AI usage.

---

#### 1.4 Output Dispatch

- **Overview:**  
  This block sends the fully structured JSON data extracted by the AI to a webhook endpoint for downstream consumption.

- **Nodes Involved:**  
  - Webhook Notifier for structured data extractor

- **Node Details:**

  - **Webhook Notifier for structured data extractor**  
    - Type: HTTP Request  
    - Role: Sends a POST request with the structured JSON data to a specified webhook URL.  
    - Configuration:  
      - URL: "https://webhook.site/bc804ce5-4a45-4177-a68a-99c80e5c86e6" (placeholder, user must update)  
      - Method: POST (default)  
      - Body Parameters:  
        - `summary`: `{{$json.output}}` (structured JSON from extractor)  
    - Inputs: From Structured Data Extractor node  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - Invalid webhook URL or network issues causing delivery failure  
      - Payload size limits or formatting issues at webhook receiver

---

#### 1.5 Documentation and Notes

- **Overview:**  
  Sticky notes provide contextual information, setup instructions, and reminders to update key parameters.

- **Nodes Involved:**  
  - Sticky Note (near input block)  
  - Sticky Note1 (near AI block)

- **Node Details:**  
  - These nodes contain no execution logic but are critical for user understanding and proper workflow configuration.

---

### 3. Summary Table

| Node Name                                   | Node Type                                   | Functional Role                          | Input Node(s)                          | Output Node(s)                               | Sticky Note                                                                                  |
|---------------------------------------------|---------------------------------------------|----------------------------------------|---------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                | Manual Trigger                              | Workflow entry point                    | None                                  | Set Amazon URL with the Bright Data Zone     |                                                                                              |
| Set Amazon URL with the Bright Data Zone    | Set                                         | Sets Amazon URL and Bright Data zone   | When clicking ‘Test workflow’          | HTTP Request to fetch the Amazon Best Seller Products | "Set the URL which you are interested to scrap the data" (node note)                        |
| HTTP Request to fetch the Amazon Best Seller Products | HTTP Request                               | Fetches raw Amazon page data via Bright Data | Set Amazon URL with the Bright Data Zone | Structured Data Extractor                      |                                                                                              |
| Google Gemini Chat Model                      | LangChain Google Gemini Chat LLM            | Provides AI model for data extraction  | None (used by Structured Data Extractor) | Structured Data Extractor                      | "Google Gemini Flash Exp model is being used. Information Extraction for building structured data" |
| Structured Data Extractor                     | LangChain Information Extractor             | Extracts structured JSON from raw data | HTTP Request to fetch the Amazon Best Seller Products | Webhook Notifier for structured data extractor |                                                                                              |
| Webhook Notifier for structured data extractor | HTTP Request                               | Sends structured JSON to webhook       | Structured Data Extractor              | None                                         |                                                                                              |
| Sticky Note                                  | Sticky Note                                 | User guidance and reminders             | None                                  | None                                         | "Deals with the Amazon Best Seller Electronic data extraction using the Bright Data and LLM for Information Extraction. Please update the 'Set Amazon URL with the Bright Data Zone' and the Webhook Notification URL" |
| Sticky Note1                                 | Sticky Note                                 | Notes on AI model usage                  | None                                  | None                                         | "Google Gemini Flash Exp model is being used. Information Extraction for building the structured data" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node to Define Amazon URL and Bright Data Zone**  
   - Name: `Set Amazon URL with the Bright Data Zone`  
   - Type: Set  
   - Add two fields:  
     - `url` (string): Set to `"https://www.amazon.in/gp/bestsellers/electronics/1389432031?product=unlocker&method=api"` (update as needed)  
     - `zone` (string): Set to `"web_unlocker1"` (update to your Bright Data zone name)  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node to Fetch Amazon Data**  
   - Name: `HTTP Request to fetch the Amazon Best Seller Products`  
   - Type: HTTP Request  
   - Set method to POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: Generic HTTP Header Authentication  
     - Configure credentials with Bright Data API key (Header Auth type)  
   - Body Parameters (send as form or JSON as per API):  
     - `zone`: Expression `{{$json.zone}}` (from previous node)  
     - `url`: Expression `{{$json.url}}` (from previous node)  
     - `format`: `"raw"`  
   - Connect output of Set node to this HTTP Request node.

4. **Create Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Type: LangChain Google Gemini Chat LLM  
   - Model Name: `models/gemini-2.0-flash-exp`  
   - Credentials: Configure Google Gemini (PaLM) API key in n8n credentials and select here.  
   - No direct input connections; this node will be referenced by the extractor node.

5. **Create Structured Data Extractor Node**  
   - Name: `Structured Data Extractor`  
   - Type: LangChain Information Extractor  
   - Parameters:  
     - Text input: Expression `{{$json.data}}` (raw data from HTTP Request node)  
     - System Prompt:  
       ```
       You are an expert extraction algorithm.
       Only extract relevant information from the text.
       If you do not know the value of an attribute asked to extract, you may omit the attribute's value.
       ```  
     - Schema Type: Manual  
     - Input Schema: Paste the JSON schema defining the expected Amazon best seller Electronics data structure (category, description, page, bestsellers array with rank, title, image, rating, offer, product_url).  
   - Connect the HTTP Request node output to this node’s input.  
   - In the node’s AI Language Model setting, select the previously created `Google Gemini Chat Model`.

6. **Create HTTP Request Node to Send Structured Data to Webhook**  
   - Name: `Webhook Notifier for structured data extractor`  
   - Type: HTTP Request  
   - URL: Set to your webhook endpoint (e.g., `https://webhook.site/...`)  
   - Method: POST  
   - Body Parameters:  
     - `summary`: Expression `{{$json.output}}` (structured JSON from extractor)  
   - Connect output of Structured Data Extractor node to this node.

7. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add notes near the input block reminding users to update the Amazon URL and webhook URL.  
   - Add notes near AI nodes explaining the use of Google Gemini Flash Exp model for extraction.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Sign up at Bright Data and create a Web Unlocker zone under Scraping Solutions for API access.                 | https://brightdata.com/                                                                         |
| Configure Bright Data API credentials in n8n as Generic HTTP Header Authentication with your API key.         | n8n Credential Setup                                                                            |
| Configure Google Gemini (PaLM) API credentials in n8n for access to the Gemini Flash Exp model.                | Google Cloud Vertex AI or Google PaLM API documentation                                        |
| Update the Amazon URL in the “Set Amazon URL with the Bright Data Zone” node to target different Amazon categories or locales. | Customization tip                                                                              |
| Update the webhook URL in the “Webhook Notifier for structured data extractor” node to forward data to your desired endpoint (Google Sheets, Airtable, Slack, etc.). | Integration customization                                                                      |
| The workflow uses Google Gemini Flash Exp model for advanced information extraction from unstructured text.   | Sticky Note in workflow                                                                        |
| This workflow is designed for manual triggering but can be scheduled or triggered via webhook for automation.  | n8n scheduling and trigger options                                                            |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.