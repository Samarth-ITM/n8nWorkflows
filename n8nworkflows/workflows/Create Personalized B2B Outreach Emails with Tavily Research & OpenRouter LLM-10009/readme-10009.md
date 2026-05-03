Create Personalized B2B Outreach Emails with Tavily Research & OpenRouter LLM

https://n8nworkflows.xyz/workflows/create-personalized-b2b-outreach-emails-with-tavily-research---openrouter-llm-10009


# Create Personalized B2B Outreach Emails with Tavily Research & OpenRouter LLM

### 1. Workflow Overview

This workflow automates the generation of highly personalized B2B outreach emails by integrating AI-powered company research with large language model (LLM) driven email drafting. It is designed for sales, marketing, and business development teams seeking to efficiently scale customized email outreach based on real-time company insights.

The workflow is logically grouped into these key blocks:

- **1.1 Input Reception and Data Extraction:** Retrieves business card data (prospect and company details) from a Google Sheets document, filtering for ready-to-process entries.

- **1.2 AI-Powered Company Research:** Uses the Tavily tool combined with OpenRouter LLM to gather and synthesize company information such as overview, recent news, key offerings, and a comprehensive summary.

- **1.3 Outreach Message Generation:** Employs an LLM chain with a carefully crafted prompt that drafts a concise, professional, and personalized outreach email body in English, leveraging the researched company data.

- **1.4 Data Logging and Lead Registration:** Appends the generated company info and outreach message back into the Google Sheet and optionally registers the lead and message with Instantly.ai for automated email campaign integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Extraction

- **Overview:**  
  Starts the workflow manually, extracts prospect and company data from Google Sheets where status is “ready”, limits the number of processed items (for testing), and splits data into batches for sequential processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Get Business card data extraction  
  - Limit(Test)  
  - Loop Over Items  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user demand.  
    - No inputs; outputs trigger the next node.  
    - Failure: None expected.  

  - **Get Business card data extraction**  
    - Type: Google Sheets  
    - Role: Reads rows from a Google Sheet filtering where `status = "ready"`.  
    - Configuration: Reads from sheet "BusinessCardList" (gid=0) in a specific Google Sheet document.  
    - Credentials: Uses Google OAuth2 for authentication.  
    - Output: JSON array of business card entries including email, full_name, company_name, website_url, etc.  
    - Potential Failures: Auth errors, API limits, sheet not found, or filter misconfiguration.  

  - **Limit(Test)**  
    - Type: Limit  
    - Role: Limits the number of processed items to 10 for testing purposes.  
    - Configuration: maxItems = 10  
    - Output: Passes up to 10 items forward.  
    - Edge Cases: Should be removed or adjusted in production to allow full dataset processing.  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes each item individually in a batch of size 1 (default) to handle asynchronous AI calls sequentially.  
    - Input: Array of items from Limit(Test).  
    - Output: Single item per iteration to next block.  
    - Edge Cases: Batch size and error handling could affect throughput and failure recovery.

---

#### 2.2 AI-Powered Company Research

- **Overview:**  
  For each prospect’s company, performs AI research by querying Tavily to gather official company information, recent news, key offerings, and synthesizes a comprehensive summary via an LLM agent.

- **Nodes Involved:**  
  - Tavily  
  - OpenRouter Chat Model2  
  - Structured Output Parser1  
  - Company Research  

- **Node Details:**

  - **Tavily**  
    - Type: Tavily Tool Node (Search API)  
    - Role: Performs a time-constrained search (last year) with a max of 3 results, prioritizing official company info.  
    - Configuration: Receives a dynamic query string based on the company name and website URL from the current item.  
    - Credentials: Tavily API key.  
    - Potential Failures: API quota exceeded, no results found, network issues.  
    - Output: Raw search results passed to the AI agent.  

  - **OpenRouter Chat Model2**  
    - Type: LLM Chat Model (OpenRouter)  
    - Role: Provides the language model backend for the Company Research AI agent.  
    - Configuration: Uses default OpenRouter options.  
    - Credentials: OpenRouter API key.  
    - Edge Cases: API throttling, latency, or malformed input issues.  

  - **Structured Output Parser1**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses the AI agent's JSON-formatted response according to a specific schema including companyOverview, recentWebsiteNews, keyOfferings, comprehensiveSummary.  
    - Configuration: JSON schema example guides the parsing.  
    - Edge Cases: Malformed JSON responses, parsing errors.  

  - **Company Research**  
    - Type: LangChain Agent Node with AI Tool Integration  
    - Role: Orchestrates Tavily search and uses the LLM to synthesize company information into structured JSON following core directives: prioritizing official site info, recent news, and conciseness.  
    - Configuration: System prompt mandates exclusive use of Tavily, factual summaries, and output format.  
    - Input: Company name and website URL from the current batch item.  
    - Output: Structured JSON with four keys (companyOverview, recentWebsiteNews, keyOfferings, comprehensiveSummary).  
    - Failure Modes: Empty or irrelevant Tavily results, LLM timeout, unexpected output format.

---

#### 2.3 Outreach Message Generation

- **Overview:**  
  Generates a tailored, professional, and concise outreach email body in English using the company research output and prospect details.

- **Nodes Involved:**  
  - OpenRouter Chat Model3  
  - Generate Outreach Message  

- **Node Details:**

  - **OpenRouter Chat Model3**  
    - Type: LLM Chat Model (OpenRouter)  
    - Role: Provides the language model for composing the outreach email.  
    - Credentials: OpenRouter API key.  
    - Edge Cases: API rate limits, latency, or invalid prompt structure.  

  - **Generate Outreach Message**  
    - Type: LangChain LLM Chain  
    - Role: Constructs the email text using a prompt that instructs the model to:  
      - Analyze company background for a strong hook  
      - Introduce YOUR_COMPANY_NAME’s AI training and automation services  
      - Explain relevance and benefits specifically tailored to the prospect’s company  
      - Suggest a soft call to action  
      - Adhere to 100–150 words and English only  
      - Output only the plain email body text (no greeting or signature)  
    - Configuration: Prompt templates include dynamic variables for prospect name, company name, and comprehensive summary from company research.  
    - Input: Data from Company Research node and Loop Over Items.  
    - OnError: Continues with regular output to avoid workflow stoppage.  
    - Edge Cases: Insufficient company background info may lead to generic messaging; prompt complexity may cause unexpected outputs.  

---

#### 2.4 Data Logging and Lead Registration

- **Overview:**  
  Stores the generated company info and outreach message back into the Google Sheet, updating or appending records, and registers the lead with the Instantly.ai platform to enable direct email campaign execution.

- **Nodes Involved:**  
  - Add Company Info to Google Sheet  
  - Add Lead to Instantly AI  

- **Node Details:**

  - **Add Company Info to Google Sheet**  
    - Type: Google Sheets (appendOrUpdate)  
    - Role: Updates or appends data for each company including status, company name, key offerings, summary, background, recent news, and outreach message.  
    - Configuration: Matches rows by `company_name` to update existing entries or append new ones.  
    - Credentials: Google OAuth2.  
    - Edge Cases: Race conditions if multiple updates on same company; permission errors; Google API quota limits.  

  - **Add Lead to Instantly AI**  
    - Type: HTTP Request Node  
    - Role: Sends a POST request to Instantly.ai API to register the lead with email, name, company, and custom outreach message.  
    - Configuration:  
      - URL: Instantly.ai leads API endpoint  
      - Headers: Authorization Bearer token and content-type application/json  
      - JSON Body: Includes campaign ID, email, full name, company name, and a custom variable containing the outreach message text.  
    - Credentials: Requires valid Instantly.ai API token and campaign ID.  
    - Edge Cases: Authentication failure, invalid or missing data, network errors, API rate limits.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                                       | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                         |
|-------------------------------|----------------------------------|-----------------------------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Starts the workflow manually                         | —                            | Get Business card data extraction | ## ① Business card data extraction: Click to start → Retrieve business card data from Google Sheets.              |
| Get Business card data extraction | Google Sheets                   | Extracts business card data where status = “ready”  | When clicking ‘Execute workflow’ | Limit(Test)                   | ## ① Business card data extraction: Retrieve business card data from Google Sheets.                                |
| Limit(Test)                   | Limit                            | Limits items to 10 for testing                       | Get Business card data extraction | Loop Over Items               | ## ① Business card data extraction: Limit node for testing, should be removed in production.                        |
| Loop Over Items               | SplitInBatches                   | Processes each item individually                      | Limit(Test)                  | Company Research              | ## ② Pick up company information and generate an outreach message: AI uses Tavily search and generates email body. |
| Tavily                       | Tavily Tool                      | Searches company information using Tavily API      | Company Research (agent input) | Company Research              | ## ② Pick up company information and generate an outreach message: Uses Tavily tool for company info search.        |
| OpenRouter Chat Model2        | LLM Chat Model (OpenRouter)      | Provides LLM for company research                    | —                            | Company Research              | ## ② Pick up company information and generate an outreach message: LLM for company research agent.                  |
| Structured Output Parser1     | LangChain Structured Output Parser | Parses AI JSON output into structured fields        | Company Research             | Company Research              | ## ② Pick up company information and generate an outreach message: Parses structured JSON from AI agent.           |
| Company Research             | LangChain Agent (AI Tool)        | Synthesizes company overview, news, offerings       | Tavily, OpenRouter Chat Model2, Structured Output Parser1 | Generate Outreach Message | ## ② Pick up company information and generate an outreach message: Synthesizes company info for email generation.   |
| OpenRouter Chat Model3        | LLM Chat Model (OpenRouter)      | Provides LLM for outreach message generation         | —                            | Generate Outreach Message     | ## ② Pick up company information and generate an outreach message: LLM to draft personalized outreach email.        |
| Generate Outreach Message     | LangChain LLM Chain              | Drafts personalized outreach email body              | Company Research             | Add Company Info to Google Sheet | ## ② Pick up company information and generate an outreach message: Drafts email body using company research data.   |
| Add Company Info to Google Sheet | Google Sheets                   | Updates/appends company info and outreach message   | Generate Outreach Message    | Add Lead to Instantly AI       | ## ③ Register leads to instantlyAI: Logs data for review or dispatch in Google Sheets.                              |
| Add Lead to Instantly AI      | HTTP Request                    | Registers lead and email content with Instantly.ai | Add Company Info to Google Sheet | Loop Over Items               | ## ③ Register leads to instantlyAI: Sends lead and email content to Instantly.ai campaign.                           |
| Sticky Note                  | Sticky Note                     | Documentation note                                   | —                            | —                             | ## ② Pick up company information and generate an outreach message: Tavily and LLM integration explanation.          |
| Sticky Note1                 | Sticky Note                     | Documentation note                                   | —                            | —                             | ## ① Business card data extraction: Instructions and testing note.                                                 |
| Sticky Note2                 | Sticky Note                     | Documentation note                                   | —                            | —                             | ## ③ Register leads to instantlyAI: Explanation of lead registration step.                                         |
| Sticky Note3                 | Sticky Note                     | Documentation note                                   | —                            | —                             | Workflow description, purpose, features, and automation overview.                                                  |
| Sticky Note4                 | Sticky Note                     | Documentation note                                   | —                            | —                             | Use cases for sales, business development, operations, and AI automation learners.                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger (to start workflow manually).

2. **Add Google Sheets Node for Business Cards**  
   - Name: `Get Business card data extraction`  
   - Operation: Read rows  
   - Document ID: Your Google Sheet ID containing business card data  
   - Sheet Name: The sheet tab name or gid (e.g., "gid=0")  
   - Filters: Set filter where `status` equals `"ready"`  
   - Credentials: Connect your Google OAuth2 credentials  
   - Connect output of Manual Trigger to this node.

3. **Add Limit Node (optional for testing)**  
   - Name: `Limit(Test)`  
   - Parameters: maxItems = 10 (adjust or remove for production)  
   - Connect output of Google Sheets node to this node.

4. **Add SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Parameters: Default batch size (1)  
   - Connect output of Limit node to this node.

5. **Add Tavily Node**  
   - Name: `Tavily`  
   - Parameters:  
     - Query: Dynamic expression using current item’s `company_name` and `website_url`  
     - Options: Time range = 1 year, max results = 3, search depth = basic  
   - Credentials: Connect your Tavily API credentials  
   - This node will be used internally by the LangChain Agent.

6. **Add OpenRouter Chat Model Node for Company Research**  
   - Name: `OpenRouter Chat Model2`  
   - Credentials: Connect OpenRouter API key  
   - Use default options.

7. **Add Structured Output Parser Node**  
   - Name: `Structured Output Parser1`  
   - Parameters: JSON schema example with keys: companyOverview, recentWebsiteNews, keyOfferings, comprehensiveSummary.

8. **Add LangChain Agent Node**  
   - Name: `Company Research`  
   - Parameters:  
     - System message and prompt instructing use of Tavily tool exclusively, focusing on official site info, recent news, and key offerings  
     - Output parser set to `Structured Output Parser1`  
     - AI tool linked to `Tavily` node  
     - Language model linked to `OpenRouter Chat Model2`  
   - Connect output of `Loop Over Items` to this node.

9. **Add OpenRouter Chat Model Node for Outreach Message**  
   - Name: `OpenRouter Chat Model3`  
   - Credentials: Connect OpenRouter API key  
   - Use default options.

10. **Add LangChain LLM Chain Node**  
    - Name: `Generate Outreach Message`  
    - Parameters:  
      - Prompt includes variables: prospect name, company name, comprehensive summary from Company Research  
      - Instructions to craft a 100-150 word personalized outreach email body in English only  
    - Language model linked to `OpenRouter Chat Model3`  
    - Connect output of `Company Research` to this node.

11. **Add Google Sheets Node for Logging**  
    - Name: `Add Company Info to Google Sheet`  
    - Operation: Append or Update rows  
    - Document ID and Sheet Name: Same as business card sheet or separate log sheet  
    - Mapping: Include status = "complete", company_name, key offerings, company summary, outreach message, and other relevant columns  
    - Credentials: Google OAuth2  
    - Connect output of `Generate Outreach Message` to this node.

12. **Add HTTP Request Node for Instantly.ai Lead Registration**  
    - Name: `Add Lead to Instantly AI`  
    - HTTP Method: POST  
    - URL: Instantly.ai leads API endpoint (e.g., https://api.instantly.ai/api/v2/leads)  
    - Headers: Authorization Bearer token, Content-Type application/json  
    - Body (JSON): Include campaign ID, email, full name, company name, custom variables with outreach message text  
    - Connect output of Google Sheets logging node to this node.

13. **Connect output of Instantly.ai node back to `Loop Over Items` node**  
    - This enables processing the next batch item until all are processed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow automates personalized B2B outreach by combining AI research and LLM-generated emails, with optional Instantly.ai integration.   | Workflow description via n8n workflow metadata and sticky notes.                                              |
| Use Tavily tool exclusively for company info search to maintain accuracy and official source prioritization.                               | System prompt in `Company Research` node.                                                                      |
| Replace `"YOUR_COMPANY_NAME"` and `"YOUR_TOKEN_HERE"` placeholders with your actual company name and API tokens in prompts and HTTP headers.| Prompt in `Generate Outreach Message` and HTTP Request node configuration.                                     |
| Remove or adjust the `Limit(Test)` node in production to process all business card entries.                                                | Sticky Note1 and node description.                                                                             |
| Google Sheets document and sheet IDs must be accessible and shared appropriately to avoid permission errors.                               | Google Sheets nodes configuration.                                                                             |
| Instantly.ai API requires a valid campaign ID and bearer token for lead registration.                                                      | HTTP Request node configuration for Instantly.ai.                                                              |
| Ensure OpenRouter API credentials are valid and have sufficient quota for LLM interactions.                                                 | OpenRouter Chat Model nodes configuration.                                                                     |
| Workflow examples and architecture inspired by best practices in AI-driven sales outreach automation.                                       | Sticky Note3 and Sticky Note4 content.                                                                          |

---

**Disclaimer:**  
The provided text is exclusively sourced from an automated workflow created using n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.