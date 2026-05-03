Multi-Platform Price Finder: Scraping Prices with Bright Data, Claude AI & Telegram

https://n8nworkflows.xyz/workflows/multi-platform-price-finder--scraping-prices-with-bright-data--claude-ai---telegram-6408


# Multi-Platform Price Finder: Scraping Prices with Bright Data, Claude AI & Telegram

### 1. Workflow Overview

This workflow is designed as a **Multi-Platform Price Finder** that scrapes product pricing data from various e-commerce platforms using **Bright Data APIs**, processes the aggregated product data using **Anthropic Claude AI**, and delivers promotional messages to users via **Telegram**. It targets use cases where users want to input search keywords and receive curated, lowest-price product offers across multiple major online retail platforms.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts user search keywords through a web form.
- **1.2 Multi-Platform Scraping Trigger:** For each platform (Amazon, eBay, Etsy, BestBuy, Target, Home Depot, Wayfair, Lowe’s), triggers Bright Data scraping jobs using the user keywords.
- **1.3 Scrape Status Polling & Data Retrieval:** For each platform, periodically checks the scrape job status until data is ready, then fetches the snapshot of scraped product data.
- **1.4 Data Filtering & Storage:** Filters product results relevant to the search keywords and appends/updates Google Sheets with product URL, title, and price.
- **1.5 Data Merging & Lowest Price Identification:** Merges all platform results, then finds the product with the lowest price using custom code.
- **1.6 AI Promotional Message Generation:** Uses Anthropic Claude AI to generate a short promotional message for the identified lowest-price product.
- **1.7 Message Cleaning & Delivery:** Cleans AI output for formatting issues and sends the final promotional message to a Telegram channel or contact.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures user input keywords via a web form to initiate the scraping process.
- **Nodes Involved:**  
  - `On form submission`
- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger node  
    - Configured with a single form field labeled "SearchHere"  
    - Triggers workflow on form submission, passing keyword input downstream  
    - Outputs JSON containing `{ SearchHere: <user keyword> }`  
    - Edge cases: No user input or malformed submission may cause empty downstream queries.

#### 2.2 Multi-Platform Scraping Trigger

- **Overview:** Initiates scraping jobs on Bright Data for each e-commerce platform using the provided keyword.
- **Nodes Involved:**  
  - `Triggers Bright Data scraping for Amazon using keyword`  
  - `Triggers Bright Data scraping for eBay`  
  - `Triggers Bright Data scraping for Etsy`  
  - `Triggers Bright Data scraping for BestBuy`  
  - `Triggers Bright Data scraping for Target`  
  - `Triggers Bright Data scraping for Home Depot`  
  - `wayfair`  
  - `lowes`
- **Node Details:**  
  Each node:  
  - Type: HTTP Request (POST)  
  - Sends JSON body with keyword(s) and custom output fields specific to the platform's product data schema  
  - Uses Bright Data API endpoint `/datasets/v3/trigger` with parameters including dataset ID, type `discover_new`, discover by `keyword`, and limit per input 4  
  - Header includes `Authorization: Bearer BRIGHT_DATA_API_KEY` credential  
  - Outputs include a `snapshot_id` for each triggered scrape job  
  - Edge Cases: API key invalid, Bright Data service downtime, malformed keyword input, HTTP timeout, or API errors.

#### 2.3 Scrape Status Polling & Data Retrieval

- **Overview:** Checks the completion status of each scrape job, waits if not ready, and fetches the final snapshot data once ready.
- **Nodes Involved:**  
  - Multiple `Wait` nodes (Wait, Wait1, Wait2, Wait3, Wait4, Wait5, Wait6, Wait7) — each corresponds to a platform  
  - Multiple "Checks scrape status for ..." HTTP Request nodes (e.g., `Checks scrape status for Amazon (status: ready)`)  
  - Multiple "Fetches final scraped data from ... snapshot" HTTP Request nodes (e.g., `Fetches final scraped data from Amazon snapshot`)  
  - Multiple `If` nodes to verify scrape status equals "ready" before proceeding
- **Node Details:**  
  - Wait nodes pause the workflow for 1 minute intervals before rechecking status  
  - Status check nodes query `/datasets/v3/progress/{snapshot_id}` API endpoint  
  - If nodes evaluate if `status == "ready"` to proceed  
  - Snapshot fetch nodes get data from `/datasets/v3/snapshot/{snapshot_id}` with `format=json`  
  - Edge cases: Long scrape times leading to multiple waits, possible status check failures, snapshot retrieval errors, and API limits.

#### 2.4 Data Filtering & Storage

- **Overview:** Filters the scraped product data for relevance based on the original search keyword and appends or updates Google Sheets with structured product info.
- **Nodes Involved:**  
  - Multiple Filter nodes (`Filter1`, `Filter`, `Filter2`, `Filter3`, `Filter4`, `Filter5`, `Filter6`, `Filter8`)  
  - Multiple Google Sheets nodes (`Google Sheets1` through `Google Sheets7`, plus `Google Sheets`)  
  - Multiple If nodes to check item count > 0 before writing
- **Node Details:**  
  - Filter nodes check if product title contains the search keyword (case-insensitive, normalized)  
  - Google Sheets nodes append or appendOrUpdate product data with columns: URL, Title, price  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Google Sheets API rate limits, empty filtered results, data format mismatches.

#### 2.5 Data Merging & Lowest Price Identification

- **Overview:** Combines all filtered platform results into one stream and identifies the cheapest product among them.
- **Nodes Involved:**  
  - `Merge` node  
  - `Code1` node (custom JavaScript code)  
- **Node Details:**  
  - Merge node configured to accept 8 inputs (one per platform) and combine them into a single array  
  - Code1 node:  
    - Validates input data structure  
    - Extracts title, URL, and price (with flexible field names)  
    - Logs debug info if no valid products found  
    - Finds product with the lowest price  
    - Returns structured result with title, price, URL, and a formatted message  
  - Edge cases: No valid products, inconsistent data fields, price parsing errors.

#### 2.6 AI Promotional Message Generation

- **Overview:** Uses Anthropic Claude AI model to create an engaging promotional message for the lowest price product.
- **Nodes Involved:**  
  - `AI Agent` (LangChain Agent node)  
  - `Anthropic Chat Model` (Anthropic Claude AI node)  
- **Node Details:**  
  - AI Agent node constructs prompt dynamically with product details (title, price, URL)  
  - Uses Claude Sonnet 4 model via Anthropic API credentials  
  - Prompt instructs generation of a short, natural, exciting promotional message ending with a call to action and the product URL  
  - Edge cases: AI API failures, prompt formatting issues, rate limits.

#### 2.7 Message Cleaning & Delivery

- **Overview:** Cleans AI output to remove unwanted Markdown/formatting and sends the message via Telegram.
- **Nodes Involved:**  
  - `Clean AI Output` (Code node)  
  - `Telegram` node  
- **Node Details:**  
  - Clean AI Output node:  
    - Removes bold markup (`**`), converts line breaks to HTML `<br>`, escapes `<` and `>`  
  - Telegram node:  
    - Sends cleaned message text to a configured Telegram chat ID  
    - Uses Telegram API credentials  
  - Edge cases: Telegram API errors, message formatting issues.

---

### 3. Summary Table

| Node Name                                        | Node Type                         | Functional Role                                 | Input Node(s)                                   | Output Node(s)                                  | Sticky Note                                                                                          |
|-------------------------------------------------|----------------------------------|------------------------------------------------|------------------------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------|
| On form submission                              | Form Trigger                     | Receives user search keyword                    | -                                              | Triggers Bright Data scraping nodes             |                                                                                                    |
| Triggers Bright Data scraping for Amazon using keyword | HTTP Request                    | Starts Amazon scraping job                       | On form submission                              | Checks scrape status for Amazon (status: ready) |                                                                                                    |
| Checks scrape status for Amazon (status: ready) | HTTP Request                    | Polls Amazon scrape status                       | Triggers Bright Data scraping for Amazon        | If                                              |                                                                                                    |
| If                                              | If                              | Proceeds if Amazon scrape ready                  | Checks scrape status for Amazon                  | Fetches final scraped data from Amazon snapshot |                                                                                                    |
| Fetches final scraped data from Amazon snapshot | HTTP Request                    | Retrieves Amazon scraped product data            | If                                              | Filter1                                         |                                                                                                    |
| Filter1                                          | Filter                         | Filters Amazon products for keyword relevance   | Fetches final scraped data from Amazon snapshot | If9                                             |                                                                                                    |
| If9                                              | If                              | Checks if filtered Amazon data length > 0       | Filter1                                          | Google Sheets1                                  |                                                                                                    |
| Google Sheets1                                   | Google Sheets                   | Appends Amazon product data to sheet            | If9                                              | Merge                                           |                                                                                                    |
| Triggers Bright Data scraping for eBay           | HTTP Request                    | Starts eBay scraping job                          | On form submission                              | Checks scrape status for eBay                     |                                                                                                    |
| Checks scrape status for eBay                     | HTTP Request                    | Polls eBay scrape status                          | Triggers Bright Data scraping for eBay           | If1                                             |                                                                                                    |
| If1                                              | If                              | Proceeds if eBay scrape ready                     | Checks scrape status for eBay                     | Fetches final scraped data from eBay snapshot    |                                                                                                    |
| Fetches final scraped data from Etsy snapshot    | HTTP Request                    | Retrieves Etsy scraped product data               | If3                                              | Filter                                          |                                                                                                    |
| Filter                                           | Filter                         | Filters Etsy products for keyword relevance      | Fetches final scraped data from Etsy snapshot    | If10                                            |                                                                                                    |
| If10                                             | If                              | Checks if filtered Etsy data length > 0          | Filter                                           | Google Sheets                                   |                                                                                                    |
| Google Sheets                                    | Google Sheets                   | Appends Etsy product data to sheet               | If10                                             | Merge                                           | Combines all results from different platforms into one stream for further processing.              |
| Triggers Bright Data scraping for BestBuy        | HTTP Request                    | Starts BestBuy scraping job                       | On form submission                              | Checks scrape status for BestBuy                  |                                                                                                    |
| Checks scrape status for BestBuy                  | HTTP Request                    | Polls BestBuy scrape status                       | Triggers Bright Data scraping for BestBuy        | If4                                             |                                                                                                    |
| If4                                              | If                              | Proceeds if BestBuy scrape ready                  | Checks scrape status for BestBuy                  | Fetches final scraped data from BestBuy snapshot |                                                                                                    |
| Fetches final scraped data from BestBuy snapshot | HTTP Request                    | Retrieves BestBuy scraped product data           | If4                                              | Filter2                                         |                                                                                                    |
| Filter2                                           | Filter                         | Filters BestBuy products for keyword relevance   | Fetches final scraped data from BestBuy snapshot | If12                                            |                                                                                                    |
| If12                                             | If                              | Checks if filtered BestBuy data length > 0       | Filter2                                           | Google Sheets3                                  |                                                                                                    |
| Google Sheets3                                   | Google Sheets                   | Appends BestBuy product data to sheet            | If12                                             | Merge                                           |                                                                                                    |
| Triggers Bright Data scraping for Target          | HTTP Request                    | Starts Target scraping job                        | On form submission                              | Checks scrape status for Target                    |                                                                                                    |
| Checks scrape status for Target                    | HTTP Request                    | Polls Target scrape status                        | Triggers Bright Data scraping for Target          | If5                                             |                                                                                                    |
| If5                                              | If                              | Proceeds if Target scrape ready                   | Checks scrape status for Target                    | Fetches final scraped data from Target snapshot   |                                                                                                    |
| Fetches final scraped data from Target snapshot   | HTTP Request                    | Retrieves Target scraped product data             | If5                                              | Filter3                                         |                                                                                                    |
| Filter3                                           | Filter                         | Filters Target products for keyword relevance     | Fetches final scraped data from Target snapshot   | If13                                            |                                                                                                    |
| If13                                             | If                              | Checks if filtered Target data length > 0        | Filter3                                           | Google Sheets4                                  |                                                                                                    |
| Google Sheets4                                   | Google Sheets                   | Appends Target product data to sheet              | If13                                             | Merge                                           |                                                                                                    |
| Triggers Bright Data scraping for Home Depot      | HTTP Request                    | Starts Home Depot scraping job                    | On form submission                              | Checks scrape status for Home Depot                |                                                                                                    |
| Checks scrape status for Home Depot                | HTTP Request                    | Polls Home Depot scrape status                    | Triggers Bright Data scraping for Home Depot      | If6                                             |                                                                                                    |
| If6                                              | If                              | Proceeds if Home Depot scrape ready               | Checks scrape status for Home Depot                | Fetches final scraped data from Home Depot snapshot |                                                                                                    |
| Fetches final scraped data from Target snapshot1  | HTTP Request                    | Retrieves additional Target snapshot data          | If6                                              | Filter4                                         |                                                                                                    |
| Filter4                                           | Filter                         | Filters additional Target products for relevance  | Fetches final scraped data from Target snapshot1  | If14                                            |                                                                                                    |
| If14                                             | If                              | Checks if filtered additional Target data > 0    | Filter4                                           | Google Sheets5                                  |                                                                                                    |
| Google Sheets5                                   | Google Sheets                   | Appends additional Target product data to sheet   | If14                                             | Merge                                           |                                                                                                    |
| wayfair                                          | HTTP Request                    | Triggers Wayfair scraping job                      | On form submission                              | Wait6                                           |                                                                                                    |
| lowes                                            | HTTP Request                    | Triggers Lowe’s scraping job                        | On form submission                              | Wait7                                           |                                                                                                    |
| Wait                                             | Wait                           | Waits before checking Amazon scrape status         | If26                                             | Checks scrape status for Amazon                    |                                                                                                    |
| Wait1                                            | Wait                           | Waits before checking eBay scrape status           | If1                                              | Checks scrape status for eBay                      |                                                                                                    |
| Wait2                                            | Wait                           | Waits before checking Etsy scrape status           | If3                                              | Checks scrape status for Etsy                      |                                                                                                    |
| Wait3                                            | Wait                           | Waits before checking BestBuy scrape status        | If4                                              | Checks scrape status for BestBuy                   |                                                                                                    |
| Wait4                                            | Wait                           | Waits before checking Target scrape status         | If5                                              | Checks scrape status for Target                    |                                                                                                    |
| Wait5                                            | Wait                           | Waits before checking Home Depot scrape status     | If6                                              | Checks scrape status for Home Depot                |                                                                                                    |
| Wait6                                            | Wait                           | Waits before fetching Wayfair scrape snapshot       | wayfair                                          | HTTP Request14                                   |                                                                                                    |
| Wait7                                            | Wait                           | Waits before fetching Lowe’s scrape snapshot         | lowes                                            | HTTP Request16                                   |                                                                                                    |
| HTTP Request14                                   | HTTP Request                   | Fetches Wayfair snapshot data                        | Wait6                                            | If7                                             |                                                                                                    |
| HTTP Request15                                   | HTTP Request                   | Fetches snapshot data (used for multiple platforms) | If21                                             | Filter5                                         |                                                                                                    |
| HTTP Request16                                   | HTTP Request                   | Fetches Lowe’s snapshot data                         | Wait7                                            | If8                                             |                                                                                                    |
| HTTP Request17                                   | HTTP Request                   | Fetches snapshot data (used for multiple platforms) | If22                                             | Filter6                                         |                                                                                                    |
| Filter5                                           | Filter                         | Filters Wayfair products for keyword relevance      | HTTP Request15                                   | If16                                            |                                                                                                    |
| Filter6                                           | Filter                         | Filters Lowe’s products for keyword relevance        | HTTP Request17                                   | If15                                            |                                                                                                    |
| Merge                                            | Merge                          | Combines all platform data streams                    | Google Sheets1, Google Sheets, Google Sheets2, Google Sheets3, Google Sheets4, Google Sheets5, Google Sheets6, Google Sheets7 | Code1                                           | Combines all results from different platforms into one stream for further processing.              |
| Code1                                            | Code                           | Finds lowest price product from merged data          | Merge                                            | AI Agent                                        | Scans all merged product data and identifies the product with the lowest price. Returns title, price, and URL with a formatted message. |
| AI Agent                                         | LangChain Agent                | Creates promotional message for lowest price product | Code1                                            | Clean AI Output                                  | Uses Anthropic Chat Model to write a short promotional message based on the lowest price product. Keeps the tone exciting and urgent.       |
| Anthropic Chat Model                             | LangChain LM Chat Anthropic   | Anthropic Claude AI model for text generation        | AI Agent                                         | AI Agent                                        |                                                                                                    |
| Clean AI Output                                  | Code                          | Cleans formatting from AI output                       | AI Agent                                         | Telegram                                        | Cleans Claude’s response: removes formatting like **bold**, fixes < >, and adds HTML line breaks.  |
| Telegram                                         | Telegram                      | Sends promotional message to Telegram chat            | Clean AI Output                                  | -                                               | Sends the promotional message (lowest price product) to your Telegram channel or contact.          |
| Sticky Note                                      | Sticky Note                   | Workflow description note                              | -                                                | -                                               | 🔎 Multi-Platform Product Price Scraper (Amazon, eBay, Etsy, Target, BestBuy, Home Depot) ...       |
| Sticky Note1                                     | Sticky Note                   | Merge node description                                 | -                                                | -                                               | Combines all results from different platforms into one stream for further processing.              |
| Sticky Note2                                     | Sticky Note                   | Code1 node description                                 | -                                                | -                                               | Scans all merged product data and identifies the product with the lowest price...                   |
| Sticky Note3                                     | Sticky Note                   | AI Agent node description                              | -                                                | -                                               | Uses Anthropic Chat Model to write a short promotional message based on the lowest price product... |
| Sticky Note4                                     | Sticky Note                   | Clean AI Output node description                       | -                                                | -                                               | Cleans Claude’s response: removes formatting like **bold**, fixes < >, and adds HTML line breaks.  |
| Sticky Note5                                     | Sticky Note                   | Telegram node description                              | -                                                | -                                               | Sends the promotional message (lowest price product) to your Telegram channel or contact.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: Form Trigger  
   - Configure form title: "Keywords"  
   - Add field: Label "SearchHere", type: text input  
   - This node initiates the workflow on form submission.

2. **Create HTTP Request Nodes to Trigger Scraping Jobs:**  
   For each platform (Amazon, eBay, Etsy, BestBuy, Target, Home Depot, Wayfair, Lowe's):  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Authentication: Set header `Authorization: Bearer BRIGHT_DATA_API_KEY` (replace with real API key)  
   - Query Parameters:  
     - dataset_id: platform-specific ID (e.g., Amazon: gd_l7q7dkf244hwjntr0)  
     - include_errors: true  
     - type: discover_new  
     - discover_by: keyword or keywords (platform-dependent)  
     - limit_per_input: 4  
   - JSON Body: Use platform-specific JSON structure with input keywords from `{{$json.SearchHere}}` and required custom output fields (refer to workflow JSON for each platform's fields)  
   - Connect "On form submission" node output to all these HTTP Request nodes in parallel.

3. **Create Wait Nodes (1 minute each) for Each Platform:**  
   - Type: Wait  
   - Duration: 1 minute  
   - Connect each platform’s scrape trigger node to its corresponding Wait node to delay before checking status.

4. **Create HTTP Request Nodes to Check Scrape Status:**  
   For each platform:  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Query parameter: format=json  
   - Header: Authorization with Bright Data API key  
   - Connect each platform’s Wait node to corresponding status check node.

5. **Create If Nodes to Check if Status is "ready":**  
   - Condition: `$json.status` equals `"ready"`  
   - Connect each status check node to its If node.

6. **Create HTTP Request Nodes to Fetch Final Scraped Snapshot Data:**  
   For each platform:  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query parameter: format=json  
   - Header: Authorization with Bright Data API key  
   - Connect the true branch of each platform’s If node to its snapshot fetch node.

7. **Create Filter Nodes for Each Platform:**  
   - Type: Filter  
   - Condition: product title contains the normalized search keyword (lowercase, trimmed, no special chars)  
   - Connect each snapshot fetch node to its corresponding filter node.

8. **Create If Nodes to Check Filter Output Length > 0:**  
   - Condition: `$items.length > 0`  
   - Connect each filter node to an If node.

9. **Create Google Sheets Nodes for Each Platform:**  
   - Operation: Append or AppendOrUpdate (as per original workflow)  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: e.g., "Sheet1"  
   - Columns: Map `URL`, `Title`, `price` from filtered data  
   - Credentials: Google Sheets OAuth2 credentials configured  
   - Connect true branch of each platform’s If node (checking length >0) to the corresponding Google Sheets node.

10. **Create Merge Node:**  
    - Number of Inputs: 8 (one per platform)  
    - Connect output of all Google Sheets nodes as inputs to this Merge node.

11. **Create Code Node (Code1) to Identify Lowest Price Product:**  
    - Paste provided JavaScript code that:  
      - Validates data  
      - Extracts title, URL, price with flexible field names  
      - Finds lowest price product  
      - Returns structured data with a message  
    - Connect Merge node output to this Code node.

12. **Create AI Agent Node:**  
    - Type: LangChain Agent  
    - Configure prompt text to generate promotional message based on product details (title, price, URL) from Code node output  
    - Connect Code node output to AI Agent node.

13. **Create Anthropic Chat Model Node:**  
    - Model: Select "claude-sonnet-4-20250514" or your preferred Claude version  
    - Credentials: Anthropic API key  
    - Connect AI Agent node output to Anthropic Chat Model node.

14. **Connect Anthropic Chat Model back to AI Agent node:**  
    - To enable LangChain conversational flow.

15. **Create Clean AI Output Code Node:**  
    - JavaScript code to remove markdown bold, convert line breaks, escape characters  
    - Connect AI Agent output to this node.

16. **Create Telegram Node:**  
    - Configure with Telegram API credentials  
    - Set `chatId` to the Telegram channel or contact ID to send messages  
    - Map message text from the cleaned AI output node  
    - Connect Clean AI Output node output to Telegram node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 🔎 Multi-Platform Product Price Scraper (Amazon, eBay, Etsy, Target, BestBuy, Home Depot) with Bright Data API integration and Telegram notification delivery | Describes the core multi-platform scraping and processing workflow                              |
| Combines all results from different platforms into one stream for further processing.                                                                         | Related to the Merge node combining data                                                        |
| Scans all merged product data and identifies the product with the lowest price. Returns title, price, and URL with a formatted message.                       | Description of the custom code node for price comparison                                        |
| Uses Anthropic Chat Model to write a short promotional message based on the lowest price product. Keeps the tone exciting and urgent.                         | Details on AI message generation node                                                           |
| Cleans Claude’s response: removes formatting like **bold**, fixes < >, and adds HTML line breaks.                                                            | Description of AI output cleaning node                                                          |
| Sends the promotional message (lowest price product) to your Telegram channel or contact.                                                                     | Telegram node functionality                                                                     |
| Bright Data API documentation: https://brightdata.com/docs/api                                                                                               | For customizing scraping parameters and troubleshooting                                        |
| Anthropic Claude API documentation: https://docs.anthropic.com/                                                                                            | For AI model usage and credential setup                                                        |
| Telegram Bot API documentation: https://core.telegram.org/bots/api                                                                                           | For setting up Telegram credentials and chat IDs                                              |

---

This document provides a detailed, structured, and comprehensive description of the workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI systems.