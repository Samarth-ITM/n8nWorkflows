Extract Top Products from Any Website with Dumpling AI and GPT-4o

https://n8nworkflows.xyz/workflows/extract-top-products-from-any-website-with-dumpling-ai-and-gpt-4o-8565


# Extract Top Products from Any Website with Dumpling AI and GPT-4o

### 1. Workflow Overview

This workflow automates the extraction and analysis of top product picks from any website, using a combination of web crawling, screenshot analysis, and AI-driven data extraction to identify the best-value products. It is designed for e-commerce market research, product comparison, and automated reporting, notably on product listings such as those on Amazon.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the target website URL via a user form.
- **1.2 Website Crawling:** Leverages Dumpling AI to crawl the website and collect product page data.
- **1.3 Screenshot Capture:** Takes screenshots of crawled pages via Dumpling AI.
- **1.4 AI Analysis:** Uses GPT-4o to analyze screenshots and extract product details focusing on best-value picks.
- **1.5 Data Parsing:** Cleans and parses the AI's JSON output into structured data.
- **1.6 Data Storage:** Saves extracted product data into a Google Sheet.
- **1.7 Notification:** Sends an email containing the link to the Google Sheet with analysis results.
- **1.8 Documentation:** Provides an embedded sticky note with project description and instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures user input (website URL) via a web form trigger.
- **Nodes Involved:** Receive Website URL
- **Node Details:**

  - **Node Name:** Receive Website URL
  - **Type:** `Form Trigger`
  - **Function:** Starts workflow on HTTP form submission.
  - **Configuration:**
    - Form titled "Website" with one input field: "Website URL"
    - Webhook configured for external form submission.
  - **Expressions:** Uses the submitted field `Website URL` to pass downstream.
  - **Connections:** Triggers `Crawl Website with Dumpling AI`
  - **Version:** 2
  - **Error Cases:** Invalid or missing URL input; network issues with webhook.

#### 1.2 Website Crawling

- **Overview:** Calls Dumpling AI API to crawl the website's pages and extract content.
- **Nodes Involved:** Crawl Website with Dumpling AI, Split Crawled Pages
- **Node Details:**

  - **Node Name:** Crawl Website with Dumpling AI
  - **Type:** `HTTP Request`
  - **Function:** POST request to Dumpling AI's crawl API endpoint.
  - **Configuration:**
    - URL: `https://app.dumplingai.com/api/v1/crawl`
    - Method: POST
    - Body: JSON with dynamic URL from input, limit set to 2 pages.
    - Auth: Custom HTTP header credential (Dumpling AI token)
  - **Expressions:** URL dynamically sets from `Website URL` form field.
  - **Connections:** Sends output to `Split Crawled Pages`
  - **Version:** 4
  - **Failure Points:** API auth failures; API rate limits; invalid website URL; empty or error in response.

  - **Node Name:** Split Crawled Pages
  - **Type:** `SplitOut`
  - **Function:** Splits array of crawled page results into individual items.
  - **Configuration:** Splits on `results` array field.
  - **Connections:** Each page item passed to `Take Screenshot of Page`
  - **Version:** 1
  - **Failures:** Empty `results` array; malformed data.

#### 1.3 Screenshot Capture

- **Overview:** For each crawled page, captures a screenshot via Dumpling AI.
- **Nodes Involved:** Take Screenshot of Page
- **Node Details:**

  - **Node Name:** Take Screenshot of Page
  - **Type:** `HTTP Request`
  - **Function:** POST request to Dumpling AI screenshot API.
  - **Configuration:**
    - URL: `https://app.dumplingai.com/api/v1/screenshot`
    - Method: POST
    - Body: JSON with dynamic URL from split page item, capturing viewport only (`fullPage=false`).
    - Auth: Same Dumpling AI credential.
  - **Expressions:** URL dynamically set from each page's `url`.
  - **Connections:** Passes screenshot URL to `Analyze Screenshot with GPT-4o`
  - **Version:** 4
  - **Failures:** API errors, invalid page URLs, network timeouts.

#### 1.4 AI Analysis

- **Overview:** Uses GPT-4o to analyze the screenshots and extract product data (top 3 best-value picks).
- **Nodes Involved:** Analyze Screenshot with GPT-4o
- **Node Details:**

  - **Node Name:** Analyze Screenshot with GPT-4o
  - **Type:** `LangChain OpenAI`
  - **Function:** Analyzes images with AI prompt to extract product information.
  - **Configuration:**
    - Model: `chatgpt-4o-latest`
    - Input: Screenshot URL from previous node.
    - Prompt: Detailed instructions to identify top 3 products with fields: `name`, `price`, `reviews`, `free_delivery_date`.
    - Context: Current date dynamically inserted.
  - **Expressions:** Uses workflow execution time for date.
  - **Connections:** Outputs markdown JSON string to `Parse and Extract Product Data`
  - **Version:** 1
  - **Credentials:** OpenAI API key.
  - **Failures:** API quota exceeded; malformed AI response; timeout; image inaccessible.

#### 1.5 Data Parsing

- **Overview:** Parses AI-generated JSON product arrays into structured JSON objects.
- **Nodes Involved:** Parse and Extract Product Data
- **Node Details:**

  - **Node Name:** Parse and Extract Product Data
  - **Type:** `Code`
  - **Function:** Cleans markdown JSON (removes ```json wrappers), parses JSON, and outputs each product as an item.
  - **Configuration:** JavaScript code handling multiple input items.
  - **Expressions:** Uses standard JS parsing.
  - **Connections:** Passes parsed product objects to `Save Products to Google Sheet`
  - **Version:** 2
  - **Failures:** JSON parse errors; missing or invalid fields; malformed AI output.

#### 1.6 Data Storage

- **Overview:** Appends extracted products as rows into a predefined Google Sheet.
- **Nodes Involved:** Save Products to Google Sheet
- **Node Details:**

  - **Node Name:** Save Products to Google Sheet
  - **Type:** `Google Sheets`
  - **Function:** Append rows to sheet.
  - **Configuration:**
    - Document ID: (predefined)
    - Sheet name: "gid=0" (default sheet)
    - Columns mapped: `product name`, `price`, `reviews no.`, `free_delivery_date` from parsed JSON fields.
    - Append operation.
  - **Connections:** Triggers `Send Email with Product Link`
  - **Version:** 4
  - **Credentials:** Google Sheets OAuth2.
  - **Failures:** Credential expiry; insufficient permissions; sheet not found; rate limits.

#### 1.7 Notification

- **Overview:** Sends an email with a link to the Google Sheet containing product picks.
- **Nodes Involved:** Send Email with Product Link
- **Node Details:**

  - **Node Name:** Send Email with Product Link
  - **Type:** `Gmail`
  - **Function:** Sends notification email.
  - **Configuration:**
    - To: `example@gmail.com` (hardcoded, customizable)
    - Subject: "Your Product Comparison Report is Ready – Check the Results"
    - Body: HTML content linking to the Google Sheet.
    - Append attribution disabled.
  - **Connections:** End of workflow.
  - **Version:** 2
  - **Credentials:** Gmail OAuth2.
  - **Failures:** Auth failure; email quota; invalid recipient.

#### 1.8 Documentation

- **Overview:** Contains workflow description, instructions, and requirements.
- **Nodes Involved:** Sticky Note
- **Node Details:**

  - **Node Name:** Sticky Note
  - **Type:** `Sticky Note`
  - **Content:** Overview of workflow steps, requirements (Dumpling AI token, OpenAI key, Google Sheet setup), and customization hints.
  - **Connections:** None.
  - **Version:** 1
  - **Failures:** None.

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                           | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                 |
|-----------------------------|----------------------|-----------------------------------------|----------------------------|-----------------------------|--------------------------------------------------------------------------------------------|
| Receive Website URL          | Form Trigger         | Receives website URL input               | -                          | Crawl Website with Dumpling AI | See block 1.1                                                                               |
| Crawl Website with Dumpling AI | HTTP Request         | Crawls website pages via Dumpling API   | Receive Website URL         | Split Crawled Pages          | See block 1.2                                                                               |
| Split Crawled Pages          | SplitOut             | Splits crawled results into pages       | Crawl Website with Dumpling AI | Take Screenshot of Page      | See block 1.2                                                                               |
| Take Screenshot of Page      | HTTP Request         | Captures screenshots of each page       | Split Crawled Pages         | Analyze Screenshot with GPT-4o | See block 1.3                                                                               |
| Analyze Screenshot with GPT-4o | LangChain OpenAI    | AI analyzes images to extract products   | Take Screenshot of Page     | Parse and Extract Product Data | See block 1.4                                                                               |
| Parse and Extract Product Data | Code                | Parses AI JSON output into structured data | Analyze Screenshot with GPT-4o | Save Products to Google Sheet | See block 1.5                                                                               |
| Save Products to Google Sheet | Google Sheets        | Saves parsed product data to spreadsheet | Parse and Extract Product Data | Send Email with Product Link  | See block 1.6                                                                               |
| Send Email with Product Link | Gmail                | Sends email notification                 | Save Products to Google Sheet | -                           | See block 1.7                                                                               |
| Sticky Note                  | Sticky Note          | Documentation and instructions          | -                          | -                           | Contains full workflow description and instructions (see block 1.8)                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("Receive Website URL"):**
   - Set form title to "Website."
   - Add a single text input field labeled "Website URL."
   - Enable webhook to receive HTTP POST requests.

2. **Add HTTP Request Node ("Crawl Website with Dumpling AI"):**
   - Method: POST
   - URL: `https://app.dumplingai.com/api/v1/crawl`
   - Body Type: JSON
   - Body Content:
     ```json
     {
       "url": "={{$json['Website URL']}}",
       "limit": "2"
     }
     ```
   - Authentication: Use HTTP header auth with Dumpling AI API token credential.
   - Connect output of "Receive Website URL" to this node.

3. **Add SplitOut Node ("Split Crawled Pages"):**
   - Configure to split on the field `results`.
   - Connect output of "Crawl Website with Dumpling AI."

4. **Add HTTP Request Node ("Take Screenshot of Page"):**
   - Method: POST
   - URL: `https://app.dumplingai.com/api/v1/screenshot`
   - Body Type: JSON
   - Body Content:
     ```json
     {
       "url": "={{$json['url']}}",
       "fullPage": false
     }
     ```
   - Authentication: Same Dumpling AI credential.
   - Connect output of "Split Crawled Pages."

5. **Add LangChain OpenAI Node ("Analyze Screenshot with GPT-4o"):**
   - Resource: Image
   - Operation: Analyze
   - Model: `chatgpt-4o-latest`
   - Input image URL: `={{$json['screenshotUrl']}}`
   - Prompt:
     ```
     You are a shopping assistant that analyzes screenshots of website listings.
     Extract top 3 products with name, price, reviews, free_delivery_date.
     Prioritize free delivery, earliest delivery date, lowest price.
     Use current date: {{$now.format('MMMM D, YYYY')}}.
     Return JSON array with fields: name, price, reviews, free_delivery_date.
     ```
   - Credential: OpenAI API key.
   - Connect output of "Take Screenshot of Page."

6. **Add Code Node ("Parse and Extract Product Data"):**
   - Paste JavaScript code to clean markdown JSON and parse product arrays.
   - The code iterates over input items, removes ````json``` wrappers, parses JSON, and outputs product objects.
   - Connect output of "Analyze Screenshot with GPT-4o."

7. **Add Google Sheets Node ("Save Products to Google Sheet"):**
   - Operation: Append
   - Document ID: Provide Google Sheets document ID.
   - Sheet Name: `gid=0` (or actual sheet name)
   - Map columns to product fields:
     - `product name`: `={{$json["name"]}}`
     - `price`: `={{$json["price"]}}`
     - `reviews no.`: `={{$json["reviews"]}}`
     - `free_delivery_date`: `={{$json["free_delivery_date"]}}`
   - Credential: Google Sheets OAuth2.
   - Connect output of "Parse and Extract Product Data."

8. **Add Gmail Node ("Send Email with Product Link"):**
   - Recipient: Customize email (default `example@gmail.com`).
   - Subject: "Your Product Comparison Report is Ready – Check the Results"
   - Content Type: HTML
   - Message Body: Contains link to Google Sheet (use actual sheet URL).
   - Disable adding attribution.
   - Credential: Gmail OAuth2.
   - Set to execute once per workflow run.
   - Connect output of "Save Products to Google Sheet."

9. **Add Sticky Note Node (Optional):**
   - Add workflow description, requirements, and notes for user reference.

10. **Connect all nodes sequentially as above.**

11. **Configure Credentials:**
   - Dumpling AI token: HTTP header credential.
   - OpenAI API key credential.
   - Google Sheets OAuth2 credential.
   - Gmail OAuth2 credential.

12. **Activate and Test Workflow:**
   - Use test URL input via form.
   - Monitor logs for errors.
   - Validate Google Sheet data and email receipt.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow "Extract Top Products from Any Website with Dumpling AI and GPT-4o" analyzes product listings via screenshot AI analysis. | Automation for e-commerce product research; adaptable to other image content by changing AI prompt.              |
| Requires API tokens for Dumpling AI, OpenAI, Google Sheets, and Gmail OAuth2 credentials.            | Credential setup in n8n for respective APIs.                                                                     |
| Google Sheet must have columns: `product name`, `price`, `reviews no.`, `free_delivery_date`.         | Sheet URL is embedded in email notification node for easy access.                                                |
| Product extraction prompt prioritizes free delivery, earliest delivery date, and affordable price.   | Prompt is customizable in the GPT-4o node's input field.                                                         |
| Dumpling AI API limits and rate limits may affect crawling and screenshot capture reliability.      | Monitor API usage and errors; consider retries or fallbacks in production.                                       |
| AI analysis may fail if screenshots are inaccessible or malformed.                                   | Validate screenshot URLs and handle AI API exceptions gracefully.                                                |
| Email recipient is hardcoded; for production, consider dynamic input or environment variable usage. | Email content can be customized to include more data or alternative notifications (e.g., Slack).                 |
| Extensive product data extracted from Amazon via Dumpling AI crawling and AI analysis.               | Potentially sensitive to website layout changes; maintain and update prompt and crawling parameters as needed.   |

---

**Disclaimer:**  
This document is generated to describe the given n8n workflow for product extraction from websites using Dumpling AI and GPT-4o. The workflow complies with content policies and processes only publicly available data. No illegal or sensitive data is handled.