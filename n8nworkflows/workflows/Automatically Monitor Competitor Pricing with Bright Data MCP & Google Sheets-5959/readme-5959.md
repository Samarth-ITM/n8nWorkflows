Automatically Monitor Competitor Pricing with Bright Data MCP & Google Sheets

https://n8nworkflows.xyz/workflows/automatically-monitor-competitor-pricing-with-bright-data-mcp---google-sheets-5959


# Automatically Monitor Competitor Pricing with Bright Data MCP & Google Sheets

### 1. Workflow Overview

This workflow automates the monitoring of competitor product pricing by scraping product data from dynamic websites (e.g., Nike) and logging it into Google Sheets on a scheduled basis. It integrates Bright Data’s MCP (Mobile Carrier Proxy) for robust web scraping, leverages OpenAI models for intelligent scraping and output parsing, and notifies stakeholders via email when new pricing data is logged.

The workflow is logically divided into four main blocks:

- **1.1 Schedule & Input Configuration:** Automates workflow triggering on a weekly schedule and provides manual URL input flexibility.
- **1.2 Competitor Price Scraping via AI Agent:** Uses an AI-powered agent integrated with Bright Data MCP and OpenAI models to scrape and parse competitor pricing data.
- **1.3 Email Notification:** Sends an email alert when new pricing data is successfully stored.
- **1.4 Data Transformation & Storage:** Reformats scraped data into structured rows and appends them to a Google Sheets document for tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Input Configuration

**Overview:**  
This block initiates the workflow on a weekly schedule and allows setting or overriding the product page URL to track.

**Nodes Involved:**  
- 📅 Run weekly Check (Schedule Trigger)  
- 🛒 Product Page URL (Set)

**Node Details:**

- **📅 Run weekly Check**  
  - Type: Schedule Trigger  
  - Role: Starts workflow every Monday at 9 AM weekly.  
  - Configuration: Weekly interval, triggers at Monday 9:00 AM.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to the URL setter node.  
  - Edge Cases: Missed triggers if instance offline; timezone considerations.  

- **🛒 Product Page URL**  
  - Type: Set  
  - Role: Defines the product URL(s) for scraping; can be edited to override URL.  
  - Configuration: Single string field “url” with a default Nike category URL.  
  - Inputs: From schedule trigger.  
  - Outputs: Connects to the scraping agent node.  
  - Edge Cases: Invalid or unreachable URLs; manual override needed for different targets.  

---

#### 1.2 Competitor Price Scraping via AI Agent

**Overview:**  
This core block uses an AI agent with Bright Data MCP to scrape competitor product details from dynamic web pages, using OpenAI to guide scraping and parse unstructured data into structured JSON.

**Nodes Involved:**  
- 🤖 Scrape Competitor Prices (MCP Agent) (AI Agent)  
- MCP Client (Bright Data MCP API Client)  
- OpenAI Chat Model (Language Model for scraping logic)  
- Auto-fixing Output Parser (Output parser with error correction)  
- Structured Output Parser1 (Final structured JSON parser)  
- OpenAI Chat Model1 (Supporting language model for parsing)

**Node Details:**

- **🤖 Scrape Competitor Prices (MCP Agent)**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates scraping using AI language model and MCP client; generates scraping instructions and collects raw data.  
  - Configuration: Prompt includes dynamic URL injection (`{{ $json.url }}`) with instructions to scrape all product details.  
  - Inputs: URL from previous node.  
  - Outputs: Raw scraped data forwarded to formatting code and email notification nodes.  
  - Edge Cases: Scraping failure due to anti-bot tech changes, network issues, or data format changes.  
  - Uses connected MCP Client, OpenAI Chat Model, and Output Parsers.

- **MCP Client**  
  - Type: MCP Client Tool  
  - Role: Executes scraping commands on Bright Data MCP proxies to bypass bot detection.  
  - Configuration: Executes “scrape_as_markdown” tool with parameters dynamically set via expression placeholder.  
  - Inputs: From AI Agent.  
  - Outputs: Returns scraped markdown data to Agent.  
  - Edge Cases: MCP service downtime, API quota limits, authorization issues.

- **OpenAI Chat Model**  
  - Type: Language Model (OpenAI GPT-4o-mini)  
  - Role: Provides AI reasoning for scraping steps, parsing, and instructions for MCP Client.  
  - Configuration: GPT-4o-mini selected for balance of power and speed.  
  - Inputs: From AI Agent.  
  - Outputs: Feeding back into AI Agent for generating scraping plans.  
  - Edge Cases: API limits, latency, or unexpected output format.

- **Auto-fixing Output Parser**  
  - Type: LangChain output parser with auto-correction  
  - Role: Parses raw scraped data and attempts to fix parsing errors automatically for more robust JSON output.  
  - Configuration: Default options.  
  - Inputs: From OpenAI Chat Model1 via structured output parser.  
  - Outputs: Clean JSON product list to AI Agent.  
  - Edge Cases: Parsing failures if data is highly irregular or incomplete.

- **Structured Output Parser1**  
  - Type: LangChain structured output parser  
  - Role: Parses JSON-formatted product data according to example schema (name, description, price, URL).  
  - Configuration: Example JSON schema provided with product sample entries to guide parsing.  
  - Inputs: From OpenAI Chat Model1.  
  - Outputs: To Auto-fixing Output Parser.  
  - Edge Cases: Schema mismatches, malformed JSON input.

- **OpenAI Chat Model1**  
  - Type: Language Model (OpenAI GPT-4o-mini)  
  - Role: Supports output parsing step with advanced reasoning.  
  - Configuration: Same model as main OpenAI Chat Model.  
  - Inputs: From AI Agent.  
  - Outputs: To structured output parser.  
  - Edge Cases: Same as other OpenAI nodes.  

---

#### 1.3 Email Notification

**Overview:**  
Notifies the user via Gmail when new competitor pricing data has been successfully logged into Google Sheets.

**Nodes Involved:**  
- 📧 Notify: Prices Logged in Sheet (Gmail)

**Node Details:**

- **📧 Notify: Prices Logged in Sheet**  
  - Type: Gmail Node (OAuth2)  
  - Role: Sends a notification email confirming data logging completion.  
  - Configuration:  
    - Recipient: shahkar.genai@gmail.com  
    - Subject: “Competitor product pricing has just logged into google sheets”  
    - Message body: Simple text informing recipient to review updated prices.  
  - Inputs: From AI Agent after data scraping completes.  
  - Outputs: None (end node).  
  - Edge Cases: OAuth token expiry, Gmail API quota exceeded, email delivery failures.

---

#### 1.4 Data Transformation & Storage

**Overview:**  
Transforms scraped product data into individual rows and appends them into a Google Sheets document for logging and tracking.

**Nodes Involved:**  
- 🧹 Format Products for Google Sheets (Code)  
- 📄 Save to Google Sheets (Competitor Pricing Log)

**Node Details:**

- **🧹 Format Products for Google Sheets**  
  - Type: Code (JavaScript)  
  - Role: Converts the array of product objects into individual items, each representing a single product row for Google Sheets.  
  - Configuration:  
    - JS code extracts `name`, `description`, `price`, and `url` fields from each product in the input array.  
  - Inputs: JSON array of products from AI Agent.  
  - Outputs: One output item per product for Google Sheets node.  
  - Edge Cases: Missing product attributes, empty product list.

- **📄 Save to Google Sheets (Competitor Pricing Log)**  
  - Type: Google Sheets Node (OAuth2)  
  - Role: Appends each product as a new row in the specified Google Sheets document and sheet.  
  - Configuration:  
    - Document ID: `11NLHG0clJ78pV6YWc_Nt3Q8pjzrojiuwDOuvASViOI4` (Competitor product pricing)  
    - Sheet: gid=0 (Sheet1)  
    - Columns mapped explicitly: URL, Name, Price, Description  
    - Operation: Append (adds new rows)  
  - Inputs: From code node formatting product items.  
  - Outputs: None (terminal node).  
  - Edge Cases: Google Sheets API quota limits, permission errors, sheet structure changes.

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                                  | Input Node(s)                      | Output Node(s)                                         | Sticky Note                                                                                              |
|-----------------------------------|----------------------------------|-------------------------------------------------|----------------------------------|-------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| 📅 Run weekly Check                | Schedule Trigger                 | Triggers workflow weekly                         | None                             | 🛒 Product Page URL                                   | Starts the workflow on a weekly schedule, triggers on Monday 9AM                                       |
| 🛒 Product Page URL                | Set                             | Defines or overrides product URL                 | 📅 Run weekly Check              | 🤖 Scrape Competitor Prices (MCP Agent)               | Allows manual URL override for flexible testing                                                        |
| 🤖 Scrape Competitor Prices (MCP Agent) | LangChain Agent Node            | Core scraping orchestrator using AI & MCP       | 🛒 Product Page URL, MCP Client, OpenAI Chat Model, Auto-fixing Output Parser | 🧹 Format Products for Google Sheets, 📧 Notify: Prices Logged in Sheet | Uses AI agent and Bright Data MCP to scrape competitor pricing data                                   |
| MCP Client                       | MCP Client Tool                 | Executes scraping on Bright Data MCP             | 🤖 Scrape Competitor Prices (MCP Agent) | 🤖 Scrape Competitor Prices (MCP Agent)               | Handles anti-bot scraping via Bright Data MCP proxies                                                 |
| OpenAI Chat Model                | Language Model (OpenAI GPT-4o-mini) | Provides AI reasoning for scraping               | 🤖 Scrape Competitor Prices (MCP Agent) | 🤖 Scrape Competitor Prices (MCP Agent)               | Supports intelligent scraping strategies                                                             |
| Auto-fixing Output Parser        | LangChain Output Parser (Auto-fixing) | Parses and auto-corrects scraped data output     | Structured Output Parser1        | 🤖 Scrape Competitor Prices (MCP Agent)               | Automatically fixes parsing errors for robust JSON output                                             |
| Structured Output Parser1        | LangChain Structured Output Parser | Parses structured JSON product data              | OpenAI Chat Model1               | Auto-fixing Output Parser                              | Uses example JSON schema to parse product data                                                       |
| OpenAI Chat Model1               | Language Model (OpenAI GPT-4o-mini) | Supports output parsing                           | 🤖 Scrape Competitor Prices (MCP Agent) | Structured Output Parser1                              | Assists in producing correctly formatted JSON output                                                |
| 🧹 Format Products for Google Sheets | Code (JavaScript)              | Transforms product list into individual rows     | 🤖 Scrape Competitor Prices (MCP Agent) | 📄 Save to Google Sheets (Competitor Pricing Log)     | Breaks array of products into single entries for Google Sheets                                       |
| 📄 Save to Google Sheets (Competitor Pricing Log) | Google Sheets Node (OAuth2)     | Appends product data rows to Google Sheets       | 🧹 Format Products for Google Sheets | None                                                  | Logs scraped competitor pricing data for tracking                                                    |
| 📧 Notify: Prices Logged in Sheet | Gmail Node (OAuth2)             | Sends email notification on data logging         | 🤖 Scrape Competitor Prices (MCP Agent) | None                                                  | Notifies user when competitor pricing data is logged                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node: "📅 Run weekly Check"**  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Mondays at 9:00 AM.  
   - No input connections.  

2. **Create Set Node: "🛒 Product Page URL"**  
   - Type: Set  
   - Add a string field named `url` with default value: `https://www.nike.com/w/training-gym-clothing-58jtoz6ymx6`  
   - Connect output of schedule trigger to input of this node.

3. **Create LangChain Agent Node: "🤖 Scrape Competitor Prices (MCP Agent)"**  
   - Type: LangChain Agent  
   - Prompt: `"scrape all the details about every product from the following url:\n{{ $json.url }}"`  
   - Set prompt type to "define" with output parser enabled.  
   - Connect output of `🛒 Product Page URL` to this node’s main input.

4. **Create MCP Client Node: "MCP Client"**  
   - Type: MCP Client Tool  
   - Tool Name: `scrape_as_markdown`  
   - Operation: `executeTool`  
   - Tool Parameters: leave as dynamic expression placeholder for AI override.  
   - Connect this node as `ai_tool` input to `🤖 Scrape Competitor Prices (MCP Agent)`.

5. **Create OpenAI Chat Model Node: "OpenAI Chat Model"**  
   - Type: LangChain OpenAI Chat  
   - Model: `gpt-4o-mini`  
   - No additional options.  
   - Connect as `ai_languageModel` input to `🤖 Scrape Competitor Prices (MCP Agent)`.

6. **Create OpenAI Chat Model Node: "OpenAI Chat Model1"**  
   - Same configuration as above.  
   - Connect as `ai_languageModel` input to `Structured Output Parser1`.

7. **Create Structured Output Parser Node: "Structured Output Parser1"**  
   - Type: LangChain Structured Output Parser  
   - Paste example JSON schema with product sample entries (name, description, price, url).  
   - Connect output of `OpenAI Chat Model1` to this node.  
   - Connect output of this node to `Auto-fixing Output Parser`.

8. **Create Auto-fixing Output Parser Node: "Auto-fixing Output Parser"**  
   - Type: LangChain Output Parser Auto-fixing  
   - Default options.  
   - Connect output to `🤖 Scrape Competitor Prices (MCP Agent)` as `ai_outputParser`.

9. **Create Code Node: "🧹 Format Products for Google Sheets"**  
   - Type: Code (JavaScript)  
   - Paste the following JS code:
     ```javascript
     const products = items[0].json.output;
     return products.map(product => ({
       json: {
         name: product.name,
         description: product.description,
         price: product.price,
         url: product.url,
       },
     }));
     ```
   - Connect main output of `🤖 Scrape Competitor Prices (MCP Agent)` to this node.

10. **Create Google Sheets Node: "📄 Save to Google Sheets (Competitor Pricing Log)"**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: `11NLHG0clJ78pV6YWc_Nt3Q8pjzrojiuwDOuvASViOI4`  
    - Sheet Name: `gid=0` (Sheet1)  
    - Columns mapping:  
      - URL → `={{ $json.url }}`  
      - Name → `={{ $json.name }}`  
      - Price → `={{ $json.price }}`  
      - Description → `={{ $json.description }}`  
    - Connect output of the code node to this node.

11. **Create Gmail Node: "📧 Notify: Prices Logged in Sheet"**  
    - Type: Gmail (OAuth2)  
    - Recipient: `shahkar.genai@gmail.com`  
    - Subject: `"Competitor product pricing has just logged into google sheets"`  
    - Message (plain text): `"The Competitor pricing has just logged into google sheets.\n\nPlease take a look and adjust the prices"`  
    - Connect main output of `🤖 Scrape Competitor Prices (MCP Agent)` also to this node so email fires after scraping completes.

12. **Credentials Setup:**  
    - MCP Client: Configure with Bright Data MCP API credentials with proper access.  
    - OpenAI Chat Models: Configure OpenAI API credentials with GPT-4o-mini access.  
    - Google Sheets: OAuth2 credentials with write access to the specified sheet.  
    - Gmail: OAuth2 credentials for sending emails on behalf of the user.

13. **Test the workflow:**  
    - Run manually or wait for the schedule trigger.  
    - Verify logging in Google Sheets and email notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow regularly scrapes competitor pricing data from dynamic websites like Nike using Bright Data MCP proxies.   | Workflow description and operational purpose                                                        |
| Bright Data MCP helps bypass anti-bot detection for JavaScript-heavy sites, enabling reliable scraping.             | https://brightdata.com/products/mobile-carrier-proxy                                                |
| OpenAI GPT-4o-mini model is used for intelligent scraping instruction generation and parsing.                       | OpenAI documentation                                                                                 |
| Email notification node keeps users informed when new competitor data is logged, avoiding manual checking.           | Gmail API / OAuth2 setup                                                                             |
| Google Sheets is used as a live price tracker with appending rows and easy data visualization (charts, filters).    | Google Sheets API documentation                                                                     |
| For support or questions, contact Yaron (workflow author) or visit his YouTube and LinkedIn for more tips:          | Email: Yaron@nofluff.online, YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Affiliate link supporting Bright Data usage: https://get.brightdata.com/1tndi4600b25                                  | Affiliate or referral link                                                                           |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated n8n workflow. All content is legal, public, and compliant with applicable policies. No illegal or protected data is included.