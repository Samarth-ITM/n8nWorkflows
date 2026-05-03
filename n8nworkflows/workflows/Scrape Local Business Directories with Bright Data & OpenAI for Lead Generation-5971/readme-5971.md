Scrape Local Business Directories with Bright Data & OpenAI for Lead Generation

https://n8nworkflows.xyz/workflows/scrape-local-business-directories-with-bright-data---openai-for-lead-generation-5971


# Scrape Local Business Directories with Bright Data & OpenAI for Lead Generation

### 1. Workflow Overview

This workflow automates the process of scraping local business information from Yelp business profiles using Bright Data’s MCP Client tool combined with AI-powered parsing via OpenAI, to generate leads for partnership outreach. The workflow takes a Yelp business URL as input, extracts structured business data, and automatically sends a tailored partnership proposal email to the business.

The logical flow is divided into three main blocks:

- **1.1 Trigger & Input URL Setup:** Manual initiation with input of a Yelp business profile URL.
- **1.2 AI-Powered Scraping & Data Processing:** An AI agent orchestrates scraping the Yelp page using Bright Data’s MCP tool, then parses and structures the raw data into JSON.
- **1.3 Automated Email Outreach:** Sends a personalized partnership proposal email using the scraped business details.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input URL Setup

**Overview:**  
This block starts the workflow manually and sets the target Yelp business profile URL for scraping.

**Nodes Involved:**  
- 🔘 Trigger: Manual Execution  
- 🔗 Set Yelp Business URL for Dr

**Node Details:**

- **🔘 Trigger: Manual Execution**  
  - *Type:* Manual trigger node; initiates workflow execution on user command.  
  - *Configuration:* No parameters; simply allows manual start.  
  - *Input/Output:* No input; outputs a trigger event to next node.  
  - *Edge cases:* User forgets to trigger workflow; no automatic scheduling.  
  - *Version:* Compatible with all n8n versions supporting manual triggers.

- **🔗 Set Yelp Business URL for Dr**  
  - *Type:* Set node; assigns the Yelp URL string to a variable `URL`.  
  - *Configuration:* Hardcoded URL value set to `"https://www.yelp.com/biz/william-kimbrough-md-washington"`.  
  - *Key Variable:* `URL` string used downstream for scraping.  
  - *Input:* Trigger node output.  
  - *Output:* Passes data with `URL` to next node.  
  - *Edge cases:* URL must be a valid Yelp business page; incorrect URLs may cause scraping failures.

---

#### 1.2 AI-Powered Scraping & Data Processing

**Overview:**  
This block uses an AI agent to scrape the Yelp URL for detailed business data. The agent leverages OpenAI’s GPT-4 model to understand scraping instructions and the Bright Data MCP Client to perform the actual data extraction. The scraped markdown is then parsed into structured JSON.

**Nodes Involved:**  
- 🤖 Agent: Scrape Yelp Business Info  
- 💬 AI Model: Process Data  
- 🌐 Bright Data MCP Client  
- 📝 Parse Scraped Data into JSON1  
- Auto-fixing Output Parser  
- OpenAI Chat Model (used internally in the agent)

**Node Details:**

- **🤖 Agent: Scrape Yelp Business Info**  
  - *Type:* Langchain AI Agent node; orchestrates the scraping and processing sub-workflow.  
  - *Configuration:* Prompt instructs scraping of specified Yelp URL for business info fields: business name, location, phone, category, rating, reviews, website.  
  - *Key expressions:* Uses `{{ $json.URL }}` from Set node.  
  - *Input:* Receives URL from previous node.  
  - *Output:* Structured JSON output of scraped business info.  
  - *Sub-workflow:* Internally invokes `💬 AI Model: Process Data`, `🌐 Bright Data MCP Client`, and parsing nodes.  
  - *Edge cases:* AI misinterpretation of instructions; MCP client failures; website layout changes causing scraping errors; rate limiting or CAPTCHAs from Yelp.

- **💬 AI Model: Process Data**  
  - *Type:* Langchain OpenAI Chat model (gpt-4.1-mini).  
  - *Configuration:* Processes the scraping prompt for the agent.  
  - *Credentials:* Uses OpenAI API credentials.  
  - *Input:* Receives prompt text from agent node.  
  - *Output:* AI-generated instructions or responses to guide scraping.  
  - *Edge cases:* API rate limits, timeouts, or invalid prompt responses.

- **🌐 Bright Data MCP Client**  
  - *Type:* MCP Client Tool node (Bright Data).  
  - *Configuration:* Uses "scrape_as_markdown" tool to scrape Yelp page as markdown text.  
  - *Credentials:* MCP API credentials required.  
  - *Input:* Receives scraping instructions from AI model.  
  - *Output:* Raw markdown data from Yelp page.  
  - *Edge cases:* Network errors, authentication failure with MCP, blocked requests by Yelp.

- **📝 Parse Scraped Data into JSON1**  
  - *Type:* Langchain Output Parser Structured node.  
  - *Configuration:* Uses a JSON schema example to parse markdown into structured JSON fields (business_name, location, contact_phone, category, rating, reviews, website).  
  - *Input:* Raw markdown scraped data.  
  - *Output:* Clean, structured JSON data.  
  - *Edge cases:* Parsing failures if data format changes; incomplete or missing fields.

- **Auto-fixing Output Parser**  
  - *Type:* Langchain Output Parser Autofixing.  
  - *Configuration:* Attempts to automatically fix parsing errors in the output.  
  - *Input:* Output from JSON parser.  
  - *Output:* Corrected JSON data.  
  - *Edge cases:* May not fix all malformed data; fallback required.

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat model (gpt-4.1-mini).  
  - *Role:* Supports parsing and data refinement inside the agent.  
  - *Credentials:* OpenAI API credentials.  
  - *Edge cases:* API limits, model availability.

---

#### 1.3 Automated Email Outreach

**Overview:**  
Using the structured data, this block sends a personalized partnership proposal email to the business email extracted from the Yelp profile.

**Nodes Involved:**  
- 📧 Send Partnership Proposal to Business Email

**Node Details:**

- **📧 Send Partnership Proposal to Business Email**  
  - *Type:* Gmail node for sending emails via OAuth2.  
  - *Configuration:*  
    - Recipient email hardcoded to `shahkar.genai@gmail.com` (note: workflow example uses a fixed email; in practice this should be dynamic from scraped data).  
    - Email subject and body dynamically populated with business name, rating, category, and website from scraped JSON (`{{ $json.output[0].business_name }}`, etc.).  
    - Email type: plain text.  
  - *Credentials:* Gmail OAuth2 credentials configured.  
  - *Input:* Receives business data JSON from agent output.  
  - *Output:* Sends email, outputs success/failure status.  
  - *Edge cases:* Email sending failures due to OAuth token expiry, invalid email addresses, or Gmail sending limits.

---

### 3. Summary Table

| Node Name                                | Node Type                               | Functional Role                          | Input Node(s)                       | Output Node(s)                        | Sticky Note                                                                                                                |
|------------------------------------------|---------------------------------------|----------------------------------------|-----------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| 🔘 Trigger: Manual Execution              | Manual Trigger                        | Start workflow manually                 | —                                 | 🔗 Set Yelp Business URL for Dr     | See Sticky Note: Section 1 - Trigger & Set Yelp Business URL                                                                |
| 🔗 Set Yelp Business URL for Dr           | Set                                  | Define target Yelp business URL         | 🔘 Trigger                        | 🤖 Agent: Scrape Yelp Business Info | See Sticky Note: Section 1 - Trigger & Set Yelp Business URL                                                                |
| 🤖 Agent: Scrape Yelp Business Info       | Langchain AI Agent                   | Orchestrate scraping & processing       | 🔗 Set Yelp Business URL for Dr   | 📧 Send Partnership Proposal to Business Email | See Sticky Note: Section 2 - AI Agent Scrapes Business Information                                                             |
| 💬 AI Model: Process Data                  | Langchain OpenAI Chat Model          | Process scraping prompt                  | — (internal to agent)             | 🤖 Agent: Scrape Yelp Business Info | See Sticky Note: Section 2                                                                                                  |
| 🌐 Bright Data MCP Client                  | MCP Client Tool                      | Scrape Yelp page content as markdown    | — (internal to agent)             | 🤖 Agent: Scrape Yelp Business Info | See Sticky Note: Section 2                                                                                                  |
| 📝 Parse Scraped Data into JSON1           | Langchain Output Parser Structured   | Parse markdown into structured JSON     | — (internal to agent)             | Auto-fixing Output Parser            | See Sticky Note: Section 2                                                                                                  |
| Auto-fixing Output Parser                  | Langchain Output Parser Autofixing   | Auto-correct parsing errors             | 📝 Parse Scraped Data into JSON1  | 🤖 Agent: Scrape Yelp Business Info | See Sticky Note: Section 2                                                                                                  |
| OpenAI Chat Model                          | Langchain OpenAI Chat Model          | Support parsing and data refinement     | — (internal to agent)             | Auto-fixing Output Parser            | See Sticky Note: Section 2                                                                                                  |
| 📧 Send Partnership Proposal to Business Email | Gmail                               | Send personalized partnership email     | 🤖 Agent: Scrape Yelp Business Info | —                                   | See Sticky Note: Section 3 - Send Email Proposal to Business                                                                |
| Sticky Note                               | Sticky Note                         | Informational note for Section 1         | —                                 | —                                   | Section 1 explanation                                                                                                      |
| Sticky Note1                              | Sticky Note                         | Informational note for Section 2         | —                                 | —                                   | Section 2 explanation                                                                                                      |
| Sticky Note2                              | Sticky Note                         | Informational note for Section 3         | —                                 | —                                   | Section 3 explanation                                                                                                      |
| Sticky Note3                              | Sticky Note                         | Full workflow summary and usage notes   | —                                 | —                                   | Full workflow explanation                                                                                                 |
| Sticky Note5                              | Sticky Note                         | Affiliate link for Bright Data           | —                                 | —                                   | Affiliate link to Bright Data                                                                                             |
| Sticky Note9                              | Sticky Note                         | Workflow assistance contact & links     | —                                 | —                                   | Contact info and resource links                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - No parameters needed. This node starts workflow manually.

2. **Add a Set Node to Define Yelp Business URL**  
   - Node Type: Set  
   - Create a string field named `URL` and set the value to the Yelp business profile URL you want to scrape, e.g., `"https://www.yelp.com/biz/william-kimbrough-md-washington"`.  
   - Connect Manual Trigger → Set Node.

3. **Add a Langchain AI Agent Node for Scraping**  
   - Node Type: Langchain Agent (agent node from `@n8n/n8n-nodes-langchain.agent`)  
   - Set prompt text to instruct scraping of the Yelp URL (use the variable `{{ $json.URL }}` in the prompt).  
   - Example prompt snippet:  
     ```
     You are a data extraction agent.
     Scrape the following Yelp URL:
     {{ $json.URL }}
     Extract data for businesses (Doctors) listed on this page. For each business, provide:
     - Business Name
     - Location
     - Contact Phone (if available)
     - Category
     - Rating
     - Reviews
     - Website (if available)
     ```
   - Use OpenAI GPT-4.1-mini model (configure credentials).  
   - This agent internally calls the following nodes (you will add them next) via sub-workflow:
     - Langchain OpenAI Chat model to process the prompt.  
     - Bright Data MCP Client tool to scrape the page as markdown.  
     - Output parsing nodes to convert scraped markdown into structured JSON.

4. **Inside the Agent: Add OpenAI Chat Model Node**  
   - Node Type: Langchain OpenAI Chat Model  
   - Set model to GPT-4.1-mini.  
   - Connect input from agent prompt processing.  
   - Configure OpenAI API credentials.

5. **Add Bright Data MCP Client Node**  
   - Node Type: MCP Client Tool (n8n-nodes-mcp.mcpClientTool)  
   - Set tool name to `"scrape_as_markdown"`.  
   - Configure credentials for Bright Data MCP Client.  
   - Input parameters will be populated by AI prompt processing.  
   - Connect output to parsing nodes.

6. **Add Langchain Output Parser Structured Node**  
   - Node Type: Output Parser Structured  
   - Provide JSON schema example for the expected business info fields (business_name, location, contact_phone, category, rating, reviews, website).  
   - Connect MCP Client output to this node.

7. **Add Langchain Output Parser Autofixing Node**  
   - Node Type: Output Parser Autofixing  
   - Connect output from the structured parser node here.  
   - This node attempts to fix parsing errors automatically.

8. **Connect Autofixing Parser Output back to Agent Output**  
   - The agent node outputs the final structured and cleaned JSON data.

9. **Add Gmail Node to Send Partnership Proposal Email**  
   - Node Type: Gmail (OAuth2)  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient email (in example workflow it is fixed to `shahkar.genai@gmail.com`, but for dynamic use, map this from scraped data).  
   - Configure email subject and body with expressions referencing the structured JSON data fields, e.g.:  
     - Subject: `Potential Partnership with {{ $json.output[0].business_name }}`  
     - Body:  
       ```
       Dear {{ $json.output[0].business_name }},
       
       I hope this message finds you well. I am reaching out to explore a potential partnership with your business, as I noticed your exceptional reputation on Yelp with a rating of {{ $json.output[0].rating }}. We specialize in providing innovative solutions for businesses in the {{ $json.output[0].category }} industry, and I believe there is a strong synergy between our services.
       
       Please let us know if you're open to discussing further details. You can visit our website at {{ $json.output[0].website }}, or feel free to reach out via email at shahkar.genai@gmail.com.
       
       Looking forward to hearing from you soon.
       
       Best regards,
       [Your Name]
       [Your Company]
       ```
   - Connect Agent output → Gmail node.

10. **Connect the main workflow nodes:**  
    - Manual Trigger → Set Yelp URL → AI Agent → Gmail Send Email

11. **Test the workflow manually by clicking Execute** and verify:  
    - URL is correctly set.  
    - Data is scraped and parsed into JSON.  
    - Email is sent with populated fields.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Affiliate link for Bright Data to support free content creation                                                                     | https://get.brightdata.com/1tndi4600b25                                                        |
| Workflow assistance and support contact: Yaron@nofluff.online                                                                      | Contact email for questions or support                                                        |
| Additional learning resources and tutorials from Yaron Been                                                                         | YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| This workflow requires valid credentials for OpenAI API (GPT-4.1-mini model) and Bright Data MCP Client with scraping tool enabled | Credentials setup required in n8n’s credentials manager                                        |

---

**Disclaimer:** The provided text is exclusively sourced from an automated n8n workflow. It strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.