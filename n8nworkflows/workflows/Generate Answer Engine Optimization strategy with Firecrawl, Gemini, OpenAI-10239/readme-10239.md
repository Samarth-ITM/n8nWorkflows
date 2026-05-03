Generate Answer Engine Optimization strategy with Firecrawl, Gemini, OpenAI

https://n8nworkflows.xyz/workflows/generate-answer-engine-optimization-strategy-with-firecrawl--gemini--openai-10239


# Generate Answer Engine Optimization strategy with Firecrawl, Gemini, OpenAI

### 1. Workflow Overview

This workflow, titled **"Generate AEO strategy from brand input using AI competitor analysis"**, is designed to automate the creation of a detailed Answer Engine Optimization (AEO) strategy for a brand. It is targeted at digital marketers, SEO agencies, brand strategists, and content teams aiming to optimize content for AI-powered search engines like ChatGPT, Google SGE, and Bing Chat.

The workflow includes the following main logical blocks:

- **1.1 Input Reception:** Collects brand data from users via a web form.
- **1.2 AI Competitor Analysis:** Uses Google Gemini AI to identify the top 3 direct competitors with detailed reasoning and structured JSON output.
- **1.3 Competitor Data Parsing & Splitting:** Extracts competitor info from AI output and prepares individual entries for scraping.
- **1.4 Web Scraping (Firecrawl):** Scrapes the main brand website and competitor websites in parallel to extract structured content-related data.
- **1.5 Scraping Job Management:** Manages Firecrawl job IDs, includes wait nodes for asynchronous scrape completion, and fetches scrape results.
- **1.6 Data Merging:** Combines scraped data from brand and competitors for comprehensive analysis.
- **1.7 AEO Strategy Generation:** Uses OpenAI GPT-4 to generate a detailed AEO strategy report based on the merged data.
- **1.8 Email Formatting and Delivery:** Formats the AI-generated strategy as a professional HTML email and sends it to the user’s provided email via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Collects brand-related inputs from users through a web form, initiating the workflow.
- **Nodes Involved:** 
  - Form Trigger - Brand Input
  - Set - Brand Variables
- **Node Details:**

  - **Form Trigger - Brand Input**
    - Type: Form Trigger
    - Role: Captures user input submitted via a web form.
    - Configuration: 
      - Form fields: Brand Name, Website Link, Niche (D2C/B2C), Product Type, Email (required).
      - Webhook ID configured to receive form submissions.
      - Form title and description customizable.
    - Inputs: External HTTP form submission.
    - Outputs: JSON data containing form fields.
    - Edge Cases: Missing required fields will prevent submission; webhook URL must be publicly accessible.
    - Version: 2.3
  
  - **Set - Brand Variables**
    - Type: Set
    - Role: Maps form inputs to workflow variables with clear naming.
    - Configuration: Assigns variables: brand_name, website_url, niche, product_type, user_email from form JSON.
    - Inputs: From Form Trigger node.
    - Outputs: JSON with normalized variable names.
    - Edge Cases: Requires consistent input naming; errors if input fields missing.
    - Version: 3.4

#### 2.2 AI Competitor Analysis

- **Overview:** Uses Google Gemini AI model to analyze brand data and identify the top 3 direct competitors with detailed scoring and explanation.
- **Nodes Involved:**
  - AI Agent - Identify Competitors
  - Google Gemini Chat Model
  - Parse Competitor Data
  - Split Out - Individual Competitors
- **Node Details:**

  - **AI Agent - Identify Competitors**
    - Type: LangChain Agent node
    - Role: Sends a detailed prompt defining competitor identification criteria and output format.
    - Configuration: 
      - Large system prompt describing role and analysis criteria.
      - User prompt with input variables (brand_name, website_url, niche, product_type).
      - Output expected in strict JSON format with competitors ranked 1-3.
    - Inputs: Brand variables from Set node.
    - Outputs: AI raw text response.
    - Edge Cases: AI output format errors, incomplete JSON, or hallucinations.
    - Version: 2.2

  - **Google Gemini Chat Model**
    - Type: LangChain Google Gemini model
    - Role: Executes AI prompt for competitor identification.
    - Configuration: Uses Google Gemini API credentials.
    - Inputs: AI Agent node call.
    - Outputs: AI JSON text response.
    - Edge Cases: API auth failure, rate limits, timeouts.
    - Version: 1

  - **Parse Competitor Data**
    - Type: Code node (JavaScript)
    - Role: Parses AI text response to extract structured competitor JSON data.
    - Configuration: Attempts multiple parsing strategies (direct JSON parse, extract from markdown code block, regex search).
    - Inputs: AI Agent raw output.
    - Outputs: JSON array of competitor objects with name, URL, overlap, reasoning.
    - Edge Cases: Parsing failures, malformed AI responses.
    - Version: 2

  - **Split Out - Individual Competitors**
    - Type: SplitOut node
    - Role: Splits competitor array into individual items for parallel processing.
    - Configuration: Splits on `competitors` field.
    - Inputs: Parsed competitor data.
    - Outputs: Individual competitor JSON objects.
    - Edge Cases: Empty competitor list.
    - Version: 1

#### 2.3 Web Scraping (Firecrawl)

- **Overview:** Uses Firecrawl API to scrape structured content data from the brand and competitor websites.
- **Nodes Involved:**
  - Scrape Target Brand Website
  - Scrape Competitor Websites
  - Wait - Brand Scraping
  - Wait - Competitor Scraping
  - Extract Brand Job ID
  - Extract Competitor Job IDs
  - Get Brand Scrape Results
  - Get Competitor Scrape Results
- **Node Details:**

  - **Scrape Target Brand Website**
    - Type: HTTP Request
    - Role: Sends POST request to Firecrawl to scrape main brand website.
    - Configuration: 
      - URL: Firecrawl API endpoint `/v2/extract`
      - Body includes target brand URL and JSON schema defining expected fields (company_name, products_services, value_proposition, target_audience, key_features, pricing_info, content_themes).
      - Auth: HTTP header authentication with Firecrawl API key.
      - Content-Type: application/json
    - Inputs: Brand website URL from variables.
    - Outputs: Firecrawl job creation response.
    - Edge Cases: API errors, invalid URL, authorization failure.
    - Version: 4.2

  - **Scrape Competitor Websites**
    - Type: HTTP Request
    - Role: Sends POST requests in parallel to Firecrawl for each competitor website.
    - Configuration: Same as brand scrape, but URL is competitor_url from split items.
    - Inputs: Individual competitor URL items.
    - Outputs: Firecrawl job creation responses.
    - Edge Cases: Parallel API limits, invalid competitor URLs.
    - Version: 4.2

  - **Wait - Brand Scraping & Wait - Competitor Scraping**
    - Type: Wait node
    - Role: Pauses the workflow for 60 seconds to allow Firecrawl to complete scraping jobs.
    - Configuration: Fixed wait time of 60 seconds.
    - Inputs: Firecrawl job creation responses.
    - Outputs: Triggers extraction of job IDs afterward.
    - Edge Cases: Scraping taking longer than wait time (may cause incomplete data).
    - Version: 1.1

  - **Extract Brand Job ID & Extract Competitor Job IDs**
    - Type: Code node
    - Role: Extracts Firecrawl job IDs from the initial job creation responses.
    - Configuration: Reads job ID and URL from API response JSON.
    - Inputs: Firecrawl job responses.
    - Outputs: JSON with job_id and URL.
    - Edge Cases: Missing or malformed job ID in response.
    - Version: 2

  - **Get Brand Scrape Results & Get Competitor Scrape Results**
    - Type: HTTP Request
    - Role: Polls Firecrawl API to retrieve completed scrape data using job IDs.
    - Configuration: GET request to Firecrawl `/v2/extract/{job_id}` endpoint with auth.
    - Inputs: Job ID JSON.
    - Outputs: Scraped structured data as per schema.
    - Edge Cases: Job not ready, API errors, data missing.
    - Version: 4.2

#### 2.4 Data Merging

- **Overview:** Combines the brand scrape results and all competitor scrape results into a single dataset for further AI analysis.
- **Nodes Involved:**
  - Merge Brand & Competitor Data
- **Node Details:**

  - **Merge Brand & Competitor Data**
    - Type: Merge node
    - Role: Combines two input streams (brand data and competitor data) into one combined JSON object.
    - Configuration: Mode set to "combine".
    - Inputs: Scrape results of brand and competitors.
    - Outputs: Unified JSON containing brand_data and competitor_data.
    - Edge Cases: Missing data from either side causes incomplete merge.
    - Version: 3

#### 2.5 AEO Strategy Generation

- **Overview:** Uses OpenAI GPT-4 to generate a comprehensive, actionable AEO strategy report based on merged brand and competitor data.
- **Nodes Involved:**
  - AI Agent - Generate AEO Strategy
  - OpenAI Chat Model
- **Node Details:**

  - **AI Agent - Generate AEO Strategy**
    - Type: LangChain Agent
    - Role: Sends detailed prompt instructing GPT-4 to analyze inputs and generate a structured AEO strategy report in HTML.
    - Configuration: 
      - Prompt includes context about AEO, inputs with brand_data and competitor_data placeholders.
      - Specifies sections: Executive Summary, Competitive Analysis, Recommendations, Content Priority Matrix, Next Steps.
      - Output format: well-structured HTML.
    - Inputs: Merged data JSON.
    - Outputs: AI-generated HTML strategy report.
    - Edge Cases: AI hallucination, incomplete generation, API errors.
    - Version: 2.2

  - **OpenAI Chat Model**
    - Type: LangChain OpenAI GPT-4 model
    - Role: Executes AI prompt.
    - Configuration: Model set to "gpt-4o" (GPT-4 optimized).
    - Inputs: AI Agent call.
    - Outputs: HTML strategy content.
    - Edge Cases: API quota, auth errors.
    - Version: 1

#### 2.6 Email Formatting and Delivery

- **Overview:** Formats the AI-generated AEO strategy HTML content into a professional email template and sends it to the user via Gmail.
- **Nodes Involved:**
  - Format Email Content
  - Send Email via Gmail
- **Node Details:**

  - **Format Email Content**
    - Type: Code node (JavaScript)
    - Role: Wraps AI-generated HTML report into a branded email template with styling and footer.
    - Configuration: 
      - Uses current date.
      - Inserts brand name and user email.
      - Applies CSS styles for readability.
    - Inputs: AI-generated HTML and user email.
    - Outputs: JSON with email fields: to, subject, html.
    - Edge Cases: Missing email address, malformed HTML.
    - Version: 2

  - **Send Email via Gmail**
    - Type: Gmail node
    - Role: Sends the formatted email to the user.
    - Configuration: 
      - Uses OAuth2 Gmail credentials.
      - Dynamic "to", "subject", and "message" fields from previous node.
    - Inputs: Email JSON.
    - Outputs: Email send confirmation.
    - Edge Cases: Gmail auth failure, quota limits, invalid recipient email.
    - Version: 2.1

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                         | Input Node(s)                        | Output Node(s)                               | Sticky Note                                                                                       |
|-------------------------------|----------------------------------|---------------------------------------|------------------------------------|----------------------------------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note - Main Explanation | Sticky Note                      | Workflow overview and explanation     | -                                  | -                                            | ## 🎯 AEO Strategy Generator with AI Competitor Analysis ... Full step description and setup instructions. |
| Sticky Note - Step 1           | Sticky Note                      | Explains Step 1: Input form collection| -                                  | -                                            | ## 📋 Step 1: Collect Brand Information ... Form trigger creates webhook to capture inputs.       |
| Sticky Note - Step 2           | Sticky Note                      | Explains Step 2: Competitor analysis  | -                                  | -                                            | ## 🔍 Step 2: AI Competitor Analysis ... Google Gemini AI identifies top 3 competitors.           |
| Sticky Note - Step 3           | Sticky Note                      | Explains Step 3: Web scraping         | -                                  | -                                            | ## 🌐 Step 3: Web Scraping ... Firecrawl scrapes brand and competitor websites in parallel.        |
| Sticky Note - Step 4           | Sticky Note                      | Explains Step 4: AEO Strategy Gen     | -                                  | -                                            | ## 🤖 Step 4: AEO Strategy Generation ... OpenAI GPT-4 generates detailed recommendations.         |
| Sticky Note - Step 5           | Sticky Note                      | Explains Step 5: Email delivery       | -                                  | -                                            | ## 📧 Step 5: Deliver Results ... Formatted HTML email sent with full report.                      |
| Form Trigger - Brand Input     | Form Trigger                    | Captures brand input from user form   | -                                  | Set - Brand Variables                         | See Step 1 sticky note                                                                           |
| Set - Brand Variables          | Set                             | Normalizes form inputs into variables | Form Trigger - Brand Input          | AI Agent - Identify Competitors               | See Step 1 sticky note                                                                           |
| AI Agent - Identify Competitors| LangChain Agent                 | Runs AI prompt to identify competitors| Set - Brand Variables              | Parse Competitor Data                         | See Step 2 sticky note                                                                           |
| Google Gemini Chat Model       | LangChain Google Gemini Model   | Executes Gemini API for competitor ID | AI Agent - Identify Competitors     | AI Agent - Identify Competitors               | See Step 2 sticky note                                                                           |
| Parse Competitor Data          | Code                           | Parses AI response JSON for competitors| AI Agent - Identify Competitors    | Split Out - Individual Competitors             | See Step 2 sticky note                                                                           |
| Split Out - Individual Competitors | SplitOut                   | Splits competitors array into items   | Parse Competitor Data              | Scrape Target Brand Website, Scrape Competitor Websites | See Step 2 sticky note                                                                           |
| Scrape Target Brand Website    | HTTP Request                   | Initiates Firecrawl scrape for brand  | Split Out - Individual Competitors | Wait - Brand Scraping                         | See Step 3 sticky note                                                                           |
| Scrape Competitor Websites     | HTTP Request                   | Initiates Firecrawl scrape for competitors | Split Out - Individual Competitors | Wait - Competitor Scraping                     | See Step 3 sticky note                                                                           |
| Wait - Brand Scraping          | Wait                           | Waits 60s for brand scraping to complete | Scrape Target Brand Website       | Extract Brand Job ID                          | See Step 3 sticky note                                                                           |
| Wait - Competitor Scraping     | Wait                           | Waits 60s for competitor scraping     | Scrape Competitor Websites          | Extract Competitor Job IDs                     | See Step 3 sticky note                                                                           |
| Extract Brand Job ID           | Code                           | Extracts Firecrawl job ID from response | Wait - Brand Scraping             | Get Brand Scrape Results                       | See Step 3 sticky note                                                                           |
| Extract Competitor Job IDs     | Code                           | Extracts Firecrawl job ID from response | Wait - Competitor Scraping        | Get Competitor Scrape Results                  | See Step 3 sticky note                                                                           |
| Get Brand Scrape Results       | HTTP Request                   | Fetches completed brand scrape data   | Extract Brand Job ID               | Merge Brand & Competitor Data                  | See Step 3 sticky note                                                                           |
| Get Competitor Scrape Results  | HTTP Request                   | Fetches completed competitor scrape data | Extract Competitor Job IDs       | Merge Brand & Competitor Data                  | See Step 3 sticky note                                                                           |
| Merge Brand & Competitor Data  | Merge                          | Combines brand and competitor scrape data | Get Brand Scrape Results, Get Competitor Scrape Results | AI Agent - Generate AEO Strategy              | See Step 4 sticky note                                                                           |
| AI Agent - Generate AEO Strategy | LangChain Agent             | Runs AI prompt to generate AEO strategy report | Merge Brand & Competitor Data     | Format Email Content                           | See Step 4 sticky note                                                                           |
| OpenAI Chat Model              | LangChain OpenAI GPT-4 Model   | Executes OpenAI GPT-4 for AEO strategy generation | AI Agent - Generate AEO Strategy | AI Agent - Generate AEO Strategy               | See Step 4 sticky note                                                                           |
| Format Email Content           | Code                           | Formats AEO strategy report as styled HTML email | AI Agent - Generate AEO Strategy | Send Email via Gmail                           | See Step 5 sticky note                                                                           |
| Send Email via Gmail           | Gmail                          | Sends the formatted strategy email    | Format Email Content              | -                                              | See Step 5 sticky note                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**
   - Node Type: Form Trigger
   - Configure form with fields:
     - Brand Name (text, required)
     - Website Link (text, required)
     - Niche (D2C or B2C, required)
     - Product Type (text, required)
     - Email (email, required)
   - Set form title to "AEO Strategy Generator" and add description.
   - Deploy webhook and note URL.

2. **Create Set Node: Brand Variables**
   - Input: Form Trigger
   - Assign variables:
     - brand_name = {{$json["Brand Name"]}}
     - website_url = {{$json["Website Link"]}}
     - niche = {{$json["Niche (D2C or B2C)"]}}
     - product_type = {{$json["Product Type"]}}
     - user_email = {{$json.Email}}

3. **Create AI Agent Node: Identify Competitors**
   - Input: Set - Brand Variables
   - Type: LangChain Agent
   - Paste detailed competitor identification prompt (as specified).
   - Use templating to insert variables.
   - Ensure proper output format JSON.
   - Connect to Google Gemini Chat Model.

4. **Create Google Gemini Chat Model Node**
   - Credentials: Google Gemini API key
   - Connect output to AI Agent - Identify Competitors.

5. **Create Code Node: Parse Competitor Data**
   - Input: AI Agent - Identify Competitors
   - Paste JS code to extract JSON from AI response.
   - Output structured competitor array.

6. **Create SplitOut Node: Individual Competitors**
   - Input: Parse Competitor Data
   - Split on `competitors` field.

7. **Create HTTP Request Node: Scrape Target Brand Website**
   - Input: SplitOut
   - POST to Firecrawl API `/v2/extract`
   - Body includes:
     - url: {{$json.website_url}}
     - schema: JSON schema defining expected fields.
   - Auth: HTTP Header Auth with Firecrawl API key.
   - Content-Type: application/json

8. **Create HTTP Request Node: Scrape Competitor Websites**
   - Input: SplitOut
   - POST to Firecrawl API `/v2/extract`
   - Body includes:
     - url: {{$json.competitor_url}}
     - schema: same JSON schema.
   - Auth: HTTP Header Auth with Firecrawl API key.
   - Content-Type: application/json

9. **Create Wait Nodes**
   - Wait - Brand Scraping (60 seconds)
     - Input: Scrape Target Brand Website
   - Wait - Competitor Scraping (60 seconds)
     - Input: Scrape Competitor Websites

10. **Create Code Nodes to Extract Job IDs**
    - Extract Brand Job ID from brand scrape response.
    - Extract Competitor Job IDs from competitor scrape responses.

11. **Create HTTP Request Nodes to Get Scrape Results**
    - Get Brand Scrape Results
      - GET Firecrawl `/v2/extract/{{job_id}}`
      - Auth: Firecrawl API key
      - Input: Extract Brand Job ID
    - Get Competitor Scrape Results
      - GET Firecrawl `/v2/extract/{{job_id}}`
      - Auth: Firecrawl API key
      - Input: Extract Competitor Job IDs

12. **Create Merge Node: Merge Brand & Competitor Data**
    - Inputs: Get Brand Scrape Results, Get Competitor Scrape Results
    - Mode: Combine

13. **Create AI Agent Node: Generate AEO Strategy**
    - Input: Merge Brand & Competitor Data
    - Paste detailed AEO generation prompt.
    - Use variables for brand_data and competitor_data.
    - Connect to OpenAI Chat Model.

14. **Create OpenAI Chat Model Node**
    - Credentials: OpenAI API key (GPT-4)
    - Model: gpt-4o (optimized GPT-4)
    - Input: AI Agent - Generate AEO Strategy

15. **Create Code Node: Format Email Content**
    - Input: AI Agent - Generate AEO Strategy
    - Paste JS code that wraps AI HTML in email template.
    - Output email fields: to, subject, html.

16. **Create Gmail Node: Send Email via Gmail**
    - Credentials: Gmail OAuth2 account
    - Use dynamic fields for recipient, subject, and message.
    - Input: Format Email Content

17. **Connect all nodes in execution order:**
    - Form Trigger -> Set Variables -> AI Agent Identify Competitors -> Google Gemini Chat Model -> Parse Competitor Data -> Split Out Competitors -> [Scrape Brand & Scrape Competitors in parallel] -> Wait nodes -> Extract Job IDs -> Get Scrape Results -> Merge Data -> AI Agent Generate Strategy -> OpenAI Chat Model -> Format Email -> Gmail Send.

18. **Configure all credentials:**
    - Google Gemini API
    - OpenAI API key
    - Firecrawl API key (HTTP Header Auth)
    - Gmail OAuth2 credentials

19. **Test workflow end-to-end with real brand input.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                       | Context or Link                                                                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow generates an actionable AEO strategy using AI competitor analysis and web scraping, optimizing for AI-powered search engines.         | Main workflow purpose                                                                                                                                    |
| Requires API keys for Google Gemini (PaLM), OpenAI GPT-4, Firecrawl API, and Gmail OAuth2 for email sending.                                        | Credentials setup                                                                                                                                       |
| The Firecrawl scraper schema defines structured extraction fields like products, value propositions, and content themes for consistent data output.| Firecrawl API documentation (https://firecrawl.dev/docs)                                                                                               |
| AI prompts are carefully designed to enforce JSON output and quality checks to minimize parsing errors and hallucinations.                         | Prompt engineering best practices                                                                                                                      |
| Wait nodes use fixed 60-second delays for scraping jobs; consider extending if scraping large sites or slow responses occur.                        | Potential enhancement for job polling                                                                                                                  |
| The email template includes branding and styling to ensure a professional appearance for recipients.                                               | Email formatting best practices                                                                                                                        |
| The workflow can be customized by changing AI models, competitor count, additional data sources, or output formats such as Slack notification.     | Customization instructions in main sticky note                                                                                                        |
| Example blog post explaining AEO and competitive intelligence strategies: https://www.example.com/aeo-competitive-intelligence                      | External resource (hypothetical)                                                                                                                      |

---

This completes the comprehensive, structured documentation of the provided n8n workflow.