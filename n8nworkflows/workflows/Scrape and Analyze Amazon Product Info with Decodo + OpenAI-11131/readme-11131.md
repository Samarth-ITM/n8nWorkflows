Scrape and Analyze Amazon Product Info with Decodo + OpenAI

https://n8nworkflows.xyz/workflows/scrape-and-analyze-amazon-product-info-with-decodo---openai-11131


# Scrape and Analyze Amazon Product Info with Decodo + OpenAI

### 1. Workflow Overview

This workflow automates the end-to-end process of scraping, analyzing, and summarizing Amazon product information using Decodo’s web scraping API combined with OpenAI’s GPT models. It is designed to extract detailed product data, advertisements, and customer insights from a given Amazon product URL, then generate AI-powered descriptive summaries and competitive analyses. The results are structured, aggregated, and exported to Google Sheets for tracking, reporting, and further data handling.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Manual trigger and input setup with the Amazon product URL.
- **1.2 Data Scraping and Extraction:** Using Decodo to scrape Amazon product data and extract multiple data sets (product details, ads).
- **1.3 AI Processing and Analysis:** Multiple OpenAI GPT calls to generate descriptive summaries, competitive analysis, and product insights from extracted data.
- **1.4 Data Aggregation and Export:** Merging AI-generated outputs, aggregating data, and appending/updating the results into a Google Sheet.
- **1.5 Documentation Notes:** Several sticky notes explain workflow steps, setup instructions, disclaimers, and processing logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
This block starts the workflow manually and sets the initial input, namely the Amazon product URL to be processed.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Set the Input Fields

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow execution on demand.  
  - Configuration: No parameters; simply triggers the workflow.  
  - Input: None  
  - Output: Triggers the next node.  
  - Edge cases: None expected; manual start.

- **Set the Input Fields**  
  - Type: Set  
  - Role: Defines the input data for the workflow (Amazon product URL).  
  - Configuration: Sets a string field `product_url` with a hardcoded Amazon product URL (`https://www.amazon.in/Sony-DualSense-Controller-Grey-PlayStation/dp/B0BQXZ11B8`).  
  - Input: Trigger from manual trigger.  
  - Output: Passes the JSON with `product_url` to the scraper node.  
  - Edge cases: URL must be valid and accessible; incorrect URL formatting will cause scraping failures.

---

#### 1.2 Data Scraping and Extraction

**Overview:**  
This block uses Decodo to scrape the Amazon product page and extract structured data. Then, it splits the scraped data into separate components: product details and advertisements.

**Nodes Involved:**  
- Decodo Web scrape for Amazon Products  
- Extract Product Details  
- Extract Ads

**Node Details:**

- **Decodo Web scrape for Amazon Products**  
  - Type: Decodo API node (community Decodo integration)  
  - Role: Scrapes the Amazon product page using Decodo’s API with an "amazon" operation.  
  - Configuration: Uses `product_url` from previous node; credentials for Decodo API provided.  
  - Input: JSON with `product_url`  
  - Output: JSON response containing Amazon product data, including product details, ads, and other metadata.  
  - Edge cases:  
    - API key or quota issues leading to auth errors or rate limiting.  
    - Unavailable or protected Amazon pages causing empty or malformed data.  
    - Network timeouts or Decodo service downtime.  
  - Retry enabled on failure.

- **Extract Product Details**  
  - Type: Code node (JavaScript)  
  - Role: Extracts the product details subset from Decodo’s raw JSON response.  
  - Configuration: JavaScript code returns `results[0].content.results.product_details` from Decodo response.  
  - Input: Output of Decodo node.  
  - Output: JSON with product details.  
  - Edge cases: If product details are missing or Decodo response structure changes, extraction will fail or return undefined.

- **Extract Ads**  
  - Type: Code node (JavaScript)  
  - Role: Extracts advertisements data from Decodo’s response.  
  - Configuration: JavaScript code returns `results[0].content.results.ads`.  
  - Input: Output of Decodo node.  
  - Output: JSON with ads data.  
  - Edge cases: Same as product details extraction; missing or malformed ads data can cause issues.

---

#### 1.3 AI Processing and Analysis

**Overview:**  
This block processes the extracted data through multiple AI-powered steps using OpenAI GPT models. It generates descriptive summaries, competitive analysis, and product insights. Output parsers ensure structured JSON results.

**Nodes Involved:**  
- Product Descriptive Summarizer (Chain LLM)  
- OpenAI Chat Model  
- Structured Output Parser  
- Competitive Analysis (Information Extractor)  
- OpenAI Chat Model for Competitive Analysis  
- Product Insights (Information Extractor)  
- OpenAI Chat Model for Amazon Product Mining

**Node Details:**

- **Product Descriptive Summarizer**  
  - Type: Chain LLM (LangChain node)  
  - Role: Generates a descriptive summary of product details using OpenAI GPT.  
  - Configuration:  
    - Prompt: "Provide me a descriptive summary of the following product details\n\n{{ $json.toJsonString() }}"  
    - Message context: "You are an expert content analyzer and summary generator"  
    - Output parsing enabled.  
  - Input: Product details from "Extract Product Details".  
  - Output: Structured summary JSON.  
  - Edge cases:  
    - Model API errors or rate limits.  
    - Prompt failure if input data is empty or malformed.

- **OpenAI Chat Model**  
  - Type: OpenAI GPT Chat (LangChain integration)  
  - Role: Provides the language model for the Product Descriptive Summarizer chain.  
  - Configuration: Model set to "gpt-4.1-mini".  
  - Credentials: OpenAI API key.  
  - Input: Pass-through from previous.  
  - Output: Text completions.  
  - Edge cases: API errors, quota exceedance.

- **Structured Output Parser**  
  - Type: LangChain structured output parser  
  - Role: Parses the GPT output into a defined JSON schema with a "summary" string property.  
  - Configuration: Manual JSON schema specifying expected output structure.  
  - Input: GPT raw output.  
  - Output: Parsed JSON.  
  - Edge cases: Parsing errors if output does not conform to schema.

- **Competitive Analysis**  
  - Type: Information Extractor (LangChain)  
  - Role: Analyzes product details to produce a competitive positioning summary.  
  - Configuration:  
    - Prompt: "Analyze the following product details and provide a competitive positioning summary\n\n{{ $json.toJsonString() }}"  
    - Output schema with a "competitive_analysis" string.  
  - Input: Product details (same as descriptive summarizer output).  
  - Output: Competitive analysis text.  
  - Edge cases: Similar to summarizer.

- **OpenAI Chat Model for Competitive Analysis**  
  - Type: OpenAI GPT Chat  
  - Role: Language model for competitive analysis chain.  
  - Configuration: Model "gpt-4.1-mini".  
  - Credentials: OpenAI API key.  
  - Edge cases: Same as other GPT nodes.

- **Product Insights**  
  - Type: Information Extractor (LangChain)  
  - Role: Generates comprehensive insights from ads data, including best value item, most reviewed, price distribution, prime eligibility, rating insights, and recommendations.  
  - Configuration:  
    - Complex nested JSON schema defining expected output structure with multiple fields and required properties related to product insights.  
    - Contains a sub-workflow with nodes: Basic LLM Chain, OpenAI Chat Model, Structured Output Parser.  
  - Input: Ads data from "Extract Ads".  
  - Output: Structured insights JSON.  
  - Edge cases: Complexity may cause parsing errors; relies on consistent ads data.

- **OpenAI Chat Model for Amazon Product Mining**  
  - Type: OpenAI GPT Chat  
  - Role: Language model for Product Insights chain.  
  - Configuration: Model "gpt-4.1-mini".  
  - Credentials: OpenAI API key.  
  - Edge cases: Same as others.

---

#### 1.4 Data Aggregation and Export

**Overview:**  
This block merges all AI-generated outputs into one combined data stream, aggregates them, and appends or updates the information in a Google Sheet for persistent storage.

**Nodes Involved:**  
- Merge  
- Aggregate  
- Append or update row in sheet

**Node Details:**

- **Merge**  
  - Type: Merge node  
  - Role: Combines three inputs into one output stream. Inputs come from: Product Insights, Product Descriptive Summarizer, and Competitive Analysis.  
  - Configuration: Number of inputs set to 3.  
  - Input: Outputs from AI analysis nodes.  
  - Output: Unified data stream.  
  - Edge cases: If any input is missing or delayed, merge waits for all inputs.

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Aggregates merged data by concatenating or combining output fields.  
  - Configuration: Aggregates on field `output`.  
  - Input: Merged output.  
  - Output: Single aggregated record.  
  - Edge cases: Misalignment in field names may cause aggregation to fail.

- **Append or update row in sheet**  
  - Type: Google Sheets node  
  - Role: Saves aggregated AI outputs to a Google Sheet with append or update operation to avoid duplicates.  
  - Configuration:  
    - Document ID and Sheet name configured to a specific Google Sheet (`Amazon Product Info with Decodo`).  
    - Matching column set to `"output"` field to decide if append or update.  
    - Google OAuth2 credentials provided.  
  - Input: Aggregated output data.  
  - Output: Success or error status.  
  - Edge cases:  
    - Google API permission errors.  
    - Rate limiting.  
    - Sheet schema mismatch causing data rejection.

---

#### 1.5 Documentation Notes

**Overview:**  
Sticky notes provide detailed explanations on workflow logic, setup, disclaimers, and AI processing stages for maintainers and users.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6  
- Sticky Note7  
- Sticky Note8

**Node Details:**  
Sticky notes are purely informational and do not affect workflow logic but are crucial for understanding, maintenance, and onboarding.

---

### 3. Summary Table

| Node Name                             | Node Type                                  | Functional Role                           | Input Node(s)                             | Output Node(s)                            | Sticky Note                                                                                                                   |
|-------------------------------------|--------------------------------------------|-----------------------------------------|------------------------------------------|------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’    | Manual Trigger                            | Starts workflow manually                 | -                                        | Set the Input Fields                      |                                                                                                                               |
| Set the Input Fields                 | Set                                       | Defines initial product URL input        | When clicking ‘Execute workflow’         | Decodo Web scrape for Amazon Products    |                                                                                                                               |
| Decodo Web scrape for Amazon Products | Decodo API node                          | Scrapes Amazon product page              | Set the Input Fields                      | Extract Product Details, Extract Ads     |                                                                                                                               |
| Extract Product Details              | Code (JavaScript)                         | Extracts product details from Decodo response | Decodo Web scrape for Amazon Products    | Product Descriptive Summarizer, Competitive Analysis | Sticky Note8: "Custom Data Extract: Extract Ads, Product Details, Reviews and AI Summary from the Decodo Response"               |
| Extract Ads                         | Code (JavaScript)                         | Extracts ads data from Decodo response    | Decodo Web scrape for Amazon Products    | Product Insights                         | Sticky Note8                                                                                                                   |
| Product Descriptive Summarizer      | Chain LLM (LangChain)                     | Generates descriptive product summary    | Extract Product Details                   | Merge                                    | Sticky Note2: "Multi-stage AI evaluation using OpenAI. Generates product insights, summaries, competitive analysis."           |
| OpenAI Chat Model                   | OpenAI GPT Chat (LangChain)               | Provides GPT model for Descriptive Summarizer | -                                        | Product Descriptive Summarizer           | Sticky Note2                                                                                                                   |
| Structured Output Parser            | LangChain Output Parser                    | Parses GPT output into structured JSON   | OpenAI Chat Model                         | Product Descriptive Summarizer           |                                                                                                                               |
| Competitive Analysis                | Information Extractor (LangChain)          | Produces competitive positioning summary | Extract Product Details                   | Merge                                    | Sticky Note2                                                                                                                   |
| OpenAI Chat Model for Competitive Analysis | OpenAI GPT Chat (LangChain)          | GPT model for Competitive Analysis        | -                                        | Competitive Analysis                     |                                                                                                                               |
| Product Insights                   | Information Extractor (LangChain)          | Generates comprehensive product insights | Extract Ads                              | Merge                                    | Sticky Note2                                                                                                                   |
| OpenAI Chat Model for Amazon Product Mining | OpenAI GPT Chat (LangChain)          | GPT model for Product Insights            | -                                        | Product Insights                        |                                                                                                                               |
| Merge                             | Merge node                                | Combines AI outputs into single stream   | Product Insights, Product Descriptive Summarizer, Competitive Analysis | Aggregate                                | Sticky Note6: "Custom Data Extract"                                                                                           |
| Aggregate                        | Aggregate node                             | Aggregates merged data                    | Merge                                    | Append or update row in sheet            | Sticky Note7: "Export Data Handling: combines outputs, ensures no duplicates, updates Google Sheets."                         |
| Append or update row in sheet      | Google Sheets                             | Saves aggregated output to Google Sheets | Aggregate                                | -                                        |                                                                                                                               |
| Sticky Note                      | Sticky Note                               | Documentation and explanation             | -                                        | -                                        | Sticky Note: "Processing Steps: Scraping, Extraction, AI summary, Ads, Analysis Phase"                                         |
| Sticky Note1                      | Sticky Note                               | Instructions and setup guide              | -                                        | -                                        | Sticky Note1: Detailed setup instructions for Decodo and OpenAI credentials and usage notes                                    |
| Sticky Note2                      | Sticky Note                               | AI processing explanation                  | -                                        | -                                        | Sticky Note2: Multi-stage AI evaluation using OpenAI                                                                         |
| Sticky Note3                      | Sticky Note                               | Label for custom data extraction           | -                                        | -                                        | Sticky Note3: "1. Custom Data Extract"                                                                                         |
| Sticky Note4                      | Sticky Note                               | Label for AI insight analyzer              | -                                        | -                                        | Sticky Note4: "2. OpenAI Insight Analyzer"                                                                                    |
| Sticky Note5                      | Sticky Note                               | Disclaimer                                | -                                        | -                                        | Sticky Note5: Disclaimer about Decodo community node availability and hosting requirements                                     |
| Sticky Note6                      | Sticky Note                               | Label for merged data                      | -                                        | -                                        | Sticky Note6: "3. Export Data Handling"                                                                                        |
| Sticky Note7                      | Sticky Note                               | Label for export data processing           | -                                        | -                                        | Sticky Note7: "Export Data: combines outputs and writes to Google Sheets"                                                     |
| Sticky Note8                      | Sticky Note                               | Label for data extraction step             | -                                        | -                                        | Sticky Note8: "Custom Data Extract: Extract Ads, Product Details, Reviews and AI Summary from the Decodo Response"             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Add Set Node**  
   - Type: Set  
   - Purpose: Define `product_url` as a string variable.  
   - Parameters:  
     - Field name: `product_url`  
     - Value: Example URL, e.g., `https://www.amazon.in/Sony-DualSense-Controller-Grey-PlayStation/dp/B0BQXZ11B8`.

3. **Configure Decodo Web Scrape Node**  
   - Type: Decodo API node (community Decodo node)  
   - Parameters:  
     - URL: Expression `={{ $json.product_url }}`  
     - Operation: Set to "amazon"  
   - Credentials: Add Decodo API key credential (named e.g., "Decodo Credentials account").  
   - Enable retry on failure.

4. **Add JavaScript Code Node to Extract Product Details**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     return $input.first().json.results[0].content.results.product_details;
     ```  
   - Input: Connect output of Decodo node.

5. **Add JavaScript Code Node to Extract Ads**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     return $input.first().json.results[0].content.results.ads;
     ```  
   - Input: Connect output of Decodo node.

6. **Add Chain LLM Node for Product Descriptive Summary**  
   - Type: Chain LLM (LangChain)  
   - Parameters:  
     - Prompt text:  
       ```
       Provide me a descriptive summary of the following product details

       {{ $json.toJsonString() }}
       ```  
     - Message context: "You are an expert content analyzer and summary generator"  
     - Enable output parser.  
   - Input: Connect from "Extract Product Details".  
   - Retry enabled.

7. **Add OpenAI Chat Model Node (GPT-4.1-mini) for Descriptive Summary**  
   - Credentials: OpenAI API key (e.g., "OpenAi account").  
   - Model: Select "gpt-4.1-mini".  
   - Connect as AI Language Model to Chain LLM node.

8. **Add Structured Output Parser Node**  
   - Define manual JSON schema with a "summary" string property.  
   - Connect to OpenAI Chat Model node for parsing output.

9. **Add Competitive Analysis Information Extractor Node**  
   - Type: Information Extractor (LangChain)  
   - Prompt:  
     ```
     Analyze the following product details and provide a competitive positioning summary

     {{ $json.toJsonString() }}
     ```  
   - Output schema: single string property "competitive_analysis".  
   - Input: Connect from "Extract Product Details".  
   - Retry enabled.

10. **Add OpenAI Chat Model Node for Competitive Analysis**  
    - Credentials: Same OpenAI API key.  
    - Model: "gpt-4.1-mini".  
    - Connect as AI Language Model to Competitive Analysis node.

11. **Add Product Insights Information Extractor Node**  
    - Type: Information Extractor (LangChain)  
    - Parameters:  
      - Complex manual JSON schema defining multiple nested fields: summary, product insights, metadata, recommendations.  
      - Prompt and messages defined internally (uses sub-workflow with Basic LLM Chain, OpenAI Model, and Structured Output Parser).  
    - Input: Connect from "Extract Ads".  
    - Retry enabled.

12. **Add OpenAI Chat Model Node for Product Insights**  
    - Credentials: OpenAI API key.  
    - Model: "gpt-4.1-mini".  
    - Connect as AI Language Model to Product Insights node.

13. **Add Merge Node**  
    - Set number of inputs to 3.  
    - Connect outputs from:  
      - Product Insights  
      - Product Descriptive Summarizer  
      - Competitive Analysis

14. **Add Aggregate Node**  
    - Aggregate on field `output` to combine merged data into one record.  
    - Connect from Merge node.

15. **Add Google Sheets Node (Append or Update Row)**  
    - Operation: Append or Update  
    - Document ID: Set to your Google Sheet ID (e.g., `"1CHZLrQK-sJwn0fJMys88-ONg1HxUrErM09xu7wdfXSE"`)  
    - Sheet Name: `"gid=0"` or actual sheet name  
    - Matching Columns: `output` (to prevent duplicates)  
    - Column Mapping: Map `output` field to the sheet column  
    - Credentials: Google Sheets OAuth2 credentials configured  
    - Connect from Aggregate node.

16. **Add Sticky Notes for Documentation (Optional but Recommended)**  
    - Add notes for:  
      - Workflow processing steps  
      - Setup instructions for Decodo and OpenAI credentials  
      - AI processing explanation  
      - Disclaimer about Decodo node availability  
      - Section labels for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                                        |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates Amazon product data scraping and AI-powered analysis using Decodo and OpenAI GPT models. It requires Decodo API credentials and OpenAI API keys configured in n8n.                                                        | Setup instructions are detailed in Sticky Note1.                                                                                      |
| The Decodo node used is a community node available only on n8n self-hosted environments due to API and integration requirements.                                                                                                            | Sticky Note5 contains this disclaimer along with the Decodo logo.                                                                     |
| AI models used throughout are "gpt-4.1-mini" indicating a lightweight GPT-4 variant optimized for cost and speed. Prompts are customizable to change the style and depth of analysis.                                                          | Sticky Note2 explains multi-stage AI evaluation.                                                                                       |
| The workflow exports results to Google Sheets for easy tracking and reporting; users should ensure proper credentials and sheet permissions are configured.                                                                                   | Google Sheets node configured with OAuth2 and example spreadsheet IDs.                                                                |
| This workflow can be customized by modifying prompt texts, output schemas, and adding additional output nodes such as Slack, databases, or notifications for extended functionality.                                                           | Mentioned in Sticky Note1 under "Customize".                                                                                           |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. This process strictly respects prevailing content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available.