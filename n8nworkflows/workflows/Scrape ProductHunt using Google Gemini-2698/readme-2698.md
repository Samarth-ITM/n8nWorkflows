Scrape ProductHunt using Google Gemini

https://n8nworkflows.xyz/workflows/scrape-producthunt-using-google-gemini-2698


# Scrape ProductHunt using Google Gemini

### 1. Workflow Overview

This workflow, titled **"Product Data Extractor"**, automates the extraction of detailed product information from Product Hunt. It is designed to receive a product name via a webhook, fetch the corresponding Product Hunt page HTML, extract embedded inline scripts, analyze these scripts using AI language models (including Google Gemini), and finally format and return a structured JSON response containing comprehensive product data.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming HTTP requests with the product name.
- **1.2 Data Retrieval:** Fetches the Product Hunt page HTML dynamically based on the product name.
- **1.3 Data Extraction:** Parses the HTML to extract inline script content relevant to product data.
- **1.4 AI Processing:** Uses language models to analyze and interpret the extracted script data.
- **1.5 Data Formatting:** Structures the AI-processed data into a predefined JSON schema.
- **1.6 Response Delivery:** Sends the final JSON back to the client via the webhook.

This modular design ensures adaptability, robustness against DOM changes on Product Hunt, and high accuracy in data extraction.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP requests containing the product name to be processed. It acts as the entry point for the workflow.

- **Nodes Involved:**  
  - Receive Product Request

- **Node Details:**

  - **Receive Product Request**  
    - **Type:** Webhook  
    - **Role:** Captures HTTP GET requests with a query parameter `product` specifying the product name (e.g., `...?product=epigram`).  
    - **Configuration:**  
      - Uses default webhook settings with no additional authentication or body parsing.  
      - Extracts the `product` parameter from the query string for downstream use.  
    - **Inputs:** None (entry node)  
    - **Outputs:** Passes the request data including the product name to the next node.  
    - **Version:** 2  
    - **Potential Failures:**  
      - Missing or empty `product` parameter could cause downstream nodes to fail or fetch invalid URLs.  
      - Unauthorized or malformed HTTP requests (though no auth is configured).  
    - **Notes:** The webhook path should be customized to fit the deployment environment.

#### 2.2 Data Retrieval

- **Overview:**  
  Constructs the Product Hunt URL dynamically using the product name and fetches the HTML content of the product page.

- **Nodes Involved:**  
  - Fetch Product HTML

- **Node Details:**

  - **Fetch Product HTML**  
    - **Type:** HTTP Request  
    - **Role:** Sends an HTTP GET request to the Product Hunt product page URL constructed from the product name.  
    - **Configuration:**  
      - URL is dynamically built using the expression referencing the `product` query parameter from the webhook node.  
      - No authentication or special headers configured.  
      - Expects HTML response.  
    - **Inputs:** Receives product name from the webhook node.  
    - **Outputs:** Outputs the raw HTML content of the product page.  
    - **Version:** 4.2  
    - **Potential Failures:**  
      - HTTP errors (404 if product not found, 500 server errors).  
      - Network timeouts or connectivity issues.  
      - Changes in Product Hunt URL structure could break URL construction.  
    - **Notes:** No caching or retries configured; could be added for robustness.

#### 2.3 Data Extraction

- **Overview:**  
  Parses the fetched HTML to extract inline `<script>` tags located within the `<head>` section, excluding any scripts with `src` attributes, to isolate embedded product data.

- **Nodes Involved:**  
  - Extract Inline Scripts

- **Node Details:**

  - **Extract Inline Scripts**  
    - **Type:** Code (JavaScript)  
    - **Role:** Processes the HTML string to parse and extract inline script contents.  
    - **Configuration:**  
      - Uses a DOM parser or regex to locate `<head>` section scripts.  
      - Filters out scripts with `src` attributes to exclude external scripts.  
      - Validates presence of inline scripts and handles cases where none are found.  
    - **Inputs:** Receives HTML content from the HTTP Request node.  
    - **Outputs:** Outputs extracted inline script content as text or JSON for AI processing.  
    - **Version:** 2  
    - **Potential Failures:**  
      - Malformed HTML causing parsing errors.  
      - Absence of inline scripts leading to empty outputs.  
      - Changes in Product Hunt page structure affecting script location.  
    - **Notes:** This approach avoids reliance on fragile DOM selectors.

#### 2.4 AI Processing

- **Overview:**  
  Analyzes the extracted inline scripts using AI language models to interpret and extract structured product data.

- **Nodes Involved:**  
  - Analyze Script with Google Gemini  
  - Process Script with LLM

- **Node Details:**

  - **Analyze Script with Google Gemini**  
    - **Type:** LangChain Google Gemini Chat Model  
    - **Role:** Performs advanced AI analysis on the script content to enrich and refine extracted data.  
    - **Configuration:**  
      - Uses Google Gemini 1.5 8B model for low latency and cost efficiency.  
      - Receives script data as input.  
      - Outputs refined AI response for further processing.  
    - **Inputs:** Receives inline script data from the extraction node or previous AI node.  
    - **Outputs:** Passes enriched data to the next AI processing node.  
    - **Version:** 1  
    - **Potential Failures:**  
      - API authentication or quota errors.  
      - Latency or timeout issues.  
      - Unexpected AI output format.  
    - **Notes:** This node feeds into the next AI processing step.

  - **Process Script with LLM**  
    - **Type:** LangChain Chain LLM  
    - **Role:** Further processes AI output to extract key product attributes and prepare data for final formatting.  
    - **Configuration:**  
      - Uses a chain of prompts and parsers to extract structured data.  
      - May include prompt templates specifying JSON schema and low temperature for accuracy.  
    - **Inputs:** Receives AI-enriched script data from Google Gemini node.  
    - **Outputs:** Outputs structured data ready for JSON formatting or direct response.  
    - **Version:** 1.5  
    - **Potential Failures:**  
      - Expression or prompt errors.  
      - AI model errors or unexpected responses.  
    - **Notes:** This node is the final AI processing step before response.

#### 2.5 Data Formatting

- **Overview:**  
  Converts the AI-processed data into a clean, validated JSON object adhering to a predefined schema representing product details.

- **Nodes Involved:**  
  - Format Product Data to JSON

- **Node Details:**

  - **Format Product Data to JSON**  
    - **Type:** LangChain Output Parser Structured  
    - **Role:** Parses AI output and enforces a JSON schema to ensure consistent and accurate data structure.  
    - **Configuration:**  
      - Defines a strict JSON schema covering fields like `id`, `slug`, `followersCount`, `name`, `tagline`, `reviewsRating`, etc.  
      - Uses low temperature settings to minimize hallucinations.  
    - **Inputs:** Receives AI output from the processing node.  
    - **Outputs:** Outputs validated JSON object representing the product data.  
    - **Version:** 1.2  
    - **Potential Failures:**  
      - Schema validation errors if AI output is malformed.  
      - Parsing errors due to unexpected AI response formats.  
    - **Notes:** Ensures output is reliable for downstream consumption.

#### 2.6 Response Delivery

- **Overview:**  
  Sends the final structured JSON response back to the client through the webhook response.

- **Nodes Involved:**  
  - Send JSON Response to Client

- **Node Details:**

  - **Send JSON Response to Client**  
    - **Type:** Respond to Webhook  
    - **Role:** Returns the final JSON data as the HTTP response to the original webhook request.  
    - **Configuration:**  
      - Configured to send the JSON output from the AI processing node (or formatted JSON node if connected).  
      - Uses default HTTP 200 status.  
    - **Inputs:** Receives structured JSON data.  
    - **Outputs:** Sends HTTP response to client; no further outputs.  
    - **Version:** 1.1  
    - **Potential Failures:**  
      - Network or client connection errors.  
      - Missing or malformed response data causing empty or invalid responses.  
    - **Notes:** Completes the workflow cycle.

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                        | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                         |
|------------------------------|----------------------------------|-------------------------------------|---------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Receive Product Request       | Webhook                          | Entry point; receives product name  | None                      | Fetch Product HTML           | Modify the webhook path to suit your application.                                                 |
| Fetch Product HTML            | HTTP Request                    | Fetches Product Hunt HTML page       | Receive Product Request    | Extract Inline Scripts       | Dynamic URL construction using product name.                                                     |
| Extract Inline Scripts        | Code                            | Extracts inline scripts from HTML    | Fetch Product HTML         | Process Script with LLM      | Excludes scripts with `src`; parses `<head>` section.                                            |
| Analyze Script with Google Gemini | LangChain LM Chat Google Gemini | AI analysis and enrichment of script | Extract Inline Scripts (indirect) | Process Script with LLM      | Uses Google Gemini 1.5 8B model for enhanced analysis.                                           |
| Process Script with LLM       | LangChain Chain LLM             | Processes AI output to structured data | Analyze Script with Google Gemini | Send JSON Response to Client | Low temperature for accuracy; chains prompts and parsers.                                        |
| Format Product Data to JSON   | LangChain Output Parser Structured | Formats AI output into JSON schema   | Process Script with LLM (ai_outputParser) | Process Script with LLM (ai_outputParser) | Ensures output adheres to predefined JSON schema.                                                |
| Send JSON Response to Client  | Respond to Webhook              | Sends final JSON response to client  | Process Script with LLM    | None                        | Returns JSON via webhook response.                                                               |
| Sticky Note                  | Sticky Note                     | Comments and instructions             | None                      | None                        | Modify the webhook path to suit your application.                                                 |
| Sticky Note1                 | Sticky Note                     | Comments and instructions             | None                      | None                        | Dynamic URL construction using product name.                                                     |
| Sticky Note2                 | Sticky Note                     | Comments and instructions             | None                      | None                        | Excludes scripts with `src`; parses `<head>` section.                                            |
| Sticky Note3                 | Sticky Note                     | Comments and instructions             | None                      | None                        | Uses Google Gemini 1.5 8B model for enhanced analysis.                                           |
| Sticky Note4                 | Sticky Note                     | Comments and instructions             | None                      | None                        | Low temperature for accuracy; chains prompts and parsers.                                        |
| Sticky Note5                 | Sticky Note                     | Comments and instructions             | None                      | None                        | Ensures output adheres to predefined JSON schema.                                                |
| Sticky Note6                 | Sticky Note                     | Comments and instructions             | None                      | None                        | Returns JSON via webhook response.                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive Product Request"**  
   - Type: Webhook (Version 2)  
   - Configure to accept HTTP GET requests.  
   - No authentication required.  
   - Extract the query parameter `product` from incoming requests.  
   - Position: Entry point.

2. **Create HTTP Request Node: "Fetch Product HTML"**  
   - Type: HTTP Request (Version 4.2)  
   - Set HTTP Method: GET.  
   - URL: Use expression to build URL dynamically, e.g., `https://www.producthunt.com/posts/{{$json["query"]["product"]}}`  
   - No authentication or special headers.  
   - Connect input from "Receive Product Request".

3. **Create Code Node: "Extract Inline Scripts"**  
   - Type: Code (Version 2)  
   - Write JavaScript code to:  
     - Parse the HTML content from the previous node.  
     - Extract all inline `<script>` tags within the `<head>` section.  
     - Exclude any `<script>` tags with a `src` attribute.  
     - Output the concatenated or array of inline script contents.  
   - Connect input from "Fetch Product HTML".

4. **Create LangChain LM Chat Node: "Analyze Script with Google Gemini"**  
   - Type: LangChain LM Chat Google Gemini (Version 1)  
   - Configure credentials for Google Gemini API.  
   - Input: Pass the extracted inline script content.  
   - Set model to Gemini 1.5 8B for low latency and cost efficiency.  
   - Connect input from "Extract Inline Scripts".

5. **Create LangChain Chain LLM Node: "Process Script with LLM"**  
   - Type: LangChain Chain LLM (Version 1.5)  
   - Configure prompt chain to:  
     - Receive AI output from Google Gemini.  
     - Extract key product data fields as per the JSON schema.  
     - Use low temperature (0) to ensure accuracy.  
   - Connect input from "Analyze Script with Google Gemini".

6. **Create LangChain Output Parser Structured Node: "Format Product Data to JSON"**  
   - Type: LangChain Output Parser Structured (Version 1.2)  
   - Define JSON schema matching expected product data fields (e.g., id, slug, followersCount, name, tagline, etc.).  
   - Connect as an `ai_outputParser` input to "Process Script with LLM" node.

7. **Create Respond to Webhook Node: "Send JSON Response to Client"**  
   - Type: Respond to Webhook (Version 1.1)  
   - Configure to send the JSON output from "Process Script with LLM" node as HTTP response.  
   - Connect input from "Process Script with LLM".

8. **Connect Nodes in Sequence:**  
   - Receive Product Request → Fetch Product HTML → Extract Inline Scripts → Analyze Script with Google Gemini → Process Script with LLM → Send JSON Response to Client  
   - Attach "Format Product Data to JSON" as `ai_outputParser` input to "Process Script with LLM".

9. **Credential Setup:**  
   - Configure Google Gemini API credentials in n8n credentials manager.  
   - No credentials needed for webhook or HTTP request nodes.

10. **Testing:**  
    - Deploy the workflow.  
    - Send HTTP GET request to webhook URL with `?product=productname`.  
    - Verify JSON response matches expected schema and data.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is designed to be resilient to Product Hunt DOM changes by relying on AI analysis rather than selectors. | Workflow Description                                                                               |
| Uses Google Gemini 1.5 8B model for efficient AI processing with low latency and minimal cost.                        | AI Integration Details                                                                            |
| Example JSON output for product "Epigram" included in the description for reference.                                  | Example Output JSON                                                                               |
| Modify webhook path and AI prompts to customize for different use cases or additional data fields.                    | Customization Options                                                                             |
| Typical response time is approximately 6 seconds per product with >95% accuracy due to strict JSON schema enforcement. | Performance Metrics                                                                              |
| Workflow is suitable for developers, marketers, and data analysts automating Product Hunt data extraction.             | Target Audience                                                                                   |
| For more information on LangChain nodes and Google Gemini integration, refer to n8n documentation and Google AI docs. | External Documentation (not included here)                                                       |

---

This completes the comprehensive reference document for the "Scrape ProductHunt using Google Gemini" workflow. It enables understanding, reproduction, and modification by both advanced users and AI agents.